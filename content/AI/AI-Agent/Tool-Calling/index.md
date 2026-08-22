---
title: Tool Calling：从 LLM Function Calling 到 Agent 工具执行的核心技术
# tags:
#   - nodejs
date: '2026-08-08'
summary: Tool Calling 让 LLM 从“生成信息”进入“调用能力”的世界，而 Agent Runtime 则负责把这种能力变成安全、可靠、可观测、可控制的真实软件执行.
---

# Tool Calling：从 LLM Function Calling 到 Agent 工具执行的核心技术

> **摘要**
>
> Tool Calling 是现代 AI Agent 最基础、也是最关键的能力之一。它解决的核心问题是：**如何让一个只能生成文本的 LLM，可靠地调用外部软件能力。**
>
> 一个 LLM 可以告诉你“应该查询订单”，但它本身无法直接访问订单数据库；通过 Tool Calling，模型可以生成结构化的工具调用请求，由 Agent Runtime 验证参数、执行工具，再把结果返回给模型。由此形成：
>
> **LLM → Tool Call → Tool Execution → Tool Result → LLM**
>
> 这条链路是 Agent 从“会回答”走向“会行动”的基础。本文从协议模型、架构、Function Calling、Tool Schema、执行生命周期、错误处理、并发调用、安全、MCP、Java/Spring AI 实现以及 Production Engineering 等方面，系统介绍 Tool Calling 的核心原理。

---

# 1. 为什么需要 Tool Calling？

大语言模型最擅长的是：

* 理解自然语言
* 生成文本
* 总结
* 推理
* 生成代码
* 结构化信息提取

但 LLM 本身并不能天然完成：

```text
查询数据库
调用内部 API
访问 Redis
查询 Kubernetes
读取生产日志
发送邮件
创建 Jira
执行代码
搜索互联网
```

例如用户说：

> “查询订单 12345 的状态。”

普通 LLM 只能：

```text
User
 ↓
LLM
 ↓
“我无法直接访问你的订单系统。”
```

如果加入 Tool：

```text
User
 ↓
LLM
 ↓
Tool Call
 ↓
getOrder(12345)
 ↓
Order Service
 ↓
Order Result
 ↓
LLM
 ↓
最终回答
```

系统就发生了本质变化。

因此可以把 Tool Calling 理解为：

> **LLM 与外部软件世界之间的标准化桥梁。**

---

# 2. Tool Calling 到底是什么？

Tool Calling 的核心思想非常简单：

> **模型不直接执行工具，而是生成一个结构化的“调用意图”，由应用程序负责真正执行。**

例如：

```json
{
  "name": "get_order",
  "arguments": {
    "orderId": "12345"
  }
}
```

注意：

**LLM 并没有真正执行 `get_order()`。**

实际流程是：

```text
                 ┌────────────┐
                 │    User    │
                 └─────┬──────┘
                       ↓
                 ┌────────────┐
                 │    LLM     │
                 └─────┬──────┘
                       │
                 Tool Call
                       │
                       ↓
               ┌───────────────┐
               │ Agent Runtime │
               └───────┬───────┘
                       ↓
                  Tool Executor
                       ↓
                 Order Service
```

因此：

> **LLM 决定调用什么，Runtime 决定是否调用以及如何调用。**

这是 Tool Calling 最重要的架构原则。

---

# 3. Function Calling 与 Tool Calling

在实际技术资料中经常看到两个词：

```text
Function Calling
Tool Calling
```

它们概念高度相关，但可以从抽象层次理解。

Function Calling 更强调：

> “模型需要调用某个函数。”

例如：

```json
{
  "function": "get_weather",
  "arguments": {
    "city": "Guangzhou"
  }
}
```

Tool Calling 的概念更加广泛。

Tool 可以是：

```text
Function
API
Database
Search Engine
File System
Code Executor
MCP Server
Kubernetes
Browser
```

因此现代 Agent 系统通常更倾向使用：

> **Tool Calling**

因为它描述的是“能力”，而不仅仅是某一个函数。

---

# 4. Tool Calling 的完整生命周期

一个完整 Tool Calling 通常经历：

