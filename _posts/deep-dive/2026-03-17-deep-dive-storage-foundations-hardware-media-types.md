---
layout: post
title: "Deep Dive: Windows 存储基础 — 硬件类型与存储介质"
date: 2026-03-17
categories: [Knowledge, Storage]
tags: [storage, hdd, ssd, nvme, pmem, san, nas, iscsi, fiber-channel, das, jbod, cloud-storage]
type: "deep-dive"
---

<!-- ============================================================ -->
<!--  中 文 版                                                      -->
<!-- ============================================================ -->

# 🇨🇳 中文版

---

## 1. 概述 — 什么是存储，为什么需要理解硬件？

> **一句话定义：** 存储就是把数据放在"某种介质"上，以便日后读取。

对于 Windows 支持工程师来说，理解存储硬件至关重要，原因有三：

1. **排查性能问题** — 磁盘延迟 > 20 ms？是介质问题还是链路问题？不了解硬件就无从判断。
2. **理解分层架构** — Windows 存储栈从上到下依次是 `应用 → 文件系统 → 卷管理器 → 端口/Miniport → 物理硬件`，硬件是最底层的基石。
3. **正确匹配工作负载** — 不同介质有不同的 IOPS、延迟、吞吐特征，选错介质是很多"性能慢"工单的根因。

```
┌─────────────────────────────────────────┐
│           应用程序 (Application)          │
├─────────────────────────────────────────┤
│         文件系统 (NTFS / ReFS)           │
├─────────────────────────────────────────┤
│       卷管理器 (Volume Manager)          │
├─────────────────────────────────────────┤
│     端口/Miniport 驱动 (StorPort等)       │
├─────────────────────────────────────────┤
│  ★ 物理硬件 (本文重点) ★                  │
│  HDD │ SSD │ NVMe │ PMEM │ SAN │ NAS   │
└─────────────────────────────────────────┘
```

---

## 2. 存储介质类型

### 2.1 HDD（机械硬盘 — 磁性介质）

> **🎯 类比：** HDD 就像一台**黑胶唱片机** — 唱臂（执行器）在旋转的唱片（盘片）上移动，读写头沿磁道寻找数据，就像唱针在唱片沟槽上滑动读取音乐一样。唱片要转到正确位置，唱臂要移到正确磁道，所以有"等待"。

#### 物理结构

```
          ┌───── 主轴电机 (Spindle Motor)
          │        旋转盘片
          │
    ┌─────┴─────┐
    │ ╔═══════╗ │  ← 盘片 (Platter) - 涂有磁性材料
    │ ║       ║ │
    │ ║   ●   ║ │  ← 主轴 (Spindle)
    │ ║       ║ │
    │ ╚═══════╝ │
    │     ↑     │
    │  读写头   │  ← 读写头 (R/W Head) - 浮在盘片上方
    │  (悬浮)   │     间距约 3-5 纳米！
    │ ←───────  │  ← 执行器臂 (Actuator Arm)
    └───────────┘
```

#### CHS 寻址 vs LBA 寻址

**CHS（柱面/磁头/扇区）** — 早期方式，直接描述物理位置：
- **Cylinder（柱面）**：同一半径上所有盘面的磁道组成一个"柱面"
- **Head（磁头）**：哪个盘面 = 哪个磁头
- **Sector（扇区）**：磁道上的哪个扇区（通常 512 字节）

```
俯视图 (Top View):
            磁道 (Track)
        ╭───────────╮
      ╭─┤  ╭─────╮  ├─╮
    ╭─┤  │  │  ●  │  │  ├─╮   ● = 主轴
    │ │  │  ╰─────╯  │  │ │
    │ ╰──┤  ╰───────╯  ├──╯   每一圈 = 1条磁道
    ╰────┤             ├────╯   同心圆上分成多个扇区(Sector)
         ╰─────────────╯
```

**LBA（逻辑块寻址）** — 现代方式，把所有扇区编号为 `0, 1, 2, 3, ...`，由硬盘固件自动映射到物理位置。操作系统不再关心物理几何结构。

| 寻址方式 | 谁管物理位置？ | 容量限制 | 现代是否使用？ |
|---------|-------------|---------|-------------|
| CHS | 操作系统 | ~8 GB | ❌ 已淘汰 |
| LBA | 硬盘固件 | 远超 PB 级 | ✅ 标准方式 |

#### 转速（RPM）与性能

| RPM | 常见场景 | 平均寻道时间 | 典型 IOPS |
|-----|---------|------------|----------|
| 4,200 | 早期笔记本 | ~15 ms | ~40 |
| 5,400 | 笔记本 / 冷存储 | ~12 ms | ~50-75 |
| 7,200 | 桌面 / 常规服务器 | ~9 ms | ~75-100 |
| 10,000 | 企业级 (WD VelociRaptor) | ~5 ms | ~140 |
| 15,000 | 高端企业级 SAS | ~3.5 ms | ~180-200 |

> **关键点：** RPM 越高 → 转一圈越快 → **旋转延迟 (rotational latency)** 越短。但 RPM 再高也受限于机械运动，这就是 SSD 诞生的原因。

