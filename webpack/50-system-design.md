# webpack 系统设计题（8 道）

> 回答结构：澄清现状与目标 → 度量 → 方案 → 关键设计 → 风险与回滚 → 效果与演进。
>
> 前端工程化的系统设计题，面试官最看重两点：**有没有数据**（优化前后的指标）和**有没有考虑落地**（灰度、回滚、团队推广）。每道题都附有「关键指标」。

---

### S1. 大型项目构建太慢，设计一个提速方案

- **难度**: senior
- **标签**: 构建速度

**澄清问题（面试时先问）**

- 慢在哪个场景：开发启动、HMR、生产构建，还是 CI 总耗时？
- 项目规模：模块数、入口数、代码行数？
- 当前工具链版本：webpack 4 还是 5？Babel 还是 swc？
- 能接受的改动范围：只调配置，还是可以更换工具？

**参考方案**

```text
度量
  ├─ 开发：启动时间、HMR 时间（P50 / P90，采集开发者本地数据）
  ├─ 生产：冷构建、热构建（有缓存）、CI 端到端耗时
  └─ 分析：ProfilingPlugin、stats timings、CPU profile

第一层：低成本配置优化（1–2 周）
  ├─ 持久化缓存 + CI 缓存 .cache 目录
  ├─ babel-loader → swc-loader；类型检查异步 / 移到 CI
  ├─ Terser → SWC minify；CSS → lightningcss
  ├─ 缩小范围：include、extensions、noParse
  └─ 开发：lazyCompilation、cheap source map、关闭生产优化

第二层：工程结构优化（2–4 周）
  ├─ 治理 barrel 文件和全量导入（图标库、工具库）
  ├─ 拆分超大入口；多入口按需启动
  └─ Monorepo：Turborepo / Nx 增量构建 + 远程缓存

第三层：更换工具（1–2 月）
  └─ 迁移 Rspack：先开发环境，再生产环境；产物对比 + 灰度
```

**关键设计**

- **先度量再优化**：每项优化单独验证，记录前后数据
- **按投入产出比排序**：缓存和转译器替换收益最大、风险最低
- **CI 缓存的 key**：lock 文件 hash + 构建配置 hash，避免拿到错误缓存
- **迁移 Rspack 的风险控制**：自研插件兼容性检查、产物对比、E2E 回归、灰度
- **防止回退**：构建耗时纳入 CI 监控，超过阈值报警

**关键指标**

- 开发启动时间、HMR P90、生产冷 / 热构建时间、CI 总耗时、缓存命中率

**口述模板（2–3 分钟）**

我会先明确慢在哪个场景，并建立度量：开发启动、HMR、生产冷构建和热构建分别采集数据，用 ProfilingPlugin 和 stats 找到热点。优化分三层推进。第一层是低成本的配置优化：开启持久化缓存，并在 CI 中按 lock 文件和配置 hash 缓存；用 swc 替换 Babel，类型检查移到独立进程；压缩换成 SWC 和 lightningcss；开发环境开启 lazyCompilation。这一层通常就能把构建时间减半。第二层是工程结构的治理，比如消除全量导入图标库、barrel 文件，Monorepo 中用 Turborepo 做增量构建和远程缓存。如果仍然不满足，第三层迁移 Rspack，它兼容 webpack 配置，先在开发环境使用，再通过产物对比和 E2E 回归后灰度到生产。最后把构建耗时纳入 CI 监控，防止退化。

**扩展追问**

- 只能做一件事，你选什么？→ 持久化缓存（含 CI 缓存）；如果缓存已经开启，就换 swc。
- 优化后如何证明价值？→ 换算成开发者等待时间和 CI 资源成本，例如每天节省的总等待时间。

---

### S2. 设计一个统一的前端构建平台（服务 100+ 项目）

- **难度**: senior
- **标签**: 工程化, 平台

**澄清问题（面试时先问）**

- 项目类型：SPA、多页、SSR、组件库、小程序？技术栈是否统一？
- 现状：每个项目自己维护 webpack 配置？版本分布？
- 构建在哪里执行：开发者本地 + CI，还是有统一的构建集群？
- 发布流程：构建产物如何部署到 CDN？

**参考方案**

