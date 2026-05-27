---
layout: post
title: "Deep Dive: AI Skills — 让 AI 从「能说」到「能做」的关键机制"
date: 2026-03-09
categories: [Knowledge, AI-Architecture]
tags: [skills, tools, function-calling, plugins, mcp, agents, semantic-kernel]
type: "deep-dive"
---

# Deep Dive: AI Skills — 让 AI 从「能说」到「能做」的关键机制

**Topic:** AI Skills (技能 / 工具调用 / 插件)
**Category:** AI Architecture & Agent Development
**Level:** 中级
**Last Updated:** 2026-03-09

---

## 中文版

---

### 1. 概述 (Overview)

**Skills**（技能）是当前 AI 领域中一个高频出现的术语，它的核心含义是：**赋予 AI 模型调用外部功能的能力，使其从单纯的「文本生成器」升级为「能够执行实际操作的智能体」**。

在不同的平台和框架中，Skills 有着不同的名称 —— OpenAI 称之为 **Function Calling / Tools**，Anthropic Claude 叫 **Tool Use**，Microsoft Semantic Kernel 叫 **Plugins**，而 MCP（Model Context Protocol）则将其标准化为 **Tools + Resources + Prompts** 三个原语。尽管名称不同，它们解决的是同一个根本问题：**LLM 只能生成文本，无法直接操作外部世界**。Skills 就是连接 AI 大脑和外部世界的「手和脚」。

你可以把 Skills 理解为一种**契约**：开发者定义好「有哪些操作可用、需要什么参数、会返回什么结果」，AI 模型则根据用户意图**自主决定何时、如何调用**这些操作。这种模式让 AI 从被动回答问题变成了主动解决问题。

### 2. 核心概念 (Core Concepts)

#### 2.1 Function Calling（函数调用）

- **定义**：LLM 原生支持的一种能力，允许模型在生成回复时「请求」调用一个外部函数，而不仅仅是输出文本
- **作用**：这是 Skills 的底层引擎。没有 Function Calling，模型就无法「选择」使用哪个 Skill
- **关系**：Function Calling 是机制，Skills/Tools/Plugins 是在此机制上的封装

> **类比**：如果 LLM 是一个人的大脑，Function Calling 就是大脑向手脚发出指令的神经系统，而 Skills 就是手脚本身的各种能力（写字、打字、开车等）。

#### 2.2 Tools（工具）

- **定义**：AI 可以调用的可执行函数，定义了名称、描述、参数 schema 和返回值
- **作用**：每个 Tool 是一个原子操作单元，比如「搜索网页」「读取文件」「查询数据库」
- **关键特征**：Tool 需要有**语义化描述**（semantic description），这样 AI 才能理解什么情况下该使用它

#### 2.3 Plugins（插件）

- **定义**：一组相关 Tools 的逻辑集合，封装了某个领域的完整功能
- **作用**：Plugin 将多个相关的函数组织在一起，比如一个「邮件 Plugin」包含发送、读取、搜索、删除等多个函数
- **与 Tools 的关系**：Plugin 是 Tool 的容器，一个 Plugin 包含多个 Tools

#### 2.4 Agents（智能体）

- **定义**：具备自主决策能力的 AI 系统，能够根据目标自动规划和执行一系列操作
- **作用**：Agent 是 Skills 的使用者。Agent 根据用户意图，自动选择和编排多个 Skills 来完成复杂任务
- **关系**：Agent = LLM + Skills + Memory + Planning（规划能力）

#### 2.5 MCP（Model Context Protocol）

- **定义**：由 Anthropic 推出的开源标准协议，定义了 AI 应用连接外部系统的统一接口
- **作用**：类似 USB-C 接口标准，让不同 AI 应用可以用统一的方式连接各种外部工具和数据源
- **三大原语**：Tools（可执行函数）、Resources（数据源）、Prompts（交互模板）

### 3. 工作原理 (How It Works)

