---
layout: post
title: "Deep Dive: Windows SMB Protocol — Architecture, Operations & Troubleshooting"
date: 2026-03-12
categories: [Knowledge, Networking]
tags: [smb, smb2, smb3, file-sharing, cifs, sofs, multichannel, troubleshooting, windows-server, file-server]
type: "deep-dive"
---

# Deep Dive: Windows SMB Protocol — Architecture, Operations & Troubleshooting

**Topic:** Windows Server Message Block (SMB) Protocol  
**Category:** Networking / File Services  
**Level:** 中级 / 高级 (Intermediate / Advanced)  
**Last Updated:** 2026-03-12

---

# 中文版

## 1. 概述 (Overview)

SMB（Server Message Block）是 Microsoft 设计的网络文件共享协议，允许应用程序和用户通过网络读取/写入文件、请求服务。它是 Windows 文件共享、打印服务和许多 RPC 通信的基础协议。

SMB 在企业网络中的核心地位：
- **文件共享**：通过 UNC 路径（如 `\\server\share`）访问远程文件
- **Active Directory**：域加入、组策略、登录脚本都依赖 SMB
- **Hyper-V over SMB**：虚拟机存储可以直接放在 SMB 共享上
- **SQL Server over SMB**：数据库文件可以存储在 SMB 共享上
- **RPC over Named Pipes**：许多管理工具通过 SMB 的 IPC$ 进行 RPC 通信

SMB 协议经历了多个版本的演进：

| 版本 | 操作系统 | 关键特性 |
|------|---------|---------|
| CIFS | Windows NT 4.0 (1996) | 古老版本 |
| SMB 1.0 | Windows 2000/XP/2003 | 已弃用，安全风险 |
| SMB 2.0 | Vista SP1 / Server 2008 | 全新设计，性能大幅提升 |
| SMB 2.1 | Windows 7 / Server 2008 R2 | Leasing、Large MTU |
| SMB 3.0 | Windows 8 / Server 2012 | 加密、SMB Direct、CA |
| SMB 3.02 | Windows 8.1 / Server 2012 R2 | 改进 |
| SMB 3.1.1 | Windows 10 / Server 2016+ | Pre-auth、加密改进 |

> **重要原则**：永远不要通过禁用 SMB2/3 来排查 SMB 问题。除非有业务关键需求，应始终建议客户停止使用 SMBv1。

## 2. 核心概念 (Core Concepts)

### 2.1 SMB Framework — 协议结构

SMB 通信遵循严格的层级框架，理解这个框架是分析网络抓包和 ETL 日志的基础：

```
1. SMB Dialect Negotiation         ← 协商协议版本
2. Authentication (Kerberos/NTLM)  ← 认证
3. SMB Session Setup               ← 建立会话 (Session ID)
4. SMB Tree Connect                ← 连接共享 (Tree ID)
5. SMB Create / File Operations    ← 文件操作 (File ID)
6. SMB Tree Disconnect             ← 断开共享
7. SMB Logoff                      ← 注销会话
```

**IPC$ 的角色**：IPC$（Interprocess Communications）是服务器服务创建的特殊共享，允许连接到 Named Pipes。打开 Named Pipe 后，可以进行 RPC over SMB 通信。

### 2.2 SMB 标识符 (Identifiers)

| 标识符 | 生成方 | 用途 |
|--------|--------|------|
| **PID** | Redirector | 唯一标识工作站上的特定应用进程 |
| **Session ID** | Server | 唯一标识用户认证后的特定会话 |
| **TID (Tree ID)** | Server | 唯一标识到服务器共享的 "tree connection" |
| **MID (Message ID)** | Redirector | 唯一标识特定的 SMB 事务 |
| **FID (File ID)** | Server | 唯一标识特定的服务器端打开文件实例 |

> SMB 2.0 的 File ID 比 SMB 1.0 更大（16 字节），包含 **Persistent** 和 **Volatile** 两部分。Persistent 部分在 Resilient/Durable Handle 场景下有效。

### 2.3 SMB2 命令类型

**协议协商、认证和共享访问：**
`NEGOTIATE`, `SESSION_SETUP`, `LOGOFF`, `TREE_CONNECT`, `TREE_DISCONNECT`

**文件、目录和卷访问：**
`CANCEL`, `CHANGE_NOTIFY`, `CLOSE`, `CREATE`, `FLUSH`, `IOCTL`, `LOCK`, `QUERY_DIRECTORY`, `QUERY_INFO`, `READ`, `SET_INFO`, `WRITE`

**其他：**
`ECHO`, `OPLOCK_BREAK`

### 2.4 SMB 认证

**Kerberos 认证（首选）：**
1. 客户端从 DC 获取 Kerberos Ticket（如果缓存中没有）
2. 在 Session Setup 中将 Ticket 发送给 SMB Server
3. Wireshark 过滤器：`kerberos`

**NTLM 认证（回退）：**
- 使用 IP 地址访问服务器时
- 跨林访问且为 legacy NTLM trust 时
- 服务器不属于域时
- 防火墙阻止了 Kerberos 端口（TCP 88）时
- Wireshark 过滤器：`ntlmssp`

### 2.5 文件权限与共享权限

**文件权限（NTFS/ReFS Permissions）**：控制文件和文件夹在存储卷上的访问。

**共享权限（Share Permissions）**：控制通过网络对共享的访问。

**有效权限（Effective Permissions）**：
- 文件权限和共享权限**取最严格**的那个
- 用户必须**同时拥有**文件权限和共享权限，否则被拒绝

**权限冲突优先级：**
1. 显式分配的 Deny
2. 显式分配的 Allow
3. 继承的 Deny
4. 继承的 Allow

> **注意**：文件权限直接由 Directory Services Team 支持，但了解其工作方式对排查文件共享问题至关重要。

## 3. 工作原理 (How It Works)

### 3.1 SMB Server 内核栈架构

