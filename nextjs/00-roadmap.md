# Next.js 学习路线（高级工程师面试向）

## 面试画像

Next.js 是 React 全栈和 SSR 方向的主流框架，面试考察集中在三层：

- **使用层**：App Router 文件约定（page、layout、loading、error）、路由（动态、并行、拦截）、数据获取、Server Actions、Route Handlers、Metadata、Image / Font / Link —— 初中级必问
- **原理层**：Server Components 与 Client Components 的边界、RSC Payload、流式渲染与 Suspense、hydration、渲染策略（静态 / 动态 / 流式 / PPR）、**缓存体系**（数据、渲染结果、客户端路由缓存）—— 中高级区分点，**缓存是最难也最常问的**
- **工程与架构**：Cache Components（`use cache`）、鉴权与 proxy、Server Actions 安全、性能优化、自托管与多实例部署、Pages Router 迁移、可观测性、安全漏洞应对 —— 高级岗必谈

高级岗典型问法：

> 「加了 `'use client'` 之后发生了什么？」「Server Component 和 SSR 是一回事吗？」「为什么我的页面数据不更新？Next.js 有几层缓存？」「Server Action 为什么要做鉴权？」「从 Pages Router 迁移到 App Router 的策略？」「Next.js 16 的 Cache Components 解决了什么问题？」

> 回答缓存相关问题前，**先说明版本和配置**（Next.js 14 / 15 / 16，是否开启 `cacheComponents`），因为默认行为在这几个版本中变化很大。

## 版本说明（截至 2026-10）

| 版本 | 时间 | 关键内容 |
|------|------|----------|
| 13 | 2022-10 | App Router（beta）、React Server Components、`next/font`；13.4 App Router 稳定 |
| 14 | 2023-10 | Server Actions 稳定、PPR 预览、`next/image` 优化 |
| **15** | 2024-10 | **React 19**；**请求相关 API 改为异步**（`cookies()`、`headers()`、`params`、`searchParams` 需要 await）；**`fetch` 和 GET Route Handler 默认不再缓存**；客户端路由缓存中动态页面默认不复用；`next dev --turbopack` 稳定；`next.config.ts`；15.1 起 `after()` 稳定 |
| **16** | 2025-10 | **Cache Components**（`'use cache'`、`cacheLife`、`cacheTag`、`updateTag`、`refresh`，基于 PPR）；**Turbopack 成为默认打包器**（`--webpack` 退回）；**`middleware.ts` 更名为 `proxy.ts`**（运行在 Node.js 运行时）；React Compiler 支持稳定；React 19.2（View Transitions、`<Activity>`、`useEffectEvent`）；`revalidateTag` 需要第二个参数（cacheLife 配置）；移除 AMP 和 `next lint`；要求 Node.js 20.9+ |
| 16.1–16.3 | 2025-12 ~ 2026 | Turbopack 文件系统缓存（dev 默认，16.3 起 build 也默认）、内存回收；Instant Navigation 相关能力 |
| **16.4** | **2026-10-06** | **官方推荐所有应用使用 Cache Components**；`create-next-app` 新建项目**默认开启**（`cacheComponents: true` + `partialPrefetching: true`）；React 19.3；Turbopack 磁盘缓存更小；**Next.js 17 将把 Cache Components 设为默认** |

**安全事件**（面试高频）：

- **CVE-2025-29927**（2025-03）：通过伪造 `x-middleware-subrequest` 请求头绕过 middleware，依赖 middleware 做鉴权的应用受影响
- **React2Shell / CVE-2025-55182**（2025-12，CVSS 10.0）：React Server Components 的 Server Function 反序列化导致**未鉴权远程代码执行**，影响 Next.js 15.x / 16.x 等，需要升级到修复版本（例如 15.0.5、15.1.9 … 15.5.7、16.0.7）

> 以上版本细节以 Next.js 官方博客和升级指南为准。很多公司的代码库仍在使用 14 或 15，面试时先确认对方的版本。

## 知识地图

