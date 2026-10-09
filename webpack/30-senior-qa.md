# webpack 高级 Q&A（26 题）

> 目标：能扛住原理追问，能讲清性能优化和线上问题的排查过程，能做构建工具的架构取舍。示例基于 webpack 5，涉及 5.10x 新特性时标注版本。

---

## 一、原理深挖

### Q1. webpack 5 内部是用什么数据结构表示模块和 chunk 的？

- **难度**: senior
- **标签**: 原理, 源码

**答案**

核心对象：

| 对象 | 说明 |
|------|------|
| `Module` | 模块，最常见的是 `NormalModule`（普通文件），还有 `ContextModule`、`ExternalModule`、`ConcatenatedModule`（Scope Hoisting 合并后的模块）等 |
| `Dependency` | 依赖关系，描述「模块 A 通过某种方式引用了 B」，如 `HarmonyImportSpecifierDependency`（ESM 具名导入）、`CommonJsRequireDependency`、`ImportDependency`（动态 import） |
| `ModuleGraph` | 模块图：记录每个 Dependency 指向哪个 Module、每个模块的导出信息（`ExportsInfo`：提供了哪些导出、哪些被使用） |
| `Chunk` | 代码块 |
| `ChunkGroup` / `Entrypoint` | chunk 组，一个入口或一个动态 import 点对应一个 ChunkGroup，可以包含多个 chunk（如拆分出的 vendors） |
| `ChunkGraph` | 记录模块与 chunk 的多对多关系、每个模块的 ID 和 hash |

webpack 4 中这些关系直接挂在 Module 和 Chunk 上；webpack 5 把关系抽到 ModuleGraph 和 ChunkGraph 中，**模块对象本身与具体的编译解耦**，这是持久化缓存能复用模块对象的前提。

从入口到产物的数据流：

```text
Entry → Dependency → (factorize) → Module → (build) → 更多 Dependency → ... ModuleGraph
ModuleGraph → (buildChunkGraph) → ChunkGroup / Chunk → ChunkGraph
ChunkGraph → (codeGeneration + render) → Source → assets
```

**口述模板（30–60 秒）**

1. 一句话结论：Module 和 Dependency 组成 ModuleGraph，Chunk 和 ChunkGroup 组成 ChunkGraph。
2. 关键机制：Dependency 描述引用关系，ModuleGraph 记录依赖指向和导出使用情况，ChunkGraph 记录模块属于哪些 chunk。
3. 对比或注意点：webpack 5 把关系从对象中抽离为图结构，让模块可以跨编译缓存。
4. 可追问时补充：Tree Shaking 的使用信息就存储在 ModuleGraph 的 ExportsInfo 中。

**常见追问**

- ChunkGroup 和 Chunk 的区别？→ ChunkGroup 是加载单元（一个入口或一个 import 点），splitChunks 会把一个 ChunkGroup 的模块拆到多个 Chunk 中，加载时并行加载组内所有 chunk。

---

### Q2. seal 阶段具体做了什么？

- **难度**: senior
- **标签**: 原理, 构建流程

**答案**

`compilation.seal()` 的主要步骤（按顺序）：

1. **`finishModules`**：所有模块构建完成，计算每个模块提供的导出（`FlagDependencyExportsPlugin`）
2. **`optimizeDependencies`**：计算导出的使用情况（`FlagDependencyUsagePlugin`，即 `usedExports`）；处理 `sideEffects`，把引用指向真正提供导出的模块（跳过无副作用的中间模块）
3. **`buildChunkGraph`**：从入口出发构建 ChunkGroup 和 Chunk，动态 import 形成新的 ChunkGroup；分析每个 chunk 中哪些模块已经在父 chunk 中可用
4. **`optimizeModules` / `optimizeChunks`**：
   - `SplitChunksPlugin`：按 cacheGroups 拆分
   - `RemoveEmptyChunksPlugin`、`MergeDuplicateChunksPlugin`
   - `ModuleConcatenationPlugin`：Scope Hoisting（`optimizeChunkModules` 钩子）
5. **`moduleIds` / `chunkIds`**：分配 ID（deterministic / named / natural）
6. **`createHash`**：计算模块 hash、chunk hash、全量 hash
7. **`codeGeneration`**：为每个模块生成代码（根据使用情况省略未使用的导出）
8. **`createChunkAssets`**：用 JavascriptModulesPlugin 等把 chunk 渲染为 Source 对象，作为 assets
9. **`processAssets`**：按阶段处理产物，压缩（Terser）、source map、realContentHash 都在这里
10. **`afterSeal`**

**口述模板（30–60 秒）**

1. 一句话结论：seal 是优化和产物生成阶段：标记导出 → 建 chunk 图 → 拆分合并 → 分配 ID → 计算 hash → 代码生成 → 处理产物。
2. 关键机制：Tree Shaking 的标记在 optimizeDependencies，splitChunks 在 optimizeChunks，压缩在 processAssets。
3. 对比或注意点：hash 在代码生成之前计算，所以需要 realContentHash 在压缩后重新计算真实 hash。
4. 可追问时补充：插件修改产物应使用 processAssets 的合适阶段，而不是在 emit 中修改。

**常见追问**

- 为什么推荐在 processAssets 中修改产物而不是 emit？→ processAssets 有明确的阶段划分，能和压缩、source map、hash 计算正确配合；emit 时 hash 已经确定，修改内容会导致 hash 和内容不一致。

---

### Q3. Tapable 是怎么实现的？为什么性能好？

- **难度**: senior
- **标签**: Tapable, 源码

**答案**

核心思想：**根据已注册的回调动态生成调用函数的代码**，而不是在每次调用时遍历回调数组并判断类型。

```js
// 注册了两个同步回调的 SyncHook，call 时生成的函数类似：
function anonymous(compilation) {
  var _x = this._x
  var _fn0 = _x[0]
  _fn0(compilation)
  var _fn1 = _x[1]
  _fn1(compilation)
}
```

实现要点：

1. `tap` 时把回调按 `stage` 和 `before` 插入到 `taps` 数组中，并把 `call` 重置为「懒编译」函数
2. 第一次调用 `call` 时，`HookCodeFactory` 根据钩子类型（Sync / AsyncSeries / AsyncParallel，Bail / Waterfall / Loop）和每个回调的注册方式（sync / async / promise），用字符串拼接出最优的调用代码，再用 `new Function` 生成函数
3. 之后的调用直接执行生成的函数，没有循环和类型判断
4. **拦截器**（`hook.intercept({ call, tap, register })`）可以在调用和注册时介入，webpack 的 ProfilingPlugin 和进度插件就是这样统计耗时的

为什么这样设计：webpack 构建过程中钩子会被调用数百万次（每个模块都会触发多个钩子），减少每次调用的开销效果明显。