```
┌──────────────────────────────────────────────┐
│              User-Mode                        │
│   LanmanServer (srvsvc.dll / sscore.dll)     │
│          (SMB Server Service)                 │
├──────────────────────────────────────────────┤
│              Kernel-Mode                      │
│                                               │
│   ┌─────────┐  ┌─────────┐  ┌──────────────┐│
│   │ SRV.sys │  │ SRV2.sys│  │  SRVNET.sys  ││
│   │ (SMB1)  │  │ (SMB2)  │  │ (Net Infra)  ││
│   └─────────┘  └─────────┘  └──────────────┘│
│                    ↓                          │
│   ┌──────────────────────────────────────────┐│
│   │     IO Manager / Cache Manager            ││
│   ├──────────────────────────────────────────┤│
│   │     Filter Manager Plug-in                ││
│   │  (Third-party filters: AV, etc.)          ││
│   ├──────────────────────────────────────────┤│
│   │     NTFS.sys / ReFS.sys                   ││
│   │       Local File System                   ││
│   ├──────────────────────────────────────────┤│
│   │     Storage / Volume                      ││
│   └──────────────────────────────────────────┘│
│                                               │
│   ┌──────────────────────────────────────────┐│
│   │  AFD.sys → TCPIP.sys (Port 445)          ││
│   │  NetBT.sys → TDX.sys (Port 139)         ││
│   │  NDIS.sys ← Incoming Network DPCs       ││
│   └──────────────────────────────────────────┘│
└──────────────────────────────────────────────┘
```

**端口说明：**
- **Port 445**：SMB Direct over TCP（推荐，现代默认）
- **Port 139**：SMB over NetBIOS over TCP（旧版兼容）

### 3.2 SMB Client 内核栈架构

```
┌──────────────────────────────────────────────┐
│  User-Mode: WebDAV user-mode, Applications   │
├──────────────────────────────────────────────┤
│  Kernel-Mode                                  │
│                                               │
│  ┌───────────────────────────────────┐       │
│  │  Filter Manager (fltmgr.sys)      │       │
│  │  + Third-party mini-filters       │       │
│  └──────────────┬────────────────────┘       │
│                 ↓                              │
│  ┌───────────────────────────────────┐       │
│  │  MUP.sys (Multiple UNC Provider)  │       │
│  │  Owns \\* (UNC namespace)         │       │
│  └──────────────┬────────────────────┘       │
│         ┌───────┴────────┐                    │
│  Surrogates:             UNC Providers:       │
│  ┌──────────┐           ┌──────────────────┐ │
│  │ DFSC.sys │           │   RDBSS.sys      │ │
│  │(DFS Cli) │           │ (Major Redirector)│ │
│  ├──────────┤           └────────┬─────────┘ │
│  │ CSC.sys  │              ┌─────┴──────┐    │
│  │(Offline) │              │ MRXSMB.sys │    │
│  └──────────┘              │(Master RDR) │    │
│                            ├────────────┤    │
│                     ┌──────┴──────┐          │
│                     │mrxsmb10.sys │          │
│                     │  (SMB1)     │          │
│                     ├─────────────┤          │
│                     │mrxsmb20.sys │          │
│                     │  (SMB2/3)   │          │
│                     └─────────────┘          │
│  Other Mini-Redirectors:                     │
│  mrxdav.sys (WebDAV), nfsrdr.sys (NFS),     │
│  rdpdr.sys (RDP)                             │
│                                               │
│  Networking: AFD.sys (WSK) → TCPIP.sys       │
└──────────────────────────────────────────────┘
```

### 3.3 关键内核组件详解

#### MUP.sys — Multiple UNC Provider
- **拥有** Remote File System 的 `\\*` 对象（UNC 命名空间）
- 所有以 `\\` 开头的调用都由 Object Manager 路由到 MUP
- Filter Manager 之后**第一个**接收 UNC IO 的驱动
- 维护状态机来协调 Surrogate Provider 和 Redirector 处理 IO
- 典型调用起始点：`mup!MupCreate` 或 `mup!MupFsdIrpPassThrough`

#### RDBSS.sys — Redirected Drive Buffering Subsystem
- 拥有符号链接 `\\??\\fsWrap`，是 "Major-Redirector"
- 为 Mini-Redirector 提供插件接口（所以 redirector 被称为 "mini"）
- 与 Cache Manager 交互实现读写缓存
- 维护所有当前活动的 UNC IO 列表
- Win7+ 使用 WSK（Winsock Kernel）通过 AFD.sys 与网络通信（取代旧的 TDI）

#### MRXSMB.sys — Master Redirector
- 拥有目标为 Port 445 和 139 的 UNC IO
- 由 mrxsmb.sys + mrxsmb10.sys (SMB1) + mrxsmb20.sys (SMB2/3) 组成
- 包含四大引擎：
  - **Connection Engine**：管理连接
  - **Transport Engine**：管理传输层
  - **CSQ Engine**：Compound Sequence Engine，打包 SMB2 请求/响应
  - **Exchange Engine**：将 RDBSS 的 RX_CONTEXT 映射为与 Server 之间的 "exchange"

#### Surrogate Providers
- **CSC.sys**（Offline Files）：CACHING surrogate，目标离线（网络问题或慢链接）时启用。**已弃用**，推荐 Work Folders / OneDrive
- **DFSC.sys**（DFS Client）：NAMESPACE surrogate，将 DFS 逻辑路径转换为实际物理目标
- 两者在 IO 传递给 UNC Provider **之前**（PRE）和**之后**（POST）都会被调用

#### Filter Manager
- 为第三方文件系统 Mini Filter（如杀毒软件）提供 PRE/POST 回调插件
- Filter 的**高度（Altitude）**决定了调用顺序
- 通常不会导致出站远程文件系统 IO 的问题
- 最坏情况：可能在实际 UNC IO 被服务之前，为相关文件触发额外 IO 造成多余流量
- **Procmon 本身就是一个 File System Mini Filter**——如果有其他第三方 filter 在其上方，procmon 的调用栈不会显示这些 filter

### 3.4 SMB 连接引擎对象

SMB Redirector 维护一组与连接资源相关的结构：

