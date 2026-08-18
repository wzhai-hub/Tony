---
title: Function Calling 深度解析
# tags:
#   - nodejs
date: '2026-08-18'
summary: 让模型能够连接到应用程序提供的外部工具和系统，从而获取数据、执行操作或参与更复杂的工作流让模型能够连接到应用程序提供的外部工具和系统，从而获取数据、执行操作或参与更复杂的工作流
---

## 引言：LLM 为什么需要 Function Calling？

Large Language Model 最初最擅长的是一件事情：

> **根据上下文生成文本。**

例如：

```text
User:
查询一下订单 ORD-10086 的状态。

LLM:
订单 ORD-10086 当前状态是已发货。
```

问题在于，LLM 本身并不知道：

```text
ORD-10086
```

在企业数据库中的真实状态。

它需要访问：

```text
Order Service
Database
Redis
Kafka
CRM
Payment System
```

这就产生了一个核心问题：

> **如何让一个只能生成文本的模型，安全、可靠地与真实的软件系统交互？**

Function Calling 正是解决这个问题的关键机制。

OpenAI 对 Function Calling 的定义是：让模型能够连接到应用程序提供的外部工具和系统，从而获取数据、执行操作或参与更复杂的工作流。现代 API 中通常称为 **Tool Calling**，Function 是 Tool 的一种具体形式。

因此，可以先建立一个非常重要的认知：

```text
Function Calling ≠ LLM 执行函数

Function Calling =
LLM 决定“需要调用什么”
        +
LLM 生成“调用参数”
        +
Application 执行真正的函数
```

这三个步骤之间的边界，是理解 Function Calling 的核心。

---

# 一、Function Calling 到底是什么？

假设我们有一个传统 Java 服务：

```java
public Order getOrder(String orderId) {
    return orderRepository.findById(orderId);
}
```

传统调用方式：

```java
Order order = getOrder("ORD-10086");
```

调用方必须知道：

```text
函数叫什么？
参数是什么？
什么时候调用？
如何处理返回值？
```

而在 LLM Application 中，我们希望用户直接说：

```text
帮我查一下订单 ORD-10086。
```

然后由模型判断：

```text
需要调用 get_order
参数：
{
    "orderId": "ORD-10086"
}
```

最终形成：

```text
User
  ↓
LLM
  ↓
Function Call
  ↓
Application
  ↓
Order Service
  ↓
Database
  ↓
Tool Result
  ↓
LLM
  ↓
Final Answer
```

这就是 Function Calling。

---

# 二、最重要的认知：模型不会真正执行 Function

这是 Function Calling 最容易被误解的地方。

假设我们定义：

```python
def get_order(order_id: str):
    return database.query(order_id)
```

我们把这个函数描述给 LLM。

模型并不会直接执行：

```python
get_order("ORD-10086")
```

模型实际上只会产生类似：

```json
{
  "name": "get_order",
  "arguments": {
    "order_id": "ORD-10086"
  }
}
```

然后：

```text
LLM
 ↓
生成 Tool Call
 ↓
你的 Application
 ↓
解析 Tool Call
 ↓
真正执行 Python / Java Function
```

所以从系统架构角度：

```text
                 ┌───────────────┐
                 │      LLM      │
                 └───────┬───────┘
                         │
                  Tool Call JSON
                         │
                         ▼
                 ┌───────────────┐
                 │ Application   │
                 │ Tool Router   │
                 └───────┬───────┘
                         │
                         ▼
                 ┌───────────────┐
                 │ Real Function │
                 └───────────────┘
```

**LLM 是决策者，Application 才是执行者。**

这个边界非常重要，因为它直接决定了安全模型。

---

# 三、Function Calling 的本质：自然语言 → API Invocation

从软件工程角度看，Function Calling 并不是一种神秘的 AI 能力。

它本质上是在完成：

```text
Natural Language
       ↓
Intent Recognition
       ↓
Tool Selection
       ↓
Argument Generation
       ↓
API Invocation
```

例如：

```text
“帮我查询上海今天的天气”
```

转换成：

```json
{
  "name": "get_weather",
  "arguments": {
    "city": "Shanghai",
    "unit": "celsius"
  }
}
```

