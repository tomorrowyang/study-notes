# Next.js 中级 Q&A

> 目标：讲清 React Server Components 的原理、组件边界、流式渲染、渲染策略、缓存体系（旧模型与 Cache Components）、Server Actions 的工作方式，以及鉴权、hydration 等常见问题。

---

## 一、RSC 原理

### Q1. 一个 App Router 页面的完整渲染流程是怎样的？

- **难度**: 中级
- **标签**: RSC, 渲染

**答案**

**首次加载（硬导航）**：

```text
服务端
  1. 匹配路由，组装 layout → page 的组件树
  2. 渲染 Server Components：执行 async 组件、获取数据
     → 生成 RSC Payload（描述 UI 树的序列化格式）
        - 服务端组件的渲染结果
        - 客户端组件的占位符 + 对应的 JS 模块引用 + 传入的 props
  3. 用 RSC Payload + 客户端组件，渲染出 HTML（客户端组件也在这里预渲染）
  4. 流式发送：HTML 外壳 → Suspense 内容陆续到达；RSC Payload 内联在 HTML 的 script 中
浏览器
  5. 显示 HTML（非交互）
  6. 加载客户端组件的 JS
  7. React 用 RSC Payload 构建组件树，hydrate 客户端组件（绑定事件）
```

**客户端导航（软导航）**：

```text
点击 Link → 请求目标路由的 RSC Payload（不是 HTML）
         → React 把新的服务端组件树合并到当前树中
         → 共享的布局不重新渲染，客户端状态保留
```

**口述模板（30–60 秒）**

1. 一句话结论：服务端先渲染服务端组件得到 RSC Payload，再结合客户端组件生成 HTML，流式发送。
2. 关键机制：浏览器显示 HTML 后加载客户端 JS，用 Payload 构建组件树并 hydrate。
3. 对比或注意点：客户端导航只请求 RSC Payload，不请求 HTML。
4. 可追问时补充：共享布局在导航中保持状态，因为 React 只合并变化的部分。

**常见追问**

- 为什么不只发送 HTML？→ HTML 只能用于首次显示；客户端导航时需要一种能与现有 React 树合并、保留客户端状态的格式，这就是 RSC Payload。

---

### Q2. RSC Payload 是什么？

- **难度**: 中级
- **标签**: RSC

**答案**

RSC Payload 是 React Server Components 渲染结果的**紧凑序列化格式**（流式的、按行分割的文本），包含：

- 服务端组件渲染出的 UI 树（元素、文本）
- **客户端组件的引用**：模块 ID 和导出名（客户端据此加载对应的 JS 块）
- 传给客户端组件的 **props**（必须可以序列化）
- Suspense 边界的占位和后续填充的内容
- Server Action 的引用（ID）

```text
（示意，非精确格式）
1:I["./LikeButton.js",["chunk-abc.js"],"LikeButton"]
0:["$","main",null,{"children":[["$","h1",null,{"children":"文章标题"}],["$","$L1",null,{"initialCount":42}]]}]
2:["$","p",null,{"children":"延迟加载的评论"}]
```

**可以序列化的 props**：基本类型、数组、普通对象、Date、Map、Set、BigInt、Promise（可以流式传递）、JSX、Server Action；**不能**：普通函数、类实例、Symbol（全局注册的除外）。

**口述模板（30–60 秒）**

1. 一句话结论：RSC Payload 是服务端组件树的序列化格式，用于首次 hydration 和客户端导航。
2. 关键机制：包含服务端组件的渲染结果、客户端组件的模块引用和 props。
3. 对比或注意点：传给客户端组件的 props 必须可以序列化。
4. 可追问时补充：格式是流式的，Suspense 内容可以陆续到达。

**常见追问**

- Payload 会很大吗？→ 服务端组件渲染的所有内容和传给客户端组件的 props 都在其中；传入大量数据（例如整个列表）给客户端组件会让 Payload 和 HTML 都变大（数据出现两次：一次在 HTML 中，一次在 Payload 中）。

---

### Q3. Server Components 和 SSR 有什么区别？

- **难度**: 中级
- **标签**: RSC, SSR

**答案**

| 对比项 | Server Components | SSR |
|--------|-------------------|-----|
| 是什么 | 一种**组件类型** | 一种**渲染时机**（在服务端生成 HTML） |
| 输出 | RSC Payload | HTML |
| 客户端代码 | 不发送 | 组件代码仍然发送，需要 hydration |
| 重新渲染 | 只在服务端（导航、refresh） | 首次在服务端，之后在客户端 |
| 可以访问 | 数据库、文件、密钥 | 取决于运行位置 |

**两者是正交的**：

- 客户端组件也会被 SSR（生成首屏 HTML）
- 服务端组件可以在构建时运行（静态），也可以在请求时运行（动态）
- 传统 SSR（Pages Router）：所有组件在服务端渲染 HTML，然后**全部**在客户端 hydrate

**口述模板（30–60 秒）**

1. 一句话结论：Server Components 是组件类型，SSR 是渲染时机，两者是正交的概念。
2. 关键机制：服务端组件的代码不发送到浏览器；SSR 的组件代码仍然发送并 hydrate。
3. 对比或注意点：客户端组件也会被 SSR。
4. 可追问时补充：服务端组件可以在构建时或请求时运行。

**常见追问**

- 有了 RSC 还需要 SSR 吗？→ 需要。RSC Payload 不是 HTML，首屏仍然需要 SSR 生成 HTML 才能快速显示。

---

### Q4. `'use client'` 边界有哪些常见误区？

- **难度**: 中级
- **标签**: RSC, 组件边界

**答案**

| 误区 | 正确理解 |
|------|----------|
| `'use client'` 组件只在客户端运行 | 首次加载时也会在服务端预渲染 HTML |
| 每个客户端组件都要写 `'use client'` | 只需要在**边界入口**写，被它导入的组件自动成为客户端组件 |
| 在根布局加 `'use client'` 很方便 | 整个应用都变成客户端组件，失去 RSC 的优势 |
| 客户端组件中导入服务端组件也可以 | 被导入的组件会变成客户端组件；如果它使用了服务端 API 会报错 |
| 使用 Context 必须把整个页面变成客户端组件 | 只需要 Provider 是客户端组件，children 仍然可以是服务端组件 |
| 第三方组件库在服务端组件中直接使用 | 没有声明 `'use client'` 的第三方交互组件需要包装一层客户端组件再导出 |

**包装第三方组件**：

```tsx
// components/carousel.tsx
'use client'
export { Carousel } from 'some-carousel-lib'
```

**口述模板（30–60 秒）**

1. 一句话结论：'use client' 是模块边界，不是「只在客户端运行」。
2. 关键机制：边界入口声明后，它导入的模块都进入客户端包。
3. 对比或注意点：不要在高层级声明，Context Provider 只需要包裹 children。
4. 可追问时补充：未声明的第三方交互组件需要包装一层再使用。

**常见追问**