| 对象 | 用途 |
|------|------|
| **SMBCEDB_SERVER_ENTRY** | 每个服务器连接分配一个 Server Entry |
| **SMBCEDB_NET_ROOT_ENTRY** | 跟踪服务器上某个共享的信息 |
| **SMBCE_V_NET_ROOT_CONTEXT** | 特定用户到某个共享的连接信息 |
| **SMBCEDB_SESSION_ENTRY** | 特定用户会话的信息中心 |

**CE 对象状态：**
- `ACTIVE`：对象当前活跃，在服务器上有关联上下文
- `DISCONNECTED`：临时断开，服务器上的关联状态已丢失
- `CONSTRUCTION_IN_PROGRESS` / `DESTRUCTION_IN_PROGRESS`：从断开到活跃的过渡状态
- `RECOVERY_IN_PROGRESS`：临时断开但服务器上的状态仍存在，正在重新验证
- `INVALID`：已失效，只能转到 DELETED 状态
- `DELETED`：已删除

### 3.5 SMB2 协议协商

**向后兼容机制：**
1. SMB 2.0 客户端首先发送 SMB 1.0 格式的 Negotiate Protocol 请求
2. 请求中包含新的 dialect string `"SMB 2.001"` 表示支持 SMB 2.0
3. 如果服务器支持 SMB 2.0，会发送 SMB 2.0 格式的响应（Protocol ID = `0xFE 'S' 'M' 'B'`）

**协商响应关键字段：**
- `ServerGuid`：服务器全局唯一标识（不用于安全识别）
- `Capabilities`：协议能力（如 DFS 支持）
- `MaxTransactSize` / `MaxReadSize` / `MaxWriteSize`：IO 大小限制
- `SecurityBuffer`：认证安全 blob（RFC 2478 格式）

### 3.6 Command Compounding（命令复合）

SMB 2.0 引入了全新的复合机制，每个命令都有自己的 SMB Header：

**Related Operations**：后续命令的 `SMB2_FLAGS_RELATED_OPERATIONS` 标志被设置，服务器将它们作为关联请求处理（如 Create → Read，Read 使用 Create 返回的 File ID）。所有响应必须复合在一起返回。

**Unrelated Operations**：标志未设置，服务器独立处理每个请求，可以分别发送响应，且不需要按顺序。

> 这消除了 SMB 1.0 `&X` 命令链的许多限制，允许在单个 SMB 中复合多个不直接关联的请求来提高性能。

### 3.7 Pipelined Reads & Writes

SMB 2.0 消除了 SMB 1.0 的 64K-1 读写限制。Redirector 使用**管道化读写**来传输大块数据：根据 Credit 可用性，同时向服务器排队多个顺序读写请求来饱和连接。

> **最佳实践**：应用应请求 1MB-4MB 的大读写，并异步排队多个请求以保持管道饱和。

## 4. 高级特性

### 4.1 Scale-Out File Server (SOFS)

SOFS 是 Windows Server 2012 引入的集群文件服务器类型，为应用数据设计：

**对比传统集群文件服务器：**

| 特性 | 传统集群文件服务器 | SOFS |
|------|-------------------|------|
| 模式 | Active-Passive | Active-Active |
| 共享类型 | 标准文件共享 | CSV（Cluster Shared Volumes） |
| 客户端要求 | SMB 1.0+ | SMB 3.0+ |
| 推荐场景 | IW 文件共享、VMM Library | Hyper-V、SQL Server over SMB |
| IW 功能 | DFS-N、DFS-R、FSRM 等 | **不兼容**大多数 IW 功能 |
| 离线文件 | 支持 | **不兼容** |

**SOFS 关键组件：**
- **Distributed Network Name (DNN)**：为 SOFS 提供 DNS 名称，注册所有节点的 IP（不使用虚拟 IP），通过 DNS Round Robin 分发客户端
- **Scale Out File Server Resource**：在每个节点上在线共享，通过 clone resources 确保所有节点一致
- **CSV 缓存**：分布式缓存保证跨集群一致性，Read-Only cache for un-buffered IO

### 4.2 Continuously Available (CA) 共享

CA 共享是 SOFS 实现零停机的关键，通过 **Persistent Handle** 和 **Resume Key Filter (RKF)** 实现：

**Persistent Handle：**
- 文件句柄状态在客户端断开期间被保留
- 存储在持久化数据库中（非持久句柄存储在易失性数据库）
- 在 Durable Handle v2 Create Context 中使用 `SMB2_DHANDLE_FLAG_PERSISTENT` 标志
- 创建响应包含 timeout 值（毫秒），服务器在此时间内等待客户端重连

**Resume Key Filter (RKF)：**
- 保护句柄状态信息，阻止其他客户端访问
- 在计划/非计划故障转移后恢复句柄状态
- 仅为具有 Continuous Availability 上下文的句柄持久化状态
- 安装在 Failover Clustering 功能中，位于文件服务器文件系统栈上

> **注意**：CA 共享使所有 IO 以同步/write-through 模式执行，**不建议用于大量小型、频繁变化的文件**。

### 4.3 SMB Witness Protocol

SMB Witness 使客户端不需要等待 TCP 超时（通常约 20 秒）即可从非计划故障中恢复：

**注册流程：**
1. SMB 客户端连接到 Node A，通知 Witness 客户端
2. Witness 客户端从 Node A 获取集群节点列表
3. **排除**数据节点（Node A），选择不同的 Witness 服务器（Node B）
4. 向 Node B 注册事件通知

**通知流程（非计划故障）：**
1. Node A 故障
2. 集群基础设施通知 Node B 上的 Witness 服务器
3. Node B 的 Witness 通知客户端 Node A 已下线
4. 客户端断开 Node A 连接，开始重连到其他节点
5. Witness 客户端尝试选择新的 Witness 服务器

**客户端重定向（对称存储）：**
```powershell
# 管理员可手动重定向 SMB 客户端到不同节点
Move-SmbWitnessClient -ClientName app-server01 -DestinationNode NodeC
```

