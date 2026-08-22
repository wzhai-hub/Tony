---
title: Workflow 深度技术博客：从流程编排到 AI Agent Workflow Runtime
# tags:
#   - nodejs
date: '2026-08-08'
summary: Workflow 是对一组任务、状态、依赖关系、条件、异常处理和执行策略进行声明式建模，并由 Runtime 持续执行的流程模型。
---
# Workflow 深度技术博客：从流程编排到 AI Agent Workflow Runtime

> **摘要**
>
> Workflow（工作流）并不是简单的“任务 A → 任务 B → 任务 C”。在传统企业软件中，Workflow 解决的是复杂业务流程的编排、状态管理、异常处理、人工审批和长期运行问题；进入 AI Agent 时代后，Workflow 又承担了一个新的职责：**把 LLM 的非确定性推理能力放进一个确定性的执行框架中。**
>
> 本文从 Workflow 的基本模型出发，深入分析 DAG、State Machine、Task、Step、Event、Condition、Parallel、Retry、Timeout、Compensation、Persistence、Checkpoint、Event Sourcing、Durable Execution 等核心机制，并进一步讨论 Workflow 与 LLM、Agent、Tool Calling、Multi-Agent 的关系，最终构建一个生产级 Agent Workflow Runtime 的架构模型。

---

# 1. 为什么 AI Agent 时代需要 Workflow？

传统程序通常是：

```text
Input
  ↓
Function A
  ↓
Function B
  ↓
Function C
  ↓
Output
```

代码直接表达执行流程：

```java
resultA = serviceA.execute();

resultB = serviceB.execute(resultA);

resultC = serviceC.execute(resultB);

return resultC;
```

这种方式非常适合：

* 短事务
* 同步执行
* 状态简单
* 生命周期短

但现实企业系统经常是：

```text
创建订单
    ↓
支付
    ↓
库存
    ↓
物流
    ↓
人工审核
    ↓
通知
```

其中任何一步都可能：

```text
失败
超时
重试
等待
暂停
恢复
人工介入
补偿
```

于是流程不再只是：

```text
A → B → C
```

而是：

```text
               ┌── Retry ──┐
               │           │
               ▼           │
A → B → C → Failure        │
        │       │           │
        │       └───────────┘
        │
        ├── Human Approval
        │
        └── Compensation
```

这就是 Workflow 要解决的问题。

---

# 2. Workflow 到底是什么？

可以把 Workflow 定义为：

> **Workflow 是对一组任务、状态、依赖关系、条件、异常处理和执行策略进行声明式建模，并由 Runtime 持续执行的流程模型。**

可以抽象为：

[
Workflow = (Tasks, Dependencies, Conditions, Events, Policies, State)
]

其中：

```text
Tasks
    任务

Dependencies
    依赖关系

Conditions
    条件

Events
    事件

Policies
    Retry / Timeout / Compensation

State
    当前执行状态
```

所以 Workflow 本质上不是一个“流程图”。

它实际上是：

> **一个可执行的计算模型。**

---

# 3. Workflow 和普通代码有什么区别？

这是理解 Workflow 的第一步。

普通代码：

```java
void process() {

    stepA();

    stepB();

    stepC();
}
```

Workflow：

```text
Workflow
 ├── Step A
 ├── Step B
 └── Step C
```

表面看起来没有区别。

真正的区别在于：

> **Workflow 把控制流从代码中提升成了可持久化、可观测、可恢复的数据模型。**

例如普通 Java：

```java
stepA();
stepB();
stepC();
```

如果 JVM 在 `stepB()` 执行过程中挂掉：

```text
JVM Crash
    ↓
Process Lost
```

系统不知道：

```text
Step A 是否完成？
Step B 是否执行了一半？
Step C 是否执行？
```

Workflow Runtime 则可以保存：

```text
Workflow ID
Current Step
Step Status
Input
Output
Retry Count
Checkpoint
Execution History
```

因此：

```text
普通代码
    = execution logic

Workflow
    = execution logic + lifecycle + persistence
```

---

# 4. Workflow 的核心组成

一个成熟 Workflow 通常包含：

```text
Workflow
│
├── Task
├── Step
├── State
├── Dependency
├── Condition
├── Event
├── Retry
├── Timeout
├── Parallelism
├── Compensation
├── Persistence
├── Checkpoint
└── Human Interaction
```

可以进一步划分为四层：

```text
┌─────────────────────────────┐
│      Workflow Definition    │
├─────────────────────────────┤
│      Execution Engine       │
├─────────────────────────────┤
│      State / Persistence    │
├─────────────────────────────┤
│      Infrastructure         │
└─────────────────────────────┘
```

---

# 5. Workflow Definition：描述“应该怎么执行”

Workflow Definition 是流程本身。

例如：

```text
A → B → C
```

或者：

```text
        ┌→ B ─┐
A ──────┤     ├→ D
        └→ C ─┘
```

甚至：

```text
A
│
├── B
│
├── C
│
└── D
     │
     ▼
     E
```

Definition 关注：

> **What should happen?**

而 Runtime 关注：

> **How is it executing now?**

这是 Workflow 系统非常重要的分层。

---

# 6. Workflow Runtime

Workflow Runtime 是整个系统的执行引擎。

例如：

```text
Workflow Definition
        │
        ▼
┌──────────────────────┐
│  Workflow Runtime    │
│                      │
│ Scheduler            │
│ Executor             │
│ State Manager        │
│ Retry Manager        │
│ Timer Manager        │
│ Event Manager        │
│ Recovery Manager     │
└──────────┬───────────┘
           │
           ▼
       Workers
```

Runtime 负责：

```text
加载 Workflow
 ↓
读取当前状态
 ↓
寻找 Ready Task
 ↓
调度 Task
 ↓
执行 Task
 ↓
保存结果
 ↓
推进 Workflow
```

因此 Workflow Runtime 可以理解成：

> **Workflow 的操作系统。**

---

# 7. Workflow 最核心的抽象：DAG

很多 Workflow 都可以表示为 DAG：

> Directed Acyclic Graph

即有向无环图。

例如：

