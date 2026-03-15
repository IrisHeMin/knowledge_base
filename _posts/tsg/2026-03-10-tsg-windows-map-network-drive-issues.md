---
layout: post
title: "TSG: Windows 映射网络驱动器故障排查"
date: 2026-03-10
categories: [TSG, SMB]
tags: [tsg, map-drive, net-use, smb, credential-manager, smbglobalmapping, network-drive, file-explorer, persistent]
type: "tsg"
---

# TSG: Windows 映射网络驱动器故障排查

**Product/Service:** Windows (File Explorer / net use / SmbGlobalMapping)
**Category:** SMB / Network Drive Mapping
**Severity Guidance:** P2-重要（影响用户日常文件访问）/ P3-一般（配置指导类）
**Last Updated:** 2026-03-10

---

## 1. 适用场景 (When to Use This TSG)

当客户报告映射网络驱动器（Map Network Drive）相关问题时，使用本 TSG：

- **典型症状**：
  - 映射的网络驱动器在重启后**消失**了
  - 映射的网络驱动器重启后显示**红叉（❌）**，无法访问
  - 映射网络驱动器时弹出"Access Denied"或反复要求输入凭据
  - 不同用户看到不同的映射驱动器，或某些用户**看不到**映射驱动器
  - `net use` 命令映射行为**不一致**：有的设备上驱动器消失，有的不消失
  - `SmbGlobalMapping` 创建后其他用户看到了但**无法访问**

- **适用条件**：
  - Windows 10 / Windows 11 / Windows Server 2016+
  - 使用以下任一方式映射驱动器：File Explorer GUI、`net use` 命令、`New-SmbGlobalMapping`
  - SMB 共享目标为标准 SMB 服务器（Windows Server、NAS、Azure Files 等）

- **不适用场景**：
  - DFS Namespace 相关故障（SmbGlobalMapping 不支持 DFS，但 GUI 和 net use 支持）
  - SMB 传输性能问题（如拷贝慢、高延迟），参考 SMB 性能 TSG
  - 共享权限本身配置错误（NTFS/Share Permission 问题，属于服务器端配置）
  - UNC 路径 `\\server\share` 本身不可达（属于网络连接或 DNS 问题，需先排查网络层）

## 2. 快速检查清单 (Quick Checklist)

在深入排查之前，先确认以下基本项（通常 5 分钟内可以快速定位方向）：

- [ ] **UNC 路径可达** — 直接在 File Explorer 中输入 `\\<server>\<share>`，能否正常访问？如果不行，问题不在映射层
- [ ] **确认映射方法** — 客户使用的是 File Explorer GUI、`net use` 还是 `New-SmbGlobalMapping`？不同方法的持久化机制完全不同
- [ ] **确认持久化设置** — 是否启用了 "Reconnect at sign-in" / `/persistent:yes` / `-Persistent $true`？
- [ ] **确认凭据保存** — 是否勾选了 "Remember my credentials" / 使用了 `/savecred`？Credential Manager 中是否有对应条目？
- [ ] **确认用户身份** — 当前登录账户是域账户还是本地账户？该账户是否有权限直接访问目标共享？
- [ ] **检查注册表** — Per-User: `HKCU\Network\<Drive>`；Per-Machine: `HKLM\...\GlobalMappings\<Drive>` 是否存在？

> 如果 UNC 路径本身不可达，问题不在映射驱动器层面，需先排查 DNS 解析、网络连通性和 SMB 服务状态。

## 3. 排查逻辑 (Troubleshooting Logic)

### 3.1 可能的根因 (Possible Root Causes)

按发生概率从高到低排列：

1. **根因 A — 未启用持久化（最常见）**：映射时未勾选 "Reconnect at sign-in" 或未使用 `/persistent:yes`，映射信息没有保存到注册表，重启后自然丢失
2. **根因 B — 凭据未保存或已失效**：映射路径保存了，但 Credential Manager 中的凭据丢失（未勾选 "Remember my credentials"、密码已更改、或被 GPO 清除），重启后无法认证，驱动器显示红叉
3. **根因 C — `/persistent` 参数继承了意外值**：`net use` 命令未显式指定 `/persistent`，继承了该设备上上次的值，导致不同设备行为不一致
4. **根因 D — `/savecred` 与 `/user:` 冲突**：在同一条 `net use` 命令中同时使用了这两个参数，导致命令报错、凭据未保存
5. **根因 E — 映射方法选错（Per-User vs Per-Machine）**：使用 GUI/net use（Per-User）映射后，其他登录用户看不到；或需要 DFS 但用了 SmbGlobalMapping
6. **根因 F — SmbGlobalMapping 凭据文件损坏**：SmbGlobalMapping 的凭据由 lsass.exe 管理，保存路径的文件损坏导致重启后认证失败

### 3.2 排查思路 (Investigation Approach)

遇到"映射网络驱动器出问题"时，理解映射的本质至关重要：**映射驱动器 = 保存共享路径 + 保存访问凭据**。任何一方丢失，都会导致问题。

排查时，首先要确认**使用了哪种映射方法**，因为三种方法的作用范围（Per-User vs Per-Machine）、持久化存储位置（注册表路径不同）和凭据存储方式（Credential Manager vs lsass.exe）完全不同，不能混淆。

确认方法后，按 **"路径是否保存 → 凭据是否保存 → 凭据是否有效"** 的顺序逐步检查。大多数问题会在前两步就定位到。

