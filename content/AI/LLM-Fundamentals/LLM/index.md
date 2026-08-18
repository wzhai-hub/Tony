---
title: LLM
# tags:
#   - nodejs
date: '2026-08-18'
summary: LLM，即 Large Language Model，本质上是一类通过大规模数据训练得到的神经网络模型
---
# LLM：从大语言模型到企业级 AI 应用的技术全景

## 引言

过去几年，Large Language Model（LLM，大语言模型）已经从一个人工智能领域的研究方向，逐渐演变为软件工程领域的重要基础设施。

如果把传统软件系统理解为：

> 用户请求 → 业务代码 → 数据库/缓存 → 返回结果

那么基于 LLM 的应用正在变成：

> 用户意图 → LLM → Tools / RAG / Memory / Agents → 企业系统 → LLM → 最终结果

这意味着，LLM 并不仅仅是一个“可以聊天的模型”。

真正值得软件工程师关注的是：

**如何围绕 LLM 构建可靠、可观测、可控、安全并且能够进入生产环境的 AI Application。**

从这个角度来看，可以把当前 LLM 技术体系划分成几个层次：

```text
┌──────────────────────────────────────────────┐
│              AI Application                 │
│   Chatbot / Copilot / Enterprise Agent      │
├──────────────────────────────────────────────┤
│              Agent Runtime                  │
│      LangGraph / CrewAI / Agent Framework    │
├──────────────────────────────────────────────┤
│       Orchestration / Workflow Layer        │
│       Workflow / HITL / State Machine        │
├──────────────────────────────────────────────┤
│          RAG / Tools / Memory               │
│   Vector DB / Search / API / Function Call  │
├──────────────────────────────────────────────┤
│              LLM Gateway                    │
│   OpenAI / Gemini / Claude / Azure OpenAI   │
├──────────────────────────────────────────────┤
│                 LLM                         │
│      GPT / Gemini / Claude / Llama          │
├──────────────────────────────────────────────┤
│          Infrastructure Layer               │
│ Kubernetes / GPU / Observability / Security │
└──────────────────────────────────────────────┘
```

理解这个分层，比单独学习某一个 Agent Framework 更重要。

---

# 一、LLM 到底是什么？

LLM，即 Large Language Model，本质上是一类通过大规模数据训练得到的神经网络模型。

从软件工程师的角度，可以先把它理解成一个非常强大的：

> **概率式语言生成系统。**

给定输入：

```text
Explain Java virtual machine GC.
```

模型会根据训练过程中学习到的参数，预测接下来最可能出现的 Token。

简单抽象为：

```text
Input
  ↓
Tokenizer
  ↓
Transformer
  ↓
Probability Distribution
  ↓
Next Token
  ↓
Repeat
  ↓
Output
```

例如：

```text
What is JVM?
```

可能生成：

```text
The JVM is...
```

然后继续预测下一个 Token。

因此，从最底层来看，LLM 并不是传统意义上的：

```java
String answer = database.query(question);
```

它更接近：

```text
P(next_token | previous_tokens)
```

通过不断预测 Token，最终生成完整回答。

---

# 二、Transformer：LLM 的核心基础

现代主流 LLM 基本都建立在 Transformer 架构之上。

Transformer 最重要的思想之一是：

> Attention。

传统 RNN 在处理长文本时，需要按照顺序处理：

```text
Token1 → Token2 → Token3 → Token4 → ...
```

而 Transformer 可以通过 Attention 建立不同 Token 之间的关系。

例如：

```text
The developer deployed the service because it was unstable.
```

模型需要判断：

```text
it
```

到底指向什么。

Attention 可以帮助模型建立：

```text
it ─────────→ service
```

这样的关联。

Transformer 通常可以抽象为：

```text
Input Tokens
     ↓
Embedding
     ↓
Self Attention
     ↓
Feed Forward Network
     ↓
Transformer Blocks
     ↓
Output
```

大型模型实际上就是堆叠了大量这样的计算结构，并通过海量数据进行训练。

---

# 三、Token：理解 LLM 的第一个关键概念

LLM 并不是直接处理“单词”。

它处理的是 Token。

例如：

```text
Hello world
```

可能被拆分成：

```text
Hello
world
```

中文：

```text
人工智能
```

也可能被拆成多个 Token。

因此 LLM 中经常看到：

