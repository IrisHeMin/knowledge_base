---
layout: post
title: "Deep Dive: AI-102 考试通关策略 — 从备考到拿证的完整指南"
date: 2026-03-15
categories: [Knowledge, AI]
tags: [ai-102, azure-ai-engineer, certification, exam-guide, study-plan, 考试攻略]
type: "deep-dive"
---

# Deep Dive: AI-102 考试通关策略 — 从备考到拿证的完整指南

**Topic:** AI-102: Designing and Implementing a Microsoft Azure AI Solution — Exam Strategy  
**Category:** Certification / Study Guide  
**Level:** Practical Exam Preparation  
**Last Updated:** 2026-03-15  

---

## 中文版 (Chinese Version)

---

### 1. 考试基本信息

| 项目 | 详情 |
|------|------|
| **考试代码** | AI-102 |
| **考试名称** | Designing and Implementing a Microsoft Azure AI Solution |
| **认证名称** | Microsoft Certified: Azure AI Engineer Associate |
| **及格分数** | 700 / 1000 |
| **考试时长** | ~120 分钟 |
| **题目数量** | ~40-60 题 |
| **题型** | 单选、多选、拖拽排序、场景案例 |
| **有效期** | 1 年（可免费在线续期） |
| **考试语言** | 英语、中文（简体）、日语、韩语等 12 种语言 |
| **费用** | $165 USD |

