# Next.js 手写题

> 目标：能在白板或在线编辑器中写出 App Router 中的常见模式，并讲清每一步的原因，重点是安全、缓存和用户体验。

## 使用方式

- **鉴权与安全**：H1 数据访问层鉴权、H2 安全的 Server Action、H6 Webhook 签名验证、H11 限流
- **数据与缓存**：H3 Cache Components 缓存与失效、H4 流式页面、H7 无限滚动分页
- **交互**：H5 乐观更新列表、H9 表单与校验错误
- **路由与工程**：H8 国际化 proxy、H10 动态 OG 图片、H12 搜索页（URL 状态）

先口述思路，再写代码，最后主动说易错点。代码基于 Next.js 16（以官方为准）。

---

### H1. 数据访问层（DAL）鉴权

- **难度**: 中级
- **标签**: 鉴权, 安全
- **面试官看什么**: 鉴权放在哪一层、是否只返回需要的字段、是否使用 server-only

**题干**

实现一个数据访问层：`verifySession()` 读取并验证 session Cookie，同一次渲染中只验证一次；`getMyProjects()` 只返回当前用户的项目，并且只返回需要的字段。

**思路口述**

鉴权必须在离数据最近的地方执行，proxy 和布局的检查都可能被绕过或不执行。用 `server-only` 防止这个模块被客户端导入；用 React 的 `cache` 在一次渲染中去重；查询时用 select 只取需要的字段，避免把敏感字段传给客户端组件。

**参考实现**

```ts
// lib/dal.ts
import 'server-only'
import { cache } from 'react'
import { cookies } from 'next/headers'
import { redirect } from 'next/navigation'
import { jwtVerify } from 'jose'
import { db } from './db'

const secret = new TextEncoder().encode(process.env.SESSION_SECRET)

/** 会话中可以信任的最小信息 */
export interface Session {
  userId: string
  role: 'user' | 'admin'
}

/**
 * 验证当前请求的会话；未登录时重定向到登录页。
 * 用 cache 包裹：同一次渲染中多个组件调用只验证一次。
 */
export const verifySession = cache(async (): Promise<Session> => {
  const token = (await cookies()).get('session')?.value
  if (!token) redirect('/login')

  try {
    const { payload } = await jwtVerify(token, secret, { algorithms: ['HS256'] })
    return { userId: String(payload.sub), role: payload.role === 'admin' ? 'admin' : 'user' }
  } catch {
    // 签名错误或过期，统一按未登录处理，不暴露具体原因
    redirect('/login')
  }
})

/** 项目列表的 DTO：只包含页面需要展示的字段 */
export interface ProjectDTO {
  id: string
  name: string
  updatedAt: Date
}

/** 获取当前用户的项目列表 */
export async function getMyProjects(): Promise<ProjectDTO[]> {
  const session = await verifySession()
  return db.project.findMany({
    where: { ownerId: session.userId },
    select: { id: true, name: true, updatedAt: true },
    orderBy: { updatedAt: 'desc' },
  })
}
```

**易错点**

- 只在 proxy 或布局中检查登录（布局在导航时不重新渲染，proxy 可能被绕过）
- 返回完整的数据库对象，敏感字段通过 props 泄露给浏览器
- `redirect()` 在 try 块中调用：`redirect` 通过抛出特殊错误实现，会被 catch 捕获；所以上面把 `redirect` 放在 catch 中而不是 try 中
- 指定 `algorithms`，防止算法混淆

---

### H2. 安全的 Server Action

- **难度**: 中级
- **标签**: Server Actions, 安全
- **面试官看什么**: 鉴权、输入校验、权限检查、错误返回给 useActionState

**题干**

实现一个更新项目名称的 Server Action，配合 `useActionState` 显示字段错误和成功状态。

**思路口述**

Server Action 是公开端点，所以必须：检查登录 → 用 Zod 校验输入 → 检查资源归属 → 更新 → 失效缓存。校验错误作为返回值而不是抛错，方便表单显示；未预期的错误再抛出给错误边界。

**参考实现**

