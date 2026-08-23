---
title: AI Cost Optimization：从 Token 控制到 Agent 成本工程的系统化实践
# tags:
#   - nodejs
date: '2026-08-05'
summary: 正确的问题使用正确的模型，在正确的 Context 下，以正确的 Agent 策略完成任务，并在整个生命周期内受到成本预算和质量指标约束
---
# AI Cost Optimization：从 Token 控制到 Agent 成本工程的系统化实践

> **摘要**
> 当企业开始规模化使用 LLM、RAG 和 Agent 后，“模型效果”不再是唯一的工程指标。一个 Agent 请求可能经历多轮推理、工具调用、检索、上下文压缩以及多个模型协作，最终成本可能是一次普通 Chat 请求的数十倍甚至上百倍。
>
> 因此，AI 系统需要从传统的 Infrastructure Cost Optimization，进一步演进到 **AI Cost Optimization**：在保证质量、延迟和可靠性的前提下，对模型、Token、Context、RAG、Agent、缓存、并发以及基础设施进行系统化成本治理。
>
> 本文从 AI 系统架构师视角，建立一套完整的 AI 成本优化体系，并重点讨论 **Token Economics、Model Routing、Context Engineering、Semantic Cache、RAG Optimization、Agent Cost Control、Observability 和 FinOps**。

---

# 1. 为什么 AI 系统需要 Cost Optimization？

传统互联网系统的成本模型通常比较容易理解：

```text
User
  ↓
API Gateway
  ↓
Application
  ↓
Database / Cache
  ↓
Infrastructure
```

成本主要来自：

* CPU
* Memory
* Storage
* Network
* Database
* Kubernetes
* Cloud Infrastructure

而 LLM 应用的成本结构完全不同：

```text
User
  ↓
Agent
  ↓
Prompt
  ↓
Context
  ↓
RAG
  ↓
LLM
  ↓
Tool Calling
  ↓
LLM
  ↓
LLM
  ↓
Final Answer
```

一次用户请求可能产生：

```text
Input Tokens
      +
Retrieved Tokens
      +
Conversation History
      +
System Prompt
      +
Tool Result
      +
Output Tokens
      +
Embedding Calls
      +
Reranking Calls
      +
Multiple LLM Calls
```

因此，AI 系统的核心问题不再是：

> “这个 API 调用了多少次？”

而是：

> **“一次业务任务到底消耗了多少 AI Compute？”**

---

# 2. AI Cost 的基本数学模型

可以把一次 AI 请求的成本抽象为：

[
C_{request} =
C_{LLM}
+
C_{Embedding}
+
C_{Reranking}
+
C_{Tool}
+
C_{Infrastructure}
]

其中：

[
C_{LLM}
=======

\sum_{i=1}^{n}
(
InputTokens_i \times P_{in}
+
OutputTokens_i \times P_{out}
)
]

如果一个 Agent 执行：

```text
1. Planner
2. Search
3. RAG
4. Tool Call
5. Reflection
6. Final Answer
```

那么它实际上不是一次模型调用，而可能是：

```text
LLM Call #1
LLM Call #2
LLM Call #3
LLM Call #4
LLM Call #5
LLM Call #6
```

假设每次调用平均消耗：

```text
Input  = 8K tokens
Output = 2K tokens
```

那么：

[
TotalTokens = 6 \times (8K + 2K)
]

也就是：

```text
60K tokens / request
```

如果每天执行：

```text
100,000 requests
```

那么每天就是：

```text
6 billion tokens
```

这时，即使单次请求成本看起来很低，规模化之后也会形成非常大的费用。

---

# 3. Cost Optimization 的核心思想

AI Cost Optimization 不能简单理解为：

> “换一个便宜的模型。”

真正成熟的优化体系应该是：

