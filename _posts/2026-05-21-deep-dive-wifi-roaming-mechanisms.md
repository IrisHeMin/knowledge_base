---
layout: post
title: "Deep Dive: WiFi Roaming Mechanisms — IHV / Network-directed / OS-driven"
date: 2026-05-21
categories: [Knowledge, Networking]
tags: [wifi, roaming, 802.11, 802.11k, 802.11v, 802.11r, wlan, ndis, wdi, wificx]
type: "deep-dive"
---

> 本文是一篇技术原理文档，与任何具体 case 无关。聚焦讲清楚 WiFi 漫游的三种发起方式（IHV-driven / Network-directed / OS-driven）以及它们在 Windows Native WiFi 架构里如何分工、如何识别、如何排查。

---

# 中文版

## 1. 概述 (Overview)

**WiFi 漫游 (Roaming)** 指的是 STA（客户端）在同一个 ESS (Extended Service Set) 内，从一个 AP（一个 BSSID）切换到另一个 AP（另一个 BSSID）的过程。从 IEEE 802.11 标准角度严格说，这其实叫 **BSS Transition**，而不是蜂窝意义上的 handoff —— 因为 802.11 的整个切换是由**客户端**驱动的，AP 只能"建议"或"驱赶"，不能像 LTE 基站那样接管会话。

漫游看起来是一个动作，但**发起方有三种本质不同的来源**，搞清楚是哪一种、由谁触发，是排查"为什么会漫"、"为什么乱漫"、"漫了为什么这么慢"等问题的第一步：

| 类别 | 中文叫法 | 谁发起 |
|---|---|---|
| **IHV-driven** | 网卡 / 固件主动漫游 | WLAN 网卡自己的固件 / 驱动 |
| **Network-directed** | AP / 网络端引导漫游 | AP 或控制器（通过 802.11v BTM 帧或 Deauth/Disassoc）|
| **OS-driven** | 操作系统主动漫游 | Windows wlansvc 的 BetterAP 算法 |

这三类在 Windows 的 ETL trace 里有完全不同的特征签名，也对应完全不同的修复路径（修网卡驱动 / 修 AP 配置 / 修 OS profile），混淆它们会导致排查方向跑偏。

## 2. 核心概念 (Core Concepts)

### BSS / ESS / BSSID / SSID
- **BSS (Basic Service Set)**：一个 AP 覆盖的逻辑网络。
- **BSSID**：这个 BSS 的标识，通常是 AP 无线接口的 MAC 地址。
- **ESS (Extended Service Set)**：多个 BSS 用同一个 SSID 和同一个分布式系统（DS，通常是有线骨干）连起来。
- **SSID**：用户可见的网络名（如 `Corp-WiFi`）。
- **漫游 = 同一个 ESS 内 BSSID 改变**。SSID 不变，DHCP 一般也不变。

### STA / IHV / wlansvc / WDI / WiFiCx
Windows Wi-Fi 栈是分层的：

```
┌──────────────────────────┐
│   User-mode UI / WMI     │  Network flyout, netsh, MDM CSP
├──────────────────────────┤
│   wlansvc.dll (UM)       │  Auto Configuration Module (ACM)
│   - Profile 管理          │  - BetterAP 算法 (OS-driven roam)
│   - 802.1X 状态机         │  - 接收 IHV 上抛的 RoamingNeeded
├──────────────────────────┤
│   NDIS / WDI / WiFiCx    │  内核接口
├──────────────────────────┤
│   IHV miniport driver    │  网卡驱动 (Intel / Qualcomm / Realtek …)
├──────────────────────────┤
│   WLAN NIC firmware      │  网卡固件 (RSSI 监测、PER、Beacon Loss)
└──────────────────────────┘
```

> **WDI (WLAN Device Driver Interface)** 是 Windows 10 引入的统一驱动模型；**WiFiCx** 是 Windows 11 起的新模型。两者都保留了"IHV 可以主动通知 OS 要漫游"的接口，行为基本一致。

### Roam Trigger 编码
ETL 里 `Roam trigger` 字段的常见取值：

