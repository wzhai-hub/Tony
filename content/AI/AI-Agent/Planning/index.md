---
title: Agent Planning 深度技术博客：从任务分解到动态规划与执行控制
# tags:
#   - nodejs
date: '2026-08-05'
summary: 不要让 Agent 只思考“下一步调用哪个 Tool”，而要让它理解“当前目标是什么、为了达到目标需要完成哪些任务、任务之间有什么依赖、执行过程中获得的新信息是否改变了原来的计划，以及什么时候已经足够完成任务”。
---

# Agent Planning 深度技术博客：从任务分解到动态规划与执行控制

> **摘要**
>
> 在 Agent Architecture 中，Tool Calling 解决的是“**Agent 能做什么**”，而 Planning 解决的是“**Agent 应该先做什么、后做什么，以及什么时候调整计划**”。
>
> 一个真正能够处理复杂任务的 Agent，不能简单地采用：
>
> `User → LLM → Tool → Answer`
>
> 而需要建立：
>
> `Goal → Planning → Task Decomposition → Execution → Observation → Re-Planning → Completion`
>
> Planning 是 Agent 从“工具调用器”进化为“任务执行系统”的关键技术。
>
> 本文从 Planning 的基本概念出发，深入分析 Task Decomposition、Plan-and-Execute、ReAct、Dynamic Planning、Hierarchical Planning、DAG Planning、Dependency Management、Re-Planning、Reflection、Planning State、Plan Validation、成本控制以及 Production Agent 中的 Planning Architecture

---

# 1. Agent 为什么需要 Planning？

Tool Calling 解决：

> “我有哪些工具可以使用？”

Planning 解决：

> “为了完成目标，我应该如何使用这些工具？”

例如用户提出：

> “分析一下昨天生产环境 API 延迟突然升高的原因。”

Agent 可能拥有：

```text
Prometheus Tool
Grafana Tool
Log Search Tool
Trace Search Tool
Kubernetes Tool
Database Tool
Git Tool
```

但问题不是：

```text
有没有工具？
```

而是：

```text
先查什么？
为什么查？
查到什么之后下一步做什么？
哪些任务可以并行？
什么时候应该停止？
如果假设不成立怎么办？
```

因此：

```text
Tool Calling
    ↓
Capability

Planning
    ↓
Strategy
```

可以把两者理解成：

> **Tool 是手，Planning 是行动策略。**

---

# 2. Planning 的本质

Agent Planning 可以定义为：

> **根据目标、当前状态、可用工具和约束，生成一个能够逐步执行并最终达到目标的行动计划。**

形式化一点：

```text
Goal
+
Current State
+
Available Actions
+
Constraints
        ↓
     Planner
        ↓
      Plan
```

例如：

```text
Goal:
分析 API latency 上升原因

Current State:
CPU 正常
Memory 未知
DB 状态未知

Available Actions:
queryMetrics()
searchLogs()
searchTraces()
queryDatabase()

Constraints:
只读
最大 10 次 Tool Call
最大 30 秒
```

Planner 可能产生：

```text
Plan:

1. 查询 API latency
2. 查询 error rate
3. 查询 CPU / Memory
4. 查询 DB latency
5. 查询 Trace
6. 综合分析
```

---

# 3. Planning 与 Workflow 的区别

这是 Agent Architecture 中非常重要的概念。

传统 Workflow：

```text
A → B → C → D
```

流程由开发人员定义。

Planning：

```text
Goal
 ↓
LLM
 ↓
A → C → B → ?
```

具体路径可能由模型根据环境动态决定。

例如：

```text
如果发现 CPU 很高
    ↓
继续检查 CPU

如果 CPU 正常
    ↓
检查 Database

如果 Database 正常
    ↓
检查 Redis
```

因此：

> **Workflow 是预定义路径，Planning 是动态生成路径。**

---

# 4. Planning 不等于生成一个 TODO List

这是一个非常容易产生的误解。

很多人认为：

```text
Planning = 

1. 查询订单
2. 查询库存
3. 查询物流
4. 输出结果
```

实际上真正的 Planning 至少包含：

```text
Goal
Task
Dependency
Constraint
Action
Expected Result
Decision
Fallback
Termination
```

例如：

```text
Task A:
查询订单

Task B:
查询库存

Dependency:
B 依赖 A 返回 productId

Task C:
查询物流

Dependency:
C 依赖 A 返回 shipmentId

Task D:
分析原因

Dependency:
D 依赖 A/B/C
```

