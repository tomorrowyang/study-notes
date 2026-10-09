# Next.js 真实面试题

> 根据近年公开面经、技术社区讨论和官方发布的变化整理，`来源倾向`表示这类题目常出现的场景，不代表某公司的固定题库。答案为要点，详细内容见对应链接。

---

### R1. 说说 App Router 和 Pages Router 的区别，你们项目为什么选（或不选）App Router？

- **来源倾向**: 高频，几乎必问

**答案**

- 组件模型：App Router 默认服务端组件，Pages Router 全部是客户端组件 + SSR
- 数据获取：组件中直接 await vs `getServerSideProps` / `getStaticProps`
- 布局：嵌套布局、导航时保持状态 vs `_app` 单一布局
- 流式：Suspense 原生支持
- 数据变更：Server Actions vs API 路由
- 选型：新项目推荐 App Router；老项目渐进迁移

**口述模板（30–60 秒）**

1. **一句话结论**：App Router 基于 React Server Components，默认服务端组件，支持嵌套布局和流式渲染。
2. **关键机制**：数据在组件中直接获取，变更用 Server Actions。
3. **对比或注意点**：Pages Router 的组件全部在客户端 hydrate，数据获取只能在页面级。
4. **可追问时补充**：结合项目说明选型理由和遇到的问题。

详见：[10-junior Q2](./10-junior-qa.md)、[30-senior Q17](./30-senior-qa.md)

---

### R2. 服务端组件和客户端组件有什么区别？怎么决定用哪个？

- **来源倾向**: 高频

**答案**

- 服务端组件：代码不发送到浏览器，可以访问数据库和密钥，不能使用状态、事件和浏览器 API
- 客户端组件：可以交互，代码发送到浏览器，首次加载也会 SSR
- 原则：默认服务端组件，需要交互的叶子组件才用客户端组件

详见：[10-junior Q7–Q9](./10-junior-qa.md)、[20-mid Q4](./20-mid-qa.md)

---

### R3. RSC 和 SSR 是一回事吗？

- **来源倾向**: 高频，考察理解深度

**答案**

- 不是。RSC 是组件类型，输出 RSC Payload；SSR 是渲染时机，输出 HTML
- 客户端组件也会被 SSR；服务端组件可以在构建时或请求时运行
- 首屏需要两者配合：RSC Payload 用于构建组件树，HTML 用于快速显示

详见：[20-mid Q3](./20-mid-qa.md)

---

### R4. 讲讲 Next.js 的缓存机制

- **来源倾向**: 高频，难点

**答案**

- **先说明版本**：
  - 14 / 15 旧模型：Request Memoization、Data Cache、Full Route Cache、Router Cache 四层
  - 15：fetch 和 GET Route Handler 默认不缓存，路由缓存页面段默认不复用
  - 16 Cache Components：默认动态，`'use cache'` 显式缓存，`cacheLife` / `cacheTag` 控制，`updateTag` / `revalidateTag` / `refresh` 失效
- 16.4 起官方推荐所有应用使用 Cache Components，17 将成为默认

**口述模板（30–60 秒）**

1. **一句话结论**：要分版本回答，旧模型有四层隐式缓存，16 的 Cache Components 改为显式缓存。
2. **关键机制**：旧模型的四层分别是请求记忆、数据缓存、完整路由缓存、客户端路由缓存；新模型用 use cache 声明，配合 cacheLife 和 cacheTag。
3. **对比或注意点**：15 改变了 fetch 的默认缓存行为。
4. **可追问时补充**：失效时 updateTag 立即生效，revalidateTag 后台更新。

详见：[20-mid Q10–Q14](./20-mid-qa.md)、[30-senior Q1–Q3](./30-senior-qa.md)

---

### R5. 数据更新了但页面没变，你怎么排查？

- **来源倾向**: 高频，实战题

**答案**

1. 确认版本和是否开启 Cache Components
2. 服务端：数据是否被缓存（fetch 缓存、`'use cache'`、Full Route Cache）？变更后是否失效了正确的标签或路径？
3. 多实例：失效是否只在一个实例生效（需要共享缓存）
4. 客户端：路由缓存中的旧页面，是否调用了 `router.refresh()` 或在 Action 中失效
5. CDN：是否缓存了 HTML
6. 开发环境和生产环境行为不同，用 `next build && next start` 复现

