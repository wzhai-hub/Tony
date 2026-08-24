---
title: Transformer：从 Self-Attention 到现代大语言模型
# tags:
#   - nodejs
date: '2026-08-05'
summary: Transformer 是过去十年人工智能领域最重要的神经网络架构之一。从机器翻译、BERT、GPT，到今天的 ChatGPT、代码模型、多模态模型和 Agent，现代生成式 AI 的核心技术栈几乎都建立在 Transformer 及其衍生架构之上
---

>
> Transformer 是过去十年人工智能领域最重要的神经网络架构之一。从机器翻译、BERT、GPT，到今天的 ChatGPT、代码模型、多模态模型和 Agent，现代生成式 AI 的核心技术栈几乎都建立在 Transformer 及其衍生架构之上。
>
> 本文不把 Transformer 简单理解为一个“神经网络模型”，而是从 **序列建模 → Attention → Self-Attention → Multi-Head Attention → Transformer Block → Encoder/Decoder → GPT → LLM → KV Cache → MoE** 的完整技术链路进行分析。

---

# 1. Transformer 到底解决了什么问题？

在 Transformer 出现之前，NLP 的主流模型之一是 RNN/LSTM。

例如：

```text
I → love → artificial → intelligence
```

RNN 按顺序处理：

```text
x1 → RNN → h1
          ↓
x2 → RNN → h2
          ↓
x3 → RNN → h3
          ↓
x4 → RNN → h4
```

存在两个核心问题。

## 问题一：无法充分并行

因为：

```text
h2 依赖 h1
h3 依赖 h2
h4 依赖 h3
```

所以：

```text
h1 → h2 → h3 → h4
```

必须按照顺序执行。

GPU 最擅长：

```text
Parallel Computation
```

而 RNN 的时间依赖限制了并行能力。

---

## 问题二：长期依赖困难

例如：

```text
The book that I bought yesterday from the bookstore
was expensive.
```

模型在预测：

```text
was
```

时，需要理解：

```text
book → was
```

但是两个词之间可能相隔几十甚至几百个 Token。

RNN 需要：

```text
h1 → h2 → h3 → ... → h100
```

逐步传播信息。

Transformer 的思路完全不同：

> **不要强迫信息逐步传播，而是让 Token 直接访问其他 Token。**

这就是：

# Attention

---

# 2. Attention 的核心思想

Attention 可以理解成：

> **当前 Token 应该关注哪些其他 Token？**

例如：

```text
The animal didn't cross the street because it was too tired.
```

这里：

```text
it
```

究竟指：

```text
animal
```

还是：

```text
street
```

模型需要根据上下文建立关联。

Attention 可以计算：

```text
it → animal
```

的高相关性。

因此：

```text
RNN：

Token
 ↓
Hidden State
 ↓
Next Token


Transformer：

Token
 ↓
Attention
 ↙ ↓ ↘
Token Token Token
```

---

# 3. Self-Attention

Transformer 最核心的技术：

> **Self-Attention**

所谓 Self，是因为：

> Query、Key、Value 都来自同一个输入序列。

假设：

```text
X = [x1, x2, x3, x4]
```

Self-Attention 会产生：

```text
x1 → 看 x1 x2 x3 x4
x2 → 看 x1 x2 x3 x4
x3 → 看 x1 x2 x3 x4
x4 → 看 x1 x2 x3 x4
```

也就是说：

> 每个 Token 都可以和其他 Token 建立关系。

这就是 Transformer 强大的核心原因。

---

# 4. Query、Key、Value

Self-Attention 最重要的三个概念：

```text
Q = Query
K = Key
V = Value
```

可以用数据库查询来类比。

假设你搜索：

```text
"Java concurrency"
```

可以理解为：

```text
Query = Java concurrency
```

数据库中的：

```text
Key
```

用于匹配。

匹配之后取出：

```text
Value
```

Transformer 中也是类似思想。

---

# 5. Q、K、V 如何产生？

输入：

```text
X
```

经过三个不同的线性变换：

```text
Q = XW_Q

K = XW_K

V = XW_V
```

其中：

```text
W_Q
W_K
W_V
```

都是模型需要训练的参数。

所以：

