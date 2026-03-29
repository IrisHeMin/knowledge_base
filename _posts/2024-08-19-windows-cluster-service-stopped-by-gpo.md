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

1. **初始日志分析（7/26）**
   - **做了什么**：检查客户上传的集群验证报告和集群日志。
   - **发现了什么**：
     - 未配置 Quorum Witness。
     - 集群日志显示 "The cluster service has been stopped and set as disabled as part of cluster node cleanup" 反复出现。
     - 节点间存在 port 3343 连接错误（error 10060）。
     - 心跳丢失达 40%。
   - **得出了什么结论**：缺少 Quorum Witness 加上网络连接问题，可能导致集群服务停止。建议先配置 Cloud Witness 并检查防火墙。

2. **配置 Cloud Witness 后仍有问题（8/8 - 8/9）**
   - **做了什么**：客户配置了 Cloud Quorum Witness，工程师重新分析日志。
   - **发现了什么**：集群日志仍然显示节点间 3343 端口连接失败（error 10060），最终导致 cluster service graceful shutdown。
   - **得出了什么结论**：Cloud Witness 已配置但问题依旧，3343 端口连接是关键问题，但需要进一步排查根因。

3. **缩小范围 — 单节点测试（8/12）**
   - **做了什么**：建议客户禁用一个节点的集群服务，只保留单节点运行，并将集群日志级别提升到 5。
   - **发现了什么**：即使只有一个节点，集群服务仍然间歇性停止。
   - **得出了什么结论**：问题**不在集群层面**（非节点间通信问题），而是 OS 层面有其他进程/应用在停止集群服务。

4. **检查计划任务（8/12 - 8/13）**
   - **做了什么**：从集群日志发现 restart task 调度记录，要求客户导出所有计划任务（`schtasks /query`）。
   - **发现了什么**：计划任务输出不够清晰，无法明确定位。
   - **得出了什么结论**：需要更详细的 OS 层面日志来追踪是谁停止了集群服务。

5. **部署 SCM + Sysmon 增强日志（8/13）**
   - **做了什么**：制定详细的日志收集方案：
     - 安装 Sysmon 监控所有进程创建/终止
     - 启用 SCM (Service Control Manager) ETL 日志
     - 配置 Task Scheduler 每 5 分钟检查 clussvc 状态，自动停止日志收集
   - **发现了什么**：客户在执行步骤时遇到困难，需要 Teams 会议协助。
   - **得出了什么结论**：需要远程协助客户完成日志收集配置。

6. **分析 SCM + Sysmon 日志 — 关键发现（8/14 - 8/15）**
   - **做了什么**：分析客户上传的 SCM ETL 日志和 Sysmon 事件日志。
   - **发现了什么**：
     - SCM ETL 日志显示 **services.exe (PID 980)** 停止了 clussvc 服务。
     - 同时 services.exe 创建了 VSSVC.exe 进程。
     - 从 Sysmon 日志确认 clussvc.exe 在 `2024-08-14 06:25:40.787` 被终止。
     - SCM 日志关键条目：
       ```
       ScSendControl: Service ClusSvc, control 1, user S-1-5-18, caller PID 0x000003d4
       ```
       0x3D4 = 980（即 services.exe）
   - **得出了什么结论**：是 SCM (services.exe) 自身在停止集群服务，可能与 VSS 服务有关联。

7. **排除 VSS 因素（8/15）**
   - **做了什么**：建议客户临时禁用并停止 VSS 服务。
   - **发现了什么**：禁用 VSS 后，问题依然存在。
   - **得出了什么结论**：VSS 不是根因，只是在同一时间点被 SCM 启动的巧合。需要继续追查 SCM 为什么设置 clussvc 为 disabled。

8. **TSS 工具数据收集遇阻（8/15 - 8/16）**
   - **做了什么**：要求客户使用 TSS 工具收集更详细的日志。
   - **发现了什么**：客户执行 TSS 命令时遇到参数错误和脚本路径问题。
   - **得出了什么结论**：需要通过 Teams 会议实时协助。

9. **Teams 会议实时排查 — 根因发现（8/16）**
   - **做了什么**：通过 Teams 会议远程协助，使用 `gpresult /SCOPE computer /H GPRESULT.HTML` 和 `GPRESULT /R` 比较两个节点的 GPO 配置。
   - **发现了什么**：
     - **正常工作的节点**：应用了 **"Enable cluster service"** GPO → 集群服务启动类型 = **Automatic**
     - **不正常的节点**：应用了 **"Default group policy"** → 集群服务启动类型 = **Disabled**
     - 两个节点所在的 OU 不同，导致应用了不同的 GPO。
   - **得出了什么结论**：**根因确认** — GPO 配置不一致导致集群服务被周期性设置为 Disabled。

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