| Trigger 值 | 含义 | 对应的发起类型 |
|---|---|---|
| **1** | Reconnect after disassociation | 通常是 Network-directed 或信号丢失的衍生结果 |
| **2** | OS-driven BetterAP roam | OS-driven |
| **3** | IHV-requested Best Effort Roam | IHV-driven |

### PMK / PMKSA / PMK Caching
WPA2/WPA3-Enterprise 下，每次跑完整 EAP 后会派生一个 **PMK (Pairwise Master Key)**。如果客户端启用了 **PMK Caching**，再次关联同一个 BSSID 时可以跳过 EAP 直接做 4-way handshake，亚秒级完成。
- 影响"漫游耗时"的关键开关。
- Windows 端控制点：`<PMKCacheMode>` profile 元素。

### 802.11r / 802.11k / 802.11v —— 三个互补的标准
- **802.11k (Radio Resource Measurement)**：AP 给客户端一份"邻居 AP 列表"（Neighbor Report），辅助选目标。
- **802.11v (Wireless Network Management)**：AP 主动告诉客户端"建议你去 BSSID X"——也就是 **BTM (BSS Transition Management) Request**。
- **802.11r (Fast BSS Transition / FT)**：跨 AP 的快速密钥协商，让漫游时不用重新跑 EAP 也能保证安全。
- 这三者是"漫游加速包"，本身不是漫游发起方，但会显著影响漫游决策质量和速度。

## 3. 工作原理 (How It Works)

### 整体架构

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ OS-driven    │     │ IHV-driven   │     │ Network-dir. │
│ (wlansvc)    │     │ (firmware)   │     │ (AP)         │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │ trigger=2          │ trigger=3          │
       │ BetterAP found     │ RoamingNeeded      │ BTM Req / Deauth
       │                    │ indication         │
       ▼                    ▼                    ▼
       └────────────► 触发同一条漫游执行路径 ◄─────┘
                       │
                       ▼
              ┌──────────────────────┐
              │ CRoamReconnectJob    │
              │ - Candidate select   │
              │ - Disassoc old AP    │
              │ - Auth + Assoc new AP│
              │ - 4-way handshake    │
              │ - (EAP 重做或 PMK 复用)│
              └──────────────────────┘
```

三类发起源在执行漫游本身时，走的是同一条 ConnectJob/Reassoc 路径；**区别只在"由谁按下扳机"**。

### 类别 1: IHV-driven（网卡主动漫游）

**触发条件（住在固件里，OS 看不到细节）：**
- RSSI 持续低于厂商设定阈值（如 -70 dBm）
- PER (Packet Error Rate) 上升到阈值
- 连续 N 个 Beacon 丢失
- Tx Failure 计数超过阈值
- 厂商自家的 Roaming Aggressiveness 等级（Lowest/Low/Medium/High/Highest）
- 部分厂商还有专有的"AP 负载/客户端数量"等私有信号

**ETL 里的特征签名：**
```
CPort::OnRoamingNeeded()
    [Roam][Link] Trigger roam as roaming needed indication received
CPort::TriggerIHVRequestedRoam()
    [Roam][IHV][BetterAP] Triggering IHV requested Best Effort Roam
CPort::StartRoamJob()
    [Roam] Roam operation triggered. Roam trigger 3
```

**典型表现：**
- OS 自己的 BetterAP 评估同时刻可能在打"信号好不需要漫游"，但漫游照样发生
- 切换目标可能不是 OS 候选列表里的 Top 1
- 排查时**必须拿 IHV 自己的日志**（如 Intel Wireless Driver Log Collector），单看 Windows 端只能看到结果

### 类别 2: Network-directed（AP 引导漫游）

**两种典型实现：**

**A. 标准的 802.11v BTM Request（推荐）**
```
AP ──BTM Request (Action frame)──► STA
   {
     Disassoc Imminent flag,
     Disassoc Timer (TBTT count),
     BSS Termination flag,
     Candidate List: [BSSID, channel, operating class, ...]
   }
