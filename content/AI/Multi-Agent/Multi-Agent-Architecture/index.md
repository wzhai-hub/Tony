---
title: Multi-Agent Architecture 深度技术博客：从单 Agent 到企业级多智能体系统
# tags:
#   - nodejs
date: '2026-08-05'
summary: 当一个复杂任务无法由单一 Agent 在有限上下文、有限工具、有限权限和有限认知能力下可靠完成时，如何通过**角色分工、任务分解、协作协议、状态管理、通信机制、共享记忆、自治决策和全局治理**，构建一个可扩展的智能软件系统。
---

# Multi-Agent Architecture 深度技术博客：从单 Agent 到企业级多智能体系统

> **Multi-Agent System（多智能体系统）并不是“同时运行多个 LLM”。**
>
> 它真正解决的问题是：当一个复杂任务无法由单一 Agent 在有限上下文、有限工具、有限权限和有限认知能力下可靠完成时，如何通过**角色分工、任务分解、协作协议、状态管理、通信机制、共享记忆、自治决策和全局治理**，构建一个可扩展的智能软件系统。
>
> 如果 Single-Agent 关注的是：
>
> ```text
> Agent 如何完成一个任务？
> ```
>
> 那么 Multi-Agent 关注的是：
>
> ```text
> 多个 Agent 如何共同完成一个任务？
> ```

---

# 一、Multi-Agent 到底解决什么问题？

先看一个典型任务：

> “分析一个大型 Java 微服务系统的线上性能问题，定位根因，修改代码，编写测试，验证性能，并生成上线方案。”

如果让一个 Agent 完成：

```text
Agent
 │
 ├── Read Code
 ├── Analyze Logs
 ├── Analyze Traces
 ├── Analyze Database
 ├── Modify Code
 ├── Write Tests
 ├── Run Tests
 ├── Benchmark
 ├── Security Check
 └── Deployment
```

理论上可以。

但实际会遇到：

```text
Context Explosion
Tool Explosion
Reasoning Complexity
Permission Complexity
Memory Complexity
Long-running Task
Error Recovery
```

于是可以拆成：

```text
                         Goal
                          │
                          ▼
                   Orchestrator
                          │
       ┌──────────────────┼──────────────────┐
       ▼                  ▼                  ▼
  Analyst Agent      Coding Agent       Testing Agent
       │                  │                  │
       ▼                  ▼                  ▼
    Analysis             Code              Tests
       │                  │                  │
       └──────────────────┼──────────────────┘
                          ▼
                    Reviewer Agent
                          │
                          ▼
                   Deployment Agent
```

这就是 Multi-Agent Architecture。

---

# 二、Multi-Agent 的核心思想

Multi-Agent Architecture 可以抽象成：

```text
Complex Goal
     │
     ▼
Task Decomposition
     │
     ▼
Agent Allocation
     │
     ▼
Agent Collaboration
     │
     ▼
Shared / Private State
     │
     ▼
Result Aggregation
     │
     ▼
Global Verification
```

其中最重要的是：

```text
Decomposition
+
Coordination
+
Communication
+
State
+
Governance
```

所以：

> **Multi-Agent Architecture 本质上是一个“分布式智能系统”。**

这一点非常重要。

传统微服务解决：

```text
业务复杂度
```

Multi-Agent 解决：

```text
认知复杂度
```

---

# 三、Single-Agent 与 Multi-Agent

## Single-Agent

```text
User
 │
 ▼
Agent
 │
 ├── Planner
 ├── Memory
 ├── Tools
 └── Executor
```

优点：

* 架构简单
* 上下文统一
* 通信成本低
* Debug 容易
* Token 成本较低

缺点：

* 单 Agent 上下文越来越大
* Tool 数量膨胀
* 权限边界模糊
* 角色冲突
* 复杂任务容易失控

---

# 四、Multi-Agent

```text
                 Orchestrator
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
     Agent A       Agent B       Agent C
        │             │             │
        ▼             ▼             ▼
      Tools         Tools         Tools
```

优点：

```text
职责隔离
上下文隔离
权限隔离
专业能力隔离
并行执行
独立记忆
```

缺点：

```text
通信复杂
协调困难
Token 增加
Latency 增加
Debug 困难
Failure Modes 增多
```

因此：

> **Multi-Agent 不是 Single-Agent 的升级版，而是一种架构权衡。**

---

# 五、什么时候应该使用 Multi-Agent？

这是实际工程中最重要的问题。

不要看到复杂任务就：

```text
Agent A
Agent B
Agent C
Agent D
Agent E
```

更合理的判断标准是：

## 1. 是否存在明显的角色差异？

例如：

```text
Researcher
Coder
Tester
Reviewer
```

如果职责完全不同，可以拆。

---

## 2. 是否存在上下文隔离需求？

例如：

```text
Security Agent
```

只需要：

```text
Security Context
```

没必要看到整个项目上下文。

---

## 3. 是否需要不同权限？

例如：

```text
Research Agent
→ Read Only

Coding Agent
→ Read / Write

Deployment Agent
→ Production Deployment
```

权限边界明显时，非常适合 Multi-Agent。

---

## 4. 是否可以并行？

例如：