```
[映射驱动器故障]
    ↓
确认映射方法（GUI / net use / SmbGlobalMapping）
    ↓
┌── Per-User（GUI / net use）──────────────────────────┐
│                                                       │
│  检查注册表 HKCU\Network\<Drive> 是否存在？            │
│    ↓ 不存在 → 解决方案 A（未持久化）                     │
│    ↓ 存在                                              │
│  检查 Credential Manager 中是否有目标服务器凭据？        │
│    ↓ 没有 → 解决方案 B（凭据未保存）                     │
│    ↓ 有                                                │
│  直接访问 \\server\share 是否成功？                      │
│    ↓ 失败 → 解决方案 C（凭据失效）                       │
│    ↓ 成功 → 解决方案 D（SMB 会话/服务问题）              │
└───────────────────────────────────────────────────────┘
│
├── Per-Machine（SmbGlobalMapping）────────────────────┐
│                                                       │
│  Get-SmbGlobalMapping 是否返回映射？                    │
│    ↓ 不存在 → 解决方案 E（未持久化 / 已丢失）            │
│    ↓ 存在                                              │
│  目标是否为 DFS 路径？                                  │
│    ↓ 是 DFS → 解决方案 F（不支持 DFS）                  │
│    ↓ 非 DFS → 检查凭据文件 → 解决方案 G                 │
└───────────────────────────────────────────────────────┘
```

## 4. 数据收集与排查步骤 (Data Collection & Investigation Steps)

### 方向 A: 确认映射方法和持久化状态

**为什么先查这个：** 三种映射方法的持久化机制完全不同，只有先确认方法，后续排查才有方向。

**收集数据：**
```powershell
# 1. 查看当前所有映射驱动器和状态
net use

# 2. 查看 Per-User 持久化映射（File Explorer / net use）
reg query HKCU\Network 2>$null

# 3. 查看 Per-Machine 持久化映射（SmbGlobalMapping）
reg query "HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings" 2>$null

# 4. 查看 SmbGlobalMapping 列表（如使用该方式）
Get-SmbGlobalMapping 2>$null

# 5. 确认当前登录身份
whoami /all
```

**怎么看结果：**
- ✅ `HKCU\Network\<Drive>` 存在 → Per-User 映射已持久化，继续方向 B 检查凭据
- ✅ `HKLM\...\GlobalMappings\<Drive>` 存在 → SmbGlobalMapping 已持久化，跳到方向 D
- ❌ 两处都不存在 → **映射未持久化**，确认为根因 A，转到 **解决方案 A**

### 方向 B: 检查 Per-User 映射的注册表详情

**为什么查这个：** 注册表中保存了共享路径和用户名信息，可以确认路径是否正确、是否使用了自定义凭据。

**收集数据：**
```powershell
# 替换 L 为实际驱动器号
reg query HKCU\Network\L
```

**怎么看结果：**
- ✅ 正常：存在 `RemotePath`（如 `\\fileserver01\shared`）、`UserName`、`ProviderName` 等键值 → 路径保存正常，继续方向 C
- ❌ `UserName` 为空 → 映射时使用的是登录凭据（未指定其他凭据）
- ❌ `RemotePath` 路径错误 → 服务器更名或路径变更，转到 **解决方案 A** 中的重新映射步骤

### 方向 C: 检查 Credential Manager 中的凭据

**为什么查这个：** 即使路径保存了，如果凭据丢失或失效，重启后也无法自动连接，驱动器会显示红叉。

**收集数据：**
```powershell
# 列出所有保存的凭据
cmdkey /list

# 查找特定服务器的凭据
cmdkey /list | findstr /i "<server>"
```

**怎么看结果：**
- ✅ 正常：存在 `Target: Domain:interactive=<server>` 或类似条目，且 `User` 信息正确 → 继续验证凭据有效性
- ❌ 没有目标服务器的凭据条目 → **凭据未保存**，确认为根因 B，转到 **解决方案 B**

**验证凭据是否仍然有效：**
```powershell
# 尝试直接访问 UNC 路径
Test-Path "\\<server>\<share>"

# 或查看当前 SMB 连接
Get-SmbConnection | Where-Object { $_.ServerName -eq "<server>" }
```

- ✅ `Test-Path` 返回 `True` → 凭据有效，映射应该正常。如仍有问题 → 转到 **解决方案 D**
- ❌ `Test-Path` 返回 `False` 或弹出凭据窗口 → **凭据失效**（密码已更改等），转到 **解决方案 C**
- ❌ 路径不可达（网络错误）→ 不是映射问题，需排查网络连接和 SMB 服务

### 方向 D: 检查 SmbGlobalMapping 状态

**为什么查这个：** SmbGlobalMapping 是 Per-Machine 级别映射，凭据存储和持久化路径都与前两种方法不同。

**收集数据：**
```powershell
# 查看 SmbGlobalMapping 状态
Get-SmbGlobalMapping

# 查看注册表持久化信息
reg query "HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings\<DriveLetter>" 2>$null

# 查看凭据文件是否存在
cmd /c "dir /a C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials"
```

**怎么看结果：**
- ✅ `Get-SmbGlobalMapping` 返回映射且 Status 正常 → 如仍有问题，可能是 DFS 路径不支持，转到 **解决方案 F**
- ❌ 映射不存在 → 转到 **解决方案 E**
- ❌ 映射存在但 Status 异常 / 凭据文件缺失 → 转到 **解决方案 G**

## 5. 解决方案 (Solutions)

### 解决方案 A: 映射未持久化 — 重启后驱动器消失

**为什么会出这个问题：**
映射网络驱动器的本质是「保存共享路径 + 保存凭据」。如果映射时未启用持久化选项，路径信息不会写入注册表，系统重启后映射关系自然丢失。对于 `net use` 命令，如果省略 `/persistent` 参数，它会继承该设备上**上次使用的值**，这就是为什么有些设备上驱动器会消失、有些不会——行为取决于历史操作。

