---
title: Structured Output 深度解析：让 LLM 从“生成文本”走向“可靠的数据接口”
# tags:
#   - nodejs
date: '2026-08-18'
summary: 让 LLM 的输出不仅“语义正确”，还必须符合应用程序定义的数据结构。
---


## 引言

在 LLM 应用开发早期，我们通常这样调用模型：

```text
User
 ↓
Prompt
 ↓
LLM
 ↓
Natural Language
```

例如：

```text
用户：
分析这个生产事故。

LLM：
这个事故主要由数据库连接池耗尽导致，
建议增加连接池大小，同时优化慢 SQL……
```

对于聊天场景，这种输出完全没有问题。

但如果 LLM 是企业软件系统中的一个组件，问题马上出现：

```text
LLM
 ↓
文本
 ↓
Java Application
 ↓
Business Logic
```

程序如何可靠地解析这段文本？

我们可能会开始写：

```java
String category = extractCategory(text);
String severity = extractSeverity(text);
String summary = extractSummary(text);
```

然后：

```text
Regex
String Parsing
Substring
JSON Parsing
```

很快系统就会变得脆弱。

真正的问题是：

> **LLM 擅长生成自然语言，而传统软件需要结构化数据。**

Structured Output 正是解决这个问题的重要技术。

它的核心思想可以概括为：

> **让 LLM 的输出不仅“语义正确”，还必须符合应用程序定义的数据结构。**

因此，Structured Output 并不是简单地：

```text
“请返回 JSON”
```

而是把：

```text
LLM Output
```

进一步变成：

```text
LLM → Schema-Constrained Data → Application
```

这意味着 LLM 开始从：

> **文本生成器**

逐渐变成：

> **软件系统中的结构化数据生成组件。**

---

# 一、为什么 Structured Output 如此重要？

传统 API：

```http
GET /orders/123
```

返回：

```json
{
  "id": "123",
  "status": "SHIPPED",
  "amount": 299.99
}
```

客户端知道：

```text
id      → String
status  → Enum
amount  → Number
```

因此：

```text
API Contract
```

是确定的。

但是普通 LLM：

```text
User
 ↓
LLM
 ↓
"The order has been shipped and the amount is..."
```

没有稳定 Contract。

于是：

```text
Traditional API

Input
  ↓
Schema
  ↓
Business Logic
  ↓
Output
```

变成：

```text
LLM Application

Input
  ↓
Prompt
  ↓
Probabilistic Model
  ↓
Text
  ↓
Parsing
  ↓
Business Logic
```

这里最大的架构问题就是：

> **LLM 和 Application 之间缺少可靠的数据契约。**

Structured Output 的目标就是重新建立这个 Contract：

```text
LLM
 ↓
Structured Schema
 ↓
Application
```

---

# 二、Structured Output 不等于 JSON

这是第一个必须澄清的概念。

很多人会认为：

```text
Structured Output = JSON
```

实际上不完全正确。

JSON 只是一种数据表示格式：

```json
{
  "name": "Vincent",
  "age": 30
}
```

而 Structured Output 关注的是：

> **输出是否符合预先定义的结构和约束。**

例如：

```json
{
  "severity": "HIGH",
  "category": "DATABASE",
  "confidence": 0.95
}
```

我们定义 Schema：

```text
severity:
    LOW | MEDIUM | HIGH | CRITICAL

category:
    DATABASE | NETWORK | APPLICATION

confidence:
    number between 0 and 1
```

这才是真正的 Structured Output。

所以：

```text
JSON
    ↓
Representation

Schema
    ↓
Contract

Structured Output
    ↓
Model Output Constrained By Contract
```

---

# 三、JSON Mode 与 Structured Output 的区别

这是理解 Structured Output 最重要的知识点之一。

假设 Prompt：

```text
Return the answer as JSON.
```

模型可能返回：

```json
{
  "name": "Vincent",
  "age": 30
}
```

这看起来很好。

但是 JSON Mode 主要解决的是：

> **输出是合法 JSON。**

它并不天然保证：

```text
Field 存在
Field 类型正确
Enum 正确
Schema 完整
业务结构符合预期
```

例如你要求：

```json
{
  "name": "string",
  "age": "number"
}
```

模型可能返回：

