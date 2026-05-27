---
layout: post
title: "Deep Dive: AI-102 Azure AI Engineer — Complete Knowledge Structure (Low → High)"
date: 2026-03-15
categories: [Knowledge, AI]
tags: [ai-102, azure-ai, generative-ai, computer-vision, nlp, agents, rag, document-intelligence, azure-ai-foundry, certification]
type: "deep-dive"
---

# Deep Dive: AI-102 Azure AI Engineer — Complete Knowledge Structure (Low → High)

**Topic:** AI-102: Designing and Implementing a Microsoft Azure AI Solution  
**Category:** Artificial Intelligence / Cloud AI Services  
**Level:** Beginner → Intermediate → Advanced (Progressive)  
**Last Updated:** 2026-03-15  

---

## 中文版 (Chinese Version)

---

### 1. 概述 (Overview)

AI-102 是 Microsoft Azure AI Engineer Associate 认证的官方课程，面向希望在 Azure 上构建 AI 解决方案的软件开发者。课程涵盖 **5 大学习路径、40 个模块**，包括：生成式 AI 应用开发、AI Agent 构建、自然语言处理、计算机视觉、以及信息提取。

本文将 AI-102 的全部知识体系从 **Level 0（基础）到 Level 7（高级）** 逐层展开，帮助你建立完整的知识地图。每一层都建立在前一层的基础上，确保你能循序渐进地掌握所有关键技能。

#### 课程总览 — 5 大学习路径

| # | 学习路径 | 模块数 | 核心主题 |
|---|---------|--------|---------|
| 1 | Develop Generative AI Apps in Azure | 8 | 生成式 AI、RAG、Fine-tuning、Prompt Flow |
| 2 | Develop AI Agents on Azure | 9 | Agent Service、MCP、Multi-agent、A2A |
| 3 | Develop Natural Language Solutions | 10 | 文本分析、CLU、语音、翻译 |
| 4 | Develop Computer Vision Solutions | 8 | 图像分析、OCR、人脸、自定义视觉 |
| 5 | Develop AI Information Extraction Solutions | 5 | Document Intelligence、AI Search、Content Understanding |

---

### 2. Level 0 — 基础平台与准备 (Foundation & Setup)

> 🎯 **目标**：理解 Azure AI 生态，搭建开发环境

#### 2.1 Azure AI 服务全景图

Azure AI 服务体系的核心组成：

```
┌─────────────────────────────────────────────────────┐
│                  Microsoft Foundry                   │
│  (统一的 AI 开发平台，原 Azure AI Studio)              │
│                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │ Model    │ │ Prompt   │ │ Agent    │             │
│  │ Catalog  │ │ Flow     │ │ Service  │             │
│  └──────────┘ └──────────┘ └──────────┘             │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │            Azure AI Services                  │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐│   │
│  │  │Language │ │Vision  │ │Speech  │ │Document││   │
│  │  │Service │ │Service │ │Service │ │Intelli.││   │
│  │  └────────┘ └────────┘ └────────┘ └────────┘│   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │          Azure OpenAI Service                 │   │
│  │   GPT-4o | GPT-4 | DALL-E | Whisper          │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │          Azure AI Search                      │   │
│  │   索引 | 技能集 | 向量搜索 | 语义排序          │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

#### 2.2 关键基础概念

| 概念 | 说明 |
|------|------|
| **Microsoft Foundry** | 统一 AI 开发平台（原 Azure AI Studio），管理模型、数据、部署 |
| **Azure AI Services** | 预构建认知服务的集合（语言、视觉、语音、决策） |
| **Azure OpenAI Service** | 通过 Azure 访问 OpenAI 模型（GPT、DALL-E、Whisper 等） |
| **Resource & Endpoint** | 每个 AI 服务都需要创建资源，通过 Endpoint + Key 或 Entra ID 访问 |
| **REST API & SDK** | 两种调用方式：直接 HTTP 调用 REST API，或使用 Python/C# SDK |

#### 2.3 开发环境准备

```
前置要求：
├── Azure 订阅（可用免费试用）
├── Python 3.8+ 或 C#/.NET
├── Visual Studio Code
│   ├── Azure AI Foundry 扩展
│   └── Python / C# 扩展
├── Azure CLI
└── Azure AI Foundry SDK (pip install azure-ai-projects)
```

#### 2.4 认证与授权

- **API Key 方式**：简单但不推荐用于生产
- **Microsoft Entra ID（原 AAD）**：推荐的生产方式，使用 `DefaultAzureCredential`
- **RBAC 角色**：`Cognitive Services User`、`Cognitive Services Contributor`

---

### 3. Level 1 — 预构建 AI 服务（开箱即用）

> 🎯 **目标**：学会直接调用 Azure 预构建 AI 能力，无需训练模型

#### 3.1 文本分析 (Text Analytics)

**服务**：Azure Language Service（Foundry Tools 中的 Language 功能）

| 功能 | 说明 | 典型用途 |
|------|------|---------|
| **情感分析** (Sentiment Analysis) | 判断文本正面/负面/中性情绪 | 客户反馈分析、社交媒体监控 |
| **关键短语提取** (Key Phrase Extraction) | 提取文本中的关键词和短语 | 文档摘要、标签生成 |
| **命名实体识别** (NER) | 识别人名、地点、组织、日期等 | 信息提取、数据结构化 |
| **语言检测** (Language Detection) | 判断文本使用的语言 | 多语言系统路由 |
| **实体链接** (Entity Linking) | 将实体链接到维基百科条目 | 知识图谱、消歧 |
| **PII 检测** | 识别个人敏感信息 | 数据脱敏、合规 |

**调用模式**：
```
客户端 → REST API / SDK → Language Endpoint → 返回 JSON 结果
```

#### 3.2 翻译服务 (Translator Service)

- **文本翻译**：支持 100+ 语言的实时翻译
- **文档翻译**：批量翻译整个文档，保留格式
- **自定义翻译器**：使用自己的术语表训练定制翻译模型
- **音译** (Transliteration)：在不同文字系统间转换（如日文假名→罗马字）

#### 3.3 语音服务 (Speech Services)

| 功能 | 方向 | 说明 |
|------|------|------|
| **Speech-to-Text (STT)** | 语音 → 文本 | 实时/批量语音识别 |
| **Text-to-Speech (TTS)** | 文本 → 语音 | 自然语音合成，支持自定义语音 |
| **Speech Translation** | 语音 → 翻译文本 | 实时语音翻译 |
| **Speaker Recognition** | 语音 → 身份 | 说话人验证和识别 |

**关键概念**：
- **Speech Config**：配置订阅密钥和区域
- **Audio Config**：配置音频输入/输出（麦克风、文件、流）
- **SSML**：Speech Synthesis Markup Language，精细控制语音合成

#### 3.4 图像分析 (Image Analysis)

**服务**：Azure Vision Service

| 功能 | 说明 |
|------|------|
| **图像描述** | 自动生成图像的自然语言描述 |
| **标签提取** | 识别图像中的对象、场景、动作 |
| **对象检测** | 定位图像中对象的位置（边界框） |
| **智能裁剪** | 基于关注区域自动裁剪 |
| **人脸检测** | 检测人脸位置和属性 |
| **OCR（光学字符识别）** | 从图像中提取文本（见 3.5） |

#### 3.5 OCR — 读取图像中的文本

**两种 API**：
- **Image Analysis Read API**（简单场景，同步）
- **Document Intelligence**（复杂文档，异步，见 Level 3）

**流程**：
```
图像输入 → 预处理 → 文本检测 → 文字识别 → 返回结构化文本
                                              (行、单词、边界框)
