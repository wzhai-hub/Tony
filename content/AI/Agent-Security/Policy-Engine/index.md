---
title: Policy Engine：AI Agent 时代的策略决策与安全执行核心
# tags:
#   - nodejs
date: '2026-08-08'
summary: 输入一个安全决策请求，输出一个确定性的策略决策。
---
# Policy Engine：AI Agent 时代的策略决策与安全执行核心

> **摘要**
>
> 在传统企业系统中，权限控制通常由 RBAC、ABAC、ACL 等机制完成。但随着 AI Agent 从“生成内容”逐渐演变为“自主调用 Tool、访问数据、操作系统和执行现实世界动作”，传统权限模型开始暴露出明显局限。
>
> Agent 不仅需要回答“用户是谁”，还需要回答：
>
> * 这个 Agent 能否调用这个 Tool？
> * 当前用户是否有权让 Agent 执行这个操作？
> * Agent 能否访问这个 Resource？
> * 当前参数是否满足安全规则？
> * 当前数据是否允许流向目标系统？
> * 当前风险是否需要人工审批？
> * 当前调用是否违反租户、环境、时间或金额限制？
>
> 这些问题不能依赖 LLM 的 Prompt 来解决，而应该由一个独立、确定性、可审计的 **Policy Engine** 负责。
>
> 本文从 Policy Engine 的基本模型开始，深入分析 RBAC、ABAC、ReBAC、PBAC、Capability Security、Policy Decision Point、Policy Enforcement Point、Policy-as-Code、OPA/Rego、Cedar、规则编译、缓存、分布式一致性、AI Agent Tool Authorization、Risk-based Policy、Human-in-the-Loop、数据流控制以及企业级架构设计。

---

# 1. 什么是 Policy Engine

Policy Engine 可以简单理解为：

> **输入一个安全决策请求，输出一个确定性的策略决策。**

最简单的模型：

```text
Policy Engine

Input:
    Subject
    Action
    Resource
    Context

Output:
    ALLOW
    DENY
    REVIEW
```

例如：

```text
Subject:
    user-123

Action:
    customer.read

Resource:
    customer-456

Context:
    tenant = bank-a
```

Policy Engine：

```text
DENY
```

因为：

```text
user-123
没有访问
customer-456
的权限。
```

---

# 2. Policy Engine 与传统权限代码的区别

很多 Java 系统直接这样写：

```java
if (user.isAdmin()) {
    allow();
}
```

随着系统复杂化，代码逐渐变成：

```java
if (
    user.isAdmin()
    || (
        user.department.equals(resource.department)
        && resource.status.equals("ACTIVE")
        && currentTime.isBefore(deadline)
        && request.amount < 10000
    )
) {
    allow();
}
```

然后继续：

```java
if (tenant.equals(resource.tenant)
    && region.equals(resource.region)
    && environment.equals("production")
    && ...
)
```

最终：

> **Authorization Logic 开始污染业务代码。**

这就是 Policy Engine 存在的核心原因。

业务代码负责：

```text
Business Logic
```

Policy Engine 负责：

```text
Security Decision
```

---

# 3. Policy Engine 的核心职责

一个企业级 Policy Engine 通常负责：

```text
Authentication Context
        |
        v
Authorization
        |
        v
Policy Evaluation
        |
        +-- RBAC
        +-- ABAC
        +-- ReBAC
        +-- Capability
        +-- Risk
        +-- Data Policy
        +-- Network Policy
        +-- Tenant Policy
        +-- Compliance
        |
        v
Decision
```

它并不一定负责：

```text
Identity Authentication
Tool Execution
Database Execution
```

更典型的职责边界是：

```text
PDP
Policy Decision Point
```

而执行权限的系统叫：

```text
PEP
Policy Enforcement Point
```

---

# 4. PDP 与 PEP

这是理解 Policy Architecture 最重要的两个概念。

## PDP

Policy Decision Point：

```text
"Should this action be allowed?"
```

负责：

```text
Evaluate Policy
```

## PEP

Policy Enforcement Point：

```text
"Enforce the decision."
```

负责：

```text
ALLOW -> Execute
DENY  -> Block
```

完整流程：

```text
User
 |
 v
Agent
 |
 v
PEP
 |
 | authorization request
 v
PDP
 |
 | ALLOW / DENY / REVIEW
 v
PEP
 |
 v
Tool
```

例如：

```text
Agent
 |
 | call send_email
 v
Tool Gateway
       |
       | PEP
       v
Policy Engine
       |
       | DENY
       v
Tool Gateway
       |
       X
   Execution blocked
```

---

# 5. 为什么 Agent 特别需要 Policy Engine

传统应用：

```text
User
 |
 v
API
 |
 v
Business Logic
```

Agent：

```text
User
 |
 v
LLM
 |
 +-- Tool A
 +-- Tool B
 +-- Tool C
 +-- Tool D
```

