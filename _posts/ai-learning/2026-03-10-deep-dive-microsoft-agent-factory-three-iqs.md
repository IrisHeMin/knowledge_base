---
layout: post
title: "Deep Dive: Microsoft Agent Factory — Work IQ, Fabric IQ & Foundry IQ"
date: 2026-03-10
categories: [Knowledge, Cloud-AI]
tags: [microsoft, agent-factory, work-iq, fabric-iq, foundry-iq, copilot, microsoft-365, azure, fabric, ai-agents]
type: "deep-dive"
---

# Deep Dive: Microsoft Agent Factory — Work IQ, Fabric IQ & Foundry IQ

**Topic:** Microsoft Agent Factory 中的三大智能层：Work IQ、Fabric IQ、Foundry IQ  
**Category:** Cloud AI / Enterprise AI Platform  
**Level:** 中级  
**Last Updated:** 2026-03-10

---

## 📌 中文版

---

### 1. 概述 (Overview)

在 AI Agent（智能体）时代，微软构建了一个宏大的愿景：让 AI 不仅仅是"聊天助手"，而是能够**理解你的工作、理解你的数据、理解你的业务**的真正"数字同事"。这个愿景的核心，就是 **Microsoft Agent Factory**——一个让企业能够大规模构建、部署和管理 AI Agent 的平台体系。

但 Agent 要做到"真正有用"，不能只靠大语言模型（LLM）本身的通用知识。它们需要**组织内部的上下文**——你的邮件、会议、文档、业务数据、数据仓库等等。这就是微软推出的 **三大 IQ 智能层** 要解决的问题：

| IQ 层 | 一句话说明 | 数据来源 |
|--------|-----------|----------|
| **Work IQ** | 理解你**怎么工作**的 | Microsoft 365（邮件、会议、聊天、文档） |
| **Fabric IQ** | 理解你**业务数据**的含义 | Microsoft Fabric（OneLake、Power BI、数据仓库） |
| **Foundry IQ** | 连接你**所有企业知识**的 | Azure Blob、SharePoint、OneLake、公共网络等 |

用一个比喻来理解：
> 如果把 AI Agent 比作一个新入职的员工，那么 **Work IQ** 让它知道"公司里大家怎么协作、最近在讨论什么"；**Fabric IQ** 让它看懂"公司的数据报表和业务指标意味着什么"；**Foundry IQ** 让它能"翻阅公司的知识库和各种文档资料"。

三者各有侧重，但可以**组合使用**，共同为 Agent 提供全面的组织上下文。

---

### 2. 核心概念 (Core Concepts)

#### 2.1 什么是 Agent Factory？

**Agent Factory** 是微软对其 AI Agent 构建平台的统称。核心理念是：**让构建 Agent 像工厂流水线一样标准化、规模化、可治理**。

Agent Factory 涵盖了多个平台和工具：
- **Microsoft Copilot Studio**：低代码/无代码的 Agent 构建工具
- **Microsoft Foundry**（原 Azure AI Foundry）：面向开发者的 Agent 开发平台
- **Agent 365**：Agent 的统一管理控制面板（注册、安全、治理）
- **三大 IQ 层**：为 Agent 提供智能上下文（Work IQ、Fabric IQ、Foundry IQ）