```

支持：打印文本、手写文本、多语言混合

#### 3.6 人脸检测与识别 (Face Detection & Recognition)

**三层能力**：
1. **Face Detection**：检测人脸位置和属性（需审批）
2. **Face Verification**：1:1 比对（这两张照片是同一个人吗？）
3. **Face Identification**：1:N 识别（这个人是谁？）

> ⚠️ **负责任 AI 注意**：人脸识别功能受限访问政策约束，需要申请审批

#### 3.7 视频分析 (Video Indexer)

Azure Video Indexer 从视频中提取多维洞察：
- 人脸识别与跟踪
- OCR（视频中的文字）
- 语音转文本
- 主题/关键词提取
- 情感分析
- 场景分割
- 内容审核

#### 3.8 预构建 Document Intelligence 模型

| 模型 | 提取内容 |
|------|---------|
| **发票模型** | 发票号、日期、金额、供应商信息 |
| **收据模型** | 商家名、交易日期、明细、总额 |
| **身份证模型** | 姓名、出生日期、证件号 |
| **名片模型** | 姓名、职位、公司、联系方式 |
| **税务文档** | W-2、1098 等美国税务表格 |

---

### 4. Level 2 — 自定义 AI 模型（训练你自己的模型）

> 🎯 **目标**：在预构建能力不足时，训练适合你业务场景的自定义模型

#### 4.1 对话语言理解 (Conversational Language Understanding — CLU)

**核心概念**：

| 概念 | 说明 | 示例 |
|------|------|------|
| **Utterance（话语）** | 用户说的一句话 | "帮我预订明天去北京的机票" |
| **Intent（意图）** | 用户想做什么 | `BookFlight` |
| **Entity（实体）** | 话语中的关键参数 | 日期=明天, 目的地=北京 |

**训练流程**：
```
1. 定义 Intent 和 Entity
2. 标注训练数据（Utterance → Intent + Entity）
3. 训练模型
4. 评估（Precision / Recall / F1）
5. 部署模型
6. 应用集成
```

#### 4.2 问答服务 (Question Answering)

基于知识库的问答系统：
- **数据源**：FAQ 网页、Word/PDF 文档、手动添加 QA 对
- **多轮对话**：支持 follow-up prompts 实现引导式对话
- **精确回答**：从段落中精确提取答案片段
- **同义词**：配置 alterations 提升匹配率

#### 4.3 自定义文本分类 (Custom Text Classification)

| 类型 | 说明 |
|------|------|
| **单标签分类** | 每个文档只属于一个类别 |
| **多标签分类** | 每个文档可属于多个类别 |

**训练要求**：
- 至少 10 个文档/类别
- 推荐 200+ 个文档/类别
- 需要标注数据

#### 4.4 自定义命名实体识别 (Custom NER)

从非结构化文本中提取自定义实体类型：
- 定义你的实体类型（如 "产品名"、"合同编号"）
- 标注训练数据
- 训练 → 评估 → 部署
- 评估指标：Precision、Recall、F1-Score

#### 4.5 自定义视觉 — 图像分类 (Custom Vision - Classification)

```
训练集图像 → 上传到 Custom Vision → 选择分类类型 → 训练 → 评估 → 发布
                                     ├── 多类别（每图一个标签）
                                     └── 多标签（每图多个标签）
