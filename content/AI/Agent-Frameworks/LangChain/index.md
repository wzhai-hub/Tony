---
title: LangChain
# tags:
#   - nodejs
date: '2026-08-05'
summary: LangChain是一个用于构建大语言模型（LLM）应用的开源开发框架
---

**LangChain** 是一个用于构建 **大语言模型（LLM）应用** 的开源开发框架。它最初主要支持 Python，后来也推出了 JavaScript/TypeScript 版本，目的是让开发者能够更方便地将大模型与外部数据、工具和业务流程结合起来，而不仅仅是简单地调用模型 API。

可以把它理解为：

> **如果 OpenAI、Anthropic 等提供的是“大脑”，那么 LangChain 提供的是把大脑连接到工具、数据库和业务系统的“神经系统”。**

## LangChain 能做什么？

它主要解决以下几类问题：

### 1. 统一调用各种大模型

不管你使用的是：

* OpenAI GPT
* Anthropic Claude
* Google Gemini
* DeepSeek
* 本地模型（如 Llama）

LangChain 提供了统一的接口，方便切换模型，而不用大改代码。

例如：

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4.1")
response = llm.invoke("介绍一下LangChain")
```

---

### 2. Prompt 管理

把 Prompt 模板化，而不是字符串拼接。

例如：

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_template(
    "请把下面内容翻译成英文：{text}"
)

prompt.invoke({
    "text": "你好"
})
```

这样方便维护复杂 Prompt。

---

### 3. 连接外部数据（RAG）

这是 LangChain 最流行的用途之一。

例如：

```
PDF
Word
数据库
网页
知识库
       │
       ▼
 文档切分
       │
       ▼
Embedding
       │
       ▼
向量数据库
       │
       ▼
LLM回答问题
```

这就是大家常说的 **RAG（Retrieval-Augmented Generation，检索增强生成）**。

常用组件包括：

* Document Loader
* Text Splitter
* Embedding
* Vector Store
* Retriever

---

### 4. 调用工具（Tools）

例如：

模型不会算：

```
345678 × 2345
```

它可以调用：

* Python
* 搜索引擎
* 天气 API
* 数据库
* 企业接口

例如：

```
用户：
北京天气

↓

LLM

↓

Weather API

↓

返回天气

↓

LLM组织语言回复
```

---

### 5. Agent（智能体）

Agent 是 LangChain 的核心能力之一。

例如用户说：

```
帮我查一下苹果公司股价，然后写一份分析。
```

Agent 可以：

```
① 理解任务

↓

② 调用股票API

↓

③ 获取价格

↓

④ 分析数据

↓

⑤ 输出报告
```

Agent 会决定：

* 是否调用工具
* 调哪个工具
* 调几次

---

### 6. Memory（对话记忆）

例如：

```
用户：
我叫Tom

AI：
好的。

用户：
我叫什么？
```

Memory 可以保存上下文。

不过，现代应用通常更倾向于自己管理对话历史，而不是依赖 LangChain 早期的 Memory 抽象。

---

## LangChain 的核心模块

```
LangChain
│
├── Models（模型）
│
├── Prompts（提示词）
│
├── Chains（流程）
│
├── Tools（工具）
│
├── Agents（智能体）
│
├── Retrievers（检索）
│
├── Vector Stores（向量库）
│
└── Output Parsers（输出解析）
```

---

## 一个简单流程

例如：

```
用户提问

↓

Prompt

↓

LLM

↓

调用搜索工具

↓

LLM

↓

返回答案
```

或者：

```
用户

↓

RAG

↓

检索知识库

↓

LLM

↓

回答
```

---

## LangChain 与其他框架的区别

| 框架                | 特点                             | 适合场景                 |
| ----------------- | ------------------------------ | -------------------- |
| LangChain         | 功能全面、生态成熟                      | RAG、Agent、多模型集成      |
| LangGraph         | 基于状态机/图的 Agent 编排，更适合复杂、多步骤工作流 | 多智能体、长流程应用           |
| LlamaIndex        | 更专注于数据接入和 RAG                  | 知识库问答、文档检索           |
| OpenAI Agents SDK | 与 OpenAI 模型和工具深度集成             | 基于 OpenAI 生态开发 Agent |

---

## LangChain 的优点

* 支持多种模型，切换成本低。
* 提供丰富的组件，覆盖 RAG、Agent、工具调用等常见场景。
* 社区活跃，生态完善，与众多向量数据库、模型和第三方服务集成良好。

## 需要注意的地方

* 学习曲线相对较陡，概念和组件较多。
* API 在近几年迭代较快，不同版本之间可能存在较大变化。
* 对于简单应用（例如单次调用模型），直接使用模型官方 SDK 往往更轻量，不一定需要引入 LangChain。

### 什么时候该用 LangChain？

如果你的项目只是：

* 调用一次 LLM 完成聊天、翻译或总结；

直接使用模型官方 SDK 通常已经足够。

如果你的项目需要：

* 接入企业知识库（RAG）；
* 调用搜索、数据库、API 等外部工具；
* 构建多步骤工作流或 Agent；
* 在不同模型之间灵活切换；

那么 LangChain 能显著减少开发工作量，并提供更好的模块化能力。随着近年来 Agent 应用的发展，很多开发者也会结合 **LangGraph** 使用 LangChain，以构建更复杂、更可靠的智能体系统。

