---
layout: post
title: "Deep Dive: TCP/IP 数据包流程精讲 — 进阶篇 (Senior)"
date: 2026-03-10
categories: [Knowledge, Networking]
tags: [tcp-ip, networking, routing, arp, subnet, packet-flow, ethernet, ip-header, routing-table, senior]
type: "deep-dive"
---

# Deep Dive: TCP/IP 数据包流程精讲 — 进阶篇 (Senior Level)

**Topic:** TCP/IP 同网段与跨网段数据包流程 — 进阶解析  
**Category:** Networking  
**Level:** 中级 (Senior)  
**Last Updated:** 2026-03-10  
**系列导航：** [Junior 入门篇]({{ site.baseurl }}{% post_url 2026-03-10-tcpip-junior-packet-flow-and-routing-basics %}) → Senior (本篇) → [Master 专家篇]({{ site.baseurl }}{% post_url 2026-03-10-tcpip-master-packet-routing-internals %})

---

## 📌 中文版

---

### 1. 概述 (Overview)

在 Junior 篇中，我们用"快递比喻"理解了同网段和跨网段通信的基本原理。现在，我们要**撕开比喻的外衣**，看看数据包在每一步**真正长什么样**：

- Ethernet 帧头里的字段是什么？
- IP 包头里哪些字段会变、哪些不变？
- ARP 请求/回复的报文结构是什么？
- 路由器的路由表怎么做决策？
- TTL 是怎么防止"快递无限转圈"的？

本篇会带你**拿着放大镜**看数据包在网络中每一跳的完整变化过程。

---

### 2. 核心概念深入 (Core Concepts — Deeper)

#### 2.1 OSI 模型与封装 — "套信封"

TCP/IP 通信本质上是**层层套信封**的过程：

```
应用层数据
  │
  ▼ 加 TCP/UDP 头
┌──────────────────────────────┐
│ TCP Header │   Application Data │  ← 第四层：传输层 Segment
└──────────────────────────────┘
  │
  ▼ 加 IP 头
┌──────────────────────────────────────┐
│ IP Header │ TCP Header │ App Data    │  ← 第三层：网络层 Packet
└──────────────────────────────────────┘
  │
  ▼ 加 Ethernet 头 + 尾（FCS）
┌──────────────────────────────────────────────────┐
│ Eth Header │ IP Header │ TCP Header │ App Data │ FCS │  ← 第二层：数据链路层 Frame
└──────────────────────────────────────────────────┘
```

**关键原则**：每过一个路由器，**第二层（Ethernet 帧头）被剥离并重写**，但**第三层（IP 包头）基本不变**（TTL 递减除外）。

#### 2.2 Ethernet 帧结构

```
┌──────────────────────────────────────────────────────┐
│ Destination MAC │ Source MAC │ EtherType │ Payload │ FCS │
│    (6 bytes)    │ (6 bytes)  │ (2 bytes) │         │     │
└──────────────────────────────────────────────────────┘

EtherType 常见值：
  0x0800 = IPv4
  0x0806 = ARP
  0x86DD = IPv6
```

- **Destination MAC**：下一跳设备的 MAC（同网段是目标主机，跨网段是网关）
- **Source MAC**：发送方当前接口的 MAC
- **EtherType**：告诉接收方"信封里装的是什么类型的包"

#### 2.3 IPv4 包头结构（关键字段）

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |    DSCP/ECN   |         Total Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|    Fragment Offset      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  ★ TTL  |  ★ Protocol  |       Header Checksum              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    ★ Source IP Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 ★ Destination IP Address                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**关键字段解析：**

| 字段 | 大小 | 作用 | 跨网段时会变吗？ |
|------|------|------|-----------------|
| **Source IP** | 4 bytes | 发送方 IP | ❌ 不变（除非 NAT） |
| **Destination IP** | 4 bytes | 接收方 IP | ❌ 不变（除非 NAT） |
| **TTL** | 1 byte | 生存时间，每过一个路由器减 1 | ✅ 每跳减 1 |
| **Protocol** | 1 byte | 上层协议（TCP=6, UDP=17, ICMP=1） | ❌ 不变 |
| **Header Checksum** | 2 bytes | 头部校验和 | ✅ 每跳重算（因为 TTL 变了） |
| **Total Length** | 2 bytes | 整个 IP 包的长度 | ❌ 一般不变（除非分片） |
| **Identification/Flags/Fragment Offset** | 分片相关 | 处理 MTU 超限时的分片 | 可能变 |

