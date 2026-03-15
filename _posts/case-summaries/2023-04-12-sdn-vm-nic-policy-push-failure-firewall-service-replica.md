---
layout: post
title: "Case Summary: SDN VM 网络适配器策略推送失败 - Firewall Service 主副本异常"
date: 2023-04-12
categories: [SDN, Azure-Stack-HCI]
tags: [sdn, network-controller, firewall-service, vfp, port-blocked, service-fabric, move-replica, ncha, sdnfw, sdnvsm, policy-push, vm-networking]
---

# Case Summary: SDN VM 网络适配器策略推送失败 — Firewall Service 主副本异常

**Product/Service:** Azure Stack HCI / Windows Server SDN (Software Defined Networking)

---

## 1. 症状 (Symptoms)

- 客户通过 WAC (Windows Admin Center) 设置 VM 网络适配器的虚拟网络时失败，UI 报错：**"couldn't save the network settings"**
- VM 无法获取虚拟 IP 地址
- 该错误为纯 SDN 策略推送问题，与 WAC UI 本身无关

## 2. 背景 (Background / Environment)

- **平台**：Azure Stack HCI，运行 SDN (Software Defined Networking) 环境
- **组件**：
  - Network Controller (NC) 集群：多节点部署
  - Hyper-V Host：运行目标 VM
  - 涉及的 SDN 微服务：SDNAPI、SDNVSM (Virtual Switch Manager)、SDNFW (Firewall Service)
- **关键网络连接（Host ↔ NC）**：
  - `NCHA → SDNAPI:6640`：Host Agent 向 NC 发送端口添加/移除通知
  - `SDNVSM/SDNFW → NCHA:6640`：NC 向 Host 推送 VNET 策略和防火墙策略
  - `NC (SDNFW/SDNVSM) → NCHA:443`：WCF 连接，用于推送 DHCP、DNS、PA 配置等策略
- **VM**：`sdn-vnet-vm01`，MAC 地址 `00-1D-D8-AA-1C-34`

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

### Step 1：确认网络适配器的真实配置状态

**做了什么**：通过 Network Controller REST API 导出网络接口 JSON 获取配置状态。

```powershell
Get-NetworkControllerNetworkInterface -ConnectionUri $uri | ConvertTo-Json -Depth 10 > C:\networkinterface.txt
```

通过 VM 的 MAC 地址定位正确的接口：

```powershell
Get-VMNetworkAdapter -VMName sdn-vnet-vm01
```

**发现了什么**：从 JSON 文件中获取了 3 条关键信息：

| 信息项 | 值 | 含义 |
|--------|-----|------|
| Instance ID | `d072ad2b-ff97-4823-a84f-8dd75c138590` | 后续 ETL 日志分析的关键标识 |
| Provisioning State | `Succeeded` | NC 成功创建了策略 |
| Configuration State | `Failure` | 策略推送到 Host **失败** |

错误详情：
```json
{
  "configurationState": {
    "status": "Failure",
    "detailedInfo": [{
      "source": "VirtualSwitch",
      "message": "The host has not yet established communication with the Network Controller.",
      "code": "HostNotConnectedToController"
    }]
  }
}
```

**得出的结论**：
- **Provisioning 成功 + Configuration 失败** = 策略在 NC 端已创建成功，但推送到 Host 的过程失败
- 错误消息 `HostNotConnectedToController` 指向 Host 与 NC 之间的网络连接问题

### Step 2：检查 Host 与 NC 之间的 6640/443 连接

**做了什么**：
- 通过 `Get-NetworkControllerReplica` 确认各微服务所在的 NC 节点
- 使用 `netstat -abno` 逐一检查所有关键连接

**发现了什么**：

| 连接 | 状态 |
|------|------|
| NCHA → SDNAPI:6640 | ✅ 正常 |
| SDNVSM → NCHA:6640 | ✅ 正常 |
| SDNFW → NCHA:6640 | ✅ 正常 |
| SDNVSM → NCHA:443 | ✅ 正常 |
| **SDNFW → NCHA:443** | **❌ 未建立** |

**得出的结论**：SDNFW 到 NCHA 的 443 TLS 连接未建立，这是策略推送失败的直接原因。

### Step 3：排查 443 连接失败原因

