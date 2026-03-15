---
layout: post
title: "SYSVOL 共享访问失败 — DisableDfs 注册表项导致 DFS Surrogate 被禁用 / SYSVOL Share Access Failure — DisableDfs Registry Key Disabling DFS Surrogate"
date: 2024-12-23
categories: [SMB, DFS, Group-Policy]
tags: [sysvol, dfs, smb, kerberos, mup, surrogate, registry, group-policy, dfsc, referral]

---

# Case Summary: SYSVOL 共享访问失败 — DisableDfs 注册表导致 DFS Surrogate 被禁用

**Product/Service:** Windows SMB Client / DFS / Active Directory Group Policy

---

## 1. 症状 (Symptoms)

- 客户报告无法访问 `\\domain\sysvol` 共享，Group Policy 无法正常应用
- 网络抓包发现 Kerberos TGS 请求中的 SPN 为 `cifs/<domain>` 而非正确的 `cifs/<DC FQDN>`，KDC 返回 `KDC_ERR_S_PRINCIPAL_UNKNOWN (7)` 错误
- SMB Tree Connect 路径为 `\\domain\sysvol`（错误），而非 `\\DC_FQDN\sysvol`（正确）
- 后续 IOCTL `FSCTL_QUERY_NETWORK_INTERFACE_INFO` 返回 `STATUS_OBJECT_NAME_NOT_FOUND`

## 2. 背景 (Background / Environment)

- **操作系统**：Windows 客户端
- **域环境**：Active Directory 域（多 DC 环境）
- **问题发生时间**：2024 年 12 月
- **涉及组件**：SMB Client、DFS Client (dfsc)、MUP (Multiple UNC Provider) 驱动
- **正常行为预期**：
  - Kerberos TGS 请求 SPN: `cifs/<DC FQDN>/<domain>`
  - SMB Tree Connect 路径: `\\DC_FQDN\sysvol`
  - DFS Referral 过程将 `\\domain\sysvol` 解析为 `\\DC_FQDN\sysvol`

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

### Step 1: 收集日志并分析网络抓包

**做了什么**：清除所有缓存，收集 SMB Client ETL + 网络抓包

**发现了什么**：

正常场景（实验室环境）的 Kerberos + SMB 流程：

```
AS Request  → Sname: krbtgt/CONTOSO.LAB
AS Response → Ticket granted
TGS Request → Sname: cifs/DC01.contoso.lab/contoso.lab   ← 正确的 FQDN SPN
TGS Response → Ticket granted
SMB2 SESSION SETUP → 成功
SMB2 TREE CONNECT → Path=\\DC01.contoso.lab\sysvol       ← 正确的 FQDN 路径
SMB2 TREE CONNECT Response → TID=0x1                      ← 成功
```

异常场景（客户环境）的 Kerberos + SMB 流程：

```
TGS Request → Sname: cifs/northwind.gov.com              ← 错误！使用了域名而非 DC FQDN
KRB_ERROR   → KDC_ERR_S_PRINCIPAL_UNKNOWN (7)            ← KDC 找不到 cifs/domain SPN
SMB2 SESSION SETUP → NTLM fallback
SMB2 TREE CONNECT → Path=\\northwind.gov.com\sysvol       ← 错误！直接访问域名路径
IOCTL FSCTL_QUERY_NETWORK_INTERFACE_INFO → STATUS_OBJECT_NAME_NOT_FOUND
```

**得出结论**：SMB 客户端在异常场景中将 DFS 路径当作普通 SMB 路径处理，没有经过 DFS Referral 将 `\\domain\sysvol` 解析为 `\\DC_FQDN\sysvol`

### Step 2: 排除 DNS 解析问题

**做了什么**：根据以往经验（类似 case 中域名解析失败导致相同现象），首先检查域名解析

**发现了什么**：SMB Client ETL 中 KNR（Kerberos Name Resolution）显示解析正常：

```
[rxtdi] SmbWskGetAddressInfoComplete() - KNR completed. Status 0
[rxtdi] RxCeInitiateConnection() - Resolved name northwind.gov.com 
        with 3 IPv4 & 0 IPv6 addresses
```

之前类似 case 中 KNR 状态为失败：
```
[rxtdi] SmbWskGetAddressInfoComplete() - KNR completed. Status c0000225  ← 解析失败
[mrxsmb] SmbCeUpdateServerAvailability() - Added unavailable server
```

**得出结论**：本次 DNS 解析正常，排除名称解析问题