**修复步骤：**

**方式一：File Explorer GUI**
1. 打开 File Explorer → 右键 "This PC" → "Map Network Drive"
2. 选择驱动器号，输入共享路径 `\\<server>\<share>`
3. ✅ **勾选** "Reconnect at sign-in"
4. 如需使用其他凭据，勾选 "Connect using different credentials"
5. 点击 Finish，在凭据窗口中 ✅ **勾选** "Remember my credentials"

**方式二：net use 命令**
```powershell
# 先删除已有映射
net use L: /delete 2>$null

# 使用 /savecred，系统会提示输入凭据并自动保存
net use L: \\<server>\<share> /persistent:yes /savecred
```

**方式三：先保存凭据再映射（推荐用于脚本场景）**
```powershell
cmdkey /add:<server> /user:<domain\username> /pass:<password>
net use L: \\<server>\<share> /persistent:yes /savecred
```

**验证修复：**
```powershell
# 确认注册表已保存
reg query HKCU\Network\L

# 确认凭据已保存
cmdkey /list | findstr /i "<server>"
```
预期结果：注册表中存在 `RemotePath` 键值，Credential Manager 中存在对应凭据条目。

**风险提示：**
- ⚠️ `/savecred` 和 `/user:` **不能**在同一条 `net use` 命令中同时使用，否则会报冲突错误。需要先用 `cmdkey` 保存凭据，再用 `/savecred` 映射
- ⚠️ 不指定 `/persistent` 时继承上次的值，建议**始终显式指定** `/persistent:yes` 或 `/persistent:no`

### 解决方案 B: 凭据未保存 — 重启后需要重新输入密码

**为什么会出这个问题：**
映射时勾选了 "Reconnect at sign-in"（路径保存了），但没有勾选 "Remember my credentials"（凭据没保存）。重启后系统能找到共享路径但无法自动认证，驱动器保留但访问时会弹出凭据窗口。

**修复步骤：**
```powershell
# 方法 1：直接添加凭据到 Credential Manager
cmdkey /add:<server> /user:<domain\username> /pass:<password>

# 方法 2：删除后重新映射，这次记得保存凭据
net use L: /delete
net use L: \\<server>\<share> /persistent:yes /savecred
```

**验证修复：**
```powershell
cmdkey /list | findstr /i "<server>"
```
预期结果：Credential Manager 中存在目标服务器的凭据条目。

### 解决方案 C: 凭据失效 — 驱动器显示红叉（❌）

**为什么会出这个问题：**
凭据曾经保存过，但现在已经失效。常见原因：
- 用户密码已更改，保存的旧密码不再有效
- 凭据被手动从 Credential Manager 中删除
- 组策略（GPO）清除了保存的凭据
- 凭据有效期到期

**修复步骤：**
```powershell
# Step 1: 删除旧的无效凭据
cmdkey /delete:<server>

# Step 2: 添加新凭据
cmdkey /add:<server> /user:<domain\username> /pass:<new_password>

# Step 3: 验证连接
net use L: /delete 2>$null
net use L: \\<server>\<share> /persistent:yes /savecred
```

**验证修复：**
```powershell
cmdkey /list | findstr /i "<server>"
Test-Path "L:\"
```
预期结果：Credential Manager 中存在更新后的凭据，`Test-Path` 返回 `True`。

**风险提示：**
⚠️ 有一种特殊情况：如果登录账户是**本地账户**，且该本地账户的密码恰好与映射驱动器使用的账户密码**相同**，即使没有保存凭据，NTLM 认证也可能"意外成功"。这是因为 NTLM 协议会用当前登录账户的密码哈希进行认证，密码一致时恰好能通过。**不要依赖此行为**——一旦任何一方密码更改，映射将立即失败。

### 解决方案 D: 配置均正常但驱动器仍无法使用

**为什么会出这个问题：**
注册表路径和 Credential Manager 凭据都存在且正确，但可能存在 SMB 会话缓存过期、SMB 客户端服务异常等问题。

**修复步骤：**
```powershell
# Step 1: 清除到目标服务器的所有 SMB 会话
net use \\<server> /delete 2>$null

# Step 2: 断开并重新映射
net use L: /delete 2>$null
net use L: \\<server>\<share> /persistent:yes

# Step 3: 如仍失败，检查 SMB 客户端服务
Get-Service LanmanWorkstation
# 如果服务未运行，启动它
Start-Service LanmanWorkstation
```

**验证修复：**
```powershell
net use
Get-SmbConnection | Where-Object { $_.ServerName -eq "<server>" }
Test-Path "L:\"
```

### 解决方案 E: SmbGlobalMapping 未持久化或已丢失

**为什么会出这个问题：**
创建 SmbGlobalMapping 时未设置 `-Persistent $true`，或者省略了该参数（此时继承上次使用的值，默认可能为 `$false`）。

SmbGlobalMapping 的持久化信息保存在两个位置：
- **注册表**：`HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings\<DriveLetter>`
- **凭据文件**：`C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials`（由 lsass.exe 管理）

**修复步骤：**
```powershell
# 方式一：交互式（弹出凭据窗口）
$creds = Get-Credential
New-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Credential $creds -LocalPath "Z:" -Persistent $true

# 方式二：脚本化（适用于自动化部署）
$User = "<Domain\UserName>"
$PWord = ConvertTo-SecureString -String '<password>' -AsPlainText -Force
$Creds = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $User, $PWord
New-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Credential $Creds -LocalPath "Z:" -Persistent $true
```

