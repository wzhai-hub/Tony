---
title: Autonomous Agent 深度技术解析：从 LLM Workflow 到真正的自主智能体
# tags:
#   - nodejs
date: '2026-08-05'
summary: Autonomous Agent（自主智能体）不是“一个会调用工具的 LLM”，而是一套能够围绕目标自主进行规划、执行、观察、反思、记忆和动态调整的闭环系统。
---
# Autonomous Agent 深度技术解析：从 LLM Workflow 到真正的自主智能体

> **Autonomous Agent（自主智能体）不是“一个会调用工具的 LLM”，而是一套能够围绕目标自主进行规划、执行、观察、反思、记忆和动态调整的闭环系统。**
>
> 如果说：
>
> * **LLM** 负责理解与生成
> * **RAG** 负责获取知识
> * **Memory** 负责保存经验
> * **Tool / MCP** 负责与外部世界交互
> * **ReAct** 负责 Reason → Act → Observe
> * **Reflection** 负责 Evaluate → Critique → Revise
>
> 那么 **Autonomous Agent** 的核心就是把这些能力组织成一个能够**持续自主完成目标**的系统。

---

# 一、什么是 Autonomous Agent？

传统 LLM Application 通常是：

```text
User
  ↓
Prompt
  ↓
LLM
  ↓
Answer
```

例如：

> “帮我总结这篇文章。”

LLM 完成一次推理后返回结果。

而 Autonomous Agent 的任务通常是：

> “帮我分析这个 Java 项目的性能问题，并提出优化方案。”

这个任务无法通过一次 LLM Call 稳定完成。

Agent 可能需要：

```text
理解目标
 ↓
分析项目
 ↓
寻找关键代码
 ↓
检查数据库
 ↓
分析日志
 ↓
提出假设
 ↓
执行测试
 ↓
发现问题
 ↓
调整方案
 ↓
再次测试
 ↓
总结结果
```

因此：

```text
LLM
=
Generate Answer

Autonomous Agent
=
Pursue Goal
```

这是二者最重要的区别。

---

# 二、从 Chatbot 到 Autonomous Agent

可以把 AI 系统的发展理解成几个阶段。

## Level 1：Prompt → Response

```text
User
 ↓
LLM
 ↓
Answer
```

这是最基本的 Chatbot。

---

## Level 2：RAG

```text
User
 ↓
Retrieve
 ↓
Context
 ↓
LLM
 ↓
Answer
```

Agent 获得了外部知识。

---

## Level 3：Tool Calling

```text
User
 ↓
LLM
 ↓
Tool
 ↓
Observation
 ↓
LLM
 ↓
Answer
```

Agent 开始能够执行操作。

---

## Level 4：ReAct

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

Agent 可以根据环境反馈动态调整行为。

---

## Level 5：Reflection

```text
Generate
 ↓
Evaluate
 ↓
Critique
 ↓
Revise
 ↓
Verify
```

Agent 开始具备自我纠错能力。

---

## Level 6：Autonomous Agent

最终形成：

```text
Goal
 ↓
Understand
 ↓
Plan
 ↓
Execute
 ↓
Observe
 ↓
Reflect
 ↓
Re-plan
 ↓
Execute
 ↓
Verify
 ↓
Complete
```

这才真正接近：

# Autonomous Agent

---

# 三、Autonomous Agent 的核心定义

一个比较工程化的定义：

> **Autonomous Agent 是一种以目标为驱动，能够在有限人工干预下，自主进行任务分解、规划、工具调用、环境观察、状态管理、错误恢复和结果验证，并持续调整执行策略直到达到终止条件的智能软件系统。**

这里面有几个关键词：

```text
Goal
Planning
Execution
Observation
Memory
Reflection
Adaptation
Verification
Termination
```

其中任何一个缺失，都可能只是：

```text
LLM Workflow
```

而不是完整意义上的：

```text
Autonomous Agent
```

---

# 四、Autonomous Agent 的核心架构

一个典型的 Autonomous Agent 可以抽象成：

```text
                         ┌───────────────┐
                         │     User      │
                         └───────┬───────┘
                                 │
                                 ▼
                         ┌───────────────┐
                         │     Goal      │
                         └───────┬───────┘
                                 │
                                 ▼
                      ┌─────────────────────┐
                      │   Agent Controller  │
                      └──────────┬──────────┘
                                 │
                ┌────────────────┼────────────────┐
                │                │                │
                ▼                ▼                ▼
             Memory           Planner           Tools
                │                │                │
                │                ▼                │
                │             Executor            │
                │                │                │
                └────────────────┼────────────────┘
                                 │
                                 ▼
                            Environment
                                 │
                                 ▼
                            Observation
                                 │
                                 ▼
                            Reflection
                                 │
                                 ▼
                              Re-plan
```

其中：

```text
Planner
Executor
Memory
Tools
Reflection
Environment
```

