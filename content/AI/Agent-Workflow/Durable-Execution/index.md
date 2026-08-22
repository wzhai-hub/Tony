---
title: Durable Execution：从一次性 Agent 调用到可恢复分布式智能工作流
# tags:
#   - nodejs
date: '2026-08-08'
summary: 一种将工作流执行状态、执行进度和关键副作用持久化，使任务能够跨越进程、机器、Pod、网络故障和人工等待继续执行的执行模型
---
# Durable Execution：从一次性 Agent 调用到可恢复分布式智能工作流

## 引言：Agent 真正进入生产环境后，为什么一定需要 Durable Execution？

过去我们设计 AI 应用，通常假设一次请求生命周期很短：

```text
User
 ↓
API
 ↓
LLM
 ↓
Response
```

例如：

```text
用户：帮我总结这篇文章。

API
 ↓
LLM
 ↓
Summary
 ↓
HTTP Response
```

整个过程可能只有几秒。

但 Agent 出现之后，事情发生了根本变化。

一个真正的企业级 Agent 可能执行：

```text
User
 ↓
Agent
 ↓
分析任务
 ↓
查询数据库
 ↓
调用搜索 API
 ↓
分析数据
 ↓
生成执行计划
 ↓
调用企业 API
 ↓
等待 Human Approval
 ↓
修改配置
 ↓
部署服务
 ↓
监控指标
 ↓
发现异常
 ↓
Rollback
 ↓
生成报告
```

这个过程可能持续：

```text
10 seconds
10 minutes
10 hours
10 days
```

期间任何东西都可能失败：

```text
LLM Timeout
Tool Timeout
Network Failure
Pod Restart
Node Failure
Database Failure
Kafka Rebalance
Human Approval Delay
Deployment
Scaling
```

如果 Agent 的状态全部存在：

```text
JVM Heap
Python Process
Node.js Process
```

那么：

```text
Process Crash
      ↓
State Lost
      ↓
Workflow Lost
```

于是 Agent 只能重新开始。

这在传统 CRUD 系统中已经是问题，在 Agent 系统中则更加严重。

因为 Agent 不只是计算：

```text
calculate()
```

它还可能产生不可逆副作用：

```text
sendEmail()
createOrder()
payment()
deleteData()
deploy()
```

因此，现代 Agent Architecture 正在从：

> **Stateless Request/Response**

逐渐演进到：

> **Durable Execution**

可以把 Durable Execution 定义为：

> **一种将工作流执行状态、执行进度和关键副作用持久化，使任务能够跨越进程、机器、Pod、网络故障和人工等待继续执行的执行模型。**

如果说：

```text
Checkpoint
```

解决的是：

> “我执行到哪里了？”

那么：

```text
Durable Execution
```

解决的是：

> **“即使执行我的那个进程死了，我还能不能可靠地继续完成整个任务？”**

---

# 一、Durable Execution 到底是什么？

Durable Execution 可以抽象成：

```text
Workflow
   ↓
Execute
   ↓
Persist Progress
   ↓
Failure
   ↓
Recover
   ↓
Resume
   ↓
Continue
```

最重要的三个字：

> **Progress is Durable**

也就是说：

```text
Execution Progress
```

不能只存在：

```text
Process Memory
```

而应该存在：

```text
Durable Storage
```

例如：

```text
Agent Worker
     │
     ▼
┌───────────────┐
│ Execute Step  │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ Persist State │
└───────┬───────┘
        │
        ▼
   Next Step
```

如果 Worker 挂掉：

```text
Worker
  💥
```

新的 Worker：

```text
New Worker
    ↓
Load Durable State
    ↓
Resume
```

---

# 二、Durable Execution 和普通 Retry 有什么区别？

这是最重要的概念之一。

很多系统认为：

```text
Failure
 ↓
Retry
```

就是 Durable Execution。

其实完全不是。

例如：

```text
Step 1
Step 2
Step 3
Step 4
Step 5
```

执行到：

```text
Step 4
 ↓
Failure
```

普通 Retry：

```text
Retry
 ↓
Step 1
 ↓
Step 2
 ↓
Step 3
 ↓
Step 4
```

而 Durable Execution：

```text
Checkpoint
        ↓
Step 4
 ↓
Failure
 ↓
Recover
 ↓
Step 4
```

如果前面的步骤已经完成：

```text
Step 1 ✓
Step 2 ✓
Step 3 ✓
```

就不应该重新执行。

所以：

```text
Retry
=
重新尝试

Durable Execution
=
从可靠执行位置继续
```

---

# 三、Durable Execution 与 Checkpoint 的关系

前面的 Checkpoint 文章中已经讨论过：

```text
Checkpoint
=
State Snapshot
+
Execution Cursor
```

而 Durable Execution 可以看成：

```text
Durable Execution
=
Checkpoint
+
Recovery
+
Replay
+
Retry
+
Idempotency
+
Workflow Coordination
```

因此：

```text
Checkpoint
        ↓
Durability Primitive

Durable Execution
        ↓
Execution Architecture
```

Checkpoint 是 Durable Execution 的基础设施之一。

但仅有 Checkpoint 还不够。

---

# 四、一个真正的 Durable Workflow

例如：

```text
A → B → C → D → E
```

定义：

```text
A = Research
B = Analyze
C = Generate Plan
D = Execute
E = Report
```

传统 Agent：

```text
A
 ↓
B
 ↓
C
 ↓
D
 ↓
E
```

如果：

```text
D
 ↓
Crash
```

全部重新开始。

Durable Agent：

```text
A
 ↓
CP1
 ↓
B
 ↓
CP2
 ↓
C
 ↓
CP3
 ↓
D
 ↓
CP4
 ↓
E
```

Crash：

```text
D
 ↓
Crash
```

恢复：

```text
Load CP3
 ↓
D
 ↓
CP4
 ↓
E
```

这里的核心不是“保存数据”，而是：

> **把 Workflow Progress 变成持久化数据。**

---

# 五、Durable Execution 的四个核心能力

可以把 Durable Execution 分成四个核心能力：

```text
Durable Execution
│
├── Persistence
├── Recovery
├── Deterministic Replay
└── Side-Effect Management
```

分别对应：

### Persistence

执行到哪里必须保存。

### Recovery

进程死了以后可以恢复。

### Deterministic Replay

恢复过程中不能因为历史执行不同而产生不可控结果。

### Side-Effect Management

不能因为 Retry 导致：

