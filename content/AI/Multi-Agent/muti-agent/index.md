---
title: Multi-Agent多智能体
# tags:
#   - nodejs
date: '2026-08-05'
summary: 让多个“AI Agent”各自负责不同任务，通过协作完成一个复杂目标。
---


AI 里的 **Multi-Agent（多智能体）**，可以简单理解成：

> **让多个“AI Agent”各自负责不同任务，通过协作完成一个复杂目标。**

它和普通的 ChatGPT/LLM 最大的区别是：**不是一个 AI 从头干到尾，而是把复杂工作拆给多个专业 Agent。**

### 1. 先理解什么是 Agent

一个普通 LLM：

```text
用户 → LLM → 答案
```

Agent 则通常是：

```text
用户目标
   ↓
Agent
   ├── 思考
   ├── 调用工具
   ├── 查询数据库
   ├── 调用 API
   ├── 执行代码
   └── 根据结果继续决策
```

例如你说：

> “帮我分析一下这个 Java 项目为什么性能差。”

一个 Agent 可以：

```text
读取代码
   ↓
分析 JVM
   ↓
分析 SQL
   ↓
分析 Redis
   ↓
分析线程池
   ↓
生成性能优化方案
```

---

# 2. Multi-Agent 是什么？

如果问题特别复杂，可以把工作拆成多个 Agent：

```text
                    ┌── Java Agent
                    │
                    ├── Database Agent
用户 → Manager Agent ├── Redis Agent
                    │
                    ├── JVM Agent
                    │
                    └── Security Agent
                           ↓
                     Manager 汇总
                           ↓
                        最终方案
```

例如：

**Manager Agent**

负责：

> “这个系统为什么慢？请组织几个专家分析。”

然后：

**Java Agent**

负责分析 Java 代码。

**Database Agent**

负责：

```text
SQL
Index
JOIN
Execution Plan
Slow Query
```

**Redis Agent**

负责：

```text
Cache
Hot Key
Big Key
TTL
Redis Cluster
```

**JVM Agent**

负责：

```text
GC
Heap
Thread
CPU
Memory
```

最后 Manager Agent：

```text
Java Agent        ──┐
Database Agent     ─┤
Redis Agent        ─┼──→ Manager → 最终报告
JVM Agent          ─┤
Security Agent     ─┘
```

这就是 Multi-Agent。

---

# 3. Multi-Agent 和微服务其实很像

如果你是 Java 后端开发，这个概念会非常容易理解。

可以把它类比成：

### 微服务

```text
Order Service
Payment Service
Inventory Service
User Service
```

每个服务负责一个领域。

### Multi-Agent

```text
Order Agent
Payment Agent
Inventory Agent
User Agent
```

每个 Agent 负责一个智能任务。

所以可以简单记：

> **Microservice 是把软件系统拆成多个服务；Multi-Agent 是把智能任务拆成多个 AI Agent。**

---

# 4. Agent之间怎么协作？

这是 Multi-Agent 最核心的问题。

常见有几种模式。

### 模式一：Supervisor

最常见。

```text
                 Supervisor
                /     |     \
               ↓      ↓      ↓
           Agent A Agent B Agent C
               \      |      /
                ↓     ↓     ↓
                 Supervisor
                     ↓
                   Result
```

Supervisor 就像一个项目经理。

例如：

> “帮我设计一个秒杀系统。”

Supervisor：

```text
→ 找 Architecture Agent
→ 找 Redis Agent
→ 找 Kafka Agent
→ 找 Database Agent
→ 找 Performance Agent
```

最后自己汇总。

这个模式非常适合你之前研究的**秒杀、高并发、系统架构设计**。

---

# 5. 模式二：Agent Pipeline

一个接一个执行：

```text
Research Agent
      ↓
Analysis Agent
      ↓
Coding Agent
      ↓
Testing Agent
      ↓
Review Agent
```

例如：

> 自动开发一个 Java REST API。

可以：

```text
需求 Agent
   ↓
架构 Agent
   ↓
Coding Agent
   ↓
Test Agent
   ↓
Code Review Agent
```

这和 CI/CD Pipeline 很像。

---

# 6. 模式三：多个 Agent 平等协作

例如：

```text
       Agent A
       ↙     ↘
 Agent B ←→ Agent C
       ↘     ↙
       Agent D
```

每个 Agent 都可以和其他 Agent 交流。

这种模式灵活，但是也更复杂，因为容易出现：

```text
Agent A → Agent B
Agent B → Agent C
Agent C → Agent A
Agent A → Agent B
...
```

形成循环。

所以实际生产系统通常会更加严格地控制 Agent 的通信。

