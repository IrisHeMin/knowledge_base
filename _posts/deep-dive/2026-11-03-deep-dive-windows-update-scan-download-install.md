---
layout: post
title: "Deep Dive: Windows Update Scan, Download, Install and Commit"
date: 2026-11-03
categories: [Knowledge, WindowsServicing]
tags: [windows-update, uso, uup, delivery-optimization, cbs, winsxs, servicing, wsus, troubleshooting]
type: "deep-dive"
permalink: /deep-dive/windows-update-scan-download-install/
---

# Deep Dive: Windows Update 扫描、下载、安装与提交

**Topic:** Windows Update end-to-end lifecycle: Scan → Download → Install → Restart/Commit<br>
**Category:** Windows Servicing / Deployment / Troubleshooting<br>
**Level:** 高级，按入门心智模型逐层深入<br>
**Last Updated:** 2026-11-03

> **30 秒结论：**Scan 决定“这台设备现在应该得到什么”；Download 取得并验证实现目标状态所需的内容；Install 由实际安装引擎改变系统状态；Commit 完成必要的重启与最终落地。**USO 负责安排，WU 负责发现与协调，DO 负责运输，CBS 或 Setup 负责实施。不要把它们当成同一个服务。**
>
> **阅读路线：**理论理解读第 1–3 节；配置核查读第 4 节；处理故障直接进入第 5 节，再回到相应机制。下半篇是独立的英文版本。
>
> **边界：**以现代 Windows client 为主，相关 CBS 原理适用于现代 Windows Server，但策略支持、服务实现和更新包行为必须按目标版本核实。传统 CBS 质量更新是主线；完整功能升级另走 Setup 分支。本文不是特定 KB 的最新安装说明，也不是 Windows Hotpatch 的详细指南。
>
> **证据声明：**本文依据公开 Microsoft 文档整理，章节中的状态图、公式、时间线和故障情景是教学模型，不是客户日志，也不是内部源代码调用链。没有读取或诊断本机 Windows Update 日志。所有示例名称均为虚构，不包含客户信息；无原始客户材料，因此无需建立真实身份映射表。引用编号对应第 8 节，链接于 2026-11-03 逐条验证；无法访问的来源未收入。

---

## 1. 概述 (Overview)

Windows Update 不是“下载一个补丁并覆盖几个 DLL”，而是一个**受策略控制、按设备状态计算、由多个执行引擎落地的状态迁移系统**。用户看到的一个 KB，可能对应多个更新对象、多个内容文件，以及多项组件操作。UI 上的一行并不等于一个文件，也不等于一个 CBS package。[R01][R02][R03]

它解决三个独立问题：哪些更新适用于本机；怎样安全、有效地取得所需内容；怎样将一个正在运行的系统转换到一致、可恢复的目标状态。因此，“扫描成功”只能证明某次发现流程成功，“下载 100%”只能证明某一下载范围完成，“需要重启”则说明还不能把所有相关变更视为最终生效。[R01][R07][R10]

在管理体系中，Intune、Group Policy、WSUS 和 Configuration Manager 决定不同层面的策略、批准或部署条件；客户端的发现、下载和安装仍然有各自的组件与日志。**管理平面告诉设备应该做什么，不代表管理服务器亲自完成了设备上的 CBS 安装。**以下把“来源”“传输”和“安装”拆开讨论，而不是用一个“更新源”概括全部行为。[R04][R05]

### 1.1 一个容易记忆的类比

| 更新角色 | 医院类比 | 必须回答的问题 |
|---|---|---|
| USO / Orchestration | 排班与预约系统 | 什么时间允许开始下一步？ |
| WU scan / Applicability | 检查与诊断 | 这位患者需要什么治疗？ |
| UUP Arbiter / Action List | 个体化治疗计划 | 为达到目标，具体需要哪些内容和动作？ |
| Delivery Optimization | 合规药品物流 | 内容从哪里来，完整性如何保证？ |
| CBS / Setup | 实施治疗的团队 | 如何实际改变系统？ |
| Reboot / Commit / Verification | 术后恢复与复查 | 变化是否最终落地，系统是否健康？ |

类比仅用于区分职责，不表示这些组件严格串行或具有一一对应的进程关系。

## 2. 核心概念 (Core Concepts)

### 2.1 Update、KB、Package、Component、Payload 不是同义词

| 概念 | 含义 | 对排查的影响 |
|---|---|---|
| KB | 公开知识文章与更新说明的标识 | 便于人查找，不足以唯一关联内部每个对象 |
| Update identity | WU 更新对象的身份，常见为 UpdateID GUID + RevisionNumber | 关联 scan、download 和 handler 的主要线索 |
| Bundle / Child update | 一个逻辑更新可包含子更新 | 父对象不一定直接包含二进制 payload |
| CBS Package | 服务化安装单位，包含或引用组件与部署信息 | 与 CBS session、DISM package inventory 关联 |
| Component | 由组件身份标识的一组资源及其版本 | 同一 KB 可影响多个组件，不是一个 DLL |
| Payload | 实际内容，例如包、压缩文件、差分数据 | 下载数量与 UI 更新数量不必一致 |

例如“KB50xxxxx 已下载”不能直接推出“其所有 CBS package 均已 Installed”。首先建立 **WU identity → handler/package → CBS session → 最终 package state** 的关联。[R01][R03][R07][R16]

### 2.2 四种常被混用的判断

**Applicable（适用）**：设备状态满足更新规则。<br>
**Offered（被提供）**：服务和部署条件允许把更新提供给该设备。<br>
**Selected / Approved（被选择或批准）**：调用方或管理配置希望部署它。<br>
**Allowed now（现在允许执行）**：暂停、时间、重启状态等条件允许进入下一阶段。

教学上可写成：

```text
可进入部署的候选集合
    = 来源中可见的更新
    ∩ 服务提供/部署范围
    ∩ 本机适用性
    ∩ 调用方筛选
    ∩ 当前策略允许的集合
```

这不是实现中的固定执行顺序。部分判断可以在服务端完成，部分在客户端完成；同一更新也可能在安装前重新检查条件。**未出现某个更新不自动等于网络坏了。**[R01][R04][R15][R20]

### 2.3 USO、WUA、Arbiter、DO、CBS、Setup 的分工

| 层 | 常见名称 / 可观察对象 | 核心职责 | 不能据此推断 |
|---|---|---|---|
| UI | Settings Windows Update 页面 | 发起请求、展示状态与通知 | UI 百分比就是底层事务百分比 |
| Orchestration | USO；`UsoSvc`；USOShared ETL | 安排 scan、download、install、restart 的时机 | 每个等待都是安装引擎故障 |
| WU client | WUA；`wuauserv`；WU ETL | 服务交互、元数据与更新对象管理、handler 协调 | WUA 自己负责所有类型更新的文件替换 |
| Composition planning | UUP UpdateAgent / Arbiter / CompDB | 根据设备状态计算内容需求和 action list | Arbiter 等同一个永远独立运行的 EXE |
| Transport | DO；`DoSvc`；特定路径上的 BITS | 获取内容、缓存、传输优化 | BITS 无任务就代表没有任何更新下载 |
| OS servicing | CBS；Windows Modules Installer / `TrustedInstaller`；可能观察到 `TiWorker.exe` | Windows 组件与 package 的安装、修复和相关状态维护 | 看见进程就证明当前目标 KB 正在推进 |
| OS upgrade | Windows Setup；Setup logs | 完整 OS 升级、兼容性与迁移 | 所有 feature update 都需要完整 Setup 流程 |

表中的名称是诊断入口，不是跨所有 Windows build 不变的二进制拓扑。UUP 是发布、发现与内容编排模型；CBS 是一种落地引擎，**UUP 不是 CBS 的替代品**。[R02][R03][R06]

### 2.4 三种“状态”必须分别记录

1. **更新对象状态**：IsInstalled、IsDownloaded、更新历史中的结果。
2. **安装操作结果**：本次 operation 成功、部分成功、失败或取消。
3. **系统最终状态**：目标 package/build 是否到位、是否仍需重启、相关功能是否正常。

WUA 的 `OperationResultCode` 定义为 0 NotStarted、1 InProgress、2 Succeeded、3 SucceededWithErrors、4 Failed、5 Aborted。**ResultCode 不是 HRESULT**；集合操作返回成功或部分成功时，还要查看每个更新的结果与重启需求。[R14][R24]

### 2.5 DataStore、下载缓存、Component Store 的职责不同

```text
%windir%\SoftwareDistribution\DataStore\DataStore.edb
    WU 本地元数据、相关记录和对象关联

%windir%\SoftwareDistribution\Download
    常见更新下载/工作目录，不是所有内容的唯一缓存

Delivery Optimization cache
    传输层缓存，可受策略改变位置与配额

%windir%\WinSxS
    Windows Component Store，支撑服务化、修复与相关回退能力

%windir%\servicing\Packages
    服务化 package 元数据
```

**删除下载缓存不能重建损坏的 Component Store。**重置 DataStore 还会改变本地发现上下文与可见历史，不能作为不留证据的第一步。WinSxS 则不是可手工清空的临时目录。[R03][R07][R12][R17]

## 3. 工作原理 (How It Works)

### 3.1 整体架构：控制平面与内容平面

```text
                Policy / Deployment / User request
                              |
                              v
                 Update Session Orchestrator
                  |          |            |
                  v          v            v
                Scan      Download      Install / Restart
                  |          |            |
                  v          v            v
             WU client    DO / content   Installer handler
                  |        delivery       |          |
            Metadata source               v          v
          Microsoft / WSUS                CBS        Setup
                  |                       |          |
                  v                       v          v
         Applicability + local       Component    Upgrade +
          state + CompDB plan          Store       migration
                  |
                  +---- required payloads / action list ----+
```

这是职责图，不是网络图。Metadata source、payload source、installer 可以完全不同：设备可能向 WSUS 扫描，却通过指定内容路径取得数据，再由本地 CBS 安装。判断“是否绕过 WSUS”时，必须先问绕过的是哪一层。[R01][R02][R04][R08]

### 3.2 Stage 0：触发、策略与前置状态

**发生什么：**自动安排、用户操作或 WUA API 调用创建工作请求。Orchestrator 决定是否及何时推进。

应记录更新类型、调用方、用户/服务上下文、请求时间、OS build、当前管理方式。不要假定 Settings、“企业部署代理”和自写 COM 脚本的搜索条件完全一致，也不要把脚本扫描结果当成 Settings 展示结果的完整复制。[R01][R14][R15]

| Expected | Unexpected / 待调查 | 最小下一步 |
|---|---|---|
| 请求后出现相关会话，或有可解释的策略等待 | 没有相关会话、反复被取消、策略来源与预期不符 | USO ETL + 生效策略 + 触发时间 |
| 暂停或重启条件存在时延后下一阶段 | 没有发现策略条件却无限等待 | 对照 WU 与 USO 事件，确定是否完成交接 |

**为什么服务是 Stopped 不等于故障？**服务可按需启动；单次状态快照不知道任务是否应该在这个时刻运行。必须结合启动类型、服务事件和目标任务，而不是把所有更新服务永久设成 Automatic。

### 3.3 Stage 1：Scan——发现候选与判断适用性

#### Step 1：选定本次逻辑更新服务

先确定 effective source。Windows Update、Microsoft Update、WSUS 是不同的逻辑服务。`WUServer` 注册表值存在并不能独立证明所有 update class 都向 WSUS 扫描；现代 scan source policy 可分别控制 Quality、Feature、Driver/Firmware、Other。[R04]

WU 日志的 ServiceID 有助于识别逻辑服务，但**不是固定物理 URL 或固定云服务器身份**。实际 endpoint 应从当前会话的配置、请求与日志确认。不能看到某个 GUID 就永久硬编码一个网络目标。[R01]

#### Step 2：同步元数据，而不是立即取得全部二进制

Scan 交换更新描述、规则及相关状态。客户端利用本地已有数据减少重复同步；因此 Full scan / Delta scan 描述的是元数据缓存背景，不是“全量补丁 / 差分补丁”的区别。[R01]

```text
Metadata question:  哪些更新对象可能与这台设备有关？
Payload question:   为执行已选定的目标，必须取得哪些内容？
```

扫描阶段出现少量网络流量，不能推断操作系统补丁已开始下载。反过来，网络流量较少也不意味着没有执行规则评估。

