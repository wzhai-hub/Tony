---
title: Advanced RAG：从向量检索到智能知识检索系统
# tags:
#   - nodejs
date: '2026-08-05'
summary: Advanced RAG 的本质，不是增加更多组件，而是让 Retrieval 从“单次相似度搜索”演变成一个多阶段、可控制、可评估的知识检索系统。
---

## 引言：RAG 的真正难点，从来不是“接一个 Vector Database”

Retrieval-Augmented Generation（RAG）最初给人的印象非常简单：

```text
User Query
    │
    ▼
Embedding
    │
    ▼
Vector Database
    │
    ▼
Top-K Documents
    │
    ▼
LLM
    │
    ▼
Answer
```

这种架构可以很好地解决一部分知识问答问题。

但是，一旦进入真实生产环境，问题马上出现：

* 用户的问题表达不准确；
* Query 和文档中的语言不一致；
* 一个问题涉及多个知识点；
* 关键词检索和语义检索各有缺陷；
* Top-K 中存在大量相似但不相关的 Chunk；
* 正确答案分散在多个文档中；
* Chunk 本身缺少上下文；
* 文档存在权限和版本问题；
* LLM Context Window 被大量无关内容占用；
* Retrieval 正确，但最终答案仍然错误；
* 无法解释为什么检索到这些文档；
* 无法系统评估 RAG 到底哪里出了问题。

于是，真正的生产级 RAG 不再是：

```text
Embedding → Vector DB → LLM
```

而逐渐演化成：

```text
                    User Query
                        │
                        ▼
                Query Understanding
                        │
                        ▼
                Query Transformation
                        │
                        ▼
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
        Dense Retrieval       Sparse Retrieval
             │                     │
             └──────────┬──────────┘
                        ▼
                   Fusion
                        │
                        ▼
                 Candidate Set
                        │
                        ▼
                    Reranker
                        │
                        ▼
               Context Compression
                        │
                        ▼
                Context Assembly
                        │
                        ▼
                      LLM
                        │
                        ▼
                Answer Validation
```

这就是 Advanced RAG 的核心。

> **Advanced RAG 的本质，不是增加更多组件，而是让 Retrieval 从“单次相似度搜索”演变成一个多阶段、可控制、可评估的知识检索系统。**

---

# 1. Naive RAG 的问题到底在哪里？

首先来看最基本的 RAG。

假设知识库：

```text
Document
   │
   ├── Chunk 1
   ├── Chunk 2
   ├── Chunk 3
   └── Chunk N
```

Index：

```text
Chunk
  │
  ▼
Embedding
  │
  ▼
Vector
  │
  ▼
Vector DB
```

Query：

```text
How can I configure Redis cluster failover?
```

转换：

```text
Query
  │
  ▼
Embedding
  │
  ▼
Vector Search
  │
  ▼
Top 5 Chunks
```

理论上非常简单。

但是这里隐含了一个非常强的假设：

> **Query 的 embedding 与正确答案 Chunk 的 embedding 足够接近。**

现实世界并不总是如此。

---

# 2. Semantic Gap：Query 和 Document 的语言可能完全不同

例如用户问：

```text
Why did my Redis cluster automatically switch to another node?
```

文档可能写的是：

```text
When the master node becomes unavailable,
Redis Cluster performs replica promotion.
```

用户没有使用：

```text
master
replica
promotion
```

这些关键词。

但是语义上：

```text
automatically switch to another node
```

其实对应：

```text
replica promotion
```

这就是：

> Semantic Gap

Vector Search 可以解决一部分 Semantic Gap，但不是全部。

因此 Advanced RAG 的第一个方向就是：

> **Query Transformation**

---

# 3. Query Transformation

Query Transformation 的思想是：

> 不要直接拿用户原始 Query 去搜索。

而是先理解 Query：

```text
User Query
    │
    ▼
Query Understanding
    │
    ▼
Transformed Query
    │
    ▼
Retriever
```