#### 2.4 ARP 报文结构

```
┌──────────────────────────────────────────┐
│ Hardware Type: 1 (Ethernet)              │
│ Protocol Type: 0x0800 (IPv4)             │
│ Hardware Size: 6 (MAC = 6 bytes)         │
│ Protocol Size: 4 (IPv4 = 4 bytes)        │
│ Opcode: 1 (Request) 或 2 (Reply)        │
├──────────────────────────────────────────┤
│ Sender MAC:  AA:AA:AA:AA:AA:AA          │
│ Sender IP:   192.168.1.100              │
│ Target MAC:  00:00:00:00:00:00 (Request) │ ← Request 时目标 MAC 为全 0
│              BB:BB:BB:BB:BB:BB (Reply)   │ ← Reply 时填入实际 MAC
│ Target IP:   192.168.1.200              │
└──────────────────────────────────────────┘
```

**ARP Request**是**广播**发送的（目的 MAC = `FF:FF:FF:FF:FF:FF`），同网段的所有设备都能收到。  
**ARP Reply**是**单播**回复的（只发给请求者）。

#### 2.5 路由表 — 路由器的"导航地图"

```
C:\> route print
===========================================================================
Network Destination    Netmask          Gateway         Interface    Metric
---------------------------------------------------------------------------
0.0.0.0               0.0.0.0          192.168.1.1     192.168.1.100   25
10.0.0.0              255.255.255.0    192.168.1.254   192.168.1.100   20
192.168.1.0           255.255.255.0    On-link         192.168.1.100   10
===========================================================================
```

**路由表查找规则 — 最长前缀匹配 (Longest Prefix Match)**：

当要发送到 `10.0.0.50` 时：
1. 检查 `10.0.0.0/24` → 匹配！网关是 `192.168.1.254`
2. 检查 `0.0.0.0/0`（默认路由）→ 也匹配，但前缀更短
3. 选择**匹配最长**的那条 → `10.0.0.0/24`（/24 比 /0 更具体）

---

### 3. 工作原理 — 完整数据包追踪 (How It Works — Full Packet Trace)

#### 3.1 场景：完整的跨网段 HTTP 请求

```
实验环境：

  PC-A (192.168.1.100/24)                    Web Server (10.0.0.50/24)
  MAC: AA:AA:AA:AA:AA:AA                     MAC: SS:SS:SS:SS:SS:SS
  Gateway: 192.168.1.1                       Gateway: 10.0.0.1
       │                                          │
       │ port 1                          port 3   │
  ┌────┴────────────────┐          ┌───────────────┴────┐
  │    Switch-1          │          │    Switch-2         │
  └────┬────────────────┘          └───────────────┬────┘
       │ port 2                          port 4   │
       │                                          │
  ┌────┴──────────────────────────────────────────┴────┐
  │                    Router-R                         │
  │  eth0: 192.168.1.1  MAC: R1:R1:R1:R1:R1:R1        │
  │  eth1: 10.0.0.1     MAC: R2:R2:R2:R2:R2:R2        │
  │                                                     │
  │  路由表:                                            │
  │  192.168.1.0/24 → directly connected (eth0)        │
  │  10.0.0.0/24    → directly connected (eth1)        │
  └─────────────────────────────────────────────────────┘
```

#### Step 0: DNS 解析（假设已完成）

PC-A 已经知道 Web Server 的 IP 是 `10.0.0.50`。

#### Step 1: PC-A 判断目的地

```
PC-A 的计算过程：

  自己的 IP:       192.168.1.100
  自己的子网掩码:  255.255.255.0
  自己的网段:      192.168.1.100 AND 255.255.255.0 = 192.168.1.0

  目的 IP:         10.0.0.50
  目的网段:        10.0.0.50 AND 255.255.255.0 = 10.0.0.0

  192.168.1.0 ≠ 10.0.0.0 → 不同网段！→ 交给默认网关 192.168.1.1
```

#### Step 2: PC-A 发送 ARP Request（查网关 MAC）

假设 ARP 缓存为空，PC-A 需要先查网关 192.168.1.1 的 MAC。