**口述模板（30–60 秒）**

1. 一句话结论：Tapable 根据注册的回调动态生成调用代码，用 `new Function` 编译，避免运行时判断。
2. 关键机制：HookCodeFactory 按钩子类型和回调类型拼接代码，注册变化时重新生成。
3. 对比或注意点：生成的代码是平铺的，V8 更容易优化。
4. 可追问时补充：拦截器可以监控每个插件的耗时，ProgressPlugin 就基于此。

**常见追问**

- 回调执行顺序如何控制？→ `stage` 数值越小越早；`before: 'OtherPlugin'` 插入到指定插件之前。

---

### Q4. Tree Shaking 深入：webpack 和 Terser 各自负责什么？barrel 文件有什么问题？

- **难度**: senior
- **标签**: Tree Shaking, 原理

**答案**

**分工**：

- **webpack**：
  - `providedExports`：每个模块提供哪些导出
  - `usedExports`：哪些导出被使用（ExportsInfo 中按运行时记录）
  - `sideEffects`：整个模块能否跳过，并把导入**直接重定向**到真正的来源模块
  - `innerGraph`：模块内顶层声明之间的引用关系
  - `mangleExports`：压缩导出名
  - 代码生成时不再输出未使用导出的定义
- **Terser / SWC minify**：真正删除没有引用的变量、函数和不可达分支；依据 `/*#__PURE__*/` 判断调用是否可以删除

**barrel 文件问题**：

```js
// components/index.js
export * from './Button'
export * from './Table'     // Table 依赖了一个大型表格库
export * from './Chart'     // Chart 依赖了 echarts
```

`import { Button } from './components'`：

- 如果包**没有声明 sideEffects: false**：Table 和 Chart 模块都会被执行（可能有副作用），echarts 进入产物
- 即使声明了无副作用，webpack 仍然需要**构建**所有被重导出的模块才能知道导出列表，构建时间变长
- 5.108 的 **lazy barrel** 优化：对无副作用的 barrel，未使用的重导出目标推迟构建，减少构建时间
- 5.110 的性能提示会报告「没有被使用的 barrel 重导出」

**CommonJS 的 Tree Shaking**：webpack 5 支持部分写法，如 `exports.foo = ...`、`module.exports.foo = ...`、`const { foo } = require('x')`、`require('x').foo`；`module.exports = { ... }` 整体赋值或动态访问则无法分析。

**口述模板（30–60 秒）**

1. 一句话结论：webpack 负责分析和标记，压缩器负责删除；sideEffects 负责跳过整个模块。
2. 关键机制：ExportsInfo 记录每个导出的提供和使用状态，sideEffects 优化会把导入重定向到真正来源，跳过中间的 barrel。
3. 对比或注意点：barrel 文件即使能删除产物中的代码，也会拖慢构建；没有 sideEffects 声明时会把整个库拉进来。
4. 可追问时补充：5.108 的 lazy barrel 和 5.110 的性能提示专门针对 barrel 问题。

**常见追问**

- 函数调用返回的常量能删除吗？→ `export const x = createX()` 默认不能，要加 `/*#__PURE__*/ createX()`。

---

### Q5. contenthash 不稳定，怎么系统地排查？

- **难度**: senior
- **标签**: 缓存, 排查

**答案**

现象：两次构建源码相同（或只改了一处），却有很多文件 hash 变化，导致 CDN 缓存命中率低。

排查步骤：

1. **复现**：在同一台机器上连续构建两次，对比产物文件名。不一致说明构建本身不确定；一致但在不同机器上不同，说明和环境有关
2. **对比产物内容**：对 hash 变化的同名 chunk 做 diff，看具体哪里变了
3. 常见原因：

| 差异内容 | 原因 | 解决 |
|----------|------|------|
| 模块 ID 变化 | 使用了 `natural` 自增 ID | `moduleIds: 'deterministic'` |
| 入口 chunk 中的 chunk 文件名映射变化 | 运行时没有抽离 | `runtimeChunk: 'single'` |
| 包含绝对路径 | Loader 或插件把绝对路径注入了代码 | 改为相对路径；设置 `output.devtoolModuleFilenameTemplate` |
| 包含时间戳、构建号 | DefinePlugin 注入了时间 | 只注入到单独的文件或 HTML 中 |
| 模块顺序变化 | 某些插件的处理结果依赖文件系统遍历顺序 | 排查插件，确保排序稳定 |
| 内容未变但 hash 变了 | 没有开启 realContentHash | 开启 |
| 不同机器构建不同 | 依赖版本不同、Node 版本不同 | 锁定 lock 文件和 Node 版本 |

4. **CI 中加入检查**：同一提交构建两次，产物 hash 必须一致

**口述模板（30–60 秒）**

1. 一句话结论：先确定是构建不确定还是环境不同，再 diff 变化的产物定位原因。
2. 关键机制：常见原因是模块 ID、运行时、绝对路径、时间戳和模块顺序。
3. 对比或注意点：只看文件名是不够的，必须对比文件内容找到差异。
4. 可追问时补充：在 CI 中验证构建的确定性，防止回退。

**常见追问**

- 依赖升级一个小版本，为什么所有 vendors 都变了？→ 所有第三方库合并在一个 vendors chunk 中；按变化频率拆分可以减小影响范围。

---

### Q6. HMR 运行时的细节：accept、decline、dispose、invalidate 分别有什么用？

- **难度**: senior
- **标签**: HMR, 运行时

**答案**

```js
if (import.meta.webpackHot) {               // ESM 中使用 import.meta.webpackHot，CJS 中使用 module.hot
  // 接受自身更新：当前模块会被重新执行
  import.meta.webpackHot.accept()

  // 接受依赖的更新：依赖变化时执行回调，当前模块不重新执行
  import.meta.webpackHot.accept('./store', () => { rebind() })

  // 更新前清理副作用，并把状态传给新模块
  import.meta.webpackHot.dispose(data => {
    clearInterval(timer)
    data.count = count
  })

  // 新模块中读取旧模块传下来的状态
  const prev = import.meta.webpackHot.data?.count

  // 声明本模块不能热更新，更新时整页刷新
  // import.meta.webpackHot.decline()

  // 在模块运行时发现无法安全更新时，主动让更新向上冒泡
  // import.meta.webpackHot.invalidate()
}
```

更新算法（`hotApply`）：

1. 对每个变化的模块，沿父模块向上查找：某个模块 accept 了这个依赖，或者某个模块 accept 了自己，就停止；遇到 decline 或到达入口，更新失败
2. 收集需要被替换的「过期模块」集合
3. 依次执行过期模块的 dispose 回调，从缓存中删除，并从父模块的子模块列表中移除
4. 注册新的模块函数，重新执行 accept 自身的模块，执行 accept 依赖的回调
5. 回调中抛出异常时，可以通过 `module.hot.status()` 看到 `fail`，dev-server 会整页刷新

