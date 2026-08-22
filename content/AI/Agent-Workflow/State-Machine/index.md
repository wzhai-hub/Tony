---
title: State Machine 深度技术博客：从有限状态机到 AI Agent 的状态驱动架构
# tags:
#   - nodejs
date: '2026-08-08'
summary: LLM 负责“智能决策”，State Machine 负责“确定性控制”。
---
# State Machine 深度技术博客：从有限状态机到 AI Agent 的状态驱动架构

> **摘要**
> State Machine（状态机）并不是一个只存在于编译原理、游戏开发或传统工作流系统中的老技术。随着 AI Agent 从“单轮问答”走向“长任务、自主决策、多 Agent 协作、工具调用和可恢复执行”，State Machine 正重新成为 Agent Runtime 的核心抽象之一。
>
> 本文从有限状态机（FSM）的数学模型出发，深入分析 State、Event、Transition、Guard、Action、Context、Persistence、Recovery 等核心概念，并进一步讨论 State Machine 如何与 LLM、Tool Calling、Workflow、Agent Loop、Memory、Human-in-the-loop 结合，最终构建一个可观测、可恢复、可验证的生产级 AI Agent Runtime。

---

# 1. 为什么 AI Agent 时代重新需要 State Machine？

很多人第一次接触 Agent 时，会认为 Agent 的核心结构非常简单：

```text
User
  ↓
LLM
  ↓
Tool
  ↓
LLM
  ↓
Tool
  ↓
Final Answer
```

于是最简单的 Agent Loop 通常被实现成：

```java
while (!finished) {
    response = llm.chat(context);

    if (response.hasToolCall()) {
        executeTool(response);
    } else {
        finished = true;
    }
}
```

这个模型对于 Demo 足够，但进入生产环境后，很快会遇到问题：

* Agent 当前到底处于什么阶段？
* Tool 调用失败后应该怎么办？
* 网络超时应该重试还是终止？
* 用户中途修改需求怎么办？
* 一个任务执行到 70% 后服务重启怎么办？
* Agent 如何从上一次执行位置恢复？
* 哪些状态允许调用支付、删除、发布等危险工具？
* 多个 Agent 如何协作？
* 如何保证状态不会非法跳转？
* 如何审计 Agent 为什么做出了某个动作？
* 如何测试 Agent 的所有执行路径？

这些问题本质上已经不是单纯的 LLM 问题。

它们属于：

> **State Management + Workflow Orchestration + Control Flow**

也就是状态管理和流程控制问题。

因此，一个生产级 Agent 往往不能只是：

```text
LLM → Tool → LLM → Tool
```

而应该逐渐演化成：

```text
                    ┌──────────────┐
                    │     LLM      │
                    └──────┬───────┘
                           │ Decision
                           ▼
┌──────────┐        ┌──────────────┐
│  Event   │───────▶│ State Machine│
└──────────┘        └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
          Tool Call      Memory       Human
              │            │            │
              └────────────┼────────────┘
                           ▼
                        Context
```

这里最重要的思想是：

> **LLM 负责“智能决策”，State Machine 负责“确定性控制”。**

这两者不是竞争关系，而是互补关系。

---

# 2. 什么是 State Machine？

State Machine 的核心思想非常简单：

> 一个系统在任意时刻处于某个 State，当 Event 发生时，根据 Transition Rules 决定是否进入另一个 State。

最基本的数学定义可以表示为：

[
M = (S, E, \delta, s_0, F)
]

其中：

* (S)：State 集合
* (E)：Event 集合
* (\delta)：状态转移函数
* (s_0)：Initial State
* (F)：Final States

例如一个订单系统：

```text
CREATED
   │
   │ PAY
   ▼
PAID
   │
   │ SHIP
   ▼
SHIPPED
   │
   │ RECEIVE
   ▼
COMPLETED
```

状态集合：

```text
CREATED
PAID
SHIPPED
COMPLETED
CANCELLED
```

事件集合：

```text
PAY
SHIP
RECEIVE
CANCEL
```

状态转移可以定义为：

```text
CREATED + PAY
    → PAID

PAID + SHIP
    → SHIPPED

SHIPPED + RECEIVE
    → COMPLETED
```

状态机真正重要的地方不是“状态”。

而是：

> **系统只能通过合法 Transition 改变状态。**

例如：

```text
CREATED → COMPLETED
```

可能就是非法的。

State Machine 因此天然提供了一种：

> **对系统行为进行约束的机制。**

---

# 3. State Machine 的六个核心元素

一个真正可用于生产系统的 State Machine，至少需要理解六个概念：

```text
State
Event
Transition
Guard
Action
Context
```

---

## 3.1 State

State 表示系统当前处于什么阶段。

例如 AI Agent：

```text
IDLE
PLANNING
EXECUTING
WAITING_TOOL
WAITING_HUMAN
RETRYING
COMPLETED
FAILED
```

State 不应该只是一个字符串。

它实际上代表：

