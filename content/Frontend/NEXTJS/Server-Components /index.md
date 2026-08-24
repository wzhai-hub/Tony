---
title: Next.js Server Components 深入解析：重新理解 Server/Client Boundary
# tags:
#   - nodejs
date: '2026-8-05'
summary: Server 负责“数据和生成”，Client 负责“交互和状态”。
---


> **Server Components 不是“运行在服务器上的普通 React Component”。**
>
> 它代表的是 React 对组件运行环境的一次重新定义：组件不再天然属于浏览器，而是可以明确地运行在 Server Runtime 或 Browser Runtime。
>
> 而 Next.js App Router 真正重要的能力之一，就是围绕这种 Server/Client Boundary 建立了一套完整的应用架构。

---

## 1. 为什么需要重新理解 React Component？

传统 React 开发模型中，我们通常认为：

```text
React Component
       │
       ▼
Browser
       │
       ├── DOM
       ├── State
       ├── Event
       └── API
```

组件最终运行在浏览器。

例如：

```tsx
function UserProfile() {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch("/api/user")
      .then(res => res.json())
      .then(setUser);
  }, []);

  return <div>{user?.name}</div>;
}
```

这是一种典型的 Client-centric 思维。

但现代 Web Application 的很多逻辑其实并不需要浏览器执行：

```text
Database Query
Authentication
Authorization
Secret Management
Backend API
File System
Data Aggregation
```

如果这些逻辑全部通过浏览器完成：

```text
Browser
   │
   ▼
JavaScript
   │
   ▼
API
   │
   ▼
Backend
   │
   ▼
Database
```

就会产生额外的：

* JavaScript Bundle
* Network Request
* Serialization
* Hydration
* Client-side State
* Loading State

React Server Components 的核心思想就是：

> **让组件可以在 Server Runtime 中完成数据获取和 UI 生成，而不是默认把所有逻辑发送到浏览器。**

---

# 2. Server Components 到底是什么？

首先纠正一个常见误解：

> Server Component ≠ SSR Component。

这是两个不同层次的概念。

### SSR

讨论的是：

> HTML 在哪里生成？

### Server Components

讨论的是：

> React Component 在哪里执行？

可以这样理解：

```text
SSR
│
└── Rendering Strategy

Server Components
│
└── Component Execution Model
```

因此：

```text
Server Component
        │
        ├── 可以参与 Static Rendering
        │
        ├── 可以参与 Dynamic Rendering
        │
        └── 可以通过 Streaming 输出结果
```

Server Component 并不简单等于：

```text
Server Component = SSR
```

这是理解 Next.js 的第一个关键点。

---

# 3. Next.js App Router 默认就是 Server Component

在 App Router 中：

```tsx
export default function Page() {
  return <h1>Hello</h1>;
}
```

默认情况下，它是 Server Component。

也就是说：

```text
app/page.tsx
       │
       ▼
Server Component
       │
       ▼
Server Runtime
```

而不是：

```text
app/page.tsx
       │
       ▼
Browser JavaScript
```

如果需要 Client Component，需要明确声明：

```tsx
"use client";

export default function Counter() {
  ...
}
```

于是：

```text
Server Component
       │
       │ "use client"
       ▼
Client Component
```

这个设计非常重要。

它意味着：

> **Server 是默认状态，Client 是显式能力。**

---

# 4. 为什么 Next.js 要让 Server 成为默认？

考虑一个商品详情页：

```text
ProductPage
│
├── ProductInfo
├── ProductDescription
├── ProductReviews
├── Recommendation
└── AddToCartButton
```

其中真正需要浏览器交互的可能只有：

```text
AddToCartButton
```

传统思路：

```text
ProductPage
     │
     ▼
"use client"
     │
     ├── ProductInfo
     ├── Description
     ├── Reviews
     ├── Recommendation
     └── AddToCartButton
```

结果整个页面进入 Client Bundle。

而 Server Components 的设计：

```text
ProductPage                 Server
│
├── ProductInfo             Server
├── Description             Server
├── Reviews                 Server
├── Recommendation          Server
│
└── AddToCartButton         Client
```

