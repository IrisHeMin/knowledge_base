---
layout: post
title: "Deep Dive: WinHTTP vs WinINET — 为什么服务用 WinHTTP，IE 走 WinINET"
date: 2026-05-11
categories: [Knowledge, Networking]
tags: [winhttp, wininet, proxy, wsus, windows-update, http]
type: "deep-dive"
---

# Deep Dive: WinHTTP vs WinINET

**Topic:** Windows HTTP 客户端栈（WinHTTP / WinINET）及代理行为差异
**Category:** Networking / Windows Platform
**Level:** 中级
**Last Updated:** 2026-05-11

---

## 1. 概述 (Overview)

Windows 上有**两套 HTTP 客户端栈**：`wininet.dll`（WinINET）和 `winhttp.dll`（WinHTTP）。两者都能发 HTTP/HTTPS 请求，看起来功能重叠，但它们的设计目标完全不同：

- **WinINET** 是给**交互式用户程序**用的（IE、Edge legacy、Outlook、老的 ClickOnce 等），强调"和用户的浏览器体验一致"——共享 Cookie、共享凭据缓存、能弹证书警告框、能弹代理认证框。
- **WinHTTP** 是给**服务（Service）和中间件**用的，强调"无人值守、可被任意账号调用、支持 impersonation 和 session 隔离"，**永不弹 UI**。

理解这两者的区别，是解释一个常见困惑的关键：**"我在 IE 里设了代理，为什么 WSUS / Windows Update / BITS 还是连不上 Microsoft Update？"** —— 因为这些都是 Windows 服务，它们读的是 WinHTTP 代理，根本不看 IE 设置。

---

## 2. 核心概念 (Core Concepts)

### WinINET (`wininet.dll`)
- 1995 年随 IE 一起出现的 HTTP/FTP/Gopher 客户端库
- 数据结构（Cookie、缓存、凭据、代理）**全部绑定到当前用户会话**
- 配置存储在 `HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings`
- 微软文档明确：**"WinINet is not supported for use in a service."**

### WinHTTP (`winhttp.dll`)
- Windows 2000 后引入，专为服务器端 / 服务场景设计
- 不依赖用户会话、Profile、Desktop、Window Station
- 配置存储在 `HKLM`，通过 `netsh winhttp` 或 `WinHttpSetDefaultProxyConfiguration` 配置
- 支持线程级 impersonation（不同线程可以以不同身份发请求）

### 代理 (Proxy) 的概念在两者中的差异
| 维度 | WinINET | WinHTTP |
|---|---|---|
| 默认代理来源 | 用户的 IE 设置 (HKCU) | `netsh winhttp` 配置 (HKLM) |
| WPAD/PAC | 默认开启自动检测 | 支持但需显式调用 `WinHttpGetProxyForUrl` |
| 代理认证弹框 | 会弹 | 永不弹（必须代码里塞凭据） |

> **类比**：WinINET 像一个"跟着你登录的浏览器助手"——你怎么设它就怎么用；WinHTTP 像一个"机房里的批处理脚本"——只读机器级配置，没有"用户偏好"概念。

### Service 上下文（Service Context）
Windows 服务运行在 `LocalSystem` / `LocalService` / `NetworkService` 等账号下，这些账号：
- 没有交互式登录会话
- 没有加载 User Profile（HKCU 是空的或默认的）
- 没有 Desktop（不能弹窗）
- 不能阻塞等待用户输入

WinINET 的全部功能都依赖前三项，所以在服务里调用 WinINET 会出现死锁、超时、`HTTP 12175` (`ERROR_INTERNET_SECURE_FAILURE`) 等莫名其妙的问题。

---

## 3. 工作原理 (How It Works)

### 整体架构