#### 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                    用户 (User)                            │
│                  "帮我查下明天的天气"                        │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                 AI Agent / Host                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   LLM Core   │  │   Memory     │  │   Planner    │  │
│  │  (大语言模型)  │  │  (上下文记忆) │  │  (任务规划)   │  │
│  └──────┬───────┘  └──────────────┘  └──────────────┘  │
│         │                                               │
│         │  Function Calling                             │
│         ▼                                               │
│  ┌──────────────────────────────────────────────────┐   │
│  │            Skills / Plugins Registry              │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │   │
│  │  │Weather   │ │Calendar  │ │Database          │  │   │
│  │  │Plugin    │ │Plugin    │ │Plugin            │  │   │
│  │  │─get_     │ │─get_     │ │─query()          │  │   │
│  │  │ weather()│ │ events() │ │─insert()         │  │   │
│  │  │─get_     │ │─create_  │ │─update()         │  │   │
│  │  │ forecast│ │  event() │ │                  │  │   │
│  │  └──────────┘ └──────────┘ └──────────────────┘  │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│               外部系统 / API / 数据源                      │
│   天气 API    日历服务    数据库    文件系统    网页搜索      │
└─────────────────────────────────────────────────────────┘
```

#### 详细流程

以用户问「帮我把台灯打开」为例，完整的 Skill 调用流程如下：

1. **Step 1: Skill 注册（Registration）**
   - 开发者定义 Skill/Plugin 的函数、参数描述、返回值
   - 将 Skill 注册到 AI Kernel/Agent 中
   - 所有可用的 Skill 会被序列化为 JSON Schema，描述每个函数的名称、功能和参数

2. **Step 2: 用户请求发送到 LLM**
   - 用户消息 + 所有可用 Skill 的 JSON Schema 描述一起发送给 LLM
   - LLM 收到的不仅是用户的话，还有一份「我有哪些工具可以用」的清单

3. **Step 3: LLM 决策（Decision Making）**
   - LLM 分析用户意图，判断是直接回答还是需要调用某个 Skill
   - 如果需要调用，LLM 输出一个结构化的 Tool Call 请求（包含函数名和参数）
   - 例如：`{ "function": "change_state", "arguments": { "id": 1, "is_on": true } }`

4. **Step 4: Skill 执行（Execution）**
   - AI 框架（如 Semantic Kernel）接收到 Tool Call 请求
   - 提取函数名和参数，在已注册的 Skill 中找到对应函数
   - 执行该函数，获取返回结果

5. **Step 5: 结果返回 LLM**
   - 函数执行结果发送回 LLM
   - LLM 根据结果生成最终的自然语言回复
   - 例如："好的，台灯已经为您打开了。"

6. **Step 6: 多轮调用（Multi-turn）**
   - 如果一个任务需要多个 Skill 协作，LLM 会自动发起多轮调用
   - 例如：用户说「帮我订一个大号披萨然后结账」→ LLM 先调用 `add_pizza_to_cart`，再调用 `checkout`

#### 关键机制

**自动编排（Auto Orchestration）**：现代 AI 框架支持自动 Function Calling，AI 模型自主决定调用哪些函数、按什么顺序调用。开发者不需要硬编码调用逻辑。

**语义描述（Semantic Description）**：每个 Skill 函数都需要有清晰的自然语言描述，这是 AI 正确选择工具的关键。描述越准确，AI 的调用越精准。

**Human-in-the-Loop**：对于敏感操作（如付款、删除数据），可以设置人工审批环节，AI 提出建议但由人类最终确认执行。

### 4. 关键配置与参数 (Key Configurations)

| 配置项/参数 | 默认值 | 说明 | 常见调优场景 |
|------------|--------|------|------------|
| `FunctionChoiceBehavior` | `Auto` | 控制 LLM 是否自动调用函数 | 设为 `None` 禁用自动调用，`Required` 强制调用 |
| Tool Description | - | 每个 Tool 的自然语言描述 | 描述不准确会导致 AI 误调用或不调用 |
| Max Iterations | 通常 10 | 单次请求中最多允许的函数调用轮数 | 复杂任务链可能需要增大此值 |
| Parallel Tool Calling | 开启 | 是否允许 LLM 同时调用多个 Tool | 对于有依赖关系的调用应禁用并行 |
| Strict Mode | 关闭 | 是否强制 LLM 输出严格符合 schema 的参数 | 生产环境建议开启，避免参数类型错误 |
| Tool Choice | `auto` | `auto`/`any`/`none`/指定工具名 | `any` 强制使用工具，指定名称则限定使用特定工具 |

### 5. 常见问题与排查 (Common Issues & Troubleshooting)

#### 问题 A: AI 不调用预期的 Skill

- **可能原因**：Tool 的描述不够清晰或与用户意图不匹配
- **排查思路**：检查 Tool 的 `description` 字段，确保准确描述了功能、适用场景和参数含义
- **解决方案**：丰富描述信息，添加使用示例（few-shot examples），说明「什么时候该用」和「什么时候不该用」

#### 问题 B: AI 调用了错误的 Skill

- **可能原因**：多个 Tool 的描述过于相似，AI 无法区分
- **排查思路**：审查所有 Tool 的描述，确保每个 Tool 有独特的功能定位
- **解决方案**：在描述中明确区分不同 Tool 的适用场景，使用对比性描述

#### 问题 C: 函数参数传递错误

- **可能原因**：参数的 schema 定义不够精确，或参数描述含糊
- **排查思路**：检查 JSON Schema 定义，特别是 `type`、`enum`、`required` 等约束
- **解决方案**：启用 Strict Mode 保证参数格式正确；为每个参数添加详细的 `description`

#### 问题 D: 多轮调用陷入无限循环

- **可能原因**：函数返回值不够明确，LLM 无法判断任务是否完成
- **排查思路**：检查函数返回值是否包含足够的状态信息
- **解决方案**：设置 `Max Iterations` 上限；确保函数返回明确的成功/失败状态

### 6. 实战经验 (Practical Tips)

- **最佳实践**：
  - 用 snake_case 命名函数（大多数 LLM 用 Python 训练，对 snake_case 更友好）
  - 每个 Skill 函数做一件事，保持原子性，不要在一个函数里塞太多逻辑
  - 为 Tool 描述投入足够精力——这是 AI 理解你工具的唯一途径
  - 优先使用 Native Code Plugin，等项目成熟后再考虑 OpenAPI / MCP 方式跨平台共享

- **常见误区**：
  - ❌ 认为 Skills 只是简单的 API wrapper —— 实际上语义描述和编排逻辑是核心
  - ❌ 给 AI 暴露过多 Tools —— 过多工具会增加 token 消耗且降低选择准确性
  - ❌ 忽略错误处理 —— 函数执行失败时应返回清晰的错误信息，让 AI 能理解并重试或告知用户

- **性能考量**：
  - 每个注册的 Tool 都会消耗 input token（描述会随每次请求发送给 LLM）
  - 工具数量过多时考虑使用 Tool Search（动态筛选相关工具）而非全部注册
  - 数据检索类 Skill 可以用缓存和中间模型优化性能

- **安全注意**：
  - 敏感操作（删除、付款、发送邮件）必须实现 Human-in-the-Loop 审批
  - 不要在 Tool 描述中暴露内部系统细节
  - 对 Tool 的输入参数做严格验证，防止 Prompt Injection 通过 Tool 执行恶意操作

### 7. 与相关技术的对比 (Comparison with Related Technologies)

| 维度 | Function Calling (原生) | Semantic Kernel Plugins | MCP (Model Context Protocol) | Custom GPTs / Actions |
|------|----------------------|----------------------|--------------------------|---------------------|
| **抽象层级** | 最底层机制 | 框架级封装 | 协议级标准 | 产品级封装 |
| **适用场景** | 直接 API 调用 | 企业级应用开发 | 跨平台工具标准化 | 终端用户自定义 AI |
| **代码要求** | 需要手动处理 JSON | 框架自动处理序列化 | SDK 抽象通信细节 | 低代码/无代码 |
| **工具共享** | 不可共享 | 同语言内共享 | 跨语言、跨平台共享 | 通过 GPT Store 共享 |
| **依赖注入** | 不支持 | 原生支持 | 依赖具体实现 | 不支持 |
| **标准化程度** | 各厂商不同 | Microsoft 生态 | 开源标准，广泛支持 | OpenAI 生态 |
| **典型代表** | OpenAI API, Claude API | Microsoft Copilot, Azure AI | VS Code, Claude Desktop, Cursor | ChatGPT Plus |

**选型建议**：
- 如果是快速原型验证 → 直接用 Function Calling
- 如果是企业级 .NET/Python/Java 应用 → Semantic Kernel Plugins
- 如果需要跨平台、跨 AI 应用共享工具 → MCP
- 如果面向终端用户的简单场景 → Custom GPTs / Actions

### 8. 具体案例 (Real-World Use Cases)

#### 案例 1: 智能家居控制

用户对 AI 说「把客厅灯调暗一点，调成暖黄色」，AI 通过注册的 `LightsPlugin`：
1. 调用 `get_lights()` 获取灯具列表
2. 找到客厅灯的 ID
3. 调用 `change_state(id=1, brightness=30, hex="FFD700")` 修改状态
4. 回复用户「客厅灯已调暗并设置为暖黄色」

#### 案例 2: 披萨订餐 Agent

完整的订餐流程通过多个 Skill 协作完成：
- `get_pizza_menu()` → 展示菜单
- `add_pizza_to_cart(size="large", toppings=["cheese","pepperoni"])` → 加入购物车
- `get_cart()` → 确认订单
- `checkout()` → 完成付款

#### 案例 3: Support Engineer 的 AI 助手

这正是我们日常使用 GitHub Copilot CLI 的场景：
- **文件搜索 Skill**：`grep`、`glob` 工具帮 AI 在代码库中查找信息
- **Shell 执行 Skill**：`powershell` 工具让 AI 执行系统命令
- **文件编辑 Skill**：`edit`、`create` 工具让 AI 直接修改代码
- **知识检索 Skill**：`web_fetch` 工具让 AI 查阅在线文档

这些 Skills 的组合让 AI 从一个「只会聊天的模型」变成了一个「能读代码、改代码、跑测试、查文档的工程师助手」。

#### 案例 4: MCP 驱动的企业数据分析

一个连接了 MCP Server 的 AI 应用：
- **Database MCP Server**：提供 `query_database` Tool + 数据库 schema Resource
- **Visualization MCP Server**：提供 `create_chart` Tool
- 用户说「分析上季度销售数据并生成趋势图」
- AI 自动：查询数据库 → 处理数据 → 生成可视化图表

### 9. 参考资料 (References)

- [Semantic Kernel - Plugins 概念](https://learn.microsoft.com/en-us/semantic-kernel/concepts/plugins/) — Microsoft 官方文档，详解 Plugin 的定义、导入和使用方式
- [Semantic Kernel - Function Calling](https://learn.microsoft.com/en-us/semantic-kernel/concepts/ai-services/chat-completion/function-calling/) — 深入讲解自动函数调用的工作机制和实现细节
- [Anthropic Claude - Tool Use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — Claude 的工具调用文档，包含 Client Tools 和 Server Tools 两种模式
- [Model Context Protocol (MCP) - Introduction](https://modelcontextprotocol.io/introduction) — MCP 官方介绍，「AI 的 USB-C 接口」标准协议

---

## English Version

---

### 1. Overview

**Skills** is one of the most frequently mentioned terms in today's AI landscape. At its core, it refers to **the ability to give AI models the power to call external functions, transforming them from mere "text generators" into "intelligent agents capable of taking real-world actions."**

Across different platforms and frameworks, Skills go by different names — OpenAI calls them **Function Calling / Tools**, Anthropic Claude uses **Tool Use**, Microsoft Semantic Kernel calls them **Plugins**, and MCP (Model Context Protocol) standardizes them into three primitives: **Tools + Resources + Prompts**. Despite the naming differences, they all solve the same fundamental problem: **LLMs can only generate text — they cannot directly interact with the external world.** Skills are the "hands and feet" that connect AI's brain to the outside world.

Think of Skills as a **contract**: developers define "what operations are available, what parameters are needed, and what results will be returned," while the AI model **autonomously decides when and how to invoke** these operations based on user intent. This pattern transforms AI from passively answering questions to actively solving problems.

### 2. Core Concepts

#### 2.1 Function Calling

- **Definition**: A native capability of LLMs that allows models to "request" the invocation of an external function during response generation, rather than just outputting text
- **Role**: This is the underlying engine of Skills. Without Function Calling, models cannot "choose" which Skill to use
- **Relationship**: Function Calling is the mechanism; Skills/Tools/Plugins are encapsulations built on top of it

> **Analogy**: If the LLM is a person's brain, Function Calling is the nervous system that sends commands from the brain to the limbs, and Skills are the various capabilities of those limbs (writing, typing, driving, etc.).

#### 2.2 Tools

- **Definition**: Executable functions that AI can invoke, defined with a name, description, parameter schema, and return value
- **Role**: Each Tool is an atomic operational unit, such as "search the web," "read a file," or "query a database"
- **Key Feature**: Tools require **semantic descriptions** so the AI can understand when to use them

#### 2.3 Plugins

- **Definition**: A logical collection of related Tools that encapsulates complete functionality for a specific domain
- **Role**: A Plugin organizes multiple related functions together — for example, an "Email Plugin" containing send, read, search, and delete functions
- **Relationship to Tools**: A Plugin is a container for Tools; one Plugin contains multiple Tools

#### 2.4 Agents

- **Definition**: AI systems with autonomous decision-making capabilities that can automatically plan and execute a series of operations based on goals
- **Role**: Agents are the consumers of Skills. An Agent automatically selects and orchestrates multiple Skills to complete complex tasks based on user intent
- **Formula**: Agent = LLM + Skills + Memory + Planning

#### 2.5 MCP (Model Context Protocol)

- **Definition**: An open-source standard protocol introduced by Anthropic that defines a unified interface for AI applications to connect to external systems
- **Role**: Similar to the USB-C interface standard — it allows different AI applications to connect to various external tools and data sources in a unified way
- **Three Primitives**: Tools (executable functions), Resources (data sources), Prompts (interaction templates)

### 3. How It Works

#### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                      User                                │
│              "Turn on the table lamp"                     │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                  AI Agent / Host                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   LLM Core   │  │   Memory     │  │   Planner    │  │
│  └──────┬───────┘  └──────────────┘  └──────────────┘  │
│         │                                               │
│         │  Function Calling                             │
│         ▼                                               │
│  ┌──────────────────────────────────────────────────┐   │
│  │            Skills / Plugins Registry              │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │   │
│  │  │Weather   │ │Calendar  │ │Database          │  │   │
│  │  │Plugin    │ │Plugin    │ │Plugin            │  │   │
│  │  └──────────┘ └──────────┘ └──────────────────┘  │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│            External Systems / APIs / Data Sources        │
│   Weather API   Calendar   Database   Filesystem   Web   │
└─────────────────────────────────────────────────────────┘
```