> **系统当前允许做什么，以及禁止做什么。**

例如：

```text
WAITING_HUMAN
```

意味着：

```text
允许：
    接收用户输入
    恢复任务

禁止：
    自动调用高风险 Tool
    自动修改核心业务数据
```

所以 State 实际上可以理解成：

[
State = Current\ Capability\ Boundary
]

即：

> 当前状态决定系统的能力边界。

---

# 4. Event

Event 是导致状态发生变化的输入。

例如：

```text
USER_REQUEST
PLAN_CREATED
TOOL_CALL
TOOL_SUCCESS
TOOL_FAILURE
TIMEOUT
USER_APPROVED
USER_REJECTED
CANCEL
RETRY
```

例如：

```text
PLANNING
    │
    │ PLAN_CREATED
    ▼
EXECUTING
```

或者：

```text
WAITING_HUMAN
    │
    │ USER_APPROVED
    ▼
EXECUTING
```

在 Agent 系统中，Event 通常比传统业务系统更加复杂，因为 Event 可能来自：

* User
* LLM
* Tool
* Message Queue
* Timer
* External API
* Another Agent
* Human Approval

因此生产级 Agent State Machine 通常实际上是一个：

> **Event-driven State Machine**

---

# 5. Transition

Transition 是：

> State + Event → New State

例如：

```text
EXECUTING + TOOL_SUCCESS
        ↓
EXECUTING
```

也可能：

```text
EXECUTING + TOOL_FAILURE
        ↓
RETRYING
```

或者：

```text
EXECUTING + TASK_COMPLETED
        ↓
COMPLETED
```

可以抽象成：

```java
record Transition(
    State from,
    Event event,
    State to
) {}
```

例如：

```java
new Transition(
    State.EXECUTING,
    Event.TOOL_FAILURE,
    State.RETRYING
);
```

---

# 6. Guard：状态机真正强大的地方

仅仅有 State + Event 还不够。

现实系统经常存在：

> 同一个 Event，在不同条件下进入不同状态。

这就是 Guard。

例如：

```text
EXECUTING
    │
    │ TOOL_FAILURE
    │
    ├── retryCount < 3 ───────▶ RETRYING
    │
    └── retryCount >= 3 ──────▶ FAILED
```

形式化表达：

[
Transition = (State, Event, Guard) \rightarrow State'
]

Java：

```java
if (event == TOOL_FAILURE) {

    if (retryCount < 3) {
        state = RETRYING;
    } else {
        state = FAILED;
    }
}
```

Guard 的重要意义在于：

> **把“什么时候允许发生某个行为”从业务代码中显式抽取出来。**

---

# 7. Action

Transition 发生之后，通常需要执行 Action。

例如：

```text
EXECUTING
    │
    │ TOOL_SUCCESS
    ▼
COMPLETED
    │
    ▼
sendNotification()
```

Action 可以包括：

```text
调用 API
发送消息
保存数据库
更新 Memory
写 Audit Log
触发 Event
启动 Timer
调用 Tool
```

因此完整模型变成：

```text
State
   +
Event
   +
Guard
   ↓
Transition
   ↓
Action
   ↓
New State
```

---

# 8. Context：State 和 Data 必须分离

一个常见设计错误是：

> 把所有业务数据都塞进 State。

例如：

```java
class AgentState {
    String state;
    String userName;
    String task;
    String toolResult;
    String conversation;
    String plan;
    int retryCount;
}
```

这里混合了两种完全不同的数据：

```text
State
Context
```

更好的设计：

```java
enum AgentState {
    IDLE,
    PLANNING,
    EXECUTING,
    WAITING_HUMAN,
    COMPLETED,
    FAILED
}
```

Context：

```java
class AgentContext {

    String taskId;

    String userRequest;

    List<String> plan;

    Map<String, Object> variables;

    List<ToolResult> toolResults;

    int retryCount;
}
```

即：

```text
State
    = 系统处于什么阶段

Context
    = 系统拥有的业务数据
```

这是非常重要的架构原则。

---

# 9. AI Agent 的 State Machine

现在把 State Machine 放入 Agent。

一个比较典型的 Agent 状态机：

```text
                 ┌─────────────┐
                 │    IDLE     │
                 └──────┬──────┘
                        │ USER_REQUEST
                        ▼
                 ┌─────────────┐
                 │  PLANNING   │
                 └──────┬──────┘
                        │ PLAN_CREATED
                        ▼
                 ┌─────────────┐
                 │  EXECUTING  │◀──────────┐
                 └──────┬──────┘           │
                        │                   │
              ┌─────────┴─────────┐         │
              │                   │         │
          TOOL_CALL           COMPLETE      │
              │                   │         │
              ▼                   ▼         │
       ┌─────────────┐      ┌───────────┐  │
       │WAITING_TOOL │      │ COMPLETED │  │
       └──────┬──────┘      └───────────┘  │
              │                            │
        TOOL_SUCCESS                       │
              │                            │
              └────────────────────────────┘
```

