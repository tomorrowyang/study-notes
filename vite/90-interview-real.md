# Vite 真实 / 高频面试题

> 根据近年公开面经、技术社区讨论和 Vite 官方文档整理，按面试中的真实问法组织。`来源倾向` 表示这类问题常见于哪类岗位或面试轮次，不代表某公司的固定题库。每题给出简短答案，详细内容见链接。

---

### R1. Vite 为什么比 webpack 快？

- **难度**: 初级
- **标签**: 原理, 对比
- **来源倾向**: 一面必问，前端通用岗

**答案**

开发时不打包，浏览器按原生 ESM 请求模块，服务器按需编译；第三方依赖一次性预构建并强缓存；转换和打包使用 Rust 工具链（Oxc、Rolldown）。webpack 开发时需要先打包整个依赖图，项目越大越慢。生产构建方面，Vite 8 使用 Rolldown，比 Rollup 快 10–30 倍。

**口述模板（30–60 秒）**

1. **一句话结论**：Vite 快在开发时不打包。
2. **关键机制**：原生 ESM 按需编译，依赖预构建并缓存，使用 Rust 工具链。
3. **对比或注意点**：webpack 启动前要打包整个项目；但 Vite 在超大项目首次加载时请求数多，也会变慢。
4. **可追问时补充**：Vite 8.1 的 Bundled Dev Mode 就是为了解决超大项目请求过多的问题。

详见：[初级 Q2](10-junior-qa.md)、[初级 Q27](10-junior-qa.md)、[中级 Q31](20-mid-qa.md)

---

### R2. 依赖预构建是什么？为什么需要？

- **难度**: 中级
- **标签**: 预构建
- **来源倾向**: 原理面高频

**答案**

启动时把第三方依赖打包为 ESM，放在 `node_modules/.vite/deps`。目的有两个：把 CJS / UMD 转为 ESM；把内部模块很多的依赖（如 lodash-es）合并，减少请求。缓存由 lockfile 和配置计算 hash，变化时重新构建。

**口述模板（30–60 秒）**

1. **一句话结论**：预构建把依赖提前打包成 ESM。
2. **关键机制**：解决 CJS 兼容和请求过多两个问题，结果按 hash 缓存。
3. **对比或注意点**：依赖之间一起打包，抽取公共 chunk，保证 React 等单例。
4. **可追问时补充**：运行时发现新依赖会重新预构建并可能整页刷新，用 optimizeDeps.include 避免。

详见：[初级 Q21](10-junior-qa.md)、[中级 Q3](20-mid-qa.md)、[高级 Q5](30-senior-qa.md)

---

### R3. 浏览器不认识 `import React from 'react'`，Vite 是怎么处理的？

- **难度**: 中级
- **标签**: 原理
- **来源倾向**: 原理面

**答案**

importAnalysis 插件用 es-module-lexer 找到导入语句，把裸模块说明符解析后重写为预构建路径 `/node_modules/.vite/deps/react.js?v=hash`，用 magic-string 替换并保留 source map。

详见：[中级 Q4](20-mid-qa.md)、[手写 H1](40-coding-handwritten.md)

---

### R4. 讲一下 Vite 的 HMR 原理

- **难度**: 中级
- **标签**: HMR
- **来源倾向**: 原理面必问

**答案**

chokidar 监听文件变化 → 模块图找到受影响模块并使缓存失效 → 沿 importers 向上查找 HMR 边界（自身 accept 或导入方 accept 了它）→ 通过 WebSocket 推送更新 → 浏览器执行 dispose，带时间戳重新 import 模块，调用 accept 回调。找不到边界或遇到循环导入时整页刷新。

**口述模板（30–60 秒）**

1. **一句话结论**：Vite HMR 基于模块图查找边界，只重新请求变化的模块。
2. **关键机制**：向上查找 accept 边界，WebSocket 通知，浏览器加时间戳重新 import。
3. **对比或注意点**：不需要重新打包，速度与项目规模无关；webpack 需要重新构建受影响的 chunk。
4. **可追问时补充**：边界以上的模块不会重新执行，持有的仍是旧引用。

详见：[中级 Q5](20-mid-qa.md)、[高级 Q4](30-senior-qa.md)、[手写 H7](40-coding-handwritten.md)

---

### R5. Vite 的 HMR 和 webpack 的 HMR 有什么区别？

- **难度**: 中级
- **标签**: HMR, 对比
- **来源倾向**: 原理面

**答案**

