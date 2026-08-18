---
title: Retrieval：从信息检索到 RAG 核心引擎的深度技术解析
# tags:
#   - nodejs
date: '2026-08-05'
summary: 给定一个 Query，从大规模知识集合中召回最可能与 Query 相关的信息，并在有限的延迟、计算和 Context Window 内，为下游 LLM 提供最高价值的证据。
---

## Summary

Retrieval 是 RAG（Retrieval-Augmented Generation）、AI Search、Enterprise Knowledge Base 和 Agent 系统中的核心环节。很多工程实践把 RAG 简化成“Embedding + Vector Database + LLM”，但真正决定 AI 应用回答质量的，往往不是 LLM 本身，而是 **Retrieval 能否在海量、异构、动态变化的数据中，准确、高效、安全地找到真正有价值的信息**。

从技术本质上看，Retrieval 解决的是一个信息检索问题：

> **给定一个 Query，从大规模知识集合中召回最可能与 Query 相关的信息，并在有限的延迟、计算和 Context Window 内，为下游 LLM 提供最高价值的证据。**

现代 Retrieval 已经从传统的 Keyword Search 演进到：

```text
Keyword Retrieval
       ↓
Semantic Retrieval
       ↓
Hybrid Retrieval
       ↓
Multi-Stage Retrieval
       ↓
Reranking
       ↓
Query Routing
       ↓
Agentic Retrieval
```

本文从 Information Retrieval 的基本原理开始，深入讨论 BM25、Dense Retrieval、Embedding、ANN、Hybrid Search、Reranking、Query Expansion、Metadata Filtering、Multi-Query、Parent-Child Retrieval、Context Compression、Evaluation，以及生产级 Retrieval Architecture，并进一步分析 Retrieval 在 RAG 和 AI Agent 中的演进方向。

---

# 1. Retrieval 到底是什么？

Retrieval 的中文通常翻译为：

> 信息检索。

最简单的定义是：

```text
Query
  ↓
Retrieval System
  ↓
Relevant Documents
```

例如用户输入：

```text
如何解决 Java 应用的内存泄漏？
```

知识库中可能存在：

```text
Document A:
Java Heap Dump Analysis

Document B:
JVM Memory Leak Troubleshooting

Document C:
Spring Boot Performance Tuning

Document D:
Kafka Consumer Optimization

Document E:
Redis Memory Management
```

Retrieval 的任务不是回答问题。

它的任务是：

```text
找到：
A
B
C
```

然后将这些信息交给：

```text
LLM
```

因此必须区分：

```text
Retrieval ≠ Generation
```

Retrieval 负责：

```text
Find Evidence
```

LLM 负责：

```text
Generate Answer
```

---

# 2. Retrieval 为什么是 RAG 的核心？

一个典型 RAG：

```text
User
 ↓
Query
 ↓
Retrieval
 ↓
Relevant Context
 ↓
LLM
 ↓
Answer
```

如果 Retrieval 找错了：

```text
Wrong Context
 ↓
LLM
 ↓
Wrong Answer
```

即使 LLM 本身能力非常强，也可能因为：

```text
Missing Information
Irrelevant Information
Outdated Information
Unauthorized Information
```

而生成错误结果。

所以 RAG 的一个核心原则是：

> **Garbage In, Garbage Out。**

对于 RAG：

```text
Bad Retrieval
     ↓
Bad Context
     ↓
Bad Answer
```

因此：

> Retrieval Quality 往往是 RAG Quality 的上限之一。

---

# 3. Retrieval 的数学抽象

假设知识库：

```text
D = {d1, d2, d3, ..., dn}
```

用户 Query：

```text
q
```

Retrieval 的目标是找到：

```text
Top-K(d | q)
```

也就是：

> 根据 Query q，找到最相关的 K 个 Document。

可以抽象为：

```text
score(q, d)
```

其中：

```text
q = Query
d = Document
```

然后：

```text
sort(score(q,d))
```

最终得到：

```text
d1
d2
d3
...
dk
```

因此 Retrieval 的核心其实是：

> **Relevance Estimation。**

---

# 4. Retrieval 的两个核心阶段

现代 Retrieval 通常可以拆成：

```text
Candidate Retrieval
        ↓
Candidate Ranking
```

也就是：

```text
Recall
  ↓
Precision
```

第一阶段：

> 尽可能找到所有可能相关的候选文档。

第二阶段：

> 从候选文档中挑选最相关的文档。