```ts
// features/projects/actions.ts
'use server'

import { z } from 'zod'
import { updateTag } from 'next/cache'
import { verifySession } from '@/lib/dal'
import { db } from '@/lib/db'

const schema = z.object({
  id: z.string().uuid(),
  name: z.string().trim().min(1, '名称不能为空').max(50, '名称最多 50 个字'),
})

/** 表单提交结果：供 useActionState 渲染 */
export interface RenameState {
  ok: boolean
  message?: string
  fieldErrors?: Partial<Record<'name', string[]>>
}

/** 重命名项目：只允许项目所有者操作 */
export async function renameProject(_prev: RenameState, formData: FormData): Promise<RenameState> {
  const session = await verifySession()

  const parsed = schema.safeParse({ id: formData.get('id'), name: formData.get('name') })
  if (!parsed.success) {
    return { ok: false, fieldErrors: parsed.error.flatten().fieldErrors }
  }

  // where 同时匹配 id 和 ownerId：不是自己的项目时更新 0 条，避免越权
  const result = await db.project.updateMany({
    where: { id: parsed.data.id, ownerId: session.userId },
    data: { name: parsed.data.name },
  })
  if (result.count === 0) return { ok: false, message: '项目不存在或无权限' }

  updateTag(`projects-${session.userId}`)
  return { ok: true, message: '已保存' }
}
```

```tsx
// features/projects/rename-form.tsx
'use client'

import { useActionState } from 'react'
import { renameProject, type RenameState } from './actions'

const initialState: RenameState = { ok: false }

export function RenameForm({ id, name }: { id: string; name: string }) {
  const [state, formAction, isPending] = useActionState(renameProject, initialState)

  return (
    <form action={formAction}>
      <input type="hidden" name="id" value={id} />
      <input name="name" defaultValue={name} aria-invalid={!!state.fieldErrors?.name} />
      {state.fieldErrors?.name && <p role="alert">{state.fieldErrors.name[0]}</p>}
      <button disabled={isPending}>{isPending ? '保存中...' : '保存'}</button>
      {state.message && <p aria-live="polite">{state.message}</p>}
    </form>
  )
}
```

**易错点**

- 信任隐藏字段中的 id 而不检查归属
- 用 `update({ where: { id } })` 再检查 ownerId，存在先查后改的竞态，且更新已经发生
- 校验失败直接抛错，表单无法显示字段错误
- 忘记失效缓存，页面显示旧数据

---

### H3. Cache Components：缓存与按需失效

- **难度**: 高级
- **标签**: 缓存, Cache Components
- **面试官看什么**: 缓存边界、标签设计、updateTag 和 revalidateTag 的选择

**题干**

商品详情页：商品信息可以缓存（后台修改后需要失效），库存需要实时，购物车按用户显示。写出页面结构、缓存函数和后台修改的 Action。

**思路口述**

按数据特性拆组件：商品信息用 `'use cache'` + `cacheTag`；库存和购物车放在 Suspense 中请求时渲染。后台修改商品后，管理员需要立即看到修改结果，用 `updateTag`；如果是 Webhook 触发则用 `revalidateTag`。

**参考实现**

```ts
// features/products/queries.ts
import 'server-only'
import { cacheLife, cacheTag } from 'next/cache'
import { db } from '@/lib/db'

/** 获取商品信息（跨请求缓存，按商品 ID 打标签） */
export async function getProduct(id: string) {
  'use cache'
  cacheLife('days')
  cacheTag(`product-${id}`, 'products')
  return db.product.findUnique({
    where: { id },
    select: { id: true, name: true, description: true, price: true, images: true },
  })
}
```

```tsx
// app/products/[id]/page.tsx
import { Suspense } from 'react'
import { notFound } from 'next/navigation'
import { getProduct } from '@/features/products/queries'

export default async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  return (
    <main>
      <ProductInfo id={id} />
      <Suspense fallback={<p>库存查询中...</p>}>
        <Stock id={id} />
      </Suspense>
      <Suspense fallback={<CartSkeleton />}>
        <UserCart />
      </Suspense>
    </main>
  )
}

async function ProductInfo({ id }: { id: string }) {
  const product = await getProduct(id)
  if (!product) notFound()
  return (
    <section>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
    </section>
  )
}

// 未缓存：在请求时执行，由外层 Suspense 流式输出
async function Stock({ id }: { id: string }) {
  const stock = await getStock(id)
  return <p>{stock > 0 ? `库存 ${stock}` : '缺货'}</p>
}
```

