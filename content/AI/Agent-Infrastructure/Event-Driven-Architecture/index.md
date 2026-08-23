---
title: Event-Driven Architecture：从消息驱动到企业级事件驱动系统
# tags:
#   - nodejs
date: '2026-08-05'
summary: 系统组件不再主要通过“调用对方”进行协作，而是通过发布和消费事件进行协作。
---
# Event-Driven Architecture 深度技术博客：从消息驱动到企业级事件驱动系统

## 一、引言：为什么现代系统越来越需要 Event-Driven Architecture？

在传统企业系统中，我们最熟悉的是：

```text
User
  |
  v
API
  |
  v
Service
  |
  v
Database
```

服务之间通过：

```text
HTTP
REST
RPC
```

进行同步调用。

例如一个订单系统：

```text
Order Service
      |
      +----> Payment Service
      |
      +----> Inventory Service
      |
      +----> Notification Service
```

看起来非常简单。

但随着系统规模不断扩大，同步调用会逐渐暴露问题：

```text
Order
  |
  v
Payment
  |
  v
Inventory
  |
  v
Notification
```

如果 Notification Service 挂了：

```text
Order Service
      |
      v
Notification Service
      X
```

那么订单业务是否应该失败？

如果 Payment Service 响应很慢：

```text
Order
  |
  v
Payment
  |
  | 10 seconds
  |
  v
Response
```

整个请求链路都会被拖慢。

如果一个订单事件需要通知 20 个下游系统：

```text
Order Service
 |
 +---- Payment
 +---- Inventory
 +---- Shipping
 +---- Coupon
 +---- CRM
 +---- Analytics
 +---- Risk
 +---- Notification
 ...
```

系统最终会变成一个巨大的同步依赖图。

这正是：

> **Event-Driven Architecture（EDA，事件驱动架构）**

开始发挥价值的地方。

EDA 的核心思想可以概括为：

> **系统组件不再主要通过“调用对方”进行协作，而是通过发布和消费事件进行协作。**

---

# 二、什么是 Event-Driven Architecture？

Event-Driven Architecture 是一种以 **Event（事件）** 为核心通信机制的软件架构。

传统架构：

```text
Service A
   |
   | Request
   v
Service B
```

EDA：

```text
Service A
   |
   | Publish Event
   v
Event Broker
   |
   +----------+----------+----------+
   |          |          |          |
   v          v          v          v
Service B  Service C  Service D  Service E
```

例如：

```text
Order Created
```

不是：

```text
Order Service
   |
   +----> Payment Service
   +----> Inventory Service
   +----> Notification Service
```

而是：

```text
Order Service
      |
      | OrderCreated
      v
    Kafka
      |
 +----+------+----------+
 |           |          |
 v           v          v
Payment   Inventory Notification
```

Order Service 不需要知道：

```text
谁消费了这个事件？
```

它只负责：

> **产生事实。**

---

# 三、Event 到底是什么？

理解 EDA，首先要理解 Event。

Event 不是普通 Message。

例如：

```text
CreateOrder
```

更像 Command。

它表达：

> “请创建订单。”

而：

```text
OrderCreated
```

是 Event。

它表达：

> “订单已经创建完成。”

这是两个完全不同的概念。

可以总结：

```text
Command
    = 我要你做什么

Event
    = 某件事情已经发生
```

例如：

```text
Command:
CreateOrder

        ↓

Order Service

        ↓

Event:
OrderCreated
```

然后：

```text
OrderCreated
        |
        +---- Payment
        +---- Inventory
        +---- Notification
        +---- Analytics
```

这是事件驱动架构最核心的思想之一。

---

# 四、Event 的基本结构

一个企业级 Event 通常不仅仅包含业务数据。

例如：

```json
{
  "eventId": "evt-12345",
  "eventType": "OrderCreated",
  "timestamp": "2026-08-23T10:00:00Z",
  "source": "order-service",
  "version": "1.0",
  "traceId": "abc-123",
  "payload": {
    "orderId": "ORD-10001",
    "customerId": "C100",
    "amount": 299.99
  }
}
```

可以拆成：

```text
Event
 |
 +-- Event ID
 +-- Event Type
 +-- Timestamp
 +-- Source
 +-- Version
 +-- Correlation ID
 +-- Trace ID
 +-- Payload
```

其中几个字段尤其重要。

---

# 五、Event ID：事件唯一标识

例如：

```text
eventId = evt-12345
```

为什么需要？

因为消息系统中可能发生：

```text
Duplicate Delivery
```

例如：

```text
Producer
   |
   v
Kafka
   |
   v
Consumer
   |
   X
Network Failure
```

Consumer 已经处理成功，但 ACK 丢失。