---

# 4. Query Rewriting

最简单的方法：

```text
Original:

Why did my Redis cluster automatically switch to another node?
```

Rewrite：

```text
Redis Cluster replica promotion after master node failure
```

这样可以显著改善：

```text
Query
  ↓
Embedding
  ↓
Retrieval
```

的匹配质量。

例如：

```text
Original Query
        │
        ▼
LLM Rewrite
        │
        ▼
Search Query
```

但 Query Rewriting 也有一个风险：

> LLM 可能在 Rewrite 时改变用户原始意图。

所以生产环境中应该保留：

```text
original_query
rewritten_query
```

而不是直接覆盖原 Query。

---

# 5. Multi-Query Retrieval

一个 Query 有时候存在多个合理的表达方式。

例如：

```text
How does Redis failover work?
```

可以生成：

```text
Query 1:
How does Redis detect master failure?

Query 2:
How is a Redis replica promoted?

Query 3:
How does Redis Cluster recover from node failure?

Query 4:
Redis Cluster automatic failover mechanism
```

然后分别进行 Retrieval：

```text
                    Original Query
                         │
                         ▼
                  Query Generator
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
         Q1             Q2             Q3
          │              │              │
          ▼              ▼              ▼
       Search          Search         Search
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                    Merge Results
```

这样可以提高 Recall。

但是也会增加：

```text
LLM Cost
Embedding Cost
Retrieval Latency
```

所以 Multi-Query 不应该对所有 Query 默认开启。

---

# 6. HyDE：先生成“假答案”，再进行 Retrieval

HyDE（Hypothetical Document Embeddings）是一个非常有意思的思想。

传统 RAG：

```text
Query
  │
  ▼
Embedding
  │
  ▼
Vector Search
```

HyDE：

```text
Query
  │
  ▼
LLM
  │
  ▼
Hypothetical Answer
  │
  ▼
Embedding
  │
  ▼
Vector Search
```

例如：

```text
Query:

How does Redis failover work?
```

LLM 先生成：

```text
Redis Cluster detects master failure and promotes
one of the replicas to become the new master...
```

然后：

```text
Hypothetical Answer
        │
        ▼
     Embedding
        │
        ▼
Vector Search
```

为什么可能有效？

因为：

```text
Query
```

通常非常短。

而：

```text
Hypothetical Answer
```

包含更多与知识库文档相似的语言。

于是：

```text
Query ↔ Document
```

变成：

```text
Hypothetical Answer ↔ Document
```

语义空间可能更加接近。

---

# 7. Multi-Query、Rewrite、HyDE 的本质

虽然它们实现方式不同，但本质上都在解决：

```text
User Query
      │
      ▼
Retrieval Space
```

之间的：

> **Representation Mismatch**

也就是说：

```text
Query Representation
        ≠
Document Representation
```

Query Transformation 的目标就是：

```text
Transform Query
       ↓
Make Query Representation
closer to Document Representation
```

这是 Advanced RAG 的第一个重要升级。

---

# 8. Hybrid Search：为什么 Vector Search 不够？

另一个经典问题：

> Vector Search 并不擅长所有类型的查询。

例如：

```text
INC-2026-001827
```

或者：

```text
ERR_CONNECTION_RESET
```

或者：

```text
java.lang.NullPointerException
```

这种 Query 对关键词非常敏感。

如果用户问：

```text
Find incident INC-2026-001827
```

传统 Dense Retrieval 未必是最好的方法。

这时需要：

```text
Dense Retrieval
+
Sparse Retrieval
```

---

# 9. Dense Retrieval

Dense Retrieval：

```text
Text
 │
 ▼
Embedding Model
 │
 ▼
Vector
 │
 ▼
Similarity Search
```

优势：

* 语义理解；
* 同义词；
* 自然语言表达；
* Concept-level matching。

例如：

```text
automatically switch node
```

可以匹配：

```text
replica promotion
```

---

# 10. Sparse Retrieval

