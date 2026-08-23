---
title: Audit：从传统审计日志到 AI Agent 时代的智能审计体系
# tags:
#   - nodejs
date: '2026-08-08'
summary: 对系统中具有业务、安全、合规或治理意义的行为进行记录、保存、分析和追溯。
---
# Audit：从传统审计日志到 AI Agent 时代的智能审计体系

> **摘要**
>
> Audit（审计）是企业级软件系统中经常被低估、但在金融、支付、医疗、政务、互联网平台以及 AI 系统中极其重要的一项基础能力。
>
> 传统 Audit 主要解决一个问题：**谁在什么时候做了什么。**
>
> 而在微服务、Cloud Native、AI Agent 和 LLM 时代，仅仅记录用户操作已经远远不够。现代 Audit 必须回答更加复杂的问题：
>
> **谁发起了操作？哪个 Agent 做出的决策？使用了什么数据？调用了哪个 Tool？经过了什么 Policy？为什么允许？最终产生了什么结果？**
>
> 因此，现代 Audit 正从传统的 Audit Log 演进为 **Audit Event → Audit Trail → Distributed Audit → AI Audit → Governance** 的完整技术体系。
>
> 本文从企业级 Java、微服务、Cloud Native 和 AI Agent 架构视角，系统分析现代 Audit 的设计思想、数据模型、事件架构、分布式审计、不可篡改、异步化、查询、合规、AI Agent 审计以及生产级架构。

---

# 一、什么是 Audit？

Audit 可以简单理解为：

> **对系统中具有业务、安全、合规或治理意义的行为进行记录、保存、分析和追溯。**

最经典的 Audit 问题是：

```text
Who
What
When
Where
Why
Result
```

例如：

```text
User:     user001
Action:   DELETE_CUSTOMER
Resource: customer123
Time:     2026-08-23 10:30:00
Result:   SUCCESS
```

但是现代系统需要进一步回答：

```text
Who initiated?
Who authorized?
Who executed?
What data was accessed?
Which service executed it?
Which policy allowed it?
Which model made the decision?
Which tool was called?
What was the final outcome?
```

因此 Audit 本质上是一条：

> **可追溯的行为证据链。**

---

# 二、Audit 和 Log 有什么区别？

这是设计 Audit 系统时最容易混淆的问题。

很多系统直接认为：

```text
Audit = Log
```

实际上不是。

## 2.1 Application Log

Application Log 主要用于：

```text
Debugging
Troubleshooting
Performance Analysis
Error Investigation
```

例如：

```text
2026-08-23 10:30:12 ERROR PaymentService
Database connection timeout
```

它主要服务于：

> Developer / SRE

---

# 三、Audit Log

Audit Log 主要用于：

```text
Security
Compliance
Governance
Forensics
Accountability
```

例如：

```text
USER_001
TRANSFER
ACCOUNT_123
$50,000
SUCCESS
```

它主要服务于：

> Security / Compliance / Auditor / Risk / Business

因此：

```text
Application Log
      ↓
帮助系统运行

Audit Log
      ↓
证明系统发生过什么
```

---

# 四、Audit 的核心原则

一个成熟 Audit 系统应该满足：

```text
Authenticity
Integrity
Completeness
Traceability
Availability
Confidentiality
Non-repudiation
```

其中最重要的是：

## 4.1 Authenticity

必须能够证明：

> 这条 Audit Event 是谁产生的。

---

## 4.2 Integrity

必须确保：

> Audit Event 没有被篡改。

---

## 4.3 Completeness

不能只记录：

```text
SUCCESS
```

而应该记录完整生命周期：

```text
REQUEST
AUTHORIZATION
EXECUTION
RESULT
```

---

## 4.4 Traceability

必须能够通过：

```text
Trace ID
Request ID
User ID
Agent ID
Transaction ID
```

找到完整调用链。

---

# 五、Audit Event

现代 Audit 系统推荐使用 Event 模型。

一个 Audit Event 可以设计为：

