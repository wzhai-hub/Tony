---
title: Agent Communication 深入技术解析：从消息协议到智能体通信基础设施
# tags:
#   - nodejs
date: '2026-08-05'
summary: 如何让一组具有不同能力、不同权限、不同状态和不同运行环境的智能体，在一个复杂分布式系统中可靠地交换任务、上下文、状态和结果。
---

# Agent Communication 深入技术解析：从消息协议到智能体通信基础设施

## 摘要

在 Multi-Agent System 中，很多工程讨论都会把注意力放在 Agent 的 Prompt、Memory、Tool Calling 和 Model Selection 上。

但当系统从两个 Agent 扩展到几十甚至几百个 Agent 后，一个更基础的问题会迅速出现：

> **Agent 之间到底应该如何通信？**

如果 Agent Communication 只是：

```text
Agent A → LLM → 自然语言 → Agent B
```

那么系统很快会遇到：

* 消息格式不稳定；
* 上下文不断膨胀；
* Agent 之间高度耦合；
* 消息无法可靠路由；
* 重试导致重复执行；
* 长任务无法恢复；
* Agent 无法知道消息是否可信；
* 无法进行跨 Agent Trace；
* Prompt Injection 可以通过消息传播；
* 很难实现权限控制和审计。

因此，Agent Communication 并不是简单的“Agent A 调用 Agent B”。

从系统架构角度看，它实际上是：

> **一种面向智能体的分布式通信基础设施。**

它需要同时解决：

```text
Identity
Addressing
Discovery
Messaging
Protocol
Routing
Context
State
Delivery
Reliability
Security
Observability
Governance
```

本文将从分布式系统、微服务、消息队列和 LLM Agent 的交叉视角，深入分析 Agent Communication 的技术体系。

---

# 一、Agent Communication 到底是什么？

首先需要明确：

> Agent Communication ≠ Agent Conversation

Conversation 更关注：

```text
A: What do you think?
B: I think...
A: Why?
B: Because...
```

Communication 则关注：

```text
Who
  |
What
  |
To Whom
  |
When
  |
Why
  |
How
  |
What State
  |
What Result
```

因此，一个真正的 Agent Message 至少应该包含：

```text
Message
 |
 +-- Identity
 +-- Sender
 +-- Receiver
 +-- Conversation
 +-- Task
 +-- Message Type
 +-- Payload
 +-- Context
 +-- Timestamp
 +-- Correlation
 +-- Security
 +-- Delivery Metadata
```

可以抽象为：

```text
Agent Communication
=
Identity
+
Protocol
+
Message
+
Routing
+
State
+
Context
+
Reliability
+
Security
+
Observability
```

这已经非常接近传统分布式系统中的 RPC + Message Broker + Service Discovery。

但 Agent Communication 多了一层：

> **Semantic Communication。**

---

# 二、为什么 Agent Communication 比微服务通信更复杂？

传统微服务：

```text
Service A
    |
    | REST / gRPC
    v
Service B
```

Service A 通常知道：

```text
API
Schema
Endpoint
Timeout
Retry
```

而 Agent：

```text
Agent A
    |
    | Semantic Request
    v
Agent B
```

Agent A 不一定知道 Agent B 的内部实现。

它可能只知道：

```text
Agent B
Capability:
    financial-analysis
```

甚至：

```text
“找一个能够分析财务风险的 Agent。”
```

于是通信过程变成：

```text
Goal
 |
 v
Capability Discovery
 |
 v
Agent Selection
 |
 v
Message Routing
 |
 v
Semantic Interpretation
 |
 v
Execution
 |
 v
Result
```

因此 Agent Communication 比传统 RPC 多了一层：

```text
Semantic Layer
```

---

# 三、Agent Communication 的五层模型

一个比较完整的 Agent Communication Architecture 可以划分为五层。

```text
+---------------------------------------+
|         Application / Agent           |
+---------------------------------------+
|       Semantic Communication          |
+---------------------------------------+
|          Agent Protocol               |
+---------------------------------------+
|       Message Transport               |
+---------------------------------------+
|       Network / Infrastructure        |
+---------------------------------------+
```

具体来看。

## Layer 1：Infrastructure

包括：

```text
HTTP
HTTPS
TCP
WebSocket
Kafka
Redis Streams
gRPC
```

解决：

> 数据怎么传过去？

---

## Layer 2：Transport

负责：

```text
Request
Response
Event
Stream
```

解决：

> 消息如何可靠传输？