所以典型架构：

```text
Query
  ↓
Retriever
  ↓
Top 50 / Top 100
  ↓
Reranker
  ↓
Top 5 / Top 10
```

这就是：

# Multi-Stage Retrieval

---

# 5. 第一代 Retrieval：Keyword Search

最传统的信息检索方式是：

```text
Keyword Search
```

例如：

```text
Query:

Java memory leak
```

搜索系统寻找包含：

```text
Java
memory
leak
```

的文档。

典型技术：

```text
Inverted Index
TF-IDF
BM25
```

---

# 6. Inverted Index

传统搜索引擎通常不会每次扫描所有 Document。

例如：

```text
Document 1:
Java Redis Kafka

Document 2:
Java Spring

Document 3:
Redis PostgreSQL
```

建立倒排索引：

```text
Java
 ├── Doc1
 └── Doc2

Redis
 ├── Doc1
 └── Doc3

Kafka
 └── Doc1

Spring
 └── Doc2
```

查询：

```text
Java Redis
```

可以快速定位：

```text
Doc1
Doc2
Doc3
```

而不是扫描全部文档。

这就是：

> Inverted Index。

---

# 7. TF-IDF

TF-IDF 是经典 Information Retrieval 方法。

TF：

```text
Term Frequency
```

表示一个词在文档中出现的频率。

IDF：

```text
Inverse Document Frequency
```

用于降低常见词的权重。

核心思想：

```text
词在当前文档出现很多次
+
这个词在整个语料库中比较少见
=
这个词更加重要
```

例如：

```text
the
is
and
```

在大量文档中都出现。

它们的信息量并不高。

而：

```text
KafkaConsumer
OutOfMemoryError
RedisCluster
```

可能更有区分度。

---

# 8. BM25

BM25 是现代 Keyword Retrieval 中非常重要的 Ranking Function。

它解决的问题是：

> 一个词出现很多次，是不是意味着这个文档一定更相关？

答案并不是。

因此 BM25 对：

```text
Term Frequency
Document Frequency
Document Length
```

进行综合计算。

可以抽象为：

```text
BM25(Query, Document)
```

最终得到一个 relevance score。

BM25 的一个重要特点是：

> 对 Term Frequency 存在饱和效应。

也就是说：

```text
第一次出现
价值很高

第二次出现
价值增加

出现 100 次
不会变成 100 倍重要
```

---

# 9. Keyword Retrieval 的优势

Keyword Search 并没有因为 Vector Search 出现而失去价值。

它特别适合：

```text
Exact Match
Error Code
Product ID
Version
Class Name
API Name
File Name
Technical Term
```

例如：

```text
Query:

ERR_CONNECTION_RESET
```

如果使用纯 Semantic Search：

```text
“网络连接错误”
```

可能得到一些语义相近的信息。

但用户真正需要的是：

```text
ERR_CONNECTION_RESET
```

对应的知识。

因此：

> 精确标识符通常应该保留 Keyword Retrieval。

---

# 10. Dense Retrieval

随着 Embedding 技术的发展，Retrieval 开始从：

```text
Keyword Matching
```

进入：

```text
Semantic Matching
```

Query：

```text
如何降低 JVM 内存使用？
```

Document：

```text
Techniques for reducing Java heap consumption
```

两者没有完全相同的关键词。

但是语义非常接近。

Dense Retrieval 的思想是：

```text
Query
 ↓
Embedding Model
 ↓
Query Vector
```

Document：

```text
Document
 ↓
Embedding Model
 ↓
Document Vector
```

然后：

```text
Similarity(Query Vector, Document Vector)
```

---

# 11. Dense Retrieval 的核心

可以抽象为：

```text
q → f(q)
d → f(d)
```

其中：

```text
f()
```

是 Embedding Model。

然后：

```text
score(q,d)
=
similarity(f(q), f(d))
```

常见：

```text
Cosine Similarity
Dot Product
Euclidean Distance
```

这使得 Retrieval 从：

```text
Lexical Matching
```

升级到：

```text
Semantic Matching
```

---

# 12. Sparse Retrieval vs Dense Retrieval

两者可以这样理解：

| 特性          | Sparse Retrieval | Dense Retrieval |
| ----------- | ---------------- | --------------- |
| 典型方法        | BM25             | Embedding       |
| 匹配方式        | Keyword          | Semantic        |
| Exact Match | 强                | 中               |
| 同义表达        | 弱                | 强               |
| 技术术语        | 强                | 视模型而定           |
| 未知词         | 较差               | 视模型             |
| 计算方式        | 倒排索引             | Vector Search   |
| 典型系统        | Elasticsearch    | Vector DB       |