#### Step 3：适用性、依赖、已安装与替代关系

可影响结果的状态包括 OS release/build、架构、产品/edition、已有组件、语言与功能配置、先决条件、已安装状态。驱动还要考虑设备匹配，不能按“版本数字更大”简单替代 OS 更新规则。[R01][R03][R15]

用依赖图理解：

```text
Detector / prerequisite ----+
                            +--> candidate update
Device state ---------------+          |
                                       +--> child payload updates
Supersedence / install state -----------+
```

“已被替代”回答的是更新关系，不代表设备已经安装了替代者；“不适用”也不等于“失败”。判断旧 KB 是否仍应出现，需要结合当前目标和已安装状态，而不是只按发布时间排序。

#### Step 4：形成可操作结果并返回调用方

WU 返回符合本次 criteria 与服务条件的对象。调用方可以继续下载、等待策略或展示可选更新。**成功且返回 0 个更新，是一个有效结果，不是自动判定 client corruption 的理由。**[R15]

| Expected | Unexpected / 待调查 | 最小下一步 |
|---|---|---|
| 服务正确、同步成功、规则评估完成 | SOAP/HTTP/TLS 失败、元数据错误、服务与预期不符 | WU ProtocolTalker + source + endpoint |
| 不适用或 hold 可解释未提供 | 已知应被提供，却无规则/策略说明 | 对比一台相同 build/策略的正常设备 |

#### Offline scan 的两种含义

“不联网”可能是使用已缓存的本地 metadata，也可能是通过已注册的离线扫描 package 服务运行检测，例如 WUA SDK 示范的离线 catalog 路径。它们不等于“从完整在线服务获得完全相同的候选与内容”，也不自动包含可安装 payload。[R01][R14]

### 3.4 Scan 的网络细节：为什么浏览器能访问还不够

将网络问题拆成：

```text
DNS -> routing/TCP -> TLS/identity -> HTTP/proxy auth
    -> update protocol response -> metadata parsing/evaluation
```

对 WUA 的 detection URL，公开文档描述 system proxy 先尝试，特定失败时尝试 user proxy；非 detection 的 WUA URL 则描述相反顺序，且 WSUS 有相关策略选择。**这个规则不能不加区分地套到 DO。**[R01]

DO 的公开代理说明描述了 WinHTTP automatic proxy discovery、用户 token 与无用户登录时的上下文差异；不是“所有 Windows 服务都只读 `netsh winhttp`”。所以浏览器成功只能证明浏览器这条链路，不能证明服务身份、PAC 结果和更新协议也成功。[R09]

Microsoft 更新 metadata 连接有证书固定要求；某些 TLS inspection 会破坏验证。CDN/payload 的访问与信任验证又不同。应按官方端点要求做精确例外，不能通过关闭系统 TLS 验证来“修好更新”。也不要把 CDN IP 固定到 hosts：地址可变化。[R10]

### 3.5 Stage 2：Download——从目标组合计算具体内容

#### Step 1：Arbiter 读取设备状态与 CompDB

CompDB 是描述目标组合、可用差分与条件规则的 XML 数据。Arbiter 将它与设备状态对照，产生 action list：需要哪些 payload，以及安装方应执行什么。[R02]

教学模型：

```text
Target composition - usable local state
    + missing prerequisites
    + required language/feature resources
    = content and action requirements
```

这不是“比较两个文件夹”的算法，也不承诺所有版本都有同样字段。它解释为什么**同一个 KB，在不同设备上可能下载不同数量或不同大小的内容**。不要拿某台设备的网络字节数当成另一台的固定预期。

#### Step 2：建立下载任务与选择来源

DO 可从 HTTP origin/CDN、Microsoft Connected Cache 和符合条件的 peer 取得内容。它是传输优化层，不决定哪些更新适用。DO 通常可以组合多个来源，不应想成永远先 peer、失败后才单线尝试 CDN。[R08][R25]

| 来源 | 解决的问题 | 不解决的问题 |
|---|---|---|
| HTTP origin / CDN / 配置的内容服务器 | 原始内容可达性 | 本机 CBS 损坏 |
| Microsoft Connected Cache | 减少重复 WAN 下载 | 改变 WSUS 批准或更新适用性 |
| Peer | 复用其他设备已有内容 | 让不可信 peer 决定可以安装什么 |

DO peer 数据使用 TCP 7680，不是 SMB 共享。阻断 peer 端口通常只影响 peer 优化；若 origin 同样不可达，才不能依赖 fallback 完成。**“有 fallback”不等于“任何网络策略下都保证成功”。**[R08]

#### Step 3：分块验证与完整文件验证

可用的可信内容 metadata 提供分块 hash。来自 peer 的内容在接受前被验证；完成下载后，安装前还要做相应的文件 hash / 数字签名检查。网络路径的可信度和内容的可信度是不同层面的保证。[R08][R10]

```text
可信 metadata / 验证信息
        |
        v
内容块校验 -> 重组内容 -> 安装前内容/签名验证 -> installer
```

因此“下载来自 HTTP”不等于“可以任意篡改后安装”，也不代表可以忽略企业对 HTTP 的访问控制。实际协议和端点按当前内容路径确认，不把所有 payload 描述成只走一种传输。

#### Step 4：解压、重建与 staging

网络下载结束之后，还可能有内容处理、差分重建或 stage 工作。**传输完成不等于所有安装输入都已准备好。**差分需要兼容的基础内容；缺失匹配 binary/component 的问题应结合实际 `PSFX_*`、CBS 上下文和 package 身份定位，不能一律归为带宽问题。[R07]

**重要区别：**DO 的网络分块与更新包的 binary differential 是两种粒度。前者优化运输，后者描述如何表示或重建目标内容。清 DO cache 不会自动修复缺失的差分基础组件。

| Expected | Unexpected / 待调查 | 最小下一步 |
|---|---|---|
| 产生内容需求，字节/任务推进，校验通过 | origin 403/404、proxy 407、重复 hash 失败、磁盘写失败 | DO 状态 + 实际 URL/响应 + 磁盘/代理证据 |
| 传输后出现处理与 stage 活动 | WU 认为下载完成，但无法 stage | WU handler/agent + 同时间 CBS/内容处理信息 |

### 3.6 Differential、SSU、LCU 与 Checkpoint：三个不同维度

**LCU** 是累计质量更新；**SSU** 修复“安装更新的基础设施”。现代支持路径通常组合交付 SSU 与 LCU，但组合交付不代表取消先决条件，具体 KB 仍可能有安装要求。[R06]

**Checkpoint cumulative update** 从 Windows 11 24H2 / Windows Server 2025 相关路径引入新的差分基线。逻辑上累计，不等于一个离线 MSU 永远自带所有所需基础文件。WU/WSUS 通常协调处理；Catalog/offline 操作必须核对所需 checkpoint。[R11]

```text
Release baseline -> checkpoint A -> checkpoint B -> target LCU
                              \             \
                        differentials relative to relevant baseline
```

不要把它解释为“必须逐月安装每个月补丁”；要满足的是所需 checkpoint 与目标 package 的依赖。离线定制 FoD / language pack 还可能需要更完整的 checkpoint 内容集合。[R11]

### 3.7 Stage 3：Install——handler 把任务交给正确引擎

WU handler 接受安装任务后，由更新类型决定落地路径。**质量更新通常进入 CBS；完整 feature upgrade 进入 Setup；驱动和其他产品更新不能照搬 CBS 的每一条结论。**[R01][R02]

#### CBS 路径的概念性流程

```text
Installation request
    -> package identity / metadata validation
    -> applicability and dependency resolution
    -> stage resources / evaluate target component state
    -> execute online servicing work
    -> queue work requiring restart, if any
    -> return operation result + reboot requirement
    -> finish remaining work across restart / finalize state
```

该图是 support 分析框架，不是保证每个 build 都严格打印这些阶段名。

**Package metadata、component manifests、catalog 和 payload 分别回答不同问题：**装什么包、组件有哪些资源、如何验证信任与完整性、实际写入哪些内容。只有二进制文件存在，不代表相应 metadata、依赖与组件状态都正确。[R07][R12]

#### Component Store 与硬链接：为什么不能手工覆盖 System32

WinSxS 中的组件支持更新、功能启用、修复和部分卸载场景；运行位置上的文件可能与组件存储中的文件硬链接到相同数据。因此目录大小可能重复计算，版本资源也不是简单的“旧文件备份”。[R12]

手工复制一个 DLL 无法保证 package identity、manifest、catalog、依赖、注册表资源和其他文件同时一致。**“文件版本看起来对”不是完整服务化成功的证据。**需要通过支持的 servicing 工具和 package state 判断。

#### CBS / CSI 与日志阅读边界

CBS 日志可能同时出现 package 层消息与 CSI component 层消息。可以把它理解为“包级计划与结果”和“组件资源/一致性相关工作”的不同观察层，但不能仅凭标签猜测未记录的内部调用。[R07]

优先找：

```text
本次目标 package
    -> 对应 session
    -> 同一事务中最早有因果意义的错误
    -> 后续传播的 HRESULT
    -> 本次 session 的最终结果
```

“最早”限定在**相关事务**。整份 CBS.log 的第一个 Error 可能来自别的任务；全文件最后一个 Error 也可能只是对原始失败的包装。完整 Setup 失败另按 Setup 的 fatal/rollback 路径分析，不能机械套用同一搜索方向。[R07][R19][R21]

#### Staged、Install Pending、Installed 如何理解

| 状态 | Support 解读 | 不要推断 |
|---|---|---|
| Staged | 包的内容已进入服务化准备状态 | 所有功能已激活 |
| Install Pending | 安装仍有待完成工作，常与重启相关 | 需要手工删除 pending 状态 |
| Installed | package inventory 报告已安装 | 当前进程已加载所有新 binary；所有更新类型均不需验证 |
| Superseded | 存在更新的替代关系/状态 | 删除任意旧组件都安全 |

不同工具展示粒度不同；状态不是本文承诺的单一全局枚举。用 DISM/Get-WindowsPackage 获取目标包的实际状态，并联看重启与 CBS 结果。[R16]

### 3.8 为什么 Install 会“慢”而网络已经不忙

把耗时拆开：

```text
T_total = T_wait_policy + T_scan + T_transfer + T_prepare
        + T_online_servicing + T_wait_restart
        + T_offline_or_boot_work + T_verification
```

这是工程测量模型，不是 Microsoft 内部计时公式。用于区分四类现象：

| 观察 | 可以提出的假设 | 还缺什么证据 |
|---|---|---|
| 网络安静、CPU 活跃、CBS 持续增长 | 内容处理或 servicing 正推进 | 目标 session、资源与动作是否变化 |
| 网络安静、磁盘延迟高 | 存储瓶颈或过滤驱动影响 | I/O trace、磁盘错误、对应文件路径 |
| 资源安静、USO 有明确等待理由 | Orchestration 等待 | effective policy 与下次触发条件 |
| 长时间同一失败反复重试 | 持续性故障而非单纯慢 | 相同对象、错误及重试时间线 |

**固定百分比不是固定步骤**；相同“20%”在不同 build/更新类型中不能定位到同一个内部函数。只有活动与对象都不推进且超出合理基线时，才有依据讨论 hang。

### 3.9 Stage 4：Restart / Commit——落地，不只是重新开机

有些变更可在线完成；另一些涉及正在使用的资源或需要后续启动阶段完成。公开 WU 模型将 Commit 与重启协调联系起来，但不要把它当成所有更新类型共享的一个 ACID 数据库事务。[R01][R13]

```text
Online work complete
        |
        +--> no restart required --> verify final state
        |
        +--> restart required
                 |
                 v
        restart scheduling / user notification
                 |
                 v
        shutdown / boot servicing work
                 |
                 +--> successful finalization --> verify
                 |
                 +--> failure --> recovery / rollback / evidence
```

**没有重启与重启后回滚是两类故障：**前者可能停在 scheduling，后者说明执行已推进到后续阶段。排查“下载慢”不能解释已经发生的 boot servicing failure。

重启后至少交叉确认目标 package state、目标 build/UBR（仅适用会改变它们的 OS 更新）、相关 session 结果与功能健康。更新历史是一份操作记录，不是完整当前库存；较新的累计更新还可能改变先前 KB 的可见性。

### 3.10 Restart deadline 的计算，为什么 Active Hours 不是无限豁免

对于文档所述的 Windows 11 22H2+ deadline 模型：