---

## Layer 3：Agent Protocol

定义：

```text
Task
Message
Capability
Response
Error
Status
```

解决：

> Agent 如何理解通信？

---

## Layer 4：Semantic Layer

定义：

```text
Intent
Goal
Context
Constraints
Expected Output
```

解决：

> Agent 到底想表达什么？

---

## Layer 5：Application

最终完成：

```text
Planning
Reasoning
Execution
Collaboration
```

因此：

> HTTP 是 Transport，不是 Agent Protocol。

同样：

> Kafka 是 Message Transport，不是 Agent Communication Protocol。

这是设计 Agent System 时非常重要的概念。

---

# 四、Agent Identity

Agent Communication 的第一个问题：

> “你是谁？”

传统微服务：

```text
service-name
```

Agent 则需要更加丰富的 Identity：

```json
{
  "agentId": "finance-agent-001",
  "name": "Financial Analysis Agent",
  "version": "2.1.0",
  "organization": "company-a",
  "capabilities": [
    "financial-analysis",
    "risk-analysis"
  ]
}
```

因此可以定义：

```text
Agent Identity
=
Agent ID
+
Version
+
Organization
+
Capabilities
+
Trust Level
```

为什么 Version 很重要？

因为：

```text
finance-agent:v1
finance-agent:v2
finance-agent:v3
```

可能拥有不同：

```text
Input Schema
Output Schema
Capability
Security Policy
Model
```

所以 Agent Identity 应该具备版本语义。

---

# 五、Agent Addressing

传统网络通信：

```text
IP + Port
```

微服务：

```text
service-name
```

Agent：

```text
agent-id
```

但更高级的 Agent System 可能使用：

```text
Capability Addressing
```

例如：

```text
capability://financial-analysis
```

意思不是：

> 调用 finance-agent-001。

而是：

> 找一个能够执行 financial-analysis 的 Agent。

于是：

```text
Task
 |
 v
Capability
 |
 v
Agent Registry
 |
 +---- Agent A
 +---- Agent B
 +---- Agent C
```

然后根据：

```text
Latency
Cost
Load
Reliability
Version
Permission
```

选择最终 Agent。

---

# 六、Agent Discovery

如果系统只有三个 Agent：

```text
research-agent
finance-agent
writer-agent
```

可以写死：

```java
if (task.type().equals("research")) {
    return "research-agent";
}
```

但是如果：

```text
Agent Count = 1000
```

就必须引入 Agent Registry。

```text
                 Agent Registry
                       |
       +---------------+---------------+
       |               |               |
       v               v               v
 Research Agent   Finance Agent    Coding Agent
```

Registry 中保存：

```text
agentId
name
version
endpoint
capabilities
status
load
health
policy
```

例如：

```json
{
  "agentId": "finance-agent-002",
  "endpoint": "https://agent.example.com/finance",
  "capabilities": [
    "financial-analysis",
    "risk-analysis"
  ],
  "status": "READY",
  "load": 0.42
}
```

这和 Kubernetes Service Discovery 或微服务注册中心非常相似。

但 Agent Registry 还多了：

> Capability Discovery。

---

# 七、Capability Discovery

假设 Supervisor 收到：

> “分析公司的信用风险。”

它不应该直接知道：

```text
finance-agent-001
```

而应该查询：

```text
Capability:
credit-risk-analysis
```

Registry 返回：

```text
Agent A
Agent B
Agent C
```

然后 Router 进行选择：

```text
Score =
Capability Match
+
Reliability
+
Latency
+
Cost
+
Load
```

于是：

```text
                    Task
                     |
                     v
              Capability Query
                     |
                     v
               Agent Registry
                     |
          +----------+----------+
          |          |          |
          v          v          v
       Agent A    Agent B    Agent C
          |          |          |
          +----------+----------+
                     |
                     v
                  Router
                     |
                     v
                Selected Agent
```

这意味着：

> Agent Communication 的第一步甚至可能不是发送 Message，而是发现“应该给谁发送 Message”。

---

# 八、Agent Message Model

一个生产级 Agent Message 不应该只是：

```json
{
  "message": "Please analyze this company"
}
```

更合理的是：

```json
{
  "messageId": "msg-10001",
  "conversationId": "conv-20001",
  "taskId": "task-30001",
  "sender": {
    "agentId": "supervisor-agent",
    "version": "3.1"
  },
  "receiver": {
    "agentId": "finance-agent"
  },
  "messageType": "TASK_REQUEST",
  "contentType": "application/json",
  "timestamp": "2026-08-22T12:00:00Z",
  "correlationId": "corr-123",
  "payload": {
    "company": "ABC",
    "analysisType": "credit-risk"
  }
}
```

