---
layout: post
title: "TCP 单条指令被拆分为两个包 — PMTU Black Hole Detection 导致 MSS 降级分片"
date: 2025-08-28
categories: [TCP, Windows-Client]
tags: [tcp, pmtu, black-hole-detection, mss, mtu, packet-splitting, retransmission, vmware, ndis, congestion, large-send-offload]

---

# Case Summary: TCP 单条指令被拆分为两个包 — 网络拥塞触发 PMTU Black Hole Detection 导致 MSS 降级和分片

**Product/Service:** Windows (VMware VM) — TCP/IP Stack (TCPIP.sys)

---

## 1. 症状 (Symptoms)

- 客户使用**自研 App** 从 Windows 端向手机端发送随机生成的 AT 指令，正常情况下手机端收到完整指令后回复 `ATOK`。
- 经反复测试，大约 **1.6%** 的概率手机端**未及时回复 ATOK**，App 端报错。
- 经接收端 App 调查，发现失败原因是：**一条完整的指令被拆分到两个 TCP 包中发送**，而接收端 App 没有 TCP 分段重组能力，因此无法识别被拆分的指令。
- Wireshark 抓包确认：源端确实将单条指令（如 728 字节）分装在两个 TCP 包中发出。

## 2. 背景 (Background / Environment)

- **源端（发送端）**：VMware 虚拟机，Windows 系统
- **接收端**：手机设备
- **指令类型**：AT 指令（如 `AT+BK_WRITE_RPMB=...`），单条指令约 728 字节
- **MTU 设置**：源端网卡 MTU = 1514
- **问题模式**：
  - 问题不能稳定重现，但多次尝试会重现（约 1.6% 概率）
  - 当源端为**物理机**时问题不出现
  - **所有 Windows OS VMware VM** 均可复现此问题

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

### 第一轮：关闭 Large Send Offload — 无效

1. **做了什么**：基于指令被分片的表象，建议关闭网卡的 Large Send Offload (LSO) 功能，防止网卡层分片：
   ```powershell
   Set-NetAdapterAdvancedProperty -Name "Ethernet" `
     -DisplayName "Large Send Offload v2 (IPv4)" -DisplayValue "Disabled"
   Set-NetAdapterAdvancedProperty -Name "Ethernet" `
     -DisplayName "Large Send Offload v2 (IPv6)" -DisplayValue "Disabled"
   ```
   同时确认了源端 MTU 设置为 1514（`netsh interface ipv4 show interfaces`）。
2. **发现了什么**：关闭 LSO 后，问题**仍然可以复现**。
3. **得出了什么结论**：分片不是由 LSO/网卡硬件 Offload 造成的，需要从 TCP/IP 协议栈内部排查。

### 第二轮：TCPIP ETL — 确认 App 层完整、NDIS 层分片

1. **做了什么**：抓取 TCPIP ETL 日志（使用 `netsh trace` 命令捕获 TCPIP、NDIS 等 Provider），分析从 App 层到 NDIS 层的数据流转。
2. **发现了什么**：

   **App 层（Winsock）**：App 发送了完整的 728 字节：
   ```
   [Winsock] send: Buffer Length 728, Seq 3047, Status STATUS_SUCCESS
   [TCPIP] TCP: connection send posted 728 bytes at 3912558683
   ```

   **WFP Stream 层**：数据完整通过 WFP 检查（728 字节）：
   ```
   [stream] WfpStreamInspectSend() - data length = 728
   [stream] StreamClassify() - length = 728, flags = 0x10000
   ```

   **NDIS 层**：数据被拆分为两个 Fragment 发出：
   ```
   [NDIS-PacketCapture] Packet Fragment (590 bytes)
   [NDIS-PacketCapture] Packet Fragment (246 bytes)
   ```

   分片计算：
   - 526 数据 + 40 TCP/IP Header + 14 Ethernet Header = **590 字节**
   - 192 数据 + 40 TCP/IP Header + 14 Ethernet Header = **246 字节**
   - 526 + 192 = **718 字节**（加上 TCP Header 中的选项等 = 728 字节数据）

   **TCP 层确认**：TCP 协议栈将 728 字节分两次 advance：
   ```
   [TCPIP] TCP: send advance 536 bytes at 3912558683
   [TCPIP] TCP: send advance 192 bytes at 3912559219
   [TCPIP] TCP: send complete 728 bytes at 3912558683 (normal)
   ```