因此二者并不是简单的替代关系。

---

# 13. Hybrid Retrieval

生产系统经常采用：

```text
Keyword Retrieval
        +
Dense Retrieval
```

即：

# Hybrid Retrieval

架构：

```text
                   Query
                     │
            ┌────────┴────────┐
            ↓                 ↓
       BM25 Search       Vector Search
            ↓                 ↓
        Top 50              Top 50
            │                 │
            └────────┬────────┘
                     ↓
                Fusion
                     ↓
                  Top-K
```

这样可以同时利用：

```text
Lexical Signal
+
Semantic Signal
```

---

# 14. 为什么 Hybrid Retrieval 很重要？

假设 Query：

```text
Spring Boot 3.2.5 Actuator
```

Keyword Search：

```text
能够准确匹配：

Spring
Boot
3.2.5
Actuator
```

Vector Search：

```text
可能理解：

Spring Boot monitoring endpoint
```

二者结合：

```text
Exact Match
+
Semantic Understanding
```

通常比任何一种单独 Retrieval 更鲁棒。

---

# 15. Retrieval Fusion

Hybrid Retrieval 必须解决：

> BM25 Score 和 Vector Score 如何融合？

因为：

```text
BM25 Score
```

和：

```text
Cosine Similarity
```

并不一定处于相同的数值空间。

直接：

```text
0.5 × BM25
+
0.5 × VectorScore
```

未必合理。

因此一种常见方法是：

# RRF

Reciprocal Rank Fusion。

核心思想：

> 不直接比较原始 score，而比较排名。

例如：

```text
BM25:

A
B
C
D

Vector:

B
C
A
E
```

根据 Rank 计算：

```text
RRF Score
```

最终得到：

```text
B
A
C
D
E
```

这样可以避免不同 Search Engine Score Scale 不一致的问题。

---

# 16. Retrieval 的真正难点：Recall

假设：

```text
知识库中真正相关文档：

A B C D E
```

Retriever 返回：

```text
A B C X Y
```

那么：

```text
Recall@5 = 3 / 5
```

也就是：

```text
60%
```

如果 Retriever 没有召回：

```text
D
E
```

那么后面的 Reranker：

```text
再强
```

也没有意义。

因此：

> Reranker 只能重新排序已经召回的文档。

这意味着：

# Retrieval Recall 是整个 Pipeline 的基础。

---

# 17. Recall 与 Precision

信息检索中两个非常重要的概念：

```text
Recall
Precision
```

Recall：

> 所有相关文档中，我找到了多少？

Precision：

> 我找出来的文档中，有多少是真正相关的？

例如：

```text
Relevant Documents = 10
Retrieved = 10
Relevant Retrieved = 8
```

那么：

```text
Recall = 8 / 10
Precision = 8 / 10
```

在 RAG 中通常需要：

```text
第一阶段：
Recall First
```

然后：

```text
第二阶段：
Precision Optimization
```

---

# 18. 为什么需要 Reranker？

假设：

```text
Retriever
 ↓
Top 100
```

这些文档只是：

> “可能相关”。

接下来需要一个更精确的模型重新判断：

```text
Query
+
Document
↓
Relevance Score
```

这就是：

# Reranking

---

# 19. Bi-Encoder vs Cross-Encoder

Dense Retrieval 常见的是：

```text
Bi-Encoder
```

Query：

```text
q → Encoder → vector
```

Document：

```text
d → Encoder → vector
```

然后比较：

```text
vector(q)
vs
vector(d)
```

优势：

```text
Fast
Scalable
```

因为 Document Vector 可以提前计算。

---

# 20. Cross-Encoder

Reranker 可以使用：

```text
Query + Document
       ↓
Cross Encoder
       ↓
Relevance Score
```

例如：

```text
Question:
如何解决 JVM 内存泄漏？

Document:
Java Heap Dump 可以用于分析对象引用链...
```

模型直接判断：

```text
Relevance = 0.93
```

因为 Query 和 Document 同时输入模型：

```text
Cross-Encoder
```

通常具有更高的相关性判断能力。

但代价是：

```text
Query × Documents
```

需要大量模型计算。

所以：

> Cross-Encoder 更适合 Reranking，而不是全库 Retrieval。

---

