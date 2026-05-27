---
layout: post
title: "Deep Dive: AI 小白学习路线图 — 从零开始的 AI 知识图谱"
date: 2026-03-09
categories: [Knowledge, AI-Fundamentals]
tags: [ai, machine-learning, deep-learning, llm, prompt-engineering, agents, learning-roadmap, knowledge-map]
type: "deep-dive"
---

# Deep Dive: AI 小白学习路线图 — 从零开始的 AI 知识图谱

**Topic:** AI Learning Roadmap & Knowledge Map
**Category:** AI Fundamentals
**Level:** 入门
**Last Updated:** 2026-03-09

---

## 中文版

---

### 1. 概述 (Overview)

这份文档是为 **AI 零基础学习者**设计的完整学习路线图。它将 AI 的庞大知识体系整理成一棵**由浅入深的知识树**，帮助你从「完全不懂 AI」到「能理解、使用并构建 AI 应用」。

AI（人工智能）这个领域看似庞大复杂，但如果你按照正确的顺序学习，其实每一步都是自然的递进。就像学开车一样：你不需要先成为机械工程师，只需要先学会方向盘和油门，然后慢慢理解发动机的原理。

这份路线图将 AI 知识分为 **7 个层级**，每个层级包含多个知识点。建议按层级顺序学习，每个知识点都会有对应的 Deep Dive 文章详细讲解。

### 2. 知识图谱总览 (Knowledge Map Overview)

```
                    ┌─────────────────────────────────────┐
                    │    🗺️ AI 小白学习路线图 (7 个层级)     │
                    └──────────────────┬──────────────────┘
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          │                            │                            │
          ▼                            ▼                            ▼
  ┌───────────────┐         ┌──────────────────┐         ┌──────────────────┐
  │ Level 0       │         │ Level 1          │         │ Level 2          │
  │ 🌱 AI 是什么   │────────▶│ 🧠 AI 怎么学习的  │────────▶│ 🏗️ 深度学习基础   │
  │               │         │                  │         │                  │
  │ • AI 的定义    │         │ • 监督学习        │         │ • 神经网络        │
  │ • AI 的历史    │         │ • 无监督学习      │         │ • CNN (图像)      │
  │ • AI 的分类    │         │ • 强化学习        │         │ • RNN (序列)      │
  │ • AI vs ML    │         │ • 训练与推理      │         │ • Transformer ⭐  │
  │  vs DL        │         │ • 过拟合/欠拟合   │         │ • 注意力机制 ⭐    │
  └───────────────┘         └──────────────────┘         └──────────────────┘
                                                                  │
                                                                  ▼
  ┌───────────────┐         ┌──────────────────┐         ┌──────────────────┐
  │ Level 5       │         │ Level 4          │         │ Level 3          │
  │ 🤖 AI Agent   │◀────────│ 🎯 Prompt 工程    │◀────────│ 💬 大语言模型     │
  │               │         │                  │         │    (LLM)         │
  │ • 什么是Agent │         │ • Prompt 基础     │         │ • 什么是 LLM     │
  │ • Skills/Tools│         │ • Zero/Few-shot  │         │ • GPT/Claude/    │
  │ • Function    │         │ • Chain of       │         │   Gemini 对比    │
  │   Calling     │         │   Thought        │         │ • Tokenization   │
  │ • MCP 协议    │         │ • System Prompt  │         │ • 训练过程        │
  │ • Multi-Agent │         │ • 高级技巧       │         │ • 上下文窗口      │
  └───────────────┘         └──────────────────┘         │ • 参数与调优      │
          │                                              └──────────────────┘
          ▼
  ┌───────────────┐         ┌──────────────────┐
  │ Level 6       │         │ Level 7          │
  │ 🔧 AI 开发实战 │────────▶│ 🛡️ AI 进阶话题   │
  │               │         │                  │
  │ • API 调用    │         │ • 多模态 AI      │
  │ • SDK 使用    │         │ • AI 安全与对齐   │
  │ • RAG 检索增强 │         │ • 负责任的 AI    │
  │ • 向量数据库   │         │ • AI 治理        │
  │ • 微调 Fine-  │         │ • 未来趋势       │
  │   tuning      │         │                  │
  └───────────────┘         └──────────────────┘
```

