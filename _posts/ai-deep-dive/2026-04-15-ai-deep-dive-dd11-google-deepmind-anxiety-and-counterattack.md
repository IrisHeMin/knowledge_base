---
layout: post
title: "🔬 DD11: Google DeepMind：巨头的AI焦虑与反击"
date: 2026-04-15
categories: [AI-Deep-Dive]
tags: [google, deepmind, gemini, search, android, pichai]
type: "ai-deep-dive"
episode: 11
---

# 🔬 AI深度解析 DD11 — Google DeepMind：巨头的AI焦虑与反击

**预计时长：约25分钟**

---

## 🎤 开场

大家好，欢迎回到AI深度解析，我是小敏。

今天咱们聊一个特别拧巴的故事——Google的AI之路。

为什么说拧巴？你想想看：Google是全世界AI研究最强的公司之一。Transformer是Google发明的。AlphaGo是Google（DeepMind）做的。Google坐拥全球最大的搜索数据、最强的计算基础设施、最多的顶级AI研究员。

然而2022年11月，ChatGPT一出来，Google慌了。

一个成立才几年的OpenAI，用Google自己发明的Transformer架构，做了一个产品，直接威胁到了Google的核心——搜索业务。

这不是打脸，这是拿Google自己锻造的剑来刺Google。

今天我们来好好聊聊，Google到底是怎么走到这一步的，以及它的反击成效如何。

---

## 📖 第一章：两支队伍的故事

要理解Google的AI故事，得先认识两支团队。

**Google Brain**（2011年成立）：Google内部的AI研究团队。Jeff Dean、Geoffrey Hinton这些大神坐镇。2017年发了那篇改变世界的论文——"Attention is All You Need"，也就是Transformer架构的诞生。

**DeepMind**（2010年成立，2014年被Google以6.5亿美元收购）：Demis Hassabis领导的团队，做出了AlphaGo、AlphaFold等令人瞩目的成果。更偏基础研究和"通向AGI"。

这两支团队长期并行存在，关系说好听是"良性竞争"，说难听就是"内耗"。

他们各搞各的模型、各写各的论文、各有各的预算。Google高层一度认为内部竞争会带来更好的成果，但实际上它导致了资源分散和战略模糊。

2023年4月，Google终于做了一个早就该做的决定：**把Google Brain和DeepMind合并为"Google DeepMind"**，由Demis Hassabis统一领导。

这个合并意味着Google终于承认了：面对OpenAI这样的聚焦型对手，内部赛马的模式不管用了。

---

## 😱 第二章：ChatGPT引发的"红色警报"

2022年底ChatGPT发布时，Google内部的反应据说是"Code Red"（红色警报）。

这不是媒体夸张。你想想Google搜索广告是什么——它是Google整个商业帝国的根基，贡献了超过60%的收入。如果人们开始用ChatGPT来获取答案，而不是Google搜索，那就是存亡级别的威胁。

Sundar Pichai直接参与了应对策略的制定。Google的反应可以分为几个阶段：

**阶段一：急匆匆的回应（2023年2月）**

Google仓促发布了Bard，效果一言难尽。发布会上Bard犯了一个事实性错误（关于詹姆斯·韦伯太空望远镜的回答），直接导致Google股价暴跌超过1000亿美元市值。

这可能是Google历史上最尴尬的产品发布之一。

**阶段二：稳住阵脚（2023年下半年）**

Google意识到匆忙反击不行，开始系统性地整合内部AI力量。

**阶段三：Gemini登场（2023年12月-2024年）**

Google发布了Gemini系列模型，这是合并后Google DeepMind的第一个重磅成果。

| Gemini版本 | 特点 |
|-----------|------|
| Gemini 1.0 | 首发三个规格（Ultra/Pro/Nano） |
| Gemini 1.5 Pro | 百万级token上下文窗口，惊艳 |
| Gemini 2.0 | 多模态能力大幅提升 |
| Gemini 2.5 Pro | 推理能力追上竞品 |

要说Gemini的亮点，我觉得是**长上下文处理能力**。当其他模型还在纠结32K、128K上下文的时候，Gemini 1.5 Pro直接给了100万token的上下文窗口。这对于分析长文档、代码库这样的场景特别有用。

---

## 🔍 第三章：搜索的AI转型

Google最核心的战场当然是搜索。

2024年，Google在搜索中推出了**AI Overviews（AI概要）**——在搜索结果顶部用AI生成答案摘要。

这个决策的纠结程度可想而知：

- 如果不做AI概要，用户可能流向ChatGPT和Perplexity
- 如果做了AI概要，用户可能不再点击网页链接，搜索广告的点击率会下降

这就是经典的"创新者的窘境"——你的核心业务越成功，你越难自我颠覆。

从目前的数据看，Google搜索的市场份额实际上保持得还不错（仍在85%以上），AI搜索替代品（如Perplexity）的实际用量还很小。但趋势是明确的——年轻用户越来越多地用AI工具来获取信息。

