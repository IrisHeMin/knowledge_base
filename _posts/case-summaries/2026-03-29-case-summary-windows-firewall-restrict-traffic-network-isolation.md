---
layout: post
title: "Case Summary: 使用 Windows Firewall 限制服务器 Inbound/Outbound 流量实现网络隔离"
date: 2026-03-29
categories: [Windows-Server, Networking]
tags: [windows-firewall, rdp, active-directory, kerberos, gpo, intune, security, powershell]
---

# Case Summary: 使用 Windows Firewall 限制服务器 Inbound/Outbound 流量实现网络隔离

**Product/Service:** Windows Server / Windows Firewall with Advanced Security (WFAS)

---

## 1. 症状 (Symptoms)

客户有一台高敏感服务器，需要实现严格的网络流量隔离：

- **Inbound 需求**：仅允许特定管理机器通过 RDP（TCP 3389）连入，阻断所有其他入站流量
- **Outbound 需求**：
  - 仅允许与 Domain Controller (DC) 的认证相关通信（域登录、组策略下发等）
  - 允许本机 RDP 到少量指定的目标机器
  - 阻断所有其他出站流量
- 客户同时关心：如果通过 GPO 或 Intune 集中推送这些防火墙规则，规则存储在目标机器的什么位置？不同来源的规则优先级如何？

## 2. 背景 (Background / Environment)

- **操作系统**：Windows Server，加入 Active Directory 域
- **网络环境**：企业内网，Domain Controller 地址为 `10.0.0.5`
- **管理方式**：支持 GPO 和/或 Intune 管理
- **安全要求**：最小权限原则，服务器只开放必要的网络通信

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

1. **分析需求并确定技术方案**
   - 使用 Windows Firewall with Advanced Security (WFAS) 作为主机防火墙
   - 核心思路：将 Inbound 和 Outbound 的默认行为都设为 **Block**，然后通过白名单 **Allow** 规则精确放行需要的流量
   - 发现：Windows Firewall 的 Outbound 默认行为是 **Allow**（全开），必须手动改为 Block 才能实现出站隔离

2. **梳理 DC 认证所需的全部端口**
   - 域认证不仅仅是 Kerberos（TCP/UDP 88），还需要一整套协议栈：
     - **DNS**（TCP/UDP 53）— SRV 记录定位 DC
     - **LDAP**（TCP/UDP 389）— 域信息查询
     - **LDAPS**（TCP 636）— 加密 LDAP（按需）
     - **SMB**（TCP 445）— Group Policy / SYSVOL 下发
     - **RPC Endpoint Mapper**（TCP 135）+ **RPC 动态端口**（TCP 49152-65535）— 域复制、策略同步
     - **NTP**（UDP 123）— Kerberos 对时钟同步有严格要求（默认 5 分钟容差）
     - **Global Catalog**（TCP 3268）— 多域环境需要
     - **Kerberos Password Change**（TCP/UDP 464）— 密码修改场景
   - 参考了 Microsoft KB179442 确认端口列表的完整性

