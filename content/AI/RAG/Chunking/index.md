---
title: Chunking：RAG 系统中最容易被低估的核心技术
# tags:
#   - nodejs
date: '2026-08-05'
summary: Chunking 位于Document Processing 和 Retrieval 之间的关键边界。
---


## Summary

在 Retrieval-Augmented Generation（RAG）系统中，很多工程团队会把主要精力放在 Vector Database、Embedding Model、LLM 和 Retrieval Algorithm 上，却忽略了一个看似简单、实际上决定整个系统上限的问题：

> **如何把一份长文档切成适合检索的 Chunk？**

Chunking 并不是简单地把文本按照固定字符数切成若干段。

一个好的 Chunk 应该同时满足：

* 语义完整；
* 检索粒度合理；
* Embedding 后具有良好的语义表达能力；
* 能够独立回答或者支持回答一个问题；
* 不产生过多冗余；
* 不破坏原始文档结构；
* 能够携带足够的上下文；
* 能够被准确地召回。

因此，从 RAG Pipeline 的角度来看：

```text
Document
   │
   ▼
Parsing
   │
   ▼
Chunking
   │
   ▼
Embedding
   │
   ▼
Vector Database
   │
   ▼
Retrieval
   │
   ▼
Reranking
   │
   ▼
LLM
```

Chunking 位于 **Document Processing 和 Retrieval 之间的关键边界**。

如果 Chunking 做错了，后面的 Embedding、Vector Database、Reranking 甚至 LLM 都很难完全弥补。

可以把它概括为：

> **Chunking 决定了 RAG 系统“以什么粒度理解知识”。**

---

# 1. 为什么 Chunking 如此重要？

假设我们有一份 200 页的企业技术文档。

用户提出：

> How do I configure Redis Cluster failover?

如果直接把整个文档作为一个向量：

```text
Document → Embedding → Vector DB
```

那么这个向量实际上表达的是：

```text
Redis
Cluster
Deployment
Security
Monitoring
Failover
Backup
Performance
...
```

它包含大量不同主题。

用户查询：

```text
Redis Cluster failover configuration
```

但整个 Document 的 embedding 是一个非常粗粒度的语义表示。

这会导致：

```text
Query
  │
  ▼
Embedding
  │
  ▼
Document Vector
```

只能回答：

> 这篇文档大概和 Redis Cluster 有关。

而不是：

> 这篇文档中具体哪一段描述了 failover configuration？

因此我们需要：

```text
Document
│
├── Introduction
├── Architecture
├── Redis Cluster
│    ├── Node Configuration
│    ├── Failover
│    ├── Replication
│    └── Recovery
├── Security
└── Monitoring
```

进一步切成：

```text
Chunk 1
Chunk 2
Chunk 3
...
Chunk N
```

于是 Retrieval 的目标从：

```text
Find the relevant document
```

变成：

```text
Find the relevant knowledge unit
```

这实际上是 RAG 最重要的一次粒度变化。

---

# 2. Chunking 的本质：知识粒度设计

从更抽象的角度来看，Chunking 本质上是在解决：

> **Knowledge Granularity Problem**

假设一个文档：

```text
D = {p1, p2, p3, ..., pn}
```

我们需要将它划分成：

```text
C = {c1, c2, c3, ..., cm}
```

其中每个：

```text
ci ⊂ D
```

理想情况下，每一个 Chunk 都应该尽可能满足：

```text
Semantic Cohesion(ci) → high
```

同时：

```text
Noise(ci) → low
```

以及：

```text
Retrieval Utility(ci) → high
```

换句话说，一个 Chunk 不应该只是：

> 长度合适。

而应该是：

> **一个具有独立语义价值的知识单元。**

---

# 3. Chunking 的三个核心矛盾

Chunking 最难的地方在于存在三个天然矛盾。

## 3.1 Chunk 太大

例如：

```text
Chunk Size = 4000 tokens
```

优点：

* 上下文完整；
* 不容易丢失信息；
* 适合复杂问题。

缺点：

* Embedding 语义被稀释；
* 无关内容增多；
* Retrieval Precision 下降；
* LLM Context 消耗增加。