- 怎么检查哪些代码进入了客户端包？→ 使用 `@next/bundle-analyzer`（16 中有新的 bundle analyzer），查看客户端包中的模块。

---

### Q5. `'use server'` 和 `'use client'` 是对称的吗？

- **难度**: 中级
- **标签**: RSC

**答案**

不完全对称：

| 指令 | 含义 |
|------|------|
| `'use client'` | 标记**从服务端到客户端**的边界：这个模块中的组件可以被服务端组件渲染，代码发送到客户端 |
| `'use server'` | 标记**从客户端到服务端**的边界：这个模块（或函数）导出的**函数**可以被客户端调用，代码留在服务端，客户端得到的是一个引用（调用时发起网络请求） |

**常见误区**：

- `'use server'` **不是**用来标记服务端组件的（服务端组件是默认的，不需要指令）
- `'use server'` 文件只能导出**异步函数**
- 不需要客户端调用的服务端函数**不要**加 `'use server'`，否则会无意中暴露为公开端点

**口述模板（30–60 秒）**

1. 一句话结论：'use client' 让组件可以从服务端传到客户端，'use server' 让函数可以从客户端调用服务端。
2. 关键机制：'use server' 的函数在客户端只是一个引用，调用时发起 POST 请求。
3. 对比或注意点：'use server' 不是用来标记服务端组件的。
4. 可追问时补充：只有需要被客户端调用的函数才加 'use server'，否则会暴露为公开端点。

**常见追问**

- 数据库查询函数应该加 `'use server'` 吗？→ 不应该，它只在服务端组件中调用，应该用 `import 'server-only'` 保护。

---

## 二、流式渲染与 hydration

### Q6. 流式渲染的原理是什么？

- **难度**: 中级
- **标签**: 流式渲染

**答案**

**基于 HTTP 分块传输**（`Transfer-Encoding: chunked`，HTTP/2 中是数据帧）：

```text
1. 服务端立即发送 HTML 外壳：布局 + Suspense 的 fallback（带占位标记）
2. 某个 Suspense 中的数据准备好后，发送一段：
   <div hidden id="S:1">真实内容</div>
   <script>$RC("B:1", "S:1")</script>   ← 把内容移动到占位符的位置
3. 同时发送对应的 RSC Payload 片段
4. 所有内容发送完毕后结束响应
```

**选择性 hydration**：

- 各个 Suspense 边界独立 hydrate
- 用户与某个尚未 hydrate 的区域交互时，React **优先** hydrate 这个区域

**注意事项**：

- 流开始后**无法修改状态码和响应头**：`notFound()`、`redirect()` 应该在第一个 Suspense 之前（或者在 metadata 中）调用；流开始后调用会通过客户端脚本处理
- 代理和 CDN 需要支持流式（不能缓冲整个响应）；Nginx 需要关闭 `proxy_buffering`

**口述模板（30–60 秒）**

1. 一句话结论：流式渲染利用分块传输，先发送外壳，数据准备好后再发送内容片段替换占位。
2. 关键机制：每个 Suspense 边界独立发送和 hydrate，用户交互的区域优先 hydrate。
3. 对比或注意点：流开始后无法修改状态码和响应头。
4. 可追问时补充：反向代理需要关闭缓冲，否则流式失效。

**常见追问**

- 为什么部署后流式不生效？→ 常见原因是反向代理或 CDN 缓冲了响应、开启了压缩缓冲，或者使用了不支持流式的运行环境。

---

### Q7. hydration mismatch 是怎么产生的？怎么处理？

- **难度**: 中级
- **标签**: hydration

**答案**

**原因**：客户端组件首次渲染的结果与服务端生成的 HTML 不一致。

| 原因 | 解决 |
|------|------|
| `Date.now()`、`Math.random()` | 在服务端生成后作为 props 传入；或者在 useEffect 中设置 |
| `typeof window !== 'undefined'` 分支 | 首次渲染保持一致，在 useEffect 中切换 |
| 读取 localStorage、屏幕宽度 | useEffect 中读取；或者 `dynamic(() => import(...), { ssr: false })` |
| 时区和语言导致的日期格式不同 | 服务端和客户端使用相同的时区，或者只在客户端格式化 |
| 不合法的 HTML 嵌套 | `<p>` 中嵌套 `<div>`、`<a>` 嵌套 `<a>` 等，浏览器会修正结构 |
| 浏览器扩展修改 DOM | 通常无法避免；`suppressHydrationWarning` 用于少量可预期的情况 |
| ID 不一致 | 使用 `useId()` |

**React 19 的改进**：hydration 错误信息会显示**差异**，便于定位。

```tsx
'use client'
function LocalTime({ iso }: { iso: string }) {
  const [text, setText] = useState<string>()
  useEffect(() => {
    setText(new Date(iso).toLocaleString())
  }, [iso])
  return <time dateTime={iso}>{text ?? iso}</time>
}
```

**口述模板（30–60 秒）**

1. 一句话结论：客户端首次渲染与服务端 HTML 不一致就会导致 hydration mismatch。
2. 关键机制：常见原因是时间、随机数、浏览器专属 API、不合法的 HTML 嵌套。
3. 对比或注意点：首次渲染保持与服务端一致，浏览器相关的值在 useEffect 中设置。
4. 可追问时补充：React 19 的错误信息会显示具体差异。

**常见追问**

- mismatch 的后果是什么？→ React 会丢弃该部分的服务端 HTML，在客户端重新渲染，损失 SSR 的性能收益，并可能闪烁。

---

## 三、渲染策略

### Q8. 旧模型中，什么会让路由变成动态渲染？

- **难度**: 中级
- **标签**: 渲染策略

**答案**

在 Next.js 14 / 15（未开启 Cache Components）中，路由默认尝试**静态渲染**，以下情况会让它变为**动态渲染**：

- 使用**动态 API**：`cookies()`、`headers()`、`draftMode()`、`connection()`、页面的 `searchParams`
- 使用不缓存的 `fetch`（15 中 fetch 默认不缓存，但**不会**单独让路由变为动态，除非显式设置 `cache: 'no-store'` 或使用了动态 API；具体以版本行为为准）
- 路由段配置 `export const dynamic = 'force-dynamic'` 或 `revalidate = 0`
- 动态路由参数且没有 `generateStaticParams`（请求时渲染后可以被缓存）

**问题**：一个组件中的一次 `cookies()` 调用，就让**整个路由**变成动态的 → 这正是 PPR 和 Cache Components 要解决的问题。

**检查方法**：`next build` 的输出中查看每个路由的符号（○ 静态、ƒ 动态、◐ PPR）。

**口述模板（30–60 秒）**

1. 一句话结论：使用 cookies、headers、searchParams 等请求相关的 API 会让路由变为动态渲染。
2. 关键机制：也可以通过路由段配置强制动态。
3. 对比或注意点：一个组件中的动态 API 会让整个路由变为动态。
4. 可追问时补充：PPR 和 Cache Components 允许静态和动态共存于同一个页面。