如果 Tool 失败：

```text
WAITING_TOOL
      │
      │ TOOL_FAILURE
      ▼
   RETRYING
      │
      ├── retry < 3 ─────▶ WAITING_TOOL
      │
      └── retry >= 3 ────▶ FAILED
```

这时候 Agent 已经不再是一个简单 Loop。

它变成了：

> **一个由事件驱动的状态转换系统。**

---

# 10. LLM 在 State Machine 中到底负责什么？

这是 Agent Architecture 中非常关键的问题。

很多系统让 LLM 决定一切：

```text
LLM：
    现在我要调用 Tool A
    调用 Tool A
    然后调用 Tool B
    然后直接完成
```

这种方式的问题是：

> LLM 是概率模型，不应该成为整个系统的确定性控制器。

更合理的架构是：

```text
               ┌──────────────┐
               │     LLM      │
               │  Reasoning   │
               └──────┬───────┘
                      │
                      │ Decision
                      ▼
               ┌──────────────┐
               │State Machine │
               │   Control    │
               └──────┬───────┘
                      │
             ┌────────┼────────┐
             ▼        ▼        ▼
           Tool     Memory    Human
```

LLM 负责：

```text
理解
推理
规划
选择
生成
```

State Machine 负责：

```text
约束
验证
执行
恢复
重试
生命周期
权限边界
```

一句话总结：

> **LLM 决定“想做什么”，State Machine 决定“现在允许做什么”。**

---

# 11. LLM 不应该直接修改 State

一个更加成熟的 Agent Runtime，不应该允许：

```text
LLM → state = COMPLETED
```

而应该：

```text
LLM
 ↓
Decision
 ↓
Event
 ↓
State Machine
 ↓
Transition Validation
 ↓
State Change
```

例如 LLM 输出：

```json
{
  "action": "CALL_TOOL",
  "tool": "search_customer"
}
```

Runtime 将其转换成：

```text
AGENT_DECISION
```

然后：

```text
EXECUTING
    +
AGENT_DECISION
    +
Guard
    ↓
CALL_TOOL
```

如果当前状态不允许调用这个 Tool：

```text
TransitionRejected
```

而不是让 LLM 直接执行。

这就是：

> **LLM-generated intent + deterministic execution**

---

# 12. State Machine 可以成为 Agent 的安全边界

这是 State Machine 在 AI 系统中的一个重要价值。

假设 Agent 有：

```text
search()
createOrder()
cancelOrder()
refund()
deleteAccount()
```

我们不希望 Agent 在任意状态都可以执行这些 Tool。

可以定义：

```text
PLANNING
    allowed:
        search

EXECUTING
    allowed:
        search
        createOrder

WAITING_HUMAN
    allowed:
        none

APPROVED
    allowed:
        createOrder
        refund
```

于是：

```text
State → Capability
```

可以形成：

```text
                 State
                   │
                   ▼
              Capability
                   │
                   ▼
                 Tools
```

因此 State Machine 可以承担部分：

> **Agent Authorization Boundary**

---

# 13. State Machine + Human-in-the-loop

企业级 Agent 经常不能完全自动执行。

例如：

```text
Agent
  ↓
分析贷款申请
  ↓
生成审批建议
  ↓
Human Approval
  ↓
执行
```

状态可以设计成：

```text
ANALYZING
    ↓
WAITING_APPROVAL
    ↓
APPROVED
    ↓
EXECUTING
    ↓
COMPLETED
```

或者：

```text
WAITING_APPROVAL
      │
      ├── APPROVED ──▶ EXECUTING
      │
      ├── REJECTED ──▶ FAILED
      │
      └── EXPIRED ───▶ CANCELLED
```

这比在代码中写：

```java
Thread.sleep(...)
```

等待用户审批要健壮得多。

---

# 14. State Machine + Persistence

如果 State Machine 只存在 JVM Memory：

```text
State
 ↓
JVM Memory
```

那么：

```text
Application Restart
       ↓
State Lost
       ↓
Agent Lost
```

生产环境必须持久化：

```text
Agent Runtime
      │
      ├── State
      ├── Context
      ├── Event
      └── Transition
             │
             ▼
        Persistent Store
```

例如：

```text
PostgreSQL
Redis
DynamoDB
MongoDB
Event Store
```

一个简单的数据结构：

```sql
CREATE TABLE agent_execution (
    task_id        VARCHAR(128) PRIMARY KEY,
    state          VARCHAR(64),
    version        BIGINT,
    context        JSONB,
    created_at     TIMESTAMP,
    updated_at     TIMESTAMP
);
```

其中：

```text
version
```

非常重要。

---

# 15. Optimistic Locking：防止状态覆盖

假设两个 Worker 同时处理：

```text
Worker A
    state = EXECUTING
    version = 10

Worker B
    state = EXECUTING
    version = 10
```

A：

```text
EXECUTING → COMPLETED
version = 11
```

B：

```text
EXECUTING → FAILED
```

