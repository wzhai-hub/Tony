---
title: Agent Infinite Loop
# tags:
#   - nodejs
date: '2026-08-05'
summary: Agent 防止无限循环通常采用多层 Guard 机制，包括最大执行步数、Token 限制、Timeout、Tool 调用次数限制、Retry 上限、重复状态检测以及 Agent 调用链的 Cycle Detection。对于 Multi-Agent，还需要通过 Supervisor、State Machine 和 Agent 权限边界控制 Agent 间的循环调用。生产环境一般不会让 LLM 完全自由地控制 Workflow，而是让 LLM 负责决策，把执行流程交给确定性的 Workflow/State Machine 控制。
---

Agent 防止**无限循环（Infinite Loop）**是生产级 Agent 系统非常重要的问题。尤其是 Multi-Agent 中，Agent 之间可能互相调用，更容易出现循环。

可以把它理解成：

> **不能让 Agent 拥有“无限执行”的权力，必须给它设置硬边界。**

---

## 1. 最基本的方法：最大执行次数

最简单也最有效：

```text
Agent
 ↓
调用 Tool
 ↓
观察结果
 ↓
继续思考
 ↓
调用 Tool
 ↓
...
 ↓
达到 Max Steps
 ↓
强制停止
```

例如：

```java
int maxSteps = 10;

for (int step = 0; step < maxSteps; step++) {

    AgentResult result = agent.run(context);

    if (result.isCompleted()) {
        return result;
    }

    context = result.getNextContext();
}

return AgentResult.failed("Maximum steps exceeded");
```

比如：

```text
max_steps = 10
```

意味着 Agent 最多执行 10 个 reasoning/action cycle。

这是**第一道保险**。

---

# 2. Token Limit

第二层是限制 Token。

例如：

```text
Agent
 │
 ├── Prompt
 ├── History
 ├── Tool Result
 ├── Reasoning
 │
 ↓
Token Counter
 │
 └── > 100K ?
        ↓
      STOP
```

例如：

```text
max_input_tokens  = 50,000
max_output_tokens = 10,000
```

如果 Agent 不断产生内容，Token 最终达到限制，就停止。

不过：

> **Token limit 不能代替 Max Steps。**

因为 Agent 可能每次只产生很少 Token，但执行几千次。

---

# 3. Timeout

第三层：

```text
Agent
  ↓
Timer
  ↓
30 seconds
  ↓
STOP
```

例如：

```java
CompletableFuture
    .supplyAsync(() -> agent.run(context))
    .orTimeout(30, TimeUnit.SECONDS);
```

生产系统通常同时设置：

```text
Max Steps
+
Max Token
+
Timeout
```

形成三重保护。

---

# 4. Tool 调用次数限制

这个非常重要。

假设 Agent 有：

```text
search()
database()
http()
code_execution()
```

不能让它无限调用。

例如：

```text
search:        max 5
database:      max 10
http:          max 10
code_execution max 3
```

可以设计：

```java
class ToolUsage {

    Map<String, Integer> counters;

    boolean allowed(String tool) {
        return counters.getOrDefault(tool, 0) < LIMIT;
    }
}
```

这样：

```text
Agent
 ↓
search()
 ↓
search()
 ↓
search()
 ↓
search()
 ↓
search()
 ↓
第 6 次
 ↓
REJECT
```

---

# 5. 检测重复调用

这是 Agent 防循环非常关键的一招。

例如 Agent 出现：

```text
search("Java Redis")
 ↓
search("Java Redis")
 ↓
search("Java Redis")
 ↓
search("Java Redis")
```

显然可能陷入循环。

可以建立一个：

```text
Tool Call Fingerprint
```

例如：

```text
hash(
    toolName
    +
    arguments
)
```

得到：

```text
ABC123
```

如果：

```text
ABC123
ABC123
ABC123
```

连续重复，就停止。

---

# 6. 检测 State 是否没有变化

比简单的重复 Tool Call 更高级。

例如：

```text
State 1
question = "xxx"
answer   = null

        ↓

State 2
question = "xxx"
answer   = null

        ↓

State 3
question = "xxx"
answer   = null
```

虽然 Tool 参数可能不同，但**系统状态实际上没有变化**。

这时候可以判断：

```text
State(N) == State(N-1)
```

或者：

```text
similarity(StateN, StateN-1) > threshold
```

然后：

```text
STOP
```

---

# 7. Multi-Agent 最危险的是 Agent A ↔ Agent B

例如：

```text
Agent A
   ↓
"请 Agent B 分析"
   ↓
Agent B
   ↓
"请 Agent A 再确认"
   ↓
Agent A
   ↓
"请 Agent B 再分析"
   ↓
Agent B
   ↓
...
```

这就是典型的：

> **Agent Communication Loop**

解决方法之一是建立：

```text
Call Stack
```

例如：

