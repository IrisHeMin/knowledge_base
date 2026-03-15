---
layout: post
title: "Azure Stack HCI 部署卡在存储网络配置 — NIC 命名错误导致 IP 分配到错误网卡"
date: 2024-12-17
categories: [Azure-Stack-HCI, Networking]
tags: [azure-stack-hci, storage-nic, pktmon, arp, packet-drop, nic-naming, cable-mapping, mellanox, connectx-5, cluster-validation]

---

# Case Summary: Azure Stack HCI 部署卡在存储网络配置 — 通过 PKTMON 逐层追踪发现 NIC 命名错误导致 IP 分配到错误网卡

**Product/Service:** Azure Stack HCI — 存储网络部署

---

## 1. 症状 (Symptoms)

- 客户在 Azure Stack HCI 环境部署过程中卡在错误：**"Configure the host network storage network failed"**
- 部署无法继续推进，存储网络始终无法成功配置。

## 2. 背景 (Background / Environment)

- **环境**：Azure Stack HCI，双节点集群
- **网络配置**（每个节点）：
  - 1 个 Management vNIC（基于 SET Teaming）
  - 2 块物理存储网卡：**NIC1** 和 **NIC2**（Mellanox ConnectX-5）
- **存储网络拓扑**：
  - Node1 NIC1 ↔ Node2 NIC1：同 VLAN，直连线缆
  - Node1 NIC2 ↔ Node2 NIC2：同 VLAN，直连线缆
- **IP 分配方式**：HCI 部署过程中通过下拉列表选择 NIC 名称来分配 IP，非自动分配
- 网络配置与已知正常运行的 HCI 环境一致，唯一差异是存储网卡之间不通

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

### 第一轮：集群验证报告 — 确认存储网卡不通

1. **做了什么**：检查 Cluster Validation Report。
2. **发现了什么**：报告显示两个节点之间的存储网卡网络连接**不可达**。
3. **得出了什么结论**：存储网络存在基础连通性问题，需要进一步从网络层排查。

### 第二轮：基础连通性测试 — Ping 超时

1. **做了什么**：
   - 检查网络抓包：存储网卡之间无心跳包发送
   - 在存储网卡之间进行 Ping 测试（已确认 ICMP 已允许）
2. **发现了什么**：Ping 请求超时（Request Timeout），完全不通。
3. **得出了什么结论**：存储网卡之间的二层/三层连通性都有问题。需要使用更底层的工具定位。

### 第三轮：PKTMON 抓包（pcapng 格式）— 看似 ARP 请求未到达

1. **做了什么**：在两个节点上使用 PKTMON 抓包，指定源/目标 IP 进行 Ping 测试：
   ```
   pktmon start --capture --etw --file-size 2048 --file-name C:\pktmoncapturenew.etl
   ping 10.71.1.125 -S 10.71.1.69
   ```
   将 PKTMON 日志转换为 **pcapng** 格式，使用 Wireshark 过滤 ARP。
2. **发现了什么**：
   - Node1 发出了 ARP 请求，但未收到 ARP 响应（因此无法获取 Node2 NIC1 的 MAC 地址，后续 ICMP 无法发送）
   - 在 pcapng 中过滤 ARP，**Node2 上看不到收到 ARP 请求**
3. **得出了什么结论**：初步看起来 ARP 请求在传输过程中丢失了。但考虑到存储网卡是**直连线缆**（没有中间交换机），理论上不应该有中间丢包。需要换个分析方式。

### 第四轮：PKTMON 转为 TXT 格式 — 关键发现：包被另一块网卡收到并丢弃

1. **做了什么**：将 PKTMON 日志转换为 **TXT 文本格式**（`pktmon convert xx.etl`），直接阅读底层日志。
2. **发现了什么**：

   **Node1 NIC1 发出的 ARP 请求**：
   ```
   AA-BB-CC-DD-E1-3A > FF-FF-FF-FF-FF-FF, ethertype ARP (0x0806),
   length 42: Request who-has 10.71.1.125 tell 10.71.1.69
   ```

   **Node2 收到了 ARP 请求，但被 DROP 了**：
   ```
   [PktMon] Drop: PktGroupId 10696049115005232, PktNumber 1,
   Appearance 1, Direction Rx, Type Ethernet,
   Component 2, Filter 0, DropReason 228, DropLocation 0xE000100F
   Drop: AA-BB-CC-DD-E1-07 > FF-FF-FF-FF-FF-FF, ethertype ARP (0x0806),
   length 56: Request who-has 10.71.1.69 tell 10.71.1.125
   ```

   **Component 2 的身份确认**：
   ```
   Component 2, Type Miniport, Name mlx5.sys,
   Mellanox ConnectX-5 Adapter #2
   Physical Address = AA-BB-CC-DD-E1-06
   ```

   对应 `ipconfig` 信息：
   ```
   Ethernet adapter Storage2:
     Description: Mellanox ConnectX-5 Adapter #2
     Physical Address: AA-BB-CC-DD-E1-06
     IPv4 Address: 10.71.2.219
   ```