```

**最佳实践**：
- 每类至少 15 张图
- 包含不同角度、光照、背景
- 使用 Smart Labeler 半自动标注

#### 4.6 自定义视觉 — 对象检测 (Custom Vision - Object Detection)

与分类的区别：不仅识别**是什么**，还定位**在哪里**（边界框）

- 需要标注边界框（Bounding Box）
- 适用场景：工业质检、货架商品识别、安防监控

#### 4.7 自定义 Document Intelligence 模型

当预构建模型无法满足需求时：
- **自定义模板模型**：适用于固定格式表单
- **自定义神经模型**：适用于半结构化和非结构化文档
- **组合模型**：组合多个模型处理不同表单类型

---

### 5. Level 3 — 生成式 AI 基础 (Generative AI Fundamentals)

> 🎯 **目标**：掌握 LLM 模型选择、部署和基本应用开发

#### 5.1 模型目录与部署 (Model Catalog & Deployment)

Microsoft Foundry 的 Model Catalog 提供多种模型：

| 模型来源 | 示例 | 部署方式 |
|---------|------|---------|
| **Azure OpenAI** | GPT-4o, GPT-4, GPT-3.5-turbo | 标准/全局部署 |
| **Meta** | Llama 3 | Serverless API / Managed Compute |
| **Mistral** | Mistral Large, Mixtral | Serverless API |
| **Microsoft** | Phi-3, Phi-4 | Serverless API / Managed Compute |
| **Cohere** | Command R+ | Serverless API |

**部署类型**：
- **Serverless API (MaaS)**：按 Token 付费，无需管理基础设施
- **Managed Compute (MaaP)**：独占计算资源，适合高吞吐
- **标准部署**：Azure OpenAI 模型的默认方式

#### 5.2 Microsoft Foundry SDK

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

# 创建项目客户端
client = AIProjectClient(
    credential=DefaultAzureCredential(),
    endpoint="https://<your-foundry>.services.ai.azure.com"
)

# 调用聊天完成
response = client.inference.get_chat_completions(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is Azure AI?"}
    ]
)
```

**核心组件**：
- `AIProjectClient`：项目级客户端，管理资源和连接
- `ChatCompletionsClient`：聊天完成调用
- `EmbeddingsClient`：文本嵌入生成

#### 5.3 Prompt Flow

Prompt Flow 是一个**可视化的 LLM 应用编排工具**：

```
输入 → [LLM 节点] → [Python 节点] → [LLM 节点] → 输出
         │               │               │
     提示模板        数据处理/API       总结/格式化
```

**核心概念**：
- **Flow（流）**：DAG（有向无环图）形式的工作流
- **Node（节点）**：LLM 调用、Python 代码、工具调用
- **Connection**：与外部服务的连接配置
- **Variant**：同一节点的不同提示版本，用于 A/B 测试

**三种 Flow 类型**：
1. **Standard Flow**：通用 LLM 应用流
2. **Chat Flow**：对话式应用，内置聊天历史管理
3. **Evaluation Flow**：评估其他流的质量

#### 5.4 视觉增强的生成式 AI 应用

使用多模态模型（如 GPT-4o）处理图像：

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "Describe this image"},
            {"type": "image_url", "image_url": {"url": image_url}}
        ]
    }]
)
```

**应用场景**：图像描述、视觉问答、图表解读、文档理解

#### 5.5 语音增强的生成式 AI 应用

结合 Speech Service 和 LLM：

```
用户语音 → STT → 文本 → LLM → 响应文本 → TTS → 语音输出
```

- 实时语音对话 AI
- 多语言语音助手
- 会议摘要和分析

#### 5.6 图像生成 (Image Generation)

使用 DALL-E 模型生成图像：
- **文本到图像**：基于自然语言描述生成图像
- **图像编辑**：修改现有图像的特定区域
- **图像变体**：基于参考图像生成变体

---

### 6. Level 4 — 高级生成式 AI (Advanced Generative AI)

> 🎯 **目标**：掌握 RAG、Fine-tuning 和企业级知识检索

#### 6.1 RAG — 检索增强生成 (Retrieval Augmented Generation)

**为什么需要 RAG？**
- LLM 的训练数据有截止日期
- LLM 不了解你的私有数据
- 直接把所有数据塞进 Prompt 会超出 Token 限制

**RAG 架构**：
```
                    ┌──────────────┐
                    │  你的数据源    │
                    │ (文档/数据库)  │
                    └──────┬───────┘
                           │ 索引阶段
                           ▼
                    ┌──────────────┐
                    │ Azure AI     │
                    │ Search Index │
                    │ (向量+关键词)  │
                    └──────┬───────┘
                           │ 查询阶段
用户提问 ────────┐         │
                 ▼         ▼
          ┌──────────────────────┐
          │   检索相关文档片段     │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │  组合 Prompt:         │
          │  System + Context    │
          │  + User Question     │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │      LLM 生成回答     │
          │   (基于检索到的上下文)  │
          └──────────────────────┘
```

**关键步骤**：
1. **数据准备**：文档切片（Chunking）、嵌入生成
2. **索引创建**：Azure AI Search 创建向量索引
3. **检索配置**：混合搜索（关键词 + 向量 + 语义排序）
4. **Prompt 工程**：设计 System Prompt 引导模型使用检索上下文
5. **响应生成**：LLM 基于检索结果生成有根据的回答

**Azure 实现**：
- 使用 Microsoft Foundry 的 "Add your data" 功能
- 或通过 SDK 编程实现完整 RAG 管道

#### 6.2 Fine-tuning — 模型微调

**何时选择 Fine-tuning 而非 RAG？**

| 场景 | RAG | Fine-tuning |
|------|-----|-------------|
| 需要最新知识 | ✅ 首选 | ❌ 不适合 |
| 改变模型行为/风格 | ❌ 有限 | ✅ 首选 |
| 学习特定格式/模式 | ❌ | ✅ |
| 减少 Token 消耗 | ❌ | ✅ |
| 快速实施 | ✅ | ❌ 需要时间 |

**Fine-tuning 流程**：
```
1. 准备训练数据（JSONL 格式的对话样本）
2. 上传数据到 Foundry
3. 选择基础模型
4. 配置训练参数（epochs, learning rate, batch size）
5. 启动训练
6. 评估微调模型
7. 部署微调模型
```

#### 6.3 Azure AI Search — 知识挖掘

**三大能力**：

| 能力 | 说明 |
|------|------|
| **数据摄取** | 从 Blob Storage、SQL、Cosmos DB 等拉取数据 |
| **AI 增强** | 使用 Skillset（OCR、实体识别、翻译等）丰富数据 |
| **智能搜索** | 全文搜索 + 向量搜索 + 语义排序 |

**Skillset（技能集）— 内置技能**：
- OCR、图像分析、关键短语提取
- 实体识别、语言检测、翻译
- 文档拆分、文本合并
- **自定义技能**：调用你自己的 API

**索引增强管道**：
```
数据源 → Indexer → Skillset → 增强文档 → 索引
                    │
            ┌───────┴───────┐
            │  Built-in     │
            │  Skills       │
            │  ┌─────────┐  │
            │  │OCR      │  │
            │  │NER      │  │
            │  │KeyPhrase│  │
            │  │Translate │  │
            │  └─────────┘  │
            └───────────────┘
