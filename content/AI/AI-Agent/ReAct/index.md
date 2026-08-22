---
title: ReAct 深入技术解析：从 Reasoning + Acting 到生产级 AI Agent
# tags:
#   - nodejs
date: '2026-08-05'
summary: ReAct（Reasoning and Acting）是现代 AI Agent 最重要的基础范式之一。它并不是简单地让大模型“边思考边调用工具”，而是一种将**推理、行动、环境反馈和状态更新**组织成闭环的 Agent 架构。
---

# ReAct 深入技术解析：从 Reasoning + Acting 到生产级 AI Agent

> ReAct（Reasoning and Acting）是现代 AI Agent 最重要的基础范式之一。它并不是简单地让大模型“边思考边调用工具”，而是一种将**推理、行动、环境反馈和状态更新**组织成闭环的 Agent 架构。
>
> 从工程角度看，ReAct 真正重要的地方在于：它把一个传统的“一次性 LLM 调用”转变成了一个能够**感知环境、执行动作、观察结果、动态调整策略**的智能系统。

---

## 1. 为什么需要 ReAct？

传统 LLM 的工作模式可以抽象为：

```text
User
  │
  ▼
LLM
  │
  ▼
Answer
```

例如用户问：

> 2026 年 8 月 22 日广州天气怎么样？是否适合跑步？

如果模型只依赖训练数据，它无法知道实时天气。

即使模型拥有一定的知识，也存在两个天然问题：

1. 知识存在时间边界；
2. 模型无法直接访问外部世界。

因此，需要引入工具：

```text
LLM
 │
 ├── Weather API
 ├── Database
 ├── Search Engine
 ├── Calculator
 └── Internal Service
```

问题随之出现：

> 模型什么时候调用工具？
>
> 调用哪个工具？
>
> 参数是什么？
>
> 调用之后如何理解结果？
>
> 如果结果不正确怎么办？
>
> 是否需要继续调用工具？
>
> 什么时候应该停止？

这实际上已经不是简单的 Prompt 问题，而是一个**决策循环问题**。

ReAct 正是解决这个问题的经典范式。

---

# 2. ReAct 到底是什么？

ReAct 的核心思想可以概括为：

```text
Reasoning
    ↓
Action
    ↓
Observation
    ↓
Reasoning
    ↓
Action
    ↓
Observation
    ↓
...
    ↓
Final Answer
```

也就是：

> **推理 → 行动 → 观察 → 再推理 → 再行动**

形式化表示：

```text
Thought_t
   ↓
Action_t
   ↓
Observation_t
   ↓
Thought_(t+1)
```

最终：

```text
Thought → Action → Observation → ... → Final Answer
```

ReAct 论文将这种思想系统化之后，对后来的：

* Tool Calling
* Function Calling
* AI Agent
* AutoGPT
* LangChain Agent
* LangGraph
* MCP Agent
* Browser Agent
* Coding Agent

都产生了非常大的影响。

---

# 3. ReAct 与普通 LLM 调用的本质区别

普通 LLM：

```text
Input → LLM → Output
```

ReAct：

```text
                ┌───────────────┐
                │               │
                ▼               │
User → LLM → Decision → Tool → Observation
        ▲                       │
        │                       │
        └───────────────────────┘
```

最大的变化不是“增加了工具”。

真正的变化是：

> **LLM 从一个静态文本生成器，变成了 Agent Loop 中的决策核心。**

---

# 4. ReAct 的四个核心概念

一个典型 ReAct Agent 至少包含四个核心概念。

## 4.1 Reasoning

Reasoning 是模型根据当前状态进行决策。

例如：

```text
用户：
帮我查询上海今天的天气，并判断是否适合跑步。
```

Agent 可能形成这样的内部决策：

```text
需要实时天气数据
        ↓
调用 weather 工具
        ↓
获得温度、湿度、降雨概率
        ↓
根据跑步标准进行判断
```

这里最重要的是：

> Reasoning 不等于“生成答案”。

它的作用是：

**决定下一步应该做什么。**

---

# 5. Action

Action 是 Agent 对外部环境执行的动作。

例如：

```json
{
  "tool": "weather",
  "arguments": {
    "city": "Shanghai"
  }
}
```

Action 可以是：

```text
调用 API
查询数据库
执行 SQL
搜索互联网
读取文件
执行代码
发送消息
创建订单
调用 MCP Tool
```

所以 Action 本质上是：

```text
Agent → Environment
```

---

# 6. Observation

Tool 执行之后返回结果：

```json
{
  "temperature": 29,
  "humidity": 75,
  "rainProbability": 0.8
}
```

这就是 Observation。

即：

```text
Environment → Agent
```

Agent 接下来必须理解：

> 这个结果意味着什么？

例如：

```text
温度 29℃
湿度 75%
降雨概率 80%
```

那么下一轮推理可能得出：

```text
降雨概率较高
不适合户外跑步
建议改为室内运动
```

因此：

> Observation 是 ReAct 闭环中非常重要的一环。

没有 Observation：

```text
Reasoning → Action → END
```