```
┌─────────────────────────────────────────────────────┐
│              Microsoft Agent Factory                 │
│  ┌───────────────┐  ┌───────────────┐  ┌──────────┐ │
│  │ Copilot Studio│  │  Microsoft    │  │ Agent 365│ │
│  │ (低代码构建)  │  │  Foundry      │  │ (治理)   │ │
│  │               │  │ (Pro-code开发)│  │          │ │
│  └───────┬───────┘  └───────┬───────┘  └────┬─────┘ │
│          │                  │               │        │
│          └──────────┬───────┘               │        │
│                     ▼                       │        │
│  ┌─────────────────────────────────────────────────┐ │
│  │           三大 IQ 智能层（Agent 的大脑）         │ │
│  │  ┌───────────┐ ┌───────────┐ ┌────────────────┐ │ │
│  │  │  Work IQ  │ │ Fabric IQ │ │  Foundry IQ    │ │ │
│  │  │ 工作上下文│ │ 业务语义  │ │ 企业知识检索   │ │ │
│  │  └───────────┘ └───────────┘ └────────────────┘ │ │
│  └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

#### 2.2 为什么需要三个 IQ？

一个关键问题是：**大语言模型本身并不知道你公司的任何事情**。它只有公共互联网的训练知识。要让 Agent 在企业内真正有用，就需要把企业的"内部知识"安全地注入到 Agent 的推理过程中。

但"企业知识"本身就分为三大类：

1. **工作协作数据**（谁和谁开了什么会、讨论了什么、邮件说了什么）→ **Work IQ**
2. **结构化业务数据**（销售数据、库存数据、客户数据，以及这些数据的"业务含义"）→ **Fabric IQ**
3. **广泛的非结构化知识**（技术文档、产品手册、知识库文章、外部网页）→ **Foundry IQ**

每一类数据的存储位置、格式、访问控制方式都完全不同，所以需要三个专门的智能层来处理。

---

### 3. 工作原理 (How It Works)

#### 3.1 Work IQ — "理解你怎么工作"

**Work IQ** 是 Microsoft 365 Copilot 背后的智能引擎。它是让 Copilot 能够理解"你的工作上下文"的关键。

##### 三层架构

Work IQ 由三个紧密集成的层组成：

```
┌─────────────────────────────────────┐
│           Skills & Tools            │  ← 执行层：具体技能和工具
│  (调度会议、检索文档、发邮件等)      │
├─────────────────────────────────────┤
│           Context Layer             │  ← 上下文层：记忆与语义理解
│  (Memory、Semantic Index、          │
│   Business Understanding)           │
├─────────────────────────────────────┤
│           Data Layer                │  ← 数据层：原始工作数据
│  (M365 Tenant Data、Copilot        │
│   Connectors、Dynamics 365/        │
│   Dataverse)                        │
└─────────────────────────────────────┘
```

##### 数据层 (Data Layer)

- **Microsoft 365 租户数据**：以 Microsoft Graph 为核心，包含：
  - SharePoint/OneDrive 中的文件（Word、Excel、PPT 等）
  - Outlook 邮件
  - Teams 会议和聊天记录
  - 用户和组织结构信息
  - 协作模式元数据（谁和谁经常合作、沟通频率等）

- **Copilot Connectors**：连接非微软系统的数据（如 Salesforce、SAP 等），预置数百个连接器，也支持自定义。

- **Dynamics 365 / Power Apps 数据（Dataverse）**：将 CRM、ERP 等业务系统数据纳入，让 Copilot 能同时推理"生产力数据"和"业务数据"。例如询问："上周我和供应商的 Teams 通话中提到的零件问题，会怎样影响我的库存和销售？"

##### 上下文层 (Context Layer)

- **Memory（记忆）**：
  - *显式记忆*：用户主动告诉 Copilot 的偏好（如"我喜欢用主动语态"）
  - *隐式记忆*：Copilot 从聊天历史中推断出的持久洞察
  - 未来将融入更多活动模式（来自 Teams、Outlook、Word 等的使用模式）

- **Semantic Index（语义索引）**：不仅做关键词匹配，还做**语义级别的检索**——理解"意思"而非仅仅"字面"。

- **Business Understanding（业务理解）**：通过 Ontology（本体）和 Glossary（术语表）捕获业务工作流的过程性知识，让 Copilot 能像"业务专家"一样理解你的任务。

##### 技能与工具层 (Skills & Tools Layer)

- **Skills（技能）**：专门的指令集，告诉 Copilot "如何做某件事"（如"如何调度会议"、"如何从外部源获取数据"）。
- **Tools（工具）**：实际执行技能的工具——MCP Server 工具、Agent Flows、API、Plugin 等。

##### 安全与隐私

Work IQ 始终尊重已有的 Microsoft 365 权限模型：
- 用户权限（只能访问你有权限看的数据）
- 安全组分配
- 敏感度标签 (Sensitivity Labels)
- DLP 策略
- 符合 GDPR 和 EU Data Boundary

##### Work IQ API

开发者可以通过 **Work IQ API**（RESTful 接口）将 Work IQ 的智能能力集成到自己的应用和 Agent 中。也有 CLI 工具和 MCP Server 模式可用。

---

#### 3.2 Fabric IQ — "理解你的业务数据意味着什么"

**Fabric IQ** 是 Microsoft Fabric 中的语义智能层。如果说 Work IQ 让 Agent 理解"人们怎么工作"，Fabric IQ 则让 Agent 理解"数据说的是什么"。

##### 为什么需要 Fabric IQ？

想象这样一个场景：你的数据仓库里有一张表叫 `tbl_cust_ord_v2`，里面有字段 `qty_shipped`、`dt_exp`、`flg_breach`。对于 AI Agent 来说，这些列名几乎无法理解。

Fabric IQ 的核心目标就是：**为数据赋予业务含义**，建立一套统一的"业务语言"，让 Agent 能理解：
- `qty_shipped` = "已发货数量"
- `dt_exp` = "预计交付日期"
- `flg_breach` = "是否存在合同违约"

##### 核心组件

| 组件 | 说明 |
|------|------|
| **Ontology（本体）** | 核心！定义企业词汇和语义层——实体类型（如 Customer、Order、Shipment）、属性、关系和业务规则，并绑定到真实数据 |
| **Graph（图）** | 图数据库能力，存储节点和边，支持路径查找和关系遍历（如"订单 → 发货 → 温度传感器 → 冷链违约"） |
| **Data Agent（数据 Agent）** | 基于生成式 AI 的对话式问答系统，连接 Ontology 理解业务概念 |
| **Operations Agent（运营 Agent）** | 监控实时数据并推荐业务操作 |
| **Power BI Semantic Model** | 为报表和交互式分析优化的语义模型（度量值、层次结构、关系） |

##### Ontology 是关键

Ontology 是 Fabric IQ 的核心，它做的事情类似于"企业数据字典 + 关系图谱"：

```
┌──────────────┐    places     ┌──────────────┐
│   Customer   │──────────────▶│    Order     │
│              │               │              │
│ - name       │               │ - order_date │
│ - segment    │               │ - total_amt  │
└──────────────┘               └──────┬───────┘
                                      │ contains
                                      ▼
                               ┌──────────────┐    triggers    ┌──────────────┐
                               │   Shipment   │───────────────▶│  Cold Chain   │
                               │              │                │  Breach      │
                               │ - shipped_qty│                │              │
                               │ - exp_date   │                │ - severity   │
                               └──────────────┘                └──────────────┘
