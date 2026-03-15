---
layout: post
title: "Deep Dive: Windows Server Failover Clustering"
date: 2026-03-10
categories: [Knowledge, Clustering]
tags: [failover-clustering, quorum, heartbeat, csv, cluster-aware-updating, health-service, high-availability, windows-server, azure-stack-hci]
type: "deep-dive"
---

# Deep Dive: Windows Server Failover Clustering

**Topic:** Windows Server Failover Clustering (WSFC)
**Category:** High Availability / Clustering
**Level:** 中级 → 高级
**Last Updated:** 2026-03-10

---

## 1. 概述 (Overview)

Windows Server Failover Clustering (WSFC) 是 Windows Server 平台内置的高可用性基础设施。它将多台独立的服务器（称为 **节点/Node**）组织成一个逻辑集群，当某个节点出现硬件故障、软件崩溃或网络中断时，集群中的其他节点能**自动接管**该节点上运行的工作负载（称为 **Failover/故障转移**），从而实现关键业务应用的持续可用。

WSFC 解决的核心问题是 **单点故障 (Single Point of Failure)**。在传统单服务器架构中，一旦服务器宕机，上面的所有服务都会中断。WSFC 通过冗余节点 + 共享存储 + 自动故障检测与转移机制，将服务中断时间从"小时级"降低到"秒级"甚至"零中断"（如 Live Migration）。

在 Microsoft 技术生态中，WSFC 是以下关键产品的底层支撑：
- **Hyper-V 高可用性**：VM 可在节点间自动迁移
- **SQL Server Always On**：数据库故障转移集群实例 (FCI) 和可用性组 (AG)
- **Scale-Out File Server (SOFS)**：横向扩展文件服务
- **Azure Stack HCI**：超融合基础设施的集群管理
- **Storage Spaces Direct (S2D)**：软件定义存储

---

## 2. 核心概念 (Core Concepts)

### 2.1 节点 (Node)

- **定义**：集群中的每一台服务器（物理机或虚拟机）
- **要求**：所有节点应运行相同版本的 Windows Server，硬件配置建议一致
- **状态**：Up（在线）、Down（离线）、Paused（暂停，不接受新的故障转移）、Joining（加入中）
- **最大数量**：Windows Server 2016+ 支持最多 **64 个节点**

> **类比**：把集群想象成一个接力赛队伍，每个节点是一位选手。当一位选手摔倒时，下一位选手立刻接棒继续跑，比赛（服务）不会因此中断。

### 2.2 集群角色 (Clustered Role) / 资源组 (Resource Group)

- **定义**：在集群上运行的一组相关资源的逻辑集合，代表一个可故障转移的工作负载
- **示例**：一个 Hyper-V VM、一个 SQL Server 实例、一个文件服务器角色
- **资源 (Resource)**：组成角色的单个组件，如 IP 地址、网络名称、磁盘、服务等
- **资源依赖 (Resource Dependency)**：定义资源之间的启动顺序和依赖关系。例如，文件服务器角色依赖于 IP 地址 → 网络名称 → 磁盘
- **Owner Node**：当前运行该角色的节点
- **Preferred Owner**：管理员配置的首选运行节点列表

### 2.3 Quorum（仲裁/法定人数）

Quorum 是 Failover Clustering 最核心的概念之一。它是集群在"分裂"场景下做出决策的机制。

- **核心问题**：当网络分区（Network Partition / Split-Brain）发生时，集群的两半都认为自己是"正确的"。如果两边都继续运行并写入共享存储，数据就会损坏
- **解决方案**：**投票机制**。每个节点和见证各有一票（或通过 Dynamic Quorum 动态调整）。只有获得**多数投票**（> 50%）的那一半才能继续运行；另一半自动停止集群服务
- **公式**：对于 N 个投票成员，需要 ⌊N/2⌋ + 1 票才能维持 Quorum

#### Quorum 模式

| 模式 | 投票成员 | 适用场景 |
|------|---------|---------|
| **Node Majority** | 仅节点投票 | 奇数个节点（如 3、5 节点） |
| **Node & Disk Majority** | 节点 + 共享磁盘见证 | 偶数个节点 + 有共享存储 |
| **Node & File Share Majority** | 节点 + 文件共享见证 | 偶数个节点 + 无共享存储（如跨站点） |
| **Cloud Witness** | 节点 + Azure Blob 见证 | 推荐方式，适用于所有拓扑（Windows Server 2016+） |

#### Dynamic Quorum（动态仲裁）

Windows Server 2012 R2+ 引入，集群会**动态调整**每个节点的投票权重：
- 当节点正常离线时，集群自动收回其投票权，重新计算 Quorum
- 这意味着一个 5 节点集群可以在逐步丢失节点时仍保持 Quorum，而非严格要求始终有 3 个节点在线
- 通过 `(Get-Cluster).DynamicQuorum` 查看和配置

### 2.4 Heartbeat（心跳）

- **定义**：集群节点之间周期性发送的 UDP 数据包（默认端口 3343），用于检测节点是否存活
- **默认参数**：
  - **SameSubnetDelay**：同子网心跳间隔，默认 1000ms
  - **SameSubnetThreshold**：同子网失败阈值，默认 10 次（Windows Server 2016+ 默认 20 次）
  - **CrossSubnetDelay**：跨子网心跳间隔，默认 1000ms
  - **CrossSubnetThreshold**：跨子网失败阈值，默认 20 次（Windows Server 2016+ 默认 40 次）
- **判定逻辑**：如果连续 N 次心跳无响应（N = Threshold），该节点被判定为 Down

> **关键理解**：心跳失败不一定意味着节点真正宕机。网络短暂中断、CPU 繁忙导致的延迟都可能触发误判。这是排查中常见的坑。

### 2.5 Cluster Shared Volumes (CSV)

- **定义**：一种集群文件系统层（CSVFS），允许集群中所有节点**同时读写**同一个 LUN/磁盘
- **核心价值**：传统集群磁盘一次只能被一个节点"拥有"（独占）；CSV 打破了这个限制，支持并发访问
- **访问路径**：所有节点通过统一路径 `C:\ClusterStorage\Volume1` 访问
- **主要用途**：
  - Hyper-V VM 的 VHDX 文件存储
  - Storage Spaces Direct 的存储卷
  - Scale-Out File Server 的数据存储
