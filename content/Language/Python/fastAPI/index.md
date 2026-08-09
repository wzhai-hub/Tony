---
title: fastAPI基本功能
# tags:
#   - nodejs
date: '2023-11-04'
summary: Python 世界里的一个现代、高性能 Web API 开发框架，
---

> **Python 世界里的一个现代、高性能 Web API 开发框架，定位有点类似 Java 的 Spring Boot，但更加轻量。**

如果你熟悉 **Java + Spring Boot**，学习 FastAPI 会非常容易。它特别适合开发 **REST API、微服务、AI 应用后端、RAG/LLM 服务**。

---

# 一、FastAPI 主要有哪些功能？

FastAPI 的核心功能可以分成下面几个方面：

| 功能                   | FastAPI 能做什么        | 类似 Java/Spring                 |
| -------------------- | ------------------- | ------------------------------ |
| HTTP API             | GET/POST/PUT/DELETE | `@GetMapping` / `@PostMapping` |
| 路由                   | URL → Python 函数     | `@RequestMapping`              |
| 参数校验                 | 自动校验请求参数            | `@Valid`                       |
| 数据模型                 | Pydantic Model      | DTO                            |
| JSON 序列化             | Python 对象 ↔ JSON    | Jackson                        |
| Swagger              | 自动生成 API 文档         | SpringDoc                      |
| OpenAPI              | 自动生成 OpenAPI Schema | OpenAPI                        |
| Dependency Injection | 依赖注入                | `@Autowired`                   |
| Middleware           | 请求/响应拦截             | Filter                         |
| Authentication       | JWT/OAuth2/API Key  | Spring Security                |
| Async                | 异步 API              | WebFlux                        |
| WebSocket            | 实时通信                | Spring WebSocket               |
| Background Task      | 后台任务                | `@Async` 等                     |
| 文件上传                 | UploadFile          | MultipartFile                  |
| CORS                 | 跨域处理                | CorsFilter                     |
| Streaming            | 流式返回                | StreamingResponse              |
| 微服务                  | REST 微服务            | Spring Boot Microservice       |

---

# 二、最核心的功能：开发 REST API

最简单的 FastAPI：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/hello")
def hello():
    return {"message": "Hello World"}
```

启动：

```bash
uvicorn main:app --reload
```

访问：

```text
http://localhost:8000/hello
```

返回：

```json
{
  "message": "Hello World"
}
```

如果用 Spring Boot 思维理解：

```java
@RestController
public class HelloController {

    @GetMapping("/hello")
    public Map<String, String> hello() {
        return Map.of("message", "Hello World");
    }
}
```

所以：

```text
FastAPI @app.get()
       ↓
Spring Boot @GetMapping
```

---

# 三、HTTP 方法

FastAPI 支持标准 HTTP 方法：

```python
@app.get("/users")
def get_users():
    pass

@app.post("/users")
def create_user():
    pass

@app.put("/users/{user_id}")
def update_user(user_id: int):
    pass

@app.delete("/users/{user_id}")
def delete_user(user_id: int):
    pass
```

对应：

```text
GET       查询
POST      创建
PUT       更新
DELETE    删除
PATCH     部分更新
```

这部分和 Spring MVC 非常类似。

---

# 四、强大的参数校验

这是 FastAPI 非常重要的一个功能。

例如：

```python
@app.get("/users/{user_id}")
def get_user(user_id: int):
    return {"user_id": user_id}
```

这里：

```python
user_id: int
```

意味着 FastAPI 会自动进行类型校验。

请求：

```text
/users/100
```

正常。

请求：

```text
/users/abc
```

FastAPI 会自动返回类似：

```json
{
  "detail": [
    {
      "type": "int_parsing",
      "loc": ["path", "user_id"],
      "msg": "Input should be a valid integer"
    }
  ]
}
```

你不需要自己写：

```python
if not isinstance(user_id, int):
    ...