#### Detailed Flow

Using the example of a user saying "Turn on the table lamp," here is the complete Skill invocation flow:

1. **Step 1: Skill Registration**
   - Developer defines the Skill/Plugin's functions, parameter descriptions, and return values
   - Registers the Skill with the AI Kernel/Agent
   - All available Skills are serialized into JSON Schema describing each function's name, purpose, and parameters

2. **Step 2: User Request Sent to LLM**
   - User message + JSON Schema descriptions of all available Skills are sent together to the LLM
   - The LLM receives not just the user's words, but also a catalog of "what tools I have available"

3. **Step 3: LLM Decision Making**
   - LLM analyzes user intent and determines whether to respond directly or invoke a Skill
   - If invocation is needed, the LLM outputs a structured Tool Call request (containing function name and arguments)
   - Example: `{ "function": "change_state", "arguments": { "id": 1, "is_on": true } }`

4. **Step 4: Skill Execution**
   - The AI framework (e.g., Semantic Kernel) receives the Tool Call request
   - Extracts the function name and parameters, locates the corresponding function in registered Skills
   - Executes the function and obtains the return result

5. **Step 5: Result Returned to LLM**
   - Function execution results are sent back to the LLM
   - LLM generates a final natural language response based on the results
   - Example: "Done! The table lamp has been turned on for you."

