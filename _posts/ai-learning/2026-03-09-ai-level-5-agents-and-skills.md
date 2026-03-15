---
layout: post
title: "AI 学习 Level 5: AI Agent 与 Skills — 让 AI 从「能说」到「能做」"
date: 2026-03-09
categories: [Knowledge, AI-Fundamentals]
tags: [ai-agent, skills, tools, function-calling, mcp, plugins, multi-agent, human-in-the-loop, beginner]
type: "deep-dive"
series: "ai-beginner"
series_order: 6
---

# Deep Dive: AI Agent 与 Skills — 让 AI 从「能说」到「能做」

**Topic:** AI Agents, Skills & Tools (Level 5)
**Category:** AI Architecture
**Level:** 中级
**Series:** AI 小白学习系列
**Last Updated:** 2026-03-09

---

## 中文版

---

### 1. 概述

到目前为止你学到的 LLM 只能做一件事：**生成文本**。但真正有用的 AI 需要能**执行操作**：搜索文件、查数据库、调 API、发邮件、改代码。**AI Agent** 就是这样的系统 —— 它不仅能"说"，还能"做"。

**Agent = LLM（大脑）+ Skills/Tools（手脚）+ Memory（记忆）+ Planning（规划）**

你正在使用的这个 GitHub Copilot CLI 就是一个 AI Agent！它的 `grep`、`powershell`、`edit`、`web_fetch` 等工具就是它的 Skills。

### 2. 核心概念

#### 2.1 AI Agent vs 普通聊天机器人

| 维度 | 普通聊天机器人 | AI Agent |
|------|-------------|---------|
| 能力 | 只能对话 | 对话 + 执行操作 |
| 决策 | 被动回答问题 | 主动规划并执行任务 |
| 工具 | 无 | 可调用外部工具/API |
| 记忆 | 仅当前对话 | 可持久化存储信息 |
| 例子 | 基础 ChatGPT 对话 | GitHub Copilot CLI、AutoGPT |

#### 2.2 Skills / Tools / Function Calling

这三个概念紧密相关：

```
Function Calling = 底层机制（LLM 请求调用函数的能力）
      ↑
    Tools = 具体的可调用函数（搜索、编辑、执行命令）
      ↑
    Skills = 一组相关 Tools 的集合（文件操作 Skill、网络诊断 Skill）
      ↑
   Plugins = Skills 的更高级封装（Semantic Kernel 中的概念）
```

**工作流程**：

```
用户："帮我找出所有包含 error 的日志文件"
  │
  ▼
LLM 分析意图 → 决定使用 grep 工具
  │
  ▼
发出 Tool Call: { "tool": "grep", "args": { "pattern": "error", "glob": "*.log" } }
  │
  ▼
框架执行 grep 命令 → 返回结果
  │
  ▼
LLM 根据结果生成回答："找到 3 个文件包含 error..."
```

#### 2.3 MCP（Model Context Protocol）

MCP 是 Anthropic 推出的开放标准，可以理解为 **AI 的 USB-C 接口**：

```
没有 MCP:                          有了 MCP:
                                   
ChatGPT ──自定义──▶ Notion          ChatGPT ──┐
ChatGPT ──自定义──▶ GitHub          Claude  ──┤
Claude  ──自定义──▶ Notion            VS Code ──┼── MCP ──▶ Notion
Claude  ──自定义──▶ GitHub          Cursor  ──┤         ▶ GitHub  
VS Code ──自定义──▶ Notion                    └──       ▶ Database
                                               ▶ 任何 MCP Server
每个组合都要单独开发               开发一次 MCP Server，所有 AI 都能用
```

MCP 定义了三种原语：
- **Tools**：可执行的函数（查询、操作）
- **Resources**：数据源（文件、数据库 schema）
- **Prompts**：交互模板（预定义的提示词）

#### 2.4 Multi-Agent（多智能体协作）

复杂任务可以由多个 Agent 分工协作完成：