```text
1. Tool Definition
       ↓
2. Tool Registration
       ↓
3. Prompt / Tool Schema
       ↓
4. User Request
       ↓
5. LLM Decision
       ↓
6. Tool Call Generation
       ↓
7. Tool Validation
       ↓
8. Tool Execution
       ↓
9. Tool Result
       ↓
10. Result → LLM
       ↓
11. Final Answer
```

完整架构：

```text
                         User
                          │
                          ↓
                    ┌───────────┐
                    │    LLM    │
                    └─────┬─────┘
                          │
                     Tool Call
                          │
                          ↓
                 ┌─────────────────┐
                 │  Agent Runtime  │
                 └────────┬────────┘
                          │
                ┌─────────┴─────────┐
                ↓                   ↓
          Validation            Permission
                │                   │
                └─────────┬─────────┘
                          ↓
                    Tool Executor
                          │
             ┌────────────┼────────────┐
             ↓            ↓            ↓
           API           DB          Search
             │            │            │
             └────────────┼────────────┘
                          ↓
                     Tool Result
                          │
                          ↓
                         LLM
                          │
                          ↓
                    Final Answer
```

---

# 5. Tool Schema：模型如何知道 Tool？

这是 Tool Calling 的第一个核心问题：

> **LLM 怎么知道有哪些 Tool？**

答案是：

> 给模型提供 Tool Schema。

例如：

```json
{
  "name": "get_order",
  "description": "Get order information by order ID.",
  "parameters": {
    "type": "object",
    "properties": {
      "orderId": {
        "type": "string",
        "description": "The unique order ID."
      }
    },
    "required": ["orderId"]
  }
}
```

模型看到这个 Schema 后，就知道：

```text
Tool:
get_order

用途：
查询订单

参数：
orderId
```

然后当用户说：

> “查询订单 12345。”

模型可能产生：

```json
{
  "name": "get_order",
  "arguments": {
    "orderId": "12345"
  }
}
```

---

# 6. Tool Description 为什么非常重要？

Tool Calling 的一个关键事实是：

> **模型并不是直接理解你的 Java 方法，而是通过 Tool Schema 理解 Tool。**

例如：

```java
public Order getOrder(String id)
```

对于模型来说，真正重要的是：

```text
Name
Description
Parameters
Parameter Description
Return Schema
```

例如：

```json
{
  "name": "get_order",
  "description": "Retrieve the current order status and shipment information.",
  "parameters": {
    "type": "object",
    "properties": {
      "orderId": {
        "type": "string",
        "description": "Unique identifier of the order."
      }
    },
    "required": ["orderId"]
  }
}
```

好的 Description 能告诉模型：

```text
什么时候应该调用？
什么时候不应该调用？
需要什么参数？
返回什么？
```

因此：

> **Tool Description 本质上是给 LLM 看的 API Documentation。**

---

# 7. Tool Schema 不只是接口文档

传统 API：

```text
OpenAPI
 ↓
Developer
 ↓
Understand API
```

Tool Schema：

```text
Tool Schema
 ↓
LLM
 ↓
Decide whether to call
```

这意味着 Tool Schema 同时承担：

```text
API Contract
+
LLM Guidance
```

因此设计 Tool 时不能只考虑：

> “程序员能不能看懂？”

还必须考虑：

> “模型能不能正确选择和调用？”

---

# 8. 一个好的 Tool 应该是什么样？

假设有一个订单系统。

错误设计：

```text
execute_order_operation
```

参数：

```json
{
  "operation": "string",
  "data": "string"
}
```

这对 LLM 非常不友好。

更好的设计：

```text
get_order
cancel_order
get_order_items
get_shipping_status
```

例如：

```json
{
  "name": "get_shipping_status",
  "description": "Get the current shipment status for an order.",
  "parameters": {
    "type": "object",
    "properties": {
      "orderId": {
        "type": "string"
      }
    },
    "required": ["orderId"]
  }
}
```

原则是：

> **一个 Tool 最好具有清晰、单一、可描述的职责。**

---

# 9. Tool Calling 不等于 Tool Execution

这是很多初学者容易忽略的问题。

