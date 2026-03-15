---
layout: post
title: "Deep Dive: Windows Failover Clustering 完全指南"
date: 2026-03-13
categories: [Knowledge, Infrastructure]
tags: [failover-clustering, high-availability, quorum, csv, windows-server, troubleshooting]
type: "deep-dive"
---

# Deep Dive: Windows Server Failover Clustering

**Topic:** Windows Server Failover Clustering  
**Category:** Infrastructure / High Availability  
**Level:** 中级到高级 / Intermediate to Advanced  
**Last Updated:** 2026-03-13

---

# 中文版

## 1. 概述 — 什么是故障转移群集

**故障转移群集 (Failover Cluster)** 是一组独立的服务器，它们协同工作如同一个单一系统，为关键业务应用提供**高可用性 (High Availability)**。

### 群集能提供什么？

| 能力 | 说明 |
|------|------|
| **高可用性 (HA)** | 消除单点故障，确保服务持续运行 |
| **故障转移 (Failover)** | 当节点故障时，自动将服务迁移到健康节点 |
| **负载均衡** | Active/Active 模式下分担工作负载 |
| **零停机维护** | 通过滚动更新实现不停机的补丁和升级 |

### Failover Clustering vs NLB

| 维度 | Failover Clustering | Network Load Balancing (NLB) |
|------|-------------------|----------------------------|
| **目的** | 应用级高可用 | 网络流量负载均衡 |
| **故障检测** | 主动健康检查 | 心跳检测 |
| **共享存储** | 需要（传统模式） | 不需要 |
| **适用场景** | 数据库、文件服务器、Exchange | Web 服务器、IIS |
| **最大节点数** | 64 (2012+) | 32 |
| **IP 地址** | 虚拟 IP 跟随资源移动 | 所有节点共享虚拟 IP |

> 🏠 **类比**：Failover Clustering 像是一个手术室有备用医生，主刀医生倒下，备用医生立刻接手继续手术；NLB 像是多个窗口同时办理业务，分散客流。

## 2. 核心术语

### 2.1 基础概念

| 术语 | 英文 | 说明 |
|------|------|------|
| **群集 (Cluster)** | Cluster | 一组协同工作的独立服务器 |
| **节点 (Node)** | Node | 群集中的每一台服务器 |
| **资源 (Resource)** | Resource | 群集管理的最小单元（如 IP 地址、磁盘、网络名称、服务） |
| **资源组 (Resource Group)** | Resource Group | 一组相关资源的集合，作为一个整体进行故障转移 |
| **故障转移 (Failover)** | Failover | 资源从故障节点自动迁移到健康节点 |
| **故障回复 (Failback)** | Failback | 资源在原节点恢复后迁回原节点 |

### 2.2 关键对象

| 术语 | 说明 |
|------|------|
| **客户端接入点 (CAP)** | 客户端访问群集服务的网络名称和 IP 地址 |
| **群集名称对象 (CNO)** | 群集本身在 Active Directory 中的计算机对象 |
| **虚拟计算机对象 (VCO)** | 群集中每个角色/服务在 AD 中创建的计算机对象 |
| **见证资源 (Witness)** | 帮助决定仲裁的额外投票资源（磁盘或文件共享） |
| **共享磁盘** | 至少两个节点可以访问的存储 |

### 2.3 服务组件

| 组件 | 说明 |
|------|------|
| **群集服务 (ClusSvc)** | 运行在每个节点上的核心服务，管理群集操作 |
| **RHS (Resource Host Subsystem)** | 资源宿主子系统，替代了旧版的资源监视器，托管和监控资源 DLL |
| **NetFT** | 群集虚拟网络适配器驱动，提供容错通信 |

## 3. 群集架构 (Deep Dive)

### 3.1 三层架构

```
┌─────────────────────────────────────────────────┐
│           顶层：群集抽象 (Abstractions)            │
│   节点、资源组、资源、故障转移策略、依赖关系         │
│   资源状态管理、故障响应                           │
├─────────────────────────────────────────────────┤
│           中层：群集操作 (Operation)               │
│   成员管理、重组操作 (Regroup)                     │
│   跨节点配置一致性维护                             │
│   GUM (全局更新管理器) 序列化更新                   │
├─────────────────────────────────────────────────┤
│           底层：操作系统交互 (OS Interaction)       │
│   分区管理器、群集磁盘驱动、NetFT 网络驱动          │
│   文件系统、安全、网络接口管理                      │
└─────────────────────────────────────────────────┘
```

### 3.2 核心组件详解

