---
layout: post
title: "🔵 Function Calling：AI 如何调用工具——从"会说"到"会做"的关键一步"
date: 2026-08-08
categories: [AI-Learning]
tags: [function-calling, tool-use, agent]
type: "ai-learning"
episode: 31
---

# 🔵 Function Calling：AI 如何调用工具——从"会说"到"会做"的关键一步

> 一个只会说话的 AI 是聊天机器人。
> 一个会"调工具、查接口、做事"的 AI 是**Agent**。
> 中间那一步，叫 **Function Calling**。

🔵 **AI 通识专栏 · 第 31 期**
📅 **2026年8月8日**
✍️ **小敏说 AI**

> 💡 **本文要点**：本文讲清 Function Calling 是什么、它如何让 AI 真正"动起来"、典型流程、5 个真实场景，以及为什么它是通往 Agent 的必经之路。

---

## 🎯 本文导读

- 🔹 一句话理解 Function Calling
- 🔹 没有 vs 有 Function Calling
- 🔹 完整调用流程
- 🔹 5 个真实场景
- 🔹 它和 MCP 的关系

---

## 一、🛠 一句话理解

**Function Calling = LLM 在回答时，知道"该调用某个外部函数"，并按约定格式返回参数。**

举个例子：

```
用户："上海明天天气怎么样？"

无 FC：AI 编一个"上海明天 25 度晴" → 大概率是瞎说
有 FC：AI 返回 {function: "get_weather", args: {city: "上海", date: "明天"}}
       → 程序真的去查接口 → 把结果再喂回 AI → AI 总结回复
```

> 💬 **关键观点**：**FC 让 LLM 从"文字生成器"变成"决策中枢"**。

---

## 二、⚖️ 没有 FC vs 有 FC

| 能力 | 没有 FC | 有 FC |
|---|---|---|
| 查实时天气 | 编 | 真查 |
| 查数据库 | 编 | 真查 |
| 发邮件 | 不会 | 调 SMTP |
| 算复杂数学 | 错 | 调计算器 |
| 控制智能家居 | 不会 | 调 API |

---

## 三、🔁 完整流程

```
1. 你定义一组函数（带名字 + 参数 schema）
2. 用户提问
3. LLM 判断：是否需要调用函数？
   - 不需要 → 直接回答
   - 需要 → 返回函数名 + 参数
4. 你的程序真的执行该函数
5. 把结果回传 LLM
6. LLM 基于结果给最终回复
```

> 🎯 **一句话总结**：**LLM 负责"想"，你的代码负责"做"**。

---

## 四、💼 5 个真实场景

1. **天气助手** —— 实时查询
2. **企业内部问答** —— 查 ERP / CRM 数据
3. **AI 下单系统** —— 帮用户调电商接口
4. **会议助手** —— 自动建日历、发邀请
5. **AI 客服** —— 查订单、改地址、申请退款

---

## 五、🧪 一段最简代码（伪代码）

```python
tools = [{
  "name": "get_weather",
  "description": "查询某城市某天天气",
  "parameters": {"city": "string", "date": "string"}
}]

response = llm.chat(messages, tools=tools)

if response.tool_call:
  result = my_weather_api(**response.tool_call.args)
  final = llm.chat(messages + [tool_result(result)])
```

40 行代码，一个真正"会查天气"的 AI 就跑起来了。

---

## 六、🔗 FC 和 MCP 的关系

```
Function Calling = AI ↔ 单个函数
        ↓
MCP（Model Context Protocol）= AI ↔ 一整套工具/资源的"标准插头"
```

> ⚠️ **重要提醒**：FC 是"会做事"的起点；**MCP 是把"做事"标准化、跨厂商可复用**。下一篇 ep33 详讲。

---

## 七、❌ 常见误区

| 误区 | 真相 |
|---|---|
| ❌ FC 是 OpenAI 特性 | ✅ 主流大模型都支持 |
| ❌ 有 FC 就不会幻觉 | ✅ 仍可能传错参数 |
| ❌ 函数定义越多越好 | ✅ 函数太多模型会蒙 |
| ❌ FC = Agent | ✅ FC 是 Agent 的组件之一 |

---

## 八、🎁 给非技术读者的 FC 心法

```
🔑 FC 让 AI "动手"，而不只是"动嘴"
🔑 它是企业 AI 应用 70% 的实现方式
🔑 它解释了为什么 Siri 终于变聪明了
🔑 它是 Agent 时代的入场券
```

📮 **今日话题**：你最希望 AI 能"代你做"的一件事是什么？

> 🔮 **下一篇预告**：ep32《Agent 是什么？从 ReAct 到 AutoGPT——AI 自主完成任务的真相》

---

🔵 本系列是「小敏说 AI · AI 通识专栏」第 31 期。
关注公众号 **小敏说 AI**，回复「AI 入门」领取《打工人 AI 自救工具箱》。