```

#### 6.4 Azure Content Understanding（多模态内容分析）

新一代信息提取服务，统一处理多种内容类型：
- **文档**：提取结构化信息
- **图像**：分析视觉内容
- **音频**：转录和分析
- **视频**：多模态综合分析

**与 Document Intelligence 的关系**：
- Content Understanding 是更新的、更统一的服务
- Document Intelligence 仍在支持，专注于表单和文档

---

### 7. Level 5 — AI Agent 开发 (AI Agent Development)

> 🎯 **目标**：构建能自主执行任务的智能代理

#### 7.1 AI Agent 核心概念

**什么是 AI Agent？**
```
传统 LLM App:  用户提问 → LLM 回答 → 结束
AI Agent:     用户给目标 → Agent 规划 → 调用工具 → 观察结果 
              → 再次规划 → 再次调用 → ... → 完成目标
```

**Agent = LLM + Tools + Memory + Planning**

| 组件 | 说明 |
|------|------|
| **LLM（大脑）** | 理解指令、推理、决策 |
| **Tools（工具）** | 搜索、代码执行、API 调用、文件操作 |
| **Memory（记忆）** | 短期（对话上下文）和长期（持久化存储） |
| **Planning（规划）** | 将复杂任务分解为步骤 |

#### 7.2 Microsoft Foundry Agent Service

通过 Azure Portal 或 VS Code 创建和管理 Agent：

```python
from azure.ai.projects import AIProjectClient
from azure.ai.agents import Agent

# 创建 Agent
agent = client.agents.create_agent(
    model="gpt-4o",
    name="my-assistant",
    instructions="You are a helpful data analyst.",
    tools=[
        {"type": "code_interpreter"},
        {"type": "file_search"}
    ]
)
```

**内置工具**：
- **Code Interpreter**：运行 Python 代码、数据分析、生成图表
- **File Search**：搜索上传的文件内容
- **Bing Grounding**：使用 Bing 搜索获取实时信息
- **Azure AI Search**：搜索企业知识库

#### 7.3 自定义工具集成 (Custom Tools)

当内置工具不够时，定义你自己的 Function：

```python
# 定义工具 schema
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Get current weather for a city",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string"}
            },
            "required": ["city"]
        }
    }
}]

# Agent 会在需要时调用此工具
# 你负责实现实际的函数逻辑并返回结果
```

#### 7.4 MCP 工具集成 (Model Context Protocol)

**MCP** 是一个开放标准协议，让 Agent 能动态发现和调用外部工具：

```
Agent ←→ MCP Client ←→ MCP Server ←→ 外部工具/服务
```

- **优势**：标准化的工具注册和调用协议
- **场景**：连接第三方服务、企业内部系统

#### 7.5 Foundry IQ — 知识增强 Agent

Foundry IQ 提供**共享知识平台**，多个 Agent 可以访问：

```
┌─────────────────────────────────┐
│         Foundry IQ              │
│  (企业知识共享平台)               │
│                                  │
│  ┌──────────┐  ┌──────────┐     │
│  │ Data     │  │ RAG      │     │
│  │ Assets   │  │ Pipeline │     │
│  └──────────┘  └──────────┘     │
└──────────┬──────────────────────┘
           │
    ┌──────┼──────┐
    ▼      ▼      ▼
 Agent A Agent B Agent C
```

- 数据优化提升检索质量
- 配置 Agent 指令确保引用源的一致性
- 支持多 Agent 共享同一知识库

#### 7.6 与 Microsoft 365 集成

将 Agent 发布到：
- **Microsoft Teams**：团队协作中直接使用
- **Microsoft 365 Copilot**：集成到 Copilot 生态
- **Work IQ**：访问 M365 中的工作数据（邮件、日历、文档）

---

### 8. Level 6 — 多 Agent 与高级编排 (Multi-Agent & Advanced Orchestration)

> 🎯 **目标**：构建多 Agent 协作系统和企业级 AI 工作流

#### 8.1 Microsoft Agent Framework (Semantic Kernel)

**Semantic Kernel** 是 Microsoft 的 AI 编排 SDK：

```python
import semantic_kernel as sk

kernel = sk.Kernel()

# 添加 AI 服务
kernel.add_service(AzureChatCompletion(
    deployment_name="gpt-4o",
    endpoint="https://...",
    api_key="..."
))

# 添加插件（工具）
kernel.add_plugin(TimePlugin(), "time")
kernel.add_plugin(MathPlugin(), "math")

