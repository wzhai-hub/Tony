---
title: RAG
# tags:
#   - nodejs
date: '2026-08-05'
summary: RAG = 先从外部知识库找到相关资料，再把资料交给大模型，让大模型基于这些资料回答问题。
---

## 一、什么是 RAG？

**RAG = Retrieval-Augmented Generation**

中文通常叫：

> **检索增强生成**

简单来说：

> **RAG = 先从外部知识库找到相关资料，再把资料交给大模型，让大模型基于这些资料回答问题。**

传统 ChatGPT 类大模型主要依赖训练阶段学到的知识：

```text
用户问题
   ↓
LLM
   ↓
生成答案
```

RAG 则变成：

```text
用户问题
   ↓
检索知识库
   ↓
找到相关资料
   ↓
把资料 + 用户问题
   ↓
LLM
   ↓
生成答案
```

例如你有一个公司内部知识库：

```text
公司员工手册.pdf
Java开发规范.pdf
项目架构文档.pdf
HR政策.pdf
产品说明书.pdf
```

用户问：

> “公司的年假政策是什么？”

传统 LLM：

```text
问题 → LLM → 根据训练知识猜答案
```

RAG：

```text
问题
 ↓
Embedding
 ↓
Vector Search
 ↓
找到 HR政策.pdf 中相关内容
 ↓
把相关内容交给 LLM
 ↓
LLM 根据公司真实文档回答
```

所以 RAG 的核心价值是：

> **让大模型能够使用“外部知识”。**

---

# 二、为什么需要 RAG？

RAG 主要解决 LLM 的几个问题。

### 1. LLM 不知道你的私有数据

例如：

```text
公司内部代码
公司数据库
内部技术文档
员工手册
客户资料
产品文档
项目 Wiki
```

这些通常没有参与公共大模型训练。

RAG 可以让 LLM 查询这些资料。

---

### 2. LLM 的知识存在时效性

例如：

```text
2026年的公司政策
2026年的产品价格
最新 API 文档
最新项目代码
最新数据库数据
```

模型训练数据不一定包含这些信息。

RAG 可以实时检索最新资料。

---

### 3. 减少 Hallucination（幻觉）

例如用户问：

> “我们公司的退款政策是什么？”

如果 LLM 不知道，很容易：

```text
LLM：
根据公司政策，退款期限是30天……
```

但是这个答案可能是**编造的**。

RAG 可以要求：

```text
只根据检索出来的公司政策回答
```

因此可以显著降低幻觉。

注意：

> **RAG 不能保证 100% 消除幻觉。**

---

# 三、RAG 的核心架构

一个典型 RAG 系统可以理解成两个阶段：

```text
                ┌───────────────┐
                │  Documents    │
                └───────┬───────┘
                        ↓
                  Document Loader
                        ↓
                    Chunking
                        ↓
                   Embedding
                        ↓
                ┌───────────────┐
                │ Vector Database│
                └───────────────┘
                        ↑
                        │
User Question → Embedding
                        ↓
                   Similarity Search
                        ↓
                 Relevant Documents
                        ↓
                Prompt + Context
                        ↓
                      LLM
                        ↓
                     Answer
```

其中非常重要的组件有：

```text
Document Loader
Chunking
Embedding
Vector Database
Retriever
Reranker
Prompt
LLM
```

---

# 四、RAG 有哪些主要功能？

如果从企业级 RAG 系统来看，我建议你把功能分成 **10 个部分**来理解。

## 1. 文档导入

RAG 首先需要获取知识。

例如：

```text
PDF
Word
Excel
TXT
Markdown
HTML
网页
数据库
Git Repository
API
Confluence
SharePoint
```

例如：

```text
Java-Interview.pdf
Spring-Boot.pdf
Company-Handbook.pdf
Architecture.md
```

---

# 2. Document Parsing

把各种格式转换成文本。

例如：

```text
PDF
 ↓
PDF Parser
 ↓
Text
```

比如：

```text
PDF：

Chapter 1
Spring Boot Introduction

Chapter 2
Spring Security
```

转换成：

```text
Spring Boot Introduction
Spring Security
```