```
群集服务组件关系图
═══════════════════════════════════════════

  客户端请求
      │
      ▼
┌──────────────┐     ┌──────────────────┐
│ Resource      │     │ Topology Manager  │
│ Control       │     │ 发现和维护网络拓扑 │
│ Manager (RCM) │     │ 调用 clnet 枚举   │
│ 故障转移策略   │     │ 所有网络接口       │
│ 资源依赖树     │     └──────────────────┘
└──────┬───────┘              │
       │                      ▼
       ▼              ┌──────────────────┐
┌──────────────┐     │ Database Manager  │
│ Object       │     │ 高可用容错数据库   │
│ Manager      │◄───►│ 注册表副本管理     │
│ 内存关系数据库│     │ CLFS 日志文件      │
│ 群集对象管理  │     │ Paxos 一致性算法   │
└──────────────┘     └──────────────────┘
       │                      │
       ▼                      ▼
┌──────────────┐     ┌──────────────────┐
│ Membership   │     │ Global Update     │
│ Manager (MM) │     │ Manager (GUM)     │
│ 重组算法      │     │ 序列化原子更新     │
│ Gossip 协议   │     │ Locker 节点模型   │
│ 节点加入/故障 │     │ GUM 锁机制        │
└──────┬───────┘     └──────────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────────┐
│ Host Manager │     │ Quorum Manager   │
│ TCP 3343端口  │     │ 仲裁判定          │
│ 节点连接管理  │     │ 投票计算          │
│ 安全握手      │────►│ Paxos 标签       │
│ NetFT路由更新 │     │ 配置副本跟踪      │
└──────────────┘     └──────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│   Security Manager + NetFT Driver    │
│   SSPI 握手 → 密钥协商 → 安全通道     │
│   消息签名/加密 → 容错多路径通信       │
└──────────────────────────────────────┘
```

### 3.3 关键组件说明

**Messaging（消息传递）**：
- 提供所有群集内通信原语
- 单播 (Unicast) + 多播 (GEM - Good Enough Multicast)
- 可靠点对点通信

**Membership Manager（成员管理器）**：
- 运行多阶段 Regroup 算法达成共识
- 使用 Gossip 协议
- 支持的场景：节点加入、节点故障、通信问题、全网格拓扑故障

**Regroup 过程（重组）**：

```
Opening → Closing → Pruning → PruneAck → GemRepair → Cleanup → Stable
   │         │         │          │          │          │         │
   │         │         │          │          │          │         └─ 稳定状态
   │         │         │          │          │          └─ 清理旧状态
   │         │         │          │          └─ 修复多播
   │         │         │          └─ 确认修剪
   │         │         └─ 移除故障节点
   │         └─ 关闭旧成员视图
   └─ 开始新的成员关系协商
```

**Global Update Manager (GUM)**：
- 所有配置更新的序列化和原子性保证
- 更新先应用到 Locker 节点，成功后再发送到其他节点
- 节点必须获取 GUM 锁才能发布更新

**Resource Control Manager (RCM)**：
- 实现故障转移机制和策略
- 维护资源状态：Online / Offline / Failed / Online Pending / Offline Pending
- 维护资源组状态：Online / Offline / Partial Online / Failed
- 建立和维护资源依赖树

## 4. 仲裁 (Quorum) — 群集的大脑

### 4.1 什么是 Quorum？

Quorum（仲裁）是群集用来决定是否有足够的"投票"来保持服务在线的共识机制。

> 🏠 **类比**：Quorum 像是董事会投票。必须有超过半数的董事到场（达到法定人数），决议才有效。群集也一样 — 必须有足够的节点"投票"同意，服务才能上线。

### 4.2 仲裁模型

| 模型 | 投票组成 | 适用场景 |
|------|---------|---------|
| **Node Majority** | 仅节点投票 | 奇数节点 |
| **Node + Disk Majority** | 节点 + 见证磁盘 | 偶数节点 + 共享磁盘 |
| **Node + File Share Majority** | 节点 + 文件共享见证 | 多站点群集 |
| **No Majority (Disk Only)** | 仅磁盘 | 不推荐 |

**基本公式**：
```
总投票数应为奇数
节点数为偶数 → 加一个见证（磁盘或文件共享）
节点数为奇数 → 可以不需要见证

群集可用条件：存活投票数 > 总投票数 / 2
```

### 4.3 动态仲裁 (Dynamic Quorum)

Windows Server 2012 引入，默认启用。

- 群集持续监控成员状态，动态调整投票权重
- 允许群集在超过 50% 节点失败后仍然运行
- 理论上群集可以在**仅一个节点**运行时继续提供服务
- 被称为 **"Last Man Standing"（最后一人站立）** 模型

