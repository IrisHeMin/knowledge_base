---
layout: post
title: "Deep Dive: Windows 存储栈架构 — 从应用到硬件的 I/O 之旅"
date: 2026-03-17
categories: [Knowledge, Storage]
tags: [storage, storage-stack, storport, disk-sys, partmgr, ntfs, refs, mbr, gpt, irp, srb, file-system, volume-manager]
type: "deep-dive"
---

# Deep Dive: Windows 存储栈架构 — 从应用到硬件的 I/O 之旅

**Topic:** Windows Storage Stack Architecture
**Category:** Storage / I/O
**Level:** Senior / 中级
**Last Updated:** 2026-03-17

---

## 1. 概述 (Overview)

Windows 存储栈采用**模块化、分层架构**：每一层只负责一个明确的职责，通过标准接口与上下层通信。这种设计使得任何一层都可以被替换或扩展，而不影响其他层——例如，你可以换一块 NVMe SSD 替换 SATA HDD，上层的 NTFS 文件系统完全不需要改变。

### 类比：邮政分拣系统 📮

把整个存储栈想象成一栋**邮政分拣大楼**：

- 一封信（一个 I/O 请求）从顶楼投入
- 每一层楼都有专门的工作人员：有人负责贴邮编（文件系统映射逻辑地址）、有人负责分拣到哪个区（分区管理器）、有人负责装车（端口驱动）
- 最终信件被送到目的地（磁盘硬件）
- 回程（读操作返回数据）则是反方向，逐层上传直到应用程序拿到结果

每一层只关心自己的工作，不需要知道信件的完整旅程。这就是 **分层解耦** 的精髓。

> **Why it matters for support:** 当 I/O 出现问题（慢、卡、报错），你需要知道**问题卡在哪一层**。理解存储栈的分层结构，就是掌握了排查存储问题的地图。

---

The Windows storage stack uses a **modular, layered architecture**: each layer has a single clear responsibility and communicates with adjacent layers through standard interfaces. This design allows any layer to be replaced or extended without affecting others — for example, you can swap an NVMe SSD for a SATA HDD, and the NTFS file system above doesn't need to change at all.

### Analogy: Postal Sorting System 📮

Think of the entire storage stack as a **postal sorting building**:

- A letter (an I/O request) is dropped in at the top floor
- Each floor has specialized workers: someone stamps the postal code (file system maps logical addresses), someone sorts it to the right district (partition manager), someone loads it onto the truck (port driver)
- The letter finally reaches its destination (disk hardware)
- The return trip (read data coming back) goes in reverse, passing up through each floor until the application gets its result

Each floor only cares about its own job and doesn't need to know the letter's complete journey. This is the essence of **layered decoupling**.

> **Why it matters for support:** When I/O goes wrong (slow, hung, errors), you need to know **which layer the problem is stuck at**. Understanding the layered structure of the storage stack is like having a map for troubleshooting storage issues.

---

## 2. 存储栈全景图 (Full Stack Diagram)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        APPLICATION                                  │
│                  (CreateFile, ReadFile, WriteFile)                   │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  Win32 API → Kernel transition
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    I/O SUBSYSTEM (I/O Manager)                      │
│              Creates IRP (I/O Request Packet)                       │
│              Routes IRP down the driver stack                       │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  IRP
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   FILE SYSTEM DRIVER                                │
│            NTFS.sys │ ReFS.sys │ FAT │ exFAT │ CSVFS              │
│       (Logical → Physical block mapping, metadata, caching)        │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  IRP (with LBN offsets)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│              VOLUME SNAPSHOT (VSS) — volsnap.sys                    │
│               Shadow Copies / Snapshots for backup                  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  IRP
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│              VOLUME MANAGER — DMIO.sys / volmgr.sys                 │
│          Volume ↔ Partition mapping, Spanned/Striped/Mirror         │
│                    LDM Database management                          │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  IRP
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│            PARTITION MANAGER — partmgr.sys                          │
│             MBR / GPT parsing, partition enumeration                │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  IRP
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│               CLASS DRIVER — disk.sys                               │
│     Standardizes all disk types into uniform interface              │
│            IRP → SRB (SCSI Request Block) conversion                │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  SRB (SCSI commands)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│            PORT DRIVER — storport.sys / ataport.sys                 │
│     ┌───────────────────────────────────────────────┐              │
│     │  MINIPORT DRIVER (vendor-specific)             │              │
│     │  stornvme.sys │ USBStor.sys │ iScsiPrt.sys    │              │
│     └───────────────────────────────────────────────┘              │
│         Translates SRB → hardware-specific commands                 │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  Hardware bus commands
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    DISK SUBSYSTEM (Hardware)                         │
│    HBA/Controller → Cable/Bus → Physical Disk (SSD/HDD/NVMe)       │
│    ┌─────────────────────────────────────────────────────┐         │
│    │  Special: iSCSI → Network Stack (NDIS) → TCP/IP     │         │
│    └─────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────┘
```

### 关键要点 / Key Takeaways

| 方向 | 数据格式 | 说明 |
|------|----------|------|
| **向下 ↓** (Write/Read request) | IRP → IRP → ... → SRB → HW cmd | IRP 在文件系统到 disk.sys 之间传递；disk.sys 将 IRP 转换为 SRB |
| **向上 ↑** (Completion) | HW response → SRB completion → IRP completion | 完成回调逐层向上，直到 I/O Manager 通知应用 |

---

## 3. 各层详解 (Layer-by-Layer Deep Dive)

### 3.1 磁盘子系统 — Disk Subsystem (Hardware)

这是存储栈的**物理底层**——真正存储数据的硬件。

| 组件 | 说明 |
|------|------|
| **HBA / Controller** | Host Bus Adapter，主机到存储的桥梁。SAS HBA、FC HBA、RAID Controller 等 |
| **Bus / Cable** | SATA cable、SAS cable、FC光纤、PCIe (NVMe) |
| **Physical Disk** | HDD（机械盘）、SSD（固态盘）、NVMe SSD |
| **Storage Array** | SAN 设备（Dell EMC、NetApp、Pure Storage 等），通过 FC 或 iSCSI 连接 |

#### iSCSI 的特殊性

iSCSI 是一个**跨越存储栈和网络栈的混合体**：

```
  disk.sys → iScsiPrt.sys (miniport) → TCP/IP Stack (tcpip.sys) → NDIS → NIC → Network → iSCSI Target
