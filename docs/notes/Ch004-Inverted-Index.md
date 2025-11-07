# 倒查表——AI分词的基础

> “倒排表（inverted index）跟 AI 的分词机制有点像”

实际上，**现代 AI 的 embedding 检索（语义向量检索）** 和 **传统搜索的倒排索引** 本质上确实是一种**信息检索的两代范式**。

下面我会用系统性、类比式的方式，帮你彻底搞懂“倒排索引”的设计哲学、数据结构、与它和 AI embedding 的关系。

---

## 🧩 一、正排表 vs 倒排表：为什么叫 “倒排”

假设我们有三个文档：

| 文档ID | 内容         |
| ---- | ---------- |
| 1    | “AI 改变世界”  |
| 2    | “AI 驱动搜索”  |
| 3    | “搜索 改变 未来” |

传统数据库是这样存储的（正排表 / forward index）：

| DocID | Terms        |
| ----- | ------------ |
| 1     | [AI, 改变, 世界] |
| 2     | [AI, 驱动, 搜索] |
| 3     | [搜索, 改变, 未来] |

而**倒排索引（Inverted Index）** 是把它“倒过来”，按“词 → 文档”存：

| Term | Posting List（文档列表） |
| ---- | ------------------ |
| AI   | [1, 2]             |
| 改变   | [1, 3]             |
| 世界   | [1]                |
| 驱动   | [2]                |
| 搜索   | [2, 3]             |
| 未来   | [3]                |

这就是“倒排”的由来。
它使得搜索 “AI 搜索” 时，只需交集 `[1,2] ∩ [2,3] = [2]`，立刻知道文档 2 命中。

---

## ⚙️ 二、倒排索引的数据结构

Lucene 的倒排表实际是一个紧凑的、压缩的结构：

```
Term Dictionary（.tim）：
   term → pointer_to_posting_list

Posting List（.doc）：
   docID_delta + freq + positions
```

### 举个例子：

词项（term）：`搜索`

Posting List 存储内容：

| docID | freq | positions |
| ----- | ---- | --------- |
| 2     | 1    | [2]       |
| 3     | 1    | [1]       |

但 Lucene 实际会编码为：

```
[docDelta=2, freq=1, pos=2]
[docDelta=1, freq=1, pos=1]
```

并用 **可变长编码 (VInt)**、**跳表 (skip list)** 优化查找速度与空间占用。

---

## 🧠 三、构建过程：分词（Tokenization）+ 规范化（Normalization）

倒排索引的构建需要先做 **分析（analysis）**，这一步和你提到的“AI 分词机制”非常相似：

| 步骤                           | 示例                       | 说明      |
| ---------------------------- | ------------------------ | ------- |
| 1️⃣ Tokenization             | “AI 改变世界” → [AI, 改变, 世界] | 切词 / 分词 |
| 2️⃣ Lowercasing              | “AI” → “ai”              | 统一大小写   |
| 3️⃣ Stopword Removal         | 去掉“的、了、是”等停用词            |         |
| 4️⃣ Stemming / Lemmatization | “driving” → “drive”      | 词形归一化   |
| 5️⃣ Encoding                 | 计算 term 的 hash 或 ord 序号  |         |

这些步骤对应 Lucene 的 **Analyzer** 流程：

```
Analyzer → Tokenizer → TokenFilterChain → IndexWriter
```

你可以看到，这和 Transformer 模型的 **Tokenizer + Embedding 层** 在概念上非常接近。

---

## 🔍 四、查询过程：布尔模型与评分机制

当我们查询 “AI 改变”：

1. 拿到 term → 查 term dictionary；
2. 得到 posting list：

   * AI → [1,2]
   * 改变 → [1,3]
3. 求交集 `[1,2] ∩ [1,3] = [1]`；
4. 返回文档 1。

更复杂的查询（如短语搜索、“AI AND 改变 NOT 搜索”）
会通过布尔代数、跳表优化快速完成。

打分阶段用的是 **TF-IDF / BM25**：

$$
score(q, d) = \sum_{t \in q} (idf_t * \frac{tf_{t,d}*(k+1)}{tf_{t,d} + k})
$$

这就是 Lucene 的核心检索算法。

---

## 🧬 五、与 AI 向量检索（embedding search）的对比

| 维度    | 传统倒排索引                      | 向量检索（Embedding Index）              |
| ----- | --------------------------- | ---------------------------------- |
| 基础单位  | 词（Term）                     | 向量（Vector）                         |
| 建索引方式 | Tokenization + Posting List | Embedding + ANN Index (e.g., HNSW) |
| 匹配方式  | 精确匹配（布尔逻辑）                  | 语义相似（向量距离）                         |
| 分数计算  | TF-IDF / BM25               | Cosine / Dot Product               |
| 数据结构  | 跳表、B+树、倒排表                  | 向量图（HNSW）、FAISS、ScaNN              |
| 优点    | 精确、轻量、可解释                   | 语义泛化、鲁棒性高                          |
| 缺点    | 词汇鸿沟（vocabulary gap）        | 计算/内存消耗大，难以更新                      |

---

## 💡 六、联系：分词 vs Embedding 的“共同起点”

你感觉它们像，是因为两者都在做一件本质相同的事：

> **把连续的自然语言，映射成可计算的离散符号空间。**

| 层次    | 倒排索引                | Embedding 检索          |
| ----- | ------------------- | --------------------- |
| 词法层   | Token（词）            | Token（子词）             |
| 向量层   | Term ID             | Dense Vector          |
| 匹配方式  | 精确匹配（term equality） | 相似匹配（cosine distance） |
| 表示稀疏性 | 稀疏（每词对应少量文档）        | 稠密（每向量维度非零）           |
| 模型类型  | 规则驱动                | 学习驱动                  |

你可以把 **倒排索引** 看作是早期“稀疏表示（sparse representation）”的极致实现，
而 **embedding** 是其深度学习时代的“稠密表示（dense representation）”版本。

---

## 🔄 七、两者结合：混合检索（Hybrid Retrieval）

现代搜索系统（包括 Elasticsearch 8.x、OpenAI RAG、FlowMind 😎）
都采用一种融合策略：

> **Hybrid Retrieval = BM25 + Embedding Search**

流程示意：

```
User Query → 分词(BM25) + 向量化(embedding)
             ↓                     ↓
       倒排检索(top-k)         向量近邻检索(top-k)
             ↓                     ↓
          结果融合 (rerank, reciprocal rank fusion)
```

这种方式结合了：

* 倒排索引的**精确度与高效过滤**；
* 向量索引的**语义理解能力**。

---

## 📘 八、与 DDIA 的对应

DDIA 第3章讲的“**Log-Structured Storage Engine**”其实是 Lucene 的 segment 存储；
而倒排索引相当于在存储层上加了**查询优化索引**，对应第2章“Data Models and Query Languages”中描述的结构化 vs 文本型模型。

---

## ✅ 九、总结一句话

> **倒排索引是稀疏语义空间中的符号检索；
> embedding 检索是稠密语义空间中的向量检索。**

两者共同目标都是：
👉 “快速在大量非结构化文本中，找到与你问题语义最接近的内容。”

---
