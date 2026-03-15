---
layout: post
title: "Deep Dive: Windows DNS Server — Architecture, Operations & Troubleshooting"
date: 2026-03-12
categories: [Knowledge, Networking]
tags: [dns, windows-dns, active-directory, dnssec, dns-policy, dns-client, name-resolution, troubleshooting]
type: "deep-dive"
---

# Deep Dive: Windows DNS Server — Architecture, Operations & Troubleshooting

**Topic:** Windows DNS Server Comprehensive Guide  
**Category:** Networking  
**Level:** 中级 / 高级 (Intermediate / Advanced)  
**Last Updated:** 2026-03-12

---

# 中文版

## 1. 概述 (Overview)

DNS（Domain Name System）是互联网和企业网络的核心基础服务，负责将人类可读的域名（如 `www.contoso.com`）解析为计算机可处理的 IP 地址（如 `192.0.2.10`）。在 Windows 生态系统中，DNS 不仅承担标准的名称解析功能，更是 Active Directory Domain Services (AD DS) 的基石——域控制器定位、Kerberos 认证、站点感知复制等关键操作都依赖 DNS 完成。

Windows DNS Server 是 Windows Server 操作系统内置的 DNS 服务角色，支持标准 DNS 协议（RFC 1034/1035）以及多项 Microsoft 增强功能，包括 AD 集成区域、动态更新、DNS Policy、DNSSEC 等。理解 Windows DNS 的架构和运作机制，对于排查企业网络中的名称解析故障、Active Directory 问题，以及混合云环境中的 DNS 配置问题至关重要。

本文从 DNS Client 架构到 DNS Server 的核心行为，从基础概念到高级特性（Policy、DNSSEC、Azure DNS 集成），系统性地梳理 Windows DNS 的技术体系，重点面向 Support Engineer 的实战需求。

## 2. 核心概念 (Core Concepts)

### 2.1 DNS 命名空间与层级结构

DNS 命名空间是一个倒置的树形结构：
- **Root Domain（根域）**：位于顶端，用 `.` 表示
- **Top-Level Domains (TLD)**：如 `.com`、`.org`、`.net`、国家代码如 `.uk`
- **Second-Level Domains**：如 `contoso.com`
- **Subdomains**：如 `corp.contoso.com`

FQDN（完全限定域名）在 DNS 协议内部转换为 **Lookup Name** 格式：
```
FQDN: myhost.corp.contoso.com
Lookup Name: (6)myhost(4)corp(7)contoso(3)com(0)
```
每个标签前的数字表示该标签的字符长度，以 `(0)` 结尾表示根域。

### 2.2 DNS 区域类型

| 区域类型 | 描述 | 适用场景 |
|---------|------|---------|
| **Primary Zone** | 区域数据的读写副本 | 主 DNS 服务器 |
| **Secondary Zone** | 区域数据的只读副本，通过区域传输获取 | 容错和负载分担 |
| **Stub Zone** | 仅包含 NS、SOA 和 glue A 记录 | 了解子域的权威服务器 |
| **AD-Integrated Zone** | 区域数据存储在 AD 数据库中 | 域控制器上推荐使用 |

AD-Integrated Zone 的优势：
- 多主复制（Multi-master replication）：任何 DC 都可以写入
- 安全动态更新（Secure dynamic updates）
- 通过 AD 复制进行区域传输，无需配置传统区域传输

### 2.3 DNS 记录类型

| 记录类型 | 用途 |
|---------|------|
| **A** | 主机名 → IPv4 地址 |
| **AAAA** | 主机名 → IPv6 地址 |
| **CNAME** | 别名 → 规范名 |
| **MX** | 域名 → 邮件服务器 |
| **NS** | 域名 → 权威名称服务器 |
| **SOA** | 区域起始授权记录 |
| **PTR** | IP 地址 → 主机名（反向查找） |
| **SRV** | 服务定位记录（AD DS 大量使用） |
| **DNAME** | 域别名（将整个子树重定向） |
| **CAA** | Certificate Authority Authorization（Azure DNS 支持） |

### 2.4 Host Name vs NetBIOS Name

- **Host Name（主机名）**：通过 Windows Sockets (Winsock) 解析，使用 DNS。长度可达 255 字符。例如：`dc01.corp.contoso.com`
- **NetBIOS Name**：通过 NetBIOS API 解析，使用 WINS。最长 15 字符。例如：`DC01`

> 现代应用程序（包括所有 Internet 应用）都使用 Windows Sockets，因此主要使用 Host Name 和 DNS。

## 3. 工作原理 (How It Works)

### 3.1 DNS Client 架构

Windows DNS Client 的内部架构由三个关键组件组成：

```
┌─────────────────────────────────────┐
│          Application Layer          │
│   (GetAddrInfo / GetAddrInfoW)      │
├─────────────────────────────────────┤
│           ws2_32.dll                │
│      (Windows Sockets API)          │
│   NSPLookupService* functions       │
├─────────────────────────────────────┤
│           DNSapi.dll                │
│     (DNS API / Query & Update)      │
│   Open socket → Send request →      │
│   Receive response → Close socket   │
├─────────────────────────────────────┤
│          DNSrslvr.dll               │
│   (DNS Client Service / Resolver)   │
│   Runs as Network Service           │
│   RPC Interface → Cache/Resolver    │
└─────────────────────────────────────┘
```

**解析流程：**
1. 应用程序调用 `GetAddrInfoW()` 发起名称解析
2. `ws2_32.dll` 通过 NSP（Namespace Provider）框架路由请求
3. `DNSapi.dll` 构造 DNS 查询报文，通过 socket 发送
4. `DNSrslvr.dll`（DNS Client Service）维护本地缓存和 Resolver 逻辑
5. 查询按配置的 DNS 服务器顺序发送

### 3.2 DNS 查询类型