```text
A
│
├──── B
│
├──── C
│
└──── D
       │
       ▼
       E
```

表示：

```text
A → B
A → C
A → D

B → E
C → E
D → E
```

如果：

```text
B
C
D
```

互不依赖，那么它们可以并行。

这就是 Workflow 最重要的性能优化之一。

---

# 8. DAG 为什么重要？

假设：

```text
A = 2s
B = 3s
C = 4s
D = 2s
```

串行执行：

[
T = 2 + 3 + 4 + 2 = 11s
]

如果：

```text
A
│
├── B
├── C
└── D
```

那么：

[
T = 2 + max(3,4,2)
]

即：

[
T = 6s
]

所以 Workflow Engine 的一个重要职责就是：

> **发现任务之间的依赖关系，并最大化可并行执行的任务。**

---

# 9. Task Dependency

一个 Task 只有满足所有依赖条件之后才能执行。

例如：

```text
A
│
├── B
│
└── C
     │
     ▼
     D
```

D 的依赖：

```text
B
C
```

因此：

```text
B = SUCCESS
C = SUCCESS
```

之后：

```text
D = READY
```

可以定义：

```java
boolean ready(Task task) {

    return task.dependencies()
        .stream()
        .allMatch(this::isCompleted);
}
```

这其实就是一个拓扑排序问题。

---

# 10. Topological Sort

如果 Workflow 是 DAG：

```text
A → B → D
A → C → D
```

拓扑顺序可能是：

```text
A
B
C
D
```

Workflow Runtime 可以使用：

```text
Kahn Algorithm
```

计算：

```text
in-degree
```

例如：

```text
A: 0
B: 1
C: 1
D: 2
```

首先：

```text
A
```

执行完成：

```text
B: 0
C: 0
```

于是：

```text
B
C
```

成为 Ready Tasks。

最后：

```text
D
```

Ready。

因此：

> **Workflow Scheduler 的核心之一就是 DAG Scheduling。**

---

# 11. Workflow 不等于 DAG

这是一个非常重要的区别。

DAG 擅长：

```text
A → B → C
```

但是很多 Workflow 有：

```text
Loop
Retry
Human Wait
Timer
Event
Compensation
```

例如：

```text
A
 ↓
B
 ↓
C
 ↓
Failure
 ↓
Retry
 ↓
B
```

这已经不是简单 DAG。

因此更准确地说：

```text
DAG
    = 一种 Workflow Execution Model

Workflow
    = 更大的流程抽象
```

复杂 Workflow 通常需要：

```text
DAG
+
State Machine
+
Event Model
+
Persistence
```

---

# 12. Workflow 与 State Machine 的关系

这是前一篇 State Machine 文章之后非常自然的延伸。

State Machine：

```text
State
 ↓
Event
 ↓
Transition
 ↓
State
```

Workflow：

```text
Task
 ↓
Dependency
 ↓
Execution
 ↓
Next Task
```

两者可以结合：

```text
Workflow
   │
   ├── Task A
   │     └── State Machine
   │
   ├── Task B
   │     └── State Machine
   │
   └── Task C
         └── State Machine
```

或者：

```text
Workflow State
      │
      ▼
Task State
      │
      ▼
Execution State
```

因此可以认为：

> **Workflow 描述宏观流程，State Machine 描述生命周期。**

---

# 13. Workflow Task 的生命周期

一个 Task 不应该只有：

```text
RUNNING
SUCCESS
FAILED
```

生产系统至少需要：

```text
PENDING
READY
SCHEDULED
RUNNING
WAITING
SUCCESS
FAILED
RETRYING
CANCELLED
TIMEOUT
```

例如：

```text
PENDING
   ↓
READY
   ↓
SCHEDULED
   ↓
RUNNING
   │
   ├── SUCCESS
   │
   ├── FAILED
   │
   └── TIMEOUT
           │
           ▼
        RETRYING
```

这就是 Task State Machine。

---

# 14. Workflow 的执行模型

一个典型 Workflow Runtime 可以运行：

```text
while workflow.notFinished():

    readyTasks = scheduler.findReadyTasks();

    for task : readyTasks:

        executor.submit(task);
```

Task 完成之后：

```text
Task Result
    ↓
State Store
    ↓
Dependency Resolver
    ↓
New Ready Tasks
    ↓
Scheduler
```

形成一个闭环：

```text
State
 ↓
Scheduler
 ↓
Executor
 ↓
Worker
 ↓
Result
 ↓
State
```

这就是 Workflow Engine 的基本心跳。

---

# 15. Worker 和 Scheduler 必须分离

一个成熟系统不要：

```text
Scheduler
   ↓
直接执行任务
```

更合理：

```text
Scheduler
   ↓
Task Queue
   ↓
Worker
```

例如：

```text
              Scheduler
                  │
                  ▼
               Kafka
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
     Worker1   Worker2   Worker3
```

好处：

* 水平扩展
* 任务削峰
* 故障隔离
* Worker 独立扩容
* 异步执行

---

# 16. Workflow Worker

Worker 的职责应该非常明确：

```text
Receive Task
    ↓
Load Context
    ↓
Execute
    ↓
Report Result
```

Worker 不应该决定整个 Workflow：

```text
Worker
   ❌ 修改 Workflow 全局逻辑
```

而应该：

```text
Worker
   ↓
Task Result
   ↓
Workflow Runtime
   ↓
Next State
```

这是一种非常重要的职责边界。

---

# 17. Workflow Persistence

Workflow 最大的优势之一：

> **长时间运行。**

例如：

```text
提交申请
 ↓
等待人工审批
 ↓
等待 3 天
 ↓
继续执行
```

因此不能依赖：

```text
JVM Thread
```

也不能依赖：

```java
Thread.sleep(3 * 24 * 3600 * 1000);
```

正确设计应该是：

```text
WAITING_APPROVAL
      │
      ▼
Persistent State
      │
      ▼
Application can shutdown
      │
      │
      │ 3 days later
      ▼
APPROVED Event
      │
      ▼
Resume Workflow
```

这就是：

> **Durable Workflow**

---

# 18. Durable Execution

Durable Execution 的核心思想：