这已经不是简单的 TODO List，而是一个：

> **Task Dependency Graph。**

---

# 5. Planning 的数学模型

可以把 Agent Planning 抽象成一个状态空间搜索问题。

定义：

```text
S = State
A = Action
T = Transition
G = Goal
```

Agent 当前状态：

```text
S0
```

执行 Action：

```text
A1
```

得到：

```text
S1
```

然后：

```text
S1
 ↓ A2
S2
 ↓ A3
S3
```

最终：

```text
Sn satisfies Goal
```

因此：

```text
S0
  ↓ A1
S1
  ↓ A2
S2
  ↓ A3
S3
  ↓
Goal
```

Agent Planning 的核心就是：

> **寻找一条从 Current State 到 Goal State 的有效路径。**

---

# 6. Planning 的基本输入

一个 Planner 至少需要五类信息。

## 6.1 Goal

例如：

```text
分析订单异常
```

---

## 6.2 Current State

例如：

```text
orderId = 12345
status = PAID
shippingStatus = WAITING
```

---

## 6.3 Available Tools

例如：

```text
getOrder
getInventory
getWarehouse
getShipping
```

---

## 6.4 Constraints

例如：

```text
Read Only
Maximum 10 tool calls
Timeout 30s
No customer PII
```

---

## 6.5 Success Criteria

例如：

```text
必须给出：
Root Cause
Evidence
Recommendation
```

于是：

```text
Planner Input
=
Goal
+
State
+
Tools
+
Constraints
+
Success Criteria
```

---

# 7. Task Decomposition：Planning 的第一步

复杂任务通常无法一步完成。

例如：

> “帮我设计一个高并发秒杀系统。”

可以分解：

```text
Goal
│
├── 1. 需求分析
│
├── 2. 流量模型
│
├── 3. 架构设计
│
├── 4. 数据库设计
│
├── 5. Redis 设计
│
├── 6. MQ 设计
│
├── 7. 限流设计
│
├── 8. 库存一致性
│
└── 9. 故障处理
```

这就是：

> **Task Decomposition。**

---

# 8. Top-Down Decomposition

一种经典方式是：

```text
Goal
 ↓
Sub Goals
 ↓
Tasks
 ↓
Actions
```

例如：

```text
设计秒杀系统
│
├── 流量控制
│   ├── Rate Limit
│   └── Queue
│
├── 库存
│   ├── Redis
│   └── DB
│
├── 一致性
│   ├── Transaction
│   └── MQ
│
└── Monitoring
    ├── Metrics
    └── Tracing
```

这种方式特别适合复杂 Enterprise Agent。

---

# 9. Hierarchical Planning

当任务非常复杂时，可以建立层次：

```text
Level 0
Goal

    ↓

Level 1
Sub Goal

    ↓

Level 2
Task

    ↓

Level 3
Action
```

例如：

```text
Level 0:
解决生产故障

Level 1:
定位性能问题

Level 2:
分析 API

Level 3:
query_latency()

Level 3:
query_error_rate()

Level 3:
query_trace()
```

这就是：

> **Hierarchical Planning。**

它最大的价值是：

> **控制复杂度。**

---

# 10. Plan Representation

Planning 的结果最好不要直接生成一段自然语言：

```text
先查询订单，然后看看库存，接着分析物流。
```

而应该结构化。

例如：

```json
{
  "goal": "Analyze order shipment delay",
  "tasks": [
    {
      "id": "T1",
      "action": "get_order",
      "arguments": {
        "orderId": "12345"
      }
    },
    {
      "id": "T2",
      "action": "get_inventory",
      "dependsOn": ["T1"]
    },
    {
      "id": "T3",
      "action": "get_shipping",
      "dependsOn": ["T1"]
    },
    {
      "id": "T4",
      "action": "analyze",
      "dependsOn": ["T2", "T3"]
    }
  ]
}
```

这样 Runtime 才能真正执行。

---

# 11. Plan 是一个执行图

上面的 Plan 可以表示为：

```text
              T1
          get_order
           /     \
          ↓       ↓
        T2         T3
     inventory   shipping
          \       /
           ↓     ↓
              T4
            analyze
```

本质上是：

> **Directed Acyclic Graph，DAG。**

这比简单的：

```text
A → B → C → D
```

更加适合 Agent。

---

# 12. 为什么 DAG 很重要？

因为很多任务之间没有依赖关系。

例如：