#### 递归查询 (Recursive Query)
- 客户端要求 DNS 服务器**必须**返回最终结果（成功或失败）
- 通过 DNS 报文的 **RD（Recursion Desired）标志**标识
- DNS Client 到本地 DNS Server 通常使用递归查询

```
Flags: RD = 1 (Recursion Desired)
       RA = 1 (Recursive query support available) — in response
```

#### 迭代查询 (Iterative Query)
- DNS 服务器返回**自己所知的最佳答案**（可能是 referral）
- DNS Server 与 Root Hints 或其他 DNS Server 之间使用迭代查询
- 服务器返回 referral 后，查询方需要继续向被引荐的服务器查询

#### Root Hints
- 包含 13 个 Internet 根服务器的 FQDN 列表
- 安装 DNS 角色时自动从 `cache.dns` 文件安装
- 当 DNS 服务器无法从本地区域、转发器或缓存解析查询时使用
- DNS 服务器与 Root Hint 服务器通信时**只使用迭代查询**

> **重要区分**：DNS 服务器上的"递归"（使用 Root Hints 解析）≠ "递归查询"（要求服务器提供完整答案）

### 3.3 DNS 转发器 (Forwarders)

**标准转发器 (Forwarder)**：将无法本地解析的查询转发到指定的外部 DNS 服务器。
- 最佳实践：使用中央转发 DNS 服务器进行 Internet 名称解析
- 可以将转发器隔离在 DMZ 中以增强安全性

**条件转发器 (Conditional Forwarder)**：根据查询的域名将查询转发到特定的 DNS 服务器。
- 例如：所有 `*.fabrikam.corp` 的查询转发到 `198.51.100.10`
- 在 Windows Server 中可通过 AD 复制将配置分发到其他 DNS 服务器
- **最佳实践**：在拥有多个内部命名空间时使用条件转发器

**跨林信任部署**：条件转发器是建立跨林信任的前提——两个林的 DNS 服务器必须能相互解析对方域名。

### 3.4 DNS 委派 (Delegation)

当需要将子域的管理权委派给不同的 DNS 服务器时，使用 DNS Delegation：

1. 在父域区域中创建委派记录（灰色文件夹图标）
2. 委派记录包含子域的 NS 记录和 glue A 记录
3. 客户端查询子域时，父域 DNS 会返回 referral 指向子域的权威服务器

**关键注意 — ForwardDelegations 注册表值：**
```
HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters
    ForwardDelegations = 0 (default)
```
- 默认值 `0`：DNS 服务器收到对委派子域的查询时，直接将查询发送给子域的权威服务器
- 值 `1`：DNS 服务器将委派子域的查询也转发给 Forwarder，就像处理其他区域的查询一样
- **这是一个常见的坑**：如果发现 delegation 不生效，查询仍然走 forwarder，检查此注册表值

### 3.5 动态更新 (Dynamic Updates)

DNS 动态更新允许客户端自动注册和更新自己的 DNS 记录。

**更新流程：**
1. DNS Client 发送 **SOA 查询**，查找 FQDN 所在区域的主服务器
2. 获取主服务器地址后，发送**动态更新请求**
3. 如果更新失败，客户端发送 NS 查询获取区域的其他 DNS 服务器
4. 重复尝试直到成功或所有服务器都失败

**非安全更新 vs 安全更新：**
- **非安全更新 (Non-secure)**：任何客户端的更新请求都会被接受。客户端**总是先尝试非安全更新**
- **安全更新 (Secure)**：仅在 AD-Integrated Zone 上可用，使用 Kerberos 认证验证客户端身份。如果非安全更新被拒绝，客户端才会尝试安全更新

**DHCP 与 DNS 动态更新集成：**
- DHCP Client 使用 **Option 81 (Client FQDN)** 告知 DHCP Server 期望的更新方式
- Flag `0`：Client 自行注册 A 记录
- Flag `1`：Client 请求 DHCP Server 代为注册 A 记录
- Flag `3`：DHCP Server 无论客户端请求如何都代为注册 A 记录
- **所有者问题**：由 DHCP Server 代注册的记录，其所有者是 DHCP Server 的凭据账户

### 3.6 Aging 和 Scavenging

用于清理区域中过期的（stale）资源记录：

- **No-Refresh Interval（非刷新间隔）**：记录创建/刷新后，在此时间内不接受刷新（默认 7 天）
- **Refresh Interval（刷新间隔）**：No-Refresh Interval 结束后，允许刷新的时间窗口（默认 7 天）
- **Scavenging**：删除在 Refresh Interval 到期后仍未刷新的记录

**时间戳行为的一个重要细节：**
> 当 `Aging_InitZoneUpdate` 函数发现现有记录的时间戳为零时，会将记录视为静态记录（禁用 Aging）。这解释了为什么 SRV 记录（如 `_kerberos`）有时保持静态即使被动态更新——如果记录最初是以零时间戳创建的，之后的动态更新不会启用 Aging。
>
> **解决方法**：删除所有静态记录，重启 Netlogon 服务触发 DNS 注册。

## 4. 高级特性

### 4.1 DNS Policy

DNS Policy（Server 2016+）允许 DNS 服务器根据客户端条件动态响应查询。类似于 DHCP Policy 的概念。

**核心概念：**

| 概念 | 说明 |
|------|------|
| **Query Resolution Policy** | 控制查询处理，分 Server Level 和 Zone Level |
| **Zone Transfer Policy** | 控制区域传输行为 |
| **Client Subnet** | 定义客户端子网，用于策略匹配 |
| **Zone Scope** | 同一区域的不同视图，可包含不同的记录集 |
| **Recursion Scope** | 控制递归查询行为 |

**Policy 条件 (Criteria)**：
- `ClientSubnet` — 客户端 IP 子网
- `Fqdn` — 查询的 FQDN
- `TimeOfDay` — 时间段
- `TransportProtocol` — TCP/UDP
- `InternetProtocol` — IPv4/IPv6
- `ServerInterfaceIP` — 服务器接口 IP
- `QType` — 查询类型（A、AAAA、SRV 等）

