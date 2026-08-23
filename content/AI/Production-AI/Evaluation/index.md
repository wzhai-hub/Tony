---
title: Agent Evaluation 深度技术博客：从“LLM 能回答”到“Agent 真正可用”
# tags:
#   - nodejs
date: '2026-08-05'
summary: Agent Evaluation 是对 Agent 的输入理解、推理过程、工具选择、知识检索、协作行为、最终输出、成本、安全性和业务结果进行系统性度量与判断的工程体系。
---

# Agent Evaluation 深度技术博客：从“LLM 能回答”到“Agent 真正可用”

## 一、引言：为什么 Agent 必须有 Evaluation？

传统软件系统通常通过：

```text
Unit Test
Integration Test
End-to-End Test
Performance Test
Security Test
```

判断系统是否可靠。

例如：

```text
Input
  ↓
Function
  ↓
Expected Output
```

只要：

```text
Actual Output == Expected Output
```

测试就可以通过。

但是 Agent 完全不同。

一个 Agent 的执行过程可能是：

```text
User
 ↓
Agent
 ↓
LLM
 ↓
RAG
 ↓
Tool A
 ↓
LLM
 ↓
Tool B
 ↓
Agent B
 ↓
LLM
 ↓
Final Answer
```

即使同一个问题：

```text
Input = "帮我分析这家公司今年的财务情况"
```

Agent 每次可能：

* 使用不同的 Tool
* 检索不同的文档
* 调用不同的 Agent
* 产生不同的中间步骤
* 使用不同的模型
* 输出不同的答案

因此传统：

```text
Input → Expected Output
```

开始失效。

Agent Evaluation 要解决的问题变成：

> **Agent 是否完成了任务？为什么完成？哪里失败？答案质量如何？成本是否合理？行为是否安全？**

因此可以给出一个定义：

> **Agent Evaluation 是对 Agent 的输入理解、推理过程、工具选择、知识检索、协作行为、最终输出、成本、安全性和业务结果进行系统性度量与判断的工程体系。**

---

# 二、Agent Evaluation 与传统 Testing 的区别

这是理解 Evaluation 的第一关键点。

| 传统 Software Testing  | Agent Evaluation         |
| -------------------- | ------------------------ |
| Expected Output      | Expected Outcome         |
| Deterministic        | Probabilistic            |
| Function Correctness | Task Correctness         |
| Unit Test            | Component Evaluation     |
| Integration Test     | Workflow Evaluation      |
| E2E Test             | Agent Evaluation         |
| Assertion            | Scoring                  |
| Pass / Fail          | Score / Grade            |
| Bug                  | Behavioral Failure       |
| Regression Test      | Evaluation Regression    |
| Performance          | Quality + Latency + Cost |

传统测试关注：

```text
代码对不对？
```

Agent Evaluation 更关注：

```text
Agent 行为是否合理？
最终任务是否成功？
```

---

# 三、Agent Evaluation 的核心模型

一个完整的 Agent Evaluation 可以抽象为：

```text
                 Agent Evaluation
                        |
       +----------------+----------------+
       |                |                |
     Input            Process           Output
       |                |                |
   Task/Data       Reasoning        Answer
   Context         Tool Call        Result
                   Retrieval        Action
                   Planning
                        |
                        v
                     Quality
                        |
       +----------------+----------------+
       |                |                |
    Correctness      Relevance        Safety
       |                |                |
       +----------------+----------------+
                        |
                        v
                     Metrics
                        |
       +----------------+----------------+
       |                |                |
     Quality          Cost            Latency
```

最终评价的不只是：

```text
Answer
```

而是：

```text
Input
  +
Context
  +
Execution
  +
Output
  +
Outcome
```

---

# 四、Agent Evaluation 的六个维度

一个成熟的 Agent Evaluation Framework，至少应该包含六个维度：

```text
1. Correctness
2. Relevance
3. Faithfulness
4. Tool / Process Quality
5. Safety
6. Efficiency
```

下面逐一分析。

---