### 3. 详细知识点分解 (Detailed Knowledge Points)

---

#### 🌱 Level 0: AI 是什么？（认知启蒙）

> **学习目标**：理解 AI 的基本概念，建立正确的认知框架，不再对 AI 感到神秘或恐惧。

| # | 知识点 | 一句话说明 | 学完你能回答 |
|---|--------|-----------|-------------|
| 0-1 | **AI 的定义与历史** | 从图灵测试到 ChatGPT，AI 走过了 70 年 | "AI 到底是什么？它是怎么发展起来的？" |
| 0-2 | **AI vs ML vs DL** | 三者是套娃关系：AI > ML > DL | "机器学习和深度学习有什么区别？" |
| 0-3 | **AI 的分类** | 弱 AI vs 强 AI vs 超级 AI | "现在的 ChatGPT 算真正的 AI 吗？" |
| 0-4 | **AI 能做什么** | 文本、图像、语音、代码、决策…… | "AI 目前能解决哪些实际问题？" |
| 0-5 | **AI 不能做什么** | AI 的局限性和常见误解 | "AI 会取代人类吗？它的边界在哪？" |

**类比理解**：
- **AI** 就像「让机器变聪明」这个大目标
- **Machine Learning** 是实现这个目标的一种方法（让机器从数据中学习）
- **Deep Learning** 是 ML 中最强大的一种技术（用多层神经网络学习）
- **LLM（大语言模型）** 是 DL 最新最火的应用（ChatGPT、Claude 都是）

```
AI（人工智能）
 └── Machine Learning（机器学习）
      ├── 传统 ML（决策树、SVM、随机森林……）
      └── Deep Learning（深度学习）
           ├── CNN（图像识别）
           ├── RNN（语音识别）
           └── Transformer（⭐ 当今 AI 革命的核心）
                └── LLM（大语言模型：GPT、Claude、Gemini……）
```

---

#### 🧠 Level 1: AI 怎么学习的？（机器学习基础）

> **学习目标**：理解机器学习的核心思想 —— 机器如何从数据中「学习」规律。

| # | 知识点 | 一句话说明 | 学完你能回答 |
|---|--------|-----------|-------------|
| 1-1 | **监督学习** | 给机器看带答案的题目，让它学会做题 | "机器怎么学会识别猫和狗的？" |
| 1-2 | **无监督学习** | 给机器一堆数据，让它自己发现规律 | "机器怎么给客户自动分群的？" |
| 1-3 | **强化学习** | 让机器在试错中学习，做对了给奖励 | "AlphaGo 是怎么学会下棋的？" |
| 1-4 | **训练与推理** | 训练 = 上学，推理 = 考试 | "ChatGPT 回答问题时在做什么？" |
| 1-5 | **过拟合与欠拟合** | 死记硬背 vs 啥也没学会 | "为什么 AI 有时会犯很蠢的错误？" |
| 1-6 | **数据集与特征** | 数据是 AI 的食物，特征是食物的营养 | "为什么都说'数据是新时代的石油'？" |
| 1-7 | **评估指标** | 怎么判断 AI 学得好不好 | "准确率 99% 的模型一定好吗？" |

**核心类比**：

| 机器学习概念 | 人类学习类比 |
|------------|------------|
| 训练数据 | 课本和练习题 |
| 模型 | 学生的大脑 |
| 训练过程 | 上课 + 做题 |
| 参数/权重 | 大脑中的突触连接 |
| 推理/预测 | 考试答题 |
| 过拟合 | 只会做原题，换个说法就不会了 |
| 欠拟合 | 上课没认真听，啥都不会 |

---

#### 🏗️ Level 2: 深度学习基础（理解 AI 的引擎）

> **学习目标**：理解深度学习的核心架构，特别是 **Transformer** —— 这是当今 AI 革命的发动机。