```
┌────────────────────────────────────────────────────────────────┐
│                  应用层 (Applications)                          │
├──────────────────────────┬─────────────────────────────────────┤
│   交互式应用              │   服务 / 后台进程                     │
│   • Internet Explorer    │   • WSUS (WsusService)              │
│   • Edge Legacy          │   • Windows Update Agent (wuauserv) │
│   • Outlook              │   • BITS                            │
│   • 老桌面程序            │   • Defender 更新                    │
│   • PowerShell           │   • SCCM 客户端                      │
│     (Invoke-WebRequest)  │   • Azure Arc Agent                 │
├──────────────────────────┼─────────────────────────────────────┤
│   wininet.dll            │   winhttp.dll                       │
│   ↓ 读 HKCU              │   ↓ 读 HKLM + netsh winhttp         │
├──────────────────────────┴─────────────────────────────────────┤
│                  Schannel (TLS) / Winsock (TCP)                 │
└────────────────────────────────────────────────────────────────┘
```

### WinINET 的代理决定流程

1. **应用调用** `InternetOpen()` → 指定 `INTERNET_OPEN_TYPE_PRECONFIG`（默认）
2. WinINET 读 `HKCU\...\Internet Settings`：
   - `ProxyEnable` (0/1)
   - `ProxyServer`（如 `proxy:8080` 或 `http=p1:80;https=p2:443`）
   - `ProxyOverride`（bypass 列表）
   - `AutoConfigURL`（PAC 脚本 URL）
3. 如果配置了 PAC，下载 PAC 文件并执行 `FindProxyForURL(url, host)`
4. 如果代理需要认证 → **弹框问用户名密码**（凭据缓存命中则跳过）
5. 发请求

### WinHTTP 的代理决定流程

1. **应用调用** `WinHttpOpen()`，指定代理访问类型：
   - `WINHTTP_ACCESS_TYPE_NO_PROXY` — 直连
   - `WINHTTP_ACCESS_TYPE_NAMED_PROXY` — 代码硬编码代理
   - `WINHTTP_ACCESS_TYPE_DEFAULT_PROXY` — 读 `netsh winhttp` 配置（最常用）
   - `WINHTTP_ACCESS_TYPE_AUTOMATIC_PROXY` (Win 8.1+) — 自动检测
2. 读取 HKLM 下的 WinHTTP proxy 配置
3. 如果调用了 `WinHttpGetProxyForUrl()`，会执行 WPAD/PAC 逻辑
4. 代理认证 → **必须代码里通过 `WinHttpSetCredentials()` 提供**，不会弹框
5. 发请求

### 为什么"在 IE 里设代理 ≠ 给 WinHTTP 设代理"

```
用户改 IE 代理设置
       ↓
写入 HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings
       ↓
       ├─ WinINET 立即生效  ✅
       └─ WinHTTP 完全不读 HKCU ❌

要让 WinHTTP 知道：
       ↓
   netsh winhttp set proxy ...
       ↓
   写入 HKLM\...\WinHttp\Connections
       ↓
   WinHTTP 生效 ✅
```

`netsh winhttp import proxy source=ie` 只是**把当前 IE 配置快照拷贝**到 WinHTTP 配置，之后 IE 再改 WinHTTP **不会跟着变**。

### 关键机制：Session 隔离与 Impersonation

WinHTTP 的 "session" 是逻辑上独立的——每个 `WinHttpOpen` 句柄有自己独立的：
- Cookie 集（不会和别的进程串）
- 凭据
- 代理配置覆盖

这对**多租户服务**很关键：一个进程内可以用不同身份并行发请求，互不污染。WinINET 做不到这一点，因为它的 Cookie Jar 和凭据缓存是用户全局共享的。

---

## 4. 关键配置与参数 (Key Configurations)

### WinHTTP 代理配置命令

| 命令 | 作用 |
|---|---|
| `netsh winhttp show proxy` | 查看当前 WinHTTP 代理 |
| `netsh winhttp set proxy proxy-server="proxy:8080" bypass-list="*.contoso.com;<local>"` | 设置代理 |
| `netsh winhttp set proxy proxy-server="http=p1:80;https=p2:443"` | 协议分别设置 |
| `netsh winhttp reset proxy` | 清除代理（直连） |
| `netsh winhttp import proxy source=ie` | 从 IE 导入快照 |

### WinINET 代理配置位置

