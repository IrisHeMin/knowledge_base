---
layout: post
title: "Deep Dive: DNS Scavenging 深入理解"
date: 2026-03-13
categories: [Knowledge, Networking]
tags: [dns, scavenging, aging, dynamic-update, active-directory, windows-server]
type: "deep-dive"
---

# Deep Dive: DNS Scavenging (DNS 清理机制)

**Topic:** Windows DNS Scavenging & Aging  
**Category:** Networking / DNS  
**Level:** 中级 / Intermediate  
**Last Updated:** 2026-03-13

---

# 中文版

## 1. 概述 (Overview)

DNS Scavenging（DNS 清理/老化清除）是 Windows DNS Server 内置的一套**自动清理过期 DNS 记录**的机制。在动态 DNS (DDNS) 环境中，客户端会自动注册自己的 A 记录和 PTR 记录。但当客户端被移除、关机、更换 IP 后，这些旧记录不会自动消失，久而久之 DNS 区域中就会堆积大量"僵尸记录"（Stale Records）。

Scavenging 就是解决这个问题的：**它像一个定时清洁工，定期检查所有记录的"时间戳"，把过期太久、没有被客户端刷新的记录删掉。**

但是，由于删除 DNS 记录是一件高风险操作（删错了可能导致服务中断），Windows 在 Scavenging 中设计了**多层安全阀门**，需要在三个地方都正确配置才能生效：
1. **资源记录 (Resource Record)** — 记录本身要有时间戳
2. **区域 (Zone)** — 区域要启用 Aging/Scavenging
3. **服务器 (Server)** — 至少一台 DNS 服务器要启用自动清理

> 🔑 **一句话总结**：三个地方都开了，Scavenging 才能工作。少了任何一个，过期记录都不会被删。

## 2. 核心概念 (Core Concepts)

### 2.1 时间戳 (Timestamp)

每条 DNS 记录都有一个**时间戳字段**，记录它最后一次被"刷新"或"更新"的时间。

- **动态注册的记录**：自动有时间戳（Windows 客户端默认每 24 小时向 DNS 注册一次）
- **静态记录**（管理员手动创建的）：时间戳为 **0**，表示**永不清理**
- 时间戳精度为**小时**（向下取整到最近的整点）

> 🏠 **类比**：时间戳就像酒店房间的"退房时间"。每次客人（客户端）续住（刷新记录），退房时间就更新。如果客人走了再也不来续住，过了退房时间就会被清理。

### 2.2 No-Refresh 间隔（不刷新间隔）

**No-Refresh Interval**（默认 7 天）是一个**禁止刷新的窗口期**。在这段时间内，即使客户端尝试刷新记录，时间戳也不会更新。

**为什么需要这个？** 因为时间戳变化 = AD 复制流量。如果每个客户端每 24 小时刷新一次记录，每次刷新都改变时间戳，就会产生大量不必要的 AD 复制。No-Refresh 间隔就是用来**减少复制流量**的。

> 🏠 **类比**：No-Refresh 就像酒店说"入住前 7 天内不用打电话确认，我们知道你住着呢"。

### 2.3 Refresh 间隔（刷新间隔）

**Refresh Interval**（默认 7 天）紧接在 No-Refresh 之后。这个窗口期内，客户端**可以且应该**来刷新时间戳。

如果在 Refresh 窗口期结束时客户端仍然没有刷新记录，该记录就变成**"可清理"（Eligible for Scavenging）**。

> 🏠 **类比**：Refresh 间隔就是酒店给你的"续住确认期"。7 天内你需要打电话说"我还住着"，否则酒店就认为你走了。

### 2.4 Scavenging 周期（清理周期）

这是 DNS 服务器多久执行一次清理操作（默认 7 天）。即使记录已经"可清理"了，也要等到服务器下一次运行 Scavenging 时才会被真正删除。

### 2.5 Update vs Refresh

这是一个**非常容易混淆**的核心概念：

| 操作 | 含义 | 是否受 No-Refresh 限制 |
|------|------|----------------------|
| **Update（更新）** | 记录的数据变了（如 IP 地址改变） | ❌ 不受限，任何时候都可以 |
| **Refresh（刷新）** | 记录数据没变，只是"续命"更新时间戳 | ✅ 受限，No-Refresh 期间被拒绝 |

## 3. 工作原理 (How It Works)

### 3.1 整体架构

