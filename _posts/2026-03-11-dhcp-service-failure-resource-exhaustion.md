---
layout: post
title: "DHCP 服务因系统资源耗尽停止工作导致楼宇用户无法获取 IP"
date: 2026-03-11
categories: [DHCP, Windows-Server]
tags: [dhcp, failover, jet-database, resource-exhaustion, esent, event-log, titan-agent, service-crash-loop]

---

# Case Summary: DHCP 服务因系统资源耗尽停止工作导致楼宇用户无法获取 IP

**Product/Service:** Windows Server 2016 — DHCP Server Role (Failover 配置)

---

## 1. 症状 (Symptoms)

- 使用该组 DHCP 服务的楼宇用户**无法获取到新 IP 地址**
- 检查备份 DHCP 服务器（dhcp-srv01.contoso.com）显示**地址池使用率 95%**，但实际地址池中并未分配那么多 IP
- 检查故障 DHCP 服务器（dhcp-srv02.contoso.com），发现 **DHCP 服务处于停止状态**
- 手动启动 DHCP 服务一直显示**"启动中"**，无法恢复
- 重启故障服务器后，DHCP 服务恢复正常，用户可正常获取 IP，地址池使用率也回归正常

## 2. 背景 (Background / Environment)

| 项目 | 详情 |
|---|---|
| 故障服务器 | dhcp-srv02.contoso.com |
| Failover 伙伴 | dhcp-srv01.contoso.com |
| 操作系统 | Windows Server 2016 Standard (Build 14393) |
| 虚拟化平台 | VMware VM |
| DHCP Scope | 10.51.141.0, 10.51.143.0, 172.17.81.0 |
| Failover 关系 | dhcp-srv01 与 dhcp-srv02 互为备份 |
| 服务器 Uptime | **463 天**（上次重启 2024-11-29，直到故障时未重启） |
| 安装的第三方软件 | Titan Agent Service for Windows, Symantec Endpoint Protection |

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

### 3.1 现场初步排查

1. **检查备份 DHCP 服务器 dhcp-srv01**
   - 发现地址池使用率显示 95%
   - 但实际活跃租约数远低于该比例
   - **结论：** 使用率虚高，可能与 Failover 状态异常有关

2. **检查故障 DHCP 服务器 dhcp-srv02**
   - DHCP 服务处于 **Stopped** 状态
   - 手动启动服务，卡在"启动中"无法完成
   - **结论：** 服务无法恢复，需要进一步排查原因

3. **重启故障服务器**
   - 管理员于 **2026-03-11 10:08** 通过 explorer.exe 发起重启
   - 系统因响应缓慢未能正常关机（Event 6008 非正常关机）
   - 重启后 DHCP 服务在 **10:16:01** 成功启动
   - Failover 状态在 **10:16:23** 恢复 NORMAL
   - 用户恢复正常获取 IP

### 3.2 Event Log 深度分析

通过分析 dhcp-srv02 的三个日志文件（system.evtx、DHCP-SERVER.evtx、application.evtx），发现了完整的故障链。

#### 3.2.1 System Log — DHCP 相关事件统计

| Event ID | 含义 | 次数 | 首次出现 | 最后出现 |
|---|---|---|---|---|
| **1010** | DHCP 数据库访问错误 | **470** | 2026-01-11 | 2026-03-07 05:50 |
| **1016** | DHCP 数据库清理错误 | **905** | 2026-01-11 | 2026-03-07 05:50 |
| **1020** | DHCP 判定自身未授权 | **3,502** | 2025-12-27 | 2026-03-11 13:16 |
| **1376** | 运行在 DC 上但无法确认授权 | **3,504** | 2025-12-27 | 2026-03-11 13:16 |
| **1054** | DHCP 致命错误，服务自行终止 | **1** | 2026-03-07 06:39 | — |
| **1059** | DNS 动态更新失败 | **15** | 2026-01-15 | 2026-03-07 05:39 |

Event 1010/1016 的错误码为 **0x4E2D (20013)**，对应 JET 数据库操作失败。

#### 3.2.2 System Log — Titan Agent Service 崩溃循环

| Event ID | 含义 | 次数 |
|---|---|---|
| **7000** | Titan Agent 启动失败 | **21,764** |
| **7031** | Titan Agent 意外终止 | **123**（日志中编号 #227→#349） |

**Event 7000 错误码分布：**