# 创建 Agent
agent = ChatCompletionAgent(
    kernel=kernel,
    name="analyst",
    instructions="You are a data analyst..."
)
```

**核心概念**：
- **Kernel**：AI 编排的核心引擎
- **Plugin**：可重用的功能模块
- **Planner**：自动将目标分解为步骤
- **Memory**：向量存储和语义记忆

#### 8.2 多 Agent 编排 (Multi-Agent Orchestration)

**Agent Chat 模式**：

```
┌─────────────────────────────────────────────┐
│            Agent Group Chat                  │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Research │  │ Analyst  │  │ Writer   │  │
│  │ Agent    │→ │ Agent    │→ │ Agent    │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│       │              │              │        │
│  搜索信息        分析数据        撰写报告      │
│                                              │
│  协调策略：                                    │
│  - Sequential（顺序）                          │
│  - Round-Robin（轮询）                         │
│  - Selection（智能选择）                        │
│  - Termination（完成条件）                      │
└─────────────────────────────────────────────┘
```

**编排策略**：
| 策略 | 说明 | 适用场景 |
|------|------|---------|
| **Sequential** | 按固定顺序轮流 | 简单流水线 |
| **Round-Robin** | 循环轮流 | 平等参与 |
| **Selection Strategy** | 由 LLM 决定下一个发言者 | 灵活协作 |
| **Termination Strategy** | 定义完成条件 | 自动停止 |

#### 8.3 A2A 协议 (Agent-to-Agent)

A2A 是一个**跨平台的 Agent 间通信协议**：

```
Agent A ←→ A2A Protocol ←→ Agent B
(本地)                      (远程/不同平台)
```

**核心功能**：
- **Agent Discovery**：发现远程 Agent 的能力
- **Direct Communication**：Agent 间直接通信
- **Task Coordination**：协调跨 Agent 的任务执行

#### 8.4 Agent 驱动的工作流 (Agent-Driven Workflows)

使用 Microsoft Foundry 构建智能工作流：
- 编排多个 Agent 和组件
- 条件分支和循环
- 错误处理和重试
- 人工审核节点（Human-in-the-loop）

---

### 9. Level 7 — 负责任 AI 与生产化 (Responsible AI & Production)

> 🎯 **目标**：确保 AI 解决方案安全、可靠、负责任

#### 9.1 负责任 AI 原则 (Responsible AI Principles)

Microsoft 的 6 大 AI 原则：

| 原则 | 说明 |
|------|------|
| **公平性 (Fairness)** | AI 系统应公平对待所有人 |
| **可靠性与安全性 (Reliability & Safety)** | AI 应可靠运行，不造成伤害 |
| **隐私与安全 (Privacy & Security)** | 保护用户数据和隐私 |
| **包容性 (Inclusiveness)** | AI 应赋能每个人 |
| **透明性 (Transparency)** | AI 决策应可理解 |
| **问责制 (Accountability)** | 人类应对 AI 系统负责 |

#### 9.2 生成式 AI 的内容安全

**Azure AI Content Safety**：

```
用户输入 → 输入过滤器 → LLM → 输出过滤器 → 返回给用户
              │                     │
        ┌─────┴─────┐        ┌─────┴─────┐
        │ 检测:      │        │ 检测:      │
        │ - 仇恨     │        │ - 仇恨     │
        │ - 暴力     │        │ - 暴力     │
        │ - 性内容   │        │ - 性内容   │
        │ - 自残     │        │ - 自残     │
        │ - Jailbreak│        │ - 错误信息 │
        └───────────┘        └───────────┘
```

**四个严重级别**：Safe → Low → Medium → High

#### 9.3 评估生成式 AI 应用

**内置评估指标**：

| 指标 | 评估内容 |
|------|---------|
| **Groundedness（有据性）** | 回答是否基于提供的上下文 |
| **Relevance（相关性）** | 回答是否与问题相关 |
| **Coherence（连贯性）** | 回答是否逻辑连贯 |
| **Fluency（流畅性）** | 语言是否自然流畅 |
| **Similarity（相似性）** | 与参考答案的相似度 |
| **F1 Score** | 与参考答案的词汇重叠 |

**评估方式**：
- **手动评估**：人工打分
- **AI 辅助评估**：使用 LLM 作为评判者
- **自动化指标**：使用 Prompt Flow 的评估流

---

### 10. 知识地图总览 (Complete Knowledge Map)

```
Level 7: 负责任 AI 与生产化
  │  负责任 AI 原则 → 内容安全 → 评估指标
  │
Level 6: 多 Agent 与高级编排
  │  Semantic Kernel → 多 Agent Chat → A2A 协议 → Agent 工作流
  │
Level 5: AI Agent 开发
  │  Foundry Agent Service → 自定义工具 → MCP → Foundry IQ → M365 集成
  │
Level 4: 高级生成式 AI
  │  RAG → Fine-tuning → AI Search → Content Understanding
  │
Level 3: 生成式 AI 基础
  │  Model Catalog → Foundry SDK → Prompt Flow → 多模态应用
  │
Level 2: 自定义 AI 模型
  │  CLU → QnA → 文本分类 → 自定义 NER → Custom Vision → Doc Intelligence
  │
Level 1: 预构建 AI 服务
  │  文本分析 → 翻译 → 语音 → 图像分析 → OCR → 人脸 → 视频 → Doc Intelligence
  │
Level 0: 基础平台
    Azure AI 服务全景 → Microsoft Foundry → 认证授权 → 开发环境
```

---

## English Version

---

### 1. Overview

AI-102 is the official course for the Microsoft Azure AI Engineer Associate certification. It covers **5 learning paths with 40 modules**, spanning: generative AI app development, AI agent building, natural language processing, computer vision, and information extraction.

This article structures ALL AI-102 knowledge from **Level 0 (Foundation) to Level 7 (Advanced)**, building a complete knowledge map where each level builds on the previous one.

#### Course Overview — 5 Learning Paths

| # | Learning Path | Modules | Core Topics |
|---|--------------|---------|-------------|
| 1 | Develop Generative AI Apps in Azure | 8 | GenAI, RAG, Fine-tuning, Prompt Flow |
| 2 | Develop AI Agents on Azure | 9 | Agent Service, MCP, Multi-agent, A2A |
| 3 | Develop Natural Language Solutions | 10 | Text Analytics, CLU, Speech, Translation |
| 4 | Develop Computer Vision Solutions | 8 | Image Analysis, OCR, Face, Custom Vision |
| 5 | Develop AI Information Extraction Solutions | 5 | Document Intelligence, AI Search, Content Understanding |

---

### 2. Level 0 — Foundation & Setup

> 🎯 **Goal**: Understand the Azure AI ecosystem and set up your development environment

#### 2.1 Azure AI Service Landscape

```
┌─────────────────────────────────────────────────────┐
│                  Microsoft Foundry                   │
│  (Unified AI Development Platform, formerly AI Studio)│
│                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │ Model    │ │ Prompt   │ │ Agent    │             │
│  │ Catalog  │ │ Flow     │ │ Service  │             │
│  └──────────┘ └──────────┘ └──────────┘             │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │            Azure AI Services                  │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐│   │
│  │  │Language │ │Vision  │ │Speech  │ │Document││   │
│  │  │Service │ │Service │ │Service │ │Intelli.││   │
│  │  └────────┘ └────────┘ └────────┘ └────────┘│   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │          Azure OpenAI Service                 │   │
│  │   GPT-4o │ GPT-4 │ DALL-E │ Whisper          │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │          Azure AI Search                      │   │
│  │   Index │ Skillset │ Vector Search │ Semantic │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