```
┌──────────────────────────────────────────────────────────┐
│                    DNS Scavenging 三层架构                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│   Layer 1: Resource Record（资源记录）                      │
│   ┌─────────────────────────────────────┐               │
│   │ Record: app-server01.contoso.com    │               │
│   │ IP: 10.1.1.100                      │               │
│   │ Timestamp: 2026/03/01 10:00:00      │  ← 时间戳     │
│   │ [x] Delete when stale               │               │
│   └─────────────────────────────────────┘               │
│                                                          │
│   Layer 2: Zone（区域）                                    │
│   ┌─────────────────────────────────────┐               │
│   │ Zone: contoso.com                   │               │
│   │ [x] Scavenge stale resource records │  ← 启用清理    │
│   │ No-Refresh: 7 days                  │               │
│   │ Refresh:    7 days                  │               │
│   │ Zone can be scavenged after:        │               │
│   │   2026/03/08 10:00:00               │  ← 安全阀     │
│   └─────────────────────────────────────┘               │
│                                                          │
│   Layer 3: Server（服务器）                                 │
│   ┌─────────────────────────────────────┐               │
│   │ Server: dns-srv01.contoso.com       │               │
│   │ [x] Enable automatic scavenging     │  ← 启用自动    │
│   │ Scavenging period: 7 days           │               │
│   └─────────────────────────────────────┘               │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 3.2 完整生命周期 — 用一个故事讲清楚

假设公司 contoso.com 的环境：
- **No-Refresh 间隔**：7 天
- **Refresh 间隔**：7 天  
- **Scavenging 周期**：7 天

一台笔记本 `laptop01.contoso.com`（IP: 10.1.1.50）的 DNS 记录一生：

```
时间线 (Timeline)
═══════════════════════════════════════════════════════════════

Day 0 (3月1日): 笔记本开机，通过 DDNS 注册记录
├── 记录创建: laptop01.contoso.com → 10.1.1.50
├── 时间戳设置为: 3月1日 09:00
└── 这是一次 "Update"（新建记录）

Day 1-7 (No-Refresh 窗口):
├── 笔记本每天尝试注册（每24h一次）
├── DNS 服务器说："No-Refresh 期间，不更新时间戳"
├── 时间戳保持: 3月1日 09:00
└── 好处：减少 AD 复制流量 ✓

Day 8-14 (Refresh 窗口):
├── 进入 Refresh 期间，现在刷新被允许了
├── Day 8: 笔记本注册 → 时间戳更新为 3月8日 09:00  ✓
├── Day 9-14: 又进入新的 No-Refresh 周期...
└── 时间戳现在是: 3月8日 09:00

Day 15: 笔记本被员工带走了，再也不开机
├── 没人来刷新这条记录了
├── No-Refresh 窗口 (Day 8-14) 正常过去
└── 时间戳冻结在: 3月8日 09:00

Day 15-21 (Refresh 窗口，但没人来):
├── 记录进入 Refresh 期，等待客户端来刷新
├── ......没人来......
└── Day 22: Refresh 窗口结束！

Day 22: 记录变为 "可清理" (Eligible for Scavenging)
├── 当前时间 > 时间戳(3/8) + No-Refresh(7天) + Refresh(7天)
├── 即: 3月22日 > 3月8日 + 14天 ✓
└── 但是！记录还没被删！要等 Scavenging 执行

Day 22-28: 等待 Scavenging 执行
├── 假设上一次 Scavenging 在 Day 21 执行过
├── 周期是 7 天，下次在 Day 28
└── 

Day 28: DNS 服务器执行 Scavenging
├── 扫描所有记录
├── 检查 laptop01: 时间戳 3/8 + 14天 = 3/22，已过期 ✓
├── 删除记录！🗑️
├── Event ID 2501: "X records scavenged"
└── laptop01.contoso.com 的记录正式消失

═══════════════════════════════════════════════════════════════
总耗时：记录最后一次刷新后，至少 14天 + 最多7天 = 14~21 天才会被删
       使用默认设置时，可能需要 14~21 天
```

### 3.3 Eligible 之后、删除之前 — 客户端还能"续命"吗？

这是一个非常关键但容易被忽视的场景：**当记录已经变为 Eligible（可清理），但 Scavenging 还没执行时，客户端突然回来注册了，会怎样？**

答案是：**可以刷新！记录会被"救回来"，整个周期重新开始。**

```
时间线 (假设全部 7 天间隔)
═════════════════════════════════════════════════════

Day 0:  时间戳 = 3月1日
        ├── No-Refresh 开始 (Day 0-7)
        │
Day 7:  No-Refresh 结束
        ├── Refresh 窗口开始 (Day 7-14)
        │   客户端可以刷新，但没来...
        │
Day 14: Refresh 窗口结束
        ├── 记录变为 "Eligible"（可清理）
        │   但还没被删！等 Scavenging 执行
        │