3. **编写 PowerShell 脚本实现规则配置**
   - 先创建所有 Allow 规则（Inbound RDP + Outbound DC 认证 + Outbound RDP），再修改默认策略为 Block
   - 关键命令：
     ```powershell
     # 设默认策略
     Set-NetFirewallProfile -Profile Domain,Private,Public `
         -DefaultInboundAction Block -DefaultOutboundAction Block

     # Inbound: 仅允许特定 IP RDP
     New-NetFirewallRule -DisplayName "Allow RDP from Trusted Hosts" `
         -Direction Inbound -Protocol TCP -LocalPort 3389 `
         -RemoteAddress "10.0.1.10,10.0.1.11" -Action Allow

     # Outbound: DC 认证端口（以 Kerberos 为例）
     New-NetFirewallRule -DisplayName "Allow Kerberos TCP to DC" `
         -Direction Outbound -Protocol TCP -RemotePort 88 `
         -RemoteAddress "10.0.0.5" -Action Allow

     # Outbound: RDP 到指定目标
     New-NetFirewallRule -DisplayName "Allow RDP Out to Trusted Hosts" `
         -Direction Outbound -Protocol TCP -RemotePort 3389 `
         -RemoteAddress "10.0.2.20,10.0.2.21" -Action Allow
     ```

4. **客户验证测试**
   - 客户按照方案在目标服务器上配置并验证，确认**方案可行**
   - Inbound：仅允许的管理 IP 可以 RDP 进入，其他 IP 被阻断
   - Outbound：域登录正常、组策略更新正常、RDP 到目标机器正常，其他出站流量被阻断

5. **进一步研究 GPO 集中推送**
   - GPO 配置路径：`Computer Configuration → Policies → Windows Settings → Security Settings → Windows Firewall with Advanced Security`
   - 规则存储在目标机器注册表：`HKLM\SOFTWARE\Policies\Microsoft\WindowsFirewall\FirewallRules`
   - GPO 规则在 `wf.msc` 中显示为灰色（不可编辑），本地管理员无法覆盖

6. **进一步研究 Intune (MDM) 推送**
   - Intune 配置路径：`Endpoint Security → Firewall → Create Policy → Windows Firewall Rules`
   - 通过 Firewall CSP (`./Vendor/MSFT/Firewall/MdmStore`) 下发
   - 规则存储在目标机器注册表：`HKLM\SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy\Mdm\FirewallRules`
   - 每个策略最多 150 条规则，可叠加多个策略

7. **澄清规则优先级问题**
   - 同一来源内：**Block 永远赢 Allow**（显式 Block > 显式 Allow > 默认行为）
   - 不同来源间：GPO/MDM 的规则优先于本地规则
   - `AllowLocalPolicyMerge` 设为 false 可完全禁用本地规则
   - GPO 和 Intune 属于同等优先级，不建议同时使用避免冲突

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| Outbound 默认是 Allow，容易忽略 | 如果不改 Outbound 默认行为，无法实现出站隔离 | 明确告知客户必须将 DefaultOutboundAction 改为 Block |
| DC 认证需要的端口较多，容易遗漏 | 遗漏端口会导致域登录失败或组策略不更新 | 参考 KB179442 完整列出所有端口，逐一创建规则 |
| 修改默认策略可能断连 | 如果先改策略再建规则，RDP 会立即断开 | 强调"先建规则后改策略"的操作顺序 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

这不是一个故障排查案例，而是一个 **方案咨询** 案例。客户需要利用 Windows 内置能力实现最小权限的网络隔离。

### Resolution

使用 **Windows Firewall with Advanced Security (WFAS)** 实现白名单策略：

1. **创建 Inbound Allow 规则**：仅允许指定管理 IP 通过 TCP 3389 (RDP) 进入
2. **创建 Outbound Allow 规则**：
   - DC 认证端口：Kerberos (88)、DNS (53)、LDAP (389)、LDAPS (636)、SMB (445)、RPC (135 + 49152-65535)、NTP (123)、GC (3268)、KPW (464) → 仅到 DC IP
   - RDP (3389) → 仅到指定目标机器 IP
3. **修改默认策略**：将三个 Profile 的 Inbound 和 Outbound 默认行为都设为 **Block**
4. **集中管理**（可选）：
   - **GPO**：通过 GPMC → WFAS 节点推送规则，存储在 `HKLM\SOFTWARE\Policies\Microsoft\WindowsFirewall`
   - **Intune**：通过 Endpoint Security → Firewall 推送规则，存储在 `...\FirewallPolicy\Mdm\FirewallRules`

### ⚠️ 关键操作顺序
```
先创建所有 Allow 规则 → 验证规则正确 → 最后才改默认策略为 Block
```

## 6. 经验教训 (Lessons Learned)

- **技术知识**：
  - Windows Firewall 的 Outbound 默认行为是 **Allow**，这是很多人忽略的点。要实现真正的出站隔离必须显式改为 Block
  - DC 认证涉及的端口远不止 Kerberos 88，DNS (53)、NTP (123)、SMB (445) 等都是必需的，遗漏任何一个都会导致认证链断裂
  - Kerberos 对时钟同步有严格要求（5 分钟容差），NTP 端口 (UDP 123) 必须放行
  - RPC 动态端口范围 (49152-65535) 较大，安全要求极高时可在 DC 上通过注册表限制 RPC 端口范围
- **排查方法**：
  - 如果 Outbound Block 后出现域登录失败，可开启 WFP 审计日志 (`auditpol /set /subcategory:"Filtering Platform Packet Drop"`)，查看 Event ID 5152 定位被阻断的端口
  - 使用 `Test-NetConnection -ComputerName <DC_IP> -Port <Port>` 验证各端口连通性
- **管理建议**：
  - GPO 推送的规则存储在 `HKLM\SOFTWARE\Policies\...`，Intune 存储在 `...\FirewallPolicy\Mdm\...`，本地规则存储在 `...\FirewallPolicy\FirewallRules`
  - 规则优先级：Block > Allow（不管来源），GPO/MDM > 本地规则
  - **不建议 GPO 和 Intune 同时管理防火墙**，避免冲突
  - 通过 `Get-NetFirewallRule | Select DisplayName, Direction, Action, PolicyStoreSource` 可查看每条规则的来源
- **预防措施**：
  - 将所有规则写成 PowerShell 脚本，方便批量部署和灾难恢复
  - 如果是 Azure VM，需在 NSG 层面同步配置相同规则（双层防护）
  - 远程操作时务必确保自己的 IP 在 Allow 列表中

## 7. 参考文档 (References)

- [Manage Windows Firewall with the command line](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/configure-with-command-line) — PowerShell 和 netsh 管理 Windows Firewall 完整参考
- [Windows Firewall rules](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/rules) — 规则优先级、Local Policy Merge 官方说明
- [How to configure a firewall for AD domains and trusts](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/config-firewall-for-ad-domains-and-trusts) — AD 域认证所需端口权威列表 (KB179442)
- [Windows Firewall tools](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/tools) — GPO 处理机制和注册表存储说明
- [Firewall policy for endpoint security in Intune](https://learn.microsoft.com/en-us/mem/intune/protect/endpoint-security-firewall-policy) — Intune 防火墙策略配置指南
- [Firewall CSP](https://learn.microsoft.com/en-us/windows/client-management/mdm/firewall-csp) — Intune MDM 防火墙 CSP 详细参考

---
---

# Case Summary: Restricting Server Inbound/Outbound Traffic with Windows Firewall for Network Isolation

**Product/Service:** Windows Server / Windows Firewall with Advanced Security (WFAS)

---

## 1. Symptoms

The customer had a high-sensitivity server requiring strict network traffic isolation:

- **Inbound requirement**: Only allow specific admin machines to RDP in (TCP 3389); block all other inbound traffic
- **Outbound requirement**:
  - Only allow authentication traffic to Domain Controller (DC) — domain login, Group Policy sync, etc.
  - Allow RDP from this machine to a few designated target machines
  - Block all other outbound traffic
- The customer also asked: if these firewall rules are deployed centrally via GPO or Intune, where are they stored on the target machine? What is the rule precedence across different sources?

## 2. Background / Environment

- **OS**: Windows Server, domain-joined to Active Directory
- **Network**: Enterprise intranet, Domain Controller at `10.0.0.5`
- **Management**: Supports both GPO and/or Intune management
- **Security requirement**: Least privilege principle — only essential network communication allowed

## 3. Investigation & Troubleshooting

1. **Analyzed requirements and determined technical approach**
   - Selected Windows Firewall with Advanced Security (WFAS) as the host firewall
   - Core strategy: Set both Inbound and Outbound default actions to **Block**, then create whitelist **Allow** rules for permitted traffic
   - Key finding: Windows Firewall's Outbound default action is **Allow** (wide open) — must be explicitly changed to Block for outbound isolation

2. **Identified all required DC authentication ports**
   - Domain authentication requires an entire protocol stack beyond just Kerberos (TCP/UDP 88):
     - **DNS** (TCP/UDP 53) — SRV record DC location
     - **LDAP** (TCP/UDP 389) — Domain queries
     - **LDAPS** (TCP 636) — Encrypted LDAP (optional)
     - **SMB** (TCP 445) — Group Policy / SYSVOL delivery
     - **RPC Endpoint Mapper** (TCP 135) + **RPC Dynamic Ports** (TCP 49152-65535) — Domain replication, policy sync
     - **NTP** (UDP 123) — Kerberos requires clock synchronization (5-minute default tolerance)
     - **Global Catalog** (TCP 3268) — Multi-domain environments
     - **Kerberos Password Change** (TCP/UDP 464) — Password change scenarios
   - Referenced Microsoft KB179442 for authoritative port list

3. **Developed PowerShell script for rule configuration**
   - Created all Allow rules first (Inbound RDP + Outbound DC auth + Outbound RDP), then changed default policy to Block
   - Key commands:
     ```powershell
     # Set default policies
     Set-NetFirewallProfile -Profile Domain,Private,Public `
         -DefaultInboundAction Block -DefaultOutboundAction Block

     # Inbound: Allow RDP from specific IPs only
     New-NetFirewallRule -DisplayName "Allow RDP from Trusted Hosts" `
         -Direction Inbound -Protocol TCP -LocalPort 3389 `
         -RemoteAddress "10.0.1.10,10.0.1.11" -Action Allow

     # Outbound: DC auth ports (Kerberos example)
     New-NetFirewallRule -DisplayName "Allow Kerberos TCP to DC" `
         -Direction Outbound -Protocol TCP -RemotePort 88 `
         -RemoteAddress "10.0.0.5" -Action Allow

     # Outbound: RDP to designated targets
     New-NetFirewallRule -DisplayName "Allow RDP Out to Trusted Hosts" `
         -Direction Outbound -Protocol TCP -RemotePort 3389 `
         -RemoteAddress "10.0.2.20,10.0.2.21" -Action Allow
     ```

