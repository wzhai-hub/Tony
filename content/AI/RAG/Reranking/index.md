---
title: Reranking：从 Top-K 召回到高精度检索的深度技术解析
# tags:
#   - nodejs
date: '2026-08-05'
summary: Reranking 的本质，就是在第一阶段高召回 Retrieval 的基础上，对候选文档进行更加精细的相关性判断
---

# Reranking：从 Top-K 召回到高精度检索的深度技术解析

## Summary

在现代 RAG（Retrieval-Augmented Generation）系统中，Retrieval 决定“哪些候选文档能够进入候选集合”，而 **Reranking 决定这些候选文档中，哪些真正值得交给 LLM**。

很多 RAG 系统的架构停留在：

```text
Query
  ↓
Embedding
  ↓
Vector Database
  ↓
Top-K
  ↓
LLM
```

这种架构简单，但在企业知识库、技术文档、法律文档、金融知识库和复杂问答中，往往会遇到一个关键问题：

> **Vector Similarity 高，并不代表 Query 与 Document 真正相关。**

Reranking 的本质，就是在第一阶段高召回 Retrieval 的基础上，对候选文档进行更加精细的相关性判断。

典型架构：

```text
                    Query
                      │
                      ↓
              Candidate Retrieval
                      │
                Top 50 / Top 100
                      │
                      ↓
                  Reranker
                      │
                 Top 5 / Top 10
                      │
                      ↓
                    LLM
```

因此，现代 RAG 可以抽象为：

> **Retriever 负责 Recall，Reranker 负责 Precision。**

本文将从 Information Retrieval、Bi-Encoder、Cross-Encoder、Late Interaction、Reranking Score、Batching、Latency、Multilingual Reranking、Hybrid Search、Context Selection、Reranking Evaluation，以及生产级 Reranking Architecture 等方面，对 Reranking 进行系统分析。

---

# 1. 为什么需要 Reranking？

先看一个简单例子。

用户查询：

```text
如何解决 Java 应用中的内存泄漏？
```

Vector Search 返回：

```text
Document A:
Java Heap Dump Analysis

Document B:
JVM Garbage Collection Tuning

Document C:
Java Memory Leak Troubleshooting

Document D:
Spring Boot Performance Optimization

Document E:
Redis Memory Optimization
```

假设 Vector Search 排名：

```text
A  0.89
B  0.87
D  0.86
C  0.85
E  0.82
```

从向量距离来看：

```text
A > B > D > C > E
```

但从用户真正的需求来看：

```text
C > A > B > D > E
```

真正的问题是：

> **Embedding Similarity 并不等价于 Query-Document Relevance。**

因此需要第二阶段模型重新判断。

---

# 2. Retrieval 与 Reranking 的职责不同

一个成熟的搜索系统通常分成两个阶段：

```text
                Query
                  │
                  ↓
          Candidate Retrieval
                  │
                  ↓
              Top 100
                  │
                  ↓
              Reranking
                  │
                  ↓
               Top 10
```

第一阶段：

```text
Retriever
```

目标：

> 尽可能不要漏掉相关文档。

第二阶段：

```text
Reranker
```

目标：

> 从候选文档中精确识别最相关的文档。

所以：

```text
Retriever → Recall
Reranker  → Precision
```

这就是两者最重要的区别。

---

# 3. 为什么 Retriever 不能直接完成 Ranking？

这是理解 Reranking 的关键。

假设 Embedding Model：

```text
Query → Vector
Document → Vector
```

然后：

```text
Cosine Similarity
```

得到：

```text
0.91
0.89
0.87
0.85
```

问题在于：

> Embedding Vector 是一种压缩后的语义表示。

它丢失了部分细粒度信息。

例如：

```text
Query:
How can I prevent Kafka consumer rebalancing?
```

两个 Document：

```text
Document A:
Kafka consumer rebalancing is triggered when...
```

```text
Document B:
Kafka consumer configuration and consumer groups...
```

二者 Embedding 可能都很接近。

但是：

```text
A = directly answers the question
B = related background
```

Vector Similarity 很难稳定地区分：

```text
Directly Relevant
```

和：

```text
Generally Related
```

而 Reranker 可以进一步判断：

```text
Question
+
Document
```

之间的真实关系。

---

# 4. Reranking 的数学抽象