3. **得出了什么结论**：App 层发出的是完整 728 字节，但 **TCP 协议栈在 NDIS 层面将其拆分为 536 + 192 字节**发出。536 这个数字非常可疑——它恰好是 RFC 879 定义的最小 MSS（Maximum Segment Size）默认值。

### 第三轮：Wireshark 对比分析 — 分片与重传的关联

1. **做了什么**：仔细分析 Wireshark 抓包，按目标 IP 分组对比正常（不分片）和异常（分片）的 TCP 会话。
2. **发现了什么**：

   | 目标 IP | 分片（异常） | 不分片（正常） | 重传现象 |
   |---------|------------|--------------|---------|
   | `10.10.209.170` | ✅ 有 | ❌ 无 | ✅ **大量重传** |
   | `10.10.165.2` | ✅ 有 | ❌ 无 | ✅ **大量重传** |
   | `10.10.250.76` | ❌ 无 | ✅ 有 | ❌ 无重传 |
   | `10.10.82.28` | ❌ 无 | ✅ 有 | ❌ 无重传 |

3. **得出了什么结论**：**出现分片的 TCP 会话中都伴随大量重传；未分片的会话中没有重传**。重传 = 丢包 = 网络拥塞。分片现象与网络拥塞高度相关。结合 536 字节的 MSS 值，怀疑 **PMTU Black Hole Detection** 机制被触发。

### 第四轮：TCPIP ETL — 确认 PMTU Black Hole 模式切换

1. **做了什么**：在 TCPIP ETL 中搜索 PMTU Black Hole 相关日志条目，特别关注出现重传的时间段。
2. **发现了什么**：在重传包附近，发现**频繁的 Black Hole 进入和退出记录**：
   ```
   [TCPIP] TCP: connection 0xFFFFE208656EBB90
     Exiting BH due to full ack received,
     BH mss 536, Original MSS 1386

   [TCPIP] TCP: connection 0xFFFFE208656EBB90
     entered BH, BH MSS 536, original MSS 1386
   ```

   关键数据点：
   - **Original MSS = 1386**（正常情况下的 MSS）
   - **BH MSS = 536**（进入 Black Hole 模式后降级到的最小 MSS）
   - TCP 连接在 BH 模式和正常模式之间**频繁切换**

3. **得出了什么结论**：**根因确认**——TCP/IP 协议栈的 PMTU Black Hole Detection 机制检测到丢包/重传后，认为路径上存在 MTU Black Hole（即某个路由器因 MTU 限制丢弃了设置了 DF 标志的大包但未返回 ICMP Fragmentation Needed），因此将 MSS 从 1386 降级到 536（最小值），导致 728 字节的指令被拆分为 536 + 192 两个 TCP Segment。

### PMTU Black Hole Detection 机制解读

**PMTU（Path MTU Discovery）正常工作流程**：
1. 源主机发送带 DF（Don't Fragment）标志的 IP 包
2. 如果中间路由器发现包太大无法在当前链路传输且不能分片，则丢弃该包
3. 路由器返回 **ICMP Type 3 Code 4**（Destination Unreachable - Fragmentation Needed and DF Set），附带其支持的最大 MTU
4. 源主机根据 ICMP 中的 MTU 值调整发送包大小

**PMTU Black Hole 场景**：
- 当中间路由器丢弃了过大的包但**没有返回 ICMP 消息**时（被防火墙过滤、路由器不支持等），就形成了 "Black Hole"
- 源主机等不到 ICMP，只看到重传超时
- Windows 的 Black Hole Detection 机制检测到连续重传后，会**将 MSS 降到 536**（最小值）尝试通过
- 收到 ACK 后退出 BH 模式，恢复正常 MSS
- 如果再次发生拥塞/重传，又会重新进入 BH 模式 → **频繁切换**