**验证修复：**
```powershell
Get-SmbGlobalMapping
reg query "HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings\Z" 2>$null
Test-Path "Z:\"
```
预期结果：映射存在、注册表键存在、驱动器可访问。

**风险提示：**
⚠️ 脚本中明文保存密码有安全风险，生产环境建议使用 Azure Key Vault 或其他安全存储方案。

### 解决方案 F: SmbGlobalMapping 不支持 DFS 路径

**为什么会出这个问题：**
SmbGlobalMapping 支持 standalone 和 failover cluster SMB shares，但**不支持 DFS Namespace folder shares**。如果目标路径是 DFS 路径，SmbGlobalMapping 可能创建成功但实际访问时失败。

**修复步骤：**
确认路径是否为 DFS 路径，如果是，改用 `net use` 或 File Explorer GUI 映射（Per-User 方式支持 DFS）。

```powershell
# 删除不支持的 SmbGlobalMapping
Remove-SmbGlobalMapping -RemotePath "\\<server>\<dfs-share>" -Force

# 改用 net use 映射
net use Z: \\<server>\<dfs-share> /persistent:yes /savecred
```

### 解决方案 G: SmbGlobalMapping 凭据文件损坏

**为什么会出这个问题：**
SmbGlobalMapping 的凭据由 lsass.exe 处理，保存在 `C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials` 路径。如果该路径下的凭据文件损坏或丢失，重启后映射无法恢复认证。

**修复步骤：**
```powershell
# Step 1: 删除现有的 SmbGlobalMapping
Remove-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Force

# Step 2: 重新创建（凭据会重新保存）
$creds = Get-Credential
New-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Credential $creds -LocalPath "Z:" -Persistent $true
```

**验证修复：**
```powershell
Get-SmbGlobalMapping
cmd /c "dir /a C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials"
```
预期结果：映射恢复正常，凭据文件已重新生成。

### Workaround

如果持久化映射反复出现问题（特别是在凭据管理复杂的环境中），可以通过 **登录脚本（Logon Script）** 或 **计划任务（Scheduled Task）** 在用户登录时自动执行映射：

```powershell
# 登录脚本示例 — 每次登录时重新建立映射
net use L: /delete 2>$null
net use L: \\<server>\<share> /persistent:no /user:<domain\username> <password>
```

> 此方案的缺点是密码可能需要硬编码在脚本中，需注意安全性。可结合 GPO 分发以降低风险。

## 6. 升级指引 (Escalation Guidance)

当以下条件满足时，应考虑升级：

- 注册表路径和凭据均正确配置，但映射驱动器仍无法在重启后恢复 — 可能涉及 SMB 客户端驱动（MUP/MRxSMB）问题
- SmbGlobalMapping 持久化后凭据文件反复丢失 — 可能涉及 lsass.exe 凭据存储缺陷
- 大规模部署场景下不同设备映射行为不一致 — 可能涉及 Group Policy 或 `/persistent` 参数继承问题
- NTLM 认证行为异常（非密码匹配场景下意外成功或失败）— 可能涉及安全策略问题

**升级时需提供的信息：**
- 已完成的排查步骤和每步结果（按本 TSG 记录）
- `net use`、`reg query HKCU\Network`、`cmdkey /list` 的输出
- `Get-SmbGlobalMapping`（如使用该方法）
- 使用的映射方式及具体参数
- 问题是否可稳定复现 + 复现步骤
- 客户环境信息：OS 版本、域/工作组、认证方式（Kerberos/NTLM）

## 7. 相关知识 (Related Knowledge)

### 技术原理：三种映射方法的核心区别

映射网络驱动器的本质是 **「保存共享路径 + 保存访问凭据」**，丢失任何一方都会导致问题。

| 特性 | File Explorer GUI | net use 命令 | SmbGlobalMapping |
|------|------------------|-------------|-----------------|
| **作用范围** | Per-User | Per-User | Per-Machine（所有用户可见） |
| **持久化注册表** | `HKCU\Network\<Drive>` | `HKCU\Network\<Drive>` | `HKLM\...\GlobalMappings\<Drive>` |
| **凭据存储** | Credential Manager | Credential Manager | lsass.exe → `NetworkService\Credentials` |
| **DFS 支持** | ✅ 支持 | ✅ 支持 | ❌ 不支持 |
| **Failover Cluster 支持** | ✅ 支持 | ✅ 支持 | ✅ 支持 |
| **主要使用场景** | 终端用户交互 | 脚本 / 批处理 | Windows Containers |
| **凭据保存选项** | "Remember my credentials" | `/savecred` | 创建时必须提供 `-Credential` |

### GUI / net use 参数对应关系

| GUI 选项 | net use 参数 | 功能 |
|---------|-------------|------|
| "Reconnect at sign-in" | `/persistent:yes` | 保存共享路径到注册表 |
| "Connect using different credentials" | `/user:<domain\user> <password>` | 使用非登录凭据 |
| "Remember my credentials" | `/savecred` | 保存凭据到 Credential Manager |

### 常见误区

- **误区 1**：`net use` 不加 `/persistent` 参数等于不持久化
  - ❌ 错！它会继承该设备上**上次使用的值**，建议始终显式指定
- **误区 2**：`/savecred` 和 `/user:` 可以一起使用
  - ❌ 错！同一条命令中同时使用会报冲突错误。需要分开：先 `cmdkey` 保存凭据，再 `net use /savecred`
- **误区 3**：SmbGlobalMapping 可以替代 net use 用于所有场景
  - ❌ 错！SmbGlobalMapping 不支持 DFS Namespace，且主要设计用于 Windows Containers