假设 LLM 输出：

```json
{
  "name": "delete_user",
  "arguments": {
    "userId": "10001"
  }
}
```

这并不意味着：

```text
User Deleted
```

真正发生的是：

```text
LLM
 ↓
Tool Call
 ↓
Runtime
 ↓
Permission Check
 ↓
Parameter Validation
 ↓
Tool Execution
 ↓
Result
```

因此：

> **Tool Call 是意图，Tool Execution 才是动作。**

这一区别对于安全设计非常重要。

---

# 10. Tool Registry

当 Agent 有几十甚至几百个 Tool 时，需要一个 Tool Registry。

例如：

```java
public interface Tool {

    String getName();

    String getDescription();

    ToolResult execute(
        Map<String, Object> arguments
    );
}
```

然后：

```java
@Component
public class ToolRegistry {

    private final Map<String, Tool> tools =
            new ConcurrentHashMap<>();

    public void register(Tool tool) {
        tools.put(tool.getName(), tool);
    }

    public Tool get(String name) {
        return tools.get(name);
    }
}
```

系统启动：

```text
ToolRegistry
│
├── get_order
├── get_customer
├── get_inventory
├── search_logs
├── query_metrics
├── search_documents
└── create_ticket
```

Agent Runtime：

```text
LLM Tool Call
      ↓
ToolRegistry
      ↓
Find Tool
      ↓
Execute
```

---

# 11. Tool Executor

Tool Registry 负责：

> 找到 Tool。

Tool Executor 负责：

> 执行 Tool。

可以设计：

```java
public class ToolExecutor {

    private final ToolRegistry registry;

    public ToolResult execute(
            ToolCall toolCall) {

        Tool tool =
            registry.get(toolCall.name());

        if (tool == null) {
            return ToolResult.failure(
                "Unknown tool"
            );
        }

        return tool.execute(
            toolCall.arguments()
        );
    }
}
```

实际生产环境中，还应该加入：

```text
Authentication
Authorization
Validation
Timeout
Retry
Circuit Breaker
Rate Limit
Audit
Tracing
```

---

# 12. Tool Calling 的核心闭环

一个 Agent 最基本的 Tool Loop 是：

```text
while (true) {

    response = llm.chat(messages, tools);

    if (response.hasFinalAnswer()) {
        return response.answer();
    }

    for (ToolCall call : response.toolCalls()) {

        result = toolExecutor.execute(call);

        messages.add(
            ToolResultMessage(result)
        );
    }
}
```

这段逻辑虽然简单，却是整个 Agent Runtime 的核心。

可以进一步抽象成：

```text
LLM
 ↓
Decision
 ↓
Tool Call
 ↓
Execution
 ↓
Observation
 ↓
LLM
```

---

# 13. Single Tool Call

最简单的情况：

```text
User:
查询订单 12345。

        ↓

LLM:
get_order(12345)

        ↓

Tool:
{
  "status": "PAID",
  "shipping": "SHIPPED"
}

        ↓

LLM:
订单 12345 已支付并已发货。
```

整个过程只有：

```text
LLM → Tool → LLM
```

---

# 14. Sequential Tool Calling

复杂问题往往需要多个 Tool。

例如：

> “分析订单 12345 为什么没有发货。”

Agent：

```text
get_order(12345)
        ↓
Order Result
        ↓
get_inventory(productId)
        ↓
Inventory Result
        ↓
get_warehouse_status(productId)
        ↓
Warehouse Result
        ↓
LLM Analysis
```

形成：

```text
LLM
 ↓
Tool A
 ↓
Result A
 ↓
LLM
 ↓
Tool B
 ↓
Result B
 ↓
LLM
 ↓
Tool C
 ↓
Result C
 ↓
Final Answer
```

这种模式非常接近经典 ReAct Agent。

---

# 15. Parallel Tool Calling

如果两个 Tool 之间没有依赖关系，就不应该：

```text
Tool A
 ↓
Tool B
```

而可以：

```text
       ┌── Tool A
LLM ───┤
       └── Tool B
```

例如：

> “分析今天 API 性能情况。”

