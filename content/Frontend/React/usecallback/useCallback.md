---
title: React hooks
# tags:
#   - nodejs
date: '2023-11-04'
summary: useCallback是 React 提供的一个性能优化 Hook，用于缓存函数引用。当依赖数组中的值没有发生变化时，多次组件 render 会返回同一个函数引用；当依赖发生变化时，才会返回新的函数。它最常见的使用场景是和 `React.memo` 配合，避免父组件重新渲染时，因为函数引用变化导致子组件不必要的重新渲染。另外，当一个函数作为 `useEffect` 等 Hook 的 dependency 时，也可以使用 `useCallback` 保持函数引用稳定。使用时最重要的是正确维护 dependency array，避免 stale closure。并且不能滥用 `useCallback`，因为它本身也是一种性能优化机制，不是所有函数都需要缓存。
---

`useCallback` 是 React Hooks 里**非常重要、也非常容易被滥用**的一个 Hook。


> **useCallback 用来缓存函数的引用（函数对象），当依赖不变时，多次渲染之间返回同一个函数引用，从而避免因为函数引用变化导致不必要的重新渲染或 Effect 重复执行。** ([React][1])

---

# 一、useCallback 到底解决什么问题？

先看一个最简单的例子：

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    console.log("click");
  };

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>

      <Child onClick={handleClick} />
    </>
  );
}
```

假设：

```jsx
const Child = React.memo(function Child({ onClick }) {
  console.log("Child render");

  return <button onClick={onClick}>Child</button>;
});
```

这里有一个很容易忽略的问题。

每次 `Parent` 重新渲染：

```jsx
const handleClick = () => {
  console.log("click");
};
```

都会创建一个**新的函数对象**。

也就是说：

```text
第一次 render：
handleClick ---> Function A

第二次 render：
handleClick ---> Function B

第三次 render：
handleClick ---> Function C
```

虽然三个函数的代码完全一样，但是：

```js
FunctionA !== FunctionB
```

因此 `React.memo` 会认为：

```text
onClick 属性发生变化
        ↓
Child 需要重新渲染
```

这时候 `useCallback` 就有价值了。

---

# 二、useCallback 的基本语法

```jsx
const cachedFunction = useCallback(
  function,
  dependencies
);
```

例如：

```jsx
const handleClick = useCallback(() => {
  console.log("click");
}, []);
```

或者：

```jsx
const handleClick = useCallback(() => {
  console.log("count:", count);
}, [count]);
```

React 官方定义中，`useCallback(fn, dependencies)` 会在依赖不变时返回之前缓存的函数引用；依赖发生变化时，则返回当前 render 中的新函数。依赖使用 `Object.is` 进行比较。 ([React][1])

---

# 三、最核心的理解：缓存的是“函数”，不是函数执行结果

这是理解 `useCallback` 的关键。

### useCallback

```jsx
const fn = useCallback(() => {
  return count * 2;
}, [count]);
```

缓存：

```text
函数本身
```

而不是：

```text
count * 2
```

所以：

```jsx
fn()
```

才会执行函数。

---

## useMemo 则不同

```jsx
const result = useMemo(() => {
  return count * 2;
}, [count]);
```

缓存的是：

```text
count * 2 的计算结果
```

因此可以简单记：

| Hook          | 缓存什么   |
| ------------- | ------ |
| `useCallback` | 函数     |
| `useMemo`     | 函数执行结果 |

React 官方也明确指出，两者本质相关：`useCallback(fn, deps)` 可以理解为一种特殊形式的 `useMemo`，区别主要在于 `useCallback` 直接缓存函数本身。 ([React][2])

---

# 四、功能一：配合 React.memo 避免子组件重复渲染

这是 `useCallback` **最经典的使用场景**。

例如：

```jsx
const Child = React.memo(function Child({ onClick }) {
  console.log("Child render");

  return (
    <button onClick={onClick}>
      Child
    </button>
  );
});
```

父组件：

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    console.log("click");
  };

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        count: {count}
      </button>

      <Child onClick={handleClick} />
    </>
  );
}
```