webpack 需要重新编译变化的模块并重新生成受影响的 chunk，客户端下载 hot-update 文件；Vite 只使变化模块的缓存失效，浏览器直接重新请求该模块，不涉及打包。API 上两者类似（`module.hot` 与 `import.meta.hot`）。

详见：[中级 Q5](20-mid-qa.md)、[webpack 中级 Q9](../webpack/20-mid-qa.md)

---

### R6. 改了一个文件，为什么页面整体刷新了？

- **难度**: 中级
- **标签**: HMR, 排查
- **来源倾向**: 场景题

**答案**

导入链上没有 HMR 边界；文件混合导出组件和非组件导致 Fast Refresh 调用 invalidate；循环导入；或者触发了依赖重新预构建。用 `vite --debug hmr` 查看原因。

详见：[中级 Q7](20-mid-qa.md)、[高级 Q14](30-senior-qa.md)

---

### R7. Vite 插件和 Rollup 插件有什么区别？写过什么插件？

- **难度**: 中级
- **标签**: 插件
- **来源倾向**: 工程化岗高频

**答案**

Vite 插件兼容 Rollup / Rolldown 插件接口，并增加了 config、configResolved、configureServer、transformIndexHtml、hotUpdate 等 Vite 特有钩子，以及 enforce、apply 等属性。开发时只调用部分 Rollup 钩子（不调用 moduleParsed 和输出钩子）。

**口述模板（30–60 秒）**

1. **一句话结论**：Vite 插件是 Rollup 插件的超集。
2. **关键机制**：兼容 resolveId / load / transform 等钩子，增加了服务器、HTML 和 HMR 相关钩子。
3. **对比或注意点**：开发时输出相关钩子不调用。
4. **可追问时补充**：举一个自己写过的插件，例如 mock 中间件、构建信息虚拟模块、国际化热更新。

详见：[中级 Q9](20-mid-qa.md)、[中级 Q10](20-mid-qa.md)、[手写 H2–H6](40-coding-handwritten.md)

---

### R8. Vite 插件的执行顺序是怎样的？

- **难度**: 中级
- **标签**: 插件
- **来源倾向**: 工程化岗

**答案**

alias → enforce: 'pre' 的用户插件 → Vite 核心插件 → 普通用户插件 → Vite 构建插件 → enforce: 'post' 的用户插件 → Vite 后置构建插件。apply 控制插件只在开发或构建时生效。

详见：[中级 Q9](20-mid-qa.md)

---

### R9. 怎么实现一个虚拟模块？

- **难度**: 中级
- **标签**: 插件
- **来源倾向**: 手写题

**答案**

resolveId 拦截 `virtual:xxx`，返回 `\0virtual:xxx`；load 根据这个 id 返回代码。`\0` 前缀告诉其他插件不要处理该模块。

详见：[中级 Q11](20-mid-qa.md)、[手写 H2](40-coding-handwritten.md)

---

### R10. Vite 开发和生产用的是同一套工具吗？会有什么问题？

- **难度**: 中级
- **标签**: 原理, 版本
- **来源倾向**: 原理面、近两年高频

**答案**

Vite 8 之前：开发用 esbuild 预构建和转换，生产用 Rollup 打包，两者在 CJS 互操作等方面行为不同，导致开发和生产不一致。Vite 8 统一使用 Rolldown 打包、Oxc 转换，行为一致，生产构建也快了 10–30 倍。

**口述模板（30–60 秒）**

1. **一句话结论**：Vite 8 之前不是，Vite 8 开始统一为 Rolldown。
2. **关键机制**：之前 esbuild 和 Rollup 两套工具，CJS 互操作、Tree Shaking 行为不同。
3. **对比或注意点**：开发仍然不打包，生产打包，所以执行顺序、CSS 顺序仍可能存在差异。
4. **可追问时补充**：Rolldown 是 Rust 实现、兼容 Rollup 插件 API 的打包器。

详见：[初级 Q30](10-junior-qa.md)、[高级 Q6](30-senior-qa.md)

---

### R11. 开发环境正常，生产环境出问题，怎么排查？

- **难度**: 高级
- **标签**: 排查
- **来源倾向**: 场景题高频

**答案**

用 `vite build && vite preview` 本地复现，逐个关闭压缩、Tree Shaking、CSS 代码分割来二分定位。常见原因：sideEffects 声明错误、CJS 默认导入、循环依赖导致的执行顺序问题、依赖函数名的代码被压缩、CSS 顺序变化、base 配置错误。

详见：[中级 Q16](20-mid-qa.md)、[高级 Q16](30-senior-qa.md)

---

### R12. 大项目用 Vite 开发，首次打开页面很慢怎么办？