1. **Initial Log Analysis (7/26)**
   - **Action**: Reviewed cluster validation report and cluster logs uploaded by customer.
   - **Finding**:
     - No Quorum Witness configured.
     - Cluster log showed repeated "The cluster service has been stopped and set as disabled as part of cluster node cleanup."
     - Port 3343 connectivity errors (error 10060) between nodes.
     - 40% heartbeat misses between nodes.
   - **Conclusion**: Missing Quorum Witness combined with network connectivity issues likely caused cluster service to stop. Recommended configuring Cloud Witness and checking firewall.

2. **Cloud Witness Configured but Issue Persists (8/8 - 8/9)**
   - **Action**: Customer configured Cloud Quorum Witness; engineer re-analyzed logs.
   - **Finding**: Cluster logs still showed port 3343 connection failures (error 10060) between nodes, resulting in cluster service graceful shutdown.
   - **Conclusion**: Cloud Witness configured but issue persisted. Port 3343 connectivity was a key concern but deeper root cause needed investigation.

3. **Narrowing Down — Single Node Test (8/12)**
   - **Action**: Suggested disabling cluster service on one node (leaving only single node running) and increasing cluster log level to 5.
   - **Finding**: Even with only one node, cluster service still intermittently stopped.
   - **Conclusion**: Issue was **not at the cluster level** (not inter-node communication). Something at the OS level was stopping the cluster service.

4. **Checking Scheduled Tasks (8/12 - 8/13)**
   - **Action**: Found restart task scheduling entries in cluster log; asked customer to export all scheduled tasks (`schtasks /query`).
   - **Finding**: Scheduled tasks output was not clear enough to pinpoint the cause.
   - **Conclusion**: Needed more detailed OS-level logging to trace what was stopping the cluster service.

5. **Deploying SCM + Sysmon Enhanced Logging (8/13)**
   - **Action**: Designed a detailed log collection plan:
     - Install Sysmon to monitor all process creation/termination
     - Enable SCM (Service Control Manager) ETL logging
     - Configure Task Scheduler to check clussvc status every 5 minutes and auto-stop logging
   - **Finding**: Customer had difficulty executing the steps and needed a Teams meeting for assistance.
   - **Conclusion**: Remote assistance needed to complete log collection setup.

6. **Analyzing SCM + Sysmon Logs — Key Discovery (8/14 - 8/15)**
   - **Action**: Analyzed SCM ETL logs and Sysmon event logs uploaded by customer.
   - **Finding**:
     - SCM ETL log showed **services.exe (PID 980)** stopped the clussvc service.
     - Simultaneously, services.exe created the VSSVC.exe process.
     - Sysmon logs confirmed clussvc.exe terminated at `2024-08-14 06:25:40.787`.
     - Key SCM log entry:
       ```
       ScSendControl: Service ClusSvc, control 1, user S-1-5-18, caller PID 0x000003d4
       ```
       0x3D4 = 980 (services.exe)
   - **Conclusion**: SCM (services.exe) itself was stopping the cluster service. VSS appeared to be a correlated but potentially unrelated event.

7. **Ruling Out VSS (8/15)**
   - **Action**: Asked customer to temporarily disable and stop VSS service.
   - **Finding**: After disabling VSS, the issue persisted.
   - **Conclusion**: VSS was not the root cause — it was coincidentally started by SCM at the same time. Needed to continue investigating why SCM was setting clussvc to disabled.

8. **TSS Tool Collection Issues (8/15 - 8/16)**
   - **Action**: Asked customer to use TSS tool for more detailed data collection.
   - **Finding**: Customer encountered parameter errors and script path issues when running TSS commands.
   - **Conclusion**: Required live Teams meeting to assist.

9. **Live Teams Session — Root Cause Found (8/16)**
   - **Action**: Conducted live troubleshooting via Teams meeting. Compared GPO configuration between nodes using `gpresult /SCOPE computer /H GPRESULT.HTML` and `GPRESULT /R`.
   - **Finding**:
     - **Working node**: Had **"Enable cluster service"** GPO applied → cluster service startup type = **Automatic**
     - **Non-working node**: Had **"Default group policy"** applied → cluster service startup type = **Disabled**
     - The two nodes were in different OUs, resulting in different GPOs being applied.
   - **Conclusion**: **Root cause confirmed** — GPO inconsistency was causing the cluster service to be periodically set to Disabled.

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