---

### 2.2 SSD（固态硬盘 — 闪存介质）

> **🎯 类比：** SSD 就像一个**巨大的开关网格** — 每个开关存储一个比特（0 或 1）。不需要等待任何东西旋转或移动，通电即可读写。

```
SSD 内部逻辑结构：

┌─────────────────────────────────────┐
│         SSD 控制器 (Controller)       │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐  │
│  │NAND │ │NAND │ │NAND │ │NAND │  │
│  │Die 0│ │Die 1│ │Die 2│ │Die 3│  │
│  │     │ │     │ │     │ │     │  │
│  │Block│ │Block│ │Block│ │Block│  │
│  │Page │ │Page │ │Page │ │Page │  │
│  └─────┘ └─────┘ └─────┘ └─────┘  │
│         DRAM Cache (缓存)            │
└─────────────────────────────────────┘

写入层级：
  Page (页) = 最小写入单位 (4-16 KB)
  Block (块) = 最小擦除单位 (包含 64-256 个 Page)
```

#### SSD 固件的关键功能

| 固件功能 | 作用 | 为什么重要？ |
|---------|------|------------|
| **ECC（纠错码）** | 检测并修正读取错误 | NAND 本身有较高的位翻转率 |
| **磨损均衡 (Wear Leveling)** | 均匀分配写入到所有闪存块 | 每个块有写入寿命（TLC ~3000次） |
| **坏块管理 (Bad Block Mapping)** | 标记和替换损坏的块 | NAND 出厂就可能有坏块 |
| **垃圾回收 (Garbage Collection)** | 整理无效页，释放可用块 | NAND 必须"先擦后写"，不能覆盖 |
| **读干扰管理 (Read Disturb)** | 检测并刷新频繁读取的相邻页 | 反复读同一区域可能干扰相邻单元 |

> **🎯 磨损均衡类比：** 就像**在电影院轮流换座位** — 如果你每次都坐同一个位置，那个座位会比其他的先坏掉。磨损均衡确保每个"座位"（闪存块）被均匀使用，这样整块 SSD 可以活更久。

```
磨损均衡原理：

没有磨损均衡:                    有磨损均衡:
Block 0: ████████ (写满!)        Block 0: ███
Block 1: ████████ (写满!)        Block 1: ████
Block 2: █                       Block 2: ███
Block 3:                         Block 3: ███
Block 4:                         Block 4: ████
→ Block 0,1 很快损坏              → 所有块均匀使用，寿命更长
```

---

### 2.3 NVMe（非易失性内存快捷通道）

> **NVMe = NVM Express = 走 PCIe 总线的 SSD**

传统 SSD 通过 SAS 或 SATA 接口连接，而 NVMe 直接走 PCIe 总线——这就像从乡间小路直接上了高速公路。

```
传统 SSD 路径:
  CPU ←→ SATA/SAS 控制器 ←→ SSD
         (瓶颈在这里!)

NVMe 路径:
  CPU ←→ PCIe 总线 ←→ NVMe SSD
         (直连，极少中间环节)
```

#### 关键优势

| 对比项 | SATA SSD | NVMe SSD |
|-------|----------|----------|
| 接口 | SATA (AHCI 协议) | PCIe (NVMe 协议) |
| 队列数 | 1 个队列，32 深度 | **65,535 个队列，每个 65,536 深度** |
| CPU 指令数 | ~数万条 / IO | **~数千条 / IO（约一半）** |
| 典型延迟 | ~100 μs | **~10-20 μs** |
| 带宽 | ~600 MB/s (SATA3上限) | **~3,500-7,000+ MB/s** |

#### NVMe-oF（NVMe over Fabrics）

把 NVMe 的性能优势扩展到网络：

```
本地 NVMe:
  服务器 ──PCIe──→ NVMe SSD     (延迟 ~10μs)

NVMe-oF:
  服务器 ──RDMA/FC──→ 远端NVMe存储  (延迟 ~数十μs)
  
  相比 iSCSI 远程存储 (延迟 ~数百μs-ms), 
  NVMe-oF 大幅降低了远程访问延迟。
```

---

### 2.4 SCM / PMEM（存储级内存 / 持久性内存）

> **🎯 类比：** 如果 HDD 是**火车**（慢但容量大），SSD 是**汽车**（快），NVMe 是**跑车**，那么 PMEM 就是**传送门** — 它直接连在内存总线上，延迟低到接近 DRAM，但数据掉电不丢失。

```
延迟对比 (数量级):

HDD        ████████████████████████████████  ~5-15 ms (毫秒)
SSD (SATA) ██████                            ~100 μs  (微秒)
NVMe       ███                               ~10-20 μs
PMEM       █                                 ~数百 ns - 数 μs
DRAM       ▏                                 ~100 ns  (纳秒)
```

#### 关键特点

| 特性 | 说明 |
|-----|------|
| 连接方式 | **内存总线** (DIMM 插槽)，不是 PCIe |
| 延迟 | 微秒级（μs），比 NVMe 更低 |
| 持久性 | 掉电不丢数据 (Non-Volatile) |
| Windows 支持 | Windows Server 2019+ |
| 使用模式 | App Direct / Memory Mode |

