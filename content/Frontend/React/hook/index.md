---
title: React hooks
# tags:
#   - nodejs
date: '2023-11-04'
summary: React Hoods(useState,useEffect,useCallback...)
---

React Hooks 可以理解成：让函数组件拥有状态、生命周期、副作用、Context、DOM 引用等能力的 API。


## 1. React Hooks 全景图

| Hook                   | 主要作用             | 掌握重要度 |
| ---------------------- | ---------------- | ----: |
| `useState`             | 管理组件状态           | ⭐⭐⭐⭐⭐ |
| `useEffect`            | 处理副作用            | ⭐⭐⭐⭐⭐ |
| `useContext`           | 读取 Context       |  ⭐⭐⭐⭐ |
| `useRef`               | 保存引用/DOM/不触发渲染的值 | ⭐⭐⭐⭐⭐ |
| `useMemo`              | 缓存计算结果           |  ⭐⭐⭐⭐ |
| `useCallback`          | 缓存函数             |  ⭐⭐⭐⭐ |
| `useReducer`           | 管理复杂状态           |  ⭐⭐⭐⭐ |
| `useLayoutEffect`      | DOM 更新后、浏览器绘制前执行 |   ⭐⭐⭐ |
| `useImperativeHandle`  | 自定义暴露给父组件的 ref   |   ⭐⭐⭐ |
| `useId`                | 生成稳定 ID          |    ⭐⭐ |
| `useTransition`        | 标记非紧急更新          |   ⭐⭐⭐ |
| `useDeferredValue`     | 延迟更新某个值          |   ⭐⭐⭐ |
| `useSyncExternalStore` | 订阅外部 Store       |    ⭐⭐ |
| `useInsertionEffect`   | CSS-in-JS 插入样式   |     ⭐ |
| Custom Hook            | 自定义逻辑复用          | ⭐⭐⭐⭐⭐ |

---

# 2. `useState` —— 管理状态

最基础，也是最重要的 Hook。

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>{count}</p>

      <button onClick={() => setCount(count + 1)}>
        +
      </button>
    </div>
  );
}
```

核心：

```text
useState()
   ↓
返回
[state, setState]
```

例如：

```javascript
const [count, setCount] = useState(0);
```

`count` 是状态。

`setCount` 修改状态。

修改之后会触发组件重新渲染。

---

# 3. `useEffect` —— 处理副作用

这是 React 面试**最重要的 Hook 之一**。

例如请求 API：

```jsx
useEffect(() => {
  fetch("/api/users")
    .then(res => res.json())
    .then(data => {
      console.log(data);
    });
}, []);
```

常见用途：

```text
API 请求
WebSocket
Timer
事件监听
订阅
DOM 操作
第三方组件
```

### `useEffect` 的第二个参数非常重要

#### ① 没有依赖数组

```jsx
useEffect(() => {
  console.log("effect");
});
```

每次渲染后执行。

---

#### ② 空数组

```jsx
useEffect(() => {
  console.log("effect");
}, []);
```

通常用于组件首次挂载后的副作用。

---

#### ③ 有依赖

```jsx
useEffect(() => {
  console.log("userId changed");
}, [userId]);
```

当 `userId` 发生变化时执行。

---

### Cleanup

```jsx
useEffect(() => {

  const timer = setInterval(() => {
    console.log("hello");
  }, 1000);

  return () => {
    clearInterval(timer);
  };

}, []);
```

可以理解：

```text
useEffect
   ↓
执行副作用
   ↓
return cleanup
   ↓
组件卸载/依赖变化时清理
```

---

# 4. `useRef` —— 保存引用，但不会触发重新渲染

这个 Hook 很容易和 `useState` 搞混。

```jsx
const countRef = useRef(0);
```

访问：

```javascript
countRef.current
```

修改：

```javascript
countRef.current++;
```

**修改 `useRef` 不会触发组件重新渲染。**

---

### DOM 操作

这是 `useRef` 最经典的用途。

```jsx
function Input() {

  const inputRef = useRef(null);

  const focusInput = () => {
    inputRef.current.focus();
  };

  return (
    <>
      <input ref={inputRef} />

      <button onClick={focusInput}>
        Focus
      </button>
    </>
  );
}
```

关系：

```text
useRef
  ↓
inputRef
  ↓
DOM Element
  ↓
inputRef.current.focus()
```

---

# 5. `useState` vs `useRef`

|            | `useState` | `useRef` |
| ---------- | ---------- | -------- |
| 保存数据       | ✅          | ✅        |
| 修改后重新渲染    | ✅          | ❌        |
| `.current` | ❌          | ✅        |
| 保存 DOM     | ❌          | ✅        |
| 保存上一次值     | 可以         | 非常适合     |
| 触发 UI 更新   | ✅          | ❌        |

例如：

```javascript
const [count, setCount] = useState(0);
```

调用：

```javascript
setCount(10);
```

会重新渲染。

而：

```javascript
const count = useRef(0);

count.current = 10;
```

不会重新渲染。

### 一句话：

> **需要改变后更新 UI，用 `useState`；只需要保存一个值，不希望触发重新渲染，用 `useRef`。**

---

# 6. `useMemo` —— 缓存计算结果

假设计算非常复杂：

```jsx
const result = useMemo(() => {
  return expensiveCalculation(data);
}, [data]);
```

只有 `data` 发生变化时，才重新计算。

```text
data
 ↓
发生变化
 ↓
重新计算
```

如果 `data` 没变：

```text
data 没变化
 ↓