```text
                 Analyze System
                 /     |      \
                /      |       \
               ▼       ▼        ▼
             Logs    Traces    DB
```

三个 Agent 可以同时执行。

---

# 六、Multi-Agent 的基本架构模型

一个成熟的 Multi-Agent System 通常包含：

```text
┌─────────────────────────────────────────────┐
│              Multi-Agent Platform           │
│                                             │
│  ┌──────────────┐                           │
│  │ Orchestrator │                           │
│  └──────┬───────┘                           │
│         │                                    │
│   ┌─────┼──────────────┐                    │
│   │     │              │                    │
│   ▼     ▼              ▼                    │
│ Agent A Agent B      Agent C                │
│   │     │              │                    │
│   └─────┼──────────────┘                    │
│         │                                    │
│         ▼                                    │
│  ┌───────────────┐                           │
│  │ Communication │                           │
│  └───────┬───────┘                           │
│          │                                    │
│  ┌───────┼────────┐                           │
│  ▼       ▼        ▼                           │
│ Memory  Tools  Environment                   │
│                                             │
└─────────────────────────────────────────────┘
```

核心组件：

```text
Agent
Agent Runtime
Orchestrator
Communication
Memory
Tool
Task Manager
State Store
Policy Engine
Evaluator
Observability
```

---

# 七、Agent 本身应该是什么？

一个 Agent 不应该简单定义成：

```java
class Agent {
    LLM llm;
}
```

更合理：

```java
class Agent {

    Identity identity;

    Role role;

    Goal goal;

    Planner planner;

    Memory memory;

    ToolRegistry tools;

    PolicyEngine policy;

    Executor executor;

    Evaluator evaluator;
}
```

甚至：

```text
Agent
│
├── Identity
├── Role
├── Goal
├── Capability
├── Memory
├── Planner
├── Executor
├── Policy
├── Evaluator
└── Runtime State
```

---

# 八、Agent Identity

Multi-Agent 中每个 Agent 应该拥有自己的 Identity。

例如：

```json
{
  "agentId": "security-agent",
  "role": "security-reviewer",
  "version": "1.3",
  "capabilities": [
    "code-analysis",
    "dependency-scan",
    "security-review"
  ]
}
```

Identity 很重要，因为系统必须知道：

```text
谁做了什么？
```

例如：

```text
Agent A → 修改代码
Agent B → 审查代码
Agent C → 执行部署
```

最终审计日志：

```text
security-agent
reviewed
commit abc123
```

---

# 九、Agent Capability

Agent 不应该拥有所有能力。

定义：

```java
class Capability {

    String name;

    Set<String> tools;

    Set<String> permissions;

    RiskLevel riskLevel;
}
```

例如：

```text
Research Agent
    search
    browser
    document-read

Coding Agent
    git-read
    git-write
    compiler
    test

Deployment Agent
    kubectl
    cloud-api
```

于是：

```text
Agent
=
Identity
+
Role
+
Capability
```

---

# 十、Agent Role

角色决定：

> “这个 Agent 应该负责什么？”

例如：

```text
Architect
Developer
Tester
Security Reviewer
DevOps
Researcher
Planner
```

Role 不等于 Prompt。

Prompt 只是实现 Role 的一种方式。

更好的架构：

```text
Role
 ↓
Policy
 ↓
Capability
 ↓
Prompt
 ↓
Tools
```

这样角色就成为真正的系统级概念。

---

# 十一、Orchestrator：Multi-Agent 的“大脑”

Multi-Agent 最大的问题不是：

```text
Agent 会不会工作？
```

而是：

> **谁决定 Agent 应该做什么？**

因此通常需要：

# Orchestrator

例如：

```text
User
 ↓
Orchestrator
 ↓
Task Decomposition
 ↓
Agent Selection
 ↓
Execution
 ↓
Aggregation
```

Orchestrator 可以是：

```text
Rule-based
LLM-based
Workflow-based
Hybrid
```

---

# 十二、Rule-Based Orchestrator

例如：

```java
if (task.type() == CODE_REVIEW) {
    return securityAgent;
}

if (task.type() == CODING) {
    return codingAgent;
}

if (task.type() == TESTING) {
    return testingAgent;
}
```

优点：

```text
稳定
可预测
容易审计
成本低
```

缺点：

```text
缺乏灵活性
复杂任务适应能力弱
```

企业系统非常适合使用 Rule + LLM Hybrid。

---

# 十三、LLM-Based Orchestrator

例如：

```text
User:
Analyze this production issue.
```

Orchestrator：

```text
Need:
1. Trace analysis
2. Log analysis
3. Database analysis
```

然后：

```text
Trace → Observability Agent
Log → Log Analysis Agent
Database → DBA Agent
```

优势：

```text
Dynamic
Flexible
Adaptive
```

缺点：

```text
不可预测
成本高
可能错误分配任务
```

---

# 十四、Hybrid Orchestrator

生产环境推荐：

```text
                    Orchestrator
                         │
              ┌──────────┴──────────┐
              │                     │
          Deterministic            LLM
              │                     │
        Security Rules       Dynamic Planning
        Permission           Task Decomposition
        Routing              Agent Selection
```

也就是：

> **确定性的事情交给程序，动态决策交给 LLM。**

