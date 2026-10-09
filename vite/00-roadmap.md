# Vite 学习路线（高级工程师面试向）

## 面试画像

国内考察 Vite 的岗位主要是**前端 / 前端工程化 / 前端基础架构**。Vite 已经是新项目的默认选择，面试通常分三层：

- **使用**：配置、环境变量、静态资源、CSS、代理、别名、构建输出、常用插件 —— 一面基础
- **原理**：为什么开发快（原生 ESM + 按需编译）、依赖预构建、裸模块重写、HMR 边界、插件机制与钩子顺序、生产构建流程、Vite 8 为什么换 Rolldown —— 中高级必问
- **工程与架构**：大型项目开发首屏慢、开发与生产不一致、写插件、SSR、Environment API、从 webpack 迁移、Monorepo、构建选型 —— 高级岗必谈

高级岗典型问法：

> 「Vite 为什么比 webpack 快？大项目还快吗？」「依赖预构建做了什么？为什么需要？」「Vite 的 HMR 和 webpack 有什么不同？」「Vite 插件和 Rollup 插件有什么区别？写过什么插件？」「Vite 8 换成 Rolldown 解决了什么问题？」「开发环境正常、生产环境出问题，你怎么排查？」

## 版本说明（截至 2026-10）

| 版本 | 时间 | 关键内容 |
|------|------|----------|
| Vite 2 | 2021-02 | 框架无关；兼容 Rollup 的插件 API；esbuild 依赖预构建 |
| Vite 3 / 4 | 2022 | Rollup 3；`@vitejs/plugin-react-swc` |
| Vite 5 | 2023-11 | Rollup 4；Node 18+；CJS Node API 废弃 |
| Vite 6 | 2024-11 | **Environment API**（实验性）与 Module Runner；Sass 默认使用 modern API |
| Vite 7 | 2025-06 | **Node 20.19+ / 22.12+**，Vite 本身只发布 ESM；`build.target` 默认 `'baseline-widely-available'`；移除 Sass legacy API 和 `splitVendorChunkPlugin` |
| rolldown-vite | 2025-05 起 | 技术预览包：Vite 7 + Rolldown，用于平滑迁移 |
| **Vite 8** | **2026-03-12** | **Rolldown 统一开发和生产打包**（取代 esbuild + Rollup），**Oxc** 负责转换和 JS 压缩，**Lightning CSS** 默认压缩 CSS；`build.rollupOptions` → `build.rolldownOptions`、`esbuild` → `oxc`（兼容层自动转换）；`manualChunks` 废弃，改用 Rolldown 的 `codeSplitting`；CJS 默认导入的互操作在开发和生产保持一致；内置 Devtools、`resolve.tsconfigPaths`、`emitDecoratorMetadata`、Wasm SSR、`server.forwardConsole`；默认 target 升级为 Chrome 111 / Safari 16.4 等；`@vitejs/plugin-react` v6 使用 Oxc，不再依赖 Babel |
| Rolldown 1.0 | 2026-05-06 | Rolldown 稳定版 |
| Vite 8.1 | 2026-06-23 | **实验性 Bundled Dev Mode**（原名 Full Bundle Mode）：`experimental.bundledDev: true`，开发时也打包，面向模块数量巨大的应用 |
| Vite 8.3 | 2026-09 | 当前维护的最新小版本（8.3.x） |
| Vite 9 | 规划中 | 计划稳定 Environment API |

> 生态背景：Vite、Rolldown、Oxc、Vitest 由 VoidZero 团队主导，形成统一工具链；2026 年 7 月 VoidZero 加入 Cloudflare，官方表示 Vite 的治理和方向不变。面试中能说清「Vite 8 为什么从双打包器改为 Rolldown」「Bundled Dev Mode 要解决什么问题」是加分项。具体配置以官方文档和迁移指南为准。

## 知识地图