```json
{
  "eventId": "evt-123",
  "eventType": "CUSTOMER_UPDATE",
  "timestamp": "2026-08-23T10:30:00Z",

  "actor": {
    "type": "USER",
    "id": "user-001"
  },

  "resource": {
    "type": "CUSTOMER",
    "id": "customer-123"
  },

  "action": "UPDATE",

  "source": {
    "service": "customer-service",
    "instance": "customer-service-01"
  },

  "request": {
    "requestId": "req-123",
    "traceId": "abc123"
  },

  "result": {
    "status": "SUCCESS"
  }
}
```

这比简单：

```text
logger.info("customer updated");
```

强很多。

---

# 六、Audit Event 的标准模型

可以把 Audit Event 抽象为：

```text
AuditEvent
│
├── Event Identity
│
├── Actor
│
├── Action
│
├── Resource
│
├── Context
│
├── Authorization
│
├── Execution
│
├── Result
│
└── Trace
```

其中：

```text
Actor
```

表示：

> 谁做的？

```text
Action
```

表示：

> 做了什么？

```text
Resource
```

表示：

> 对什么做？

```text
Result
```

表示：

> 结果是什么？

```text
Trace
```

表示：

> 从哪里来的？

---

# 七、Audit 的 Actor 模型

传统系统只有：

```text
User
```

现代系统可能存在：

```text
User
Service
Admin
System
Agent
AI Agent
Batch Job
Scheduler
External System
```

因此不能简单设计：

```text
userId
```

而应该：

```json
{
  "actorType": "AGENT",
  "actorId": "payment-agent-001"
}
```

进一步：

```text
Actor
 ├── type
 ├── id
 ├── identity
 ├── role
 ├── session
 └── parentActor
```

这对 AI Agent 审计尤其重要。

---

# 八、Delegation Audit

AI Agent 最大的特殊性之一是：

```text
User
 ↓
Agent
 ↓
Tool
 ↓
Service
```

真正的问题是：

> 到底应该审计谁？

例如：

```text
User = user001

Agent = payment-agent

Tool = transfer-money

Service = payment-service
```

如果最终产生：

```text
$50,000 transfer
```

Audit 必须保留完整的 Delegation Chain：

```text
User
 ↓
Agent
 ↓
Tool
 ↓
Service
 ↓
Transaction
```

因此：

```json
{
  "initiator": "user001",
  "agent": "payment-agent",
  "tool": "transfer-money",
  "service": "payment-service",
  "transaction": "tx-001"
}
```

这就是：

> **Delegated Audit**

---

# 九、Audit Trail

Audit Log 是单个事件。

Audit Trail 是多个事件形成的完整行为链。

例如：

```text
Login
 ↓
Search Customer
 ↓
View Account
 ↓
Modify Address
 ↓
Approve Transaction
 ↓
Payment
```

这些事件组合起来：

```text
Audit Trail
```

因此：

```text
Audit Event
      ↓
Audit Log
      ↓
Audit Trail
```

Audit Trail 的核心价值是：

> **还原完整业务行为。**

---

# 十、Distributed Audit

在单体应用中：

```text
Request
 ↓
Application
 ↓
Database
```

比较容易审计。

但是微服务：

```text
API Gateway
 ↓
Order Service
 ↓
Payment Service
 ↓
Risk Service
 ↓
Notification Service
 ↓
Kafka
```

每一个服务都可能产生 Audit Event。

因此：

```text
Audit Event
      ↓
Distributed System
```

必须能够关联起来。

最重要的字段之一就是：

```text
Trace ID
```

例如：

```text
traceId = abc123
```

所有服务：

```text
Order Service
Payment Service
Risk Service
Notification Service
```

都写入：

```text
traceId=abc123
```

最终可以还原：

```text
abc123
│
├── Order Created
├── Risk Checked
├── Payment Authorized
├── Payment Completed
└── Notification Sent
```

---

# 十一、Audit + OpenTelemetry

现代微服务 Audit 可以与 OpenTelemetry 深度结合。

例如：