可以同时查询：

```text
Prometheus
Log Search
Trace Search
```

流程：

```text
              ┌── Prometheus
              │
LLM → Runtime ├── Log Search
              │
              └── Trace Search
```

然后：

```text
Tool A Result
       │
Tool B Result
       ├──→ LLM
Tool C Result
```

这样可以显著降低：

```text
Latency
```

---

# 16. Tool Dependency Graph

复杂 Agent 可以把 Tool 调用关系看成一个 DAG：

```text
                 get_order
                     │
              ┌──────┴──────┐
              ↓             ↓
        get_inventory   get_payment
              │
              ↓
       get_warehouse
              │
              ↓
          final analysis
```

这里：

```text
get_inventory
get_payment
```

可以并行。

而：

```text
get_warehouse
```

依赖：

```text
get_inventory
```

所以不能提前执行。

因此 Production Agent 的 Tool Executor 实际上可能需要：

> **Dependency-Aware Execution。**

---

# 17. Tool Result 设计

Tool Result 是另一个非常关键的部分。

不推荐：

```text
"查询失败"
```

推荐：

```json
{
  "success": false,
  "errorCode": "ORDER_NOT_FOUND",
  "message": "Order does not exist."
}
```

成功：

```json
{
  "success": true,
  "data": {
    "orderId": "12345",
    "status": "PAID"
  }
}
```

这样 LLM 可以判断：

```text
success = true
```

或者：

```text
success = false
errorCode = ORDER_NOT_FOUND
```

然后决定下一步。

---

# 18. Tool Error Handling

Tool 不可能永远成功。

例如：

```text
Timeout
Connection Refused
404
500
Authentication Failed
Rate Limited
Invalid Parameter
Business Error
```

因此 Tool Result 应该明确区分：

```text
Business Failure
System Failure
```

例如：

```json
{
  "success": false,
  "type": "BUSINESS_ERROR",
  "code": "INSUFFICIENT_INVENTORY"
}
```

或者：

```json
{
  "success": false,
  "type": "SYSTEM_ERROR",
  "code": "TIMEOUT"
}
```

这样 Agent 才能进行不同策略：

```text
Business Error
 → Explain to user

Timeout
 → Retry

Rate Limit
 → Backoff

Permission Error
 → Ask for authorization
```

---

# 19. Retry 不是简单地重试

假设：

```text
Tool
 ↓
Timeout
```

不能无限：

```text
retry()
retry()
retry()
retry()
```

应该：

```text
Attempt 1
 ↓
Timeout
 ↓
Backoff
 ↓
Attempt 2
 ↓
Timeout
 ↓
Attempt 3
 ↓
Stop
```

典型策略：

```text
Exponential Backoff
+
Jitter
+
Maximum Retry
```

例如：

```text
100ms
 ↓
300ms
 ↓
900ms
```

但对于：

```text
DELETE
TRANSFER
SEND_PAYMENT
```

必须特别考虑：

> **Tool 是否幂等？**

---

# 20. Tool Idempotency

例如：

```text
create_payment()
```

如果 Agent 因为 Timeout 重试：

```text
create_payment()
create_payment()
```

可能产生：

```text
Double Payment
```

所以对于有副作用的 Tool：

```text
create
update
delete
transfer
send
```

必须设计：

```text
Idempotency Key
```

例如：

```json
{
  "paymentId": "PAY-12345",
  "idempotencyKey": "agent-task-abc-001"
}
```

这是 Agent 与传统分布式系统结合时非常重要的一点。

---

# 21. Tool Permission

不是所有 Agent 都应该看到所有 Tool。

例如：

```text
Customer Support Agent
```

允许：

```text
get_order
get_customer
get_shipping
```

不允许：

```text
delete_customer
refund_payment
execute_shell
```

可以设计：

```text
Agent
 ↓
Permission Policy
 ↓
Allowed Tools
```

例如：

```java
if (!permissionService.allowed(
        agentId,
        tool.name())) {

    throw new AccessDeniedException();
}
```

因此：

> **Tool Registry 解决“有什么能力”，Permission 解决“谁可以使用能力”。**

---