```ts
// features/products/actions.ts
'use server'

import { updateTag } from 'next/cache'
import { requireAdmin } from '@/lib/dal'

/** 管理员修改商品价格，修改后立即可见 */
export async function updatePrice(id: string, price: number) {
  await requireAdmin()
  if (!Number.isFinite(price) || price <= 0) throw new Error('价格不合法')
  await db.product.update({ where: { id }, data: { price } })
  updateTag(`product-${id}`)
}
```

**易错点**

- 在 `'use cache'` 函数中读取 cookies（会把用户数据共享给所有人）
- 实时数据没有用 Suspense 包裹，开启 Cache Components 后构建报错
- 标签只打单个商品，列表页不更新
- Route Handler（Webhook）中使用 `updateTag`（只能在 Server Action 中使用）

---

### H4. 流式页面：并行获取 + Suspense

- **难度**: 中级
- **标签**: 流式渲染, 性能
- **面试官看什么**: 避免瀑布、Suspense 的粒度、传 Promise 给客户端组件

**题干**

仪表盘页面有统计卡片（快）、图表（慢，需要客户端交互）、最近订单（中等）。实现页面，使外壳立即显示，各部分独立加载。

**思路口述**

三个部分没有依赖关系，拆成独立组件各自获取数据，各自用 Suspense 包裹。图表是客户端组件，在服务端组件中**启动**请求但不 await，把 Promise 传给客户端组件用 `use()` 读取，这样不会阻塞外壳。

**参考实现**

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react'
import { SalesChart } from './sales-chart'

export default function DashboardPage() {
  // 不 await：请求立即开始，Promise 传给客户端组件
  const salesPromise = getSalesSeries()

  return (
    <div className="grid">
      <Suspense fallback={<CardsSkeleton />}>
        <StatCards />
      </Suspense>
      <Suspense fallback={<ChartSkeleton />}>
        <SalesChart salesPromise={salesPromise} />
      </Suspense>
      <Suspense fallback={<TableSkeleton />}>
        <RecentOrders />
      </Suspense>
    </div>
  )
}

async function StatCards() {
  const [revenue, orders, users] = await Promise.all([getRevenue(), getOrderCount(), getUserCount()])
  return <Cards revenue={revenue} orders={orders} users={users} />
}
```

```tsx
// app/dashboard/sales-chart.tsx
'use client'

import { use } from 'react'

export function SalesChart({ salesPromise }: { salesPromise: Promise<{ date: string; value: number }[]> }) {
  // use 读取 Promise：未完成时挂起，由外层 Suspense 显示 fallback
  const series = use(salesPromise)
  return <Chart data={series} />
}
```

**易错点**

- 在页面顶层依次 await 三个请求（瀑布且阻塞外壳）
- 在客户端组件中创建 Promise 并传给 `use()`：每次渲染都会创建新的 Promise，导致无限挂起
- 一个 Suspense 包裹所有部分，最慢的决定所有内容的显示时间
- 骨架屏尺寸与真实内容不一致，导致 CLS

---

### H5. 乐观更新的待办列表

- **难度**: 中级
- **标签**: useOptimistic, Server Actions
- **面试官看什么**: useOptimistic 的用法、失败回滚、临时 ID

**题干**

待办列表支持添加和切换完成状态，操作后立即更新 UI，失败时回滚并提示。

**参考实现**

```tsx
'use client'

import { useOptimistic, useRef, useTransition } from 'react'
import { addTodo, toggleTodo } from './actions'

interface Todo {
  id: string
  title: string
  done: boolean
  pending?: boolean
}

type OptimisticAction =
  | { type: 'add'; todo: Todo }
  | { type: 'toggle'; id: string }