4. **Customer verification**
   - Customer implemented the solution and confirmed **it works as expected**
   - Inbound: Only allowed admin IPs can RDP in; others are blocked
   - Outbound: Domain login works, Group Policy updates work, RDP to target machines works; all other outbound traffic is blocked

5. **Researched GPO centralized deployment**
   - GPO configuration path: `Computer Configuration → Policies → Windows Settings → Security Settings → Windows Firewall with Advanced Security`
   - Rules stored on target machine at: `HKLM\SOFTWARE\Policies\Microsoft\WindowsFirewall\FirewallRules`
   - GPO rules appear grayed-out (read-only) in `wf.msc`; local admins cannot override them

6. **Researched Intune (MDM) deployment**
   - Intune configuration path: `Endpoint Security → Firewall → Create Policy → Windows Firewall Rules`
   - Delivered via Firewall CSP (`./Vendor/MSFT/Firewall/MdmStore`)
   - Rules stored on target machine at: `HKLM\SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy\Mdm\FirewallRules`
   - Each policy supports up to 150 rules; multiple policies can be stacked

7. **Clarified rule precedence**
   - Within same source: **Block always wins over Allow** (Explicit Block > Explicit Allow > Default Action)
   - Across sources: GPO/MDM rules take precedence over local rules
   - `AllowLocalPolicyMerge` set to false completely ignores local rules
   - GPO and Intune are equal priority — not recommended to use both simultaneously to avoid conflicts