```text
effective restart deadline =
    max(first scan discovery time + configured deadline,
        restart-required time + grace period)
```

到达 effective deadline 后可强制重启，不再受 Active Hours 一直保护。Windows 10、较旧 Windows 11 和不同补丁水平有差异；不能把当前公式与策略优先级无条件外推。[R18]

教学例子：设备 11 月 3 日 10:00 首次发现更新，deadline 2 天；11 月 4 日 18:00 进入 restart required，grace 1 天。两条边界分别是 11 月 5 日 10:00 和 11 月 5 日 18:00，因此有效边界是后者。该例仅说明计算，不是本机配置，也不是推荐所有设备采用这组值。

### 3.11 分支：完整 Feature Update 走 Windows Setup

完整 OS upgrade 除了安装新 OS，还要处理兼容性、驱动、应用与用户状态迁移。经典分析阶段为 Downlevel、SafeOS、First Boot、Second Boot/OOBE。[R22]

| 阶段 | 关键问题 | 首选证据 |
|---|---|---|
| Downlevel | 兼容性、准备、输入内容与迁移收集是否成功？ | `C:\$Windows.~BT\Sources\Panther` |
| SafeOS | 离线环境、存储、启动相关工作是否成功？ | Setup phase/operation、Panther 与 Rollback |
| First Boot | 新 OS 下驱动与初始配置能否完成？ | Setup + 设备安装/驱动证据 |
| Second Boot / OOBE | 后续迁移与配置是否完成？ | Panther / UnattendGC / Rollback |
| Rollback | 哪个 fatal failure 导致回退？ | Rollback logs + SetupDiag |

上述经典分期有 Windows 10 文档来源；现代版本实际 phase、操作及重启序列以本次 Setup 日志为准，不能保证每次升级具有完全相同的重启次数。Enablement package 这种轻量 feature transition 不应被强行描述成完整 Setup 升级。

Dynamic Update 是更新 Setup/升级所用内容的机制，不能因为名字里有 Update 就把它当作每月 LCU 的同义词。[R23]

### 3.12 闭环：Reporting 与 Compliance 不是瞬时一致

客户端安装状态、WU 重新检测、管理端报告和 UI 展示可能属于不同时间点。诊断“控制台还说缺补丁”时，先确认控制台数据时间，再看客户端实际最终状态；不能为修复 reporting 问题直接重装已经正常安装的更新。

对比机器时，应匹配 **OS build、架构、语言/FoD、来源、策略、目标 update identity、时间范围**。只比较“同为 Windows 11”容易把合理的适用性差异误认为故障。[R01][R03][R04][R11]

## 4. 关键配置与参数 (Key Configurations)

### 4.1 策略默认值不等于设备当前生效值

下表的“默认”只描述未配置策略时的文档级语义；企业基线、MDM、版本与补丁可能改变实际行为。**不要把 registry value 缺失一律解读为关闭。**

| 配置项/参数 | 默认值/未配置语义 | 说明 | 常见核查场景 |
|---|---|---|---|
| Intranet update service / `WUServer`, `WUStatusServer`, `UseWUServer` | 未配置时，不指定企业 WSUS 路径 | endpoint 配置与是否使用需组合看 | 扫错来源、迁移残留 |
| `SetPolicyDrivenUpdateSourceForQualityUpdates` | 未配置则依 OS 与其他来源配置判断 | 0 = WU，1 = WSUS；使用正式策略配置 | 质量更新来源 |
| 对应 Feature / Driver / Other scan-source settings | 同上 | CSP 配置应覆盖相关类别；不要仅写孤立 registry 值 | 混合管理 |
| Pause / deferral / target release | 未配置不人为指定这些限制；仍受服务提供条件影响 | 控制提供/节奏，不是 CBS 修复参数 | “别的机器有、这台没有” |
| Configure Automatic Updates | 不给所有版本设定一个通用数值默认 | 影响自动行为；与 deadline 的关系按版本核实 | 安装未触发 |
| Active Hours | 可由设备/用户/策略决定 | 不是无限阻止重启的保证 | 已安装但待重启 |
| Deadline / grace | 未配置即未额外强制本文讨论的 deadline 值 | 明确起算点与 effective deadline | “为什么在使用时间重启” |
| `DODownloadMode` | DO reference 描述 LAN(1) 为默认 client 模式 | 0 禁 peer，1 LAN，2 Group，3 Internet，99 Simple；100 在 Windows 11 已弃用 | 禁 peer 不要禁掉下载能力 |
| `DOGroupID` / `DOGroupIDSource` | 未配置时按文档的身份来源选择规则 | 主要关联 Group 模式 | 跨子网/站点优化 |
| `DOMaxCacheAge` | 259200 秒（3 天） | 内容保留上限之一 | 内容复用与缓存 churn |
| `DOMaxCacheSize` | 20% | 配额不是保证所有文件都保留 | 小磁盘/缓存效果 |
| DO foreground/background bandwidth | 相关 reference 中 0 表示动态调整 | 0 不等于零速率；绝对值参数注意 KB/s 单位 | 下载过慢或占满 WAN |
| Repair source / `/Source` | 策略允许时可使用 WU | 修复内容来源独立核查 | DISM 报 source missing |
| DISM `/LimitAccess` | 不指定时不额外阻止 WU repair fallback | 只影响相应 servicing 操作 | 隔离网络、验证本地 source |

来源：[R04][R13][R17][R18][R25]。Windows Update 策略建议通过 GPO/MDM 管理，不将本文注册表路径当成直接写入脚本。

### 4.2 常用只读核查位置

```powershell
# 在目标设备的 Windows PowerShell 中按需运行。
# 这些查询不安装更新；权限不足或键不存在应单独记录，不直接认定故障。
Get-Service -Name wuauserv,UsoSvc,DoSvc,BITS,TrustedInstaller |
    Select-Object Name,Status,StartType

Get-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate' `
    -ErrorAction SilentlyContinue
Get-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU' `
    -ErrorAction SilentlyContinue
Get-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\DeliveryOptimization' `
    -ErrorAction SilentlyContinue

Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' |
    Select-Object ProductName,DisplayVersion,CurrentBuild,UBR

netsh winhttp show proxy
DISM.exe /Online /Get-Packages /Format:Table
```

Registry 快照只是证据的一部分。GPO 可用 `gpresult /scope computer /h C:\WU-Evidence\gpresult.html`（先创建目录）核实来源；MDM 配置要结合管理诊断。**`gpresult` 不代表完整 MDM 结果。**`netsh winhttp show proxy` 也不代表 DO 每次请求最终选择的代理。[R04][R09]

## 5. 常见问题与排查 (Common Issues & Troubleshooting)

### 5.1 先回答六个问题，再运行修复命令

| 问题 | 最小需要的信息 |
|---|---|
| 更新是什么？ | KB/title、质量/功能/驱动、架构 |
| 机器是什么？ | OS release、build/UBR、edition、语言/FoD、管理方式 |
| 哪个阶段失败？ | 最后已完成交接点、尚未完成的下一动作 |
| 何时失败？ | 本地时间、时区、触发方式、重试/重启时间 |
| 哪个对象失败？ | WU identity、package、CBS/Setup session、DO FileId（若存在） |
| 最终结果是什么？ | error、rollback、pending restart 或仅管理端延迟 |

没有真实 trace 时只能提出假设，不能把“可能有代理问题”写成已确认根因。

### 5.2 日志地图：哪个问题看哪一层

| 阶段/问题 | 原始证据 | 能证明什么 | 不能单独证明什么 |
|---|---|---|---|
| Scan / WU coordination | `%windir%\Logs\WindowsUpdate\*.etl`；转换后的 WindowsUpdate.log | 本次服务交互、对象、handler 进度 | CBS 所有资源的最终正确性 |
| 下一阶段未触发 | `C:\ProgramData\USOShared\Logs` 中 orchestration ETL | 编排请求、决策与交接 | 仅凭等待就证明 downloader 损坏 |
| 通知未出现 | 同目录 NotificationUxBroker ETL（若存在） | 通知路径是否执行 | 更新未安装 |
| Download | DO operational log、`Get-DeliveryOptimizationStatus` | 当前任务、内容与来源统计（视版本） | 无活动任务等于过去没有下载 |
| CBS install | `%windir%\Logs\CBS\CBS.log` 及保留的归档 | package/session、组件处理与错误 | 只看一条历史错误就定位本次失败 |
| DISM repair | `%windir%\Logs\DISM\dism.log` + CBS | 命令参数、servicing/repair 结果 | DISM 100% 必然修复目标故障 |
| Feature upgrade | Panther、Rollback、MoSetup、SetupDiag | 阶段、操作、迁移与回退 | 直接替代普通 LCU 的 CBS 分析 |
| 高层结果 | WindowsUpdateClient/Operational、Setup/System events | 时间线与结果锚点 | 单个 event 覆盖全部因果细节 |

目录和可用 channel 随版本变化，应先枚举。保留原始 ETL/EVTX 和 CBS 归档；生成文本不等于替代原始证据。[R03][R19][R21]

### 5.3 最小采集示例：保留证据，不重置更新组件

以下脚本**只用于明确授权的目标设备**。它创建证据目录、导出事件与日志，不安装更新，不执行 cache reset。证据可能含设备/账号/URL 信息，对外分享前脱敏。长日志、锁定文件与空间不足需按实际处理；出现 copy warning 不代表文件从未存在。

```powershell
# 建议使用提升权限的 Windows PowerShell。
$root = Join-Path $env:SystemDrive (
    'WU-Evidence-' + (Get-Date -Format 'yyyyMMdd-HHmmss')
)
New-Item -ItemType Directory -Path $root -ErrorAction Stop | Out-Null

Get-Date -Format o | Out-File (Join-Path $root 'capture-time.txt')
Get-TimeZone | Format-List | Out-File (Join-Path $root 'timezone.txt')
Get-Service -Name wuauserv,UsoSvc,DoSvc,BITS,TrustedInstaller |
    Select-Object Name,Status,StartType |
    Export-Csv (Join-Path $root 'services.csv') -NoTypeInformation

$channels = @(
    'Microsoft-Windows-WindowsUpdateClient/Operational',
    'Microsoft-Windows-DeliveryOptimization/Operational',
    'Setup',
    'System'
)
foreach ($channel in $channels) {
    $safeName = $channel -replace '[/\\]', '_'
    $destination = Join-Path $root ($safeName + '.evtx')
    & wevtutil.exe epl $channel $destination
    if ($LASTEXITCODE -ne 0) {
        Write-Warning "Export failed or channel unavailable: $channel"
    }
}

$sources = @(
    (Join-Path $env:windir 'Logs\WindowsUpdate'),
    (Join-Path $env:windir 'Logs\CBS'),
    (Join-Path $env:windir 'Logs\DISM'),
    (Join-Path $env:ProgramData 'USOShared\Logs')
)
$index = 0
foreach ($source in $sources) {
    $index++
    if (Test-Path -LiteralPath $source) {
        Copy-Item -LiteralPath $source `
            -Destination (Join-Path $root ("raw-$index")) `
            -Recurse -ErrorAction Continue
    }
}