这是企业级 Multi-Agent 的关键设计思想。

---

# 十五、Multi-Agent 的五种经典架构

可以把 Multi-Agent Architecture 归纳为五种。

---

## Architecture 1：Supervisor

```text
              Supervisor
             /     |      \
            ▼      ▼       ▼
        Agent A  Agent B  Agent C
```

Supervisor 负责：

```text
Task
Routing
Coordination
Aggregation
```

最容易实现。

---

# 十六、Supervisor Pattern

例如：

```text
User
 ↓
Supervisor
 ↓
"需要查询数据库"
 ↓
DB Agent
 ↓
Result
 ↓
Supervisor
 ↓
"需要分析代码"
 ↓
Coding Agent
```

核心：

```text
Supervisor
=
Central Coordinator
```

优点：

```text
简单
集中管理
容易实现
容易观察
```

缺点：

```text
Central Bottleneck
Single Point of Failure
Context Bottleneck
```

---

# 十七、Architecture 2：Hierarchical Multi-Agent

复杂任务：

```text
                    Root Agent
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         Research Lead Dev Lead Test Lead
              │          │          │
          ┌───┴───┐   ┌──┴───┐   ┌──┴───┐
          ▼       ▼   ▼      ▼   ▼      ▼
        Agent   Agent Agent Agent Agent Agent
```

这是：

# Hierarchical Agent Architecture

类似传统企业组织结构：

```text
CEO
 ↓
Manager
 ↓
Engineer
```

优点：

```text
适合大型复杂任务
上下文隔离
职责清晰
```

缺点：

```text
层级通信成本
决策延迟
管理复杂
```

---

# 十八、Architecture 3：Peer-to-Peer

Agent 之间直接通信：

```text
Agent A
 ↕
Agent B
 ↕
Agent C
 ↕
Agent D
```

没有中心 Supervisor。

例如：

```text
Researcher → Analyst
Analyst → Reviewer
Reviewer → Researcher
```

优点：

```text
去中心化
灵活
高自治
```

缺点：

```text
通信复杂
容易产生循环
难以治理
难以 Debug
```

因此生产环境需要：

```text
Message Broker
Correlation ID
TTL
Loop Detection
```

---

# 十九、Architecture 4：Blackboard

所有 Agent 共享一个：

# Blackboard

例如：

```text
                  Blackboard
              ┌───────────────┐
              │ Task State    │
              │ Findings      │
              │ Decisions     │
              │ Artifacts     │
              └───────┬───────┘
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
     Agent A       Agent B       Agent C
```

Agent：

```text
Read Blackboard
 ↓
Do Work
 ↓
Write Result
```

这非常适合：

```text
Research
Analysis
Collaborative Problem Solving
```

---

# 二十、Architecture 5：Pipeline

类似流水线：

```text
Agent A
  ↓
Agent B
  ↓
Agent C
  ↓
Agent D
```

例如：

```text
Research
 ↓
Analysis
 ↓
Implementation
 ↓
Testing
 ↓
Review
```

非常适合：

```text
明确阶段
明确输入输出
```

但动态性比较低。

---

# 二十一、五种架构对比

| Architecture | 控制   | 灵活性 | 复杂度 | 适用场景 |
| ------------ | ---- | --: | --: | ---- |
| Supervisor   | 中心化  |   高 |   低 | 通用   |
| Hierarchical | 分层   |   高 |   高 | 大型任务 |
| P2P          | 去中心  |  很高 |  很高 | 高自治  |
| Blackboard   | 共享状态 |   高 |   中 | 协作分析 |
| Pipeline     | 固定流程 |   低 |   低 | 稳定流程 |

实际生产环境往往是：

```text
Hierarchical
+
Supervisor
+
Pipeline
+
Blackboard
```

混合架构。

---

# 二十二、Multi-Agent Communication

Multi-Agent 最核心的问题之一：

> Agent 怎么通信？

最简单：

```text
Agent A
 ↓
Agent B
```

但生产系统应该定义：

```text
Message
```

例如：

```json
{
  "messageId": "msg-123",
  "sender": "research-agent",
  "receiver": "coding-agent",
  "taskId": "task-001",
  "type": "FINDING",
  "timestamp": "2026-08-22T10:00:00Z",
  "payload": {
    "rootCause": "slow database query"
  }
}
```

---

# 二十三、Agent Message Protocol

推荐消息至少包含：

```text
messageId
taskId
sender
receiver
type
timestamp
correlationId
causationId
payload
priority
ttl
```

其中：

```text
correlationId
```

非常重要。

例如：

```text
User Request
correlationId = 1001
```

所有 Agent：

```text
Agent A → 1001
Agent B → 1001
Agent C → 1001
```

这样可以完整追踪一次任务。

---

# 二十四、同步 vs 异步通信

## Synchronous

```text
Agent A
 ↓
Agent B
 ↓
Response
```

适合：

```text
简单查询
实时决策
低延迟
```

---

## Asynchronous

```text
Agent A
 ↓
Message Broker
 ↓
Agent B
```

例如：

```text
Kafka
RabbitMQ
Redis Streams
```

适合：

```text
长任务
高并发
事件驱动
Agent 解耦
```

---

# 二十五、为什么 Multi-Agent 很像微服务？

