---
title: Next.js App Router 深度解析：从文件路由到现代 React 应用架构
# tags:
#   - nodejs
date: '2026-8-05'
summary: Next.js App Router 不只是一次路由 API 的升级，而是 Next.js 对 React 应用运行模型的一次重新设计。
---


> **Next.js App Router 不只是一次路由 API 的升级，而是 Next.js 对 React 应用运行模型的一次重新设计。**
>
> 真正理解 App Router，需要同时理解 **Server Components、Client Components、Layout、Streaming、Data Fetching、Caching、Server Actions 和 Route Handlers**。

---

## 1. 为什么 Next.js 要重新设计 Router？

在传统 React SPA 中，典型架构是：

```text
Browser
   │
   ▼
React Application
   │
   ├── React Router
   │
   ├── API Client
   │
   └── State Management
             │
             ▼
        Backend API
```

页面渲染主要发生在浏览器。

例如：

```text
GET /dashboard
        │
        ▼
Browser
        │
        ▼
Download JS
        │
        ▼
React Bootstrap
        │
        ▼
Call API
        │
        ▼
Render UI
```

这种模式非常适合高度交互的 Web Application，但也存在一些问题：

* 首屏 JavaScript 较多
* SEO 需要额外处理
* 数据获取容易形成 waterfall
* 页面之间共享布局比较麻烦
* Loading / Error 状态需要大量手工管理
* Server 与 Client 的边界不够明确

Next.js App Router 的设计目标，就是让 **Server Rendering、Client Rendering 和 Routing 成为统一的应用模型**。

---

# 2. App Router 的核心思想

App Router 最重要的思想不是：

> “目录名字变成了 `app`。”

而是：

> **路由结构、React Component Tree、Server/Client Boundary、数据获取和渲染生命周期被统一起来。**

可以理解为：

```text
                  URL
                   │
                   ▼
              App Router
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
      Layout      Page      Loading
        │          │          │
        └──────────┼──────────┘
                   │
                   ▼
             React Tree
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
 Server Components     Client Components
        │                     │
        ▼                     ▼
   Server Runtime        Browser Runtime
```

所以 App Router 实际上连接了三个层面：

```text
Routing
   +
Rendering
   +
Component Architecture
```

这也是它和传统 React Router 最大的区别之一。

---

# 3. App Router 的目录结构

一个典型项目：

```text
app/
├── layout.tsx
├── page.tsx
├── loading.tsx
├── error.tsx
├── not-found.tsx
│
├── dashboard/
│   ├── layout.tsx
│   ├── page.tsx
│   │
│   ├── users/
│   │   └── page.tsx
│   │
│   └── orders/
│       └── page.tsx
│
├── products/
│   └── [id]/
│       └── page.tsx
│
└── api/
    └── users/
        └── route.ts
```

最重要的几个特殊文件：

| 文件              | 作用             |
| --------------- | -------------- |
| `page.tsx`      | 页面             |
| `layout.tsx`    | 共享布局           |
| `loading.tsx`   | Loading UI     |
| `error.tsx`     | Error Boundary |
| `not-found.tsx` | 404 UI         |
| `route.ts`      | Route Handler  |

---

# 4. `page.tsx`：Route Segment 的入口

最简单的：

```text
app/
└── page.tsx
```

对应：

```text
/
```

例如：

```tsx
export default function HomePage() {
  return <h1>Hello Next.js</h1>;
}
```

创建：

```text
app/dashboard/page.tsx
```

对应：

```text
/dashboard
```

创建：

```text
app/users/page.tsx
```

对应：

```text
/users
```

因此：

```text
app/
├── page.tsx
├── users/
│   └── page.tsx
└── dashboard/
    └── page.tsx
```

对应：

```text
/
├── /users
└── /dashboard
```

这就是 File-system Based Routing。

但 App Router 的真正价值并不在这里。

---

# 5. `layout.tsx`：App Router 最重要的设计之一

传统 SPA 经常需要自己实现：

```text
Application
   │
   ├── Header
   ├── Sidebar
   ├── Content
   └── Footer
```

App Router 把这种共享结构直接融入路由树。

例如：

```text
app/
├── layout.tsx
└── dashboard/
    ├── layout.tsx
    └── page.tsx
```

可以理解为：

```text
RootLayout
    │
    └── DashboardLayout
            │
            └── DashboardPage
```

最终：

