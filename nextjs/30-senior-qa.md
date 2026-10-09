# Next.js 高级 Q&A

> 目标：深入 Cache Components、PPR、RSC 协议、自托管与多实例部署、安全事件、大型应用架构与迁移；能讲清取舍和线上问题的排查思路。

---

## 一、Cache Components 与 PPR 深入

### Q1. 开启 Cache Components 后，构建时是怎么区分静态和动态的？

- **难度**: 高级
- **标签**: Cache Components, PPR

**答案**

**构建时预渲染**：Next.js 渲染每个路由，遇到以下情况时的处理：

| 遇到的内容 | 处理 |
|------------|------|
| 纯计算、同步 I/O、模块导入 | 进入静态外壳 |
| `'use cache'` 的组件 / 函数 | 执行并缓存结果，进入静态外壳 |
| 访问请求数据（cookies、headers、searchParams、params 未提供）或未缓存的异步 I/O（数据库查询、fetch） | **必须**被 Suspense 包裹 → fallback 进入外壳，内容在请求时渲染 |
| 上述情况但**没有** Suspense 包裹 | **构建报错**：要求你明确选择缓存或流式 |

**判断原理**：框架在预渲染时让渲染「在一个任务内完成」，在此期间没有完成的 Promise（例如未缓存的数据库查询）被视为动态，由最近的 Suspense 边界接管。

**设计意图**：强制开发者对每一段数据做出**显式选择**：

- 「可以缓存」→ `'use cache'`
- 「必须实时」→ Suspense 流式
- 不存在「意外地变成动态」或「意外地被缓存」

**口述模板（30–60 秒）**

1. 一句话结论：构建时，缓存的内容和同步内容进入静态外壳，未缓存的异步操作和请求数据必须由 Suspense 包裹。
2. 关键机制：预渲染中未完成的 Promise 被视为动态，由最近的 Suspense 处理。
3. 对比或注意点：没有 Suspense 包裹的动态访问会导致构建报错。
4. 可追问时补充：设计意图是强制开发者对每段数据显式选择缓存或流式。

**常见追问**

- 整个页面都需要用户数据怎么办？→ 用户相关部分放在 Suspense 中；页面的框架（导航、布局）作为外壳；或者在根层级使用 Suspense，外壳只包含 fallback。

---

### Q2. 怎么设计一个页面的缓存策略？

- **难度**: 高级
- **标签**: 缓存, 架构

**答案**

**以电商商品详情页为例，按数据的特性分类**：

| 数据 | 变化频率 | 个性化 | 策略 |
|------|----------|--------|------|
| 布局、导航、页脚 | 很少 | 否 | 静态（`'use cache'` + `cacheLife('max')` 或纯静态） |
| 商品信息、图片、描述 | 低（后台修改） | 否 | `'use cache'` + `cacheTag('product-123')` + `cacheLife('days')`，修改时 `revalidateTag` |
| 价格 | 中（促销） | 否 | `cacheLife('minutes')` + 标签 |
| 库存 | 高 | 否 | Suspense 流式，或 `cacheLife('seconds')` |
| 购物车、推荐、会员价 | 实时 | 是 | Suspense 流式；或 `'use cache: private'` |
| 评论 | 中 | 否 | 缓存 + 标签，评论提交后 `updateTag` |

**原则**：

- 按「**变化频率 × 是否个性化**」划分组件边界，而不是按 UI 结构
- 个性化数据**不能**进入共享缓存
- 每个缓存都要有**失效路径**（标签 + 谁触发失效）
- 用标签的粒度平衡失效精度和复杂度：`product-123`（单个）、`products`（列表）、`category-x`

**口述模板（30–60 秒）**

1. 一句话结论：按数据的变化频率和是否个性化划分组件，分别选择缓存时长或流式。
2. 关键机制：不变的内容静态化，低频变化的内容缓存加标签，个性化和实时的内容流式。
3. 对比或注意点：个性化数据不能进入共享缓存；每个缓存都要有失效路径。
4. 可追问时补充：标签的粒度在失效精度和复杂度之间权衡。

**常见追问**

- 一个商品修改后，商品详情页和列表页都要更新怎么办？→ 详情页的缓存打 `product-123` 和 `products` 两个标签（或在列表上打 `products`），修改时失效两个标签。

---

### Q3. Cache Components 中的缓存存储在哪里？多实例部署怎么办？

- **难度**: 高级
- **标签**: 缓存, 部署

**答案**

**默认存储**：

- `'use cache'`：服务端**内存 LRU**（每个实例独立）+ 构建时预渲染的结果
- 客户端：路由缓存（由 `stale` 控制）

**多实例的问题**：

- 实例 A 失效了标签，实例 B 的内存中仍然是旧数据
- 每个实例各自缓存一份，命中率低、回源压力大

**解决方案**：

```ts
// next.config.ts
const nextConfig: NextConfig = {
  cacheComponents: true,
  cacheHandlers: {
    // 'use cache: remote' 使用的处理器
    remote: require.resolve('./cache-handlers/redis.mjs'),
  },
}
```

- 对需要跨实例共享的缓存使用 `'use cache: remote'`，配置 Redis 等共享存储的 `cacheHandlers`
- 标签失效也需要通过共享存储传播（处理器实现 `expireTags` 等方法）
- 旧模型（ISR / Data Cache）使用 `cacheHandler`（单数）配置，原理相同

**取舍**：

| 存储 | 延迟 | 一致性 | 成本 |
|------|------|--------|------|
| 内存 | 最低 | 实例之间不一致 | 低 |
| 远程（Redis） | 网络往返 | 一致 | 额外的服务 |

**口述模板（30–60 秒）**

1. 一句话结论：默认缓存在每个实例的内存中，多实例部署需要配置共享的缓存处理器。
2. 关键机制：用 use cache: remote 配合 Redis 等 cacheHandlers，标签失效也通过共享存储传播。
3. 对比或注意点：内存最快但不一致，远程一致但有网络开销。
4. 可追问时补充：可以混合使用，高频的小数据用内存，需要一致性的用远程。