LLM 具有：

```text
Non-determinism
Autonomy
Planning
Context dependence
Tool selection
```

因此不能把安全规则写成：

```text
System Prompt:

Never call delete_user.
```

因为：

```text
Prompt
    !=
Security Boundary
```

真正安全的架构应该：

```text
LLM
 |
 | "I want to call delete_user"
 v
Policy Engine
 |
 | DENY
 v
Tool Gateway
 |
 X
```

---

# 6. Policy Engine 的基本数学模型

可以将 Policy Decision 抽象成：

```text
Decision = P(S, A, R, C)
```

其中：

```text
S = Subject
A = Action
R = Resource
C = Context
```

例如：

```text
P(
    user-123,
    customer.read,
    customer-456,
    {
        tenant: bank-a,
        ip: 10.10.1.20,
        time: 10:30
    }
)
```

返回：

```text
DENY
```

更复杂的 Agent 场景：

```text
Decision =
P(
    User,
    Agent,
    Tool,
    Action,
    Resource,
    Data,
    Context,
    Risk
)
```

因此 Agent Policy 比传统 RBAC 丰富得多。

---

# 7. Policy Decision 不应该只有 Allow / Deny

传统：

```text
ALLOW
DENY
```

Agent 系统更适合：

```text
ALLOW
DENY
REVIEW
STEP_UP
REDACT
TRANSFORM
```

例如：

```text
send_email
```

如果发送内部邮件：

```text
ALLOW
```

如果发送外部邮件：

```text
REVIEW
```

如果包含 Secret：

```text
DENY
```

如果只是包含 PII：

```text
REDACT
```

例如：

```text
Original:

Customer SSN:
123-45-6789
```

Policy：

```text
REDACT
```

最终：

```text
Customer SSN:
***-**-6789
```

因此 Policy Engine 可以从：

> Authorization Engine

逐渐演化成：

> **Decision Engine**

---

# 8. RBAC：Role-Based Access Control

RBAC 是最经典的权限模型：

```text
User
 |
 v
Role
 |
 v
Permission
```

例如：

```text
Alice
 |
 +-- ADMIN
       |
       +-- user.read
       +-- user.write
       +-- user.delete
```

优点：

```text
简单
易理解
容易实现
容易管理
```

缺点：

```text
角色爆炸
上下文能力弱
资源粒度不足
不适合复杂 Agent
```

例如：

```text
customer-support-agent
```

不能简单地拥有：

```text
customer.read
```

因为还需要判断：

```text
哪个 customer？
哪个 tenant？
什么 purpose？
当前用户是否有权限？
```

---

# 9. Role Explosion

例如企业存在：

```text
Department
Region
Level
Tenant
Environment
```

组合以后：

```text
Engineer-US-Prod
Engineer-US-Dev
Engineer-EU-Prod
Engineer-EU-Dev
Manager-US-Prod
Manager-US-Dev
...
```

最终角色数量：

```text
RoleCount =
Department × Region × Level × Environment × Tenant
```

这就是典型：

> Role Explosion

因此现代 Policy Engine 通常会从：

```text
RBAC
```

扩展到：

```text
ABAC
```

---

# 10. ABAC：Attribute-Based Access Control

ABAC 不再只看：

```text
Role
```

而是看：

```text
Attributes
```

例如：

```text
Subject:
    department = finance
    level = manager

Resource:
    department = finance
    classification = confidential

Context:
    environment = production
```

Policy：

```text
ALLOW if:

subject.department == resource.department
AND
subject.level >= manager
AND
resource.classification <= confidential
```

ABAC 的核心优势：

> **权限来自属性，而不是角色本身。**

---

# 11. Agent 最适合 ABAC + RBAC

实际企业系统不应该简单选择：

```text
RBAC vs ABAC
```

而应该：

```text
RBAC
+
ABAC
```

例如：

```text
Role:
    customer-support

Attributes:
    tenant = bank-a
    region = US
    clearance = confidential
```

Policy：

```text
ALLOW customer.read

IF:
    role == customer-support
    AND
    resource.tenant == subject.tenant
    AND
    resource.region == subject.region
```

这样既保留：

```text
Role 管理简单
```

又具备：

```text
Attribute 灵活性
```

---

# 12. ReBAC：Relationship-Based Access Control

ReBAC 进一步回答：

> **Subject 与 Resource 是什么关系？**

例如：

```text
User
 |
 +-- member_of --> Team A
 |
 +-- owns -------> Project A
 |
 +-- manages ----> Department A
```

Resource：

```text
Project A
```

Policy：

```text
ALLOW
if user is member of project.team
```

这种模型非常适合：

```text
GitHub
Google Drive
Slack
Jira
CRM
Enterprise Knowledge Base
```

因为权限通常来自：

