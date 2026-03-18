---
layout: post
title: "Deep Dive: Model Context Protocol (MCP) — 架构、原理与网络数据流全解析"
date: 2026-03-18
categories: [Knowledge, AI]
tags: [mcp, model-context-protocol, llm, ai-agent, rag, function-calling, json-rpc, copilot]
type: "deep-dive"
---

# Deep Dive: Model Context Protocol (MCP) — 架构、原理与网络数据流全解析

**Topic:** Model Context Protocol (MCP)  
**Category:** AI / Protocol / Architecture  
**Level:** 中级 / Intermediate  
**Last Updated:** 2026-03-18

---

## 1. 概述 (Overview)

Model Context Protocol（MCP）是由 Anthropic 推出的一个**开放标准协议**，用于将 AI 模型（LLM）与外部数据源和工具连接起来。它基于 JSON-RPC 构建，提供了一个有状态的会话协议，专注于上下文交换和采样协调。

MCP 解决的核心问题是：**AI 应用如何以标准化、安全、可扩展的方式连接到外部系统**。在 MCP 出现之前，每个 AI 应用都需要为每个外部工具编写定制的集成代码，形成 M×N 的集成复杂度。MCP 将其简化为 M+N——每个 AI 应用实现一次 MCP Client，每个工具实现一次 MCP Server，即可互联互通。

在整个 AI 技术体系中，MCP 处于 **AI 应用层与外部能力层之间的协议层**，类似于 Web 领域的 HTTP 协议——它定义了通信标准，而不关心具体的业务逻辑。

> **类比：MCP 就像 AI 世界的 USB 接口** — 让不同的 AI 模型可以用统一的方式"插入"各种外部能力，而不需要为每个工具写定制集成。

---

## 2. 核心概念 (Core Concepts)

### 2.1 MCP Host（宿主）

MCP Host 是 MCP 架构中的**最外层应用程序**，即用户直接交互的 AI 应用。

- **角色**：容器和协调者（Container & Coordinator）
- **职责**：
  - 创建并管理多个 MCP Client 实例
  - 控制 Client 的连接权限和生命周期
  - 执行安全策略和用户授权
  - 协调 AI/LLM 集成与上下文聚合
- **常见实例**：Claude Desktop、VS Code + Copilot、GitHub Copilot CLI、Cursor、自定义 AI 应用

```
用户 ↔ MCP Host（AI 应用）
            ├── MCP Client 1 ↔ MCP Server A（GitHub）
            ├── MCP Client 2 ↔ MCP Server B（文件系统）
            └── MCP Client 3 ↔ MCP Server C（数据库）
```

> **类比**：如果 MCP 是 USB 标准，那么 Host 就是电脑本身——它提供 USB 接口（Client），让你插入各种外设（Server）。

### 2.2 MCP Client（客户端）

MCP Client 是 MCP 架构中的**中间层**，负责在 Host 和 Server 之间建立并维护连接。

- **角色**：协议层桥梁（Protocol Bridge）
- **关键特性**：
  - 与 MCP Server 保持 **1:1 的有状态会话**
  - 处理协议协商和能力交换（Capability Exchange）
  - 双向路由协议消息
  - 管理订阅和通知
  - 维护 Server 之间的安全隔离
- **生命周期**：由 Host 创建和管理，不独立存在

| 特性 | 说明 |
|------|------|
| 1:1 关系 | 每个 Client 只连接一个 Server |
| 由 Host 创建 | Client 不独立存在，由 Host 管理生命周期 |
| 协议协商 | 启动时与 Server 协商支持的能力（capabilities） |
| 透明转发 | AI 模型不直接感知 Client，Host 统一调度 |

> **类比**：Client 就是 USB 端口和驱动程序——你不会直接操作它，但没有它设备就连不上。

### 2.3 MCP Server（服务器）

MCP Server 是 MCP 架构中的**能力提供者**，负责将具体的工具、数据和功能暴露给 AI 模型使用。

- **角色**：专注于提供特定能力的独立服务
- **关键特性**：
  - 通过 MCP 原语暴露 Resources、Tools 和 Prompts
  - 独立运行，职责单一
  - 可以是本地进程或远程服务
  - 必须遵守安全约束

**MCP Server 暴露的三类能力：**

| 能力类型 | 说明 | 示例 |
|----------|------|------|
| **Tools** | AI 可调用的函数/操作 | `search_repositories`、`create_issue` |
| **Resources** | AI 可读取的数据源 | 文件内容、数据库记录 |
| **Prompts** | 预定义的提示模板 | 代码审查模板、总结模板 |

