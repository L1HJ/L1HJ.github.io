## 1. 引言
在 **语义检索 (Semantic Search)**、**问答系统 (QA)** 和 **推荐系统** 中，常见的两类编码结构是 **Bi-Encoder** 和 **Cross-Encoder**。  
它们都基于 Transformer 架构，但在 **向量计算方式、性能与效果** 上有显著差异。

---

## 2. Bi-Encoder

### 2.1 基本原理
- 输入：两个独立文本（例如 Query 和 Document）  
- 模型分别对每个文本编码，得到独立的向量表示：

$$
q = f_\theta(\text{Query}), \quad d = f_\theta(\text{Document})
$$

- [[向量相似度]]计算（常用余弦相似度或内积）：

$$
\text{score}(q, d) = \cos(q, d) \quad \text{or} \quad q \cdot d
$$

### 2.2 特点
- **优点**：
  - 向量可以预先计算并存储
  - 支持 **高效 ANN 检索**（Faiss、HNSW 等）
  - 检索速度快，适合大规模在线服务

- **缺点**：
  - 交互方式有限（只通过向量相似度）
  - 对复杂语义关系捕捉能力较弱

### 2.3 典型应用
- 大规模向量召回（First-stage Retrieval）  
- 推荐系统候选集生成  
- 语义搜索（快速 Top-K 检索）

---

## 3. Cross-Encoder

### 3.1 基本原理
- 输入：将 Query 与 Document 拼接后 **一起输入 Transformer**  
- 模型直接输出匹配得分：

$$
\text{score}(q, d) = g_\theta([\text{Query}; \text{Document}])
$$

### 3.2 特点
- **优点**：
  - 允许 Token 级别的交互
  - 匹配精度高，能捕捉复杂语义关系

- **缺点**：
  - 无法预先计算向量
  - 每次查询都要重新计算，速度慢
  - 不适合大规模检索，只能用于重排序

### 3.3 典型应用
- Rerank 阶段（对 Bi-Encoder 的候选集进行精排）  
- 小规模匹配任务（如问答对打分）

---

## 4. Bi-Encoder vs Cross-Encoder 对比

| 特性     | Bi-Encoder    | Cross-Encoder |
| ------ | ------------- | ------------- |
| 编码方式   | 独立编码          | 拼接后联合编码       |
| 速度     | 快，可扩展到百万/亿级数据 | 慢，每次需要重新计算    |
| 可预计算向量 | ✅             | ❌             |
| 表达能力   | 较弱，只通过向量相似度   | 较强，捕捉跨文本细粒度交互 |
| 适用场景   | 检索/召回阶段       | 精排/重排序阶段      |

---

## 5. 实际工程实践

- **两阶段检索框架**：
  1. **Bi-Encoder**：快速召回候选（Top-K）  
  2. **Cross-Encoder**：对候选集进行精细打分与排序  

- **组合优势**：
  - 兼顾 **效率**（Bi-Encoder）和 **效果**（Cross-Encoder）  
  - 常见于现代语义检索系统，如 OpenAI embedding + reranker 模型

---

## 6. 总结

- **Bi-Encoder**：适合大规模向量检索，速度快，但精度一般  
- **Cross-Encoder**：适合精排，速度慢，但精度高  
- **最佳实践**：两者结合，先用 Bi-Encoder 召回，再用 Cross-Encoder 排序  

---