Get-WindowsUpdateLog -LogPath (Join-Path $root 'WindowsUpdate.log')
Write-Output $root
```

`Get-WindowsUpdateLog` 产生**静态快照**，后续重现后要重新生成；保留原始 ETL 尤其重要。上面的 live copy 不是原子快照，文件可能持续写入，所以将它当作初步采集而非保证无丢失的完整取证。全文命令仅供阅读和选择，本文编写时没有执行它们。[R03]

### 5.4 关联日志时的正确顺序

1. 用用户时间和结果事件界定一次尝试，不把上周错误混入今天。
2. 在 WU 找 ServiceID、UpdateID + revision、caller、操作结果。
3. Download 问题关联实际 content/FileId/URL（如果日志提供）。
4. Install 问题从 handler 交接关联 package 与 CBS session。
5. 确认相应 session 中错误如何传播，并检查最终结果。
6. 重启后继续追踪；时区和日志时间表示不一致时先标准化。

PID/TID 只在相关进程生命周期中有意义；revision number、service-side revision ID 和 client local ID 不可互换。不能拿两台机器相同的 Local ID 就断言是同一个更新。[R03]

推荐的证据条目格式：

```text
File: <原始文件名 / 归档名>
Location: <行号或 ETW timestamp / record index>
Object: <UpdateID.revision / package / session>
Raw evidence: <逐字原文，附前后文>
Established fact: <此证据直接证明什么>
Inference: <推断，及替代解释>
Missing evidence: <还缺少什么才能确定>
Next action: <最小化下一采集或验证>
```

### 5.5 问题 A：Scan 成功，但目标更新不出现

**可能原因：**选错 logical source、WSUS 未同步/批准、更新不适用、目标 release/pause/deferral、服务未提供、feature safeguard hold。它们是候选方向，不是凭 UI 就能确定的根因。[R04][R15][R20]

**排查顺序：**确认 scan 完成与 source → 比较相同设备条件 → 核实 offering/policy → 查本机适用性与更新说明。功能升级有 safeguard 时，不建议为了“证明能装”直接强行绕过。

**Expected：**没有提供，但有完整条件解释。<br>
**Unexpected：**source、设备状态和规则都满足预期，却仍缺对象，且缺少过滤原因。<br>
**下一步：**提供该次 scan 的 WU ETL、effective policy、目标 identity 与正常设备对比，不先删除 DataStore。

### 5.6 问题 B：扫描直接报错

**可能原因：**DNS/TCP、TLS inspection、代理认证、WSUS 服务响应或 metadata 协议错误。

**排查思路：**按网络层次拆开，确认失败请求到底属于哪个 endpoint 和身份；SOAP fault 要读 fault 内容，不能见 SOAP 就认定一定是服务器故障。WSUS-only 环境中，某条外部 Service Locator 警告不必然使本地 WSUS scan 失败。[R01][R10][R26]

**关键工具：**WU ProtocolTalker、代理日志、证书链与服务端 IIS/WSUS 证据。TCP 测试只证明端口连通，不证明 TLS、认证和请求语义正确。

### 5.7 问题 C：Scan 有更新，Download 0% 或反复重试

先区分 **未创建下载任务** 与 **已创建但无法传输**：

| 已有证据 | 最可能停点 | 下一调查方向 |
|---|---|---|
| scan 返回候选，USO 没有发起下一阶段 | 编排/策略边界 | USO ETL、暂停/执行条件 |
| DO 有任务且 origin 响应失败 | transport/content access | 实际 URL、状态码、代理与内容服务器 |
| peer 不可用但 origin 持续增长 | 优化失效而非整体下载失败 | peer scope/端口；不要误报整个更新阻断 |
| 字节接收后反复校验失败 | 内容一致性或传输设备改写等候选方向 | hash/完整错误、同一内容多次结果 |

**关键命令：**`Get-DeliveryOptimizationStatus`（若目标版本提供）；必要时枚举 DO 模块和 event channel。不要只看 BITS。

**典型配置陷阱：**Windows 11 的 DO mode 100 已弃用，某些内容可能失败并出现 `0x80D03002`。若意图是禁 peer，核对 mode 0，而不是把 DO 服务禁用；变更前记录当前策略及批准影响。[R08][R09][R25][R27]

### 5.8 问题 D：下载完成，但安装没有开始

**Expected：**USO 明确等待允许时间、策略或其他前置条件。<br>
**Unexpected：**没有解释性等待，也没有对应 handler/CBS session。<br>
**卡点：**Orchestrator → install handler 的交接，而不是尚未证明的 CBS 执行失败。

先看 USOShared orchestration，再看 WU handler 是否被调用。只有确认 handler 已调用 CBS，才把 CBS 作为本次主要失败层。[R03]

### 5.9 问题 E：CBS 安装失败、source missing 或 component corruption

| Code | 官方语义/定位提示 | 不能直接下的结论 |
|---|---|---|
| `0x800F081F` | `CBS_E_SOURCE_MISSING` | 只是重新下载目标 MSU 就一定好 |
| `0x80073712` | `ERROR_SXS_COMPONENT_STORE_CORRUPT` | 删除 SoftwareDistribution 可重建 WinSxS |
| `0x800F0831` | `CBS_E_STORE_CORRUPTION` | 看到此码已确定唯一缺失文件 |
| `0x800F0982` | `PSFX_E_MATCHING_COMPONENT_NOT_FOUND` | 所有 PSFX 错误都是同一种网络故障 |
| `0x800F0984` | `PSFX_E_MATCHING_BINARY_MISSING` | 组件目录存在就代表 binary 完整 |
| `0x800F0986` | `PSFX_E_APPLY_FORWARD_DELTA_FAILED` | 差分失败一定说明磁盘损坏 |

错误码缩小调查范围，不替代相关对象与上下文。先从 CBS 确认缺什么、属于哪个组件/版本、是 metadata、payload 还是依赖。[R07]

**有证据后，按批准范围选择修复：**

```powershell
# 查询/诊断：CheckHealth 不执行一次完整新扫描。
DISM.exe /Online /Cleanup-Image /CheckHealth
DISM.exe /Online /Cleanup-Image /ScanHealth

# 以下会修改系统，需维护许可、空间与可用 source。
DISM.exe /Online /Cleanup-Image /RestoreHealth
sfc.exe /scannow
```

DISM repair 与 SFC 不是同义操作：前者关注 image/component store 修复，后者检查与修复受保护系统文件。修复后查看日志与结果，再测试目标更新；不要只凭命令进度到 100% 宣告根因消失。[R07][R17]

**离线/受限网络 source 示例，必须先替换路径与 index：**

```powershell
DISM.exe /Get-WimInfo /WimFile:D:\sources\install.wim
DISM.exe /Online /Cleanup-Image /RestoreHealth `
    /Source:WIM:D:\sources\install.wim:1 /LimitAccess
```

`:1` 只是示例，必须与目标映像匹配；检查 release、架构、edition/index、语言和所需 servicing level。较旧 source 缺少所需新版本文件时仍会失败。`/LimitAccess` 阻止该 repair 操作向 WU 获取备用内容，可能正是修复失败的条件之一。[R17]

### 5.10 问题 F：Install 成功但长期 Pending Restart

**可能原因：**维护窗口、Active Hours、用户安排、deadline/grace、更新类型或设备行为差异。

先确定是“未发生预期重启”，还是“已经重启，但待完成状态仍存在”。前者看 scheduling；后者看 CBS/boot servicing 最终结果。`RebootPending` 等 registry 标记可作线索，不存在一个能代表所有产品与全部重启原因的万能键。[R13][R18]

不要手工删除 pending 标记来让控制台变绿。这改变的是证据/状态，而不是保证执行完原本排队的工作。

### 5.11 问题 G：重启时回滚

**Expected：**安装/启动失败后系统按可用机制恢复到可运行状态。<br>
**Unexpected：**相同目标、相同阶段持续失败，或者恢复后仍有损坏。<br>
**核心判断：**回滚是结果，不是原因。

普通 LCU 先看跨重启 CBS 与相关 Setup/System events；完整 feature upgrade 先运行/读取 SetupDiag，再读相应 Panther/Rollback。对 Setup，需要最终 fatal failure、phase/operation 及其前置因果；不是看到 `setuperr.log` 任何一条 Error 就完成诊断。[R19][R21]

```powershell
# 使用已从可信 Microsoft 来源获取的 SetupDiag，按实际路径执行。
.\SetupDiag.exe /LogsPath:C:\WU-Evidence\UpgradeLogs `
    /Output:C:\WU-Evidence\SetupDiagResults.log