# 五、Correctness：答案正确吗？

这是最基本的 Evaluation。

例如：

```text
Question:
What is 15 * 8?

Agent:
120
```

可以直接判断：

```text
Correct = true
```

但是现实中的 Agent：

```text
Question:
What caused the production incident?
```

答案可能不是简单的：

```text
true / false
```

而是：

```text
0.0 ~ 1.0
```

例如：

```text
Correctness = 0.92
```

---

# 六、Exact Match

对于某些任务，可以使用：

```text
Exact Match
```

例如：

```text
Expected:
SUCCESS

Actual:
SUCCESS
```

结果：

```text
1
```

适用于：

* 分类
* 标签
* JSON
* SQL
* 数学
* 固定格式输出

但是不适用于：

```text
开放式问答
文章生成
复杂分析
Agent Task
```

---

# 七、Semantic Similarity

对于自然语言，可以比较：

```text
Expected Answer
        |
        v
Embedding
        |
        v
Vector
```

以及：

```text
Actual Answer
        |
        v
Embedding
        |
        v
Vector
```

计算：

```text
Cosine Similarity
```

例如：

```text
Expected:
Redis is an in-memory data store.

Actual:
Redis primarily stores data in memory.
```

语义高度接近：

```text
Similarity = 0.94
```

但是：

> Semantic Similarity ≠ Correctness。

两个错误答案可能也具有很高的语义相似度。

---

# 八、LLM-as-a-Judge

因此 Agent Evaluation 中非常重要的一种方式：

> **LLM-as-a-Judge**

架构：

```text
                Agent
                  |
                  v
             Agent Output
                  |
                  v
             Judge LLM
                  |
       +----------+----------+
       |          |          |
       v          v          v
   Correctness Relevance Safety
       |          |          |
       +----------+----------+
                  |
                  v
                Score
```

例如：

```json
{
  "correctness": 0.92,
  "relevance": 0.95,
  "completeness": 0.87,
  "safety": 1.0,
  "overall": 0.93
}
```

---

# 九、Judge Prompt 如何设计？

一个 Judge 不应该简单：

```text
Is this answer correct?
```

而应该定义明确的 Rubric。

例如：

```text
You are an expert evaluator.

Evaluate the agent response according to:

1. Correctness
   - Is the information factually correct?

2. Relevance
   - Does the answer directly address the question?

3. Completeness
   - Are important aspects missing?

4. Groundedness
   - Is the answer supported by the provided context?

Score each dimension from 0 to 1.

Return JSON only.
```

最终：

```json
{
  "correctness": 0.9,
  "relevance": 0.95,
  "completeness": 0.8,
  "groundedness": 0.93
}
```

---

# 十、为什么 LLM Judge 不能盲目信任？

因为：

> Judge 本身也是一个概率模型。

它可能出现：

```text
Judge Bias
Position Bias
Verbosity Bias
Self-preference Bias
Prompt Sensitivity
```

例如：

```text
Answer A:
非常详细，但核心错误。

Answer B:
简洁，但完全正确。
```

Judge 可能因为 Answer A 更详细而给更高分。

因此不能：

```text
Agent LLM
    ↓
Judge LLM
    ↓
100% Trust
```

更合理：

```text
LLM Judge
   +
Rules
   +
Reference
   +
Human Evaluation
   +
Statistical Analysis
```

---

# 十一、Reference-Based Evaluation

如果存在标准答案：

```text
Question
   |
   +---- Reference Answer
   |
   +---- Agent Answer
```

可以进行比较。

例如：

```text
Reference:
Redis uses an in-memory data structure
store and supports persistence.

Agent:
Redis is an in-memory data store
with persistence capabilities.
```

可以判断：

```text
Correctness
Completeness
Semantic Similarity
```

---

# 十二、Reference-Free Evaluation

现实中很多 Agent 没有标准答案。

例如：

```text
写一份技术方案
分析这个事故
设计一个系统架构
总结会议
制定旅行计划
```

这时候：

```text
Expected Answer
```

很难预先定义。