从这个角度看，Function Calling 实际上是：

> **LLM 驱动的动态 API Invocation。**

传统系统：

```text
REST API
    ↓
固定参数
    ↓
固定调用
```

AI 系统：

```text
Natural Language
    ↓
LLM
    ↓
Dynamic API Selection
    ↓
Tool
```

这就是它革命性的地方。

---

# 四、Function Calling 的核心组成

一个完整的 Function Calling 系统至少包含四个部分：

```text
1. Tool Definition
2. Tool Selection
3. Argument Generation
4. Tool Execution
```

进一步展开：

```text
                 User Request
                      │
                      ▼
                 ┌─────────┐
                 │   LLM   │
                 └────┬────┘
                      │
             ┌────────┴────────┐
             │                 │
        No Tool Needed     Tool Required
             │                 │
             ▼                 ▼
        Text Response     Tool Selection
                               │
                               ▼
                         Argument Generation
                               │
                               ▼
                         Application Validation
                               │
                               ▼
                         Tool Execution
                               │
                               ▼
                         Tool Result
                               │
                               ▼
                              LLM
                               │
                               ▼
                         Final Response
```

因此，一个 Tool 本质上就是：

```text
Name
+
Description
+
JSON Schema
+
Implementation
+
Authorization Policy
```

---

# 五、Tool Schema：LLM 如何知道一个函数？

模型并不知道：

```python
get_order()
```

是什么。

所以 Application 需要向模型提供工具描述。

例如：

```json
{
  "type": "function",
  "name": "get_order",
  "description": "Get an order by its order ID.",
  "parameters": {
    "type": "object",
    "properties": {
      "order_id": {
        "type": "string",
        "description": "The unique order ID."
      }
    },
    "required": ["order_id"],
    "additionalProperties": false
  },
  "strict": true
}
```

这里真正重要的是：

```text
description
+
parameters
+
JSON Schema
```

模型通过这些信息决定：

```text
是否需要调用
↓
调用哪个 Tool
↓
需要哪些参数
↓
参数应该是什么类型
```

现代 OpenAI Function Calling 支持 Structured Outputs；在支持的配置下，将 Function 定义为 `strict: true` 可以让模型生成的函数参数遵循给定 JSON Schema。

---

# 六、为什么 Function Schema 本身就是 Prompt？

这是一个非常值得深入理解的问题。

很多工程师认为：

```text
System Prompt
```

才是 Prompt。

实际上 Tool Definition 也会参与模型决策。

例如：

```json
{
  "name": "refund_order",
  "description": "Refund a completed customer order."
}
```

这段 description 会影响模型：

```text
User:
I want my money back.
```

模型是否选择：

```text
refund_order
```

所以：

> **Tool Description 本身就是一种 Machine-Readable Prompt。**

因此 Tool Schema 的设计质量，会直接影响：

```text
Tool Selection Accuracy
Argument Accuracy
Agent Reliability
```

一个糟糕的 Tool：

```json
{
  "name": "query",
  "description": "query something"
}
```

一个好的 Tool：

```json
{
  "name": "get_customer_orders",
  "description":
    "Retrieve orders belonging to a specific customer. "
    "Use this tool when the user asks about order history, "
    "order status, or recent purchases."
}
```

第二个 Tool 明显更容易被模型正确选择。

---

# 七、Tool Description 应该如何设计？

一个好的 Tool Description 应该回答四个问题：

```text
What?
When?
Input?
Output?
```

例如：

```text
Name:
get_customer_orders

What:
Retrieve orders for a customer.

When:
Use when the user asks about order history,
recent purchases, or order status.

Input:
customer_id
date_range

Output:
List of matching orders.
```

可以进一步加入：

```text
Do NOT use this tool when:
- The user asks about product inventory.
- The user asks to create a new order.
```

于是 Tool Definition 实际上变成了：

```text
Tool Contract
+
Tool Policy
+
Tool Metadata
```

---

# 八、Function Calling 的完整执行循环

现代 Tool Calling 并不是：

```text
Request → Tool → Response
```

而是一个循环。

典型流程：