- **I/O 模式**：
  - **Direct I/O**（正常模式）：节点直接访问底层 SAN 存储，性能最佳
  - **Redirected I/O**（重定向模式）：I/O 通过网络重定向到 Coordinator Node，发生在磁盘维护或文件系统不对齐时
  - **Block Redirected I/O**（块重定向模式）：S2D 场景中通过网络访问不在本地的数据块

### 2.6 Cluster Network（集群网络）

集群中的网络通常分为几种角色：

| 网络角色 | 用途 | 说明 |
|---------|------|------|
| **Cluster Only** | 仅集群内部通信 | 心跳、集群状态同步 |
| **Client Access** | 仅客户端访问 | 对外提供服务的网络 |
| **Cluster and Client** | 两者兼用 | 默认配置，同时承载心跳和客户端访问 |
| **Not Used** | 集群不使用 | 管理网络或 iSCSI 专用网络 |

---

## 3. 工作原理 (How It Works)

### 3.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Failover Cluster                          │
│                                                             │
│  ┌──────────┐    Heartbeat (UDP 3343)    ┌──────────┐       │
│  │  Node 1  │◄──────────────────────────►│  Node 2  │       │
│  │ (Active) │                            │(Standby) │       │
│  │          │    Heartbeat (UDP 3343)    │          │       │
│  │ ┌──────┐ │◄──────────────────────────►│ ┌──────┐ │       │
│  │ │ Role │ │                            │ │(idle)│ │       │
│  │ │ (VM) │ │     ┌──────────┐           │ └──────┘ │       │
│  │ └──────┘ │     │  Node 3  │           │          │       │
│  └─────┬────┘     │(Standby) │           └──────────┘       │
│        │          └──────────┘                              │
│        │                                                    │
│  ┌─────▼──────────────────────────────────────────┐         │
│  │          Shared Storage / CSV / S2D             │         │
│  │    ┌────────┐  ┌────────┐  ┌────────┐          │         │
│  │    │ Disk 1 │  │ Disk 2 │  │Witness │          │         │
│  │    │ (CSV)  │  │ (CSV)  │  │ (Quor) │          │         │
│  │    └────────┘  └────────┘  └────────┘          │         │
│  └────────────────────────────────────────────────┘         │
│                                                             │
│  ┌─────────────────────────────────────────────┐            │
│  │  Cluster Database (CLUSDB) - 复制到每个节点    │            │
│  └─────────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 集群形成流程 (Cluster Formation)

1. **Step 1: 验证 (Validation)**
   - 运行 `Test-Cluster` 验证所有节点的硬件、软件、网络、存储是否满足集群要求
   - 这是创建集群前**必须执行**的步骤

2. **Step 2: 创建集群 (Create)**
   - 运行 `New-Cluster` 在 Active Directory 中创建 Cluster Name Object (CNO)
   - 初始化集群数据库 (CLUSDB)，复制到所有节点
   - 分配集群 IP 地址和网络名称

3. **Step 3: 配置 Quorum**
   - 系统自动选择 Quorum 模式（Windows Server 2016+ 推荐使用 Cloud Witness）
   - 管理员可通过 `Set-ClusterQuorum` 手动调整

4. **Step 4: 添加存储和角色**
   - 将共享磁盘添加到集群，配置为 CSV
   - 创建集群角色（如 Hyper-V VM、File Server 等）

### 3.3 故障转移流程 (Failover Process)

这是集群最核心的工作流程。当 Node 1 发生故障时：

```
时间线:
t=0s     Node 2/3 检测到 Node 1 心跳丢失
t=1-10s  连续多次心跳超时，累积到阈值
t=10-20s 集群判定 Node 1 为 Down（Event 1135）
         ↓
         Quorum 检查：剩余节点是否仍持有多数投票？
         ├── 是 → 继续故障转移
         └── 否 → 集群服务停止（Event 1177: Quorum Lost）
         ↓
t=20-25s Resource Hosting Subsystem (RHS) 在目标节点启动资源
         ├── 按资源依赖顺序逐个启动：
         │   1. Physical Disk / CSV → Online
         │   2. IP Address → Online
         │   3. Network Name → Online
         │   4. Application/Service → Online
         └── 每个资源有独立的超时设置
         ↓
t=25-60s 角色在新节点上完全恢复运行
         客户端通过集群网络名称重新连接
```

#### 故障转移类型

| 类型 | 触发方式 | 影响 |
|------|---------|------|
| **Planned Failover** | 管理员手动移动角色 | 零中断（先在目标节点启动，再在源节点停止） |
| **Unplanned Failover** | 节点故障自动触发 | 有短暂中断（秒级到分钟级） |
| **Live Migration** | 管理员或 CAU 触发的 VM 迁移 | 对 Hyper-V VM 几乎零中断 |
| **Quick Migration** | 先保存 VM 状态再迁移 | 有明显中断 |

### 3.4 健康监控机制

集群通过多层机制持续监控健康状态：

**层级 1：节点级 — Heartbeat**
- 节点间通过 UDP 3343 端口交换心跳
- 每秒一次（默认），连续失败达阈值判定节点 Down

**层级 2：资源级 — IsAlive / LooksAlive**
- **LooksAlive**：轻量级检查（默认每 5 秒），验证资源进程是否存在
- **IsAlive**：深度检查（默认每 60 秒），验证资源是否真正可用（如数据库能否响应查询）
- 如果资源连续失败，触发 **资源重启 → 故障转移 → 无进一步操作** 的逐级升级策略

**层级 3：Health Service（Windows Server 2016+）**
- 主要用于 Storage Spaces Direct 场景
- 提供磁盘生命周期管理、性能历史、故障报告
- 自动执行磁盘退役和数据重建

---

## 4. 关键配置与参数 (Key Configurations)

### 4.1 心跳与故障检测参数

| 参数 | 默认值 (2016+) | 说明 | 调优场景 |
|------|---------------|------|---------|
| `SameSubnetDelay` | 1000ms | 同子网心跳发送间隔 | 网络延迟高时可增大 |
| `SameSubnetThreshold` | 20 | 同子网心跳失败阈值 | 频繁误判时可增大 |
| `CrossSubnetDelay` | 1000ms | 跨子网心跳发送间隔 | 跨站点集群需评估 |
| `CrossSubnetThreshold` | 40 | 跨子网心跳失败阈值 | 跨站点集群需评估 |
| `RouteHistoryLength` | 10 | 路由历史记录长度 | 多路径网络排查时 |