```text
┌─────────────────────────────┐
│ Root Layout                 │
│                             │
│ ┌─────────────────────────┐ │
│ │ Dashboard Layout        │ │
│ │                         │ │
│ │ ┌─────────────────────┐ │ │
│ │ │ Dashboard Page      │ │ │
│ │ └─────────────────────┘ │ │
│ └─────────────────────────┘ │
└─────────────────────────────┘
```

---

# 6. Layout 为什么重要？

假设后台系统：

```text
/dashboard
/dashboard/users
/dashboard/orders
/dashboard/settings
```

这些页面都需要：

```text
Header
Sidebar
User Menu
```

传统做法可能是：

```tsx
function DashboardPage() {
  return (
    <>
      <Header />
      <Sidebar />
      <Content />
    </>
  );
}
```

每个页面重复。

App Router：

```tsx
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div>
      <Header />
      <Sidebar />

      <main>
        {children}
      </main>
    </div>
  );
}
```

然后：

```text
dashboard/
├── layout.tsx
├── page.tsx
├── users/
│   └── page.tsx
└── orders/
    └── page.tsx
```

所有 Dashboard 页面自动共享这个 Layout。

---

# 7. Nested Layout：真正的路由树

App Router 可以形成多层 Layout：

```text
RootLayout
    │
    ├── PublicLayout
    │       ├── Home
    │       └── About
    │
    └── DashboardLayout
            │
            ├── Dashboard
            ├── Users
            └── Orders
```

例如：

```text
app/
├── layout.tsx
│
├── page.tsx
│
└── dashboard/
    ├── layout.tsx
    ├── page.tsx
    │
    ├── users/
    │   └── page.tsx
    │
    └── orders/
        └── page.tsx
```

这实际上形成了一个：

```text
Component Tree
```

而不是简单的：

```text
URL → Component
```

这就是理解 App Router 的一个关键转折点。

---

# 8. Server Component 是 App Router 的核心

App Router 默认使用：

> **React Server Components**

例如：

```tsx
export default async function UsersPage() {
  const users = await getUsers();

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>
          {user.name}
        </li>
      ))}
    </ul>
  );
}
```

这个 Component 默认运行在 Server。

因此可以：

```text
Server Component
       │
       ├── Database
       ├── Internal API
       ├── File System
       ├── Environment Variables
       └── Backend Services
```

而不需要把这些逻辑发送到浏览器。

---

# 9. Client Component

如果组件需要：

* `useState`
* `useEffect`
* Browser API
* Event Handler
* DOM Interaction

就需要：

```tsx
"use client";
```

例如：

```tsx
"use client";

import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

此时：

```text
Server Component
       │
       │
       ▼
Client Component
       │
       ▼
Browser
```

---

# 10. 为什么 Server / Client Boundary 非常重要？

这是 App Router 与传统 SPA 思维最大的区别之一。

考虑：

```text
Page
 │
 ├── ProductInfo
 │
 ├── ProductPrice
 │
 └── AddToCartButton
```

其实不需要整个页面都成为 Client Component。

更合理：

```text
Page                         Server
 │
 ├── ProductInfo             Server
 │
 ├── ProductPrice            Server
 │
 └── AddToCartButton         Client
```

也就是说：

> **只把真正需要交互的部分放到 Client。**

这能够减少浏览器需要执行的 JavaScript。

---

# 11. 一个典型的 Server/Client 架构

例如电商页面：

```text
ProductPage
│
├── ProductDetails
│      └── Server Component
│
├── ProductReviews
│      └── Server Component
│
├── Recommendation
│      └── Server Component
│
└── AddToCart
       └── Client Component
```

这比：

```tsx
"use client";

export default function ProductPage() {
   // everything client-side
}
```

更加符合 App Router 的设计理念。

---

# 12. Dynamic Route

App Router 支持动态路由。

例如：

```text
app/products/[id]/page.tsx
```

对应：

```text
/products/100
/products/200
/products/300
```

代码：

```tsx
export default async function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;

  return <h1>Product: {id}</h1>;
}
```

这里：

```text
[id]
```

就是 Dynamic Segment。

---

# 13. Catch-all Route

例如：

```text
app/docs/[...slug]/page.tsx
```

可以匹配：

```text
/docs/react
/docs/react/hooks
/docs/react/hooks/use-effect
```

参数类似：

```text
slug = ["react", "hooks", "use-effect"]
```

如果使用：

```text
[[...slug]]
```

则还可以匹配：

```text
/docs
```

这对于：

* CMS
* Documentation
* Blog
* Knowledge Base

非常有用。

---

# 14. Route Groups

Route Group 是 App Router 一个非常实用的设计。

例如：

```text
app/
├── (marketing)/
│   ├── about/
│   └── pricing/
│
└── (dashboard)/
    ├── dashboard/
    └── settings/