```

`/LogsPath` 指向保存的升级日志进行离线分析，不表示正在修复 offline Windows image。Setup 自动运行时的结果还可能位于 `%windir%\Logs\SetupDiag\SetupDiagResults.xml`。[R19]

### 5.12 问题 H：手动 MSU 成功/失败，能推断什么？

| 对照结果 | 支持的事实 | 仍未证明 |
|---|---|---|
| Catalog 包成功，管理路径失败 | 该本地包在当时条件下可以安装 | 一定是 scan source 问题；网络从未参与 |
| 两条路径在同一 CBS package 同样失败 | 共同 servicing 层更值得关注 | 两次 payload 与先决条件完全相同 |
| MSU 报不适用 | 这次 package 检查未接受目标 | OS 无需任何后续修复 |

离线安装前先看 KB 的先决条件与 checkpoint；DISM `/Add-Package` 并不代替所有完整依赖检查。不推荐用 `/IgnoreCheck` 把适用性问题强行“绕过去”。[R11][R16]

### 5.13 教学时间线：怎样把“卡住”改写成可证伪结论

以下是**虚构教学事件**，不是原始日志语法或真实案例：

```text
10:00:00  Scan 返回目标更新 U
10:01:00  Download 任务建立
10:04:00  所需传输完成并通过相关验证
10:05:00  Handler 交给 CBS package P，session S
10:05:30  S 报告缺少所需 component binary，随后失败
10:05:31  WU 收到安装失败
```

**可确认的事实：**请求已穿过 scan、download 和 installer handoff；失败发生在 S 对应的安装路径。<br>
**合理推断：**优先调查本地组件/修复内容与 package 依赖，而非“没有网”。<br>
**尚不能确定：**binary 为什么缺失、哪次操作导致、是否存在存储问题。<br>
**最小下一步：**取得 S 的原始 CBS 上下文与 package/component 身份，核对修复 source；有存储异常迹象再扩展 I/O 调查。

这比“Windows Update 卡住，重置一下”更精确，也保留了否定假设的空间。

## 6. 实战经验 (Practical Tips)

### 6.1 最佳实践：先找责任边界，再选择修复动作

```text
没有扫描结果 -> source / applicability / offering / scan network
没有下载任务 -> orchestration handoff
有任务无内容 -> transport / proxy / origin / disk
有内容无安装 -> readiness / policy / handler handoff
CBS 执行失败 -> package / component / dependency / repair source
没有预期重启 -> scheduling
重启后失败 -> finalization / boot servicing / Setup rollback
本机成功而报表旧 -> report freshness / management synchronization
```

这是定位路线，不是根因列表。每次向下进入一层，都应有上一个交接已经完成的证据。

### 6.2 常见误区与反例

| 误区 | 正确理解 |
|---|---|
| `winget upgrade` 没有更新，说明 Windows 没有补丁 | WinGet 包源的应用升级与 OS Windows Update 是不同工作流 |
| 浏览器能打开网站，所以 WU 网络正常 | 身份、代理、协议与 TLS pinning 都可能不同 |
| 所有更新下载都由 BITS 执行 | 现代 DO 路径不能靠 BITS 任务单独判断 |
| 100% 一定代表新版本已经全部生效 | 下载、在线安装、重启与验证是不同边界 |
| CBS 有历史 Error，所以本次失败根因就是它 | 必须匹配本次对象、session、时间与最终结果 |
| SSU 合并 LCU 后不再有依赖 | 交付合并不取消 servicing 依赖和 KB 要求 |
| Cumulative 表示永远只需要一个 MSU | checkpoint 和离线定制是重要例外 |
| WinSxS 很大，删掉旧目录即可 | 硬链接与组件库存让手工删除危险且容易误判空间 |
| `UsoClient`/`wuauclt` 返回了就证明 scan/install 成功 | 不以未承诺的命令行为或进程退出替代会话结果 |

WinGet 仅用于说明产品边界；本文不把之前某次应用升级命令的结果用作 OS 更新诊断。

### 6.3 性能考量：分开“等待时间”与“执行时间”

下载优化关注内容复用、合理 peer group、MCC 和网络限速；安装优化关注足够磁盘空间、I/O、CPU 和组件健康；重启时长关注需要在后续阶段完成的工作。把更大带宽当成所有安装慢问题的答案，会错过真正瓶颈。[R08][R12][R25]

没有适用于所有更新的固定磁盘空间数值。要考虑下载内容、展开/重建、Component Store 工作空间和恢复相关空间；不能只看 MSU 文件大小。

### 6.4 安全注意：证据优先、最小变更、可回退

不要常规化执行：清空 WinSxS、删除 `pending.xml`、改写 CBS registry、手工替换系统 binary、禁用 signature/TLS 验证、长期禁用更新服务。它们不能作为本文的通用修复建议。

Cache reset、服务重置与 component cleanup 都有范围、证据损失和回退影响；只有与已确认问题匹配并有批准时才使用。生产设备先保留证据与维护安排，特别是集群、关键服务和加密设备。

### 6.5 验证完成的退出条件

| 层 | 退出条件 |
|---|---|
| 扫描 | 目标 source 上会话完成，结果可解释 |
| 内容 | 当前目标所需内容可用且验证成功 |
| 安装 | 目标 installer/session 成功，或清楚报告剩余 restart 工作 |
| 重启 | 必要重启与最终处理完成，没有同一事务继续回滚 |
| 库存 | 目标 package/build 与适用性检查相符 |
| 业务 | 关键功能健康；管理报表最终与客户端状态协调 |

“不再弹错误”不足以替代这些条件；同样，“管理报表还没更新”也不足以否定本机已完成。

## 7. 与相关技术的对比 (Comparison with Related Technologies)

| 维度 | WU / Microsoft Update + client policies | WSUS / Configuration Manager 路径 | Catalog + WUSA / DISM |
|---|---|---|---|
| 发现/提供 | 服务与设备策略共同影响 | 管理服务、批准/部署与客户端评估影响 | 人工选包，必须自己确认目标与说明 |
| 内容路径 | DO/CDN、可能的缓存/peer | 取决于实际管理部署与内容配置 | 指定本地/共享位置，某些条件仍可需要其他内容 |
| 安装引擎 | 依据更新类型 | 依据更新类型；不替代 CBS/Setup | 同样仍受 servicing 与依赖约束 |
| 主要优势 | 自动化、分阶段提供与管理 | 企业批准与部署控制 | 隔离、对照验证、离线映像维护 |
| 主要风险 | 策略/提供差异被误认为安装故障 | 混合来源与部署层增加关联复杂度 | 架构/版本/checkpoint/source 选错 |
| 最佳证据 | WU + USO + DO + installer | 管理端证据 + 同样的客户端证据 | 命令结果 + DISM/CBS + KB 要求 |

| 技术对比 | 为什么不是替代关系 |
|---|---|
| UUP vs CBS | 内容与 action planning 模型 vs component servicing engine |
| DO vs BITS | 传输机制取决于路径；都不是适用性或 CBS 修复引擎 |
| DISM vs SFC | image/store servicing vs 受保护系统文件检查/修复 |
| Quality update vs full Feature Upgrade | 组件服务化为主 vs OS 升级与迁移为主 |
| Scan metadata vs repair source | 判断更新对象 vs 提供修复所需文件 |
| WinGet vs Windows Update | 应用 package 管理工作流 vs OS/相关更新工作流 |

选择工具要按问题层：需要证明运输用 DO/network 证据；需要证明 servicing 用 CBS；需要证明迁移/回退用 Setup。没有一个“万能更新日志”能替代所有层。[R01][R02][R03][R07][R16][R19]

## 8. 参考资料 (References)

以下均为公开 Microsoft 文档，已在 **2026-11-03** 通过 `web_fetch` 访问验证。链接验证只代表当天可访问，不保证链接永久有效。编号在中英文两部分共享。

| ID | 文档与完整 URL | 用途 |
|---|---|---|
| R01 | [How Windows Update works](https://learn.microsoft.com/en-us/windows/deployment/update/how-windows-update-works) — https://learn.microsoft.com/en-us/windows/deployment/update/how-windows-update-works | 四阶段、扫描、ServiceID、WUA proxy |
| R02 | [Get started with Windows Update](https://learn.microsoft.com/en-us/windows/deployment/update/windows-update-overview) — https://learn.microsoft.com/en-us/windows/deployment/update/windows-update-overview | UUP、USO、Arbiter、CompDB、Action List |
| R03 | [Windows Update log files](https://learn.microsoft.com/en-us/windows/deployment/update/windows-update-logs) — https://learn.microsoft.com/en-us/windows/deployment/update/windows-update-logs | ETW、USO/CBS、标识关联、静态文本快照 |
| R04 | [Use Windows Update client policies and WSUS together](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-wsus) — https://learn.microsoft.com/en-us/windows/deployment/update/wufb-wsus | 分类别 scan source 与版本差异 |
| R05 | [Windows client updates, channels, and tools](https://learn.microsoft.com/windows/deployment/update/get-started-updates-channels-tools) — https://learn.microsoft.com/windows/deployment/update/get-started-updates-channels-tools | 更新与管理体系概览 |
| R06 | [Servicing stack updates](https://learn.microsoft.com/en-us/windows/deployment/update/servicing-stack-updates) — https://learn.microsoft.com/en-us/windows/deployment/update/servicing-stack-updates | SSU 与 LCU |
| R07 | [Fix Windows Update corruptions and installation failures](https://learn.microsoft.com/en-us/troubleshoot/windows-server/installing-updates-features-roles/fix-windows-update-errors) — https://learn.microsoft.com/en-us/troubleshoot/windows-server/installing-updates-features-roles/fix-windows-update-errors | CBS、CSI、PSFX、repair 与错误语义 |
| R08 | [How Delivery Optimization works](https://learn.microsoft.com/en-us/windows/deployment/do/delivery-optimization-workflow) — https://learn.microsoft.com/en-us/windows/deployment/do/delivery-optimization-workflow | 内容来源、peer、完整性校验 |
| R09 | [Using a proxy with Delivery Optimization](https://learn.microsoft.com/en-us/windows/deployment/do/delivery-optimization-proxy) — https://learn.microsoft.com/en-us/windows/deployment/do/delivery-optimization-proxy | DO 代理、身份与 TLS inspection |
| R10 | [Windows Update security](https://learn.microsoft.com/en-us/windows/deployment/update/windows-update-security) — https://learn.microsoft.com/en-us/windows/deployment/update/windows-update-security | metadata TLS、pinning、内容信任 |
| R11 | [Checkpoint cumulative updates and Microsoft Update Catalog usage](https://learn.microsoft.com/en-us/windows/deployment/update/catalog-checkpoint-cumulative-updates) — https://learn.microsoft.com/en-us/windows/deployment/update/catalog-checkpoint-cumulative-updates | checkpoint 与离线安装 |
| R12 | [Manage the Component Store](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/manage-the-component-store?view=windows-11) — https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/manage-the-component-store?view=windows-11 | WinSxS、组件与硬链接 |
| R13 | [Manage device restarts after updates](https://learn.microsoft.com/en-us/windows/deployment/update/waas-restart) — https://learn.microsoft.com/en-us/windows/deployment/update/waas-restart | Active Hours、重启控制与旧策略限制 |
| R14 | [Searching, Downloading, and Installing Updates](https://learn.microsoft.com/en-us/windows/win32/wua_sdk/searching--downloading--and-installing-updates) — https://learn.microsoft.com/en-us/windows/win32/wua_sdk/searching--downloading--and-installing-updates | WUA API 示例与 offline scan |
| R15 | [IUpdateSearcher::Search](https://learn.microsoft.com/en-us/windows/win32/api/wuapi/nf-wuapi-iupdatesearcher-search) — https://learn.microsoft.com/en-us/windows/win32/api/wuapi/nf-wuapi-iupdatesearcher-search | 查询条件与返回语义 |
| R16 | [DISM OS Package Servicing Command-Line Options](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism-operating-system-package-servicing-command-line-options?view=windows-11) — https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism-operating-system-package-servicing-command-line-options?view=windows-11 | package inventory、安装约束与 checkpoint |
| R17 | [Configure a Windows Repair Source](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-a-windows-repair-source?view=windows-11) — https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-a-windows-repair-source?view=windows-11 | source 匹配、语言、servicing level |
| R18 | [Enforce compliance deadlines with policies](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-compliancedeadlines) — https://learn.microsoft.com/en-us/windows/deployment/update/wufb-compliancedeadlines | deadline、grace 与 effective deadline |
| R19 | [SetupDiag](https://learn.microsoft.com/en-us/windows/deployment/upgrade/setupdiag) — https://learn.microsoft.com/en-us/windows/deployment/upgrade/setupdiag | 升级诊断、结果与 offline analysis |
| R20 | [Safeguard holds for Windows](https://learn.microsoft.com/en-us/windows/deployment/update/safeguard-holds) — https://learn.microsoft.com/en-us/windows/deployment/update/safeguard-holds | 功能升级未提供与安全边界 |
| R21 | [Log files and resolving upgrade errors](https://learn.microsoft.com/en-us/windows/deployment/upgrade/log-files) — https://learn.microsoft.com/en-us/windows/deployment/upgrade/log-files | Panther、Rollback 与致命失败关联 |
| R22 | [Windows 10 upgrade issues troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-client/setup-upgrade-and-drivers/windows-10-upgrade-issues-troubleshooting) — https://learn.microsoft.com/en-us/troubleshoot/windows-client/setup-upgrade-and-drivers/windows-10-upgrade-issues-troubleshooting | 经典 Setup 阶段；外推现代行为需核实 |
| R23 | [Update Windows installation media with Dynamic Update](https://learn.microsoft.com/windows/deployment/update/media-dynamic-update?branch=main) — https://learn.microsoft.com/windows/deployment/update/media-dynamic-update?branch=main | Dynamic Update 与映像维护 |
| R24 | [OperationResultCode](https://learn.microsoft.com/en-us/windows/win32/api/wuapi/ne-wuapi-operationresultcode) — https://learn.microsoft.com/en-us/windows/win32/api/wuapi/ne-wuapi-operationresultcode | WUA 操作结果枚举 |
| R25 | [Delivery Optimization reference](https://learn.microsoft.com/en-us/windows/deployment/do/waas-delivery-optimization-reference) — https://learn.microsoft.com/en-us/windows/deployment/do/waas-delivery-optimization-reference | DO 设置、默认值与带宽参数 |
| R26 | [Windows Update error code list by component](https://learn.microsoft.com/en-us/windows/deployment/update/windows-update-error-reference) — https://learn.microsoft.com/en-us/windows/deployment/update/windows-update-error-reference | WU 子系统错误码 |
| R27 | [Delivery Optimization FAQ](https://learn.microsoft.com/en-us/windows/deployment/do/waas-delivery-optimization-faq) — https://learn.microsoft.com/en-us/windows/deployment/do/waas-delivery-optimization-faq | mode 100 弃用与 `0x80D03002` |
| R28 | [Repair a Windows Image](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/repair-a-windows-image?view=windows-11) — https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/repair-a-windows-image?view=windows-11 | CheckHealth/ScanHealth/RestoreHealth 与 LimitAccess |

---

# Deep Dive: Windows Update Scan, Download, Install and Commit

**Topic:** Windows Update end-to-end lifecycle<br>
**Category:** Windows Servicing / Deployment / Troubleshooting<br>
**Level:** Advanced, explained progressively<br>
**Last Updated:** 2026-11-03

> **Thirty-second model:** Scan determines what this device should receive; Download obtains and validates the required content; Install delegates the change to the appropriate engine; Restart/Commit completes any remaining work. **USO schedules, WU discovers and coordinates, DO transports, and CBS or Setup implements the change.**
>
> **Scope and evidence:** This is a public-documentation-based learning article, not a diagnosis of a particular computer. Examples and timelines are schematic, not customer logs or internal source-code traces. No Windows Update diagnostics were run while writing it. Modern Windows clients are the primary scope; validate Server policies and release-specific behavior separately. Traditional CBS quality updates are the main path, with full feature upgrades treated as a separate branch. This is not a detailed Hotpatch guide.
>
> Read sections 1–3 for the model, section 4 for configuration, and section 5 for investigation. Reference identifiers resolve to the verified public source table in the Chinese section and the English reference list at the end.

## 1. Overview

Windows Update is a **policy-governed transition from the device's current state to an intended state**, not simply a file downloader. A visible KB entry can involve multiple update identities, payload files, packages, and component changes. A UI row, a WU update object, and a CBS package are different units of observation.[R01][R02][R03]

Three questions must remain separate: What applies? How is the content obtained safely? How is the running OS changed consistently? A successful scan does not prove installation, and completed transfer does not prove that restart-dependent changes are active. Management systems add targeting, approval, and scheduling decisions without replacing the client's actual installer.[R01][R04][R05]

The most useful support question is therefore not “Is Windows Update broken?” but **“What is the last proven handoff, and which expected transition has not happened?”**

### 1.1 A responsibility model

| Responsibility | Question |
|---|---|
| Orchestration | May this operation proceed now? |
| Discovery and applicability | Which updates are relevant and visible? |
| Composition planning | What content and actions are required? |
| Transport | Can the required content be obtained and verified? |
| Installation | Can the intended package/component transition execute? |
| Finalization | Did required restart work complete successfully? |
| Verification/reporting | Does the actual state match the intended state? |

This is a conceptual model, not a promise of strictly serial execution or a one-process-per-row implementation.

## 2. Core Concepts

### 2.1 Identity is the bridge between logs

| Identity/object | Meaning | Investigation use |
|---|---|---|
| KB number | Human-facing article/update identifier | Locate installation requirements and release notes |
| UpdateID + RevisionNumber | WU update object and revision | Correlate discovery and operation records |
| Service-side revision ID | Source-specific identifier | Do not compare blindly across WU and WSUS |
| Client local ID | Device-local datastore identifier | Do not compare blindly across computers |
| Bundle / child update | Parent/child update relationships | A parent need not carry its own payload |
| CBS package identity | Servicing deployment object | Link handler activity to CBS sessions and inventory |
| Component identity | Versioned OS resource grouping | Identify the resource or dependency that fails |
| Payload/content identity | Content required for deployment | Link download failures to specific files/tasks |

Build the chain **WU identity → handler/package → installer session → final state**. A familiar title alone is not sufficient.[R01][R03][R12]

### 2.2 Applicability, offering, selection, and permission are different

An update may be applicable but not currently offered, offered but not selected, or selected but waiting for an execution condition. A useful set model is:

```text
Deployment candidates =
    source-visible updates
    intersect service/deployment offering
    intersect device applicability
    intersect caller selection
    intersect effective policy