```text
Input
  │
  ├──→ W_Q → Q
  │
  ├──→ W_K → K
  │
  └──→ W_V → V
```

这是 Transformer 最重要的基础计算之一。

---

# 6. Attention Score

有了 Q 和 K 后，需要计算：

> Query 与 Key 到底有多相关？

使用：

```text
QKᵀ
```

例如：

```text
Q = [q1, q2, q3]

K = [k1, k2, k3]
```

计算：

```text
QKᵀ
```

得到：

```text
          k1   k2   k3

q1        8    2    1
q2        1    7    3
q3        2    2    9
```

这个矩阵表示：

```text
Token1 对 Token1/2/3 的关注程度
Token2 对 Token1/2/3 的关注程度
Token3 对 Token1/2/3 的关注程度
```

---

# 7. 为什么要除以 sqrt(d_k)？

Transformer 的经典 Attention 公式：

```text
Attention(Q,K,V)
=
softmax(
QKᵀ / √d_k
)V
```

这里：

```text
d_k
```

是 Key 的维度。

为什么需要：

```text
√d_k
```

？

因为当维度变大时：

```text
QKᵀ
```

的数值可能越来越大。

例如：

```text
d_k = 64
```

如果不进行缩放：

```text
QKᵀ
```

可能产生非常大的数值。

经过 Softmax 后：

```text
[0.000001, 0.999998, 0.000001]
```

容易导致：

> Softmax 饱和。

梯度也可能变得非常小。

因此：

```text
QKᵀ / √d_k
```

可以让数值保持在更加稳定的范围。

这就是：

> **Scaled Dot-Product Attention**

---

# 8. Softmax

得到 Attention Score 后：

```text
S = QKᵀ / √d_k
```

需要转换成概率分布：

```text
A = softmax(S)
```

例如：

```text
[2.0, 1.0, 0.1]
```

经过 Softmax：

```text
[0.66, 0.24, 0.10]
```

于是模型可以理解：

```text
Token A
 ↓
66% attention → Token1
24% attention → Token2
10% attention → Token3
```

这就是：

> Attention Weight

---

# 9. 最终 Attention 输出

最后：

```text
Attention(Q,K,V)
=
softmax(
QKᵀ / √d_k
)V
```

实际上就是：

```text
Attention Weight
        ×
       Value
        ↓
Weighted Sum
```

例如：

```text
Attention Weight:

[0.7, 0.2, 0.1]
```

Value：

```text
V1
V2
V3
```

最终：

```text
Output
=
0.7V1
+
0.2V2
+
0.1V3
```

因此 Attention 本质上是：

> **根据相关性，对 Value 做加权聚合。**

---

# 10. Self-Attention 的完整流程

完整过程：

```text
Input X
   │
   ├─────────────┐
   │             │
   ↓             ↓
  W_Q           W_K
   ↓             ↓
   Q             K
    \           /
     \         /
       QKᵀ
        │
        ↓
      Scale
        │
        ↓
      Softmax
        │
        ↓
 Attention Weights
        │
        ×
        │
        V
        │
        ↓
      Output
```

公式：

```text
Attention(Q,K,V)
=
softmax(
QKᵀ / √d_k
)V
```

如果只记住 Transformer 一个公式：

> **记住这个。**

---

# 11. Multi-Head Attention

单个 Attention Head 只能学习一种关系。

但是自然语言中存在很多不同类型的关系：

```text
语法关系
语义关系
指代关系
位置关系
实体关系
上下文关系
```

因此 Transformer 不使用一个 Attention，而是：

> **Multi-Head Attention**

例如：

```text
Head 1 → 语法关系
Head 2 → 语义关系
Head 3 → 指代关系
Head 4 → 长距离关系
```

---

# 12. Multi-Head Attention 的结构

假设：

```text
d_model = 512
```

使用：

```text
8 heads
```

那么每个 Head：

```text
d_head = 512 / 8 = 64
```

结构：

```text
                 Input
                   │
        ┌──────────┼──────────┐
        ↓          ↓          ↓
      Head1      Head2       Head8
        │          │           │
        ↓          ↓           ↓
      Attn       Attn        Attn
        │          │           │
        └──────────┼───────────┘
                   ↓
               Concat
                   ↓
                Linear
                   ↓
                Output
```

