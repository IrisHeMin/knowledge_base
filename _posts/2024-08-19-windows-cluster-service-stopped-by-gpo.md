---
layout: post
title: "Windows Failover Cluster Service Intermittently Stopped — Root Cause: GPO Misconfiguration"
date: 2024-08-19
categories: [Windows-Server, Failover-Clustering]
tags: [cluster, gpo, group-policy, clussvc, scm, sysmon, windows-server-2022, sql-cluster, quorum]

---

# Case Summary: Windows Failover Cluster Service Intermittently Stopped by GPO

**Product/Service:** Windows Server 2022 Failover Clustering / SQL Server Cluster

---

## 中文版本

### 1. 症状 (Symptoms)

- 客户新搭建的两节点 Windows Server 2022 Failover Cluster（用于 SQL Server 迁移）中，Cluster Service (clussvc) **间歇性自动停止并被设置为 Disabled 状态**。
- 两个节点上都会发生此问题，但**不会同时发生**。
- 启动集群服务后，服务运行一段时间后会自动停止并切换为 disabled 状态。
- 该环境是**全新搭建**的，从未正常工作过。
- 问题紧迫：客户计划在周末将 SQL 2012 数据库迁移到此新集群。

### 2. 背景 (Background / Environment)

- **操作系统**：Windows Server 2022
- **集群配置**：两节点 Failover Cluster（节点：sqlnode01, sqlnode02）
- **用途**：SQL Server 集群，计划用于 SQL 2012 数据库迁移
- **网络配置**：
  - 节点 1 (sqlnode01) IP: 10.10.20.18
  - 节点 2 (sqlnode02) IP: 10.10.148.18
  - 集群通信端口：3343
  - 防火墙已开放端口：135, 3343, 445, 5022, 49152-65535, 137
- **初始状态**：未配置 Quorum Witness（两节点集群缺少仲裁见证）

### 3. Troubleshooting 过程 (Investigation & Troubleshooting)

#### Phase 1：初始日志分析（7/26）

**Action Plan：** 检查客户上传的集群验证报告、集群日志和事件日志。要求客户额外收集：
```powershell
# 收集防火墙和网络配置
Get-NetFirewallProfile > C:\temp\fwprofile.txt
Get-NetFirewallRule > C:\temp\FWRule.txt
ipconfig /all > C:\temp\ip.txt
```

**日志分析 — 集群验证报告（无 Quorum Witness）：**
```
Start: 24/07/2024 5:44:23 PM.
Validating cluster quorum settings.
Witness Type: No Witness Configured
Witness Resource: No Witness Configured
The cluster is not configured with a quorum witness. As a best practice, configure a 
quorum witness to help achieve the highest availability of the cluster.
```

**日志分析 — 集群日志（clussvc 反复被停止并设为 disabled）：**
```
360329 [Operational] 000036ec.000019ac::2024/07/07-12:16:59.868 INFO  The cluster service has been stopped and set as disabled as part of cluster node cleanup.
360338 [Operational] 00001b50.000035c8::2024/07/09-04:32:31.105 INFO  The cluster service has been stopped and set as disabled as part of cluster node cleanup.
360347 [Operational] 00002418.00000ea4::2024/07/10-03:36:28.779 INFO  The cluster service has been stopped and set as disabled as part of cluster node cleanup.
360356 [Operational] 0000141c.0000427c::2024/07/10-03:43:59.926 INFO  The cluster service has been stopped and set as disabled as part of cluster node cleanup.
360365 [Operational] 00003544.000042f4::2024/07/10-03:54:05.214 INFO  The cluster service has been stopped and set as disabled as part of cluster node cleanup.
360690 [Operational] 000040d4.00003778::2024/07/16-02:38:39.674 INFO  The cluster service has been stopped and set as disabled as part of cluster node cleanup.
```

**日志分析 — 集群日志（节点间 3343 端口连接错误 + 心跳丢失）：**

sqlnode01 → sqlnode02 连接失败：
```
495454 00001320.00002da0::2024/07/24-07:35:14.032 INFO  [NETWORK] Error when processing connection to node sqlnode02, error (10060)
495457 00001320.0000292c::2024/07/24-07:35:15.037 INFO  [NETWORK] Error when processing connection to node sqlnode02, error (10060)
...
495590 00001320.00002da0::2024/07/24-07:35:40.790 INFO  [NETWORK] Processing initial connection to node sqlnode02
495969 00001320.00002cf8::2024/07/24-07:35:41.115 INFO  [NETWORK] Connected and authenticated to node sqlnode02, now considering for cluster participation
496115 00001320.000022ec::2024/07/24-07:35:42.041 INFO  [NETWORK] Full mesh connectivity with nodes sqlnode01, sqlnode02
497896 00001320.00001724::2024/07/24-07:47:30.789 WARN  [NETWORK] Missed 40% of the heart beats with node 'sqlnode02' (2) on route 10.10.20.18:~3343~->10.10.148.18:~3343~
498334 00001320.00002cf8::2024/07/24-07:56:35.251 WARN  [NETWORK] Lost connection to node sqlnode02
```

最终导致 graceful shutdown：
```
498701 00001320.00000964::2024/07/24-08:00:29.857 INFO  [NETWORK] Local node is shutting down, disconnecting from other nodes
498708 0000264c.00000738::2024/07/24-08:00:29.973 WARN  [RHS] Cluster service has terminated. Cluster.Service.Running.Event got signaled.
```