给定：

```text
Query = q
```

Retriever 产生：

```text
D = {d1, d2, ..., dn}
```

Reranker 计算：

```text
score(q, di)
```

然后：

```text
Rank(D)
```

最终：

```text
Top-K
```

所以整个流程：

```text
q
 ↓
Retriever(q)
 ↓
D'
 ↓
Reranker(q, D')
 ↓
Ranked Documents
```

注意：

> Reranker 通常不搜索整个知识库。

它只处理 Retriever 返回的 Candidate Set。

---

# 5. 为什么不能让 Reranker 搜索整个数据库？

假设：

```text
Knowledge Base = 10M documents
```

如果 Reranker 对每一个 Document 都计算：

```text
Query + Document
```

那么需要：

```text
10M model inference
```

显然不可接受。

因此采用：

```text
10M
 ↓
Retriever
 ↓
100
 ↓
Reranker
 ↓
10
```

这就是：

# Two-Stage Retrieval

也可以称为：

```text
Coarse Retrieval
        ↓
Fine Ranking
```

---

# 6. Bi-Encoder：第一阶段 Retrieval

理解 Reranker，必须先理解 Bi-Encoder。

Bi-Encoder：

```text
Query
  ↓
Encoder
  ↓
Query Vector
```

Document：

```text
Document
  ↓
Encoder
  ↓
Document Vector
```

然后：

```text
Similarity(Query Vector, Document Vector)
```

因此 Document Vector 可以提前计算：

```text
Document
 ↓
Embedding
 ↓
Vector
 ↓
Vector Database
```

Query 到达之后：

```text
Query
 ↓
Embedding
 ↓
ANN Search
```

这种架构非常快。

---

# 7. Bi-Encoder 的优势

Bi-Encoder 最大优势：

> Document Embedding 可以离线计算。

例如：

```text
1,000,000 documents
```

可以提前：

```text
Document → Embedding
```

然后存储在 Vector Database。

用户 Query 到来：

```text
Query → Embedding
```

只需要一次 Query Embedding。

然后进行 ANN：

```text
Query Vector
      ↓
HNSW / IVF
      ↓
Top-K
```

所以非常适合：

```text
Large Scale Retrieval
High QPS
Low Latency
```

---

# 8. Cross-Encoder：Reranker 的核心

Cross-Encoder 与 Bi-Encoder 最大的区别：

Bi-Encoder：

```text
Query       → Vector
Document    → Vector
                 ↓
             Similarity
```

Cross-Encoder：

```text
Query + Document
        ↓
      Model
        ↓
 Relevance Score
```

例如：

```text
[CLS]
How to solve JVM memory leak?
[SEP]
JVM heap dump can be used to analyze object retention...
```

整个 Query 和 Document 一起进入 Transformer。

模型直接输出：

```text
Relevance = 0.94
```

因此 Cross-Encoder 能看到：

```text
Query
+
Document
```

之间更加细粒度的交互。

---

# 9. 为什么 Cross-Encoder 更准确？

因为 Transformer 可以直接建立：

```text
Query Token
       ↕
Document Token
```

之间的 Attention。

例如：

```text
Query:
Kafka consumer rebalancing problem
```

Document：

```text
Kafka consumer group rebalancing happens when...
```

模型可以直接学习：

```text
consumer
     ↕
consumer

rebalancing
     ↕
rebalancing

problem
     ↕
happens when
```

这种 Token-Level Interaction。

而 Bi-Encoder 在进入 Similarity 阶段之前：

```text
Query → Vector
Document → Vector
```

已经把大量信息压缩掉了。

因此通常：

```text
Cross-Encoder Accuracy
        >
Bi-Encoder Similarity
```

但：

```text
Cross-Encoder Latency
        >
Bi-Encoder Latency
```

---

# 10. Bi-Encoder vs Cross-Encoder

| 特性                 | Bi-Encoder | Cross-Encoder |
| ------------------ | ---------- | ------------- |
| Query/Document     | 分开编码       | 一起编码          |
| Document Embedding | 可预计算       | 不可直接预计算       |
| 搜索规模               | 百万/亿级      | 候选几十/几百       |
| Latency            | 低          | 高             |
| Recall             | 高          | 不负责召回         |
| Ranking Precision  | 中          | 高             |
| ANN                | 支持         | 不适合           |
| 典型用途               | Retrieval  | Reranking     |