```

括号中的：

```text
(marketing)
(dashboard)
```

不会出现在 URL 中。

因此：

```text
app/(marketing)/about/page.tsx
```

仍然是：

```text
/about
```

Route Groups 的意义在于：

> **URL 结构和代码组织结构可以解耦。**

---

# 15. Route Groups 的另一个价值：Layout 隔离

例如：

```text
app/
├── (public)/
│   ├── layout.tsx
│   ├── login/
│   └── register/
│
└── (dashboard)/
    ├── layout.tsx
    ├── dashboard/
    └── settings/
```

可以得到：

```text
Public Layout
   │
   ├── Login
   └── Register


Dashboard Layout
   │
   ├── Dashboard
   └── Settings
```

非常适合企业级应用。

---

# 16. Parallel Routes

Parallel Routes 是 App Router 更高级的能力。

例如一个后台页面：

```text
┌─────────────────────────────────────┐
│ Header                              │
├───────────┬─────────────────────────┤
│ Sidebar   │ Main                    │
│           │                         │
│           │ Analytics               │
│           │                         │
│           ├─────────────────────────┤
│           │ Notifications           │
└───────────┴─────────────────────────┘
```

不同区域可以拥有独立的路由状态。

例如：

```text
app/dashboard/
├── @analytics/
├── @notifications/
└── layout.tsx
```

Layout：

```tsx
export default function DashboardLayout({
  analytics,
  notifications,
}: {
  analytics: React.ReactNode;
  notifications: React.ReactNode;
}) {
  return (
    <>
      {analytics}
      {notifications}
    </>
  );
}
```

这样一个 URL 下可以管理多个独立 UI Slot。

---

# 17. Intercepting Routes

Intercepting Routes 是另一个非常有价值的功能。

典型场景：

> 点击商品列表中的商品，希望弹出 Modal，而不是离开当前页面。

例如：

```text
/products
```

点击：

```text
/products/123
```

可以变成：

```text
┌───────────────────────────────┐
│ Product List                  │
│                               │
│ ┌───────────────────────────┐ │
│ │ Product Detail Modal      │ │
│ │                           │ │
│ │ Product #123              │ │
│ └───────────────────────────┘ │
└───────────────────────────────┘
```

但直接访问：

```text
/products/123
```

则可以显示完整 Product Page。

这就是 Intercepting Routes 很典型的使用场景。

---

# 18. `loading.tsx`

App Router 对 Loading UI 提供了原生支持。

例如：

```text
dashboard/
├── page.tsx
└── loading.tsx
```

```tsx
export default function Loading() {
  return <div>Loading dashboard...</div>;
}
```

当页面加载时：

```text
Request
   │
   ▼
Loading UI
   │
   ▼
Server Rendering
   │
   ▼
Page
```

这背后实际上和：

> **React Suspense + Streaming**

密切相关。

---

# 19. Streaming

Streaming 是现代 Next.js 非常重要的能力。

传统 SSR：

```text
Request
   │
   ▼
Wait for everything
   │
   ▼
Generate complete HTML
   │
   ▼
Browser
```

Streaming：

```text
Request
   │
   ▼
Render Shell
   │
   ├── Header
   ├── Navigation
   └── Skeleton
           │
           ▼
       Stream Result
           │
           ▼
      Browser Update
```

例如：

```tsx
<Suspense fallback={<Loading />}>
  <SlowComponent />
</Suspense>
```

这样慢组件不会阻塞整个页面。

---

# 20. `error.tsx`

App Router 还提供 Error Boundary。

例如：

```text
dashboard/
├── page.tsx
└── error.tsx
```

通常：

```tsx
"use client";

export default function Error({
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>

      <button onClick={() => reset()}>
        Try again
      </button>
    </div>
  );
}
```

因此页面可以形成：

```text
                    Page
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Loading      Error      NotFound
```

这使得页面状态管理更加结构化。

---

# 21. `not-found.tsx`

当资源不存在时：

```tsx
import { notFound } from "next/navigation";

export default async function ProductPage() {
  const product = await getProduct();

  if (!product) {
    notFound();
  }

  return <Product product={product} />;
}
```

Next.js 会进入：

```text
not-found.tsx
```

这比传统 React：

```tsx
if (!data) {
    return <NotFound />;
}
```

更加系统化。

---

# 22. Route Handlers

App Router 还可以定义 Backend Endpoint：

```text
app/api/users/route.ts
```

例如：

```tsx
import { NextResponse } from "next/server";