问题：

```text
Parent render
     ↓
创建新的 handleClick
     ↓
Child 的 onClick 引用变化
     ↓
React.memo 失效
     ↓
Child render
```

---

## 使用 useCallback

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => {
    console.log("click");
  }, []);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        count: {count}
      </button>

      <Child onClick={handleClick} />
    </>
  );
}
```

现在：

```text
第一次 render

handleClick
    ↓
Function A


第二次 render

handleClick
    ↓
Function A


第三次 render

handleClick
    ↓
Function A
```

因为依赖：

```jsx
[]
```

没有变化，所以 React 返回相同的函数引用。

于是：

```text
Parent render
     ↓
handleClick 引用不变
     ↓
Child props 没变
     ↓
React.memo
     ↓
Child 不需要重新 render
```

这也是 React 官方重点介绍的 `useCallback + memo` 使用方式。 ([React][1])

---

# 五、注意：useCallback 单独使用通常没有意义

这是非常重要的面试点。

比如：

```jsx
function App() {
  const handleClick = useCallback(() => {
    console.log("hello");
  }, []);

  return (
    <button onClick={handleClick}>
      Click
    </button>
  );
}
```

这种场景通常：

```text
没有明显收益
```

因为：

```jsx
<button onClick={handleClick}>
```

并没有一个经过 `React.memo` 优化的子组件需要比较这个函数引用。

所以：

> **不要看到函数就使用 useCallback。**

React 官方也明确建议，不应该把 `useCallback` 到处添加，它主要用于有明确性能优化目的的场景。 ([React][1])

---

# 六、功能二：避免 useEffect 无限/频繁执行

这是第二个非常重要的场景。

例如：

```jsx
function ChatRoom({ roomId }) {

  function createOptions() {
    return {
      roomId: roomId
    };
  }

  useEffect(() => {
    const options = createOptions();

    connect(options);

    return () => disconnect();
  }, [createOptions]);

}
```

这里存在问题。

每次 render：

```jsx
function createOptions() {}
```

都会创建新函数。

所以：

```text
第一次：
createOptions = Function A

第二次：
createOptions = Function B

第三次：
createOptions = Function C
```

那么：

```jsx
[createOptions]
```

每次都发生变化。

结果：

```text
render
 ↓
createOptions 变化
 ↓
useEffect
 ↓
重新连接
 ↓
render
 ↓
createOptions 又变化
 ↓
useEffect 又执行
```

---

## 使用 useCallback

```jsx
function ChatRoom({ roomId }) {

  const createOptions = useCallback(() => {
    return {
      roomId: roomId
    };
  }, [roomId]);

  useEffect(() => {

    const options = createOptions();

    connect(options);

    return () => disconnect();

  }, [createOptions]);

}
```

现在：

```text
roomId 不变
   ↓
createOptions 引用不变
   ↓
useEffect 不重复执行
```

如果：

```jsx
roomId = 100
```

变成：

```jsx
roomId = 200
```

那么：

```text
roomId 改变
   ↓
useCallback 产生新函数
   ↓
createOptions 改变
   ↓
