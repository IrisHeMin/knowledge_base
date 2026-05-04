---
layout: post
title: "Tech Share: Print Spooler 与 DNS — Win2012R2 起 ANY 查询对 CNAME 的行为变化"
date: 2022-05-16
categories: [DNS, Print-Spooler, Windows-Server]
tags: [dns, cname, wtype-any, spooler, infoblox, ttt-trace, name-resolution]
shared_by: Nico Chen (collab. Luyao / Yang Yang)
source: "Internal Tech Share — APGCK Network Group"
excerpt: "客户用 CNAME 'print' 访问打印服务器返回 0x00000709 'invalid printer name'。根因是上游 Infoblox DNS 在 wtype=ANY 查询 CNAME 时返回了不该带的 IP，绕过了 Spooler 自 Win2012R2 起依赖的两阶段查询逻辑。"
---

# Tech Share: Print Spooler 与 DNS — Win2012R2 起 ANY 查询对 CNAME 的行为变化

> **来源**：内部 Tech Share — Windows APGCK Network Techshare Group  
> **分享人**：Nico Chen（协作：Luyao Feng、Yang Yang）  
> **关键词**：Print Spooler · DNS · CNAME · `wtype=ANY` · Infoblox · `0x00000709`

---

## 1. 问题是什么 (Problem)

客户在 Infoblox DNS 上为打印服务器创建了一条 CNAME，希望客户端用别名 **`print`** 访问：

```
print  →  print.contoso.net.au  →  wpprint20.retail.ad.contoso.net.au
```

但客户端连接打印服务器时，Spooler **失败并返回 `0x00000709 "invalid printer name"`**，直接走 FQDN（`wpprint20.retail.ad.contoso.net.au`）则正常。

**核心矛盾**：CNAME 在 `nslookup` / 浏览器层面解析正常，唯独 Print Spooler 拒绝。

---

## 2. 排查思路 (Investigation)

### 2.1 先理解 Spooler 怎么用 DNS

Spooler 在校验客户端访问的服务器名时，逻辑是：

1. 服务启动时，对**自身 FQDN** 发起一次 DNS 查询，把返回的 IP 列表 + 名字列表缓存进内存的 `NameCache`。
2. 客户端连进来时，对**客户端使用的名字**（这里是 `print`）再做一次 DNS 查询。
3. 把第 2 步的响应**和第 1 步的缓存做匹配**。匹配上才允许，匹配不上就 `0x00000709`。

注意：Spooler 用的查询类型是 **`wtype = ANY`**（不是常见的 `A`）。

### 2.2 Win2012R2 起的关键行为变化

DNS Server 对 `wtype=ANY` 查询 **CNAME** 的响应方式从 Server 2012R2 起变了：

| 版本 | `ANY` 查询 CNAME 的响应 |
|---|---|
| Server 2008R2 及之前 | 返回 CNAME **+ 解析出的 IP** |
| Server 2012R2 及之后 | **只返回 CNAME，没有 IP** |

`nslookup` 也能复现：`set type=A` 查 CNAME 有 IP；`set type=ANY` 只有别名。

为了适配，Spooler 自 2012R2 起也升级了逻辑：

1. **第一轮 `ANY`**：如果响应里没 IP，说明这是个 CNAME → 触发第二轮。
2. **第二轮 `A`**：拿到 conditional name + 真实 IP，再去和缓存比对。
3. 如果第一轮 ANY 就**直接带回 IP**，Spooler 会认为这是 A 记录，**不会发起第二轮**。

### 2.3 解码 TTT trace 看实际响应

用工具把 Spooler 查询代码片段 hook 出来，并解码客户的 TTT trace，发现：

```
spoolsv!TNameResolutionCache::ProcessDnsQueryResults+0x28e
  pszName  = "Print"
  szA      = "172.25.44.44"   ← 居然带了 IP！
```

`172.25.44.44` 正是 Infoblox 的 DNS 服务器 IP——而**不是**打印服务器的 IP。

→ Spooler 拿这个 `172.25.44.44` 去和 `wpprint20...` 缓存的 IP 比对，匹配失败 → `0x00000709`。

### 2.4 收敛根因

- Win2012R2 起，标准 DNS 对 `ANY` 查 CNAME **不应**返回 IP；
- 但本案的 Infoblox **返回了 IP**（且 IP 是 DNS 自身），破坏了 Spooler 的两阶段查询前提；
- Spooler 误把它当作 A 记录处理，比对失败。

---

## 3. 解决办法 (Solution)

| 方向 | 动作 |
|---|---|
| **根治** | 与 **Infoblox 厂商**确认并修复其在 `wtype=ANY` 查询 CNAME 时错误返回 IP 的行为，使其与 Windows DNS Server 2012R2+ 行为对齐 |
| **临时规避** | 让客户端直接使用打印服务器 **FQDN** 而非 CNAME 别名访问 |
| **验证手段** | 用 nslookup `set type=ANY` 对 CNAME 查询，确认响应**只含别名、不含 IP**；再观察 Spooler 是否触发第二轮 `A` 查询（TTT / Procmon / 网络抓包） |

> 本案不是 Windows 侧 bug，Spooler 的两阶段逻辑是**正确的**——它信任的是 RFC + Win2012R2 起 Microsoft DNS 的标准行为。问题在第三方 DNS 实现差异。

---

## 4. Takeaway

1. **CNAME + 三方 DNS = 隐藏雷区**。当 Windows 组件依赖某个 DNS 行为细节时，混入 Infoblox / BIND / F5 这类第三方解析器，就要警惕版本/实现差异。
2. **`wtype=ANY` 不是"什么都给我"**。它在 Win2012R2 起对 CNAME 的语义是"先告诉我这是不是别名"，配合二次 `A` 查询用——别误以为它等价于 `A+AAAA+CNAME` 的合集。
3. **Spooler 报 `0x00000709 "invalid printer name"`，名字解析层是必查项**。错误文案听起来是"名字不对"，根因往往是 IP 比对失败。
4. **排查名字解析问题时，看 Spooler/客户端实际拿到的 IP**，而不是只看 nslookup `type=A` 的结果——客户端用的查询类型可能完全不同。
5. **TTT + Symbol 解析变量（pszName/szA）** 是这种"代码层 vs DNS 实际响应"对照的最快路径。

---

## 参考 (References)

- [DNS Wire Format & Query Types — RFC 1035](https://www.rfc-editor.org/rfc/rfc1035)
- [Time Travel Debugging Overview](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-overview)
- [Print Spooler troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-server/printing/print-spooler-troubleshoot)