Sparse Retrieval 典型方法：

```text
BM25
```

核心优势是：

> Exact Keyword Matching

例如：

```text
INC-2026-001827
```

或者：

```text
RedisTemplate
```

或者：

```text
OpenTelemetry Java Agent
```

这种专有名词、错误码、ID、类名、API 名称，Sparse Search 往往非常有效。

---

# 11. Hybrid Retrieval

因此生产级 RAG 通常采用：

```text
                  Query
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
    Dense Retrieval      Sparse Retrieval
          │                   │
          ▼                   ▼
       Top 50              Top 50
          │                   │
          └─────────┬─────────┘
                    ▼
                 Fusion
                    │
                    ▼
                 Top 50
```

这实际上是在组合：

```text
Semantic Search
+
Lexical Search
```

---

# 12. Reciprocal Rank Fusion

Hybrid Search 的一个常见问题是：

> Dense Search 和 Sparse Search 的 score 不能直接比较。

例如：

```text
Dense Score:
0.92
0.87
0.83
```

而 BM25：

```text
BM25:
18.2
12.7
9.3
```

两个 score scale 完全不同。

因此可以使用：

> Reciprocal Rank Fusion（RRF）

基本思想：

```text
RRF(d) =
Σ 1 / (k + rank(d))
```

例如：

```text
Dense:

Document A → Rank 1
Document B → Rank 2

BM25:

Document B → Rank 1
Document C → Rank 2
```

那么：

```text
Document B
```

同时在两个 Retriever 中排名靠前，因此最终排名会上升。

RRF 的价值在于：

> 不要求不同 Retriever 的 score 具有相同尺度。

---

# 13. Reranking：Retrieval 的第二阶段

Hybrid Search 解决了 Recall。

但是：

```text
Top 50
```

仍然太多。

所以进入第二阶段：

```text
Candidate Retrieval
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

这就是：

> Multi-Stage Retrieval

---

# 14. Bi-Encoder vs Cross-Encoder

这是理解 RAG Retrieval Architecture 的关键。

## Bi-Encoder

Embedding Model：

```text
Query → Vector
Document → Vector
```

然后：

```text
Similarity(Query, Document)
```

优势：

```text
Fast
Scalable
Pre-computable
```

因为 Document Embedding 可以提前计算。

---

## Cross-Encoder

Cross-Encoder：

```text
(Query, Document)
       │
       ▼
    Model
       │
       ▼
Relevance Score
```

它直接同时理解：

```text
Query
+
Document
```

因此通常具有更强的相关性判断能力。

但是计算成本更高：

```text
Query × Candidate Documents
```

所以：

```text
Bi-Encoder
```

适合：

> First-stage Retrieval

而：

```text
Cross-Encoder
```

适合：

> Second-stage Reranking

于是形成经典架构：

```text
                    Query
                      │
                      ▼
                Dense Retrieval
                      │
                      ▼
                    Top 100
                      │
                      ▼
                Sparse Retrieval
                      │
                      ▼
                   Fusion
                      │
                      ▼
                    Top 50
                      │
                      ▼
                 Cross Encoder
                      │
                      ▼
                     Top 5
```

---

# 15. Context Compression

即使经过 Reranking：

```text
Top 5 Chunks
```

每个 Chunk 可能仍然有大量无关内容。

例如：

```text
Chunk:

Redis Cluster consists of...
[大量背景介绍]

...
When a master node fails...
[真正答案]

...
Monitoring...
[大量无关信息]
```

如果直接送给 LLM：

```text
Chunk
 ↓
LLM
```

会浪费大量 Context Window。

所以 Advanced RAG 会增加：

> Context Compression

---

# 16. Context Compression 的核心思想

不是：

> 找更多内容。

而是：

> **从已经找到的内容中提取真正有用的内容。**

架构：

```text
Retriever
   │
   ▼
Top 10 Chunks
   │
   ▼
Compression
   │
   ▼
Relevant Passages
   │
   ▼
