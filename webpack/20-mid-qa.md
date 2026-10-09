# webpack 中级 Q&A（32 题）

> 目标：能讲清 webpack 的工作原理和常见优化手段的「为什么」，能排查常见问题。示例基于 webpack 5。

---

## 一、构建流程与扩展机制

### Q1. 说一下 webpack 的构建流程

- **难度**: mid
- **标签**: 原理, 高频

**答案**

整体可以分为五个阶段：

1. **初始化**
   - 合并命令行参数和配置文件，填充默认值
   - 创建 `Compiler` 对象
   - 调用每个插件的 `apply(compiler)` 注册钩子
   - `WebpackOptionsApply` 根据配置注册内置插件（如 `EntryPlugin`、各种优化插件）
2. **开始编译**：`compiler.run()` → 创建模块工厂 → 触发 `compile` 钩子 → 创建本次编译的 `Compilation` 对象
3. **make（构建模块图）**：从每个 entry 开始：
   - **resolve**：解析模块路径，确定要使用的 Loader
   - **build**：用 loader-runner 执行 Loader，把内容转换为 JS
   - **parse**：用 JavaScript 解析器生成 AST，找出 `import` / `require` 等依赖
   - 对每个依赖递归执行上述过程，最终得到完整的**模块依赖图**
4. **seal（封装与优化）**：
   - 分析导出的使用情况（Tree Shaking 标记）
   - 根据入口和动态 import 生成 chunk，执行 splitChunks
   - 生成模块 ID、chunk ID，计算 hash
   - 为每个模块生成代码，把 chunk 渲染为产物（assets）
   - 执行 `processAssets` 钩子（压缩等）
5. **emit（输出）**：触发 `emit` 钩子（最后修改产物的机会），把产物写入文件系统，触发 `done`

```text
初始化 → run → compile → make（resolve → loader → parse → 递归）→ seal（优化 → chunk → id → hash → 代码生成 → processAssets）→ emit → done
```

**口述模板（30–60 秒）**

1. 一句话结论：初始化、编译、构建模块图、封装优化、输出五个阶段。
2. 关键机制：make 阶段从入口递归，用 Loader 转换、用解析器找依赖，建成模块图；seal 阶段把模块分组成 chunk，优化并生成代码。
3. 对比或注意点：整个流程由 Tapable 钩子串联，插件在各阶段介入。
4. 可追问时补充：watch 模式下每次文件变化都会创建新的 Compilation，但 Compiler 只有一个，未变化的模块走缓存。

**常见追问**

- Loader 在哪个阶段执行？→ make 阶段构建每个模块时。
- 压缩在哪个阶段？→ seal 阶段的 `processAssets` 钩子（`OPTIMIZE_SIZE` 阶段）。

---

### Q2. Compiler 和 Compilation 有什么区别？

- **难度**: mid
- **标签**: 原理, 高频

**答案**

| | Compiler | Compilation |
|---|---|---|
| 数量 | 整个 webpack 进程中只有一个 | 每次编译创建一个（watch 下每次变更都新建） |
| 职责 | 管理全局生命周期：读取配置、启动编译、监听、输出文件 | 负责一次具体的编译：构建模块、生成 chunk、优化、生成产物 |
| 持有 | `options`、文件系统、全局钩子（`run`、`compile`、`make`、`emit`、`done`） | `modules`、`chunks`、`assets`、`moduleGraph`、`chunkGraph`，以及构建和优化相关的钩子 |

插件中常见的用法：在 Compiler 的 `thisCompilation` / `compilation` 钩子中拿到 Compilation，再注册 Compilation 的钩子。

```js
compiler.hooks.thisCompilation.tap('MyPlugin', compilation => {
  compilation.hooks.processAssets.tap(
    { name: 'MyPlugin', stage: compilation.constructor.PROCESS_ASSETS_STAGE_ADDITIONS },
    assets => { /* 读写产物 */ },
  )
})
```

**口述模板（30–60 秒）**

1. 一句话结论：Compiler 是全局唯一的调度者，Compilation 是一次编译的上下文。
2. 关键机制：Compiler 负责启动和输出，Compilation 负责构建模块图、分 chunk 和生成产物。
3. 对比或注意点：watch 模式下 Compilation 每次重建，所以插件不能把状态挂在 Compilation 上跨次使用。
4. 可追问时补充：`thisCompilation` 不会传递给子编译器（如 html-webpack-plugin 的子编译），`compilation` 会。

**常见追问**

- 子编译器（child compiler）是什么？→ 插件在当前编译中再发起一次独立编译，例如 html-webpack-plugin 编译模板、worker 编译。

---

### Q3. Tapable 是什么？有哪些钩子类型？

- **难度**: mid
- **标签**: Tapable, 高频

**答案**

Tapable 是 webpack 的事件流库，提供各种类型的钩子，webpack 的整个构建流程就是通过钩子串联起来的。

按执行方式分类：

| 钩子 | 特点 |
|------|------|
| `SyncHook` | 同步依次执行 |
| `SyncBailHook` | 同步，任一回调返回非 `undefined` 就中止 |
| `SyncWaterfallHook` | 同步，上一个回调的返回值作为下一个的参数 |
| `SyncLoopHook` | 同步，回调返回非 `undefined` 时从头重新执行 |
| `AsyncParallelHook` | 异步并行 |
| `AsyncParallelBailHook` | 异步并行，任一返回值就结束 |
| `AsyncSeriesHook` | 异步串行 |
| `AsyncSeriesBailHook` | 异步串行，可中止 |
| `AsyncSeriesWaterfallHook` | 异步串行，传递返回值 |

注册方式：

- `tap`：同步回调（同步钩子和异步钩子都能用）
- `tapAsync`：回调风格，最后一个参数是 `callback`
- `tapPromise`：返回 Promise

```js
const { AsyncSeriesHook } = require('tapable')

const hook = new AsyncSeriesHook(['compilation'])
hook.tapPromise('A', async c => { await doA(c) })
hook.tapAsync('B', (c, cb) => doB(c, cb))
await hook.promise(compilation)       // 触发
```

`tap` 时可以传 `stage`（数字越小越早）和 `before`（在指定插件之前）控制顺序。

**口述模板（30–60 秒）**

1. 一句话结论：Tapable 是 webpack 的插件机制核心，一组发布订阅式的钩子。
2. 关键机制：按同步异步、串行并行、是否可中断、是否传递返回值组合出多种钩子。
3. 对比或注意点：注册时用 `tap` / `tapAsync` / `tapPromise`，要和钩子类型匹配，同步钩子不能用异步注册。
4. 可追问时补充：Tapable 会根据注册的回调动态生成调用函数的代码（`new Function`），减少运行时判断，提高性能。

**常见追问**

- `make` 是什么类型的钩子？→ `AsyncParallelHook`，多个入口可以并行构建。
- `emit` 呢？→ `AsyncSeriesHook`。

---

### Q4. Loader 的执行顺序是怎样的？什么是 pitch？

- **难度**: mid
- **标签**: Loader, 高频

**答案**

**执行顺序**：一个 rule 中的 `use: [a, b, c]`，normal 阶段从右到左执行：`c → b → a`。

**pitch 阶段**：在执行 normal 方法之前，会先从左到右执行每个 Loader 的 `pitch` 方法：

```text
a.pitch → b.pitch → c.pitch → 读取文件 → c → b → a
```

**熔断**：如果某个 Loader 的 pitch 返回了非 `undefined` 的值，就跳过后续的 Loader 和读取文件，直接回到前一个 Loader 的 normal 方法：

```text
a.pitch → b.pitch（返回值）→ a（接收 b.pitch 的返回值）
```

**enforce 与内联**：多个 rule 匹配同一个文件时，按 `pre → normal → inline → post` 的顺序执行 normal 方法（pitch 阶段相反）。