#### 2.2 Key Foundation Concepts

| Concept | Description |
|---------|-------------|
| **Microsoft Foundry** | Unified AI dev platform (formerly Azure AI Studio) for managing models, data, deployments |
| **Azure AI Services** | Collection of pre-built cognitive services (Language, Vision, Speech, Decision) |
| **Azure OpenAI Service** | Access OpenAI models (GPT, DALL-E, Whisper) through Azure |
| **Resource & Endpoint** | Each AI service requires a resource, accessed via Endpoint + Key or Entra ID |
| **REST API & SDK** | Two invocation methods: direct HTTP REST calls, or Python/C# SDKs |

#### 2.3 Development Environment Setup

```
Prerequisites:
├── Azure subscription (free trial available)
├── Python 3.8+ or C#/.NET
├── Visual Studio Code
│   ├── Azure AI Foundry extension
│   └── Python / C# extension
├── Azure CLI
└── Azure AI Foundry SDK (pip install azure-ai-projects)
```

#### 2.4 Authentication & Authorization

- **API Key**: Simple but not recommended for production
- **Microsoft Entra ID (formerly AAD)**: Recommended for production using `DefaultAzureCredential`
- **RBAC Roles**: `Cognitive Services User`, `Cognitive Services Contributor`

---

### 3. Level 1 — Pre-built AI Services (Out-of-the-Box)

> 🎯 **Goal**: Learn to consume Azure's pre-built AI capabilities without training models

#### 3.1 Text Analytics

**Service**: Azure Language Service (Language in Foundry Tools)

| Capability | Description | Use Case |
|-----------|-------------|----------|
| **Sentiment Analysis** | Determine positive/negative/neutral sentiment | Customer feedback, social monitoring |
| **Key Phrase Extraction** | Extract keywords and phrases | Document summarization, tagging |
| **Named Entity Recognition** | Identify people, places, orgs, dates | Information extraction |
| **Language Detection** | Identify the language of text | Multi-language routing |
| **Entity Linking** | Link entities to Wikipedia entries | Knowledge graphs, disambiguation |
| **PII Detection** | Identify personally identifiable information | Data masking, compliance |

#### 3.2 Translation Service

- **Text Translation**: Real-time translation across 100+ languages
- **Document Translation**: Batch translate entire documents with format preservation
- **Custom Translator**: Train custom translation models with your terminology
- **Transliteration**: Convert between writing systems

#### 3.3 Speech Services

| Capability | Direction | Description |
|-----------|-----------|-------------|
| **Speech-to-Text** | Audio → Text | Real-time/batch speech recognition |
| **Text-to-Speech** | Text → Audio | Natural voice synthesis with custom voices |
| **Speech Translation** | Audio → Translated Text | Real-time spoken language translation |
| **Speaker Recognition** | Audio → Identity | Speaker verification and identification |

**Key Concepts**: Speech Config, Audio Config, SSML (Speech Synthesis Markup Language)

#### 3.4 Image Analysis

**Service**: Azure Vision Service

| Capability | Description |
|-----------|-------------|
| **Image Description** | Auto-generate natural language descriptions |
| **Tag Extraction** | Identify objects, scenes, actions |
| **Object Detection** | Locate objects with bounding boxes |
| **Smart Cropping** | Auto-crop based on regions of interest |
| **Face Detection** | Detect face locations and attributes |
| **OCR** | Extract text from images |

#### 3.5 OCR — Reading Text in Images

- **Image Analysis Read API**: Simple scenarios, synchronous
- **Document Intelligence**: Complex documents, asynchronous (see Level 4)
- Supports: printed text, handwriting, multi-language

#### 3.6 Face Detection & Recognition

Three capability tiers:
1. **Face Detection**: Detect face location and attributes (approval required)
2. **Face Verification**: 1:1 comparison — are these the same person?
3. **Face Identification**: 1:N recognition — who is this person?

> ⚠️ Face recognition features are subject to Limited Access policy

#### 3.7 Video Analysis (Video Indexer)

Extracts multi-dimensional insights: face tracking, OCR, speech-to-text, topics, sentiment, scene segmentation, content moderation.

#### 3.8 Prebuilt Document Intelligence Models

| Model | Extracts |
|-------|----------|
| **Invoice** | Invoice number, dates, amounts, vendor info |
| **Receipt** | Merchant, transaction date, line items, totals |
| **ID Document** | Name, DOB, document number |
| **Business Card** | Name, title, company, contact info |
| **Tax Documents** | W-2, 1098 US tax form fields |

---

### 4. Level 2 — Custom AI Models (Train Your Own)

> 🎯 **Goal**: Train custom models when pre-built capabilities aren't sufficient

#### 4.1 Conversational Language Understanding (CLU)

| Concept | Description | Example |
|---------|-------------|---------|
| **Utterance** | What the user says | "Book me a flight to Tokyo tomorrow" |
| **Intent** | What the user wants to do | `BookFlight` |
| **Entity** | Key parameters in the utterance | destination=Tokyo, date=tomorrow |

**Training workflow**: Define Intents/Entities → Label training data → Train → Evaluate (Precision/Recall/F1) → Deploy → Integrate

