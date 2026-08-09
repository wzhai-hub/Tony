---
title: 类组件与函数组件区别？
# tags:
#   - nodejs
date: '2023-11-04'
summary: 类组件是基于 ES6 Class 实现的组件，通过 `this.state` 管理状态，通过 `this.setState` 更新状态，并通过 `componentDidMount`、`componentDidUpdate`、`componentWillUnmount` 等生命周期方法管理组件生命周期。 函数组件本质上是 JavaScript 函数，通过 Hooks，例如 `useState` 管理状态，通过 `useEffect` 处理副作用和生命周期相关逻辑。类组件需要处理 `this` 绑定，代码相对复杂；函数组件没有 `this`，代码更加简洁，并且可以通过 Custom Hooks 很方便地复用业务逻辑。
---


在 React 中，类组件（Class Component）和函数组件（Function Component）是两种组件写法。现在实际开发中，函数组件 + Hooks 是主流。

## 1. 最直观的区别

### 类组件

使用 `class` 和 `extends React.Component`：

```jsx
class User extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      name: "Vincent"
    };
  }

  handleClick = () => {
    this.setState({
      name: "Tom"
    });
  };

  render() {
    return (
      <div>
        <p>{this.state.name}</p>
        <button onClick={this.handleClick}>
          Change
        </button>
      </div>
    );
  }
}
```

### 函数组件

普通 JavaScript 函数：

```jsx
function User() {
  const [name, setName] = React.useState("Vincent");

  const handleClick = () => {
    setName("Tom");
  };

  return (
    <div>
      <p>{name}</p>
      <button onClick={handleClick}>
        Change
      </button>
    </div>
  );
}
```

---

# 2. 核心区别

| 对比         | 类组件                   | 函数组件         |
| ---------- | --------------------- | ------------ |
| 写法         | `class`               | `function`   |
| 状态         | `this.state`          | `useState`   |
| 修改状态       | `this.setState()`     | `setState()` |
| 生命周期       | `componentDidMount` 等 | `useEffect`  |
| `this`     | 大量使用                  | 不需要          |
| 代码量        | 相对较多                  | 更简洁          |
| 逻辑复用       | HOC、Render Props      | Hooks        |
| 学习成本       | 较高                    | 较低           |
| React 当前主流 | 旧项目较多                 | **主流**       |
| 性能         | 没有本质优势                | 没有本质劣势       |

---

# 3. State 的区别

### 类组件

```jsx
class Counter extends React.Component {

  state = {
    count: 0
  };

  increment = () => {
    this.setState({
      count: this.state.count + 1
    });
  };

  render() {
    return <div>{this.state.count}</div>;
  }
}
```

通过：

```javascript
this.state
```

读取状态。

通过：

```javascript
this.setState()
```

修改状态。

---

### 函数组件

```jsx
function Counter() {

  const [count, setCount] = useState(0);

  const increment = () => {
    setCount(count + 1);
  };

  return <div>{count}</div>;
}
```

通过：

```javascript
count
```

读取。

通过：

```javascript
setCount()
```

修改。

---

# 4. 生命周期区别

类组件有比较明确的生命周期：

```text
Mount
  ↓
constructor
  ↓
render
  ↓
componentDidMount
  ↓
Update
  ↓
render
  ↓
componentDidUpdate
  ↓
Unmount
  ↓
componentWillUnmount
```

例如：

```jsx
class User extends React.Component {

  componentDidMount() {
    console.log("组件挂载");
  }

  componentDidUpdate() {
    console.log("组件更新");
  }

  componentWillUnmount() {
    console.log("组件卸载");
  }

  render() {
    return <div>Hello</div>;
  }
}
```

---

函数组件没有这些生命周期方法。

主要使用：

```javascript
useEffect()
```

例如：

```jsx
function User() {

  useEffect(() => {
    console.log("组件挂载");

    return () => {
      console.log("组件卸载");
    };
  }, []);

  return <div>Hello</div>;
}
```

可以理解成：

```text
componentDidMount
        ↓
useEffect(..., [])
```

但是**不要简单认为 `useEffect` 就等于 `componentDidMount`**。

这是面试中一个很重要的细节。

---

# 5. Props 的区别

类组件：

```jsx
class User extends React.Component {

  render() {
    return <div>{this.props.name}</div>;
  }
}
```

函数组件：

```jsx
function User(props) {
  return <div>{props.name}</div>;
}
```

还可以使用解构：

```jsx
function User({ name, age }) {
  return (
    <div>
      {name} - {age}
    </div>
  );
}
```

函数组件明显更加简洁。