这就是 Server/Client Boundary 的价值。

---

# 5. 什么是 Server/Client Boundary？

可以把 Boundary 理解成：

> **Server Runtime 与 Browser Runtime 之间的一条组件边界。**

例如：

```text
                 Server Runtime
                       │
                       │
             Server/Client Boundary
                       │
                       ▼
                 Browser Runtime
```

在 React Component Tree 中：

```text
Page
│
├── Header                 Server
├── ProductInfo            Server
├── ProductReviews         Server
│
└── AddToCart              Client
```

Boundary 位于：

```text
ProductPage
       │
       ▼
AddToCart
```

也就是说：

```text
Server Component Tree
        │
        ▼
Client Component Subtree
```

这就是一个非常重要的设计原则：

> **Server Component 可以包含 Client Component，但 Client Component 不能直接把 Server Component 当普通子组件重新执行。**

---

# 6. Server Component 可以使用什么？

Server Component 适合处理：

### 数据获取

```tsx
const users = await db.user.findMany();
```

### 后端 API

```tsx
const response = await fetch(
  "https://api.example.com/users"
);
```

### 数据库访问

```tsx
const user = await prisma.user.findUnique({
  where: { id }
});
```

### Secrets

```tsx
const apiKey = process.env.INTERNAL_API_KEY;
```

### 文件系统

```tsx
import fs from "fs";
```

### 服务端业务逻辑

```text
Authentication
Authorization
Data Aggregation
Permission Check
```

这些东西通常都不应该被发送到 Browser。

---

# 7. Server Component 不能做什么？

Server Component 不适合：

```text
useState
useEffect
useReducer
Browser Event
window
document
localStorage
navigator
```

例如：

```tsx
export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

这会产生问题。

因为：

```text
useState
   ↓
需要持续运行的组件状态
   ↓
Browser Runtime
```

因此应该：

```tsx
"use client";

export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

---

# 8. `"use client"` 到底意味着什么？

这是 Next.js 开发中最容易被误解的地方。

很多人认为：

```tsx
"use client";
```

意味着：

> “这个组件只在浏览器运行。”

这种理解并不准确。

更准确的理解是：

> **`"use client"` 声明这个模块属于 Client Component Module Graph，可以使用 Client-only React 能力，并会成为客户端代码的一部分。**

这非常重要。

例如：

```tsx
"use client";

export default function Button() {
  return <button>Click</button>;
}
```

它建立了一个 Client Boundary。

从这个模块开始，它所依赖的模块会进入 Client Component 的依赖图。

因此：

```text
"use client"
     │
     ▼
Client Module Graph
     │
     ├── Button
     ├── useState
     ├── helpers
     └── dependencies
```

这也是为什么 `"use client"` 不能随便加。

---

# 9. `"use client"` 是一个 Boundary Declaration

假设：

```text
app/
├── page.tsx
├── components/
│   ├── Header.tsx
│   ├── ProductList.tsx
│   └── CartButton.tsx
```

如果：

```tsx
// CartButton.tsx

"use client";

export default function CartButton() {
  ...
}
```

那么：

```text
Page
│
├── Header
│
├── ProductList
│
└── CartButton
        │
        ▼
     Client
```

而：

```text
Header
ProductList
Page
```

仍然可以保持 Server Component。

这就是：

> **Fine-grained Client Boundary**

它比“整个页面 Client 化”更加合理。

---

# 10. Server Component → Client Component

这是最常见的组合。

例如：

```tsx
// page.tsx

import AddToCartButton from "./AddToCartButton";

export default async function ProductPage() {
  const product = await getProduct();

  return (
    <div>
      <h1>{product.name}</h1>

      <p>{product.description}</p>

      <AddToCartButton
        productId={product.id}
      />
    </div>
  );
}
```

Client Component：