例如一个 Chunk：

```text
Redis Cluster Architecture
+
Redis Security
+
Redis Backup
+
Redis Monitoring
+
Redis Performance
```

用户只问：

```text
How does Redis failover work?
```

这个 Chunk 包含大量无关信息。

---

## 3.2 Chunk 太小

例如：

```text
Chunk Size = 100 tokens
```

可能产生：

```text
Chunk 1:
Redis Cluster consists of...

Chunk 2:
Each master node...

Chunk 3:
A replica is promoted...

Chunk 4:
The promotion process...
```

虽然每个 Chunk 都很精确，但是语义可能不完整。

例如：

```text
A replica is promoted...
```

如果不知道前面的：

```text
Under master node failure...
```

那么这句话本身的语义是不完整的。

因此：

```text
Small Chunk
→ High Precision
→ Low Context Completeness
```

---

## 3.3 Chunk 太多

如果 Chunk 很小：

```text
1 Document
→ 1000 Chunks
```

那么 Retrieval Search Space 会迅速扩大。

这可能造成：

```text
More Candidates
       │
       ▼
More Similar Chunks
       │
       ▼
More Redundancy
       │
       ▼
Lower Context Diversity
```

最终 LLM 收到的 Top-K 结果可能是：

```text
Chunk 10
Chunk 11
Chunk 12
Chunk 13
```

它们实际上都来自同一个段落。

这就是典型的：

> **Retrieval Redundancy**

---

# 4. Fixed-Size Chunking

最简单的 Chunking 方法是固定长度切分。

例如：

```text
chunk_size = 500 tokens
```

算法：

```python
def chunk(text, chunk_size):
    chunks = []

    for i in range(0, len(text), chunk_size):
        chunks.append(text[i:i + chunk_size])

    return chunks
```

实际上生产环境中通常还会增加 overlap。

例如：

```text
chunk_size = 500
overlap = 100
```

那么：

```text
Chunk 1:
0 ───────────── 500

Chunk 2:
400 ───────────── 900

Chunk 3:
800 ───────────── 1300
```

这样可以避免句子或语义在 Chunk 边界处被完全切断。

---

# 5. Overlap 为什么重要？

假设：

```text
Chunk 1:
Redis Cluster uses a master-replica architecture.
When the master node fails, Sentinel detects...
```

如果恰好在这里切断：

```text
Chunk 1:
Redis Cluster uses a master-replica architecture.
When the master node fails...

Chunk 2:
Sentinel detects the failure and promotes...
```

那么：

```text
Chunk 2
```

缺失：

```text
What failure?
```

通过 overlap：

```text
Chunk 1:
...
When the master node fails,
Sentinel detects...

Chunk 2:
When the master node fails,
Sentinel detects the failure and promotes...
```

Chunk 2 的语义完整性明显提高。

---

# 6. Overlap 不是越大越好

很多初学者会认为：

> overlap 越大，信息丢失越少。

实际上不是。

假设：

```text
chunk_size = 500
overlap = 400
```

那么：

```text
Chunk 1: 0 - 500
Chunk 2: 100 - 600
Chunk 3: 200 - 700
Chunk 4: 300 - 800
```

大量文本被重复 embedding。

最终：

```text
Storage ↑
Embedding Cost ↑
Retrieval Redundancy ↑
Index Size ↑
```

而实际获得的 recall improvement 可能非常有限。

因此 Overlap 的目标不是：

> 最大化重叠。

而是：

> **降低 Chunk Boundary Information Loss。**

---

# 7. Recursive Chunking

比 Fixed-Size 更好的方法是：

> Recursive Chunking

核心思想是：

```text
Document
   │
   ▼
Paragraph
   │
   ▼
Sentence
   │
   ▼
Phrase
   │
   ▼
Token
```

优先按照更高层次的语义边界切分。

例如：

```text
1. Introduction

Paragraph 1
Paragraph 2

2. Architecture

Paragraph 3
Paragraph 4

3. Failover

Paragraph 5
Paragraph 6
```

系统可以优先尝试：

```text
Section
→ Paragraph
→ Sentence
→ Token
```

