---
title: "一文掌握21种RAG分块策略（含进阶技巧）"
source: "https://mp.weixin.qq.com/s/Uidn-YnkzNZuB4G-aV4j2Q"
author:
  - "[[南七无名式]]"
published:
created: 2025-08-31
description: "检索增强生成（Retrieval-Augmented Generation，简称 RAG）是很多 AI 工程师"
tags:
  - "clippings"
---
原创 南七无名式 *2025年07月19日 09:00*

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZ05iaiaxa7ictnPv6jfGu6J9VfRVFbzaryhYcicPC3mYP2icmRtosYmiaD2fw/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

检索增强生成（Retrieval-Augmented Generation，简称 RAG）是很多 AI 工程师“ 又爱又恨 ”的一项技术。理论上，它听起来很简单：  
“从自定义数据中检索出上下文，让 LLM 基于这些内容生成答案。”

但在实践中，你可能会陷入这样的循环：

- 调整分块方式
- 更换嵌入模型
- 替换检索器
- 微调排序器
- 重写 prompts

🔁 最后，模型仍然告诉你：“我不知道。” 甚至还信誓旦旦地生成一堆 **幻觉内容** 。

其中有一个看似不起眼却影响巨大的环节： **Chunking（分块）** 。数据格式不同、结构不同、用途不同，对应的分块方式也不同。

❗选错了方法，模型要么抓不到重点，要么直接跑偏。

## 1️⃣ 朴素分块（Naive Chunking）

📌按 **每个换行符** 拆分文本。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZpAmHicH0ZVEVQYm6K61lnMjHLorQPpQvQrmDptkK8Sfnh4RKxeiarZpw/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- 笔记、FAQ、聊天记录、逐行内容独立的转录文本

> ⚠️ 如果每行太长，容易超出 LLM Token 限制；太短，可能缺失上下文。

### 2️⃣ 固定窗口分块（Fixed-size Chunking）

📌按 **字符数或词数** 平均拆分，即使切断语义也无所谓。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZI6XbvLmtSBnILHLzr0GCVicrF2x6eppXHPicGKtSSbyQhXj77ArwoncA/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- 无结构的大型文本，如扫描件、杂乱转录、纯文本数据

### 3️⃣ 滑动窗口分块（Sliding Window Chunking）

📌与固定窗口类似，但 **每个块有一定重叠** 。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZMCribwmiaz1zc1EkmfGPmspWcJst7icXx7EhLuagbzHHjDnibj5mswpKAA/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- 思路连贯的长句文本，如散文、报告、论文
- 无格式的自由文本

> ⚖️权衡 token 重复 vs 上下文连续性

### 4️⃣ 基于句子的分块

📌在句号、问号、感叹号后拆分。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZUfY3CiaLhBSTLP7zAtVWJTicTfHEFUUAs3e6mV9svia3FjQeArjJvmclg/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- 博客、技术文档、摘要等每句表达独立思想的文本
- 用于后续更复杂分块的初步预处理

### 5️⃣ 基于段落的分块

📌每个段落一个块（通常以双换行分隔）。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZMKg7mpm2ibUd6IiaI96OHmUemU0qU6slujUtSiaTCvR9QmTh1tFWGfFqA/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- 结构清晰但缺少标题的文本
- 用关键词判断段落主题切换时

### 6️⃣ 基于页面的分块

📌一页就是一个 chunk。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZQQvROnew2KtyCVsGCsEltFMt26ce41dwgd1SkNVlDm31kEbRpHo61g/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- PDF、幻灯片、扫描文档等分页内容
- 需要引用页码的检索系统

### 7️⃣ 结构化分块

📌按照结构标记（如 HTML 标签、日志时间戳、JSON 字段）进行拆分。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZuu5MmrPf3Y81uLbx2WZ3Ww2oB3qQRev9joqRVXQLUhxibLvibhsozdoA/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- 处理日志、JSON、HTML、CSV 等结构化数据

### 8️⃣ 基于文档结构的分块

📌根据标题、小节等自然结构划分。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZskq6bab0bibwjkZdfQDBfpTLelRd59YbtzE5axNMibhaxqGlics1Haj2w/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- 有明确标题的文档，如教材、文章、报告、论文
- 适用于构建多级结构或分层检索系统

### 9️⃣ 关键词分块