```powershell
# 查看当前心跳参数
(Get-Cluster).SameSubnetDelay
(Get-Cluster).SameSubnetThreshold
(Get-Cluster).CrossSubnetDelay
(Get-Cluster).CrossSubnetThreshold

# 修改参数示例（增加容忍度）
(Get-Cluster).SameSubnetThreshold = 30
(Get-Cluster).CrossSubnetThreshold = 60
```

### 4.2 资源故障恢复策略

| 参数 | 默认值 | 说明 | 调优场景 |
|------|--------|------|---------|
| `RestartAction` | Restart | 资源失败时的操作 | 某些资源不应自动重启 |
| `RestartThreshold` | 3 | 在 RestartPeriod 内最大重启次数 | 频繁重启的资源 |
| `RestartPeriod` | 900000ms (15min) | 重启计数的时间窗口 | 配合 RestartThreshold |
| `RetryPeriodOnFailure` | 3600000ms (1h) | 重试间隔 | 资源恢复缓慢时 |
| `FailoverThreshold` | 0xFFFFFFFF | 在 FailoverPeriod 内最大故障转移次数 | 防止"乒乓"效应 |
| `FailoverPeriod` | 21600000ms (6h) | 故障转移计数的时间窗口 | 配合 FailoverThreshold |

```powershell
# 查看资源恢复策略
Get-ClusterResource "SQL Server" | Format-List RestartAction, RestartThreshold, RestartPeriod

# 设置故障转移阈值（防止乒乓）
(Get-ClusterGroup "SQL Server").FailoverThreshold = 3
(Get-ClusterGroup "SQL Server").FailoverPeriod = 21600 # 6 小时内最多 3 次
```

### 4.3 Quorum 配置

```powershell
# 查看当前 Quorum 配置
Get-ClusterQuorum

# 配置 Cloud Witness
Set-ClusterQuorum -CloudWitness -AccountName <StorageAccountName> -AccessKey <Key>

# 配置 File Share Witness
Set-ClusterQuorum -FileShareWitness "\\fileserver01\witness-share"

# 配置 Disk Witness
Set-ClusterQuorum -DiskWitness "Cluster Disk 1"
```

### 4.4 集群网络配置

```powershell
# 查看集群网络
Get-ClusterNetwork | Format-Table Name, State, Role

# Role 值：0=不使用, 1=仅集群, 3=集群和客户端访问

# 修改网络角色
(Get-ClusterNetwork "Cluster Network 1").Role = 1  # 仅集群通信

# 查看集群网络接口
Get-ClusterNetworkInterface | Format-Table Node, Network, State
```

---

## 5. 常见问题与排查 (Common Issues & Troubleshooting)

### 问题 A：节点意外被踢出集群 (Event 1135)

**典型事件**：
```
Event 1135: Cluster node 'NODE01' was removed from the active failover cluster membership.
```

**可能原因**：
- 网络瞬断或丢包导致心跳超时
- 节点 CPU 100% 导致心跳响应延迟
- 防火墙阻断了 UDP 3343 端口
- 网络驱动/固件问题
- VMQ (Virtual Machine Queue) 或 RSS 配置问题

**排查思路**：
1. 查看 Cluster Log：`Get-ClusterLog -Destination C:\temp -TimeSpan 60`
2. 在 Cluster Log 中搜索 `lost communication` 和 `missed heartbeat`
3. 检查节点在故障时的 CPU、内存使用率
4. 检查网络适配器的 `Get-NetAdapterStatistics` 有无大量错误/丢弃
5. 确认 Windows Firewall 的 "Failover Clusters" 规则已启用

**关键命令**：
```powershell
# 生成集群日志
Get-ClusterLog -Destination C:\temp -TimeSpan 60

# 检查节点状态
Get-ClusterNode | Format-Table Name, State, DynamicWeight

# 检查网络适配器状态
Get-NetAdapterStatistics | Format-Table Name, ReceivedErrors, OutboundErrors

# 检查防火墙规则
Get-NetFirewallRule -Group "@FirewallAPI.dll,-36751" | Format-Table Name, Enabled
```

### 问题 B：Quorum 丢失，集群停止 (Event 1177)

**典型事件**：
```
Event 1177: The Cluster service is shutting down because quorum was lost.
```

**可能原因**：
- 多数节点同时宕机或网络隔离
- Witness 不可用 + 节点丢失
- Dynamic Quorum 被禁用时，节点逐一离线导致投票不足
- Witness 文件共享权限丢失或 Cloud Witness 的 Azure 存储连接中断

**排查思路**：
1. 确认有多少节点在线、各自的投票权重
2. 检查 Witness 状态
3. 如果需要强制启动集群（紧急恢复），使用 `Start-ClusterNode -ForceQuorum`

**关键命令**：
```powershell
# 查看 Quorum 信息
Get-ClusterQuorum

# 查看每个节点的投票权重
Get-ClusterNode | Format-Table Name, State, DynamicWeight, NodeWeight

# 强制启动集群（紧急情况）
Start-ClusterNode -Name NODE01 -ForceQuorum

# 修复后恢复正常 Quorum
Start-ClusterNode -Name NODE02  # 其他节点正常加入即可
```

### 问题 C：资源无法上线 (Failed to Online)

**典型事件**：
```
Event 1069: Cluster resource 'XXX' of type 'YYY' in clustered role 'ZZZ' failed.
```

**可能原因**：
- 存储连接丢失（SAN 路径断开）
- 磁盘被其他进程锁定
- 服务启动依赖项缺失（如 SQL Server 依赖的数据库文件不可访问）
- IP 地址冲突
- Active Directory 中 CNO (Cluster Name Object) 权限不足

**排查思路**：
1. 检查 WER 报告：`%ProgramData%\Microsoft\Windows\WER\ReportQueue`
2. 查看资源类型的 DumpLogQuery 收集的诊断日志
3. 手动尝试 Online 资源并观察错误：`Start-ClusterResource "Resource Name"`
4. 检查存储连通性：`Test-Path \\ClusterStorage\Volume1`
5. 检查 AD 中 CNO 权限