# 22. Tool Sandboxing

对于危险 Tool：

```text
Shell
Python
Docker
Kubernetes
SQL
Browser
```

不能直接给：

```text
Production Environment
```

应该增加 Sandbox：

```text
Agent
 ↓
Tool
 ↓
Sandbox
 ↓
Execution
```

例如 Coding Agent：

```text
LLM
 ↓
Generate Code
 ↓
Sandbox
 ↓
Compile
 ↓
Run Tests
 ↓
Result
```

而不是：

```text
LLM
 ↓
Production Server
 ↓
Execute Shell
```

---

# 23. SQL Tool 是一个典型案例

很多企业都会做：

> Text-to-SQL Agent。

用户：

> “查询今年每个月的销售额。”

Agent：

```text
LLM
 ↓
Generate SQL
 ↓
SQL Validator
 ↓
Read-Only DB
 ↓
Execute
 ↓
Result
 ↓
LLM
 ↓
Answer
```

这里千万不能简单：

```java
jdbc.execute(
    llmGeneratedSql
);
```

至少应该：

```text
SQL Parse
 ↓
Read / Write Check
 ↓
Table Permission
 ↓
Row Limit
 ↓
Timeout
 ↓
Read-Only Connection
 ↓
Execute
```

这说明：

> **Tool Calling 本质上是 AI 与真实系统连接的安全边界。**

---

# 24. Tool Calling 与 Prompt 的关系

Tool Calling 不是简单：

```text
Prompt:
你可以调用 get_order。
```

而应该由 Runtime 提供结构化 Tool Definition：

```text
LLM
├── System Instructions
├── Conversation
├── Tool Definitions
└── Tool Results
```

模型根据：

```text
用户意图
+
当前上下文
+
Tool Schema
+
Tool Result
```

决定：

```text
继续调用
```

还是：

```text
返回最终答案
```

因此：

> **Prompt 定义 Agent 的行为，Tool Schema 定义 Agent 的能力。**

---

# 25. Tool Calling 与 Structured Output

Tool Calling 和 Structured Output 很容易混淆。

Structured Output：

```text
LLM
 ↓
JSON
```

例如：

```json
{
  "sentiment": "positive",
  "score": 0.92
}
```

它的目的是：

> **让模型输出符合 Schema 的数据。**

Tool Calling：

```text
LLM
 ↓
Tool Call
 ↓
External System
```

它的目的是：

> **让模型产生一个可执行的外部动作请求。**

因此：

```text
Structured Output
= Structured Data

Tool Calling
= Structured Action
```

---

# 26. Tool Calling 与 Agent 的关系

可以把两者关系理解为：

```text
LLM
 │
 ├── Reasoning
 │
 └── Tool Calling
          │
          ↓
       Tool
          │
          ↓
       Result
          │
          ↓
        LLM
```

Agent 则在这个基础上增加：

```text
Memory
Planning
State
Reflection
Guardrails
Evaluation
```

所以：

> **Tool Calling 是 Agent 的执行能力之一。**

没有 Tool Calling：

```text
Agent
 ↓
只能思考
```

有 Tool Calling：

```text
Agent
 ↓
可以行动
```

---

# 27. MCP 与 Tool Calling

随着 Agent 生态发展，一个新的问题出现：

> Tool 太多了，而且不同系统定义方式不同。

MCP（Model Context Protocol）解决的核心问题之一就是：

> **以标准化方式向 AI 应用暴露 Tools、Resources 和其他上下文能力。**

可以理解为：

```text
                 AI Agent
                    │
                   MCP
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
      Git           DB          API
```

传统方式：

```text
Agent
 ↓
Custom Integration
 ↓
Git API
```

MCP：

```text
Agent
 ↓
MCP Client
 ↓
MCP Server
 ↓
Git
```

因此：

```text
Tool Calling
```

解决的是：

> 模型如何提出工具调用。

而：

```text
MCP
```

更关注：

> AI 应用如何标准化发现和使用外部能力。

二者不是简单的竞争关系，而可以结合。

---

# 28. Tool Calling 与 MCP 的关系

可以用这一层次理解：