```text
User
  |
Relationship
  |
Resource
```

---

# 13. Agent + ReBAC

例如：

```text
Agent:
    customer-support-agent

User:
    Alice

Ticket:
    Ticket-123
```

关系：

```text
Alice
 |
assigned_to
 |
Ticket-123
```

Policy：

```text
ALLOW
if
    user.assigned_to(ticket)
```

Agent 可以代表 Alice 查询：

```text
Ticket-123
```

但不能查询：

```text
Ticket-999
```

这比：

```text
customer.read = true
```

安全得多。

---

# 14. Capability Security

在 Agent 世界中，Capability Security 非常重要。

一个 Capability 可以表示：

```text
Can perform action X
on resource Y
under constraints Z
```

例如：

```text
Capability:
    email.send

Resource:
    internal.company.com

Constraint:
    maxRecipients = 10
```

Agent 获得：

```text
Capability
```

而不是：

```text
Full Role
```

因此：

```text
Agent
 |
 +-- customer.read
 +-- ticket.create
 +-- email.send
```

这是 Agent 最自然的权限模型之一。

---

# 15. Policy Engine 的输入模型

企业级 Policy Request 可以定义：

```json
{
  "subject": {
    "id": "user-123",
    "type": "user",
    "roles": ["support"]
  },
  "agent": {
    "id": "support-agent",
    "version": "2.1"
  },
  "action": "customer.read",
  "resource": {
    "type": "customer",
    "id": "customer-456",
    "tenant": "bank-a"
  },
  "context": {
    "environment": "production",
    "ip": "10.0.1.10"
  }
}
```

Policy Engine：

```text
Request
   |
   v
Policy Evaluation
   |
   v
Decision
```

---

# 16. Decision Response

可以返回：

```json
{
  "decision": "ALLOW",
  "policy": "customer-read-v3",
  "reason": "same-tenant",
  "obligations": []
}
```

或者：

```json
{
  "decision": "REVIEW",
  "policy": "external-email-v2",
  "reason": "external-recipient",
  "obligations": [
    "human_approval"
  ]
}
```

或者：

```json
{
  "decision": "DENY",
  "policy": "secret-exfiltration-v1",
  "reason": "secret-to-external-destination"
}
```

这比单纯：

```text
true / false
```

更加适合企业系统。

---

# 17. Policy-as-Code

Policy Engine 最重要的技术思想之一：

> **Policy as Code**

不要：

```java
if (...) {
}
```

把所有权限逻辑硬编码在 Java。

而是：

```text
Policy
 |
 v
Version Control
 |
 v
Testing
 |
 v
Review
 |
 v
Deployment
```

例如：

```text
policies/
 ├── customer.rego
 ├── payment.rego
 ├── email.rego
 ├── tenant.rego
 └── data.rego
```

这样 Policy 可以：

```text
Git
Code Review
CI/CD
Versioning
Rollback
Testing
```

---

# 18. OPA：Open Policy Agent

OPA 是 Policy-as-Code 的经典实现。

架构：

```text
Application
    |
    | Query
    v
OPA
    |
    v
Policy
```

Policy 使用 Rego 表达。

例如：

```rego
package agent.authz

default allow = false

allow {
    input.subject.tenant == input.resource.tenant
    input.action == "customer.read"
}
```

Java 服务：

```text
Spring Boot
     |
     v
OPA
     |
     v
ALLOW / DENY
```

这样业务代码不需要知道具体权限规则。

---

# 19. Rego 的思想

Rego 不是传统：

```text
if / else
```

而是：

```text
Policy =
Data
+
Rules
```

例如：

```rego
allow {
    input.user.role == "admin"
}
```

或者：

```rego
allow {
    input.user.department == input.resource.department
    input.action == "read"
}
```

这种方式特别适合：

```text
Complex Authorization
Infrastructure Policy
Kubernetes Policy
API Authorization
Agent Security
```

---

# 20. Cedar：另一种现代 Policy Language

Cedar 的核心思想也是：

```text
Policy as Code
```

例如概念上：

```text
permit (
    principal,
    action == Action::"Read",
    resource
)
when {
    principal.department == resource.department
};
```

它强调：

```text
Formal Authorization Model
Predictable Evaluation
Typed Entities
```

因此现代企业可以根据场景选择：

```text
OPA / Rego
Cedar
自研 Policy DSL
```

---

# 21. Policy Engine 的内部架构

一个成熟 Policy Engine 内部可以分为：

```text
                Policy Engine
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
Policy Store    Compiler       Schema
       |             |             |
       +-------------+-------------+
                     |
                     v
               Evaluation Core
                     |
              +------+------+
              |             |
              v             v
          Decision       Explain
              |
              v
          Cache Layer
```

核心组件：

```text
Policy Store
Policy Loader
Policy Compiler
Schema Validator
Evaluation Engine
Decision Cache
Explain Engine
Audit
```