```text
查询 CPU
查询 Memory
查询 DB
查询 Redis
```

这些任务可以并行：

```text
        ┌── CPU
        │
        ├── Memory
Agent ──┼── DB
        │
        └── Redis
```

如果使用串行：

```text
CPU
 ↓
Memory
 ↓
DB
 ↓
Redis
```

总耗时：

```text
T = T1 + T2 + T3 + T4
```

如果并行：

```text
T ≈ max(T1,T2,T3,T4)
```

这对 Agent Performance 非常重要。

---

# 13. Planning 与 Parallel Execution

一个成熟的 Planner 应该能够判断：

```text
Can Run In Parallel?
```

例如：

```text
T1 → T2
T1 → T3

T2 与 T3 没有依赖
```

那么：

```text
        T1
       /  \
      ↓    ↓
     T2    T3
      \    /
       ↓  ↓
        T4
```

Runtime：

```java
CompletableFuture<ToolResult> t2 =
    executor.submit(task2);

CompletableFuture<ToolResult> t3 =
    executor.submit(task3);

CompletableFuture.allOf(t2, t3)
    .thenRun(task4);
```

这样 Planning 就真正与 Distributed Systems Engineering 联系起来了。

---

# 14. Plan-and-Execute

一种非常经典的 Agent Planning Architecture 是：

```text
Planner
   ↓
Plan
   ↓
Executor
   ↓
Result
```

流程：

```text
User Goal
   ↓
Planner
   ↓
Generate Plan
   ↓
Executor
   ↓
Execute Tasks
   ↓
Final Result
```

例如：

```text
Planner:

1. get_order
2. get_inventory
3. get_shipping
4. analyze
```

Executor：

```text
执行 T1
执行 T2
执行 T3
执行 T4
```

---

# 15. Plan-and-Execute 的优点

最大优点：

> **Planning 与 Execution 解耦。**

可以：

```text
Planner
```

专注：

```text
我要做什么？
```

而：

```text
Executor
```

专注：

```text
怎么执行？
```

类似传统软件架构：

```text
Controller
   ↓
Service
   ↓
Repository
```

Agent 也可以：

```text
Planner
   ↓
Execution Engine
   ↓
Tools
```

---

# 16. Plan-and-Execute 的缺点

问题也很明显。

如果：

```text
Planner
 ↓
一次性生成完整 Plan
```

而执行过程中环境发生变化：

```text
T1
 ↓
T2
 ↓
发现异常
```

原计划可能已经失效。

例如：

```text
Plan:
1. 查询库存
2. 创建订单
3. 支付
```

但执行 T1 后发现：

```text
库存 = 0
```

那么：

```text
T2 Create Order
```

已经没有意义。

所以：

> **静态 Plan 无法完全解决动态环境问题。**

这就引出了：

# Re-Planning

---

# 17. Re-Planning：动态重新规划

成熟 Agent 不应该：

```text
Plan Once
 ↓
Execute Forever
```

而应该：

```text
Plan
 ↓
Execute
 ↓
Observe
 ↓
Plan Still Valid?
 ├── Yes → Continue
 └── No  → Re-Plan
```

例如：

```text
Initial Plan
│
├── T1
├── T2
├── T3
└── T4
```

执行 T1：

```text
Observation:
Inventory = 0
```

Planner：

```text
Original Plan Invalid
```

重新生成：

```text
New Plan
│
├── Notify User
├── Find Alternative Warehouse
└── Recommend Restock
```

这就是：

> **Dynamic Planning。**

---

# 18. ReAct 与 Planning 的关系

ReAct：

```text
Reason
 ↓
Act
 ↓
Observe
 ↓
Reason
 ↓
Act
```

Planning：

```text
Goal
 ↓
Plan
 ↓
Execute
 ↓
Observe
 ↓
Re-Plan
```

二者并不是互斥的。

可以组合：

```text
High-Level Planner
        ↓
      Plan
        ↓
     ReAct Loop
        ↓
     Tool Calls
        ↓
    Observations
        ↓
    Re-Planning
```

因此现代 Agent 往往是：

> **Planning + ReAct，而不是二选一。**

---

# 19. Planning 的三个层次

可以把现代 Agent Planning 分成三个层次。

## Level 1：Reactive

```text
Observe
 ↓
Act
```

没有明确长期计划。

---

## Level 2：Local Planning

```text
Think
 ↓
Next Action
 ↓
Execute
```

每一步只考虑下一步。

---

## Level 3：Global Planning