```tsx
"use client";

import { useState } from "react";

export default function AddToCartButton({
  productId,
}: {
  productId: string;
}) {
  const [loading, setLoading] = useState(false);

  async function addToCart() {
    setLoading(true);

    // ...
  }

  return (
    <button disabled={loading}>
      Add to Cart
    </button>
  );
}
```

架构：

```text
ProductPage
   │
   ├── Product Data
   │      └── Server
   │
   ├── Description
   │      └── Server
   │
   └── AddToCartButton
          └── Client
```

这是非常典型的 Next.js 架构。

---

# 11. 为什么不能简单地在 Client Component 中 import Server Component？

这是很多开发者最容易踩的坑。

假设：

```tsx
"use client";

import ServerComponent from "./ServerComponent";
```

然后：

```tsx
export default async function ServerComponent() {
  const data = await getData();

  return <div>{data}</div>;
}
```

这种设计会破坏 Server/Client Boundary 的语义。

因为 Client Component 的 Module Graph 是：

```text
Browser
   │
   ▼
Client Component
   │
   ▼
Server Component ?
```

但 Server Component 本身需要：

```text
Database
Server Runtime
Secrets
Backend
```

这些东西不能直接进入浏览器。

---

# 12. 正确方式：Server Component 作为 children

一个非常重要的模式是：

```tsx
"use client";

export default function ClientWrapper({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="interactive">
      {children}
    </div>
  );
}
```

然后 Server Component：

```tsx
import ClientWrapper from "./ClientWrapper";

export default async function Page() {
  const data = await getData();

  return (
    <ClientWrapper>
      <ServerContent data={data} />
    </ClientWrapper>
  );
}
```

这里的关键不是：

```text
Client → Server Import
```

而是：

```text
Server
   │
   ├── ServerContent
   │
   └── ClientWrapper
          │
          └── children
```

Client Wrapper 不需要重新执行 Server Component。

它只是接收已经由 Server 生成的内容。

---

# 13. Component Tree 与 Module Graph

真正理解 Server Components，需要区分两个概念：

## Component Tree

```text
Page
│
├── Header
├── Product
└── Button
```

## Module Graph

```text
page.tsx
   │
   ├── Header.tsx
   │
   ├── Product.tsx
   │
   └── Button.tsx
           │
           ├── useState
           └── cart.ts
```

`"use client"` 影响的是：

> **Module Graph**

而 Server/Client Component 最终体现为：

> **Component Tree 中的 Runtime Boundary**

这两个概念必须分开理解。

---

# 14. RSC：Server Components 背后的关键机制

React Server Components 的结果并不是简单地：

```text
Server
   ↓
HTML
```

现代 React Server Components 还会产生：

> **RSC Payload**

可以粗略理解为：

```text
Server Components
       │
       ▼
React Server Render
       │
       ├── HTML
       │
       └── RSC Payload
```

RSC Payload 包含 React Server Component Tree 的相关描述以及 Client Component 的引用等信息。

浏览器随后可以利用这些信息完成 React 应用的更新和交互。

因此现代 Next.js 的页面传输并不应该简单理解为：

```text
Server → HTML → Browser
```

更接近：

```text
Server
   │
   ├── HTML
   │
   └── RSC Payload
          │
          ▼
       Browser
          │
          ▼
   Client Components
          │
          ▼
      Hydration
```

---

# 15. Server Component 的真正价值：减少 Client JavaScript

这是 Server Components 最重要的工程价值之一。

传统方式：

```text
Browser
 │
 ├── React Runtime
 ├── Page Component
 ├── Product Component
 ├── Review Component
 ├── Recommendation Component
 └── Event Logic
```

而 Server Components：

```text
Server
 │
 ├── Product
 ├── Review
 └── Recommendation
          │
          ▼
       Render
          │
          ▼
       Browser

Browser
 │
 └── AddToCart
```

因此：

```text
Less Client JS
       ↓
Less Download
       ↓
Less Parse
       ↓
Less Execute
       ↓
Less Hydration
       ↓
Better Performance
```

注意：

> **Server Components 的目标不是“把所有东西放到 Server”，而是让不需要交互的 UI 不必成为 Client JavaScript。**

---