useEffect 执行
```

这正是 `useCallback` 的第二个重要用途。 ([React][1])

不过 React 官方还特别提醒：**如果函数只在一个 Effect 中使用，通常直接把函数移动到 Effect 内部更简单，不一定需要 useCallback。** ([React][1])

例如：

```jsx
useEffect(() => {

  function createOptions() {
    return {
      roomId
    };
  }

  const options = createOptions();

  connect(options);

}, [roomId]);
```

这种写法其实更简单。

---

# 七、功能三：更新 State 时减少依赖

这是实际开发中非常实用的技巧。

例如：

```jsx
function TodoList() {

  const [todos, setTodos] = useState([]);

  const addTodo = useCallback((text) => {

    setTodos([
      ...todos,
      {
        id: Date.now(),
        text
      }
    ]);

  }, [todos]);

}
```

这里：

```jsx
[todos]
```

是必须的。

因为函数使用了：

```jsx
todos
```

但是可以进一步优化。

改成：

```jsx
function TodoList() {

  const [todos, setTodos] = useState([]);

  const addTodo = useCallback((text) => {

    setTodos(todos => [
      ...todos,
      {
        id: Date.now(),
        text
      }
    ]);

  }, []);

}
```

现在：

```jsx
[]
```

就可以了。

为什么？

因为：

```jsx
setTodos(todos => ...)
```

使用的是 React 提供的 **state updater function**。

所以不需要从闭包里面读取旧的 `todos`。

React 官方也推荐这种方式来减少 memoized callback 的依赖。 ([React][1])

---

# 八、为什么这里非常重要？

假设：

```jsx
const addTodo = useCallback(() => {
  setTodos([...todos, newTodo]);
}, [todos]);
```

每当：

```text
todos 改变
```

就会：

```text
addTodo 改变
```

如果：

```jsx
<Child onAdd={addTodo} />
```

那么 Child 也可能重新渲染。

而：

```jsx
const addTodo = useCallback(() => {
  setTodos(todos => [...todos, newTodo]);
}, []);
```

则：

```text
todos 改变
      ↓
addTodo 引用不变
```

这可以减少不必要的依赖变化。

---

# 九、功能四：自定义 Hook 返回函数

假设你自己写一个 Hook：

```jsx
function useRouter() {

  const navigate = (url) => {
    // ...
  };

  const goBack = () => {
    // ...
  };

  return {
    navigate,
    goBack
  };
}
```

调用：

```jsx
const { navigate } = useRouter();
```

每次组件 render，`navigate` 都可能是新的函数。

因此可以：

```jsx
function useRouter() {

  const navigate = useCallback((url) => {
    // ...
  }, []);

  const goBack = useCallback(() => {
    // ...
  }, []);

  return {
    navigate,
    goBack
  };
}
```

这样自定义 Hook 的消费者就可以更容易进行优化。

React 官方也建议，自定义 Hook 返回的函数可以使用 `useCallback` 包装，以便消费者能够优化自己的组件。 ([React][1])

---

# 十、useCallback 最容易犯的错误：依赖数组写错

这是面试非常喜欢问的。

例如：

```jsx
function User({ userId }) {

  const handleClick = useCallback(() => {
    console.log(userId);
  }, []);

}
```

这里有问题。

函数使用了：

```jsx
userId
```

但是：

```jsx
[]
```

没有声明。

于是可能产生 **stale closure（过期闭包）**。

例如：

```text
第一次 render

userId = 100
handleClick 捕获 100


第二次 render

userId = 200

但是 handleClick 没有更新

handleClick 仍然使用 100
```

所以应该：

```jsx
const handleClick = useCallback(() => {
  console.log(userId);
}, [userId]);
```

React 官方要求依赖数组包含 callback 中使用的响应式值，例如 props、state 和组件内部声明的变量/函数。 ([React][1])

---

# 十一、不要随便写空数组 []

很多初学者看到：

```jsx
useCallback(() => {
  ...
}, []);
```

就认为：

> “这样性能最好。”

这是错误的。

例如：

```jsx
const handleClick = useCallback(() => {
  console.log(count);
}, []);
```

如果 `count` 会变化，那么这个 callback 很可能一直看到旧的 `count`。

正确：

```jsx
const handleClick = useCallback(() => {
  console.log(count);
}, [count]);
```

---

# 十二、依赖数组到底怎么判断？

记住一个简单规则：

> **Callback 里面使用了哪些来自组件外部的响应式变量，就考虑哪些变量是否应该作为 dependency。**

例如：

```jsx
function App({ userId, theme }) {

  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => {

    console.log(userId);

    console.log(theme);

    console.log(count);

  }, [userId, theme, count]);

}
```

这里：

```text
userId
theme
count
```

都被 callback 使用了。

因此：

```jsx
[userId, theme, count]
```

---

# 十三、useCallback 和 useMemo 的区别

这个一定要掌握。

### useCallback

```jsx
const fn = useCallback(() => {
  return count * 2;
}, [count]);
```

缓存：

```text
函数
```

---

### useMemo

```jsx
const result = useMemo(() => {
  return count * 2;
}, [count]);
```

缓存：

```text
计算结果
```

可以这样记：

```text
useCallback
      ↓