**常见追问**

- 为什么我的页面在本地是动态的，部署后是静态的？→ 开发环境总是按请求渲染；生产构建时才进行静态分析。

---

### Q9. 什么是 PPR（部分预渲染）？

- **难度**: 中级
- **标签**: PPR, 渲染策略

**答案**

**PPR（Partial Prerendering）**：同一个页面中，**静态部分**在构建时预渲染为外壳，**动态部分**用 Suspense 包裹，在请求时流式填充。

```tsx
export default function ProductPage() {
  return (
    <>
      <Header />                       {/* 静态：构建时预渲染 */}
      <ProductDetails />               {/* 静态（使用缓存的数据） */}
      <Suspense fallback={<CartSkeleton />}>
        <UserCart />                   {/* 动态：读取 cookies，请求时渲染 */}
      </Suspense>
    </>
  )
}
```

**请求过程**：

```text
请求 → 立即从 CDN / 缓存返回静态外壳（含 fallback），TTFB 极低
     → 同一个响应中，服务端继续渲染动态部分，流式发送
```

**优点**：兼得静态页面的速度和动态页面的个性化；不需要在路由级别二选一。

**版本**：14 中实验性引入；**16 中通过 Cache Components 正式提供**（`cacheComponents: true` 即启用基于 PPR 的模型）。

**口述模板（30–60 秒）**

1. 一句话结论：PPR 让同一页面的静态部分预渲染为外壳，动态部分请求时流式填充。
2. 关键机制：Suspense 边界划分静态和动态；外壳可以从缓存立即返回。
3. 对比或注意点：不再需要在路由级别选择静态或动态。
4. 可追问时补充：16 中通过 Cache Components 正式提供。

**常见追问**

- 动态部分必须用 Suspense 包裹吗？→ 是的，开启 Cache Components 后，在 Suspense 外部访问请求数据（且没有缓存）会在构建时报错，提示你明确选择缓存或流式。

---

## 四、缓存体系

### Q10. 旧模型中 Next.js 有哪几层缓存？

- **难度**: 中级
- **标签**: 缓存

**答案**

适用于 Next.js 14 / 15（未开启 Cache Components）：

| 缓存 | 位置 | 内容 | 持续时间 | 失效方式 |
|------|------|------|----------|----------|
| **Request Memoization** | 服务端，单次渲染 | 相同的 fetch GET 请求结果 | 一次请求的渲染过程 | 自动 |
| **Data Cache** | 服务端，持久化 | fetch 的结果 | 跨请求、跨部署（可配置） | `revalidate` 时间、`revalidateTag`、`revalidatePath` |
| **Full Route Cache** | 服务端，持久化 | 静态路由的 HTML 和 RSC Payload | 直到重新验证或重新部署 | 数据缓存失效、重新部署 |
| **Router Cache** | 客户端内存 | 访问过和预取的路由的 RSC Payload | 会话期间 / 一段时间 | `router.refresh()`、Server Action 中调用 revalidate、cookies 修改 |

**关系**：

```text
请求 → Full Route Cache 命中？→ 返回
     → 未命中：渲染 → 每个 fetch：Request Memoization → Data Cache → 数据源
客户端导航 → Router Cache 命中？→ 直接使用
```

**15 的变化**：fetch 默认不进入 Data Cache；Router Cache 中页面段的 `staleTime` 默认为 0（每次导航都重新获取，布局等仍然复用）。

**口述模板（30–60 秒）**

1. 一句话结论：旧模型有四层缓存：请求记忆、数据缓存、完整路由缓存、客户端路由缓存。
2. 关键机制：前两层缓存数据，第三层缓存渲染结果，第四层在浏览器缓存 RSC Payload。
3. 对比或注意点：15 中 fetch 默认不缓存，路由缓存的页面段默认不复用。
4. 可追问时补充：这种隐式的多层缓存让很多开发者困惑，这也是 16 推出 Cache Components 的原因。

**常见追问**

- 「数据改了但页面不更新」通常是哪层缓存的问题？→ 先确认版本；常见是 Data Cache 或 Full Route Cache 没有失效（14），或者 Router Cache 中的旧页面（客户端），需要在变更后调用 revalidate 或 router.refresh。

---

### Q11. Next.js 16 的 Cache Components 是什么？

- **难度**: 中级
- **标签**: 缓存, Cache Components

**答案**

**Cache Components** 是 Next.js 16 引入的新缓存编程模型，16.4 起官方推荐所有应用使用，新项目默认开启，Next.js 17 将成为默认。

```ts
// next.config.ts
const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true,  // 16.4 中作为模型的一部分
}
```

**核心理念**：

- **默认动态**：数据默认在请求时获取，不再有隐式缓存
- **显式缓存**：用 `'use cache'` 标记可以缓存的组件、函数或文件，类似于组件级别的 `Cache-Control`
- **静态外壳**：构建时，可以缓存的部分和 Suspense 的 fallback 组成静态外壳（PPR）

```tsx
import { cacheLife, cacheTag } from 'next/cache'

async function ProductList({ category }: { category: string }) {
  'use cache'
  cacheLife('hours')
  cacheTag(`products-${category}`)
  const products = await db.product.findMany({ where: { category } })
  return <List items={products} />
}
```

**替代的旧 API**：`fetch` 的 `cache` / `next.revalidate` 选项、路由段配置 `dynamic`、`revalidate`、`fetchCache`、`unstable_cache`。

**口述模板（30–60 秒）**

1. 一句话结论：Cache Components 把缓存从隐式变为显式，默认动态，用 'use cache' 声明缓存。
2. 关键机制：'use cache' 可以用在组件、函数、文件上，配合 cacheLife 和 cacheTag 控制时长和失效。
3. 对比或注意点：替代了 fetch 缓存选项和路由段配置。
4. 可追问时补充：16.4 起新项目默认开启，17 将成为默认。

**常见追问**

- 为什么要从隐式改为显式？→ 旧模型中缓存的行为难以预测（为什么这里被缓存了？为什么数据不更新？），显式声明让缓存的位置和时长一目了然。

---

### Q12. `'use cache'` 的缓存键是怎么确定的？有哪些限制？

- **难度**: 中级
- **标签**: 缓存, Cache Components

**答案**

**缓存键**由编译器自动生成，包括：

- 构建 ID
- 函数的唯一 ID（位置）
- **可序列化的参数**
- **闭包中捕获的变量**

```tsx
async function Price({ productId, currency }: { productId: string; currency: string }) {
  'use cache'
  // productId 和 currency 都是缓存键的一部分
  return <span>{await getPrice(productId, currency)}</span>
}
```

**限制**：

- 使用 `'use cache'` 的函数必须是 **async**
- 参数和返回值必须可以**序列化**
- **不能在缓存函数内部直接读取请求数据**（`cookies()`、`headers()`、`searchParams`）：缓存的内容会被所有用户共享。需要时在外部读取，作为参数传入
- 可以接收不可序列化的值（例如 `children`、Server Action）**透传**，但不能在内部读取或调用它们