如果没有并发控制：

```text
A 写入 COMPLETED
B 写入 FAILED
```

最终：

```text
FAILED
```

A 的状态被覆盖。

可以使用：

```sql
UPDATE agent_execution
SET state = 'COMPLETED',
    version = version + 1
WHERE task_id = ?
AND version = 10;
```

如果：

```text
affectedRows = 0
```

说明发生并发冲突。

因此：

> **State Machine + Versioning 是分布式 Agent Runtime 的基础能力。**

---

# 16. State Transition 应该具有原子性

一个完整 Transition 通常包含：

```text
读取 State
    ↓
验证 Event
    ↓
验证 Guard
    ↓
执行 Transition
    ↓
更新 State
    ↓
记录 Event
    ↓
触发 Action
```

如果这些操作不是原子的，就可能出现：

```text
State 更新成功
Action 执行失败
```

或者：

```text
Action 执行成功
State 更新失败
```

这会导致系统进入不一致状态。

因此可以采用：

```text
Transaction
+
Outbox Pattern
```

架构：

```text
┌─────────────────────────────┐
│        Database TX          │
│                             │
│  State Update               │
│  Event Record               │
│  Outbox Record              │
│                             │
└──────────────┬──────────────┘
               │
               ▼
            Kafka
               │
               ▼
          Action Worker
```

这样可以避免：

> State 已经变化，但是 Event 没有可靠发布。

---

# 17. State Machine + Event Sourcing

进一步可以使用 Event Sourcing。

传统模式：

```text
Current State
```

Event Sourcing：

```text
Event 1
Event 2
Event 3
Event 4
   ↓
Rebuild State
```

例如 Agent：

```text
USER_REQUEST
PLAN_CREATED
TOOL_CALLED
TOOL_SUCCESS
USER_APPROVED
TASK_COMPLETED
```

State 可以通过 Event Replay 得到：

```text
IDLE
 ↓
PLANNING
 ↓
EXECUTING
 ↓
WAITING_HUMAN
 ↓
EXECUTING
 ↓
COMPLETED
```

这带来非常强的审计能力。

你可以回答：

> Agent 为什么最终进入 COMPLETED？

因为：

```text
USER_REQUEST
→ PLAN_CREATED
→ TOOL_SUCCESS
→ USER_APPROVED
→ TASK_COMPLETED
```

对于金融、医疗、企业审批等场景，这种能力非常重要。

---

# 18. State Machine 与 Event Sourcing 的关系

两者并不是同一个东西。

State Machine：

> 描述“允许如何变化”。

Event Sourcing：

> 描述“变化历史如何保存”。

可以组合：

```text
             Events
               │
               ▼
        ┌──────────────┐
        │State Machine │
        └──────┬───────┘
               │
               ▼
          Current State
```

或者：

```text
Event Store
    │
    ▼
Replay
    │
    ▼
State Machine
    │
    ▼
Current State
```

因此：

```text
State Machine = Behavior Model

Event Sourcing = Persistence Model
```

---

# 19. FSM 的局限性

传统 FSM 很适合：

```text
状态数量较少
状态关系清晰
流程比较扁平
```

但是复杂 Agent 很快会产生：

```text
10 states
20 events
50 transitions
```

甚至：

```text
100+ states
```

这时会出现经典问题：

> **State Explosion**

例如：

```text
LOGIN
LOGIN_FAILED
LOGIN_RETRY
PASSWORD_EXPIRED
MFA_REQUIRED
MFA_FAILED
MFA_RETRY
ACCOUNT_LOCKED
...
```

状态数量迅速膨胀。

因此复杂系统需要更高级的状态模型。

---

# 20. Hierarchical State Machine

Hierarchical State Machine（HSM）允许状态嵌套。

例如：

```text
AGENT
│
├── PLANNING
│
├── EXECUTION
│   ├── TOOL_SELECTION
│   ├── TOOL_CALLING
│   ├── TOOL_RESULT
│   └── RETRYING
│
├── HUMAN_INTERACTION
│   ├── WAITING
│   └── APPROVED
│
└── TERMINATED
    ├── COMPLETED
    └── FAILED
```

这里：

```text
EXECUTION
```

本身就是一个 State Group。

这样可以减少大量重复 Transition。

---

# 21. Statechart：比 FSM 更适合复杂 Agent

Statechart 在传统 FSM 上进一步增加：

* Hierarchy
* Parallel States
* History
* Guard
* Action
* Event

例如一个 Agent 可以同时：

```text
EXECUTION
 ├── Tool Execution
 ├── Memory Update
 └── Observability
```

三个子系统并行运行。

抽象成：

```text
             EXECUTION
          /      |       \
         /       |        \
      TOOL     MEMORY    TRACE
       │         │         │
       ▼         ▼         ▼
    Running    Updating   Recording
```

这种模型非常适合：

> **复杂 Agent Runtime 和多步骤 Workflow。**

---

# 22. Parallel State：Multi-Agent 的基础