**容易出问题的地方**：模块有副作用（定时器、事件监听、全局注册）但没有 dispose，每次热更新都叠加一份，导致重复执行和内存泄漏。

**口述模板（30–60 秒）**

1. 一句话结论：accept 声明更新边界，dispose 清理副作用并传递状态，decline 拒绝热更新，invalidate 主动冒泡。
2. 关键机制：从变化模块向上找 accept 边界，删除过期模块缓存并重新执行。
3. 对比或注意点：有副作用的模块必须写 dispose，否则会重复注册。
4. 可追问时补充：框架的热更新插件就是自动注入这些代码。

**常见追问**

- 为什么热更新几次之后页面变卡？→ 副作用没有在 dispose 中清理，定时器或监听器累积。

---

### Q7. Module Federation 深入：shared 的版本协商、eager、MF 2.0

- **难度**: senior
- **标签**: Module Federation, 微前端

**答案**

**shared 配置项**：

| 选项 | 作用 |
|------|------|
| `singleton` | 整个页面只使用一个版本（React、Vue、状态管理库必须设置） |
| `requiredVersion` | 需要的版本范围，不满足时警告（singleton）或使用自己的版本 |
| `strictVersion` | 不满足时直接报错 |
| `eager` | 共享模块打包进入口 chunk，同步可用，不需要异步边界，但会增大入口体积且失去共享 |
| `shareScope` | 共享作用域名称，用于隔离不同的共享组 |

**版本协商**：

1. 每个应用初始化时，把自己 shared 的依赖（版本 + 加载函数）注册到共享作用域
2. 使用时，非 singleton：从已注册版本中选择满足 `requiredVersion` 的最高版本
3. singleton：使用已注册的最高版本，即使不满足要求（只警告）
4. 因此**版本不兼容是运行时问题**，构建时无法发现

**异步边界的原因**：共享依赖需要等所有容器初始化、完成版本协商之后才能确定使用哪个版本，所以第一次使用前必须有一个异步点。

**Module Federation 2.0**（`@module-federation/enhanced`，Rspack 内置）：

- **manifest**（`mf-manifest.json`）：描述暴露的模块、共享依赖、资源列表，支持预加载和部署平台集成
- **运行时 API**：`loadRemote`、`registerRemotes` 动态注册 remote，不需要在构建时写死地址
- **运行时插件**：自定义加载失败重试、降级、修改共享策略
- **类型提示**：自动生成并拉取 remote 的类型声明
- **Chrome 开发者工具**：把线上 remote 替换为本地开发版本

**生产实践要点**：remote 加载失败要有降级 UI（ErrorBoundary）；remote 版本化部署，host 通过配置中心指定版本以便回滚；共享依赖的大版本升级需要所有应用协调。

**口述模板（30–60 秒）**

1. 一句话结论：shared 通过共享作用域在运行时协商版本，singleton 保证只有一个实例。
2. 关键机制：每个容器初始化时注册自己的版本，使用时按 requiredVersion 选择版本；因为要先协商，所以需要异步边界。
3. 对比或注意点：版本不兼容只能在运行时发现；eager 能去掉异步边界但会失去共享。
4. 可追问时补充：MF 2.0 提供 manifest、动态 remote、运行时插件和类型提示，生产上要有加载失败的降级和版本回滚。

**常见追问**

- remote 发布了不兼容的改动，host 立刻挂了怎么办？→ remote 地址带版本号，由配置中心控制 host 使用哪个版本，实现灰度和回滚；暴露的模块视为公共 API 做兼容性管理。

---

### Q8. 运行时代码由哪些部分组成？ESM 输出模式有什么不同？

- **难度**: senior
- **标签**: 运行时

**答案**

webpack 5 的运行时由一组 **RuntimeModule** 组成，**按需注入**：只有用到了某个功能才会生成对应的代码。

| 运行时变量 | 功能 |
|------------|------|
| `__webpack_require__` | 模块加载 |
| `.m` / `.c` | 模块表 / 模块缓存 |
| `.d` / `.r` / `.o` / `.n` | 定义导出 / 标记 ESM / hasOwn / 获取默认导出（兼容 CJS） |
| `.e` / `.f` / `.u` / `.l` | 加载 chunk / chunk 加载策略 / chunk 文件名 / 加载脚本 |
| `.p` | publicPath |
| `.O` | 等待依赖 chunk 加载完成后执行入口（splitChunks 拆出初始 chunk 时） |
| `.hmrF` 等 | HMR 相关 |

chunk 加载方式由 `output.chunkLoading` 决定：`jsonp`（web）、`import`（ESM）、`require` / `async-node`（Node）、`import-scripts`（Worker）。

**ESM 输出**（`output.module: true`，5.111 起稳定）：

- 入口产物是真正的 ES 模块（`<script type="module">`），使用 `export` 导出
- 异步 chunk 使用原生 `import()` 加载，不再需要 JSONP 全局变量
- 打包库时输出干净的 ESM，便于使用方 Tree Shaking
- `futureDefaults` 下，target 支持 ESM 时默认开启

**口述模板（30–60 秒）**

1. 一句话结论：运行时由按需注入的 RuntimeModule 组成，负责模块加载、chunk 加载和互操作。
2. 关键机制：`__webpack_require__` 上挂载各种辅助函数，chunk 加载方式根据 target 选择 jsonp、import 或 require。
3. 对比或注意点：ESM 输出使用原生 import 加载 chunk，没有全局变量冲突问题。
4. 可追问时补充：运行时抽成单独的 chunk 有利于长效缓存，也可以内联到 HTML。

**常见追问**

- 多个入口在同一个页面，各自带一份运行时会怎样？→ 模块会被执行两次（两个模块缓存），单例失效；需要 `runtimeChunk: 'single'`。

---

### Q9. 持久化缓存是怎么实现的？

- **难度**: senior
- **标签**: 缓存, 原理

**答案**

组成部分：

1. **缓存内容**：模块构建结果（loader 输出、解析出的依赖）、解析结果（resolve）、代码生成结果、部分插件自己的缓存（通过 `compilation.getCache(name)` 获取 API）
2. **序列化**：自研的序列化机制，对象需要注册序列化器（`makeSerializable`）；数据分成多个 pack 文件，按访问频率组织，新近修改的放在单独的文件中，避免每次重写全部数据（5.110 起使用 V8 的值序列化器编码）
3. **快照（snapshot）**：每个缓存项记录依赖的文件、目录、缺失的文件（missing dependencies，比如 resolve 时尝试过但不存在的路径）的状态
   - 项目文件：时间戳，必要时用内容 hash 兜底
   - `managedPaths`（默认 node_modules）：只比较包的 package.json 版本，速度快
   - `immutablePaths`：认为永远不变
