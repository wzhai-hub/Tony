---
title: LangGraph
# tags:
#   - nodejs
date: '2026-08-05'
summary: 专门用来构建“有状态、可循环、可控制”的 AI Agent 工作流框架。
---

**LangGraph** 可以理解成：

> **专门用来构建“有状态、可循环、可控制”的 AI Agent 工作流框架。**

如果说 **LangChain 是工具箱**，那么 **LangGraph 更像是流程引擎 / 状态机**。

这两个经常一起使用。

---

# 1. 为什么需要 LangGraph？

先看一个普通的 LLM：

```text
用户
 ↓
Prompt
 ↓
LLM
 ↓
回答
```

很简单。

但是实际的 AI Agent 往往是：

```text
用户问题
   ↓
分析问题
   ↓
需要查询数据库？
   ├── 是 → 查询数据库
   │          ↓
   │       分析结果
   │          ↓
   │       结果是否正确？
   │          ├── 否 → 再次查询
   │          └── 是
   ↓
生成答案
```

甚至：

```text
用户
 ↓
Agent
 ↓
调用工具
 ↓
得到结果
 ↓
Agent重新思考
 ↓
调用另一个工具
 ↓
得到结果
 ↓
Agent重新思考
 ↓
最终答案
```

这种**有状态、分支、循环、人工介入、多 Agent 协作**的流程，就非常适合 LangGraph。

---

# 2. LangGraph 最核心的思想

LangGraph 的名字里面有一个 **Graph（图）**。

它把 Agent 应用抽象成：

```text
        ┌──────────────┐
        │              ↓
START → 分析问题 → 查询数据库
        │              │
        │              ↓
        └────────── 检查结果
                       │
                ┌──────┴──────┐
                ↓             ↓
              正确           错误
                ↓             │
              生成答案 ←──────┘
                │
               END
```

这里有三个非常重要的概念：

```text
State
Node
Edge
```

---

# 3. State —— 状态

State 可以理解成：

> **整个 Agent 工作过程中需要保存的数据。**

例如：

```python
class State(TypedDict):
    question: str
    sql: str
    result: str
    answer: str
```

执行过程中：

```text
question
   ↓
sql
   ↓
result
   ↓
answer
```

State 会不断被更新。

例如：

```text
State：

question = "查询昨天销售额"

sql = ""

result = ""

answer = ""
```

SQL Agent 执行之后：

```text
State：

question = "查询昨天销售额"

sql = "SELECT SUM(amount) ..."

result = "125000"

answer = ""
```

最后：

```text
State：

question = "查询昨天销售额"

sql = "SELECT SUM(amount) ..."

result = "125000"

answer = "昨天销售额为125000元"
```

---

# 4. Node —— 节点

Node 就是一个具体的处理步骤。

例如：

```text
Node 1：分析问题

Node 2：生成 SQL

Node 3：执行 SQL

Node 4：检查 SQL

Node 5：生成答案
```

可以理解成 Java 中的方法：

```java
analyzeQuestion()

generateSQL()

executeSQL()

checkResult()

generateAnswer()
```

---

# 5. Edge —— 边

Edge 决定：

> **下一步执行哪个 Node。**

例如：

```text
generateSQL
      ↓
executeSQL
      ↓
checkSQL
```

但是 `checkSQL` 可能有两种结果：

```text
          checkSQL
          /      \
       正确      错误
        ↓          ↓
     answer    generateSQL
```

这就是 **Conditional Edge（条件边）**。

---

# 6. LangGraph 最重要的能力：循环

这是 LangGraph 和普通 Chain 一个非常重要的区别。

例如 Agent：

```text
思考
 ↓
调用工具
 ↓
观察结果
 ↓
思考
 ↓
调用工具
 ↓
观察结果
 ↓
思考
 ↓
最终答案
```

可以形成：

```text
        ┌─────────────┐
        ↓             │
      Think           │
        ↓             │
      Tool            │
        ↓             │
    Observation       │
        ↓             │
   是否完成？ ── 否 ──┘
        │
       是
        ↓
      Answer
```

这种循环非常适合 Agent。

---

# 7. LangChain vs LangGraph