假设一个任务：

> “分析一家公司的投资价值。”

可以拆成：

```text
Research Agent
Financial Agent
News Agent
Risk Agent
```

它们可以并行执行：

```text
                 MASTER
                   │
          ┌────────┼────────┐
          ▼        ▼        ▼
       Research Financial  Risk
          │        │        │
          └────────┼────────┘
                   ▼
                Synthesis
                   │
                   ▼
                COMPLETED
```

这里实际上已经出现：

> **Parallel State Machine**

Master Agent 维护：

```text
research = COMPLETED
financial = COMPLETED
risk = RUNNING
```

只有：

```text
ALL_COMPLETED
```

才允许进入：

```text
SYNTHESIS
```

---

# 23. State Machine + Multi-Agent Collaboration

在 Multi-Agent 系统中，可以把每一个 Agent 看成一个 State Machine。

例如：

```text
                    Orchestrator
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      Research         Coding          Review
       Agent            Agent           Agent
          │              │              │
       State           State           State
       Machine         Machine         Machine
```

Agent 之间通过 Event 通信：

```text
RESEARCH_COMPLETED
        ↓
CODE_AGENT_START
        ↓
CODE_COMPLETED
        ↓
REVIEW_START
```

这时候：

> Agent Communication 本质上可以建立在 Event-driven State Transition 之上。

这比 Agent 之间直接共享大量自然语言 Context 更容易控制。

---

# 24. State Machine + LLM Memory

Agent Memory 也可以通过 State 驱动。

例如：

```text
PLANNING
    ↓
读取 Long-term Memory
    ↓
EXECUTING
    ↓
生成新的事实
    ↓
MEMORY_UPDATE
    ↓
COMPLETED
```

但是需要注意：

> Memory 不等于 State。

Memory：

```text
我上周已经分析过这个客户。
```

State：

```text
当前任务正在等待审批。
```

前者是知识。

后者是生命周期。

所以：

```text
Memory ≠ State
Context ≠ State
History ≠ State
```

它们应该被明确分离。

---

# 25. State Machine 与 Agent Loop 的区别

传统 Agent Loop：

```text
while(true) {
    think();
    act();
    observe();
}
```

State Machine：

```text
State
 ↓
Event
 ↓
Guard
 ↓
Transition
 ↓
Action
 ↓
State
```

两者可以结合：

```text
               ┌──────────────┐
               │ State Machine│
               └───────┬──────┘
                       │
                       ▼
                 EXECUTING
                       │
                       ▼
                     LLM
                       │
                 ┌─────┴─────┐
                 ▼           ▼
              Tool Call     Final
                 │
                 ▼
               Event
                 │
                 ▼
           State Machine
```

因此：

> Agent Loop 是执行机制，State Machine 是控制模型。

---

# 26. 一个生产级 Agent Runtime 的架构

一个成熟的 Agent Runtime 可以设计成：

```text
┌──────────────────────────────────────────┐
│                 API Layer                │
└─────────────────────┬────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────┐
│              Agent Runtime               │
│                                          │
│  ┌──────────────┐   ┌──────────────┐    │
│  │ State Machine│   │ Policy Engine│    │
│  └──────┬───────┘   └──────────────┘    │
│         │                                │
│         ▼                                │
│  ┌──────────────┐                        │
│  │ Agent Engine │                        │
│  └──────┬───────┘                        │
│         │                                │
│    ┌────┼────┬────────┐                  │
│    ▼    ▼    ▼        ▼                  │
│   LLM  Tool Memory  Human                │
│                                          │
└─────────────────────┬────────────────────┘
                      │
          ┌───────────┼─────────────┐
          ▼           ▼             ▼
      PostgreSQL     Redis        Kafka
          │
          ▼
      Event Store
```

其中：

```text
State Machine
```

是控制核心。

---

# 27. Java 实现一个简单 State Machine

可以定义：

```java
public enum AgentState {
    IDLE,
    PLANNING,
    EXECUTING,
    WAITING_HUMAN,
    COMPLETED,
    FAILED
}
```

Event：

```java
public enum AgentEvent {
    USER_REQUEST,
    PLAN_CREATED,
    TOOL_SUCCESS,
    TOOL_FAILURE,
    USER_APPROVED,
    USER_REJECTED,
    TASK_COMPLETED
}
```

Transition：

```java
public record Transition(
        AgentState from,
        AgentEvent event,
        AgentState to) {
}
```

然后建立 Transition Table：

```java
Map<String, AgentState> transitions = Map.of(
    "IDLE:USER_REQUEST", AgentState.PLANNING,
    "PLANNING:PLAN_CREATED", AgentState.EXECUTING,
    "EXECUTING:TOOL_FAILURE", AgentState.FAILED,
    "EXECUTING:TASK_COMPLETED", AgentState.COMPLETED
);
```

执行：

