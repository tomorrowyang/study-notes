# Next.js 初级 Q&A

> 目标：熟练使用 App Router 的文件约定、路由、Server / Client Components、数据获取、Server Actions、Route Handlers 和内置优化组件。示例默认基于 Next.js 15 / 16 + TypeScript，与版本相关的行为会单独标注。

---

## 一、基础与路由

### Q1. Next.js 是什么？解决了什么问题？

- **难度**: 初级
- **标签**: 基础

**答案**

Next.js 是基于 React 的**全栈框架**，在 React 之上提供：

- **路由**：基于文件系统的路由（App Router）
- **多种渲染方式**：静态生成、服务端渲染、流式渲染、部分预渲染（PPR），可以按路由甚至按组件选择
- **React Server Components**：组件可以在服务端运行，直接访问数据库，不发送代码到浏览器
- **数据变更**：Server Actions
- **API**：Route Handlers
- **内置优化**：图片、字体、脚本、代码分割、预取
- **构建工具**：Turbopack（16 起默认）

**解决的问题**：纯客户端渲染的 React 应用存在首屏白屏、SEO 差、需要自己搭建路由 / SSR / 构建的问题。

**口述模板（30–60 秒）**

1. 一句话结论：Next.js 是 React 的全栈框架，提供路由、多种渲染方式、服务端组件和内置优化。
2. 关键机制：基于文件系统的路由，Server Components 在服务端运行，可以按路由选择渲染策略。
3. 对比或注意点：解决了纯 CSR 应用的首屏、SEO 和工程搭建问题。
4. 可追问时补充：16 起 Turbopack 成为默认构建工具。

**常见追问**

- Next.js 和 React 是什么关系？→ React 是 UI 库，React 官方推荐通过框架（Next.js 等）使用 Server Components 等全栈能力。

---

### Q2. App Router 和 Pages Router 有什么区别？

- **难度**: 初级
- **标签**: 路由

**答案**

| 对比项 | Pages Router（`pages/`） | App Router（`app/`） |
|--------|--------------------------|----------------------|
| 组件默认类型 | 客户端组件（全部 hydrate） | **Server Components** |
| 数据获取 | `getServerSideProps`、`getStaticProps`、`getStaticPaths` | 组件中直接 `await`、`fetch` |
| 布局 | `_app`、`_document`，嵌套布局需要自己实现 | `layout.tsx` 嵌套布局，导航时保持状态 |
| 加载 / 错误 | 自己实现 | `loading.tsx`、`error.tsx` 文件约定 |
| 流式渲染 | 不支持 | 支持（Suspense） |
| 数据变更 | API Routes | Server Actions |
| 状态 | 仍然支持，维护中 | **推荐** |

两种 Router 可以在同一个项目中共存，便于渐进迁移。

**口述模板（30–60 秒）**

1. 一句话结论：App Router 基于 Server Components，支持嵌套布局和流式渲染，是推荐的方式。
2. 关键机制：数据在组件中直接获取，不再需要 getServerSideProps。
3. 对比或注意点：Pages Router 中所有组件都要 hydrate，App Router 中只有客户端组件需要。
4. 可追问时补充：两者可以共存，支持渐进迁移。

**常见追问**

- 为什么 App Router 能减少客户端 JS？→ Server Components 的代码不发送到浏览器，只有标记了 'use client' 的组件才进入客户端包。

---

### Q3. App Router 有哪些特殊文件？

- **难度**: 初级
- **标签**: 路由, 文件约定

**答案**

| 文件 | 作用 |
|------|------|
| `page.tsx` | 页面，让该路由段可以被访问 |
| `layout.tsx` | 布局，包裹子路由；**导航时保持状态，不重新渲染** |
| `template.tsx` | 类似布局，但每次导航都**重新挂载** |
| `loading.tsx` | 加载 UI，自动包裹在 Suspense 中 |
| `error.tsx` | 错误边界（必须是客户端组件） |
| `global-error.tsx` | 根布局的错误边界 |
| `not-found.tsx` | `notFound()` 或未匹配时显示 |
| `route.ts` | Route Handler（API 端点） |
| `default.tsx` | 并行路由的后备内容 |
| `proxy.ts`（项目根目录） | 请求拦截（16 之前为 `middleware.ts`） |

**嵌套结构**：

```text
<Layout>
  <Template>
    <ErrorBoundary fallback={<Error />}>
      <Suspense fallback={<Loading />}>
        <ErrorBoundary fallback={<NotFound />}>
          <Page />
```

**口述模板（30–60 秒）**

1. 一句话结论：App Router 用特殊文件约定定义页面、布局、加载、错误和 API。
2. 关键机制：layout 导航时保持状态，loading 自动包裹在 Suspense 中，error 是错误边界。
3. 对比或注意点：template 每次导航都重新挂载，layout 不会。
4. 可追问时补充：error.tsx 必须是客户端组件，因为错误边界需要在客户端运行。

**常见追问**

- layout 中的错误会被同级的 error.tsx 捕获吗？→ 不会，error.tsx 包裹在 layout 内部；layout 的错误由上一级的 error.tsx 捕获，根布局的错误由 global-error.tsx 捕获。

---

### Q4. 动态路由怎么定义？

- **难度**: 初级
- **标签**: 路由

**答案**

| 文件夹 | 匹配 | params |
|--------|------|--------|
| `app/blog/[slug]/page.tsx` | `/blog/a` | `{ slug: 'a' }` |
| `app/shop/[...slug]/page.tsx` | `/shop/a/b` | `{ slug: ['a', 'b'] }`（不匹配 `/shop`） |
| `app/shop/[[...slug]]/page.tsx` | `/shop`、`/shop/a/b` | 可选的全部捕获 |

```tsx
// Next.js 15+：params 是 Promise
export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  const post = await getPost(slug)
  return <article>{post.title}</article>
}

// 构建时预渲染的参数
export async function generateStaticParams() {
  const posts = await getPosts()
  return posts.map((post) => ({ slug: post.slug }))
}
```