只有当上一层粒度超过目标 Chunk Size 时，才继续向下一层切分。

这种策略比：

```text
every 500 tokens
```

更符合自然语言结构。

---

# 8. Semantic Chunking

进一步的思路是：

> 不按照长度切，而按照语义变化切。

假设：

```text
Sentence 1:
Redis Cluster contains multiple nodes.

Sentence 2:
Each master node can have replicas.

Sentence 3:
Replicas provide redundancy.

Sentence 4:
Prometheus can monitor Redis metrics.

Sentence 5:
Grafana provides visualization.
```

前 3 句话属于：

```text
Redis Architecture
```

后 2 句话属于：

```text
Monitoring
```

因此：

```text
Chunk 1:
Redis Cluster contains multiple nodes.
Each master node can have replicas.
Replicas provide redundancy.

Chunk 2:
Prometheus can monitor Redis metrics.
Grafana provides visualization.
```

这里的核心不是：

```text
Chunk Size
```

而是：

```text
Semantic Similarity
```

---

# 9. Semantic Chunking 的基本算法

可以把每个 Sentence 转换成 embedding：

```text
S1 → E1
S2 → E2
S3 → E3
S4 → E4
S5 → E5
```

计算相邻句子的 cosine similarity：

```text
sim(E1, E2)
sim(E2, E3)
sim(E3, E4)
sim(E4, E5)
```

假设：

```text
0.91
0.89
0.32
0.93
```

那么：

```text
S1 ─ S2 ─ S3
          │
          │ semantic boundary
          ▼
S4 ─ S5
```

因此：

```text
Chunk 1 = S1 + S2 + S3
Chunk 2 = S4 + S5
```

这种方法可以自动发现 Topic Boundary。

---

# 10. Structure-Aware Chunking

对于企业文档、技术文档、PDF、Markdown、HTML，单纯 Semantic Chunking 仍然不够。

因为文档本身包含大量结构信息：

```text
Title
Heading
Subheading
Paragraph
List
Table
Code
Quote
Figure
Caption
```

这些结构实际上也是：

> **Metadata**

例如：

```markdown
## Redis Cluster

### Failover

When a master node fails...
```

如果只保存：

```text
When a master node fails...
```

那么 embedding 之后：

```text
Query:
How does Redis Cluster failover work?
```

可能无法很好地理解：

```text
When a master node fails...
```

真正有价值的信息是：

```text
Redis Cluster
→ Failover
→ When a master node fails...
```

因此可以构建：

```text
Chunk Text =
Document Title
+
Section Title
+
Subsection Title
+
Content
```

例如：

```text
Document: Redis Architecture Guide

Section: Redis Cluster

Subsection: Failover

Content:
When a master node fails, a replica can be promoted...
```

这比单独 embedding：

```text
When a master node fails...
```

具有更好的语义定位能力。

---

# 11. Parent-Child Chunking

这是生产级 RAG 中非常重要的一种方法。

假设：

```text
Parent Chunk
```

保存完整上下文：

```text
Redis Cluster Failover
    │
    ├── Child 1
    ├── Child 2
    ├── Child 3
    └── Child 4
```

检索时：

```text
Query
  │
  ▼
Child Chunk Retrieval
  │
  ▼
Find relevant child
  │
  ▼
Load Parent Chunk
  │
  ▼
LLM
```

例如：

```text
Child Chunk:

When a master fails, the replica with the highest
replication offset can be promoted.
```

这个 Child 非常适合 Retrieval。

但是 LLM 最终获得：

```text
Parent Chunk:

Redis Cluster Failover

Redis uses replicas to provide high availability.

When a master fails, the replica with the highest
replication offset can be promoted.

The cluster then redirects traffic...
```

于是同时获得：

```text
High Retrieval Precision
+
High Context Completeness
```

这实际上解决了：

> Search Granularity 与 Generation Context 之间的矛盾。

---

# 12. Document Hierarchy Chunking

对于大型知识库，可以进一步构建层次结构：

```text
Document
   │
   ├── Section
   │      │
   │      ├── Subsection
   │      │       │
   │      │       ├── Chunk
   │      │       ├── Chunk
   │      │       └── Chunk
   │      │
   │      └── Subsection
   │
   └── Section
```