| 注册表路径 | 关键值 |
|---|---|
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings` | `ProxyEnable`, `ProxyServer`, `ProxyOverride`, `AutoConfigURL` |
| GPO: `User Configuration\Preferences\Control Panel Settings\Internet Settings` | 推送到用户 |

### 应用栈对照表

| 应用 / 组件 | 用什么栈 | 备注 |
|---|---|---|
| Internet Explorer | WinINET | 当然 |
| Edge (Chromium) | 自己的网络栈 | 但**默认读 IE/系统代理**，不读 WinHTTP |
| Outlook | WinINET | EWS、Autodiscover 等 |
| WSUS Server (同步) | WinHTTP | + WSUS 控制台里的代理设置 |
| Windows Update Agent | WinHTTP | 客户端拉更新 |
| BITS | WinHTTP | 后台传输 |
| Defender 签名更新 | WinHTTP | |
| .NET `HttpClient` (默认) | WinHTTP (`SocketsHttpHandler`) | .NET Core / 5+ |
| .NET `WebClient` / `HttpWebRequest` | WinINET 的封装 | .NET Framework |
| PowerShell `Invoke-WebRequest` | .NET 栈 → 实际走 WinINET (Framework) 或 sockets (.NET 7+) | **测试代理时容易误判** |
| `curl.exe` (Win10+ 自带) | 自己的栈 | 不读任何 Windows 代理 |

---

## 5. 常见问题与排查 (Common Issues & Troubleshooting)

### 问题 A: WSUS 同步失败，提示连接超时
- **可能原因**：管理员在 IE 里设了代理，但没给 WinHTTP 设
- **排查思路**：
  1. `netsh winhttp show proxy` 看是不是 "Direct access (no proxy server)"
  2. 看 `SoftwareDistribution.log` 是否有 `WebException`
- **修复**：
  ```cmd
  netsh winhttp set proxy proxy-server="proxy.contoso.com:8080" bypass-list="*.contoso.com;<local>"
  ```
  或者在 WSUS 控制台 → Options → Update Source and Proxy Server 里配置

### 问题 B: Windows Update 报 0x80072EE2 / 0x8024401C
- **可能原因**：WinHTTP 代理没配 / 代理需要认证但没提供凭据
- **排查思路**：
  ```powershell
  netsh winhttp show proxy
  # 测连通性（注意：Invoke-WebRequest 不一定走 WinHTTP！）
  ```
- **关键工具**：用 `wget`/`curl` 不靠谱（它们不读 WinHTTP）。**最准确**的方法是用一个真正调用 WinHTTP 的小工具，例如 [PsPing] / [WinHTTPTest]，或者直接看 BITS/WUAgent 日志。

### 问题 C: PowerShell 测试连接通，但服务还是失败
- **可能原因**：`Invoke-WebRequest` 默认走 .NET → WinINET（Framework）/ socket（Core），而服务走 WinHTTP，两条路代理不同
- **修复**：测试时模拟 WinHTTP 行为：
  ```powershell
  $proxy = [System.Net.WebRequest]::GetSystemWebProxy()
  $proxy.GetProxy('https://sws.update.microsoft.com')
  ```

### 问题 D: 代理需要认证，服务卡住或 407
- **可能原因**：WinHTTP 不会弹框，需要凭据但代码没塞
- **修复**：
  - 部分组件支持把凭据存在配置里（WSUS 控制台代理设置里有"使用用户凭据连接代理"）
  - 或在代理上对该机器/服务账号做 IP 白名单 / NTLM 透传

### 问题 E: 改了 `netsh winhttp` 后服务还是用老代理
- **可能原因**：服务进程启动时读了一次配置后缓存
- **修复**：重启服务（`Restart-Service wuauserv` / `Restart-Service WsusService`）

---

## 6. 实战经验 (Practical Tips)

### 最佳实践
- **服务器上配代理**：永远先 `netsh winhttp set proxy`，再考虑 IE/GPO
- **域环境**：用 GPO **Computer Configuration → Administrative Templates → Network → Network Isolation / Proxy** 推 WinHTTP 代理，比手动 netsh 可维护性高
- **测试代理是否对服务生效**：触发一次 Windows Update 检查（`UsoClient StartScan`）观察行为，比用 PowerShell 准

### 常见误区
- ❌ "我改了 IE 代理，WSUS 应该也通了" → 不会，两套配置完全独立
- ❌ "`Invoke-WebRequest` 通了，服务也应该通" → 走的栈不一样
- ❌ "`netsh winhttp import proxy source=ie` 之后 IE 改了 WinHTTP 会跟着改" → 不会，是一次性快照
- ❌ "Edge Chromium 用 WinHTTP" → 不，它读系统/IE 代理

### 性能考量
- WinHTTP 是异步 I/O (IOCP)，在高并发场景比 WinINET 性能更好
- WinINET 的 Cookie/Cache 共享在多线程下有锁竞争

### 安全注意
- WinHTTP 不会自动弹证书警告，**TLS 证书校验失败就直接失败**——这是好事（更安全），但调试时容易被自签名证书坑
- WinINET 在某些版本会"记住"用户的"忽略证书"决定，这在企业里是合规风险
- **TLS 1.2 启用**：老系统（Server 2012 R2 / 2016）上的 WinHTTP 默认不用 TLS 1.2，必须改注册表 `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings\WinHttp\DefaultSecureProtocols = 0xA00`

---

## 7. 与相关技术的对比 (Comparison with Related Technologies)

| 维度 | **WinINET** | **WinHTTP** | **.NET HttpClient** | **libcurl / curl.exe** |
|---|---|---|---|---|
| 设计目标 | 交互式应用 | 服务 / 后台 | 跨平台应用 | 通用 CLI / 库 |
| 代理来源 | IE 设置 (HKCU) | netsh winhttp (HKLM) | 系统代理 / 自定义 | 仅命令行参数 / 环境变量 |
| 服务里能用？ | ❌ 不支持 | ✅ 推荐 | ✅ 可以 | ✅ 可以 |
| Impersonation | ❌ | ✅ | 部分支持 | ❌ |
| 弹 UI | ✅ 会弹 | ❌ 永不弹 | ❌ | ❌ |
| Cookie Jar | ✅ 全局 | ❌ 无 | ✅ Per-handler | ✅ 可选 |
| FTP | ✅ | ❌ | ❌ (.NET Core 移除) | ✅ |
| 异步 I/O | 旧式回调 | IOCP | 现代 async | event-driven |
| 适用场景 | 桌面浏览器/邮件 | Windows 服务 | 现代 .NET 应用 | 脚本/排查 |

**选型建议**：
- 写 Windows 服务 / IIS 模块 → **WinHTTP** 或 **.NET HttpClient**
- 写桌面交互应用，需要和 IE 共享 Cookie → **WinINET**
- 跨平台、新写代码 → **.NET HttpClient (SocketsHttpHandler)**
- 排查和测试 → 多种工具一起用，**别用一种工具的结果推测另一种**

---

## 8. 参考资料 (References)

- [WinINet vs. WinHTTP — Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/wininet/wininet-vs-winhttp) — 微软官方对照表，权威依据
- [WinHTTP Start Page — Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/winhttp/winhttp-start-page) — WinHTTP API 总入口
- [About WinINet — Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/wininet/about-wininet) — WinINET 概念和限制
- [netsh winhttp commands — Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-winhttp) — netsh winhttp 命令参考

---

---

# Deep Dive: WinHTTP vs WinINET (English Version)

**Topic:** Windows HTTP client stacks (WinHTTP / WinINET) and proxy behavior differences
**Category:** Networking / Windows Platform
**Level:** Intermediate
**Last Updated:** 2026-05-11

---

## 1. Overview

Windows ships **two HTTP client stacks**: `wininet.dll` (WinINET) and `winhttp.dll` (WinHTTP). Both can issue HTTP/HTTPS requests, but they were designed for fundamentally different use cases:

- **WinINET** powers **interactive user-facing applications** (Internet Explorer, Edge legacy, Outlook, older ClickOnce apps). It shares cookies and credentials with the user's browser session, can pop UI dialogs (cert warnings, proxy auth prompts), and reads per-user configuration.
- **WinHTTP** is built for **services and middleware**. It supports unattended operation, can be called from any account (including `LocalSystem`), supports thread impersonation and session isolation, and **never displays UI**.

Understanding this distinction explains a very common confusion: **"I configured a proxy in Internet Explorer — why can't WSUS / Windows Update / BITS reach Microsoft Update?"** Because all three are Windows services, and they read the **WinHTTP** proxy, not IE's.

---

## 2. Core Concepts

### WinINET (`wininet.dll`)
- Shipped with Internet Explorer in 1995 as the HTTP/FTP/Gopher client library
- All state (cookies, cache, credentials, proxy) is **bound to the current user session**
- Configuration lives in `HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings`
- Microsoft documentation explicitly states: **"WinINet is not supported for use in a service."**

### WinHTTP (`winhttp.dll`)
- Introduced in Windows 2000 specifically for server-side / service scenarios
- Does not depend on user session, profile, desktop, or window station
- Configuration in `HKLM`, configured via `netsh winhttp` or `WinHttpSetDefaultProxyConfiguration`
- Supports per-thread impersonation

### Proxy semantics differ
| Aspect | WinINET | WinHTTP |
|---|---|---|
| Default proxy source | User's IE settings (HKCU) | `netsh winhttp` config (HKLM) |
| WPAD/PAC | Auto-detect on by default | Supported, but requires explicit `WinHttpGetProxyForUrl` |
| Proxy auth prompt | Pops UI | Never pops; credentials must be set in code |

> **Analogy**: WinINET is like a "browser helper that follows you when you log in" — it uses whatever you configured. WinHTTP is like a "batch job in a server room" — it only reads machine-level config, with no concept of "user preferences".

### Service Context
Windows services run as `LocalSystem` / `LocalService` / `NetworkService` accounts. These accounts:
- Have no interactive logon session
- Don't load a user profile (HKCU is empty/default)
- Have no desktop (can't show UI)
- Can't block waiting for user input

WinINET requires the first three. Calling WinINET from a service leads to deadlocks, timeouts, and cryptic errors like `HTTP 12175` (`ERROR_INTERNET_SECURE_FAILURE`).

---

## 3. How It Works

### Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                  Application Layer                              │
├──────────────────────────┬─────────────────────────────────────┤
│   Interactive apps       │   Services / background processes   │
│   • Internet Explorer    │   • WSUS (WsusService)              │
│   • Edge Legacy          │   • Windows Update Agent (wuauserv) │
│   • Outlook              │   • BITS                            │
│   • Legacy desktop apps  │   • Defender updates                │
│   • PowerShell           │   • SCCM client                     │
│     (Invoke-WebRequest)  │   • Azure Arc Agent                 │
├──────────────────────────┼─────────────────────────────────────┤
│   wininet.dll            │   winhttp.dll                       │
│   ↓ reads HKCU           │   ↓ reads HKLM + netsh winhttp      │
├──────────────────────────┴─────────────────────────────────────┤
│                  Schannel (TLS) / Winsock (TCP)                 │
└────────────────────────────────────────────────────────────────┘
```

