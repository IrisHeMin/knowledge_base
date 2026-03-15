---
layout: post
title: "DNS 解析间歇性失败 — 本地 Zone 劫持 CNAME 链"
date: 2026-02-03
categories: [DNS, Windows-Server]
tags: [dns, zone, cname, authoritative, forwarder, ttl, cache, azurewebsites]
type: "case-summary"
---

# Case Summary: DNS 解析间歇性失败 — 本地 Zone 劫持 CNAME 链

**Date:** 2026-02-03
**Product/Service:** Windows DNS Server

---

## 1. 症状 (Symptoms)

- 客户端无法解析 `app.powerbi.com`，DNS 查询**无法返回 A 记录（IP 地址）**
- 问题**间歇性发生**：时而正常，时而失败
- 如果在客户端手动指定公共 DNS forwarder（如 `203.0.113.1` 或 `203.0.113.2`）进行查询，则**始终能正常解析**

## 2. 背景 (Background / Environment)

- **DNS Server:** `corpdc01.fabrikam.corp`（IP: `10.0.1.10`），同时作为 Domain Controller
- **客户端 IP:** `10.0.2.50`
- **DNS Forwarder 配置:** `203.0.113.1`、`203.0.113.2`
- **关键配置:** DNS Server 上存在一个本地 Zone `azurewebsites.net`，该 Zone 内仅包含 NS 记录，**无任何 A 记录**
- `app.powerbi.com` 的完整 CNAME 解析链：
  ```
  app.powerbi.com
    → app.privatelink.analysis.windows.net          (CNAME, TTL: 785s)
    → a1b2c3d4-e5f6-7890-abcd-ef1234567890.trafficmanager.net  (CNAME, TTL: 3628s)
    → app-pbi-wfe-region-v3.pbi-wfe-region-v3-ase.p.azurewebsites.net  (CNAME, TTL: 300s)
    → [A record IP]  (最终需要解析的 A 记录)
  ```

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

1. **在 DNS Server 上抓包，从客户端发起查询**
   - **做了什么：** 在 DNS Server `corpdc01.fabrikam.corp` 上启动网络抓包（Network Monitor），同时从客户端 `10.0.2.50` 发起对 `app.powerbi.com` 的 DNS A 记录查询
   - **发现了什么：**
     - Frame 7095（10:52:26）：客户端向 DNS Server 发送 A 记录查询请求 `app.powerbi.com`
     - Frame 7295（10:52:27）：DNS Server 返回响应，状态码 Success，包含 **3 条 CNAME 记录**，但**没有最终的 A 记录（无 IP）**：
       ```
       app.powerbi.com → app.privatelink.analysis.windows.net (CNAME)
       app.privatelink.analysis.windows.net → a1b2c3d4-...trafficmanager.net (CNAME)
       a1b2c3d4-...trafficmanager.net → app-pbi-wfe-region-v3...p.azurewebsites.net (CNAME)
       ```
   - **得出结论：** DNS Server 解析了完整的 CNAME 链，但无法将最终 CNAME 解析为 A 记录。问题出在 DNS Server 对最后一跳 `*.p.azurewebsites.net` 的解析环节

2. **检查 DNS Server 上的 Zone 配置**
   - **做了什么：** 查看 DNS Server 上的 Zone List
   - **发现了什么：** DNS Server 上配置了一个本地 Zone `azurewebsites.net`，该 Zone 内**只有 NS 记录，没有任何 A 记录**
   - **得出结论：** 当 DNS Server 尝试解析 `app-pbi-wfe-region-v3...p.azurewebsites.net` 时，命中了本地 Zone `azurewebsites.net`。由于 DNS Server 认为自己是该 Zone 的权威服务器（authoritative），它**不会再将查询转发给 forwarder**。而本地 Zone 中无对应 A 记录，因此返回空结果

3. **验证直接使用 Forwarder 可以正常解析**
   - **做了什么：** 在客户端直接指定 forwarder `203.0.113.1` / `203.0.113.2` 进行查询
   - **发现了什么：** 能正常获取到完整的 CNAME 链和 A 记录（IP 地址）
   - **得出结论：** 上游 DNS 解析正常，问题完全出在本地 DNS Server 的 Zone 配置

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| 问题间歇性发生，难以按需复现 | 增加排查难度，需要深入理解 DNS Cache TTL 机制才能解释时好时坏的现象 | 通过分析 CNAME（TTL 长）和 A 记录（TTL 短）的 TTL 差异，推导出完整的间歇性故障触发机制（详见根因分析） |
| 本地 Zone `azurewebsites.net` 静默劫持了 CNAME 解析 | DNS Server 不报错，只是静默返回无 A 记录的响应，不易察觉 | 通过网络包分析确认响应中缺少 A 记录，再关联 Zone 配置定位根因 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