```text
Trace
│
├── HTTP
│
├── Order Service
│    └── Audit Event
│
├── Payment Service
│    └── Audit Event
│
└── Risk Service
     └── Audit Event
```

Span Attributes：

```text
audit.event
audit.action
audit.actor
audit.resource
audit.result
```

这样可以把：

```text
Observability
+
Audit
```

结合起来。

但是必须注意：

> **Trace 是诊断数据，Audit 是证据数据。**

不能简单认为：

```text
Trace = Audit
```

Trace 数据通常具有采样、生命周期和性能优化策略。

Audit 对完整性和持久化要求更高。

---

# 十二、Audit Event Pipeline

企业级 Audit 不建议每个业务服务直接：

```text
INSERT audit_log
```

因为这会导致：

```text
Business Transaction
        +
Audit Transaction
```

强耦合。

更好的架构：

```text
Business Service
       ↓
Audit Event
       ↓
Kafka
       ↓
Audit Processor
       ↓
Audit Storage
```

例如：

```text
Order Service
      │
      ↓
Audit Event
      │
      ↓
     Kafka
      │
      ├────────→ Elasticsearch
      │
      ├────────→ PostgreSQL
      │
      └────────→ Object Storage
```

---

# 十三、为什么 Kafka 适合 Audit？

Audit 通常具有：

```text
High Throughput
Append-only
Asynchronous
Replay
Durability
Multiple Consumers
```

Kafka 非常适合。

例如：

```text
Audit Event
     ↓
   Kafka
     │
     ├── Audit Storage
     ├── Security Analytics
     ├── Fraud Detection
     ├── Compliance
     └── Data Warehouse
```

一个 Audit Event 可以被多个系统消费。

---

# 十四、同步 Audit vs 异步 Audit

## 同步

```text
Request
 ↓
Business
 ↓
Audit DB
 ↓
Response
```

优点：

```text
Strong consistency
```

缺点：

```text
Higher latency
Audit DB failure impacts business
```

---

## 异步

```text
Request
 ↓
Business
 ↓
Kafka
 ↓
Response

Kafka
 ↓
Audit Consumer
 ↓
Audit DB
```

优点：

```text
Low latency
Decoupling
High throughput
```

缺点：

```text
Eventual consistency
```

企业系统通常会：

> **根据风险等级决定同步还是异步。**

---

# 十五、Critical Audit

不是所有操作的 Audit 等级都一样。

可以定义：

```text
LOW
MEDIUM
HIGH
CRITICAL
```

例如：

```text
READ_PROFILE
    ↓
LOW

UPDATE_PROFILE
    ↓
MEDIUM

DELETE_CUSTOMER
    ↓
HIGH

TRANSFER_MONEY
    ↓
CRITICAL
```

对于：

```text
CRITICAL
```

可以采用：

```text
Synchronous Audit
+
Durable Storage
+
Strong Integrity
+
Human Approval
```

---

# 十六、Audit Database

Audit Storage 可以采用不同技术。

## PostgreSQL

适合：

```text
Structured Audit
Strong Consistency
SQL Query
Compliance
```

例如：

```sql
CREATE TABLE audit_event (
    event_id UUID PRIMARY KEY,
    event_type VARCHAR(100),
    actor_id VARCHAR(100),
    actor_type VARCHAR(50),
    action VARCHAR(100),
    resource_type VARCHAR(100),
    resource_id VARCHAR(100),
    trace_id VARCHAR(100),
    result VARCHAR(50),
    created_at TIMESTAMP
);
```

---

# 十七、Elasticsearch

适合：

```text
Full Text Search
Large-scale Search
Operational Analytics
Fast Filtering
```

例如：

```text
actor:user001
action:TRANSFER
result:FAILED
```

可以快速查询。

---

# 十八、Object Storage

对于长期归档：

```text
S3
Object Storage
Data Lake
```

非常合适。

例如：

```text
2026/
 ├── 01/
 ├── 02/
 ├── 03/
 └── ...
```

Audit 数据可以：

```text
Hot
 ↓
Warm
 ↓
Cold
```