> **Workflow 的执行状态应该独立于具体 Worker 进程。**

例如：

```text
Worker A
   ↓
Task A
   ↓
Persist
   ↓
Worker Crash
```

之后：

```text
Worker B
   ↓
Load State
   ↓
Resume Task B
```

因此：

```text
Workflow
    ≠
Process
```

这是 Workflow 与普通程序最大的区别之一。

---

# 19. Checkpoint

Workflow 需要不断保存 Checkpoint。

例如：

```text
Workflow ID: W1001

Completed:
A
B
C

Running:
D

Pending:
E
F
```

Checkpoint：

```json
{
  "workflowId": "W1001",
  "completed": ["A", "B", "C"],
  "running": ["D"],
  "pending": ["E", "F"]
}
```

如果 Runtime 重启：

```text
Load Checkpoint
      ↓
Recover
      ↓
Continue
```

而不是：

```text
Start From Beginning
```

---

# 20. Event Sourcing

Workflow 也非常适合 Event Sourcing。

例如：

```text
WORKFLOW_STARTED
TASK_A_STARTED
TASK_A_COMPLETED
TASK_B_STARTED
TASK_B_COMPLETED
TASK_C_FAILED
TASK_C_RETRY
TASK_C_COMPLETED
WORKFLOW_COMPLETED
```

通过 Event Replay 可以重新构造 Workflow 状态。

优势：

```text
Audit
Debug
Replay
Recovery
History
```

尤其是 AI Workflow：

> 为什么 Agent 最终得出这个结果？

可以通过 Event History 还原。

---

# 21. Retry 是 Workflow 的基础能力

生产 Workflow 几乎必然需要 Retry。

例如：

```text
Task A
   ↓
FAILED
   ↓
RETRY
   ↓
FAILED
   ↓
RETRY
   ↓
SUCCESS
```

但是 Retry 不能简单：

```java
for (int i = 0; i < 3; i++) {
    execute();
}
```

因为真正的 Workflow Retry 需要：

```text
Retry Count
Backoff
Jitter
Retry Policy
Failure Type
Maximum Attempts
Timeout
```

---

# 22. Exponential Backoff

经典策略：

[
delay = min(maxDelay,\ baseDelay \times 2^n)
]

例如：

```text
Attempt 1 → 1s
Attempt 2 → 2s
Attempt 3 → 4s
Attempt 4 → 8s
```

再加入随机 Jitter：

[
delay = base \times 2^n + random()
]

避免大量 Worker 同时 Retry：

```text
Thundering Herd
```

---

# 23. Retry 不是万能的

Workflow Engine 必须区分：

```text
Transient Failure
Permanent Failure
```

Transient：

```text
Network Timeout
503
Connection Reset
Rate Limit
```

可以 Retry。

Permanent：

```text
Invalid Parameter
Permission Denied
Business Rule Violation
```

通常不应该 Retry。

所以：

```text
Failure
  │
  ├── Retryable
  │       ↓
  │    RETRY
  │
  └── Non-Retryable
          ↓
        FAILED
```

这是一项非常重要的 Workflow Engineering 能力。

---

# 24. Timeout

每个 Task 都应该拥有：

```text
executionTimeout
scheduleTimeout
workflowTimeout
```

例如：

```text
Task Timeout = 30s
Workflow Timeout = 1h
```

如果：

```text
Task Running > 30s
```

触发：

```text
TASK_TIMEOUT
```

然后由 Workflow Policy 决定：

```text
Retry
Fail
Compensate
Escalate
```

---

# 25. Cancellation

用户可能随时：

```text
Cancel Workflow
```

但是：

```text
Cancel
```

并不意味着：

```text
Kill Process
```

需要考虑：

```text
Current Task
Running Tool
External Side Effect
Child Workflow
```

例如：

```text
Workflow
   ↓
Payment
   ↓
Cancel
```

如果 Payment 已经成功：

```text
Cancel
```

不能简单地让 Workflow 消失。

可能需要：

```text
COMPENSATING
```

因此：

> Cancellation 本身也是 Workflow State Transition。

---

# 26. Compensation

Workflow 经常无法真正 Rollback。

例如：

```text
Create User
Charge Payment
Send Email
```

数据库 Transaction 可以：

```text
ROLLBACK
```

但：

```text
Email
Payment
External API
```

无法简单 rollback。

因此使用：

> Compensation

例如：

```text
Charge Payment
    ↓
Compensation:
Refund Payment
```

Workflow：

```text
A
 ↓
B
 ↓
C
 ↓
Failure
 ↓
Compensate C
 ↓
Compensate B
 ↓
Compensate A
```

这就是 Saga Pattern。

---

# 27. Workflow + Saga

可以将 Workflow 看成：

```text
Forward Execution
```

Saga 提供：

```text
Backward Compensation
```

整体：

```text
          Forward
A ──→ B ──→ C ──→ D
                 │
                 │ Failure
                 ▼
          Compensation
                 │
             Compensate C
                 │
             Compensate B
                 │
             Compensate A
```

这是分布式长事务的经典解决方案。

---

# 28. Workflow 中的 Human-in-the-loop

企业 Workflow 经常需要人工参与。

例如：

```text
Agent Analysis
      ↓
Risk Evaluation
      ↓
Human Approval
      ↓
Execute
```

Workflow Runtime 进入：

```text
WAITING_HUMAN
```

然后释放 Worker。

这是非常重要的设计：

```text
Worker
   ❌ 一直等待
```

而应该：

```text
Persist State
   ↓
Release Worker
```

等事件回来：

```text
HUMAN_APPROVED
   ↓
Resume Workflow
```

这样可以支持：

```text
等待几分钟
等待几小时
等待几天
甚至等待几个月
```

---

# 29. Workflow 中的 Condition

Workflow 不一定是固定路径。

例如：

```text
                 Validate
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
      amount < 1000       amount >= 1000
          │                   │
          ▼                   ▼
     Auto Approve        Human Review
```

Condition：

```java
if (amount < 1000) {
    autoApprove();
} else {
    humanReview();
}
```

在 Workflow Definition 中则应该变成：

```text
Condition
   ├── true  → AutoApprove
   └── false → HumanReview
```

