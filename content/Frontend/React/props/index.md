---
title: 组件之间传参数
# tags:
#   - nodejs
date: '2023-11-04'
summary: 传子用 props，子传父用 callback，兄弟组件通常通过共同父组件传递；跨层级可以用 Context，更复杂的全局状态可以用 Redux/Zustand 等状态管理方案。
---

React 组件之间传参数:

最核心的一句话：

> **父传子用 props，子传父用 callback，兄弟组件通常通过共同父组件传递；跨层级可以用 Context，更复杂的全局状态可以用 Redux/Zustand 等状态管理方案。**

---

# 一、先看完整分类

```text
                    React 组件通信
                         │
       ┌─────────────────┼─────────────────┐
       ↓                 ↓                 ↓
     父 → 子           子 → 父           兄弟组件
       │                 │                 │
     props          callback函数       状态提升
       │                 │                 │
       └─────────────────┴────────┬────────┘
                                  ↓
                              共同父组件


       跨层级组件
            │
      ┌─────┴─────┐
      ↓           ↓
   Context    状态管理
                │
          Redux / Zustand
```

另外还有一些特殊方式：

* `ref / useImperativeHandle`：父组件调用子组件暴露的方法
* URL / Router 参数：页面级组件之间传递
* 浏览器存储：localStorage/sessionStorage
* 事件总线：特殊场景使用，不建议作为常规方案

---

# 二、父组件 → 子组件：props

这是 React **最基本、最重要**的组件通信方式。

例如：

```jsx
function Parent() {
  const user = {
    id: 1,
    name: "Tom"
  };

  return <Child user={user} />;
}
```

子组件：

```jsx
function Child({ user }) {
  return (
    <div>
      {user.name}
    </div>
  );
}
```

数据流：

```text
Parent
   │
   │ props
   ↓
Child
```

可以传：

```jsx
<Child
  name="Tom"
  age={20}
  user={user}
  onClick={handleClick}
  items={items}
/>
```

本质上都是：

```text
props
```

---

# 三、父传子不仅可以传数据，也可以传函数

例如：

```jsx
function Parent() {

  const handleDelete = (id) => {
    console.log("delete", id);
  };

  return (
    <Child
      onDelete={handleDelete}
    />
  );
}
```

子组件：

```jsx
function Child({ onDelete }) {

  return (
    <button onClick={() => onDelete(100)}>
      Delete
    </button>
  );
}
```

这里实际上是：

```text
Parent
   │
   │ onDelete function
   ↓
Child
```

所以：

> **props 不仅可以传数据，也可以传函数。**

这正是实现“子传父”的基础。

---

# 四、子组件 → 父组件：Callback

React 没有类似 Vue `$emit` 的机制。

React 最常见的方式是：

> **父组件把一个函数通过 props 传给子组件，子组件调用这个函数，把数据传回父组件。**

例如：

```jsx
function Parent() {

  const handleMessage = (message) => {
    console.log("来自子组件：", message);
  };

  return (
    <Child
      onMessage={handleMessage}
    />
  );
}
```

子组件：

```jsx
function Child({ onMessage }) {

  const handleClick = () => {
    onMessage("Hello Parent");
  };

  return (
    <button onClick={handleClick}>
      Send
    </button>
  );
}
```

数据流：

```text
Parent
   │
   │ function
   ↓
Child
   │
   │ callback(data)
   ↓
Parent
```

---

# 五、这也是 React 单向数据流的核心

React 强调：

```text
             Parent
             │
       props ↓
             │
          Child
```

而不是让 Child 直接修改 Parent 的数据。

正确的方式：

```text
Parent
  │
  │ state
  ↓
Child
  │
  │ callback
  ↓
Parent
  │
  │ setState
  ↓
重新 render
```

例如：

```jsx
function Parent() {

  const [count, setCount] = useState(0);

  return (
    <>
      <div>{count}</div>

      <Child
        onIncrement={() => setCount(c => c + 1)}
      />
    </>
  );
}
```