> **模糊了 DRAM 与存储的界限：** 传统上，DRAM 是"快但掉电丢数据"的内存，存储是"慢但数据持久"的硬盘。PMEM 打破了这个界限 — 它既快又持久。

---

## 3. 存储连接类型

### 3.1 DAS（直连存储）

> **DAS = Direct Attached Storage = 直接插在服务器上的硬盘**

```
┌──────────────┐
│   服务器      │
│  ┌────┐      │
│  │ CPU│      │
│  └──┬─┘      │
│     │ (背板)  │
│  ┌──┴──────┐ │
│  │ 磁盘0   │ │
│  │ 磁盘1   │ │  ← JBOD (Just a Bunch of Disks)
│  │ 磁盘2   │ │     "就是一堆硬盘"
│  │ 磁盘3   │ │
│  └─────────┘ │
└──────────────┘
```

- **优势：** 简单、低延迟、低成本
- **劣势：** 只有这台服务器能用，不能共享
- **JBOD（Just a Bunch of Disks）：** 多块硬盘通过背板连接，没有 RAID 控制器，每块盘独立呈现给操作系统

---

### 3.2 SAN（存储区域网络）

> **SAN = Storage Area Network = 通过专用网络连接的块级存储**

```
┌──────────┐     Fiber Channel     ┌──────────────┐
│ 服务器A   │ ←─────────────────→  │              │
│ (HBA卡)  │                      │  SAN 存储阵列  │
└──────────┘     FC 交换机          │              │
                  ┌────┐           │  LUN 0       │
┌──────────┐     │ FC │           │  LUN 1       │
│ 服务器B   │ ←──┤交换│──────→    │  LUN 2       │
│ (HBA卡)  │     │ 机 │           │              │
└──────────┘     └────┘           └──────────────┘
```

#### SAN 关键概念

| 术语 | 含义 |
|-----|------|
| **HBA** (Host Bus Adapter) | 服务器上的光纤卡，类似网卡但用于存储 |
| **LUN** (Logical Unit Number) | 存储阵列划分出的逻辑磁盘单元 |
| **Zoning** | 控制哪些 HBA 能看到哪些 LUN（类似网络的 VLAN） |
| **FC (Fiber Channel)** | 专用的存储网络协议/物理层 |
| **WWPN** | 光纤端口的全球唯一地址 |

> **核心特征：块级存储** — SAN 呈现给 Windows 的是一个"本地磁盘"（尽管它其实在远端）。在磁盘管理中看起来和本地盘一样。Windows 在上面创建分区、格式化 NTFS，完全不知道数据实际在几公里外的机房。

---

### 3.3 NAS（网络附属存储）

> **NAS = Network Attached Storage = 通过网络共享的文件级存储**

```
┌──────────┐                      ┌──────────────┐
│ Windows  │  SMB/CIFS 协议        │              │
│ 客户端    │ ──────────────────→  │  NAS 设备     │
│          │  \\NAS01\ShareName   │              │
│ 映射为   │                      │  文件系统管理   │
│ Z: 盘    │                      │  由 NAS 自己   │
└──────────┘                      └──────────────┘
```

- 通过 **SMB/CIFS** 协议（Windows）或 **NFS** 协议（Linux）
- 使用 **UNC 路径**访问：`\\ServerName\ShareName\folder\file.txt`
- **文件级存储** — NAS 设备管理自己的文件系统，客户端只能看到文件和文件夹

---

### 3.4 iSCSI

> **🎯 类比：** iSCSI 就像把一封**信件（SCSI 命令）装进普通信封（TCP/IP 数据包）** — 它使用现有的**邮政系统（以太网）** 来投递，而不需要专门的**快递服务（Fiber Channel）**。效果一样 — 都是把块级存储命令送达目的地，只是运输方式更便宜、更普及。

```
iSCSI 封装原理：

┌─────────────────────────────────┐
│  以太网帧 (Ethernet Frame)       │
│  ┌───────────────────────────┐  │
│  │  TCP/IP 包                 │  │
│  │  ┌─────────────────────┐  │  │
│  │  │  iSCSI PDU          │  │  │
│  │  │  ┌───────────────┐  │  │  │
│  │  │  │ SCSI 命令/数据  │  │  │  │
│  │  │  └───────────────┘  │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

#### iSCSI 关键术语

| 术语 | 含义 |
|-----|------|
| **Initiator（发起者）** | 客户端（Windows 内置 iSCSI Initiator） |
| **Target（目标）** | 存储端，提供磁盘 |
| **IQN** (iSCSI Qualified Name) | 唯一标识，格式：`iqn.yyyy-mm.com.vendor:target-name` |
| **Portal** | Target 的 IP:Port（默认端口 3260） |

#### iSCSI vs Fiber Channel

| 对比项 | iSCSI | Fiber Channel |
|-------|-------|---------------|
| 网络类型 | 标准以太网 | 专用光纤网络 |
| 成本 | 低（复用现有网络） | 高（专用 HBA + 交换机） |
| 性能 | 好（10/25 GbE 可接近 FC） | 极好（16/32 Gbps FC） |
| 管理复杂度 | 较低 | 较高（需要 Zoning 等） |
| 适用场景 | 中小企业、测试环境 | 大型企业、关键业务 |

---

### 3.5 文件级 vs 块级存储对比

| 对比维度 | 文件级 (NAS) | 块级 (SAN / iSCSI / DAS) |
|---------|-------------|-------------------------|
| 协议 | SMB, NFS | FC, iSCSI, SAS |
| Windows 中呈现 | 网络共享 (UNC 路径) | 本地磁盘 (盘符) |
| 文件系统 | NAS 设备管理 | Windows 自己管理 (NTFS/ReFS) |
| 典型用途 | 文件共享、Home Drive | 数据库、Hyper-V、Exchange |
| 性能 | 受协议开销影响 | 更接近原生磁盘性能 |
| 灵活性 | 即插即用 | 需要更多规划配置 |

```
文件级 vs 块级 — 一图看懂：