# 21. 两阶段 Retrieval

现代 RAG 常见架构：

```text
                 Query
                   ↓
             Dense Retrieval
                   ↓
               Top 100
                   ↓
              Reranker
                   ↓
                Top 10
                   ↓
                 LLM
```

这实际上是一种：

```text
Coarse Retrieval
        ↓
Fine Ranking
```

也就是：

> 先快后精。

---

# 22. Query Understanding

很多 Retrieval 问题其实不是 Retriever 的问题。

而是：

> Query 本身就不好。

例如：

```text
“Kafka 不行了怎么办？”
```

这是一个非常模糊的 Query。

可能实际意思是：

```text
Kafka Consumer Lag
Kafka Broker Failure
Kafka Producer Timeout
Kafka Rebalance
Kafka Partition
```

因此现代 Retrieval Pipeline 往往增加：

# Query Understanding

---

# 23. Query Rewrite

LLM 可以把：

```text
Kafka 不行了怎么办？
```

转换成：

```text
Kafka consumer lag troubleshooting
Kafka broker availability
Kafka consumer rebalance
Kafka producer timeout
```

然后分别 Retrieval。

架构：

```text
Original Query
      ↓
Query Rewrite
      ↓
Structured Query
      ↓
Retriever
```

这样可以显著改善 Retrieval Quality。

---

# 24. Multi-Query Retrieval

对于复杂问题：

```text
如何设计一个高并发订单系统？
```

可以生成多个 Query：

```text
Q1:
high concurrency order system architecture

Q2:
distributed order processing

Q3:
inventory consistency

Q4:
order idempotency

Q5:
distributed transaction
```

分别 Retrieval：

```text
Q1 → Docs
Q2 → Docs
Q3 → Docs
Q4 → Docs
Q5 → Docs
```

最后：

```text
Merge
 ↓
Deduplicate
 ↓
Rerank
```

这种方法称为：

# Multi-Query Retrieval

---

# 25. Query Expansion

另一种方法是：

```text
Original Query
      ↓
Synonym / Related Terms
      ↓
Expanded Query
```

例如：

```text
JVM memory leak
```

扩展：

```text
Java heap leak
OutOfMemoryError
heap dump
memory retention
GC memory issue
```

这样可以提高：

```text
Recall
```

特别是对于术语差异较大的知识库。

---

# 26. HyDE

另一个有趣的方法是：

# Hypothetical Document Embeddings

简称：

```text
HyDE
```

思路：

```text
User Query
    ↓
LLM
    ↓
Hypothetical Answer
    ↓
Embedding
    ↓
Vector Search
```

例如用户：

```text
什么是 JVM Metaspace？
```

先让 LLM 生成一个假想回答：

```text
Metaspace is an area used by JVM to store class metadata...
```

然后对这个“假想文档”进行 Embedding。

这样 Query Vector 更接近真正的：

```text
Knowledge Document
```

从而可能改善 Retrieval。

---

# 27. Metadata Filtering

Retrieval 不能只考虑：

```text
Semantic Similarity
```

还需要：

```text
Metadata
```

例如：

```json id="bcld0u"
{
  "tenant": "enterprise-a",
  "department": "engineering",
  "year": 2026,
  "document_type": "architecture"
}
```

Query：

```text
tenant = enterprise-a
AND department = engineering
AND year >= 2025
```

然后：

```text
Semantic Search
```

这叫：

# Filtered Retrieval

---

# 28. 为什么权限过滤必须进入 Retrieval？

假设：

```text
User A
```

没有权限读取：

```text
Salary.xlsx
```

如果流程是：

```text
Vector Search
 ↓
Top 10
 ↓
Authorization Filter
```

可能出现：

```text
Salary.xlsx
```

已经被 Retrieval 召回。

如果后续流程出现错误：

```text
LLM Context
```

可能泄露敏感信息。

更安全的方式：

```text
Authorization Filter
        ↓
Candidate Space
        ↓
Vector Retrieval
```

即：

> **Access Control 必须成为 Retrieval Constraint。**

---

# 29. Parent-Child Retrieval

RAG 中经常遇到一个问题：

> Retrieval 粒度与 Context 粒度并不一致。

例如一个章节：

```text
Chapter
 ├── Section A
 ├── Section B
 ├── Section C
 └── Section D
```

Embedding 时：

```text
Section A → Vector
Section B → Vector
Section C → Vector
Section D → Vector
```

如果 Query 命中：

```text
Section B
```