这是一个非常有意思的架构类比：

| Microservices     | Multi-Agent            |
| ----------------- | ---------------------- |
| Service           | Agent                  |
| API               | Message / Tool         |
| Service Registry  | Agent Registry         |
| Kafka             | Agent Message Bus      |
| DB                | Agent Memory           |
| Workflow          | Orchestration          |
| IAM               | Agent Permission       |
| Distributed Trace | Agent Trace            |
| Circuit Breaker   | Agent Failure Recovery |

因此：

> **Multi-Agent Engineering 很大程度上是在把传统 Distributed Systems 的思想应用到 AI 系统。**

对于 Java 后端工程师，这是非常有优势的切入点。

---

# 二十六、Agent Registry

如果 Agent 数量越来越多：

```text
Agent A
Agent B
Agent C
...
Agent N
```

Orchestrator 如何知道：

```text
谁能做什么？
```

需要：

# Agent Registry

例如：

```json
{
  "agentId": "database-agent",
  "capabilities": [
    "sql-analysis",
    "query-optimization",
    "schema-analysis"
  ],
  "status": "AVAILABLE",
  "load": 0.35
}
```

Registry 类似：

```text
Service Discovery
```

---

# 二十七、Agent Discovery

高级系统甚至可以：

```text
Task
 ↓
Capability Matching
 ↓
Find Agents
 ↓
Select Agent
```

例如：

```text
Task:
Analyze PostgreSQL performance.
```

匹配：

```text
database-agent
observability-agent
performance-agent
```

然后根据：

```text
Capability
Load
Latency
Cost
Trust
Permission
```

选择。

这已经很接近：

# Intelligent Service Discovery

---

# 二十八、Agent Load Balancing

如果有：

```text
5 Coding Agents
```

可以根据：

```text
CPU
Queue
Token Cost
Latency
Availability
```

进行：

```text
Agent Selection
```

例如：

```text
Agent A load = 90%
Agent B load = 30%
Agent C load = 50%
```

选择：

```text
Agent B
```

因此 Multi-Agent Platform 最终甚至会出现：

```text
Agent Scheduler
```

---

# 二十九、Agent Scheduling

任务可以定义：

```text
priority
deadline
cost
requiredCapability
```

例如：

```json
{
  "task": "security-scan",
  "priority": "HIGH",
  "deadline": 300,
  "requiredCapability": "security-analysis"
}
```

Scheduler：

```text
Task Queue
     │
     ▼
Capability Match
     │
     ▼
Load Balancing
     │
     ▼
Agent Assignment
```

这已经非常像 Kubernetes Scheduler。

---

# 三十、Shared Memory vs Private Memory

Multi-Agent 中一个核心问题：

> Agent 应该共享 Memory 吗？

有三种方案。

## Private Memory

```text
Agent A → Memory A
Agent B → Memory B
```

优点：

```text
隔离
安全
上下文干净
```

缺点：

```text
信息孤岛
```

---

## Shared Memory

```text
Agent A ─┐
Agent B ─┼→ Shared Memory
Agent C ─┘
```

优点：

```text
协作简单
信息共享
```

缺点：

```text
污染
冲突
权限问题
```

---

## Hybrid Memory

生产环境更推荐：

```text
Private Memory
+
Shared Task Memory
+
Global Knowledge
```

例如：

```text
Agent Private Memory
        │
        ▼
   Task Memory
        │
        ▼
 Knowledge Base
```

---

# 三十一、Agent Memory 的一致性

多个 Agent 同时修改共享状态：

```text
Agent A ─┐
         ├──> Task State
Agent B ─┘
```

就会出现：

```text
Race Condition
Lost Update
Conflict
Stale Data
```

因此可以借鉴分布式系统：

```text
Optimistic Lock
Version
CAS
Event Sourcing
CRDT
```

例如：

```json
{
  "taskId": "T001",
  "version": 12,
  "status": "CODING"
}
```

Agent 更新：

```text
UPDATE task
SET status = 'TESTING',
    version = 13
WHERE task_id = 'T001'
AND version = 12;
```

这就是：

# Optimistic Concurrency Control

---

# 三十二、Multi-Agent 中的 Event Sourcing

可以记录：

```text
TaskCreated
AgentAssigned
PlanCreated
ToolExecuted
ObservationReceived
AgentFailed
TaskReplanned
TaskCompleted
```

例如：

```text
TaskCreated
 ↓
AgentAssigned
 ↓
PlanCreated
 ↓
CodeModified
 ↓
TestsFailed
 ↓
ReflectionTriggered
 ↓
CodeModified
 ↓
TestsPassed
 ↓
TaskCompleted
```

这样可以实现：

```text
Audit
Replay
Debug
Recovery
```

对于长期运行的 Agent，非常重要。

---

# 三十三、Multi-Agent Failure Model

Multi-Agent 比 Single-Agent 最大的变化之一：

> **Failure Surface 增大。**

可能发生：

```text
Agent Failure
Message Failure
Tool Failure
Memory Failure
Planner Failure
Network Failure
State Conflict
Coordination Failure
```

因此必须设计：

```text
Retry
Timeout
Circuit Breaker
Dead Letter Queue
Fallback Agent
Compensation
Human Escalation
```

