---
title: Agent Observability：构建可观测、可诊断、可治理的 Agent Platform
# tags:
#   - nodejs
date: '2026-08-05'
summary: Agent Observability 不是简单地给 Agent 加上 Metrics、Logs、Traces，而是要对 Agent 的“决策、执行、协作和结果”进行完整观测
---
# Agent Observability 深度技术博客：构建可观测、可诊断、可治理的 Agent Platform

## 一、引言：为什么传统 Observability 已经不够用了？

在传统微服务系统中，我们通常认为：

```text
Observability
    |
    +-- Metrics
    +-- Logs
    +-- Traces
```

例如一个 HTTP 请求：

```text
User
  |
  v
API Gateway
  |
  v
Order Service
  |
  v
Payment Service
  |
  v
Database
```

我们可以通过：

* Metrics 看 QPS、Latency、Error Rate
* Logs 看错误信息
* Traces 看请求经过哪些服务

这套体系非常成熟。

但是进入 Agent 时代以后，一个请求可能变成：

```text
User
  |
  v
Agent
  |
  +---- LLM Call
  |
  +---- Tool Call
  |
  +---- Memory Retrieval
  |
  +---- LLM Call
  |
  +---- Agent B
           |
           +---- Tool Call
           |
           +---- LLM Call
  |
  +---- Final Response
```

这时候传统的：

```text
HTTP Trace
```

已经无法完整回答：

> **Agent 为什么做出了这个决定？**

更重要的是，Agent 的失败可能不是传统意义上的：

```text
HTTP 500
```

而可能是：

```text
Agent 选择了错误的 Tool
Agent 使用了错误的参数
Agent 调用了过多 Tool
Agent 陷入循环
Agent 获取了错误的 Memory
LLM 输出格式错误
RAG 检索结果不相关
Agent 花费过高
Agent 推理步骤过长
Agent 调用了没有必要的 Agent
```

因此：

> **Agent Observability 不是简单地给 Agent 加上 Metrics、Logs、Traces，而是要对 Agent 的“决策、执行、协作和结果”进行完整观测。**

---

# 二、Agent Observability 到底是什么？

可以给出一个比较工程化的定义：

> **Agent Observability 是通过 Trace、Metrics、Logs、Events、LLM Telemetry、Evaluation 等手段，对 Agent 的运行状态、决策过程、工具调用、模型调用、上下文、成本和最终效果进行可观测与诊断的技术体系。**

可以抽象成：

```text
                    Agent Observability
                            |
        +-------------------+-------------------+
        |                   |                   |
      Signals             Context             Quality
        |                   |                   |
  +-----+-----+       +-----+-----+       +-----+-----+
  |     |     |       |     |     |       |     |     |
Trace Metric Log    Prompt Memory Tool   Eval  Score Cost
```

这与传统 Observability 最大的区别是：

```text
Traditional Observability
        ↓
系统发生了什么？

Agent Observability
        ↓
系统发生了什么？
为什么发生？
Agent 为什么这样决策？
Agent 最终完成得怎么样？
```

---

# 三、Agent Observability 的核心目标

一个成熟的 Agent Observability 平台应该回答六个问题：

### 1. What？

```text
发生了什么？
```

例如：

```text
Agent 调用了 Search Tool
```

### 2. When？

```text
什么时候发生？
```

### 3. Where？

```text
发生在哪一个 Agent / Tool / Runtime？
```

### 4. Why？

```text
为什么调用这个 Tool？
为什么选择这个 Agent？
```

### 5. How？

```text
Agent 是怎么一步一步完成任务的？
```

### 6. How Good？

```text
最终结果到底好不好？
```

传统 Observability 主要解决：

```text
What
When
Where
```

Agent Observability 必须进一步解决：

```text
Why
How
How Good
```

---

# 四、Agent Observability 总体架构

一个企业级 Agent Observability 可以设计为：