### Step 3: 对比 DFS 客户端日志

**做了什么**：对比正常和异常场景的 DFSC (DFS Client) ETL 日志

**发现了什么**：

正常场景 — DFS Referral 正常工作：
```
[dfsc] DfscRmGetReferral() - domain service name contoso.lab       ← 发起 Referral 请求
[dfsc] DfscRewritePath() - new path is \DC01.contoso.lab\IPC$      ← 重写为 DC FQDN
[dfsc] DfscRewritePath() - new path is \DC01.contoso.lab\SysVol\...\gpt.ini  ← SYSVOL 路径正确
[dfsc] DfscCmInitState() - lookup found ref at ...                  ← 找到 Referral 缓存
```

异常场景 — 没有 DFS Referral：
```
[dfsc] DfscRewritePath() - new path is \northwind.gov.com\sysvol\...\gpt.ini  ← 未重写为 DC FQDN！
[dfsc] DfscCmInitState() - lookup found negative cache entry        ← 负缓存
[dfsc] DfscCmInitState() - context → PATH_TYPE_NON_DFS             ← 当作非 DFS 路径处理
```

**得出结论**：客户端完全没有发起 DFS Referral 请求，导致路径始终保持 `\\domain\sysvol` 不变

### Step 4: 发现 DisableDfs 注册表项

**做了什么**：进一步排查为什么 DFS Referral 没有触发

**发现了什么**：在客户机器上发现以下注册表值：

```
HKLM\SYSTEM\CurrentControlSet\services\MUP
    DisableDfs    REG_DWORD    0x1
```

**得出结论**：`DisableDfs = 1` 禁用了 DFS 功能，这是问题的直接原因

### Step 5: 实验室复现

**做了什么**：在本地实验室环境中手动设置 `DisableDfs = 1`

**发现了什么**：成功复现问题！表现完全一致：
- TGS 请求中 SPN 变为 `cifs/contoso.lab`（域名而非 DC FQDN）
- KDC 返回 `KDC_ERR_S_PRINCIPAL_UNKNOWN (7)`
- Tree Connect 路径为 `\\contoso.lab\sysvol`
- DFSC 日志显示 `PATH_TYPE_NON_DFS`，无 Referral 请求

### Step 6: 深入分析 — DFS Surrogate 失败机制

**做了什么**：与 EE 讨论并深入对比 working/non-working 的 MUP 驱动日志

**发现了什么**：根本差异在于 **DFS Surrogate 调用失败**。

在 SMB 客户端架构中，DFS 和 CSC（Client-Side Caching）Surrogate 是 MUP 驱动的组成部分，在 I/O 操作传递给 UNC Provider 之前被调用。

**Non-working 场景**（DisableDfs = 1）— Surrogate 失败：
```
[rxce]  RxInitializeContext() - IRP ... CREATE \Device\LanmanRedirector\contoso.lab\sysvol
[mup]   MupCallSurrogatePrePost() - IrpContext ...
[mup]   MupCallSurrogatePrePost() - Exit status 0xc0000016    ← Surrogate 调用失败！
[mup]   MupCreate() - FileName \contoso.lab\sysvol status 0xc0000201
```

**Working 场景** — Surrogate 成功：
```
[rxce]  RxInitializeContext() - IRP ... CREATE \Device\LanmanRedirector\contoso.lab\sysvol\contoso.lab
[mup]   MupCallSurrogatePrePost() - Exit status 0x0           ← Surrogate 成功
[mup]   MupCreate() - FileName \contoso.lab\sysvol\contoso.lab ← 正常继续
```

**得出结论**：`DisableDfs = 1` 导致 DFS Surrogate 在 `MupCallSurrogatePrePost()` 阶段直接返回错误 `0xc0000016`，后续 DFS Referral 完全不会触发

### Step 7: SMB Client ETL 中如何定位 DFS Surrogate

**做了什么**：总结了在 SMB Client ETL 中分析 DFS Surrogate 行为的方法

**方法论**：

1. 使用正则表达式过滤 `RxInitializeContext.*Create`（这是 I/O 的第一个调用）
2. 找到包含目标路径的条目（如 `\contoso.lab\sysvol`）
3. 添加 `MupCreate()` 过滤，找到对应路径的条目
4. 添加 `MupCallSurrogatePrePost()` 过滤
5. 在 `RxInitializeContext` 和 `MupCreate()` 之间的 Surrogate 状态中，**第一个通常是 DFS Surrogate**