**变体**：

| 指令 | 存储 |
|------|------|
| `'use cache'` | 默认缓存（服务端内存 + 客户端路由缓存） |
| `'use cache: remote'` | 平台提供的远程缓存（Redis、KV），适合多实例 |
| `'use cache: private'` | 可以读取 cookies 等用户数据，**只缓存在客户端**，不在服务端共享 |

**口述模板（30–60 秒）**

1. 一句话结论：缓存键由构建 ID、函数 ID、参数和闭包变量自动生成。
2. 关键机制：参数和返回值必须可以序列化，函数必须是 async。
3. 对比或注意点：缓存函数中不能读取 cookies 等请求数据，否则会把用户数据共享给所有人。
4. 可追问时补充：多实例使用 use cache: remote，用户相关数据使用 use cache: private。

**常见追问**

- 为什么不能在缓存函数里读 cookies？→ 缓存结果在服务端被所有请求共享，读取了用户 A 的 cookie 生成的内容可能返回给用户 B；框架在构建时就会报错阻止这种用法。

---

### Q13. `cacheLife`、`cacheTag`、`updateTag`、`revalidateTag`、`refresh` 有什么区别？

- **难度**: 中级
- **标签**: 缓存

**答案**

**`cacheLife`**：设置缓存的生命周期（预设配置或自定义）：

```ts
cacheLife('minutes')                       // 预设：seconds / minutes / hours / days / weeks / max
cacheLife({ stale: 60, revalidate: 300, expire: 3600 })
```

- `stale`：客户端在这段时间内直接使用缓存，不检查服务端
- `revalidate`：服务端在这之后，下一个请求触发后台重新生成（返回旧内容）
- `expire`：超过后不再返回旧内容，必须重新生成

**`cacheTag`**：给缓存打标签，便于按需失效。

**失效 API**：

| API | 行为 | 使用位置 | 场景 |
|-----|------|----------|------|
| `updateTag(tag)` | **立即**失效，当前请求就能读到新数据（读到自己的写入） | 只能在 Server Action 中 | 用户修改了自己的资料，提交后立即看到 |
| `revalidateTag(tag, profile)` | 标记为过期，按 stale-while-revalidate 在后台更新 | Server Action、Route Handler | 后台发布了文章，稍后可见即可 |
| `revalidatePath(path)` | 按路径失效 | Server Action、Route Handler | 不方便打标签时 |
| `refresh()` | 刷新客户端路由，重新获取**未缓存**的动态数据，不影响缓存 | Server Action | 刷新通知数量等动态内容 |

**口述模板（30–60 秒）**

1. 一句话结论：cacheLife 设置缓存时长，cacheTag 打标签，updateTag、revalidateTag、refresh 用于失效和刷新。
2. 关键机制：updateTag 立即失效，用户能立刻看到自己的修改；revalidateTag 后台更新。
3. 对比或注意点：updateTag 只能在 Server Action 中使用。
4. 可追问时补充：refresh 只刷新动态数据，不碰缓存。

**常见追问**

- Webhook（例如 CMS 发布）应该用哪个？→ Route Handler 中调用 `revalidateTag(tag, 'max')`，不需要「读到自己的写入」。

---

### Q14. 客户端路由缓存（Router Cache）是怎么工作的？

- **难度**: 中级
- **标签**: 缓存, 导航

**答案**

- 浏览器内存中缓存**访问过**和**预取**的路由段的 RSC Payload
- 前进 / 后退时直接使用缓存，并恢复滚动位置
- **布局**在导航中复用（这也是布局能保持状态的原因）

**时长**（旧模型）：

- 14：动态页面 30 秒，静态 / 完整预取的页面 5 分钟
- 15：**页面段默认 `staleTime` 为 0**（导航时重新获取），布局和 loading 仍然复用；可以通过 `experimental.staleTimes` 调整

**16 Cache Components**：客户端缓存的时长由 `cacheLife` 的 `stale` 值决定；动态部分每次导航都获取。

**失效**：

- Server Action 中调用 `revalidatePath` / `revalidateTag` / `updateTag` → 清除客户端路由缓存
- 在 Server Action 中修改 cookies
- `router.refresh()`
- 刷新页面（硬导航）

**口述模板（30–60 秒）**

1. 一句话结论：客户端路由缓存在浏览器内存中保存访问过和预取的路由段。
2. 关键机制：前进后退和共享布局直接复用缓存。
3. 对比或注意点：15 中页面段默认不复用；16 中由 cacheLife 的 stale 决定。
4. 可追问时补充：Server Action 中失效缓存或修改 cookies 会清除客户端缓存。

**常见追问**

- 在客户端组件中修改了数据（调用了外部 API），页面上的服务端组件数据怎么更新？→ 调用 `router.refresh()`；更好的方式是通过 Server Action 修改并在其中失效相关缓存。

---

## 五、Server Actions

### Q15. Server Actions 的底层原理是什么？

- **难度**: 中级
- **标签**: Server Actions

**答案**

**编译时**：

- 打包工具识别 `'use server'` 函数，为每个函数生成唯一的 **Action ID**
- 服务端保留函数代码，客户端得到一个**引用**（包含 ID 的代理函数）
- 内联在服务端组件中的 Action 捕获的闭包变量会被**加密**后发送给客户端，调用时再发送回来解密

**调用时**：

```text
客户端调用 action(args) 或表单提交
  → POST 请求到当前页面的 URL
     请求头：Next-Action: <Action ID>
     请求体：序列化的参数（FormData 或 RSC 格式）
  → 服务端根据 ID 找到函数并执行
  → 响应：返回值 + （如果调用了 revalidate / redirect）更新后的 RSC Payload
  → 客户端在一次往返中同时得到结果和新的 UI
```

**特性**：

- 同一个客户端发起的多个 Action **按顺序执行**（排队）
- 表单在 JS 加载前提交，退化为标准的表单 POST（渐进增强）
- 只能使用 POST，内置检查 `Origin` 与 `Host` 是否一致（防 CSRF）

**口述模板（30–60 秒）**

1. 一句话结论：Server Action 编译为一个带 ID 的引用，调用时向当前页面发送带 Action ID 的 POST 请求。
2. 关键机制：服务端执行函数，返回结果和更新后的 RSC Payload，一次往返完成。
3. 对比或注意点：同一客户端的 Action 按顺序执行；闭包变量会被加密。
4. 可追问时补充：内置 Origin 检查防 CSRF，但鉴权仍需自己实现。

**常见追问**

- 闭包变量加密了，是不是就安全了？→ 加密防止篡改和窥探，但 Action 的参数仍然由客户端控制，所有参数都要校验；而且不要依赖闭包传递敏感数据。

---

### Q16. Server Actions 有哪些安全注意事项？

- **难度**: 中级
- **标签**: Server Actions, 安全

**答案**