```

This is an explanatory model, not a fixed sequence of internal evaluators. Decisions can occur on the service and the client. A missing feature update can be deliberate protection rather than a scan defect.[R04][R15][R20]

### 2.3 Component boundaries

| Component | Role | Important boundary |
|---|---|---|
| Settings UI | Request and display | Display progress is not a transactional progress measure |
| USO / `UsoSvc` | Sequence and schedule operations | Waiting is not automatically installer failure |
| WUA / `wuauserv` | Update service interactions, objects, coordination | Not every file change is performed by WUA |
| UUP UpdateAgent / Arbiter | Plan composition and content/actions | Not necessarily a permanently separate executable |
| DO / `DoSvc` | Content delivery and reuse | Does not decide update applicability |
| CBS / Windows Modules Installer | Windows component/package servicing | Does not replace update offering policy |
| Windows Setup | Full OS upgrade and migration | Not the universal path for every feature transition |

UUP and CBS are complementary: one organizes publishing/content/planning, while the other can implement component servicing.[R02][R06]

Process names, service arrangements, and detailed traces are build-dependent. Observing `TrustedInstaller` or `TiWorker.exe` is a lead to correlate, not proof that the target KB is progressing.

### 2.4 Three layers of state

Keep the update object's properties, the operation result, and the final machine state separate. WUA operation result values are 0 NotStarted, 1 InProgress, 2 Succeeded, 3 SucceededWithErrors, 4 Failed, and 5 Aborted. These values are **not HRESULTs**. Inspect individual update outcomes and restart requirements rather than interpreting only a batch result.[R14][R24]

### 2.5 Three stores, three purposes

`SoftwareDistribution\DataStore` holds local WU metadata/records; download working areas and the DO cache hold acquired content; `WinSxS` is the Component Store used for servicing and recovery-related capabilities. They are not interchangeable.[R03][R12][R25]

Deleting transport cache cannot recreate missing component-store metadata or binaries. Resetting local metadata can remove useful correlation and change subsequent scan behavior. Never treat WinSxS as disposable download cache.

## 3. How It Works

### 3.1 Overall architecture

```text
Policies / deployment / user or API request
                    |
                    v
              Orchestrator
                    |
             WU discovery
                    |
           Applicability result
                    |
       Device state + CompDB / Arbiter
                    |
       Required content + action list
                    |
      DO / origin / cache / eligible peers
                    |
          Validation / preparation
                    |
            Installer handler
               /         \
              v           v
             CBS         Setup
              |           |
         Component     OS upgrade /
          servicing      migration
               \         /
                v       v
           Restart if required
                    |
            Final verification
```

This describes responsibilities, not a network route. The metadata source and payload source can differ, and both differ from the local installation engine. “The client bypassed WSUS” is incomplete until the affected plane is identified.[R01][R02][R04][R08]

### 3.2 Stage 0: request and orchestration

A scheduled operation, user action, or supported API caller initiates work. Record the update type, caller, time, OS build, management configuration, and execution context. Different callers can use different criteria and workflows; a custom API search is not an exact reproduction of Settings behavior.[R14][R15]

**Expected:** A related session begins, or a documented/evidenced scheduling condition explains waiting.<br>
**Unexpected:** No correlated session, repeated cancellation, or effective policy inconsistent with intent.<br>
**Next evidence:** USO orchestration ETL, policy origin, and the precise trigger time.

A stopped service snapshot is not enough to prove a defect: on-demand behavior and the task's actual execution window matter. Do not permanently change every update service to Automatic as a generic repair.

### 3.3 Stage 1: scan

#### Step 1: select the effective logical service

Determine the source for this update class. Windows Update, Microsoft Update, and WSUS are separate logical services. Modern scan-source policies can direct different categories to different services; the presence of `WUServer` alone does not establish the source of every operation.[R04]

A ServiceID is a client-side abstraction, not a permanent physical endpoint mapping. Confirm the current request/configuration instead of hard-coding a hostname from a GUID.[R01]

#### Step 2: synchronize metadata

Scanning exchanges update descriptions/rules and relevant state; it is not equivalent to downloading all applicable installation binaries. Cached metadata can reduce later synchronization. Full/delta scan terminology describes the metadata context, not full/differential payload encoding.[R01]

Separate “Which objects might apply?” from “Which files must be acquired for the chosen target?” Network traffic alone does not answer which question is currently being processed.

#### Step 3: evaluate state and relationships

OS release/build, architecture, edition/product, installed components, language/features, prerequisites, and existing install state can affect applicability. Driver matching adds device-specific considerations. Update supersedence describes relationships; it does not prove the superseding update is already installed.[R01][R12][R15]

```text
Device attributes + prerequisite detection + installation state
                + update relationships
                           |
                           v
                 Relevant update objects
```

An “already installed,” “not applicable,” or “not offered” outcome can all be valid and require different explanations.

#### Step 4: return results

The caller receives the outcome for its configured criteria and service. A successful search with zero results is not, on its own, corruption.[R15]

| Expected | Unexpected | Next evidence |
|---|---|---|
| Correct service, completed synchronization, explainable result | Protocol failure, wrong source, evaluation error | WU ProtocolTalker, source/configuration, failing request |
| Policy or hold explains absence | Apparently equivalent devices differ without explanation | Matched policy/build/language comparison |

“Offline scan” can mean cached local evaluation or a registered offline scan-package service. Neither implies a complete equivalent of the online catalog or the presence of installable payloads.[R14]

### 3.4 Scan networking: more than browser connectivity

Investigate the chain **DNS → TCP → TLS identity → proxy/HTTP authentication → update protocol → metadata evaluation**.

WUA detection URLs and other WUA URLs have documented differences in proxy selection/fallback; WSUS also has policy-specific behavior. DO has a separate documented proxy model using WinHTTP automatic discovery and context-dependent authentication. “All services only use `netsh winhttp`” is therefore an unsafe generalization.[R01][R09]

Pinned Microsoft update-service connections can fail under TLS inspection. Allowing a browser to reach a URL does not establish that the service context can perform the required pinned protocol exchange. Use narrowly scoped, documented proxy exceptions, not disabled certificate validation. Do not freeze CDN addressing in hosts files.[R10]

### 3.5 Stage 2: plan and download

#### Step 1: compute the required composition

CompDB describes target composition, available differential packages, and conditions. The Arbiter combines that information with device state to produce required content and actions.[R02]

```text
Desired target
    minus usable local state
    plus missing prerequisites and required language/feature resources
    -> content requirements and installation actions
```

This is a reasoning model, not a specified arithmetic algorithm. It explains why two computers installing the same KB need not download identical byte counts.

#### Step 2: acquire content

DO can combine the original HTTP source, Microsoft Connected Cache, and eligible peers. It is an optimization and delivery mechanism, not the applicability engine. Peer communication uses TCP 7680 rather than SMB. CDN/origin fallback is useful only when the relevant origin remains reachable and allowed.[R08][R25]

No peer traffic can simply mean no eligible/useful peers. An active origin transfer can still represent a successful update download. Conversely, a local peer port test does not prove that origin, cloud coordination, or metadata access works.

#### Step 3: validate content

Trusted metadata supports piece verification; peer content is checked before acceptance. The completed installation content is also subject to relevant hashes and digital signatures. Transport security and publisher/content authenticity are separate protection layers.[R08][R10]

```text
Trusted verification data
       -> validate received pieces
       -> assemble content
       -> validate installation content
       -> installer
```

Do not equate HTTP payload delivery with permission to install tampered content. Equally, do not assume every current content path uses exactly one transport protocol.

#### Step 4: prepare/stage

Transfer completion can be followed by decompression, differential reconstruction, and staging. Differential reconstruction can fail when a required matching component or binary is missing. That is a different question from whether the network received bytes successfully.[R07]

**DO pieces and binary differentials are different granularities.** The former concern delivery; the latter concern representing/reconstructing updated resources.

| Expected | Unexpected | Next evidence |
|---|---|---|
| Tasks/bytes progress and validation succeeds | Origin/proxy denial, repeated validation failure, write failure | Actual content URL/status, DO evidence, disk/proxy records |
| Preparation follows transfer | Transfer completes but staging fails | WU handler and corresponding preparation/CBS context |

### 3.6 SSU, LCU, and checkpoints

An SSU improves the servicing infrastructure; an LCU delivers cumulative quality changes. Combined delivery on modern supported paths does not abolish servicing prerequisites or package-specific installation instructions.[R06]

Checkpoint cumulative updates, introduced on the documented Windows 11 24H2 / Server 2025 paths, provide newer differential baselines. WU/WSUS normally coordinate dependencies. Catalog/offline workflows must account for required checkpoints rather than assuming one MSU always contains every required baseline.[R11]

This does not mean “install every intervening month.” It means satisfy the relevant checkpoint dependencies. Offline language/FoD customization can require the complete applicable checkpoint content set.

### 3.7 Stage 3: installation

The handler delegates according to update type. Typical OS quality updates use CBS; full OS upgrades use Setup. Drivers and other products are not governed by every detail of the CBS path.[R01][R02]

#### A conceptual CBS progression

```text
Request
 -> validate package/metadata and relevant conditions
 -> resolve dependencies and intended component state
 -> prepare resources
 -> execute online servicing
 -> record remaining restart-dependent work where required
 -> return result and restart requirement
 -> complete later work and final state
```

This is a diagnostic abstraction, not a guaranteed sequence of log strings.

Package metadata, component manifests, catalogs, and payloads serve different purposes. Possessing the expected DLL does not establish consistency of the package, its metadata, dependencies, and associated resources.[R07][R12]

#### Why WinSxS matters

The Component Store supports updates, features, repair, and supported removal/recovery scenarios. Hard links can expose the same underlying data through multiple paths, so naive folder-size totals overstate unique disk consumption.[R12]

Manually copying a DLL into System32 cannot reproduce the entire servicing transaction. File version alone is not sufficient verification; inspect the supported inventory and installer results.

#### Read CBS by transaction, not by isolated keywords

Identify the target package and session, locate the causally meaningful failure in that transaction, then follow propagated errors to its final result. CBS/CSI labels offer different levels of observation; they do not justify inventing internal calls that were not recorded.[R07]

The first Error in an entire multi-session log may be unrelated. The final Error may be a wrapper. Full Setup upgrades need their own fatal-failure/phase analysis rather than blindly applying one search-direction rule.[R19][R21]

#### Inventory states

| Observed state | Practical interpretation | Unsafe conclusion |
|---|---|---|
| Staged | Content is prepared in the servicing system | The change is fully active |
| Install Pending | Work remains, often restart-related | Delete pending state to complete installation |
| Installed | Inventory reports the package installed | Every process has loaded every new resource |
| Superseded | A replacement relationship/state exists | All older component resources are disposable |

Use the target's actual DISM/package output. This table is not a universal enum for every update technology.[R16]

### 3.8 Why installation can be slow without network activity

An engineering timing model is:

```text
Total time =
    policy wait + scan + transfer + preparation
    + online servicing + restart wait
    + boot/offline work + verification
