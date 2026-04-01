---
layout: post
title: "DNS Conditional Forwarder CNAME 跨域解析失败 - Secure Cache Against Pollution 导致 A 记录丢弃"
date: 2026-04-01
categories: [DNS, Windows-Server]
tags: [dns, conditional-forwarder, cname, secure-cache, pollution-protection, forwarder, debug-log]

---

# Case Summary: DNS Conditional Forwarder CNAME 跨域解析失败

**Product/Service:** Windows DNS Server / Conditional Forwarder

---

## 1. 症状 (Symptoms)

- 客户端在访问阿里云某些服务（如 `lingma-api.tongyi.example-cloud.com`）时，**间歇性出现 DNS 解析超时**
- 使用 `nslookup` 测试时，DNS Server 返回 `NOERROR`，但响应中**只有 CNAME 记录，没有最终的 A 记录（IP 地址）**：
  ```
  > server 10.0.1.71
  > lingma-api.tongyi.example-cloud.com
  Non-authoritative answer:
  Name:    lingma-api.tongyi.example-cloud.com
  ```
  （无 IP 地址返回）
- 阿里云工程师在其 DNS Server 端抓包确认：**DNS 响应包中完整包含了 CNAME 和 A 记录**，但客户端始终只拿到 CNAME
- 问题**时有时无**：大部分时间解析正常，但偶尔出现持续数分钟的解析失败

---

## 2. 背景 (Background / Environment)

### 架构

```
客户端 → Windows DNS Server（3 台） → Conditional Forwarder → 阿里云 DNS Server（4 台）
```

### 配置

- Windows DNS Server 配置了 `example-cloud.com` 的 **Conditional Forwarder**，指向 4 台阿里云 DNS（如 `10.199.162.112` 等）
- Conditional Forwarder 列表中**第 3 台 DNS Server 不可达**，但不影响前 2 台正常工作
- **Secure Cache Against Pollution** 功能默认开启（DNS Manager → Advanced → "Secure cache against pollution" ✅）
- 未配置 DNS Policy
- 网络通信正常，排除了网络层问题

### 阿里云 DNS 返回的响应结构（Wireshark 抓包确认）

阿里云 DNS 返回了完整的 CNAME 链 + A 记录：

```
Answer Section:
① CNAME: lingma-api.tongyi.example-cloud.com
         → lingma-api.tongyi.example-cloud.com.gds.example-alibabadns.com    TTL=190s

② CNAME: lingma-api.tongyi.example-cloud.com.gds.example-alibabadns.com
         → ga-xxxx.example-aliyunga0019.com                                  TTL=80s

③ A:     ga-xxxx.example-aliyunga0019.com → 47.57.xxx.171                   TTL=194s
④ A:     ga-xxxx.example-aliyunga0019.com → 47.57.xxx.142                   TTL=194s
```

关键特征：**CNAME 链跨越了 3 个不同的域名**：
- `example-cloud.com`（Conditional Forwarder 覆盖范围）
- `example-alibabadns.com`（不在 Conditional Forwarder 范围内）
- `example-aliyunga0019.com`（不在 Conditional Forwarder 范围内）

---

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

### Step 1：确认 DNS 基本配置

**做了什么**：检查 Conditional Forwarder 配置和 DNS Zone

```powershell
Get-DnsServerZone | Where-Object {$_.ZoneName -like "*example-cloud*"}
# ZoneType = Forwarder（即 Conditional Forwarder）

Get-DnsServerQueryResolutionPolicy
# 无 DNS Policy 配置
```

**发现**：只有一个 Conditional Forwarder Zone，无冲突 Zone，无 DNS Policy。

**结论**：基本配置排除了 Zone 冲突和 Policy 过滤的可能。

---

### Step 2：确认 Secure Cache Against Pollution 状态

**做了什么**：检查注册表和 DNS 管理控制台

```powershell
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\DNS\Parameters" | Select SecureResponses
# 返回空 — 注册表键不存在，使用默认值
```

**发现**：`SecureResponses` 键不存在 = 使用默认值。根据微软文档（KB241352），Windows 2000 SP3 及以后版本**默认开启** Secure Cache Against Pollution。客户在 DNS Manager → Advanced 确认 "Secure cache against pollution" **已勾选**。

**结论**：Secure Cache Against Pollution 处于开启状态。

---

### Step 3：开启 DNS Debug Log 并复现问题