```powershell
# 查看 Dynamic Quorum 状态
(Get-Cluster).DynamicQuorum   # 1 = 启用

# 查看当前仲裁配置
Get-ClusterQuorum

# 设置仲裁模型
Set-ClusterQuorum -NodeAndDiskMajority "Cluster Disk 1"
Set-ClusterQuorum -NodeAndFileShareMajority "\\fileserver\share"
```

### 4.4 Force Quorum 和 Prevent Quorum

| 选项 | 命令 | 用途 |
|------|------|------|
| **Force Quorum (/FQ)** | `net start clussvc /FQ` | 强制启动群集服务，即使投票不足（DR 站点场景） |
| **Prevent Quorum (/PQ)** | `net start clussvc /PQ` | 启动群集服务但不允许形成群集，只能加入现有群集 |

### 4.5 节点权重 (Node Weighting)

用于多站点群集场景，控制哪些节点参与仲裁计算：

```powershell
# 查看节点权重
(Get-ClusterNode "Node1").NodeWeight

# 设置节点权重为 0（不参与投票）
(Get-ClusterNode "DRNode1").NodeWeight = 0
```

### 4.6 Paxos 标签

群集使用 Paxos 一致性算法跟踪配置副本：
```
<PaxosTag> 2026/03/13-14:25:22.523_41:2026/03/13-14:25:22.523_41:13192 </PaxosTag>
```

## 5. 群集网络

### 5.1 NetFT 虚拟适配器

- PnP 驱动，在设备管理器中显示为网络适配器
- MAC 地址基于第一个物理 NIC
- 提供容错多路径通信
- 与 IPsec、DHCP、主机防火墙互操作

### 5.2 心跳机制 (Heartbeat)

```
节点A                              节点B
  │                                  │
  │──── UDP 3343 (Heartbeat) ────►   │
  │◄─── UDP 3343 (Heartbeat) ────   │
  │                                  │
  │  如果连续 N 次没有收到心跳        │
  │  → 标记对方为 Unreachable        │
  │  → 触发 Regroup 过程             │
```

**心跳参数：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| **SameSubnetDelay** | 1000ms | 同子网心跳间隔 |
| **CrossSubnetDelay** | 1000ms | 跨子网心跳间隔 |
| **SameSubnetThreshold** | 10 | 同子网丢失心跳次数阈值 |
| **CrossSubnetThreshold** | 20 | 跨子网丢失心跳次数阈值 |

```powershell
# 查看心跳设置
(Get-Cluster).SameSubnetThreshold
(Get-Cluster).CrossSubnetThreshold

# 调整心跳阈值（更宽容）
(Get-Cluster).SameSubnetThreshold = 20
(Get-Cluster).CrossSubnetThreshold = 40
```

### 5.3 网络角色

| 角色 | 值 | 说明 |
|------|---|------|
| **Private (仅群集)** | Role 1 | 仅内部群集通信，不绑定群集 IP，承载 CSV 和 Live Migration 流量 |
| **Public (客户端+群集)** | Role 3 | 客户端访问 + 内部群集通信 |
| **Not Used** | Role 0 | 群集不使用，不监控健康状态 |

> ⚠️ 群集加入时使用 TCP，运行时心跳使用 UDP 3343。

## 6. 群集存储

### 6.1 共享存储要求

- 至少两个节点可以访问
- 必须支持 **SCSI-3 持久预留 (Persistent Reservations)**
- 支持的类型：**基本磁盘 (MBR)** 和 **GPT**

**支持的存储接口：**

| 接口 | 说明 |
|------|------|
| **Fibre Channel (FC)** | 最传统的 SAN 连接 |
| **Serial Attached SCSI (SAS)** | 直连共享存储 |
| **iSCSI** | 基于 IP 网络的存储 |
| **Storage Spaces Direct (S2D)** | 使用本地磁盘的超融合方案 |
| **Shared VHDX** | 虚拟机共享磁盘 |

### 6.2 磁盘隔离 (Disk Fencing)

磁盘隔离是控制磁盘访问权限的过程：
- 群集磁盘驱动从 PnP 管理器收到新磁盘通知
- 确认为群集磁盘后，**PartMgr.sys** 执行隔离
- 使用 `DISK_ONLINE` 和 `DISK_OFFLINE` IOCTL
- 主动节点上磁盘 Online，被动节点上磁盘 Offline

### 6.3 Cluster Shared Volumes (CSV)

CSV 是群集存储的核心技术：