Day 14-21: 等待 Scavenging 执行（最多7天）
        │
        │  ★ Day 17: 客户端突然回来了！发送 DDNS 注册
        │  ├── 服务器检查：当前处于 No-Refresh 吗？
        │  ├── No-Refresh 是基于旧时间戳(3/1) + 7天 = 3/8
        │  ├── 现在是 Day 17 (3/18)，早就过了 No-Refresh
        │  ├── → 刷新被接受！✅
        │  └── 时间戳更新为：3月18日 ← 全新的时间戳！
        │
        │  接下来会怎样？
        │  ┌──────────────────────────────┐
        │  │ 新时间戳 = 3月18日            │
        │  │ + No-Refresh 7天 = 3月25日    │
        │  │ + Refresh 7天 = 4月1日        │
        │  │                              │
        │  │ 记录不再 Eligible！            │
        │  │ 整个周期重新开始 🔄            │
        │  └──────────────────────────────┘
        │
Day 21: Scavenging 执行时
        ├── 检查该记录：时间戳 = 3/18（已被刷新）
        ├── 3/18 + 14天 = 4/1，还没过期
        └── 跳过！记录存活 ✅
```

**Eligible 期间不同操作的结果：**

| 情况 | 结果 |
|------|------|
| Eligible 期间客户端**回来刷新（Refresh）** | ✅ 时间戳更新，周期重启，记录被救 |
| Eligible 期间客户端**更换了 IP（Update）** | ✅ 这是 Update，任何时候都允许，时间戳也更新 |
| Eligible 期间**没人来** → Scavenging 执行 | 🗑️ 记录被删除 |

> 💡 **设计思想**：Scavenging 不是"一旦 Eligible 就立刻删"，而是有一个缓冲期（等待 Scavenging 周期到来）。在这段缓冲期内，客户端仍有机会"续命"。这也是为什么 No-Refresh 的判断是**基于记录当前时间戳**而非区域的固定时间窗口 — 旧时间戳的 No-Refresh 早已过期，所以刷新请求会被接受。

### 3.4 Scavenging 执行时的检查清单

当 DNS 服务器启动一次 Scavenging 时，它按以下顺序检查：

```
Scavenging 执行流程
═══════════════════

开始 Scavenging
    │
    ▼
[1] 区域是否启用了 Scavenging？ ─── 否 ──→ 跳过该区域
    │ 是
    ▼
[2] 区域是否启用了动态更新？ ─── 否 ──→ 跳过该区域
    │ 是
    ▼
[3] 当前服务器是否有权清理该区域？ ─── 否 ──→ 跳过
    │ 是                   (dnscmd /zoneresetscavengeservers)
    ▼
[4] "Zone can be scavenged after" 
    时间是否已过？ ─── 否 ──→ 跳过（给客户端和复制留时间）
    │ 是
    ▼
[5] AD 复制是否正常？ ─── 否 ──→ 跳过（防止误删）
    │ 是
    ▼
  ┌─────────────────────────────────┐
  │ 逐条检查区域内的每条记录：        │
  │                                 │
  │ 记录时间戳 = 0？                 │
  │ ├── 是 → 静态记录，跳过          │
  │ └── 否 ↓                        │
  │                                 │
  │ 当前时间 > 时间戳                 │
  │         + No-Refresh             │
  │         + Refresh ？             │
  │ ├── 否 → 记录未过期，跳过        │
  │ └── 是 → 删除该记录！🗑️          │
  └─────────────────────────────────┘
    │
    ▼
记录 Event Log:
  - Event ID 2501: 有记录被清理
  - Event ID 2502: 没有记录被清理
```

### 3.5 一个具体的时间计算例子

让我们用一个更具体的数字来做计算：

**环境设置**：
- No-Refresh 间隔：**3 天**
- Refresh 间隔：**3 天**
- 服务器 Scavenging 周期：**3 天**
- 上一次 Event ID 2501/2502：**1月1日 6:00 AM**
- 记录时间戳：**1月1日 12:00 PM（中午）**

**计算过程**：

```
记录时间戳:        1月1日 12:00 PM
+ No-Refresh (3天): 1月4日 12:00 PM   ← 这之前不允许刷新
+ Refresh (3天):    1月7日 12:00 PM   ← 这之后记录"可清理"

记录变为 Eligible: 1月7日 12:00 PM

上次 Scavenging:    1月1日 6:00 AM
+ 周期 (3天):       1月4日 6:00 AM    ← 第二次
+ 周期 (3天):       1月7日 6:00 AM    ← 第三次 (还早于 12:00 PM)
+ 周期 (3天):       1月10日 6:00 AM   ← 第四次 ✓ 此时记录已经 eligible