**常见追问**

- 不配共享缓存，多实例会出什么问题？→ 用户刷新时可能落到不同实例，看到新旧数据交替出现；失效只在处理该请求的实例上生效。

---

### Q4. Partial Prefetching 是什么？和旧的 Link 预取有什么区别？

- **难度**: 高级
- **标签**: 预取, 导航

**答案**

**旧模型的预取**：

- 静态路由：预取整个路由的 RSC Payload
- 动态路由：只预取到最近的 `loading.tsx` 边界
- 问题：要么预取太多（浪费），要么太少（导航时仍然要等待）

**Partial Prefetching（16.4 中作为新应用的默认配置之一）**：

- 预取目标路由的**静态外壳**（基于 PPR / Cache Components）
- 导航时立即显示外壳，动态部分流式到达
- 共享的布局只下载一次（布局去重）
- 预取请求是增量的：已经缓存的段不再重复获取

**效果**：几乎所有导航都能立即显示有意义的内容（不只是 loading），同时不预取个性化数据。

**口述模板（30–60 秒）**

1. 一句话结论：Partial Prefetching 预取目标路由的静态外壳，导航时立即显示，动态部分流式到达。
2. 关键机制：基于 PPR，共享布局去重，只预取未缓存的段。
3. 对比或注意点：旧模型的动态路由只能预取到 loading 边界。
4. 可追问时补充：16.4 起新应用默认开启。

**常见追问**

- 预取会不会带来大量请求？→ Link 只在进入视口时预取，且布局去重和增量预取减少了数据量；对于非常长的列表可以设置 `prefetch={false}` 或改为悬停时预取。

---

## 二、RSC 协议与底层

### Q5. RSC 的 Flight 协议是怎么工作的？

- **难度**: 高级
- **标签**: RSC, 协议

**答案**

**Flight**是 React 用于序列化 Server Components 输出的协议（RSC Payload 的格式）：

- **按行分块**：每行是一个带 ID 的块（`ID:类型+数据`），可以流式发送
- **引用**：块之间通过 `$ID` 相互引用，支持循环和延迟的内容
- **客户端模块引用**（`I` 类型）：模块 ID、需要加载的 chunk、导出名
- **Promise / Suspense**：先发送占位引用，数据就绪后再发送对应 ID 的块
- **Server Reference**：Server Action 的 ID 和绑定的参数

**客户端解析**：`createFromReadableStream` / `createFromFetch` 逐行解析，构建 React 元素树，遇到客户端模块引用时加载对应的 chunk；未到达的块表现为挂起的 Promise，由 Suspense 处理。

**反方向（客户端 → 服务端）**：Server Action 的参数也通过类似的格式序列化（`encodeReply` / `decodeReply`）。**React2Shell 漏洞正是出在服务端反序列化这些数据的过程中**。

**口述模板（30–60 秒）**

1. 一句话结论：Flight 是一种按行分块、带引用的流式序列化协议。
2. 关键机制：客户端组件以模块引用表示，Promise 先以占位发送，之后填充。
3. 对比或注意点：客户端逐行解析并构建元素树，未到达的部分由 Suspense 处理。
4. 可追问时补充：反方向的 Server Action 参数也用类似格式，React2Shell 就出在服务端反序列化中。

**常见追问**

- 为什么不用 JSON？→ JSON 必须完整才能解析，不支持流式、引用和延迟内容；Flight 可以边接收边渲染。

---

### Q6. 服务端组件中的数据怎么安全地传给客户端组件？

- **难度**: 高级
- **标签**: RSC, 安全

**答案**

**风险**：传给客户端组件的 props 会**完整地**序列化到 RSC Payload 中，在浏览器中可见。

```tsx
// 危险：user 对象包含 passwordHash、email 等敏感字段
const user = await db.user.findUnique({ where: { id } })
return <ProfileCard user={user} />
```

**措施**：

1. **数据传输对象（DTO）**：在数据访问层只返回需要的字段

```ts
export async function getProfileDTO(id: string) {
  const user = await db.user.findUnique({ where: { id }, select: { name: true, avatar: true } })
  return user
}
```

2. **React taint API**（实验性，需要开启 `experimental.taint`）：

```ts
import { experimental_taintObjectReference, experimental_taintUniqueValue } from 'react'

experimental_taintObjectReference('不要把完整的用户对象传给客户端', user)
experimental_taintUniqueValue('不要把 API 密钥传给客户端', process.env, process.env.API_KEY!)
```

taint 是**额外的防护层**，不能替代 DTO。

3. `server-only` 防止数据访问模块被导入到客户端
4. 环境变量不加 `NEXT_PUBLIC_` 前缀

**口述模板（30–60 秒）**

1. 一句话结论：传给客户端组件的 props 会完整出现在浏览器中，必须只传需要的字段。
2. 关键机制：数据访问层返回 DTO，用 select 限制字段。
3. 对比或注意点：React taint API 是额外的防护层，不能替代 DTO。
4. 可追问时补充：server-only 防止数据模块被客户端导入。

**常见追问**

- 服务端组件自己渲染的内容会泄露吗？→ 渲染结果本来就是给用户看的；风险在于传给客户端组件的 props 中包含了没有渲染出来的字段。

---

### Q7. Server Actions 的闭包加密是怎么工作的？有什么局限？

- **难度**: 高级
- **标签**: Server Actions, 安全

**答案**

```tsx
export default async function Page() {
  const secretId = await getInternalId()
  async function action() {
    'use server'
    await doSomething(secretId)   // 闭包捕获 secretId
  }
  return <form action={action}>...</form>
}
```

