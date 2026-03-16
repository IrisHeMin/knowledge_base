---
layout: post
title: "Case Summary: Windows 11 SMB File Copy Slowness over VPN — Win11 4x Slower than Win10"
date: 2026-01-06
categories: [SMB, Networking]
tags: [smb, vpn, windows-11, tcp, packet-loss, cubic-tcp, f5-vpn, cisco-anyconnect, procmon, wireshark, performance]
---

# Case Summary: Windows 11 通过 VPN 进行 SMB 文件复制速度慢 — Win11 比 Win10 慢 4 倍

**Product/Service:** Windows 11 / SMB / VPN (F5 & Cisco AnyConnect)

---

## 1. 症状 (Symptoms)

客户报告：
- 通过 VPN 从远程 SMB 共享 `\\filesrv01.contoso.corp\App-Deployment` 复制 162 MB 的 zip 文件到本地
- **Windows 11 机器耗时 5 分 18 秒**（吞吐量 ~2.3 MB/s）
- **Windows 10 机器仅需 1 分 17 秒**（吞吐量 ~6.9 MB/s）
- Win11 比 Win10 慢约 **4.1 倍**
- 问题持续存在，非偶发，影响所有通过 VPN 的 SMB 文件传输

## 2. 背景 (Background / Environment)

| 配置项 | Windows 10 机器 | Windows 11 机器 |
|--------|----------------|----------------|
| **主机名** | `PC-WIN10-01` | `PC-WIN11-01` |
| **OS 版本** | Windows 10 | Windows 11 24H2 (Build 26100) |
| **VPN 客户端** | **Cisco AnyConnect** | **F5 VPN** |
| **VPN 端点** | `192.0.2.1:443` (UDP) | `198.51.100.112:8080` |
| **客户端 IP** | 同网段 | `10.1.10.63` |
| **SMB 服务器** | `filesrv01.contoso.corp` (`10.2.20.98`) | 同左 |
| **SMB 协议** | SMB 2.x / 3.0.2 | SMB 3.1.1 |
| **TCP 拥塞控制** | Compound TCP (CTCP) | **Cubic TCP** |
| **EDR 产品** | Cybereason (58K 事件) | Cybereason (**840K 事件, 14.3x**) |
| **安全产品 I/O** | ~290K 事件 | **~1,454K 事件 (5x)** |

⚠️ **关键发现**: 两台机器使用**完全不同的 VPN 客户端**，这是一个重大变量。

## 3. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

Win11 SMB 文件复制慢是**多因素叠加**的结果：

| # | 因素 | 影响程度 |
|---|------|---------|
| 1 | **VPN 链路丢包/乱序** — Duplicate ACK 率 2-2.5%，177-425 次重传 | 🔴 **70%** |
| 2 | **Win11 Cubic TCP vs Win10 CTCP** — Cubic 对丢包更敏感，恢复更慢 | 🔴 **20%** |
| 3 | **VPN 客户端不同** (Cisco vs F5) — 隧道开销、MTU、路由策略不同 | 🔴 加剧因素 |
| 4 | **Cybereason EDR Win11 上 14.3 倍活跃** + 安全产品总负载 5 倍 | 🟡 加剧因素 |
| 5 | **Explorer 删除文件导致双重复制** (两台机器均有) | 🟡 中 |
| 6 | **TCP MSS 差异** (1,240 vs 1,350) 及 IP 分片 | 🟡 **5%** |
| 7 | **IP 访问导致 SMB 降级到 2.1** (SPN 不匹配) | 🟡 **5%** |

**核心洞察**: 在同等 VPN 丢包条件下，Win10 的 Compound TCP 能够快速恢复拥塞窗口维持较高吞吐，而 Win11 的 Cubic TCP 在每次检测到丢包时更激进地缩小拥塞窗口且恢复更慢，导致吞吐量低 36-47%。

### Resolution

