---
layout: post
title: "NCSI 网络检测被劫持 — ISP HTTP 注入导致跳转到反诈骗网站"
date: 2026-02-15
categories: [NCSI, Windows-Client]
tags: [ncsi, http-hijacking, ttl-analysis, 302-redirect, isp-injection, network-connectivity, akamai-cdn, active-probing]

---

# Case Summary: NCSI 网络检测被劫持 — 通过 TTL 分析定位 ISP HTTP 注入导致跳转反诈骗网站

**Product/Service:** Windows 10/11 — NCSI (Network Connectivity Status Indicator)

---

## 1. 症状 (Symptoms)

- 客户报告 Windows 设备在进行网络连接检测时，**被自动跳转到一个反诈骗网站**（`http://198.51.100.101`），而非正常完成 NCSI 探测。
- 问题表现为间歇性发生——并非每次网络检测都会触发跳转，具有随机性。
- 正常情况下，NCSI 探测应透明完成，用户不会感知到任何网页跳转。

## 2. 背景 (Background / Environment)

- **操作系统**：Windows 10 / Windows 11
- **NCSI 工作原理**：
  - NCSI（Network Connectivity Status Indicator）是 Windows 用于检测当前网络是否具有 Internet 连接的内置组件
  - NCSI 通过 **Active Probing（主动探测）** 工作：
    1. 解析 `www.msftconnecttest.com`，获取 CDN IP 地址
    2. 向该 CDN IP 发起 HTTP 请求：`http://www.msftconnecttest.com/connecttest.txt`
    3. 正常情况下，CDN 返回 HTTP 200 OK，内容为 `Microsoft Connect Test`
    4. NCSI 据此判断 Internet 连接正常
  - NCSI 探测服务器自 2023 年 6 月起由 **Akamai CDN** 托管
- **问题时的 CDN IP**：解析 `www.msftconnecttest.com` 得到 CDN IP `203.0.113.50`
- **网络环境**：客户位于中国大陆，使用电信 ISP 接入

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

### 第一轮：确认 NCSI 正常工作流程

1. **做了什么**：梳理 NCSI 的工作逻辑，确认正常的 Active Probing 流程。
2. **发现了什么**：正常情况下：
   - DNS 解析 `www.msftconnecttest.com` → 获得 Akamai CDN IP
   - HTTP GET `http://www.msftconnecttest.com/connecttest.txt` → 返回 HTTP 200，Body 为 `Microsoft Connect Test`
3. **得出了什么结论**：有问题时，HTTP 请求被返回了 **302 重定向**到反诈骗网站 `http://198.51.100.101`，而非正常的 200 响应。需要确认是谁发送了这个 302 重定向。

### 第二轮：网络抓包对比 — 好的与坏的场景

1. **做了什么**：在问题时间段抓取网络流量，分别捕获到正常（200 OK）和异常（302 Redirect）两种场景的 HTTP 响应。
2. **发现了什么**：对比两种响应包发现了一个 **关键差异**：
   - **302 重定向响应（异常）**：IP 包的 **TTL = 51**
   - **200 OK 响应（正常）**：IP 包的 **TTL = 41**
   - 两者 TTL 相差 **10 跳**
3. **得出了什么结论**：同一个目标 IP（CDN `203.0.113.50`）返回的两种响应，TTL 差了 10 跳，这极不正常。说明 302 响应很可能**不是 CDN 服务器本身发出的**，而是路径上的某个中间设备伪造注入的。

### 第三轮：TTL 逆向分析 — 定位注入点

1. **做了什么**：基于 TTL 差异进行逆向分析，推算 302 响应的真实来源。

   **TTL 分析逻辑**：
   - 常见操作系统初始 TTL：Windows = 128，Linux = 64，Cisco = 255
   - Akamai CDN 服务器运行 Linux，初始 TTL 应为 **64**
   - **200 OK 响应**：TTL = 41 → 经过 64 - 41 = **23 跳**到达客户端 → 这是 CDN 到客户端的真实跳数
   - **302 响应**：TTL = 51 → 经过 64 - 51 = **13 跳**到达客户端 → 这个响应来自第 **13 跳**的设备
   - 注入设备距离客户端比 CDN **近了 10 跳**，位于客户端和 CDN 之间的网络路径上