最终给 LLM 的可能不是：

```text
Section B
```

而是：

```text
Parent Chapter
```

这就是：

# Parent-Child Retrieval

---

# 30. 为什么 Parent-Child 有价值？

小 Chunk：

```text
Retrieval Precision ↑
```

大 Chunk：

```text
Context Completeness ↑
```

Parent-Child Retrieval 可以同时实现：

```text
Small Chunk
    ↓
Accurate Retrieval
    ↓
Large Parent Context
    ↓
LLM
```

因此它解决了：

> Search Granularity 与 Context Granularity 的冲突。

---

# 31. Contextual Retrieval

普通 Chunk：

```text
“它可以通过增加副本解决这个问题。”
```

如果单独 Embedding：

```text
Context 不完整
```

因为：

```text
它
```

指什么？

可以在 Chunk 前增加上下文：

```text
Document:
Kafka Consumer Architecture

Section:
Consumer Scaling

Contextualized Chunk:
In Kafka Consumer Scaling, increasing the number
of consumer instances can improve throughput...
```

然后再 Embedding。

这样 Retrieval 更容易理解 Chunk 的语义。

---

# 32. Context Compression

假设：

```text
Retriever
 ↓
Top 20
```

每个 Chunk：

```text
1,000 tokens
```

最终：

```text
20,000 tokens
```

但是其中可能只有：

```text
2,000 tokens
```

真正有价值。

因此可以：

```text
Retrieved Documents
       ↓
Context Compression
       ↓
Relevant Sentences
       ↓
LLM
```

这可以减少：

```text
Token Cost
Context Noise
Latency
```

---

# 33. Retrieval 与 Context Window

LLM Context Window 越来越大，但：

> Context Window 大，不代表应该把更多文档全部塞进去。

例如：

```text
Top 100 Documents
```

即使 LLM 可以容纳：

```text
100K tokens
```

也可能出现：

```text
Context Dilution
Attention Competition
Noise
Cost
```

因此：

> Retrieval 的目标不是最大化 Context，而是最大化 Context Utility。

---

# 34. Retrieval 的最终目标

因此一个优秀 Retrieval System 应该优化：

```text
Relevant Information
        /
Latency + Cost + Context
```

而不是：

```text
Retrieved Documents Count
```

---

# 35. Retrieval Evaluation

这是很多 RAG 项目最容易忽略的部分。

如果没有 Evaluation：

```text
Retriever 改了
 ↓
RAG 似乎更好了
```

但无法知道：

```text
Recall 是否提高？
Precision 是否提高？
Latency 是否下降？
```

因此必须建立 Evaluation Dataset。

---

# 36. Retrieval Evaluation Dataset

可以构建：

```text
Query
Relevant Documents
Relevant Chunk
Expected Answer
```

例如：

```text
Query:
如何解决 Redis 热 Key？

Relevant:
doc-123
chunk-45
```

然后测试：

```text
Top-1
Top-5
Top-10
Top-20
```

---

# 37. Retrieval Evaluation Metrics

重要指标包括：

```text
Recall@K
Precision@K
MRR
NDCG
Hit Rate
```

---

# 38. Hit Rate

最简单：

```text
Top-K
```

中是否包含至少一个正确文档。

例如：

```text
Relevant:
A

Retrieved Top-5:
B C A D E
```

那么：

```text
Hit@5 = 1
```

它适合快速判断：

> Retrieval 是否至少找到了正确答案。

---

# 39. MRR

MRR：

```text
Mean Reciprocal Rank
```

如果正确文档排名：

```text
Rank 1
```

得分：

```text
1
```

如果：

```text
Rank 2
```

得分：

```text
1/2
```

如果：

```text
Rank 10
```

得分：

```text
1/10
```

所以：

> 正确答案越靠前，MRR 越高。

它非常适合评估：

```text
Ranking Quality
```

---

# 40. NDCG

NDCG 更适合：

```text
多个相关文档
+
不同相关程度
```

例如：

```text
Document A = Highly Relevant
Document B = Relevant
Document C = Slightly Relevant
```

NDCG 可以同时考虑：

```text
Relevance
+
Ranking Position
```

因此在复杂 Retrieval Evaluation 中非常有价值。

---

# 41. Retrieval Evaluation 与 LLM Evaluation

应该区分：

```text
Retrieval Evaluation
```

和：

```text
Generation Evaluation
```

例如：