---

# 22. Policy Store

Policy 可以存储在：

```text
Git
Database
Object Storage
Config Service
Kubernetes ConfigMap
```

推荐：

```text
Git
 +
CI/CD
 +
Policy Distribution
```

例如：

```text
Git
 |
 v
Policy CI
 |
 +-- Syntax Check
 +-- Unit Test
 +-- Security Test
 |
 v
Policy Bundle
 |
 v
Policy Engine
```

---

# 23. Policy Compilation

如果每次请求都解析：

```text
YAML
JSON
Rego
DSL
```

性能会非常差。

因此：

```text
Policy Source
      |
      v
Parse
      |
      v
Compile
      |
      v
Optimized Representation
      |
      v
Runtime Evaluation
```

请求路径只做：

```text
Evaluate
```

而不是：

```text
Parse + Compile + Evaluate
```

---

# 24. Policy Evaluation 性能

在高并发系统中：

```text
API Request
```

可能达到：

```text
100k QPS
```

如果每次都：

```text
Network -> Policy Server -> DB
```

延迟可能非常高。

例如：

```text
Application
   |
   v
Policy Engine
   |
   v
Database
```

形成：

```text
P99 = 50ms
```

对于一个简单 API：

```text
Business logic = 10ms
Authorization = 50ms
```

明显不可接受。

---

# 25. Sidecar / Embedded Policy Engine

解决方案之一：

```text
Application
 |
 +-- Policy Sidecar
 |
 +-- Business Service
```

或者：

```text
Application
 |
 +-- Embedded Policy Engine
```

架构：

```text
              Service Pod
        +---------------------+
        |                     |
        |  Application        |
        |       |             |
        |       v             |
        |  Policy Engine      |
        |                     |
        +---------------------+
```

优点：

```text
低延迟
无网络 Hop
高吞吐
```

缺点：

```text
Policy Distribution
Memory
Version Consistency
```

需要解决。

---

# 26. Policy Distribution

假设有：

```text
1000
```

个微服务实例。

Policy 更新：

```text
v10 -> v11
```

需要：

```text
Policy Control Plane
        |
        v
Distribution
        |
 +------+------+------+
 |      |      |      |
 v      v      v      v
Pod1   Pod2   Pod3   PodN
```

常见方式：

```text
Polling
Push
Pub/Sub
Kafka
gRPC Streaming
Config Center
```

---

# 27. Policy Consistency

一个非常重要的问题：

```text
Pod A -> Policy v10
Pod B -> Policy v11
```

这时：

```text
同一个请求
```

可能：

```text
A -> ALLOW
B -> DENY
```

因此 Policy Engine 需要：

```text
Policy Version
```

例如：

```json
{
  "decision": "DENY",
  "policy_version": "2026.08.23.17"
}
```

这样审计时可以知道：

> **这个决策是在什么版本的 Policy 下产生的？**

---

# 28. Decision Cache

Policy Decision 很多时候具有高重复性。

例如：

```text
user-123
customer.read
customer-456
```

连续调用 100 次。

可以缓存：

```text
Cache Key =
subject
+
action
+
resource
+
context
+
policyVersion
```

例如：

```text
ALLOW
TTL = 30s
```

但必须注意：

> **Authorization Cache 最大的问题不是性能，而是权限撤销后的 stale decision。**

---

# 29. Cache Invalidation

例如：

```text
10:00
User has permission

10:01
Admin revokes permission

10:02
Cache still says ALLOW
```

于是：

```text
Security Violation
```

因此高风险权限：

```text
Payment
Delete
Production
Secret
```

不应该简单使用长 TTL。

可以采用：

```text
Risk-based TTL
```

例如：

```text
READ:
    60s

WRITE:
    10s

PAYMENT:
    0s

DELETE:
    0s
```

---

# 30. Explainability：为什么允许？

企业 Policy Engine 必须解决：

> Why was this request allowed?

例如：

```text
ALLOW
```

还需要：

```text
Policy:
    customer-read-v3

Reason:
    same tenant

Matched:
    subject.tenant == resource.tenant
```

这叫：

> Policy Decision Explanation

对于：

```text
Compliance
Audit
Security Investigation
Production Debugging
```

非常重要。

---

# 31. 为什么 Agent 更需要 Explainability

例如 Agent：

```text
Agent:
    customer-support-agent
```

突然调用：

```text
delete_customer
```

Security Team 必须回答：

```text
Who allowed this?
Which policy?
Which user?
Which agent version?
Which resource?
Which context?
```

所以一个完整的 Agent Decision Trace：

```text
User Request
     |
     v
Agent Reasoning
     |
     v
Tool Selection
     |
     v
Policy Request
     |
     v
Policy Decision
     |
     v
Tool Execution
```