这里有几个非常重要的 ID。

---

# 九、messageId、taskId、conversationId、correlationId

这四个 ID 经常被混淆。

## messageId

表示：

> 这一条消息。

```text
msg-001
```

---

## taskId

表示：

> 一个业务任务。

```text
task-001
```

一个 Task 可以产生很多 Message：

```text
task-001
 |
 +-- msg-001
 +-- msg-002
 +-- msg-003
 +-- msg-004
```

---

## conversationId

表示：

> 一组相关的 Agent Conversation。

例如：

```text
conversation-001
```

里面可能包含多个 Task。

---

## correlationId

表示：

> 当前请求与相关异步消息之间的关联关系。

例如：

```text
Request
  |
  correlationId = abc
  |
  +--> Agent A
  |
  +--> Agent B
  |
  +--> Agent C
```

这对异步系统尤其重要。

---

# 十、Message Type

Agent Communication 应该明确 Message Type。

例如：

```text
TASK_REQUEST
TASK_ACCEPTED
TASK_REJECTED
TASK_STARTED
TASK_PROGRESS
TASK_RESULT
TASK_FAILED
TASK_CANCELLED
CAPABILITY_QUERY
CAPABILITY_RESPONSE
REVIEW_REQUEST
REVIEW_RESULT
```

不要让 Agent 通过自然语言猜：

```text
“这个消息到底是让我开始执行，
还是让我检查结果？”
```

应该直接：

```json
{
  "messageType": "REVIEW_REQUEST"
}
```

这就是：

> **Protocol over Prompt。**

---

# 十一、自然语言与结构化消息应该如何结合？

不是说自然语言没有价值。

正确架构应该是：

```text
Structured Protocol
       |
       +-- Metadata
       |
       +-- Schema
       |
       +-- Policy
       |
       +-- Semantic Payload
```

例如：

```json
{
  "messageType": "TASK_REQUEST",
  "payload": {
    "goal": "Analyze credit risk",
    "company": "ABC",
    "constraints": {
      "maxLatencyMs": 5000
    }
  }
}
```

其中：

```text
messageType
taskId
sender
receiver
```

是确定性的。

而：

```text
goal
reason
context
```

可以使用自然语言。

因此：

> **机器负责协议，LLM 负责语义。**

---

# 十二、Synchronous Agent Communication

最简单的 Agent Communication 是 Request/Response。

```text
Agent A
   |
   | Request
   v
Agent B
   |
   | Response
   v
Agent A
```

类似：

```text
HTTP
gRPC
```

适合：

```text
短任务
低延迟
明确输入输出
```

例如：

```text
Supervisor
    |
    | analyze()
    v
Finance Agent
    |
    | result
    v
Supervisor
```

Java 可以抽象成：

```java
interface AgentClient {

    AgentResult execute(
        AgentRequest request
    );
}
```

问题是：

如果 Agent B 执行 5 分钟：

```text
A
|
| HTTP connection
|
v
B
```

显然不合理。

---

# 十三、Asynchronous Agent Communication

对于长任务，更适合：

```text
Agent A
   |
   | publish
   v
Message Broker
   |
   v
Agent B
```

例如 Kafka：

```text
agent.task.request
```

Agent B：

```text
consume
   |
   v
execute
   |
   v
publish result
```

结果：

```text
agent.task.result
```

于是：

```text
A
|
v
Kafka
|
v
B
|
v
Kafka
|
v
A
```

这种模型天然支持：

```text
Decoupling
Retry
Buffering
Scaling
Replay
Backpressure
```

---

# 十四、Event-Driven Agent Communication

更进一步，Agent 不需要知道谁会处理消息。

例如：

```text
agent.task.completed
```

可能被：

```text
Reviewer
Auditor
Metrics
Notification
Supervisor
```

同时消费。

```text
                Kafka
                  |
          agent.task.completed
                  |
      +-----------+-----------+
      |           |           |
      v           v           v
  Reviewer      Audit      Metrics
```

这就是：

> Event-driven Agent Collaboration。

Agent 从：

```text
Point-to-Point
```

升级成：

```text
Event-driven
```

系统耦合度会明显下降。

---

# 十五、Request/Response 与 Event 的区别

