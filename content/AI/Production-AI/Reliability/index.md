---
title: Agent Reliability：如何构建高可靠的 Agent 系统
# tags:
#   - nodejs
date: '2026-08-05'
summary: Agent Reliability 是 Agent 在面对正常请求、异常输入、依赖故障、模型不确定性以及复杂执行路径时，持续完成目标并保持安全、可控和可恢复行为的能力。
---

# Agent Reliability 深度技术博客：如何构建高可靠的 Agent 系统

## 一、引言：Agent 最大的问题不是“聪明”，而是“可靠”

传统软件系统追求：

> **Deterministic Correctness**

即：

```text
Input
  ↓
Business Logic
  ↓
Expected Result
```

只要代码正确，系统通常能够稳定地产生相同结果。

Agent 则完全不同：

```text
User
 ↓
Agent
 ↓
LLM
 ↓
Planning
 ↓
Tool
 ↓
RAG
 ↓
Another Agent
 ↓
LLM
 ↓
Action
 ↓
Result
```

其中任何一个环节都可能失败。

例如：

```text
LLM 输出错误
Tool 调用失败
Tool 参数错误
RAG 检索错误
Context 丢失
Agent 无限循环
Agent 选择错误的 Tool
外部 API Timeout
网络抖动
模型服务限流
Token 超限
上下文污染
Prompt Injection
```

所以 Agent Engineering 中一个非常核心的问题是：

> **如何让一个具有概率性行为的系统，表现得像一个可靠的软件系统？**

这就是：

# Agent Reliability

---

# 二、什么是 Agent Reliability？

可以定义为：

> **Agent Reliability 是 Agent 在面对正常请求、异常输入、依赖故障、模型不确定性以及复杂执行路径时，持续完成目标并保持安全、可控和可恢复行为的能力。**

它不仅仅是：

```text
Availability
```

也不是：

```text
LLM Accuracy
```

而是：

```text
Reliability
=
Availability
+
Correctness
+
Resilience
+
Recoverability
+
Safety
+
Controllability
```

因此：

```text
Agent Reliability
        |
        +-- Availability
        |
        +-- Correctness
        |
        +-- Resilience
        |
        +-- Recovery
        |
        +-- Safety
        |
        +-- Control
```

---

# 三、为什么 Agent Reliability 比传统微服务更难？

传统微服务：

```text
Service A
   ↓
Service B
   ↓
Database
```

通常可以明确：

```text
Timeout
Retry
Circuit Breaker
Fallback
```

Agent：

```text
Agent
  ↓
LLM
  ↓
Tool
  ↓
LLM
  ↓
Search
  ↓
Agent B
  ↓
Tool
```

这里增加了一个非常特殊的问题：

> **Agent 自己决定下一步做什么。**

例如：

```text
if API failed:
    retry
```

传统程序由开发者决定。

Agent 则可能：

```text
Tool failed
   ↓
LLM
   ↓
Should I retry?
   ↓
Maybe retry
```

甚至：

```text
Retry
 ↓
Retry
 ↓
Retry
 ↓
Retry
```

因此 Agent Reliability 必须同时解决：

```text
Infrastructure Reliability
+
LLM Reliability
+
Behavioral Reliability
```

---

# 四、Agent Reliability 的七层模型

一个成熟的 Agent Platform 可以把 Reliability 分成七层：

```text
                    Agent Reliability
                           |
       +-------------------+-------------------+
       |                   |                   |
   Infrastructure       Runtime           Intelligence
       |                   |                   |
    Network             Timeout             LLM
    Storage             Retry               Planning
    Queue               State               Reasoning
       |                   |                   |
       +-------------------+-------------------+
                           |
                     Dependency
                           |
                   Tool / API / RAG
                           |
                     Recovery
                           |
                  Safety / Governance
```

进一步可以抽象成：

```text
L1 Infrastructure
L2 Dependency
L3 Runtime
L4 State
L5 Agent Behavior
L6 Recovery
L7 Governance
```

---

# 五、Reliability 的第一原则：不要相信任何依赖

Agent 系统中的依赖包括：

```text
LLM Provider
Vector DB
Database
Redis
Kafka
External API
MCP Server
Other Agent
Search Engine
Object Storage
```

任何一个依赖都可能：