## 4. Blockers & How They Were Resolved

| Blocker | Impact | Resolution |
|---------|--------|-----------|
| Outbound default is Allow — easy to overlook | Without changing Outbound default, outbound isolation is impossible | Explicitly advised customer to change DefaultOutboundAction to Block |
| DC authentication requires many ports — easy to miss some | Missing any port causes domain login failure or GP sync issues | Referenced KB179442 for complete port list; created rules for each port |
| Changing default policy may disconnect RDP | If policy changed before rules are created, RDP drops immediately | Emphasized "rules first, policy last" operation order |

## 5. Root Cause & Resolution

### Root Cause

This was not a break-fix case but a **solution advisory** case. The customer needed to leverage Windows built-in capabilities to achieve least-privilege network isolation.

### Resolution

Implemented a **whitelist strategy using Windows Firewall with Advanced Security (WFAS)**:

1. **Create Inbound Allow rules**: Only allow designated admin IPs via TCP 3389 (RDP)
2. **Create Outbound Allow rules**:
   - DC authentication ports: Kerberos (88), DNS (53), LDAP (389), LDAPS (636), SMB (445), RPC (135 + 49152-65535), NTP (123), GC (3268), KPW (464) → DC IP only
   - RDP (3389) → Designated target machine IPs only
3. **Set default policies**: Change all three Profiles' Inbound and Outbound default actions to **Block**
4. **Centralized management** (optional):
   - **GPO**: Deploy via GPMC → WFAS node; stored at `HKLM\SOFTWARE\Policies\Microsoft\WindowsFirewall`
   - **Intune**: Deploy via Endpoint Security → Firewall; stored at `...\FirewallPolicy\Mdm\FirewallRules`