```js
{ test: /\.js$/, enforce: 'pre', use: 'eslint-loader' }   // 最先处理源码
```

内联 Loader 的前缀：

| 前缀 | 作用 |
|------|------|
| `!` | 禁用配置中的 normal Loader |
| `-!` | 禁用 pre 和 normal Loader |
| `!!` | 禁用所有配置的 Loader（pre、normal、post） |

**pitch 的典型用途**：style-loader 在 pitch 中返回一段 JS 代码，用 `!!css-loader!./a.css`（剩余请求 `remainingRequest`）去 `require` CSS 内容，然后注入 `<style>`。这样它不需要拿到 css-loader 输出的 JS 字符串再去执行。

**口述模板（30–60 秒）**

1. 一句话结论：normal 从右到左，pitch 从左到右，pitch 有返回值就熔断。
2. 关键机制：执行顺序是 pitch 依次向右，读文件，再 normal 依次向左；多个 rule 按 pre、normal、inline、post 排序。
3. 对比或注意点：pitch 熔断后，只会回到前面 Loader 的 normal 方法。
4. 可追问时补充：style-loader 利用 pitch 加内联请求 `!!css-loader!file` 获取 CSS 模块的导出。

**常见追问**

- 为什么 Loader 设计成从右往左？→ 函数组合 `a(b(c(source)))` 的习惯，最靠近文件的 Loader 最先处理。

---

### Q5. 怎么写一个 Loader？Loader 上下文有哪些常用 API？

- **难度**: mid
- **标签**: Loader

**答案**

Loader 是导出一个函数的模块，`this` 是 loader context：

```js
/**
 * 把 .txt 文件中的 {{name}} 替换为配置的值，导出为字符串模块
 */
module.exports = function templateLoader(source) {
  const options = this.getOptions({            // 读取并校验配置
    type: 'object',
    properties: { vars: { type: 'object' } },
  })
  const result = source.replace(/\{\{(\w+)\}\}/g, (_, key) => options.vars?.[key] ?? '')
  return `export default ${JSON.stringify(result)}`
}
```

常用 API：

| API | 作用 |
|-----|------|
| `this.getOptions(schema?)` | 获取 Loader 配置并校验 |
| `this.async()` | 声明为异步 Loader，返回 callback |
| `this.callback(err, content, sourceMap?, meta?)` | 返回多个结果（如同时返回 source map） |
| `this.cacheable(false)` | 声明结果不可缓存（默认可缓存） |
| `this.addDependency(file)` | 添加文件依赖，该文件变化时重新构建 |
| `this.resourcePath` / `this.resourceQuery` | 当前文件路径 / 查询参数 |
| `this.emitFile(name, content)` | 输出额外文件 |
| `this.emitWarning` / `this.emitError` | 报告警告 / 错误 |
| `module.exports.raw = true` | 接收 Buffer 而不是字符串（处理二进制文件） |
| `module.exports.pitch` | 定义 pitch 方法 |

**口述模板（30–60 秒）**

1. 一句话结论：Loader 是一个函数，接收源码返回新源码，通过 this 访问上下文。
2. 关键机制：同步直接 return，异步用 `this.async()`；需要返回 source map 用 `this.callback`。
3. 对比或注意点：Loader 要保持纯函数、无副作用，才能被缓存和并行执行；读取了其他文件要 `addDependency`。
4. 可追问时补充：最后一个 Loader 必须输出 JS（或模块类型能识别的内容）。

**常见追问**

- Loader 中读取了配置文件，修改配置文件后没有重新编译？→ 需要 `this.addDependency(configPath)`。

---

### Q6. webpack 是怎么解析模块路径的？

- **难度**: mid
- **标签**: resolve

**答案**

webpack 使用 **enhanced-resolve** 解析路径：

1. **alias 替换**：先应用 `resolve.alias`
2. **判断路径类型**：
   - 绝对路径：直接使用
   - 相对路径：相对于当前文件所在目录
   - 模块路径（如 `react`）：从当前目录开始向上，依次在 `resolve.modules`（默认 `node_modules`）中查找
3. **找到包后确定入口**：
   - 有 `exports` 字段：按 `resolve.conditionNames`（如 `import`、`require`、`browser`、`development`）匹配
   - 没有 `exports`：按 `resolve.mainFields`（web 目标默认 `['browser', 'module', 'main']`）
4. **文件还是目录**：
   - 有扩展名就直接查找；否则依次尝试 `resolve.extensions`
   - 是目录则查找 `resolve.mainFiles`（默认 `index`）
5. **symlinks**：默认把软链接解析为真实路径

解析结果会被缓存；持久化缓存中也会保存解析结果和相关文件的快照。

**口述模板（30–60 秒）**

1. 一句话结论：enhanced-resolve 按 alias、路径类型、包入口、扩展名的顺序解析。
2. 关键机制：模块路径向上逐级查找 node_modules；包入口优先 exports 条件导出，其次 mainFields。
3. 对比或注意点：`exports` 会限制只能导入声明的子路径，深层导入会报错。
4. 可追问时补充：缩小解析范围（少写 extensions、配置 alias、指定 modules）可以加快构建。

**常见追问**

- 为什么同一个包被打包了两份？→ 不同依赖解析到不同版本或不同路径（如 Monorepo 中软链接 + 各自的 node_modules），用 `resolve.alias` 统一，或检查包管理器的去重。

---

### Q7. 打包后的代码是什么结构？`__webpack_require__` 做了什么？

- **难度**: mid
- **标签**: 运行时, 高频

**答案**

简化后的产物（development 模式，不使用 Scope Hoisting）：

```js
(() => {
  // 1. 模块表：模块 ID → 模块函数
  var __webpack_modules__ = {
    './src/math.js': (module, exports, __webpack_require__) => {
      __webpack_require__.r(exports)                          // 标记为 ESM
      __webpack_require__.d(exports, { add: () => add })      // 定义导出的 getter
      const add = (a, b) => a + b
    },
    './src/index.js': (module, exports, __webpack_require__) => {
      var _math = __webpack_require__('./src/math.js')
      console.log((0, _math.add)(1, 2))
    },
  }

  // 2. 模块缓存
  var __webpack_module_cache__ = {}

  // 3. 自己实现的 require
  function __webpack_require__(moduleId) {
    var cached = __webpack_module_cache__[moduleId]
    if (cached !== undefined) return cached.exports
    var module = (__webpack_module_cache__[moduleId] = { exports: {} })
    __webpack_modules__[moduleId](module, module.exports, __webpack_require__)
    return module.exports
  }

  // 4. 运行时辅助函数：.d 定义导出、.r 标记 ESM、.o hasOwnProperty、.e 加载 chunk、.p publicPath …

  // 5. 执行入口
  __webpack_require__('./src/index.js')
})()
```

关键点：

- 每个模块被包装成函数，实现作用域隔离
- **先缓存再执行**：模块执行前就放入缓存，从而支持循环依赖
- ESM 导出通过 getter 定义，实现 ESM 的「实时绑定」（live binding）
- 调用写成 `(0, _math.add)()`，保证 `this` 为 undefined，符合 ESM 语义

**口述模板（30–60 秒）**

1. 一句话结论：产物是一个自执行函数，包含模块表、模块缓存和自己实现的 require。
2. 关键机制：模块被包装成函数，`__webpack_require__` 查缓存、执行模块函数、返回 exports。
3. 对比或注意点：ESM 导出用 getter 实现实时绑定；先放缓存再执行，所以循环依赖拿到的是未执行完的 exports。
4. 可追问时补充：生产模式开启 Scope Hoisting 后，多个模块会被合并到一个函数作用域中，模块表会小很多。

**常见追问**

- `__webpack_require__.r` 是做什么的？→ 在 exports 上定义 `__esModule: true` 和 `Symbol.toStringTag: 'Module'`，用于 ESM 与 CommonJS 互操作。