# 16. Hydration 也需要重新理解

传统 React SSR：

```text
Server
   │
   ▼
HTML
   │
   ▼
Browser
   │
   ▼
Download JS
   │
   ▼
Hydration
```

Server Components 体系：

```text
Server Component
       │
       ▼
RSC Payload
       │
       ▼
HTML
       │
       ▼
Browser
       │
       ├── Server-rendered UI
       │
       └── Client Components
                │
                ▼
             Hydration
```

因此：

> **不是整个页面都需要按照传统 SPA 的方式 Hydrate。**

只有需要 Client Runtime 的部分才需要对应的客户端 JavaScript。

---

# 17. 一个更合理的页面架构

例如企业后台：

```text
IncidentPage
│
├── IncidentHeader             Server
│
├── IncidentMetadata           Server
│
├── IncidentTimeline           Server
│
├── IncidentMetrics            Server
│
├── CommentList                Server
│
└── CommentEditor              Client
        │
        ├── useState
        ├── Form Event
        └── Browser Interaction
```

这是非常典型的：

> **Server-heavy + Client-islands**

架构。

而不是：

```text
IncidentPage
   │
   ▼
"use client"
   │
   ├── Everything
   ├── API
   ├── State
   ├── Rendering
   └── Interaction
```

---

# 18. Server Components 与 Data Fetching

Server Component 最大的优势之一就是可以直接获取数据。

例如：

```tsx
export default async function IncidentPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;

  const incident = await getIncident(id);

  return (
    <IncidentDetails incident={incident} />
  );
}
```

数据流：

```text
Request
   │
   ▼
Next.js Server
   │
   ▼
Server Component
   │
   ▼
Data Source
   │
   ├── Database
   ├── Redis
   └── Spring Boot
   │
   ▼
React Render
   │
   ▼
Browser
```

相比：

```text
Browser
   │
   ▼
useEffect
   │
   ▼
API
   │
   ▼
Spring Boot
   │
   ▼
Database
```

Server Component 可以减少中间的一部分客户端数据获取流程。

---

# 19. Server Component 与 Spring Boot

对于 Java Full-Stack 开发者，这里尤其值得注意。

例如：

```text
Browser
   │
   ▼
Next.js Server Component
   │
   ▼
Spring Boot
   │
   ▼
PostgreSQL
```

Next.js Server Component：

```tsx
export default async function UsersPage() {
  const response = await fetch(
    `${process.env.BACKEND_URL}/users`
  );

  const users = await response.json();

  return <Users users={users} />;
}
```

这样：

```text
BACKEND_URL
API Token
Internal Credentials
```

都可以留在 Server。

而不是：

```text
Browser
   │
   ▼
Public API
```

直接暴露内部服务。

这使 Next.js 很适合承担：

> **BFF（Backend for Frontend）**

角色。

---

# 20. Server Components 与 Security Boundary

Server/Client Boundary 同时也是一个重要的安全边界。

例如：

```tsx
const users = await db.user.findMany();
```

数据库连接：

```text
DATABASE_URL
```

应该只存在于 Server Runtime。

同样：

```text
API_SECRET
PRIVATE_KEY
INTERNAL_SERVICE_TOKEN
DATABASE_CREDENTIAL
```

都不应该进入 Client Bundle。

因此可以把：

```text
Server Component
```

理解成：

> **Application Server Boundary**

而：

```text
Client Component
```

则是：

> **Browser Runtime Boundary**

这对于设计企业级应用非常重要。

---

# 21. 但是 Server Component 不等于安全授权

这是另一个非常重要的误区。

有人可能认为：

> “因为代码运行在 Server，所以这个操作天然安全。”

这是错误的。

例如：

```tsx
export default async function AdminPage() {
  const users = await getUsers();

  return <Users users={users} />;
}
```

你仍然需要：

```text
Authentication
Authorization
Permission Check
```

正确模型应该是：

```text
Request
   │
   ▼
Authentication
   │
   ▼
Authorization
   │
   ▼
Server Component
   │
   ▼
Data Access
```