公式：

```text
MultiHead(Q,K,V)
=
Concat(head1,...,headh)W_O
```

其中：

```text
head_i
=
Attention(
QW_Q^i,
KW_K^i,
VW_V^i
)
```

---

# 13. 为什么 Multi-Head 有意义？

假设一句话：

```text
The developer fixed the bug because he understood the system.
```

不同 Head 可以学习不同关系：

```text
Head 1:
developer → he

Head 2:
fixed → bug

Head 3:
understood → system
```

因此：

> Multi-Head Attention 让模型能够在不同表示子空间中同时学习不同类型的关系。

---

# 14. Positional Encoding

Transformer 有一个非常重要的问题：

> Attention 本身不知道 Token 的顺序。

例如：

```text
I love AI
```

和：

```text
AI love I
```

如果只看 Token 集合，Self-Attention 本身无法天然区分顺序。

因此必须加入：

> Position Information

---

# 15. 原始 Transformer 的 Positional Encoding

论文中的经典方法：

```text
PE(pos, 2i)
=
sin(
pos / 10000^(2i/d_model)
)
```

```text
PE(pos, 2i+1)
=
cos(
pos / 10000^(2i/d_model)
)
```

最终：

```text
Input Embedding
+
Position Encoding
```

得到：

```text
Transformer Input
```

---

# 16. 为什么使用 Sin/Cos？

因为正弦和余弦具有连续周期结构。

模型可以利用：

```text
sin
cos
```

表示不同位置。

例如：

```text
Position 1
Position 2
Position 3
...
```

产生不同的向量。

而且不同维度具有不同频率。

因此可以编码：

```text
绝对位置
+
相对位置信息
```

---

# 17. 现代 Transformer 的位置编码

需要特别注意：

> 现代 LLM 并不一定使用原始 Transformer 的 Sin/Cos Positional Encoding。

例如很多现代模型采用：

```text
RoPE
```

即：

> Rotary Positional Embedding

另外还有：

```text
ALiBi
Relative Position Encoding
```

等方法。

尤其在现代 GPT 类模型中：

> **RoPE 是理解现代 LLM 架构非常重要的知识点。**

---

# 18. Transformer Encoder

经典 Transformer 论文：

> Attention Is All You Need

提出了 Encoder-Decoder 架构。

Encoder：

```text
Input
 ↓
Embedding
 ↓
Positional Encoding
 ↓
Encoder Block
 ↓
Encoder Block
 ↓
...
 ↓
Output
```

每个 Encoder Block 主要包含：

```text
Multi-Head Self-Attention
        ↓
Add & Norm
        ↓
Feed Forward Network
        ↓
Add & Norm
```

---

# 19. Transformer Encoder Block

结构：

```text
                 Input
                   │
                   ↓
        Multi-Head Attention
                   │
                   ↓
             Add + Norm
                   │
                   ↓
           Feed Forward
                   │
                   ↓
             Add + Norm
                   │
                   ↓
                 Output
```

两个非常重要的组件：

```text
Residual Connection
Layer Normalization
```

---

# 20. Residual Connection

Residual Connection：

```text
Output = F(x) + x
```

而不是：

```text
Output = F(x)
```

结构：

```text
        ┌──────────────┐
        │              │
        │              ↓
x ─────→ F(x) ───────→ (+)
                       │
                       ↓
                    Output
```

为什么这样做？

因为深层网络容易：

```text
梯度消失
训练困难
```

Residual Connection 提供了一条：

> Gradient Highway

使梯度可以更稳定地传播。

ResNet、Transformer 都大量使用这种思想。

---

# 21. Layer Normalization

Transformer 大量使用：

> LayerNorm

它与 BatchNorm 有重要区别。

BatchNorm：

```text
Batch Dimension
```

LayerNorm：

```text
Feature Dimension
```

对于 NLP/Transformer 来说：

> LayerNorm 更适合序列数据。

基本形式：

```text
LN(x)
=
γ
(
x - μ
)
/
√(σ² + ε)
+
β
```

其中：

```text
μ
σ²
```

是当前样本特征维度上的统计量。

---

# 22. Feed Forward Network

Transformer Block 中另一个重要组件：

> Feed Forward Network

通常：