Agent 无法根据真实世界反馈调整策略。

---

# 7. ReAct 的核心：Agent Loop

从软件工程角度看，ReAct 最重要的不是 Prompt，而是：

```text
Agent Loop
```

最简单的伪代码：

```java
while (!finished) {

    Decision decision = llm.decide(state);

    if (decision.isFinalAnswer()) {
        return decision.getAnswer();
    }

    Action action = decision.getAction();

    Observation observation =
        toolExecutor.execute(action);

    state.addObservation(observation);
}
```

这段代码实际上已经表达了 ReAct 的核心。

---

# 8. 一个完整的 ReAct 执行过程

假设用户：

> 帮我计算今天北京的天气，并判断是否适合跑步。

Agent 第一次调用：

```text
User Request
    ↓
LLM
```

模型发现：

```text
需要实时天气
```

于是：

```text
Action:
weather("Beijing")
```

工具执行：

```text
Observation:
temperature = 26
humidity = 45
rainProbability = 10%
windSpeed = 2m/s
```

Agent 第二轮：

```text
Observation
    ↓
LLM
    ↓
Reasoning
```

模型判断：

```text
26℃
湿度适中
降雨概率低
风速较低
```

最终：

```text
Final Answer:
今天北京适合跑步。
```

完整过程：

```text
User
 │
 ▼
LLM
 │
 │ Action: weather(Beijing)
 ▼
Weather API
 │
 │ Observation
 ▼
LLM
 │
 ▼
Final Answer
```

---

# 9. ReAct 的状态模型

生产级 Agent 通常不会只保存一个字符串。

更合理的状态：

```java
class AgentState {

    String userQuery;

    List<Message> messages;

    List<ToolCall> toolCalls;

    List<Observation> observations;

    Map<String, Object> variables;

    int iteration;

    boolean finished;
}
```

可以抽象为：

```text
AgentState
│
├── User Input
├── Conversation History
├── Current Reasoning Context
├── Tool Calls
├── Tool Results
├── Memory
├── Variables
├── Iteration Count
└── Execution Status
```

这实际上已经开始接近现代 Agent Framework 的核心设计。

---

# 10. ReAct 与 Chain-of-Thought 的区别

这是理解 ReAct 非常重要的一点。

Chain-of-Thought：

```text
Problem
  ↓
Reasoning
  ↓
Answer
```

ReAct：

```text
Problem
  ↓
Reasoning
  ↓
Action
  ↓
Observation
  ↓
Reasoning
  ↓
Action
  ↓
Observation
  ↓
Answer
```

因此：

| 能力         | CoT | ReAct |
| ---------- | --- | ----- |
| 多步推理       | ✅   | ✅     |
| 调用外部工具     | ❌   | ✅     |
| 获取实时信息     | ❌   | ✅     |
| 环境反馈       | ❌   | ✅     |
| 动态调整策略     | 有限  | ✅     |
| Agent Loop | ❌   | ✅     |

简单来说：

> CoT 主要解决“怎么想”。

而：

> ReAct 解决“怎么想 + 怎么做”。

---

# 11. ReAct 与 Function Calling 的区别

现代 LLM API 通常支持：

```text
Function Calling
```

例如：

```json
{
  "name": "getWeather",
  "arguments": {
    "city": "Beijing"
  }
}
```

这是不是 ReAct？

严格来说：

> Function Calling 是一种工具调用机制，而 ReAct 是一种 Agent 决策范式。

二者属于不同层次。

可以理解为：

```text
                 Agent Architecture
                        │
                      ReAct
                        │
                 ┌──────┴──────┐
                 │             │
              Reasoning      Acting
                                │
                         Function Calling
                                │
                              Tool
```

因此：

```text
ReAct ≠ Function Calling
```

但是现代 Agent 经常：

```text
ReAct + Function Calling
```

一起使用。

---

# 12. ReAct 与 Plan-and-Execute

另外一个重要架构是：

```text
Plan → Execute
```

例如：

```text
用户：
帮我规划一次日本旅行。
```

Plan-and-Execute：

```text
Planner
 │
 ├── 查询航班
 ├── 查询酒店
 ├── 规划路线
 ├── 查询景点
 └── 生成预算
       ↓
Executor
```

而 ReAct 更倾向于：

```text
Think
 ↓
Action
 ↓
Observe
 ↓
Think
 ↓
Action
 ↓
Observe
```

两者区别：

| 特征    | ReAct | Plan-and-Execute |
| ----- | ----- | ---------------- |
| 动态调整  | 强     | 中                |
| 前期规划  | 弱     | 强                |
| 工具调用  | 强     | 强                |
| 长任务   | 一般    | 更适合              |
| 环境变化  | 非常适合  | 需要重新规划           |
| 实现复杂度 | 中     | 高                |

生产系统甚至可以组合：

```text
Planner
   ↓
ReAct Executor
   ↓
Tool
   ↓
Observation
   ↓
Re-plan
```

---

# 13. ReAct 的真正工程难点

很多教程会把 ReAct 简化成：