**重要澄清：MCP Server 不是一台物理机器，而是一个软件程序/进程。** 它通常运行在你本地电脑上，本身一般不存储数据，而是作为 AI 和真实数据源之间的**桥梁/适配器**。

```
MCP Server = 翻译器/适配器

它的工作是：
  把 AI 的请求（MCP 协议格式）
  翻译成
  后端服务能理解的调用（REST API、SQL、CLI 命令等）
```

| MCP Server | 它自己存什么 | 真正的数据在哪 |
|------------|------------|--------------|
| github-mcp-server | 只存 GitHub Token | 数据在 github.com |
| postgres-mcp-server | 只存连接字符串 | 数据在 PostgreSQL 数据库 |
| filesystem-mcp-server | 什么都不存 | 数据在本地磁盘文件 |

> **类比**：Server 就是 U 盘/键盘/摄像头——它提供具体的功能，插上就能用。

### 2.4 MCP Server 与 LLM 模型的关系

**核心要点：MCP Server 和 LLM 模型之间没有直接连接，由 Host 居中调度。**

```
LLM 模型 ←──✕ 不直连 ✕──→ MCP Server
      ↑                        ↑
      └──── Host 居中调度 ──────┘
```

| | LLM 模型 | MCP Server |
|--|---------|------------|
| 做什么 | 理解问题、推理、决定要不要用工具 | 提供工具、执行具体操作 |
| 在哪里 | 通常在云端（OpenAI/Azure） | 通常在本地电脑 |
| 知不知道对方 | 只知道"有哪些工具可用"（名称+描述） | **完全不知道**模型的存在 |

这种解耦设计带来的好处：
- **换模型**：Host 改个配置就行，Server 不用动
- **换工具**：加个 Server 就行，模型不用动
- **安全性**：Host 统一控制权限和授权策略

对 LLM 来说，**MCP 工具和普通 function calling 工具没有任何区别**。MCP 是 Host 侧的实现细节，模型完全无感知。

---

## 3. 工作原理 (How It Works)

### 3.1 整体架构

```
┌─────────────────────────────────────────────────────┐
│  MCP Host (AI 应用，如 Copilot CLI)                  │
│  ┌───────────────────────────────────────────────┐  │
│  │  MCP Client 1 (1:1 连接)                      │  │
│  └──────────────────┬────────────────────────────┘  │
│  ┌───────────────────┼──────────────────────────┐   │
│  │  MCP Client 2     │                          │   │
│  └──────────────────┬┼──────────────────────────┘   │
└─────────────────────┼┼──────────────────────────────┘
                      ││  stdio / HTTP+SSE (传输层)
┌─────────────────────┼┼──────────────────────────────┐
│  MCP Server A       ▼│                              │
│  ┌─────────┐ ┌─────────┐ ┌──────────┐              │
│  │ tool_1  │ │ tool_2  │ │ resource │              │
│  └─────────┘ └─────────┘ └──────────┘              │
└─────────────────────────────────────────────────────┘
┌──────────────────────┼──────────────────────────────┐
│  MCP Server B        ▼                              │
│  ┌─────────┐ ┌─────────┐                           │
│  │ tool_3  │ │ prompt_1│                           │
│  └─────────┘ └─────────┘                           │
└─────────────────────────────────────────────────────┘
```

### 3.2 两种传输模式

| 传输方式 | 通信方式 | 适用场景 | 示例 |
|----------|---------|---------|------|
| **stdio** | 标准输入/输出管道 | 本地进程，Host 直接启动 Server 子进程 | github-mcp-server、filesystem-mcp-server |
| **Streamable HTTP** | HTTP + Server-Sent Events | 远程服务，跨网络通信 | 云端部署的 MCP Server |

### 3.3 完整 Turn（一次完整交互）的详细流程

以用户问 "帮我搜索 GitHub 上的 MCP 项目" 为例：

**Step 1: 用户发送消息**
```
用户输入 → Copilot CLI (Host) 接收（纯本地操作）
```

**Step 2: Host 构造请求发送给云端 LLM**
```
POST https://api.githubcopilot.com/chat/completions
Body: {
  messages: [对话历史 + 用户消息],
  tools: [从 MCP Server 获取的工具列表],   ← MCP 工具在这里注入
  model: "claude-opus-4.6-1m"
}
★ 出站 HTTPS 流量 → 云端
```