```

---

# 五、Pydantic：FastAPI 的核心

如果你要认真学习 FastAPI，**Pydantic 是必须掌握的**。

例如创建用户：

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int
    email: str
```

然后：

```python
@app.post("/users")
def create_user(user: User):
    return user
```

客户端：

```json
{
    "name": "Vincent",
    "age": 30,
    "email": "vincent@example.com"
}
```

FastAPI 会自动：

```text
JSON
 ↓
Pydantic
 ↓
User 对象
 ↓
业务代码
```

这和 Java：

```java
public class UserDTO {

    private String name;

    private Integer age;

    private String email;
}
```

非常类似。

---

# 六、自动生成 Swagger API 文档

这是 FastAPI 最好用的功能之一。

启动：

```bash
uvicorn main:app --reload
```

访问：

```text
http://localhost:8000/docs
```

就可以看到 Swagger UI。

它会自动显示：

```text
GET     /users
POST    /users
PUT     /users/{id}
DELETE  /users/{id}
```

而且可以直接点击：

```text
Try it out
```

测试 API。

另外：

```text
http://localhost:8000/redoc
```

可以看到 ReDoc 文档。

所以你不需要手动维护 API 文档。

---

# 七、OpenAPI

FastAPI 天生支持 OpenAPI。

访问：

```text
http://localhost:8000/openapi.json
```

可以得到 API Schema。

例如：

```json
{
  "openapi": "3.1.0",
  "paths": {
    "/users": {
      "get": {
        ...
      }
    }
  }
}
```

这对于：

* API Gateway
* API 文档
* 自动生成客户端
* 前后端协作
* 自动化测试

都非常有用。

---

# 八、Dependency Injection 依赖注入

FastAPI 也有自己的 DI。

例如：

```python
from fastapi import Depends

def get_db():
    db = create_database_connection()

    try:
        yield db
    finally:
        db.close()
```

Controller：

```python
@app.get("/users")
def get_users(db = Depends(get_db)):
    return db.query_users()
```

这里：

```python
Depends(get_db)
```

就是依赖注入。

如果你来自 Spring：

```java
@Autowired
private UserService userService;
```

FastAPI 的：

```python
Depends(...)
```

可以理解成一种轻量级 DI 机制。

---

# 九、Middleware

FastAPI 支持 Middleware。

例如记录请求时间：

```python
@app.middleware("http")
async def log_request(request, call_next):

    start = time.time()

    response = await call_next(request)

    duration = time.time() - start

    print(f"Request took {duration}s")

    return response
```

典型用途：

```text
Request
   ↓
Middleware
   ↓
Authentication
   ↓
Controller
   ↓
Service
   ↓
Response
   ↓
Middleware
   ↓
Client
```

可以用于：

* Logging
* Trace ID
* Authentication
* CORS
* Metrics
* Request timing
* Exception handling

---

# 十、Authentication / Authorization

FastAPI 对安全认证支持也很好。

例如：

```text
Authorization: Bearer eyJhbGciOi...
```

可以使用：

```text
OAuth2
JWT
API Key
HTTP Basic
```

典型架构：

```text
Client
   ↓
JWT
   ↓
FastAPI
   ↓
Authentication
   ↓
Authorization
   ↓
Business Service
```

如果你熟悉 Spring Security，可以把这一块理解成：

```text
FastAPI Security
       ≈
Spring Security
```

当然，两者实现方式差别比较大。

---

# 十一、异步编程

这是 FastAPI 非常重要的特点。

可以直接写：

```python
@app.get("/users")
async def get_users():
    users = await query_users()
    return users
```

Python：

```text
async
await
```

配合：

```text
asyncio
```

可以实现高并发 I/O 服务。

例如：

```text
HTTP Request
      ↓
FastAPI
      ↓
async
      ↓
Database
      ↓
Redis
      ↓
External API
```

特别适合：

* AI API
* LLM
* RAG
* 外部 API 调用
* 高并发 I/O
* 微服务