**关键命令**：
```powershell
# 查看资源状态
Get-ClusterResource | Format-Table Name, State, OwnerGroup, OwnerNode

# 查看资源失败信息
Get-ClusterResource "Problem Resource" | Get-ClusterParameter

# 检查 WER 报告中的 DumpLogQuery
(Get-ClusterResourceType -Name "Physical Disk").DumpLogQuery

# 尝试手动上线
Start-ClusterResource "Problem Resource" -Verbose
```

### 问题 D：CSV 进入重定向模式（Redirected I/O）

**可能原因**：
- 直接存储连接丢失，I/O 被重定向到 Coordinator Node
- 磁盘进入维护模式
- ReFS 格式的 CSV 在 SAN 场景下默认使用重定向模式
- NTLM 认证问题（Windows Server 2012 R2 及以前）

**排查思路**：
```powershell
# 查看 CSV 状态
Get-ClusterSharedVolume | Format-Table Name, State, @{
    L='FileSystemRedirectedIOReason';E={$_.SharedVolumeInfo.Partition.FileSystemRedirectedIOReason}
}

# 查看 CSV 详细信息
Get-ClusterSharedVolumeState
```

### 问题 E：集群验证失败

**排查思路**：
```powershell
# 运行完整验证
Test-Cluster -Node NODE01, NODE02 -Include "Storage","Network","Inventory","System Configuration"

# 查看验证报告
# 报告生成在 %SystemRoot%\Cluster\Reports\
```

---

## 6. 实战经验 (Practical Tips)

### 最佳实践

- **始终配置 Witness**：即使是 2 节点集群也要配置 Witness（推荐 Cloud Witness），避免双节点同时失联时无法仲裁
- **奇数投票**：确保集群总投票数为奇数（节点数 + 见证 = 奇数），简化 Quorum 计算
- **分离网络**：将心跳网络和客户端访问网络分开，减少互相干扰
- **定期运行验证**：`Test-Cluster` 不仅在创建时需要运行，每次硬件/网络变更后都应重新验证
- **使用 CAU 进行滚动更新**：Cluster-Aware Updating (CAU) 可自动化节点逐一更新，保持集群可用性
- **保持节点配置一致**：相同的硬件、驱动、固件版本可减少故障转移后的兼容性问题
- **监控 Cluster Log**：定期检查 `Get-ClusterLog` 输出中的 Warning 和 Error

### 常见误区

- ❌ **"集群 = 永不宕机"**：集群提供的是高可用性而非 100% 可用性。如果所有节点同时故障或 Quorum 丢失，集群仍会停止
- ❌ **"Quorum 就是多数节点在线"**：Quorum 是基于**投票**的，不仅仅是节点数量。Witness 也参与投票
- ❌ **"节点越多越安全"**：节点数量增加意味着更多的心跳通信开销和更复杂的网络拓扑。根据业务需求选择合适的节点数
- ❌ **"心跳失败 = 节点宕机"**：心跳失败可能是网络瞬断、CPU 繁忙等原因，不一定是真正的硬件故障
- ❌ **"ForceQuorum 可以随便用"**：`Start-ClusterNode -ForceQuorum` 会绕过正常的 Quorum 机制，如果在 Split-Brain 场景下不当使用可能导致数据损坏

### 性能考量

- **心跳网络延迟**：如果跨站点延迟超过 100ms，需要调大 CrossSubnetDelay 和 Threshold 以避免误判
- **CSV Redirected I/O**：如果频繁出现重定向模式，检查存储连接和 CSV 的 Coordinator Node 分布
- **Live Migration 网络**：配置专用的 Live Migration 网络，避免占用客户端访问带宽
- **S2D 缓存**：确保缓存盘（NVMe/SSD）容量和数量足够，避免性能瓶颈

### 安全注意

- **CNO 权限**：Cluster Name Object 在 AD 中需要 "Create Computer Objects" 权限
- **Kerberos Constrained Delegation**：Live Migration 需要配置 Kerberos 委派
- **证书**：Windows Server 2016+ 的集群通信使用证书认证；确保证书有效
- **防火墙规则**：确保 "Failover Clusters" 防火墙规则组在所有节点上已启用
- **Cloud Witness 安全**：使用 SAS Token 而非直接存储 Azure 存储密钥；定期轮换 Key

---

## 7. 与相关技术的对比 (Comparison with Related Technologies)

| 维度 | Windows Failover Clustering | Linux Pacemaker/Corosync | VMware vSphere HA | Azure Availability Set |
|------|----------------------------|--------------------------|-------------------|----------------------|
| **平台** | Windows Server | Linux | VMware ESXi | Azure Cloud |
| **最大节点数** | 64 | 无硬性限制（推荐 ≤ 32） | 64 | N/A（自动管理） |
| **仲裁机制** | Quorum（投票制） | Quorum + STONITH/Fencing | Primary/Secondary HA Agent | Azure Fabric Controller |
| **存储要求** | 共享存储/S2D/ReFS | SAN/DRBD/共享存储 | VMFS/vSAN/NFS | Azure Managed Disks |
| **故障检测** | Heartbeat (UDP 3343) | Corosync (Totem) | HA Agent Heartbeat | Azure 主机健康监控 |
| **典型工作负载** | Hyper-V, SQL Server, File Server | MySQL, PostgreSQL, SAP | vSphere VM | Azure VM |
| **管理工具** | Failover Cluster Manager, PowerShell | Pacemaker CLI, HAWK Web UI | vSphere Client | Azure Portal |
| **适用场景** | 企业本地/混合云 Windows 环境 | 企业本地 Linux 环境 | 虚拟化环境 | 公有云环境 |

### 选型建议

- **Windows 生态 + 本地部署**：WSFC 是自然选择，与 Hyper-V、SQL Server 深度集成
- **混合云**：WSFC + Cloud Witness + Azure Site Recovery 提供跨本地和云的 DR 能力
- **纯 Linux 环境**：Pacemaker/Corosync 是标准方案
- **纯公有云**：使用云原生 HA 方案（如 Azure Availability Set/Zone、AWS Auto Scaling）更合适

---

## 8. 附录：关键命令速查 (Quick Command Reference)