---

### Q8. 动态 import 是怎么实现的？

- **难度**: mid
- **标签**: 运行时, 代码分割, 高频

**答案**

`import('./Page')` 会被编译为：

```js
__webpack_require__.e('src_Page_js')                                      // 加载 chunk
  .then(__webpack_require__.bind(__webpack_require__, './src/Page.js'))   // 执行模块
```

`__webpack_require__.e` 的过程（web 目标，JSONP 方式）：

1. 检查 `installedChunks[chunkId]` 状态：`0` 表示已加载；数组 `[resolve, reject, promise]` 表示加载中；`undefined` 表示未加载
2. 未加载时：创建 Promise 并记录，然后用 `__webpack_require__.l` 创建 `<script>` 标签，地址为 `publicPath + chunk 文件名`
3. chunk 文件的内容是：

```js
(self['webpackChunkmy_app'] = self['webpackChunkmy_app'] || []).push([
  ['src_Page_js'],                            // chunk ID
  { './src/Page.js': (module, exports, require) => { /* ... */ } },  // 模块
])
```

4. 运行时重写了这个全局数组的 `push` 方法（`webpackJsonpCallback`）：把新模块合并到模块表，将 chunk 标记为 `0`，调用 resolve
5. 如果 script 加载失败，reject 一个 `ChunkLoadError`，并把状态重置为 `undefined`，以便之后重试

`webpackChunkmy_app` 的名字来自 `output.chunkLoadingGlobal`，默认基于 `output.uniqueName`（取自 package.json 的 name）。

**口述模板（30–60 秒）**

1. 一句话结论：动态 import 被编译为 `__webpack_require__.e` 加载 chunk，再 `__webpack_require__` 执行模块。
2. 关键机制：通过插入 script 标签加载 chunk 文件，chunk 文件调用全局数组的 push，运行时拦截 push 注册模块并 resolve Promise。
3. 对比或注意点：同一个页面上有多个 webpack 应用时，要保证 `uniqueName` 不同，否则全局变量冲突。
4. 可追问时补充：ESM 输出（`output.module`）时用原生 `import()` 加载 chunk，不再使用 JSONP。

**常见追问**

- 一个异步 chunk 依赖另一个公共 chunk 怎么办？→ `__webpack_require__.e` 会并行加载所有需要的 chunk（`Promise.all`）。

---

### Q9. HMR 的原理是什么？

- **难度**: mid
- **标签**: HMR, 高频

**答案**

完整链路：

1. **监听文件**：webpack watch 模式监听文件变化，触发增量编译（只重新构建变化的模块）
2. **生成更新文件**：编译完成后生成两个文件：
   - `[runtime].[旧hash].hot-update.json`：manifest，说明哪些 chunk 有更新、哪些被删除
   - `[chunk].[旧hash].hot-update.js`：包含更新后的模块代码
3. **通知浏览器**：webpack-dev-server 通过 WebSocket 推送 `hash` 和 `ok` 消息（只推送 hash，不推送代码）
4. **客户端检查更新**：HMR runtime 调用 `module.hot.check()`，通过 fetch 拉取 manifest，再通过 script 标签加载更新的 chunk
5. **应用更新**（`hotApply`）：
   - 从变化的模块开始**向上冒泡**，查找声明了 `module.hot.accept` 的模块（accept 边界）
   - 执行旧模块的 `dispose` 回调（清理副作用，可通过 `data` 传递状态）
   - 从模块缓存中删除旧模块，用新代码重新执行
   - 执行 accept 回调
6. **失败回退**：冒泡到入口仍没有模块 accept，或者更新过程中出错，就整页刷新（`hot: 'only'` 时不刷新）

**口述模板（30–60 秒）**

1. 一句话结论：增量编译 + WebSocket 通知 + 客户端拉取更新 + 冒泡查找 accept 边界替换模块。
2. 关键机制：服务端只推 hash，客户端根据 hash 拉取 manifest 和更新 chunk；运行时从变化模块向上找 accept，删除旧缓存、执行新模块。
3. 对比或注意点：没有 accept 边界时回退整页刷新；HMR 不是「重新打包后刷新」。
4. 可追问时补充：style-loader、vue-loader、React Refresh 都是在模块中自动注入了 accept 代码。

**常见追问**

- 为什么修改某些文件总是整页刷新？→ 修改的模块及其所有上游都没有 accept，比如修改了入口文件或非组件的工具模块被入口直接引用。

---

### Q10. React Fast Refresh 和 HMR 是什么关系？

- **难度**: mid
- **标签**: HMR, React

**答案**

Fast Refresh 是**构建在 HMR 之上**的 React 组件热更新方案：

- `react-refresh/babel`（或 SWC 的 refresh 选项）在编译时给每个组件注册签名（包括使用了哪些 Hooks）
- `@pmmmwh/react-refresh-webpack-plugin` 为**只导出 React 组件**的模块自动注入 `module.hot.accept`
- 更新时，React 用新的组件函数重新渲染，**保留 state**；如果 Hooks 的签名变了（增删 Hook），则重置该组件的状态

限制：

- 模块同时导出组件和非组件（如常量、工具函数）时，无法作为 accept 边界，更新会冒泡到上层
- class 组件不保留状态
- 可以用 `// @refresh reset` 强制重置

**口述模板（30–60 秒）**

1. 一句话结论：Fast Refresh 是基于 HMR 的 React 组件级热更新。
2. 关键机制：编译时注册组件和 Hook 签名，运行时让 React 用新组件函数重新渲染并保留状态。
3. 对比或注意点：一个文件只导出组件才能成为热更新边界。
4. 可追问时补充：Hook 签名变化时会重置状态，避免状态与新代码不匹配。

**常见追问**

- 为什么有些修改会整页刷新？→ 修改的文件导出了非组件内容，或者更新冒泡到了入口。

---

## 二、产物优化

### Q11. Tree Shaking 的原理是什么？什么情况下会失效？

- **难度**: mid
- **标签**: Tree Shaking, 高频

**答案**

webpack 的 Tree Shaking 是**标记 + 删除**两步：

1. **make 阶段**：收集每个模块提供了哪些导出（`providedExports`）
2. **seal 阶段**（`optimization.usedExports`）：从入口出发，分析每个导出是否被使用，没被使用的导出不再生成 `__webpack_require__.d` 导出定义，变成模块内的死代码（开发模式下可看到 `/* unused harmony export sub */` 注释）
3. **压缩阶段**：Terser 删除没有被引用的死代码
4. **`sideEffects`**：如果一个模块被标记为无副作用，并且它的导出一个都没被使用，整个模块直接跳过，不用 Terser 分析
5. **`innerGraph`**：分析模块内部的顶层变量引用关系，例如未使用的导出函数引用了另一个导入，那么这个导入也可以被删除

常见失效原因：

| 原因 | 说明 |
|------|------|
| 使用 CommonJS | `require` / `module.exports` 是动态的，webpack 5 只支持部分 CommonJS 写法 |
| Babel 把 ESM 转成了 CommonJS | `@babel/preset-env` 设置了 `modules: 'commonjs'` |
| 模块有副作用 | 顶层代码修改了全局对象、调用了函数，压缩器不敢删除 |
| 没有声明 sideEffects | webpack 无法跳过整个模块 |
| 导入整个命名空间并动态访问 | `import * as utils` 然后 `utils[name]` |
| 顶层函数调用没有 PURE 注解 | `export const x = createX()` 无法判断 `createX` 是否有副作用 |
| 开发模式 | 默认不开启 |
| 类的方法 | 类的未使用方法无法删除 |

**口述模板（30–60 秒）**