### 4.4 SMB Multichannel

SMB Multichannel 在客户端和服务器之间自动建立多条 TCP 连接：
- 利用多个网络适配器实现**聚合带宽**
- 一个 NIC 故障时自动转移到另一个，提供**网络容错**
- 无需手动配置，禁用一个 NIC 时文件传输继续

### 4.5 SMB Encryption

SMB Encryption（SMB 3.0+）增强 SMB 通信的安全性：

```powershell
# 为单个共享启用加密
Set-SmbShare -Name "SecureData" -EncryptData $true

# 为整个文件服务器启用加密
Set-SmbServerConfiguration -EncryptData $true
```

> **性能注意**：加密使用文件服务器 CPU，大量共享和用户访问时可能导致**高 CPU**。建议仅为少量重要文件启用，不要加密整个服务器。

### 4.6 SMB over QUIC

SMB over QUIC（Server 2022 Azure Edition+）使用 HTTP/3 和 TLS 1.3，提供快速、安全的文件传输：

```powershell
# 通过 QUIC 传输访问
net use * \\fileserver.contoso.com\Sales /transport:quic
```

要求：Windows Server 2022 Azure Datacenter Edition + 证书 + Windows Admin Center 配置。

### 4.7 其他 SMB 特性

**Access-Based Enumeration (ABE)**：根据权限控制共享文件夹的可见性。
- Windows Server 2003 R2 引入
- 没有访问权限的用户看不到文件/文件夹
- 每次需要检查权限，大量共享+用户时可能导致**高 CPU**
- **Domain Admin 不受 ABE 限制**，且会覆盖 NTFS Full Deny 权限

**BranchCache**：帮助分支机构缓存文件服务器内容，减少 WAN 带宽使用。

**CSC (Client-Side Caching / Offline Files)**：允许离线访问网络文件。
- CSC 数据存储在 `C:\Windows\CSC\v2.0.6\namespace\`
- 绕过 CSC 进行排查：使用 `\\server$nocsc$\share`
- **已弃用**，推荐 Work Folders 或 OneDrive

**UNC Hardening**：增强 UNC 路径的安全性。
- `RequireMutualAuthentication`
- `RequireIntegrity`
- `RequirePrivacy`

## 5. 常见问题与排查 (Common Issues & Troubleshooting)

### 5.1 无法访问文件共享

**症状**：`\\server\share` 无法访问

**排查步骤：**
1. **TCP 连通性**：`Test-NetConnection server -Port 445`
2. **服务状态**：检查 Server Service 是否运行
   - 抓包中如果 TCP SYN→SYN/ACK→ACK 后收到 RST，可能是 Server Service 未运行
3. **SMB 协议协商**：检查 Negotiate 是否成功
4. **认证**：Session Setup 是否返回错误
5. **共享访问**：Tree Connect 是否成功
6. **名称解析**：DNS 是否正确解析服务器名

**当 NTLM 被意外使用时**：
- 使用 IP 地址访问会强制 NTLM
- 跨林 legacy trust 场景
- Kerberos 端口（TCP 88）被防火墙阻止

### 5.2 Kerberos 认证失败 (KRB5KRB_AP_ERR_MODIFIED)

**症状**：无法通过 Kerberos 访问网络共享，Session Setup 失败

**常见原因**：SMB Server 的机器账户密码与 AD 中不同步（例如：机器账户被重置）

**排查步骤：**
1. 抓 Wireshark，过滤 `kerberos`
2. 检查 SPN（SName String）是否正确
3. 比较 Server 本地 LSA Secret 中的密码时间戳与 AD 中 `pwdLastSet` 是否一致
4. 在 Server 上使用 ETL 抓取 auth 日志，检查是否 "unable to decrypt the ticket"

**解决方案：**
```powershell
# 以本地管理员登录 SMB Server
Reset-ComputerMachinePassword -Server "dc01"
```

### 5.3 文件复制慢

**排查关键原则：**
- **必须有对比基准**：需要明确期望速度和实际速度
- **不能用 iPerf 与 SMB 比较**：iPerf 测试网络吞吐量，不考虑存储性能
- **不能用 NFS 与 SMB 比较**：完全不同的协议
- **唯一有效的方法**：在两个场景下收集日志进行对比

**网络层排查：**
1. 检查网络抓包中的 **TCP 问题**：
   - 丢包（重传、快速重传、重复 ACK）
   - 高延迟
   - TCP Window Scaling 问题
   - 数据包篡改
2. 检查 **SMB 错误**：
   - Wireshark: `smb2.error.data`
   - Netmon: `smb2.ErrorMessage`
3. 检查 TCP 全局设置和 MTU 大小

**存储层排查（Performance Monitor 计数器）：**

| 计数器 | 正常值 | 异常 |
|--------|--------|------|
| `\PhysicalDisk\Avg. Disk sec/Transfer` | < 10ms | > 50ms = 严重 IO 瓶颈 |
| `\PhysicalDisk\Avg. Disk Queue Length` | < 2x spindles | > 2x = 瓶颈可能 |
| `\Server Work Queues\Available Work Items` | > 0 | 持续 = 0 可能是 nonpaged pool 不足 |
| `\Server Work Queues\Queue Length` | = 0 | > 0 = 请求积压 |
| `\Server Work Queues\Active Threads` | ≤ 10 (默认最大) | 持续满载 |

**File Explorer 慢但命令行正常**：如果 `dir` 命令正常但 Explorer 慢，可能是 Explorer UI 问题，需要 engage OS Team。

### 5.4 SMB Slowness 分析方法论

**Scoping 问题清单：**
1. 访问路径是什么？IP / FQDN / NetBIOS / DFSN / Cluster path？
2. 客户端和服务器的 OS 版本？
3. 问题影响范围？特定用户/客户端还是所有人？是否有规律？
4. 如果是 DFSN 路径：Namespace 返回慢还是实际文件夹目标慢？
5. 如果是 Cluster 路径：文件存储在哪个磁盘？收集性能日志时在 Active Owner 节点上收集

**Office 文件打开慢**：
- 先测试 Notepad (TXT) 文件是否也慢
- 如果只有 Office 文件慢，需要 engage Office Team（但网络仍需排查）
- Office 文件的 IO 比 Notepad 文件复杂得多
- 建议客户截图或录制视频显示最长的 Hang 画面

### 5.5 SOFS Scale-Out Rebalancing 不工作

**症状**：CSV 故障转移后，SMB 连接没有切换到新的 CSV Owner 节点

**关键结论：**
- 如果之前的 Owner 节点**仍可访问**，现有 SMB 连接**不会**切换到新 CSV Owner。只有**新连接**会去往 CSV Owner 节点
- 仅当之前的 Owner 节点**不可访问**时，现有 SMB 连接才会切换
- 设置 `AsymmetryMode` 注册表值不会改变此行为：
```
HKLM\System\CurrentControlSet\Services\LanmanServer\Parameters
    AsymmetryMode (DWORD) = 2
