---
layout: post
title: "休眠唤醒后 NAS 映射驱动器断开 — 网卡断电导致 Kerberos 认证失败"
date: 2025-10-04
categories: [SMB, Windows-Client]
tags: [smb, mapped-drive, standby, hibernate, nic-power-management, kerberos, credential-manager, net-use, guest-account, nas, modern-standby]

---

# Case Summary: 休眠唤醒后 NAS 映射驱动器断开 — 网卡休眠断电触发 Kerberos 认证失败链

**Product/Service:** Windows 10/11 — SMB Mapped Drive + NAS Share

---

## 1. 症状 (Symptoms)

- 客户报告 Windows 桌面设备**长时间休眠后唤醒**，发现**无法连接 NAS 映射驱动器**。
- 使用 `net use` 命令重新连接时，报错 **"密码失效"**，要求重新输入账号密码。
- 原本使用 guest 账号映射的驱动器，唤醒后凭据丢失，需手动重新认证。

## 2. 背景 (Background / Environment)

- **客户端**：Windows 桌面设备（云桌面环境）
- **NAS 共享**：云 NAS 服务
- **映射命令**：`net use \\nas-file01.cn-region.nas.cloudprovider.com\myshare`
- **认证方式**：使用 Guest 账号映射
- **域环境**：设备加入域 `CLOUD-DESKTOP-198.VDI.LOCAL`
- **电源管理**：设备支持 Modern Standby

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

### 第一轮：网络抓包分析 — SMB 协商后客户端 Reset

1. **做了什么**：在唤醒后重新连接 NAS 时抓取网络流量，分析 SMB 会话建立过程。
2. **发现了什么**：SMB2 Negotiate 完成后，客户端**主动发送 TCP RST** 断开连接：
   ```
   16:21:22.237 Client(10.10.24.57) → NAS: TCP SYN (Port 445)
   16:21:22.237 NAS → Client: TCP SYN-ACK
   16:21:22.237 Client → NAS: TCP ACK
   16:21:22.237 Client → NAS: SMB2 NEGOTIATE Request
   16:21:22.239 NAS → Client: SMB2 NEGOTIATE Response
                 (Authentication Method: GSSAPI)
   16:21:22.248 Client → NAS: TCP RST ← 客户端主动断开！
   ```
3. **得出了什么结论**：客户端在收到 SMB Negotiate 响应后，在发起 Session Setup（认证）之前就放弃了连接。问题出在客户端的认证准备阶段。

### 第二轮：SMB Client ETL — MupCreate 直接报错

1. **做了什么**：检查客户端的 SMB Client ETL 日志。
2. **发现了什么**：`MupCreate` 操作直接返回错误 `0xc0000388`：
   ```
   CscSurrogatePostProcess() -
     MupStatus = 0xc0000388
     FO.Name = '\nas-file01.cn-region.nas.cloudprovider.com\pipe'
   MupCreate() -
     FileName \nas-file01.cn-region.nas.cloudprovider.com\pipe
     status 0xc0000388
   ```
   错误码 `0xc0000388` = **"The system cannot contact a domain controller to service the authentication request. Please try again later."**
3. **得出了什么结论**：SMB 客户端在创建连接时无法联系域控制器完成认证。问题转向 Kerberos 认证层。

### 第三轮：Kerberos ETL — 找不到域控制器

1. **做了什么**：检查客户端的 Kerberos ETL 日志。
2. **发现了什么**：Kerberos 尝试获取 TGT 时，无法找到域控制器：
   ```
   KerbCheckCredMgrForGivenTarget() -
     TargetServerName: cifs/nas-file01.cn-region.nas.cloudprovider.com
   KerbGetTgtForService() -
     refreshing primary TGT for account
   KerbRefreshPrimaryTgt() -
     getting new TGT for account
   KerbGetKdcBinding() -
     No MS DC for domain CLOUD-DESKTOP-198.VDI.LOCAL,
     account name NULL, locator flags 0x600: 1355
   KerbMakeSocketCallEx() -
     Failed to get KDC binding for CLOUD-DESKTOP-198.VDI.LOCAL:
     0xc000005e -> STATUS_NO_LOGON_SERVERS
   ```