**结论**：缺少 Quorum Witness + 节点间 3343 连接不稳定，导致集群服务频繁停止。

**Action Plan（给客户）：**
1. 配置 Quorum Witness（Cloud Witness / File Share Witness / Disk Witness）
2. 确保两节点网络配置一致，防火墙 Public Profile 允许入站和出站

> 配置 Cloud Witness 步骤：
> - 打开 Failover Cluster Manager → 右键集群名 → More Actions → Configure Cluster Quorum Settings
> - 选择 Select the quorum witness → Configure a cloud witness
> - 输入 Azure Storage Account Name、Access Key 和 Endpoint URL

---

#### Phase 2：配置 Cloud Witness 后仍有问题（8/8 - 8/9）

客户配置了 Cloud Witness 后，集群仍然间歇性失败。

**日志分析 — sqlnode01 集群日志（连接失败 → 服务关闭）：**
```
445773 00003948.00003a38::2024/08/08-06:50:05.236 INFO  ---+ LOG BEGIN +---
446134 00003948.00003338::2024/08/08-06:50:26.487 INFO  [NETWORK] Error when processing connection to node sqlnode02, error (10060)
446137 00003948.000037e8::2024/08/08-06:50:27.487 INFO  [NETWORK] Error when processing connection to node sqlnode02, error (10060)
446185 00003948.00003338::2024/08/08-06:51:27.487 INFO  [NETWORK] Error when processing connection to node sqlnode02, error (10060)
...
446552 00003948.00003338::2024/08/08-06:56:08.842 INFO  [NETWORK] Processing initial connection to node sqlnode02
446940 00003948.00003f6c::2024/08/08-06:56:09.165 INFO  [NETWORK] Connected and authenticated to node sqlnode02
447099 00003948.000037e8::2024/08/08-06:56:10.088 INFO  [NETWORK] Full mesh connectivity with nodes sqlnode01, sqlnode02
449570 00003948.000022bc::2024/08/08-07:31:58.449 INFO  [NETWORK] Local node is shutting down, disconnecting from other nodes
449577 00003818.000021c4::2024/08/08-07:31:58.569 WARN  [RHS] Cluster service has terminated.
```

**日志分析 — sqlnode02 集群日志（连接失败 → Cloud Witness offline → 服务关闭）：**
```
124064 000033dc.00003078::2024/08/08-07:55:41.151 INFO  [NODE] Node 2: New join with n1: stage: 'Attempt Initial Connection' status (10060) reason: 'Failed to connect to remote endpoint 10.10.20.18:~3343~'
124096 000033dc.000023d4::2024/08/08-07:56:57.095 INFO  [CS] Service Stopping...
124098 000033dc.000023d4::2024/08/08-07:56:57.095 INFO  [NODE] Node 2: Farthest reported progress joining with node sqlnode01 (id 1) is: Attempt Initial Connection at time 2024/08/08-07:56:41.153: status 10060 Failed to connect to remote endpoint 10.10.20.18:~3343~
124100 000033dc.00001e68::2024/08/08-07:56:57.095 INFO  [CORE] Graceful shutdown reported by node sqlnode02, reason ServiceStopReason::ServiceShutdown, nodeIsUp false (payload 327682)
```

Cloud Witness 也随之 offline：
```
124193 000033dc.00002e2c::2024/08/08-07:56:57.301 INFO  [RCM] HandleMonitorReply: OFFLINERESOURCE for 'Cloud Witness'
124203 000033dc.00003288::2024/08/08-07:56:57.301 WARN  [QUORUM] Node 2: One off quorum (2)
124209 000033dc.00003288::2024/08/08-07:56:57.301 INFO  [QUORUM] Node 2: death timer is already running, 90 seconds left.
```

**结论**：Cloud Witness 已配置但集群服务仍然停止。3343 连接失败是节点间的问题，但需要更深层排查。

---

#### Phase 3：缩小范围 — 单节点测试（8/12）

**Action Plan：** 通过隔离单节点来排除集群交互层面的问题。

```powershell
# Step 1: 提升集群日志级别到 5
Set-ClusterLog –Level 5

# Step 2: 禁用并停止另一个节点的集群服务（只保留一个节点运行）
# 在另一个节点上执行：
Stop-Service clussvc
Set-Service clussvc -StartupType Disabled
```

**排查逻辑：**
- 如果单节点集群服务**不再停止** → 问题与节点间通信相关（网络/节点加入重试）
- 如果单节点集群服务**仍然停止** → 问题在 OS 层面，有其他进程在停止集群服务

**结果**：即使只有一个节点，集群服务仍然间歇性停止。

**结论**：问题**不在集群层面**，而是 OS 层面有其他进程/应用在停止集群服务。

---

#### Phase 4：检查计划任务（8/12 - 8/13）

**日志分析 — 集群日志发现 restart task 调度记录：**
```
518370 00002f48.000014e0::2024/08/12-17:26:38.404 DBG   [RCM] scheduling next RunRestartTasks task...
518371 00002f48.0000360c::2024/08/12-17:26:46.688 DBG   [CORE] Service Alive, Checking Core Lock
518378 00002f48.0000360c::2024/08/12-17:26:48.405 DBG   [RCM] scheduling next RunRestartTasks task...
518380 00002f48.0000364c::2024/08/12-17:26:53.682 DBG   [NODE] Node 1: eating message sent to the dead node 2
518381 00002f48.0000364c::2024/08/12-17:26:53.682 INFO  [CS] Service Stopping...
518383 00002f48.00002c9c::2024/08/12-17:26:53.682 INFO  [CORE] Graceful shutdown reported by node sqlnode01, reason ServiceStopReason::ServiceShutdown, nodeIsUp false (payload 327681)
```