```text
Supervisor
 → ArchitectureAgent
   → DatabaseAgent
     → ArchitectureAgent
```

发现：

```text
ArchitectureAgent
```

已经在当前调用链中：

```text
ArchitectureAgent
 → DatabaseAgent
   → ArchitectureAgent ❌
```

直接拒绝。

类似 Java：

```text
A()
 ↓
B()
 ↓
A()
```

本质上就是检测 recursion。

---

# 8. 给 Agent 设置“职责边界”

这个其实比技术限制更重要。

不要让所有 Agent 都拥有：

```text
所有 Tool
+
调用所有 Agent
```

例如：

```text
Supervisor
   │
   ├── Architecture Agent
   │       └── 可以调用 DB Agent
   │
   ├── DB Agent
   │       └── 可以查询数据库
   │
   └── Security Agent
           └── 可以调用 Security Tools
```

而不是：

```text
Every Agent
   ↓
Every Agent
   ↓
Every Tool
```

后者非常容易形成复杂循环。

---

# 9. 使用状态机控制 Agent

生产系统里我非常推荐这个思路。

不要：

```text
LLM 自己决定下一步
```

而是：

```text
             ┌──────────┐
             │   START  │
             └────┬─────┘
                  ↓
             ┌──────────┐
             │ PLANNING │
             └────┬─────┘
                  ↓
             ┌──────────┐
             │ EXECUTE  │
             └────┬─────┘
                  ↓
             ┌──────────┐
             │ VERIFY   │
             └────┬─────┘
                  ↓
             ┌──────────┐
             │  DONE    │
             └──────────┘
```

状态只能按照允许的方向移动。

例如：

```text
PLANNING
   ↓
EXECUTE
   ↓
VERIFY
   ↓
DONE
```

而不允许：

```text
VERIFY
 ↓
PLANNING
 ↓
EXECUTE
 ↓
VERIFY
 ↓
PLANNING
 ↓
...
```

除非明确允许重试，而且：

```text
retry <= 3
```

---

# 10. Retry 也必须有限制

很多 Agent 无限循环实际上是：

```text
Tool failed
 ↓
Retry
 ↓
Tool failed
 ↓
Retry
 ↓
Tool failed
 ↓
Retry
 ↓
...
```

所以必须：

```java
if (retryCount >= 3) {
    throw new AgentExecutionException(
        "Maximum retry exceeded"
    );
}
```

更进一步，可以使用：

```text
Exponential Backoff
```

例如：

```text
1s
 ↓
2s
 ↓
4s
 ↓
8s
 ↓
STOP
```

---

# 11. 最好建立一个 Agent Guard

如果你以后真正做 Multi-Agent 项目，我建议把这些能力统一抽象出来：

```text
                 Agent Guard
                     │
        ┌────────────┼─────────────┐
        ↓            ↓             ↓
   Max Steps     Timeout       Token Limit
        │            │             │
        ↓            ↓             ↓
   Tool Limit    Retry Limit   Cost Limit
        │            │             │
        └────────────┼─────────────┘
                     ↓
              Loop Detection
                     ↓
                ALLOW / STOP
```

例如：

```java
public class AgentGuard {

    private final int maxSteps = 10;
    private final int maxToolCalls = 20;
    private final int maxRetries = 3;

    public boolean allow(AgentContext context) {

        if (context.getSteps() >= maxSteps) {
            return false;
        }

        if (context.getToolCalls() >= maxToolCalls) {
            return false;
        }

        if (context.getRetries() >= maxRetries) {
            return false;
        }

        if (context.isRepeatedState()) {
            return false;
        }

        if (context.isCycleDetected()) {
            return false;
        }

        return true;
    }
}
```

---

# 12. 最终可以形成一套生产级防护体系

我建议你记住这个架构：

```text
                  User
                   │
                   ↓
              Supervisor
                   │
                   ↓
              Agent Runtime
                   │
          ┌────────┴────────┐
          │   Agent Guard   │
          └────────┬────────┘
                   │
      ┌────────────┼─────────────┐
      ↓            ↓             ↓
  Max Steps     Timeout      Token Limit
      ↓            ↓             ↓
  Tool Limit    Retry Limit   Cost Limit
      └────────────┼─────────────┘
                   ↓
             Loop Detection
                   ↓
              State Machine
                   ↓
             Agent Execution
                   ↓
                Result
```

### 总结

> **Agent 防止无限循环通常采用多层 Guard 机制，包括最大执行步数、Token 限制、Timeout、Tool 调用次数限制、Retry 上限、重复状态检测以及 Agent 调用链的 Cycle Detection。对于 Multi-Agent，还需要通过 Supervisor、State Machine 和 Agent 权限边界控制 Agent 间的循环调用。生产环境一般不会让 LLM 完全自由地控制 Workflow，而是让 LLM 负责决策，把执行流程交给确定性的 Workflow/State Machine 控制。**

