---
layout: post
title: "Deep Dive: AI-102 题库深度分析 — 715 道题分类统计与备考策略"
date: 2026-03-15
categories: [Knowledge, AI]
tags: [ai-102, exam-analysis, question-bank, azure-ai-engineer, certification, 题库分析]
type: "deep-dive"
---

# Deep Dive: AI-102 题库深度分析 — 715 道题分类统计与备考策略

**Topic:** AI-102 Exam Question Bank Analysis  
**Category:** Certification / Exam Preparation  
**Data Source:** 2 PDF dumps (333题带讨论 + Microsoft-AI-102)  
**Total Questions Analyzed:** 715  
**Last Updated:** 2026-03-15  

---

## 中文版 (Chinese Version)

---

### 1. 数据概览

我们对两份 AI-102 考试题库 PDF 进行了完整提取和自动分类：

| PDF 文件 | 页数 | 题目数 | 内容特点 |
|---------|------|--------|---------|
| **AI-102 333题带讨论** | 728 页 | 334 题 | 含社区讨论和投票，18 个 Topic |
| **Microsoft-AI-102** | 549 页 | 381 题 | 标准题库，含 2 个 Case Study + 364 道独立题 |
| **合计** | 1,277 页 | **715 题** | — |

**MS-AI102 PDF 的 Topic 结构**：

| Topic | 内容 | 题数 |
|-------|------|------|
| Topic 1 | Wide World Importers Case Study | 7 |
| Topic 2 | Contoso, Ltd. Case Study | 10 |
| Topic 3 | Misc. Questions | 364 |

---

### 2. 按考试域分类统计