**`dynamicParams`**：不在 `generateStaticParams` 中的参数，默认在请求时渲染；设为 `false` 返回 404。

**口述模板（30–60 秒）**

1. 一句话结论：用方括号定义动态段，三个点表示全部捕获，双方括号表示可选。
2. 关键机制：params 通过页面的 props 传入，15 起是 Promise，需要 await。
3. 对比或注意点：generateStaticParams 在构建时预渲染指定的参数。
4. 可追问时补充：dynamicParams 控制未预渲染的参数是否在请求时渲染。

**常见追问**

- 为什么 15 把 `params` 改为 Promise？→ 让框架可以在不需要这些值之前就开始渲染（为 PPR 和流式做准备），只有真正读取时才等待。

---

### Q5. 路由组、并行路由、拦截路由是什么？

- **难度**: 初级
- **标签**: 路由

**答案**

**路由组 `(folder)`**：不影响 URL，用于组织代码或为不同部分使用不同的布局。

```text
app/(marketing)/about/page.tsx   → /about
app/(shop)/cart/page.tsx         → /cart
```

**并行路由 `@folder`**：在同一个布局中同时渲染多个页面（插槽），每个插槽独立加载和出错。

```text
app/dashboard/layout.tsx
app/dashboard/@analytics/page.tsx
app/dashboard/@team/page.tsx
```

```tsx
export default function Layout({ children, analytics, team }: LayoutProps) {
  return <>{children}{analytics}{team}</>
}
```

**拦截路由 `(.)folder`**：在当前布局中显示另一个路由的内容（例如点击图片时以弹窗形式打开 `/photo/1`，刷新页面时显示完整的图片页）。

- `(.)` 同级、`(..)` 上一级、`(...)` 根目录

**典型组合**：并行路由 + 拦截路由实现可分享 URL 的弹窗。

**口述模板（30–60 秒）**

1. 一句话结论：路由组用于组织代码，并行路由同时渲染多个页面，拦截路由在当前上下文中显示其他路由。
2. 关键机制：路由组的括号不进入 URL；并行路由的插槽作为 props 传给布局。
3. 对比或注意点：并行路由需要 default.tsx 作为未匹配时的后备。
4. 可追问时补充：两者组合可以实现刷新后仍然有效的弹窗路由。

**常见追问**

- 为什么并行路由需要 `default.tsx`？→ 硬导航（刷新）时，Next.js 无法恢复其他插槽之前的状态，需要 default 作为后备；16 起缺少 default 会构建报错。

---

### Q6. 怎么在 Next.js 中导航？

- **难度**: 初级
- **标签**: 路由

**答案**

```tsx
import Link from 'next/link'

<Link href="/dashboard" prefetch>仪表盘</Link>
<Link href={{ pathname: '/blog/[slug]', query: { slug: 'hello' } }}>文章</Link>
```

```tsx
'use client'
import { useRouter, usePathname, useSearchParams } from 'next/navigation'

const router = useRouter()
router.push('/dashboard')
router.replace('/login')
router.refresh()   // 重新请求当前路由的服务端组件，不丢失客户端状态
router.back()
```

```tsx
// 服务端组件、Server Action 中
import { redirect, notFound } from 'next/navigation'
redirect('/login')
notFound()
```

**Link 的特点**：

- 客户端导航，不刷新整个页面，布局保持状态
- **预取**：Link 进入视口时预取目标路由（生产环境）

**口述模板（30–60 秒）**

1. 一句话结论：使用 Link 组件或 useRouter 进行客户端导航，服务端用 redirect 和 notFound。
2. 关键机制：Link 进入视口时预取目标路由，导航时布局保持状态。
3. 对比或注意点：useRouter 来自 next/navigation，不是 Pages Router 的 next/router。
4. 可追问时补充：router.refresh 重新获取服务端组件的数据，客户端状态保留。

**常见追问**

- `redirect` 为什么不需要 return？→ 它内部抛出一个特殊的错误，由框架捕获并处理重定向，所以不能把它放在 try / catch 中（或者要重新抛出）。

---

## 二、Server Components 与 Client Components

### Q7. 什么是 Server Components？

- **难度**: 初级
- **标签**: RSC

**答案**

**Server Components（服务端组件）**是只在服务端运行的 React 组件，App Router 中**默认**所有组件都是服务端组件。

```tsx
// app/posts/page.tsx
import { db } from '@/lib/db'

export default async function PostsPage() {
  const posts = await db.post.findMany()  // 直接访问数据库
  return (
    <ul>
      {posts.map((post) => <li key={post.id}>{post.title}</li>)}
    </ul>
  )
}
```

**优点**：

- 直接访问数据库、文件系统、密钥，不暴露给客户端
- **代码不发送到浏览器**，减少客户端 JS（例如 markdown 解析库只在服务端使用）
- 数据获取靠近数据源，减少客户端请求瀑布

**限制**：不能使用状态和副作用（`useState`、`useEffect`）、事件处理函数、浏览器 API、Context。

**口述模板（30–60 秒）**

1. 一句话结论：Server Components 只在服务端运行，App Router 中默认是服务端组件。
2. 关键机制：可以直接访问数据库和密钥，代码不发送到浏览器。
3. 对比或注意点：不能使用状态、副作用、事件处理和浏览器 API。
4. 可追问时补充：它们的输出是序列化的 RSC Payload，而不是 HTML。

**常见追问**

- 服务端组件会重新渲染吗？→ 不会在客户端重新渲染；只有导航、`router.refresh()` 或 Server Action 触发时，服务端重新渲染并发送新的 Payload。

---

### Q8. Client Components 是什么？`'use client'` 做了什么？

- **难度**: 初级
- **标签**: RSC

**答案**

```tsx
'use client'

import { useState } from 'react'

export function LikeButton({ initialCount }: { initialCount: number }) {
  const [count, setCount] = useState(initialCount)
  return <button onClick={() => setCount(count + 1)}>{count}</button>
}
```