形成生命周期管理。

---

# 十九、Audit Data Lifecycle

企业 Audit 不应该无限保存。

可以设计：

```text
Hot Storage
0 ~ 30 days
```

用于快速查询。

```text
Warm Storage
30 ~ 365 days
```

用于审计调查。

```text
Cold Storage
1 ~ 7 years
```

用于合规归档。

最终：

```text
Application
 ↓
Kafka
 ↓
Hot DB
 ↓
Warm Storage
 ↓
Cold Archive
```

---

# 二十、Audit Data Integrity

Audit 最大的安全问题：

> **Audit Log 本身会不会被攻击者修改？**

例如：

```text
Hacker
 ↓
Delete audit record
```

如果攻击者可以修改：

```text
audit_log
```

那么整个审计系统失去意义。

因此 Audit Storage 必须：

```text
Append Only
```

而不是：

```text
UPDATE
DELETE
```

---

# 二十一、Hash Chain

一种经典方法是 Hash Chain。

第一个 Event：

```text
Hash1 = SHA256(Event1)
```

第二个：

```text
Hash2 = SHA256(Event2 + Hash1)
```

第三个：

```text
Hash3 = SHA256(Event3 + Hash2)
```

形成：

```text
Event1
  ↓
Hash1
  ↓
Event2
  ↓
Hash2
  ↓
Event3
  ↓
Hash3
```

如果有人修改：

```text
Event2
```

那么：

```text
Hash2
```

就发生变化。

因此可以检测：

> Audit 是否被篡改。

---

# 二十二、Digital Signature

对于更高安全等级，可以采用数字签名：

```text
Audit Event
      ↓
Hash
      ↓
Private Key
      ↓
Signature
```

验证：

```text
Audit Event
      ↓
Hash
      ↓
Public Key
      ↓
Verify Signature
```

这样可以实现：

> **Non-repudiation**

---

# 二十三、Tamper-Evident Audit

真正企业级 Audit 不一定需要区块链。

很多情况下：

```text
Append-only
+
Hash Chain
+
Digital Signature
+
Immutable Storage
```

已经可以提供非常强的完整性保证。

因此：

> Audit 不等于 Blockchain。

Blockchain 只是某些特殊场景下的一种技术选择。

---

# 二十四、Audit 和 Transaction

一个非常重要的问题：

```text
Business Transaction
+
Audit Transaction
```

假设：

```text
Payment SUCCESS
```

但是：

```text
Audit INSERT FAILED
```

怎么办？

如果使用：

```text
@Transactional
```

把业务和 Audit 放在同一个数据库：

```text
Business
+
Audit
```

可以获得强一致性。

但是微服务环境：

```text
Payment DB
Audit DB
```

很难使用传统数据库事务。

因此需要：

```text
Outbox Pattern
```

---

# 二十五、Audit Outbox Pattern

架构：

```text
Payment Service
      │
      ├── Payment Table
      │
      └── Outbox Table
              │
              ↓
          CDC / Polling
              │
              ↓
            Kafka
              │
              ↓
        Audit Service
```

Payment 和 Outbox：

```text
Same Database Transaction
```

因此：

```text
Payment Success
+
Audit Event Persisted
```

具有更强的一致性保证。

---

# 二十六、Audit Idempotency

Kafka Consumer 可能重复消费。

例如：

```text
Event A
 ↓
Consumer
 ↓
DB Insert
 ↓
ACK failed
 ↓
Kafka Retry
 ↓
DB Insert Again
```

因此 Audit 必须支持：

> Idempotency

例如：

```text
event_id UNIQUE
```

数据库：

```sql
CREATE UNIQUE INDEX idx_audit_event_id
ON audit_event(event_id);
```

重复事件：

```text
event_id = abc123
```

不会生成重复 Audit。

---

# 二十七、Audit Ordering

在分布式系统中：

```text
Event A
Event B
Event C
```

不一定按照顺序到达。

因此必须考虑：

```text
Event Time
Ingestion Time
Sequence Number
```

