---
title: asyncio 的核心知识
# tags:
#   - nodejs
date: '2023-11-04'
summary: asyncio = Python 的异步 I/O 框架，核心解决“一个线程如何高效地同时处理大量 I/O 等待”。
---


> **asyncio = Python 的异步 I/O 框架，核心解决“一个线程如何高效地同时处理大量 I/O 等待”。**

---

# 一、asyncio 到底解决什么问题？

假设 FastAPI 同时收到 3 个请求：

```text
Request A → 调用数据库 → 等待 100ms
Request B → 调用 Redis   → 等待 50ms
Request C → 调用 HTTP API → 等待 200ms
```

传统同步代码：

```text
A ────────等待────────>
                     B ─────等待────>
                                  C ───────────等待────────>
```

总耗时可能接近：

```text
100 + 50 + 200 = 350ms
```

异步：

```text
A ──等待──┐
B ─等待───┼──→ Event Loop
C ───等待─┘
```

当 A 在等待数据库时，程序可以去处理 B。

所以：

```text
asyncio
   ↓
Event Loop
   ↓
Task
   ↓
await
   ↓
非阻塞 I/O
```

这是理解 asyncio 最重要的一条主线。

---

# 二、asyncio 的核心知识地图

如果按照重要程度，我建议你这样划分：

```text
★★★★★ 必须掌握

async / await
Coroutine
Event Loop
Task
Future
asyncio.gather()
asyncio.create_task()
同步 vs 异步
阻塞 vs 非阻塞
异步 I/O

★★★★ 强烈建议掌握

asyncio.Queue
asyncio.Lock
asyncio.Semaphore
asyncio.Event
asyncio.wait_for()
asyncio.timeout()
asyncio.wait()
asyncio.as_completed()

★★★ 工作中需要

asyncio.to_thread()
run_in_executor()
TaskGroup
异步上下文管理器
异步迭代器

★★ 了解即可

Protocol
Transport
低级 Event Loop API
自定义 Event Loop
```

---

# 三、第一必须掌握：async / await

这是 asyncio 的基础。

```python
import asyncio

async def hello():
    print("Hello")

asyncio.run(hello())
```

这里：

```python
async def
```

定义的是 **Coroutine Function（协程函数）**。

而：

```python
hello()
```

返回的是：

```text
Coroutine Object
```

---

# 四、await 是最重要的关键字

例如：

```python
async def task():
    print("Start")

    await asyncio.sleep(2)

    print("End")
```

注意：

```python
await asyncio.sleep(2)
```

不是：

> 当前线程睡眠 2 秒。

而是：

> **当前协程暂时挂起，把执行机会让给 Event Loop。**

所以：

```text
task A
  ↓
await
  ↓
Event Loop
  ↓
task B
  ↓
await
  ↓
task C
```

这就是异步并发。

---

# 五、第二必须掌握：Coroutine

这是 Java 开发者最容易混淆的地方。

```python
async def foo():
    return 100
```

执行：

```python
result = foo()
```

此时：

```text
result ≠ 100
```

而是：

```text
Coroutine Object
```

必须：

```python
result = await foo()
```

才能得到：

```text
100
```

所以：

```text
async def
   ↓
Coroutine Function

foo()
   ↓
Coroutine Object

await foo()
   ↓
执行 Coroutine
   ↓
得到结果
```

这三个概念一定要分清。

---

# 六、第三必须掌握：Event Loop

这是 asyncio 的核心。

可以把 Event Loop 理解成：

> **负责调度所有异步任务的“调度器”。**

例如：

```text
                Event Loop
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
      Task A      Task B      Task C
        │           │           │
      await       await       await
        │           │           │
        └───────────┼───────────┘
                    ↓
               I/O Ready
                    ↓
               继续执行
```

简单例子：

```python
import asyncio

async def main():
    print("Hello")
    
asyncio.run(main())
```

这里：

```python
asyncio.run(main())
```

实际上就是：

```text
创建 Event Loop
       ↓
运行 main()
       ↓
处理异步任务
       ↓
关闭 Event Loop
```

---

# 七、第四必须掌握：Task

这是非常重要的。

Coroutine：

```python
coro = foo()
```

并不代表它已经被并发调度。

如果：

```python
task = asyncio.create_task(foo())
```

那么：

> **把 Coroutine 包装成 Task，并交给 Event Loop 调度。**

例如：

```python
async def task_a():
    await asyncio.sleep(2)
    return "A"

async def task_b():
    await asyncio.sleep(1)
    return "B"
```

同步执行：

```python
await task_a()
await task_b()
```

大约：

```text
2s + 1s = 3s
```

并发执行：

```python
a = asyncio.create_task(task_a())
b = asyncio.create_task(task_b())

result_a = await a
result_b = await b
```

大约：

```text
max(2s, 1s) = 2s
```

这就是 asyncio 的价值。

---

# 八、第五必须掌握：asyncio.gather()

实际项目里非常常用。

```python
results = await asyncio.gather(
    task_a(),
    task_b(),
    task_c()
)
```

例如：

