---
layout: post
title: "🟣 多模态对齐：CLIP / SigLIP / 视觉 Token 化"
date: 2026-10-10
categories: [AI-Learning]
tags: [multimodal, clip, siglip, vision-token]
type: "ai-learning"
episode: 58
season: 2
---

# 🟣 多模态对齐：CLIP / SigLIP / 视觉 Token 化

> 一张图怎么"翻译"成 LLM 能听懂的话？
> 这背后的桥梁，叫 **多模态对齐（Vision-Language Alignment）**。
> 它有两条路线，决定了 2026 年所有多模态模型的样子。

🟣 **AI 深度派专栏 · 第 S2-12 期**
📅 **2026年10月10日**
✍️ **小敏说 AI**

> 💡 **本文要点**：本文讲清 CLIP 的开创性、SigLIP 的改进、把图像变成 LLM "Token"的两条主流路径（连接器 vs 早融合），以及主流多模态模型架构对比。

---

## 🎯 本文导读

- 🔹 多模态对齐到底是对齐什么
- 🔹 CLIP：双塔对比学习
- 🔹 SigLIP：用 Sigmoid 替换 Softmax
- 🔹 视觉 Token 化：两条路线
- 🔹 主流 VLM 架构对比

---

## 一、🎯 对齐到底是对齐什么

**对齐 = 让"图像"和"对应的文字"在同一个向量空间里靠近**。

```
🐶 一张狗的图
"a photo of a dog"

→ 这两个向量应该非常接近
```

> 💬 **关键观点**：**多模态的本质就是"同一意义、不同模态，应有相近的表示"**。

---

## 二、🌟 CLIP（OpenAI, 2021）：开创者

### 核心训练

收集 **4 亿对(图，文)**，搞个对比学习：

```
图编码器 → 图向量
文编码器 → 文向量

让"配对的"靠近，"不配对的"远离
```

Loss：InfoNCE（一种 Softmax 对比损失）。

### 三大影响
1. **零样本分类**："a photo of a {类别}" 即可分类，不用训练
2. **Diffusion 的灵魂** —— Stable Diffusion 用它做文本条件
3. **多模态 LLM 的视觉编码器** —— LLaVA、MiniGPT、Qwen-VL 都用过

---

## 三、🔥 SigLIP（Google, 2023）：CLIP 的改良

CLIP 的 Softmax 在大 batch 时**内存爆炸 + 不稳定**。

SigLIP 的核心改动：**用 Sigmoid 替换 Softmax**。

| 维度 | CLIP | SigLIP |
|---|---|---|
| Loss | Softmax InfoNCE | **Sigmoid 二分类** |
| Batch 依赖 | 强（要全 batch 比较） | **每对独立** |
| 训练 | 复杂 | **简单稳定** |
| 效果 | 强 | **更强** |

✅ 现在主流多模态模型（包括 PaliGemma、最新 Qwen-VL）都换成 SigLIP。

> 🎯 **一句话总结**：**SigLIP 是工程上更友好的 CLIP**——能用更小 batch 训出更好对齐。

---

## 四、🧩 怎么把图像喂给 LLM

LLM 只懂 Token。**图怎么变 Token？**

### 路线 1：连接器（Connector）
```
图 → 视觉编码器（CLIP / SigLIP）→ 视觉特征
     → MLP / Q-Former / Resampler → "视觉 Token"
     → 拼到文本 Token 一起 → LLM
```

代表：**LLaVA、Qwen-VL、MiniGPT-4**

✅ 简单、训练快、效果稳

### 路线 2：早融合 / Native 多模态
```
图直接切 Patch → 当作 Token → 喂给 LLM
（不用独立视觉编码器）
```

代表：**Fuyu、Chameleon、GPT-4o、Gemini Native**

✅ 更"原生"，能交叉生成
❌ 训练更难、数据要求更多

> 💬 **关键观点**：**两条路线 = "拼接派"vs"统一派"**。短期连接器更主流，长期 Native 是趋势。

---

## 五、📊 主流 VLM 架构对比

| 模型 | 视觉编码器 | 连接方式 | 路线 |
|---|---|---|---|
| **LLaVA 1.5** | CLIP | MLP | 连接器 |
| **Qwen2-VL** | ViT 自训 | MLP + 动态分辨率 | 连接器 |
| **GPT-4o** | 未公开 | **早融合** | Native |
| **Gemini 2** | 未公开 | **早融合** | Native |
| **Chameleon (Meta)** | 离散视觉 token | **早融合** | Native |

---

## 六、🪜 关键技术细节

### 动态分辨率
固定分辨率（如 224×224）会丢失大图细节。Qwen2-VL、GPT-4o 都支持**变长视觉 Token**。

### Resampler / Q-Former
把成千上百的视觉 Patch 压缩成几十个高语义 Token，**省 LLM 算力**。

### 高分辨率 OCR
读 PDF / 表格需要的"超清视觉 Token 化"。

---

## 七、❌ 常见误区

| 误区 | 真相 |
|---|---|
| ❌ 视觉编码器越强越好 | ✅ 还要看 connector 设计 |
| ❌ Native 一定取代连接器 | ✅ 短期连接器仍占主流 |
| ❌ CLIP 已过时 | ✅ 仍是基础设施 |
| ❌ Token 越多看得越清 | ✅ 太多会"稀释"语言 |

---

## 八、🎁 给非工程读者的"多模态对齐心法"

```
🔑 多模态本质 = "图与字"在同一向量空间
🔑 CLIP 开创、SigLIP 优化、Native 是未来
🔑 看一个 VLM 强不强，看 3 件事：
   - 视觉编码器
   - 连接方式
   - 分辨率支持
🔑 端到端 Native 路线 = "多模态原生"模型
```

📮 **今日话题**：你最希望多模态 AI 解决你的什么场景？

> 🔮 **下一篇预告**：s2ep13《训练数据工程：从 The Pile 到合成数据》

---

🟣 本系列是「小敏说 AI · AI 深度派专栏」第 S2-12 期。
关注公众号 **小敏说 AI**，回复「深度派」领取第二季合集。