```

这个 Ontology 告诉 Agent：
- "Customer"实体有哪些属性，数据在 lakehouse 的哪张表
- "Order"和"Customer"之间是什么关系
- "Cold Chain Breach"代表什么，严重性意味着什么

有了这些语义信息，Agent 就可以回答类似这样的问题：
> "哪些 A 级客户的最近订单出现了冷链违约，违约严重程度是高的？"

##### Fabric IQ 的价值

- **数据统一**：跨 OneLake 中的多个数据源（lakehouse、eventhouse、语义模型）建立统一视图
- **语言一致**：一个概念只定义一次，Power BI、Notebook、Agent 都使用相同的语义
- **治理和信任**：减少跨团队的定义不一致和数据重复
- **跨域推理**：通过 Graph 遍历关系链（如"订单 → 发货 → 传感器 → 违约"）来解释结果
- **AI 就绪**：为 Copilot 和 Agent 提供结构化的 grounding，让回答基于企业语言和业务规则

---

#### 3.3 Foundry IQ — "连接企业所有知识"

**Foundry IQ** 是 Microsoft Foundry（原 Azure AI Foundry）平台中的托管知识层。它解决的是一个非常实际的问题：**企业的知识散布在各处**——Azure Blob Storage、SharePoint 文档库、OneLake、甚至公共网页——Agent 需要一个统一的方式来检索这些知识。

##### 核心概念

```
┌─────────────────────────────────────────┐
│           Knowledge Base                │  ← 顶层资源，编排检索行为
│  ┌─────────────┐  ┌─────────────┐       │
│  │ Knowledge   │  │ Knowledge   │ ...   │  ← 知识源（Azure Blob、
│  │ Source A     │  │ Source B    │       │     SharePoint、OneLake、Web）
│  │(SharePoint) │  │(Azure Blob) │       │
│  └─────────────┘  └─────────────┘       │
│                                         │
│  ┌─────────────────────────────────────┐│
│  │      Agentic Retrieval Engine       ││  ← 智能检索引擎
│  │  - 分解复杂问题为子查询             ││
│  │  - 并行执行搜索                     ││
│  │  - 语义重排序                       ││
│  │  - 聚合统一结果                     ││
│  └─────────────────────────────────────┘│
└─────────────────────────────────────────┘
```

##### Knowledge Base（知识库）

知识库是 Foundry IQ 的顶层资源，它定义：
- 要查询哪些知识源
- 控制检索行为的参数（如检索推理力度：minimal / low / medium）

##### Knowledge Sources（知识源）

支持的数据源包括：
- **Azure Blob Storage**：存储的各种文档
- **SharePoint**：企业文档库
- **OneLake**：Fabric 的统一数据湖
- **公共网络数据**：Web 抓取

Foundry IQ 会自动处理：
- 文档分块 (Chunking)
- 向量嵌入生成 (Vector Embedding)
- 元数据提取
- 增量数据刷新

##### Agentic Retrieval（智能检索）

这是 Foundry IQ 最核心的能力——不是简单的关键词搜索，而是：

1. **问题分解**：将复杂问题拆分为多个子查询
2. **并行搜索**：同时在多个知识源中执行子查询
3. **语义重排序**：用 LLM 对结果进行语义重新排序
4. **结果聚合**：将多源结果统一返回
5. **引用追溯**：返回的数据带有引用标注，可追溯到源文档

##### 权限感知

Foundry IQ 内置了严格的权限控制：
- 同步 Access Control Lists (ACLs)
- 尊重 Microsoft Purview 敏感度标签
- 在查询时强制执行权限（用户只能看到有权限的内容）
- 使用调用者的 Microsoft Entra 身份进行端到端权限验证

---

### 4. 三大 IQ 对比 (Comparison)

| 维度 | Work IQ | Fabric IQ | Foundry IQ |
|------|---------|-----------|------------|
| **定位** | 工作协作智能层 | 业务数据语义层 | 企业知识检索层 |
| **核心问题** | Agent 如何理解"人们的工作方式" | Agent 如何理解"数据的业务含义" | Agent 如何查找"散布各处的知识" |
| **所属平台** | Microsoft 365 Copilot | Microsoft Fabric | Microsoft Foundry (Azure) |
| **数据来源** | 邮件、会议、Teams 聊天、文档、Dynamics 365 | OneLake（lakehouse、eventhouse）、Power BI 语义模型 | Azure Blob、SharePoint、OneLake、公共网络 |
| **核心能力** | 语义索引、记忆系统、技能/工具执行 | Ontology 本体建模、Graph 图遍历、Data Agent | 知识库管理、Agentic Retrieval 智能检索 |
| **典型问题** | "上周和 Alice 的会议里讨论了什么？" | "哪些 A 级客户上月的销售额下降了？" | "公司关于退货政策的文档是怎么说的？" |
| **权限模型** | M365 权限、敏感度标签、DLP | Fabric 工作区权限 | Entra ID + ACL 同步 + Purview 标签 |
| **独立还是组合** | 可独立使用，也可与其他 IQ 组合 | 可独立使用，也可与其他 IQ 组合 | 可独立使用，也可与其他 IQ 组合 |

### 5. 关键配置与参数 (Key Configurations)

#### Work IQ

| 配置项 | 说明 | 常见场景 |
|--------|------|----------|
| Custom Instructions | 用户级自定义指令（如语气偏好、格式要求） | 个性化 Copilot 行为 |
| Saved Memories | 持久化的用户记忆 | 让 Copilot 记住长期偏好 |
| Copilot Connectors | 非微软数据源连接器 | 连接 Salesforce、ServiceNow 等外部系统 |
| Sensitivity Labels | 敏感度标签设置 | 控制哪些内容可被 Copilot 处理 |
| Work IQ API | RESTful API 接入 | 开发者将 Work IQ 集成到自定义应用 |

#### Fabric IQ

| 配置项 | 说明 | 常见场景 |
|--------|------|----------|
| Ontology 定义 | 实体类型、属性、关系、约束和规则 | 建立企业统一数据语义 |
| 数据源绑定 | 将 Ontology 实体绑定到 lakehouse/eventhouse/语义模型 | 连接实际数据 |
| Graph 配置 | 节点、边、遍历规则 | 复杂关系查询和影响分析 |
| Data Agent 知识源 | 连接 Ontology 作为 Agent 数据源 | 让 Agent 理解业务术语 |

#### Foundry IQ

| 配置项 | 说明 | 常见场景 |
|--------|------|----------|
| Knowledge Base | 知识库定义和检索参数 | 设置检索范围和推理力度 |
| Knowledge Sources | 数据源连接（Blob、SharePoint、OneLake、Web） | 连接企业知识存储 |
| Retrieval Reasoning Effort | 检索推理力度（minimal/low/medium） | 平衡检索质量和成本 |
| Indexer Schedule | 索引器增量刷新计划 | 保持知识库数据新鲜 |
| ACL 同步 | 权限控制列表同步 | 确保权限感知的检索 |

---

### 6. 实战经验 (Practical Tips)

#### 最佳实践

- **从 Work IQ 开始**：如果你的组织已有 Microsoft 365 Copilot 许可，Work IQ 是即开即用的。先让员工体验 Copilot 的工作上下文能力。
- **用 Fabric IQ 统一数据语义**：在引入 AI Agent 之前，先花时间定义好 Ontology——这是"一次定义、到处使用"的投资。
- **Foundry IQ 适合知识密集型场景**：如果你的 Agent 需要查阅大量文档（如客服知识库、产品手册），优先配置 Foundry IQ。
- **三者组合使用效果最佳**：让 Agent 同时拥有"工作上下文 + 业务语义 + 知识检索"三重能力。

#### 常见误区

- ❌ **认为只要有 LLM 就够了**：LLM 没有你公司的任何知识，IQ 层才是让 Agent 变"聪明"的关键。
- ❌ **混淆三个 IQ 的用途**：Work IQ ≠ Foundry IQ。前者是理解工作模式，后者是检索文档知识。
- ❌ **忽略权限配置**：IQ 层会继承已有的权限模型，但如果你的 M365/Fabric/Azure 权限本身就配置不当，Agent 返回的结果也会有问题。
- ❌ **跳过 Ontology 直接用 Data Agent**：没有好的 Ontology 定义，Data Agent 无法理解你的业务语言。

#### 安全注意

- 三个 IQ 层都**不存储**用户的源数据，只做按需检索
- 所有检索都基于调用者的身份和权限
- 支持 Microsoft Purview 敏感度标签
- 符合 GDPR 和欧盟数据边界要求
- Agent 365 提供统一的 Agent 安全治理控制面板

---

### 7. 三者如何协同工作 (How They Work Together)

一个完整的企业 Agent 场景可能同时需要三个 IQ 层：

```
用户提问: "上周和供应商 Contoso 的会议中提到的零件短缺问题，
          会怎样影响我们的 Q2 销售预测？请参考公司的供应链应急手册。"

