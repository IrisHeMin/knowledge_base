---
layout: post
title: "Deep Dive: 使用 Windows Firewall 限制服务器 Inbound/Outbound 流量"
date: 2026-03-29
categories: [Knowledge, Networking]
tags: [windows-firewall, rdp, active-directory, kerberos, security, powershell]
type: "deep-dive"
---

# Deep Dive: 使用 Windows Firewall 限制服务器 Inbound/Outbound 流量

**Topic:** Windows Firewall Advanced Security — Inbound/Outbound 白名单策略  
**Category:** Networking / Security  
**Level:** 中级  
**Last Updated:** 2026-03-29

---

## 1. 概述 (Overview)

在企业环境中，某些高敏感服务器（如跳板机、安全审计服务器、或特定业务服务器）需要严格控制网络流量：

- **Inbound**：仅允许特定管理员 IP 通过 RDP 连入
- **Outbound**：仅允许与 Domain Controller (DC) 的认证流量，以及 RDP 到特定目标机器，其他所有出站流量全部阻断

Windows Firewall with Advanced Security (WFAS) 是 Windows 系统内置的主机防火墙，可以通过将默认策略设为 **Block**，再添加白名单 **Allow** 规则来实现这种"最小权限"网络隔离。本文从实战出发，完整梳理实现思路、所需端口、PowerShell 命令和注意事项。

## 2. 核心概念 (Core Concepts)

### 默认行为 (Default Action)

Windows Firewall 有三个 Profile：**Domain**、**Private**、**Public**。每个 Profile 分别有 Inbound 和 Outbound 的默认行为：

- **DefaultInboundAction**：默认为 `Block`（已经是封锁状态）
- **DefaultOutboundAction**：默认为 `Allow`（默认放行所有出站流量）

> 关键认知：大多数管理员只关注 Inbound 封锁，而忽略 Outbound 默认是全开的。要实现真正的流量隔离，**必须将 Outbound 默认行为改为 Block**。

### 规则优先级 (Rule Precedence)

1. 显式 Allow 规则优先于默认 Block 设置
2. 显式 Block 规则优先于任何 Allow 规则
3. 更具体的规则优先于更宽泛的规则

这意味着：当 Default Action 为 Block 时，只有明确创建的 Allow 规则才能放行流量 —— 这正是"白名单"策略的基础。

### DC 认证所需端口

一台 Domain-joined 的机器要完成域登录认证，必须与 DC 通信。涉及的协议和端口不仅仅是 Kerberos，还包括 DNS、LDAP、SMB、RPC 等一整套协议栈。如果 Outbound 设为 Block 却遗漏了某个端口，会导致域登录失败、组策略无法下发等问题。

## 3. 工作原理 (How It Works)

### 整体策略架构

```
┌─────────────────────────────────────────────────────┐
│              Target Server (被管控的机器)              │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │       Windows Firewall (WFAS)                │   │
│  │                                              │   │
│  │  Default Inbound:  BLOCK                     │   │
│  │  Default Outbound: BLOCK                     │   │
│  │                                              │   │
│  │  Inbound Allow Rules:                        │   │
│  │    ✅ TCP 3389 from Admin IPs only           │   │
│  │                                              │   │
│  │  Outbound Allow Rules:                       │   │
│  │    ✅ DC 认证端口 (88,53,389,636,445,       │   │
│  │       135,49152-65535,123,3268) → DC IP      │   │
│  │    ✅ TCP 3389 → 允许 RDP 到的目标 IP        │   │
│  │    ❌ 其他所有流量 → BLOCKED                  │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
         │                    │                 │
    ▼ Inbound            ▼ Outbound         ▼ Outbound
 Admin RDP 进入       认证流量到 DC     RDP 到目标机器
```

### 详细实施流程

#### Step 1: 设置默认策略为 Block

```powershell
# 将三个 Profile 的 Inbound 和 Outbound 默认行为都设为 Block
Set-NetFirewallProfile -Profile Domain,Private,Public `
    -DefaultInboundAction Block `
    -DefaultOutboundAction Block
```

> ⚠️ **关键提醒**：一定要先创建好所有 Allow 规则，最后再执行这条命令。否则执行完毕瞬间，你的 RDP 连接就会断开，把自己锁在外面。

#### Step 2: 创建 Inbound 规则 — 仅允许特定 IP RDP 进入

```powershell
New-NetFirewallRule -DisplayName "Allow RDP from Trusted Admin Hosts" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 3389 `
    -RemoteAddress "10.0.1.10,10.0.1.11,10.0.2.0/24" `
    -Action Allow `
    -Profile Domain
```

- `RemoteAddress` 支持单个 IP、逗号分隔多个 IP、CIDR 子网
- 根据需要选择 Profile（Domain、Private、Public 或组合）

#### Step 3: 创建 Outbound 规则 — 允许到 DC 的认证流量

```powershell
$DC = "10.0.0.5"  # Domain Controller 的 IP

# Kerberos (认证核心)
New-NetFirewallRule -DisplayName "Allow Kerberos TCP to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 88 `
    -RemoteAddress $DC -Action Allow
New-NetFirewallRule -DisplayName "Allow Kerberos UDP to DC" `
    -Direction Outbound -Protocol UDP -RemotePort 88 `
    -RemoteAddress $DC -Action Allow

# DNS (域认证依赖 SRV 记录定位 DC)
New-NetFirewallRule -DisplayName "Allow DNS TCP to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 53 `
    -RemoteAddress $DC -Action Allow