```text
Payment × 2
Email × 2
Order × 2
```

---

# 六、Durable Execution 的核心架构

一个生产级 Agent Runtime 可以设计成：

```text
                         User
                           │
                           ▼
                     Agent API
                           │
                           ▼
                  ┌─────────────────┐
                  │ Agent Runtime   │
                  └────────┬────────┘
                           │
                           ▼
                    Workflow Engine
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
           Planner      State       Scheduler
              │            │            │
              └────────────┼────────────┘
                           │
                           ▼
                    Durable Executor
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
        Checkpoint      Event Log     Effect Log
             │             │             │
             └─────────────┼─────────────┘
                           │
                           ▼
                      Tool Gateway
                           │
                           ▼
                    External Systems
```

这个架构已经非常接近：

```text
AI Workflow Engine
```

而不仅仅是：

```text
LLM Agent
```

---

# 七、Durable Execution 的基本执行模型

可以把一个 Workflow 表示成：

```text
Workflow
=
Sequence of Durable Steps
```

例如：

```text
Step 1: Search
Step 2: Analyze
Step 3: Generate Plan
Step 4: Approval
Step 5: Execute
Step 6: Verify
Step 7: Report
```

每个 Step 都有一个生命周期：

```text
CREATED
   ↓
RUNNING
   ↓
SUCCESS
   │
   └── FAILED
```

而 Durable Execution 要保证：

```text
SUCCESS
```

一旦持久化：

> 后续恢复时不要再次执行这个 Step。

---

# 八、Step 是 Durable Execution 的最小单位

传统代码：

```java
search();
analyze();
deploy();
```

如果没有 Durable Execution：

```text
search()
analyze()
deploy()
```

只是三个普通函数。

而 Durable Execution：

```text
step("search", () -> search());

step("analyze", () -> analyze());

step("deploy", () -> deploy());
```

意味着：

```text
step()
```

成为一个 Durable Boundary。

概念上：

```text
Step
 ↓
Execute
 ↓
Persist Result
 ↓
Continue
```

---

# 九、为什么 Step 必须有唯一 ID？

假设：

```text
Step 1 = Search
Step 2 = Analyze
Step 3 = Deploy
```

必须有：

```text
stepId
```

例如：

```text
research.search
research.analyze
deployment.execute
```

恢复时：

```text
stepId = deployment.execute
```

系统才能知道：

```text
这个 Step 是否已经成功执行？
```

因此：

```text
Execution Identity
=
workflowId
+
runId
+
stepId
```

进一步可以：

```text
tenantId
+
workflowId
+
runId
+
stepId
+
attempt
```

形成完整执行身份。

---

# 十、Durable Execution 与 Exactly Once

这是一个非常容易产生误解的地方。

很多人会认为：

> Durable Execution = Exactly Once Execution。

并不是。

分布式系统中真正的：

```text
Exactly Once
```

非常困难。

例如：

```text
Agent
 ↓
Payment API
 ↓
Payment Success
 ↓
Network Timeout
```

Agent 不知道：

```text
Payment
```

到底成功还是失败。

于是：

```text
Retry Payment
```

可能产生：

```text
Double Payment
```

所以 Durable Execution 通常需要：

```text
At-Least-Once Execution
+
Idempotency
```

最终实现：

```text
Effectively Once
```

---

# 十一、At-Least-Once 是现实世界的常态

假设：

```text
Step A
 ↓
External API
```

请求成功：

```text
HTTP 200
```

但响应丢失：

```text
Network
 ↓
Timeout
```

系统只能判断：

```text
UNKNOWN
```

它无法确定：

```text
Executed?
```

于是通常只能：

```text
Retry
```

这意味着：

```text
At-Least-Once
```

是非常现实的执行语义。

因此：

> **Durable Execution 的关键不是消灭 Retry，而是让 Retry 安全。**

---

# 十二、Idempotency 是 Durable Execution 的灵魂

例如：

```java
payment(orderId, idempotencyKey);
```

其中：

```text
idempotencyKey
=
workflowId + stepId
```

第一次：

```text
payment("ORDER-1001", "WF1:PAYMENT")
```

成功。

第二次：

```text
payment("ORDER-1001", "WF1:PAYMENT")
```

服务端发现：

```text
WF1:PAYMENT
```

已经执行。

于是返回：

```text
Previous Result
```

而不是：

```text
Execute Again
```

因此：

```text
Durable Execution
        +
Idempotent Effects
        =
Reliable Recovery
```

---

# 十三、Side Effect 是 Durable Execution 最难的问题

纯计算：

```text
calculateTax()
```

比较容易。

因为：

```text
same input
→
same output
```

但是：

```text
sendEmail()
```

是副作用。

还有：

```text
payment()
createOrder()
deleteUser()
deploy()
```

都是 Side Effect。

因此 Durable Execution 通常把 Workflow 分成：

```text
Deterministic Logic
+
Side Effects
```

例如：

```text
Workflow
 │
 ├── Compute
 │
 ├── Decision
 │
 └── Activity / Effect
```

Activity：

> 可以产生外部副作用的可持久化执行单元。

---

# 十四、Activity Pattern

可以设计：

```text
Workflow
   ↓
Activity
   ↓
External System
```

例如：

```text
Workflow:
    paymentActivity()
```

Activity：

```text
PaymentActivity
 ↓
Payment Service
```

Workflow 本身负责：

```text
Decision
```

Activity 负责：

```text
Side Effect
```

这与传统 Durable Workflow Engine 的思想高度一致。

---

# 十五、为什么要把 LLM 放在 Activity 中？

这是 Agent Durable Execution 非常关键的设计问题。

LLM 是：

```text
Probabilistic
External
Non-Deterministic
```

因此不要简单地把：

```text
LLM Call
```

当成普通 deterministic function。

更合理：

```text
Workflow
   ↓
LLM Activity
   ↓
Model
   ↓
Persist Result
```

例如：

```text
PlanActivity
AnalyzeActivity
SummarizeActivity
```

每次执行结果保存：

```text
prompt
model
parameters
response
```

恢复时：

```text
Load Historical Result
```

而不是盲目重新调用 LLM。

---

# 十六、为什么 LLM Replay 特别困难？

假设：

```text
2026-08-22
Model A
```

输出：

```text
Plan A
```

一周之后：

```text
Model A updated
```

再次调用：

```text
Plan B
```

即使：

```text
Prompt
```

完全一样。

