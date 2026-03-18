---
layout: post
title: "AI 行业 Top 50 术语词典 + Agentic AI 深度问答"
date: 2026-03-18
categories: [Knowledge, AI-Fundamentals]
tags: [ai-glossary, terminology, agentic-ai, agent, llm, deep-learning, machine-learning, beginner, intermediate]
type: "deep-dive"
series: "ai-beginner"
series_order: 8
---

# AI 行业 Top 50 术语词典 + Agentic AI 深度问答

**Topic:** AI Industry Top 50 Glossary & Agentic AI Q&A  
**Category:** AI Fundamentals  
**Level:** 入门 → 中级  
**Last Updated:** 2026-03-18

---

## 中文版

---

### 第一部分：AI 行业 Top 50 核心术语

> 以下术语按从基础到进阶的顺序排列，涵盖 AI 行业最常用的 50 个概念。

---

#### 🔵 基础概念（1-10）

| # | 英文术语 | 中文术语 | 解释 |
|---|---------|---------|------|
| 1 | **Artificial Intelligence (AI)** | 人工智能 | 让机器模拟人类智能（如学习、推理、决策）的技术总称。 |
| 2 | **Machine Learning (ML)** | 机器学习 | AI 的子领域，让机器通过数据自动学习规律，而不是靠人写规则。 |
| 3 | **Deep Learning (DL)** | 深度学习 | 机器学习的子领域，使用多层神经网络来处理复杂模式（如图像、语音）。 |
| 4 | **Neural Network (NN)** | 神经网络 | 模仿人脑神经元结构的计算模型，由多层节点组成，层层传递和处理信息。 |
| 5 | **Algorithm** | 算法 | 解决特定问题的一组明确步骤和规则，是 AI 模型的数学基础。 |
| 6 | **Training** | 训练 | 用大量数据让模型学习规律和模式的过程，类比人类"刷题"。 |
| 7 | **Inference** | 推理 | 训练好的模型在新数据上做预测/生成的过程，类比"考试"。 |
| 8 | **Dataset** | 数据集 | 用于训练和评估模型的结构化数据集合。 |
| 9 | **Model** | 模型 | 从数据中学到的"知识"的数学表示，能对新输入做出预测。 |
| 10 | **Parameter** | 参数 | 模型内部的可学习变量，参数越多模型越大（如 GPT-4 有万亿级参数）。 |

#### 🟢 模型与架构（11-20）

| # | 英文术语 | 中文术语 | 解释 |
|---|---------|---------|------|
| 11 | **Large Language Model (LLM)** | 大语言模型 | 在海量文本上训练的超大规模语言模型，如 GPT-4、Claude、Gemini。 |
| 12 | **Transformer** | Transformer 架构 | 2017 年 Google 提出的神经网络架构，是 GPT/BERT/LLM 的核心基础。 |
| 13 | **Attention Mechanism** | 注意力机制 | Transformer 的核心组件，让模型能"关注"输入中最重要的部分。 |
| 14 | **Foundation Model** | 基础模型 | 在大规模数据上预训练的通用大模型，可适配各种下游任务。 |
| 15 | **Pre-training** | 预训练 | 在大量无标注数据上训练模型学习通用知识的第一阶段。 |
| 16 | **Fine-tuning** | 微调 | 在预训练模型基础上，用特定领域数据进一步训练以适应特定任务。 |
| 17 | **Transfer Learning** | 迁移学习 | 将一个任务上学到的知识应用到另一个相关任务，减少训练成本。 |
| 18 | **Embedding** | 嵌入/向量表示 | 将文本/图像等转换为高维数字向量，使计算机能理解语义相似性。 |
| 19 | **Tokenization** | 分词/Token 化 | 将文本切分成模型能处理的最小单元（token），如"Hello"→["Hel","lo"]。 |
| 20 | **Diffusion Model** | 扩散模型 | 通过逐步去噪生成图像/视频的模型架构，如 Stable Diffusion、DALL-E。 |

#### 🟡 生成式 AI 与交互（21-30）