```text
                         Agent Application
                                |
                                v
                        ┌───────────────┐
                        │ Agent Runtime │
                        └───────┬───────┘
                                |
                     OpenTelemetry SDK
                                |
              +-----------------+-----------------+
              |                 |                 |
              v                 v                 v
            Trace            Metrics            Logs
              |                 |                 |
              +-----------------+-----------------+
                                |
                                v
                    OpenTelemetry Collector
                                |
          +---------------------+---------------------+
          |                     |                     |
          v                     v                     v
        Tempo               Prometheus              Loki
          |                     |                     |
          +---------------------+---------------------+
                                |
                                v
                             Grafana
                                |
          +---------------------+---------------------+
          |                     |                     |
          v                     v                     v
      Trace View          Metrics Dashboard      Log Search
                                |
                                v
                     Agent Evaluation Platform
```

如果结合 Agent Platform：

```text
                  Agent Platform
                        |
       +----------------+----------------+
       |                                 |
   Control Plane                     Data Plane
                                         |
                                   Agent Runtime
                                         |
              +------------------+-------+-------+
              |                  |               |
             LLM               MCP              A2A
              |                  |               |
              +------------------+---------------+
                                         |
                                         v
                               OpenTelemetry
                                         |
                                         v
                              Observability Platform
```

---

# 五、传统 Observability 与 Agent Observability

| 传统微服务         | Agent                  |
| ------------- | ---------------------- |
| Request       | Task                   |
| Service       | Agent                  |
| RPC           | Agent-to-Agent         |
| API           | Tool                   |
| Database      | Memory / Knowledge     |
| Request Trace | Agent Trace            |
| HTTP Error    | Agent Failure          |
| Latency       | Execution Latency      |
| CPU / Memory  | Token / Cost / Context |
| Logs          | Agent Events           |
| SLA           | Task Completion        |
| Unit Test     | Agent Evaluation       |

这张表非常重要。

因为：

> **Agent Observability 本质上是在传统 Observability 上增加了一层 AI-aware Telemetry。**

---

# 六、Agent Trace：Observability 的核心

Agent 系统最重要的能力之一：

> **End-to-End Agent Trace**

例如：

```text
Trace: 8f7c...

User Task
│
├── Agent: ResearchAgent
│
├── LLM: Reasoning Model
│
├── Tool: WebSearch
│
├── Tool: WebSearch
│
├── Memory Retrieval
│
├── LLM: Reasoning Model
│
├── A2A: DataAgent
│   │
│   ├── LLM
│   ├── Tool: SQL
│   └── Tool: Database
│
├── LLM
│
└── Final Response
```

一个 Trace 不再只是：

```text
HTTP Request
```

而应该是：

> **Agent Execution Graph**

---

# 七、Agent Trace 的数据结构

一个 Agent Trace 可以包含：

```json
{
  "traceId": "trace-10001",
  "taskId": "task-20001",
  "agentId": "research-agent",
  "agentVersion": "3.2.1",
  "startTime": "...",
  "endTime": "...",
  "status": "COMPLETED",
  "steps": [
    {
      "type": "LLM",
      "model": "reasoning-model",
      "latencyMs": 1800,
      "inputTokens": 3200,
      "outputTokens": 850
    },
    {
      "type": "TOOL",
      "tool": "web-search",
      "latencyMs": 600
    }
  ]
}
```

这样我们可以进一步计算：

```text
Total Latency
LLM Latency
Tool Latency
Memory Latency
A2A Latency
Token Usage
Cost
```

---

# 八、Trace Span 应该怎么设计？

传统微服务：

```text
Span
 |
 +-- HTTP
 +-- SQL
 +-- RPC
```

Agent：

```text
Span
 |
 +-- AGENT
 +-- LLM
 +-- TOOL
 +-- MEMORY
 +-- RETRIEVAL
 +-- A2A
 +-- WORKFLOW
 +-- HUMAN
```

例如：