> **注意**：Surrogate 仅在以下 IRP 类型时被调用：
> `IRP_MJ_CREATE`、`IRP_MJ_CREATE_MAILSLOT`、`IRP_MJ_CREATE_NAMED_PIPE`

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| 初期怀疑 DNS 解析问题（基于历史 case 经验） | 排查方向偏差，需要验证排除 | 检查 SMB Client ETL 中 KNR 状态确认解析正常后排除 |
| DFS/CSC Surrogate 在 ETL 中无明确名称标识 | 难以直接辨别哪个 Surrogate 失败 | 通过 `RxInitializeContext(CREATE)` → `MupCallSurrogatePrePost()` → `MupCreate()` 的调用链顺序定位，第一个 Surrogate 为 DFS |
| 需要理解 MUP 驱动内部 Surrogate 调用机制 | 需要代码级别分析确认行为 | 与 EE 讨论并确认 Surrogate 仅在 `IRP_MJ_CREATE` 类操作时调用 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

客户机器上存在注册表项 `HKLM\SYSTEM\CurrentControlSet\services\MUP\DisableDfs = 1`，该设置禁用了 MUP 驱动的 DFS Surrogate 功能。

当 DFS Surrogate 被禁用时：
1. `MupCallSurrogatePrePost()` 返回错误 `0xc0000016`
2. DFS Referral 请求不会被触发（`DfscRmGetReferral()` 不会调用）
3. `DfscRewritePath()` 无法将 `\\domain\sysvol` 重写为 `\\DC_FQDN\sysvol`
4. SMB 客户端直接以 `\\domain\sysvol` 作为 SMB 路径，使用 `cifs/domain` 作为 Kerberos SPN
5. KDC 找不到 `cifs/domain` 这个 SPN，返回 `KDC_ERR_S_PRINCIPAL_UNKNOWN (7)`
6. SMB 回退到 NTLM 认证或直接连接失败，SYSVOL 不可访问

### Resolution

删除客户端上的 `DisableDfs` 注册表项：

```powershell
# 检查注册表项
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\services\MUP" -Name "DisableDfs"

# 删除注册表项
Remove-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\services\MUP" -Name "DisableDfs"

# 重启后生效（或重启 MUP 驱动相关服务）
Restart-Computer
```

删除该注册表项后，DFS Surrogate 恢复正常工作，SYSVOL 访问恢复。

> **注意**：不建议通过在 DC 上手动添加错误的 SPN（如 `cifs/domain`）来绕过问题，这不是正确的修复方式。

## 6. 经验教训 (Lessons Learned)

- **技术知识**：
  - `HKLM\SYSTEM\CurrentControlSet\services\MUP\DisableDfs` 是一个隐蔽但影响重大的注册表项，设为 1 会完全禁用 DFS 客户端功能
  - SYSVOL 依赖 DFS Referral 将 `\\domain\sysvol` 解析为 `\\DC_FQDN\sysvol`，禁用 DFS 会直接导致 SYSVOL 访问失败
  - DFS Surrogate 和 CSC Surrogate 是 MUP 驱动架构中的中间层，在 I/O 到达 UNC Provider 之前执行
  - Surrogate 仅对 `IRP_MJ_CREATE`、`IRP_MJ_CREATE_MAILSLOT`、`IRP_MJ_CREATE_NAMED_PIPE` 类型的 IRP 调用

- **排查方法**：
  - 当 `cifs/<domain>` SPN 出现在 Kerberos TGS 中时（而非 `cifs/<DC FQDN>`），说明 DFS Referral 未正常工作
  - 排查 DFS Referral 失败时，除了检查 DNS 解析，还需检查 `DisableDfs` 注册表项
  - 在 SMB Client ETL 中定位 DFS Surrogate 的方法：`RxInitializeContext(CREATE)` → `MupCallSurrogatePrePost()` → `MupCreate()` 调用链，Surrogate 状态 `0x0` 为正常，`0xc0000016` 为失败
  - 对比 working/non-working 日志是定位问题的有效方法

- **预防措施**：
  - 在部署脚本或安全加固策略中，应检查是否误设了 `DisableDfs` 注册表项
  - 对于 SYSVOL 访问失败的问题，可将 `DisableDfs` 作为早期排查项之一

## 7. 参考文档 (References)