#### 4.2 Question Answering

Knowledge-base-powered Q&A system:
- **Data Sources**: FAQ pages, Word/PDF docs, manual QA pairs
- **Multi-turn conversations**: Follow-up prompts for guided dialogue
- **Precise answers**: Extract exact answer spans from passages
- **Synonyms**: Configure alterations to improve matching

#### 4.3 Custom Text Classification

| Type | Description |
|------|-------------|
| **Single-label** | Each document belongs to exactly one category |
| **Multi-label** | Each document can belong to multiple categories |

Requirements: At least 10 documents/category; recommended 200+

#### 4.4 Custom Named Entity Recognition (NER)

Extract custom entity types from unstructured text:
- Define your entity types (e.g., "ProductName", "ContractNumber")
- Label training data → Train → Evaluate → Deploy
- Metrics: Precision, Recall, F1-Score

#### 4.5 Custom Vision — Image Classification

```
Training images → Upload → Choose classification type → Train → Evaluate → Publish
                            ├── Multi-class (one label per image)
                            └── Multi-label (multiple labels per image)
```

Best practices: At least 15 images/class, diverse angles/lighting/backgrounds

#### 4.6 Custom Vision — Object Detection

Locates objects with bounding boxes — not just **what** but **where**.
Use cases: industrial inspection, retail shelf monitoring, security

#### 4.7 Custom Document Intelligence Models

- **Custom template model**: Fixed-format forms
- **Custom neural model**: Semi-structured and unstructured documents
- **Composed model**: Combine multiple models for different form types

---

### 5. Level 3 — Generative AI Fundamentals

> 🎯 **Goal**: Master LLM model selection, deployment, and basic app development

#### 5.1 Model Catalog & Deployment

| Model Provider | Examples | Deployment |
|---------------|----------|------------|
| **Azure OpenAI** | GPT-4o, GPT-4, GPT-3.5-turbo | Standard/Global |
| **Meta** | Llama 3 | Serverless API / Managed Compute |
| **Mistral** | Mistral Large, Mixtral | Serverless API |
| **Microsoft** | Phi-3, Phi-4 | Serverless API / Managed Compute |
| **Cohere** | Command R+ | Serverless API |

**Deployment Types**:
- **Serverless API (MaaS)**: Pay-per-token, no infrastructure management
- **Managed Compute (MaaP)**: Dedicated compute, high throughput
- **Standard Deployment**: Default for Azure OpenAI models

#### 5.2 Microsoft Foundry SDK

Core components:
- `AIProjectClient`: Project-level client for managing resources
- `ChatCompletionsClient`: Chat completion calls
- `EmbeddingsClient`: Text embedding generation

#### 5.3 Prompt Flow

Visual LLM application orchestration tool:
- **Flow**: DAG-based workflow
- **Nodes**: LLM calls, Python code, tool invocations
- **Connections**: External service configurations
- **Variants**: Different prompt versions for A/B testing
- **Three Flow Types**: Standard, Chat, Evaluation

#### 5.4 Multimodal Generative AI

- **Vision-enabled apps**: GPT-4o processing images (descriptions, visual Q&A, chart interpretation)
- **Speech-enabled apps**: STT → LLM → TTS pipeline for voice assistants
- **Image generation**: DALL-E for text-to-image, editing, and variations

---

### 6. Level 4 — Advanced Generative AI

> 🎯 **Goal**: Master RAG, Fine-tuning, and enterprise knowledge retrieval

#### 6.1 RAG — Retrieval Augmented Generation

**Why RAG?** LLMs have training cutoff dates, don't know your private data, and prompts have token limits.

**RAG Architecture**:
```
Your Data → Chunking → Embedding → Azure AI Search Index
                                         ↓
User Question → Retrieve relevant chunks → Combine with prompt → LLM → Grounded answer
```

**Key Steps**:
1. Data preparation: Document chunking & embedding generation
2. Index creation: Azure AI Search vector index
3. Retrieval: Hybrid search (keyword + vector + semantic ranking)
4. Prompt engineering: System prompt guiding model to use retrieved context
5. Response generation: LLM generates grounded answers

#### 6.2 Fine-tuning

| Scenario | RAG | Fine-tuning |
|----------|-----|-------------|
| Need latest knowledge | ✅ Preferred | ❌ Not suitable |
| Change model behavior/style | ❌ Limited | ✅ Preferred |
| Learn specific formats | ❌ | ✅ |
| Reduce token consumption | ❌ | ✅ |
| Quick implementation | ✅ | ❌ Takes time |

**Process**: Prepare JSONL training data → Upload → Select base model → Configure hyperparameters → Train → Evaluate → Deploy

#### 6.3 Azure AI Search — Knowledge Mining

**Three capabilities**: Data Ingestion → AI Enrichment (Skillsets) → Intelligent Search

**Built-in Skills**: OCR, image analysis, key phrase extraction, entity recognition, language detection, translation, document cracking, text merge, **custom skills** (call your own API)

#### 6.4 Azure Content Understanding

Next-generation multimodal content analysis:
- Documents, images, audio, video — unified processing
- Relationship to Document Intelligence: Content Understanding is newer and more unified

---

### 7. Level 5 — AI Agent Development

> 🎯 **Goal**: Build autonomous agents that can plan and execute tasks

#### 7.1 Core Agent Concepts

**Agent = LLM + Tools + Memory + Planning**

| Component | Purpose |
|-----------|---------|
| **LLM (Brain)** | Understanding, reasoning, decision-making |
| **Tools** | Search, code execution, API calls, file operations |
| **Memory** | Short-term (conversation) and long-term (persistent) |
| **Planning** | Decompose complex tasks into steps |

#### 7.2 Microsoft Foundry Agent Service

Built-in tools:
- **Code Interpreter**: Run Python, analyze data, generate charts
- **File Search**: Search uploaded file contents
- **Bing Grounding**: Real-time web information
- **Azure AI Search**: Enterprise knowledge base search