```text
Goal
 ↓
Task Decomposition
 ↓
Plan
 ↓
Execution
 ↓
Re-Planning
```

复杂 Agent 通常需要 Level 3。

---

# 20. Static Planning vs Dynamic Planning

| 维度       | Static Planning | Dynamic Planning |
| -------- | --------------- | ---------------- |
| Plan     | 一次生成            | 动态调整             |
| 环境变化     | 处理较弱            | 处理较强             |
| 可预测性     | 高               | 中                |
| 灵活性      | 低               | 高                |
| Token 成本 | 较低              | 较高               |
| 适合       | 稳定流程            | 不确定任务            |

实际生产环境通常采用：

> **Static Plan + Dynamic Re-Planning**

即：

```text
先规划
 ↓
执行
 ↓
必要时重新规划
```

而不是每执行一步都重新规划。

---

# 21. Planning State

要支持 Re-Planning，就必须保存状态。

例如：

```java
public class PlanningState {

    private String goal;

    private Plan currentPlan;

    private Map<String, TaskState> tasks;

    private Map<String, Object> observations;

    private int iteration;

    private PlanningStatus status;
}
```

Task 状态：

```text
PENDING
RUNNING
SUCCESS
FAILED
SKIPPED
BLOCKED
```

例如：

```text
T1 SUCCESS
T2 SUCCESS
T3 FAILED
T4 BLOCKED
```

Planner 就可以根据：

```text
Current State
```

重新规划。

---

# 22. Plan State Machine

可以设计：

```text
              INIT
               │
               ↓
            PLANNING
               │
               ↓
           VALIDATING
               │
               ↓
           EXECUTING
               │
       ┌───────┴────────┐
       ↓                ↓
   OBSERVING          FAILED
       │                │
       ↓                ↓
    EVALUATE        RE-PLANNING
       │                │
   ┌───┴───┐            │
   ↓       ↓            │
DONE    REPLAN ─────────┘
```

这其实已经非常接近一个：

> **Agent Workflow Engine。**

---

# 23. Plan Validation

LLM 生成 Plan 后不能直接执行。

例如模型生成：

```text
1. Delete production database
2. Restart all services
```

Runtime 必须先验证：

```text
Plan Validation
```

至少检查：

```text
Tool Exists?
Arguments Valid?
Dependencies Valid?
Permission Valid?
Risk Level?
Timeout?
Budget?
```

例如：

```text
Task:
delete_database

Risk:
CRITICAL

Policy:
Human Approval Required
```

因此：

```text
LLM Plan
 ↓
Validator
 ↓
Policy
 ↓
Execution
```

而不是：

```text
LLM Plan
 ↓
Execute
```

---

# 24. Planning Constraints

现实任务通常存在很多约束。

例如：

```text
必须在 30 秒内完成
只能调用 10 次 Tool
不能访问 PII
不能修改数据库
成本不能超过 $0.50
必须人工审批高风险操作
```

可以表示：

```json
{
  "maxIterations": 10,
  "timeoutSeconds": 30,
  "maxCost": 0.5,
  "readOnly": true,
  "humanApprovalRequired": true
}
```

Planner 需要把这些约束考虑进去。

因此：

> **Planning 不是单纯寻找“能完成任务”的路径，而是寻找“满足约束的可执行路径”。**

---

# 25. Cost-Aware Planning

假设 Agent 有三个工具：

```text
Tool A
Cost = $0.001

Tool B
Cost = $0.01

Tool C
Cost = $0.1
```

如果三个 Tool 都能获得相同信息：

```text
A > B > C
```

Planner 应该倾向选择：

```text
A
```

而不是：

```text
C
```

因此可以定义：

```text
Plan Score
=
Success Probability
-
Cost
-
Latency
-
Risk
```

概念上：

```text
Score =
α × Success
- β × Cost
- γ × Latency
- δ × Risk
```

这就是：

> **Cost-Aware Planning。**

---

# 26. Planning 与 Token Cost

Agent Planning 很容易出现：

```text
Planning
 ↓
大量思考
 ↓
大量 Token
```

例如：

```text
Goal
 ↓
Plan
 ↓
Detailed Plan
 ↓
Detailed Sub Plan
 ↓
Re-Plan
 ↓
Reflection
```

最后可能：

```text
Token > Tool Cost
```

因此 Production Agent 应该控制：

```text
Planning Depth
Planning Frequency
Context Size
Plan Verbosity
```