LLM
```

这可以降低：

```text
Context Tokens
LLM Cost
Latency
Noise
```

同时提高：

```text
Context Precision
```

---

# 17. Parent-Child Retrieval

这是解决 Chunking 与 Context 问题的重要技术。

Index：

```text
Parent Document
       │
       ├── Child Chunk 1
       ├── Child Chunk 2
       ├── Child Chunk 3
       └── Child Chunk 4
```

Embedding：

```text
Child Chunk → Vector
```

Retrieval：

```text
Query
 ↓
Child Chunk
 ↓
Parent
 ↓
Context
```

也就是说：

> 用小 Chunk 找，用大 Chunk 读。

这是一个非常漂亮的设计：

```text
Retrieval Granularity
        ≠
Generation Granularity
```

这两个问题不应该强行使用同一个 Chunk。

---

# 18. Multi-Hop Retrieval

很多企业问题不是：

```text
Question → One Chunk
```

而是：

```text
Question → Chunk A + Chunk B + Chunk C
```

例如：

> Which service owns the API, who is the technical lead, and what Kubernetes namespace is it deployed in?

需要：

```text
Service
   │
   ▼
Ownership
   │
   ▼
Technical Lead
   │
   ▼
Deployment
   │
   ▼
Kubernetes Namespace
```

这种问题需要：

> Multi-Hop Retrieval

---

# 19. Multi-Hop RAG

可以设计：

```text
                  Complex Query
                       │
                       ▼
                Query Decomposition
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
             Q1       Q2       Q3
              │        │        │
              ▼        ▼        ▼
           Search    Search    Search
              │        │        │
              └────────┼────────┘
                       ▼
                  Intermediate
                   Knowledge
                       │
                       ▼
                 Next Retrieval
                       │
                       ▼
                    Context
                       │
                       ▼
                      LLM
```

这里 Retrieval 不再是一次性的：

```text
Retrieve once
```

而是：

```text
Retrieve
  ↓
Reason
  ↓
Retrieve again
  ↓
Reason
```

这已经开始接近：

> Agentic Retrieval。

---

# 20. Query Decomposition

复杂问题可以拆成多个 Sub-Queries。

例如：

```text
How does our payment service handle
failure recovery and what monitoring
metrics should we use?
```

可以拆成：

```text
Q1:
How does payment service failure recovery work?

Q2:
What are the recovery mechanisms?

Q3:
What monitoring metrics are available?
```

分别 Retrieval：

```text
Q1 → Architecture Docs
Q2 → Recovery Docs
Q3 → Monitoring Docs
```

然后：

```text
Merge
 ↓
Rerank
 ↓
Context
 ↓
LLM
```

这对于复杂企业知识库非常重要。

---

# 21. Self-RAG

传统 RAG：

```text
Retrieve
 ↓
Generate
```

Self-RAG 的核心思想是：

> LLM 自己判断什么时候需要 Retrieval，以及检索结果是否足够支持答案。

可以抽象为：

```text
Question
   │
   ▼
Need Retrieval?
   │
 ┌─┴─┐
No  Yes
│     │
│     ▼
│   Retrieve
│     │
│     ▼
│  Relevant?
│     │
└──┬──┘
   ▼
Generate
   │
   ▼
Critique
```

因此 RAG 从：

```text
Pipeline
```

开始向：

```text
Decision Loop
```

演化。

---

# 22. Corrective RAG

Corrective RAG（CRAG）的核心思想：

> Retrieval 结果可能是错误的，所以需要对 Retrieval Quality 进行判断。

流程：

```text
Query
  │
  ▼
Retriever
  │
  ▼
Retrieved Documents
  │
  ▼
Retrieval Evaluator
  │
  ├── Good
  │    ↓
  │   Generate
  │
  └── Bad
       ↓
   Correct / Reformulate
       ↓
   External Search / Another Retriever
       ↓
     Generate