文件级 (NAS):
  App → "读取 \\NAS\share\report.docx"
  NAS 设备负责找到文件并返回内容

块级 (SAN/iSCSI):
  App → NTFS → "读取 LBA 第 4096 扇区"
  Windows 自己的 NTFS 管理文件在哪些块
  SAN 只负责提供原始的"块"
```

---

## 4. 云与混合存储

### 4.1 云存储类型

| 类型 | 示例 | 特点 |
|-----|------|------|
| **文件同步** | OneDrive, Dropbox | 客户端同步，按需下载 |
| **Blob/对象存储** | Azure Blob, AWS S3 | 海量非结构化数据，REST API 访问 |
| **云文件共享** | Azure Files | 兼容 SMB 协议，可直接挂载 |
| **云块存储** | Azure Managed Disk | VM 的虚拟硬盘 |

### 4.2 混合存储

```
混合存储架构：

┌──────────────┐          ┌─────────────────┐
│  本地服务器    │   同步    │   Azure Cloud   │
│              │ ←─────→  │                 │
│  热数据       │          │  冷数据 / 归档    │
│  (频繁访问)   │          │  (不常访问)      │
│              │  分层策略  │                 │
│  SSD / NVMe  │ ────────→ │  Blob Storage   │
└──────────────┘          └─────────────────┘
```

- **Azure StorSimple**：自动在本地 SSD 和云之间分层
- **Azure File Sync**：将本地文件服务器与 Azure Files 同步，本地保留缓存
- **核心理念：** 热数据放本地（快），冷数据放云端（便宜），实现成本与性能的平衡

---

## 5. 总线类型对比表

| 类型 | 总线/接口 | 典型延迟 | 典型 IOPS | 典型带宽 | 适用场景 |
|------|----------|---------|----------|---------|---------|
| **HDD** | SAS / SATA | 5-15 ms | 75-200 | 100-250 MB/s | 大容量、冷数据、备份 |
| **SSD** | SAS / SATA | 50-100 μs | 20K-100K | 500-600 MB/s | 常规工作负载 |
| **NVMe** | PCIe | ~10-20 μs | 100K-1M+ | 3-7 GB/s | 高性能、数据库 |
| **SCM/PMEM** | Memory Bus | 数百 ns-数 μs | 数百万+ | 10+ GB/s | 极致性能、内存数据库 |

> **速度差异可视化：**
> ```
> 如果 HDD 读取一次需要 1 天 (代表 10 ms)...
>   SSD  = 约 14 分钟  (100 μs)
>   NVMe = 约 3 分钟   (20 μs)
>   PMEM = 约 9 秒      (1 μs)
>   DRAM = 约 1 秒      (100 ns)
> ```

---

## 6. 关键术语表

| 术语 | 全称 | 中文解释 |
|------|------|---------|
| **HDD** | Hard Disk Drive | 机械硬盘，使用旋转磁盘 |
| **SSD** | Solid State Drive | 固态硬盘，使用闪存芯片 |
| **NVMe** | Non-Volatile Memory Express | 基于 PCIe 的存储协议 |
| **PMEM** | Persistent Memory | 持久性内存，连接内存总线 |
| **SCM** | Storage Class Memory | 存储级内存，PMEM 的同义词 |
| **CHS** | Cylinder/Head/Sector | 早期磁盘物理寻址方式 |
| **LBA** | Logical Block Addressing | 逻辑块寻址，现代标准 |
| **RPM** | Revolutions Per Minute | 每分钟转数，衡量 HDD 转速 |
| **NAND** | Not AND (闪存类型) | SSD 使用的闪存技术 |
| **ECC** | Error Correction Code | 纠错码 |
| **DAS** | Direct Attached Storage | 直连存储 |
| **SAN** | Storage Area Network | 存储区域网络 |
| **NAS** | Network Attached Storage | 网络附属存储 |
| **iSCSI** | Internet SCSI | 基于 TCP/IP 的 SCSI 协议 |
| **FC** | Fiber Channel | 光纤通道 |
| **HBA** | Host Bus Adapter | 主机总线适配器（光纤卡） |
| **LUN** | Logical Unit Number | 逻辑单元号 |
| **WWPN** | World Wide Port Name | 全球唯一端口名 |
| **IQN** | iSCSI Qualified Name | iSCSI 限定名 |
| **JBOD** | Just a Bunch of Disks | 简单磁盘捆绑 |
| **SMB** | Server Message Block | Windows 文件共享协议 |
| **NFS** | Network File System | Linux/Unix 文件共享协议 |
| **IOPS** | I/O Operations Per Second | 每秒 IO 操作数 |
| **NVMe-oF** | NVMe over Fabrics | NVMe 远程访问协议 |
| **RDMA** | Remote Direct Memory Access | 远程直接内存访问 |

---

---

<!-- ============================================================ -->
<!--  E N G L I S H   V E R S I O N                               -->
<!-- ============================================================ -->

# 🇺🇸 English Version

---

## 1. Overview — What Is Storage and Why Does Hardware Matter?

> **One-liner:** Storage is about placing data on "some medium" so it can be read back later.

For Windows support engineers, understanding storage hardware is critical for three reasons:

1. **Troubleshooting performance** — Disk latency > 20 ms? Is it the media or the path? Without hardware knowledge, you can't tell.
2. **Understanding the stack** — The Windows storage stack runs `Application → File System → Volume Manager → Port/Miniport → Physical Hardware`. Hardware is the foundation.
3. **Matching workloads correctly** — Different media have different IOPS, latency, and throughput characteristics. A mismatch is the root cause of many "slow performance" tickets.

```
┌─────────────────────────────────────────┐
│           Application                    │
├─────────────────────────────────────────┤
│         File System (NTFS / ReFS)        │
├─────────────────────────────────────────┤
│       Volume Manager                     │
├─────────────────────────────────────────┤
│     Port/Miniport Driver (StorPort etc.) │
├─────────────────────────────────────────┤
│  ★ Physical Hardware (This Article) ★    │
│  HDD │ SSD │ NVMe │ PMEM │ SAN │ NAS    │
└─────────────────────────────────────────┘
```

---

## 2. Storage Media Types

### 2.1 HDD (Hard Disk Drive — Magnetic Media)

> **🎯 Analogy:** Think of an HDD like a **vinyl record player** — the arm (actuator) moves across spinning platters to read/write data, just like a needle riding the grooves of a record to play music. The platter has to spin to the right spot and the arm has to seek to the right track, so there's always "waiting."

#### Physical Structure

```
          ┌───── Spindle Motor
          │      (spins the platters)
          │
    ┌─────┴─────┐
    │ ╔═══════╗ │  ← Platter - coated with magnetic material
    │ ║       ║ │
    │ ║   ●   ║ │  ← Spindle
    │ ║       ║ │
    │ ╚═══════╝ │
    │     ↑     │
    │  R/W Head │  ← Read/Write Head - floats above platter
    │ (flying)  │     at ~3-5 nanometers!
    │ ←───────  │  ← Actuator Arm
    └───────────┘