```
=== ARP Request Frame（在线缆上的实际数据）===

Ethernet Header:
  Dst MAC:  FF:FF:FF:FF:FF:FF  (广播)
  Src MAC:  AA:AA:AA:AA:AA:AA  (PC-A)
  Type:     0x0806 (ARP)

ARP Payload:
  Opcode:      1 (Request)
  Sender MAC:  AA:AA:AA:AA:AA:AA
  Sender IP:   192.168.1.100
  Target MAC:  00:00:00:00:00:00  (未知，待查)
  Target IP:   192.168.1.1
```

Switch-1 收到这个广播帧后，从**所有端口**（除来源端口外）转发出去。

#### Step 3: Router 回复 ARP Reply

Router 的 eth0 接口 IP 是 192.168.1.1，匹配 ARP 请求。

```
=== ARP Reply Frame ===

Ethernet Header:
  Dst MAC:  AA:AA:AA:AA:AA:AA  (单播回给 PC-A)
  Src MAC:  R1:R1:R1:R1:R1:R1  (Router eth0)
  Type:     0x0806 (ARP)

ARP Payload:
  Opcode:      2 (Reply)
  Sender MAC:  R1:R1:R1:R1:R1:R1
  Sender IP:   192.168.1.1
  Target MAC:  AA:AA:AA:AA:AA:AA
  Target IP:   192.168.1.100
```

PC-A 收到后，更新 ARP 缓存：`192.168.1.1 → R1:R1:R1:R1:R1:R1`

#### Step 4: PC-A 构建完整的数据帧并发送

PC-A 构建 HTTP GET 请求，层层封装：

```
=== PC-A 发出的完整 Frame #1 ===

┌─── Layer 2: Ethernet ──────────────────────────────┐
│ Dst MAC:  R1:R1:R1:R1:R1:R1  ← 网关的 MAC！       │
│ Src MAC:  AA:AA:AA:AA:AA:AA                        │
│ Type:     0x0800 (IPv4)                            │
├─── Layer 3: IP ────────────────────────────────────┤
│ Version:  4                                        │
│ TTL:      128  ← Windows 默认                      │
│ Protocol: 6 (TCP)                                  │
│ Src IP:   192.168.1.100                            │
│ Dst IP:   10.0.0.50  ← 最终目的地！                │
├─── Layer 4: TCP ───────────────────────────────────┤
│ Src Port: 49152 (随机高端口)                        │
│ Dst Port: 80 (HTTP)                                │
│ SYN flag                                           │
├─── Layer 7: (SYN 包无应用数据) ────────────────────┤
└────────────────────────────────────────────────────┘
```

🔑 **关键观察**：  
- **Dst MAC = 网关 MAC**（不是 Web Server 的 MAC！）  
- **Dst IP = Web Server IP**（最终目的地）  
- 这就是"MAC 看下一跳，IP 看终点"的核心体现

#### Step 5: Switch-1 转发

Switch-1 查 MAC 地址表：
- `R1:R1:R1:R1:R1:R1` → port 2
- 从 port 2 转发帧 → Router eth0 接口收到

（Switch 完全不看 IP 地址，只看 MAC 地址！）

#### Step 6: Router 处理 — 拆帧、查路由、重新封帧

```
Router 的处理步骤：

1️⃣ 收到帧 → 检查 Dst MAC 是自己（R1:R1:R1:R1:R1:R1）→ 接受
2️⃣ 剥离 Ethernet 帧头 → 看 IP 包头
3️⃣ 检查 Dst IP: 10.0.0.50
4️⃣ 查路由表：
   10.0.0.0/24 → directly connected via eth1 → 从 eth1 发出
5️⃣ TTL 减 1：128 → 127（如果 TTL=0 则丢弃并返回 ICMP Time Exceeded）
6️⃣ 重算 IP Header Checksum（因为 TTL 变了）
7️⃣ 需要知道 10.0.0.50 的 MAC → ARP 查询（如果缓存没有）
8️⃣ 构建新的 Ethernet 帧，从 eth1 发出
```

#### Step 7: Router 发出新帧（如果 ARP 已缓存）