```powershell
# ========== 集群信息 ==========
Get-Cluster                                    # 集群基本信息
Get-ClusterNode                                # 所有节点状态
Get-ClusterGroup                               # 所有角色/资源组
Get-ClusterResource                            # 所有资源
Get-ClusterNetwork                             # 集群网络
Get-ClusterQuorum                              # Quorum 配置

# ========== 日志与诊断 ==========
Get-ClusterLog -Destination C:\temp -TimeSpan 60   # 生成最近 60 分钟的集群日志
Test-Cluster -Node NODE01,NODE02                    # 集群验证
Get-ClusterDiagnosticInfo -Destination C:\temp      # 集群诊断信息（S2D 场景）

# ========== 故障转移操作 ==========
Move-ClusterGroup "SQL Server" -Node NODE02        # 手动移动角色
Start-ClusterResource "Resource Name"              # 手动上线资源
Stop-ClusterResource "Resource Name"               # 停止资源
Suspend-ClusterNode NODE01 -Drain                  # 暂停节点并排空工作负载
Resume-ClusterNode NODE01                          # 恢复节点

# ========== Quorum 管理 ==========
Set-ClusterQuorum -CloudWitness -AccountName <Name> -AccessKey <Key>
Set-ClusterQuorum -FileShareWitness "\\server\share"
Start-ClusterNode -ForceQuorum                     # 紧急强制启动

# ========== CSV 管理 ==========
Get-ClusterSharedVolume                            # CSV 列表
Get-ClusterSharedVolumeState                       # CSV 详细状态
Add-ClusterSharedVolume "Cluster Disk 1"           # 添加 CSV
Remove-ClusterSharedVolume "Cluster Disk 1"        # 移除 CSV

# ========== CAU (Cluster-Aware Updating) ==========
Add-CauClusterRole                                 # 启用自更新模式
Invoke-CauRun -ClusterName CLUSTER01               # 手动触发更新运行
Get-CauRun -ClusterName CLUSTER01                  # 查看更新状态
```

---

## 9. 参考资料 (References)