┌──────────────────────────────────────────────────────────┐
│                    AI Agent 的推理过程                      │
│                                                          │
│  Step 1: [Work IQ] 查找上周与 Contoso 的 Teams 会议记录  │
│          → 找到会议摘要：提到 Part-X 短缺，预计延迟 3 周   │
│                                                          │
│  Step 2: [Fabric IQ] 查询业务数据                         │
│          → Ontology 理解 "Part-X" 是什么                  │
│          → 查询 Order 和 Inventory 实体                   │
│          → 分析 Q2 销售预测受影响的订单                    │
│                                                          │
│  Step 3: [Foundry IQ] 检索知识文档                        │
│          → 在 SharePoint 知识库中找到《供应链应急手册》    │
│          → 提取相关的应急措施建议                          │
│                                                          │
│  Step 4: 综合推理并生成回答                               │
│          → 结合三个来源的信息，给出有据可查的分析报告       │
└──────────────────────────────────────────────────────────┘
```

这就是三大 IQ 协同工作的威力——**单独一个 IQ 只能回答部分问题，三者组合才能给出完整的、有上下文的、可追溯的答案**。

---

### 8. 参考资料 (References)

- [Powering Frontier Transformation with Copilot and agents](https://www.microsoft.com/en-us/microsoft-365/blog/2026/03/09/powering-frontier-transformation-with-copilot-and-agents/) — Microsoft 365 Copilot Wave 3 官方公告
- [A Closer Look at Work IQ](https://techcommunity.microsoft.com/blog/microsoft365copilotblog/a-closer-look-at-work-iq/4499789) — Work IQ 深度解析（Tech Community）
- [Microsoft Agent 365: The control plane for AI agents](https://www.microsoft.com/en-us/microsoft-365/blog/2025/11/18/microsoft-agent-365-the-control-plane-for-ai-agents/) — Agent 365 介绍
- [What is Microsoft Foundry](https://learn.microsoft.com/en-us/azure/ai-foundry/what-is-ai-foundry) — Microsoft Foundry 官方文档
- [What is Foundry IQ](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/concepts/what-is-foundry-iq) — Foundry IQ 概念说明
- [Fabric IQ Overview](https://learn.microsoft.com/en-us/fabric/iq/overview) — Fabric IQ 官方概述
- [Work IQ CLI and MCP Server](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/workiq-overview) — Work IQ 开发者文档
- [Why Copilot Studio is the foundation for agentic business transformation](https://www.microsoft.com/en-us/microsoft-copilot/blog/copilot-studio/why-microsoft-copilot-studio-is-the-foundation-for-agentic-business-transformation/) — Copilot Studio 与 Agent Factory

---
---

## 📌 English Version

---

### 1. Overview

In the age of AI Agents, Microsoft has built a grand vision: AI should not just be a "chat assistant," but a true **"digital coworker"** that understands your work, your data, and your business. At the heart of this vision is the **Microsoft Agent Factory** — an ecosystem for enterprises to build, deploy, and manage AI Agents at scale.

But for an Agent to be truly useful, it cannot rely solely on the general knowledge of a Large Language Model (LLM). It needs **organizational context** — your emails, meetings, documents, business data, data warehouses, and more. This is exactly the problem Microsoft's **three IQ layers** are designed to solve:

| IQ Layer | One-Line Summary | Data Source |
|----------|------------------|-------------|
| **Work IQ** | Understands **how you work** | Microsoft 365 (email, meetings, chats, documents) |
| **Fabric IQ** | Understands **what your business data means** | Microsoft Fabric (OneLake, Power BI, data warehouses) |
| **Foundry IQ** | Connects **all your enterprise knowledge** | Azure Blob, SharePoint, OneLake, public web |

An analogy:
> If an AI Agent is a new hire, then **Work IQ** tells it "how people collaborate and what's being discussed"; **Fabric IQ** helps it understand "what data reports and business metrics actually mean"; and **Foundry IQ** enables it to "browse the company's knowledge base and documentation."

The three are independent but can be **used together** to provide comprehensive organizational context for agents.

---

### 2. Core Concepts

#### 2.1 What is Agent Factory?

**Agent Factory** is Microsoft's umbrella term for its AI Agent building platform ecosystem. The core principle: **make building agents as standardized, scalable, and governable as a factory assembly line**.

The Agent Factory encompasses:
- **Microsoft Copilot Studio**: Low-code/no-code agent builder
- **Microsoft Foundry** (formerly Azure AI Foundry): Pro-code agent development platform
- **Agent 365**: Unified management control plane for agents (registry, security, governance)
- **Three IQ Layers**: Provide intelligent context for agents (Work IQ, Fabric IQ, Foundry IQ)

```
┌──────────────────────────────────────────────────────┐
│               Microsoft Agent Factory                 │
│  ┌───────────────┐  ┌────────────────┐  ┌──────────┐ │
│  │ Copilot Studio│  │  Microsoft     │  │ Agent 365│ │
│  │ (Low-code)    │  │  Foundry       │  │(Govern)  │ │
│  │               │  │  (Pro-code)    │  │          │ │
│  └───────┬───────┘  └───────┬────────┘  └────┬─────┘ │
│          │                  │                │        │
│          └──────────┬───────┘                │        │
│                     ▼                        │        │
│  ┌──────────────────────────────────────────────────┐ │
│  │         Three IQ Layers (The Agent Brain)         │ │
│  │  ┌───────────┐ ┌───────────┐ ┌──────────────────┐│ │
│  │  │  Work IQ  │ │ Fabric IQ │ │   Foundry IQ     ││ │
│  │  │Work Context│ │Biz Semant.│ │ Knowledge Retriev││ │
│  │  └───────────┘ └───────────┘ └──────────────────┘│ │
│  └──────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