直接使用之前计算结果
```

---

# 7. `useCallback` —— 缓存函数

例如：

```jsx
const handleClick = useCallback(() => {
  console.log("click");
}, []);
```

它主要缓存的是：

> **函数对象本身**

而 `useMemo` 缓存的是：

> **计算结果**

所以：

```javascript
useMemo
```

和：

```javascript
useCallback
```

可以这样记：

```text
useMemo
   ↓
缓存 value

useCallback
   ↓
缓存 function
```

---

# 8. `useMemo` vs `useCallback`

非常高频。

```jsx
const value = useMemo(() => {
  return calculate(a, b);
}, [a, b]);
```

缓存：

```text
calculate(a, b) 的结果
```

而：

```jsx
const fn = useCallback(() => {
  calculate(a, b);
}, [a, b]);
```

缓存：

```text
函数 fn
```

可以简单记：

> **Memo = Memorize Value**
>
> **Callback = Memorize Function**

---

# 9. `useContext` —— 跨组件共享数据

假设：

```text
App
 ↓
User
 ↓
Profile
 ↓
Avatar
```

如果 `Avatar` 需要 User 信息。

传统方式可能需要：

```text
App
 ↓ props
User
 ↓ props
Profile
 ↓ props
Avatar
```

这就是 **Prop Drilling**。

使用 Context：

```jsx
const UserContext = createContext(null);
```

Provider：

```jsx
<UserContext.Provider value={user}>
  <App />
</UserContext.Provider>
```

子组件：

```jsx
const user = useContext(UserContext);
```

就可以直接获得：

```javascript
user
```

---

# 10. `useReducer` —— 复杂状态管理

当状态比较复杂的时候：

```javascript
const [state, dispatch] = useReducer(reducer, initialState);
```

例如：

```jsx
function reducer(state, action) {

  switch (action.type) {

    case "increment":
      return {
        ...state,
        count: state.count + 1
      };

    case "decrement":
      return {
        ...state,
        count: state.count - 1
      };

    default:
      return state;
  }
}
```

调用：

```javascript
dispatch({
  type: "increment"
});
```

---

### `useState` vs `useReducer`

简单状态：

```text
useState
```

复杂状态：

```text
useReducer
```

例如：

```javascript
const [name, setName] = useState("");
```

非常适合 `useState`。

但如果：

```text
user
 ├── name
 ├── age
 ├── address
 ├── orders
 ├── loading
 └── error
```

状态之间存在复杂关联：

```text
ACTION
 ↓
reducer
 ↓
new state
```

通常 `useReducer` 更清晰。

---

# 11. `useLayoutEffect`

和：

```javascript
useEffect
```

非常容易被问到。

区别主要在执行时机。

大致可以理解：

```text
React 更新 DOM
      ↓
useLayoutEffect
      ↓
浏览器绘制
      ↓
useEffect
```

所以：

### `useLayoutEffect`

适合需要：

```text
读取 DOM
测量 DOM
计算位置
同步修改 DOM
```

例如：

```jsx
useLayoutEffect(() => {
  const height = ref.current.offsetHeight;

  console.log(height);
}, []);
```

而普通 API 请求通常使用：

```javascript
useEffect
```

---

# 12. `useImperativeHandle`

这个 Hook 相对高级。

通常配合：

```text
forwardRef
```

让父组件能够调用子组件暴露的方法。

例如子组件：

```jsx
useImperativeHandle(ref, () => ({
  focus() {
    inputRef.current.focus();
  }
}));
```

父组件：

```javascript
childRef.current.focus();
```

适合：

```text
focus
scroll
reset
open
close
play
pause
```

这种命令式操作。

---

# 13. `useId`

React 提供稳定 ID：

```jsx
const id = useId();
```

例如：

```jsx
<label htmlFor={id}>
  Username
</label>

<input id={id} />
```

特别适合：

```text
表单
label
accessibility
SSR
```

---

# 14. `useTransition`

用于 React 的并发更新场景。

例如：

```jsx
const [isPending, startTransition] = useTransition();
```

然后：

```jsx
startTransition(() => {
  setSearchResult(result);
});
```

告诉 React：

> 这个更新不是特别紧急，可以降低优先级。

例如搜索：

```text
用户输入
 ↓
立即更新 Input
 ↓
搜索结果更新可以稍后
```

可以提高 UI 响应性。

---

# 15. `useDeferredValue`

和 `useTransition` 有点类似。

```jsx
const deferredValue = useDeferredValue(value);
```

例如：

```text
用户输入
 ↓
value：立即更新
 ↓
deferredValue：稍后更新
```

适合：

```text
搜索
大量列表
复杂计算
大型 UI
```

---

# 16. Custom Hook

React 允许我们自己创建 Hook。

命名必须以：

```text
use
```

开头。

例如：

```jsx
function useUser(userId) {

  const [user, setUser] = useState(null);

  useEffect(() => {

    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setUser);

  }, [userId]);

  return user;
}
```

组件：

```jsx
function UserProfile({ userId }) {

  const user = useUser(userId);

  return (
    <div>
      {user?.name}
    </div>
  );
}
```

这就是：

> **Custom Hook = 可复用的 React 业务逻辑。**

---

# 17. 最重要的几个 Hook 怎么区分？

你可以记住这张图：

```text
                  React Hooks
                       │
       ┌───────────────┼────────────────┐
       │               │                │
     状态             副作用             引用
       │               │                │
  useState          useEffect         useRef
  useReducer        useLayoutEffect
       │
       │
       └──────────────┐
                      │
                    Context
                      │
                 useContext
                      
                      
       性能优化
          │
    ┌─────┴─────┐
    │           │
 useMemo    useCallback
    │           │
 缓存结果      缓存函数
```

---