尤其不要让 Agent 对简单任务进行过度规划。

---

# 27. Adaptive Planning

一个非常实用的思想是：

> **任务越简单，Planning 越轻；任务越复杂，Planning 越深。**

例如：

### 简单任务

```text
“查询订单 12345。”

→ 直接 Tool Call
```

不需要：

```text
Planning
Reflection
Re-Planning
```

---

### 中等任务

```text
“分析订单为什么没有发货。”

→ Short Plan
```

---

### 复杂任务

```text
“分析生产环境 API 延迟升高的根因。”

→ Hierarchical Planning
→ Parallel Execution
→ Re-Planning
→ Reflection
```

因此：

```text
Planning Depth ∝ Task Complexity
```

这是非常重要的 Production Design Principle。

---

# 28. Planning Heuristic

并不是所有任务都需要最优解。

Agent 通常应该寻找：

> **足够好的可执行方案。**

例如：

```text
Goal:
找到性能下降原因。
```

不一定需要检查：

```text
CPU
Memory
GC
Database
Redis
Kafka
Network
Disk
Kernel
DNS
...
```

可能只需要：

```text
Latency
 ↓
Trace
 ↓
Database
```

就已经找到 Root Cause。

因此：

> **好的 Planner 不是调用最多工具，而是用最少的有效动作完成任务。**

---

# 29. Planning Termination

Agent 必须知道：

> **什么时候应该停止规划和执行？**

常见终止条件：

```text
Goal Achieved
No More Useful Actions
Maximum Iterations
Timeout
Budget Exceeded
Fatal Error
Human Rejection
```

例如：

```java
if (goalAchieved(state)) {
    return COMPLETED;
}

if (state.getIteration() >= 10) {
    return MAX_ITERATION;
}

if (timeoutExceeded(state)) {
    return TIMEOUT;
}
```

否则 Agent 可能陷入：

```text
Tool A
 ↓
Tool B
 ↓
Tool A
 ↓
Tool B
 ↓
...
```

---

# 30. Planning Loop

一个 Production Planning Loop 可以抽象为：

```java
while (!state.isTerminal()) {

    if (needPlan(state)) {

        Plan plan =
            planner.createPlan(state);

        validator.validate(plan);

        state.setPlan(plan);
    }

    Task task =
        scheduler.nextTask(state);

    ToolResult result =
        executor.execute(task);

    state.update(result);

    if (needRePlan(state)) {
        state.invalidatePlan();
    }
}
```

这已经是一个真正的：

> **Agent Planning Runtime。**

---

# 31. Planner 与 Executor 解耦

推荐架构：

```text
              Agent
                │
                ↓
             Planner
                │
                ↓
               Plan
                │
                ↓
             Validator
                │
                ↓
             Scheduler
                │
                ↓
             Executor
                │
                ↓
              Tools
```

每一层职责清晰：

### Planner

```text
What should we do?
```

### Validator

```text
Is this plan allowed?
```

### Scheduler

```text
What can execute now?
```

### Executor

```text
How do we execute?
```

### Tool

```text
Perform the actual action.
```

这种架构非常适合 Java 后端工程师。

---

# 32. Scheduler：Planning 与并发控制的连接点

Plan 产生之后，并不意味着所有任务立即执行。

Scheduler 可以根据：

```text
Dependency
Priority
Resource
Concurrency Limit
Risk
Deadline
```

选择任务。

例如：

```text
Task Queue

T1 READY
T2 BLOCKED
T3 READY
T4 BLOCKED
```

Scheduler：

```text
T1
T3
```

可以并行执行。

完成：

```text
T1 SUCCESS
T3 SUCCESS
```

之后：

```text
T2 READY
T4 READY
```

这种设计与：

```text
Distributed Task Scheduler
```

非常接近。

---

# 33. Priority Planning

有些任务应该优先执行。

例如 Incident Agent：

```text
查询 latency
Priority = HIGH

查询 CPU
Priority = HIGH

查询日志
Priority = MEDIUM

查询部署历史
Priority = LOW
```

Scheduler：

```text
Priority Queue
```

可以优先：

```text
Latency
CPU
```

这样能够降低：

```text
Time To Diagnosis
```

---

# 34. Planning Failure

Planning 也可能失败。

例如：

```text
Goal:
自动修复生产环境问题
```

但 Agent 没有：

```text
Kubernetes Permission
Deployment Tool
Rollback Tool
```

那么 Planner 可能生成：