必须全部可以关联。

---

# 32. Policy 与 Agent Tool Security

对于 Tool Security，可以定义：

```text
Tool Policy
```

例如：

```yaml
tool: send_email

allow:
  - agent: support-agent
    recipient_domain:
      - company.com

deny:
  - data.classification: SECRET

review:
  - recipient.external: true
```

于是：

```text
LLM
 |
 | send_email
 v
Policy Engine
 |
 +-- recipient
 +-- data
 +-- agent
 +-- user
 +-- risk
 |
 v
Decision
```

---

# 33. Risk-Based Policy

传统 Policy：

```text
ALLOW / DENY
```

Agent 更适合：

```text
Risk
```

例如：

```text
Risk = f(
    tool,
    data,
    user,
    resource,
    destination,
    amount,
    context
)
```

然后：

```text
Risk < 30
    ALLOW

30-70
    ALLOW + MONITOR

70-90
    REVIEW

> 90
    DENY
```

---

# 34. Policy 与 Risk Engine 的区别

两者不能混为一谈。

Policy Engine：

```text
"Is this allowed?"
```

Risk Engine：

```text
"How dangerous is this?"
```

因此：

```text
Risk Engine
     |
     v
Risk = 82
     |
     v
Policy Engine
     |
     v
REVIEW
```

架构：

```text
             Tool Request
                  |
        +---------+---------+
        |                   |
        v                   v
 Authorization         Risk Engine
        |                   |
        +---------+---------+
                  |
                  v
            Policy Engine
                  |
                  v
       ALLOW / DENY / REVIEW
```

---

# 35. Policy Obligation

Policy 不仅可以返回：

```text
ALLOW
```

还可以返回：

```text
Obligation
```

例如：

```json
{
  "decision": "ALLOW",
  "obligations": [
    "mask_pii",
    "audit",
    "limit_to_10_records"
  ]
}
```

于是：

```text
Tool Gateway
```

执行：

```text
Mask PII
+
Limit Results
+
Audit
```

这使 Policy Engine 从：

```text
Authorization
```

扩展到：

```text
Control Plane
```

---

# 36. Policy 与数据安全

例如：

```text
Resource:
    customer-profile

Classification:
    PII
```

Tool：

```text
send_email
```

Destination：

```text
external
```

Policy：

```text
IF
    data.classification == PII
AND
    destination == external

THEN
    DENY
```

或者：

```text
THEN
    REDACT
```

这种设计可以实现：

> Data-centric Security

而不是只做：

> API-centric Security

---

# 37. Policy Engine 与 Zero Trust

Zero Trust 的核心：

```text
Never Trust
Always Verify
```

Policy Engine 正是 Zero Trust 的决策核心。

每一次：

```text
Tool Call
API Call
Data Access
Production Operation
```

都应该：

```text
Authenticate
Authorize
Evaluate
Enforce
Audit
```

因此：

```text
Agent Security
+
Policy Engine
+
Zero Trust
```

天然适配。

---

# 38. Multi-Tenant Policy

SaaS / Enterprise Agent 必须支持：

```text
Tenant
```

例如：

```text
Tenant A
Tenant B
Tenant C
```

Policy：

```text
ALLOW
if
subject.tenant == resource.tenant
```

这是最基础的 Tenant Isolation。

更复杂：

```text
tenant.policy
```

每个租户可以拥有：

```text
Allowed Tools
Data Classification
External Domains
Approval Rules
Retention
Region
```

例如：

```yaml
tenant: bank-a

tools:
  allow:
    - customer.read
    - ticket.create

external_domains:
  allow: []

approval:
  payment: required
```

---

# 39. Policy Hierarchy

企业 Policy 经常存在多层：

```text
Global Policy
      |
      v
Organization Policy
      |
      v
Tenant Policy
      |
      v
Agent Policy
      |
      v
User Policy
```

最终：

```text
Effective Policy
```

可以定义：

```text
EffectivePolicy =
Global
∩
Organization
∩
Tenant
∩
Agent
∩
User
```

注意：

> 权限通常应该取交集，而不是并集。

这样可以防止下层策略绕过上层安全约束。

---

# 40. Policy Conflict

假设：

```text
Global:
    DENY external email

Tenant:
    ALLOW external email
```

最终应该：

```text
DENY
```

因为：

```text
Global Security Boundary
```

不能被 Tenant Policy 放宽。

可以定义 Policy Priority：

```text
Deny
    >
Allow
```

或者：

```text
Security Level
    >
Tenant Level
    >
Agent Level
```

---

# 41. Policy Evaluation Algorithm

一种典型流程：

```text
1. Load Subject
2. Load Agent
3. Load Resource
4. Load Action
5. Load Context
6. Load Data Classification
7. Evaluate Global Policy
8. Evaluate Tenant Policy
9. Evaluate Agent Policy
10. Evaluate User Policy
11. Evaluate Risk
12. Resolve Conflict
13. Generate Decision
14. Generate Obligations
15. Audit
```