3. **得出了什么结论**：Kerberos 使用的是**当前登录的域账号**（而非原始 guest 账号）尝试认证，但此时无法联系 DC。关键问题：为什么使用域账号而非 guest 账号？为什么 DC 不可达？

### 第四轮：System Event Log — 休眠期间网卡断网

1. **做了什么**：检查 System Event Log 中与电源管理相关的事件。
2. **发现了什么**：
   ```
   Source: Microsoft-Windows-Kernel-Power
   Event: Connectivity state in standby: Disconnected
   Reason: NIC compliance
   ```
3. **得出了什么结论**：在休眠/待机期间，**网卡被电源管理关闭**（NIC compliance），导致网络断开。

### 第五轮：还原完整故事链

基于以上所有日志，还原事件链：

1. **休眠期间**：电源管理关闭了网卡（NIC compliance）→ 网络断开
2. **网络断开后**：NAS 的 SMB Session 超时失效 → Mapped Drive 断开 → SMB Cache 清空
3. **唤醒后重新连接**：由于 SMB Cache 已清空，凭据丢失（原始 guest 账号信息不再保留）
4. **认证回退到域账号**：SMB 客户端使用当前登录的**域账号**尝试 Kerberos 认证（而非原始 guest 账号）
5. **DC 不可达**：网卡刚恢复，网络尚未完全就绪，或 DC 发现延迟 → Kerberos 无法获取 TGT → 返回 `STATUS_NO_LOGON_SERVERS`
6. **最终结果**：认证失败，用户被提示重新输入密码

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| 问题需要长时间休眠才能复现，不易在短时间内触发 | 排查时间拉长，需等待自然触发 | 通过分析已有日志（网络包 + ETL + Event Log）进行事后分析 |
| 客户设备的网卡 Power Management 选项在 UI 中不可见 | 无法通过常规 UI 操作禁用网卡省电 | 提供了两种注册表方法绕过 UI 限制 |
| 根因涉及多个组件交互（电源管理 + SMB + Kerberos），单独查看任一组件都不能完整解释 | 需要综合多个日志源才能还原完整事件链 | 交叉分析网络抓包、SMB ETL、Kerberos ETL、System Event Log，逐步拼接完整逻辑 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

**电源管理在休眠期间关闭网卡，导致 SMB 会话断开和凭据丢失，唤醒后触发 Kerberos 认证失败链。**

具体机制：
1. 设备进入休眠/待机时，电源管理关闭网卡（Reason: NIC compliance）
2. 网络断开导致 NAS 的 SMB Session 失效，Mapped Drive 断开，SMB Cache（含 guest 凭据）清空
3. 唤醒后 SMB 客户端尝试重新连接，但由于凭据已丢失，回退使用当前登录的域账号进行 Kerberos 认证
4. 网卡刚恢复、DC 尚不可达 → Kerberos 获取 TGT 失败 → `STATUS_NO_LOGON_SERVERS (0xc000005e)`
5. 认证失败，报错 `0xc0000388`（无法联系域控制器），用户被要求重新输入凭据

### Resolution（三种方案）

#### 方案一：防止休眠期间网卡断电（治本）

禁用网卡的电源管理省电选项，防止休眠时网络断开：

**方法 A — 通过注册表添加 PlatformAoAcOverride：**
1. 打开注册表编辑器，定位到 `HKLM\SYSTEM\CurrentControlSet\Control\Power`
2. 新建 DWORD 值 `PlatformAoAcOverride`，设置为 `0`
3. 重启设备
4. 打开设备管理器 → 找到网卡 → 属性 → Power Management → 取消勾选 **"Allow the computer to turn off this device to save power"**

**方法 B — 通过 PnPCapabilities 注册表：**
1. 定位到 `HKLM\SYSTEM\CurrentControlSet\Control\Class\{4D36E972-E325-11CE-BFC1-08002bE10318}\<DeviceNumber>`
2. 在该键下的 Driver Description 确认正确的网卡设备编号
3. 修改 `PnPCapabilities` 值为 `24`（十进制）
4. 重启设备