**`'use client'` 的含义**：

- 标记一个**模块边界**：这个文件以及它导入的所有模块，都成为客户端包的一部分
- 客户端组件**仍然会在服务端预渲染 HTML**（首次加载时），然后在客户端 hydrate
- 只需要在边界的入口文件声明，它导入的组件自动成为客户端组件

**什么时候需要**：使用 state、effect、事件处理、浏览器 API、依赖这些的第三方库、Context Provider。

**最佳实践**：把 `'use client'` 放在**尽量小的叶子组件**上，页面主体保持为服务端组件。

**口述模板（30–60 秒）**

1. 一句话结论：'use client' 标记客户端模块边界，该模块及其导入都进入客户端包。
2. 关键机制：客户端组件首次加载时仍然在服务端渲染 HTML，然后 hydrate。
3. 对比或注意点：不是「只在客户端运行」，而是「也会在客户端运行」。
4. 可追问时补充：应该放在尽量小的交互组件上，减少客户端 JS。

**常见追问**

- `'use client'` 组件中能访问 `window` 吗？→ 只能在 useEffect 或事件处理中访问；渲染阶段访问会在服务端预渲染时报错。

---

### Q9. 服务端组件和客户端组件怎么组合？

- **难度**: 初级
- **标签**: RSC, 组合

**答案**

**规则**：

| 组合方式 | 是否可以 |
|----------|----------|
| 服务端组件导入客户端组件 | ✅ |
| 客户端组件**导入**服务端组件 | ❌（被导入的组件会变成客户端组件） |
| 客户端组件通过 `children` / props **接收**服务端组件 | ✅ |

```tsx
// app/page.tsx（服务端组件）
import { Modal } from './modal'      // 客户端组件
import { Cart } from './cart'        // 服务端组件

export default function Page() {
  return (
    <Modal>
      <Cart />  {/* 服务端组件作为 children 传入，在服务端渲染 */}
    </Modal>
  )
}
```

**从服务端组件向客户端组件传 props**：props 必须可以**序列化**（字符串、数字、普通对象、数组、Date、Map、Set、Promise、JSX 等），**不能传函数**（Server Action 除外）、类实例。

**Context**：Provider 是客户端组件，在根布局中包裹 `children`，服务端组件作为 children 不受影响。

**口述模板（30–60 秒）**

1. 一句话结论：服务端组件可以导入客户端组件，客户端组件只能通过 children 或 props 接收服务端组件。
2. 关键机制：客户端组件导入的模块都进入客户端包。
3. 对比或注意点：传给客户端组件的 props 必须可以序列化，不能传普通函数。
4. 可追问时补充：Context Provider 包裹 children 的方式不会把子组件变成客户端组件。

**常见追问**

- 为什么可以通过 children 传入？→ 服务端组件在服务端已经渲染为 RSC Payload，客户端组件收到的是渲染结果，而不是组件代码。

---

### Q10. 怎么防止服务端代码泄露到客户端？

- **难度**: 初级
- **标签**: 安全, RSC

**答案**

**环境变量**：

- 只有以 `NEXT_PUBLIC_` 开头的变量会被打包进客户端代码
- 其他变量只在服务端可用（在客户端组件中读取得到 `undefined`）

**`server-only` 包**：

```ts
// lib/db.ts
import 'server-only'

export async function getUser(id: string) {
  // 使用数据库密钥
}
```

如果客户端组件导入了这个模块，**构建时报错**。对应的有 `client-only`。

**数据层（DAL）**：把数据访问集中在一个服务端模块中，在其中做鉴权，只返回需要的字段（DTO），避免把完整的数据库记录（包括密码哈希）传给客户端组件。

**Taint API（实验）**：`experimental_taintObjectReference` 标记对象不允许传给客户端。

**口述模板（30–60 秒）**

1. 一句话结论：只有 NEXT_PUBLIC_ 前缀的环境变量会进入客户端，服务端模块用 server-only 防止被客户端导入。
2. 关键机制：server-only 让客户端导入时构建报错。
3. 对比或注意点：传给客户端组件的 props 会序列化发送到浏览器，只传需要的字段。
4. 可追问时补充：推荐建立数据访问层，统一鉴权和字段过滤。

**常见追问**

- 服务端组件中的数据都安全吗？→ 渲染出的内容和传给客户端组件的 props 都会发送到浏览器，只是代码本身不发送。

---

## 三、数据获取

### Q11. App Router 中怎么获取数据？

- **难度**: 初级
- **标签**: 数据获取

**答案**

**服务端组件中直接获取（推荐）**：

```tsx
export default async function Page() {
  const response = await fetch('https://api.example.com/products')
  const products: Product[] = await response.json()
  // 或者直接查询数据库 / 调用 ORM
  return <ProductList products={products} />
}
```

**并行获取**，避免瀑布：

```tsx
const [user, orders] = await Promise.all([getUser(id), getOrders(id)])
```

**客户端组件中获取**：使用 SWR、TanStack Query，或者 React 的 `use()` 读取服务端传来的 Promise：

```tsx
// 服务端组件：不 await，把 Promise 传给客户端组件
const commentsPromise = getComments(id)
return (
  <Suspense fallback={<Spinner />}>
    <Comments commentsPromise={commentsPromise} />
  </Suspense>
)

// 客户端组件
'use client'
function Comments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const comments = use(commentsPromise)
  return <ul>{comments.map((c) => <li key={c.id}>{c.text}</li>)}</ul>
}
```

**口述模板（30–60 秒）**

1. 一句话结论：App Router 推荐在服务端组件中直接 await 获取数据。
2. 关键机制：可以直接用 fetch 或查询数据库；多个请求用 Promise.all 并行。
3. 对比或注意点：注意避免串行的请求瀑布。
4. 可追问时补充：可以把 Promise 传给客户端组件，用 use() 配合 Suspense 流式显示。

**常见追问**

- 多个组件请求同一个接口会重复请求吗？→ 同一次渲染中，相同 URL 和参数的 fetch GET 请求会被去重（Request Memoization）；非 fetch 的数据获取用 React 的 `cache()` 包装去重。