```text
                    AI Cost Optimization
                            │
        ┌───────────────────┼───────────────────┐
        ↓                   ↓                   ↓
   Model Optimization   Context Optimization   Architecture
        │                   │                   │
   Model Routing       Prompt Optimization   Cache
   Model Cascade       Context Window        RAG
   Batch Inference     Compression            Agent
   Quantization        Summarization           Queue
        │                   │                   │
        └───────────────────┼───────────────────┘
                            ↓
                    Cost Observability
                            │
                            ↓
                       FinOps / Governance
```

可以总结为六个方向：

1. **少调用模型**
2. **少传 Token**
3. **使用更便宜的模型**
4. **减少 Agent 无效推理**
5. **提高缓存命中率**
6. **建立完整的成本可观测性**

---

# 4. 第一原则：Don't Call LLM If You Don't Need It

这是最重要的成本优化原则。

很多 AI 系统存在一个典型问题：

```text
所有请求
   ↓
LLM
```

实际上很多任务根本不需要 LLM。

例如：

```text
"订单 12345 当前状态是什么？"
```

如果数据库已经存在：

```text
order.status = SHIPPED
```

完全没有必要：

```text
User
 ↓
LLM
 ↓
Tool
 ↓
Database
 ↓
LLM
 ↓
Answer
```

更合理的方式是：

```text
User
 ↓
Intent Router
 ├── deterministic query
 │       ↓
 │    Database
 │
 └── complex reasoning
         ↓
        LLM
```

这就是：

> **LLM as a selective reasoning engine，而不是整个系统的默认执行引擎。**

---

# 5. Model Routing：不要让所有请求使用最贵模型

这是 AI Cost Optimization 最重要的架构模式之一。

假设系统拥有：

```text
Model A
高能力
高成本

Model B
中等能力
中等成本

Model C
基础能力
低成本
```

不要设计成：

```text
所有请求
   ↓
Model A
```

而应该：

```text
                Request
                   │
                   ↓
              Classifier
                   │
        ┌──────────┼──────────┐
        ↓          ↓          ↓
      Easy       Medium      Hard
        │          │          │
        ↓          ↓          ↓
     Model C     Model B    Model A
```

例如：

| 请求类型       | 推荐模型         |
| ---------- | ------------ |
| 分类         | Small Model  |
| 信息抽取       | Small Model  |
| FAQ        | Small Model  |
| 简单代码生成     | Medium Model |
| RAG QA     | Medium Model |
| 复杂推理       | Large Model  |
| 高难度 Coding | Large Model  |

---

# 6. Model Cascade

更进一步，可以设计 Model Cascade。

例如：

```text
Request
   ↓
Small Model
   ↓
Confidence Score
   │
   ├── High
   │     ↓
   │   Answer
   │
   └── Low
         ↓
      Large Model
```

假设：

```text
80% 请求 → Small Model
20% 请求 → Large Model
```

那么相比：

```text
100% → Large Model
```

成本可能显著降低。

这里最关键的问题是：

> 如何判断 Small Model 是否有能力解决？

可以使用：

```text
Confidence
Intent
Complexity
Token Length
Retrieval Quality
Previous Failure Rate
```

构建一个：

```text
Request Complexity Score
```

例如：

[
Score =
w_1 \times Complexity
+
w_2 \times ContextSize
+
w_3 \times ReasoningRequired
+
w_4 \times HistoricalFailureRate
]

然后根据 Score 做模型路由。

---

# 7. Context Engineering：最大的隐藏成本

很多团队优化 Token 时，只关注 Prompt。

实际上真正的问题通常是：

> **Context 太大。**

例如：

```text
System Prompt      3K
Conversation        8K
RAG Documents     20K
Tool Results       10K
User Request       2K
-----------------------
Total              43K
```

如果每次 Agent Loop 都把这 43K Token 重新发送给模型：

```text
Iteration 1 → 43K
Iteration 2 → 43K
Iteration 3 → 43K
Iteration 4 → 43K
```

那么：

```text
172K input tokens
```

可能只是完成一个用户请求。

---

# 8. Context Compression

Context Optimization 的核心不是：

> “把 Prompt 写得更短。”

而是：