New-NetFirewallRule -DisplayName "Allow DNS UDP to DC" `
    -Direction Outbound -Protocol UDP -RemotePort 53 `
    -RemoteAddress $DC -Action Allow

# LDAP (域查询)
New-NetFirewallRule -DisplayName "Allow LDAP TCP to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 389 `
    -RemoteAddress $DC -Action Allow
New-NetFirewallRule -DisplayName "Allow LDAP UDP to DC" `
    -Direction Outbound -Protocol UDP -RemotePort 389 `
    -RemoteAddress $DC -Action Allow

# LDAPS (加密 LDAP，按需启用)
New-NetFirewallRule -DisplayName "Allow LDAPS to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 636 `
    -RemoteAddress $DC -Action Allow

# SMB (Group Policy / SYSVOL 下发)
New-NetFirewallRule -DisplayName "Allow SMB to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 445 `
    -RemoteAddress $DC -Action Allow

# RPC Endpoint Mapper + 动态端口 (域复制、策略同步)
New-NetFirewallRule -DisplayName "Allow RPC EPM to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 135 `
    -RemoteAddress $DC -Action Allow
New-NetFirewallRule -DisplayName "Allow RPC Dynamic to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 49152-65535 `
    -RemoteAddress $DC -Action Allow

# NTP 时间同步 (Kerberos 依赖时钟同步，默认容差 5 分钟)
New-NetFirewallRule -DisplayName "Allow NTP to DC" `
    -Direction Outbound -Protocol UDP -RemotePort 123 `
    -RemoteAddress $DC -Action Allow

# Global Catalog (多域/跨域环境需要)
New-NetFirewallRule -DisplayName "Allow Global Catalog to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 3268 `
    -RemoteAddress $DC -Action Allow

# Kerberos Password Change (密码修改场景)
New-NetFirewallRule -DisplayName "Allow Kerberos PW Change to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 464 `
    -RemoteAddress $DC -Action Allow
New-NetFirewallRule -DisplayName "Allow Kerberos PW Change UDP to DC" `
    -Direction Outbound -Protocol UDP -RemotePort 464 `
    -RemoteAddress $DC -Action Allow
```

#### Step 4: 创建 Outbound 规则 — 允许 RDP 到特定目标机器

```powershell
New-NetFirewallRule -DisplayName "Allow RDP Out to Trusted Hosts" `
    -Direction Outbound `
    -Protocol TCP `
    -RemotePort 3389 `
    -RemoteAddress "10.0.2.20,10.0.2.21,10.0.3.0/24" `
    -Action Allow `
    -Profile Domain
```

## 4. 关键配置与参数 (Key Configurations)

### DC 认证所需端口完整列表

| 端口 | 协议 | 用途 | 必要性 |
|------|------|------|--------|
| 88 | TCP/UDP | Kerberos 认证 | ✅ 必须 — 域认证核心 |
| 53 | TCP/UDP | DNS 查询 | ✅ 必须 — SRV 记录定位 DC |
| 389 | TCP/UDP | LDAP | ✅ 必须 — 域信息查询 |
| 636 | TCP | LDAPS (加密 LDAP) | 🔶 按需 — 如果使用 LDAPS |
| 445 | TCP | SMB | ✅ 必须 — GPO/SYSVOL |
| 135 | TCP | RPC Endpoint Mapper | ✅ 必须 — RPC 服务定位 |
| 49152-65535 | TCP | RPC 动态端口 | ✅ 必须 — LSA/SAM/Netlogon |
| 123 | UDP | NTP 时间同步 | ✅ 必须 — Kerberos 时钟同步 |
| 3268 | TCP | Global Catalog | 🔶 按需 — 多域环境 |
| 3269 | TCP | Global Catalog SSL | 🔶 按需 — 多域环境 + SSL |
| 464 | TCP/UDP | Kerberos Password Change | 🔶 按需 — 密码修改场景 |

### PowerShell 关键 cmdlet

| cmdlet | 用途 |
|--------|------|
| `Get-NetFirewallProfile` | 查看当前 Profile 设置 |
| `Set-NetFirewallProfile` | 修改 Profile 默认行为 |
| `New-NetFirewallRule` | 创建新规则 |
| `Get-NetFirewallRule` | 查看已有规则 |
| `Set-NetFirewallRule` | 修改已有规则 |
| `Remove-NetFirewallRule` | 删除规则 |
| `Get-NetFirewallRule -DisplayName "xxx" \| Get-NetFirewallAddressFilter` | 查看规则的地址过滤条件 |

## 5. 常见问题与排查 (Common Issues & Troubleshooting)

### 问题 A: 改完策略后 RDP 断连，无法重新登录

- **原因**：先改了 Default Outbound 为 Block，但还没创建 Allow 规则，或者遗漏了自己的 IP
- **排查思路**：
  1. 如果是 Azure VM，通过 Azure Portal 的 **Run Command** 或 **Serial Console** 恢复
  2. 如果是物理机/本地 VM，通过 Console 登录后恢复
- **恢复命令**：
  ```powershell
  Set-NetFirewallProfile -Profile Domain,Private,Public -DefaultOutboundAction Allow
  ```

### 问题 B: Outbound Block 后域登录失败

- **原因**：遗漏了某个 DC 认证必要端口
- **排查思路**：
  1. 在目标机器上开启 Firewall 审计日志：
     ```powershell
     auditpol /set /subcategory:"Filtering Platform Packet Drop" /success:enable /failure:enable
     ```
  2. 查看 Windows Event Log `Security` 中的 Event ID **5152**（被 WFP 丢弃的包），分析被 Block 的目标端口
  3. 根据被 Block 的端口补充 Allow 规则
- **关键工具**：Event Viewer → Security Log → Filter by Event ID 5152

### 问题 C: 组策略不更新

- **原因**：可能遗漏了 SMB (445) 或 RPC (135 + 动态端口) 的出站规则
- **排查思路**：
  1. 手动触发 `gpupdate /force` 观察报错
  2. 检查 445 和 135 端口到 DC 的连通性：
     ```powershell
     Test-NetConnection -ComputerName 10.0.0.5 -Port 445
     Test-NetConnection -ComputerName 10.0.0.5 -Port 135
     ```
  3. 补充遗漏的规则

### 问题 D: 时间偏移导致 Kerberos 认证失败

- **原因**：遗漏了 NTP (UDP 123) 出站规则，机器时间与 DC 偏差超过 5 分钟
- **症状**：Event Log 中出现 Kerberos 相关错误，提示 "clock skew too great"
- **修复**：添加 NTP 出站规则，手动同步时间 `w32tm /resync`

## 6. 实战经验 (Practical Tips)

### 最佳实践

- **先建规则，后改策略**：永远先把所有 Allow 规则创建好，确认无误后再将 Default Action 改为 Block
- **脚本化部署**：将所有规则写成一个 PowerShell 脚本，方便批量部署和灾难恢复
- **规则命名规范**：使用清晰的 DisplayName（如 `Allow Kerberos TCP to DC`），便于后续审计和管理
- **多 DC 环境**：如果有多台 DC，`RemoteAddress` 参数中列出所有 DC 的 IP

### 常见误区

- ❌ **只改 Inbound 不改 Outbound**：达不到真正的网络隔离目的
- ❌ **Outbound 放行所有到 DC 的流量**：应该精确到端口，不要用 `Any` port
- ❌ **忘记 DNS 端口**：Kerberos 依赖 DNS SRV 记录定位 DC，没有 DNS 就无法找到 DC
- ❌ **忘记 NTP 端口**：Kerberos 对时钟同步有严格要求（默认 5 分钟容差）

### 安全注意

- RPC 动态端口范围 (49152-65535) 较大，如果安全要求极高，可在 DC 上通过注册表限制 RPC 使用的端口范围，然后在防火墙上缩小对应范围
- 定期审计防火墙规则：`Get-NetFirewallRule | Where-Object {$_.Enabled -eq 'True'} | Format-Table DisplayName, Direction, Action`
- 如果服务器在 Azure 上，**NSG 规则需与 Windows Firewall 规则同步配置**，双层防护

### 验证方法

配置完成后，建议进行以下验证：

```powershell
# 1. 确认 Profile 默认策略
Get-NetFirewallProfile | Format-Table Name, DefaultInboundAction, DefaultOutboundAction

