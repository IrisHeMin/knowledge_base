---
layout: post
title: "DNS Ping 解析失败 — CNAME 记录中隐藏的前导空格导致 DNS 客户端拒绝响应"
date: 2025-11-24
categories: [DNS, Windows-Client]
tags: [dns, ping, nslookup, resolve-dnsname, cname, dnscache, etl, name-validation, error-invalid-data]

---

# Case Summary: DNS Ping 解析失败 — CNAME 记录中隐藏的前导空格导致 DNS 客户端服务拒绝响应

**Product/Service:** Windows 11 25H2 — DNS Client Service (dnscache)

---

## 1. 症状 (Symptoms)

- 客户报告在 Windows 11 25H2 机器上，使用 `ping` 命令解析特定域名（如 `newsletter.contoso-marketing.example.com`）时失败，报错 **"could not find host"**。
- **关键差异**：
  - `nslookup newsletter.contoso-marketing.example.com` → **可以正常解析**
  - `Resolve-DnsName newsletter.contoso-marketing.example.com` → **可以正常解析**
  - `ping newsletter.contoso-marketing.example.com` → **失败**
- 问题仅影响特定域名，其他域名解析正常。

## 2. 背景 (Background / Environment)

- **操作系统**：Windows 11 25H2
- **问题域名**：`newsletter.contoso-marketing.example.com`（该域名配有 CNAME 记录）
- **DNS 解析路径差异**：
  - `ping.exe` 依赖 DNS Client Service（`dnscache`）进行名称解析
  - `nslookup` 绕过 DNS Client Service，直接向 DNS Server 发起查询
  - `Resolve-DnsName` 虽然也使用 DNS API，但在某些场景下的处理路径与 `ping` 不完全相同

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

### 第一轮：确认 DNS Server 侧是否正常

1. **做了什么**：分别使用 `nslookup` 和 `Resolve-DnsName` 命令解析问题域名。
2. **发现了什么**：两个命令都能成功返回 IP 地址，DNS Server 响应正常。
3. **得出了什么结论**：DNS Server 侧没有问题，解析记录存在且可返回。问题出在客户端侧的 DNS 解析路径上。

### 第二轮：网络抓包确认 DNS 响应

1. **做了什么**：在执行 `ping` 的同时抓取网络流量，分析 DNS 请求和响应报文。
2. **发现了什么**：DNS Server 正常返回了 IP 地址给客户端，报文中包含完整的 DNS 响应。
3. **得出了什么结论**：DNS 响应已到达客户端，但 DNS Client Service 未能正确处理该响应。问题聚焦到 **DNS Client Service（dnscache）为何拒绝有效的 DNS 响应**。

### 第三轮：DNS Client ETL 日志分析 — 发现 ERROR_INVALID_DATA

1. **做了什么**：启用 DNS Client Service ETL 追踪，捕获问题时间段的详细日志。
2. **发现了什么**：ETL 日志中出现明确的错误：

   ```
   [Microsoft-Windows-DNS Client Events/Operational]
   Query response for name newsletter.contoso-marketing.example.com,
   type 1, interface index 0 and network index 0
   returned 13 with results , client PID 3716
   ```

   ```
   [DNSAPI] Query_ProcessCacheResults() -
   Query name newsletter.contoso-marketing.example.com,
   wType 1, status 13(ERROR_INVALID_DATA),
   pResults 0000000000000000
   ```

   **关键发现** — DNS Client Service 返回了 **错误码 13（ERROR_INVALID_DATA）**，且 `pResults` 为空指针，说明解析结果被完全丢弃。

3. **得出了什么结论**：DNS Client Service 收到了 DNS 响应，但在处理过程中判定数据无效并拒绝了。需要进一步深入分析为什么数据被判定为 "invalid"。

### 第四轮：深挖 ETL — 发现名称验证失败的真正原因