**Action Plan：** 导出计划任务列表检查
```cmd
schtasks /query > C:\task.txt
```

**结果**：计划任务输出不够清晰，无法明确定位根因。

**结论**：需要更深层 OS 级别的日志来追踪是谁在停止集群服务。

---

#### Phase 5：部署 SCM + Sysmon 增强日志收集（8/13）

**Action Plan（完整步骤）：**

**Step 1：安装 Sysmon 监控所有进程创建/终止**
```cmd
:: 1. 下载 Sysmon: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
:: 2. 解压到 C:\temp
:: 3. 以管理员权限打开 CMD，导航到 Sysmon 路径，执行安装：
sysmon -I -accepteula

:: 4. 验证安装成功：
fltmc

:: 5. 启用进程监控模式：
sysmon -m

:: 安装后 Sysmon 事件位于：
:: Event Viewer → Application and Services Logs → Microsoft → Windows → Sysmon → Operational
```

**Step 2：启用 SCM (Service Control Manager) ETL 日志**
```cmd
:: 6. 以管理员权限打开另一个 CMD，执行以下命令启动 SCM 日志收集：
logman create trace "base_screg" -ow -o c:\base_screg.etl -p "Service Control Manager" 0xffffffffffffffff 0xff -nb 16 16 -bs 1024 -mode Circular -f bincirc -max 4096 -ets
logman update trace "base_screg" -p "Microsoft-Windows-Services" 0xffffffffffffffff 0xff -ets
logman update trace "base_screg" -p {EBCCA1C2-AB46-4A1D-8C2A-906C2FF25F39} 0xffffffffffffffff 0xff -ets
```

**Step 3：配置自动停止日志（Task Scheduler + 脚本）**

创建 `C:\temp\StopLogging.bat`：
```bat
logman stop "base_screg" -ets
```

创建 `C:\temp\SCMLoggingCapture.ps1`：
```powershell
$serviceName = "clussvc"
$serviceStatus = Get-Service -Name $serviceName | Select-Object -ExpandProperty Status
if ($serviceStatus -ne "Running") {
    Start-Process -FilePath "C:\Temp\StopLogging.bat"
}
```

配置 Task Scheduler：
1. 打开 Task Scheduler → Create Task
2. Triggers 选项卡 → New → On a schedule → 间隔 5 分钟
3. Actions 选项卡 → New → Start a program → PowerShell.exe → 参数填 `C:\temp\SCMLoggingCapture.ps1`

**需要收集的日志清单：**
- Sysmon 事件日志
- 全部事件日志：`%SystemRoot%\System32\Winevt\Logs`
- SCM ETL 日志：`C:\base_screg.etl`
- 集群日志
- 进程列表：`tasklist /svc > C:\tasklist.txt`

---

#### Phase 6：分析 SCM + Sysmon 日志 — 关键发现（8/14 - 8/15）

这是整个排查过程中最关键的一步。通过三层日志的交叉关联，定位到了停止集群服务的确切进程。

**日志分析 — 第 1 层：SCM ETL 日志（谁停止了 clussvc）**

SCM ETL 日志记录了 clussvc 被停止的完整过程：
```
427637 [2] 03D4.2B8C::08/14/24-14:25:40.4517713 [SCM] CWin32ServiceRecord::SetStatus: Service Record updated with new status for service ClusSvc
427638 [2]03D4.2B8C::08/14/24-14:25:40.4517754 [Microsoft-Windows-Services/Diagnostic] CurrentState=3, StartType=2, PID=15144, ServiceName=ClusSvc
427639 [6] 03D4.46C0::08/14/24-14:25:40.4519111 [SCM] ScSendControl: Service ClusSvc, control 1, user S-1-5-18, caller PID 0x000003d4
427681 [6] 03D4.46C0::08/14/24-14:25:40.4841063 [SCM] ScSendControl: Sending control 0x00000051 to service/module VSS, caller PID 0x00000474, channel 0x0000019DA5713160
427692 [4] 03D4.46C0::08/14/24-14:25:40.4843767 [SCM] ScSendControl: Successfully sent start control to service VSS, user S-1-5-18
427729 [0] 03D4.46C0::08/14/24-14:25:40.6130898 [SCM] RSetServiceStatus: service ClusSvc, state 0x00000003(SERVICE_STOP_PENDING)
427732 [0] 03D4.46C0::08/14/24-14:25:40.7958679 [SCM] RSetServiceStatus: service ClusSvc, state 0x00000001(SERVICE_STOPPED)
```

**关键分析：**
- `caller PID 0x000003d4` → **0x3D4 十六进制转十进制 = 980**
- PID 980 是谁？ → 需要从 Sysmon 确认

**日志分析 — 第 2 层：Sysmon 日志（确认 PID 980 = services.exe + clussvc 终止）**

clussvc.exe 进程终止记录：
```
Process terminated:
RuleName: -
UtcTime: 2024-08-14 06:25:40.787
ProcessGuid: {3a33b69e-4081-66bc-4246-000000001700}
ProcessId: 15144
Image: C:\Windows\Cluster\clussvc.exe
User: NT AUTHORITY\SYSTEM
```

**日志分析 — 第 3 层：Sysmon 日志（PID 980 = services.exe，同时创建了 VSSVC.exe）**

