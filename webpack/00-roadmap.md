# webpack 学习路线（高级工程师面试向）

## 面试画像

国内大厂考察 webpack 的岗位主要是**前端 / 前端工程化 / 前端基础架构**。尽管 Vite、Rspack 越来越普及，webpack 仍是面试中「工程化」部分的主线，因为它的概念（模块图、Chunk、Loader、Plugin、Tree Shaking、HMR、长效缓存）是所有打包器的共同基础。面试通常分三层：

- **配置与使用**：entry / output / loader / plugin / mode、常用 loader 和插件、devServer、source map、多环境配置 —— 一面基础
- **原理**：构建流程（初始化 → make → seal → emit）、Compiler 与 Compilation、Tapable、Loader 执行顺序与 pitch、打包产物与运行时、动态 import、HMR、Tree Shaking —— 中高级必问
- **优化与架构**：构建速度、产物体积、splitChunks 与长效缓存、持久化缓存、Module Federation 与微前端、发布与缓存策略、迁移 Vite / Rspack、构建平台治理 —— 高级岗必谈

高级岗典型问法：

> 「说一下 webpack 的构建流程。」「Loader 和 Plugin 的区别？写过吗？」「HMR 是怎么实现的？」「Tree Shaking 为什么没生效？」「你们项目构建从几分钟优化到多少？怎么做的？」「contenthash 为什么改一个文件很多 hash 都变了？」「为什么要迁移到 Rspack / Vite，迁移遇到了什么问题？」

## 版本说明（截至 2026-10）

| 版本 | 时间 | 关键内容 |
|------|------|----------|
| v4 | 2018-02 | 引入 `mode`；`splitChunks` 取代 `CommonsChunkPlugin`；零配置默认值 |
| v5 | 2020-10 | **持久化缓存** `cache: { type: 'filesystem' }`；**Module Federation**；**Asset Modules** 取代 file/url/raw-loader；Tree Shaking 增强（嵌套导出、inner graph、部分 CommonJS）；`moduleIds/chunkIds: 'deterministic'`；**移除 Node 核心模块自动 polyfill**；`[hash]` 改为 `[fullhash]` |
| 5.104–5.106 | 2025–2026 | `import.meta.env` 支持增强、`import.meta.filename/dirname` |
| 5.107 | 2026 | 实验性 `experiments.typescript`：使用 Node 的 `module.stripTypeScriptTypes` 擦除类型（Node 22.6+，只支持可擦除语法，不支持 `.tsx`、`enum`，不做类型检查） |
| 5.108 | 2026 | **lazy barrel**：无副作用的 barrel 文件中，未使用的重导出目标不再构建 |
| 5.109 | 2026 | `experiments.css` / `experiments.html` 默认为 `"auto"`（没有为该类文件注册 loader 时启用内置支持）；`import.meta.glob`；`import.meta.env` 默认值（`MODE`、`DEV`、`PROD` 等）；`?raw` / `?url` / `?inline` 资源后缀（futureDefaults 下） |
| 5.110 | 2026 | **性能提示**（重复包、循环依赖、未使用的 barrel 重导出、缓存效果等）；`import defer` / `import source` 支持完善 |
| **5.111** | **2026-09** | **ESM 输出稳定**：直接设置 `output.module: true`，`experiments.outputModule` 被移除；`futureDefaults` 在支持 ESM 的 target 上默认输出 ESM；内置复制静态文件；按浏览器目标处理 CSS；运行时体积和内存进一步减少 |
| v6 | 未发布 | 官方不公布时间。2026 路线图：CSS / HTML / TypeScript 进一步内置（目标是不再需要 html-webpack-plugin 等插件）、loader-runner 合并进核心、更通用的运行时目标（Node / Deno / Bun）、多线程探索 |

同类工具：

| 工具 | 现状 |
|------|------|
| **Rspack 2.0** | 2026-04 发布，Rust 实现、兼容 webpack 配置和大部分 loader / 插件；核心包改为纯 ESM，要求 Node 20.19+ / 22.12+；支持 React Server Components、Module Federation 内置。上层封装为 **Rsbuild 2.0** |
| **Vite** | 开发环境基于原生 ESM 按需编译；新版本使用 Rust 实现的 **Rolldown** 统一开发和生产打包（细节以官方为准） |
| **Turbopack** | Next.js 新版本中的默认打包器 |

> 面试中能说出「webpack 5 的持久化缓存、Module Federation、Asset Modules」是基本要求；能说出「5.10x 的内置 CSS/HTML/TS 支持和 ESM 输出」「Rspack 2.0 与 webpack 的兼容关系」是加分项。具体配置以官方 changelog 和迁移指南为准。

## 知识地图