```text
Step 1:
User → LLM

Step 2:
LLM → Tool Call

Step 3:
Application → Execute Tool

Step 4:
Application → Tool Result

Step 5:
Tool Result → LLM

Step 6:
LLM → Final Answer
```

OpenAI 的 Function Calling 文档也明确描述了这种多步骤循环：应用把可调用工具提供给模型，模型返回 Tool Call，应用执行工具，再将 Tool Output 发送回模型，模型最终返回文本或者继续产生更多 Tool Calls。

抽象成：

```text
while (!finished) {

    response = LLM(messages, tools);

    if (response.hasToolCall()) {

        for (ToolCall call : response.toolCalls()) {

            result = execute(call);

            messages.add(result);
        }

    } else {

        return response.text();
    }
}
```

这个循环实际上已经非常接近：

> **Agent Runtime。**

---

# 九、Function Calling 与 Agent 的关系

很多人会问：

> Function Calling 和 Agent 到底是什么关系？

可以这样理解：

```text
LLM
 ↓
Function Calling
 ↓
Tool
 ↓
Tool Result
 ↓
LLM
 ↓
Another Tool
 ↓
Tool Result
```

当这个循环开始具备：

```text
State
+
Planning
+
Tool Selection
+
Iteration
+
Termination
```

它就逐渐成为 Agent。

所以：

```text
Function Calling
```

是 Agent 的基础能力之一。

但：

```text
Function Calling ≠ Agent
```

因为单次：

```text
User → LLM → Tool → Result
```

完全可以没有 Agent。

而 Agent 更强调：

```text
Goal
 ↓
Planning
 ↓
Action
 ↓
Observation
 ↓
Replanning
 ↓
Action
 ↓
...
```

---

# 十、从 Function Calling 到 ReAct

进一步看，可以把 Agent 抽象为：

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
  ↓
Observe
```

Function Calling 提供了：

```text
Act
```

的执行接口。

例如：

```text
User:
分析生产环境 incident INC-123。

LLM:
调用 query_incident()

Tool:
返回 incident 信息。

LLM:
调用 search_logs()

Tool:
返回日志。

LLM:
调用 query_metrics()

Tool:
返回 CPU / Memory / Latency。

LLM:
综合分析。

LLM:
调用 create_incident_report()

Tool:
报告创建完成。
```

这就是：

```text
LLM
 ↓
Tool
 ↓
Observation
 ↓
LLM
 ↓
Tool
 ↓
Observation
 ↓
LLM
```

Function Calling 因此成为 Agent Runtime 的执行基础。

---

# 十一、Structured Outputs 为什么非常重要？

传统 LLM：

```text
"Please call get_order with order_id ORD-10086"
```

Application 需要自己解析：

```text
Regex
String Parsing
JSON Parsing
Validation
```

这非常脆弱。

Function Calling 将它变成：

```json
{
  "name": "get_order",
  "arguments": {
    "order_id": "ORD-10086"
  }
}
```

进一步使用：

```text
strict: true
```

可以让工具参数遵循定义的 JSON Schema。OpenAI 的 Structured Outputs 文档明确区分了 JSON Mode 和 Structured Outputs：前者主要保证 JSON 可解析，后者则用于让输出符合指定 Schema。

因此：

```text
JSON Mode
    ↓
Valid JSON

Structured Outputs
    ↓
Schema-Constrained JSON
```

对于企业应用而言，第二种更重要。

---

# 十二、Schema Validation 仍然不能省略

即使：

```text
strict: true
```

也不能意味着：

```text
Application = 不需要验证
```

原因很简单：

> **Schema 正确，不代表业务语义正确。**

例如：

```json
{
  "amount": 1000000000
}
```

JSON Schema 可以接受：

```text
amount: number
```

但是业务系统可能规定：

```text
Maximum Refund Amount = $10,000
```

所以必须：

```text
LLM
 ↓
Schema Validation
 ↓
Business Validation
 ↓
Authorization
 ↓