我们将 715 道题按照官方 AI-102 考试的 **6 大考试域**进行了自动分类：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AI-102 题库分布 vs 官方考试权重                    │
│                                                                      │
│  考试域                      题数   题库占比  官方权重   差异          │
│  ─────────────────────────────────────────────────────────────────   │
│  D5-NLP (自然语言处理)        238    33.3%    15-20%   ⚠️ 偏高       │
│  D1-Plan&Manage (规划管理)    189    26.4%    20-25%   ✅ 匹配       │
│  D2-GenerativeAI (生成式AI)   107    15.0%    15-20%   ✅ 匹配       │
│  D6-KnowledgeMining (知识挖掘) 75    10.5%    15-20%   ⚠️ 偏低       │
│  D4-ComputerVision (计算机视觉) 74    10.3%    10-15%   ✅ 匹配       │
│  D3-Agentic (Agent 代理)        8     1.1%     5-10%   🔴 严重偏低   │
│  未分类                         24     3.4%      —      —            │
└─────────────────────────────────────────────────────────────────────┘
```

**可视化分布**：

```
D5-NLP            ████████████████████████████████████████████████  238 (33.3%)
D1-Plan&Manage    █████████████████████████████████████             189 (26.4%)
D2-GenerativeAI   ████████████████████                              107 (15.0%)
D6-KnowledgeMining ██████████████                                    75 (10.5%)
D4-ComputerVision  ██████████████                                    74 (10.3%)
D3-Agentic         █                                                  8 ( 1.1%)
```

---

### 3. 子主题详细拆解

#### 3.1 Domain 5: NLP — 自然语言处理 (238 题 / 33.3%)

| 子主题 | 题数 | 占比 | 重要度 |
|--------|------|------|--------|
| **CLU/LUIS (Intent & Entity)** | 103 | 14.4% | 🔴🔴🔴 最高频考点 |
| Bot / Dispatch | 41 | 5.7% | 🔴🔴 |
| Question Answering | 29 | 4.1% | 🔴 |
| Speech Services | 19 | 2.7% | 🟡 |
| Translation | 16 | 2.2% | 🟡 |
| Text Analytics | 15 | 2.1% | 🟡 |
| NLP General | 14 | 2.0% | 🟡 |
| Custom Text Classification | 1 | 0.1% | 🟢 |

> 💡 **CLU/LUIS 是整个考试的最大单一考点**（103 题），必须精通 Intent、Entity、Utterance 的定义、训练和评估。注意 LUIS 已迁移到 CLU，但题库中旧 LUIS 题仍大量存在。

#### 3.2 Domain 1: Plan & Manage — 规划管理 (189 题 / 26.4%)

| 子主题 | 题数 | 占比 | 重要度 |
|--------|------|------|--------|
| Planning General | 45 | 6.3% | 🔴🔴 |
| **Resource Provisioning** | 40 | 5.6% | 🔴🔴 |
| **Responsible AI & Content Safety** | 33 | 4.6% | 🔴🔴 |
| Security & Authentication | 22 | 3.1% | 🔴 |
| Monitoring & Diagnostics | 18 | 2.5% | 🟡 |
| Cost & Scaling | 16 | 2.2% | 🟡 |
| Container Deployment | 15 | 2.1% | 🟡 |

> 💡 **"选择正确的服务"** 是这个域最常见的题型。你需要熟记每个 Azure AI 服务的适用场景和边界。**负责任 AI** 相关题（Content Safety、Content Filter、Blocklist）出题频率在上升。

#### 3.3 Domain 2: Generative AI — 生成式 AI (107 题 / 15.0%)

| 子主题 | 题数 | 占比 | 重要度 |
|--------|------|------|--------|
| **RAG & Grounding** | 55 | 7.7% | 🔴🔴🔴 GenAI 最重要考点 |
| GenAI General | 30 | 4.2% | 🔴 |
| Parameter Tuning | 9 | 1.3% | 🟡 |
| Embeddings | 5 | 0.7% | 🟡 |
| Fine-tuning | 4 | 0.6% | 🟡 |
| Image Generation (DALL-E) | 2 | 0.3% | 🟢 |
| Content Safety (GenAI) | 2 | 0.3% | 🟢 |

> 💡 **RAG 占 GenAI 题的一半以上**（55/107），必须掌握完整流程：Chunking → Embedding → Index → Retrieval → Grounding → Generation。参数调优（temperature/top_p/max_tokens）也是常考点。

#### 3.4 Domain 4: Computer Vision — 计算机视觉 (74 题 / 10.3%)

| 子主题 | 题数 | 占比 | 重要度 |
|--------|------|------|--------|
| Vision General | 28 | 3.9% | 🔴 |
| **Custom Vision (Classification)** | 18 | 2.5% | 🔴 |
| Face API | 12 | 1.7% | 🟡 |
| Video Analysis | 7 | 1.0% | 🟡 |
| OCR / Read Text | 7 | 1.0% | 🟡 |
| Object Detection | 2 | 0.3% | 🟢 |

> 💡 **Custom Vision（图像分类 vs 对象检测的区别）** 和 **Face API（PersonGroup 训练流程）** 是重点。需要知道 Image Analysis API 返回的 JSON 结构。

#### 3.5 Domain 6: Knowledge Mining — 知识挖掘 (75 题 / 10.5%)

| 子主题 | 题数 | 占比 | 重要度 |
|--------|------|------|--------|
| **Azure AI Search** | 39 | 5.5% | 🔴🔴 |
| Info Extraction General | 22 | 3.1% | 🔴 |
| Document Intelligence | 13 | 1.8% | 🟡 |
| Content Understanding | 1 | 0.1% | 🟢 |

> 💡 **AI Search 的 Indexer + Skillset + Index 三件套**是这个域的核心。需要掌握数据源配置、内置/自定义 Skill、查询语法（Lucene/简单查询）、Knowledge Store 投影。

#### 3.6 Domain 3: Agentic — Agent 代理 (8 题 / 1.1%)

| 子主题 | 题数 | 占比 | 重要度 |
|--------|------|------|--------|
| Agent Development | 8 | 1.1% | ⚠️ 题库少但趋势增加 |

> ⚠️ **重要警告**：Agentic 是 2025 年底新增的考试域（官方权重 5-10%），但题库还没跟上（仅 8 题）。**实际考试中 Agent 题目会比题库覆盖的更多**，建议额外学习 Foundry Agent Service 和 Semantic Kernel。

---

### 4. 题型分布

```
┌────────────────────────────────────────────────────────┐
│                    题型分布 (715 题)                     │
├────────────────────┬──────┬───────┬────────────────────┤
│ 题型               │ 题数 │  占比 │ 可视化              │
├────────────────────┼──────┼───────┼────────────────────┤
│ Multiple Choice    │  328 │ 45.9% │ ██████████████████ │
│ Other (Mixed)      │  232 │ 32.4% │ ████████████       │
│ Hotspot            │   89 │ 12.4% │ █████              │
│ Drag-Drop          │   40 │  5.6% │ ██                 │
│ Yes/No             │   26 │  3.6% │ █                  │
└────────────────────┴──────┴───────┴────────────────────┘
```

**各域题型特点**：

| 域 | 最常见题型 | 特殊题型 |
|----|-----------|---------|
| D1-Plan&Manage | 多选 (46%) | Hotspot 较多 (14%) — 选择正确的配置项 |
| D2-GenerativeAI | 多选 (37%) | Drag-Drop 最多 (19%) — 排列 RAG 步骤 |
| D4-ComputerVision | 多选 (53%) | Hotspot (15%) — 选择 API 参数 |
| D5-NLP | 多选 (44%) | Other 最多 (39%) — 代码填空题 |
| D6-KnowledgeMining | 多选 (49%) | Yes/No (7%) — 判断 Skillset 配置正确性 |

---

### 5. 高频考点 TOP 10

```
┌────┬──────────────────────────────────────────┬──────┬───────┬─────────┐
│ #  │ 考点                                      │ 题数 │  占比  │ 域      │
├────┼──────────────────────────────────────────┼──────┼───────┼─────────┤
│  1 │ CLU/LUIS (Intent & Entity)               │  103 │ 14.4% │ D5-NLP  │
│  2 │ RAG & Grounding                          │   55 │  7.7% │ D2-GenAI│
│  3 │ Planning General                         │   45 │  6.3% │ D1-Plan │
│  4 │ Bot / Dispatch                           │   41 │  5.7% │ D5-NLP  │
│  5 │ Resource Provisioning                    │   40 │  5.6% │ D1-Plan │
│  6 │ Azure AI Search                          │   39 │  5.5% │ D6-KM   │
│  7 │ Responsible AI & Content Safety          │   33 │  4.6% │ D1-Plan │
│  8 │ GenAI General                            │   30 │  4.2% │ D2-GenAI│
│  9 │ Question Answering                       │   29 │  4.1% │ D5-NLP  │
│ 10 │ Vision General                           │   28 │  3.9% │ D4-CV   │
└────┴──────────────────────────────────────────┴──────┴───────┴─────────┘
```

> 🎯 **TOP 10 考点合计覆盖 443 题（62%）**。集中精力掌握这些就能覆盖大部分考试内容。

---

### 6. 关键发现与备考建议

#### 6.1 关键发现

```
┌────────────────────────────────────────────────────────────────────┐
│  发现 1: NLP 题量远超官方权重                                       │
│  ├── 题库占 33.3%，官方仅 15-20%                                   │
│  ├── CLU/LUIS 单独占 103 题（14.4%），是最大考点                     │
│  └── 原因：题库积累时间长，LUIS→CLU 迁移期新旧题并存                  │
│                                                                     │
│  发现 2: RAG 是 GenAI 的绝对核心                                    │
│  ├── RAG 相关 55 题，占 GenAI 域的 51%                              │
│  └── 检索增强生成的完整流程必须倒背如流                               │
│                                                                     │
│  发现 3: Agentic 题库严重不足                                       │
│  ├── 仅 8 题（1.1%），官方权重 5-10%                                │
│  ├── 2025 年底新增领域，题库尚未跟上                                 │
│  └── ⚠️ 实际考试中 Agent 题会比题库多，需额外准备                     │
│                                                                     │
│  发现 4: 两份 PDF 有显著重叠                                        │
│  ├── 分布模式高度一致（NLP > Plan > GenAI > CV ≈ KM）               │
│  └── 建议以 MS-AI102 (381题) 为主，333题版参考社区讨论               │
│                                                                     │
│  发现 5: Content Safety 出题趋势上升                                │
│  ├── 社区讨论中多人反馈考试出现大量 Content Safety 题                 │
│  └── 建议重点学习 Azure AI Content Safety 服务                       │
└────────────────────────────────────────────────────────────────────┘
```

#### 6.2 备考优先级

```
🔴 P0 - 必须精通 (合计 270 题 / 38%):
   ├── CLU/LUIS: Intent, Entity, Utterance         (103 题)
   ├── RAG & Grounding: 检索增强生成               ( 55 题)
   ├── Resource Provisioning: 服务选择与部署         ( 40 题)
   ├── Responsible AI: 内容安全、伦理               ( 33 题)
   └── Azure AI Search: Indexer/Skillset/Index     ( 39 题)