**重要链接**：
- [官方 Study Guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-102)
- [免费 Practice Assessment](https://learn.microsoft.com/en-us/credentials/certifications/exams/ai-102/practice/assessment?assessment-type=practice&assessmentId=61)
- [考试沙盒体验](https://aka.ms/examdemo)
- [认证主页](https://learn.microsoft.com/en-us/credentials/certifications/azure-ai-engineer/)

---

### 2. 考试权重分布 — 决定学习优先级

```
┌─────────────────────────────────────────────────────────────────┐
│                    AI-102 考试权重分布                            │
│                                                                  │
│  1. 规划和管理 AI 解决方案          ████████████████████  20-25%  │
│  2. 生成式 AI 解决方案              ███████████████      15-20%  │
│  3. Agentic 解决方案                █████                 5-10%  │
│  4. 计算机视觉解决方案              ██████████           10-15%  │
│  5. 自然语言处理解决方案            ███████████████      15-20%  │
│  6. 知识挖掘和信息提取              ███████████████      15-20%  │
│                                                                  │
│  🔴 高优先级: 域1 + 域2 + 域6 = 50-65% (拿稳这三块基本能过)      │
│  🟡 中优先级: 域5 + 域4 = 25-35%                                 │
│  🟢 低优先级: 域3 = 5-10%                                        │
└─────────────────────────────────────────────────────────────────┘
```

> 💡 **核心洞察**：域 1（规划管理）+ 域 2（生成式 AI）+ 域 6（知识挖掘）合计占 **50-65%**，这三块拿稳基本就能过。

---

### 3. 六大考试领域详细考点

#### 域 1：规划和管理 AI 解决方案 (20-25%) 🔴

这是**分值最高**的领域，侧重"选择正确的服务"和"管理部署"。

##### 3.1.1 选择合适的 Microsoft Foundry 服务

| 场景需求 | 应选择的服务 |
|---------|-------------|
| 图像中提取文字 | Azure Vision (OCR) 或 Document Intelligence |
| 分析客户情感 | Azure Language — Sentiment Analysis |
| 实时语音翻译 | Azure Speech — Speech Translation |
| 从发票提取数据 | Document Intelligence — Invoice 预构建模型 |
| 聊天机器人理解意图 | Azure Language — CLU |
| 搜索企业文档 | Azure AI Search |
| 生成文本/代码 | Azure OpenAI (GPT-4o) |
| 生成图像 | Azure OpenAI (DALL-E) |
| 分析视频内容 | Azure Video Indexer |
| 多模态内容分析 | Azure Content Understanding |

> 🎯 **考试技巧**：这类题占比很大，核心是记住**每个服务的边界和最佳适用场景**。

##### 3.1.2 规划、创建和部署

| 考点 | 你需要知道的 |
|------|------------|
| **Responsible AI 原则** | 6 大原则：公平、可靠、隐私、包容、透明、问责 |
| **创建 Azure AI 资源** | 单服务 vs 多服务资源的区别 |
| **模型选择** | GPT-4o vs GPT-4 vs Phi-3 vs Llama 的选择依据 |
| **部署方式** | Serverless API vs Managed Compute vs Standard |
| **SDK 和 API** | Python SDK (azure-ai-projects) vs REST API |
| **Endpoint 确定** | 如何找到和使用服务端点 |
| **CI/CD 集成** | 将 AI 服务集成到 DevOps 管道 |
| **容器部署** | 在本地/边缘部署 AI 容器 |

##### 3.1.3 管理、监控和安全

| 考点 | 关键知识 |
|------|---------|
| **监控** | Azure Monitor、Application Insights、诊断日志 |
| **成本管理** | Token 计费、配额管理、成本预算 |
| **密钥管理** | Key Vault 存储密钥、密钥轮换 |
| **认证方式** | API Key vs Entra ID（推荐） vs Managed Identity |

##### 3.1.4 负责任 AI 实施

| 考点 | 关键知识 |
|------|---------|
| **内容审核** | Azure AI Content Safety 服务 |
| **内容安全** | 4 类有害内容（仇恨/暴力/性/自残）× 4 级严重程度 |
| **内容过滤器** | Input filter + Output filter 配置 |
| **Blocklist** | 自定义屏蔽词列表 |
| **Prompt Shield** | 防御 Prompt Injection / Jailbreak 攻击 |

---

#### 域 2：生成式 AI 解决方案 (15-20%) 🔴

##### 3.2.1 使用 Microsoft Foundry 构建

| 考点 | 关键知识 |
|------|---------|
| **Hub & Project** | Hub 是共享资源，Project 是工作空间 |
| **模型部署** | Model Catalog → 选择模型 → 部署 |
| **Prompt Flow** | Flow 类型（Standard/Chat/Evaluation）、节点类型、Connection |
| **RAG 模式** | Chunking → Embedding → Index → Retrieval → Grounding |
| **模型评估** | Groundedness / Relevance / Coherence / Fluency 指标 |
| **Foundry SDK** | AIProjectClient 使用、ChatCompletions 调用 |
| **Prompt Template** | 参数化提示模板、变量替换 |

##### 3.2.2 Azure OpenAI 生成内容

| 考点 | 关键知识 |
|------|---------|
| **资源预配** | 创建 Azure OpenAI 资源、区域选择 |
| **模型选择部署** | GPT-4o / GPT-4 / DALL-E / Whisper |
| **Prompt 工程** | System/User/Assistant 消息角色 |
| **DALL-E 图像生成** | 文本到图像、图像编辑 |
| **多模态** | GPT-4o 处理文本+图像输入 |
| **应用集成** | REST API 和 SDK 调用方式 |

##### 3.2.3 优化和运维

| 考点 | 关键知识 |
|------|---------|
| **参数调优** | temperature / top_p / max_tokens / frequency_penalty |
| **模型监控** | 性能指标、资源消耗、诊断设置 |
| **资源优化** | 扩展性、配额管理、模型更新 |
| **Tracing & Feedback** | 启用追踪、收集用户反馈 |
| **Model Reflection** | 模型自我评估和改进 |
| **容器部署** | 本地和边缘设备部署 |
| **多模型编排** | 编排多个生成式模型 |
| **Prompt 工程技巧** | Few-shot、Chain-of-thought、结构化输出 |
| **Fine-tuning** | 训练数据准备、超参数配置、评估 |

---

#### 域 3：Agentic 解决方案 (5-10%) 🟢

虽然权重最低，但这是 **2025 年新增**的热门领域：

| 考点 | 关键知识 |
|------|---------|
| **Agent 概念** | Agent = LLM + Tools + Memory + Planning |
| **使用场景** | 何时该用 Agent vs 普通 LLM 调用 |
| **资源配置** | 创建 Agent 所需的 Azure 资源 |
| **Foundry Agent Service** | 通过 Portal/SDK 创建和管理 Agent |
| **Microsoft Agent Framework** | Semantic Kernel 构建复杂 Agent |
| **多 Agent 编排** | Agent Group Chat、编排策略 |
| **测试/优化/部署** | Agent 的测试和发布流程 |

---

#### 域 4：计算机视觉解决方案 (10-15%) 🟡

##### 3.4.1 图像分析

| 考点 | 关键知识 |
|------|---------|
| **Visual Features** | Tags / Description / Objects / Faces / Read / SmartCrops |
| **对象检测** | 返回边界框 (bounding box) 坐标 |
| **OCR** | Image Analysis Read API、手写识别 |
| **API 响应解读** | JSON 结构中各字段含义 |

##### 3.4.2 自定义视觉模型

| 考点 | 关键知识 |
|------|---------|
| **分类 vs 检测** | 分类=是什么，检测=是什么+在哪里 |
| **标注** | 分类标签 vs 边界框标注 |
| **训练和评估** | Precision / Recall / mAP 指标 |
| **发布和使用** | Prediction URL、SDK 调用 |
| **代码优先** | 通过代码创建 Custom Vision 项目 |

##### 3.4.3 视频分析

| 考点 | 关键知识 |
|------|---------|
| **Video Indexer** | 从视频/直播提取洞察 |
| **Spatial Analysis** | 检测视频中人员的存在和移动 |

---

#### 域 5：自然语言处理解决方案 (15-20%) 🟡

##### 3.5.1 分析和翻译文本

| 考点 | 关键知识 |
|------|---------|
| **关键短语提取** | Key Phrase Extraction API |
| **实体识别** | 预构建 NER + Entity Linking |
| **情感分析** | 文档/句子级情感 + 意见挖掘 |
| **语言检测** | 自动识别文本语言 |
| **PII 检测** | 检测和脱敏个人敏感信息 |
| **文本翻译** | Translator Service、文档翻译 |

##### 3.5.2 处理和翻译语音

| 考点 | 关键知识 |
|------|---------|
| **生成式 AI 语音** | 在应用中集成 AI 语音能力 |
| **STT/TTS** | SpeechConfig、AudioConfig 配置 |
| **SSML** | 控制语音的语速、语调、停顿 |
| **自定义语音** | Custom Speech 模型训练 |
| **意图识别** | 结合 CLU 的语音意图识别 |
| **语音翻译** | 语音到语音、语音到文本翻译 |

##### 3.5.3 自定义语言模型

| 考点 | 关键知识 |
|------|---------|
| **CLU** | Intent / Entity / Utterance 定义和标注 |
| **训练评估部署** | Train → Evaluate (P/R/F1) → Deploy → Test |
| **模型优化** | 备份、恢复、迭代改进 |
| **问答服务** | 创建项目、导入源、QA 对 |
| **知识库** | 训练、测试、发布 |
| **多轮对话** | Follow-up prompts 配置 |
| **替代短语** | 同义词和闲聊 (Chit-chat) |
| **多语言** | 多语言问答解决方案 |
| **自定义文本分类** | 单标签 vs 多标签分类 |
| **自定义 NER** | 自定义实体类型定义和训练 |
| **自定义翻译** | 训练、改进、发布自定义翻译模型 |

---

#### 域 6：知识挖掘和信息提取 (15-20%) 🔴

##### 3.6.1 Azure AI Search 解决方案

| 考点 | 关键知识 |
|------|---------|
| **预配资源** | 创建 Search 资源、定义 Index |
| **Skillset** | 内置技能 + 自定义技能 |
| **数据源和索引器** | Blob / SQL / Cosmos DB 数据源 |
| **查询语法** | 简单查询、Lucene 查询、过滤、排序、通配符 |
| **Knowledge Store** | File / Object / Table 投影 |
| **语义和向量** | 语义搜索配置、向量存储 |

##### 3.6.2 Document Intelligence 解决方案

| 考点 | 关键知识 |
|------|---------|
| **预配资源** | 创建 Document Intelligence 资源 |
| **预构建模型** | Invoice / Receipt / ID / Business Card / Tax |
| **自定义模型** | Template vs Neural 模型 |
| **训练测试发布** | 标注数据、训练、评估 |
| **组合模型** | Composed Model 组合多个子模型 |

##### 3.6.3 Content Understanding

| 考点 | 关键知识 |
|------|---------|
| **OCR 管道** | 从图像/文档提取文字 |
| **文档分析** | 摘要、分类、属性检测 |
| **实体/表格/图像提取** | 从文档中提取结构化信息 |
| **多模态处理** | 文档、图像、视频、音频统一处理 |

---

### 4. 四步通关策略

```
┌──────────────────────────────────────────────────────┐
│                 AI-102 通关路线图                      │
│                                                       │
│  Step 1: 摸底 (Day 1)                                 │
│  ├── 做 Practice Assessment（裸考）                     │
│  ├── 对照考试权重找出薄弱领域                            │
│  └── 制定个人学习计划                                   │
│                                                       │
│  Step 2: 系统学习 (Day 2-7)                            │
│  ├── 🔴 域1: 规划管理 (花最多时间)                      │
│  ├── 🔴 域2: 生成式 AI                                 │
│  ├── 🔴 域6: 知识挖掘                                  │
│  ├── 🟡 域5: NLP                                      │
│  ├── 🟡 域4: 计算机视觉                                │
│  └── 🟢 域3: Agent                                    │
│                                                       │
│  Step 3: 动手实验 (Day 5-9)                            │
│  ├── 每个 Learning Path 的 Lab 至少做一遍               │
│  ├── 重点实操 RAG、AI Search、Document Intelligence     │
│  └── 熟悉 Portal 操作 + SDK 代码                       │
│                                                       │
│  Step 4: 冲刺 (Day 10-14)                             │
│  ├── 再做 Practice Assessment（目标 > 80%）             │
│  ├── 逐条过 Study Guide 每个 bullet point              │
│  ├── 重点复习错题领域                                   │
│  └── 约考 → 通过 ✅                                    │
└──────────────────────────────────────────────────────┘
```

---

### 5. 高频考试场景题型总结

#### 5.1 "应该选择哪个服务？" 类型

```
📝 题目模式：给你一个业务场景，问你应该用哪个 Azure AI 服务

💡 解题关键：
├── 需要理解自然语言意图？ → CLU
├── 需要回答 FAQ？ → Question Answering
├── 需要从发票提取数据？ → Document Intelligence (Invoice 模型)
├── 需要搜索企业文档？ → Azure AI Search
├── 需要生成文本？ → Azure OpenAI
├── 需要从图像读取文字？ → Vision (OCR) 或 Document Intelligence
├── 需要分析视频？ → Video Indexer
├── 需要翻译？ → Translator Service
├── 需要语音识别？ → Speech Service
└── 需要分析多种格式内容？ → Content Understanding
```

#### 5.2 "参数配置" 类型

```
📝 题目模式：给定需求，问你应该设置什么参数

💡 关键参数速记：
├── temperature: 0 = 确定性输出, 1 = 创造性输出
├── top_p: 控制词汇选择范围（与 temperature 类似效果）
├── max_tokens: 控制输出最大长度
├── frequency_penalty: 减少重复（正值=少重复）
├── presence_penalty: 鼓励新话题（正值=更多样）
└── stop: 定义停止生成的标记
```

#### 5.3 "实现步骤排序" 类型

```
📝 题目模式：拖拽排列实现步骤的正确顺序

💡 常见流程速记：

RAG 实现:
1. 准备数据源 → 2. 文档切片 → 3. 生成嵌入 → 4. 创建搜索索引
→ 5. 配置检索 → 6. 设计 Prompt → 7. 测试和评估

CLU 训练:
1. 创建项目 → 2. 定义 Intent → 3. 定义 Entity
→ 4. 添加标注 Utterance → 5. 训练 → 6. 评估 → 7. 部署

AI Search 管道:
1. 创建 Search 资源 → 2. 定义数据源 → 3. 定义 Skillset
→ 4. 定义 Index → 5. 创建 Indexer → 6. 运行 Indexer → 7. 查询
```

#### 5.4 "代码片段" 类型

```
📝 题目模式：给出代码片段，填空或找错

💡 需要熟悉的 SDK 模式：
├── Azure OpenAI: ChatCompletionsClient, messages 角色
├── Language: TextAnalyticsClient, analyze_sentiment()
├── Speech: SpeechConfig, AudioConfig, SpeechRecognizer
├── Vision: ImageAnalysisClient, analyze()
├── Document Intelligence: DocumentIntelligenceClient, begin_analyze_document()
└── AI Search: SearchClient, search() 查询语法
```

---

### 6. 学习资源推荐

| 资源 | 说明 | 链接 |
|------|------|------|
| **官方 Learning Path** | 5 个学习路径，40 个模块 | [AI-102 Course](https://learn.microsoft.com/training/courses/ai-102t00) |
| **Practice Assessment** | 免费官方模拟题 | [Practice Test](https://learn.microsoft.com/credentials/certifications/exams/ai-102/practice/assessment?assessment-type=practice&assessmentId=61) |
| **Study Guide** | 考试技能清单 | [Study Guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-102) |
| **Exam Sandbox** | 考试界面体验 | [Exam Demo](https://aka.ms/examdemo) |
| **AI-102 知识结构** | 本站完整知识地图 | [Knowledge Map](/case-knowledge-base/knowledge/ai/2026/03/15/deep-dive-ai-102-azure-ai-engineer-complete-guide.html) |

---

### 7. 考试日 Tips

1. **提前 15 分钟**到达考试中心（或在线考试提前准备环境）
2. **先做简单题**，标记不确定的题目稍后回来
3. **场景题仔细读**：注意关键词如 "least administrative effort"、"minimize cost"、"most secure"
4. **排除法**：不确定时先排除明显错误选项
5. **时间管理**：平均每题 2-3 分钟，不要在一题上卡太久
6. **Case Study**：案例题分值高，仔细阅读所有约束条件

---

## English Version

---

### 1. Exam Overview

| Item | Details |
|------|---------|
| **Exam Code** | AI-102 |
| **Exam Name** | Designing and Implementing a Microsoft Azure AI Solution |
| **Certification** | Microsoft Certified: Azure AI Engineer Associate |
| **Passing Score** | 700 / 1000 |
| **Duration** | ~120 minutes |
| **Questions** | ~40-60 |
| **Question Types** | Single/multi-select, drag-and-drop, case studies |
| **Validity** | 1 year (free online renewal) |
| **Cost** | $165 USD |

---

### 2. Exam Weight Distribution

```
┌─────────────────────────────────────────────────────────────────┐
│                    AI-102 Exam Weight Distribution                │
│                                                                  │
│  1. Plan & manage AI solution           ████████████████  20-25% │
│  2. Generative AI solutions             ████████████      15-20% │
│  3. Agentic solutions                   ████               5-10% │
│  4. Computer vision solutions           ████████          10-15% │
│  5. NLP solutions                       ████████████      15-20% │
│  6. Knowledge mining & extraction       ████████████      15-20% │
│                                                                  │
│  🔴 High priority: Domain 1+2+6 = 50-65% (secure these to pass) │
│  🟡 Medium: Domain 5+4 = 25-35%                                  │
│  🟢 Lower: Domain 3 = 5-10%                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

### 3. Four-Step Strategy to Pass

#### Step 1: Baseline Assessment (Day 1)
- Take the [free Practice Assessment](https://learn.microsoft.com/en-us/credentials/certifications/exams/ai-102/practice/assessment?assessment-type=practice&assessmentId=61) cold (without studying first)
- Identify weak areas against the exam weight distribution
- Create your personalized study plan

#### Step 2: Systematic Study (Day 2-7)

**Priority order by exam weight:**

| Priority | Domain | Weight | Key Focus Areas |
|----------|--------|--------|-----------------|
| 🔴 Highest | 1. Plan & Manage | 20-25% | Service selection, deployment options, auth, Responsible AI |
| 🔴 Highest | 2. Generative AI | 15-20% | RAG pattern, Prompt Flow, model deployment, fine-tuning, parameters |
| 🔴 Highest | 6. Knowledge Mining | 15-20% | AI Search (Indexer/Skillset/Index), Document Intelligence, Content Understanding |
| 🟡 High | 5. NLP | 15-20% | CLU (Intent/Entity), Q&A service, text analytics, Speech STT/TTS, SSML |
| 🟡 Medium | 4. Computer Vision | 10-15% | Image Analysis API, Custom Vision (classification vs detection), OCR, Video Indexer |
| 🟢 Lower | 3. Agents | 5-10% | Foundry Agent Service basics, multi-agent orchestration concepts |

#### Step 3: Hands-on Labs (Day 5-9)
- Complete labs for each Learning Path (at least once)
- Focus on RAG, AI Search, and Document Intelligence
- Get comfortable with both Portal and SDK approaches

#### Step 4: Final Sprint (Day 10-14)
- Retake Practice Assessment (target >80% accuracy)
- Review every bullet point in the [Study Guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-102)
- Focus on areas where you got questions wrong
- Schedule and take the exam ✅

---

### 4. Common Question Patterns

#### "Which service should you use?"
The most frequent question type. You need to know the **boundary and best-fit scenario** for each Azure AI service.

| Scenario | Service |
|----------|---------|
| Understand natural language intent | CLU |
| Answer FAQs | Question Answering |
| Extract data from invoices | Document Intelligence (Invoice model) |
| Search enterprise documents | Azure AI Search |
| Generate text/code | Azure OpenAI (GPT-4o) |
| Read text from images | Vision (OCR) or Document Intelligence |
| Analyze video | Video Indexer |
| Translate text | Translator Service |
| Speech recognition | Speech Service |
| Multimodal content | Content Understanding |

#### "Parameter configuration"
Key parameters to memorize:
- `temperature`: 0 = deterministic, 1 = creative
- `top_p`: Controls vocabulary selection range
- `max_tokens`: Maximum output length
- `frequency_penalty`: Reduce repetition (positive = less repetition)
- `presence_penalty`: Encourage new topics (positive = more diverse)

#### "Order the implementation steps"

**RAG**: Prepare data → Chunk documents → Generate embeddings → Create search index → Configure retrieval → Design prompt → Test & evaluate

**CLU**: Create project → Define intents → Define entities → Add labeled utterances → Train → Evaluate → Deploy

**AI Search**: Create resource → Define data source → Define skillset → Define index → Create indexer → Run indexer → Query

---

### 5. Exam Day Tips

1. **Arrive 15 minutes early** (or prepare online exam environment ahead)
2. **Do easy questions first**, flag uncertain ones for later
3. **Read scenario questions carefully**: Watch for keywords like "least administrative effort", "minimize cost", "most secure"
4. **Use elimination**: When unsure, rule out obviously wrong options first
5. **Time management**: Average 2-3 minutes per question, don't get stuck
6. **Case studies**: High-value questions — read all constraints carefully

---

### 6. Recommended Study Timeline

| Phase | Time | Activities |
|-------|------|-----------|
| Baseline | Day 1 | Practice Assessment + review knowledge structure |
| Core Study | Day 2-7 | 5 Learning Paths (prioritize first 3) |
| Hands-on Labs | Day 5-9 | Complete at least one lab per Path |
| Sprint | Day 10-12 | Practice Assessment + fill knowledge gaps |
| Exam | Day 13-14 | Schedule and pass ✅ |

> With Azure + Python background: **2 weeks** is sufficient. Zero background: plan for 3-4 weeks.

---

### 7. References

- [AI-102 Course](https://learn.microsoft.com/training/courses/ai-102t00) — Official course with 5 learning paths
- [Study Guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-102) — Detailed skills measured checklist
- [Practice Assessment](https://learn.microsoft.com/credentials/certifications/exams/ai-102/practice/assessment?assessment-type=practice&assessmentId=61) — Free official practice questions
- [Exam Sandbox](https://aka.ms/examdemo) — Experience the exam interface
- [Certification Page](https://learn.microsoft.com/credentials/certifications/azure-ai-engineer/) — Certification details and scheduling