**核心原则**：每个 Server Action 都是一个**公开的 HTTP 端点**，任何人都可以用任意参数调用它，即使 UI 上没有对应的按钮。

| 风险 | 措施 |
|------|------|
| 未鉴权调用 | 每个 Action 内部检查登录状态 |
| 越权（修改别人的数据） | 检查资源归属和权限：`where: { id, ownerId: session.userId }` |
| 恶意输入 | 用 Zod 等校验所有参数（类型、长度、格式） |
| 不要相信隐藏字段 | 价格、用户 ID、角色等不能从表单读取，在服务端获取 |
| 敏感数据返回 | 只返回需要的字段 |
| 滥用（刷接口） | 限流 |
| 无意暴露 | 不需要客户端调用的函数不要加 `'use server'` |
| CSRF | 内置 Origin 检查；配置 `serverActions.allowedOrigins`（使用反向代理时） |
| 框架漏洞 | 及时升级（React2Shell 就是 Server Function 处理请求时的反序列化漏洞） |

```ts
'use server'

export async function deletePost(postId: unknown) {
  const session = await getSession()
  if (!session) throw new Error('未登录')

  const id = z.string().uuid().parse(postId)
  // 同时匹配 ID 和作者，防止删除别人的文章
  const result = await db.post.deleteMany({ where: { id, authorId: session.userId } })
  if (result.count === 0) throw new Error('无权限或文章不存在')

  updateTag(`posts-${session.userId}`)
}
```

**口述模板（30–60 秒）**

1. 一句话结论：Server Action 是公开端点，必须在内部做鉴权、权限检查和输入校验。
2. 关键机制：任何人都可以用任意参数调用，不能依赖 UI 的限制。
3. 对比或注意点：价格、用户 ID 等关键数据必须在服务端获取，不能来自客户端。
4. 可追问时补充：及时升级框架，React2Shell 就是 Server Function 的漏洞。

**常见追问**

- proxy 中检查了登录，Action 中还要检查吗？→ 要。proxy 的 matcher 可能不覆盖、可能被绕过（CVE-2025-29927），鉴权应该在离数据最近的地方执行。

---

### Q17. 怎么实现乐观更新？

- **难度**: 中级
- **标签**: Server Actions, 用户体验

**答案**

```tsx
'use client'

import { useOptimistic, useTransition } from 'react'
import { toggleLike } from './actions'

export function LikeButton({ postId, liked, count }: { postId: string; liked: boolean; count: number }) {
  const [optimistic, setOptimistic] = useOptimistic(
    { liked, count },
    (state, nextLiked: boolean) => ({ liked: nextLiked, count: state.count + (nextLiked ? 1 : -1) }),
  )
  const [isPending, startTransition] = useTransition()

  function handleClick() {
    startTransition(async () => {
      setOptimistic(!optimistic.liked)   // 立即更新 UI
      await toggleLike(postId)            // Action 中失效缓存，服务端返回最新数据
    })
  }

  return (
    <button onClick={handleClick} aria-pressed={optimistic.liked}>
      ♥ {optimistic.count}
    </button>
  )
}
```

- `useOptimistic` 的乐观值只在 transition **进行中**有效
- transition 结束后，组件使用服务端返回的最新 props（真实数据）
- Action 失败时，乐观值自动丢弃，UI 回到真实状态（需要额外提示用户）

**口述模板（30–60 秒）**

1. 一句话结论：用 useOptimistic 在 Action 执行期间显示预期的结果。
2. 关键机制：乐观值只在 transition 进行中生效，结束后使用服务端的真实数据。
3. 对比或注意点：失败时自动回到真实状态，需要提示用户。
4. 可追问时补充：Action 中要失效缓存，确保返回的是最新数据。

**常见追问**

- 连续快速点击会怎样？→ Action 按顺序执行，乐观更新会基于当前的乐观状态叠加；可以在 pending 时禁用按钮或做防抖。

---

## 六、路由与中间层

### Q18. proxy 适合做什么？不适合做什么？

- **难度**: 中级
- **标签**: proxy

**答案**

**适合**：

- 基于路径、Cookie、请求头的**快速**重定向和重写
- 国际化：根据 `Accept-Language` 或 Cookie 重定向到语言前缀
- A/B 测试：根据 Cookie 分组，重写到不同的页面变体
- 多租户：根据域名或子域名重写到租户路由
- **乐观的**登录检查：没有 session Cookie 直接跳转登录页
- 添加安全响应头（CSP nonce）

**不适合**：

- 作为**唯一的**鉴权和授权（应该在数据访问层再次检查）
- 慢的数据查询（对每个匹配的请求都执行）
- 复杂的业务逻辑

**CVE-2025-29927 的教训**：

- 攻击者通过伪造内部请求头 `x-middleware-subrequest` 让 Next.js 跳过 middleware 执行
- 只在 middleware 中做鉴权的应用，受保护的页面被直接访问
- 修复：升级到修复版本（14.2.25、15.2.3 等）；在网关层剥离该请求头；**纵深防御**：鉴权放在数据层

**口述模板（30–60 秒）**

1. 一句话结论：proxy 适合快速的重定向、重写、国际化和乐观的登录检查。
2. 关键机制：对每个匹配的请求都执行，要保持轻量。
3. 对比或注意点：不能作为唯一的鉴权手段，CVE-2025-29927 就是绕过了 middleware。
4. 可追问时补充：鉴权应该在数据访问层再次检查，形成纵深防御。

**常见追问**

- 用 proxy 实现 A/B 测试怎么避免缓存问题？→ 重写到不同的 URL（每个变体可以独立缓存），而不是在同一个 URL 上返回不同的内容。

---

### Q19. App Router 中怎么实现鉴权？

- **难度**: 中级
- **标签**: 鉴权

**答案**

**分层鉴权**：

```text
proxy：            乐观检查（有没有 session Cookie），快速重定向到登录页
布局 / 页面：       读取 session，未登录时 redirect（注意：布局在导航时不重新渲染，不能只依赖布局检查）
数据访问层（DAL）： 每个数据函数都检查 session 和权限 ← 核心
Server Actions：   每个 Action 都检查
Route Handlers：   每个处理函数都检查
```

**数据访问层示例**：

```ts
// lib/dal.ts
import 'server-only'
import { cache } from 'react'
import { cookies } from 'next/headers'

// cache 让同一次渲染中的多次调用只验证一次
export const verifySession = cache(async () => {
  const token = (await cookies()).get('session')?.value
  const session = token ? await decryptSession(token) : null
  if (!session) redirect('/login')
  return session
})

export async function getMyOrders() {
  const session = await verifySession()
  return db.order.findMany({
    where: { userId: session.userId },
    select: { id: true, total: true, status: true },  // 只返回需要的字段
  })
}
```

**Session 方案**：无状态（加密的 JWT Cookie，使用 `jose`）或数据库 session；Cookie 设置 `HttpOnly`、`Secure`、`SameSite=Lax`。