| 模式               | 特征   | 适用场景       |
| ---------------- | ---- | ---------- |
| Request/Response | 强耦合  | 短任务        |
| Async Request    | 弱耦合  | 长任务        |
| Event            | 松耦合  | 广播         |
| Stream           | 持续通信 | 实时任务       |
| Pub/Sub          | 多消费者 | 多 Agent 协作 |

一个成熟系统往往同时使用：

```text
HTTP/gRPC
+
Kafka
+
WebSocket/SSE
```

而不是只使用一种通信方式。

---

# 十六、Streaming Communication

Agent 有一个特殊需求：

> 用户不希望等待 30 秒后一次性看到结果。

所以需要：

```text
Agent
 |
 +--> progress
 +--> reasoning summary
 +--> tool result
 +--> partial result
 +--> final result
```

例如：

```text
TASK_STARTED
      |
      v
PROGRESS
      |
      v
TOOL_EXECUTED
      |
      v
PROGRESS
      |
      v
PARTIAL_RESULT
      |
      v
TASK_COMPLETED
```

前端可以通过：

```text
SSE
WebSocket
```

接收。

注意：

> Streaming 是 Delivery Layer，不应该和 Agent Protocol 混为一谈。

---

# 十七、Agent Communication 的 State Machine

消息通信不能只关注 Message。

还需要关注：

```text
Task State
```

例如：

```text
CREATED
   |
   v
SENT
   |
   v
RECEIVED
   |
   v
ACCEPTED
   |
   v
RUNNING
   |
   +------> WAITING
   |          |
   |          v
   |        RUNNING
   |
   +------> COMPLETED
   |
   +------> FAILED
```

这解决一个关键问题：

> “消息发出去了以后，到底发生了什么？”

---

# 十八、Delivery Semantics

传统消息系统有：

```text
At Most Once
At Least Once
Exactly Once
```

Agent Communication 最现实的选择通常是：

> **At Least Once + Idempotency**

原因是 Agent Task 经常存在：

```text
timeout
network partition
agent crash
```

例如：

```text
Agent A
   |
   | TASK_REQUEST
   v
Agent B
   |
   | execute
   |
   | timeout
   X
```

A 不知道：

```text
B 没执行？
还是执行成功但 response 丢失？
```

如果 A 重试：

```text
TASK_REQUEST
TASK_REQUEST
```

B 可能执行两次。

因此需要：

```text
idempotencyKey
```

例如：

```text
taskId + operationId
```

---

# 十九、Agent Idempotency

Tool：

```text
payment.execute
```

Request：

```json
{
  "taskId": "task-001",
  "operationId": "payment-001",
  "amount": 100
}
```

服务端：

```text
SETNX
payment:payment-001
```

如果已经执行：

```text
return previous result
```

否则：

```text
execute
store result
return
```

这与传统分布式支付系统完全相似。

所以：

> **Agent Communication 的可靠性问题，本质上仍然是分布式系统问题。**

只是调用者从 Service 变成了 Agent。

---

# 二十、Message Ordering

假设：

```text
Message A:
TASK_STARTED

Message B:
TASK_COMPLETED
```

如果 B 先到：

```text
COMPLETED
   |
   v
STARTED
```

就会产生状态异常。

因此对于需要严格顺序的消息，需要：

```text
sequenceNumber
```

例如：

```json
{
  "taskId": "task-001",
  "sequence": 12
}
```

Consumer：

```text
lastSequence = 11
```

收到：

```text
sequence = 13
```

则：

```text
buffer
```

等待：

```text
sequence = 12
```

---

# 二十一、Agent Context Propagation

这是 Agent Communication 中非常容易被低估的问题。

假设：

```text
User
 |
 v
Supervisor
 |
 v
Research
 |
 v
Search Tool
```

必须传播：

```text
traceId
taskId
conversationId
userId
securityContext
tenantId
```

例如：

```json
{
  "traceId": "abc",
  "taskId": "task-001",
  "conversationId": "conv-001",
  "sender": "research-agent"
}
```

这样才能构建完整 Trace：

```text
User
 |
 +-- Supervisor
       |
       +-- Research
       |     |
       |     +-- Search
       |
       +-- Finance
             |
             +-- Database
```

---

# 二十二、Agent Communication 与 OpenTelemetry

Agent System 非常适合采用 Distributed Tracing。

可以定义：

