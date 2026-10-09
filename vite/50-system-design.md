# Vite 系统设计与场景

> 回答结构：先澄清需求和约束 → 给出整体方案（文字图）→ 讲关键设计和取舍 → 用指标证明效果 → 回答扩展追问。

---

### S1. 大型 webpack 项目迁移到 Vite

- **难度**: 高级
- **标签**: 迁移, 工程化

**澄清问题（面试时先问）**

- 项目规模：模块数、页面数、开发人数？
- 当前痛点：开发启动慢、HMR 慢还是构建慢？
- 依赖了哪些 webpack 特有能力：自定义 loader、模块联邦、`require.context`？
- 是否有 SSR、微前端？
- 浏览器兼容要求？

**参考方案**

```text
阶段 0：评估
  ├─ 扫描 webpack 配置：loader、plugin、resolve、DefinePlugin
  ├─ 扫描源码：require、require.context、process.env、~ 导入、隐式扩展名
  └─ 输出迁移清单和工作量；webpack 特有能力过多时，评估改用 Rspack

阶段 1：源码兼容（webpack 仍在用）
  ├─ codemod：require → import，require.context → import.meta.glob
  ├─ 环境变量统一封装为 env.ts，两套构建都能工作
  └─ 消除循环依赖

阶段 2：Vite 开发环境
  ├─ index.html 移到根目录，配置别名、代理、define
  ├─ 自定义 loader 改写为 Vite 插件
  └─ 开发者可选使用 Vite 开发，webpack 仍负责生产构建

阶段 3：Vite 生产构建
  ├─ 对比两套产物：体积、chunk、关键页面 E2E 测试
  └─ 灰度发布：按比例或按用户切换产物

阶段 4：移除 webpack
```

**关键设计**

- **源码兼容层先行**：先让源码同时兼容两套构建，可以随时回退
- **开发先于生产**：开发环境收益最大、风险最低
- **自动化**：codemod 批量改写，避免手工修改几千个文件
- **验证**：E2E 测试 + 视觉回归 + 灰度监控错误率

**关键指标**

- 开发启动时间、HMR 耗时、生产构建时间
- 灰度期间 JS 错误率、白屏率
- 产物体积变化

**口述模板（2–3 分钟）**

我会先评估项目对 webpack 特有能力的依赖程度，如果自定义 loader 和模块联邦用得很多，我会建议迁移 Rspack，因为成本低得多。确定迁移 Vite 后分四个阶段：第一阶段在不改变构建工具的前提下改造源码，用 codemod 把 require、require.context 和 process.env 改成 ESM 写法，并消除循环依赖，这样源码同时兼容两套构建；第二阶段先让开发环境切到 Vite，因为开发体验收益最大、风险最低，生产仍用 webpack；第三阶段生产切换，对比两套产物的体积和 E2E 结果，再灰度发布、监控错误率；最后移除 webpack。衡量指标是开发启动时间、HMR 时间、构建时间和灰度期间的错误率。

**扩展追问**

- 开发用 Vite、生产用 webpack 会不会有不一致？→ 会，所以这只是过渡阶段，E2E 测试必须跑在生产产物上。
- 迁移期间新代码怎么约束？→ ESLint 规则禁止 require 和 process.env。

---

### S2. 设计企业级 Vite 构建平台 / 脚手架

- **难度**: 高级
- **标签**: 工程化, 平台

**澄清问题（面试时先问）**

- 有多少项目？技术栈是否统一？
- 需要统一哪些能力：构建、规范、发布、监控？
- 业务方需要多大的定制空间？

**参考方案**

```text
@company/create-app       脚手架：选择模板生成项目
@company/vite-config      预设配置：defineCompanyConfig(options)
  ├─ 基础配置：别名、代理、环境变量规范、分包策略、source map
  ├─ 内置插件：构建信息、版本检测、监控注入、CDN 上传
  └─ 扩展点：业务通过 mergeConfig 或 options 定制
@company/vite-plugin-*    公司插件：mock、权限路由、国际化等
CI 模板                    lint、类型检查、测试、构建、体积预算、发布
```