而不是：

```text
Server Component
      ↓
直接访问数据库
```

---

# 22. Server Component 与 Client Component 的数据传递

例如 Server：

```tsx
export default async function Page() {
  const user = await getUser();

  return (
    <UserMenu user={user} />
  );
}
```

Client：

```tsx
"use client";

export default function UserMenu({
  user,
}: {
  user: User;
}) {
  return (
    <button>
      {user.name}
    </button>
  );
}
```

这里存在一个非常重要的问题：

> **Server → Client 的数据必须能够被 React/Next.js 传递和序列化。**

因此不要简单地把复杂 Server-only 对象直接传给 Client Component。

应该尽量传递：

```text
string
number
boolean
plain object
array
```

以及符合 React/Next.js 支持规则的数据类型。

更重要的是：

> **不要把 Server-only Secret 通过 Props 传给 Client。**

例如：

```tsx
<ClientComponent
  apiKey={process.env.SECRET_KEY}
/>
```

这种设计是错误的。

---

# 23. Server Component 与 Client Component 的职责划分

可以建立一个简单判断模型：

```text
                    Component
                       │
             ┌─────────┴─────────┐
             │                   │
       Need Browser?         No Browser?
             │                   │
             ▼                   ▼
          Client               Server
             │                   │
      ┌──────┼──────┐       ┌────┼─────┐
      │      │      │       │    │     │
    State  Event   DOM     DB   API   Secret
```

如果组件需要：

```text
useState
useEffect
onClick
onChange
window
document
localStorage
WebSocket
```

优先考虑 Client。

如果组件主要做：

```text
Data Fetching
Database Access
Server API
Authorization
Static UI
Data Transformation
```

优先考虑 Server。

---

# 24. 一个高级实践：把 Client Boundary 放在叶子节点

例如：

```text
ProductPage
│
├── ProductHeader
├── ProductDescription
├── ProductReviews
├── ProductRecommendation
│
└── ProductActions
      │
      ├── QuantitySelector
      ├── AddToCart
      └── BuyNow
```

最合理的设计可能是：

```text
ProductPage                 Server
├── ProductHeader           Server
├── Description             Server
├── Reviews                 Server
├── Recommendation          Server
│
└── ProductActions          Client
      ├── QuantitySelector
      ├── AddToCart
      └── BuyNow
```

这就是：

> **Keep the Client Boundary as small as possible.**

---

# 25. 一个常见反模式：Client Component 包住整个应用

例如：

```tsx
"use client";

export default function App() {
  return (
    <Header>
      <Sidebar>
        <Dashboard>
          <Users />
        </Dashboard>
      </Sidebar>
    </Header>
  );
}
```

如果所有组件因此进入 Client Module Graph，那么：

```text
Client Bundle
       │
       ├── Header
       ├── Sidebar
       ├── Dashboard
       ├── Users
       └── Everything
```

这会失去 Server Components 的很多优势。

更合理：

```text
Server
 │
 ├── Header
 ├── Sidebar
 ├── Dashboard
 │
 └── Client
       └── InteractiveWidget
```

---

# 26. Context Provider 应该放在哪里？

React Context 通常需要 Client Component。

例如：

```tsx
"use client";

export function ThemeProvider({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <ThemeContext.Provider>
      {children}
    </ThemeContext.Provider>
  );
}
```

Root Layout：

```tsx
import { ThemeProvider } from "./ThemeProvider";

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html>
      <body>
        <ThemeProvider>
          {children}
        </ThemeProvider>
      </body>
    </html>
  );
}
```

这里非常重要：

> **Layout 可以是 Server Component，同时包含 Client Provider。**

因此：

```text
RootLayout                Server
   │
   └── ThemeProvider      Client
          │
          └── children    Server/Client
```

这体现了 Server/Client Boundary 的灵活性。

---

# 27. Suspense 与 Server Components

Server Components 和 Suspense 可以结合：

```tsx
import { Suspense } from "react";

export default function Dashboard() {
  return (
    <>
      <Header />

      <Suspense fallback={<ChartSkeleton />}>
        <SlowChart />
      </Suspense>
    </>
  );
}
```