```text
Context Window
Token
Input Tokens
Output Tokens
Max Tokens
```

这些概念。

例如一个模型支持：

```text
128K Context Window
```

意味着模型一次请求能够处理大约 128K Token 的上下文。

这也是为什么：

```text
Prompt 太长
```

会成为实际工程问题。

因为：

```text
Token 数量 ↑
        ↓
计算成本 ↑
        ↓
延迟 ↑
        ↓
Context 管理变得重要
```

---

# 四、Prompt Engineering 只是 LLM 应用的第一层

LLM 应用最早被大量讨论的是 Prompt Engineering。

例如：

```text
You are a Java expert.

Analyze the following code and identify
potential concurrency problems.

Code:
...
```

这种方式本质上是在通过自然语言告诉模型：

```text
Role
+
Context
+
Task
+
Constraints
+
Expected Output
```

一个比较成熟的 Prompt 通常包含：

```text
System Instruction
        +
User Input
        +
Context
        +
Examples
        +
Output Format
```

例如：

```text
System:
You are a senior Java architect.

Context:
The application uses Spring Boot and Redis.

Task:
Analyze the following architecture.

Constraints:
Focus on scalability and fault tolerance.

Output:
Return the result as Markdown.
```

但是需要注意：

**Prompt Engineering 并不是企业级 LLM 应用的终点。**

真正复杂的系统很快会遇到：

```text
模型不知道企业内部数据
模型不能访问数据库
模型不能调用业务 API
模型无法长期记住用户状态
模型输出不稳定
模型无法自主完成复杂任务
```

于是产生了下一层技术。

---

# 五、Function Calling：让 LLM 开始“使用工具”

LLM 本身并不能真正执行：

```text
查询数据库
发送邮件
创建 Jira
调用支付接口
修改 Kubernetes Deployment
```

但是可以让 LLM 产生结构化的 Tool Call。

例如：

```json
{
  "name": "get_weather",
  "arguments": {
    "city": "Shanghai"
  }
}
```

系统收到这个请求以后：

```text
LLM
 ↓
Tool Call
 ↓
Application
 ↓
Weather API
 ↓
Result
 ↓
LLM
 ↓
Final Answer
```

这实际上形成了一个闭环：

```text
Think
 ↓
Act
 ↓
Observe
 ↓
Think
 ↓
Act
```

这正是 Agent 系统的重要基础。

---

# 六、RAG：解决“模型不知道企业数据”的问题

LLM 有一个天然问题：

> 它并不知道你的企业内部数据。

例如公司内部存在：

```text
HR Policy
Architecture Documents
Customer Data
Product Manuals
Internal Wiki
Source Code
```

这些信息通常不会直接存在于基础模型中。

这时候可以使用 RAG：

> Retrieval-Augmented Generation

基本流程：

```text
User Question
      ↓
Embedding
      ↓
Vector Search
      ↓
Retrieve Documents
      ↓
Context
      ↓
LLM
      ↓
Answer
```

例如：

```text
用户：
公司的年假政策是什么？
```

系统首先搜索：

```text
HR Knowledge Base
```

找到：

```text
Annual Leave Policy.pdf
```

然后：

```text
Question
+
Relevant Documents
        ↓
       LLM
        ↓
     Answer
```

所以 RAG 的本质不是：

> “给 LLM 加一个数据库。”

而是：

> **在生成答案之前动态地向模型提供相关知识。**

---

# 七、RAG 并不等于 Vector Database

这是一个非常容易混淆的问题。

RAG 是一种应用架构。

Vector Database 只是 RAG 可能使用的一种基础设施。

完整的 RAG Pipeline 通常包括：

```text
Document
 ↓
Parsing
 ↓
Chunking
 ↓
Embedding
 ↓
Vector Store
 ↓
Retriever
 ↓
Reranker
 ↓
Context
 ↓
LLM
```

因此：

```text
RAG ≠ Vector Database
```

而应该理解为：

```text
RAG
├── Document Processing
├── Embedding
├── Retrieval
├── Reranking
├── Context Construction
└── Generation
```

这也是为什么生产环境中的 RAG 远比一个简单的：

```python
vector_db.similarity_search()
```

复杂。

---

# 八、Memory：Agent 为什么需要记忆？

传统 HTTP 服务通常是：

```text
Request
   ↓
Processing
   ↓
Response
```