共同组成 Agent 的“认知循环”。

---

# 五、Autonomous Agent 与普通 Workflow 的根本区别

这是理解 Agent 最重要的问题之一。

假设任务：

> 读取一份代码，然后生成测试。

Workflow：

```text
Read Code
 ↓
Generate Test
 ↓
Run Test
 ↓
Return
```

步骤是预定义的。

Agent：

```text
Read Code
 ↓
Analyze
 ↓
Decide what to inspect
 ↓
Generate Test
 ↓
Run Test
 ↓
Observe Failure
 ↓
Decide whether to modify test
 ↓
Run again
 ↓
Evaluate
 ↓
Finish
```

区别可以总结为：

| Workflow         | Autonomous Agent |
| ---------------- | ---------------- |
| 流程预定义            | 动态决定下一步          |
| 固定步骤             | 自适应步骤            |
| 控制逻辑由程序决定        | 部分控制逻辑由 Agent 决定 |
| 输入输出明确           | 状态持续变化           |
| 失败通常终止           | 可以恢复             |
| 很少重新规划           | 可以 Re-plan       |
| Deterministic 较强 | Probabilistic 较强 |

因此：

> **Workflow 是“程序告诉 AI 怎么做”，Agent 是“AI 在约束范围内决定下一步做什么”。**

---

# 六、Agent 的 Goal 是什么？

Autonomous Agent 与普通 Chatbot 的一个重大区别：

```text
Chatbot
=
Question Driven
```

而 Agent：

```text
Agent
=
Goal Driven
```

例如：

```text
Goal:
Improve API latency from 500ms to <100ms.
```

Agent 不应该只回答：

> “可以使用 Redis。”

而应该继续：

```text
Inspect API
 ↓
Collect metrics
 ↓
Analyze trace
 ↓
Find slow query
 ↓
Inspect SQL
 ↓
Suggest index
 ↓
Benchmark
 ↓
Verify latency
```

最终目标是：

```text
Latency < 100ms
```

而不是：

```text
Generate an answer
```

所以 Agent 必须有：

# Goal State

---

# 七、Goal State 与 Termination Condition

一个 Autonomous Agent 必须知道：

> **什么时候算完成？**

例如：

```text
Goal:
Fix failing tests.
```

终止条件：

```text
all tests passed
```

或者：

```text
Goal:
Reduce API latency.
```

终止条件：

```text
p95 < 100ms
```

或者：

```text
Goal:
Deploy application.
```

终止条件：

```text
deployment.status == SUCCESS
```

因此 Agent 应该定义：

```java
class Goal {

    String objective;

    List<Constraint> constraints;

    List<SuccessCriteria> successCriteria;

    List<TerminationCondition> terminationConditions;
}
```

这比简单的：

```text
String prompt;
```

高级很多。

---

# 八、Agent State：自主系统的核心

如果 Agent 只依赖当前 Prompt：

```text
Prompt
 ↓
LLM
```

它无法真正持续执行复杂任务。

需要一个：

```text
Agent State
```

例如：

```json
{
  "goal": "fix production latency",
  "status": "EXECUTING",
  "current_plan": [
    "inspect traces",
    "find slow service",
    "analyze database"
  ],
  "completed_steps": [
    "trace analysis"
  ],
  "current_step": "database analysis",
  "observations": [],
  "errors": [],
  "iteration": 3
}
```

Agent 每一步都在修改 State。

因此：

```text
Agent
=
LLM
+
State
+
Environment
+
Tools
+
Control Loop
```

---

# 九、Agent Control Loop

Autonomous Agent 最核心的代码其实不是 LLM，而是：

```text
while (!goalAchieved) {
    observe();
    plan();
    act();
    evaluate();
}
```

更完整：

```text
while (running) {

    State state = observe();

    Plan plan = planner.plan(
        goal,
        state,
        memory
    );

    Action action =
        executor.select(plan);

    Observation observation =
        tools.execute(action);

    state.update(observation);

    Evaluation evaluation =
        evaluator.evaluate(
            goal,
            state
        );

    if (evaluation.completed()) {
        break;
    }

    if (evaluation.failed()) {
        state = recovery(state);
    }

    if (needReplan(state)) {
        continue;
    }
}
```

这才是 Autonomous Agent 的核心。

---

# 十、Planner：Agent 的规划系统

Planner 的任务：

> **下一步应该做什么？**

例如：

```text
Goal:
Build a production-ready Spring Boot application.
```

Planner：

```text
1. Analyze requirements
2. Design architecture
3. Create project
4. Implement backend
5. Implement tests
6. Run tests
7. Fix failures
8. Security scan
9. Package
10. Deploy
```

但是：

> **计划不是静态的。**

如果：

```text
Step 6
Tests failed
```

Planner 应该重新规划：

```text
Analyze failure
 ↓
Locate bug
 ↓
Modify code
 ↓
Retest
```

