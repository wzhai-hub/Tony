---
title: Checkpoint：构建可恢复、可暂停、可回溯的 AI Agent 执行系统
# tags:
#   - nodejs
date: '2026-08-08'
summary: Checkpoint 是 Agent 在某个确定执行边界上的状态快照。
---
# Checkpoint：构建可恢复、可暂停、可回溯的 AI Agent 执行系统

## 引言：Agent 真正进入生产环境后，最容易被忽略的是什么？

很多 Agent Demo 看起来非常简单：

```text
User
 ↓
LLM
 ↓
Tool
 ↓
Result
```

但真正进入生产环境之后，Agent 的执行过程往往变成：

```text
User
 ↓
Agent
 ↓
Planning
 ↓
Tool A
 ↓
Tool B
 ↓
Human Approval
 ↓
Tool C
 ↓
External API
 ↓
Tool D
 ↓
Database
 ↓
Final Answer
```

这个过程可能持续：

* 几秒
* 几分钟
* 几小时
* 甚至几天

而 Agent 执行过程中随时可能发生：

```text
Pod Restart
Network Timeout
LLM Timeout
Tool Failure
Database Failure
Human Approval
Process Crash
Deployment
```

如果 Agent 没有 Checkpoint，那么一旦执行到：

```text
Step 17
```

进程突然挂掉：

```text
Step 1
Step 2
...
Step 16
Step 17
💥
```

系统可能只能从头开始：

```text
Step 1
Step 2
...
```

这对于 Agent 来说不仅浪费计算资源，更危险的是：

> **已经执行过的副作用操作可能被重复执行。**

例如：

```text
支付
发送邮件
创建订单
部署服务
修改数据库
```

因此，Checkpoint 的真正意义不是简单的“保存状态”。

它解决的是一个更加核心的问题：

> **如何让一个长时间运行的 AI Agent，在任意中断之后，从一个一致、可验证的执行位置继续运行？**

LangGraph 当前的 Checkpointer 就是按照这一思想设计的：将 graph state 按 super-step 保存到 thread 中，从而支持 human-in-the-loop、fault tolerance、time travel 和跨交互状态恢复。

---

# 一、什么是 Checkpoint？

最简单的定义：

> **Checkpoint 是 Agent 在某个确定执行边界上的状态快照。**

例如 Agent 执行：

```text
START
 ↓
Analyze
 ↓
Search
 ↓
Generate Plan
 ↓
Execute
 ↓
END
```

可以保存：

```text
Checkpoint 0
    ↓
Checkpoint 1
    ↓
Checkpoint 2
    ↓
Checkpoint 3
    ↓
Checkpoint 4
```

假设：

```text
Checkpoint 3
```

之后执行失败：

```text
Checkpoint 3
      ↓
Execute
      ↓
💥
```

恢复时：

```text
Checkpoint 3
      ↓
Resume
      ↓
Execute
```

而不是：

```text
START
 ↓
Analyze
 ↓
Search
 ↓
Generate Plan
 ↓
Execute
```

所以：

```text
Checkpoint = Recovery Point
```

但这还不够。

一个真正生产级的 Checkpoint 通常还需要：

```text
State
+
Execution Position
+
Version
+
Metadata
+
Pending Writes
+
Identity
```

---

# 二、Checkpoint 不等于 Memory

这是理解 Agent Architecture 时最容易混淆的地方。

很多人会说：

> Checkpoint 就是 Agent Memory。

实际上不是。

可以简单区分：

```text
Checkpoint
    ↓
当前 Agent 执行状态

Memory / Store
    ↓
长期知识和用户信息
```

例如：

```text
User:
我的名字叫 Vincent。
```

这可能属于：

```text
Long-Term Memory
```

而：

```text
Agent 当前正在执行：

Step 8
正在等待数据库查询结果
```

属于：

```text
Checkpoint State
```

LangGraph 当前文档也明确区分了两类 persistence：

```text
Checkpointer
    ↓
thread-scoped state

Store
    ↓
cross-thread durable data
```

也就是说，Checkpoint 更接近：

> **Execution State**

而 Store 更接近：

> **Application Memory**。([GitHub][2])

---

# 三、Checkpoint 的核心价值

Checkpoint 主要解决六类问题。

## 1. Fault Tolerance

Agent 崩溃之后恢复：

```text
Step 10
 ↓
Step 11
 ↓
💥
```

恢复：

```text
Step 10
 ↓
Step 11
```

---

## 2. Human-in-the-Loop

Agent：

```text
Plan
 ↓
Approval Required
 ↓
WAIT
```

Checkpoint 保存：

```text
state = ...
status = WAITING_APPROVAL
next = execute
```

用户第二天批准：

```text
APPROVED
 ↓
Resume
```

而不是要求 Agent 从头执行。

LangGraph 的 persistence 正是 HITL 的基础能力之一，因为系统需要在中断后保存状态，并在人工修改或批准后恢复。([Docs by LangChain][1])

---

## 3. Long-Running Workflow

例如：

```text
Research Agent

Search
 ↓
Analyze
 ↓
Search
 ↓
Analyze
 ↓
Generate Report
 ↓
Human Review
 ↓
Publish
```

整个流程可能持续几个小时。

Checkpoint 可以让：

