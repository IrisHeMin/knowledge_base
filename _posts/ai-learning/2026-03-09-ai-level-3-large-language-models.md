---
layout: post
title: "AI 学习 Level 3: 大语言模型 LLM — ChatGPT 和 Claude 是怎么工作的？"
date: 2026-03-09
categories: [Knowledge, AI-Fundamentals]
tags: [llm, large-language-model, gpt, claude, gemini, tokenization, context-window, hallucination, temperature, beginner]
type: "deep-dive"
series: "ai-beginner"
series_order: 4
---

# Deep Dive: 大语言模型 LLM — ChatGPT 和 Claude 是怎么工作的？

**Topic:** Large Language Models (Level 3)
**Category:** AI Fundamentals
**Level:** 入门
**Series:** AI 小白学习系列
**Last Updated:** 2026-03-09

---

## 中文版

---

### 1. 概述

**大语言模型（Large Language Model，LLM）** 就是 ChatGPT、Claude、Gemini 背后的技术。它是在**海量文本数据**上训练的**超大规模 Transformer 模型**，能够理解和生成人类语言。

LLM 的核心原理出奇地简单：**预测下一个词（Next Token Prediction）**。当你问 ChatGPT 一个问题时，它不是在"思考答案"，而是在一个词一个词地预测"最可能出现的下一个词"。但当这个能力被训练到极致、模型规模足够大时，就涌现出了令人惊叹的"智能"。

### 2. 核心概念

#### 2.1 LLM 的本质：超级自动补全

你手机输入法的「自动补全」功能就是 LLM 的微缩版：

```
普通自动补全：                     LLM 的"自动补全"：
"今天天气" → "真好"               "解释一下量子力学" → [生成一篇完整的、
                                  逻辑清晰的量子力学科普文章]

区别：手机补全只看前几个词            LLM 看整个上下文（几万到几十万词）
     靠简单统计                      靠万亿参数的深度理解
```

> 💡 **关键认知**：LLM 不是在"理解"你的问题，它是在做极其复杂的"下一个词是什么"的概率预测。但这个预测能力强到可以模拟出推理、创作、编程等"智能"行为。

#### 2.2 主流 LLM 对比

| 模型 | 开发商 | 特点 | 适合场景 |
|------|--------|------|---------|
| **GPT-4/5** | OpenAI | 综合能力强，生态最完善 | 通用对话、内容创作 |
| **Claude Opus/Sonnet** | Anthropic | 推理能力强，编码优秀，安全性高 | 编程、深度分析、企业应用 |
| **Gemini** | Google | 多模态强，与 Google 生态集成 | 搜索增强、多模态任务 |
| **LLaMA** | Meta | 开源，可本地部署 | 研究、定制化、隐私敏感场景 |
| **Qwen** | 阿里巴巴 | 中文能力强，开源 | 中文场景、国内部署 |

#### 2.3 Tokenization：AI 的阅读方式

LLM 不直接读取文字，它读的是 **Token**（词元）。Tokenization 就是把文本切分成 Token 的过程。

```
英文："Hello, how are you?" → ["Hello", ",", " how", " are", " you", "?"]
                              6 个 tokens

中文："你好世界" → ["你", "好", "世", "界"]  或  ["你好", "世界"]
                 取决于 tokenizer，中文通常每个字 1-2 个 token

代码："print('hello')" → ["print", "('", "hello", "')"]
                        4 个 tokens
```

**为什么 Token 很重要？**

| 影响 | 说明 |
|------|------|
| **费用** | API 按 Token 数量计费。中文通常比英文消耗更多 Token |
| **上下文限制** | 上下文窗口以 Token 计算，不是以字数计算 |
| **速度** | Token 越多，生成越慢 |

#### 2.4 LLM 的训练三阶段

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│  ① Pre-training  │────▶│  ② Fine-tuning   │────▶│  ③ RLHF / RLAIF    │
│  预训练           │     │  监督微调 (SFT)    │     │  人类反馈强化学习     │
│                  │     │                  │     │                     │
│ 读完互联网上的     │     │ 用人工编写的高     │     │ 人类对 AI 的多个     │
│ 几万亿词的文本     │     │ 质量对话示例进     │     │ 回答进行排名，AI     │
│                  │     │ 行训练            │     │ 据此学习人类偏好      │
│ 学到：语言规律     │     │ 学到：如何对话     │     │ 学到：什么是"好"     │
│ 常识、世界知识     │     │ 如何按指令行事     │     │ 的回答               │
│                  │     │                  │     │                     │
│ 成本：数千万美元   │     │ 成本：几十万美元   │     │ 成本：持续投入        │
└─────────────────┘     └──────────────────┘     └─────────────────────┘