子组件：

```jsx
function Child({ onIncrement }) {

  return (
    <button onClick={onIncrement}>
      +1
    </button>
  );
}
```

---

# 六、兄弟组件之间：状态提升

这是面试非常常见的问题：

> **两个兄弟组件怎么传参数？**

例如：

```text
        Parent
        /    \
       ↓      ↓
   ChildA   ChildB
```

ChildA 想把数据传给 ChildB。

不要直接：

```text
ChildA → ChildB
```

而是：

```text
ChildA
   │
   ↓
Parent
   │
   ↓
ChildB
```

也就是：

> **把共享状态提升到最近的共同父组件。**

---

## 示例

```jsx
function Parent() {

  const [message, setMessage] = useState("");

  return (
    <>
      <ChildA
        onMessage={setMessage}
      />

      <ChildB
        message={message}
      />
    </>
  );
}
```

ChildA：

```jsx
function ChildA({ onMessage }) {

  return (
    <button onClick={() => onMessage("Hello")}>
      Send
    </button>
  );
}
```

ChildB：

```jsx
function ChildB({ message }) {

  return (
    <div>
      {message}
    </div>
  );
}
```

数据流：

```text
              Parent
             /      \
            ↓        ↓
        ChildA      ChildB
            │          ↑
            │          │
            └──────────┘
               state
```

这叫：

> **Lifting State Up（状态提升）**

---

# 七、跨层级组件：Context

假设组件结构：

```text
App
 │
 ├── A
 │    └── B
 │         └── C
 │              └── D
```

现在 App 有一个：

```jsx
user
```

需要传给 D。

传统 props：

```text
App
 ↓
A
 ↓
B
 ↓
C
 ↓
D
```

即使 A/B/C 根本不需要 user，也得一层层传。

这就是：

> **Prop Drilling（Props 逐层传递）**

---

# 八、使用 Context

可以：

```jsx
const UserContext = createContext(null);
```

Provider：

```jsx
function App() {

  const user = {
    id: 1,
    name: "Tom"
  };

  return (
    <UserContext.Provider value={user}>
      <A />
    </UserContext.Provider>
  );
}
```

D：

```jsx
function D() {

  const user = useContext(UserContext);

  return (
    <div>
      {user.name}
    </div>
  );
}
```

数据：

```text
App
 │
 │ Context Provider
 ↓
 A
 ↓
 B
 ↓
 C
 ↓
 D
 │
 │ useContext
 ↓
 user
```

中间组件：

```text
A
B
C
```

完全不需要传 `user`。

---

# 九、Context 适合什么场景？

典型场景：

### 1. 用户信息

```text
CurrentUser
```

### 2. Theme

```text
light / dark
```

### 3. 国际化

```text
zh-CN / en-US
```

### 4. 权限

```text
roles / permissions
```

### 5. 全局配置

```text
API config
```

---

# 十、Context 不等于 Redux

这是面试经常问的：

> Context 和 Redux 有什么区别？

简单理解：

```text
Context
    ↓
解决“数据怎么传过去”
```

而 Redux：

```text
Redux
    ↓
解决“复杂共享状态怎么管理”
```

例如：

```text
大型应用
    │
    ├── User
    ├── Cart
    ├── Order
    ├── Permission
    ├── Notification
    └── Product
```

如果状态之间存在大量：

```text
读取
修改
异步操作
派生状态
状态追踪
```

通常会考虑状态管理库。

现代 React 项目除了 Redux，也经常使用 Zustand 等方案。

---

# 十一、ref：父组件调用子组件方法

这是比较特殊的一种通信方式。

正常情况下：

```text
Parent
   ↓ props
Child
```

但是有时候父组件需要：

> “直接调用子组件的方法。”

例如：

```text
Parent
   ↓
调用 Child.focus()
```

可以使用：

```jsx
useRef
```

配合：

```jsx
useImperativeHandle
```

---

## 示例

子组件：

