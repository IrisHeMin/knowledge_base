---
layout: post
title: "DHCP Duplicate IP — Option 82 子网不匹配导致租约被提前删除"
date: 2025-12-08
categories: [DHCP, Windows-Server]
tags: [dhcp, option82, duplicate-ip, nack, relay-agent, failover, lease, subnet-mismatch, linux-client]

---

# Case Summary: DHCP Duplicate IP — Option 82 子网不匹配导致租约提前删除并重分配

**Product/Service:** Windows Server 2022 — DHCP Server Role

---

## 1. 症状 (Symptoms)

- 客户环境中多台设备告警 **Duplicate IP**，同一个 IP 地址同时被两台设备使用，业务间歇性中断。
- 在 DHCP Server 上查询租约时发现：某条 IP 租约**在过期时间未到之前就被删除**，随后同一个 IP 被分配给了另一台设备。
  - 例如：`Get-DhcpServerv4Lease -IPAddress 10.20.232.99` 在 12:11:34 查到租约过期时间约 12:23:17，但在 12:11:45 再查时该租约已消失；12:14:51 该 IP 已被分配给另一台设备。
- 问题频繁发生，涉及多个 IP 段。

## 2. 背景 (Background / Environment)

- **DHCP Server**：Windows Server 2022
- **DHCP Failover**：已配置，模式为 **Load Balance**
- **MCLT（Maximum Client Lead Time）**：15 分钟
- **Lease 时间**：4 小时
- **注册表配置**：`DhcpGlobalFlagForSubnetChangeDHCPRequest = 1`（位于 `HKLM\SYSTEM\CurrentControlSet\Services\DhcpServer\Parameters`）
- **客户端**：Linux 设备，配置双网卡，分属不同 VLAN
- **网络架构**：跨子网部署，使用 DHCP Relay Agent 转发，Relay 配置了 Option 82

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

### 第一轮：租约为何提前消失？

1. **做了什么**：在 DHCP Server 上使用 `Get-DhcpServerv4Lease` 反复查询问题 IP（如 `10.20.232.99`）的租约状态。
2. **发现了什么**：
   - 12:11:34 租约存在，过期时间约 12:23:17
   - 12:11:45 同一 IP 的租约已不存在
   - 12:14:51 该 IP 已被分配给另一台设备
3. **得出了什么结论**：DHCP Server 在租约有效期内主动删除了该租约，并将 IP 重新分配。需要确认是客户端主动释放还是 Server 端行为。

### 第二轮：客户端是否主动释放？

1. **做了什么**：抓取问题时间段的网络流量，重点筛查 `DHCP Message Type = DHCP Release`。
2. **发现了什么**：
   - **没有**看到客户端发送任何 DHCPRELEASE 报文
   - 客户端本机上 IP 仍然在使用中
3. **得出了什么结论**：不是客户端主动释放 IP，删除租约的动作来自 DHCP Server 本身。

### 第三轮：DHCP 审计日志回溯

1. **做了什么**：查看 DHCP 审计日志（Audit Log），还原问题 IP 的完整事件链。
2. **发现了什么**：一个固定的事件模式：
   - 客户端首次成功获取 IP（Assign），MCLT 为 15 分钟
   - 在租约时间一半（~7.5 分钟）时客户端发起续租（DHCPREQUEST）
   - DHCP Server 返回 **NACK**（DHCPNAK）
   - 随后同一 IP 被 Assign 给其他设备

   审计日志片段示例：
   ```
   10,12/08/25,11:04:12,Assign,10.20.232.99,,AABBCCDD1122,...
   15,12/08/25,11:11:42,NACK,10.20.232.99,,AABBCCDD1122,...
   15,12/08/25,11:15:27,NACK,10.20.232.99,,AABBCCDD1122,...
   15,12/08/25,11:17:19,NACK,10.20.232.99,,AABBCCDD1122,...
   10,12/08/25,11:18:15,Assign,10.20.232.99,,AABBCCDD1122,...
   ```
3. **得出了什么结论**：行为链路清晰——客户端续租 → Server NACK → Server 删除旧租约 → IP 重新分配。需要进一步确认 NACK 的原因。

### 第四轮：NACK 根因——Option 82 子网不匹配