```
用户需求："分析这个 pcap 文件并写一份报告"

┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Agent 1      │    │ Agent 2      │    │ Agent 3      │
│ 网络分析专家  │───▶│ 数据整理专家  │───▶│ 报告撰写专家  │
│              │    │              │    │              │
│ 解析 pcap    │    │ 提取关键指标  │    │ 生成结构化报告│
│ 识别异常流量  │    │ 整理时间线    │    │ 添加建议      │
└──────────────┘    └──────────────┘    └──────────────┘
```

#### 2.5 Human-in-the-Loop（人机协作）

对于敏感操作，AI 提议但人类最终决定：

```
AI: "我建议删除这 50 个过期文件，以下是列表：[...] 是否确认？"
人类: "确认" 或 "不，保留 2024 年之后的文件"
AI: 根据人类反馈调整执行
```

### 3. 你身边的 Agent 实例

| Agent | Skills/Tools | 你用它做什么 |
|-------|-------------|------------|
| **GitHub Copilot CLI** | grep, glob, powershell, edit, view, web_fetch, sql | 搜文件、改代码、跑命令、查文档 |
| **ChatGPT with Plugins** | 联网搜索、代码执行、DALL-E 画图 | 搜索最新信息、运行代码 |
| **Claude with MCP** | 连接本地文件、数据库、GitHub | 操作你的开发环境 |
| **Microsoft Copilot** | Office 365 集成（Word、Excel、PPT） | 自动写文档、做 PPT |

### 4. 小结 & 下一步

🎉 **Level 5 完成！** 你现在理解了：
- ✅ AI Agent = LLM + Skills + Memory + Planning
- ✅ Function Calling 是 AI 调用工具的底层机制
- ✅ MCP 是 AI 连接外部系统的标准协议
- ✅ Multi-Agent 让多个 AI 分工协作
- ✅ Human-in-the-Loop 确保安全

📚 **下一课**：[Level 6] **AI 开发实战** — 学习如何用 API 和框架构建自己的 AI 应用。

### 5. 参考资料

- [Semantic Kernel - Plugins](https://learn.microsoft.com/en-us/semantic-kernel/concepts/plugins/) — Semantic Kernel 插件概念
- [Anthropic Claude - Tool Use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — Claude 工具调用
- [Model Context Protocol](https://modelcontextprotocol.io/introduction) — MCP 协议介绍

---

## English Version

---

### 1. Overview

The LLMs you've learned about can only **generate text**. But useful AI needs to **take actions**: search files, query databases, call APIs, send emails, edit code. **AI Agents** are systems that can both "talk" AND "do."

**Agent = LLM (brain) + Skills/Tools (hands & feet) + Memory + Planning**

The GitHub Copilot CLI you're using right now IS an AI Agent! Its `grep`, `powershell`, `edit`, `web_fetch` are all Skills.

### 2. Core Concepts

| Concept | What It Is | Analogy |
|---------|-----------|---------|
| **Agent** | AI system that can plan and execute tasks | A smart employee who works independently |
| **Skills/Tools** | External functions AI can call | The employee's specific abilities |
| **Function Calling** | LLM's ability to request tool invocation | The nervous system connecting brain to hands |
| **MCP** | Standardized protocol for AI-tool connections | USB-C port — one standard for all connections |
| **Multi-Agent** | Multiple Agents collaborating on complex tasks | A team of specialists working together |
| **Human-in-the-Loop** | Human approval for sensitive operations | Manager signing off on important decisions |

### 3. Summary & What's Next

📚 **Next**: [Level 6] **AI Development Hands-on** — Building your own AI applications with APIs and frameworks.

### 4. References

- [Semantic Kernel - Plugins](https://learn.microsoft.com/en-us/semantic-kernel/concepts/plugins/) — Plugin concepts
- [Anthropic Claude - Tool Use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — Claude tool use
- [Model Context Protocol](https://modelcontextprotocol.io/introduction) — MCP introduction