# 2. 列出所有启用的规则
Get-NetFirewallRule | Where-Object {$_.Enabled -eq 'True'} |
    Format-Table DisplayName, Direction, Action, Profile

# 3. 测试 DC 连通性
Test-NetConnection -ComputerName 10.0.0.5 -Port 88
Test-NetConnection -ComputerName 10.0.0.5 -Port 53
Test-NetConnection -ComputerName 10.0.0.5 -Port 389
Test-NetConnection -ComputerName 10.0.0.5 -Port 445

# 4. 测试域认证
nltest /dsgetdc:contoso.com

# 5. 测试组策略更新
gpupdate /force
```

## 7. 通过 GPO 集中推送防火墙规则 (Deploy via Group Policy)

### 配置位置

在 **Group Policy Management Console (GPMC)** 中：

```
Computer Configuration
  → Policies
    → Windows Settings
      → Security Settings
        → Windows Firewall with Advanced Security
```

在这个节点下可以看到和本地 `wf.msc` 一样的界面，可配置 Inbound/Outbound Rules 和 Profile 默认行为。

### 推到机器上的存储位置

| 类型 | 注册表路径 |
|------|-----------|
| **GPO 推送的规则** | `HKLM\SOFTWARE\Policies\Microsoft\WindowsFirewall\FirewallRules` |
| **GPO Profile 设置** | `HKLM\SOFTWARE\Policies\Microsoft\WindowsFirewall\DomainProfile` 等 |
| **本地规则** | `HKLM\SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy\FirewallRules` |

### GPO 规则特点

- GPO 推送的规则在 `wf.msc` 中显示为 **灰色（不可编辑）**
- 本地管理员只能查看，不能修改或删除 GPO 规则
- 默认每 **90 分钟**后台刷新（随机偏移 0-30 分钟）
- 防火墙监控注册表变更 → 通知 **Windows Filtering Platform (WFP)** → 读取所有规则 → 应用新 filter → 移除旧 filter

### 验证命令

```powershell
# 强制刷新 GPO
gpupdate /force

# 查看 GPO 推送的规则
Get-NetFirewallRule -PolicyStore ActiveStore |
    Where-Object { $_.PolicyStoreSource -ne "PersistentStore" } |
    Format-Table DisplayName, Direction, Action

# 直接查看注册表
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\WindowsFirewall\FirewallRules"
```

## 8. 通过 Intune (MDM) 推送防火墙规则 (Deploy via Intune)

### 配置位置

在 **Microsoft Intune admin center** 中：

```
Endpoint Security
  → Firewall
    → Create Policy
      Platform: Windows
      Profile:
        ├── Windows Firewall        ← 配置 Profile 级别设置（默认行为）
        └── Windows Firewall Rules  ← 配置具体规则（每个策略最多 150 条）