例如：

```json
{
  "eventId": "event-003",
  "sequence": 3,
  "eventTime": "...",
  "ingestionTime": "..."
}
```

对于关键业务，可以使用：

```text
Transaction ID
Aggregate ID
Sequence
```

恢复业务顺序。

---

# 二十八、Audit Query

Audit 查询通常不是：

```sql
SELECT *
FROM audit_event;
```

而是：

```text
By User
By Resource
By Action
By Time
By Trace
By Transaction
By Agent
By Risk
```

因此数据库索引必须设计好。

例如：

```sql
CREATE INDEX idx_audit_actor
ON audit_event(actor_id, created_at);

CREATE INDEX idx_audit_resource
ON audit_event(resource_id, created_at);

CREATE INDEX idx_audit_trace
ON audit_event(trace_id);

CREATE INDEX idx_audit_action
ON audit_event(action, created_at);
```

---

# 二十九、Audit 与 Security

Security Audit 是最典型的场景。

例如：

```text
Login
Logout
Password Change
Role Change
Permission Change
Token Creation
Token Revocation
```

尤其是：

```text
Privilege Escalation
```

必须重点审计。

例如：

```text
USER
 ↓
ROLE_CHANGE
 ↓
ADMIN
```

这是一个高风险 Audit Event。

---

# 三十、Audit 与 Data Access

现代数据治理中，Audit 最重要的问题之一：

> 谁访问了什么数据？

例如：

```text
Agent A
 ↓
Customer Data
 ↓
PII
```

Audit：

```json
{
  "actor": "agent-A",
  "action": "READ",
  "resource": "CUSTOMER",
  "classification": "PII"
}
```

进一步记录：

```text
Purpose
Policy
Authorization
Fields Accessed
```

形成：

> **Data Access Audit**

---

# 三十一、AI Agent Audit

进入 AI Agent 时代，传统 Audit 模型必须升级。

传统：

```text
User
 ↓
API
 ↓
Database
```

AI：

```text
User
 ↓
Prompt
 ↓
Agent
 ↓
LLM
 ↓
Reasoning
 ↓
Tool Selection
 ↓
Tool
 ↓
Service
 ↓
Database
```

因此必须审计：

```text
User
Prompt
Agent
Model
Tool
Policy
Data
Action
Result
```

---

# 三十二、AI Agent Audit Event

例如用户：

> “帮我把客户 A 的信用额度提高到 100 万。”

Agent 可能执行：

```text
Prompt
 ↓
Agent
 ↓
Customer Search
 ↓
Risk Check
 ↓
Policy Check
 ↓
Credit Update
```

Audit 应记录：

```json
{
  "user": "user001",
  "agent": "credit-agent",
  "model": "LLM",
  "action": "UPDATE_CREDIT_LIMIT",
  "resource": "customer001",
  "oldValue": 100000,
  "newValue": 1000000,
  "riskScore": 0.81,
  "policy": "CREDIT_HIGH_VALUE",
  "decision": "REVIEW",
  "humanApproval": "manager001"
}
```

这才是完整的 AI Audit。

---

# 三十三、不要直接保存完整 Prompt

AI Audit 一个重要问题：

> Prompt 能不能全部保存？

不能简单回答“全部保存”。

因为 Prompt 可能包含：

```text
PII
Passwords
Secrets
Financial Data
Confidential Information
```

因此更合理的是：

```text
Raw Prompt
     ↓
Sensitive Data Detection
     ↓
Masking
     ↓
Audit Record
```

例如：

```text
Original:
My SSN is 123-45-6789

Audit:
My SSN is ***-**-****
```

---

# 三十四、AI Audit 的 Trace

AI Agent 可以建立：

```text
Agent Trace
```

例如：

```text
Trace ID
│
├── User Request
│
├── Agent Planning
│
├── LLM Call
│
├── Tool: Search
│
├── Tool: Risk
│
├── Tool: Payment
│
└── Final Response
```

因此：

```text
Agent Trace
+
Audit Event
+
OpenTelemetry
```