**快速修复 — 切换 Win11 TCP 拥塞控制算法**：
```powershell
# 以管理员身份运行 PowerShell
# 将 Win11 TCP 拥塞控制从 Cubic 切换为 Compound TCP (Win10 默认)
netsh int tcp set global congestionprovider=ctcp

# 确保 TCP 自动调优为正常模式
netsh int tcp set global autotuninglevel=normal

# 验证设置
netsh int tcp show global
Get-NetTCPSetting | Select-Object SettingName, CongestionProvider, AutoTuningLevelLocal
```

> ⚡ **预期效果**: 恢复到接近 Win10 的吞吐水平 (~25 Mbps)，提升 30-50%

**其他建议操作**：

1. **调查 VPN 链路丢包根因**（优先级最高）
   ```powershell
   # 测试 VPN 丢包率
   Test-Connection -ComputerName 10.2.20.98 -Count 100 -BufferSize 1200
   Test-NetConnection -ComputerName 10.2.20.98 -TraceRoute
   ```
   - 检查 VPN 集中器 CPU/内存负载
   - 检查接口错误/丢包计数器
   - 检查 QoS 策略是否限制了 SMB (端口 445) 流量

2. **修复 MTU/MSS 问题**
   ```powershell
   netsh interface ipv4 show subinterfaces
   # 如果 MTU 过大导致分片，减小到 1300
   netsh interface ipv4 set subinterface "VPN_Interface" mtu=1300 store=persistent
   ```

3. **在同一 VPN 客户端下对比测试** — 排除 VPN 客户端本身的差异影响

4. **调查 Cybereason EDR 配置差异** — 为什么 Win11 上活跃度高 14.3 倍

5. **使用主机名而非 IP 地址访问** — 确保 Kerberos → SMB 3.1.1

### Workaround

- 使用 Robocopy 多线程复制提高吞吐:
  ```powershell
  robocopy "\\filesrv01.contoso.corp\App-Deployment\AppPackage_v54" "C:\local\path" /MT:8 /Z
  ```
- 考虑使用 HTTPS 文件传输替代 SMB（绕过 SMB over VPN 的限制）

## 4. 经验教训 (Lessons Learned)

### 技术知识

- **Windows 11 默认使用 Cubic TCP** vs Windows 10 的 Compound TCP — 在丢包环境下表现差异显著。Cubic 设计用于高带宽广域网但对丢包更敏感，CTCP 在丢包环境下恢复更快
- **通过 IP 地址访问 SMB 共享会导致 Kerberos SPN 不匹配** → 回退 NTLM → 可能降级到 SMB 2.1，失去 3.x 优化
- **TCP MSS 1,240 bytes + VPN 封装头 = 1,340 bytes > MTU 1,280** → 导致 IP 分片 → 放大丢包影响
- **同一 EDR 产品在不同 OS 上行为可能截然不同** — Cybereason 在 Win11 上全盘扫描 vs Win10 上仅操作本地数据库

### 排查方法

- **不要仅依赖过滤后的日志** — 过滤 CSV 导致遗漏关键信息（如 Cybereason 存在于 Win10）。必须分析完整日志
- **早期确认环境一致性** — 本 case 初始假设两台机器使用相同 VPN，浪费了排查时间。应在第一步就确认所有环境变量
- **三组抓包对比法**非常有效 — 同一目标，不同客户端/访问方式，快速定位差异点
- **TCP 拥塞控制算法虽然抓包不可见，但可通过行为推断** — Win10 重传更多 (1,122) 但吞吐更高，说明其窗口恢复更激进
- **ProcMon 进程事件分布统计**是排查安全产品干扰的最佳方法 — 量化每个安全产品的 I/O 事件数

### 排查命令速查