```

- iSCSI 将 SCSI 命令封装在 TCP/IP 数据包中通过以太网传输
- 因此 iSCSI 的性能和稳定性同时受到**存储栈**和**网络栈**的影响
- 排查 iSCSI 问题时，需要**同时检查存储日志和网络日志**

> **Support Tip:** iSCSI 问题是存储和网络的交叉地带。网络丢包、延迟、MTU 不匹配都可能导致"存储"问题。

---

This is the **physical bottom** of the storage stack — the hardware that actually stores data.

#### iSCSI Special Case

iSCSI is a **hybrid that crosses both the storage stack and the network stack**:

- iSCSI encapsulates SCSI commands inside TCP/IP packets transmitted over Ethernet
- Therefore iSCSI performance and stability are affected by **both the storage stack and the network stack**
- When troubleshooting iSCSI issues, you need to **check both storage logs and network logs simultaneously**

> **Support Tip:** iSCSI issues live at the intersection of storage and networking. Packet loss, latency, or MTU mismatches can all manifest as "storage" problems.

---

### 3.2 端口驱动 / 微端口驱动 — Port Driver / Miniport Driver

这是 Windows 能控制的**最底层**，也是存储诊断中的**黄金层**。

#### 端口驱动 (Port Driver)

| 端口驱动 | 总线类型 | 说明 |
|----------|----------|------|
| **storport.sys** | SCSI / SAS / FC / iSCSI / NVMe | 现代存储的标准端口驱动，最常用 |
| **ataport.sys** | IDE / SATA (legacy) | 旧版 ATA/IDE 设备专用（已逐渐淘汰） |

端口驱动的职责：
- 管理 SRB 队列和调度
- 提供 DMA（直接内存访问）支持
- 处理超时、重试、错误恢复
- 提供 **ETW 日志**（storport trace）— 这是诊断存储性能问题的关键数据源

#### 微端口驱动 (Miniport Driver)

微端口由**硬件厂商**开发，负责与特定硬件通信：

| 微端口驱动 | 用途 |
|------------|------|
| **stornvme.sys** | Windows 内置的 NVMe 微端口驱动 |
| **USBStor.sys** | USB 大容量存储设备 |
| **iScsiPrt.sys** | Microsoft iSCSI Initiator |
| **Vendor HBA drivers** | LSI/Broadcom MegaRAID、QLogic FC、Emulex FC 等 |

#### 类比：邮局分拣机 + 社区邮递员 📬

- **Port Driver (storport.sys)** = 邮局的**自动分拣机**：它不知道每个社区的门牌号，但它管理整个分拣流程——排队、调度、处理退信
- **Miniport Driver** = **社区邮递员**：他知道每家每户的具体位置，负责最后一公里的投递

> **为什么 storport 日志是诊断金矿？** 因为 storport.sys 是 Windows 能控制的最低层——它能看到每个 SRB 的提交时间、完成时间、超时和错误。如果一个 I/O 在 storport 层就超时了，问题一定在 storport 以下（硬件、线缆、SAN）。

#### Dump 驱动 (Crash Dump Drivers)

当系统蓝屏时，正常的驱动栈可能已经不可用。Windows 使用**精简版的 dump 驱动**来写入 memory dump：

- **DumpStorPort.sys** — storport 的 dump 版本
- **Dump_stornvme.sys** — NVMe 的 dump 版本
- 这些驱动在 crash 上下文中运行，功能极其精简

> **Support Tip:** 如果 crash dump 不完整或为 0 字节，排查方向应该是 dump filter 驱动或 dump 存储路径的问题。

---

This is the **lowest layer Windows can control** and the **gold mine** for storage diagnostics.

#### Analogy: Post Office Sorting Machine + Neighborhood Mailman 📬

- **Port Driver (storport.sys)** = the post office **automated sorting machine**: it doesn't know every house number in every neighborhood, but it manages the entire sorting process — queuing, scheduling, handling return-to-sender
- **Miniport Driver** = the **neighborhood mailman**: he knows the exact location of every house and handles last-mile delivery

> **Why are storport logs diagnostic gold?** Because storport.sys is the lowest layer Windows controls — it can see every SRB's submission time, completion time, timeouts, and errors. If an I/O times out at the storport layer, the problem is definitely **below** storport (hardware, cables, SAN).

#### Dump Drivers

During a bluescreen, the normal driver stack may be unavailable. Windows uses **stripped-down dump drivers** to write memory dumps:

- **DumpStorPort.sys** — dump version of storport
- **Dump_stornvme.sys** — dump version for NVMe
- These drivers run in crash context with extremely minimal functionality

---

### 3.3 类驱动 — Class Driver (disk.sys)

disk.sys 是存储栈中的**标准化层**——它的工作是让所有磁盘设备对上层看起来都一样。

#### 核心职责

1. **抽象化 (Abstraction)**：无论底层是 NVMe SSD、SATA HDD、USB 闪存盘还是 iSCSI LUN，disk.sys 都对上层暴露一个统一的**磁盘设备对象 (Disk Device Object)**
2. **IRP → SRB 转换**：这是关键转换点。上层传来的 IRP（I/O Request Packet）在这里被转换为 SRB（SCSI Request Block），然后发给 Port Driver
3. **错误处理**：处理磁盘级别的错误和重试
4. **SMART / Health Monitoring**：暴露磁盘健康信息给上层

#### 数据格式转换点

```
  上层 (File System, Volume Manager, etc.)
       │
       │  IRP (I/O Request Packet)
       │  - 包含文件偏移、字节数等高层信息
       ▼
  ┌─────────────┐
  │   disk.sys   │  ← IRP → SRB 转换发生在这里
  └─────────────┘
       │
       │  SRB (SCSI Request Block)
       │  - 包含 SCSI CDB (Command Descriptor Block)
       │  - LBA (Logical Block Address)
       │  - Transfer length
       ▼
  Port/Miniport Driver