最终：

```text
Decision
```

---

# 42. Policy Engine 的 Java 架构

对于 Spring Boot，可以设计：

```text
com.example.policy
|
+-- controller
|    +-- PolicyController
|
+-- model
|    +-- AuthorizationRequest
|    +-- Decision
|    +-- PolicyContext
|
+-- engine
|    +-- PolicyEngine
|    +-- RuleEvaluator
|    +-- ConflictResolver
|
+-- repository
|    +-- PolicyRepository
|
+-- cache
|    +-- DecisionCache
|
+-- audit
|    +-- PolicyAuditService
```

核心接口：

```java
public interface PolicyEngine {

    Decision evaluate(
        AuthorizationRequest request
    );
}
```

---

# 43. AuthorizationRequest

```java
public record AuthorizationRequest(
    Subject subject,
    AgentContext agent,
    String action,
    Resource resource,
    Map<String, Object> context
) {}
```

Decision：

```java
public record Decision(
    DecisionType type,
    String policyId,
    String reason,
    List<Obligation> obligations
) {}
```

其中：

```java
public enum DecisionType {
    ALLOW,
    DENY,
    REVIEW
}
```

---

# 44. Policy Evaluation Pipeline

可以采用责任链：

```java
public Decision evaluate(
        AuthorizationRequest request) {

    Decision global =
        globalPolicy.evaluate(request);

    if (global.isDenied()) {
        return global;
    }

    Decision tenant =
        tenantPolicy.evaluate(request);

    if (tenant.isDenied()) {
        return tenant;
    }

    Decision agent =
        agentPolicy.evaluate(request);

    if (agent.isDenied()) {
        return agent;
    }

    return userPolicy.evaluate(request);
}
```

但生产系统中不建议无限堆叠 `if`。

应该抽象成：

```text
Policy Evaluation Pipeline
```

---

# 45. Policy Rule Engine

可以设计：

```java
public interface PolicyRule {

    boolean matches(PolicyContext context);

    Decision evaluate(PolicyContext context);
}
```

例如：

```java
public class SameTenantRule
        implements PolicyRule {

    @Override
    public boolean matches(
            PolicyContext context) {

        return context.subject().tenant()
            .equals(context.resource().tenant());
    }

    @Override
    public Decision evaluate(
            PolicyContext context) {

        return Decision.allow(
            "same-tenant"
        );
    }
}
```

这样规则可以：

```text
组合
测试
替换
排序
版本化
```

---

# 46. Policy DSL

企业规模继续扩大后，可以设计 DSL：

```text
permit customer.read
when
    subject.tenant == resource.tenant
    and subject.role in ["support", "manager"]
    and context.environment != "sandbox"
```

或者：

```text
deny email.send
when
    data.classification == SECRET
```

DSL 的价值：

```text
Security Policy
```

可以由：

```text
Security Team
Platform Team
Compliance Team
```

共同维护，而不需要所有人修改 Java。

---

# 47. Policy Testing

Policy-as-Code 最大的优势之一是：

> Policy 可以像代码一样测试。

例如：

```text
Given:
    user.department = finance

And:
    resource.department = finance

When:
    action = read

Then:
    ALLOW
```

测试：

```text
Given:
    user.department = finance

And:
    resource.department = hr

Then:
    DENY
```

---

# 48. Property-Based Policy Testing

更高级的方式：

> 不只测试具体案例，而是测试安全属性。

例如：

```text
Property:

If subject.tenant != resource.tenant,
decision must never be ALLOW.
```

可以形式化：

```text
∀ request:
    request.subject.tenant
    !=
    request.resource.tenant

=> DENY
```

这对于：

```text
Multi-Tenant
Data Isolation
Financial Systems
```

非常重要。

---

# 49. Policy Mutation Testing

还可以故意修改 Policy：

```text
Original:

deny secret -> external
```

Mutation：

```text
allow secret -> external
```

然后：

```text
Security Test
```

必须失败。

如果测试仍然通过：

> Policy Test Suite 不够强。

这是非常值得企业安全平台采用的方法。

---

# 50. Policy Engine 的高可用设计

Policy Engine 如果挂掉：

```text
所有业务请求
```

可能受到影响。

因此需要：

```text
Policy Engine Cluster
```

例如：

```text
              Load Balancer
                    |
          +---------+---------+
          |         |         |
          v         v         v
       PDP-1     PDP-2     PDP-3
          |         |         |
          +---------+---------+
                    |
                Policy Store
```

Policy 本身尽可能：

```text
Read-only
Immutable
Locally Cached
```

这样即使：

```text
Policy Store
```

短暂不可用：

```text
PDP
```