```

这里增加了：

> Retrieval Critic

这说明一个非常重要的架构思想：

> **Retriever 不是绝对可靠的。**

---

# 23. Graph RAG

普通 RAG：

```text
Query
 ↓
Vector Similarity
 ↓
Chunks
```

Graph RAG：

```text
Documents
   │
   ▼
Entities
   │
   ▼
Relationships
   │
   ▼
Knowledge Graph
```

例如：

```text
PaymentService
      │
      ├── calls → OrderService
      │
      ├── uses → Redis
      │
      └── deployed-in → payment-prod
```

用户问：

> Which services depend on Redis?

Vector Search 可能找到：

```text
PaymentService
OrderService
RiskService
```

但 Graph Query 可以直接：

```text
Redis
 ↑
uses
 ↑
Services
```

这对于：

```text
Relationship Query
Multi-hop Query
Dependency Analysis
Architecture Analysis
```

非常有效。

---

# 24. Vector RAG 和 Graph RAG 不是互斥的

高级系统通常不是：

```text
Vector RAG
OR
Graph RAG
```

而是：

```text
Vector Retrieval
+
Graph Retrieval
+
Metadata Filtering
+
Keyword Search
```

例如：

```text
                       Query
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Vector          BM25           Graph
       Search          Search         Query
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                       Fusion
                         │
                         ▼
                      Reranker
                         │
                         ▼
                        LLM
```

这才是真正意义上的：

> Multi-Retriever Architecture。

---

# 25. Metadata Filtering

企业 RAG 还有一个非常重要的问题：

> 不是所有用户都有权限访问所有知识。

例如：

```text
Document
 ├── department = HR
 ├── department = Finance
 ├── department = Engineering
 └── department = Security
```

用户：

```text
department = Engineering
```

那么 Retrieval 应该首先限制：

```text
department = Engineering
```

再进行：

```text
Vector Search
```

也就是：

```text
User Query
   │
   ▼
Authorization Filter
   │
   ▼
Candidate Documents
   │
   ▼
Vector Retrieval
```

而不是：

```text
Vector Search
   │
   ▼
Retrieve unauthorized documents
   │
   ▼
Filter
```

后者存在严重的数据泄露风险。

---

# 26. Temporal Retrieval

企业知识库还存在：

> Version Problem

例如：

```text
API Documentation v1
API Documentation v2
API Documentation v3
```

用户问：

> What is the current timeout configuration?

如果 Retrieval 同时找到：

```text
v1
v2
v3
```

LLM 可能把旧版本和新版本混在一起。

所以 Metadata 应该包含：

```text
version
effective_from
effective_to
status
```

然后：

```text
Query
 ↓
Current Version Filter
 ↓
Retrieval
```

这就是：

> Temporal-Aware Retrieval。

---

# 27. Advanced RAG 的完整架构

把前面的技术组合起来，可以形成一个生产级 Advanced RAG：

```text
                         User Query
                              │
                              ▼
                    Query Understanding
                              │
                              ▼
                    Query Classification
                              │
               ┌──────────────┼──────────────┐
               │              │              │
               ▼              ▼              ▼
          Query Rewrite   Decomposition     HyDE
               │              │              │
               └──────────────┼──────────────┘
                              ▼
                     Multi-Retriever
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
     Dense Search         BM25 Search        Graph Search
          │                   │                   │
          └───────────────────┼───────────────────┘
                              ▼
                         Fusion / RRF
                              │
                              ▼
                         Candidate Set
                              │
                              ▼
                          Reranker
                              │
                              ▼
                       Parent Retrieval
                              │
                              ▼
                     Context Compression
                              │
                              ▼
                      Context Assembly
                              │
                              ▼
                             LLM
                              │
                              ▼
                       Answer Validation
                              │
                              ▼
                         Final Answer