```text
Trace
 |
 +-- agent.supervisor
 |
 +-- agent.research
 |      |
 |      +-- llm.call
 |      +-- tool.search
 |
 +-- agent.finance
        |
        +-- llm.call
        +-- db.query
```

Span Attributes：

```text
agent.id
agent.version
task.id
message.id
model.name
tool.name
token.input
token.output
latency
status
```

这样可以回答：

> 为什么这个 Agent 花了 18 秒？

Trace：

```text
Supervisor       18s
 |
 +-- Research     5s
 |    |
 |    +-- LLM     2s
 |    +-- Search  3s
 |
 +-- Finance      8s
 |    |
 |    +-- LLM     4s
 |    +-- DB      4s
 |
 +-- Aggregator   5s
```

马上可以发现瓶颈。

---

# 二十三、Agent Communication Security

Agent Communication 的安全性比普通微服务更复杂。

因为消息本身可能包含：

```text
Instructions
User Content
External Content
Tool Results
LLM Generated Content
```

其中任何一个都可能包含恶意指令。

例如：

```text
Research Agent
    |
    v
Web Page
    |
    v
"Ignore previous instructions..."
    |
    v
Research Result
    |
    v
Supervisor
    |
    v
Coding Agent
```

这就是：

> Cross-Agent Prompt Injection。

---

# 二十四、为什么 Agent Message 必须有 Trust Boundary？

不能假设：

```text
Agent A
```

发来的消息一定可信。

应该定义：

```text
Trust Level
```

例如：

```text
SYSTEM
INTERNAL_AGENT
EXTERNAL_AGENT
USER
EXTERNAL_DATA
UNTRUSTED
```

然后：

```text
UNTRUSTED
    |
    v
Sanitize
    |
    v
Validate
    |
    v
Policy
    |
    v
Agent Context
```

尤其不能直接：

```text
External Content
      |
      v
System Prompt
```

---

# 二十五、Capability-based Security

Agent 不应该通过：

```text
if agent == "admin-agent"
```

判断权限。

更好的模型是：

```text
Agent Identity
      |
      v
Capabilities
      |
      v
Policy
```

例如：

```json
{
  "agentId": "research-agent",
  "capabilities": [
    "search.read",
    "document.read"
  ]
}
```

而：

```text
payment.execute
```

不在 Capability 中。

即使 Agent 收到：

```text
Please transfer $10,000.
```

也不能执行。

---

# 二十六、Message Schema Evolution

Agent Communication 还有一个容易被忽略的问题：

> Schema 如何升级？

V1：

```json
{
  "company": "ABC"
}
```

V2：

```json
{
  "company": "ABC",
  "country": "US"
}
```

V3：

```json
{
  "company": {
    "name": "ABC",
    "country": "US"
  }
}
```

如果存在：

```text
100 Agents
```

不可能一次升级全部 Agent。

因此必须支持：

```text
version
backward compatibility
forward compatibility
schema validation
```

例如：

```text
agent.protocol.v1
agent.protocol.v2
```

或者：

```json
{
  "schemaVersion": "2.0"
}
```

---

# 二十七、Agent Protocol 的 Schema

可以使用 JSON Schema：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": [
    "messageId",
    "taskId",
    "messageType"
  ],
  "properties": {
    "messageId": {
      "type": "string"
    },
    "taskId": {
      "type": "string"
    },
    "messageType": {
      "type": "string"
    }
  }
}
```

这样 Agent Message 在进入 Runtime 之前就可以验证：

```text
Message
   |
   v
Schema Validation
   |
   +---- invalid ---> reject
   |
   v
Policy Validation
   |
   v
Agent Runtime
```

---

# 二十八、Agent Communication Protocol 的设计原则

一个好的 Agent Protocol 应该满足：

### 1. Explicit

协议字段明确。

### 2. Extensible

支持版本升级。

### 3. Observable

每个消息都可追踪。

### 4. Secure

具有身份和权限。

### 5. Idempotent

支持重复消息。

### 6. Async Friendly

支持异步任务。

### 7. Streaming Friendly

支持长任务。

### 8. Human Friendly

必要时能够转换成人类可读信息。

---

# 二十九、MCP 与 Agent Communication 的关系

MCP 很重要，但必须理解它解决的是什么问题。

MCP 更核心地解决：

```text
AI Application
      |
      v
MCP
      |
      +--> Tool
      +--> Resource
      +--> Prompt