```

#### CHS Addressing vs LBA Addressing

**CHS (Cylinder/Head/Sector)** — the legacy approach, describing physical location directly:
- **Cylinder:** All tracks at the same radius across all platters form a "cylinder"
- **Head:** Which platter surface = which read/write head
- **Sector:** Which sector on the track (typically 512 bytes)

```
Top View:
            Track
        ╭───────────╮
      ╭─┤  ╭─────╮  ├─╮
    ╭─┤  │  │  ●  │  │  ├─╮   ● = Spindle
    │ │  │  ╰─────╯  │  │ │
    │ ╰──┤  ╰───────╯  ├──╯   Each ring = 1 track
    ╰────┤             ├────╯   Tracks are divided into sectors
         ╰─────────────╯
```

**LBA (Logical Block Addressing)** — the modern approach. All sectors are numbered `0, 1, 2, 3, ...` and the drive firmware handles the mapping to physical locations. The OS no longer cares about physical geometry.

| Addressing | Who manages physical location? | Capacity limit | Still used? |
|-----------|-------------------------------|---------------|-------------|
| CHS | Operating System | ~8 GB | ❌ Deprecated |
| LBA | Drive firmware | Far beyond PB | ✅ Standard |

#### RPM Speeds and Performance

| RPM | Typical Use | Avg Seek Time | Typical IOPS |
|-----|------------|---------------|-------------|
| 4,200 | Early laptops | ~15 ms | ~40 |
| 5,400 | Laptops / cold storage | ~12 ms | ~50-75 |
| 7,200 | Desktop / general servers | ~9 ms | ~75-100 |
| 10,000 | Enterprise (WD VelociRaptor) | ~5 ms | ~140 |
| 15,000 | High-end enterprise SAS | ~3.5 ms | ~180-200 |

> **Key point:** Higher RPM → faster rotation → lower **rotational latency**. But no matter how fast it spins, it's still limited by mechanical movement — which is exactly why SSDs were invented.

---

### 2.2 SSD (Solid State Drive — Flash Media)

> **🎯 Analogy:** An SSD is like a **giant grid of light switches** — each switch stores a bit (0 or 1). No waiting for anything to spin or move. Flip a switch, get your data.

```
SSD Internal Logical Structure:

┌─────────────────────────────────────┐
│         SSD Controller               │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐  │
│  │NAND │ │NAND │ │NAND │ │NAND │  │
│  │Die 0│ │Die 1│ │Die 2│ │Die 3│  │
│  │     │ │     │ │     │ │     │  │
│  │Block│ │Block│ │Block│ │Block│  │
│  │Page │ │Page │ │Page │ │Page │  │
│  └─────┘ └─────┘ └─────┘ └─────┘  │
│         DRAM Cache                   │
└─────────────────────────────────────┘

Write hierarchy:
  Page = smallest write unit (4-16 KB)
  Block = smallest erase unit (contains 64-256 Pages)
```

#### Key SSD Firmware Functions

| Firmware Function | Purpose | Why It Matters |
|------------------|---------|----------------|
| **ECC (Error Correction Code)** | Detects and corrects read errors | NAND has a relatively high bit-flip rate |
| **Wear Leveling** | Distributes writes evenly across all flash blocks | Each block has a write endurance limit (TLC ~3000 P/E cycles) |
| **Bad Block Mapping** | Marks and replaces damaged blocks | NAND may ship with factory-defective blocks |
| **Garbage Collection** | Consolidates invalid pages, frees usable blocks | NAND requires "erase before write" — no in-place overwrite |
| **Read Disturb Management** | Detects and refreshes pages adjacent to heavily-read areas | Repeatedly reading one area can corrupt neighboring cells |

> **🎯 Wear Leveling Analogy:** It's like **rotating which seats you sit in at a movie theater** — if you always sit in the same seat, that one wears out first. Wear leveling ensures every "seat" (flash block) gets used equally, so the entire SSD lasts longer.

```
Wear Leveling Visualized:

Without Wear Leveling:               With Wear Leveling:
Block 0: ████████ (worn out!)         Block 0: ███
Block 1: ████████ (worn out!)         Block 1: ████
Block 2: █                            Block 2: ███
Block 3:                              Block 3: ███
Block 4:                              Block 4: ████
→ Blocks 0,1 fail early               → All blocks used evenly, longer life
```

---

### 2.3 NVMe (Non-Volatile Memory Express)

> **NVMe = an SSD that connects via the PCIe bus (not SAS/SATA)**

Traditional SSDs connect through SAS or SATA interfaces, while NVMe goes directly over the PCIe bus — like jumping from a country road onto a highway.

```
Traditional SSD Path:
  CPU ←→ SATA/SAS Controller ←→ SSD
         (bottleneck here!)

NVMe Path:
  CPU ←→ PCIe Bus ←→ NVMe SSD
         (direct, minimal overhead)
```

#### Key Advantages

| Comparison | SATA SSD | NVMe SSD |
|-----------|----------|----------|
| Interface | SATA (AHCI protocol) | PCIe (NVMe protocol) |
| Queue count | 1 queue, depth 32 | **65,535 queues, each depth 65,536** |
| CPU instructions | ~tens of thousands / IO | **~thousands / IO (~half)** |
| Typical latency | ~100 μs | **~10-20 μs** |
| Bandwidth | ~600 MB/s (SATA3 max) | **~3,500-7,000+ MB/s** |

#### NVMe-oF (NVMe over Fabrics)

Extends NVMe performance advantages across a network:

```
Local NVMe:
  Server ──PCIe──→ NVMe SSD         (latency ~10μs)

NVMe-oF:
  Server ──RDMA/FC──→ Remote NVMe   (latency ~tens of μs)

  Compare with iSCSI remote storage  (latency ~hundreds of μs to ms)
  NVMe-oF dramatically reduces remote access latency.
```

---

### 2.4 SCM / PMEM (Storage Class Memory / Persistent Memory)

> **🎯 Analogy:** If HDD is a **train** (slow but high capacity), SSD is a **car** (fast), NVMe is a **sports car**, then PMEM is a **teleporter** — it connects directly to the memory bus, with latency approaching DRAM, but data survives power loss.

```
Latency Comparison (orders of magnitude):