```text
Agent Process
```

与：

```text
Workflow State
```

解耦。

---

# 四、Checkpoint 的真正核心：Execution Cursor

从分布式系统角度看：

> Checkpoint 不只是 Snapshot，它实际上还是 Agent 的执行 Cursor。

例如：

```json
{
  "threadId": "thread-001",
  "checkpointId": "cp-007",
  "nextNode": "execute_payment",
  "state": {
    "orderId": "ORDER-1001",
    "amount": 1000,
    "approved": true
  }
}
```

这里：

```text
state
```

描述：

> Agent 现在是什么状态？

而：

```text
nextNode
```

描述：

> Agent 下一步应该执行什么？

所以：

```text
Checkpoint
=
State Snapshot
+
Execution Cursor
```

这是理解 Checkpoint 的关键。

---

# 五、为什么 Checkpoint 必须有 Thread？

如果没有 Thread：

```text
Agent
 ↓
Checkpoint
```

系统无法知道：

```text
这个 Checkpoint 属于谁？
```

因此通常：

```text
Thread
 ├── Checkpoint 1
 ├── Checkpoint 2
 ├── Checkpoint 3
 └── Checkpoint 4
```

例如：

```text
thread_id = conversation-1001
```

对应：

```text
cp-001
cp-002
cp-003
cp-004
```

LangGraph 的设计中，`thread_id` 是 Checkpoint 持久化和恢复的核心标识；没有它，系统无法正确加载对应 thread 的状态来恢复执行。([GitHub][3])

---

# 六、Checkpoint 是时间序列

Checkpoint 更准确的模型是：

```text
Thread
   │
   ├── CP1
   │
   ├── CP2
   │
   ├── CP3
   │
   ├── CP4
   │
   └── CP5
```

所以它天然形成：

```text
State Timeline
```

例如：

```text
10:00 CP1
10:01 CP2
10:02 CP3
10:05 CP4
10:10 CP5
```

这带来了一个非常重要的能力：

> **Time Travel**

你可以查看：

```text
10:02
```

Agent 到底是什么状态。

甚至可以：

```text
CP3
 ↓
Fork
 ↓
Alternative Execution
```

LangGraph 的 Checkpoint 设计支持从历史状态检查、恢复和 fork 出新的执行轨迹，这也是它支持 time-travel debugging 的基础。([GitHub][3])

---

# 七、Checkpoint 与 Event Sourcing 的关系

Checkpoint 和 Event Sourcing 很像，但并不完全相同。

Event Sourcing：

```text
Event 1
Event 2
Event 3
Event 4
```

通过：

```text
Replay(Event1...Event4)
```

恢复 State。

Checkpoint：

```text
Checkpoint
```

直接保存：

```text
Current State
```

因此：

```text
Event Sourcing
    ↓
Replay events
    ↓
State
```

而：

```text
Checkpoint
    ↓
Load snapshot
    ↓
State
```

实际生产系统可以结合：

```text
Events
+
Checkpoint
```

例如：

```text
CP100
 ↓
Event 101
Event 102
Event 103
Event 104
```

恢复：

```text
Load CP100
 ↓
Replay 101-104
```

这样可以减少：

```text
Replay Cost
```

这其实就是经典数据库和分布式系统中的：

> **Snapshot + Log**

思想。

---

# 八、Checkpoint 与 Kafka Offset

如果你熟悉 Kafka，会发现 Checkpoint 和 Consumer Offset 有非常强的相似性。

Kafka：

```text
Partition
 ↓
Offset 100
 ↓
Consumer
```

Agent：

```text
Workflow
 ↓
Checkpoint 100
 ↓
Agent Runtime
```

Kafka 的 Offset 表示：

> 消费到哪里。

Agent Checkpoint 表示：

> 执行到哪里。

因此：

```text
Kafka Offset
≈
Agent Execution Cursor
```

但 Agent Checkpoint 更复杂，因为它通常还需要保存：

```text
State
Tool Result
Plan
Pending Task
Human Approval
Execution Metadata
```

---

# 九、Checkpoint 的数据模型

一个生产级 Checkpoint 可以抽象成：

```json
{
  "checkpointId": "cp-100",
  "threadId": "thread-001",
  "parentCheckpointId": "cp-099",
  "sequence": 100,

  "state": {
    "messages": [],
    "plan": {},
    "variables": {},
    "toolResults": {}
  },

  "execution": {
    "currentNode": "payment",
    "nextNode": "notification",
    "status": "WAITING"
  },

  "metadata": {
    "agentId": "payment-agent",
    "runId": "run-001",
    "createdAt": "2026-08-22T10:00:00Z"
  },

  "pendingWrites": []
}
```

核心字段可以分成：

```text
Identity
State
Cursor
Version
Metadata
Pending Writes
```

---

# 十、为什么需要 Parent Checkpoint？

因为 Checkpoint 本质上形成一个版本链：

```text
CP1
 ↓
CP2
 ↓
CP3
 ↓
CP4
```

于是：

```text
parent = CP3
current = CP4
```

如果发生：

```text
Time Travel
```

可能出现：

```text
        CP1
         ↓
        CP2
         ↓
        CP3
       /   \
     CP4   CP4'
      ↓
     CP5
```

这就变成：