3. **得出了什么结论**：**关键发现** — ARP 请求是被 **Node2 的 NIC2（Storage2）** 收到并丢弃的，而不是预期中的 **Node2 NIC1**！如果线缆确实是 Node1NIC1↔Node2NIC1 直连，那 Node2NIC2 不可能收到这个包。

### 第五轮：反向验证 — 交叉网卡丢包现象

1. **做了什么**：检查从 Node2 NIC1 → Node1 NIC1 方向的 PKTMON 日志。
2. **发现了什么**：同样的现象——包被 **Node1 的 NIC2** 丢弃（而非预期的 Node1 NIC1 收到）。
3. **得出了什么结论**：两个方向都出现了**交叉网卡收包**现象：
   - 发给 NIC1 的包被 NIC2 收到
   - 发给 NIC2 的包被 NIC1 收到
   
   这强烈暗示**线缆插反**或 **IP 分配到了错误的网卡**。

### 第六轮：排除线缆插反 — 确认是 NIC 命名错误

1. **做了什么**：建议客户交换 Node2 上 NIC1 和 NIC2 的线缆插头。
2. **发现了什么**：客户反馈——在准备网络时已多次尝试不同的插线方式，**只有当前配置才能让端口灯亮起并显示已连接**。其他插法都不亮灯。
3. **得出了什么结论**：
   - NIC 的 LED 灯亮说明物理链路层是通的，**线缆的物理连接是正确的**
   - 真实的物理连接实际上是：**Node1 NIC1 ↔ Node2 NIC2**，**Node1 NIC2 ↔ Node2 NIC1**
   - 但客户在 HCI 部署时，根据 NIC 名称在下拉列表中选择了 NIC 来分配 IP
   - **客户命名 NIC 时搞反了**：物理上连接到 VLAN1 的 NIC 被命名为了 NIC2，物理上连接到 VLAN2 的 NIC 被命名为了 NIC1
   - 结果：VLAN1 的 IP 被分配到了实际连在 VLAN2 上的网卡，反之亦然

   **实际线缆连接**（正确的物理连接）：
   ```
   Node1 "NIC1"(实际连VLAN2) ←线缆→ Node2 "NIC2"(实际连VLAN2) ✓ 物理正确
   Node1 "NIC2"(实际连VLAN1) ←线缆→ Node2 "NIC1"(实际连VLAN1) ✓ 物理正确
   ```

   **IP 分配**（基于错误的 NIC 名称）：
   ```
   Node2 "NIC1"(物理在VLAN1) ← 被分配了 VLAN2 的 IP  ✗ 错误！
   Node2 "NIC2"(物理在VLAN2) ← 被分配了 VLAN1 的 IP  ✗ 错误！
   ```

### 第七轮：修复验证

1. **做了什么**：客户在一个节点上手动交换 NIC1 和 NIC2 的 IP 地址。
2. **发现了什么**：交换 IP 后，存储网卡之间的网络连通性**立即恢复**。
3. **得出了什么结论**：根因确认为 NIC 命名错误导致 IP 分配到错误网卡。修复后 HCI 部署继续推进。

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| PKTMON 转 pcapng 格式在 Wireshark 中看不到 Node2 收到 ARP 请求，造成误导 | 一度认为 ARP 请求在传输中丢失，而实际上是被收到后丢弃 | 改用 PKTMON 转 TXT 文本格式，直接看底层日志发现 Drop 记录，包含具体的 Component ID 和 DropReason |
| 客户认为线缆插法已经多次尝试，当前是唯一正确的物理连接 | 排除了"换线缆"的简单解决方案 | 转变思路：物理连接是对的，但 NIC 的逻辑命名/IP 分配是错的 |
| 存储网卡直连无中间设备，排除了传统的"中间丢包"可能性 | 缩小了排查范围但增加了困惑——直连为何还会丢包？ | 通过 PKTMON 的 Component ID 精确定位到是"错误的 NIC"收到了包 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