```python
async def get_user():
    ...

async def get_orders():
    ...

async def get_products():
    ...

user, orders, products = await asyncio.gather(
    get_user(),
    get_orders(),
    get_products()
)
```

这相当于：

```text
             FastAPI
                │
       ┌────────┼────────┐
       ↓        ↓        ↓
     User     Orders   Products
       │        │        │
       └────────┼────────┘
                ↓
             gather
                ↓
             Response
```

对于 FastAPI：

> **这是非常值得掌握的 API。**

---

# 九、Task 和 gather 有什么区别？

这是面试经常问的问题。

### create_task

```python
task = asyncio.create_task(foo())
```

重点：

> 创建一个可以独立调度的 Task。

### gather

```python
await asyncio.gather(
    foo(),
    bar()
)
```

重点：

> **并发执行多个 awaitable，并收集结果。**

可以理解：

```text
create_task()
     ↓
创建任务

gather()
     ↓
批量并发 + 收集结果
```

---

# 十、Future 是什么？

这是必须理解，但不需要大量使用。

可以简单理解：

> **Future 是一个代表“未来某个时间会产生结果”的低层对象。**

关系：

```text
Coroutine
    ↓
Task
    ↓
Future
    ↓
最终结果
```

但在现代 Python 开发中：

> **业务代码很少直接操作 Future。**

你需要知道它是什么，但不需要每天写：

```python
asyncio.Future()
```

---

# 十一、阻塞和非阻塞必须彻底搞懂

这个对 FastAPI 特别重要。

例如：

```python
import time

async def foo():
    time.sleep(5)
```

这是一个**错误示范**。

因为：

```python
time.sleep(5)
```

是阻塞操作。

它会阻塞 Event Loop。

---

正确：

```python
async def foo():
    await asyncio.sleep(5)
```

这样：

```text
Event Loop
   │
   ├── Task A
   │      ↓
   │    await sleep
   │
   ├── Task B ← 继续执行
   │
   └── Task C ← 继续执行
```

所以你一定要理解：

```text
async ≠ 自动异步

await ≠ 所有操作都非阻塞
```

真正关键的是：

> **await 的操作本身必须是异步/非阻塞的，或者正确地把阻塞工作移出 Event Loop。**

---

# 十二、asyncio.sleep()

这是学习 asyncio 最简单的例子。

```python
await asyncio.sleep(1)
```

它主要用于：

* 模拟 I/O
* 定时任务
* 重试
* 限速
* 测试异步代码

例如：

```python
async def retry():
    for i in range(3):
        try:
            return await call_api()
        except Exception:
            await asyncio.sleep(1)
```

---

# 十三、asyncio.Queue

这个在微服务和后台任务中很重要。

```python
queue = asyncio.Queue()
```

生产者：

```python
await queue.put(data)
```

消费者：

```python
data = await queue.get()
```

形成：

```text
Producer
   ↓
┌──────────────┐
│ asyncio.Queue│
└──────────────┘
   ↓
Consumer
```

可以实现：

```text
生产者 / 消费者
异步任务处理
任务缓冲
并发控制
```

---

# 十四、asyncio.Lock

如果多个 Coroutine 修改共享数据：

```python
lock = asyncio.Lock()
```

使用：

```python
async with lock:
    balance -= 100
```

类似 Java：

```java
synchronized
```

或者：

```java
Lock
```

但是注意：

> `asyncio.Lock` 是给**同一个 Event Loop 中的协程并发控制**使用的，不是 Java `ReentrantLock` 的直接替代品。

---

# 十五、Semaphore

这个非常实用。

假设你最多允许：

```text
同时 10 个 HTTP 请求
```

可以：

```python
semaphore = asyncio.Semaphore(10)
```

然后：

```python
async with semaphore:
    await call_api()
```

形成：

```text
1000 requests
      ↓
 Semaphore(10)
      ↓
最多10个同时执行
```

这和你之前学习的 **Rate Limiter / 并发控制**非常相关。

---

# 十六、asyncio.Event

用于协程之间通知。

```python
event = asyncio.Event()
```

等待：

```python
await event.wait()
```

触发：

```python
event.set()
```

例如：

```text
Worker
  ↓
等待系统初始化
  ↓
Event.wait()
  ↓
初始化完成
  ↓
Event.set()
  ↓
Worker继续执行
```

---

# 十七、超时控制

实际生产环境非常重要。

例如：

```python
await asyncio.wait_for(
    call_api(),
    timeout=3
)
```

意思：

> 3 秒还没完成，就超时。

现代 Python 也可以使用：

```python
async with asyncio.timeout(3):
    await call_api()
```

这对于：

```text
HTTP API
Database
Redis
LLM
Microservice
```

都非常重要。

---

# 十八、asyncio.to_thread()

这个对你特别重要。

假设你有一个同步阻塞函数：

```python
def blocking_function():
    time.sleep(5)
```

如果直接：

```python
await blocking_function()
```

是不行的。

可以：

```python
result = await asyncio.to_thread(
    blocking_function
)
```

变成：

```text
Event Loop
     │
     ├───────────────┐
     ↓               ↓
Async Task       Thread
                    ↓
              Blocking Task
```

