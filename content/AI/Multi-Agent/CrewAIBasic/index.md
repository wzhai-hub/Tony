---
title: CrewAI
# tags:
#   - nodejs
date: '2026-08-05'
summary: CrewAI 是一个 Python 的 AI Agent / Multi-Agent 编排框架
---


## 1. CrewAI 是什么？

**CrewAI 是一个 Python 的 AI Agent / Multi-Agent 编排框架**，核心思想是：

> 不让一个 AI Agent 什么都干，而是让多个具有不同角色的 Agent 像一个团队一样协作完成任务。

官方目前把 CrewAI 分成两个核心概念：

* **Crew**：多个 Agent 组成的“团队”
* **Flow**：控制整个业务流程的“工作流”

CrewAI 本身是独立实现的，并不依赖 LangChain.

可以把它理解成：

```text
                    用户需求
                       │
                       ▼
                ┌──────────────┐
                │   Crew / Flow │
                └──────┬───────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     Researcher     Developer     Reviewer
       Agent          Agent         Agent
          │            │            │
          └────────────┼────────────┘
                       ▼
                    最终结果
```

比如你让 AI：

> “帮我研究 Java 微服务架构，并生成一份技术方案。”

可以拆成：

```text
Researcher Agent
    ↓
搜索微服务最新技术

Architect Agent
    ↓
设计整体架构

Java Expert Agent
    ↓
分析 Spring Boot / Spring Cloud

Reviewer Agent
    ↓
检查方案

Writer Agent
    ↓
生成最终技术方案
```

这就是 CrewAI 最典型的应用场景。

---

# 2. CrewAI 的核心概念

你学习 CrewAI，首先掌握下面 6 个东西：

```text
Agent
  ↓
Task
  ↓
Crew
  ↓
Process
  ↓
Tool
  ↓
Flow
```

其中最重要的是：

```text
Agent + Task + Crew
```

---

## 3. Agent 是什么？

Agent 就是一个“AI 员工”。

例如：

```text
Researcher
```

可以定义成：

```text
Role:
    Senior Technology Researcher

Goal:
    Research the latest Java technologies

Backstory:
    You are an experienced Java architect
    with 15 years of experience.
```

代码大致是：

```python
from crewai import Agent

researcher = Agent(
    role="Senior Technology Researcher",
    goal="Research the latest Java technologies",
    backstory="""
    You are a senior Java architect with 15 years
    of experience in enterprise software architecture.
    """,
    verbose=True
)
```

Agent 本质上就是：

```text
LLM
+
System Prompt
+
Role
+
Goal
+
Tools
+
Memory
+
Reasoning
```

所以你可以把它理解成：

> **Agent = 一个有角色、有目标、有工具的 LLM。**

官方现在的 Agent 还支持 tools、memory、knowledge、structured output 等能力。([CrewAI Documentation][1])

---

# 4. Task 是什么？

Task 就是给 Agent 的任务。

例如：

```python
from crewai import Task

research_task = Task(
    description="""
    Research the latest developments in Java 21
    and Spring Boot 3.
    """,
    expected_output="""
    A technical report describing the major
    changes and important features.
    """,
    agent=researcher
)
```

可以理解为：

```text
Agent = 谁来做

Task = 做什么

expected_output = 最终需要什么结果
```

---

# 5. Crew 是什么？

Crew 就是把多个 Agent 和 Task 组织起来。

例如：

```python
from crewai import Crew, Process

crew = Crew(
    agents=[
        researcher,
        writer
    ],
    tasks=[
        research_task,
        writing_task
    ],
    process=Process.sequential,
    verbose=True
)
```

然后：

```python
result = crew.kickoff()
```

整个流程：

```text
Crew
 │
 ├── Researcher
 │      │
 │      └── Research Java 21
 │
 └── Writer
        │
        └── Write technical report
```

---

# 6. Process 是什么？

Process 控制 Agent 怎么协作。

最简单的是：

```python
Process.sequential
```

也就是：

```text
Task 1
  ↓
Task 2
  ↓
Task 3
  ↓
Task 4
```

例如：

```text
Research
   ↓
Analysis
   ↓
Writing
   ↓
Review
```

另外还有更复杂的协作方式，例如 hierarchical。

可以理解成：

```text
              Manager Agent
                    │
          ┌─────────┼─────────┐
          ↓         ↓         ↓
     Researcher  Developer  Reviewer
```