```

> **类比**：disk.sys 就像机场的**值机柜台**——无论你是乘坐国内航班还是国际航班、经济舱还是头等舱，你拿到的登机牌（SRB）格式都是一样的。航空公司（port driver）只需要看登机牌就能处理。

---

disk.sys is the **standardization layer** in the storage stack — its job is to make all disk devices look the same to upper layers.

#### Core Responsibilities

1. **Abstraction**: Whether the underlying device is an NVMe SSD, SATA HDD, USB flash drive, or iSCSI LUN, disk.sys exposes a uniform **Disk Device Object** to upper layers
2. **IRP → SRB Conversion**: This is the critical conversion point. IRPs from upper layers are converted to SRBs (SCSI Request Blocks) here, then sent to the Port Driver
3. **Error Handling**: Handles disk-level errors and retries
4. **SMART / Health Monitoring**: Exposes disk health information to upper layers

> **Analogy**: disk.sys is like the **check-in counter** at an airport — no matter if you're on a domestic or international flight, economy or first class, the boarding pass (SRB) you receive has the same format. The airline (port driver) only needs to look at the boarding pass to process you.

---

### 3.4 分区管理器 — Partition Manager (partmgr.sys)

partmgr.sys 负责读取和管理磁盘上的**分区表**，将一块物理磁盘划分为多个逻辑分区。

#### MBR vs GPT 对比

| 特性 | MBR (Master Boot Record) | GPT (GUID Partition Table) |
|------|--------------------------|---------------------------|
| **位置** | 磁盘第 0 扇区 (Sector 0, LBA 0) | 磁盘开头和结尾各一份（Primary + Backup） |
| **主分区数量** | 最多 **4 个**主分区（或 3 个主分区 + 1 个扩展分区） | 最多 **128 个**分区（Windows 默认） |
| **最大磁盘容量** | **2 TiB**（2³² × 512 bytes）— 注意：常被误说为 7.8 GiB，实际是 2 TiB | **18 EB (Exabytes)**（2⁶⁴ × 512 bytes，理论值）；实际 Windows 限制约 **256 TiB** |
| **冗余** | 无冗余，分区表损坏即丢失 | Primary GPT + Backup GPT（磁盘末尾），双份冗余 |
| **校验** | 无 CRC 校验 | CRC32 校验保护分区表完整性 |
| **UEFI / Secure Boot** | ❌ 不支持 | ✅ 必须使用 GPT |
| **Legacy BIOS** | ✅ 支持 | ❌ 不支持（除非 Protective MBR 兼容模式） |
| **操作系统支持** | 所有 Windows 版本 | Windows Server 2003 SP1+ (data), Vista+ (boot) |

#### MBR 结构

```
  Sector 0 (512 bytes)
  ┌──────────────────────────┐
  │  Boot Code (446 bytes)    │  ← BIOS 引导代码
  ├──────────────────────────┤
  │  Partition Entry 1 (16B)  │  ← 分区 1
  │  Partition Entry 2 (16B)  │  ← 分区 2
  │  Partition Entry 3 (16B)  │  ← 分区 3
  │  Partition Entry 4 (16B)  │  ← 分区 4
  ├──────────────────────────┤
  │  Signature: 0x55AA (2B)   │  ← MBR 签名
  └──────────────────────────┘
```

#### GPT 结构

```
  LBA 0:   Protective MBR         ← 防止旧工具误操作
  LBA 1:   Primary GPT Header     ← GPT 头，含分区数量、CRC 等
  LBA 2-33: Partition Entry Array  ← 128 个分区条目（每个 128 bytes）
  ...
  LBA N-33 to N-2: Backup Partition Entry Array
  LBA N-1: Backup GPT Header      ← 备份 GPT 头