```
┌──────────────────────────────────────────────┐
│              CSV 架构                         │
├──────────────────────────────────────────────┤
│                                              │
│   Node 1 (协调者/Owner)    Node 2            │
│   ┌─────────────┐        ┌─────────────┐   │
│   │ 元数据操作    │        │ 直接 I/O     │   │
│   │ (NTFS 元数据  │◄──────│ (读写 VHD    │   │
│   │  通过此节点)  │ SMB    │  直接到磁盘)  │   │
│   └──────┬──────┘        └─────────────┘   │
│          │                                   │
│          ▼                                   │
│   ┌─────────────────────────┐               │
│   │   共享存储 (LUN)          │               │
│   │   C:\ClusterStorage\xxx  │               │
│   └─────────────────────────┘               │
│                                              │
│   特点：                                      │
│   • 所有节点并发访问                            │
│   • 元数据操作路由到协调者节点                    │
│   • 数据 I/O 直接写盘（高性能）                  │
│   • 基于 NTFS 文件系统                          │
│   • 挂载点: C:\ClusterStorage\                 │
└──────────────────────────────────────────────┘
```

**CSV 管理命令：**
```powershell
Add-ClusterSharedVolume -Name "Cluster Disk 2"
Get-ClusterSharedVolume
Move-ClusterSharedVolume -Name "Cluster Disk 2" -Node "Node2"
Remove-ClusterSharedVolume -Name "Cluster Disk 2"
```

**CSV 维护注意：**
- 运行 CHKDSK、Defrag、Shrink 需要将卷置于 **Redirected Access** 模式
- 必须从**协调者节点**发起

## 7. 资源管理

### 7.1 资源状态

```
              ┌───────────┐
         ┌───►│  Online   │◄───┐
         │    └─────┬─────┘    │
         │          │ 故障      │ 恢复
         │          ▼          │
    ┌────┴────┐  ┌─────┐  ┌──┴──────┐
    │ Online  │  │Failed│  │ Offline  │
    │ Pending │  └──┬──┘  │ Pending  │
    └─────────┘     │     └──────────┘
                    │ 重启/转移
                    ▼
              ┌───────────┐
              │  Offline   │
              └───────────┘
```

### 7.2 资源依赖 (Dependencies)

资源可以依赖同一资源组内的其他资源，支持 **AND** 和 **OR** 操作符：

```
文件服务器资源依赖树示例：
═══════════════════════

    File Server Role
         │
    ┌────┴────┐
    │   AND   │
    ├─────────┤
    │         │
 Network   Physical
  Name      Disk
    │
    │ (OR)
    ├───────────┐
    │           │
  IPv4        IPv6
 Address     Address
```

### 7.3 资源策略 (默认值)

| 策略 | 默认值 | 说明 |
|------|--------|------|
| **重启次数** | 1 次 | 资源故障后在当前节点重启的次数 |
| **重启窗口** | 15 分钟 | 如果在此时间内再次故障，执行故障转移 |
| **故障转移策略** | 转移整个资源组 | 到另一个节点 |
| **重试间隔** | 1 小时 | 所有节点都失败后，等待多久再试 |
| **Pending 超时** | 3 分钟 | 资源在 Pending 状态的最大时间 |
| **基本健康检查** | 每 5 秒 | IsAlive 检查 |
| **深度健康检查** | 每 60 秒 | LooksAlive 检查 |

### 7.4 RHS 死锁检测与恢复

```
RHS 死锁恢复流程：
═══════════════════

1. RHS 调用资源 DLL 的入口点
        │
        ▼
2. RHS 等待 DeadlockTimeout (5 分钟) 等资源响应
        │ 超时
        ▼
3. 群集服务终止 RHS 进程以恢复
        │
        ▼
4. 群集服务等待 DeadlockTimeout × 4 (20 分钟) 
   等 RHS 进程终止
        │ 超时
        ▼
5. NetFT 触发蓝屏 STOP 0x9E 以恢复
   (因为 RHS 进程无法终止)
```

**2012 改进**：
- **Resource Re-attach**：健康资源重新附加到新 RHS，无需重启
- **核心资源隔离**：内置资源 → Core RHS，Physical Disk → Storage RHS，第三方 → 独立 RHS

### 7.5 资源组故障转移属性

| 属性 | 默认值 | 说明 |
|------|--------|------|
| **Preferred Owners** | 无（所有节点均可） | 设置后控制故障转移目标 |
| **故障转移次数** | 节点数 - 1 | 每个节点尝试一次 |
| **故障转移周期** | 6 小时 | 如果在此时间内超过故障转移次数，资源组保持 Failed |
| **自动回复 (Failback)** | 不自动回复 | 节点恢复后不自动迁回 |

## 8. 群集与 Active Directory

### 8.1 CNO 和 VCO 关系