**做了什么**：按常规 TLS 连接问题排查三要素检查：
1. TCP 443 端口连通性 → ✅ 正常
2. SDNFW 服务健康状态：
   ```powershell
   Get-ServiceFabricServiceHealth -ServiceName fabric:/NetworkController/FirewallService
   # AggregatedHealthState: Ok
   ```
   → ✅ 正常
3. 证书/信任链 → ✅ 正常

**得出的结论**：三项基础检查均正常，需要进一步从 VM 端口状态和 NC ETL 日志深入分析。

### Step 4：检查 VM 网络接口的端口状态

**做了什么**：先获取端口信息再查询端口状态。

```powershell
# 获取 VM 网络适配器的 Port Profile
Get-SDNVmNetworkAdapterPortProfile -AllVMs
```

**发现了什么**：ProfileId 为空（全零 GUID），表示端口配置未被正确应用：

```
VMName      : sdn-vnet-vm01
MacAddress  : 001DD8AA1C34
ProfileId   : {00000000-0000-0000-0000-000000000000}   ← 空！
ProfileData : 2
PortId      : D267D29E-9E28-4546-BAF9-016437966ADC
```

检查端口状态：

```powershell
vfpctrl.exe /port D267D29E-9E28-4546-BAF9-016437966ADC /get-port-state
# PORT STATE
#   Enabled : FALSE
#   Blocked : TRUE    ← 端口被阻塞！
```

**得出的结论**：VM 网络端口被 VFP 阻塞（Blocked: TRUE），且 ProfileId 为空，说明 SDNFW 未成功将防火墙策略推送到 Host，导致端口保持默认阻塞状态。

> **补充**：尝试手动设置 ProfileId 到已知的 Instance ID：
> ```powershell
> Set-SdnVMNetworkAdapterPortProfile -VMName "sdn-vnet-vm01" -MACAddress 001DD8AA1C34 -ProfileID "d072ad2b-ff97-4823-a84f-8dd75c138590"
> ```
> 在其他类似 case 中此方法可以临时解决问题，但在本案例中未能立即生效。

### Step 5：分析 SDNFW NC ETL 日志

**做了什么**：从 SDNFW 所在的 NC 节点收集 ETL 日志（路径：`C:\Windows\tracing\SDNDiagnostics\Logs`）。

**发现了什么**：日志中出现两个关键错误：

**错误 1 — WCF 通信对象已被中止**：
```
ControllerTransientException: An internal error occurred in a WCF operation
---> System.ServiceModel.CommunicationObjectAbortedException:
The communication object, System.ServiceModel.Channels.ClientReliableDuplexSessionChannel,
cannot be used for communication because it has been Aborted.
```

**错误 2 — Service Fabric 主副本异常**：
```
Primary recovery task failed: System.AggregateException
---> ImosException: Network Controller operation failed
---> System.Fabric.FabricNotPrimaryException:
Must be primary to begin read/write session
```

**得出的结论**：SDNFW 服务的主副本（Primary Replica）虽然报告健康状态为 OK，但实际上出现了内部异常，WCF 通信通道被中止，导致无法与 Host Agent 建立 443 连接推送防火墙策略。

### Step 6：迁移 Firewall Service 主副本

**做了什么**：将 SDNFW 主副本迁移到另一个 NC 节点：

```powershell
Move-ServiceFabricPrimaryReplica -ServiceName fabric:/NetworkController/FirewallService -NodeName NC03
```

**结果**：问题解决。

### Step 7：验证修复

**做了什么**：全面验证所有指标。

**验证结果**：

| 检查项 | 结果 |
|--------|------|
| VM 获取虚拟 IP | ✅ 成功 |
| NCHA → SDNAPI:6640 | ✅ 正常 |
| SDNVSM/SDNFW → NCHA:6640 | ✅ 正常 |
| SDNVSM → NCHA:443 | ✅ 正常 |
| **SDNFW → NCHA:443** | **✅ 正常** |
| Configuration State | ✅ Success |
| Provisioning State | ✅ Succeeded |
| VM 端口状态 | ✅ Allowed |

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| SDNFW 服务健康状态报告 OK，但实际通信异常 | 常规健康检查无法发现问题，排查方向一度受阻 | 结合端口状态检查（VFP blocked）和 ETL 日志中的 `FabricNotPrimaryException` 错误定位根因 |
| 隐藏记录不可见 | ProfileId 为空不直观反映根因 | 通过理解 SDN 策略推送完整链路（SDNAPI → SDNVSM → SDNFW → NCHA → VFP），层层排查定位到 SDNFW 环节 |
| 手动设置 ProfileId 的临时方案未生效 | 无法快速恢复业务 | 需要从根本上修复 SDNFW 服务，而非仅修补端口配置 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