**口述模板（30–60 秒）**

1. **一句话结论**：按「服务端缓存 → 多实例 → 客户端路由缓存 → CDN」逐层排查。
2. **关键机制**：检查变更后是否失效了对应的标签，失效 API 是否在正确的位置调用。
3. **对比或注意点**：开发环境没有完整的缓存行为，要用生产构建复现。
4. **可追问时补充**：多实例需要共享的缓存处理器，否则失效只在一个实例生效。

详见：[20-mid Q10、Q14](./20-mid-qa.md)、[30-senior Q3](./30-senior-qa.md)

---

### R6. Server Actions 是怎么实现的？安全吗？

- **来源倾向**: 高频

**答案**

- 编译时生成 Action ID，客户端得到引用，调用时 POST 到当前页面并带上 `Next-Action` 头
- 返回结果和更新后的 RSC Payload
- 安全：每个 Action 都是公开端点，需要鉴权、校验、权限检查；内置 Origin 检查防 CSRF；闭包变量加密
- React2Shell 就是 Server Function 请求反序列化的漏洞

详见：[20-mid Q15–Q16](./20-mid-qa.md)、[30-senior Q7、Q15](./30-senior-qa.md)、[40-coding H2](./40-coding-handwritten.md)

---

### R7. Server Actions 和 Route Handlers 怎么选？

- **来源倾向**: 中频

**答案**

- Server Actions：应用内部的数据变更、表单，类型安全、渐进增强、和缓存失效集成
- Route Handlers：外部调用（Webhook、移动端、第三方）、需要控制 HTTP 细节、流式响应、GET 请求
- Server Actions 串行执行，不适合并发读取

详见：[10-junior Q14](./10-junior-qa.md)

---

### R8. middleware（proxy）能做什么？能用来做鉴权吗？

- **来源倾向**: 高频，常结合 CVE 追问

**答案**

- 适合重定向、重写、国际化、A/B 测试、乐观的登录检查
- 不能作为唯一的鉴权：CVE-2025-29927 通过伪造 `x-middleware-subrequest` 跳过 middleware
- 16 中改名为 proxy，运行在 Node.js 运行时
- 鉴权要在数据访问层再次检查

详见：[20-mid Q18–Q19](./20-mid-qa.md)、[30-senior Q14](./30-senior-qa.md)

---

### R9. 讲讲你知道的 Next.js / React 安全漏洞

- **来源倾向**: 中频，2025 年后明显增多

**答案**

- **CVE-2025-29927**（2025-03）：middleware 绕过，自托管受影响，教训是纵深防御
- **React2Shell CVE-2025-55182**（2025-12，CVSS 10）：RSC Server Function 反序列化导致未授权远程代码执行，影响所有 RSC 框架；修复版本 React 19.0.1 / 19.1.2 / 19.2.1，Next.js 15.x / 16.x 对应补丁
- 应对：依赖监控、快速升级流程、网关层缓解、怀疑被利用时轮换密钥

**口述模板（30–60 秒）**

1. **一句话结论**：最重要的两个是 middleware 绕过和 React2Shell 远程代码执行。
2. **关键机制**：前者是信任了外部可控的内部请求头，后者是服务端反序列化 Server Function 请求时的缺陷。
3. **对比或注意点**：前者只影响依赖 middleware 鉴权的自托管应用，后者影响几乎所有 RSC 应用。
4. **可追问时补充**：应对措施是依赖监控、快速升级、纵深防御。

详见：[30-senior Q14–Q16](./30-senior-qa.md)

---

### R10. 什么是 hydration？hydration mismatch 怎么解决？

- **来源倾向**: 高频

**答案**

- hydration：React 在服务端生成的 HTML 上绑定事件、建立组件树
- mismatch 原因：时间、随机数、浏览器 API、不合法的 HTML 嵌套、浏览器扩展
- 解决：首次渲染保持一致，浏览器相关的值在 useEffect 中设置；`ssr: false`；`suppressHydrationWarning`（少量情况）

详见：[20-mid Q7](./20-mid-qa.md)

---

