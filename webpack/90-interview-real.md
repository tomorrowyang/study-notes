# webpack 真实面试题汇总

> 根据近年公开面经、社区高频题（掘金、各类前端面试题库）和国内前端 / 前端工程化岗位面试趋势整理。`来源倾向` 表示题目常见的出现场景，不代表某公司的固定题库。题干尽量保留面试中的真实问法。

---

## 一、基础

### R1. 说一下 webpack 的构建流程

- **难度**: mid
- **标签**: 构建流程
- **来源倾向**: 前端工程化必问，几乎每场二面都会出现

**答案**

初始化（合并配置、创建 Compiler、注册插件）→ `run` / `compile`（创建 Compilation）→ make（从入口开始：resolve 路径 → Loader 转换 → 解析 AST 收集依赖 → 递归，得到模块依赖图）→ seal（标记导出使用情况、生成 chunk、splitChunks、分配 ID、计算 hash、代码生成、processAssets 压缩）→ emit（输出文件）→ done。整个过程由 Tapable 钩子串联，插件在各阶段介入。Compiler 全局唯一，Compilation 每次编译新建。

**口述模板（30–60 秒）**

1. **一句话结论**：初始化、编译、构建模块图、封装优化、输出五个阶段。
2. **关键机制**：make 阶段从入口递归，用 Loader 转换、解析 AST 找依赖，建成模块图；seal 阶段把模块分组为 chunk、优化并生成代码。
3. **注意点**：Compiler 是全局调度者，Compilation 是一次编译的上下文，watch 模式下每次变化都新建 Compilation。
4. **可追问时补充**：Loader 在 make 阶段执行，压缩在 seal 的 processAssets 阶段执行；插件修改产物应使用 processAssets 而不是 emit。

详见：[20-mid-qa Q1](20-mid-qa.md)、[20-mid-qa Q2](20-mid-qa.md)、[30-senior-qa Q2](30-senior-qa.md)

---

### R2. Loader 和 Plugin 有什么区别？你写过哪些？

- **难度**: mid
- **标签**: Loader, Plugin
- **来源倾向**: 高频，「写过吗」是常见追问，最好准备一个真实例子

**答案**

Loader 是函数，输入单个文件的内容，输出转换后的内容，在 make 阶段构建模块时执行，配置在 `module.rules`；Plugin 是带 `apply(compiler)` 方法的对象，通过 Tapable 钩子介入整个构建生命周期，可以访问所有模块和产物，配置在 `plugins`。可以举的例子：Markdown 转模块的 Loader、生成产物清单并检查体积预算的插件、检测重复依赖的插件、注入版本号并输出 version.json 的插件。

详见：[10-junior-qa Q7](10-junior-qa.md)、[20-mid-qa Q5](20-mid-qa.md)、[40-coding-handwritten H2](40-coding-handwritten.md)、[40-coding-handwritten H4](40-coding-handwritten.md)

---

### R3. module、chunk、bundle 的区别？hash、chunkhash、contenthash 的区别？

- **难度**: junior
- **标签**: 概念, 缓存
- **来源倾向**: 一面高频，两个问题常常连着问

**答案**

module 是源文件经 Loader 处理后的模块；chunk 是打包过程中按入口、动态 import、splitChunks 规则形成的模块分组；bundle 是 chunk 最终输出的文件，一个 chunk 可以输出多个文件。hash（webpack 5 叫 fullhash）是整次编译的 hash；chunkhash 是 chunk 级别，同 chunk 的 JS 和 CSS 共享；contenthash 只与输出文件的内容有关，最适合长效缓存。

详见：[10-junior-qa Q3](10-junior-qa.md)、[10-junior-qa Q9](10-junior-qa.md)

---

### R4. 你常用哪些 Loader 和 Plugin？分别解决什么问题？

- **难度**: junior
- **标签**: 生态
- **来源倾向**: 一面开场常见

**答案**