```text
LLM
 │
 │ Tool Call
 ↓
Agent Runtime
 │
 │ MCP
 ↓
MCP Server
 │
 ↓
External System
```

例如：

```text
LLM:
call search_logs(...)
```

Runtime 可以通过：

```text
MCP Client
```

找到：

```text
Log MCP Server
```

最终：

```text
Log System
```

这样 Agent 不需要为每个系统实现一套完全定制的连接方式。

---

# 29. Java 中设计 Tool

对于 Spring Boot 项目，可以设计：

```java
public interface AgentTool {

    String name();

    String description();

    ToolSchema schema();

    ToolResult execute(
        ToolContext context,
        Map<String, Object> arguments
    );
}
```

例如订单 Tool：

```java
@Component
public class GetOrderTool
        implements AgentTool {

    @Override
    public String name() {
        return "get_order";
    }

    @Override
    public String description() {
        return """
            Get order information including
            payment and shipment status.
            """;
    }

    @Override
    public ToolResult execute(
            ToolContext context,
            Map<String, Object> arguments) {

        String orderId =
            (String) arguments.get("orderId");

        Order order =
            orderService.findById(orderId);

        return ToolResult.success(order);
    }
}
```

---

# 30. Spring AI 中的 Tool 思想

如果使用 Spring AI，核心思想仍然是：

```text
Java Method
 ↓
Tool Definition
 ↓
LLM
 ↓
Tool Call
 ↓
Java Method
 ↓
Tool Result
```

例如概念上的代码：

```java
@Tool(
    description = "Get order information by order ID"
)
public Order getOrder(String orderId) {

    return orderService.findById(orderId);
}
```

然后：

```text
LLM
 ↓
发现 getOrder Tool
 ↓
生成 Tool Call
 ↓
Spring AI
 ↓
执行 Java Method
 ↓
返回 Tool Result
```

这使传统 Java Service 很容易成为 Agent 的能力。

---

# 31. Tool Context

Production Tool 通常不能只传：

```java
arguments
```

还需要：

```text
User
Tenant
Agent
Trace ID
Security Context
Request ID
Locale
Deadline
```

可以设计：

```java
public class ToolContext {

    private String requestId;

    private String traceId;

    private String userId;

    private String tenantId;

    private String agentId;

    private Instant deadline;
}
```

于是：

```text
Tool Call
+
Tool Arguments
+
Execution Context
```

才构成一次完整 Tool Execution。

---

# 32. Tool Timeout

每一个 Tool 都应该有明确的 Timeout。

例如：

```text
Search Tool       3s
Database Tool     5s
HTTP API          5s
Kubernetes        10s
Code Execution    30s
```

不能允许：

```text
Agent
 ↓
Tool
 ↓
hang forever
```

否则：

```text
Tool Timeout
 ↓
Agent Timeout
 ↓
Request Timeout
 ↓
Thread / Connection Resource Leak
```

因此：

> **Tool Timeout 是 Agent Runtime 的基本能力，而不是业务 Tool 自己随意决定。**

---

# 33. Tool Observability

Tool Calling 必须可观测。

建议每次 Tool Call 都记录：

```text
traceId
agentId
toolName
arguments
startTime
duration
status
error
retryCount
```

例如：

```text
Trace: abc123

Agent Task
│
├── LLM Call
│
├── Tool: get_order
│   ├── Duration: 82ms
│   ├── Status: SUCCESS
│   └── OrderId: 12345
│
├── Tool: get_inventory
│   ├── Duration: 120ms
│   └── Status: SUCCESS
│
└── LLM Call
```

这与传统 Microservices Observability 非常类似。

如果使用 OpenTelemetry，可以建立：

```text
Agent Span
   │
   ├── LLM Span
   │
   ├── Tool Span
   │
   ├── DB Span
   │
   └── HTTP Span
```

最终形成完整的：

> **AI → Tool → Microservice → Database Trace**

---

# 34. Tool Calling 的安全边界

Tool 是 Agent 最危险的地方。

因为 Tool 让：

```text
LLM Decision
```

变成：

```text
Real-world Action
```

因此应该建立：