> **Checkpoint Version Tree**

而不是简单的线性历史。

这对于：

```text
Debugging
Simulation
What-if Analysis
Agent Planning
```

非常有价值。

---

# 十一、Super-Step：Checkpoint 的真正保存边界

这是理解现代 Agent Runtime 的关键概念。

假设：

```text
A → B → C
```

可以理解为：

```text
SuperStep 1
A

SuperStep 2
B

SuperStep 3
C
```

每个 Super-Step 完成后：

```text
Persist Checkpoint
```

因此：

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
```

LangGraph 当前实现就是以 super-step 作为完整 checkpoint 的边界，同时还保存 super-step 内各 task 的中间 writes，用于失败恢复。([GitHub][3])

---

# 十二、为什么不能只在 Agent 完成后保存？

错误设计：

```text
Agent Start
 ↓
100 steps
 ↓
Save
```

如果：

```text
Step 99
 ↓
💥
```

前面 98 步全部丢失。

因此：

```text
Checkpoint Frequency
```

非常重要。

一般：

```text
Step
 ↓
Checkpoint
 ↓
Step
 ↓
Checkpoint
```

这样才能控制：

```text
Recovery Point Objective
```

---

# 十三、Checkpoint 与 RPO

Checkpoint 可以直接套用传统分布式系统中的 RPO 概念。

假设：

```text
Checkpoint Interval = 30 seconds
```

那么理论上：

```text
Maximum Lost Execution State
≈ 30 seconds
```

即：

```text
RPO ≈ Checkpoint Interval
```

当然，真实系统还要考虑：

```text
Async Persistence
Write Buffer
Crash Window
```

因此不能简单认为完全等价。

---

# 十四、Sync / Async Checkpoint

Checkpoint Persistence 通常有三种策略。

## 1. Synchronous

```text
Execute
 ↓
Persist
 ↓
Next Step
```

优点：

```text
Durability 高
```

缺点：

```text
Latency 高
```

---

## 2. Asynchronous

```text
Execute
 ↓
Next Step
 ↓
Background Persist
```

优点：

```text
性能好
```

缺点：

```text
Crash Window
```

可能出现：

```text
Step 10
 ↓
Step 11
 ↓
💥
```

而：

```text
CP10
```

可能还没写完。

LangGraph 当前提供 `exit`、`async` 和 `sync` 三种 durability 模式，用于在性能与持久性之间做不同取舍；`sync` 会在下一步开始前确保 checkpoint 持久化，`async` 则允许持久化与下一步并行。([GitHub][3])

---

# 十五、Checkpoint 与 Pending Writes

这是非常值得深入理解的一个设计。

假设：

```text
SuperStep 10
```

同时运行：

```text
Node A
Node B
Node C
```

结果：

```text
A → Success
B → Success
C → Failure
```

如果只保存：

```text
SuperStep 10 Checkpoint
```

可能意味着：

```text
A
B
```

的结果也丢失。

重新执行：

```text
A
B
C
```

会浪费资源，甚至造成副作用重复。

因此可以保存：

```text
Pending Writes
```

例如：

```text
CP10
 ├── A = SUCCESS
 ├── B = SUCCESS
 └── C = FAILED
```

恢复：

```text
A → Skip
B → Skip
C → Retry
```

这就是非常典型的：

> **Partial Progress Recovery**

LangGraph 当前 Checkpointer 的 `put_writes` 就承担类似职责：同一 super-step 中已经成功完成的节点，其 writes 可以持久化下来，恢复时无需重复运行这些成功任务。([Docs by LangChain][1])

---

# 十六、Checkpoint 最大的陷阱：副作用

这是整个 Checkpoint 设计中最重要的问题。

假设：

```text
Agent
 ↓
sendEmail()
 ↓
Checkpoint
```

执行：

```text
sendEmail()
 ↓
Success
 ↓
💥
```

但是：

```text
Checkpoint
```

还没有保存。

恢复：

```text
sendEmail()
```

再次执行。

于是：

```text
Email
Email
```

用户收到两封邮件。

所以：

> **Checkpoint 解决的是状态恢复问题，不自动解决副作用重复问题。**

必须结合：

```text
Idempotency
+
Transactional Boundary
+
Effect Journal
```

---

# 十七、Checkpoint + Idempotency

例如：

```java
String idempotencyKey =
    checkpointId + ":" + toolCallId;
```

调用：

```text
sendPayment(
    orderId,
    idempotencyKey
)
```

外部系统保证：

```text
same idempotencyKey
=
same operation
```

于是：

```text
First execution
 ↓
Payment Success

Retry
 ↓
idempotencyKey already exists
 ↓
Return previous result
```

这才是真正的：

```text
Durable Agent Execution
```

---

# 十八、Checkpoint + Transaction

更进一步：

```text
Agent State
+
Tool Side Effect
```

最好不要被认为是两个完全独立的事情。

例如：

```text
DB Transaction
```

可以：

```text
BEGIN

Update Business Data

Insert Tool Effect Journal

COMMIT
```

然后：

```text
Checkpoint
```

记录：

```text
effectId
```

恢复时：

```text
effectId exists
```

则：

```text
Don't execute again
```

这其实是把 Agent Execution 与传统分布式事务思想连接起来。

---

# 十九、Checkpoint 与 Two-Phase Execution

对于高风险 Tool，可以设计：

```text
Prepare
 ↓