#### 方案二：使用 Global Map Drive 保留凭据

使用 Global Mapping 映射驱动器，凭据会自动保存到 Credential Manager，休眠唤醒后可自动重连：

```powershell
# 方法 A: net use 加 /global 参数
net use Z: \\nas-file01.cn-region.nas.cloudprovider.com\myshare /global /user:guest *

# 方法 B: PowerShell New-SmbGlobalMapping
$cred = Get-Credential
New-SmbGlobalMapping -RemotePath "\\nas-file01.cn-region.nas.cloudprovider.com\myshare" -Credential $cred -LocalPath Z:
```

#### 方案三：Per-User Map Drive + 预存凭据

继续使用 Per-User Map Drive，但提前将 guest 凭据保存到 Credential Manager：

```cmd
# 方法 A: 使用 cmdkey 预存凭据
cmdkey /add:nas-file01.cn-region.nas.cloudprovider.com /user:guest /pass:<password>
net use Z: \\nas-file01.cn-region.nas.cloudprovider.com\myshare /savecred /persistent:yes

# 方法 B: 在 UI 映射时勾选 "Remember my credentials"
```

### Workaround

在以上方案实施前，用户可手动执行 `net use` 重新输入凭据恢复连接。

## 6. 经验教训 (Lessons Learned)

- **技术知识**：
  - **Modern Standby 与 NIC compliance**：Windows 设备在 Modern Standby 模式下，电源管理可能因 NIC compliance 关闭网卡，导致网络完全断开。这会级联影响所有依赖网络的服务（SMB 会话、VPN、远程桌面等）。System Event Log 中 `Microsoft-Windows-Kernel-Power` 事件源的 `Connectivity state in standby: Disconnected` 是关键线索。
  - **SMB Mapped Drive 凭据丢失机制**：当 SMB Session 断开且 Cache 清空后，Per-User Map Drive 的凭据（尤其是 guest 等非域账号）不会被保留。重连时 SMB 客户端会回退使用当前登录账号进行认证，如果该账号是域账号，则触发 Kerberos 认证流程。
  - **Global Map Drive vs Per-User Map Drive**：Global Map Drive（`/global` 参数或 `New-SmbGlobalMapping`）会将凭据持久化到 Credential Manager，在会话断开后仍可自动重连。Per-User Map Drive 的凭据仅在当前会话有效，除非使用 `cmdkey` 或 "Remember my credentials" 显式保存。
  - **PlatformAoAcOverride 注册表**：当设备管理器中网卡的 Power Management 选项不可见时（常见于 Modern Standby 设备），需要先添加 `PlatformAoAcOverride = 0` 到 `HKLM\SYSTEM\CurrentControlSet\Control\Power`，重启后才能看到并修改该选项。

- **排查方法**：
  - **休眠/唤醒网络问题的排查层次**：① System Event Log（电源事件、NIC 断开）→ ② 网络抓包（连接建立过程）→ ③ SMB Client ETL（MUP/CSC 错误）→ ④ Kerberos ETL（认证失败）→ ⑤ Credential Manager 状态。
  - **多组件交互的排查技巧**：当问题涉及电源管理、网络、SMB、Kerberos 等多个组件时，单独查看任一组件可能无法完整解释问题。需要将多个日志源按**时间线对齐**，拼接完整的事件链。
  - **错误码追踪链**：`0xc0000388`（无法联系 DC）→ `0xc000005e`（STATUS_NO_LOGON_SERVERS）→ NIC compliance disconnect。从应用层错误逆向追溯到硬件层根因。

- **预防措施**：
  - 在云桌面/VDI 环境中使用 NAS Mapped Drive 时，**优先使用 Global Mapping 或预存凭据**，避免休眠唤醒后凭据丢失。
  - 对于需要持续网络连接的场景，评估是否需要**禁用网卡电源管理**或调整 Modern Standby 策略。
  - 批量部署时，可通过 GPO 或脚本统一配置 `PnPCapabilities` 注册表或 `PlatformAoAcOverride`。

## 7. 参考文档 (References)