1. 一句话结论：webpack 负责标记未使用的导出，压缩器负责删除，sideEffects 负责整个模块的跳过。
2. 关键机制：基于 ESM 的静态结构，在 seal 阶段分析使用情况，不使用的导出变成死代码。
3. 对比或注意点：CommonJS、副作用代码、Babel 转换模块、动态访问都会导致失效。
4. 可追问时补充：顶层的函数调用可以加 `/*#__PURE__*/` 注解告诉压缩器这个调用没有副作用。

**常见追问**

- 怎么验证 Tree Shaking 生效了？→ 开发模式 + `usedExports: true` 查看产物中的 `unused harmony export` 注释，或者用 bundle-analyzer 对比。

---

### Q12. package.json 的 `sideEffects` 有什么用？为什么配置后 CSS 不见了？

- **难度**: mid
- **标签**: Tree Shaking

**答案**

`sideEffects` 声明哪些文件有副作用（执行时会影响外部，如修改全局变量、注入样式、注册 polyfill）。

```json
{
  "sideEffects": false
}
```

```json
{
  "sideEffects": ["*.css", "*.scss", "./src/polyfills.js"]
}
```

作用：当一个模块被标记为无副作用，且它的导出**一个都没被使用**时，webpack 会跳过整个模块，连它的依赖也不处理。对于 barrel 文件（`index.js` 重导出很多模块）效果尤其明显。

**CSS 消失的原因**：`import './style.css'` 没有使用任何导出，如果 CSS 被视为无副作用，就会被整个删掉。所以 `sideEffects` 需要包含样式文件。

也可以在 `module.rules` 中针对某类文件设置 `sideEffects: true`。

webpack 5.108 引入了 **lazy barrel** 优化：无副作用的 barrel 文件中未使用的重导出目标，连构建都会被推迟或跳过，进一步加快构建。

**口述模板（30–60 秒）**

1. 一句话结论：sideEffects 告诉 webpack 哪些文件可以在导出未使用时整体删除。
2. 关键机制：无副作用 + 导出没被使用 → 整个模块和它的依赖都被跳过。
3. 对比或注意点：设置为 false 时要把 CSS、polyfill 这类「只导入不使用」的文件排除在外。
4. 可追问时补充：它和 usedExports 互补：usedExports 删除模块内的未使用导出，sideEffects 删除整个模块。

**常见追问**

- `sideEffects: false` 设错了会怎样？→ 有副作用的代码（如注册全局组件、polyfill）被删除，线上功能异常，而且很难排查。

---

### Q13. Scope Hoisting 是什么？

- **难度**: mid
- **标签**: 优化

**答案**

Scope Hoisting（作用域提升，`optimization.concatenateModules`，由 ModuleConcatenationPlugin 实现）把多个 ESM 模块**合并到同一个函数作用域**中，而不是每个模块一个函数。

```js
// 合并前：两个模块函数 + __webpack_require__ 调用
// 合并后：
const add = (a, b) => a + b
console.log(add(1, 2))
```

好处：

- 减少函数包装代码，产物更小
- 减少运行时函数调用，执行更快
- 合并后压缩器能看到更多上下文，可以进一步内联和删除代码

无法合并的情况：

- 非 ESM 模块（CommonJS）
- 被多个 chunk 共享的模块
- 使用了 `eval`、被动态 import 的模块
- 开启 HMR 时

**口述模板（30–60 秒）**

1. 一句话结论：把多个 ESM 模块合并到一个作用域，减少包装代码和运行时开销。
2. 关键机制：ESM 的静态结构让 webpack 能安全地重命名变量并合并模块。
3. 对比或注意点：CommonJS 和跨 chunk 共享的模块不能合并。
4. 可追问时补充：可以在 stats 中用 `optimizationBailout` 查看模块不能合并的原因。

**常见追问**

- 和 Rollup 的产物有什么区别？→ Rollup 默认就是作用域合并的产物；webpack 在能合并时效果类似。

---

### Q14. splitChunks 的默认配置是什么？怎么设计拆包策略？

- **难度**: mid
- **标签**: 代码分割, 高频

**答案**

webpack 5 默认配置：

```js
splitChunks: {
  chunks: 'async',             // 只处理异步 chunk
  minSize: 20000,              // 拆出的 chunk 至少 20KB
  minChunks: 1,                // 至少被 1 个 chunk 引用
  maxAsyncRequests: 30,        // 按需加载时最大并行请求数
  maxInitialRequests: 30,      // 入口的最大并行请求数
  enforceSizeThreshold: 50000, // 超过此大小强制拆分，忽略其他限制
  cacheGroups: {
    defaultVendors: { test: /[\\/]node_modules[\\/]/, priority: -10, reuseExistingChunk: true },
    default: { minChunks: 2, priority: -20, reuseExistingChunk: true },
  },
}
```

常见的生产拆包策略：

```js
optimization: {
  runtimeChunk: 'single',
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      framework: {                       // 框架：几乎不变，长期缓存
        test: /[\\/]node_modules[\\/](react|react-dom|scheduler|react-router)[\\/]/,
        name: 'framework',
        priority: 40,
        enforce: true,
      },
      ui: {                              // 大型 UI 库单独拆分
        test: /[\\/]node_modules[\\/](antd|@ant-design)[\\/]/,
        name: 'ui',
        priority: 30,
      },
      vendors: {                         // 其他第三方库
        test: /[\\/]node_modules[\\/]/,
        name: 'vendors',
        priority: 20,
      },
      common: {                          // 业务公共模块
        minChunks: 2,
        priority: 10,
        reuseExistingChunk: true,
      },
    },
  },
}
```

设计原则：

- **按变化频率拆分**：框架几乎不变，业务代码经常变
- **按加载时机拆分**：首屏需要的进初始 chunk，其他按路由异步加载
- **避免过细**：请求数过多、chunk 太小会增加开销；HTTP/2 下可以适当细一些
- 一个大的 `vendors` 包含所有第三方库时，任何依赖升级都会让整个 vendors 缓存失效，而且非首屏的库也进入了首屏

**口述模板（30–60 秒）**

1. 一句话结论：默认只拆异步 chunk，生产中通常改为 all，并按变化频率设计 cacheGroups。
2. 关键机制：cacheGroups 通过 test 和 priority 决定模块归属，minSize、maxInitialRequests 等限制拆分程度。
3. 对比或注意点：给所有第三方库一个固定 name 会导致所有页面都加载它，异步页面的依赖也被拉进首屏。
4. 可追问时补充：用 `maxSize` 可以把过大的 chunk 再拆分，便于并行加载和缓存。

**常见追问**

- `name` 设置为固定值有什么问题？→ 所有入口和异步 chunk 中匹配的模块都会合并到这一个 chunk，导致首屏加载了不需要的代码。

---

### Q15. 怎么做长效缓存？为什么只改了一个文件，很多文件的 hash 都变了？

- **难度**: mid
- **标签**: 缓存, 高频

**答案**

长效缓存的目标：**内容不变的文件，文件名不变**，浏览器和 CDN 可以一直缓存。

配置：

```js
output: {
  filename: '[name].[contenthash:8].js',
  chunkFilename: '[name].[contenthash:8].js',
},
optimization: {
  runtimeChunk: 'single',            // 运行时单独抽出
  moduleIds: 'deterministic',        // 根据路径生成稳定的短 ID（production 默认）
  chunkIds: 'deterministic',
  realContentHash: true,             // 基于最终内容计算 hash（production 默认）
},
```

hash 意外变化的常见原因：

1. **运行时代码在入口 chunk 中**：运行时包含所有 chunk 的文件名映射，任何异步 chunk 的 hash 变化都会让入口 chunk 变化 → 用 `runtimeChunk: 'single'` 抽出
2. **模块 ID 不稳定**：使用自增数字 ID 时，新增一个模块会导致后续所有 ID 偏移 → `deterministic`
3. **chunk 内容确实变了**：一个公共模块被拆进了不同的 chunk、引入顺序变化
4. **CSS 和 JS 共享 chunkhash**：用 contenthash 分开
5. **构建环境不一致**：绝对路径、时间戳、环境变量被注入到代码中