这样 Retrieval 不再只是：

```text
Vector → Chunk
```

而可以：

```text
Query
  │
  ▼
Section Retrieval
  │
  ▼
Subsection Retrieval
  │
  ▼
Chunk Retrieval
```

这实际上已经开始接近：

> Hierarchical Retrieval

---

# 13. Chunk Metadata

Chunk 不应该只有：

```json
{
  "text": "..."
}
```

生产系统应该尽可能保留丰富 Metadata：

```json
{
  "document_id": "redis-guide-001",
  "section": "Redis Cluster",
  "subsection": "Failover",
  "chunk_id": "chunk-042",
  "parent_id": "section-07",
  "page": 42,
  "source": "redis-cluster-guide.pdf",
  "language": "en",
  "version": "v3",
  "created_at": "2026-08-18"
}
```

这些 Metadata 可以参与：

```text
Filtering
Ranking
Authorization
Debugging
Observability
Citation
Version Control
```

例如：

```text
Query
+
document_type = architecture
+
version = v3
+
department = engineering
```

然后再进行 Vector Search。

这通常比单纯依赖 embedding 更可靠。

---

# 14. Chunking 与 Embedding 的关系

一个经常被忽略的问题：

> Chunk Size 应该和 Embedding Model 配合设计。

假设 Embedding Model 的输入窗口非常大，并不意味着：

```text
Larger Chunk = Better Embedding
```

Embedding 的目标是：

```text
Text
 ↓
Semantic Representation
 ↓
Vector
```

如果一个 Chunk 同时包含：

```text
Database
Security
Networking
Monitoring
Deployment
```

那么最终 Vector 很可能成为这些 Topic 的混合表示。

可以粗略理解为：

```text
Embedding Vector
≈
Topic A
+
Topic B
+
Topic C
+
Topic D
```

于是 Query：

```text
Database indexing
```

可能无法很好地和这个 Chunk 对齐。

因此：

> **Embedding Context Window 是 Chunk Size 的上限，而不是 Chunk Size 的目标。**

---

# 15. Chunking 与 Retrieval 的关系

Chunking 最终必须通过 Retrieval 来验证。

假设：

```text
Top-K = 5
```

理想情况：

```text
Query
 │
 ├── Chunk A  ★★★
 ├── Chunk B  ★★★
 ├── Chunk C  ★★
 ├── Chunk D  ★
 └── Chunk E  ★
```

但如果 Chunk 太大：

```text
Chunk A:
很多主题混合
```

Similarity 可能不够集中。

如果 Chunk 太小：

```text
Chunk A
Chunk B
Chunk C
Chunk D
```

可能全部来自同一个语义片段。

于是：

```text
Recall ↑
Precision ↓
Diversity ↓
```

因此 Chunking 实际上会直接影响：

```text
Recall
Precision
MRR
NDCG
Context Relevance
Answer Faithfulness
```

---

# 16. Chunking 与 Reranking

很多团队发现：

```text
Vector Search
```

效果不好，于是增加：

```text
Reranker
```

例如：

```text
Query
  │
  ▼
Vector Search
  │
  ▼
Top 50
  │
  ▼
Reranker
  │
  ▼
Top 5
```

Reranker 确实可以改善 Retrieval。

但是：

> **Reranking 不能完全解决错误 Chunking。**

假设正确答案被错误地切成：

```text
Chunk A:
When Redis master fails...

Chunk B:
The replica with highest replication offset...

Chunk C:
is promoted to master...
```

三个 Chunk 分别只有一句。

Reranker 即使发现：

```text
A + B + C
```

相关，也需要把多个 Chunk 拼起来。

这会增加：

```text
Context Length
```

同时增加：

```text
Redundancy
```

所以：

```text
Good Chunking
+
Good Retrieval
+
Good Reranking
```

才是完整方案。

---

# 17. Query-Aware Chunking

更加高级的思路是：

> Chunking 不一定应该完全独立于 Query。

传统方式：

```text
Document
  ↓
Chunk
  ↓
Embedding
  ↓
Vector DB
```

Query-Aware Retrieval 则可以：