用 PID 980 过滤 Sysmon 日志，发现 services.exe 在停止 clussvc 的同时创建了 VSSVC.exe 进程：
```
Process Create:
RuleName: -
UtcTime: 2024-08-14 06:25:40.446
ProcessGuid: {3a33b69e-4de4-66bc-e946-000000001700}
ProcessId: 17080
Image: C:\Windows\System32\VSSVC.exe
FileVersion: 10.0.20348.2520
Description: Microsoft® Volume Shadow Copy Service
CommandLine: C:\Windows\system32\vssvc.exe
User: NT AUTHORITY\SYSTEM
ParentProcessId: 980
ParentImage: C:\Windows\System32\services.exe
ParentCommandLine: C:\Windows\system32\services.exe
```

**三层日志关联分析总结：**

| 时间戳 | 事件 | 来源 |
|--------|------|------|
| 14:25:40.451 | SCM 更新 ClusSvc 状态，caller PID 0x3D4 (980) | SCM ETL |
| 14:25:40.446 | services.exe (PID 980) 创建 VSSVC.exe (PID 17080) | Sysmon |
| 14:25:40.484 | SCM 发送启动控制到 VSS 服务 | SCM ETL |
| 14:25:40.613 | ClusSvc 状态变为 SERVICE_STOP_PENDING | SCM ETL |
| 14:25:40.787 | clussvc.exe (PID 15144) 进程终止 | Sysmon |
| 14:25:40.795 | ClusSvc 状态变为 SERVICE_STOPPED | SCM ETL |

**结论**：是 **SCM (services.exe)** 自身在停止集群服务。VSS 在同一时间被创建，可能是相关联或巧合。

---

#### Phase 7：排除 VSS 因素（8/15）

**Action Plan：** 临时禁用并停止 VSS 服务，观察集群服务是否仍然停止。

**结果**：禁用 VSS 后，问题依然存在。

**结论**：VSS 不是根因，只是在同一时间点被 SCM 启动的巧合。需要继续追查 **SCM 为什么将 clussvc 设为 disabled**。

---

#### Phase 8：TSS 工具数据收集遇阻（8/15 - 8/16）

**Action Plan：** 使用 MS TSS 工具收集更详细的集群诊断日志。

```powershell
# 1. 下载 TSS: https://aka.ms/getTSS
# 2. 解压到 C:\temp\TSS
# 3. 以管理员 PowerShell 执行：
Set-ExecutionPolicy -ExecutionPolicy Bypass -force -Scope Process
.\TSS.ps1 -SHA_MsCluster -Procmon -WPR General -WaitEvent Process:clussvc
```

**客户遇到的问题：**
- 参数错误：`-WPR General-WaitEvent` 被当作一个参数（缺少空格）
- 脚本路径错误：未切换到 TSS 目录

**结论**：需要通过 Teams 会议实时协助。

---

#### Phase 9：Teams 会议实时排查 — 根因发现（8/16） 🎯

**Action Plan：** 远程协助，使用 GPO 诊断工具比较两个节点的配置差异。

```cmd
:: 生成 HTML 格式的 GPO 报告
gpresult /SCOPE computer /H GPRESULT.HTML

:: 列出所有应用的 GPO
GPRESULT /R
```

**GPO 比较结果 — 正常节点 vs 异常节点：**

| 比较项 | 正常工作的节点 (sqlnode01) | 不正常的节点 (sqlnode02) |
|--------|--------------------------|--------------------------|
| Winning GPO | **"Enable cluster service"** | **"Default group policy"** |
| Cluster Service 启动类型 | **Automatic** | **Disabled** |
| OU 位置 | 包含集群策略的 OU | 未包含集群策略的默认 OU |

**GPRESULT /R 输出对比：**
- 正常节点的 Applied Group Policy Objects 列表中包含 **"Enable cluster service"**
- 异常节点的 Applied Group Policy Objects 列表中**不包含**此策略，只有 "Default group policy"

**根因确认**：两个节点位于不同 AD OU，应用了不同 GPO。异常节点的 GPO 将集群服务启动类型设为 Disabled，导致每次 GPO 刷新时（~90 分钟）集群服务被重新设置为 Disabled 并停止。

### 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| 未配置 Quorum Witness | 初始分析被网络连接问题误导，延迟了根因定位 | 客户配置 Cloud Witness 后排除了此因素 |
| 单节点测试仍有问题时方向不明确 | 排除集群层面后需要 OS 层面增强日志 | 设计了 SCM + Sysmon 组合日志方案 |
| 客户执行 TSS/Sysmon 步骤困难 | 多次邮件往返，延迟了日志收集 | 安排 Teams 会议实时协助 |
| 客户生产环境紧急事务 | 中间有一天客户无法配合排查 | 等待客户恢复后继续 |
| VSS 干扰项 | 误以为 VSS 是导致停止的原因 | 禁用 VSS 后问题依旧，快速排除 |

### 5. 根因与解决方案 (Root Cause & Resolution)

#### Root Cause

**Group Policy Object (GPO) 配置不一致**。两个集群节点位于不同的 Active Directory OU 中，导致应用了不同的 GPO：

- 正常节点：应用了 **"Enable cluster service"** GPO，该策略将集群服务启动类型设置为 **Automatic**。
- 异常节点：仅应用了 **"Default group policy"**，该策略将集群服务启动类型设置为 **Disabled**。

当 GPO 定期刷新时（默认每 90 分钟），异常节点上的集群服务会被重新设置为 Disabled 并停止，造成间歇性故障的现象。