**口述模板（30–60 秒）**

1. 一句话结论：contenthash + 抽离 runtime + 稳定的模块和 chunk ID。
2. 关键机制：运行时记录所有 chunk 的文件名，不抽离的话任何 chunk 变化都会导致入口变化；自增 ID 会因新增模块而偏移。
3. 对比或注意点：HTML 不能长缓存，静态资源才能强缓存。
4. 可追问时补充：runtime 很小，也可以内联到 HTML 中，省掉一个请求。

**常见追问**

- `realContentHash` 是什么？→ hash 先根据模块计算，压缩等处理后再根据最终内容重新计算，保证内容相同 hash 一定相同。

---

### Q16. 构建速度怎么优化？

- **难度**: mid
- **标签**: 构建速度, 高频

**答案**

先分析，再优化。按收益排序：

1. **持久化缓存**（收益最大）：`cache: { type: 'filesystem' }`，二次构建通常能快数倍
2. **替换转译器**：用 `swc-loader` 或 `esbuild-loader` 代替 babel-loader，类型检查交给独立进程
3. **压缩器**：`TerserPlugin` 使用 `minify: TerserPlugin.swcMinify` 或 `esbuildMinify`；CSS 使用 lightningcss
4. **缩小范围**：
   - Loader 配置 `include: path.resolve('src')` / `exclude: /node_modules/`
   - `resolve.extensions` 尽量短，常用的放前面
   - `module.noParse` 跳过已经打包好且没有依赖的库
5. **开发环境**：
   - `devtool: 'eval-cheap-module-source-map'`
   - `experiments.lazyCompilation`：入口和异步 chunk 被访问时才编译
   - 关闭不需要的优化（`optimization.splitChunks: false`、`removeAvailableModules: false`）
6. **多进程**：`thread-loader` 适合非常耗时的 Loader；使用 swc 后通常不再需要（进程通信本身有开销）
7. **升级工具**：webpack 5 最新版本、Node 新版本；终极方案是迁移 Rspack

已过时的方案：`DllPlugin`、`HardSourceWebpackPlugin`、`cache-loader`（webpack 5 的持久化缓存已取代）。

分析工具：`--profile --json` 生成 stats、`ProfilingPlugin`（生成 Chrome 可读的性能文件）、`speed-measure-webpack-plugin`（对 webpack 5 部分插件不兼容）。

**口述模板（30–60 秒）**

1. 一句话结论：先测量，再按缓存、转译器、缩小范围、开发环境专项的顺序优化。
2. 关键机制：持久化缓存避免重复构建，swc / esbuild 用原生代码代替 Babel，include / exclude 减少处理文件数。
3. 对比或注意点：thread-loader 有进程开销，小项目反而更慢；DLL 已经不需要了。
4. 可追问时补充：如果优化后仍然很慢，迁移 Rspack 通常可以再快 5–10 倍，且改动较小。

**常见追问**

- 你们项目实际优化了多少？→ 需要准备一个有数据的案例，见 [30-senior-qa Q12](30-senior-qa.md)。

---

### Q17. 持久化缓存怎么配置？什么时候会失效？

- **难度**: mid
- **标签**: 缓存, 构建速度

**答案**

```js
module.exports = {
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename],          // 配置文件变化时缓存失效
    },
    version: process.env.APP_ENV,    // 不同环境变量产生不同的代码时，区分缓存
    name: `${process.env.NODE_ENV}-cache`,
  },
}
```

- 缓存默认存放在 `node_modules/.cache/webpack`
- 缓存的内容：模块的构建结果、解析结果、代码生成结果等
- **如何判断失效**：每个缓存项记录了依赖文件的**快照**（snapshot）。对项目文件默认比较时间戳（必要时比较内容 hash）；对 `node_modules` 中的包（`managedPaths`）比较 package.json 中的版本号

需要手动处理的失效场景：

- **配置文件或其依赖变化**：通过 `buildDependencies` 声明
- **环境变量影响了构建结果**（如 DefinePlugin 的值）：通过 `version` 区分
- **直接修改 node_modules 中的文件**（patch 但版本号不变）：缓存不会失效，需要清理缓存或把该包移出 `managedPaths`
- CI 中复用缓存需要保存和恢复 `node_modules/.cache` 目录

**口述模板（30–60 秒）**

1. 一句话结论：`cache.type: 'filesystem'` 把构建结果写入磁盘，二次构建复用。
2. 关键机制：每个缓存项都有依赖文件的快照，项目文件比较时间戳或 hash，node_modules 比较版本号。
3. 对比或注意点：配置和环境变量的变化需要通过 buildDependencies 和 version 声明，否则可能拿到错误的旧结果。
4. 可追问时补充：CI 中要缓存 `.cache` 目录，并在 lock 文件变化时更新缓存 key。

**常见追问**

- 修改了 Babel 配置没有生效？→ 将 babel 配置文件加到 `buildDependencies` 中。

---

### Q18. 产物体积怎么优化？

- **难度**: mid
- **标签**: 体积优化, 高频

**答案**

先用 bundle-analyzer 找到问题，再针对性优化：

1. **按需加载**：路由懒加载、大型组件（编辑器、图表）在使用时才加载
2. **Tree Shaking**：使用 ESM 版本的库（`lodash-es`），按名称导入；检查 sideEffects
3. **替换大依赖**：moment → dayjs；只引入需要的语言包（`IgnorePlugin` / `ContextReplacementPlugin`）
4. **去除重复依赖**：同一个库的多个版本 → `resolve.alias` 或包管理器的 dedupe / overrides
5. **合理的 polyfill 策略**：按目标浏览器和使用情况引入（`useBuiltIns: 'usage'`），现代浏览器单独构建
6. **压缩**：JS（Terser / SWC）、CSS（CssMinimizer / lightningcss）、图片、开启 gzip / brotli
7. **合理拆包**：首屏只加载必要代码
8. **externals + CDN**：特定场景
9. **检查打包进来的意外内容**：source map、大型 JSON、Node polyfill

**口述模板（30–60 秒）**

1. 一句话结论：先分析，再从「少加载、删无用、换小库、压缩」四个方向优化。
2. 关键机制：懒加载减少首屏代码，Tree Shaking 删除无用代码，替换和去重减少依赖体积。
3. 对比或注意点：关注的是首屏关键路径上的体积，而不是总体积。
4. 可追问时补充：可以在 CI 中设置体积预算，防止体积回退。

**常见追问**

- 体积优化之后如何验证效果？→ 看首屏 JS 传输体积和 LCP、TTI 等指标，而不只看总体积。

---

### Q19. source map 的原理是什么？生产环境怎么处理？

- **难度**: mid
- **标签**: source map

**答案**

source map 文件结构：

```json
{
  "version": 3,
  "file": "main.js",
  "sources": ["webpack://app/./src/index.ts"],
  "sourcesContent": ["..."],
  "names": ["add", "a", "b"],
  "mappings": "AAAA,SAASA,IAAIC..."
}
```

- `mappings` 用 **Base64 VLQ** 编码，每一段记录「产物列 → 源文件索引、源码行、源码列、名称索引」，并且都是相对上一段的偏移量，以压缩体积
- 产物末尾的 `//# sourceMappingURL=main.js.map` 让浏览器找到 map 文件

生产环境方案：

1. 构建时生成 `hidden-source-map`（不在产物中添加引用注释）
2. 上传到错误监控平台（Sentry 等），关联版本号
3. 部署前**从产物目录中删除** `.map` 文件，或者只允许内网访问
4. 监控平台根据 map 还原错误堆栈

**口述模板（30–60 秒）**