/** 待办列表：服务端数据通过 props 传入，本地只维护乐观状态 */
export function TodoList({ todos }: { todos: Todo[] }) {
  const [optimisticTodos, applyOptimistic] = useOptimistic(todos, (state: Todo[], action: OptimisticAction) => {
    switch (action.type) {
      case 'add':
        return [...state, action.todo]
      case 'toggle':
        return state.map((todo) => (todo.id === action.id ? { ...todo, done: !todo.done } : todo))
    }
  })
  const [, startTransition] = useTransition()
  const formRef = useRef<HTMLFormElement>(null)

  async function handleAdd(formData: FormData) {
    const title = String(formData.get('title') ?? '').trim()
    if (!title) return
    formRef.current?.reset()
    // 临时 ID 仅用于 key；服务端返回后 props 中是真实 ID，乐观项自动被替换
    applyOptimistic({ type: 'add', todo: { id: `temp-${crypto.randomUUID()}`, title, done: false, pending: true } })
    const result = await addTodo(title)
    if (!result.ok) alert(result.message)
  }

  function handleToggle(id: string) {
    startTransition(async () => {
      applyOptimistic({ type: 'toggle', id })
      const result = await toggleTodo(id)
      // 失败时 transition 结束，乐观值被丢弃，UI 自动回到 props 中的真实状态
      if (!result.ok) alert(result.message)
    })
  }

  return (
    <>
      <form ref={formRef} action={handleAdd}>
        <input name="title" placeholder="新待办" />
        <button>添加</button>
      </form>
      <ul>
        {optimisticTodos.map((todo) => (
          <li key={todo.id} style={{ opacity: todo.pending ? 0.5 : 1 }}>
            <label>
              <input
                type="checkbox"
                checked={todo.done}
                disabled={todo.pending}
                onChange={() => handleToggle(todo.id)}
              />
              {todo.title}
            </label>
          </li>
        ))}
      </ul>
    </>
  )
}
```

**易错点**

- 在 transition 之外调用乐观更新函数（`form action` 本身就在 transition 中，事件处理函数中需要 `startTransition`）
- Action 中没有失效缓存，transition 结束后 props 仍然是旧数据，乐观项「消失」
- 临时项可以被操作（应该禁用，因为服务端还没有这个 ID）

---

### H6. Webhook Route Handler：签名验证

- **难度**: 高级
- **标签**: Route Handler, 安全
- **面试官看什么**: 原始请求体、常量时间比较、重放防护、幂等

**题干**

CMS 发布内容时调用 `POST /api/webhooks/cms`，请求头中有 `x-signature`（HMAC-SHA256）和 `x-timestamp`。验证后失效对应的缓存。

**思路口述**

签名要基于**原始请求体**计算，不能先解析 JSON 再序列化；用常量时间比较防止时序攻击；检查时间戳防止重放；处理要幂等，因为 CMS 可能重试。

**参考实现**

```ts
// app/api/webhooks/cms/route.ts
import { createHmac, timingSafeEqual } from 'node:crypto'
import { revalidateTag } from 'next/cache'

const SECRET = process.env.CMS_WEBHOOK_SECRET!
const MAX_SKEW_MS = 5 * 60 * 1000

function verifySignature(rawBody: string, timestamp: string, signature: string): boolean {
  const expected = createHmac('sha256', SECRET).update(`${timestamp}.${rawBody}`).digest('hex')
  const a = Buffer.from(expected, 'hex')
  const b = Buffer.from(signature, 'hex')
  // timingSafeEqual 要求长度相同，长度不同直接判定失败
  return a.length === b.length && timingSafeEqual(a, b)
}