Loader：babel-loader / swc-loader（转译）、css-loader（解析 `@import` 和 `url()`）、style-loader（注入样式）、postcss-loader、sass-loader；webpack 5 用 Asset Modules 替代 file-loader 和 url-loader。Plugin：HtmlWebpackPlugin（生成 HTML 并注入产物）、MiniCssExtractPlugin（抽取 CSS）、DefinePlugin（编译时常量）、TerserPlugin / CssMinimizerPlugin（压缩）、ForkTsCheckerWebpackPlugin（独立进程类型检查）、webpack-bundle-analyzer（体积分析）、ModuleFederationPlugin。

详见：[10-junior-qa Q5](10-junior-qa.md)、[10-junior-qa Q6](10-junior-qa.md)

---

### R5. source map 有哪些类型？开发和生产分别怎么选？

- **难度**: junior
- **标签**: source map
- **来源倾向**: 高频，常追问「生产环境如何定位报错」

**答案**

devtool 由 eval、source-map、cheap、module、inline、hidden、nosources 等关键字组合而成。开发用 `eval-cheap-module-source-map`（重建快、能映射到 Loader 处理前的源码）；生产用 `hidden-source-map`，构建后上传到 Sentry 等监控平台，再从部署产物中删除，避免源码泄露。原理是 mappings 字段用 Base64 VLQ 编码记录产物位置到源码位置的映射。

详见：[10-junior-qa Q16](10-junior-qa.md)、[20-mid-qa Q19](20-mid-qa.md)

---

## 二、原理

### R6. Loader 的执行顺序是怎样的？pitch 有什么用？

- **难度**: mid
- **标签**: Loader
- **来源倾向**: 中高级常问，常以 style-loader 为例追问

**答案**

normal 阶段从右到左（从下到上）；之前先从左到右执行各 Loader 的 pitch。pitch 返回非 undefined 时熔断：跳过后续 Loader 和读取文件，直接回到前一个 Loader 的 normal。多个 rule 按 pre → normal → inline → post 排序。style-loader 在 pitch 中返回一段代码，通过 `!!css-loader!./a.css` 内联请求拿到 CSS 内容并注入页面，`!!` 防止再次匹配自己。

详见：[20-mid-qa Q4](20-mid-qa.md)、[40-coding-handwritten H3](40-coding-handwritten.md)

---

### R7. Tapable 是什么？有哪些钩子？

- **难度**: mid
- **标签**: Tapable
- **来源倾向**: 中高级常问，高级岗会问实现原理

**答案**

Tapable 是 webpack 的事件流库，整个构建流程通过它的钩子串联。按执行方式分为 Sync / AsyncSeries / AsyncParallel，按语义分为普通、Bail（有返回值就中止）、Waterfall（传递返回值）、Loop。注册方式有 `tap`、`tapAsync`、`tapPromise`，可用 `stage` 和 `before` 控制顺序。实现上会根据注册的回调用 `new Function` 动态生成调用代码，避免运行时判断。

详见：[20-mid-qa Q3](20-mid-qa.md)、[30-senior-qa Q3](30-senior-qa.md)、[40-coding-handwritten H6](40-coding-handwritten.md)

---

### R8. 打包后的代码长什么样？能手写一个简单的打包器吗？

- **难度**: senior
- **标签**: 运行时, 手写
- **来源倾向**: 高级岗手写题，常见于工程化方向

**答案**

产物是一个自执行函数：模块表（模块 ID → 包装函数）、模块缓存、自己实现的 `__webpack_require__`（查缓存 → 创建 module 并先放入缓存 → 执行模块函数 → 返回 exports），以及按需注入的运行时辅助函数。手写打包器的核心：Babel 解析 AST 收集 import → 广度优先构建依赖图并记录「相对路径 → 模块 ID」映射 → 把 ESM 转为 CommonJS → 生成包含模块表和 require 的 bundle。

详见：[20-mid-qa Q7](20-mid-qa.md)、[40-coding-handwritten H1](40-coding-handwritten.md)

---

### R9. 动态 import 是怎么实现按需加载的？

- **难度**: mid
- **标签**: 代码分割, 运行时
- **来源倾向**: 中高级常问

**答案**

`import('./Page')` 被编译为 `__webpack_require__.e(chunkId).then(__webpack_require__.bind(null, moduleId))`。`.e` 根据 chunk 状态（0 已加载、数组表示加载中、undefined 未加载）决定是否插入 script 标签；chunk 文件执行时调用全局数组的 `push`，运行时已劫持这个 push：把模块合并到模块表、标记 chunk 已加载、resolve Promise。加载失败时重置状态并抛出 ChunkLoadError。