**Step 3: LLM 推理并决策**
```
云端 LLM 分析问题，决定调用 search_repositories 工具
返回: {tool_calls: [{name: "search_repositories", args: {query: "MCP"}}]}
★ 入站 HTTPS 流量 ← 云端
```

**Step 4: Host 将请求路由到 MCP Server**
```
Host → MCP Client → stdio 管道 → github-mcp-server（本地进程）
★ 无网络流量（本地进程间通信）
```

**Step 5: MCP Server 执行实际操作**
```
github-mcp-server → GET https://api.github.com/search/repositories?q=MCP
★ 出站 HTTPS 流量 → GitHub API
```

**Step 6: 结果逐层返回**
```
GitHub API → github-mcp-server → stdio → MCP Client → Host
★ 仅 GitHub API 返回涉及网络流量
```

**Step 7: Host 将工具结果发回 LLM**
```
POST https://api.githubcopilot.com/chat/completions
Body: { messages: [..., 工具执行结果] }
★ 出站 HTTPS 流量 → 云端
```

**Step 8: LLM 生成最终回复**
```
LLM 根据工具结果生成自然语言回答 → 流式返回给用户
★ 入站 HTTPS 流量 ← 云端
```

### 3.4 网络流量全景

```
你的电脑 (Windows)                          互联网                     云端
━━━━━━━━━━━━━━━━━━━                   ━━━━━━━━━━━━━              ━━━━━━━━━━━━━━━

┌─────────────────┐                                      ┌──────────────────┐
│  Copilot CLI    │ ── HTTPS ──────────────────────────→ │  GitHub Copilot  │
│  (MCP Host)     │    消息 + 工具定义                     │  API 服务器       │
│                 │                                       │  ┌────────────┐  │
│                 │ ← HTTPS（流式）─────────────────────── │  │ LLM 模型   │  │
│                 │    模型回复 / 工具调用指令               │  └────────────┘  │
│                 │                                       └──────────────────┘
│  ┌───────────┐  │
│  │MCP Client │──┼── stdio（本地管道）──┐
│  └───────────┘  │                     ▼
└─────────────────┘          ┌─────────────────────┐
                             │  github-mcp-server   │
                             │  (本地进程)           │
                             │  → HTTPS ──────────────→ api.github.com
                             │  ← HTTPS ──────────────← api.github.com
                             └─────────────────────┘
```

**流量统计：**

| 场景 | 出站请求次数 | 涉及端点 |
|------|------------|---------|
| 纯聊天（不调工具） | 1 次往返 | Copilot API |
| 调用 1 个工具 | 2 次 Copilot API + 1 次工具 API | Copilot API + github.com |
| 调用 N 个工具 | 至少 2 次 Copilot API + N 次工具 API | 多个端点 |

**关键事实：你本地没有 LLM 模型（模型通常几百 GB），所有推理都在云端完成，断网无法使用。**

---

## 4. 关键配置与参数 (Key Configurations)

| 配置项 | 说明 | 典型值 | 常见调优场景 |
|--------|------|--------|------------|
| **传输方式** | Server 与 Client 的通信方式 | `stdio` / `streamable-http` | 本地用 stdio，远程用 HTTP |
| **Server 启动命令** | Host 如何启动 Server 进程 | `python server.py` | 配置不同语言的 Server |
| **能力协商** | Client-Server 初始化时交换支持的功能 | tools, resources, prompts | 按需启用/禁用特定能力 |
| **安全策略** | Host 控制哪些工具可用 | 白名单/黑名单 | 限制敏感操作 |
| **超时设置** | 工具调用的超时时间 | 30s-120s | 长时间运行的工具需要增大 |
| **并发限制** | 同时活跃的 Client 数量 | 无硬性限制 | Server 过多时影响性能 |

---

## 5. 常见问题与排查 (Common Issues & Troubleshooting)

### 问题 A: MCP Server 无法连接

- **可能原因**：Server 进程未启动、stdio 管道断开、Server 崩溃
- **排查思路**：
  1. 检查 Server 进程是否在运行
  2. 查看 Server 的 stderr 输出（日志）
  3. 确认 Server 的启动命令和路径正确
- **关键命令**：`Get-Process | Where-Object {$_.ProcessName -like "*mcp*"}`

### 问题 B: 工具调用超时

- **可能原因**：后端 API（如 GitHub API）响应慢、网络问题、API 限流
- **排查思路**：
  1. 直接调用后端 API 确认响应时间
  2. 检查 API Token 是否有效
  3. 查看是否触发了 Rate Limiting