```text
FFN(x)
=
Activation(xW1 + b1)W2 + b2
```

例如：

```text
512
 ↓
2048
 ↓
512
```

结构：

```text
Input
  ↓
Linear
  ↓
Activation
  ↓
Linear
  ↓
Output
```

经典 Transformer 使用：

```text
ReLU
```

现代 LLM 常见：

```text
GELU
SwiGLU
GeGLU
```

---

# 23. FFN 到底在干什么？

Attention 主要负责：

> Token 与 Token 之间的信息交互。

FFN 更像：

> 对每一个 Token 独立进行非线性特征变换。

因此可以简单理解：

```text
Attention
=
Token之间通信

FFN
=
Token内部计算
```

这是理解 Transformer Block 非常好的抽象。

---

# 24. Attention + FFN

因此一个 Transformer Block 可以抽象成：

```text
Input
 │
 ↓
Attention
 │
 ↓
Token Communication
 │
 ↓
FFN
 │
 ↓
Feature Transformation
 │
 ↓
Output
```

从这个角度看：

> Transformer 本质上是一个不断进行“信息通信 + 特征计算”的深层网络。

---

# 25. Transformer Decoder

Decoder 与 Encoder 不完全相同。

Decoder 主要包含：

```text
Masked Self-Attention
        ↓
Cross Attention
        ↓
Feed Forward
```

结构：

```text
Input
 ↓
Masked Self-Attention
 ↓
Add & Norm
 ↓
Cross Attention
 ↓
Add & Norm
 ↓
FFN
 ↓
Add & Norm
```

---

# 26. 为什么 Decoder 需要 Mask？

考虑生成：

```text
I love artificial intelligence
```

当模型生成：

```text
artificial
```

时，它不能偷看：

```text
intelligence
```

否则训练任务就作弊了。

因此 Decoder 使用：

> Causal Mask / Look-Ahead Mask

形成：

```text
Token1 → Token1

Token2 → Token1 Token2

Token3 → Token1 Token2 Token3

Token4 → Token1 Token2 Token3 Token4
```

Attention Matrix：

```text
1 0 0 0
1 1 0 0
1 1 1 0
1 1 1 1
```

这就是：

> **Causal Attention**

---

# 27. GPT 为什么只需要 Decoder？

现代 GPT 类模型通常采用：

> Decoder-only Transformer

结构：

```text
Tokens
  ↓
Embedding
  ↓
Transformer Block
  ↓
Transformer Block
  ↓
...
  ↓
Transformer Block
  ↓
LM Head
  ↓
Next Token
```

它不需要传统 Transformer 中的 Encoder。

原因是 GPT 的核心任务是：

> **Autoregressive Language Modeling**

即：

```text
P(x_t | x_1, x_2, ..., x_(t-1))
```

预测下一个 Token。

---

# 28. GPT 的本质

假设：

```text
Input:

The weather today is
```

模型计算：

```text
P(
    next_token
    |
    The weather today is
)
```

可能得到：

```text
good      0.32
beautiful 0.18
sunny     0.12
...
```

然后选择：

```text
good
```

得到：

```text
The weather today is good
```

继续：

```text
P(
    next_token
    |
    The weather today is good
)
```

于是不断生成：

```text
Token1
 ↓
Token2
 ↓
Token3
 ↓
Token4
```

这就是 LLM 最基础的生成机制。

---

# 29. Transformer 与 LLM 的关系

需要明确：

> Transformer ≠ LLM

Transformer 是：

> Neural Network Architecture

LLM 是：

> Large Language Model

关系可以理解为：

```text
Transformer
    ↓
模型架构
    ↓
GPT / LLaMA / Qwen 等
    ↓
大规模参数 + 大规模数据 + 大规模训练
    ↓
LLM
```

因此：

```text
Transformer
```

是建筑结构。

而：

```text
LLM
```

是利用这种建筑结构建出来的大型系统。

---

# 30. Tokenization

LLM 并不是直接处理：

```text
"Hello world"
```

而是：

```text
Text
 ↓
Tokenizer
 ↓
Token IDs
```

例如：

```text
Hello world
```

可能变成：

```text
[15496, 995]
```

具体 Token ID 取决于 Tokenizer。

然后：