因此：

# Planning ≠ One-time Planning

真正的 Agent：

```text
Plan
 ↓
Execute
 ↓
Observe
 ↓
Re-plan
```

---

# 十一、Planning 的三个层次

## 1. Static Planning

一开始规划完整任务：

```text
A → B → C → D
```

适合：

```text
流程稳定
环境变化少
```

---

## 2. Dynamic Planning

每完成一步重新规划：

```text
A
 ↓
Observation
 ↓
B
 ↓
Observation
 ↓
C
```

适合复杂环境。

---

## 3. Hierarchical Planning

大型任务：

```text
Goal
│
├── SubGoal A
│   ├── Task A1
│   ├── Task A2
│   └── Task A3
│
├── SubGoal B
│   ├── Task B1
│   └── Task B2
│
└── SubGoal C
```

这就是：

# Hierarchical Task Planning

对于复杂 Agent 非常重要。

---

# 十二、Planner 与 Executor 必须分离

一个非常好的架构原则：

```text
Planner
=
What should we do?

Executor
=
How do we execute it?
```

例如：

```text
Planner:
"Check database performance."
```

Executor：

```text
Run EXPLAIN
 ↓
Collect query plan
 ↓
Analyze index
```

这样可以避免：

```text
LLM
直接控制所有底层动作
```

从而提升：

```text
Security
Maintainability
Observability
```

---

# 十三、ReAct 是 Autonomous Agent 的底层循环之一

我们前面讨论过 ReAct：

```text
Reason
 ↓
Act
 ↓
Observe
```

把它放进 Autonomous Agent：

```text
Goal
 ↓
Plan
 ↓
Reason
 ↓
Act
 ↓
Observe
 ↓
Reflect
 ↓
Re-plan
```

因此：

> **ReAct 可以看作 Autonomous Agent 的执行循环，而不是 Autonomous Agent 的全部。**

一个完整 Agent 通常还需要：

```text
Goal
Planning
Memory
Reflection
Verification
Safety
```

---

# 十四、Memory：让 Agent 具有持续性

Autonomous Agent 如果没有 Memory：

```text
Task
 ↓
Execute
 ↓
Forget
```

有 Memory：

```text
Task
 ↓
Execute
 ↓
Learn
 ↓
Store
 ↓
Future Task
 ↓
Retrieve
 ↓
Better Decision
```

Memory 可以分为：

```text
Short-Term Memory
Long-Term Memory
Episodic Memory
Semantic Memory
Procedural Memory
```

---

# 十五、Short-Term Memory

保存当前任务上下文：

```text
Goal
Plan
Observations
Tool Results
Errors
Intermediate Results
```

例如：

```json
{
  "current_task": "fix latency",
  "current_step": "analyze SQL",
  "last_observation": "query takes 800ms"
}
```

生命周期：

```text
Task scoped
```

---

# 十六、Long-Term Memory

保存长期知识：

```text
User Preferences
Past Solutions
Successful Strategies
Known Failures
Project Knowledge
```

例如：

```text
For this project:
Redis is used for caching.
PostgreSQL is the primary database.
Kafka is used for async processing.
```

---

# 十七、Episodic Memory

记录：

> **发生过什么。**

例如：

```text
2026-08-20

Task:
Optimize payment API.

Actions:
- analyzed trace
- found DB bottleneck
- added index

Result:
p95 reduced from 800ms to 120ms.
```

这对 Autonomous Agent 很重要。

因为 Agent 可以学习：

```text
过去做过类似事情
```

---

# 十八、Procedural Memory

保存：

> **怎么做。**

例如：

```text
When PostgreSQL query latency is high:

1. Check EXPLAIN ANALYZE
2. Check index usage
3. Check row estimation
4. Check sequential scan
5. Check statistics
```

这相当于 Agent 的：

```text
Skill
```

---

# 十九、Reflection：Agent 如何自我纠错

Autonomous Agent 不应该：

```text
失败
 ↓
停止
```

而应该：

```text
失败
 ↓
Analyze
 ↓
Reflect
 ↓
Identify Root Cause
 ↓
Revise Plan
 ↓
Retry
```

例如：

```text
Deploy
 ↓
Failed
 ↓
Reflection
 ↓
Missing environment variable
 ↓
Update configuration
 ↓
Deploy
 ↓
Success
```

这就是：

# Autonomous Recovery

---

# 二十、Agent 的 Recovery 能力

一个真正 Autonomous 的系统必须考虑失败。

典型失败：

```text
Tool Failure
LLM Failure
Network Failure
Authentication Failure
Invalid Input
Unexpected State
Timeout
Rate Limit
```

Agent 不应该全部统一处理。

例如：

```text
Timeout
→ Retry

401
→ Refresh credential

404
→ Re-plan

Validation Error
→ Fix parameters

Business Rule Violation
→ Stop
```

