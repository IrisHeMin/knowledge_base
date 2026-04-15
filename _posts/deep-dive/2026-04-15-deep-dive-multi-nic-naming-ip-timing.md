---
layout: post
title: "Deep Dive: 多网卡场景 — 命名规则与 IP 获取时序分析"
date: 2026-04-15
categories: [Knowledge, Networking]
tags: [ndis, pci, dhcp, miniport, nic-naming, multi-nic, kvm, virtualization, windows-networking]
type: "deep-dive"
---

# Deep Dive: 多网卡场景 — 命名规则与 IP 获取时序分析

**Topic:** Windows Multi-NIC Naming Rules & IP Acquisition Timing  
**Category:** Networking / NDIS / DHCP  
**Level:** 中高级 / Intermediate-Advanced  
**Last Updated:** 2026-04-15

---

# 中文版

## 前置知识：什么是 PCI？什么是 PCI Slot？

在深入讨论网卡命名规则之前，我们需要先理解 PCI 和 PCI Slot 这两个基础概念，因为它们是 Windows 设备枚举和命名的底层基础。

### 什么是 PCI？

**PCI（Peripheral Component Interconnect，外设互连标准）** 是计算机中 CPU 与外部硬件设备之间通信的总线标准。无论是物理机还是虚拟机，操作系统要和网卡、显卡、磁盘控制器等硬件设备交互，都需要通过某种总线协议。PCI（以及它的升级版 **PCIe — PCI Express**）是当今最主流的总线标准。

```
┌──────────┐                              ┌──────────────┐
│          │    PCI / PCIe 总线（通信公路）    │  网卡 (NIC)   │
│   CPU    │◄════════════════════════════►│  显卡 (GPU)   │
│          │                              │  磁盘控制器    │
└──────────┘                              └──────────────┘
```

在 Windows 设备管理器中，你会看到设备的硬件 ID 格式为 `PCI\VEN_XXXX&DEV_XXXX`，这就是 PCI 体系下的设备标识。其中 `VEN` 是厂商 ID，`DEV` 是设备 ID。

