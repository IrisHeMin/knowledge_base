---
layout: post
title: "AI 学习 Level 4: Prompt Engineering — 与 AI 对话的艺术"
date: 2026-03-09
categories: [Knowledge, AI-Fundamentals]
tags: [prompt-engineering, zero-shot, few-shot, chain-of-thought, system-prompt, structured-output, beginner]
type: "deep-dive"
series: "ai-beginner"
series_order: 5
---

# Deep Dive: Prompt Engineering — 与 AI 对话的艺术

**Topic:** Prompt Engineering (Level 4)
**Category:** AI Fundamentals
**Level:** 入门 → 中级
**Series:** AI 小白学习系列
**Last Updated:** 2026-03-09

---

## 中文版

---

### 1. 概述

**Prompt Engineering（提示工程）** 是通过精心设计给 AI 的指令（Prompt），来引导 AI 给出更好回答的技术。它是**投入产出比最高的 AI 技能** —— 不需要写代码，只需要学会"说话"，就能让 AI 的表现提升数倍。

**一句话总结**：同样的 AI 模型，给不同的 Prompt，效果天差地别。Prompt Engineering 就是让你从"会用 AI"到"用好 AI"的关键。

### 2. 核心技巧

#### 2.1 基础原则：清晰、具体、给上下文

**黄金法则**：把你的 Prompt 给一个对任务一无所知的同事看。如果他会困惑，AI 也会困惑。

| 差的 Prompt ❌ | 好的 Prompt ✅ | 为什么好 |
|---------------|--------------|---------|
| "帮我写个邮件" | "用正式商务语气写一封英文邮件给客户 John，告知他项目将延期两周，表示歉意并解释原因是供应链问题" | 明确了语气、语言、对象、内容、原因 |
| "总结这篇文章" | "用 3 个要点总结这篇文章的核心观点，每个要点不超过 2 句话，适合发给不懂技术的老板看" | 明确了格式、长度、受众 |
| "写代码" | "用 Python 写一个函数，输入一个日期字符串（格式 YYYY-MM-DD），返回该日期是星期几。包含错误处理和类型提示" | 明确了语言、输入输出、格式、要求 |

#### 2.2 Zero-shot vs Few-shot

| 方式 | 说明 | 什么时候用 |
|------|------|-----------|
| **Zero-shot** | 不给例子，直接让 AI 做 | 简单任务、AI 已经很擅长的事 |
| **One-shot** | 给 1 个例子 | 需要指定格式或风格 |
| **Few-shot** | 给 3-5 个例子 | 复杂任务、需要一致性 |

```
Few-shot 示例：

请将以下客户反馈分类为"正面"、"负面"或"中性"。

示例：
反馈："产品非常好用，超出预期！" → 正面
反馈："送货太慢了，等了两周" → 负面
反馈："东西收到了，和描述一致" → 中性

现在请分类：
反馈："质量不错，但包装有点简陋" → ?
```

#### 2.3 ⭐ Chain of Thought（思维链）

让 AI **一步一步地思考**，而不是直接给出答案，可以大幅提升推理准确率。

```
❌ 差的方式：
"一个班有 30 人，男生比女生多 4 人，男生有多少？"
AI 可能直接答"17"（对了）或"20"（错了）

✅ 好的方式（加上思维链引导）：
"一个班有 30 人，男生比女生多 4 人，男生有多少？
请一步一步地思考：
1. 先设未知数
2. 列方程
3. 求解"

AI 回答：
1. 设男生 x 人，女生 y 人
2. x + y = 30，x - y = 4
3. 2x = 34，x = 17
男生有 17 人。
```

> 💡 最简单的思维链 Prompt：在问题后面加上 **"Let's think step by step"** 或 **"请一步一步思考"**。

#### 2.4 System Prompt（系统提示）

System Prompt 是给 AI 设定**角色、规则和行为边界**的特殊指令。它在对话开始前就告诉 AI "你是谁、该怎么做"。

```
System Prompt 示例：

"你是一名资深 Windows 系统管理员，专注于 Active Directory 和网络问题排查。
请用以下规则回答：
1. 先确认问题环境（OS 版本、网络拓扑）
2. 给出排查步骤，从最可能的原因开始
3. 每个步骤附上具体命令
4. 如果不确定，明确说出来
5. 用中文回答，技术术语保留英文"
```