```text
Thought
Action
Observation
```

但生产系统真正困难的是下面这些问题：

```text
1. Tool Selection
2. Parameter Generation
3. Tool Failure
4. Hallucination
5. Infinite Loop
6. Context Explosion
7. State Management
8. Permission Control
9. Parallel Execution
10. Termination
```

下面逐个分析。

---

# 14. Tool Selection

假设系统有 100 个工具：

```text
search()
weather()
database()
jira()
github()
email()
calendar()
payment()
...
```

用户：

> 查询一下订单状态。

Agent 必须选择：

```text
getOrderStatus()
```

而不是：

```text
search()
```

Tool Selection 本质上是：

```text
User Intent
     ↓
Tool Matching
     ↓
Tool Selection
```

因此工具描述非常重要。

例如：

```json
{
  "name": "getOrderStatus",
  "description": "根据订单ID查询订单当前状态",
  "parameters": {
    "orderId": "string"
  }
}
```

如果 description 写得很差：

```text
查询订单
```

模型的工具选择准确率通常会下降。

---

# 15. Tool Description 实际上是一种 API Contract

从软件工程角度看：

```text
Tool Definition
```

实际上类似：

```text
Interface
```

例如 Java：

```java
interface WeatherService {

    Weather getWeather(String city);
}
```

LLM 看到的是：

```json
{
  "name": "getWeather",
  "description": "查询指定城市实时天气",
  "parameters": {
    "city": {
      "type": "string"
    }
  }
}
```

所以可以把 Tool 看成：

```text
LLM-facing API
```

这意味着：

> Tool Schema 设计本身就是 Agent Engineering 的重要能力。

---

# 16. Tool Calling 的参数问题

用户：

> 查一下广州天气。

LLM 必须生成：

```json
{
  "city": "Guangzhou"
}
```

但是可能出现：

```json
{
  "city": "广州市"
}
```

甚至：

```json
{
  "city": null
}
```

所以生产系统需要：

```text
LLM Output
    ↓
Schema Validation
    ↓
Parameter Validation
    ↓
Tool Execution
```

例如：

```java
if (city == null || city.isBlank()) {
    throw new InvalidToolArgumentException();
}
```

不能直接信任 LLM。

这是一个非常重要的工程原则：

> **LLM 是概率系统，Tool Executor 必须是确定性系统。**

---

# 17. Tool Failure

现实环境中的工具一定会失败。

例如：

```text
HTTP 500
Timeout
Rate Limit
Invalid Parameter
Authentication Error
Database Error
```

如果：

```text
Tool → Exception
```

直接终止 Agent：

```text
Agent
 ↓
Tool
 ↓
Exception
 ↓
END
```

用户体验很差。

更合理的是：

```text
Tool
 ↓
Failure
 ↓
Observation
 ↓
LLM
 ↓
Retry / Alternative Tool / Ask User
```

例如：

```text
Observation:
Weather API timeout.
```

Agent 可以：

```text
Action:
调用备用天气 API
```

或者：

```text
Action:
稍后重试
```

---

# 18. Retry 不能简单地无限重试

错误设计：

```java
while (true) {
    callTool();
}
```

这会产生：

```text
Infinite Loop
```

生产级 Agent 应该有：

```text
maxIterations
maxToolCalls
timeout
tokenBudget
retryLimit
```

例如：

```java
final int MAX_ITERATIONS = 10;
final int MAX_TOOL_CALLS = 20;
```

Agent Loop：

```java
for (int i = 0; i < MAX_ITERATIONS; i++) {

    Decision decision = llm.decide(state);

    if (decision.isFinal()) {
        return decision.answer();
    }

    execute(decision.action());
}
```

这其实是 Agent 系统的“熔断机制”。

---

# 19. ReAct 为什么容易产生 Infinite Loop？

例如：

```text
LLM
 ↓
search()
 ↓
Observation
 ↓
LLM
 ↓
search()
 ↓
Observation
 ↓
LLM
 ↓
search()
```

模型可能不断重复相同动作。

因此生产系统应该检测：

```text
Same Tool
+
Same Parameters
+
Same Context
```

例如：

```java
if (lastAction.equals(currentAction)) {
    repeatCount++;
}
```

如果：

```text
repeatCount >= 3
```

可以：

```text
Stop
```

或者：

```text
Ask LLM to change strategy
```

---

# 20. ReAct 的终止条件

Agent 必须知道：

> 什么时候停止？

通常有几种条件。

### 条件一：模型产生 Final Answer

```text
LLM → Final
```

### 条件二：达到最大迭代次数

```text
iteration >= MAX_ITERATIONS
```

### 条件三：超时

```text
executionTime >= timeout
```

### 条件四：Token Budget 用尽

```text
tokens >= maxTokens
```

### 条件五：工具调用次数超过限制

```text
toolCalls >= maxToolCalls
```

所以：

```text
Termination Policy
```

是生产级 Agent 必不可少的组件。

---

# 21. ReAct 的上下文问题

这是大型 Agent 最容易遇到的问题之一。