```text
Timeout
Unavailable
Rate Limited
Malformed Response
Slow Response
Partial Failure
```

所以应该遵循：

> **Every dependency is unreliable by default.**

架构：

```text
Agent
 |
 +---- LLM
 |
 +---- Tool
 |
 +---- RAG
 |
 +---- Agent B
 |
 +---- Database
```

每个依赖边界都应该存在：

```text
Timeout
Retry
Circuit Breaker
Bulkhead
Fallback
Observability
```

---

# 六、Timeout：Reliability 的第一道防线

最危险的系统通常不是：

```text
Fast Failure
```

而是：

```text
Slow Failure
```

例如：

```text
Agent
 ↓
Tool
 ↓
等待 30 秒
 ↓
等待 30 秒
 ↓
等待 30 秒
```

最终：

```text
User Request = 90s
```

因此 Agent Runtime 必须设计：

```text
Request Timeout
Step Timeout
Tool Timeout
LLM Timeout
Workflow Timeout
```

例如：

```text
Global Timeout = 60s

Step Timeout = 20s

Tool Timeout = 5s
```

形成：

```text
User Request
      |
      +---------------- 60s ----------------+
      |
      +-- Step 1 -- 10s
      |
      +-- Step 2 -- 15s
      |
      +-- Step 3 -- 20s
      |
      +-- Recovery -- 10s
```

---

# 七、Timeout 必须具有层级性

不能简单：

```java
timeout = 60s;
```

应该：

```text
Request Timeout
       |
       +-- Planning Timeout
       |
       +-- LLM Timeout
       |
       +-- Tool Timeout
       |
       +-- Recovery Timeout
```

例如：

```text
Request = 60s

Planning = 5s
LLM = 15s
Tool = 10s
Recovery = 10s
```

这样 Agent 才不会出现：

```text
Tool 调用了 60 秒
```

导致整个 Runtime 没有恢复空间。

---

# 八、Retry：不是所有失败都应该重试

Agent 系统中非常容易犯的错误：

```text
Failure
 ↓
Retry
 ↓
Failure
 ↓
Retry
```

这是危险的。

应该区分：

```text
Transient Failure
Permanent Failure
Business Failure
```

---

# 九、Transient Failure

例如：

```text
HTTP 503
Connection Reset
Temporary Network Error
Rate Limit
```

可以：

```text
Retry
```

例如：

```text
Attempt 1
   ↓
503
   ↓
Wait
   ↓
Attempt 2
   ↓
Success
```

---

# 十、Exponential Backoff

不要：

```text
Retry every 1 second
```

应该：

```text
1s
2s
4s
8s
16s
```

公式：

```text
delay = base × 2^attempt
```

同时增加：

```text
Jitter
```

例如：

```text
delay =
min(
    maxDelay,
    base * 2^attempt
)
+ random()
```

避免大量 Agent 同时重试造成：

> **Retry Storm**

---

# 十一、Retry Budget

这是 Agent Reliability 中非常值得引入的概念。

不要只定义：

```text
maxRetries = 3
```

还应该定义：

```text
Retry Budget
```

例如：

```text
每个 Request
最多允许 20% 时间用于 Retry
```

如果：

```text
Request Budget = 60s
```

那么：

```text
Retry Budget = 12s
```

这样可以避免：

```text
Agent
 ↓
Retry
 ↓
Retry
 ↓
Retry
 ↓
Retry
```

耗尽整个请求生命周期。

---

# 十二、Idempotency：Agent Reliability 的核心

这是 Agent 系统中非常重要的问题。

假设 Agent：

```text
CreatePayment
```

第一次：

```text
Request sent
 ↓
Payment succeeded
 ↓
Network timeout
```

Agent 不知道结果。

于是：

```text
Retry
```

如果没有 Idempotency：

```text
Payment
Payment
```

可能执行两次。

因此所有有副作用的 Tool：

```text
CreateOrder
CreatePayment
SendEmail
TransferMoney
DeleteResource
CreateTicket
```

应该支持：

```text
Idempotency-Key
```

例如：

```text
request_id = agent-task-123
```

第一次：

```text
POST /payment
Idempotency-Key: agent-task-123
```

再次 Retry：

```text
POST /payment
Idempotency-Key: agent-task-123
```

服务端：