| # | 英文术语 | 中文术语 | 解释 |
|---|---------|---------|------|
| 21 | **Generative AI (GenAI)** | 生成式 AI | 能创造新内容（文本、图像、代码、音频）的 AI 系统。 |
| 22 | **Prompt** | 提示词 | 用户给 AI 的输入指令，AI 根据 prompt 生成响应。 |
| 23 | **Prompt Engineering** | 提示工程 | 设计和优化 prompt 以获得更好 AI 输出的技术和方法论。 |
| 24 | **Context Window** | 上下文窗口 | 模型一次能处理的最大 token 数量（如 128K tokens）。 |
| 25 | **Token** | Token | LLM 处理文本的基本单位，约等于 0.75 个英文单词或 0.5 个中文字。 |
| 26 | **Temperature** | 温度 | 控制 AI 输出随机性的参数：0=确定性高，1=创造性强。 |
| 27 | **Hallucination** | 幻觉 | AI 生成看似合理但实际上错误或虚构的内容，是 LLM 的核心挑战。 |
| 28 | **Grounding** | 接地/落地 | 将 AI 输出与可靠数据源关联，减少幻觉的技术。 |
| 29 | **Chain-of-Thought (CoT)** | 思维链 | 让 AI 逐步推理而非直接给答案的提示技术，提高复杂问题的准确性。 |
| 30 | **Few-shot / Zero-shot Learning** | 少样本/零样本学习 | 模型只需少量甚至零个示例就能完成新任务的能力。 |

#### 🟠 Agent 与工具（31-40）

| # | 英文术语 | 中文术语 | 解释 |
|---|---------|---------|------|
| 31 | **Agentic AI** | 智能体 AI | 能自主规划、决策、使用工具并迭代完成复杂任务的 AI 系统。 |
| 32 | **AI Agent** | AI 智能体 | 具备感知-思考-行动循环能力的自主 AI 程序。 |
| 33 | **Function Calling** | 函数调用 | LLM 输出结构化的工具调用指令，由外部代码执行真实操作。 |
| 34 | **Tool Use** | 工具使用 | AI Agent 调用外部工具（搜索、API、数据库等）来扩展能力。 |
| 35 | **ReAct (Reasoning + Acting)** | 推理+行动 | Agent 的核心模式：思考→行动→观察→再思考，循环直到完成。 |
| 36 | **Multi-Agent System** | 多智能体系统 | 多个专业 Agent 分工协作完成复杂任务的系统架构。 |
| 37 | **Orchestration** | 编排 | 协调多个 AI 组件/Agent 协同工作的管理层。 |
| 38 | **MCP (Model Context Protocol)** | 模型上下文协议 | Anthropic 提出的开放协议，标准化 AI 与外部工具/数据的连接方式。 |
| 39 | **Plugin / Skill** | 插件/技能 | 为 AI 添加特定能力的可扩展模块（如搜索网页、操作数据库）。 |
| 40 | **Human-in-the-Loop** | 人机协同 | AI 在关键决策点请求人类审核和确认的安全机制。 |

#### 🔴 高级概念与应用（41-50）

| # | 英文术语 | 中文术语 | 解释 |
|---|---------|---------|------|
| 41 | **RAG (Retrieval-Augmented Generation)** | 检索增强生成 | 先从知识库检索相关信息，再让 LLM 基于检索结果生成回答，减少幻觉。 |
| 42 | **Vector Database** | 向量数据库 | 专门存储和检索 Embedding 向量的数据库，是 RAG 的核心组件。 |
| 43 | **RLHF (Reinforcement Learning from Human Feedback)** | 人类反馈强化学习 | 用人类偏好反馈来优化模型输出的训练方法，让 AI 更符合人类期望。 |
| 44 | **Alignment** | 对齐 | 确保 AI 的行为和目标与人类价值观一致的研究方向。 |
| 45 | **Responsible AI** | 负责任的 AI | 确保 AI 系统公平、透明、安全、可解释的设计原则和实践。 |
| 46 | **AGI (Artificial General Intelligence)** | 通用人工智能 | 具备人类级别通用智能的 AI，能处理任何智力任务（尚未实现）。 |
| 47 | **Edge AI** | 边缘 AI | 在本地设备（手机、IoT）上运行的 AI，无需云端，低延迟高隐私。 |
| 48 | **Multimodal AI** | 多模态 AI | 能同时处理多种输入类型（文本+图像+语音+视频）的 AI 模型。 |
| 49 | **Synthetic Data** | 合成数据 | 由 AI 生成的用于训练其他模型的人造数据，解决数据稀缺和隐私问题。 |
| 50 | **AI Governance** | AI 治理 | 管理 AI 开发和使用的政策、流程和法规框架。 |