2. **发现了什么**：TTL 分析表明 302 响应由路径上第 13 跳的设备注入，而非来自 CDN 服务器。
3. **得出了什么结论**：存在 **ISP HTTP 劫持/注入** — 网络路径上的某个中间设备（位于第 13 跳）拦截了明文 HTTP 请求，并抢先发送伪造的 302 重定向响应。建议执行 `tracert` 确认第 13 跳设备的具体 IP。

### 第四轮：Tracert 确认注入设备身份

1. **做了什么**：执行 `tracert 203.0.113.50`，查看第 13 跳的 IP 地址。
2. **发现了什么**：第 13 跳的 IP 为 `192.0.2.98`，经查询确认为 **电信骨干网设备**。
3. **得出了什么结论**：**ISP（电信）的骨干网设备在进行 HTTP 流量劫持**，针对 `http://www.msftconnecttest.com/connecttest.txt` 的明文 HTTP 请求注入 302 重定向，将流量导向反诈骗网站。这是一种典型的 ISP HTTP 注入行为。

### 第五轮：联系 ISP 解决

1. **做了什么**：将分析结果反馈给客户，建议联系电信 ISP 解除 HTTP 劫持。
2. **发现了什么**：电信确认了劫持行为，并在其设备上解除了针对该 URL 的 HTTP 注入规则。
3. **得出了什么结论**：问题根因确认为 ISP HTTP 劫持，ISP 解除后问题完全恢复。

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| 问题间歇性发生，不易稳定复现 | 需要多次抓包才能同时捕获到好的和坏的场景 | 持续抓包直到同时捕获到 200 和 302 两种响应，进行对比分析 |
| 302 响应伪装来自 CDN IP，表面上难以区分是 CDN 行为还是中间设备注入 | 如果只看 HTTP 层，容易误判为 CDN 或 Server 问题 | 通过 IP 层 TTL 差异分析，证明 302 响应的物理来源与 200 响应不同 |
| 需要 ISP 配合排查和修改 | 最终解决方案依赖 ISP 侧操作，非客户自己可控 | 提供完整的 TTL 分析和 tracert 证据，帮助客户与 ISP 沟通 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

**ISP（电信）骨干网设备对明文 HTTP 流量进行了劫持注入（HTTP Hijacking）。**

具体机制：
1. NCSI 发起 HTTP（非 HTTPS）请求 `http://www.msftconnecttest.com/connecttest.txt`
2. 该请求为**明文 HTTP**，路径上所有网络设备均可读取和修改
3. ISP 骨干网上的 DPI（Deep Packet Inspection）设备检测到该 HTTP 请求
4. 该设备**抢先于真正的 CDN 响应**，伪造了一个来自 CDN IP 的 302 重定向响应，将 URL 指向反诈骗网站 `http://198.51.100.101`
5. 由于 ISP 设备距离客户端（13 跳）比 CDN（23 跳）更近，伪造的 302 响应先于真正的 200 响应到达客户端
6. 客户端接受了先到达的 302 响应，导致 NCSI 检测异常并触发浏览器跳转

### Resolution

1. 通过 TTL 分析 + tracert 定位到注入设备为电信骨干网设备（`192.0.2.98`）
2. 客户联系电信 ISP，反馈 HTTP 劫持问题
3. 电信在其设备上**解除了针对该 URL 的 HTTP 注入规则**
4. 解除后 NCSI 探测恢复正常，不再发生跳转

### Workaround

在 ISP 修复之前，可考虑以下临时缓解措施：
- **切换为 HTTPS 探测**：通过组策略或注册表将 NCSI 的 Active Probe URL 改为 HTTPS（如支持），避免明文 HTTP 被劫持
- **使用 VPN**：通过加密隧道绕过 ISP 的 DPI 设备
- **手动指定 DNS**：如果 ISP 同时劫持 DNS，可切换到可信的公共 DNS

## 6. 经验教训 (Lessons Learned)