### WinINET proxy resolution flow

1. App calls `InternetOpen()` with `INTERNET_OPEN_TYPE_PRECONFIG` (default)
2. WinINET reads `HKCU\...\Internet Settings`:
   - `ProxyEnable` (0/1)
   - `ProxyServer` (e.g., `proxy:8080` or `http=p1:80;https=p2:443`)
   - `ProxyOverride` (bypass list)
   - `AutoConfigURL` (PAC URL)
3. If PAC is configured, downloads it and runs `FindProxyForURL(url, host)`
4. If proxy needs auth → **prompts the user** (cached creds skip this)
5. Sends request

### WinHTTP proxy resolution flow

1. App calls `WinHttpOpen()` with an access type:
   - `WINHTTP_ACCESS_TYPE_NO_PROXY` — direct
   - `WINHTTP_ACCESS_TYPE_NAMED_PROXY` — hardcoded in code
   - `WINHTTP_ACCESS_TYPE_DEFAULT_PROXY` — reads `netsh winhttp` (most common)
   - `WINHTTP_ACCESS_TYPE_AUTOMATIC_PROXY` (Win 8.1+) — auto-detect
2. Reads HKLM WinHTTP proxy config
3. If `WinHttpGetProxyForUrl()` is called, executes WPAD/PAC logic
4. Proxy auth → **must be provided via `WinHttpSetCredentials()`**, no UI
5. Sends request

