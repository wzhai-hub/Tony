---
title: Agent
# tags:
#   - nodejs
date: '2026-08-05'
summary: 一个能够理解目标、进行决策、调用工具、执行任务，并根据执行结果继续行动的 AI 系统。
---

**AI Agent（AI 智能体）**可以简单理解为：

> **一个能够理解目标、进行决策、调用工具、执行任务，并根据执行结果继续行动的 AI 系统。**

如果说普通 ChatGPT 是：

> **“你问我答”**

那么 AI Agent 更像：

> **“你给我一个目标，我自己想办法完成。”**

---

# 一、先看最简单的区别

### 普通 LLM

比如你问：

> “Spring Boot 中 @Transactional 为什么会失效？”

流程：

```text
用户问题
   ↓
LLM
   ↓
回答
```

LLM 主要是在**生成文本**。

---

### AI Agent

你告诉 Agent：

> “帮我分析这个 Spring Boot 项目的事务问题，并给出修改方案。”

Agent 可能自己执行：

```text
用户目标
   ↓
Agent
   ↓
分析任务
   ↓
读取项目代码
   ↓
搜索相关代码
   ↓
查询数据库
   ↓
分析 @Transactional
   ↓
发现问题
   ↓
修改代码
   ↓
运行测试
   ↓
检查测试结果
   ↓
生成报告
```

这里的关键变化是：

> **Agent 不只是回答问题，而是在执行任务。**

---

# 二、AI Agent 的核心组成

一个典型 Agent 可以理解成：

```text
                 ┌──────────────┐
                 │     User     │
                 └──────┬───────┘
                        ↓
                 ┌──────────────┐
                 │    Agent     │
                 │              │
                 │     LLM      │
                 │      +       │
                 │   Planning   │
                 │      +       │
                 │    Memory    │
                 │      +       │
                 │    Tools     │
                 └──────┬───────┘
                        ↓
              ┌─────────┼─────────┐
              ↓         ↓         ↓
           Database     API      RAG
              ↓         ↓         ↓
              └─────────┼─────────┘
                        ↓
                    Tool Result
                        ↓
                     Agent
                        ↓
                    Next Step
```

核心可以记成：

> **Agent = LLM + Tools + Memory/State + Planning/Workflow**

---

# 三、Agent 最重要的能力：Tool Calling

这是理解 Agent 的关键。

假设用户问：

> “帮我查一下我的订单 #12345，然后告诉我什么时候能收到。”

LLM 本身不能直接访问你的订单数据库。

所以给 Agent 一个工具：

```text
getOrder(orderId)
```

Agent 可以：

```text
User
 ↓
Agent
 ↓
LLM 判断：
需要查询订单
 ↓
调用 getOrder()
 ↓
Order Service
 ↓
返回订单信息
 ↓
Agent
 ↓
LLM 分析
 ↓
回答用户
```

例如：

```json
{
  "orderId": "12345",
  "status": "SHIPPED",
  "estimatedDelivery": "2026-08-18"
}
```

然后 Agent：

> “订单 12345 已经发货，预计 8 月 18 日送达。”

---

# 四、Agent 和 RAG 的区别

这个你刚刚学习过 RAG，所以一定要把两者区分开。

### RAG

核心是：

> **查资料。**

```text
Question
   ↓
Retriever
   ↓
Knowledge Base
   ↓
Relevant Documents
   ↓
LLM
   ↓
Answer
```

例如：

> “公司的报销政策是什么？”

RAG 去查：

```text
员工手册
财务政策
公司 Wiki
```

然后回答。

---

### Agent

核心是：

> **完成任务。**

例如：

> “帮我处理这个客户退款。”

Agent 可能：

```text
Agent
 ↓
查询订单
 ↓
查询退款政策
 ↓
判断是否符合条件
 ↓
调用退款 API
 ↓
检查退款结果
 ↓
通知客户
```

所以：

> **RAG 是 Agent 可以使用的一种能力。**

例如：

```text
                    Agent
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
         RAG        Database      API
          ↓           ↓           ↓
       查资料       查数据       执行动作
```