这使流程变成：

> **Declarative**

而不是：

> **Hard-coded**

---

# 30. Workflow 的声明式设计

命令式：

```java
if (...) {
    stepA();
} else {
    stepB();
}
```

声明式：

```yaml
workflow:
  steps:
    - validate

  branches:
    - condition: amount < 1000
      next: autoApprove

    - condition: amount >= 1000
      next: humanReview
```

声明式 Workflow 的优势：

```text
可视化
可修改
可版本化
可分析
可验证
可审计
```

---

# 31. Workflow Definition Versioning

这是生产系统中经常被忽略的问题。

假设：

```text
Workflow v1
```

已经运行：

```text
10,000 executions
```

此时修改 Workflow：

```text
Workflow v2
```

问题来了：

> 已经运行中的 Workflow 到底应该使用 v1 还是 v2？

正确答案通常是：

```text
Execution → Definition Version
```

例如：

```text
W1001 → v1
W1002 → v1
W1003 → v2
W1004 → v2
```

不要直接：

```text
UPDATE workflow_definition
```

然后让所有实例自动改变。

---

# 32. Workflow Migration

如果必须迁移：

```text
v1
 ↓
v2
```

需要考虑：

```text
State Mapping
Task Mapping
Context Migration
History Compatibility
```

例如：

```text
v1:
A → B → C

v2:
A → B → D → C
```

已经运行到：

```text
B
```

应该怎么办？

需要定义：

```text
Migration Rule
```

否则 Workflow 升级会成为灾难。

---

# 33. Workflow Idempotency

Workflow 天然会遇到：

```text
Retry
Duplicate Message
Worker Crash
Network Timeout
```

例如：

```text
Worker
   ↓
Create Payment
   ↓
Payment Success
   ↓
Worker Timeout
```

Worker 以为失败：

```text
Retry
```

于是：

```text
Create Payment
```

执行第二次。

如果没有幂等：

```text
Double Payment
```

因此 Workflow Task 必须尽可能支持：

```text
Idempotency Key
```

例如：

```text
workflowId + taskId + attempt
```

或者：

```text
businessRequestId
```

---

# 34. Exactly Once 是一个危险的幻想

分布式 Workflow 中经常有人说：

> “我要 Exactly Once Execution。”

现实中很难做到真正的：

```text
Exactly Once
```

更常见的是：

```text
At Least Once Delivery
+
Idempotent Execution
```

即：

```text
Message
   ↓
At Least Once
   ↓
Worker
   ↓
Idempotency
   ↓
Effectively Once
```

这是更加现实的工程模型。

---

# 35. Workflow Queue

Workflow Scheduler 通常不会直接调用 Worker。

中间需要：

```text
Queue
```

例如：

```text
Workflow Engine
      ↓
Task Queue
      ↓
Worker
```

Queue 可以提供：

```text
Load Balancing
Backpressure
Retry
Dead Letter
Partition
Ordering
```

Kafka 非常适合：

```text
Event Stream
```

Redis Streams、RabbitMQ 等也可以用于 Task Queue。

---

# 36. Backpressure

假设：

```text
Workflow:
100,000 tasks/s
```

但是 Worker：

```text
10,000 tasks/s
```

如果 Scheduler 不控制：

```text
Task Queue
    ↓
无限增长
```

最终：

```text
Memory
CPU
Network
```

全部被压垮。

因此 Workflow Runtime 需要：

```text
Concurrency Limit
Rate Limit
Queue Limit
Admission Control
Backpressure
```

例如：

```text
maxConcurrentTasks = 1000
```

只有：

```text
runningTasks < 1000
```

才继续调度。

---

# 37. Workflow 与分布式系统

Workflow Engine 本质上是一个分布式系统。

它需要解决：

```text
Concurrency
Consistency
Durability
Failure
Ordering
Partition
Recovery
Leader Election
```

因此 Workflow Engine 并不是：

> 一个简单的流程图执行器。

而更接近：

> **Distributed Execution Platform**

---

# 38. Scheduler 的竞争问题

假设：

```text
Scheduler A
Scheduler B
```

同时看到：

```text
Task T = READY
```

两者都调度：

```text
Worker A → T
Worker B → T
```

于是出现：

```text
Duplicate Execution
```

可以使用：

```text
Distributed Lock
```

或者：

```text
Compare-And-Set
```

例如：

```sql
UPDATE task
SET status = 'SCHEDULED'
WHERE task_id = ?
AND status = 'READY';
```

只有：

```text
affectedRows = 1
```

的 Scheduler 获得执行权。

---

# 39. Leader Election

另一种架构：

```text
Scheduler
   │
   ├── Leader
   ├── Follower
   └── Follower
```

只有 Leader 负责：

```text
Scheduling
```

Follower：

```text
Standby
```

Leader Crash：

```text
Follower
   ↓
Leader Election
   ↓
New Leader
```

但需要注意：

> Leader Election 不是解决所有一致性问题的万能方案。

最终仍然需要：

```text
Idempotency
Version
CAS
Persistence
```

---

# 40. Workflow 与 AI Agent 的结合

现在进入最重要的部分。

传统 Workflow：

```text
Step A
 ↓
Step B
 ↓
Step C
```

Agent Workflow：

```text
Analyze
 ↓
LLM Decision
 ↓
Tool Selection
 ↓
Tool Execution
 ↓
Observation
 ↓
LLM Reasoning
 ↓
Next Action
```

关键变化：

> **Workflow 中出现了概率性决策节点。**

因此：

```text
Traditional Workflow
    = deterministic

Agent Workflow
    = deterministic workflow
      +
probabilistic decision
```

---

# 41. LLM 应该成为 Workflow Task，而不是 Workflow Runtime

一个常见错误：

```text
LLM
 ↓
决定整个 Workflow
```

例如：

```text
LLM:
我认为下一步应该是：
A → B → C → D
```

问题：

```text
LLM Hallucination
Invalid Tool
Illegal Transition
Missing Step
Unsafe Action
```

更好的设计：

```text
Workflow Runtime
      │
      ▼
LLM Task
      │
      ▼
Structured Decision
      │
      ▼
Policy Validation
      │
      ▼
Workflow Transition
```