```text
if key exists:
    return previous result
else:
    execute
```

---

# 十三、Agent Runtime 应该区分 Read Tool 和 Write Tool

这是非常重要的架构原则。

```text
Tools
 |
 +-- Read
 |
 +-- Write
```

Read：

```text
Search
GetUser
GetOrder
QueryDatabase
```

通常：

```text
Retry = relatively safe
```

Write：

```text
CreateOrder
DeleteUser
SendEmail
TransferMoney
```

必须：

```text
Idempotency
Authorization
Audit
Approval
```

因此：

```text
Agent → Tool
```

不应该是一个统一调用模型。

---

# 十四、Circuit Breaker

如果 Tool 连续失败：

```text
Agent
 ↓
Payment API
 ↓
Failure
 ↓
Failure
 ↓
Failure
```

继续调用只会：

```text
浪费资源
增加延迟
扩大故障
```

所以：

```text
Circuit Breaker
```

状态：

```text
CLOSED
   |
   | failures > threshold
   v
OPEN
   |
   | wait
   v
HALF_OPEN
   |
   +---- success → CLOSED
   |
   +---- failure → OPEN
```

---

# 十五、Agent 场景中的 Circuit Breaker 更复杂

传统：

```text
Service A → Service B
```

Agent：

```text
Agent
 |
 +-- Search
 +-- CRM
 +-- Payment
 +-- Email
 +-- SQL
```

应该：

```text
Tool-level Circuit Breaker
```

例如：

```text
Search Tool     CLOSED
CRM Tool        CLOSED
Payment Tool    OPEN
Email Tool      CLOSED
```

不能因为：

```text
Payment Tool
```

失败，就把整个 Agent 都关闭。

---

# 十六、Bulkhead：防止一个 Agent 拖垮整个系统

Agent 的执行具有：

```text
Long-running
Variable concurrency
Unpredictable tool calls
```

因此非常容易出现：

```text
Agent A
  ↓
Tool Slow
  ↓
Threads Occupied
  ↓
Thread Pool Exhausted
  ↓
Agent B
  ↓
Cannot Execute
```

这就是：

> **Cascading Failure**

解决：

```text
Bulkhead
```

例如：

```text
Agent Runtime
 |
 +-- Customer Agent Pool
 |
 +-- Coding Agent Pool
 |
 +-- Data Agent Pool
 |
 +-- Background Agent Pool
```

每个拥有独立：

```text
Concurrency Limit
Queue
Timeout
Resource Budget
```

---

# 十七、Concurrency Limit

Agent 不应该无限并发。

例如：

```text
Max Concurrent Agents = 100
```

进一步：

```text
Max Concurrent Tool Calls = 500
```

甚至：

```text
Per Agent = 10
Per User = 5
Per Tool = 50
```

形成：

```text
Global
  ↓
Tenant
  ↓
Agent
  ↓
Task
  ↓
Tool
```

多层限流。

---

# 十八、Agent Loop 是 Reliability 最大的敌人之一

Agent 很容易出现：

```text
Think
 ↓
Tool
 ↓
Observe
 ↓
Think
 ↓
Tool
 ↓
Observe
 ↓
Think
 ↓
Tool
```

如果没有终止条件：

```text
Infinite Loop
```

因此 Agent Runtime 必须设计：

```text
Max Steps
Max Tool Calls
Max Tokens
Max Execution Time
Max Cost
```

例如：

```text
maxSteps = 20
maxToolCalls = 30
maxTokens = 100000
maxExecutionTime = 60s
maxCost = $0.50
```

---

# 十九、Agent Budget

可以把这些统一成：

# Agent Budget

```text
Agent Budget
 |
 +-- Time Budget
 |
 +-- Token Budget
 |
 +-- Tool Budget
 |
 +-- Cost Budget
 |
 +-- Step Budget
 |
 +-- Retry Budget
```

这是 Agent Runtime 非常重要的设计。

例如：

```json
{
  "maxExecutionTime": 60000,
  "maxSteps": 20,
  "maxToolCalls": 30,
  "maxTokens": 100000,
  "maxCost": 0.5
}
```

一旦超限：

```text
STOP
```

而不是：

```text
继续让 Agent 自己决定
```

---

# 二十、State Reliability

Agent 通常是 Stateful 的：