---

### 第二部分：Agentic AI 深度问答（Session 记录）

> 以下是一次关于 Agentic AI 的完整问答记录，从概念理解到实现原理再到实践路径。

---

#### Q1: 什么是 Agentic AI？

**Agentic AI**（智能体 AI）是指能够**自主感知环境、制定计划、做出决策并采取行动**来完成目标的 AI 系统。

与传统 AI（被动响应单次提示）不同，Agentic AI 的核心特征包括：

- **自主性**：能独立分解复杂任务并逐步执行
- **规划能力**：制定多步骤策略来达成目标
- **工具使用**：调用外部工具（搜索、代码执行、API 等）
- **反思与迭代**：评估结果，遇到失败时调整策略重试
- **持续交互**：在多轮循环中与环境互动，而非一次性输出

**简单类比**：传统 AI 像一个回答问题的助手，Agentic AI 像一个能独立完成项目的员工——你给目标，它自己想办法做到。

---

#### Q2: Agentic AI 具体是怎么实现的？

核心是一个 **循环架构（Agent Loop）**，基于 **ReAct 模式**：

```
用户目标 → 思考(Reason) → 行动(Act) → 观察结果(Observe) → 再思考 → ... → 完成
```

**关键组件：**

| 组件 | 作用 | 实现方式 |
|------|------|----------|
| **LLM 大脑** | 理解、推理、决策 | GPT/Claude 等大模型 |
| **工具系统** | 与外界交互 | Function Calling / Tool Use API |
| **记忆** | 维持上下文 | 对话历史 + 向量数据库 |
| **规划器** | 分解复杂任务 | Prompt 引导 / CoT / 任务图 |
| **反思机制** | 自我纠错 | 检查输出、重试策略 |

**技术要点：**
- **工具调用**：LLM 输出结构化的函数调用指令，运行时执行后将结果返回给 LLM
- **多 Agent 协作**：多个专业 Agent 分工，通过消息传递协同
- **编排框架**：LangChain、AutoGen、CrewAI 等封装了上述模式

---

#### Q3: Agentic AI 是一个有代码写出来的软件吗？举个实现例子？

**是的，本质就是用代码写出来的软件。** 核心是一个循环程序 + LLM API 调用：

```python
def agent_loop(user_request):
    messages = [{"role": "user", "content": user_request}]
    
    while True:
        # 1. 调用 LLM API
        response = call_llm(messages, tools=available_tools)
        
        # 2. 任务完成 → 返回答案
        if response.is_final_answer:
            return response.text
        
        # 3. 需要调用工具 → 执行工具
        for tool_call in response.tool_calls:
            result = execute_tool(tool_call)
            messages.append({"role": "tool", "content": result})
        
        # 4. 工具结果喂回 LLM → 回到第1步
```

**具体例子**——用户说"帮我写一篇 post"时的执行流程：

```
第1轮: LLM 决定调用 skill → 启动 knowledge-deep-dive
第2轮: LLM 调用 glob 了解文件结构 → 返回文件列表
第3轮: LLM 生成内容 → 调用 create 创建 markdown 文件
第4轮: LLM 调用 powershell → git add && commit && push
第5轮: LLM 判断完成 → 返回最终答案，循环结束
```

**Agent = 普通代码（while 循环 + 工具）+ LLM API（大脑），没有魔法。**

---

#### Q4: 我如果自己想创建一个 Agentic AI，可行吗？

**完全可行！** 门槛比想象的低。你需要准备：

| 需要 | 说明 | 成本 |
|------|------|------|
| **LLM API** | OpenAI / Azure OpenAI / Claude API | 按量付费 |
| **定义工具** | JSON Schema 描述可用函数 | 写配置 |
| **工具逻辑** | 真正执行操作的代码 | Python/JS 函数 |

**三种路径：**
- 🟢 **入门**（1天）：直接用 API 的 Function Calling + while 循环
- 🟡 **进阶**（几天）：用框架 LangChain / Semantic Kernel / AutoGen
- 🔵 **高级**（持续迭代）：多 Agent 协作系统

---

#### Q5: 不会代码，还能怎么实现？

**零代码也完全可以！**