#### 2.2 Why Three IQs?

LLMs know nothing about your company. To make agents useful in an enterprise, you need to inject "internal knowledge" securely into the agent's reasoning process.

Enterprise knowledge falls into three categories:

1. **Work collaboration data** (who met whom, what was discussed, what emails said) → **Work IQ**
2. **Structured business data** (sales, inventory, customer data, and their "business meaning") → **Fabric IQ**
3. **Broad unstructured knowledge** (technical docs, product manuals, KB articles, web pages) → **Foundry IQ**

Each category has different storage locations, formats, and access control mechanisms, requiring specialized intelligence layers.

---

### 3. How It Works

#### 3.1 Work IQ — "Understanding How You Work"

**Work IQ** is the intelligence engine behind Microsoft 365 Copilot. It makes Copilot understand your work context.

##### Three-Layer Architecture

- **Data Layer**: M365 tenant data (via Microsoft Graph), Copilot Connectors for non-Microsoft systems, Dynamics 365/Dataverse for business system data
- **Context Layer**: Memory (explicit + implicit), Semantic Index (meaning-based retrieval), Business Understanding (ontologies and glossaries from business workflows)
- **Skills & Tools Layer**: Specialized instructions (skills) and execution tools (MCP servers, APIs, plugins)

##### Key Capabilities
- Personalized responses based on your work patterns, relationships, and communication history
- Cross-system reasoning: connect Teams meeting content with Dynamics 365 sales data
- Enterprise-grade security: respects M365 permissions, sensitivity labels, DLP policies