假设执行：

```text
Iteration 1
Iteration 2
Iteration 3
...
Iteration 30
```

每一步都有：

```text
Thought
Action
Observation
```

上下文可能越来越大。

例如：

```text
User
 ↓
Thought 1
Action 1
Observation 1
 ↓
Thought 2
Action 2
Observation 2
 ↓
...
 ↓
Thought 30
```

最终：

```text
Context Window Overflow
```

所以生产系统必须进行：

```text
Context Management
```

---

# 22. Context Compression

可以把历史：

```text
Observation 1
Observation 2
Observation 3
Observation 4
...
```

压缩为：

```text
Summary:
已经查询了北京、上海、广州天气。
北京晴，上海有雨，广州多云。
```

于是：

```text
Raw History
      ↓
Summarizer
      ↓
Compressed Memory
```

这就是 Agent Memory / Context Management 的基础。

---

# 23. ReAct + Memory

现代 Agent 通常需要区分：

```text
Short-Term Memory
Long-Term Memory
```

### Short-Term Memory

当前任务：

```text
User
 ↓
Agent
 ↓
Tool
 ↓
Observation
```

### Long-Term Memory

长期信息：

```text
User Preference
User Profile
Past Conversations
Business Knowledge
```

架构：

```text
                 Agent
                   │
          ┌────────┴────────┐
          │                 │
 Short-Term Memory    Long-Term Memory
          │                 │
      Context          Vector DB
```

---

# 24. ReAct + RAG

ReAct 与 RAG 结合非常自然。

例如：

> 根据公司内部开发规范回答这个问题。

Agent：

```text
Reasoning
   ↓
需要查询公司知识库
   ↓
Action: retrieve()
   ↓
Observation: 文档片段
   ↓
Reasoning
   ↓
Final Answer
```

于是：

```text
ReAct
 +
RAG
```

可以形成：

```text
Reason
 ↓
Retrieve
 ↓
Observe
 ↓
Reason
 ↓
Answer
```

这比简单：

```text
User → RAG → Answer
```

更加灵活。

---

# 25. ReAct + MCP

MCP 可以理解为：

```text
Model Context Protocol
```

它解决的是：

> 如何用统一协议让 Agent 发现和调用外部工具、资源与能力。

因此：

```text
ReAct
    │
    ├── Tool
    ├── RAG
    ├── API
    └── MCP
```

Agent 的 Action 层可以变成：

```text
Action
 │
 ├── Local Tool
 ├── REST API
 ├── Database
 ├── RAG Retriever
 └── MCP Tool
```

这使 ReAct 从单纯的 Prompt 技巧进一步演化为：

```text
Agent Runtime Architecture
```

---

# 26. ReAct + MCP 的一个典型场景

例如企业 AI Agent：

```text
User
 │
 ▼
Enterprise Agent
 │
 ▼
ReAct Controller
 │
 ├──────────────┐
 │              │
 ▼              ▼
MCP Server     RAG
 │              │
 ├─ Jira        ├─ Wiki
 ├─ GitHub      ├─ Design Docs
 └─ Database    └─ SOP
```

用户：

> 查询一下支付服务最近是否有严重故障，并给出可能原因。

Agent 可以：

```text
1. 查询监控
2. 查询日志
3. 查询 Jira Incident
4. 查询最近 Git Commit
5. 查询架构文档
6. 综合分析
```

这已经是典型的企业级 Agent。

---

# 27. ReAct + Observability

Agent 比普通微服务更需要可观测性。

普通服务：

```text
Request
 ↓
Service
 ↓
Database
```

Agent：

```text
Request
 ↓
LLM
 ↓
Tool
 ↓
LLM
 ↓
Tool
 ↓
LLM
 ↓
Final
```

所以必须记录：

```text
Trace ID
Agent Run ID
Iteration
Model
Prompt Token
Completion Token
Tool Name
Tool Arguments
Tool Latency
Tool Result
Error
Final Answer
```

可以形成：

```text
Agent Run
│
├── LLM Call #1
│
├── Tool Call #1
│
├── LLM Call #2
│
├── Tool Call #2
│
└── Final Answer
```

这与分布式系统中的 Trace 非常类似。

---

# 28. ReAct 与 OpenTelemetry

如果你本身熟悉 OpenTelemetry，那么 Agent Observability 可以直接借鉴 Distributed Tracing。

例如：

```text
agent.request
   │
   ├── llm.chat
   │
   ├── tool.weather
   │
   ├── llm.chat
   │
   └── tool.search
```

Span Attributes：

```text
agent.name
agent.iteration
llm.model
llm.prompt_tokens
llm.completion_tokens
tool.name
tool.latency
tool.status
```

于是可以在 Grafana / Tempo 中看到：

```text
Agent Trace
```

这实际上是：

> **AI Agent Observability 与传统 Distributed Observability 的结合。**

---

# 29. ReAct 的核心架构

一个生产级 ReAct Agent 可以设计为：