```text
Task
 ↓
State
 ↓
Step
 ↓
State
 ↓
Step
```

因此不能只放在：

```text
JVM Memory
```

否则：

```text
Agent Runtime Crash
```

整个任务状态丢失。

应该：

```text
Agent
 ↓
State Store
```

例如：

```text
Redis
PostgreSQL
DynamoDB
```

保存：

```text
Task ID
Current Step
Plan
Tool Results
Checkpoint
Retry Count
Budget
Status
```

---

# 二十一、Checkpoint

复杂 Agent 可以：

```text
Step 1
 ↓
Checkpoint
 ↓
Step 2
 ↓
Checkpoint
 ↓
Step 3
```

如果：

```text
Step 3 Failure
```

不需要：

```text
Restart from Step 1
```

可以：

```text
Resume from Step 3
```

架构：

```text
Task
 |
 +-- Step 1 ── Checkpoint
 |
 +-- Step 2 ── Checkpoint
 |
 +-- Step 3 ── Failure
 |
 +-- Recovery
 |
 +-- Resume
```

---

# 二十二、Event-Driven Reliability

对于长时间 Agent：

```text
User
 ↓
Agent
 ↓
Tool
 ↓
等待外部系统
```

不要一直占着 HTTP Connection。

应该：

```text
API
 ↓
Task Created
 ↓
Kafka
 ↓
Agent Runtime
 ↓
Execute
 ↓
Event
 ↓
State Store
```

例如：

```text
AgentTaskCreated
AgentStepStarted
ToolCalled
ToolCompleted
AgentStepFailed
AgentRetrying
AgentCompleted
AgentFailed
```

这样可以实现：

> **Durable Execution**

---

# 二十三、Agent Failure Recovery

Agent Failure 可以分为：

```text
Failure
 |
 +-- LLM Failure
 |
 +-- Tool Failure
 |
 +-- Network Failure
 |
 +-- State Failure
 |
 +-- Logic Failure
 |
 +-- Safety Failure
 |
 +-- Budget Exhaustion
```

不同 Failure：

```text
不同 Recovery Strategy
```

---

# 二十四、Recovery Matrix

可以建立：

| Failure           | Retry | Fallback | Replan | Human |
| ----------------- | ----: | -------: | -----: | ----: |
| LLM Timeout       |     ✓ |        ✓ |        |       |
| Tool Timeout      |     ✓ |        ✓ |      ✓ |       |
| Tool 500          |     ✓ |        ✓ |        |       |
| Invalid Tool Args |       |          |      ✓ |       |
| Knowledge Missing |       |        ✓ |      ✓ |       |
| Budget Exhausted  |       |          |        |     ✓ |
| Payment Failed    |     ✓ |          |      ✓ |     ✓ |
| Safety Violation  |       |          |        |     ✓ |

这比：

```text
catch(Exception e)
```

高级得多。

---

# 二十五、Fallback

Agent 不应该只有：

```text
Success
Failure
```

可以：

```text
Primary
   ↓
Failure
   ↓
Fallback
```

例如 LLM：

```text
GPT Model
   ↓
Timeout
   ↓
Backup Model
```

Tool：

```text
Primary Search
   ↓
Failure
   ↓
Backup Search
```

RAG：

```text
Vector Search
   ↓
No Result
   ↓
Keyword Search
```

---

# 二十六、Graceful Degradation

可靠系统的一个重要原则：

> **失败时不要全部失败。**

例如：

```text
Recommendation Agent
```

推荐服务失败：

```text
Recommendation unavailable
```

但：

```text
Order Service
Payment Service
User Service
```

仍然正常。

即：

```text
Partial Failure
      ↓
Partial Success
```

而不是：

```text
One Tool Failure
      ↓
Entire Agent Failure
```

---

# 二十七、Multi-Agent Reliability

如果系统变成：

```text
Supervisor
 |
 +-- Research Agent
 |
 +-- Coding Agent
 |
 +-- Data Agent
 |
 +-- Review Agent
```

Reliability 问题会进一步扩大。

因为：

```text
Agent A
 ↓
Agent B
 ↓
Agent C
 ↓
Agent D
```

形成：

> **Distributed Agent System**

---

# 二十八、Agent-to-Agent Failure

例如：