**做了什么**：在 DNS Server 上开启 debug log，同时从客户端 `10.65.90.12` 执行 nslookup 复现

**nslookup 结果**：
```
> server 10.0.1.71
DNS request timed out.
    timeout was 2 seconds.

> lingma-api.tongyi.example-cloud.com
Non-authoritative answer:
Name:    lingma-api.tongyi.example-cloud.com
```
只返回了域名（CNAME），没有 A 记录。

---

### Step 4：分析 DNS Debug Log — 客户端查询

**做了什么**：在 debug log 中筛选客户端 IP `10.65.90.12` 的查询

**发现**：
```
206121 | 11:48:35 | Rcv 10.65.90.12 → Q  A    lingma-api.tongyi.example-cloud.com.CORP.BIZ
206123 | 11:48:35 | Snd 10.65.90.12 ← R  NXDOMAIN  ← DNS 后缀追加，失败

206217 | 11:48:35 | Rcv 10.65.90.12 → Q  A    lingma-api.tongyi.example-cloud.com
206219 | 11:48:35 | Snd 10.65.90.12 ← R  NOERROR   ← 从缓存直接返回
```

**结论**：客户端查询时，DNS Server **直接从缓存返回**，没有触发转发到阿里云。说明缓存中已经有了该域名的记录，但缓存内容只有 CNAME，没有 A 记录。

---

### Step 5：分析 DNS Debug Log — 转发到阿里云

**做了什么**：筛选 DNS Server 与阿里云 Conditional Forwarder (`10.199.162.112`) 之间的 `lingma-api` 相关流量

**发现**：3 次转发到阿里云，全部返回 NOERROR

```
26145 | 11:47:33 | Snd 10.199.162.112 → Q lingma-api.tongyi.example-cloud.com   ← 转发
26155 | 11:47:33 | Rcv 10.199.162.112 ← R NOERROR                                ← 阿里云返回成功
26157 | 11:47:33 | Snd 10.64.116.94   ← R NOERROR                                ← 返回给客户端

29515 | 11:47:34 | Snd 10.199.162.112 → Q（第 2 次转发）
29529 | 11:47:34 | Rcv 10.199.162.112 ← R NOERROR

59323 | 11:47:45 | Snd 10.199.162.112 → Q（第 3 次转发）
59343 | 11:47:45 | Rcv 10.199.162.112 ← R NOERROR
```

**结论**：阿里云每次都返回 NOERROR，但 DNS Server 返回给客户端的响应中只有 CNAME（从 nslookup 结果确认）。说明阿里云响应中的 A 记录在 DNS Server 内部被处理掉了。

---

### Step 6：分析 DNS Debug Log — 发现 CNAME Chase 行为（关键发现）

**做了什么**：搜索 `alibabadns` 关键字，查找 CNAME 追踪行为

**发现**：
```
118985 | 11:48:06 | Snd 10.111.125.34 → Q lingma-api.tongyi.example-cloud.com.gds.example-alibabadns.com
119193 | 11:48:06 | Rcv 10.111.125.34 ← R NOERROR
```

**关键分析**：
1. DNS Server 在 **11:48:06** 向**通用 Forwarder** (`10.111.125.34`) 发起了查询
2. 查询的域名是 `lingma-api.tongyi.example-cloud.com.gds.example-alibabadns.com` — 这是阿里云 CNAME 链中的第二跳域名
3. 这个查询发到的是**通用 Forwarder**，而不是阿里云 Conditional Forwarder (`10.199.162.112`)
4. 原因：`gds.example-alibabadns.com` 不在 `example-cloud.com` 的 Conditional Forwarder 管辖范围内

**结论**：DNS Server 收到阿里云的响应后，**只保留了在 `example-cloud.com` 域内的 CNAME 记录，丢弃了 `example-alibabadns.com` 和 `example-aliyunga0019.com` 域下的记录**（包括最终的 A 记录）。然后 DNS Server 试图自行解析 CNAME 目标，但走了错误的解析路径（通用 Forwarder 而非阿里云 DNS）。

---

### Step 7：分析 DNS Debug Log — 验证不只影响一个域名

**做了什么**：搜索所有 `alibabadns` 相关的 CNAME Chase 记录

**发现**：多个域名都存在同样的跨域 CNAME Chase 行为：