```text
Agent Span
│
├── LLM Span
│
├── Tool Span
│
├── Retrieval Span
│
├── LLM Span
│
├── A2A Span
│
└── Finalization Span
```

---

# 九、Agent Span 的关键 Attributes

一个 LLM Span 至少可以记录：

```text
agent.id
agent.version

llm.provider
llm.model

llm.input_tokens
llm.output_tokens
llm.total_tokens

llm.temperature

request.id
task.id
session.id

latency
status
```

Tool Span：

```text
tool.name
tool.version
tool.type
tool.latency
tool.status
tool.retry_count
```

A2A Span：

```text
source.agent
target.agent
task.id
communication.protocol
latency
status
```

---

# 十、为什么 OpenTelemetry 非常适合 Agent？

OpenTelemetry 的核心优势在于：

```text
Vendor Neutral
Standardized
Distributed Tracing
Metrics
Logs
Context Propagation
Collector
```

传统微服务：

```text
Service A
   |
   | trace context
   v
Service B
   |
   v
Service C
```

Agent：

```text
Agent A
   |
   | trace context
   v
LLM
   |
   v
Tool
   |
   v
Agent B
   |
   v
Tool
```

因此：

> **OpenTelemetry 可以成为 Agent Observability 的底层 Telemetry Standard。**

尤其适合你之前关注的：

```text
OpenTelemetry
Collector
Tempo
Prometheus
Grafana
```

这些技术可以直接演进到 Agent Observability。

---

# 十一、Agent Context Propagation

这是一个非常容易被忽略的问题。

例如：

```text
Agent A
   |
   | A2A
   v
Agent B
   |
   | MCP
   v
Tool
```

如果 Trace Context 丢失：

```text
Trace A
   |
   X
Agent B
   |
   X
Tool
```

最终 Grafana 中看到：

```text
三个孤立 Trace
```

而不是：

```text
一个完整 Trace
```

所以必须传播：

```text
traceId
spanId
trace flags
baggage
```

形成：

```text
Agent A
  |
  +---- Trace Context
             |
             v
          Agent B
             |
             +---- Trace Context
                        |
                        v
                       Tool
```

---

# 十二、Agent Metrics

Trace 解决：

> 单次请求发生了什么？

Metrics 解决：

> 系统整体怎么样？

Agent Metrics 可以分成六类。

---

## 1. Runtime Metrics

```text
agent_task_total
agent_task_success_total
agent_task_failure_total

agent_task_duration
agent_task_queue_depth
agent_task_active
```

例如：

```text
Task Throughput = 1000/min
Task Success Rate = 97.5%
P95 Latency = 8.2s
```

---

## 2. LLM Metrics

```text
llm_request_total
llm_request_duration
llm_error_total

llm_input_tokens
llm_output_tokens
llm_total_tokens
```

---

## 3. Tool Metrics

```text
tool_call_total
tool_call_failure_total
tool_call_duration
tool_retry_total
```

---

## 4. Memory Metrics

```text
memory_read_total
memory_write_total
memory_retrieval_latency
memory_hit_rate
```

---

## 5. A2A Metrics

```text
a2a_request_total
a2a_latency
a2a_failure_total
a2a_timeout_total
```

---

## 6. Cost Metrics

这是 Agent 特有的重要指标：

```text
token_cost
tool_cost
execution_cost
agent_cost
tenant_cost
```

例如：

```text
Daily Agent Cost

Research Agent       $32
Coding Agent         $85
Data Agent           $17
Customer Agent      $142
```

---

# 十三、不要只监控 Token

很多 Agent 系统会重点监控：

```text
Token Usage
```

但：

> Token 并不等于价值。

例如：

```text
Agent A
1000 tokens
完成任务

Agent B
20000 tokens
没有完成任务
```

单纯看 Token：

```text
Agent A < Agent B
```

但实际效果：

```text
Agent A > Agent B
```

因此需要建立：

```text
Cost
+
Quality
+
Latency
+
Success
```

四维指标体系。

---

# 十四、Agent Quality Metrics