```json
{
  "name": "Vincent"
}
```

它是合法 JSON。

但是：

```text
Schema = Invalid
```

因此：

```text
JSON Mode
    ↓
Valid JSON

Structured Output
    ↓
Valid JSON
+
Schema Conformance
```

这是两者最重要的区别。

---

# 四、Structured Output 的核心：Schema

Structured Output 的真正核心不是 JSON，而是：

> **Schema。**

例如定义一个订单：

```json
{
  "type": "object",
  "properties": {
    "orderId": {
      "type": "string"
    },
    "status": {
      "type": "string",
      "enum": [
        "CREATED",
        "PAID",
        "SHIPPED",
        "COMPLETED",
        "CANCELLED"
      ]
    },
    "amount": {
      "type": "number"
    }
  },
  "required": [
    "orderId",
    "status",
    "amount"
  ],
  "additionalProperties": false
}
```

这个 Schema 就是：

```text
LLM
 ↓
Contract
 ↓
Application
```

之间的协议。

从架构角度看：

```text
Structured Output
        =
AI Contract
```

这和传统 API 中的：

```text
OpenAPI
JSON Schema
Protobuf
Avro
GraphQL Schema
```

具有类似的工程思想。

---

# 五、Structured Output 的本质：约束生成

LLM 的普通生成过程可以抽象成：

```text
Prompt
  ↓
Token Probability
  ↓
Next Token
  ↓
Next Token
  ↓
...
```

也就是：

```text
P(token | context)
```

而 Structured Output 希望增加一个约束：

```text
P(token | context, schema constraints)
```

可以抽象成：

```text
                    LLM
                     │
              Token Generation
                     │
              Schema Constraint
                     │
                     ▼
             Valid Structured Data
```

所以 Structured Output 的核心思想并不是：

> “生成 JSON。”

而是：

> **在生成过程中加入结构约束。**

这也是为什么真正的 Structured Output 和单纯 Prompt：

```text
Please return valid JSON.
```

在可靠性上存在本质区别。

---

# 六、为什么“请返回 JSON”不可靠？

假设：

```text
Prompt:

Return the following information as JSON:

name
age
occupation
```

模型可能返回：

```json
{
  "name": "Vincent",
  "age": 35,
  "occupation": "Software Engineer"
}
```

很好。

但也可能：

```text
{
  "name": "Vincent",
  "age": "35",
  "occupation": "Software Engineer",
  "summary": "..."
}
```

甚至：

```text
Here is the JSON:

{
  ...
}
```

对于人类来说都很容易理解。

对于程序：

```text
Parser
 ↓
Unexpected Output
 ↓
Exception
```

所以：

```text
Prompt Instruction
```

是一种：

> **Soft Constraint**

而：

```text
Schema-Constrained Output
```

更接近：

> **Hard Constraint**

可以抽象为：

```text
Prompt
   ↓
Behavior Guidance

Schema
   ↓
Structural Constraint
```

这就是 Structured Output 的工程价值。

---

# 七、一个完整的 Structured Output Pipeline

生产环境可以抽象为：

```text
                    User Input
                        │
                        ▼
                   Prompt Builder
                        │
                        ▼
                       LLM
                        │
                  Structured Output
                        │
                        ▼
                  Schema Validation
                        │
               ┌────────┴────────┐
               │                 │
             Valid             Invalid
               │                 │
               ▼                 ▼
         Business Logic       Retry / Repair
               │
               ▼
             Result
```

进一步可以增加：

```text
Authentication
Authorization
Business Validation
Observability
Evaluation
```

最终：

```text
User
 ↓
Prompt
 ↓
LLM
 ↓
Structured Output
 ↓
Schema Validation
 ↓
Business Validation
 ↓
Authorization
 ↓
Business Logic
```

这已经非常接近传统企业 API Pipeline。

---

# 八、从自然语言到 Java Object

对于 Java 后端工程师来说，可以把 Structured Output 理解成：

```text
LLM
 ↓
JSON
 ↓
DTO
```

例如：

```java
public record IncidentAnalysis(
    String incidentId,
    Severity severity,
    IncidentCategory category,
    double confidence,
    String rootCause,
    List<String> recommendations
) {}
```

定义：