Execution
```

这几个层次不能混在一起。

---

# 十三、Schema Validation 与 Business Validation

建议至少分成三层。

### 第一层：Schema Validation

验证：

```text
Type
Required Fields
Enum
Structure
Format
```

例如：

```text
order_id: string
currency: enum
amount: number
```

---

### 第二层：Business Validation

例如：

```text
amount > 0
amount <= refundableAmount
order belongs to customer
order status == COMPLETED
```

---

### 第三层：Authorization

例如：

```text
当前用户是否可以退款？

当前 Agent 是否有退款权限？

当前 Tool 是否允许操作生产数据？
```

最终：

```text
Tool Call
   ↓
Schema Validation
   ↓
Business Validation
   ↓
Authorization
   ↓
Execution
```

这才是生产级设计。

---

# 十四、Function Calling 最大的安全问题

Function Calling 一旦能够执行真实操作，风险会迅速扩大。

例如 Tool：

```text
delete_user()
refund_payment()
send_email()
deploy_production()
execute_sql()
delete_database()
```

如果把这些工具直接暴露给 LLM：

```text
LLM
 ↓
Tool
 ↓
Production
```

这是非常危险的。

正确架构应该是：

```text
LLM
 ↓
Tool Request
 ↓
Policy Engine
 ↓
Authorization
 ↓
Risk Assessment
 ↓
Human Approval?
 ↓
Execution
```

也就是说：

> **Tool Calling 本身不是安全边界。**

真正的安全边界应该在 Tool Execution Layer。

---

# 十五、把 Tool 分成 Read 与 Write

这是一个非常实用的企业架构原则。

### Read Tools

例如：

```text
get_order()
query_customer()
search_logs()
query_metrics()
get_inventory()
```

特点：

```text
Read-only
Low Risk
```

---

### Write Tools

例如：

```text
create_order()
refund_order()
send_email()
delete_user()
deploy_service()
```

特点：

```text
State Change
High Risk
```

因此可以设计：

```text
Tool Risk Level

READ
LOW

WRITE
MEDIUM

FINANCIAL
HIGH

PRODUCTION
CRITICAL
```

然后：

```text
READ
→ 自动执行

LOW WRITE
→ 自动执行 + Audit

HIGH RISK
→ Human Approval

CRITICAL
→ Human Approval + MFA / Policy
```

这比简单地告诉模型：

```text
Do not perform dangerous operations.
```

可靠得多。

---

# 十六、Human-in-the-loop 如何与 Function Calling 结合？

这也是 Function Calling 进入企业环境后非常重要的一点。

例如用户说：

```text
把订单 ORD-10086 退款。
```

Agent 产生：

```json
{
  "name": "refund_order",
  "arguments": {
    "order_id": "ORD-10086"
  }
}
```

此时不要直接执行。

而是：

```text
LLM
 ↓
Tool Call
 ↓
Risk Engine
 ↓
Human Approval
 ↓
refund_order()
```

UI 可以显示：

```text
AI wants to execute:

refund_order(
    order_id = ORD-10086
)

Amount:
$2,300

Reason:
Customer requested refund.

[Approve] [Reject]
```

批准之后：

```text
Human
 ↓
Approval
 ↓
Tool Execution
```

这样 AI 获得了：

> **建议权**

但人类保留：

> **最终执行权**

这就是 Agent Governance 的基础。

---

# 十七、Parallel Function Calling

假设用户问：

```text
告诉我上海、北京、广州现在的天气。
```

模型可以产生：

```text
get_weather("Shanghai")
get_weather("Beijing")
get_weather("Guangzhou")
```

如果三个调用之间互不依赖，就没必要：

```text
Shanghai
 ↓
Beijing
 ↓
Guangzhou
```

可以：

```text
Shanghai ─┐
Beijing  ─┼→ Parallel
Guangzhou ─┘
```

这样可以显著减少：

```text
Latency
```

现代 Function Calling 支持并行工具调用；其核心价值就是在多个独立工具调用之间减少串行 Round Trips.

因此 Agent Runtime 应该区分：

```text
Independent Tool Calls
        ↓
Parallel Execution

Dependent Tool Calls
        ↓
Sequential Execution
```

---

# 十八、Tool Dependency Graph

更进一步，可以把 Tool Calling 看成一个 DAG。

例如：

```text
get_customer()
       │
       ▼