- 渲染时，捕获的变量被**加密**后随 Action 引用一起发送给客户端
- 调用时，加密的数据发送回服务端，解密后恢复闭包
- 密钥默认在每次构建时生成；**多实例或多次构建**需要设置 `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` 保持一致
- Action ID 在每次构建时也会变化（未使用的 Action 不会被生成端点）

**局限**：

- 加密防止窥探和篡改，但**不能**代替鉴权：用户仍然可以用合法的加密数据重放调用
- 不要依赖闭包传递敏感信息；优先在 Action 中重新读取
- 版本偏差：旧客户端调用新部署的 Action ID 可能找不到（见 Q12）

**口述模板（30–60 秒）**

1. 一句话结论：闭包变量被加密后发送给客户端，调用时传回服务端解密。
2. 关键机制：多实例需要设置统一的加密密钥。
3. 对比或注意点：加密不能代替鉴权，用户可以重放合法的调用。
4. 可追问时补充：敏感数据应在 Action 中重新读取，不依赖闭包。

**常见追问**

- 多实例部署不设置密钥会怎样？→ 实例 A 渲染的页面，Action 请求落到实例 B 时无法解密，调用失败。

---

## 三、性能

### Q8. 一个 App Router 应用 TTFB 很高，怎么排查？

- **难度**: 高级
- **标签**: 性能, 排查

**答案**

**排查步骤**：

1. **确认渲染方式**：`next build` 输出中该路由是静态、动态还是 PPR？意外的动态（某个组件调用了 cookies）是常见原因
2. **定位慢的数据请求**：OpenTelemetry 追踪（Next.js 内置渲染和 fetch 的 span），或在数据层打日志
3. **检查请求瀑布**：

```tsx
// 串行：布局等待用户，页面等待订单
const user = await getUser()
const orders = await getOrders(user.id)
const recommendations = await getRecommendations()  // 不依赖前面的结果，却在等待
```

改为并行：`Promise.all`，或者把独立的部分拆成组件，各自在 Suspense 中获取

4. **检查是否阻塞在 Suspense 之外**：页面顶层 await 慢请求会阻塞整个外壳
5. **数据库和外部服务**：连接池、慢查询、跨区域访问（函数和数据库不在同一区域）
6. **冷启动**（Serverless）：包体积、初始化逻辑
7. **缓存命中率**：多实例时是否每个实例都在回源

**优化方向**：

- 慢的部分放进 Suspense，外壳立即返回
- 缓存可以缓存的数据
- 并行请求
- 函数部署在数据库附近

**口述模板（30–60 秒）**

1. 一句话结论：先确认路由是否意外变成动态，再用追踪定位慢请求和瀑布。
2. 关键机制：串行的数据请求改为并行，慢的部分放进 Suspense。
3. 对比或注意点：页面顶层的慢请求会阻塞整个外壳。
4. 可追问时补充：检查数据库和函数是否在同一区域、多实例的缓存命中率。

**常见追问**

- 怎么避免布局和页面之间的瀑布？→ 布局和页面是并行渲染的；但嵌套布局中各自 await 的数据仍然是各自的，需要注意不要在父组件中 await 子组件不需要等待的数据。

---

### Q9. 怎么避免请求瀑布？有哪些模式？

- **难度**: 高级
- **标签**: 性能, 数据获取

**答案**

| 模式 | 写法 | 适用 |
|------|------|------|
| `Promise.all` | 同一组件中并行启动 | 都需要等待才能渲染 |
| **组件拆分 + Suspense** | 每个组件获取自己的数据，各自流式 | 各部分可以独立显示 |
| **预加载（preload）** | 提前启动请求，不 await | 父组件知道子组件需要什么数据 |
| **Promise 传给客户端组件** | 服务端组件启动请求，把 Promise 传下去，客户端用 `use()` 读取 | 客户端组件需要数据，但不想阻塞外壳 |

**预加载模式**：

```ts
// lib/data.ts
import 'server-only'
import { cache } from 'react'

export const getProduct = cache(async (id: string) => db.product.findUnique({ where: { id } }))
export const preloadProduct = (id: string) => {
  void getProduct(id)  // 启动但不等待，React cache 保证后续调用复用同一个 Promise
}
```

```tsx
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  preloadProduct(id)           // 立即启动
  const allowed = await checkPermission(id)  // 同时进行的另一个请求
  if (!allowed) notFound()
  return <ProductDetails id={id} />  // 内部 getProduct(id) 复用已经启动的请求
}
```

**口述模板（30–60 秒）**

1. 一句话结论：避免瀑布的方式有并行请求、组件拆分配合 Suspense、预加载、把 Promise 传给客户端组件。
2. 关键机制：预加载利用 React cache，提前启动的请求被后续调用复用。
3. 对比或注意点：只在真正有依赖关系时才串行。
4. 可追问时补充：传 Promise 给客户端组件可以不阻塞外壳。

**常见追问**

- `React.cache` 和 `'use cache'` 有什么区别？→ `React.cache` 只在单次请求的渲染中去重，不跨请求；`'use cache'` 跨请求持久化缓存。

---

### Q10. 大型 Next.js 应用的构建速度怎么优化？

- **难度**: 高级
- **标签**: 工程化, 构建

**答案**

- **Turbopack**（16 默认）：增量计算，构建速度明显快于 webpack；16.1 起开发环境的文件系统缓存稳定（重启后复用）
- **减少构建时的预渲染**：`generateStaticParams` 只返回热门页面，其余按需生成
- **构建时数据请求**：并发限制、缓存（构建时的 fetch 会去重）；避免每个页面都请求大量数据
- **monorepo**：Turborepo / Nx 的远程缓存，未改动的包不重新构建
- **类型检查和 lint 分离**：CI 中并行运行，`next build` 中跳过（`typescript.ignoreBuildErrors` 慎用，必须保证 CI 中另外检查）
- **依赖**：`optimizePackageImports`、避免在构建时执行重型脚本