---

### Q12. Next.js 中 `fetch` 的缓存行为是怎样的？

- **难度**: 初级
- **标签**: 数据获取, 缓存

**答案**

**版本差异**（这是常见的坑）：

| 版本 | 默认行为 |
|------|----------|
| Next.js 14 | `fetch` 默认缓存（`force-cache`） |
| Next.js 15 | `fetch` **默认不缓存**（`no-store`）；GET Route Handler 默认也不缓存 |
| Next.js 16 + Cache Components | 数据默认在**请求时**获取；需要缓存时使用 `'use cache'` 显式声明 |

**旧模型中的配置**：

```ts
fetch(url, { cache: 'force-cache' })            // 缓存
fetch(url, { cache: 'no-store' })               // 不缓存
fetch(url, { next: { revalidate: 60 } })        // 缓存 60 秒（ISR）
fetch(url, { next: { tags: ['products'] } })    // 打标签，可以按标签失效
```

**路由段配置**：

```ts
export const revalidate = 60
export const dynamic = 'force-dynamic'
```

（开启 Cache Components 后，这些路由段配置被 `'use cache'` 和 `cacheLife` 取代。）

**口述模板（30–60 秒）**

1. 一句话结论：fetch 的缓存默认值在各版本中不同，回答前要先说明版本。
2. 关键机制：14 默认缓存，15 默认不缓存，16 开启 Cache Components 后需要 'use cache' 显式缓存。
3. 对比或注意点：可以用 revalidate 设置缓存时间，用 tags 按需失效。
4. 可追问时补充：缓存从隐式变为显式，是 Next.js 近几个版本最大的变化。

**常见追问**

- 为什么 15 改为默认不缓存？→ 14 的隐式缓存让很多开发者困惑（数据不更新），改为默认不缓存更符合直觉，需要缓存时显式声明。

---

### Q13. 什么是 Server Actions？

- **难度**: 初级
- **标签**: Server Actions

**答案**

Server Actions（Server Functions 的一种）是运行在服务端、可以在客户端调用的**异步函数**，主要用于**数据变更**。

```tsx
// app/actions.ts
'use server'

import { revalidatePath } from 'next/cache'
import { z } from 'zod'

const schema = z.object({ title: z.string().min(1).max(100) })

export async function createPost(formData: FormData) {
  const session = await auth()
  if (!session) throw new Error('未登录')

  const parsed = schema.safeParse({ title: formData.get('title') })
  if (!parsed.success) return { error: '标题不合法' }

  await db.post.create({ data: { title: parsed.data.title, authorId: session.userId } })
  revalidatePath('/posts')
}
```

```tsx
// 在表单中使用
<form action={createPost}>
  <input name="title" />
  <button type="submit">发布</button>
</form>
```

**特点**：

- 表单在 **JS 加载前也能提交**（渐进增强）
- 一次往返中同时完成数据变更和返回更新后的 UI
- 可以在客户端组件的事件处理中直接调用

**安全**：每个 Server Action 都是一个**公开的 HTTP 端点**，必须做鉴权和输入校验。

**口述模板（30–60 秒）**

1. 一句话结论：Server Actions 是在服务端运行、可以从客户端调用的异步函数，用于数据变更。
2. 关键机制：用 'use server' 标记，可以作为表单的 action，支持渐进增强。
3. 对比或注意点：每个 Action 都是公开端点，必须做鉴权和输入校验。
4. 可追问时补充：变更后用 revalidatePath 或 updateTag 刷新相关数据。

**常见追问**

- 能用 Server Action 获取数据吗？→ 不推荐；Server Action 是按顺序执行的（同一客户端的多个调用排队），使用 POST，不适合并行获取和缓存。数据获取放在服务端组件中。

---

### Q14. Route Handlers 是什么？和 Server Actions 怎么选？

- **难度**: 初级
- **标签**: Route Handlers

**答案**

```ts
// app/api/products/route.ts
import { NextRequest } from 'next/server'

export async function GET(request: NextRequest) {
  const category = request.nextUrl.searchParams.get('category')
  const products = await getProducts(category)
  return Response.json(products)
}

export async function POST(request: Request) {
  const body = await request.json()
  // 校验和鉴权...
  return Response.json(await createProduct(body), { status: 201 })
}
```

基于 Web 标准的 `Request` / `Response`。

| 场景 | 选择 |
|------|------|
| 表单提交、与 UI 紧密相关的变更 | Server Action |
| 供第三方、移动端、其他服务调用的 API | Route Handler |
| Webhook（支付回调等） | Route Handler |
| 需要自定义 HTTP 方法、响应头、流式响应、文件下载 | Route Handler |
| 服务端组件中获取数据 | 都不需要，直接调用数据函数 |

**注意**：服务端组件中**不要** fetch 自己的 Route Handler（多一次网络往返），直接调用底层函数。

**口述模板（30–60 秒）**

1. 一句话结论：Route Handler 是基于 Web Request / Response 的 API 端点。
2. 关键机制：在 route.ts 中导出 GET、POST 等函数。
3. 对比或注意点：UI 相关的变更用 Server Action，对外的 API 和 Webhook 用 Route Handler。
4. 可追问时补充：服务端组件中不要请求自己的 Route Handler，直接调用数据函数。

**常见追问**

- 同一目录下能同时有 `page.tsx` 和 `route.ts` 吗？→ 不能，同一个路由段只能是页面或 API 之一。

---

### Q15. 表单的提交状态和校验错误怎么处理？

- **难度**: 初级
- **标签**: Server Actions, 表单

**答案**

```tsx
// actions.ts
'use server'

type State = { error?: string; success?: boolean }

export async function subscribe(prevState: State, formData: FormData): Promise<State> {
  const email = formData.get('email')
  if (typeof email !== 'string' || !email.includes('@')) {
    return { error: '邮箱格式不正确' }
  }
  await saveSubscriber(email)
  return { success: true }
}
```