```text
Agent A → Agent B
```

Agent B：

```text
Timeout
```

Agent A 是否：

```text
Retry?
Fallback?
Replan?
Stop?
```

因此 A2A 通信应该具备：

```text
Request ID
Correlation ID
Timeout
Retry
Idempotency
Capability
Authentication
Authorization
Trace Context
```

---

# 二十九、Distributed Reliability

Multi-Agent 系统实际上类似：

```text
Distributed System
```

所以需要：

```text
Correlation ID
Distributed Trace
Timeout
Retry
Circuit Breaker
Event
State
Checkpoint
Saga
```

尤其是：

> **不要把 Multi-Agent 当成简单的函数调用。**

它实际上是：

```text
Distributed Workflow
```

---

# 三十、Saga Pattern

如果 Agent 执行：

```text
Create Order
 ↓
Reserve Inventory
 ↓
Charge Payment
 ↓
Send Notification
```

中间：

```text
Charge Payment
```

失败怎么办？

可以设计：

```text
Create Order
      ↓
Reserve Inventory
      ↓
Charge Payment
      X
      |
      v
Compensation
      |
      +-- Release Inventory
      |
      +-- Cancel Order
```

这就是：

> **Saga / Compensation**

特别适合 Agent 执行复杂业务流程。

---

# 三十一、Human-in-the-Loop

并不是所有事情都应该让 Agent 自动恢复。

例如：

```text
Payment
Delete Production Database
Transfer Money
Deploy Production
Send Legal Document
```

应该：

```text
Agent
 ↓
Risk Detection
 ↓
Human Approval
 ↓
Execute
```

架构：

```text
                 Agent
                   |
                   v
              Risk Engine
                   |
        +----------+----------+
        |                     |
      Low Risk             High Risk
        |                     |
     Auto Execute          Human
                              |
                           Approve
                              |
                           Execute
```

这实际上是：

> **Reliability + Safety + Governance**

的结合。

---

# 三十二、Reliability 与 Safety 的边界

Reliability：

```text
系统是否按照预期工作？
```

Safety：

```text
即使工作正常，会不会做危险的事情？
```

例如 Agent：

```text
正确调用 DeleteDatabase Tool
```

从 Reliability 看：

```text
SUCCESS
```

从 Safety 看：

```text
DISASTER
```

所以：

```text
Reliable ≠ Safe
```

必须同时考虑。

---

# 三十三、Observability 是 Reliability 的基础

没有 Observability：

```text
Agent Failed
```

你不知道：

```text
为什么？
```

所以必须记录：

```text
Trace
Logs
Metrics
Events
```

例如：

```text
Trace ID: abc123

Agent Start
   ↓
LLM Call
   ↓
Tool Search
   ↓
Tool Timeout
   ↓
Retry
   ↓
Tool Success
   ↓
LLM
   ↓
Final Answer
```

这样才能进行：

```text
Root Cause Analysis
```

---

# 三十四、Agent Reliability Metrics

至少需要以下指标。

### Availability

```text
Availability =
Successful Requests
/
Total Requests
```

---

### Task Success Rate

```text
Task Success Rate =
Successful Tasks
/
Total Tasks
```

---

### Failure Rate

```text
Failure Rate =
Failed Tasks
/
Total Tasks
```

---

### Recovery Rate

```text
Recovery Rate =
Recovered Tasks
/
Failed Tasks
```

这个指标非常适合 Agent。

因为：

```text
Failure
```

本身不一定意味着：

```text
Task Failure
```

如果：

```text
Tool Timeout
 ↓
Retry
 ↓
Success
```

Agent 仍然可靠。

---

# 三十五、MTTR

传统系统：

```text
Mean Time To Recovery
```

Agent 也需要。

例如：

```text
Tool Failure
 ↓
Detection
 ↓
Retry
 ↓
Fallback
 ↓
Success
```

测量：

```text
Recovery Time
```

---

# 三十六、Agent Reliability SLO

企业 Agent 可以定义：

```text
Task Success Rate >= 99%
```

```text
p95 Latency <= 10s
```

```text
Tool Failure Recovery >= 95%
```

```text
Hallucination Rate <= 1%
```

```text
Unsafe Action Rate <= 0.01%
```

```text
Budget Violation = 0
```

注意：