因此：

```text
Replay
```

可能不是：

```text
Same Result
```

所以 Durable Agent 需要区分：

```text
Recovery
```

和：

```text
Re-execution
```

Recovery：

```text
Load previous result
```

Re-execution：

```text
Call LLM again
```

生产系统通常应该优先：

> **Replay historical outputs，避免无意义地重新调用外部非确定性服务。**

---

# 十七、Deterministic Workflow

Durable Workflow 最核心的思想之一：

> **Workflow Control Flow 必须尽可能 Deterministic。**

例如：

```java
if (result.score() > 0.8) {
    approve();
}
```

没问题。

但是：

```java
if (Math.random() > 0.5) {
    approve();
}
```

会破坏 Replay。

因为第一次：

```text
random = 0.2
```

第二次：

```text
random = 0.9
```

Control Flow 不一样。

因此：

```text
Workflow Logic
```

应该尽量避免：

```text
Random
Current Time
Uncontrolled IO
Unrecorded External State
```

---

# 十八、时间也是 Non-Deterministic Input

例如：

```java
if (Instant.now().isAfter(deadline)) {
    timeout();
}
```

第一次：

```text
10:00:00
```

Replay：

```text
10:10:00
```

结果不同。

因此 Durable Workflow 通常需要：

```text
Workflow Clock
```

而不是直接：

```text
System.currentTimeMillis()
```

例如：

```java
workflow.now();
```

第一次：

```text
10:00
```

之后 Replay：

```text
10:00
```

保持一致。

---

# 十九、Random 也必须被控制

如果需要随机：

```java
workflow.random();
```

系统应该把：

```text
Random Seed
```

持久化。

这样：

```text
Replay
```

仍然可以产生：

```text
Same Random Sequence
```

这就是：

> **Deterministic Randomness**

---

# 二十、Durable Timer

这是 Durable Execution 非常重要的能力。

假设：

```text
Agent
 ↓
Wait 24 hours
 ↓
Continue
```

错误：

```java
Thread.sleep(24 * 60 * 60 * 1000);
```

这会占用：

```text
Thread
Pod
Memory
```

正确：

```text
Durable Timer
```

系统保存：

```text
wakeUpAt = 2026-08-23T10:00
```

然后：

```text
Worker Released
```

24 小时后：

```text
Scheduler
 ↓
Wake Workflow
 ↓
Load Checkpoint
 ↓
Resume
```

因此：

> **Durable Timer 是 Durable Execution 的核心组成部分。**

---

# 二十一、Approval 本质上也是 Durable Timer + Checkpoint

例如：

```text
Agent
 ↓
Approval Required
 ↓
WAIT
```

可能等待：

```text
5 minutes
5 hours
5 days
```

系统不能：

```text
Thread.sleep()
```

而应该：

```text
Checkpoint
 ↓
WAITING
 ↓
Worker Release
```

用户批准：

```text
Event
 ↓
Resume Workflow
```

因此前面的：

```text
Approval
Checkpoint
Durable Execution
```

实际上是一条完整的技术链。

---

# 二十二、Durable Execution 与 Event Driven Architecture

恢复事件可以通过：

```text
Kafka
```

例如：

```text
Agent Workflow
 ↓
WAITING_APPROVAL
```

保存：

```text
Checkpoint
```

然后发布：

```text
ApprovalRequired
```

用户：

```text
Approve
```

产生：

```text
ApprovalApproved
```

Kafka：

```text
ApprovalApproved
 ↓
Workflow Resume Consumer
 ↓
Load Checkpoint
 ↓
Resume
```

完整架构：

```text
              Kafka
               │
       ┌───────┴────────┐
       │                │
ApprovalRequested   ApprovalApproved
       │                │
       ▼                ▼
  Human UI         Resume Worker
                        │
                        ▼
                   Checkpoint
                        │
                        ▼
                     Resume
```

这非常适合微服务架构。

---

# 二十三、Durable Execution + Kafka

如果使用 Java 技术栈，可以把 Kafka 作为：

```text
Workflow Event Backbone
```

例如：

```text
WorkflowStarted
StepStarted
StepCompleted
ApprovalRequired
ApprovalApproved
StepFailed
WorkflowCompleted
```

Kafka：

```text
workflow-events
```

但需要注意：

> Kafka Event Log 不等于 Workflow State。

可以：

```text
Kafka
 ↓
Event Stream
```

同时：

```text
PostgreSQL
 ↓
Durable State
```

即：

```text
Kafka
=
Event Transport

PostgreSQL
=
Workflow State
```

两者职责不同。

---

# 二十四、Durable Execution + Redis

Redis 非常适合：

```text
Distributed Lock
Lease
Task Queue
Hot State
Timer Scheduling
Deduplication
```

例如：

```text
workflow:lock:{workflowId}
```

但是：

```text
Redis
```

不应该天然被视为唯一的 Durable Source of Truth。

更合理：

```text
PostgreSQL
    ↓
Source of Truth

Redis
    ↓
Hot Runtime State
```

---

# 二十五、Durable Execution + PostgreSQL

如果使用 Spring Boot：

```text
Agent Runtime
      ↓
PostgreSQL
```

可以建立：

```sql
CREATE TABLE workflow_run (
    run_id          VARCHAR(64) PRIMARY KEY,
    workflow_id     VARCHAR(128),
    status          VARCHAR(32),
    current_step    VARCHAR(128),
    version         BIGINT,
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP
);
```

Step：

```sql
CREATE TABLE workflow_step (
    run_id          VARCHAR(64),
    step_id         VARCHAR(128),
    status          VARCHAR(32),
    attempt         INT,
    input           JSONB,
    output          JSONB,
    started_at      TIMESTAMP,
    completed_at    TIMESTAMP,
    PRIMARY KEY(run_id, step_id)
);
```

Checkpoint：

```sql
CREATE TABLE workflow_checkpoint (
    run_id          VARCHAR(64),
    checkpoint_id   VARCHAR(64),
    sequence_no     BIGINT,
    state           JSONB,
    created_at      TIMESTAMP,
    PRIMARY KEY(run_id, checkpoint_id)
);
```

---

# 二十六、Workflow State Machine

Durable Execution 的核心其实是：

```text
State Machine
```

例如：

```text
CREATED
  ↓
RUNNING
  ↓
WAITING
  ↓
RUNNING
  ↓
COMPLETED
```

异常：