企业 RAG 中，这一步其实非常重要。

因为现实中的 PDF 可能包含：

```text
文字
表格
图片
页眉
页脚
代码
扫描件
```

所以需要比较复杂的 Document Parsing。

---

# 3. Chunking

这是 RAG 最核心的功能之一。

假设一个 PDF 有：

```text
500 pages
```

不能直接把整个 PDF 给 LLM。

需要拆成很多小块：

```text
Document
   ↓
Chunk 1
Chunk 2
Chunk 3
Chunk 4
...
Chunk 1000
```

例如：

```text
Chunk 1:
Spring Boot 是一个快速开发框架……

Chunk 2:
Spring Boot 自动配置机制……

Chunk 3:
Spring Boot Starter……

Chunk 4:
Spring Boot Actuator……
```

为什么要 Chunk？

因为：

```text
LLM Context Window
```

是有限的。

而且 Chunk 越精准，检索结果通常越准确。

---

# 4. Embedding

这是理解 RAG 的另一个核心概念。

Embedding 的作用是：

> **把文字转换成向量。**

例如：

```text
"Java Spring Boot"
```

可能转换成：

```text
[0.12, -0.53, 0.82, 0.11, ...]
```

另一个句子：

```text
"Spring Boot framework"
```

可能：

```text
[0.14, -0.50, 0.79, 0.13, ...]
```

两个向量比较接近。

说明：

```text
Java Spring Boot
        ≈
Spring Boot framework
```

语义比较相似。

---

# 5. Vector Database

Embedding 之后，需要保存向量。

这就是：

> **Vector Database（向量数据库）**

常见的有：

* pgvector
* Milvus
* Pinecone
* Weaviate
* Qdrant
* Elasticsearch
* OpenSearch
* Redis
* azure openAI search

如果你本身已经熟悉 **Redis、PostgreSQL、Elasticsearch**，其实非常有优势。

例如 PostgreSQL：

```text
Document
   ↓
Chunk
   ↓
Embedding
   ↓
PostgreSQL + pgvector
```

就可以构建一个 RAG 知识库。

---

# 6. Retrieval

用户提出问题：

> “Spring Boot 如何实现事务？”

首先把问题进行 Embedding：

```text
Question
   ↓
Embedding
   ↓
Vector
```

然后去 Vector Database 搜索：

```text
Similarity Search
```

找到：

```text
Chunk 27
Chunk 81
Chunk 125
Chunk 322
```

这些就是：

> Relevant Context

---

# 7. Reranking

仅仅 Vector Search 有时候还不够。

例如搜索：

```text
Spring Boot transaction
```

Vector Search 找到：

```text
Document A
Document B
Document C
Document D
Document E
```

但是相关性可能不是特别准确。

于是可以再使用：

```text
Retriever
   ↓
Top 20
   ↓
Reranker
   ↓
Top 5
```

Reranker 会重新判断：

```text
Query
    ↓
Document 1 → 95%
Document 2 → 87%
Document 3 → 63%
Document 4 → 42%
```

最终只把最相关的内容交给 LLM。

这在企业级 RAG 中非常重要。

---

# 8. Context Injection

得到相关文档以后，把它们放进 Prompt。

例如：

```text
System:
You are a Java expert.

Context:
Spring transaction is implemented using...

User:
How does Spring @Transactional work?
```

最终：

```text
Context + Question
       ↓
      LLM
       ↓
    Answer
```

这就是：

> **Augmented Generation**

也就是 RAG 中的 **AG**。

---

# 9. Answer Generation

LLM 最终根据：

```text
用户问题
+
检索结果
+
System Prompt
```

生成答案。

例如：

```text
用户：
Spring @Transactional 为什么会失效？

RAG：
检索 Spring 官方文档
        ↓
找到 @Transactional proxy 机制
        ↓
LLM
        ↓
回答：
因为 @Transactional 通常基于 Spring AOP Proxy，
self-invocation 不会经过 proxy……
```

这样回答的依据就来自知识库。

---

# 10. Citation / Source

企业级 RAG 很重要的功能：

> **告诉用户答案来自哪里。**

例如：