Kafka 认为：

```text
Message not consumed
```

于是再次发送。

最终：

```text
Consumer
   |
   +---- Process Event
   |
   +---- Process Event again
```

所以：

> Event ID 是实现幂等消费的重要基础。

---

# 六、Event Type：事件类型

例如：

```text
OrderCreated
OrderPaid
OrderCancelled
OrderShipped
OrderCompleted
```

Event Type 应该表达：

> **已经发生的业务事实。**

而不是：

```text
CreateOrder
PayOrder
CancelOrder
```

后者更像 Command。

因此建议：

```text
OrderCreated
PaymentCompleted
UserRegistered
InvoiceGenerated
```

而不是：

```text
CreateOrder
PayPayment
RegisterUser
GenerateInvoice
```

---

# 七、Event Version：事件演进

事件一旦进入消息系统：

```text
Producer
   |
   v
Kafka
   |
   +---- Consumer A
   +---- Consumer B
   +---- Consumer C
```

Producer 不能随便修改 Schema。

例如 V1：

```json
{
  "orderId": "1001",
  "amount": 100
}
```

后来变成：

```json
{
  "orderId": "1001",
  "amount": 100,
  "currency": "USD"
}
```

这就是 Schema Evolution。

更复杂的变化：

```text
V1
 ↓
V2
 ↓
V3
```

因此企业 EDA 必须考虑：

```text
Schema Registry
Schema Compatibility
Backward Compatibility
Forward Compatibility
```

---

# 八、EDA 的核心角色

一个完整的 Event-Driven Architecture 通常包含：

```text
Producer
    |
    v
Event Broker
    |
    v
Consumer
```

进一步：

```text
+-------------+
|   Producer  |
+------+------+
       |
       v
+------+------+
| Event Bus   |
| / Broker    |
+------+------+
       |
  +----+----+----+
  |         |    |
  v         v    v
Consumer  Consumer Consumer
```

分别负责：

### Producer

产生事件。

### Event Broker

存储、路由和传输事件。

### Consumer

消费并处理事件。

---

# 九、Event Broker 是 EDA 的核心基础设施

常见 Event Broker：

```text
Kafka
RabbitMQ
Pulsar
AWS EventBridge
Google Pub/Sub
Azure Event Hubs
```

其中 Kafka 更偏向：

> Distributed Event Streaming Platform

而 RabbitMQ 更常用于：

> Message Queue / Work Queue

这两个概念不要完全混淆。

---

# 十、Event Queue 与 Event Stream

这是 EDA 中一个非常重要的区别。

## Queue

多个 Consumer：

```text
Queue
 |
 +---- Consumer A
 +---- Consumer B
 +---- Consumer C
```

一条消息通常由一个 Consumer Group 中的一个消费者处理。

适合：

```text
Task Processing
Work Queue
Background Job
```

---

## Event Stream

事件流：

```text
Event Stream
 |
 +---- Consumer Group A
 |
 +---- Consumer Group B
 |
 +---- Consumer Group C
```

每个 Consumer Group 都可以独立消费同一批事件。

例如：

```text
OrderCreated
       |
       v
     Kafka
       |
 +-----+-----+---------+
 |           |         |
 v           v         v
Payment    Analytics  CRM
Group      Group      Group
```

这正是 Kafka 在事件驱动架构中非常强大的原因。

---

# 十一、Publish/Subscribe

EDA 最典型的模式是：

> Publish / Subscribe

Producer：

```text
Order Service
```

发布：

```text
OrderCreated
```

Broker：

```text
Kafka Topic
```

多个消费者：

```text
Payment
Inventory
Notification
Analytics
```

结构：

```text
             OrderCreated
                  |
                  v
                Kafka
                  |
       +----------+----------+
       |          |          |
       v          v          v
    Payment    Inventory  Analytics
```

Producer 完全不需要知道：

```text
Consumer 是谁？
```

这就是：

> **Loose Coupling**

---

# 十二、EDA 最大的价值：解耦

传统：

```text
A → B
```

A 必须知道：

```text
B 的地址
B 的 API
B 的协议
B 的 SLA
B 是否可用
```

EDA：

```text
A → Event Bus
```

A 只需要知道：

```text
Event Schema
```

因此：

```text
Service A
    |
    v
Event
```

和：

```text
Service B
Service C
Service D
```

形成松耦合关系。

这意味着：

> Producer 可以独立演进。

---

# 十三、Temporal Decoupling

EDA 不仅解决空间解耦：

```text
Who calls whom?
```

还解决时间解耦：

```text
When does consumer process?
```

例如：

```text
Order Service
      |
      v
Kafka
```

此时：

```text
Notification Service
```