**口述模板（30–60 秒）**

1. 一句话结论：使用 Turbopack、减少构建时的预渲染、利用 monorepo 远程缓存。
2. 关键机制：generateStaticParams 只预渲染热门页面，其余按需生成。
3. 对比或注意点：类型检查和 lint 可以在 CI 中并行运行。
4. 可追问时补充：16.1 起开发环境的 Turbopack 文件系统缓存稳定。

**常见追问**

- 有自定义 webpack loader 的项目怎么迁移到 Turbopack？→ Turbopack 支持部分 webpack loader（通过 `turbopack.rules` 配置）；不支持的插件需要寻找替代方案，或暂时使用 `--webpack`。

---

## 四、部署与运维

### Q11. 自托管 Next.js 需要注意什么？

- **难度**: 高级
- **标签**: 部署

**答案**

| 方面 | 注意事项 |
|------|----------|
| 构建产物 | `output: 'standalone'`，只包含必要的依赖；`public` 和 `.next/static` 需要手动复制或上传到 CDN |
| 静态资源 | `.next/static` 带哈希，可以设置长期缓存；配置 `assetPrefix` 指向 CDN |
| 缓存共享 | 多实例配置 `cacheHandler` / `cacheHandlers`（Redis） |
| Server Actions 密钥 | 多实例设置 `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` |
| 构建 ID | `generateBuildId` 保证多实例使用同一个构建 |
| 图片优化 | 默认使用 sharp，需要 CPU；或配置外部 loader（CDN 图片服务） |
| 流式 | 反向代理关闭缓冲（Nginx `proxy_buffering off` 或 `X-Accel-Buffering: no`） |
| 健康检查与优雅关闭 | 处理 SIGTERM，等待进行中的请求完成 |
| 版本偏差 | 部署期间新旧版本共存（见 Q12） |
| 安全 | 网关层剥离内部请求头（`x-middleware-subrequest` 等）、及时升级 |

**Docker 示例要点**：多阶段构建 → 只复制 standalone 输出、静态文件和 public → 非 root 用户运行 → `node server.js`。

**Deployment Adapters**（16.x 引入的部署适配器 API）：平台可以通过适配器支持 Next.js 的所有特性，减少对 Vercel 的依赖（以官方进展为准）。

**口述模板（30–60 秒）**

1. 一句话结论：自托管使用 standalone 输出，静态资源上 CDN，多实例共享缓存和密钥。
2. 关键机制：多实例需要统一的构建 ID、Server Actions 密钥和共享的缓存处理器。
3. 对比或注意点：反向代理要关闭缓冲，否则流式失效。
4. 可追问时补充：网关层剥离内部请求头，及时升级安全补丁。

**常见追问**

- 图片优化在自托管时有什么问题？→ 图片处理消耗 CPU，高并发时影响页面渲染；可以独立部署或使用 CDN 的图片服务（自定义 loader）。

---

### Q12. 什么是版本偏差（version skew）？怎么处理？

- **难度**: 高级
- **标签**: 部署

**答案**

**问题**：部署新版本时，用户浏览器中运行的仍然是旧版本的客户端代码：

- 旧客户端请求的 JS chunk 在新部署中不存在 → 加载失败
- 旧客户端调用的 Server Action ID 在新版本中不存在 → 调用失败
- 旧客户端请求的 RSC Payload 格式和新版本不兼容

**Next.js 的处理**：

- 检测到部署 ID 变化时，客户端导航退化为**硬导航**（整页刷新加载新版本）
- 配置 `deploymentId`，请求中带上部署 ID

**部署策略**：

- 保留旧版本的静态资源一段时间（CDN 上不删除旧的 `_next/static`）
- 平台的版本偏差保护：把请求路由到与客户端相同版本的部署（Vercel Skew Protection）
- 滚动发布期间，新旧实例共存，要保证兼容性
- Server Action 的接口变更要考虑兼容（新增参数可选）

**口述模板（30–60 秒）**

1. 一句话结论：版本偏差是部署后浏览器中的旧客户端与新服务端不匹配。
2. 关键机制：会导致 chunk 加载失败、Server Action 找不到。
3. 对比或注意点：Next.js 检测到部署 ID 变化时退化为整页刷新。
4. 可追问时补充：保留旧静态资源、路由到同版本部署、保持 Action 接口兼容。

**常见追问**

- 用户反馈「点击按钮没反应」，日志显示 Action 找不到，可能是什么原因？→ 版本偏差：用户的页面是旧版本，调用的 Action ID 在新部署中不存在；或者多实例的构建不一致。

---

### Q13. Serverless 和长驻 Node 服务部署 Next.js 各有什么取舍？

- **难度**: 高级
- **标签**: 部署

**答案**

| 对比项 | Serverless（Vercel、Lambda） | 长驻服务（容器、虚拟机） |
|--------|-----------------------------|----------------------------|
| 扩展 | 自动、按请求 | 需要配置自动扩缩 |
| 冷启动 | 有（Fluid Compute 等方案有所缓解） | 无 |
| 内存缓存 | 实例生命周期短，命中率低 | 有效 |
| 数据库连接 | 容易耗尽连接，需要连接池代理 | 连接池 |
| 后台任务 | 需要 `after()` / `waitUntil` | 可以直接运行 |
| WebSocket | 不支持（需要第三方服务） | 支持 |
| 成本 | 低流量便宜，高流量可能更贵 | 稳定流量更便宜 |
| 运维 | 少 | 多 |

**口述模板（30–60 秒）**

1. 一句话结论：Serverless 扩展方便、运维少，长驻服务没有冷启动、缓存和连接池更有效。
2. 关键机制：Serverless 要注意数据库连接耗尽和后台任务。
3. 对比或注意点：WebSocket 在 Serverless 中需要第三方服务。
4. 可追问时补充：成本取决于流量模式。

**常见追问**