```text
Question
 ↓
Retriever
 ↓
Documents
 ↓
LLM
 ↓
Answer
```

可以拆成：

```text
Retriever:
Recall / Precision / NDCG

LLM:
Faithfulness
Correctness
Relevance
```

这样才能定位问题。

如果答案错误：

```text
是 Retriever 找错了？
```

还是：

```text
Retriever 找对了，
LLM 没有正确使用 Context？
```

---

# 42. Retrieval Failure Taxonomy

生产环境可以把 Retrieval Failure 分成几类。

## 1. Query Failure

用户 Query 本身不清晰。

```text
“这个怎么解决？”
```

---

## 2. Index Failure

数据没有正确建立索引。

```text
Document
 ↓
Embedding Failure
 ↓
Missing Vector
```

---

## 3. Chunk Failure

Chunk 切分错误：

```text
Context 被拆散
```

---

## 4. Retrieval Failure

正确文档没有被召回。

```text
Recall ↓
```

---

## 5. Ranking Failure

正确文档被召回，但是排名太低。

```text
Recall OK
Precision / Ranking ↓
```

---

## 6. Context Failure

文档正确，但是上下文被截断。

---

## 7. Generation Failure

Retrieval 正确，LLM 仍然回答错误。

---

# 43. Retrieval Observability

生产环境必须能够看到：

```text
Query
 ↓
Query Rewrite
 ↓
Embedding
 ↓
Retriever
 ↓
Candidates
 ↓
Reranker
 ↓
Final Context
 ↓
LLM
```

因此应该记录：

```text
query_id
tenant_id
query
retrieval_method
top_k
candidate_count
reranker_score
latency
documents
```

例如：

```text
Retrieval Trace

Query: "Kafka consumer lag"
   
BM25:
  42 ms
  Top 50

Vector:
  18 ms
  Top 50

Fusion:
  3 ms

Reranker:
  86 ms

Final:
  Top 5
```

这样才能定位：

```text
为什么 RAG 慢？
```

---

# 44. Retrieval Latency Budget

假设整个 API：

```text
P95 < 2 seconds
```

可以设计：

```text
Query Rewrite       150ms
Embedding            50ms
Keyword Search       50ms
Vector Search        30ms
Fusion               5ms
Reranker            200ms
LLM                1200ms
Network              100ms
```

总计：

```text
≈ 1.785 seconds
```

这样才能进行真正的：

> Retrieval Performance Engineering。

---

# 45. Retrieval Cache

对于高频 Query：

```text
“什么是 Kubernetes？”
```

可能重复出现。

可以使用：

```text
Query Cache
Embedding Cache
Retrieval Result Cache
```

例如：

```text
Query
 ↓
Normalize
 ↓
Hash
 ↓
Cache
```

但需要注意：

```text
Knowledge Base 更新
```

以后：

```text
Old Retrieval Result
```

可能失效。

因此 Cache 必须考虑：

```text
TTL
Version
Knowledge Base Revision
Tenant
Permission
```

---

# 46. Retrieval 与数据新鲜度

企业知识库通常不断更新：

```text
Document v1
Document v2
Document v3
```

如果 Retrieval 返回旧版本：

```text
Outdated Context
```

LLM 可能生成：

```text
Outdated Answer
```

因此 Retrieval 需要考虑：

```text
Document Version
Updated At
Effective Date
Expiration
```

例如：

```text
WHERE effective_from <= now()
AND effective_to > now()
```

---

# 47. Temporal Retrieval

有些问题具有时间语义：

```text
2024 年公司的报销政策是什么？
```

和：

```text
2026 年公司的报销政策是什么？
```

答案可能不同。

因此 Retrieval 不仅需要：

```text
Semantic Similarity
```

还需要：

```text
Temporal Filtering
```

即：

```text
Query
 ↓
Time Understanding
 ↓
Metadata Filter
 ↓
Retrieval
```

---

# 48. Retrieval Router

不同 Query 最适合不同 Retrieval Strategy。

例如：

```text
“Redis Cluster 是什么？”
```

适合：

```text
Vector Search
```

而：

```text
“Redis 7.2.1 configuration”
```

可能适合：

```text
Keyword Search
```

复杂问题：

```text
“为什么 Redis Cluster 在网络分区下可能出现……”
```

可以：

```text
Hybrid Search
+
Reranking
```

因此可以增加：

# Retrieval Router

```text
Query
 ↓
Classifier / LLM
 ↓
┌─────────────┬─────────────┬─────────────┐
│ Keyword     │ Vector      │ Hybrid      │
└─────────────┴─────────────┴─────────────┘
```