因此：

> Bi-Encoder 和 Cross-Encoder 不是竞争关系，而是互补关系。

---

# 11. Reranking Pipeline

典型 RAG：

```text
User Query
     │
     ↓
Query Embedding
     │
     ↓
Vector Database
     │
     ↓
Top 100
     │
     ↓
┌───────────────────┐
│     Reranker      │
│                   │
│ Query + Document  │
│ Query + Document  │
│ Query + Document  │
└─────────┬─────────┘
          ↓
      Top 10
          ↓
     Context Builder
          ↓
         LLM
```

Reranker 的主要任务：

```text
Top 100
   ↓
High Quality Top 10
```

---

# 12. Reranking 并不是简单重新排序

很多人认为：

```text
Vector Search
 ↓
Reranker
 ↓
Sort
```

实际上 Reranker 需要解决的是：

> Query-Document Relevance Modeling。

例如：

```text
Query:
如何避免 Redis 缓存击穿？

Document A:
Redis Cache Penetration

Document B:
Redis Cache Breakdown

Document C:
Redis Distributed Lock

Document D:
Redis Cluster
```

模型需要理解：

```text
A ≈ Direct Match
B ≈ Direct Match
C ≈ Possible Solution
D ≈ Related Background
```

这已经不是简单的：

```text
Vector Distance
```

而是：

```text
Semantic Relevance
```

---

# 13. Reranking Score

Reranker 输出通常可以抽象成：

```text
score(q, d)
```

例如：

```text
Document A → 0.95
Document B → 0.91
Document C → 0.67
Document D → 0.41
```

然后排序：

```text
A
B
C
D
```

但是需要注意：

> 不同 Reranker 的 score 不一定具有跨模型可比较性。

因此不要简单认为：

```text
0.9 = 90% relevance
```

它通常只是：

> 模型内部用于排序的 relevance score。

---

# 14. Reranker Score Threshold

有时可以增加：

```text
score > threshold
```

例如：

```text
Top 20
```

最终：

```text
score >= 0.7
```

才进入 Context。

架构：

```text
Retriever
   ↓
Top 100
   ↓
Reranker
   ↓
Score Filter
   ↓
Top N
```

但是 Threshold 不能拍脑袋设置。

应该通过：

```text
Evaluation Dataset
```

确定。

---

# 15. Reranking 的最大工程问题：Latency

假设：

```text
Retriever Top-K = 100
```

Reranker：

```text
100 documents
```

如果单次推理：

```text
20 ms
```

串行执行：

```text
100 × 20ms
=
2 seconds
```

显然不可接受。

因此生产系统必须：

# Batch Inference

---

# 16. Batch Reranking

把：

```text
Query + Doc1
Query + Doc2
Query + Doc3
...
```

组成 Batch：

```text
Batch
[
  Query + Doc1,
  Query + Doc2,
  Query + Doc3,
  ...
]
```

一次送入 GPU：

```text
GPU
 ↓
Batch Inference
 ↓
Scores
```

这样可以显著提高：

```text
GPU Utilization
Throughput
```

降低：

```text
Per-Document Inference Cost
```

---

# 17. Dynamic Batching

生产环境通常不会简单使用固定 Batch。

例如：

```text
Request A → 50 documents
Request B → 20 documents
Request C → 100 documents
```

Inference Server 可以动态组合：

```text
Batch
 ├── Request A
 ├── Request B
 └── Request C
```

这样可以提高：

```text
GPU Utilization
```

但需要控制：

```text
Max Batch Size
Max Waiting Time
Max Tokens
```

否则：

```text
Batch Size ↑
```

可能导致：

```text
Latency ↑
```

---

# 18. Reranking Candidate Size 如何选择？

这是非常关键的参数：

```text
Top-N Retriever
```

例如：

```text
Top 20
Top 50
Top 100
Top 200
```

理论上：

```text
Candidate Size ↑
     ↓
Recall ↑
     ↓
Reranking Cost ↑
```

所以：

```text
Retriever Top 100
```

不一定比：

```text
Retriever Top 50
```

更好。

需要通过实验寻找：

```text
Recall
vs
Reranking Latency
```

的最佳平衡点。

---

# 19. Candidate Generation 的原则