仍然可以继续决策。

---

# 51. Fail Open vs Fail Closed

这是 Policy Engine 最关键的架构选择之一。

如果 Policy Engine 不可用：

```text
ALLOW
```

叫：

```text
Fail Open
```

如果：

```text
DENY
```

叫：

```text
Fail Closed
```

对于：

```text
Payment
Delete
Secret
Production
```

应该：

```text
Fail Closed
```

对于：

```text
Public Read
Search
Non-sensitive operation
```

可以根据业务考虑：

```text
Fail Open
```

因此更合理的是：

> **Risk-based Failure Mode**

---

# 52. Policy Engine 的性能目标

对于在线 Tool Authorization：

建议关注：

```text
P50
P95
P99
QPS
CPU
Memory
Cache Hit Rate
Policy Load Time
```

例如目标：

```text
P99 < 5ms
```

如果采用：

```text
Local PDP
Compiled Policy
In-memory Data
```

可以达到非常低的延迟。

但如果：

```text
Application
 -> Network
 -> PDP
 -> Database
```

延迟会明显增加。

---

# 53. Policy Engine 与 Redis

Redis 很适合：

```text
Decision Cache
Policy Metadata
Short-lived Authorization Context
Rate Limit
Revocation State
```

例如：

```text
policy:decision:
    user-123:
    customer.read:
    customer-456
```

但是：

> Redis 不能成为 Policy 的唯一 Source of Truth。

更推荐：

```text
Git / Policy Store
        |
        v
Policy Distribution
        |
        v
PDP Memory
        |
        v
Redis / Local Cache
```

---

# 54. Policy Engine 与 Kafka

Kafka 可以用于：

```text
Policy Update Event
```

例如：

```text
Policy Repository
      |
      v
Policy Updated
      |
      v
Kafka
      |
 +----+----+----+
 |    |    |    |
 v    v    v    v
PDP1 PDP2 PDP3 PDPN
```

每个 PDP 收到：

```text
policy-version = 2026.08.23.18
```

然后加载新 Policy。

这样可以构建：

> Event-driven Policy Distribution

---

# 55. Policy Engine 与 OpenTelemetry

Policy Engine 也应该被纳入 Trace。

例如：

```text
Trace
 |
 +-- agent.request
      |
      +-- tool.call
           |
           +-- policy.evaluate
           |
           +-- tool.execute
```

Policy Span：

```text
policy.id
policy.version
decision
reason
subject
action
resource
evaluation.time
```

注意：

> 不要把密码、Token、Secret、完整 PII 放入 Trace Attributes。

Observability 本身也必须遵守 Data Security Policy。

---

# 56. Policy Engine 的安全边界

Policy Engine 本身是高价值目标。

攻击者如果能够：

```text
Modify Policy
```

那么：

```text
整个安全系统
```

都可能被绕过。

因此必须保护：

```text
Policy Source
Policy Distribution
Policy Engine
Policy Cache
Policy Update API
```

特别是：

```text
Policy Update
```

必须具备：

```text
Authentication
Authorization
Approval
Versioning
Audit
Rollback
```

---

# 57. Policy Signing

进一步可以对 Policy Bundle 进行签名：

```text
Policy Source
    |
    v
Compile
    |
    v
Bundle
    |
    v
Sign
    |
    v
Deploy
```

PDP：

```text
Bundle
   |
   v
Verify Signature
   |
   v
Load
```

这样可以防止：

```text
Policy Supply Chain Attack
```

---

# 58. Policy Supply Chain Security

企业 Agent 平台可能依赖：

```text
Policy Package
Tool Package
MCP Server
Model
Prompt
Plugin
```

这些都形成：

```text
AI Supply Chain
```

Policy Engine 必须关注：

```text
Who created policy?
Who approved it?
Which version?
Who deployed it?
Was it modified?
```

这与传统：

```text
Software Supply Chain Security
```

非常类似。

---

# 59. Policy Engine 与 MCP

如果 Agent 使用 MCP Tool：

```text
Agent
 |
 v
MCP Client
 |
 v
MCP Server
 |
 v
Tool
```

推荐增加：

```text
Policy Gateway
```

变成：

```text
Agent
 |
 v
MCP Client
 |
 v
Policy Gateway
 |
 v
MCP Server
 |
 v
Tool
```

Policy Gateway 可以控制：

```text
Which MCP Server
Which Tool
Which Resource
Which User
Which Tenant
Which Data
```

因此：

> MCP 解决 Tool Communication，Policy Engine 解决 Tool Authorization。

---

# 60. 一个完整的 Agent Policy Decision

例如用户：

```text
帮我把客户 123 的资料发送到外部邮箱。
```

Agent：

```text
Tool:
    send_email
```

Policy Request：