1. **做了什么**：分析问题时间段的网络抓包和 DHCP Server ETL 日志，重点关注 DHCPREQUEST 中的 Option 82 信息。
2. **发现了什么**：
   - 客户端首次获取的 IP 为 `10.20.225.63`，属于 Subnet A
   - 续租时 DHCPREQUEST 报文中 **Option 82 携带的子网信息指向 Subnet B**（`203.0.113.0`）
   - DHCP Server ETL 日志明确记录：
     ```
     REQUEST for address 10.20.225.63
     REQUEST from relay agent: 198.51.100.30
     Interface subnet = 198.51.100.30
     Mismatch in ClientSubnet determined from request message
       (option 82 suboption 5 section) 203.0.113.0 and present in database 203.0.113.0
     DhcpDoDynDnsCheckDelete() - Deleting record 10.20.225.63
     Invalid DHCPREQUEST for 10.20.225.63 Nack'd.
     ```
3. **得出了什么结论**：
   - 客户端拿着 Subnet A 的 IP，续租时 Option 82 却声明位于 Subnet B
   - DHCP Server 发现子网不匹配，判定为"客户端已换子网"
   - 由于启用了 `DhcpGlobalFlagForSubnetChangeDHCPRequest = 1`，Server 执行：**删除旧租约 → 返回 NACK → 期待客户端重新 DISCOVER**

### 第五轮：验证注册表行为——By Design

1. **做了什么**：检查 DHCP Server 注册表配置 `HKLM\SYSTEM\CurrentControlSet\Services\DhcpServer\Parameters\DhcpGlobalFlagForSubnetChangeDHCPRequest`。
2. **发现了什么**：值为 **1**。此设置的含义为：
   - 当 DHCPREQUEST 中 Option 82 指示客户端已"换子网"时
   - DHCP Server 将：找到旧租约 → 删除 → 返回 NACK → 强制客户端重新 DISCOVER
3. **得出了什么结论**：DHCP Server 的删除租约 + NACK 行为是 **By Design** 的增强行为，目的是避免客户端在换子网后仍持有旧子网 IP。Server 侧逻辑完全正常。

### 第六轮：客户端未按协议响应 NACK

1. **做了什么**：与客户确认客户端实际行为。经确认，客户端确实因双网卡配置从 Subnet A 漂移到了 Subnet B（场景 2）。
2. **发现了什么**：
   - 按照 DHCP 协议预期：客户端收到 NACK 后应**丢弃当前 IP → 重新发起 DHCPDISCOVER**
   - 实际行为：客户端收到 NACK 后**没有重新 DISCOVER**，继续使用已被 Server 回收的旧 IP
   - 同时该 IP 被 Server 正常分配给另一台设备 → **Duplicate IP**
3. **得出了什么结论**：根因在客户端——Linux 设备双网卡跨 VLAN 配置问题导致对 DHCPNAK 的处理不符合协议预期。

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| 初始怀疑方向是 DHCP Server 分配异常，需多轮日志分析才定位到 Option 82 | 延长了排查时间 | 通过 Audit Log + ETL 日志 + 网络抓包三方交叉验证，还原完整事件链 |
| 需要确认客户端是否真正发生子网漂移（场景 1 vs 场景 2） | 无法判断是 Relay 配置问题还是客户端真实漂移 | 与客户确认实际网络拓扑和客户端物理位置变化 |
| 客户端为 Linux 设备，DHCP 行为非 Windows 标准实现 | 无法直接调试客户端协议栈 | 确认为客户端侧问题后，转交客户自行与对应厂商排查 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

**多因素叠加导致 Duplicate IP：**

1. **客户端（Linux，双网卡）发生子网漂移**：客户端从 Subnet A 移动到 Subnet B，续租时 Option 82 携带了 Subnet B 的信息。
2. **DHCP Server 正确响应（By Design）**：由于启用了 `DhcpGlobalFlagForSubnetChangeDHCPRequest = 1`，Server 检测到子网不匹配后删除旧租约并返回 NACK，期待客户端重新 DISCOVER。
3. **客户端未正确处理 NACK**：Linux 客户端收到 NACK 后未丢弃旧 IP、未重新 DISCOVER，继续使用已被回收的 IP。
4. **IP 冲突产生**：被回收的 IP 被 Server 分配给其他设备，导致两台设备同时使用同一 IP。

### Resolution

1. 确认 DHCP Server 行为正常——`DhcpGlobalFlagForSubnetChangeDHCPRequest = 1` 在此环境下是必要且正确的配置。
2. 将问题聚焦到客户端侧：Linux 设备双网卡配置导致 DHCP 协议处理异常。
3. 客户自行与 Linux 设备厂商/团队排查并修复客户端的 DHCP 行为（确保收到 NACK 后正确丢弃 IP 并重新 DISCOVER）。