4. **缓存失效**：
   - 快照校验失败 → 该项失效
   - `buildDependencies`（配置文件及其依赖的模块）变化 → **整个缓存失效**
   - `cache.version` 变化 → 整个缓存失效
5. **写入时机**：编译完成后在空闲时写入（`idleTimeout`），进程退出前确保写完

为什么 missing dependencies 很重要：解析 `import './a'` 时尝试了 `a.ts`、`a.tsx`、`a.js`，如果后来新建了 `a.ts`，缓存的解析结果（指向 `a.js`）必须失效。

**口述模板（30–60 秒）**

1. 一句话结论：把模块构建、解析和代码生成结果序列化到磁盘，用快照判断是否有效。
2. 关键机制：快照记录依赖文件、目录和缺失文件的状态，node_modules 只比较版本号；配置变化通过 buildDependencies 让整体失效。
3. 对比或注意点：直接修改 node_modules 文件版本号不变，缓存不会失效。
4. 可追问时补充：插件可以通过 Cache API 把自己的计算结果也存入持久化缓存。

**常见追问**

- 缓存越来越大怎么办？→ 设置 `maxAge`（未使用的缓存项过期时间）和 `maxMemoryGenerations`；CI 中按 lock 文件 hash 区分缓存。

---

### Q10. webpack 5.10x 的内置 CSS / HTML / TypeScript 支持对现有项目有什么影响？

- **难度**: senior
- **标签**: 新特性, 版本

**答案**

webpack 正在把常用的 Loader 和插件能力内置到核心中，为 webpack 6 做准备：

| 能力 | 版本 | 说明 |
|------|------|------|
| CSS（含 CSS Modules） | `experiments.css`，5.109 起默认 `"auto"` | 没有为 CSS 注册 Loader 时启用内置处理，可以替代 css-loader + MiniCssExtractPlugin |
| HTML 入口 | `experiments.html`，5.109 起默认 `"auto"` | HTML 可以作为入口，自动注入 JS 和 CSS；路线图目标是替代 html-webpack-plugin |
| TypeScript | `experiments.typescript`（5.107，futureDefaults 自动开启） | 使用 Node 22.6+ 的类型擦除，只支持可擦除语法，不支持 `.tsx`、`enum` |
| ESM 输出 | 5.111 稳定 | `output.module: true` |
| `import.meta.env` / `import.meta.glob` / `?raw` | 5.109 | 与 Vite 写法对齐 |
| 性能提示 | 5.110 | 报告构建中的常见问题 |

对现有项目的影响：

- **"auto" 的含义是「没有配置 Loader 才启用」**，已有 css-loader 配置的项目行为不变
- `experiments.futureDefaults: true` 可以提前体验 webpack 6 的默认值，适合新项目或在分支上评估升级
- 这些能力与 Vite 的写法对齐，降低了在工具之间迁移的成本

**口述模板（30–60 秒）**

1. 一句话结论：webpack 正在把 CSS、HTML、TS、ESM 输出等能力内置，减少对 Loader 和插件的依赖，为 webpack 6 铺路。
2. 关键机制：5.109 起 CSS 和 HTML 支持默认为 auto，没有配置 Loader 时才启用；5.111 ESM 输出稳定。
3. 对比或注意点：现有配置不受影响；内置 TS 只做类型擦除，不支持 tsx 和 enum。
4. 可追问时补充：可以用 futureDefaults 提前评估 webpack 6 的默认行为。

**常见追问**

- 现在应该把项目改成内置 CSS 吗？→ 新项目可以尝试；存量项目没有明显收益时不必急于迁移，等 webpack 6 稳定后统一评估。

---

## 二、性能与排查

### Q11. 构建很慢，你会怎么系统地定位瓶颈？

- **难度**: senior
- **标签**: 构建速度, 排查

**答案**

**第一步：明确是哪个场景慢**

- 冷启动构建、二次构建、开发模式启动、HMR 更新，这四个场景的瓶颈完全不同

**第二步：分阶段测量**

| 工具 | 用途 |
|------|------|
| `stats.timings` / `--profile --json` | 每个模块的构建耗时（包括各 Loader） |
| `ProfilingPlugin` | 生成 Chrome DevTools 可读的性能文件，查看各插件和钩子的耗时 |
| `infrastructureLogging: { debug: /webpack\.cache/ }` | 查看缓存命中情况 |
| `node --cpu-prof` | 整个进程的 CPU 火焰图 |
| `ProgressPlugin` | 观察卡在哪个阶段 |

**第三步：按阶段定位**

- **make 阶段慢**：Loader 慢（Babel、Sass、TS 类型检查）、模块数量多（引入了整个图标库、barrel 文件）、解析慢（extensions 太多）
- **seal 阶段慢**：Terser 压缩（通常最慢）、source map 生成、大量 chunk 的 splitChunks 计算
- **emit 慢**：文件系统写入、产物过多
- **二次构建慢**：缓存没有生效（快照失效、buildDependencies 配置错误）

**第四步：针对性优化后再次测量**，每次只改一处，记录数据

**口述模板（30–60 秒）**

1. 一句话结论：先区分场景，再分阶段测量，找到真正的瓶颈后针对性优化。
2. 关键机制：stats 看模块和 Loader 耗时，ProfilingPlugin 看插件和钩子耗时，CPU profile 看整体热点。
3. 对比或注意点：压缩和 source map 通常是生产构建中最慢的部分，开发环境则更多是 Loader 和模块数量。
4. 可追问时补充：优化要有前后数据对比，每次只改一个变量。

**常见追问**

- 二次构建和第一次一样慢？→ 检查缓存是否生效：配置文件中有随机值、环境变量导致 version 变化、CI 没有保存缓存目录。

---

### Q12. 讲一次大型项目构建提速的完整过程

- **难度**: senior
- **标签**: 构建速度, 项目经验, 高频

**答案**

参考案例（可以按自己的项目改写，数据要真实）：

**背景**：一个有 5000+ 模块的中后台项目，生产构建 8 分钟，开发启动 2 分钟，HMR 5–10 秒，影响开发效率和发布速度。

**分析**：

- ProfilingPlugin 显示：babel-loader 占 make 阶段 60%，Terser 占 seal 阶段 70%
- 开发启动时所有路由页面都被编译
- 二次构建没有提升 → 发现 CI 中缓存目录没有持久化
- bundle 分析发现图标库被完整打包

**措施与效果**：