这样不会直接堵住 Event Loop。

---

# 十九、asyncio.TaskGroup

Python 新版本中非常值得学习。

例如：

```python
async with asyncio.TaskGroup() as tg:

    tg.create_task(task_a())

    tg.create_task(task_b())

    tg.create_task(task_c())
```

它提供结构化并发。

可以理解成：

```text
TaskGroup
   │
   ├── Task A
   ├── Task B
   └── Task C
```

相比过去大量使用：

```python
asyncio.create_task()
```

TaskGroup 对任务生命周期和异常处理更加规范。

---

# 二十、异步上下文管理器

你以后使用数据库、HTTP Client 时经常遇到：

```python
async with ...
```

例如：

```python
async with http_client as client:
    response = await client.get(url)
```

对应同步：

```python
with ...
```

异步：

```python
async with ...
```

这个在 FastAPI 项目中应该掌握。

---

# 二十一、异步 HTTP

真正开发 FastAPI 时，你会经常使用：

```text
httpx
aiohttp
```

例如 HTTPX：

```python
import httpx

async with httpx.AsyncClient() as client:

    response = await client.get(
        "https://example.com"
    )
```

这才是实际的：

```text
FastAPI
   ↓
await
   ↓
Async HTTP Client
   ↓
External Service
```

---

# 二十二、asyncio 和多线程、多进程不要混淆

这是面试重点。

### asyncio

主要解决：

```text
I/O 并发
```

例如：

```text
HTTP
DB
Redis
Kafka
文件
网络
```

### Thread

适合：

```text
阻塞 I/O
同步第三方库
```

### Process

适合：

```text
CPU 密集型任务
```

例如：

```text
图像处理
大规模计算
机器学习计算
复杂算法
```

可以简单记：

```text
             Python 并发
                 │
       ┌─────────┼─────────┐
       ↓         ↓         ↓
    asyncio    Thread    Process
       │         │         │
      I/O      I/O      CPU
```

---

# 二十三、你必须掌握的知识分级

如果你的目标是：

> **FastAPI + AI Full-Stack + 微服务开发**

我建议：

## 第一优先级 ⭐⭐⭐⭐⭐

一定掌握：

```text
async / await
Coroutine
Event Loop
Task
create_task()
gather()
asyncio.run()
sleep()
阻塞 vs 非阻塞
同步 vs 异步
```

尤其是这张图：

```text
async def
    ↓
Coroutine
    ↓
create_task()
    ↓
Task
    ↓
Event Loop
    ↓
await
    ↓
I/O
    ↓
Task Resume
```

---

## 第二优先级 ⭐⭐⭐⭐

工作中非常有用：

```text
asyncio.Queue
asyncio.Lock
asyncio.Semaphore
asyncio.Event
asyncio.wait_for()
asyncio.timeout()
asyncio.as_completed()
asyncio.wait()
```

---

## 第三优先级 ⭐⭐⭐

建议掌握：

```text
asyncio.to_thread()
TaskGroup
Async Context Manager
Async Iterator
Async Generator
run_in_executor()
```

---

## 第四优先级 ⭐⭐

知道概念即可：

```text
Future
Transport
Protocol
低级 Event Loop API
自定义 Event Loop
```

---

# 二十四、结合 FastAPI，你真正应该掌握什么？

我建议你不要单独把 asyncio 学成一门“理论课”。

直接结合 FastAPI 学：

```text
                    FastAPI
                       │
                       ↓
                  async def
                       │
                       ↓
                     await
                       │
             ┌─────────┼─────────┐
             ↓         ↓         ↓
           HTTP       Redis      DB
             │         │         │
           httpx     redis      asyncpg
             │         │         │
             └─────────┼─────────┘
                       ↓
                   Event Loop
```

然后重点练习：

### 实战 1：并发调用 3 个 API

```python
await asyncio.gather(
    call_api_1(),
    call_api_2(),
    call_api_3()
)
```

### 实战 2：限制并发

```python
semaphore = asyncio.Semaphore(10)
```

### 实战 3：超时

```python
asyncio.timeout(3)
```

### 实战 4：任务队列

```python
asyncio.Queue()
```

### 实战 5：阻塞代码

```python
await asyncio.to_thread(...)
```

### 实战 6：FastAPI + Async HTTP

```python
httpx.AsyncClient
```

### 实战 7：FastAPI + Async PostgreSQL

```text
FastAPI
   ↓
SQLAlchemy Async
   ↓
asyncpg
   ↓
PostgreSQL
```

---

# 二十五、最后给你一个非常重要的认知

如果你以前是 Java/Spring Boot 开发者，可以这样建立映射：

```text
Java                         Python

Thread Pool              →   asyncio Event Loop
Future                   →   asyncio Future
CompletableFuture        →   Coroutine / Task / gather
ExecutorService          →   Task scheduling
synchronized             →   asyncio.Lock
Semaphore                →   asyncio.Semaphore
Blocking I/O             →   不应该阻塞 Event Loop
WebFlux                  →   async FastAPI
```

不过不要把它们简单认为是“一一对应”，尤其是 **CompletableFuture ≠ Coroutine**。

---