这意味着 Agent 需要：

# Failure Classification

---

# 二十一、Retry ≠ Recovery

这是一个非常重要的区别。

简单 Retry：

```text
Failed
 ↓
Retry
 ↓
Failed
 ↓
Retry
```

这是：

```text
Blind Retry
```

真正的 Autonomous Recovery：

```text
Failed
 ↓
Analyze Failure
 ↓
Classify Failure
 ↓
Determine Cause
 ↓
Change Strategy
 ↓
Retry
```

例如：

```text
API timeout
```

第一次：

```text
retry same request
```

仍失败。

Agent 应该：

```text
reduce request size
 ↓
change endpoint
 ↓
increase timeout
 ↓
retry
```

这才是：

> **Adaptive Recovery**

---

# 二十二、Environment：Agent 的“世界”

Agent 如果没有环境，就很难称为真正的 Autonomous Agent。

Environment 可以是：

```text
Operating System
Database
Browser
Cloud
Git Repository
Kubernetes
REST API
Enterprise System
```

例如 Coding Agent：

```text
Environment
├── Git
├── File System
├── Compiler
├── Test Framework
├── Terminal
└── CI/CD
```

Agent：

```text
Think
 ↓
Act on Environment
 ↓
Observe Environment
```

这就是：

# Agent-Environment Interaction

---

# 二十三、Tool 是 Agent 的执行器

工具可以抽象为：

```java
interface Tool {

    String name();

    ToolSchema schema();

    ToolResult execute(
        ToolInput input
    );
}
```

例如：

```text
SearchTool
DatabaseTool
ShellTool
GitTool
BrowserTool
KubernetesTool
CloudTool
```

LLM：

```text
Decide:
Use GitTool
```

然后：

```text
GitTool
 ↓
git diff
 ↓
Observation
```

Agent 再决定下一步。

---

# 二十四、MCP 在 Autonomous Agent 中的位置

MCP 可以解决一个非常重要的问题：

> **如何标准化 Agent 与外部工具、数据和服务的连接。**

例如：

```text
Agent
 │
 ▼
MCP
 │
 ├── GitHub
 ├── Database
 ├── Files
 ├── Search
 ├── Jira
 └── Cloud
```

因此：

```text
MCP
=
Tool / Context Integration Layer
```

但需要注意：

> MCP 本身不是 Agent。

MCP 提供：

```text
Connectivity
```

而 Agent 提供：

```text
Autonomy
```

---

# 二十五、Autonomous Agent 的核心闭环

现在可以把前面所有能力组合起来：

```text
                         Goal
                          │
                          ▼
                    ┌───────────┐
                    │   State   │
                    └─────┬─────┘
                          │
                          ▼
                    ┌───────────┐
                    │  Planner  │
                    └─────┬─────┘
                          │
                          ▼
                    ┌───────────┐
                    │  Executor │
                    └─────┬─────┘
                          │
                          ▼
                       Tools
                          │
                          ▼
                     Environment
                          │
                          ▼
                     Observation
                          │
                          ▼
                    ┌───────────┐
                    │ Evaluator │
                    └─────┬─────┘
                          │
                    ┌─────┴─────┐
                    │           │
                  Pass        Fail
                    │           │
                    ▼           ▼
                  Done      Reflection
                                │
                                ▼
                            Re-planner
                                │
                                └───────→ Executor
```

这就是 Autonomous Agent 最核心的：

# Agent Control Loop

---

# 二十六、为什么 Agent 必须有 Evaluator？

如果只有：

```text
Plan
 ↓
Execute
 ↓
Done
```

Agent 无法知道：

> “我真的完成了吗？”

因此：

```text
Evaluator
```

负责：

```text
Did we achieve the goal?
```

例如：

```text
Goal:
All tests pass.
```

Evaluator：

```text
mvn test
```

结果：

```text
FAIL
```

Agent：

```text
Not Done
```

然后继续。

---

# 二十七、Evaluator 可以是确定性的

和 Reflection 一样：

> 不要什么都交给 LLM。

例如：

```text
Code
→ Compiler

Tests
→ JUnit

API
→ HTTP Status + Schema

SQL
→ Execution Plan

Deployment
→ Kubernetes Status

Security
→ Scanner
```

LLM 负责：

```text
Interpret
Plan
Reason
```

确定性工具负责：

```text
Verify
```

这是生产级 Agent 的重要设计原则。

---

# 二十八、Agent 的自主程度

“Autonomous”不是一个二元概念。

可以定义：

```text
Level 0
Human Driven

Level 1
LLM Assisted

Level 2
Tool-Using Agent

Level 3
Planning Agent

Level 4
Reflective Agent

Level 5
Autonomous Agent
```

例如：

### Level 0

```text
Human
 ↓
LLM
```

### Level 2

```text
LLM
 ↓
Tool
```

### Level 4