**在客户环境中的具体表现**：
- 源端从物理机迁移到 VMware 平台后，网络路径发生变化
- 新路径上偶尔出现网络拥塞导致丢包和重传
- PMTU Black Hole Detection 误判为 MTU Black Hole，降级 MSS 到 536
- 728 字节指令被拆分为 536 + 192 → 接收端 App 无法重组 → 报错

### 第五轮：关闭 PMTU Black Hole Detection — 问题解决

1. **做了什么**：在客户端注册表中关闭 PMTU Black Hole Detection：
   ```
   Registry Path: HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters
   Value Name:    EnablePMTUBHDetect
   Value Type:    REG_DWORD
   Value Data:    0
   ```
   修改后重启机器。
2. **发现了什么**：重启后反复测试，**指令不再被拆分**，接收端 App 正常接收完整指令，问题消失。
3. **得出了什么结论**：关闭 PMTU Black Hole Detection 后，TCP 不再因网络拥塞导致的重传而误降 MSS，指令以完整的单个 TCP Segment 发送，问题彻底解决。

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| 问题仅 1.6% 概率出现，不能稳定重现 | 需要多次测试才能捕获到问题发生的瞬间 | 长时间运行测试 + 持续抓包/ETL 日志，利用多次复现积累足够的问题样本 |
| 关闭 LSO 后问题仍存在，初始排查方向偏移 | 排除了硬件 Offload 因素，需要转向协议栈内部分析 | 通过 TCPIP ETL 日志逐层（App → Winsock → TCP → NDIS）追踪数据流转，精确定位到 TCP 层的 MSS 降级 |
| 536 字节这个分片大小需要理论知识来解释 | 不了解 PMTU BH 机制的工程师可能无法将 536 与 Black Hole Detection 关联 | 通过 Microsoft KB 文档中关于 PMTU Black Hole 的说明，建立 536 MSS 与 BH 模式的关联 |
| 问题仅在 VMware VM 上出现，物理机不受影响 | 需要理解 VM 化后的网络路径差异 | 分析得出 VMware 平台的虚拟化网络路径引入了额外跳数和潜在拥塞点 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

**VMware 虚拟化环境中网络路径偶发拥塞，触发 Windows TCP/IP 协议栈的 PMTU Black Hole Detection 机制，导致 MSS 从 1386 降级到 536，使得大于 536 字节的 TCP 负载被拆分为多个 Segment。**

具体机制：
1. 源端从物理机迁移到 VMware VM 后，网络路径变化，引入虚拟化网络层
2. 到部分目标 IP 的路径偶尔出现**网络拥塞**，导致丢包和 TCP 重传
3. Windows TCP/IP 的 PMTU Black Hole Detection 机制检测到连续重传，误判为路径上存在 MTU Black Hole
4. MSS 从正常值 **1386** 降级到最小值 **536**
5. 原本 728 字节的指令在 MSS=536 时被拆分为 **536 + 192** 两个 TCP Segment
6. 接收端 App 无 TCP 分段重组能力，收到不完整数据 → 认为指令无效 → 不回复 ATOK → 发送端报错

### Resolution

关闭 PMTU Black Hole Detection：
1. 打开注册表编辑器
2. 定位到 `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters`
3. 新建或修改 DWORD 值 `EnablePMTUBHDetect`，设置为 **`0`**
4. 重启机器
5. 验证问题不再出现

### Workaround

如果不希望修改注册表，可考虑以下替代方案：
- **接收端 App 增加 TCP 分段重组逻辑**：作为应用层最佳实践，TCP 是流协议，应用层不应假设一次 `send()` 对应一次 `recv()`，应实现完整的消息边界处理
- **调整网络路径**：优化 VMware 虚拟网络配置，减少中间跳数和拥塞风险

## 6. 经验教训 (Lessons Learned)