1. 一句话结论：source map 用 VLQ 编码记录产物位置到源码位置的映射。
2. 关键机制：mappings 中每段记录相对偏移，浏览器根据 sourceMappingURL 加载并还原。
3. 对比或注意点：生产环境 map 不能公开，否则源码泄露。
4. 可追问时补充：用 hidden-source-map 生成后上传到监控平台，然后删除。

**常见追问**

- 为什么开发环境推荐 `eval-cheap-module-source-map`？→ eval 方式重新构建快；cheap 只映射行，生成快；module 映射到 Loader 处理前的源码，便于调试。

---

### Q20. polyfill 怎么处理？应用和库有什么不同？

- **难度**: mid
- **标签**: 兼容, Babel

**答案**

| 方案 | 说明 | 适用 |
|------|------|------|
| `useBuiltIns: 'entry'` | 入口 `import 'core-js/stable'`，按目标浏览器替换为需要的模块 | 应用，最稳妥 |
| `useBuiltIns: 'usage'` | 按代码中使用的 API 自动引入 | 应用，体积小，但不分析 node_modules 时可能遗漏 |
| `@babel/plugin-transform-runtime` + `corejs: 3` | 引入不污染全局的 polyfill 版本，并复用 Babel 辅助函数 | **库** |
| 不处理 | 在文档中声明需要的环境，由使用方处理 | 库 |
| 动态 polyfill 服务 | 根据 UA 返回需要的 polyfill | 应用，需要可靠的服务 |

库不应该污染全局（修改 `Array.prototype` 等），否则可能和使用方的 polyfill 冲突。

webpack 5 不再自动 polyfill Node 核心模块（如 `buffer`、`process`），见 Q27。

**口述模板（30–60 秒）**

1. 一句话结论：应用用 preset-env 的 usage 或 entry 引入全局 polyfill，库用 transform-runtime 避免污染全局。
2. 关键机制：browserslist 决定目标环境，core-js 提供 API 实现。
3. 对比或注意点：usage 默认不分析 node_modules，第三方库用了新 API 可能缺 polyfill。
4. 可追问时补充：可以做现代和传统两套构建，现代浏览器不加载 polyfill。

**常见追问**

- 为什么只转语法不够？→ 语法转换无法处理 `Promise`、`Object.groupBy` 等运行时 API，需要 polyfill 提供实现。

---

### Q21. CSS 相关的优化有哪些？`Conflicting order` 警告是什么？

- **难度**: mid
- **标签**: CSS

**答案**

优化：

- 生产环境用 MiniCssExtractPlugin 抽取，并行加载、独立缓存
- CssMinimizerPlugin 压缩（可以用 lightningcss 提速）
- 异步 chunk 的 CSS 同样按需加载（MiniCssExtractPlugin 支持）
- 关键 CSS 内联（critters / beasties），其余异步加载
- 清除未使用的 CSS（PurgeCSS，Tailwind 已内置）

**`Conflicting order` 警告**：MiniCssExtractPlugin 合并多个模块的 CSS 到同一个文件时，发现不同入口或 chunk 中这些 CSS 的导入顺序不一致，无法确定一个满足所有顺序的合并结果。CSS 的顺序会影响层叠结果，所以会警告。

处理：

- 统一导入顺序
- 使用 CSS Modules 等局部作用域方案，让顺序不再重要
- 确认无影响后可以用 `ignoreOrder: true` 关闭警告

**口述模板（30–60 秒）**

1. 一句话结论：抽取、压缩、按需加载、关键 CSS 内联、清除未使用的样式。
2. 关键机制：CSS 跟随 chunk 拆分和按需加载。
3. 对比或注意点：Conflicting order 是因为不同 chunk 的 CSS 导入顺序不一致，合并后层叠可能不同。
4. 可追问时补充：使用局部作用域方案可以从根本上避免顺序问题。

**常见追问**

- 异步加载的组件 CSS 什么时候加载？→ 加载异步 chunk 时运行时同时插入对应的 `<link>`，并等待 CSS 加载完成。

---

### Q22. 打包后的循环依赖会出现什么问题？

- **难度**: mid
- **标签**: 运行时

**答案**

webpack 的模块在执行前就放入缓存，所以循环依赖不会死循环，但会拿到**未执行完的模块**：

```js
// a.js
import { b } from './b'
export const a = 'a'
console.log(b)

// b.js
import { a } from './a'
export const b = 'b'
console.log(a)          // 入口是 a.js 时：a 还没执行到 export，访问 a 抛出 ReferenceError（TDZ）
```

- ESM：通过 getter 实时绑定，访问时如果变量还在 TDZ 中就会报错；函数声明因为会提升，通常可用
- CommonJS：拿到的是部分完成的 `exports` 对象，属性为 `undefined`

常见症状：`Cannot access 'X' before initialization`、`X is not a function`、某个类继承时父类为 undefined。

排查与解决：

- `circular-dependency-plugin` 或 webpack 5.110 性能提示中的循环依赖报告
- 抽取公共部分到第三个模块；把顶层执行的代码改为在函数中延迟执行；避免 barrel 文件互相引用

**口述模板（30–60 秒）**

1. 一句话结论：循环依赖不会死循环，但会拿到还没初始化完成的导出。
2. 关键机制：模块先进入缓存再执行，循环时返回的是正在执行中的模块。
3. 对比或注意点：ESM 下访问未初始化的 const 会报 TDZ 错误，CommonJS 下是 undefined。
4. 可追问时补充：barrel 文件是循环依赖的常见来源。

**常见追问**

- 为什么开发环境正常，生产环境报错？→ Scope Hoisting 改变了模块的合并和执行顺序。

---

## 三、生态与工程

### Q23. Module Federation 是什么？基本原理是什么？

- **难度**: mid
- **标签**: Module Federation, 微前端, 高频

**答案**

Module Federation（模块联邦）是 webpack 5 引入的功能：**多个独立构建、独立部署的应用，在运行时互相共享模块**。

```js
// remote 应用（提供模块）
new ModuleFederationPlugin({
  name: 'shop',
  filename: 'remoteEntry.js',
  exposes: { './Cart': './src/Cart' },
  shared: { react: { singleton: true, requiredVersion: '^19.0.0' } },
})

// host 应用（使用模块）
new ModuleFederationPlugin({
  name: 'main',
  remotes: { shop: 'shop@https://shop.example.com/remoteEntry.js' },
  shared: { react: { singleton: true, requiredVersion: '^19.0.0' } },
})

// host 中使用
const Cart = React.lazy(() => import('shop/Cart'))
```

原理：

1. remote 构建时生成 `remoteEntry.js`，它是一个容器，提供 `init(shareScope)` 和 `get(moduleName)` 两个方法
2. host 运行时加载 `remoteEntry.js`，调用 `init` 把自己的共享依赖放入共享作用域（share scope）
3. 双方根据 `shared` 配置做**版本协商**：`singleton` 只使用一个版本，否则选择满足 `requiredVersion` 的最高版本
4. host 调用 `get('./Cart')` 拿到模块工厂，加载对应的 chunk 后执行

**异步边界**：共享依赖需要异步初始化，所以入口通常写成 `import('./bootstrap')`，否则会报 `Shared module is not available for eager consumption`。

**口述模板（30–60 秒）**

1. 一句话结论：MF 让独立部署的应用在运行时共享模块和依赖。
2. 关键机制：remote 暴露一个容器入口，host 加载后通过 init 共享依赖、通过 get 获取模块；共享依赖按版本协商。
3. 对比或注意点：React 等有全局状态的库要设置 singleton，入口需要异步边界。
4. 可追问时补充：现在通常使用 Module Federation 2.0（`@module-federation/enhanced`），提供 manifest、运行时插件、动态 remote 和类型提示。

**常见追问**

- MF 和 npm 包共享的区别？→ npm 包是构建时集成，更新需要重新构建发布所有使用方；MF 是运行时集成，remote 独立发布后立即生效，但需要处理版本兼容和运行时故障。

---