```

This is not a Microsoft implementation formula. It helps distinguish productive CPU/I/O work, scheduling waits, persistent retries, and genuine lack of progress.

| Observation | Working hypothesis | Missing proof |
|---|---|---|
| Network quiet; CBS/session actions continue | Servicing/preparation is progressing | Target object and resources actually advance |
| High disk latency during related work | Storage or filtering bottleneck | Correlated I/O/error evidence |
| Quiet resources; explicit USO wait | Scheduling rather than execution | Effective policy and next eligibility |
| Same object/error repeats | Persistent failure/retry loop | Full causal error and retry timeline |

A fixed percentage is not a stable internal phase identifier across builds and update types. Do not label a hang from percentage alone.

### 3.9 Stage 4: restart and commit

Some work completes online; other work requires later shutdown/boot processing. The documented WU Commit model is not a guarantee of one universal ACID transaction spanning all update types.[R01][R13]

```text
Online install complete
    -> no restart needed -> verify
    -> restart needed -> schedule -> restart -> finalize
                                             -> failure/recovery/rollback
```

“Restart never happened” and “restart happened and rolled back” locate the problem at different boundaries. After restart, correlate installer completion, target package state, applicable build/UBR, and functional health.

History describes operations; inventory describes current state. Neither should be used in isolation to represent the entire machine.

### 3.10 Deadline and grace-period reasoning

For the documented Windows 11 22H2-and-later deadline model:

```text
Effective restart deadline =
    later of:
      first scan discovery + configured deadline
      restart-required time + configured grace period
```

After that effective boundary, active hours do not grant indefinite deferral. Older releases and different update levels have different policy interactions; verify the target rather than applying the formula universally.[R18]

For illustration only: discovery at November 3, 10:00 plus two days yields November 5, 10:00; restart-required at November 4, 18:00 plus one day yields November 5, 18:00. The latter is the effective boundary. These are fictional times and not recommended universal settings.

### 3.11 Separate branch: full feature upgrades

A full feature upgrade includes OS replacement/configuration and migration, not merely a larger CBS quality update. The classic phase model is Downlevel, SafeOS, First Boot, and Second Boot/OOBE.[R22]

| Phase | Main investigation question | Evidence |
|---|---|---|
| Downlevel | Did compatibility/preparation/migration gathering succeed? | `$Windows.~BT\Sources\Panther` |
| SafeOS | Did offline, boot, and storage-related work succeed? | Phase/operation and Panther/Rollback |
| First Boot | Can the new OS initialize with required drivers/configuration? | Setup and driver/device evidence |
| Second Boot / OOBE | Did later migration/configuration finish? | Panther, UnattendGC, Rollback |
| Rollback | Which fatal operation caused recovery? | Rollback and SetupDiag |

The classic phase reference is Windows 10 documentation. Verify actual phase/operation and reboot behavior for modern targets; do not promise an identical number of reboots. Lightweight enablement-package transitions should not be forced into the full-upgrade model.

Dynamic Update refreshes content used for Setup/upgrading/media maintenance; it is not another name for the monthly LCU.[R23]

### 3.12 Reporting closes the loop

Client inventory, a later applicability scan, UI history, and management reports can represent different observation times. Confirm report freshness before treating a stale console as an installation failure.

For machine comparisons, match release/build, architecture, language/FoD, effective policy/source, target identity, and time window. “Both are Windows 11” is not a sufficient control group.

## 4. Key Configurations

### 4.1 Defaults versus effective values

| Setting | Default/unconfigured meaning | Purpose | Typical investigation |
|---|---|---|---|
| Intranet update service / `WUServer` / `UseWUServer` | No explicit enterprise WSUS source when unconfigured | Enterprise endpoint selection | Wrong source or migration residue |
| Per-class scan source | Depends on OS and other source policy | Quality/Feature/Driver/Other source | Mixed-management behavior |
| Pause/deferral/target release | No additional restriction from that unconfigured policy | Offering/cadence | Missing target update |
| Configure Automatic Updates | No single numeric default claimed for every release | Automatic scheduling | Work not initiated |
| Active hours | Device/user/policy-dependent | Avoid disruption before enforced boundaries | Restart delays |
| Deadline/grace | No extra configured deadline value | Compliance enforcement | Unexpected restart timing |
| `DODownloadMode` | Client reference describes LAN(1) | 0 no peers; 1 LAN; 2 Group; 3 Internet; 99 Simple | Delivery design |
| DO mode 100 | Deprecated on Windows 11 | Not a recommended bypass remedy | Some content can fail |
| `DOMaxCacheAge` | 259200 seconds / three days | Cache retention | Poor content reuse |
| `DOMaxCacheSize` | 20% | Cache quota | Disk/cache pressure |
| DO bandwidth settings | Documented 0 values mean dynamic adjustment | Rate management; check units | Slow delivery/WAN contention |
| Repair source | WU when permitted, or configured files | Component repair content | Source missing |
| `/LimitAccess` | Not applied unless specified | Prevent WU repair fallback for that operation | Isolated repair testing |

Sources: [R04][R13][R17][R18][R25]. An absent registry value is not a universal “disabled” state. Manage policy through GPO/MDM rather than copying raw values without their documented prerequisites.

### 4.2 Read-only inspection examples

```powershell
Get-Service -Name wuauserv,UsoSvc,DoSvc,BITS,TrustedInstaller |
    Select-Object Name,Status,StartType

Get-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate' `
    -ErrorAction SilentlyContinue
Get-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU' `
    -ErrorAction SilentlyContinue
Get-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\DeliveryOptimization' `
    -ErrorAction SilentlyContinue

Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' |
    Select-Object ProductName,DisplayVersion,CurrentBuild,UBR

netsh winhttp show proxy
DISM.exe /Online /Get-Packages /Format:Table
```

Capture policy origin as well as values. `gpresult` does not report the complete MDM configuration, and WinHTTP's displayed static proxy is not a record of every actual DO proxy decision.[R04][R09]

### 4.3 API observation without downloading or installing

This optional example performs a **real metadata scan**, which can contact the configured service and update local metadata. It is not a pure read of an existing log, not a complete Settings reproduction, and not an installation script. Run only on an authorized device.

```powershell
$session = New-Object -ComObject Microsoft.Update.Session
$session.ClientApplicationID = 'WU Learning Diagnostic Scan'
$searcher = $session.CreateUpdateSearcher()
$result = $searcher.Search(
    "IsInstalled=0 and IsHidden=0 and Type='Software'"
)

[pscustomobject]@{
    ResultCode = [int]$result.ResultCode
    Count = $result.Updates.Count
}

for ($i = 0; $i -lt $result.Updates.Count; $i++) {
    $update = $result.Updates.Item($i)
    [pscustomobject]@{
        Title = $update.Title
        UpdateID = $update.Identity.UpdateID
        Revision = $update.Identity.RevisionNumber
        Downloaded = $update.IsDownloaded
    }
}
```

The configured search options and service matter. Zero matches are a result to explain, not automatic evidence of a broken client. This snippet is illustrative and was not executed in producing the article.[R14][R15][R24]

## 5. Common Issues & Troubleshooting

### 5.1 Establish the attempt before diagnosing it

Record target KB/title/type, OS build and architecture, language/features, management/source, exact trigger time and timezone, last completed handoff, update/package/session identities, and final observed result.

Without actual traces, hypotheses must stay hypotheses.

### 5.2 Evidence map

| Question | Primary evidence | Limitation |
|---|---|---|
| Did scan/coordination succeed? | `%windir%\Logs\WindowsUpdate\*.etl` and generated text | Does not prove every installer resource is healthy |
| Why did the next operation not start? | `C:\ProgramData\USOShared\Logs` orchestration ETL | Waiting alone does not prove a broken downloader |
| Did a notification execute? | NotificationUxBroker ETL where present | Notification failure is not installation failure |
| What is the download doing? | DO Operational and available DO status cmdlets | Empty current status does not prove no historical transfer |
| Why did CBS fail? | CBS.log and retained archives | Must match the target transaction |
| What did repair do? | DISM log plus CBS | Reaching 100% does not prove the update now works |
| Why did a full upgrade roll back? | Panther/Rollback/MoSetup/SetupDiag | Not a universal replacement for LCU CBS analysis |
| When did high-level results occur? | WindowsUpdateClient, Setup, System events | Summary events do not contain the whole causal chain |

Enumerate actual logs/channels on the target. Preserve raw ETL/EVTX and archives. `Get-WindowsUpdateLog` produces a static snapshot and must be rerun after later activity.[R03][R19][R21]

### 5.3 Evidence collection without cache reset

The Chinese section contains a complete initial collection script. For an English-only workflow, the minimum commands below create output files and may require elevation; they do not install updates:

```powershell
$out = Join-Path $env:SystemDrive (
    'WU-Evidence-' + (Get-Date -Format 'yyyyMMdd-HHmmss')
)
New-Item -ItemType Directory -Path $out -ErrorAction Stop | Out-Null
Get-Date -Format o | Out-File (Join-Path $out 'capture-time.txt')
Get-TimeZone | Format-List | Out-File (Join-Path $out 'timezone.txt')
Get-WindowsUpdateLog -LogPath (Join-Path $out 'WindowsUpdate.log')
wevtutil.exe epl Microsoft-Windows-WindowsUpdateClient/Operational `
    (Join-Path $out 'WindowsUpdateClient.evtx')
```

Also preserve the relevant raw WU ETLs, USO logs, CBS archives, and Setup logs when applicable. Live copies are not atomic snapshots; record missing, locked, or changing files. Evidence can contain identifiers and URLs: redact before external sharing.

### 5.4 Correlation method

1. Bound a single attempt using time, trigger, and result.
2. Identify service, update GUID/revision, caller, and operation in WU.
3. For download issues, associate actual content/task/URL identifiers.
4. For install issues, associate handler, package, and installer session.
5. Follow the relevant failure to the final operation result.
6. Continue across restart; normalize timezones before ordering events.

PID/TID can change across sessions. Update revision numbers, service revision IDs, and client local IDs are not interchangeable.[R03]

Use the following evidence contract:

```text
File / archive:
Line or timestamp / record:
Target object / session:
Verbatim evidence with context:
Established fact:
Inference and alternatives:
Missing evidence:
Smallest next action:
```

### 5.5 Issue A: scan succeeds but the update is absent

**Possible causes:** source mismatch, deployment/approval, non-applicability, pause/deferral/target-release policy, service offering, or feature safeguard hold.[R04][R15][R20]

**Reasoning:** First prove the scan completed against the intended source, then check offering and device equivalence. Do not immediately reset local metadata or bypass a safety hold.

**Expected:** Absence has an evidenced policy/applicability explanation.<br>
**Unexpected:** Apparently equivalent state and policy produce different results with no explanation.<br>
**Next collection:** The specific scan ETLs, effective policy, target identity, and a properly matched healthy-device comparison.

### 5.6 Issue B: scan fails outright

Locate the exact failing request and stage: resolution, connection, TLS, authentication, protocol, or evaluation. Read SOAP fault details rather than assuming every SOAP fault is a server defect.

A blocked external Service Locator request in a WSUS-only environment does not necessarily invalidate a locally configured WSUS scan. Correlate the failure that actually terminates the intended operation.[R01][R10][R26]

### 5.7 Issue C: download remains at 0% or retries

First distinguish “no download task was created” from “a task exists but cannot transfer.”

| Observation | Boundary | Next investigation |
|---|---|---|
| Candidate exists; no next-stage request | Orchestration | USO and execution conditions |
| Task exists; origin/proxy rejects access | Transport | Actual URL, response, identity, proxy |
| No peers; origin progresses | Optimization only | Peer eligibility/scope if bandwidth matters |
| Received content repeatedly fails validation | Content integrity path | Full validation error and repeated content identity |

Use available DO status/operational evidence rather than BITS alone. Windows 11 mode 100 is deprecated; if the requirement is “disable peers,” review mode 0 rather than disabling the downloader. `0x80D03002` can be associated with an unsupported bypass configuration for some content.[R08][R25][R27]

### 5.8 Issue D: download completes but installation never starts

**Expected:** There is an explicit, explainable orchestration wait.<br>
**Unexpected:** No wait reason and no corresponding handler/installer session.<br>
**Boundary:** Orchestrator-to-installer handoff, not yet proven CBS execution failure.

Investigate USO and WU handoff before diagnosing a CBS transaction that may not exist.[R03]