```text
Plan
 ↓
Execute
 ↓
Reflect
 ↓
Retry
```

### Level 5

```text
Goal
 ↓
Plan
 ↓
Execute
 ↓
Observe
 ↓
Recover
 ↓
Re-plan
 ↓
Verify
 ↓
Complete
```

---

# 二十九、Autonomy 并不等于无限权限

这是企业级 Agent 最重要的问题之一。

如果 Agent 可以：

```text
Delete Database
Deploy Production
Send Email
Transfer Money
Modify IAM
```

那么：

```text
Autonomy ↑
Risk ↑
```

所以：

# Autonomy 必须建立在 Constraints 上。

---

# 三十、Agent Safety Architecture

建议增加：

```text
                Agent
                  │
                  ▼
             Policy Engine
                  │
          ┌───────┴────────┐
          │                │
        Allow             Deny
          │
          ▼
        Tool
```

例如：

```text
Tool:
delete_database
```

Policy：

```text
environment == production
→ DENY
```

又比如：

```text
deploy_production
```

要求：

```text
risk > threshold
→ Human Approval
```

---

# 三十一、Human-in-the-loop

企业级 Agent 通常不是：

```text
100% Autonomous
```

而是：

```text
Human
   ↓
Agent
   ↓
Risk Assessment
   ↓
┌───────────────┐
│ Low Risk      │ → Auto Execute
│ Medium Risk   │ → Review
│ High Risk     │ → Human Approval
└───────────────┘
```

这叫：

# Risk-Aware Autonomy

---

# 三十二、权限系统

每个 Agent Tool 都应该有：

```text
Permission
Scope
Rate Limit
Audit
Approval
```

例如：

```json
{
  "tool": "production_deploy",
  "permission": "DEPLOY",
  "environment": "production",
  "approvalRequired": true
}
```

Agent 不是超级用户。

而应该：

> **在明确的 Capability Boundary 中自主行动。**

---

# 三十三、Agent 的 Sandbox

对于 Coding Agent：

```text
Agent
 ↓
Sandbox
 ↓
File System
Compiler
Terminal
Tests
```

而不是：

```text
Agent
 ↓
Production Server
```

Sandbox 可以限制：

```text
File Access
Network Access
Process
CPU
Memory
Secrets
```

这是防止 Agent 失控的重要手段。

---

# 三十四、Agent Loop 为什么可能失控？

典型问题：

```text
Plan
 ↓
Action
 ↓
Failure
 ↓
Re-plan
 ↓
Failure
 ↓
Re-plan
 ↓
...
```

造成：

```text
Infinite Loop
```

所以必须设置：

```text
maxIterations
maxTime
maxCost
maxToolCalls
maxFailures
```

例如：

```java
if (state.iteration() > 20) {
    terminate();
}
```

---

# 三十五、Agent Budget

可以把 Agent 资源定义成：

```text
Token Budget
Tool Budget
Time Budget
Money Budget
Action Budget
```

例如：

```json
{
  "maxIterations": 15,
  "maxToolCalls": 30,
  "maxTokens": 100000,
  "timeoutSeconds": 300,
  "maxCost": 2.0
}
```

这样 Agent 才真正具有：

# Bounded Autonomy

---

# 三十六、Agent 的状态机设计

生产系统中，我非常推荐显式 State Machine。

例如：

```java
enum AgentState {

    INITIALIZING,

    PLANNING,

    EXECUTING,

    OBSERVING,

    EVALUATING,

    REFLECTING,

    REPLANNING,

    WAITING_APPROVAL,

    COMPLETED,

    FAILED,

    TERMINATED
}
```

状态转移：

```text
INITIALIZING
      ↓
PLANNING
      ↓
EXECUTING
      ↓
OBSERVING
      ↓
EVALUATING
      │
 ┌────┴─────┐
 ↓          ↓
PASS       FAIL
 ↓          ↓
COMPLETED REFLECTING
             ↓
          REPLANNING
             ↓
          EXECUTING
```

这比：

```text
LLM + while(true)
```

更加适合生产环境。

---

# 三十七、Agent Orchestrator

整个 Agent 可以有一个：

```text
AgentOrchestrator
```

例如：

```java
class AgentOrchestrator {

    private Planner planner;
    private Executor executor;
    private Memory memory;
    private Evaluator evaluator;
    private ReflectionEngine reflection;
    private PolicyEngine policy;

    public Result run(Goal goal) {

        AgentState state =
            initialize(goal);

        while (!state.isTerminal()) {

            Plan plan =
                planner.plan(state);

            Action action =
                executor.next(plan);

            if (!policy.allow(action)) {
                return handleApproval(action);
            }

            Observation observation =
                executor.execute(action);

            state =
                state.update(observation);

            Evaluation evaluation =
                evaluator.evaluate(state);

            if (!evaluation.success()) {
                state =
                    reflection.reflect(state);
            }
        }

        return state.result();
    }
}
```