可以形成完整的：

> **AI Behavioral Audit**

---

# 三十五、Audit Explainability

对于 AI 决策：

```text
Decision = BLOCK
```

远远不够。

应该回答：

```text
Who?
What?
Why?
Which Model?
Which Policy?
Which Data?
Which Risk?
Which Tool?
```

例如：

```text
Decision:
BLOCK

Reason:
High transaction risk

Risk:
0.93

Policy:
PAYMENT_HIGH_RISK

Agent:
payment-agent

Tool:
transfer-money
```

这就是：

> **Explainable Audit**

---

# 三十六、Audit + Risk Management

Audit 和 Risk Management 实际上是两个闭环：

```text
Risk
 ↓
Decision
 ↓
Action
 ↓
Audit
 ↓
Analysis
 ↓
Risk Model
 ↓
New Risk
```

因此：

```text
Risk Engine
      ↓
Decision
      ↓
Audit
      ↓
Analytics
      ↓
Risk Model
```

Audit 不只是“保存历史”。

它还可以成为：

> **Risk Management 的数据源。**

---

# 三十七、Audit + Fraud Detection

Audit Event 可以实时进入 Fraud Detection：

```text
Audit Event
 ↓
Kafka
 ↓
Fraud Detection
 ↓
Behavior Analysis
 ↓
Risk Score
```

例如：

```text
User A
 ↓
Login US
 ↓
5 minutes
 ↓
Login Singapore
 ↓
Transfer $100K
```

系统发现：

```text
Impossible Travel
```

风险上升：

```text
Risk = 0.95
```

最终：

```text
BLOCK
```

同时生成：

```text
Audit Event
```

---

# 三十八、Audit + SIEM

企业安全体系中，Audit Event 可以进入 SIEM：

```text
Application
Service
Agent
Database
Cloud
Network
      ↓
Audit Events
      ↓
SIEM
      ↓
Correlation
      ↓
Security Alert
```

例如：

```text
100 failed login
+
Privilege escalation
+
Sensitive data access
```

组合成：

```text
Potential Account Compromise
```

---

# 三十九、Audit Dashboard

一个企业级 Audit Dashboard 可以包含：

```text
Total Audit Events
Failed Actions
High Risk Actions
Privilege Changes
Sensitive Data Access
Agent Tool Calls
Policy Violations
Human Approvals
```

例如：

```text
Audit Events
──────────────────────
10,238,912

High Risk
──────────────────────
1,231

Blocked
──────────────────────
892

Human Review
──────────────────────
339

Policy Violations
──────────────────────
76
```

进一步提供：

```text
User Timeline
Agent Timeline
Transaction Timeline
Risk Timeline
```

---

# 四十、Audit Architecture

一个完整的 Enterprise Audit Architecture：

```text
                         Client
                           │
                           ↓
                    API Gateway
                           │
                           ↓
                    Business Service
                           │
                ┌──────────┴──────────┐
                │                     │
                ↓                     ↓
          Business Event          Audit Event
                                      │
                                      ↓
                                    Kafka
                                      │
                 ┌────────────────────┼──────────────────┐
                 ↓                    ↓                  ↓
             Audit DB           Elasticsearch       Object Storage
                 │                    │                  │
                 └────────────────────┼──────────────────┘
                                      ↓
                              Audit Query Service
                                      │
                    ┌─────────────────┼────────────────┐
                    ↓                 ↓                ↓
                Dashboard         Compliance        Security
```

如果进一步加入 AI：

```text
User
 ↓
Agent
 ↓
LLM
 ↓
Risk Engine
 ↓
Policy
 ↓
Tool
 ↓
Business Service
 ↓
Audit
 ↓
Kafka
 ↓
Audit Platform
```

---

# 四十一、Audit Service 应该如何设计？

可以把 Audit Service 独立成平台能力：

```text
Audit Service
│
├── Event Ingestion
├── Event Validation
├── Event Enrichment
├── Event Deduplication
├── Event Storage
├── Query
├── Export
├── Retention
├── Integrity Verification
└── Audit Analytics
```