### 5.9 Issue E: CBS source/component failures

| Error | Meaning/direction |
|---|---|
| `0x800F081F` | `CBS_E_SOURCE_MISSING` |
| `0x80073712` | `ERROR_SXS_COMPONENT_STORE_CORRUPT` |
| `0x800F0831` | `CBS_E_STORE_CORRUPTION` |
| `0x800F0982` | `PSFX_E_MATCHING_COMPONENT_NOT_FOUND` |
| `0x800F0984` | `PSFX_E_MATCHING_BINARY_MISSING` |
| `0x800F0986` | `PSFX_E_APPLY_FORWARD_DELTA_FAILED` |

Decode the error, then identify the relevant component/package/version and missing metadata or content. The code narrows the search; it does not establish the unique cause.[R07]

After preserving evidence and obtaining maintenance approval:

```powershell
# Diagnostic inspection; CheckHealth does not perform a full new scan.
DISM.exe /Online /Cleanup-Image /CheckHealth
DISM.exe /Online /Cleanup-Image /ScanHealth

# These commands can modify the system.
DISM.exe /Online /Cleanup-Image /RestoreHealth
sfc.exe /scannow
```

For a selected repair source:

```powershell
DISM.exe /Get-WimInfo /WimFile:D:\sources\install.wim
DISM.exe /Online /Cleanup-Image /RestoreHealth `
    /Source:WIM:D:\sources\install.wim:1 /LimitAccess
```

The path and index are placeholders to verify, not defaults to copy blindly. Check OS release, architecture, edition/index, languages, and required servicing level. A source below the required level can still fail. `/LimitAccess` prevents WU fallback for that repair operation; it is not a general Windows networking switch.[R17][R28]

DISM/store repair and SFC/protected-system-file repair serve related but different purposes. Inspect results and retry the original target rather than declaring success from progress percentage.

### 5.10 Issue F: installation reports success but restart remains pending

Separate “the expected restart has not occurred” from “a restart occurred but required work did not finish.” Investigate scheduling for the former and servicing/finalization for the latter.

Active hours, deadlines, grace, user scheduling, and version-specific policy can matter. No single registry key universally describes every product's restart requirement. Never delete pending markers merely to improve compliance display.[R13][R18]

### 5.11 Issue G: rollback after restart

Rollback is an outcome, not the root cause. For traditional quality updates, correlate CBS across restart and relevant events. For a full upgrade, use SetupDiag and the correct Panther/Rollback phase logs.

```powershell
.\SetupDiag.exe /LogsPath:C:\WU-Evidence\UpgradeLogs `
    /Output:C:\WU-Evidence\SetupDiagResults.log
```

This parses saved logs; it does not repair an offline Windows image. Setup's automatically generated result can be at `%windir%\Logs\SetupDiag\SetupDiagResults.xml`.[R19]

Locate the fatal phase/operation and causal context. A random earlier Error in `setuperr.log` is not sufficient; SetupDiag's matched result must still be interpreted with the underlying logs.[R19][R21]

### 5.12 Issue H: manual package installation differs

Manual success proves that the selected local package could install under those conditions. It does not uniquely prove a source-policy defect. The same package failing in the same CBS session context through two paths strengthens the case for a shared servicing problem, but compare inputs and prerequisites first.

Honor architecture, release, checkpoint, and KB requirements. DISM package installation is not a substitute for complete dependency reasoning, and `/IgnoreCheck` is not a recommended generic applicability workaround.[R11][R16]

### 5.13 A schematic timeline

The following is fictional teaching data, not real log syntax:

```text
10:00  Scan identifies update U
10:01  Download task begins
10:04  Required transfer and related validation complete
10:05  Handler delegates package P to CBS session S
10:05  S reports missing required component binary and fails
10:05  WU receives installation failure
```

**Established:** Discovery, transport, and installer handoff occurred.<br>
**Inference:** Prioritize component/package/repair-source investigation rather than assuming no internet access.<br>
**Not established:** Why the resource disappeared or whether storage is involved.<br>
**Next evidence:** Raw CBS context for S, package/component identity, and compatible repair-source availability.

## 6. Practical Tips

### 6.1 Route by the last proven boundary

```text
No scan result -> source / offering / applicability / protocol
No download task -> orchestration
Task without usable content -> transport / proxy / origin / disk
Content without installer session -> readiness / policy / handoff
CBS failure -> component / package / dependency / repair source
No expected restart -> scheduling
Failure after restart -> finalization / boot work / Setup rollback
Healthy client, stale console -> reporting freshness / synchronization
```

This is an investigation map, not a list of confirmed causes.

### 6.2 Avoid common category errors

| Misconception | Correction |
|---|---|
| WinGet reports no upgrade, so the OS is fully patched | Application package sources and OS updating are different workflows |
| Browser access proves WU service access | Context, proxy, protocol, and pinning can differ |
| BITS has no jobs, so no download exists | DO may own the relevant transfer |
| UI 100% means all changes are active | Transfer, install, restart, and final verification are separate |
| Any CBS Error explains the present failure | Match object, session, time, and final outcome |
| Combined SSU/LCU has no dependencies | Delivery packaging does not remove servicing requirements |
| Cumulative means a single offline file always suffices | Checkpoint and customization dependencies matter |
| WinSxS folder size equals safely reclaimable space | Hard links and required resources invalidate that assumption |
| A command/process returned, so updating succeeded | Verify the actual operation and final target state |

### 6.3 Performance

Optimize the measured segment: transport reuse for WAN load, CPU/I/O and component health for servicing, and restart coordination for end-to-end compliance. A faster connection cannot resolve every installation bottleneck.

There is no universal safe free-space number for all packages and releases. Account for transfer, expansion/reconstruction, servicing workspace, and recovery needs, not just the MSU's compressed size.

### 6.4 Safety

Do not routinely delete WinSxS, `pending.xml`, CBS registry state, or protected binaries; disable signature/TLS verification; or permanently disable update services. Such actions are not generic repairs.

Cache resets and cleanup have evidence and recovery consequences. Preserve the relevant artifacts, obtain approval, scope the change, and verify the original failure condition afterward.

### 6.5 Exit criteria

Confirm explainable scan results, usable validated content, successful target installer state, completion of required restart work, correct inventory/build where applicable, and functional health. Reconcile management reporting separately if its data is delayed.

“The error disappeared” is weaker than this evidence, while “the console has not refreshed” is not enough to disprove local success.

## 7. Comparison with Related Technologies

| Dimension | WU/Microsoft Update with client policies | WSUS/Configuration Manager paths | Catalog + WUSA/DISM |
|---|---|---|---|
| Discovery/offering | Service and device policy | Enterprise approval/deployment plus client evaluation | Administrator selects package |
| Content delivery | DO/origin/cache/peers as configured | Depends on management/content design | Specified package location, with dependencies still relevant |
| Installer | Update-type-dependent | Same underlying client engines remain relevant | Servicing/dependency rules still apply |
| Strength | Automated offering and orchestration | Enterprise control | Isolation tests and image/offline maintenance |
| Main trap | Offering differences mistaken for installation defects | Mixed-source complexity | Wrong release/architecture/checkpoint/source |
| Evidence | WU + USO + DO + installer | Management evidence plus client evidence | Command results + DISM/CBS + KB requirements |

| Pair | Distinction |
|---|---|
| UUP / CBS | Publishing/composition/content model / component servicing engine |
| DO / BITS | Delivery mechanisms; neither is an applicability or store-repair engine |
| DISM / SFC | Image/component-store servicing / protected-system-file checks and repair |
| Quality update / full feature upgrade | Component servicing / OS upgrade and migration |
| Scan metadata / repair source | Update-object reasoning / required repair resources |
| WinGet / Windows Update | Application package-management workflow / OS update workflow |

Choose evidence by the responsibility under investigation. There is no single log that fully substitutes for all layers.[R01][R02][R03][R07][R16][R19]

## 8. References

All references below were accessed with `web_fetch` on **November 3, 2026**. Only successfully retrieved public Microsoft sources are included. Reference IDs are shared with the Chinese source table.

- **R01** [How Windows Update works](https://learn.microsoft.com/en-us/windows/deployment/update/how-windows-update-works) — lifecycle, source identity, scan/proxy behavior.
- **R02** [Get started with Windows Update](https://learn.microsoft.com/en-us/windows/deployment/update/windows-update-overview) — UUP, USO, Arbiter, CompDB, action lists.
- **R03** [Windows Update log files](https://learn.microsoft.com/en-us/windows/deployment/update/windows-update-logs) — logs and identity correlation.
- **R04** [Windows Update client policies and WSUS](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-wsus) — scan-source policies.
- **R05** [Windows client updates, channels, and tools](https://learn.microsoft.com/windows/deployment/update/get-started-updates-channels-tools) — management/update context.
- **R06** [Servicing stack updates](https://learn.microsoft.com/en-us/windows/deployment/update/servicing-stack-updates) — SSU and cumulative delivery.
- **R07** [Fix Windows Update corruptions and installation failures](https://learn.microsoft.com/en-us/troubleshoot/windows-server/installing-updates-features-roles/fix-windows-update-errors) — CBS/CSI, error semantics, repair.
- **R08** [How Delivery Optimization works](https://learn.microsoft.com/en-us/windows/deployment/do/delivery-optimization-workflow) — delivery sources and verification.
- **R09** [Using a proxy with Delivery Optimization](https://learn.microsoft.com/en-us/windows/deployment/do/delivery-optimization-proxy) — proxy context and inspection.
- **R10** [Windows Update security](https://learn.microsoft.com/en-us/windows/deployment/update/windows-update-security) — metadata and content security.
- **R11** [Checkpoint cumulative updates and Catalog usage](https://learn.microsoft.com/en-us/windows/deployment/update/catalog-checkpoint-cumulative-updates) — checkpoint dependencies.
- **R12** [Manage the Component Store](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/manage-the-component-store?view=windows-11) — component storage and hard links.
- **R13** [Manage device restarts after updates](https://learn.microsoft.com/en-us/windows/deployment/update/waas-restart) — restart policies.
- **R14** [Searching, Downloading, and Installing Updates](https://learn.microsoft.com/en-us/windows/win32/wua_sdk/searching--downloading--and-installing-updates) — WUA SDK workflow.
- **R15** [IUpdateSearcher::Search](https://learn.microsoft.com/en-us/windows/win32/api/wuapi/nf-wuapi-iupdatesearcher-search) — criteria and search results.
- **R16** [DISM package servicing options](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism-operating-system-package-servicing-command-line-options?view=windows-11) — inventory and package operations.
- **R17** [Configure a Windows Repair Source](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-a-windows-repair-source?view=windows-11) — source compatibility and maintenance.
- **R18** [Enforce compliance deadlines](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-compliancedeadlines) — deadline/grace calculation.
- **R19** [SetupDiag](https://learn.microsoft.com/en-us/windows/deployment/upgrade/setupdiag) — upgrade-log analysis.
- **R20** [Safeguard holds for Windows](https://learn.microsoft.com/en-us/windows/deployment/update/safeguard-holds) — protected feature-update offering.
- **R21** [Log files and resolving upgrade errors](https://learn.microsoft.com/en-us/windows/deployment/upgrade/log-files) — phase/log/fatal-error correlation.
- **R22** [Windows 10 upgrade troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-client/setup-upgrade-and-drivers/windows-10-upgrade-issues-troubleshooting) — classic upgrade phases; verify modern-target details.
- **R23** [Update installation media with Dynamic Update](https://learn.microsoft.com/windows/deployment/update/media-dynamic-update?branch=main) — upgrade/media content.
- **R24** [OperationResultCode](https://learn.microsoft.com/en-us/windows/win32/api/wuapi/ne-wuapi-operationresultcode) — operation outcomes.
- **R25** [Delivery Optimization reference](https://learn.microsoft.com/en-us/windows/deployment/do/waas-delivery-optimization-reference) — policy defaults and parameters.
- **R26** [Windows Update errors by component](https://learn.microsoft.com/en-us/windows/deployment/update/windows-update-error-reference) — subsystem error interpretation.
- **R27** [Delivery Optimization FAQ](https://learn.microsoft.com/en-us/windows/deployment/do/waas-delivery-optimization-faq) — deprecated bypass mode and related failures.
- **R28** [Repair a Windows Image](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/repair-a-windows-image?view=windows-11) — repair commands and LimitAccess.