```text
webpack
├─ 核心概念：entry、output、module、chunk、bundle、loader、plugin、mode、target
├─ 资源处理：babel-loader / swc-loader、ts-loader、css-loader / style-loader / MiniCssExtractPlugin、PostCSS、Asset Modules、html-webpack-plugin
├─ 开发体验：webpack-dev-server、HMR、React Fast Refresh、source map、proxy、lazyCompilation
├─ 模块解析：enhanced-resolve、alias、extensions、mainFields、exports 条件导出、symlinks
├─ 构建流程：Compiler → Compilation → make（构建模块图）→ seal（优化、分 Chunk、生成代码）→ emit
├─ 扩展机制：Tapable（Sync/Async、Bail/Waterfall/Loop）、Loader（normal / pitch、loader-runner）、Plugin（compiler / compilation hooks、processAssets）
├─ 产物与运行时：__webpack_require__、模块缓存、JSONP 分块加载、publicPath、runtimeChunk、ESM 输出
├─ 优化：Tree Shaking（usedExports + sideEffects + 压缩）、Scope Hoisting、splitChunks、长效缓存（contenthash、deterministic ids）、压缩、预加载
├─ 构建速度：持久化缓存、swc/esbuild 转译、thread-loader、缩小解析范围、lazyCompilation、分析工具
├─ 进阶：Module Federation、打包库（library、externals）、SSR 双构建、微前端、Monorepo
└─ 生态与选型：Vite（Rolldown）、Rspack / Rsbuild、Turbopack、esbuild、Rollup
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 核心概念、常用 loader 和插件、devServer、source map、代码分割入门
2. [中级 Q&A](20-mid-qa.md) — 构建流程、Compiler 与 Compilation、Tapable、Loader 原理、产物与运行时、HMR、Tree Shaking、splitChunks、长效缓存、构建提速、Module Federation
3. [高级 Q&A](30-senior-qa.md) — 模块图与 seal 阶段、Tapable 实现、Tree Shaking 深入、hash 稳定性、持久化缓存原理、构建性能排查、线上问题、插件设计、迁移 Rspack / Vite、构建选型
4. [手写与现场编码](40-coding-handwritten.md) — 12 道题：迷你打包器、Loader、Plugin、Tapable、运行时、配置题等
5. [系统设计与场景](50-system-design.md) — 构建提速、构建平台、微前端、发布与缓存、组件库、迁移方案等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. webpack 的定位：以模块图为核心的静态打包器，一切资源都是模块
2. module、chunk、bundle 的区别；hash、chunkhash、contenthash 的区别
3. 构建流程：初始化参数 → 创建 Compiler → 注册插件 → run → compile → make（从入口递归构建）→ seal → emit → done
4. Compiler 与 Compilation 的区别；watch 模式下 Compilation 会重新创建
5. Tapable 的钩子类型以及 `tap` / `tapAsync` / `tapPromise`
6. Loader 的本质（函数，输入源码输出源码）、执行顺序（从右到左）、pitch 阶段与熔断
7. Plugin 的本质（带 `apply(compiler)` 的对象）、常用钩子（`compilation`、`processAssets`、`emit`、`done`）
8. 打包产物结构：`__webpack_require__`、模块缓存、`__webpack_require__.e` 动态加载 Chunk
9. HMR 全链路：文件监听 → 增量编译 → WebSocket 推送 hash → 客户端拉取 manifest 和更新 Chunk → 冒泡查找 accept → 替换或刷新
10. Tree Shaking 三要素：ESM 静态结构、`usedExports` 标记、压缩器删除；`sideEffects` 的作用和误删 CSS 问题
11. Scope Hoisting 的原理与作用
12. splitChunks 的默认规则、cacheGroups 设计，以及拆分过细的问题
13. 长效缓存：contenthash、`runtimeChunk`、`moduleIds: 'deterministic'`、`realContentHash`
14. 构建提速：持久化缓存、swc/esbuild 替代 Babel、缩小范围、lazyCompilation、分析定位
15. 持久化缓存的失效机制：`buildDependencies`、`version`、snapshot
16. Module Federation 的原理：remoteEntry、shared 单例与版本协商、异步边界
17. 发布与缓存：HTML 不缓存、静态资源强缓存、ChunkLoadError 处理
18. 选型：webpack、Vite、Rspack 的原理差异与迁移成本

## 相关技术

- [JavaScript](../javascript/00-roadmap.md)：ESM 与 CommonJS、动态 import、闭包（运行时实现）
- [TypeScript](../typescript/00-roadmap.md)：tsconfig、模块解析、类型检查与转译分离
- [React](../react/00-roadmap.md)：Fast Refresh、懒加载、SSR 构建
- [CSS](../css/00-roadmap.md)：CSS Modules、PostCSS、关键 CSS
- [HTML](../html/00-roadmap.md)：资源提示（preload / prefetch）、script 加载策略
- [Node.js](../node/00-roadmap.md)：webpack 本身运行在 Node 上，构建性能排查与 Node 相同

## 自检清单

- [ ] 能不看资料讲清 webpack 的完整构建流程
- [ ] 能手写一个 Loader 和一个 Plugin，并说清各自适用的场景
- [ ] 能解释打包产物的结构，以及动态 import 是怎么加载 Chunk 的
- [ ] 能讲清 HMR 的完整链路
- [ ] 能排查 Tree Shaking 不生效的原因
- [ ] 能为一个中大型项目设计 splitChunks 和长效缓存方案
- [ ] 能讲一次构建提速或体积优化的完整过程（有数据）
- [ ] 能讲清 webpack、Vite、Rspack 的差异和迁移方案