```text
Token ID
 ↓
Embedding
 ↓
Vector
```

最终 Transformer 接收的是：

```text
Vector Sequence
```

---

# 31. Embedding

假设：

```text
Vocabulary = 100000
Embedding Dimension = 4096
```

Embedding Matrix：

```text
E ∈ R^(100000 × 4096)
```

每一个 Token 都对应一个向量：

```text
Token ID
   ↓
Embedding Lookup
   ↓
[0.12, -0.37, ..., 0.82]
```

所以：

> Embedding 是把离散 Token 映射到连续向量空间。

---

# 32. Transformer 的完整数据流

现代 Decoder-only LLM 可以抽象为：

```text
                Text
                 │
                 ↓
              Tokenizer
                 │
                 ↓
             Token IDs
                 │
                 ↓
              Embedding
                 │
                 ↓
           Position Encoding
                 │
                 ↓
       ┌─────────────────────┐
       │ Transformer Block   │
       │                     │
       │ LayerNorm            │
       │ ↓                    │
       │ Self-Attention       │
       │ ↓                    │
       │ Residual             │
       │ ↓                    │
       │ LayerNorm            │
       │ ↓                    │
       │ FFN                  │
       │ ↓                    │
       │ Residual             │
       └─────────────────────┘
                 │
                ...
                 │
                 ↓
            Final LayerNorm
                 │
                 ↓
               LM Head
                 │
                 ↓
               Logits
                 │
                 ↓
              Softmax
                 │
                 ↓
           Next Token
```

这基本就是现代 GPT 类 LLM 的核心计算链路。

---

# 33. Logits

Transformer 最终不会直接输出：

```text
"hello"
```

而是输出：

```text
logits
```

例如词表：

```text
hello
world
AI
Java
Python
...
```

模型可能输出：

```text
hello → 2.3
world → 1.8
AI    → 4.2
Java  → 0.7
...
```

经过 Softmax：

```text
hello → 0.08
world → 0.05
AI    → 0.72
Java  → 0.01
...
```

然后根据：

```text
Greedy
Temperature
Top-K
Top-P
```

等策略选择下一个 Token。

---

# 34. Temperature

Temperature 用于控制随机性。

Softmax：

```text
P_i =
exp(z_i / T)
/
Σ exp(z_j / T)
```

当：

```text
T < 1
```

分布更加尖锐：

```text
更确定
```

当：

```text
T > 1
```

分布更加平滑：

```text
更随机
```

因此：

```text
T ↓ → deterministic
T ↑ → creative
```

---

# 35. Top-K

假设：

```text
Vocabulary = 100000
```

Top-K：

```text
K = 50
```

只保留概率最高的：

```text
50 tokens
```

其他：

```text
99950 tokens
```

直接忽略。

这样可以减少：

> 极低概率 Token 导致的异常生成。

---

# 36. Top-P

Top-P 又叫：

> Nucleus Sampling

例如：

```text
P = 0.9
```

按照概率从高到低累加：

```text
0.40
0.25
0.15
0.07
0.03
...
```

直到：

```text
累计概率 ≥ 0.9
```

只从这些 Token 中采样。

相比固定：

```text
Top-K
```

Top-P 更动态。

---

# 37. Transformer 的计算复杂度

Self-Attention 最大的问题：

```text
QKᵀ
```

如果：

```text
Sequence Length = n
```

Attention Matrix：

```text
n × n
```

因此时间/空间复杂度大致与：

```text
O(n²)
```

相关。

例如：

```text
n = 1,000
```

Attention Matrix：

```text
1,000 × 1,000
=
1,000,000
```

如果：

```text
n = 100,000
```

那么：

```text
100,000 × 100,000
=
10,000,000,000
```

这就是：

> Long Context Transformer 的核心挑战之一。

---

# 38. 为什么 LLM 需要 KV Cache？

生成式 LLM 是逐 Token 生成的。

例如：

```text
Token1
 ↓
Token2
 ↓
Token3
 ↓
Token4
```

如果每生成一个 Token 都重新计算之前所有 Token 的：

```text
K
V
```

会产生大量重复计算。

因此缓存：

```text
Key
Value
```

即：

> KV Cache

---

# 39. KV Cache 的工作原理

第一次：