```text
                    LLM
                     │
                Tool Call
                     ↓
             ┌───────────────┐
             │ Policy Engine │
             └───────┬───────┘
                     ↓
              Permission Check
                     ↓
              Input Validation
                     ↓
                 Sandbox
                     ↓
              Tool Execution
```

这比：

```text
LLM → Java Method
```

安全得多。

---

# 35. Read Tool 与 Write Tool

生产系统中非常推荐把 Tool 分成：

```text
Read Tool
Write Tool
```

例如：

### Read

```text
get_order
get_customer
search_logs
query_metrics
```

### Write

```text
cancel_order
refund_payment
create_ticket
deploy_service
```

Read Tool：

```text
风险较低
```

Write Tool：

```text
风险较高
```

因此可以设计不同权限：

```text
Read Tool
 → Auto Execute

Write Tool
 → Policy Check

High Risk Tool
 → Human Approval
```

---

# 36. Human Approval Tool

例如：

```text
refund_payment
```

Agent：

```text
LLM
 ↓
refund_payment(1000)
 ↓
Risk Engine
 ↓
HIGH RISK
 ↓
Human Approval
```

用户批准：

```text
Approve
```

之后：

```text
Tool Execution
```

这是一种非常重要的 Agent Architecture：

> **AI 决策，人类控制高风险动作。**

---

# 37. Tool Calling 的常见错误

## 错误一：把 Tool 当普通 API

传统 API 只考虑：

```text
Developer → API
```

Tool 需要考虑：

```text
LLM → Tool
```

因此必须考虑模型：

```text
能不能理解？
会不会选错？
参数会不会填错？
```

---

## 错误二：Tool 太大

例如：

```text
execute_business_operation()
```

一个 Tool 包含几十种操作。

这会让模型非常难选择。

---

## 错误三：Tool 太多

如果给模型：

```text
500 Tools
```

模型的 Tool Selection 可能变得困难，而且：

```text
Prompt Size
Token Cost
Latency
```

都会增加。

因此需要：

> **Dynamic Tool Discovery / Tool Routing**

---

# 38. Tool Routing

当系统有大量 Tool 时，可以先增加一个 Router：

```text
User
 ↓
Tool Router
 ↓
Relevant Tools
 ↓
LLM
 ↓
Tool Call
```

例如：

```text
用户问订单
 ↓
Order Tools

用户问监控
 ↓
Observability Tools

用户问代码
 ↓
Coding Tools
```

这样模型每次只看到相关 Tool。

这可以减少：

```text
Context
Token
Tool Selection Error
```

---

# 39. Tool Calling 的性能优化

主要关注：

```text
LLM Latency
Tool Latency
Network Latency
Serialization
Context Size
```

优化策略包括：

### 并行 Tool Calling

```text
Tool A ─┐
Tool B ─┼→ Parallel
Tool C ─┘
```

### Tool Result 压缩

不要把：

```text
10000 行日志
```

全部返回 LLM。

应该：

```text
Log Search
 ↓
Filter
 ↓
Aggregate
 ↓
Relevant Result
 ↓
LLM
```

### Tool Result Pagination

对于大量数据：

```text
page=1
page=2
page=3
```

而不是一次返回所有数据。

---

# 40. Tool Result 不应该等于数据库 Result

这是一个非常重要的工程原则。

错误：

```text
SELECT *
FROM orders;
```

直接把几十万条数据交给 LLM。

正确：

```text
Database
 ↓
Tool
 ↓
Aggregation
 ↓
Filtering
 ↓
Business Result
 ↓
LLM
```

例如：

```json
{
  "totalOrders": 12030,
  "totalRevenue": 5230000,
  "growthRate": 0.18,
  "topProducts": [
    "Product A",
    "Product B"
  ]
}
```

这样 Tool 才是真正的：

> **AI-oriented API。**

---

# 41. Tool 应该是“AI-Native API”

传统 API：

```text
GET /orders/{id}
```

面向：

```text
Application
```

Agent Tool：

```text
get_order
```

面向：

```text
LLM
```

AI-Native Tool 应该具有：