```text
Next.js
├─ 路由：app 目录、文件约定（page / layout / template / loading / error / not-found / default）、
│        动态路由、路由组、并行路由、拦截路由、Link / useRouter / redirect / notFound
├─ 渲染：Server Components、Client Components（'use client'）、组件组合规则、
│        RSC Payload、流式渲染（Suspense / loading）、hydration
├─ 渲染策略：静态预渲染、动态渲染、ISR、PPR、Cache Components
├─ 数据：服务端组件中直接取数、fetch 扩展、Server Actions（'use server'）、Route Handlers、
│        表单与 useActionState、乐观更新（useOptimistic）
├─ 缓存：
│   ├─ 旧模型（14 / 15）：Request Memoization、Data Cache、Full Route Cache、Router Cache
│   └─ 新模型（16 Cache Components）：'use cache'、cacheLife、cacheTag、updateTag / revalidateTag / refresh
├─ 中间层：proxy.ts（原 middleware.ts）、重写 / 重定向、国际化、A/B 测试
├─ 优化：next/image、next/font、next/script、Metadata API、包优化、Turbopack
├─ 运行时：Node.js、Edge；after()、instrumentation
├─ 部署：Vercel、自托管（standalone、Docker）、多实例缓存（cacheHandler）、CDN
├─ 安全：Server Actions 鉴权和校验、环境变量、server-only、CSRF、CVE 应对
└─ 迁移：Pages Router → App Router、14 → 15 → 16 升级
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — App Router 文件约定、路由、Server / Client Components 基础、数据获取、Server Actions、优化组件
2. [中级 Q&A](20-mid-qa.md) — RSC 原理、组件边界、流式渲染、渲染策略、缓存体系、Server Actions 原理、proxy、鉴权
3. [高级 Q&A](30-senior-qa.md) — Cache Components、PPR、RSC 协议、hydration、性能、部署、安全事件、迁移、架构取舍
4. [手写与现场编码](40-coding-handwritten.md) — 12 道题：鉴权 proxy、Server Action、乐观更新、流式页面、缓存失效等
5. [系统设计与场景](50-system-design.md) — 电商、内容站、SaaS 后台、多租户、国际化、自托管等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. App Router 与 Pages Router 的区别，为什么引入 RSC
2. Server Component 与 SSR 的区别：前者是组件类型，后者是渲染时机
3. `'use client'` 是**模块边界**：该文件及其导入的模块都进入客户端包；客户端组件仍然会在服务端预渲染 HTML
4. 组件组合规则：客户端组件不能导入服务端组件，但可以通过 `children` / props 接收
5. RSC Payload 是什么，客户端导航时如何使用
6. 流式渲染：`loading.tsx`、Suspense 边界、选择性 hydration
7. 渲染策略：静态、动态、ISR、PPR；什么会让路由变成动态
8. 旧缓存模型的四层缓存及其失效方式
9. Next.js 15 的缓存默认值变化；Next.js 16 Cache Components 的显式缓存模型
10. `'use cache'` 的缓存键、`cacheLife`、`cacheTag`、`updateTag` 与 `revalidateTag` 的区别
11. Server Actions 的原理：是**公开的 HTTP 端点**，必须做鉴权和输入校验
12. Server Actions 与 Route Handlers 的选型
13. proxy（middleware）的适用场景和局限：不要把它作为唯一的鉴权手段（CVE-2025-29927）
14. hydration mismatch 的原因和处理
15. 性能：减少客户端 JS、图片和字体优化、流式、PPR、包分析
16. 自托管：standalone 输出、多实例共享缓存、ISR 在多实例下的一致性
17. 安全：环境变量（`NEXT_PUBLIC_`）、`server-only`、RSC 漏洞的应对和升级流程
18. 迁移：Pages Router 到 App Router 的渐进迁移，15 / 16 的破坏性变更

## 相关技术

- [React](../react/00-roadmap.md)：Server Components、Suspense、`use`、`useActionState`、`useOptimistic`、React Compiler —— Next.js 的基础
- [Node.js](../node/00-roadmap.md)：服务端运行时、流、AsyncLocalStorage
- [性能优化](../performance/00-roadmap.md)、[浏览器原理](../browser/00-roadmap.md)：LCP、INP、流式 HTML、缓存
- [前端安全](../frontend-security/00-roadmap.md)：Server Actions 安全、CSRF、RSC 漏洞、供应链
- [Vue](../vue/00-roadmap.md)：Nuxt 与 Next.js 的对比
- [Docker](../docker/00-roadmap.md)：standalone 镜像、自托管部署

## 自检清单

- [ ] 能说清 `'use client'` 的确切含义和组件组合规则
- [ ] 能画出一个请求从服务端渲染到客户端 hydration 的流程，并说明 RSC Payload 的作用
- [ ] 能说出旧模型的四层缓存，以及 15、16 版本的变化
- [ ] 能用 `'use cache'` + `cacheTag` + `updateTag` 实现一个可以按需失效的页面
- [ ] 能说明 Server Action 的安全要点并写出带鉴权和校验的示例
- [ ] 能解释 PPR 的原理和收益
- [ ] 能设计自托管多实例的缓存方案
- [ ] 能讲清 CVE-2025-29927 和 React2Shell 的影响和应对