- Serverless 中数据库连接耗尽怎么办？→ 使用连接池代理（PgBouncer、Prisma Accelerate 等）或 HTTP 协议的数据库驱动。

---

## 五、安全

### Q14. 讲讲 CVE-2025-29927（middleware 绕过）

- **难度**: 高级
- **标签**: 安全, 事件

**答案**

**时间**：2025 年 3 月披露，CVSS 9.1。

**原理**：

- Next.js 内部使用请求头 `x-middleware-subrequest` 标记「由 middleware 发起的子请求」，防止 middleware 无限递归
- 框架**信任**了外部请求中的这个头：攻击者在请求中加上这个头（值满足特定条件），middleware 就会被**跳过**
- 只在 middleware 中做鉴权的应用，受保护页面可以被直接访问

**影响**：自托管的 Next.js 11.1.4 至 15.2.2（Vercel、Netlify 等平台上的部署不受影响，因为平台的路由层处理了这个头）。

**修复**：

- 升级到 12.3.5、13.5.9、14.2.25、15.2.3 或更高版本
- 无法立即升级：在网关或反向代理层**剥离**该请求头

**教训**：

- **不要信任可以被外部控制的内部信号**（请求头）
- **纵深防御**：鉴权必须在数据访问层再次检查，不能只依赖 middleware
- 关注框架的安全公告，有升级预案

**口述模板（30–60 秒）**

1. 一句话结论：框架信任了外部请求中的内部请求头，攻击者可以加上这个头跳过 middleware。
2. 关键机制：x-middleware-subrequest 本来用于防止递归，却被外部伪造。
3. 对比或注意点：只在 middleware 中做鉴权的自托管应用受影响。
4. 可追问时补充：修复是升级和在网关剥离请求头；根本教训是鉴权要分层。

**常见追问**

- 如果你的应用受影响，你会怎么应急？→ 立即在网关剥离该请求头 → 升级版本 → 检查访问日志中带有该头的请求，评估是否被利用 → 在数据层补充鉴权。

---

### Q15. 讲讲 React2Shell（CVE-2025-55182）

- **难度**: 高级
- **标签**: 安全, 事件

**答案**

**时间**：2025 年 12 月 3 日披露，CVSS 10.0。

**原理（概要）**：

- React Server Components 在服务端处理 **Server Function（Server Action）请求**时，需要反序列化客户端发来的数据（Flight 格式）
- 反序列化过程中存在缺陷，攻击者构造恶意的请求体，可以在服务端**执行任意代码**
- **不需要鉴权**：只要应用支持 RSC，即使没有定义任何 Server Action 也可能受影响

**影响**：

- React 19.0、19.1、19.2 的 `react-server-dom-*` 包
- 所有基于 RSC 的框架：Next.js（15.x、16.x 的 App Router）、React Router 的 RSC 模式、Waku、`@vitejs/plugin-rsc` 等
- 披露后很快出现了大规模的自动化利用

**修复**：

- React：19.0.1、19.1.2、19.2.1
- Next.js：15.0.5、15.1.9、15.2.6、15.3.6、15.4.8、15.5.7、16.0.7 等（以官方公告为准）
- 升级后**轮换**服务端的密钥（如果怀疑已经被利用）

**教训**：

- RSC 把服务端和客户端的边界变成了一个**序列化协议**，协议解析器本身就是攻击面
- 框架级漏洞影响所有用户：需要依赖监控（Dependabot、Renovate）、快速升级流程
- WAF 规则可以作为临时缓解，但不能替代升级

**口述模板（30–60 秒）**

1. 一句话结论：React2Shell 是 RSC 处理 Server Function 请求时的反序列化漏洞，可以未经鉴权远程执行代码。
2. 关键机制：攻击者构造恶意的 Flight 格式请求体，在服务端反序列化时触发。
3. 对比或注意点：影响所有基于 RSC 的框架，不只是 Next.js。
4. 可追问时补充：修复是升级 React 和框架，怀疑被利用时轮换密钥。

**常见追问**

- 和 CVE-2025-29927 有什么不同？→ 29927 是鉴权绕过，只影响依赖 middleware 鉴权的应用；React2Shell 是远程代码执行，影响几乎所有使用 RSC 的应用，严重程度更高。

---

### Q16. Next.js 应用的安全清单

- **难度**: 高级
- **标签**: 安全

**答案**

| 类别 | 措施 |
|------|------|
| **鉴权** | 数据访问层检查；每个 Server Action、Route Handler 都检查；proxy 只做乐观检查 |
| **输入** | 所有 Action / Route Handler 的参数用 Zod 校验 |
| **数据泄露** | DTO、`server-only`、taint、环境变量前缀 |
| **XSS** | 避免 `dangerouslySetInnerHTML`；必须时用 DOMPurify；CSP（proxy 中生成 nonce） |
| **CSRF** | Server Actions 内置 Origin 检查；Route Handler 处理 POST 时检查 Origin 或使用 token；Cookie `SameSite` |
| **SSRF** | 服务端 fetch 用户提供的 URL 时限制域名；`images.remotePatterns` 严格配置 |
| **开放重定向** | `redirect()` 的目标来自用户输入时校验为站内路径 |
| **响应头** | `headers()` 配置 HSTS、X-Content-Type-Options、Referrer-Policy、frame-ancestors |
| **依赖** | 锁文件、及时升级、监控安全公告 |
| **部署** | 网关剥离内部请求头；密钥管理；生产环境不暴露 source map（或上传到监控平台后删除） |

**CSP nonce 示例要点**：

```ts
// proxy.ts
const nonce = Buffer.from(crypto.randomUUID()).toString('base64')
const csp = `script-src 'self' 'nonce-${nonce}' 'strict-dynamic'; object-src 'none'; base-uri 'none'`
// 设置到请求头（让 Next.js 给内联脚本加上 nonce）和响应头
```