每一次请求之间通常没有天然的长期记忆。

但是 Agent 往往需要：

```text
Conversation History
User Preferences
Task State
Intermediate Results
Long-term Memory
```

例如：

```text
User:
我喜欢 Java。

Agent:
好的。

User:
帮我推荐学习路线。

Agent:
根据你之前提到的 Java 背景...
```

系统需要保存：

```text
User Preference
       ↓
Memory
       ↓
Future Conversation
```

因此 AI Application 开始出现新的状态管理问题：

```text
Conversation State
        +
Agent State
        +
Business State
```

这也是 Agent Framework 与传统 Chatbot 最大的区别之一。

---

# 九、Agent：从“回答问题”走向“完成任务”

Chatbot 的典型模式：

```text
User
 ↓
LLM
 ↓
Answer
```

Agent 则更像：

```text
User
 ↓
Agent
 ↓
Plan
 ↓
Tool
 ↓
Observe
 ↓
Reason
 ↓
Tool
 ↓
Observe
 ↓
Final Answer
```

例如用户说：

```text
帮我分析昨天的生产事故，
找出根因并创建 Jira Ticket。
```

Agent 可能执行：

```text
1. 查询 Incident System
2. 获取日志
3. 查询 Grafana
4. 分析 Trace
5. 判断 Root Cause
6. 生成 Incident Report
7. 创建 Jira Ticket
8. 返回结果
```

这已经不是简单的文本生成。

它更接近：

> **AI-driven Workflow Execution**

---

# 十、为什么 Human-in-the-Loop 非常重要？

Agent 最大的问题之一是：

> AI 可以行动，但 AI 不应该拥有无限制的行动权限。

例如：

```text
Delete Database
Transfer Money
Deploy Production
Send Customer Email
Terminate VM
```

这些操作不应该让 Agent 自动执行。

因此需要：

```text
Agent
  ↓
Proposed Action
  ↓
Human Approval
  ↓
Execute
```

例如：

```text
Agent:
I recommend deleting 500 obsolete records.

Human:
Approve

System:
Execute deletion
```

这就是：

> Human-in-the-loop（HITL）

它实际上是企业级 Agent 架构中非常重要的一层。

可以把它理解为：

```text
AI Autonomy
      ↓
Human Governance
      ↓
Controlled Execution
```

这也是为什么企业 Agent 最终往往不是追求：

> 100% Autonomous

而是追求：

> **Controlled Autonomy**

---

# 十一、LangChain、LangGraph、CrewAI 到底属于什么？

如果从架构层次理解，这几个技术就非常容易区分。

### LangChain

主要解决：

```text
LLM Application Development
```

包括：

```text
Prompt
Model
Tool
Retriever
Memory
Chain
Agent
```

它更像一个 AI Application Development Framework。

### LangGraph

更强调：

```text
State
Graph
Workflow
Agent
Human-in-the-loop
```

例如：

```text
START
  ↓
Analyze
  ↓
Need Tool?
 ├── Yes → Tool
 │          ↓
 │       Analyze
 │
 └── No → Human Review
              ↓
             END
```

它非常适合复杂、有状态、需要人工介入的 Agent Workflow。

### CrewAI

更强调：

```text
Multi-Agent
Role
Task
Crew
Collaboration
```

例如：

```text
Research Agent
       ↓
Writer Agent
       ↓
Reviewer Agent
       ↓
Publisher Agent
```

因此可以简单理解：

```text
LangChain
    ↓
LLM Application Components

LangGraph
    ↓
Stateful Agent Workflow

CrewAI
    ↓
Multi-Agent Collaboration
```

三者并不是简单的竞争关系。

---

# 十二、OpenAI Python SDK 属于哪一层？

OpenAI Python SDK 更底层。

它主要解决：

```text
Application
    ↓
OpenAI SDK
    ↓
OpenAI API
    ↓
LLM
```

例如：

```python
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-5",
    input="Explain JVM garbage collection."
)

print(response.output_text)
```

这里 SDK 的职责非常明确：

> **让应用程序方便地调用模型能力。**

所以：

```text
OpenAI SDK
    ↓
Model Access Layer
```

而：

```text
LangChain
LangGraph
CrewAI
```

通常位于更高层。

---

# 十三、Harness 属于什么？

在 AI 系统中，Harness 可以理解成围绕 Agent/LLM 建立的一层：