##### Developer Access
- **Work IQ API**: RESTful interface for integrating Work IQ intelligence into custom apps
- **Work IQ CLI & MCP Server**: `npm install -g @microsoft/workiq` for terminal and VS Code integration

---

#### 3.2 Fabric IQ — "Understanding What Your Data Means"

**Fabric IQ** is the semantic intelligence layer for Microsoft Fabric. While Work IQ understands "how people work," Fabric IQ understands "what data tells you."

##### Core Components

| Component | Purpose |
|-----------|---------|
| **Ontology** | Define enterprise vocabulary — entity types (Customer, Order, Shipment), properties, relationships, rules, bound to real data |
| **Graph** | Graph storage and traversal for relationship-heavy queries (e.g., Order → Shipment → Sensor → Breach) |
| **Data Agent** | Conversational Q&A connected to Ontology for business-aware answers |
| **Operations Agent** | Real-time data monitoring with business action recommendations |
| **Power BI Semantic Model** | Curated analytics models for reporting and interactive analysis |

##### Why Ontology Matters
Without Ontology, column names like `qty_shipped` and `flg_breach` are meaningless to agents. Ontology provides the "Rosetta Stone" that translates raw data into business language.

##### Key Benefits
- **Data unification** across OneLake sources
- **Consistent language** — define a concept once, use it everywhere (Power BI, notebooks, agents)
- **Cross-domain reasoning** via Graph traversals
- **AI-ready grounding** for Copilot and agents