🟡 P1 - 重点掌握 (合计 170 题 / 24%):
   ├── Bot / Dispatch: 聊天机器人框架               ( 41 题)
   ├── GenAI General: OpenAI 模型部署/调用          ( 30 题)
   ├── Question Answering: 问答服务                 ( 29 题)
   ├── Vision General: 图像分析 API                 ( 28 题)
   ├── Security: 认证/密钥/网络                     ( 22 题)
   └── Speech Services: STT/TTS/SSML              ( 19 题)

🟢 P2 - 基本了解 (合计 99 题 / 14%):
   ├── Custom Vision: 图像分类与对象检测             ( 18 题)
   ├── Monitoring: 监控诊断                         ( 18 题)
   ├── Cost & Scaling: 成本与扩展                   ( 16 题)
   ├── Translation: 翻译服务                        ( 16 题)
   ├── Container Deployment: 容器部署               ( 15 题)
   ├── Text Analytics: 文本分析                     ( 15 题)
   └── Document Intelligence: 文档智能              ( 13 题)

⚠️ 补充 - 题库不足但考试会考:
   └── Agent Development: Foundry Agent Service    (  8 题, 趋势↑)
```

---

### 7. 两份 PDF 对比

| 维度 | 333题带讨论 | MS-AI102 |
|------|-----------|----------|
| **题数** | 334 | 381 |
| **页数** | 728 | 549 |
| **结构** | 18 Topics | 3 Topics (2 Case Study + Misc) |
| **特色** | 含社区讨论投票 | 标准格式，有详细解释 |
| **Case Study** | 分散在各 Topic | Topic 1: Wide World Importers, Topic 2: Contoso |
| **NLP 占比** | 35.9% | 31.0% |
| **Plan&Manage 占比** | 26.9% | 26.0% |
| **GenAI 占比** | 15.9% | 14.2% |
| **CV 占比** | 9.0% | 11.5% |
| **KM 占比** | 9.3% | 11.5% |
| **Agent 占比** | 0.3% | 1.8% |

**建议使用方式**：
- **主要刷题**：MS-AI102 (381题) — 格式规范，解释详细
- **争议题参考**：333题版 — 社区讨论可帮助判断正确答案
- **Case Study 重点**：Wide World Importers + Contoso 两个场景必须熟悉

---

### 8. 考试实战建议

基于 715 道题的分析，以下是针对性的考试策略：

#### 8.1 每个域的应对策略

| 域 | 题量预估 (60题考试) | 应对策略 |
|----|-------------------|---------|
| **D1 Plan&Manage** | 12-15 题 | 记住服务选择决策树 + Responsible AI 4 类内容 4 级严重度 |
| **D2 GenAI** | 9-12 题 | 掌握 RAG 完整流程 + temperature/top_p 参数含义 |
| **D3 Agentic** | 3-6 题 | 了解 Agent 概念 + Foundry Agent Service 基础操作 |
| **D4 CV** | 6-9 题 | 区分 Classification vs Detection + Face API PersonGroup |
| **D5 NLP** | 9-12 题 | CLU 三要素 (Intent/Entity/Utterance) + Q&A 知识库流程 |
| **D6 KM** | 9-12 题 | AI Search 三件套 (Indexer/Skillset/Index) + 查询语法 |

#### 8.2 必须记住的关键区分

```
📌 CLU vs QnA Maker:
   CLU = 理解用户意图 (Intent) 和提取参数 (Entity)
   QnA = 从知识库匹配最佳答案