所以采用：

```text
Reference-Free Evaluation
```

评价：

```text
Relevance
Quality
Coherence
Safety
Completeness
```

这也是 LLM Judge 最常见的应用场景之一。

---

# 十三、RAG Evaluation

Agent Platform 中经常会出现：

```text
Agent
 ↓
Retriever
 ↓
Vector DB
 ↓
Documents
 ↓
LLM
```

那么答案错误，到底是谁的问题？

可能是：

```text
Retriever 错了
```

也可能：

```text
LLM 推理错了
```

所以 RAG Evaluation 至少需要拆成两个阶段。

---

# 十四、Retrieval Evaluation

核心指标：

### Precision

检索结果中：

```text
Relevant Documents
------------------
Retrieved Documents
```

### Recall

```text
Relevant Documents Retrieved
----------------------------
All Relevant Documents
```

例如：

```text
100 个相关文档
检索出 80 个
```

那么：

```text
Recall = 80%
```

---

# 十五、Top-K Evaluation

例如：

```text
Top K = 5
```

Retriever 返回：

```text
D1
D2
D3
D4
D5
```

其中：

```text
D1 Relevant
D2 Relevant
D3 Irrelevant
D4 Relevant
D5 Irrelevant
```

可以计算：

```text
Precision@5 = 3 / 5 = 60%
```

---

# 十六、MRR

如果我们关心：

> 第一个正确文档出现得有多快？

可以使用：

```text
Mean Reciprocal Rank
```

例如：

```text
Rank 1 → Relevant
```

则：

```text
RR = 1
```

如果：

```text
Rank 5 → Relevant
```

则：

```text
RR = 1/5
```

MRR 非常适合：

```text
Search
RAG
Knowledge Retrieval
```

---

# 十七、NDCG

如果搜索结果存在：

```text
Highly Relevant
Relevant
Partially Relevant
Irrelevant
```

可以使用：

```text
NDCG
```

它不仅考虑：

```text
Relevant / Irrelevant
```

还考虑：

```text
Ranking Position
```

因此适合：

```text
Enterprise Search
RAG
Recommendation
Knowledge Retrieval
```

---

# 十八、Faithfulness：Agent 有没有胡说？

这是 RAG/Agent Evaluation 中非常关键的指标。

例如 Context：

```text
Redis is an in-memory data store.
```

Agent：

```text
Redis was created by Google.
```

虽然回答：

```text
语言流畅
```

但：

```text
Faithfulness = 0
```

Faithfulness 关注：

> **回答是否能够被提供的上下文支持。**

因此：

```text
Context
   ↓
Claims
   ↓
Evidence
```

需要判断：

```text
Claim
  |
  +-- Supported → TRUE
  |
  +-- Unsupported → FALSE
```

---

# 十九、Hallucination Evaluation

可以进一步定义：

```text
Hallucination Rate
```

例如：

```text
100 个回答
12 个存在无法验证的事实
```

则：

```text
Hallucination Rate = 12%
```

生产环境可以设置：

```text
Hallucination Rate < 1%
```

对于：

```text
金融
医疗
法律
企业知识库
```

尤其重要。

---

# 二十、Tool Evaluation

Agent 最大的特点之一：

> 会调用 Tool。

因此必须评估：

```text
是否选择正确 Tool？
参数是否正确？
调用顺序是否正确？
是否需要调用 Tool？
是否调用太多次？
```

---

# 二十一、Tool Selection Accuracy

例如系统提供：

```text
SearchTool
SQLTool
CalculatorTool
EmailTool
```

用户：

```text
查询数据库中的订单
```

正确：

```text
SQLTool
```

Agent：

```text
SearchTool
```

则：

```text
Tool Selection = FAIL
```

可以定义：

```text
Tool Selection Accuracy
=
Correct Tool Calls
/
Total Tool Calls
```

---

# 二十二、Tool Argument Evaluation

假设：

```json
{
  "tool": "get_order",
  "order_id": "12345"
}
```

Agent 生成：