- [DFS Namespaces Overview](https://learn.microsoft.com/en-us/windows-server/storage/dfs-namespaces/dfs-overview) — DFS 命名空间概述，解释 DFS Referral 机制
- [Enable or Disable Referrals and Client Failback](https://learn.microsoft.com/en-us/windows-server/storage/dfs-namespaces/enable-or-disable-referrals-and-client-failback) — DFS Referral 和客户端回退配置
- [SMB File Server Overview](https://learn.microsoft.com/en-us/windows-server/storage/file-server/file-server-smb-overview) — SMB 协议概述
- [Applying Group Policy Troubleshooting Guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/group-policy/applying-group-policy-troubleshooting-guidance) — Group Policy 应用问题排查指南

---

# Case Summary: SYSVOL Share Access Failure — DisableDfs Registry Key Disabling DFS Surrogate

**Product/Service:** Windows SMB Client / DFS / Active Directory Group Policy

---

## 1. Symptoms

- Customer reported inability to access the `\\domain\sysvol` share; Group Policy failed to apply
- Network capture showed Kerberos TGS request with SPN `cifs/<domain>` instead of the expected `cifs/<DC FQDN>`, resulting in `KDC_ERR_S_PRINCIPAL_UNKNOWN (7)` error
- SMB Tree Connect path was `\\domain\sysvol` (incorrect) instead of `\\DC_FQDN\sysvol` (correct)
- Subsequent IOCTL `FSCTL_QUERY_NETWORK_INTERFACE_INFO` returned `STATUS_OBJECT_NAME_NOT_FOUND`

## 2. Background / Environment

- **Operating System**: Windows client
- **Domain**: Active Directory domain (multi-DC environment)
- **Timeline**: December 2024
- **Components Involved**: SMB Client, DFS Client (dfsc), MUP (Multiple UNC Provider) driver
- **Expected Normal Behavior**:
  - Kerberos TGS request SPN: `cifs/<DC_FQDN>/<domain>`
  - SMB Tree Connect path: `\\DC_FQDN\sysvol`
  - DFS Referral resolves `\\domain\sysvol` → `\\DC_FQDN\sysvol`

## 3. Investigation & Troubleshooting

### Step 1: Collect Logs and Analyze Network Capture

**Action**: Cleared all caches, collected SMB Client ETL + network capture

**Findings**:

Normal scenario (lab environment) — Kerberos + SMB flow:

```
AS Request  → Sname: krbtgt/CONTOSO.LAB
AS Response → Ticket granted
TGS Request → Sname: cifs/DC01.contoso.lab/contoso.lab   ← Correct FQDN SPN
TGS Response → Ticket granted
SMB2 SESSION SETUP → Success
SMB2 TREE CONNECT → Path=\\DC01.contoso.lab\sysvol       ← Correct FQDN path
SMB2 TREE CONNECT Response → TID=0x1                      ← Success
```

Problem scenario (customer environment) — Kerberos + SMB flow:

```
TGS Request → Sname: cifs/northwind.gov.com              ← Wrong! Using domain name, not DC FQDN
KRB_ERROR   → KDC_ERR_S_PRINCIPAL_UNKNOWN (7)            ← KDC cannot find cifs/domain SPN
SMB2 SESSION SETUP → NTLM fallback
SMB2 TREE CONNECT → Path=\\northwind.gov.com\sysvol       ← Wrong! Direct domain path
IOCTL FSCTL_QUERY_NETWORK_INTERFACE_INFO → STATUS_OBJECT_NAME_NOT_FOUND
```

**Conclusion**: The SMB client in the problem scenario was treating the DFS path as a plain SMB path, without going through DFS Referral to resolve `\\domain\sysvol` to `\\DC_FQDN\sysvol`

### Step 2: Rule Out DNS Resolution Issues

**Action**: Based on prior experience with a similar case (DNS resolution failure causing the same symptoms), first verified domain name resolution

**Findings**: SMB Client ETL showed KNR (Kerberos Name Resolution) completed successfully:

```
[rxtdi] SmbWskGetAddressInfoComplete() - KNR completed. Status 0
[rxtdi] RxCeInitiateConnection() - Resolved name northwind.gov.com 
        with 3 IPv4 & 0 IPv6 addresses
```

In the previous similar case, KNR status showed failure:
```
[rxtdi] SmbWskGetAddressInfoComplete() - KNR completed. Status c0000225  ← Resolution failed
[mrxsmb] SmbCeUpdateServerAvailability() - Added unavailable server
```

**Conclusion**: DNS resolution was working correctly; ruled out name resolution as the cause

### Step 3: Compare DFS Client Logs

**Action**: Compared DFSC (DFS Client) ETL logs between working and non-working scenarios

**Findings**:

Working scenario — DFS Referral functions correctly:
```
[dfsc] DfscRmGetReferral() - domain service name contoso.lab       ← Referral request initiated
[dfsc] DfscRewritePath() - new path is \DC01.contoso.lab\IPC$      ← Rewritten to DC FQDN
[dfsc] DfscRewritePath() - new path is \DC01.contoso.lab\SysVol\...\gpt.ini  ← Correct SYSVOL path
[dfsc] DfscCmInitState() - lookup found ref at ...                  ← Referral cache hit
```

Non-working scenario — No DFS Referral at all:
```
[dfsc] DfscRewritePath() - new path is \northwind.gov.com\sysvol\...\gpt.ini  ← NOT rewritten to DC FQDN!
[dfsc] DfscCmInitState() - lookup found negative cache entry        ← Negative cache
[dfsc] DfscCmInitState() - context → PATH_TYPE_NON_DFS             ← Treated as non-DFS path
```

**Conclusion**: The client was not sending any DFS Referral request, causing the path to remain as `\\domain\sysvol`

### Step 4: Discovered the DisableDfs Registry Key

**Action**: Investigated why DFS Referral was not being triggered

**Findings**: Found the following registry value on the customer's machine:

```
HKLM\SYSTEM\CurrentControlSet\services\MUP
    DisableDfs    REG_DWORD    0x1
```

**Conclusion**: `DisableDfs = 1` was disabling DFS functionality — this was the direct cause

### Step 5: Lab Reproduction

**Action**: Manually set `DisableDfs = 1` in the lab environment

**Findings**: Issue was successfully reproduced with identical symptoms:
- TGS request SPN became `cifs/contoso.lab` (domain name instead of DC FQDN)
- KDC returned `KDC_ERR_S_PRINCIPAL_UNKNOWN (7)`
- Tree Connect path was `\\contoso.lab\sysvol`
- DFSC logs showed `PATH_TYPE_NON_DFS`, no Referral request

### Step 6: Deep Dive — DFS Surrogate Failure Mechanism

**Action**: Discussed with EE and performed detailed comparison of MUP driver logs

**Findings**: The root difference was **DFS Surrogate call failure**.

In the SMB client architecture, DFS and CSC (Client-Side Caching) Surrogates are components of the MUP driver. They are called before I/O operations reach the UNC providers, acting as intermediaries.

**Non-working scenario** (DisableDfs = 1) — Surrogate fails:
```
[rxce]  RxInitializeContext() - IRP ... CREATE \Device\LanmanRedirector\contoso.lab\sysvol
[mup]   MupCallSurrogatePrePost() - IrpContext ...
[mup]   MupCallSurrogatePrePost() - Exit status 0xc0000016    ← Surrogate call FAILED!
[mup]   MupCreate() - FileName \contoso.lab\sysvol status 0xc0000201
```

**Working scenario** — Surrogate succeeds:
```
[rxce]  RxInitializeContext() - IRP ... CREATE \Device\LanmanRedirector\contoso.lab\sysvol\contoso.lab
[mup]   MupCallSurrogatePrePost() - Exit status 0x0           ← Surrogate succeeded
[mup]   MupCreate() - FileName \contoso.lab\sysvol\contoso.lab ← Proceeds normally
```

**Conclusion**: `DisableDfs = 1` causes the DFS Surrogate to return error `0xc0000016` at the `MupCallSurrogatePrePost()` stage, preventing all subsequent DFS Referral operations

### Step 7: How to Locate DFS Surrogate in SMB Client ETL

**Action**: Documented the methodology for analyzing DFS Surrogate behavior in SMB Client ETL

**Methodology**:

1. Filter with regex `RxInitializeContext.*Create` (this is the first I/O call)
2. Locate the entry containing the target path (e.g., `\contoso.lab\sysvol`)
3. Add `MupCreate()` filter, find the matching path entry
4. Add `MupCallSurrogatePrePost()` filter
5. Between `RxInitializeContext` and `MupCreate()`, the Surrogate status entries appear — **the first one is typically the DFS Surrogate**

> **Note**: Surrogates are only invoked for these IRP types:
> `IRP_MJ_CREATE`, `IRP_MJ_CREATE_MAILSLOT`, `IRP_MJ_CREATE_NAMED_PIPE`

## 4. Blockers & How They Were Resolved

| Blocker | Impact | Resolution |
|---------|--------|------------|
| Initial suspicion of DNS resolution issue (based on prior case experience) | Investigation direction was initially wrong | Verified KNR status in SMB Client ETL confirmed resolution was working, ruled out DNS |
| DFS/CSC Surrogates have no distinct name labels in ETL | Difficult to directly identify which Surrogate failed | Used the call chain sequence: `RxInitializeContext(CREATE)` → `MupCallSurrogatePrePost()` → `MupCreate()` — the first Surrogate is DFS |
| Understanding MUP driver internal Surrogate call mechanism | Required code-level analysis to confirm behavior | Discussed with EE; confirmed Surrogates are only invoked for `IRP_MJ_CREATE`-type operations |

## 5. Root Cause & Resolution

### Root Cause

The registry key `HKLM\SYSTEM\CurrentControlSet\services\MUP\DisableDfs = 1` was present on the customer's machine, disabling the MUP driver's DFS Surrogate functionality.

When DFS Surrogate is disabled:
1. `MupCallSurrogatePrePost()` returns error `0xc0000016`
2. DFS Referral request is never triggered (`DfscRmGetReferral()` is not called)
3. `DfscRewritePath()` cannot rewrite `\\domain\sysvol` to `\\DC_FQDN\sysvol`
4. SMB client uses `\\domain\sysvol` directly as the SMB path, with `cifs/domain` as Kerberos SPN
5. KDC cannot find the `cifs/domain` SPN, returns `KDC_ERR_S_PRINCIPAL_UNKNOWN (7)`
6. SMB falls back to NTLM authentication or connection fails entirely; SYSVOL becomes inaccessible

### Resolution

Remove the `DisableDfs` registry key from the client:

```powershell
# Check the registry value
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\services\MUP" -Name "DisableDfs"

# Remove the registry value
Remove-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\services\MUP" -Name "DisableDfs"

# Reboot to take effect
Restart-Computer
```

After removing the registry key, DFS Surrogate resumed normal operation and SYSVOL access was restored.

> **Note**: Do NOT attempt to work around this by manually adding the incorrect SPN (e.g., `cifs/domain`) to the DC. This is not the correct fix.

## 6. Lessons Learned

- **Technical Knowledge**:
  - `HKLM\SYSTEM\CurrentControlSet\services\MUP\DisableDfs` is a subtle but high-impact registry key — setting it to 1 completely disables DFS client functionality
  - SYSVOL relies on DFS Referral to resolve `\\domain\sysvol` to `\\DC_FQDN\sysvol`; disabling DFS directly breaks SYSVOL access
  - DFS Surrogate and CSC Surrogate are middleware layers in the MUP driver architecture, executing before I/O reaches UNC providers
  - Surrogates are only invoked for `IRP_MJ_CREATE`, `IRP_MJ_CREATE_MAILSLOT`, and `IRP_MJ_CREATE_NAMED_PIPE` IRP types

- **Troubleshooting Methodology**:
  - When `cifs/<domain>` SPN appears in Kerberos TGS (instead of `cifs/<DC_FQDN>`), it indicates DFS Referral is not functioning
  - When investigating DFS Referral failure, check the `DisableDfs` registry key in addition to DNS resolution
  - To locate DFS Surrogate in SMB Client ETL: trace the `RxInitializeContext(CREATE)` → `MupCallSurrogatePrePost()` → `MupCreate()` call chain; Surrogate status `0x0` = success, `0xc0000016` = failure
  - Comparing working vs non-working ETL logs side-by-side is an effective approach for pinpointing the issue

- **Prevention**:
  - Deployment scripts and security hardening policies should verify the `DisableDfs` registry key is not inadvertently set
  - For SYSVOL access failures, `DisableDfs` should be an early checklist item

## 7. References

- [DFS Namespaces Overview](https://learn.microsoft.com/en-us/windows-server/storage/dfs-namespaces/dfs-overview) — DFS Namespaces overview explaining DFS Referral mechanism
- [Enable or Disable Referrals and Client Failback](https://learn.microsoft.com/en-us/windows-server/storage/dfs-namespaces/enable-or-disable-referrals-and-client-failback) — DFS Referral and client failback configuration
- [SMB File Server Overview](https://learn.microsoft.com/en-us/windows-server/storage/file-server/file-server-smb-overview) — SMB protocol overview
- [Applying Group Policy Troubleshooting Guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/group-policy/applying-group-policy-troubleshooting-guidance) — Group Policy application troubleshooting guide