```

这已经远远超过：

```text
Vector DB + LLM
```

---

# 28. Advanced RAG 的关键不是组件，而是 Pipeline Design

一个常见错误是：

```text
我们增加 BM25。
```

然后：

```text
我们增加 Reranker。
```

然后：

```text
我们增加 Graph RAG。
```

最后：

```text
系统越来越复杂。
```

但效果却没有明显提高。

原因是：

> Advanced RAG 不是组件堆砌。

真正应该考虑的是：

```text
Query
 ↓
What kind of question?
 ↓
What retrieval strategy?
 ↓
How many candidates?
 ↓
How to rank?
 ↓
How much context?
 ↓
Is evidence sufficient?
 ↓
Should we retrieve again?
 ↓
Can the answer be supported?
```

也就是说：

> **Advanced RAG 的核心是 Retrieval Control。**

---

# 29. Query Router

因此，一个高级 RAG 系统通常需要 Query Router。

例如：

```text
                     Query
                       │
                       ▼
                    Router
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
    Simple FAQ      Keyword       Complex Query
        │              │              │
        ▼              ▼              ▼
   Vector Search      BM25       Multi-Hop RAG
        │              │              │
        └──────────────┼──────────────┘
                       ▼
                    Answer
```

例如：

```text
"How do I reset my password?"
```

直接 FAQ Retrieval。

而：

```text
"Which services depend on Redis and
what happens if Redis becomes unavailable?"
```

则进入：

```text
Graph + Vector + Multi-Hop
```

这比所有 Query 都走同一条 Pipeline 更高效。

---

# 30. RAG Evaluation：Advanced RAG 的核心基础设施

如果没有 Evaluation：

```text
RAG Pipeline
```

很容易变成：

```text
Engineering Guess
```

成熟系统应该至少分成三层 Evaluation。

---

## 30.1 Retrieval Evaluation

衡量：

```text
Recall@K
Precision@K
MRR
NDCG
Hit Rate
```

核心问题：

> 正确知识有没有被找回来？

---

## 30.2 Context Evaluation

即使 Retrieval 正确，也要判断：

```text
Context Relevance
Context Precision
Context Completeness
```

核心问题：

> 找回来的内容是否足够支持回答？

---

## 30.3 Generation Evaluation

最终：

```text
Answer Accuracy
Faithfulness
Groundedness
Citation Correctness
```

核心问题：

> LLM 是否基于证据正确回答？

---

# 31. RAG Evaluation 应该形成一条完整链路

不要只测试：

```text
LLM Answer
```

而应该：

```text
Query
 │
 ▼
Retrieval
 │
 ├── Recall
 ├── Precision
 ├── MRR
 └── NDCG
 │
 ▼
Context
 │
 ├── Relevance
 ├── Completeness
 └── Noise
 │
 ▼
Generation
 │
 ├── Accuracy
 ├── Faithfulness
 └── Citation
```

这样才能定位：

> 到底是 Retrieval 错了，还是 Generation 错了。

---

# 32. Observability：RAG 也需要 Distributed Tracing

Advanced RAG 系统越来越像一个分布式系统。

一次请求可能经过：

```text
API
 ↓
Query Router
 ↓
Embedding Service
 ↓
Vector DB
 ↓
BM25
 ↓
Graph DB
 ↓
Reranker
 ↓
LLM
```

因此非常适合建立完整 Trace：

```text
trace_id
   │
   ├── query_rewrite
   ├── embedding
   ├── vector_search
   ├── bm25_search
   ├── graph_search
   ├── fusion
   ├── reranking
   ├── compression
   └── llm_generation
```

每一个 Span 记录：

```text
latency
token_usage
top_k
scores
document_ids
model
prompt
retrieval_strategy
```

这样当用户说：

> “为什么这个回答错了？”

工程师可以真正追踪：

```text
Query
 ↓
Retrieved Documents
 ↓
Ranking
 ↓
Context
 ↓
Prompt
 ↓
LLM
 ↓
Answer
```

这就是：

> **RAG Observability。**

---

# 33. Advanced RAG 的成本模型

RAG 不只是 Accuracy 问题。

还需要考虑：

```text
Cost
Latency
Throughput
Scalability
```

例如：

```text
Query Rewrite
    +