```java
public AgentState transition(
        AgentState current,
        AgentEvent event) {

    String key = current + ":" + event;

    AgentState next = transitions.get(key);

    if (next == null) {
        throw new IllegalStateException(
            "Illegal state transition: " + key
        );
    }

    return next;
}
```

核心原则就是：

> **非法 Transition 必须被拒绝。**

---

# 28. 不要把 State Machine 写成巨型 if-else

最初可能写成：

```java
if (state == IDLE) {
    if (event == USER_REQUEST) {
        state = PLANNING;
    }
} else if (state == PLANNING) {
    if (event == PLAN_CREATED) {
        state = EXECUTING;
    }
}
```

当系统扩大之后：

```text
1000+ lines
```

会非常难维护。

更好的方式是：

```text
Transition Table
```

或者：

```text
State Pattern
```

甚至直接使用：

```text
State Machine Framework
```

核心思想是：

> **让状态转移成为数据，而不是散落在业务代码里的控制流。**

---

# 29. State Pattern 与 State Machine 的区别

两者经常被混淆。

### State Pattern

主要解决：

> 当前对象在不同 State 下具有不同的行为。

例如：

```java
interface State {
    void handle(Context context);
}
```

不同 State：

```text
PendingState
RunningState
CompletedState
```

---

### State Machine

关注的是：

> State 之间如何按照 Event 进行合法转换。

```text
State
 ↓
Event
 ↓
Guard
 ↓
Transition
 ↓
State
```

所以：

```text
State Pattern
    = 行为封装

State Machine
    = 状态转换模型
```

两者完全可以组合。

---

# 30. Agent State Machine 的错误恢复

真正生产环境中，最重要的能力之一是：

> Failure Recovery。

例如：

```text
EXECUTING
    │
    │ Tool Timeout
    ▼
RETRYING
    │
    ├── retry < 3
    │       ↓
    │   EXECUTING
    │
    └── retry >= 3
            ↓
          FAILED
```

但是简单 Retry 仍然不够。

还需要：

```text
Exponential Backoff
Circuit Breaker
Timeout
Idempotency
Dead Letter Queue
Compensation
```

例如：

[
delay = min(cap,\ base \times 2^{retry})
]

第一次：

```text
1s
```

第二次：

```text
2s
```

第三次：

```text
4s
```

---

# 31. Compensation State

如果 Agent 执行了：

```text
Create Order
Charge Payment
Reserve Inventory
```

然后最后一步失败：

```text
Order Created
Payment Charged
Inventory Failed
```

不能简单：

```text
FAILED
```

因为系统可能已经产生副作用。

需要：

```text
EXECUTING
    ↓
PARTIAL_FAILURE
    ↓
COMPENSATING
    ↓
ROLLBACK_PAYMENT
    ↓
RELEASE_ORDER
    ↓
FAILED
```

这实际上是：

> **Saga + State Machine**

非常适合长事务 Agent。

---

# 32. State Machine + Saga

可以设计：

```text
PROCESSING
    │
    ├── Step A
    ├── Step B
    ├── Step C
    │
    ▼
SUCCESS
```

失败：

```text
PROCESSING
    ↓
STEP_C_FAILED
    ↓
COMPENSATING
    ↓
UNDO_B
    ↓
UNDO_A
    ↓
FAILED
```

因此复杂 Agent 的状态机不仅可以描述：

> “任务执行到哪里了”

还可以描述：

> “失败之后如何恢复系统一致性”。

---

# 33. Timeout 也是一种 Event

不要把 Timeout 当作特殊代码。

应该把它建模成 Event：

```text
EXECUTING
    │
    │ TIMEOUT
    ▼
RETRYING
```

或者：

```text
WAITING_HUMAN
    │
    │ APPROVAL_TIMEOUT
    ▼
EXPIRED
```

这样 Timer 只是 Event Producer：

```text
Timer
 ↓
TIMEOUT Event
 ↓
State Machine
```

整个系统就更加统一。

---

# 34. State Machine 的可观测性

Agent 系统必须回答：

```text
Current State?
Previous State?
Why transitioned?
Which Event?
Which Agent?
Which Tool?
How long?
How many retries?
```

因此建议记录：

```text
task_id
agent_id
from_state
event
to_state
reason
timestamp
duration
trace_id
span_id
```

例如：

```json
{
  "taskId": "task-1001",
  "from": "EXECUTING",
  "event": "TOOL_FAILURE",
  "to": "RETRYING",
  "reason": "TIMEOUT",
  "retry": 2,
  "traceId": "abc123"
}
```

这样可以把：

```text
State Machine
+
OpenTelemetry
```

结合起来。

---

# 35. State Machine 与 Distributed Tracing

一个 Agent Task 可能执行：

```text
API
 ↓
Orchestrator
 ↓
Planner Agent
 ↓
Search Tool
 ↓
Database
 ↓
LLM
 ↓
Reviewer Agent
```

可以建立：

```text
Trace
 ├── State Transition
 ├── LLM Call
 ├── Tool Call
 ├── DB Query
 └── Agent Communication
```

例如：