| # | 知识点 | 一句话说明 | 学完你能回答 |
|---|--------|-----------|-------------|
| 2-1 | **神经网络基础** | 模仿人脑神经元的数学模型 | "神经网络是怎么工作的？" |
| 2-2 | **CNN（卷积神经网络）** | 专门处理图像的网络结构 | "AI 是怎么识别照片里的人脸的？" |
| 2-3 | **RNN/LSTM** | 专门处理序列数据（文字、语音）的网络 | "AI 是怎么理解一句话的先后顺序的？" |
| 2-4 | **⭐ Transformer** | 现代 AI 的核心架构，GPT/Claude 的基石 | "为什么 Transformer 改变了整个 AI 行业？" |
| 2-5 | **⭐ 注意力机制** | 让 AI 学会「重点关注」的能力 | "AI 怎么知道一段话里哪些词更重要？" |
| 2-6 | **预训练与迁移学习** | 先学通识，再学专业 | "为什么大模型都要先预训练？" |

**为什么 Transformer 是重中之重？**

2017 年 Google 发表论文《Attention Is All You Need》提出了 Transformer 架构，从此改变了整个 AI 行业：
- **GPT**（OpenAI）= **G**enerative **P**re-trained **T**ransformer
- **BERT**（Google）= **B**idirectional **E**ncoder **R**epresentations from **T**ransformers
- **Claude**（Anthropic）= 基于 Transformer 的大语言模型

可以说，**不理解 Transformer，就无法真正理解现代 AI**。

---

#### 💬 Level 3: 大语言模型 LLM（当今 AI 的主角）

> **学习目标**：深入理解 ChatGPT、Claude 等 LLM 的工作原理，知道它们能做什么、不能做什么、为什么会「幻觉」。

| # | 知识点 | 一句话说明 | 学完你能回答 |
|---|--------|-----------|-------------|
| 3-1 | **什么是 LLM** | 在海量文本上训练的超大 Transformer 模型 | "ChatGPT 的本质是什么？" |
| 3-2 | **主流 LLM 对比** | GPT、Claude、Gemini、LLaMA 各有千秋 | "我该用哪个 AI 模型？" |
| 3-3 | **Tokenization** | AI 不读字，它读「Token」 | "为什么中文消耗的 Token 比英文多？" |
| 3-4 | **训练三阶段** | Pre-training → Fine-tuning → RLHF | "ChatGPT 是怎么被训练出来的？" |
| 3-5 | **上下文窗口** | AI 的「短期记忆」容量 | "为什么对话太长 AI 会忘记前面的内容？" |
| 3-6 | **Temperature 等参数** | 控制 AI 回答的「创造力」和「确定性」 | "为什么同一个问题 AI 每次回答不同？" |
| 3-7 | **幻觉（Hallucination）** | AI 一本正经地胡说八道 | "为什么 AI 会编造不存在的信息？" |
| 3-8 | **Token 计费与成本** | LLM 按 Token 数量收费 | "用 AI API 要花多少钱？" |

---

#### 🎯 Level 4: Prompt Engineering（与 AI 对话的艺术）

> **学习目标**：掌握如何写出好的 Prompt，让 AI 给出你想要的回答。这是**最实用、见效最快**的技能。

| # | 知识点 | 一句话说明 | 学完你能回答 |
|---|--------|-----------|-------------|
| 4-1 | **Prompt 基础** | Prompt 就是你给 AI 的指令 | "怎么让 AI 更好地理解我的需求？" |
| 4-2 | **Zero-shot vs Few-shot** | 不给例子 vs 给几个例子 | "给 AI 看几个例子真的有用吗？" |
| 4-3 | **Chain of Thought** | 让 AI「一步一步想」，答案更准确 | "怎么让 AI 做复杂的推理？" |
| 4-4 | **System Prompt** | 给 AI 设定角色和规则 | "怎么让 AI 一直按我的要求回答？" |
| 4-5 | **结构化输出** | 让 AI 输出 JSON、表格等特定格式 | "怎么让 AI 输出程序能读取的格式？" |
| 4-6 | **Prompt 高级技巧** | XML 标签、角色扮演、思维链等 | "顶级 Prompt 工程师都用什么技巧？" |
| 4-7 | **常见 Prompt 模式** | CRISPE、RACE 等经典模板 | "有没有写 Prompt 的万能模板？" |

**为什么 Prompt Engineering 排在开发之前？**

因为对于大多数人来说，你**不需要写代码**就能通过好的 Prompt 让 AI 帮你完成大量工作。这是投入产出比最高的 AI 技能。