这是 Agent Observability 与传统 Observability 最大的区别。

传统系统：

```text
HTTP 200
```

并不代表：

```text
业务正确
```

Agent 更明显。

例如：

```text
HTTP 200
Agent Response = 完整废话
```

系统依然是：

```text
200 OK
```

但用户体验：

```text
FAIL
```

因此需要：

> Quality Observability。

---

# 十五、Agent Evaluation

可以定义：

```text
Task Success Rate
```

例如：

```text
1000 tasks
 |
 +-- 950 success
 +-- 50 failure

Success Rate = 95%
```

还可以：

```text
Tool Selection Accuracy
Response Accuracy
Groundedness
Faithfulness
Relevance
Safety
```

---

# 十六、LLM-as-a-Judge

一种常见方式：

```text
Agent Output
      |
      v
Evaluator LLM
      |
      v
Score
```

例如：

```text
Accuracy       0.92
Relevance      0.95
Completeness   0.88
Safety         0.99
```

最终：

```text
Agent Quality Score = 0.93
```

但是需要注意：

> Evaluator LLM 也可能产生偏差。

因此生产环境最好组合：

```text
Rule-based
+
Deterministic Evaluation
+
LLM-as-a-Judge
+
Human Evaluation
```

---

# 十七、Agent Logs

传统 Log：

```text
INFO request received
ERROR database timeout
```

Agent Log 应该更加结构化：

```json
{
  "timestamp": "...",
  "traceId": "...",
  "taskId": "...",
  "agentId": "research-agent",
  "event": "tool_call",
  "tool": "web-search",
  "status": "SUCCESS",
  "latencyMs": 520
}
```

推荐：

> Structured Logging

而不是：

```text
System.out.println(...)
```

---

# 十八、Agent Event

Agent 的运行过程天然可以事件化：

```text
AgentStarted
AgentStepStarted
LLMRequestStarted
LLMRequestCompleted
ToolCallStarted
ToolCallCompleted
MemoryRead
MemoryWrite
A2ARequest
A2AResponse
HumanApprovalRequested
AgentCompleted
AgentFailed
```

这些 Event 可以进入：

```text
Kafka
```

然后被：

```text
Audit
Analytics
Billing
Monitoring
Evaluation
```

消费。

---

# 十九、Trace + Event 的区别

两者不要混淆。

### Trace

回答：

> 一次执行是怎样发生的？

```text
Task
 ↓
LLM
 ↓
Tool
 ↓
LLM
 ↓
Result
```

### Event

回答：

> 系统中发生了哪些事实？

```text
ToolCalled
ToolCompleted
AgentFailed
```

因此：

```text
Trace = Execution View

Event = System Event View
```

两者结合：

```text
Agent Runtime
      |
 +----+----+
 |         |
Trace     Event
 |         |
 v         v
Tempo     Kafka
```

是非常强的架构组合。

---

# 二十、Agent Observability 的三层模型

可以把整个体系分成：

```text
┌──────────────────────────────┐
│        Business Layer        │
│                              │
│ Success Rate                 │
│ Quality                      │
│ User Satisfaction            │
│ Business Outcome             │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│         AI Layer             │
│                              │
│ LLM                          │
│ Prompt                       │
│ Token                        │
│ Tool                         │
│ Memory                       │
│ Retrieval                    │
│ Agent Decision               │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│      Infrastructure Layer    │
│                              │
│ CPU                          │
│ Memory                       │
│ Network                      │
│ Kubernetes                   │
│ Database                     │
│ Kafka                        │
└──────────────────────────────┘
```

这就是：

> **Business + AI + Infrastructure 三层 Observability。**

---

# 二十一、Agent Debugging

假设用户说：

> “这个 Agent 为什么给出了错误答案？”

传统监控只能告诉你：

```text
HTTP 200
```

但是 Agent Observability 应该能够打开 Trace：