暂时不可用。

事件仍然保存在：

```text
Kafka
```

等 Notification Service 恢复：

```text
Kafka
   |
   v
Notification Service
```

继续处理。

这就是：

> **Temporal Decoupling**

---

# 十四、EDA 与同步调用的本质区别

同步：

```text
A
|
| HTTP
v
B
|
| HTTP
v
C
```

A 必须等待 B。

B 必须等待 C。

最终：

```text
Latency
=
A + B + C
```

EDA：

```text
A
|
| Event
v
Broker
|
+---- B
+---- C
+---- D
```

A 不需要等待所有消费者完成。

因此：

```text
Request Latency
```

与：

```text
Downstream Processing
```

可以解耦。

---

# 十五、EDA 不是“所有事情都异步”

这是使用 EDA 最容易产生的误区。

并不是：

> “用了 Kafka，就应该全部异步。”

例如：

```text
Get Account Balance
```

通常还是：

```text
Client
 ↓
Account Service
 ↓
Response
```

因为它是：

> Query

而：

```text
OrderCreated
PaymentCompleted
UserRegistered
```

非常适合：

> Event

因此更合理的架构是：

```text
Query
 ↓
Sync API

Command
 ↓
Service

Event
 ↓
Event Bus
```

---

# 十六、Command、Event、Query 三者关系

可以使用经典的：

```text
CQRS
```

思想理解。

```text
Command
    |
    v
Command Handler
    |
    v
Domain Model
    |
    v
Event
    |
    v
Event Bus
```

Query：

```text
Query
 ↓
Read Model
 ↓
Response
```

最终：

```text
Command → State Change → Event
Query   → Read Model
```

这就是现代 Event-Driven Architecture 与 CQRS 经常结合的原因。

---

# 十七、Event-Driven Architecture 与 DDD

EDA 与领域驱动设计有天然联系。

例如订单领域：

```text
Order
```

产生：

```text
OrderCreated
OrderPaid
OrderCancelled
OrderShipped
```

这些事件其实就是：

> Domain Events

因此可以形成：

```text
Domain Model
      |
      v
Domain Event
      |
      v
Event Bus
      |
 +----+----+----+
 |         |    |
Billing   CRM  Shipping
```

这就是：

> **Domain-Driven Event Architecture**

---

# 十八、Domain Event 与 Integration Event

这两个概念也需要区分。

Domain Event：

```text
OrderCreated
```

主要存在于：

```text
Order Domain
```

Integration Event：

```text
OrderCreatedIntegrationEvent
```

用于：

```text
跨服务通信
```

为什么需要区分？

因为 Domain Model 不应该被外部系统完全绑死。

例如：

```text
Order Domain
   |
   v
Domain Event
   |
   v
Event Publisher
   |
   v
Integration Event
```

这样可以避免：

> 内部领域模型直接暴露给外部系统。

---

# 十九、Event Storming

在设计复杂 Event-Driven System 时，一个非常有价值的方法是：

> Event Storming

例如电商系统：

```text
UserRegistered
      ↓
CartCreated
      ↓
OrderCreated
      ↓
PaymentCompleted
      ↓
InventoryReserved
      ↓
OrderShipped
      ↓
OrderCompleted
```

把业务流程从：

```text
API
Database
Table
```

转换成：

```text
Business Event
```

这样更容易发现：

```text
Bounded Context
Aggregate
Domain Event
Command
Policy
```

---

# 二十、Event Chain

一个业务动作可能触发一系列事件：

```text
OrderCreated
     |
     v
PaymentRequested
     |
     v
PaymentCompleted
     |
     v
InventoryReserved
     |
     v
ShipmentCreated
     |
     v
OrderCompleted
```

这种模式称为：

> Event Chaining

优点：

```text
Loose Coupling
```

但问题也非常明显：

```text
Debugging
Tracing
Ordering
Failure Handling
```

最终可能变成：

```text
A
 ↓
B
 ↓
C
 ↓
D
 ↓
E
```

工程师很难知道：

> 为什么 E 没有发生？

因此 EDA 必须配合 Observability。

---

# 二十一、Event Choreography 与 Orchestration

这是 EDA 中非常重要的架构选择。

## Choreography

没有中央协调者。

```text
Order
 ↓
OrderCreated
 ↓
Payment
 ↓
PaymentCompleted
 ↓
Inventory
 ↓
InventoryReserved
```

每个服务自己响应事件。

优点：

```text
Decoupling
```

缺点：

```text
业务流程难以理解
```

---

## Orchestration

存在一个中央 Orchestrator：

```text
              Order Workflow
                    |
          +---------+---------+
          |         |         |
          v         v         v
       Payment  Inventory  Shipping
```