SDNFW (Network Controller Firewall Service) 的 Service Fabric **主副本出现内部异常**（`FabricNotPrimaryException`），导致：

1. SDNFW 无法正常执行读写会话（Primary Recovery 任务失败）
2. SDNFW 与 NCHA 之间的 WCF 443 通信通道被中止（`CommunicationObjectAbortedException`）
3. 防火墙策略无法推送到 Hyper-V Host
4. VM 网络适配器的 VFP 端口保持默认阻塞状态（Blocked: TRUE，ProfileId 为空）
5. VM 无法获取虚拟 IP 地址

整个策略推送链路：`SDNAPI → SDNVSM → SDNFW → NCHA:443 → VFP Port`，在 SDNFW → NCHA:443 环节断裂。

### Resolution

将 SDNFW 主副本迁移到健康的 NC 节点：

```powershell
Move-ServiceFabricPrimaryReplica -ServiceName fabric:/NetworkController/FirewallService -NodeName NC03
```

迁移后，SDNFW 在新节点上正常初始化主副本，WCF 通道重新建立，策略成功推送到 Host。

### Workaround

在某些类似场景中，手动设置 VM 网络适配器的 ProfileId 可以临时恢复连通性：

```powershell
Set-SdnVMNetworkAdapterPortProfile -VMName "<VMName>" -MACAddress <MAC> -ProfileID "<InstanceID>"
```

但此方法仅为临时方案，不能替代修复 SDNFW 服务本身。

## 6. 经验教训 (Lessons Learned)

### 技术知识

- **SDN 策略推送链路**：`SDNAPI → SDNVSM → SDNFW → NCHA → VFP`。理解完整链路是排查 SDN 问题的基础
- **Host ↔ NC 三大关键连接**：
  - `NCHA → SDNAPI:6640`（端口通知）
  - `SDNVSM/SDNFW → NCHA:6640`（VNET/FW 策略推送）
  - `SDNFW/SDNVSM → NCHA:443`（WCF，推送 DHCP/DNS/PA 等）
- **Provisioning vs Configuration State**：
  - Provisioning = NC 端策略创建状态
  - Configuration = 策略推送到 Host 的状态
  - Provisioning 成功 + Configuration 失败 = 推送链路问题
- **ProfileId 为空 + VFP Port Blocked** = 防火墙策略未到达 Host 的典型症状
- **Service Fabric 健康状态可能"欺骗性"报告 OK**，实际服务内部可能存在副本异常

### 排查方法

1. **从 networkinterface.json 入手**：通过 MAC 地址定位接口，检查 Provisioning 和 Configuration State
2. **逐一验证 Host ↔ NC 的所有关键端口**：使用 `Get-NetworkControllerReplica` 确认微服务节点，`netstat -abno` 检查连接
3. **检查 VFP 端口状态**：`vfpctrl.exe /port <PortId> /get-port-state` 是判断端口是否被防火墙策略阻塞的直接方法
4. **NC ETL 日志是深度诊断的关键**：位于 `C:\Windows\tracing\SDNDiagnostics\Logs`，需根据微服务位置从对应的 NC 节点收集
5. **对比正常 vs 异常场景**：建立正常场景的基线（ProfileId 非空、Port State Enabled、Configuration Success），有助于快速定位偏差

### 关键命令速查

```powershell
# 导出网络接口配置
Get-NetworkControllerNetworkInterface -ConnectionUri $uri | ConvertTo-Json -Depth 10

# 查看 VM 网络适配器
Get-VMNetworkAdapter -VMName <VMName>

# 查看端口 Profile
Get-SDNVmNetworkAdapterPortProfile -AllVMs

# 查看 VFP 端口状态
vfpctrl.exe /port <PortId> /get-port-state

# 查看 NC 微服务副本分布
Get-NetworkControllerReplica

# 检查 Service Fabric 服务健康
Get-ServiceFabricServiceHealth -ServiceName fabric:/NetworkController/FirewallService

# 迁移主副本
Move-ServiceFabricPrimaryReplica -ServiceName fabric:/NetworkController/FirewallService -NodeName <NodeName>

# 手动设置 Port Profile（临时方案）
Set-SdnVMNetworkAdapterPortProfile -VMName <VMName> -MACAddress <MAC> -ProfileID "<InstanceID>"
```