```text
                    ┌──────────────┐
                    │     User     │
                    └──────┬───────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │ Agent Controller│
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │  Context Build  │
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │       LLM       │
                  └────────┬────────┘
                           │
                ┌──────────┴──────────┐
                │                     │
             Final                 Tool Call
                │                     │
                ▼                     ▼
             Answer              Tool Executor
                                      │
                                      ▼
                                 Observation
                                      │
                                      ▼
                                State Update
                                      │
                                      └──────────┐
                                                 │
                                                 ▼
                                                LLM
```

这就是 ReAct Agent 的核心循环。

---

# 30. 从状态机角度理解 ReAct

如果进一步抽象，ReAct 实际上非常像一个 State Machine。

状态：

```text
START
  ↓
REASONING
  ↓
ACTION
  ↓
OBSERVATION
  ↓
REASONING
  ↓
FINAL
```

可以定义：

```java
enum AgentState {
    START,
    REASONING,
    ACTION,
    OBSERVATION,
    FINAL,
    ERROR
}
```

状态转移：

```text
START
  ↓
REASONING
  ↓
ACTION
  ↓
OBSERVATION
  ├──→ REASONING
  ├──→ ERROR
  └──→ FINAL
```

这也是为什么现代 Agent Framework 越来越倾向于：

```text
Graph
+
State
+
Node
+
Edge
```

而不是简单的：

```text
while(true)
```

---

# 31. ReAct 为什么最终走向 Graph Agent？

简单 ReAct：

```text
LLM
 ↓
Tool
 ↓
LLM
 ↓
Tool
```

只能表达线性流程。

现实任务可能是：

```text
             ┌── Search ──┐
             │             │
User → Plan ─┼── RAG ──────┼→ Analyze
             │             │
             └── Database ┘
```

甚至：

```text
              ┌→ Tool A ─┐
Planner ──────┼→ Tool B ─┼→ Aggregator
              └→ Tool C ─┘
```

这就需要：

```text
Graph Agent
```

ReAct 可以作为 Graph 中的一个 Loop：

```text
Planner
   ↓
ReAct Node
   ↓
Tool
   ↓
Observation
   ↓
ReAct Node
```

---

# 32. ReAct 中的并行工具调用

例如用户：

> 比较北京、上海、广州今天的天气。

传统 ReAct：

```text
weather(Beijing)
      ↓
weather(Shanghai)
      ↓
weather(Guangzhou)
```

总耗时：

```text
T = T1 + T2 + T3
```

如果工具互相独立，可以并行：

```text
          ┌→ Beijing ───┐
Agent ────┼→ Shanghai ──┼→ Aggregate
          └→ Guangzhou ─┘
```

总耗时接近：

```text
T ≈ max(T1, T2, T3)
```

因此生产级 Agent 应该支持：

```text
Parallel Tool Calling
```

但需要注意：

> 并不是所有 Action 都可以并行。

例如：

```text
createOrder()
     ↓
payOrder()
```

存在依赖关系：

```text
payOrder
  depends on
createOrder
```

所以需要判断：

```text
Dependency Graph
```

---

# 33. ReAct 的安全问题

Agent 与普通 Chatbot 最大的区别之一是：

> Agent 可以执行动作。

如果工具包括：

```text
deleteDatabase()
sendEmail()
transferMoney()
deployProduction()
```

风险非常高。

所以不能简单：

```text
LLM → Tool
```

而应该：

```text
LLM
 ↓
Policy Engine
 ↓
Permission Check
 ↓
Human Approval
 ↓
Tool
```

例如：

```text
Read Operation
    ↓
自动执行

Write Operation
    ↓
Policy Check

High Risk Operation
    ↓
Human Approval
```

这就是：

```text
Human-in-the-Loop
```

---

# 34. ReAct 中的权限模型

可以把工具分成：

```text
READ
WRITE
DELETE
ADMIN
```

例如：

```text
searchJira()       READ
createJiraIssue()  WRITE
deleteJiraIssue()  DELETE
deployProduction() ADMIN
```

然后定义：

```java
enum RiskLevel {
    LOW,
    MEDIUM,
    HIGH,
    CRITICAL
}
```

Agent Action：

```text
Action
 ↓
Risk Assessment
 ↓
Authorization
 ↓
Execution
```

这比单纯依赖 Prompt 安全得多。

---

# 35. ReAct 的 Prompt 到底怎么设计？

经典 ReAct Prompt 会让模型遵循：

```text
Question
Thought
Action
Observation
Thought
Action
Observation
Final Answer
```

但现代 Function Calling Agent 不一定需要显式输出：

```text
Thought:
```

更常见的是：

```text
System Prompt
+
Tool Schema
+
Conversation
+
Tool Result
```

然后让模型通过 API 原生产生：

```text
tool_calls
```

因此：

> **现代 ReAct 的核心已经从“Prompt 格式”逐渐转移到了“Agent Runtime”。**

这是理解现代 Agent 的一个关键变化。

---

# 36. 传统 ReAct 与现代 Agent 的区别

传统：

```text
Prompt
 ↓
Thought
 ↓
Action
 ↓
Observation
```

现代：