#### Resolution

1. 联系 AD 管理员，将异常节点移动到与正常节点相同的 OU，或者
2. 在 Default Group Policy 中将集群服务启动类型设置为 Automatic，或者
3. 将 "Enable cluster service" GPO 链接到异常节点所在的 OU。

客户选择修复 GPO 配置后，SQL 集群恢复正常运行。

### 6. 经验教训 (Lessons Learned)

- **技术知识**：
  - GPO 可以控制 Windows 服务的启动类型。当集群服务被 GPO 设置为 Disabled 时，表现为间歇性停止（与 GPO 刷新周期相关）。
  - SCM ETL 日志可以精确追踪哪个进程停止了哪个服务，配合 Sysmon 可以构建完整的进程行为链。
  - `gpresult /SCOPE computer /H GPRESULT.HTML` 是比较节点间 GPO 差异的有效工具。

- **排查方法**：
  - 当集群服务在单节点也停止时，应该立即将排查方向从集群层面转向 OS 层面。
  - 对于"服务被意外停止"的场景，**SCM ETL + Sysmon** 是黄金组合：SCM 告诉你"谁停的"，Sysmon 告诉你"当时发生了什么"。
  - 当怀疑是外部因素修改了服务配置时，**GPO 应该是首先检查的对象之一**，尤其是在多节点环境中节点行为不一致时。

- **流程改进**：
  - 对于新搭建的集群环境，应在初始检查清单中加入 GPO 一致性验证。
  - 远程协助（Teams 会议）在客户执行复杂步骤困难时应尽早安排，避免多次邮件往返。

- **预防措施**：
  - 部署集群前，确保所有节点在同一 OU 或应用了相同的 GPO。
  - 使用 `gpresult /R` 验证所有集群节点的 GPO 配置一致性。
  - 配置 Quorum Witness（Cloud Witness / File Share Witness）作为最佳实践。

### 7. 参考文档 (References)