get_orders()
       │
       ├─────────────┐
       ▼             ▼
get_payment()    get_shipping()
       │             │
       └──────┬──────┘
              ▼
        generate_report()
```

这里：

```text
get_customer
```

必须先完成。

而：

```text
get_payment
get_shipping
```

可以并行。

因此 Agent Runtime 可以进行：

```text
Dependency Analysis
        ↓
Execution Planning
        ↓
Parallel Scheduling
```

这时候 Function Calling 已经开始从：

> API 调用

演进为：

> **AI-driven workflow execution。**

---

# 十九、Function Calling 的错误处理

生产系统中 Tool 一定会失败。

例如：

```text
Database Timeout
HTTP 500
Permission Denied
Rate Limit
Invalid Parameter
Business Error
Service Unavailable
```

不要简单地：

```python
try:
    execute()
except Exception:
    return "error"
```

更好的方式是把错误作为结构化 Tool Result 返回给模型：

```json
{
  "success": false,
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Order ORD-10086 does not exist."
  }
}
```

然后：

```text
Tool Error
    ↓
LLM
    ↓
Decide:
    Retry?
    Another Tool?
    Ask User?
    Final Answer?
```

这使 Agent 能够进行：

```text
Error Recovery
```

---

# 二十、但是不要让 Agent 无限 Retry

这是生产环境非常容易出现的问题。

例如：

```text
LLM
 ↓
Tool
 ↓
Timeout
 ↓
Retry
 ↓
Timeout
 ↓
Retry
 ↓
Timeout
 ↓
...
```

最终：

```text
Cost ↑
Latency ↑
Load ↑
```

因此 Agent Runtime 应该具备：

```text
Max Iterations
Max Tool Calls
Timeout
Retry Policy
Backoff
Circuit Breaker
Budget
```

例如：

```text
max_iterations = 10
max_tool_calls = 20
timeout = 30s
```

这和传统微服务中的：

```text
Timeout
Retry
Circuit Breaker
Bulkhead
Rate Limiting
```

非常类似。

因此：

> **Agent Runtime 本质上正在重新使用大量传统分布式系统设计思想。**

---

# 二十一、Idempotency：Function Calling 中经常被忽略的问题

假设 Agent 调用：

```text
refund_order()
```

第一次：

```text
Request
 ↓
Payment Service
 ↓
Refund Success
```

但是网络超时：

```text
Payment Success
        ↓
Network Timeout
        ↓
Agent thinks: Failed
```

然后 Agent 再次：

```text
refund_order()
```

如果没有幂等设计：

```text
Refund
+
Refund
```

就可能产生严重业务问题。

因此 Write Tool 必须考虑：

```text
Idempotency Key
```

例如：

```text
agent_execution_id
+
tool_call_id
```

形成：

```text
idempotency_key =
agent-123:toolcall-456
```

这样：

```text
Retry
 ↓
Same Idempotency Key
 ↓
Return Existing Result
```

而不是重新执行。

这是传统分布式系统经验在 Agent 世界中的直接迁移。

---

# 二十二、Tool Calling 与传统 API Gateway 的区别

Function Calling 看起来很像 API Gateway。

但是两者解决的问题不同。

传统 API Gateway：

```text
Client
 ↓
API Gateway
 ↓
Fixed API
```

Function Calling：

```text
Natural Language
 ↓
LLM
 ↓
Dynamic Tool Selection
 ↓
API
```

传统 API Gateway 的核心是：

```text
Routing
Authentication
Rate Limiting
Load Balancing
```

而 Agent Tool Layer 更强调：

```text
Intent
Tool Selection
Argument Generation
Policy
Execution
Observation
```

因此未来企业 AI Architecture 中，很可能出现：

```text
                    User
                      │
                      ▼
                 AI Gateway
                      │
                      ▼
                  Agent
                      │
                      ▼
                 Tool Gateway
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
     REST API       MCP          Internal
                                 Services
```

---

# 二十三、Function Calling 与 MCP

随着 AI Tool Ecosystem 的发展，MCP（Model Context Protocol）成为另一个重要概念。

Function Calling 更像：

```text
Application
    ↓
Define Tool
    ↓
