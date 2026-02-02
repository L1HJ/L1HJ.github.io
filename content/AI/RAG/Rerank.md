## 1. 引言
在信息检索和推荐系统中，常用 **两阶段检索架构**：

1. **第一阶段（召回 Recall）**  
   - 使用快速的向量检索方法（如 Bi-Encoder + Faiss/HNSW）  
   - 从海量候选集中选出一个 **Top-K 候选集**  

2. **第二阶段（重排序 Rerank）**  
   - 对候选集进行更精细的建模和排序  
   - 常用 **Cross-Encoder** 或更复杂的深度模型  

**Rerank 的目标**：在保证召回速度的前提下，提升最终结果的 **精度与相关性**。

---

## 2. Rerank 的基本流程

1. **输入**：查询向量 $q$ 与候选集合 $$\{d_1, d_2, ..., d_K\}$$  
2. **模型计算打分**：  
   - [[Bi-Encoder]]（粗排）：  
     $$
     s_i = \cos(f(q), f(d_i))
     $$  
   - Cross-Encoder（精排）：  
     $$
     s_i = g([q; d_i])
     $$  
1. **排序**：根据得分 $s_i$ 对候选集合进行排序  
2. **输出**：返回最终 Top-N 结果

---

## 3. 常见的 Rerank 方法

### 3.1 Cross-Encoder Reranker
- 将 Query 与 Document 拼接后输入 Transformer，输出相关性分数  
- 优点：精度高  
- 缺点：速度慢，不适合大规模候选集

### 3.2 Hybrid Rerank
- 将 **稀疏检索（BM25）** 与 **稠密检索（Bi-Encoder）** 的候选集合并  
- 使用轻量模型（如小型 BERT）做 Rerank  
- 平衡效率与效果

### 3.3 学习排序 (Learning to Rank, LTR)
- 特征工程 + 排序模型（如 LambdaMART、XGBoost）  
- 适合传统搜索引擎场景  
- 现在常与深度模型结合

---

## 4. Rerank 的评价指标

常见指标包括：

- **NDCG (Normalized Discounted Cumulative Gain)**  
  衡量排序质量，考虑结果的位次影响  
- **MRR (Mean Reciprocal Rank)**  
  测量第一个相关结果的位置  
- **Recall@K / Precision@K**  
  衡量在前 K 个结果中的召回率与精确率  

例如，NDCG 的计算公式：

$$
\text{NDCG}@K = \frac{DCG@K}{IDCG@K}
$$

其中：

$$
DCG@K = \sum_{i=1}^{K} \frac{2^{rel_i} - 1}{\log_2(i+1)}
$$

---

## 5. 工程实践

- **候选集大小 K**：  
  - 一般在 100～1000 之间  
  - 越大覆盖率越好，但 Rerank 代价越高  

- **模型选择**：  
  - 在线系统：小型 Cross-Encoder（如 DistilBERT-based reranker）  
  - 离线场景：大型 Cross-Encoder（如 RoBERTa-large）  

- **多模态场景**：  
  - 文本 + 图像检索中，也会使用 Rerank 模型融合多模态特征  

---

## 6. 应用场景

- **搜索引擎**：对初步召回结果进行排序，提升搜索质量  
- **推荐系统**：对候选物品进行精排，增加个性化体验  
- **问答系统 (QA)**：对候选答案排序，选择最相关的答案  
- **语义检索**：结合 Bi-Encoder + Cross-Encoder，实现快速且精确的语义搜索  

---

## 7. 总结

- Rerank 是 **信息检索系统中的关键环节**，连接 **高效召回** 和 **高精度排序**  
- 常见实现：Bi-Encoder 召回 + Cross-Encoder 重排序  
- 核心权衡：**速度 vs 精度**  
- 实际应用中通常结合 **ANN + Rerank** 构建两阶段架构，效果最佳  

---