---

#### 3.3 Foundry IQ — "Connecting All Enterprise Knowledge"

**Foundry IQ** is the managed knowledge layer in Microsoft Foundry. It solves a practical problem: **enterprise knowledge is scattered** across Azure Blob Storage, SharePoint, OneLake, and even public websites — agents need a unified way to retrieve it.

##### Core Architecture

- **Knowledge Base**: Top-level resource orchestrating retrieval. Defines which sources to query and retrieval parameters.
- **Knowledge Sources**: Connections to Azure Blob, SharePoint, OneLake, and web data. Automatic chunking, vector embedding, metadata extraction, and incremental refresh.
- **Agentic Retrieval Engine**: Multi-query pipeline that:
  1. Decomposes complex questions into subqueries
  2. Executes them in parallel across sources
  3. Semantically reranks results
  4. Returns unified responses with citations

##### Permission-Aware Retrieval
- ACL synchronization for supported sources
- Honors Microsoft Purview sensitivity labels
- Queries run under the caller's Microsoft Entra identity
- End-to-end permission enforcement

---

### 4. Comparison

| Dimension | Work IQ | Fabric IQ | Foundry IQ |
|-----------|---------|-----------|------------|
| **Purpose** | Work collaboration intelligence | Business data semantics | Enterprise knowledge retrieval |
| **Core Question** | How do people work? | What does the data mean? | Where is the knowledge? |
| **Platform** | Microsoft 365 Copilot | Microsoft Fabric | Microsoft Foundry (Azure) |
| **Data Sources** | Email, meetings, Teams chats, docs, Dynamics 365 | OneLake (lakehouse, eventhouse), Power BI semantic models | Azure Blob, SharePoint, OneLake, public web |
| **Key Capability** | Semantic index, memory, skills/tools | Ontology modeling, Graph traversal, Data Agent | Knowledge base, Agentic Retrieval |
| **Example Query** | "What was discussed in last week's meeting with Alice?" | "Which A-tier customers had declining sales last month?" | "What does our return policy document say?" |