### Q24. 怎么用 webpack 打包一个库？

- **难度**: mid
- **标签**: 库

**答案**

```js
module.exports = {
  entry: './src/index.ts',
  output: {
    path: path.resolve('dist'),
    filename: 'index.js',
    library: { type: 'module' },      // ESM 输出
    module: true,                      // 5.111 起稳定，不再需要 experiments.outputModule
  },
  externals: { react: 'react', 'react-dom': 'react-dom' },   // peer 依赖外部化
  externalsType: 'module',
  optimization: { minimize: false },   // 库通常不压缩，交给使用方
}
```

`library.type` 常用值：`module`（ESM）、`commonjs2`、`umd`、`var`（全局变量）。

要点：

- **peer 依赖外部化**，否则使用方会出现两份 React
- 优先输出 ESM，保证使用方可以 Tree Shaking
- 配置 package.json 的 `exports`、`types`、`sideEffects`
- 类型声明用 `tsc --emitDeclarationOnly` 生成

实际上，**库更推荐用 Rollup、tsup、Rslib 等工具**：产物更干净，ESM 输出更成熟，多格式输出更方便。webpack 更适合打包应用。

**口述模板（30–60 秒）**

1. 一句话结论：通过 `output.library` 指定导出格式，peer 依赖用 externals 外部化。
2. 关键机制：webpack 5.111 起 ESM 输出已稳定。
3. 对比或注意点：库要提供 ESM 和类型声明，正确设置 exports 和 sideEffects。
4. 可追问时补充：库开发通常选 Rollup 系工具，webpack 的运行时代码对库来说是多余的。

**常见追问**

- 为什么库通常不压缩？→ 使用方会统一压缩，未压缩的代码便于调试，并且有利于使用方做 Tree Shaking。

---

### Q25. ESM 和 CommonJS 混用时 webpack 怎么处理？

- **难度**: mid
- **标签**: 模块化

**答案**

webpack 支持 ESM 和 CommonJS 混用，核心是 **interop（互操作）**：

- **ESM 导入 CommonJS**：`import x from 'cjs'` 时，如果 CommonJS 模块有 `__esModule: true` 标记（通常是 Babel 编译的），`x` 取 `exports.default`；否则 `x` 是整个 `module.exports`
- **CommonJS 引入 ESM**：`require('./esm')` 得到的是命名空间对象，默认导出在 `.default` 上

常见问题：

- `xxx is not a function` / `xxx.default is not a function`：默认导出的互操作不一致
- 在 ESM 文件中使用 `module.exports`：webpack 会把模块视为 CommonJS，导致导出异常
- 包的 `exports` 字段中 `import` 和 `require` 条件指向不同的实现，导致同一个包被加载两次（dual package hazard）

webpack 按以下方式判断模块类型：`.mjs` / `.cjs` 扩展名、package.json 的 `type` 字段、`module.rules` 中的 `type: 'javascript/esm'`；5.108 起顶层 await 和 `import.meta` 也会被视为 ESM 标记。

**口述模板（30–60 秒）**

1. 一句话结论：webpack 通过 `__esModule` 标记在 ESM 和 CommonJS 之间做默认导出的互操作。
2. 关键机制：ESM 导入 CJS 时，有 `__esModule` 则取 default，否则取整个 exports。
3. 对比或注意点：混用最容易出现默认导出不一致的问题；同一文件不要同时用两种语法。
4. 可追问时补充：双模式包可能被打包两份，导致单例失效。

**常见追问**

- 为什么混用会影响 Tree Shaking？→ CommonJS 模块无法静态分析，ESM 导入了 CommonJS 也无法删除未使用的部分。

---

### Q26. `import(\`./locales/${lang}\`)` 这种动态路径是怎么打包的？

- **难度**: mid
- **标签**: 代码分割

**答案**

webpack 无法在编译时知道 `lang` 的值，会创建一个 **context module**：把 `./locales/` 目录下**所有可能匹配的文件**都打包，每个文件生成一个异步 chunk，运行时根据实际路径选择加载。

```js
// 限制范围，避免打包无关文件
import(
  /* webpackInclude: /\.json$/ */
  /* webpackExclude: /\.test\./ */
  /* webpackChunkName: "locale-[request]" */
  `./locales/${lang}.json`
)
```

- 路径中必须包含静态的前缀目录，`import(variable)` 完全动态的路径无法打包
- `require.context(dir, recursive, regExp)`：webpack 特有 API，批量导入目录下的模块
- webpack 5.109 起支持 `import.meta.glob('./pages/*.tsx')`（Vite 风格），返回路径到懒加载函数的映射
- `ContextReplacementPlugin` 可以修改第三方库的 context（如只保留 moment 的中文语言包）

**口述模板（30–60 秒）**

1. 一句话结论：动态表达式会生成 context module，把目录下所有可能的文件都打包。
2. 关键机制：每个匹配的文件生成一个 chunk，运行时根据实际路径映射加载。
3. 对比或注意点：范围过大会打包很多无用文件，要用 webpackInclude 等注释限制。
4. 可追问时补充：新版本支持 `import.meta.glob`，与 Vite 写法统一。

**常见追问**

- moment 的语言包为什么很大？→ moment 内部用了动态 require 语言包，webpack 生成了包含所有语言的 context module。

---

### Q27. 升级 webpack 5 后报 `Module not found: Can't resolve 'crypto'` 怎么办？

- **难度**: mid
- **标签**: 升级, Node polyfill

**答案**

webpack 4 会**自动为 Node 核心模块提供浏览器 polyfill**（如 `buffer`、`crypto`、`path`、`process`），webpack 5 移除了这个功能，因为这些 polyfill 往往体积很大，且开发者并不知道自己引入了它们。

处理方式：

1. **首选：确认是否真的需要**。很多时候是误用了面向 Node 的库，换成浏览器版本的库
2. 不需要：`resolve.fallback: { crypto: false }`
3. 确实需要：手动添加 polyfill

```js
resolve: {
  fallback: {
    buffer: require.resolve('buffer/'),
    stream: require.resolve('stream-browserify'),
    crypto: false,
  },
},
plugins: [
  new webpack.ProvidePlugin({
    Buffer: ['buffer', 'Buffer'],
    process: 'process/browser',
  }),
],
```

**口述模板（30–60 秒）**

1. 一句话结论：webpack 5 不再自动 polyfill Node 核心模块，需要显式配置。
2. 关键机制：resolve.fallback 指定替代模块或 false，ProvidePlugin 注入全局变量。
3. 对比或注意点：优先检查是否误用了 Node 专用库，不要无脑补 polyfill。
4. 可追问时补充：这个变化的目的是让开发者感知到这些隐藏的体积。

**常见追问**

- `process is not defined`？→ 代码中直接访问了 `process`（而不是被 DefinePlugin 替换的 `process.env.XXX`），通过 ProvidePlugin 注入或修改代码。

---

### Q28. Monorepo 中使用 webpack 有哪些常见问题？

- **难度**: mid
- **标签**: Monorepo

**答案**

1. **workspace 包不被转译**：内部包通过软链接指向源码（TS / JSX），但 babel-loader 的 `exclude: /node_modules/` 把它排除了 → 软链接默认解析为真实路径（`resolve.symlinks: true`），所以用 `include` 列出 `packages/*/src`
2. **依赖重复**：每个包有自己的 node_modules，同一个库被打包多次（React 多实例导致 Hooks 报错）→ 用 `resolve.alias` 指向唯一位置，或者依赖提升到根目录
3. **内部包的入口**：package.json 的 `main` 指向 `dist` 但还没构建 → 开发时用 `exports` 的自定义条件（如 `"development": "./src/index.ts"`）指向源码，并配置 `resolve.conditionNames`
4. **缓存**：持久化缓存默认认为 `node_modules` 中的包不会变化（`managedPaths`），软链接的内部包会被解析为真实路径从而不受影响，但需要确认
5. **构建编排**：用 Turborepo / Nx 管理依赖顺序和缓存，只构建受影响的包