```text
Token1
 ↓
K1 V1
```

第二次：

```text
Token1 Token2
 ↓
K1 V1
K2 V2
```

第三次：

```text
Token1 Token2 Token3
 ↓
K1 V1
K2 V2
K3 V3
```

之前计算过的：

```text
K1 V1
K2 V2
```

可以直接复用。

因此：

```text
KV Cache
=
避免重复计算
```

---

# 40. KV Cache 的工程代价

KV Cache 虽然提升了推理速度，但需要大量 GPU Memory。

大致可以理解：

```text
KV Memory
∝
Batch Size
×
Sequence Length
×
Number of Layers
×
KV Heads
×
Head Dimension
```

因此 LLM Serving 系统中非常重要的问题包括：

```text
GPU Memory
KV Cache Management
Batching
Paged Attention
Continuous Batching
```

这已经从：

> AI Model

进入：

> AI Systems Engineering

领域。

---

# 41. MQA 和 GQA

为了降低 KV Cache 的内存成本，现代模型引入：

### MQA

> Multi-Query Attention

多个 Query Head：

```text
Q1 Q2 Q3 Q4
```

共享：

```text
K V
```

---

### GQA

> Grouped-Query Attention

多个 Query Head 分组共享：

```text
K/V Heads
```

例如：

```text
Q1 Q2 → KV1
Q3 Q4 → KV2
Q5 Q6 → KV3
Q7 Q8 → KV4
```

相比传统 MHA：

```text
Q1 → K1 V1
Q2 → K2 V2
Q3 → K3 V3
...
```

GQA 可以明显降低：

```text
KV Cache
```

同时保持较好的模型能力。

---

# 42. FlashAttention

Transformer 的另一个重要优化：

> FlashAttention

它并不是简单地修改 Attention 数学公式，而是：

> **通过 IO-aware 的 GPU Kernel 优化 Attention 计算。**

传统 Attention：

```text
Q
 ↓
QKᵀ
 ↓
Softmax
 ↓
× V
```

中间可能产生巨大的：

```text
N × N
```

矩阵。

FlashAttention 通过更加高效的：

```text
Tiling
Memory Hierarchy
Kernel Fusion
```

减少 GPU HBM 与 SRAM 之间的数据搬运。

核心思想可以简单理解成：

> **不要只是优化 FLOPs，更要优化 GPU Memory Access。**

这是现代 LLM 系统优化的重要思想。

---

# 43. Pre-Norm vs Post-Norm

原始 Transformer：

```text
x
 ↓
Attention
 ↓
Add
 ↓
LayerNorm
```

属于：

> Post-Norm

现代 LLM 更常见：

```text
x
 ↓
LayerNorm
 ↓
Attention
 ↓
Add
```

属于：

> Pre-Norm

即：

```text
x + Attention(LN(x))
```

相比：

```text
LN(x + Attention(x))
```

Pre-Norm 通常具有更好的深层训练稳定性。

---

# 44. RMSNorm

现代 LLM 中还经常看到：

> RMSNorm

它比 LayerNorm 更简单。

LayerNorm：

```text
减均值
+
除标准差
```

RMSNorm：

```text
只考虑 RMS
```

形式大致：

```text
RMS(x)
=
sqrt(
mean(x²) + ε
)
```

然后：

```text
RMSNorm(x)
=
x / RMS(x) × weight
```

优点：

```text
计算简单
参数少
效率高
```

---

# 45. 激活函数：从 ReLU 到 SwiGLU

经典 Transformer：

```text
FFN
 ↓
ReLU
```

现代 LLM 经常使用：

```text
SwiGLU
```

例如：

```text
SwiGLU(x)
=
(SiLU(xW_g))
⊙
(xW_u)
```

然后：

```text
Output
=
SwiGLU(x)W_d
```

SwiGLU 的目标是：

> 在控制计算成本的同时增强模型表达能力。

因此现代 LLM 的 Transformer Block 与 2017 年原始论文中的 Transformer 已经存在不少差异。

---

# 46. MoE：Transformer 的进一步演化

当模型越来越大：

```text
7B
13B
70B
400B
...
```

一个问题出现：

> 如果所有参数每次都参与计算，成本非常高。

于是出现：

> Mixture of Experts（MoE）