### ⚠️ Critical Operation Order
```
Create all Allow rules → Verify rules are correct → Change default policy to Block LAST
```

## 6. Lessons Learned

- **Technical knowledge**:
  - Windows Firewall's Outbound default is **Allow** — a commonly overlooked fact. Must explicitly change to Block for outbound isolation
  - DC authentication involves far more than just Kerberos port 88; DNS (53), NTP (123), SMB (445) are all essential — missing any one breaks the authentication chain
  - Kerberos has strict clock sync requirements (5-minute tolerance); NTP port (UDP 123) must be allowed
  - RPC dynamic port range (49152-65535) is large; for higher security, restrict RPC ports on the DC via registry
- **Troubleshooting methods**:
  - If domain login fails after Outbound Block, enable WFP audit logging (`auditpol /set /subcategory:"Filtering Platform Packet Drop"`) and check Event ID 5152 to identify blocked ports
  - Use `Test-NetConnection -ComputerName <DC_IP> -Port <Port>` to verify individual port connectivity
- **Management recommendations**:
  - GPO rules stored at `HKLM\SOFTWARE\Policies\...`, Intune at `...\FirewallPolicy\Mdm\...`, local at `...\FirewallPolicy\FirewallRules`
  - Rule precedence: Block > Allow (regardless of source); GPO/MDM > Local rules
  - **Do not use GPO and Intune simultaneously** for firewall management — avoid conflicts
  - Use `Get-NetFirewallRule | Select DisplayName, Direction, Action, PolicyStoreSource` to verify each rule's source
- **Prevention**:
  - Script all rules in PowerShell for repeatable deployment and disaster recovery
  - For Azure VMs, configure matching NSG rules for defense-in-depth
  - When operating remotely, always ensure your own IP is in the Allow list

## 7. References

- [Manage Windows Firewall with the command line](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/configure-with-command-line) — Complete PowerShell and netsh firewall management reference
- [Windows Firewall rules](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/rules) — Rule precedence and Local Policy Merge documentation
- [How to configure a firewall for AD domains and trusts](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/config-firewall-for-ad-domains-and-trusts) — Authoritative AD authentication port list (KB179442)
- [Windows Firewall tools](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/tools) — GPO processing and registry storage details
- [Firewall policy for endpoint security in Intune](https://learn.microsoft.com/en-us/mem/intune/protect/endpoint-security-firewall-policy) — Intune firewall policy configuration guide
- [Firewall CSP](https://learn.microsoft.com/en-us/windows/client-management/mdm/firewall-csp) — Intune MDM Firewall CSP detailed reference