```text
开发者
  ↓ 使用
构建 CLI（统一封装）
  ├─ 预设：spa / mpa / ssr / lib（基于 Rspack 或 Rsbuild）
  ├─ 项目配置：app.config.ts（只写业务相关的少量配置）
  ├─ 扩展点：modifyConfig、插件机制（受控）
  └─ 内置：缓存、拆包、压缩、source map、体积预算、版本注入
  ↓
CI 构建
  ├─ 依赖安装缓存 + 构建缓存
  ├─ 产物检查：体积预算、重复依赖、密钥扫描、构建确定性
  ├─ source map 上传监控平台后删除
  └─ 构建数据上报
  ↓
发布
  ├─ 静态资源增量上传 CDN（不删除旧版本）
  └─ HTML 更新（灰度 / 回滚）
  ↓
数据平台：构建耗时、产物体积、缓存命中率、版本分布
```

**关键设计**

- **配置分层**：平台预设 → 团队预设 → 项目配置，项目只写差异
- **受控扩展**：提供 `modifyConfig` 钩子，但记录哪些项目使用了扩展，便于升级时评估影响
- **版本管理**：CLI 语义化版本，升级提供 codemod；数据平台展示各项目使用的版本，推动升级
- **底层工具可替换**：项目配置不直接暴露 webpack 配置，平台可以把底层从 webpack 切换到 Rspack 而不影响业务
- **统一的质量门禁**：体积、重复依赖、安全扫描在 CI 中统一执行

**关键指标**

- 项目接入率、CLI 版本分布、平均构建耗时、产物体积趋势、构建失败率

**口述模板（2–3 分钟）**

核心思路是「配置收敛 + 数据驱动」。我会提供一个统一的构建 CLI，内置 SPA、多页、SSR、库等预设，底层基于 Rspack 或 Rsbuild，缓存、拆包、压缩、source map、体积预算都作为默认能力，项目只需要写少量业务配置；对于特殊需求，提供受控的 modifyConfig 扩展点。项目配置不直接暴露底层工具，这样平台可以平滑地把底层从 webpack 切换到 Rspack。CI 中统一执行质量门禁：体积预算、重复依赖、密钥扫描，source map 上传监控后删除。发布时静态资源增量上传，保留旧版本，HTML 支持灰度和回滚。所有构建数据上报到平台，按项目展示耗时和体积趋势、CLI 版本分布，用数据推动升级和治理。

**扩展追问**

- 有项目不愿意接入怎么办？→ 用数据证明收益（构建时间、体积）；先从新项目和痛点项目开始，提供迁移工具和支持。
- 扩展点被滥用怎么办？→ 统计扩展使用情况，把高频需求沉淀为平台能力。

---

### S3. 基于 Module Federation 设计微前端架构

- **难度**: senior
- **标签**: 微前端, Module Federation

**澄清问题（面试时先问）**

- 拆分的原因：团队独立发布？技术栈不同？应用规模太大？
- 子应用数量和团队数量？
- 技术栈是否统一（都是 React）？
- 需要应用级隔离（JS 沙箱、样式隔离）还是组件级共享？

**参考方案**

```text
用户 → 主应用（host：路由、布局、登录、权限）
           │ 运行时从配置中心获取 remote 列表和版本
           ├─ 订单（remote）  ── orders/v1.3.2/mf-manifest.json
           ├─ 商品（remote）  ── products/v2.0.1/mf-manifest.json
           └─ 报表（remote）  ── reports/v0.9.0/mf-manifest.json

共享依赖（singleton）：react、react-dom、react-router、设计系统、状态管理
基础能力（由 host 提供或共享库）：用户信息、权限、埋点、请求封装

发布：
  remote 独立构建 → 版本化目录上传 CDN（不可变）→ 配置中心修改版本 → 灰度 → 全量
  回滚：配置中心改回上一个版本
```

**关键设计**

- **版本化部署 + 配置中心**：remote 每个版本放在不可变的目录中，host 通过配置中心决定加载哪个版本，实现灰度和秒级回滚
- **共享依赖治理**：React、路由、设计系统设置 singleton；共享依赖的大版本升级由平台统一组织
- **契约**：remote 暴露的模块视为公共 API，类型通过 MF 2.0 的类型提示同步，破坏性改动需要升级版本号
- **容错**：每个 remote 包裹 ErrorBoundary，加载失败显示降级 UI；加载重试；监控各 remote 的加载成功率
- **样式隔离**：CSS Modules 或 CSS-in-JS，禁止全局样式
- **性能**：mf-manifest 预加载；共享依赖避免重复；首屏只加载必要的 remote
- **本地开发**：MF 开发者工具把线上 remote 替换为本地版本