Approval
 ↓
Execute
 ↓
Commit
```

例如支付：

```text
Agent
 ↓
Prepare Payment
 ↓
Checkpoint
 ↓
Approval
 ↓
Execute Payment
 ↓
Checkpoint
```

这类似：

```text
Transaction Protocol
```

而不是：

```text
LLM
 ↓
payment()
```

---

# 二十、Checkpoint 与 Approval

上一章讨论过 Approval。

现在把两者结合起来：

```text
Agent
 ↓
Plan
 ↓
Checkpoint
 ↓
Approval Required
 ↓
WAIT
```

用户几小时之后：

```text
Approve
```

系统：

```text
Load Checkpoint
 ↓
Apply Approval
 ↓
Resume
```

这说明：

> **Approval 本质上是 Checkpoint 的一个特殊暂停点。**

没有 Checkpoint：

```text
Approval
```

很难可靠实现。

现代 Agent Runtime 中，这也是 Checkpoint 与 HITL 紧密结合的原因。([Docs by LangChain][1])

---

# 二十一、Checkpoint + Human-in-the-Loop

完整流程：

```text
                    Agent
                      │
                      ▼
                   Planning
                      │
                      ▼
                  Checkpoint
                      │
                      ▼
               Approval Required
                      │
                      ▼
                    WAIT
                      │
             Human reviews
                      │
            ┌─────────┴─────────┐
            │                   │
          Reject              Approve
            │                   │
            ▼                   ▼
          Stop                Resume
                                │
                                ▼
                             Execute
```

这里：

```text
WAIT
```

期间：

```text
Worker
```

完全可以释放。

这是生产级 Agent Runtime 非常重要的能力。

---

# 二十二、Checkpoint 与 Worker 解耦

错误架构：

```text
Agent Request
 ↓
Worker Thread
 ↓
Wait for human
 ↓
Worker Thread blocked
```

如果等待：

```text
2 hours
```

这会浪费：

```text
CPU
Thread
Memory
Connection
Pod
```

正确架构：

```text
Agent
 ↓
Checkpoint
 ↓
WAIT
 ↓
Release Worker
```

两小时后：

```text
Approval Event
 ↓
Task Queue
 ↓
New Worker
 ↓
Load Checkpoint
 ↓
Resume
```

因此：

> **Checkpoint 让 Agent Workflow 与 Worker 生命周期解耦。**

这是 Durable Execution 的核心思想之一。生产级 Agent Runtime 可以让执行暂停后释放 worker，之后由其他 worker 从最新 checkpoint 恢复。([LangChain][4])

---

# 二十三、Checkpoint 与 Kubernetes

在 Kubernetes 环境中：

```text
Pod
 ↓
Agent Runtime
```

Pod 随时可能：

```text
Restart
Reschedule
Scale Down
Node Failure
Deployment
```

如果状态存在：

```text
JVM Heap
```

那么：

```text
Pod Death
=
Agent State Loss
```

如果：

```text
Agent State
 ↓
PostgreSQL
```

那么：

```text
Pod Death
 ↓
New Pod
 ↓
Load Checkpoint
 ↓
Resume
```

这就是：

> **Stateless Worker + Stateful Workflow**

一个非常适合 Cloud Native Agent 的架构。

---

# 二十四、Checkpoint Storage 怎么选？

常见选择：

```text
Memory
SQLite
PostgreSQL
Redis
Object Storage
Distributed Database
```

---

## Memory

适合：

```text
Development
Testing
```

不适合生产。

因为：

```text
Process Restart
=
State Lost
```

LangGraph 文档也明确说明，内存型 saver 在进程重启后不会保留 checkpoint；生产环境需要持久化 backend，例如 PostgreSQL。([GitHub][2])

---

# 二十五、为什么 PostgreSQL 很适合 Checkpoint？

PostgreSQL 的优势：

```text
ACID
Transactions
Indexes
JSONB
Concurrency
Durability
HA
```

可以设计：

```text
checkpoint
checkpoint_writes
thread
effect_journal
```

例如：

```sql
CREATE TABLE agent_checkpoint (
    thread_id           VARCHAR(128),
    checkpoint_id       VARCHAR(128),
    parent_checkpoint   VARCHAR(128),
    sequence_no         BIGINT,
    state               JSONB,
    metadata            JSONB,
    created_at          TIMESTAMP,
    PRIMARY KEY (
        thread_id,
        checkpoint_id
    )
);
```

再建立：

```sql
CREATE INDEX idx_checkpoint_thread
ON agent_checkpoint(thread_id, sequence_no DESC);
```

这样：

```text
Load latest checkpoint
```

可以快速完成。

---

# 二十六、Redis 适不适合？

Redis 非常适合：

```text
Hot State
Lock
Lease
Cache
Fast Resume Metadata
```

但如果直接把 Redis 当作唯一 Checkpoint Store，要慎重考虑：

```text
Durability
Persistence
Memory Cost
Retention
Large State
HA
Disaster Recovery
```

一个更合理的架构可能是：

```text
PostgreSQL
    ↓
Source of Truth

Redis
    ↓