---

# 五、Agent 和传统程序有什么区别？

这是 AI 面试很喜欢问的问题。

传统程序：

```java
if (conditionA) {
    callServiceA();
} else if (conditionB) {
    callServiceB();
}
```

开发人员提前定义：

```text
什么时候调用 A
什么时候调用 B
执行顺序是什么
异常怎么办
```

也就是说：

> **程序员定义 Workflow。**

---

Agent：

```text
Goal
 ↓
LLM
 ↓
判断下一步
 ↓
Tool
 ↓
观察结果
 ↓
LLM 再判断
 ↓
Tool
 ↓
观察结果
 ↓
完成任务
```

也就是说：

> **LLM 在运行过程中参与决策。**

这就是 Agent 的一个核心特点。

---

# 六、一个非常典型的 Agent Loop

你可以把 Agent 记成这个循环：

```text
        ┌─────────────┐
        │    Goal     │
        └──────┬──────┘
               ↓
        ┌─────────────┐
        │   Observe   │
        └──────┬──────┘
               ↓
        ┌─────────────┐
        │   Reason    │
        └──────┬──────┘
               ↓
        ┌─────────────┐
        │    Plan     │
        └──────┬──────┘
               ↓
        ┌─────────────┐
        │     Act     │
        └──────┬──────┘
               ↓
            Tool/API
               ↓
          Tool Result
               │
               └──────────→ Observe
```

这个循环可能执行：

```text
1 次
5 次
10 次
```

直到：

```text
任务完成
```

或者：

```text
达到最大执行次数
```

---

# 七、举一个你比较容易理解的 Java 开发场景

假设你做了一个：

> **Java 项目代码分析 Agent**

用户：

> “帮我分析这个 Spring Boot 项目为什么 CPU 使用率很高。”

Agent 可以拥有这些 Tools：

```text
Tool 1:
readFile()

Tool 2:
searchCode()

Tool 3:
queryDatabase()

Tool 4:
queryPrometheus()

Tool 5:
searchKnowledgeBase()

Tool 6:
runTest()
```

Agent 执行：

```text
User
 ↓
Agent
 ↓
查询 Prometheus
 ↓
发现 CPU 95%
 ↓
搜索代码
 ↓
发现某个接口
 ↓
读取代码
 ↓
发现 while loop
 ↓
搜索历史知识库
 ↓
发现类似问题
 ↓
分析
 ↓
生成解决方案
```

这就已经是一个非常典型的：

> **Software Engineering Agent**

---

# 八、Agent 的 Memory 是什么？

Agent 还需要记住执行过程。

比如：

```text
用户：
帮我分析订单 12345

Agent：
查询订单……

Tool：
订单不存在

Agent：
那我查询客户 ID……

Tool：
客户 ID = 888

Agent：
查询客户 888 的订单……

Tool：
找到订单 12345
```

Agent 需要保存：

```text
Order ID
Customer ID
Previous Tool Results
Previous Actions
Current Task State
```

这就是：

> **Memory / State**

现代 Agent 系统通常更强调 **State（状态）**，而不只是传统意义上的聊天 Memory。

---

# 九、Agent 可以使用哪些 Tool？

这是 Agent 最强大的地方。

例如：

### 数据库

```text
MySQL
PostgreSQL
Oracle
MongoDB
```

Agent 可以：

```text
SQL Query
```

---

### API

```text
Payment API
Order API
Weather API
CRM API
```

---

### 搜索

```text
Web Search
Enterprise Search
Knowledge Base
```

---

### 文件

```text
PDF
Word
Excel
CSV
Code
```

---

### DevOps

甚至可以：

```text
Git
Jenkins
Kubernetes
Docker
Prometheus
Grafana
```

比如：

> “帮我检查生产环境为什么订单服务响应变慢。”

Agent：

```text
Prometheus
   ↓
Grafana
   ↓
Logs
   ↓
Trace
   ↓
Kubernetes
   ↓
Code
   ↓
分析
```

这就非常接近真正的企业级 AI Agent。

---

# 十、Agent、Workflow、RAG 三者关系