```tsx
// subscribe-form.tsx
'use client'

import { useActionState } from 'react'
import { useFormStatus } from 'react-dom'
import { subscribe } from './actions'

function SubmitButton() {
  // 必须在 form 内部的组件中调用
  const { pending } = useFormStatus()
  return <button disabled={pending}>{pending ? '提交中...' : '订阅'}</button>
}

export function SubscribeForm() {
  const [state, formAction, isPending] = useActionState(subscribe, {})
  return (
    <form action={formAction}>
      <input name="email" type="email" />
      <SubmitButton />
      {state.error && <p role="alert">{state.error}</p>}
      {state.success && <p>订阅成功</p>}
    </form>
  )
}
```

- `useActionState`（React 19）：管理 Action 的返回状态和 pending
- `useFormStatus`：读取**父级 form** 的提交状态

**口述模板（30–60 秒）**

1. 一句话结论：用 useActionState 管理 Action 的返回状态，用 useFormStatus 获取提交中状态。
2. 关键机制：Action 第一个参数是上一次的状态，返回新状态显示错误或结果。
3. 对比或注意点：useFormStatus 必须在 form 内部的子组件中调用。
4. 可追问时补充：服务端校验是必需的，客户端校验只是改善体验。

**常见追问**

- 校验错误用 throw 还是 return？→ 预期内的错误（校验失败）用 return 返回状态；意外错误才 throw，由 error.tsx 处理。

---

## 四、渲染与缓存基础

### Q16. Next.js 有哪些渲染方式？

- **难度**: 初级
- **标签**: 渲染

**答案**

| 方式 | 时机 | 场景 |
|------|------|------|
| **静态渲染**（SSG） | 构建时，或重新验证后 | 博客、文档、营销页 |
| **ISR**（增量静态再生） | 静态页面按时间或按需在后台重新生成 | 商品页、新闻 |
| **动态渲染**（SSR） | 每次请求时 | 个性化页面、依赖 Cookie 的页面 |
| **流式渲染** | 按 Suspense 边界分块发送 | 有慢数据的页面 |
| **PPR**（部分预渲染） | 静态外壳 + 动态部分流式填充 | 大部分静态、少量个性化内容的页面 |
| **CSR** | 客户端组件中获取数据 | 高度交互、无 SEO 需求 |

**旧模型中让路由变为动态的情况**：使用 `cookies()`、`headers()`、`searchParams`、`connection()`，或者不缓存的数据请求。

**Next.js 16 + Cache Components**：页面默认在请求时渲染，用 `'use cache'` 标记可以缓存的部分，结合 PPR 生成静态外壳。

**口述模板（30–60 秒）**

1. 一句话结论：Next.js 支持静态、ISR、动态、流式、PPR 和 CSR 多种渲染方式。
2. 关键机制：旧模型中，使用 cookies、headers 等请求相关 API 会让路由变为动态。
3. 对比或注意点：PPR 让同一个页面同时有静态部分和动态部分。
4. 可追问时补充：16 的 Cache Components 让缓存变为显式声明。

**常见追问**

- 构建时怎么知道哪些路由是静态的？→ `next build` 的输出中会标注每个路由的类型（静态、动态、PPR）。

---

### Q17. ISR 是什么？怎么使用？

- **难度**: 初级
- **标签**: 渲染, ISR

**答案**

**ISR（Incremental Static Regeneration）**：页面静态生成并缓存，在过期后，**下一个请求仍然返回旧页面**，同时在后台重新生成，生成完成后替换缓存（stale-while-revalidate）。

```tsx
// 基于时间（旧模型）
export const revalidate = 3600

// 按需失效：数据更新时调用
'use server'
import { revalidatePath, revalidateTag } from 'next/cache'

export async function updateProduct(id: string, data: ProductInput) {
  await db.product.update({ where: { id }, data })
  revalidatePath(`/products/${id}`)
  // 或者按标签失效所有使用该数据的页面
  revalidateTag('products', 'max')  // 16 起需要第二个参数
}
```

**优势**：静态页面的性能 + 数据可以更新，不需要重新构建整个站点。

**注意**：自托管多实例时，各实例的缓存需要共享（自定义 `cacheHandler`），否则失效只影响一个实例。

**口述模板（30–60 秒）**

1. 一句话结论：ISR 让静态页面可以在后台按时间或按需重新生成。
2. 关键机制：过期后先返回旧页面，后台重新生成，完成后替换。
3. 对比或注意点：可以用 revalidatePath 或 revalidateTag 按需失效。
4. 可追问时补充：自托管多实例需要共享缓存。

**常见追问**

- `revalidateTag` 在 16 中为什么需要第二个参数？→ 第二个参数指定 cacheLife 配置（例如 `'max'`），实现 stale-while-revalidate 行为；需要立即看到新数据（读到自己的写入）时，在 Server Action 中使用 `updateTag`。

---

### Q18. `loading.tsx` 和 Suspense 有什么作用？

- **难度**: 初级
- **标签**: 流式渲染

**答案**

**`loading.tsx`**：自动把页面包裹在 `<Suspense fallback={<Loading />}>` 中：

- 导航时**立即显示**加载 UI（布局保持可交互）
- 页面数据准备好后，流式替换为真实内容

**手动 Suspense**：更细粒度地控制，慢的部分不阻塞快的部分：

```tsx
export default function ProductPage({ params }: Props) {
  return (
    <>
      <ProductInfo params={params} />          {/* 快，先显示 */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews params={params} />            {/* 慢，流式填充 */}
      </Suspense>
      <Suspense fallback={<RecommendSkeleton />}>
        <Recommendations params={params} />
      </Suspense>
    </>
  )
}
```

**流式渲染的效果**：TTFB 和 FCP 提前；不同部分独立加载；客户端组件可以**选择性 hydration**（先 hydrate 用户正在交互的部分）。

**口述模板（30–60 秒）**