官方文档也提供 sequential、hierarchical 等任务组织方式。([CrewAI Documentation][1])

---

# 7. Tool 是什么？

这是非常重要的一个概念。

Agent 本身只能调用 LLM。

如果希望 Agent：

```text
搜索互联网
读取文件
查询数据库
调用 REST API
操作 GitHub
执行代码
查询 Elasticsearch
```

就需要给 Agent 配 Tool。

例如：

```python
researcher = Agent(
    role="Researcher",
    goal="Research Java technologies",
    backstory="Experienced Java architect",
    tools=[
        search_tool
    ]
)
```

于是：

```text
Agent
  │
  ├── LLM
  │
  └── Tools
       ├── Web Search
       ├── Database
       ├── API
       └── File System
```

这就是 Agent 与普通 ChatGPT 对话最大的区别之一。

---

# 8. CrewAI 最简单的完整例子

> **让两个 AI Agent 协作研究 Java 21。**

### Agent 1：Researcher

```python
from crewai import Agent

researcher = Agent(
    role="Java Researcher",
    goal="Research Java 21 features",
    backstory="""
    You are an experienced Java developer
    and technology researcher.
    """,
    verbose=True
)
```

### Agent 2：Writer

```python
writer = Agent(
    role="Technical Writer",
    goal="Write a clear Java 21 technical report",
    backstory="""
    You are an experienced technical writer
    specializing in Java technologies.
    """,
    verbose=True
)
```

### Task 1

```python
from crewai import Task

research_task = Task(
    description="""
    Research the major features introduced
    in Java 21.
    """,
    expected_output="""
    A detailed list of Java 21 features
    with explanations and examples.
    """,
    agent=researcher
)
```

### Task 2

```python
writing_task = Task(
    description="""
    Based on the research results, write
    a concise technical report about Java 21.
    """,
    expected_output="""
    A structured technical report.
    """,
    agent=writer
)
```

### 创建 Crew

```python
from crewai import Crew, Process

crew = Crew(
    agents=[
        researcher,
        writer
    ],
    tasks=[
        research_task,
        writing_task
    ],
    process=Process.sequential,
    verbose=True
)
```

### 执行

```python
result = crew.kickoff()

print(result)
```

执行过程大概是：

```text
User
 │
 │ "Research Java 21"
 ▼
Crew
 │
 ▼
Researcher Agent
 │
 │ Research
 ▼
Research Result
 │
 ▼
Writer Agent
 │
 │ Write report
 ▼
Final Result
```

---

# 9. CrewAI 怎么安装？

目前官方推荐使用 `uv` 管理环境，要求 Python 版本为 **3.10 到 3.13**。官方安装方式包括：

```bash
uv pip install crewai
```

如果需要额外工具：

```bash
uv pip install 'crewai[tools]'
```

([GitHub][2])

如果你习惯传统 Python：

```bash
pip install crewai
```

也可以，但如果你准备系统学习 CrewAI，我更建议：

```text
uv
+
Python 3.12
+
CrewAI
```

---

# 10. 创建 CrewAI 项目

官方现在提供 CLI。

例如：

```bash
crewai create flow my_ai_project
```

然后：

```bash
cd my_ai_project
```

项目结构大致类似：

```text
my_ai_project/
│
├── pyproject.toml
├── .env
│
└── src/
    └── my_ai_project/
        │
        ├── main.py
        │
        └── crews/
            │
            └── content_crew/
                │
                ├── content_crew.py
                │
                └── config/
                    ├── agents.yaml
                    └── tasks.yaml
```

官方 Quickstart 目前也是推荐通过 Flow 项目开始学习。([CreWai 文档][3])

---

# 11. YAML 定义 Agent

CrewAI 一个很方便的地方是：

**Agent 可以用 YAML 配置。**

例如：

```yaml
researcher:
  role: >
    Senior Java Researcher

  goal: >
    Research the latest Java technologies

  backstory: >
    You are a senior Java architect
    with extensive enterprise experience.
```

Task：

```yaml
research_task:
  description: >
    Research Java 21 features.

  expected_output: >
    A detailed technical report.
```

然后 Python：

```python
@agent
def researcher(self):
    return Agent(
        config=self.agents_config["researcher"]
    )
```

这样：

```text
业务配置
   ↓
YAML

执行逻辑
   ↓
Python
```

对于大型项目非常有用。

---