Hot Checkpoint Cache
```

即：

```text
Agent
 ↓
Redis
 ↓
PostgreSQL
```

而不是完全依赖 Redis。

---

# 二十七、Checkpoint Storage 的分层架构

大型 Agent 系统可以设计：

```text
             Agent Runtime
                   │
                   ▼
            ┌──────────────┐
            │ Redis Cache  │
            └──────┬───────┘
                   │
                   ▼
            ┌──────────────┐
            │ PostgreSQL   │
            │ Checkpoint   │
            └──────┬───────┘
                   │
                   ▼
            Object Storage
```

例如：

```text
Small State
 ↓
PostgreSQL JSONB

Large Artifact
 ↓
S3 / Object Storage

Hot State
 ↓
Redis
```

Checkpoint 不应该盲目把所有数据塞进数据库。

---

# 二十八、Checkpoint Size 是一个非常严重的问题

假设 Agent 每一步保存：

```json
{
  "messages": [
    "...",
    "...",
    "...",
    "..."
  ]
}
```

如果：

```text
100 steps
```

每次都保存完整：

```text
messages
```

那么：

```text
Checkpoint Size
```

会快速增长。

假设：

```text
平均 state = 500 KB
1000 checkpoints
```

就是：

```text
500 MB
```

而真实系统通常还有：

```text
Metadata
Indexes
Versions
Replication
```

所以实际成本更高。

LangGraph 当前文档也特别提醒，长对话中每个 super-step 保存完整 state 可能导致存储快速增长，并提供增量 delta 机制作为一种存储优化方向。([Docs by LangChain][1])

---

# 二十九、Full Snapshot vs Delta Checkpoint

两种主要方案。

## Full Snapshot

```text
CP1 = Full State
CP2 = Full State
CP3 = Full State
```

优点：

```text
恢复简单
读取快
```

缺点：

```text
存储大
```

---

## Delta

```text
CP1 = Full State

CP2 = Delta 2
CP3 = Delta 3
CP4 = Delta 4
```

恢复：

```text
CP1
 +
Delta2
 +
Delta3
 +
Delta4
```

优点：

```text
Storage Efficient
```

缺点：

```text
Recovery Cost
Complexity
```

因此可以：

```text
Full Snapshot
+
Delta
```

结合使用。

---

# 三十、Checkpoint Compaction

一个长期运行的 Agent：

```text
CP1
CP2
CP3
...
CP10000
```

显然不可能永久保存全部版本。

因此需要：

```text
Retention
+
Compaction
```

例如：

```text
Keep latest 100
```

或者：

```text
Keep:
Daily Snapshot
+
Last 100 checkpoints
```

例如：

```text
CP1 ─┐
CP2  │
...  ├── Delete
CP9000
CP9900 ── Keep
CP10000 ─ Keep
```

生产系统还应该考虑：

```text
Compliance Retention
Legal Hold
Audit Retention
```

所以：

> **Checkpoint retention 不是单纯的数据库清理问题，而是 Governance 问题。**

---

# 三十一、Checkpoint Serialization

Checkpoint 中通常会包含：

```text
Message
Tool Result
Plan
State
Metadata
```

因此需要序列化：

```text
Object
 ↓
Serializer
 ↓
Bytes
 ↓
Storage
```

常见：

```text
JSON
MessagePack
Protobuf
Avro
```

JSON：

```text
可读
调试方便
体积大
```

Protobuf：

```text
高性能
Schema
体积小
```

MessagePack：

```text
二进制
较紧凑
```

选择取决于：

```text
State Size
Performance
Compatibility
Security
```

此外，反序列化必须考虑安全边界。LangGraph 当前 checkpoint 文档特别提示了序列化数据反序列化的安全问题，并提供 strict MessagePack 等限制机制。([GitHub][5])

---

# 三十二、Checkpoint Schema Evolution

这是生产系统非常容易忽略的问题。

今天：

```json
{
  "user": "Vincent",
  "age": 30
}
```

半年之后：

```json
{
  "user": "Vincent",
  "profile": {
    "age": 30
  }
}
```

那么：

```text
Old Checkpoint
```

如何恢复？

因此 Checkpoint 必须有：

```text
schemaVersion
```

例如：

```json
{
  "schemaVersion": 3,
  "state": {}
}
```

恢复：

```text
v1
 ↓
Migration
 ↓
v3
```

这和：

```text
Database Migration
```

非常类似。

---

# 三十三、Checkpoint 与版本兼容

一个更完整的模型：

```text
Checkpoint
{
    schemaVersion
    agentVersion
    workflowVersion
    modelVersion
}
```

为什么需要：

```text
agentVersion
```

因为：

```text
Agent v1
```

生成的 Plan：

```text
plan format A
```

可能无法被：

```text
Agent v2
```

直接理解。

所以：

> **Checkpoint 是 Agent Workflow 的持久化 ABI。**

这是一个非常重要的架构观点。

---

# 三十四、Checkpoint 与 Model Version

还有一个更容易被忽略的问题：

```text
Checkpoint
 ↓
LLM
```

如果今天使用：

```text
Model A
```

明天切换：

```text
Model B
```

恢复旧 Workflow：

```text
CP100
 ↓
Model B
 ↓