即：

> LLM 是 Workflow 中的一个智能节点，而不是 Workflow Engine 本身。

---

# 42. Agent Workflow 的经典模型

可以设计：

```text
                 ┌─────────────┐
                 │   START     │
                 └──────┬──────┘
                        ▼
                 ┌─────────────┐
                 │   PLANNER   │
                 │     LLM     │
                 └──────┬──────┘
                        ▼
                 ┌─────────────┐
                 │  VALIDATOR  │
                 └──────┬──────┘
                        ▼
                 ┌─────────────┐
                 │  EXECUTOR   │
                 └──────┬──────┘
                        ▼
                 ┌─────────────┐
                 │  OBSERVER   │
                 └──────┬──────┘
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
          Continue              Finish
              │
              └──────→ PLANNER
```

这其实已经是：

> **Agentic Workflow Loop**

---

# 43. Workflow 与 Agent Loop 的关系

Agent Loop：

```text
Think
 ↓
Act
 ↓
Observe
 ↓
Think
```

Workflow：

```text
Task
 ↓
Dependency
 ↓
Task
```

二者结合：

```text
Workflow
   │
   └── Agent Task
           │
           ├── Think
           ├── Act
           └── Observe
```

因此：

> **Workflow 提供宏观结构，Agent Loop 提供局部智能。**

这是非常值得掌握的架构思想。

---

# 44. Agent Workflow 中的 Human Approval

例如：

```text
User Request
      ↓
Agent Planning
      ↓
Risk Analysis
      ↓
Human Approval
      ↓
Tool Execution
      ↓
Result
```

Workflow：

```text
START
 ↓
PLAN
 ↓
ANALYZE_RISK
 ↓
WAITING_APPROVAL
 ↓
APPROVED
 ↓
EXECUTE
 ↓
VERIFY
 ↓
COMPLETED
```

这种设计特别适合：

```text
金融
医疗
企业审批
生产运维
代码发布
安全操作
```

---

# 45. Multi-Agent Workflow

进一步：

```text
                    Orchestrator
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      Research        Coding         Review
       Agent           Agent          Agent
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                     Synthesis
                         │
                         ▼
                      Final
```

Workflow Engine 负责：

```text
Dependency
Scheduling
Retry
Timeout
State
Persistence
```

Agent 负责：

```text
Reasoning
Planning
Tool Selection
Natural Language
```

这是比较成熟的 Multi-Agent Architecture。

---

# 46. Workflow Orchestration vs Choreography

这是分布式 Workflow 非常重要的两个模型。

## Orchestration

中心 Workflow Engine：

```text
             Orchestrator
             /     |     \
            A      B      C
```

Orchestrator 决定：

```text
谁执行
什么时候执行
执行什么
失败怎么办
```

优点：

```text
流程清晰
可观测
易审计
易管理
```

缺点：

```text
中心化
Orchestrator 复杂度高
```

---

## Choreography

没有中央 Controller：

```text
A
 ↓ Event
B
 ↓ Event
C
 ↓ Event
D
```

每个服务根据 Event 自己决定行为。

优点：

```text
松耦合
```

缺点：

```text
流程难以理解
Debug 困难
全局状态困难
```

---

# 47. Agent Workflow 更适合哪一种？

对于复杂企业 Agent：

通常推荐：

```text
Orchestration
```

例如：

```text
Workflow Engine
        │
        ├── Research Agent
        ├── Analysis Agent
        ├── Coding Agent
        └── Review Agent
```

原因是：

Agent 本身已经具有不确定性。

如果再让整个系统使用完全去中心化的 Choreography：

```text
Agent A
 ↓
Agent B
 ↓
Agent C
 ↓
Agent D
```

系统复杂度会迅速上升。

因此：

> **Agent 可以是自治的，但 Workflow 应该尽可能保持可控。**

---

# 48. Workflow 与 Policy Engine

企业 Agent Workflow 还需要 Policy。

例如：

```text
if amount > 10000:
    requireHumanApproval()
```

或者：

```text
if tool == delete_account:
    requireHumanApproval()
```

于是：

```text
Workflow
    ↓
Policy Engine
    ↓
Allow / Deny / Require Approval
```

这形成：

```text
LLM
 ↓
Workflow
 ↓
Policy
 ↓
Tool
```

而不是：

```text
LLM
 ↓
Tool
```

---

# 49. Workflow 的安全模型

一个成熟 Workflow Runtime 至少需要：

```text
Authentication
Authorization
Tool Permission
State Permission
Data Permission
Tenant Isolation
Audit
```

可以定义：

```text
State = PLANNING
Allowed Tools:
    search
    retrieve

State = APPROVED
Allowed Tools:
    create_order
    charge_payment
```

于是：

> Workflow State 可以参与 Authorization。

这是 Agent 安全架构非常重要的一层。

---

# 50. Workflow Observability

生产 Workflow 必须支持：

```text
Workflow Metrics
Task Metrics
Queue Metrics
Worker Metrics
LLM Metrics
Tool Metrics
State Metrics
```

例如：

```text
workflow_execution_total
workflow_execution_duration
task_execution_duration
task_failure_total
task_retry_total
workflow_active
queue_depth
worker_utilization
```

AI Workflow 还可以增加：

```text
llm_latency
llm_token_usage
llm_cost
tool_call_count
agent_retry
agent_handoff
human_approval_time
```

---

# 51. Workflow Trace

一个完整 Trace：

```text
Workflow: W1001

 ├── PLAN
 │    └── LLM
 │
 ├── RESEARCH
 │    ├── Search
 │    └── Database
 │
 ├── ANALYSIS
 │    └── LLM
 │
 ├── HUMAN_APPROVAL
 │
 ├── EXECUTION
 │    ├── Tool A
 │    └── Tool B
 │
 └── COMPLETED
```

可以使用：

```text
OpenTelemetry
```

将：

```text
Workflow
Task
Agent
LLM
Tool
Database
```

连接到同一条 Trace。

---

# 52. Workflow Cost Control

AI Workflow 有一个传统 Workflow 没有的问题：