注意：使用 nonce 后页面必须动态渲染（每个请求的 nonce 不同）。

**口述模板（30–60 秒）**

1. 一句话结论：安全清单涵盖鉴权、输入校验、数据泄露、XSS、CSRF、SSRF、响应头、依赖和部署。
2. 关键机制：鉴权在数据层，输入用 Zod，数据用 DTO。
3. 对比或注意点：CSP nonce 需要页面动态渲染。
4. 可追问时补充：网关剥离内部请求头，及时升级依赖。

**常见追问**

- CSP nonce 和静态页面冲突怎么办？→ 静态页面可以使用基于哈希的 CSP（`'sha256-...'`），或者通过 SRI 的实验性支持；需要在性能和安全之间权衡。

---

## 六、迁移与架构

### Q17. 大型项目怎么从 Pages Router 迁移到 App Router？

- **难度**: 高级
- **标签**: 迁移

**答案**

**两种路由可以共存**，按路由逐步迁移：

```text
阶段 1：准备
  - 升级到最新版本，修复 React 19 不兼容的代码
  - 梳理全局逻辑：_app（Provider、布局）、_document、API 路由、数据获取方式
  - 确认依赖库对 RSC 的兼容性

阶段 2：建立基础
  - 创建 app/layout.tsx，迁移 _document 和 _app 中的全局内容
  - Provider 放到客户端组件中
  - 公共组件区分服务端 / 客户端

阶段 3：逐个路由迁移（从简单、低风险的页面开始）
  - getStaticProps / getServerSideProps → 服务端组件中直接获取数据
  - getStaticPaths → generateStaticParams
  - next/router → next/navigation（useRouter、usePathname、useSearchParams）
  - Head → Metadata API
  - API 路由可以保留，或者迁移到 Route Handlers / Server Actions

阶段 4：清理
  - 删除 pages 目录中已迁移的页面
```

**注意**：

- 同一个路径不能同时存在于两个目录中
- 两个路由之间的导航是**硬导航**（整页刷新）：相互链接频繁的页面一起迁移
- 先用客户端组件「平移」（保持原有逻辑），再逐步把数据获取移到服务端

**口述模板（30–60 秒）**

1. 一句话结论：两种路由可以共存，按路由逐步迁移，从简单的页面开始。
2. 关键机制：先建立根布局和 Provider，再迁移数据获取、路由 API 和元数据。
3. 对比或注意点：两种路由之间的导航是整页刷新，相互链接多的页面一起迁移。
4. 可追问时补充：先用客户端组件平移，再逐步把数据获取移到服务端。

**常见追问**

- 迁移中最容易出问题的是什么？→ 第三方库不兼容服务端组件（访问 window、使用 Context）、全局状态管理的初始化方式、`next/router` 的事件（`routeChangeStart`）在 App Router 中没有直接对应。

---

### Q18. 怎么组织大型 App Router 项目的目录结构？

- **难度**: 高级
- **标签**: 架构

**答案**

```text
src/
├── app/                      # 只放路由相关文件，保持很薄
│   ├── (marketing)/          # 路由组：营销页面，独立布局
│   ├── (app)/                # 路由组：应用页面，需要登录
│   │   ├── dashboard/
│   │   │   ├── page.tsx      # 组合 features 中的组件
│   │   │   └── _components/  # 私有文件夹：只有这个路由使用的组件
│   └── api/
├── features/                 # 按业务领域组织
│   ├── orders/
│   │   ├── components/       # 服务端和客户端组件
│   │   ├── actions.ts        # Server Actions
│   │   ├── queries.ts        # 数据访问（server-only）
│   │   └── schemas.ts        # Zod schema（前后端共享）
│   └── products/
├── components/ui/            # 通用 UI 组件
├── lib/                      # 基础设施：db、auth、cache
└── proxy.ts
```

**原则**：

- `app/` 只负责路由和组合，业务逻辑在 `features/`
- 数据访问集中在带 `server-only` 的模块中
- 边界规则用 ESLint（`import/no-restricted-paths`）约束：features 之间不能直接相互导入内部文件

**口述模板（30–60 秒）**

1. 一句话结论：app 目录保持很薄，只负责路由组合，业务按领域放在 features 中。
2. 关键机制：每个领域包含组件、Actions、数据查询和 schema。
3. 对比或注意点：数据访问模块用 server-only 保护。
4. 可追问时补充：用 ESLint 规则约束模块之间的依赖。

**常见追问**

- 为什么不把所有东西都放在 app 目录中？→ 路由结构和业务结构不一定一致；一个业务组件可能被多个路由使用；app 目录过重会难以维护。

---

### Q19. 多个 Next.js 应用怎么在 monorepo 中共享代码？

- **难度**: 高级
- **标签**: monorepo

**答案**

```text
apps/
  web/        # Next.js 主站
  admin/      # Next.js 后台
  docs/       # 文档站
packages/
  ui/         # 组件库
  db/         # 数据访问（server-only）
  auth/       # 鉴权
  config/     # ESLint、TypeScript 配置
```

**要点**：

- 工具：pnpm workspace + Turborepo（任务编排、远程缓存）
- 内部包可以不预先构建，用 `transpilePackages` 让 Next.js 直接编译源码
- 共享 UI 包：交互组件在包内声明 `'use client'`（每个需要的文件），服务端组件不声明
- 服务端专用包：`import 'server-only'`
- 依赖版本统一（React、Next.js 只有一个版本）
- **多区域（Multi-Zones）**：多个 Next.js 应用部署在同一个域名下的不同路径（rewrites），各自独立部署

**口述模板（30–60 秒）**

1. 一句话结论：用 pnpm workspace 和 Turborepo 管理，共享 UI、数据访问、鉴权和配置包。
2. 关键机制：transpilePackages 直接编译内部包源码，不需要预先构建。
3. 对比或注意点：共享 UI 包中的交互组件要声明 'use client'。
4. 可追问时补充：多个应用可以通过 Multi-Zones 部署在同一个域名下。