```
客户端可以接受（回 BTM Response，然后漫走）也可以拒绝。如果 AP 设了 Disassoc Imminent，超时未漫就会被踢。

**B. 非标准的 vendor-specific steering（band steering / load balancing）**
- AP 故意忽略客户端在某个 band 的 Probe Request
- AP 主动 Deauth/Disassoc 把客户端踢下线（Reason Code 4=Inactivity, 5=AP busy, 等）
- 客户端被动重连，自然就会选别的 AP

**ETL 里的特征签名：**
- 收到 BTM 帧：`WnmAction` / `BSS Transition Management Request` / `TransitionMgmt`
- 被踢：先看到 Disassoc 帧（带 Reason Code），随后 `OnDisassociated → TriggerReconnect → Roam trigger 1`

**典型表现：**
- 漫游时机和 AP 端策略变化相关（AP 升级、负载策略调整、Roaming Hint 上线）
- Windows ETL 只能看到 BTM/Deauth 帧本身和触发的重连，**"为什么 AP 决定要让你走"得问 AP 端日志**
- vendor-specific 的 steering Windows 几乎看不到，必须 AP 端配合

### 类别 3: OS-driven（Windows 自己决定）

**触发条件（住在 wlansvc 里，可观测）：**
- 周期性评估当前连接的 Link Quality 和候选 AP 列表
- 当前 LQ 跌到阈值以下 → 找候选
- 候选里有更高 rank 的 AP → 漫
- rank 计算包括：原始 LQ、Band boost（5GHz/6GHz 加分）、最近一次失败时间窗（block list）等

**ETL 里的特征签名：**
```
CPort::DetermineIfSwitchToBetterAPNeeded()
    [Roam][BetterAP] Roaming as link quality is low (xx)
        OR
    [Roam][BetterAP] Better AP found, current rank=xx, target rank=yy
CPort::StartRoamJob()
    [Roam] Roam operation triggered. Roam trigger 2