```text
Document
  ↓
Fine-grained Chunks
  ↓
Index

Query
  ↓
Understand Intent
  ↓
Retrieve
  ↓
Merge related chunks
```

例如用户问：

> What happens when Redis master fails and how does the cluster recover?

这个问题实际上包含两个 sub-topics：

```text
1. Failure Detection
2. Recovery / Promotion
```

系统可以将 Query 分解：

```text
Q1:
How is master failure detected?

Q2:
How is a replica promoted?
```

然后：

```text
Q1 → Chunk A
Q2 → Chunk B
```

最终：

```text
A + B → LLM
```

这比单纯：

```text
Embedding(Query) → Top-K
```

更加适合复杂问题。

---

# 18. Code Chunking

对于代码知识库，普通文本 Chunking 往往效果很差。

例如：

```java
public class RedisService {

    public void save(String key, String value) {
        ...
    }

    public String get(String key) {
        ...
    }
}
```

如果简单按照 500 tokens 切：

```text
Chunk 1:
public class RedisService {
    public void save(...)
```

```text
Chunk 2:
public String get(...)
}
```

那么代码结构被破坏。

更合理的方法是：

```text
Repository
   │
   ├── Package
   │
   ├── Class
   │    ├── Method
   │    ├── Method
   │    └── Method
   │
   └── Test
```

例如：

```text
Chunk:
Class = RedisService
Method = save

public void save(String key, String value) {
    ...
}
```

Metadata：

```json
{
  "language": "java",
  "class": "RedisService",
  "method": "save",
  "package": "com.example.redis"
}
```

这比普通文本 Chunking 更适合：

```text
Code Search
Code RAG
Repository QA
Bug Analysis
```

---

# 19. Table Chunking

表格是 RAG 中另一个非常容易出问题的场景。

例如：

| Region | Q1 Revenue | Q2 Revenue |
| ------ | ---------: | ---------: |
| US     |        100 |        120 |
| EU     |         80 |         95 |
| APAC   |         60 |         90 |

如果 PDF Parser 把它变成：

```text
US 100 120
EU 80 95
APAC 60 90
```

那么：

```text
What was APAC revenue in Q2?
```

可能仍然可以检索。

但如果问题是：

```text
Compare APAC revenue growth between Q1 and Q2.
```

就需要保留：

```text
Column Headers
+
Row
+
Table Title
```

因此更合理的 Chunk：

```text
Table:
Regional Revenue

Columns:
Region | Q1 Revenue | Q2 Revenue

Row:
APAC | 60 | 90
```

甚至可以进一步转换成结构化文本：

```text
Regional Revenue.
APAC revenue was 60 in Q1 and 90 in Q2.
```

对于复杂表格，结构化处理通常优于简单文本切分。

---

# 20. Chunking Pipeline

一个成熟的企业 RAG Pipeline 可以设计为：

```text
                Document
                   │
                   ▼
              Document Parser
                   │
                   ▼
             Structure Analysis
                   │
          ┌────────┴────────┐
          │                 │
          ▼                 ▼
       Markdown           PDF
          │                 │
          └────────┬────────┘
                   ▼
             Semantic Units
                   │
                   ▼
           Structure-Aware
              Chunking
                   │
                   ▼
          Parent / Child Chunks
                   │
                   ▼
             Metadata Enrich
                   │
                   ▼
               Embedding
                   │
                   ▼
             Vector Database
```

这个 Pipeline 比：

```text
split every N characters
```

成熟得多。

---

# 21. 如何选择 Chunk Size？

没有一个适用于所有场景的：

```text
Best Chunk Size
```

这是一个非常重要的工程认知。

Chunk Size 应该由以下因素共同决定：

```text
Document Type
+
Embedding Model
+
Query Type
+
Retrieval Strategy
+
LLM Context Window
+
Answer Complexity
```

一个可以用于实验的起始范围：

| 场景                      |           Chunk Size |
| ----------------------- | -------------------: |
| FAQ                     |       100–300 tokens |
| Technical Documentation |       300–800 tokens |
| General Articles        |       300–700 tokens |
| Legal Documents         |      500–1000 tokens |
| Academic Papers         |      500–1200 tokens |
| Code                    | Function/Class based |
| Tables                  |      Structure based |