- **难度**: 高级
- **标签**: 性能
- **来源倾向**: 高级岗场景题

**答案**

原因是请求数多和请求瀑布。优化：治理 barrel 文件、按需导入图标和组件库、路由懒加载、`server.warmup` 预热、审计慢插件、稳定预构建。模块数极大时试用 Vite 8.1 的 Bundled Dev Mode。

**口述模板（30–60 秒）**

1. **一句话结论**：Vite 启动快，但大项目首次加载可能因为请求过多而慢。
2. **关键机制**：每个模块一个请求，依赖层层发现形成瀑布，barrel 文件会加载大量无用模块。
3. **对比或注意点**：先用 Network 面板和 inspect 量化，再针对性优化。
4. **可追问时补充**：Bundled Dev Mode 开发时也打包，官方数据请求数减少约 10 倍。

详见：[中级 Q22](20-mid-qa.md)、[高级 Q11](30-senior-qa.md)、[系统设计 S3](50-system-design.md)

---

### R13. 什么情况下会出现「new dependencies optimized, reloading page」？

- **难度**: 中级
- **标签**: 预构建, 排查
- **来源倾向**: 实际项目经验

**答案**

扫描阶段没有发现的依赖在运行时才被导入，Vite 需要重新预构建，导致整页刷新。把这些依赖加入 `optimizeDeps.include`，或者用 `optimizeDeps.entries` 覆盖所有页面入口。

详见：[高级 Q5](30-senior-qa.md)、[高级 Q13](30-senior-qa.md)

---

### R14. Vite 的环境变量怎么用？为什么要 `VITE_` 前缀？

- **难度**: 初级
- **标签**: 环境变量
- **来源倾向**: 一面基础

**答案**

按 mode 加载 `.env`、`.env.[mode]` 等文件，客户端通过 `import.meta.env` 访问。只有 `VITE_` 前缀的变量会暴露给客户端，防止服务器密钥被误打包进产物。构建时静态替换为字面量，配置文件中要用 `loadEnv` 读取。

详见：[初级 Q6](10-junior-qa.md)、[中级 Q17](20-mid-qa.md)

---

### R15. 生产构建怎么做代码分割？Vite 8 有什么变化？

- **难度**: 中级
- **标签**: 构建
- **来源倾向**: 性能优化相关

**答案**

动态 import 自动分割；手动分组在 Vite 8 中使用 Rolldown 的 `codeSplitting.groups`（支持 priority、minSize、maxSize），取代 `manualChunks`（对象形式已移除，函数形式已废弃）。原因是 manualChunks 不处理依赖，容易产生循环 chunk 和执行顺序问题。

详见：[初级 Q15](10-junior-qa.md)、[中级 Q14](20-mid-qa.md)

---

### R16. Vite 构建产物是怎么避免请求瀑布的？

- **难度**: 中级
- **标签**: 构建, 性能
- **来源倾向**: 性能优化相关

**答案**

入口依赖的 chunk 在 HTML 中生成 modulepreload；动态导入被改写为 `__vitePreload`，加载 chunk 时同时并行加载它依赖的 JS 和 CSS。

详见：[中级 Q15](20-mid-qa.md)

---

### R17. 发版后用户访问报 chunk 加载失败，怎么处理？

- **难度**: 高级
- **标签**: 线上问题
- **来源倾向**: 场景题高频

**答案**

原因通常是旧页面加载已被删除的旧 chunk。监听 `vite:preloadError`，调用 `preventDefault` 后刷新页面，并加冷却时间防止无限刷新；根本上要保留旧资源、index.html 不缓存、资源 404 时不回退到 index.html。

**口述模板（30–60 秒）**

1. **一句话结论**：通常是发版删除了旧 chunk。
2. **关键机制**：Vite 提供 `vite:preloadError` 事件，可以捕获后刷新页面。
3. **对比或注意点**：要防止无限刷新，并且根本措施是保留旧资源。
4. **可追问时补充**：可以配合版本检测，主动提示用户刷新。

详见：[高级 Q15](30-senior-qa.md)、[手写 H11](40-coding-handwritten.md)

---

### R18. Vite 怎么处理 CommonJS 依赖？

- **难度**: 中级
- **标签**: 模块化
- **来源倾向**: 原理面

**答案**

开发时预构建转为 ESM，生产时 Rolldown 原生处理 CJS。主要问题是默认导入的互操作，Vite 8 统一了开发和生产的规则（`.mjs`、`type: module` 或没有 `__esModule` 标记时，default 为 `module.exports`）。

详见：[中级 Q19](20-mid-qa.md)

---

### R19. Vite 中 TypeScript 会做类型检查吗？

