---
layout: post
title: "Deep Dive: Certificate Strong Mapping 与 Mac 802.1X 认证失败排查"
date: 2026-03-17
categories: [Knowledge, Authentication]
tags: [certificate, strong-mapping, kerberos, 802.1x, nps, eap-tls, adcs, mac, mdm, altsecurityidentities, schannel]
type: "deep-dive"
---

# Deep Dive: Certificate Strong Mapping 与 Mac 802.1X 认证失败排查

**Topic:** Kerberos Certificate Strong Mapping / 802.1X EAP-TLS Authentication for Mac  
**Category:** Authentication / PKI / Networking  
**Level:** 高级 / Advanced  
**Last Updated:** 2026-03-17

---

## 1. 概述 (Overview)

自 2022 年 5 月微软发布 KB5014754 安全更新以来，Windows 域控制器（KDC）对基于证书的身份验证实施了更严格的映射检查——即 **Strong Certificate Mapping（强证书映射）**。该变更旨在消除 CVE-2022-34691、CVE-2022-26931 和 CVE-2022-26923 带来的证书仿冒提权漏洞。

在传统的 Windows 域环境中，证书映射到 AD 用户通常依赖 UPN（User Principal Name）或其他弱映射方式。但在 Strong Mapping 强制模式下，KDC 要求证书中必须包含可直接关联到 AD 对象的强标识（如 SID），否则认证将被拒绝。

对于 **Mac 设备通过第三方 MDM 向 Windows CA 申请证书、再通过 802.1X EAP-TLS 接入无线网络** 的场景，这一变更影响尤为显著——因为第三方 MDM 签发的证书通常不包含 AD SID，导致 Kerberos 强映射校验失败，WiFi 连接被拒。

---

## 2. 核心概念 (Core Concepts)

### 2.1 Certificate Strong Mapping（强证书映射）

- **定义**：KDC 在验证证书时，要求证书中包含能唯一且可靠地映射到 AD 对象的标识符
- **背景**：KB5014754 引入，修复证书仿冒漏洞
- **强制时间线**：
  - 2022-05-10：引入 Compatibility 模式（审计 + 警告）
  - 2025-02-11：进入 Enforcement 模式（弱映射仍可回退兼容）
  - 2025-09-09：**完全强制**，注册表回退选项移除

### 2.2 强映射 vs 弱映射

| 映射类型 | 映射方式 | 安全等级 | 示例 |
|---------|---------|---------|------|
| **强映射 (Strong)** | 基于 SID（OID 1.3.6.1.4.1.311.25.2 或 SAN URI） | ✅ 高 | 证书中嵌入 `tag:microsoft.com,2022-09-14:sid:<SID>` |
| **强映射 (Strong)** | Issuer + SerialNumber | ✅ 高 | `X509:<I>DC=local,DC=contoso,CN=CA<SR>SerialNum` |
| **强映射 (Strong)** | SKI (Subject Key Identifier) | ✅ 高 | `X509:<SKI>SubjectKeyIdentifier` |
| **强映射 (Strong)** | SHA1 Public Key Hash | ✅ 高 | `X509:<SHA1-PUKEY>PublicKeyHash` |
| **弱映射 (Weak)** | 基于 UPN | ⚠️ 低 | 证书 SAN 中的 UPN 匹配 AD userPrincipalName |
| **弱映射 (Weak)** | 基于 Email | ⚠️ 低 | 证书中的 Email 匹配 AD mail 属性 |

> **类比理解**：弱映射就像"凭名字进门"——别人也可能同名冒充；强映射就像"凭身份证号进门"——每人独一无二，无法伪造。

### 2.3 证书模板：Supply in the Request vs Build from AD Information

- **Build from AD Information**：CA 签发证书时从 AD 对象自动提取 UPN、SID 等信息写入证书。Windows 域成员通过 MMC 或 autoenrollment 使用此模板时，证书天然满足强映射
- **Supply in the Request**：请求者自行提供 Subject/SAN 信息。第三方 MDM（如 Jamf）通过 SCEP 代理申请证书时通常使用此模式，CA 不会自动注入 SID

### 2.4 altSecurityIdentities 属性

- AD 用户/计算机对象上的多值属性
- 用于建立 **证书 ↔ AD 对象** 的显式映射
- 当证书本身不包含 SID 时，这是实现强映射的"补救"方案
- 格式示例：
  - `X509:<I>DC=local,DC=contoso,CN=ContosoCA<SR>0123456789abcdef`（推荐）
  - `X509:<SKI>abcdef0123456789`
  - `X509:<SHA1-PUKEY>0123456789abcdef`

### 2.5 802.1X EAP-TLS 认证流程（NPS 作为 RADIUS）