> **Reliability 不应该只有 Availability SLO。**

应该是：

```text
Quality SLO
+
Reliability SLO
+
Safety SLO
+
Cost SLO
```

---

# 三十七、Reliability Dashboard

可以设计：

```text
Agent Reliability Dashboard

Task Success Rate       98.7%
Availability             99.95%
Recovery Rate            96.2%
Tool Failure Rate         1.8%
LLM Failure Rate          0.4%
Timeout Rate              0.7%
Loop Detection            0.1%
p95 Latency               8.4s
p99 Latency              19.2s
Avg Tool Calls             4.3
Avg Cost                 $0.06
```

然后继续向下钻取：

```text
Agent
 ↓
Task
 ↓
Trace
 ↓
Step
 ↓
Tool
 ↓
Failure
```

---

# 三十八、Reliability 与 Evaluation 的关系

前面我们讨论过：

> Evaluation：Agent 做得好不好？

Reliability：

> Agent 能不能稳定地把事情做好？

二者关系：

```text
                Agent Quality
                     |
          +----------+----------+
          |                     |
      Evaluation            Reliability
          |                     |
     Is it good?          Can it keep
                          doing it?
```

例如：

```text
Agent A
Accuracy = 95%
Availability = 90%
```

Agent B：

```text
Accuracy = 93%
Availability = 99.9%
```

生产环境中：

```text
Agent B
```

可能更加可靠。

---

# 三十九、Reliability + Evaluation + Observability

这三个体系应该统一起来：

```text
                 Agent Platform
                       |
       +---------------+---------------+
       |               |               |
 Observability     Evaluation      Reliability
       |               |               |
 What happened?    How good?      Will it keep
                                  working?
       |               |               |
       +---------------+---------------+
                       |
                     AgentOps
```

最终形成：

```text
Observe
   ↓
Evaluate
   ↓
Detect
   ↓
Recover
   ↓
Improve
```

---

# 四十、Agent Reliability Reference Architecture

一个企业级 Agent Platform 可以设计成：

```text
                         Client
                           |
                           v
                    API Gateway
                           |
                           v
                    Agent Gateway
                           |
             +-------------+-------------+
             |                           |
             v                           v
        Agent Runtime              Policy Engine
             |                           |
     +-------+-------+                   |
     |       |       |                   |
   LLM     Tools    RAG                  |
     |       |       |                   |
     +-------+-------+-------------------+
             |
             v
        State Manager
             |
       +-----+------+
       |            |
     Redis       PostgreSQL
       |
       v
     Kafka
       |
       v
 Event / Workflow Engine
       |
       +------------------+
       |                  |
       v                  v
   Checkpoint          Recovery
       |                  |
       +--------+---------+
                |
                v
          Observability
                |
        +-------+-------+
        |       |       |
      Trace   Metrics   Logs
        |       |       |
        +-------+-------+
                |
                v
          Evaluation
                |
                v
           AgentOps
```

---

# 四十一、Agent Runtime 中的 Reliability Middleware

一个非常实用的设计是：

```text
Agent Runtime
      |
      v
Reliability Middleware
      |
 +----+----+----+----+----+
 |    |    |    |    |    |
Timeout Retry Breaker Budget Checkpoint
```

例如：

```java
public AgentResult execute(AgentTask task) {

    return timeout.execute(() ->
        retry.execute(() ->
            circuitBreaker.execute(() ->
                budget.execute(() ->
                    agent.run(task)
                )
            )
        )
    );
}
```

实际生产环境还需要：

```text
Idempotency
Tracing
Metrics
Audit
Policy
```

---

# 四十二、不要让 LLM 决定 Reliability Policy

这是一个非常重要的原则。

错误：

```text
Tool Failure
   ↓
LLM decides:
"Maybe retry 5 times"
```

正确：

```text
Tool Failure
   ↓
Runtime Policy
   |
   +-- Retry <= 3
   +-- Timeout <= 5s
   +-- Budget <= 10s
   +-- Circuit Breaker
   +-- Fallback
```

然后才：

```text
LLM Replan
```

即：

> **LLM 决定业务行为，Runtime 决定系统边界。**

这是 Agent Platform 非常重要的架构原则。

---

# 四十三、Policy 与 Intelligence 分离

可以进一步总结成：

