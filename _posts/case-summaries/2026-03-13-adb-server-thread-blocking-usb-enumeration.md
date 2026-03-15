---
layout: post
title: "ADB Server Thread Blocking — USB Device Enumeration Deadlock Analysis"
date: 2026-03-13
categories: [ADB, Windows-Debugging]
tags: [adb, usb, procmon, etw, winusb, deadlock, mcafee, process-monitor, thread-blocking]
---

# Case Summary: ADB Server 线程阻塞 — USB 设备枚举死锁分析

**Product/Service:** Android Debug Bridge (ADB) on Windows 10/11

---

## 1. 症状 (Symptoms)

- 客户在 Windows 机器上执行 `adb devices` 命令后，**命令长时间挂起无响应**，无法返回设备列表
- 多次重试（包括 `adb kill-server` 后重启）均无效
- 问题持续存在，尝试修复失败

## 2. 背景 (Background / Environment)

- **操作系统**：Windows 10/11 Enterprise (Build 26100.7840, amd64)
- **ADB 路径**：`E:\Software\UIAutomation\platform-tools-latest-windows2\platform-tools\adb.exe`（32 位 WoW64 模式运行）
- **已安装软件**：
  - Docker Desktop（`kubernetes.docker.internal` 作为 localhost 别名）
  - McAfee Endpoint Security（mcshield.exe 实时扫描 + mfemactl.exe 管理代理）
  - 企业终端管理软件（CGEData/CGEComm/CGESA 系列）
  - 飞书 (Feishu)、Chrome、IntelliJ IDEA 等
- **USB 设备**：客户通过 USB 连接 Android 设备进行 UI 自动化测试

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

客户在 adb 问题尝试修复失败时抓取了 **Process Monitor (ProcMon) PML trace** 和 **ETW network trace**。

### 3.1 ProcMon 分析 — 进程识别

1. **识别 adb 进程树**：PML 文件中发现 5 个 adb.exe 进程：

   | PID | 角色 | 父 PID | 说明 |
   |-----|------|--------|------|
   | 75708 | Server 父进程 | 29056 | adb server 守护进程 |
   | 9856 | **Server 工作进程** | 75708 | ⚠️ **卡死的核心进程** |
   | 58532 | Client #1 | 107596 | 旧客户端，15:02:41 退出 |
   | 56312 | Client #2 | 107596 | 新客户端，15:02:59~15:03:09 |
   | 59972 | Client #3 | 107596 | 15:05:16 出现 |

### 3.2 ProcMon 分析 — TCP 通信时间线

2. **还原 TCP 通信流程**：
   - `15:02:59.180` — Client (PID 56312) 通过端口 58567 连接到 Server 端口 5037
   - `15:02:59.180` — Server (PID 9856) **TCP Accept** + **TCP Receive** 收到 16 字节请求
   - 16 字节 = adb 协议的 `000chost:devices` 请求（4 位十六进制长度前缀 + 命令字符串）
   - **此后 Server 零事件**：无文件 I/O、无注册表访问、无 TCP 发送

3. **确认 Server 完全静默**：PID 9856 在整个 ProcMon trace 期间（15:02:17 → 15:05:53, 约 3.5 分钟）**仅有 Process_Profiling 心跳事件**（TID=0），所有工作线程无任何可见 I/O 操作。

### 3.3 ETW TCP Trace 分析 — 铁证确认

4. **ETW trace 验证**（15:04:23 → 15:06:10，约 106 秒）：
   - 连接 `127.0.0.1:5037 → 127.0.0.1:58624` 仍处于 EstablishedState
   - Server 端所有 TCP send 事件：**BytesSent = 0**（仅发送 TCP ACK，从未返回应用数据）
   - 产生 **466 次 "Duplicate Segment" 丢包**（TCP keepalive 重传循环）
   - 15:04:51 又一个新客户端从端口 60138 发送 16 字节请求，**同样无响应**

   **结论：Server 在整个观察期内（>3 分钟）从未向任何客户端返回过一个字节的应用数据。**

### 3.4 McAfee 实时扫描行为分析

5. **发现 McAfee mcshield.exe (PID 6280) 异常活跃**：
   - 在 adb 客户端启动的 **同一时刻** (15:02:59)，McAfee 对 `platform-tools` 目录产生 **248 次文件扫描操作**
   - 扫描目标包括：`adb.exe`、`AdbWinApi.dll`、`IPHLPAPI.DLL` 等关键文件
   - 操作类型：QueryOpen、CreateFile、QueryDirectory、CloseFile 等

### 3.5 DLL 加载分析