```
Mac Client ──Wi-Fi──> AP (Authenticator) ──RADIUS──> NPS Server ──Kerberos/LDAP──> Domain Controller
                                                        │
                                                   EAP-TLS handshake
                                                   Certificate validation
                                                   Certificate → AD mapping
```

---

## 3. 工作原理 (How It Works)

### 3.1 整体架构

```
┌──────────────┐     802.1X      ┌───────────┐    RADIUS     ┌─────────────┐   Kerberos    ┌──────────┐
│  Mac Client  │ ──EAP-TLS───>  │  Wireless  │ ──────────>  │  NPS Server │ ──S4U2Self──> │    DC     │
│  (via MDM    │                 │    AP      │              │  (RADIUS)   │              │  (KDC)   │
│  cert)       │                 │            │              │             │              │          │
└──────────────┘                 └───────────┘              └─────────────┘              └──────────┘
                                                                  │
                                                           Schannel TLS
                                                           cert mapping
                                                           ↓
                                                     ┌─────────────────┐
                                                     │ AD CS (CA Server)│
                                                     │ Certificate      │
                                                     │ Trust Chain      │
                                                     └─────────────────┘
```

### 3.2 详细认证流程

1. **Step 1: EAP-TLS 握手**
   - Mac 客户端连接 WiFi，AP 触发 802.1X 认证
   - NPS 服务器与客户端进行 EAP-TLS 握手
   - 客户端呈交 MDM 下发的用户证书

2. **Step 2: 证书链验证（Schannel/CAPI2）**
   - NPS 上的 Schannel 验证证书链信任（Root CA → Intermediate CA → User Cert）
   - 检查证书是否在吊销列表中（CRL/OCSP）
   - ✅ 此步骤通常通过（证书信任链正常）

3. **Step 3: 证书到 AD 用户的映射（关键步骤）**
   - Schannel 调用 `SslTryS4U2Self()` 尝试通过证书中的 UPN 做 S4U2Self 映射
   - NPS 将映射请求发送到 DC：`SslMapCertAtDC()`
   - DC（KDC）检查证书是否满足强映射条件：
     - 是否包含 SID（OID 扩展或 SAN URI）？
     - 如果没有 SID，检查 `altSecurityIdentities` 是否有显式映射？
     - 如果都没有，且处于 Enforcement 模式 → **映射失败**

4. **Step 4: 认证结果**
   - 映射成功：NPS 返回 RADIUS Access-Accept，WiFi 连接成功
   - 映射失败：KDC 返回 `KDC_ERR_CERTIFICATE_MISMATCH (66)`，Schannel 返回 `0xc000006d (STATUS_LOGON_FAILURE)` 和 `0x8009030b (SEC_E_NO_CREDENTIALS)`，WiFi 连接失败

### 3.3 关键失败机制分析

当 Mac 通过第三方 MDM（如 Jamf）使用 **Supply in the Request** 模板申请证书时：

```
证书内容:
  Subject: CN=john.smith
  SAN: UPN=john.smith@contoso.com
  ❌ 没有 SID OID 扩展
  ❌ 没有 SAN URI (tag:microsoft.com,2022-09-14:sid:xxx)

KDC 校验:
  1. 检查 SID OID → 未找到
  2. 检查 SAN URI SID → 未找到
  3. 检查 altSecurityIdentities → 未配置
  4. 回退到 UPN 弱映射 → Enforcement 模式下拒绝
  5. 返回 KDC_ERR_CERTIFICATE_MISMATCH (66)
```

对比使用 **Build from AD Information** 模板时：

```
证书内容:
  Subject: CN=john.smith
  SAN: UPN=john.smith@contoso.com
  ✅ SID OID 扩展: 1.3.6.1.4.1.311.25.2 = S-1-5-21-xxx-xxx-xxx-1234
  (或 SAN URI: tag:microsoft.com,2022-09-14:sid:S-1-5-21-xxx-xxx-xxx-1234)

KDC 校验:
  1. 检查 SID OID → ✅ 找到，匹配 AD 用户 SID
  2. 强映射成功 → 认证通过
```

---

## 4. 关键配置与参数 (Key Configurations)