1. 一句话结论：loading.tsx 和 Suspense 实现流式渲染，慢的部分不阻塞整个页面。
2. 关键机制：服务端先发送外壳和加载状态，数据准备好后流式发送内容替换。
3. 对比或注意点：Suspense 边界的粒度决定了用户体验，不宜太粗也不宜太细。
4. 可追问时补充：流式渲染配合选择性 hydration，交互的部分优先激活。

**常见追问**

- 流式渲染对 SEO 有影响吗？→ 没有，流式发送的仍然是 HTML，搜索引擎能获取完整内容；状态码在流开始后无法修改，所以 notFound 需要在流开始前调用。

---

### Q19. `error.tsx` 怎么使用？

- **难度**: 初级
- **标签**: 错误处理

**答案**

```tsx
'use client'  // 错误边界必须是客户端组件

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    reportError(error)
  }, [error])

  return (
    <div>
      <h2>出错了</h2>
      <button onClick={() => reset()}>重试</button>
    </div>
  )
}
```

- 捕获**同级 page 及其子组件**的错误，不影响布局和其他部分
- `reset()` 尝试重新渲染该路由段
- 生产环境中，服务端组件的错误信息会被**隐藏**（只传递 `digest`，可以在服务端日志中查找对应的错误），防止泄露敏感信息
- `global-error.tsx`：捕获根布局的错误，必须包含自己的 `<html>` 和 `<body>`

**口述模板（30–60 秒）**

1. 一句话结论：error.tsx 是路由段级别的错误边界，必须是客户端组件。
2. 关键机制：捕获同级页面及子组件的错误，reset 重新渲染。
3. 对比或注意点：生产环境中服务端错误只传递 digest，不暴露详情。
4. 可追问时补充：根布局的错误需要 global-error.tsx。

**常见追问**

- Server Action 中的错误会被 error.tsx 捕获吗？→ 抛出的错误会被最近的错误边界捕获；预期内的错误应该通过返回值处理。

---

### Q20. 怎么设置页面的 Metadata（SEO）？

- **难度**: 初级
- **标签**: SEO, Metadata

**答案**

**静态 metadata**：

```tsx
export const metadata: Metadata = {
  title: { default: '我的商店', template: '%s | 我的商店' },
  description: '...',
  openGraph: { images: ['/og.png'] },
}
```

**动态 metadata**：

```tsx
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { id } = await params
  const product = await getProduct(id)  // 与页面中的相同请求会被去重
  return {
    title: product.name,
    description: product.summary,
    alternates: { canonical: `/products/${id}` },
  }
}
```

**其他文件约定**：`sitemap.ts`、`robots.ts`、`opengraph-image.tsx`（动态生成 OG 图片）、`icon.tsx`、`manifest.ts`。

**口述模板（30–60 秒）**

1. 一句话结论：通过导出 metadata 对象或 generateMetadata 函数设置页面元信息。
2. 关键机制：layout 和 page 中的 metadata 会合并，title 支持模板。
3. 对比或注意点：generateMetadata 中的请求与页面中的相同请求会被去重。
4. 可追问时补充：sitemap、robots、OG 图片都有文件约定。

**常见追问**

- metadata 能在客户端组件中导出吗？→ 不能，只能在服务端组件中导出。

---

## 五、内置优化

### Q21. `next/image` 做了哪些优化？

- **难度**: 初级
- **标签**: 性能, 图片

**答案**

```tsx
import Image from 'next/image'
import hero from './hero.jpg'

<Image src={hero} alt="首页大图" priority placeholder="blur" sizes="100vw" />
<Image src="https://cdn.example.com/a.jpg" alt="" width={400} height={300} />
```

**优化**：

- **自动格式转换**：根据浏览器支持返回 WebP / AVIF
- **按尺寸生成**：根据 `sizes` 和设备生成合适宽度的图片（srcset）
- **防止布局偏移**：要求 width / height（或 fill），预留空间，避免 CLS
- **懒加载**：默认 `loading="lazy"`
- **`priority`**（16 中也可以用 `preload`）：首屏 LCP 图片预加载，不懒加载
- **模糊占位**：`placeholder="blur"`

**远程图片**：需要在 `next.config` 中配置 `images.remotePatterns`（16 起 `images.domains` 已废弃），防止被滥用为任意图片代理。

**自托管注意**：图片优化在服务端执行，消耗 CPU；可以配置自定义 loader 使用图片 CDN。

**口述模板（30–60 秒）**

1. 一句话结论：next/image 自动转换格式、按尺寸生成、预留空间、默认懒加载。
2. 关键机制：根据 sizes 生成 srcset，浏览器选择合适的尺寸。
3. 对比或注意点：首屏 LCP 图片要设置 priority，不能懒加载。
4. 可追问时补充：远程图片需要配置 remotePatterns 白名单。

**常见追问**

- 为什么要配置远程图片白名单？→ 否则任何人都可以利用你的服务器优化任意图片，消耗资源，甚至用于攻击。

---

### Q22. `next/font` 解决了什么问题？

- **难度**: 初级
- **标签**: 性能, 字体

**答案**

```tsx
import { Inter } from 'next/font/google'
import localFont from 'next/font/local'

const inter = Inter({ subsets: ['latin'], display: 'swap', variable: '--font-inter' })
const brand = localFont({ src: './brand.woff2', variable: '--font-brand' })

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="zh-CN" className={`${inter.variable} ${brand.variable}`}>
      <body>{children}</body>
    </html>
  )
}
```

**优化**：

- **构建时下载** Google Fonts 并自托管，运行时不请求 Google（隐私、减少连接）
- 自动**预加载**字体
- 自动生成**后备字体的度量调整**（`size-adjust` 等），减少字体切换时的布局偏移（CLS）
- 子集化

**口述模板（30–60 秒）**

1. 一句话结论：next/font 在构建时自托管字体，自动预加载并减少布局偏移。
2. 关键机制：生成调整过度量的后备字体，字体切换时布局变化很小。
3. 对比或注意点：运行时不请求第三方字体服务。
4. 可追问时补充：通过 CSS 变量与 Tailwind 等结合使用。