```java
enum Severity {
    LOW,
    MEDIUM,
    HIGH,
    CRITICAL
}
```

然后模型输出：

```json
{
  "incidentId": "INC-10086",
  "severity": "HIGH",
  "category": "DATABASE",
  "confidence": 0.93,
  "rootCause": "Connection pool exhaustion",
  "recommendations": [
    "Increase connection pool capacity",
    "Investigate slow queries"
  ]
}
```

Application：

```text
JSON
 ↓
Jackson
 ↓
IncidentAnalysis
 ↓
Business Logic
```

这时：

> LLM 输出实际上已经成为 Java Application 的一种数据源。

---

# 九、Structured Output 与 Function Calling 的关系

这是一个非常重要的问题。

两者很容易混淆。

## Function Calling

目标是：

> **让模型产生 Tool Invocation。**

例如：

```json
{
  "name": "get_order",
  "arguments": {
    "order_id": "ORD-10086"
  }
}
```

重点是：

```text
Tool Selection
+
Tool Arguments
```

---

## Structured Output

目标是：

> **让模型产生符合 Schema 的结构化结果。**

例如：

```json
{
  "category": "PAYMENT",
  "severity": "HIGH",
  "confidence": 0.97
}
```

重点是：

```text
Output Structure
```

因此：

```text
Function Calling
       ↓
Structured Tool Arguments

Structured Output
       ↓
Structured Application Result
```

二者可以组合。

例如：

```text
User
 ↓
LLM
 ↓
Function Call
 ↓
Tool
 ↓
Raw Data
 ↓
LLM
 ↓
Structured Output
 ↓
Java DTO
```

这是企业 Agent 中非常常见的模式。

---

# 十、Structured Output 与 RAG 的关系

RAG 通常输出：

```text
Retrieved Documents
```

然后交给 LLM。

例如：

```text
Question:
为什么订单支付失败？

Context:
Document 1...
Document 2...
Document 3...
```

如果最终只生成自然语言：

```text
支付失败主要是因为银行卡余额不足。
```

程序很难进一步处理。

如果 Structured Output：

```json
{
  "answer": "支付失败主要是因为银行卡余额不足。",
  "sources": [
    "payment-policy-001",
    "payment-guide-002"
  ],
  "confidence": 0.91,
  "requires_human_review": false
}
```

Application 就可以：

```text
answer
 ↓
UI

sources
 ↓
Citation

confidence
 ↓
Evaluation

requires_human_review
 ↓
Workflow
```

所以：

```text
RAG
 ↓
Knowledge Retrieval

Structured Output
 ↓
Knowledge Packaging
```

两者结合可以显著增强企业 AI Application 的可控性。

---

# 十一、Nested Schema：真正复杂的业务对象

真实企业数据往往不是：

```json
{
  "name": "Vincent"
}
```

而是复杂嵌套结构。

例如 Incident：

```json
{
  "incidentId": "INC-10086",
  "severity": "HIGH",
  "service": {
    "name": "payment-service",
    "version": "2.4.1"
  },
  "rootCause": {
    "category": "DATABASE",
    "description": "Connection pool exhaustion",
    "confidence": 0.92
  },
  "impact": {
    "customersAffected": 1520,
    "durationMinutes": 23
  },
  "recommendations": [
    {
      "priority": "HIGH",
      "action": "Increase pool size"
    },
    {
      "priority": "MEDIUM",
      "action": "Optimize slow queries"
    }
  ]
}
```

Schema：

```text
Incident
├── incidentId
├── severity
├── service
│   ├── name
│   └── version
├── rootCause
│   ├── category
│   ├── description
│   └── confidence
├── impact
│   ├── customersAffected
│   └── durationMinutes
└── recommendations[]
    ├── priority
    └── action
```

这说明 Structured Output 已经可以处理：

```text
Object
Nested Object
Array
Enum
Primitive
Optional Field
```

从工程角度，它越来越接近：

> **AI-generated DTO。**

---

# 十二、Enum 是 Structured Output 非常重要的能力

例如：

```text
severity:
LOW
MEDIUM
HIGH
CRITICAL
```

如果只使用 Prompt：

```text
Please return severity as LOW, MEDIUM,
HIGH or CRITICAL.
```

模型可能输出：