```json
{
  "tool": "get_order",
  "order_id": "12354"
}
```

Tool 本身正常。

但是 Agent 参数错误。

所以必须把：

```text
Tool Selection
```

和：

```text
Tool Argument
```

分开评估。

---

# 二十三、Tool Call Efficiency

Agent 可能：

```text
Search
Search
Search
Search
Search
```

最终才找到答案。

虽然：

```text
Task Success = TRUE
```

但：

```text
Cost ↑
Latency ↑
```

因此需要：

```text
Average Tool Calls
Tool Calls / Successful Task
Redundant Tool Calls
Retry Rate
```

例如：

```text
Agent A
Success = 95%
Tool Calls = 3.2

Agent B
Success = 96%
Tool Calls = 15.7
```

从生产角度：

```text
Agent A
```

可能明显更优秀。

---

# 二十四、Agent Planning Evaluation

复杂 Agent 通常：

```text
Goal
 ↓
Plan
 ↓
Step 1
 ↓
Step 2
 ↓
Step 3
 ↓
Result
```

因此可以评价：

```text
Plan Correctness
Plan Completeness
Step Efficiency
Plan Stability
```

例如目标：

```text
完成一次数据分析
```

Agent：

```text
1. 获取数据
2. 清洗数据
3. 分析数据
4. 生成报告
```

这是合理计划。

如果：

```text
1. 生成报告
2. 获取数据
3. 删除数据
```

就是明显异常。

---

# 二十五、Agent Trajectory Evaluation

Agent Evaluation 不应该只看：

```text
Final Answer
```

还应该看：

```text
Trajectory
```

即：

> Agent 是怎么走到最终结果的。

例如：

```text
Task
 |
 +-- Plan
 |
 +-- Tool A
 |
 +-- Tool B
 |
 +-- Tool C
 |
 +-- Replan
 |
 +-- Tool D
 |
 +-- Final
```

可以定义：

```text
Trajectory Score
```

评价：

```text
Correctness
Efficiency
Tool Usage
Planning
```

---

# 二十六、Agent Outcome Evaluation

最终最重要的问题：

> 任务完成了吗？

例如：

```text
用户：
帮我创建一个会议并邀请团队成员。
```

Agent：

```text
调用 Calendar Tool
调用 Email Tool
```

最终：

```text
Calendar Created = TRUE
Invitation Sent = TRUE
```

那么：

```text
Task Success = TRUE
```

这比单纯：

```text
LLM Output Quality = 0.95
```

更重要。

---

# 二十七、Business Outcome Evaluation

企业最终关心的不是：

```text
LLM Accuracy = 93%
```

而是：

```text
业务结果怎么样？
```

例如客服 Agent：

```text
Answer Quality
       ↓
Customer Resolution
       ↓
Customer Satisfaction
       ↓
Cost Reduction
```

Coding Agent：

```text
Code Quality
      ↓
Tests Passed
      ↓
PR Accepted
      ↓
Production Defects
```

所以：

> **Business Outcome 是 Agent Evaluation 的最高层。**

---

# 二十八、Evaluation Pyramid

可以建立一个非常重要的 Evaluation Pyramid：

```text
                     Business Outcome
                           ▲
                           |
                    Task Success
                           |
                    Agent Behavior
                           |
                 Tool / RAG / Planning
                           |
                      LLM Quality
                           |
                Infrastructure Metrics
```

越往上：

```text
越接近业务价值
```

越往下：

```text
越接近系统实现
```

因此不能只做：

```text
LLM Evaluation
```

而应该做：

```text
End-to-End Agent Evaluation
```

---

# 二十九、Online Evaluation 与 Offline Evaluation

这是 Agent Evaluation 架构中的另一个核心概念。

## Offline Evaluation

在上线前：

```text
Dataset
 ↓
Agent
 ↓
Evaluation
 ↓
Score
```

例如：

```text
1000 Test Cases
```

用于：

```text
Regression
Model Selection
Prompt Optimization
Agent Version Comparison
```

---

# 三十、Online Evaluation