**库**：Auth.js（NextAuth）、Better Auth、Clerk 等。

**口述模板（30–60 秒）**

1. 一句话结论：鉴权要分层，proxy 只做乐观检查，核心在数据访问层。
2. 关键机制：数据访问层的每个函数都验证 session 和权限，只返回需要的字段。
3. 对比或注意点：布局在导航时不重新渲染，不能只在布局中检查。
4. 可追问时补充：用 React 的 cache 让同一次渲染中只验证一次。

**常见追问**

- 为什么不能只在布局中检查登录？→ 布局在客户端导航时不会重新渲染，而且页面和布局是并行渲染的，页面中的数据请求不受布局检查的保护。

---

### Q20. 怎么实现国际化（i18n）？

- **难度**: 中级
- **标签**: 国际化

**答案**

**路由结构**：

```text
app/[lang]/layout.tsx
app/[lang]/page.tsx
app/[lang]/products/[id]/page.tsx
```

**proxy 中检测语言**：

```ts
export function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl
  const hasLocale = locales.some((locale) => pathname.startsWith(`/${locale}/`) || pathname === `/${locale}`)
  if (hasLocale) return
  // 优先级：Cookie > Accept-Language > 默认语言
  const locale = request.cookies.get('locale')?.value ?? matchLocale(request.headers.get('accept-language'))
  request.nextUrl.pathname = `/${locale}${pathname}`
  return NextResponse.redirect(request.nextUrl)
}
```

**字典**：服务端组件中按语言加载字典（`import(`./dictionaries/${lang}.json`)`），**字典不进入客户端包**；客户端组件需要的文案通过 props 传入，或者只传入部分字典。

**静态生成**：`generateStaticParams` 返回所有语言。

**SEO**：`alternates.languages`（hreflang）、`<html lang>`。

**库**：next-intl 等。

**口述模板（30–60 秒）**

1. 一句话结论：用 [lang] 动态段组织路由，proxy 检测语言并重定向。
2. 关键机制：字典在服务端组件中按语言加载，不进入客户端包。
3. 对比或注意点：客户端组件只传入需要的文案。
4. 可追问时补充：设置 hreflang 和 html lang 改善 SEO。

**常见追问**

- 为什么语言要放在 URL 中而不是只用 Cookie？→ URL 中的语言可以被搜索引擎索引、可以分享、可以被 CDN 分别缓存。

---

## 七、工程实践

### Q21. 客户端状态管理在 App Router 中怎么做？

- **难度**: 中级
- **标签**: 状态管理

**答案**

**原则**：服务端数据优先由服务端组件获取，客户端只管理**真正的客户端状态**。

| 状态 | 方案 |
|------|------|
| 服务端数据 | 服务端组件获取；客户端需要轮询或无限加载时使用 TanStack Query / SWR |
| URL 状态（筛选、分页、标签页） | `searchParams`（可分享、服务端可读）；nuqs 等库 |
| 全局 UI 状态（主题、侧边栏） | Context 或 Zustand |
| 表单状态 | `useActionState`、React Hook Form |

**在 App Router 中使用 Zustand 等全局 store 的注意事项**：

- **不要**定义模块级的全局 store：服务端是多请求共享的，会导致跨请求的状态泄露
- 每个请求创建 store：在客户端组件的 Provider 中用 `useRef` / `useState` 创建，通过 Context 提供
- 服务端组件**不能**读取或写入客户端 store

**口述模板（30–60 秒）**

1. 一句话结论：服务端数据由服务端组件获取，客户端只管理真正的客户端状态。
2. 关键机制：URL 状态用 searchParams，UI 状态用 Context 或 Zustand。
3. 对比或注意点：不要定义模块级的 store，服务端会跨请求共享。
4. 可追问时补充：在 Provider 中为每个请求创建 store 实例。

**常见追问**

- 为什么筛选条件推荐放 URL？→ 服务端组件可以读取 searchParams 直接获取筛选后的数据；可以分享、可以刷新保留、浏览器前进后退正常。

---

### Q22. `after()` 是什么？

- **难度**: 中级
- **标签**: API

**答案**

`after()`（15.1 起稳定）在**响应发送给用户之后**执行任务，不阻塞响应：

```tsx
import { after } from 'next/server'

export default async function Page() {
  const data = await getData()
  after(async () => {
    // 不影响页面的响应时间
    await logPageView()
    await syncAnalytics()
  })
  return <View data={data} />
}
```

- 可以在服务端组件、Server Action、Route Handler、proxy 中使用
- 适合日志、统计、缓存预热、发送通知
- 在 Serverless 平台上，平台需要支持在响应后继续执行（`waitUntil`）

**口述模板（30–60 秒）**

1. 一句话结论：after 在响应发送后执行任务，不阻塞用户。
2. 关键机制：可以在组件、Action、Route Handler 和 proxy 中使用。
3. 对比或注意点：Serverless 平台需要支持响应后继续执行。
4. 可追问时补充：适合日志、统计、通知等非关键任务。

**常见追问**

- 为什么不直接不 await 一个 Promise？→ Serverless 环境中响应发送后函数可能被冻结，未完成的 Promise 不会执行；after 会通知平台延长执行时间。

---

### Q23. 怎么做可观测性（日志、追踪、错误上报）？

- **难度**: 中级
- **标签**: 可观测性

**答案**

**`instrumentation.ts`**：服务启动时执行一次，用于初始化监控：

```ts
// instrumentation.ts
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    await import('./instrumentation.node')  // 初始化 OpenTelemetry
  }
}

// 捕获服务端错误（服务端组件、Route Handler、Server Action、proxy）
export async function onRequestError(error: unknown, request: { path: string; method: string }, context: { routerKind: string; routeType: string }) {
  await reportError(error, { path: request.path, routeType: context.routeType })
}
```

**`instrumentation-client.ts`**：客户端初始化（性能监控、错误监控），在应用代码之前执行。

**OpenTelemetry**：Next.js 内置了渲染、fetch、路由等的追踪 span；配合 `@vercel/otel` 或自己配置导出器。

**客户端**：`useReportWebVitals` 上报核心网页指标；error.tsx 中上报客户端错误。

**口述模板（30–60 秒）**

1. 一句话结论：服务端用 instrumentation.ts 初始化监控和捕获错误，客户端用 instrumentation-client.ts。
2. 关键机制：onRequestError 统一捕获服务端各种入口的错误。
3. 对比或注意点：Next.js 内置了 OpenTelemetry 的追踪 span。
4. 可追问时补充：客户端用 useReportWebVitals 上报性能指标。

**常见追问**

- 生产环境服务端错误为什么只看到 digest？→ 为了不把错误详情泄露给用户；用 digest 在服务端日志或 onRequestError 上报的数据中查找对应的完整错误。

---

### Q24. 怎么分析和减小客户端包体积？

- **难度**: 中级
- **标签**: 性能

**答案**