Orchestrator 负责：

```text
下一步是什么？
失败怎么办？
重试怎么办？
补偿怎么办？
```

对于复杂业务流程：

> Orchestration 往往更加容易治理。

---

# 二十二、EDA 与 Saga Pattern

跨多个微服务的业务事务：

```text
Order
Payment
Inventory
Shipping
```

很难使用：

```text
Distributed Transaction
```

因此可以使用：

> Saga

例如：

```text
Create Order
    ↓
Payment
    ↓
Reserve Inventory
    ↓
Create Shipment
```

如果库存失败：

```text
Cancel Inventory
      ↓
Refund Payment
      ↓
Cancel Order
```

形成：

```text
Forward Transactions
        +
Compensating Transactions
```

Saga 可以通过：

```text
Event Choreography
```

或者：

```text
Saga Orchestrator
```

实现。

---

# 二十三、Eventual Consistency

EDA 的另一个核心概念：

> 最终一致性。

例如：

```text
Order DB
Payment DB
Inventory DB
```

不再通过一个全局事务保证：

```text
ACID
```

而是：

```text
OrderCreated
      ↓
Payment
      ↓
PaymentCompleted
      ↓
Inventory
```

短时间内：

```text
Order = Created
Payment = Pending
Inventory = Available
```

这是正常的。

最终：

```text
Order = Paid
Payment = Completed
Inventory = Reserved
```

因此 EDA 通常接受：

> **Eventual Consistency**

---

# 二十四、Outbox Pattern

EDA 中一个非常经典的问题：

```text
Database
+
Kafka
```

例如：

```text
Order Service

DB Transaction:
    INSERT Order

Kafka:
    publish OrderCreated
```

如果：

```text
DB 成功
Kafka 失败
```

就出现：

```text
Order Created
但
OrderCreated Event 丢失
```

反过来也一样。

如何解决？

> Outbox Pattern。

---

# 二十五、Outbox Pattern 的核心思想

不要直接：

```text
Business DB
    +
Kafka
```

而是：

```text
Database Transaction
       |
       +---- Order
       |
       +---- Outbox Event
```

例如：

```text
BEGIN TRANSACTION

INSERT INTO orders ...

INSERT INTO outbox_events ...

COMMIT
```

然后：

```text
Outbox Publisher
      |
      v
Kafka
```

结构：

```text
              Order Service
                   |
           +-------+-------+
           |               |
           v               v
       Order DB       Outbox Table
                           |
                           v
                     Event Publisher
                           |
                           v
                         Kafka
```

这样：

```text
Order
```

和：

```text
Event
```

可以在同一个数据库事务中保持一致。

---

# 二十六、CDC：进一步演进 Outbox

Outbox Publisher 可以自己轮询：

```text
SELECT *
FROM outbox_events
WHERE status = 'NEW'
```

也可以使用：

> CDC（Change Data Capture）

例如：

```text
Database
   |
   v
Debezium
   |
   v
Kafka
```

架构：

```text
Order DB
   |
   | WAL / Binlog
   v
Debezium
   |
   v
Kafka
```

这在大型企业系统中非常常见。

---

# 二十七、Exactly-Once 是不是必须？

EDA 中经常讨论：

```text
At Most Once
At Least Once
Exactly Once
```

### At Most Once

最多一次：

```text
Send
 ↓
No Retry
```

可能丢消息。

### At Least Once

至少一次：

```text
Retry
```

可能重复。

### Exactly Once

理论上：

```text
Exactly one processing
```

但在复杂分布式系统中，真正做到端到端 Exactly Once 非常困难。

因此企业系统通常更关注：

> **At-Least-Once Delivery + Idempotent Consumer**

---

# 二十八、Idempotent Consumer

假设：

```text
OrderPaid
```

重复消费：

```text
Consumer
 |
 +---- Process
 |
 +---- Process again
```

如果业务：

```text
UPDATE balance
SET balance = balance - 100
```

可能扣款两次。

所以 Consumer 应该设计成：

```text
Event
 ↓
Check Event ID
 ↓
Already processed?
 ├── Yes → Ignore
 └── No
      ↓
   Process
      ↓
   Save Event ID
```

例如：

```text
processed_events
----------------
event_id
consumer
processed_at
```

---

# 二十九、Kafka Partition 与事件顺序

Kafka 中一个非常重要的概念：

> Partition。

例如：

```text
Topic: order-events

Partition 0
Partition 1
Partition 2
```

Kafka 可以保证：

> **同一个 Partition 内的消息有序。**

但不能天然保证：

```text
整个 Topic 全局有序
```

因此订单系统经常使用：

```text
key = orderId
```