- **技术知识**：
  - **PMTU Black Hole Detection 机制**：当 Windows TCP/IP 检测到连续重传（可能因网络拥塞、丢包），会怀疑路径上存在 MTU Black Hole（路由器丢弃大包但不返回 ICMP），此时将 MSS 降到最小值 536 尝试通过。通过注册表 `EnablePMTUBHDetect = 0` 可关闭此机制。TCPIP ETL 中 `entered BH` / `Exiting BH` 日志是确认此行为的直接证据。
  - **536 字节的含义**：536 是 RFC 879 定义的 TCP 最小 MSS 默认值（576 字节最小 IP MTU - 20 字节 IP Header - 20 字节 TCP Header = 536）。当 TCPIP ETL 中看到 `BH MSS 536` 时，说明 TCP 已进入 PMTU Black Hole 模式。
  - **VMware 虚拟化与网络路径变化**：从物理机迁移到 VM 后，数据包经过虚拟交换机、物理交换机、可能的 VXLAN/Overlay 网络，路径更长、更复杂，更容易遇到拥塞和 MTU 问题。这解释了为什么同样的 App 在物理机上正常但在 VM 上出问题。
  - **TCP 是流协议**：应用层不应假设 TCP 会按 `send()` 的边界交付数据。无论是否存在 PMTU BH 问题，接收端 App 都应实现 TCP 分段重组逻辑，这是应用层编程的最佳实践。

- **排查方法**：
  - **TCPIP ETL 逐层分析法**：当怀疑 TCP 层面分片时，使用 TCPIP ETL 从 **App 层（Winsock）→ WFP 层 → TCP 层 → NDIS 层**逐层追踪数据大小变化，精确定位分片发生在哪一层。
  - **Wireshark 分组对比法**：按目标 IP 分组，对比正常和异常会话的差异（有无重传、有无分片），快速建立关联——"有分片的会话都有重传"是 PMTU BH 问题的典型信号。
  - **关键 ETL 关键词**：搜索 `entered BH`、`Exiting BH`、`BH MSS`、`Original MSS` 可快速确认 PMTU Black Hole Detection 是否被触发。
  - **排查顺序建议**：① 确认 LSO/Offload 是否相关 → ② TCPIP ETL 逐层追踪 → ③ Wireshark 对比正常/异常会话 → ④ ETL 搜索 BH 相关日志 → ⑤ 注册表修改验证。

- **预防措施**：
  - 在 VMware 等虚拟化环境中部署对 TCP 分段敏感的应用时，建议提前评估是否需要关闭 PMTU Black Hole Detection。
  - 应用层开发应遵循 TCP 流协议的最佳实践，实现消息边界和分段重组逻辑，不依赖底层 TCP 的 Segment 边界。
  - 虚拟化网络架构设计中，注意 MTU 一致性（包括 VXLAN Overlay 的 MTU Overhead），避免路径中出现 MTU 不匹配。

## 7. 参考文档 (References)