```text
Vite
├─ 核心理念：开发时 no-bundle（原生 ESM 按需编译）、生产时打包；index.html 是入口
├─ 基础使用：vite.config、dev / build / preview、mode 与 .env、import.meta.env、静态资源（?url / ?raw / ?inline / ?worker）、public 目录
├─ 样式：CSS Modules、预处理器、PostCSS、Lightning CSS、CSS 代码分割
├─ 开发服务器：connect 中间件、插件容器、模块图、文件监听、WebSocket、代理、server.fs 安全
├─ 依赖预构建：扫描入口、CJS → ESM、合并小模块、node_modules/.vite 缓存、重新优化
├─ HMR：import.meta.hot、HMR 边界、模块失效、自定义事件、Fast Refresh
├─ 插件：Rollup / Rolldown 兼容钩子 + Vite 特有钩子（config、configureServer、transformIndexHtml、hotUpdate）、enforce / apply、虚拟模块、钩子过滤器
├─ 生产构建：Rolldown 打包、codeSplitting、modulepreload、资源 hash、manifest、库模式、legacy
├─ 进阶：SSR、Environment API、Module Runner、Bundled Dev Mode、Monorepo、微前端
└─ 生态：Rolldown、Oxc、Vitest、VitePress、tsdown；对比 webpack / Rspack / Turbopack
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 配置、命令、环境变量、静态资源、CSS、TS、代理、代码分割、库模式入门
2. [中级 Q&A](20-mid-qa.md) — 开发服务器请求处理流程、依赖预构建、HMR 原理、插件机制与钩子、生产构建、开发与生产差异、SSR 与 Environment API 入门、Vite 8 迁移
3. [高级 Q&A](30-senior-qa.md) — 源码架构、插件容器与模块图、HMR 传播算法、预构建深入、Rolldown 与 Bundled Dev Mode、性能排查、线上问题、插件设计、迁移与选型
4. [手写与现场编码](40-coding-handwritten.md) — 12 道题：迷你 Vite、HMR 客户端、各类插件、配置题等
5. [系统设计与场景](50-system-design.md) — webpack 迁移、构建平台、大型项目开发体验、SSR 框架、组件库、微前端等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. Vite 开发快的本质：不打包，浏览器按原生 ESM 请求，服务器按需编译；启动时间与项目规模基本无关
2. 依赖预构建的两个目的：CJS / UMD 转 ESM、把有大量内部模块的依赖合并以减少请求；缓存与失效机制
3. 裸模块导入（`import 'react'`）为什么需要重写，怎么重写
4. 一个模块请求在开发服务器中经历的流程：resolveId → load → transform → 导入分析 → 返回
5. HMR：文件变化 → 模块图找受影响模块 → 向上查找 accept 边界 → WebSocket 推送 → 浏览器带时间戳重新 import
6. 与 webpack HMR 的区别：不需要重新打包 chunk，更新速度与项目规模无关
7. 插件机制：兼容 Rollup 钩子 + Vite 特有钩子；`enforce` / `apply`；开发时哪些钩子会被调用
8. 虚拟模块、钩子过滤器、`transformIndexHtml`、`configureServer`、`hotUpdate`
9. 生产构建：Rolldown 打包、CSS 代码分割、modulepreload、资源 hash、Oxc 压缩
10. Vite 8 的意义：统一打包器，消除开发和生产的不一致、10–30 倍的构建提速
11. 开发与生产不一致的常见来源，以及排查方法
12. 环境变量：mode、`.env` 加载顺序、`VITE_` 前缀、静态替换、不能放密钥
13. 大型项目开发首次加载慢的原因（请求数、瀑布）和优化（warmup、治理 barrel、Bundled Dev Mode）
14. 频繁「new dependencies optimized」整页刷新的原因和处理
15. SSR：中间件模式、`ssrLoadModule` / Module Runner、`ssr.noExternal`、客户端 manifest
16. Environment API：一个开发服务器中同时管理多个运行环境（client / ssr / edge）
17. 生产 chunk 加载失败与 `vite:preloadError`
18. 选型：Vite、webpack、Rspack、Turbopack 的原理差异和迁移成本

## 相关技术

- [webpack](../webpack/00-roadmap.md)：模块图、chunk、Tree Shaking、HMR 等通用原理；与 Vite 对比阅读效果最好
- [JavaScript](../javascript/00-roadmap.md)：ESM、动态 import、`import.meta`
- [TypeScript](../typescript/00-roadmap.md)：`isolatedModules`、类型检查与转译分离
- [React](../react/00-roadmap.md)：Fast Refresh、懒加载、SSR
- [Node.js](../node/00-roadmap.md)：Vite 开发服务器运行在 Node 上；`require(esm)`
- [HTML](../html/00-roadmap.md)：`<script type="module">`、modulepreload

## 自检清单

- [ ] 能讲清 Vite 为什么开发时快，以及大项目中的局限
- [ ] 能讲清依赖预构建的目的、过程和缓存失效
- [ ] 能讲清一个模块请求在开发服务器中的处理流程
- [ ] 能讲清 Vite HMR 的完整链路，以及与 webpack 的区别
- [ ] 能手写一个虚拟模块插件和一个 transform 插件
- [ ] 能讲清 Vite 8 换成 Rolldown 的原因和迁移要点
- [ ] 能排查开发和生产行为不一致的问题
- [ ] 能讲清 SSR 和 Environment API 的基本原理