1. **做了什么**：继续分析 ETL 日志中更底层的 DNSRPCLIB 调用栈，关注名称验证逻辑。
2. **发现了什么**：发现了三条关键的错误日志：

   ```
   [DNSRPCLIB] IsNameValid_New() -
   rrread.c:IsNameValid_New:384
   ' ' (20) is invalid, rejecting " contoso-marketing"
   ```

   ```
   [DNSRPCLIB] Dns_ReadPacketNameImpl_New() -
   ERROR: Invalid name " contoso-marketing"
   ```

   ```
   [DNSRPCLIB] Ptr_RecordRead_New() -
   ERROR: bad packet name, name validation = 0x00000001(TRUE)
   ```

   **极关键的细节**：在错误日志 `rejecting " contoso-marketing"` 中，引号和域名之间有一个**不易察觉的空格**。字符 `' '` 的 ASCII 值为 `0x20`（空格），正是这个**前导空格**导致 `IsNameValid_New()` 函数判定域名无效。

3. **得出了什么结论**：DNS Client Service 的名称验证函数发现域名标签（label）中包含非法字符（空格 `0x20`），按照 DNS 协议标准（RFC 1035）拒绝了该名称。问题指向 DNS 记录本身可能包含隐藏的前导空格。

### 第五轮：回溯验证 — 网络抓包和 Resolve-DnsName 确认空格

1. **做了什么**：
   - 重新检查网络抓包中 DNS 响应的 CNAME 记录详细内容
   - 使用 `Resolve-DnsName` 查看解析结果中的 CNAME 值
2. **发现了什么**：
   - 网络抓包中，CNAME 记录的目标域名前有**额外的 dot（点）**，这些 dot 实际是空格字符在抓包工具中的显示形式
   - `Resolve-DnsName` 的输出中也能看到域名前有一个空格
   - 在 DNS Server 管理界面中，该空格**不容易被肉眼发现**
3. **得出了什么结论**：CNAME 记录的目标域名值中确实包含一个**前导空格**（即记录值为 `" contoso-marketing.example.com"` 而非 `"contoso-marketing.example.com"`）。

### 第六轮：修复验证

1. **做了什么**：在 DNS Server 上编辑 CNAME 记录，删除目标域名前的空格字符。
2. **发现了什么**：修改后，`ping newsletter.contoso-marketing.example.com` **立即恢复正常**，可以成功解析并 ping 通。
3. **得出了什么结论**：问题确认为 CNAME 记录中的前导空格导致。修复后问题完全解决。

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| `nslookup` 和 `Resolve-DnsName` 正常但 `ping` 失败，表象矛盾导致初始方向不清 | 需要理解三种工具的底层解析路径差异才能定位问题 | 通过网络抓包 + DNS Client ETL 日志组合分析，确认问题在 DNS Client Service 层 |
| ETL 日志中的前导空格极难发现（引号和域名之间只差一个空格字符） | 如果不仔细比对日志，容易忽略这个关键细节 | 逐字符比对 `IsNameValid_New()` 日志中 rejecting 的域名字符串，注意到 `' ' (20)` 的提示 |
| DNS Server 管理界面不直观显示空格字符 | 无法直接在 Server UI 上确认空格是否存在 | 通过编辑 CNAME 记录并重新保存来确认并清除隐藏字符 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

**CNAME 记录的目标域名中包含一个隐藏的前导空格字符（ASCII 0x20）。**

这导致：
1. DNS Server 正常返回包含该 CNAME 的 DNS 响应（Server 侧不做严格的名称字符校验）
2. DNS Client Service（`dnscache`）在解析响应时，调用 `IsNameValid_New()` 函数验证域名标签
3. 该函数发现标签中包含空格字符（`0x20`），判定为**非法 DNS 名称**（违反 RFC 1035 域名语法规则）
4. 整条 DNS 响应被标记为 `ERROR_INVALID_DATA`（错误码 13），解析结果被丢弃
5. `ping.exe` 依赖 DNS Client Service，因此收到解析失败的结果，报错 "could not find host"
6. `nslookup` 绕过 DNS Client Service 直接查询 DNS Server，因此不受影响
7. `Resolve-DnsName` 虽然也使用 DNS API，但在处理链中对某些情况有不同的容错行为

### Resolution

1. 在 DNS Server 上定位到问题 CNAME 记录（`newsletter.contoso-marketing.example.com` 的 CNAME 目标）
2. 编辑该 CNAME 记录，**删除目标域名前的前导空格**
3. 保存后验证 `ping` 恢复正常

### Workaround

如果无法立即修改 DNS 记录（例如由第三方管理），可临时通过以下方式绕过：
- 在本地 `hosts` 文件中添加静态解析条目
- 使用 `nslookup` 获取 IP 地址后直接使用 IP 访问