一个非常重要的原则：

> **Reranker 无法找回 Retriever 没有召回的文档。**

假设真正答案：

```text
Document X
```

但：

```text
Retriever Top 100
```

没有 X。

那么：

```text
Reranker Top 100
```

也不可能出现 X。

因此：

```text
Retrieval Recall
```

永远是 Reranking 的上限。

---

# 20. Recall@100 的意义

假设：

```text
Retriever Recall@100 = 95%
```

说明：

```text
95% relevant information
```

已经进入 Reranker Candidate Set。

如果：

```text
Recall@100 = 60%
```

即使 Reranker 非常优秀：

```text
NDCG ↑
Precision ↑
```

也无法解决剩余：

```text
40%
```

没有被召回的问题。

所以优化顺序通常应该是：

```text
1. Recall
2. Ranking
3. Context
```

而不是反过来。

---

# 21. Reranking 与 Hybrid Search

如果 Retrieval 同时使用：

```text
BM25
+
Vector Search
```

可以：

```text
BM25 Top 50
Vector Top 50
        ↓
RRF
        ↓
Top 100
        ↓
Reranker
        ↓
Top 10
```

这是非常强大的生产架构。

完整流程：

```text
                   Query
                     │
             ┌───────┴───────┐
             ↓               ↓
           BM25            Vector
             ↓               ↓
           Top50           Top50
             │               │
             └───────┬───────┘
                     ↓
                    RRF
                     ↓
                  Top100
                     ↓
                 Reranker
                     ↓
                   Top10
                     ↓
                    LLM
```

---

# 22. 为什么 Reranker 可以解决 Hybrid Score 问题？

BM25：

```text
score = 12.8
```

Vector：

```text
score = 0.83
```

二者无法直接比较。

RRF 可以根据：

```text
Rank
```

进行融合。

然后 Reranker 再对：

```text
Query + Document
```

进行统一判断。

因此：

```text
BM25
+
Vector
+
RRF
+
Reranker
```

形成一个完整的 Multi-Stage Ranking Pipeline。

---

# 23. Reranking 与 Chunk Size

Reranker 并不能解决所有 Chunk 问题。

假设 Chunk：

```text
10 tokens
```

虽然 Reranker 能判断：

```text
Query
+
Chunk
```

但是信息本身可能不完整。

例如：

```text
“它可以通过增加副本解决。”
```

Reranker 很难知道：

```text
它 = Kafka Consumer？
Redis？
Database？
```

因此 Reranking 之前：

```text
Chunking
Contextualization
```

仍然非常重要。

---

# 24. Parent-Child + Reranking

一种高级 RAG 架构：

```text
Document
   ↓
Small Chunks
   ↓
Embedding
   ↓
Vector Retrieval
   ↓
Top 50 Chunks
   ↓
Reranker
   ↓
Top 10 Chunks
   ↓
Parent Documents
   ↓
Context Construction
   ↓
LLM
```

这样：

```text
Small Chunk
```

负责：

```text
Precision Retrieval
```

Parent：

```text
Context Completeness
```

Reranker：

```text
Relevance
```

三者结合效果通常优于简单：

```text
Document → Embedding → Top-K → LLM
```

---

# 25. Context Compression + Reranking

进一步可以：

```text
Retriever
   ↓
Top 100
   ↓
Reranker
   ↓
Top 10
   ↓
Context Compression
   ↓
Relevant Sentences
   ↓
LLM
```

这里有三个不同层次：

```text
Retriever
→ 找候选文档

Reranker
→ 找最相关文档

Compressor
→ 找文档中最相关内容
```

可以理解成：

```text
Document Level
      ↓
Chunk Level
      ↓
Sentence Level
```

这是非常重要的层级化 Retrieval Architecture。

---

# 26. Multi-Stage Ranking

现代搜索系统实际上可以设计成：

```text
Stage 1
BM25 / ANN
↓
Top 1000

Stage 2
Fusion
↓
Top 200

Stage 3
Cross-Encoder
↓
Top 20

Stage 4
LLM / Contextual Reranker
↓
Top 5
```

每一层：

```text
Candidate Count ↓
Precision ↑
Compute Cost ↑
```

因此：

> 越靠后的 Ranking Stage 越精确，也越昂贵。

---