- **误区 4**：没保存凭据但重启后映射居然还能用
  - 这可能是 NTLM 认证的特殊行为 — 当登录账户是本地账户且密码与映射凭据密码相同时，NTLM 哈希碰巧能通过认证。**不要依赖此行为**

### 关键注册表路径

- **`HKCU\Network\<DriveLetter>`**：Per-User 映射的持久化信息
  - `RemotePath`：共享 UNC 路径
  - `UserName`：映射使用的账户（为空表示使用登录凭据）
  - `ProviderName`：网络提供程序名称
- **`HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings\<DriveLetter>`**：SmbGlobalMapping 持久化信息

## 8. 参考资料 (References)

- [New-SmbGlobalMapping (SmbShare) | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/smbshare/new-smbglobalmapping) — SmbGlobalMapping cmdlet 官方文档，含全部参数说明
- [Net use | Microsoft Learn](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/gg651155(v=ws.11)) — net use 命令参考，含 /persistent 和 /savecred 等参数详解
- [Cmdkey | Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/cmdkey) — cmdkey 命令参考，用于管理 Credential Manager 中的凭据

## 修订历史 (Revision History)

| 日期 | 版本 | 修改内容 | 来源 |
|------|------|---------|------|
| 2026-03-10 | 1.0 | 初始版本 | 基于 Windows Map Network Drive 技术研究文档提炼 |

---

# TSG: Windows Map Network Drive Troubleshooting

**Product/Service:** Windows (File Explorer / net use / SmbGlobalMapping)
**Category:** SMB / Network Drive Mapping
**Severity Guidance:** P2-Important (impacts daily file access) / P3-Normal (configuration guidance)
**Last Updated:** 2026-03-10

---

## 1. When to Use This TSG

Use this TSG when customers report issues related to mapped network drives:

- **Typical Symptoms:**
  - Mapped network drive **disappears** after reboot
  - Mapped network drive shows a **red cross (❌)** and is inaccessible after reboot
  - "Access Denied" error or repeated credential prompts when mapping a drive
  - Different users see different mapped drives, or some users **cannot see** the mapped drive
  - **Inconsistent** `net use` behavior: drives disappear on some devices but not others
  - `SmbGlobalMapping` created but other users **cannot access** it

- **Applicable Conditions:**
  - Windows 10 / Windows 11 / Windows Server 2016+
  - Drive mapped via File Explorer GUI, `net use` command, or `New-SmbGlobalMapping`
  - SMB share target is a standard SMB server (Windows Server, NAS, Azure Files, etc.)

- **Not Applicable:**
  - DFS Namespace issues (SmbGlobalMapping does not support DFS, but GUI and net use do)
  - SMB transfer performance issues (slow copy, high latency) — refer to SMB performance TSG
  - Share permission misconfiguration (NTFS/Share Permission issues on the server side)
  - UNC path `\\server\share` itself is unreachable (network connectivity or DNS issue — investigate network layer first)

## 2. Quick Checklist

Before deep investigation, verify these basics (typically takes under 5 minutes):

- [ ] **UNC path reachable** — Type `\\<server>\<share>` directly in File Explorer. If unreachable, the issue is not at the mapping layer
- [ ] **Confirm mapping method** — Did the customer use File Explorer GUI, `net use`, or `New-SmbGlobalMapping`? Each has completely different persistence mechanisms
- [ ] **Confirm persistence setting** — Was "Reconnect at sign-in" / `/persistent:yes` / `-Persistent $true` enabled?
- [ ] **Confirm credential saving** — Was "Remember my credentials" checked / `/savecred` used? Is there a matching entry in Credential Manager?
- [ ] **Confirm user identity** — Is the logon account a domain or local account? Does it have direct permission to access the target share?
- [ ] **Check registry** — Per-User: `HKCU\Network\<Drive>`; Per-Machine: `HKLM\...\GlobalMappings\<Drive>` — does it exist?

> If the UNC path itself is unreachable, the issue is not at the drive mapping layer. Investigate DNS resolution, network connectivity, and SMB service status first.

## 3. Troubleshooting Logic

### 3.1 Possible Root Causes

Ordered by probability from most to least common:

1. **Root Cause A — Persistence not enabled (most common):** "Reconnect at sign-in" was not checked or `/persistent:yes` was not used. Mapping info was never saved to the registry, so it's naturally lost after reboot
2. **Root Cause B — Credentials not saved or expired:** The mapping path is saved, but Credential Manager credentials are missing ("Remember my credentials" not checked, password changed, or cleared by GPO). After reboot, authentication fails and the drive shows a red cross
3. **Root Cause C — `/persistent` parameter inherited an unexpected value:** `net use` was run without explicitly specifying `/persistent`, so it inherited the last used value on that machine, causing inconsistent behavior across devices
4. **Root Cause D — `/savecred` and `/user:` conflict:** Both parameters were used in the same `net use` command, causing an error and preventing credential saving
5. **Root Cause E — Wrong mapping method (Per-User vs Per-Machine):** Used GUI/net use (Per-User) but expected other users to see it; or need DFS but used SmbGlobalMapping
6. **Root Cause F — SmbGlobalMapping credential file corrupted:** SmbGlobalMapping credentials are managed by lsass.exe. Corruption of the credential file causes authentication failure after reboot

### 3.2 Investigation Approach

When facing "mapped network drive issues," understanding the essence of drive mapping is critical: **mapping a drive = saving the share path + saving access credentials**. Losing either one causes problems.

