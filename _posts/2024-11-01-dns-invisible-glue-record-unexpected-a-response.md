---
layout: post
title: "Case Summary: DNS 服务器因隐藏 Glue 记录返回意外 A 记录"
date: 2024-11-01
categories: [DNS, Windows-Server]
tags: [dns, glue-record, cname, delegation, nslookup, powershell, dns-etl, adsi, invisible-record]
---

# Case Summary: DNS 服务器因隐藏 Glue 记录返回意外 A 记录

**Product/Service:** Windows DNS Server

---

## 1. 症状 (Symptoms)

- 客户报告 Windows DNS 服务器对特定 DNS 名称 `mydesk.contoso.corp` 返回了**意外的 A 记录**
- 在 **非递归查询（norecurse）** 模式下，DNS 服务器不仅返回了预期的 CNAME 记录，还返回了一条 A 记录（IP: `10.50.70.59`）。正常行为下，非递归模式应该只返回 DNS 服务器本地拥有的记录（即 CNAME）
- 对比另一条正常工作的 DNS 名称 `collaborate.contoso.corp`，非递归查询只返回 CNAME，符合预期
- 返回的 A 记录 IP 地址 `10.50.70.59` 与 F5 DNS 上配置的实际 IP 不同，是**错误的 IP**

## 2. 背景 (Background / Environment)

- **DNS 架构**：
  - Windows DNS 服务器（`YOURDC01.contoso.corp`，IP: `10.50.16.41`）为 `contoso.corp` 区域的权威服务器
  - `glb.contoso.corp` 是 `contoso.corp` 区域下的一个 **委派（delegation）**，指向 F5 DNS 服务器
  - F5 DNS 服务器托管 `glb.contoso.corp` 子域下的 A 记录
- **DNS 记录配置**：
  - Windows DNS 服务器上配置了 CNAME：`mydesk.contoso.corp` → `mydesk.glb.contoso.corp`
  - F5 DNS 服务器上配置了 A 记录：`mydesk.glb.contoso.corp`
- **客户端**：Windows 客户端

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

1. **对比非递归与递归查询行为**
   - 对有问题的名称 `mydesk.contoso.corp` 和正常名称 `collaborate.contoso.corp` 分别执行非递归和递归查询
   - **发现**：
     - `mydesk.contoso.corp`：非递归查询返回 CNAME + A（2 条 answers），**不正常**
     - `collaborate.contoso.corp`：非递归查询只返回 CNAME（1 条 answer），**正常**
   - **结论**：问题仅影响 `mydesk.contoso.corp`，DNS 服务器在非递归模式下错误地返回了 A 记录

   ```
   # 有问题的名称 - 非递归查询（不应返回 A 记录，但实际返回了）
   nslookup -debug -norecurse mydesk.contoso.corp. 10.50.16.41
   # answers = 2（CNAME + A）← 异常

   # 正常名称 - 非递归查询（只返回 CNAME，符合预期）
   nslookup -debug -norecurse collaborate.contoso.corp. 10.50.16.41
   # answers = 1（仅 CNAME）← 正常
   ```

2. **在 DNS 服务器上抓取网络跟踪**
   - 在 DNS 服务器上按以下步骤操作以避免缓存干扰：
     1. 启动网络跟踪
     2. 清除 DNS 服务器缓存
     3. 从客户端发起查询
     4. 停止跟踪
   - **发现**：DNS 服务器直接返回了 CNAME 和 A 记录，确认 A 记录来自 DNS 服务器本身
   - **额外发现**：
     - 返回的 A 记录 IP `10.50.70.59` 与 F5 上配置的 IP **不一致**
     - A 记录的 TTL **始终保持不变**，多次查询不递减。如果是从外部 DNS 缓存的动态记录，TTL 应随时间递减
   - **结论**：该 A 记录确实来自 Windows DNS 服务器本地，且**不是动态缓存**

3. **检查 DNS 控制台和 ADSI**
   - 在 DNS 管理控制台中检查 `mydesk.glb.contoso.corp` → **未找到任何 A 记录**
   - 在 ADSI Edit 中检查 `contoso.corp` 区域的 AD 复制分区 → **也未找到 mydesk 记录**
   - **结论**：该记录在常规管理工具中**不可见**