# 27. Reranking 的模型选择

可以从三个维度选择：

```text
Accuracy
Latency
Cost
```

例如：

### 小模型

```text
Latency ↓
Cost ↓
Accuracy 中等
```

适合：

```text
High QPS
```

### 中型模型

```text
Accuracy ↑
Latency 中等
```

适合：

```text
Enterprise RAG
```

### 大型 Reranker

```text
Accuracy ↑↑
Latency ↑↑
Cost ↑↑
```

适合：

```text
High-value Search
Complex QA
Legal
Financial
Research
```

因此不存在：

> 最大模型一定最好。

---

# 28. Cross-Encoder 的输入长度问题

Reranker 需要处理：

```text
Query + Document
```

如果 Document 很长：

```text
10,000 tokens
```

那么：

```text
Cross-Encoder
```

计算成本会明显增加。

因此通常需要：

```text
Chunking
Truncation
Compression
Max Sequence Length
```

例如：

```text
Query = 50 tokens
Document = 500 tokens
```

通常比：

```text
Query = 50
Document = 5000
```

更适合高吞吐 Reranking。

---

# 29. Token Complexity

Transformer 的 Attention 计算通常与序列长度高度相关。

粗略理解：

```text
Sequence Length ↑
        ↓
Attention Cost ↑
        ↓
Latency ↑
Memory ↑
```

因此：

```text
Document Chunk 太大
```

可能导致：

```text
Reranking Cost ↑↑
```

这也是为什么：

> Chunking 不仅影响 Retrieval Quality，也直接影响 Reranking Performance。

---

# 30. Multilingual Reranking

企业知识库可能同时存在：

```text
中文
English
日文
韩文
```

用户可能：

```text
中文 Query
```

而文档：

```text
English
```

例如：

```text
Query:
如何配置 Kafka consumer？

Document:
Kafka consumer configuration can be customized through...
```

这需要：

```text
Multilingual Embedding
+
Multilingual Reranker
```

否则：

```text
Retrieval
```

可能已经失败。

因此：

> Reranker 的语言覆盖范围必须与 Embedding / Retrieval Strategy 一致。

---

# 31. Domain-Specific Reranking

通用 Reranker：

```text
General Language
```

但企业领域可能有：

```text
Medical
Legal
Financial
Cybersecurity
Software Engineering
```

例如软件工程 Query：

```text
Why does HikariCP connection pool timeout?
```

一个通用模型可能认为：

```text
Database Connection
```

相关就可以。

而专业 Reranker 可以进一步理解：

```text
HikariCP
Connection Pool
Timeout
maxPoolSize
connectionTimeout
```

之间的关系。

因此对于专业知识库：

> Domain-Specific Reranker Fine-Tuning 可能带来明显收益。

---

# 32. Reranker Fine-Tuning

如果企业拥有：

```text
Query
Document
Relevance
```

数据集：

```text
q1, d1, relevant
q1, d2, irrelevant

q2, d3, relevant
q2, d4, irrelevant
```

可以用于：

```text
Supervised Fine-Tuning
```

或者：

```text
Pairwise Ranking
```

---

# 33. Pointwise Ranking

Pointwise：

```text
(Query, Document)
        ↓
Relevance Score
```

例如：

```text
0.93
```

训练目标：

```text
Relevant = 1
Irrelevant = 0
```

比较简单。

---

# 34. Pairwise Ranking

Pairwise：

```text
Query
 ↓
Document A
Document B
```

目标：

```text
A > B
```

例如：

```text
A = Relevant
B = Irrelevant
```

训练：

```text
score(A) > score(B)
```

这种方式更加直接地优化：

> Ranking。

---

# 35. Listwise Ranking

Listwise：

```text
Query
 ↓
[D1, D2, D3, D4, D5]
```

模型直接学习：

```text
最佳排序：
D3 > D1 > D5 > D2 > D4
```

这种方法直接关注整个 Ranking List。

但训练复杂度通常更高。

---

# 36. Reranking Evaluation

Reranker 最重要的指标不是：

```text
Accuracy
```

而是 Ranking Metrics。

常见：

```text
MRR
NDCG@K
Precision@K
Recall@K
Hit Rate@K
```

---

# 37. NDCG 为什么特别重要？

假设：

```text
Query
```

有：