```
反向证据（OS 主动选择不漫）：
```
[Roam][BetterAP] Not roaming as Link quality of network not low enough (88)
```

**典型表现：**
- 信号确实差到了 OS 阈值（一般 LQ < 50~60）才动
- 行为可被 profile / 注册表调（比如禁用某些 BSSID 进 block list）
- 往往是三类里最"克制"的，纯软件决策，不会被 IHV 那种厂商私有逻辑误导

### 关键机制：漫游执行的统一路径

不管是哪类发起，触发后都会走相同的 6 步：

1. **候选选择**：从 BSS list 里挑 N 个候选（CheckAndUpdateCandidates）
2. **Disassoc/Deauth 老 AP**（IHV 漫游里这一步往往是"先发生再通知 OS"）
3. **Auth + Assoc 新 AP**（如果支持 802.11r FT，这一步直接带密钥协商）
4. **EAP / PMK 决策**：
   - 如果 PMK Cache 命中 → 跳过 EAP 直接 4-way handshake（亚秒级）
   - 如果未命中 → 完整 EAP（事件 12011/12014，可能十几秒，特别是云端 RADIUS）
5. **4-way handshake** → 派生 PTK
6. **数据面恢复** + 通知上层（NDIS media connect → DHCP renew/无变化 → 应用层）

## 4. 关键配置与参数 (Key Configurations)

| 配置项 | 位置 | 默认值 | 说明 | 调优场景 |
|---|---|---|---|---|
| `PMKCacheMode` | WLAN profile XML (security) | `enabled` | PMK 缓存开关 | 企业云端 RADIUS 时务必启用，避免每次漫游全量 EAP |
| `PMKCacheTTL` | WLAN profile XML | 720 (分钟) | PMK 在客户端缓存的有效期 | 长会议室场景可适当延长 |
| `PMKCacheSize` | WLAN profile XML | 128 | 同时可缓存的 PMK 条数（每个对应一个 BSSID）| 高密度部署可适当调大 |
| `preAuthMode` | WLAN profile XML | `disabled` | 802.11i Pre-Authentication（漫游前提前对邻居 AP 完成 EAP）| 部署支持 pre-auth 的环境时启用 |
| Roaming Aggressiveness | Intel/厂商驱动 advanced property | Medium (3) | IHV 漫游算法激进度 (1-5) | 出现 ping-pong 漫游时调低 |
| 802.11r FT (Fast Transition) | AP 侧 + 客户端协商 | 取决于 AP 配置 | 跨 AP 快速密钥协商 | 实时业务（VoIP/视频会议）必开 |
| 802.11v BTM | AP 侧 | 取决于 AP | 网络端引导漫游能力 | 高密度部署的负载均衡需要 |
| 802.11k Neighbor Report | AP 侧 | 取决于 AP | 邻居列表 | 让客户端少做 active scan，省电省时 |
| `BSSEntryTimeout` | wlansvc 注册表 (`HKLM\SOFTWARE\Microsoft\wlansvc\Parameters`) | — | BSS list entry 老化时间 | 高移动场景下避免用过期的 BSS 信息 |

## 5. 常见问题与排查 (Common Issues & Troubleshooting)

### 问题 A：客户端"无理由"频繁漫游 / Ping-pong 漫游
- **症状**：信号稳定的环境下还是几十秒一次切 BSSID，Teams 频繁掉
- **可能原因**：IHV-driven 算法过于激进，或 AP 端做了过激的 band/load steering
- **排查思路**：
  1. 抓 net_wlan ETL 看 `Roam trigger`，区分 2 / 3 / 1
  2. trigger=3 → IHV 问题，调 Roaming Aggressiveness 到 Lowest 验证
  3. trigger=1 → 看是否有 BTM 帧或 Deauth Reason Code → AP 端问题
  4. trigger=2 → 看 BetterAP 阈值，是否环境 LQ 真的在阈值上下抖动
- **关键工具**：`netsh trace start scenario=wlan`、Intel Wireless Driver Log Collector、AP 端 client log

### 问题 B：每次漫游断网 5~30 秒
- **症状**：BSSID 切换时业务明显卡顿，TCP 大量重传
- **可能原因**：
  - PMKCacheMode = disabled，每次漫游全量 EAP
  - 没启用 802.11r FT
  - RADIUS 在云端，EAP 往返延迟大
- **排查思路**：
  1. 看 WLAN-AutoConfig 事件日志，每次漫游是否都有 12014 (auth restarted)
  2. 看 profile XML 里 `<PMKCacheMode>` 取值
  3. 看 Auth Algo 是否协商到 FT-802.1X
- **关键命令**：
  ```cmd
  netsh wlan show profile name="<SSID>" key=clear
  netsh wlan show interface
  ```

### 问题 C：移动到新 AP 附近但客户端死活不漫
- **症状**：用户已经走到新 AP 旁边，旧 AP 信号 -85 dBm，但客户端不切
- **可能原因**：
  - IHV 算法保守，且 OS BetterAP 的"低信号阈值"太低
  - profile autoSwitch=false 或单 BSSID profile
- **排查思路**：
  1. 看 ETL 是否有 `[Roam][BetterAP] Not roaming as Link quality not low enough`
  2. 看 Roaming Aggressiveness 是否被调到了 Lowest
  3. 检查 AP 是否广播 BTM，但客户端没接受

### 问题 D：漫游后 IP 变了 / DHCP 重新拿 IP
- **症状**：漫游后业务彻底中断需要重连
- **可能原因**：跨 VLAN 漫游 / 跨 subnet ESS（部署不当）
- **排查思路**：检查 AP/控制器是否把同一 SSID 桥接到同一 VLAN

## 6. 实战经验 (Practical Tips)

- **判断"谁让我漫的"看 Roam trigger 比看任何东西都快**：trigger 3 = 网卡，trigger 2 = OS，trigger 1 = 被动跟随（再倒回去找 Disassoc/BTM 来源）。
- **Windows ETL 看不到 IHV 的内部理由**，永远不要在 trigger=3 的场景里说"是 Windows 决定漫的"——这是常见误判。
- **PMKCacheMode 跟漫游"是否发生"无关，只跟漫游"花多久"有关**。两件事经常被混在一起谈，分清楚再下结论。
- **企业云端 RADIUS（Portnox / Azure NPS / ISE Cloud）部署务必同时启用 PMK Cache + 802.11r FT**，否则每次漫游都是公网往返一次 EAP-TLS。
- **Windows wlan-report (`netsh wlan show wlanreport`)** 会生成一个 HTML 报告，包含最近的连接/断开/漫游摘要，适合快速过一遍是不是有大量漫游事件。
- **block-list 机制**：连接失败的 BSSID 会被 wlansvc 临时拉黑（默认几分钟），所以"客户端为什么不选信号最好的那个 AP"有时答案就是它在 block list 里。
- **2.4GHz vs 5GHz vs 6GHz 的"band steering"** 是 AP 厂商私有的，不属于上面任何一类的"标准"逻辑，但表现出来通常是 #2 (Network-directed)。

## 7. 与相关技术的对比 (Comparison)

| 维度 | IHV-driven | Network-directed | OS-driven |
|---|---|---|---|
| 触发方 | 网卡固件 | AP / 控制器 | Windows wlansvc |
| 标准化程度 | 厂商私有 | 802.11v 标准 / 或 vendor 私有 | OS 私有，但阈值可观测 |
| 反应速度 | 最快（µs~ms 级感知信号变化） | 中（取决于 AP 策略） | 慢（秒级评估周期） |
| 决策质量 | 看厂商，差异大 | 取决于网络规划 | 偏保守，依赖好的 LQ 信号 |
| OS 可见度 | 只看到结果 | 看到 BTM 帧或 Deauth | 全可见 |
| 修复路径 | 改驱动设置 / 抓 IHV 日志 | 改 AP 配置 | 改 profile / 注册表 |
| 跟 802.11r FT | 兼容 | 兼容（BTM 可携带 FT 信息）| 兼容 |
| 跟 PMK Cache | 受益 | 受益 | 受益 |

**选型建议**：企业网部署最理想的组合是 **OS-driven 兜底 + 802.11k 提供候选 + 802.11v 做主动引导 + 802.11r 做快速切换**，IHV 算法保持中等激进度。把所有决策权完全交给 IHV（或完全交给 OS）都会出问题。

## 8. 参考资料 (References)

- [Native 802.11 Software Architecture (Microsoft Docs)](https://learn.microsoft.com/en-us/windows-hardware/drivers/network/native-802-11-software-architecture) — Windows Native WiFi 整体架构和 IHV/OS 分工
- [WLAN_profile schema — security (MSM) element](https://learn.microsoft.com/en-us/windows/win32/nativewifi/wlan-profileschema-security-msm-element) — PMKCacheMode/TTL/Size、preAuthMode 等 profile 元素的官方 schema
- [WLAN Universal Windows Driver Model (WDI) / WiFiCx](https://learn.microsoft.com/en-us/windows-hardware/drivers/network/wifi-universal-driver-model) — Windows 10/11 Wi-Fi 驱动模型（WDI 维护态、WiFiCx 是 Win11 推荐）
- [IEEE 802.11r-2008 (Fast BSS Transition) — Wikipedia](https://en.wikipedia.org/wiki/IEEE_802.11r-2008) — FT 协议背景和 rationale
- [IEEE 802.11v (Wireless Network Management) — Wikipedia](https://en.wikipedia.org/wiki/IEEE_802.11v) — BTM 标准来源
- [IEEE 802.11k-2008 (Radio Resource Measurement) — Wikipedia](https://en.wikipedia.org/wiki/IEEE_802.11k-2008) — Neighbor Report 标准来源
- [Intel Support: Wireless Adapter Settings (Roaming Aggressiveness)](https://www.intel.com/content/www/us/en/support/articles/000005585/wireless.html) — Intel 网卡 IHV 漫游算法的可配置项

---

# English Version

## 1. Overview

**WiFi roaming** is the process where an STA (client) switches from one AP (one BSSID) to another within the same ESS (Extended Service Set). Per IEEE 802.11 strictly, this is called **BSS Transition** rather than handoff in the cellular sense — because the entire transition is **client-driven**. The AP can suggest or evict, but it cannot take over the session like an LTE base station.

Roaming looks like a single action, but **there are three fundamentally different originators** of the trigger. Identifying which one is at play is the first step in diagnosing "why did it roam", "why is it bouncing", or "why is the roam so slow":

| Class | Common name | Originator |
|---|---|---|
| **IHV-driven** | NIC / firmware-initiated | The WLAN NIC's own firmware / driver |
| **Network-directed** | AP-/controller-initiated | The AP via 802.11v BTM or Deauth/Disassoc |
| **OS-driven** | Operating system-initiated | Windows wlansvc BetterAP algorithm |

These three have completely different signatures in Windows ETL traces and map to completely different remediation paths (fix the NIC driver / fix the AP config / fix the OS profile). Confusing them sends troubleshooting in the wrong direction.

## 2. Core Concepts

### BSS / ESS / BSSID / SSID
- **BSS**: the logical network covered by one AP
- **BSSID**: identifier of that BSS, usually the AP radio's MAC
- **ESS**: multiple BSSes sharing the same SSID and DS (wired backbone)
- **SSID**: user-visible name (e.g., `Corp-WiFi`)
- **Roaming = BSSID changes within the same ESS**. SSID stays; DHCP usually doesn't re-lease.

### Windows WiFi stack layering
```
User UI / WMI
   ▼