| 措施 | 效果 |
|------|------|
| 开启持久化缓存，CI 中按 lock 文件 hash 缓存 `.cache` 目录 | 二次构建 8 分钟 → 2 分钟 |
| babel-loader → swc-loader，类型检查移到 ForkTsChecker 和 CI | 冷构建 -40% |
| Terser → SWC 压缩，CSS 使用 lightningcss | seal 阶段 -60% |
| 开发环境 lazyCompilation | 开发启动 2 分钟 → 15 秒 |
| 图标改为按需导入，修复 barrel 导入 | 模块数 -30%，产物 -20% |
| 最后迁移 Rspack（内置 swc 和 lightningcss） | 冷构建降至 1 分钟以内，HMR < 1 秒 |

**沉淀**：构建耗时纳入 CI 监控；构建配置统一封装，避免各项目重复踩坑。

**口述模板（30–60 秒）**

1. 一句话结论：先测量，再从缓存、转译器、压缩器、按需编译、减少模块数五个方面优化，最后考虑迁移工具。
2. 关键机制：用 ProfilingPlugin 和 stats 找到热点，每一步都有数据对比。
3. 对比或注意点：优化方向的优先级按投入产出比排序，缓存和转译器替换收益最大。
4. 可追问时补充：把构建耗时纳入监控，防止回退。

**常见追问**

- swc 替换 Babel 遇到了什么问题？→ 部分 Babel 插件（如特定的 i18n、埋点插件）没有对应的 swc 插件，需要改为 Loader 或 swc 的 Wasm 插件，或者保留少数文件走 Babel。

---

### Q13. 开发环境启动慢、HMR 慢怎么优化？

- **难度**: senior
- **标签**: 开发体验

**答案**

**启动慢**：

- `experiments.lazyCompilation: { entries: false, imports: true }`：异步 chunk 在被访问时才编译，大型多页面项目效果明显
- 持久化缓存（开发环境默认是内存缓存，可以改为 filesystem）
- 多入口项目只启动需要开发的入口（通过环境变量过滤）
- devtool 使用 `eval-cheap-module-source-map`
- 关闭开发环境不需要的优化：`optimization: { removeAvailableModules: false, removeEmptyChunks: false, splitChunks: false }`
- 转译器换为 swc / esbuild

**HMR 慢**：

- HMR 的耗时包括：重新编译变化的模块 → 重新 seal（所有 chunk 的优化和代码生成）→ 浏览器拉取和执行
- 模块数量多时 seal 阶段的开销与项目规模相关，这是 webpack 的架构限制
- 减少文件监听范围：`watchOptions.ignored: /node_modules/`
- 类型检查不阻塞 HMR（ForkTsChecker 的 `async: true`）
- 最终方案：迁移 Rspack（增量编译在 Rust 中完成）或 Vite（不打包）

**口述模板（30–60 秒）**

1. 一句话结论：启动慢靠 lazyCompilation 和缓存，HMR 慢的根本原因是每次都要对整个 chunk 图重新 seal。
2. 关键机制：lazyCompilation 让未访问的页面不编译；关闭开发环境不需要的优化；类型检查异步进行。
3. 对比或注意点：webpack 的 HMR 耗时与项目规模相关，这是架构限制。
4. 可追问时补充：架构限制的问题，迁移 Rspack 或 Vite 是根本解决办法。

**常见追问**

- lazyCompilation 有什么副作用？→ 第一次访问某个页面时需要等待编译；对依赖 SSR 或预渲染的场景不适用。

---

### Q14. 构建时 Node 内存溢出（JavaScript heap out of memory）怎么排查？

- **难度**: senior
- **标签**: 内存, 排查

**答案**

临时手段：`NODE_OPTIONS=--max-old-space-size=8192`，但需要找到真正的原因。

常见原因：

1. **source map**：大型项目生成完整的 source map 非常耗内存 → 检查 devtool 选择，生产使用 `hidden-source-map` 时确认第三方库的 source map 是否也被处理（`source-map-loader`）
2. **模块数量巨大**：引入了整个图标库、大量的 context module、意外打包了大型 JSON 或测试数据
3. **Terser 并行压缩**：每个 worker 都占用内存 → 减少 `parallel` 的数量，或使用 swc 压缩
4. **插件内存泄漏**：watch 模式下插件把 Compilation 对象保存在闭包或模块级变量中，每次重新编译都积累
5. **多配置并行构建**：导出数组配置时多个编译同时运行
6. **thread-loader 的 worker 数量过多**

排查：

- `node --heapsnapshot-near-heap-limit=1` 在接近上限时自动生成堆快照，用 Chrome DevTools 分析
- watch 模式下对比多次编译后的堆快照，找到持续增长的对象及其引用链

**口述模板（30–60 秒）**

1. 一句话结论：先临时调大内存，再用堆快照定位真正原因。
2. 关键机制：常见原因是 source map、模块数过多、并行压缩、插件泄漏 Compilation。
3. 对比或注意点：watch 模式下内存持续增长通常是插件持有了旧的 Compilation。
4. 可追问时补充：`--heapsnapshot-near-heap-limit` 可以在 OOM 前自动生成快照。

**常见追问**

- 写插件时如何避免泄漏？→ 不要在插件实例上保存 Compilation 或模块引用；需要跨编译的数据使用 Cache API 或只保存必要的原始值。

---

### Q15. 首屏性能优化中，构建层面能做什么？

- **难度**: senior
- **标签**: 性能, 体积优化

**答案**

按「关键路径」思路组织：

1. **减少首屏 JS**
   - 路由级懒加载 + 大组件懒加载
   - splitChunks 避免把非首屏依赖拉进首屏（检查 `vendors` 的范围）
   - Tree Shaking 生效检查、替换大依赖
2. **加快下载**
   - 压缩：Terser / SWC，brotli 预压缩（`compression-webpack-plugin`）
   - CDN + HTTP/2 或 HTTP/3
   - 长效缓存（contenthash）提高回访命中
3. **加快解析执行**
   - 现代 / 传统双构建：现代浏览器加载 ES2017+ 产物，体积更小、执行更快，不需要 polyfill
   - 减少运行时 polyfill
4. **优先级与预加载**
   - 关键资源 preload，下一页 prefetch
   - 关键 CSS 内联，非关键 CSS 异步加载
   - 运行时内联到 HTML 中，减少一次请求
5. **度量与预算**
   - 在 CI 中设置入口体积预算（`performance.maxEntrypointSize`）
   - 线上监控 LCP、INP 并关联版本

**口述模板（30–60 秒）**

1. 一句话结论：围绕关键路径：少加载、快下载、快执行、合理的优先级。
2. 关键机制：懒加载和拆包减少首屏 JS，压缩和缓存加快下载，现代构建减少解析执行时间。
3. 对比或注意点：不能只看产物总体积，要看首屏关键路径上的资源。
4. 可追问时补充：要有体积预算和线上指标监控，形成闭环。

**常见追问**

- 现代 / 传统双构建怎么实现？→ 见 [30-senior-qa Q24](30-senior-qa.md)。

---

### Q16. 线上出现 ChunkLoadError，原因和解决方案是什么？