```
=== Router 发出的 Frame #2（从 eth1） ===

┌─── Layer 2: Ethernet（全新的！）───────────────────┐
│ Dst MAC:  SS:SS:SS:SS:SS:SS  ← Web Server 的 MAC   │
│ Src MAC:  R2:R2:R2:R2:R2:R2  ← Router eth1 的 MAC  │
│ Type:     0x0800 (IPv4)                             │
├─── Layer 3: IP（几乎不变）─────────────────────────┤
│ TTL:      127  ← 从 128 减到了 127                  │
│ Src IP:   192.168.1.100  ← 没变！                   │
│ Dst IP:   10.0.0.50      ← 没变！                   │
├─── Layer 4: TCP（完全不变）────────────────────────┤
│ Src Port: 49152                                     │
│ Dst Port: 80                                        │
│ SYN flag                                            │
└─────────────────────────────────────────────────────┘
```

**对比 Frame #1 和 Frame #2：**

| 字段 | Frame #1 (PC-A→Router) | Frame #2 (Router→Server) |
|------|------------------------|--------------------------|
| Dst MAC | R1:R1:R1:R1:R1:R1 (网关) | SS:SS:SS:SS:SS:SS (Server) |
| Src MAC | AA:AA:AA:AA:AA:AA (PC-A) | R2:R2:R2:R2:R2:R2 (Router eth1) |
| Dst IP | 10.0.0.50 | 10.0.0.50 (**不变**) |
| Src IP | 192.168.1.100 | 192.168.1.100 (**不变**) |
| TTL | 128 | 127 (**减 1**) |

#### Step 8: Switch-2 转发 → Web Server 收到

Switch-2 根据 `SS:SS:SS:SS:SS:SS` 转发到 Server 端口 → Server 收到 SYN → TCP 三次握手开始 → HTTP 请求/响应...

---

### 4. 多跳路由场景 (Multi-Hop Routing)

实际网络中，数据包往往要经过**多个路由器**：

```
PC-A ──→ Router-1 ──→ Router-2 ──→ Router-3 ──→ Server

帧在每一跳的变化：

Hop 1: PC-A → Router-1
  Eth Src MAC: PC-A          Eth Dst MAC: Router-1 (eth0)
  IP Src: PC-A               IP Dst: Server
  TTL: 128

Hop 2: Router-1 → Router-2
  Eth Src MAC: Router-1 (eth1)   Eth Dst MAC: Router-2 (eth0)
  IP Src: PC-A               IP Dst: Server        ← IP 不变
  TTL: 127                                         ← TTL 减 1

Hop 3: Router-2 → Router-3
  Eth Src MAC: Router-2 (eth1)   Eth Dst MAC: Router-3 (eth0)
  IP Src: PC-A               IP Dst: Server        ← IP 不变
  TTL: 126                                         ← TTL 再减 1

Hop 4: Router-3 → Server
  Eth Src MAC: Router-3 (eth1)   Eth Dst MAC: Server
  IP Src: PC-A               IP Dst: Server        ← IP 不变
  TTL: 125                                         ← TTL 再减 1
```

**tracert/traceroute 的工作原理就是利用 TTL：**
1. 发 TTL=1 的包 → 第一个路由器收到，TTL 减为 0，丢弃并回复 ICMP "Time Exceeded" → 你知道了第一跳
2. 发 TTL=2 的包 → 到达第二个路由器，TTL 减为 0 → 你知道了第二跳
3. 依此类推...

---

### 5. 关键配置与参数 (Key Configurations)

#### 5.1 Windows TCP/IP 配置

| 参数 | 查看命令 | 说明 | 配错的后果 |
|------|----------|------|-----------|
| IP Address | `ipconfig` | 设备的网络地址 | IP 冲突或无法通信 |
| Subnet Mask | `ipconfig` | 定义网段范围 | 错误判断同/跨网段 → 部分 IP 不通 |
| Default Gateway | `ipconfig` | 跨网段出口 | 无法访问外网 |
| DNS Server | `ipconfig /all` | 域名解析 | 能 ping IP 但打不开网页 |
| ARP Cache | `arp -a` | IP→MAC 映射缓存 | ARP 缓存中毒 → 安全风险 |
| Routing Table | `route print` | 路由决策依据 | 路由错误 → 流量走错路径 |
| TTL | 默认 128 (Windows) / 64 (Linux) | 防环和 tracert 依据 | 网络路径过长时包被丢弃 |