这样：

```text
Order A
 ↓
Partition 1

Order B
 ↓
Partition 2
```

同一个订单：

```text
Order A Created
Order A Paid
Order A Shipped
```

都会进入同一个 Partition。

从而保持：

```text
Created
 ↓
Paid
 ↓
Shipped
```

顺序。

---

# 三十、Consumer Group

Kafka Consumer Group 是 EDA 设计的核心机制。

例如：

```text
Topic
 |
 +-----------------------------+
 |                             |
Payment Group              Analytics Group
 |                             |
 +-- Consumer 1                +-- Consumer 1
 +-- Consumer 2                +-- Consumer 2
```

不同 Group：

```text
独立消费
```

同一个 Group：

```text
负载均衡
```

因此：

```text
Payment
Inventory
Analytics
```

可以拥有不同的 Consumer Group。

---

# 三十一、Backpressure

假设 Producer：

```text
10,000 events/sec
```

Consumer：

```text
2,000 events/sec
```

那么：

```text
Lag
 ↑
 ↑
 ↑
```

消息不断堆积。

这就是：

> Backpressure Problem

Runtime 必须考虑：

```text
Consumer Lag
Queue Depth
Processing Rate
Retry Rate
```

可以通过：

```text
Horizontal Scaling
Partition Scaling
Batch Consumption
Rate Limiting
Load Shedding
```

解决。

---

# 三十二、Dead Letter Queue

消息可能永远无法成功处理：

```text
Invalid Schema
Business Error
Poison Message
Data Corruption
```

不能无限 Retry：

```text
Retry
Retry
Retry
Retry
...
```

应该：

```text
Main Topic
    |
    v
Consumer
    |
 failure
    v
Retry Topic
    |
 failure
    v
DLQ
```

即：

> Dead Letter Queue

例如：

```text
order-events
      |
      v
order-consumer
      |
      v
retry-topic
      |
      v
dead-letter-topic
```

---

# 三十三、Retry Strategy

Retry 不应该简单：

```text
retry 3 times
```

更合理：

```text
Immediate Retry
      ↓
1s
      ↓
5s
      ↓
30s
      ↓
5min
      ↓
DLQ
```

即：

> Exponential Backoff

同时需要：

```text
Jitter
```

避免大量 Consumer 同时重试。

---

# 三十四、Poison Message

假设一条消息：

```text
Event ID = E100
```

永远无法解析。

如果 Consumer：

```text
Retry
Retry
Retry
```

那么整个 Partition 可能被阻塞。

这就是：

> Poison Message

因此需要：

```text
Retry Limit
DLQ
Error Classification
```

例如：

```text
Transient Error
    → Retry

Permanent Error
    → DLQ
```

这是生产级 EDA 的基本能力。

---

# 三十五、Event Schema Governance

大型企业中可能有：

```text
500+ Services
5000+ Events
```

如果每个团队自己定义：

```text
OrderCreated
order_created
Order-Created
OrderCreatedEvent
```

系统最终会失控。

因此需要：

> Event Governance

包括：

```text
Naming
Schema
Version
Ownership
Compatibility
Documentation
Retention
Security
```

例如：

```text
com.company.order.v1.OrderCreated
```

---

# 三十六、Event Contract

Producer 和 Consumer 之间真正共享的是：

> Event Contract

例如：

```json
{
  "eventType": "OrderCreated",
  "version": 1,
  "data": {
    "orderId": "10001",
    "customerId": "C100",
    "amount": 99.9
  }
}
```

Consumer 不应该依赖：

```text
Producer 的数据库表
```

而应该依赖：

```text
Event Contract
```

这也是微服务真正解耦的重要基础。

---

# 三十七、Event Sourcing

EDA 还有一个非常重要的高级概念：

> Event Sourcing

传统：

```text
Current State
```

例如：

```text
Account Balance = 1000
```

Event Sourcing：

```text
AccountCreated
Deposit 500
Withdraw 200
Deposit 700
```

当前状态：

```text
1000
```

由事件重建。

即：

```text
Events
  |
  v
State Projection
  |
  v
Current State
```

---

# 三十八、Event Sourcing 与普通 EDA 的区别

这两个概念经常被混淆。

普通 EDA：

```text
State
  |
  v
Event
  |
  v
Consumers
```

Event Sourcing：

```text
Event Store
  |
  +---- Event 1
  +---- Event 2
  +---- Event 3
  +---- Event 4
        |
        v
   Reconstruct State
```

所以：

> Event-driven 不等于 Event Sourcing。

Event Sourcing 是 EDA 的一种高级数据架构模式。

---

# 三十九、CQRS + Event Sourcing

三者经常组合：