详见：[20-mid-qa Q8](20-mid-qa.md)、[40-coding-handwritten H7](40-coding-handwritten.md)

---

### R10. 说一下 HMR 的原理

- **难度**: mid
- **标签**: HMR
- **来源倾向**: 大厂前端高频，常要求讲完整链路

**答案**

watch 监听文件变化触发增量编译，生成 hot-update.json（manifest）和 hot-update.js（更新的模块）；dev-server 通过 WebSocket 只推送新的 hash；浏览器端 HMR runtime 拉取 manifest 和更新 chunk；从变化的模块向上冒泡查找 `module.hot.accept` 边界，执行旧模块的 dispose，删除旧缓存，执行新模块和 accept 回调；冒泡到入口仍没有 accept 或出错，则整页刷新。

**口述模板（30–60 秒）**

1. **一句话结论**：增量编译 + WebSocket 通知 + 客户端拉取更新 + 冒泡查找 accept 边界替换模块。
2. **关键机制**：服务端只推 hash，客户端 runtime 根据 hash 请求 manifest 和更新的 chunk；运行时找到 accept 边界后删除过期模块缓存并重新执行。
3. **注意点**：HMR 不是重新打包后刷新；没有 accept 边界会回退为整页刷新；有副作用的模块要在 dispose 中清理。
4. **可追问时补充**：style-loader、vue-loader、React Fast Refresh 都是自动注入 accept 代码；Fast Refresh 还会根据 Hook 签名决定是否保留状态。

详见：[20-mid-qa Q9](20-mid-qa.md)、[20-mid-qa Q10](20-mid-qa.md)、[30-senior-qa Q6](30-senior-qa.md)

---

### R11. Tree Shaking 的原理？为什么有时候不生效？

- **难度**: mid
- **标签**: Tree Shaking
- **来源倾向**: 高频，常给出代码让你找问题

**答案**

基于 ESM 的静态结构：webpack 在 make 阶段收集模块提供的导出，在 seal 阶段标记哪些导出被使用（usedExports），未使用的导出不再生成导出定义，最后由 Terser 等压缩器删除死代码；`sideEffects` 让无副作用且导出都未被使用的模块被整体跳过。失效原因：使用 CommonJS、Babel 把 ESM 转成了 CommonJS、模块有顶层副作用、没有声明 sideEffects、命名空间被动态访问、顶层函数调用缺少 PURE 注解、在开发模式下验证。

**口述模板（30–60 秒）**

1. **一句话结论**：webpack 标记，压缩器删除，sideEffects 负责跳过整个模块。
2. **关键机制**：ESM 静态分析得到导出使用情况，未使用的导出变成死代码。
3. **注意点**：sideEffects 设为 false 时要排除 CSS 和 polyfill，否则会被误删。
4. **可追问时补充**：barrel 文件会拖慢构建并可能把整个库拉进来，webpack 5.108 的 lazy barrel 和 5.110 的性能提示专门针对这个问题。

详见：[20-mid-qa Q11](20-mid-qa.md)、[30-senior-qa Q4](30-senior-qa.md)、[40-coding-handwritten H11](40-coding-handwritten.md)

---

## 三、优化

### R12. 你们项目的构建速度是怎么优化的？

- **难度**: senior
- **标签**: 构建速度
- **来源倾向**: 工程化方向几乎必问，必须有数据

**答案**

先度量（ProfilingPlugin、stats timings），区分开发启动、HMR、生产构建。按投入产出比优化：持久化缓存（CI 中缓存 `.cache` 目录）→ swc / esbuild 替代 Babel、类型检查移到独立进程 → SWC 压缩和 lightningcss → 缩小 Loader 和解析范围 → 开发环境 lazyCompilation → 治理全量导入和 barrel 文件 → 迁移 Rspack。DllPlugin、HardSource 已被 webpack 5 的持久化缓存取代。

**口述模板（30–60 秒）**