📌在特定关键词处拆分，如“问题：”、“主题：”、“注意事项”等。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZDs06tMvUXqHcSFceAxf3NnhfQicUcWU6PTdbQyaic6uico0325tOY2Qgw/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- 无结构文档中，用关键词来标记主题切换点

### 🔟 实体分块（NER）

📌使用命名实体识别提取实体，并将相关内容聚集成块。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZian2PDuS7wFyER6ILoI5SFNpjsvhmRcibVxKicGaaLD8FLcXyR8VN5nfQ/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- 法律合同、新闻稿、剧本、案例研究等强调“人/地/物”的文档

### 1️⃣1️⃣ 基于 Token 分块

📌按 token 数量拆分（使用 tokenizer）。

**适用场景：**

- 无结构文本 + token 限制严格的模型环境

> ✅ 可结合句子分块避免切断句子

### 1️⃣2️⃣ 基于主题的分块

📌先按小单位拆分，再用主题建模（如 LDA、聚类）分组。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZciazWdmBBq4qsGlg03ia09ftxmic2iaggRpKkoPtcIKAIHBrExMroHb83Q/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- 包含多个主题但无明显标记的文档
- 需要保持语义一致性的分块

### 1️⃣3️⃣ 表格感知分块

📌识别并独立处理表格（按行、列或整体）。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZqowQJePWrDr74v1ibbdp4n2805qu9py2fVCbruaBiahc2Hoo7mMrdQLA/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- 含结构化表格内容的报告、财务文档等

### 1️⃣4️⃣ 内容感知分块

📌根据内容类型调整策略（表格保留、段落合并、列表识别）。

**适用场景：**

- 混合格式文档（图文混排、表格+正文）

### 1️⃣5️⃣ 上下文增强分块

📌在分块前由 LLM 添加上下文或摘要。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZIbxt18cSDibJ8LWO22tTDWzbiaN0FzIiaeXc8d4y9EicHpSK05Micb2uEyw/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- 复杂文档（如财报、合同）
- 知识库内容量适中，能整体放入 LLM 处理时

### 1️⃣6️⃣ 语义分块

📌利用嵌入相似度将话题相关内容归组。

**适用场景：**

- 多主题长文档，简单方法无效时使用

> 💡 聚焦“讲的是一件事”而非“放在哪段”

### 1️⃣7️⃣ 递归分块

📌先粗后细，逐层递归拆分直到满足长度要求。

**适用场景：**

- 内容长度波动大，如对话、采访、发言稿等

### 1️⃣8️⃣ 基于嵌入的分块

📌先嵌入每句话，再按相似度动态聚合。

**适用场景：**

- 完全无结构文本
- 滑动窗口等方法效果差时可尝试

### 1️⃣9️⃣ 基于 LLM 的分块（Agentic Chunking）

📌让大模型自己判断如何拆分。

**适用场景：**

- 内容复杂到需要“人类判断”的情况

> 💸 计算成本高，需谨慎使用

### 2️⃣0️⃣ 分层分块（Hierarchical Chunking）

📌划分多层级，如“章节 ➝ 小节 ➝ 段落”，保留上下文结构。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZ9DyplWkILdMNlpVSTWqLsvQt7Exiagvbk2Sj3u7SYrylfzAU2bUGpxQ/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**适用场景：**

- 教材、手册、研究论文等有层级结构的内容
- 适合“先看大纲，再看细节”的用户体验需求

### 2️⃣1️⃣ 模态感知分块（Modality-Aware）

📌将不同类型的内容（如文字、图片、表格）分开处理。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/SaeK9tW7Bu8p8gmjXx7InE5mIQZvjxoZbYEuCVPyeWlcSPXRYiccBKTIIJ9ic0bGXZ4iaIfx5GbEjjUu7bn8T3Cew/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

## 🎁附加策略：混合分块（Hybrid Chunking）

📌融合多种方法、嵌入、启发式规则或 LLM，打造更稳健的分块系统。

**适用场景：**

- 没有一种方法适配你的数据时
- 多种文档类型共存时

## ✅总结

选择合适的 Chunking 策略，不只是技术细节，而是决定 RAG 系统成败的 **核心变量** 。

🚀 **掌握这 21 种策略，能让你的 LLM 系统更聪明、更稳健、更可靠。**

📢 **如果你觉得本文有帮助：**  
欢迎点赞、收藏或转发给正在构建 RAG 系统的朋友们 🙌  
如果你希望我继续写更多 RAG、向量检索、嵌入优化相关内容，也欢迎留言告诉我！

  