```jsx
const Input = forwardRef(function Input(props, ref) {

  const inputRef = useRef();

  useImperativeHandle(ref, () => ({
    focus() {
      inputRef.current.focus();
    }
  }));

  return (
    <input ref={inputRef} />
  );
});
```

父组件：

```jsx
function Parent() {

  const inputRef = useRef();

  const handleClick = () => {
    inputRef.current.focus();
  };

  return (
    <>
      <Input ref={inputRef} />

      <button onClick={handleClick}>
        Focus
      </button>
    </>
  );
}
```

数据/控制关系：

```text
Parent
   │
   │ ref
   ↓
Child
   │
   │ expose
   ↓
focus()
```

---

# 十二、什么时候使用 ref？

比较典型的是：

```text
focus()
scrollTo()
open()
close()
reset()
play()
pause()
```

例如：

```jsx
modalRef.current.open();
```

但是不要把它当成普通状态传递机制。

React 更推荐：

```text
props
state
callback
context
```

而不是大量使用：

```text
ref
```

因为 ref 是一种**命令式操作**。

---

# 十三、Router 参数

如果是页面之间传递参数，还可以使用 React Router。

例如：

```text
/users/100
```

其中：

```text
100
```

就是：

```text
userId
```

页面可以读取：

```jsx
const { userId } = useParams();
```

也可以通过 query：

```text
/users?id=100
```

读取：

```jsx
const [searchParams] = useSearchParams();

const id = searchParams.get("id");
```

这类参数实际上是：

> **URL 层面的组件通信。**

特别适合：

```text
详情页
搜索条件
分页
筛选条件
页面状态
```

---

# 十四、localStorage / sessionStorage

还有一种方式：

```text
Component A
      ↓
localStorage
      ↓
Component B
```

例如：

```jsx
localStorage.setItem(
  "token",
  token
);
```

另一个组件：

```jsx
const token = localStorage.getItem("token");
```

但是需要注意：

> **localStorage 更适合持久化数据，不应该作为普通 React 组件通信的首选方案。**

因为它本身不会自动触发 React render。

---

# 十五、事件总线

还可以：

```text
Component A
      │
      ↓
 Event Bus
      │
      ↓
Component B
```

例如以前常见：

```text
EventEmitter
Pub/Sub
window.dispatchEvent
```

这种方式可以实现：

```text
A → B
```

甚至：

```text
A → X
A → Y
A → Z
```

但是在现代 React 项目里：

> **不推荐把 Event Bus 当成主要组件通信机制。**

因为容易造成：

```text
数据来源不清楚
生命周期难管理
事件订阅泄漏
代码难维护
调试困难
```

---

# 十六、最终可以整理成这张表

| 场景       | 推荐方式             | 典型 API                       |
| -------- | ---------------- | ---------------------------- |
| 父 → 子    | Props            | `props`                      |
| 子 → 父    | Callback         | `onXXX`                      |
| 兄弟 → 兄弟  | 状态提升             | `useState`                   |
| 跨多层组件    | Context          | `createContext/useContext`   |
| 大型应用共享状态 | State Management | Redux/Zustand                |
| 父调用子方法   | Ref              | `useRef/useImperativeHandle` |
| 页面 → 页面  | URL 参数           | React Router                 |
| 持久化共享数据  | Storage          | localStorage                 |
| 解耦事件通信   | Event Bus        | EventEmitter                 |

---

# 十七、总结

```text
组件通信
    │
    ├── 父子？
    │    │
    │    ├── 父 → 子 → props
    │    │
    │    └── 子 → 父 → callback
    │
    ├── 兄弟？
    │    │
    │    └── 状态提升到共同父组件
    │
    ├── 跨很多层？
    │    │
    │    └── Context
    │
    ├── 多个模块共享复杂状态？
    │    │
    │    └── Redux / Zustand
    │
    ├── 父组件需要命令式调用子组件？
    │    │
    │    └── ref / useImperativeHandle
    │
    └── 页面之间？
         │
         └── Router 参数
```

---