```text
RUNNING
  ↓
FAILED
  ↓
RETRYING
  ↓
RUNNING
```

永久失败：

```text
RETRYING
  ↓
FAILED_PERMANENT
```

取消：

```text
RUNNING
  ↓
CANCELLED
```

所以 Durable Execution 并不是简单的：

```text
while(true)
```

而是：

> **Persistent State Machine + Scheduler + Recovery Engine**

---

# 二十七、Retry Policy

Durable Execution 必须设计 Retry Policy。

最简单：

```text
Retry 3 times
```

更合理：

```text
maxAttempts = 5
```

配合：

```text
Exponential Backoff
```

例如：

```text
1s
2s
4s
8s
16s
```

再加入：

```text
Jitter
```

变成：

```text
delay = base * 2^attempt + random(jitter)
```

避免大量 Agent 同时恢复导致：

```text
Thundering Herd
```

---

# 二十八、Retryable vs Non-Retryable Error

不是所有错误都应该 Retry。

例如：

```text
HTTP 500
Timeout
503
Network Error
```

通常：

```text
Retryable
```

而：

```text
400
401
403
Invalid Parameter
Business Rule Violation
```

通常：

```text
Non-Retryable
```

因此：

```text
Error
 ↓
Classifier
 ↓
┌──────────────┐
│ Retryable    │
│ Non-Retryable│
└──────────────┘
```

Agent Runtime 必须有统一的错误分类机制。

---

# 二十九、Circuit Breaker

如果某个 Tool：

```text
Payment Service
```

连续失败：

```text
100%
```

不能让 Agent：

```text
Retry
Retry
Retry
Retry
...
```

应该：

```text
Circuit Breaker
```

进入：

```text
OPEN
```

Workflow：

```text
Payment Activity
 ↓
Circuit Open
 ↓
Wait
 ↓
Resume Later
```

这可以避免 Agent 把下游服务彻底打挂。

---

# 三十、Timeout 是 Durable Execution 的第一等公民

需要区分：

```text
Workflow Timeout
Step Timeout
Activity Timeout
Network Timeout
Human Timeout
```

例如：

```text
Workflow:
24h

LLM:
60s

Search:
10s

Payment:
30s

Approval:
8h
```

不要设计成：

```text
global timeout = 60s
```

因为 Agent Workflow 天生可能是长生命周期。

---

# 三十一、Cancellation

生产 Agent 必须支持：

```text
Cancel Workflow
```

例如用户：

```text
停止任务。
```

系统：

```text
RUNNING
 ↓
CANCELLING
 ↓
CANCELLED
```

但如果当前正在执行：

```text
Payment
```

就不能简单：

```text
kill thread
```

因为：

```text
Payment
```

可能已经提交。

因此 Cancellation 需要：

```text
Cooperative Cancellation
```

而不是：

```text
Force Kill
```

---

# 三十二、Graceful Shutdown

Kubernetes：

```text
Deployment
```

导致：

```text
SIGTERM
```

Agent Worker 应该：

```text
Stop Accepting New Work
 ↓
Finish Current Safe Point
 ↓
Persist Checkpoint
 ↓
Release Lease
 ↓
Exit
```

而不是：

```text
SIGTERM
 ↓
kill -9
```

这就是：

> **Durability-aware shutdown**

---

# 三十三、Worker Lease

一个 Workflow 被：

```text
Worker A
```

执行。

需要：

```text
Lease
```

例如：

```text
leaseOwner = worker-A
leaseUntil = 10:05
```

Worker A 挂掉：

```text
Lease expires
```

Worker B：

```text
Acquire Lease
 ↓
Load Checkpoint
 ↓
Resume
```

因此：

```text
Worker
```

是短生命周期的。

```text
Workflow
```

是长生命周期的。

这是一种非常重要的架构分离：

```text
Worker Lifecycle
≠
Workflow Lifecycle
```

---

# 三十四、Durable Execution 的核心原则：Externalize State

传统：

```text
Worker
 ├── State
 ├── Progress
 └── Execution
```

Worker 挂了：

```text
Everything Lost
```

Durable：

```text
Worker
 └── Execution

External Storage
 ├── State
 ├── Progress
 ├── History
 └── Effects
```

Worker 挂了：

```text
Execution Lost
```

但：

```text
State Survives
```

新 Worker：

```text
Load State
 ↓
Continue
```

这是 Durable Execution 最核心的架构思想。

---

# 三十五、Durable Execution 与 Kubernetes 的最佳实践

在 Kubernetes 中推荐：

```text
                 Kubernetes
                     │
          ┌──────────┴──────────┐
          │                     │
      Worker Pod A          Worker Pod B
          │                     │
          └──────────┬──────────┘
                     │
                     ▼
             Durable Storage
                     │
          ┌──────────┼──────────┐
          │          │          │
       Postgres    Redis      Kafka
```

Worker：

```text
Stateless
```

Workflow：

```text
Stateful
```

这是最适合 Cloud Native 的模型。

---

# 三十六、Durable Execution 与 Cloud Native

传统服务：

```text
Pod
=
Application
+
State
```

Cloud Native：

```text
Pod
=
Ephemeral Compute
```

所以：

```text
State
```

必须离开 Pod。

Agent Workflow 又天然具有：

```text
Long Running
```

因此：

```text
Durable Execution
+
Cloud Native
```

天然匹配。

可以总结：

```text
Ephemeral Compute
+
Durable State
=
Reliable Workflow
```

---

# 三十七、Durable Execution 与 Agent Memory

再次区分：

```text
Memory
```

和：

```text
Durable Execution
```

Agent Memory：

```text
What do I know?
```

Checkpoint：

```text
Where am I?
```

Durable Execution：

```text
How can I continue?
```

例如：

```text
Memory:
客户喜欢 Java。

Checkpoint:
正在执行订单 1001。

Durable Runtime:
Worker 挂掉后从 Step 7 恢复。
```

三者是不同层次。

---

# 三十八、Durable Execution 与 Agent Communication

Multi-Agent：

```text
Supervisor
 ↓
Research Agent
 ↓
Coding Agent
 ↓
Deployment Agent
```

如果：

```text
Coding Agent
```

发送：

```text
CODE_READY
```

Supervisor 必须持久化：

```text
Message Received
```

否则：

```text
Supervisor Crash
```

可能导致：

```text
Message Lost
```

因此 Agent Communication 也应该考虑：

```text
Durable Message
```

可以设计：