```text
Highly Relevant
Relevant
Partially Relevant
Irrelevant
```

Reranker：

```text
A
B
C
D
```

理想：

```text
A > B > C > D
```

如果：

```text
D > C > B > A
```

虽然：

```text
Relevant Documents
```

可能仍然存在，但 Ranking Quality 非常差。

NDCG 可以很好地衡量：

```text
Relevance
+
Position
```

所以它非常适合 Reranker Evaluation。

---

# 38. Reranker Evaluation Dataset

不要只建立：

```text
Query → Relevant Document
```

更好的 Dataset：

```text
Query
Candidate Documents
Relevance Grade
```

例如：

```text
Query:
如何避免 Kafka Consumer Rebalance？

Document A → 3 Highly Relevant
Document B → 2 Relevant
Document C → 1 Weakly Relevant
Document D → 0 Irrelevant
```

这样可以计算：

```text
NDCG@5
```

并更准确地比较不同 Reranker。

---

# 39. A/B Testing

生产环境中不要仅仅依赖离线 Evaluation。

可以：

```text
Version A
Old Reranker

Version B
New Reranker
```

进行：

```text
A/B Testing
```

观察：

```text
Search CTR
Answer Quality
User Feedback
Task Success Rate
Latency
Cost
```

特别是在 RAG 中：

> Retrieval Quality 的提升最终应该体现在业务指标上。

---

# 40. Reranking 的 Observability

一个完整 Trace：

```text
Query:
"Kafka consumer lag troubleshooting"

Retriever:
Top 100

RRF:
Top 80

Reranker:
Top 10

Scores:
0.98
0.95
0.93
0.88
...
```

同时记录：

```text
Retriever Latency
Reranker Latency
Candidate Count
Final Count
Score Distribution
Token Count
```

可以发现：

```text
为什么 Top 10 质量下降？
```

例如：

```text
Retriever Recall:
95%

Reranker:
NDCG ↓

```

说明：

> 问题主要在 Ranking。

如果：

```text
Retriever Recall:
60%
```

那么：

> 优先修 Retriever，而不是继续调 Reranker。

---

# 41. Reranking 与 Cache

Reranker 的计算通常比较昂贵。

可以缓存：

```text
Query + Document ID
```

例如：

```text
hash(query, document_id)
```

得到：

```text
relevance score
```

如果相同 Query 再次出现：

```text
Cache Hit
```

可以直接使用。

但需要注意：

```text
Document Version
Model Version
Tenant
Permission
```

否则可能出现：

```text
Old Score
```

---

# 42. Model Versioning

Reranker 模型升级：

```text
v1
 ↓
v2
```

可能导致：

```text
Score Distribution
```

发生变化。

因此 Cache Key 最好包含：

```text
model_version
```

例如：

```text
query
document_id
model_version
```

否则：

```text
v2
```

可能错误使用：

```text
v1 score
```

---

# 43. Reranking Service Architecture

企业级 Reranking Service 可以设计为：

```text
                 Retrieval Service
                        │
                        ↓
                Candidate Documents
                        │
                        ↓
                 Reranking Gateway
                        │
                 ┌──────┴──────┐
                 ↓             ↓
             CPU Model      GPU Model
                 │             │
                 └──────┬──────┘
                        ↓
                  Batch Scheduler
                        ↓
                  Model Inference
                        ↓
                    Scores
                        ↓
                     Top-K
```

其中：

```text
Reranking Gateway
```

负责：

```text
Routing
Batching
Timeout
Fallback
Rate Limit
Model Version
```

---

# 44. GPU 与 CPU 的选择

如果：

```text
QPS Low
Document Short
Model Small
```

CPU 可能已经足够。

如果：

```text
QPS High
Batch Large
Model Large
Document Long
```

GPU 更有优势。

所以不要简单认为：

```text
Reranker = GPU
```

应该根据：

```text
Throughput
Latency
Batch Size
Model Size
Sequence Length
```

进行 Benchmark。

---

# 45. Reranking Failure Handling

生产环境中：

```text
Reranker
```

可能：

```text
Timeout
Unavailable
GPU OOM
Overloaded
```

不能让：

```text
整个 RAG
```

完全不可用。

可以设计：

```text
Retriever
   ↓
Reranker
   │
   ├── Success → Top-K
   │
   └── Failure → Retriever Top-K
```