---

# 7. Multi-Agent 最重要的几个组件

一个完整 Multi-Agent 系统通常包含：

```text
                    ┌──────────────┐
                    │   User       │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │ Orchestrator │
                    │ / Supervisor │
                    └──────┬───────┘
                           ↓
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
        ┌─────────┐   ┌─────────┐   ┌─────────┐
        │ Agent A │   │ Agent B │   │ Agent C │
        └────┬────┘   └────┬────┘   └────┬────┘
             ↓             ↓             ↓
          Tools          Tools          Tools
             ↓             ↓             ↓
        ┌────────────────────────────────────┐
        │ Database / API / Search / Code     │
        └────────────────────────────────────┘
                           ↓
                        Memory
                           ↓
                     Final Result
```

几个核心概念：

| 组件           | 作用          |
| ------------ | ----------- |
| Agent        | 执行具体任务      |
| LLM          | Agent 的“大脑” |
| Tool         | Agent 的“手”  |
| Memory       | Agent 的记忆   |
| Orchestrator | Agent 调度器   |
| Message      | Agent 之间通信  |
| Planner      | 任务规划        |
| Workflow     | 控制执行流程      |

---

# 8. Agent 和 LLM 的关系

这是学习 Multi-Agent 时非常重要的一点。

**LLM ≠ Agent**

可以理解为：

```text
LLM
 ↓
负责思考/生成
```

而：

```text
Agent =
LLM
+
Prompt
+
Tools
+
Memory
+
Planning
+
Execution
```

所以：

```text
             Agent
        ┌──────────────┐
        │     LLM      │ ← 大脑
        │              │
        │   Memory     │ ← 记忆
        │              │
        │   Tools      │ ← 工具
        │              │
        │   Planning   │ ← 规划
        │              │
        │   Actions    │ ← 行动
        └──────────────┘
```

Multi-Agent 就是：

```text
Agent A
Agent B
Agent C
Agent D
   ↓
Communication
   ↓
Collaboration
```

---

# 9. 举一个你比较容易理解的 Java 项目例子

假设：

> **“帮我开发一个订单系统。”**

Multi-Agent 可以这样设计：

```text
                    Product Manager Agent
                              ↓
                    Architecture Agent
                              ↓
             ┌────────────────┼────────────────┐
             ↓                ↓                ↓
        Java Agent       Database Agent    Redis Agent
             ↓                ↓                ↓
         Coding Agent     SQL Agent       Cache Agent
             └────────────────┼────────────────┘
                              ↓
                       Testing Agent
                              ↓
                       Security Agent
                              ↓
                       Review Agent
                              ↓
                         Final Agent
```

最终可能生成：

```text
Spring Boot
Spring Cloud
PostgreSQL
Redis
Kafka
Docker
Kubernetes
OpenTelemetry
```

甚至可以让 Agent 自己：

```text
写代码
 ↓
运行测试
 ↓
发现错误
 ↓
修改代码
 ↓
重新测试
 ↓
Code Review
 ↓
生成 PR
```

这时候 Multi-Agent 就已经不是简单的“聊天机器人”了。

它更接近：

> **AI Software Engineering Team**

---

# 10. Multi-Agent 和 RAG 的关系

这也是现在 AI 面试经常问的。

**RAG：让 AI 获得知识。**

```text
User
 ↓
Question
 ↓
Embedding
 ↓
Vector DB
 ↓
Retrieve
 ↓
LLM
 ↓
Answer
```

而 Multi-Agent：

> **让多个 AI 分工协作完成任务。**

两者可以结合：

```text
                  Supervisor
                 /     |     \
                ↓      ↓      ↓
           Java Agent DB Agent Security Agent
                ↓      ↓      ↓
               RAG    RAG     RAG
                ↓      ↓      ↓
              Vector Database
```

所以：

> **RAG 解决“Agent 去哪里获取知识”，Multi-Agent 解决“多个 Agent 如何协作完成任务”。**

---

# 11. 现在比较重要的 Multi-Agent Framework

你如果准备往 **AI Full-Stack / AI Agent Developer** 方向发展，这些值得了解：

* **LangGraph** —— 很重要，适合构建有状态、可控的 Agent workflow
* **AutoGen** —— Microsoft 的多 Agent 框架
* **CrewAI** —— 强调 Agent + Role + Task + Crew
* **OpenAI Agents SDK** —— 用于构建 Agent 应用
* **Google ADK** —— Google 的 Agent Development Kit
* **MCP** —— 不完全是 Multi-Agent Framework，但现在 Agent 与外部工具/系统连接非常重要

---