我的看法是：Google搜索在未来5年内不会被颠覆，但会被逐步侵蚀。对Google来说，这个转型就像在高速行驶的汽车上换轮胎——你不能停下来，但你必须换。

---

## 📱 第四章：AI无处不在的Google生态

除了搜索，Google在把AI塞进自己的每一个产品：

**Android上的AI**：
- Gemini Nano在手机端运行，支持离线AI功能
- 替代了传统的Google Assistant
- 通话摘要、智能回复、实时翻译

**Google Workspace**：
- Gemini集成进Gmail、Docs、Sheets
- AI辅助写作、数据分析、演示文稿生成
- 但坦白说，采用率并没有Google希望的那么高

**NotebookLM——被低估的杀手级应用**：

这个我要单独说说。NotebookLM可能是Google AI产品里最被低估的一个。它能把你上传的文档变成一个AI知识库，甚至能生成播客式的对话。

很多人（包括我）觉得NotebookLM比Google的很多"大产品"更有实用价值。它代表了一种新的AI使用范式——不是通用的聊天机器人，而是基于你自己数据的专属AI助手。

**Google Cloud/Vertex AI**：
- 提供Gemini模型的API服务
- 在企业AI市场与Azure和AWS竞争
- 云收入增长强劲（年增长率超过30%）

---

## 🧬 第五章：AlphaFold和科学AI——Google的王牌

当所有人都在关注ChatGPT和大语言模型的时候，Google DeepMind在另一个方向上做了可能影响更深远的事情——**AlphaFold**。

AlphaFold2解决了蛋白质结构预测这个困扰生物学界50年的问题。AlphaFold3更进一步，能预测蛋白质与其他分子的相互作用。Demis Hassabis因此获得了2024年的诺贝尔化学奖。

**这不是夸张——一位AI公司的CEO因为AI研究成果获得诺贝尔奖。** 这本身就说明了Google DeepMind在科学AI领域的地位。

除了AlphaFold，Google DeepMind还在做：
- **AlphaGeometry**：数学几何问题的AI推理
- **GNoME**：材料科学发现
- **天气预测模型**：精度超过传统数值模型

这些可能不像ChatGPT那样吸引眼球，但它们代表了AI的另一个重要方向——**AI for Science**。如果说大语言模型解决的是"让AI像人一样聊天"，那科学AI解决的是"让AI推动人类知识的边界"。

在这个维度上，Google DeepMind是当之无愧的领导者。

---

## 🏢 第六章：大公司病——Google的结构性挑战

说了这么多优势，也得说说Google面临的结构性问题。

**1. 决策速度慢**

Google是一家超过18万员工的巨型公司。任何产品决策都要经过层层审批、多轮评审。一个在OpenAI可能一周能决定的事情，在Google可能要一个季度。

**2. 风险厌恶**

当你每年赚3000亿美元的广告收入时，你最大的恐惧不是错过新机会，而是搞砸现有业务。这种心态让Google在AI产品上过于保守——Bard（后来的Gemini聊天）发布了很多次都让人感觉"差口气"。

**3. 人才流失**

一个残酷的现实：Google培养的AI人才，很多最终去了OpenAI、Anthropic、或者创业。Transformer的发明者团队基本都不在Google了。Google成了AI行业的"黄埔军校"，但学员毕业后都去了竞争对手那里。

**4. 缺乏"明星产品"叙事**

OpenAI有ChatGPT，Anthropic有Claude。Google有什么？Gemini？Bard？Google搜索的AI概要？太分散了，没有一个产品能承载"这就是Google的AI"的叙事。

---

## 🔮 第七章：Google正在追赶还是落后？

这是一个看你从哪个角度看的问题。

**如果你看模型性能**：Gemini 2.5 Pro在很多基准测试上已经追平甚至超过了GPT-4o和Claude。Google没有落后。

**如果你看产品体验**：Google的AI产品给人的感觉总是"差一点点"。不够有惊喜感，不够有让人"wow"的时刻。这可能是最大的问题。

**如果你看基础设施**：Google的TPU芯片、数据中心、云计算能力是世界顶级的。长期来看，算力优势可能比模型技巧更重要。

**如果你看商业模式**：Google已经在从AI中赚钱了（通过云服务和广告优化），而OpenAI还在巨亏。

我的判断是：**Google不会输掉AI竞赛，但它可能无法像在搜索时代那样统治AI时代。** 未来的AI格局更可能是多强并立，而不是一家独大。Google会是其中一个强者，但不会是唯一的强者。

---

## 👋 结尾

好了，今天关于Google DeepMind的故事就讲到这里。

Google的AI故事让我想到一句话：**"拥有一切的人，往往是最害怕失去的人。"** Google拥有最好的技术、人才和基础设施，但正是因为现有业务太成功了，反而让它在AI的新浪潮中显得犹豫和保守。

下一期我们来看一个完全不同的故事——DeepSeek。一家从中国量化基金里走出来的AI公司，用极低的成本训练出了震惊全球的模型。它是怎么做到的？

我是小敏，我们下期见。

---

*AI深度解析播客 DD11 · 发布日期：2026年4月15日*