**Policy 动作**：Allow、Deny

**注册表位置：**
```
Server Policies: HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\DNS Server\Policies
Zone Policies:   HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\DNS Server\Zones\<ZoneName>\Policies
Client Subnets:  HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\DNS Server\Client Subnets
Zone Scopes:     HKLM\...\Zones\<ZoneName>\ZoneScopes
AD 位置:         CN=ZoneScopeContainer,DC=<ZoneName>,CN=MicrosoftDNS,DC=DomainDnsZones,DC=<domain>
```

**常见应用场景：**
- **基于地理位置的流量管理** (Geo-location based traffic management)
- **Split-Brain DNS**：内外网返回不同解析结果
- **基于时间的 DNS 响应**
- **阻止特定类型的查询**（如阻止 SRV 查询）
- **递归限制**：限制特定客户端或域名的递归查询

**PowerShell 配置示例：**
```powershell
# 创建客户端子网
Add-DnsServerClientSubnet -Name 'InternalNet' -IPv4Subnet '10.0.0.0/24'

# 创建区域作用域
Add-DnsServerZoneScope -Name 'InternalScope' -ZoneName 'contoso.com'

# 在 scope 中添加不同的记录
Add-DnsServerResourceRecordA -Name 'www' -ZoneName 'contoso.com' `
    -ZoneScope 'InternalScope' -IPv4Address '10.0.0.100'

# 创建策略 — 内网客户端使用 InternalScope
Add-DnsServerQueryResolutionPolicy -Name 'InternalPolicy' `
    -ZoneName 'contoso.com' -ZoneScope 'InternalScope' `
    -ClientSubnet 'EQ,InternalNet'

# 阻止递归查询特定域名
Add-DnsServerQueryResolutionPolicy -Name 'BlockRecursion' `
    -Fqdn 'EQ,*.example.com' -Action DENY -ApplyOnRecursion
```

> **注意**：Policy 的 Processing Order 非常重要。如果多个 Policy 匹配，按顺序第一个匹配的生效。使用 `Set-DnsServerQueryResolutionPolicy -ProcessingOrder` 调整顺序。

> **时区注意**：TimeOfDay criteria 在系统时区变更后**不会自动同步**。

### 4.2 DNSSEC

DNSSEC（DNS Security Extensions）通过数字签名防止 DNS 欺骗（DNS Spoofing）攻击。

**工作原理：**
1. 权威 DNS 服务器对区域进行签名（Zone Signing）
2. 签名后产生额外的 DNSSEC 资源记录
3. 递归 DNS 服务器使用 DNSKEY 验证响应的数字签名
4. 如果验证通过，将结果返回客户端；如果失败，返回 SERVFAIL

**DNSSEC 资源记录：**

| 记录类型 | 用途 |
|---------|------|
| **RRSIG** | 包含密码学签名 |
| **DNSKEY** | 包含公开签名密钥 |
| **DS** | 包含 DNSKEY 记录的哈希值 |
| **NSEC / NSEC3** | 用于显式否定存在（Denial-of-Existence） |

**密钥类型：**
- **KSK (Key Signing Key)**：用于签名所有 DNSKEY 记录，是信任链的一部分。客户端用 KSK 验证 DNS 响应
- **ZSK (Zone Signing Key)**：用于签名区域数据，通常比 KSK 更频繁滚动更新

**部署步骤：**
1. 在 DNS Manager 中使用 Zone Signing Wizard 签名区域
2. 配置 KSK 和 ZSK（推荐使用 PDC 作为 Key Master）
3. 配置 NSEC 或 NSEC3
4. 分发 Trust Anchors
5. 通过 GPO 配置 NRPT (Name Resolution Policy Table) 策略让客户端请求 DNSSEC 验证

**Trust Anchors 位置：**
- DNS Server Console 中的 Trust Points
- ADSI Edit 中的 AD 对象

> **常见问题**：客户配置了较短的密钥有效期（1-2 年），密钥过期后 DNSSEC 验证失败。**建议延长密钥生命周期**或建立密钥滚动更新流程。

**PowerShell 示例：**
```powershell
# 使用默认设置签名区域
Invoke-DnsServerZoneSign -ZoneName contoso.com -SignWithDefault -Force

# 查看区域记录（含 DNSSEC 记录）
Get-DnsServerResourceRecord -ZoneName contoso.com

# 启用 RFC 5011 自动密钥滚动
Set-DnsServerDnsSecZoneSetting -ZoneName "contoso.com" -EnableRfc5011Rollover $true -PassThru -Verbose
```

### 4.3 DNS Global Names Zone (GNZ)

当组织拥有很长的域名（如 `dns.china.east.shanghai.child.corp.contoso.com`），Global Names Zone 可以为这些长名称创建简短的 CNAME 别名。

**特点：**
- **仅在 AD 集成区域中生效**
- 创建 CNAME 记录指向长域名
- 用户可以直接 ping 短名称，无需配置 DNS 后缀

```powershell
# 启用 Global Names Zone
Set-DnsServerGlobalNameZone -Enable $True -PassThru

# 创建 GlobalNames 区域
Add-DnsServerPrimaryZone -Name "GlobalNames" -ZoneFile "GlobalNames.dns"

# 添加短名称别名
Add-DnsServerResourceRecordCName -Name "MyApp" `
    -HostNameAlias "app.china.east.corp.contoso.com" `
    -ZoneName "GlobalNames"

# 现在可以直接 ping myapp
```

### 4.4 Azure DNS 集成

**Azure DNS Public Zone：**
- 在 Azure Portal 创建，遵循 RFC 命名规范
- 默认自动创建 SOA 和 NS 记录
- 支持 A、AAAA、CAA、CNAME、MX、NS、TXT、PTR 记录
- 最多 10,000 条记录
- 注意：Azure DNS 支持 CAA 记录，而 On-Prem DNS 不支持