```text
LLM
 ↓
Structured Tool Call
 ↓
Tool Runtime
 ↓
Observation
 ↓
State Machine
 ↓
LLM
```

区别在于：

```text
Reasoning
```

不再一定以文本形式暴露出来。

而是：

```text
Model Decision
```

由 Agent Runtime 管理。

---

# 37. 为什么不要把模型的 Thought 当成系统真相？

这是 Agent Engineering 非常重要的一点。

LLM 生成的：

```text
Thought:
我认为应该调用数据库……
```

本质上仍然是模型生成的文本。

不能把它当成：

```text
Trusted Execution Plan
```

真正可信的是：

```text
Structured Action
```

例如：

```json
{
  "tool": "queryOrder",
  "arguments": {
    "orderId": "12345"
  }
}
```

然后由：

```text
Schema Validation
+
Authorization
+
Tool Runtime
```

决定是否执行。

因此：

> **Reasoning 可以由模型产生，但 Execution 必须由系统控制。**

---

# 38. ReAct 的一个生产级 Java 实现

可以设计：

```java
public class ReActAgent {

    private final LlmClient llm;

    private final ToolRegistry toolRegistry;

    private final AgentStateManager stateManager;

    public String run(String userInput) {

        AgentState state =
            stateManager.create(userInput);

        for (int i = 0; i < 10; i++) {

            Decision decision =
                llm.decide(state);

            if (decision.isFinal()) {
                return decision.getAnswer();
            }

            Tool tool =
                toolRegistry.get(
                    decision.getToolName()
                );

            validate(decision);

            Observation observation =
                tool.execute(
                    decision.getArguments()
                );

            state.addObservation(observation);
        }

        throw new AgentExecutionException(
            "Maximum iterations exceeded"
        );
    }
}
```

这个结构已经可以演化成一个真正的 Agent Runtime。

---

# 39. Tool Registry

工具注册中心：

```java
public interface Tool {

    String name();

    String description();

    JsonSchema inputSchema();

    ToolResult execute(
        Map<String, Object> arguments
    );
}
```

例如：

```java
@Component
public class WeatherTool implements Tool {

    @Override
    public String name() {
        return "getWeather";
    }

    @Override
    public String description() {
        return "查询指定城市的实时天气";
    }

    @Override
    public ToolResult execute(
        Map<String, Object> arguments) {

        String city =
            (String) arguments.get("city");

        return weatherService.getWeather(city);
    }
}
```

然后：

```java
ToolRegistry
```

维护：

```text
getWeather
search
queryDatabase
getJiraIssue
getGithubCommit
...
```

---

# 40. ReAct 的核心数据结构

一个比较合理的模型：

```java
class AgentDecision {

    DecisionType type;

    String toolName;

    Map<String, Object> arguments;

    String finalAnswer;
}
```

其中：

```java
enum DecisionType {

    TOOL_CALL,

    FINAL_ANSWER
}
```

这样：

```text
LLM
 ↓
AgentDecision
 ↓
switch
```

而不是：

```text
解析自然语言：
"我现在应该调用天气工具……"
```

后者非常脆弱。

---

# 41. ReAct Runtime 的完整流程

生产环境可以设计成：

```text
                    User Request
                          │
                          ▼
                 ┌─────────────────┐
                 │ Context Manager │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │       LLM       │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ Decision Parser │
                 └────────┬────────┘
                          │
                ┌─────────┴─────────┐
                │                   │
             FINAL               TOOL_CALL
                │                   │
                ▼                   ▼
             Response        ┌─────────────┐
                             │ Policy      │
                             │ Validation  │
                             └──────┬──────┘
                                    │
                                    ▼
                             ┌─────────────┐
                             │Tool Runtime │
                             └──────┬──────┘
                                    │
                                    ▼
                              Observation
                                    │
                                    ▼
                              State Update
                                    │
                                    └───────→ LLM
```

---

# 42. ReAct 的性能问题

Agent 最大的问题之一：

> LLM 调用次数可能很多。

假设：

```text
一次 LLM = 1 秒
一次 Tool = 200 ms
```

如果：

```text
LLM → Tool → LLM → Tool → LLM
```

可能需要：

```text
3 × 1s + 2 × 0.2s
= 3.4s
```

如果 10 次迭代：

```text
10 × LLM
```

延迟会迅速增加。

所以 Agent 性能优化重点通常不是：

```text
Java 代码优化
```

而是：

```text
减少 LLM Calls
减少 Token
并行 Tool Calls
缓存
模型路由
```

---

# 43. LLM Model Routing

不同任务使用不同模型：

```text
简单任务
 ↓
Small Model

复杂推理
 ↓
Large Model
```

例如：

```text
Intent Classification
      ↓
Small Model

Tool Selection
      ↓
Small / Medium Model

Complex Reasoning
      ↓
Large Model
```

这样可以降低：

```text
Cost
Latency
```

---

# 44. ReAct 的成本模型

Agent 成本可以近似：

```text
Total Cost
=
Σ LLM Input Tokens
+
Σ LLM Output Tokens
+
Σ Tool Cost
```