1. **一句话结论**：先测量找瓶颈，再按缓存、转译器、压缩器、范围、按需编译的顺序优化，最后考虑换工具。
2. **关键机制**：持久化缓存避免重复构建；swc 用原生代码替代 Babel；lazyCompilation 让未访问的页面不编译。
3. **注意点**：每次只改一项并记录数据；thread-loader 有进程开销，不一定有收益。
4. **可追问时补充**：给出自己项目的具体数据，比如冷构建从 8 分钟降到 1 分钟，HMR 从 5 秒降到 1 秒以内。

详见：[20-mid-qa Q16](20-mid-qa.md)、[30-senior-qa Q11](30-senior-qa.md)、[30-senior-qa Q12](30-senior-qa.md)、[50-system-design S1](50-system-design.md)

---

### R13. 怎么减小打包体积 / 优化首屏加载？

- **难度**: mid
- **标签**: 体积优化
- **来源倾向**: 高频

**答案**

先用 bundle-analyzer 分析。方向：路由和大组件懒加载、Tree Shaking（ESM 版本的库、按名称导入）、替换大依赖（moment → dayjs）、去除重复依赖、合理拆包（首屏只加载必要代码）、按目标浏览器引入 polyfill 或现代 / 传统双构建、JS / CSS 压缩和 brotli、关键资源 preload、长效缓存提高回访速度。最后在 CI 中设置体积预算防止回退。

详见：[20-mid-qa Q18](20-mid-qa.md)、[30-senior-qa Q15](30-senior-qa.md)、[50-system-design S7](50-system-design.md)

---

### R14. splitChunks 怎么配置？你们的拆包策略是什么？

- **难度**: mid
- **标签**: 代码分割
- **来源倾向**: 中级高频，常要求现场写配置

**答案**

默认只拆异步 chunk（`chunks: 'async'`），生产通常改为 `all` 并加 `runtimeChunk: 'single'`。按变化频率和加载时机设计 cacheGroups：框架（React 全家桶）单独一个 chunk 长期缓存；大型 UI 库单独拆分；只在部分页面使用的大库（如 echarts）限定 `chunks: 'async'`；其余第三方库不设固定名称，避免一个巨大的 vendors 被所有页面加载；业务公共组件用 `minChunks` 提取。拆分后用 bundle-analyzer 验证，避免拆得过细。

详见：[20-mid-qa Q14](20-mid-qa.md)、[40-coding-handwritten H9](40-coding-handwritten.md)

---

### R15. 为什么只改了一个文件，很多文件的 hash 都变了？

- **难度**: mid
- **标签**: 缓存
- **来源倾向**: 中高级常问

**答案**

常见原因：运行时代码在入口 chunk 中，它记录了所有 chunk 的文件名，任何 chunk 变化都会让入口变化（用 `runtimeChunk: 'single'` 抽离）；模块 ID 使用自增数字，新增模块导致后续 ID 偏移（用 `deterministic`）；使用了 chunkhash 而不是 contenthash；代码中注入了时间戳或绝对路径。排查方法是对比两次构建中变化文件的内容。

详见：[20-mid-qa Q15](20-mid-qa.md)、[30-senior-qa Q5](30-senior-qa.md)

---

### R16. webpack 5 的持久化缓存是怎么工作的？什么时候会失效？

- **难度**: mid
- **标签**: 缓存
- **来源倾向**: webpack 5 相关高频

**答案**

`cache: { type: 'filesystem' }` 把模块构建、解析、代码生成的结果序列化到 `node_modules/.cache/webpack`。每个缓存项保存依赖文件的快照：项目文件比较时间戳或内容 hash，node_modules 中的包只比较 package.json 版本号。`buildDependencies` 中的文件（如配置文件）变化或 `cache.version` 变化时整体失效。直接修改 node_modules 中的文件不会让缓存失效，环境变量影响构建结果时需要通过 version 区分。

详见：[20-mid-qa Q17](20-mid-qa.md)、[30-senior-qa Q9](30-senior-qa.md)

---

### R17. 配置了 `sideEffects: false` 之后样式丢了，为什么？

- **难度**: mid
- **标签**: Tree Shaking
- **来源倾向**: 常见实战问题

**答案**