---

# 十二、WebSocket

FastAPI 不只是 REST API。

还支持 WebSocket。

例如：

```python
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):

    await websocket.accept()

    while True:
        data = await websocket.receive_text()

        await websocket.send_text(
            f"Message: {data}"
        )
```

可以用于：

```text
聊天系统
实时通知
在线状态
实时监控
AI Streaming
```

---

# 十三、文件上传

例如：

```python
from fastapi import UploadFile, File

@app.post("/upload")
async def upload(file: UploadFile = File(...)):

    content = await file.read()

    return {
        "filename": file.filename
    }
```

可以处理：

```text
图片
PDF
Excel
CSV
Word
音频
视频
```

这对于 AI 应用特别重要。

例如：

```text
用户上传 PDF
      ↓
FastAPI
      ↓
PDF Parser
      ↓
Text
      ↓
Embedding
      ↓
Vector DB
```

这就是典型的 RAG 后端。

---

# 十四、Background Tasks

FastAPI 可以执行后台任务。

例如：

```python
from fastapi import BackgroundTasks

@app.post("/send-email")
def send_email(
    background_tasks: BackgroundTasks
):

    background_tasks.add_task(
        send_email_task
    )

    return {
        "message": "Email will be sent"
    }
```

请求不需要等待：

```text
Client
  ↓
FastAPI
  ↓
立即返回
  ↓
Background Task
  ↓
Send Email
```

不过要注意：

> FastAPI BackgroundTasks 适合轻量后台任务，不等同于 Kafka/Celery 这种真正的分布式任务队列。

---

# 十五、CORS

如果你的前端是 React：

```text
React
localhost:3000
       ↓
FastAPI
localhost:8000
```

会遇到跨域。

FastAPI 可以：

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

这和你之前做 React + Spring Boot 时遇到的 CORS 问题非常类似。

---

# 十六、Streaming Response

这个功能对你尤其值得学习。

比如 AI：

```text
用户
 ↓
React
 ↓
FastAPI
 ↓
LLM
 ↓
Token 1
Token 2
Token 3
Token 4
...
```

而不是：

```text
用户
 ↓
FastAPI
 ↓
等待 10 秒
 ↓
一次性返回完整答案
```

可以实现：

```text
Hello
Hello, how
Hello, how are
Hello, how are you
```

这种流式输出。

所以现在很多：

```text
ChatGPT 类应用
AI Agent
RAG
LLM Gateway
```

都会使用 FastAPI。

---

# 十七、数据库

FastAPI 本身**不是 ORM**。

它通常和其他数据库框架组合。

例如：

```text
FastAPI
   │
   ├── SQLAlchemy
   │
   ├── SQLModel
   │
   ├── Tortoise ORM
   │
   └── asyncpg
          │
          ↓
       PostgreSQL
```

例如：

```python
@app.get("/users")
async def users():
    result = await db.execute(...)
    return result
```

你可以使用：

```text
PostgreSQL
MySQL
Oracle
MongoDB
Redis
Elasticsearch
```

等。

---

# 十八、FastAPI 的整体架构

如果按照你熟悉的 Spring Boot 思维，可以这样理解：

```text
                    ┌───────────────┐
                    │    React      │
                    └───────┬───────┘
                            │
                           HTTP
                            │
                    ┌───────▼───────┐
                    │    FastAPI    │
                    │   Controller  │
                    └───────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
          Pydantic       Depends      Middleware
              │             │             │
              └─────────────┼─────────────┘
                            │
                    ┌───────▼───────┐
                    │    Service    │
                    └───────┬───────┘
                            │
             ┌──────────────┼──────────────┐
             │              │              │
           Redis        PostgreSQL       Kafka
             │              │              │
             └──────────────┼──────────────┘
                            │
                    ┌───────▼───────┐
                    │ External APIs  │
                    │   LLM / AI     │
                    └───────────────┘
```

---