---

# 三十四、Agent Circuit Breaker

例如某 Agent 连续失败：

```text
Agent B
FAIL
FAIL
FAIL
FAIL
```

不要继续发送：

```text
Task → Agent B
```

而是：

```text
Agent B
   ↓
Circuit OPEN
   ↓
Fallback Agent C
```

和微服务的 Circuit Breaker 完全类似。

---

# 三十五、Dead Letter Queue

如果任务：

```text
Agent A
 ↓
Message
 ↓
Agent B
 ↓
Failed
```

经过多次 Retry 后仍失败：

```text
Retry
Retry
Retry
```

应该进入：

```text
DLQ
```

然后：

```text
Human Review
```

而不是：

```text
Infinite Retry
```

---

# 三十六、Agent Consensus

多个 Agent 得到不同结论：

```text
Agent A → Root Cause = DB
Agent B → Root Cause = Redis
Agent C → Root Cause = Network
```

怎么办？

需要：

# Consensus

简单方式：

```text
Majority Vote
```

高级方式：

```text
Reviewer Agent
```

例如：

```text
A ─┐
B ─┼→ Consensus Agent → Final Decision
C ─┘
```

---

# 三十七、Debate Architecture

一种比较有意思的 Multi-Agent 模式：

```text
                Problem
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
   Agent A                  Agent B
   Propose                  Critique
        │                     │
        └──────────┬──────────┘
                   ▼
              Judge Agent
                   │
                   ▼
                Decision
```

例如：

```text
Agent A:
Use Redis.

Agent B:
Redis introduces consistency problems.

Judge:
Use local cache + Redis fallback.
```

这种架构适合：

```text
Complex Reasoning
Architecture Design
Risk Analysis
Security Review
```

但 Token 成本明显更高。

---

# 三十八、Blackboard Architecture 的深入理解

Blackboard 可以设计成：

```text
Task
│
├── Objective
├── Current State
├── Findings
├── Decisions
├── Artifacts
├── Open Questions
├── Errors
└── Evidence
```

Agent：

```text
Research Agent
    ↓
write Finding

DB Agent
    ↓
write Evidence

Security Agent
    ↓
write Risk

Architect
    ↓
write Decision
```

最终：

```text
Blackboard
```

成为整个系统的：

# Shared Cognitive State

---

# 三十九、Artifacts

Multi-Agent 不应该只传文本。

还应该传：

```text
Code
SQL
Report
Test Result
Trace
Image
Dataset
Architecture Diagram
```

所以消息系统最好支持：

```text
Message
+
Artifact Reference
```

例如：

```json
{
  "type": "ANALYSIS_RESULT",
  "artifact": {
    "type": "TRACE",
    "location": "trace://abc123"
  }
}
```

避免 Agent 之间传输巨大 payload。

---

# 四十、Agent Context Engineering

Multi-Agent 最大优势之一：

# Context Isolation

例如：

```text
Architect Agent
```

只看到：

```text
Requirements
Architecture Constraints
```

Coding Agent：

```text
Architecture
Code
Task
```

Testing Agent：

```text
Code
Test Requirements
```

Security Agent：

```text
Code
Security Policy
```

这样：

```text
Context ↓
Noise ↓
Reasoning Quality ↑
```

---

# 四十一、Context Passing

Agent A 不能把所有上下文直接传给 Agent B：

```text
A
 ↓
10MB Context
 ↓
B
```

更好的方式：

```text
A
 ↓
Structured Result
 ↓
B
```

例如：

```json
{
  "finding": "N+1 query",
  "confidence": 0.93,
  "evidence": [
    "trace-123",
    "sql-456"
  ],
  "recommendation": "batch query"
}
```

这就是：

# Semantic Context Passing

---

# 四十二、为什么 Agent-to-Agent 不应该依赖自然语言？

这是工程上非常重要的一点。

不推荐：

```text
Agent A:
"I think the database may have some problems..."
```

推荐：

```json
{
  "type": "ROOT_CAUSE",
  "service": "order-service",
  "cause": "N_PLUS_ONE_QUERY",
  "confidence": 0.93,
  "evidence": [
    "trace-123",
    "sql-456"
  ]
}
```

因为：

```text
Natural Language
→ Flexible

Structured Protocol
→ Reliable
```

所以：

> **LLM 可以用自然语言思考，但 Agent 之间应该尽可能使用结构化协议通信。**

---

# 四十三、Multi-Agent Protocol Layer

可以定义：

```text
Agent Communication Protocol
```

包括：

```text
TASK_REQUEST
TASK_ACCEPTED
TASK_REJECTED
TASK_PROGRESS
FINDING
PROPOSAL
CRITIQUE
APPROVAL_REQUEST
TOOL_RESULT
ERROR
FINAL_RESULT
```

例如：

```text
TASK_REQUEST
      ↓
TASK_ACCEPTED
      ↓
PROGRESS
      ↓
FINDING
      ↓
PROPOSAL
      ↓
APPROVAL
      ↓
FINAL_RESULT
```

这比简单：

```text
Agent → Agent
```

健壮很多。

---

# 四十四、Multi-Agent Security

Multi-Agent 的安全风险比 Single-Agent 更复杂。

因为：