```text
Task
 |
 +-- Prompt
 |
 +-- Memory Retrieval
 |      |
 |      +-- Document A
 |      +-- Document B
 |
 +-- LLM
 |      |
 |      +-- Decision
 |
 +-- Tool Call
 |
 +-- LLM
 |
 +-- Final Response
```

然后发现：

```text
Memory Retrieval
      ↓
错误 Document
      ↓
LLM
      ↓
错误推理
      ↓
错误 Answer
```

这才是真正的：

> Agent Root Cause Analysis。

---

# 二十二、Agent Debugging 的核心问题

可以把问题定位拆成：

```text
Input Problem
      ↓
Context Problem
      ↓
Retrieval Problem
      ↓
Reasoning Problem
      ↓
Tool Problem
      ↓
Execution Problem
      ↓
Output Problem
```

例如：

### Input Problem

用户需求理解错误。

### Context Problem

上下文不完整。

### Retrieval Problem

RAG 找到了错误文档。

### Reasoning Problem

LLM 推理错误。

### Tool Problem

选择了错误 Tool。

### Execution Problem

Tool 参数错误。

### Output Problem

最终回答格式错误。

这比传统：

```text
HTTP 500
```

复杂很多。

---

# 二十三、Agent Loop Detection

Agent 可能出现：

```text
LLM
 ↓
Tool A
 ↓
LLM
 ↓
Tool A
 ↓
LLM
 ↓
Tool A
```

形成：

> Agent Loop。

如果没有监控，可能：

```text
Token ↑
Cost ↑
Latency ↑
```

最终：

```text
Timeout
```

因此 Runtime 应记录：

```text
step_count
tool_sequence
repeated_action
```

例如检测：

```text
A → B → A → B → A → B
```

发现异常后：

```text
STOP
```

或者：

```text
REPLAN
```

---

# 二十四、Agent Cost Observability

企业部署 Agent 后，一个非常现实的问题：

> “为什么这个月 AI 成本突然增加了 300%？”

传统 Monitoring 很难回答。

Agent Observability 可以：

```text
Tenant
  |
  +-- Agent
       |
       +-- Model
       |
       +-- Token
       |
       +-- Tool
       |
       +-- Execution
```

最终得到：

```text
Tenant A
  $1000

Research Agent
  $300

Coding Agent
  $500

Customer Agent
  $200
```

继续往下：

```text
Coding Agent
 |
 +-- Model Cost
 +-- Tool Cost
 +-- Retry Cost
 +-- Failed Task Cost
```

这就是：

> **AI FinOps。**

---

# 二十五、Agent SLO

传统微服务：

```text
Availability
Latency
Error Rate
```

Agent 可以进一步：

```text
Task Success Rate
Task Completion Time
Tool Success Rate
Quality Score
Cost Per Task
```

例如：

```text
Agent SLO

Task Success Rate      >= 98%
P95 Latency             < 10s
Tool Failure Rate       < 1%
Quality Score           >= 0.90
Cost Per Task           < $0.10
```

这才是企业真正需要的：

> Agent SLO。

---

# 二十六、Agent Error Budget

如果：

```text
SLO = 99%
```

那么：

```text
Error Budget = 1%
```

可以进一步定义：

```text
Failed Tasks
Poor Quality Tasks
Timeout Tasks
Safety Violations
```

而不是只统计：

```text
HTTP 500
```

例如：

```text
10000 Tasks

Allowed Failure = 100

Actual Failure = 80

Remaining Budget = 20
```

---

# 二十七、Agent Observability 与 Security

Observability 和 Security 有一个非常重要的冲突：

> **越详细的 Trace，包含的信息越多；信息越多，敏感数据泄露风险越大。**

例如 LLM Prompt 可能包含：

```text
Customer Name
Account Number
Internal Documents
Credentials
Personal Information
```

所以不能简单：

```text
log(prompt)
```

生产环境必须考虑：

```text
PII Masking
Data Redaction
Encryption
Access Control
Retention
Audit
```

例如：