```

### 5.6 集群文件共享排查

1. 确认 Server Service 能处理非集群共享请求
2. 从集群角度检查底层资源（IP、Name）是否正常
3. 收集：网络抓包 + Process Explorer + T.cmd Trace (srv.etl)

## 6. 诊断工具与方法 (Diagnostic Tools)

### 6.1 T.cmd — SMB ETW Tracing

T.cmd 是收集 SMB Redirector 和相关组件 ETW Trace 的核心工具：

**客户端抓取：**
```cmd
cd c:\logs
t.cmd clion          ← 开始
:: 复现问题
t.cmd clioff         ← 停止
```

**服务器端抓取：**
```cmd
cd c:\logs
t.cmd srvon          ← 开始
:: 复现问题
t.cmd srvoff         ← 停止
```

> **注意**：T10.cmd 用于 Win10/Server 2016 及更高版本。

**T.cmd 产生的日志文件：**

| ETL 文件 | 包含的组件 |
|---------|-----------|
| `fskm.etl` | csckm, dav, dfsc, mup, rdbss, mrxsmb, smb20, witnesscli |
| `fsum.etl` | cscnet, cscdll, cscapi, cscui, cscum, WebClnt |
| `dns.etl` | dnsapi, dns |
| `sec.etl` | kerberos, ntlm, negoexts, pku2u |
| `tcp.etl` | tcp |
| `nbt.etl` | nbtsmb |

**T.cmd 日志格式解读：**
```
[0] 03CC.0918::03/31/2013-20:48:54.801 [mup] fileio_c507 MupCreate() - FileName \\server\IPC$ ...
 ^    ^              ^                   ^      ^            ^                ^
 1    2              3                   4      5            6                7
```
1. Processor# | 2. PID.TID | 3. 时间戳 | 4. 组件名 | 5. 源文件_行号 | 6. 函数名 | 7. 跟踪消息

### 6.2 T.cmd 分析方法

1. **从 MupCreate() 开始**
2. **过滤 Error**，查看第一个报错位置
3. 使用 `MRxSmb2AssembleSmb` 和 `SmbCeReceiveInd` 匹配网络抓包：
   - `MRxSmb2AssembleSmb` → 发送数据包
   - `SmbCeReceiveInd` → 接收数据包
4. 通过 **MID (Message ID)**、**SID (Session ID)**、**TID (Tree ID)** 进行匹配

### 6.3 RDR Autologger Tracing

特别适合调试崩溃问题的持久化 trace：

```
Registry: HKLM\SYSTEM\CurrentControlSet\Control\WMI\Autologger\RdrLog
```

- 配置 `MinimumBuffers` / `MaximumBuffers` 设置缓冲区大小
- 使用 Ring Buffer 模式，消耗 Non-Paged Pool
- 获取 crash dump 后，使用 `!wmitrace.strdup` / `!wmitrace.logsave` 从内存提取 trace

### 6.4 Performance Monitor

```powershell
Logman.exe create counter Perf-1Second -f bincirc -max 512 -c `
    "\LogicalDisk(*)\*" "\Memory\*" "\Network Interface(*)\*" `
    "\PhysicalDisk(*)\*" "\Server\*" "\System\*" "\Process(*)\*" `
    "\Processor(*)\*" "\Cache\*" "\SMB Client Shares(*)\*" `
    "\Server Work Queues(*)\*" "\SMB Server(*)\*" `
    "\SMB Server Sessions(*)\*" "\SMB Server Shares(*)\*" `
    -si 00:00:01 -o C:\PerfMonLogs\Perf-1Second.blg
```

### 6.5 Process Monitor

- 了解 SMB Client 或 Server 上在问题发生时的进程行为
- Procmon 本身是 File System Mini Filter
- 如果有其他第三方 filter 在其上方，callstack 不会显示这些 filter
- **养成习惯运行 PROCMON trace**来了解负责触发 UNC IO 的整个内核+用户模式栈

### 6.6 Debugger Extensions

```
!fskd.d <addr>     ← Dump any tagged structure
!fskd.arxc         ← List all active RX_CONTEXTs
!winde.svq         ← Print Workstation Work Queues
```

### 6.7 SMB Configuration 查看

```powershell
# Server 配置
Get-SmbServerConfiguration
# Registry: HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters

# Client 配置
Get-SmbClientConfiguration
# Registry: HKLM:\SYSTEM\CurrentControlSet\Services\LanmanWorkStation\Parameters
```

### 6.8 Event Logs

```
Application and Service Logs → Microsoft → Windows → SMBClient → *
Application and Service Logs → Microsoft → Windows → SMBServer → *
```

## 7. 关键配置与参数 (Key Configurations)

| 配置项 | 位置 | 默认值 | 说明 |
|--------|------|--------|------|
| SMB Server Config | `HKLM:\...\LanmanServer\Parameters` | — | 服务器端 SMB 配置 |
| SMB Client Config | `HKLM:\...\LanmanWorkStation\Parameters` | — | 客户端 SMB 配置 |
| `AsymmetryMode` | LanmanServer\Parameters | 0 | SOFS 非对称模式控制 |
| `EncryptData` | 共享/服务器级别 | $false | SMB Encryption |
| Server Work Queue Max Threads | — | 10/CPU | 服务器工作队列线程 |
| CSC Parameters | `HKLM:\...\Services\CSC\Parameters` | — | Offline Files 配置 |
| CSC Flags | `HKLM:\...\LanmanServer\Shares` | — | 每个共享的 CSC 标志 |
| FormatDatabase | CSC\Parameters | — | 清除 CSC 缓存（重置） |
| RDR Autologger | `HKLM:\...\WMI\Autologger\RdrLog` | — | 持久化 Redirector tracing |
| UNC Hardening | Group Policy / Registry | — | UNC 路径安全增强 |

## 8. SMB Share Permission 备份与迁移

在灾难恢复场景下，SMB 共享权限的备份和恢复：

- **磁盘迁移** → SHA Team
- **NTFS 权限** → AD Team  
- **SMB 共享权限** → Networking Team

**使用 PowerShell 脚本进行共享权限导出/导入：**
```powershell
# 导出共享权限到 XML
Import-Module .\Function_Get-SharePermission.psm1
Get-SharePermission

# 在新服务器上导入
Set-SharePermission -ImportXML SharePermission.xml
```

**文件和文件夹迁移保持权限：**
```powershell
# 使用 Robocopy 保持 NTFS 权限
robocopy \\source\share \\dest\share /E /COPYALL /R:1 /W:1
```

## 9. 与相关技术的对比 (Comparison)

| 维度 | SMB | NFS | WebDAV |
|------|-----|-----|--------|
| 协议 | Microsoft 专有（开放规范） | 开放标准 | HTTP 扩展 |
| 端口 | 445 (139) | 2049 | 80/443 |
| 认证 | Kerberos/NTLM | Kerberos/AUTH_SYS | HTTP Auth |
| 加密 | SMB 3.0+ 原生支持 | NFS 4.1 Kerberos | HTTPS |
| AD 集成 | 原生 | 有限 | 无 |
| Windows 驱动 | mrxsmb*.sys | nfsrdr.sys | mrxdav.sys |
| 适用场景 | Windows 文件共享 | Linux/Unix 环境 | Internet 文件访问 |
| 性能对比 | **不能**与 NFS 直接比较 | **不能**与 SMB 直接比较 | 较低 |

> **关键提醒**：SMB 和 NFS 是完全不同的协议，不能用来做速度对比。iPerf 测试网络吞吐量也不能与 SMB 速度对比，因为 iPerf 不考虑存储性能。

## 10. 参考资料 (References)

暂无可验证的参考文档

---

# English Version

## 1. Overview

SMB (Server Message Block) is a network file sharing protocol designed by Microsoft that allows applications and users to read/write files and request services over a network. It is the foundational protocol for Windows file sharing, print services, and many RPC communications.

SMB's core role in enterprise networks:
- **File Sharing**: Access remote files via UNC paths (e.g., `\\server\share`)
- **Active Directory**: Domain join, Group Policy, logon scripts all rely on SMB
- **Hyper-V over SMB**: VM storage can reside directly on SMB shares
- **SQL Server over SMB**: Database files can be stored on SMB shares
- **RPC over Named Pipes**: Many management tools use RPC via SMB's IPC$

SMB protocol evolution:

| Version | OS | Key Features |
|---------|----|--------------| 
| CIFS | Windows NT 4.0 (1996) | Ancient version |
| SMB 1.0 | Windows 2000/XP/2003 | Deprecated, security risk |
| SMB 2.0 | Vista SP1 / Server 2008 | Completely redesigned, major perf improvement |
| SMB 2.1 | Windows 7 / Server 2008 R2 | Leasing, Large MTU |
| SMB 3.0 | Windows 8 / Server 2012 | Encryption, SMB Direct, CA |
| SMB 3.02 | Windows 8.1 / Server 2012 R2 | Improvements |
| SMB 3.1.1 | Windows 10 / Server 2016+ | Pre-auth, encryption improvements |

> **Critical principle**: Never troubleshoot SMB by disabling SMB2/3. Always recommend customers stop using SMBv1 unless there's a business-critical requirement.

## 2. Core Concepts

### 2.1 SMB Framework — Protocol Structure

SMB communication follows a strict hierarchical framework — understanding this is the foundation for analyzing network captures and ETL logs:

```
1. SMB Dialect Negotiation         ← Negotiate protocol version
2. Authentication (Kerberos/NTLM)  ← Authenticate
3. SMB Session Setup               ← Establish session (Session ID)
4. SMB Tree Connect                ← Connect to share (Tree ID)
5. SMB Create / File Operations    ← File operations (File ID)
6. SMB Tree Disconnect             ← Disconnect from share
7. SMB Logoff                      ← Log off session
```

**IPC$ Role**: IPC$ (Interprocess Communications) is a special share created by the Server Service to allow connections to Named Pipes. Once a Named Pipe is open, RPC over SMB communication can occur.

### 2.2 SMB Identifiers

| Identifier | Generated By | Purpose |
|-----------|--------------|---------|
| **PID** | Redirector | Uniquely identifies a particular application process |
| **Session ID** | Server | Uniquely identifies a particular authenticated user session |
| **TID (Tree ID)** | Server | Uniquely identifies a 'tree connection' to a share |
| **MID (Message ID)** | Redirector | Uniquely identifies a particular SMB transaction |
| **FID (File ID)** | Server | Uniquely identifies a particular server-side open file instance |

> SMB 2.0 File ID is larger (16 bytes) with **Persistent** and **Volatile** portions. The Persistent portion is valid for Resilient/Durable Handles.

### 2.3 SMB Authentication

**Kerberos (Preferred)**:
1. Client obtains Kerberos Ticket from DC (if not in cache)
2. Sends Ticket to SMB Server in Session Setup
3. Wireshark filter: `kerberos`

**NTLM (Fallback)** — used when:
- Authenticating to a server using an IP address
- Cross-forest with legacy NTLM trust
- Server doesn't belong to a domain
- Firewall blocks Kerberos (TCP 88)
- Wireshark filter: `ntlmssp`

### 2.4 File Permissions vs Share Permissions

**Effective Permissions**: When combining file system and shared folder permissions, the **most restrictive** is applied. User must have **both** to access.

**Permission conflict precedence**: Explicit Deny > Explicit Allow > Inherited Deny > Inherited Allow

## 3. How It Works

### 3.1 SMB Server Kernel Stack

```
User-Mode:  LanmanServer (srvsvc.dll / sscore.dll)
            ↕