```text
Command
   |
   v
Aggregate
   |
   v
Event Store
   |
   v
Events
   |
   +----------+
   |          |
   v          v
Read Model   Other Services
```

Query：

```text
Client
 ↓
Query
 ↓
Read Model
```

这形成：

> CQRS + Event Sourcing + EDA

适合：

```text
Financial System
Order System
Audit System
Complex Domain
```

---

# 四十、Event Replay

Event Sourcing 一个非常强的能力：

> Replay

例如：

```text
Event 1
Event 2
Event 3
...
Event 1,000,000
```

可以重新构建：

```text
Read Model
```

例如：

```text
Events
   |
   v
Projection V1
```

后来业务逻辑改变：

```text
Projection V2
```

重新 Replay：

```text
Events
   |
   v
Projection V2
```

不需要修改原始业务数据。

---

# 四十一、EDA 的 Observability

事件驱动系统最大的挑战之一：

> 请求链路不再是同步的。

传统：

```text
HTTP Request
 ↓
Service A
 ↓
Service B
 ↓
Service C
```

很容易追踪。

EDA：

```text
Request
 ↓
Service A
 ↓
Kafka
 ↓
Service B
 ↓
Kafka
 ↓
Service C
 ↓
Kafka
 ↓
Service D
```

因此必须使用：

```text
traceId
correlationId
eventId
causationId
```

把整个事件链串起来。

---

# 四十二、Correlation ID 与 Causation ID

例如：

```text
User Request
    |
    v
OrderCreated
    |
    v
PaymentCompleted
    |
    v
InventoryReserved
```

可以：

```text
correlationId = ORDER-1001
```

表示：

> 这些事件属于同一个业务流程。

而：

```text
causationId
```

表示：

> 当前事件由哪个事件触发。

形成：

```text
OrderCreated
     |
     | causationId
     v
PaymentRequested
     |
     v
PaymentCompleted
```

这样可以建立：

> Event Causality Graph

---

# 四十三、EDA 的安全问题

Event Bus 不是一个天然安全的系统。

必须考虑：

```text
Authentication
Authorization
Encryption
Data Privacy
Schema Validation
Tenant Isolation
Audit
```

例如：

```text
Payment Event
```

不能随意包含：

```text
Credit Card Number
Password
Sensitive PII
```

Event 一旦进入 Kafka：

```text
Retention = 7 days
```

意味着敏感数据可能长期存在。

因此：

> Event Schema 设计本身就是 Security Design。

---

# 四十四、Event Retention

Kafka 中事件可以保留：

```text
1 day
7 days
30 days
1 year
```

Retention 的设计需要考虑：

```text
Storage Cost
Replay
Compliance
Audit
Business Requirement
```

例如：

```text
Operational Event
    → 7 days

Audit Event
    → 1 year

Financial Event
    → 根据监管要求
```

不要默认：

> 所有 Event 永久保存。

---

# 四十五、EDA 的性能模型

假设：

```text
Producer = 100,000 events/sec
```

Event Broker：

```text
Kafka
```

可以：

```text
Partition
Partition
Partition
...
```

进行水平扩展。

Consumer：

```text
Consumer Group
  |
  +---- Consumer 1
  +---- Consumer 2
  +---- Consumer 3
  +---- Consumer 4
```

通过增加：

```text
Partitions
Consumers
```

实现吞吐量扩展。

因此：

> EDA 非常适合高吞吐、异步、流式处理场景。

---

# 四十六、EDA 的典型应用场景

EDA 非常适合：

### 1. 电商

```text
OrderCreated
PaymentCompleted
InventoryReserved
ShipmentCreated
```

### 2. 金融

```text
TransactionCreated
PaymentAuthorized
FraudDetected
SettlementCompleted
```

### 3. IoT

```text
DeviceConnected
TemperatureChanged
DeviceOffline
AlertTriggered
```

### 4. 日志与监控

```text
LogCreated
MetricGenerated
AlertTriggered
IncidentCreated
```

### 5. 数据平台

```text
DataCreated
DataUpdated
DataProcessed
DataIndexed
```

### 6. AI / Agent

```text
AgentTaskCreated
ToolCalled
ToolCompleted
AgentCompleted
```

这也是 EDA 与 Agent Runtime 结合的重要方向。

---

# 四十七、EDA + Agent Runtime

未来 Agent 系统很可能天然采用事件驱动架构。

例如：

```text
Agent Task Created
        |
        v
Agent Runtime
        |
        v
Task Event
        |
        +---- Tool Agent
        |
        +---- Research Agent
        |
        +---- Security Agent
```

Agent Runtime 可以产生：