```text
Original:
Customer SSN = 123-45-6789

Trace:
Customer SSN = ***-**-6789
```

---

# 二十八、Prompt Observability

Prompt 是 Agent 的核心输入。

因此需要记录：

```text
Prompt Version
System Prompt
User Prompt
Retrieved Context
Tool Result
```

但这里必须解决：

```text
Sensitive Data
Token Size
Storage Cost
Privacy
```

一个比较合理的设计：

```text
Prompt Metadata
       +
Prompt Hash
       +
Encrypted Content
```

而不是所有环境都直接保存完整 Prompt。

---

# 二十九、Agent Observability Data Pipeline

完整数据流：

```text
Agent Runtime
      |
      v
OpenTelemetry SDK
      |
      v
OTel Collector
      |
 +----+---------+---------+
 |              |         |
 v              v         v
Trace          Metric     Log
 |              |         |
Tempo       Prometheus    Loki
 |              |         |
 +--------------+---------+
                |
                v
              Grafana
                |
        +-------+-------+
        |               |
        v               v
   Operations       Analytics
        |
        v
   Evaluation
```

如果 Event 也加入：

```text
Agent Runtime
      |
      +---- Trace
      |
      +---- Metric
      |
      +---- Log
      |
      +---- Event
              |
              v
            Kafka
              |
       +------+------+
       |      |      |
     Audit  Billing  BI
```

---

# 三十、Agent Observability 存储架构

不同数据应该使用不同存储。

| 数据         | 推荐存储                        |
| ---------- | --------------------------- |
| Trace      | Tempo / Jaeger              |
| Metrics    | Prometheus                  |
| Logs       | Loki / Elasticsearch        |
| Events     | Kafka                       |
| Execution  | PostgreSQL                  |
| Memory     | Redis / Vector DB           |
| Analytics  | ClickHouse / Data Warehouse |
| Evaluation | PostgreSQL / Data Lake      |

不要：

```text
Everything → PostgreSQL
```

也不要：

```text
Everything → Elasticsearch
```

应该根据数据访问模式选择存储。

---

# 三十一、为什么 ClickHouse 很适合 Agent Analytics？

Agent 会产生大量：

```text
Execution
LLM Call
Tool Call
Token
Cost
Evaluation
```

这些数据天然适合：

```text
OLAP
```

例如查询：

```sql
SELECT
    agent_id,
    SUM(total_tokens),
    SUM(cost),
    AVG(latency)
FROM agent_execution
GROUP BY agent_id;
```

可以快速得到：

```text
Agent
Token
Cost
Latency
```

因此：

```text
Operational Observability
        ↓
Prometheus / Tempo / Loki

Analytical Observability
        ↓
ClickHouse / Data Warehouse
```

是一个很合理的架构。

---

# 三十二、Agent Dashboard 应该怎么看？

一个生产 Agent Dashboard 不应该只有：

```text
CPU
Memory
QPS
```

而应该至少包括：

### Overview

```text
Task Throughput
Task Success Rate
P95 Latency
Quality Score
Total Cost
```

### Agent

```text
Top Agents
Failure Rate
Average Steps
Average Cost
```

### LLM

```text
Token Usage
Model Latency
Model Error
Model Cost
```

### Tool

```text
Top Tools
Failure
Latency
Retry
```

### A2A

```text
Agent Calls
Agent Latency
Agent Failure
```

### Evaluation

```text
Quality
Accuracy
Safety
Regression
```

---

# 三十三、Grafana 中的 Agent Trace

最终可以形成：

```text
Dashboard
   |
   +-- Task Success = 97.8%
   |
   +-- P95 = 8.3s
   |
   +-- Cost = $1,235
   |
   +-- Quality = 0.93
   |
   +-- Failure = 2.2%
```

点击：

```text
Failure = 2.2%
```

进入：

```text
Agent Trace
```

继续点击：

```text
Tool Failure
```

进入：

```text
Tool Logs
```