∴ 记录被删除时间: 约 1月10日 6:00 AM
```

从时间戳到实际被删除：**约 9 天**。

## 4. 关键配置与参数 (Key Configurations)

| 配置项 | 默认值 | 位置 | 说明 | 建议 |
|--------|--------|------|------|------|
| **记录时间戳** | 动态记录自动设置；静态为 0 | 记录属性 | 最后刷新/更新时间 | 不要对静态记录启用 scavenging |
| **No-Refresh Interval** | 7 天 | Zone → Aging | 禁止刷新的时间窗口 | 可适当缩短（如 3-4 天）以加快清理 |
| **Refresh Interval** | 7 天 | Zone → Aging | 允许刷新的时间窗口 | 保持默认或稍长，给客户端足够机会 |
| **Scavenge stale records** | ❌ 未启用 | Zone → Aging | 区域级开关 | 按需启用 |
| **Enable automatic scavenging** | ❌ 未启用 | Server → Advanced | 服务器级开关 | 只在 1 台 DNS 服务器上启用 |
| **Scavenging Period** | 7 天 | Server → Advanced | 多久执行一次 | 与 Refresh 间隔一致或更短 |

### PowerShell 配置命令

```powershell
# 查看区域的 Aging 设置
Get-DnsServerZoneAging -Name "contoso.com"

# 启用区域 Aging
Set-DnsServerZoneAging -Name "contoso.com" -Aging $true -NoRefreshInterval 7.00:00:00 -RefreshInterval 7.00:00:00

# 启用服务器 Scavenging
Set-DnsServerScavenging -ScavengingState $true -ScavengingInterval 7.00:00:00

# 手动触发 Scavenging（仍受安全阀限制）
Start-DnsServerScavenging

# 查看特定记录的时间戳
Get-DnsServerResourceRecord -ZoneName "contoso.com" -Name "laptop01" | Select-Object HostName, Timestamp, TimeToLive

# 限制只允许特定服务器 Scavenging
dnscmd dns-srv01.contoso.com /zoneresetscavengeservers contoso.com 10.1.1.10
```

## 5. 常见问题与排查 (Common Issues & Troubleshooting)

### 问题 A: Scavenging 没有工作，过期记录不被清理

**可能原因**：
1. 三层配置中某一层没开（最常见）
2. "Zone can be scavenged after" 时间还没到
3. 记录时间戳为 0（静态记录）
4. AD 复制有问题

**排查思路**：
```powershell
# 1. 检查区域 Aging 是否启用
Get-DnsServerZoneAging -Name "contoso.com"
# 看 AgingEnabled 是否为 True

# 2. 检查服务器 Scavenging 是否启用
Get-DnsServerScavenging
# 看 ScavengingState 是否为 True

# 3. 检查目标记录的时间戳
Get-DnsServerResourceRecord -ZoneName "contoso.com" -Name "target-host"
# 时间戳为 0 = 静态记录，不会被清理