```ts
export default defineCompanyConfig({
  app: 'order-admin',
  cdn: true,
  mock: true,
  vite: { server: { port: 3001 } },
})
```

**关键设计**

- **配置分层**：公司默认 → 团队预设 → 项目配置，用 `mergeConfig` 合并
- **版本管理**：预设包遵循语义化版本，Vite 大版本升级在预设中统一完成，业务只需升级预设版本
- **可观测**：构建插件上报构建耗时、产物体积，形成大盘
- **逃生通道**：允许业务完全覆盖配置，避免平台成为瓶颈

**关键指标**

- 新项目创建到可运行的时间
- Vite 大版本升级的覆盖率和耗时
- 构建耗时、产物体积大盘

**口述模板（2–3 分钟）**

我会把平台分为脚手架、预设配置包、公司插件和 CI 模板四部分。核心是预设配置包，它导出 defineCompanyConfig，内置别名、代理、环境变量规范、分包、source map 上传、版本检测这些通用能力，业务通过参数或 mergeConfig 定制。这样 Vite 升级到 8 这类工作可以在预设包中统一完成，业务只需要升级版本。平台插件会上报构建耗时和产物体积，形成大盘，配合 CI 中的体积预算防止劣化。同时保留逃生通道，允许业务完全覆盖配置。

**扩展追问**

- 业务需要的配置与预设冲突怎么办？→ 提供明确的覆盖顺序，冲突时以项目配置为准，并在文档中说明。
- 怎么推动所有项目升级？→ 大盘展示各项目版本，提供 codemod，并设定旧版本的停止支持时间。

---

### S3. 大型应用的开发体验优化

- **难度**: 高级
- **标签**: 性能, 开发体验

**澄清问题（面试时先问）**

- 慢在哪里：服务器启动、首次打开页面、HMR，还是刷新？
- 模块数量、首屏请求数？
- 开发环境：本地、容器、远程开发机？

**参考方案**

```text
度量：埋点收集启动时间、首屏加载时间、HMR 耗时 → 大盘

请求数治理
  ├─ ESLint 禁止从 barrel 文件导入（或用插件改写为直接路径）
  ├─ 图标、组件库按需导入
  └─ 路由懒加载

预构建稳定
  ├─ optimizeDeps.include 覆盖动态依赖
  └─ optimizeDeps.entries 覆盖所有页面

转换加速
  ├─ 插件审计：加过滤器、移除不必要的插件
  └─ server.warmup 预热常用文件

终极方案
  └─ 试点 Bundled Dev Mode（Vite 8.1 实验性）
```

**关键设计**

- **先度量后优化**：在 Vite 插件中统计并上报各阶段耗时
- **从源头减少模块数**：barrel 文件是大项目的主要问题
- **避免整页刷新**：预构建稳定 + HMR 边界规范（组件文件只导出组件）
- **环境问题**：容器中开启轮询，远程开发时注意网络延迟

**关键指标**

- 冷启动 / 热启动时间
- 首次打开页面的请求数和耗时
- HMR P90 耗时、整页刷新次数

**口述模板（2–3 分钟）**

我会先做度量，通过插件上报启动时间、首屏加载时间和 HMR 耗时，确定瓶颈。大项目最常见的问题是首屏请求数过多，原因主要是 barrel 文件和全量导入，所以第一步用 ESLint 规则或改写插件消除 barrel 导入，配合路由懒加载。第二步保证预构建稳定，用 include 和 entries 避免运行时发现依赖导致的整页刷新。第三步审计插件耗时，加钩子过滤器，用 warmup 预热常用文件。如果做完这些模块数仍然很大，就在部分开发者中试点 Bundled Dev Mode。最终看请求数、首屏时间和 HMR P90 的变化。

**扩展追问**