**常见追问**

- 中文字体怎么处理？→ 中文字体体积大，next/font 的 Google Fonts 子集化对中文帮助有限；通常使用系统字体，或者按需子集化后作为本地字体。

---

### Q23. `next/script` 怎么加载第三方脚本？

- **难度**: 初级
- **标签**: 性能, 第三方脚本

**答案**

```tsx
import Script from 'next/script'

<Script src="https://example.com/analytics.js" strategy="afterInteractive" />
<Script src="https://example.com/chat.js" strategy="lazyOnload" />
<Script id="init" strategy="afterInteractive">{`window.dataLayer = []`}</Script>
```

| 策略 | 时机 | 场景 |
|------|------|------|
| `beforeInteractive` | 页面 hydration 之前（放在根布局中） | 机器人检测、同意管理 |
| `afterInteractive`（默认） | hydration 之后 | 统计、标签管理 |
| `lazyOnload` | 浏览器空闲时 | 客服、社交插件 |
| `worker`（实验） | 在 Web Worker 中（Partytown） | 重型第三方脚本 |

脚本只加载一次，即使在多个页面中使用。

**口述模板（30–60 秒）**

1. 一句话结论：next/script 通过 strategy 控制第三方脚本的加载时机。
2. 关键机制：默认在 hydration 之后加载，不重要的脚本在空闲时加载。
3. 对比或注意点：beforeInteractive 只用于必须最早执行的脚本。
4. 可追问时补充：第三方脚本是 INP 和 TBT 的主要来源之一。

**常见追问**

- `@next/third-parties` 是什么？→ 官方为 Google Analytics、Google Tag Manager、YouTube 嵌入等常用第三方提供的优化组件。

---

### Q24. Link 的预取机制是怎样的？

- **难度**: 初级
- **标签**: 性能, 导航

**答案**

- 生产环境中，`<Link>` **进入视口**时自动预取目标路由
- **静态路由**：预取完整的 RSC Payload
- **动态路由**（默认 `prefetch` 为自动）：只预取到最近的 `loading.tsx` 边界（共享布局 + 加载状态），导航时立即显示加载 UI，再获取动态内容
- `prefetch={true}`：完整预取动态路由
- `prefetch={false}`：不预取（例如页面中有大量链接时）
- 预取的数据存放在客户端路由缓存中

**16.x 的变化**：Cache Components 模型下的 **Partial Prefetching**：预取路由中缓存的部分（静态外壳），动态部分在导航时获取；16.4 起被视为 Cache Components 模型的一部分（具体行为以官方文档为准）。

**口述模板（30–60 秒）**

1. 一句话结论：Link 进入视口时自动预取目标路由，导航时可以立即显示。
2. 关键机制：静态路由完整预取，动态路由预取到 loading 边界。
3. 对比或注意点：链接很多的页面可以关闭预取，避免过多请求。
4. 可追问时补充：16 的 Partial Prefetching 预取缓存的部分。

**常见追问**

- 开发环境为什么感觉没有预取？→ 预取只在生产环境中生效，开发环境是按需编译的。

---

### Q25. 什么是 proxy（原 middleware）？

- **难度**: 初级
- **标签**: proxy, middleware

**答案**

`proxy.ts`（Next.js 16 起；之前为 `middleware.ts`）在**请求到达路由之前**执行，可以修改请求和响应：

```ts
// proxy.ts（项目根目录或 src 下）
import { NextResponse, type NextRequest } from 'next/server'

export function proxy(request: NextRequest) {
  const token = request.cookies.get('session')?.value
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  const response = NextResponse.next()
  response.headers.set('x-request-id', crypto.randomUUID())
  return response
}

export const config = {
  matcher: ['/dashboard/:path*', '/((?!_next/static|_next/image|favicon.ico).*)'],
}
```

**常见用途**：重定向、重写（A/B 测试、多租户）、国际化语言检测、乐观的登录检查、设置请求头。

**更名原因**：`middleware` 容易与 Express 等框架的中间件混淆；新名称强调它是位于应用前面的**网络边界**。16 中 proxy 运行在 **Node.js 运行时**。

**注意**：不要把 proxy 作为**唯一的**鉴权手段（参见 CVE-2025-29927），数据访问层中还要再次校验。

**口述模板（30–60 秒）**

1. 一句话结论：proxy 在请求到达路由之前执行，用于重定向、重写和请求头处理。
2. 关键机制：通过 matcher 配置匹配的路径，返回 NextResponse 控制请求。
3. 对比或注意点：16 将 middleware 更名为 proxy，运行在 Node.js 运行时。
4. 可追问时补充：proxy 只做乐观的检查，真正的鉴权在数据访问层。

**常见追问**

- proxy 中能查询数据库吗？→ 技术上可以，但它对每个匹配的请求都执行，应该保持轻量；复杂逻辑放在页面或数据层。

---

### Q26. Next.js 中怎么使用环境变量？

- **难度**: 初级
- **标签**: 配置

**答案**

| 文件 | 加载时机 |
|------|----------|
| `.env` | 所有环境 |
| `.env.local` | 所有环境，覆盖 `.env`（不提交到仓库） |
| `.env.development` / `.env.production` | 对应环境 |
| `.env.test` | 测试环境 |

- 默认**只在服务端可用**
- `NEXT_PUBLIC_` 前缀的变量在**构建时内联**到客户端代码中

**常见坑**：

- `NEXT_PUBLIC_` 变量在构建时就固定了，同一个镜像部署到不同环境时**无法通过运行时环境变量修改** → 需要运行时配置时，在服务端组件中读取后传给客户端，或者提供配置接口
- 不要把密钥放在 `NEXT_PUBLIC_` 变量中

**口述模板（30–60 秒）**

1. 一句话结论：环境变量默认只在服务端可用，NEXT_PUBLIC_ 前缀的会内联到客户端代码中。
2. 关键机制：内联发生在构建时。
3. 对比或注意点：同一镜像多环境部署时，NEXT_PUBLIC_ 变量无法在运行时修改。
4. 可追问时补充：密钥绝不能使用 NEXT_PUBLIC_ 前缀。