> **让模型只看到当前任务真正需要的信息。**

可以建立：

```text
Raw Context
     ↓
Relevance Filter
     ↓
Deduplication
     ↓
Compression
     ↓
Summarization
     ↓
Relevant Context
     ↓
LLM
```

例如原始文档：

```text
100K tokens
```

通过：

```text
Retriever
+
Reranker
+
Compression
```

最终只保留：

```text
8K tokens
```

那么：

[
CostReduction \approx 92%
]

当然，实际效果还需要结合模型价格和缓存机制计算。

---

# 9. Context Window 不等于 Context Budget

这是 Agent 系统中非常重要的一个概念。

模型支持：

```text
1M Context
```

不代表你的 Agent 应该使用：

```text
1M Context
```

应该定义：

```text
Context Budget
```

例如：

```yaml
context:
  system: 2000
  conversation: 4000
  retrieved_docs: 6000
  tool_results: 4000
  user_query: 1000
```

总预算：

```text
17K tokens
```

而不是无限制地：

```text
append(context)
```

---

# 10. Semantic Cache：AI 系统中的关键成本杠杆

传统 Cache：

```text
GET /product/123
```

使用：

```text
Key = product:123
```

但是 LLM 请求通常不是完全相同的：

```text
"What is Kubernetes?"
"Can you explain Kubernetes?"
"Please explain Kubernetes to me."
```

语义不同，但答案高度相似。

因此可以使用：

> **Semantic Cache**

架构：

```text
User Query
    ↓
Embedding
    ↓
Vector Search
    ↓
Similarity
    │
    ├── similarity > threshold
    │          ↓
    │        Cache Hit
    │
    └── similarity < threshold
               ↓
              LLM
               ↓
             Cache
```

例如：

```text
threshold = 0.92
```

如果：

```text
similarity = 0.96
```

直接返回 Cache。

---

# 11. Semantic Cache 的成本模型

假设：

```text
100,000 requests/day
```

其中：

```text
Cache Hit = 40%
Cache Miss = 60%
```

那么真正调用 LLM 的请求：

```text
60,000
```

而不是：

```text
100,000
```

理论上：

[
LLMCallsReduced = 40%
]

如果 LLM 是整个系统最大的成本项，Semantic Cache 的收益可能非常明显。

但需要注意：

> Cache 命中率越高不一定越好。

因为错误的 Semantic Cache 会产生：

```text
Wrong Answer
Stale Answer
Context Mismatch
Authorization Leak
```

因此 Cache Key 必须考虑：

```text
tenant
user
permission
model
prompt version
knowledge version
language
query embedding
```

---

# 12. RAG Cost Optimization

RAG 是另一个非常容易产生隐性成本的地方。

典型 RAG：

```text
Query
 ↓
Embedding
 ↓
Vector Search
 ↓
Top-K
 ↓
Reranker
 ↓
LLM
```

问题在于：

```text
Top-K = 20
```

意味着可能把 20 个 Document 全部送给 LLM。

更合理：

```text
Vector Search
    ↓
Top 20
    ↓
Reranker
    ↓
Top 5
    ↓
Context Compression
    ↓
Top 3 relevant chunks
    ↓
LLM
```

这就是：

> **Retrieve More, Send Less**

---

# 13. RAG 中不要把 Retrieval 和 Generation 混在一起优化

RAG Cost 可以拆成：

[
C_{RAG}
=======

C_{Embedding}
+
C_{VectorSearch}
+
C_{Reranking}
+
C_{Generation}
]

通常：

```text
Generation Cost
>
Reranking Cost
>
Embedding Cost
>
Vector Search Cost
```

因此不要为了省一点 Vector Search CPU，而把大量无关文档传给 LLM。

真正应该优化的是：

```text
LLM Context Size
```

---

# 14. Agent Cost Optimization

Agent 是 AI Cost Optimization 最复杂的场景。

普通 Chat：

```text
User
 ↓
LLM
 ↓
Answer
```

Agent：