- **关键命令**：`curl -v https://api.github.com/rate_limit`

### 问题 C: 模型未调用预期的工具

- **可能原因**：工具描述不够清晰、模型判断不需要工具、工具列表未正确注入
- **排查思路**：
  1. 检查工具的 name 和 description 是否准确描述了功能
  2. 确认 Host 已将工具列表传给 LLM
  3. 尝试在 prompt 中明确要求使用某个工具

---

## 6. 实战经验 (Practical Tips)

### 最佳实践
- **工具描述要精确**：LLM 完全依赖工具的 name + description 来决定是否调用，模糊的描述会导致工具被忽略或误用
- **Server 职责单一**：每个 MCP Server 专注一个领域（如 GitHub、文件系统、数据库），不要做"万能 Server"
- **使用 stdio 传输**：本地场景优先使用 stdio，性能最好且无网络开销
- **注意 stdout 污染**：stdio 模式下，Server 代码中**绝不能向 stdout 打印**，否则会破坏 JSON-RPC 消息流

### 常见误区
- ❌ "MCP Server 是一台服务器" → 它是一个程序/进程，通常运行在你本地
- ❌ "LLM 直接连接 MCP Server" → LLM 完全不知道 MCP 的存在，由 Host 居中调度
- ❌ "MCP Server 存储数据" → Server 只是适配器/桥梁，数据在后端系统中
- ❌ "MCP 只能用 Python" → MCP SDK 支持 Python、TypeScript、C#、Go、Java、Kotlin、Rust、Swift 等

### 安全注意
- MCP Server 可以执行敏感操作（如写文件、调用 API），Host 必须实施权限控制
- API Token 和凭证应通过环境变量传递，不要硬编码在 Server 代码中
- 远程 MCP Server 应使用 HTTPS + 认证

---

## 7. 与相关技术的对比 (Comparison with Related Technologies)

### MCP vs RAG

| 维度 | MCP | RAG |
|------|-----|-----|
| **本质** | 通信协议/标准 | 技术模式/架构 |
| **方向** | 双向：请求 ↔ 响应 | 单向：检索 → 生成 |
| **能力** | 可以**读写和执行操作** | 只能**读取**信息 |
| **数据源** | 任何工具/API/服务 | 向量数据库、文档库 |
| **典型场景** | 调用 GitHub API、操作数据库、执行命令 | 知识问答、文档搜索 |
| **关系** | 可以作为 RAG 的传输层 | 可以封装为 MCP Server 的一个工具 |

> **一句话**：RAG 解决"让 AI 知道更多"，MCP 解决"让 AI 做到更多"。两者互补，可组合使用——用 MCP 协议调用 RAG 检索服务。

```
┌─────────────────────────────────────────────────┐
│  MCP Host (AI 应用)                              │
│                                                  │
│  MCP Client A ──→ MCP Server: 向量数据库          │
│                   （封装 RAG 检索能力）      ← RAG │
│                                                  │
│  MCP Client B ──→ MCP Server: GitHub             │
│                   （代码搜索、创建 Issue）          │
│                                                  │
│  MCP Client C ──→ MCP Server: 文件系统            │
│                   （读写本地文件）                  │
└─────────────────────────────────────────────────┘
```

### MCP vs 传统 Function Calling

| 维度 | MCP | 传统 Function Calling |
|------|-----|-----------------------|
| **标准化** | 开放协议，跨应用通用 | 每个 LLM 厂商自定义格式 |
| **工具发现** | 动态发现（Server 自报能力） | 静态定义（硬编码在代码中） |
| **可复用性** | 一个 Server 可被多个 Host 使用 | 通常绑定特定应用 |
| **生态系统** | 社区共建，数百个现成 Server | 需要自己开发每个集成 |

---

## 8. 代码示例 (Code-Level Example)

### 8.1 MCP Server（能力提供者）

```python
# server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("weather-server")

@mcp.tool()
async def get_weather(city: str) -> str:
    """Get current weather for a city."""
    return f"Weather in {city}: 22°C, Sunny"

@mcp.tool()
async def get_forecast(city: str, days: int = 3) -> str:
    """Get weather forecast for upcoming days."""
    return f"{days}-day forecast for {city}: Sunny → Cloudy → Rain"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### 8.2 MCP Client（协议桥梁）

```python
# client.py
from mcp.client import Client
from mcp.client.stdio import stdio_client, StdioServerParameters