这已经非常接近一个真正的 Agent Runtime。

---

# 三十八、Agent Runtime

大型系统中，可以进一步抽象：

```text
                Agent Application
                       │
                       ▼
                Agent Runtime
                       │
       ┌───────────────┼───────────────┐
       │               │               │
    Planner          Memory           Tools
       │               │               │
       ├───────────────┼───────────────┤
       │               │               │
       ▼               ▼               ▼
    Executor        Storage        MCP/API
       │
       ▼
   Environment
```

Agent Runtime 类似：

```text
JVM
Application Server
Workflow Engine
```

为 Agent 提供：

```text
Execution
State
Memory
Scheduling
Tool Management
Security
Observability
Recovery
```

这是 AI Platform Engineer 非常值得研究的方向。

---

# 三十九、Agent 与 Workflow Engine

这两个概念未来会长期共存。

Workflow：

```text
A → B → C → D
```

Agent：

```text
A
 ↓
Observation
 ↓
LLM decides B/C/D
```

生产系统更可能采用：

```text
Workflow
+
Agent
```

例如：

```text
                    Workflow
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
       Agent A      Approval      Agent B
          │                         │
          ▼                         ▼
       Dynamic                   Dynamic
       Planning                  Planning
```

即：

> **Deterministic Workflow 管理边界，Agent 管理动态决策。**

这是非常重要的架构模式。

---

# 四十、Multi-Agent Autonomous System

单 Agent 解决不了所有问题时，可以拆成多个 Agent。

例如软件工程：

```text
                    Project Goal
                         │
                         ▼
                  Project Manager
                         │
         ┌───────────────┼────────────────┐
         ▼               ▼                ▼
    Architect        Developer          Tester
         │               │                │
         ▼               ▼                ▼
      Design           Code             Test
         │               │                │
         └───────────────┼────────────────┘
                         ▼
                       Reviewer
                         │
                         ▼
                       Deploy
```

不同 Agent：

```text
Planner Agent
Coding Agent
Testing Agent
Security Agent
Reviewer Agent
Deployment Agent
```

形成：

# Multi-Agent System

---

# 四十一、Multi-Agent 不一定比 Single-Agent 好

这是一个很重要的误区。

Multi-Agent 增加：

```text
Communication Cost
Token Cost
Latency
Coordination Complexity
Failure Modes
```

如果：

```text
Single Agent
```

就能解决：

```text
不要为了“看起来高级”而使用 Multi-Agent。
```

真正应该拆分的情况通常是：

```text
任务职责明显不同
工具权限明显不同
上下文明显不同
专业能力明显不同
安全边界明显不同
```

---

# 四十二、Autonomous Agent 的观察能力

Agent 不应该只观察：

```text
Tool Response
```

还应该观察：

```text
Environment State
System Metrics
Logs
Errors
Events
User Feedback
External Data
```

例如 Kubernetes Agent：

```text
kubectl get pods
 ↓
Pod CrashLoopBackOff
 ↓
kubectl logs
 ↓
OOMKilled
 ↓
Inspect memory
 ↓
Update resource limit
```

Agent 的智能很大程度来自：

# Observation Quality

---

# 四十三、Observation 不等于 Context

Context：

```text
当前模型看到的信息
```

Observation：

```text
Agent 从环境获得的新状态
```

例如：

```text
Action:
Run SQL
```

Observation：

```text
Query took 2.8 seconds
Rows scanned: 5 million
Index: unused
```

然后 Agent：

```text
Reason
```

所以：

```text
Action
 ↓
Observation
 ↓
State Update
 ↓
Reason
```

是 Autonomous Agent 的基本循环。

---

# 四十四、Agent 的世界模型

高级 Agent 还需要形成：

```text
World Model
```

例如：

```text
Service A
 ↓
Service B
 ↓
Database

Service B
 ↓
Redis
```

Agent 知道：

```text
如果 Service B 出问题，
可能影响 Service A。
```

因此：

> Agent 不只是执行命令，而是在不断构建对环境的内部模型。

---

# 四十五、Agent Planning 的本质

可以把 Planning 看成：

```text
Current State
+
Goal State
↓
Plan
```

例如：

```text
Current:
API latency = 800ms

Goal:
API latency < 100ms
```

Planner：

```text
Analyze trace
 ↓
Find slow SQL
 ↓
Optimize query
 ↓
Add index
 ↓
Benchmark
```

如果：

```text
Benchmark = 150ms
```

则：

```text
Goal not achieved
```

重新：

```text
Analyze cache
 ↓
Add Redis
 ↓
Benchmark
```

这就是：

# Goal-Oriented Planning

---

# 四十六、Autonomous Agent 与传统 AI 的区别

传统 AI：

```text
Input
 ↓
Model
 ↓
Prediction
```

Agent：