# 4. 检查 Scavenging Event Log
Get-WinEvent -FilterHashtable @{LogName='DNS Server';Id=2501,2502} -MaxEvents 10
```

### 问题 B: 正常的记录被误删了（Over-Scavenging）

**可能原因**：
1. 管理员运行了 `dnscmd /ageallrecords`，给所有记录（包括静态记录）加了时间戳
2. No-Refresh + Refresh 间隔设置太短
3. 客户端 DDNS 注册有问题（如被防火墙阻止）
4. 记录的 ACL 权限不对，客户端无法更新

**排查思路**：
- 检查被删记录的原始所有者（Security tab）
- 检查客户端是否能成功执行 `ipconfig /registerdns`
- 检查 DNS 动态更新是否被安全策略阻止
- **临时措施**：把记录改为静态（时间戳设为 0）

### 问题 C: 时间戳不更新（不复制到其他 DNS 服务器）

**可能原因**：
- 区域 Scavenging 未启用时，时间戳更新**不会被复制**
- 这是设计行为：如果区域没开 Aging，时间戳无意义，不浪费复制带宽

**关键点**：一旦启用区域 Aging，时间戳才会开始正常复制。

### 问题 D: 首次启用 Scavenging 后大量记录被删

**原因**：历史遗留的过期记录一次性被清理  
**预防**：按照"三阶段启用法"（见下文实战经验）

## 6. 实战经验 (Practical Tips)

### 最佳实践

1. **只在 1 台 DNS 服务器上启用 Scavenging**
   - 因为区域数据是 AD 复制的，1 台清理即可
   - 便于集中查看 Event Log
   - 使用 `dnscmd /zoneresetscavengeservers` 精确控制

2. **三阶段安全启用法**（针对现有环境）

   ```
   阶段 1 — 准备（Setup）:
   ├── 关闭所有服务器的 Scavenging
   ├── 启用目标区域的 Aging（设好 No-Refresh 和 Refresh）
   └── 等待 No-Refresh + Refresh 时间过去（默认 14 天）
   
   阶段 2 — 健全检查（Sanity Check）:
   ├── 检查是否有时间戳过旧的记录
   ├── 如果有 → 排查 DDNS 注册问题，修复后再等一个周期
   └── 确认所有活跃客户端的记录时间戳都在合理范围内
   
   阶段 3 — 启用（Enable）:
   ├── 在 1 台服务器上启用 Scavenging
   ├── 创建一条测试记录，计算预期删除时间
   ├── 观察 Event ID 2501/2502
   └── 确认测试记录在预期时间被删除 ✓
   ```

3. **监控 Event Log**
   - **Event ID 2501**：有记录被清理，显示数量
   - **Event ID 2502**：执行了 Scavenging 但没有记录被清理
   - 通过这两个事件可以精确计算下次 Scavenging 时间

### 常见误区

| 误区 | 事实 |
|------|------|
| "启用 Zone Aging 就够了" | ❌ 还需要在服务器上启用 Scavenging |
| "手动 Scavenge 可以绕过安全检查" | ❌ 手动触发仍然受所有安全阀门限制 |
| "`dnscmd /ageallrecords` 很安全" | ⚠️ 这会给所有静态记录加时间戳，导致它们被清理！ |
| "Scavenging 应该在每台 DNS 服务器上开" | ❌ 建议只开 1 台，方便排查和控制 |
| "No-Refresh 期间记录完全不能更新" | ❌ 只有 Refresh（同 IP 续命）被拒绝，Update（IP 变更）不受限 |

### 安全注意

- **永远不要**在不理解的情况下运行 `dnscmd /ageallrecords`
- **备份 DNS 区域**后再启用 Scavenging
- 使用 `Export-DnsServerZone` 导出区域数据作为回滚手段
- SRV 记录和 _msdcs 区域的记录要特别小心

## 7. 与相关技术的对比 (Comparison with Related Technologies)

| 维度 | DNS Scavenging | DNS TTL (Time-to-Live) | DHCP Lease Cleanup |
|------|---------------|----------------------|-------------------|
| **作用对象** | DNS Server 上的资源记录 | DNS Client/Resolver 的缓存 | DHCP 地址分配 |
| **清理的是什么** | 服务器端的过期记录 | 客户端缓存的过期条目 | 过期的 IP 地址租约 |
| **谁来执行** | DNS Server | DNS Client/Resolver | DHCP Server |
| **触发方式** | 定时自动 + 可手动 | 缓存到期自动清除 | 租约到期自动回收 |
| **影响范围** | 所有查询该服务器的客户端 | 仅本地客户端 | IP 地址池管理 |
| **默认是否启用** | ❌ 默认不启用 | ✅ 始终生效 | ✅ 始终生效 |
| **风险** | 删错记录导致解析失败 | 缓存过期导致短暂查询延迟 | 地址回收导致 IP 冲突 |

> **关联说明**：DNS Scavenging 和 DHCP Lease 通常需要协调。理想情况下，DNS 记录的生命周期应该和 DHCP 租约周期匹配。如果 DHCP 租约 8 天，DNS Scavenging 的 No-Refresh + Refresh 最好 ≥ 8 天。

## 8. 参考资料 (References)

- [Set up DNS scavenging](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/dns-scavenging-setup) — 微软官方排查文档，包含完整的 Scavenging 设置示例和计算方法

---
---

# English Version

## 1. Overview

DNS Scavenging is a built-in mechanism in Windows DNS Server that **automatically cleans up stale (outdated) DNS resource records**. In Dynamic DNS (DDNS) environments, clients automatically register their A and PTR records. However, when clients are decommissioned, shut down, or change IP addresses, these old records don't automatically disappear. Over time, DNS zones accumulate "zombie records" (stale records).

Scavenging solves this problem: **it acts like a scheduled janitor, periodically checking timestamps on all records and deleting those that haven't been refreshed by clients for too long.**

Because deleting DNS records is a high-risk operation (deleting the wrong record could cause service outages), Windows has designed **multiple safety valves** into the scavenging mechanism. Configuration must be correct in three places:
1. **Resource Record** — The record must have a timestamp
2. **Zone** — Aging/Scavenging must be enabled on the zone
3. **Server** — At least one DNS server must have automatic scavenging enabled

> 🔑 **Key takeaway**: All three must be enabled for scavenging to work. If any one is missing, stale records won't be deleted.

## 2. Core Concepts

### 2.1 Timestamp

Every DNS resource record has a **timestamp field** recording when it was last "refreshed" or "updated."

- **Dynamically registered records**: Automatically have timestamps (Windows clients register with DNS every 24 hours by default)
- **Static records** (manually created by admins): Timestamp is **0**, meaning **never scavenged**
- Timestamp precision is **hourly** (rounded down to the nearest hour)

> 🏠 **Analogy**: The timestamp is like a hotel room's "checkout time." Each time the guest (client) extends their stay (refreshes the record), the checkout time is updated. If the guest leaves and never comes back, the room gets cleaned up after checkout.

### 2.2 No-Refresh Interval

The **No-Refresh Interval** (default: 7 days) is a window during which a record's timestamp **cannot be refreshed**. A "Refresh" means a dynamic update where the record data doesn't change — just touching the timestamp. If a client changes the IP of a host record, this is an "Update" and is exempt from the No-Refresh restriction.

**Why is this needed?** Because timestamp changes = AD replication traffic. The No-Refresh interval **reduces unnecessary replication**.

> 🏠 **Analogy**: No-Refresh is like a hotel saying "No need to call during the first 7 days — we know you're still staying."

### 2.3 Refresh Interval

The **Refresh Interval** (default: 7 days) follows immediately after No-Refresh. During this window, clients **can and should** refresh their timestamps.

If the client fails to refresh during the Refresh window, the record becomes **"Eligible for Scavenging."**

> 🏠 **Analogy**: The Refresh interval is the hotel's "stay confirmation period." You need to call within 7 days to say "I'm still here," or the hotel will assume you've left.

### 2.4 Scavenging Period

This is how often the DNS server runs the scavenging process (default: 7 days). Even if a record is "eligible," it won't be deleted until the next scavenging cycle runs.

### 2.5 Update vs Refresh

A **critical distinction** that is often confused:

| Operation | Meaning | Subject to No-Refresh? |
|-----------|---------|----------------------|
| **Update** | Record data changed (e.g., IP changed) | ❌ No — always allowed |
| **Refresh** | Record data unchanged, just updating timestamp | ✅ Yes — blocked during No-Refresh |

## 3. How It Works

### 3.1 Architecture

```
┌──────────────────────────────────────────────────────────┐
│              DNS Scavenging Three-Layer Architecture       │
├──────────────────────────────────────────────────────────┤
│                                                          │
│   Layer 1: Resource Record                               │
│   ┌─────────────────────────────────────┐               │
│   │ Record: app-server01.contoso.com    │               │
│   │ IP: 10.1.1.100                      │               │
│   │ Timestamp: 2026/03/01 10:00:00      │  ← timestamp  │
│   │ [x] Delete when stale               │               │
│   └─────────────────────────────────────┘               │
│                                                          │
│   Layer 2: Zone                                          │
│   ┌─────────────────────────────────────┐               │
│   │ Zone: contoso.com                   │               │
│   │ [x] Scavenge stale resource records │  ← enabled    │
│   │ No-Refresh: 7 days                  │               │
│   │ Refresh:    7 days                  │               │
│   │ Zone can be scavenged after:        │               │
│   │   2026/03/08 10:00:00               │  ← safety     │
│   └─────────────────────────────────────┘               │
│                                                          │
│   Layer 3: Server                                        │
│   ┌─────────────────────────────────────┐               │
│   │ Server: dns-srv01.contoso.com       │               │
│   │ [x] Enable automatic scavenging     │  ← enabled    │
│   │ Scavenging period: 7 days           │               │
│   └─────────────────────────────────────┘               │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 3.2 Complete Lifecycle — A Story-Based Example