**分析**：`@next/bundle-analyzer`（16.x 中有新的内置分析工具）；`next build` 输出每个路由的首次加载 JS 大小。

**减小**：

- **保持组件在服务端**：把 `'use client'` 下推到叶子组件；重型的渲染库（Markdown、语法高亮、日期格式化）放在服务端组件中
- **动态导入**：

```tsx
import dynamic from 'next/dynamic'
const Chart = dynamic(() => import('./chart'), { ssr: false, loading: () => <Skeleton /> })
```

- `optimizePackageImports`：大型图标库和工具库按需导入
- 替换大体积依赖（moment → date-fns / Temporal）
- 第三方脚本使用 `next/script` 延迟加载
- 避免在客户端组件中导入只在服务端使用的工具

**口述模板（30–60 秒）**

1. 一句话结论：最有效的方式是让更多组件保持在服务端，其次是动态导入和按需导入。
2. 关键机制：服务端组件的依赖不进入客户端包。
3. 对比或注意点：'use client' 下推到叶子组件。
4. 可追问时补充：用 bundle analyzer 找到大依赖。

**常见追问**

- `dynamic` 的 `ssr: false` 能在服务端组件中使用吗？→ 不能，需要在客户端组件中使用。

---

### Q25. 从 Next.js 14 升级到 15 有哪些主要变化？

- **难度**: 中级
- **标签**: 升级

**答案**

| 变化 | 影响 | 处理 |
|------|------|------|
| React 19 | `useFormState` → `useActionState`；ref 可以作为 prop；部分 API 移除 | 按 React 19 升级指南修改 |
| **请求 API 异步化** | `cookies()`、`headers()`、`draftMode()`、`params`、`searchParams` 返回 Promise | `await`；官方 codemod：`npx @next/codemod@canary upgrade` |
| **fetch 默认不缓存** | 依赖默认缓存的页面变成每次请求 | 需要缓存的显式设置 `cache: 'force-cache'` |
| GET Route Handler 默认不缓存 | 同上 | `export const dynamic = 'force-static'` |
| 客户端路由缓存页面段 `staleTime` 为 0 | 导航时重新获取页面 | `experimental.staleTimes` 调整 |
| `next/font`、`@next/font` | `@next/font` 移除 | 改为 `next/font` |

**口述模板（30–60 秒）**

1. 一句话结论：15 的主要变化是 React 19、请求 API 异步化和缓存默认值改变。
2. 关键机制：cookies、headers、params 等需要 await。
3. 对比或注意点：fetch 和 GET Route Handler 默认不再缓存。
4. 可追问时补充：官方提供 codemod 自动处理大部分修改。

**常见追问**

- 升级后页面变慢了可能是什么原因？→ 原来依赖默认的 fetch 缓存，升级后每次请求都访问数据源；需要显式配置缓存。

---

### Q26. 从 Next.js 15 升级到 16 有哪些主要变化？

- **难度**: 中级
- **标签**: 升级

**答案**

| 变化 | 处理 |
|------|------|
| **Turbopack 成为默认** | 有自定义 webpack 配置的项目需要迁移，或使用 `next build --webpack` |
| **`middleware.ts` → `proxy.ts`** | 重命名文件，导出函数改名为 `proxy`；proxy 运行在 Node.js 运行时 |
| 请求 API **只支持异步**访问 | 15 中的同步兼容被移除 |
| `revalidateTag` 需要第二个参数 | `revalidateTag(tag, 'max')`；需要立即生效用 `updateTag` |
| 并行路由必须有 `default.tsx` | 补充 default 文件 |
| `images.domains` 废弃 | 改用 `remotePatterns` |
| 移除 AMP、`next lint` | lint 直接使用 ESLint / Biome |
| Node.js 20.9+、TypeScript 5.1+ | 升级运行环境 |
| 新特性：Cache Components、React Compiler 支持稳定、React 19.2 | 按需开启 |

**口述模板（30–60 秒）**

1. 一句话结论：16 的主要变化是 Turbopack 默认、middleware 更名为 proxy、Cache Components。
2. 关键机制：请求 API 只支持异步，revalidateTag 需要缓存配置参数。
3. 对比或注意点：有自定义 webpack 配置的项目需要特别注意。
4. 可追问时补充：Cache Components 是可选的，但 16.4 起官方推荐所有应用迁移。

**常见追问**

- 为什么 proxy 改为 Node.js 运行时？→ Edge 运行时的 API 限制给开发者带来困扰；Node.js 运行时可以使用完整的 Node API 和生态。

---

## 八、其他

### Q27. `generateStaticParams` 和动态路由的静态生成是怎么工作的？

- **难度**: 中级
- **标签**: 静态生成

**答案**

```tsx
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await getPopularPosts(100)  // 只预渲染热门的 100 篇
  return posts.map((post) => ({ slug: post.slug }))
}

export const dynamicParams = true  // 默认：其他 slug 在首次请求时渲染并缓存
```

- **构建时**预渲染返回的参数组合
- 未返回的参数：`dynamicParams = true` 时首次请求按需渲染并缓存（旧模型）；`false` 时返回 404
- 嵌套动态段：父级的 `generateStaticParams` 返回的参数会传给子级
- 构建时 `generateStaticParams` 中的 fetch 请求会自动去重

**大型站点策略**：只预渲染热门页面，其余按需生成，避免构建时间过长。

**口述模板（30–60 秒）**

1. 一句话结论：generateStaticParams 在构建时返回需要预渲染的参数组合。
2. 关键机制：未返回的参数默认在首次请求时渲染并缓存。
3. 对比或注意点：dynamicParams 设为 false 时，未预渲染的参数返回 404。
4. 可追问时补充：大型站点只预渲染热门页面，控制构建时间。

**常见追问**

- 开启 Cache Components 后还需要它吗？→ 需要，它仍然用于告诉框架有哪些参数需要在构建时预渲染。

---

### Q28. 怎么处理需要浏览器 API 的第三方库？

- **难度**: 中级
- **标签**: 客户端组件

**答案**

| 场景 | 方案 |
|------|------|
| 组件在渲染时访问 `window` | 包装为客户端组件，在 useEffect 中初始化 |
| 库在**导入时**就访问 `window`（模块顶层） | `dynamic(() => import('./map'), { ssr: false })` |
| 只在交互时使用 | 事件处理中动态 `import()` |
| 需要检测是否在客户端 | `useSyncExternalStore` 或 useEffect 中设置 mounted 状态 |

```tsx
'use client'
import dynamic from 'next/dynamic'

const Map = dynamic(() => import('./leaflet-map'), {
  ssr: false,
  loading: () => <div style={{ height: 400 }}>地图加载中...</div>,
})
```

**注意**：`ssr: false` 的组件不参与服务端渲染，loading 占位要设置尺寸，避免 CLS。

**口述模板（30–60 秒）**