📌 Custom Vision Classification vs Object Detection:
   Classification = 是什么 (整图一个标签)
   Detection = 是什么 + 在哪里 (边界框)

📌 RAG vs Fine-tuning:
   RAG = 运行时注入知识 (最新数据, 快速实施)
   Fine-tuning = 训练时改变行为 (特定风格/格式)

📌 API Key vs Managed Identity:
   API Key = 简单但不安全
   Managed Identity = 推荐生产方式，无需管理密钥

📌 Serverless API vs Managed Compute:
   Serverless = 按 Token 付费，无需管理基础设施
   Managed Compute = 独占资源，适合高吞吐

📌 AI Search 内置 Skill vs 自定义 Skill:
   内置 = OCR, NER, Key Phrase, Language Detection
   自定义 = 调用你自己的 REST API (Azure Function)
```

---

## English Version

---

### 1. Data Overview

| PDF File | Pages | Questions | Features |
|---------|-------|-----------|----------|
| **AI-102 333 Questions with Discussion** | 728 | 334 | Community discussions and votes, 18 Topics |
| **Microsoft-AI-102** | 549 | 381 | Standard format, 2 Case Studies + 364 standalone |
| **Total** | 1,277 | **715** | — |

---

### 2. Classification by Exam Domain

| Domain | Questions | Bank % | Official Weight | Gap Analysis |
|--------|-----------|--------|----------------|-------------|
| D5-NLP | 238 | 33.3% | 15-20% | ⚠️ Over-represented |
| D1-Plan & Manage | 189 | 26.4% | 20-25% | ✅ Matched |
| D2-Generative AI | 107 | 15.0% | 15-20% | ✅ Matched |
| D6-Knowledge Mining | 75 | 10.5% | 15-20% | ⚠️ Under-represented |
| D4-Computer Vision | 74 | 10.3% | 10-15% | ✅ Matched |
| D3-Agentic | 8 | 1.1% | 5-10% | 🔴 Severely under-represented |

---

### 3. Top 10 High-Frequency Topics

| # | Topic | Count | % | Domain |
|---|-------|-------|---|--------|
| 1 | **CLU/LUIS (Intent & Entity)** | 103 | 14.4% | D5-NLP |
| 2 | **RAG & Grounding** | 55 | 7.7% | D2-GenAI |
| 3 | Planning General | 45 | 6.3% | D1-Plan |
| 4 | Bot / Dispatch | 41 | 5.7% | D5-NLP |
| 5 | Resource Provisioning | 40 | 5.6% | D1-Plan |
| 6 | Azure AI Search | 39 | 5.5% | D6-KM |
| 7 | Responsible AI & Content Safety | 33 | 4.6% | D1-Plan |
| 8 | GenAI General | 30 | 4.2% | D2-GenAI |
| 9 | Question Answering | 29 | 4.1% | D5-NLP |
| 10 | Vision General | 28 | 3.9% | D4-CV |

> 🎯 **Top 10 topics cover 443 questions (62%)**. Master these and you'll cover the majority of the exam.

---

### 4. Key Findings

1. **NLP is over-represented** (33.3% vs 15-20% official) — CLU/LUIS alone has 103 questions
2. **RAG is the GenAI core** — 55/107 GenAI questions are RAG-related
3. **Agentic is severely under-covered** — Only 8 questions vs 5-10% official weight; expect more Agent questions on the actual exam
4. **Two PDFs overlap significantly** — Use MS-AI102 (381q) as primary, 333q version for community discussion
5. **Content Safety trending up** — Multiple test-takers report increased Content Safety questions

---

### 5. Study Priority

| Priority | Topics | Questions | % |
|----------|--------|-----------|---|
| 🔴 P0 Must Master | CLU/LUIS, RAG, Resource Provisioning, Responsible AI, AI Search | 270 | 38% |
| 🟡 P1 Important | Bot/Dispatch, GenAI General, Q&A, Vision, Security, Speech | 170 | 24% |
| 🟢 P2 Basic Understanding | Custom Vision, Monitoring, Cost, Translation, Containers, Text Analytics, Doc Intelligence | 99 | 14% |
| ⚠️ Supplement | Agent Development (low in bank but increasing in real exam) | 8 | 1% |

---

### 6. Question Type Distribution

| Type | Count | % |
|------|-------|---|
| Multiple Choice | 328 | 45.9% |
| Other (Mixed/Code) | 232 | 32.4% |
| Hotspot | 89 | 12.4% |
| Drag-Drop | 40 | 5.6% |
| Yes/No | 26 | 3.6% |

---

### 7. References

- [AI-102 Official Study Guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-102)
- [Free Practice Assessment](https://learn.microsoft.com/credentials/certifications/exams/ai-102/practice/assessment?assessment-type=practice&assessmentId=61)
- [AI-102 知识结构完整指南](/knowledge_base/knowledge/ai/2026/03/15/deep-dive-ai-102-azure-ai-engineer-complete-guide/) — 本站 AI-102 知识地图
- [AI-102 考试通关策略](/knowledge_base/knowledge/ai/2026/03/15/ai-102-exam-study-strategy-guide/) — 本站备考策略指南
- 分类数据集：[ai102-questions-classified.json](https://github.com/IrisHeMin/knowledge_base/blob/master/files/ai102-questions-classified.json)