```text
Control
Execution
Safety
Evaluation
Observability
Governance
```

简单来说：

```text
LLM
 ↓
Agent
 ↓
Harness
 ↓
Tools / Enterprise Systems
```

Harness 的价值在于：

> 不让 Agent 直接、无限制地控制真实系统。

例如：

```text
Agent wants to:
    deploy production

Harness:
    Check permission
    Check policy
    Check environment
    Require approval
    Execute
    Record audit log
```

因此，从企业架构角度看：

```text
LLM = Intelligence

Agent = Decision / Planning

Tools = Capabilities

Harness = Control Plane

Human = Governance
```

这是理解企业级 AI Architecture 非常重要的一组概念。

---

# 十四、从 LLM 到 Agentic AI 的完整演进

整个技术演进可以总结为：

```text
LLM
 ↓
Prompt Engineering
 ↓
RAG
 ↓
Tool Calling
 ↓
Workflow
 ↓
Agent
 ↓
Multi-Agent
 ↓
Human-in-the-loop
 ↓
Agent Governance
 ↓
Enterprise Agent Platform
```

也就是说：

```text
LLM
```

解决的是：

> “理解和生成语言。”

而：

```text
RAG
```

解决：

> “获取外部知识。”

```text
Tools
```

解决：

> “与外部世界交互。”

```text
Agent
```

解决：

> “自主完成复杂任务。”

```text
HITL
```

解决：

> “人类如何控制 AI。”

```text
Harness / Governance
```

解决：

> “如何让 Agent 安全进入生产环境。”

---

# 十五、企业级 LLM Architecture

如果设计一个真正的企业级 AI Platform，可以考虑如下架构：

```text
                    User
                      │
                      ▼
                API Gateway
                      │
                      ▼
              AI Application
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
       Agent                  Workflow
          │                       │
          └───────────┬───────────┘
                      │
                 Agent Runtime
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
      Tools          RAG          Memory
        │             │             │
        ▼             ▼             ▼
   Enterprise     Vector DB      State Store
    Systems       Search DB
                      │
                      ▼
                 LLM Gateway
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
        GPT        Gemini       Claude
```

同时还需要横向能力：

```text
Security
Observability
Evaluation
Governance
Audit
Cost Control
Rate Limiting
```

最终形成：

```text
             Enterprise AI Platform

 ┌──────────────────────────────────────────┐
 │ Security / Governance / Audit            │
 ├──────────────────────────────────────────┤
 │ Observability / Evaluation / Monitoring  │
 ├──────────────────────────────────────────┤
 │ Agent / Workflow / HITL                  │
 ├──────────────────────────────────────────┤
 │ RAG / Memory / Tools                     │
 ├──────────────────────────────────────────┤
 │ LLM Gateway                              │
 ├──────────────────────────────────────────┤
 │ GPT / Gemini / Claude / Open Models      │
 └──────────────────────────────────────────┘
```

这已经越来越接近传统企业软件平台架构。

---

# 十六、LLM 应用最大的工程挑战

当 LLM 从 Demo 进入 Production，真正的问题通常不是：

> “怎么调用 GPT？”

而是：

### 1. Reliability

同样的 Prompt，模型可能产生不同结果。

因此需要：

```text
Evaluation
Guardrails
Structured Output
Fallback
Retry
Validation
```

### 2. Latency

一次 Agent 请求可能：

```text
LLM
 ↓
Tool
 ↓
LLM
 ↓
RAG
 ↓
LLM
```

调用链越来越长。

因此：

```text
Caching
Parallel Execution
Streaming
Model Routing
Timeout
```

都会变得重要。

### 3. Cost

Token 是成本。

所以需要：

```text
Token Management
Model Selection
Prompt Optimization
Caching
Batching
```

### 4. Security

Agent 一旦能够调用：

```text
Database
API
Shell
Cloud
Kubernetes
```

安全问题会被放大。

因此需要：

```text
Authentication
Authorization
Least Privilege
Sandbox
Approval
Audit
```

### 5. Observability

传统微服务观察：

```text
CPU
Memory
Latency
Error Rate
Trace
```

AI 系统还需要观察：

```text
Prompt
Token
Model
Tool Call
Retrieved Documents
Agent Steps
Reasoning State
Evaluation Score
```

所以 AI Observability 会成为传统 Observability 的扩展。

---