```text
Trace: task-1001

SPAN: STATE PLANNING
SPAN: LLM CALL
SPAN: STATE EXECUTING
SPAN: TOOL search
SPAN: STATE RETRYING
SPAN: TOOL search
SPAN: STATE COMPLETED
```

这对于 Debug Agent 非常重要。

---

# 36. State Machine 的可测试性

这是传统 LLM Agent 很难做到，而 State Machine 非常擅长的地方。

可以测试：

```text
IDLE
 + USER_REQUEST
 = PLANNING
```

```text
PLANNING
 + PLAN_CREATED
 = EXECUTING
```

```text
EXECUTING
 + TOOL_FAILURE
 = RETRYING
```

```text
RETRYING
 + RETRY_EXHAUSTED
 = FAILED
```

更进一步可以做：

> **State Transition Coverage**

测试所有：

```text
State × Event
```

组合。

例如：

```text
                 Event
             A    B    C    D
State A      ✓    ✓    X    X
State B      X    ✓    ✓    X
State C      X    X    ✓    ✓
```

这比单纯测试 LLM 输出可靠得多。

---

# 37. Property-based Testing

State Machine 还非常适合 Property-based Testing。

例如定义：

### Property 1

```text
COMPLETED
```

不能再进入：

```text
EXECUTING
```

### Property 2

```text
FAILED
```

不能调用支付 Tool。

### Property 3

```text
WAITING_HUMAN
```

不能自动执行高风险 Action。

### Property 4

```text
retryCount <= maxRetry
```

通过随机生成 Event Sequence：

```text
A → B → C → A → D → ...
```

检查：

```text
State Machine 是否始终满足约束。
```

这是一种非常强的工程能力。

---

# 38. LLM + State Machine 的最佳分工

可以总结成一个表：

| 能力                      | LLM | State Machine |
| ----------------------- | --: | ------------: |
| 自然语言理解                  |   ✓ |               |
| 推理                      |   ✓ |               |
| 任务规划                    |   ✓ |               |
| Tool 选择建议               |   ✓ |               |
| 状态约束                    |     |             ✓ |
| Transition 验证           |     |             ✓ |
| 权限边界                    |     |             ✓ |
| Retry                   |     |             ✓ |
| Timeout                 |     |             ✓ |
| Recovery                |     |             ✓ |
| 审计                      |     |             ✓ |
| Deterministic Execution |     |             ✓ |
| Context Generation      |   ✓ |               |
| Workflow Control        |     |             ✓ |

因此一个优秀的 Agent Runtime 应该遵循：

```text
Probabilistic Intelligence
          +
Deterministic Control
```

---

# 39. State Machine 不是 Agent 的“大脑”

这是理解 AI Agent Architecture 的关键。

可以把 Agent 类比成人：

```text
LLM
    = Reasoning

Memory
    = Knowledge

Tools
    = Hands

State Machine
    = Nervous System / Control System

Policy Engine
    = Rules

Event Bus
    = Communication

Observability
    = Monitoring
```

LLM 很聪明。

但：

> **聪明不等于可靠。**

生产系统需要的是：

```text
Intelligence
+
Control
+
Safety
+
Persistence
+
Recovery
+
Observability
```

State Machine 正是其中的 Control Layer。

---

# 40. 从 FSM 到 Agent Runtime 的演进路线

可以把整个技术演进理解成：

```text
FSM
 │
 ▼
State Pattern
 │
 ▼
State Machine
 │
 ▼
Hierarchical State Machine
 │
 ▼
Statechart
 │
 ▼
Workflow Engine
 │
 ▼
Event-driven Workflow
 │
 ▼
Agent Runtime
 │
 ▼
Multi-Agent Runtime
```

而 AI Agent 并没有抛弃传统状态机。

相反：

> **AI Agent 把 State Machine 从传统业务流程控制带到了概率计算和自主决策系统。**

---

# 41. 一个值得采用的 Agent State Machine 模型

如果设计一个生产级 Agent，我比较推荐：

```text
                 ┌──────────────┐
                 │     IDLE     │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │   PLANNING   │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │   VALIDATE   │
                 └──────┬───────┘
                        │
                 ┌──────┴──────┐
                 │             │
                 ▼             ▼
             APPROVED       REJECTED
                 │             │
                 ▼             ▼
             EXECUTING       FAILED
                 │
       ┌─────────┼─────────┐
       │         │         │
       ▼         ▼         ▼
    TOOL_CALL  HUMAN     MEMORY
       │
       ▼
    OBSERVE
       │
       ▼
   ┌───┴────┐
   │        │
SUCCESS    FAILURE
   │        │
   ▼        ▼
NEXT      RETRY
   │        │
   └────┬───┘
        ▼
    COMPLETED
```

这个模型已经可以支撑很多企业级 Agent 场景。

---

# 42. 最重要的架构原则

如果只记住本文几个核心原则，我建议记住下面十条。

### 原则 1：State 和 Context 分离

```text
State ≠ Data
```

