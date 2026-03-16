---
layout: post
title: "Deep Dive: NTFS 卷扩展与 $Bitmap 死锁挂起问题"
date: 2026-03-16
categories: [Knowledge, Storage]
tags: [ntfs, bitmap, checkpoint, caching, windows, storage]
type: "deep-dive"
---

# Deep Dive: NTFS 卷扩展与 `$Bitmap` 死锁挂起问题 / NTFS Volume Extension Hang due to `$Bitmap` Deadlock

**Topic:** NTFS 卷扩展、元数据文件 `$Bitmap` 与缓存死锁问题  
**Category:** Storage  
**Level:** 中级  
**Last Updated:** 2026-03-16

---

## 1. 概述 (Overview)

在云环境中，创建 Windows 虚拟机时，常见的做法是在系统首次启动（first boot）阶段自动扩展系统盘 C 盘，使其占满底层磁盘或预期大小。看似简单的“扩容 C 盘”操作，实际上会触发文件系统 NTFS 内部一系列对元数据文件的更新和缓存管理逻辑。

本案例中，当系统在扩展系统卷（特别是 C 盘）时，NTFS 在更新内部元数据文件 `$Bitmap` 并进行缓存清理、检查点（checkpoint）操作时，由于内部两个线程之间的竞态和资源锁（EResource）竞争，导致形成死锁，使系统在扩容过程中长时间无响应/挂起。

从 support 角度看，这个问题背后涉及多个 Storage 相关知识点：卷扩展、NTFS 元数据、文件系统缓存（Cache Manager）、VACB、NTFS 资源锁以及注册表控制的行为开关等。理解这些概念，有助于在类似“扩 C 盘导致挂起”的故障里快速定位方向，而不是只停留在“某个补丁有 bug”这种表层结论。

---

## 2. 核心概念 (Core Concepts)

### 2.1 卷 (Volume) 与 C 盘