- **难度**: senior
- **标签**: 线上问题, 发布, 高频

**答案**

现象：`ChunkLoadError: Loading chunk xxx failed`，用户点击某个页面时白屏。

常见原因：

1. **发布后旧 chunk 被删除**：用户打开的是旧版本页面（旧的运行时记录了旧的 chunk 文件名），新版本发布时覆盖了产物目录，旧 chunk 不存在 → 404
2. **网络问题**：弱网、CDN 节点故障、请求被浏览器插件拦截
3. **publicPath 错误**：部署到子路径或 CDN 地址配置错误
4. **CDN 回源时新文件还未上传完成**：发布顺序错误，先发布了 HTML

解决方案：

- **保留旧版本的静态资源**：产物上传到 CDN 时增量上传，不删除旧文件（保留几个版本或一段时间）—— 这是最根本的方案
- **发布顺序**：先上传静态资源，再更新 HTML
- **加载失败重试**：对 `import()` 包装重试逻辑；webpack 加载失败后会重置 chunk 状态，所以再次 `import()` 会重新请求
- **重试仍失败时刷新页面**：如果是版本变化导致的，刷新后加载新版本（要防止无限刷新，用 sessionStorage 记录）
- **版本检测**：定期请求 `version.json`，发现新版本时提示用户刷新
- **监控**：上报 ChunkLoadError 的数量、chunk 名、CDN 节点

**口述模板（30–60 秒）**

1. 一句话结论：最常见的原因是发布后旧 chunk 被删除，最根本的解决是保留旧版本资源。
2. 关键机制：旧页面的运行时记录的是旧文件名，覆盖发布导致 404。
3. 对比或注意点：发布顺序要先资源后 HTML；客户端加重试和刷新兜底，并防止无限刷新。
4. 可追问时补充：结合版本检测提示用户刷新，并监控错误率。

**常见追问**

- 重试为什么有效？→ webpack 加载失败后会把 chunk 状态重置为未加载，再次调用 import() 会重新创建 script 标签。

---

### Q17. 产物中同一个库出现多个版本怎么排查和处理？

- **难度**: senior
- **标签**: 体积优化, 依赖

**答案**

发现：

- bundle-analyzer 中看到同一个库出现多次
- 5.110 的性能提示会报告重复的包
- `duplicate-package-checker-webpack-plugin`
- `npm ls lodash` / `pnpm why lodash`

原因：

- 不同依赖要求的版本范围不兼容（`^3` 和 `^4`）
- lock 文件中没有去重（版本范围兼容但安装了多个版本）
- Monorepo 中各个包有独立的 node_modules
- 同一个包同时有 ESM 和 CJS 两个入口被分别引入

处理：

1. 版本兼容的情况：`npm dedupe` / `pnpm dedupe`
2. 强制统一：package.json 的 `overrides`（npm）/ `pnpm.overrides` / `resolutions`（yarn）
3. 构建层面：`resolve.alias` 指向同一位置（要确认 API 兼容）
4. 升级依赖，使它们使用相同的大版本
5. 单例库（React、styled-components）重复会导致运行时错误，必须解决

**口述模板（30–60 秒）**

1. 一句话结论：用分析工具发现重复，再通过 dedupe、overrides 或 alias 统一版本。
2. 关键机制：重复的原因是版本范围不兼容、lock 文件未去重、Monorepo 结构或双模式包。
3. 对比或注意点：强制统一版本前要确认 API 兼容；单例库重复会导致运行时错误。
4. 可追问时补充：可以在 CI 中加入重复依赖检查，防止回退。

**常见追问**

- 为什么 React 有两份会报错？→ Hooks 依赖 React 内部的全局状态（dispatcher），组件使用的 React 和渲染器使用的不是同一个实例。

---

## 三、工程与架构

### Q18. 写一个生产级的 webpack 插件需要注意什么？

- **难度**: senior
- **标签**: Plugin, 最佳实践

**答案**

1. **选择正确的钩子**：
   - 修改产物用 `compilation.hooks.processAssets` 并选择合适的 `stage`（如 `ADDITIONS` 添加文件、`OPTIMIZE_SIZE` 压缩、`SUMMARIZE` 汇总、`REPORT` 报告）
   - 不要在 `emit` 中修改产物内容
2. **使用正确的 API**：
   - `compilation.emitAsset(name, source)` / `updateAsset` / `deleteAsset`
   - 用 `webpack.sources`（`RawSource`、`ConcatSource`）构造产物，保留 source map
   - 从 `compiler.webpack` 获取 webpack 实例，避免安装了不同版本的 webpack
3. **支持缓存**：耗时计算使用 `compilation.getCache(name)` 存入持久化缓存
4. **兼容 watch 模式**：不在实例上保存 Compilation；使用 `thisCompilation` 还是 `compilation` 要清楚是否需要作用于子编译
5. **错误处理**：通过 `compilation.errors.push(new WebpackError(...))` 报告错误，而不是直接抛出，让 webpack 正常输出 stats
6. **性能**：避免在每个模块的钩子中做耗时操作；使用 `infrastructureLogger` 记录日志
7. **选项校验**：使用 `schema-utils` 校验选项
8. **不要修改 `compiler.options`**（在 apply 之后修改可能不会生效，且会影响其他插件）

**口述模板（30–60 秒）**

1. 一句话结论：选对钩子、用官方 API 操作产物、支持缓存和 watch、规范地报告错误。
2. 关键机制：产物修改使用 processAssets 的对应阶段和 emitAsset / updateAsset。
3. 对比或注意点：不要在 emit 中改产物、不要持有 Compilation、不要直接抛异常。
4. 可追问时补充：从 `compiler.webpack` 获取依赖，保证与当前运行的 webpack 版本一致，同时也便于兼容 Rspack。

**常见追问**

- 怎样让插件同时兼容 webpack 和 Rspack？→ 只使用公开钩子和 `compiler.webpack` 上的 API；Rspack 只实现了部分钩子，需要查兼容列表并测试。

---

### Q19. 大型 webpack 项目迁移到 Rspack，你怎么规划？有哪些风险？

- **难度**: senior
- **标签**: Rspack, 迁移

**答案**

**评估阶段**：

- 列出所有 Loader 和插件，对照 Rspack 的兼容列表；找出依赖 webpack 内部 API 的自研插件
- 用一个分支快速试跑，记录构建时间和产物差异

**迁移阶段**：

1. 替换依赖：`webpack` → `@rspack/core`，CLI 和 dev-server 同步替换
2. 替换为内置实现：`builtin:swc-loader`、`CssExtractRspackPlugin`、`SwcJsMinimizerRspackPlugin`、`LightningCssMinimizerRspackPlugin`、`HtmlRspackPlugin`
3. Babel 插件迁移：有 swc 对应插件就迁移，没有的单独对少数文件使用 babel-loader
4. 自研插件改造：只使用 Rspack 支持的钩子
5. 注意 Rspack 2.0 要求 Node 20.19+ / 22.12+，核心包为纯 ESM