```text
"Very High"
```

或者：

```text
"Critical"
```

甚至：

```text
"CRIT"
```

如果 Schema 定义：

```json
{
  "type": "string",
  "enum": [
    "LOW",
    "MEDIUM",
    "HIGH",
    "CRITICAL"
  ]
}
```

那么：

```text
Output
 ↓
Enum Constraint
 ↓
Valid Value
```

这对：

```text
Classification
Routing
Workflow
State Machine
```

特别重要。

---

# 十三、Structured Output + State Machine

这是一个非常有意思的架构组合。

例如订单 Agent：

```text
CREATED
   ↓
PAID
   ↓
SHIPPED
   ↓
COMPLETED
```

模型需要决定下一步状态：

```json
{
  "current_state": "PAID",
  "next_action": "SHIP_ORDER"
}
```

Schema：

```text
current_state:
CREATED | PAID | SHIPPED | COMPLETED

next_action:
PAY_ORDER | SHIP_ORDER | COMPLETE_ORDER
```

Application：

```text
Structured Output
       ↓
State Validation
       ↓
Allowed Transition?
       ↓
Execute
```

因此：

> **Structured Output 可以成为 LLM 与 State Machine 之间的桥梁。**

这对于 LangGraph 一类的 Stateful Agent Workflow 尤其重要。

---

# 十四、Structured Output + Human-in-the-loop

假设 Agent 分析贷款申请：

```json
{
  "decision": "REVIEW",
  "riskScore": 0.78,
  "reasons": [
    "High debt-to-income ratio",
    "Insufficient credit history"
  ],
  "requiresHumanApproval": true
}
```

Application：

```text
Structured Output
       ↓
Risk Policy
       ↓
requiresHumanApproval = true
       ↓
Human Review
```

UI：

```text
AI Decision

Decision:
REVIEW

Risk Score:
0.78

Reasons:
- High debt-to-income ratio
- Insufficient credit history

[Approve]
[Reject]
```

因此 Structured Output 不只是“方便解析 JSON”。

它可以直接驱动：

```text
Workflow
Decision
Approval
Routing
Escalation
```

---

# 十五、Structured Output + Tool Calling + HITL

把前面的技术组合起来：

```text
                        User
                          │
                          ▼
                         LLM
                          │
                ┌─────────┴─────────┐
                │                   │
         Structured Output      Tool Call
                │                   │
                ▼                   ▼
          Decision Object       Tool Request
                │                   │
                ▼                   ▼
           Policy Engine       Authorization
                │                   │
                └─────────┬─────────┘
                          ▼
                    Human Approval
                          │
                          ▼
                       Execute
```

例如：

```json
{
  "action": "REFUND_ORDER",
  "orderId": "ORD-10086",
  "amount": 2500,
  "riskLevel": "HIGH",
  "requiresApproval": true
}
```

这已经非常接近一个：

> **AI Decision Contract。**

---

# 十六、Schema Validation 与业务规则必须分离

这是企业系统设计中非常重要的一点。

假设：

```json
{
  "amount": 5000,
  "currency": "USD"
}
```

Schema 可能定义：

```text
amount = number
currency = USD | EUR | GBP
```

那么：

```text
5000 USD
```

结构合法。

但是业务规则可能规定：

```text
VIP Customer:
Maximum Refund = $10,000

Normal Customer:
Maximum Refund = $500
```

因此：

```text
Schema Validation
```

只能回答：

> “这个数据结构正确吗？”

而：

```text
Business Validation
```

回答：

> “这个操作在业务上允许吗？”

所以：

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

绝不能因为：

```text
Structured Output = Valid
```

就直接：

```text
Execute
```

---

# 十七、Structured Output 并不能解决 Hallucination

这是另一个非常重要的认知。

假设 Schema：

```json
{
  "customerName": "string",
  "balance": "number"
}
```

模型输出：

```json
{
  "customerName": "John",
  "balance": 1000000
}
```

Schema 完全正确。

但是：

```text
balance = 1000000
```

可能是模型编造的。

所以：

```text
Structured Output
≠
Factually Correct Output
```

它保证的是：

```text
Structure
```

而不是：

```text
Truth
```

因此企业系统仍然需要：