**常见追问**

- Multi-Zones 之间导航有什么限制？→ 跨应用的导航是硬导航（整页加载），共享状态需要通过 Cookie 或服务端。

---

### Q20. App Router 架构下怎么做微前端？

- **难度**: 高级
- **标签**: 微前端, 架构

**答案**

| 方案 | 说明 | 取舍 |
|------|------|------|
| **Multi-Zones** | 按路径拆分应用，每个应用独立部署 | 最简单；跨应用导航是硬导航 |
| Module Federation | 运行时加载远程模块 | 对 App Router / RSC 支持有限，复杂 |
| 构建时集成（npm 包） | 各团队发布包，主应用集成 | 需要重新部署主应用 |
| iframe | 完全隔离 | 体验和通信差 |

**推荐**：

- 优先考虑 **monorepo + 按领域拆分 + 独立构建缓存**，在一个应用中满足大部分团队协作需求
- 确实需要独立部署时，用 **Multi-Zones** 按业务线拆分
- RSC 下的运行时模块共享仍不成熟，谨慎使用

**口述模板（30–60 秒）**

1. 一句话结论：App Router 下推荐 Multi-Zones 按路径拆分，或者 monorepo 中按领域组织。
2. 关键机制：Multi-Zones 通过 rewrites 把不同路径交给不同应用。
3. 对比或注意点：Module Federation 对 RSC 支持有限。
4. 可追问时补充：先问是否真的需要独立部署，很多时候 monorepo 就够了。

**常见追问**

- 多个 Zone 的公共头部怎么保持一致？→ 共享 UI 包，各 Zone 引用相同的版本；或者头部由每个 Zone 自行渲染共享组件。

---

### Q21. 怎么设计 Next.js 应用的 BFF（服务端聚合层）？

- **难度**: 高级
- **标签**: 架构, BFF

**答案**

**Next.js 本身就可以作为 BFF**：

```text
浏览器 ← RSC / Server Actions ← Next.js（服务端组件 + 数据访问层） ← 微服务 / 数据库
```

- 服务端组件直接调用内部服务，**聚合**数据后渲染，浏览器不需要多次请求
- Server Actions 作为写操作入口
- Route Handlers 给外部（移动端、第三方）提供 API

**设计要点**：

- 数据访问层封装内部服务调用：统一鉴权（传递用户令牌）、超时、重试、错误转换
- 内部服务的地址和密钥只在服务端
- 用 `React.cache` 在单次渲染中去重，用 `'use cache'` 跨请求缓存
- 熔断和降级：某个服务不可用时，对应的 Suspense 区域显示降级内容（error.tsx 或组件内处理），不影响整个页面

**何时需要独立的 BFF**：多个客户端（Web、App、小程序）共享同一套聚合逻辑；后端团队维护。

**口述模板（30–60 秒）**

1. 一句话结论：Next.js 的服务端组件和 Server Actions 本身就可以作为 BFF。
2. 关键机制：服务端组件聚合多个服务的数据，数据访问层统一处理鉴权、超时、错误。
3. 对比或注意点：某个服务不可用时，在 Suspense 区域内降级，不影响整个页面。
4. 可追问时补充：多个客户端共享聚合逻辑时，考虑独立的 BFF 服务。

**常见追问**

- 移动端也需要同样的数据怎么办？→ 把聚合逻辑放在共享的服务层（数据访问层），服务端组件和 Route Handler 都调用它。

---

### Q22. 服务端组件中怎么处理部分失败？

- **难度**: 高级
- **标签**: 错误处理, 架构

**答案**

**问题**：页面中某个推荐服务失败，不应该让整个页面显示错误。

**策略**：

1. **组件级错误边界**：用 Suspense 和错误边界包裹非关键区域

```tsx
<ErrorBoundary fallback={<RecommendationsUnavailable />}>
  <Suspense fallback={<Skeleton />}>
    <Recommendations />
  </Suspense>
</ErrorBoundary>
```

注意：`ErrorBoundary` 是客户端组件（例如 `react-error-boundary`），可以包裹服务端组件。

2. **组件内捕获**：非关键数据在数据层返回空值或默认值

```tsx
async function Recommendations() {
  const items = await getRecommendations().catch((error) => {
    reportError(error)
    return []
  })
  if (items.length === 0) return null
  return <List items={items} />
}
```

3. **路由级**：`error.tsx` 处理整个路由段的错误
4. **超时**：给外部调用设置超时（`AbortSignal.timeout(2000)`），慢服务不拖垮页面

**原则**：区分**关键数据**（失败则整页错误或 404）和**非关键数据**（失败则降级）。

**口述模板（30–60 秒）**

1. 一句话结论：区分关键和非关键数据，非关键区域用错误边界或组件内捕获来降级。
2. 关键机制：错误边界是客户端组件，可以包裹服务端组件。
3. 对比或注意点：外部调用设置超时，慢服务不拖垮页面。
4. 可追问时补充：降级时也要上报错误。

**常见追问**

- 流已经开始，某个 Suspense 中的组件抛错会怎样？→ 由最近的错误边界处理，只影响该区域；没有错误边界时冒泡到 error.tsx。

---

### Q23. 怎么设计一个 Next.js 内容站的发布流程（CMS + 缓存）？

- **难度**: 高级
- **标签**: 缓存, 架构

**答案**

```text
编辑在 CMS 中发布
  → CMS 调用 Webhook：POST /api/revalidate（带签名）
  → Route Handler 验证签名
  → revalidateTag('post-slug-x', 'max') 和 revalidateTag('posts', 'max')
  → 下一次访问时后台重新生成，返回旧内容，生成完毕后更新
```

**草稿预览**：

- `draftMode()`：编辑点击预览 → 访问开启草稿模式的 Route Handler（验证密钥）→ 设置 Cookie → 页面在草稿模式下绕过缓存，获取草稿内容

