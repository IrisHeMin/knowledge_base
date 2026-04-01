---
layout: post
title: "DNS Conditional Forwarder CNAME Cross-Domain Resolution Failure - Secure Cache Against Pollution"
date: 2026-04-01
categories: case-summaries
tags: [dns, conditional-forwarder, cname, secure-cache, pollution-protection, debug-log]
---

# Case Summary: DNS Conditional Forwarder CNAME Cross-Domain Resolution Failure

**Product/Service:** Windows DNS Server / Conditional Forwarder

---

## Issue Definition

Clients intermittently failed to resolve `lingma-api.tongyi.aliyun.com` via Windows DNS Server. When the issue occurred, DNS Server returned `NOERROR` but the response contained **only a CNAME record without the final A record (IP address)**, causing application timeouts. The issue was intermittent — worked most of the time, failed occasionally for several minutes.

**Architecture:**
```
Client --> Windows DNS Server (x3) --> Conditional Forwarder (aliyun.com) --> Alibaba Cloud DNS (x4)
```

---

## Information Gathered

1. **Conditional Forwarder** configured for `aliyun.com`, pointing to 4 Alibaba DNS servers (e.g., `10.199.162.112`). The 3rd server in the list was unreachable.

2. **Secure Cache Against Pollution** was enabled (default). Confirmed via DNS Manager GUI — "Secure cache against pollution" checkbox was ticked. Registry key `SecureResponses` did not exist, meaning default value (enabled) applied.

3. **Alibaba Cloud Wireshark capture** (captured on Alibaba DNS server side) showed their DNS response contained a complete CNAME chain + A records:
   ```
   1. CNAME: lingma-api.tongyi.aliyun.com
             --> lingma-api.tongyi.aliyun.com.gds.alibabadns.com          TTL=190s

   2. CNAME: lingma-api.tongyi.aliyun.com.gds.alibabadns.com
             --> ga-bpladtuvv8ekkixpbejij.aliyunga0019.com                TTL=80s

   3. A:     ga-bpladtuvv8ekkixpbejij.aliyunga0019.com --> 47.57.143.171  TTL=194s
   4. A:     ga-bpladtuvv8ekkixpbejij.aliyunga0019.com --> 47.57.7.142    TTL=194s
   ```
   The CNAME chain crossed 3 domains: `aliyun.com` --> `alibabadns.com` --> `aliyunga0019.com`

4. **Client nslookup** reproduced the issue — returned only CNAME, no A record:
   ```
   > server 10.107.125.71
   > lingma-api.tongyi.aliyun.com
   Non-authoritative answer:
   Name:    lingma-api.tongyi.aliyun.com
   (No IP address)
   ```

5. **No DNS Policy** configured. No conflicting zones. Network connectivity confirmed good.

---

## DNS Debug Log Findings

Debug log was enabled on the DNS Server during reproduction. Key findings:

### Finding 1: Client query was served from cache

The nslookup client (`10.65.90.12`) at `11:48:35` received a response directly from cache — DNS Server did NOT forward to Alibaba at that moment:
```
206217 | 11:48:35 | Rcv 10.65.90.12     --> Q  A  lingma-api.tongyi.aliyun.com
206219 | 11:48:35 | Snd 10.65.90.12     <-- R  NOERROR   (from cache, no forwarding)
```
This means the cache already contained a CNAME-only entry (no A record).

### Finding 2: Earlier forwards to Alibaba all returned NOERROR

Other clients triggered cache refresh earlier. All 3 forwards to Alibaba returned `NOERROR`:
```
26145 | 11:47:33 | Snd 10.199.162.112  --> Q  lingma-api.tongyi.aliyun.com  (forward to Alibaba)
26155 | 11:47:33 | Rcv 10.199.162.112  <-- R  NOERROR                        (Alibaba responded)
26157 | 11:47:33 | Snd 10.64.116.94    <-- R  NOERROR                        (returned to client)

29515 | 11:47:34 | (2nd forward to Alibaba --> NOERROR)
59323 | 11:47:45 | (3rd forward to Alibaba --> NOERROR)
```
Note: Debug log only records packet metadata (direction, IP, response code, query name). It does **NOT** record Answer Section content. So we can see `NOERROR` but cannot see what records were inside the response.

### Finding 3: CNAME Chase to general forwarder (KEY FINDING)

At `11:48:06`, DNS Server sent a query to the **general forwarder** (`10.111.125.34`) — NOT the Alibaba Conditional Forwarder — for the CNAME target domain:
```
118985 | 11:48:06 | Snd 10.111.125.34  --> Q  lingma-api.tongyi.aliyun.com.gds.alibabadns.com
119193 | 11:48:06 | Rcv 10.111.125.34  <-- R  NOERROR
```

This same CNAME chase pattern appeared for multiple other domains:
```
g.alicdn.com                  --> g.alicdn.com.gds.alibabadns.com
bluedot.is.autonavi.com       --> bluedot.is.autonavi.com.gds.alibabadns.com
d-gm.mmstat.com               --> d-gm.mmstat.com.gds.alibabadns.com
```

### Finding 4: Event statistics

Over ~2.5 minutes of debug log: 752 client queries all served from cache, only 3 forwards to Alibaba, 1 CNAME chase.

---

## Client-Side Packet Capture — Direct Evidence

A Wireshark capture between the Windows DNS Server (`10.107.125.71`) and the general forwarder (`10.111.125.34`) provided direct evidence of the CNAME chase and the intermittency:

**Frame 81366 (11:42:41) — Failure case:**
```
10.107.125.71 --> 10.111.125.34:  Q  lingma-api.tongyi.aliyun.com.gds.alibabadns.com
10.111.125.34 --> 10.107.125.71:  R  Answer RRs: 1

  Answer:
    CNAME: lingma-api.tongyi.aliyun.com.gds.alibabadns.com
           --> ga-bp1mt0z4o22gjtuolz4j1.aliyunga0017.com
    TTL: 300s

  Additional: OPT only (NO A record!)
```

**Frame 2191x (11:47:42) — Success case:**
```
10.107.125.71 --> 10.111.125.34:  Q  lingma-api.tongyi.aliyun.com.gds.alibabadns.com
10.111.125.34 --> 10.107.125.71:  R  CNAME + A 47.57.143.171  (A record present)
```

This capture directly confirmed:
1. **CNAME Chase is happening** — DNS Server queries the general forwarder for `*.gds.alibabadns.com`
2. **Intermittency root cause** — The general forwarder (`10.111.125.34`) returns inconsistent results: sometimes CNAME-only (no A), sometimes CNAME + A
3. **The general forwarder should never have been involved** — This query should have gone to Alibaba Cloud DNS, not the general forwarder

---

## Logical Reasoning

**Evidence chain:**

1. Alibaba Cloud DNS returns complete CNAME + A records (confirmed by Alibaba-side Wireshark)
2. Client only receives CNAME without A (confirmed by nslookup)
3. DNS Server performs CNAME chase to general forwarder for `*.gds.alibabadns.com` (confirmed by both debug log AND client-side Wireshark)
4. General forwarder returns inconsistent results — sometimes with A, sometimes without (confirmed by client-side Wireshark)

**Inference:**

DNS Server's **Secure Cache Against Pollution** (enabled by default) performs Bailiwick checks on each record in Alibaba's response:

- CNAME for `lingma-api.tongyi.aliyun.com` → within `aliyun.com` scope → **ACCEPTED**
- CNAME for `*.gds.alibabadns.com` → outside `aliyun.com` scope → **DISCARDED**
- A records for `*.aliyunga0019.com` → outside `aliyun.com` scope → **DISCARDED**

Per Microsoft documentation:
- [KB241352](https://jeffpar.github.io/kbarchive/kb/241/Q241352/): "a Windows-based DNS server can filter out the responses for these non-secure records"
- [CERT VU#109475](https://www.kb.cert.org/vuls/id/109475/): "the DNS server will cache any records in a response even if those records are outside the namespace delegated to the remote DNS server" (when Secure Cache is disabled; when enabled, such out-of-bailiwick records are discarded)

DNS Server then attempts to resolve the CNAME target independently via the general forwarder, which produces inconsistent results.

**Note:** The A record being discarded by Secure Cache is inference — we do not have a Wireshark capture on the DNS Server itself comparing inbound (from Alibaba) vs outbound (to client) packets. However, the evidence chain strongly supports this conclusion: if the A record were cached from Alibaba's response, the CNAME chase to the general forwarder would not occur.

---

## Root Cause

The issue was caused by two layers of problems:

**Layer 1 — Secure Cache Against Pollution + Conditional Forwarder scope mismatch:**
- Alibaba Cloud DNS returns CNAME chains crossing domain boundaries (`aliyun.com` → `alibabadns.com` → `aliyunga00xx.com`)
- Conditional Forwarder only covers `aliyun.com`
- Secure Cache (default enabled) discards records outside `aliyun.com` scope, including the final A records
- DNS Server is forced to do CNAME chase via general forwarder instead of using Alibaba's complete response

**Layer 2 — General forwarder inconsistent resolution:**
- The general forwarder (`10.111.125.34`) resolves `*.gds.alibabadns.com` inconsistently
- Sometimes returns CNAME + A → works
- Sometimes returns CNAME only → fails
- This explains the intermittent nature of the issue

Customer confirmed the general forwarder behavior was their issue.

---

## Resolution

### Option 1 (Recommended): Expand Conditional Forwarder coverage

```powershell
# On all 3 DNS Servers:
Add-DnsServerConditionalForwarderZone -Name "alibabadns.com" -MasterServers "Ali_DNS_IP1","Ali_DNS_IP2","Ali_DNS_IP3","Ali_DNS_IP4"
```

This ensures `*.gds.alibabadns.com` queries go to Alibaba Cloud DNS (stable) instead of the general forwarder (inconsistent).

### Option 2: Fix general forwarder resolution

Investigate why the general forwarder (`10.111.125.34`) returns inconsistent results for `*.gds.alibabadns.com`.

### Workaround for verification

```powershell
# Temporarily disable Secure Cache (no restart required):
Set-DnsServerCache -PollutionProtection $false
Clear-DnsServerCache
# Test, then restore:
Set-DnsServerCache -PollutionProtection $true
```

---

## References

- [KB241352 - How to Prevent DNS Cache Pollution](https://jeffpar.github.io/kbarchive/kb/241/Q241352/)
- [CERT VU#109475 - Microsoft Windows DNS non-authoritative RRs](https://www.kb.cert.org/vuls/id/109475/)
- [Set-DnsServerCache PowerShell Reference](https://learn.microsoft.com/en-us/powershell/module/dnsserver/set-dnsservercache?view=windowsserver2025-ps)