async def create_client():
    server_params = StdioServerParameters(
        command="python", args=["server.py"],
    )

    async with stdio_client(server_params) as (read, write):
        client = Client("my-client", "1.0.0")
        await client.initialize(read, write)  # 能力协商

        tools = await client.list_tools()           # 发现工具
        result = await client.call_tool(            # 调用工具
            "get_weather", {"city": "Seattle"}
        )
        return result  # "Weather in Seattle: 22°C, Sunny"
```

### 8.3 MCP Host（AI 应用 — 完整 Turn 编排）

```python
# host.py
import json
from mcp.client import Client
from mcp.client.stdio import stdio_client, StdioServerParameters
from openai import OpenAI

class MCPHost:
    def __init__(self):
        self.clients: dict[str, Client] = {}
        self.all_tools: list = []
        self.llm = OpenAI()

    async def connect_server(self, name, command, args):
        """创建 MCP Client 并连接到 MCP Server"""
        params = StdioServerParameters(command=command, args=args)
        transport = await stdio_client(params).__aenter__()
        client = Client(name, "1.0.0")
        await client.initialize(*transport)
        self.clients[name] = client
        self.all_tools.extend(await client.list_tools())

    def tools_as_openai_format(self):
        """将 MCP 工具转换为 LLM 的 function calling 格式"""
        return [{
            "type": "function",
            "function": {
                "name": t.name,
                "description": t.description,
                "parameters": t.inputSchema,
            },
        } for t in self.all_tools]

    async def route_tool_call(self, tool_name, args):
        """将工具调用路由到正确的 MCP Server"""
        for name, client in self.clients.items():
            server_tools = await client.list_tools()
            if any(t.name == tool_name for t in server_tools):
                return await client.call_tool(tool_name, args)

    async def chat_turn(self, user_message: str):
        """一个完整的 Turn"""
        messages = [{"role": "user", "content": user_message}]

        # Step 1: 发送给 LLM（携带 MCP 工具列表）
        response = self.llm.chat.completions.create(
            model="gpt-4", messages=messages,
            tools=self.tools_as_openai_format(),
        )
        choice = response.choices[0].message

        # Step 2: 如果 LLM 决定调用工具
        if choice.tool_calls:
            for tc in choice.tool_calls:
                tool_name = tc.function.name
                tool_args = json.loads(tc.function.arguments)

                # Step 3: 通过 MCP Client → MCP Server 执行
                result = await self.route_tool_call(tool_name, tool_args)

                # Step 4: 将结果传回 LLM
                messages.append(choice.model_dump())
                messages.append({
                    "role": "tool",
                    "tool_call_id": tc.id,
                    "content": str(result),
                })

            # Step 5: LLM 基于工具结果生成最终回复
            final = self.llm.chat.completions.create(
                model="gpt-4", messages=messages,
            )
            return final.choices[0].message.content

        return choice.content

# 使用示例
async def main():
    host = MCPHost()
    await host.connect_server("weather", "python", ["server.py"])
    answer = await host.chat_turn("What's the weather in Seattle?")
    print(answer)  # "It's 22°C and sunny in Seattle!"
```

---

## 9. 参考资料 (References)

- [MCP Introduction — modelcontextprotocol.io](https://modelcontextprotocol.io/introduction) — MCP 官方入门介绍
- [MCP Architecture Specification](https://modelcontextprotocol.io/specification/2025-03-26/architecture) — MCP 架构规范（Client-Host-Server 详细定义）
- [MCP Quickstart: Building a Server](https://modelcontextprotocol.io/quickstart/server) — 官方 Server 开发快速入门教程
- [MCP Python SDK — GitHub](https://github.com/modelcontextprotocol/python-sdk) — Python SDK 官方仓库，含完整 API 文档
- [MCP Reference Servers — GitHub](https://github.com/modelcontextprotocol/servers) — 官方参考实现和社区 Server 集合

---
---

# Deep Dive: Model Context Protocol (MCP) — Architecture, Internals & Network Data Flow

**Topic:** Model Context Protocol (MCP)  
**Category:** AI / Protocol / Architecture  
**Level:** Intermediate  
**Last Updated:** 2026-03-18

---

## 1. Overview

Model Context Protocol (MCP) is an **open standard protocol** introduced by Anthropic for connecting AI models (LLMs) with external data sources and tools. Built on JSON-RPC, it provides a stateful session protocol focused on context exchange and sampling coordination.

The core problem MCP solves is: **how AI applications can connect to external systems in a standardized, secure, and scalable way**. Before MCP, every AI application needed custom integration code for every external tool, creating M×N integration complexity. MCP reduces this to M+N — each AI application implements one MCP Client, each tool implements one MCP Server, and they can interoperate.

In the broader AI technology stack, MCP sits at the **protocol layer between the AI application layer and the external capability layer**, similar to how HTTP works for the Web — it defines the communication standard without concerning itself with specific business logic.

> **Analogy: MCP is like the USB port of the AI world** — it allows different AI models to "plug in" to various external capabilities using a unified standard, without writing custom integrations for each tool.

---

## 2. Core Concepts

### 2.1 MCP Host

The MCP Host is the **outermost application** in the MCP architecture — the AI application users directly interact with.

- **Role**: Container and Coordinator
- **Responsibilities**:
  - Creates and manages multiple MCP Client instances
  - Controls Client connection permissions and lifecycle
  - Enforces security policies and user authorization
  - Coordinates AI/LLM integration and context aggregation
- **Common examples**: Claude Desktop, VS Code + Copilot, GitHub Copilot CLI, Cursor

```
User ↔ MCP Host (AI Application)
            ├── MCP Client 1 ↔ MCP Server A (GitHub)
            ├── MCP Client 2 ↔ MCP Server B (Filesystem)
            └── MCP Client 3 ↔ MCP Server C (Database)