```text
RAG
Grounding
Tool Verification
Database Lookup
Business Validation
```

例如：

```text
LLM
 ↓
Structured Output
 ↓
Customer ID
 ↓
Database
 ↓
Verify Balance
```

这样才能把：

```text
Structure
```

和：

```text
Truth
```

结合起来。

---

# 十八、Structured Output 与 Temperature

LLM 输出具有概率性。

很多人会认为：

```text
temperature = 0
```

就能保证结构稳定。

实际上：

```text
Temperature
```

主要影响采样行为。

它不能替代：

```text
Schema Constraint
```

所以不要采用：

```text
temperature = 0
+
Prompt = "Return valid JSON"
```

作为可靠性方案。

更合理的是：

```text
Schema
+
Structured Output
+
Validation
```

然后再根据任务需求调整：

```text
Temperature
```

---

# 十九、Structured Output 的 Failure Modes

生产系统中至少需要考虑以下失败模式。

## 1. Schema Validation Failure

```text
LLM
 ↓
Invalid Structure
```

处理：

```text
Retry
Repair
Fallback
```

---

## 2. Semantic Validation Failure

结构正确：

```json
{
  "amount": 999999999
}
```

但业务不允许。

处理：

```text
Business Validation
```

---

## 3. Hallucination

结构正确：

```text
customerId = "C999"
```

但数据库不存在。

处理：

```text
External Verification
```

---

## 4. Missing Information

模型无法确定：

```text
customer_id
```

不要让模型猜。

可以设计：

```json
{
  "customerId": null,
  "needsClarification": true
}
```

这比：

```text
customerId = "C123"
```

安全得多。

---

## 5. Model Failure

```text
Timeout
Rate Limit
Service Unavailable
```

需要：

```text
Retry
Fallback Model
Circuit Breaker
```

---

# 二十、Nullable 与 Optional Field 的设计

这是 Structured Output 很容易踩坑的地方。

假设：

```text
customerMiddleName
```

可能没有。

不要让模型：

```text
"customerMiddleName": "N/A"
```

因为：

```text
N/A
Unknown
None
null
""
```

语义可能不同。

更好的设计：

```json
{
  "customerMiddleName": null
}
```

然后 Schema 明确：

```text
string | null
```

这样 Application 可以明确区分：

```text
Value exists
Value unavailable
```

---

# 二十一、不要让 Schema 过度复杂

理论上 Schema 越详细：

```text
Constraint ↑
```

但并不意味着：

```text
Reliability 无限提高
```

过于复杂的 Schema 会带来：

```text
Prompt / Schema Size ↑
Model Complexity ↑
Latency ↑
Evaluation Complexity ↑
```

因此应该遵循：

> **Schema 只表达 Application 真正需要的结构。**

不要把：

```text
100 个字段
```

全部塞给模型。

如果应用真正只需要：

```text
category
severity
summary
```

那么：

```json
{
  "category": "...",
  "severity": "...",
  "summary": "..."
}
```

可能反而更可靠。

---

# 二十二、Structured Output 的性能问题

Schema 越复杂，生成过程可能越复杂。

同时：

```text
Input Tokens
+
Schema Tokens
+
Output Tokens
```

都会影响：

```text
Cost
Latency
```

因此生产系统需要考虑：

```text
Schema Size
Prompt Size
Context Size
Output Size
```

尤其当：

```text
100+ Tools
+
Large JSON Schema
+
RAG Context
```

同时存在时：

```text
Context Window
```

会迅速增长。

所以 Structured Output 设计也属于：

> **Context Engineering。**

---

# 二十三、Structured Output 与 Context Engineering

可以把一个 LLM Request 看成：

```text
System Prompt
+
User Prompt
+
RAG Context
+
Tool Definitions
+
Conversation History
+
Output Schema
```

因此：

```text
Total Context
=
Instruction
+
Knowledge
+
Tools
+
State
+
Schema
```

Structured Output 的 Schema 本身也会占据 Context。

所以不能只考虑：

```text
Schema 是否正确？
```

还需要考虑：

```text
Schema 是否必要？
Schema 是否过大？
Schema 是否可以复用？
Schema 是否应该拆分？
```

这已经进入：

> **Context Engineering**

领域。

---