wlansvc.dll (UM)  ← Auto Configuration Module (ACM), BetterAP, 802.1X SM
   ▼
NDIS / WDI / WiFiCx
   ▼
IHV miniport driver
   ▼
WLAN NIC firmware
```
> **WDI** is the unified driver model from Win10. **WiFiCx** is the new model since Win11. Both keep the "IHV may proactively notify the OS to roam" interface.

### Roam trigger codes (in ETL)
| Trigger | Meaning | Originator class |
|---|---|---|
| **1** | Reconnect after disassociation | Usually a side-effect of network-directed roams or signal loss |
| **2** | OS-driven BetterAP roam | OS-driven |
| **3** | IHV-requested Best Effort Roam | IHV-driven |

### PMK / PMKSA / PMK Caching
Under WPA2/WPA3-Enterprise, each full EAP run derives a **PMK**. With **PMK Caching**, reassociation to a known BSSID skips EAP and goes straight to 4-way handshake — sub-second.
- The single biggest knob for "how long a roam takes".
- Windows control: `<PMKCacheMode>` profile element.

### 802.11r / 802.11k / 802.11v — three complementary amendments
- **802.11k** (RRM): AP gives the client a Neighbor Report
- **802.11v** (WNM): AP sends a **BTM (BSS Transition Management) Request** to suggest/force a target
- **802.11r** (FT): cross-AP fast key negotiation, no full EAP needed
- These are "roaming accelerators", not originators, but they significantly affect decision quality and speed.

## 3. How It Works

### Big picture
```
OS-driven         IHV-driven         Network-directed
(wlansvc)         (firmware)         (AP)
   │ trig=2           │ trig=3            │ BTM Req / Deauth
   │ BetterAP         │ RoamingNeeded     │
   ▼                  ▼                   ▼
   └──── all three feed the same execution path ────┘
                     │
                     ▼
        ┌─────────────────────────┐
        │ CRoamReconnectJob       │
        │ - candidate selection   │
        │ - disassoc old AP       │
        │ - auth + assoc new AP   │
        │ - 4-way handshake       │
        │ - (full EAP or PMK reuse)│
        └─────────────────────────┘