### Why "setting proxy in IE ≠ setting proxy for WinHTTP"

```
User changes IE proxy settings
       ↓
Written to HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings
       ↓
       ├─ WinINET sees it immediately ✅
       └─ WinHTTP never reads HKCU    ❌

To make WinHTTP aware:
       ↓
   netsh winhttp set proxy ...
       ↓
   Written to HKLM\...\WinHttp\Connections
       ↓
   WinHTTP picks it up ✅
```

`netsh winhttp import proxy source=ie` is a **one-time snapshot copy** — later IE changes do **not** propagate.

### Key mechanism: Session isolation & impersonation

A WinHTTP "session" (per `WinHttpOpen` handle) is logically isolated:
- Independent cookie jar (no cross-process bleed)
- Independent credentials
- Independent proxy override

This is critical for **multi-tenant services**: a single process can issue requests as different identities in parallel without contamination. WinINET cannot — its cookie jar and credential cache are user-global.

---

## 4. Key Configurations

### WinHTTP proxy commands

| Command | Purpose |
|---|---|
| `netsh winhttp show proxy` | View current WinHTTP proxy |
| `netsh winhttp set proxy proxy-server="proxy:8080" bypass-list="*.contoso.com;<local>"` | Set proxy |
| `netsh winhttp set proxy proxy-server="http=p1:80;https=p2:443"` | Per-protocol |
| `netsh winhttp reset proxy` | Clear (direct) |
| `netsh winhttp import proxy source=ie` | Snapshot from IE |