```text
Clear Intent
Clear Schema
Small Scope
Structured Result
Predictable Error
Explicit Permission
```

因此：

> **Agent Tool 不是简单给 REST API 套一层 Function Calling。**

它需要针对 AI 的行为重新设计。

---

# 42. 一个完整的 Tool Calling 示例

用户：

> “帮我检查订单 12345 为什么还没有发货。”

Agent：

```text
Step 1
LLM
 ↓
get_order(12345)
```

结果：

```json
{
  "status": "PAID",
  "shippingStatus": "WAITING",
  "productId": "P001"
}
```

Agent：

```text
Step 2
LLM
 ↓
get_inventory(P001)
```

结果：

```json
{
  "available": 0,
  "reserved": 0
}
```

Agent：

```text
Step 3
LLM
 ↓
get_warehouse_status(P001)
```

结果：

```json
{
  "status": "TRANSFER"
}
```

最终：

```text
订单 12345 已支付，但当前库存为 0。
商品 P001 正处于仓库调拨状态，因此订单暂未发货。
```

注意：

最终答案不是预先写死的。

Agent 是：

```text
根据 Tool Result
        ↓
动态形成结论
```

---

# 43. Tool Calling 的本质

从更高层次看：

```text
Traditional Software

Code
 ↓
API
 ↓
System
```

而 Agent：

```text
Natural Language
 ↓
LLM
 ↓
Tool Call
 ↓
API
 ↓
System
```

于是出现了一个非常重要的软件工程变化：

> **自然语言开始成为软件系统的高层控制接口。**

以前：

```text
Developer
 ↓
Code
 ↓
API
```

现在：

```text
User
 ↓
Natural Language
 ↓
LLM
 ↓
Tool
 ↓
API
```

这正是 Tool Calling 的真正价值。

---

# 44. Tool Calling 的工程本质

Tool Calling 看起来像 AI 技术，但深入之后会发现，它实际上是：

```text
AI
+
API Design
+
Distributed Systems
+
Security
+
Runtime
+
Observability
```

一个 Production Tool 至少应该考虑：

```text
Schema
Validation
Permission
Timeout
Retry
Idempotency
Rate Limit
Circuit Breaker
Audit
Tracing
Result Normalization
```

因此：

> **Tool Calling 是 LLM 与传统软件工程之间的连接层。**

---

# 45. 总结

如果把整个 Tool Calling 压缩成一张架构图：

```text
                         User
                          │
                          ↓
                    ┌───────────┐
                    │    LLM    │
                    └─────┬─────┘
                          │
                     Tool Decision
                          │
                          ↓
                  ┌───────────────┐
                  │ Agent Runtime │
                  └───────┬───────┘
                          │
             ┌────────────┼────────────┐
             ↓            ↓            ↓
        Validation    Permission    Policy
             │            │            │
             └────────────┼────────────┘
                          ↓
                    Tool Executor
                          │
             ┌────────────┼────────────┐
             ↓            ↓            ↓
            API           DB          MCP
             │            │            │
             └────────────┼────────────┘
                          ↓
                     Tool Result
                          │
                          ↓
                         LLM
                          │
                          ↓
                    Final Answer
```

可以用一句话概括：

> **Tool Calling 让 LLM 从“生成信息”进入“调用能力”的世界，而 Agent Runtime 则负责把这种能力变成安全、可靠、可观测、可控制的真实软件执行。**

因此，一个成熟的 Tool Calling 系统并不是：

```text
LLM + Function
```

而应该是：

```text
Tool Calling
=
Tool Schema
+
Tool Selection
+
Tool Validation
+
Permission
+
Execution
+
Error Handling
+
Retry
+
Idempotency
+
Observability
+
Security
+
Human Approval
```

最终形成：

```text
                 Intelligence
                      │
                     LLM
                      │
                Tool Calling
                      │
                      ↓
               Agent Runtime
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
        API          DB          MCP
          │           │           │
          └───────────┼───────────┘
                      ↓
                 Real World
```

**这就是 Tool Calling 在 Agent Architecture 中最核心的位置：它是连接“AI 推理”和“真实世界执行”的桥梁。**