### Workaround

在客户端侧修复之前，可考虑的临时缓解措施：
- 为频繁出问题的设备配置 **DHCP Reservation**，避免 IP 被回收后分配给其他设备。
- 缩短 Lease 时间以减少冲突窗口期。

## 6. 经验教训 (Lessons Learned)

- **技术知识**：
  - `DhcpGlobalFlagForSubnetChangeDHCPRequest` 是一个关键的增强行为注册表开关。启用后，DHCP Server 在检测到续租请求中 Option 82 指示的子网与旧租约不一致时，会主动删除租约 + NACK，强制客户端重新获取 IP。这在 EVPN/VXLAN/多 VLAN 环境中尤为重要。
  - DHCP Option 82 是 Relay Agent 为 DHCP 报文添加的"地理标签"，帮助 Server 判断客户端的网络拓扑位置。当 Relay 配置或客户端位置发生变化时，Option 82 信息会随之改变。
  
- **排查方法**：
  - 处理 Duplicate IP 问题的推荐排查顺序：**① Server 租约行为**（是否提前删除、是否 NACK）→ **② 报文中的 Option 82 / GIADDR**（是否子网漂移）→ **③ 客户端对 NACK 的处理逻辑**（是否正确重新获取 IP）。
  - DHCP Server 三重日志交叉验证法：**Audit Log**（事件时间线）+ **ETL 日志**（Server 内部决策逻辑）+ **网络抓包**（实际报文内容），三者结合才能还原完整行为链。
  
- **流程改进**：
  - 在复杂网络（多 VLAN / EVPN / 双网卡）环境中部署 DHCP 时，应同时验证客户端对 DHCPNAK 的处理是否符合协议标准（RFC 2131）。
  - 不要只盯着 DHCP Server——"Server 看上去像背锅侠，但实际上逻辑正常；真正的问题可能在客户端"是此类问题的典型场景。
  
- **预防措施**：
  - 在多 VLAN / 跨子网环境中启用 `DhcpGlobalFlagForSubnetChangeDHCPRequest` 时，应确保所有 DHCP 客户端（尤其是非 Windows 客户端如 Linux / IoT 设备）能正确处理 DHCPNAK。
  - 建议在部署前对各类客户端进行 NACK 响应测试。

## 7. 参考文档 (References)