如果：

```text
N = Agent Iterations
```

则：

```text
Cost ≈ Σ Cost(LLM_i)
```

因此：

> Agent Optimization 的第一原则通常是减少无意义的迭代。

---

# 45. Agent Evaluation

普通 LLM 可以评估：

```text
Answer Accuracy
```

Agent 更复杂，需要评估：

```text
Tool Selection Accuracy
Argument Accuracy
Task Success Rate
Number of Steps
Latency
Cost
Failure Recovery
Safety
```

例如：

```text
Task Success Rate = 92%

Tool Selection Accuracy = 96%

Average Steps = 4.2

Average Latency = 3.8s

Average Cost = $0.012
```

这才是 Agent 的工程指标。

---

# 46. ReAct 的错误类型

可以建立：

```text
Agent Failure Taxonomy
```

### Type 1：Reasoning Error

模型理解错任务。

### Type 2：Tool Selection Error

选择了错误工具。

### Type 3：Argument Error

工具参数错误。

### Type 4：Execution Error

工具本身失败。

### Type 5：Observation Interpretation Error

模型错误理解工具结果。

### Type 6：Termination Error

应该停止却继续调用。

### Type 7：Loop Error

重复执行相同 Action。

### Type 8：Safety Error

执行了不应该执行的操作。

这套分类非常适合做 Agent Evaluation。

---

# 47. ReAct 与传统微服务架构的一个有趣对应

如果你熟悉微服务，可以这样理解：

```text
传统微服务：

API Gateway
     ↓
Service
     ↓
Database
```

Agent：

```text
Agent Gateway
     ↓
LLM
     ↓
Tool
     ↓
External System
```

传统系统的控制流：

```text
Program → Function
```

Agent 系统的控制流：

```text
LLM → Tool
```

传统系统：

```text
代码决定下一步
```

Agent：

```text
模型决定下一步
```

因此：

> **Agent 的本质变化，是把部分控制流从确定性代码交给概率模型。**

这也是 Agent Engineering 最重要的架构变化之一。

---

# 48. Agent Engineering 的核心原则

因此，一个成熟的 ReAct 系统应该遵循：

```text
LLM负责：
- 理解
- 推理
- 规划
- 决策

代码负责：
- 校验
- 权限
- 执行
- 状态
- 重试
- 超时
- 熔断
- 监控
```

可以总结为：

> **让 LLM 决定“做什么”，让 Runtime 决定“能不能做、怎么做、什么时候停止”。**

---

# 49. ReAct 并不是万能的

ReAct 很适合：

```text
开放式任务
动态环境
工具调用
多步问题
需要反馈的任务
```

但是不一定适合：

```text
简单 FAQ
固定工作流
严格确定性流程
高频低延迟接口
```

例如：

```text
查询订单
```

如果业务流程非常固定：

```text
API
 ↓
Service
 ↓
Database
```

没有必要引入 Agent。

否则会增加：

```text
Latency
Cost
Complexity
Uncertainty
```

---

# 50. ReAct 最适合什么场景？

典型场景：

### 1. Coding Agent

```text
理解需求
 ↓
查看代码
 ↓
修改代码
 ↓
运行测试
 ↓
读取错误
 ↓
修改代码
 ↓
再次测试
```

这是天然的 ReAct。

---

### 2. Research Agent

```text
Search
 ↓
Read
 ↓
Analyze
 ↓
Search More
 ↓
Compare
 ↓
Write
```

---

### 3. Enterprise Agent

```text
查询 Jira
 ↓
查询 Git
 ↓
查询日志
 ↓
查询监控
 ↓
分析 Incident
```

---

### 4. Customer Service Agent

```text
理解用户
 ↓
查询订单
 ↓
查询物流
 ↓
查询退款状态
 ↓
执行操作
 ↓
回复用户
```

---

# 51. Coding Agent 为什么是 ReAct 的绝佳应用？

例如：

> 修复这个 Java 项目的 NullPointerException。

Agent：

```text
Reason
 ↓
读取异常日志
 ↓
Action: readFile()
 ↓
Observation
 ↓
Reason
 ↓
Action: searchCode()
 ↓
Observation
 ↓
Reason
 ↓
Action: editFile()
 ↓
Observation
 ↓
Action: runTest()
 ↓
Observation
 ↓
发现测试失败
 ↓
Reason
 ↓
修改代码
 ↓
runTest()
 ↓
成功
 ↓
Final
```

这几乎就是：

```text
ReAct
```

的完美体现。

---

# 52. ReAct 的终极抽象

如果把所有实现细节去掉：

```text
                ┌──────────────┐
                │     Goal     │
                └──────┬───────┘
                       │
                       ▼
                ┌──────────────┐
                │    Reason    │
                └──────┬───────┘
                       │
                       ▼
                ┌──────────────┐
                │     Act      │
                └──────┬───────┘
                       │
                       ▼
                ┌──────────────┐
                │   Observe    │
                └──────┬───────┘
                       │
                       ▼
                ┌──────────────┐
                │ Update State │
                └──────┬───────┘
                       │
                       └──────────→ Reason
```