Callback
      ↓
函数

useMemo
      ↓
Memoized Value
      ↓
值
```

---

# 十四、useCallback + useMemo + memo 三者关系

这是 React 性能优化中一个非常重要的组合。

```text
                Parent
                  │
                  │
          useCallback
                  │
                  ↓
              function
                  │
                  ↓
               Child
                  │
                memo
                  │
                  ↓
         避免不必要 render
```

例如：

```jsx
const Child = memo(function Child({ onClick }) {

  console.log("Child render");

  return <button onClick={onClick}>Click</button>;

});
```

父组件：

```jsx
function Parent() {

  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => {
    console.log("click");
  }, []);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>

      <Child onClick={handleClick} />
    </>
  );

}
```

这里：

```text
useCallback
    ↓
保证 onClick 引用稳定

memo
    ↓
发现 props 没变化

Child
    ↓
跳过 render
```

React 官方将 `memo` 与 `useMemo`、`useCallback` 都作为减少不必要重新渲染的性能优化工具。 ([React][3])

---

# 十五、useCallback 并不会阻止函数创建

这是一个非常容易被误解的地方。

很多人以为：

```jsx
const fn = useCallback(() => {
  console.log("hello");
}, []);
```

意味着 React 根本不会创建新函数。

实际上不是。

每次组件执行：

```jsx
() => {
  console.log("hello");
}
```

这个函数表达式仍然会产生一个新的函数。

React 做的是：

```text
render 1
   ↓
创建 Function A
   ↓
useCallback 保存 Function A


render 2
   ↓
创建 Function B
   ↓
发现 dependencies 没变化
   ↓
返回之前缓存的 Function A
```

所以：

```text
创建函数 ≠ 返回给你的函数引用发生变化
```

React 官方文档明确说明了这一点。 ([React][1])

---

# 十六、不要在循环中使用 useCallback

错误：

```jsx
items.map(item => {

  const handleClick = useCallback(() => {
    console.log(item);
  }, [item]);

  return <Item onClick={handleClick} />;

});
```

这是不允许的。

Hook 必须在组件顶层调用，不能放在：

```text
if
for
while
map
普通嵌套函数
```

里面。 ([React][1])

应该拆组件：

```jsx
function List({ items }) {

  return items.map(item => (
    <ListItem
      key={item.id}
      item={item}
    />
  ));

}