/** CMS 发布回调：验证签名后按标签失效缓存 */
export async function POST(request: Request) {
  const signature = request.headers.get('x-signature') ?? ''
  const timestamp = request.headers.get('x-timestamp') ?? ''
  const rawBody = await request.text()

  const age = Math.abs(Date.now() - Number(timestamp))
  if (!Number.isFinite(age) || age > MAX_SKEW_MS) {
    return Response.json({ error: 'timestamp expired' }, { status: 401 })
  }
  if (!verifySignature(rawBody, timestamp, signature)) {
    return Response.json({ error: 'invalid signature' }, { status: 401 })
  }

  const event = JSON.parse(rawBody) as { type: string; slug?: string }
  if (event.type === 'post.published' || event.type === 'post.updated') {
    // 失效操作本身是幂等的，CMS 重试也不会产生副作用
    if (event.slug) revalidateTag(`post-${event.slug}`, 'max')
    revalidateTag('posts', 'max')
  }

  return Response.json({ ok: true })
}
```

**易错点**

- 用 `request.json()` 后再 `JSON.stringify` 计算签名（键顺序、空格可能不同）
- 用 `===` 比较签名（时序攻击）
- 不检查时间戳，旧请求可以被重放
- 在 Route Handler 中使用 `updateTag`（只能在 Server Action 中使用）
- 有副作用的处理（例如发送通知）没有按事件 ID 去重

---

### H7. 无限滚动分页

- **难度**: 中级
- **标签**: 数据获取, Server Actions
- **面试官看什么**: 首屏服务端渲染、游标分页、IntersectionObserver、并发保护

**题干**

文章列表首屏由服务端渲染前 20 条，滚动到底部时加载更多。

**参考实现**

```ts
// features/posts/actions.ts
'use server'

import { z } from 'zod'
import { getPostsPage } from './queries'

/** 加载下一页：游标分页，避免 offset 在数据变化时重复或遗漏 */
export async function loadMorePosts(cursor: unknown) {
  const parsed = z.string().nullable().parse(cursor)
  return getPostsPage({ cursor: parsed, limit: 20 })
}
```

```tsx
// app/posts/page.tsx
export default async function PostsPage() {
  const firstPage = await getPostsPage({ cursor: null, limit: 20 })
  return <PostList initialItems={firstPage.items} initialCursor={firstPage.nextCursor} />
}
```

```tsx
// features/posts/post-list.tsx
'use client'

import { useEffect, useRef, useState } from 'react'
import { loadMorePosts } from './actions'

interface Post {
  id: string
  title: string
}

/** 文章列表：首屏数据来自服务端，后续页通过 Server Action 加载 */
export function PostList({ initialItems, initialCursor }: { initialItems: Post[]; initialCursor: string | null }) {
  const [items, setItems] = useState(initialItems)
  const [cursor, setCursor] = useState(initialCursor)
  const [error, setError] = useState(false)
  const loadingRef = useRef(false)
  const sentinelRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    const sentinel = sentinelRef.current
    if (!sentinel || cursor === null) return

    const observer = new IntersectionObserver(async ([entry]) => {
      // 用 ref 而不是 state 做并发保护：回调中读取的 state 可能是旧值
      if (!entry.isIntersecting || loadingRef.current) return
      loadingRef.current = true
      try {
        const page = await loadMorePosts(cursor)
        setItems((prev) => [...prev, ...page.items])
        setCursor(page.nextCursor)
        setError(false)
      } catch {
        setError(true)
      } finally {
        loadingRef.current = false
      }
    }, { rootMargin: '200px' })

    observer.observe(sentinel)
    return () => observer.disconnect()
  }, [cursor])

  return (
    <>
      <ul>
        {items.map((post) => <li key={post.id}>{post.title}</li>)}
      </ul>
      {error && <button onClick={() => setError(false)}>加载失败，点击重试</button>}
      {cursor !== null && !error && <div ref={sentinelRef} aria-hidden>加载中...</div>}
      {cursor === null && <p>没有更多了</p>}
    </>
  )
}
```

**易错点**

- 使用 offset 分页，新增数据导致重复
- 没有并发保护，快速滚动时重复加载同一页
- 首屏也在客户端加载（失去 SSR 和 SEO）
- Server Action 是串行执行的，大量并发读请求时可以改用 Route Handler 或 TanStack Query

---

### H8. 国际化 proxy

- **难度**: 中级
- **标签**: proxy, 国际化
- **面试官看什么**: matcher 排除静态资源、语言协商优先级、重定向与重写的选择

**题干**

支持 `zh`、`en`，默认 `zh`。访问没有语言前缀的路径时，根据 Cookie 和 `Accept-Language` 重定向到对应语言。

**参考实现**

```ts
// proxy.ts
import { NextResponse, type NextRequest } from 'next/server'

const LOCALES = ['zh', 'en'] as const
const DEFAULT_LOCALE = 'zh'
type Locale = (typeof LOCALES)[number]

function isLocale(value: string | undefined): value is Locale {
  return LOCALES.includes(value as Locale)
}