```text
                 Agent
                   |
       +-----------+-----------+
       |                       |
 Intelligence               Runtime
       |                       |
 Planning                  Timeout
 Reasoning                 Retry
 Tool Selection            Budget
 Replanning               Security
                           Circuit Breaker
                           State
```

LLM：

```text
What should I do?
```

Runtime：

```text
What am I allowed to do?
How long can I run?
How many times can I retry?
How much can I spend?
```

这是一种：

> **Policy / Intelligence Separation**

---

# 四十四、Agent Reliability 的五大原则

如果把全文浓缩成五条架构原则：

## 原则一：Everything Can Fail

```text
LLM
Tool
Network
Database
Agent
```

都可能失败。

---

## 原则二：Bound Everything

任何 Agent 都必须有：

```text
Time
Token
Step
Tool
Cost
Retry
```

Budget。

---

## 原则三：Make Side Effects Idempotent

所有：

```text
Write Operation
```

必须考虑：

```text
Idempotency
```

---

## 原则四：Failure Must Be Recoverable

不要：

```text
Failure → Crash
```

而应该：

```text
Failure
 ↓
Detect
 ↓
Classify
 ↓
Retry / Fallback / Replan
 ↓
Recover
```

---

## 原则五：Runtime Controls the Agent

最重要：

```text
LLM = Intelligence
Runtime = Control
```

不要让：

```text
LLM
```

突破：

```text
Security
Budget
Timeout
Concurrency
Policy
```

---

# 四十五、Agent Reliability 的最终架构思想

传统系统：

```text
Code
 ↓
Test
 ↓
Deploy
 ↓
Monitor
```

Agent 系统：

```text
Agent
 ↓
Evaluate
 ↓
Deploy
 ↓
Observe
 ↓
Detect Failure
 ↓
Recover
 ↓
Learn
 ↓
Improve
```

因此 Agent Platform 最终应该形成：

```text
                ┌───────────────────┐
                │   Agent Runtime   │
                └─────────┬─────────┘
                          │
                    Execute Task
                          │
          ┌───────────────┼───────────────┐
          │               │               │
       Timeout          Budget         Policy
          │               │               │
          └───────────────┼───────────────┘
                          │
                     Tool / LLM
                          │
                     Failure?
                          │
                    ┌─────┴─────┐
                    │           │
                   NO          YES
                    │           │
                    │       Classify
                    │           │
                    │    ┌──────┼──────┐
                    │    │      │      │
                    │  Retry  Fallback Replan
                    │    │      │      │
                    │    └──────┼──────┘
                    │           │
                    └─────┬─────┘
                          │
                       Recover
                          │
                       Complete
                          │
                    Observability
                          │
                      Evaluation
                          │
                       Improve
```

---

# 四十六、结语：Reliability 是 Agent 从 Demo 走向生产的分水岭

Agent 最容易做出来的是：

```text
User
 ↓
LLM
 ↓
Tool
 ↓
Answer
```

真正困难的是：

```text
User
 ↓
Agent
 ↓
LLM
 ↓
Tool
 ↓
Timeout
 ↓
Retry
 ↓
Tool
 ↓
Invalid Result
 ↓
Replan
 ↓
Fallback
 ↓
Checkpoint
 ↓
Resume
 ↓
Human Approval
 ↓
Execute
 ↓
Result
```

这才是真正的：

# Production-Grade Agent

所以，从 Agent Platform 架构角度看，**Reliability 不是一个附加功能，而是 Agent Runtime 的核心能力。**

可以把整个 Agent Platform 的核心能力概括为：

```text
                  Agent Platform
                        |
       +----------------+----------------+
       |                |                |
     Runtime        Observability     Evaluation
       |                |                |
       +----------------+----------------+
                        |
                    Reliability
                        |
       +----------------+----------------+
       |                |                |
     Resilience       Recovery        Governance
       |                |                |
       +----------------+----------------+
                        |
                    Production
```

最终形成一套完整的 Agent Engineering 闭环：

> **Runtime 保证执行，Observability 负责看见，Evaluation 负责判断，Reliability 负责恢复，Governance 负责约束，AgentOps 负责持续改进。**

这也是从普通 **LLM Application** 走向真正 **Enterprise Agent Platform** 的关键一步。