---

### 原则 2：LLM 不应该直接控制系统状态

应该：

```text
LLM
 ↓
Intent / Event
 ↓
State Machine
```

---

### 原则 3：非法 Transition 必须拒绝

```text
Unknown Event
    ↓
Reject
```

而不是：

```text
Best Effort
```

---

### 原则 4：状态变化必须持久化

```text
State + Version + Context
```

---

### 原则 5：Event 应该是一等公民

```text
User Event
Tool Event
Timer Event
Agent Event
System Event
```

统一进入 State Machine。

---

### 原则 6：失败必须建模

不要：

```text
catch(Exception)
```

然后：

```text
FAILED
```

应该明确：

```text
TIMEOUT
RETRYING
PARTIAL_FAILURE
COMPENSATING
FAILED
```

---

### 原则 7：副作用必须考虑幂等

Agent 可能重复执行：

```text
createOrder()
```

因此必须考虑：

```text
Idempotency Key
```

---

### 原则 8：状态变化必须可观测

至少记录：

```text
from
event
to
reason
duration
traceId
```

---

### 原则 9：复杂 Agent 使用 Hierarchical State Machine

避免：

```text
100+ flat states
```

---

### 原则 10：State Machine 是 Deterministic Layer

最理想的架构不是：

```text
LLM Everything
```

而是：

```text
                 LLM
                  │
           Intelligence
                  │
                  ▼
             State Machine
                  │
        Deterministic Control
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
     Tools      Memory      Human
```

---

# 43. 最终思考：State Machine 是 Agent Engineering 的“骨架”

AI Agent 的核心竞争力并不只是：

```text
Prompt
+
LLM
```

真正进入生产环境后，你需要面对：

```text
Lifecycle
State
Event
Transition
Persistence
Concurrency
Retry
Timeout
Recovery
Compensation
Security
Observability
Audit
Human-in-the-loop
Multi-Agent
```

而这些问题最终都会汇聚到一个核心问题：

> **Agent 当前处于什么状态？发生了什么事件？下一步允许做什么？**

这正是 State Machine 所解决的问题。

因此可以把现代 Agent 架构概括成：

```text
                 ┌─────────────────┐
                 │       LLM       │
                 │ Intelligence    │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │  Decision Layer │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ State Machine   │
                 │ Control Layer   │
                 └────────┬────────┘
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
         Tool Layer    Memory Layer   Human Layer
            │             │             │
            └─────────────┼─────────────┘
                          ▼
                 ┌─────────────────┐
                 │ Event / Message │
                 │      Bus        │
                 └────────┬────────┘
                          ▼
                 ┌─────────────────┐
                 │ Persistence &    │
                 │ Observability   │
                 └─────────────────┘
```

最终可以得到一个非常重要的结论：

> **LLM 让 Agent 具备了“思考能力”，而 State Machine 让 Agent 具备了“可控的行为边界”。**

如果说传统软件工程追求：

> **Deterministic Software**

那么 Agent Engineering 真正需要解决的问题是：

> **如何让 Probabilistic Intelligence 运行在 Deterministic Control 之上。**

而 State Machine，正是连接这两个世界的重要架构抽象。

---

# 44. 进一步的技术方向

如果继续深入 State Machine + AI Agent，可以进一步研究以下几个方向：

```text
State Machine
      │
      ├── Hierarchical State Machine
      │
      ├── Statechart
      │
      ├── Event Sourcing
      │
      ├── Saga / Compensation
      │
      ├── Workflow Engine
      │
      ├── Durable Execution
      │
      ├── Agent Runtime
      │
      ├── Multi-Agent Orchestration
      │
      ├── Policy / Guard Engine
      │
      └── Formal Verification
```

其中一个非常值得深入研究的方向是：

```text
LLM
 ↓
Agent Decision
 ↓
State Machine
 ↓
Policy / Guard
 ↓
Tool
 ↓
Event
 ↓
State Transition
 ↓
Durable Execution
```

这已经逐渐接近下一代：

> **Production-grade Agent Runtime Architecture**

而不是简单的：

> **LLM + Prompt + Tool Calling**

---

## 总结

State Machine 表面上是一种经典的软件设计技术，但它在 AI Agent 时代获得了新的生命力。

它解决的不是“如何让 LLM 更聪明”，而是更加工程化的问题：

```text
Agent 到底在哪里？
为什么到了这里？
接下来允许做什么？
什么时候应该停止？
失败后怎么办？
重启之后怎么恢复？
多个 Agent 如何协调？
如何证明 Agent 没有越权？
如何审计 Agent 的行为？
```

这些问题决定了一个 Agent 是：

```text
Demo
```

还是：

```text
Production System
```

因此，对于准备深入 **AI Agent Architecture / Agent Runtime / Multi-Agent System** 的工程师而言，State Machine 不应该被看成传统技术栈中的一个普通设计模式，而应该被看成：

> **构建可靠 Agent Runtime 的核心控制抽象之一。**