```text
User
 ↓
Planner
 ↓
Tool
 ↓
Observation
 ↓
Reasoning
 ↓
Tool
 ↓
Observation
 ↓
Reflection
 ↓
LLM
 ↓
Answer
```

如果 Agent 无限循环：

```text
while (!done) {
    callLLM();
    executeTool();
}
```

成本非常危险。

---

# 15. Agent Budget

生产环境中的 Agent 必须具有：

```text
Max Steps
Max Tokens
Max Tool Calls
Max Time
Max Cost
```

例如：

```yaml
agent:
  maxSteps: 8
  maxTokens: 30000
  maxToolCalls: 10
  timeout: 60s
  maxCost: 0.10
```

这样 Agent 不会因为错误规划进入：

```text
Tool
 ↓
LLM
 ↓
Tool
 ↓
LLM
 ↓
Tool
 ↓
LLM
...
```

无限循环。

---

# 16. Cost-aware Agent

进一步，可以让 Agent 自己感知成本。

例如：

```text
Agent State

remaining_budget = $0.05

current_cost = $0.03

remaining_budget = $0.02
```

此时 Agent 可以动态改变策略：

```text
High Budget
    ↓
Large Model
    ↓
Deep Reasoning
    ↓
Multiple Tools

Low Budget
    ↓
Small Model
    ↓
Short Context
    ↓
Limited Tools
```

这可以称为：

> **Cost-Aware Agent**

Agent 的 Objective 不再只是：

[
Maximize\ Quality
]

而是：

[
Maximize\ Quality - \lambda Cost
]

进一步可以形成：

[
Utility =
Quality
-------

## \lambda_1 Cost

## \lambda_2 Latency

\lambda_3 Risk
]

这实际上是 AI Agent 从“能工作”走向“可生产化”的重要一步。

---

# 17. Multi-Agent 系统尤其需要 Cost Governance

Multi-Agent 架构通常类似：

```text
                    Supervisor
                        │
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
      Researcher      Coder       Reviewer
          │             │             │
          ↓             ↓             ↓
         LLM           LLM           LLM
```

问题是：

```text
1 User Request
       ↓
Supervisor
       ↓
Researcher × 3
       ↓
Coder × 2
       ↓
Reviewer × 2
       ↓
Supervisor
```

最终可能产生：

```text
10+ LLM calls
```

因此 Multi-Agent 必须考虑：

```text
Agent Budget
Agent Priority
Agent Timeout
Agent Max Calls
Agent Model Tier
Agent Context Budget
```

---

# 18. Model Selection Matrix

可以为 Agent 定义模型等级：

```text
Tier 1
Cheap Model
       ↓
Classification
Extraction
Simple Tool Calling

Tier 2
Medium Model
       ↓
RAG
Planning
Coding

Tier 3
Premium Model
       ↓
Complex Reasoning
Architecture
Critical Decisions
```

Supervisor 根据任务动态分配：

```text
Task
 ↓
Complexity
 ↓
Model Tier
 ↓
Budget
```

而不是：

```text
所有 Agent
 ↓
Premium Model
```

---

# 19. Prompt Optimization

Prompt Optimization 不是单纯减少文字。

真正应该关注：

```text
Prompt Length
Instruction Duplication
Few-shot Examples
Output Length
Context Duplication
Tool Description
```

例如：

```text
System Prompt = 8K
Tool Definitions = 10K
```

每次调用都发送：

```text
18K tokens
```

这其实是非常大的固定成本。

因此可以考虑：

```text
Dynamic Tool Loading
```

不要给 Agent：

```text
100 tools
```

而是：

```text
User Request
    ↓
Tool Retrieval
    ↓
Relevant Tools
    ↓
LLM
```

例如：

```text
100 tools
 ↓
Tool Router
 ↓
5 tools
 ↓
LLM
```

这同时降低：

```text
Token Cost
Reasoning Cost
Tool Selection Complexity
Latency
```

---

# 20. Tool Description Optimization