- **技术知识**：
  - **NCSI Active Probing 使用明文 HTTP**：`http://www.msftconnecttest.com/connecttest.txt` 是一个 HTTP（非 HTTPS）请求，因此天然容易被路径上的中间设备拦截、修改或注入。这是 ISP HTTP 劫持的典型受害场景。
  - **TTL 是识别 HTTP 注入的利器**：当同一个目标 IP 的不同 HTTP 响应呈现不同 TTL 值时，强烈暗示其中一个响应不是来自真实服务器，而是路径上的中间设备注入的。TTL 差值 = 注入点与真实服务器之间的跳数差。
  - **ISP HTTP 劫持的典型特征**：
    - 302/301 重定向响应的 TTL **高于**正常 200 响应的 TTL（因为注入设备更近）
    - 间歇性发生（ISP 设备处理能力有限，不一定每个请求都来得及注入）
    - 仅影响明文 HTTP，HTTPS 流量不受影响

- **排查方法**：
  - **TTL 对比分析法**：遇到可疑的 HTTP 重定向时，同时抓取正常和异常的响应包，**比较 IP 层的 TTL 值**。TTL 差异是识别中间设备注入的最直接证据。
  - **TTL 逆推公式**：
    - 确定可能的初始 TTL（Windows=128，Linux=64，Cisco=255）
    - 跳数 = 初始 TTL - 收到的 TTL
    - 将跳数与 `tracert` 结果对应，可精确定位注入设备
  - 排查顺序建议：**① 抓包对比正常/异常响应** → **② TTL 分析确认是否中间注入** → **③ tracert 定位注入设备** → **④ 联系 ISP/网络管理员**

- **预防措施**：
  - 在已知存在 ISP HTTP 劫持风险的网络环境中，优先使用 HTTPS 替代 HTTP 进行关键探测和通信。
  - 企业环境中可通过组策略自定义 NCSI 探测 URL，使用企业自己的 HTTPS 端点。
  - 遇到 NCSI 异常跳转时，不要仅关注 HTTP/应用层，要**深入到 IP 层的 TTL** 进行分析。

## 7. 参考文档 (References)