First, identify **which mapping method was used**, because the three methods have completely different scopes (Per-User vs Per-Machine), persistence storage locations (different registry paths), and credential storage mechanisms (Credential Manager vs lsass.exe). These must not be confused.

After confirming the method, check in this order: **"Is the path saved? → Are credentials saved? → Are credentials valid?"** Most issues are identified in the first two steps.

```
[Map Drive Issue]
    ↓
Identify mapping method (GUI / net use / SmbGlobalMapping)
    ↓
┌── Per-User (GUI / net use) ─────────────────────────┐
│                                                       │
│  Does HKCU\Network\<Drive> exist?                     │
│    ↓ No → Solution A (not persisted)                  │
│    ↓ Yes                                              │
│  Is there a credential in Credential Manager?         │
│    ↓ No → Solution B (credentials not saved)          │
│    ↓ Yes                                              │
│  Can \\server\share be accessed directly?              │
│    ↓ No → Solution C (credentials expired)            │
│    ↓ Yes → Solution D (SMB session/service issue)     │
└───────────────────────────────────────────────────────┘
│
├── Per-Machine (SmbGlobalMapping) ───────────────────┐
│                                                       │
│  Does Get-SmbGlobalMapping return the mapping?        │
│    ↓ No → Solution E (not persisted / lost)           │
│    ↓ Yes                                              │
│  Is the target a DFS path?                            │
│    ↓ Yes → Solution F (DFS not supported)             │
│    ↓ No → Check credential file → Solution G          │
└───────────────────────────────────────────────────────┘
```

## 4. Data Collection & Investigation Steps

### Direction A: Confirm Mapping Method and Persistence Status

**Why check this first:** The three mapping methods have completely different persistence mechanisms. Identifying the method gives direction for all subsequent steps.

**Collect data:**
```powershell
# 1. View all current mapped drives and status
net use

# 2. Check Per-User persistent mappings (File Explorer / net use)
reg query HKCU\Network 2>$null

# 3. Check Per-Machine persistent mappings (SmbGlobalMapping)
reg query "HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings" 2>$null

# 4. Check SmbGlobalMapping list (if using this method)
Get-SmbGlobalMapping 2>$null

# 5. Confirm current logon identity
whoami /all
```

**How to read results:**
- ✅ `HKCU\Network\<Drive>` exists → Per-User mapping is persisted, continue to Direction B
- ✅ `HKLM\...\GlobalMappings\<Drive>` exists → SmbGlobalMapping is persisted, jump to Direction D
- ❌ Neither location has the entry → **Mapping was not persisted**, confirmed as Root Cause A → go to **Solution A**

### Direction B: Check Per-User Mapping Registry Details

**Why check this:** The registry stores the share path and username information, which lets you confirm whether the path is correct and whether custom credentials were used.

**Collect data:**
```powershell
# Replace L with the actual drive letter
reg query HKCU\Network\L
```

**How to read results:**
- ✅ Normal: Contains `RemotePath` (e.g., `\\fileserver01\shared`), `UserName`, `ProviderName` → Path is saved correctly, continue to Direction C
- ❌ `UserName` is empty → Logon credentials were used (no custom credentials specified)
- ❌ `RemotePath` shows wrong path → Server renamed or path changed, go to remap steps in **Solution A**

### Direction C: Check Credential Manager

**Why check this:** Even if the path is saved, missing or expired credentials mean the system can't authenticate after reboot, showing a red cross on the drive.

**Collect data:**
```powershell
# List all saved credentials
cmdkey /list

# Search for the target server
cmdkey /list | findstr /i "<server>"
```

**How to read results:**
- ✅ Normal: An entry exists for the target server with correct user info → Continue to validate credential effectiveness
- ❌ No credential entry for target server → **Credentials not saved**, confirmed as Root Cause B → go to **Solution B**

**Validate credential effectiveness:**
```powershell
# Try direct UNC access
Test-Path "\\<server>\<share>"

# Check current SMB connections
Get-SmbConnection | Where-Object { $_.ServerName -eq "<server>" }
```

- ✅ `Test-Path` returns `True` → Credentials are valid, mapping should work. If still failing → go to **Solution D**
- ❌ `Test-Path` returns `False` or credential prompt appears → **Credentials expired** → go to **Solution C**
- ❌ Path unreachable (network error) → Not a mapping issue, investigate network and SMB service

### Direction D: Check SmbGlobalMapping Status

**Why check this:** SmbGlobalMapping is a per-machine level mapping with different credential storage and persistence paths.

**Collect data:**
```powershell
# Check SmbGlobalMapping status
Get-SmbGlobalMapping

# Check persistence registry
reg query "HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings\<DriveLetter>" 2>$null

# Check credential files
cmd /c "dir /a C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials"
```

**How to read results:**
- ✅ `Get-SmbGlobalMapping` returns the mapping with normal Status → If still failing, may be a DFS path → go to **Solution F**
- ❌ Mapping does not exist → go to **Solution E**
- ❌ Mapping exists but Status is abnormal / credential files missing → go to **Solution G**

## 5. Solutions

### Solution A: Mapping Not Persisted — Drive Disappears After Reboot

**Why this happens:**
The essence of mapping a network drive is "save the share path + save credentials." If persistence was not enabled, the path info is never written to the registry, so the mapping is naturally lost after reboot. For `net use`, if `/persistent` is omitted, it **inherits the last used value** on that machine — this is why drives disappear on some devices but not others.

**Fix steps:**