LLM
```

而 MCP 更强调：

```text
AI Client
    ↓
MCP Protocol
    ↓
MCP Server
    ↓
Tools / Resources
```

因此可以简单理解：

```text
Function Calling
=
模型如何调用工具

MCP
=
工具如何以标准协议暴露给 AI 系统
```

二者不是完全竞争关系。

Function Calling 可以作为 Agent Runtime 的调用机制，而 MCP 可以成为工具发现和连接的一种标准化方式。

---

# 二十四、Function Calling 与传统微服务架构的结合

对于 Java 后端工程师来说，这可能是最重要的部分。

假设已有微服务：

```text
Order Service
Payment Service
Customer Service
Inventory Service
Shipping Service
```

传统架构：

```text
Frontend
   ↓
API Gateway
   ↓
Microservices
```

引入 Agent：

```text
                    User
                      │
                      ▼
                  AI Agent
                      │
              ┌───────┼───────┐
              ▼       ▼       ▼
           Order   Payment  Customer
            Tool     Tool      Tool
              │       │       │
              ▼       ▼       ▼
           Service  Service  Service
```

这里 Tool Layer 可以成为：

> **AI 与企业微服务之间的 Anti-Corruption Layer。**

这是非常重要的架构思想。

不要让 LLM 直接：

```text
LLM → Database
```

更不要：

```text
LLM → Production Shell
```

而应该：

```text
LLM
 ↓
Domain Tool
 ↓
Business Service
 ↓
Database
```

例如：

```text
get_customer_orders()
```

比：

```text
execute_sql()
```

安全得多。

---

# 二十五、为什么不应该给 Agent 一个 execute_sql Tool？

很多 Demo 会这样：

```text
execute_sql(sql)
```

然后：

```text
User:
查询过去一个月销售额最高的客户。
```

LLM：

```sql
SELECT ...
```

虽然很方便，但生产环境风险极高。

因为模型可以产生：

```sql
DELETE
UPDATE
DROP
ALTER
```

甚至可能受到 Prompt Injection 影响。

更好的方式：

```text
get_top_customers(
    start_date,
    end_date,
    limit
)
```

也就是说：

> **Expose Domain-Level Tools, not Infrastructure-Level Tools.**

这是企业 Agent 设计中非常重要的原则。

---

# 二十六、Tool 粒度：太粗和太细都不好

Tool 设计存在一个经典问题：

### Tool 太粗

```text
manage_customer()
```

内部包含：

```text
create
update
delete
query
disable
refund
```

模型很难准确使用。

### Tool 太细

```text
validate_customer_id()
get_customer_name()
get_customer_status()
get_customer_address()
...
```

Tool 数量爆炸。

最终：

```text
Tool Selection Complexity ↑
Prompt Size ↑
Context Size ↑
```

因此通常应该按照：

> **Business Capability**

设计 Tool。

例如：

```text
get_customer_profile()
get_customer_orders()
create_order()
cancel_order()
refund_order()
```

而不是：

```text
execute_database_query()
```

---

# 二十七、Tool 数量也是一个架构问题

假设 Agent 有：

```text
10 tools
```

模型比较容易选择。

但如果：

```text
500 tools
```

问题就来了：

```text
Tool Definitions ↑
Context ↑
Selection Complexity ↑
Latency ↑
Cost ↑
```

所以企业级 Agent Platform 通常需要：

```text
Tool Registry
       ↓
Tool Discovery
       ↓
Relevant Tools
       ↓
LLM
```

而不是每一次请求都把所有 Tool 定义发送给模型。

这与传统微服务中的：

```text
Service Discovery
```

有一定相似性。

---

# 二十八、Function Calling 的 Observability

传统微服务 Trace：

```text
HTTP
 ↓
Service A
 ↓
Service B
 ↓
Database
```

Agent Trace：

```text
User Request
 ↓
LLM
 ↓
Tool Selection
 ↓
Tool Execution
 ↓
LLM
 ↓
Tool Execution
 ↓
Final Answer
```

所以需要记录：

```text
Trace ID
Conversation ID
Agent ID
Model
Prompt Version
Tool Name
Tool Arguments
Tool Latency
Tool Result
Token Usage
Error
Retry
Human Approval
```

一个完整 Trace 可能是：

```text
Trace: abc-123