export async function GET() {
  const users = await getUsers();

  return NextResponse.json(users);
}
```

对应：

```text
GET /api/users
```

也可以：

```tsx
export async function POST(request: Request) {
  const body = await request.json();

  // create user

  return NextResponse.json({
    success: true,
  });
}
```

所以 Next.js 可以同时承担：

```text
Frontend
   +
Backend for Frontend
```

---

# 23. App Router + Server Actions

如果是数据修改场景，可以进一步使用 Server Actions。

例如：

```tsx
"use server";

export async function createUser(formData: FormData) {
  const name = formData.get("name");

  // database operation
}
```

然后：

```tsx
<form action={createUser}>
  <input name="name" />
  <button type="submit">
    Create
  </button>
</form>
```

架构变成：

```text
Browser
   │
   ▼
React Form
   │
   ▼
Server Action
   │
   ▼
Database
```

这意味着某些场景下不再需要：

```text
Client
   ↓
fetch()
   ↓
/api/users
   ↓
Server
   ↓
Database
```

而可以：

```text
Client
   ↓
Server Action
   ↓
Database
```

但这并不意味着 REST API 已经没有价值。

如果你的后端是：

```text
Spring Boot
```

或者存在：

```text
Mobile App
Third-party Client
External Integration
Microservices
```

REST API 依然非常重要。

---

# 24. App Router 与 Spring Boot 如何结合？

对于你的 Java Full-Stack 背景，我反而推荐深入研究这种架构：

```text
                     Browser
                        │
                        ▼
                 Next.js App Router
                        │
          ┌─────────────┴─────────────┐
          │                           │
   Server Components            Client Components
          │                           │
          ▼                           ▼
   Backend API                 User Interaction
          │
          ▼
      Spring Boot
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
 PostgreSQL Redis Kafka
```

Next.js 的定位可以是：

```text
Presentation Layer
+
BFF
```

而 Spring Boot：

```text
Domain Layer
+
Business Services
```

这是一种非常值得写进技术博客的企业架构。

---

# 25. App Router 中的数据获取

例如：

```tsx
export default async function UsersPage() {
  const response = await fetch(
    "https://api.example.com/users"
  );

  const users = await response.json();

  return (
    <UserList users={users} />
  );
}
```

这里最重要的问题不是：

> “怎么 fetch？”

而是：

> **这个请求什么时候执行？在哪里执行？是否缓存？缓存多久？数据发生变化后如何重新验证？**

因此 Next.js 数据获取实际上和：

```text
Rendering
+
Caching
+
Routing
```

是紧密联系的。

这也是 App Router 和传统 React Router 最大的认知差异之一。

---

# 26. App Router 的缓存体系

理解 App Router，必须理解它的缓存。

可以简化为：

```text
                  Next.js
                     │
          ┌──────────┴──────────┐
          │                     │
     Data Cache             Full Route Cache
          │                     │
          │                     │
          └──────────┬──────────┘
                     │
                     ▼
                Router Cache
                     │
                     ▼
                  Browser
```

因此：

```text
Request
   ↓
Data Fetch
   ↓
Cache
   ↓
Rendering
   ↓
Route Cache
   ↓
Browser
```

这也是为什么 Next.js 性能优化不能简单理解为：

> “减少 React Component。”

而应该从：

```text
Network
Server
Data
Cache
Rendering
JavaScript
Browser
```

整个链路分析。

---

# 27. 一个企业级 App Router 示例

假设我们做一个：

> **Enterprise Incident Management System**

目录可以设计为：

```text
app/
│
├── layout.tsx
│
├── login/
│   └── page.tsx
│
├── (dashboard)/
│   ├── layout.tsx
│   │
│   ├── dashboard/
│   │   └── page.tsx
│   │
│   ├── incidents/
│   │   ├── page.tsx
│   │   └── [id]/
│   │       └── page.tsx
│   │
│   ├── users/
│   │   └── page.tsx
│   │
│   └── settings/
│       └── page.tsx
│
└── api/
    └── incidents/
        └── route.ts
```

架构：

```text
                       Next.js
                          │
                 ┌────────┴────────┐
                 │                 │
            Server Side        Client Side
                 │                 │
                 ▼                 ▼
          Spring Boot API      React UI
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
      Redis   PostgreSQL  Kafka