**验证阶段**：

- 产物对比：chunk 数量、体积、主要模块的归属
- 功能回归：E2E 测试、关键页面人工验证
- 兼容性：目标浏览器下的语法降级和 polyfill 是否一致（swc 与 Babel 的转换结果可能不同）

**上线阶段**：

- 先在开发环境使用，生产仍用 webpack 一段时间；或者双构建并行对比
- 灰度发布，监控错误率和性能指标

**风险**：自研插件不兼容、swc 与 Babel 转换差异、CSS 处理顺序差异、source map 差异影响错误监控。

**口述模板（30–60 秒）**

1. 一句话结论：评估兼容性 → 替换依赖和内置实现 → 产物对比和回归 → 先开发后生产、灰度上线。
2. 关键机制：Rspack 兼容 webpack 配置，主要工作在 Babel 插件和自研插件的适配。
3. 对比或注意点：swc 和 Babel 的降级结果、CSS 顺序可能不同，必须做产物对比和完整回归。
4. 可追问时补充：可以先只在开发环境使用 Rspack，快速获得开发体验的收益，降低风险。

**常见追问**

- 开发用 Rspack、生产用 webpack 有什么问题？→ 两套构建可能出现行为不一致，只能作为过渡，最终要统一。

---

### Q20. 从 webpack 迁移到 Vite 有哪些坑？

- **难度**: senior
- **标签**: Vite, 迁移

**答案**

| 问题 | 原因 | 处理 |
|------|------|------|
| `require` 不可用 | 开发环境是原生 ESM | 改为 `import`，或使用插件转换 |
| `process.env.XXX` 不可用 | Vite 使用 `import.meta.env`，变量需要 `VITE_` 前缀 | 替换或通过 `define` 兼容 |
| `require.context` | webpack 特有 | 改为 `import.meta.glob` |
| 魔法注释失效 | webpack 特有 | 使用 Rollup 的 `manualChunks` 等 |
| 省略 `.vue` 等扩展名 | Vite 默认不尝试所有扩展名 | 配置 `resolve.extensions` 或补全扩展名 |
| CommonJS 依赖 | 开发时依赖预构建转换，个别包有问题 | `optimizeDeps.include` |
| Less / Sass 中的 `~` 前缀 | webpack 特有写法 | 改为别名 |
| 自定义 Loader | Vite 没有 Loader 概念 | 改写为 Vite 插件的 `transform` 钩子 |
| 首次加载慢 | 大项目开发时请求数巨大 | 预热常用文件（`server.warmup`）、减少 barrel 文件 |
| 开发与生产不一致 | 开发是原生 ESM，生产是打包 | 生产构建后做完整回归 |

**口述模板（30–60 秒）**

1. 一句话结论：主要问题是 webpack 特有语法、环境变量、CommonJS 依赖和自定义 Loader 的迁移。
2. 关键机制：Vite 开发时基于原生 ESM，不支持 require 和 webpack 的特有 API。
3. 对比或注意点：大项目开发时首次加载可能更慢；开发和生产构建方式不同。
4. 可追问时补充：大型项目如果只想提速，迁移 Rspack 的成本通常更低。

**常见追问**

- 怎么降低迁移风险？→ 先把 webpack 特有写法改为标准写法（`import.meta.glob`、`new URL`），让代码在两种工具下都能工作，再切换。

---

### Q21. 2026 年新项目，你怎么选择构建工具？

- **难度**: senior
- **标签**: 选型

**答案**

先看约束：

| 约束 | 倾向 |
|------|------|
| 使用 Next.js、Nuxt、Remix 等框架 | 跟随框架的默认工具（Turbopack、Vite 等） |
| 中小型 SPA、组件库开发 | Vite（生态好，开发体验好） |
| 大型项目、需要 webpack 生态（Module Federation、特定插件） | Rspack / Rsbuild |
| 存量 webpack 项目 | 先优化，再迁移 Rspack |
| 打包库 | tsdown、Rslib、Rollup 等 |
| 团队有大量 webpack 定制 | 继续 webpack 或 Rspack，迁移成本最低 |

评估维度：

- **构建性能**：冷启动、HMR、生产构建时间
- **生态和插件**：需要的功能是否有成熟实现
- **开发与生产一致性**
- **迁移成本**和团队熟悉度
- **长期维护**：项目活跃度、背后的团队

**口述模板（30–60 秒）**

1. 一句话结论：先看框架约束，再按项目规模和生态需求选择。
2. 关键机制：中小项目选 Vite；大型项目和需要 webpack 生态的选 Rspack；框架项目跟随框架。
3. 对比或注意点：选型要考虑一致性、迁移成本和长期维护，不只是速度。
4. 可追问时补充：Rust 工具链（Rolldown、Rspack、Turbopack）是明确的趋势。

**常见追问**

- webpack 会被淘汰吗？→ 短期不会：存量项目巨大，webpack 仍在持续迭代（5.111、webpack 6 规划）；但新项目选择 webpack 的比例在下降，Rspack 承接了其生态。

---

### Q22. 构建产物有哪些安全问题？

- **难度**: senior
- **标签**: 安全

**答案**

1. **source map 泄露**：公开的 map 文件暴露完整源码 → 使用 hidden source map，上传到监控平台后删除
2. **敏感信息注入**：通过 DefinePlugin / `.env` 把密钥打进了前端代码 → 前端只放公开配置，CI 中扫描产物中的密钥特征
3. **依赖供应链**：恶意包、被投毒的依赖 → lock 文件、`npm audit`、限制安装脚本、私有镜像、依赖审查
4. **CDN 资源篡改**：使用 SRI（Subresource Integrity），`webpack-subresource-integrity` 为 script 和 link 添加 `integrity` 属性
5. **CSP**：严格的 CSP 不允许内联脚本 → 运行时内联到 HTML 时需要 nonce，webpack 支持 `__webpack_nonce__` 为动态插入的 script 添加 nonce
6. **开发服务器暴露**：dev-server 监听 `0.0.0.0` 且关闭了 host 检查（`allowedHosts: 'all'`），可能被 DNS 重绑定攻击
7. **构建插件执行任意代码**：webpack 配置和插件在构建机上有完整权限，第三方插件同样要审查

**口述模板（30–60 秒）**

1. 一句话结论：source map 泄露、密钥注入、供应链、CDN 篡改和 CSP 是主要问题。
2. 关键机制：hidden source map、SRI、nonce 和产物扫描分别对应这些风险。
3. 对比或注意点：前端代码中的任何值都是公开的。
4. 可追问时补充：构建流程本身也是攻击面，构建机的权限和依赖都需要管控。