- 为什么不直接上 Bundled Dev Mode？→ 仍是实验性，插件兼容性未知；而且源头治理（barrel）对生产也有收益。

---

### S4. 基于 Vite 设计 SSR 框架

- **难度**: 高级
- **标签**: SSR, 框架

**澄清问题（面试时先问）**

- 渲染模式：SSR、SSG、流式、ISR？
- 部署目标：Node 服务器、Serverless、边缘？
- 路由方式：约定式还是配置式？
- 数据获取方式？

**参考方案**

```text
开发：
  Node 服务器 ── vite.middlewares（客户端资源、HMR）
            └─ 路由匹配 → Module Runner 加载 entry-server → render(url) → HTML

构建：
  client 环境 → dist/client（含 manifest）
  ssr 环境    → dist/server/entry-server.js
  （Environment API：builder.buildApp 统一构建）

运行：
  请求 → 路由匹配 → loader 获取数据 → renderToPipeableStream
       → 根据 manifest 注入当前路由的 JS / CSS / preload
       → 数据序列化注入 HTML → 客户端 hydrate
```

**关键设计**

- **路由**：`import.meta.glob` 生成路由表，服务端和客户端共享
- **资源注入**：manifest 记录路由对应的 chunk，服务端渲染时输出 CSS 和 modulepreload，避免样式闪烁和请求瀑布
- **数据**：路由级 loader，服务端执行后序列化（注意转义防 XSS），客户端注水复用
- **服务端 HMR**：Module Runner 在模块变化时自动失效
- **多运行时**：通过 Environment API 定义 edge 环境，开发时直接运行在目标运行时
- **错误处理**：`ssrFixStacktrace` 修正堆栈；渲染失败降级为 CSR

**关键指标**

- TTFB、FCP、LCP
- 服务端渲染耗时、错误率、降级率

**口述模板（2–3 分钟）**

开发时我会用 Vite 的中间件模式，客户端资源和 HMR 由 Vite 中间件处理，页面请求经过路由匹配后通过 Module Runner 加载服务端入口并渲染，这样服务端代码也支持 HMR。构建时分为 client 和 ssr 两个环境，client 产物生成 manifest。运行时，服务端执行路由 loader 获取数据，流式渲染，根据 manifest 注入当前路由需要的 CSS 和 modulepreload，数据序列化后注入页面供客户端注水。如果需要部署到边缘，通过 Environment API 定义 edge 环境，开发时就运行在目标运行时中。错误时降级为客户端渲染，指标看 TTFB、LCP 和降级率。

**扩展追问**

- 流式渲染时 CSS 怎么处理？→ 在 shell 中输出首屏路由的 CSS，懒加载部分的 CSS 随 Suspense 边界输出。

---

### S5. 组件库与文档站的工程化方案

- **难度**: 中级
- **标签**: 组件库, 工程化

**澄清问题（面试时先问）**

- 组件数量、使用方的技术栈和构建工具？
- 是否需要按需加载、主题定制？
- 是否需要同时支持 ESM 和 CJS？

**参考方案**

```text
packages/
  ui/            Vite 库模式：ESM、preserveModules、类型声明
  icons/         SVG → 组件，按需导出
  theme/         CSS 变量主题
docs/            VitePress 或 Storybook 文档站
测试             Vitest + Testing Library；浏览器模式测试交互；视觉回归
发布             Changesets 管理版本和变更日志
```

**关键设计**

- **按需加载**：ESM + `preserveModules` + 正确的 `sideEffects`，使用方无需插件即可 Tree Shaking
- **样式方案**：CSS 变量做主题；每个组件的样式独立文件，或使用零运行时 CSS-in-JS
- **类型**：`vite-plugin-dts` 生成类型，检查导出的类型完整性
- **兼容性**：只输出 ESM 越来越常见；使用方是旧版 webpack 时需要评估
- **质量**：组件测试、视觉回归、体积预算

**关键指标**