**关键指标**

- 各 remote 的加载成功率和耗时、共享依赖重复率、发布频率、回滚时长

**口述模板（2–3 分钟）**

如果团队技术栈统一都是 React，我会选择 Module Federation 而不是 iframe 或 qiankun 这类应用级方案，因为它可以共享依赖、粒度更灵活。主应用负责路由、布局、登录和权限，各业务团队的应用作为 remote 独立构建和发布。关键是版本化部署：每个 remote 版本放在不可变的 CDN 目录中，host 在运行时从配置中心获取每个 remote 的版本地址，这样可以灰度发布并且秒级回滚。共享依赖方面，React、路由、设计系统都设置 singleton，大版本升级由平台统一组织。remote 暴露的模块当作公共 API 来管理，通过 MF 2.0 的类型提示同步类型。容错方面每个 remote 有 ErrorBoundary 和降级 UI，并监控加载成功率。样式用 CSS Modules 隔离，本地开发用 MF 的开发者工具替换线上 remote。

**扩展追问**

- 共享依赖版本不一致导致线上问题？→ 在 CI 中检查各应用 shared 依赖的版本范围是否兼容；singleton 不满足 requiredVersion 时上报告警。
- 什么情况不适合 MF？→ 技术栈不统一、需要强隔离（第三方应用接入）时，iframe 或 qiankun 更合适。

---

### S4. 设计前端的发布与缓存策略

- **难度**: senior
- **标签**: 发布, 缓存, 高频

**澄清问题（面试时先问）**

- 部署方式：静态资源放在 CDN 还是服务器？HTML 由谁提供（Nginx、Node SSR、CDN）？
- 发布频率？是否需要灰度和快速回滚？
- 是否有 Service Worker？

**参考方案**

```text
构建：
  JS / CSS / 图片：[name].[contenthash:8].ext
  HTML：index.html（不带 hash）
  version.json

发布顺序：
  1. 静态资源增量上传 CDN（不覆盖、不删除，保留最近 N 个版本 / 30 天）
  2. 校验资源可访问
  3. 更新 HTML（按灰度规则：用户 ID / 比例 / 白名单）

缓存头：
  带 hash 的静态资源：Cache-Control: public, max-age=31536000, immutable
  HTML：Cache-Control: no-cache（每次协商）
  version.json：no-store

回滚：HTML 切回上一个版本即可（旧资源仍在 CDN 上）

客户端：
  ChunkLoadError → 重试 → 检测新版本 → 刷新（带冷却）
  定期检查 version.json → 提示用户刷新
```

**关键设计**

- **静态资源不可变**：内容变化则文件名变化，所以可以永久缓存
- **HTML 是唯一的入口**，必须每次验证，否则用户拿不到新版本
- **先资源后 HTML**：避免新 HTML 引用的资源还没上传
- **保留旧资源**：已经打开页面的用户还在使用旧的运行时，懒加载旧 chunk 时不会 404
- **灰度**：在 HTML 层控制（Nginx / 网关根据规则返回不同版本的 HTML）
- **hash 稳定性**：runtimeChunk、deterministic ID 保证未变化的文件名不变，提高 CDN 命中率

**关键指标**

- CDN 命中率、ChunkLoadError 率、发布后错误率、回滚耗时

**口述模板（2–3 分钟）**

核心原则是静态资源不可变、HTML 每次验证。构建时所有 JS、CSS、图片使用 contenthash 命名，配合 runtimeChunk 和 deterministic ID 保证没变的文件名不变；这些资源设置一年的强缓存加 immutable。HTML 不带 hash，设置 no-cache，每次协商。发布时先增量上传静态资源到 CDN，旧版本不删除，至少保留最近几个版本，然后再更新 HTML；灰度在 HTML 层控制，按用户或比例返回不同版本的 HTML，回滚只需要把 HTML 切回去，因为旧资源都还在。客户端侧做兜底：ChunkLoadError 时重试，仍然失败就检查 version.json，有新版本则带冷却地刷新页面；也可以定期检查版本提示用户刷新。监控 CDN 命中率和 ChunkLoadError 率。