HDD        ████████████████████████████████  ~5-15 ms (milliseconds)
SSD (SATA) ██████                            ~100 μs  (microseconds)
NVMe       ███                               ~10-20 μs
PMEM       █                                 ~hundreds of ns - a few μs
DRAM       ▏                                 ~100 ns  (nanoseconds)
```

#### Key Characteristics

| Feature | Details |
|---------|---------|
| Connection | **Memory bus** (DIMM slot), not PCIe |
| Latency | Microsecond-range (μs), lower than NVMe |
| Persistence | Survives power loss (Non-Volatile) |
| Windows support | Windows Server 2019+ |
| Usage modes | App Direct / Memory Mode |

> **Blurs the line between DRAM and storage:** Traditionally, DRAM is "fast but volatile" memory and storage is "slow but persistent" disks. PMEM breaks this boundary — it is both fast AND persistent.

---

## 3. Storage Connection Types

### 3.1 DAS (Direct Attached Storage)

> **DAS = disks physically attached directly to the server**

```
┌──────────────┐
│   Server      │
│  ┌────┐      │
│  │ CPU│      │
│  └──┬─┘      │
│     │(backplane)
│  ┌──┴──────┐ │
│  │ Disk 0  │ │
│  │ Disk 1  │ │  ← JBOD (Just a Bunch of Disks)
│  │ Disk 2  │ │     Exactly what it sounds like!
│  │ Disk 3  │ │
│  └─────────┘ │
└──────────────┘
```

- **Pros:** Simple, low latency, low cost
- **Cons:** Only the directly-connected server can use it; not shareable
- **JBOD (Just a Bunch of Disks):** Multiple disks connected via backplane, no RAID controller, each disk presented independently to the OS

---

### 3.2 SAN (Storage Area Network)

> **SAN = block-level storage accessed over a dedicated network**

```
┌──────────┐     Fiber Channel     ┌──────────────┐
│ Server A  │ ←─────────────────→  │              │
│ (HBA)    │                      │  SAN Storage   │
└──────────┘     FC Switch         │  Array        │
                  ┌────┐           │              │
┌──────────┐     │ FC │           │  LUN 0       │
│ Server B  │ ←──┤Swit│──────→    │  LUN 1       │
│ (HBA)    │     │ ch │           │  LUN 2       │
└──────────┘     └────┘           │              │
                                  └──────────────┘
```

#### SAN Key Concepts

| Term | Meaning |
|------|---------|
| **HBA** (Host Bus Adapter) | Fiber channel card in the server (like a NIC, but for storage) |
| **LUN** (Logical Unit Number) | A logical disk unit carved from the storage array |
| **Zoning** | Controls which HBAs can see which LUNs (like VLANs for storage) |
| **FC (Fiber Channel)** | Dedicated storage network protocol/physical layer |
| **WWPN** | World Wide Port Name — globally unique address for a fiber port |

> **Core characteristic: Block-level storage** — a SAN LUN appears to Windows as a "local disk" (even though it's physically remote). In Disk Management, it looks identical to a local drive. Windows creates partitions, formats NTFS, completely unaware the data is actually in a datacenter miles away.

---

### 3.3 NAS (Network Attached Storage)

> **NAS = file-level storage shared over a standard network**

```
┌──────────┐                      ┌──────────────┐
│ Windows  │  SMB/CIFS Protocol    │              │
│ Client   │ ──────────────────→  │  NAS Device   │
│          │  \\NAS01\ShareName   │              │
│ Mapped   │                      │  File system   │
│ as Z:    │                      │  managed by    │
└──────────┘                      │  NAS itself    │
                                  └──────────────┘
```

- Accessed via **SMB/CIFS** protocol (Windows) or **NFS** protocol (Linux)
- Uses **UNC paths**: `\\ServerName\ShareName\folder\file.txt`
- **File-level storage** — the NAS device manages its own file system; clients only see files and folders

---

### 3.4 iSCSI

> **🎯 Analogy:** iSCSI is like putting a **letter (SCSI command) inside a regular envelope (TCP/IP packet)** — it uses the existing **postal system (Ethernet)** instead of a **special courier service (Fiber Channel)**. The result is the same — block-level storage commands reach their destination — but the transport is cheaper and more widely available.

```
iSCSI Encapsulation:

┌─────────────────────────────────┐
│  Ethernet Frame                  │
│  ┌───────────────────────────┐  │
│  │  TCP/IP Packet             │  │
│  │  ┌─────────────────────┐  │  │
│  │  │  iSCSI PDU          │  │  │
│  │  │  ┌───────────────┐  │  │  │
│  │  │  │ SCSI Cmd/Data  │  │  │  │
│  │  │  └───────────────┘  │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

#### iSCSI Key Terms

| Term | Meaning |
|------|---------|
| **Initiator** | The client (Windows has a built-in iSCSI Initiator) |
| **Target** | The storage side, providing disks |
| **IQN** (iSCSI Qualified Name) | Unique identifier, format: `iqn.yyyy-mm.com.vendor:target-name` |
| **Portal** | Target's IP:Port (default port 3260) |

#### iSCSI vs Fiber Channel

| Comparison | iSCSI | Fiber Channel |
|-----------|-------|---------------|
| Network type | Standard Ethernet | Dedicated fiber network |
| Cost | Low (reuses existing infrastructure) | High (dedicated HBAs + switches) |
| Performance | Good (10/25 GbE approaches FC) | Excellent (16/32 Gbps FC) |
| Management complexity | Lower | Higher (requires Zoning, etc.) |
| Typical use case | SMB, test environments | Large enterprise, mission-critical |

---

### 3.5 File-Level vs Block-Level Storage Comparison

| Dimension | File-Level (NAS) | Block-Level (SAN / iSCSI / DAS) |
|-----------|-----------------|--------------------------------|
| Protocol | SMB, NFS | FC, iSCSI, SAS |
| Appears in Windows as | Network share (UNC path) | Local disk (drive letter) |
| File system | Managed by NAS device | Managed by Windows (NTFS/ReFS) |
| Typical use | File sharing, home drives | Databases, Hyper-V, Exchange |
| Performance | Subject to protocol overhead | Closer to native disk performance |
| Flexibility | Plug-and-play | Requires more planning/configuration |