**客户在 HCI 部署前对存储网卡的命名出现了错误，导致 IP 地址被分配到了物理上连接到另一个 VLAN 的网卡。**

具体机制：
1. 两个节点的存储网卡通过直连线缆互联，物理连接本身是正确的
2. 但客户在命名网卡时，将物理上连接到 VLAN1 的网卡命名为 "NIC2"，将连接到 VLAN2 的网卡命名为 "NIC1"（其中一个节点搞反了）
3. HCI 部署过程中根据 NIC 名称分配 IP → VLAN1 的 IP 被分配到了物理上在 VLAN2 的网卡
4. 结果：Node1 NIC1（IP 在 VLAN1）通过线缆连到 Node2 NIC2（IP 在 VLAN2），IP 地址不在同一子网
5. ARP 请求到达错误的 NIC，该 NIC 的 IP 与请求的目标不匹配 → ARP 被丢弃 → 无法建立二层邻接 → 完全不通

### Resolution

在其中一个节点上**交换 NIC1 和 NIC2 的 IP 地址**，使 IP 地址与物理连接的 VLAN 匹配：
1. 记录当前 NIC1 和 NIC2 的 IP 配置
2. 将 NIC1 的 IP 设置到 NIC2 上
3. 将 NIC2 的 IP 设置到 NIC1 上
4. 验证存储网卡之间的 Ping 恢复正常
5. 继续 HCI 部署流程

### Workaround

如果不想交换 IP，也可以物理交换线缆（但客户环境中只有当前插法灯才亮，说明端口速率/类型可能有匹配要求）。

## 6. 经验教训 (Lessons Learned)

- **技术知识**：
  - **PKTMON TXT 格式 vs pcapng 格式的差异**：将 PKTMON 导出为 pcapng 在 Wireshark 中分析时，**被丢弃的包可能不会显示在 pcapng 中**。而转换为 TXT 文本格式后可以看到完整的 Drop 记录，包括关键信息：`Component ID`（哪个网卡丢的）、`DropReason`（为什么丢的）、`Direction Rx`（是收到后丢弃，不是中途丢失）。在排查丢包问题时，**PKTMON TXT 是比 pcapng 更可靠的分析格式**。
  - **ARP 交叉收包是 NIC 映射错误的典型信号**：当直连环境中，发给 NIC1 的包被 NIC2 收到，这意味着物理连接和逻辑 NIC 标识之间存在不匹配——要么线缆插反，要么 NIC 命名/IP 分配反了。
  - **HCI 部署中的 NIC 命名重要性**：Azure Stack HCI 部署流程中，IP 地址是根据 NIC 名称分配的。如果 NIC 命名与物理连接不匹配，会导致整个存储网络配置失败。在部署前必须验证 NIC 名称与物理端口的对应关系。

- **排查方法**：
  - **PKTMON 排查丢包的标准流程**：① `pktmon start --capture` 抓包 → ② 先转 pcapng 看整体流量 → ③ 如果怀疑丢包，**务必转 TXT 格式查看 Drop 记录** → ④ 根据 `Component ID` 定位是哪个组件丢的包 → ⑤ 根据 `DropReason` 分析原因。
  - **直连环境的排查思路**：当两个设备通过直连线缆互联时，如果出现"包发了但对方没收到"，首先排除的不是"中间丢包"（因为没有中间设备），而应该检查：① 包是否被**另一块网卡**收到了？② NIC 命名/IP 分配是否与物理连接一致？
  - **LED 灯状态的利用**：NIC 端口灯亮/灭可以帮助判断物理连接的正确性。如果只有特定插法灯才亮，说明物理连接是确定的，问题在逻辑配置层面。

- **预防措施**：
  - HCI 部署前，应使用 **物理手段**（如端口灯状态、拔插线缆测试）确认每个 NIC 名称对应的物理端口，再进行 IP 分配。
  - 建议在 NIC 命名前，先通过 `Get-NetAdapter` 查看 MAC 地址和链路状态，与物理标签/端口对照，避免命名错误。
  - 部署后立即进行存储网卡之间的 Ping 测试，尽早发现连通性问题。