| 错误码 | 含义 | 次数 | 首次出现 |
|---|---|---|---|
| %%1053 | SERVICE_REQUEST_TIMEOUT（启动超时） | **20,948** | 2025-12-27 |
| %%1455 | NO_SYSTEM_RESOURCES（无系统资源） | **711** | 2026-01-09 |
| %%1450 | NO_SYSTEM_RESOURCES（无系统资源） | **38** | 2026-01-13 |
| %%8 | NOT_ENOUGH_MEMORY（内存不足） | **18** | 2026-01-13 |
| %%1054 | SERVICE_ALREADY_RUNNING | 42 | 2026-01-11 |

**Titan Agent 每日失败频率：**
- 2025-12-27 首日出现，368 次
- 2025-12-28 至 2026-01-10：连续 14 天每天 **576 次**（≈ 每 2.5 分钟崩溃重启一次）
- 2026-01-28 至 2026-02-07：再次连续 11 天每天 576 次
- SCM 配置：每次崩溃后 120 秒自动重启

**错误类型按周演进——资源耗尽的渐进过程：**

| 周次 | 总失败 | %%1053 超时 | %%1455 无资源 | %%8 内存不足 |
|---|---|---|---|---|
| W52 (12/27) | 944 | 944 (100%) | 0 | 0 |
| W01 (1月初) | 2,304 | 2,304 (100%) | 0 | 0 |
| W02 (1/9) | 4,031 | 4,023 | **6 ⚠️ 首现** | 0 |
| W03 (1/13) | 2,557 | 2,312 | **211 📈** | **7 ⚠️ 首现** |
| W10 (3月初) | 218 | 179 | 32 | 2 |

#### 3.2.3 System Log — 系统级资源耗尽

| 事件 | 次数 | 时间范围 | 说明 |
|---|---|---|---|
| Kernel-General Event 6（注册表刷新失败） | **43** | 2026-01-17 → 2026-03-07 | OS 内核无法将注册表写入磁盘 |
| DCOM Event 10010（服务超时） | **~880** | 持续 | DCOM 服务无法在规定时间注册 |

#### 3.2.4 Application Log — ESENT/JET 数据库错误

| Event ID | 含义 | 次数 | 关键数据 |
|---|---|---|---|
| **215** | DHCP 数据库备份中止 | **875** | svchost PID 1904, dhcp.mdb |
| **217** | 数据库备份失败 -1011 (JET_errOutOfMemory) | **3** | 2/25, 2/26, 2/28 |
| **482** | Windows Update DB 写入失败 | **1** | error 8 (NOT_ENOUGH_MEMORY), 3/1 |
| **104** | Windows Update JET 引擎停止 | **1** | error -1011, 3/3 |

ESENT Event 217 的 XML 明确记录：
```
进程: svchost (PID 1904)
错误: -1011 (JET_errOutOfMemory)
数据库: C:\Windows\system32\dhcp\dhcp.mdb
```

#### 3.2.5 DHCP-SERVER Log — Failover 状态

DHCP-SERVER.evtx 仅包含 **4 种事件**，共 1,951 条：

| Event ID | 含义 | 次数 |
|---|---|---|
| 20322 | DNS 动态更新失败 | 1,946 |
| 20251 | Failover 状态转换 | 3 |
| 20254 | Failover 连接建立 | 1 |
| 20259 | Partner 状态通知 | 1 |

**关键发现：所有 Failover 事件仅出现在重启后（2026-03-11 10:16:23）**

重启后的 Failover 状态转换：
```
STARTUP → COMMUNICATION_INT → NORMAL （数秒内完成）
```

**整个故障期间（3/7 → 3/11），Failover 没有被设置为 "Partner Down"。**
dhcp-srv01 一直处于 **COMMUNICATION_INT** 状态 → 这就是为什么地址池显示 95% 使用率。

### 3.3 DHCP 停止的直接机制

DHCP 服务是**自行停止**的（非崩溃），证据：
- **有 Event 1054**（DHCP 主动决定停止）→ 紧接 Event 7036（进入 stopped 状态）
- **没有 Event 7031/7034**（无意外终止或异常崩溃记录）

Event 1054 含义："The DHCP/BINL service has determined that it is not the valid/authorized DHCP server, or has encountered an error that prevents it from continuing to serve DHCP clients."

这是 Windows DHCP Server 的**内置安全机制**：如果 DHCP 无法在 AD 中验证自身授权状态，会主动停止以防成为"流氓 DHCP 服务器"。

崩溃前最后 2 小时的事件序列：
```
05:50:44  Event 1016/1010 — DHCP 数据库访问失败 (0x4E2D)
05:50:44  Event 1020/1376 — DHCP 无法验证 AD 授权
05:39:33  Event 1059 — DNS 动态更新失败（最后一次）
05:40:42  Kernel Event 6 — 注册表刷新失败（最后一次）
06:39:33  Event 1054 — DHCP 致命错误，决定自行终止 💀
06:40:28  Event 7036 — "DHCP Server service entered the stopped state"
```