```powershell
# 查看当前 TCP 拥塞控制算法
Get-NetTCPSetting | Select-Object SettingName, CongestionProvider

# 切换 TCP 拥塞控制
netsh int tcp set global congestionprovider=ctcp

# 查看 TCP 自动调优级别
netsh int tcp show global

# 检查 SMB 签名配置
Get-SmbClientConfiguration | Select-Object RequireSecuritySignature, EnableSecuritySignature

# 检查 SMB 连接状态
Get-SmbConnection

# 检查网络接口 MTU
netsh interface ipv4 show subinterfaces

# VPN 丢包测试
Test-Connection -ComputerName <target_ip> -Count 100 -BufferSize 1200
```

### 预防措施

- Windows 11 部署前，评估 TCP 拥塞控制算法对现有 VPN 环境的影响
- 安全产品在 OS 升级后需要重新评估其行为和 I/O 影响
- SMB 共享访问统一使用主机名，避免 IP 地址（保持 Kerberos + SMB 3.x）

## 5. 参考文档 (References)

- [Troubleshoot slow SMB file transfer](https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/slow-file-transfer) — Official Microsoft guide for troubleshooting slow SMB transfers
- [Set-NetTCPSetting](https://learn.microsoft.com/en-us/powershell/module/nettcpip/set-nettcpsetting) — PowerShell cmdlet for modifying TCP settings including CongestionProvider
- [SMB Direct (RDMA)](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-direct) — SMB performance features and RDMA overview

---
---

# Case Summary: Windows 11 SMB File Copy Slowness over VPN — Win11 4x Slower than Win10

**Product/Service:** Windows 11 / SMB / VPN (F5 & Cisco AnyConnect)

---

## 1. Symptoms

Customer reported:
- Copying a 162 MB zip file from remote SMB share `\\filesrv01.contoso.corp\App-Deployment` to local disk via VPN
- **Windows 11 machine: 5 min 18 sec** (throughput ~2.3 MB/s)
- **Windows 10 machine: 1 min 17 sec** (throughput ~6.9 MB/s)
- Win11 approximately **4.1x slower** than Win10
- Issue is persistent and affects all SMB file transfers over VPN

## 2. Background / Environment

| Configuration | Windows 10 Machine | Windows 11 Machine |
|---------------|--------------------|--------------------|
| **Hostname** | `PC-WIN10-01` | `PC-WIN11-01` |
| **OS Version** | Windows 10 | Windows 11 24H2 (Build 26100) |
| **VPN Client** | **Cisco AnyConnect** | **F5 VPN** |
| **VPN Endpoint** | `192.0.2.1:443` (UDP) | `198.51.100.112:8080` |
| **Client IP** | Same subnet | `10.1.10.63` |
| **SMB Server** | `filesrv01.contoso.corp` (`10.2.20.98`) | Same |
| **SMB Protocol** | SMB 2.x / 3.0.2 | SMB 3.1.1 |
| **TCP Congestion** | Compound TCP (CTCP) | **Cubic TCP** |
| **EDR Product** | Cybereason (58K events) | Cybereason (**840K events, 14.3x**) |
| **Security I/O** | ~290K events | **~1,454K events (5x)** |

⚠️ **Critical Finding**: The two machines used **entirely different VPN clients** — a major uncontrolled variable.

## 3. Root Cause & Resolution

### Root Cause

The SMB copy slowness on Win11 is a **multi-factor compound issue**:

| # | Factor | Impact |
|---|--------|--------|
| 1 | **VPN link packet loss/reordering** — 2-2.5% Dup ACK rate | 🔴 **70%** |
| 2 | **Win11 Cubic TCP vs Win10 CTCP** — Cubic more sensitive to loss | 🔴 **20%** |
| 3 | **Different VPN clients** (Cisco vs F5) — different tunnel overhead, MTU, routing | 🔴 Amplifier |
| 4 | **Cybereason EDR 14.3x more active on Win11** + 5x total security I/O | 🟡 Amplifier |
| 5 | **Explorer deletes file causing double copy** (both machines) | 🟡 Medium |
| 6 | **TCP MSS difference** (1,240 vs 1,350) and IP fragmentation | 🟡 **5%** |
| 7 | **IP access downgrades SMB to 2.1** (SPN mismatch) | 🟡 **5%** |

**Core Insight**: Under identical VPN packet loss conditions, Win10's Compound TCP quickly recovers its congestion window maintaining higher throughput, while Win11's Cubic TCP aggressively reduces its window on each loss detection and recovers more slowly, resulting in 36-47% lower throughput.

### Resolution

**Quick Fix — Switch Win11 TCP Congestion Control**:
```powershell
# Run PowerShell as Administrator
netsh int tcp set global congestionprovider=ctcp
netsh int tcp set global autotuninglevel=normal

# Verify
netsh int tcp show global
Get-NetTCPSetting | Select-Object SettingName, CongestionProvider, AutoTuningLevelLocal
```

> ⚡ **Expected Result**: Restore throughput to near Win10 levels (~25 Mbps), 30-50% improvement

**Additional Recommended Actions**:
1. Investigate VPN link packet loss root cause (check concentrator load, interface errors, QoS)
2. Fix MTU/MSS: `netsh interface ipv4 set subinterface "VPN_Interface" mtu=1300 store=persistent`
3. Re-test with same VPN client on both machines to isolate VPN client impact
4. Investigate Cybereason configuration differences between the two machines
5. Use hostname (not IP) for SMB access to ensure Kerberos → SMB 3.1.1

### Workaround

```powershell
# Multi-threaded Robocopy
robocopy "\\filesrv01.contoso.corp\App-Deployment\AppPackage_v54" "C:\local" /MT:8 /Z
```

## 4. Lessons Learned

### Technical Knowledge
- **Windows 11 defaults to Cubic TCP** vs Windows 10's Compound TCP — significant performance difference under packet loss. Cubic is designed for high-bandwidth WANs but is more sensitive to loss
- **Accessing SMB shares by IP address causes Kerberos SPN mismatch** → NTLM fallback → potential SMB 2.1 downgrade
- **Same EDR product can behave drastically differently across OS versions** — Cybereason performed full-disk scanning on Win11 vs local-DB-only on Win10

### Troubleshooting Methods
- **Never rely on filtered logs alone** — filtered CSV missed critical information. Always analyze full unfiltered logs
- **Verify environment consistency early** — this case initially assumed same VPN on both machines, wasting investigation time
- **Three-capture comparison method** is highly effective — same target, different clients/access methods
- **TCP congestion control can be inferred from behavior** — higher retransmissions + higher throughput = more aggressive window recovery (CTCP)
- **ProcMon process event distribution statistics** are the best method to quantify security product interference

### Quick Reference Commands
```powershell
# Check TCP congestion control algorithm
Get-NetTCPSetting | Select-Object SettingName, CongestionProvider

# Switch TCP congestion control
netsh int tcp set global congestionprovider=ctcp

# Check SMB signing
Get-SmbClientConfiguration | Select RequireSecuritySignature, EnableSecuritySignature

# Check network interface MTU
netsh interface ipv4 show subinterfaces

# VPN packet loss test
Test-Connection -ComputerName <target> -Count 100 -BufferSize 1200
```

### Prevention
- Before Win11 deployment, evaluate TCP congestion control algorithm impact on existing VPN environment
- Re-evaluate security product behavior and I/O impact after OS upgrades
- Standardize SMB share access using hostnames (not IP) to maintain Kerberos + SMB 3.x

## 5. References

- [Troubleshoot slow SMB file transfer](https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/slow-file-transfer) — Official Microsoft guide for troubleshooting slow SMB transfers
- [Set-NetTCPSetting](https://learn.microsoft.com/en-us/powershell/module/nettcpip/set-nettcpsetting) — PowerShell cmdlet for modifying TCP settings including CongestionProvider
- [SMB Direct (RDMA)](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-direct) — SMB performance features and RDMA overview