# 二十四、Structured Output 与 Prompt Engineering 的关系

Prompt Engineering：

```text
告诉模型应该怎么回答
```

Structured Output：

```text
限制模型必须输出什么结构
```

可以理解为：

```text
Prompt
 ↓
Soft Behavioral Constraint

Schema
 ↓
Structural Constraint
```

两者结合：

```text
System Prompt
      +
Task Prompt
      +
Context
      +
Schema
      ↓
     LLM
      ↓
Structured Result
```

因此：

> Prompt 决定“行为”，Schema 约束“结构”。

---

# 二十五、Structured Output 与 Type System

如果从编程语言角度思考，这是一个非常有趣的问题。

传统程序：

```java
record User(
    String name,
    int age
) {}
```

编译器知道：

```text
name = String
age = int
```

LLM 则天然没有这样的强类型约束。

Structured Output 相当于给模型增加：

```text
Type System
```

例如：

```text
User
├── name: String
├── age: Integer
└── role: Enum
```

因此可以把：

```text
Structured Output
```

理解成：

> **LLM 与传统类型系统之间的桥梁。**

传统软件：

```text
Dynamic / Natural Language
```

开始向：

```text
Typed AI Interface
```

演进。

---

# 二十六、从 DTO 到 AI DTO

传统后端：

```java
public record CreateOrderRequest(
    String productId,
    int quantity
) {}
```

前端：

```json
{
  "productId": "P100",
  "quantity": 2
}
```

现在：

```text
User Natural Language
        ↓
LLM
        ↓
Structured Output
        ↓
CreateOrderRequest
        ↓
Order Service
```

例如：

```text
用户：

帮我买两个 iPhone 17。
```

模型生成：

```json
{
  "productId": "IPHONE-17",
  "quantity": 2
}
```

然后：

```java
CreateOrderRequest request =
    objectMapper.readValue(json, CreateOrderRequest.class);

orderService.create(request);
```

这意味着：

> **自然语言开始成为一种新的 API Input。**

而 Structured Output 就是：

> **Natural Language → Typed DTO**

的关键桥梁。

---

# 二十七、Structured Output 与 AI Gateway

在企业级架构中，可以建立统一的 AI Gateway：

```text
Application
     ↓
AI Gateway
     ↓
Model Router
     ↓
LLM
```

Gateway 可以负责：

```text
Prompt Management
Schema Management
Model Routing
Retry
Fallback
Token Tracking
Observability
Evaluation
Security
```

例如：

```text
Incident Analysis
Schema v3

Customer Classification
Schema v2

Order Extraction
Schema v5
```

这样：

```text
Schema
+
Prompt
+
Model
```

可以作为一个版本化的 AI Contract。

例如：

```text
incident-analysis:v3
order-extraction:v5
customer-classification:v2
```

这已经非常接近传统 API：

```text
/api/v1/order
/api/v2/order
```

的思想。

---

# 二十八、Structured Output 的 Versioning

Schema 发生变化时：

```text
v1:
{
    "severity": "HIGH"
}
```

升级：

```text
v2:
{
    "severity": "HIGH",
    "confidence": 0.95
}
```

如果消费者仍然只支持：

```text
v1
```

就会产生兼容性问题。

所以 AI Application 同样需要：

```text
Schema Versioning
Backward Compatibility
Migration
Contract Testing
```

例如：

```text
IncidentAnalysis v1
IncidentAnalysis v2
IncidentAnalysis v3
```

这与：

```text
API Contract Evolution
```

非常类似。

---

# 二十九、Structured Output Contract Testing

传统微服务有：

```text
Contract Test
```

AI 系统也可以建立类似机制。

例如：

```text
Input:
Production incident text

Expected Schema:
IncidentAnalysis v2

Expected:
severity ∈ [LOW, MEDIUM, HIGH, CRITICAL]
confidence ∈ [0, 1]
```

测试：

```text
Schema Valid?
Enum Valid?
Required Fields Present?
Business Rules Valid?
```

再进一步：

```text
Semantic Evaluation
```

例如：

```text
Root Cause Accuracy
Recommendation Quality
Confidence Calibration
```

于是：

```text
Schema Validation
+
Semantic Evaluation
=
AI Contract Testing
```

---