| 平台 | 特点 | 难度 |
|------|------|------|
| **Microsoft Copilot Studio** | 微软官方，集成 M365/Azure | ⭐ 最简单 |
| **Dify.ai** | 开源，可视化拖拽编排 | ⭐⭐ |
| **Coze（扣子）** | 字节跳动，中文友好 | ⭐ |
| **GPTs（ChatGPT）** | ChatGPT 内配置自定义 Agent | ⭐ 最简单 |
| **Power Automate** | 低代码自动化流程 + AI | ⭐⭐ |

以 Copilot Studio 为例，全程不写一行代码：
1. 打开 Copilot Studio（浏览器）
2. 新建 Agent → 自然语言描述角色
3. 添加知识源（文档/SharePoint）
4. 添加动作（发邮件/创建工单）
5. 测试 → 发布

---

### 第三部分：关键要点总结

> **Agentic AI 的本质：** LLM（大脑） + 工具（手脚） + 循环（行为模式）

```
┌─────────────────────────────────────────────────┐
│              Agentic AI 核心公式                  │
│                                                  │
│   Agent = while 循环 + LLM API + 工具函数         │
│                                                  │
│   每一轮:                                        │
│   用户目标 → LLM思考 → 调用工具 → 观察结果 → 继续  │
│                                                  │
│   实现门槛:                                       │
│   会代码 → 几十行 Python 即可                      │
│   不会代码 → Copilot Studio / Dify / GPTs         │
└─────────────────────────────────────────────────┘
```

---
---

## English Version

---

### Part 1: Top 50 AI Industry Terms

> The following terms are arranged from foundational to advanced, covering the 50 most essential concepts in the AI industry.

---

#### 🔵 Foundational Concepts (1-10)

| # | Term | Definition |
|---|------|-----------|
| 1 | **Artificial Intelligence (AI)** | The broad field of creating machines that simulate human intelligence — learning, reasoning, and decision-making. |
| 2 | **Machine Learning (ML)** | A subset of AI where machines learn patterns from data automatically, rather than following hand-coded rules. |
| 3 | **Deep Learning (DL)** | A subset of ML using multi-layered neural networks to handle complex patterns like images and speech. |
| 4 | **Neural Network (NN)** | A computing model inspired by the human brain, consisting of interconnected layers of nodes that process information. |
| 5 | **Algorithm** | A set of well-defined steps and rules for solving a specific problem — the mathematical foundation of AI models. |
| 6 | **Training** | The process of feeding large amounts of data to a model so it learns patterns — like a student studying with practice problems. |
| 7 | **Inference** | Using a trained model to make predictions on new data — like taking an exam after studying. |
| 8 | **Dataset** | A structured collection of data used to train and evaluate models. |
| 9 | **Model** | A mathematical representation of learned "knowledge" from data, capable of making predictions on new inputs. |
| 10 | **Parameter** | Learnable variables inside a model. More parameters = larger model (e.g., GPT-4 has trillions of parameters). |

#### 🟢 Models & Architecture (11-20)

| # | Term | Definition |
|---|------|-----------|
| 11 | **Large Language Model (LLM)** | Massive language models trained on enormous text corpora, such as GPT-4, Claude, and Gemini. |
| 12 | **Transformer** | A neural network architecture proposed by Google in 2017 — the core foundation of GPT, BERT, and all modern LLMs. |
| 13 | **Attention Mechanism** | The core component of Transformers that allows the model to "focus" on the most relevant parts of the input. |
| 14 | **Foundation Model** | A large model pre-trained on massive data that can be adapted for various downstream tasks. |
| 15 | **Pre-training** | The first phase of training where the model learns general knowledge from large-scale unlabeled data. |
| 16 | **Fine-tuning** | Further training a pre-trained model on domain-specific data to adapt it for a particular task. |
| 17 | **Transfer Learning** | Applying knowledge learned from one task to a related task, reducing training costs significantly. |
| 18 | **Embedding** | Converting text, images, etc. into high-dimensional numeric vectors so computers can understand semantic similarity. |
| 19 | **Tokenization** | Splitting text into minimal units (tokens) that a model can process, e.g., "Hello" → ["Hel", "lo"]. |
| 20 | **Diffusion Model** | A model architecture that generates images/videos through iterative denoising, e.g., Stable Diffusion, DALL-E. |

#### 🟡 Generative AI & Interaction (21-30)