**Option 1: File Explorer GUI**
1. Open File Explorer → Right-click "This PC" → "Map Network Drive"
2. Select a drive letter, enter the share path `\\<server>\<share>`
3. ✅ **Check** "Reconnect at sign-in"
4. If using different credentials, check "Connect using different credentials"
5. Click Finish, and in the credential prompt ✅ **check** "Remember my credentials"

**Option 2: net use command**
```powershell
# Delete any existing mapping
net use L: /delete 2>$null

# Map with /savecred — system will prompt for credentials and save them
net use L: \\<server>\<share> /persistent:yes /savecred
```

**Option 3: Save credentials first, then map (recommended for scripts)**
```powershell
cmdkey /add:<server> /user:<domain\username> /pass:<password>
net use L: \\<server>\<share> /persistent:yes /savecred
```

**Verify fix:**
```powershell
# Confirm registry is saved
reg query HKCU\Network\L

# Confirm credentials are saved
cmdkey /list | findstr /i "<server>"
```
Expected: Registry contains `RemotePath` key, Credential Manager shows the credential entry.

**Risk note:**
- ⚠️ `/savecred` and `/user:` **cannot** be used together in the same `net use` command — this causes a conflict error. Use `cmdkey` to save credentials first, then map with `/savecred`
- ⚠️ Omitting `/persistent` inherits the last used value — **always explicitly specify** `/persistent:yes` or `/persistent:no`

### Solution B: Credentials Not Saved — Password Required After Reboot

**Why this happens:**
"Reconnect at sign-in" was checked (path is saved), but "Remember my credentials" was not checked (credentials are not saved). After reboot, the system finds the share path but cannot auto-authenticate, so the drive is retained but a credential prompt appears on access.

**Fix steps:**
```powershell
# Option 1: Add credentials directly to Credential Manager
cmdkey /add:<server> /user:<domain\username> /pass:<password>

# Option 2: Delete and remap with credential saving
net use L: /delete
net use L: \\<server>\<share> /persistent:yes /savecred
```

**Verify fix:**
```powershell
cmdkey /list | findstr /i "<server>"
```
Expected: Credential Manager shows an entry for the target server.

### Solution C: Credentials Expired — Drive Shows Red Cross (❌)

**Why this happens:**
Credentials were previously saved but are now invalid. Common causes:
- User password was changed; the saved old password no longer works
- Credentials were manually deleted from Credential Manager
- Group Policy (GPO) cleared saved credentials
- Credentials expired

**Fix steps:**
```powershell
# Step 1: Delete old invalid credentials
cmdkey /delete:<server>

# Step 2: Add new credentials
cmdkey /add:<server> /user:<domain\username> /pass:<new_password>

# Step 3: Verify connection
net use L: /delete 2>$null
net use L: \\<server>\<share> /persistent:yes /savecred
```

**Verify fix:**
```powershell
cmdkey /list | findstr /i "<server>"
Test-Path "L:\"
```
Expected: Updated credentials in Credential Manager, `Test-Path` returns `True`.

**Risk note:**
⚠️ There is a special case: if the logon account is a **local account** and that local account's password happens to be **identical** to the mapped drive credential password, NTLM authentication may "unexpectedly succeed" even without saved credentials. This is because NTLM uses the current logon account's password hash for authentication, and matching passwords happen to pass. **Do not rely on this behavior** — if either password changes, the mapping will immediately fail.

### Solution D: Configuration Is Correct but Drive Still Fails

**Why this happens:**
Registry path and Credential Manager credentials both exist and are correct, but there may be stale SMB session caches or SMB client service issues.

**Fix steps:**
```powershell
# Step 1: Clear all SMB sessions to the target server
net use \\<server> /delete 2>$null

# Step 2: Disconnect and remap
net use L: /delete 2>$null
net use L: \\<server>\<share> /persistent:yes

# Step 3: If still failing, check SMB client service
Get-Service LanmanWorkstation
# If not running, start it
Start-Service LanmanWorkstation
```

**Verify fix:**
```powershell
net use
Get-SmbConnection | Where-Object { $_.ServerName -eq "<server>" }
Test-Path "L:\"
```

### Solution E: SmbGlobalMapping Not Persisted or Lost

**Why this happens:**
`-Persistent $true` was not set when creating SmbGlobalMapping, or the parameter was omitted (it inherits the last used value, which may default to `$false`).

SmbGlobalMapping persistence is stored in two locations:
- **Registry:** `HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings\<DriveLetter>`
- **Credential file:** `C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials` (managed by lsass.exe)

**Fix steps:**
```powershell
# Option 1: Interactive (credential prompt appears)
$creds = Get-Credential
New-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Credential $creds -LocalPath "Z:" -Persistent $true

# Option 2: Scripted (for automated deployment)
$User = "<Domain\UserName>"
$PWord = ConvertTo-SecureString -String '<password>' -AsPlainText -Force
$Creds = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $User, $PWord
New-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Credential $Creds -LocalPath "Z:" -Persistent $true
```

**Verify fix:**
```powershell
Get-SmbGlobalMapping
reg query "HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings\Z" 2>$null
Test-Path "Z:\"
```
Expected: Mapping exists, registry key exists, drive is accessible.

**Risk note:**
⚠️ Storing passwords in plaintext in scripts is a security risk. Use Azure Key Vault or other secure storage in production environments.

### Solution F: SmbGlobalMapping Does Not Support DFS Paths

**Why this happens:**
SmbGlobalMapping supports standalone and failover cluster SMB shares, but **does not support DFS Namespace folder shares**. If the target path is a DFS path, SmbGlobalMapping may appear to create successfully but fail during actual access.

**Fix steps:**
Confirm whether the path is a DFS path. If so, switch to `net use` or File Explorer GUI (Per-User methods that support DFS).