| 配置项/参数 | 默认值 | 说明 | 常见调优场景 |
|------------|--------|------|------------|
| `StrongCertificateBindingEnforcement` (HKLM\SYSTEM\CurrentControlSet\Services\Kdc) | 1 (Compatibility) → 2 (Enforcement) | 0=禁用, 1=兼容, 2=强制 | 2025-09-09 后无法再设为 0 或 1 |
| `CertificateMappingMethods` (Schannel 下) | 0x1F | 控制 Schannel 证书映射方法 | 排查时可临时调整 |
| 证书模板 Subject Name 设置 | 取决于模板 | "Build from AD Information" vs "Supply in the Request" | Mac/MDM 场景需特别注意 |
| `altSecurityIdentities` (AD 用户属性) | 空 | 显式证书-用户映射 | 第三方 MDM 证书无 SID 时必须配置 |
| NPS Network Policy - Auth Method | EAP-TLS | 802.1X 认证方法 | 确保选对了 EAP 类型和证书 |
| Schannel EventLogging | 1 | 7=详细日志 | 排查时临时设为 7 |

### 4.1 altSecurityIdentities 格式详解

```
# 推荐格式 1: Issuer + Serial Number（最常用）
X509:<I>DC=local,DC=contoso,CN=ContosoCA<SR>0a1b2c3d4e5f

# 格式 2: Subject Key Identifier
X509:<SKI>abcdef0123456789abcdef

# 格式 3: SHA1 Public Key Hash
X509:<SHA1-PUKEY>0123456789abcdef0123456789abcdef01234567

# 格式 4: Issuer + Subject (弱映射，不推荐)
X509:<I>DC=local,DC=contoso,CN=ContosoCA<S>CN=john.smith
```

**PowerShell 批量设置示例**：
```powershell
# 获取证书序列号（从导出的 .cer 文件）
$cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2("C:\certs\user.cer")
$serial = $cert.SerialNumber
$issuer = $cert.Issuer

# 设置 altSecurityIdentities
Set-ADUser -Identity "john.smith" -Add @{
    altSecurityIdentities = "X509:<I>$issuer<SR>$serial"
}
```

---

## 5. 常见问题与排查 (Common Issues & Troubleshooting)

### 问题 A: KDC_ERR_CERTIFICATE_MISMATCH (66)

- **症状**：Mac 通过 802.1X WiFi 认证失败，NPS 日志显示 `KRB_ERROR - KDC_ERR_CERTIFICATE_MISMATCH (66)`
- **可能原因**：
  1. 证书不包含 AD SID（第三方 MDM 使用 Supply in the Request 模板）
  2. `altSecurityIdentities` 未配置
  3. KDC 处于 Enforcement 模式
- **排查思路**：
  1. 检查证书内容：`certutil -dump user.cer`，确认是否有 SID OID (1.3.6.1.4.1.311.25.2) 或 SAN URI
  2. 检查 AD 用户的 `altSecurityIdentities`：`Get-ADUser -Identity john.smith -Properties altSecurityIdentities`
  3. 检查 KDC 模式：`Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Kdc" -Name StrongCertificateBindingEnforcement`
  4. DC 上查看 Event ID 39/40/41（KDC 证书映射审计事件）
- **关键日志证据**：
  ```
  SslTryS4U2Self() - Looking for UPN name john.smith@contoso.com
  SslMapCertAtDC() - Remote call to DC to do the mapping
  SslMapCertAtDC() - SslMapCertAtDC returned sub-status 0xc000006d
  SslRemoteMapCredential() - CertMapping: SslMapCertAtDC Failed with 0xc000006d
  ```

### 问题 B: SEC_E_NO_CREDENTIALS (0x8009030b)

- **症状**：EAP-TLS 认证日志显示 `QuerySecurityContextToken failed and returned 0x8009030b`
- **可能原因**：Schannel 在尝试将 TLS 会话映射到 Windows 用户安全上下文时失败，"证书 → AD 账户"的映射失败
- **排查思路**：
  1. 确认这是证书映射问题而非证书信任问题
  2. 检查 CAPI2 日志确认证书链是否正常（通常是正常的）
  3. 聚焦排查强映射问题（同问题 A）
- **关键日志证据**：
  ```
  EapTlsMakeMessage(contoso\john.smith)
  CheckUserName() - QuerySecurityContextToken failed and returned 0x8009030b
  MakeAlert(49, Manual)
  MakeReplyMessage() - State change to SentFinished. Error: 0x8009030b
  ```

### 问题 C: Mac/MDM 无法使用 Build from AD Information 模板

- **症状**：使用 Build from AD Information 模板时，Mac 通过 MDM 申请到的证书 Subject 是 MDM 代理平台的计算机 FQDN，而非用户信息
- **原因**：Mac 非域成员，MDM 代理是以"机器身份"向 CA 提交请求，CA 从 AD 拉取的是代理机器的信息
- **解决方案**：
  1. 改用 altSecurityIdentities 显式映射
  2. 咨询 MDM 厂商是否支持 SID 注入（Jamf 自 2025 年起支持）
  3. 使用 Intune 替代（原生支持 SID 注入到 SCEP 证书）

---