基本思想：

```text
Input
  ↓
Router
  ↓
┌────┬────┬────┬────┐
Expert1 Expert2 Expert3 Expert4
└────┴────┴────┴────┘
       ↓
Selected Experts
       ↓
Output
```

例如：

```text
总参数：
100B

每个 Token 实际激活：
20B
```

于是：

```text
Total Parameters ≠ Active Parameters
```

这成为现代大模型架构的重要方向。

---

# 47. Transformer 的工程视角

对于传统软件工程师来说，可以把 Transformer 看成一个大型分布式计算系统的“计算核心”。

一个 Token 进入系统后：

```text
Token
 ↓
Embedding
 ↓
Position
 ↓
Attention
 ↓
Communication
 ↓
FFN
 ↓
Transformation
 ↓
Next Block
 ↓
...
 ↓
Logits
```

从软件架构角度：

```text
Embedding
    ↓
Data Representation

Attention
    ↓
Information Routing

FFN
    ↓
Feature Processing

Residual
    ↓
Stable Data Flow

LayerNorm
    ↓
Numerical Stabilization

KV Cache
    ↓
State / Cache Management

Batching
    ↓
Throughput Optimization

GPU Kernel
    ↓
Execution Optimization
```

因此：

> **现代 LLM 不只是机器学习问题，也是一个极其复杂的分布式系统和高性能计算问题。**

---

# 48. Transformer 与 Agent 的关系

如果进一步进入 Agent 领域，可以看到：

```text
User
 ↓
LLM
 ↓
Reasoning
 ↓
Tool Selection
 ↓
Tool
 ↓
Observation
 ↓
LLM
 ↓
Next Action
```

这里 Transformer 负责：

```text
Context Understanding
Reasoning
Generation
Tool Calling
Planning
```

而 Agent 系统负责：

```text
Memory
Tools
State
Workflow
Execution
Observation
```

因此可以理解：

> Transformer 是 Agent 的“大脑计算核心”，而 Agent 系统则是在 Transformer 外部构建了状态、工具、环境和执行能力。

---

# 49. Transformer 的完整知识地图

把整个 Transformer 技术体系串起来：

```text
                         Transformer
                              │
             ┌────────────────┼────────────────┐
             │                │                │
        Embedding          Attention          FFN
             │                │                │
             │         ┌──────┴──────┐         │
             │         │             │         │
             │       Self        Multi-Head    │
             │      Attention      Attention   │
             │         │             │         │
             │         └──────┬──────┘         │
             │                │                │
             └────────────────┼────────────────┘
                              │
                       Residual + Norm
                              │
                              ↓
                      Transformer Block
                              │
                              ↓
                    ┌─────────┴─────────┐
                    │                   │
                 Encoder             Decoder
                    │                   │
                  BERT                 GPT
                                        │
                                        ↓
                                      LLM
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                  RoPE                KV Cache             MoE
                    │                   │                   │
                    ↓                   ↓                   ↓
              Position Info         Serving            Scaling
```

---

# 50. Transformer 最重要的几个公式

如果准备 AI Engineer / LLM Engineer 面试，建议至少熟悉下面几个公式。

## Attention

```text
Attention(Q,K,V)
=
softmax(
QKᵀ / √d_k
)V
```

---

## Multi-Head Attention

```text
MultiHead(Q,K,V)
=
Concat(head1,...,headh)W_O
```

---

## FFN

```text
FFN(x)
=
Activation(xW1+b1)W2+b2
```

---

## Autoregressive Language Modeling

```text
P(x1,...,xn)
=
∏ P(x_t | x_<t)
```

---

## Cross Entropy

语言模型训练通常最小化：

```text
L
=
-Σ y_t log(p_t)
```

本质上：

> 让正确的下一个 Token 获得更高概率。

---

# 51. Transformer 为什么如此成功？

可以归纳成五个原因。

## 第一：Attention

解决长距离依赖。

---

## 第二：高度并行

适合：

```text
GPU
TPU
Distributed Training
```

---

## 第三：可扩展

可以通过增加：

```text
Layers
Hidden Size
Attention Heads
Training Data
Parameters
```

持续扩大模型。

---

## 第四：统一架构

文本：

```text
Token
```