```

每条规则可配的字段：

| 字段 | 说明 | 示例 |
|------|------|------|
| Name | 规则名称 | Allow RDP from Admin |
| Direction | Inbound / Outbound | Inbound |
| Action | Allow / Block | Allow |
| Protocol | TCP (6) / UDP (17) | 6 |
| Local Port Ranges | 本地端口 | 3389 |
| Remote Port Ranges | 远程端口 | 3389 |
| Remote Address Ranges | 远程 IP | 10.0.1.10,10.0.1.11 |
| Profile | Domain/Private/Public | Domain |

### 推到机器上的存储位置

Intune 通过 **Firewall CSP** (`./Vendor/MSFT/Firewall/MdmStore`) 下发规则：

| 类型 | 注册表路径 |
|------|-----------|
| **Intune (MDM) 规则** | `HKLM\SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy\Mdm\FirewallRules` |
| **Intune Profile 设置** | `...\FirewallPolicy\Mdm\DomainProfile` 等 |

### 验证命令

```powershell
# 查看 MDM 推送的规则（注册表）
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy\Mdm\FirewallRules"

# 用 PowerShell 查看所有规则及来源
Get-NetFirewallRule | Select-Object DisplayName, Direction, Action, PolicyStoreSource |
    Format-Table -AutoSize
# PolicyStoreSource 值：
#   PersistentStore  = 本地规则
#   YOURPC\MDM       = Intune 推送
#   GroupPolicy       = GPO 推送