- 单组件引入后的体积
- 构建时间、测试覆盖率
- 使用方升级成本（破坏性变更次数）

**口述模板（2–3 分钟）**

我会用 Monorepo 组织组件、图标、主题和文档。组件库用 Vite 库模式构建，只输出 ESM，开启 preserveModules，并在 package.json 中正确声明 sideEffects，这样使用方不需要按需加载插件就能 Tree Shaking。主题用 CSS 变量，类型用 vite-plugin-dts 生成。文档站用 VitePress 或 Storybook，测试用 Vitest 加浏览器模式和视觉回归。版本用 Changesets 管理。指标看单组件引入体积和使用方升级成本。

**扩展追问**

- 纯 TS 工具包也用 Vite 库模式吗？→ 可以考虑 tsdown，更专注于库打包，配置更简单。

---

### S6. Vite 项目的微前端方案

- **难度**: 高级
- **标签**: 微前端

**澄清问题（面试时先问）**

- 子应用技术栈是否统一？
- 隔离要求：JS 沙箱、样式隔离？
- 是否需要共享依赖？
- 是否已有 qiankun 体系？

**参考方案**

```text
方案 A：模块联邦（技术栈统一，需要共享依赖）
  主应用（@module-federation/vite）
    ├─ shared: react、react-dom、公司组件库（singleton）
    └─ 运行时 loadRemote('order/App')

方案 B：无界 / iframe（强隔离、技术栈不统一）
  主应用 → 子应用运行在 iframe 中，DOM 通过 Web Component 渲染

方案 C：qiankun + 社区插件（已有体系）
  子应用生命周期挂到全局，动态 import 加载；JS 沙箱能力减弱
```

**关键设计**

- **ESM 与沙箱的冲突**：ESM 不能被 eval，Proxy 沙箱失效，需要接受或用 iframe 隔离
- **共享依赖**：React 必须单例，版本需要协调
- **样式隔离**：CSS Modules、命名前缀或 Shadow DOM
- **部署**：子应用独立部署，主应用通过配置中心获取子应用地址，支持灰度
- **容错**：子应用加载失败时降级展示

**关键指标**

- 子应用加载时间
- 独立部署频率
- 子应用故障对主应用的影响

**口述模板（2–3 分钟）**

首先要说明 Vite 产物是 ESM，与 qiankun 基于 eval 的 JS 沙箱不兼容。所以选型时看隔离需求：技术栈统一且需要共享依赖，我会选模块联邦，用 @module-federation/vite，React 和组件库设置为单例；需要强隔离或技术栈不统一时选无界，它用 iframe 执行 JS，天然支持 ESM；如果已有 qiankun 体系，可以用社区插件过渡，但要接受沙箱能力减弱，并用 CSS Modules 或前缀做样式隔离。子应用地址由配置中心下发，支持灰度和降级。

**扩展追问**

- 模块联邦的共享依赖版本冲突怎么办？→ 设置 `requiredVersion`，版本不兼容时加载自己的副本或告警。

---

### S7. 前端 Mock 与联调平台

- **难度**: 中级
- **标签**: 开发效率

**澄清问题（面试时先问）**

- 接口定义来源：OpenAPI、手写？
- 需要支持哪些场景：本地开发、测试、演示？
- 是否需要在浏览器中直接切换真实接口和 mock？

**参考方案**

```text
OpenAPI 文档 → 生成 TS 类型 + 请求函数 + 默认 mock 数据

本地开发
  ├─ 方案 1：Vite 插件（configureServer 中间件），mock 文件热更新
  └─ 方案 2：MSW（Service Worker 拦截），与构建工具无关，测试中可复用

控制面板（开发时注入页面）
  ├─ 按接口切换：mock / 真实 / 延迟 / 错误
  └─ 场景预设：空数据、异常、超长列表

测试：Vitest / Playwright 复用同一套 mock handlers
```

**关键设计**

