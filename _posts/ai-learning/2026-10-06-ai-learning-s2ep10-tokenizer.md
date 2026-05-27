---
layout: post
title: "🟣 Tokenizer 深挖：BPE / Tiktoken / 中文分词"
date: 2026-10-06
categories: [AI-Learning]
tags: [tokenizer, bpe, chinese]
type: "ai-learning"
episode: 56
season: 2
---

# 🟣 Tokenizer 深挖：BPE / Tiktoken / 中文分词

> 同一句中文，
> GPT-4 吃 30 个 Token，
> 国产模型吃 12 个。
> **同样的提问，价格能差 2-3 倍**。
> 这背后是一个被严重低估的组件——Tokenizer。

🟣 **AI 深度派专栏 · 第 S2-10 期**
📅 **2026年10月6日**
✍️ **小敏说 AI**

> 💡 **本文要点**：本文讲清 Tokenizer 是什么、BPE 算法原理、Tiktoken / SentencePiece 差异、中文 Tokenizer 的进化、它对成本与能力的真实影响。

---

## 🎯 本文导读

- 🔹 Tokenizer 在 LLM 里的位置
- 🔹 BPE 算法原理
- 🔹 Tiktoken vs SentencePiece
- 🔹 中文 Tokenizer 的演进
- 🔹 它如何影响成本与能力

---

## 一、🧩 Tokenizer 在 LLM 里的位置

```
"我爱 AI" 
   ↓ Tokenizer
[1234, 5678, 9012]  # 数字 ID
   ↓ Embedding
[向量, 向量, 向量]
   ↓ Transformer
[预测]
```

> 💬 **关键观点**：**Tokenizer 是 LLM 的"翻译官"**——把人话变成数字、把数字变回人话。

它的设计直接决定：
- 同一段文字"几个 Token"
- 模型"看到"什么粒度
- 多语言 / 代码 / Emoji 怎么处理

---

## 二、🔤 BPE 算法（Byte Pair Encoding）

最主流算法，源自 1994 年压缩算法。

### 训练流程
```
1. 初始词表 = 所有字符（含字节）
2. 找出语料里"最常一起出现的两个 Token 对"
3. 合并为新 Token，加入词表
4. 重复，直到词表达到目标大小（如 50000 / 100000 / 200000）
```

### 例子（英文）
```
初始：l, o, w, e, r
高频共现：l+o → "lo"
继续：lo+w → "low"
继续：low+er → "lower"
最终："lower" 变成 1 个 Token
```

✅ **效果**：高频词整体 = 1 Token；罕见词拆成多个 sub-word。

---

## 三、⚖️ Tiktoken vs SentencePiece

| 工具 | 出品 | 谁在用 |
|---|---|---|
| **Tiktoken** | OpenAI | GPT-3/4/5 |
| **SentencePiece** | Google | Llama / Qwen / Gemini |
| **HuggingFace Tokenizers** | HF | 通用 Rust 实现 |

### 关键差异
| 维度 | Tiktoken | SentencePiece |
|---|---|---|
| 预处理 | 复杂正则切分 | 直接 byte 级 |
| 空格处理 | 把空格拼到下一个词 | 显式标 `▁` |
| 多语言友好 | 较弱 | **强** |
| 训练简单 | 较难 | 容易 |

> 🎯 **一句话总结**：**SentencePiece 多语言场景更友好**——这就是国产模型选它的核心原因。

---

## 四、🇨🇳 中文 Tokenizer 的演进

### 早期（GPT-3.5 / GPT-4）
- 中文一个字常拆成 **2-3 个 Token**
- 同样一句话，**比英文贵 2-3 倍**
- 上下文窗口实际"短一截"

### 改进路径
| 模型 | 改进 |
|---|---|
| GPT-4o | 中文 Token 压缩约 1.4× |
| Llama 3 | 词表扩到 128K，中文友好 |
| Qwen / DeepSeek | **专门为中英双语训练 Tokenizer** |
| Yi / GLM | 中文优化版 |

### 实测对比
同一段 1000 字中文文章：

| 模型 | Token 数 |
|---|---|
| GPT-3.5（cl100k_base） | ~1500 |
| GPT-4o（o200k_base） | ~1000 |
| Llama 3 | ~900 |
| Qwen / DeepSeek | **~600** |

> 💬 **关键观点**：**Token 数差 2-3 倍 = 上下文能力 / 成本差 2-3 倍**。

---

## 五、💸 它如何影响成本与能力

| 维度 | 解释 |
|---|---|
| **API 费用** | 按 Token 计费，少 Token = 便宜 |
| **上下文有效长度** | 128K Token，英文能装 90 页、中文只能 50 页 |
| **训练效率** | 同样语料，Token 少 = 训得快 |
| **生成速度** | Token 少 = 生成快 |
| **代码 / 数学** | 不同 Tokenizer 对代码碎片化程度不同 |

---

## 六、🪓 Tokenizer 引发的"奇怪 bug"

### bug 1：数字处理
GPT 早期对长数字 Tokenize 怪异，**导致数学算错**。后来强制"按位拆分"才修。

### bug 2：SolidGoldMagikarp
Reddit 早期挖到的"幽灵 Token"——训练数据里只出现一两次，模型对它会**异常行为**。

### bug 3：多语言不公平
非英语用户支付更多 Token 费——**这是一种事实上的"语言税"**。

---

## 七、❌ 常见误区

| 误区 | 真相 |
|---|---|
| ❌ Tokenizer 是次要细节 | ✅ 影响 30% 推理成本 |
| ❌ 1 个 Token = 1 个字 | ✅ 中文常 0.5-2 个字 / Token |
| ❌ 词表越大越好 | ✅ Embedding 表也大 |
| ❌ 训完就完事 | ✅ 新数据出现要更新 |

---

## 八、🎁 给非工程读者的"Tokenizer 心法"

```
🔑 看一个模型对中文友好 → 看 Tokenizer
🔑 同样的钱，国产模型常买到 2× 中文上下文
🔑 用 tiktoken 库可以本地估算 Token 数
🔑 极端格式（JSON / 代码）尤其要看 Token 化效率
```

📮 **今日话题**：你算过自己用 AI 一年总共"花在 Token 上"多少钱吗？

> 🔮 **下一篇预告**：s2ep11《Embedding 深度：从 word2vec 到 contrastive》

---

🟣 本系列是「小敏说 AI · AI 深度派专栏」第 S2-10 期。
关注公众号 **小敏说 AI**，回复「深度派」领取第二季合集。