```

它主要解决：

> **Agent/Application 如何标准化访问外部能力。**

因此：

```text
MCP
=
Agent ↔ Tool/Resource
```

而 Agent-to-Agent：

```text
Agent A ↔ Agent B
```

解决的是：

> **Agent 如何发现、委托、协作和交换任务。**

因此二者不是简单替代关系。

可以组合成：

```text
                  Supervisor Agent
                         |
             +-----------+-----------+
             |                       |
             v                       v
       Agent Communication         MCP
             |                       |
             v                       v
        Worker Agent             Tools
             |
             v
            MCP
             |
             v
           Tools
```

---

# 三十、A2A 与 Agent-to-Agent Communication

A2A 类协议的核心价值可以理解为：

> **让一个 Agent 能够以标准方式发现和调用另一个 Agent 的能力。**

其核心思想包括：

```text
Agent Identity
Agent Capability
Agent Discovery
Task
Message
Artifact
Status
```

因此：

```text
MCP
Agent → Tool

A2A
Agent → Agent
```

这两个方向可以互补。

一个更完整的架构：

```text
                 User
                   |
                   v
             Supervisor
                   |
             A2A Communication
                   |
       +-----------+-----------+
       |           |           |
       v           v           v
   Research     Finance      Coding
       |           |           |
       +-----------+-----------+
                   |
                  MCP
                   |
          +--------+--------+
          |        |        |
         DB       API      Search
```

这实际上形成了两种通信：

```text
Horizontal:
Agent ↔ Agent

Vertical:
Agent ↔ Tool
```

这是理解现代 Agent Protocol 非常关键的架构模型。

---

# 三十一、Agent Communication 与 Kafka 的结合

如果企业已经有 Kafka，那么可以把 Agent Event Bus 建立在 Kafka 上。

Topic：

```text
agent.task.request
agent.task.accepted
agent.task.started
agent.task.progress
agent.task.completed
agent.task.failed
agent.task.cancelled
agent.review.request
agent.review.completed
```

Producer：

```text
Orchestrator
```

Consumer：

```text
Agent Runtime
```

例如：

```text
                 Kafka
                   |
      +------------+------------+
      |            |            |
      v            v            v
 research      finance       coding
 consumer      consumer      consumer
```

Partition Key：

```text
taskId
```

可以保证同一 Task 的消息尽可能进入同一 Partition，从而降低乱序问题。

---

# 三十二、Agent Communication 与 Redis

Redis 更适合：

```text
Short-lived State
Task Lock
Deduplication
Session
Presence
Rate Limit
```

例如：

```text
task:001 -> RUNNING
```

Idempotency：

```text
SETNX message:msg-001
```

Distributed Lock：

```text
lock:task:001
```

Agent Heartbeat：

```text
agent:finance:heartbeat
```

但 Redis 不应该承担所有 Agent Message 的可靠持久化职责。

比较合理的是：

```text
Kafka
=
Durable Event

Redis
=
Fast State
```

---

# 三十三、Agent Communication 与数据库

数据库主要负责：

```text
Task Metadata
Agent Metadata
Audit
Configuration
Long-term State
```

例如：

```text
agent
----------------
id
name
version
status

task
----------------
id
parent_id
agent_id
status
created_at

message
----------------
id
task_id
sender
receiver
type
status
created_at
```

这样系统出现异常时，可以从数据库恢复：

```text
Task
  |
  +-- Messages
  |
  +-- State
  |
  +-- Agent
```

---

# 三十四、一个生产级 Agent Communication 架构

综合起来：

```text
                         User
                           |
                           v
                    API Gateway
                           |
                           v
                  Agent Gateway
                           |
                           v
                    Orchestrator
                           |
              +------------+------------+
              |            |            |
              v            v            v
         Agent Router   Policy       Context
              |
              v
        Agent Registry
              |
      +-------+-------+
      |       |       |
      v       v       v
   Agent A Agent B Agent C
      |       |       |
      +-------+-------+
              |
              v
        Message Broker
              |
             Kafka
              |
      +-------+-------+
      |       |       |
      v       v       v
    State   Memory   Audit
   Redis    Vector    DB
              |
              v
        Tool Gateway
              |
             MCP
              |
      +-------+-------+
      |       |       |
      v       v       v
     API     DB    External Service

              |
              v
        Observability
              |
        OpenTelemetry
              |
       +------+------+
       |             |
    Metrics        Trace
       |             |
   Prometheus       Tempo
       |             |
       +------+------+
              |
           Grafana