```

> **Analogy**: If MCP is the USB standard, the Host is the computer itself — it provides USB ports (Clients) for you to plug in peripherals (Servers).

### 2.2 MCP Client

The MCP Client is the **middle layer** in the MCP architecture, responsible for establishing and maintaining connections between Host and Server.

- **Role**: Protocol Bridge
- **Key characteristics**:
  - Maintains a **1:1 stateful session** with an MCP Server
  - Handles protocol negotiation and capability exchange
  - Bidirectional protocol message routing
  - Manages subscriptions and notifications
  - Maintains security isolation between Servers
- **Lifecycle**: Created and managed by the Host; does not exist independently

> **Analogy**: The Client is the USB port and driver — you don't interact with it directly, but devices can't connect without it.

### 2.3 MCP Server

The MCP Server is the **capability provider** in the MCP architecture, exposing specific tools, data, and functionality for AI models to use.

- **Role**: Focused, independent capability service
- **Exposes three types of primitives**:

| Primitive Type | Description | Examples |
|---------------|-------------|----------|
| **Tools** | Functions the AI can invoke | `search_repositories`, `create_issue` |
| **Resources** | Data sources the AI can read | File contents, database records |
| **Prompts** | Predefined prompt templates | Code review templates, summarization templates |

**Critical clarification: An MCP Server is not a physical machine. It's a software program/process.** It typically runs on your local computer and usually doesn't store data itself — it acts as a **bridge/adapter** between the AI and the actual data backend.

| MCP Server | What it stores | Where data actually lives |
|------------|---------------|--------------------------|
| github-mcp-server | Only a GitHub Token | Data lives on github.com |
| postgres-mcp-server | Only a connection string | Data lives in PostgreSQL |
| filesystem-mcp-server | Nothing at all | Data lives on local disk |

> **Analogy**: Servers are the USB drives, keyboards, and webcams — they provide specific functionality, plug in and go.

### 2.4 Relationship Between MCP Server and LLM

**Key point: There is no direct connection between MCP Server and the LLM. The Host mediates all communication.**

```
LLM Model ←── ✕ No direct link ✕ ──→ MCP Server
     ↑                                   ↑
     └───── Host orchestrates ────────────┘
```

| | LLM Model | MCP Server |
|--|-----------|------------|
| What it does | Understands questions, reasons, decides whether to use tools | Provides tools, executes specific operations |
| Where it lives | Usually in the cloud (OpenAI/Azure) | Usually on your local machine |
| Knows the other? | Only knows "available tools" (name + description) | **Completely unaware** of the model's existence |

To the LLM, **MCP tools are indistinguishable from regular function calling tools**. MCP is an implementation detail on the Host side — the model is completely unaware of it.

---

## 3. How It Works

### 3.1 Overall Architecture

```
┌─────────────────────────────────────────────────────┐
│  MCP Host (AI App, e.g., Copilot CLI)               │
│  ┌───────────────────────────────────────────────┐  │
│  │  MCP Client 1 (1:1 connection)                │  │
│  └──────────────────┬────────────────────────────┘  │
│  ┌───────────────────┼──────────────────────────┐   │
│  │  MCP Client 2     │                          │   │
│  └──────────────────┬┼──────────────────────────┘   │
└─────────────────────┼┼──────────────────────────────┘
                      ││  stdio / HTTP+SSE (Transport)