### 3.4 备份服务器地址池 95% 使用率的原因

当 dhcp-srv02 停止工作后，dhcp-srv01 检测到通信丢失，自动进入 **COMMUNICATION_INT**（通信中断）状态：

| 特征 | COMMUNICATION_INT 状态 | Partner Down 状态 |
|---|---|---|
| 触发方式 | 自动（检测到通信丢失） | 需管理员手动设置 |
| 可用地址 | 仅自己分配的那部分（~50%） | 可使用全部地址池 |
| 租约清理 | 不清理对方的租约 | MCLT 后可回收对方租约 |
| 本次情况 | ✅ 正是此状态 | ❌ 未执行 |

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| DHCP 服务无法手动启动（卡在"启动中"） | 服务 4 天无法恢复（3/7 → 3/11） | 重启服务器后恢复 |
| 系统资源全面耗尽，无法创建新线程/分配内存 | 不仅 DHCP，多个系统服务均受影响 | 重启服务器释放所有资源 |
| 备份服务器未设置为 Partner Down | 存活服务器地址池受限，部分用户无法获取 IP | 故障服务器重启后 Failover 自动恢复 NORMAL |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

**根本原因：Titan Agent Service for Windows 从 2025-12-27 起持续崩溃重启，75 天内产生 21,764 次启动失败，以每 2.5 分钟一次的频率反复消耗系统资源，最终导致服务器全面资源耗尽。**

完整因果链：

```
Titan Agent 崩溃循环（75天, 21,764次, 每2.5分钟一次）
  ↓ 每次循环：分配进程内存 → 启动超时 → SCM 终止 → 资源未完全释放
系统资源逐步耗尽（内存/线程/页面文件/句柄）
  ↓ 1/9 首现 NO_SYSTEM_RESOURCES, 1/13 首现 NOT_ENOUGH_MEMORY
DHCP JET 数据库无法操作（dhcp.mdb, error -1011 JET_errOutOfMemory）
  ↓ 1/11 开始，累计 1,375 次数据库访问错误 + 875 次备份失败
DHCP 无法查询 AD 验证授权（LDAP/RPC 调用因资源不足失败）
  ↓ Event 1020 × 3,502 + Event 1376 × 3,504
DHCP 内置安全机制触发 → 服务自行终止（Event 1054）
  ↓ 2026-03-07 06:39
DHCP 手动重启也失败（系统连创建新线程都做不到）
  ↓ 4 天停机
重启服务器后一切恢复（内存释放，JET 数据库正常打开）
```

**加剧因素：服务器 463 天（2024-11-29 起）未重启**，泄漏的资源永远无法被回收。

### Resolution

1. 管理员于 2026-03-11 10:08 重启故障服务器 dhcp-srv02
2. 服务器于 10:15:51 完成启动
3. DHCP 服务于 10:16:01 成功启动
4. Failover 于 10:16:23 恢复 NORMAL 状态，两台服务器同步租约数据库
5. 用户恢复正常获取 IP，地址池使用率回归正常

### 后续建议

| 优先级 | 操作 |
|---|---|
| 🔴 紧急 | **修复或移除 Titan Agent Service** — 该第三方 Agent 是根因，从未成功启动，需联系 Titan 供应商排查或禁用该服务 |
| 🔴 紧急 | **建立定期重启计划** — 463 天未重启导致资源泄漏累积，建议每月或每季度维护重启 |
| 🟠 高 | **配置 DHCP 服务监控告警** — 监控 Event 1010/1016/1054，在数据库错误阶段即发出预警 |
| 🟠 高 | **制定 Failover "Partner Down" 操作文档** — 当确认 partner 宕机时，应在存活服务器上执行 `Set-DhcpServerv4Failover -PartnerDown` 释放全部地址池 |
| 🟡 中 | **增大页面文件** — 多次出现 "paging file too small" 错误 |
| 🟡 中 | **考虑升级操作系统** — Server 2016 即将结束扩展支持 |

## 6. 经验教训 (Lessons Learned)