**常见追问**

- 怎么防止开发者误把密钥打进产物？→ CI 中用 gitleaks 等工具扫描产物，并约定只有特定前缀的环境变量会被注入。

---

### Q23. SSR 项目的构建是怎么组织的？

- **难度**: senior
- **标签**: SSR

**答案**

SSR 需要**两套构建**：

| | 客户端构建 | 服务端构建 |
|---|---|---|
| target | `web` / browserslist | `node` |
| 入口 | `entry-client` | `entry-server` |
| 输出 | 带 hash 的静态资源 | 供 Node 执行的服务端 bundle（通常 `library: commonjs2` 或 ESM） |
| 依赖 | 全部打包 | `externals` 排除 node_modules（`webpack-node-externals`），但需要转换的依赖（如 CSS 导入）要保留 |
| CSS | 抽取 | 不输出 CSS（只需要类名，如 CSS Modules 的 `exportOnlyLocals`） |
| 资源 | 输出文件 | 不重复输出，只生成相同的 URL |

关键点：

- **manifest**：客户端构建生成资源清单（如 `webpack-manifest-plugin` 或 React 的 loadable-components 统计文件），服务端渲染时根据渲染用到的组件注入对应的 script 和 link，避免水合时再加载
- **两边的模块 ID 和资源 URL 必须一致**
- 懒加载组件在服务端需要同步加载
- 开发时两套构建都要 watch，服务端 bundle 更新后要重新加载

现在通常直接使用 Next.js、Remix 等框架，它们内部处理了这些问题。

**口述模板（30–60 秒）**

1. 一句话结论：客户端和服务端两套构建，通过 manifest 关联。
2. 关键机制：服务端 target 为 node，排除 node_modules，不输出 CSS 和资源文件；客户端输出资源并生成 manifest。
3. 对比或注意点：两套构建的资源 URL 和 CSS 类名必须一致，否则水合不匹配。
4. 可追问时补充：服务端根据 manifest 注入首屏需要的 chunk，避免水合前的加载瀑布。

**常见追问**

- 为什么服务端构建不能全部 externals？→ 一些依赖导入了 CSS 或者是 ESM-only 的包，需要经过 webpack 处理。

---

### Q24. 现代 / 传统双构建怎么实现？还有必要吗？

- **难度**: senior
- **标签**: 兼容, 性能

**答案**

思路：同一份源码构建两次，

- **现代构建**：target 为支持 ES 模块的浏览器（如 `supports es6-module`），几乎不需要语法降级和 polyfill，体积小、执行快
- **传统构建**：target 为需要兼容的旧浏览器，完整降级和 polyfill

HTML 中：

```html
<script type="module" src="/modern/main.js"></script>
<script nomodule src="/legacy/main.js"></script>
```

支持 ES 模块的浏览器只执行 `type="module"`，旧浏览器只执行 `nomodule`。

实现：导出两个 webpack 配置（数组），各自设置 browserslist 环境和输出目录，由 HTML 插件合并注入。

**还有必要吗？** 2026 年绝大多数用户的浏览器都支持现代语法，很多项目直接只构建现代版本，把最低支持版本设置为 ES2020 左右。只有需要兼容旧设备（如部分政企、IoT 场景）的项目才需要双构建。

**口述模板（30–60 秒）**

1. 一句话结论：构建现代和传统两套产物，用 module / nomodule 让浏览器自动选择。
2. 关键机制：现代版本不降级、不打 polyfill，体积和执行时间显著减少。
3. 对比或注意点：构建时间翻倍，需要评估用户浏览器分布。
4. 可追问时补充：现在多数项目只构建现代版本，根据实际用户数据决定兼容范围。

**常见追问**

- 怎么决定兼容范围？→ 根据线上用户的浏览器统计数据，结合业务要求确定 browserslist。

---

### Q25. 大型团队如何统一和治理构建配置？

- **难度**: senior
- **标签**: 工程化, 治理

**答案**

问题：每个项目各自维护 webpack 配置，版本和优化水平参差不齐，升级困难。

方案：

1. **封装构建工具**：基于 webpack / Rspack 封装统一的 CLI 或预设（或直接使用 Rsbuild），项目只写少量业务配置；提供受控的扩展点（如 `modifyConfig` 钩子），避免直接修改底层配置
2. **版本管理**：构建工具以 npm 包发布，语义化版本；大版本升级提供迁移脚本（codemod）
3. **统一的优化基线**：缓存、转译器、拆包、压缩、source map、体积预算在工具中默认配置
4. **度量与可观测**：构建时间、产物体积、缓存命中率上报到平台，按项目展示趋势
5. **CI 门禁**：体积预算、重复依赖检查、构建确定性检查
6. **构建缓存共享**：Monorepo 中使用 Turborepo / Nx 的远程缓存

**口述模板（30–60 秒）**

1. 一句话结论：封装统一的构建工具，用度量和门禁持续治理。
2. 关键机制：项目只写业务配置，优化基线由工具保证；通过扩展点处理特殊需求。
3. 对比或注意点：封装要保留扩展能力，否则项目会绕过工具。
4. 可追问时补充：构建数据上报平台，可以发现退化并推动优化。

**常见追问**

- 自研封装还是用 Rsbuild？→ 新的封装可以基于 Rsbuild 再加团队约定，减少维护成本。

---

### Q26. 如何建立构建产物的体积治理机制？

- **难度**: senior
- **标签**: 体积优化, 治理

**答案**

1. **预算**：为入口、首屏和单个异步 chunk 设置体积上限（`performance.maxEntrypointSize`、`maxAssetSize`，或使用 size-limit）
2. **CI 对比**：每个 PR 构建后与主分支对比 stats，在 PR 中评论体积变化（新增了哪些模块、哪个 chunk 变大）；超出预算时阻止合并或需要审批
3. **新依赖审查**：新增依赖时检查体积（bundlephobia）、是否支持 Tree Shaking、是否有重复
4. **可视化与趋势**：每次发布记录体积数据，长期跟踪趋势
5. **定期治理**：清理未使用的依赖（knip、depcheck）、未使用的导出和文件
6. **关联线上指标**：体积变化与 LCP 等指标的关系，用数据证明治理的价值

**口述模板（30–60 秒）**

1. 一句话结论：预算 + CI 对比 + 依赖审查 + 趋势监控。
2. 关键机制：每个 PR 对比体积变化，超预算阻止合并，让体积问题在合入前被发现。
3. 对比或注意点：预算要关注首屏关键资源，而不是总体积。
4. 可追问时补充：结合线上性能指标，证明体积治理带来的收益。

**常见追问**

- 预算超出但业务确实需要这个功能怎么办？→ 先考虑懒加载和替换方案，确实需要的可以经过审批调整预算，并记录原因。