`import './style.css'` 没有使用任何导出，如果 CSS 被视为无副作用，webpack 会整体跳过这个模块。解决：`"sideEffects": ["*.css", "*.scss"]`，或在 module.rules 中对样式文件设置 `sideEffects: true`。同理，polyfill、全局注册类的文件也要列入。

详见：[20-mid-qa Q12](20-mid-qa.md)

---

## 四、工程与选型

### R18. Module Federation 是什么？你们怎么用它做微前端？

- **难度**: senior
- **标签**: Module Federation, 微前端
- **来源倾向**: 微前端方向高频

**答案**

webpack 5 的模块联邦让独立构建、独立部署的应用在运行时共享模块。remote 通过 `exposes` 暴露模块并生成 remoteEntry；host 加载后通过 `init` 共享依赖、`get` 获取模块；`shared` 中的依赖按版本协商，React 等需要 `singleton`；入口需要 `import('./bootstrap')` 异步边界。生产实践：remote 版本化部署，host 从配置中心获取版本地址实现灰度和回滚；每个 remote 有 ErrorBoundary 降级；MF 2.0 提供 manifest、动态 remote、运行时插件和类型提示。

详见：[20-mid-qa Q23](20-mid-qa.md)、[30-senior-qa Q7](30-senior-qa.md)、[40-coding-handwritten H12](40-coding-handwritten.md)、[50-system-design S3](50-system-design.md)

---

### R19. 发布新版本后，用户访问懒加载页面报错白屏，为什么？怎么解决？

- **难度**: senior
- **标签**: 发布, 线上问题
- **来源倾向**: 高级岗实战题

**答案**

ChunkLoadError：用户打开的是旧页面，旧运行时记录的是旧 chunk 文件名，新版本发布时旧资源被删除导致 404。根本方案：静态资源增量上传、保留旧版本，先发资源后发 HTML。客户端兜底：`import()` 失败重试（webpack 会重置 chunk 状态），重试失败时检查 version.json，有新版本则刷新页面并用 sessionStorage 防止无限刷新；监控 ChunkLoadError 率。

**口述模板（30–60 秒）**

1. **一句话结论**：最常见原因是发布时删除了旧 chunk，根本解决是保留旧版本资源。
2. **关键机制**：已打开的旧页面运行时记录的是旧文件名，懒加载时请求的旧文件已不存在。
3. **注意点**：发布顺序先资源后 HTML；客户端重试 + 版本检测 + 带冷却的刷新。
4. **可追问时补充**：缓存策略上，带 hash 的资源强缓存 immutable，HTML 设置 no-cache，回滚只需切换 HTML。

详见：[30-senior-qa Q16](30-senior-qa.md)、[40-coding-handwritten H8](40-coding-handwritten.md)、[50-system-design S4](50-system-design.md)

---

### R20. Vite 为什么比 webpack 快？你会怎么选？

- **难度**: mid
- **标签**: 选型, Vite
- **来源倾向**: 近年高频

**答案**

Vite 开发时不打包：浏览器按原生 ESM 请求模块，服务器按需编译，启动时间与项目规模基本无关；依赖用 esbuild（新版本 Rolldown）预构建；HMR 只让浏览器重新请求变化的模块。webpack 开发时也要先构建完整模块图并打包，项目越大越慢。Vite 的问题：大项目首次加载请求数多、开发与生产构建方式不同。选型：新项目和中小型应用用 Vite；大型存量 webpack 项目提速选 Rspack；框架项目跟随框架。

详见：[20-mid-qa Q31](20-mid-qa.md)、[30-senior-qa Q21](30-senior-qa.md)

---

### R21. 了解 Rspack 吗？从 webpack 迁移到 Rspack 要做什么？

- **难度**: mid
- **标签**: Rspack
- **来源倾向**: 近年新增，国内使用较多

**答案**

Rspack 是 Rust 实现、兼容 webpack 配置和大部分 Loader / 插件的打包器，2026 年 4 月发布 2.0（纯 ESM、Node 20.19+、支持 RSC）。迁移：替换 `@rspack/core` 和 CLI；babel-loader 换为 `builtin:swc-loader`，CSS 抽取和压缩换为内置插件；检查依赖 webpack 内部 API 的自研插件；产物对比（chunk 结构、体积、CSS 顺序、降级结果）+ E2E 回归；先开发环境后生产灰度，保留 webpack 构建作为回滚手段。