```
g.alicdn.com                          → g.alicdn.com.gds.example-alibabadns.com
bluedot.is.autonavi.com               → bluedot.is.autonavi.com.gds.example-alibabadns.com
tyjr-cn-shanghai-finance-1.example-cloud.com → *.vipgds.example-alibabadns.com
d-gm.mmstat.com                       → d-gm.mmstat.com.gds.example-alibabadns.com
```

**结论**：这是阿里云 DNS 架构的通用模式（Global DNS Service），不只影响 `lingma-api` 一个域名。

---

### Step 8：统计 debug log 中的事件分布

**做了什么**：统计 `lingma-api` 相关的所有事件类型

```
事件类型                          数量
CLIENT_QUERY（客户端查询）         752
CACHE_RSP_TO_CLIENT（缓存响应）    752
FORWARDER（转发到阿里云）            6
CNAME_CHASE（CNAME 追踪）           2
```

**结论**：在 debug log 覆盖的时间段内（约 2.5 分钟），752 次客户端查询全部从缓存返回。只有 3 次触发了转发到阿里云（6 行 = 3 次 Snd + 3 次 Rcv），1 次 CNAME Chase。这说明缓存中的 CNAME-only 记录在 TTL 期间（约 190 秒）影响了所有客户端。

---

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| DNS debug log 不记录 Answer Section 内容 | 无法直接看到 A 记录被丢弃的动作，只能通过间接证据推理 | 结合阿里云端 Wireshark 抓包（确认返回了 A）+ DNS Server debug log（确认 CNAME Chase 行为）+ Secure Cache 文档，形成完整的证据链 |
| 问题间歇性，不易复现 | 无法按需触发问题 | 提前开启 debug log 持续记录，等待问题自然发生后分析 |
| `SecureResponses` 注册表键不存在 | 初期误以为 Secure Cache 未配置 | 查阅文档确认：键不存在 = 使用默认值（开启）。通过 DNS Manager GUI 确认了开启状态 |

---

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

问题由以下三个因素的组合导致：

1. **阿里云 DNS 返回的 CNAME 链跨越多个域名**：
   ```
   lingma-api.tongyi.example-cloud.com                    （example-cloud.com 域）
     → CNAME: *.gds.example-alibabadns.com                （example-alibabadns.com 域）
       → CNAME: *.example-aliyunga0019.com                （example-aliyunga0019.com 域）
         → A: 47.57.xxx.171 / 47.57.xxx.142
   ```

2. **Windows DNS Server 的 Conditional Forwarder 只配置了 `example-cloud.com`**：DNS Server 认为阿里云 DNS 只对 `example-cloud.com` 有授权

3. **Secure Cache Against Pollution 功能（默认开启）执行了 Bailiwick 检查**：
   - ✅ CNAME 记录（`lingma-api.tongyi.example-cloud.com`）→ 在 `example-cloud.com` 域内 → **接受**
   - ❌ 第二跳 CNAME（`*.gds.example-alibabadns.com`）→ 不在 `example-cloud.com` 域内 → **丢弃**
   - ❌ A 记录（`*.example-aliyunga0019.com`）→ 不在 `example-cloud.com` 域内 → **丢弃**

**结果**：DNS Server 只缓存了第一条 CNAME（无 A 记录），后续所有客户端查询在 CNAME TTL（190 秒）内都只能拿到 CNAME，无法获得最终的 IP 地址。

DNS Server 随后尝试通过通用 Forwarder 解析 CNAME 目标域名（`*.gds.example-alibabadns.com`），但通用 Forwarder 不一定能解析这些阿里云内部域名，导致间歇性失败。

**核心：这不是任何单一组件的 bug，而是 Conditional Forwarder 配置范围与阿里云 DNS 多域名架构之间的不匹配。**

### 间歇性原因

- **大部分时间正常**：DNS Server 的 CNAME Chase 通过通用 Forwarder 碰巧解析成功 → A 记录被缓存 → 客户端正常
- **偶尔失败**：CNAME Chase 通过通用 Forwarder 解析失败/超时 → 只有 CNAME 被缓存 → 接下来 ~3 分钟（CNAME TTL）内所有客户端只拿到 CNAME → 超时
- 阿里云 CDN 调度可能在不同时间返回不同的 CNAME 链，部分链可能不跨域

### Resolution

**方案 1（推荐）：扩大 Conditional Forwarder 覆盖范围**