Assume the contoso.com environment has:
- **No-Refresh Interval**: 7 days
- **Refresh Interval**: 7 days
- **Scavenging Period**: 7 days

The lifecycle of a DNS record for `laptop01.contoso.com` (IP: 10.1.1.50):

```
Timeline
═══════════════════════════════════════════════════════════════

Day 0 (March 1): Laptop boots up, registers via DDNS
├── Record created: laptop01.contoso.com → 10.1.1.50
├── Timestamp set to: March 1, 09:00
└── This is an "Update" (new record creation)

Day 1-7 (No-Refresh Window):
├── Laptop attempts registration daily (every 24h)
├── DNS Server: "No-Refresh period — timestamp NOT updated"
├── Timestamp remains: March 1, 09:00
└── Benefit: Reduced AD replication traffic ✓

Day 8-14 (Refresh Window):
├── Refresh period begins — refreshes now accepted
├── Day 8: Laptop registers → Timestamp updated to March 8, 09:00 ✓
├── Day 9-14: New No-Refresh cycle begins...
└── Timestamp now: March 8, 09:00

Day 15: Employee takes laptop away permanently
├── No one refreshes the record anymore
├── No-Refresh window (Day 8-14) passes normally
└── Timestamp frozen at: March 8, 09:00

Day 15-21 (Refresh Window, but no one comes):
├── Record enters Refresh period, waiting for client
├── ......no one comes......
└── Day 22: Refresh window expires!

Day 22: Record becomes "Eligible for Scavenging"
├── Current time > Timestamp(3/8) + No-Refresh(7d) + Refresh(7d)
├── i.e.: March 22 > March 8 + 14 days ✓
└── BUT! Record not deleted yet — must wait for scavenging cycle

Day 22-28: Waiting for Scavenging to Execute
├── Last scavenging ran on Day 21
├── Period is 7 days, next run on Day 28
└── 

Day 28: DNS Server executes Scavenging
├── Scans all records
├── Checks laptop01: Timestamp 3/8 + 14d = 3/22, expired ✓
├── Record deleted! 🗑️
├── Event ID 2501: "X records scavenged"
└── laptop01.contoso.com record is gone

═══════════════════════════════════════════════════════════════
Total time: At least 14 days + up to 7 days = 14-21 days after last refresh
```