## 6. 实战经验 (Practical Tips)

### 最佳实践

- **优先从源头解决**：尽量在证书签发阶段即完成 SID 注入，而非事后在 AD 补充映射
- **Intune 原生支持**：Microsoft Intune 已通过 SCEP profile 中的 `OnPremisesSecurityIdentifier` 变量在 SAN 中注入用户 SID，覆盖 Windows、iOS、macOS、Android
- **Jamf 支持**：Jamf 自 2025 年起也更新支持了 Active Directory Strong Certificate Mapping Requirements
- **自动化 altSecurityIdentities**：如果必须用显式映射，建议通过脚本或自动化流程批量维护，并纳入用户新增/证书更新的标准流程

### 常见误区

- ❌ **误区**：UPN 匹配就够了
  - ✅ **事实**：Enforcement 模式下 UPN 属于弱映射，会被拒绝
- ❌ **误区**：证书信任链没问题 = 认证会成功
  - ✅ **事实**：信任链验证和证书-用户映射是两个独立步骤，信任链通过后映射仍可能失败
- ❌ **误区**：配一次 altSecurityIdentities 就永久有效
  - ✅ **事实**：证书更新/重签后 SerialNumber 会变，需要更新映射（推荐将更新纳入流程自动化）

### 性能考量

- altSecurityIdentities 是在每次认证时查询 AD 的，大量用户时确保 DC 性能充足
- NPS 到 DC 的 `SslMapCertAtDC()` 是远程调用，网络延迟会影响认证时间

### 安全注意

- **不要禁用强映射**：`StrongCertificateBindingEnforcement=0` 会暴露系统于 CVE-2022-26923 等漏洞
- **altSecurityIdentities 权限**：修改此属性需要 Domain Admin 或被委派的写入权限
- **证书私钥保护**：通过 MDM 下发的证书私钥应标记为不可导出

---

## 7. 与相关技术的对比 (Comparison with Related Technologies)

### 7.1 不同 MDM 平台的强映射支持

| 维度 | Microsoft Intune | Jamf Pro | 其他第三方 MDM |
|------|-----------------|----------|-------------|
| SID 注入支持 | ✅ 原生支持（OnPremisesSecurityIdentifier 变量） | ✅ 2025 年起支持 | ⚠️ 需逐个确认 |
| 证书协议 | SCEP / PKCS | SCEP (AD CS connector) | SCEP（通常） |
| 覆盖平台 | Windows, iOS, macOS, Android | macOS, iOS | 取决于厂商 |
| 无需 altSecurityIdentities | ✅ | ✅（配置正确时） | ❌ 通常需要 |

### 7.2 强映射实现方式对比

| 方式 | 自动化程度 | 维护成本 | 适用场景 |
|------|-----------|---------|---------|
| Build from AD Information 模板 + SID OID | 全自动 | 零 | Windows 域成员 autoenrollment |
| MDM SID 注入（Intune/Jamf） | 全自动 | 低 | MDM 管理的跨平台设备 |
| altSecurityIdentities 显式映射 | 半自动（需脚本） | 中-高 | 第三方 MDM 不支持 SID 注入时 |
| SAN URI 手动请求 | 手动 | 高 | 特殊场景、测试 |

---

## 8. 参考资料 (References)