```text
restart_service()
```

但 Tool 不存在。

因此 Planning Failure 包括：

```text
No Available Tool
No Valid Plan
Unsatisfied Dependency
Permission Denied
Constraint Conflict
Insufficient Information
```

例如：

```text
Goal requires write access.

Current policy:
Read-only.

Result:
No valid plan.
```

这比：

```text
LLM hallucinate a solution
```

更加安全。

---

# 35. Planner 应该允许“不知道”

一个成熟 Agent 不应该总是生成 Plan。

例如：

```text
用户：
为什么服务器变慢？
```

但 Agent 没有：

```text
Metrics Tool
Log Tool
Trace Tool
```

正确行为：

```text
Insufficient Information
```

而不是：

```text
服务器可能是 CPU 太高。
```

因此：

> **Planner 必须允许 No-Plan / Insufficient-Information 状态。**

这是提高 Agent Reliability 的关键。

---

# 36. Planning 与 Reflection

Planning：

```text
我要怎么做？
```

Reflection：

```text
我刚才做得对吗？
```

二者可以组合：

```text
Goal
 ↓
Plan
 ↓
Execute
 ↓
Reflect
 ↓
Plan Valid?
 ├── Yes → Continue
 └── No → Re-Plan
```

例如：

```text
Plan:
查询 DB

Result:
DB latency 正常

Reflection:
DB 不是主要原因

Re-Plan:
检查 Redis
```

这让 Agent 不只是：

> “执行计划。”

而是：

> **验证计划假设。**

---

# 37. Planning 与 Hypothesis

在故障诊断类 Agent 中，Planning 可以进一步变成：

```text
Hypothesis
 ↓
Evidence
 ↓
Verification
 ↓
Conclusion
```

例如：

```text
Hypothesis 1:
Database latency increased.

Action:
query_db_metrics()

Result:
Normal.

Confidence:
Low.
```

然后：

```text
Hypothesis 2:
Redis latency increased.

Action:
query_redis_metrics()

Result:
High latency.

Confidence:
High.
```

最终：

```text
Root Cause:
Redis latency increase.
```

这已经接近：

> **AI-driven Root Cause Analysis。**

---

# 38. Planning 与 Tree Search

复杂任务可以不只生成一条 Plan：

```text
             Goal
           /  |  \
          A   B   C
         / \     / \
        A1 A2   C1 C2
```

Agent 可以探索多个候选计划：

```text
Plan A
Plan B
Plan C
```

然后根据：

```text
Cost
Probability
Risk
Expected Value
```

选择：

```text
Best Plan
```

这属于更高级的：

> **Search-Based Planning。**

---

# 39. Planning 与 Tree of Thoughts

Tree of Thoughts 可以理解为：

```text
Problem
  │
  ├── Thought A
  │    ├── A1
  │    └── A2
  │
  ├── Thought B
  │    ├── B1
  │    └── B2
  │
  └── Thought C
```

Planner 可以：

```text
Generate Candidates
 ↓
Evaluate
 ↓
Select
 ↓
Execute
```

但在 Production Agent 中必须注意：

> **搜索空间可能快速爆炸。**

因此需要：

```text
Beam Width
Depth Limit
Cost Limit
Early Stop
```

---

# 40. Planning 与 MCTS

在更复杂的 Agent Research 中，可以进一步使用：

> **Monte Carlo Tree Search**

基本思想：

```text
Current State
      ↓
Generate Actions
      ↓
Explore
      ↓
Evaluate
      ↓
Select
      ↓
Expand
      ↓
Repeat
```

它适合：

```text
Game
Complex Search
Planning
Long-Horizon Decision
```

但对于企业 Agent：

```text
Token Cost
Latency
Implementation Complexity
```

都需要考虑。

所以并不是所有 Agent 都需要 MCTS。

---

# 41. Long-Horizon Planning

简单 Agent：

```text
Goal
 ↓
3 steps
```

复杂 Agent：

```text
Goal
 ↓
50 steps
 ↓
100 steps
```

步骤越长：

```text
Error Probability ↑
State Drift ↑
Cost ↑
```

例如假设每一步成功率：

```text
p = 0.95
```

10 个步骤全部成功：

```text
0.95^10 ≈ 59.9%
```

20 个步骤：

```text
0.95^20 ≈ 35.8%
```

这说明：

> **Long-Horizon Agent 最大的问题之一是错误累积。**

因此应该：