```text
Agent A
 ↓
Agent B
 ↓
Tool
 ↓
Production
```

攻击面扩大。

典型风险：

```text
Prompt Injection
Tool Injection
Privilege Escalation
Agent Impersonation
Data Leakage
Memory Poisoning
Cross-Agent Trust
```

---

# 四十五、Agent Identity 与 Zero Trust

每一个 Agent 都应该被视为：

```text
Untrusted Workload
```

即：

```text
Never Trust
Always Verify
```

每次调用：

```text
Agent A → Agent B
```

应该验证：

```text
Identity
Permission
Capability
Task Scope
Token
Signature
```

这就是：

# Agent Zero Trust

---

# 四十六、Memory Poisoning

一个非常危险的问题：

```text
Agent A
 ↓
写入 Shared Memory
 ↓
Agent B
 ↓
相信错误信息
```

例如：

```text
Shared Memory:
"Production DB password = xxx"
```

或者：

```text
"Security policy allows unrestricted deployment."
```

因此 Memory 也必须有：

```text
ACL
Provenance
Trust Level
Validation
Version
Audit
```

---

# 四十七、Agent Trust Model

可以给 Agent 输出增加：

```text
confidence
source
provenance
timestamp
verificationStatus
```

例如：

```json
{
  "finding": "Database bottleneck",
  "confidence": 0.91,
  "source": "observability-agent",
  "verified": true,
  "evidence": [
    "trace-001",
    "metric-002"
  ]
}
```

这会极大提升系统可靠性。

---

# 四十八、Human Approval Architecture

高风险任务：

```text
Agent
 ↓
Risk Engine
 ↓
Risk = HIGH
 ↓
Human Approval
 ↓
Execute
```

例如：

```text
Deploy Production
Delete Data
Rotate Credentials
Change IAM
```

而低风险：

```text
Read Logs
Search Code
Run Unit Tests
```

可以：

```text
Auto Execute
```

因此：

# Risk-Based Autonomy

是企业级 Multi-Agent 非常重要的方向。

---

# 四十九、Agent Observability

Multi-Agent 的可观测性至少需要三个层面。

## Agent Level

```text
Agent Success Rate
Agent Latency
Agent Token Usage
Agent Error Rate
```

## Task Level

```text
Task Completion
Task Duration
Task Cost
Task Replan Count
```

## Collaboration Level

```text
Message Count
Agent Handoffs
Communication Latency
Consensus Rate
Conflict Rate
```

---

# 五十、Distributed Tracing

如果你熟悉 OpenTelemetry，可以把 Multi-Agent Trace 设计成：

```text
trace: task-001

root
│
├── orchestrator.plan
│
├── agent.research
│   ├── tool.search
│   └── tool.browser
│
├── agent.database
│   └── tool.sql
│
├── agent.coding
│   ├── tool.git
│   └── tool.maven
│
├── agent.testing
│   └── tool.junit
│
└── agent.reviewer
```

这样可以回答：

```text
为什么任务花了 120 秒？
```

例如：

```text
Research Agent     10s
DB Agent            5s
Coding Agent       60s
Testing Agent      35s
Reviewer            8s
```

瓶颈立刻清晰。

---

# 五十一、Multi-Agent Metrics

推荐指标：

```text
agent_task_success_total
agent_task_failure_total
agent_execution_duration
agent_tool_calls_total
agent_message_total
agent_replan_total
agent_handoff_total
agent_token_usage
agent_cost
agent_human_approval_total
agent_security_violation_total
```

还可以增加：

```text
agent_conflict_total
agent_consensus_total
agent_memory_retrieval_total
agent_memory_write_total
```

---

# 五十二、Agent Cost Control

Multi-Agent 最大的商业问题之一：

```text
Cost Explosion
```

例如：

```text
1 Task
 ↓
5 Agents
 ↓
Each 10 LLM Calls
 ↓
50 LLM Calls
```

如果每次：

```text
$0.05
```

那么：

```text
$2.50 / Task
```

规模一大：

```text
100,000 tasks
→ $250,000
```

所以需要：

```text
Model Routing
Caching
Context Compression
Agent Budget
Early Termination
Parallel Execution
```

---

# 五十三、Model Routing

不是所有 Agent 都需要最强模型。

例如：

```text
Simple Classification
→ Small Model

Coding
→ Strong Coding Model

Architecture
→ Reasoning Model

Summarization
→ Cheap Model
```

形成：

```text
Task
 ↓
Model Router
 ↓
Model Selection
```

进一步：

```text
Agent Role
+
Task Complexity
+
Risk
→ Model
```

---

# 五十四、Agent Parallelism

如果任务：

```text
Analyze Logs
Analyze Traces
Analyze Database
```

没有依赖关系：

```text
        Task
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
  Logs  Trace   DB
    │     │     │
    └─────┼─────┘
          ▼
       Aggregator
```

可以并行。

Latency：

```text
Sequential:

T = T1 + T2 + T3

Parallel:

T = max(T1,T2,T3)
```

这是 Multi-Agent 最有价值的工程优势之一。

---

# 五十五、Dependency Graph

复杂任务可以建模成：

```text
Task Graph

A
├── B
├── C
└── D

B → E
C → E
D → F

E + F → G
```