> Token Cost。

例如：

```text
Workflow
 ├── Agent A → 5K tokens
 ├── Agent B → 10K tokens
 ├── Agent C → 8K tokens
 └── Reviewer → 12K tokens
```

总成本可能快速增长。

因此 Workflow Runtime 可以加入：

```text
Token Budget
Cost Budget
Time Budget
Tool Budget
```

例如：

```text
maxTokens = 100000
maxCost = $1
maxDuration = 10min
maxToolCalls = 50
```

超过预算：

```text
BUDGET_EXCEEDED
```

然后：

```text
STOP
```

或者：

```text
HUMAN_APPROVAL
```

---

# 53. Workflow 的资源治理

一个企业级 Agent Workflow 需要同时管理：

```text
CPU
Memory
LLM Tokens
Tool Calls
API Rate
Database
Queue
```

所以 Workflow Runtime 可以进一步成为：

> **Resource Governance Layer**

例如：

```text
Tenant A
    max 100 concurrent workflows

Tenant B
    max 50 concurrent workflows

Agent X
    max 10 LLM calls/min

Tool Y
    max 100 calls/min
```

这和传统微服务限流、配额管理可以很好地结合。

---

# 54. Workflow 的版本化

一个成熟 Workflow Definition 应该像代码一样：

```text
workflow-v1
workflow-v2
workflow-v3
```

支持：

```text
Git
Code Review
CI/CD
Testing
Deployment
Rollback
```

例如：

```text
Production
   │
   ├── v1 → 60%
   └── v2 → 40%
```

甚至可以：

```text
A/B Testing
Canary
Blue/Green
```

这使 Workflow 本身进入：

> **Software Delivery Lifecycle**

---

# 55. Workflow DSL

复杂 Workflow 可以设计 DSL：

```yaml
workflow:
  name: investment-analysis

  steps:

    - id: research
      agent: research-agent

    - id: financial-analysis
      agent: finance-agent
      dependsOn:
        - research

    - id: risk-analysis
      agent: risk-agent
      dependsOn:
        - research

    - id: review
      agent: reviewer
      dependsOn:
        - financial-analysis
        - risk-analysis

    - id: human-approval
      type: human
      dependsOn:
        - review
```

Runtime 读取：

```text
YAML
 ↓
Workflow Definition
 ↓
DAG
 ↓
Scheduler
 ↓
Execution
```

这样 Workflow 就真正成为：

> **可配置、可版本化的执行模型。**

---

# 56. Workflow DSL 的本质

DSL 并不是为了让 YAML 看起来漂亮。

它真正的价值是：

```text
Workflow
 ↓
AST / Graph
 ↓
Validation
 ↓
Optimization
 ↓
Execution
```

例如 Runtime 可以在执行前发现：

```text
Cycle
Missing Dependency
Invalid State
Unknown Task
Duplicate Task ID
Invalid Condition
```

甚至可以：

```text
Static Analysis
```

提前发现错误。

---

# 57. Workflow 编译器

更进一步，可以把 Workflow 看成一种 Programming Language。

例如：

```text
Workflow DSL
      ↓
Parser
      ↓
AST
      ↓
Validator
      ↓
Execution Plan
      ↓
Runtime
```

这与：

```text
Source Code
 ↓
Compiler
 ↓
AST
 ↓
IR
 ↓
Machine Code
```

非常类似。

因此：

> **高级 Workflow Engine 本质上正在逐渐演化成一种流程编程语言 Runtime。**

---

# 58. Workflow Optimization

既然 Workflow 是 Graph，就可以进行优化。

例如：

```text
A → B
A → C
```

可以并行。

又例如：

```text
A → B → C
```

如果：

```text
B
```

只是简单数据转换，可以进行：

```text
Task Fusion
```

即：

```text
B + C
```

合并。

还可以进行：

```text
Parallel Scheduling
Resource-aware Scheduling
Priority Scheduling
Critical Path Optimization
```

---

# 59. Critical Path

Workflow 的总执行时间通常取决于：

> Critical Path

例如：

```text
A(2s)
│
├── B(10s)
│     │
│     ▼
│     D(5s)
│
└── C(3s)
      │
      ▼
      D
```

关键路径：

```text
A → B → D
```

时间：

[
2 + 10 + 5 = 17s
]

C 即使从：

```text
3s → 1s
```

整个 Workflow 也不会明显改善。

所以 Workflow Optimization 应该优先：

> **优化 Critical Path。**

---

# 60. Agent Workflow 的 Critical Path

AI Workflow 特别容易出现：

```text
LLM
 ↓
Tool
 ↓
LLM
 ↓
Tool
 ↓
LLM
```

如果所有调用都是串行：

```text
Latency
=
LLM1
+
Tool1
+
LLM2
+
Tool2
+
LLM3
```

因此可以尽可能：

```text
                 ┌── Tool A
LLM Decision ────┼── Tool B
                 └── Tool C
```

然后：

```text
Results
   ↓
LLM Synthesis
```

这样可以大幅降低：

```text
Latency
```

---

# 61. Workflow 与 Parallelism

需要注意：

> 并行不是越多越好。

如果同时执行：

```text
1000 LLM Calls
```

可能导致：

```text
Rate Limit
Cost Explosion
Database Pressure
API Overload
```

因此需要：

```text
Concurrency Control
```

例如：

```text
Workflow concurrency = 100
Agent concurrency = 20
Tool concurrency = 10
```

形成层级资源控制。

---

# 62. Workflow 的故障模型

生产 Workflow 必须假设：

```text
Worker Crash
Scheduler Crash
Network Partition
Database Failure
Queue Failure
LLM Timeout
Tool Timeout
Duplicate Message
Out-of-order Event
```

因此设计目标不是：

> “系统永远不失败。”

而是：

> **“系统失败之后能够继续正确运行。”**

这就是：

> Fault-tolerant Workflow。

---

# 63. Workflow Recovery

一个完整 Recovery：

```text
Worker Crash
    ↓
Task Lease Expired
    ↓
Task Recovered
    ↓
Retry
    ↓
Worker B
    ↓
Continue
```

这里通常需要：