**常见追问**

- 怎么实现「构建一次，部署到多个环境」？→ 客户端需要的配置在服务端组件中从运行时环境变量读取，通过 props 或 Context 传给客户端组件。

---

### Q27. Next.js 中怎么管理样式？

- **难度**: 初级
- **标签**: 样式

**答案**

| 方案 | 说明 |
|------|------|
| **Tailwind CSS** | 官方推荐，`create-next-app` 默认集成，零运行时 |
| CSS Modules | `*.module.css`，类名局部化，零运行时 |
| 全局 CSS | 在根布局中导入 |
| Sass | 安装 sass 即可 |
| CSS-in-JS | 运行时方案（styled-components、Emotion）**只能在客户端组件中使用**，需要配置样式注册表；推荐零运行时的方案（vanilla-extract、Panda CSS） |

**为什么运行时 CSS-in-JS 与 Server Components 不兼容**：运行时方案依赖 Context 和在渲染时注入样式，而服务端组件不支持 Context，也不会在客户端渲染。

**口述模板（30–60 秒）**

1. 一句话结论：推荐 Tailwind 或 CSS Modules 等零运行时方案。
2. 关键机制：零运行时方案在构建时生成 CSS，与服务端组件兼容。
3. 对比或注意点：运行时 CSS-in-JS 只能在客户端组件中使用。
4. 可追问时补充：需要 CSS-in-JS 的写法时，选择零运行时的方案。

**常见追问**

- 16 中 CSS Modules 有什么变化？→ 16.4 起生产环境的类名更短，并支持导出名压缩，减小体积（以官方说明为准）。

---

### Q28. 静态资源和 `public` 目录怎么使用？

- **难度**: 初级
- **标签**: 资源

**答案**

- `public/` 中的文件通过根路径访问：`public/logo.png` → `/logo.png`
- `public` 中的文件**不经过构建处理**，没有内容哈希，缓存策略需要自己配置
- 图片推荐通过 `import` 导入（获得尺寸信息和哈希文件名）或 `next/image`
- 构建产物在 `/_next/static/` 下，带内容哈希，设置为长期缓存（`immutable`）

**口述模板（30–60 秒）**

1. 一句话结论：public 目录中的文件按原路径提供，不经过构建处理。
2. 关键机制：/_next/static 下的构建产物带哈希，可以长期缓存。
3. 对比或注意点：public 中的文件没有哈希，修改后可能被缓存。
4. 可追问时补充：图片优先通过 import 或 next/image 使用。

**常见追问**

- 部署到 CDN 怎么配置？→ 设置 `assetPrefix` 指向 CDN，`/_next/static` 上传到 CDN。

---

### Q29. `next.config` 中有哪些常用配置？

- **难度**: 初级
- **标签**: 配置

**答案**

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  cacheComponents: true,              // 16：启用 Cache Components
  reactCompiler: true,                // 16：启用 React Compiler
  output: 'standalone',               // 自托管：输出最小化的独立服务
  images: {
    remotePatterns: [{ protocol: 'https', hostname: 'cdn.example.com' }],
  },
  async redirects() {
    return [{ source: '/old', destination: '/new', permanent: true }]
  },
  async rewrites() {
    return [{ source: '/api/:path*', destination: 'https://backend.example.com/:path*' }]
  },
  async headers() {
    return [{ source: '/(.*)', headers: [{ key: 'X-Frame-Options', value: 'DENY' }] }]
  },
  experimental: {
    optimizePackageImports: ['lodash-es', 'lucide-react'],
  },
}

export default nextConfig
```

**口述模板（30–60 秒）**

1. 一句话结论：next.config 用于配置渲染特性、输出方式、图片、重定向、重写和响应头。
2. 关键机制：16 中通过 cacheComponents 和 reactCompiler 开启新特性。
3. 对比或注意点：自托管使用 standalone 输出。
4. 可追问时补充：optimizePackageImports 优化大型库的按需导入。

**常见追问**

- `rewrites` 和 `redirects` 的区别？→ redirect 让浏览器跳转到新 URL（地址栏变化）；rewrite 在服务端把请求映射到其他地址（地址栏不变），可以用作反向代理。

---

### Q30. 怎么部署 Next.js 应用？

- **难度**: 初级
- **标签**: 部署

**答案**

| 方式 | 说明 |
|------|------|
| **Vercel** | 官方平台，零配置，所有特性完整支持（ISR、PPR、图片优化、边缘） |
| **Node.js 服务器** | `next build` + `next start`，支持全部特性 |
| **Docker** | `output: 'standalone'`，只包含运行需要的文件，镜像小 |
| **静态导出** | `output: 'export'`，生成纯静态文件；不支持服务端特性（Server Actions、动态渲染、ISR、默认的图片优化） |
| 其他平台 | Netlify、Cloudflare（OpenNext 适配器）等 |

**自托管需要注意**：

- 多实例时共享缓存（`cacheHandler` 配置到 Redis 等）
- 图片优化消耗 CPU，可以用外部 CDN 的 loader
- 部署新版本时，旧客户端可能请求已不存在的资源（版本偏差），需要保留旧资源或配置 `deploymentId`
- 设置 `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`，保证多实例和多次部署之间 Server Action 的加密一致

**口述模板（30–60 秒）**

1. 一句话结论：可以部署到 Vercel、Node 服务器、Docker，或者导出为静态站点。
2. 关键机制：standalone 输出适合容器化部署。
3. 对比或注意点：静态导出不支持服务端特性。
4. 可追问时补充：自托管多实例需要共享缓存和统一的 Server Action 加密密钥。

**常见追问**

- 什么是版本偏差（version skew）？→ 用户还在使用旧版本的页面，新版本部署后，旧页面请求的 JS 块或 Server Action ID 在新服务器上不存在，导致报错。