## 6. 经验教训 (Lessons Learned)

- **技术知识**：
  - **`ping` vs `nslookup` vs `Resolve-DnsName` 的解析路径差异**：`ping.exe` 依赖 DNS Client Service（`dnscache`）；`nslookup` 完全绕过 `dnscache`，直接构造 DNS 查询发送到 DNS Server；`Resolve-DnsName` 使用 DNS API 但在某些场景下有不同的容错处理。因此三者表现不一致时，问题大概率出在 DNS Client Service 层。
  - **DNS Client Service 的名称验证（Name Validation）机制**：Windows DNS Client Service 在解析 DNS 响应时会调用 `IsNameValid_New()` 对域名标签进行严格校验。任何非法字符（包括空格、控制字符等）都会导致整条响应被拒绝，返回 `ERROR_INVALID_DATA`（错误码 13）。
  - **隐藏字符在 DNS 记录中的危害**：空格、制表符等不可见字符可能在 DNS 记录创建/编辑过程中被意外引入（如从文档复制粘贴域名时），DNS Server 管理界面不一定能直观显示这些字符。

- **排查方法**：
  - 当 `ping` 解析失败但 `nslookup` 正常时，应立即怀疑 DNS Client Service 层的问题，优先使用 **DNS Client ETL 日志**（`Microsoft-Windows-DNS Client Events`）进行排查。
  - 在 ETL 日志中关注 **`IsNameValid_New()`** 和 **`ERROR_INVALID_DATA`** 关键字，它们通常指向 DNS 记录中的非法字符问题。
  - 三重验证法：**DNS Client ETL**（内部处理逻辑）+ **网络抓包**（实际响应内容）+ **`Resolve-DnsName` 输出**（对比分析），三者结合定位隐藏字符。

- **预防措施**：
  - 创建或修改 DNS 记录时，**避免从文档、网页、邮件中直接复制粘贴域名**，以防引入不可见字符。
  - 如果必须复制粘贴，建议先粘贴到纯文本编辑器中确认无隐藏字符，或使用 `Trim()` 等方法清理首尾空白。
  - DNS 管理工具应考虑增加对输入值的前后空白字符校验和告警。

## 7. 参考文档 (References)