- [TCP/IP Performance Known Issues — Microsoft Troubleshoot](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/tcpip-performance-known-issues) — TCP/IP 性能已知问题，包括吞吐量和窗口调优
- [RFC 1191 — Path MTU Discovery](https://datatracker.ietf.org/doc/html/rfc1191) — PMTU 发现机制的标准定义

---
---

# Case Summary: Single TCP Command Split Into Two Packets — PMTU Black Hole Detection Causing MSS Downgrade and Segmentation

**Product/Service:** Windows (VMware VM) — TCP/IP Stack (TCPIP.sys)

---

## 1. Symptoms

- The customer used a **custom App** to send randomly generated AT commands from a Windows machine to mobile devices. Normally, the mobile device replies with `ATOK` after receiving the complete command.
- After repeated testing, approximately **1.6%** of the time the mobile device **failed to reply ATOK** in time, causing the App to report an error.
- Investigation on the receiving App revealed: **a single complete command was split across two TCP packets**, and the receiving App lacked TCP segment reassembly capability, so it couldn't recognize the split command.
- Wireshark captures confirmed: the source indeed sent a single command (e.g., 728 bytes) in two separate TCP packets.

## 2. Background / Environment

- **Source (sender)**: VMware virtual machine, Windows OS
- **Receiver**: Mobile phone device
- **Command type**: AT commands (e.g., `AT+BK_WRITE_RPMB=...`), approximately 728 bytes per command
- **MTU setting**: Source NIC MTU = 1514
- **Issue pattern**:
  - Not consistently reproducible but occurs with repeated testing (~1.6% probability)
  - Does **not occur** when the source is a **physical machine**
  - Reproducible on **all Windows OS VMware VMs**

## 3. Investigation & Troubleshooting

### Round 1: Disabled Large Send Offload — Ineffective

1. **Action**: Based on the packet-splitting symptom, disabled Large Send Offload (LSO) on the NIC:
   ```powershell
   Set-NetAdapterAdvancedProperty -Name "Ethernet" `
     -DisplayName "Large Send Offload v2 (IPv4)" -DisplayValue "Disabled"
   Set-NetAdapterAdvancedProperty -Name "Ethernet" `
     -DisplayName "Large Send Offload v2 (IPv6)" -DisplayValue "Disabled"
   ```
   Also confirmed the source MTU was set to 1514 (`netsh interface ipv4 show interfaces`).
2. **Finding**: After disabling LSO, the problem **could still be reproduced**.
3. **Conclusion**: The splitting was not caused by LSO/NIC hardware offload. Investigation needed to move to the TCP/IP stack internals.

### Round 2: TCPIP ETL — Confirmed App Layer Complete, NDIS Layer Split

1. **Action**: Captured TCPIP ETL logs (using `netsh trace` with TCPIP, NDIS providers) and analyzed data flow from App layer to NDIS layer.
2. **Finding**:

   **App layer (Winsock)**: App sent the complete 728 bytes:
   ```
   [Winsock] send: Buffer Length 728, Seq 3047, Status STATUS_SUCCESS
   [TCPIP] TCP: connection send posted 728 bytes at 3912558683
   ```

   **WFP Stream layer**: Data passed through WFP inspection intact (728 bytes):
   ```
   [stream] WfpStreamInspectSend() - data length = 728
   [stream] StreamClassify() - length = 728, flags = 0x10000
   ```

   **NDIS layer**: Data was split into two fragments:
   ```
   [NDIS-PacketCapture] Packet Fragment (590 bytes)
   [NDIS-PacketCapture] Packet Fragment (246 bytes)
   ```

   Fragment calculation:
   - 526 data + 40 TCP/IP Header + 14 Ethernet Header = **590 bytes**
   - 192 data + 40 TCP/IP Header + 14 Ethernet Header = **246 bytes**

   **TCP layer confirmation**: TCP stack advanced 728 bytes in two batches:
   ```
   [TCPIP] TCP: send advance 536 bytes at 3912558683
   [TCPIP] TCP: send advance 192 bytes at 3912559219
   [TCPIP] TCP: send complete 728 bytes at 3912558683 (normal)
   ```

3. **Conclusion**: The App sent a complete 728 bytes, but the **TCP stack split it into 536 + 192 bytes at the NDIS layer**. The number 536 is highly suspicious — it's exactly the minimum default MSS (Maximum Segment Size) defined in RFC 879.

### Round 3: Wireshark Comparative Analysis — Correlation Between Splitting and Retransmissions

1. **Action**: Analyzed Wireshark captures, grouping TCP conversations by destination IP and comparing normal (no splitting) vs. abnormal (splitting) sessions.
2. **Finding**:

   | Destination IP | Splitting (Abnormal) | No Splitting (Normal) | Retransmissions |
   |---------------|---------------------|----------------------|----------------|
   | `10.10.209.170` | ✅ Yes | ❌ No | ✅ **Heavy retransmissions** |
   | `10.10.165.2` | ✅ Yes | ❌ No | ✅ **Heavy retransmissions** |
   | `10.10.250.76` | ❌ No | ✅ Yes | ❌ None |
   | `10.10.82.28` | ❌ No | ✅ Yes | ❌ None |

3. **Conclusion**: **TCP sessions with packet splitting always had heavy retransmissions; sessions without splitting had no retransmissions**. Retransmissions = packet loss = network congestion. Splitting was strongly correlated with network congestion. Combined with the 536-byte MSS, **PMTU Black Hole Detection** was the prime suspect.

### Round 4: TCPIP ETL — Confirmed PMTU Black Hole Mode Switching

1. **Action**: Searched TCPIP ETL for PMTU Black Hole related log entries, focusing on time windows around retransmissions.
2. **Finding**: Near the retransmission packets, found **frequent Black Hole entry/exit records**:
   ```
   [TCPIP] TCP: connection 0xFFFFE208656EBB90
     Exiting BH due to full ack received,
     BH mss 536, Original MSS 1386

   [TCPIP] TCP: connection 0xFFFFE208656EBB90
     entered BH, BH MSS 536, original MSS 1386
   ```

   Key data points:
   - **Original MSS = 1386** (normal MSS)
   - **BH MSS = 536** (degraded MSS in Black Hole mode)
   - TCP connection was **frequently switching** between BH mode and normal mode

3. **Conclusion**: **Root cause confirmed** — The TCP/IP stack's PMTU Black Hole Detection mechanism detected consecutive retransmissions and assumed a MTU Black Hole existed on the path (a router dropping DF-flagged packets without returning ICMP Fragmentation Needed). It degraded MSS from 1386 to 536 (minimum), causing the 728-byte command to be split into 536 + 192 bytes across two TCP segments.

### PMTU Black Hole Detection Mechanism Explained

**Normal PMTU Discovery workflow**:
1. Source sends IP packets with the DF (Don't Fragment) flag set
2. If an intermediate router finds the packet too large for the link and cannot fragment it, it drops the packet
3. Router returns **ICMP Type 3 Code 4** (Destination Unreachable - Fragmentation Needed and DF Set) with the supported MTU
4. Source adjusts packet size based on the ICMP-reported MTU

**PMTU Black Hole scenario**:
- When a router drops oversized packets but **fails to return ICMP messages** (filtered by firewall, unsupported, etc.), a "Black Hole" forms
- Source sees only retransmission timeouts with no ICMP feedback
- Windows Black Hole Detection detects consecutive retransmissions → **degrades MSS to 536** (minimum) to try to pass through
- On receiving an ACK, exits BH mode and restores normal MSS
- If congestion/retransmissions recur, re-enters BH mode → **frequent switching**

**In the customer's environment**:
- Source migrated from physical to VMware platform → network path changed
- New paths to certain destinations experienced occasional congestion → packet loss and retransmissions
- PMTU BH Detection misinterpreted congestion-induced retransmissions as MTU Black Holes → MSS degraded to 536
- 728-byte command split into 536 + 192 → receiving App couldn't reassemble → error

### Round 5: Disabled PMTU Black Hole Detection — Problem Resolved

1. **Action**: Disabled PMTU Black Hole Detection via registry:
   ```
   Registry Path: HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters
   Value Name:    EnablePMTUBHDetect
   Value Type:    REG_DWORD
   Value Data:    0
   ```
   Restarted the machine.
2. **Finding**: After restart, repeated testing confirmed **commands were no longer split**. The receiving App correctly received complete commands, and the issue disappeared.
3. **Conclusion**: With PMTU BH Detection disabled, TCP no longer degraded MSS due to congestion-induced retransmissions. Commands were sent as complete single TCP segments, fully resolving the issue.

## 4. Blockers & How They Were Resolved

| Blocker | Impact | How It Was Resolved |
|---------|--------|---------------------|
| Issue only occurred ~1.6% of the time; not consistently reproducible | Required many test iterations to capture the problem occurrence | Extended testing runs + continuous packet capture/ETL logging to accumulate sufficient problem samples |
| Disabling LSO didn't help; initial investigation direction was off | Ruled out hardware offload; needed to pivot to protocol stack internals | Used TCPIP ETL to trace data layer by layer (App → Winsock → TCP → NDIS), pinpointing the split at the TCP layer MSS downgrade |
| The 536-byte split size required domain knowledge to interpret | Engineers unfamiliar with PMTU BH might not connect 536 to Black Hole Detection | Referenced Microsoft KB documentation on PMTU Black Hole to establish the link between 536 MSS and BH mode |
| Issue only appeared on VMware VMs, not physical machines | Required understanding the network path differences in virtualized environments | Analysis concluded that VMware's virtualized network path introduced additional hops and potential congestion points |

## 5. Root Cause & Resolution

### Root Cause

**Intermittent network congestion in the VMware virtualized environment triggered Windows TCP/IP's PMTU Black Hole Detection mechanism, degrading MSS from 1386 to 536, causing TCP payloads larger than 536 bytes to be split into multiple segments.**

Detailed mechanism:
1. Source migrated from physical to VMware VM → network path changed with additional virtualization layers
2. Paths to certain destination IPs experienced occasional **network congestion** → packet loss and TCP retransmissions
3. Windows TCP/IP's PMTU Black Hole Detection detected consecutive retransmissions → misinterpreted as an MTU Black Hole on the path
4. MSS degraded from **1386** to the minimum **536**
5. The 728-byte command was split into **536 + 192** bytes across two TCP segments
6. Receiving App had no TCP segment reassembly capability → received incomplete data → didn't reply ATOK → sender reported error

### Resolution

Disable PMTU Black Hole Detection:
1. Open Registry Editor
2. Navigate to `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters`
3. Create or modify DWORD value `EnablePMTUBHDetect`, set to **`0`**
4. Restart the machine
5. Verify the issue no longer occurs

### Workaround

Alternative approaches if registry modification is not preferred:
- **Implement TCP segment reassembly in the receiving App**: As an application-layer best practice, TCP is a stream protocol — applications should not assume a 1:1 correspondence between `send()` and `recv()` calls, and should implement proper message boundary handling
- **Optimize network path**: Adjust VMware virtual network configuration to reduce intermediate hops and congestion risk

## 6. Lessons Learned

- **Technical Knowledge**:
  - **PMTU Black Hole Detection Mechanism**: When Windows TCP/IP detects consecutive retransmissions (possibly from network congestion/packet loss), it suspects an MTU Black Hole on the path (router dropping large packets without ICMP response) and degrades MSS to the minimum value 536. The registry key `EnablePMTUBHDetect = 0` disables this mechanism. ETL entries `entered BH` / `Exiting BH` are direct evidence of this behavior.
  - **Significance of 536 bytes**: 536 is the TCP minimum default MSS defined in RFC 879 (576 min IP MTU - 20 IP Header - 20 TCP Header = 536). When TCPIP ETL shows `BH MSS 536`, TCP has entered PMTU Black Hole mode.
  - **VMware Virtualization and Network Path Changes**: After migrating from physical to VM, packets traverse virtual switches, physical switches, and potentially VXLAN/overlay networks — longer, more complex paths with greater congestion and MTU mismatch risk. This explains why the same App works on physical machines but fails on VMs.
  - **TCP is a Stream Protocol**: Applications should never assume TCP delivers data aligned to `send()` boundaries. Regardless of PMTU BH issues, receiving applications should implement TCP segment reassembly logic — this is a fundamental best practice.

- **Troubleshooting Methodology**:
  - **TCPIP ETL Layer-by-Layer Analysis**: When TCP-layer splitting is suspected, use TCPIP ETL to trace data size changes from **App (Winsock) → WFP → TCP → NDIS**, precisely identifying which layer performs the split.
  - **Wireshark Grouped Comparison**: Group by destination IP, compare normal vs. abnormal sessions for differences (retransmissions, splitting). "Sessions with splitting all have retransmissions" is a classic PMTU BH indicator.
  - **Key ETL Keywords**: Search for `entered BH`, `Exiting BH`, `BH MSS`, `Original MSS` to quickly confirm whether PMTU Black Hole Detection was triggered.
  - **Investigation Order**: ① Check LSO/Offload → ② TCPIP ETL layer-by-layer tracing → ③ Wireshark compare normal/abnormal sessions → ④ ETL search for BH entries → ⑤ Registry modification and verification.

- **Prevention**:
  - When deploying TCP segment-sensitive applications in VMware or other virtualized environments, proactively evaluate whether PMTU Black Hole Detection should be disabled.
  - Application-layer development should follow TCP stream protocol best practices, implementing message boundaries and segment reassembly without relying on TCP segment boundaries.
  - In virtualized network architecture design, ensure MTU consistency (including VXLAN overlay MTU overhead) to avoid path MTU mismatches.

## 7. References

- [TCP/IP Performance Known Issues — Microsoft Troubleshoot](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/tcpip-performance-known-issues) — TCP/IP performance known issues including throughput and window tuning
- [RFC 1191 — Path MTU Discovery](https://datatracker.ietf.org/doc/html/rfc1191) — Standard definition of the PMTU Discovery mechanism