> 参考：[Supported Ethernet NICs for Network Kernel Debugging](https://learn.microsoft.com/windows-hardware/drivers/debugger/supported-ethernet-nics-for-network-kernel-debugging-in-windows-10)
> *"The vendor and device IDs are shown as VEN\_VendorID and DEV\_DeviceID. For example, if you see PCI\VEN\_8086&DEV\_104B, the vendor ID is 8086, and the device ID is 104B."*

### 什么是 PCI Slot？

在物理机上，PCI Slot 就是主板上的**物理插槽** — 你把网卡插到哪个槽，它就占哪个 slot。

但更准确地说，PCI 设备的"位置"用 **BDF（Bus:Device.Function）三元组**来标识：

```
PCI 地址格式：  Domain : Bus : Device . Function
示例：          0000   : 00  : 03     . 0
                │        │     │        │
                │        │     │        └─ Function（同一设备的子功能，多数为 0）
                │        │     └────────── Device（≈ 就是所谓的 "Slot 号"）
                │        └──────────────── Bus（总线编号）
                └───────────────────────── Domain（通常为 0000）
```

**所以 "PCI Slot" 本质上就是 BDF 中的 Device 编号**，代表设备在总线上的位置编号。

> 参考：[Hard Disk Location Path Format](https://learn.microsoft.com/windows-hardware/manufacture/desktop/hard-disk-location-path-format)
> 位置路径格式示例：`PCIROOT(0)#PCI(0300)` — 其中 `0300` 表示 Bus 0、Device 03、Function 0。

### 物理机 vs 虚拟机：谁决定 PCI Slot？

**关键区别**在于谁控制 PCI 拓扑的分配：

| 场景 | 谁决定 PCI Slot？ | 说明 |
|------|-------------------|------|
| **物理机** | 主板硬件设计 + 物理插槽位置 | 网卡插到哪个物理槽，就是哪个 Device 号 |
| **虚拟机** | 虚拟化平台（Hypervisor）| 虚拟化平台在创建 VM 时分配虚拟 PCI 地址 |

**但无论 PCI Slot 由谁分配，Windows 侧的行为是完全一致的：**

1. Windows PnP 管理器让 PCI 总线驱动（`Pci.sys`）枚举设备
2. 枚举顺序基于 BDF 编号
3. 枚举顺序决定了 NET_LUID 分配和网卡命名

> 参考：[Device nodes and device stacks - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/kernel/device-nodes-and-device-stacks)
> *"During startup, the PnP manager asks the driver for each bus to enumerate child devices that are connected to the bus. For example, the PnP manager asks the PCI bus driver (Pci.sys) to enumerate the devices that are connected to the PCI bus."*

**简单来说：PCI 是"公路"，PCI Slot 是"门牌号"，Windows 按门牌号从小到大依次"敲门"（枚举），先敲到的网卡就叫"以太网"，后敲到的叫"以太网 2"。至于门牌号由谁编排 — 物理机由主板决定，虚拟机由 Hypervisor 决定 — 这部分不影响 Windows 的行为逻辑。**

---

## 问题 1：多网卡的命名规则是什么？是否与 PCI Slot 有关？

### 结论

是的，Windows 中网卡的连接名称（如"以太网"、"以太网 2"）与设备在 PCI 总线上的位置（PCI Slot / BDF 编号）密切相关。

### Windows 侧的命名机制

Windows 的网络适配器命名涉及以下几个关键标识符：

| 标识符 | 说明 | 是否可预测 |
|--------|------|-----------|
| **PnP location (PCI slot number)** | PCI 设备的物理/虚拟位置 | ✅ 是 |
| **NET_LUID (NetLuidIndex + IfType)** | 网卡的本地唯一标识，跨重启持久 | 否 |
| **ifAlias** | 用户可见的连接名称，如"以太网" | 有时可预测 |
| **ifIndex** | 接口索引，跨重启不保证稳定 | 否 |

根据微软官方文档 [Network interfaces](https://learn.microsoft.com/windows/win32/network-interfaces) 的描述：

> **PnP location (PCI slot number)** — Is unique on the system: **Yes**; Is predictable: **Yes**; Persists across reboots: **Yes**.

> **ifAlias** — Is predictable: **Sometimes** (Note 5: "Only if the firmware supports Consistent Device Naming. Typically, servers have this feature.")

**完整的命名流程如下：**

1. **PnP 管理器枚举设备**：启动时，PnP 管理器请求 PCI 总线驱动（`Pci.sys`）枚举总线上的子设备。PCI 总线驱动为每个设备创建设备对象（Device Node）。

   > 参考：[Device nodes and device stacks - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/kernel/device-nodes-and-device-stacks)
   > *"During startup, the PnP manager asks the driver for each bus to enumerate child devices that are connected to the bus. For example, the PnP manager asks the PCI bus driver (Pci.sys) to enumerate the devices that are connected to the PCI bus."*

2. **NDIS 分配 NET_LUID**：NDIS 为每个 miniport adapter 调用 `NdisIfAllocateNetLuidIndex` 分配一个 NetLuidIndex，该索引按枚举顺序递增，跨重启持久。

   > 参考：[NET_LUID Value - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/network/net-luid-value)
   > *"The NetLuidIndex member of the NET_LUID union is a 24-bit NET_LUID index that NDIS allocates when an interface provider calls the NdisIfAllocateNetLuidIndex function. NDIS and interface providers use this index to distinguish between multiple interfaces that have the same interface type."*

3. **NetCfg 分配 ifAlias（连接名称）**：NetCfg 组件负责确保每个 NIC 的 ifAlias 唯一且非空。第一个同类型网卡通常命名为"以太网"（Ethernet），后续同类型网卡追加编号"以太网 2"、"以太网 3"。

   > 参考：[Network interfaces](https://learn.microsoft.com/windows/win32/network-interfaces)
   > *"NetCfg enforces that an ifAlias is a non-empty string and is unique among all NICs."* (Note 4)

**由于 PCI 总线驱动按 BDF (Bus:Device.Function) 顺序枚举设备，BDF 编号较小的设备先被枚举，因此先获得较小的 NetLuidIndex，并先被分配基础名称（如 "以太网"）。BDF 编号较大的设备后被枚举，获得带编号后缀的名称（如 "以太网 2"）。**

### 验证方法

```powershell
# 查看网卡名称与 PnP Instance ID 的对应关系
Get-NetAdapter | Select-Object Name, InterfaceIndex, PnPDeviceID, MacAddress | Sort-Object InterfaceIndex

# 查看更清晰的 PCI Location Path
Get-PnpDevice -Class Net -Status OK |
  Select-Object FriendlyName, InstanceId,
    @{N='LocationPath'; E={(Get-PnpDeviceProperty -InstanceId $_.InstanceId `
      -KeyName DEVPKEY_Device_LocationPaths).Data}} |
  Format-List
```

**PnPDeviceID 末尾的数字**即为 PCI Device 编号（Slot），例如 `PCI\VEN_1AF4&DEV_1041&...\4&abc123&0&03` 中的 `03`。该数字越小的网卡，名称编号越靠前。

### 关于虚拟化平台侧

PCI Slot 的分配方式取决于虚拟化平台的实现。**这部分属于虚拟化平台的实现细节，建议咨询虚拟化平台侧确认其 PCI 拓扑的分配策略。** Windows 侧只关心最终呈现的 PCI BDF 编号，并据此枚举和命名。

---

## 问题 2：先初始化完成的网卡为什么反而迟获取到 IP？

### 结论

Miniport 驱动初始化完成只是网卡从"无"到"可用"的第一步。从 miniport 初始化完成到 IP 地址生效，还需要经过多个异步阶段，任何一个阶段的时序差异都可能导致先初始化的网卡反而更晚获取 IP。

### 从 Miniport 初始化到 IP 生效的完整链路

```
(1) Miniport 驱动初始化 (MiniportInitializeEx)
     ↓
(2) NDIS 协议绑定 (ProtocolBindAdapterEx — TCPIP 协议栈绑定到适配器)
     ↓
(3) Media Connect 状态指示 (NDIS_STATUS_LINK_STATE — 网卡报告链路已连接)
     ↓
(4) DHCP Client 服务发送 DHCP Discover
     ↓
(5) DHCP ACK 收到后，客户端执行重复地址检测 (Gratuitous ARP)
     ↓
(6) IP 地址生效
```

以下逐一分析可能导致时序反转的环节：

---

#### 环节 (1) → (2)：NDIS 协议绑定是异步的

NDIS 在 miniport adapter 可用后，调用协议驱动的 `ProtocolBindAdapterEx` 回调来建立绑定。**这个过程是异步的，可以返回 NDIS_STATUS_PENDING。**

> 参考：[Binding to an Adapter - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/network/binding-to-an-adapter)
> *"NDIS calls a protocol driver's ProtocolBindAdapterEx function to open a binding whenever an underlying adapter to which the driver can bind becomes available."*
> *"If a protocol driver returns NDIS_STATUS_PENDING from ProtocolBindAdapterEx, it must call NdisCompleteBindAdapterEx with the final status to complete the bind request."*

> 参考：[Binding States of a Protocol Driver](https://learn.microsoft.com/windows-hardware/drivers/network/binding-states-of-a-protocol-driver)
> 绑定状态机：Unbound → Opening → Paused → Running

**影响：** 即使网卡 A 的 miniport 先初始化完成，TCPIP 协议栈对网卡 A 和网卡 B 的绑定操作是独立的异步过程，不保证先后顺序与 miniport 初始化顺序一致。

---

#### 环节 (2) → (3)：Media Connect 状态指示的时机

Miniport 驱动通过 `NDIS_STATUS_LINK_STATE` 状态指示来通知上层链路状态变化。**只有当链路状态为 `MediaConnectStateConnected` 时，上层协议才会开始 DHCP 流程。**

> 参考：[NDIS_STATUS_LINK_STATE](https://learn.microsoft.com/windows-hardware/drivers/network/ndis-status-link-state)
> *"Miniport drivers use the NDIS_STATUS_LINK_STATE status indication to notify NDIS and overlying drivers that there has been a change in the physical characteristics of a medium."*

**影响：** 在虚拟化场景中，网卡的 link-up 通知取决于虚拟化后端（如虚拟交换机）的状态。即使 miniport 初始化完成，如果 link-up 事件晚到达，DHCP 流程就不会启动。**这部分时序取决于虚拟化平台的 link 状态通知机制，建议向平台侧确认。**

---

#### 环节 (4)：DHCP Client 服务的调度

Windows 的 DHCP Client 是一个**单一系统服务**（`svc: Dhcp`），它为所有启用 DHCP 的网卡处理地址获取请求。当多个网卡几乎同时进入"需要获取 IP"的状态时，DHCP Client 内部的处理顺序由其自身调度逻辑决定，不保证与 miniport 初始化顺序一致。

> 参考：[Troubleshoot problems on the DHCP client](https://learn.microsoft.com/troubleshoot/windows-server/networking/troubleshoot-problems-dhcp-client)

此外，还有一个已知问题：`AFD.SYS` 的启动时机会影响 DHCP 服务及其他网络服务的就绪时间。

> 参考：[Network adapter fails to acquire IP at startup (KB3139296)](https://learn.microsoft.com/troubleshoot/windows-hardware/drivers/network-adapter-fails-acquire-ip-address)
> *"The default value for the HKLM\SYSTEM\CurrentControlSet\Services\AFD registry key with the REG_DWORD value that's named Start is 0x2. This setting causes the AFD.SYS service to load late in the startup process. In turn, delays the startup of the DHCP service."*

---

#### 环节 (5)：重复地址检测 (DAD / Gratuitous ARP)

当 DHCP 客户端收到 DHCP ACK 后，**在正式使用该 IP 地址之前，Windows 会发送 Gratuitous ARP 请求来做客户端侧的冲突检测。**

> 参考：[Event ID 4199 and Windows client can't get IP from DHCP](https://learn.microsoft.com/troubleshoot/windows-server/networking/event-4199-windows-client-cannot-get-ip-address-dhcp-server)
> *"Windows DHCP clients that obtain an IP address use a gratuitous ARP request to perform a client-based conflict detection before completing configuration and using the server-offered IP address."*
> *"If the switch or router sends out an ARP probe for the client while the Windows client is in the duplicate address detection (DAD) phase, the client detects the probe as a duplicate IP address and declines the IP address offered by the DHCP server."*

**影响：** 如果网卡 A 的 DAD 阶段被某些网络设备（如启用了 IP Device Tracking 的交换机）干扰，它可能 decline 地址并重新请求，导致 IP 生效时间大幅延后。

---

### 排查建议

```powershell
# 1. 检查各网卡的接口信息和 DHCP 状态
Get-NetIPInterface | Sort-Object InterfaceMetric |
  Select-Object ifIndex, InterfaceAlias, AddressFamily, InterfaceMetric, Dhcp

# 2. 查看 DHCP 客户端事件日志，对比各网卡获取 IP 的时间戳
Get-WinEvent -LogName "Microsoft-Windows-Dhcp-Client/Admin" -MaxEvents 30 |
  Select-Object TimeCreated, Id, Message | Format-Table -AutoSize

# 3. 查看系统日志中 NDIS 相关事件
Get-WinEvent -LogName "System" -MaxEvents 100 |
  Where-Object { $_.ProviderName -match "NDIS|Tcpip|Dhcp" } |
  Select-Object TimeCreated, ProviderName, Id, Message | Format-Table -AutoSize

# 4. 精确时序分析（需要复现问题时使用）
netsh trace start scenario=netconnection capture=yes tracefile=C:\temp\nettrace.etl
# 复现问题后...
netsh trace stop
```

### 总结

| 阶段 | 可能导致时序反转的原因 | 属于 Windows 还是平台侧 |
|------|----------------------|----------------------|
| Miniport 初始化 → 协议绑定 | `ProtocolBindAdapterEx` 异步执行，绑定顺序不保证 | **Windows (NDIS)** |
| 协议绑定 → Link Up | 虚拟网卡的 link-up 状态通知时机 | **虚拟化平台侧** |
| Link Up → DHCP Discover | DHCP Client 服务的内部调度顺序；AFD.SYS 加载时机 | **Windows (DHCP/AFD)** |
| DHCP ACK → IP 生效 | Gratuitous ARP / DAD 阶段被网络设备干扰 | **网络环境** |

---

## 参考文献

1. [Network interfaces - Win32 apps](https://learn.microsoft.com/windows/win32/network-interfaces) — 网络接口标识符属性表
2. [Device nodes and device stacks - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/kernel/device-nodes-and-device-stacks) — PnP 管理器枚举 PCI 总线设备
3. [NET_LUID Value - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/network/net-luid-value) — NET_LUID 的分配机制
4. [Initializing a Miniport Driver - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/network/initializing-a-miniport-driver) — Miniport 驱动初始化流程
5. [Binding to an Adapter - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/network/binding-to-an-adapter) — NDIS 协议绑定过程
6. [Binding States of a Protocol Driver - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/network/binding-states-of-a-protocol-driver) — 协议驱动绑定状态机
7. [NDIS_STATUS_LINK_STATE - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/network/ndis-status-link-state) — 链路状态指示
8. [Configure the Order of Network Interfaces - Windows Server](https://learn.microsoft.com/windows-server/networking/technologies/network-subsystem/net-sub-interface-metric) — 网络接口顺序与 InterfaceMetric
9. [Event ID 4199 - Windows client can't get IP from DHCP](https://learn.microsoft.com/troubleshoot/windows-server/networking/event-4199-windows-client-cannot-get-ip-address-dhcp-server) — DAD/Gratuitous ARP 冲突
10. [Network adapter fails to acquire IP at startup (KB3139296)](https://learn.microsoft.com/troubleshoot/windows-hardware/drivers/network-adapter-fails-acquire-ip-address) — AFD.SYS 加载时机影响 DHCP
11. [Troubleshoot problems on the DHCP client](https://learn.microsoft.com/troubleshoot/windows-server/networking/troubleshoot-problems-dhcp-client) — DHCP 客户端排查指南
12. [DHCP troubleshooting guidance](https://learn.microsoft.com/troubleshoot/windows-server/networking/troubleshoot-dhcp-guidance) — DHCP 综合排查指南

---

# English Version

## Background: What Is PCI? What Is a PCI Slot?

Before diving into NIC naming rules, we need to understand PCI and PCI Slot — they are the foundation of how Windows enumerates and names devices.

### What Is PCI?

**PCI (Peripheral Component Interconnect)** is the bus standard for communication between the CPU and external hardware devices. Whether on physical or virtual machines, the OS interacts with NICs, GPUs, and disk controllers through a bus protocol. PCI (and its successor **PCIe — PCI Express**) is the most widely used bus standard today.

```
┌──────────┐                                    ┌──────────────┐
│          │    PCI / PCIe Bus (communication    │  NIC          │
│   CPU    │◄══════════════════════════════════►│  GPU          │
│          │           highway)                  │  Disk Ctrl    │
└──────────┘                                    └──────────────┘
```

In Windows Device Manager, you'll see hardware IDs like `PCI\VEN_XXXX&DEV_XXXX` — this is the PCI identification scheme. `VEN` is the vendor ID, `DEV` is the device ID.

> Reference: [Supported Ethernet NICs for Network Kernel Debugging](https://learn.microsoft.com/windows-hardware/drivers/debugger/supported-ethernet-nics-for-network-kernel-debugging-in-windows-10)
> *"The vendor and device IDs are shown as VEN\_VendorID and DEV\_DeviceID. For example, if you see PCI\VEN\_8086&DEV\_104B, the vendor ID is 8086, and the device ID is 104B."*

### What Is a PCI Slot?

On physical machines, a PCI Slot is a **physical connector on the motherboard** — whichever slot you plug the NIC into determines its slot number.

More precisely, a PCI device's "location" is identified by the **BDF (Bus:Device.Function) triplet**:

```
PCI Address Format:  Domain : Bus : Device . Function
Example:             0000   : 00  : 03     . 0
                     │        │     │        │
                     │        │     │        └─ Function (sub-function, usually 0)
                     │        │     └────────── Device (≈ the "Slot number")
                     │        └──────────────── Bus (bus number)
                     └───────────────────────── Domain (usually 0000)
```

**So "PCI Slot" is essentially the Device number in the BDF triplet**, representing the device's position on the bus.

> Reference: [Hard Disk Location Path Format](https://learn.microsoft.com/windows-hardware/manufacture/desktop/hard-disk-location-path-format)
> Location path format example: `PCIROOT(0)#PCI(0300)` — where `0300` means Bus 0, Device 03, Function 0.

### Physical vs Virtual: Who Decides the PCI Slot?

The **key difference** is who controls the PCI topology assignment:

| Scenario | Who Decides PCI Slot? | Description |
|----------|----------------------|-------------|
| **Physical machine** | Motherboard design + physical slot | Whichever slot you plug the NIC into |
| **Virtual machine** | Virtualization platform (Hypervisor) | The hypervisor assigns virtual PCI addresses when creating the VM |

**Regardless of who assigns the PCI Slot, Windows behavior is identical:**

1. Windows PnP manager asks the PCI bus driver (`Pci.sys`) to enumerate devices
2. Enumeration order is based on BDF numbers
3. Enumeration order determines NET_LUID assignment and NIC naming

> Reference: [Device nodes and device stacks](https://learn.microsoft.com/windows-hardware/drivers/kernel/device-nodes-and-device-stacks)
> *"During startup, the PnP manager asks the driver for each bus to enumerate child devices that are connected to the bus."*

**In simple terms: PCI is the "highway", PCI Slot is the "house number". Windows knocks on doors (enumerates) from smallest to largest number — the first NIC it finds becomes "Ethernet", the next becomes "Ethernet 2". Who assigns the house numbers — the motherboard (physical) or the hypervisor (virtual) — does not change Windows' behavior.**

---

## Question 1: What Are the NIC Naming Rules? Is It Related to PCI Slot?

### Conclusion

Yes, the Windows network adapter connection name (e.g., "Ethernet", "Ethernet 2") is closely related to the device's position on the PCI bus (PCI Slot / BDF number).

### Windows NIC Naming Mechanism

Key network interface identifiers in Windows:

| Identifier | Description | Predictable? |
|------------|-------------|-------------|
| **PnP location (PCI slot number)** | Physical/virtual PCI location | ✅ Yes |
| **NET_LUID (NetLuidIndex + IfType)** | Locally unique ID, persists across reboots | No |
| **ifAlias** | User-visible connection name | Sometimes |
| **ifIndex** | Interface index, not guaranteed stable across reboots | No |

Per Microsoft documentation [Network interfaces](https://learn.microsoft.com/windows/win32/network-interfaces):

> **PnP location (PCI slot number)** — Is unique on the system: **Yes**; Is predictable: **Yes**; Persists across reboots: **Yes**.

> **ifAlias** — Is predictable: **Sometimes** (Note 5: "Only if the firmware supports Consistent Device Naming. Typically, servers have this feature.")

**The complete naming flow:**

1. **PnP Manager enumerates devices**: At startup, the PnP manager asks the PCI bus driver (`Pci.sys`) to enumerate child devices on the bus. The PCI bus driver creates a device object (Device Node) for each device.

   > Reference: [Device nodes and device stacks](https://learn.microsoft.com/windows-hardware/drivers/kernel/device-nodes-and-device-stacks)

2. **NDIS allocates NET_LUID**: NDIS allocates a NetLuidIndex for each miniport adapter by calling `NdisIfAllocateNetLuidIndex`. The index increments with enumeration order and persists across reboots.

   > Reference: [NET_LUID Value](https://learn.microsoft.com/windows-hardware/drivers/network/net-luid-value)

3. **NetCfg assigns ifAlias (connection name)**: The NetCfg component ensures each NIC's ifAlias is unique and non-empty. The first NIC of a given type is named "Ethernet", subsequent ones get "Ethernet 2", "Ethernet 3", etc.

   > Reference: [Network interfaces](https://learn.microsoft.com/windows/win32/network-interfaces) — Note 4

**Since the PCI bus driver enumerates devices by BDF (Bus:Device.Function) order, devices with smaller BDF numbers are enumerated first, receiving smaller NetLuidIndex values and the base name (e.g., "Ethernet"). Devices with larger BDF numbers get suffixed names (e.g., "Ethernet 2").**

### Regarding the Virtualization Platform

PCI Slot assignment depends on the virtualization platform's implementation. **This is a platform-specific detail — consult your virtualization platform vendor for their PCI topology assignment strategy.** Windows only cares about the resulting PCI BDF numbers and enumerates/names accordingly.

---

## Question 2: Why Does the First-Initialized NIC Get Its IP Address Later?

### Conclusion

Miniport driver initialization is only the first step. From miniport initialization to IP address becoming effective, there are multiple asynchronous stages. Timing differences in any stage can cause the first-initialized NIC to acquire its IP later.

### Complete Chain from Miniport Initialization to IP Effective

```
(1) Miniport driver initialization (MiniportInitializeEx)
     ↓
(2) NDIS protocol binding (ProtocolBindAdapterEx — TCPIP binds to adapter)
     ↓
(3) Media Connect status indication (NDIS_STATUS_LINK_STATE — NIC reports link up)
     ↓
(4) DHCP Client service sends DHCP Discover
     ↓
(5) After DHCP ACK, client performs Duplicate Address Detection (Gratuitous ARP)
     ↓
(6) IP address becomes effective
```

#### Stage (1) → (2): NDIS Protocol Binding Is Asynchronous

NDIS calls the protocol driver's `ProtocolBindAdapterEx` callback to establish binding after a miniport adapter becomes available. **This process is asynchronous and can return NDIS_STATUS_PENDING.**

> Reference: [Binding to an Adapter](https://learn.microsoft.com/windows-hardware/drivers/network/binding-to-an-adapter)

**Impact:** Even if NIC A's miniport initializes first, the TCPIP protocol stack's binding to NIC A and NIC B are independent async operations — the order is not guaranteed.

#### Stage (2) → (3): Media Connect Indication Timing

Miniport drivers notify upper layers of link state changes via `NDIS_STATUS_LINK_STATE`. **Only when link state is `MediaConnectStateConnected` will upper protocols initiate DHCP.**

> Reference: [NDIS_STATUS_LINK_STATE](https://learn.microsoft.com/windows-hardware/drivers/network/ndis-status-link-state)

**Impact:** In virtualization scenarios, the link-up notification depends on the virtual backend. **This timing depends on the virtualization platform's link state notification mechanism.**

#### Stage (4): DHCP Client Service Scheduling

Windows DHCP Client is a **single system service** handling all DHCP-enabled adapters. When multiple NICs need IP addresses simultaneously, the internal processing order is determined by its own scheduling logic.

> Reference: [Troubleshoot problems on the DHCP client](https://learn.microsoft.com/troubleshoot/windows-server/networking/troubleshoot-problems-dhcp-client)

Additionally, `AFD.SYS` loading timing can affect DHCP service readiness.

> Reference: [Network adapter fails to acquire IP at startup (KB3139296)](https://learn.microsoft.com/troubleshoot/windows-hardware/drivers/network-adapter-fails-acquire-ip-address)

#### Stage (5): Duplicate Address Detection (DAD / Gratuitous ARP)

After receiving DHCP ACK, **Windows sends a Gratuitous ARP request for client-side conflict detection before using the IP address.**

> Reference: [Event ID 4199](https://learn.microsoft.com/troubleshoot/windows-server/networking/event-4199-windows-client-cannot-get-ip-address-dhcp-server)

**Impact:** If NIC A's DAD phase is interfered with by network devices (e.g., switches with IP Device Tracking enabled), it may decline the address and re-request, significantly delaying IP effectiveness.

### Summary

| Stage | Possible Cause of Timing Reversal | Windows or Platform Side |
|-------|----------------------------------|------------------------|
| Miniport Init → Protocol Binding | `ProtocolBindAdapterEx` async, order not guaranteed | **Windows (NDIS)** |
| Protocol Binding → Link Up | Virtual NIC link-up notification timing | **Virtualization Platform** |
| Link Up → DHCP Discover | DHCP Client scheduling; AFD.SYS load timing | **Windows (DHCP/AFD)** |
| DHCP ACK → IP Effective | Gratuitous ARP / DAD interference by network devices | **Network Environment** |

---

## References

1. [Network interfaces - Win32 apps](https://learn.microsoft.com/windows/win32/network-interfaces)
2. [Device nodes and device stacks - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/kernel/device-nodes-and-device-stacks)
3. [NET_LUID Value - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/network/net-luid-value)
4. [Initializing a Miniport Driver - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/network/initializing-a-miniport-driver)
5. [Binding to an Adapter - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/network/binding-to-an-adapter)
6. [Binding States of a Protocol Driver - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/network/binding-states-of-a-protocol-driver)
7. [NDIS_STATUS_LINK_STATE - Windows drivers](https://learn.microsoft.com/windows-hardware/drivers/network/ndis-status-link-state)
8. [Configure the Order of Network Interfaces - Windows Server](https://learn.microsoft.com/windows-server/networking/technologies/network-subsystem/net-sub-interface-metric)
9. [Event ID 4199 - Windows client can't get IP from DHCP](https://learn.microsoft.com/troubleshoot/windows-server/networking/event-4199-windows-client-cannot-get-ip-address-dhcp-server)
10. [Network adapter fails to acquire IP at startup (KB3139296)](https://learn.microsoft.com/troubleshoot/windows-hardware/drivers/network-adapter-fails-acquire-ip-address)
11. [Troubleshoot problems on the DHCP client](https://learn.microsoft.com/troubleshoot/windows-server/networking/troubleshoot-problems-dhcp-client)
12. [DHCP troubleshooting guidance](https://learn.microsoft.com/troubleshoot/windows-server/networking/troubleshoot-dhcp-guidance)