- [DNS Overview — Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/dns/dns-top) — Windows Server DNS 服务概述
- [RFC 1035 — Domain Names: Implementation and Specification](https://datatracker.ietf.org/doc/html/rfc1035) — DNS 域名语法规则定义（Section 2.3.1 定义了合法的域名字符集）
- [DNS Client Resolution Timeouts — Microsoft Troubleshoot](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/dns-client-resolution-timeouts) — DNS 客户端解析超时行为说明

---
---

# Case Summary: DNS Ping Resolution Failure — Hidden Leading Space in CNAME Record Causes DNS Client Service to Reject Response

**Product/Service:** Windows 11 25H2 — DNS Client Service (dnscache)

---

## 1. Symptoms

- The customer reported that `ping` to a specific hostname (e.g., `newsletter.contoso-marketing.example.com`) failed on a Windows 11 25H2 machine with the error **"could not find host"**.
- **Key discrepancy**:
  - `nslookup newsletter.contoso-marketing.example.com` → **resolved successfully**
  - `Resolve-DnsName newsletter.contoso-marketing.example.com` → **resolved successfully**
  - `ping newsletter.contoso-marketing.example.com` → **failed**
- The issue only affected this specific hostname; other names resolved normally.

## 2. Background / Environment

- **Operating System**: Windows 11 25H2
- **Affected Hostname**: `newsletter.contoso-marketing.example.com` (has a CNAME record)
- **DNS Resolution Path Differences**:
  - `ping.exe` relies on the DNS Client Service (`dnscache`) for name resolution
  - `nslookup` bypasses the DNS Client Service and queries the DNS Server directly
  - `Resolve-DnsName` also uses the DNS API but may have different fault-tolerance behavior in certain scenarios

## 3. Investigation & Troubleshooting

### Round 1: Verify DNS Server-Side Resolution

1. **Action**: Used `nslookup` and `Resolve-DnsName` to resolve the problem hostname.
2. **Finding**: Both commands successfully returned the IP address — the DNS Server responded correctly.
3. **Conclusion**: The DNS Server was functioning normally with valid records. The problem was on the client-side DNS resolution path.

### Round 2: Network Capture Confirms DNS Response Delivery

1. **Action**: Captured network traffic while executing `ping` and analyzed the DNS request/response packets.
2. **Finding**: The DNS Server returned the IP address to the client successfully — the response packet was complete and correct.
3. **Conclusion**: The DNS response reached the client, but the DNS Client Service failed to process it properly. Investigation shifted to **why the DNS Client Service (dnscache) was rejecting a valid DNS response**.

### Round 3: DNS Client ETL Analysis — ERROR_INVALID_DATA Discovered

1. **Action**: Enabled DNS Client Service ETL tracing and captured detailed logs during the problem window.
2. **Finding**: The ETL log showed a clear error:

   ```
   [Microsoft-Windows-DNS Client Events/Operational]
   Query response for name newsletter.contoso-marketing.example.com,
   type 1, interface index 0 and network index 0
   returned 13 with results , client PID 3716
   ```

   ```
   [DNSAPI] Query_ProcessCacheResults() -
   Query name newsletter.contoso-marketing.example.com,
   wType 1, status 13(ERROR_INVALID_DATA),
   pResults 0000000000000000
   ```

   **Key finding** — The DNS Client Service returned **error code 13 (ERROR_INVALID_DATA)** with a null `pResults` pointer, meaning the resolution result was completely discarded.

3. **Conclusion**: The DNS Client Service received the DNS response but deemed the data invalid and rejected it. Deeper analysis was needed to understand why.

### Round 4: Deep Dive into ETL — Name Validation Failure Root Cause

1. **Action**: Analyzed lower-level DNSRPCLIB function calls in the ETL trace, focusing on name validation logic.
2. **Finding**: Three critical error log entries:

   ```
   [DNSRPCLIB] IsNameValid_New() -
   rrread.c:IsNameValid_New:384
   ' ' (20) is invalid, rejecting " contoso-marketing"
   ```

   ```
   [DNSRPCLIB] Dns_ReadPacketNameImpl_New() -
   ERROR: Invalid name " contoso-marketing"
   ```

   ```
   [DNSRPCLIB] Ptr_RecordRead_New() -
   ERROR: bad packet name, name validation = 0x00000001(TRUE)
   ```

   **Critical detail**: In the log entry `rejecting " contoso-marketing"`, there was a **barely noticeable space between the quote and the domain name**. The character `' '` with ASCII value `0x20` (space) was the **leading space** that caused `IsNameValid_New()` to reject the name as invalid.

3. **Conclusion**: The DNS Client Service's name validation function detected an illegal character (space `0x20`) in a DNS name label, and per DNS protocol standards (RFC 1035), rejected the name. This pointed to the DNS record itself containing a hidden leading space.

### Round 5: Cross-Validation — Network Capture and Resolve-DnsName Confirm the Space

1. **Action**:
   - Re-examined the network capture, inspecting the CNAME record details in the DNS response
   - Reviewed `Resolve-DnsName` output to check the CNAME target value
2. **Finding**:
   - In the network capture, the CNAME target showed **extra dots before the name**, which were the space character rendered differently by the capture tool
   - `Resolve-DnsName` output also revealed a space before the domain name
   - On the DNS Server management UI, the space was **not easily visible to the naked eye**
3. **Conclusion**: The CNAME record's target value indeed contained a **leading space** (i.e., the value was `" contoso-marketing.example.com"` instead of `"contoso-marketing.example.com"`).

### Round 6: Fix and Verification

1. **Action**: Edited the CNAME record on the DNS Server, removing the leading space character from the target hostname.
2. **Finding**: After the fix, `ping newsletter.contoso-marketing.example.com` **immediately succeeded** — resolution and connectivity were fully restored.
3. **Conclusion**: Root cause confirmed as the leading space in the CNAME record. Issue fully resolved.

## 4. Blockers & How They Were Resolved

| Blocker | Impact | How It Was Resolved |
|---------|--------|---------------------|
| `nslookup` and `Resolve-DnsName` worked but `ping` failed — contradictory symptoms confused initial investigation direction | Required understanding the differences in resolution paths between the three tools to correctly scope the problem | Combined network capture + DNS Client ETL logs to confirm the issue was at the DNS Client Service layer |
| The leading space in ETL logs was extremely hard to spot (only one space character between the quote and the domain name) | Easy to overlook this critical detail if logs were read casually | Carefully compared the `IsNameValid_New()` rejection string character by character; the `' ' (20)` hint was the key giveaway |
| DNS Server management UI did not visually display the space character | Could not directly confirm the space's existence from the Server UI | Edited and re-saved the CNAME record to remove the hidden character; verified fix through `ping` |

## 5. Root Cause & Resolution

### Root Cause

**The CNAME record's target hostname contained a hidden leading space character (ASCII 0x20).**

This caused the following chain of events:
1. The DNS Server returned the DNS response containing the CNAME normally (the server does not perform strict name character validation on stored records)
2. The DNS Client Service (`dnscache`) on the client parsed the response and called `IsNameValid_New()` to validate the domain name labels
3. The function detected a space character (`0x20`) in the label, deeming it an **illegal DNS name** (violating RFC 1035 domain name syntax rules)
4. The entire DNS response was marked as `ERROR_INVALID_DATA` (error code 13), and the resolution result was discarded
5. `ping.exe`, which relies on the DNS Client Service, received a resolution failure and reported "could not find host"
6. `nslookup` bypasses the DNS Client Service and queries the DNS Server directly, so it was unaffected
7. `Resolve-DnsName` also uses the DNS API but has different fault-tolerance behavior in certain processing paths

### Resolution

1. Located the problematic CNAME record on the DNS Server (`newsletter.contoso-marketing.example.com`'s CNAME target)
2. Edited the CNAME record, **removing the leading space** from the target hostname
3. Verified that `ping` resolved successfully after the fix

### Workaround

If the DNS record cannot be modified immediately (e.g., managed by a third party):
- Add a static entry in the local `hosts` file
- Use `nslookup` to obtain the IP address and connect via IP directly

## 6. Lessons Learned

- **Technical Knowledge**:
  - **Resolution path differences: `ping` vs `nslookup` vs `Resolve-DnsName`**: `ping.exe` depends on the DNS Client Service (`dnscache`); `nslookup` completely bypasses `dnscache` and sends DNS queries directly to the DNS Server; `Resolve-DnsName` uses the DNS API but may have different fault-tolerance handling. When these three tools show inconsistent results, the problem almost certainly lies at the DNS Client Service layer.
  - **DNS Client Service Name Validation mechanism**: The Windows DNS Client Service calls `IsNameValid_New()` to strictly validate domain name labels when parsing DNS responses. Any illegal character (including spaces, control characters, etc.) causes the entire response to be rejected with `ERROR_INVALID_DATA` (error code 13).
  - **Hidden characters in DNS records**: Invisible characters like spaces and tabs can be accidentally introduced during DNS record creation/editing (e.g., copy-pasting domain names from documents), and DNS Server management UIs may not visually display them.

- **Troubleshooting Methodology**:
  - When `ping` fails to resolve but `nslookup` succeeds, immediately suspect an issue at the DNS Client Service layer. Prioritize **DNS Client ETL logs** (`Microsoft-Windows-DNS Client Events`) for investigation.
  - In ETL logs, look for **`IsNameValid_New()`** and **`ERROR_INVALID_DATA`** keywords — they typically point to illegal character issues in DNS records.
  - Triple validation: **DNS Client ETL** (internal processing logic) + **Network capture** (actual response content) + **`Resolve-DnsName` output** (comparative analysis) — combine all three to locate hidden characters.

- **Prevention**:
  - When creating or modifying DNS records, **avoid directly copy-pasting domain names from documents, web pages, or emails** to prevent introducing invisible characters.
  - If copy-pasting is necessary, paste into a plain text editor first to verify there are no hidden characters, or use methods like `Trim()` to clean leading/trailing whitespace.
  - DNS management tools should consider adding validation and warnings for leading/trailing whitespace in input values.

## 7. References

- [DNS Overview — Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/dns/dns-top) — Windows Server DNS service overview
- [RFC 1035 — Domain Names: Implementation and Specification](https://datatracker.ietf.org/doc/html/rfc1035) — DNS domain name syntax rules (Section 2.3.1 defines the legal character set for domain names)
- [DNS Client Resolution Timeouts — Microsoft Troubleshoot](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/dns-client-resolution-timeouts) — DNS client resolution timeout behavior documentation