### 3.3 After Eligible, Before Deletion — Can the Client Still "Save" the Record?

This is a critical but often overlooked scenario: **When a record has become Eligible for scavenging, but the scavenging cycle hasn't run yet, what happens if the client comes back and registers?**

The answer is: **Yes, the refresh is accepted! The record is "saved" and the entire cycle restarts.**

```
Timeline (assuming all 7-day intervals)
═════════════════════════════════════════════════════

Day 0:  Timestamp = March 1
        ├── No-Refresh begins (Day 0-7)
        │
Day 7:  No-Refresh ends
        ├── Refresh window begins (Day 7-14)
        │   Client can refresh, but doesn't come...
        │
Day 14: Refresh window expires
        ├── Record becomes "Eligible" for scavenging
        │   But NOT deleted yet — waiting for scavenging cycle
        │
Day 14-21: Waiting for Scavenging to execute (up to 7 days)
        │
        │  ★ Day 17: Client suddenly comes back! Sends DDNS registration
        │  ├── Server checks: Are we in No-Refresh?
        │  ├── No-Refresh = old timestamp (3/1) + 7 days = 3/8
        │  ├── Current date is Day 17 (3/18), well past No-Refresh
        │  ├── → Refresh accepted! ✅
        │  └── Timestamp updated to: March 18 ← brand new timestamp!
        │
        │  What happens next?
        │  ┌──────────────────────────────────┐
        │  │ New timestamp = March 18          │
        │  │ + No-Refresh 7 days = March 25    │
        │  │ + Refresh 7 days = April 1        │
        │  │                                   │
        │  │ Record is NO LONGER Eligible!      │
        │  │ Entire cycle restarts 🔄           │
        │  └──────────────────────────────────┘
        │
Day 21: When Scavenging executes
        ├── Checks this record: Timestamp = 3/18 (refreshed!)
        ├── 3/18 + 14 days = 4/1, not yet expired
        └── Skipped! Record survives ✅
```

**Results of different actions during the Eligible period:**

| Scenario | Result |
|----------|--------|
| Client **comes back and refreshes** during Eligible period | ✅ Timestamp updated, cycle restarts, record saved |
| Client **changes IP (Update)** during Eligible period | ✅ Updates are always allowed, timestamp also updated |
| **No one comes** → Scavenging executes | 🗑️ Record deleted |

> 💡 **Design philosophy**: Scavenging doesn't delete immediately upon Eligible status — there's a buffer period (waiting for the scavenging cycle). During this buffer, clients still have a chance to "save" their records. This is why the No-Refresh check is based on the **record's current timestamp** rather than a fixed zone-wide window — the old timestamp's No-Refresh has long expired, so refresh requests are accepted.

### 3.4 Concrete Calculation Example

**Environment**:
- No-Refresh Interval: **3 days**
- Refresh Interval: **3 days**
- Server Scavenging Period: **3 days**
- Last Event ID 2501/2502: **Jan 1, 6:00 AM**
- Record Timestamp: **Jan 1, 12:00 PM (noon)**

**Calculation**:

```
Record Timestamp:       Jan 1, 12:00 PM
+ No-Refresh (3 days):  Jan 4, 12:00 PM  ← No refresh allowed before this
+ Refresh (3 days):     Jan 7, 12:00 PM  ← Record eligible after this

Record becomes eligible: Jan 7, 12:00 PM

Last Scavenging:         Jan 1, 6:00 AM
+ Period (3 days):       Jan 4, 6:00 AM   ← 2nd cycle
+ Period (3 days):       Jan 7, 6:00 AM   ← 3rd cycle (before 12:00 PM)
+ Period (3 days):       Jan 10, 6:00 AM  ← 4th cycle ✓ record is now eligible

∴ Record deleted at: approximately Jan 10, 6:00 AM
```

From timestamp to actual deletion: **approximately 9 days**.

## 4. Key Configurations