```text
Goal
 ↓
State
 ↓
Planning
 ↓
Action
 ↓
Environment
 ↓
Observation
 ↓
Planning
```

传统 AI：

```text
Prediction
```

Agent：

```text
Decision + Action
```

---

# 四十七、Agent 与 Reinforcement Learning 的关系

二者有相似之处。

RL：

```text
State
 ↓
Action
 ↓
Reward
 ↓
Next State
```

Agent：

```text
State
 ↓
Reason
 ↓
Action
 ↓
Observation
 ↓
Evaluation
 ↓
Next State
```

可以看到：

```text
Agent
≈
LLM-driven decision loop
```

但是不要简单认为：

```text
Agent = RL
```

大多数现代 LLM Agent 并不是通过在线强化学习训练出来的，而是在推理阶段通过：

```text
Prompt
Planning
Tools
Memory
Reflection
Feedback
```

实现自主行为。

---

# 四十八、Agent 的“智能”到底来自哪里？

这是一个很值得思考的问题。

一个 Agent：

```text
LLM
+
Tools
+
Memory
+
Planner
+
Reflection
```

它的智能并不完全来自 LLM。

可以认为：

```text
Agent Intelligence
=
Model Intelligence
+
Tool Intelligence
+
Environment Feedback
+
Memory
+
Control Policy
```

因此：

> **Agent 是一个系统，而不是一个模型。**

这是理解 Agent Engineering 最重要的思想之一。

---

# 四十九、Autonomous Agent 的工程挑战

真正做生产级 Agent，困难并不是：

```text
怎么调用 LLM API
```

而是：

## 1. Reliability

```text
Agent 会不会走错？
```

## 2. Controllability

```text
Agent 会不会做危险的事情？
```

## 3. Observability

```text
为什么 Agent 做了这个决定？
```

## 4. Cost

```text
一次任务花多少钱？
```

## 5. Latency

```text
为什么需要 2 分钟？
```

## 6. Recovery

```text
失败以后怎么办？
```

## 7. State Management

```text
复杂任务状态如何保存？
```

## 8. Security

```text
Agent 能访问什么？
```

---

# 五十、Agent Observability

生产 Agent 强烈建议使用：

```text
OpenTelemetry
```

记录：

```text
agent.run
agent.plan
agent.tool_call
agent.observation
agent.reflection
agent.memory.retrieve
agent.memory.write
agent.approval
```

例如 Trace：

```text
agent.run
│
├── memory.retrieve
│
├── planner
│
├── tool.git.diff
│
├── observation
│
├── reflection
│
├── tool.maven.test
│
├── observation
│
└── final
```

Metrics：

```text
agent_success_rate
agent_failure_rate
agent_iterations
agent_tool_calls
agent_latency
agent_token_usage
agent_cost
agent_replan_rate
agent_human_intervention_rate
```

对于你之前研究的 **OpenTelemetry + Grafana + Tempo** 体系，这一块尤其值得深入：Agent 本质上可以被设计成一个**可观测的分布式执行系统**。

---

# 五十一、Agent Evaluation

传统 LLM：

```text
Accuracy
```

Agent：

```text
Task Success Rate
```

还应该包括：

```text
Planning Quality
Tool Selection Accuracy
Recovery Rate
Reflection Effectiveness
Cost Efficiency
Latency
Safety Violations
Human Intervention Rate
```

例如：

```text
Agent Evaluation

Task Success       92%
Planning Success   89%
Tool Accuracy      95%
Recovery Rate      76%
Safety Violation    0%
Avg Iterations      4.2
Avg Cost           $0.18
```

这才是真正的 Agent Evaluation。

---

# 五十二、Agent 的关键 KPI

可以定义：

### Success Rate

```text
成功任务数 / 总任务数
```

### Recovery Rate

```text
成功恢复失败任务 / 失败任务
```

### Tool Efficiency

```text
有效 Tool Calls / Total Tool Calls
```

### Planning Efficiency

```text
Successful Tasks / Planning Steps
```

### Cost Efficiency

```text
Task Success / Cost
```

### Autonomy Rate

```text
无需人工干预完成的任务 / 总任务
```

---

# 五十三、一个生产级 Autonomous Coding Agent

如果设计一个：

# Java Autonomous Coding Agent

可以这样：

```text
                        User
                         │
                         ▼
                    Goal Manager
                         │
                         ▼
                     Planner
                         │
                         ▼
                     Executor
                         │
       ┌─────────────────┼──────────────────┐
       ▼                 ▼                  ▼
    Git Tool          Shell Tool         Search
       │                 │                  │
       └─────────────────┼──────────────────┘
                         ▼
                    Environment
                         │
                         ▼
                    Test Runner
                         │
                         ▼
                     Evaluator
                         │
                    ┌────┴─────┐
                    ▼          ▼
                  PASS        FAIL
                    │          │
                    ▼          ▼
                  Done      Reflection
                               │
                               ▼
                            Re-plan
                               │
                               └────→ Executor
```