#### 5.2 ARP 缓存管理

```powershell
# 查看 ARP 缓存
arp -a

# 典型输出
# Interface: 192.168.1.100 --- 0x5
#   Internet Address    Physical Address      Type
#   192.168.1.1         r1-r1-r1-r1-r1-r1    dynamic   ← 动态学习，会过期
#   192.168.1.200       bb-bb-bb-bb-bb-bb     dynamic
#   192.168.1.255       ff-ff-ff-ff-ff-ff     static    ← 广播地址，静态

# 清除 ARP 缓存（排查时常用）
arp -d *

# 添加静态 ARP（一般不推荐，除非防 ARP 欺骗）
arp -s 192.168.1.200 BB-BB-BB-BB-BB-BB
```

**ARP 缓存超时：**
- Windows 默认：条目在 15-45 秒内如果没有再次使用就会过期
- 重新使用后可延长到 10 分钟
- 可通过注册表 `ArpCacheLife` 和 `ArpCacheMinReferencedLife` 调整

---

### 6. 常见问题与排查 (Common Issues & Troubleshooting)

#### 问题 1: "能 ping 通网关，但 ping 不通外网"

```
排查思路：

1. 确认网关是否配置正确
   > ipconfig | findstr "Gateway"

2. 确认网关是否可达
   > ping 192.168.1.1

3. 确认 DNS 是否正常（先 ping IP，排除 DNS 问题）
   > ping 8.8.8.8

4. 如果 ping IP 也不通，检查路由
   > tracert 8.8.8.8
   → 看到在哪一跳超时 → 定位问题路由器

5. 检查路由表是否有默认路由
   > route print | findstr "0.0.0.0"
   → 如果没有 0.0.0.0 的默认路由 → 需要添加

6. 可能原因：
   - 网关路由器没有上行路由
   - ISP 链路故障
   - 防火墙规则阻断
   - NAT 配置问题
```

#### 问题 2: "ping 对方 IP 偶尔丢包"

```
排查思路：

1. 持续 ping 看丢包规律
   > ping -t 10.0.0.50

2. 检查 ARP 是否稳定
   > arp -a
   → 如果 MAC 地址在变化 → ARP 欺骗或 IP 冲突

3. 双向 tracert 检查路径
   > tracert 10.0.0.50
   （从对方也做一次 tracert 回来）

4. 检查是否有链路拥塞
   → 持续 ping 的同时观察延迟变化
   → 延迟突然飙升 → 链路拥塞或带宽不足
```

#### 问题 3: "新设备接入网络后无法通信"

```
排查清单：

□ IP 地址是否正确配置（ipconfig）
□ 子网掩码是否和同网段其他设备一致
□ 默认网关是否正确
□ 网线/Wi-Fi 是否连接正常（物理层）
□ 交换机端口是否 UP
□ 是否有 DHCP 分配失败（169.254.x.x = APIPA 自动分配 → DHCP 失败）
□ ARP 是否能解析网关（arp -a）
□ 是否有 802.1X 认证拦截
□ VLAN 是否配置正确
```

---

### 7. 实战案例 (Real-World Cases)

#### 案例 1: 子网掩码不匹配导致单向通信

```
环境：
  Server-A: 10.10.1.100 / 255.255.255.0  (网段: 10.10.1.0/24)
  Server-B: 10.10.1.200 / 255.255.0.0    (网段: 10.10.0.0/16) ← 掩码错误！

现象：
  Server-A ping Server-B → 成功 ✅
  Server-B ping Server-A → 失败 ❌（有时）

分析：
  A → B: A 认为 B 在同网段（10.10.1.x/24）→ ARP 直接查 → 成功
  B → A: B 认为 A 在同网段（10.10.x.x/16）→ ARP 直接查 → 也成功

  但如果 B 要访问 10.10.2.x？
  B 认为 10.10.2.x 也在同网段 → ARP 直接查 → 但 10.10.2.x 实际在另一个广播域
  → ARP 查不到 → 失败！

修复：统一子网掩码为 255.255.255.0
```

#### 案例 2: ARP 缓存过期导致间歇性延迟