很多 MCP / Agent 系统存在一个问题：

```text
Tool Definition
    ↓
Name
Description
Parameters
Examples
Constraints
```

如果系统拥有：

```text
200 tools
```

Tool schema 本身就可能消耗大量 Context。

因此可以采用：

```text
Tool Registry
      ↓
Semantic Search
      ↓
Top-K Tools
      ↓
LLM
```

也就是说：

> **Tools should be retrieved just like documents.**

这是 Agent 系统非常值得关注的优化方向。

---

# 21. Batch Processing

不是所有 AI 请求都要求实时。

例如：

```text
Document Classification
Data Extraction
Embedding
Summarization
Report Generation
```

可以从：

```text
Real-time
```

转为：

```text
Batch
```

架构：

```text
Application
    ↓
Kafka
    ↓
Batch Queue
    ↓
LLM Worker
    ↓
Result Storage
```

这样可以：

```text
提高吞吐
降低峰值资源
减少 API overhead
提高资源利用率
```

---

# 22. AI Infrastructure Cost

如果企业开始自建模型，那么成本模型又发生变化：

```text
GPU
CPU
Memory
Storage
Network
Kubernetes
Inference Server
Model Storage
Observability
```

此时核心指标变成：

[
CostPerToken =
\frac{TotalInfrastructureCost}
{GeneratedTokens}
]

或者：

[
CostPerRequest =
\frac{GPUCost + InfraCost}
{Requests}
]

---

# 23. GPU Utilization

GPU 成本优化不能简单理解为：

> “让 GPU 使用率达到 100%。”

真正重要的是：

```text
GPU Utilization
+
Tokens/sec
+
Latency
+
Batch Size
+
Memory Utilization
```

例如：

```text
GPU A
Utilization = 35%
```

可能意味着：

```text
Batch 太小
Request 太少
KV Cache 不合理
模型没有充分并发
```

可以通过：

```text
Dynamic Batching
Continuous Batching
Request Queue
Model Quantization
KV Cache Optimization
```

提升单位 GPU 的 Token 产出。

---

# 24. Quantization

如果模型允许，可以使用：

```text
FP16
↓
INT8
↓
INT4
```

降低：

```text
GPU Memory
Bandwidth
Inference Cost
```

但 Quantization 的核心不是：

> “精度越低越好。”

而是：

[
CostSaving

>

QualityLoss
]

因此应该通过 Evaluation Dataset 验证：

```text
Accuracy
Reasoning
Hallucination
Latency
Cost
```

---

# 25. AI Cost Observability

没有 Observability，就无法进行 Cost Optimization。

传统监控：

```text
CPU
Memory
QPS
Latency
Error Rate
```

AI 系统必须增加：

```text
Input Tokens
Output Tokens
Cached Tokens
Model
Provider
Prompt Version
Agent Steps
Tool Calls
Embedding Calls
RAG Documents
Cost
```

---

# 26. AI Cost Dashboard

建议建立这样的 Dashboard：

```text
                AI Cost Dashboard

Total Cost
$12,530

Cost / Request
$0.018

Total Tokens
2.3B

Cache Hit Rate
42%

Average Agent Steps
3.2

Model Distribution
Small      55%
Medium     35%
Large      10%
```

进一步按照：

```text
Tenant
Application
API
Agent
User
Model
Provider
Environment
```

进行成本分析。

---

# 27. Cost Attribution

企业级 AI 平台最重要的能力之一是：

> **Who is spending the money?**

例如：

```text
Company
 ├── Business A
 │     ├── Chatbot
 │     └── RAG
 │
 ├── Business B
 │     └── Coding Agent
 │
 └── Business C
       └── Customer Service
```

最终应该能够看到：

```text
Business A
$5,200

Business B
$12,800

Business C
$3,100
```

进一步：

```text
Application
    ↓
Agent
    ↓
Model
    ↓
Token
    ↓
Cost
```

这就是：

> **AI FinOps**

---

# 28. Cost Unit Economics