**扩展追问**

- 使用 Service Worker 时要注意什么？→ SW 自身的更新策略（skipWaiting 的时机）、预缓存清单与版本的一致性，否则用户长期停留在旧版本。
- HTML 也放 CDN 怎么办？→ 设置短缓存或 no-cache，发布时主动刷新 CDN 的 HTML 缓存。

---

### S5. 设计企业级组件库的构建与发布

- **难度**: senior
- **标签**: 组件库, 打包库

**澄清问题（面试时先问）**

- 使用方的构建工具：webpack、Vite、Rspack 都有？是否有 SSR？
- 样式方案：CSS、Less、CSS-in-JS、Tailwind？
- 是否需要支持按需加载？是否需要 UMD（CDN 直接使用）？
- 是否包含图标库、主题定制？

**参考方案**

```text
源码：packages/ui/src/{Button,Table,...}/index.tsx + style.css

构建（Rslib / tsdown / Rollup，按文件保留模块结构 bundleless）：
  dist/es/Button/index.js       ESM，保留目录结构，供 Tree Shaking
  dist/es/Button/style.css
  dist/types/Button/index.d.ts  tsc 生成声明
  （可选）dist/umd/ui.min.js     供 CDN 使用

package.json：
  "type": "module",
  "exports": {
    ".": { "types": "./dist/types/index.d.ts", "import": "./dist/es/index.js" },
    "./*": { "types": "./dist/types/*/index.d.ts", "import": "./dist/es/*/index.js" },
    "./style.css": "./dist/es/style.css"
  },
  "sideEffects": ["**/*.css"],
  "peerDependencies": { "react": ">=18" }

发布：changesets 管理版本和变更日志 → CI 构建 → 发布 npm
质量：每个组件的体积预算（size-limit）、视觉回归测试、API 文档
```

**关键设计**

- **bundleless（保留模块结构）**：每个组件一个文件，使用方的打包器可以按需 Tree Shaking，不需要 babel-plugin-import
- **sideEffects 只包含 CSS**：JS 无副作用，CSS 保留
- **peer 依赖**：React 等不打包
- **样式方案**：输出 CSS 文件（或 CSS 变量实现主题），避免运行时 CSS-in-JS 的性能开销和 SSR 复杂度
- **只输出 ESM**：主流工具和 Node（`require(esm)`）都已支持；必须兼容旧环境时再加 CJS，注意双包问题
- **类型**：`exports` 中每个条件都提供 `types`
- **语法降级**：库输出较新的语法（如 ES2020），由使用方决定降级目标

**关键指标**

- 单组件引入后的体积、Tree Shaking 后的体积、构建时间、使用方的兼容性问题数量

**口述模板（2–3 分钟）**

组件库我会使用 Rslib 或 tsdown 这类库构建工具，而不是用 webpack 打包成一个文件。构建采用 bundleless 模式，保留源码的目录结构输出 ESM，每个组件一个文件，这样使用方的打包器可以直接 Tree Shaking，不需要额外的按需加载插件。类型声明由 tsc 生成。package.json 的 exports 字段为每个入口提供 types 和 import 条件，sideEffects 只声明 CSS 文件，React 作为 peer 依赖不打包。样式输出 CSS 文件并用 CSS 变量实现主题，避免运行时 CSS-in-JS 的开销。输出较新的语法，降级交给使用方。版本用 changesets 管理，CI 中用 size-limit 检查每个组件的体积，配合视觉回归测试保证质量。

**扩展追问**

- 为什么不用 webpack 打包组件库？→ webpack 的产物包含运行时代码，ESM 输出直到 5.111 才稳定，而且打包成单文件不利于使用方 Tree Shaking。
- 同时输出 ESM 和 CJS 有什么问题？→ 使用方可能同时加载两份（dual package hazard），单例状态（如 Context）失效。

---

### S6. 设计一个大型 webpack 项目迁移到 Rspack（或 Vite）的方案

- **难度**: senior
- **标签**: 迁移, Rspack, Vite

**澄清问题（面试时先问）**

- 迁移的目标：开发体验、构建速度，还是技术栈统一？
- 当前配置的复杂度：自研插件、Babel 插件、特殊 Loader 有多少？
- 是否有 Module Federation、SSR？
- 可以接受多长的迁移周期？有没有 E2E 测试？