LLM Call #1
  Model: GPT
  Tokens: 1200

Tool Call #1
  Tool: get_customer
  Latency: 35ms

Tool Call #2
  Tool: get_orders
  Latency: 120ms

LLM Call #2
  Tokens: 900

Final Response
```

这就是 Agent Observability。

---

# 二十九、Function Calling 的 Metrics

建议至少监控：

```text
Tool Selection Accuracy
Tool Call Success Rate
Tool Execution Latency
Tool Error Rate
Tool Retry Rate
Invalid Argument Rate
Human Approval Rate
Agent Completion Rate
Average Tool Calls / Request
Average LLM Calls / Request
Token Cost
```

尤其值得关注：

```text
Average Tool Calls / Request
```

如果：

```text
正常 = 3
现在 = 15
```

很可能意味着 Agent 出现：

```text
Loop
Poor Tool Selection
Retry Problem
Prompt Regression
```

这和传统系统中的：

```text
Request per Second
Error Rate
Latency
```

同样重要。

---

# 三十、Function Calling 的 Evaluation

Function Calling 不应该只测试：

```text
最终答案对不对？
```

还应该测试：

```text
Tool Selection
Argument Accuracy
Execution
Final Answer
```

例如：

```text
Test Case:

User:
查询 ORD-10086 的订单状态。
```

Expected：

```text
Tool:
get_order

Arguments:
{
    "order_id": "ORD-10086"
}
```

Evaluation：

```text
Tool Selection = Correct
Argument = Correct
Execution = Success
Final Answer = Correct
```

因此可以建立：

```text
Function Calling Evaluation Dataset
```

例如：

```text
1000 User Requests
        ↓
Expected Tool
Expected Arguments
Expected Result
        ↓
Agent
        ↓
Compare
```

这比单纯人工测试可靠得多。

---

# 三十一、一个生产级 Function Calling 架构

综合前面的讨论，一个比较完整的架构可以是：

```text
                         User
                           │
                           ▼
                     API Gateway
                           │
                           ▼
                    Agent Runtime
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
             LLM Gateway          State Store
                │
                ▼
              LLM
                │
          Tool Call Decision
                │
                ▼
            Tool Router
                │
        ┌───────┼────────┐
        ▼       ▼        ▼
      Policy  Schema   AuthZ
      Engine  Validate
        │       │        │
        └───────┼────────┘
                │
                ▼
          Human Approval
             (optional)
                │
                ▼
           Tool Executor
                │
        ┌───────┼─────────┐
        ▼       ▼         ▼
      Order   Payment   Customer
      Service Service   Service
        │       │         │
        └───────┼─────────┘
                ▼
           Tool Result
                │
                ▼
               LLM
                │
                ▼
          Final Response
```

横向还需要：

```text
Observability
Security
Audit
Rate Limiting
Timeout
Retry
Circuit Breaker
Evaluation
Cost Management
```

这已经不是一个简单的：

```python
client.chat.completions.create(...)
```

而是一个完整的：

> **Agent Execution Platform。**

---

# 三十二、Function Calling 与传统软件工程的本质结合

如果把 Function Calling 放到软件架构演进中来看，会发现一个非常有意思的变化。

传统：

```text
User
 ↓
API
 ↓
Business Logic
 ↓
Database
```

AI：

```text
User
 ↓
LLM
 ↓
Intent
 ↓
Tool
 ↓
Business Logic
 ↓
Database
```

传统系统的核心是：

```text
Deterministic Control Flow
```

AI 系统增加了：

```text
Probabilistic Decision Making
```

于是架构变成：

```text
Probabilistic Layer
        ↓
Deterministic Layer
```

这其实是 Function Calling 最深层的架构价值：

> **让概率性的 LLM 与确定性的传统软件系统建立可控连接。**

---

# 三十三、最重要的架构原则

如果要把 Function Calling 总结成几条生产级原则，我认为最重要的是下面这些。

### 原则一：LLM 不执行 Tool

```text
LLM = Decide
Application = Execute
```

---

### 原则二：Tool 是 Contract

```text
Name
Description
Schema
Policy
Implementation
```

---

### 原则三：Schema Validation 不等于 Business Validation

```text
Schema
 ↓