4. **分析 DNS Server ETL 日志**
   - 收集并分析 DNS 服务 ETL 日志
   - **发现**：日志显示该记录被标识为 **glue record（粘合记录）**
   ```
   lookupNodeForPacketInCache() - Returning glue node mydesk.glb.contoso.corp.
   with higher rank data than cache node for type 1
   ```
   - **结论**：DNS 服务器内部将该 A 记录作为 glue record 存储和返回

5. **通过 DNS 服务 Dump 确认**
   - 收集 DNS 服务内存 dump，使用 `!mex.dnss` 扩展搜索
   - **发现**：
     ```
     # 有问题的名称 - 在数据库和缓存中都存在
     !mex.dnss -s mydesk.glb.contoso.corp
     Database: mydesk.glb.contoso.corp
     Cache:    mydesk.glb.contoso.corp

     # 正常名称 - 数据库中不存在
     !mex.dnss -s collaborate.glb.contoso.corp
     Database: Not found.
     ```
   - **结论**：该 glue record 确实存储在 DNS 服务数据库中

6. **实验室复现行为**
   - 在测试环境中成功复现了该行为：
     1. 在区域中创建 `glb` 委派
     2. 在主区域中创建名为 `mydesk.glb` 的 A 记录
     3. 该记录被自动存储为 `glb` 委派下的 glue record
     4. 刷新 DNS 控制台后，该记录**消失**（不可见）
     5. 在 ADSI Edit 中可以看到（但客户环境中 ADSI 也看不到，原因未知）
     6. 使用 PowerShell 命令可以查询到

7. **确认客户环境并修复**
   - 建议客户使用 PowerShell 查询：
     ```powershell
     Get-DnsServerResourceRecord -ZoneName "contoso.corp" -Name "mydesk.glb"
     ```
   - **发现**：记录确实存在
   - 使用 PowerShell 删除该记录：
     ```powershell
     Remove-DnsServerResourceRecord -ZoneName "contoso.corp" -Name "mydesk.glb" -RRType "A" -RecordData "10.50.70.59"
     ```
   - **结果**：问题解决，非递归查询不再返回意外的 A 记录

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| 隐藏记录在 DNS 控制台中不可见 | 无法通过常规 GUI 工具定位问题记录 | 通过 DNS ETL 日志、服务 dump、PowerShell 命令多种手段交叉验证 |
| 隐藏记录在 ADSI Edit 中也不可见（客户环境） | 无法通过 AD 复制层面确认记录存在 | 通过实验室复现确认行为模式，使用 PowerShell `Get-DnsServerResourceRecord` 直接查询 |
| 不清楚记录是缓存还是静态配置 | 排查方向不确定 | 通过观察 TTL 不递减的特征，结合 ETL 日志中 "glue record" 的标识确认为静态记录 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

在 Windows DNS 服务器上，当区域中已存在一个 **委派（delegation）**（如 `glb.contoso.corp`），如果管理员在该区域中手动创建了一条名称包含委派前缀的记录（如创建名为 `mydesk.glb` 的 A 记录），该记录会被 DNS 服务器存储为委派下的 **glue record**。

Glue record 的特殊行为：
- 在 DNS 管理控制台中**刷新后不可见**
- 在某些情况下 ADSI Edit 中也不可见
- 但 DNS 服务内部数据库中**实际存在**，会在查询时被返回
- 该记录在非递归查询中也会被附加返回，导致客户端收到意外的 A 记录

在本案例中，该 glue record 中存储的 IP 地址 `10.50.70.59` 是过时/错误的，与 F5 DNS 上的实际配置不同，导致客户端解析到错误的 IP。

### Resolution

使用 PowerShell 命令定位并删除该隐藏的 glue record：

1. **查询隐藏记录**：
   ```powershell
   Get-DnsServerResourceRecord -ZoneName "contoso.corp" -Name "mydesk.glb"
   ```

2. **删除记录**：
   ```powershell
   Remove-DnsServerResourceRecord -ZoneName "contoso.corp" -Name "mydesk.glb" -RRType "A" -RecordData "10.50.70.59"
   ```