不要只关注：

```text
Monthly AI Bill
```

更重要的是：

```text
Cost per User
Cost per Request
Cost per Successful Task
Cost per Document
Cost per Conversation
Cost per Agent Task
```

尤其是：

[
CostPerSuccessfulTask
=====================

\frac{TotalCost}
{SuccessfulTasks}
]

这个指标非常重要。

假设：

```text
System A

Cost = $10,000
Success Rate = 50%

System B

Cost = $12,000
Success Rate = 95%
```

那么：

```text
Cost / Successful Task
```

B 可能反而更便宜。

所以：

> **Cost Optimization 不能脱离 Quality Optimization。**

---

# 29. Cost vs Quality Optimization

AI 系统实际上是在优化一个多目标函数：

[
Objective =
\alpha Quality
--------------

## \beta Cost

## \gamma Latency

\delta Risk
]

不同业务的权重不同。

例如：

### 客服

```text
Quality       50%
Cost          30%
Latency       20%
```

### Coding Agent

```text
Quality       70%
Cost          15%
Latency       15%
```

### Enterprise Batch Processing

```text
Quality       50%
Cost          40%
Latency       10%
```

因此不存在：

> “最低成本就是最佳方案。”

真正目标是：

> **Minimum Cost under Quality Constraint**

---

# 30. 一个完整的 AI Cost Optimization Architecture

最终可以形成如下架构：

```text
                         User
                           │
                           ↓
                    API Gateway
                           │
                           ↓
                    AI Cost Gateway
                           │
        ┌──────────────────┼──────────────────┐
        ↓                  ↓                  ↓
   Cache Layer       Model Router       Budget Manager
        │                  │                  │
        ↓                  ↓                  ↓
 Semantic Cache      Small Model        Cost Policy
        │             Medium Model       Rate Limit
        │             Large Model        Token Budget
        │                  │
        └──────────────────┼──────────────────┘
                           ↓
                         Agent
                           │
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
           RAG           Tools         Memory
             │             │             │
             ↓             ↓             ↓
         Retrieval      Tool Router   Context Manager
             │             │             │
             └─────────────┼─────────────┘
                           ↓
                      LLM Provider
                           │
                           ↓
                    Observability
                           │
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
          Metrics         Logs          Traces
             │             │             │
             └─────────────┼─────────────┘
                           ↓
                       AI FinOps
                           │
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
         Cost Report   Cost Alert    Optimization
```

---

# 31. AI Cost Gateway

在企业架构中，我非常推荐增加一个：

> **AI Cost Gateway**

它类似：

```text
API Gateway
```

但专门负责 AI 成本治理。

核心职责：

```text
1. Model Routing
2. Token Budget
3. Rate Limiting
4. Semantic Cache
5. Cost Tracking
6. Provider Routing
7. Model Fallback
8. Tenant Quota
9. Cost Alert
10. Policy Enforcement
```

调用链：

```text
Application
     ↓
AI Cost Gateway
     ↓
Policy Engine
     ↓
Model Router
     ↓
LLM Provider
```

---

# 32. Multi-Provider Cost Optimization

企业通常不会只使用一个模型供应商。

例如：

```text
Provider A
Provider B
Provider C
Self-hosted Model
```

可以建立：

```text
LLM Router
```

根据：

```text
Cost
Latency
Availability
Quality
Region
Data Policy
```

动态选择 Provider。

例如：

```text
                 Request
                    ↓
               LLM Router
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
   Provider A   Provider B   Self-hosted
      Cheap       Fast          Secure
```

这实际上类似传统 Cloud FinOps 中的：

> **Cloud Workload Placement**

---

# 33. Cost-aware Fallback

传统系统：

```text
Primary Model
      ↓
Failure
      ↓
Backup Model
```

AI 系统可以进一步：

```text
Primary Model
      ↓
Failure / Budget Exceeded
      ↓
Cheaper Model
```

例如：