结果：一个百科全书式      结果：一个能对话的       结果：一个有礼貌、
的"知识库"（但不会        助手（但可能说错话、     安全、有用的 AI 助手
对话）                   不太安全）              （ChatGPT/Claude）
```

#### 2.5 上下文窗口（Context Window）

上下文窗口是 LLM 一次能"看到"的最大文本量，相当于 AI 的**短期记忆容量**。

| 模型 | 上下文窗口 | 大约等于 |
|------|-----------|---------|
| GPT-3 (2020) | 4K tokens | ~3,000 字 |
| GPT-4 (2023) | 128K tokens | ~100,000 字（一本小说） |
| Claude Opus 4.6 (2026) | 200K tokens（标准）/ 1M tokens（扩展） | ~150,000 字 / ~750,000 字 |

**超出上下文窗口会怎样？** 最早的内容会被"遗忘"。这就是为什么很长的对话中，AI 可能忘记你开头说的话。

#### 2.6 Temperature 等推理参数

| 参数 | 作用 | 值低 | 值高 |
|------|------|------|------|
| **Temperature** | 控制"创造力" | 确定性强、重复性高（适合代码、事实） | 创造性强、多样性高（适合写作、头脑风暴） |
| **Top-p** | 控制候选词范围 | 只考虑最可能的词 | 考虑更多可能的词 |
| **Max Tokens** | 限制输出长度 | 短回答 | 长回答 |

```
Temperature = 0.0:  "法国的首都是巴黎。"  (每次都一样)
Temperature = 0.7:  "法国的首都是巴黎，这座浪漫的城市..." (有变化)
Temperature = 1.5:  "法国的首都是巴黎，光之城在塞纳河畔舞蹈..." (更有创意但可能偏离)
```

#### 2.7 幻觉（Hallucination）

**幻觉**是 LLM 最大的问题之一：**AI 一本正经地编造不存在的信息**。

**为什么会幻觉？** 因为 LLM 的本质是"预测下一个最可能的词"，不是"查找真实信息"。当它不确定时，它不会说"我不知道"，而是会自信地给出一个"看起来合理"的答案 —— 即使这个答案是编造的。

| 幻觉类型 | 例子 |
|---------|------|
| 编造事实 | "爱因斯坦在 1921 年获得数学诺贝尔奖"（不存在数学诺贝尔奖） |
| 编造引用 | 生成看似真实但完全虚构的论文标题和 DOI |
| 编造链接 | 给出格式正确但不存在的 URL |
| 时间错乱 | 混淆不同时期的事件 |

**如何应对幻觉？**
1. 对事实性信息**始终验证**
2. 使用 **RAG**（检索增强生成）让 AI 基于真实文档回答
3. 降低 Temperature 减少"创造性发挥"
4. 明确要求 AI "如果不确定就说不知道"

### 3. 小结 & 下一步

🎉 **Level 3 完成！** 你现在理解了：
- ✅ LLM 的本质是"超级自动补全" — 预测下一个 Token
- ✅ 主流 LLM 的区别和选择
- ✅ Tokenization 及其对费用和性能的影响
- ✅ LLM 训练的三个阶段
- ✅ 上下文窗口 = AI 的短期记忆
- ✅ Temperature 控制 AI 的"创造力"
- ✅ 幻觉问题及应对方法

📚 **下一课**：[Level 4] **Prompt Engineering** — 学会如何写出好的 Prompt，这是你用好 AI 最关键的实战技能！

### 4. 参考资料

- [How Generative AI and LLMs Work](https://learn.microsoft.com/en-us/dotnet/ai/conceptual/how-genai-and-llms-work) — 生成式 AI 和 LLM 的工作原理
- [Anthropic Claude Models Overview](https://docs.anthropic.com/en/docs/about-claude/models/overview) — Claude 模型家族对比

---

## English Version

---

### 1. Overview

**Large Language Models (LLMs)** are the technology behind ChatGPT, Claude, and Gemini. They are **massive Transformer models** trained on **enormous amounts of text data** that can understand and generate human language.

The core principle is surprisingly simple: **Next Token Prediction**. When you ask ChatGPT a question, it's not "thinking about the answer" — it's predicting "the most likely next word" one token at a time. But when this capability is trained to the extreme at sufficient scale, stunning "intelligence" emerges.

### 2. Core Concepts

#### The Essence: Super Auto-Complete

Your phone's autocomplete is a miniature version of what LLMs do. The difference: your phone looks at a few words with simple statistics; LLMs look at the entire context (tens to hundreds of thousands of words) with trillions of parameters.

#### Training Pipeline

| Phase | What Happens | Result |
|-------|-------------|--------|
| **① Pre-training** | Read trillions of words from the internet | Encyclopedia-like knowledge base (can't chat yet) |
| **② Fine-tuning (SFT)** | Train on high-quality dialogue examples | A conversational assistant (may say unsafe things) |
| **③ RLHF** | Humans rank AI responses; AI learns preferences | A polite, safe, helpful AI assistant |

#### Key Parameters

| Parameter | What It Controls | Low Value | High Value |
|-----------|-----------------|-----------|-----------|
| **Temperature** | "Creativity" | Deterministic, repetitive (code, facts) | Creative, diverse (writing, brainstorming) |
| **Context Window** | "Memory capacity" | Short conversations only | Can process entire books |

#### Hallucination: AI's Biggest Problem

LLMs confidently fabricate non-existent information because they predict "likely words," not "true facts." Always verify factual claims. Use RAG to ground AI in real documents.

### 3. Summary & What's Next

📚 **Next**: [Level 4] **Prompt Engineering** — The most practical skill for using AI effectively.

### 4. References

- [How Generative AI and LLMs Work](https://learn.microsoft.com/en-us/dotnet/ai/conceptual/how-genai-and-llms-work) — How generative AI and LLMs work
- [Anthropic Claude Models Overview](https://docs.anthropic.com/en/docs/about-claude/models/overview) — Claude model comparison