---

#### 🤖 Level 5: AI Agent 与 Skills（让 AI 真正「做事」）

> **学习目标**：理解 AI 如何从「只会说话」进化为「能执行操作的智能体」。

| # | 知识点 | 一句话说明 | 学完你能回答 |
|---|--------|-----------|-------------|
| 5-1 | **什么是 AI Agent** | 能自主规划和执行任务的 AI 系统 | "Agent 和普通聊天机器人有什么区别？" |
| 5-2 | **⭐ Skills / Tools** | 赋予 AI 调用外部功能的能力 | "AI 怎么操作文件、查数据库的？" |
| 5-3 | **Function Calling** | LLM 调用外部函数的底层机制 | "AI 是怎么知道该调用哪个工具的？" |
| 5-4 | **Plugins 插件系统** | 一组相关 Tools 的逻辑集合 | "ChatGPT 的插件是怎么工作的？" |
| 5-5 | **⭐ MCP 协议** | AI 连接外部系统的「USB-C 标准」 | "怎么让不同的 AI 都能用同一个工具？" |
| 5-6 | **Multi-Agent** | 多个 AI Agent 协作完成复杂任务 | "多个 AI 怎么分工合作？" |
| 5-7 | **Human-in-the-Loop** | 人机协作，人类审批关键决策 | "怎么确保 AI 不会做出危险的操作？" |

> 📌 **注意**：你已经在日常使用中体验了 Skills！你用的这个 GitHub Copilot CLI 就是一个 AI Agent，它的 `grep`、`powershell`、`edit`、`web_fetch` 等都是 Skills/Tools。

---

#### 🔧 Level 6: AI 开发实战（动手构建 AI 应用）

> **学习目标**：学会使用 API 和框架构建自己的 AI 应用。

| # | 知识点 | 一句话说明 | 学完你能回答 |
|---|--------|-----------|-------------|
| 6-1 | **AI API 入门** | 通过 API 调用 GPT/Claude 的能力 | "怎么在自己的程序里调用 AI？" |
| 6-2 | **⭐ RAG 检索增强生成** | 让 AI 基于你的私有数据回答问题 | "怎么让 AI 回答我们公司的内部问题？" |
| 6-3 | **向量数据库** | 存储和搜索语义相似内容的专用数据库 | "AI 怎么在海量文档中找到相关信息？" |
| 6-4 | **Semantic Kernel / LangChain** | 构建 AI 应用的主流框架 | "有没有帮我快速搭 AI 应用的框架？" |
| 6-5 | **Fine-tuning 微调** | 用自己的数据定制模型 | "怎么让 AI 学会我们行业的专业知识？" |
| 6-6 | **Embedding 嵌入** | 将文本转化为 AI 能理解的数字向量 | "AI 怎么理解两段文字是否相似？" |
| 6-7 | **AI 应用架构** | 设计完整的 AI 应用系统 | "一个企业级 AI 应用的架构长什么样？" |

---

#### 🛡️ Level 7: AI 进阶话题（视野拓展）

> **学习目标**：理解 AI 的前沿发展和社会影响，形成全面的 AI 世界观。

| # | 知识点 | 一句话说明 | 学完你能回答 |
|---|--------|-----------|-------------|
| 7-1 | **多模态 AI** | 同时理解文字、图片、音频、视频的 AI | "AI 怎么看懂图片并描述出来的？" |
| 7-2 | **AI 安全与对齐** | 确保 AI 的行为符合人类意图 | "怎么防止 AI 做出有害的事？" |
| 7-3 | **负责任的 AI** | 公平、透明、可解释的 AI | "AI 做出的决定公平吗？" |
| 7-4 | **AI 治理与法规** | AI 的法律、合规和伦理框架 | "使用 AI 有什么法律风险？" |
| 7-5 | **AI 前沿趋势** | AGI、AI OS、World Model 等 | "AI 的下一步会走向哪里？" |

---

### 4. 推荐学习路径 (Recommended Learning Path)

根据你的角色和目标，可以选择不同的学习路径：

#### 路径 A：「AI 使用者」（最快见效 ⚡）