```powershell
# Remove unsupported SmbGlobalMapping
Remove-SmbGlobalMapping -RemotePath "\\<server>\<dfs-share>" -Force

# Use net use instead
net use Z: \\<server>\<dfs-share> /persistent:yes /savecred
```

### Solution G: SmbGlobalMapping Credential File Corrupted

**Why this happens:**
SmbGlobalMapping credentials are handled by lsass.exe and stored at `C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials`. If the credential file at this path is corrupted or missing, the mapping cannot authenticate after reboot.

**Fix steps:**
```powershell
# Step 1: Remove the existing SmbGlobalMapping
Remove-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Force

# Step 2: Recreate (credentials will be re-saved)
$creds = Get-Credential
New-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Credential $creds -LocalPath "Z:" -Persistent $true
```

**Verify fix:**
```powershell
Get-SmbGlobalMapping
cmd /c "dir /a C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials"
```
Expected: Mapping restored, credential files regenerated.

### Workaround

If persistent mapping repeatedly fails (especially in complex credential management environments), use a **Logon Script** or **Scheduled Task** to automatically map on user logon:

```powershell
# Logon script example — re-establish mapping on each logon
net use L: /delete 2>$null
net use L: \\<server>\<share> /persistent:no /user:<domain\username> <password>
```

> Downside: the password may need to be hardcoded in the script. Consider distributing via GPO to mitigate the risk.

## 6. Escalation Guidance

Consider escalation when:

- Registry path and credentials are correctly configured but the mapped drive still fails to restore after reboot — may involve SMB client driver (MUP/MRxSMB) issues
- SmbGlobalMapping credential files repeatedly disappear after persistence is set — may involve lsass.exe credential storage defects
- Inconsistent mapping behavior across devices in large-scale deployment — may involve Group Policy or `/persistent` parameter inheritance issues
- Abnormal NTLM authentication behavior (unexpected success or failure outside the password-match scenario) — may involve security policy issues

**Information to provide when escalating:**
- Completed troubleshooting steps and results from each step (documented per this TSG)
- Output of `net use`, `reg query HKCU\Network`, `cmdkey /list`
- Output of `Get-SmbGlobalMapping` (if using this method)
- Mapping method and specific parameters used
- Whether the issue is consistently reproducible + repro steps
- Customer environment: OS version, domain/workgroup, authentication method (Kerberos/NTLM)

## 7. Related Knowledge

### Technical Principle: Core Differences Between Three Mapping Methods

The essence of mapping a network drive is **"save the share path + save access credentials"** — losing either one causes problems.

| Feature | File Explorer GUI | net use Command | SmbGlobalMapping |
|---------|------------------|----------------|-----------------|
| **Scope** | Per-User | Per-User | Per-Machine (visible to all users) |
| **Persistence Registry** | `HKCU\Network\<Drive>` | `HKCU\Network\<Drive>` | `HKLM\...\GlobalMappings\<Drive>` |
| **Credential Storage** | Credential Manager | Credential Manager | lsass.exe → `NetworkService\Credentials` |
| **DFS Support** | ✅ Supported | ✅ Supported | ❌ Not supported |
| **Failover Cluster Support** | ✅ Supported | ✅ Supported | ✅ Supported |
| **Primary Use Case** | End user interaction | Scripts / Batch | Windows Containers |
| **Credential Save Option** | "Remember my credentials" | `/savecred` | Must provide `-Credential` at creation |

### GUI / net use Parameter Mapping

| GUI Option | net use Parameter | Function |
|-----------|------------------|----------|
| "Reconnect at sign-in" | `/persistent:yes` | Save share path to registry |
| "Connect using different credentials" | `/user:<domain\user> <password>` | Use non-logon credentials |
| "Remember my credentials" | `/savecred` | Save credentials to Credential Manager |

### Common Misconceptions

- **Misconception 1:** `net use` without `/persistent` defaults to non-persistent
  - ❌ Wrong! It **inherits the last used value** on that machine. Always explicitly specify
- **Misconception 2:** `/savecred` and `/user:` can be used together
  - ❌ Wrong! Using both in the same command causes a conflict error. Separate them: use `cmdkey` to save credentials first, then `net use /savecred`
- **Misconception 3:** SmbGlobalMapping can replace net use for all scenarios
  - ❌ Wrong! SmbGlobalMapping does not support DFS Namespace, and is primarily designed for Windows Containers
- **Misconception 4:** Credentials weren't saved but mapping still works after reboot
  - This may be a special NTLM behavior — when the logon account is a local account and its password matches the mapped credential password, the NTLM hash coincidentally passes. **Do not rely on this behavior**

### Key Registry Paths

- **`HKCU\Network\<DriveLetter>`** — Per-User mapping persistence
  - `RemotePath`: Share UNC path
  - `UserName`: Account used for mapping (empty means logon credentials were used)
  - `ProviderName`: Network provider name
- **`HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings\<DriveLetter>`** — SmbGlobalMapping persistence

## 8. References

- [New-SmbGlobalMapping (SmbShare) | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/smbshare/new-smbglobalmapping) — Official SmbGlobalMapping cmdlet documentation with full parameter details
- [Net use | Microsoft Learn](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/gg651155(v=ws.11)) — Net use command reference with /persistent and /savecred parameter details
- [Cmdkey | Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/cmdkey) — Cmdkey command reference for managing credentials in Credential Manager

## Revision History

| Date | Version | Changes | Source |
|------|---------|---------|--------|
| 2026-03-10 | 1.0 | Initial version | Distilled from Windows Map Network Drive technical research document |