Different Decision
```

那么：

```text
Replay
```

可能产生不同结果。

所以对于需要严格可重复性的场景，需要记录：

```text
model
modelVersion
temperature
systemPromptVersion
toolVersion
```

例如：

```json
{
  "model": "model-x",
  "modelVersion": "2026-08",
  "promptVersion": "v17",
  "toolVersion": "v4"
}
```

---

# 三十五、Checkpoint 与 Determinism

传统程序：

```text
Input
 ↓
Function
 ↓
Output
```

通常：

```text
deterministic
```

但 Agent：

```text
Input
 ↓
LLM
 ↓
Probabilistic Decision
```

天然具有非确定性。

所以：

```text
Checkpoint Resume
```

并不意味着：

```text
Replay = Same Result
```

这点非常重要。

Checkpoint 的目标应该是：

> **恢复执行状态**

而不是：

> **保证 LLM 重新生成完全相同的输出。**

如果需要 replay consistency，需要进一步保存：

```text
LLM Response
Tool Result
Random Seed
Model Version
Prompt Version
```

甚至直接：

```text
Replay Stored Outputs
```

---

# 三十六、Checkpoint 与 Replay

可以设计两种模式。

## Resume

```text
Load Checkpoint
 ↓
Continue
 ↓
Call LLM
```

适合：

```text
Production Recovery
```

---

## Replay

```text
Load Checkpoint
 ↓
Replay historical outputs
 ↓
Analyze
```

适合：

```text
Debug
Audit
Testing
```

所以：

```text
Resume ≠ Replay
```

这是 Agent Observability 中非常重要的概念。

---

# 三十七、Checkpoint + Observability

生产系统应该记录：

```text
traceId
runId
threadId
checkpointId
nodeId
toolCallId
```

例如：

```text
traceId
  │
  ├── Agent Plan
  │      checkpointId=CP01
  │
  ├── Search
  │      checkpointId=CP02
  │
  ├── Tool
  │      checkpointId=CP03
  │
  └── Approval
         checkpointId=CP04
```

这样可以做到：

```text
Trace
 ↓
Checkpoint
 ↓
State
 ↓
Tool
 ↓
External System
```

这会让 Agent Debugging 从：

```text
“模型为什么这样回答？”
```

升级成：

```text
“Agent 在 CP37 时为什么选择 Tool A？”
```

---

# 三十八、Checkpoint 与 Distributed Lock

多 Worker 场景：

```text
Worker A
Worker B
Worker C
```

可能同时恢复：

```text
thread-001
```

导致：

```text
Worker A → Execute
Worker B → Execute
```

所以必须有：

```text
Lease
+
Distributed Lock
```

例如：

```text
thread-001
 ↓
Lease acquired by Worker A
```

Worker B：

```text
Cannot acquire lease
```

执行完成：

```text
Checkpoint
 ↓
Release Lease
```

或者使用：

```text
Optimistic Concurrency Control
```

通过：

```text
version
```

检测冲突。

---

# 三十九、Checkpoint Version Conflict

假设：

```text
CP10 version=10
```

Worker A：

```text
Load CP10
```

Worker B：

```text
Load CP10
```

A 执行：

```text
CP11
version=11
```

B 也尝试：

```text
CP11
```

系统应该拒绝：

```text
Version conflict
```

否则：

```text
Last Write Wins
```

可能直接覆盖 Agent 状态。

所以：

```text
Checkpoint
+
Optimistic Lock
```

是非常合理的组合。

---

# 四十、Checkpoint 与 Multi-Agent

Multi-Agent 环境更加复杂：

```text
Supervisor
   │
   ├── Research Agent
   │
   ├── Coding Agent
   │
   └── Deployment Agent
```

每个 Agent 可能有：

```text
Thread
Checkpoint
State
```

例如：

```text
Supervisor Thread
       │
       ├── Research Thread
       │
       ├── Coding Thread
       │
       └── Deployment Thread
```

这就产生：

> **Checkpoint Namespace**

必须防止：

```text
Research CP
```

污染：

```text
Deployment CP
```

因此通常需要：

```text
tenant
+
agent
+
thread
+
checkpoint
```

形成唯一上下文。

---

# 四十一、Checkpoint Namespace

一个企业级 Key 可以是：

```text
tenantId
/
applicationId
/
agentId
/
threadId
/
checkpointId
```

例如：

```text
banking
/
payment-platform
/
payment-agent
/
order-1001
/
cp-007
```

这样可以解决：

```text
Multi-Tenant
Multi-Agent
Multi-Workflow
```

环境下的数据隔离问题。

---

# 四十二、Checkpoint Security

Checkpoint 中可能保存：

```text
User Data
Messages
API Results
Tool Arguments
Credentials
Business Data
```

因此不能简单认为：

```text
Checkpoint = harmless state
```

它可能是：

> **高敏感度数据资产。**

必须考虑：

```text
Encryption at Rest
Encryption in Transit
RBAC
Tenant Isolation
PII Masking
Secret Filtering
Retention
Audit
```

特别要避免：

```json
{
  "apiKey": "sk-xxxxx"
}
```

直接进入 Checkpoint。

更好的方式：

```text
Secret
 ↓
Secret Manager
 ↓