```
Active Directory
├── CNO (Cluster Name Object)
│   └── 群集本身的计算机对象
│       例: CLUSTER01$
│       ├── 创建群集时自动创建
│       ├── 用于更新 VCO 密码
│       └── 必须有 SELF 完全控制权限
│
└── VCO (Virtual Computer Object)
    └── 群集角色/服务的计算机对象
        例: FILESERVER01$, SQLCLUSTER01$
        ├── 创建群集角色时自动创建
        ├── CNO 必须有权更新 VCO 密码
        └── 客户端通过 VCO 访问群集服务
```

### 8.2 Event 1207 — AD 权限问题

**现象**：群集无法更新计算机对象密码

**排查步骤**：
1. 阅读事件中的所有错误信息，注意 error code
2. CNO 必须有 **SELF 完全控制**
3. CNO 必须有 **VCO 的完全权限**（可以更新 VCO 密码）
4. 检查 DC 同步和网络连接
5. 参考：Event ID 1207 — Active Directory Permissions for Cluster Accounts

> ❓ **为什么资源仍然在线？** 密码更新失败不会立即影响资源，但长期未更新可能导致 Kerberos 认证失败。

## 9. 管理工具与操作

### 9.1 管理工具

| 工具 | 说明 |
|------|------|
| **Failover Cluster Manager** | GUI 管理界面 |
| **PowerShell** | 最推荐的命令行工具，功能最全 |
| **cluster.exe** | 旧版命令行工具（2012+ 默认不安装） |

### 9.2 关键 PowerShell 命令

```powershell
# ===== 群集管理 =====
New-Cluster -Name "Cluster01" -Node "Node1","Node2" -StaticAddress 10.0.0.10
Get-Cluster | Format-List *
Test-Cluster -Node "Node1","Node2"

# ===== 节点管理 =====
Get-ClusterNode
Suspend-ClusterNode -Name "Node1" -Drain   # 暂停节点（排空工作负载）
Resume-ClusterNode -Name "Node1"            # 恢复节点

# ===== 资源组管理 =====
Get-ClusterGroup
Move-ClusterGroup "MyFileServer" -Node "Node2"
Get-ClusterNode "Node3" | Get-ClusterGroup | Move-ClusterGroup  # 移走所有

# ===== 资源管理 =====
Get-ClusterResource
Get-ClusterGroup "GroupName" | Get-ClusterResource | Get-ClusterResourceDependency

# ===== 存储管理 =====
Get-ClusterGroup "Available Storage" | Get-ClusterResource
Add-ClusterSharedVolume -Name "Cluster Disk 2"

# ===== 日志收集 =====
Get-ClusterLog -Destination "C:\temp" -UseLocalTime
Set-ClusterLog -Size 300   # 设置日志大小 (MB)
Set-ClusterLog -Level 3    # 设置日志级别

# ===== 仲裁管理 =====
Get-ClusterQuorum
Set-ClusterQuorum -NodeAndDiskMajority "Cluster Disk 1"

# ===== 添加高可用角色 =====
Add-ClusterFileServerRole -Storage "Cluster Disk 3" -Name "FS01" -StaticAddress 10.0.0.31
```

### 9.3 补丁与滚动更新

**手动滚动更新步骤：**
```
1. 将所有资源移离要更新的节点
2. 在 Failover Cluster Manager 中暂停该节点
3. 应用补丁/热修复，按需重启
4. 取消暂停节点
5. 对其余节点重复步骤 1-4
```

> 💡 **最佳实践**：打补丁前先重启服务器，确保干净启动。这样可以避免已有问题被错误归咎于补丁。

### 9.4 Cluster Aware Updating (CAU)

CAU 是群集感知的自动更新功能：
- 自动排空工作负载（Live/Quick Migration）
- 将节点置于维护模式
- 下载安装补丁，按需重启
- 取消维护模式，迁回工作负载
- 逐节点重复直到所有节点更新完毕
- 支持 WU/MU 或 WSUS

```powershell
# 手动触发 CAU
Invoke-CauRun -ClusterName "Cluster01"

# 收集 CAU 日志
Save-CauDebugTrace -ClusterName "Cluster01"
```

## 10. 故障排查 (Troubleshooting)

### 10.1 诊断工具

| 工具 | 用途 | 位置/命令 |
|------|------|----------|
| **群集日志** | 最主要的排查工具 | `Get-ClusterLog -Destination C:\temp` |
| **事件日志** | 系统级事件 | Application and Services Logs → Microsoft → Windows → FailoverClustering |
| **群集验证** | 检查配置是否正确 | `Test-Cluster` |
| **SDP** | 自动收集诊断数据 | 替代了 MPSreport |
| **网络监视器** | 抓包分析 | Network Monitor / Wireshark |
| **性能计数器** | 性能监控 | Performance Monitor |

### 10.2 群集日志详解