上线以后：

```text
Real User
   ↓
Agent
   ↓
Production Trace
   ↓
Evaluation
```

例如：

```text
Production Tasks
       ↓
Sampling
       ↓
Judge
       ↓
Quality Score
```

用于：

```text
Drift Detection
Quality Monitoring
Production Regression
```

---

# 三十一、Offline + Online

完整体系：

```text
                 Evaluation
                      |
          +-----------+-----------+
          |                       |
       Offline                  Online
          |                       |
      Dataset                  Production
          |                       |
          v                       v
      Regression              Monitoring
          |                       |
          +-----------+-----------+
                      |
                      v
                 Improvement
```

这就形成：

> **Evaluation Loop**

---

# 三十二、Evaluation Dataset

没有 Dataset：

```text
Evaluation
```

很难工程化。

一个 Dataset 可以：

```json
{
  "id": "case-001",
  "input": "What is Redis?",
  "context": [
    "Redis is an in-memory data store."
  ],
  "expected": "Redis is an in-memory data store."
}
```

进一步：

```json
{
  "id": "case-002",
  "input": "...",
  "tools": ["search", "sql"],
  "expected_tool": "sql"
}
```

---

# 三十三、Dataset 不应该只有 Happy Path

很多团队的 Evaluation Dataset：

```text
90% Normal
10% Edge Case
```

这是不够的。

应该覆盖：

```text
Normal Cases
Edge Cases
Ambiguous Cases
Adversarial Cases
Failure Cases
Security Cases
Long Context
Multi-turn
Tool Failure
Network Failure
LLM Failure
```

例如：

```text
Tool Timeout
     ↓
Agent 是否 Retry？

Tool Returns Invalid JSON
     ↓
Agent 是否 Recovery？

Knowledge Not Found
     ↓
Agent 是否 Hallucinate？
```

这些才是真正有价值的 Evaluation Case。

---

# 三十四、Evaluation Dataset 分层

建议：

```text
Dataset
 |
 +-- Smoke Set
 |
 +-- Regression Set
 |
 +-- Golden Set
 |
 +-- Adversarial Set
 |
 +-- Production Sample
```

### Smoke Set

几十个核心 Case。

用于：

```text
每次部署
```

### Regression Set

几百/几千 Case。

用于：

```text
版本升级
```

### Golden Set

人工高质量标注。

用于：

```text
核心质量评估
```

### Adversarial Set

专门测试：

```text
Prompt Injection
Tool Abuse
Hallucination
Boundary
```

### Production Sample

真实生产数据采样。

用于：

```text
线上质量
```

---

# 三十五、Evaluation Regression

Agent 系统最容易发生一个问题：

```text
Prompt V1
  ↓
Quality = 92%
```

升级：

```text
Prompt V2
  ↓
Quality = 95%
```

看起来更好了。

但是：

```text
Tool Selection
92% → 85%
```

这就是：

> Hidden Regression。

所以 Evaluation 必须支持：

```text
Version Comparison
```

例如：

```text
                V1       V2
Correctness     92%      95%
Relevance       90%      93%
Tool Accuracy   94%      85%
Cost            $0.08    $0.12
Latency         5.2s     6.8s
```

最终：

```text
Overall Improvement?
```

不能只看一个指标。

---

# 三十六、Multi-Dimensional Evaluation

建议建立 Scorecard：

```text
Agent Scorecard

Correctness       0.94
Relevance         0.92
Faithfulness      0.96
Tool Accuracy     0.91
Safety            0.99
Efficiency        0.88
Cost              0.82
```

最终：

```text
Overall Score
```

可以通过加权：

```text
Score =
0.30 * Correctness
+ 0.20 * Relevance
+ 0.20 * Faithfulness
+ 0.10 * Tool Accuracy
+ 0.10 * Safety
+ 0.10 * Efficiency
```

但是：

> 不建议盲目使用一个 Overall Score。

因为：

```text
Safety = 0.5
```

不能被：

```text
Correctness = 1.0
```

平均掉。