这实际上就是：

# DAG

Agent Scheduler 可以根据 DAG：

```text
Ready Tasks
 ↓
Assign Agents
 ↓
Parallel Execute
 ↓
Update Graph
 ↓
Unlock Next Tasks
```

这比简单 Supervisor 更强。

---

# 五十六、Multi-Agent 与 DAG

例如软件开发：

```text
Requirement
    │
    ▼
Architecture
    │
 ┌──┴──┐
 ▼     ▼
Backend Frontend
 │      │
 └──┬───┘
    ▼
Integration Test
    │
    ▼
Security
    │
    ▼
Deployment
```

其中 Backend 和 Frontend 可以并行。

这就是：

```text
Agent
+
DAG Scheduler
```

---

# 五十七、Multi-Agent Runtime 的完整架构

一个企业级平台可以设计成：

```text
                         User / API
                             │
                             ▼
                     ┌──────────────┐
                     │ API Gateway  │
                     └──────┬───────┘
                            ▼
                    ┌────────────────┐
                    │ Task Manager   │
                    └───────┬────────┘
                            ▼
                    ┌────────────────┐
                    │ Orchestrator   │
                    └───────┬────────┘
                            │
                ┌───────────┼───────────┐
                ▼           ▼           ▼
             Planner    Scheduler    Policy
                │           │           │
                └───────────┼───────────┘
                            ▼
                    ┌────────────────┐
                    │ Agent Runtime  │
                    └───────┬────────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
      Agent Pool        Agent Pool        Agent Pool
          │                 │                 │
          ▼                 ▼                 ▼
        Tools             Tools             Tools
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ▼
                    Environment
```

底层：

```text
Memory
Message Bus
State Store
Artifact Store
Vector DB
Observability
Security
```

这已经不再是简单的 AI Application，而是：

# Agent Platform

---

# 五十八、Java/Spring Boot 如何实现 Multi-Agent？

对于 Java 工程师，可以设计：

```text
multi-agent-platform
│
├── agent-core
│
├── agent-runtime
│
├── agent-orchestrator
│
├── agent-memory
│
├── agent-communication
│
├── agent-tool
│
├── agent-policy
│
├── agent-evaluation
│
└── agent-observability
```

---

# 五十九、Agent Core

```java
public interface Agent {

    AgentId id();

    AgentRole role();

    Set<Capability> capabilities();

    AgentResult execute(
        AgentTask task,
        AgentContext context
    );
}
```

---

# 六十、Agent Runtime

```java
public class AgentRuntime {

    public AgentResult run(
        Agent agent,
        AgentTask task
    ) {

        AgentContext context =
            contextManager.create(task);

        while (!context.isCompleted()) {

            Plan plan =
                planner.plan(context);

            Action action =
                executor.next(plan);

            policy.check(agent, action);

            Observation observation =
                executor.execute(action);

            context.update(observation);

            evaluator.evaluate(context);
        }

        return context.result();
    }
}
```

---

# 六十一、Orchestrator

```java
public class AgentOrchestrator {

    public TaskResult execute(Task task) {

        List<SubTask> subtasks =
            planner.decompose(task);

        List<AgentAssignment> assignments =
            scheduler.assign(subtasks);

        List<CompletableFuture<Result>> futures =
            assignments.stream()
                .map(this::executeAsync)
                .toList();

        List<Result> results =
            futures.stream()
                .map(CompletableFuture::join)
                .toList();

        return aggregator.aggregate(results);
    }
}
```

这就可以利用 Java 的：

```text
CompletableFuture
Virtual Threads
ExecutorService
Reactive Streams
Kafka
Redis
PostgreSQL
```

构建 Agent Runtime。

---

# 六十二、为什么 Virtual Threads 对 Agent Runtime 很有价值？

Agent 调用大量外部服务：

```text
LLM
HTTP
Database
Search
MCP
Git
Kubernetes
```

这些通常是：

```text
I/O Bound
```

Java Virtual Threads 可以降低传统线程模型的成本：

```text
Agent Task
 ↓
Virtual Thread
 ↓
LLM Call
 ↓
Tool Call
 ↓
DB
```

非常适合：

```text
High-Concurrency Agent Orchestration
```

---

# 六十三、Kafka 在 Multi-Agent 中的作用

可以设计：

```text
                    Kafka
                      │
       ┌──────────────┼───────────────┐
       ▼              ▼               ▼
agent.task        agent.event     agent.result
       │              │               │
       ▼              ▼               ▼
  Agent Pool       Event Bus      Aggregator
```

Kafka 特别适合：

```text
Event Driven
Async Task
Agent Decoupling
Replay
Audit
```

---

# 六十四、Redis 的作用

Redis 可以用于：

```text
Agent State
Task Lock
Distributed Lock
Short-Term Memory
Rate Limit
Agent Registry Cache
Session
Pub/Sub
```

例如：

```text
agent:task:T001
agent:state:T001
agent:lock:T001
agent:memory:T001
```

但长期 Memory 不应该简单全部塞 Redis。

---

# 六十五、PostgreSQL 的作用

PostgreSQL 可以保存：

```text
Agent Metadata
Task
Task State
Messages
Audit
Execution History
Agent Configuration
```