再点击：

```text
Execution ID
```

查看：

```text
Prompt
Context
Model
Tool
Result
```

这才是真正的：

> **Observability → Investigation → Root Cause Analysis**

---

# 三十四、Agent Observability 的四个层次

可以把成熟度划分为：

## Level 1：Infrastructure Observability

```text
CPU
Memory
Network
Kubernetes
```

---

## Level 2：Service Observability

```text
HTTP
RPC
Database
Kafka
```

---

## Level 3：Agent Observability

```text
Agent
LLM
Tool
Memory
A2A
Token
Cost
```

---

## Level 4：AI Quality Observability

```text
Reasoning
Quality
Accuracy
Safety
Task Success
Business Outcome
```

企业真正的目标应该是：

```text
Level 1
   ↓
Level 2
   ↓
Level 3
   ↓
Level 4
```

---

# 三十五、Agent Observability 最难的地方

真正困难的并不是：

```text
OpenTelemetry SDK
Prometheus
Grafana
```

这些技术相对成熟。

真正困难的是：

### 1. 如何定义 Agent Telemetry Schema？

```text
Agent
Task
Step
LLM
Tool
Memory
A2A
```

如何标准化？

---

### 2. 如何观测“决策”？

传统 Trace：

```text
Service A → Service B
```

Agent：

```text
Agent → Why Tool A?
```

“Why”是非常困难的问题。

---

### 3. 如何定义 Agent Quality？

```text
HTTP 200
```

不代表：

```text
Task Success
```

---

### 4. 如何控制 Observability Cost？

如果每天：

```text
10M Tasks
```

每个 Task：

```text
20 Steps
```

可能产生：

```text
200M Spans
```

全部保存成本非常高。

因此需要：

```text
Sampling
Aggregation
Retention
Tiered Storage
```

---

# 三十六、Agent Trace Sampling

不能所有 Trace 都永久保存。

可以：

```text
Normal Trace
   ↓
1% Sampling

Error Trace
   ↓
100%

High Cost Trace
   ↓
100%

Quality Regression
   ↓
100%
```

进一步：

```text
Production
   |
   +-- Normal → 1%
   +-- Error → 100%
   +-- Slow → 100%
   +-- Expensive → 100%
   +-- Security → 100%
```

这就是：

> Intelligent Sampling。

---

# 三十七、从 Observability 走向 Agent Governance

当 Observability 数据积累以后，就可以进一步做：

```text
Observe
  ↓
Analyze
  ↓
Evaluate
  ↓
Govern
```

例如：

```text
发现某 Agent
平均 20 次 Tool Call
```

进一步：

```text
成本过高
```

然后：

```text
Policy
 ↓
限制 Tool Calls <= 10
```

再进一步：

```text
自动切换到更便宜的 Model
```

因此最终：

> **Observability 不只是“看”，而是 Agent Governance 的数据基础。**

---

# 三十八、Agent Observability 与 AgentOps

可以形成完整闭环：

```text
                  AgentOps
                     |
       +-------------+-------------+
       |             |             |
    Develop       Deploy         Run
       |             |             |
       +-------------+-------------+
                     |
                Observability
                     |
          +----------+----------+
          |                     |
       Monitor              Evaluate
          |                     |
          +----------+----------+
                     |
                  Improve
                     |
                     v
                  Release
```

所以：

> Agent Observability 是 AgentOps 的核心基础设施。

---

# 三十九、一个生产级 Agent Observability Stack

如果基于 Cloud Native 技术构建，可以考虑：

```text
                    Agent Runtime
                          |
                          v
                 OpenTelemetry SDK
                          |
                          v
                OpenTelemetry Collector
                          |
        +-----------------+----------------+
        |                 |                |
        v                 v                v
      Tempo          Prometheus           Loki
        |                 |                |
        +-----------------+----------------+
                          |
                          v
                       Grafana
                          |
                          v
                  Agent Observability
```

Event：