| Setting | Default | Location | Description | Recommendation |
|---------|---------|----------|-------------|---------------|
| **Record Timestamp** | Auto for dynamic; 0 for static | Record Properties | Last refresh/update time | Don't enable scavenging on static records |
| **No-Refresh Interval** | 7 days | Zone → Aging | Window during which refresh is blocked | Can shorten (e.g., 3-4 days) for faster cleanup |
| **Refresh Interval** | 7 days | Zone → Aging | Window during which refresh is allowed | Keep default or slightly longer |
| **Scavenge stale records** | ❌ Disabled | Zone → Aging | Zone-level switch | Enable as needed |
| **Enable automatic scavenging** | ❌ Disabled | Server → Advanced | Server-level switch | Enable on only 1 DNS server |
| **Scavenging Period** | 7 days | Server → Advanced | How often scavenging runs | Match or shorter than Refresh interval |

### PowerShell Commands

```powershell
# View zone aging settings
Get-DnsServerZoneAging -Name "contoso.com"

# Enable zone aging
Set-DnsServerZoneAging -Name "contoso.com" -Aging $true -NoRefreshInterval 7.00:00:00 -RefreshInterval 7.00:00:00

# Enable server scavenging
Set-DnsServerScavenging -ScavengingState $true -ScavengingInterval 7.00:00:00

# Manually trigger scavenging (still subject to safety checks)
Start-DnsServerScavenging

# Check specific record timestamp
Get-DnsServerResourceRecord -ZoneName "contoso.com" -Name "laptop01" | Select-Object HostName, Timestamp, TimeToLive

# Restrict scavenging to specific servers
dnscmd dns-srv01.contoso.com /zoneresetscavengeservers contoso.com 10.1.1.10
```

## 5. Common Issues & Troubleshooting

### Issue A: Scavenging Not Working — Stale Records Not Deleted

**Possible causes**:
1. One of the three layers is not enabled (most common)
2. "Zone can be scavenged after" time hasn't elapsed
3. Record timestamp is 0 (static record)
4. AD replication issues

**Troubleshooting**:
```powershell
# 1. Check zone aging
Get-DnsServerZoneAging -Name "contoso.com"

# 2. Check server scavenging
Get-DnsServerScavenging

# 3. Check target record timestamp
Get-DnsServerResourceRecord -ZoneName "contoso.com" -Name "target-host"

# 4. Check Scavenging Event Log
Get-WinEvent -FilterHashtable @{LogName='DNS Server';Id=2501,2502} -MaxEvents 10
```

### Issue B: Valid Records Deleted (Over-Scavenging)

**Possible causes**:
1. Admin ran `dnscmd /ageallrecords` — sets timestamps on ALL records including static ones
2. No-Refresh + Refresh intervals set too short
3. Client DDNS registration failing (firewall, permissions)
4. Record ACL prevents client from updating

### Issue C: Timestamps Not Replicating

**Key insight**: When zone scavenging is **not enabled**, timestamp updates are **not replicated**. This is by design — if aging isn't enabled, timestamps are irrelevant, so replication bandwidth is saved.

## 6. Practical Tips

### Best Practices

1. **Enable scavenging on only 1 DNS server** — zone data is AD-replicated, so one server cleaning is sufficient
2. **Use the three-phase safe enablement** for existing environments (Setup → Sanity Check → Enable)
3. **Monitor Event IDs 2501/2502** — these tell you exactly when scavenging runs and how many records are cleaned

### Common Misconceptions

| Misconception | Reality |
|--------------|---------|
| "Enabling Zone Aging is enough" | ❌ Server scavenging must also be enabled |
| "Manual Scavenge bypasses safety checks" | ❌ Still subject to all safety valves |
| "`dnscmd /ageallrecords` is safe" | ⚠️ Timestamps ALL records including static — they will be scavenged! |
| "Enable scavenging on every DNS server" | ❌ Recommend only 1 server for easier management |
| "No-Refresh blocks all record changes" | ❌ Only Refresh (same-IP timestamp update) is blocked; Updates (IP change) always allowed |

## 7. Comparison with Related Technologies

| Dimension | DNS Scavenging | DNS TTL (Time-to-Live) | DHCP Lease Cleanup |
|-----------|---------------|----------------------|-------------------|
| **Target** | Resource records on DNS Server | Cache entries on DNS Client/Resolver | DHCP address assignments |
| **What's cleaned** | Stale server-side records | Expired client-side cache | Expired IP leases |
| **Executed by** | DNS Server | DNS Client/Resolver | DHCP Server |
| **Trigger** | Scheduled + manual | Automatic on expiry | Automatic on expiry |
| **Default enabled** | ❌ No | ✅ Always | ✅ Always |
| **Risk** | Wrong deletion → resolution failure | Cache expiry → brief query delay | Address reclaim → IP conflict |

## 8. References

- [Set up DNS scavenging](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/dns-scavenging-setup) — Official Microsoft troubleshooting doc with complete setup example and calculation method