3. **验证修复**：
   ```powershell
   # 非递归查询应该只返回 CNAME
   nslookup -debug -norecurse mydesk.contoso.corp. <DNS_Server_IP>
   ```

### Workaround

无需 workaround，直接删除记录即可彻底解决。

## 6. 经验教训 (Lessons Learned)

- **技术知识**：
  - 当在 DNS 区域中创建名称格式为 `<name>.<delegation>` 的记录时，该记录会被存储为委派下的 glue record
  - Glue record 在 DNS 管理控制台中刷新后不可见，但 PowerShell `Get-DnsServerResourceRecord` 可以查询到
  - Glue record 的 TTL 不会随时间递减（因为它是本地静态记录，而非缓存）
  - DNS ETL 日志中的 `lookupNodeForPacketInCache() - Returning glue node` 消息可用于识别 glue record 问题

- **排查方法**：
  - 当 DNS 服务器返回意外记录时，用**非递归 vs 递归查询对比法**是快速定位问题来源的有效方法
  - 抓包时的正确顺序：先启动跟踪 → 清缓存 → 查询 → 停止跟踪，确保不受缓存干扰
  - 当 GUI 工具看不到记录时，不要放弃。使用 ETL 日志、DNS dump（`!mex.dnss`）、PowerShell 等多种手段交叉验证
  - TTL 不递减是判断记录来源（本地 vs 缓存）的关键信号

- **预防措施**：
  - 在有委派的 DNS 区域中创建记录时，避免使用 `<name>.<delegation>` 格式的名称
  - 如果怀疑存在隐藏 glue record，使用以下命令检查：
    ```powershell
    Get-DnsServerResourceRecord -ZoneName "<zone>" -Name "<name>.<delegation>"
    ```

## 7. 参考文档 (References)