即：

# Graceful Degradation

例如：

```text
Reranker Timeout
```

则：

```text
Fallback:
Vector/BM25 Ranking
```

这样：

```text
Quality ↓
```

但：

```text
Availability remains high
```

这对于生产系统非常重要。

---

# 46. Reranking 的 Cost Optimization

主要手段：

### 1. 减少 Candidate Size

```text
Top 200
 ↓
Top 100
```

---

### 2. 更小模型

```text
Large Reranker
 ↓
Small Reranker
```

---

### 3. Batch Inference

```text
Single
 ↓
Batch
```

---

### 4. Context Compression

减少输入 Token。

---

### 5. Query Cache

重复 Query 直接使用结果。

---

### 6. Routing

简单 Query 使用轻量 Reranker：

```text
Complex Query → Large Model
Simple Query → Small Model
```

---

# 47. Adaptive Reranking

更高级的架构：

```text
Query
 ↓
Complexity Classifier
 ↓
┌───────────────┬────────────────┐
│ Simple        │ Complex        │
↓               ↓
Small Reranker  Large Reranker
```

例如：

```text
“Redis 是什么？”
```

可能只需要：

```text
Top 20
```

而：

```text
“比较 Redis Cluster、Sentinel 和 Codis 的一致性、故障转移和扩展机制。”
```

需要：

```text
Top 100
+
Large Reranker
```

这种方法可以在：

```text
Quality
```

和：

```text
Cost
```

之间取得更好的平衡。

---

# 48. LLM Reranking

除了专门的 Cross-Encoder，还可以让 LLM 进行 Ranking：

```text
Query
+
Documents
 ↓
LLM
 ↓
Rank
```

例如：

```text
D1
D2
D3
D4
```

要求 LLM：

```text
根据 Query 相关性排序。
```

这种方法可能具有：

```text
强语义理解
复杂推理能力
```

但成本通常更高：

```text
Latency ↑
Token Cost ↑
```

因此：

> LLM Reranking 更适合高价值、复杂 Query，而不是所有请求。

---

# 49. LLM Reranking 的另一个问题

LLM 可能出现：

```text
Position Bias
```

例如：

```text
Document A
Document B
Document C
```

模型可能倾向于：

```text
第一项
```

或者：

```text
最后一项
```

因此生产环境可能需要：

```text
Randomization
Pairwise Comparison
Multiple Passes
Structured Output
```

来降低排序偏差。

---

# 50. Reranking 与 Lost in the Middle

LLM Context 中：

```text
Top
Middle
Bottom
```

不同位置的信息利用率可能不同。

因此即使：

```text
Top 20
```

都相关，也不能简单：

```text
Document 1
Document 2
...
Document 20
```

全部塞给 LLM。

需要进一步：

```text
Reranking
+
Context Ordering
```

把最重要的信息放在更合适的位置。

---

# 51. Reranking 不只是“重新排序”

从更高层次看，Reranking 实际上承担三个任务：

```text
1. Relevance Estimation
2. Candidate Selection
3. Context Optimization
```

因此：

```text
Retriever
```

解决：

> What could be relevant?

Reranker：

> What is most relevant?

Context Manager：

> What should the LLM actually see?

这是三个不同的问题。

---

# 52. 一个完整的 RAG Retrieval Pipeline

综合所有技术，可以形成：

```text
                         User Query
                              │
                              ↓
                    Query Understanding
                              │
                              ↓
                       Query Rewrite
                              │
                              ↓
                     Retrieval Router
                              │
              ┌───────────────┼───────────────┐
              ↓               ↓               ↓
            BM25           Vector           Graph
              │               │               │
              └───────────────┼───────────────┘
                              ↓
                           Fusion
                              ↓
                         Top 100
                              ↓
                       Cross-Encoder
                         Reranker
                              ↓
                          Top 20
                              ↓
                    Context Compression
                              ↓
                           Top 5-10
                              ↓
                     Context Ordering
                              ↓
                             LLM
```

这才是现代 RAG 中真正成熟的 Retrieval Architecture。

---

# 53. Reranking 的关键 Trade-off

Reranking 永远存在几个核心 Trade-off：

```text
Accuracy
    ↕
Latency

Candidate Size
    ↕
Compute Cost

Model Size
    ↕
Inference Cost

Context Size
    ↕
LLM Cost

Recall
    ↕
Precision
```