---

# 6. `this` 是两者非常重要的区别

类组件大量使用 `this`：

```jsx
class User extends React.Component {

  state = {
    name: "Vincent"
  };

  handleClick() {
    console.log(this.state.name);
  }

  render() {
    return (
      <button onClick={this.handleClick}>
        Click
      </button>
    );
  }
}
```

这里甚至可能遇到：

```text
this is undefined
```

因此以前经常需要：

```jsx
this.handleClick = this.handleClick.bind(this);
```

或者使用箭头函数：

```jsx
handleClick = () => {
  console.log(this.state.name);
};
```

---

函数组件没有这个问题：

```jsx
function User() {

  const [name, setName] = useState("Vincent");

  const handleClick = () => {
    console.log(name);
  };

  return (
    <button onClick={handleClick}>
      Click
    </button>
  );
}
```

**没有 `this`，也就没有 `this` 绑定问题。**

---

# 7. 逻辑复用的区别

这是为什么 Hooks 出现后函数组件越来越重要。

以前类组件复用逻辑，经常使用：

### HOC

```jsx
const UserWithAuth = withAuth(User);
```

或者：

### Render Props

```jsx
<DataProvider>
  {data => (
    <User data={data} />
  )}
</DataProvider>
```

这些方式容易造成组件层级复杂：

```text
HOC
 ↓
HOC
 ↓
HOC
 ↓
Component
```

---

函数组件可以使用 **Custom Hook**：

```jsx
function useUser() {

  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser().then(setUser);
  }, []);

  return user;
}
```

组件直接使用：

```jsx
function User() {

  const user = useUser();

  return (
    <div>
      {user?.name}
    </div>
  );
}
```

这也是现代 React 非常重要的思想：

> **把可复用的业务逻辑抽取成 Custom Hook。**

---

# 8. 一个面试非常重要的问题：函数组件是不是每次都会重新执行？

**是。**

例如：

```jsx
function Counter() {

  console.log("render");

  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

当：

```javascript
setCount()
```

发生之后，React 会重新执行：

```javascript
Counter()
```

也就是说：

```text
第一次：

Counter()
 ↓
JSX
 ↓
React DOM


setCount()
 ↓
Counter() 重新执行
 ↓
新的 JSX
 ↓
React Diff
 ↓
更新 DOM
```

但是这并不意味着整个 DOM 都重新创建。

React 会进行 **Reconciliation / Diff**。

---

# 9. 类组件是不是不会重新执行？

类组件不是简单的“重新执行整个 class”。

例如：

```jsx
class Counter extends React.Component {

  state = {
    count: 0
  };

  render() {
    console.log("render");

    return <div>{this.state.count}</div>;
  }
}
```

调用：

```javascript
this.setState(...)
```

之后，React 会调用：

```javascript
render()
```

而不是重新创建整个组件实例。

所以可以简单记：

```text
函数组件：
重新执行 Function

类组件：
保留 Class Instance
重新执行 render()
```

---

# 10. 为什么现在推荐函数组件？

主要有几个原因：

### ① 更简单

```jsx
function User() {
    const [name, setName] = useState("");
}
```

比：

```jsx
class User extends React.Component {
    constructor(props) {
        super(props);
        this.state = {};
    }
}
```

简单很多。

### ② 没有 this

避免：

```text
this
this.state
this.props
this.setState
this.bind
```

### ③ Hooks 让逻辑复用更加自然

```text
useState
useEffect
useMemo
useCallback
useRef
useContext
Custom Hook
```

### ④ 更容易拆分业务逻辑

例如：

```text
useUser()
useAuth()
usePermission()
useRequest()
useWebSocket()
usePagination()
```

可以把复杂业务逻辑拆开。

---

# 11. 总结：React 类组件和函数组件有什么区别？

> 类组件是基于 ES6 Class 实现的组件，通过 `this.state` 管理状态，通过 `this.setState` 更新状态，并通过 `componentDidMount`、`componentDidUpdate`、`componentWillUnmount` 等生命周期方法管理组件生命周期。
>
> 函数组件本质上是 JavaScript 函数，通过 Hooks，例如 `useState` 管理状态，通过 `useEffect` 处理副作用和生命周期相关逻辑。
>
> 类组件需要处理 `this` 绑定，代码相对复杂；函数组件没有 `this`，代码更加简洁，并且可以通过 Custom Hooks 很方便地复用业务逻辑。
>
> 在现代 React 开发中，**函数组件 + Hooks 是主流方式**，但理解类组件仍然很重要，因为很多历史 React 项目仍然使用类组件。