6. **Client PID 56312 的 DLL 加载序列** 确认了 USB 子系统的参与：
   - `AdbWinApi.dll` — ADB USB 接口库
   - `AdbWinUsbApi.dll` — ADB WinUSB API 适配层
   - `winusb.dll` — Windows USB 用户态驱动接口
   - `mswsock.dll` — Windows Socket 扩展
   - `IPHLPAPI.DLL` — IP Helper API

### 3.6 WUDFHost.exe 活动分析

7. **WUDFHost.exe (PID 1564)**：USB 用户态驱动宿主在关键窗口产生 190 个事件，主要是 Intel DPTF 热管理相关文件 I/O。虽然与 ADB USB 通信不直接相关，但说明 USB 子系统处于活跃状态。

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| ProcMon 无法捕获 USB DeviceIoControl 调用 | 无法直接看到 Server 阻塞在哪个 USB 调用上 | 通过排除法（零事件 = 阻塞在内核级调用）推断出 USB 枚举是阻塞源 |
| ProcMon 堆栈地址未解析为符号 | 无法精确定位阻塞函数 | 需要通过 procdump 抓取进程 dump 做进一步分析 |
| ETW trace 在 ProcMon 之后启动 | 未捕获到最初的 TCP 连接建立过程 | 通过 ProcMon TCP 事件补充了早期时间线 |
| 无法确认 USB 设备状态 | 不清楚是否有异常 USB 设备导致枚举挂起 | 建议客户提供 Device Manager 截图 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

**ADB Server 的 USB 设备枚举线程卡死在 WinUSB/内核驱动层调用中，导致 `transport_lock` 互斥锁被无限期持有，进而阻塞所有客户端请求处理线程。**

详细机制：

```
USB Tracker 线程 (持有 transport_lock):
  usb_discover()
    → WinUsb_* API 调用
      → DeviceIoControl → 内核 USB 驱动栈
        → ⚠️ USB 设备无响应 / 驱动层死锁
        → 线程永久阻塞，锁无法释放

Client Handler 线程 (等待 transport_lock):
  host_devices()
    → 尝试获取 transport_lock
      → ⚠️ 被 USB tracker 线程持有，无限等待
      → 客户端永远收不到响应
```

加剧因素：
- **McAfee 实时扫描** (248 次扫描操作) 可能延迟 USB 设备文件/管道访问
- **多层企业安全软件** 增加系统调用拦截延迟
- **Docker Desktop** 可能与 USB 资源存在竞争

### Resolution

问题在调查阶段，以下为建议的解决步骤：

1. **抓取 adb server 进程 dump** 确认线程调用栈：
   ```
   procdump -ma <adb_server_pid> C:\temp\adb_dump.dmp
   ```
2. **检查 USB 设备状态**：打开 Device Manager，查看是否有黄色感叹号的 USB 设备
3. **尝试拔插 USB 设备**：物理断开所有 USB Android 设备，观察 server 是否恢复
4. **重启 adb server**：
   ```
   adb kill-server
   taskkill /F /IM adb.exe    （确保所有实例被终止）
   adb start-server
   adb devices
   ```

### Workaround

- 将 `platform-tools` 目录加入 McAfee 排除列表，减少扫描干扰
- 临时关闭 Docker Desktop 排除 USB 资源冲突
- 使用 `adb tcpip 5555` 切换为 WiFi 调试模式，绕过 USB 枚举

## 6. 经验教训 (Lessons Learned)

- **技术知识**：
  - ADB server 使用 `transport_lock` 互斥锁保护设备传输列表；USB tracker 线程卡死会导致级联阻塞所有客户端请求
  - ProcMon **无法捕获** USB DeviceIoControl 等内核级设备 I/O 调用；当一个进程在 ProcMon 中 "完全沉默"（仅有 Process_Profiling），通常意味着线程阻塞在内核态调用中
  - adb.exe 的 `host:devices` 请求格式为 16 字节：`000chost:devices`（4 位十六进制长度 + 12 字符命令）

- **排查方法**：
  - **ProcMon + ETW 联合分析** 非常有效：ProcMon 提供进程/线程/文件/注册表维度，ETW 提供 TCP 层面精确的字节级证据
  - **"零事件" 本身就是关键证据**：Server 接收请求后零 I/O 活动 = 阻塞在不可见的内核调用中
  - **多客户端均超时** 是判断 server 级问题（而非单次客户端问题）的重要信号
  - 分析 PML 文件时，先用 `procmon-parser` Python 库快速提取关键进程，再做定向深入分析

- **预防措施**：
  - 在有企业安全软件（McAfee 等）的环境中，将 adb 相关目录加入扫描排除列表
  - 避免在 Docker Desktop 运行时同时进行 USB 设备调试
  - 启用 `ADB_TRACE=all` 环境变量可在问题发生时获取更详细的 server 日志