# 三十、Structured Output 的 Evaluation Framework

完整的 Evaluation 不应该只有：

```text
Pass / Fail
```

可以分为四层：

```text
Level 1
Structural Accuracy

Level 2
Semantic Accuracy

Level 3
Business Accuracy

Level 4
Operational Quality
```

例如：

### Level 1

```text
Schema Valid = 99.5%
```

### Level 2

```text
Classification Accuracy = 94%
```

### Level 3

```text
Business Decision Accuracy = 91%
```

### Level 4

```text
P95 Latency = 1.8s
Cost / Request = $0.02
```

这样才能真正衡量一个 Structured Output 系统。

---

# 三十一、Structured Output 的 Observability

建议记录：

```text
request_id
model
prompt_version
schema_version
input_tokens
output_tokens
schema_validation_result
business_validation_result
retry_count
latency
final_result
```

例如：

```text
Trace ID: 12345

Model:
GPT

Prompt:
incident-analysis:v4

Schema:
incident-analysis:v3

Schema Validation:
PASS

Business Validation:
PASS

Latency:
1.43s

Tokens:
2,103
```

这样出现问题时可以回答：

```text
为什么这次 AI 输出错了？
```

而不是：

> “模型今天好像不太聪明。”

---

# 三十二、Structured Output 与可靠性工程

Structured Output 并不能让 LLM 变成 100% deterministic。

但是它可以显著缩小：

```text
Output Space
```

例如普通输出：

```text
无限自然语言
```

Structured Output：

```text
{
  category: enum,
  severity: enum,
  confidence: 0..1
}
```

从：

```text
Open-ended Generation
```

变成：

```text
Constrained Generation
```

因此可以把 Structured Output 看作：

> **LLM Reliability Engineering 的一个重要组成部分。**

---

# 三十三、Structured Output 与传统 API 的对比

| 特性            | REST API        | LLM + Structured Output         |
| ------------- | --------------- | ------------------------------- |
| Input         | JSON            | Natural Language / JSON         |
| Processing    | Deterministic   | Probabilistic                   |
| Output        | Schema          | Schema-constrained              |
| Validation    | Schema          | Schema + Semantic               |
| Errors        | HTTP Error      | Model / Schema / Business Error |
| Retry         | HTTP Retry      | Model + Tool Retry              |
| Contract      | OpenAPI         | JSON Schema / Typed Schema      |
| Testing       | Unit / Contract | Eval + Contract                 |
| Observability | Trace           | LLM Trace + Schema              |
| Security      | Auth            | Auth + AI Policy                |

最重要的区别：

```text
REST
=
Deterministic Contract

LLM Structured Output
=
Probabilistic Reasoning
+
Deterministic Structural Contract
```

这句话非常值得记住。

---

# 三十四、一个成熟的 Structured Output Architecture

最终可以形成如下架构：

```text
                           User
                             │
                             ▼
                      AI Application
                             │
                             ▼
                       Prompt Engine
                             │
                 ┌───────────┴───────────┐
                 │                       │
              Context                 Schema
                 │                       │
                 └───────────┬───────────┘
                             ▼
                            LLM
                             │
                     Structured Output
                             │
                             ▼
                     Schema Validator
                             │
                      ┌──────┴──────┐
                      │             │
                    Valid         Invalid
                      │             │
                      ▼             ▼
               Business Rules     Retry
                      │
                      ▼
                 Authorization
                      │
                      ▼
                 Business Logic
                      │
                      ▼
                 Enterprise System
```

横向：

```text
Observability
Evaluation
Security
Audit
Cost Management
Schema Registry
Prompt Registry
```

这已经是一个完整的：

> **Enterprise LLM Output Architecture**

---

# 三十五、最容易犯的十个错误

## 错误一：认为 JSON 就是 Structured Output

```text
JSON ≠ Schema
```

---

## 错误二：只依赖 Prompt

```text
"Please return valid JSON"
```

不是可靠的 Contract。

---

## 错误三：没有 Schema Validation

即使模型支持 Structured Output：

```text
Application
```

仍然应该进行必要验证。

---

## 错误四：把 Schema Validation 当 Business Validation

```text
Valid JSON
≠
Valid Business Operation
```

---

## 错误五：认为 Structured Output 能解决 Hallucination