┌─────────────────────┼┼──────────────────────────────┐
│  MCP Server A       ▼│                              │
│  [tool_1] [tool_2] [resource]                       │
└─────────────────────────────────────────────────────┘
┌──────────────────────┼──────────────────────────────┐
│  MCP Server B        ▼                              │
│  [tool_3] [prompt_1]                                │
└─────────────────────────────────────────────────────┘
```

### 3.2 Transport Modes

| Transport | Communication | Use Case | Example |
|-----------|--------------|----------|---------|
| **stdio** | Standard input/output pipes | Local processes; Host spawns Server as child process | github-mcp-server, filesystem-mcp-server |
| **Streamable HTTP** | HTTP + Server-Sent Events | Remote services, cross-network | Cloud-hosted MCP Servers |

### 3.3 A Complete Turn — Step by Step

Example: User asks "Search GitHub for MCP projects"

**Step 1: User sends message**
```
User input → Copilot CLI (Host) receives it (purely local)
```

**Step 2: Host sends request to cloud LLM**
```
POST https://api.githubcopilot.com/chat/completions
Body: {
  messages: [conversation history + user message],
  tools: [tool list obtained from MCP Servers],  ← MCP tools injected here
  model: "claude-opus-4.6-1m"
}
★ Outbound HTTPS traffic → Cloud
```

**Step 3: LLM reasons and decides**
```
Cloud LLM analyzes the question, decides to call search_repositories
Returns: {tool_calls: [{name: "search_repositories", args: {query: "MCP"}}]}
★ Inbound HTTPS traffic ← Cloud
```

**Step 4: Host routes request to MCP Server**
```
Host → MCP Client → stdio pipe → github-mcp-server (local process)
★ No network traffic (local inter-process communication)
```

**Step 5: MCP Server executes actual operation**
```
github-mcp-server → GET https://api.github.com/search/repositories?q=MCP
★ Outbound HTTPS traffic → GitHub API
```

**Step 6: Results return up the chain**
```
GitHub API → github-mcp-server → stdio → MCP Client → Host
★ Only GitHub API response involves network traffic
```

**Step 7: Host sends tool results back to LLM**
```
POST https://api.githubcopilot.com/chat/completions
Body: { messages: [..., tool execution results] }
★ Outbound HTTPS traffic → Cloud
```

**Step 8: LLM generates final response**
```
LLM generates natural language answer based on tool results → streams back to user
★ Inbound HTTPS traffic ← Cloud
```

### 3.4 Network Traffic Overview

```
Your Machine (Windows)                Internet                    Cloud
━━━━━━━━━━━━━━━━━━━                ━━━━━━━━━━━              ━━━━━━━━━━━━━━━

┌─────────────────┐                                   ┌──────────────────┐
│  Copilot CLI    │ ── HTTPS ───────────────────────→ │  GitHub Copilot  │
│  (MCP Host)     │    message + tool definitions      │  API Server      │
│                 │                                    │  ┌────────────┐  │
│                 │ ← HTTPS (streaming) ────────────── │  │ LLM Model  │  │
│                 │    model reply / tool call          │  └────────────┘  │
│                 │                                    └──────────────────┘
│  ┌───────────┐  │
│  │MCP Client │──┼── stdio (local pipe) ──┐
│  └───────────┘  │                        ▼
└─────────────────┘             ┌─────────────────────┐
                                │  github-mcp-server   │
                                │  (local process)     │
                                │  → HTTPS ──────────────→ api.github.com
                                │  ← HTTPS ──────────────← api.github.com
                                └─────────────────────┘