```json
{
  "subject": "user-123",
  "agent": "support-agent",
  "action": "email.send",
  "resource": "customer-123",
  "destination": "external",
  "dataClassification": "PII"
}
```

Policy Engine：

```text
1. User has customer.read
2. Agent has email.send
3. Customer belongs to user tenant
4. Data = PII
5. Destination = external
6. External PII prohibited
```

最终：

```text
DENY
```

注意：

Agent 即使已经：

```text
成功读取客户资料
```

也不意味着：

```text
可以把资料发送到任何地方。
```

这就是：

> **Fine-grained Policy Enforcement**

---

# 61. Policy Engine 的最终架构

一个完整企业级架构可以设计成：

```text
                         User
                           |
                           v
                    +-------------+
                    |   Identity  |
                    +------+------+
                           |
                           v
                    +-------------+
                    | AI Agent    |
                    +------+------+
                           |
                           v
                     Tool Request
                           |
                           v
                +----------------------+
                |    Policy Gateway   |
                +----------------------+
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
      Identity         Policy PDP       Risk Engine
          |                |                |
          |          +-----+-----+          |
          |          |           |          |
          |          v           v          |
          |        RBAC        ABAC         |
          |          |           |          |
          +----------+-----------+----------+
                           |
                           v
                   Decision Resolver
                           |
              +------------+------------+
              |            |            |
              v            v            v
           ALLOW        REVIEW        DENY
              |
              v
          DLP / Obligation
              |
              v
          Tool Executor
              |
       +------+------+
       |             |
       v             v
   Enterprise     External
    Systems        Systems
              |
              v
            Audit
              |
              v
        OpenTelemetry
```

---

# 62. Policy Engine 最核心的设计思想

如果只记住几个概念，可以记住：

```text
Policy
=
Who
+
Can Do What
+
To What
+
Under Which Conditions
```

对于 Agent：

```text
Policy
=
User
+
Agent
+
Tool
+
Action
+
Resource
+
Data
+
Context
+
Risk
```

最终：

```text
Decision
=
ALLOW
|
DENY
|
REVIEW
|
OBLIGATION
```

---

# 63. Policy Engine 与传统 RBAC 的演进

整个权限技术可以看成：

```text
ACL
 |
 v
RBAC
 |
 v
ABAC
 |
 v
ReBAC
 |
 v
PBAC
 |
 v
Capability Security
 |
 v
Risk-based Authorization
 |
 v
Agent Policy Engine
```

未来 Agent Security 很可能不是某一种模型取代另一种模型，而是：

```text
RBAC
+
ABAC
+
ReBAC
+
Capability
+
Risk
+
Data Policy
```

统一进入：

> **Policy Decision Platform**

---

# 64. 从 Authorization Engine 到 Agent Policy Engine

传统 Authorization：

```text
Can user access resource?
```

Agent Policy：

```text
Can this user,
through this agent,
using this capability,
perform this action,
against this resource,
with this data,
under this context,
with this risk,
at this moment?
```

这就是两者最本质的区别。

---

# 65. 最终总结

Policy Engine 并不是简单的：

```text
if user.isAdmin()
```

它是企业 AI Agent 的：

```text
Decision Control Plane
```

它把：

```text
Identity
Authorization
RBAC
ABAC
ReBAC
Capability
Risk
Data Classification
Tenant Isolation
Human Approval
Audit
```

统一起来。

最重要的架构原则可以总结成：

```text
LLM decides intent.
        |
        v
Policy Engine decides permission.
        |
        v
Tool Gateway enforces decision.
        |
        v
Sandbox limits execution.
        |
        v
DLP controls information flow.
        |
        v
Audit records everything.
```

因此，真正成熟的 Agent 平台，不应该是：

```text
LLM
 +
Tools
```

而应该是：

```text
                    +----------------+
                    |      User      |
                    +-------+--------+
                            |
                            v
                    +---------------+
                    |      Agent    |
                    +-------+-------+
                            |
                            v
                  +-------------------+
                  |   Policy Engine   |
                  +-------------------+
                            |
                +-----------+-----------+
                |           |           |
                v           v           v
             Identity     Risk        DLP
                |           |           |
                +-----------+-----------+
                            |
                            v
                     Tool Gateway
                            |
                            v
                         Tools
                            |
                            v
                     Real World
```

最终可以用一句话概括：

> **Policy Engine 的价值，不是告诉 AI 应该做什么，而是在 AI 决定做什么之后，确定它究竟有没有资格做。**

而在 Agent 时代，这个“资格”已经不再只是传统的 User → Permission，而是 **User → Agent → Capability → Tool → Action → Resource → Data → Context → Risk** 的多维授权模型。

如果你接下来要继续深入 Agent Security，最值得展开的是 **① OPA/Rego 实战实现、② Agent Authorization + Capability Security、③ Policy Engine + Tool Gateway 企业级架构**，你更想先看哪一个？