# 查看 Intune 同步状态
dsregcmd /status
```

### Intune 规则特点

- 在 `wf.msc` 中也显示为 **灰色（不可编辑）**，某些版本可能不直接显示，需用 PowerShell 查看
- 每个策略最多 150 条规则，可叠加多个策略
- 默认每 **8 小时**同步一次（或手动触发 Sync）

## 9. 规则优先级深度解析 (Rule Precedence Deep Dive)

当 GPO、Intune、本地同时存在防火墙规则，且规则之间有重叠或冲突时，优先级如何判定？

### 维度一：同一来源内 — Block vs Allow

**不管规则来自 GPO、Intune 还是本地，同一层内的优先级逻辑一致：**

```
① 显式 Block 规则  >  ② 显式 Allow 规则  >  ③ 默认行为 (Default Action)
```

| 优先级 | 规则类型 | 说明 |
|--------|---------|------|
| **最高** | 显式 Block 规则 | 只要有 Block，无论有没有 Allow，流量都被阻断 |
| **中** | 显式 Allow 规则 | 在没有 Block 冲突的前提下放行 |
| **最低** | Default Action | 没有任何匹配规则时才使用默认行为 |

> ⚠️ **核心原则：Block 永远赢 Allow**。如果同一个流量同时匹配了一条 Block 规则和一条 Allow 规则，Block 胜出。

另外：**更具体的规则优先于更宽泛的规则**（例如单 IP 规则优先于 IP 范围规则），但显式 Block 仍然优先于任何 Allow。

### 维度二：不同来源之间 — GPO vs Intune vs 本地

```
┌─────────────────────────────────────────────────┐
│           最终生效的规则集 (Active Store)          │
│                                                 │
│   评估顺序（所有来源的规则合并后统一评估）：         │
│                                                 │
│   1. GPO/MDM 的 Block 规则     ← 最高优先级      │
│   2. GPO/MDM 的 Allow 规则                       │
│   3. 本地的 Block 规则 *                          │
│   4. 本地的 Allow 规则 *                          │
│   5. Default Action                             │
│                                                 │
│   * 前提：AllowLocalPolicyMerge = true（默认）   │
│     如果设为 false，本地规则被完全忽略             │
└─────────────────────────────────────────────────┘
```

### AllowLocalPolicyMerge 的影响

| AllowLocalPolicyMerge | 本地规则的行为 |
|----------------------|---------------|
| **true**（默认） | 本地规则与 GPO/MDM 规则**合并**，但 GPO/MDM 的 Block 仍优先 |
| **false** | 本地规则**完全被忽略**，只有 GPO/MDM 推送的规则生效 |

可通过 GPO 或 Intune CSP 设置：
- **GPO**：Windows Firewall Properties → Domain Profile → Settings → Rule merging → Apply local firewall rules: **No**
- **Intune CSP**：`./Vendor/MSFT/Firewall/MdmStore/DomainProfile/AllowLocalPolicyMerge` 设为 **0**

### 冲突示例

假设有以下规则：

| 来源 | 规则 | 方向 | 端口 | Action |
|------|------|------|------|--------|
| GPO | Rule A | Inbound | TCP 3389 from 10.0.1.0/24 | **Allow** |
| Intune | Rule B | Inbound | TCP 3389 from Any | **Block** |
| 本地 | Rule C | Inbound | TCP 3389 from 10.0.1.5 | **Allow** |

**结果**：
- 来自 `10.0.1.5` 的 RDP → ❌ **被 Block**（Intune 的显式 Block 优先于所有 Allow）
- 来自 `10.0.1.100` 的 RDP → ❌ **被 Block**（同理）
- **Block 永远赢**，不管 Allow 来自 GPO 还是本地

### GPO 与 Intune 之间的关系

- GPO 和 Intune 推送的规则属于**同等优先级**
- 如果两者推送**冲突的设置**（如同一个 Profile 设置不同的默认行为），结果不确定
- **微软建议：选择一种管理方式**，不要 GPO 和 Intune 同时管理防火墙

## 10. 与相关技术的对比 (Comparison with Related Technologies)

| 维度 | Windows Firewall (WFAS) | Azure NSG | GPO Firewall Rules | Intune Firewall |
|------|------------------------|-----------|-------------------|-----------------|
| 作用层级 | 主机级别 (Host-based) | 网络级别 (VNet/Subnet/NIC) | 主机级别，域统一下发 | 主机级别，云统一下发 |
| 配置方式 | PowerShell / GUI / netsh | Azure Portal / CLI / ARM | Group Policy Editor | Intune Admin Center |
| 粒度 | 端口 + IP + 程序 + 服务 | 端口 + IP + Service Tag | 同 WFAS | 同 WFAS |
| 适用设备 | 任何 Windows | Azure VM | Domain-joined | Entra ID joined / Intune enrolled |
| 适用场景 | 单机隔离 | Azure VM 网络隔离 | 域内批量统一策略 | 云管理设备统一策略 |
| 规则存储 | 本地注册表 | Azure 平台 | `HKLM\SOFTWARE\Policies\...` | `...\FirewallPolicy\Mdm\...` |
| 是否可叠加 | ✅ | ✅ 与 WFAS 叠加 | ✅ 通过 GPO 下发到 WFAS | ✅ 通过 CSP 下发到 WFAS |

**选型建议**：
- 单台服务器快速隔离 → WFAS + PowerShell
- Azure VM → NSG + WFAS 双层防护
- 域内批量管控 → GPO 下发 Firewall Rules
- 云管理设备 → Intune Endpoint Security Firewall Policy
- ⚠️ 不建议 GPO 和 Intune 同时管理防火墙，避免冲突

## 11. 参考资料 (References)

- [Manage Windows Firewall with the command line](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/configure-with-command-line) — PowerShell 和 netsh 管理 Windows Firewall 的完整参考
- [Windows Firewall rules](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/rules) — 规则优先级、Local Policy Merge 和规则类型的官方说明
- [Best practices for configuring Windows Firewall](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/best-practices-configuring) — GUI 和高级配置示例
- [How to configure a firewall for AD domains and trusts](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/config-firewall-for-ad-domains-and-trusts) — AD 域认证所需端口的权威列表 (KB179442)
- [Windows Firewall tools](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/tools) — GPO 处理机制和注册表存储说明
- [Firewall policy for endpoint security in Intune](https://learn.microsoft.com/en-us/mem/intune/protect/endpoint-security-firewall-policy) — Intune 防火墙策略配置指南
- [Firewall CSP](https://learn.microsoft.com/en-us/windows/client-management/mdm/firewall-csp) — Intune MDM 防火墙 CSP 详细参考

---
---

# Deep Dive: Restricting Server Inbound/Outbound Traffic with Windows Firewall

**Topic:** Windows Firewall Advanced Security — Inbound/Outbound Whitelist Policy  
**Category:** Networking / Security  
**Level:** Intermediate  
**Last Updated:** 2026-03-29

---

## 1. Overview

In enterprise environments, certain high-sensitivity servers (jump boxes, security audit servers, or dedicated business servers) require strict network traffic control:

- **Inbound**: Only allow specific administrator IPs to RDP in
- **Outbound**: Only allow authentication traffic to Domain Controllers (DC), plus RDP to specific target machines — block everything else

Windows Firewall with Advanced Security (WFAS) is the built-in host firewall in Windows. By setting the default policy to **Block** and adding whitelist **Allow** rules, you can achieve this "least privilege" network isolation. This article provides the complete implementation walkthrough including required ports, PowerShell commands, and key considerations.

## 2. Core Concepts

### Default Action

Windows Firewall has three Profiles: **Domain**, **Private**, and **Public**. Each profile has separate Inbound and Outbound default actions:

- **DefaultInboundAction**: Defaults to `Block` (already restrictive)
- **DefaultOutboundAction**: Defaults to `Allow` (all outbound traffic is permitted by default)

> Key insight: Most administrators focus on inbound blocking while overlooking that outbound is wide open by default. To achieve true traffic isolation, **you must change the Outbound default action to Block**.

### Rule Precedence

1. Explicitly defined Allow rules take precedence over the default Block setting
2. Explicit Block rules take precedence over any conflicting Allow rules
3. More specific rules take precedence over less specific rules

This means: when Default Action is Block, only explicitly created Allow rules can permit traffic — this is the foundation of a "whitelist" strategy.

### Ports Required for DC Authentication

A domain-joined machine needs to communicate with the DC for domain login authentication. The required protocols go beyond just Kerberos — they include DNS, LDAP, SMB, RPC, and more. Missing any port when Outbound is set to Block will cause domain login failures, Group Policy sync issues, etc.

## 3. How It Works

### Overall Architecture

```
┌─────────────────────────────────────────────────────┐
│              Target Server (Restricted Machine)       │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │       Windows Firewall (WFAS)                │   │
│  │                                              │   │
│  │  Default Inbound:  BLOCK                     │   │
│  │  Default Outbound: BLOCK                     │   │
│  │                                              │   │
│  │  Inbound Allow Rules:                        │   │
│  │    ✅ TCP 3389 from Admin IPs only           │   │
│  │                                              │   │
│  │  Outbound Allow Rules:                       │   │
│  │    ✅ DC auth ports (88,53,389,636,445,      │   │
│  │       135,49152-65535,123,3268) → DC IP      │   │
│  │    ✅ TCP 3389 → Allowed RDP targets         │   │
│  │    ❌ All other traffic → BLOCKED            │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### Implementation Steps

