---
layout: post
title: "⚙️ 推理工程：vLLM / TensorRT / SGLang"
date: 2026-11-05
categories: [AI-Learning]
tags: [inference, vllm, tensorrt, sglang]
type: "ai-learning"
episode: 69
season: 2
---

# ⚙️ 推理工程：vLLM / TensorRT / SGLang

> 同样的卡、同样的模型，
> 推理性能差 5-10 倍——
> **差距全在推理引擎**。
> 这一篇拆 2026 年三大主流方案。

⚙️ **AI 深度派专栏 · 第 S2-23 期**
📅 **2026年11月5日**
✍️ **小敏说 AI**

> 💡 **本文要点**：本文讲清 LLM 推理的核心瓶颈（KV Cache + batch），介绍 vLLM 的 PagedAttention、TensorRT-LLM 的极致优化、SGLang 的 RadixAttention，以及该如何选型。

---

## 🎯 本文导读

- 🔹 推理性能的两个维度
- 🔹 vLLM：开源王者
- 🔹 TensorRT-LLM：极致优化
- 🔹 SGLang：新晋黑马
- 🔹 选型指南

---

## 一、📐 推理性能的两个维度

```
延迟（Latency）：单请求多快出第一个 token
吞吐（Throughput）：一秒能处理多少 token / 多少请求
```

通常**鱼与熊掌不可兼得**：
- 大 batch → 吞吐高 + 延迟高
- 小 batch → 延迟低 + 吞吐低

推理引擎的核心目标：**在保延迟前提下最大化吞吐**。

---

## 二、🧠 关键瓶颈：KV Cache + Batch

回顾 s2ep05：
- KV Cache 占大头显存
- 每个请求长度不一，**显存碎片化严重**
- 传统调度：**预留最大长度 → 极度浪费**

> 💬 **关键观点**：**LLM 推理的真正瓶颈，是"显存调度"而不是"算力"**。

---

## 三、🚀 vLLM：PagedAttention 开源王者

UC Berkeley 出品（Sky Computing Lab），现在是**开源事实标准**。

### 核心创新：PagedAttention
借鉴操作系统**虚拟内存分页**：

```
KV Cache 切成固定大小的"块"（page）
不同请求按需分块申请
彻底消除碎片
```

✅ **吞吐量 vs HF Transformers 提升 10-24×**
✅ 支持 continuous batching（动态拼 batch）
✅ 多模型 / 多 LoRA / Speculative Decoding 都有

### 谁在用
- 几乎所有开源服务商
- 字节、阿里、腾讯部分内部
- HuggingFace TGI 也借鉴

---

## 四、💪 TensorRT-LLM：NVIDIA 官方极致

NVIDIA 自家推理引擎，**性能巅峰**。

### 核心特性
- **kernel 级深度优化**（人手写的 PTX）
- **FP8 / INT8 / FP4 量化支持完整**
- **In-flight batching**（continuous batching）
- **Multi-LoRA、Speculative、KV reuse**
- **NVL72 / NVLink-Switch 优化**

### 优势 vs vLLM
- **同卡吞吐高 30-100%**
- 低延迟场景更稳

### 劣势
- 只支持 NVIDIA
- 不够"开箱即用"——要编译 engine
- 模型升级要重新转换

> 🎯 **一句话总结**：**vLLM 易用，TensorRT-LLM 极致**。

---

## 五、🌟 SGLang：黑马新势力

LMSYS（做 Chatbot Arena 那帮人）出品。

### 核心创新：RadixAttention
**自动复用 prompt 之间的 KV Cache**：

```
两个用户都问 "请把以下中文翻译成英文：xxx"
前缀完全相同 → 直接复用 KV Cache
```

✅ **多用户共享系统提示词**场景下，吞吐**翻 3-5 倍**
✅ Agent / 多轮对话 / RAG 受益巨大

### 其他特性
- 优秀的 control flow（结构化输出 / JSON 模式）
- 高效的多模态推理
- 大模型 + 小模型联合调度

### 谁在用
- **越来越多 LLM 服务商**
- DeepSeek 官方推荐
- Llama 3.1/4 都有官方支持

---

## 六、📊 三者对比

| 维度 | vLLM | TensorRT-LLM | SGLang |
|---|---|---|---|
| 上手难度 | ⭐ | ⭐⭐⭐ | ⭐⭐ |
| 通用吞吐 | 高 | **极高** | 高 |
| 共享前缀 | 一般 | 一般 | **极强** |
| 多 LoRA | 支持 | 支持 | 支持 |
| 多硬件 | NV/AMD/部分国产 | NV only | NV/AMD |
| 量化支持 | 多 | **极全** | 多 |
| 适用场景 | 通用部署 | 企业 NV 集群 | Agent / 长前缀 |

---

## 七、🛠 其他值得知道的

| 引擎 | 特点 |
|---|---|
| **HuggingFace TGI** | 简单易部署 |
| **DeepSpeed-Inference** | MS 出品，重训练背景 |
| **MLC-LLM** | 端侧 + WebGPU |
| **Ollama / llama.cpp** | 个人 + Mac |
| **lmdeploy** | 商汤，国内成熟 |

---

## 八、🎯 选型指南

| 你的场景 | 推荐 |
|---|---|
| 自建公司服务，要稳定 | **vLLM** |
| 已采购大量 H100/B200 集群 | **TensorRT-LLM** |
| 重 Agent、多轮、共享 prompt | **SGLang** |
| 个人 / Mac | Ollama / llama.cpp |
| 端侧 / 浏览器 | MLC-LLM |

---

## 九、❌ 常见误区

| 误区 | 真相 |
|---|---|
| ❌ "Transformers 跑模型"就够 | ✅ 吞吐差 10-20× |
| ❌ 引擎都差不多 | ✅ 场景适配差 2-5× |
| ❌ 引擎换上就快 | ✅ 仍要调参（batch / 量化） |
| ❌ 量化必掉精度 | ✅ FP8 几乎无损 |

---

## 十、🎁 给非工程读者的"推理工程心法"

```
🔑 推理速度 = 模型 × 硬件 × 引擎，三件套
🔑 vLLM 是默认起点
🔑 SGLang 是 Agent / 长前缀场景神器
🔑 TensorRT-LLM 是 NVIDIA 集群极限榨取
🔑 推理优化空间，往往比加卡更大
```

📮 **今日话题**：如果你正在跑 LLM 推理，目前用的哪个引擎？

> 🔮 **下一篇预告**：s2ep24《AI 网络：InfiniBand vs RoCE 的世纪对决》

---

⚙️ 本系列是「小敏说 AI · AI 深度派专栏」第 S2-23 期。
关注公众号 **小敏说 AI**，回复「深度派」领取第二季合集。