**参考方案**

```text
选型：
  目标是提速 + 配置复杂 → Rspack（兼容 webpack 配置）
  目标是统一到 Vite 生态 + 项目可以改造 → Vite

阶段 0：评估（1 周）
  ├─ 清点 Loader / 插件 / Babel 插件，对照兼容列表
  ├─ 分支试跑：记录构建时间、产物差异
  └─ 输出风险清单与工作量估算

阶段 1：代码去 webpack 化（可选，Vite 必需）
  ├─ require.context → import.meta.glob
  ├─ process.env → import.meta.env / define
  └─ worker-loader → new Worker(new URL(...))

阶段 2：配置迁移
  ├─ 替换为内置实现（swc-loader、CSS 抽取、压缩）
  ├─ 自研插件改造
  └─ Babel 插件迁移到 swc 插件或保留少量 babel-loader

阶段 3：验证
  ├─ 产物对比：chunk 结构、体积、关键模块归属
  ├─ 浏览器兼容：降级结果、polyfill
  ├─ E2E 回归 + 关键路径人工验证
  └─ source map 和错误监控是否正常

阶段 4：上线
  ├─ 先开发环境（收益快、风险低）
  ├─ 生产环境灰度（部分流量），监控错误率和性能指标
  └─ 保留 webpack 构建一段时间作为回滚手段
```

**关键设计**

- **可回滚**：迁移期间两套构建都能用，问题出现时可以快速切回
- **产物对比是重点**：功能测试覆盖不到的差异（CSS 顺序、polyfill、chunk 拆分）通过产物对比发现
- **分阶段收益**：开发环境先迁移，团队很快感受到收益，降低推动阻力
- **监控**：上线后对比错误率、性能指标、ChunkLoadError

**关键指标**

- 开发启动 / HMR / 生产构建时间的前后对比、线上错误率、性能指标

**口述模板（2–3 分钟）**

我会先根据目标选型：如果主要目标是提速、配置又比较复杂，选 Rspack，因为它兼容 webpack 的配置和大部分插件；如果希望统一到 Vite 生态并且项目可以改造，再考虑 Vite。迁移分阶段推进。先用一周评估：清点所有 Loader、插件和 Babel 插件，对照兼容列表，在分支上试跑并记录构建时间和产物差异。然后把 webpack 特有的写法改为标准写法，比如 require.context 改为 import.meta.glob，这一步对迁移 Vite 是必需的。配置迁移时尽量替换为内置的高性能实现，自研插件按新工具的钩子改造。验证阶段重点做产物对比，包括 chunk 结构、体积、CSS 顺序和语法降级，再加上 E2E 回归。上线时先在开发环境使用，再在生产环境灰度，并保留 webpack 构建一段时间作为回滚手段。

**扩展追问**

- 迁移后体积变大了？→ 对比拆包配置和压缩器设置，检查 swc 和 Babel 的降级目标是否一致。
- 团队成员不熟悉新工具？→ 配置封装在平台中，业务开发者几乎无感知。

---

### S7. 设计前端产物体积治理体系

- **难度**: mid
- **标签**: 体积优化, 治理

**澄清问题（面试时先问）**

- 当前首屏 JS 体积和性能指标（LCP、INP）？
- 体积问题的主要来源：依赖膨胀、全量导入、拆包不合理？
- 团队规模、PR 数量？

**参考方案**

```text
预防（PR 阶段）
  ├─ 构建 stats → 与主分支对比
  ├─ PR 评论：各 chunk 的体积变化、新增的模块、新增的依赖
  ├─ 预算：入口 / 首屏 / 单个异步 chunk，超出则需审批
  └─ 新依赖检查：体积、是否支持 Tree Shaking、是否与已有依赖重复

发现（持续）
  ├─ 每次发布记录体积数据 → 趋势图
  ├─ 重复依赖、未使用依赖（knip）、未使用文件检查
  └─ 关联线上指标：体积变化 vs LCP

治理（定期）
  ├─ 大依赖替换（moment → dayjs）
  ├─ 全量导入改为按需
  └─ 拆包策略调整
```

**关键设计**