**设计要点**：

- Webhook 必须**验证签名**，并且幂等
- 标签粒度：单篇文章、列表、分类、作者
- 失效失败的重试和告警
- 重要内容（例如撤稿）需要**立即**生效：可以使用更激进的失效方式，同时清除 CDN 缓存
- 定时发布：CMS 的定时任务触发 Webhook，或使用 `cacheLife` 的 `revalidate` 时间作为兜底

**口述模板（30–60 秒）**

1. 一句话结论：CMS 发布时通过签名的 Webhook 调用 Route Handler，按标签失效缓存。
2. 关键机制：revalidateTag 采用后台更新，下一次访问时重新生成。
3. 对比或注意点：Webhook 要验证签名、幂等，失败时有重试和告警。
4. 可追问时补充：草稿预览使用 draftMode 绕过缓存。

**常见追问**

- 前面还有 CDN 缓存怎么办？→ 设置合理的 `Cache-Control`（较短的 `s-maxage` + `stale-while-revalidate`），或者在 Webhook 中同时调用 CDN 的清除 API。

---

### Q24. 怎么评估是否在项目中开启 Cache Components？

- **难度**: 高级
- **标签**: 升级, 决策

**答案**

**收益**：

- 缓存行为显式、可预测，减少「为什么数据不更新」类的问题
- PPR：静态外壳 + 流式动态内容，首屏更快
- Partial Prefetching：导航更快
- 官方方向：16.4 起推荐，17 将成为默认

**成本**：

- 需要重构：移除路由段配置（`dynamic`、`revalidate`）和 fetch 缓存选项，改为 `'use cache'`
- 构建时会对未包裹 Suspense 的动态访问报错，需要逐个处理
- 团队需要学习新模型
- 多实例部署需要配置 `cacheHandlers`
- 第三方库兼容性

**迁移策略**：

1. 在新项目或新路由上先使用
2. 现有项目开启后，根据构建错误逐个处理：决定是缓存还是放进 Suspense
3. 先把明显可以缓存的数据（CMS 内容、商品信息）改为 `'use cache'` + 标签
4. 监控缓存命中率和 TTFB

**口述模板（30–60 秒）**

1. 一句话结论：Cache Components 让缓存显式可预测，并带来 PPR 和更快的导航，代价是重构和学习成本。
2. 关键机制：开启后构建会对未处理的动态访问报错，需要逐个选择缓存或流式。
3. 对比或注意点：多实例部署需要额外配置缓存处理器。
4. 可追问时补充：官方方向是 17 成为默认，新项目建议直接使用。

**常见追问**

- 迁移中最常见的构建错误是什么？→ 在 Suspense 外部访问 cookies / headers / searchParams，或者未缓存的数据请求在页面顶层被 await。

---

### Q25. 如何回答「Next.js 太复杂了」这个观点？

- **难度**: 高级
- **标签**: 观点, 取舍

**答案**

**承认的部分**：

- App Router 引入了服务端 / 客户端组件、多层缓存、Server Actions 等新概念，学习曲线陡峭
- 13–15 的缓存默认值多次变化，造成困惑
- 自托管的完整功能需要额外工作
- 框架安全事件说明新的攻击面需要时间成熟

**辩证的部分**：

- 复杂性来自问题本身：同时追求首屏性能、SEO、交互、个性化、缓存，本质上就复杂；Next.js 把这些整合在一个模型中
- Cache Components 正在把缓存从隐式变为显式，降低心智负担
- 很多复杂性是**可选的**：可以只用静态生成，或者只用客户端组件

**我的选型建议**：根据需求选择，内部工具用 Vite + React，需要 SEO 和首屏性能的产品用 Next.js；在团队中建立约定（数据访问层、缓存策略文档），控制复杂性。

**口述模板（30–60 秒）**

1. 一句话结论：复杂性确实存在，但很大程度来自问题本身，并且很多是可选的。
2. 关键机制：缓存默认值的反复变化是主要的困惑来源，Cache Components 在改善这一点。
3. 对比或注意点：不需要 SEO 的项目不一定需要 Next.js。
4. 可追问时补充：通过团队约定控制复杂性。

**常见追问**

- 你在项目中遇到的最大的坑是什么？→ 结合自己的经验回答，例如缓存不更新、hydration mismatch、多实例缓存不一致、升级时的破坏性变更。

---

### Q26. Next.js 和 React 的未来方向是什么？

- **难度**: 高级
- **标签**: 趋势

**答案**

（截至 2026-10，以官方为准）

- **Cache Components 成为默认**：Next.js 17 计划默认开启，显式缓存 + PPR 成为标准模型
- **Turbopack**：开发和构建的默认工具，持续提升速度和兼容性
- **React Compiler**：自动 memo 普及，手动优化减少
- **React 新能力**：Activity、View Transitions、Fragment refs 等逐步稳定
- **部署适配器**：降低对单一平台的依赖，其他平台可以支持完整特性
- **AI 相关**：流式 UI、AI SDK、面向 AI 代理的开发工具（Next.js DevTools MCP 等）
- **安全**：RSC 协议经历了严重漏洞之后，框架和社区更加重视序列化边界的安全
- **RSC 生态扩展**：React Router、Waku、Vite 插件都在支持 RSC，RSC 不再只属于 Next.js

**口述模板（30–60 秒）**

1. 一句话结论：方向是显式缓存加 PPR 成为默认，Turbopack 和 React Compiler 普及。
2. 关键机制：Cache Components 在 17 中计划默认开启。
3. 对比或注意点：RSC 生态正在扩展到其他框架。
4. 可追问时补充：安全事件让社区更重视序列化边界的安全。

**常见追问**

- 你会怎么跟进这些变化？→ 关注官方博客和升级指南、在非核心项目中试用新特性、关注安全公告、在团队中分享。