```
File-Level vs Block-Level — At a Glance:

File-Level (NAS):
  App → "Read \\NAS\share\report.docx"
  NAS device locates the file and returns its content

Block-Level (SAN/iSCSI):
  App → NTFS → "Read LBA sector 4096"
  Windows NTFS manages which blocks contain which files
  SAN only provides raw "blocks"
```

---

## 4. Cloud & Hybrid Storage

### 4.1 Cloud Storage Types

| Type | Examples | Characteristics |
|------|----------|----------------|
| **File Sync** | OneDrive, Dropbox | Client-side sync, on-demand download |
| **Blob/Object Storage** | Azure Blob, AWS S3 | Massive unstructured data, REST API access |
| **Cloud File Shares** | Azure Files | SMB-compatible, directly mountable |
| **Cloud Block Storage** | Azure Managed Disk | Virtual hard disks for VMs |

### 4.2 Hybrid Storage

```
Hybrid Storage Architecture:

┌──────────────┐          ┌─────────────────┐
│  On-Premises  │   Sync   │   Azure Cloud   │
│  Server       │ ←─────→ │                 │
│              │          │                 │
│  Hot Data    │          │  Cold Data /     │
│ (frequently  │          │  Archive        │
│  accessed)   │ Tiering  │  (rarely        │
│  SSD / NVMe  │ ───────→ │  accessed)      │
└──────────────┘          │  Blob Storage   │
                          └─────────────────┘
```

- **Azure StorSimple:** Automatically tiers data between local SSD and cloud
- **Azure File Sync:** Syncs on-premises file servers with Azure Files, keeping a local cache
- **Core idea:** Hot data stays local (fast), cold data goes to cloud (cheap) — balancing cost and performance

---

## 5. Bus Types Comparison Table

| Type | Bus / Interface | Typical Latency | Typical IOPS | Typical Bandwidth | Use Case |
|------|----------------|-----------------|-------------|-------------------|----------|
| **HDD** | SAS / SATA | 5-15 ms | 75-200 | 100-250 MB/s | Capacity, cold data, backup |
| **SSD** | SAS / SATA | 50-100 μs | 20K-100K | 500-600 MB/s | General workloads |
| **NVMe** | PCIe | ~10-20 μs | 100K-1M+ | 3-7 GB/s | High performance, databases |
| **SCM/PMEM** | Memory Bus | hundreds of ns - μs | millions+ | 10+ GB/s | Extreme performance, in-memory DB |

> **Speed difference visualized:**
> ```
> If an HDD read takes 1 day (representing 10 ms)...
>   SSD  = ~14 minutes  (100 μs)
>   NVMe = ~3 minutes   (20 μs)
>   PMEM = ~9 seconds   (1 μs)
>   DRAM = ~1 second    (100 ns)
> ```

---

## 6. Key Terms Reference (Glossary)

| Term | Full Name | Explanation |
|------|-----------|-------------|
| **HDD** | Hard Disk Drive | Mechanical drive using spinning magnetic platters |
| **SSD** | Solid State Drive | Drive using flash memory chips, no moving parts |
| **NVMe** | Non-Volatile Memory Express | PCIe-based storage protocol |
| **PMEM** | Persistent Memory | Memory-bus-attached, non-volatile storage |
| **SCM** | Storage Class Memory | Synonym for PMEM |
| **CHS** | Cylinder/Head/Sector | Legacy physical disk addressing scheme |
| **LBA** | Logical Block Addressing | Modern logical addressing standard |
| **RPM** | Revolutions Per Minute | Measures HDD spindle rotation speed |
| **NAND** | Not AND (flash type) | Flash memory technology used in SSDs |
| **ECC** | Error Correction Code | Detects/corrects bit errors |
| **DAS** | Direct Attached Storage | Storage physically attached to a server |
| **SAN** | Storage Area Network | Block-level storage over a dedicated network |
| **NAS** | Network Attached Storage | File-level storage over a standard network |
| **iSCSI** | Internet SCSI | SCSI protocol encapsulated in TCP/IP |
| **FC** | Fiber Channel | Dedicated storage network protocol |
| **HBA** | Host Bus Adapter | Fiber channel card in a server |
| **LUN** | Logical Unit Number | A logical disk carved from a storage array |
| **WWPN** | World Wide Port Name | Globally unique fiber port identifier |
| **IQN** | iSCSI Qualified Name | Unique iSCSI endpoint identifier |
| **JBOD** | Just a Bunch of Disks | Disks presented individually without RAID |
| **SMB** | Server Message Block | Windows file sharing protocol |
| **NFS** | Network File System | Linux/Unix file sharing protocol |
| **IOPS** | I/O Operations Per Second | Measures storage throughput in operations |
| **NVMe-oF** | NVMe over Fabrics | Protocol for remote NVMe access |
| **RDMA** | Remote Direct Memory Access | Low-latency network data transfer |

---

> **📌 Next in this series:** Deep Dive into RAID levels, Storage Spaces, and Windows disk management.