这些不是固定标准，而应该作为：

> **Experimental Starting Point**

---

# 22. Chunk Size 应该通过实验决定

不要问：

> Chunk Size 应该设置成 500 还是 1000？

更专业的问题应该是：

> 在我的 Query Distribution 下，哪个 Chunk Size 能获得最好的 Retrieval Quality？

例如建立实验：

```text
Chunk Size:
200
400
600
800
1000
```

分别测试：

```text
Recall@5
Recall@10
MRR
NDCG
Answer Accuracy
Faithfulness
Latency
Cost
```

最终得到：

```text
Chunk Size
    │
    ▼
Retrieval Quality
    │
    ▼
Answer Quality
```

例如实验结果：

```text
200 → Recall 82%
400 → Recall 89%
600 → Recall 93%
800 → Recall 92%
1000 → Recall 88%
```

那么：

```text
600
```

可能是更合理的选择。

---

# 23. Chunking Evaluation

Chunking 不能只通过：

> 看起来切得挺好。

来判断。

应该建立 Evaluation Dataset：

```text
Question
Ground Truth Chunk
Expected Answer
```

例如：

```json
{
  "question": "How does Redis failover work?",
  "relevant_chunks": [
    "redis-failover-042"
  ]
}
```

然后测试：

```text
Query
 ↓
Retriever
 ↓
Top-K
 ↓
Compare Ground Truth
```

计算：

### Recall@K

```text
Recall@K =
Relevant Chunks Retrieved
-------------------------
Total Relevant Chunks
```

### Precision@K

```text
Precision@K =
Relevant Chunks Retrieved
-------------------------
K
```

### MRR

如果正确 Chunk 排名越靠前：

```text
MRR ↑
```

这意味着：

> Chunking 不再是一个“经验参数”，而成为可以通过数据优化的系统参数。

---

# 24. Chunking 的一个更深层理解

如果从 Information Retrieval 的角度来看：

```text
Document
```

是一个非常大的 Information Unit。

Chunking 做的实际上是：

```text
Document
   ↓
Information Units
   ↓
Retrieval Units
```

因此 Chunk 并不是：

> Document 的一小段。

而是：

> **Searchable Knowledge Unit**

这是理解 Chunking 最关键的一点。

---

# 25. Chunking 的最终目标

优秀的 Chunking Strategy 应该让：

```text
Query
   │
   ▼
Relevant Knowledge
   │
   ▼
Relevant Chunk
   │
   ▼
Relevant Context
   │
   ▼
Correct Answer
```

形成一条稳定的信息链。

而糟糕的 Chunking 往往形成：

```text
Query
   │
   ▼
Wrong Chunk
   │
   ▼
Reranker
   │
   ▼
More Chunks
   │
   ▼
LLM
   │
   ▼
Hallucination
```

所以很多所谓：

> “RAG 效果不好”

实际上并不是：

```text
LLM 不够强
```

也不是：

```text
Vector Database 不够好
```

而可能是：

```text
Chunking Strategy 不合理
```

---

# 26. 推荐的企业级 Chunking Architecture

如果设计一个生产级 RAG 系统，我更推荐：

```text
                    Documents
                        │
                        ▼
                 Document Parser
                        │
                        ▼
              Structure Extraction
                        │
                        ▼
        ┌───────────────┴───────────────┐
        │                               │
        ▼                               ▼
   Text Structure                  Tables / Code
        │                               │
        ▼                               ▼
 Semantic / Recursive             Specialized Parser
      Chunking                          │
        │                               │
        └───────────────┬───────────────┘
                        ▼
                Parent / Child
                     Chunks
                        │
                        ▼
                  Metadata Layer
                        │
                        ▼
                    Embedding
                        │
                        ▼
                 Vector Database
                        │
                        ▼
                    Retrieval
                        │
                        ▼
                    Reranking
                        │
                        ▼
                   Context Builder
                        │
                        ▼
                       LLM
```

这里最重要的思想是：

> **不同类型的数据，不应该使用同一种 Chunking Strategy。**