- **难度**: 初级
- **标签**: TypeScript
- **来源倾向**: 一面基础

**答案**

不会，只用 Oxc 单文件转译去掉类型，所以需要开启 `isolatedModules`。类型检查交给 `tsc --noEmit`、IDE 或 `vite-plugin-checker`。

详见：[初级 Q9](10-junior-qa.md)

---

### R20. Vite 的 SSR 是怎么实现的？

- **难度**: 高级
- **标签**: SSR
- **来源倾向**: SSR / 全栈方向

**答案**

开发时使用中间件模式，用 `ssrLoadModule` 或 Module Runner 加载服务端入口，代码经过同一套插件转换，支持 HMR；构建时分别构建客户端（生成 manifest）和服务端；运行时根据 manifest 注入资源。依赖默认 external，需要转换的依赖配置 `ssr.noExternal`。

详见：[中级 Q24](20-mid-qa.md)、[高级 Q8](30-senior-qa.md)、[高级 Q21](30-senior-qa.md)

---

### R21. 了解 Vite 的 Environment API 吗？

- **难度**: 高级
- **标签**: Environment API
- **来源倾向**: 高级岗、框架方向

**答案**

Vite 6 引入（实验性，计划在 Vite 9 稳定）。一个开发服务器中可以定义多个环境（client、ssr、edge），每个环境有独立的模块图和插件管线；配合 Module Runner，代码可以在非 Node 运行时（如 workerd）中执行，让开发环境更接近生产。主要面向框架和平台作者。

详见：[中级 Q25](20-mid-qa.md)、[系统设计 S8](50-system-design.md)

---

### R22. Vite 8 为什么要用 Rolldown？

- **难度**: 高级
- **标签**: Rolldown, 版本
- **来源倾向**: 高级岗、近期热点

**答案**

解决双打包器（esbuild + Rollup）导致的开发和生产不一致，以及 Rollup 生产构建慢的问题。Rolldown 是 Rust 实现，兼容 Rollup 插件 API，内置 CJS、resolve、TS 转换等能力。代价是 JS 插件有跨语言开销（需要钩子过滤器），部分 Rollup 钩子不支持。

**口述模板（30–60 秒）**

1. **一句话结论**：为了统一开发和生产的打包器，并大幅提升构建速度。
2. **关键机制**：Rolldown 是 Rust 实现，兼容 Rollup 插件生态。
3. **对比或注意点**：JS 插件跨语言调用有开销，需要钩子过滤器。
4. **可追问时补充**：迁移要点是 rolldownOptions、oxc、codeSplitting，大项目可以先用 rolldown-vite 过渡。

详见：[高级 Q6](30-senior-qa.md)、[中级 Q12](20-mid-qa.md)、[中级 Q32](20-mid-qa.md)

---

### R23. 如果让你把一个 webpack 项目迁移到 Vite，你会怎么做？

- **难度**: 高级
- **标签**: 迁移
- **来源倾向**: 高级岗系统设计

**答案**

先评估对 webpack 特有能力的依赖（依赖多时考虑 Rspack）；用 codemod 改造源码兼容两套构建；先切换开发环境，再切换生产构建；对比产物和 E2E 测试，灰度上线，监控错误率。

详见：[高级 Q19](30-senior-qa.md)、[系统设计 S1](50-system-design.md)

---

### R24. Vite 项目怎么接入 qiankun？有什么问题？

- **难度**: 高级
- **标签**: 微前端
- **来源倾向**: 中后台 / 微前端方向

**答案**

qiankun 通过 fetch + eval 执行子应用脚本，而 Vite 产物是 ESM，`import` 语法无法在 eval 中执行，所以需要社区插件改为动态 import 加载，代价是 JS 沙箱能力减弱。需要强隔离可以选无界（iframe 执行 JS），需要共享依赖可以选模块联邦。

详见：[高级 Q22](30-senior-qa.md)、[系统设计 S6](50-system-design.md)

---

### R25. 2026 年新项目怎么选构建工具？

- **难度**: 高级
- **标签**: 选型
- **来源倾向**: 高级岗、架构岗

**答案**

新的 SPA、组件库选 Vite；使用 Nuxt、SvelteKit、Astro、React Router 等框架时跟随框架（大多基于 Vite）；Next.js 使用 Turbopack；存量 webpack 大项目选 Rspack；纯 TS 库可以用 tsdown。评估维度：开发体验、构建速度、生态、迁移成本、团队熟悉度。

详见：[高级 Q24](30-senior-qa.md)、[高级 Q25](30-senior-qa.md)