```text
AgentStarted
AgentThinking
ToolCalled
ToolCompleted
AgentWaiting
AgentCompleted
AgentFailed
```

然后进入：

```text
Kafka
```

其他系统可以订阅：

```text
Observability
Audit
Billing
Analytics
Notification
```

---

# 四十八、EDA + A2A + MCP

把前面学习的 Agent 技术结合起来：

```text
                       Agent Runtime
                             |
              +--------------+--------------+
              |              |              |
             A2A            MCP           Event
              |              |              |
              v              v              v
           Agent B         Tools          Kafka
```

三个机制分别解决：

```text
A2A
Agent ↔ Agent

MCP
Agent ↔ Tool

Event
Agent ↔ Event Ecosystem
```

最终：

```text
                 Agent Platform
                       |
       +---------------+---------------+
       |               |               |
      A2A             MCP             EDA
       |               |               |
    Agents           Tools          Events
```

这是未来企业级 Agent Platform 非常值得关注的架构组合。

---

# 四十九、Agent Event Architecture

可以设计：

```text
Agent Runtime
      |
      v
Kafka
      |
 +----+------------------+
 |                       |
 v                       v
Observability          Agent Analytics
 |
 +---- Audit
 |
 +---- Billing
 |
 +---- Monitoring
```

例如 Agent Runtime 发布：

```json
{
  "eventType": "AgentToolCalled",
  "agentId": "security-agent",
  "taskId": "task-1001",
  "tool": "vulnerability-scan",
  "timestamp": "...",
  "traceId": "..."
}
```

然后：

```text
Billing Service
```

计算：

```text
Tool Cost
```

而：

```text
Observability Service
```

计算：

```text
Tool Latency
```

这就是：

> **Event-Driven Agent Platform**

---

# 五十、EDA 的优势

总结一下 EDA 的核心优势。

## Loose Coupling

```text
Producer
   |
 Event
   |
Consumers
```

Producer 不依赖 Consumer。

---

## Scalability

通过：

```text
Partitions
Consumer Groups
Horizontal Scaling
```

支持高吞吐。

---

## Resilience

Consumer 暂时不可用：

```text
Event
 ↓
Broker
 ↓
Later Process
```

---

## Extensibility

增加新的 Consumer：

```text
OrderCreated
     |
 +---+---+---+
 |   |   |   |
 A   B   C   D
```

不需要修改 Producer。

---

## Replay

事件保留后可以：

```text
Replay
Reprocess
Rebuild
```

这是传统同步 API 很难做到的。

---

# 五十一、EDA 的缺点

EDA 不是银弹。

它也会增加系统复杂度。

## 1. Debugging Complexity

```text
A
 ↓
Kafka
 ↓
B
 ↓
Kafka
 ↓
C
```

问题定位比同步调用困难。

---

## 2. Eventual Consistency

系统不是立即一致。

---

## 3. Operational Complexity

需要维护：

```text
Kafka
Schema Registry
Consumer
DLQ
Monitoring
Replay
```

---

## 4. Ordering

跨 Partition 的全局顺序很难保证。

---

## 5. Duplicate

At-least-once 消费通常意味着：

```text
Duplicate Events
```

必须设计幂等。

---

# 五十二、什么时候应该使用 EDA？

非常适合：

```text
High Throughput
Async Processing
Loose Coupling
Event Streaming
Distributed Systems
Integration
Audit
Analytics
Workflow
Agent Systems
```

不一定适合：

```text
Simple CRUD
Simple Query
Strong Immediate Consistency
Very Small Application
```

不要为了：

> “看起来很高级”

而引入 Kafka。

架构应该由业务特征驱动。

---

# 五十三、企业级 EDA 的完整架构

一个成熟企业级 EDA 平台可以设计成：

```text
                           Client
                              |
                              v
                       API Gateway
                              |
                 +------------+------------+
                 |                         |
              Sync API                  Command
                 |                         |
                 v                         v
             Service                  Domain Model
                                           |
                                           v
                                      Domain Event
                                           |
                                           v
                                  +----------------+
                                  | Event Platform |
                                  |                |
                                  | Kafka          |
                                  | Schema Registry|
                                  | ACL            |
                                  | DLQ            |
                                  +-------+--------+
                                          |
                     +--------------------+--------------------+
                     |                    |                    |
                     v                    v                    v
                  Service A            Service B            Service C
                     |                    |                    |
                     v                    v                    v
                    DB                   DB                   DB
```

外围：

```text
OpenTelemetry
Prometheus
Grafana
ELK
Security
Governance
```

---

# 五十四、EDA 的设计原则

可以总结为十条。

### 原则一：Event 表达事实

```text
OrderCreated
```

而不是：

```text
CreateOrder
```