```text
Agent Message
+
Message ID
+
Delivery Status
+
Checkpoint
```

最终形成：

```text
Durable Agent Communication
```

---

# 三十九、Durable Execution 与 Saga

如果 Agent Workflow：

```text
Create Order
 ↓
Reserve Inventory
 ↓
Charge Payment
 ↓
Ship
```

中间：

```text
Charge Payment
```

成功：

```text
Ship
```

失败。

怎么办？

不能简单：

```text
Retry forever
```

需要：

```text
Compensation
```

例如：

```text
Ship failed
 ↓
Refund Payment
 ↓
Release Inventory
 ↓
Cancel Order
```

这就是：

> **Saga Pattern**

因此复杂 Agent Workflow：

```text
Durable Execution
+
Saga
```

是非常自然的组合。

---

# 四十、Agent Workflow 的 Saga 模型

例如：

```text
Step 1
Create Order
Compensation:
Cancel Order

Step 2
Reserve Inventory
Compensation:
Release Inventory

Step 3
Charge Payment
Compensation:
Refund Payment

Step 4
Ship
Compensation:
Cancel Shipment
```

Workflow：

```text
Create
 ↓
Reserve
 ↓
Pay
 ↓
Ship
```

如果：

```text
Ship FAILED
```

执行：

```text
Refund
 ↓
Release Inventory
 ↓
Cancel Order
```

Durable Execution 负责：

```text
保存执行进度
```

Saga 负责：

```text
处理失败后的业务补偿
```

---

# 四十一、Checkpoint + Saga + Approval

到这里可以把前面几篇文章连接起来。

一个生产 Agent：

```text
              Agent
                │
                ▼
             Planning
                │
                ▼
          ┌─────────────┐
          │ Checkpoint  │
          └──────┬──────┘
                 │
                 ▼
              Policy
                 │
                 ▼
             Approval
                 │
                 ▼
              Execute
                 │
                 ▼
               Saga
                 │
          ┌──────┴──────┐
          ▼             ▼
       Success       Failure
                         │
                         ▼
                    Compensation
```

这已经非常接近：

> **Enterprise Agent Runtime**

---

# 四十二、Durable Execution 的数据库一致性问题

假设：

```text
1. Tool Execution Success
2. Save Checkpoint
```

如果：

```text
1 Success
2 Failure
```

怎么办？

下一次恢复：

```text
Retry Tool
```

可能产生重复副作用。

因此需要考虑：

```text
Tool Effect
+
Checkpoint
```

之间的一致性。

常见解决方案：

### 方案一：Idempotency

最简单：

```text
Effect ID
```

保证重复调用不会产生重复结果。

### 方案二：Transactional Outbox

```text
Business Transaction
 ↓
Outbox Event
 ↓
Commit
```

之后：

```text
Outbox
 ↓
Dispatcher
 ↓
External System
```

### 方案三：Effect Journal

保存：

```text
effectId
status
result
```

恢复：

```text
effectId exists
```

则不再执行。

---

# 四十三、Transactional Outbox 与 Agent

例如：

```text
Agent
 ↓
Update Workflow State
 ↓
Publish Event
```

不要：

```text
DB Commit
 ↓
Kafka Publish
```

因为可能：

```text
DB Success
Kafka Failure
```

应该：

```text
BEGIN

UPDATE workflow

INSERT INTO outbox

COMMIT
```

然后：

```text
Outbox Worker
 ↓
Kafka
```

这样：

```text
Workflow State
+
Event
```

具有更强的一致性。

对于 Java/Spring Boot Agent Runtime，这是非常值得采用的企业级模式。

---

# 四十四、Durable Execution 的性能问题

Durability 不是免费的。

每个 Step 都：

```text
Serialize
 ↓
Network
 ↓
Database
 ↓
Commit
```

会增加：

```text
Latency
I/O
Storage
CPU
```

因此必须在：

```text
Durability
vs
Performance
```

之间平衡。

可以设计：

```text
Critical Step
 ↓
Sync Checkpoint
```

普通 Step：

```text
Async Checkpoint
```

甚至：

```text
Checkpoint Every N Steps
```

但要明确：

> Checkpoint 越少，恢复时可能重新执行的工作越多。

---

# 四十五、Durability Level

可以设计多个级别：

```text
LEVEL 0
Memory Only

LEVEL 1
Async Persistence

LEVEL 2
Checkpoint per Step

LEVEL 3
Checkpoint + Effect Journal

LEVEL 4
Checkpoint + Effect Journal + Saga

LEVEL 5
Fully Durable Workflow
```

不同业务选择不同等级。

例如：

```text
Chatbot
→ Level 0/1

Research Agent
→ Level 2

Enterprise Workflow
→ Level 3

Payment Agent
→ Level 4/5
```

---

# 四十六、Durable Execution 的恢复算法

一个简单恢复算法：

```text
1. Load latest checkpoint

2. Validate schema

3. Validate workflow version

4. Acquire workflow lease

5. Load completed steps

6. Load pending effects

7. Determine next executable step

8. Execute step

9. Persist result

10. Release lease
```

伪代码：

```java
public void resume(String runId) {

    WorkflowState state =
        repository.loadState(runId);

    acquireLease(runId);

    while (!state.completed()) {

        Step step =
            scheduler.next(state);

        if (effectJournal.exists(step.id())) {
            state.apply(
                effectJournal.result(step.id())
            );
            continue;
        }

        StepResult result =
            execute(step, state);

        persistStepResult(
            runId,
            step,
            result
        );

        checkpoint(state);
    }

    releaseLease(runId);
}
```

这就是 Durable Execution 最基本的 Runtime Loop。

---

# 四十七、真正复杂的是 Scheduler

在简单 Workflow：

```text
A → B → C
```

Scheduler 很简单。

但是 Agent Workflow 可能是：

```text
       ┌── Search A ──┐
       │              │
Start ─┤              ├─ Analyze
       │              │
       └── Search B ──┘
```

甚至：

```text
       ┌──── Agent A ────┐
       │                 │
Supervisor               │
       │                 │
       └──── Agent B ────┘
```

于是 Scheduler 需要处理：

```text
Dependencies
Parallelism
Retries
Timeouts
Human Wait
Compensation
Cancellation
```

这也是为什么：

> **Durable Execution 最终会演化成 Workflow Orchestration。**

---

# 四十八、Parallel Execution

例如：