# 12. Crew 和 Flow 是 CrewAI 现在最重要的区别

这个概念你一定要理解。

CrewAI 早期很多教程主要讲：

```text
Agent
 ↓
Task
 ↓
Crew
```

现在官方更加重视：

```text
Flow
 ↓
Crew
 ↓
Agents
```

官方建议：

> **Flow 负责整个应用的流程控制，Crew 负责复杂的 Agent 协作。**

([CrewAI Documentation][1])

可以类比成：

```text
                 Flow
                  │
       ┌──────────┼──────────┐
       ↓          ↓          ↓
    Step 1      Crew       Step 3
                  │
          ┌───────┼───────┐
          ↓       ↓       ↓
        Agent   Agent   Agent
```

---

# 13. Flow 是什么？

例如：

```python
from crewai.flow.flow import Flow, start, listen

class ResearchFlow(Flow):

    @start()
    def start_research(self):
        print("Start research")

    @listen(start_research)
    def analyze(self):
        print("Analyze research")

    @listen(analyze)
    def generate_report(self):
        print("Generate report")
```

执行：

```python
ResearchFlow().kickoff()
```

流程：

```text
start_research()
       ↓
analyze()
       ↓
generate_report()
```

Flow 可以做：

```text
条件判断
   ↓
循环
   ↓
状态管理
   ↓
事件触发
   ↓
调用 Crew
   ↓
继续业务流程
```

官方文档把 Flow 定位为结构化、事件驱动的工作流，并支持状态、条件、循环等控制能力。([CreWai 文档][4])

---

# 14. Crew 和 Flow 怎么选择？

非常简单：

### Crew

适合：

```text
“让几个 AI 自己合作完成任务”
```

例如：

```text
研究一个技术
写文章
做市场分析
分析代码
生成报告
```

---

### Flow

适合：

```text
“我明确规定业务流程怎么走”
```

例如：

```text
接收订单
   ↓
检查库存
   ↓
调用 AI
   ↓
人工审核？
   ↓
YES → 人工审核
NO  → 自动执行
   ↓
保存数据库
```

---

### 两者结合

真正生产系统更可能是：

```text
                 Flow
                  │
          ┌───────┴───────┐
          ↓               ↓
       Business          Crew
        Logic              │
                           │
                 ┌─────────┼─────────┐
                 ↓         ↓         ↓
             Researcher  Analyst  Reviewer
```

这也是目前 CrewAI 官方推荐的方向。([CrewAI Documentation][1])

---

# 15. CrewAI 和 LangGraph 有什么区别？

可以先这样理解：

|          | CrewAI      | LangGraph      |
| -------- | ----------- | -------------- |
| 核心       | Multi-Agent | Agent Workflow |
| 语言       | Python      | Python / JS    |
| Agent 协作 | ⭐⭐⭐⭐⭐       | ⭐⭐⭐⭐           |
| 工作流控制    | ⭐⭐⭐⭐        | ⭐⭐⭐⭐⭐          |
| 上手难度     | 较低          | 较高             |
| 自主 Agent | 很强          | 很强             |
| 状态机      | ⭐⭐⭐         | ⭐⭐⭐⭐⭐          |
| 企业复杂流程   | ⭐⭐⭐⭐        | ⭐⭐⭐⭐⭐          |
| 多 Agent  | ⭐⭐⭐⭐⭐       | ⭐⭐⭐⭐           |
| 学习曲线     | 较平          | 较陡             |

最简单的理解：

```text
CrewAI

“AI 团队怎么合作？”
```

而：

```text
LangGraph

“AI Agent 的状态和执行流程怎么精确控制？”
```

当然现在两者的能力有不少重叠。

另外一个重要区别是：**CrewAI 是独立框架，不建立在 LangChain 之上**.

---

# 16.CrewAI

```text
Agent
Task
Crew
```


```text
             AI Application
                   │
        ┌──────────┴──────────┐
        │                     │
      Agent                 Workflow
        │                     │
     CrewAI                 Flow
        │                     │
 ┌──────┼──────┐       ┌──────┼──────┐
 │      │      │       │      │      │
LLM   Tool  Memory    State  Router  Event
 │      │
 │      ├── Web
 │      ├── DB
 │      ├── API
 │      └── RAG
 │
 └── OpenAI / Claude / Gemini / Ollama
```

然后把它和你熟悉的后端架构结合起来。

---