**Azure DNS Private Zone：**
- 用于 Azure 虚拟网络内部的名称解析
- 不暴露到公网

**混合 DNS 基础设施：**
- Azure Private Resolver 可以实现 On-Prem 和 Azure 之间的 DNS 解析互通
- 条件转发器将 Azure 域名查询转发到 Azure DNS

## 5. 常见问题与排查 (Common Issues & Troubleshooting)

### 5.1 DNS 解析失败

**症状**：客户端无法解析域名

**排查思路：**
1. 检查 DNS Client 配置：`ipconfig /all` 查看 DNS 服务器配置
2. 清除 DNS 缓存：`ipconfig /flushdns`
3. 查看 DNS 缓存：`ipconfig /displaydns`
4. 使用 `Resolve-DnsName`（推荐）或 `nslookup` 测试
5. 检查 DNS Server 的转发器和根提示配置
6. 检查条件转发器是否正确配置

> **重要**：排查时优先使用 `Resolve-DnsName`，不要使用 `nslookup`。`nslookup` 有自己的 DNS 解析逻辑，不经过标准的 DNS Client Service，可能产生误导性的结果。

### 5.2 Ping 与 Nslookup 结果不同

**原因**：
- `ping` 使用 `GetAddrInfoW` → DNS Client Service（包含缓存）
- `nslookup` 直接发送 DNS 查询到 DNS Server，绕过 DNS Client Cache
- 如果缓存中有记录（如手动添加的缓存），ping 会使用缓存结果，而 nslookup 会从 DNS Server 获取

### 5.3 DNS 后缀附加 (Suffix Appending) 问题

**行为：**
- Ping 短名称时，DNS Client 会按照**主 DNS 后缀**和**附加后缀列表**依次附加后缀并查询
- Ping FQDN（带 `.` 结尾）时，不附加任何后缀
- DNS 后缀递减（Devolution）可以控制附加的子后缀层级
- `nslookup` 的后缀附加行为与 `ping` 不同

### 5.4 动态更新失败

**可能原因：**
- 区域不允许动态更新
- 安全更新：客户端没有更新权限
- DHCP 代注册：记录所有者是旧的 DHCP 服务器账户
- 网络问题导致无法到达权威 DNS 服务器

### 5.5 DNS 记录时间戳异常

**症状**：SRV 记录（如 `_kerberos`）保持静态，即使被动态更新

**原因**：当 `Aging_InitZoneUpdate` 发现现有记录的时间戳为零时，视为静态记录，不启用 Aging

**解决方法**：删除所有静态记录，重启 Netlogon 服务触发重新注册

### 5.6 Delegation 不生效

**症状**：DNS 查询没有发送到委派的子域服务器，而是走了 Forwarder

**排查**：检查 `ForwardDelegations` 注册表值
```
HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters
    ForwardDelegations (DWORD) — 默认 0
```
如果值为 `1`，DNS Server 会将委派域的查询也转发给 Forwarder。

### 5.7 AD 集成 DNS 区域意外删除

**恢复方法 1 — 通过 AD Recycle Bin：**
```powershell
# 查找已删除的区域对象
Get-ADObject -Filter 'isdeleted -eq $true -and msds-lastKnownRdn -eq "..Deleted-contoso.com"' `
    -IncludeDeletedObjects `
    -SearchBase "DC=DomainDnsZones,DC=corp,DC=contoso,DC=com" -Property *

# 恢复区域
Get-ADObject -Filter 'isdeleted -eq $true -and msds-lastKnownRdn -eq "..Deleted-contoso.com"' `
    -IncludeDeletedObjects `
    -SearchBase "DC=DomainDnsZones,DC=corp,DC=contoso,DC=com" | Restore-ADObject

# 恢复区域内的记录
Get-ADObject -Filter 'isdeleted -eq $true -and lastKnownParent -like "*DC=contoso.com*"' `
    -IncludeDeletedObjects `
    -SearchBase "DC=DomainDnsZones,DC=corp,DC=contoso,DC=com" | Restore-ADObject
```

> 注意：区域名和 DNS Partition 名可能不同（例如区域 `contoso.edu`，分区 `corp.contoso.com`）

**恢复方法 2 — 通过区域文件备份：**
```powershell
# 导出区域文件（预先备份）
Export-DnsServerZone -Name "contoso.com" -FileName "contoso.com.dns" -Verbose

# 恢复步骤：
# 1. 创建 Primary Zone（非 AD 集成），使用现有文件
# 2. 将区域转换为 AD 集成
# 3. 启用安全动态更新
# 4. 配置复制范围
# 5. 触发 AD 复制：repadmin /syncall
```

## 6. DNS ETW Tracing 实战 (Practical Tracing)

### 6.1 DNS Client ETL 抓取

```cmd
netsh trace start netconnection capture=yes ^
    provider={406F31B6-E81C-457a-B5C3-62C1BE5778C1} ^
    level=7 keywords=0x3FFE57 ^
    tracefile=DnsDebug.etl
```

### 6.2 ETL 分析工具

- **TextAnalysisTool (TAT)**：https://textanalysistool.github.io/
- 使用 **DNS.AllPurpose.tat** 过滤器简化分析
- 排除不相关线程的噪音

### 6.3 ETL 中的关键事件

**GetAddrInfoW 调用：**
```
[Microsoft-Windows-Winsock NameResolution Event/Operational]
GetAddrInfoW is called for queryName server01, serviceName NULL, 
flags 2, family 0, socketType 0, protocol 0
```

**解析完成：**
```
GetAddrInfoW is completed for queryName server01 
with status 0 and result 192.0.2.11;
```

### 6.4 DNS Client 在网络切换时的行为

当从 LAN 切换到 Wi-Fi 时：
1. 旧网络适配器断开
2. DNS Client 清除该适配器相关的缓存记录
3. 新网络适配器获取新的 DNS 配置
4. 触发新的 DNS 注册