### WinINET proxy locations

| Registry path | Key values |
|---|---|
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings` | `ProxyEnable`, `ProxyServer`, `ProxyOverride`, `AutoConfigURL` |
| GPO: `User Configuration → Preferences → Control Panel Settings → Internet Settings` | Pushed to users |

### Stack matrix

| Component | Stack used | Notes |
|---|---|---|
| Internet Explorer | WinINET | Of course |
| Edge (Chromium) | Own net stack | But **reads IE/system proxy by default**, not WinHTTP |
| Outlook | WinINET | EWS, Autodiscover, etc. |
| WSUS Server (sync) | WinHTTP | + WSUS console proxy settings |
| Windows Update Agent | WinHTTP | Client-side update fetch |
| BITS | WinHTTP | Background transfers |
| Defender signature update | WinHTTP | |
| .NET `HttpClient` (default) | WinHTTP (`SocketsHttpHandler`) | .NET Core / 5+ |
| .NET `WebClient` / `HttpWebRequest` | Wraps WinINET | .NET Framework |
| PowerShell `Invoke-WebRequest` | .NET stack — WinINET (Framework) or sockets (.NET 7+) | **Easy to mislead when testing proxies** |
| `curl.exe` (Win10+) | Own stack | Reads no Windows proxy |

---

## 5. Common Issues & Troubleshooting

### Issue A: WSUS sync fails with connection timeout
- **Likely cause**: Admin set the proxy in IE but not in WinHTTP
- **Diagnosis**:
  1. `netsh winhttp show proxy` — does it say "Direct access"?
  2. Check `SoftwareDistribution.log` for `WebException`
- **Fix**:
  ```cmd
  netsh winhttp set proxy proxy-server="proxy.contoso.com:8080" bypass-list="*.contoso.com;<local>"
  ```
  Or set via WSUS Console → Options → Update Source and Proxy Server.

### Issue B: Windows Update returns 0x80072EE2 / 0x8024401C
- **Likely cause**: WinHTTP proxy not set, or proxy requires auth and credentials not provided
- **Diagnosis**:
  ```powershell
  netsh winhttp show proxy
  ```
- **Key tools**: `wget`/`curl` don't read WinHTTP and are unreliable here. Use a WinHTTP-based test tool, or read BITS/WUAgent logs directly.

### Issue C: PowerShell connectivity test passes, but service still fails
- **Likely cause**: `Invoke-WebRequest` uses .NET → WinINET (Framework) / sockets (Core), while the service uses WinHTTP. Different proxy paths.
- **Workaround**: Simulate WinHTTP behavior:
  ```powershell
  $proxy = [System.Net.WebRequest]::GetSystemWebProxy()
  $proxy.GetProxy('https://sws.update.microsoft.com')
  ```

### Issue D: Proxy requires auth — service hangs or returns 407
- **Likely cause**: WinHTTP doesn't prompt; credentials must be supplied programmatically
- **Fix**:
  - Some components support stored creds (e.g., WSUS console "use user credentials to connect to proxy")
  - Or whitelist the machine/service account on the proxy / use NTLM passthrough

### Issue E: After `netsh winhttp set proxy`, service still uses old proxy
- **Likely cause**: Service cached config at startup
- **Fix**: Restart the service (`Restart-Service wuauserv` / `Restart-Service WsusService`)

---

## 6. Practical Tips

### Best practices
- **Server proxy config**: Always `netsh winhttp set proxy` first; consider IE/GPO second
- **Domain environments**: Use GPO under **Computer Configuration → Administrative Templates → Network → Network Isolation / Proxy** to push WinHTTP proxy
- **Verifying a service sees the proxy**: Trigger a real Windows Update scan (`UsoClient StartScan`) — more reliable than PowerShell tests

### Common pitfalls
- ❌ "I changed IE proxy — WSUS should work now" → No, two independent configs
- ❌ "`Invoke-WebRequest` works, so the service should too" → Different stacks
- ❌ "`netsh winhttp import proxy source=ie` keeps WinHTTP in sync with IE" → No, one-time snapshot
- ❌ "Edge Chromium uses WinHTTP" → No, it reads system/IE proxy

### Performance considerations
- WinHTTP uses async I/O (IOCP), better under high concurrency
- WinINET's shared cookie/cache has lock contention under multi-threading

### Security notes
- WinHTTP does not auto-prompt cert warnings — TLS validation failures fail hard (good for security, but self-signed certs can confuse debugging)
- WinINET in some versions remembers "ignore cert" decisions — a compliance risk in enterprises
- **TLS 1.2 enablement**: On older systems (Server 2012 R2 / 2016), WinHTTP doesn't enable TLS 1.2 by default. Set `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings\WinHttp\DefaultSecureProtocols = 0xA00`.

---

## 7. Comparison with Related Technologies

| Aspect | **WinINET** | **WinHTTP** | **.NET HttpClient** | **libcurl / curl.exe** |
|---|---|---|---|---|
| Design target | Interactive apps | Services / background | Cross-platform apps | Generic CLI / lib |
| Proxy source | IE settings (HKCU) | netsh winhttp (HKLM) | System proxy / custom | CLI args / env vars |
| Service-safe? | ❌ Not supported | ✅ Recommended | ✅ Yes | ✅ Yes |
| Impersonation | ❌ | ✅ | Partial | ❌ |
| UI prompts | ✅ Yes | ❌ Never | ❌ | ❌ |
| Cookie jar | ✅ Global | ❌ None | ✅ Per-handler | ✅ Optional |
| FTP | ✅ | ❌ | ❌ (removed in .NET Core) | ✅ |
| Async I/O | Old callbacks | IOCP | Modern async | Event-driven |
| Use case | Desktop browser/mail | Windows services | Modern .NET apps | Scripting/diagnostics |

**Selection guidance**:
- Writing a Windows service / IIS module → **WinHTTP** or **.NET HttpClient**
- Writing a desktop interactive app sharing cookies with IE → **WinINET**
- New cross-platform code → **.NET HttpClient (SocketsHttpHandler)**
- Diagnostics → use multiple tools, **don't extrapolate one tool's result to another stack**

---

## 8. References

- [WinINet vs. WinHTTP — Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/wininet/wininet-vs-winhttp) — Authoritative side-by-side comparison
- [WinHTTP Start Page — Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/winhttp/winhttp-start-page) — WinHTTP API entry point
- [About WinINet — Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/wininet/about-wininet) — WinINET concepts and limitations
- [netsh winhttp commands — Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-winhttp) — netsh winhttp command reference