```
日志位置: %systemroot%\Cluster\Reports
默认大小: 300 MB (WS2012)
默认级别: 3
时间格式: UTC (默认)，可用 -UseLocalTime 显示本地时间

# 收集日志
Get-ClusterLog -Destination C:\temp -UseLocalTime

# 调整日志大小和级别
Set-ClusterLog -Size 500
Set-ClusterLog -Level 5    # 更详细

# 日志格式示例：
# PID.TID::YYYY/MM/DD-HH:MM:SS.mmm LEVEL [Component] Message
00003750.00003150::2026/03/13-16:39:30.381 INFO [IM] got event: Remote endpoint 10.1.1.71:~3343~ unreachable
```

### 10.3 常见问题与排查

#### Event 1069 — 群集资源失败

**现象**：群集服务或应用中的资源失败

**排查**：
```powershell
# 1. 查看哪个资源失败
Get-ClusterResource | Where-Object {$_.State -eq "Failed"}

# 2. 查看资源详情
Get-ClusterResource "ResourceName" | Format-List *

# 3. 检查依赖关系
Get-ClusterResourceDependency "ResourceName"

# 4. 查看群集日志中该资源的详细错误
Get-ClusterLog -Destination C:\temp
# 搜索资源名称找到相关条目
```

#### Event 1135 — 节点被移除 / 心跳丢失

**这是最常见的群集问题之一！**

**首先判断**：这是初始错误，还是以下原因的结果？
- 节点重启
- 群集服务重启
- 服务器挂起

**如果是心跳丢失（最典型情况）**：

```
排查步骤（Action Plan）：
═══════════════════════

1. 确认 UDP 3343 是否被阻止
   └── 检查防火墙规则

2. 检查网络问题
   └── 网络丢包、延迟、断连

3. NIC 卸载设置（特别是虚拟化环境）
   └── 尝试禁用 RSS、VMQ、TCP Chimney

4. 如果使用 NIC Teaming
   └── 尝试拆分 Teaming 隔离问题

5. 安装最新群集补丁
   └── clussvc, tcpip, ndis 相关

6. 调整心跳阈值
   └── 增大 SameSubnetThreshold / CrossSubnetThreshold

7. 分析网络抓包
   └── 查看心跳包是否有丢失

群集日志示例：
[IM] got event: Remote endpoint 10.1.1.71:~3343~ unreachable from 10.1.1.72:~3343~
[IM] Marking Route from 10.1.1.72:~3343~ to 10.1.1.71:~3343~ as down
[NDP] All routes for route (virtual) local 169.254.2.213:~0~ to remote 169.254.1.57:~0~ are down
```

#### Event 1207 — 无法更新计算机账号密码

```
排查步骤：
1. 仔细阅读事件中的所有信息，注意 error code
2. CNO 必须有 SELF 完全控制
3. CNO 必须有 VCO 的完全权限
4. 注意 DC 同步和网络问题
```

#### Event 5120 — CSV 问题

```
排查步骤：
1. 注意事件日志中的 reason code
   例: STATUS_CLUSTER_CSV_AUTO_PAUSE_ERROR(c0130021)

2. 安装最新群集补丁

3. 检查 HBA、SAN 和存储问题
   └── 更新驱动和固件

4. 是否在备份运行时发生？

5. 检查存储连接性
```

#### Event 1146 — RHS 意外停止

```
排查步骤：
1. 群集日志 → 查看详细信息
2. RHS 进程 dump → 分析崩溃原因
3. 系统内存 dump → 深入分析（但会导致蓝屏）
   └── 蓝屏可能是可接受的，因为资源会故障转移

配置 dump 收集：
- 2008/2008R2: 配置 WER user dump 和 OS dump
- 2012/2012R2: 可通过注册表配置
```

#### Bugcheck 0x9E

这是 NetFT 触发的蓝屏，通常因为 RHS 死锁无法恢复：

```
参考: Decoding Bugcheck 0x0000009E
https://blogs.msdn.com/b/clustering/archive/2013/11/13/10467483.aspx
```

### 10.4 存储问题通用排查

```
存储排查决策树：
═══════════════

1. 什么是底层存储架构？
   └── iSCSI? SAN (FC/SAS)? 虚拟化? 跨地域?

2. 问题是 OS 内部还是外部导致？
   └── 方法：禁用群集磁盘驱动来隔离

3. 磁盘签名是否改变？

4. 运行群集验证 (Test-Cluster)

5. 联系存储厂商
   └── 使用厂商工具检查磁盘状态 (如 PowerPath)

6. 跨地域群集特别注意
   └── 某些症状可能是预期行为
```

## 11. 多站点群集 (Multi-Site Clusters)