```

#### 分区 = 空盒子

分区只是磁盘上的一段**连续扇区范围**——它定义了起始位置和大小，但里面什么都没有（没有文件系统、没有数据）。就像买了一块空地，只画了**地界线**。

**关键限制：分区不能跨磁盘 (Partitions CANNOT span disks)。** 一个分区只能存在于一块物理磁盘上。如果需要跨磁盘，你需要上一层的**卷管理器 (Volume Manager)**。

#### 类比：停车场的画线区域 🅿️

- **磁盘** = 一整块空地
- **分区** = 用油漆画出的停车位区域
- MBR = 只能画 4 个大区域
- GPT = 可以画 128 个区域，还有备用图纸
- 画好的区域（分区）里面是空的——你还需要铺设路面、安装标识（格式化文件系统）才能使用
- **你不能把一个停车位画到两块不相邻的空地上** — 分区不能跨磁盘

---

partmgr.sys is responsible for reading and managing **partition tables** on disks, dividing a physical disk into multiple logical partitions.

#### MBR vs GPT Comparison

| Feature | MBR | GPT |
|---------|-----|-----|
| **Location** | Sector 0 (LBA 0) | Start and end of disk (Primary + Backup) |
| **Max Partitions** | 4 primary (or 3 primary + 1 extended) | 128 (Windows default) |
| **Max Disk Size** | 2 TiB | ~256 TiB (Windows practical limit) |
| **Redundancy** | None | Dual copy (Primary + Backup GPT) |
| **Integrity Check** | None | CRC32 |
| **UEFI / Secure Boot** | ❌ Not supported | ✅ Required |

#### Partitions = Empty Boxes

A partition is just a **contiguous range of sectors** on a disk — it defines a starting position and size, but is empty inside (no file system, no data). It's like buying a plot of land and only drawing **boundary lines**.

**Key limitation: Partitions CANNOT span disks.** A partition can only exist on a single physical disk. If you need to span disks, you need the **Volume Manager** layer above.

#### Analogy: Parking Lot Sections 🅿️

- **Disk** = an entire empty lot
- **Partitions** = painted parking sections
- MBR = can only paint 4 big sections
- GPT = can paint 128 sections, with backup blueprints
- The painted sections are empty — you still need to pave the surface and install signs (format with a file system) before they're usable
- **You can't paint a single parking section across two separate lots** — partitions cannot span disks

---

### 3.5 卷管理器 — Volume Manager (DMIO.sys)

卷管理器在分区之上构建**卷 (Volume)**——卷是文件系统的容器，可以比分区更灵活。

#### 核心特性

| 特性 | 说明 |
|------|------|
| **跨磁盘** | ✅ 卷**可以跨多个磁盘**（Spanned Volume） |
| **RAID 功能** | 条带卷 (Striped)、镜像卷 (Mirrored)、RAID-5 卷 |
| **动态磁盘** | 通过 LDM (Logical Disk Manager) 数据库管理动态磁盘配置 |
| **简单卷** | Simple Volume = 1 个分区 → 1 个卷（最常见的配置） |

#### LDM Database

- 存储在动态磁盘末尾的 **1 MB** 区域
- 记录所有卷的布局信息（哪些分区组成哪个卷）
- 每个动态磁盘都有一份 LDM 数据库副本
- **注意**：LDM 是旧方案，Windows Server 2012+ 推荐使用 **Storage Spaces** 替代

#### 分区 vs 卷

```
  ┌─ Disk 0 ──────────┐   ┌─ Disk 1 ──────────┐
  │ Partition A        │   │ Partition B        │
  │   ↓                │   │   ↓                │
  │ Volume C: (simple) │   │ ┌──────────────────┤
  │                    │   │ │                  │
  └────────────────────┘   │ │  Volume D:       │
                           │ │  (spanned across │
  ┌─ Disk 2 ──────────┐   │ │   Disk 1 + Disk 2│
  │ Partition C        │   │ │                  │
  │   ↓                │   └─┤                  │
  │ ┌──────────────────┤     │                  │
  │ │  Volume D: cont. │     └──────────────────┘
  │ └──────────────────┤
  └────────────────────┘
```

> **类比**：如果分区是停车场的画线区域，那么卷就是在这些区域上建造的**立体停车楼**——它可以跨越多个停车位区域（跨分区），甚至跨越多个停车场（跨磁盘）。卷管理器就是这栋楼的**建筑工程师**。

---

The Volume Manager builds **volumes** on top of partitions — volumes are containers for file systems and can be more flexible than partitions.

#### Partition vs Volume — Key Difference

| | Partition | Volume |
|---|-----------|--------|
| **Span disks?** | ❌ Cannot | ✅ Can |
| **Who manages?** | partmgr.sys | DMIO.sys / volmgr.sys |
| **Contains** | Nothing (empty box) | File system data |
| **RAID** | ❌ | ✅ Stripe, Mirror, RAID-5 |

> **Analogy**: If partitions are painted sections in a parking lot, then volumes are **multi-story parking structures** built on those sections — they can span across multiple painted areas (partitions) and even across multiple parking lots (disks). The Volume Manager is the **architect** of these structures.

---

### 3.6 卷影复制 — Volume Shadow Copy Service (VSS)

VSS (volsnap.sys) 在卷管理器之上提供**时间点快照**能力。

#### 核心概念

- **Shadow Copy**：卷在某一时刻的只读快照
- **Copy-on-Write**：当原始数据块被修改时，先把旧数据复制到 shadow copy 存储区域，然后才写入新数据
- **VSS Writer**：应用程序（如 SQL Server、Exchange、Hyper-V）注册的组件，确保快照时数据处于一致状态
- **VSS Provider**：实际执行快照操作的组件（软件或硬件）

#### VSS 在存储栈中的位置

VSS 位于文件系统和卷管理器之间，拦截写 I/O：

```
  File System (NTFS)
       │
       │  Write IRP
       ▼
  volsnap.sys (VSS)
       │
       ├──→ 如果该块有 shadow copy 且未备份
       │    → 先 Copy-on-Write 旧数据到快照区域
       │
       │  Write IRP (继续)
       ▼
  Volume Manager
```

> **Support Tip:** VSS shadow copy 会消耗额外的 I/O 和存储空间。如果 shadow copy 存储区域 (diff area) 满了，最旧的 shadow copy 会被自动删除。

---

VSS (volsnap.sys) provides **point-in-time snapshot** capability above the Volume Manager.

- **Shadow Copy**: A read-only snapshot of a volume at a specific moment
- **Copy-on-Write**: When an original data block is modified, the old data is first copied to shadow storage before new data is written
- **VSS Writer**: Components registered by applications (SQL Server, Exchange, Hyper-V) to ensure data consistency during snapshots
- **VSS Provider**: The component that actually performs the snapshot (software or hardware)

> **Support Tip:** VSS shadow copies consume additional I/O and storage space. If the diff area fills up, the oldest shadow copies are automatically deleted.

---

### 3.7 文件系统 — File Systems

文件系统是存储栈中用户最"可见"的一层——它把卷上的原始数据块组织成**文件和目录**。

#### NTFS (New Technology File System)

Windows 的主力文件系统，自 Windows NT 3.1 (1993) 起一直演进至今。

| 特性 | 值 |
|------|-----|
| **最大卷大小** | **256 TiB**（Windows Server 2016+，64KB 簇大小） |
| **最大文件大小** | 256 TiB - 64 KB（受卷大小限制） |
| **文件名长度** | 255 个 Unicode 字符 |
| **路径长度** | MAX_PATH = 260 字符 ← ⚠️ **这不是 NTFS 的限制！这是 Win32 API 的限制** |
| **NTFS 实际路径限制** | **32,767 个 Unicode 字符**（通过 `\\?\` 前缀或启用 LongPathsEnabled） |
| **日志** | ✅ 事务日志 (Transactional NTFS journal, $LogFile) |
| **ACL** | ✅ 完整的 NTFS 权限和审计 |
| **压缩** | ✅ 文件/文件夹级别压缩 |
| **加密** | ✅ EFS (Encrypting File System) |
| **硬链接/符号链接** | ✅ |
| **配额** | ✅ 磁盘配额 |

##### MFT (Master File Table)

MFT 是 NTFS 的核心数据结构——**卷上每个文件和目录**都有一个 MFT 记录（通常 1 KB）。

```
  MFT Record Layout:
  ┌────────────────────────────────┐
  │  Standard Information          │  ← 创建/修改时间、权限等
  │  File Name                     │  ← 文件名（可能有多个：短名+长名）
  │  Data (or Data Run List)       │  ← 小文件：数据直接在 MFT 中
  │                                │     大文件：指向数据块的指针列表
  │  Security Descriptor           │  ← ACL
  │  ...                           │
  └────────────────────────────────┘
```

> **MAX_PATH 260 的真相**：很多人以为 260 字符是 NTFS 的限制——实际上这是 Win32 API 中 `MAX_PATH` 常量的定义 (`#define MAX_PATH 260`)。NTFS 本身支持长达 32,767 字符的路径。可以通过使用 `\\?\` 前缀的路径或在 Windows 10 1607+ 中启用 `LongPathsEnabled` 注册表项来突破此限制。

#### ReFS (Resilient File System)

微软为现代存储需求设计的下一代文件系统。

| 特性 | 值 |
|------|-----|
| **最大卷大小** | **4.7 ZB (Zettabytes)**（理论值） |
| **最大文件大小** | 18 EB（理论值） |
| **Integrity Streams** | ✅ 自动检测和修复数据损坏（需要 Storage Spaces Mirror） |
| **Block Cloning** | ✅ 快速复制大文件（Hyper-V checkpoint 场景） |
| **Sparse VDL** | ✅ 快速创建大文件而无需写入零（VM 创建场景） |
| **Data Deduplication** | ✅（Windows Server 2019+） |
| **不支持** | ❌ 压缩、EFS 加密、磁盘配额、硬链接、短文件名 (8.3)、引导分区（不能从 ReFS 启动） |

##### Integrity Streams

ReFS 的杀手级特性——每个数据块和元数据块都有校验和：

```
  写入时：Data Block → 计算 CRC → 存储 CRC
  读取时：Data Block → 计算 CRC → 与存储的 CRC 比较
                                    │
                          ┌─────────┴─────────┐
                          │ 匹配 → 返回数据     │
                          │ 不匹配 → 从镜像副本  │
                          │   读取并自动修复      │
                          └───────────────────┘
```

#### CSVFS (Cluster Shared Volume File System)

**CSVFS 不是一个真正的文件系统**——它是 Failover Cluster 中 CSV (Cluster Shared Volumes) 的**协调层**。

- CSVFS 在实际文件系统（NTFS 或 ReFS）之上运行
- 它的作用是协调多个集群节点对同一卷的并发访问
- **Coordinator Node**：拥有该 CSV 卷的 NTFS/ReFS 直接访问权
- **其他节点**：通过 SMB 3.0 重定向 I/O 到 Coordinator Node（Direct I/O 模式下除外）

#### FAT / exFAT (Legacy)

| 特性 | FAT32 | exFAT |
|------|-------|-------|
| **最大文件大小** | 4 GiB - 1 byte | 16 EB（理论） |
| **最大卷大小** | 2 TiB | 128 PiB（理论） |
| **日志** | ❌ | ❌ |
| **ACL** | ❌ | ❌ |
| **典型用途** | USB 闪存盘、SD 卡（兼容性） | 大容量 USB 设备、SD 卡 |

#### 文件系统对比总结表

| 特性 | NTFS | ReFS | FAT32 | exFAT |
|------|------|------|-------|-------|
| **最大卷** | 256 TiB | 4.7 ZB | 2 TiB | 128 PiB |
| **最大文件** | 256 TiB | 18 EB | 4 GiB | 16 EB |
| **日志/事务** | ✅ | ✅ | ❌ | ❌ |
| **ACL/权限** | ✅ | ✅ | ❌ | ❌ |
| **数据完整性** | ❌ | ✅ Integrity Streams | ❌ | ❌ |
| **压缩** | ✅ | ❌ | ❌ | ❌ |
| **可引导** | ✅ | ❌ | ✅ (BIOS) | ❌ |
| **推荐场景** | 通用 Windows | Hyper-V, S2D, 大规模存储 | 小型可移动设备 | 大型可移动设备 |

---

File systems are the most "visible" layer of the storage stack to users — they organize raw data blocks on a volume into **files and directories**.

#### NTFS Key Points

- Max volume: **256 TiB**; Max path: **32,767 chars** (NTFS native) vs **260 chars** (Win32 `MAX_PATH`)
- **MAX_PATH 260 is NOT an NTFS limit** — it's a Win32 API constant. NTFS supports paths up to 32,767 characters via `\\?\` prefix or the `LongPathsEnabled` registry key (Windows 10 1607+)
- MFT (Master File Table): core data structure where every file/directory has a record

#### ReFS Key Points

- Max volume: **4.7 ZB** (theoretical); designed for resilience and scale
- **Integrity Streams**: automatic CRC verification on every data and metadata block — with Storage Spaces Mirror, corrupted blocks are auto-repaired from the mirror copy
- Cannot be used as a boot volume

#### CSVFS

- **Not a real file system** — it's a coordination layer for Cluster Shared Volumes in Failover Clustering
- Runs on top of NTFS or ReFS
- Coordinates concurrent multi-node access to the same volume

#### File System Comparison Summary

| Feature | NTFS | ReFS | FAT32 | exFAT |
|---------|------|------|-------|-------|
| **Max Volume** | 256 TiB | 4.7 ZB | 2 TiB | 128 PiB |
| **Max File** | 256 TiB | 18 EB | 4 GiB | 16 EB |
| **Journal** | ✅ | ✅ | ❌ | ❌ |
| **ACL** | ✅ | ✅ | ❌ | ❌ |
| **Data Integrity** | ❌ | ✅ | ❌ | ❌ |
| **Bootable** | ✅ | ❌ | ✅ (BIOS) | ❌ |
| **Best for** | General Windows | Hyper-V, S2D | Small removable | Large removable |

---

### 3.8 I/O 子系统 — I/O Subsystem (I/O Manager)

I/O 子系统是存储栈的**入口**——它是内核中负责接收用户态 I/O 请求并将其转化为内核对象的组件。

#### IRP (I/O Request Packet)

IRP 是 Windows 内核中 I/O 操作的**核心数据结构**：

```
  IRP Structure:
  ┌─────────────────────────────────┐
  │  IRP Header                      │
  │  - MdlAddress (Memory Descriptor │
  │    List for buffer)              │
  │  - IoStatus (completion status)  │
  │  - RequestorMode (User/Kernel)   │
  ├─────────────────────────────────┤
  │  I/O Stack Location [0]          │  ← File System 的参数
  │  - MajorFunction: IRP_MJ_READ   │
  │  - Parameters.Read.ByteOffset   │
  │  - Parameters.Read.Length        │
  ├─────────────────────────────────┤
  │  I/O Stack Location [1]          │  ← Volume Manager 的参数
  ├─────────────────────────────────┤
  │  I/O Stack Location [2]          │  ← Class Driver 的参数
  ├─────────────────────────────────┤
  │  I/O Stack Location [N]          │  ← Port Driver 的参数
  └─────────────────────────────────┘
```

#### IRP 的生命周期

1. **创建**：I/O Manager 根据用户的 `ReadFile()` / `WriteFile()` 调用创建 IRP
2. **向下传递**：IRP 沿着驱动栈逐层向下传递，每一层在自己的 **I/O Stack Location** 中填入处理参数
3. **转换**：在 disk.sys 层，IRP 被转换为 SRB
4. **执行**：Port/Miniport 将 SRB 发给硬件
5. **完成**：硬件完成后，完成回调逐层向上，最终 I/O Manager 通知用户态应用

#### IRP vs SRB

| | IRP | SRB |
|---|-----|-----|
| **全称** | I/O Request Packet | SCSI Request Block |
| **使用范围** | 文件系统 → disk.sys | disk.sys → Port/Miniport |
| **包含信息** | 文件偏移、字节数、用户缓冲区 | SCSI CDB、LBA、传输长度 |
| **转换点** | — | disk.sys 将 IRP 转换为 SRB |

> **类比**：IRP 就像一个**文件夹**，从最高层一路收集每层的指示单（I/O Stack Location），到 disk.sys 这一层时，文件夹里的信息被翻译成一张标准的**工单 (SRB)**，然后交给工厂（硬件）去执行。

---

The I/O Subsystem is the **entry point** of the storage stack — it's the kernel component responsible for receiving user-mode I/O requests and converting them into kernel objects.

#### IRP (I/O Request Packet)

- The core data structure for I/O operations in the Windows kernel
- Contains a **header** (buffer info, status) and multiple **I/O Stack Locations** (one per driver in the stack)
- Each driver fills its own stack location with processing parameters as the IRP passes down

#### IRP vs SRB

| | IRP | SRB |
|---|-----|-----|
| **Full Name** | I/O Request Packet | SCSI Request Block |
| **Used Between** | File System → disk.sys | disk.sys → Port/Miniport |
| **Contains** | File offset, byte count, user buffer | SCSI CDB, LBA, transfer length |
| **Conversion** | — | disk.sys converts IRP → SRB |

> **Analogy**: An IRP is like a **folder** that collects instruction slips (I/O Stack Locations) from each floor as it travels down. At the disk.sys floor, all the information in the folder is translated into a standard **work order (SRB)** and handed to the factory (hardware) for execution.

---

## 4. 完整 I/O 流程：一个读请求的旅程 (Complete I/O Flow: A Read Request Walkthrough)

让我们追踪一个完整的读请求——从应用程序调用 `ReadFile()` 到数据返回——经过存储栈的每一层。

### 场景：应用读取文件 C:\Data\report.txt 的前 4096 字节

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  Step 1: APPLICATION                                            │
  │  App calls: ReadFile(hFile, buffer, 4096, &bytesRead, NULL)     │
  │  → Win32 API → NtReadFile() syscall → Kernel mode              │
  └───────────────────────────┬─────────────────────────────────────┘
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Step 2: I/O MANAGER                                            │
  │  - Creates IRP with MajorFunction = IRP_MJ_READ                │
  │  - Allocates I/O Stack Locations (one per driver in stack)      │
  │  - Sets ByteOffset = file position, Length = 4096               │
  │  - Sends IRP to the top of the device stack (File System)       │
  └───────────────────────────┬─────────────────────────────────────┘
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Step 3: NTFS.sys (File System)                                 │
  │  - Checks cache manager: is this data already cached?           │
  │    → If YES (cache hit): return data immediately, IRP completes │
  │    → If NO (cache miss): continue ↓                             │
  │  - Looks up MFT record for report.txt                           │
  │  - Translates file offset → Volume Cluster Number (VCN → LCN)  │
  │  - File byte offset 0-4095 → Logical Cluster Number on volume  │
  │  - Passes IRP down with volume-relative byte offset             │
  └───────────────────────────┬─────────────────────────────────────┘
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Step 4: volsnap.sys (VSS)                                      │
  │  - Read request: VSS is mostly pass-through for reads           │
  │  - (For writes: would check if Copy-on-Write is needed)         │
  │  - Passes IRP down unchanged                                    │
  └───────────────────────────┬─────────────────────────────────────┘
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Step 5: DMIO.sys / volmgr.sys (Volume Manager)                 │
  │  - Maps Volume offset → Partition offset                        │
  │  - For simple volumes: trivial 1:1 mapping                      │
  │  - For spanned/striped: calculates which disk + offset          │
  │  - Passes IRP down with partition-relative offset               │
  └───────────────────────────┬─────────────────────────────────────┘
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Step 6: partmgr.sys (Partition Manager)                        │
  │  - Maps Partition offset → Disk offset                          │
  │  - Adds partition starting offset to get absolute disk offset   │
  │  - Passes IRP down with disk-relative byte offset               │
  └───────────────────────────┬─────────────────────────────────────┘
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Step 7: disk.sys (Class Driver)                                │
  │  - ★ CRITICAL CONVERSION POINT ★                               │
  │  - Converts IRP → SRB (SCSI Request Block)                     │
  │  - Creates SCSI CDB: READ(10) or READ(16) command              │
  │  - CDB contains: LBA (Logical Block Address), Transfer Length   │
  │  - Byte offset ÷ Sector size (512) = LBA                       │
  │  - Sends SRB to Port Driver                                     │
  └───────────────────────────┬─────────────────────────────────────┘
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Step 8: storport.sys (Port Driver)                             │
  │  - Queues the SRB                                               │
  │  - Manages I/O scheduling, queue depth                          │
  │  - Calls Miniport's StartIo/BuildIo routine                     │
  │  - Starts timeout timer                                         │
  │  - Miniport translates SRB → hardware-specific commands         │
  └───────────────────────────┬─────────────────────────────────────┘
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Step 9: HARDWARE                                               │
  │  - HBA/Controller sends command on the bus                      │
  │  - Disk reads data from platters/flash cells                    │
  │  - Data transferred via DMA to system memory                    │
  │  - Controller raises interrupt → Miniport ISR fires             │
  └───────────────────────────┬─────────────────────────────────────┘
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Step 10: COMPLETION (Return Trip ↑)                            │
  │                                                                  │
  │  HW → Miniport ISR → storport DPC → SRB completion callback    │
  │    → disk.sys: SRB complete → mark IRP success                  │
  │    → partmgr.sys: pass completion up                            │
  │    → volmgr.sys: pass completion up                             │
  │    → volsnap.sys: pass completion up                            │
  │    → NTFS.sys: cache the data, complete IRP                     │
  │    → I/O Manager: copy data to user buffer                      │
  │    → App: ReadFile() returns, bytesRead = 4096                  │
  └─────────────────────────────────────────────────────────────────┘
```

### 地址转换总结

```
  File offset (字节)
       │
       │  NTFS: VCN → LCN mapping
       ▼
  Volume offset (字节)
       │
       │  Volume Manager: Volume → Partition mapping
       ▼
  Partition offset (字节)
       │
       │  Partition Manager: + Partition starting offset
       ▼
  Disk offset (字节)
       │
       │  disk.sys: Byte offset ÷ Sector size (512/4096)
       ▼
  LBA (Logical Block Address)
       │
       │  Port/Miniport: SCSI READ command
       ▼
  Physical location on disk
```

---

### Complete I/O Flow: A Read Request Walkthrough

Let's trace a complete read request — from `ReadFile()` to data return — through every layer.

**Scenario**: App reads the first 4096 bytes of `C:\Data\report.txt`

| Step | Layer | What Happens |
|------|-------|-------------|
| 1 | **Application** | Calls `ReadFile(hFile, buffer, 4096, ...)` → syscall to kernel |
| 2 | **I/O Manager** | Creates IRP (`IRP_MJ_READ`), allocates I/O Stack Locations, sends to file system |
| 3 | **NTFS.sys** | Checks cache → miss → looks up MFT → translates file offset → volume LCN offset |
| 4 | **volsnap.sys** | Read = pass-through (no copy-on-write needed) |
| 5 | **Volume Manager** | Maps volume offset → partition offset (1:1 for simple volumes) |
| 6 | **partmgr.sys** | Adds partition start offset → absolute disk byte offset |
| 7 | **disk.sys** | ★ Converts IRP → SRB, creates SCSI READ CDB with LBA |
| 8 | **storport.sys** | Queues SRB, calls miniport, starts timeout timer |
| 9 | **Hardware** | Reads data from media, DMA to memory, raises interrupt |
| 10 | **Completion ↑** | SRB completes → IRP completes → each layer processes up → data returned to app |

### Address Translation Summary

```
  File offset → (NTFS VCN→LCN) → Volume offset → (VolMgr) → Partition offset
  → (PartMgr + start) → Disk offset → (disk.sys ÷ sector) → LBA → Physical
```

---

## 5. 关键限制对照表 (Key Limitations Reference Table)

### 分区限制 (Partition Limits)

| 限制项 | MBR | GPT |
|--------|-----|-----|
| 最大分区数 | 4 主分区 | 128 分区 |
| 最大磁盘容量 | 2 TiB | 18 EB（理论）/ ~256 TiB（Windows 实际） |
| 最大分区大小 | 2 TiB | 18 EB（理论）/ ~256 TiB（Windows 实际） |
| 冗余分区表 | ❌ | ✅ (Primary + Backup) |
| UEFI 引导 | ❌ | ✅ |
| 跨磁盘 | ❌ | ❌ |

### 卷限制 (Volume Limits)

| 限制项 | 基本卷 (Basic) | 动态卷 (Dynamic) | Storage Spaces |
|--------|---------------|-----------------|----------------|
| 跨磁盘 | ❌ | ✅ | ✅ |
| RAID (Stripe/Mirror) | ❌ | ✅ | ✅ |
| 最大卷大小 | 受分区表限制 | 受分区表限制 | 取决于池大小 |

### 文件系统限制 (File System Limits)

| 限制项 | NTFS | ReFS | FAT32 | exFAT |
|--------|------|------|-------|-------|
| 最大卷大小 | 256 TiB | 4.7 ZB | 2 TiB | 128 PiB |
| 最大文件大小 | 256 TiB | 18 EB | 4 GiB - 1 B | 16 EB |
| 最大文件名长度 | 255 字符 | 255 字符 | 8.3 / 255 (LFN) | 255 字符 |
| 最大路径长度 | 32,767 字符 (NTFS) / 260 (Win32 API) | 32,767 字符 | 260 字符 | 260 字符 |
| 最大簇大小 | 2 MiB (Win Server 2019+) | 64 KiB | 32 KiB | 32 MiB |
| 日志 | ✅ | ✅ | ❌ | ❌ |
| ACL | ✅ | ✅ | ❌ | ❌ |
| 数据完整性校验 | ❌ | ✅ | ❌ | ❌ |
| 可引导 | ✅ | ❌ | ✅ (BIOS) | ❌ |

---

### Partition Limits

| Limit | MBR | GPT |
|-------|-----|-----|
| Max Partitions | 4 primary | 128 |
| Max Disk Size | 2 TiB | 18 EB (theory) / ~256 TiB (Windows) |
| Redundant Table | ❌ | ✅ |
| UEFI Boot | ❌ | ✅ |
| Span Disks | ❌ | ❌ |

### File System Limits

| Limit | NTFS | ReFS | FAT32 | exFAT |
|-------|------|------|-------|-------|
| Max Volume | 256 TiB | 4.7 ZB | 2 TiB | 128 PiB |
| Max File | 256 TiB | 18 EB | ~4 GiB | 16 EB |
| Max Path | 32,767 (NTFS) / 260 (Win32) | 32,767 | 260 | 260 |
| Journal | ✅ | ✅ | ❌ | ❌ |
| ACL | ✅ | ✅ | ❌ | ❌ |
| Integrity | ❌ | ✅ | ❌ | ❌ |
| Bootable | ✅ | ❌ | ✅ | ❌ |

---

## 6. 排查速查 (Troubleshooting Quick Reference)

### 根据症状定位层级

| 症状 | 最可能的层级 | 排查方向 |
|------|------------|---------|
| **文件访问慢** | File System / Cache | NTFS MFT 碎片、缓存命中率、防病毒过滤驱动 |
| **磁盘响应慢 (高延迟)** | Port/Miniport / Hardware | storport ETW trace、SMART data、HBA 队列深度 |
| **磁盘离线** | Partition Manager / Class Driver | 分区表损坏、disk signature 冲突、SAN policy |
| **I/O 超时 / Hung I/O** | Port Driver (storport) | storport timeout (default 30s)、SRB 排队 |
| **卷无法挂载** | Volume Manager / File System | LDM database 损坏、NTFS $LogFile 不一致 |
| **Shadow Copy 失败** | VSS | VSS writer 状态、diff area 空间不足 |
| **分区丢失** | Partition Manager | MBR/GPT 损坏、GPT backup header |
| **蓝屏 (BSOD) 存储相关** | Any layer | Bugcheck 分析、storport dump filter |
| **iSCSI 连接断开** | Miniport + Network | iSCSI initiator 日志 + 网络抓包 |

### 关键诊断工具

| 工具 | 作用 |
|------|------|
| `storport ETW trace` | storport 层 I/O 延迟分析 |
| `diskpart` | 分区和卷管理 |
| `fsutil` | 文件系统查询和配置 |
| `wmic diskdrive / logicaldisk` | 磁盘和卷信息 |
| `vssadmin list shadows/writers` | VSS 状态检查 |
| `windbg !irp / !devstack` | 内核级 IRP 和驱动栈分析 |

---

### Troubleshooting Quick Reference

| Symptom | Likely Layer | Investigation |
|---------|-------------|---------------|
| **Slow file access** | File System / Cache | MFT fragmentation, cache hit ratio, AV filter drivers |
| **High disk latency** | Port/Miniport / HW | storport ETW trace, SMART, HBA queue depth |
| **Disk offline** | PartMgr / Class Driver | Partition table corruption, disk signature collision |
| **I/O timeout / hung** | Port Driver | storport timeout (30s default), SRB queue |
| **Volume won't mount** | VolMgr / File System | LDM database corruption, NTFS $LogFile |
| **Shadow copy failure** | VSS | VSS writer state, diff area space |
| **iSCSI disconnect** | Miniport + Network | iSCSI initiator logs + packet capture |

---

## 7. 总结 (Summary)

Windows 存储栈是一个**精心设计的分层系统**，每一层各司其职：

```
  应用 → I/O Manager (IRP) → 文件系统 (逻辑→物理映射)
       → VSS (快照) → 卷管理器 (卷→分区)
       → 分区管理器 (分区→磁盘) → 类驱动 (IRP→SRB)
       → 端口/微端口 (SRB→硬件) → 磁盘硬件
```

**核心记忆点：**

1. **IRP 是上半部的通用语言**，SRB 是下半部的通用语言，转换点在 disk.sys
2. **分区不能跨磁盘**，卷可以
3. **storport 是诊断金矿**——它是 Windows 能控制的最底层
4. **MAX_PATH 260 不是 NTFS 的限制**——是 Win32 API 的历史遗留
5. 排查存储问题时，先确定**问题在哪一层**，再深入该层分析

---

The Windows Storage Stack is a **well-engineered layered system** where each layer has a clear responsibility:

**Key takeaways:**

1. **IRP is the common language of the upper half**, SRB is the common language of the lower half, with disk.sys as the conversion point
2. **Partitions cannot span disks**; volumes can
3. **storport is diagnostic gold** — it's the lowest layer Windows controls
4. **MAX_PATH 260 is not an NTFS limit** — it's a Win32 API legacy constraint
5. When troubleshooting storage, first determine **which layer** the problem is at, then dig deeper into that layer