```text
Premium Model
     ↓
Quota exceeded
     ↓
Medium Model
     ↓
Quota exceeded
     ↓
Small Model
```

这可以保证：

```text
系统可用性
+
成本可控
```

---

# 34. Token Budget 是 AI 系统的“资源配额”

传统系统：

```text
CPU Quota
Memory Quota
API Rate Limit
```

AI 系统应该增加：

```text
Token Quota
Cost Quota
Agent Step Quota
Tool Call Quota
```

例如：

```yaml
tenant:
  enterprise-a:
    dailyTokenLimit: 100000000
    dailyCostLimit: 500
    maxAgentSteps: 10

  enterprise-b:
    dailyTokenLimit: 20000000
    dailyCostLimit: 100
    maxAgentSteps: 5
```

这让 AI 平台真正具备：

> **Resource Governance**

---

# 35. AI Cost Optimization 的八层模型

可以把整个体系抽象成八层：

```text
Layer 8 ─ Business Optimization
             ↑
Layer 7 ─ AI FinOps
             ↑
Layer 6 ─ Observability
             ↑
Layer 5 ─ Agent Optimization
             ↑
Layer 4 ─ RAG / Context Optimization
             ↑
Layer 3 ─ Model Routing
             ↑
Layer 2 ─ Token Optimization
             ↑
Layer 1 ─ Infrastructure Optimization
```

很多团队只做：

```text
Layer 1
```

例如：

```text
GPU Optimization
Kubernetes Optimization
```

但真正的 AI 成本往往更多来自：

```text
Model
Token
Context
Agent
```

因此：

> **AI Cost Optimization 是 Application Architecture 问题，而不仅仅是 Infrastructure 问题。**

---

# 36. 一套完整的优化优先级

实际项目中，可以按照下面的优先级进行。

## Level 1：减少无意义调用

```text
Don't Call LLM
```

优先使用：

```text
Rule
Cache
Database
Deterministic Logic
```

---

## Level 2：Model Routing

```text
Easy → Small Model
Medium → Medium Model
Hard → Large Model
```

---

## Level 3：Context Optimization

重点优化：

```text
Prompt
History
RAG
Tool Description
Agent Memory
```

---

## Level 4：Caching

建立：

```text
Exact Cache
Semantic Cache
Embedding Cache
RAG Cache
Tool Result Cache
```

---

## Level 5：Agent Optimization

控制：

```text
Steps
Tokens
Tools
Iterations
Reflection
Planning
```

---

## Level 6：Infrastructure Optimization

最后再优化：

```text
GPU
CPU
Memory
Batching
Quantization
Kubernetes
```

---

# 37. 一个真实生产系统应该监控什么？

建议至少建立以下 Metrics：

```text
ai_request_total

ai_request_cost

ai_input_tokens

ai_output_tokens

ai_cached_tokens

ai_model_usage

ai_cache_hit_rate

ai_agent_steps

ai_tool_calls

ai_rag_documents

ai_rag_context_tokens

ai_latency

ai_success_rate

ai_cost_per_successful_task
```

进一步可以建立：

```text
cost_by_model
cost_by_tenant
cost_by_application
cost_by_agent
cost_by_api
cost_by_user
```

---

# 38. 最容易被忽略的 Cost Anti-Patterns

## Anti-Pattern 1：所有请求都使用大模型

```text
100%
 ↓
Premium Model
```

解决：

```text
Model Routing
```

---

## Anti-Pattern 2：整个 Conversation 全量发送

```text
History = 100K
 ↓
Every Request
```

解决：

```text
Summarization
Context Windowing
Memory
Compression
```

---

## Anti-Pattern 3：RAG Top-K 太大

```text
Top 50
 ↓
LLM
```

解决：

```text
Retrieve → Rerank → Compress → Generate
```

---

## Anti-Pattern 4：Agent 无限循环

```text
while(true)
```

解决：

```text
Max Steps
Max Cost
Max Tokens
Timeout
```

---

## Anti-Pattern 5：所有 Tool 都暴露给 Agent