```text
Agent Runtime
      |
      v
    Kafka
      |
 +----+----+------+
 |         |      |
Audit    Billing  Analytics
```

Analytics：

```text
Kafka
  ↓
ClickHouse
  ↓
Agent Analytics
```

Evaluation：

```text
Execution
   ↓
Evaluation Engine
   ↓
Quality Score
   ↓
ClickHouse
```

---

# 四十、Agent Observability 的最终架构

综合起来：

```text
                           User
                            |
                            v
                    ┌───────────────┐
                    │ Agent Gateway │
                    └───────┬───────┘
                            |
                            v
                    ┌───────────────┐
                    │ Agent Runtime │
                    └───────┬───────┘
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
            LLM            MCP             A2A
             |              |              |
             +--------------+--------------+
                            |
                            v
                   OpenTelemetry SDK
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
           Trace          Metric           Log
             |              |              |
           Tempo       Prometheus         Loki
             |              |              |
             +--------------+--------------+
                            |
                            v
                         Grafana
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
          Runtime        AI Quality      Cost
          Metrics        Evaluation      Analysis
             |              |              |
             +--------------+--------------+
                            |
                            v
                       AgentOps
                            |
                            v
                       Governance
```

---

# 四十一、总结：Agent Observability 的本质

如果把传统 Observability 总结成：

```text
Metrics
+
Logs
+
Traces
```

那么 Agent Observability 应该升级为：

```text
                 Agent Observability
                         |
       +-----------------+-----------------+
       |                 |                 |
    Telemetry          Context          Quality
       |                 |                 |
   +---+---+        +----+----+       +----+----+
   |   |   |        |    |    |       |    |    |
Trace Metric Log   Prompt Memory Tool Eval Cost
```

进一步可以形成：

```text
Observe
   ↓
Understand
   ↓
Evaluate
   ↓
Govern
   ↓
Optimize
```

这就是 Agent Observability 与传统 Monitoring 最大的区别。

---

# 四十二、从架构师视角理解 Agent Observability

对于企业级 Agent Platform，我认为最值得建立的一个完整认知模型是：

```text
                         Agent Platform
                               |
                               v
                         Agent Runtime
                               |
              +----------------+----------------+
              |                |                |
             LLM              Tool             A2A
              |                |                |
              +----------------+----------------+
                               |
                               v
                        Agent Telemetry
                               |
       +-----------------------+-----------------------+
       |                       |                       |
      Trace                  Metric                  Log
       |                       |                       |
       +-----------------------+-----------------------+
                               |
                               v
                         Event / Analytics
                               |
             +-----------------+----------------+
             |                 |                |
          Runtime            Quality          Cost
             |                 |                |
             +-----------------+----------------+
                               |
                               v
                          AgentOps
                               |
                               v
                         Governance
```

因此，**Agent Observability 不是 Agent Platform 的附属功能，而应该被设计成 Agent Runtime 的一等公民。**

尤其是你前面已经关注过的 **OpenTelemetry + Prometheus + Grafana + Tempo + Kafka + Agent Runtime + A2A + MCP**，实际上完全可以串成一套完整的企业级技术体系：

```text
                    Agent Platform
                           |
                    ┌──────┴──────┐
                    |             |
              Control Plane   Data Plane
                                  |
                            Agent Runtime
                                  |
             +--------------------+-------------------+
             |                    |                   |
            LLM                  MCP                 A2A
             |                    |                   |
             +--------------------+-------------------+
                                  |
                         OpenTelemetry
                                  |
                +-----------------+-----------------+
                |                 |                 |
              Trace            Metric             Log
                |                 |                 |
              Tempo          Prometheus            Loki
                |                 |                 |
                +-----------------+-----------------+
                                  |
                               Grafana
                                  |
                           Agent Evaluation
                                  |
                           Agent Governance
```

这套架构已经不再是单纯的 **AI Application Development**，而是在进入：

> **AI Platform Engineering / Agent Infrastructure Engineering。**