**口述模板（30–60 秒）**

1. 一句话结论：主要问题是内部包的转译、依赖重复和入口指向。
2. 关键机制：软链接被解析为真实路径，需要用 include 而不是 exclude 控制转译范围。
3. 对比或注意点：React 等单例库出现多份会导致运行时错误。
4. 可追问时补充：用 exports 的自定义条件让开发时直接引用源码，配合 Turborepo 做增量构建。

**常见追问**

- `Invalid hook call` 错误？→ 通常是打包了两份 React，用 `npm ls react` 检查，并用 alias 统一。

---

### Q29. Web Worker 和 WebAssembly 在 webpack 5 中怎么使用？

- **难度**: mid
- **标签**: Worker, WASM

**答案**

webpack 5 原生支持 Worker，不再需要 worker-loader：

```js
const worker = new Worker(new URL('./heavy.worker.js', import.meta.url), { type: 'module' })
worker.postMessage(data)
```

webpack 识别 `new Worker(new URL(..., import.meta.url))` 这个写法，把 worker 文件作为新的入口单独打包，并替换为正确的 URL。这个写法同时也是浏览器标准写法，不依赖 webpack 特有语法。

WebAssembly：

```js
// webpack.config.js：5.109 起 experiments.asyncWebAssembly 默认 "auto"
import { add } from './math.wasm'
```

- `asyncWebAssembly`：把 wasm 作为异步模块，支持 ESM 导入
- 通常配合 wasm-pack 生成的 JS 胶水代码使用

**口述模板（30–60 秒）**

1. 一句话结论：webpack 5 原生识别 `new Worker(new URL(..., import.meta.url))`，WASM 通过 asyncWebAssembly 支持。
2. 关键机制：Worker 文件被作为独立入口打包，URL 在构建时替换。
3. 对比或注意点：这是浏览器标准写法，迁移到 Vite 等工具也能用。
4. 可追问时补充：Worker 中的 publicPath 需要注意，跨域 CDN 加载 Worker 有同源限制。

**常见追问**

- 为什么 Worker 部署到 CDN 后无法创建？→ Worker 脚本要求同源，需要通过 Blob URL 或同源代理加载。

---

### Q30. 微前端场景下 webpack 有哪些需要注意的问题？

- **难度**: mid
- **标签**: 微前端

**答案**

多个独立构建的 webpack 应用运行在同一个页面中时：

1. **publicPath**：子应用的资源地址在运行时才确定，入口最前面设置 `__webpack_public_path__ = window.__INJECTED_PUBLIC_PATH_BY_QIANKUN__`（qiankun）
2. **全局变量冲突**：多个应用的 JSONP 全局变量同名会互相覆盖 → 每个应用设置不同的 `output.uniqueName`（或 `chunkLoadingGlobal`）
3. **库格式**：qiankun 需要子应用以 UMD 格式输出生命周期函数（`output.library.type: 'umd'`）
4. **样式隔离**：CSS Modules、命名前缀或 Shadow DOM
5. **公共依赖重复加载**：每个子应用都打包了 React → externals 共享或改用 Module Federation
6. **开发时跨域**：子应用的 devServer 需要设置 CORS 头

**口述模板（30–60 秒）**

1. 一句话结论：主要是运行时 publicPath、全局变量冲突、样式隔离和依赖共享。
2. 关键机制：用 `__webpack_public_path__` 动态设置资源前缀，用 uniqueName 避免 JSONP 全局变量冲突。
3. 对比或注意点：qiankun 方案基于 UMD 和 JS 沙箱，Module Federation 方案基于模块共享。
4. 可追问时补充：两种方案可以结合：qiankun 负责应用编排，MF 负责共享依赖。

**常见追问**

- 子应用的异步 chunk 404？→ publicPath 没有在入口最前面设置，或者设置得太晚。

---

### Q31. Vite 为什么开发时快？webpack 和 Vite 怎么选？

- **难度**: mid
- **标签**: 选型, Vite, 高频

**答案**

**Vite 开发时快的原因**：

1. **不打包**：利用浏览器原生 ESM，浏览器请求哪个模块，服务器才编译哪个模块，启动时间与项目规模基本无关
2. **依赖预构建**：用 esbuild（新版本用 Rolldown）把 node_modules 中的依赖预先打包为 ESM，并强缓存
3. **HMR 精确**：只需要让浏览器重新请求变化的模块，更新速度与项目规模无关

**webpack 慢的原因**：开发时也需要先构建完整的模块图并打包，项目越大启动越慢；JS 实现，单线程为主。

**Vite 的问题**：

- 大型项目首次加载时请求数巨大（上千个模块请求），页面打开慢
- 开发和生产使用不同的打包方式，可能出现行为不一致（Rolldown 正在统一两者）

**选型**：

| 场景 | 选择 |
|------|------|
| 新项目、中小型应用 | Vite |
| 大型存量 webpack 项目要提速 | Rspack（迁移成本低） |
| 复杂定制、依赖大量 webpack 特有插件 | 继续 webpack 或 Rspack |
| 框架内置（Next.js 等） | 跟随框架 |

**口述模板（30–60 秒）**

1. 一句话结论：Vite 开发时不打包，按需编译，所以启动和热更新与项目规模无关。
2. 关键机制：浏览器原生 ESM 请求模块，服务器按需转换；依赖预构建减少请求数。
3. 对比或注意点：超大项目首次加载请求过多；开发和生产构建方式不同可能不一致。
4. 可追问时补充：存量大型 webpack 项目更推荐迁移 Rspack，兼容 webpack 配置，改动小。

**常见追问**

- Vite 的生产构建为什么不用 esbuild？→ esbuild 在代码分割、CSS 处理和插件生态上不够完善，所以用 Rollup；新版本改用 Rolldown 兼顾速度和兼容性。

---

### Q32. Rspack 是什么？从 webpack 迁移有多大成本？

- **难度**: mid
- **标签**: Rspack, 选型

**答案**

Rspack 是用 Rust 实现的打包器，**架构和配置与 webpack 基本一致**，目标是在兼容 webpack 生态的前提下大幅提升构建速度。2026 年 4 月发布 2.0。

特点：

- 兼容 webpack 的配置、大部分 Loader 和插件（插件需要使用 Rspack 实现的钩子子集）
- 内置高性能实现：`builtin:swc-loader`、lightningcss、HTML 插件、CSS 抽取、Module Federation
- 2.0：核心包纯 ESM，Node 20.19+ / 22.12+，支持 React Server Components、`import defer`
- 上层工具 Rsbuild 提供开箱即用的配置

迁移步骤：

1. `webpack` → `@rspack/core`，`webpack-cli` → `@rspack/cli`
2. 替换为内置实现：babel-loader → `builtin:swc-loader`，MiniCssExtractPlugin → `rspack.CssExtractRspackPlugin`，TerserPlugin → `rspack.SwcJsMinimizerRspackPlugin`
3. 检查不兼容的插件（依赖 webpack 内部 API 的插件）
4. 对比产物（体积、chunk 结构）和功能回归

**口述模板（30–60 秒）**

1. 一句话结论：Rspack 是 Rust 实现、兼容 webpack 生态的打包器，迁移成本低、提速明显。
2. 关键机制：配置格式和插件钩子与 webpack 一致，内置了 swc、lightningcss 等高性能实现。
3. 对比或注意点：深度依赖 webpack 内部 API 的插件可能不兼容，需要逐一验证。
4. 可追问时补充：迁移后要对比产物结构和体积，并做完整的功能回归。

**常见追问**

- 为什么不直接迁 Vite？→ Vite 的配置和插件体系与 webpack 完全不同，大项目迁移改动大；Rspack 基本是替换依赖和少量配置。