对于某些指标应该使用：

```text
Hard Constraint
```

例如：

```text
Safety < 0.99
    ↓
FAIL
```

---

# 三十七、Evaluation Gateway

在企业 Agent Platform 中，可以建立：

```text
                    Agent
                      |
                      v
              Evaluation Gateway
                      |
          +-----------+-----------+
          |           |           |
       Quality      Safety      Cost
          |           |           |
          +-----------+-----------+
                      |
                 Pass / Reject
```

例如：

```text
Agent Version 2.3
       ↓
Regression Test
       ↓
Quality = 94%
Safety = 99.9%
Cost = $0.07
       ↓
PASS
       ↓
Production
```

如果：

```text
Quality = 82%
```

则：

```text
BLOCK RELEASE
```

这就是：

> Evaluation-driven CI/CD。

---

# 三十八、Agent Evaluation CI/CD

可以设计成：

```text
Git Push
   ↓
Build
   ↓
Unit Test
   ↓
Agent Evaluation
   ↓
Regression
   ↓
Security Evaluation
   ↓
Cost Evaluation
   ↓
Quality Gate
   ↓
Deploy
```

例如：

```text
Quality >= 0.90
Safety >= 0.99
Task Success >= 0.95
Cost <= $0.10
```

全部满足：

```text
Deploy
```

否则：

```text
Reject
```

---

# 三十九、Evaluation 与 Observability 的关系

这是非常关键的一点。

Observability：

```text
发生了什么？
```

Evaluation：

```text
做得好不好？
```

例如：

```text
Trace
 |
 +-- LLM
 +-- Tool
 +-- RAG
 +-- A2A
```

Observability 告诉你：

```text
Agent 调用了 Tool A。
```

Evaluation 告诉你：

```text
Agent 本来应该调用 Tool B。
```

因此：

```text
Observability
       ↓
Execution Data
       ↓
Evaluation
       ↓
Quality
```

两者必须结合。

---

# 四十、Evaluation + Observability 架构

完整架构可以设计成：

```text
                         Agent Runtime
                               |
                     OpenTelemetry
                               |
          +--------------------+-------------------+
          |                    |                   |
        Trace                Metrics              Logs
          |                    |                   |
        Tempo              Prometheus             Loki
          |                    |                   |
          +--------------------+-------------------+
                               |
                               v
                        Evaluation Engine
                               |
          +--------------------+-------------------+
          |                    |                   |
     Rule Evaluator       LLM Judge         Custom Evaluator
          |                    |                   |
          +--------------------+-------------------+
                               |
                               v
                         Quality Metrics
                               |
          +--------------------+-------------------+
          |                    |                   |
       Dashboard          Regression          Governance
```

这已经非常接近企业级：

> **Agent Quality Platform。**

---

# 四十一、Evaluation Engine 的核心设计

可以抽象为：

```text
Evaluation Engine
       |
       +-- Dataset Manager
       |
       +-- Runner
       |
       +-- Evaluator
       |
       +-- Scorer
       |
       +-- Reporter
       |
       +-- Regression Detector
```

---

# 四十二、Evaluation Runner

Runner 负责：

```text
Load Dataset
     ↓
Invoke Agent
     ↓
Capture Trace
     ↓
Store Result
     ↓
Run Evaluators
```

例如：

```text
for case in dataset:

    result = agent.execute(case.input)

    trace = observability.getTrace(result)

    scores = evaluator.evaluate(
        case,
        result,
        trace
    )

    store(scores)
```

---

# 四十三、Evaluator 接口设计

一个比较好的架构可以抽象成：

```java
public interface Evaluator {

    EvaluationResult evaluate(
        EvaluationCase testCase,
        AgentResult result,
        AgentTrace trace
    );
}
```

不同 Evaluator：

```text
CorrectnessEvaluator
RelevanceEvaluator
FaithfulnessEvaluator
ToolEvaluator
SafetyEvaluator
CostEvaluator
LatencyEvaluator
```

这样：

```text
Evaluator
```

就成为：