- **卷 (Volume)**：逻辑上的存储单元，可以对应一个物理磁盘、一个分区，或多个物理磁盘组合。Windows 中我们看到的 `C:\`、`D:\` 就是不同的卷。
- **扩展卷 (Extend Volume)**：在不破坏现有数据的前提下，增加卷所使用的物理空间。例如，把 C 盘从 50 GB 扩到 100 GB。
- **系统盘 C 盘**：通常承载操作系统、系统文件和部分应用，是最敏感的卷；任何在 C 盘上的长时间挂起，往往被用户感知为“整个系统卡死”。

### 2.2 NTFS 与元数据文件 `$Bitmap`

- **NTFS 文件系统**：Windows 上最常见的文件系统，负责管理文件、目录、权限、日志以及磁盘上哪些空间已用/未用。
- **元数据文件 (Metadata Files)**：NTFS 自己用来“记账”的内部文件，例如：
  - `$MFT`：记录所有文件/目录的元数据。
  - `$LogFile`：记录文件系统事务日志。
  - `$Bitmap`：用一串 bit 记录磁盘上每个簇（cluster）是“占用”还是“空闲”。
- **`$Bitmap` 的作用**：
  - 卷扩容时，新的物理空间需要被标记为“可分配”；
  - 这就必须更新 `$Bitmap`，在这块新空间对应的 bit 上标记为“空闲”。

### 2.3 文件系统缓存与 VACB

- **文件系统缓存 (Cache Manager)**：为提高 I/O 性能，Windows 会把文件内容的一部分缓存到内存，避免每次读都访问磁盘。
- **VACB（Virtual Address Control Block）**：可以简单理解为“缓存视图的描述符”：
  - 描述某个文件的一段数据是如何映射到内存地址的；
  - 记录有多少线程正在使用这块缓存视图（引用计数、活动计数）。
- **缓存清理 (Purge)**：当需要释放缓存或保证一致性时，Cache Manager 会：
  - 取消映射（unmap）某些 VACB；
  - 等待所有正在使用这些视图的线程完成访问（计数归零）。

### 2.4 NTFS 资源锁 (EResource)

- **EResource**：NT 内核中的一种读写锁实现，NTFS 用它来保护关键数据结构（如 FCB、卷状态等）。
- 锁模式：
  - 共享 (Shared)：多个读者可以并发持有；
  - 独占 (Exclusive)：只有一个写者可以持有，期间其他线程不能读写。
- 在 NTFS 中，很多敏感操作（例如更改卷大小、更新元数据）会在持有独占锁的情况下进行，保证一致性，但也增加了死锁风险。

### 2.5 Checkpoint（检查点）

- **Checkpoint**：NTFS 周期性做的“对账动作”，确保：
  - 内存中的变更和磁盘上的记录是同步、可恢复的；
  - 日志（`$LogFile`）和元数据状态一致，便于系统崩溃后恢复。
- 在做 checkpoint 时，会：
  - 访问并锁定一些关键元数据文件（如 `$Bitmap`）；
  - 防止在这段时间内发生破坏一致性的并发修改。

---

## 3. 工作原理 (How It Works)

### 3.1 整体架构 (High-Level Architecture)

可以用一个简化的文字架构图来表示卷扩展时涉及的组件：

- 用户/脚本
  - 调用卷管理接口（如 `DeviceIoControl` 执行 FSCTL 扩展卷）。
- 卷管理服务
  - 例如 `vds.exe`（Virtual Disk Service）等组件，负责与文件系统/存储栈交互。
- NTFS 文件系统驱动 (`Ntfs.sys`)
  - 接收“扩展卷”请求；
  - 更新卷大小、元数据文件（特别是 `$Bitmap`）；
  - 触发/配合缓存清理和 checkpoint 逻辑。
- Cache Manager
  - 管理文件缓存、VACB；
  - 处理 `CcPurgeCacheSection` 等缓存清理调用。
- 系统线程（如 checkpoint worker）
  - 周期性对卷进行检查点操作，访问 `$Bitmap` 等元数据。

### 3.2 关键流程：扩展系统卷时发生了什么

以“扩展 C 盘”为例，简化后的核心流程如下：

1. **Step 1: 用户或自动化脚本请求扩展卷**
   - 通过存储管理工具、API 或云平台的自动化脚本，调用底层接口（如 `DeviceIoControl`），请求扩展 C 盘卷大小。

2. **Step 2: 卷管理服务调用 NTFS**
   - 例如 `vds.exe` 的线程向 NTFS 发送文件系统控制请求（FSCTL），入口类似：
     - `DeviceIoControl` → `NtFsControlFile` → `NtfsChangeVolumeSize`。

3. **Step 3: NTFS 更改卷大小并更新 `$Bitmap`**
   - NTFS 在内部执行：
     - 调整卷的逻辑大小；
     - 更新 `$Bitmap`，标记新增空间对应的 bit 为“可分配”。
   - 为了保证安全，NTFS 会：
     - 持有卷级别或 FCB 级别的独占锁（EResource）；
     - 调用 `NtfsPurgeCacheSection` / `CcPurgeCacheSection` 清理相关缓存，确保元数据状态一致。

4. **Step 4: Cache Manager 清理 `$Bitmap` 缓存**
   - `CcPurgeCacheSection` 负责取消映射与 `$Bitmap` 相关的一些 VACB：
     - 调用 `CcUnmapVacbArray` unmap 视图；
     - 然后等待这些 VACB 的“活动计数”和“映射视图计数”归零。

5. **Step 5: 系统 checkpoint 线程访问 `$Bitmap`**
   - 另一边，系统线程（checkpoint worker）周期性地对卷做 checkpoint：
     - 调用 `NtfsCheckpointAllVolumesWorker` → `NtfsCheckpointVolume`；
     - 为了保证一致性，通过 `NtfsLockSystemFilePages` / `NtfsLockUnlockSystemFilePages` 对 `$Bitmap` 文件的页面加锁；
     - 读 `$Bitmap` 数据，路径类似 `NtfsCommonRead` → `NtfsFsdRead` 等。

6. **Step 6: 竞态和死锁的形成**
   - 问题出在 `CcPurgeCacheSection` 与 `NtfsLockUnlockSystemFilePages` 的交互上：
     - 一边（扩卷线程）认为 VACB 已经被 unmap，正在等待其引用计数清零；
     - 另一边（checkpoint 线程）却在这个时机又重新 map/使用同一个 VACB；
   - 结果：
     - 扩卷线程：
       - 持有 `$Bitmap`/卷的 EResource 独占锁；
       - 在 `CcPurgeCacheSection` 中一直等 VACB 活动计数/映射计数归零，但由于有新的使用者，这个计数降不下来；
     - checkpoint 线程：
       - 为了读 `$Bitmap`，需要获取同一把 EResource（共享锁）；
       - 但锁被扩卷线程独占持有，只能等待。
   - 两个线程互相等待：
     - 扩卷线程等 VACB 释放（由 checkpoint 线程完成）；
     - checkpoint 线程等锁释放（由扩卷线程完成）；
   - 这就是典型的**死锁 (deadlock)**，对外表现为“扩展 C 盘时系统长时间无响应/挂起”。

### 3.3 关键机制：NtfsLockSystemFilePages 与注册表控制

- `NtfsLockSystemFilePages` / `NtfsLockUnlockSystemFilePages`：
  - 用于在某些内部流程（如 checkpoint）中，锁定系统文件（例如 `$Bitmap`）的页面，避免在关键窗口期被换出或被其他操作干扰。
- 注册表键：
  - 路径：`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\FileSystem\NtfsLockSystemFilePages`
  - 含义：是否启用对系统文件页面的这类锁定机制。
- 在本案例中：
  - 开启该机制会走“锁系统文件页面”这条路径，从而参与到上述死锁链条；
  - 作为 workaround，将其设置为 `0` 可以绕开这条路径，打破死锁条件。  

---

## 4. 关键配置与参数 (Key Configurations)

| 配置项/参数                              | 默认值    | 说明                                                                 | 常见调优/修改场景                                      |
|-----------------------------------------|-----------|----------------------------------------------------------------------|--------------------------------------------------------|
| 卷大小 / 磁盘布局                        | 视环境而定 | 决定卷可用空间和扩展策略                                             | 云平台创建镜像、自动化扩展 C 盘                       |
| 文件系统类型                             | NTFS      | 决定元数据结构和行为（如 `$Bitmap`、日志等）                        | 选择支持高级特性的文件系统（权限、配额、日志等）     |
| `NtfsLockSystemFilePages` (Registry)    | 通常为启用 | 控制是否对系统文件（如 `$Bitmap`）页面进行额外锁定                 | 针对已知死锁/挂起问题的临时规避手段                  |
| Checkpoint 周期（内部机制）             | 内部策略   | 决定 NTFS 多久做一次检查点                                           | 一般不直接调优；理解其存在有助于分析并发访问行为     |
| Cache Manager 策略（VACB、映射窗口等）   | 内部策略   | 决定文件缓存映射/清理行为                                           | 出问题时通过堆栈分析理解其行为，不直接做配置修改     |

在本案例中，最直接相关、可操作的配置是注册表键 `NtfsLockSystemFilePages`，其调整直接影响是否走会触发死锁的代码路径。

---

## 5. 常见问题与排查 (Common Issues & Troubleshooting)

### 问题 A: 扩展 C 盘时系统无响应/挂起

- **症状描述**：
  - 在执行“扩展系统盘 C 盘”操作时，系统长时间无响应；
  - 扩容操作无法完成，可能需要重启机器；
  - 在云环境中，首启脚本卡住，影响虚拟机初始化成功率。

- **可能原因**：
  - NTFS 在扩展卷时更新 `$Bitmap`，同时 Cache Manager 正在清理 `$Bitmap` 缓存；
  - 系统 checkpoint 线程在这一时间窗口内尝试锁定并读取 `$Bitmap` 页面；
  - `CcPurgeCacheSection` 与 `NtfsLockUnlockSystemFilePages` 之间发生竞态，导致两个线程对同一 EResource/VACB 互相等待，形成死锁。

- **排查思路**（适用于有内核转储/调试环境的场景）：
  1. 收集内核 dump 或使用 live debugging，检查挂起时有哪些线程处于等待状态；
  2. 遍历线程堆栈，关注：
     - 是否有线程堆栈类似：`vds.exe` → `DeviceIoControl` → `NtFsControlFile` → `NtfsChangeVolumeSize` → `NtfsPurgeCacheSection` → `CcPurgeCacheSection` → `CcUnmapVacbArray`；
     - 是否有系统线程堆栈类似：`System` → `NtfsCheckpointAllVolumesWorker` → `NtfsCheckpointVolume` → `NtfsLockSystemFilePages` → `NtfsLockUnlockSystemFilePages` → `NtfsCommonRead` 等；
  3. 使用调试扩展（如 `!mex.t`、`!fileobj`、`!ddt` 等）查看：
     - 哪些线程持有某个 EResource（卷/FCB 锁）；
     - 哪些线程在等待同一个 EResource；
     - `Shared Cache Map` 和 `_PRIVATE_CACHE_MAP` 中与 `$Bitmap` 相关的字段（如 `NumMappedVacb`、`NumActiveVacb`）。
  4. 确认：
     - 扩卷线程独占持有某个资源锁；
     - checkpoint 线程在等待该锁，同时又在使用 VACB，导致 `CcPurgeCacheSection` 一直等不到计数归零。

- **关键命令/工具**：
  - `!thread` / `!mex.t`：查看线程堆栈和等待信息；
  - `!locks` / `!mex.locks`：查看资源锁持有/等待情况；
  - `!fileobj`：确认 `$Bitmap` 文件对象；
  - `!ddt`：查看 `SharedCacheMap` 与 `_PRIVATE_CACHE_MAP` 内容。

### 问题 B: 卷扩展失败但未明显挂起（边缘场景）

- **症状描述**：
  - 扩展卷操作偶尔失败或超时，但系统未完全挂死；
  - 日志中可能出现 NTFS 或 volmgr 相关错误事件。

- **可能原因**：
  - 同一类竞态在较轻微情况下表现为重试/超时；
  - 资源竞争严重程度不一时，可能只是性能问题或操作失败，而非完全死锁。

- **排查思路**：
  - 检查 System 日志中的 NTFS、volmgr、disk 事件；
  - 分析失败时间点附近是否发生大量元数据操作或其他卷级别操作；
  - 如有可能，复现场景并收集性能计数器与 ETW trace。  

---

## 6. 实战经验 (Practical Tips)

- **最佳实践**：
  - 在设计云平台或自动化脚本时，将“首启扩展 C 盘”作为一个明确的步骤，并考虑该步骤失败时的回滚或重试机制；
  - 在大规模环境中，尽量避免在同一时间窗口对大量 VM 同时做重度存储操作，以降低元数据访问的高并发风险。

- **在已知问题版本上的规避策略**：
  - 对于确认存在该 NTFS 死锁问题的 OS 版本，可在受控范围内启用注册表 workaround：
    - `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\FileSystem\NtfsLockSystemFilePages = 0`；
  - 先在测试/预发环境验证：
    - 扩展 C 盘操作是否稳定完成；
    - System 日志中是否出现新的 NTFS/volmgr 错误事件；
  - 再逐步推广到生产镜像。

- **常见误区**：
  - 误以为“扩容 C 盘挂起 = 纯粹磁盘性能问题或某个驱动超时”，忽略了 NTFS 元数据和缓存层面的死锁可能性；
  - 只从存储硬件/虚拟化层排查，而不去看 NTFS 堆栈和锁等待信息。

- **性能考量**：
  - 卷扩展操作本身已经是重度元数据更新操作，在大并发场景下对 NTFS 和 Cache Manager 压力较大；
  - 关闭 `NtfsLockSystemFilePages` 虽然能规避本案例中的死锁，但在极端内存压力或高并发元数据访问场景下，理论上会略微降低系统文件访问的“保护力度”，需要靠前期测试来平衡风险。

- **安全注意**：
  - 任何对注册表、尤其是文件系统相关键值的修改，都应：
    - 有明确的变更记录和回退方案；
    - 在受控环境先验证；
    - 尽量只在受影响版本/镜像上启用，而不是“一刀切”全局开启。

---

## 7. 与相关技术的对比 (Comparison with Related Technologies)

| 维度             | NTFS + `$Bitmap` 机制                 | 其他文件系统/机制示例 (概念性)                |
|------------------|----------------------------------------|----------------------------------------------|
| 空间分配方式     | 通过 `$Bitmap` 记录簇是否被占用       | 某些文件系统使用区段表、B-Tree 等结构        |
| 元数据文件形式   | 使用内部文件（如 `$MFT`、`$Bitmap`） | 部分文件系统将元数据分布在固定区域           |
| 缓存与映射       | 使用 Cache Manager + VACB 映射视图    | 其他系统可能用不同的缓存/映射策略            |
| 一致性保障       | 使用日志 (`$LogFile`) + checkpoint    | 可能采用 journaling、copy-on-write 等机制    |
| 并发控制         | 通过 EResource、FCB 锁等控制访问      | 其他系统可能用不同粒度的锁或事务机制         |
| 行为调优/开关    | 通过注册表控制部分行为（如本文中的键）| 其他系统可能通过配置文件或挂载参数调整行为   |

对于 support 工程师而言，关键不是精通所有文件系统实现，而是：
- 知道 NTFS 这些内部机制大致长什么样；
- 出问题时能从堆栈和锁信息中看出是“元数据/缓存层死锁”，而不只是“磁盘忙”。

---

## 8. 参考资料 (References)

暂无可验证的参考文档

---

# Deep Dive (EN): NTFS Volume Extension and `$Bitmap` Deadlock Hang

**Topic:** NTFS volume extension, `$Bitmap` metadata file and cache deadlock  
**Category:** Storage  
**Level:** Intermediate  
**Last Updated:** 2026-03-16

---

## 1. Overview (EN)

In many cloud environments, when provisioning Windows virtual machines, it is common to extend the system drive (C: volume) during the first boot so that it uses all available disk space or reaches a predefined size. This seemingly simple "extend C drive" operation actually triggers a series of internal NTFS operations on metadata files and the file system cache.

In this case, when extending the system volume (especially C:), NTFS updates the internal metadata file `$Bitmap` and coordinates with cache purge and checkpoint operations. Due to a race condition between two internal threads and contention on an EResource lock, a deadlock can be formed, causing the system to become unresponsive or hang during the volume extension.

From a support perspective, this problem touches multiple storage-related topics: volume extension, NTFS metadata, the file system cache (Cache Manager), VACBs, NTFS resource locks, and behavior switches controlled by registry keys. Understanding these concepts helps you quickly narrow down issues like "system hangs when extending C:" instead of stopping at a shallow conclusion such as "some patch is buggy".

---

## 2. Core Concepts (EN)

### 2.1 Volume and C Drive

- **Volume**: A logical storage unit that may correspond to a physical disk, a partition, or a combination of disks. In Windows, `C:\`, `D:\` etc. are different volumes.
- **Extend Volume**: Increasing the amount of physical space used by a volume without destroying existing data, e.g., extending C: from 50 GB to 100 GB.
- **System Drive (C:)**: Typically holds the operating system, system files, and some applications. Any long hang on C: is often perceived as "the whole system is frozen".

### 2.2 NTFS and the `$Bitmap` Metadata File

- **NTFS File System**: The most common file system on Windows, responsible for managing files, directories, permissions, logs, and which areas of the disk are used or free.
- **Metadata Files**: Internal files used by NTFS to keep track of its own state, for example:
  - `$MFT`: Master File Table, stores metadata of files and directories.
  - `$LogFile`: Transaction log for the file system.
  - `$Bitmap`: A bit array indicating which clusters on the volume are in use or free.
- **Role of `$Bitmap`**:
  - When the volume is extended, the new physical space must be marked as allocatable;
  - NTFS achieves this by updating `$Bitmap` and marking the bits corresponding to the new space as "free".

### 2.3 File System Cache and VACB

- **File System Cache (Cache Manager)**: To improve I/O performance, Windows caches parts of file data in memory to avoid hitting the disk on every read.
- **VACB (Virtual Address Control Block)**: Conceptually, this is a descriptor for a cached view:
  - It describes how a section of a file is mapped into virtual memory;
  - It tracks how many threads are using this view (reference count, active count).
- **Cache Purge**: When the system needs to free cache or ensure consistency, the Cache Manager:
  - Unmaps certain VACBs;
  - Waits for all active users of those views to finish (counts go to zero).

### 2.4 NTFS Resource Locks (EResource)

- **EResource**: A read/write lock primitive used in NT to protect critical data structures such as FCBs and volume state.
- Lock modes:
  - Shared: Multiple readers can hold the lock concurrently;
  - Exclusive: Only one writer can hold the lock; others are blocked.
- In NTFS, many sensitive operations (e.g., changing volume size, updating metadata) take place while holding exclusive locks to guarantee consistency, but this also increases the risk of deadlocks.

### 2.5 Checkpoint

- **Checkpoint**: A periodic reconciliation process in NTFS to ensure that:
  - In-memory changes and on-disk records are consistent and recoverable;
  - The log (`$LogFile`) and metadata state match, which is important for crash recovery.
- During a checkpoint, NTFS:
  - Accesses and locks important metadata files such as `$Bitmap`;
  - Prevents concurrent modifications that could break consistency.

---

## 3. How It Works (EN)

### 3.1 High-Level Architecture

The following components are involved when extending a volume:

- User / Automation Script
  - Issues a request to extend the volume (e.g., via `DeviceIoControl` with appropriate FSCTLs).
- Volume Management Service
  - For example, `vds.exe` (Virtual Disk Service) interacts with the file system and storage stack.
- NTFS File System Driver (`Ntfs.sys`)
  - Receives the "extend volume" request;
  - Updates the volume size and metadata files (especially `$Bitmap`);
  - Cooperates with cache purge and checkpoint logic.
- Cache Manager
  - Manages file cache and VACBs;
  - Handles `CcPurgeCacheSection` calls for cache cleanup.
- System Threads (e.g., checkpoint worker)
  - Periodically perform checkpoint operations on volumes, accessing `$Bitmap` and other metadata.

### 3.2 Key Flow: What Happens When Extending the System Volume

For "extending the C: drive", a simplified core flow is:

1. **Step 1: User or Script Requests Volume Extension**
   - A storage management tool, API, or cloud automation script requests to extend C:.

2. **Step 2: Volume Management Service Calls NTFS**
   - A thread in `vds.exe` (for example) sends a file system control request (FSCTL) to NTFS:
     - `DeviceIoControl` → `NtFsControlFile` → `NtfsChangeVolumeSize`.

3. **Step 3: NTFS Changes Volume Size and Updates `$Bitmap`**
   - Internally, NTFS:
     - Adjusts the logical size of the volume;
     - Updates `$Bitmap` to mark the new area as allo catable.
   - To ensure safety, NTFS:
     - Holds an EResource (e.g., volume/FCB lock) in exclusive mode;
     - Calls `NtfsPurgeCacheSection` / `CcPurgeCacheSection` to purge related cache for consistency.

4. **Step 4: Cache Manager Purges `$Bitmap` Cache**
   - `CcPurgeCacheSection`:
     - Calls `CcUnmapVacbArray` to unmap certain views for `$Bitmap`;
     - Waits for VACB active counts and mapped view counts to reach zero.

5. **Step 5: System Checkpoint Thread Accesses `$Bitmap`**
   - Meanwhile, a system thread (checkpoint worker) periodically performs checkpoints:
     - `NtfsCheckpointAllVolumesWorker` → `NtfsCheckpointVolume`;
     - Uses `NtfsLockSystemFilePages` / `NtfsLockUnlockSystemFilePages` to lock pages of `$Bitmap`;
     - Reads `$Bitmap` through paths such as `NtfsCommonRead` / `NtfsFsdRead`.

6. **Step 6: Race and Deadlock Formation**
   - The race is between `CcPurgeCacheSection` and `NtfsLockUnlockSystemFilePages`:
     - The extension thread believes a VACB has been unmapped and is waiting for its usage counts to go to zero;
     - The checkpoint thread, during this critical window, remaps or uses the same VACB again.
   - As a result:
     - The extension thread:
       - Holds the EResource for `$Bitmap`/volume exclusively;
       - Stays in `CcPurgeCacheSection` waiting for VACB counts to drop to zero, which never happens because there is a new user.
     - The checkpoint thread:
       - Needs to acquire the same EResource (in shared mode) to read `$Bitmap`;
       - Is blocked because the lock is held exclusively by the extension thread.
   - This is a classic **deadlock**:
     - The extension thread waits for VACBs to be released by the checkpoint thread;
     - The checkpoint thread waits for the lock to be released by the extension thread;
     - Externally, the system appears hung during the volume extension.

### 3.3 Key Mechanism: NtfsLockSystemFilePages and Registry Control

- `NtfsLockSystemFilePages` / `NtfsLockUnlockSystemFilePages`:
  - Used in internal flows such as checkpoint to lock system file pages (e.g., `$Bitmap`) during sensitive operations.
- Registry Key:
  - Path: `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\FileSystem\NtfsLockSystemFilePages`;
  - Meaning: enables or disables this additional locking mechanism for system file pages.
- In this case:
  - When enabled, the checkpoint path participates in the deadlock chain described above;
  - As a workaround, setting the key to `0` bypasses this path and breaks the deadlock condition.

---

## 4. Key Configurations (EN)

| Setting / Parameter                      | Default      | Description                                                       | Common Tuning / Change Scenarios                          |
|-----------------------------------------|--------------|-------------------------------------------------------------------|-----------------------------------------------------------|
| Volume size / disk layout               | Environment  | Determines available space and extension strategy                 | Cloud image design, automated C: extension                |
| File system type                        | NTFS         | Determines metadata structure and behavior (`$Bitmap`, logs, etc.)| Choosing a feature-rich file system (ACLs, quotas, logs)  |
| `NtfsLockSystemFilePages` (Registry)    | Typically on | Controls extra locking of system file pages (e.g., `$Bitmap`)    | Temporary mitigation for known hang/deadlock issues       |
| Checkpoint interval (internal)          | Internal     | Determines how often NTFS performs checkpoints                    | Not usually tuned directly; important for understanding concurrency |
| Cache Manager policies (VACB, views)    | Internal     | Determines how file cache views are mapped and purged             | Used for analysis in debugging; not tuned directly        |

---

## 5. Common Issues & Troubleshooting (EN)

### Issue A: System Hangs While Extending C:

- **Symptoms**:
  - System becomes unresponsive for a long time during C: volume extension;
  - The extension operation does not complete and may require a reboot;
  - In a cloud environment, first-boot scripts are stuck, affecting VM provisioning success.

- **Possible Cause**:
  - NTFS updates `$Bitmap` while Cache Manager is purging `$Bitmap` cache;
  - A checkpoint thread tries to lock and read `$Bitmap` pages in the same time window;
  - A race between `CcPurgeCacheSection` and `NtfsLockUnlockSystemFilePages` causes two threads to wait on each other through EResource and VACB dependencies, creating a deadlock.

- **Troubleshooting Approach** (for kernel debugging scenarios):
  1. Capture a kernel dump or use live debugging to inspect hung threads;
  2. Walk through thread stacks and look for:
     - A thread in `vds.exe` with a stack like: `DeviceIoControl` → `NtFsControlFile` → `NtfsChangeVolumeSize` → `NtfsPurgeCacheSection` → `CcPurgeCacheSection` → `CcUnmapVacbArray`;
     - A system thread with a stack like: `NtfsCheckpointAllVolumesWorker` → `NtfsCheckpointVolume` → `NtfsLockSystemFilePages` → `NtfsLockUnlockSystemFilePages` → `NtfsCommonRead`;
  3. Use debugger extensions to check:
     - Which thread holds a particular EResource exclusively;
     - Which thread is waiting on the same resource;
     - `SharedCacheMap` / `_PRIVATE_CACHE_MAP` fields that relate to `$Bitmap` (e.g., `NumMappedVacb`, `NumActiveVacb`).
  4. Confirm the deadlock pattern: one thread holds the lock and waits for VACBs to be released; another thread uses the VACBs and waits for the lock.

- **Key Commands / Tools**:
  - `!thread` / equivalent helpers to inspect stacks and wait reasons;
  - `!locks` to inspect lock ownership and waiters;
  - `!fileobj` to confirm the `$Bitmap` file object;
  - `!ddt` to inspect `SharedCacheMap` and `_PRIVATE_CACHE_MAP` structures.

### Issue B: Volume Extension Fails or Times Out Without a Full Hang

- **Symptoms**:
  - Volume extension occasionally fails or times out, but the system does not fully hang;
  - System event log may contain NTFS or volmgr errors.

- **Possible Cause**:
  - A lighter manifestation of similar contention, leading to retries or timeouts rather than a hard deadlock.

- **Troubleshooting Approach**:
  - Review System event logs for NTFS/volmgr/disk events around the failure time;
  - Analyze whether there were heavy metadata operations or other volume-level operations at the same time;
  - If possible, reproduce with performance counters and ETW traces.

---

## 6. Practical Tips (EN)

- **Best Practices**:
  - Treat "first-boot C: extension" as an explicit step in cloud or automation workflows and design proper retry/rollback mechanisms;
  - Avoid performing heavy storage operations on a large number of VMs in the same time window to reduce metadata contention.

- **Mitigation on Affected OS Versions**:
  - For OS versions confirmed to have this NTFS deadlock issue, consider enabling the registry-based workaround in a controlled manner:
    - `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\FileSystem\NtfsLockSystemFilePages = 0`;
  - Validate in test/staging environments first:
    - Confirm that C: extension completes reliably;
    - Check that there are no new critical NTFS/volmgr errors in the System log;
  - Then roll out to production images step by step.

- **Common Pitfalls**:
  - Assuming that "hang while extending C:" is purely a disk performance or driver timeout issue and ignoring NTFS metadata/cache deadlock possibilities;
  - Focusing only on the storage hardware/virtualization layer and not checking NTFS stacks and lock states.

- **Performance Considerations**:
  - Volume extension is already a metadata-heavy operation; under high concurrency it puts stress on NTFS and the Cache Manager;
  - Disabling `NtfsLockSystemFilePages` mitigates this specific deadlock, but theoretically reduces some protection for system file page access under extreme pressure. Proper testing is required to balance the risk.

- **Security / Change Management**:
  - Any changes to registry keys related to the file system should:
    - Have clear change records and rollback plans;
    - Be validated in controlled environments first;
    - Be limited to affected versions/images instead of being applied globally without discrimination.

---

## 7. Comparison with Related Technologies (EN)

| Dimension         | NTFS + `$Bitmap` Mechanism                      | Other File Systems / Mechanisms (Conceptual)            |
|-------------------|-------------------------------------------------|---------------------------------------------------------|
| Space allocation  | Uses `$Bitmap` to track used/free clusters      | Some use extent tables, B-Trees, or other structures    |
| Metadata storage  | Internal files like `$MFT`, `$Bitmap`           | Some have fixed metadata regions                        |
| Caching & mapping | Cache Manager + VACB views                      | Other systems may use different cache/mapping strategies|
| Consistency       | Journaling via `$LogFile` + checkpoints         | Journaling, copy-on-write, or other mechanisms          |
| Concurrency       | EResource locks, FCB locks                      | Different lock granularities or transactional models    |
| Behavior tuning   | Registry keys (e.g., this case’s key)           | Config files, mount options, or other tunables          |

---

## 8. References (EN)

No verifiable public reference documents at the moment.