业务系统只负责：

```text
publishAuditEvent(...)
```

而不是自己管理：

```text
Database
Index
Retention
Archive
Hash
Query
```

这样能够实现：

> **Audit as a Platform**

---

# 四十二、Java Audit SDK

对于 Spring Boot，可以提供统一 SDK：

```java
public interface AuditService {

    void audit(AuditEvent event);

}
```

业务代码：

```java
auditService.audit(
    AuditEvent.builder()
        .action("UPDATE_CUSTOMER")
        .actor("user001")
        .resource("customer123")
        .result("SUCCESS")
        .build()
);
```

甚至可以使用注解：

```java
@Audit(
    action = "UPDATE_CUSTOMER",
    resource = "CUSTOMER"
)
public void updateCustomer(...) {
}
```

通过 AOP 自动生成 Audit Event：

```text
Controller
   ↓
AOP
   ↓
Audit Interceptor
   ↓
Business Logic
```

---

# 四十三、Spring Boot Audit AOP

典型架构：

```text
@Audit
   ↓
Aspect
   ↓
Capture Context
   ↓
Business Method
   ↓
Capture Result
   ↓
Publish Audit Event
```

Context 可以来自：

```text
SecurityContext
RequestContext
TraceContext
MDC
```

最终：

```text
actorId
traceId
requestId
ip
userAgent
```

自动注入 Audit Event。

---

# 四十四、Audit 与 MDC

Java 微服务中可以通过 MDC：

```java
MDC.put("traceId", traceId);
MDC.put("requestId", requestId);
MDC.put("userId", userId);
```

Audit：

```java
AuditEvent.builder()
    .traceId(MDC.get("traceId"))
    .requestId(MDC.get("requestId"))
    .actorId(MDC.get("userId"))
    .build();
```

这样业务代码不需要反复传递：

```text
traceId
requestId
userId
```

---

# 四十五、Audit 性能设计

Audit 本质上是一个高吞吐系统。

假设：

```text
10,000 requests/sec
```

每个请求：

```text
2 Audit Events
```

那么：

```text
20,000 events/sec
```

一天：

```text
20,000 × 86,400
≈ 1.728 billion events
```

因此 Audit Architecture 必须考虑：

```text
Partition
Batch
Compression
Async Processing
Storage Tiering
Retention
Index Strategy
```

---

# 四十六、Kafka Partition

可以根据：

```text
tenantId
userId
resourceId
transactionId
```

进行 Partition。

例如：

```text
hash(tenantId) % N
```

这样可以保证：

```text
Same Tenant
 ↓
Same Partition
```

同时避免：

```text
Single Hot Partition
```

---

# 四十七、Audit Backpressure

如果：

```text
Business TPS ↑
```

但是：

```text
Audit Consumer TPS ↓
```

Kafka：

```text
Lag ↑
```

因此必须监控：

```text
Consumer Lag
Producer Latency
Storage Latency
Retry Count
Dead Letter Queue
```

Audit 系统不能因为下游存储缓慢导致整个业务系统雪崩。

---

# 四十八、Audit Failure Strategy

如果 Audit 服务挂了怎么办？

必须根据业务等级定义策略。

### Low Risk

```text
Business Continue
Audit Retry
```

### High Risk

```text
Business Pause
Audit Required
```

### Critical

```text
Fail Closed
```

例如：

```text
Payment
 ↓
Audit unavailable
 ↓
BLOCK
```

这就是：

> **Fail Open vs Fail Closed**

---

# 四十九、Audit Security

Audit 自己也必须安全。

必须考虑：

```text
Encryption at Rest
Encryption in Transit
Access Control
PII Masking
Secret Masking
Retention
Immutability
Key Management
```

尤其不能出现：

```text
password=123456
token=xxxx
creditCard=xxxx
```

出现在 Audit Log 中。

---

# 五十、Audit 的最终架构模型

可以把整个 Audit 技术体系总结为：