这个概念你以后面试一定要掌握。

### Workflow

人定义流程：

```text
A → B → C → D
```

例如：

```text
收到订单
 ↓
检查库存
 ↓
扣款
 ↓
发货
```

流程是确定的。

---

### RAG

主要解决：

```text
我不知道
 ↓
去知识库找
 ↓
找到资料
 ↓
回答
```

---

### Agent

主要解决：

```text
我有一个目标
 ↓
我需要做什么？
 ↓
选择 Tool
 ↓
执行
 ↓
看结果
 ↓
决定下一步
 ↓
继续执行
```

可以总结成：

| 技术           | 核心作用                |
| ------------ | ------------------- |
| LLM          | 思考/生成               |
| RAG          | 获取知识                |
| Tool Calling | 使用工具                |
| Workflow     | 固定流程                |
| Agent        | 动态决策和执行             |
| Memory/State | 保存上下文               |
| LangGraph    | 构建复杂 Agent Workflow |

---

# 十一、Agent 和 LangChain / LangGraph 的关系

你刚刚问过 LangChain，现在就可以串起来了。

```text
                    AI Application
                          │
             ┌────────────┴────────────┐
             ↓                         ↓
           RAG                       Agent
             │                         │
             ↓                         ↓
        LangChain              LangChain / LangGraph
                                       │
                         ┌─────────────┼────────────┐
                         ↓             ↓            ↓
                       Tools        Memory       Workflow
                         ↓             ↓            ↓
                       API          State         Nodes
```

简单理解：

> **LangChain：提供 AI 应用开发组件。**

> **LangGraph：更适合构建有状态、复杂、可循环的 Agent Workflow。**

---

# 十二、一个企业级 AI Agent 架构

如果你以后做 AI Full-Stack 项目，可以设计成：

```text
                    React
                      │
                      ↓
               Spring Boot API
                      │
                      ↓
                 AI Gateway
                      │
                      ↓
                    Agent
                      │
       ┌──────────────┼───────────────┐
       ↓              ↓               ↓
      RAG           Tools           Memory
       │              │               │
       ↓              ↓               ↓
 Vector DB       REST API          Redis
 pgvector        Database
 Elasticsearch   Kubernetes
                 Git
                 Prometheus
                      │
                      ↓
                     LLM
                      │
             ┌────────┼────────┐
             ↓        ↓        ↓
            GPT     Claude    Gemini
```

这已经是一个很典型的 **Enterprise AI Agent Architecture**。

---

# 十三、AI Agent 最重要的几个问题


1. **什么是 AI Agent？**
2. Agent 和 LLM 有什么区别？
3. Agent 和 RAG 有什么区别？
4. 什么是 Tool Calling？
5. Function Calling 是什么？
6. Agent 如何选择 Tool？
7. Agent 如何处理 Tool 执行失败？
8. Agent Memory 和 State 有什么区别？
9. Agent 如何防止无限循环？
10. Agent 如何控制成本？
11. Agent 如何保证安全？
12. 什么是 Multi-Agent？
13. Agent 和 Workflow 有什么区别？
14. LangChain 和 LangGraph 有什么区别？
15. 如何设计一个企业级 Agent？
16. 如何对 Agent 做 Observability？
17. 如何评价 Agent 的效果？
18. 如何防止 Agent 执行危险操作？

其中第 **6、7、8、10、14、15、16** 个问题，对于有 Java/Spring 后端经验的人尤其值得深入。

---

## 最后用一句话记住

你可以把现在学到的三个概念记成：

```text
LLM
 ↓
负责“思考和生成”

RAG
 ↓
负责“查知识”

Tool Calling
 ↓
负责“使用工具”

Agent
 ↓
负责“决定做什么 + 调用工具 + 根据结果继续行动”
```

而进一步组合起来：

```text
              AI Agent
                 │
        ┌────────┼────────┐
        ↓        ↓        ↓
       LLM      RAG      Tools
        │        │        │
        └────────┼────────┘
                 ↓
            State/Memory
                 ↓
             Workflow
                 ↓
            完成任务
```