```text
答案：

Spring 的 @Transactional 默认基于代理机制。

来源：
Spring Transaction Management
第 3.2 节
```

甚至可以：

```text
Answer
  ↓
Source 1
Source 2
Source 3
```

用户可以点击来源查看原始文档。

这对企业系统非常重要，因为它可以提高：

```text
可信度
可审计性
可追溯性
```

---

# 五、RAG 的完整流程

你可以把整个 RAG 记成：

```text
                 【离线阶段】

Documents
   ↓
Parse
   ↓
Chunk
   ↓
Embedding
   ↓
Vector Database


                 【在线阶段】

User Question
      ↓
Query Understanding
      ↓
Embedding
      ↓
Vector Search
      ↓
Keyword Search
      ↓
Hybrid Search
      ↓
Reranking
      ↓
Top-K Context
      ↓
Prompt Construction
      ↓
LLM
      ↓
Answer
      ↓
Citation
```

---

# 六、RAG 不只是 Vector Search

这是很多初学者容易误解的地方。

很多人认为：

> RAG = Vector Database

实际上：

> **Vector Database 只是 RAG 的一个组件。**

现代 RAG 通常还会使用：

### 1. Vector Search

```text
语义搜索
```

### 2. Keyword Search

例如：

```text
BM25
```

适合：

```text
Java Class Name
API Name
Error Code
Exception
Product ID
```

比如：

```text
NullPointerException
ERR-50001
@Transactional
```

这种精确关键词搜索有时候比 Vector Search 更好。

---

### 3. Hybrid Search

把：

```text
Vector Search
+
Keyword Search
```

结合起来。

例如：

```text
Query
 ↓
 ┌──────────────┐
 │ Vector Search│
 └──────┬───────┘
        │
        ├─────┐
        │     │
 ┌──────▼─────▼──┐
 │ Hybrid Search │
 └──────┬────────┘
        ↓
     Reranker
        ↓
      LLM
```

这是目前企业 RAG 很常见的架构。

---

# 七、RAG 可以解决哪些实际问题？

你作为 Java / Full Stack 开发者，可以重点关注这些场景。

### 企业知识库

```text
员工问：
公司的报销政策是什么？

        ↓

RAG
        ↓

HR Documents
        ↓

回答
```

### IT Support

```text
用户：
这个错误怎么解决？

        ↓

RAG
        ↓

Knowledge Base
        ↓

找到历史解决方案
        ↓

LLM
        ↓

生成解决方案
```

### Code Assistant

```text
开发人员：
我们的 PaymentService 怎么调用？

        ↓

RAG
        ↓

Git Repository
        ↓

检索代码
        ↓

LLM
        ↓

解释代码
```

### Customer Service

```text
客户问题
    ↓
产品文档
    ↓
FAQ
    ↓
历史知识
    ↓
RAG
    ↓
AI Customer Service
```

### 企业内部搜索

传统搜索：

```text
关键词 → 文档
```

RAG：

```text
自然语言问题
      ↓
理解语义
      ↓
搜索知识
      ↓
生成答案
```

所以它实际上可以变成：

> **AI Search Engine**

---

# 八、RAG 和 Fine-tuning 的区别

这是 AI 面试非常常见的问题。

|        | RAG    | Fine-tuning |
| ------ | ------ | ----------- |
| 目的     | 增加外部知识 | 改变模型行为/能力   |
| 数据变化   | 很容易更新  | 需要重新训练      |
| 私有知识   | 非常适合   | 可以，但成本高     |
| 实时数据   | 很适合    | 不适合         |
| 成本     | 相对低    | 相对高         |
| 可追溯    | 很好     | 较差          |
| 最新知识   | 很好     | 较差          |
| 改变回答风格 | 一般     | 很好          |
| 改变模型能力 | 有限     | 更适合         |

简单记：

> **知识问题 → RAG**

> **行为/能力问题 → Fine-tuning**

例如：

```text
公司的最新员工手册
        ↓
       RAG
```

而：

```text
让模型始终使用某种特殊格式回答
        ↓
Fine-tuning
```

当然，实际项目中也可以：

```text
RAG + Fine-tuning
```

一起使用。

---