6. **Step 6: Multi-turn Invocation**
   - If a task requires collaboration from multiple Skills, the LLM automatically initiates multiple rounds of calls
   - Example: User says "Order a large pizza and checkout" → LLM first calls `add_pizza_to_cart`, then calls `checkout`

#### Key Mechanisms

**Auto Orchestration**: Modern AI frameworks support automatic Function Calling where the AI model autonomously decides which functions to call and in what order. Developers don't need to hardcode invocation logic.

**Semantic Description**: Every Skill function needs clear natural language descriptions — this is the key to AI correctly selecting tools. The more accurate the description, the more precise the AI's invocations.

**Human-in-the-Loop**: For sensitive operations (payments, data deletion), approval checkpoints can be configured where AI proposes actions but humans provide final confirmation.

### 4. Key Configurations

| Configuration | Default | Description | Common Tuning Scenarios |
|--------------|---------|-------------|----------------------|
| `FunctionChoiceBehavior` | `Auto` | Controls whether LLM auto-invokes functions | Set to `None` to disable, `Required` to force invocation |
| Tool Description | - | Natural language description of each Tool | Inaccurate descriptions cause AI to mis-invoke or skip tools |
| Max Iterations | ~10 | Maximum function call rounds per request | Complex task chains may need a higher limit |
| Parallel Tool Calling | Enabled | Whether LLM can call multiple Tools simultaneously | Disable for calls with dependencies |
| Strict Mode | Off | Whether to force LLM to output schema-compliant parameters | Recommended for production to avoid type errors |
| Tool Choice | `auto` | `auto`/`any`/`none`/specific tool name | `any` forces tool use; specific name restricts to one tool |