### 5. How They Work Together

A real-world scenario requiring all three:

```
User: "The parts shortage discussed in last week's Teams call with
      supplier Contoso — how will it impact our Q2 sales forecast?
      Please reference our supply chain contingency playbook."

Agent Reasoning:
  Step 1: [Work IQ] Find last week's Teams meeting with Contoso
          → Meeting summary: Part-X shortage, 3-week delay expected

  Step 2: [Fabric IQ] Query business data
          → Ontology resolves "Part-X" entity
          → Query Order and Inventory entities
          → Analyze impacted Q2 sales forecast

  Step 3: [Foundry IQ] Retrieve knowledge documents
          → Find "Supply Chain Contingency Playbook" in SharePoint KB
          → Extract relevant contingency measures

  Step 4: Synthesize and respond
          → Combine all three sources into a cited analysis report
```

**No single IQ can answer the full question. Together, they provide complete, contextual, traceable answers.**

---

### 6. References

- [Powering Frontier Transformation with Copilot and agents](https://www.microsoft.com/en-us/microsoft-365/blog/2026/03/09/powering-frontier-transformation-with-copilot-and-agents/) — M365 Copilot Wave 3 announcement
- [A Closer Look at Work IQ](https://techcommunity.microsoft.com/blog/microsoft365copilotblog/a-closer-look-at-work-iq/4499789) — Work IQ deep dive (Tech Community)
- [Microsoft Agent 365: The control plane for AI agents](https://www.microsoft.com/en-us/microsoft-365/blog/2025/11/18/microsoft-agent-365-the-control-plane-for-ai-agents/) — Agent 365 introduction
- [What is Microsoft Foundry](https://learn.microsoft.com/en-us/azure/ai-foundry/what-is-ai-foundry) — Microsoft Foundry docs
- [What is Foundry IQ](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/concepts/what-is-foundry-iq) — Foundry IQ concept
- [Fabric IQ Overview](https://learn.microsoft.com/en-us/fabric/iq/overview) — Fabric IQ overview
- [Work IQ CLI and MCP Server](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/workiq-overview) — Work IQ developer docs
- [Why Copilot Studio is the foundation for agentic business transformation](https://www.microsoft.com/en-us/microsoft-copilot/blog/copilot-studio/why-microsoft-copilot-studio-is-the-foundation-for-agentic-business-transformation/) — Copilot Studio and Agent Factory