```

这已经不是一个简单的 Agent Framework。

它实际上是：

> **Agent Communication Platform。**

---

# 三十五、Agent Communication 的核心性能指标

生产环境必须监控通信质量。

## Latency

```text
Message Latency
Agent Response Latency
End-to-End Task Latency
```

---

## Throughput

```text
messages/sec
tasks/sec
agent executions/sec
```

---

## Reliability

```text
delivery success rate
task success rate
retry rate
failure rate
```

---

## Cost

```text
tokens/message
tokens/task
LLM cost/task
communication overhead
```

---

## Collaboration Efficiency

可以定义：

```text
Collaboration Efficiency
=
Successful Tasks
/
Total Agent Interactions
```

如果一个任务需要：

```text
100 messages
```

才能完成：

```text
简单任务
```

那么架构可能已经过度设计。

---

# 三十六、Agent Communication 的一个重要指标：Communication Overhead

假设：

```text
Task
 |
 +-- Agent A
 |     |
 |     +-- 10 messages
 |
 +-- Agent B
 |     |
 |     +-- 15 messages
 |
 +-- Agent C
       |
       +-- 20 messages
```

总通信：

```text
45 messages
```

如果其中只有：

```text
5 messages
```

真正产生有效信息，那么：

```text
Communication Efficiency
=
5 / 45
≈ 11%
```

这是非常低的。

所以：

> Agent 越多，不一定越智能。

Multi-Agent 的一个核心优化目标就是：

> **减少无效通信。**

---

# 三十七、Communication Compression

Agent 之间不应该不断传递完整上下文。

例如：

```text
Agent A
```

产生：

```text
50,000 tokens
```

Agent B 不需要全部接收。

可以生成：

```text
Context Summary
```

例如：

```json
{
  "keyFindings": [
    "Revenue decreased 12%",
    "Debt ratio increased"
  ],
  "riskScore": 0.82,
  "evidenceIds": [
    "doc-001",
    "doc-002"
  ]
}
```

这样：

```text
50,000 tokens
        |
        v
    Compression
        |
        v
    500 tokens
```

这对大型 Multi-Agent System 非常重要。

---

# 三十八、Context Reference 比 Context Copy 更好

不要：

```text
Agent A
  |
  | copy 100KB context
  v
Agent B
```

应该：

```text
Agent A
  |
  | artifactId
  v
Agent B
```

例如：

```json
{
  "artifactId": "artifact-1001",
  "summary": "Financial analysis completed",
  "location": "knowledge-store"
}
```

Agent B：

```text
artifactId
   |
   v
Artifact Store
   |
   v
retrieve
```

这样可以减少：

```text
Token
Network Traffic
Memory
Context Window
```

---

# 三十九、Artifact-oriented Communication

复杂 Agent 系统可以将结果分成：

```text
Message
Artifact
State
```

Message：

```text
“分析已经完成。”
```

Artifact：

```text
financial-report.pdf
```

State：

```text
task.status = COMPLETED
```

不要把所有内容都塞进 Message。

因此：

```text
Agent Communication
      |
      +-- Message
      +-- Artifact
      +-- State
```

这比：

```text
Everything = Message
```

更加可扩展。

---

# 四十、Agent Communication 的典型反模式

## 反模式一：Agent 之间直接调用 HTTP

```text
A -> B
B -> C
C -> D
```

最终：

```text
Tight Coupling
```

应该通过：

```text
Orchestrator
+
Protocol
```

或者：

```text
Event Bus
```

进行解耦。

---

## 反模式二：所有内容使用自然语言

```text
"Hey, I think..."
```

结果：

```text
Ambiguity
```

应该：

```text
Structured Message
+
Semantic Payload
```

---

## 反模式三：共享全部 Context

结果：

```text
Context Explosion
```

应该：

```text
Context Isolation
+
Selective Sharing
```

---

## 反模式四：无限 Retry

结果：

```text
Agent Loop
```

应该：

```text
Retry Budget
+
Timeout
+
Circuit Breaker
+
Re-plan
```

---

## 反模式五：Agent 直接拥有全部 Tool

```text
Agent
 |
 +-- DB
 +-- Shell
 +-- Payment
 +-- Deploy
 +-- Email
```

这是巨大的安全风险。

应该：

```text
Agent
 |
 v
Capability Policy
 |
 v