Memory：

```text
Project Knowledge
Past Fixes
Coding Patterns
Known Failures
```

Safety：

```text
Sandbox
Permission
Approval
```

Observability：

```text
OpenTelemetry
Prometheus
Grafana
Tempo
```

这已经是一套完整的：

# Autonomous Software Engineer

---

# 五十四、Autonomous Agent 的最终架构

把全文浓缩：

```text
                         ┌───────────────┐
                         │     Goal      │
                         └───────┬───────┘
                                 │
                                 ▼
                        ┌────────────────┐
                        │ Agent Runtime  │
                        └───────┬────────┘
                                │
               ┌────────────────┼────────────────┐
               │                │                │
               ▼                ▼                ▼
            Memory           Planner          Policy
               │                │                │
               └────────────────┼────────────────┘
                                │
                                ▼
                            Executor
                                │
                                ▼
                              Tools
                                │
                                ▼
                           Environment
                                │
                                ▼
                           Observation
                                │
                                ▼
                           Evaluator
                                │
                         ┌──────┴───────┐
                         │              │
                       PASS           FAIL
                         │              │
                         ▼              ▼
                       Done         Reflection
                                        │
                                        ▼
                                     Re-plan
                                        │
                                        └──────→ Executor
```

外围再增加：

```text
Security
Observability
Human Approval
Cost Control
State Persistence
```

这才是一个真正可生产化的 Autonomous Agent。

---

# 五十五、最核心的 Agent Loop

如果只记住一个公式：

```text
Goal
 ↓
Observe
 ↓
Plan
 ↓
Act
 ↓
Observe
 ↓
Evaluate
 ↓
Reflect
 ↓
Re-plan
 ↓
Act
 ↓
Verify
 ↓
Goal Achieved
```

可以进一步抽象成：

```text
                    ┌──────────────┐
                    │     Goal     │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │     Plan     │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │     Act      │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │   Observe    │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │   Evaluate   │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │   Reflect    │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │   Re-plan    │
                    └──────┬───────┘
                           │
                           └──────────→ Act
```

因此：

# **Autonomous Agent = Goal + State + Planning + Action + Observation + Memory + Reflection + Verification + Control Loop**

而不是：

```text
Autonomous Agent = LLM + Tool Calling
```

---

# 五十六、从 AI Developer 到 Agent Engineer

如果把这套技术体系整理成学习路线，我建议形成下面这条主线：

```text
                    LLM
                     │
                     ▼
               Prompt Engineering
                     │
                     ▼
                    RAG
                     │
                     ▼
               Tool Calling
                     │
                     ▼
                    MCP
                     │
                     ▼
                   ReAct
                     │
                     ▼
                  Memory
                     │
                     ▼
                Reflection
                     │
                     ▼
                  Planning
                     │
                     ▼
              Autonomous Agent
                     │
             ┌───────┴────────┐
             ▼                ▼
        Multi-Agent       Agent Runtime
             │                │
             └───────┬────────┘
                     ▼
             Production Agent
```

再往后就是：

```text
Agent Evaluation
Agent Observability
Agent Security
Agent Governance
Agent Platform
```

这条路线实际上已经从：

> **“学习怎么调用大模型”**

进入：

> **“设计 AI 软件系统”**

的阶段。

---

# 五十七、最终总结

Autonomous Agent 最重要的变化不是：

```text
LLM 更聪明了。
```

而是：

```text
LLM
从一个
“回答问题的模型”

变成了

“参与环境、持续执行目标的决策组件”。
```

传统 LLM：

```text
Prompt
 ↓
Response
```

Agent：

```text
Goal
 ↓
State
 ↓
Plan
 ↓
Action
 ↓
Observation
 ↓
Reflection
 ↓
Re-plan
 ↓
Verification
 ↓
Completion
```

而生产级 Autonomous Agent：

```text
Autonomy
+
Memory
+
Planning
+
ReAct
+
Reflection
+
Tools/MCP
+
Verification
+
Security
+
Observability
+
Human-in-the-loop
```

最终可以用一句话概括：

> **Autonomous Agent 的本质，是让 LLM 从“回答器”变成“受约束的目标驱动执行器”：它能够理解目标、维护状态、制定计划、调用工具、观察环境、从失败中恢复、重新规划，并通过验证判断任务是否真正完成。**

对于后端/Java 工程师而言，真正值得深入的方向并不是只学习某一个 Agent Framework，而是理解背后的 **Agent Runtime + State Machine + Planner + Tool/MCP + Memory + Reflection + Evaluation + Observability + Security**。掌握这些之后，无论框架如何变化，都能自己设计 Agent 系统。

**下一篇更适合继续深入哪一块：① Agent Runtime 架构与 Java/Spring Boot 实现，② Planning/Task Decomposition 算法，③ Multi-Agent System？**