#### Step 1: Set Default Policy to Block

```powershell
Set-NetFirewallProfile -Profile Domain,Private,Public `
    -DefaultInboundAction Block `
    -DefaultOutboundAction Block
```

> ⚠️ **Critical**: Always create all Allow rules FIRST, then change the default policy. Otherwise, you'll immediately lose your RDP connection.

#### Step 2: Inbound Rule — Allow RDP from Specific IPs Only

```powershell
New-NetFirewallRule -DisplayName "Allow RDP from Trusted Admin Hosts" `
    -Direction Inbound -Protocol TCP -LocalPort 3389 `
    -RemoteAddress "10.0.1.10,10.0.1.11,10.0.2.0/24" `
    -Action Allow -Profile Domain
```

#### Step 3: Outbound Rules — Allow DC Authentication Traffic

```powershell
$DC = "10.0.0.5"  # Domain Controller IP

# Kerberos (core authentication)
New-NetFirewallRule -DisplayName "Allow Kerberos TCP to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 88 `
    -RemoteAddress $DC -Action Allow
New-NetFirewallRule -DisplayName "Allow Kerberos UDP to DC" `
    -Direction Outbound -Protocol UDP -RemotePort 88 `
    -RemoteAddress $DC -Action Allow

# DNS (required for SRV record DC location)
New-NetFirewallRule -DisplayName "Allow DNS TCP to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 53 `
    -RemoteAddress $DC -Action Allow
New-NetFirewallRule -DisplayName "Allow DNS UDP to DC" `
    -Direction Outbound -Protocol UDP -RemotePort 53 `
    -RemoteAddress $DC -Action Allow

# LDAP
New-NetFirewallRule -DisplayName "Allow LDAP TCP to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 389 `
    -RemoteAddress $DC -Action Allow
New-NetFirewallRule -DisplayName "Allow LDAP UDP to DC" `
    -Direction Outbound -Protocol UDP -RemotePort 389 `
    -RemoteAddress $DC -Action Allow

# LDAPS (encrypted LDAP)
New-NetFirewallRule -DisplayName "Allow LDAPS to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 636 `
    -RemoteAddress $DC -Action Allow

# SMB (Group Policy / SYSVOL)
New-NetFirewallRule -DisplayName "Allow SMB to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 445 `
    -RemoteAddress $DC -Action Allow

# RPC Endpoint Mapper + Dynamic Ports
New-NetFirewallRule -DisplayName "Allow RPC EPM to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 135 `
    -RemoteAddress $DC -Action Allow
New-NetFirewallRule -DisplayName "Allow RPC Dynamic to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 49152-65535 `
    -RemoteAddress $DC -Action Allow

# NTP Time Sync (Kerberos requires clock synchronization)
New-NetFirewallRule -DisplayName "Allow NTP to DC" `
    -Direction Outbound -Protocol UDP -RemotePort 123 `
    -RemoteAddress $DC -Action Allow

# Global Catalog (multi-domain environments)
New-NetFirewallRule -DisplayName "Allow Global Catalog to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 3268 `
    -RemoteAddress $DC -Action Allow

# Kerberos Password Change
New-NetFirewallRule -DisplayName "Allow Kerberos PW Change to DC" `
    -Direction Outbound -Protocol TCP -RemotePort 464 `
    -RemoteAddress $DC -Action Allow
New-NetFirewallRule -DisplayName "Allow Kerberos PW Change UDP to DC" `
    -Direction Outbound -Protocol UDP -RemotePort 464 `
    -RemoteAddress $DC -Action Allow
```

#### Step 4: Outbound Rule — Allow RDP to Specific Target Machines

```powershell
New-NetFirewallRule -DisplayName "Allow RDP Out to Trusted Hosts" `
    -Direction Outbound -Protocol TCP -RemotePort 3389 `
    -RemoteAddress "10.0.2.20,10.0.2.21,10.0.3.0/24" `
    -Action Allow -Profile Domain
```

## 4. Key Configurations

### Complete DC Authentication Port List

| Port | Protocol | Purpose | Required |
|------|----------|---------|----------|
| 88 | TCP/UDP | Kerberos Authentication | ✅ Must — Core auth |
| 53 | TCP/UDP | DNS Query | ✅ Must — DC location via SRV |
| 389 | TCP/UDP | LDAP | ✅ Must — Domain queries |
| 636 | TCP | LDAPS (Encrypted LDAP) | 🔶 If using LDAPS |
| 445 | TCP | SMB | ✅ Must — GPO/SYSVOL |
| 135 | TCP | RPC Endpoint Mapper | ✅ Must — RPC service location |
| 49152-65535 | TCP | RPC Dynamic Ports | ✅ Must — LSA/SAM/Netlogon |
| 123 | UDP | NTP Time Sync | ✅ Must — Kerberos clock sync |
| 3268 | TCP | Global Catalog | 🔶 Multi-domain environments |
| 3269 | TCP | Global Catalog SSL | 🔶 Multi-domain + SSL |
| 464 | TCP/UDP | Kerberos Password Change | 🔶 Password change scenarios |

## 5. Common Issues & Troubleshooting

### Issue A: RDP Disconnected After Policy Change

- **Cause**: Changed Default Outbound to Block before creating Allow rules, or missed your own IP
- **Recovery**: Use Azure Run Command / Serial Console, or local console access
  ```powershell
  Set-NetFirewallProfile -Profile Domain,Private,Public -DefaultOutboundAction Allow
  ```

### Issue B: Domain Login Fails After Outbound Block