- [NCSI Overview — Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/ncsi/ncsi-overview) — NCSI 网络连接状态指示器概述，包括 Active Probing 工作原理
- [NCSI Frequently Asked Questions](https://learn.microsoft.com/en-us/windows-server/networking/ncsi/ncsi-frequently-asked-questions) — NCSI 常见问题，包括 Akamai 托管变更说明
- [An Internet Explorer or Edge window opens when connecting to a network](https://learn.microsoft.com/en-us/troubleshoot/windows-client/networking/internet-explorer-edge-open-connect-corporate-public-network) — NCSI 触发浏览器打开的相关排查说明

---
---

# Case Summary: NCSI Network Detection Hijacked — ISP HTTP Injection Causing Redirect to Anti-Fraud Website

**Product/Service:** Windows 10/11 — NCSI (Network Connectivity Status Indicator)

---

## 1. Symptoms

- The customer reported that during network connectivity detection on Windows, the device was **automatically redirected to an anti-fraud website** (`http://198.51.100.101`) instead of completing the NCSI probe normally.
- The issue occurred **intermittently** — not every network detection triggered the redirect; it happened randomly.
- Under normal conditions, NCSI probing should complete transparently without any visible webpage redirection.

## 2. Background / Environment

- **Operating System**: Windows 10 / Windows 11
- **NCSI Workflow**:
  - NCSI (Network Connectivity Status Indicator) is a built-in Windows component that detects whether the device has Internet connectivity
  - NCSI works through **Active Probing**:
    1. Resolves `www.msftconnecttest.com` to obtain the CDN IP address
    2. Sends an HTTP request to the CDN IP: `http://www.msftconnecttest.com/connecttest.txt`
    3. Normally, the CDN returns HTTP 200 OK with body content `Microsoft Connect Test`
    4. NCSI uses this to determine Internet connectivity is available
  - NCSI probe servers have been hosted by **Akamai CDN** since June 2023
- **CDN IP at Time of Issue**: DNS resolution of `www.msftconnecttest.com` returned CDN IP `203.0.113.50`
- **Network Environment**: Customer located in mainland China, using China Telecom ISP

## 3. Investigation & Troubleshooting

### Round 1: Understand the Normal NCSI Workflow

1. **Action**: Reviewed the normal NCSI Active Probing workflow.
2. **Finding**: Normal behavior:
   - DNS resolves `www.msftconnecttest.com` → Akamai CDN IP
   - HTTP GET `http://www.msftconnecttest.com/connecttest.txt` → Returns HTTP 200, body = `Microsoft Connect Test`
3. **Conclusion**: In the problematic scenario, the HTTP request received a **302 redirect** to the anti-fraud website `http://198.51.100.101` instead of the normal 200 response. Need to determine who sent the 302 redirect.

### Round 2: Network Capture Comparison — Good vs. Bad Scenarios

1. **Action**: Captured network traffic during the problem window, obtaining both normal (200 OK) and abnormal (302 Redirect) HTTP responses.
2. **Finding**: Comparing the two response packets revealed a **critical difference**:
   - **302 Redirect response (abnormal)**: IP packet **TTL = 51**
   - **200 OK response (normal)**: IP packet **TTL = 41**
   - TTL difference: **10 hops**
3. **Conclusion**: Two different responses from the same destination IP (CDN `203.0.113.50`) showed a 10-hop TTL difference — this is highly abnormal. It strongly suggests the 302 response was **not sent by the CDN server itself**, but was injected by an intermediate device on the network path.

### Round 3: Reverse TTL Analysis — Locating the Injection Point

1. **Action**: Performed reverse TTL analysis to calculate the true origin of the 302 response.

   **TTL Analysis Logic**:
   - Common initial TTL values: Windows = 128, Linux = 64, Cisco = 255
   - Akamai CDN servers run Linux, so initial TTL is likely **64**
   - **200 OK response**: TTL = 41 → Traversed 64 - 41 = **23 hops** from CDN to client → This is the real hop count from CDN to client
   - **302 response**: TTL = 51 → Traversed 64 - 51 = **13 hops** from injection point to client → This response originated from a device at hop **13**
   - The injecting device is **10 hops closer** to the client than the CDN, sitting between the client and the CDN on the network path

2. **Finding**: TTL analysis indicated the 302 response was injected by a device at hop 13, not from the CDN server.
3. **Conclusion**: **ISP HTTP hijacking/injection** was occurring — an intermediate device at hop 13 on the network path intercepted the plaintext HTTP request and raced to send a forged 302 redirect response. Recommended running `tracert` to identify the specific device at hop 13.

### Round 4: Tracert Confirms the Injecting Device

1. **Action**: Executed `tracert 203.0.113.50` and checked the IP address at hop 13.
2. **Finding**: Hop 13 showed IP `192.0.2.98`, which was confirmed to be a **China Telecom backbone network device**.
3. **Conclusion**: **The ISP's (China Telecom) backbone device was performing HTTP traffic hijacking**, injecting 302 redirects for plaintext HTTP requests to `http://www.msftconnecttest.com/connecttest.txt`, redirecting traffic to the anti-fraud website. This is a classic ISP HTTP injection attack.

### Round 5: ISP Resolution

1. **Action**: Shared the analysis results with the customer and recommended contacting China Telecom ISP to remove the HTTP hijacking.
2. **Finding**: China Telecom confirmed the injection behavior and removed the HTTP injection rule targeting this URL from their device.
3. **Conclusion**: Root cause confirmed as ISP HTTP hijacking. After ISP removal, the issue was fully resolved.

## 4. Blockers & How They Were Resolved

| Blocker | Impact | How It Was Resolved |
|---------|--------|---------------------|
| Issue occurred intermittently, difficult to reproduce consistently | Required multiple capture attempts to catch both good and bad scenarios | Performed continuous packet capture until both 200 and 302 responses were captured for comparison |
| The 302 response was spoofed to appear from the CDN IP, making it hard to distinguish from legitimate CDN behavior at the HTTP layer | Could easily be misdiagnosed as a CDN or server-side issue if only examining the HTTP layer | Used IP-layer TTL difference analysis to prove the 302 response came from a physically different source than the 200 response |
| Final resolution required ISP cooperation | The fix was outside the customer's direct control | Provided complete TTL analysis and tracert evidence to help the customer communicate effectively with the ISP |

## 5. Root Cause & Resolution

### Root Cause

**The ISP's (China Telecom) backbone network device was performing HTTP hijacking/injection on plaintext HTTP traffic.**

Detailed mechanism:
1. NCSI initiated an HTTP (not HTTPS) request to `http://www.msftconnecttest.com/connecttest.txt`
2. Since this was **plaintext HTTP**, all network devices on the path could read and modify the traffic
3. A DPI (Deep Packet Inspection) device on the ISP's backbone network detected the HTTP request
4. The device **raced ahead of the real CDN response** and forged a 302 redirect response spoofing the CDN's source IP, redirecting to the anti-fraud website `http://198.51.100.101`
5. Because the ISP device (13 hops away) was **closer to the client than the CDN** (23 hops), the forged 302 response arrived before the genuine 200 response
6. The client accepted the first-arriving 302 response, causing abnormal NCSI behavior and browser redirection

### Resolution

1. Identified the injection device as a China Telecom backbone device (`192.0.2.98`) through TTL analysis + tracert
2. Customer contacted China Telecom ISP to report the HTTP hijacking issue
3. China Telecom **removed the HTTP injection rule** targeting this URL from their device
4. After removal, NCSI probing returned to normal with no further redirects

### Workaround

Temporary mitigations before ISP resolution:
- **Switch to HTTPS probing**: Modify NCSI's Active Probe URL to HTTPS via Group Policy or registry (if supported), preventing plaintext HTTP from being hijacked
- **Use VPN**: Encrypted tunnel bypasses the ISP's DPI device
- **Manual DNS configuration**: If the ISP also hijacks DNS, switch to trusted public DNS servers

## 6. Lessons Learned

- **Technical Knowledge**:
  - **NCSI Active Probing uses plaintext HTTP**: `http://www.msftconnecttest.com/connecttest.txt` is an HTTP (not HTTPS) request, making it inherently vulnerable to interception, modification, or injection by intermediate devices. This is a classic victim scenario for ISP HTTP hijacking.
  - **TTL is a powerful tool for identifying HTTP injection**: When different HTTP responses from the same destination IP show different TTL values, it strongly indicates that one response did not originate from the real server but was injected by an intermediate device. TTL difference = hop count difference between the injection point and the real server.
  - **Typical characteristics of ISP HTTP hijacking**:
    - 302/301 redirect response TTL is **higher than** the normal 200 response TTL (because the injecting device is closer)
    - Occurs intermittently (ISP device has limited processing capacity; not every request is intercepted in time)
    - Only affects plaintext HTTP; HTTPS traffic is unaffected

- **Troubleshooting Methodology**:
  - **TTL comparison analysis**: When encountering suspicious HTTP redirects, capture both normal and abnormal response packets and **compare IP-layer TTL values**. TTL differences are the most direct evidence of intermediate device injection.
  - **TTL reverse calculation formula**:
    - Determine the likely initial TTL (Windows=128, Linux=64, Cisco=255)
    - Hops traversed = Initial TTL - Received TTL
    - Map the hop count to `tracert` results to precisely locate the injecting device
  - Recommended investigation order: **① Capture and compare normal/abnormal responses** → **② TTL analysis to confirm intermediate injection** → **③ tracert to locate the injecting device** → **④ Contact ISP/network administrator**

- **Prevention**:
  - In network environments with known ISP HTTP hijacking risks, prioritize HTTPS over HTTP for critical probes and communications.
  - In enterprise environments, customize the NCSI probe URL via Group Policy to use an enterprise-owned HTTPS endpoint.
  - When encountering abnormal NCSI redirects, don't limit investigation to the HTTP/application layer — **dig into IP-layer TTL** for analysis.

## 7. References

- [NCSI Overview — Windows Server](https://learn.microsoft.com/en-us/windows-server/networking/ncsi/ncsi-overview) — NCSI Network Connectivity Status Indicator overview, including Active Probing mechanism
- [NCSI Frequently Asked Questions](https://learn.microsoft.com/en-us/windows-server/networking/ncsi/ncsi-frequently-asked-questions) — NCSI FAQ, including Akamai hosting migration details
- [An Internet Explorer or Edge window opens when connecting to a network](https://learn.microsoft.com/en-us/troubleshoot/windows-client/networking/internet-explorer-edge-open-connect-corporate-public-network) — Troubleshooting NCSI-triggered browser behavior