DNS Server 上配置了一个本地 Zone `azurewebsites.net`（仅含 NS 记录，无 A 记录），导致 CNAME 解析链在最后一跳被本地 Zone "劫持"。完整的间歇性故障周期如下：

1. 客户端首次查询 `app.powerbi.com` → DNS Server 缓存为空 → 转发到 forwarder → 获取完整 CNAME 链 + A 记录 → ✅ **解析成功**
2. DNS Server 缓存 CNAME 记录（TTL 较长：785s ~ 3628s）和 A 记录（TTL 较短：~300s）
3. A 记录 TTL 到期，从缓存中移除；但 **CNAME 记录仍在缓存中**
4. 客户端再次查询 → DNS Server 从缓存取到 CNAME 链 → 需要重新解析最终 CNAME `xxx.p.azurewebsites.net`
5. DNS Server 发现本地有 Zone `azurewebsites.net` → 认为自己是权威服务器 → **不转发到 forwarder**
6. 本地 Zone 中无对应 A 记录 → 返回无 IP → ❌ **解析失败**
7. 等到所有 CNAME 缓存也过期后 → DNS Server 重新向 forwarder 查询 → 又获取完整结果 → ✅ **问题暂时消失**
8. 循环往复，形成间歇性故障

### Resolution

**推荐方案：删除本地 Zone `azurewebsites.net`**（该 Zone 内仅有 NS 记录，无实际用途）

操作步骤：