Reference ID
```

Checkpoint 只保存：

```json
{
  "secretRef": "secret/payment/api"
}
```

---

# 四十三、Checkpoint 与 GDPR / Data Governance

如果 Agent 处理：

```text
Customer Data
```

Checkpoint 可能成为一个隐藏的数据复制源。

例如：

```text
Original DB
      ↓
Agent State
      ↓
Checkpoint
      ↓
Backup
      ↓
Replica
```

于是删除用户数据时：

```text
DELETE User
```

并不代表：

```text
Checkpoint
```

中的数据也删除了。

因此需要设计：

```text
Data Retention
Data Deletion
Checkpoint Purging
Backup Purging
```

这是企业 Agent Governance 中非常重要的一环。

---

# 四十四、Checkpoint 与传统 Workflow Engine

如果你熟悉：

```text
Camunda
Temporal
Airflow
Cadence
```

会发现它们都有：

```text
Workflow State
Execution History
Retry
Resume
```

因此：

> **Agent Checkpoint 本质上正在把 LLM Agent 从“请求响应模型”带向“Durable Workflow Model”。**

传统：

```text
HTTP Request
 ↓
Response
```

Agent：

```text
Workflow
 ↓
State
 ↓
Checkpoint
 ↓
Resume
```

这是 Agent Architecture 非常重要的一次范式变化。

---

# 四十五、Checkpoint 与 Temporal 的思想联系

Temporal 的核心理念之一是：

```text
Workflow State
+
Durable Execution
```

Agent 也越来越需要：

```text
Durable Agent Execution
```

因此未来的 Agent Runtime 很可能逐渐具备：

```text
Workflow Engine
+
LLM
+
Tool Runtime
+
Checkpoint
+
Approval
+
Policy
```

形成：

```text
AI Workflow Engine
```

而不再只是：

```text
Chatbot
```

---

# 四十六、Checkpoint 的完整架构

一个生产级 Agent Checkpoint Architecture 可以设计为：

```text
                         User
                           │
                           ▼
                     Agent API
                           │
                           ▼
                  ┌────────────────┐
                  │ Agent Runtime  │
                  └───────┬────────┘
                          │
                    Execute Node
                          │
                          ▼
                  ┌────────────────┐
                  │ State Manager  │
                  └───────┬────────┘
                          │
                          ▼
                  ┌────────────────┐
                  │ Checkpoint     │
                  │ Manager        │
                  └───────┬────────┘
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
          Redis       PostgreSQL     Object
          Cache       Source Truth   Storage
             │            │            │
             └────────────┼────────────┘
                          │
                          ▼
                    Recovery Engine
                          │
                          ▼
                    Agent Worker
```

---

# 四十七、一个 Java/Spring Boot 版本的设计

如果使用你熟悉的 Java/Spring Boot 技术栈，可以定义：

```java
public interface CheckpointStore {

    void save(Checkpoint checkpoint);

    Optional<Checkpoint> load(
        String threadId,
        String checkpointId
    );

    Optional<Checkpoint> latest(
        String threadId
    );

    List<Checkpoint> history(
        String threadId
    );
}
```

Checkpoint：

```java
public class Checkpoint {

    private String checkpointId;

    private String threadId;

    private String parentCheckpointId;

    private long sequence;

    private String node;

    private AgentState state;

    private CheckpointStatus status;

    private long version;

    private Instant createdAt;
}
```

---

# 四十八、Agent Runtime

可以进一步定义：

```java
public class AgentRuntime {

    private final CheckpointStore checkpointStore;

    public AgentResult run(
        String threadId,
        AgentInput input) {

        Checkpoint checkpoint =
            checkpointStore.latest(threadId);

        AgentState state;

        if (checkpoint == null) {
            state = initialize(input);
        } else {
            state = checkpoint.getState();
        }

        while (!state.isCompleted()) {

            Node node = planner.next(state);

            NodeResult result =
                node.execute(state);

            state.apply(result);

            Checkpoint next =
                createCheckpoint(
                    threadId,
                    state,
                    node
                );

            checkpointStore.save(next);
        }

        return AgentResult.success(state);
    }
}
```

这就是最基础的：

```text
Load
 ↓
Execute
 ↓
Update State
 ↓
Checkpoint
 ↓
Next
```

---

# 四十九、真正生产级的 Resume

实际生产系统不能简单：

```java
load();
execute();
save();
```

还需要：

```text
Lease
Version
Idempotency
Retry
Timeout
Serialization
Schema Migration
Audit
Observability
```

完整流程：

```text
Load Checkpoint
      ↓
Acquire Lease
      ↓
Validate Version
      ↓
Validate Schema
      ↓
Restore State
      ↓
Determine Next Node
      ↓
Execute
      ↓
Record Side Effect
      ↓
Create Checkpoint
      ↓
Commit
      ↓