因此：

> 不存在“最好的 Reranker”，只有适合当前系统约束的 Reranking Strategy。

---

# 54. 从架构师角度理解 Reranking

如果只是知道：

```text
CrossEncoder
```

还不够。

真正需要掌握：

```text
为什么 Retriever Top-100？

为什么不是 Top-10？

为什么 Reranker 不能搜索整个数据库？

为什么 Cross-Encoder 比 Bi-Encoder 更精确？

为什么需要 Batch Inference？

为什么 Candidate Size 会影响 Latency？

为什么 Recall 是 Reranking 的上限？

为什么需要 NDCG？

为什么需要 Reranker Fallback？

为什么 Reranker 需要 Model Versioning？

为什么 Query Complexity 可以决定 Reranker Model？

为什么 Reranker 应该成为独立服务？
```

这些问题才是真正的：

> **Reranking Engineering。**

---

# 55. Reranking 的技术全景

可以把整个技术体系总结为：

```text
                         Reranking
                             │
        ┌────────────────────┼────────────────────┐
        ↓                    ↓                    ↓
     Models               Pipeline             Systems
        │                    │                    │
        ↓                    ↓                    ↓
 Bi-Encoder            Candidate Set         Batching
 Cross-Encoder          Fusion                GPU
 Late Interaction       Reranking             Cache
 LLM Reranker            Context              Timeout
                         Selection             Fallback
        │                    │                    │
        └────────────────────┼────────────────────┘
                             ↓
                         Evaluation
                             │
              ┌──────────────┼──────────────┐
              ↓              ↓              ↓
            NDCG            MRR          Precision
```

---

# 56. Retrieval、Reranking 与 Generation 的关系

可以用一个非常清晰的三层模型理解现代 RAG：

```text
┌─────────────────────────────┐
│          Generation         │
│                             │
│             LLM             │
│       Generate Answer       │
└──────────────┬──────────────┘
               ↑
               │ Context
               │
┌──────────────┴──────────────┐
│          Reranking          │
│                             │
│   Query-Document Relevance  │
│       Precision Layer       │
└──────────────┬──────────────┘
               ↑
               │ Candidates
               │
┌──────────────┴──────────────┐
│          Retrieval          │
│                             │
│  BM25 / Vector / Hybrid     │
│        Recall Layer         │
└─────────────────────────────┘
```

三者分别解决：

```text
Retrieval:
“可能相关的是什么？”

Reranking:
“真正相关的是什么？”

Generation:
“如何利用这些信息回答问题？”
```

---

# 57. 最终总结

Reranking 是现代 RAG Retrieval Pipeline 中非常关键的一层。

它解决的不是：

> 如何从海量数据库中搜索文档？

这是 Retriever 的任务。

Reranking 解决的是：

> **在已经召回的候选文档中，哪些文档与当前 Query 真正最相关？**

因此典型架构是：

```text
10M Documents
      ↓
Retriever
      ↓
Top 100
      ↓
Reranker
      ↓
Top 10
      ↓
Context Optimization
      ↓
LLM
```

其中：

```text
Retriever
    = Recall

Reranker
    = Precision

LLM
    = Generation
```

从模型角度：

```text
Bi-Encoder
    ↓
Fast Retrieval

Cross-Encoder
    ↓
Precise Reranking
```

从系统角度：

```text
Candidate Retrieval
        ↓
Fusion
        ↓
Reranking
        ↓
Context Selection
        ↓
Generation
```

从性能角度：

```text
Candidate Size
Batch Size
Sequence Length
Model Size
GPU Utilization
Cache
Timeout
Fallback
```

从质量角度：

```text
Recall@K
MRR
NDCG@K
Precision@K
Hit Rate
```

从架构角度：

```text
Retrieval
+
Reranking
+
Context Engineering
+
Generation
```

共同构成现代 RAG 的核心链路。

最终可以用一句话概括 Reranking：

> **Retrieval 的目标是“不漏掉正确答案”，Reranking 的目标是“把正确答案排到最前面”，而 Context Engineering 的目标则是“让 LLM 看到最有价值的信息”。**

这三层结合起来，才构成真正意义上的高质量 RAG Retrieval Architecture。