> 可插拔的 Evaluation Plugin。

---

# 四十四、Evaluation Result

例如：

```json
{
  "caseId": "case-001",
  "agentVersion": "2.1.0",
  "scores": {
    "correctness": 0.95,
    "relevance": 0.92,
    "faithfulness": 0.97,
    "tool_accuracy": 1.0,
    "safety": 1.0
  },
  "latency_ms": 4300,
  "tokens": 3250,
  "cost": 0.043,
  "passed": true
}
```

这样 Evaluation 数据本身就可以进入：

```text
ClickHouse
```

进行分析。

---

# 四十五、Evaluation 的统计学问题

Agent Evaluation 不能只跑：

```text
10 cases
```

然后宣布：

```text
Accuracy = 90%
```

因为：

```text
样本太小
```

可能导致：

```text
统计偏差
```

例如：

```text
10 Cases
9 Success
```

看起来：

```text
90%
```

但生产：

```text
10000 Cases
```

可能完全不同。

因此需要：

```text
Large Dataset
Confidence Interval
Statistical Significance
```

尤其是：

```text
A/B Testing
Model Comparison
Prompt Comparison
```

---

# 四十六、Agent A/B Evaluation

例如：

```text
Prompt A
VS
Prompt B
```

随机：

```text
50% → A
50% → B
```

然后比较：

```text
Task Success
Quality
Cost
Latency
User Satisfaction
```

例如：

```text
                A       B

Success        92%     95%
Quality        89%     94%
Cost           $0.10   $0.08
Latency        5.8s    4.9s
```

显然：

```text
B
```

更有优势。

---

# 四十七、Human Evaluation

在高价值场景：

```text
LLM Judge
```

不能完全替代：

```text
Human
```

可以采用：

```text
Human
   +
LLM Judge
   +
Rule
```

例如：

```text
10000 Production Cases
        |
        v
     Sampling
        |
        v
    500 Cases
        |
   +----+----+
   |         |
   v         v
LLM Judge   Human
   |         |
   +----+----+
        |
        v
Calibration
```

通过 Human Evaluation 校准 Judge。

---

# 四十八、Judge Calibration

例如：

```text
Human Score
      vs
LLM Judge Score
```

如果：

```text
Correlation = 0.92
```

说明 Judge 比较可靠。

如果：

```text
Correlation = 0.60
```

说明：

```text
Judge Prompt
```

需要重新设计。

---

# 四十九、Evaluation 的数据闭环

成熟平台最终应该形成：

```text
Production
    ↓
Trace
    ↓
Sample
    ↓
Evaluation
    ↓
Bad Cases
    ↓
Dataset
    ↓
Regression Test
    ↓
Prompt / Model / Agent Improvement
    ↓
Production
```

这是一条非常重要的：

> **Agent Quality Flywheel。**

---

# 五十、Failure Case Mining

生产系统里最有价值的数据通常不是：

```text
Successful Cases
```

而是：

```text
Failed Cases
```

可以自动挖掘：

```text
Low Score
Tool Failure
Hallucination
High Cost
Long Latency
User Negative Feedback
```

形成：

```text
Failure Dataset
```

然后：

```text
Failure Dataset
       ↓
Regression Test
```

最终让：

> 线上事故成为下一版本的测试用例。

---

# 五十一、Evaluation 与 Agent Runtime 的结合

Agent Runtime 不应该只负责：

```text
Execute
```

还应该输出：

```text
Execution Context
```

例如：

```text
AgentRuntime
 |
 +-- Task
 +-- Step
 +-- LLM
 +-- Tool
 +-- Memory
 +-- A2A
 +-- Result
 +-- Trace
```

Evaluation Engine 读取：

```text
Agent Execution Graph
```

然后分析：

```text
Outcome
Trajectory
Tool
Reasoning
Cost
```

所以：

> **Agent Runtime 是 Evaluation 的数据源。**

---

# 五十二、Evaluation 与 Agent Platform 的关系

企业级 Agent Platform 可以设计成：