最终：

```text
Goal
 ↓
Reason
 ↓
Act
 ↓
Observe
 ↓
Update State
 ↓
Reason
 ↓
...
 ↓
Goal Achieved
```

这就是 Agent 最核心的运行机制。

---

# 53. 从 ReAct 到 Agentic AI

可以把 AI Agent 的发展理解成几个阶段：

```text
Stage 1
LLM
 ↓
Answer
```

↓

```text
Stage 2
LLM + Prompt
 ↓
Structured Output
```

↓

```text
Stage 3
LLM + Tool Calling
 ↓
Tool
```

↓

```text
Stage 4
ReAct
 ↓
Reason
 ↓
Act
 ↓
Observe
```

↓

```text
Stage 5
Agent
 ↓
Memory
 ↓
RAG
 ↓
Tools
 ↓
Planning
 ↓
Reflection
```

↓

```text
Stage 6
Multi-Agent
 ↓
Planner
 ↓
Specialized Agents
 ↓
Tools
 ↓
Shared Memory
```

因此：

> ReAct 并不是 Agent 的终点，而是理解 Agent Architecture 最重要的起点之一。

---

# 54. 最值得记住的 10 个结论

如果面试官问：

> “你如何理解 ReAct？”

可以浓缩成下面十点：

**第一：**

```text
ReAct = Reasoning + Acting
```

**第二：**

它通过：

```text
Reason → Action → Observation
```

形成闭环。

**第三：**

它解决的不只是推理问题，而是：

```text
Reasoning + Environment Interaction
```

**第四：**

Function Calling 是工具调用机制，而 ReAct 是 Agent 决策范式。

**第五：**

现代 ReAct 不一定需要显式输出 Thought。

**第六：**

LLM 负责：

```text
Decision
```

Runtime 负责：

```text
Execution
```

**第七：**

生产级 Agent 必须解决：

```text
Timeout
Retry
Loop
Context
Permission
Observability
```

**第八：**

Agent State 是核心：

```text
User
+
Messages
+
Actions
+
Observations
+
Memory
```

**第九：**

ReAct 可以与：

```text
RAG
MCP
Memory
Planning
Graph
Human-in-the-loop
```

组合。

**第十：**

ReAct 的本质是：

> **让 LLM 成为动态控制器，而不是一次性的文本生成器。**

---

# 55. 最终架构总结

一个比较成熟的企业级 Agent 可以最终演化成：

```text
                         User
                           │
                           ▼
                  ┌─────────────────┐
                  │ Agent Gateway   │
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │ Context Manager │
                  └────────┬────────┘
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
        Short Memory                Long Memory
             │                           │
             └─────────────┬─────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │   ReAct Engine  │
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │       LLM       │
                  └────────┬────────┘
                           │
                    Decision
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
          Final                       Action
             │                           │
             ▼                           ▼
          Answer                 ┌────────────────┐
                                 │ Policy Engine  │
                                 └───────┬────────┘
                                         │
                                         ▼
                                 ┌────────────────┐
                                 │ Tool Runtime   │
                                 └───────┬────────┘
                                         │
                       ┌─────────────────┼─────────────────┐
                       │                 │                 │
                       ▼                 ▼                 ▼
                     RAG               MCP              API/DB
                       │                 │                 │
                       └─────────────────┼─────────────────┘
                                         │
                                         ▼
                                   Observation
                                         │
                                         ▼
                                   State Update
                                         │
                                         └──────────→ LLM
```

再加上：

```text
Observability
Security
Evaluation
Human-in-the-loop
```

就基本构成了一个生产级 Agent Runtime。

---

# 56. 结语

很多人第一次接触 ReAct 时，会认为：

> ReAct 就是让 ChatGPT 输出 Thought、Action、Observation。

这是对 ReAct 最表层的理解。

从 AI Agent 工程的角度看，更准确的理解应该是：

```text
ReAct
=
LLM Decision
+
Tool Execution
+
Environment Observation
+
State Transition
+
Iterative Control
```

而进一步看：

```text
ReAct
        ↓
Agent Loop
        ↓
State Machine
        ↓
Tool Runtime
        ↓
Memory / RAG / MCP
        ↓
Planning
        ↓
Observability
        ↓
Security
        ↓
Production Agent
```

所以真正值得掌握的不是某一个 ReAct Prompt，而是下面这套思维：

> **把 LLM 看成一个概率性的决策引擎，把 Tool Runtime 看成一个确定性的执行引擎，再通过 State + Observation 把两者连接起来。**

这也是从：

```text
AI Application Developer
```

走向：

```text
AI Agent Engineer
```

最重要的一步。

---

## 一句话总结

> **ReAct 的本质不是“让 AI 思考”，而是让 AI 在真实环境中形成“思考 → 行动 → 观察 → 再思考”的闭环，并通过确定性的 Agent Runtime 将概率性的 LLM 决策转化为可控、可观测、可恢复的系统行为。**