Kernel:     SRV.sys (SMB1) | SRV2.sys (SMB2) | SRVNET.sys (Net Infra)
            ↕
            IO Manager / Cache Manager
            Filter Manager (third-party AV filters)
            NTFS.sys / ReFS.sys (Local FS)
            Storage / Volume
            ↕
            AFD.sys → TCPIP.sys (Port 445)
            NetBT.sys → TDX.sys (Port 139)
            NDIS.sys
```

### 3.2 SMB Client Kernel Stack

```
Filter Manager (fltmgr.sys)
    ↓
MUP.sys (Multiple UNC Provider — owns \\* namespace)
    ↓
    ├── Surrogates: DFSC.sys (DFS), CSC.sys (Offline Files)
    └── RDBSS.sys (Major Redirector)
            └── MRXSMB.sys (Master Redirector)
                    ├── mrxsmb10.sys (SMB1)
                    └── mrxsmb20.sys (SMB2/3)
Other: mrxdav.sys (WebDAV), nfsrdr.sys (NFS), rdpdr.sys (RDP)
Transport: AFD.sys (WSK) → TCPIP.sys
```

### 3.3 Key Kernel Components

**MUP.sys**: Owns the `\\*` object (UNC namespace). All calls starting with `\\` are routed to MUP. First driver to get UNC IO after the Filter Manager. Maintains a state machine to orchestrate surrogates/redirectors.

**RDBSS.sys**: The "Major-Redirector" providing a plug-in interface for mini-redirectors. Handles caching via Cache Manager. Uses WSK (Win7+) through AFD.sys for network communication.

**MRXSMB.sys**: Master redirector owning UNC IOs on ports 445 and 139. Contains Connection Engine, Transport Engine, CSQ Engine, and Exchange Engine.

**Surrogate Providers**: CSC.sys (Offline Files — deprecated) and DFSC.sys (DFS Client) are called in PRE/POST phases around UNC provider IO.

**Filter Manager**: Third-party mini-filters (AV, etc.) intercept IOs via PRE/POST callbacks. Procmon itself is a mini-filter. If other filters sit above procmon, their callstacks won't show in procmon traces.

### 3.4 SMB2 Protocol Negotiation

For backward compatibility:
1. SMB 2.0 client sends SMB 1.0 format Negotiate with `"SMB 2.001"` dialect
2. If server supports SMB 2.0, it responds with SMB 2.0 format (Protocol ID = `0xFE 'S' 'M' 'B'`)

Key response fields: `ServerGuid`, `Capabilities`, `MaxTransactSize`/`MaxReadSize`/`MaxWriteSize`, `SecurityBuffer`

### 3.5 Command Compounding

SMB 2.0 allows multiple commands with their own headers in a single message:
- **Related**: `SMB2_FLAGS_RELATED_OPERATIONS` set — handled as a chain (e.g., Create → Read)
- **Unrelated**: Independent processing, responses can be sent separately in any order

### 3.6 Pipelined Reads & Writes

Removes SMB 1.0's 64K-1 limitation. Redirector queues multiple sequential reads/writes simultaneously based on credit availability. Applications should request 1-4MB reads/writes with multiple async requests.

## 4. Advanced Features

### 4.1 Scale-Out File Server (SOFS)

Active-active clustered file server (Server 2012+) for application workloads (Hyper-V, SQL Server):
- Uses CSV (Cluster Shared Volumes)
- Requires SMB 3.0+ clients
- Distributed Network Name (DNN) — DNS Round Robin, no virtual IPs
- **Incompatible** with IW features (Offline Files, DFS-R, FSRM)

### 4.2 Continuously Available (CA) Shares

Zero-downtime via **Persistent Handles** and **Resume Key Filter (RKF)**:
- Handle state preserved during disconnection in persistent DB
- RKF blocks other client access during failover, resumes state after
- All IO operates in write-through mode — **not recommended for many small, frequently changing files**

### 4.3 SMB Witness Protocol

Faster failover recovery without TCP timeouts (~20s):
- Client registers with a Witness server on a **different node** than the data node
- Witness notifies client immediately when data node fails
- Client reconnects to another cluster node without waiting

### 4.4 SMB Multichannel

Automatically establishes multiple TCP connections across NICs for aggregated bandwidth and network fault tolerance.

### 4.5 SMB Encryption (SMB 3.0+)

```powershell
Set-SmbShare -Name "SecureData" -EncryptData $true
Set-SmbServerConfiguration -EncryptData $true  # Entire server
```
> **Warning**: Encryption uses file server CPU. May cause high CPU with many shares/users. Enable only for critical data.

### 4.6 SMB over QUIC (Server 2022 Azure Edition)

Uses HTTP/3 + TLS 1.3: `net use * \\server\share /transport:quic`

### 4.7 Other Features

- **ABE**: Controls file visibility based on permissions. Domain Admin overrides NTFS Full Deny.
- **BranchCache**: Caches content at branch offices to reduce WAN bandwidth.
- **CSC (Offline Files)**: Deprecated. Bypass with `\\server$nocsc$\share`. Replaced by Work Folders / OneDrive.
- **UNC Hardening**: RequireMutualAuthentication, RequireIntegrity, RequirePrivacy.

## 5. Common Issues & Troubleshooting

### 5.1 Cannot Access File Share

**Troubleshooting steps:**
1. TCP connectivity: `Test-NetConnection server -Port 445`
2. Server Service status — TCP RST after handshake = service not running
3. SMB Negotiate success?
4. Session Setup errors? (Authentication)
5. Tree Connect success? (Share access)
6. Name resolution correct?

### 5.2 Kerberos Authentication Failure (KRB5KRB_AP_ERR_MODIFIED)

**Common cause**: Machine account password mismatch between server's LSA Secret and AD

**Solution**:
```powershell
# On the SMB server (logged in as local admin)
Reset-ComputerMachinePassword -Server "dc01"
```

### 5.3 File Copy Slowness

**Key principles:**
- Must have a **comparison baseline** (expected vs actual speed)
- **Cannot compare** iPerf with SMB (iPerf ignores storage performance)
- **Cannot compare** NFS with SMB (completely different protocols)
- Only valid method: **collect logs in two scenarios and compare**

**Network layer**: Check for retransmits, high latency, TCP Window Scaling issues, MTU problems.

**Storage layer (PerfMon)**:
- `\PhysicalDisk\Avg. Disk sec/Transfer`: < 10ms good, > 50ms serious bottleneck
- `\Server Work Queues\Queue Length`: Should be 0
- `\Server Work Queues\Available Work Items`: Should be > 0

**File Explorer slow but CMD normal**: Likely Explorer UI issue, engage OS Team.

### 5.4 SOFS Scale-Out Rebalancing Not Working

After CSV failover, if the previous owner node is **still accessible**, existing SMB connections are **not** changed. Only **new connections** go to the new CSV owner. Connections change only when the previous owner becomes **inaccessible**.

### 5.5 Kerberos vs NTLM Authentication Identification

```
Wireshark filters:
  Kerberos auth: kerberos
  NTLM auth:     ntlmssp