- **Cause**: Missing required DC authentication port
- **Troubleshooting**:
  1. Enable WFP audit: `auditpol /set /subcategory:"Filtering Platform Packet Drop" /success:enable /failure:enable`
  2. Check Security Event Log for Event ID **5152** (dropped packets)
  3. Add missing Allow rules based on blocked ports

### Issue C: Group Policy Not Updating

- **Cause**: Missing SMB (445) or RPC (135 + dynamic ports) outbound rules
- **Verification**: `Test-NetConnection -ComputerName <DC_IP> -Port 445`

### Issue D: Kerberos Failure Due to Clock Skew

- **Cause**: Missing NTP (UDP 123) outbound rule; clock drift exceeds 5-minute tolerance
- **Fix**: Add NTP rule, then `w32tm /resync`

## 6. Practical Tips

### Best Practices

- **Rules first, policy last**: Always create all Allow rules before changing Default Action to Block
- **Script your deployment**: Put all rules in a PowerShell script for repeatable deployment and disaster recovery
- **Clear naming convention**: Use descriptive DisplayName (e.g., `Allow Kerberos TCP to DC`)
- **Multiple DCs**: List all DC IPs in the `RemoteAddress` parameter

### Common Mistakes

- ❌ Only blocking Inbound without changing Outbound — doesn't achieve true isolation
- ❌ Allowing all traffic to DC IP with `Any` port — always be port-specific
- ❌ Forgetting DNS — Kerberos relies on DNS SRV records to locate DCs
- ❌ Forgetting NTP — Kerberos has strict clock synchronization requirements (5-minute default tolerance)

### Security Notes

- The RPC dynamic port range (49152-65535) is large. For tighter security, restrict RPC ports on the DC via registry, then narrow the firewall range accordingly
- Regularly audit rules: `Get-NetFirewallRule | Where-Object {$_.Enabled -eq 'True'}`
- For Azure VMs, **NSG rules must be configured in sync with Windows Firewall rules** for defense in depth

## 7. Deploy via Group Policy (GPO)

### Configuration Location

In **Group Policy Management Console (GPMC)**:

```
Computer Configuration
  → Policies
    → Windows Settings
      → Security Settings
        → Windows Firewall with Advanced Security
```

This provides the same interface as the local `wf.msc`, where you can configure Inbound/Outbound Rules and Profile default actions.

### Storage Location on Target Machines

| Type | Registry Path |
|------|--------------|
| **GPO-pushed rules** | `HKLM\SOFTWARE\Policies\Microsoft\WindowsFirewall\FirewallRules` |
| **GPO Profile settings** | `HKLM\SOFTWARE\Policies\Microsoft\WindowsFirewall\DomainProfile` etc. |
| **Local rules** | `HKLM\SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy\FirewallRules` |

### GPO Rule Characteristics

- GPO-pushed rules appear **grayed-out (read-only)** in `wf.msc`
- Local administrators can view but cannot modify or delete GPO rules
- Refreshed every **90 minutes** by default (with 0-30 minute random offset)
- Firewall monitors registry changes → notifies **Windows Filtering Platform (WFP)** → reads all rules → applies new filters → removes old filters

### Verification Commands

```powershell
# Force GPO refresh
gpupdate /force

# View GPO-pushed rules
Get-NetFirewallRule -PolicyStore ActiveStore |
    Where-Object { $_.PolicyStoreSource -ne "PersistentStore" } |
    Format-Table DisplayName, Direction, Action

# Check registry directly
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\WindowsFirewall\FirewallRules"
```

## 8. Deploy via Intune (MDM)

### Configuration Location

In **Microsoft Intune admin center**:

```
Endpoint Security
  → Firewall
    → Create Policy
      Platform: Windows
      Profile:
        ├── Windows Firewall        ← Profile-level settings (default actions)
        └── Windows Firewall Rules  ← Specific rules (up to 150 per policy)
```

Each rule supports the following fields:

| Field | Description | Example |
|-------|-------------|---------|
| Name | Rule name | Allow RDP from Admin |
| Direction | Inbound / Outbound | Inbound |
| Action | Allow / Block | Allow |
| Protocol | TCP (6) / UDP (17) | 6 |
| Local Port Ranges | Local ports | 3389 |
| Remote Port Ranges | Remote ports | 3389 |
| Remote Address Ranges | Remote IPs | 10.0.1.10,10.0.1.11 |
| Profile | Domain/Private/Public | Domain |

### Storage Location on Target Machines

Intune delivers rules via **Firewall CSP** (`./Vendor/MSFT/Firewall/MdmStore`):

| Type | Registry Path |
|------|--------------|
| **Intune (MDM) rules** | `HKLM\SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy\Mdm\FirewallRules` |
| **Intune Profile settings** | `...\FirewallPolicy\Mdm\DomainProfile` etc. |

### Verification Commands

```powershell
# View MDM-pushed rules (registry)
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy\Mdm\FirewallRules"

# View all rules with source
Get-NetFirewallRule | Select-Object DisplayName, Direction, Action, PolicyStoreSource |
    Format-Table -AutoSize
# PolicyStoreSource values:
#   PersistentStore  = Local rules
#   YOURPC\MDM       = Intune-pushed
#   GroupPolicy       = GPO-pushed

# Check Intune sync status
dsregcmd /status
```

### Intune Rule Characteristics

- Appear **grayed-out (read-only)** in `wf.msc`; some versions may not display them directly — use PowerShell to verify
- Up to 150 rules per policy; multiple policies can be stacked
- Syncs every **8 hours** by default (or manually triggered)