### 关键考虑

| 维度 | 注意事项 |
|------|---------|
| **网络** | 站点间延迟、带宽、路由 |
| **仲裁** | 使用文件共享见证放在第三个站点；使用节点权重控制故障转移行为 |
| **存储** | 需要存储级别的虚拟化/复制 |
| **心跳** | 增大 CrossSubnetThreshold 和 CrossSubnetDelay |
| **故障转移** | 手动 vs 自动，使用 /FQ 和 /PQ 控制 |

---
---

# English Version

## 1. Overview — What is Failover Clustering

**Failover Clustering** is a group of independent servers working together as a single system, providing **High Availability (HA)** for mission-critical applications.

### What a Cluster Provides

| Capability | Description |
|-----------|------------|
| **High Availability** | Eliminates single points of failure |
| **Failover** | Automatically migrates services to healthy nodes on failure |
| **Load Balancing** | Distributes workloads in Active/Active configurations |
| **Zero Downtime Maintenance** | Rolling updates without service interruption |

### Failover Clustering vs NLB

| Dimension | Failover Clustering | Network Load Balancing (NLB) |
|-----------|-------------------|----------------------------|
| **Purpose** | Application-level HA | Network traffic load balancing |
| **Shared Storage** | Required (traditional) | Not required |
| **Use Cases** | Databases, File Servers, Exchange | Web Servers, IIS |
| **Max Nodes** | 64 (2012+) | 32 |

## 2. Core Terminology

| Term | Description |
|------|------------|
| **Cluster** | Group of independent servers working as one system |
| **Node** | Each server in the cluster |
| **Resource** | Smallest unit managed by the cluster (IP, disk, network name, service) |
| **Resource Group** | Collection of related resources that failover as a unit |
| **Failover** | Automatic migration of resources from failed to healthy node |
| **Failback** | Migration of resources back to original node after recovery |
| **CNO** | Cluster Name Object — the cluster's AD computer account |
| **VCO** | Virtual Computer Object — each cluster role's AD computer account |
| **Witness** | Extra voting resource (disk or file share) for quorum |
| **RHS** | Resource Host Subsystem — hosts and monitors resource DLLs |
| **NetFT** | Cluster virtual network adapter providing fault-tolerant communications |

## 3. Cluster Architecture

### Three-Tier Architecture

```
┌────────────────────────────────────────────┐
│   Top Tier: Cluster Abstractions            │
│   Nodes, Groups, Resources, Policies        │
├────────────────────────────────────────────┤
│   Middle Tier: Cluster Operation            │
│   Membership, Regroup, GUM, Consistency     │
├────────────────────────────────────────────┤
│   Bottom Tier: OS Interaction               │
│   PartMgr, ClusDisk, NetFT, NTFS, Security │
└────────────────────────────────────────────┘
```

### Key Components

| Component | Role |
|-----------|------|
| **Messaging** | All intra-cluster communication (Unicast + GEM Multicast) |
| **Membership Manager** | Regroup algorithm, Gossip protocol, node join/failure |
| **Global Update Manager (GUM)** | Serialized atomic updates, Locker node model |
| **Resource Control Manager** | Failover policies, resource dependency trees, state management |
| **Database Manager** | Fault-tolerant registry-based database, Paxos consensus |
| **Quorum Manager** | Determines if cluster has quorum, tracks all replicas |
| **Host Manager** | TCP 3343 connections, security handshake, NetFT routing |
| **Topology Manager** | Network topology discovery and maintenance |
| **Security Manager** | SSPI handshake, message signing/encryption |
| **NetFT Driver** | Fault-tolerant multi-path communication between nodes |

### Regroup Process

```
Opening → Closing → Pruning → PruneAck → GemRepair → Cleanup → Stable
```
This multi-stage algorithm ensures all nodes reach consensus on cluster membership after any topology change.

## 4. Quorum — The Brain of the Cluster

### Quorum Models

| Model | Votes | Best For |
|-------|-------|----------|
| **Node Majority** | Nodes only | Odd number of nodes |
| **Node + Disk Majority** | Nodes + witness disk | Even nodes + shared disk |
| **Node + File Share Majority** | Nodes + file share witness | Multi-site clusters |
| **No Majority (Disk Only)** | Disk only | Not recommended |

### Dynamic Quorum (2012+)

- Continuously adjusts vote weights based on active membership
- Allows cluster to survive with >50% nodes down
- **"Last Man Standing"** — can theoretically run on a single node
- Enabled by default: `(Get-Cluster).DynamicQuorum = 1`

### Force Quorum and Prevent Quorum