```

## 6. Diagnostic Tools & Methods

### 6.1 T.cmd — SMB ETW Tracing

```cmd
# Client                          # Server
cd c:\logs                         cd c:\logs
t.cmd clion                        t.cmd srvon
:: reproduce issue                 :: reproduce issue
t.cmd clioff                       t.cmd srvoff
```

> Use T10.cmd for Win10/Server 2016+.

**ETL files produced**: fskm.etl (core SMB), sec.etl (Kerberos/NTLM), dns.etl, tcp.etl, nbt.etl

**T.cmd analysis method:**
1. Start from `MupCreate()`
2. Filter errors — find the first error location
3. Match `MRxSmb2AssembleSmb` (SEND) and `SmbCeReceiveInd` (RECV) with network trace using MID/SID/TID

### 6.2 Network Capture Analysis

Always start from network trace (Netmon/Wireshark). Include SMB Request/Response details. If unexplained, check T.cmd.

**Quick SMB error filters:**
- Wireshark: `smb2.error.data`
- Netmon: `smb2.ErrorMessage`

### 6.3 Performance Monitor

Collect at 1-second intervals covering: LogicalDisk, PhysicalDisk, Memory, Network Interface, Server, Server Work Queues, SMB Client/Server Shares, Process, Processor.

### 6.4 RDR Autologger Tracing

For crash debugging: `HKLM\...\WMI\Autologger\RdrLog`. Uses ring buffer (non-paged pool). Extract from memory dump with `!wmitrace.strdup`/`!wmitrace.logsave`.

### 6.5 Debugger Extensions

```
!fskd.d <addr>   — Dump tagged structure
!fskd.arxc       — List active RX_CONTEXTs
!winde.svq       — Print Workstation Work Queues
```

### 6.6 SMB Configuration

```powershell
Get-SmbServerConfiguration    # HKLM:\...\LanmanServer\Parameters
Get-SmbClientConfiguration    # HKLM:\...\LanmanWorkStation\Parameters
```

## 7. Key Configurations

| Configuration | Location | Default | Description |
|--------------|----------|---------|-------------|
| SMB Server Config | `HKLM:\...\LanmanServer\Parameters` | — | Server-side SMB settings |
| SMB Client Config | `HKLM:\...\LanmanWorkStation\Parameters` | — | Client-side SMB settings |
| `AsymmetryMode` | LanmanServer\Parameters | 0 | SOFS asymmetric mode |
| `EncryptData` | Share/Server level | $false | SMB Encryption |
| Work Queue Max Threads | — | 10/CPU | Server work queue threads |
| CSC Parameters | `HKLM:\...\Services\CSC\Parameters` | — | Offline Files config |
| CSC Flags | `HKLM:\...\LanmanServer\Shares` | — | Per-share CSC flags |
| `FormatDatabase` | CSC\Parameters | — | Clear CSC cache (reset) |
| RDR Autologger | `HKLM:\...\WMI\Autologger\RdrLog` | — | Persistent RDR tracing |
| UNC Hardening | Group Policy / Registry | — | UNC path security |

## 8. SMB Share Permission Backup & Migration

- **Disk migration** → Storage Team
- **NTFS permissions** → AD Team
- **SMB share permissions** → Networking Team

```powershell
# Export
Import-Module .\Function_Get-SharePermission.psm1
Get-SharePermission  # Saves SharePermission.xml

# Import on new server
Set-SharePermission -ImportXML SharePermission.xml
```

## 9. Comparison with Related Technologies

| Dimension | SMB | NFS | WebDAV |
|-----------|-----|-----|--------|
| Protocol | Microsoft (open spec) | Open standard | HTTP extension |
| Port | 445 (139) | 2049 | 80/443 |
| Auth | Kerberos/NTLM | Kerberos/AUTH_SYS | HTTP Auth |
| Encryption | SMB 3.0+ native | NFS 4.1 Kerberos | HTTPS |
| AD Integration | Native | Limited | None |
| Windows Driver | mrxsmb*.sys | nfsrdr.sys | mrxdav.sys |
| Best For | Windows file sharing | Linux/Unix environments | Internet file access |

> **Key reminder**: SMB and NFS are entirely different protocols — do not compare speeds between them. iPerf tests network throughput without storage consideration — cannot compare with SMB speed.

## 10. References

暂无可验证的参考文档 / No verifiable reference links available at this time.