```text
          ┌── Search Google
          │
Start ────┼── Search DB
          │
          └── Search GitHub
                   │
                   ▼
                Merge
```

三个 Task 可以并行：

```text
A
B
C
```

完成后：

```text
Join
```

Checkpoint 必须记录：

```text
A = DONE
B = DONE
C = RUNNING
```

如果：

```text
Worker C Crash
```

恢复：

```text
A → Skip
B → Skip
C → Resume
```

这就是：

> **Fine-Grained Durable Parallelism**

---

# 四十九、Durable Execution 与 Map-Reduce

Agent 研究任务：

```text
100 documents
```

可以：

```text
Map
 ↓
100 Agents
 ↓
Analyze
 ↓
Reduce
 ↓
Summary
```

如果其中：

```text
Agent 73
```

失败。

不应该重新：

```text
Agent 1...100
```

而应该：

```text
Agent 73
 ↓
Retry
```

因此 Checkpoint：

```text
Task-level durability
```

非常重要。

---

# 五十、Durable Execution 与 Long-Running Agent

一个真正的 Autonomous Agent 可能：

```text
Day 1
Research

Day 2
Analyze

Day 3
Wait for approval

Day 4
Execute

Day 5
Monitor

Day 6
Generate report
```

这时候：

```text
HTTP Request
```

已经完全不适合。

正确模型：

```text
Long Running Workflow
```

因此 Agent Runtime 必须支持：

```text
Pause
Resume
Sleep
Wake
Retry
Recover
Cancel
Compensate
```

这就是 Durable Execution。

---

# 五十一、Durable Execution 与传统微服务架构

传统微服务：

```text
Request
 ↓
Service A
 ↓
Service B
 ↓
Service C
```

Agent：

```text
Workflow
 ↓
Agent
 ↓
Tool A
 ↓
Tool B
 ↓
Human
 ↓
Tool C
```

传统微服务主要解决：

```text
Service Availability
```

Agent Workflow 还需要解决：

```text
Execution Continuity
```

因此：

```text
Microservice Architecture
+
Durable Workflow
=
Enterprise Agent Architecture
```

---

# 五十二、Durable Execution 与 Observability

普通微服务：

```text
Trace
 ↓
Span
 ↓
Service
```

Agent：

```text
Trace
 ↓
Workflow Run
 ↓
Step
 ↓
Checkpoint
 ↓
Tool
 ↓
External Effect
```

因此建议统一：

```text
traceId
workflowId
runId
stepId
checkpointId
effectId
```

例如：

```text
traceId = T001
workflowId = W001
runId = R001
stepId = S007
checkpointId = CP007
effectId = E007
```

这样可以建立：

```text
Trace
  ↕
Workflow
  ↕
Checkpoint
  ↕
Effect
```

这对于你之前关注的 OpenTelemetry / Tempo / Grafana 类技术栈尤其重要。

---

# 五十三、Durable Execution Metrics

至少应该监控：

```text
workflow_started_total
workflow_completed_total
workflow_failed_total

workflow_duration_seconds

step_execution_total
step_retry_total
step_failure_total

checkpoint_write_total
checkpoint_write_latency

checkpoint_recovery_total
checkpoint_recovery_latency

workflow_waiting_total

activity_execution_total
activity_retry_total

workflow_stuck_total
```

尤其重要：

```text
Recovery Rate
Retry Rate
Recovery Latency
Workflow Duration
Checkpoint Storage Growth
```

---

# 五十四、Workflow Stuck Detection

长时间 Workflow 最容易出现：

```text
WAITING
```

但实际上已经：

```text
Dead
```

例如：

```text
WAITING_APPROVAL
```

持续：

```text
30 days
```

应该检测：

```text
Workflow Stuck
```

可以设计：

```text
Heartbeat
Last Progress Timestamp
Expected Next Event
Timeout
```

例如：

```text
lastProgressAt = 10:00
now = 12:00

timeout = 30m

→ STUCK
```

然后：

```text
Alert
Escalate
Cancel
Retry
```

---

# 五十五、Durable Execution 与 Security

Durable State 本身就是敏感资产。

因此必须考虑：

```text
Who can resume?
Who can cancel?
Who can inspect?
Who can modify?
Who can fork?
```

例如：

```text
Developer
 ↓
Can inspect

Release Manager
 ↓
Can approve

Security Admin
 ↓
Can cancel

System
 ↓
Can execute
```

不能让：

```text
GET /workflow/{id}/state
```

直接返回：

```text
API Keys
PII
Credentials
```

---

# 五十六、Checkpoint Mutation 应该非常谨慎

如果允许：

```text
User
 ↓
Modify Checkpoint
```

那么用户可能修改：

```text
approved = true
```

然后绕过 Approval。

所以更合理：

```text
Checkpoint
=
Immutable History
```

如果要修改：

```text
Create New Version
```

即：

```text
CP10
 ↓
Fork
 ↓
CP10'
```

而不是：

```text
UPDATE CP10
```

这样 Audit Trail 才完整。

---

# 五十七、Durable Execution 与 Time Travel

如果：

```text
CP10
```

发现 Agent 决策有问题。

可以：

```text
CP10
 ↓
Fork
 ↓
Alternative Workflow
```

例如：

```text
Original
CP10
 ↓
Deploy Production
 ↓
Failure
```

实验：

```text
CP10
 ↓
Deploy Canary
 ↓
Monitor
 ↓
Success
```

这就是：

> **What-if Execution**

对于 Agent Evaluation、Debugging 和复杂 Workflow 优化非常有价值。

---

# 五十八、Durable Execution 与 Agent Evaluation

假设一个 Agent：

```text
Workflow
 ↓
CP1
 ↓
CP2
 ↓
CP3
 ↓
CP4
```

可以保存整个执行轨迹：

```text
State
Decision
Tool Call
Tool Result
Latency
Token Usage
```

然后离线分析：

```text
为什么 CP3 决策错误？
```

或者：

```text
如果换一个 Prompt 会怎么样？
```

可以从：

```text
CP2
```

Fork：

```text
Prompt A
Prompt B
Prompt C
```

然后比较：

```text
Success Rate
Cost
Latency
Accuracy
```

因此 Durable Execution 不只是可靠性技术，也是：

> **Agent Evaluation Infrastructure**

---

# 五十九、Durable Execution 与成本控制

Agent 失败重跑可能非常昂贵。

例如：

```text
100 LLM calls
```

一次：

```text
$2
```