```text
Plan
 ↓
Execute Few Steps
 ↓
Verify
 ↓
Re-Plan
```

而不是：

```text
Plan 100 Steps
 ↓
Blind Execution
```

---

# 42. Planning 的可靠性设计

Production Agent 可以采用：

```text
Short-Horizon Planning
+
Frequent Verification
+
Re-Planning
```

例如：

```text
Plan 3 steps
 ↓
Execute
 ↓
Verify
 ↓
Plan next 3 steps
```

这类似：

> **Model Predictive Control**

即：

```text
只规划未来一部分
执行后重新观察环境
再继续规划
```

这比一次生成几十步计划更加可靠。

---

# 43. Planning 的生产级架构

一个比较完整的 Agent Planning Platform：

```text
                         User
                           │
                           ↓
                    ┌─────────────┐
                    │ Agent API   │
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │ Goal Parser │
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │   Planner   │
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │ Plan Model  │
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │  Validator  │
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │  Scheduler  │
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │  Executor   │
                    └──────┬──────┘
                           ↓
                  ┌────────┴────────┐
                  ↓        ↓        ↓
                Tool A   Tool B   Tool C
                  │        │        │
                  └────────┼────────┘
                           ↓
                      Observation
                           ↓
                    ┌─────────────┐
                    │  Evaluator  │
                    └──────┬──────┘
                           │
                 ┌─────────┴─────────┐
                 ↓                   ↓
              Continue             Re-Plan
                 │                   │
                 └─────────┬─────────┘
                           ↓
                          LLM
```

这已经不是一个简单的：

```text
LLM API
```

而是：

> **Agent Planning Runtime。**

---

# 44. Java 中实现 Planner

可以定义：

```java
public interface Planner {

    Plan createPlan(
        PlanningContext context
    );
}
```

Plan：

```java
public class Plan {

    private String goal;

    private List<PlanTask> tasks;

    private List<Constraint> constraints;
}
```

Task：

```java
public class PlanTask {

    private String id;

    private String tool;

    private Map<String, Object> arguments;

    private List<String> dependencies;

    private TaskStatus status;
}
```

例如：

```java
Plan plan = new Plan();

plan.addTask(
    new PlanTask(
        "T1",
        "get_order",
        Map.of("orderId", "12345"),
        List.of()
    )
);

plan.addTask(
    new PlanTask(
        "T2",
        "get_inventory",
        Map.of(),
        List.of("T1")
    )
);
```

---

# 45. Java Planner Runtime

可以实现：

```java
public class PlannerRuntime {

    public AgentResult execute(
            PlanningContext context) {

        while (!context.isFinished()) {

            if (context.needPlanning()) {

                Plan plan =
                    planner.createPlan(context);

                validator.validate(plan);

                context.setPlan(plan);
            }

            List<PlanTask> readyTasks =
                scheduler.getReadyTasks(
                    context.getPlan()
                );

            executeParallel(readyTasks);

            context.update();

            if (evaluator.shouldRePlan(context)) {
                context.invalidatePlan();
            }
        }

        return context.result();
    }
}
```

核心思想就是：

```text
Plan
 ↓
Schedule
 ↓
Execute
 ↓
Observe
 ↓
Evaluate
 ↓
Re-Plan
```

---

# 46. Planning 与传统 Distributed Systems

对于 Java 后端工程师来说，Planning 有很多熟悉的概念。

| Agent Planning | Distributed Systems   |
| -------------- | --------------------- |
| Plan           | Workflow              |
| Task           | Job                   |
| Dependency     | DAG                   |
| Scheduler      | Task Scheduler        |
| State          | State Store           |
| Tool           | Service/API           |
| Retry          | Retry                 |
| Timeout        | Timeout               |
| Re-Planning    | Dynamic Workflow      |
| Observation    | Event                 |
| Guardrail      | Policy                |
| Execution      | Distributed Execution |

因此 Agent Planning 并不是完全新的工程范式。

更准确地说：

> **它是在传统软件执行系统之上增加了 LLM-driven Decision Making。**

---

# 47. Planning 与 Kubernetes 的类比

甚至可以做一个非常有意思的类比。

Kubernetes：

```text
Desired State
      ↓
Controller
      ↓
Observe Actual State
      ↓
Reconcile
      ↓
Desired State
```

Agent Planning：

```text
Goal
 ↓
Planner
 ↓
Plan
 ↓
Execute
 ↓
Observe
 ↓
Re-Plan
```

二者都有：