```

这已经不是一个简单的 Next.js Demo。

而是一个完整的 Full-Stack Application。

---

# 28. App Router 的设计原则

如果要真正掌握 App Router，我建议记住下面几个原则。

### 原则一：默认 Server

不要一上来：

```tsx
"use client";
```

先问：

> 这个组件是否真的需要 Browser Runtime？

---

### 原则二：把 Client Component 控制在最小范围

例如：

```text
Page
│
├── Server
│
├── Server
│
└── Client
      │
      ├── Button
      ├── Modal
      └── Form
```

而不是：

```text
Page
└── Client
      ├── Server-like logic
      ├── API
      ├── State
      └── Everything
```

---

### 原则三：Layout 用于稳定的 UI Boundary

例如：

```text
Root Layout
   │
   └── Dashboard Layout
          │
          ├── Users
          ├── Orders
          └── Settings
```

---

### 原则四：Loading/Error 是路由的一部分

不要所有页面都手工：

```tsx
if (loading) ...
if (error) ...
```

合理使用：

```text
loading.tsx
error.tsx
not-found.tsx
```

---

### 原则五：数据获取应该尽量靠近 Server

```text
Page
 │
 └── Server Component
          │
          ▼
       Fetch Data
          │
          ▼
       Render UI
```

而不是：

```text
Browser
   │
   ▼
useEffect
   │
   ▼
fetch()
   │
   ▼
API
```

当然，实时交互等场景依然需要 Client Fetching。

---

# 29. App Router 最容易犯的错误

## 错误一：所有组件都加 `"use client"`

这是最常见的问题。

结果：

```text
Large Client Bundle
       ↓
More JavaScript
       ↓
More Hydration
       ↓
Poor Performance
```

---

## 错误二：把 Server Component 当传统 React Component

例如在 Server Component 中滥用：

```tsx
useEffect()
useState()
```

Server Component 的运行环境本身就不同。

---

## 错误三：为了获取数据全部使用 Client `useEffect`

例如：

```tsx
"use client";

useEffect(() => {
  fetch("/api/users")
}, []);
```

很多场景下其实可以直接：

```tsx
export default async function Page() {
  const users = await getUsers();

  return <Users users={users} />;
}
```

---

## 错误四：不理解 Cache

很多 Next.js 问题最后都会变成：

> “为什么我更新数据库之后页面还是旧数据？”

答案经常和：

```text
Data Cache
Route Cache
Router Cache
Revalidation
```

有关。

---

# 30. App Router 真正改变的是什么？

如果只看表面：

```text
pages/
```

变成：

```text
app/
```

似乎变化不大。

但从架构层看：

```text
传统 Next.js
        │
        ▼
Pages Router
        │
        ▼
Page Rendering
        │
        ▼
React Application
```

逐渐演变成：

```text
                App Router
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
    Routing      Rendering     Data
       │            │            │
       │       Server/Client     │
       │            │            │
       └────────────┼────────────┘
                    ▼
                 Cache
                    │
                    ▼
                Streaming
                    │
                    ▼
              Browser Runtime
```

因此：

> **App Router 本质上是 Next.js 对 React Application Runtime 的重新建模。**

---

# 31. 从架构师角度理解 App Router

如果把整个 Next.js Application 看成一个系统：

```text
                    User
                     │
                     ▼
                  Browser
                     │
                     ▼
              ┌──────────────┐
              │ App Router   │
              └──────┬───────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
      Layout        Page       Route Handler
        │            │            │
        ▼            ▼            ▼
 Server Components  RSC       Backend API
        │
        ▼
    Data Fetch
        │
        ▼
      Cache
        │
        ▼
    Rendering
        │
        ▼
    Streaming
        │
        ▼
     Browser
```

你会发现 App Router 实际上已经不只是：

> Routing Framework

而更接近：

> **React Application Runtime + Rendering Architecture + BFF Framework**

这才是理解 Next.js App Router 的核心。

---

# 32. 总结

如果只记住几个关键点，可以记住这张图：

```text
                         Next.js App Router
                                │
       ┌────────────────────────┼────────────────────────┐
       │                        │                        │
       ▼                        ▼                        ▼
   File Routing             Component Tree          Rendering
       │                        │                        │
       │                  ┌─────┴─────┐          ┌───────┴───────┐
       │                  │           │          │               │
       ▼                Server      Client      Static        Dynamic
   page.tsx             Component   Component   Rendering     Rendering
   layout.tsx                │           │
   loading.tsx               │           │
   error.tsx                 ▼           ▼
   not-found.tsx           Server      Browser
                            │           │
                            ▼           ▼
                         Database     Events
                         API          State
                         Cache        DOM
                             
                                │
                                ▼
                           Streaming
                                │
                                ▼
                             Browser
```