详见：[20-mid-qa Q32](20-mid-qa.md)、[30-senior-qa Q19](30-senior-qa.md)、[50-system-design S6](50-system-design.md)

---

### R22. webpack 5 相比 webpack 4 有哪些重要变化？最近的 5.x 版本有什么新东西？

- **难度**: mid
- **标签**: 版本
- **来源倾向**: 常见，考察是否持续关注

**答案**

webpack 5：持久化缓存、Module Federation、Asset Modules 取代 file/url/raw-loader、Tree Shaking 增强（嵌套导出、inner graph、部分 CommonJS）、deterministic 模块和 chunk ID、移除 Node 核心模块的自动 polyfill（需要 `resolve.fallback`）。近期 5.x：5.105 `resolve.tsconfig`，5.107 实验性 TypeScript 类型擦除，5.108 lazy barrel，5.109 内置 CSS / HTML 默认 auto、`import.meta.glob`，5.110 性能提示，5.111 ESM 输出稳定（`output.module`）。webpack 6 没有公布时间，路线图是继续内置 CSS、HTML、TS。细节以官方 changelog 为准。

详见：[00-roadmap 版本说明](00-roadmap.md)、[20-mid-qa Q27](20-mid-qa.md)、[30-senior-qa Q10](30-senior-qa.md)

---

### R23. 写 webpack 插件要注意什么？讲一个你写过的插件

- **难度**: senior
- **标签**: Plugin
- **来源倾向**: 高级工程化岗位

**答案**

选对钩子：修改产物用 `processAssets` 并选择合适的 stage（REPORT 阶段拿到的是压缩后的最终结果），不在 emit 中修改产物；用 `emitAsset` / `updateAsset` 和 `webpack.sources` 操作产物；从 `compiler.webpack` 获取 webpack 实例；耗时计算用 Cache API 接入持久化缓存；不持有 Compilation 防止 watch 模式内存泄漏；错误 push 到 `compilation.errors` / `warnings` 而不是直接抛出。例子：产物清单与体积预算插件、重复依赖检测插件。

详见：[30-senior-qa Q18](30-senior-qa.md)、[40-coding-handwritten H4](40-coding-handwritten.md)、[40-coding-handwritten H5](40-coding-handwritten.md)

---

### R24. publicPath 有什么用？微前端中子应用资源 404 怎么办？

- **难度**: mid
- **标签**: publicPath, 微前端
- **来源倾向**: 微前端项目经验常问

**答案**

publicPath 是浏览器加载资源（异步 chunk、图片、字体、CSS 中的 url）时使用的 URL 前缀。子应用被主应用加载时，资源地址应该指向子应用自己的域名，但构建时可能不知道部署地址：在入口最前面设置 `__webpack_public_path__`（如 qiankun 注入的变量），或者使用 `publicPath: 'auto'` 根据当前脚本地址推断。同时多个应用要设置不同的 `output.uniqueName`，避免 JSONP 全局变量冲突。

详见：[10-junior-qa Q19](10-junior-qa.md)、[20-mid-qa Q30](20-mid-qa.md)

---

### R25. Babel 的 polyfill 怎么配置？`useBuiltIns` 的几个选项有什么区别？

- **难度**: mid
- **标签**: Babel, 兼容
- **来源倾向**: 中级常问

**答案**

`@babel/preset-env` 根据 browserslist 转换语法，API 补全依赖 core-js。`useBuiltIns: 'entry'`：入口引入 `core-js`，按目标浏览器替换为需要的全部模块；`'usage'`：按代码中实际使用的 API 引入，体积更小但默认不分析 node_modules；`false`：不处理。应用用 entry 或 usage；库用 `@babel/plugin-transform-runtime` + `corejs: 3`，不污染全局。现代浏览器占比高时可以只构建现代版本，减少 polyfill。

详见：[10-junior-qa Q12](10-junior-qa.md)、[20-mid-qa Q20](20-mid-qa.md)、[30-senior-qa Q24](30-senior-qa.md)