- [Failover Clustering Overview](https://learn.microsoft.com/en-us/windows-server/failover-clustering/failover-clustering-overview) — WSFC 官方概述，涵盖架构、功能和资源链接
- [Manage Cluster Quorum (Configure Quorum Witnesses)](https://learn.microsoft.com/en-us/windows-server/failover-clustering/manage-cluster-quorum) — Quorum Witness 配置详细指南（Cloud Witness、Disk Witness、File Share Witness）
- [Failover Clustering Hardware Requirements and Storage Options](https://learn.microsoft.com/en-us/windows-server/failover-clustering/clustering-requirements) — 硬件和存储配置要求
- [Cluster Shared Volumes (CSV)](https://learn.microsoft.com/en-us/windows-server/failover-clustering/failover-cluster-csvs) — CSV 工作原理、I/O 模式和配置指南
- [Cluster-Aware Updating (CAU)](https://learn.microsoft.com/en-us/windows-server/failover-clustering/cluster-aware-updating) — 集群感知更新功能详解
- [Health Service Overview](https://learn.microsoft.com/en-us/windows-server/failover-clustering/health-service-overview) — S2D 场景下的 Health Service 自动化管理
- [Troubleshooting using Windows Error Reporting](https://learn.microsoft.com/en-us/windows-server/failover-clustering/troubleshooting-using-wer-reports) — 使用 WER 报告排查集群资源故障
- [Failover Clustering System Events](https://learn.microsoft.com/en-us/windows-server/failover-clustering/system-events) — 集群系统事件日志完整参考（Event 1000-1574+）

---
---

# Deep Dive: Windows Server Failover Clustering

**Topic:** Windows Server Failover Clustering (WSFC)
**Category:** High Availability / Clustering
**Level:** Intermediate → Advanced
**Last Updated:** 2026-03-10

---

## 1. Overview

Windows Server Failover Clustering (WSFC) is a built-in high availability infrastructure in Windows Server. It organizes multiple independent servers (called **Nodes**) into a logical cluster. When a node experiences hardware failure, software crash, or network interruption, other nodes in the cluster automatically **take over** the workloads (called **Failover**), ensuring continuous availability of critical business applications.

The core problem WSFC solves is **Single Point of Failure (SPOF)**. In a traditional single-server architecture, if the server goes down, all services are interrupted. WSFC uses redundant nodes + shared storage + automatic fault detection and failover mechanisms to reduce service interruption from "hours" to "seconds" or even "zero downtime" (e.g., Live Migration).

Within the Microsoft ecosystem, WSFC underpins these critical products:
- **Hyper-V High Availability**: VMs can automatically migrate between nodes
- **SQL Server Always On**: Database Failover Cluster Instances (FCI) and Availability Groups (AG)
- **Scale-Out File Server (SOFS)**: Horizontally scaled file services
- **Azure Stack HCI**: Hyperconverged infrastructure cluster management
- **Storage Spaces Direct (S2D)**: Software-defined storage

---

## 2. Core Concepts

### 2.1 Node

- **Definition**: Each server (physical or virtual) in the cluster
- **Requirements**: All nodes should run the same Windows Server version with consistent hardware configurations
- **States**: Up (Online), Down (Offline), Paused (won't accept new failovers), Joining
- **Maximum count**: Windows Server 2016+ supports up to **64 nodes**

> **Analogy**: Think of a cluster as a relay race team. Each node is a runner. When one runner falls, the next runner immediately picks up the baton and continues — the race (service) never stops.

### 2.2 Clustered Role / Resource Group

- **Definition**: A logical collection of related resources running on the cluster, representing a failover-capable workload
- **Examples**: A Hyper-V VM, a SQL Server instance, a file server role
- **Resources**: Individual components that make up a role — IP address, network name, disk, service, etc.
- **Resource Dependencies**: Define startup order and dependencies between resources. E.g., file server depends on: IP Address → Network Name → Disk
- **Owner Node**: The node currently running the role
- **Preferred Owner**: Admin-configured list of preferred nodes

### 2.3 Quorum

Quorum is one of the most critical concepts in Failover Clustering. It's the mechanism for making decisions in "split" scenarios.

- **Core problem**: When a Network Partition (Split-Brain) occurs, both halves of the cluster think they are "correct." If both continue running and writing to shared storage, data corruption occurs
- **Solution**: **Voting mechanism**. Each node and witness has one vote (or dynamically adjusted via Dynamic Quorum). Only the partition holding a **majority** (> 50%) continues running; the other half automatically stops
- **Formula**: For N voting members, ⌊N/2⌋ + 1 votes are needed to maintain Quorum

#### Quorum Modes

| Mode | Voting Members | Use Case |
|------|---------------|----------|
| **Node Majority** | Nodes only | Odd number of nodes (3, 5, etc.) |
| **Node & Disk Majority** | Nodes + shared disk witness | Even nodes + shared storage available |
| **Node & File Share Majority** | Nodes + file share witness | Even nodes + no shared storage (e.g., cross-site) |
| **Cloud Witness** | Nodes + Azure Blob witness | Recommended for all topologies (Windows Server 2016+) |

#### Dynamic Quorum

Introduced in Windows Server 2012 R2+, the cluster **dynamically adjusts** each node's voting weight:
- When a node gracefully goes offline, the cluster revokes its vote and recalculates Quorum
- This means a 5-node cluster can survive progressive node loss while maintaining Quorum
- Check and configure via `(Get-Cluster).DynamicQuorum`

### 2.4 Heartbeat

- **Definition**: Periodic UDP packets sent between cluster nodes (default port 3343) to detect node availability
- **Default parameters**:
  - **SameSubnetDelay**: Same-subnet heartbeat interval, default 1000ms
  - **SameSubnetThreshold**: Same-subnet failure threshold, default 20 (on Windows Server 2016+)
  - **CrossSubnetDelay**: Cross-subnet heartbeat interval, default 1000ms
  - **CrossSubnetThreshold**: Cross-subnet failure threshold, default 40 (on Windows Server 2016+)
- **Detection logic**: If N consecutive heartbeats fail (N = Threshold), the node is declared Down

> **Key insight**: Heartbeat failure doesn't necessarily mean the node is truly down. Transient network interruptions or CPU saturation causing delays can trigger false positives — a common pitfall in troubleshooting.

### 2.5 Cluster Shared Volumes (CSV)

- **Definition**: A cluster file system layer (CSVFS) that allows all cluster nodes to **simultaneously read/write** the same LUN/disk
- **Core value**: Traditional cluster disks can only be "owned" by one node at a time (exclusive); CSV removes this limitation
- **Access path**: All nodes access through unified path `C:\ClusterStorage\Volume1`
- **Primary uses**: Hyper-V VHDX files, S2D storage volumes, Scale-Out File Server data
- **I/O Modes**:
  - **Direct I/O** (normal): Node accesses underlying SAN storage directly — best performance
  - **Redirected I/O**: I/O redirected via network to Coordinator Node — during maintenance or filesystem misalignment
  - **Block Redirected I/O**: S2D scenario — accessing data blocks not stored locally

### 2.6 Cluster Network

Cluster networks typically serve different roles:

| Network Role | Purpose | Description |
|-------------|---------|-------------|
| **Cluster Only** | Internal cluster communication only | Heartbeat, cluster state sync |
| **Client Access** | Client access only | Service-facing network |
| **Cluster and Client** | Both | Default configuration |
| **Not Used** | Not used by cluster | Management or iSCSI dedicated network |

---

## 3. How It Works

### 3.1 Overall Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Failover Cluster                          │
│                                                             │
│  ┌──────────┐    Heartbeat (UDP 3343)    ┌──────────┐       │
│  │  Node 1  │◄──────────────────────────►│  Node 2  │       │
│  │ (Active) │                            │(Standby) │       │
│  │          │    Heartbeat (UDP 3343)    │          │       │
│  │ ┌──────┐ │◄──────────────────────────►│ ┌──────┐ │       │
│  │ │ Role │ │                            │ │(idle)│ │       │
│  │ │ (VM) │ │     ┌──────────┐           │ └──────┘ │       │
│  │ └──────┘ │     │  Node 3  │           │          │       │
│  └─────┬────┘     │(Standby) │           └──────────┘       │
│        │          └──────────┘                              │
│        │                                                    │
│  ┌─────▼──────────────────────────────────────────┐         │
│  │          Shared Storage / CSV / S2D             │         │
│  │    ┌────────┐  ┌────────┐  ┌────────┐          │         │
│  │    │ Disk 1 │  │ Disk 2 │  │Witness │          │         │
│  │    │ (CSV)  │  │ (CSV)  │  │ (Quor) │          │         │
│  │    └────────┘  └────────┘  └────────┘          │         │
│  └────────────────────────────────────────────────┘         │
│                                                             │
│  ┌─────────────────────────────────────────────┐            │
│  │  Cluster Database (CLUSDB) - replicated to   │            │
│  │  every node                                  │            │
│  └─────────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Cluster Formation Process

1. **Step 1: Validation**
   - Run `Test-Cluster` to verify all nodes meet hardware, software, network, and storage requirements
   - This is a **mandatory** step before cluster creation

2. **Step 2: Create Cluster**
   - Run `New-Cluster` to create a Cluster Name Object (CNO) in Active Directory
   - Initialize the cluster database (CLUSDB), replicated to all nodes
   - Assign cluster IP address and network name

3. **Step 3: Configure Quorum**
   - System automatically selects Quorum mode (Cloud Witness recommended on 2016+)
   - Administrators can manually adjust via `Set-ClusterQuorum`

4. **Step 4: Add Storage and Roles**
   - Add shared disks to cluster, configure as CSV
   - Create clustered roles (Hyper-V VM, File Server, etc.)

### 3.3 Failover Process

This is the cluster's most critical workflow. When Node 1 fails:

```
Timeline:
t=0s     Node 2/3 detect heartbeat loss from Node 1
t=1-10s  Consecutive heartbeat timeouts accumulate to threshold
t=10-20s Cluster declares Node 1 as Down (Event 1135)
         ↓
         Quorum check: Do remaining nodes hold majority votes?
         ├── Yes → Proceed with failover
         └── No  → Cluster service stops (Event 1177: Quorum Lost)
         ↓
t=20-25s Resource Hosting Subsystem (RHS) starts resources on target node
         ├── Resources start in dependency order:
         │   1. Physical Disk / CSV → Online
         │   2. IP Address → Online
         │   3. Network Name → Online
         │   4. Application/Service → Online
         └── Each resource has independent timeout settings
         ↓
t=25-60s Role fully operational on new node
         Clients reconnect via cluster network name
```

#### Failover Types

| Type | Trigger | Impact |
|------|---------|--------|
| **Planned Failover** | Admin manually moves role | Zero downtime (starts on target before stopping on source) |
| **Unplanned Failover** | Automatic on node failure | Brief interruption (seconds to minutes) |
| **Live Migration** | Admin or CAU-triggered VM migration | Near-zero downtime for Hyper-V VMs |
| **Quick Migration** | Saves VM state before migration | Noticeable interruption |

### 3.4 Health Monitoring Mechanisms

The cluster continuously monitors health through multiple layers:

**Layer 1: Node Level — Heartbeat**
- Nodes exchange heartbeats via UDP port 3343
- Once per second (default); consecutive failures reaching threshold declare node Down

**Layer 2: Resource Level — IsAlive / LooksAlive**
- **LooksAlive**: Lightweight check (default every 5 seconds) — verifies resource process exists
- **IsAlive**: Deep check (default every 60 seconds) — verifies resource is truly functional (e.g., database responds to queries)
- Consecutive failures trigger escalation: **Resource Restart → Failover → No Further Action**

**Layer 3: Health Service (Windows Server 2016+)**
- Primarily for Storage Spaces Direct scenarios
- Provides disk lifecycle management, performance history, fault reporting
- Automatically handles disk retirement and data rebuild

---

## 4. Key Configurations

### 4.1 Heartbeat and Failure Detection Parameters

| Parameter | Default (2016+) | Description | Tuning Scenario |
|-----------|-----------------|-------------|-----------------|
| `SameSubnetDelay` | 1000ms | Same-subnet heartbeat interval | Increase when network latency is high |
| `SameSubnetThreshold` | 20 | Same-subnet failure threshold | Increase when false positives occur |
| `CrossSubnetDelay` | 1000ms | Cross-subnet heartbeat interval | Evaluate for cross-site clusters |
| `CrossSubnetThreshold` | 40 | Cross-subnet failure threshold | Evaluate for cross-site clusters |

```powershell
# View current heartbeat parameters
(Get-Cluster).SameSubnetDelay
(Get-Cluster).SameSubnetThreshold

# Modify example (increase tolerance)
(Get-Cluster).SameSubnetThreshold = 30
(Get-Cluster).CrossSubnetThreshold = 60
```

### 4.2 Resource Recovery Policy

| Parameter | Default | Description | Tuning Scenario |
|-----------|---------|-------------|-----------------|
| `RestartAction` | Restart | Action on resource failure | Some resources shouldn't auto-restart |
| `RestartThreshold` | 3 | Max restarts within RestartPeriod | Frequently restarting resources |
| `RestartPeriod` | 900000ms (15min) | Restart count time window | Pair with RestartThreshold |
| `FailoverThreshold` | 0xFFFFFFFF | Max failovers within FailoverPeriod | Prevent "ping-pong" effect |
| `FailoverPeriod` | 21600000ms (6h) | Failover count time window | Pair with FailoverThreshold |

```powershell
# View resource recovery policy
Get-ClusterResource "SQL Server" | Format-List RestartAction, RestartThreshold, RestartPeriod

# Set failover threshold (prevent ping-pong)
(Get-ClusterGroup "SQL Server").FailoverThreshold = 3
(Get-ClusterGroup "SQL Server").FailoverPeriod = 21600
```

### 4.3 Quorum Configuration

```powershell
# View current Quorum configuration
Get-ClusterQuorum

# Configure Cloud Witness
Set-ClusterQuorum -CloudWitness -AccountName <StorageAccountName> -AccessKey <Key>

# Configure File Share Witness
Set-ClusterQuorum -FileShareWitness "\\fileserver01\witness-share"

# Configure Disk Witness
Set-ClusterQuorum -DiskWitness "Cluster Disk 1"
```

---

## 5. Common Issues & Troubleshooting

### Issue A: Node Unexpectedly Removed from Cluster (Event 1135)

**Typical Event**:
```
Event 1135: Cluster node 'NODE01' was removed from the active failover cluster membership.
```

**Possible Causes**:
- Transient network interruption causing heartbeat timeout
- Node CPU at 100% causing heartbeat response delay
- Firewall blocking UDP 3343
- Network driver/firmware issues
- VMQ or RSS misconfiguration

**Troubleshooting Approach**:
1. Generate Cluster Log: `Get-ClusterLog -Destination C:\temp -TimeSpan 60`
2. Search for `lost communication` and `missed heartbeat` in the log
3. Check node CPU/memory utilization at time of failure
4. Check `Get-NetAdapterStatistics` for errors/drops
5. Verify Windows Firewall "Failover Clusters" rules are enabled

### Issue B: Quorum Lost, Cluster Stops (Event 1177)

**Possible Causes**:
- Majority of nodes simultaneously down or network-isolated
- Witness unavailable + node loss
- Dynamic Quorum disabled + progressive node departure

**Key Commands**:
```powershell
Get-ClusterQuorum
Get-ClusterNode | Format-Table Name, State, DynamicWeight, NodeWeight
# Emergency: Force start
Start-ClusterNode -Name NODE01 -ForceQuorum
```

### Issue C: Resource Failed to Come Online (Event 1069)

**Possible Causes**:
- Storage connectivity loss (SAN path down)
- Disk locked by another process
- Service dependency missing
- IP address conflict
- CNO permission insufficient in AD

**Key Commands**:
```powershell
Get-ClusterResource | Format-Table Name, State, OwnerGroup, OwnerNode
Start-ClusterResource "Problem Resource" -Verbose
(Get-ClusterResourceType -Name "Physical Disk").DumpLogQuery
```

### Issue D: CSV Enters Redirected I/O Mode

**Possible Causes**:
- Direct storage connection lost
- Disk in maintenance mode
- ReFS-formatted CSV on SAN defaults to redirected mode

**Key Commands**:
```powershell
Get-ClusterSharedVolume | Select Name, State
Get-ClusterSharedVolumeState
```

---

## 6. Practical Tips

### Best Practices

- **Always configure a Witness**: Even 2-node clusters need a Witness (Cloud Witness recommended)
- **Odd vote count**: Ensure total votes (nodes + witness) is odd for simpler Quorum math
- **Separate networks**: Keep heartbeat and client access on separate networks
- **Run validation regularly**: `Test-Cluster` after every hardware/network change
- **Use CAU for rolling updates**: Cluster-Aware Updating automates node-by-node patching
- **Keep node configurations consistent**: Same hardware, drivers, and firmware versions
- **Monitor Cluster Logs**: Regularly review `Get-ClusterLog` for warnings and errors

### Common Misconceptions

- ❌ **"Cluster = never goes down"**: Clustering provides high availability, not 100% uptime. If all nodes fail or Quorum is lost, the cluster stops
- ❌ **"Quorum just means majority of nodes online"**: Quorum is vote-based — Witnesses also vote
- ❌ **"More nodes = safer"**: More nodes means more heartbeat overhead and complex network topology
- ❌ **"Heartbeat failure = node crashed"**: Could be transient network issue or CPU saturation
- ❌ **"ForceQuorum is safe to use anytime"**: It bypasses normal Quorum and can cause data corruption in Split-Brain scenarios

### Performance Considerations

- **Heartbeat network latency**: For cross-site latency > 100ms, increase CrossSubnetDelay and Threshold
- **CSV Redirected I/O**: Frequent redirection indicates storage connectivity issues
- **Live Migration network**: Use dedicated network to avoid consuming client bandwidth
- **S2D cache**: Ensure sufficient NVMe/SSD cache capacity

### Security Notes

- **CNO permissions**: Cluster Name Object needs "Create Computer Objects" in AD
- **Kerberos Constrained Delegation**: Required for Live Migration
- **Certificates**: Windows Server 2016+ cluster communication uses certificate authentication
- **Firewall rules**: Ensure "Failover Clusters" firewall rule group is enabled on all nodes
- **Cloud Witness security**: Uses SAS Tokens instead of storing Azure storage keys directly; rotate keys regularly

---

## 7. Comparison with Related Technologies

| Dimension | Windows Failover Clustering | Linux Pacemaker/Corosync | VMware vSphere HA | Azure Availability Set |
|-----------|----------------------------|--------------------------|-------------------|----------------------|
| **Platform** | Windows Server | Linux | VMware ESXi | Azure Cloud |
| **Max Nodes** | 64 | No hard limit (≤32 recommended) | 64 | N/A (auto-managed) |
| **Quorum** | Vote-based Quorum | Quorum + STONITH/Fencing | Primary/Secondary HA Agent | Azure Fabric Controller |
| **Storage** | Shared storage/S2D/ReFS | SAN/DRBD/Shared | VMFS/vSAN/NFS | Azure Managed Disks |
| **Fault Detection** | Heartbeat (UDP 3343) | Corosync (Totem) | HA Agent Heartbeat | Azure host health monitoring |
| **Typical Workloads** | Hyper-V, SQL Server, File Server | MySQL, PostgreSQL, SAP | vSphere VMs | Azure VMs |
| **Best For** | Enterprise on-premises/hybrid Windows | Enterprise on-premises Linux | Virtualized environments | Public cloud |

---

## 8. Quick Command Reference

```powershell
# ========== Cluster Information ==========
Get-Cluster                                    # Basic cluster info
Get-ClusterNode                                # All node states
Get-ClusterGroup                               # All roles/resource groups
Get-ClusterResource                            # All resources
Get-ClusterNetwork                             # Cluster networks
Get-ClusterQuorum                              # Quorum configuration

# ========== Logs & Diagnostics ==========
Get-ClusterLog -Destination C:\temp -TimeSpan 60   # Generate cluster log (last 60 min)
Test-Cluster -Node NODE01,NODE02                    # Cluster validation
Get-ClusterDiagnosticInfo -Destination C:\temp      # Cluster diagnostics (S2D)

# ========== Failover Operations ==========
Move-ClusterGroup "SQL Server" -Node NODE02        # Manually move role
Start-ClusterResource "Resource Name"              # Manually online resource
Suspend-ClusterNode NODE01 -Drain                  # Pause node and drain workloads
Resume-ClusterNode NODE01                          # Resume node

# ========== Quorum Management ==========
Set-ClusterQuorum -CloudWitness -AccountName <Name> -AccessKey <Key>
Start-ClusterNode -ForceQuorum                     # Emergency force start

# ========== CSV Management ==========
Get-ClusterSharedVolume                            # List CSVs
Get-ClusterSharedVolumeState                       # Detailed CSV status
Add-ClusterSharedVolume "Cluster Disk 1"           # Add CSV

# ========== CAU (Cluster-Aware Updating) ==========
Add-CauClusterRole                                 # Enable self-updating mode
Invoke-CauRun -ClusterName CLUSTER01               # Trigger update run
Get-CauRun -ClusterName CLUSTER01                  # Check update status
```

---

## 9. References

- [Failover Clustering Overview](https://learn.microsoft.com/en-us/windows-server/failover-clustering/failover-clustering-overview) — Official WSFC overview covering architecture, features, and resource links
- [Manage Cluster Quorum (Configure Quorum Witnesses)](https://learn.microsoft.com/en-us/windows-server/failover-clustering/manage-cluster-quorum) — Detailed Quorum Witness configuration guide (Cloud, Disk, File Share)
- [Failover Clustering Hardware Requirements and Storage Options](https://learn.microsoft.com/en-us/windows-server/failover-clustering/clustering-requirements) — Hardware and storage configuration requirements
- [Cluster Shared Volumes (CSV)](https://learn.microsoft.com/en-us/windows-server/failover-clustering/failover-cluster-csvs) — CSV internals, I/O modes, and configuration guide
- [Cluster-Aware Updating (CAU)](https://learn.microsoft.com/en-us/windows-server/failover-clustering/cluster-aware-updating) — Automated rolling update feature for failover clusters
- [Health Service Overview](https://learn.microsoft.com/en-us/windows-server/failover-clustering/health-service-overview) — Health Service automation for Storage Spaces Direct
- [Troubleshooting using Windows Error Reporting](https://learn.microsoft.com/en-us/windows-server/failover-clustering/troubleshooting-using-wer-reports) — Using WER reports to troubleshoot cluster resource failures
- [Failover Clustering System Events](https://learn.microsoft.com/en-us/windows-server/failover-clustering/system-events) — Complete reference for cluster system event log entries