## 7. 参考文档 (References)

- [Troubleshoot the Windows Server SDN Stack](https://learn.microsoft.com/en-us/windows-server/networking/sdn/troubleshoot/troubleshoot-windows-server-software-defined-networking-stack) — SDN 故障排查总体指南，包含 Configuration State 检查方法
- [Move-ServiceFabricPrimaryReplica](https://learn.microsoft.com/en-us/powershell/module/servicefabric/move-servicefabricprimaryreplica) — 迁移 Service Fabric 有状态服务主副本的 PowerShell 命令

---

# Case Summary: SDN VM Network Adapter Policy Push Failure — Firewall Service Primary Replica Exception

**Product/Service:** Azure Stack HCI / Windows Server SDN (Software Defined Networking)

---

## 1. Symptoms

- Customer failed to set VM network adapter to virtual network via WAC (Windows Admin Center). The UI reported: **"couldn't save the network settings"**
- VM could not obtain a virtual IP address
- This was a pure SDN policy push issue, unrelated to the WAC UI itself

## 2. Background / Environment

- **Platform**: Azure Stack HCI with SDN (Software Defined Networking)
- **Components**:
  - Network Controller (NC) cluster: multi-node deployment
  - Hyper-V Host: running the target VM
  - SDN microservices involved: SDNAPI, SDNVSM (Virtual Switch Manager), SDNFW (Firewall Service)
- **Critical network connections (Host ↔ NC)**:
  - `NCHA → SDNAPI:6640`: Host Agent sends port add/remove notifications to NC
  - `SDNVSM/SDNFW → NCHA:6640`: NC pushes VNET and firewall policies to Host
  - `NC (SDNFW/SDNVSM) → NCHA:443`: WCF connection for pushing DHCP, DNS, PA configurations
- **VM**: `sdn-vnet-vm01`, MAC address `00-1D-D8-AA-1C-34`

## 3. Investigation & Troubleshooting

### Step 1: Verify the actual configuration state of the network adapter

**Action**: Exported network interface JSON from Network Controller REST API.

```powershell
Get-NetworkControllerNetworkInterface -ConnectionUri $uri | ConvertTo-Json -Depth 10 > C:\networkinterface.txt
```

Located the correct interface by VM's MAC address:

```powershell
Get-VMNetworkAdapter -VMName sdn-vnet-vm01
```

**Finding**: Three critical pieces of information from the JSON:

| Item | Value | Meaning |
|------|-------|---------|
| Instance ID | `d072ad2b-ff97-4823-a84f-8dd75c138590` | Key identifier for ETL log analysis |
| Provisioning State | `Succeeded` | NC successfully created the policy |
| Configuration State | `Failure` | Policy push to Host **failed** |

Error details:
```json
{
  "configurationState": {
    "status": "Failure",
    "detailedInfo": [{
      "source": "VirtualSwitch",
      "message": "The host has not yet established communication with the Network Controller.",
      "code": "HostNotConnectedToController"
    }]
  }
}
```

**Conclusion**:
- **Provisioning Succeeded + Configuration Failure** = Policy was created on NC side, but the push to Host failed
- Error message `HostNotConnectedToController` points to network connectivity between Host and NC

### Step 2: Check 6640/443 connections between Host and NC

**Action**:
- Used `Get-NetworkControllerReplica` to identify which NC node hosts each microservice
- Used `netstat -abno` to check each connection

**Finding**:

| Connection | Status |
|------------|--------|
| NCHA → SDNAPI:6640 | ✅ OK |
| SDNVSM → NCHA:6640 | ✅ OK |
| SDNFW → NCHA:6640 | ✅ OK |
| SDNVSM → NCHA:443 | ✅ OK |
| **SDNFW → NCHA:443** | **❌ Not established** |

**Conclusion**: The 443 TLS connection from SDNFW to NCHA was not established — this is the direct cause of the policy push failure.

### Step 3: Investigate 443 connection failure

**Action**: Followed standard TLS connection troubleshooting:
1. TCP 443 port connectivity → ✅ OK
2. SDNFW service health:
   ```powershell
   Get-ServiceFabricServiceHealth -ServiceName fabric:/NetworkController/FirewallService
   # AggregatedHealthState: Ok
   ```
   → ✅ OK
3. Certificate/trust chain → ✅ OK

**Conclusion**: All three baseline checks passed. Deeper investigation via VM port state and NC ETL logs was needed.

### Step 4: Check VM network interface port state

**Action**: Retrieved port profile and port state.

```powershell
Get-SDNVmNetworkAdapterPortProfile -AllVMs
```

**Finding**: ProfileId was empty (all-zero GUID), indicating the port configuration was never applied:

```
VMName      : sdn-vnet-vm01
MacAddress  : 001DD8AA1C34
ProfileId   : {00000000-0000-0000-0000-000000000000}   ← Empty!
ProfileData : 2
PortId      : D267D29E-9E28-4546-BAF9-016437966ADC
```

Port state check:

```powershell
vfpctrl.exe /port D267D29E-9E28-4546-BAF9-016437966ADC /get-port-state
# PORT STATE
#   Enabled : FALSE
#   Blocked : TRUE    ← Port is blocked!
```

**Conclusion**: The VM network port was blocked by VFP (Blocked: TRUE) and ProfileId was empty, confirming SDNFW never successfully pushed firewall policies to the Host.

> **Note**: Attempted to manually set ProfileId to the known Instance ID:
> ```powershell
> Set-SdnVMNetworkAdapterPortProfile -VMName "sdn-vnet-vm01" -MACAddress 001DD8AA1C34 -ProfileID "d072ad2b-ff97-4823-a84f-8dd75c138590"
> ```
> This approach worked in a different similar case but did not take effect immediately in this scenario.

### Step 5: Analyze SDNFW NC ETL logs

**Action**: Collected ETL logs from the NC node hosting SDNFW (path: `C:\Windows\tracing\SDNDiagnostics\Logs`).

**Finding**: Two critical errors in the logs:

**Error 1 — WCF communication object aborted**:
```
ControllerTransientException: An internal error occurred in a WCF operation
---> System.ServiceModel.CommunicationObjectAbortedException:
The communication object, System.ServiceModel.Channels.ClientReliableDuplexSessionChannel,
cannot be used for communication because it has been Aborted.
```

**Error 2 — Service Fabric primary replica exception**:
```
Primary recovery task failed: System.AggregateException
---> ImosException: Network Controller operation failed
---> System.Fabric.FabricNotPrimaryException:
Must be primary to begin read/write session
```

**Conclusion**: The SDNFW primary replica had an internal exception (`FabricNotPrimaryException`). Despite reporting healthy status, the WCF communication channel was aborted, preventing the 443 connection to NCHA for policy push.

### Step 6: Move Firewall Service primary replica

**Action**: Moved the SDNFW primary replica to a different NC node:

```powershell
Move-ServiceFabricPrimaryReplica -ServiceName fabric:/NetworkController/FirewallService -NodeName NC03
```

**Result**: Issue resolved immediately.

### Step 7: Verification

**All indicators confirmed healthy**:

| Check | Result |
|-------|--------|
| VM obtained virtual IP | ✅ Success |
| NCHA → SDNAPI:6640 | ✅ OK |
| SDNVSM/SDNFW → NCHA:6640 | ✅ OK |
| SDNVSM → NCHA:443 | ✅ OK |
| **SDNFW → NCHA:443** | **✅ OK** |
| Configuration State | ✅ Success |
| Provisioning State | ✅ Succeeded |
| VM port state | ✅ Allowed |

## 4. Blockers & How They Were Resolved

| Blocker | Impact | Resolution |
|---------|--------|------------|
| SDNFW health reported OK despite internal communication failure | Standard health checks couldn't surface the problem, misdirected investigation | Combined VFP port state check (blocked) with ETL log analysis revealing `FabricNotPrimaryException` |
| Empty ProfileId didn't directly indicate root cause | Not intuitive to trace back to SDNFW issue | Understood the full SDN policy push chain (SDNAPI → SDNVSM → SDNFW → NCHA → VFP) and traced layer by layer |
| Manual ProfileId workaround didn't take effect | Couldn't quickly restore service | Required fixing the SDNFW service itself rather than patching the port configuration |

## 5. Root Cause & Resolution

### Root Cause

The SDNFW (Network Controller Firewall Service) **Service Fabric primary replica experienced an internal exception** (`FabricNotPrimaryException`), causing:

1. SDNFW could not perform read/write sessions (Primary Recovery task failed)
2. WCF 443 communication channel between SDNFW and NCHA was aborted (`CommunicationObjectAbortedException`)
3. Firewall policies could not be pushed to the Hyper-V Host
4. VM network adapter VFP port remained in default blocked state (Blocked: TRUE, ProfileId empty)
5. VM could not obtain a virtual IP address

The full policy push chain: `SDNAPI → SDNVSM → SDNFW → NCHA:443 → VFP Port` was broken at the `SDNFW → NCHA:443` segment.

### Resolution

Move the SDNFW primary replica to a healthy NC node:

```powershell
Move-ServiceFabricPrimaryReplica -ServiceName fabric:/NetworkController/FirewallService -NodeName NC03
```

After migration, SDNFW properly initialized the primary replica on the new node, the WCF channel was re-established, and policies were successfully pushed to the Host.

### Workaround

In some similar scenarios, manually setting the VM network adapter's ProfileId can temporarily restore connectivity:

```powershell
Set-SdnVMNetworkAdapterPortProfile -VMName "<VMName>" -MACAddress <MAC> -ProfileID "<InstanceID>"
```

This is a temporary fix only and does not substitute for repairing the SDNFW service itself.

## 6. Lessons Learned

### Technical Knowledge

- **SDN policy push chain**: `SDNAPI → SDNVSM → SDNFW → NCHA → VFP`. Understanding the full chain is foundational for SDN troubleshooting
- **Three critical Host ↔ NC connections**:
  - `NCHA → SDNAPI:6640` (port notifications)
  - `SDNVSM/SDNFW → NCHA:6640` (VNET/FW policy push)
  - `SDNFW/SDNVSM → NCHA:443` (WCF for DHCP/DNS/PA)
- **Provisioning vs Configuration State**:
  - Provisioning = policy creation status on NC side
  - Configuration = policy push status to Host
  - Provisioning Succeeded + Configuration Failed = push chain problem
- **Empty ProfileId + VFP Port Blocked** = classic symptom that firewall policies haven't reached the Host
- **Service Fabric health status can be misleading** — the service may report OK while the primary replica has internal exceptions

### Troubleshooting Methodology

1. **Start with networkinterface.json**: Locate the interface by MAC address, check Provisioning and Configuration State
2. **Verify all critical Host ↔ NC ports**: Use `Get-NetworkControllerReplica` to identify microservice nodes, `netstat -abno` to check connections
3. **Check VFP port state**: `vfpctrl.exe /port <PortId> /get-port-state` directly shows whether the port is blocked by firewall policies
4. **NC ETL logs are key for deep diagnosis**: Located at `C:\Windows\tracing\SDNDiagnostics\Logs`, collect from the correct NC node based on microservice placement
5. **Compare working vs non-working scenarios**: Establish a baseline of normal behavior (non-empty ProfileId, Port State Enabled, Configuration Success) to quickly identify deviations

### Key Command Reference

```powershell
# Export network interface configuration
Get-NetworkControllerNetworkInterface -ConnectionUri $uri | ConvertTo-Json -Depth 10

# View VM network adapter
Get-VMNetworkAdapter -VMName <VMName>

# View port profile
Get-SDNVmNetworkAdapterPortProfile -AllVMs

# Check VFP port state
vfpctrl.exe /port <PortId> /get-port-state

# View NC microservice replica distribution
Get-NetworkControllerReplica

# Check Service Fabric service health
Get-ServiceFabricServiceHealth -ServiceName fabric:/NetworkController/FirewallService

# Move primary replica
Move-ServiceFabricPrimaryReplica -ServiceName fabric:/NetworkController/FirewallService -NodeName <NodeName>

# Manually set port profile (temporary workaround)
Set-SdnVMNetworkAdapterPortProfile -VMName <VMName> -MACAddress <MAC> -ProfileID "<InstanceID>"
```

## 7. References

- [Troubleshoot the Windows Server SDN Stack](https://learn.microsoft.com/en-us/windows-server/networking/sdn/troubleshoot/troubleshoot-windows-server-software-defined-networking-stack) — Comprehensive SDN troubleshooting guide covering Configuration State checks and diagnostic tools
- [Move-ServiceFabricPrimaryReplica](https://learn.microsoft.com/en-us/powershell/module/servicefabric/move-servicefabricprimaryreplica) — PowerShell cmdlet to move Service Fabric stateful service primary replica to a specified node