```
现象：每隔几分钟，访问某台服务器的第一个包延迟特别高（>100ms），后续恢复正常

分析：
  ARP 缓存过期 → 第一个包触发 ARP 查询 → 需要等待 ARP Reply
  → 额外延迟 = ARP 往返时间 + 重发时间
  
  如果 ARP Reply 丢失 → 需要重试 → 延迟更高

验证：
  > arp -a                        # 查看缓存
  > ping -t 10.0.0.50             # 持续 ping
  → 观察是否每隔固定时间出现一次高延迟

解决：
  - 如果是关键服务器，可以配置静态 ARP
  - 或者调大 ARP 缓存超时时间
```

#### 案例 3: 路由表缺失导致回程流量丢失

```
环境：
  PC (192.168.1.100) → Router-A → Router-B → Server (172.16.0.50)

现象：PC ping Server → 请求发出去了，但收不到回复

分析（抓包验证）：
  在 Router-B 抓包 → 看到 ICMP Request 到达 Server
  在 Server 抓包   → 看到 Request，也看到 Server 发了 Reply
  在 Router-B 抓包 → 看到 Reply 从 Server 出来

  但 Reply 的目的 IP 是 192.168.1.100
  Router-B 查路由表 → 没有 192.168.1.0/24 的路由条目！
  → 走默认路由 → 默认路由指向互联网 → 包被丢弃或送错方向

修复：在 Router-B 上添加回程路由
  ip route 192.168.1.0 255.255.255.0 [Router-A 的接口 IP]
```

---

### 8. 与 NAT 的关系 (How NAT Changes Things)

在实际的企业和家庭网络中，**NAT**（Network Address Translation）会改变我们之前说的"IP 地址全程不变"的规则：

```
没有 NAT 时：
  Src IP: 192.168.1.100 → 一路不变 → Server 看到的是 192.168.1.100

有 NAT 时（家庭路由器的典型行为）：
  Src IP: 192.168.1.100 → 到达 NAT 路由器 → 被替换为公网 IP（如 203.0.113.5）
  Server 看到的 Src IP: 203.0.113.5
  回复时 Dst IP: 203.0.113.5 → 到达 NAT 路由器 → 被还原为 192.168.1.100
```

**NAT 改变了 IP 包头的源/目的 IP**，这是"IP 不变"规则的唯一主要例外。

---

### 9. 参考资料 (References)