- **单一数据源**：类型和 mock 都从 OpenAPI 生成，接口变化时自动同步
- **mock 热更新**：插件中使用 `ssrLoadModule` 加载 mock 文件，修改后不需要重启
- **可切换**：按接口粒度控制，方便后端接口逐个联调
- **复用**：MSW 的 handlers 可以同时用于开发、单元测试和 E2E 测试
- **安全**：mock 插件 `apply: 'serve'`，确保不进入生产

**关键指标**

- 前后端并行开发的比例
- 联调阶段的问题数
- mock 维护成本

**口述模板（2–3 分钟）**

我会以 OpenAPI 为单一数据源，自动生成 TS 类型、请求函数和默认 mock 数据。本地开发可以用 Vite 插件在 configureServer 中注册中间件拦截请求，mock 文件通过 ssrLoadModule 加载，修改后自动生效；如果希望测试也复用，就用 MSW。开发时注入控制面板，按接口切换 mock 还是真实接口，并预设空数据、异常等场景，方便后端逐个接口联调。mock 插件只在开发时生效，不会进入生产。

**扩展追问**

- Vite 插件方案和 MSW 怎么选？→ 只需要开发时用选插件；需要测试复用、在 Network 面板中看到真实请求的话选 MSW。

---

### S8. 多运行时应用的构建设计（浏览器 + Node + 边缘）

- **难度**: 高级
- **标签**: Environment API, 架构

**澄清问题（面试时先问）**

- 有哪些运行环境：浏览器、Node SSR、边缘函数、Web Worker？
- 各环境之间共享哪些代码？
- 开发时是否要求与生产运行时一致？

**参考方案**

```ts
export default defineConfig({
  environments: {
    client: {
      build: { outDir: 'dist/client' },
    },
    ssr: {
      build: { outDir: 'dist/server' },
    },
    edge: {
      resolve: {
        conditions: ['workerd', 'worker', 'browser'],
        noExternal: true, // 边缘环境没有 node_modules，依赖需要全部打包
      },
      build: { outDir: 'dist/edge' },
    },
  },
  builder: {
    // 统一构建所有环境
    async buildApp(builder) {
      await Promise.all(
        Object.values(builder.environments).map((env) => builder.build(env)),
      )
    },
  },
})
```

```text
插件按环境生效：
  - 只对 client：applyToEnvironment(env) => env.name === 'client'
  - 在 hook 中用 this.environment 区分处理

开发：
  client → 浏览器
  ssr    → Node（Module Runner）
  edge   → workerd（例如 Cloudflare Vite 插件提供的运行时）
```

**关键设计**

- **环境隔离**：每个环境有独立的模块图、解析条件和插件管线，避免 Node API 被打进浏览器代码
- **共享代码**：业务逻辑放在与运行时无关的包中，运行时相关部分通过条件导出区分
- **开发与生产一致**：边缘代码开发时就运行在 workerd 中，提前发现不支持的 API
- **构建编排**：`buildApp` 控制构建顺序（例如 SSR 构建依赖 client 的 manifest 时需要串行）

**关键指标**

- 各环境产物体积（边缘函数通常有体积限制）
- 开发与生产行为差异导致的问题数

**口述模板（2–3 分钟）**

我会使用 Vite 的 Environment API，在一个配置中定义 client、ssr 和 edge 三个环境，每个环境有独立的模块图、解析条件和输出目录。边缘环境没有 node_modules，所以依赖全部打包，并使用 workerd、worker 等解析条件。插件通过 applyToEnvironment 或 this.environment 区分环境。开发时边缘代码通过 Cloudflare 等平台提供的插件直接运行在 workerd 中，与生产一致。构建时用 buildApp 统一编排，如果 SSR 需要 client 的 manifest 就串行构建。需要注意 Environment API 目前仍是实验性，计划在 Vite 9 稳定。

**扩展追问**

- 为什么开发时要运行在真实运行时？→ 边缘运行时不支持很多 Node API，在 Node 中开发会掩盖这些问题，上线后才暴露。