VPN 连接时：
- VPN 适配器获取额外的 DNS 配置
- DNS 缓存行为可能受到 NRPT 策略影响

## 7. 关键配置与参数 (Key Configurations)

| 配置项/参数 | 默认值 | 说明 | 常见调优场景 |
|------------|--------|------|------------|
| `ForwardDelegations` | 0 | 控制委派域查询是否走 Forwarder | Delegation 不生效时检查 |
| No-Refresh Interval | 7 天 | 不接受刷新的时间段 | Scavenging 过度删除记录时调整 |
| Refresh Interval | 7 天 | 允许刷新的时间段 | 与 No-Refresh 配合调整 |
| Dynamic Update TTL | 15 分钟 | DHCP Client 注册的 Host 记录 TTL | 解析延迟时检查 |
| Update Interval | 24 小时 | 客户端自动刷新注册的间隔 | 网络频繁变更时考虑调整 |
| DNS Suffix Devolution | Enabled | 控制后缀递减搜索 | 短名称解析异常时检查 |
| DNSSEC KSK Lifetime | 可配置 | KSK 密钥有效期 | 避免设置过短导致验证失败 |
| DNSSEC ZSK Lifetime | 168 小时 (默认) | ZSK 密钥有效期 | 延长以减少滚动频率 |

## 8. 与相关技术的对比 (Comparison)

| 维度 | Windows DNS Server | BIND | Azure DNS |
|------|-------------------|------|-----------|
| 平台 | Windows Server | Linux/Unix | Azure Cloud |
| AD 集成 | 原生支持 | 不支持 | 不适用 |
| 动态更新 | 支持（含安全更新） | 支持（基础） | 通过 API/Portal |
| DNS Policy | 支持 (Server 2016+) | Views (类似) | Traffic Manager |
| DNSSEC | 支持 | 支持 | Public Zone 支持 |
| 管理方式 | GUI / PowerShell / dnscmd | 配置文件 | Portal / CLI / API |
| 适用场景 | 企业 AD 环境 | Internet 权威服务器 | 云原生应用 |

## 9. 参考资料 (References)

暂无可验证的参考文档

---

# English Version

## 1. Overview

DNS (Domain Name System) is a foundational infrastructure service for both the Internet and enterprise networks, responsible for translating human-readable domain names (e.g., `www.contoso.com`) into machine-processable IP addresses (e.g., `192.0.2.10`). In the Windows ecosystem, DNS not only provides standard name resolution but also serves as the cornerstone of Active Directory Domain Services (AD DS) — domain controller location, Kerberos authentication, and site-aware replication all depend on DNS.

Windows DNS Server is a built-in DNS service role in the Windows Server operating system, supporting standard DNS protocols (RFC 1034/1035) along with several Microsoft enhancements including AD-Integrated Zones, Dynamic Updates, DNS Policy, and DNSSEC. Understanding Windows DNS architecture and operational mechanics is critical for troubleshooting name resolution failures, Active Directory issues, and hybrid-cloud DNS configuration problems.

This article systematically covers the Windows DNS technology stack — from DNS Client architecture to DNS Server core behaviors, from fundamental concepts to advanced features (Policy, DNSSEC, Azure DNS integration) — with a focus on practical needs for Support Engineers.

## 2. Core Concepts

### 2.1 DNS Namespace and Hierarchy

The DNS namespace is an inverted tree structure:
- **Root Domain**: At the top, represented by `.`
- **Top-Level Domains (TLD)**: `.com`, `.org`, `.net`, country codes like `.uk`
- **Second-Level Domains**: e.g., `contoso.com`
- **Subdomains**: e.g., `corp.contoso.com`

FQDNs are internally converted to **Lookup Name** format in the DNS protocol:
```
FQDN: myhost.corp.contoso.com
Lookup Name: (6)myhost(4)corp(7)contoso(3)com(0)
```

### 2.2 DNS Zone Types

| Zone Type | Description | Use Case |
|-----------|-------------|----------|
| **Primary Zone** | Read/write copy of zone data | Primary DNS server |
| **Secondary Zone** | Read-only copy via zone transfer | Fault tolerance and load distribution |
| **Stub Zone** | Contains only NS, SOA, and glue A records | Discovering authoritative servers for subdomains |
| **AD-Integrated Zone** | Zone data stored in the AD database | Recommended on domain controllers |

AD-Integrated Zone advantages:
- Multi-master replication: any DC can write
- Secure dynamic updates
- Zone transfer via AD replication (no traditional zone transfer configuration needed)

### 2.3 DNS Record Types

| Record Type | Purpose |
|-------------|---------|
| **A** | Hostname → IPv4 address |
| **AAAA** | Hostname → IPv6 address |
| **CNAME** | Alias → Canonical name |
| **MX** | Domain → Mail server |
| **NS** | Domain → Authoritative name server |
| **SOA** | Start of Authority record |
| **PTR** | IP address → Hostname (reverse lookup) |
| **SRV** | Service locator record (heavily used by AD DS) |
| **DNAME** | Domain alias (redirects an entire subtree) |
| **CAA** | Certificate Authority Authorization (supported by Azure DNS) |

### 2.4 Host Name vs NetBIOS Name

- **Host Name**: Resolved through Windows Sockets (Winsock) using DNS. Up to 255 characters. Example: `dc01.corp.contoso.com`
- **NetBIOS Name**: Resolved through the NetBIOS API using WINS. Up to 15 characters. Example: `DC01`

> Modern applications (including all Internet apps) use Windows Sockets and therefore primarily use host names and DNS.

## 3. How It Works

### 3.1 DNS Client Architecture

The Windows DNS Client's internal architecture consists of three key components:

```
┌─────────────────────────────────────┐
│          Application Layer          │
│   (GetAddrInfo / GetAddrInfoW)      │
├─────────────────────────────────────┤
│           ws2_32.dll                │
│      (Windows Sockets API)          │
│   NSPLookupService* functions       │
├─────────────────────────────────────┤
│           DNSapi.dll                │
│     (DNS API / Query & Update)      │
│   Open socket → Send request →      │
│   Receive response → Close socket   │
├─────────────────────────────────────┤
│          DNSrslvr.dll               │
│   (DNS Client Service / Resolver)   │
│   Runs as Network Service           │
│   RPC Interface → Cache/Resolver    │
└─────────────────────────────────────┘
```

**Resolution flow:**
1. Application calls `GetAddrInfoW()` to initiate name resolution
2. `ws2_32.dll` routes the request through the NSP (Namespace Provider) framework
3. `DNSapi.dll` constructs the DNS query packet and sends it over a socket
4. `DNSrslvr.dll` (DNS Client Service) maintains the local cache and resolver logic
5. Queries are sent to configured DNS servers in order

### 3.2 DNS Query Types

#### Recursive Query
- Client requests that the DNS server **must** return a final answer (success or failure)
- Identified by the **RD (Recursion Desired) flag** in the DNS message
- DNS Client → local DNS Server typically uses recursive queries

```
Flags: RD = 1 (Recursion Desired)
       RA = 1 (Recursive query support available) — in response
```

#### Iterative Query
- DNS server returns the **best answer it knows** (may be a referral)
- Used between DNS Servers and Root Hints or other DNS Servers
- After receiving a referral, the querying party must continue querying the referred server

#### Root Hints
- A list of 13 Internet root server FQDNs
- Automatically installed from `cache.dns` when installing the DNS role
- Used when the DNS server cannot resolve a query from local zones, forwarders, or cache
- DNS server communicates with root hint servers using **iterative queries only**

> **Important distinction**: "Recursion" on a DNS server (using Root Hints to resolve) ≠ "Recursive query" (asking a server to provide a complete answer)

### 3.3 DNS Forwarders

**Standard Forwarder**: Forwards queries that cannot be resolved locally to specified external DNS servers.
- Best practice: Use a central forwarding DNS server for Internet name resolution
- Isolate the forwarder in a DMZ for enhanced security

**Conditional Forwarder**: Forwards queries based on the queried domain name to specific DNS servers.
- Example: All `*.fabrikam.corp` queries forwarded to `198.51.100.10`
- In Windows Server, configuration can be distributed via AD replication
- **Best practice**: Use conditional forwarders when you have multiple internal namespaces

**Cross-Forest Trust Deployment**: Conditional forwarders are a prerequisite for cross-forest trust — DNS servers in both forests must be able to resolve each other's domain names.

### 3.4 DNS Delegation

Used when you need to delegate management of a subdomain to different DNS servers:

1. Create delegation records in the parent zone (grey folder icon)
2. Delegation records contain NS records and glue A records for the subdomain
3. When clients query the subdomain, the parent DNS returns a referral to the subdomain's authoritative server

**Critical Note — ForwardDelegations registry value:**
```
HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters
    ForwardDelegations = 0 (default)
```
- Default `0`: DNS server sends queries for delegated subzones directly to the subzone's authoritative server
- Value `1`: DNS server forwards queries for delegated subzones to forwarders, just like queries for other zones
- **This is a common pitfall**: If delegation doesn't work and queries still go to forwarders, check this registry value

### 3.5 Dynamic Updates

DNS Dynamic Updates allow clients to automatically register and update their DNS records.

**Update flow:**
1. DNS Client sends an **SOA query** to find the primary server for the zone containing its FQDN
2. Sends a **dynamic update request** to the primary server
3. If update fails, client sends an NS query to get other DNS servers for the zone
4. Retries until successful or all servers are exhausted

**Non-Secure vs Secure Updates:**
- **Non-secure**: Any client's update request is accepted. Clients **always try non-secure updates first**
- **Secure**: Only available on AD-Integrated Zones, uses Kerberos to authenticate client identity. Clients try secure updates only if non-secure is refused

**DHCP and DNS Dynamic Update Integration:**
- DHCP Client uses **Option 81 (Client FQDN)** to inform the DHCP Server of desired update behavior
- Flag `0`: Client registers the A record itself
- Flag `1`: Client requests DHCP Server to register the A record
- Flag `3`: DHCP Server registers the A record regardless of client request
- **Ownership issue**: Records registered by the DHCP Server are owned by the DHCP Server's credential account

### 3.6 Aging and Scavenging

Mechanism for cleaning up stale resource records:

- **No-Refresh Interval**: After record creation/refresh, no refresh accepted during this period (default 7 days)
- **Refresh Interval**: After No-Refresh Interval, time window for refresh (default 7 days)
- **Scavenging**: Removes records not refreshed after Refresh Interval expires

**Important timestamp behavior detail:**
> When `Aging_InitZoneUpdate` finds an existing record with a zero timestamp, it treats the record as static (Aging disabled). This explains why SRV records (e.g., `_kerberos`) sometimes remain static even after dynamic updates — if the record was initially created with a zero timestamp, subsequent dynamic updates won't enable Aging.
>
> **Solution**: Delete all static records and restart the Netlogon service to trigger re-registration.

## 4. Advanced Features

### 4.1 DNS Policy

DNS Policy (Server 2016+) allows DNS servers to respond dynamically to queries based on client criteria.

**Core concepts:**

| Concept | Description |
|---------|-------------|
| **Query Resolution Policy** | Controls query processing, at Server Level and Zone Level |
| **Zone Transfer Policy** | Controls zone transfer behavior |
| **Client Subnet** | Defines client subnets for policy matching |
| **Zone Scope** | Different views of the same zone with different record sets |
| **Recursion Scope** | Controls recursive query behavior |