/** 解析 Accept-Language，按 q 值排序后返回第一个支持的语言 */
function matchAcceptLanguage(header: string | null): Locale | undefined {
  if (!header) return undefined
  const candidates = header
    .split(',')
    .map((part) => {
      const [tag, q] = part.trim().split(';q=')
      return { lang: tag.toLowerCase().split('-')[0], q: q ? Number(q) : 1 }
    })
    .sort((a, b) => b.q - a.q)
  return candidates.find((c) => isLocale(c.lang))?.lang as Locale | undefined
}

/** 为没有语言前缀的请求补上语言前缀 */
export function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl
  const firstSegment = pathname.split('/')[1]
  if (isLocale(firstSegment)) return NextResponse.next()

  const cookieLocale = request.cookies.get('locale')?.value
  const locale =
    (isLocale(cookieLocale) ? cookieLocale : undefined) ??
    matchAcceptLanguage(request.headers.get('accept-language')) ??
    DEFAULT_LOCALE

  const url = request.nextUrl.clone()
  url.pathname = `/${locale}${pathname === '/' ? '' : pathname}`
  return NextResponse.redirect(url)
}

export const config = {
  // 排除 API、静态资源和带扩展名的文件
  matcher: ['/((?!api|_next/static|_next/image|.*\\..*).*)'],
}
```

**易错点**

- matcher 没有排除静态资源，图片和 JS 也被重定向
- 用重写（rewrite）而不是重定向：URL 中没有语言，搜索引擎无法区分不同语言的页面
- 忽略 `q` 权重
- 查询参数丢失（`clone()` 保留了 search）

---

### H9. 带服务端校验的注册表单

- **难度**: 中级
- **标签**: 表单, 校验
- **面试官看什么**: 前后端共享 schema、渐进增强、错误时保留输入

**题干**

注册表单包含邮箱和密码，客户端即时校验，服务端再次校验并检查邮箱是否已被注册。JS 未加载时也能提交。

**参考实现**

```ts
// features/auth/schema.ts
import { z } from 'zod'

/** 前后端共享的注册校验规则 */
export const signUpSchema = z.object({
  email: z.string().trim().toLowerCase().email('邮箱格式不正确'),
  password: z.string().min(8, '密码至少 8 位').max(72, '密码最多 72 位'),
})

export type SignUpFields = z.infer<typeof signUpSchema>
```

```ts
// features/auth/actions.ts
'use server'

import { redirect } from 'next/navigation'
import { signUpSchema } from './schema'

export interface SignUpState {
  values?: { email: string }
  errors?: Partial<Record<keyof import('./schema').SignUpFields | 'form', string[]>>
}

/** 注册：服务端是校验的最终依据，客户端校验只用于体验 */
export async function signUp(_prev: SignUpState, formData: FormData): Promise<SignUpState> {
  const raw = { email: String(formData.get('email') ?? ''), password: String(formData.get('password') ?? '') }
  const parsed = signUpSchema.safeParse(raw)
  // 出错时只回填邮箱，不回填密码
  if (!parsed.success) return { values: { email: raw.email }, errors: parsed.error.flatten().fieldErrors }

  const exists = await db.user.findUnique({ where: { email: parsed.data.email }, select: { id: true } })
  if (exists) return { values: { email: raw.email }, errors: { email: ['该邮箱已注册'] } }

  await createUser(parsed.data.email, await hashPassword(parsed.data.password))
  // redirect 抛出特殊错误，必须在 try/catch 之外调用
  redirect('/welcome')
}
```

```tsx
// features/auth/sign-up-form.tsx
'use client'

import { useActionState } from 'react'
import { signUp, type SignUpState } from './actions'