Tool Gateway
```

---

# 四十一、Agent Communication 的设计原则

最终可以浓缩成十二条原则。

### Principle 1

**Protocol First**

不要依赖自然语言作为唯一协议。

### Principle 2

**Identity First**

每个 Agent 必须有明确身份。

### Principle 3

**Capability First**

按能力发现 Agent，而不是硬编码 Agent ID。

### Principle 4

**Task First**

通信围绕 Task，而不是围绕 Conversation。

### Principle 5

**Async by Default**

长任务优先异步。

### Principle 6

**At Least Once + Idempotency**

不要幻想所有 Agent 调用都能 Exactly Once。

### Principle 7

**Context Isolation**

Agent 只接收真正需要的信息。

### Principle 8

**Reference over Copy**

大型 Context 使用 Artifact Reference。

### Principle 9

**Policy before Execution**

Agent 的意图不能直接等价于执行权限。

### Principle 10

**Observable by Design**

所有 Agent Message 都应该能够追踪。

### Principle 11

**Version Everything**

Agent、Protocol、Schema 都需要版本管理。

### Principle 12

**Minimize Communication**

最好的 Multi-Agent System 不是消息最多，而是用最少的通信完成任务。

---

# 四十二、最终架构认知

如果把传统系统的通信模型总结为：

```text
Client
  |
  v
API
  |
  v
Service
```

那么 Agent Communication 可以抽象为：

```text
Goal
 |
 v
Agent
 |
 v
Capability Discovery
 |
 v
Agent Selection
 |
 v
Protocol
 |
 v
Message
 |
 v
Routing
 |
 v
Agent
 |
 v
Task Execution
 |
 v
Artifact
 |
 v
Event
 |
 v
Next Agent
```

于是整个 Agent System 最终形成：

```text
                       Goal
                        |
                        v
                   Orchestrator
                        |
                Capability Discovery
                        |
                        v
                 Agent Selection
                        |
                        v
                 Communication
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
       Agent A       Agent B       Agent C
          |             |             |
          +-------------+-------------+
                        |
                        v
                    Artifacts
                        |
                        v
                     Events
                        |
                        v
                  Next Decision
                        |
                        +------+
                               |
                               v
                          Orchestrator
```

这其实形成了一个闭环：

```text
Discover
   ↓
Communicate
   ↓
Execute
   ↓
Observe
   ↓
Evaluate
   ↓
Re-plan
   ↓
Communicate
```

这才是 Agent Collaboration 真正意义上的通信机制。

---

# 四十三、结语

Agent Communication 的本质，并不是让两个 LLM “互相聊天”。

它真正解决的是：

> **如何让一组具有不同能力、不同权限、不同状态和不同运行环境的智能体，在一个复杂分布式系统中可靠地交换任务、上下文、状态和结果。**

因此，一个成熟的 Agent Communication Architecture 至少应该包含：

```text
Agent Identity
        +
Capability Discovery
        +
Message Protocol
        +
Task Management
        +
Routing
        +
Async Messaging
        +
Context Propagation
        +
State Management
        +
Idempotency
        +
Security
        +
Observability
        +
Schema Evolution
```

最终可以形成这样一个核心架构公式：

```text
Agent Communication
=
Semantic Protocol
+
Distributed Messaging
+
Context Engineering
+
Capability Discovery
+
Security
+
Reliability
+
Observability
```

而 MCP、A2A、HTTP、gRPC、Kafka、WebSocket 等技术，并不是互相竞争的同一层技术。

更合理的理解是：

```text
                Agent System
                     |
          +----------+----------+
          |                     |
          v                     v
    Agent-to-Agent          Agent-to-Tool
       A2A-like                 MCP
          |                     |
          +----------+----------+
                     |
                Transport
                     |
        +------------+------------+
        |            |            |
       HTTP         gRPC        Kafka
        |            |            |
        +------------+------------+
                     |
                Infrastructure
```

未来真正有竞争力的 Agent 平台，不会只是“拥有很多 Agent”，而会拥有一套成熟的：

> **Agent Communication Fabric**

它类似于今天微服务世界中的：

```text
API Gateway
+
Service Mesh
+
Message Bus
+
Service Discovery
+
Distributed Tracing
+
IAM
```

只不过服务的主体从：

```text
Service
```

逐渐演进成：

```text
Agent
```

而通信的内容也从：

```text
RPC Request
```

逐渐演进成：

```text
Goal
+
Task
+
Context
+
Capability
+
Artifact
+
Decision
```

这意味着 Agent Communication 很可能成为未来 Agent Infrastructure 中与 **Model、Memory、Tool、Workflow** 同等重要的一层基础设施。