- [New-SmbGlobalMapping — PowerShell](https://learn.microsoft.com/en-us/powershell/module/smbshare/new-smbglobalmapping) — SMB Global Mapping PowerShell 命令参考
- [SMB Overview — Windows Server](https://learn.microsoft.com/en-us/windows-server/storage/file-server/file-server-smb-overview) — SMB 协议概述
- [Modern Standby Network Connectivity — Windows Hardware](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/modern-standby-network-connectivity) — Modern Standby 网络连接行为说明

---
---

# Case Summary: NAS Mapped Drive Disconnects After Standby — NIC Power-Off Triggers Kerberos Authentication Failure Chain

**Product/Service:** Windows 10/11 — SMB Mapped Drive + NAS Share

---

## 1. Symptoms

- Customer reported that after **waking from extended standby/hibernate**, the Windows desktop device could **not connect to the NAS mapped drive**.
- Using `net use` to reconnect showed a **"password expired"** error, requiring the user to re-enter credentials.
- The drive was originally mapped using a guest account, but after wake, the credentials were lost, requiring manual re-authentication.

## 2. Background / Environment

- **Client**: Windows desktop device (cloud desktop environment)
- **NAS Share**: Cloud NAS service
- **Mapping command**: `net use \\nas-file01.cn-region.nas.cloudprovider.com\myshare`
- **Authentication**: Mapped using Guest account
- **Domain environment**: Device joined to domain `CLOUD-DESKTOP-198.VDI.LOCAL`
- **Power management**: Device supports Modern Standby

## 3. Investigation & Troubleshooting

### Round 1: Network Capture — Client RST After SMB Negotiate

1. **Action**: Captured network traffic during the NAS reconnection attempt after wake.
2. **Finding**: After SMB2 Negotiate completed, the client **sent a TCP RST** to terminate the connection:
   ```
   16:21:22.237 Client(10.10.24.57) → NAS: TCP SYN (Port 445)
   16:21:22.237 NAS → Client: TCP SYN-ACK
   16:21:22.237 Client → NAS: TCP ACK
   16:21:22.237 Client → NAS: SMB2 NEGOTIATE Request
   16:21:22.239 NAS → Client: SMB2 NEGOTIATE Response
                 (Authentication Method: GSSAPI)
   16:21:22.248 Client → NAS: TCP RST ← Client aborted!
   ```
3. **Conclusion**: The client abandoned the connection after receiving the SMB Negotiate response, before initiating Session Setup (authentication). The issue was in the client's authentication preparation phase.

### Round 2: SMB Client ETL — MupCreate Error

1. **Action**: Examined the client's SMB Client ETL logs.
2. **Finding**: The `MupCreate` operation immediately returned error `0xc0000388`:
   ```
   CscSurrogatePostProcess() -
     MupStatus = 0xc0000388
     FO.Name = '\nas-file01.cn-region.nas.cloudprovider.com\pipe'
   MupCreate() -
     FileName \nas-file01.cn-region.nas.cloudprovider.com\pipe
     status 0xc0000388
   ```
   Error `0xc0000388` = **"The system cannot contact a domain controller to service the authentication request. Please try again later."**
3. **Conclusion**: The SMB client could not contact a domain controller during connection creation. Investigation shifted to the Kerberos authentication layer.

### Round 3: Kerberos ETL — Domain Controller Unreachable

1. **Action**: Examined the client's Kerberos ETL logs.
2. **Finding**: Kerberos failed to locate a domain controller while attempting to obtain a TGT:
   ```
   KerbCheckCredMgrForGivenTarget() -
     TargetServerName: cifs/nas-file01.cn-region.nas.cloudprovider.com
   KerbGetTgtForService() -
     refreshing primary TGT for account
   KerbRefreshPrimaryTgt() -
     getting new TGT for account
   KerbGetKdcBinding() -
     No MS DC for domain CLOUD-DESKTOP-198.VDI.LOCAL,
     account name NULL, locator flags 0x600: 1355
   KerbMakeSocketCallEx() -
     Failed to get KDC binding for CLOUD-DESKTOP-198.VDI.LOCAL:
     0xc000005e -> STATUS_NO_LOGON_SERVERS
   ```
3. **Conclusion**: Kerberos was using the **currently logged-in domain account** (not the original guest account) for authentication, but couldn't reach the DC. Key questions: Why domain account instead of guest? Why is the DC unreachable?

### Round 4: System Event Log — NIC Disconnected During Standby

1. **Action**: Examined System Event Log for power management events.
2. **Finding**:
   ```
   Source: Microsoft-Windows-Kernel-Power
   Event: Connectivity state in standby: Disconnected
   Reason: NIC compliance
   ```
3. **Conclusion**: During standby, the **NIC was powered off** by power management (NIC compliance), causing a complete network disconnect.

### Round 5: Reconstructing the Complete Event Chain

Based on all logs, the full event chain was reconstructed:

1. **During standby**: Power management powered off the NIC (NIC compliance) → network disconnected
2. **Network disconnect**: NAS SMB session timed out → Mapped drive disconnected → SMB cache (including guest credentials) cleared
3. **After wake**: SMB client attempted reconnection, but credentials were lost; fell back to current domain account for Kerberos authentication
4. **DC unreachable**: NIC just recovered, network not fully ready or DC discovery delayed → Kerberos TGT request failed → `STATUS_NO_LOGON_SERVERS`
5. **Result**: Authentication failed with `0xc0000388`, user prompted to re-enter credentials

## 4. Blockers & How They Were Resolved

| Blocker | Impact | How It Was Resolved |
|---------|--------|---------------------|
| Issue required extended standby to reproduce; could not easily trigger on demand | Extended troubleshooting timeline waiting for natural occurrence | Performed post-mortem analysis using existing logs (network capture + ETL + Event Log) |
| NIC Power Management option was not visible in the device UI | Could not disable NIC power-saving through standard UI | Provided two registry-based methods to bypass the UI limitation |
| Root cause involved multi-component interaction (Power Management + SMB + Kerberos); examining any single component couldn't fully explain the issue | Required correlating multiple log sources to reconstruct the complete event chain | Cross-analyzed network capture, SMB ETL, Kerberos ETL, and System Event Log; assembled the complete logical chain step by step |

## 5. Root Cause & Resolution

### Root Cause

**Power management powered off the NIC during standby, causing SMB session disconnect and credential loss, triggering a Kerberos authentication failure chain on wake.**

Detailed mechanism:
1. Device enters standby → power management powers off the NIC (Reason: NIC compliance)
2. Network disconnect causes NAS SMB session to expire → Mapped drive disconnected → SMB cache (including guest credentials) cleared
3. On wake, the SMB client attempts reconnection; with credentials lost, falls back to the current domain account for Kerberos authentication
4. NIC just recovered, DC not yet reachable → Kerberos TGT request fails → `STATUS_NO_LOGON_SERVERS (0xc000005e)`
5. Authentication fails with `0xc0000388` (cannot contact DC), user prompted to re-enter credentials

### Resolution (Three Approaches)

#### Approach 1: Prevent NIC Power-Off During Standby (Root Fix)

Disable NIC power management to prevent network disconnect during standby:

**Method A — PlatformAoAcOverride Registry:**
1. Open Registry Editor, navigate to `HKLM\SYSTEM\CurrentControlSet\Control\Power`
2. Create DWORD value `PlatformAoAcOverride`, set to `0`
3. Restart the device
4. Open Device Manager → Find NIC → Properties → Power Management → Uncheck **"Allow the computer to turn off this device to save power"**

**Method B — PnPCapabilities Registry:**
1. Navigate to `HKLM\SYSTEM\CurrentControlSet\Control\Class\{4D36E972-E325-11CE-BFC1-08002bE10318}\<DeviceNumber>`
2. Identify the correct NIC device number by checking the Driver Description under each device number
3. Set `PnPCapabilities` to `24` (decimal)
4. Restart the device

#### Approach 2: Use Global Map Drive to Persist Credentials

Use Global Mapping so credentials are automatically saved in Credential Manager and survive session disconnects:

```powershell
# Method A: net use with /global parameter
net use Z: \\nas-file01.cn-region.nas.cloudprovider.com\myshare /global /user:guest *

# Method B: PowerShell New-SmbGlobalMapping
$cred = Get-Credential
New-SmbGlobalMapping -RemotePath "\\nas-file01.cn-region.nas.cloudprovider.com\myshare" -Credential $cred -LocalPath Z:
```

#### Approach 3: Per-User Map Drive + Pre-Save Credentials

Keep using Per-User Map Drive but pre-save guest credentials to Credential Manager:

```cmd
# Method A: Use cmdkey to pre-save credentials
cmdkey /add:nas-file01.cn-region.nas.cloudprovider.com /user:guest /pass:<password>
net use Z: \\nas-file01.cn-region.nas.cloudprovider.com\myshare /savecred /persistent:yes

# Method B: Check "Remember my credentials" when mapping via UI
```

### Workaround

Before implementing the above solutions, users can manually run `net use` with re-entered credentials to restore the connection.

## 6. Lessons Learned

- **Technical Knowledge**:
  - **Modern Standby and NIC Compliance**: In Modern Standby mode, power management may power off the NIC due to NIC compliance, causing a complete network disconnect. This cascading impacts all network-dependent services (SMB sessions, VPN, remote desktop, etc.). The `Microsoft-Windows-Kernel-Power` event source with `Connectivity state in standby: Disconnected` in System Event Log is the key indicator.
  - **SMB Mapped Drive Credential Loss Mechanism**: When an SMB session disconnects and the cache is cleared, Per-User Map Drive credentials (especially non-domain accounts like guest) are not retained. On reconnection, the SMB client falls back to the current logged-in account for authentication — if it's a domain account, this triggers Kerberos authentication flow.
  - **Global Map Drive vs Per-User Map Drive**: Global Map Drive (`/global` parameter or `New-SmbGlobalMapping`) persists credentials to Credential Manager and can auto-reconnect after session loss. Per-User Map Drive credentials are only valid for the current session unless explicitly saved via `cmdkey` or "Remember my credentials".
  - **PlatformAoAcOverride Registry**: When the NIC Power Management option is not visible in Device Manager (common on Modern Standby devices), add `PlatformAoAcOverride = 0` to `HKLM\SYSTEM\CurrentControlSet\Control\Power` and restart to make the option visible.

- **Troubleshooting Methodology**:
  - **Standby/Wake Network Issue Investigation Layers**: ① System Event Log (power events, NIC disconnect) → ② Network capture (connection establishment) → ③ SMB Client ETL (MUP/CSC errors) → ④ Kerberos ETL (authentication failures) → ⑤ Credential Manager state.
  - **Multi-Component Interaction Analysis**: When an issue involves power management, networking, SMB, and Kerberos, examining any single component may not fully explain the problem. Multiple log sources must be **time-aligned** to reconstruct the complete event chain.
  - **Error Code Trace Chain**: `0xc0000388` (cannot contact DC) → `0xc000005e` (STATUS_NO_LOGON_SERVERS) → NIC compliance disconnect. Trace backward from the application-layer error to the hardware-layer root cause.

- **Prevention**:
  - When using NAS Mapped Drives in cloud desktop/VDI environments, **prefer Global Mapping or pre-saved credentials** to prevent credential loss after standby/wake cycles.
  - For scenarios requiring persistent network connectivity, evaluate whether **NIC power management should be disabled** or Modern Standby policies adjusted.
  - For bulk deployment, use GPO or scripts to uniformly configure `PnPCapabilities` registry or `PlatformAoAcOverride`.

## 7. References

- [New-SmbGlobalMapping — PowerShell](https://learn.microsoft.com/en-us/powershell/module/smbshare/new-smbglobalmapping) — SMB Global Mapping PowerShell command reference
- [SMB Overview — Windows Server](https://learn.microsoft.com/en-us/windows-server/storage/file-server/file-server-smb-overview) — SMB protocol overview
- [Modern Standby Network Connectivity — Windows Hardware](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/modern-standby-network-connectivity) — Modern Standby network connectivity behavior documentation