### 技术知识
- **DHCP Event 1054 的真正含义**：不仅仅是"未授权"，当 DHCP 无法访问 AD（因为资源不足导致 LDAP/RPC 失败）时，也会触发此安全机制自行终止
- **DHCP Failover COMMUNICATION_INT 状态**：存活服务器只能使用约 50% 的地址池，不会自动扩展。必须手动设置 Partner Down 才能释放全部地址
- **JET_errOutOfMemory (-1011)**：当系统内存不足时，DHCP 使用的 JET（ESENT）数据库引擎无法正常操作 dhcp.mdb
- **Event 7000 错误码演进**：%%1053（超时）→ %%1455/%%1450（无资源）→ %%8（内存不足）的渐进变化，是系统资源逐步耗尽的明确信号

### 排查方法
- **Event Log XML 解析**：当 Message 字段为空时，通过 `$event.ToXml()` 解析 `<EventData>` 获取完整参数
- **Binary 字段解码**：Event 1010/1016 的 Binary 字段 `2D4E0000` 为小端序，实际错误码为 `0x00004E2D = 20013`
- **关联分析**：DHCP 停止本身只是表象，需关联 SCM 事件（7000/7031）、ESENT 事件（215/217）、内核事件（Kernel-General 6）还原完整过程
- **按周聚合错误类型**：通过观察错误码的"演进"精确判断资源耗尽的时间节点

### 预防措施
- 为 DHCP 服务器设置定期维护重启窗口
- 监控系统资源（内存、句柄、线程数），设置阈值告警
- 对第三方 Agent 的 SCM 恢复策略设置最大重试次数（如 3 次后停止重试）
- 定期审查 Event Log 中的 7000/7031 事件，及时发现服务崩溃循环

## 7. 参考文档 (References)

暂无可验证的参考文档。

---
---

# Case Summary: DHCP Service Stopped Due to System Resource Exhaustion Causing IP Allocation Failure

**Product/Service:** Windows Server 2016 — DHCP Server Role (Failover Configuration)

---

## 1. Symptoms

- Users in the building served by this DHCP pair **could not obtain new IP addresses**
- The backup DHCP server (dhcp-srv01.contoso.com) showed **95% address pool utilization**, but actual lease count did not match
- The faulty DHCP server (dhcp-srv02.contoso.com) had its **DHCP service in a Stopped state**
- Manual attempts to start the DHCP service **stuck at "Starting"** and never completed
- After rebooting the faulty server, DHCP service recovered, users could obtain IPs, and pool utilization returned to normal

## 2. Background / Environment

| Item | Details |
|---|---|
| Faulty Server | dhcp-srv02.contoso.com |
| Failover Partner | dhcp-srv01.contoso.com |
| Operating System | Windows Server 2016 Standard (Build 14393) |
| Virtualization | VMware VM |
| DHCP Scopes | 10.51.141.0, 10.51.143.0, 172.17.81.0 |
| Failover Relationship | dhcp-srv01 and dhcp-srv02 in Hot Standby/Load Balance |
| Server Uptime | **463 days** (last reboot 2024-11-29) |
| Third-Party Software | Titan Agent Service for Windows, Symantec Endpoint Protection |

## 3. Investigation & Troubleshooting

### 3.1 Initial On-Site Investigation

1. **Checked backup DHCP server dhcp-srv01**
   - Address pool showed 95% utilization
   - Actual active leases were far fewer
   - **Conclusion:** Inflated utilization, likely related to abnormal failover state

2. **Checked faulty DHCP server dhcp-srv02**
   - DHCP service was in **Stopped** state
   - Manual start attempt hung at "Starting" indefinitely
   - **Conclusion:** Service unrecoverable, deeper investigation needed

3. **Rebooted the faulty server**
   - Administrator initiated reboot at **2026-03-11 10:08** via explorer.exe
   - System was too unresponsive for clean shutdown (Event 6008)
   - DHCP service started successfully at **10:16:01**
   - Failover returned to NORMAL at **10:16:23**

### 3.2 Deep Event Log Analysis

#### 3.2.1 Titan Agent Service Crash Loop — The Root Cause

The **Titan Agent Service for Windows** had been in a continuous crash-restart loop since **2025-12-27**:

- **21,764 startup failures** (Event 7000) over 75 days
- **123 unexpected terminations** (Event 7031, numbered #227→#349)
- Restart interval: 120 seconds → **576 failures per day** (every 2.5 minutes)

Error code evolution showing progressive resource exhaustion:

| Week | Total | %%1053 Timeout | %%1455 No Resources | %%8 No Memory |
|---|---|---|---|---|
| W52 (12/27) | 944 | 944 (100%) | 0 | 0 |
| W02 (1/9) | 4,031 | 4,023 | **6 ⚠️ First** | 0 |
| W03 (1/13) | 2,557 | 2,312 | **211 📈** | **7 ⚠️ First** |
| W10 (Mar) | 218 | 179 | 32 | 2 |

#### 3.2.2 DHCP Database Failures

| Event ID | Meaning | Count | Key Data |
|---|---|---|---|
| 1010/1016 | Database access/cleanup error | 1,375 | Error 0x4E2D, from 2026-01-11 |
| ESENT 215 | DHCP backup halted | 875 | from 2026-01-11 |
| ESENT 217 | JET_errOutOfMemory (-1011) | 3 | 2/25, 2/26, 2/28 |

#### 3.2.3 DHCP Service Self-Termination

The DHCP service **stopped itself gracefully** (not a crash):
- **Event 1054 present** → followed by Event 7036 (stopped state)
- **No Event 7031/7034** (no unexpected termination)
- This is DHCP's built-in safety mechanism: stops when it cannot verify AD authorization

#### 3.2.4 Failover State

- **No "Partner Down" state was ever set** during the 4-day outage
- dhcp-srv01 remained in **COMMUNICATION_INT** → explaining 95% pool utilization
- All failover events only appeared after reboot: STARTUP → COMMUNICATION_INT → NORMAL

### 3.3 How DHCP Service Stopped — The Mechanism

```
05:50:44  Event 1016/1010 — Database access failed (0x4E2D)
05:50:44  Event 1020/1376 — Cannot verify AD authorization
06:39:33  Event 1054 — Fatal error, self-terminate decision 💀
06:40:28  Event 7036 — "DHCP Server service entered the stopped state"
```

## 4. Blockers & How They Were Resolved

| Blocker | Impact | Resolution |
|---------|--------|-----------|
| DHCP service stuck at "Starting" | 4-day outage (3/7 → 3/11) | Server reboot |
| System-wide resource exhaustion | Multiple services affected | Server reboot released all resources |
| Backup server not set to Partner Down | Address pool restricted to ~50% | Failover auto-recovered after reboot |

## 5. Root Cause & Resolution

### Root Cause

**The Titan Agent Service for Windows had been in a continuous crash-restart loop since 2025-12-27, generating 21,764 startup failures over 75 days at a rate of once every 2.5 minutes, progressively exhausting all system resources.**

```
Titan Agent crash loop (75 days, 21,764 failures, every 2.5 minutes)
  ↓ Each cycle leaks memory/handles
System resource exhaustion (memory/threads/page file/handles)
  ↓
DHCP JET database fails (dhcp.mdb, JET_errOutOfMemory -1011)
  ↓
DHCP cannot verify AD authorization
  ↓
DHCP safety mechanism → self-terminates (Event 1054)
  ↓
Manual restart fails (no resources to create threads)
  ↓
Server reboot restores everything
```

**Aggravating factor: 463 days without reboot** — leaked resources never reclaimed.

### Resolution

1. Server rebooted at 2026-03-11 10:08
2. DHCP service started at 10:16:01
3. Failover recovered to NORMAL at 10:16:23
4. Users resumed normal IP acquisition

### Recommendations

| Priority | Action |
|---|---|
| 🔴 Critical | **Fix or remove Titan Agent Service** — never successfully started; contact vendor or disable |
| 🔴 Critical | **Establish regular reboot schedule** — prevent resource leak accumulation |
| 🟠 High | **Configure DHCP monitoring** — alert on Event 1010/1016/1054 |
| 🟠 High | **Document Failover "Partner Down" procedure** |
| 🟡 Medium | Increase page file size, consider OS upgrade |

## 6. Lessons Learned

### Technical Knowledge
- **Event 1054** triggers not only for unauthorized servers, but also when resource starvation prevents AD authorization checks
- **Failover COMMUNICATION_INT** limits surviving server to ~50% of address pool; manual Partner Down is required
- **JET_errOutOfMemory (-1011)** directly caused by system memory exhaustion
- **Event 7000 error code progression** (%%1053 → %%1455 → %%8) signals gradual resource depletion

### Troubleshooting Methods
- Parse Event Log XML (`$event.ToXml()`) when Message fields are empty
- Decode Binary fields (little-endian): `2D4E0000` → `0x4E2D = 20013`
- Cross-log correlation: SCM + ESENT + Kernel events reveal complete picture
- Weekly error aggregation identifies resource exhaustion timeline

### Prevention
- Regular maintenance reboot windows for infrastructure servers
- System resource monitoring with threshold alerts
- Set maximum retry count on third-party agent SCM recovery actions
- Audit Event Log 7000/7031 events regularly to detect crash loops early

## 7. References

No verified reference documents available.