适合：想立即用 AI 提升工作效率的人

```
Level 0 (AI 基本概念)
    ↓ [1-2 天]
Level 3 (LLM 基础，重点: 3-1, 3-5, 3-7)
    ↓ [2-3 天]
Level 4 (Prompt Engineering，重点学习 ⭐)
    ↓ [持续练习]
Level 5 (AI Agent 概念，重点: 5-1, 5-2)
```

> 💡 **这条路径不需要任何编程基础**，4-7 天就能显著提升你使用 AI 的能力。

#### 路径 B：「AI 理解者」（建立完整认知 🧠）

适合：想系统理解 AI 原理的技术人员

```
Level 0 → Level 1 → Level 2 → Level 3 → Level 4 → Level 5
```

> 按层级顺序学习，每个层级 3-5 天，总共约 3-4 周。

#### 路径 C：「AI 开发者」（动手构建 💻）

适合：想开发 AI 应用的程序员

```
Level 0 → Level 1（快速过） → Level 3 → Level 4 → Level 5 → Level 6
```

> 重点在 Level 5 和 Level 6，需要编程基础。

### 5. 知识点文章追踪表 (Article Tracking)

以下是每个知识点对应的 Deep Dive 文章状态：

| 层级 | 知识点编号 | 标题 | 状态 |
|------|-----------|------|------|
| Level 0 | 0-1 | AI 的定义与历史 | 📝 待编写 |
| Level 0 | 0-2 | AI vs ML vs DL：三者的关系 | 📝 待编写 |
| Level 0 | 0-3 | AI 的分类：弱 AI vs 强 AI | 📝 待编写 |
| Level 0 | 0-4 | AI 能做什么：当前 AI 的能力范围 | 📝 待编写 |
| Level 0 | 0-5 | AI 不能做什么：局限性与误解 | 📝 待编写 |
| Level 1 | 1-1 | 监督学习：给机器做带答案的题 | 📝 待编写 |
| Level 1 | 1-2 | 无监督学习：让机器自己找规律 | 📝 待编写 |
| Level 1 | 1-3 | 强化学习：在试错中成长 | 📝 待编写 |
| Level 1 | 1-4 | 训练与推理：上学 vs 考试 | 📝 待编写 |
| Level 1 | 1-5 | 过拟合与欠拟合 | 📝 待编写 |
| Level 1 | 1-6 | 数据集与特征工程 | 📝 待编写 |
| Level 1 | 1-7 | 评估指标：怎么判断 AI 好不好 | 📝 待编写 |
| Level 2 | 2-1 | 神经网络基础 | 📝 待编写 |
| Level 2 | 2-2 | CNN：AI 怎么看懂图片 | 📝 待编写 |
| Level 2 | 2-3 | RNN/LSTM：AI 怎么理解序列 | 📝 待编写 |
| Level 2 | 2-4 | ⭐ Transformer：现代 AI 的核心引擎 | 📝 待编写 |
| Level 2 | 2-5 | ⭐ 注意力机制：AI 的「聚焦」能力 | 📝 待编写 |
| Level 2 | 2-6 | 预训练与迁移学习 | 📝 待编写 |
| Level 3 | 3-1 | 什么是大语言模型 (LLM) | 📝 待编写 |
| Level 3 | 3-2 | 主流 LLM 对比：GPT vs Claude vs Gemini | 📝 待编写 |
| Level 3 | 3-3 | Tokenization：AI 的阅读方式 | 📝 待编写 |
| Level 3 | 3-4 | LLM 训练三阶段 | 📝 待编写 |
| Level 3 | 3-5 | 上下文窗口：AI 的短期记忆 | 📝 待编写 |
| Level 3 | 3-6 | Temperature 等推理参数 | 📝 待编写 |
| Level 3 | 3-7 | 幻觉 (Hallucination)：AI 的胡说八道 | 📝 待编写 |
| Level 3 | 3-8 | Token 计费与成本优化 | 📝 待编写 |
| Level 4 | 4-1 | Prompt 基础：与 AI 对话的正确姿势 | 📝 待编写 |
| Level 4 | 4-2 | Zero-shot vs Few-shot Prompting | 📝 待编写 |
| Level 4 | 4-3 | Chain of Thought：让 AI 一步步思考 | 📝 待编写 |
| Level 4 | 4-4 | System Prompt：给 AI 设定角色 | 📝 待编写 |
| Level 4 | 4-5 | 结构化输出：JSON、表格等 | 📝 待编写 |
| Level 4 | 4-6 | Prompt 高级技巧 | 📝 待编写 |
| Level 4 | 4-7 | 常见 Prompt 模式与模板 | 📝 待编写 |
| Level 5 | 5-1 | 什么是 AI Agent | 📝 待编写 |
| Level 5 | 5-2 | ⭐ Skills / Tools：AI 的手和脚 | ✅ 已发布 |
| Level 5 | 5-3 | Function Calling 底层机制 | 📝 待编写 |
| Level 5 | 5-4 | Plugins 插件系统 | 📝 待编写 |
| Level 5 | 5-5 | ⭐ MCP 协议 | 📝 待编写 |
| Level 5 | 5-6 | Multi-Agent 多智能体协作 | 📝 待编写 |
| Level 5 | 5-7 | Human-in-the-Loop 人机协作 | 📝 待编写 |
| Level 6 | 6-1 | AI API 入门 | 📝 待编写 |
| Level 6 | 6-2 | ⭐ RAG 检索增强生成 | 📝 待编写 |
| Level 6 | 6-3 | 向量数据库 | 📝 待编写 |
| Level 6 | 6-4 | Semantic Kernel / LangChain | 📝 待编写 |
| Level 6 | 6-5 | Fine-tuning 微调 | 📝 待编写 |
| Level 6 | 6-6 | Embedding 嵌入 | 📝 待编写 |
| Level 6 | 6-7 | AI 应用架构设计 | 📝 待编写 |
| Level 7 | 7-1 | 多模态 AI | 📝 待编写 |
| Level 7 | 7-2 | AI 安全与对齐 | 📝 待编写 |
| Level 7 | 7-3 | 负责任的 AI | 📝 待编写 |
| Level 7 | 7-4 | AI 治理与法规 | 📝 待编写 |
| Level 7 | 7-5 | AI 前沿趋势 | 📝 待编写 |