export function SignUpForm() {
  const [state, formAction, isPending] = useActionState<SignUpState, FormData>(signUp, {})

  return (
    <form action={formAction} noValidate>
      <label>
        邮箱
        <input name="email" type="email" required defaultValue={state.values?.email} />
      </label>
      {state.errors?.email && <p role="alert">{state.errors.email[0]}</p>}
      <label>
        密码
        <input name="password" type="password" required minLength={8} autoComplete="new-password" />
      </label>
      {state.errors?.password && <p role="alert">{state.errors.password[0]}</p>}
      <button disabled={isPending}>注册</button>
    </form>
  )
}
```

**易错点**

- 只做客户端校验
- 出错后表单被清空（需要回填 `defaultValue`）
- 回填密码
- 「该邮箱已注册」可能导致用户枚举：高安全场景可以统一提示，并通过邮件通知
- 检查后再创建存在竞态，数据库需要唯一索引作为最终保障

---

### H10. 动态 OG 图片

- **难度**: 初级
- **标签**: Metadata, 图片
- **面试官看什么**: 文件约定、ImageResponse 的限制、缓存

**题干**

为每篇文章生成包含标题和作者的分享图片。

**参考实现**

```tsx
// app/blog/[slug]/opengraph-image.tsx
import { ImageResponse } from 'next/og'
import { getPost } from '@/features/posts/queries'

export const size = { width: 1200, height: 630 }
export const contentType = 'image/png'
export const alt = '文章分享图'