```

| Scenario | Round-trips | Endpoints involved |
|----------|------------|-------------------|
| Pure chat (no tools) | 1 Copilot API round-trip | Copilot API only |
| 1 tool call | 2 Copilot API + 1 tool API | Copilot API + github.com |
| N tool calls | At least 2 Copilot API + N tool APIs | Multiple endpoints |

**Key fact: There is no LLM on your local machine (models are typically hundreds of GBs). All inference happens in the cloud. No internet = no functionality.**

---

## 4. Key Configurations

| Config | Description | Typical Value | Tuning Scenario |
|--------|-------------|---------------|-----------------|
| **Transport** | Server-Client communication method | `stdio` / `streamable-http` | Use stdio locally, HTTP for remote |
| **Server command** | How Host launches the Server process | `python server.py` | Configure for different language runtimes |
| **Capability negotiation** | Features exchanged during Client-Server init | tools, resources, prompts | Enable/disable specific capabilities |
| **Security policy** | Host controls which tools are available | Allowlist / Blocklist | Restrict sensitive operations |
| **Timeout** | Tool invocation timeout | 30s-120s | Increase for long-running tools |

---

## 5. Common Issues & Troubleshooting

### Issue A: MCP Server won't connect
- **Possible causes**: Server process not started, stdio pipe broken, Server crashed
- **Investigation**:
  1. Check if Server process is running
  2. Review Server's stderr output (logs)
  3. Verify Server launch command and path are correct

### Issue B: Tool call timeout
- **Possible causes**: Backend API slow, network issues, API rate limiting
- **Investigation**:
  1. Call backend API directly to confirm response time
  2. Check if API token is valid
  3. Check for rate limiting

### Issue C: Model doesn't call expected tool
- **Possible causes**: Tool description unclear, model decided tool isn't needed, tool list not properly injected
- **Investigation**:
  1. Review tool name and description for accuracy
  2. Confirm Host passed tool list to LLM
  3. Try explicitly requesting tool usage in the prompt

---

## 6. Practical Tips

### Best Practices
- **Write precise tool descriptions**: The LLM relies entirely on name + description to decide whether to invoke a tool
- **Keep Servers focused**: Each MCP Server should cover one domain (GitHub, filesystem, database) — avoid "god Servers"
- **Prefer stdio transport**: For local scenarios, stdio has the best performance with zero network overhead
- **Never print to stdout in stdio mode**: This corrupts JSON-RPC messages and breaks the Server

### Common Misconceptions
- ❌ "MCP Server is a physical server" → It's a program/process, usually running on your local machine
- ❌ "LLM connects directly to MCP Server" → LLM is completely unaware of MCP; the Host orchestrates everything
- ❌ "MCP Server stores data" → Server is just an adapter/bridge; data lives in backend systems
- ❌ "MCP only works with Python" → SDKs available for Python, TypeScript, C#, Go, Java, Kotlin, Rust, Swift, and more

### Security Considerations
- MCP Servers can execute sensitive operations (write files, call APIs); Hosts must implement access control
- API tokens and credentials should be passed via environment variables, never hardcoded
- Remote MCP Servers should use HTTPS + authentication

---

## 7. Comparison with Related Technologies

### MCP vs RAG

| Dimension | MCP | RAG |
|-----------|-----|-----|
| **Nature** | Communication protocol/standard | Technical pattern/architecture |
| **Direction** | Bidirectional: request ↔ response | Unidirectional: retrieve → generate |
| **Capability** | Read, write, and execute operations | Read-only information retrieval |
| **Data sources** | Any tool/API/service | Vector databases, document stores |
| **Typical use** | Call GitHub API, operate databases, execute commands | Knowledge Q&A, document search |
| **Relationship** | Can serve as RAG's transport layer | Can be wrapped as an MCP Server tool |

> **In one sentence**: RAG solves "making AI know more", MCP solves "making AI do more". They're complementary and can be combined — use MCP protocol to invoke RAG retrieval services.

### MCP vs Traditional Function Calling

| Dimension | MCP | Traditional Function Calling |
|-----------|-----|------------------------------|
| **Standardization** | Open protocol, cross-application | Vendor-specific formats |
| **Tool discovery** | Dynamic (Servers self-report capabilities) | Static (hardcoded in application) |
| **Reusability** | One Server can be used by many Hosts | Typically bound to specific applications |
| **Ecosystem** | Community-built, hundreds of ready-made Servers | Must build each integration yourself |

---

## 8. References

- [MCP Introduction — modelcontextprotocol.io](https://modelcontextprotocol.io/introduction) — Official MCP introduction
- [MCP Architecture Specification](https://modelcontextprotocol.io/specification/2025-03-26/architecture) — MCP architecture spec (detailed Client-Host-Server definitions)
- [MCP Quickstart: Building a Server](https://modelcontextprotocol.io/quickstart/server) — Official Server development quickstart tutorial
- [MCP Python SDK — GitHub](https://github.com/modelcontextprotocol/python-sdk) — Python SDK official repository with full API docs
- [MCP Reference Servers — GitHub](https://github.com/modelcontextprotocol/servers) — Official reference implementations and community Server collection