#### 7.3 Custom Tool Integration

Define function schemas for your own tools; the agent calls them when needed and you implement the actual logic.

#### 7.4 MCP Tools (Model Context Protocol)

Open standard protocol for dynamic tool discovery and invocation:
```
Agent ←→ MCP Client ←→ MCP Server ←→ External Tools/Services
```

#### 7.5 Foundry IQ — Knowledge-Enhanced Agents

Shared knowledge platform that multiple agents can access:
- Data optimization for retrieval quality
- Agent instruction configuration for consistent, cited responses

#### 7.6 Microsoft 365 Integration

Publish agents to Teams, M365 Copilot, and access workplace data through Work IQ.

---

### 8. Level 6 — Multi-Agent & Advanced Orchestration

> 🎯 **Goal**: Build multi-agent collaborative systems and enterprise AI workflows

#### 8.1 Microsoft Agent Framework (Semantic Kernel)

Core concepts:
- **Kernel**: AI orchestration engine
- **Plugins**: Reusable capability modules
- **Planner**: Auto-decompose goals into steps
- **Memory**: Vector storage and semantic memory

#### 8.2 Multi-Agent Orchestration

**Orchestration Strategies**:

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **Sequential** | Fixed order rotation | Simple pipelines |
| **Round-Robin** | Cyclic rotation | Equal participation |
| **Selection Strategy** | LLM decides next speaker | Flexible collaboration |
| **Termination Strategy** | Define completion conditions | Auto-stop |

#### 8.3 A2A Protocol (Agent-to-Agent)

Cross-platform agent communication protocol:
- **Agent Discovery**: Discover remote agent capabilities
- **Direct Communication**: Agent-to-agent messaging
- **Task Coordination**: Cross-agent task orchestration

#### 8.4 Agent-Driven Workflows

Build intelligent workflows with Microsoft Foundry:
- Orchestrate multiple agents and components
- Conditional branching and loops
- Error handling and retries
- Human-in-the-loop review nodes

---

### 9. Level 7 — Responsible AI & Production

> 🎯 **Goal**: Ensure AI solutions are safe, reliable, and responsible

#### 9.1 Responsible AI Principles

| Principle | Description |
|-----------|-------------|
| **Fairness** | AI should treat all people fairly |
| **Reliability & Safety** | AI should perform reliably without causing harm |
| **Privacy & Security** | Protect user data and privacy |
| **Inclusiveness** | AI should empower everyone |
| **Transparency** | AI decisions should be understandable |
| **Accountability** | Humans should be accountable for AI systems |

#### 9.2 Content Safety for Generative AI

**Azure AI Content Safety** provides input/output filtering:
```
User Input → Input Filter → LLM → Output Filter → Response
               │                      │
          Detect:                 Detect:
          - Hate                 - Hate
          - Violence             - Violence
          - Sexual               - Sexual
          - Self-harm            - Self-harm
          - Jailbreak            - Misinformation
```

Four severity levels: Safe → Low → Medium → High

#### 9.3 Evaluating Generative AI Applications

| Metric | Evaluates |
|--------|-----------|
| **Groundedness** | Is the answer based on provided context? |
| **Relevance** | Is the answer relevant to the question? |
| **Coherence** | Is the answer logically coherent? |
| **Fluency** | Is the language natural and fluent? |
| **Similarity** | How similar to the reference answer? |
| **F1 Score** | Token overlap with reference answer |

Evaluation methods: Manual scoring, AI-assisted (LLM-as-judge), Automated metrics via Prompt Flow evaluation flows

---

### 10. Complete Knowledge Map

```
Level 7: Responsible AI & Production
  │  Responsible AI Principles → Content Safety → Evaluation Metrics
  │
Level 6: Multi-Agent & Advanced Orchestration
  │  Semantic Kernel → Multi-Agent Chat → A2A Protocol → Agent Workflows
  │
Level 5: AI Agent Development
  │  Foundry Agent Service → Custom Tools → MCP → Foundry IQ → M365 Integration
  │
Level 4: Advanced Generative AI
  │  RAG → Fine-tuning → AI Search → Content Understanding
  │
Level 3: Generative AI Fundamentals
  │  Model Catalog → Foundry SDK → Prompt Flow → Multimodal Apps
  │
Level 2: Custom AI Models
  │  CLU → Q&A → Text Classification → Custom NER → Custom Vision → Doc Intelligence
  │
Level 1: Pre-built AI Services
  │  Text Analytics → Translation → Speech → Image Analysis → OCR → Face → Video → Doc Intelligence
  │
Level 0: Foundation
    Azure AI Landscape → Microsoft Foundry → Auth & Security → Dev Environment
```

---

### 8. 参考资料 (References)

- [AI-102 Course Page](https://learn.microsoft.com/en-us/training/courses/ai-102t00) — Official course landing page
- [Develop Generative AI Apps Learning Path](https://learn.microsoft.com/en-us/training/paths/create-custom-copilots-ai-studio/) — Path 1: GenAI development
- [Develop AI Agents on Azure Learning Path](https://learn.microsoft.com/en-us/training/paths/develop-ai-agents-azure/) — Path 2: AI Agent development
- [Develop Natural Language Solutions Learning Path](https://learn.microsoft.com/en-us/training/paths/develop-language-solutions-azure-ai/) — Path 3: NLP solutions
- [Develop Computer Vision Solutions Learning Path](https://learn.microsoft.com/en-us/training/paths/create-computer-vision-solutions-azure-ai/) — Path 4: Computer Vision
- [Develop AI Information Extraction Solutions Learning Path](https://learn.microsoft.com/en-us/training/paths/ai-extract-information/) — Path 5: Information extraction
- [Azure AI Engineer Associate Certification](https://learn.microsoft.com/en-us/credentials/certifications/azure-ai-engineer/) — Certification details