- [KB5014754: Certificate-based authentication changes on Windows domain controllers](https://support.microsoft.com/en-us/topic/kb5014754-certificate-based-authentication-changes-on-windows-domain-controllers-ad2c23b0-15d8-4340-a468-4d4f3b188f16) — 微软官方 KB，详细描述强映射变更时间线、注册表配置和故障排查
- [Support Tip: Implementing strong mapping in Microsoft Intune certificates](https://techcommunity.microsoft.com/blog/intunecustomersuccess/support-tip-implementing-strong-mapping-in-microsoft-intune-certificates/4053376) — Intune 通过 SCEP 实现 SID 注入的完整指南
- [Preview of SAN URI for Certificate Strong Mapping for KB5014754](https://techcommunity.microsoft.com/t5/ask-the-directory-services-team/preview-of-san-uri-for-certificate-strong-mapping-for-kb5014754/ba-p/3789785) — SAN URI 格式的强映射实现方式及示例 INF 文件

---

## NPS 日志采集脚本参考 (NPS Trace Collection Script)

用于 NPS 服务器上采集 802.1X 认证相关的完整诊断日志：

```batch
@echo off
md C:\NPSTrace
md C:\NPSTrace\EventLogs

REM === NPS / RADIUS 相关 ETL 追踪 ===
logman create trace "net_nps" -ow -o C:\NPSTrace\net_nps.etl -p {B2CBF6DC-392A-43AE-98D2-1AA66DFCB2C3} 0xffffffffffffffff 0xff -nb 16 16 -bs 1024 -mode Circular -f bincirc -max 512 -ets
logman update trace "net_nps" -p {344EB4B5-5B35-48D7-B116-28E3E3976B49} 0xffffffffffffffff 0xff -ets
logman update trace "net_nps" -p {997590EF-d144-4d41-b7fb-7028ae295b04} 0xffffffffffffffff 0xff -ets

REM === EAPHost ETL 追踪 ===
logman create trace "net_eaphost" -ow -o C:\NPSTrace\net_eaphost.etl -p {40DAC86F-DD09-4B47-A112-3AE54DA2222E} 0xffffffffffffffff 0xff -nb 16 16 -bs 1024 -mode Circular -f bincirc -max 4096 -ets
logman update trace "net_eaphost" -p "Microsoft-Windows-EapHost" 0xffffffffffffffff 0xff -ets

REM === Schannel / CAPI2 / Security ETL ===
logman create trace "ds_security" -ow -o C:\NPSTrace\ds_security.etl -p {37D2C3CD-C5D4-4587-8531-4696C44244C8} 0xffffffffffffffff 0xff -nb 16 16 -bs 1024 -mode Circular -f bincirc -max 4096 -ets
logman update trace "ds_security" -p "Schannel" 0xffffffffffffffff 0xff -ets
logman update trace "ds_security" -p "Microsoft-Windows-CAPI2" 0xffffffffffffffff 0xff -ets

REG ADD "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL" /f /v EventLogging /t REG_DWORD /d 7
wevtutil.exe set-log Microsoft-Windows-CAPI2/Operational /enabled:true

REM === 网络抓包 ===
netsh trace start capture=yes tracefile=C:\NPSTrace\nettrace.etl maxsize=512 overwrite=yes report=no correlation=no

echo === 日志采集已启动，请复现问题后按任意键停止 ===
pause

REM === 停止所有追踪 ===
logman stop "net_nps" -ets
logman stop "net_eaphost" -ets
logman stop "ds_security" -ets
netsh trace stop

REG ADD "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL" /f /v EventLogging /t REG_DWORD /d 1
wevtutil.exe set-log Microsoft-Windows-CAPI2/Operational /enabled:false
wevtutil.exe export-log Microsoft-Windows-CAPI2/Operational C:\NPSTrace\capi2.evtx /overwrite:true

echo === 日志已保存到 C:\NPSTrace ===
```

---

---

# English Version

---

# Deep Dive: Certificate Strong Mapping & Mac 802.1X Authentication Failure Troubleshooting

**Topic:** Kerberos Certificate Strong Mapping / 802.1X EAP-TLS Authentication for Mac  
**Category:** Authentication / PKI / Networking  
**Level:** Advanced  
**Last Updated:** 2026-03-17

---

## 1. Overview

Since Microsoft released the KB5014754 security update in May 2022, Windows Domain Controllers (KDC) enforce stricter validation for certificate-based authentication — known as **Strong Certificate Mapping**. This change addresses elevation of privilege vulnerabilities in CVE-2022-34691, CVE-2022-26931, and CVE-2022-26923 related to certificate spoofing.

In traditional Windows domain environments, certificate-to-AD-user mapping often relied on UPN (User Principal Name) or other weak mapping methods. Under Strong Mapping enforcement, the KDC requires certificates to contain a strong identifier (such as the user/device SID) that directly ties to an AD object. Otherwise, authentication is denied.

This is particularly impactful for **Mac devices using third-party MDM to request certificates from Windows CA, then connecting via 802.1X EAP-TLS wireless authentication** — because third-party MDM-issued certificates typically lack the AD SID, causing Kerberos strong mapping validation to fail and WiFi connection to be rejected.

---

## 2. Core Concepts

### 2.1 Certificate Strong Mapping

- **Definition**: The KDC requires that certificates contain an identifier that can uniquely and reliably map to an AD object during authentication
- **Background**: Introduced via KB5014754 to fix certificate spoofing vulnerabilities
- **Enforcement Timeline**:
  - 2022-05-10: Compatibility mode introduced (audit + warning)
  - 2025-02-11: Enforcement mode (weak mapping fallback still available via registry)
  - 2025-09-09: **Full enforcement** — registry fallback option removed entirely

### 2.2 Strong Mapping vs Weak Mapping

| Mapping Type | Method | Security Level | Example |
|-------------|--------|---------------|---------|
| **Strong** | SID-based (OID 1.3.6.1.4.1.311.25.2 or SAN URI) | ✅ High | `tag:microsoft.com,2022-09-14:sid:<SID>` embedded in cert |
| **Strong** | Issuer + SerialNumber | ✅ High | `X509:<I>DC=local,DC=contoso,CN=CA<SR>SerialNum` |
| **Strong** | SKI (Subject Key Identifier) | ✅ High | `X509:<SKI>SubjectKeyIdentifier` |
| **Strong** | SHA1 Public Key Hash | ✅ High | `X509:<SHA1-PUKEY>PublicKeyHash` |
| **Weak** | UPN-based | ⚠️ Low | Certificate SAN UPN matches AD userPrincipalName |
| **Weak** | Email-based | ⚠️ Low | Certificate email matches AD mail attribute |

> **Analogy**: Weak mapping is like "entering by stating your name" — someone else could claim the same name. Strong mapping is like "entering by showing your government ID number" — unique and unforgeable.

### 2.3 Certificate Templates: Supply in the Request vs Build from AD Information

- **Build from AD Information**: The CA automatically pulls UPN, SID, and other attributes from the AD object when issuing the certificate. Certificates issued to domain-joined Windows machines via autoenrollment inherently satisfy strong mapping
- **Supply in the Request**: The requestor provides Subject/SAN information. Third-party MDMs (e.g., Jamf) using SCEP typically use this mode, and the CA does not automatically inject the SID

### 2.4 altSecurityIdentities Attribute

- A multi-valued attribute on AD user/computer objects
- Establishes an **explicit certificate ↔ AD object mapping**
- The "remediation" approach when certificates don't contain a SID
- Format examples:
  - `X509:<I>DC=local,DC=contoso,CN=ContosoCA<SR>0123456789abcdef` (recommended)
  - `X509:<SKI>abcdef0123456789`
  - `X509:<SHA1-PUKEY>0123456789abcdef`

### 2.5 802.1X EAP-TLS Authentication Flow (NPS as RADIUS)

```
Mac Client ──Wi-Fi──> AP (Authenticator) ──RADIUS──> NPS Server ──Kerberos/LDAP──> Domain Controller
                                                        │
                                                   EAP-TLS handshake
                                                   Certificate validation
                                                   Certificate → AD mapping
```

---

## 3. How It Works

### 3.1 Architecture Overview

```
┌──────────────┐     802.1X      ┌───────────┐    RADIUS     ┌─────────────┐   Kerberos    ┌──────────┐
│  Mac Client  │ ──EAP-TLS───>  │  Wireless  │ ──────────>  │  NPS Server │ ──S4U2Self──> │    DC     │
│  (via MDM    │                 │    AP      │              │  (RADIUS)   │              │  (KDC)   │
│  cert)       │                 │            │              │             │              │          │
└──────────────┘                 └───────────┘              └─────────────┘              └──────────┘
                                                                  │
                                                           Schannel TLS
                                                           cert mapping
                                                           ↓
                                                     ┌─────────────────┐
                                                     │ AD CS (CA Server)│
                                                     │ Certificate      │
                                                     │ Trust Chain      │
                                                     └─────────────────┘
```

### 3.2 Detailed Authentication Flow

1. **Step 1: EAP-TLS Handshake**
   - Mac client connects to WiFi; the AP triggers 802.1X authentication
   - NPS server performs EAP-TLS handshake with the client
   - Client presents the user certificate deployed by MDM

2. **Step 2: Certificate Chain Validation (Schannel/CAPI2)**
   - Schannel on NPS validates the certificate trust chain (Root CA → Intermediate CA → User Cert)
   - Checks certificate revocation status (CRL/OCSP)
   - ✅ This step usually passes (trust chain is healthy)

3. **Step 3: Certificate-to-AD-User Mapping (Critical Step)**
   - Schannel calls `SslTryS4U2Self()` to attempt S4U2Self mapping using the UPN from the certificate
   - NPS sends the mapping request to the DC: `SslMapCertAtDC()`
   - The DC (KDC) checks whether the certificate satisfies strong mapping:
     - Does it contain a SID (OID extension or SAN URI)?
     - If not, does `altSecurityIdentities` have an explicit mapping?
     - If neither, and Enforcement mode is active → **Mapping fails**

4. **Step 4: Authentication Result**
   - Mapping succeeds: NPS returns RADIUS Access-Accept; WiFi connects
   - Mapping fails: KDC returns `KDC_ERR_CERTIFICATE_MISMATCH (66)`, Schannel returns `0xc000006d (STATUS_LOGON_FAILURE)` and `0x8009030b (SEC_E_NO_CREDENTIALS)`; WiFi connection fails

### 3.3 Failure Mechanism Analysis

When a Mac uses a third-party MDM (e.g., Jamf) with a **Supply in the Request** template:

```
Certificate contents:
  Subject: CN=john.smith
  SAN: UPN=john.smith@contoso.com
  ❌ No SID OID extension
  ❌ No SAN URI (tag:microsoft.com,2022-09-14:sid:xxx)

KDC validation:
  1. Check SID OID → Not found
  2. Check SAN URI SID → Not found
  3. Check altSecurityIdentities → Not configured
  4. Fallback to UPN weak mapping → Rejected in Enforcement mode
  5. Return KDC_ERR_CERTIFICATE_MISMATCH (66)
```

Versus using a **Build from AD Information** template:

```
Certificate contents:
  Subject: CN=john.smith
  SAN: UPN=john.smith@contoso.com
  ✅ SID OID extension: 1.3.6.1.4.1.311.25.2 = S-1-5-21-xxx-xxx-xxx-1234
  (or SAN URI: tag:microsoft.com,2022-09-14:sid:S-1-5-21-xxx-xxx-xxx-1234)

KDC validation:
  1. Check SID OID → ✅ Found, matches AD user SID
  2. Strong mapping succeeds → Authentication passes
```

---

## 4. Key Configurations

| Configuration | Default | Description | Common Tuning Scenario |
|--------------|---------|-------------|----------------------|
| `StrongCertificateBindingEnforcement` (HKLM\...\Kdc) | 1→2 | 0=Disabled, 1=Compatibility, 2=Enforcement | Cannot be set to 0 or 1 after 2025-09-09 |
| `CertificateMappingMethods` (under Schannel) | 0x1F | Controls Schannel certificate mapping methods | Temporary adjustment during troubleshooting |
| Certificate Template Subject Name | Depends | "Build from AD Information" vs "Supply in the Request" | Critical for Mac/MDM scenarios |
| `altSecurityIdentities` (AD user attribute) | Empty | Explicit certificate-to-user mapping | Required when third-party MDM certs lack SID |
| Schannel EventLogging | 1 | 7=Verbose logging | Set to 7 temporarily during investigation |

### 4.1 altSecurityIdentities Format Reference

```
# Recommended: Issuer + Serial Number (most common)
X509:<I>DC=local,DC=contoso,CN=ContosoCA<SR>0a1b2c3d4e5f

# Subject Key Identifier
X509:<SKI>abcdef0123456789abcdef

# SHA1 Public Key Hash
X509:<SHA1-PUKEY>0123456789abcdef0123456789abcdef01234567
```

**PowerShell Bulk Configuration Example**:
```powershell
# Get certificate serial number from exported .cer file
$cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2("C:\certs\user.cer")
$serial = $cert.SerialNumber
$issuer = $cert.Issuer

# Set altSecurityIdentities
Set-ADUser -Identity "john.smith" -Add @{
    altSecurityIdentities = "X509:<I>$issuer<SR>$serial"
}
```

---

## 5. Common Issues & Troubleshooting

### Issue A: KDC_ERR_CERTIFICATE_MISMATCH (66)

- **Symptom**: Mac fails 802.1X WiFi authentication; NPS logs show `KRB_ERROR - KDC_ERR_CERTIFICATE_MISMATCH (66)`
- **Possible Causes**:
  1. Certificate lacks AD SID (third-party MDM with Supply in the Request template)
  2. `altSecurityIdentities` not configured
  3. KDC in Enforcement mode
- **Troubleshooting Approach**:
  1. Inspect certificate: `certutil -dump user.cer` — check for SID OID (1.3.6.1.4.1.311.25.2) or SAN URI
  2. Check AD user's `altSecurityIdentities`: `Get-ADUser -Identity john.smith -Properties altSecurityIdentities`
  3. Check KDC mode: `Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Kdc" -Name StrongCertificateBindingEnforcement`
  4. Review Event ID 39/40/41 on the DC (KDC certificate mapping audit events)
- **Key Log Evidence**:
  ```
  SslTryS4U2Self() - Looking for UPN name john.smith@contoso.com
  SslMapCertAtDC() - Remote call to DC to do the mapping
  SslMapCertAtDC() - SslMapCertAtDC returned sub-status 0xc000006d
  SslRemoteMapCredential() - CertMapping: SslMapCertAtDC Failed with 0xc000006d
  ```

### Issue B: SEC_E_NO_CREDENTIALS (0x8009030b)

- **Symptom**: EAP-TLS authentication logs show `QuerySecurityContextToken failed and returned 0x8009030b`
- **Cause**: Schannel failed to map the TLS session to a Windows user security context — the "certificate → AD account" mapping failed
- **Key Log Evidence**:
  ```
  EapTlsMakeMessage(contoso\john.smith)
  CheckUserName() - QuerySecurityContextToken failed and returned 0x8009030b
  MakeAlert(49, Manual)
  MakeReplyMessage() - State change to SentFinished. Error: 0x8009030b
  ```

### Issue C: Mac/MDM Cannot Use Build from AD Information Template

- **Symptom**: When using Build from AD Information, Mac certificates requested via MDM contain the MDM proxy machine's FQDN instead of user information
- **Cause**: Mac is not domain-joined; MDM agent submits requests as its machine identity; CA pulls the agent machine's info from AD
- **Solutions**:
  1. Use altSecurityIdentities explicit mapping
  2. Check if MDM vendor supports SID injection (Jamf supports this since 2025)
  3. Use Microsoft Intune (native SID injection into SCEP certificates)

---

## 6. Practical Tips

### Best Practices

- **Fix at the source**: Inject the SID during certificate issuance rather than patching in AD after the fact
- **Intune native support**: Microsoft Intune supports SID injection via the `OnPremisesSecurityIdentifier` variable in SCEP profiles, covering Windows, iOS, macOS, and Android
- **Jamf support**: Jamf added support for Active Directory Strong Certificate Mapping Requirements starting in 2025
- **Automate altSecurityIdentities**: If explicit mapping is unavoidable, use scripts or automation to bulk-maintain, and incorporate into user onboarding / cert renewal workflows

### Common Misconceptions

- ❌ **Myth**: UPN match is sufficient
  - ✅ **Fact**: UPN-based mapping is classified as weak and will be rejected under Enforcement mode
- ❌ **Myth**: Trust chain validation passing means authentication will succeed
  - ✅ **Fact**: Trust chain validation and certificate-user mapping are independent steps; the former passing does not guarantee the latter
- ❌ **Myth**: Setting altSecurityIdentities once is permanent
  - ✅ **Fact**: When a certificate is renewed/reissued, the serial number changes, requiring the mapping to be updated

### Performance Considerations

- altSecurityIdentities lookups occur on every authentication against AD; ensure DC capacity for large user bases
- The `SslMapCertAtDC()` call from NPS to DC is a remote call; network latency impacts authentication time

### Security Notes

- **Never disable strong mapping**: `StrongCertificateBindingEnforcement=0` exposes your environment to CVE-2022-26923 and similar vulnerabilities
- **altSecurityIdentities permissions**: Modifying this attribute requires Domain Admin privileges or delegated write access
- **Private key protection**: Certificates deployed via MDM should have private keys marked as non-exportable

---

## 7. Comparison with Related Technologies

### 7.1 Strong Mapping Support Across MDM Platforms

| Dimension | Microsoft Intune | Jamf Pro | Other Third-Party MDM |
|-----------|-----------------|----------|----------------------|
| SID Injection | ✅ Native (OnPremisesSecurityIdentifier variable) | ✅ Since 2025 | ⚠️ Varies; verify individually |
| Certificate Protocol | SCEP / PKCS | SCEP (AD CS connector) | SCEP (typically) |
| Platform Coverage | Windows, iOS, macOS, Android | macOS, iOS | Vendor-dependent |
| altSecurityIdentities needed | ❌ Not required | ❌ Not required (if configured correctly) | ✅ Usually required |

### 7.2 Strong Mapping Implementation Approaches

| Approach | Automation | Maintenance Cost | Applicable Scenario |
|----------|-----------|-----------------|-------------------|
| Build from AD Information + SID OID | Fully automatic | None | Windows domain-joined autoenrollment |
| MDM SID Injection (Intune/Jamf) | Fully automatic | Low | MDM-managed cross-platform devices |
| altSecurityIdentities explicit mapping | Semi-auto (scripted) | Medium-High | Third-party MDM without SID injection |
| SAN URI manual request | Manual | High | Special scenarios, testing |

---

## 8. References

- [KB5014754: Certificate-based authentication changes on Windows domain controllers](https://support.microsoft.com/en-us/topic/kb5014754-certificate-based-authentication-changes-on-windows-domain-controllers-ad2c23b0-15d8-4340-a468-4d4f3b188f16) — Official Microsoft KB detailing strong mapping changes, timeline, registry configurations, and troubleshooting
- [Support Tip: Implementing strong mapping in Microsoft Intune certificates](https://techcommunity.microsoft.com/blog/intunecustomersuccess/support-tip-implementing-strong-mapping-in-microsoft-intune-certificates/4053376) — Complete guide for Intune SCEP SID injection implementation
- [Preview of SAN URI for Certificate Strong Mapping for KB5014754](https://techcommunity.microsoft.com/t5/ask-the-directory-services-team/preview-of-san-uri-for-certificate-strong-mapping-for-kb5014754/ba-p/3789785) — SAN URI-based strong mapping implementation and example INF files