- [DHCP Server Overview — Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-top) — Windows Server DHCP 服务概述
- [DHCP Failover — Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-failover) — DHCP 故障转移配置说明
- [RFC 3046 — DHCP Relay Agent Information Option](https://datatracker.ietf.org/doc/html/rfc3046) — DHCP Option 82 标准定义

---
---

# Case Summary: DHCP Duplicate IP — Option 82 Subnet Mismatch Causing Premature Lease Deletion and Reassignment

**Product/Service:** Windows Server 2022 — DHCP Server Role

---

## 1. Symptoms

- Multiple devices in the customer's environment reported **Duplicate IP** alerts — the same IP address was simultaneously in use by two different devices, causing intermittent business disruptions.
- Querying leases on the DHCP Server revealed that IP leases were being **deleted before their expiration time**, and the same IP was subsequently reassigned to another device.
  - Example: `Get-DhcpServerv4Lease -IPAddress 10.20.232.99` showed the lease at 12:11:34 with expiry ~12:23:17, but by 12:11:45 the lease had disappeared; by 12:14:51 the IP was assigned to a different device.
- The issue occurred frequently across multiple IP ranges.

## 2. Background / Environment

- **DHCP Server**: Windows Server 2022
- **DHCP Failover**: Configured in **Load Balance** mode
- **MCLT (Maximum Client Lead Time)**: 15 minutes
- **Lease Duration**: 4 hours
- **Registry Setting**: `DhcpGlobalFlagForSubnetChangeDHCPRequest = 1` (located at `HKLM\SYSTEM\CurrentControlSet\Services\DhcpServer\Parameters`)
- **Clients**: Linux devices with dual NICs on different VLANs
- **Network Architecture**: Cross-subnet deployment with DHCP Relay Agent forwarding; Relay configured with Option 82

## 3. Investigation & Troubleshooting

### Round 1: Why Are Leases Disappearing Early?

1. **Action**: Repeatedly queried the lease status for affected IPs (e.g., `10.20.232.99`) using `Get-DhcpServerv4Lease` on the DHCP Server.
2. **Finding**:
   - At 12:11:34 — lease existed with expiry ~12:23:17
   - At 12:11:45 — lease no longer present
   - At 12:14:51 — IP assigned to a different device
3. **Conclusion**: The DHCP Server proactively deleted the lease before its expiration and reassigned the IP. Need to determine if the client released it or the server acted on its own.

### Round 2: Did the Client Release the IP?

1. **Action**: Captured network traffic during the problem window, filtering for `DHCP Message Type = DHCP Release`.
2. **Finding**:
   - **No** DHCPRELEASE packets were observed from the client
   - The IP was still active and in use on the client machine
3. **Conclusion**: The client did not actively release the IP. The lease deletion was initiated by the DHCP Server itself.

### Round 3: DHCP Audit Log Timeline Reconstruction

1. **Action**: Reviewed DHCP audit logs to reconstruct the full event chain for affected IPs.
2. **Finding**: A consistent pattern emerged:
   - Client initially obtained an IP (Assign), MCLT of 15 minutes
   - At the half-lease mark (~7.5 minutes), the client attempted to renew (DHCPREQUEST)
   - DHCP Server responded with **NACK** (DHCPNAK)
   - Shortly after, the same IP was assigned to another device

   Audit log excerpt:
   ```
   10,12/08/25,11:04:12,Assign,10.20.232.99,,AABBCCDD1122,...
   15,12/08/25,11:11:42,NACK,10.20.232.99,,AABBCCDD1122,...
   15,12/08/25,11:15:27,NACK,10.20.232.99,,AABBCCDD1122,...
   15,12/08/25,11:17:19,NACK,10.20.232.99,,AABBCCDD1122,...
   10,12/08/25,11:18:15,Assign,10.20.232.99,,AABBCCDD1122,...
   ```
3. **Conclusion**: Clear behavioral chain — client renew → server NACK → server delete old lease → IP reassigned. Need to investigate why the NACK was issued.

### Round 4: Root Cause of NACK — Option 82 Subnet Mismatch

1. **Action**: Analyzed network captures and DHCP Server ETL logs from the problem window, focusing on Option 82 information in DHCPREQUEST packets.
2. **Finding**:
   - The client originally obtained IP `10.20.225.63` belonging to Subnet A
   - During renewal, the DHCPREQUEST packet's **Option 82 indicated Subnet B** (`203.0.113.0`)
   - DHCP Server ETL log clearly recorded:
     ```
     REQUEST for address 10.20.225.63
     REQUEST from relay agent: 198.51.100.30
     Interface subnet = 198.51.100.30
     Mismatch in ClientSubnet determined from request message
       (option 82 suboption 5 section) 203.0.113.0 and present in database 203.0.113.0
     DhcpDoDynDnsCheckDelete() - Deleting record 10.20.225.63
     Invalid DHCPREQUEST for 10.20.225.63 Nack'd.
     ```
3. **Conclusion**:
   - The client held a Subnet A IP but its renewal request (via Option 82) claimed it was on Subnet B
   - The DHCP Server detected the subnet mismatch and treated it as a "subnet change"
   - With `DhcpGlobalFlagForSubnetChangeDHCPRequest = 1` enabled, the server executed: **delete old lease → return NACK → expect client to re-DISCOVER**

### Round 5: Registry Behavior Validation — By Design

1. **Action**: Verified the DHCP Server registry setting at `HKLM\SYSTEM\CurrentControlSet\Services\DhcpServer\Parameters\DhcpGlobalFlagForSubnetChangeDHCPRequest`.
2. **Finding**: Value was **1**. When enabled:
   - If a DHCPREQUEST's Option 82 indicates the client has changed subnets
   - The server will: locate the old lease → delete it → return NACK → force the client to re-DISCOVER
3. **Conclusion**: The DHCP Server's lease deletion + NACK behavior is **By Design** — an enhanced behavior to prevent clients from retaining old-subnet IPs after a subnet change. Server-side logic was completely correct.

### Round 6: Client Failed to Properly Handle NACK

1. **Action**: Confirmed with the customer that the client actually migrated from Subnet A to Subnet B (Scenario 2: genuine subnet change due to dual NIC configuration).
2. **Finding**:
   - Per DHCP protocol (RFC 2131): after receiving NACK, the client should **discard the current IP → initiate a new DHCPDISCOVER**
   - Actual behavior: the client **did not re-DISCOVER** after receiving NACK and continued using the reclaimed IP
   - Meanwhile, the server legitimately assigned the same IP to another device → **Duplicate IP**
3. **Conclusion**: The root cause was on the client side — the Linux device's dual-NIC cross-VLAN configuration caused improper handling of DHCPNAK, violating protocol expectations.

## 4. Blockers & How They Were Resolved

| Blocker | Impact | How It Was Resolved |
|---------|--------|---------------------|
| Initial suspicion pointed at DHCP Server allocation anomaly; required multiple rounds of log analysis to identify Option 82 as the trigger | Extended troubleshooting time | Cross-validated using Audit Log + ETL logs + network capture to reconstruct the complete event chain |
| Needed to confirm whether the client truly changed subnets (Scenario 1: Relay misconfiguration vs. Scenario 2: actual client migration) | Could not determine whether Relay config or client movement was the cause | Confirmed with customer about actual network topology and client physical location changes |
| Client was a Linux device with non-standard DHCP implementation | Unable to directly debug the client's protocol stack | After confirming client-side issue, handed off to the customer to troubleshoot with the device vendor |

## 5. Root Cause & Resolution

### Root Cause

**Duplicate IP caused by multiple contributing factors:**

1. **Client (Linux, dual NIC) underwent subnet migration**: The client moved from Subnet A to Subnet B; its renewal request's Option 82 carried Subnet B information.
2. **DHCP Server responded correctly (By Design)**: With `DhcpGlobalFlagForSubnetChangeDHCPRequest = 1` enabled, the server detected the subnet mismatch, deleted the old lease, and returned NACK — expecting the client to re-DISCOVER.
3. **Client failed to properly handle NACK**: The Linux client did not discard the old IP or re-DISCOVER after receiving NACK, continuing to use the reclaimed IP.
4. **IP conflict occurred**: The reclaimed IP was reassigned by the server to another device, resulting in two devices using the same IP simultaneously.

### Resolution

1. Confirmed DHCP Server behavior was correct — `DhcpGlobalFlagForSubnetChangeDHCPRequest = 1` is a necessary and proper configuration in this environment.
2. Focused the issue on the client side: Linux device dual-NIC configuration caused DHCP protocol handling anomalies.
3. Customer engaged their Linux device vendor/team to investigate and fix the client's DHCP behavior (ensuring proper IP discard and re-DISCOVER after receiving NACK).

### Workaround

Temporary mitigations while the client-side fix was pending:
- Configure **DHCP Reservations** for frequently affected devices to prevent reclaimed IPs from being reassigned.
- Shorten lease duration to reduce the conflict window.

## 6. Lessons Learned

- **Technical Knowledge**:
  - `DhcpGlobalFlagForSubnetChangeDHCPRequest` is a critical enhanced-behavior registry key. When enabled, the DHCP Server proactively deletes leases and NACKs renewal requests where Option 82 indicates a subnet change. This is especially important in EVPN/VXLAN/multi-VLAN environments.
  - DHCP Option 82 serves as a "geographic tag" added by the Relay Agent, helping the Server determine the client's network topology position. When the Relay configuration or client location changes, Option 82 information changes accordingly.

- **Troubleshooting Methodology**:
  - Recommended investigation order for Duplicate IP issues: **① Server lease behavior** (premature deletion? NACK?) → **② Option 82 / GIADDR in packets** (subnet drift?) → **③ Client NACK handling** (does it properly re-acquire an IP?).
  - DHCP Server triple-log cross-validation: **Audit Log** (event timeline) + **ETL log** (server internal decision logic) + **Network capture** (actual packet content) — all three combined provide the complete behavioral chain.

- **Process Improvement**:
  - When deploying DHCP in complex networks (multi-VLAN / EVPN / dual-NIC), verify that clients properly handle DHCPNAK per protocol standards (RFC 2131).
  - Don't only look at the DHCP Server — "The server appears to be the scapegoat, but its logic is correct; the real problem may be on the client side" is a classic pattern in these types of issues.

- **Prevention**:
  - When enabling `DhcpGlobalFlagForSubnetChangeDHCPRequest` in multi-VLAN / cross-subnet environments, ensure all DHCP clients (especially non-Windows clients such as Linux / IoT devices) can correctly handle DHCPNAK.
  - Recommend NACK response testing for various client types before deployment.

## 7. References

- [DHCP Server Overview — Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-top) — Windows Server DHCP service overview
- [DHCP Failover — Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-failover) — DHCP failover configuration guide
- [RFC 3046 — DHCP Relay Agent Information Option](https://datatracker.ietf.org/doc/html/rfc3046) — DHCP Option 82 standard definition