```text
                         AUDIT
                           │
        ┌──────────────────┼──────────────────┐
        ↓                  ↓                  ↓
      Event              Trail             Evidence
        │                  │                  │
        ↓                  ↓                  ↓
    Kafka/EventBus      Trace ID          Hash Chain
        │                  │                  │
        ↓                  ↓                  ↓
      Storage          Distributed        Integrity
                           │
                           ↓
                    ┌─────────────┐
                    │ AI / Agent  │
                    └──────┬──────┘
                           ↓
                   Model / Tool / Data
                           ↓
                        Audit
                           ↓
                    Risk Management
                           ↓
                      Governance
```

---

# 五十一、传统 Audit 到 AI Audit

整个技术演进可以总结为：

```text
第一阶段
Application Log
       ↓
谁调用了 API？

第二阶段
Audit Log
       ↓
谁做了什么？

第三阶段
Distributed Audit
       ↓
整个分布式链路发生了什么？

第四阶段
Security Audit
       ↓
谁访问了什么敏感资源？

第五阶段
AI Audit
       ↓
Agent 为什么做这个决定？

第六阶段
Autonomous Governance
       ↓
AI 是否应该被允许做这个决定？
```

因此未来的 Audit 不再只是：

> **记录过去发生了什么。**

而是：

> **建立企业数字行为的可信证据链。**

---

# 五十二、Audit 最重要的设计原则

如果让我从架构师角度总结现代 Audit，我会重点坚持以下十条原则：

### 1. Audit ≠ Application Log

日志解决诊断。

Audit 解决责任和证据。

### 2. Audit Event 必须结构化

不要依赖：

```text
String Log
```

应该使用：

```text
Structured Event
```

### 3. Audit 必须可追踪

至少关联：

```text
traceId
requestId
transactionId
```

### 4. Audit 必须考虑完整性

使用：

```text
Append-only
Hash
Signature
Immutable Storage
```

### 5. Audit 必须支持异步化

高吞吐场景：

```text
Kafka
```

是非常自然的选择。

### 6. Audit 必须支持幂等

```text
eventId UNIQUE
```

### 7. Audit 必须进行数据脱敏

尤其是：

```text
PII
Secret
Credential
Financial Data
```

### 8. Audit 必须支持生命周期管理

```text
Hot
Warm
Cold
Archive
```

### 9. AI Agent 必须审计 Delegation Chain

```text
User
 ↓
Agent
 ↓
Tool
 ↓
Service
 ↓
Resource
```

### 10. Audit 最终应该服务于 Governance

最终形成：

```text
Audit
 ↓
Risk
 ↓
Compliance
 ↓
Governance
```

---

# 五十三、结语

如果说：

```text
Authentication
```

回答的是：

> **你是谁？**

那么：

```text
Authorization
```

回答的是：

> **你能做什么？**

而：

```text
Risk Management
```

回答的是：

> **现在做这件事情风险有多大？**

那么：

```text
Audit
```

回答的就是：

> **到底发生了什么？谁做的？为什么做？经过了什么控制？最终结果是什么？**

在传统系统中，Audit 是一个基础设施能力。

在微服务时代，Audit 演变成 Distributed Audit。

在 Cloud Native 时代，Audit 与 Observability、Security、Compliance 深度融合。

而在 AI Agent 时代，Audit 将进一步升级为：

```text
User
 ↓
Intent
 ↓
Agent
 ↓
Model
 ↓
Reasoning
 ↓
Policy
 ↓
Risk
 ↓
Tool
 ↓
Action
 ↓
Result
```

完整记录这条链路。

最终，企业真正需要的不是简单的：

> **Audit Log System**

而是一套：

> **AI-native Audit & Governance Platform**

它应该让任何一次重要操作都能够被回答：

```text
Who?
What?
When?
Where?
Why?
Which Agent?
Which Model?
Which Data?
Which Tool?
Which Policy?
What Risk?
Who Approved?
What Result?
```

这才是现代企业 Audit 的真正价值：

> **让系统不仅能够执行，而且能够证明自己为什么这样执行。**