### R11. 流式渲染是怎么实现的？有什么注意事项？

- **来源倾向**: 中频

**答案**

- HTTP 分块传输，先发送外壳和 fallback，数据就绪后发送内容和替换脚本
- 选择性 hydration：用户交互的区域优先
- 注意：流开始后不能修改状态码；反向代理要关闭缓冲

详见：[20-mid Q6](./20-mid-qa.md)

---

### R12. 什么是 PPR？和 ISR 有什么区别？

- **来源倾向**: 中频，近年上升

**答案**

- PPR：同一页面静态外壳 + 动态部分流式，16 中通过 Cache Components 提供
- ISR：整个页面静态生成，按时间或按需重新生成
- PPR 解决的是「一个页面中既有静态内容又有个性化内容」的问题；ISR 解决的是「静态页面如何更新」

详见：[20-mid Q9](./20-mid-qa.md)、[10-junior Q17](./10-junior-qa.md)

---

### R13. Next.js 16 有哪些重要变化？

- **来源倾向**: 中频，考察是否跟进

**答案**

- Cache Components（`'use cache'`、`cacheLife`、`cacheTag`、`updateTag`、`refresh`）
- Turbopack 默认
- `middleware.ts` → `proxy.ts`（Node.js 运行时）
- React Compiler 支持稳定、React 19.2
- `revalidateTag` 需要第二个参数
- 移除 AMP、`next lint`；Node.js 20.9+
- 16.4：新应用默认开启 Cache Components 和 Partial Prefetching

详见：[20-mid Q26](./20-mid-qa.md)、[00-roadmap](./00-roadmap.md)

---

### R14. 怎么优化 Next.js 应用的首屏性能？

- **来源倾向**: 高频

**答案**

- 更多组件保持在服务端，减少客户端 JS
- 静态化和缓存：可以缓存的数据缓存，PPR 静态外壳
- 流式：慢的部分放进 Suspense
- 避免请求瀑布：并行、预加载
- `next/image`（LCP 图片 `priority`）、`next/font`、`next/script`
- 部署：CDN、函数靠近数据库

详见：[20-mid Q24](./20-mid-qa.md)、[30-senior Q8–Q9](./30-senior-qa.md)

---

### R15. `next/image` 做了哪些优化？

- **来源倾向**: 中频

**答案**

- 自动生成多尺寸和现代格式（WebP / AVIF），`srcset`
- 默认懒加载，`priority` 用于 LCP 图片
- 要求宽高（或 `fill`）防止 CLS
- 远程图片需要配置 `remotePatterns`（防止被滥用为图片代理）
- 自托管时注意 CPU 消耗

详见：[10-junior Q21](./10-junior-qa.md)

---

### R16. 怎么在 App Router 中做鉴权？

- **来源倾向**: 高频

**答案**

- 分层：proxy 乐观检查 → 数据访问层验证（核心） → 每个 Action / Route Handler 都检查
- 数据访问层用 `server-only` 和 `React.cache`，返回 DTO
- 不能只在布局中检查（导航时不重新渲染）
- Cookie：`HttpOnly`、`Secure`、`SameSite`

详见：[20-mid Q19](./20-mid-qa.md)、[40-coding H1](./40-coding-handwritten.md)

---

### R17. 怎么防止服务端数据泄露到客户端？

- **来源倾向**: 中频

**答案**

- 传给客户端组件的 props 会完整出现在 RSC Payload 中
- DTO 只返回需要的字段
- `server-only` 防止服务端模块被客户端导入
- 环境变量不加 `NEXT_PUBLIC_`
- React taint API 作为额外防护

详见：[30-senior Q6](./30-senior-qa.md)

---

### R18. 自托管 Next.js 遇到过哪些问题？

- **来源倾向**: 中频，偏实战

**答案**

- 多实例缓存不一致 → 共享 `cacheHandler`
- Server Action 解密失败 → 统一加密密钥
- 发布期间 chunk 加载失败 → 保留旧静态资源、版本偏差处理
- 流式不生效 → 关闭代理缓冲
- 图片优化占用 CPU → 独立服务或 CDN

详见：[30-senior Q11–Q12](./30-senior-qa.md)、[50-system-design S6](./50-system-design.md)

---