function ListItem({ item }) {

  const handleClick = useCallback(() => {
    console.log(item);
  }, [item]);

  return (
    <Item onClick={handleClick} />
  );
}
```

---

# 十七、useCallback 并不是越多越好

这是你面试 Senior Java/Full Stack / React 岗位时特别值得掌握的一点。

不要这样：

```jsx
function App() {

  const fn1 = useCallback(() => {}, []);
  const fn2 = useCallback(() => {}, []);
  const fn3 = useCallback(() => {}, []);
  const fn4 = useCallback(() => {}, []);
  const fn5 = useCallback(() => {}, []);

}
```

然后认为：

> “我做了性能优化。”

不一定。

因为 `useCallback` 本身也有：

```text
依赖比较
缓存管理
代码复杂度
```

等成本。

而且代码会变得更难理解。

React 官方目前明确建议：`useCallback` 应该作为性能优化手段，而不是保证代码正确性的手段；如果没有明确的性能收益，不需要到处使用。 ([React][1])

---

# 十八、什么时候应该使用？

我建议你把下面这个表直接记住。

| 场景                          | 是否推荐    |
| --------------------------- | ------- |
| 普通事件函数                      | ❌ 通常不需要 |
| 函数传给普通子组件                   | ❌ 通常不需要 |
| 函数传给 `React.memo` 子组件       | ✅ 常见场景  |
| 函数作为 `useEffect` dependency | ✅ 可能需要  |
| 自定义 Hook 返回函数               | ✅ 常见    |
| 减少 callback dependency      | ✅ 可以    |
| 为了“看起来高级”                   | ❌       |
| 所有函数全部 useCallback          | ❌       |
| 解决代码 bug                    | ❌       |
| 性能分析发现子组件重复渲染               | ✅       |

---

# 十九、一个完整实战案例

比如：

```jsx
const UserList = memo(function UserList({
  users,
  onDelete
}) {

  console.log("UserList render");

  return (
    <div>
      {users.map(user => (
        <div key={user.id}>
          {user.name}

          <button onClick={() => onDelete(user.id)}>
            Delete
          </button>
        </div>
      ))}
    </div>
  );
});
```

父组件：

```jsx
function App() {

  const [users, setUsers] = useState([
    { id: 1, name: "Tom" },
    { id: 2, name: "Jerry" }
  ]);

  const [count, setCount] = useState(0);

  const handleDelete = useCallback((id) => {

    setUsers(users =>
      users.filter(user => user.id !== id)
    );

  }, []);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        count: {count}
      </button>

      <UserList
        users={users}
        onDelete={handleDelete}
      />
    </>
  );
}
```

这里的关键：

```jsx
const handleDelete = useCallback((id) => {
  setUsers(users =>
    users.filter(user => user.id !== id)
  );
}, []);
```

为什么可以：

```jsx
[]
```

因为我们没有直接读取：

```jsx
users
```

而是：

```jsx
setUsers(users => ...)
```

使用 updater function。

所以：

```text
count 改变
   ↓
App render
   ↓
handleDelete 引用不变
   ↓
users 引用不变
   ↓
UserList props 不变
   ↓
React.memo
   ↓
UserList 不重新 render
```

这是一个非常典型的真实项目优化方式。

---

# 二十、现在还有一个新的知识点：React Compiler

如果你学习的是现代 React，这一点也值得知道。

React 官方现在已经提供 **React Compiler**，它可以自动对值和函数进行 memoization，从而减少手动写 `useCallback`、`useMemo` 的需要。 ([React][1])

所以未来的趋势是：

```text
过去：

开发者
 ↓
手动 useCallback
 ↓
手动 useMemo
 ↓
手动 memo


现在/未来：

React Compiler
 ↓
自动进行部分 memoization
```

但这并不意味着你现在就不需要学习 `useCallback`。

**面试中仍然非常重要**，因为它背后考察的是：

```text
JavaScript 函数引用
        ↓
React Render
        ↓
Object.is
        ↓
React.memo
        ↓
闭包
        ↓
Hook dependency
        ↓
性能优化
```

---


## 二十一、最后用一张图记住

```text
                 useCallback
                      │
                      ↓
               缓存函数引用
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
      React.memo              useEffect
          │                       │
          ↓                       ↓
  子组件避免重复 render      避免 Effect
                            频繁执行
          │
          ↓
     性能优化


关键：
────────────────────────────
useCallback ≠ 执行函数
useCallback ≠ 缓存返回值
useCallback = 缓存函数引用
────────────────────────────

注意：
────────────────────────────
1. dependency 必须正确
2. 防止 stale closure
3. 不要到处使用
4. 不能在 if/for/map 中调用
5. 经常和 memo 配合
6. 可以使用 updater function 减少依赖
────────────────────────────
```