- [Address Resolution Protocol - Wikipedia](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) — ARP 协议详解
- [IPv4 Header - Wikipedia](https://en.wikipedia.org/wiki/IPv4#Header) — IPv4 包头字段解析
- [Routing Table - Wikipedia](https://en.wikipedia.org/wiki/Routing_table) — 路由表原理

---
---

## 📌 English Version

---

### 1. Overview

In the Junior post, we understood same-subnet and cross-subnet communication through analogies. Now we **peel away the metaphor** and examine what data packets actually look like at each step:

- What fields are in the Ethernet frame header?
- Which IP header fields change at each hop and which don't?
- What does an ARP request/reply actually look like?
- How does a router's routing table make forwarding decisions?
- How does TTL prevent routing loops?

---

### 2. Core Concepts — Deeper

#### 2.1 Encapsulation — "Nesting Envelopes"

TCP/IP communication is fundamentally a **layered encapsulation** process:

```
Application Data
  ↓ Add TCP/UDP Header
[ TCP Header | Application Data ]                    ← Layer 4: Segment
  ↓ Add IP Header
[ IP Header | TCP Header | Application Data ]        ← Layer 3: Packet
  ↓ Add Ethernet Header + FCS
[ Eth Header | IP Header | TCP Header | App Data | FCS ]  ← Layer 2: Frame
```

**Key principle**: At each router hop, **Layer 2 (Ethernet) is stripped and rewritten**, but **Layer 3 (IP) stays mostly the same** (only TTL decrements).

#### 2.2 Ethernet Frame Structure

```
┌──────────────────────────────────────────────────────┐
│ Destination MAC │ Source MAC │ EtherType │ Payload │ FCS │
│    (6 bytes)    │ (6 bytes)  │ (2 bytes) │         │     │
└──────────────────────────────────────────────────────┘
```

#### 2.3 IPv4 Header — Key Fields

| Field | Size | Purpose | Changes at each hop? |
|-------|------|---------|---------------------|
| Source IP | 4B | Sender's IP | ❌ No (unless NAT) |
| Destination IP | 4B | Receiver's IP | ❌ No (unless NAT) |
| TTL | 1B | Time to Live — decremented at each router | ✅ Yes, -1 per hop |
| Protocol | 1B | Upper layer protocol (TCP=6, UDP=17) | ❌ No |
| Header Checksum | 2B | Integrity check | ✅ Recalculated (TTL changed) |

#### 2.4 ARP Packet Structure

- **ARP Request**: Broadcast (Dst MAC = FF:FF:FF:FF:FF:FF), Target MAC = 00:00:00:00:00:00
- **ARP Reply**: Unicast back to requester with resolved MAC

#### 2.5 Routing Table — Longest Prefix Match

When a router receives a packet for `10.0.0.50`:
1. Check all matching routes: `10.0.0.0/24` matches, `0.0.0.0/0` matches
2. Select the **most specific** (longest prefix): `/24` wins over `/0`

---

### 3. How It Works — Full Packet Trace

#### Cross-Subnet HTTP Request

```
PC-A (192.168.1.100/24) ──[Switch-1]──[Router]──[Switch-2]── Server (10.0.0.50/24)
         MAC: AA:AA...    eth0: 192.168.1.1  eth1: 10.0.0.1    MAC: SS:SS...
                          MAC: R1:R1...      MAC: R2:R2...
```

**Frame #1: PC-A → Router**

| Field | Value | Explanation |
|-------|-------|-------------|
| Dst MAC | R1:R1:R1:R1:R1:R1 | Gateway's MAC (next hop) |
| Src MAC | AA:AA:AA:AA:AA:AA | PC-A's MAC |
| Dst IP | 10.0.0.50 | Final destination |
| Src IP | 192.168.1.100 | Origin |
| TTL | 128 | Windows default |

**Router processes:**
1. Accepts frame (Dst MAC matches eth0)
2. Strips Ethernet header, reads IP header
3. Looks up routing table: `10.0.0.0/24 → eth1`
4. Decrements TTL: 128→127
5. Recalculates IP checksum
6. ARPs for Server's MAC (if not cached)
7. Builds new Ethernet frame

**Frame #2: Router → Server**

| Field | Value | Changed? |
|-------|-------|----------|
| Dst MAC | SS:SS:SS:SS:SS:SS | ✅ New (Server's MAC) |
| Src MAC | R2:R2:R2:R2:R2:R2 | ✅ New (Router eth1) |
| Dst IP | 10.0.0.50 | ❌ Same |
| Src IP | 192.168.1.100 | ❌ Same |
| TTL | 127 | ✅ Decremented |

---

### 4. Multi-Hop Routing & tracert

At each hop: **MAC addresses are completely rewritten; IP addresses remain constant; TTL decrements by 1.**

`tracert` exploits this: sends packets with TTL=1, TTL=2, TTL=3... Each expired TTL triggers an ICMP "Time Exceeded" reply from that hop's router, revealing the path.

---

### 5. Common Issues & Troubleshooting

| Symptom | Likely Cause | Key Diagnostic |
|---------|-------------|----------------|
| Can ping gateway, not Internet | Missing default route, ISP link down, firewall | `tracert 8.8.8.8`, `route print` |
| Intermittent first-packet delay | ARP cache expiry | `arp -a`, look for periodic cache refresh |
| Subnet mask mismatch → partial connectivity | Incorrect neighbor determination | Compare `ipconfig` on both endpoints |
| Asymmetric reachability (A→B works, B→A fails) | Missing return route | Check routing tables on intermediate routers |
| 169.254.x.x address | DHCP failure (APIPA) | `ipconfig /release && ipconfig /renew` |

---

### 6. How NAT Changes Things

NAT is the major exception to "IP addresses never change":

```
Without NAT: Src IP stays 192.168.1.100 end-to-end
With NAT:    Router replaces Src IP → public IP (e.g., 203.0.113.5)
             Server sees 203.0.113.5, not 192.168.1.100
             Return traffic: Router translates back to 192.168.1.100
```

---

### 7. References

- [Address Resolution Protocol - Wikipedia](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) — ARP protocol details
- [IPv4 Header - Wikipedia](https://en.wikipedia.org/wiki/IPv4#Header) — IPv4 header field reference
- [Routing Table - Wikipedia](https://en.wikipedia.org/wiki/Routing_table) — Routing table mechanics