```
The three originators converge on the same Connect/Reassoc path. The difference is purely **who pulls the trigger**.

### Class 1: IHV-driven

**Triggers (live inside firmware, opaque to OS):**
- RSSI sustained below vendor threshold (e.g., -70 dBm)
- PER (Packet Error Rate) above threshold
- N consecutive missed beacons
- Tx Failure counter exceeds threshold
- Vendor's "Roaming Aggressiveness" level (Lowest..Highest)
- Some vendors include AP-load / client-count signals

**ETL signature:**
```
CPort::OnRoamingNeeded()
    [Roam][Link] Trigger roam as roaming needed indication received
CPort::TriggerIHVRequestedRoam()
    [Roam][IHV][BetterAP] Triggering IHV requested Best Effort Roam
CPort::StartRoamJob()
    [Roam] Roam operation triggered. Roam trigger 3
```

**Typical symptoms:**
- OS BetterAP at the same instant might say "signal good, no roam needed", but it roams anyway
- The chosen target may not be the OS's top-ranked candidate
- You **must** collect the IHV's own log (e.g., Intel Wireless Driver Log Collector) — Windows alone only shows the result

### Class 2: Network-directed

**Two flavors:**

**A. Standardized 802.11v BTM Request (preferred)**
```
AP ──BTM Request──► STA
   {
     Disassoc Imminent flag,
     Disassoc Timer (TBTT count),
     BSS Termination flag,
     Candidate List: [BSSID, channel, op class, ...]
   }