如果失败后全部重跑：

```text
$2 × 10 retries
=
$20
```

如果 Checkpoint：

```text
成功的 95 个 Step 不重跑
```

只需要：

```text
5 steps
```

可能只花：

```text
$0.1
```

所以：

> **Checkpoint 不仅降低故障恢复时间，也直接降低 LLM Token Cost。**

---

# 六十、Durable Execution 与 Token Economics

Agent 的成本通常来自：

```text
LLM Token
Tool Calls
External API
Compute
Storage
```

如果没有 Durable Execution：

```text
Failure
 ↓
Re-run
 ↓
LLM Token × N
```

有 Durable Execution：

```text
Failure
 ↓
Resume
 ↓
Only Retry Failed Step
```

因此：

```text
Durability
→
Reliability
→
Cost Efficiency
```

这是一个经常被忽视的商业价值。

---

# 六十一、Durable Execution 的典型反模式

## 反模式 1：把 Workflow 放在线程里

```java
while (...) {
    execute();
    Thread.sleep();
}
```

问题：

```text
Pod Restart
State Lost
```

---

## 反模式 2：把状态存在 JVM Memory

```java
Map<String, WorkflowState>
```

生产环境不可接受。

---

## 反模式 3：失败就从头开始

```text
Retry
 ↓
Start Over
```

浪费：

```text
Time
Token
Money
```

---

## 反模式 4：Tool 没有 Idempotency

最危险。

---

## 反模式 5：Workflow 中直接调用 System.currentTimeMillis()

破坏 Replay。

---

## 反模式 6：Workflow 中直接调用随机数

破坏 Determinism。

---

## 反模式 7：Checkpoint 保存 API Secret

形成新的安全漏洞。

---

## 反模式 8：允许直接修改历史 Checkpoint

破坏 Audit 和一致性。

---

## 反模式 9：无限 Retry

最终把下游服务打崩。

---

## 反模式 10：等待 Human 时一直占 Worker

严重浪费资源。

---

# 六十二、一个完整的 Durable Agent 示例

假设设计：

> “自动分析生产故障并执行修复。”

Workflow：

```text
1. Receive Incident
2. Query Logs
3. Analyze Metrics
4. Identify Root Cause
5. Generate Fix Plan
6. Approval
7. Execute Fix
8. Verify
9. Rollback if Needed
10. Generate Report
```

Durable Workflow：

```text
Incident
  ↓
CP1
  ↓
Logs
  ↓
CP2
  ↓
Metrics
  ↓
CP3
  ↓
Root Cause
  ↓
CP4
  ↓
Fix Plan
  ↓
CP5
  ↓
Approval
  ↓
WAIT
```

用户批准：

```text
APPROVED
```

恢复：

```text
CP5
 ↓
Execute Fix
 ↓
CP6
 ↓
Verify
```

如果 Worker Crash：

```text
Worker 💥
```

新 Worker：

```text
Load CP6
 ↓
Verify
```

如果 Verify 失败：

```text
Saga Compensation
 ↓
Rollback
 ↓
CP7
```

最终：

```text
Report
 ↓
CP8
 ↓
Completed
```

这就是一个真正的：

> **Durable AI Agent Workflow**

---

# 六十三、Java/Spring Boot 中的架构映射

如果采用 Java 技术栈，可以这样划分：

```text
agent-runtime
    │
    ├── workflow-engine
    │
    ├── checkpoint
    │
    ├── scheduler
    │
    ├── retry
    │
    ├── activity
    │
    ├── approval
    │
    ├── policy
    │
    ├── effect-journal
    │
    └── observability
```

基础接口：

```java
public interface DurableWorkflow {

    WorkflowResult execute(
        WorkflowContext context
    );
}
```

Activity：

```java
public interface Activity<I, O> {

    O execute(
        I input,
        ActivityContext context
    );
}
```

Checkpoint：

```java
public interface CheckpointStore {

    void save(Checkpoint checkpoint);

    Checkpoint loadLatest(String runId);
}
```

Effect：

```java
public interface EffectJournal {

    boolean exists(String effectId);

    void record(
        String effectId,
        EffectResult result
    );
}
```

Scheduler：

```java
public interface WorkflowScheduler {

    void schedule(String runId);

    void resume(String runId);

    void cancel(String runId);
}
```

最终形成：

```text
Agent Runtime
      │
      ├── Workflow Engine
      │
      ├── Scheduler
      │
      ├── Checkpoint Store
      │
      ├── Activity Executor
      │
      ├── Effect Journal
      │
      ├── Approval Engine
      │
      └── Policy Engine
```

---

# 六十四、Durable Execution 与 Spring Transaction

需要特别注意：

```text
@Transactional
```

不能解决整个 Agent Workflow 的 Durability。

因为：

```text
@Transactional
```

通常只覆盖：

```text
Database Transaction
```

而 Agent：

```text
DB
+
LLM
+
Kafka
+
HTTP
+
Human
```

可能持续数小时。

不可能：

```text
BEGIN TRANSACTION
 ↓
Wait 5 hours
 ↓
Commit
```

所以 Agent 应该采用：

```text
Short Local Transactions
+
Durable Workflow State
+
Saga
+
Outbox
+
Idempotency
```

而不是一个巨型数据库事务。

---

# 六十五、Durable Execution 与分布式事务的区别

传统：

```text
Distributed Transaction
```

希望：

```text
All or Nothing
```

Agent Workflow 更现实：

```text
Step-by-Step Progress
+
Compensation
```

因此：

```text
Transaction
=
Atomicity

Durable Workflow
=
Continuity
```

而：

```text
Saga
=
Business Compensation
```

三者解决不同问题。

---

# 六十六、Durable Execution 的核心抽象

最终可以把一个 Durable Workflow 看成：

```text
Workflow
{
    State
    History
    Steps
    Effects
    Timers
    Signals
    Checkpoints
}
```

它不是：

```text
Function
```

而是：

> **Persistent Process**

这个概念非常重要。

传统：

```text
Process
```

是：

```text
Ephemeral
```

Durable Workflow：

```text
Process
+
Persistent State
```

于是：

```text
Process Death
≠
Workflow Death
```

这是 Durable Execution 最核心的思想。

---

# 六十七、从传统程序到 Durable Agent

可以看成四次架构演进。

### 第一代：Request/Response

```text
Request
 ↓
LLM
 ↓
Response
```

---

### 第二代：Agent