### 6. 参考资料 (References)

- [Microsoft AI Fundamentals Learning Path](https://learn.microsoft.com/en-us/training/modules/get-started-ai-fundamentals/) — 微软官方 AI 基础入门课程，涵盖 AI 概念、生成式 AI、NLP、计算机视觉等
- [Anthropic Claude Documentation](https://docs.anthropic.com/en/docs/overview) — Claude 官方文档，包含模型能力、API 使用和最佳实践
- [Anthropic Prompting Best Practices](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/claude-prompting-best-practices) — Claude 的 Prompt 工程最佳实践指南
- [Semantic Kernel Overview](https://learn.microsoft.com/en-us/semantic-kernel/overview/) — Microsoft 开源 AI 开发框架，用于构建 AI Agent 和集成 LLM
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction) — AI 连接外部系统的开放标准协议

---

## English Version

---

### 1. Overview

This document is a complete **learning roadmap designed for AI beginners**. It organizes the vast AI knowledge landscape into a **progressive knowledge tree**, helping you go from "knowing nothing about AI" to "understanding, using, and building AI applications."

The AI field may seem enormous and complex, but with the right learning sequence, each step is a natural progression. It's like learning to drive — you don't need to be a mechanical engineer first; just learn the steering wheel and gas pedal, then gradually understand how the engine works.

This roadmap divides AI knowledge into **7 levels**, each containing multiple knowledge points. We recommend learning in level order, with each knowledge point having its own detailed Deep Dive article.

### 2. Knowledge Map Overview

```
                    ┌──────────────────────────────────────────┐
                    │    🗺️ AI Beginner Learning Roadmap        │
                    │         (7 Levels)                        │
                    └──────────────────┬───────────────────────┘
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          │                            │                            │
          ▼                            ▼                            ▼
  ┌───────────────┐         ┌──────────────────┐         ┌──────────────────┐
  │ Level 0       │         │ Level 1          │         │ Level 2          │
  │ 🌱 What is AI? │───────▶│ 🧠 How AI Learns  │───────▶│ 🏗️ Deep Learning  │
  │               │         │                  │         │    Basics        │
  │ • Definition  │         │ • Supervised     │         │ • Neural Nets    │
  │ • History     │         │ • Unsupervised   │         │ • CNN (Images)   │
  │ • AI vs ML    │         │ • Reinforcement  │         │ • RNN (Sequence) │
  │   vs DL       │         │ • Train vs Infer │         │ • Transformer ⭐ │
  │ • Capabilities│         │ • Over/Underfit  │         │ • Attention ⭐   │
  └───────────────┘         └──────────────────┘         └──────────────────┘
                                                                  │
                                                                  ▼
  ┌───────────────┐         ┌──────────────────┐         ┌──────────────────┐
  │ Level 5       │         │ Level 4          │         │ Level 3          │
  │ 🤖 AI Agents  │◀────────│ 🎯 Prompt        │◀────────│ 💬 Large Language│
  │   & Skills    │         │   Engineering    │         │    Models (LLM)  │
  │               │         │                  │         │                  │
  │ • Agents      │         │ • Prompt Basics  │         │ • What is LLM    │
  │ • Skills/Tools│         │ • Zero/Few-shot  │         │ • GPT vs Claude  │
  │ • Function    │         │ • Chain of       │         │ • Tokenization   │
  │   Calling     │         │   Thought        │         │ • Training       │
  │ • MCP Protocol│         │ • System Prompt  │         │ • Context Window │
  │ • Multi-Agent │         │ • Advanced Tips  │         │ • Parameters     │
  └───────────────┘         └──────────────────┘         └──────────────────┘
          │
          ▼
  ┌───────────────┐         ┌──────────────────┐
  │ Level 6       │         │ Level 7          │
  │ 🔧 AI Dev     │────────▶│ 🛡️ Advanced      │
  │   Hands-on    │         │   Topics         │
  │               │         │                  │
  │ • AI APIs     │         │ • Multi-modal AI │
  │ • SDKs        │         │ • AI Safety      │
  │ • RAG         │         │ • Responsible AI │
  │ • Vector DBs  │         │ • AI Governance  │
  │ • Fine-tuning │         │ • Future Trends  │
  └───────────────┘         └──────────────────┘
```

### 3. Detailed Knowledge Points

---

#### 🌱 Level 0: What is AI? (Cognitive Foundation)

> **Learning Goal**: Understand AI's basic concepts, build a correct mental framework, and demystify AI.

| # | Knowledge Point | One-line Summary | After Learning, You Can Answer |
|---|----------------|------------------|-------------------------------|
| 0-1 | **AI Definition & History** | From the Turing Test to ChatGPT — 70 years of AI | "What actually IS artificial intelligence?" |
| 0-2 | **AI vs ML vs DL** | Nested relationship: AI > ML > DL | "What's the difference between ML and DL?" |
| 0-3 | **Types of AI** | Narrow AI vs General AI vs Super AI | "Is ChatGPT 'real' AI?" |
| 0-4 | **AI Capabilities** | Text, images, speech, code, decisions... | "What problems can AI currently solve?" |
| 0-5 | **AI Limitations** | What AI cannot do and common misconceptions | "Will AI replace humans? Where are its boundaries?" |

**Key Analogy**:
- **AI** = The big goal of "making machines intelligent"
- **Machine Learning** = One approach to achieve this goal (learning from data)
- **Deep Learning** = The most powerful technique within ML (multi-layer neural networks)
- **LLM** = The latest, hottest application of DL (ChatGPT, Claude, etc.)

```
AI (Artificial Intelligence)
 └── Machine Learning (ML)
      ├── Traditional ML (Decision Trees, SVM, Random Forest...)
      └── Deep Learning (DL)
           ├── CNN (Image Recognition)
           ├── RNN (Speech Recognition)
           └── Transformer (⭐ Core of today's AI revolution)
                └── LLM (Large Language Models: GPT, Claude, Gemini...)
```

---

#### 🧠 Level 1: How AI Learns (Machine Learning Basics)

> **Learning Goal**: Understand ML's core idea — how machines "learn" patterns from data.

| Concept | Human Learning Analogy |
|---------|----------------------|
| Training Data | Textbooks and practice problems |
| Model | The student's brain |
| Training Process | Attending classes + doing exercises |
| Parameters/Weights | Synaptic connections in the brain |
| Inference/Prediction | Taking an exam |
| Overfitting | Can only solve exact same problems, can't generalize |
| Underfitting | Didn't pay attention in class, knows nothing |

---

#### 🏗️ Level 2: Deep Learning Basics (Understanding AI's Engine)

> **Learning Goal**: Understand deep learning's core architectures, especially **Transformer** — the engine of today's AI revolution.

**Why is Transformer the most important?**

In 2017, Google published "Attention Is All You Need," introducing the Transformer architecture that changed the entire AI industry:
- **GPT** (OpenAI) = **G**enerative **P**re-trained **T**ransformer
- **BERT** (Google) = **B**idirectional **E**ncoder **R**epresentations from **T**ransformers
- **Claude** (Anthropic) = Transformer-based large language model

**Without understanding Transformer, you cannot truly understand modern AI.**

---

#### 💬 Level 3: Large Language Models (Today's AI Protagonist)

> **Learning Goal**: Deeply understand how ChatGPT, Claude, and other LLMs work — what they can do, can't do, and why they "hallucinate."

---

#### 🎯 Level 4: Prompt Engineering (The Art of Talking to AI)

> **Learning Goal**: Master writing effective Prompts to get AI to produce the answers you want. This is the **most practical, fastest-impact** skill.

**Why is Prompt Engineering placed before development?**

Because for most people, you **don't need to write code** to leverage AI for massive productivity gains through good prompts. This is the highest ROI AI skill.

---

#### 🤖 Level 5: AI Agents & Skills (Making AI Actually "Do Things")

> **Learning Goal**: Understand how AI evolves from "just talking" to "an intelligent agent that can execute operations."

> 📌 **Note**: You're already experiencing Skills daily! This GitHub Copilot CLI you're using is an AI Agent, and its `grep`, `powershell`, `edit`, `web_fetch`, etc. are all Skills/Tools.

---

#### 🔧 Level 6: AI Development Hands-on (Building AI Applications)

> **Learning Goal**: Learn to use APIs and frameworks to build your own AI applications.

---

#### 🛡️ Level 7: Advanced Topics (Expanding Your Horizon)

> **Learning Goal**: Understand AI's frontier developments and societal impact, forming a comprehensive AI worldview.

---

### 4. Recommended Learning Paths

#### Path A: "AI User" (Fastest Impact ⚡)

For: Anyone wanting to immediately boost work efficiency with AI

```
Level 0 (AI Basic Concepts) → Level 3 (LLM Basics) → Level 4 (Prompt Engineering ⭐) → Level 5 (Agent Concepts)
```

> 💡 **No programming required.** 4-7 days to significantly improve your AI usage ability.

#### Path B: "AI Understander" (Complete Cognition 🧠)

For: Technical professionals wanting systematic AI understanding

```
Level 0 → Level 1 → Level 2 → Level 3 → Level 4 → Level 5
```

> Follow level order, 3-5 days per level, approximately 3-4 weeks total.

#### Path C: "AI Developer" (Hands-on Building 💻)

For: Programmers wanting to develop AI applications

```
Level 0 → Level 1 (quick pass) → Level 3 → Level 4 → Level 5 → Level 6
```

> Focus on Level 5 and Level 6. Programming experience required.

### 5. References

- [Microsoft AI Fundamentals Learning Path](https://learn.microsoft.com/en-us/training/modules/get-started-ai-fundamentals/) — Microsoft's official AI fundamentals course covering AI concepts, generative AI, NLP, and computer vision
- [Anthropic Claude Documentation](https://docs.anthropic.com/en/docs/overview) — Claude official documentation including model capabilities, API usage, and best practices
- [Anthropic Prompting Best Practices](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/claude-prompting-best-practices) — Claude's prompt engineering best practices guide
- [Semantic Kernel Overview](https://learn.microsoft.com/en-us/semantic-kernel/overview/) — Microsoft's open-source AI development framework for building AI Agents and integrating LLMs
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction) — Open standard protocol for connecting AI applications to external systems