**Policy Criteria:**
- `ClientSubnet` — Client IP subnet
- `Fqdn` — Queried FQDN
- `TimeOfDay` — Time range
- `TransportProtocol` — TCP/UDP
- `InternetProtocol` — IPv4/IPv6
- `ServerInterfaceIP` — Server interface IP
- `QType` — Query type (A, AAAA, SRV, etc.)

**Common use cases:**
- **Geo-location based traffic management**
- **Split-Brain DNS**: Return different results for internal/external clients
- **Time-based DNS responses**
- **Blocking specific query types** (e.g., blocking SRV queries)
- **Recursion restriction**: Limit recursive queries for specific clients or domains

**PowerShell configuration example:**
```powershell
# Create client subnet
Add-DnsServerClientSubnet -Name 'InternalNet' -IPv4Subnet '10.0.0.0/24'

# Create zone scope
Add-DnsServerZoneScope -Name 'InternalScope' -ZoneName 'contoso.com'

# Add different records in the scope
Add-DnsServerResourceRecordA -Name 'www' -ZoneName 'contoso.com' `
    -ZoneScope 'InternalScope' -IPv4Address '10.0.0.100'

# Create policy — internal clients use InternalScope
Add-DnsServerQueryResolutionPolicy -Name 'InternalPolicy' `
    -ZoneName 'contoso.com' -ZoneScope 'InternalScope' `
    -ClientSubnet 'EQ,InternalNet'

# Block recursive queries for specific domain
Add-DnsServerQueryResolutionPolicy -Name 'BlockRecursion' `
    -Fqdn 'EQ,*.example.com' -Action DENY -ApplyOnRecursion
```

> **Note**: Policy Processing Order is critical. When multiple policies match, the first match in order takes effect. Use `Set-DnsServerQueryResolutionPolicy -ProcessingOrder` to adjust.

> **Timezone caveat**: TimeOfDay criteria does **not automatically sync** when the system timezone changes.

### 4.2 DNSSEC

DNSSEC (DNS Security Extensions) prevents DNS spoofing attacks through digital signatures.

**How it works:**
1. Authoritative DNS server signs the zone (Zone Signing)
2. Signing produces additional DNSSEC resource records
3. Recursive DNS server uses DNSKEY to validate response digital signatures
4. If validation passes, returns result to client; if it fails, returns SERVFAIL

**DNSSEC Resource Records:**

| Record Type | Purpose |
|-------------|---------|
| **RRSIG** | Contains a cryptographic signature |
| **DNSKEY** | Contains the public signing key |
| **DS** | Contains the hash of a DNSKEY record |
| **NSEC / NSEC3** | For explicit denial-of-existence |

**Key Types:**
- **KSK (Key Signing Key)**: Signs all DNSKEY records, part of the chain of trust
- **ZSK (Zone Signing Key)**: Signs zone data, typically rolled over more frequently than KSK

**Deployment steps:**
1. Use Zone Signing Wizard in DNS Manager to sign the zone
2. Configure KSK and ZSK (recommend PDC as Key Master)
3. Configure NSEC or NSEC3
4. Distribute Trust Anchors
5. Configure NRPT (Name Resolution Policy Table) via GPO for client-side DNSSEC validation

> **Common issue**: Customers configure short key lifetimes (1-2 years), and when keys expire, DNSSEC validation fails. **Recommend extending key lifetime** or establishing a key rollover process.

### 4.3 DNS Global Names Zone (GNZ)

When organizations have very long domain names (e.g., `dns.china.east.shanghai.child.corp.contoso.com`), Global Names Zone can create short CNAME aliases.

**Features:**
- **Only works in AD-Integrated zones**
- Creates CNAME records pointing to long domain names
- Users can directly ping the short name without configuring DNS suffixes

```powershell
# Enable Global Names Zone
Set-DnsServerGlobalNameZone -Enable $True -PassThru

# Create GlobalNames zone
Add-DnsServerPrimaryZone -Name "GlobalNames" -ZoneFile "GlobalNames.dns"

# Add short name alias
Add-DnsServerResourceRecordCName -Name "MyApp" `
    -HostNameAlias "app.china.east.corp.contoso.com" `
    -ZoneName "GlobalNames"
```

### 4.4 Azure DNS Integration

**Azure DNS Public Zone:**
- Created in Azure Portal, follows RFC naming conventions
- Automatically creates SOA and NS records by default
- Supports A, AAAA, CAA, CNAME, MX, NS, TXT, PTR records
- Maximum 10,000 records
- Note: Azure DNS supports CAA records, while On-Prem DNS does not

**Azure DNS Private Zone:**
- For name resolution within Azure virtual networks
- Not exposed to the public Internet

**Hybrid DNS Infrastructure:**
- Azure Private Resolver enables DNS resolution between On-Prem and Azure
- Conditional forwarders route Azure domain queries to Azure DNS

## 5. Common Issues & Troubleshooting

### 5.1 DNS Resolution Failure

**Symptom**: Client cannot resolve domain names

**Troubleshooting approach:**
1. Check DNS Client configuration: `ipconfig /all` to view DNS server config
2. Clear DNS cache: `ipconfig /flushdns`
3. View DNS cache: `ipconfig /displaydns`
4. Test with `Resolve-DnsName` (recommended) or `nslookup`
5. Verify DNS Server forwarder and root hints configuration
6. Check conditional forwarder configuration

> **Important**: Use `Resolve-DnsName` for troubleshooting, not `nslookup`. `nslookup` has its own DNS resolution logic, bypasses the standard DNS Client Service, and may produce misleading results.

### 5.2 Different Results Between Ping and Nslookup

**Cause**:
- `ping` uses `GetAddrInfoW` → DNS Client Service (includes cache)
- `nslookup` sends DNS queries directly to DNS Server, bypassing DNS Client Cache
- If cache has entries (e.g., manually added), ping uses cache results while nslookup fetches from DNS Server

### 5.3 DNS Suffix Appending Issues