Release Lease
```

---

# 五十、Checkpoint 最重要的工程原则

可以总结为十条。

### 原则 1

```text
Checkpoint State
≠
Long-Term Memory
```

---

### 原则 2

```text
Checkpoint
=
State
+
Execution Cursor
```

---

### 原则 3

Checkpoint 必须有：

```text
Thread ID
```

---

### 原则 4

Checkpoint 必须具备：

```text
Version
```

---

### 原则 5

Checkpoint 必须考虑：

```text
Schema Evolution
```

---

### 原则 6

Checkpoint 不等于：

```text
Exactly Once
```

必须结合：

```text
Idempotency
```

---

### 原则 7

高风险 Tool 必须：

```text
Checkpoint
+
Approval
+
Authorization
```

---

### 原则 8

长时间等待必须：

```text
Persist
+
Release Worker
+
Resume
```

---

### 原则 9

Checkpoint Store 必须：

```text
Durable
+
Highly Available
```

---

### 原则 10

Checkpoint 本身必须受到：

```text
Security
+
Privacy
+
Governance
```

约束。

---

# 五十一、从 Checkpoint 到 Durable Agent

如果把整个演进过程画出来：

```text
Level 1
Stateless LLM
```

```text
Level 2
LLM + Conversation Memory
```

```text
Level 3
Agent + Tools
```

```text
Level 4
Agent + Checkpoint
```

```text
Level 5
Agent + Checkpoint + Approval
```

```text
Level 6
Agent + Checkpoint + Policy + Idempotency
```

```text
Level 7
Durable Agent Runtime
```

最终：

```text
                Durable Agent
                     │
       ┌─────────────┼─────────────┐
       │             │             │
       ▼             ▼             ▼
   Checkpoint     Policy       Approval
       │             │             │
       ▼             ▼             ▼
   Recovery     Authorization  Human
       │
       ▼
   Idempotency
       │
       ▼
   Tool Gateway
       │
       ▼
   External World
```

这时 Agent 才真正具备：

```text
Pause
Resume
Retry
Recover
Rollback
Replay
Fork
Audit
```

---

# 五十二、Checkpoint 最终解决的是什么问题？

如果只用一句话总结：

> **Checkpoint 解决的是 Agent Execution Continuity（执行连续性）。**

传统程序：

```text
Process
=
State
```

Process 死掉：

```text
State Lost
```

Checkpoint 架构：

```text
Process
   +
Externalized State
```

于是：

```text
Worker A
   ↓
Checkpoint
   ↓
Worker A dies
   ↓
Worker B
   ↓
Load Checkpoint
   ↓
Continue
```

因此：

> **Checkpoint 把 Agent 的“状态”从进程内存中解放出来，使 Agent Workflow 可以跨越进程、Pod、机器、时间甚至人工等待继续运行。**

---

# 五十三、Checkpoint 是 Agent Control Plane 的基础

结合前面讨论的 Approval，可以得到一个更加完整的 Agent Runtime：

```text
                         Agent
                           │
                           ▼
                     Agent Runtime
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
      Planning          Policy          Checkpoint
          │                │                │
          │                ▼                ▼
          │          Authorization      Persistence
          │                │                │
          └────────────────┼────────────────┘
                           │
                           ▼
                       Approval
                           │
                           ▼
                      Tool Gateway
                           │
                           ▼
                     External World
                           │
                           ▼
                         Audit
```

其中：

```text
Planning
    → 决定下一步做什么

Policy
    → 决定能不能做

Approval
    → 决定这一次是否允许做

Checkpoint
    → 记录执行到哪里

Tool Gateway
    → 控制如何执行

Audit
    → 记录最终发生了什么
```

这几个能力组合起来，才构成真正的：

> **Production-Grade Agent Runtime**

---

# 五十四、最终总结

Checkpoint 表面上只是：

```text
Save State
```

但从系统架构角度看，它实际上是：

```text
State Persistence
+
Execution Cursor
+
Recovery Point
+
Version History
+
Workflow Continuity
+
Human Pause/Resume
+
Failure Recovery
+
Time Travel
```

因此，Checkpoint 并不是一个简单的数据库表。

它是 Agent Runtime 的：

> **Durability Layer**

而当我们把：

```text
Checkpoint
+
Idempotency
+
Approval
+
Policy
+
Tool Gateway
+
Observability
```

组合起来之后，Agent 才真正从：

```text
LLM Application
```

进化成：

```text
Durable Autonomous System
```

这也是为什么现代 Agent 框架越来越强调 persistence、resumability 和 durable execution。OpenAI Agents SDK 当前也提供 sessions 与 resumable runs，用于在中断后恢复运行状态；其文档还特别强调审批 checkpoint 与 `RunState` 的结合。([OpenAI][6])

最终可以用一个公式概括：

```text
Production Agent
=
LLM
+
Tool Calling
+
State
+
Checkpoint
+
Policy
+
Approval
+
Idempotency
+
Observability
+
Security
```

其中 **Checkpoint 是把“智能”变成“可靠执行”的关键桥梁。**

如果把 Agent 看成一个“会思考的分布式 Workflow”，那么：

```text
LLM
       = Brain

Tool
       = Hands

Memory
       = Knowledge

Checkpoint
       = Continuity

Policy
       = Rules

Approval
       = Human Authority

Idempotency
       = Execution Safety

Observability
       = Eyes
```

这套思想对于从传统 **Java/Spring Cloud/Kubernetes 微服务架构** 转向 AI Agent Architecture 尤其重要：你真正需要掌握的不是某个 Agent Framework API，而是如何把 **Agent Execution 建造成一个能够跨 Worker、跨 Pod、跨时间安全恢复的分布式系统**。