例如：

```text
agents
agent_tasks
agent_messages
agent_executions
agent_artifacts
agent_audit_logs
```

---

# 六十六、Vector Database 的作用

Vector DB 更适合：

```text
Semantic Memory
Knowledge
Past Tasks
Past Solutions
Agent Skills
```

例如：

```text
Task:
"PostgreSQL query is slow."

Retrieve:
Past similar tasks
```

得到：

```text
Previously:
EXPLAIN showed sequential scan.
Adding index solved the issue.
```

Agent 就能提高效率。

---

# 六十七、Multi-Agent 的最终数据模型

可以抽象：

```text
Agent
│
├── AgentIdentity
├── Capability
├── Policy
├── Memory
└── Runtime

Task
│
├── Goal
├── State
├── DAG
├── Assignments
└── Result

Message
│
├── Sender
├── Receiver
├── TaskId
├── CorrelationId
└── Payload

Execution
│
├── Agent
├── Tool
├── Input
├── Output
├── Duration
└── Cost
```

---

# 六十八、Multi-Agent 的核心设计原则

我认为真正值得记住的是下面 10 条。

### 原则 1：不要为了 Multi-Agent 而 Multi-Agent

先问：

```text
Single Agent 能不能解决？
```

---

### 原则 2：Agent 应该拥有明确职责

不要：

```text
Super Agent
```

什么都做。

应该：

```text
Clear Role
Clear Capability
Clear Permission
```

---

### 原则 3：LLM 决策，程序控制边界

```text
LLM
→ Decide

Program
→ Enforce
```

---

### 原则 4：Agent 之间优先结构化通信

```text
JSON / Schema
>
Natural Language
```

---

### 原则 5：共享状态必须治理

```text
ACL
Version
Audit
Provenance
```

---

### 原则 6：失败必须是架构的一部分

```text
Timeout
Retry
Recovery
Fallback
DLQ
Human Escalation
```

---

### 原则 7：每个 Agent 都应该可观测

```text
Trace
Metric
Log
Event
```

---

### 原则 8：限制 Agent 的自治边界

```text
Budget
Permission
Iteration
Time
Risk
```

---

### 原则 9：确定性的流程尽量使用 Workflow

```text
Workflow
→ Deterministic

Agent
→ Dynamic
```

---

### 原则 10：最终结果必须 Verify

不要：

```text
Agent says SUCCESS
```

就相信。

应该：

```text
Agent
 ↓
Verification
 ↓
Actual SUCCESS
```

---

# 六十九、Multi-Agent Architecture 的终极抽象

把整篇文章压缩成一个模型：

```text
                     ┌───────────────┐
                     │     GOAL      │
                     └───────┬───────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Orchestrator   │
                    └───────┬────────┘
                            │
                     Task Decomposition
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          Agent A         Agent B         Agent C
             │              │              │
             ▼              ▼              ▼
          Private         Private         Private
          Context         Context         Context
             │              │              │
             └──────────────┼──────────────┘
                            │
                      Shared State
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          Memory         Message Bus      Tools
             │              │              │
             └──────────────┼──────────────┘
                            ▼
                       Environment
                            │
                            ▼
                       Observation
                            │
                            ▼
                        Evaluator
                            │
                   ┌────────┴────────┐
                   ▼                 ▼
                 PASS              FAIL
                   │                 │
                   ▼                 ▼
                 Result           Recovery
                                     │
                                     ▼
                                  Re-plan
```

这就是：

# Multi-Agent Operating Model

---

# 七十、Single-Agent → Multi-Agent 的真正演进

可以把整个 AI Agent 技术体系理解成：

```text
LLM
 │
 ▼
RAG
 │
 ▼
Tool Calling
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
Autonomous Agent
 │
 ▼
Multi-Agent
 │
 ▼
Agent Runtime
 │
 ▼
Agent Platform
 │
 ▼
Agentic Enterprise
```

最终的目标不是：

> “让几个 Agent 聊天。”

而是：

> **构建一个可以把复杂目标分解给不同智能角色，在受控权限下并行执行，通过结构化通信共享结果，遇到失败能够恢复，最终经过确定性验证完成业务目标的分布式智能系统。**

---

# 七十一、最终总结

Multi-Agent Architecture 最核心的不是：

```text
多几个 LLM
```

而是：

```text
Decomposition
+
Coordination
+
Communication
+
State
+
Memory
+
Capability
+
Policy
+
Verification
+
Observability
```

从软件架构角度来看：

```text
Single Agent
≈
Application

Multi-Agent
≈
Distributed System

Agent Platform
≈
Operating System / Runtime
```

这也是为什么真正进入企业级 Agent 开发后，会重新遇到大量我们在传统后端架构中已经非常熟悉的问题：

```text
Service Discovery
Message Queue
Distributed Lock
State Machine
DAG Scheduling
Circuit Breaker
Retry
Idempotency
Event Sourcing
Authentication
Authorization
Observability
Cost Control
```

区别只是：

```text
传统分布式系统：
Service 是计算单元

Multi-Agent：
Agent 是认知/决策单元
```

所以，**Multi-Agent Engineering 的本质可以理解成“Distributed Systems + LLM Reasoning + Tool Use + State Management”的结合。**