架构：

```text
Dashboard
│
├── Header
│
└── Suspense
      │
      └── SlowChart
```

服务器可以先输出：

```text
Header
Skeleton
```

之后：

```text
SlowChart
```

准备好后再 Streaming 到浏览器。

这使：

```text
Server Components
+
Suspense
+
Streaming
```

形成一个完整的现代 Rendering Model。

---

# 28. Server Components 与性能

从性能角度：

```text
Traditional SPA
       │
       ▼
Large JS Bundle
       │
       ▼
Download
       │
       ▼
Parse
       │
       ▼
Execute
       │
       ▼
Hydration
```

Server Components：

```text
Server Component
       │
       ▼
Server Render
       │
       ▼
RSC Payload
       │
       ▼
Browser
       │
       └── Only required Client JS
```

因此它主要优化的是：

```text
JavaScript
Network
Hydration
Rendering
Data Fetching
```

但不能简单理解成：

> “Server Components 一定更快。”

如果 Server 端：

```text
Database
   ↓
Slow API
   ↓
Server Rendering
```

本身很慢，那么用户仍然会等待。

所以真正的性能优化应该综合：

```text
Server Performance
+
Data Fetching
+
Caching
+
Streaming
+
Client Bundle
```

---

# 29. Server Components 与缓存

Server Component 经常与 Next.js Data Cache 一起工作。

例如：

```tsx
const products = await fetch(
  "https://api.example.com/products",
  {
    next: {
      revalidate: 60,
    },
  }
);
```

可以形成：

```text
Request
   │
   ▼
Server Component
   │
   ▼
Data Fetch
   │
   ▼
Next.js Cache
   │
   ├── Cache Hit
   │      ↓
   │    Render
   │
   └── Cache Miss
          ↓
       Backend API
```

因此：

> Server Components 不是孤立特性，而是 Next.js Rendering + Data + Cache 体系的一部分。

---

# 30. Server Component 设计模式

我比较推荐企业项目采用：

```text
Page
 │
 ├── Data Fetching
 │
 ├── Data Transformation
 │
 └── UI Composition
          │
          ├── Server Components
          │
          └── Client Components
```

例如：

```tsx
export default async function DashboardPage() {
  const dashboard = await getDashboardData();

  return (
    <>
      <DashboardHeader
        data={dashboard.summary}
      />

      <IncidentList
        incidents={dashboard.incidents}
      />

      <InteractiveChart
        data={dashboard.chart}
      />
    </>
  );
}
```

其中：

```text
DashboardHeader     Server
IncidentList         Server
InteractiveChart     Client
```

---

# 31. 一个完整的 Server/Client Boundary 示例

考虑一个订单详情页：

```text
OrderPage
│
├── OrderHeader
│
├── CustomerInfo
│
├── OrderItems
│
├── PaymentInfo
│
├── ShippingTimeline
│
└── OrderActions
       │
       ├── CancelOrder
       ├── Refund
       └── UpdateAddress
```

推荐：

```text
OrderPage                  Server
│
├── OrderHeader             Server
├── CustomerInfo            Server
├── OrderItems              Server
├── PaymentInfo             Server
├── ShippingTimeline        Server
│
└── OrderActions            Client
      ├── CancelOrder       Client
      ├── Refund            Client
      └── UpdateAddress     Client
```

这样：

```text
Database
   │
   ▼
Server Component
   │
   ▼
HTML/RSC
   │
   ▼
Browser
   │
   └── OrderActions
          │
          ▼
      Interaction
```

这就是比较成熟的 App Router 设计。

---

# 32. 从 Java/Spring 开发者角度重新理解

如果你长期使用 Spring MVC，可以进行一个有趣的类比。

传统：

```text
Spring MVC
     │
     ├── Controller
     ├── Service
     ├── Repository
     └── Database
```

Next.js Server Component：

```text
Server Component
     │
     ├── Fetch Data
     ├── Transform Data
     └── Render UI
```