### 5. Common Issues & Troubleshooting

#### Issue A: AI Doesn't Invoke the Expected Skill

- **Possible Cause**: Tool description is unclear or doesn't match user intent
- **Troubleshooting**: Review the Tool's `description` field; ensure it accurately describes functionality, use cases, and parameter meanings
- **Solution**: Enrich descriptions, add few-shot examples, explain "when to use" and "when not to use"

#### Issue B: AI Invokes the Wrong Skill

- **Possible Cause**: Multiple Tools have overly similar descriptions; AI can't distinguish them
- **Troubleshooting**: Audit all Tool descriptions to ensure each has a unique functional positioning
- **Solution**: Use comparative descriptions to clearly differentiate applicable scenarios

#### Issue C: Incorrect Function Parameters

- **Possible Cause**: Parameter schema isn't precise enough, or parameter descriptions are vague
- **Troubleshooting**: Check JSON Schema definitions, especially `type`, `enum`, and `required` constraints
- **Solution**: Enable Strict Mode for guaranteed parameter format correctness; add detailed `description` for each parameter

#### Issue D: Multi-turn Calls Enter Infinite Loop

- **Possible Cause**: Function return values lack clarity; LLM can't determine if the task is complete
- **Troubleshooting**: Check if function returns include sufficient status information
- **Solution**: Set `Max Iterations` cap; ensure functions return explicit success/failure status