| Option | Use Case |
|--------|----------|
| `/FQ` (Force Quorum) | Force cluster start when insufficient votes (DR scenario) |
| `/PQ` (Prevent Quorum) | Start service but only allow joining existing cluster |

## 5. Cluster Networking

### Heartbeat Mechanism

- **Protocol**: UDP port 3343
- **Controlled by**: NetFT driver
- Node join uses TCP; runtime heartbeat uses UDP

| Parameter | Default | Description |
|-----------|---------|------------|
| SameSubnetDelay | 1000ms | Same-subnet heartbeat interval |
| CrossSubnetDelay | 1000ms | Cross-subnet heartbeat interval |
| SameSubnetThreshold | 10 | Missed heartbeats before marking unreachable |
| CrossSubnetThreshold | 20 | Cross-subnet missed heartbeat threshold |

### Network Roles

| Role | Value | Description |
|------|-------|------------|
| **Private** | 1 | Internal cluster communication only, carries CSV/Live Migration traffic |
| **Public** | 3 | Client access + internal cluster communication |
| **Not Used** | 0 | Cluster ignores this network, no health monitoring |

## 6. Cluster Storage

### Requirements
- Shared by at least 2 nodes
- SCSI-3 Persistent Reservations required
- Supported: FC, SAS, iSCSI, S2D, Shared VHDX

### Cluster Shared Volumes (CSV)
- **Concurrent access** from all cluster nodes
- **Metadata** routed to coordinator (owner) node
- **Data I/O** written directly to disk (high performance)
- Mount point: `C:\ClusterStorage\`
- Based on NTFS

### Disk Fencing
- Controls disk access (online/offline per node)
- Handled by PartMgr.sys via `DISK_ONLINE` / `DISK_OFFLINE` IOCTLs
- Prevents data corruption from simultaneous uncoordinated access

## 7. Resource Management

### Resource States
Online → Failed → Offline (with Online Pending / Offline Pending transitions)

### Default Resource Policies

| Policy | Default |
|--------|---------|
| Restart count | 1 time |
| Restart window | 15 minutes |
| Failover action | Move entire group |
| Retry interval | 1 hour |
| Pending timeout | 3 minutes |
| Basic health check | Every 5 seconds |
| Thorough health check | Every 60 seconds |

### RHS Deadlock Recovery

1. RHS waits **5 minutes** for resource response
2. Cluster service terminates RHS process
3. Waits **20 minutes** for RHS to terminate
4. If RHS won't terminate → NetFT triggers **STOP 0x9E** (bugcheck)

## 8. Troubleshooting

### Key Diagnostic Tools

| Tool | Command |
|------|---------|
| Cluster Log | `Get-ClusterLog -Destination C:\temp -UseLocalTime` |
| Validation | `Test-Cluster -Node Node1,Node2` |
| Event Logs | FailoverClustering channels in Event Viewer |
| Log Size/Level | `Set-ClusterLog -Size 500`, `Set-ClusterLog -Level 5` |

### Common Issues

#### Event 1135 — Node Removed (Heartbeat Loss)

**Action Plan:**
1. Check if UDP 3343 is blocked (firewall)
2. Check network issues (packet loss, latency)
3. Review NIC offload settings (disable RSS, VMQ in virtual environments)
4. Break NIC teaming to isolate
5. Install latest cluster patches (clussvc, tcpip, ndis)
6. Tune heartbeat thresholds (increase SameSubnetThreshold/CrossSubnetThreshold)
7. Analyze network captures

#### Event 1069 — Resource Failure
- Identify which resource failed
- Check dependencies
- Review cluster log for detailed error

#### Event 1207 — AD Permission Issues
- CNO needs SELF full control
- CNO needs full permission on VCO
- Check DC sync and network

#### Event 5120 — CSV Issues
- Note the reason code in event log
- Install latest patches
- Check HBA/SAN/storage
- Check if backup was running

#### Event 1146 — RHS Crash
- Cluster log for details
- RHS process dump for crash analysis
- System memory dump for thorough analysis (causes bugcheck)

#### Bugcheck 0x9E
- Caused by NetFT when RHS deadlock cannot be recovered
- RHS process failed to terminate within 20 minutes

### Storage Troubleshooting Checklist
1. Identify storage infrastructure (iSCSI? SAN? Virtual? Geo?)
2. Isolate: OS internal vs external (disable cluster disk driver to test)
3. Check disk signature changes
4. Run cluster validation (`Test-Cluster`)
5. Involve storage vendor
6. Special care for geo-clusters (some symptoms may be expected)

## 9. References

- Windows Server Failover Clustering training materials (M01-M15, S01-S09)
- [Failover Clustering Overview](https://learn.microsoft.com/en-us/windows-server/failover-clustering/failover-clustering-overview)