1. 一句话结论：渲染时访问浏览器 API 的放到 useEffect，导入时访问的用 ssr: false 动态导入。
2. 关键机制：ssr: false 让组件只在客户端加载和渲染。
3. 对比或注意点：占位要预留尺寸，避免布局偏移。
4. 可追问时补充：只在交互时使用的库，在事件中动态 import。

**常见追问**

- 为什么 `typeof window` 判断会导致 hydration 问题？→ 服务端和客户端首次渲染的分支不同，输出不一致。

---

### Q29. App Router 中怎么测试？

- **难度**: 中级
- **标签**: 测试

**答案**

| 类型 | 工具 | 说明 |
|------|------|------|
| 单元测试 | Vitest / Jest + Testing Library | 客户端组件、同步的服务端组件、工具函数 |
| **async 服务端组件** | **端到端测试**为主 | 单元测试工具对 async 服务端组件的支持有限 |
| Server Actions | 作为普通 async 函数测试（mock 数据库和 session） | 重点测试鉴权和校验分支 |
| Route Handlers | 直接调用导出的 GET / POST，传入 `Request` | |
| 端到端 | Playwright | 完整的渲染、导航、表单、缓存行为 |

**Server Action 测试示例**：

```ts
vi.mock('@/lib/session', () => ({ getSession: vi.fn() }))

it('未登录时拒绝删除', async () => {
  vi.mocked(getSession).mockResolvedValue(null)
  await expect(deletePost('id')).rejects.toThrow('未登录')
})
```

**口述模板（30–60 秒）**

1. 一句话结论：客户端组件用单元测试，async 服务端组件以端到端测试为主。
2. 关键机制：Server Action 和 Route Handler 可以作为普通函数测试。
3. 对比或注意点：重点测试 Action 的鉴权和校验分支。
4. 可追问时补充：缓存和导航行为用 Playwright 在生产构建上测试。

**常见追问**

- 为什么端到端测试要在生产构建上跑？→ 开发模式没有缓存和预取，与生产行为不同。

---

### Q30. Next.js 和 Nuxt、Remix（React Router）有什么区别？

- **难度**: 中级
- **标签**: 对比

**答案**

| 对比项 | Next.js | Nuxt | React Router v7（原 Remix） |
|--------|---------|------|-----------------------------|
| UI 框架 | React | Vue | React |
| 服务端组件 | RSC（核心） | 服务端组件 / Islands（实验） | 支持 RSC（较新） |
| 数据获取 | 服务端组件中直接获取 | `useFetch` / `useAsyncData` | `loader`（路由级） |
| 数据变更 | Server Actions | 服务端 API（`server/api`） | `action`（路由级） |
| 缓存 | Cache Components / 多层缓存 | routeRules（SWR、ISR） | 依赖 HTTP 缓存 |
| 构建 | Turbopack | Vite | Vite |
| 部署 | Vercel 最佳，自托管需要额外配置 | Nitro，适配多个平台 | 适配多个平台 |
| 哲学 | 框架承担更多（缓存、优化） | 约定优于配置 | 贴近 Web 标准（HTTP、表单） |

**口述模板（30–60 秒）**

1. 一句话结论：三者都是全栈框架，Next.js 以 RSC 为核心，Nuxt 面向 Vue，React Router 贴近 Web 标准。
2. 关键机制：Next.js 在组件中获取数据，React Router 在路由级的 loader 中获取。
3. 对比或注意点：Next.js 的缓存最复杂也最强大；React Router 主要依赖 HTTP 缓存。
4. 可追问时补充：Nuxt 基于 Nitro，部署适配性好。

**常见追问**

- 怎么选型？→ 技术栈（React / Vue）、对 RSC 的需求、部署环境（是否用 Vercel）、团队对缓存复杂度的接受程度。

---

### Q31. React 19 / 19.2 中与 Next.js 相关的新能力有哪些？

- **难度**: 中级
- **标签**: React

**答案**

| 特性 | 版本 | 在 Next.js 中的用途 |
|------|------|---------------------|
| Server Components / Server Functions 稳定 | 19 | App Router 的基础 |
| `useActionState`、`useFormStatus`、`useOptimistic` | 19 | 表单与 Server Actions |
| `use()` | 19 | 读取 Promise（流式传给客户端组件）和 Context |
| ref 作为 prop、`<Context>` 作为 Provider | 19 | 简化代码 |
| 文档元数据（`<title>` 等自动提升到 head） | 19 | 组件中声明元数据 |
| 资源预加载 API（`preload`、`preinit`） | 19 | 精细控制资源加载 |
| **View Transitions** | 19.2（Canary） / 19.3 稳定（以官方为准） | 路由切换动画 |
| **`<Activity>`** | 19.2 | 隐藏的 UI 保留状态（类似 KeepAlive），Next.js 16 在导航中使用它保留路由状态 |
| `useEffectEvent` | 19.2 | 在 effect 中读取最新值而不作为依赖 |
| React Compiler 1.0 | 2025-10 | 自动 memo，Next.js 16 中 `reactCompiler: true` |

**口述模板（30–60 秒）**

1. 一句话结论：React 19 稳定了 Server Components 和表单相关的 Hooks，19.2 加入 Activity、View Transitions 等。
2. 关键机制：useActionState 和 useOptimistic 配合 Server Actions；use() 读取流式传递的 Promise。
3. 对比或注意点：React Compiler 自动 memo，减少手动优化。
4. 可追问时补充：Next.js 16 使用 Activity 在导航中保留路由状态。

**常见追问**

- React Compiler 开启后还需要 `useMemo` 吗？→ 大部分情况不需要；编译器自动缓存计算结果和回调，但仍需遵守 React 的规则（纯渲染）。

---

### Q32. 什么时候不应该使用 Next.js？

- **难度**: 中级
- **标签**: 选型

**答案**

- **纯客户端的内部工具 / 后台**：没有 SEO 需求，用户都在内网 → Vite + React（或 TanStack Router）更简单，没有服务端运维成本
- **团队不熟悉服务端**：RSC 和缓存模型的学习成本高，出问题时排查困难
- **部署环境受限**：只能部署静态文件 → 可以用 `output: 'export'`，但失去大部分优势
- **对框架锁定敏感**：部分特性在 Vercel 之外需要额外的工作（缓存共享、图片优化、ISR）
- **需要极致的控制**：自定义服务端、非 React 的渲染需求

**适合**：内容站、电商、营销站点、SaaS 应用（需要 SEO、首屏性能、全栈开发）。

**口述模板（30–60 秒）**

1. 一句话结论：纯客户端的内部工具、只能静态部署的环境、团队不熟悉服务端时不一定适合。
2. 关键机制：Next.js 的价值在服务端渲染、SEO 和全栈能力。
3. 对比或注意点：自托管时部分特性需要额外的工作。
4. 可追问时补充：内容站、电商、SaaS 是最适合的场景。

**常见追问**

- 已经用 Next.js 的后台系统，要不要迁走？→ 评估实际成本：如果运维和复杂度带来了问题再考虑；可以只使用客户端组件和简单的路由，不使用复杂的缓存特性。