为阿里云 CNAME 链涉及的域名添加 Conditional Forwarder，指向同一批阿里云 DNS Server：

```powershell
# 在 3 台 DNS Server 上执行
Add-DnsServerConditionalForwarderZone -Name "alibabadns.com" -MasterServers "阿里云DNS_IP1","阿里云DNS_IP2","阿里云DNS_IP3","阿里云DNS_IP4"
```

注意：需要和阿里云确认 CNAME 链涉及的所有域名模式。如 `aliyunga0019.com` 是动态生成的域名，需要确认是否有固定的父域模式。

**方案 2：改用通用 Forwarder 架构**

如果阿里云的域名模式不可预测，可以考虑将阿里云 DNS 加入通用 Forwarder 列表，替代 Conditional Forwarder。

### Workaround

临时关闭 Secure Cache Against Pollution（**仅用于验证根因，不建议作为长期方案**）：

```powershell
Set-DnsServerCache -PollutionProtection $false
Clear-DnsServerCache
# 客户端测试后恢复
Set-DnsServerCache -PollutionProtection $true
```

不需要重启 DNS 服务，立即生效。

---

## 6. 经验教训 (Lessons Learned)

### 技术知识

- **Secure Cache Against Pollution** 会对 DNS 响应中的每条记录**逐条做 Bailiwick（管辖权）检查**，不在 Conditional Forwarder 配置域名范围内的记录会被丢弃
- 即使 DNS Server 收到了包含完整 CNAME 链 + A 记录的响应，Secure Cache 机制也**不会智能地保留最终 A 记录**，而是机械地按域名范围过滤
- Windows DNS debug log 只记录包的元数据（方向、IP、响应码、查询名），**不记录 Answer Section 的具体内容**。需要配合 Wireshark/netmon 抓包才能看到完整响应
- `SecureResponses` 注册表键不存在 = 使用默认值（开启）。Windows Server 2003 及以后版本默认开启

### 排查方法

- **DNS Debug Log 的 CNAME Chase 模式**是识别 Secure Cache 问题的关键信号：当看到 DNS Server 向通用 Forwarder 查询 `原始域名.xxx.alibabadns.com` 格式的域名时，说明 DNS Server 只缓存了 CNAME 而丢弃了后续记录
- 区分 debug log 中的**缓存命中** vs **转发查询**：
  - 缓存命中：Rcv/Snd 的远端 IP 相同（都是客户端 IP），XID 相同，2 行完成
  - 转发查询：涉及客户端 IP + Forwarder IP，4 行完成（Rcv 客户端 → Snd Forwarder → Rcv Forwarder → Snd 客户端）
- 使用 TextAnalyzer 分析 DNS debug log 时，推荐的过滤条件：
  - `lingma-api AND 10.199.162.112`：查看转发到 Conditional Forwarder 的流量
  - `lingma-api AND alibabadns`：查看 CNAME Chase 行为
  - `alibabadns`：查看所有受影响的域名

### 预防措施

- 配置 Conditional Forwarder 时，**需要了解目标 DNS 可能返回的 CNAME 链涉及的所有域名**，确保 Conditional Forwarder 覆盖范围足够
- 对于使用 CDN/全局负载均衡的第三方 DNS 服务，CNAME 链跨域是常见模式，需要特别注意与 Windows DNS Secure Cache 的兼容性

---

## 7. 参考文档 (References)