| # | Term | Definition |
|---|------|-----------|
| 21 | **Generative AI (GenAI)** | AI systems that can create new content — text, images, code, audio, and video. |
| 22 | **Prompt** | The input instruction given to an AI model, which generates a response based on the prompt. |
| 23 | **Prompt Engineering** | The art and science of designing and optimizing prompts to get better AI outputs. |
| 24 | **Context Window** | The maximum number of tokens a model can process at once (e.g., 128K tokens). |
| 25 | **Token** | The basic unit LLMs use to process text — approximately 0.75 English words or 0.5 Chinese characters. |
| 26 | **Temperature** | A parameter controlling output randomness: 0 = deterministic, 1 = creative. |
| 27 | **Hallucination** | When AI generates plausible-sounding but factually incorrect or fabricated content — a core LLM challenge. |
| 28 | **Grounding** | Connecting AI outputs to reliable data sources to reduce hallucinations. |
| 29 | **Chain-of-Thought (CoT)** | A prompting technique that makes AI reason step-by-step rather than jumping to answers, improving accuracy. |
| 30 | **Few-shot / Zero-shot Learning** | The model's ability to complete new tasks with few or zero examples. |

#### 🟠 Agents & Tools (31-40)

| # | Term | Definition |
|---|------|-----------|
| 31 | **Agentic AI** | AI systems capable of autonomously planning, deciding, using tools, and iterating to complete complex tasks. |
| 32 | **AI Agent** | An autonomous AI program with a perceive-think-act loop capability. |
| 33 | **Function Calling** | LLM outputs structured tool invocation instructions that external code executes for real-world actions. |
| 34 | **Tool Use** | AI Agents calling external tools (search, APIs, databases) to extend their capabilities. |
| 35 | **ReAct (Reasoning + Acting)** | The core Agent pattern: Think → Act → Observe → Think again, looping until task completion. |
| 36 | **Multi-Agent System** | An architecture where multiple specialized Agents collaborate to complete complex tasks. |
| 37 | **Orchestration** | The management layer coordinating multiple AI components/Agents to work together. |
| 38 | **MCP (Model Context Protocol)** | An open protocol by Anthropic that standardizes how AI connects with external tools and data. |
| 39 | **Plugin / Skill** | Extensible modules that add specific capabilities to AI (e.g., web search, database operations). |
| 40 | **Human-in-the-Loop** | A safety mechanism where AI requests human review and confirmation at critical decision points. |

#### 🔴 Advanced Concepts & Applications (41-50)

| # | Term | Definition |
|---|------|-----------|
| 41 | **RAG (Retrieval-Augmented Generation)** | Retrieves relevant information from a knowledge base first, then has the LLM generate answers based on retrieved results. |
| 42 | **Vector Database** | A specialized database for storing and searching Embedding vectors — a core RAG component. |
| 43 | **RLHF (Reinforcement Learning from Human Feedback)** | A training method that uses human preference feedback to optimize model outputs for better alignment. |
| 44 | **Alignment** | Research ensuring AI behavior and goals are consistent with human values and intentions. |
| 45 | **Responsible AI** | Design principles and practices ensuring AI systems are fair, transparent, safe, and explainable. |
| 46 | **AGI (Artificial General Intelligence)** | Human-level general intelligence capable of any intellectual task — not yet achieved. |
| 47 | **Edge AI** | AI running on local devices (phones, IoT) without cloud dependency — low latency, high privacy. |
| 48 | **Multimodal AI** | AI models that process multiple input types simultaneously (text + images + audio + video). |
| 49 | **Synthetic Data** | AI-generated artificial data used to train other models, solving data scarcity and privacy issues. |
| 50 | **AI Governance** | Policy, process, and regulatory frameworks for managing AI development and deployment. |

---

### Part 2: Agentic AI Deep-Dive Q&A (Session Record)

> A complete Q&A session covering Agentic AI from concept to implementation to practical paths.

---

#### Q1: What is Agentic AI?

**Agentic AI** refers to AI systems that can **autonomously perceive their environment, make plans, make decisions, and take actions** to achieve goals.

Unlike traditional AI (passively responding to single prompts), Agentic AI features:

- **Autonomy**: Independently decompose complex tasks and execute step by step
- **Planning**: Create multi-step strategies to achieve goals
- **Tool Use**: Call external tools (search, code execution, APIs, etc.)
- **Reflection & Iteration**: Evaluate results, adjust strategy when failures occur
- **Continuous Interaction**: Interact with the environment in multi-turn loops