---

# 49. Agentic Retrieval

传统 RAG：

```text
Question
 ↓
Retrieve
 ↓
Answer
```

Agentic Retrieval：

```text
Question
 ↓
Plan
 ↓
Search
 ↓
Analyze
 ↓
Need More Information?
 ├── Yes → Search Again
 └── No
       ↓
     Answer
```

例如：

```text
“比较 Kafka、Pulsar 和 RabbitMQ 在高吞吐场景下的优缺点。”
```

Agent 可以拆成：

```text
Query 1:
Kafka throughput

Query 2:
Pulsar architecture

Query 3:
RabbitMQ performance

Query 4:
Compare messaging semantics
```

然后：

```text
Parallel Retrieval
 ↓
Merge
 ↓
Evidence Analysis
 ↓
Answer
```

这标志着 Retrieval 从：

```text
Static Search
```

向：

```text
Dynamic Information Seeking
```

演进。

---

# 50. Graph Retrieval

并不是所有知识都适合纯 Vector Retrieval。

例如：

```text
Employee
 ↓
works_for
 ↓
Company
 ↓
owns
 ↓
Project
 ↓
uses
 ↓
Technology
```

这种关系型知识更适合：

```text
Knowledge Graph
```

因此未来 Retrieval 很可能变成：

```text
                Query
                  ↓
          Retrieval Router
                  ↓
      ┌───────────┼───────────┐
      ↓           ↓           ↓
   Keyword     Vector       Graph
      ↓           ↓           ↓
      └───────────┼───────────┘
                  ↓
                Fusion
                  ↓
               Reranker
                  ↓
                Context
                  ↓
                  LLM
```

这就是：

# Multi-Modal Retrieval Architecture

这里的“Multi-Modal”不仅可以指：

```text
Text
Image
Audio
```

也可以理解为：

```text
Keyword
Vector
Graph
Structured Data
```

多种 Retrieval Modality 的融合。

---

# 51. Retrieval 与 AI Agent Memory

AI Agent 需要记住：

```text
Conversation
User Preferences
Previous Tasks
Knowledge
Tool Results
```

这些信息可以进入不同 Memory：

```text
Short-Term Memory
Long-Term Memory
Episodic Memory
Semantic Memory
```

Retrieval 则负责：

```text
当前任务
 ↓
需要什么历史信息？
 ↓
Retrieval
 ↓
Memory
```

因此：

> Retrieval 是 Agent Memory 被“取出来”的核心机制。

---

# 52. 一个成熟 Retrieval Architecture

综合前面的技术，可以得到一个生产级架构：

```text
                           User Query
                               │
                               ↓
                       Query Understanding
                               │
                       ┌───────┴────────┐
                       ↓                ↓
                  Query Rewrite    Query Routing
                       │                │
                       └───────┬────────┘
                               ↓
                    ┌─────────────────────┐
                    │ Retrieval Layer     │
                    │                     │
                    │ BM25                │
                    │ Vector Search       │
                    │ Metadata Filter     │
                    │ Graph Retrieval     │
                    └──────────┬──────────┘
                               ↓
                         Candidate Set
                               ↓
                         Fusion / Merge
                               ↓
                           Reranker
                               ↓
                       Context Compression
                               ↓
                        Context Selection
                               ↓
                              LLM
```

这已经不是简单的：

```text
Vector DB Search
```

而是完整的：

# Retrieval System

---

# 53. Retrieval System 的五层架构

可以进一步抽象为五层。

## Layer 1：Query Layer

负责：

```text
Query Understanding
Query Rewrite
Query Expansion
Query Classification
```

---

## Layer 2：Retrieval Layer

负责：

```text
Keyword Retrieval
Dense Retrieval
Graph Retrieval
Structured Retrieval
```

---

## Layer 3：Ranking Layer

负责：

```text
Fusion
Reranking
Scoring
Deduplication
```

---

## Layer 4：Context Layer

负责：

```text
Context Selection
Compression
Ordering
Parent Retrieval
Token Budget
```

---

## Layer 5：Evaluation & Observability

负责：

```text
Recall
Precision
NDCG
Latency
Cost
Tracing
Quality Evaluation
```

---

# 54. Retrieval 的本质：Information Bottleneck

从架构角度看，LLM 的 Context Window 是有限的。

假设知识库：

```text
10M documents
```