## 7. 参考文档 (References)

- [Network ATC — Azure Stack HCI](https://learn.microsoft.com/en-us/azure-stack/hci/deploy/network-atc) — Azure Stack HCI 网络 ATC 部署指南
- [Packet Monitor (Pktmon) — Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) — PKTMON 网络诊断工具使用说明
- [Physical Network Requirements — Azure Stack HCI](https://learn.microsoft.com/en-us/azure-stack/hci/concepts/physical-network-requirements) — Azure Stack HCI 物理网络要求
- [Validate an Azure Stack HCI Cluster](https://learn.microsoft.com/en-us/azure-stack/hci/deploy/validate) — 集群验证流程

---
---

# Case Summary: Azure Stack HCI Deployment Stuck on Storage Network Configuration — NIC Naming Error Caused IP Assignment to Wrong Adapter

**Product/Service:** Azure Stack HCI — Storage Network Deployment

---

## 1. Symptoms

- Customer's Azure Stack HCI environment deployment was stuck at the error: **"Configure the host network storage network failed"**
- Deployment could not proceed; storage network configuration consistently failed.

## 2. Background / Environment

- **Environment**: Azure Stack HCI, two-node cluster
- **Network configuration** (per node):
  - 1 Management vNIC (based on SET teaming)
  - 2 physical storage NICs: **NIC1** and **NIC2** (Mellanox ConnectX-5)
- **Storage network topology**:
  - Node1 NIC1 ↔ Node2 NIC1: Same VLAN, direct cable connection
  - Node1 NIC2 ↔ Node2 NIC2: Same VLAN, direct cable connection
- **IP assignment method**: During HCI deployment, IPs were assigned by selecting NICs from a dropdown list based on NIC names (not automatic)
- Network configuration was identical to a known-working HCI environment; the only difference was storage NICs being unreachable

## 3. Investigation & Troubleshooting

### Round 1: Cluster Validation Report — Storage NICs Unreachable

1. **Action**: Reviewed the Cluster Validation Report.
2. **Finding**: Report showed network connectivity between storage NICs across the two nodes was **not reachable**.
3. **Conclusion**: Fundamental connectivity issue with the storage network. Needed deeper network-layer investigation.

### Round 2: Basic Connectivity Test — Ping Timeout

1. **Action**:
   - Examined network traces: no heartbeat traffic on storage NICs
   - Performed ping tests between storage NICs (ICMP confirmed allowed)
2. **Finding**: Ping request timed out — complete failure to communicate.
3. **Conclusion**: Both L2 and L3 connectivity were broken between storage NICs. Lower-level diagnostic tools were needed.

### Round 3: PKTMON Capture (pcapng) — ARP Requests Seemingly Not Arriving

1. **Action**: Ran PKTMON on both nodes, then performed directed ping tests:
   ```
   pktmon start --capture --etw --file-size 2048 --file-name C:\pktmoncapturenew.etl
   ping 10.71.1.125 -S 10.71.1.69
   ```
   Converted PKTMON logs to **pcapng** format and filtered for ARP in Wireshark.
2. **Finding**:
   - Node1 sent ARP requests but received no ARP responses (thus couldn't resolve Node2 NIC1's MAC; subsequent ICMP couldn't be sent)
   - Filtering ARP in pcapng, **Node2 appeared to never receive the ARP request**
3. **Conclusion**: Initially appeared that ARP requests were lost in transit. But since storage NICs were **direct-cabled** (no intermediate switches), mid-path packet loss should be impossible. Needed a different analysis approach.

### Round 4: PKTMON Text Format — Critical Discovery: Packet Received by WRONG NIC and Dropped

1. **Action**: Converted PKTMON logs to **TXT text format** (`pktmon convert xx.etl`) to examine raw low-level logs.
2. **Finding**:

   **Node1 NIC1 sent the ARP request**:
   ```
   AA-BB-CC-DD-E1-3A > FF-FF-FF-FF-FF-FF, ethertype ARP (0x0806),
   length 42: Request who-has 10.71.1.125 tell 10.71.1.69
   ```

   **Node2 received the ARP request but DROPPED it**:
   ```
   [PktMon] Drop: PktGroupId 10696049115005232, PktNumber 1,
   Appearance 1, Direction Rx, Type Ethernet,
   Component 2, Filter 0, DropReason 228, DropLocation 0xE000100F
   Drop: AA-BB-CC-DD-E1-07 > FF-FF-FF-FF-FF-FF, ethertype ARP (0x0806),
   length 56: Request who-has 10.71.1.69 tell 10.71.1.125
   ```

   **Component 2 identification**:
   ```
   Component 2, Type Miniport, Name mlx5.sys,
   Mellanox ConnectX-5 Adapter #2
   Physical Address = AA-BB-CC-DD-E1-06
   → This is Node2 NIC2 (Storage2), IP: 10.71.2.219
   ```

3. **Conclusion**: **Critical finding** — The ARP request was received and dropped by **Node2 NIC2 (Storage2)**, NOT the expected **Node2 NIC1**! If the cable was truly Node1NIC1 ↔ Node2NIC1 direct-connected, Node2NIC2 should never have received this packet.

### Round 5: Reverse Verification — Cross-NIC Packet Drop Pattern

1. **Action**: Examined PKTMON logs for the reverse direction: Node2 NIC1 → Node1 NIC1 ping.
2. **Finding**: Same pattern — packets were dropped by **Node1 NIC2** (not received by the expected Node1 NIC1).
3. **Conclusion**: Both directions showed **cross-NIC packet reception**:
   - Packets destined for NIC1 were received by NIC2
   - Packets destined for NIC2 were received by NIC1
   
   This strongly indicated either **cables plugged into wrong ports** or **IPs assigned to wrong NICs**.

### Round 6: Ruled Out Cable Swap — Confirmed NIC Naming Error

1. **Action**: Suggested the customer swap the cable connections between NIC1 and NIC2 on Node2.
2. **Finding**: Customer reported that during network preparation, they had tried multiple cable configurations, and **only the current configuration resulted in port LEDs lighting up and NICs showing connected**. All other configurations showed no link.
3. **Conclusion**:
   - NIC LED indicators confirmed the **physical cable connections were correct**
   - The actual physical connections were: **Node1 NIC1 ↔ Node2 NIC2**, **Node1 NIC2 ↔ Node2 NIC1**
   - During HCI deployment, the customer selected NICs by name from a dropdown to assign IPs
   - **The customer had named the NICs incorrectly on one node**: the NIC physically connected to VLAN1 was named "NIC2", and the NIC connected to VLAN2 was named "NIC1"
   - Result: VLAN1 IPs were assigned to the NIC physically on VLAN2, and vice versa

   **Actual cable connections** (physically correct):
   ```
   Node1 "NIC1"(physically on VLAN2) ←cable→ Node2 "NIC2"(physically on VLAN2) ✓
   Node1 "NIC2"(physically on VLAN1) ←cable→ Node2 "NIC1"(physically on VLAN1) ✓
   ```

   **IP assignment** (based on wrong NIC names):
   ```
   Node2 "NIC1"(physically on VLAN1) ← assigned VLAN2 IP  ✗ Wrong!
   Node2 "NIC2"(physically on VLAN2) ← assigned VLAN1 IP  ✗ Wrong!
   ```

### Round 7: Fix and Verification

1. **Action**: Customer manually swapped the IP addresses between NIC1 and NIC2 on one node.
2. **Finding**: After swapping IPs, storage NIC connectivity was **immediately restored**.
3. **Conclusion**: Root cause confirmed as NIC naming error leading to IP assignment on the wrong adapter. HCI deployment proceeded successfully after the fix.

## 4. Blockers & How They Were Resolved

| Blocker | Impact | How It Was Resolved |
|---------|--------|---------------------|
| PKTMON pcapng format in Wireshark didn't show Node2 receiving the ARP request, creating a misleading picture | Initially believed ARP requests were lost in transit, when they were actually received and dropped | Switched to PKTMON TXT text format, which revealed complete Drop records including Component ID and DropReason |
| Customer confirmed cables had been tried multiple ways; only the current physical connection worked (LED link up) | Eliminated "swap cables" as a simple fix | Shifted thinking: physical connections were correct, but the logical NIC naming/IP assignment was wrong |
| Direct-cable topology with no intermediate devices eliminated traditional "mid-path drop" possibilities | Narrowed scope but increased confusion — why would a direct connection drop packets? | PKTMON Component ID precisely identified that the "wrong NIC" received the packet, revealing the NIC mapping mismatch |

## 5. Root Cause & Resolution

### Root Cause

**The customer incorrectly named the storage NICs on one node before HCI deployment, causing IP addresses to be assigned to NICs that were physically connected to the wrong VLAN.**

Detailed mechanism:
1. Storage NICs on both nodes were direct-cabled; the physical connections were correct
2. However, the customer named the NICs incorrectly on one node: the NIC physically connected to VLAN1 was named "NIC2", and the NIC connected to VLAN2 was named "NIC1"
3. During HCI deployment, IP addresses were assigned based on NIC names → VLAN1 IPs went to a NIC physically on VLAN2
4. Result: Node1 NIC1 (VLAN1 IP) was cabled to Node2 NIC2 (VLAN2 IP) — IPs not in the same subnet
5. ARP requests arrived at the wrong NIC (mismatched IP/VLAN) → ARP dropped → no L2 adjacency → complete connectivity failure

### Resolution

**Swapped IP addresses between NIC1 and NIC2 on one node** to align IP assignments with physical cable connections:
1. Documented the current IP configurations of NIC1 and NIC2
2. Assigned NIC1's IP to NIC2
3. Assigned NIC2's IP to NIC1
4. Verified ping connectivity between storage NICs restored
5. Resumed HCI deployment successfully

### Workaround

Alternatively, the NIC names could be corrected to match the physical connections, followed by redeployment.

## 6. Lessons Learned

- **Technical Knowledge**:
  - **PKTMON TXT format vs pcapng format**: When converting PKTMON to pcapng for Wireshark analysis, **dropped packets may not appear in the pcapng file**. Converting to TXT text format reveals complete Drop records including: `Component ID` (which NIC dropped it), `DropReason` (why it was dropped), and `Direction Rx` (received then dropped, not lost in transit). **PKTMON TXT is the more reliable format for diagnosing packet drops**.
  - **Cross-NIC packet reception is a classic signal of NIC mapping errors**: In direct-cable environments, when packets sent to NIC1 are received by NIC2, it means there's a mismatch between physical connections and logical NIC identifiers — either cables are swapped or NIC naming/IP assignment is reversed.
  - **NIC naming criticality in HCI deployment**: Azure Stack HCI deployment assigns IP addresses based on NIC names selected from a dropdown. If NIC names don't match physical port connections, the entire storage network configuration will fail. NIC-to-physical-port mapping must be verified before deployment.

- **Troubleshooting Methodology**:
  - **PKTMON packet drop investigation workflow**: ① `pktmon start --capture` → ② Convert to pcapng for overall traffic analysis → ③ If drops are suspected, **always convert to TXT format for Drop records** → ④ Use `Component ID` to identify which component dropped the packet → ⑤ Analyze `DropReason` for the cause.
  - **Direct-cable environment troubleshooting**: When two devices are direct-cabled, "packet sent but not received" should NOT lead to investigating "mid-path drops" (no intermediate devices). Instead check: ① Was the packet received by the **wrong NIC**? ② Do NIC names/IP assignments match the physical connections?
  - **LED link status as a diagnostic tool**: NIC port LED indicators (link up/down) help determine physical connection correctness. If only one cable configuration lights up the LEDs, the physical connection is deterministic — any remaining issue is in the logical configuration layer.

- **Prevention**:
  - Before HCI deployment, use **physical methods** (LED status, cable plug/unplug tests) to confirm which NIC name maps to which physical port, then assign IPs accordingly.
  - Recommended: run `Get-NetAdapter` to view MAC addresses and link states, cross-reference with physical labels/ports before naming NICs.
  - Immediately after deployment, perform ping tests between storage NICs to catch connectivity issues early.

## 7. References

- [Network ATC — Azure Stack HCI](https://learn.microsoft.com/en-us/azure-stack/hci/deploy/network-atc) — Azure Stack HCI Network ATC deployment guide
- [Packet Monitor (Pktmon) — Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) — PKTMON network diagnostic tool documentation
- [Physical Network Requirements — Azure Stack HCI](https://learn.microsoft.com/en-us/azure-stack/hci/concepts/physical-network-requirements) — Azure Stack HCI physical network requirements
- [Validate an Azure Stack HCI Cluster](https://learn.microsoft.com/en-us/azure-stack/hci/deploy/validate) — Cluster validation process