/** 生成文章的 Open Graph 图片 */
export default async function Image({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  const post = await getPost(slug)

  return new ImageResponse(
    (
      <div
        style={{
          width: '100%',
          height: '100%',
          display: 'flex',
          flexDirection: 'column',
          justifyContent: 'space-between',
          padding: 64,
          background: 'linear-gradient(135deg, #0f172a, #1e3a8a)',
          color: 'white',
        }}
      >
        <div style={{ fontSize: 64, fontWeight: 700, lineClamp: 3 }}>{post?.title ?? '文章'}</div>
        <div style={{ fontSize: 32, opacity: 0.8 }}>{post?.author ?? ''}</div>
      </div>
    ),
    size,
  )
}
```

**易错点**

- ImageResponse 只支持 flexbox 和部分 CSS，不支持 grid
- 中文需要加载字体文件（通过 `fonts` 选项传入），否则显示为方框
- 图片按路由缓存，文章更新后需要失效
- 文件约定会自动生成 `og:image` 元数据，不需要手动设置

---

### H11. Route Handler 限流

- **难度**: 高级
- **标签**: 安全, 限流
- **面试官看什么**: 算法选择、多实例共享存储、原子操作、标识客户端的方式

**题干**

为 `/api/ai/chat` 实现每个用户每分钟最多 10 次请求的限流，支持多实例部署。

**思路口述**

多实例必须使用共享存储（Redis）。使用固定窗口实现最简单，但窗口边界会出现双倍流量；使用滑动窗口（有序集合）更精确。用 Lua 脚本保证「检查 + 记录」的原子性。登录用户按用户 ID 限流，未登录按 IP（注意只信任代理设置的 IP 头）。

**参考实现**

```ts
// lib/rate-limit.ts
import 'server-only'
import { redis } from './redis'

// 滑动窗口：移除窗口外的记录 → 统计数量 → 未超限则记录本次请求；整个过程在 Redis 中原子执行
const SLIDING_WINDOW_SCRIPT = `
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local member = ARGV[4]
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
local count = redis.call('ZCARD', key)
if count >= limit then
  return {0, count}
end
redis.call('ZADD', key, now, member)
redis.call('PEXPIRE', key, window)
return {1, count + 1}
`

/** 限流结果 */
export interface RateLimitResult {
  allowed: boolean
  remaining: number
}

/** 基于 Redis 有序集合的滑动窗口限流，适用于多实例部署 */
export async function rateLimit(key: string, limit: number, windowMs: number): Promise<RateLimitResult> {
  const now = Date.now()
  const [allowed, count] = (await redis.eval(SLIDING_WINDOW_SCRIPT, {
    keys: [`ratelimit:${key}`],
    arguments: [String(now), String(windowMs), String(limit), `${now}-${crypto.randomUUID()}`],
  })) as [number, number]
  return { allowed: allowed === 1, remaining: Math.max(0, limit - count) }
}
```

```ts
// app/api/ai/chat/route.ts
import { rateLimit } from '@/lib/rate-limit'
import { getOptionalSession } from '@/lib/dal'

export async function POST(request: Request) {
  const session = await getOptionalSession()
  // x-forwarded-for 可以被伪造，只能在可信代理覆盖了这个头的部署中使用
  const clientIp = request.headers.get('x-forwarded-for')?.split(',')[0]?.trim() ?? 'unknown'
  const key = session ? `user:${session.userId}` : `ip:${clientIp}`

  const { allowed, remaining } = await rateLimit(key, 10, 60_000)
  if (!allowed) {
    return Response.json(
      { error: '请求过于频繁，请稍后再试' },
      { status: 429, headers: { 'Retry-After': '60', 'X-RateLimit-Remaining': '0' } },
    )
  }

  const body = await request.json()
  const response = await handleChat(body)
  response.headers.set('X-RateLimit-Remaining', String(remaining))
  return response
}
```

**易错点**

- 使用内存 Map 限流（多实例下每个实例单独计数，重启后丢失）
- 「查询 + 写入」分成两次调用，并发时超限
- 直接信任客户端发送的 `X-Forwarded-For`
- 有序集合的成员相同时会覆盖，需要唯一值
- Redis 不可用时的策略：失败放行（可用性优先）还是失败拒绝（安全优先）需要明确

---

### H12. 搜索页：URL 作为状态

- **难度**: 中级
- **标签**: searchParams, 导航
- **面试官看什么**: 服务端读取 searchParams、防抖更新 URL、useTransition 显示加载状态

**题干**

搜索页：输入关键词后更新 URL（`?q=xxx&page=1`），服务端组件根据 URL 获取结果。刷新和分享链接保留搜索条件。

**参考实现**

```tsx
// app/search/page.tsx
import { Suspense } from 'react'
import { SearchInput } from './search-input'

export default async function SearchPage({ searchParams }: { searchParams: Promise<{ q?: string; page?: string }> }) {
  const { q = '', page = '1' } = await searchParams
  const pageNumber = Math.max(1, Number.parseInt(page, 10) || 1)

  return (
    <main>
      <SearchInput defaultValue={q} />
      {/* key 变化时重新挂起，显示 fallback */}
      <Suspense key={`${q}-${pageNumber}`} fallback={<p>搜索中...</p>}>
        <Results q={q} page={pageNumber} />
      </Suspense>
    </main>
  )
}

async function Results({ q, page }: { q: string; page: number }) {
  if (!q) return <p>输入关键词开始搜索</p>
  const { items, total } = await searchProducts({ q, page, pageSize: 20 })
  return <ResultList items={items} total={total} page={page} />
}
```

```tsx
// app/search/search-input.tsx
'use client'

import { useEffect, useRef, useTransition } from 'react'
import { usePathname, useRouter, useSearchParams } from 'next/navigation'

/** 搜索输入框：防抖后把关键词写入 URL，结果由服务端根据 URL 渲染 */
export function SearchInput({ defaultValue }: { defaultValue: string }) {
  const router = useRouter()
  const pathname = usePathname()
  const searchParams = useSearchParams()
  const [isPending, startTransition] = useTransition()
  const timerRef = useRef<ReturnType<typeof setTimeout>>(undefined)

  useEffect(() => () => clearTimeout(timerRef.current), [])

  function handleChange(value: string) {
    clearTimeout(timerRef.current)
    timerRef.current = setTimeout(() => {
      const params = new URLSearchParams(searchParams)
      if (value) params.set('q', value)
      else params.delete('q')
      // 关键词变化时回到第一页
      params.delete('page')
      startTransition(() => {
        // replace：输入过程不产生大量历史记录
        router.replace(`${pathname}?${params.toString()}`, { scroll: false })
      })
    }, 300)
  }

  return (
    <div>
      <input
        type="search"
        defaultValue={defaultValue}
        onChange={(event) => handleChange(event.target.value)}
        placeholder="搜索商品"
        aria-busy={isPending}
      />
      {isPending && <span>更新中...</span>}
    </div>
  )
}
```

**易错点**

- 每次输入都更新 URL（没有防抖），产生大量请求
- 用 `push` 导致后退时需要一个一个字退回
- 关键词变化时没有重置页码
- 不校验 `page`（`NaN`、负数）
- `useSearchParams` 所在的客户端组件在静态渲染时需要被 Suspense 包裹，否则整个页面会退化为客户端渲染