```text
LLM
 ↓
Tool
 ↓
Tool
 ↓
Response
```

---

### 第三代：Stateful Agent

```text
Agent
 ↓
State
 ↓
Checkpoint
 ↓
Resume
```

---

### 第四代：Durable Agent

```text
                    Durable Agent
                          │
          ┌───────────────┼────────────────┐
          │               │                │
     Workflow State     Scheduler       Activity
          │               │                │
     Checkpoint        Timer          Side Effect
          │               │                │
          └───────────────┼────────────────┘
                          │
                     Recovery
                          │
                     Replay
                          │
                     Compensation
```

到了第四代：

> Agent 已经不再是简单的 LLM Application，而是一个真正的 Distributed Workflow Runtime。

---

# 六十八、Durable Execution、Checkpoint、Approval 三者的关系

这三个概念非常容易混淆。

可以这样理解：

```text
Checkpoint
    ↓
保存状态

Approval
    ↓
暂停并等待人类决策

Durable Execution
    ↓
保证整个 Workflow 可以持续运行
```

关系：

```text
              Durable Execution
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
     Checkpoint    Retry      Recovery
          │
          ▼
       Approval
          │
          ▼
       Resume
```

所以：

> **Checkpoint 是 Durable Execution 的状态基础，Approval 是 Durable Execution 的暂停/恢复场景之一。**

---

# 六十九、Durable Execution、Agent Communication、Approval 的统一模型

如果继续把 Agent Collaboration 加进来：

```text
                    Supervisor
                        │
               ┌────────┴────────┐
               │                 │
          Research Agent     Coding Agent
               │                 │
               └────────┬────────┘
                        │
                   Checkpoint
                        │
                    Approval
                        │
                   Deployment
                        │
                  Durable Execution
                        │
                    Verification
```

所有东西最终都可以落到：

```text
Persistent Workflow State
```

因此：

> **Checkpoint、Approval、Agent Communication、Retry、Timer、Saga，实际上都可以统一到 Durable Workflow Model。**

---

# 七十、未来的 Agent Runtime 会长什么样？

未来企业 Agent Runtime 很可能不再是：

```text
LLM + Tools
```

而是：

```text
┌───────────────────────────────────────┐
│          Agent Runtime                │
│                                       │
│  Planner                              │
│  Memory                               │
│  Tool Calling                         │
│                                       │
│  ──────────────────────────────────   │
│                                       │
│  Durable Execution                    │
│  Checkpoint                           │
│  Scheduler                            │
│  Timer                                │
│  Retry                                │
│  Recovery                             │
│  Saga                                 │
│                                       │
│  ──────────────────────────────────   │
│                                       │
│  Policy                               │
│  Authorization                        │
│  Approval                             │
│  Audit                                │
│                                       │
└───────────────────────────────────────┘
```

这实际上正在把：

```text
AI Agent
```

和：

```text
Workflow Engine
```

融合起来。

---

# 七十一、Durable Execution 的最终设计原则

如果从架构师角度总结，可以归纳成十二条原则。

### 1. State Must Survive Process Death

状态不能依赖：

```text
JVM Heap
Pod Memory
Thread
```

---

### 2. Workflow Must Have Identity

至少：

```text
workflowId
runId
```

---

### 3. Every Durable Step Must Be Identifiable

```text
stepId
```

必须稳定。

---

### 4. Side Effects Must Be Idempotent

尤其：

```text
Payment
Order
Email
Deployment
```

---

### 5. Workflow Logic Should Be Deterministic

避免：

```text
Uncontrolled Random
Current Time
External IO
```

---

### 6. External Calls Should Be Activities

```text
Workflow
 ↓
Activity
 ↓
External System
```

---

### 7. Long Wait Must Release Workers

```text
WAIT
 ↓
Persist
 ↓
Release
```

---

### 8. Retry Must Be Policy Driven

```text
Retryable?
Backoff?
Max Attempts?
```

---

### 9. Failure Must Have Recovery Strategy

```text
Retry
Compensate
Escalate
Abort
```

---

### 10. History Should Be Auditable

```text
Who
What
When
Why
Result
```

---

### 11. Checkpoint Must Support Versioning

```text
schemaVersion
workflowVersion
modelVersion
```

---

### 12. Worker Must Be Disposable

这是最重要的一条：

```text
Worker
=
Disposable Compute

Workflow
=
Durable State
```

---

# 七十二、结语：Durable Execution 才是 Agent 走向生产的真正分水岭

今天很多 Agent Demo 的核心能力是：

```text
LLM
+
Prompt
+
Tool
```

但真正的企业级 Agent 需要解决的是：

```text
如果 Agent 执行到一半崩溃怎么办？

如果等待用户批准 8 小时怎么办？

如果 Kubernetes Pod 被重启怎么办？

如果 Kafka Consumer Rebalance 怎么办？

如果外部 API 调用成功但响应丢失怎么办？

如果 LLM Retry 导致结果不同怎么办？

如果 Payment 被执行两次怎么办？

如果 Workflow 运行 30 天怎么办？

如果一个 Agent 调用另外三个 Agent 怎么办？

如果 Workflow 需要 Rollback 怎么办？
```

这些问题已经不是：

```text
Prompt Engineering
```

能够解决的。

它们属于：

```text
Distributed Systems
+
Workflow Orchestration
+
State Management
+
Fault Tolerance
+
Transaction
+
Security
+
AI Agent
```

而 Durable Execution 正好站在这些技术的交汇点上。

最终可以用一个公式总结：

```text
Reliable Agent
=
Durable State
+
Durable Steps
+
Deterministic Workflow
+
Idempotent Effects
+
Recovery
+
Retry
+
Compensation
+
Observability
```

再进一步：

```text
Production Agent
=
LLM
+
Tools
+
Memory
+
Checkpoint
+
Durable Execution
+
Policy
+
Approval
+
Idempotency
+
Observability
```

其中最关键的一次架构升级是：

```text
Agent
```

不再等同于：

```text
Process
```

而变成：

```text
Agent
=
Persistent Workflow
+
Ephemeral Workers
```

这意味着：

```text
Worker 可以死
Pod 可以重启
机器可以故障
网络可以中断
用户可以离开
LLM 可以超时
```

但是：

```text
Workflow
```

依然可以继续。

**这就是 Durable Execution 的真正价值：它把 Agent 从“运行中的程序”变成了“不会因为运行它的程序死亡而死亡的持久化执行体”。**