```text
Lease
Heartbeat
Timeout
Retry
Idempotency
```

例如 Worker 获得：

```text
Lease = 30s
```

每隔：

```text
10s
```

发送 Heartbeat。

如果：

```text
Heartbeat Lost
```

Scheduler 可以认为：

```text
Worker Dead
```

然后重新调度 Task。

---

# 64. Workflow Lease

数据库可以设计：

```text
task_id
status
worker_id
lease_until
version
```

Worker 获取任务：

```sql
UPDATE task
SET worker_id = ?,
    lease_until = NOW() + INTERVAL '30 seconds',
    status = 'RUNNING'
WHERE task_id = ?
AND status = 'READY';
```

Worker 定期：

```text
Heartbeat
```

更新：

```text
lease_until
```

如果过期：

```text
RUNNING
   ↓
RECOVERABLE
```

重新进入：

```text
READY
```

这是分布式 Worker 系统非常常见的模式。

---

# 65. Workflow 与 Exactly Once 的现实设计

一个更加现实的执行语义：

```text
Workflow
    ↓
At Least Once
    ↓
Idempotent Task
    ↓
Durable State
    ↓
Effectively Once
```

而不是试图做到：

```text
Exactly Once
```

这与 Kafka、消息队列、分布式任务系统的工程思想高度一致。

---

# 66. Workflow Engine 的核心模块

一个生产级 Workflow Engine 可以拆成：

```text
┌──────────────────────────────────────────┐
│              Workflow API               │
├──────────────────────────────────────────┤
│          Workflow Definition             │
├──────────────────────────────────────────┤
│          Workflow Validator              │
├──────────────────────────────────────────┤
│              Scheduler                  │
├──────────────────────────────────────────┤
│            Task Dispatcher              │
├──────────────────────────────────────────┤
│             State Manager               │
├──────────────────────────────────────────┤
│             Retry Manager               │
├──────────────────────────────────────────┤
│              Timer Service              │
├──────────────────────────────────────────┤
│          Recovery Manager               │
├──────────────────────────────────────────┤
│             Event Bus                   │
├──────────────────────────────────────────┤
│            Persistence                  │
├──────────────────────────────────────────┤
│           Observability                 │
└──────────────────────────────────────────┘
```

这已经不是一个简单的 Spring Boot Service。

它本质上是：

> **一个分布式执行平台。**

---

# 67. Workflow Engine 的数据模型

核心表可以设计：

```text
workflow_instance
```

保存：

```text
workflow_id
definition_version
status
context
created_at
updated_at
```

Task：

```text
workflow_task
```

保存：

```text
task_id
workflow_id
task_type
status
attempt
worker_id
lease_until
input
output
error
```

Event：

```text
workflow_event
```

保存：

```text
event_id
workflow_id
task_id
event_type
payload
timestamp
```

这样：

```text
Workflow
Task
Event
```

形成完整生命周期模型。

---

# 68. Workflow + Redis + Kafka + PostgreSQL

如果使用你比较熟悉的技术栈，可以设计：

```text
                 API
                  │
                  ▼
          Workflow Engine
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
   PostgreSQL   Redis       Kafka
       │          │           │
 Definition     Lock       Event Bus
 State          Cache      Task Queue
 History        Lease
                  │
                  ▼
                Worker
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
     Tool        LLM       Database
```

职责可以明确分离：

### PostgreSQL

```text
Source of Truth
```

保存：

```text
Workflow
Task
State
Execution History
```

### Redis

```text
Fast Coordination
```

用于：

```text
Lock
Lease
Cache
Rate Limit
Short-lived State
```

### Kafka

```text
Event / Task Transport
```

用于：

```text
Task Queue
Event Bus
Workflow Event
```

这种架构对于 Java/Spring Cloud 背景的工程师非常容易落地。

---

# 69. AI Agent Workflow Runtime

最终可以形成一个完整架构：

```text
                         User
                          │
                          ▼
                  ┌───────────────┐
                  │ Workflow API  │
                  └───────┬───────┘
                          │
                          ▼
                ┌───────────────────┐
                │ Workflow Runtime  │
                │                   │
                │ State Machine     │
                │ Scheduler         │
                │ Policy Engine     │
                │ Retry Manager     │
                │ Timer Manager     │
                │ Recovery Manager  │
                └─────────┬─────────┘
                          │
                     Task Queue
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
       Agent Worker   Tool Worker   Human Task
            │
            ▼
           LLM
            │
            ▼
         Decision
            │
            ▼
       Policy Validation
            │
            ▼
          Action
            │
            ▼
          Event
            │
            ▼
      Workflow Runtime
```

这个架构实际上就是：

> **Agent Runtime + Workflow Engine + Event-driven Architecture**

---

# 70. Workflow 与传统微服务架构的关系

微服务解决：

```text
Service Boundary
```

Workflow 解决：

```text
Process Boundary
```

例如：

```text
Order Service
Payment Service
Inventory Service
Shipping Service
```

Workflow：

```text
Order
 ↓
Payment
 ↓
Inventory
 ↓
Shipping
```

因此：

```text
Microservice
    = Who owns the data/capability?

Workflow
    = Who executes the process and in what order?
```

两者应该结合，而不是互相替代。

---

# 71. Workflow 与 Kafka 的关系

Kafka 是：

```text
Event Transport
```

Workflow 是：

```text
Process Execution
```

不要认为：

```text
Kafka = Workflow Engine
```

Kafka 可以传递：

```text
TASK_READY
TASK_COMPLETED
PAYMENT_SUCCESS
APPROVAL_GRANTED
```

但 Kafka 本身并不会天然解决：

```text
Workflow State
Dependency
Human Wait
Compensation
Task Lifecycle
Versioning
```

因此：

```text
Kafka
    = Event Infrastructure

Workflow Engine
    = Process Orchestration
```

---

# 72. Workflow 与 Kubernetes 的关系

Kubernetes 负责：

```text
Container
Pod
Deployment
Service
Infrastructure Scheduling
```

Workflow 负责：

```text
Business / Agent Process
Task Dependencies
Retry
Approval
Compensation
Long-running Execution
```

例如：