|                   | LangChain                | LangGraph                   |
| ----------------- | ------------------------ | --------------------------- |
| 核心思想              | LLM 应用组件                 | Agent 工作流                   |
| 抽象                | Chain / Tool / Retriever | Graph / State / Node / Edge |
| 简单 LLM            | ⭐⭐⭐⭐⭐                    | ⭐⭐                          |
| RAG               | ⭐⭐⭐⭐⭐                    | ⭐⭐⭐⭐                        |
| Agent             | ⭐⭐⭐⭐                     | ⭐⭐⭐⭐⭐                       |
| 循环                | 一般                       | ⭐⭐⭐⭐⭐                       |
| 状态管理              | 一般                       | ⭐⭐⭐⭐⭐                       |
| 条件分支              | 有                        | 很强                          |
| Human-in-the-loop | 有相关能力                    | 很适合                         |
| 多 Agent           | 可以                       | 很适合                         |
| 复杂工作流             | 一般                       | ⭐⭐⭐⭐⭐                       |

简单来说：

```text
LangChain
    ↓
提供各种 AI 组件

LangGraph
    ↓
把这些组件组织成复杂的 Agent 工作流
```

---

# 8. 一个实际例子：SQL Agent

假设你做一个企业数据分析 AI。

用户：

```text
帮我分析一下今年销售额下降的原因。
```

Agent 可能需要：

```text
             用户问题
                 ↓
          分析问题
                 ↓
          生成 SQL
                 ↓
          查询数据库
                 ↓
          分析数据
                 ↓
       是否需要更多数据？
          ↙          ↘
        是            否
        ↓              ↓
    再次查询          生成报告
        ↓
      分析
        │
        └───────────┐
                    ↓
                 最终答案
```

用 LangGraph，就可以把它表示成：

```text
State
  │
  ├── question
  ├── sql
  ├── database_result
  ├── analysis
  └── answer

Node
  │
  ├── analyze_question
  ├── generate_sql
  ├── execute_sql
  ├── analyze_result
  └── generate_answer

Edge
  │
  ├── analyze → generate_sql
  ├── generate_sql → execute_sql
  ├── execute_sql → analyze_result
  └── analyze_result → generate_sql / answer
```

这就非常像一个**有状态的工作流引擎**。

---

# 9. 为什么 Java 后端工程师特别容易理解 LangGraph？

你本身有 Java、Spring、微服务、Kubernetes 等后端背景的话，可以把 LangGraph 类比成：

```text
Spring Application
       ↓
业务流程
       ↓
Service A
       ↓
Service B
       ↓
Service C
```

而 LangGraph：

```text
Agent Workflow
       ↓
Node A
       ↓
Node B
       ↓
Node C
       ↓
根据 State 决定下一步
       ↓
Node A / Node D / END
```

甚至可以把它理解成：

> **State Machine + Workflow Engine + LLM Agent**

这个理解非常接近它的核心思想。

---

# 10. LangGraph 和 Spring StateMachine 的感觉很像

如果你以前接触过状态机，可以这样理解：

```text
StateMachine

State
 ↓
Event
 ↓
Transition
 ↓
Next State
```

LangGraph：

```text
State
 ↓
Node
 ↓
Edge
 ↓
Next Node
```

例如订单：

```text
CREATED
   ↓
PAID
   ↓
SHIPPED
   ↓
DELIVERED
```

Agent：

```text
QUESTION
   ↓
THINK
   ↓
TOOL_CALL
   ↓
OBSERVATION
   ↓
THINK
   ↓
ANSWER
```

所以对于后端开发者来说，**LangGraph 的思维其实并不陌生**。

---

# 11. LangGraph 在 AI Agent 架构中的位置

现在你可以把整个 AI 技术栈理解成：

```text
                    AI Application
                         │
             ┌───────────┴───────────┐
             ↓                       ↓
            RAG                    Agent
             │                       │
       ┌─────┴─────┐          ┌──────┴──────┐
       ↓           ↓          ↓             ↓
   Vector DB    Retriever    Tools       Workflow
                               │             │
                               │         LangGraph
                               ↓
                              API
                               │
                               ↓
                            Database
```

而底层：

```text
                LLM
                 │
        ┌────────┼────────┐
        ↓        ↓        ↓
      GPT     Claude   Gemini
```

---