- [Understand cluster and pool quorum](https://learn.microsoft.com/en-us/azure-stack/hci/concepts/quorum) — 集群仲裁概念说明
- [Configure and manage quorum](https://learn.microsoft.com/en-us/windows-server/failover-clustering/manage-cluster-quorum) — 配置和管理集群仲裁
- [Deploy a Cloud Witness for a Failover Cluster](https://learn.microsoft.com/en-us/windows-server/failover-clustering/deploy-cloud-witness) — 部署 Cloud Witness
- [Sysmon - Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) — System Monitor 工具下载与说明

---

## English Version

### 1. Symptoms

- On a newly built two-node Windows Server 2022 Failover Cluster (intended for SQL Server migration), the **Cluster Service (clussvc) intermittently stopped and was set to Disabled** state.
- The issue occurred on both nodes but **not simultaneously**.
- After starting the cluster service, it would run for some time then automatically stop and switch to disabled.
- The environment was **brand new** and had never worked properly.
- Urgency: The customer planned to migrate SQL 2012 databases to this new cluster over the weekend.

### 2. Background / Environment

- **OS**: Windows Server 2022
- **Cluster Configuration**: Two-node Failover Cluster (nodes: sqlnode01, sqlnode02)
- **Purpose**: SQL Server cluster for SQL 2012 database migration
- **Network Configuration**:
  - Node 1 (sqlnode01) IP: 10.10.20.18
  - Node 2 (sqlnode02) IP: 10.10.148.18
  - Cluster communication port: 3343
  - Firewall ports opened: 135, 3343, 445, 5022, 49152-65535, 137
- **Initial State**: No Quorum Witness configured (critical for a two-node cluster)

### 3. Investigation & Troubleshooting

#### Phase 1: Initial Log Analysis (7/26)

**Action Plan:** Review cluster validation report, cluster logs, and event logs. Request additional data:
```powershell
# Collect firewall and network configuration
Get-NetFirewallProfile > C:\temp\fwprofile.txt
Get-NetFirewallRule > C:\temp\FWRule.txt
ipconfig /all > C:\temp\ip.txt
```

**Log Analysis — Cluster Validation Report (No Quorum Witness):**
```
Start: 24/07/2024 5:44:23 PM.
Validating cluster quorum settings.
Witness Type: No Witness Configured
Witness Resource: No Witness Configured
The cluster is not configured with a quorum witness. As a best practice, configure a 
quorum witness to help achieve the highest availability of the cluster.
```

**Log Analysis — Cluster Log (clussvc repeatedly stopped and disabled):**
```
360329 [Operational] 000036ec.000019ac::2024/07/07-12:16:59.868 INFO  The cluster service has been stopped and set as disabled as part of cluster node cleanup.
360338 [Operational] 00001b50.000035c8::2024/07/09-04:32:31.105 INFO  The cluster service has been stopped and set as disabled as part of cluster node cleanup.
360347 [Operational] 00002418.00000ea4::2024/07/10-03:36:28.779 INFO  The cluster service has been stopped and set as disabled as part of cluster node cleanup.
360356 [Operational] 0000141c.0000427c::2024/07/10-03:43:59.926 INFO  The cluster service has been stopped and set as disabled as part of cluster node cleanup.
360365 [Operational] 00003544.000042f4::2024/07/10-03:54:05.214 INFO  The cluster service has been stopped and set as disabled as part of cluster node cleanup.
360690 [Operational] 000040d4.00003778::2024/07/16-02:38:39.674 INFO  The cluster service has been stopped and set as disabled as part of cluster node cleanup.
```

**Log Analysis — Cluster Log (Port 3343 connectivity errors + heartbeat misses):**

sqlnode01 → sqlnode02 connection failures:
```
495454 00001320.00002da0::2024/07/24-07:35:14.032 INFO  [NETWORK] Error when processing connection to node sqlnode02, error (10060)
...
495590 00001320.00002da0::2024/07/24-07:35:40.790 INFO  [NETWORK] Processing initial connection to node sqlnode02
495969 00001320.00002cf8::2024/07/24-07:35:41.115 INFO  [NETWORK] Connected and authenticated to node sqlnode02
496115 00001320.000022ec::2024/07/24-07:35:42.041 INFO  [NETWORK] Full mesh connectivity with nodes sqlnode01, sqlnode02
497896 00001320.00001724::2024/07/24-07:47:30.789 WARN  [NETWORK] Missed 40% of the heart beats with node 'sqlnode02' (2) on route 10.10.20.18:~3343~->10.10.148.18:~3343~
498334 00001320.00002cf8::2024/07/24-07:56:35.251 WARN  [NETWORK] Lost connection to node sqlnode02
498701 00001320.00000964::2024/07/24-08:00:29.857 INFO  [NETWORK] Local node is shutting down, disconnecting from other nodes
498708 0000264c.00000738::2024/07/24-08:00:29.973 WARN  [RHS] Cluster service has terminated.
```

**Conclusion:** Missing Quorum Witness + unstable port 3343 connectivity between nodes caused cluster service to repeatedly stop.

**Action Plan (to customer):**
1. Configure Quorum Witness (Cloud Witness / File Share Witness / Disk Witness)
2. Ensure both nodes have consistent network configurations and firewall Public Profile allows inbound/outbound

---

#### Phase 2: Cloud Witness Configured but Issue Persists (8/8 - 8/9)

**Log Analysis — sqlnode01 cluster log (connection failures → service shutdown):**
```
446134 00003948.00003338::2024/08/08-06:50:26.487 INFO  [NETWORK] Error when processing connection to node sqlnode02, error (10060)
...
446552 00003948.00003338::2024/08/08-06:56:08.842 INFO  [NETWORK] Processing initial connection to node sqlnode02
446940 00003948.00003f6c::2024/08/08-06:56:09.165 INFO  [NETWORK] Connected and authenticated to node sqlnode02
449570 00003948.000022bc::2024/08/08-07:31:58.449 INFO  [NETWORK] Local node is shutting down, disconnecting from other nodes
```

**Log Analysis — sqlnode02 cluster log (connection failure → Cloud Witness offline → shutdown):**
```
124064 000033dc.00003078::2024/08/08-07:55:41.151 INFO  [NODE] Node 2: stage: 'Attempt Initial Connection' status (10060) reason: 'Failed to connect to remote endpoint 10.10.20.18:~3343~'
124096 000033dc.000023d4::2024/08/08-07:56:57.095 INFO  [CS] Service Stopping...
124100 000033dc.00001e68::2024/08/08-07:56:57.095 INFO  [CORE] Graceful shutdown reported by node sqlnode02, reason ServiceStopReason::ServiceShutdown
124193 000033dc.00002e2c::2024/08/08-07:56:57.301 INFO  [RCM] HandleMonitorReply: OFFLINERESOURCE for 'Cloud Witness'
124203 000033dc.00003288::2024/08/08-07:56:57.301 WARN  [QUORUM] Node 2: One off quorum (2)
```

**Conclusion:** Cloud Witness configured but cluster service still stops. Port 3343 connectivity was a concern but deeper investigation needed.

---

#### Phase 3: Narrowing Down — Single Node Test (8/12)

**Action Plan:**
```powershell
# Step 1: Increase cluster log level to 5
Set-ClusterLog –Level 5

# Step 2: Disable and stop cluster service on the other node
Stop-Service clussvc
Set-Service clussvc -StartupType Disabled
```

**Diagnostic Logic:**
- If single-node cluster service **stops running** → issue is at the OS level (another process stopping it)
- If single-node cluster service **stays running** → issue relates to inter-node communication

**Result:** Even with only one node, cluster service still intermittently stopped.

**Conclusion:** Issue was **not at the cluster level**. Something at the OS level was stopping the cluster service.

---

#### Phase 4: Checking Scheduled Tasks (8/12 - 8/13)

**Log Analysis — Cluster log showing restart task scheduling:**
```
518370 00002f48.000014e0::2024/08/12-17:26:38.404 DBG   [RCM] scheduling next RunRestartTasks task...
518380 00002f48.0000364c::2024/08/12-17:26:53.682 DBG   [NODE] Node 1: eating message sent to the dead node 2
518381 00002f48.0000364c::2024/08/12-17:26:53.682 INFO  [CS] Service Stopping...
518383 00002f48.00002c9c::2024/08/12-17:26:53.682 INFO  [CORE] Graceful shutdown reported by node sqlnode01, reason ServiceStopReason::ServiceShutdown
```

**Action Plan:**
```cmd
schtasks /query > C:\task.txt
```

**Result:** Scheduled tasks output not clear enough to pinpoint the cause. Need deeper OS-level logging.

---

#### Phase 5: Deploying SCM + Sysmon Enhanced Logging (8/13)

**Action Plan (Complete Steps):**

**Step 1: Install Sysmon to monitor all process creation/termination**
```cmd
:: 1. Download Sysmon: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
:: 2. Unzip to C:\temp
:: 3. Open CMD as admin, navigate to Sysmon path, install:
sysmon -I -accepteula

:: 4. Verify installation:
fltmc

:: 5. Enable process monitoring mode:
sysmon -m

:: After installation, Sysmon events appear at:
:: Event Viewer → Application and Services Logs → Microsoft → Windows → Sysmon → Operational
```

**Step 2: Enable SCM (Service Control Manager) ETL logging**
```cmd
:: Open another CMD as admin and run:
logman create trace "base_screg" -ow -o c:\base_screg.etl -p "Service Control Manager" 0xffffffffffffffff 0xff -nb 16 16 -bs 1024 -mode Circular -f bincirc -max 4096 -ets
logman update trace "base_screg" -p "Microsoft-Windows-Services" 0xffffffffffffffff 0xff -ets
logman update trace "base_screg" -p {EBCCA1C2-AB46-4A1D-8C2A-906C2FF25F39} 0xffffffffffffffff 0xff -ets
```

**Step 3: Configure auto-stop logging (Task Scheduler + scripts)**

Create `C:\temp\StopLogging.bat`:
```bat
logman stop "base_screg" -ets
```

Create `C:\temp\SCMLoggingCapture.ps1`:
```powershell
$serviceName = "clussvc"
$serviceStatus = Get-Service -Name $serviceName | Select-Object -ExpandProperty Status
if ($serviceStatus -ne "Running") {
    Start-Process -FilePath "C:\Temp\StopLogging.bat"
}
```

Configure Task Scheduler:
1. Open Task Scheduler → Create Task
2. Triggers tab → New → On a schedule → Repeat every 5 minutes
3. Actions tab → New → Start a program → PowerShell.exe → Arguments: `C:\temp\SCMLoggingCapture.ps1`

**Logs to collect:**
- Sysmon event log
- All event logs: `%SystemRoot%\System32\Winevt\Logs`
- SCM ETL log: `C:\base_screg.etl`
- Cluster log
- Process list: `tasklist /svc > C:\tasklist.txt`

---

#### Phase 6: Analyzing SCM + Sysmon Logs — Key Discovery (8/14 - 8/15) 🔍

This was the most critical step. Cross-correlating three layers of logs pinpointed the exact process stopping the cluster service.

**Log Analysis — Layer 1: SCM ETL Log (Who stopped clussvc)**
```
427637 [2] 03D4.2B8C::08/14/24-14:25:40.4517713 [SCM] CWin32ServiceRecord::SetStatus: Service Record updated with new status for service ClusSvc
427639 [6] 03D4.46C0::08/14/24-14:25:40.4519111 [SCM] ScSendControl: Service ClusSvc, control 1, user S-1-5-18, caller PID 0x000003d4
427681 [6] 03D4.46C0::08/14/24-14:25:40.4841063 [SCM] ScSendControl: Sending control 0x00000051 to service/module VSS, caller PID 0x00000474
427692 [4] 03D4.46C0::08/14/24-14:25:40.4843767 [SCM] ScSendControl: Successfully sent start control to service VSS, user S-1-5-18
427729 [0] 03D4.46C0::08/14/24-14:25:40.6130898 [SCM] RSetServiceStatus: service ClusSvc, state 0x00000003(SERVICE_STOP_PENDING)
427732 [0] 03D4.46C0::08/14/24-14:25:40.7958679 [SCM] RSetServiceStatus: service ClusSvc, state 0x00000001(SERVICE_STOPPED)
```

**Key Analysis:**
- `caller PID 0x000003d4` → **0x3D4 hex = 980 decimal**
- Who is PID 980? → Need Sysmon to confirm.

**Log Analysis — Layer 2: Sysmon Log (Confirm PID 980 = services.exe + clussvc termination)**

clussvc.exe process termination:
```
Process terminated:
UtcTime: 2024-08-14 06:25:40.787
ProcessId: 15144
Image: C:\Windows\Cluster\clussvc.exe
User: NT AUTHORITY\SYSTEM
```

**Log Analysis — Layer 3: Sysmon Log (PID 980 = services.exe, simultaneously created VSSVC.exe)**
```
Process Create:
UtcTime: 2024-08-14 06:25:40.446
ProcessId: 17080
Image: C:\Windows\System32\VSSVC.exe
Description: Microsoft® Volume Shadow Copy Service
CommandLine: C:\Windows\system32\vssvc.exe
User: NT AUTHORITY\SYSTEM
ParentProcessId: 980
ParentImage: C:\Windows\System32\services.exe
ParentCommandLine: C:\Windows\system32\services.exe
```

**Three-Layer Log Correlation Summary:**

| Timestamp | Event | Source |
|-----------|-------|--------|
| 14:25:40.451 | SCM updates ClusSvc status, caller PID 0x3D4 (980) | SCM ETL |
| 14:25:40.446 | services.exe (PID 980) creates VSSVC.exe (PID 17080) | Sysmon |
| 14:25:40.484 | SCM sends start control to VSS service | SCM ETL |
| 14:25:40.613 | ClusSvc state → SERVICE_STOP_PENDING | SCM ETL |
| 14:25:40.787 | clussvc.exe (PID 15144) process terminated | Sysmon |
| 14:25:40.795 | ClusSvc state → SERVICE_STOPPED | SCM ETL |

**Conclusion:** **SCM (services.exe)** itself was stopping the cluster service. VSS was created at the same time — possibly related or coincidental.

---

#### Phase 7: Ruling Out VSS (8/15)

**Action Plan:** Temporarily disable and stop VSS service, then observe if cluster service still stops.

**Result:** After disabling VSS, the issue persisted.

**Conclusion:** VSS was not the root cause — coincidental timing. Need to investigate why SCM was setting clussvc to disabled.

---

#### Phase 8: TSS Tool Collection Issues (8/15 - 8/16)

**Action Plan:**
```powershell
# 1. Download TSS: https://aka.ms/getTSS
# 2. Unzip to C:\temp\TSS
# 3. Open admin PowerShell:
Set-ExecutionPolicy -ExecutionPolicy Bypass -force -Scope Process
.\TSS.ps1 -SHA_MsCluster -Procmon -WPR General -WaitEvent Process:clussvc
```

**Customer Issues:**
- Parameter error: `-WPR General-WaitEvent` treated as single argument (missing space)
- Script not found: didn't navigate to TSS directory first

**Conclusion:** Live Teams meeting needed for assistance.

---

#### Phase 9: Live Teams Session — Root Cause Found (8/16) 🎯

**Action Plan:** Compare GPO configuration between nodes using GPO diagnostic tools.
```cmd
:: Generate HTML-format GPO report
gpresult /SCOPE computer /H GPRESULT.HTML

:: List all applied GPOs
GPRESULT /R
```

**GPO Comparison Results — Working vs Non-Working Node:**

| Comparison | Working Node (sqlnode01) | Non-Working Node (sqlnode02) |
|------------|------------------------|------------------------------|
| Winning GPO | **"Enable cluster service"** | **"Default group policy"** |
| Cluster Service Startup Type | **Automatic** | **Disabled** |
| OU Location | OU with cluster policy | Default OU without cluster policy |

**Root Cause Confirmed:** The two nodes were in different AD OUs with different GPOs applied. The non-working node's GPO set the cluster service startup type to Disabled, causing it to be periodically reset to Disabled on every GPO refresh (~90 minutes).

### 4. Blockers & How They Were Resolved

| Blocker | Impact | Resolution |
|---------|--------|------------|
| No Quorum Witness configured | Initial analysis was misdirected by network connectivity issues | Customer configured Cloud Witness, eliminating this factor |
| Single-node test still failing — unclear direction | After ruling out cluster-level, needed OS-level enhanced logging | Designed SCM + Sysmon combined logging approach |
| Customer difficulty executing TSS/Sysmon steps | Multiple email round-trips delayed log collection | Scheduled Teams meeting for live assistance |
| Customer busy with production emergencies | One-day delay in troubleshooting | Waited for customer availability to resume |
| VSS red herring | Temporarily misidentified VSS as the cause | Quickly ruled out after disabling VSS with no improvement |

### 5. Root Cause & Resolution

#### Root Cause

**Group Policy Object (GPO) misconfiguration**. The two cluster nodes were in different Active Directory OUs, resulting in different GPOs being applied:

- **Working node**: Had **"Enable cluster service"** GPO applied, which set the cluster service startup type to **Automatic**.
- **Non-working node**: Only had **"Default group policy"** applied, which set the cluster service startup type to **Disabled**.

When GPO refreshed periodically (default every ~90 minutes), the cluster service on the non-working node would be set back to Disabled and stopped, causing the intermittent failure pattern.

#### Resolution

1. Contact the AD team to move the non-working node to the same OU as the working node, OR
2. Set cluster service startup type to Automatic in the Default Group Policy, OR
3. Link the "Enable cluster service" GPO to the OU containing the non-working node.

The customer fixed the GPO configuration, and the SQL cluster began functioning normally.

### 6. Lessons Learned

- **Technical Knowledge**:
  - GPO can control Windows service startup types. When cluster service is set to Disabled by GPO, it manifests as intermittent stopping (correlating with GPO refresh cycles).
  - SCM ETL logs can precisely trace which process stopped which service. Combined with Sysmon, you can build a complete process behavior chain.
  - `gpresult /SCOPE computer /H GPRESULT.HTML` is an effective tool for comparing GPO differences between nodes.

- **Troubleshooting Methodology**:
  - When cluster service stops even on a single node, immediately shift focus from cluster-level to OS-level investigation.
  - For "service unexpectedly stopped" scenarios, **SCM ETL + Sysmon** is the golden combination: SCM tells you "who stopped it," Sysmon tells you "what else was happening."
  - When suspecting external factors modifying service configuration, **GPO should be one of the first things to check**, especially when nodes behave inconsistently in a multi-node environment.

- **Process Improvements**:
  - For newly built cluster environments, add GPO consistency verification to the initial checklist.
  - Arrange remote assistance (Teams meeting) early when customers struggle with complex diagnostic steps, rather than multiple email round-trips.

- **Preventive Measures**:
  - Before deploying a cluster, ensure all nodes are in the same OU or have the same GPOs applied.
  - Use `gpresult /R` to verify GPO configuration consistency across all cluster nodes.
  - Configure Quorum Witness (Cloud Witness / File Share Witness) as a best practice.

### 7. References

- [Understand cluster and pool quorum](https://learn.microsoft.com/en-us/azure-stack/hci/concepts/quorum) — Cluster quorum concepts
- [Configure and manage quorum](https://learn.microsoft.com/en-us/windows-server/failover-clustering/manage-cluster-quorum) — Quorum witness configuration guide
- [Deploy a Cloud Witness for a Failover Cluster](https://learn.microsoft.com/en-us/windows-server/failover-clustering/deploy-cloud-witness) — Cloud Witness deployment
- [Sysmon - Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) — System Monitor tool