```text
Desired State
+
Current State
+
Controller
+
Reconciliation
```

因此可以把 Agent Planner 理解成：

> **面向任务的智能 Controller。**

---

# 48. Planning 的真正难点

表面上看：

```text
Planning = 让 LLM 生成 Plan
```

实际上真正困难的是：

### 1. Plan 是否正确？

```text
Plan Correctness
```

### 2. Plan 是否可执行？

```text
Executability
```

### 3. Plan 是否满足权限？

```text
Authorization
```

### 4. Plan 是否高效？

```text
Efficiency
```

### 5. Plan 是否能够适应环境变化？

```text
Adaptability
```

### 6. Plan 是否最终完成目标？

```text
Task Success
```

所以：

> **Agent Planning 的核心不是“生成计划”，而是“生成并持续维护一个可执行计划”。**

---

# 49. 一个完整的 Planning 思维模型

可以把整个过程抽象为：

```text
                   Goal
                    │
                    ↓
              Understand Goal
                    │
                    ↓
              Analyze State
                    │
                    ↓
             Available Tools
                    │
                    ↓
                Planning
                    │
                    ↓
             Task Decomposition
                    │
                    ↓
              Dependency Graph
                    │
                    ↓
              Plan Validation
                    │
                    ↓
                Scheduling
                    │
                    ↓
                Execution
                    │
                    ↓
               Observation
                    │
                    ↓
                Evaluation
                    │
             ┌──────┴──────┐
             ↓             ↓
           Done          Re-Plan
             │             │
             │             └──────→ Planning
             ↓
           Result
```

这就是一个完整的：

> **Closed-Loop Planning System。**

---

# 50. Planning 最终应该解决什么问题？

一个成熟 Agent Planner 最终需要回答六个问题：

```text
1. What?
   我要完成什么？

2. Why?
   为什么执行这个任务？

3. How?
   应该怎么做？

4. When?
   什么顺序执行？

5. What if?
   如果执行失败怎么办？

6. When stop?
   什么时候结束？
```

如果一个 Agent 能够回答这六个问题，它才真正具备：

> **Task-Level Intelligence。**

---

# 51. 总结

Tool Calling 解决：

```text
Agent 能做什么？
```

Planning 解决：

```text
Agent 应该怎么做？
```

二者结合：

```text
                 Goal
                   │
                   ↓
               Planning
                   │
                   ↓
                 Plan
                   │
                   ↓
              Tool Calling
                   │
                   ↓
               Execution
                   │
                   ↓
              Observation
                   │
                   ↓
              Evaluation
                   │
             ┌─────┴─────┐
             ↓           ↓
          Complete    Re-Plan
                         │
                         └────→ Planning
```

因此可以把 Agent Planning 总结成：

```text
Planning
=
Goal Understanding
+
Task Decomposition
+
Dependency Management
+
Action Selection
+
Constraint Handling
+
Scheduling
+
Execution
+
Observation
+
Re-Planning
+
Termination
```

而 Production Agent Planning 更进一步：

```text
Production Planning
=
Planning
+
Validation
+
Policy
+
Concurrency
+
Timeout
+
Retry
+
Cost Control
+
State Management
+
Observability
+
Human Approval
```

最终，一个成熟的 Agent 不应该只是：

```text
LLM
 ↓
Tool
```

而应该是：

```text
                     Goal
                      │
                      ↓
                ┌───────────┐
                │  Planner  │
                └─────┬─────┘
                      ↓
                    Plan
                      │
                ┌─────┴─────┐
                ↓           ↓
             Validate    Schedule
                │           │
                └─────┬─────┘
                      ↓
                  Execute
                      │
              ┌───────┼───────┐
              ↓       ↓       ↓
            Tool A  Tool B  Tool C
              │       │       │
              └───────┼───────┘
                      ↓
                 Observation
                      │
                      ↓
                  Evaluate
                      │
              ┌───────┴───────┐
              ↓               ↓
          Goal Achieved     Re-Plan
              │               │
              ↓               └──────→ Planner
           Result
```

**Planning 的核心思想可以浓缩成一句话：**

> **不要让 Agent 只思考“下一步调用哪个 Tool”，而要让它理解“当前目标是什么、为了达到目标需要完成哪些任务、任务之间有什么依赖、执行过程中获得的新信息是否改变了原来的计划，以及什么时候已经足够完成任务”。**

这正是 Agent 从 **Tool User** 走向 **Autonomous Task Executor** 的关键一步。

---