Multi Query
    +
Dense Search
    +
BM25
    +
Reranking
    +
Context Compression
    +
LLM
```

如果每一个 Query 都走完整 Pipeline：

```text
Latency ↑
Cost ↑
```

所以生产系统通常应该：

> **根据 Query Complexity 动态选择 Pipeline。**

例如：

```text
Simple Query
    ↓
Vector Search
    ↓
LLM
```

而：

```text
Complex Query
    ↓
Rewrite
    ↓
Hybrid Search
    ↓
Rerank
    ↓
Multi-Hop
    ↓
Compression
    ↓
LLM
```

---

# 34. 从 Advanced RAG 到 Agentic RAG

Advanced RAG 仍然主要是：

```text
Pipeline
```

而 Agentic RAG 更进一步：

```text
Goal
 │
 ▼
Agent
 │
 ├── Search
 ├── Reason
 ├── Search Again
 ├── Call Tool
 ├── Validate
 └── Search Again
 │
 ▼
Answer
```

区别可以简单理解为：

### Traditional RAG

```text
Query
 ↓
Retrieve
 ↓
Generate
```

### Advanced RAG

```text
Query
 ↓
Transform
 ↓
Retrieve
 ↓
Rerank
 ↓
Compress
 ↓
Generate
```

### Agentic RAG

```text
Goal
 ↓
Plan
 ↓
Retrieve
 ↓
Reason
 ↓
Evaluate
 ↓
Retrieve Again
 ↓
Tool Call
 ↓
Verify
 ↓
Answer
```

这代表 RAG 从：

> Retrieval Pipeline

逐渐演化成：

> **Knowledge-Seeking Agent。**

---

# 35. Advanced RAG 技术栈可以如何分层？

可以把整个技术体系划分成六层。

```text
┌──────────────────────────────┐
│       Generation Layer       │
│       LLM / Citation         │
├──────────────────────────────┤
│       Context Layer          │
│ Compression / Assembly       │
├──────────────────────────────┤
│       Ranking Layer          │
│       Reranker / Fusion      │
├──────────────────────────────┤
│       Retrieval Layer        │
│ Vector / BM25 / Graph        │
├──────────────────────────────┤
│       Query Layer            │
│ Rewrite / HyDE / Decompose   │
├──────────────────────────────┤
│       Knowledge Layer        │
│ Chunking / Metadata / Index  │
└──────────────────────────────┘
```

这样看，RAG 就不再是：

```text
Vector DB + LLM
```

而是一整套：

> **Knowledge Retrieval Architecture**

---

# 36. Advanced RAG 的核心设计原则

最终可以总结出几个非常重要的工程原则。

## 原则一：不要迷信 Vector Search

Vector Search 只是 Retriever 之一。

应该根据 Query 类型组合：

```text
Dense
+
Sparse
+
Metadata
+
Graph
```

---

## 原则二：不要把 Top-K 当作 Retrieval 的终点

真正的 Pipeline 应该是：

```text
Retrieve
 ↓
Fuse
 ↓
Rerank
 ↓
Compress
 ↓
Assemble
```

---

## 原则三：Retrieval Granularity 和 Generation Granularity 可以不同

这是：

```text
Parent-Child Retrieval
```

最核心的思想。

---

## 原则四：复杂 Query 应该分解

不要强迫一个 Retriever 解决：

```text
A + B + C + D
```

可以：

```text
Q
 ↓
Q1 + Q2 + Q3
 ↓
Retrieve
 ↓