### R19. 怎么设计一个页面的渲染和缓存策略？（给出一个电商详情页）

- **来源倾向**: 高频，系统设计类

**答案**

- 按「变化频率 × 是否个性化」拆分组件
- 商品信息：缓存 + 标签；价格：短缓存；库存、购物车：Suspense 流式
- 每个缓存都有失效路径
- 个性化数据不进入共享缓存

**口述模板（30–60 秒）**

1. **一句话结论**：按数据的变化频率和是否个性化拆分组件，分别选择缓存或流式。
2. **关键机制**：商品信息用 use cache 加标签，后台修改时失效；库存和购物车放在 Suspense 中请求时渲染。
3. **对比或注意点**：个性化数据不能进入共享缓存。
4. **可追问时补充**：热门商品在构建时预渲染，其余按需生成。

详见：[30-senior Q2](./30-senior-qa.md)、[50-system-design S1](./50-system-design.md)

---

### R20. 客户端状态管理在 App Router 中怎么做？

- **来源倾向**: 中频

**答案**

- 服务端数据由服务端组件获取，客户端只管理客户端状态
- URL 状态用 searchParams
- 全局 store 不能定义在模块级（服务端跨请求共享），在 Provider 中按请求创建

详见：[20-mid Q21](./20-mid-qa.md)

---

### R21. 从 Pages Router 迁移到 App Router 怎么做？

- **来源倾向**: 中频

**答案**

- 两者可以共存，按路由逐步迁移
- 先建立根布局和 Provider，再迁移数据获取（`getServerSideProps` → 组件中获取）、路由 API、元数据
- 相互链接多的页面一起迁移（跨路由器的导航是硬导航）

详见：[30-senior Q17](./30-senior-qa.md)

---

### R22. 如何实现乐观更新？

- **来源倾向**: 中频

**答案**

- `useOptimistic` + `startTransition` + Server Action
- 乐观值只在 transition 期间有效，结束后使用服务端的真实数据
- 失败时自动回到真实状态，需要提示用户

详见：[20-mid Q17](./20-mid-qa.md)、[40-coding H5](./40-coding-handwritten.md)

---

### R23. Next.js、Nuxt、Remix（React Router）怎么选？

- **来源倾向**: 中频

**答案**

- 技术栈：React 或 Vue
- Next.js：RSC 核心、缓存最强大也最复杂、Vercel 支持最好
- React Router：贴近 Web 标准，loader / action
- Nuxt：Vue 生态，Nitro 部署适配好
- 不需要 SEO 的后台：Vite + React 也是好选择

详见：[20-mid Q30、Q32](./20-mid-qa.md)

---

### R24. 讲一个你在 Next.js 项目中解决的性能 / 线上问题

- **来源倾向**: 高频，项目深挖

**答案**

使用 STAR 结构：

- **背景**：什么页面、什么指标异常（例如 TTFB 从 300ms 升到 2s）
- **定位**：用追踪发现某个组件调用了 `cookies()` 导致整个路由变为动态，并且三个请求串行
- **方案**：把个性化部分放进 Suspense、其余数据缓存、并行请求
- **结果**：TTFB 和 LCP 的具体改善、缓存命中率
- **复盘**：建立规范（数据访问层、缓存策略文档、构建产物检查）

详见：[30-senior Q8](./30-senior-qa.md)

---

### R25. 你怎么看 Next.js 的复杂度和未来？

- **来源倾向**: 中频，开放题

**答案**

- 承认：学习曲线陡、缓存默认值多次变化、自托管需要额外工作、新攻击面
- 辩证：复杂性来自问题本身，很多是可选的
- 方向：Cache Components 默认、Turbopack、React Compiler、部署适配器、RSC 生态扩展到其他框架

**口述模板（30–60 秒）**

1. **一句话结论**：复杂度确实高，但很大程度上来自同时追求性能、SEO 和个性化这一问题本身。
2. **关键机制**：缓存从隐式到显式的转变正在降低心智负担。
3. **对比或注意点**：不是所有项目都需要 Next.js。
4. **可追问时补充**：通过团队约定控制复杂度，持续关注官方变化和安全公告。

详见：[30-senior Q25–Q26](./30-senior-qa.md)