---

# 27. Chunking Strategy Decision Matrix

可以建立如下决策模型：

```text
                    Document
                       │
            ┌──────────┼──────────┐
            │          │          │
           FAQ       Article     Code
            │          │          │
         Small      Semantic    Function
         Chunk      Chunking     Chunking
            │          │          │
            └──────────┼──────────┘
                       │
                 Enterprise RAG
```

进一步：

| 文档类型           | 推荐策略                      |
| -------------- | ------------------------- |
| FAQ            | Question-Answer Chunk     |
| Markdown       | Heading-Aware Chunk       |
| HTML           | DOM-Aware Chunk           |
| PDF            | Layout + Semantic Chunk   |
| Technical Docs | Hierarchical Chunk        |
| Code           | AST / Function Chunk      |
| Tables         | Table-Aware Chunk         |
| Legal          | Section / Clause Chunk    |
| Email          | Thread / Message Chunk    |
| Logs           | Time-window / Event Chunk |

这说明：

> Chunking 应该是 Data-Type Aware 的。

---

# 28. 从 Chunking 走向 Context Engineering

Chunking 最终会自然演化成：

> Context Engineering

因为真正的问题不是：

```text
How to split documents?
```

而是：

```text
How to construct the optimal context
for an LLM?
```

这会进一步涉及：

```text
Chunking
+
Retrieval
+
Reranking
+
Context Compression
+
Context Filtering
+
Metadata Filtering
+
Query Expansion
+
Parent Context
+
Conversation Context
```

最终形成：

```text
User Query
      │
      ▼
Query Understanding
      │
      ▼
Retrieval
      │
      ▼
Reranking
      │
      ▼
Context Compression
      │
      ▼
Context Assembly
      │
      ▼
LLM
```

因此：

> **Chunking 是 Context Engineering 的基础设施。**

---

# 29. 最值得记住的几个结论

如果只记住这篇文章的几个核心观点，可以总结为：

### 第一

> Chunking 不是文本切割问题，而是知识粒度设计问题。

### 第二

> Chunk Size 没有 universally optimal value，必须通过 Retrieval Evaluation 确定。

### 第三

> Chunk 应该优先保持 Semantic Cohesion，而不是机械追求固定长度。

### 第四

> Overlap 的目标是减少 Boundary Information Loss，而不是越大越好。

### 第五

> Structure-aware Chunking 通常优于纯 Fixed-size Chunking。

### 第六

> Parent-Child Chunking 可以同时解决 Retrieval Precision 和 Context Completeness。

### 第七

> Code、Table、PDF、Legal Document 等不同数据类型应该使用不同 Chunking Strategy。

### 第八

> Chunking 的质量最终应该通过 Recall、Precision、MRR、NDCG 和 End-to-End Answer Quality 来验证。

---

# 30. Conclusion

在 RAG 系统中：

```text
Embedding 决定如何表示知识
Vector Database 决定如何存储知识
Retriever 决定如何寻找知识
Reranker 决定哪些知识更重要
LLM 决定如何利用知识
```

而：

```text
Chunking
```

决定的是：

> **系统究竟把什么东西定义为“知识”。**

这是为什么 Chunking 看似简单，却是 RAG 系统中最值得深入研究的基础技术之一。

一个成熟的 RAG 系统不应该简单地使用：

```python
text[:500]
```

然后认为问题已经解决。

真正生产级的 Chunking 应该逐渐演化为：

```text
Structure-Aware
+
Semantic-Aware
+
Metadata-Aware
+
Query-Aware
+
Domain-Aware
+
Evaluation-Driven
```

最终目标不是生成更多 Chunk，而是生成：

> **更容易被检索、更容易被理解、更容易被正确利用的 Knowledge Units。**

从这个角度来看，Chunking 并不是 RAG Pipeline 中一个简单的 preprocessing step。

它实际上是：

```text
Document
   ↓
Knowledge Representation
   ↓
Retrieval Unit
   ↓
Context Construction
   ↓
LLM Reasoning
```

之间的关键桥梁。

而这也是为什么：

> **RAG 的效果上限，很大程度上取决于你如何切分知识。**