图片：

```text
Patch
```

音频：

```text
Audio Token
```

代码：

```text
Code Token
```

本质上都可以转化成：

```text
Embedding Sequence
```

然后交给 Transformer。

---

## 第五：Scaling Law

Transformer 与大规模：

```text
Data
Compute
Parameters
```

结合后表现出非常强的 Scaling 能力。

这为今天的：

```text
Foundation Model
LLM
Multimodal Model
Agent
```

奠定了基础。

---

# 52. 从 RNN 到 Transformer 的真正技术演进

可以把整个过程理解成：

```text
RNN
 │
 │ 解决序列建模
 ↓
LSTM
 │
 │ 解决长期记忆
 ↓
Attention
 │
 │ 解决信息瓶颈
 ↓
Transformer
 │
 │ 解决并行计算与大规模训练
 ↓
GPT / BERT / T5
 │
 │ 模型规模扩大
 ↓
LLM
 │
 │ Tool / Memory / Multimodal
 ↓
Agent
```

因此学习 Transformer 时，不应该只记：

```text
Q
K
V
```

真正应该理解的是：

> **Transformer 为什么出现？**

答案是：

```text
RNN 的顺序计算
        +
长期依赖问题
        +
信息压缩瓶颈
        ↓
Attention
        ↓
Transformer
```

---

# 53. AI 工程师学习 Transformer 的正确层次

如果目标是从传统后端工程进入：

```text
AI Engineer
LLM Engineer
Agent Engineer
```

建议分四层掌握。

## Level 1：原理

必须理解：

```text
Embedding
Attention
Q/K/V
Softmax
Multi-Head Attention
Residual
LayerNorm
FFN
Position Encoding
```

---

## Level 2：模型

理解：

```text
Encoder
Decoder
BERT
GPT
T5
RoPE
RMSNorm
SwiGLU
GQA
MoE
```

---

## Level 3：推理

重点学习：

```text
KV Cache
Batching
Continuous Batching
FlashAttention
Quantization
Tensor Parallel
Pipeline Parallel
Speculative Decoding
```

---

## Level 4：AI Systems

进一步进入：

```text
LLM Serving
Model Gateway
Prompt Management
RAG
Vector Database
Tool Calling
Agent
Memory
Observability
Evaluation
```

最终形成：

```text
                    AI Application
                          │
                     ┌────┴────┐
                     ↓         ↓
                    RAG      Agent
                     │         │
                     └────┬────┘
                          ↓
                         LLM
                          ↓
                    Transformer
                          ↓
                 GPU / Distributed System
```

---

# 54. 最终总结

Transformer 最核心的思想可以浓缩成一句话：

> **让序列中的每一个 Token 能够动态地选择自己应该关注的信息。**

它通过：

```text
Embedding
   ↓
Position
   ↓
Q/K/V
   ↓
Self-Attention
   ↓
Multi-Head Attention
   ↓
Residual + Norm
   ↓
FFN
   ↓
Transformer Blocks
   ↓
Logits
   ↓
Next Token
```

构成现代 LLM 的基本计算框架。

如果 RNN 的核心是：

```text
State
```

那么 Transformer 的核心就是：

```text
Attention
```

如果进一步看现代 LLM：

```text
Attention
+
FFN
+
Residual
+
Normalization
+
Position Encoding
+
Scaling
+
KV Cache
+
GPU Optimization
+
Distributed Training
```

共同构成了今天的大模型技术栈。

而对于 AI Engineer 来说，最重要的认知不是：

> “Transformer 是一个很复杂的神经网络。”

而应该是：

> **Transformer 是一种高度可并行、可扩展的序列信息处理架构；Attention 负责信息路由，FFN 负责特征变换，Residual/Normalization 保证深层训练稳定，而 KV Cache、FlashAttention、GQA、MoE 等技术则把它从一个论文模型逐渐演化成了今天能够支撑大规模 LLM 的工业级计算系统。**

从 RNN 到 Transformer，再到 LLM 和 Agent，这实际上是一条非常完整的 AI 技术演进主线。

你下一步更适合深入 **① Attention/Transformer 手写源码**、**② GPT/LLM 内部架构**，还是 **③ KV Cache + FlashAttention + LLM 推理系统**？