- [Remove-DnsServerResourceRecord](https://learn.microsoft.com/en-us/powershell/module/dnsserver/remove-dnsserverresourcerecord) — PowerShell 命令，用于删除 DNS 服务器资源记录
- [Get-DnsServerResourceRecord](https://learn.microsoft.com/en-us/powershell/module/dnsserver/get-dnsserverresourcerecord) — PowerShell 命令，用于查询 DNS 服务器资源记录
- [Troubleshoot DNS servers](https://learn.microsoft.com/en-us/windows-server/networking/dns/troubleshoot/troubleshoot-dns-server) — DNS 服务器故障排查通用指南
- [Troubleshoot DNS data collection](https://learn.microsoft.com/en-us/windows-server/networking/dns/troubleshoot/troubleshoot-dns-data-collection) — DNS 数据收集与网络跟踪方法

---

# Case Summary: DNS Server Returns Unexpected A Record Due to Hidden Glue Record

**Product/Service:** Windows DNS Server

---

## 1. Symptoms

- The customer reported that the Windows DNS server returned an **unexpected A record** for the DNS name `mydesk.contoso.corp`
- In **non-recursive query (norecurse)** mode, the DNS server returned both a CNAME and an A record (IP: `10.50.70.59`). Expected behavior in non-recursive mode is to return only locally owned records (i.e., only the CNAME)
- A comparison with the working DNS name `collaborate.contoso.corp` confirmed that non-recursive queries correctly returned only the CNAME
- The A record IP `10.50.70.59` did **not match** the IP configured on the F5 DNS server — it was an **incorrect IP**

## 2. Background / Environment

- **DNS Architecture**:
  - Windows DNS Server (`YOURDC01.contoso.corp`, IP: `10.50.16.41`) is authoritative for the `contoso.corp` zone
  - `glb.contoso.corp` is a **delegation** under the `contoso.corp` zone, pointing to F5 DNS servers
  - F5 DNS servers host A records for the `glb.contoso.corp` subdomain
- **DNS Record Configuration**:
  - Windows DNS Server has a CNAME: `mydesk.contoso.corp` → `mydesk.glb.contoso.corp`
  - F5 DNS has an A record for `mydesk.glb.contoso.corp`
- **Client**: Windows client

## 3. Investigation & Troubleshooting

1. **Compared non-recursive vs recursive query behavior**
   - Performed non-recursive and recursive queries for both the problematic name `mydesk.contoso.corp` and the working name `collaborate.contoso.corp`
   - **Finding**:
     - `mydesk.contoso.corp`: non-recursive returned CNAME + A (2 answers) — **abnormal**
     - `collaborate.contoso.corp`: non-recursive returned only CNAME (1 answer) — **normal**
   - **Conclusion**: The issue only affects `mydesk.contoso.corp`; the DNS server incorrectly returns an A record in non-recursive mode

   ```
   # Problematic name - norecurse (should not return A, but did)
   nslookup -debug -norecurse mydesk.contoso.corp. 10.50.16.41
   # answers = 2 (CNAME + A) ← abnormal

   # Working name - norecurse (returns only CNAME, as expected)
   nslookup -debug -norecurse collaborate.contoso.corp. 10.50.16.41
   # answers = 1 (CNAME only) ← normal
   ```

2. **Captured network trace on DNS server**
   - Followed this procedure on the DNS server to avoid cache interference:
     1. Start network trace
     2. Clear DNS server cache
     3. Issue query from client
     4. Stop trace
   - **Finding**: DNS server directly returned CNAME and A records, confirming the A record originated from the DNS server itself
   - **Additional findings**:
     - The A record IP `10.50.70.59` **did not match** the IP configured on F5
     - The A record TTL **remained constant** across multiple queries — it did not decrease. Dynamically cached records from external DNS servers should show decreasing TTL
   - **Conclusion**: The A record is definitely from the local Windows DNS server and is **not a dynamic cache entry**

3. **Checked DNS Console and ADSI**
   - Checked DNS Management Console for `mydesk.glb.contoso.corp` → **no A record found**
   - Checked ADSI Edit for the `contoso.corp` zone AD replication partition → **no mydesk record found**
   - **Conclusion**: The record is **invisible** through standard management tools

4. **Analyzed DNS Server ETL logs**
   - Collected and analyzed DNS service ETL logs
   - **Finding**: The log identified the record as a **glue record**
   ```
   lookupNodeForPacketInCache() - Returning glue node mydesk.glb.contoso.corp.
   with higher rank data than cache node for type 1
   ```
   - **Conclusion**: The DNS server internally stores and returns this A record as a glue record

5. **Confirmed via DNS Service Dump**
   - Collected a DNS service memory dump and searched using the `!mex.dnss` extension
   - **Finding**:
     ```
     # Problematic name - exists in both database and cache
     !mex.dnss -s mydesk.glb.contoso.corp
     Database: mydesk.glb.contoso.corp
     Cache:    mydesk.glb.contoso.corp

     # Working name - not found in database
     !mex.dnss -s collaborate.glb.contoso.corp
     Database: Not found.
     ```
   - **Conclusion**: The glue record is indeed stored in the DNS service database

6. **Reproduced in lab environment**
   - Successfully reproduced the behavior in a test environment:
     1. Created a `glb` delegation in the zone
     2. Created an A record named `mydesk.glb` in the primary zone
     3. The record was automatically stored as a glue record under the `glb` delegation
     4. After refreshing the DNS console, the record **disappeared** (invisible)
     5. Visible in ADSI Edit (but not in the customer's environment — reason unknown)
     6. Queryable via PowerShell cmdlets

7. **Confirmed in customer environment and remediated**
   - Advised the customer to query using PowerShell:
     ```powershell
     Get-DnsServerResourceRecord -ZoneName "contoso.corp" -Name "mydesk.glb"
     ```
   - **Finding**: The record was confirmed to exist
   - Removed the record using PowerShell:
     ```powershell
     Remove-DnsServerResourceRecord -ZoneName "contoso.corp" -Name "mydesk.glb" -RRType "A" -RecordData "10.50.70.59"
     ```
   - **Result**: Issue resolved — non-recursive queries no longer return the unexpected A record

## 4. Blockers & How They Were Resolved

| Blocker | Impact | Resolution |
|---------|--------|------------|
| Hidden record not visible in DNS Console | Could not locate the problematic record via standard GUI tools | Used DNS ETL logs, service dump, and PowerShell cmdlets for cross-verification |
| Hidden record not visible in ADSI Edit (customer environment) | Could not confirm record existence at AD replication level | Reproduced in lab to confirm behavior pattern; used PowerShell `Get-DnsServerResourceRecord` to directly query |
| Unclear whether the record was a cache entry or static config | Investigation direction was uncertain | Identified by observing non-decreasing TTL and the "glue record" label in ETL logs |

## 5. Root Cause & Resolution

### Root Cause

On a Windows DNS server, when a **delegation** already exists in a zone (e.g., `glb.contoso.corp`), if an administrator manually creates a record with a name that includes the delegation prefix (e.g., an A record named `mydesk.glb`), the DNS server stores it as a **glue record** under that delegation.

Glue record special behavior:
- **Invisible** in the DNS Management Console after refresh
- In some cases, also invisible in ADSI Edit
- **Actually exists** in the DNS service internal database and is returned during queries
- Attached to responses even in non-recursive queries, causing clients to receive unexpected A records

In this case, the glue record contained a stale/incorrect IP address `10.50.70.59` that differed from the actual F5 DNS configuration, causing clients to resolve to the wrong IP.

### Resolution

Locate and delete the hidden glue record using PowerShell:

1. **Query the hidden record**:
   ```powershell
   Get-DnsServerResourceRecord -ZoneName "contoso.corp" -Name "mydesk.glb"
   ```

2. **Delete the record**:
   ```powershell
   Remove-DnsServerResourceRecord -ZoneName "contoso.corp" -Name "mydesk.glb" -RRType "A" -RecordData "10.50.70.59"
   ```

3. **Verify the fix**:
   ```powershell
   # Non-recursive query should now return only CNAME
   nslookup -debug -norecurse mydesk.contoso.corp. <DNS_Server_IP>
   ```

### Workaround

No workaround needed — directly deleting the record fully resolves the issue.

## 6. Lessons Learned

- **Technical Knowledge**:
  - When a record is created in a DNS zone with a name in the format `<name>.<delegation>`, it gets stored as a glue record under the delegation
  - Glue records are invisible in DNS Management Console after refresh, but queryable via PowerShell `Get-DnsServerResourceRecord`
  - Glue record TTL does not decrease over time (because it is a local static record, not a cache entry)
  - The DNS ETL log message `lookupNodeForPacketInCache() - Returning glue node` is the key indicator for identifying glue record issues

- **Troubleshooting Methods**:
  - When a DNS server returns unexpected records, the **non-recursive vs recursive query comparison** is an effective technique to quickly identify the source
  - Correct packet capture sequence: Start trace → Clear cache → Query → Stop trace — to ensure no cache interference
  - When GUI tools fail to show the record, don't give up. Use ETL logs, DNS dump (`!mex.dnss`), and PowerShell for cross-verification
  - Non-decreasing TTL is a key signal to determine whether a record is local vs cached

- **Prevention**:
  - Avoid creating records with the `<name>.<delegation>` naming format in zones that have delegations
  - If hidden glue records are suspected, check using:
    ```powershell
    Get-DnsServerResourceRecord -ZoneName "<zone>" -Name "<name>.<delegation>"
    ```

## 7. References

- [Remove-DnsServerResourceRecord](https://learn.microsoft.com/en-us/powershell/module/dnsserver/remove-dnsserverresourcerecord) — PowerShell cmdlet to remove DNS server resource records
- [Get-DnsServerResourceRecord](https://learn.microsoft.com/en-us/powershell/module/dnsserver/get-dnsserverresourcerecord) — PowerShell cmdlet to query DNS server resource records
- [Troubleshoot DNS servers](https://learn.microsoft.com/en-us/windows-server/networking/dns/troubleshoot/troubleshoot-dns-server) — General DNS server troubleshooting guide
- [Troubleshoot DNS data collection](https://learn.microsoft.com/en-us/windows-server/networking/dns/troubleshoot/troubleshoot-dns-data-collection) — DNS data collection and network tracing methods