### 原则二：Event Contract First

先定义：

```text
Schema
```

再开发 Producer / Consumer。

### 原则三：Consumer 必须幂等

不要假设：

```text
Event only once
```

### 原则四：接受 Eventual Consistency

不要强行把所有场景做成：

```text
Distributed Transaction
```

### 原则五：设计 Retry + DLQ

不要无限 Retry。

### 原则六：事件需要版本化

```text
v1
v2
v3
```

### 原则七：控制 Event Size

不要把大型文件直接塞进 Kafka。

### 原则八：事件必须可观测

至少包含：

```text
eventId
traceId
correlationId
```

### 原则九：Event Schema 必须治理

避免：

```text
Schema Chaos
```

### 原则十：不要为了异步而异步

架构必须服务于业务。

---

# 五十五、从传统微服务走向 Event-Driven Microservices

传统微服务：

```text
             API Gateway
                  |
       +----------+----------+
       |          |          |
       v          v          v
    Order      Payment    Inventory
       |          |          |
       +----------+----------+
```

Event-Driven Microservices：

```text
             API Gateway
                  |
                  v
               Order
                  |
                  v
              Event Bus
                  |
       +----------+----------+
       |          |          |
       v          v          v
    Payment    Inventory  Notification
```

进一步：

```text
                       Event Mesh
                           |
       +-------------------+-------------------+
       |                   |                   |
     Domain A            Domain B            Domain C
       |                   |                   |
    Services            Services            Services
```

这就是现代企业架构从：

> **Service-Centric**

逐渐走向：

> **Event-Centric**

---

# 五十六、Event-Driven Architecture 的最终认知模型

可以用一张图总结：

```text
                    Event-Driven System

                         Command
                            |
                            v
                      Domain Service
                            |
                            v
                     State Change
                            |
                            v
                         Event
                            |
                            v
                     Event Broker
                            |
          +-----------------+-----------------+
          |                 |                 |
          v                 v                 v
      Consumer A        Consumer B        Consumer C
          |                 |                 |
          v                 v                 v
        State             State             State
```

如果加入现代 AI：

```text
                         User
                           |
                           v
                    Agent Runtime
                           |
                 +---------+---------+
                 |                   |
                A2A                 MCP
                 |                   |
                 v                   v
              Agents              Tools
                 |
                 v
               Events
                 |
                 v
              Kafka
                 |
       +---------+---------+
       |         |         |
     Audit    Billing   Analytics
```

最终形成：

```text
┌────────────────────────────────────────────┐
│              AI / Applications             │
├────────────────────────────────────────────┤
│ Agent Runtime / Microservices              │
├────────────────────────────────────────────┤
│ A2A / MCP / REST                           │
├────────────────────────────────────────────┤
│ Event-Driven Architecture                  │
│ Kafka / Pulsar / Event Bus                 │
├────────────────────────────────────────────┤
│ Workflow / Saga / CQRS                     │
├────────────────────────────────────────────┤
│ Observability / Security / Governance      │
├────────────────────────────────────────────┤
│ Kubernetes / Cloud / Database              │
└────────────────────────────────────────────┘
```

---

# 五十七、结语：Event 是分布式系统的“事实传播机制”

如果 REST 的核心思想是：

> **Call**

那么消息队列的核心思想是：

> **Deliver**

而 Event-Driven Architecture 的核心思想是：

> **Publish Facts**

系统不再围绕：

```text
"谁调用谁？"
```

设计。

而是围绕：

```text
"发生了什么？"
```

设计。

例如：

```text
OrderCreated
PaymentCompleted
InventoryReserved
ShipmentCreated
```

这些 Event 构成了整个企业系统的：

> **Business Event Flow**

而现代 Agent 系统进一步把这一思想扩展到了 AI：

```text
AgentStarted
TaskCreated
ToolCalled
ToolCompleted
AgentWaiting
AgentCompleted
AgentFailed
```

最终形成：

```text
                   Event-Driven World

      Human
        |
        v
      Agent
        |
       A2A
        |
      Agent
        |
       MCP
        |
      Tool
        |
      Event
        |
       Kafka
        |
 +------+------+------+
 |      |      |      |
Audit Billing Analytics Workflow
```

因此，从架构演进的角度看：

```text
Monolith
   ↓
Microservices
   ↓
Event-Driven Microservices
   ↓
Cloud Native
   ↓
Agentic Architecture
   ↓
Event-Driven Agent Platform
```

**EDA 并没有因为 Agent 的出现而过时，反而可能成为 Agent Runtime、A2A、MCP 和企业分布式系统之间的重要连接层。**


> **Event → State → Workflow → Runtime → Agent**

架构思维。