## 9. Rule Precedence Deep Dive

When GPO, Intune, and local rules coexist with overlapping or conflicting configurations, how is precedence determined?

### Dimension 1: Within Same Source — Block vs Allow

**Regardless of whether rules come from GPO, Intune, or local, the precedence logic is consistent:**

```
① Explicit Block rules  >  ② Explicit Allow rules  >  ③ Default Action
```

| Priority | Rule Type | Description |
|----------|----------|-------------|
| **Highest** | Explicit Block | If a Block rule matches, traffic is blocked regardless of any Allow rules |
| **Medium** | Explicit Allow | Permits traffic only when no conflicting Block rule exists |
| **Lowest** | Default Action | Applied only when no matching rules exist |

> ⚠️ **Core principle: Block always wins over Allow.** If traffic matches both a Block and an Allow rule, Block prevails.

Additionally: **More specific rules take precedence over broader rules** (e.g., single IP > IP range), but explicit Block still overrides any Allow.

### Dimension 2: Across Sources — GPO vs Intune vs Local

```
┌─────────────────────────────────────────────────────┐
│           Final Effective Ruleset (Active Store)      │
│                                                     │
│   Evaluation order (all sources merged):             │
│                                                     │
│   1. GPO/MDM Block rules      ← Highest priority    │
│   2. GPO/MDM Allow rules                            │
│   3. Local Block rules *                             │
│   4. Local Allow rules *                             │
│   5. Default Action                                  │
│                                                     │
│   * Requires: AllowLocalPolicyMerge = true (default) │
│     If set to false, local rules are ignored entirely │
└─────────────────────────────────────────────────────┘
```

### AllowLocalPolicyMerge Impact

| AllowLocalPolicyMerge | Local Rule Behavior |
|----------------------|---------------------|
| **true** (default) | Local rules **merge** with GPO/MDM rules, but GPO/MDM Block still takes priority |
| **false** | Local rules are **completely ignored**; only GPO/MDM rules apply |

Configurable via:
- **GPO**: Windows Firewall Properties → Domain Profile → Settings → Rule merging → Apply local firewall rules: **No**
- **Intune CSP**: `./Vendor/MSFT/Firewall/MdmStore/DomainProfile/AllowLocalPolicyMerge` set to **0**

### Conflict Example

Given these rules:

| Source | Rule | Direction | Port | Action |
|--------|------|-----------|------|--------|
| GPO | Rule A | Inbound | TCP 3389 from 10.0.1.0/24 | **Allow** |
| Intune | Rule B | Inbound | TCP 3389 from Any | **Block** |
| Local | Rule C | Inbound | TCP 3389 from 10.0.1.5 | **Allow** |

**Result**:
- RDP from `10.0.1.5` → ❌ **Blocked** (Intune's explicit Block overrides all Allow rules)
- RDP from `10.0.1.100` → ❌ **Blocked** (same reason)
- **Block always wins**, regardless of which source the Allow comes from

### GPO vs Intune Relationship

- GPO and Intune rules are at **equal priority level**
- If both push **conflicting settings** (e.g., different default actions for the same Profile), the result is unpredictable
- **Microsoft recommends: choose one management method** — do not manage firewall with both GPO and Intune simultaneously

## 10. Comparison with Related Technologies

| Dimension | Windows Firewall (WFAS) | Azure NSG | GPO Firewall Rules | Intune Firewall |
|-----------|------------------------|-----------|-------------------|-----------------|
| Scope | Host-level | Network-level (VNet/Subnet/NIC) | Host-level, domain-wide | Host-level, cloud-managed |
| Configuration | PowerShell / GUI / netsh | Azure Portal / CLI / ARM | Group Policy Editor | Intune Admin Center |
| Granularity | Port + IP + Program + Service | Port + IP + Service Tag | Same as WFAS | Same as WFAS |
| Target Devices | Any Windows | Azure VMs | Domain-joined | Entra ID joined / Intune enrolled |
| Best For | Single-machine isolation | Azure VM network isolation | Bulk policy across domain | Cloud-managed device policy |
| Rule Storage | Local registry | Azure platform | `HKLM\SOFTWARE\Policies\...` | `...\FirewallPolicy\Mdm\...` |
| Stackable | ✅ | ✅ Layers with WFAS | ✅ Pushes to WFAS via GPO | ✅ Pushes to WFAS via CSP |

**Selection guidance**:
- Single-server quick isolation → WFAS + PowerShell
- Azure VMs → NSG + WFAS defense-in-depth
- Domain-wide bulk management → GPO Firewall Rules
- Cloud-managed devices → Intune Endpoint Security Firewall Policy
- ⚠️ Do not use GPO and Intune simultaneously for firewall management to avoid conflicts

## 11. References

- [Manage Windows Firewall with the command line](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/configure-with-command-line) — Complete PowerShell and netsh firewall management reference
- [Windows Firewall rules](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/rules) — Rule precedence, Local Policy Merge documentation
- [How to configure a firewall for AD domains and trusts](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/config-firewall-for-ad-domains-and-trusts) — Authoritative AD authentication port list (KB179442)
- [Windows Firewall tools](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/tools) — GPO processing and registry storage details
- [Firewall policy for endpoint security in Intune](https://learn.microsoft.com/en-us/mem/intune/protect/endpoint-security-firewall-policy) — Intune firewall policy configuration guide
- [Firewall CSP](https://learn.microsoft.com/en-us/windows/client-management/mdm/firewall-csp) — Intune MDM Firewall CSP detailed reference