但是二者不是完全等价的。

更合理的企业架构：

```text
Next.js
│
├── Server Components
│       │
│       └── Presentation / BFF
│
└── Client Components
        │
        └── Browser Interaction


Spring Boot
│
├── Controller
├── Application Service
├── Domain Service
├── Repository
└── Database
```

也就是说：

> **不要因为 Next.js 支持 Server Components，就把整个 Spring Boot 业务层搬进 Next.js。**

Server Components 更适合承担：

```text
UI Composition
Data Fetching
BFF
Presentation Logic
```

复杂核心业务仍然应该保持清晰的 Domain Boundary。

---

# 33. Server Components 的架构原则

可以总结为 7 条。

## 1. Server First

默认 Server。

只有明确需要 Browser Runtime 时才进入 Client。

---

## 2. Client Boundary 最小化

把：

```text
"use client"
```

尽可能放在靠近交互源头的位置。

---

## 3. Data Close to Server

敏感数据和数据获取尽量留在 Server。

---

## 4. UI Close to Data

Server Component 可以直接获取它需要的数据，然后完成 UI Composition。

---

## 5. Avoid Client Waterfall

避免：

```text
Browser
 ↓
API 1
 ↓
API 2
 ↓
API 3
```

尽可能利用 Server 端并行数据获取。

---

## 6. Server ≠ Automatically Secure

必须明确：

```text
Authentication
Authorization
Validation
```

---

## 7. Server Components ≠ SSR

必须区分：

```text
Component Runtime
```

和：

```text
Rendering Strategy
```

---

# 34. 最终建立一个完整心智模型

如果要真正掌握 Next.js Server Components，可以把整个模型记成：

```text
                         Next.js
                            │
                     App Router
                            │
                            ▼
                     React Component Tree
                            │
                 ┌──────────┴──────────┐
                 │                     │
              Server                  Client
             Components             Components
                 │                     │
                 ▼                     ▼
          Server Runtime          Browser Runtime
                 │                     │
        ┌────────┼────────┐       ┌────┼─────┐
        │        │        │       │    │     │
       DB       API     Cache    DOM  State  Event
        │        │        │       │    │     │
        └────────┼────────┘       └────┼─────┘
                 │                     │
                 ▼                     ▼
              RSC Payload          Client JS
                 │                     │
                 └──────────┬──────────┘
                            ▼
                         Browser
                            │
                            ▼
                       Interactive UI
```

这张图实际上就是 Server Components 的核心。

---

# 35. 结语：重新理解 Server/Client Boundary

很多开发者学习 Next.js 时，第一反应是学习：

```text
page.tsx
layout.tsx
use client
fetch
Server Actions
```

但如果只记 API，很快就会陷入：

> “这个 API 怎么用？”

更重要的问题应该是：

> **这个 Component 应该在哪里运行？**

然后继续问：

```text
它需要 Browser Runtime 吗？
        │
        ├── Yes → Client Component
        │
        └── No
             │
             ▼
        Server Component
             │
             ├── Data Fetch
             ├── Database
             ├── API
             ├── Authorization
             └── Rendering
```

因此，Next.js Server Components 真正带来的改变并不是：

> “React 可以在服务器运行了。”

而是：

> **React Application 开始从一个以 Browser 为中心的 Component Model，演进成一个同时拥有 Server Runtime 和 Browser Runtime 的分布式 Component Model。**

而 **Server/Client Boundary** 就是连接这两个世界的核心架构边界。

对于企业级 Next.js 应用，真正优秀的架构通常不是：

```text
Everything Server
```

也不是：

```text
Everything Client
```

而是：

```text
                 Application
                     │
          ┌──────────┴──────────┐
          │                     │
       Server                 Client
          │                     │
   Data / Security        Interaction
   Rendering / Cache      State / Event
          │                     │
          └──────────┬──────────┘
                     ▼
                  UI
```

**Server 负责“数据和生成”，Client 负责“交互和状态”。**

这就是理解 Next.js App Router 和 Server Components 最重要的思维转变。