- [KB241352 - How to Prevent DNS Cache Pollution](https://jeffpar.github.io/kbarchive/kb/241/Q241352/) — 微软官方 KB，说明 SecureResponses 注册表键和 Secure Cache 功能
- [CERT VU#109475 - Microsoft Windows DNS allow non-authoritative RRs to be cached](https://www.kb.cert.org/vuls/id/109475/) — CERT 安全公告，详细解释了 Secure Cache 机制：DNS Server 开启后会丢弃不在授权域名空间内的记录
- [Set-DnsServerCache PowerShell 文档](https://learn.microsoft.com/en-us/powershell/module/dnsserver/set-dnsservercache?view=windowsserver2025-ps) — PollutionProtection 参数说明

---
---

# Case Summary: DNS Conditional Forwarder CNAME Cross-Domain Resolution Failure

**Product/Service:** Windows DNS Server / Conditional Forwarder

---

## 1. Symptoms

- Clients experienced **intermittent DNS resolution timeouts** when accessing certain Alibaba Cloud services (e.g., `lingma-api.tongyi.example-cloud.com`)
- When tested with `nslookup`, DNS Server returned `NOERROR`, but the response contained **only CNAME records without the final A record (IP address)**:
  ```
  > server 10.0.1.71
  > lingma-api.tongyi.example-cloud.com
  Non-authoritative answer:
  Name:    lingma-api.tongyi.example-cloud.com
  ```
  (No IP address returned)
- Alibaba Cloud engineers confirmed via packet capture on their DNS Server side: **the DNS response contained complete CNAME + A records**, but clients only received the CNAME
- The issue was **intermittent**: resolution worked most of the time but occasionally failed for several minutes

---

## 2. Background / Environment

### Architecture

```
Client → Windows DNS Server (3 servers) → Conditional Forwarder → Alibaba Cloud DNS (4 servers)
```

### Configuration

- Windows DNS Server had a **Conditional Forwarder** configured for `example-cloud.com`, pointing to 4 Alibaba Cloud DNS servers (e.g., `10.199.162.112`)
- The 3rd DNS server in the Conditional Forwarder list was **unreachable**, but this did not affect the first 2 servers
- **Secure Cache Against Pollution** was enabled by default (DNS Manager → Advanced → "Secure cache against pollution" ✅)
- No DNS Policy configured
- Network connectivity confirmed good

### Alibaba Cloud DNS Response Structure (confirmed via Wireshark)

Alibaba Cloud DNS returned a complete CNAME chain + A records:

```
Answer Section:
① CNAME: lingma-api.tongyi.example-cloud.com
         → lingma-api.tongyi.example-cloud.com.gds.example-alibabadns.com     TTL=190s

② CNAME: lingma-api.tongyi.example-cloud.com.gds.example-alibabadns.com
         → ga-xxxx.example-aliyunga0019.com                                    TTL=80s

③ A:     ga-xxxx.example-aliyunga0019.com → 47.57.xxx.171                     TTL=194s
④ A:     ga-xxxx.example-aliyunga0019.com → 47.57.xxx.142                     TTL=194s
```

Key characteristic: **The CNAME chain crosses 3 different domains**:
- `example-cloud.com` (covered by Conditional Forwarder)
- `example-alibabadns.com` (NOT covered by Conditional Forwarder)
- `example-aliyunga0019.com` (NOT covered by Conditional Forwarder)

---

## 3. Investigation & Troubleshooting

### Step 1: Verify Basic DNS Configuration

**Action**: Checked Conditional Forwarder configuration and DNS Zones

```powershell
Get-DnsServerZone | Where-Object {$_.ZoneName -like "*example-cloud*"}
# ZoneType = Forwarder (i.e., Conditional Forwarder)

Get-DnsServerQueryResolutionPolicy
# No DNS Policy configured
```

**Finding**: Only one Conditional Forwarder Zone existed, no conflicting zones, no DNS Policy.

**Conclusion**: Ruled out Zone conflicts and Policy filtering.

---

### Step 2: Verify Secure Cache Against Pollution Status

**Action**: Checked registry and DNS Management Console

```powershell
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\DNS\Parameters" | Select SecureResponses
# Empty — registry key does not exist, using default value
```

**Finding**: `SecureResponses` key absent = default value applies. Per Microsoft documentation (KB241352), Windows 2000 SP3+ **enables Secure Cache by default**. Customer confirmed via DNS Manager GUI that the checkbox was ticked.

**Conclusion**: Secure Cache Against Pollution was active.

---

### Step 3: Enable DNS Debug Log and Reproduce

**Action**: Enabled DNS debug log on DNS Server, ran nslookup from client `10.65.90.12`

**nslookup result**: Returned only the domain name (CNAME), no A record.

---

### Step 4: Analyze Debug Log — Client Query

**Action**: Filtered debug log for client IP `10.65.90.12`

**Finding**:
```
206217 | 11:48:35 | Rcv 10.65.90.12 → Q  A    lingma-api.tongyi.example-cloud.com
206219 | 11:48:35 | Snd 10.65.90.12 ← R  NOERROR   ← Served directly from cache
```

**Conclusion**: DNS Server responded from cache without forwarding to Alibaba. The cache contained only CNAME, no A record.

---

### Step 5: Analyze Debug Log — Forwarding to Alibaba Cloud

**Action**: Filtered traffic between DNS Server and Alibaba Conditional Forwarder (`10.199.162.112`)

**Finding**: 3 forwards to Alibaba, all returned NOERROR:

```
26145 | 11:47:33 | Snd 10.199.162.112 → Q lingma-api.tongyi.example-cloud.com  ← Forward
26155 | 11:47:33 | Rcv 10.199.162.112 ← R NOERROR                               ← Alibaba responded
26157 | 11:47:33 | Snd 10.64.116.94   ← R NOERROR                               ← Sent to client
```

**Conclusion**: Alibaba responded successfully each time, but DNS Server returned only CNAME to clients. The A records from Alibaba's response were not cached.

---

### Step 6: Analyze Debug Log — CNAME Chase Behavior (Key Finding)

**Action**: Searched for `alibabadns` keyword in debug log

**Finding**:
```
118985 | 11:48:06 | Snd 10.111.125.34 → Q lingma-api.tongyi.example-cloud.com.gds.example-alibabadns.com
119193 | 11:48:06 | Rcv 10.111.125.34 ← R NOERROR
```

**Critical analysis**:
1. DNS Server sent a query to the **general forwarder** (`10.111.125.34`), NOT the Alibaba Conditional Forwarder
2. The queried domain was the CNAME target from Alibaba's response
3. `gds.example-alibabadns.com` falls outside the `example-cloud.com` Conditional Forwarder scope, so DNS Server routed it via the general forwarder instead

**Conclusion**: DNS Server received Alibaba's response, **kept only the CNAME within `example-cloud.com` scope, and discarded records for `example-alibabadns.com` and `example-aliyunga0019.com` domains** (including the final A records). The DNS Server then attempted to resolve the CNAME target independently via the general forwarder, which may not resolve Alibaba's internal DNS domains.

---

### Step 7: Verify Multiple Domains Affected

**Action**: Searched all `alibabadns` CNAME chase records

**Finding**: Multiple domains exhibited the same cross-domain CNAME chase pattern:

```
g.alicdn.com              → g.alicdn.com.gds.example-alibabadns.com
bluedot.is.autonavi.com   → bluedot.is.autonavi.com.gds.example-alibabadns.com
d-gm.mmstat.com           → d-gm.mmstat.com.gds.example-alibabadns.com
```

**Conclusion**: This is a systemic pattern in Alibaba Cloud's DNS architecture (Global DNS Service), not limited to a single domain.

---

### Step 8: Event Distribution Statistics

```
Event Type                    Count
CLIENT_QUERY                    752
CACHE_RSP_TO_CLIENT             752
FORWARDER (to Alibaba)            6
CNAME_CHASE (to general fwd)      2
```

During the ~2.5 minute debug log window, all 752 client queries were served from cache. Only 3 forwards to Alibaba and 1 CNAME chase occurred. The CNAME-only cache entry affected all clients for the duration of the CNAME TTL (~190 seconds).

---

## 4. Blockers & How They Were Resolved

| Blocker | Impact | Resolution |
|---------|--------|------------|
| DNS debug log does not record Answer Section content | Cannot directly observe A record being discarded; must rely on indirect evidence | Combined Alibaba-side Wireshark capture (confirmed A was sent) + DNS Server debug log (confirmed CNAME Chase behavior) + Secure Cache documentation to form complete evidence chain |
| Intermittent issue, difficult to reproduce | Cannot trigger on demand | Pre-enabled debug log for continuous recording, analyzed after natural occurrence |
| `SecureResponses` registry key absent | Initially appeared as if Secure Cache was not configured | Confirmed via documentation: absent key = default value (enabled). Verified via DNS Manager GUI |

---

## 5. Root Cause & Resolution

### Root Cause

The issue was caused by the combination of three factors:

1. **Alibaba Cloud DNS returns CNAME chains that cross multiple domains**:
   ```
   lingma-api.tongyi.example-cloud.com          (example-cloud.com domain)
     → CNAME: *.gds.example-alibabadns.com      (example-alibabadns.com domain)
       → CNAME: *.example-aliyunga0019.com       (example-aliyunga0019.com domain)
         → A: 47.57.xxx.171 / 47.57.xxx.142
   ```

2. **Conditional Forwarder only covers `example-cloud.com`**: DNS Server considers Alibaba DNS authoritative only for `example-cloud.com`

3. **Secure Cache Against Pollution (enabled by default) performs Bailiwick check on each record**:
   - ✅ CNAME (`lingma-api.tongyi.example-cloud.com`) → within `example-cloud.com` → **accepted**
   - ❌ 2nd CNAME (`*.gds.example-alibabadns.com`) → outside `example-cloud.com` → **discarded**
   - ❌ A records (`*.example-aliyunga0019.com`) → outside `example-cloud.com` → **discarded**

**Result**: DNS Server cached only the first CNAME (no A record). All subsequent client queries during the CNAME TTL (~190s) received only CNAME without IP address.

The DNS Server then attempted to chase the CNAME via the general forwarder, which may or may not successfully resolve Alibaba's internal DNS domains, causing intermittent failures.

**Core: This is not a bug in any single component, but a mismatch between the Conditional Forwarder scope and Alibaba Cloud's multi-domain DNS architecture.**

### Intermittency Explanation

- **Works most of the time**: CNAME chase via general forwarder happens to succeed → A record cached → clients work normally
- **Occasionally fails**: CNAME chase via general forwarder fails/times out → only CNAME cached → all clients fail for ~3 minutes (CNAME TTL)
- Alibaba's CDN scheduling may return different CNAME chains at different times

### Resolution

**Option 1 (Recommended): Expand Conditional Forwarder coverage**

Add Conditional Forwarders for domains involved in Alibaba's CNAME chain:

```powershell
# Execute on all 3 DNS Servers
Add-DnsServerConditionalForwarderZone -Name "alibabadns.com" -MasterServers "AliDNS_IP1","AliDNS_IP2","AliDNS_IP3","AliDNS_IP4"
```

Note: Coordinate with Alibaba Cloud to identify all domain patterns in their CNAME chains. Dynamic domains like `aliyunga0019.com` may require broader coverage.

**Option 2: Switch to General Forwarder architecture**

If Alibaba's domain patterns are unpredictable, add Alibaba DNS to the general forwarder list instead of Conditional Forwarder.

### Workaround

Temporarily disable Secure Cache Against Pollution (**for root cause verification only, not recommended long-term**):

```powershell
Set-DnsServerCache -PollutionProtection $false
Clear-DnsServerCache
# Test, then restore
Set-DnsServerCache -PollutionProtection $true
```

No DNS service restart required; takes effect immediately.

---

## 6. Lessons Learned

### Technical Knowledge

- **Secure Cache Against Pollution** performs per-record Bailiwick checks on DNS responses — records outside the Conditional Forwarder's configured domain scope are discarded
- Even when DNS Server receives a complete CNAME chain + A records, Secure Cache does **not intelligently preserve the final A record** — it mechanically filters by domain scope
- Windows DNS debug log records only packet metadata (direction, IP, response code, query name), **not Answer Section content**. Wireshark/netmon packet capture is needed for full response visibility
- `SecureResponses` registry key absent = default value (enabled) on Windows Server 2003+

### Troubleshooting Methods

- **CNAME Chase pattern in DNS debug log** is the key signal for Secure Cache issues: when DNS Server queries a general forwarder for `original-domain.xxx.alibabadns.com` format domains, it indicates the DNS Server cached only the CNAME and discarded subsequent records
- Distinguishing **cache hits** vs **forwarded queries** in debug log:
  - Cache hit: Rcv/Snd remote IPs identical (both client IP), same XID, 2 lines
  - Forwarded query: involves client IP + Forwarder IP, 4 lines

### Prevention

- When configuring Conditional Forwarders, **understand all domains that may appear in the target DNS's CNAME chains** to ensure adequate coverage
- For third-party DNS services using CDN/global load balancing, cross-domain CNAME chains are common and require special attention for compatibility with Windows DNS Secure Cache

---

## 7. References

- [KB241352 - How to Prevent DNS Cache Pollution](https://jeffpar.github.io/kbarchive/kb/241/Q241352/) — Microsoft KB explaining SecureResponses registry key and Secure Cache functionality
- [CERT VU#109475 - Microsoft Windows DNS allow non-authoritative RRs to be cached](https://www.kb.cert.org/vuls/id/109475/) — CERT advisory with detailed explanation: DNS Server with Secure Cache enabled discards records outside the delegated namespace
- [Set-DnsServerCache PowerShell Reference](https://learn.microsoft.com/en-us/powershell/module/dnsserver/set-dnsservercache?view=windowsserver2025-ps) — PollutionProtection parameter documentation