**Behavior:**
- Pinging a short name: DNS Client appends suffixes from the **primary DNS suffix** and **additional suffix list** in order
- Pinging an FQDN (ending with `.`): No suffixes appended
- DNS suffix devolution controls subdomain suffix levels appended
- `nslookup` suffix appending behavior differs from `ping`

### 5.4 Dynamic Update Failures

**Possible causes:**
- Zone does not allow dynamic updates
- Secure updates: Client lacks update permissions
- DHCP proxy registration: Record owner is an old DHCP server account
- Network issues preventing access to the authoritative DNS server

### 5.5 DNS Record Timestamp Anomalies

**Symptom**: SRV records (e.g., `_kerberos`) remain static despite being dynamically updated

**Cause**: When `Aging_InitZoneUpdate` finds existing records with zero timestamps, it treats them as static

**Solution**: Delete all static records and restart Netlogon service to trigger re-registration

### 5.6 Delegation Not Working

**Symptom**: DNS queries not sent to delegated subdomain server, going to forwarder instead

**Check**: `ForwardDelegations` registry value
```
HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters
    ForwardDelegations (DWORD) — default 0
```
If value is `1`, DNS Server forwards delegated domain queries to forwarders.

### 5.7 Accidentally Deleted AD-Integrated DNS Zone

**Recovery Method 1 — Via AD Recycle Bin:**
```powershell
# Find deleted zone object
Get-ADObject -Filter 'isdeleted -eq $true -and msds-lastKnownRdn -eq "..Deleted-contoso.com"' `
    -IncludeDeletedObjects `
    -SearchBase "DC=DomainDnsZones,DC=corp,DC=contoso,DC=com" -Property *

# Restore zone
Get-ADObject -Filter 'isdeleted -eq $true -and msds-lastKnownRdn -eq "..Deleted-contoso.com"' `
    -IncludeDeletedObjects `
    -SearchBase "DC=DomainDnsZones,DC=corp,DC=contoso,DC=com" | Restore-ADObject

# Restore records within the zone
Get-ADObject -Filter 'isdeleted -eq $true -and lastKnownParent -like "*DC=contoso.com*"' `
    -IncludeDeletedObjects `
    -SearchBase "DC=DomainDnsZones,DC=corp,DC=contoso,DC=com" | Restore-ADObject
```

**Recovery Method 2 — Via zone file backup:**
```powershell
# Export zone file (pre-backup)
Export-DnsServerZone -Name "contoso.com" -FileName "contoso.com.dns" -Verbose

# Recovery steps:
# 1. Create Primary Zone (non-AD-integrated) using existing file
# 2. Convert zone to AD-Integrated
# 3. Enable secure dynamic updates
# 4. Configure replication scope
# 5. Trigger AD replication: repadmin /syncall
```

## 6. DNS ETW Tracing in Practice

### 6.1 DNS Client ETL Capture

```cmd
netsh trace start netconnection capture=yes ^
    provider={406F31B6-E81C-457a-B5C3-62C1BE5778C1} ^
    level=7 keywords=0x3FFE57 ^
    tracefile=DnsDebug.etl
```

### 6.2 ETL Analysis Tools

- **TextAnalysisTool (TAT)**: https://textanalysistool.github.io/
- Use the **DNS.AllPurpose.tat** filter to simplify analysis
- Exclude noise from unrelated threads

### 6.3 Key Events in ETL

**GetAddrInfoW call:**
```
[Microsoft-Windows-Winsock NameResolution Event/Operational]
GetAddrInfoW is called for queryName server01, serviceName NULL, 
flags 2, family 0, socketType 0, protocol 0
```

**Resolution complete:**
```
GetAddrInfoW is completed for queryName server01 
with status 0 and result 192.0.2.11;
```

### 6.4 DNS Client Behavior During Network Switching

When switching from LAN to Wi-Fi:
1. Old network adapter disconnects
2. DNS Client clears cache entries related to that adapter
3. New network adapter gets new DNS configuration
4. Triggers new DNS registration

During VPN connection:
- VPN adapter receives additional DNS configuration
- DNS cache behavior may be affected by NRPT policies

## 7. Key Configurations

| Configuration | Default | Description | Common Tuning Scenario |
|--------------|---------|-------------|----------------------|
| `ForwardDelegations` | 0 | Controls whether delegated zone queries go to forwarders | Check when delegation doesn't work |
| No-Refresh Interval | 7 days | Period during which no refresh accepted | Adjust when scavenging deletes records too aggressively |
| Refresh Interval | 7 days | Window for refresh after No-Refresh period | Adjust in conjunction with No-Refresh |
| Dynamic Update TTL | 15 min | TTL for DHCP Client registered host records | Check when experiencing resolution delays |
| Update Interval | 24 hours | Automatic client registration refresh interval | Consider adjusting for frequent network changes |
| DNS Suffix Devolution | Enabled | Controls suffix devolution search | Check when short name resolution is abnormal |
| DNSSEC KSK Lifetime | Configurable | KSK key validity period | Avoid setting too short to prevent validation failures |
| DNSSEC ZSK Lifetime | 168 hours (default) | ZSK key validity period | Extend to reduce rollover frequency |

## 8. Comparison with Related Technologies

| Dimension | Windows DNS Server | BIND | Azure DNS |
|-----------|-------------------|------|-----------|
| Platform | Windows Server | Linux/Unix | Azure Cloud |
| AD Integration | Native support | Not supported | N/A |
| Dynamic Updates | Supported (including secure) | Supported (basic) | Via API/Portal |
| DNS Policy | Supported (Server 2016+) | Views (similar concept) | Traffic Manager |
| DNSSEC | Supported | Supported | Public Zone supported |
| Management | GUI / PowerShell / dnscmd | Configuration files | Portal / CLI / API |
| Best For | Enterprise AD environments | Internet authoritative servers | Cloud-native applications |

## 9. References

暂无可验证的参考文档 / No verifiable reference links available at this time.