**Analogy**: Traditional AI is like an assistant who answers questions. Agentic AI is like an employee who can independently complete projects — you give the goal, it figures out how.

---

#### Q2: How is Agentic AI implemented technically?

The core is a **loop architecture (Agent Loop)** based on the **ReAct pattern**:

```
User Goal → Reason → Act → Observe → Reason again → ... → Complete
```

**Key Components:**

| Component | Role | Implementation |
|-----------|------|---------------|
| **LLM Brain** | Understanding, reasoning, decisions | GPT/Claude models |
| **Tool System** | Interact with the real world | Function Calling / Tool Use API |
| **Memory** | Maintain context | Conversation history + Vector DB |
| **Planner** | Decompose complex tasks | Prompt guidance / CoT / Task graphs |
| **Reflection** | Self-correction | Output checking, retry strategies |

**Technical highlights:**
- **Tool Calling**: LLM outputs structured function call instructions; runtime executes them and returns results
- **Multi-Agent Collaboration**: Specialized Agents divide work, communicate via messages
- **Orchestration Frameworks**: LangChain, AutoGen, CrewAI encapsulate these patterns

---

#### Q3: Is Agentic AI software written in code? Can you give a concrete example?

**Yes, it's fundamentally software written in code.** The core is a loop program + LLM API calls:

```python
def agent_loop(user_request):
    messages = [{"role": "user", "content": user_request}]
    
    while True:
        response = call_llm(messages, tools=available_tools)
        
        if response.is_final_answer:
            return response.text
        
        for tool_call in response.tool_calls:
            result = execute_tool(tool_call)
            messages.append({"role": "tool", "content": result})
```

**Concrete example** — when a user says "write a post":

```
Round 1: LLM decides to invoke skill → starts knowledge-deep-dive
Round 2: LLM calls glob to understand file structure → returns file list
Round 3: LLM generates content → calls create to make markdown files
Round 4: LLM calls powershell → git add && commit && push
Round 5: LLM determines completion → returns final answer, loop ends
```

**Agent = Normal code (while loop + tools) + LLM API (brain). No magic.**

---

#### Q4: Can I create my own Agentic AI?

**Absolutely! The barrier is lower than you think.**

| What You Need | Description | Cost |
|-------------|-------------|------|
| **LLM API** | OpenAI / Azure OpenAI / Claude API | Pay-per-use |
| **Tool Definitions** | JSON Schema describing available functions | Configuration |
| **Tool Logic** | Code that actually performs operations | Python/JS functions |

**Three paths:**
- 🟢 **Beginner** (1 day): API Function Calling + while loop
- 🟡 **Intermediate** (a few days): Frameworks like LangChain / Semantic Kernel / AutoGen
- 🔵 **Advanced** (ongoing): Multi-Agent collaboration systems

---

#### Q5: What if I don't know how to code?

**Zero-code options are absolutely available!**

| Platform | Features | Difficulty |
|----------|----------|-----------|
| **Microsoft Copilot Studio** | Microsoft official, integrates with M365/Azure | ⭐ Easiest |
| **Dify.ai** | Open source, visual drag-and-drop Agent builder | ⭐⭐ |
| **Coze** | ByteDance, generous free tier | ⭐ |
| **GPTs (ChatGPT)** | Configure custom Agents inside ChatGPT | ⭐ Easiest |
| **Power Automate** | Low-code automation workflows + AI | ⭐⭐ |

Example with Copilot Studio — zero code required:
1. Open Copilot Studio (browser)
2. Create Agent → describe role in natural language
3. Add knowledge sources (documents/SharePoint)
4. Add actions (send email/create ticket)
5. Test → Publish

---

### Part 3: Key Takeaways

> **The essence of Agentic AI:** LLM (brain) + Tools (hands) + Loop (behavior pattern)

```
┌─────────────────────────────────────────────────────┐
│              Agentic AI Core Formula                 │
│                                                      │
│   Agent = while loop + LLM API + Tool functions      │
│                                                      │
│   Each round:                                        │
│   Goal → LLM thinks → Call tools → Observe → Continue│
│                                                      │
│   Implementation barrier:                            │
│   Can code → A few dozen lines of Python             │
│   Can't code → Copilot Studio / Dify / GPTs          │
└─────────────────────────────────────────────────────┘
```