#### 2.5 结构化输出

让 AI 输出程序可以直接使用的格式：

```
Prompt：
"分析以下服务器日志，提取所有错误信息，以 JSON 格式输出：
{
  'errors': [
    {
      'timestamp': '时间',
      'level': '错误级别',
      'message': '错误信息',
      'possible_cause': '可能原因'
    }
  ]
}"
```

#### 2.6 高级技巧汇总

| 技巧 | 说明 | 示例 |
|------|------|------|
| **角色扮演** | 让 AI 扮演特定角色 | "你是一个有 20 年经验的网络工程师" |
| **XML 标签分隔** | 用标签区分不同内容 | `<context>...</context><question>...</question>` |
| **反面说明** | 告诉 AI 不要做什么 | "不要编造信息，不确定时说'我不知道'" |
| **分步指令** | 用编号列出步骤 | "步骤 1: ...；步骤 2: ...；步骤 3: ..." |
| **输出约束** | 限制格式和长度 | "用不超过 3 句话回答" |
| **思考空间** | 让 AI 先分析再回答 | "先分析问题的可能原因，再给出建议" |

### 3. Prompt 模板速查

#### 通用分析模板
```
# 角色
你是 [专业角色]。

# 背景
[提供上下文信息]

# 任务
请完成以下任务：[具体任务描述]

# 要求
1. [格式要求]
2. [长度要求]
3. [风格要求]

# 输出格式
[指定输出格式]
```

#### 排查问题模板
```
我遇到了以下问题：
环境：[OS、软件版本、网络拓扑]
症状：[具体错误信息或表现]
已尝试：[已做过的排查步骤]

请帮我：
1. 分析最可能的原因（按可能性从高到低排列）
2. 给出每个原因的排查步骤和命令
3. 如果前面的排查没有结果，给出进一步的方向
```

### 4. 小结 & 下一步

🎉 **Level 4 完成！** 你掌握了：
- ✅ 清晰、具体、给上下文的基础原则
- ✅ Zero-shot / Few-shot 的选择
- ✅ Chain of Thought 思维链推理
- ✅ System Prompt 设定角色
- ✅ 结构化输出和高级技巧

📚 **下一课**：[Level 5] **AI Agent 与 Skills** — 了解 AI 如何从"只会说话"进化为"能做事的智能体"。

### 5. 参考资料

- [Anthropic Prompting Best Practices](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/claude-prompting-best-practices) — Claude Prompt 工程最佳实践
- [Microsoft AI Fundamentals](https://learn.microsoft.com/en-us/training/modules/get-started-ai-fundamentals/) — AI 基础

---

## English Version

---

### 1. Overview

**Prompt Engineering** is the art of crafting instructions (prompts) to guide AI toward better responses. It's the **highest ROI AI skill** — no coding required, just learning to "talk" to AI effectively can multiply its output quality.

### 2. Core Techniques

| Technique | What It Does | When to Use |
|-----------|-------------|-------------|
| **Be Clear & Specific** | Eliminate ambiguity in your instructions | Always — this is the foundation |
| **Few-shot Examples** | Provide 3-5 examples of desired input/output | Complex tasks needing consistency |
| **Chain of Thought** | Ask AI to "think step by step" | Math, logic, complex reasoning |
| **System Prompt** | Set AI's role, rules, and boundaries | When you need consistent behavior |
| **Structured Output** | Request specific format (JSON, table, etc.) | When output feeds into another system |
| **Role Playing** | Assign AI a specific expert persona | Domain-specific tasks |

### 3. Universal Prompt Template

```
# Role: You are [expert role].
# Context: [background information]
# Task: [specific task]
# Requirements: [format, length, style constraints]
# Output Format: [desired format]
```

### 4. Summary & What's Next

📚 **Next**: [Level 5] **AI Agents & Skills** — How AI evolves from "just talking" to "actually doing things."

### 5. References

- [Anthropic Prompting Best Practices](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/claude-prompting-best-practices) — Claude prompt engineering guide
- [Microsoft AI Fundamentals](https://learn.microsoft.com/en-us/training/modules/get-started-ai-fundamentals/) — AI fundamentals