Merge
```

---

## 原则五：Retrieval 结果应该被验证

不要假设：

```text
Retriever = Always Correct
```

应该增加：

```text
Evaluator
Critic
Corrective Retrieval
```

---

## 原则六：RAG 必须可观测

如果不知道：

```text
What was retrieved?
Why was it retrieved?
What was the score?
What context reached LLM?
```

那么生产环境很难真正维护 RAG。

---

# 37. 一个成熟的 Advanced RAG Blueprint

如果让我设计一个企业级 RAG 平台，我会优先考虑如下架构：

```text
                           User
                            │
                            ▼
                     API / Gateway
                            │
                            ▼
                     Query Router
                            │
              ┌─────────────┴─────────────┐
              │                           │
        Simple Query                Complex Query
              │                           │
              ▼                           ▼
       Basic Retrieval              Query Planner
                                          │
                              ┌───────────┼───────────┐
                              ▼           ▼           ▼
                           Rewrite     Decompose     HyDE
                              │           │           │
                              └───────────┼───────────┘
                                          ▼
                                  Retrieval Layer
                                          │
                       ┌──────────────────┼──────────────────┐
                       ▼                  ▼                  ▼
                     Vector             BM25               Graph
                       │                  │                  │
                       └──────────────────┼──────────────────┘
                                          ▼
                                       Fusion
                                          │
                                          ▼
                                      Reranker
                                          │
                                          ▼
                                   Parent Retrieval
                                          │
                                          ▼
                                 Context Compression
                                          │
                                          ▼
                                  Context Assembly
                                          │
                                          ▼
                                         LLM
                                          │
                                          ▼
                                   Answer Validator
                                          │
                              ┌───────────┴───────────┐
                              │                       │
                           Pass                     Fail
                              │                       │
                              ▼                       ▼
                           Answer                Retrieve Again
```

外围再增加：

```text
Evaluation
Observability
Security
Authorization
Caching
Cost Control
```

这才是真正可以进入生产环境的 RAG Architecture。

---

# 38. Conclusion：Advanced RAG 的本质

如果用一句话定义 Advanced RAG：

> **Advanced RAG 是通过 Query Understanding、多路 Retrieval、Ranking、Context Engineering、Validation 和 Feedback Loop，将一次简单的向量搜索升级为一个可控制、可评估、可优化的知识检索系统。**

Naive RAG：

```text
Query
 ↓
Vector Search
 ↓
LLM
```

Advanced RAG：

```text
Query
 ↓
Understand
 ↓
Transform
 ↓
Decompose
 ↓
Multi-Retrieval
 ↓
Fusion
 ↓
Rerank
 ↓
Parent Retrieval
 ↓
Compress
 ↓
Context Assembly
 ↓
Generate
 ↓
Validate
```

而进一步的 Agentic RAG：

```text
Goal
 ↓
Plan
 ↓
Retrieve
 ↓
Reason
 ↓
Evaluate
 ↓
Retrieve Again
 ↓
Tool
 ↓
Verify
 ↓
Answer
```

因此，真正值得研究的 RAG 已经不再是：

> **How do I connect an LLM to a Vector Database?**

而是：

> **How do I build a reliable knowledge retrieval and reasoning system around an LLM?**

这也是 Advanced RAG 与普通 RAG 最大的区别。

从架构演进的角度看，可以把整个路线总结为：

```text
Naive RAG
    │
    ▼
Chunking Optimization
    │
    ▼
Hybrid Retrieval
    │
    ▼
Reranking
    │
    ▼
Query Transformation
    │
    ▼
Context Engineering
    │
    ▼
Multi-Hop Retrieval
    │
    ▼
Corrective / Self RAG
    │
    ▼
Graph RAG
    │
    ▼
Agentic RAG
```

最终，RAG 的竞争力并不只是来自一个更强的 LLM，而越来越来自：

```text
Better Knowledge
+
Better Retrieval
+
Better Context
+
Better Reasoning
+
Better Evaluation
+
Better Observability
```

而这也意味着，对于希望从传统后端/全栈工程师向 **AI Full-Stack / AI Engineer / AI Architect** 转型的开发者来说，真正值得掌握的并不是某一个 RAG Framework 的 API，而是背后的这套 **Retrieval Architecture 思维**。