```text
Kubernetes
    ↓
运行 Workflow Worker

Workflow
    ↓
运行 Business Process
```

两者属于不同层次。

---

# 73. Workflow 与 Temporal 类系统的核心思想

现代 Durable Workflow 系统的一个重要思想是：

> **Workflow Code 可以像普通代码一样写，但 Runtime 会记录执行历史并在故障后恢复。**

例如概念上：

```java
Workflow.execute(() -> {

    Result a = stepA();

    Result b = stepB(a);

    Result c = stepC(b);

    return c;
});
```

背后 Runtime 实际维护：

```text
Workflow History
Task State
Timer
Retry
Checkpoint
Recovery
```

这使开发者能够：

> 用接近普通程序的方式编写长期运行流程。

这是 Workflow Engine 发展的一个重要方向：

> **Durable Execution。**

---

# 74. Workflow 的真正价值：把时间纳入程序模型

这是我认为 Workflow 最深层的一个概念。

传统程序：

```text
Request
 ↓
Execution
 ↓
Response
```

Workflow：

```text
Start
 ↓
Execute
 ↓
Wait 3 days
 ↓
Resume
 ↓
Execute
 ↓
Wait human
 ↓
Resume
 ↓
Complete
```

因此：

> **Workflow 把“时间”变成了程序的一部分。**

普通代码主要描述：

```text
What to do
```

Workflow 还描述：

```text
When to do
What happens while waiting
What happens after failure
How to resume
```

这就是 Workflow 的本质价值。

---

# 75. AI Agent Workflow 的真正变化

传统 Workflow：

```text
Human defines:
A → B → C → D
```

AI Workflow：

```text
Human defines:
A → Agent → B → Agent → C
```

Agent 节点能够：

```text
Reason
Plan
Select Tool
Generate Decision
Adapt Strategy
```

但是：

```text
Workflow Runtime
```

仍然负责：

```text
State
Permission
Persistence
Retry
Timeout
Recovery
Audit
```

所以未来更可能出现：

```text
Deterministic Workflow
        +
Probabilistic Agent
        +
Policy Engine
        +
Durable Runtime
```

而不是：

```text
Everything controlled by LLM
```

---

# 76. Workflow Architecture 的最终抽象

可以把整个系统抽象成：

[
AgentWorkflow =
Workflow
+
StateMachine
+
LLM
+
Tool
+
Event
+
Persistence
+
Policy
]

其中：

```text
Workflow
    → Process

State Machine
    → Lifecycle

LLM
    → Intelligence

Tool
    → Capability

Event
    → Communication

Persistence
    → Durability

Policy
    → Safety
```

最终形成：

```text
              Intelligence
                   │
                  LLM
                   │
                   ▼
              Decision
                   │
                   ▼
            ┌─────────────┐
            │  Workflow   │
            │   Engine    │
            └──────┬──────┘
                   │
          ┌────────┼────────┐
          ▼        ▼        ▼
       State      Event    Policy
          │        │        │
          └────────┼────────┘
                   ▼
                Action
                   │
          ┌────────┼────────┐
          ▼        ▼        ▼
        Tool     Human     Agent
```

---

# 77. 最重要的十条 Workflow Engineering 原则

如果只保留本文最重要的内容，我建议记住：

### 1. Workflow 是可执行的流程模型

不是简单流程图。

### 2. Workflow Definition 和 Runtime 分离

```text
Definition ≠ Execution
```

### 3. DAG 是基础，但不是全部

复杂 Workflow 需要：

```text
DAG + State Machine + Event
```

### 4. Worker 与 Scheduler 分离

```text
Scheduler → Queue → Worker
```

### 5. Workflow 必须 Durable

```text
Process Crash ≠ Workflow Crash
```

### 6. Retry 必须考虑幂等

```text
At Least Once
+
Idempotency
```

### 7. Failure 必须显式建模

```text
Retry
Timeout
Compensation
Recovery
```

### 8. Human Wait 不能占用 Worker

```text
Persist → Release → Resume
```

### 9. Workflow 必须支持 Versioning

```text
Workflow v1
Workflow v2
Workflow v3
```

### 10. LLM 应该是 Workflow 中的智能节点

而不是：

```text
LLM = Workflow Runtime
```

---

# 78. 最终总结

Workflow 的发展经历了一个非常有意思的过程：

```text
传统流程
   ↓
BPM
   ↓
DAG
   ↓
Workflow Engine
   ↓
Distributed Workflow
   ↓
Durable Execution
   ↓
AI Workflow
   ↓
Agent Workflow
   ↓
Multi-Agent Workflow
```

传统 Workflow 解决的是：

> **如何可靠地执行一个复杂流程。**

AI Agent 解决的是：

> **如何让机器具备推理和自主决策能力。**

两者结合以后，真正有价值的系统变成：

> **让非确定性的 AI 决策运行在确定性的 Workflow Runtime 之中。**

因此，一个成熟的 Agent 系统不应该只是：

```text
Prompt
 ↓
LLM
 ↓
Tool
```

而应该是：

```text
                         User
                          │
                          ▼
                  ┌──────────────┐
                  │   Workflow   │
                  │   Runtime    │
                  └──────┬───────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
           State       Policy      Event
              │          │          │
              └──────────┼──────────┘
                         ▼
                    Agent / LLM
                         │
                    Decision
                         │
                         ▼
                       Tool
                         │
                         ▼
                       Result
                         │
                         ▼
                       Event
                         │
                         ▼
                  Workflow Runtime
```

最终可以用一句话概括：

> **State Machine 管理“状态”，Workflow 管理“流程”，LLM 提供“智能”，Tool 提供“能力”，Event 负责“连接”，Durable Runtime 保证“可靠执行”。**

这六者很可能会成为下一代 **Agent Runtime Architecture** 的核心组成部分。

而真正成熟的 Agent Engineering，最终追求的不是：

> “让 Agent 尽可能自主。”

而是：

> **在明确的边界、状态、流程、策略和持久化机制中，让 Agent 尽可能自主。**

下一篇建议选 **“Durable Execution 深度技术博客”** 或 **“Agent Workflow Runtime 源码级架构设计”**。