1. **备份 Zone：**
   ```powershell
   Export-DnsServerZone -Name "azurewebsites.net" -FileName "azurewebsites.net.bak.dns"
   ```
   导出的文件保存在 `C:\Windows\System32\dns\`，可用文本编辑器打开查看所有记录

2. **将备份文件移至安全位置**（如 backup 文件夹）

3. **删除该 Zone**

4. **验证** `app.powerbi.com` 可以正常解析

**如需后续恢复该 Zone（回滚步骤）：**
1. 通过 DNS 管理界面创建一个同名新 Zone
2. 右键属性 → 将新 Zone 改为本地文件类型（File-backed）
3. 此时在 `C:\Windows\System32\dns\` 下会创建一个空的 Zone 文件
4. 删除该新 Zone
5. 将之前备份的 `azurewebsites.net.bak.dns` 重命名为 `azurewebsites.net.dns`，放入 `C:\Windows\System32\dns\`
6. 重启 DNS Service → Zone 文件会被自动加载，所有记录恢复
7. 如需 AD 集成，在 Zone 属性中修改为 AD-integrated

### Workaround（备选方案）

- **方案 B：** 在 Zone `azurewebsites.net` 中手动添加对应 A 记录 → **不推荐**，Azure 的 IP 会变动，维护成本高
- **方案 C：** 在 Zone `azurewebsites.net` 下创建 Delegation `p.azurewebsites.net`，将 NS 指向 forwarder → 当解析 `*.p.azurewebsites.net` 时会命中此 delegation，查询被正确转发

## 6. 经验教训 (Lessons Learned)

- **技术知识：**
  - DNS Server 上如果配置了某个 Zone，则该 Zone 域名下的**所有查询都会被本地处理**，不会转发到 forwarder。这是 DNS 的 authoritative 行为——一旦 Server 认为自己是该 Zone 的权威，就不会向外部查询
  - CNAME 链中不同记录的 TTL 往往不同。A 记录 TTL 通常较短（~300s），CNAME TTL 较长（785s ~ 3628s）。当 A 记录过期而 CNAME 仍在缓存中时，DNS Server 需要重新解析最终 CNAME，此时可能走不同的解析路径，触发隐藏问题
- **排查方法：**
  - 对于间歇性 DNS 解析失败，**第一步检查 DNS Server 上的 Zone List**，确认是否存在可能"劫持"解析链的本地 Zone
  - 在 DNS Server 端抓包是关键手段——可以清楚看到 Server 实际返回的响应内容，对比是否缺少 A 记录
  - 使用 `nslookup` 直接指向 forwarder 测试是快速隔离问题位置（本地 DNS vs 上游）的有效方法
- **预防措施：**
  - 定期审查 DNS Server 上的 Zone 配置，清理无用的空 Zone——特别是 Azure 公共域名相关的 Zone（如 `*.azurewebsites.net`、`*.azure-api.net`、`*.database.windows.net` 等）
  - 在配置 Private Endpoint / Private DNS Zone 时，注意检查是否会与现有 Zone 产生冲突

## 7. 参考文档 (References)

- [DNS Server 不转发本地 Zone 的查询](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/dns-server-not-forwarding-queries) — 解释 DNS Server 对本地 authoritative Zone 不做 forwarding 的行为
- [Export-DnsServerZone](https://learn.microsoft.com/en-us/powershell/module/dnsserver/export-dnsserverzone) — Zone 备份导出命令参考
- [Azure Private Endpoint DNS 配置](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns) — Private Link / Private DNS Zone 的 DNS 配置最佳实践
- [Name Resolution for Resources in Azure Virtual Networks](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-name-resolution-for-vms-and-role-instances) — Azure 网络中的 DNS 解析架构说明

---
---

# Case Summary: Intermittent DNS Resolution Failure — Local Zone Hijacking CNAME Chain

**Date:** 2026-02-03
**Product/Service:** Windows DNS Server

---

## 1. Symptoms

- Client unable to resolve `app.powerbi.com` — DNS query **returns no A record (no IP address)**
- Issue is **intermittent**: resolution succeeds at times and fails at others
- Querying directly against public DNS forwarders (`203.0.113.1` or `203.0.113.2`) **always succeeds**

## 2. Background / Environment

- **DNS Server:** `corpdc01.fabrikam.corp` (IP: `10.0.1.10`), also serving as Domain Controller
- **Client IP:** `10.0.2.50`
- **DNS Forwarders:** `203.0.113.1`, `203.0.113.2`
- **Key Configuration:** A local zone `azurewebsites.net` exists on the DNS Server, containing **only NS records — no A records**
- Full CNAME resolution chain for `app.powerbi.com`:
  ```
  app.powerbi.com
    → app.privatelink.analysis.windows.net          (CNAME, TTL: 785s)
    → a1b2c3d4-e5f6-7890-abcd-ef1234567890.trafficmanager.net  (CNAME, TTL: 3628s)
    → app-pbi-wfe-region-v3.pbi-wfe-region-v3-ase.p.azurewebsites.net  (CNAME, TTL: 300s)
    → [A record IP]  (final A record to be resolved)
  ```

## 3. Investigation & Troubleshooting

1. **Captured network trace on DNS Server while querying from the client**
   - **Action:** Started a network capture (Network Monitor) on DNS Server `corpdc01.fabrikam.corp`; initiated a DNS A record query for `app.powerbi.com` from client `10.0.2.50`
   - **Finding:**
     - Frame 7095 (10:52:26): Client sent A record query for `app.powerbi.com` to DNS Server
     - Frame 7295 (10:52:27): DNS Server returned a Success response with **3 CNAME records but no final A record (no IP)**:
       ```
       app.powerbi.com → app.privatelink.analysis.windows.net (CNAME)
       app.privatelink.analysis.windows.net → a1b2c3d4-...trafficmanager.net (CNAME)
       a1b2c3d4-...trafficmanager.net → app-pbi-wfe-region-v3...p.azurewebsites.net (CNAME)
       ```
   - **Conclusion:** DNS Server resolved the full CNAME chain but failed to resolve the final CNAME to an A record. The issue lies in the DNS Server's handling of the last-hop `*.p.azurewebsites.net`

2. **Examined DNS Server zone configuration**
   - **Action:** Reviewed the Zone List on the DNS Server
   - **Finding:** A local zone `azurewebsites.net` exists on the DNS Server, containing **only NS records — no A records**
   - **Conclusion:** When the DNS Server attempted to resolve `app-pbi-wfe-region-v3...p.azurewebsites.net`, it matched the local zone `azurewebsites.net` and considered itself authoritative — therefore it did **not** forward the query to forwarders. Since the zone had no matching A record, it returned no IP

3. **Verified resolution works when pointing directly to forwarders**
   - **Action:** Queried `app.powerbi.com` directly against forwarders `203.0.113.1` / `203.0.113.2`
   - **Finding:** Successfully received both the complete CNAME chain and A records (IP addresses)
   - **Conclusion:** Upstream DNS resolution is working correctly. The issue is entirely caused by the local DNS Server's zone configuration

## 4. Blockers & How They Were Resolved

| Blocker | Impact | How Resolved |
|---------|--------|--------------|
| Intermittent nature made the issue hard to reproduce on demand | Increased troubleshooting difficulty; required understanding of DNS cache TTL mechanics to explain the on-off pattern | Analyzed CNAME (long TTL) vs. A record (short TTL) difference to deduce the complete intermittent failure cycle (see Root Cause) |
| Local zone `azurewebsites.net` silently hijacked CNAME resolution | DNS Server returned no error — just a response with no A record, making it hard to detect | Identified through network packet analysis showing missing A records, then correlated with zone configuration |

## 5. Root Cause & Resolution

### Root Cause

A local zone `azurewebsites.net` (containing only NS records, no A records) on the DNS Server hijacked the final hop of the CNAME resolution chain. The complete intermittent failure cycle:

1. Client's first query for `app.powerbi.com` → empty cache → forwarded to forwarders → full CNAME chain + A record returned → ✅ **success**
2. DNS Server caches CNAME records (long TTL: 785s–3628s) and A record (short TTL: ~300s)
3. A record TTL expires → removed from cache; **CNAME records remain cached**
4. Client queries again → DNS Server retrieves CNAME chain from cache → needs to re-resolve final CNAME `xxx.p.azurewebsites.net`
5. DNS Server finds local zone `azurewebsites.net` → considers itself **authoritative** → does **not** forward
6. No matching A record in local zone → returns no IP → ❌ **resolution fails**
7. Once all CNAME cache entries also expire → DNS Server forwards the full query to forwarders again → ✅ **temporarily works**
8. Cycle repeats, producing intermittent failure

### Resolution

**Recommended: Delete the local zone `azurewebsites.net`** (it serves no purpose — contains only NS records).

Steps:

1. **Backup the zone:**
   ```powershell
   Export-DnsServerZone -Name "azurewebsites.net" -FileName "azurewebsites.net.bak.dns"
   ```
   Exported file is saved to `C:\Windows\System32\dns\` and can be opened with a text editor

2. **Move the backup file to a safe location** (e.g., a backup folder)

3. **Delete the zone**

4. **Verify** `app.powerbi.com` resolves correctly

**Rollback procedure (to restore the zone if needed):**
1. Create a new zone (same name) via DNS Manager GUI
2. Right-click → Properties → change to file-backed zone type
3. A new empty zone file is created in `C:\Windows\System32\dns\`
4. Delete this new zone
5. Rename the backup `azurewebsites.net.bak.dns` → `azurewebsites.net.dns`, place in `C:\Windows\System32\dns\`
6. Restart DNS Service → the zone file will be loaded automatically with all records restored
7. If AD integration is needed, change zone properties to AD-integrated

### Workaround (Alternatives)

- **Option B:** Add a static A record in the zone → **Not recommended** — Azure IPs change frequently, high maintenance burden
- **Option C:** Create a delegation `p.azurewebsites.net` under the zone, with NS records pointing to the forwarders → queries for `*.p.azurewebsites.net` will hit the delegation and be properly forwarded

## 6. Lessons Learned

- **Technical Knowledge:**
  - When a DNS Server hosts a local zone, **all queries for names under that zone are resolved locally** — they are never forwarded, even if the zone has no matching records. This is standard DNS authoritative behavior
  - CNAME and A records in a resolution chain often have different TTLs. A records typically have shorter TTLs (~300s) while CNAMEs are longer (785s–3628s). When the A record expires but CNAMEs remain cached, the server must re-resolve the final hop, potentially hitting a different resolution path and exposing hidden issues
- **Troubleshooting Approach:**
  - For intermittent DNS resolution failures, **first check the Zone List** on the DNS Server for zones that might "hijack" the resolution chain
  - Capturing packets on the DNS Server side is a key technique — it reveals exactly what the server returns and whether A records are missing
  - Using `nslookup` pointed directly at a forwarder is a fast way to isolate whether the issue is on the local DNS Server or upstream
- **Prevention:**
  - Periodically audit DNS Server zone configurations; clean up unused or empty zones — especially Azure public domain zones (`*.azurewebsites.net`, `*.azure-api.net`, `*.database.windows.net`, etc.)
  - When configuring Private Endpoints / Private DNS Zones, verify that existing local zones do not conflict with the CNAME resolution chain

## 7. References

- [DNS Server does not forward queries for local zones](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/dns-server-not-forwarding-queries) — Explains DNS authoritative zone non-forwarding behavior
- [Export-DnsServerZone](https://learn.microsoft.com/en-us/powershell/module/dnsserver/export-dnsserverzone) — Zone backup export command reference
- [Azure Private Endpoint DNS configuration](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns) — Best practices for Private Link DNS setup
- [Name Resolution for Resources in Azure Virtual Networks](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-name-resolution-for-vms-and-role-instances) — Azure network DNS resolution architecture