### 6. Practical Tips

- **Best Practices**:
  - Use snake_case for function names (most LLMs are trained with Python and work better with snake_case)
  - Each Skill function should do one thing — keep it atomic; don't cram too much logic into one function
  - Invest adequate effort in Tool descriptions — it's the only way AI understands your tools
  - Start with Native Code Plugins; consider OpenAPI / MCP for cross-platform sharing as projects mature

- **Common Mistakes**:
  - ❌ Thinking Skills are just simple API wrappers — semantic descriptions and orchestration logic are the core
  - ❌ Exposing too many Tools to AI — too many tools increase token consumption and reduce selection accuracy
  - ❌ Ignoring error handling — return clear error messages when functions fail so AI can understand and retry or inform the user

- **Performance Considerations**:
  - Every registered Tool consumes input tokens (descriptions are sent with every request to the LLM)
  - When tool count is high, consider Tool Search (dynamic relevant tool filtering) instead of registering all
  - Data retrieval Skills can be optimized with caching and intermediate models

- **Security Notes**:
  - Sensitive operations (delete, pay, send email) must implement Human-in-the-Loop approval
  - Don't expose internal system details in Tool descriptions
  - Strictly validate Tool input parameters to prevent Prompt Injection attacks via Tool execution

### 7. Comparison with Related Technologies