但 LLM 最终只能看到：

```text
10 documents
```

那么 Retrieval 就成为一个：

> **Information Bottleneck。**

它必须完成：

```text
10M
 ↓
1000
 ↓
100
 ↓
10
```

每一次压缩都会产生信息损失。

因此 Retrieval 真正解决的是：

> **如何在有限的信息预算中最大化有效信息密度。**

这比单纯的：

```text
“找到相似文档”
```

要深得多。

---

# 55. Retrieval 的核心 Trade-off

一个生产级 Retrieval System 永远存在以下权衡：

```text
Recall
   ↕
Precision

Latency
   ↕
Quality

K
   ↕
Context Noise

Freshness
   ↕
Index Cost

Accuracy
   ↕
Memory

Reranking Quality
   ↕
Compute Cost
```

不存在一个：

```text
Best Retrieval Algorithm
```

只有：

> **针对特定业务场景的最佳 Retrieval Architecture。**

---

# 56. 从架构师角度重新理解 Retrieval

如果只会：

```text
vector_db.search()
```

只能说明：

> 会调用 Retrieval API。

真正理解 Retrieval，需要能够回答：

```text
为什么使用 BM25？

为什么使用 Vector Search？

为什么需要 Hybrid Search？

为什么 Retriever Top-100？

为什么 Reranker Top-10？

为什么需要 Query Rewrite？

为什么需要 Metadata Filter？

为什么需要 Parent-Child Retrieval？

为什么 Recall 比 Precision 更先优化？

为什么需要 Evaluation Dataset？

为什么 Retrieval Latency 会成为 RAG 性能瓶颈？

为什么权限必须进入 Retrieval？

为什么 Agent 需要 Iterative Retrieval？
```

当能够回答这些问题时，才真正进入：

> Retrieval Architecture。

---

# 57. Retrieval 技术栈全景

可以把现代 Retrieval 技术体系总结成：

```text
                    Retrieval
                        │
        ┌───────────────┼────────────────┐
        ↓               ↓                ↓
    Query           Retrieval         Ranking
  Processing          Engine           Engine
        │               │                │
        ↓               ↓                ↓
 Query Rewrite       BM25            RRF
 Query Expansion     Vector          Reranker
 Multi-Query         Hybrid          NDCG
 HyDE                Graph           Dedup
        │               │                │
        └───────────────┼────────────────┘
                        ↓
                 Context Management
                        │
                        ↓
                 Context Compression
                        │
                        ↓
                       LLM
```

---

# 58. 总结

Retrieval 是 AI 应用中连接：

```text
Knowledge
        ↓
LLM
```

的核心桥梁。

它经历了：

```text
Keyword Search
      ↓
BM25
      ↓
Dense Retrieval
      ↓
Vector Search
      ↓
Hybrid Search
      ↓
Reranking
      ↓
Query Optimization
      ↓
Multi-Stage Retrieval
      ↓
Agentic Retrieval
```

一个成熟的 Retrieval Pipeline 通常是：

```text
User Query
    ↓
Query Understanding
    ↓
Query Rewrite
    ↓
Query Routing
    ↓
┌───────────────────────┐
│ Keyword Retrieval     │
│ Vector Retrieval      │
│ Metadata Filtering    │
│ Graph Retrieval       │
└───────────┬───────────┘
            ↓
       Candidate Set
            ↓
       Fusion / Merge
            ↓
         Reranker
            ↓
     Context Compression
            ↓
      Context Selection
            ↓
            LLM
```

而 Retrieval 的最终目标不是：

> **找到最多的文档。**

也不是：

> **找到最相似的向量。**

而是：

> **在有限的延迟、计算、Token 和安全约束下，为下游模型提供最相关、最可靠、最新且用户有权限访问的信息。**

因此，如果把现代 AI Application 抽象成：

```text
AI Application
      ↓
     Agent
      ↓
     RAG
      ↓
  Retrieval
      ↓
┌─────┼──────┐
│     │      │
BM25 Vector Graph
│     │      │
└─────┼──────┘
      ↓
   Reranker
      ↓
   Context
      ↓
     LLM
```

那么可以认为：

> **LLM 决定 AI 能不能“理解和生成”，而 Retrieval 决定 AI 能不能“找到正确的信息”。**

这也是为什么在企业级 AI 系统中，Retrieval 正逐渐从一个简单的 Search Component，演变为独立的 **Retrieval Engineering / Retrieval Architecture** 领域。