```
The client may accept (BTM Response then roam) or reject. With Disassoc Imminent set, ignoring the request leads to forced eviction.

**B. Vendor-specific steering (band steering / load balancing)**
- AP intentionally ignores Probe Requests on a band
- AP sends Deauth/Disassoc with Reason Code (4=Inactivity, 5=AP busy, etc.)
- Client passively reconnects — and naturally picks a different AP

**ETL signature:**
- BTM frame received: `WnmAction` / `BSS Transition Management Request` / `TransitionMgmt`
- Eviction: Disassoc with Reason Code → `OnDisassociated → TriggerReconnect → Roam trigger 1`

### Class 3: OS-driven

**Triggers (live in wlansvc, observable):**
- Periodic evaluation of Link Quality + candidate AP list
- Current LQ falls below threshold → search candidates
- Candidate has higher rank → roam
- Rank includes raw LQ, band boost (5/6 GHz), recent failure block-list timing

**ETL signature:**
```
CPort::DetermineIfSwitchToBetterAPNeeded()
    [Roam][BetterAP] Roaming as link quality is low (xx)
        OR
    [Roam][BetterAP] Better AP found, current rank=xx, target rank=yy
CPort::StartRoamJob()
    [Roam] Roam operation triggered. Roam trigger 2
```
Counter-evidence (OS chooses NOT to roam):
```
[Roam][BetterAP] Not roaming as Link quality of network not low enough (88)
```

### Unified execution path (regardless of originator)
1. Candidate selection
2. Disassoc/Deauth old AP (in IHV roams this often happens *before* notifying OS)
3. Auth + Assoc new AP (with FT this carries key material)
4. EAP / PMK decision: cache hit → skip EAP; cache miss → full EAP (events 12011/12014)
5. 4-way handshake → derive PTK
6. Data plane resumes → NDIS media connect → DHCP renew (or no change) → app

## 4. Key Configurations

| Setting | Location | Default | Purpose | When to tune |
|---|---|---|---|---|
| `PMKCacheMode` | WLAN profile XML | `enabled` | PMK cache switch | Always enable for cloud RADIUS |
| `PMKCacheTTL` | WLAN profile XML | 720 (min) | Cache lifetime | Extend for long-stay scenarios |
| `PMKCacheSize` | WLAN profile XML | 128 | Max cached entries | Bump for high-density deployments |
| `preAuthMode` | WLAN profile XML | `disabled` | 802.11i Pre-Authentication | Enable when AP supports pre-auth |
| Roaming Aggressiveness | Vendor driver advanced property | Medium (3) | IHV roam aggressiveness | Lower it on ping-pong roams |
| 802.11r FT | AP + client | depends | Fast key negotiation | Mandatory for VoIP/RT video |
| 802.11v BTM | AP | depends | Network-directed steering | Required for load balancing |
| 802.11k Neighbor Report | AP | depends | Neighbor list | Reduces active scan cost |

## 5. Common Issues & Troubleshooting

### Issue A: Frequent / ping-pong roaming with no apparent cause
- **Likely cause**: aggressive IHV algorithm or aggressive AP-side steering
- **Approach**: capture net_wlan ETL → check `Roam trigger` (2/3/1) → branch accordingly
- **Tools**: `netsh trace start scenario=wlan`, IHV log collector, AP client log

### Issue B: Each roam blacks out for 5–30 seconds
- **Likely cause**: PMK Cache disabled / no FT / cloud RADIUS RTT
- **Approach**: check WLAN-AutoConfig events for repeated 12014; check profile `<PMKCacheMode>`; check FT-802.1X negotiation
- **Commands**: `netsh wlan show profile name="<SSID>" key=clear`, `netsh wlan show interface`

### Issue C: Refuses to roam even with very low signal
- **Likely cause**: conservative IHV + low BetterAP threshold; or `autoSwitch=false`
- **Approach**: check ETL for "Not roaming as LQ not low enough"; check Roaming Aggressiveness; check whether BTM is being broadcast but ignored

### Issue D: IP changes after roam
- **Likely cause**: cross-VLAN roam (deployment issue)
- **Approach**: ensure same SSID bridges to same VLAN on the controller

## 6. Practical Tips

- **`Roam trigger` is the fastest single signal** to identify the originator. 3 = NIC, 2 = OS, 1 = passive (trace back to Disassoc/BTM).
- **Windows ETL never reveals the IHV's internal reason.** Don't claim "Windows decided to roam" when trigger=3.
- **PMKCacheMode does not affect whether a roam happens, only how long it takes.** Keep these two questions separated.
- For enterprise cloud-RADIUS deployments (Portnox, Azure NPS in cloud, ISE Cloud) **always enable PMK Cache and 802.11r FT** — otherwise every roam round-trips EAP-TLS over the public internet.
- **`netsh wlan show wlanreport`** gives a digestible HTML summary of recent connect/disconnect/roam events.
- **Block-list**: failed BSSIDs get temporarily blacklisted by wlansvc. "Why doesn't it pick the strongest AP?" sometimes answers itself — that AP is in the block list.

## 7. Comparison

| Dimension | IHV-driven | Network-directed | OS-driven |
|---|---|---|---|
| Originator | NIC firmware | AP / controller | wlansvc |
| Standardization | Vendor-proprietary | 802.11v standard or vendor-proprietary | OS-proprietary, observable |
| Reaction speed | Fastest (µs~ms) | Medium (depends on AP policy) | Slowest (seconds) |
| Decision quality | Vendor-dependent, varies widely | Depends on network design | Conservative, depends on accurate LQ |
| OS visibility | Result only | BTM frame or Deauth visible | Fully visible |
| Fix path | Driver settings / IHV log | AP config | Profile / registry |
| Works with FT | Yes | Yes (BTM can carry FT) | Yes |
| Works with PMK cache | Yes | Yes | Yes |

**Recommendation**: the ideal enterprise combo is **OS-driven as fallback + 802.11k for candidate hints + 802.11v for proactive steering + 802.11r for fast switching**, with IHV aggressiveness kept at Medium. Handing all decisions to a single layer (only IHV, or only OS) tends to misbehave.

## 8. References

- [Native 802.11 Software Architecture (Microsoft Docs)](https://learn.microsoft.com/en-us/windows-hardware/drivers/network/native-802-11-software-architecture)
- [WLAN_profile schema — security (MSM) element](https://learn.microsoft.com/en-us/windows/win32/nativewifi/wlan-profileschema-security-msm-element)
- [WLAN Universal Windows Driver Model (WDI) / WiFiCx](https://learn.microsoft.com/en-us/windows-hardware/drivers/network/wifi-universal-driver-model)
- [IEEE 802.11r-2008 (Fast BSS Transition) — Wikipedia](https://en.wikipedia.org/wiki/IEEE_802.11r-2008)
- [IEEE 802.11v (Wireless Network Management) — Wikipedia](https://en.wikipedia.org/wiki/IEEE_802.11v)
- [IEEE 802.11k-2008 (Radio Resource Measurement) — Wikipedia](https://en.wikipedia.org/wiki/IEEE_802.11k-2008)
- [Intel Support: Wireless Adapter Settings (Roaming Aggressiveness)](https://www.intel.com/content/www/us/en/support/articles/000005585/wireless.html)