| Dimension | Function Calling (Native) | Semantic Kernel Plugins | MCP (Model Context Protocol) | Custom GPTs / Actions |
|-----------|-------------------------|----------------------|--------------------------|---------------------|
| **Abstraction Level** | Lowest-level mechanism | Framework-level encapsulation | Protocol-level standard | Product-level encapsulation |
| **Use Case** | Direct API calls | Enterprise app development | Cross-platform tool standardization | End-user AI customization |
| **Code Required** | Manual JSON handling | Framework auto-serialization | SDK abstracts communication | Low-code/No-code |
| **Tool Sharing** | Not shareable | Within same language | Cross-language, cross-platform | Via GPT Store |
| **Dependency Injection** | Not supported | Natively supported | Implementation-dependent | Not supported |
| **Standardization** | Varies by vendor | Microsoft ecosystem | Open standard, broadly supported | OpenAI ecosystem |
| **Examples** | OpenAI API, Claude API | Microsoft Copilot, Azure AI | VS Code, Claude Desktop, Cursor | ChatGPT Plus |

**Selection Guide**:
- For quick prototyping → Use Function Calling directly
- For enterprise .NET/Python/Java applications → Semantic Kernel Plugins
- For cross-platform, cross-AI-app tool sharing → MCP
- For simple end-user-facing scenarios → Custom GPTs / Actions

### 8. Real-World Use Cases

#### Case 1: Smart Home Control

User tells AI "Dim the living room light and set it to warm yellow." AI uses the registered `LightsPlugin`:
1. Calls `get_lights()` to get the list of lights
2. Identifies the living room light's ID
3. Calls `change_state(id=1, brightness=30, hex="FFD700")` to modify state
4. Responds: "The living room light has been dimmed and set to warm yellow."

#### Case 2: Pizza Ordering Agent

Complete ordering flow through multi-Skill collaboration:
- `get_pizza_menu()` → Display menu
- `add_pizza_to_cart(size="large", toppings=["cheese","pepperoni"])` → Add to cart
- `get_cart()` → Confirm order
- `checkout()` → Complete payment

#### Case 3: Support Engineer's AI Assistant

This is exactly the scenario of our daily GitHub Copilot CLI usage:
- **File Search Skill**: `grep`, `glob` tools help AI find information in codebases
- **Shell Execution Skill**: `powershell` tool lets AI execute system commands
- **File Edit Skill**: `edit`, `create` tools let AI directly modify code
- **Knowledge Retrieval Skill**: `web_fetch` tool lets AI consult online documentation

The combination of these Skills transforms AI from a "chatbot" into an "engineering assistant that can read code, modify code, run tests, and look up documentation."

#### Case 4: MCP-Driven Enterprise Data Analysis

An AI application connected to MCP Servers:
- **Database MCP Server**: Provides `query_database` Tool + database schema Resource
- **Visualization MCP Server**: Provides `create_chart` Tool
- User says "Analyze last quarter's sales data and generate a trend chart"
- AI automatically: queries database → processes data → generates visualization

### 9. References

- [Semantic Kernel - Plugins Concept](https://learn.microsoft.com/en-us/semantic-kernel/concepts/plugins/) — Microsoft official documentation detailing Plugin definition, import, and usage patterns
- [Semantic Kernel - Function Calling](https://learn.microsoft.com/en-us/semantic-kernel/concepts/ai-services/chat-completion/function-calling/) — In-depth explanation of automatic function calling mechanics and implementation details
- [Anthropic Claude - Tool Use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — Claude's tool invocation documentation, covering both Client Tools and Server Tools patterns
- [Model Context Protocol (MCP) - Introduction](https://modelcontextprotocol.io/introduction) — Official MCP introduction, the "USB-C port for AI" standard protocol