- **在 PR 阶段拦截**：体积问题合入后很难追溯，PR 中可见、可阻止
- **预算关注首屏关键资源**：总体积增加但都在懒加载页面中，影响较小
- **数据可解释**：PR 评论中说明「哪个模块导致了增长」，而不只是数字
- **与业务指标关联**：用 LCP 等指标证明治理的价值

**关键指标**

- 首屏 JS 传输体积、各入口体积、超预算 PR 数、LCP

**口述模板（2–3 分钟）**

体积治理我分为预防、发现和治理三部分。预防最重要：每个 PR 构建后与主分支对比 stats，在 PR 中评论每个 chunk 的变化以及导致增长的模块和新增依赖；对入口、首屏和单个异步 chunk 设置预算，超出需要审批。新增依赖时检查体积、是否支持 Tree Shaking、是否与现有依赖重复。发现方面，每次发布记录体积数据形成趋势，定期运行重复依赖和未使用代码检查。治理方面，定期处理大依赖替换、全量导入和拆包策略。预算重点关注首屏关键资源，而不是总体积。最后把体积变化和 LCP 等线上指标关联起来，用数据证明治理的价值。

**扩展追问**

- 如何避免预算形同虚设？→ 预算调整需要审批并记录原因；定期复盘超预算的情况。

---

### S8. 设计 SSR 应用的构建体系

- **难度**: senior
- **标签**: SSR

**澄清问题（面试时先问）**

- 使用框架（Next.js 等）还是自研 SSR？
- 是否需要流式渲染、React Server Components？
- 部署环境：Node 服务器、Serverless、边缘运行时？
- 首屏性能和 SEO 的具体要求？

**参考方案**

```text
构建（两套配置，共享基础配置）
  client：target web → 带 hash 的静态资源 + client-manifest.json（模块 → chunk 文件）
  server：target node → server bundle（externals node_modules，但保留需要处理的依赖）
          CSS 只导出类名，不输出文件；资源只生成 URL

运行时
  请求 → Node 服务加载 server bundle → 渲染 HTML（流式）
       → 根据渲染中用到的懒加载组件，从 client-manifest 中查出需要的 chunk
       → 注入 <link rel="preload"> 和 <script>
       → 浏览器加载客户端代码 → 水合

开发
  两套构建同时 watch；server bundle 变化后重新加载（清除 require 缓存或重启 worker）
  客户端 HMR 正常工作
```

**关键设计**

- **两套构建的一致性**：资源 URL、CSS Modules 类名、模块 ID 必须一致，否则水合不匹配
- **manifest 驱动的资源注入**：服务端知道首屏用了哪些组件，提前注入对应 chunk，避免水合前的加载瀑布
- **server bundle 的 externals**：大部分依赖 externals 加快构建，但 ESM-only 或导入 CSS 的包需要打包
- **流式渲染**：先输出 HTML 头部和关键资源，减少 TTFB
- **部署**：server bundle 和静态资源版本一致；静态资源先上传
- **现实选择**：大多数情况直接使用 Next.js 等框架；自研只在有特殊需求时考虑

**关键指标**

- TTFB、LCP、水合耗时、服务端渲染耗时和错误率

**口述模板（2–3 分钟）**

SSR 需要客户端和服务端两套构建，共享基础配置。客户端构建输出带 hash 的静态资源和一份 manifest，记录每个模块对应的 chunk 文件；服务端构建 target 为 node，大部分依赖 externals，但需要处理 CSS 或 ESM-only 的依赖保留打包，CSS 只导出类名、资源只生成 URL，保证和客户端一致。运行时服务端渲染 HTML，根据渲染中用到的懒加载组件查 manifest，把对应的 chunk 以 preload 和 script 注入，浏览器加载后直接水合，避免加载瀑布。配合流式渲染降低 TTFB。开发时两套构建同时 watch，服务端 bundle 变化后重新加载。部署时保证 server bundle 和静态资源版本一致。实际项目中，除非有特殊需求，我会直接选择 Next.js 这类框架。

**扩展追问**

- 水合不匹配的常见原因？→ 两套构建的类名或资源地址不一致、服务端和客户端的环境变量不同、渲染中使用了时间或随机数。
- RSC 对构建有什么影响？→ 需要按 `"use client"` 边界把模块分到服务端和客户端两个模块图，并生成客户端引用清单，复杂度高，通常依赖框架（Rspack 2.0 已原生支持 RSC）。