```text
                    Agent Platform
                          |
        +-----------------+-----------------+
        |                 |                 |
   Agent Runtime      Evaluation        Observability
        |                 |                 |
        |                 |                 |
        +-----------------+-----------------+
                          |
                       AgentOps
                          |
        +-----------------+-----------------+
        |                 |                 |
     Release          Governance          Analytics
```

其中：

```text
Observability
```

负责：

```text
What happened?
```

```text
Evaluation
```

负责：

```text
How good?
```

```text
AgentOps
```

负责：

```text
What should we do?
```

---

# 五十三、Agent Evaluation 的成熟度模型

可以把企业 Evaluation 分成五级。

## Level 0：No Evaluation

```text
Agent
 ↓
Production
```

完全依赖用户反馈。

---

## Level 1：Manual Evaluation

```text
Agent
 ↓
Human Review
```

适合 PoC。

---

## Level 2：Automated Evaluation

```text
Agent
 ↓
Automated Dataset
 ↓
Score
```

开始工程化。

---

## Level 3：Continuous Evaluation

```text
Production
 ↓
Sampling
 ↓
Evaluation
 ↓
Regression
```

进入生产质量体系。

---

## Level 4：Evaluation-Driven Agent Platform

```text
Develop
 ↓
Evaluate
 ↓
Deploy
 ↓
Observe
 ↓
Evaluate
 ↓
Govern
 ↓
Improve
```

这是企业级 Agent Platform 的目标。

---

# 五十四、Evaluation 最终应该解决什么？

如果让我把整个 Agent Evaluation 总结成一张图：

```text
                         Agent
                           |
                           v
                    Agent Execution
                           |
          +----------------+----------------+
          |                |                |
         LLM              Tool             RAG
          |                |                |
          +----------------+----------------+
                           |
                           v
                        Trace
                           |
                           v
                    Evaluation Engine
                           |
       +-------------------+-------------------+
       |                   |                   |
     Outcome            Behavior            Quality
       |                   |                   |
   Task Success       Tool Accuracy       Correctness
   Business Result    Planning            Relevance
                      Efficiency           Faithfulness
                      Safety               Groundedness
                           |
                           v
                         Score
                           |
              +------------+------------+
              |                         |
         Regression                  Governance
              |                         |
              v                         v
          Improve Agent             Control Agent
```

---

# 五十五、结语：Evaluation 是 Agent Engineering 的“质量操作系统”

传统 AI Application 的核心问题是：

> **如何让 LLM 回答得更好？**

Agent Engineering 的核心问题正在变成：

> **如何证明 Agent 做得足够好？**

这是两个完全不同的问题。

一个成熟 Agent 系统最终需要建立：

```text
             Agent Quality
                  |
      +-----------+-----------+
      |           |           |
   Correctness  Safety      Outcome
      |           |           |
      +-----------+-----------+
                  |
          +-------+-------+
          |       |       |
       Quality  Cost   Latency
          |       |       |
          +-------+-------+
                  |
             Evaluation
                  |
       +----------+----------+
       |                     |
   Offline                  Online
       |                     |
   Regression             Monitoring
       |                     |
       +----------+----------+
                  |
             AgentOps
                  |
             Governance
```

因此，**Evaluation 并不是 Agent Platform 中一个独立的测试模块，而应该成为贯穿 Agent 生命周期的质量基础设施。**

从架构师视角来看，未来的 Agent Platform 至少应该形成这样一条完整技术链：

```text
Agent Development
       ↓
Agent Runtime
       ↓
OpenTelemetry
       ↓
Trace / Event
       ↓
Evaluation Engine
       ↓
Quality Metrics
       ↓
Regression Testing
       ↓
CI/CD Quality Gate
       ↓
Production
       ↓
Online Evaluation
       ↓
Failure Mining
       ↓
Dataset
       ↓
Agent Improvement
```

最终形成：

> **Observe → Evaluate → Govern → Improve**

这也是 Agent Engineering 从“Prompt Engineering”走向“Platform Engineering”的一个重要标志。