它主要解决：

```text
Structure
```

而不是：

```text
Truth
```

---

## 错误六：Schema 设计过度复杂

```text
Schema 越复杂
≠
系统越可靠
```

---

## 错误七：没有处理 null / missing

模型不知道的信息不应该强迫它猜。

---

## 错误八：没有 Schema Version

AI Contract 同样需要版本管理。

---

## 错误九：没有 Evaluation Dataset

没有测试集，就无法知道：

```text
Prompt V2
```

到底是不是比：

```text
Prompt V1
```

更好。

---

## 错误十：把 LLM 输出直接用于高风险操作

必须：

```text
LLM
 ↓
Schema
 ↓
Business Validation
 ↓
Authorization
 ↓
Human Approval
 ↓
Execution
```

---

# 三十六、从 Structured Output 走向 AI Engineering

如果把我们前面讨论的几个技术连接起来：

```text
Prompt Engineering
        ↓
Structured Output
        ↓
Function Calling
        ↓
RAG
        ↓
Memory
        ↓
Agent
        ↓
Workflow
        ↓
Human-in-the-loop
        ↓
Observability
        ↓
Evaluation
        ↓
Governance
```

会发现一个非常清晰的技术演进。

Prompt Engineering 解决：

> **如何告诉 LLM 做什么。**

Structured Output 解决：

> **如何让 LLM 按程序需要的结构返回结果。**

Function Calling 解决：

> **如何让 LLM 请求外部能力。**

RAG 解决：

> **如何让 LLM 获得外部知识。**

Agent 解决：

> **如何让 LLM 连续完成复杂任务。**

HITL 解决：

> **如何让人类控制高风险行为。**

Governance 解决：

> **如何让整个系统进入企业生产环境。**

---

# 三十七、真正的核心：从“生成文本”到“AI Contract”

如果只把 Structured Output 理解成：

```text
LLM 返回 JSON
```

那么它只是一个 API Feature。

但是从软件架构角度看，它完成的是更深层的变化：

```text
过去：

LLM
 ↓
Text
 ↓
Application
```

变成：

```text
现在：

LLM
 ↓
Schema-Constrained Data
 ↓
Typed Application Object
 ↓
Business Logic
```

也就是说：

> **Structured Output 为 LLM 建立了一个类似传统 API Contract 的接口层。**

这非常重要。

因为软件工程之所以能够规模化，很大程度上依赖：

```text
Contract
Type
Interface
Schema
Validation
Testing
Versioning
```

而 Structured Output 正在把这些思想引入 LLM Application。

---

# 三十八、总结

Structured Output 的真正价值，并不是让 LLM “学会 JSON”。

它解决的是一个更加根本的问题：

> **如何让概率性的语言模型与确定性的传统软件系统建立可靠的数据契约？**

完整过程可以抽象成：

```text
Natural Language
       ↓
      LLM
       ↓
Schema-Constrained Generation
       ↓
Structured Output
       ↓
Schema Validation
       ↓
Business Validation
       ↓
Authorization
       ↓
Typed Application Object
       ↓
Business Logic
```

因此可以把 Structured Output 定义为：

> **LLM 与传统软件系统之间的 Type-Safe Boundary。**

它让 LLM 从：

```text
Chat Interface
```

逐渐变成：

```text
Intelligent Software Component
```

而当 Structured Output 与 Function Calling 结合之后：

```text
Structured Output
        +
Function Calling
        ↓
Typed Decision
        +
Typed Action
```

再进一步与：

```text
RAG
Memory
Agent
Workflow
Human-in-the-loop
Evaluation
Observability
Governance
```

结合，就形成了真正意义上的：

> **Production-Grade AI Application。**


> **如何把一个不完全确定的外部系统，转换成我们可以在强类型软件系统中安全消费的数据？**

过去我们通过：

```text
REST
JSON
OpenAPI
DTO
Schema
Validation
```

解决这个问题。

现在，在 AI 世界里，我们开始通过：

```text
Prompt
Structured Output
JSON Schema
Validation
Evaluation
```

解决同一个本质问题。

这也是 Structured Output 最重要的工程意义：

> **它不是让 AI 更会说话，而是让 AI 更容易成为软件系统的一部分。**