Business Rules
 ↓
Authorization
```

---

### 原则四：Read 与 Write 分离

```text
Read Tool
Write Tool
High-Risk Tool
```

不同风险等级使用不同执行策略。

---

### 原则五：不要直接暴露基础设施能力

不要：

```text
execute_sql()
execute_shell()
kubectl()
```

优先：

```text
get_customer_orders()
restart_service()
refund_order()
```

也就是：

> **Domain Tool > Infrastructure Tool**

---

### 原则六：所有 Write Tool 都要考虑 Idempotency

尤其：

```text
Payment
Order
Email
Deployment
Database Mutation
```

---

### 原则七：Agent 必须有限制

至少限制：

```text
Max Iterations
Max Tool Calls
Timeout
Token Budget
Execution Budget
```

---

### 原则八：高风险操作需要 Human-in-the-loop

```text
AI Recommendation
       ↓
Human Approval
       ↓
Execution
```

---

# 三十四、Function Calling 的未来

Function Calling 的发展实际上正在推动一个新的软件架构模式：

```text
Traditional Software
       ↓
API
       ↓
AI Tool
       ↓
Agent
       ↓
Agent Platform
       ↓
Multi-Agent System
```

未来企业系统中可能出现：

```text
                    AI Application
                          │
                    Agent Runtime
                          │
                ┌─────────┼─────────┐
                ▼         ▼         ▼
              Tools      RAG      Memory
                │
         ┌──────┼──────┐
         ▼      ▼      ▼
       REST    MCP    Events
         │      │      │
         └──────┼──────┘
                ▼
          Enterprise Systems
```

最终：

> Tool 不再只是一个函数。

它会逐渐变成一种：

> **AI 可发现、可调用、可授权、可观测的业务能力。**

---

# 三十五、总结：Function Calling 真正改变了什么？

如果只从 API 使用角度理解 Function Calling：

```text
LLM → JSON → Function
```

那么它只是一个方便的 API Feature。

但如果从软件架构角度看：

```text
User
 ↓
Natural Language
 ↓
LLM
 ↓
Intent
 ↓
Tool Selection
 ↓
Structured Arguments
 ↓
Policy
 ↓
Authorization
 ↓
Business Service
 ↓
Tool Result
 ↓
LLM
 ↓
Final Response
```

Function Calling 实际上完成了一件非常重要的事情：

> **把自然语言世界和确定性的软件世界连接起来。**

它让：

```text
LLM
```

从一个只能：

```text
Generate Text
```

的模型，逐渐变成能够：

```text
Observe
Decide
Call Tools
Receive Results
Take Next Action
```

的智能执行单元。

而当 Function Calling 再与：

```text
RAG
+
Memory
+
Workflow
+
Agent
+
Human-in-the-loop
+
Observability
+
Governance
```

结合之后，就形成了现代 Agentic AI Application 的核心架构。

因此，对于软件工程师来说，真正值得掌握的并不是：

> “如何调用一个 Function？”

而是下面这个更深层的问题：

> **如何把 LLM 的非确定性决策能力，安全地连接到企业系统的确定性执行能力？**

这个问题的答案，就是：

```text
                 LLM
                  │
              Function Calling
                  │
            Structured Contract
                  │
              Tool Gateway
                  │
        ┌─────────┼─────────┐
        │         │         │
      Policy     Auth     Validation
        │         │         │
        └─────────┼─────────┘
                  │
             Tool Executor
                  │
          Enterprise Services
                  │
          Human / Governance
```

**Function Calling 不是简单的“让 AI 调函数”，而是 AI Application 从“生成内容”走向“执行能力”的关键架构边界。**

而一旦理解了这一点，就能自然理解后面的：

```text
Function Calling
       ↓
Tool Calling
       ↓
Agent
       ↓
LangGraph
       ↓
Multi-Agent
       ↓
Human-in-the-loop
       ↓
Agent Runtime
       ↓
Enterprise AI Platform
```

这条技术演进路线。