## 7. 参考文档 (References)

- [Android Debug Bridge (adb) — Android Developers](https://developer.android.com/tools/adb) — ADB 架构说明（client-server-daemon 模型）
- [WinUSB Installation — Microsoft Learn](https://learn.microsoft.com/en-us/windows-hardware/drivers/usbcon/winusb-installation) — WinUSB 驱动安装与工作机制
- [Process Monitor — Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon) — ProcMon 工具使用指南

---

# Case Summary: ADB Server Thread Blocking — USB Device Enumeration Deadlock Analysis

**Product/Service:** Android Debug Bridge (ADB) on Windows 10/11

---

## 1. Symptoms

- Customer ran `adb devices` on a Windows machine and the command **hung indefinitely** without returning the device list
- Multiple retries (including `adb kill-server` and restart) did not resolve the issue
- Problem persisted continuously; attempted fixes failed

## 2. Background / Environment

- **OS**: Windows 10/11 Enterprise (Build 26100.7840, amd64)
- **ADB Path**: `E:\Software\UIAutomation\platform-tools-latest-windows2\platform-tools\adb.exe` (running in WoW64 mode)
- **Installed Software**:
  - Docker Desktop (`kubernetes.docker.internal` as localhost alias)
  - McAfee Endpoint Security (mcshield.exe real-time scanning + mfemactl.exe management agent)
  - Enterprise endpoint management software (CGEData/CGEComm/CGESA suite)
  - Feishu, Chrome, IntelliJ IDEA, etc.
- **USB Devices**: Android device connected via USB for UI automation testing

## 3. Investigation & Troubleshooting

Customer captured **Process Monitor (ProcMon) PML trace** and **ETW network trace** during a failed repair attempt.

### 3.1 ProcMon Analysis — Process Identification

1. **Identified adb process tree**: 5 adb.exe processes found in the PML file:

   | PID | Role | Parent PID | Notes |
   |-----|------|-----------|-------|
   | 75708 | Server parent | 29056 | adb server daemon |
   | 9856 | **Server worker** | 75708 | ⚠️ **The stuck process** |
   | 58532 | Client #1 | 107596 | Old client, exited at 15:02:41 |
   | 56312 | Client #2 | 107596 | New client, 15:02:59~15:03:09 |
   | 59972 | Client #3 | 107596 | Appeared at 15:05:16 |

### 3.2 ProcMon Analysis — TCP Communication Timeline

2. **Reconstructed TCP communication flow**:
   - `15:02:59.180` — Client (PID 56312) connected via port 58567 to Server port 5037
   - `15:02:59.180` — Server (PID 9856) **TCP Accept** + **TCP Receive** received 16-byte request
   - 16 bytes = adb protocol `000chost:devices` request (4-digit hex length prefix + command string)
   - **Server went completely silent afterward**: zero file I/O, zero registry access, zero TCP sends

3. **Confirmed server total silence**: PID 9856 during the entire ProcMon trace (15:02:17 → 15:05:53, ~3.5 minutes) had **only Process_Profiling heartbeat events** (TID=0). All worker threads showed zero visible I/O activity.

### 3.3 ETW TCP Trace Analysis — Definitive Proof

4. **ETW trace verification** (15:04:23 → 15:06:10, ~106 seconds):
   - Connection `127.0.0.1:5037 → 127.0.0.1:58624` remained in EstablishedState
   - All server TCP send events: **BytesSent = 0** (only TCP ACKs, never returned application data)
   - Generated **466 "Duplicate Segment" drops** (TCP keepalive retransmission loop)
   - At 15:04:51, another new client from port 60138 sent 16-byte request, **also no response**

   **Conclusion: The server never returned a single byte of application data to any client during the entire observation period (>3 minutes).**

### 3.4 McAfee Real-time Scanning Behavior

5. **McAfee mcshield.exe (PID 6280) was abnormally active**:
   - At the **exact moment** of adb client startup (15:02:59), McAfee generated **248 file scanning operations** on the `platform-tools` directory
   - Scan targets included: `adb.exe`, `AdbWinApi.dll`, `IPHLPAPI.DLL`
   - Operation types: QueryOpen, CreateFile, QueryDirectory, CloseFile

### 3.5 DLL Loading Analysis

6. **Client PID 56312's DLL loading sequence** confirmed USB subsystem involvement:
   - `AdbWinApi.dll` — ADB USB interface library
   - `AdbWinUsbApi.dll` — ADB WinUSB API adaptation layer
   - `winusb.dll` — Windows USB user-mode driver interface
   - `mswsock.dll` — Windows Socket extensions
   - `IPHLPAPI.DLL` — IP Helper API

### 3.6 WUDFHost.exe Activity

7. **WUDFHost.exe (PID 1564)**: The USB user-mode driver host generated 190 events during the critical window, primarily Intel DPTF thermal management file I/O. While not directly related to ADB USB communication, it indicates the USB subsystem was actively working.

## 4. Blockers & How They Were Resolved

| Blocker | Impact | Resolution |
|---------|--------|-----------|
| ProcMon cannot capture USB DeviceIoControl calls | Cannot directly see which USB call the server is blocked on | Used elimination method (zero events = blocked in kernel-level call) to infer USB enumeration as the blocking source |
| ProcMon stack addresses not resolved to symbols | Cannot precisely locate the blocking function | Need procdump to capture process dump for further analysis |
| ETW trace started after ProcMon | Missed the initial TCP connection establishment | Supplemented early timeline using ProcMon TCP events |
| Unable to confirm USB device status | Unknown if abnormal USB devices caused enumeration hang | Recommended customer provide Device Manager screenshot |

## 5. Root Cause & Resolution

### Root Cause

**The ADB server's USB device enumeration thread was stuck in a WinUSB/kernel driver call, causing the `transport_lock` mutex to be held indefinitely, which in turn blocked all client request handler threads.**

Detailed mechanism:

```
USB Tracker Thread (holding transport_lock):
  usb_discover()
    → WinUsb_* API call
      → DeviceIoControl → Kernel USB driver stack
        → ⚠️ USB device unresponsive / driver-level deadlock
        → Thread permanently blocked, lock cannot be released

Client Handler Thread (waiting for transport_lock):
  host_devices()
    → Attempt to acquire transport_lock
      → ⚠️ Held by USB tracker thread, infinite wait
      → Client never receives response
```

Contributing factors:
- **McAfee real-time scanning** (248 scan operations) may delay USB device file/pipe access
- **Multiple enterprise security software layers** add system call interception latency
- **Docker Desktop** may compete for USB resources

### Resolution

Issue is in investigation phase. Recommended resolution steps:

1. **Capture adb server process dump** to confirm thread call stacks:
   ```
   procdump -ma <adb_server_pid> C:\temp\adb_dump.dmp
   ```
2. **Check USB device status**: Open Device Manager, look for USB devices with yellow exclamation marks
3. **Try unplugging USB devices**: Physically disconnect all USB Android devices, observe if server recovers
4. **Restart adb server**:
   ```
   adb kill-server
   taskkill /F /IM adb.exe    (ensure all instances are terminated)
   adb start-server
   adb devices
   ```

### Workaround

- Add `platform-tools` directory to McAfee exclusion list to reduce scanning interference
- Temporarily close Docker Desktop to eliminate USB resource contention
- Use `adb tcpip 5555` to switch to WiFi debugging mode, bypassing USB enumeration

## 6. Lessons Learned

- **Technical Knowledge**:
  - ADB server uses a `transport_lock` mutex to protect the device transport list; a stuck USB tracker thread causes cascading blocking of all client requests
  - ProcMon **cannot capture** kernel-level device I/O calls like USB DeviceIoControl; when a process is "completely silent" in ProcMon (only Process_Profiling), it typically means threads are blocked in kernel-mode calls
  - The adb `host:devices` request format is 16 bytes: `000chost:devices` (4-digit hex length + 12-character command)

- **Troubleshooting Methods**:
  - **ProcMon + ETW combined analysis** is very effective: ProcMon provides process/thread/file/registry dimensions, ETW provides precise byte-level TCP evidence
  - **"Zero events" is itself key evidence**: Server receiving request followed by zero I/O activity = blocked in invisible kernel call
  - **Multiple clients all timing out** is an important signal for server-level issues (vs. single client issues)
  - When analyzing PML files, use the `procmon-parser` Python library to quickly extract key processes, then do targeted deep analysis

- **Prevention**:
  - In environments with enterprise security software (McAfee etc.), add adb-related directories to scan exclusion lists
  - Avoid USB device debugging while Docker Desktop is running
  - Enable `ADB_TRACE=all` environment variable to get more detailed server logs when issues occur

## 7. References

- [Android Debug Bridge (adb) — Android Developers](https://developer.android.com/tools/adb) — ADB architecture (client-server-daemon model)
- [WinUSB Installation — Microsoft Learn](https://learn.microsoft.com/en-us/windows-hardware/drivers/usbcon/winusb-installation) — WinUSB driver installation and mechanism
- [Process Monitor — Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon) — ProcMon tool usage guide