```text
100+ Tools
 ↓
LLM
```

解决：

```text
Tool Retrieval
```

---

## Anti-Pattern 6：没有 Cost Observability

```text
Monthly Bill
    ↓
"为什么这么贵？"
```

这是最危险的状态。

---

# 39. 从“成本控制”走向“成本智能”

成熟的 AI 平台最终应该能够做到：

```text
Observe
   ↓
Analyze
   ↓
Predict
   ↓
Optimize
   ↓
Enforce
```

例如系统发现：

```text
Agent X

Average Cost = $0.18
Success Rate = 82%
```

而：

```text
Agent Y

Average Cost = $0.07
Success Rate = 89%
```

系统可以自动判断：

```text
Agent X
 ↓
High Cost
 ↓
Investigate
 ↓
Context too large
 ↓
Optimize
 ↓
Cost = $0.09
```

这就是：

> **AI-driven AI Cost Optimization**

也就是用 AI 本身优化 AI 系统的成本。

---

# 40. 未来：Cost Optimization 将成为 Agent Runtime 的核心能力

未来的 Agent Runtime 很可能不再只是：

```text
Planner
Executor
Memory
Tools
```

而是：

```text
                    Agent Runtime
                          │
       ┌──────────────────┼──────────────────┐
       ↓                  ↓                  ↓
    Planning           Execution          Memory
       │                  │                  │
       └──────────────────┼──────────────────┘
                          ↓
                    Cost Controller
                          │
       ┌──────────────────┼──────────────────┐
       ↓                  ↓                  ↓
   Model Router      Token Budget       Tool Budget
       │                  │                  │
       ↓                  ↓                  ↓
    Model Tier        Context Limit      Call Limit
```

Agent 的每一次行动都可以考虑：

```text
Expected Value
Expected Cost
Expected Latency
Risk
```

然后选择：

[
Action^*
========

argmax
(
ExpectedUtility
---------------

ExpectedCost
)
]

这意味着未来 Agent 不只是：

> **Reasoning Agent**

而会逐渐成为：

> **Resource-Aware Agent**

---

# 41. 总结

AI Cost Optimization 绝不是简单的：

```text
换便宜模型
```

真正完整的 AI 成本工程应该覆盖：

```text
                 AI Cost Optimization

                         ↓

             ┌─────────────────────┐
             │ Don't Call LLM      │
             └──────────┬──────────┘
                        ↓
             ┌─────────────────────┐
             │ Model Routing       │
             └──────────┬──────────┘
                        ↓
             ┌─────────────────────┐
             │ Token Optimization  │
             └──────────┬──────────┘
                        ↓
             ┌─────────────────────┐
             │ Context Engineering │
             └──────────┬──────────┘
                        ↓
             ┌─────────────────────┐
             │ RAG Optimization    │
             └──────────┬──────────┘
                        ↓
             ┌─────────────────────┐
             │ Agent Optimization  │
             └──────────┬──────────┘
                        ↓
             ┌─────────────────────┐
             │ Semantic Cache      │
             └──────────┬──────────┘
                        ↓
             ┌─────────────────────┐
             │ AI Observability    │
             └──────────┬──────────┘
                        ↓
             ┌─────────────────────┐
             │ AI FinOps           │
             └─────────────────────┘
```

最终目标不是：

> **Make AI Cheap**

而是：

> **Maximize AI Business Value per Dollar.**

也就是说，真正应该优化的指标是：

[
\boxed{
AI\ ROI =
\frac{Business\ Value}
{AI\ Cost}
}
]

对于企业级 AI 系统而言，最成熟的架构不是“最强模型 + 无限 Token + 无限 Agent Loop”，而是：

> **正确的问题使用正确的模型，在正确的 Context 下，以正确的 Agent 策略完成任务，并在整个生命周期内受到成本预算和质量指标约束。**

这也是 **AI Cost Optimization 从单点技术优化走向 AI Platform Engineering / AI FinOps 的核心演进方向。**

