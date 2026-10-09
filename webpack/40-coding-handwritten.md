# webpack 手写与现场编码

## 使用方式

webpack 的手写题主要有三类：

1. **原理题**：手写迷你打包器、Tapable、运行时，考察是否理解 webpack 的核心机制
2. **扩展题**：手写 Loader、Plugin，考察 API 熟练度和对构建流程的理解
3. **配置与诊断题**：按场景写配置、读代码找问题，考察实战经验

先自己写、边写边讲思路，再对照参考实现。代码基于 webpack 5 和 Node 20+。

---

### H1. 手写一个迷你打包器

- **难度**: senior
- **标签**: 原理, 高频
- **面试官看什么**: 是否理解「解析依赖 → 构建依赖图 → 生成运行时」的核心流程，模块缓存的作用

**题干**

实现一个打包器：从入口文件出发，解析 ESM 的 `import` 依赖，构建依赖图，输出一个可以在浏览器中运行的 bundle。

**思路口述**

1. **解析单个模块**：读取文件 → 用 Babel 解析为 AST → 遍历 `ImportDeclaration` 收集依赖 → 把 ESM 转为 CommonJS（便于用自己实现的 `require` 执行）
2. **构建依赖图**：从入口广度优先遍历，每个模块分配一个 ID，记录「相对路径 → 模块 ID」的映射，已访问的模块不重复解析（处理循环依赖和重复引用）
3. **生成 bundle**：把每个模块包装成函数，放进模块表；实现 `require`：查缓存 → 创建 module → 执行模块函数 → 返回 exports
4. **先缓存再执行**，循环依赖时才不会死循环

**参考实现**

```js
import fs from 'node:fs'
import path from 'node:path'
import { parse } from '@babel/parser'
import traverseModule from '@babel/traverse'
import { transformFromAstSync } from '@babel/core'

// @babel/traverse 是 CommonJS 包，ESM 导入时默认导出在 .default 上
const traverse = traverseModule.default ?? traverseModule

let nextId = 0

function resolveFile(base) {
  for (const candidate of [base, `${base}.js`, path.join(base, 'index.js')]) {
    if (fs.existsSync(candidate) && fs.statSync(candidate).isFile()) return candidate
  }
  throw new Error(`Cannot resolve module: ${base}`)
}

/** 解析单个模块：收集 import 依赖，并把 ESM 转为 CommonJS */
function createModule(filename) {
  const source = fs.readFileSync(filename, 'utf8')
  const ast = parse(source, { sourceType: 'module' })

  const deps = []
  traverse(ast, {
    ImportDeclaration({ node }) {
      deps.push(node.source.value)
    },
  })

  const { code } = transformFromAstSync(ast, source, { presets: ['@babel/preset-env'] })
  return { id: nextId++, filename, deps, code, mapping: {} }
}

/** 从入口广度优先遍历，构建模块依赖图 */
function createGraph(entry) {
  const entryModule = createModule(resolveFile(path.resolve(entry)))
  const graph = [entryModule]
  const byFile = new Map([[entryModule.filename, entryModule]])

  // for...of 遍历过程中 push 的新元素也会被遍历到，相当于队列
  for (const mod of graph) {
    const dir = path.dirname(mod.filename)
    for (const request of mod.deps) {
      const file = resolveFile(path.resolve(dir, request))
      let child = byFile.get(file)
      if (!child) {
        child = createModule(file)
        byFile.set(file, child)
        graph.push(child)
      }
      mod.mapping[request] = child.id
    }
  }
  return graph
}

/** 把依赖图生成为可在浏览器执行的 bundle 代码 */
function bundle(graph) {
  const modules = graph
    .map(m => `${m.id}: [function (require, module, exports) {\n${m.code}\n}, ${JSON.stringify(m.mapping)}]`)
    .join(',\n')

  return `(function (modules) {
  var cache = {}
  function require(id) {
    if (cache[id]) return cache[id].exports
    var fn = modules[id][0]
    var mapping = modules[id][1]
    // 先放入缓存再执行：循环依赖时返回未执行完的 exports，而不是无限递归
    var module = (cache[id] = { exports: {} })
    // 模块内部使用相对路径 require，这里翻译为模块 ID
    fn(function (request) { return require(mapping[request]) }, module, module.exports)
    return module.exports
  }
  require(0)
})({
${modules}
})`
}

fs.mkdirSync('dist', { recursive: true })
fs.writeFileSync('dist/bundle.js', bundle(createGraph('./src/index.js')))
```

**易错点**

- 依赖的相对路径要相对于**当前模块所在目录**解析，而不是入口目录
- 同一个文件被多个模块引用时只能解析一次，否则会生成多个实例
- 模块执行前就要放入缓存
- 和 webpack 的差异：没有 Loader、没有动态 import 和代码分割、没有 Tree Shaking、模块 ID 直接用自增数字

---

### H2. 手写一个 Loader：Markdown 转 HTML 模块

- **难度**: mid
- **标签**: Loader
- **面试官看什么**: Loader 的函数签名、异步处理、配置读取、输出合法的 JS 模块

**题干**

实现 `markdown-loader`：把 `.md` 文件转为 JS 模块，默认导出 HTML 字符串，并把文件头部的 front matter 解析后作为 `meta` 具名导出。支持配置 `gfm`。

**思路口述**

1. Loader 是一个函数，接收源码字符串，返回 JS 代码字符串
2. 异步处理可以直接返回 Promise（webpack 5 支持），也可以用 `this.async()`，二者不要混用
3. 通过 `this.getOptions(schema)` 读取并校验配置
4. 输出的字符串要用 `JSON.stringify` 转义，避免引号和换行破坏代码
5. 不需要声明 `cacheable`，Loader 默认可缓存

**参考实现**

```js
// loaders/markdown-loader.js
const FRONT_MATTER = /^---\r?\n([\s\S]*?)\r?\n---\r?\n?/

const schema = {
  type: 'object',
  properties: { gfm: { type: 'boolean' } },
  additionalProperties: false,
}

function parseFrontMatter(source) {
  const match = FRONT_MATTER.exec(source)
  if (!match) return { meta: {}, body: source }

  const meta = {}
  for (const line of match[1].split(/\r?\n/)) {
    const index = line.indexOf(':')
    if (index > 0) meta[line.slice(0, index).trim()] = line.slice(index + 1).trim()
  }
  return { meta, body: source.slice(match[0].length) }
}

/** 把 Markdown 转为 JS 模块：默认导出 HTML，具名导出 front matter */
module.exports = async function markdownLoader(source) {
  const { gfm = true } = this.getOptions(schema)
  // marked 新版本只提供 ESM，CommonJS 的 Loader 中通过动态 import 加载
  const { marked } = await import('marked')

  const { meta, body } = parseFrontMatter(source)
  const html = await marked.parse(body, { gfm })

  return [
    `export const meta = ${JSON.stringify(meta)};`,
    `export default ${JSON.stringify(html)};`,
  ].join('\n')
}
```

```js
// webpack.config.js
module: {
  rules: [
    {
      test: /\.md$/,
      use: [{ loader: path.resolve(__dirname, 'loaders/markdown-loader.js'), options: { gfm: true } }],
    },
  ],
}

// 使用
import html, { meta } from './posts/hello.md'
```

**易错点**

- 直接拼接 `export default "${html}"`，HTML 中的引号和换行会导致语法错误
- 同时使用 `this.async()` 和返回 Promise
- Loader 中读取了其他文件（如引用的图片、模板）却没有 `this.addDependency`，修改这些文件不会触发重新编译
- 想让 Markdown 中的图片也被 webpack 处理，需要把 `<img src>` 转为 `import` 语句，而不是保留字符串

---

### H3. 用 pitch 手写一个简化版 style-loader

- **难度**: senior
- **标签**: Loader, pitch
- **面试官看什么**: 是否理解 pitch 熔断、剩余请求（remainingRequest）和内联 Loader 前缀

**题干**

实现 `simple-style-loader`，与 css-loader 配合使用（`use: ['simple-style-loader', 'css-loader']`），把 CSS 注入页面，并支持 HMR。

**思路口述**

1. 如果用 normal 方法，拿到的是 css-loader 输出的 **JS 代码字符串**，不是 CSS 内容，无法直接注入
2. 所以在 **pitch** 中返回一段新的 JS 代码，它通过 `import` 引用「剩余的请求」（`css-loader!./a.css`），由 webpack 正常构建这个请求并在运行时拿到 CSS 内容
3. pitch 有返回值就熔断，css-loader 在本轮不会执行；新的 import 请求会触发 css-loader 处理
4. 新请求前面加 `!!`，禁用配置中的所有 Loader，否则会再次匹配到本 Loader，导致无限循环
5. HMR：模块接受自身更新，dispose 时删除旧的 style 标签

**参考实现**

```js
// loaders/simple-style-loader.js
module.exports = function () {}

/** 在 pitch 阶段返回注入样式的代码，通过 !! 内联请求获取 css-loader 的结果 */
module.exports.pitch = function (remainingRequest) {
  // 转为相对路径，避免把构建机的绝对路径写进产物，影响 hash 稳定性
  const request = JSON.stringify(this.utils.contextify(this.context, `!!${remainingRequest}`))

  return `
import content from ${request};

const style = document.createElement('style');
style.textContent = String(content);
document.head.appendChild(style);

if (import.meta.webpackHot) {
  import.meta.webpackHot.accept();
  import.meta.webpackHot.dispose(() => style.remove());
}
`
}
```

执行过程：

```text
第一轮：simple-style-loader.pitch（返回代码，熔断）→ css-loader 不执行
生成的代码中 import "!!css-loader!./a.css"
第二轮：构建这个新请求，只执行 css-loader → 导出 CSS 内容
运行时：注入 <style>
```

**易错点**

- 忘记 `!!` 前缀导致无限递归
- 在 normal 阶段处理：拿到的是 JS 代码，需要执行它才能得到 CSS，复杂且不安全
- css-loader 的默认导出是一个带 `toString()` 的数组对象，用 `String(content)` 获取 CSS 文本
- HMR 时不删除旧标签，样式会越积越多

---

### H4. 手写一个 Plugin：生成产物清单并检查体积预算

- **难度**: mid
- **标签**: Plugin, 高频
- **面试官看什么**: 插件结构、选择正确的钩子和阶段、使用官方 API 输出产物和报告警告

**题干**

实现 `AssetManifestPlugin`：

1. 输出 `asset-manifest.json`，包含每个入口对应的文件列表和每个产物的大小
2. 产物超过预算时输出警告（不让构建失败）

**思路口述**

1. 插件是带 `apply(compiler)` 的类
2. 在 `thisCompilation` 中拿到 compilation，注册 `processAssets`
3. 阶段选 `PROCESS_ASSETS_STAGE_REPORT`：此时压缩和 `realContentHash` 都已完成，文件名和大小是最终结果
4. 用 `compilation.emitAsset` 输出文件，`new WebpackError()` 报告警告
5. webpack 相关的类从 `compiler.webpack` 获取，保证与当前运行的 webpack 版本一致

**参考实现**

```js
const PLUGIN_NAME = 'AssetManifestPlugin'

/** 输出入口与产物的清单文件，并对超出体积预算的产物发出警告 */
class AssetManifestPlugin {
  constructor(options = {}) {
    this.options = { filename: 'asset-manifest.json', budget: 300 * 1024, ...options }
  }

  apply(compiler) {
    const { Compilation, sources, WebpackError } = compiler.webpack
    const { filename, budget } = this.options

    compiler.hooks.thisCompilation.tap(PLUGIN_NAME, compilation => {
      compilation.hooks.processAssets.tap(
        // REPORT 阶段：压缩和真实内容 hash 都已完成，文件名与大小是最终值
        { name: PLUGIN_NAME, stage: Compilation.PROCESS_ASSETS_STAGE_REPORT },
        () => {
          const entrypoints = {}
          for (const [name, entrypoint] of compilation.entrypoints) {
            entrypoints[name] = entrypoint.getFiles().filter(file => !file.endsWith('.map'))
          }

          const files = {}
          const oversized = []
          for (const { name, source } of compilation.getAssets()) {
            if (name.endsWith('.map')) continue
            const size = source.size()
            files[name] = size
            if (size > budget) oversized.push(`${name} (${(size / 1024).toFixed(1)} KiB)`)
          }

          compilation.emitAsset(filename, new sources.RawSource(JSON.stringify({ entrypoints, files }, null, 2)))

          if (oversized.length > 0) {
            compilation.warnings.push(
              new WebpackError(`[${PLUGIN_NAME}] 以下产物超过 ${budget / 1024} KiB 预算：\n${oversized.join('\n')}`),
            )
          }
        },
      )
    })
  }
}

module.exports = AssetManifestPlugin
```

**易错点**

- 在 `emit` 钩子中修改产物：webpack 5 推荐使用 `processAssets`
- 阶段选得太早（如 `ADDITIONS`），拿到的是压缩前的大小和临时文件名
- `require('webpack')` 而不是 `compiler.webpack`：项目中存在多个 webpack 版本时会出现 `instanceof` 判断失败
- 直接 `throw` 错误会中断构建且信息不友好；需要构建失败时 push 到 `compilation.errors`

---

### H5. 手写一个 Plugin：检测重复的依赖包

- **难度**: senior
- **标签**: Plugin, 依赖治理
- **面试官看什么**: 能否从模块对象中拿到包信息，理解模块解析结果中的数据

**题干**

实现 `DuplicatePackagePlugin`：构建完成后，如果同一个 npm 包有多个版本被打包，输出警告并列出版本和所在路径。

**思路口述**

1. 在 `finishModules` 钩子中遍历所有模块
2. 普通模块（NormalModule）的 `resourceResolveData` 中有 `descriptionFileData`（所属 package.json 的内容）和 `descriptionFileRoot`（包的根目录）
3. 只统计位于 `node_modules` 中的包，按「包名 → 版本 → 路径集合」聚合
4. 同一个包有多个版本时报告警告

**参考实现**

```js
const PLUGIN_NAME = 'DuplicatePackagePlugin'

/** 检测同一个 npm 包被打包了多个版本的情况 */
class DuplicatePackagePlugin {
  apply(compiler) {
    const { WebpackError } = compiler.webpack

    compiler.hooks.thisCompilation.tap(PLUGIN_NAME, compilation => {
      compilation.hooks.finishModules.tap(PLUGIN_NAME, modules => {
        /** @type {Map<string, Map<string, Set<string>>>} 包名 → 版本 → 包所在目录 */
        const packages = new Map()

        for (const module of modules) {
          const data = module.resourceResolveData
          const pkg = data?.descriptionFileData
          const root = data?.descriptionFileRoot
          if (!pkg?.name || !pkg.version || !root?.includes('node_modules')) continue

          if (!packages.has(pkg.name)) packages.set(pkg.name, new Map())
          const versions = packages.get(pkg.name)
          if (!versions.has(pkg.version)) versions.set(pkg.version, new Set())
          versions.get(pkg.version).add(root)
        }

        for (const [name, versions] of packages) {
          if (versions.size < 2) continue
          const detail = [...versions]
            .map(([version, roots]) => `  ${version}: ${[...roots].join(', ')}`)
            .join('\n')
          compilation.warnings.push(new WebpackError(`[${PLUGIN_NAME}] ${name} 存在 ${versions.size} 个版本：\n${detail}`))
        }
      })
    })
  }
}

module.exports = DuplicatePackagePlugin
```

**易错点**

- 用模块路径中的 `node_modules/xxx` 字符串推断包名，无法正确处理 scoped 包（`@scope/name`）和 pnpm 的目录结构；使用 package.json 数据更可靠
- `finishModules` 时还没有做 Tree Shaking，统计的是「被构建的」模块，可能包含最终被删除的模块
- 从持久化缓存恢复的模块，`resourceResolveData` 可能不完整，必要时在无缓存构建中运行检查
- 同一版本出现在多个路径（pnpm 的不同 peer 依赖组合）也会导致重复打包，可以一并报告

---

### H6. 手写简化版 Tapable

- **难度**: senior
- **标签**: Tapable, 原理
- **面试官看什么**: 是否理解各种钩子的语义、三种注册方式、执行顺序控制

**题干**

实现 `SyncHook`、`SyncBailHook`、`SyncWaterfallHook`、`AsyncSeriesHook`、`AsyncParallelHook`，支持 `tap` / `tapAsync` / `tapPromise` 和 `stage` 排序。

**思路口述**

1. 基类保存回调列表，注册时按 `stage` 插入到正确位置（stage 越小越靠前，相同时按注册顺序）
2. 同步钩子：Sync 依次调用；Bail 遇到非 undefined 返回值就停止；Waterfall 把返回值传给下一个
3. 异步钩子：把三种注册方式统一转换为返回 Promise 的函数，Series 依次 await，Parallel 用 `Promise.all`
4. 真实的 Tapable 会根据回调动态生成代码（`new Function`），这里用循环实现语义

**参考实现**

```ts
type TapType = 'sync' | 'async' | 'promise'
type TapOptions = string | { name: string; stage?: number }
interface Tap {
  name: string
  stage: number
  type: TapType
  fn: (...args: any[]) => any
}

/** 钩子基类：管理回调的注册与排序 */
abstract class Hook {
  protected taps: Tap[] = []

  protected insert(options: TapOptions, type: TapType, fn: Tap['fn']): void {
    const { name, stage = 0 } = typeof options === 'string' ? { name: options } : options
    const tap: Tap = { name, stage, type, fn }
    // 插到第一个 stage 更大的回调之前：stage 相同时保持注册顺序
    const index = this.taps.findIndex(t => t.stage > stage)
    if (index === -1) this.taps.push(tap)
    else this.taps.splice(index, 0, tap)
  }

  tap(options: TapOptions, fn: Tap['fn']): void {
    this.insert(options, 'sync', fn)
  }
}

/** 同步依次执行所有回调 */
export class SyncHook extends Hook {
  call(...args: unknown[]): void {
    for (const tap of this.taps) tap.fn(...args)
  }
}

/** 同步执行，任一回调返回非 undefined 时停止并返回该值 */
export class SyncBailHook extends Hook {
  call(...args: unknown[]): unknown {
    for (const tap of this.taps) {
      const result = tap.fn(...args)
      if (result !== undefined) return result
    }
    return undefined
  }
}

/** 同步执行，上一个回调的返回值作为下一个回调的第一个参数 */
export class SyncWaterfallHook extends Hook {
  call(value: unknown, ...rest: unknown[]): unknown {
    for (const tap of this.taps) {
      const result = tap.fn(value, ...rest)
      if (result !== undefined) value = result
    }
    return value
  }
}

/** 把三种注册方式统一为返回 Promise 的调用 */
function invoke(tap: Tap, args: unknown[]): Promise<unknown> {
  switch (tap.type) {
    case 'sync':
      return new Promise(resolve => resolve(tap.fn(...args)))   // 同步抛错会变成 reject
    case 'async':
      return new Promise((resolve, reject) => {
        tap.fn(...args, (err?: unknown, result?: unknown) => (err ? reject(err) : resolve(result)))
      })
    case 'promise':
      return Promise.resolve(tap.fn(...args))
  }
}

/** 异步钩子基类：额外支持 tapAsync / tapPromise 注册和 promise / callAsync 触发 */
abstract class AsyncHook extends Hook {
  tapAsync(options: TapOptions, fn: Tap['fn']): void {
    this.insert(options, 'async', fn)
  }

  tapPromise(options: TapOptions, fn: Tap['fn']): void {
    this.insert(options, 'promise', fn)
  }

  abstract promise(...args: unknown[]): Promise<void>

  callAsync(...argsAndCallback: unknown[]): void {
    const callback = argsAndCallback.pop() as (err?: unknown) => void
    this.promise(...argsAndCallback).then(() => callback(), callback)
  }
}

/** 异步串行：前一个完成后才执行下一个 */
export class AsyncSeriesHook extends AsyncHook {
  async promise(...args: unknown[]): Promise<void> {
    for (const tap of this.taps) await invoke(tap, args)
  }
}

/** 异步并行：全部完成后结束，任一失败则整体失败 */
export class AsyncParallelHook extends AsyncHook {
  async promise(...args: unknown[]): Promise<void> {
    await Promise.all(this.taps.map(tap => invoke(tap, args)))
  }
}

// 使用
const hook = new AsyncSeriesHook()
hook.tapPromise({ name: 'B', stage: 10 }, async () => console.log('B'))
hook.tapAsync('A', (cb: () => void) => setTimeout(() => (console.log('A'), cb()), 10))
await hook.promise()   // A, B
```

**易错点**

- stage 相同时打乱了注册顺序
- Waterfall 中回调返回 undefined 时应保留上一个值
- 回调风格的 `callAsync` 中，回调在成功时也要被调用
- 同步钩子不能使用 tapAsync / tapPromise 注册（真实 Tapable 会报错）

---

### H7. 手写 webpack 运行时：模块加载与 JSONP 分块加载

- **难度**: senior
- **标签**: 运行时, 代码分割
- **面试官看什么**: 是否理解 chunk 的三种状态、并发请求去重、全局数组 push 劫持、失败重置

**题干**

实现简化的 webpack 运行时：`__webpack_require__`、`__webpack_require__.e(chunkId)` 动态加载 chunk，chunk 文件通过全局数组 `push` 注册模块。

**思路口述**

1. chunk 状态：`0` 已加载；`[resolve, reject, promise]` 加载中；`undefined` 未加载
2. `ensureChunk`：已加载直接 resolve；加载中返回同一个 Promise（去重）；未加载就创建 Promise 并插入 script
3. chunk 文件执行 `self.webpackChunkdemo.push([[chunkIds], modules])`；运行时把这个数组的 `push` 替换为自己的回调：合并模块、标记已加载、resolve
4. 运行时启动时，先处理在它之前已经 push 进数组的 chunk（chunk 文件可能比运行时先执行）
5. 加载失败时把状态重置为 `undefined`，下次可以重试

**参考实现**

```js
// runtime.js（产物中的运行时部分）
;(function () {
  var modules = {
    './src/index.js': function (module, exports, require) {
      document.querySelector('#about').onclick = function () {
        require.e('about').then(function () {
          require('./src/about.js').render()
        })
      }
    },
  }
  var cache = {}

  function require(moduleId) {
    if (cache[moduleId]) return cache[moduleId].exports
    var module = (cache[moduleId] = { exports: {} })
    modules[moduleId](module, module.exports, require)
    return module.exports
  }

  // chunk 状态：0 已加载；[resolve, reject, promise] 加载中；undefined 未加载
  var installedChunks = { main: 0 }

  require.p = '/static/'
  require.u = function (chunkId) {
    return chunkId + '.js'
  }

  require.e = function ensureChunk(chunkId) {
    var data = installedChunks[chunkId]
    if (data === 0) return Promise.resolve()
    if (data) return data[2]                  // 正在加载：复用同一个 Promise，避免重复请求

    var promise = new Promise(function (resolve, reject) {
      data = installedChunks[chunkId] = [resolve, reject]
    })
    data[2] = promise

    var script = document.createElement('script')
    script.src = require.p + require.u(chunkId)
    script.onerror = function () {
      var pending = installedChunks[chunkId]
      // 重置为未加载，使下一次 import() 能够重新请求
      installedChunks[chunkId] = undefined
      var error = new Error('Loading chunk ' + chunkId + ' failed.')
      error.name = 'ChunkLoadError'
      if (pending) pending[1](error)
    }
    document.head.appendChild(script)
    return promise
  }

  function jsonpCallback(payload) {
    var chunkIds = payload[0]
    var moreModules = payload[1]
    for (var id in moreModules) modules[id] = moreModules[id]
    chunkIds.forEach(function (chunkId) {
      var pending = installedChunks[chunkId]
      installedChunks[chunkId] = 0
      if (pending) pending[0]()
    })
  }

  var chunkQueue = (self.webpackChunkdemo = self.webpackChunkdemo || [])
  // chunk 文件可能先于运行时执行，先处理已经入队的 chunk
  chunkQueue.forEach(jsonpCallback)
  chunkQueue.push = jsonpCallback

  require('./src/index.js')
})()
```

```js
// about.js（异步 chunk）
;(self.webpackChunkdemo = self.webpackChunkdemo || []).push([
  ['about'],
  {
    './src/about.js': function (module, exports) {
      exports.render = function () {
        document.body.append('About page')
      }
    },
  },
])
```

**易错点**

- 没有处理「加载中」状态，快速点击会发出多个相同请求
- 失败后没有重置状态，之后再也无法加载
- 没有处理运行时之前就已经入队的 chunk
- 真实 webpack 还会在 `onload` 中检查：脚本加载成功但没有注册 chunk（如返回了错误页面）也要 reject；并设置超时（`output.chunkLoadTimeout`，默认 120 秒）

---

### H8. 处理 ChunkLoadError：重试、版本检测与防无限刷新

- **难度**: mid
- **标签**: 线上问题, 发布
- **面试官看什么**: 能否区分网络抖动和版本更新两类原因，兜底方案是否会造成无限刷新

**题干**

实现 `importWithRetry(factory)`：动态导入失败时按指数退避重试；重试仍失败时，如果检测到服务端已发布新版本，则刷新页面（要防止无限刷新）；否则把错误抛给 ErrorBoundary。

**思路口述**

1. 只处理 chunk 加载错误（`ChunkLoadError`、CSS chunk 加载失败），业务错误直接抛出
2. webpack 加载失败后会重置 chunk 状态，所以再次调用 `import()` 会重新请求
3. 重试失败后请求 `version.json`（禁用缓存），与构建时注入的版本号比较
4. 版本不同说明旧 chunk 可能已被删除，刷新页面加载新版本；用 sessionStorage 记录刷新时间，短时间内不重复刷新

**参考实现**

```ts
declare const __APP_VERSION__: string   // 构建时由 DefinePlugin 注入

const RELOAD_KEY = 'chunk-error-reload-at'
const RELOAD_COOLDOWN_MS = 10_000

const sleep = (ms: number) => new Promise(resolve => setTimeout(resolve, ms))

function isChunkLoadError(err: unknown): boolean {
  return (
    err instanceof Error &&
    (err.name === 'ChunkLoadError' || /Loading (CSS )?chunk .+ failed/.test(err.message))
  )
}

async function hasNewVersion(): Promise<boolean> {
  try {
    const res = await fetch(`/version.json?t=${Date.now()}`, { cache: 'no-store' })
    const { version } = (await res.json()) as { version: string }
    return version !== __APP_VERSION__
  } catch {
    return false
  }
}

function reloadOnce(): boolean {
  const last = Number(sessionStorage.getItem(RELOAD_KEY) ?? 0)
  // 冷却期内已经刷新过，说明刷新解决不了问题，避免进入无限刷新
  if (Date.now() - last < RELOAD_COOLDOWN_MS) return false
  sessionStorage.setItem(RELOAD_KEY, String(Date.now()))
  location.reload()
  return true
}

/** 带重试的动态导入：网络抖动时重试，检测到新版本时刷新页面 */
export async function importWithRetry<T>(
  factory: () => Promise<T>,
  { retries = 2, baseDelayMs = 500 } = {},
): Promise<T> {
  for (let attempt = 0; ; attempt++) {
    try {
      return await factory()
    } catch (err) {
      if (!isChunkLoadError(err)) throw err
      if (attempt < retries) {
        await sleep(baseDelayMs * 2 ** attempt)
        continue
      }
      if ((await hasNewVersion()) && reloadOnce()) {
        // 页面即将刷新：保持 pending，避免刷新前闪现错误界面
        return new Promise<never>(() => {})
      }
      throw err
    }
  }
}

// 使用
const Settings = lazy(() => importWithRetry(() => import('./pages/Settings')))
```

```js
// webpack.config.js：注入版本号，并输出 version.json
const version = process.env.GIT_COMMIT ?? String(Date.now())

plugins: [
  new webpack.DefinePlugin({ __APP_VERSION__: JSON.stringify(version) }),
  {
    apply(compiler) {
      const { Compilation, sources } = compiler.webpack
      compiler.hooks.thisCompilation.tap('VersionFile', compilation => {
        compilation.hooks.processAssets.tap(
          { name: 'VersionFile', stage: Compilation.PROCESS_ASSETS_STAGE_ADDITIONAL },
          () => compilation.emitAsset('version.json', new sources.RawSource(JSON.stringify({ version }))),
        )
      })
    },
  },
]
```

**易错点**

- 所有错误都重试，导致业务异常被吞掉或延迟
- 刷新没有冷却控制，资源确实丢失时页面无限刷新
- `version.json` 被 CDN 或浏览器缓存，检测不到新版本：需要 `no-store` 并在服务端配置不缓存
- 前端兜底只是补救，根本方案是发布时**保留旧版本静态资源**，并且先发布资源、后发布 HTML

---

### H9. 配置题：按场景设计 splitChunks

- **难度**: mid
- **标签**: 代码分割, 配置
- **面试官看什么**: 是否理解 cacheGroups、priority、chunks、name 的含义及其取舍

**题干**

一个 React 中后台 SPA：

- 30 个路由页面，全部懒加载
- 布局和大部分页面使用 antd
- echarts 只在 2 个报表页面使用
- `src/components` 中的通用组件被很多页面使用
- 要求：框架代码长期缓存；echarts 不能进入首屏；入口并行请求不超过 6 个；单个第三方 chunk 尽量不超过 300KB

**思路口述**

1. `runtimeChunk: 'single'` + `chunks: 'all'`，让初始 chunk 中的依赖也参与拆分
2. **framework**：React 全家桶几乎不变，固定名称、最高优先级、`enforce` 忽略大小限制
3. **antd**：布局就使用了 antd，放在首屏可以接受；固定名称方便缓存
4. **echarts**：只处理异步 chunk（`chunks: 'async'`），两个报表页共享同一个异步 chunk，不进入首屏
5. **vendors**：其余第三方库**不设置固定名称**，让 webpack 按使用它们的 chunk 组合自动拆分，避免一个巨大的 vendors 被所有页面加载；`maxSize` 控制单个大小
6. **common**：被至少 3 个 chunk 引用的业务组件

**参考实现**

```js
optimization: {
  runtimeChunk: 'single',
  moduleIds: 'deterministic',
  splitChunks: {
    chunks: 'all',
    maxInitialRequests: 6,
    cacheGroups: {
      framework: {
        test: /[\\/]node_modules[\\/](react|react-dom|scheduler|react-router|react-router-dom)[\\/]/,
        name: 'framework',
        priority: 40,
        enforce: true,
      },
      antd: {
        test: /[\\/]node_modules[\\/](antd|@ant-design|rc-[^\\/]+)[\\/]/,
        name: 'antd',
        priority: 30,
      },
      echarts: {
        test: /[\\/]node_modules[\\/](echarts|zrender)[\\/]/,
        name: 'echarts',
        chunks: 'async',
        priority: 30,
      },
      vendors: {
        test: /[\\/]node_modules[\\/]/,
        priority: 20,
        maxSize: 300 * 1024,
        reuseExistingChunk: true,
      },
      common: {
        test: /[\\/]src[\\/]components[\\/]/,
        minChunks: 3,
        priority: 10,
        reuseExistingChunk: true,
      },
      default: false,
    },
  },
},
```

**易错点**

- 给 `vendors` 设置固定 `name`：所有第三方库合并为一个 chunk，异步页面独有的依赖也被拉进首屏
- 优先级设置错误：echarts 同时匹配 echarts 和 vendors，必须让 echarts 的 priority 更高
- 忘记 `chunks: 'all'`：默认只拆分异步 chunk，入口中的 React 不会被拆出来
- 拆得太细：大量小 chunk 增加请求数和运行时开销；拆分后要用 bundle-analyzer 验证

---

### H10. 从零写一个生产级 React + TypeScript 的 webpack 配置

- **难度**: mid
- **标签**: 配置, 高频
- **面试官看什么**: 配置是否完整、开发和生产的差异是否合理、每一项是否能说出原因

**题干**

写出 React + TypeScript 项目的 webpack 配置，要求：swc 转译、类型检查不阻塞构建、CSS Modules、开发环境 HMR + Fast Refresh、生产环境抽取 CSS、长效缓存、持久化缓存、source map 合理。

**思路口述**

1. 用函数导出配置，根据 mode 区分环境
2. 转译用 swc-loader，类型检查交给 ForkTsChecker（独立进程）
3. CSS：开发用 style-loader（支持 HMR），生产用 MiniCssExtractPlugin
4. 长效缓存：contenthash + runtimeChunk + deterministic ID
5. 持久化缓存：配置文件作为 buildDependencies
6. source map：开发 `eval-cheap-module-source-map`，生产 `hidden-source-map`

**参考实现**

```js
// webpack.config.js
const path = require('node:path')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin')
const TerserPlugin = require('terser-webpack-plugin')
const ForkTsCheckerWebpackPlugin = require('fork-ts-checker-webpack-plugin')
const ReactRefreshPlugin = require('@pmmmwh/react-refresh-webpack-plugin')

module.exports = (_env, argv) => {
  const isProd = argv.mode === 'production'

  const cssLoaders = modules => [
    isProd ? MiniCssExtractPlugin.loader : 'style-loader',
    {
      loader: 'css-loader',
      options: {
        importLoaders: 1,
        modules: modules && { localIdentName: isProd ? '[hash:base64:6]' : '[name]__[local]' },
      },
    },
    'postcss-loader',
  ]

  return {
    mode: isProd ? 'production' : 'development',
    entry: './src/main.tsx',
    output: {
      path: path.resolve(__dirname, 'dist'),
      filename: isProd ? 'js/[name].[contenthash:8].js' : 'js/[name].js',
      chunkFilename: isProd ? 'js/[name].[contenthash:8].js' : 'js/[name].js',
      assetModuleFilename: 'assets/[name].[contenthash:8][ext]',
      publicPath: '/',
      clean: true,
    },
    resolve: {
      extensions: ['.tsx', '.ts', '.js'],
      alias: { '@': path.resolve(__dirname, 'src') },
    },
    module: {
      rules: [
        {
          test: /\.[jt]sx?$/,
          include: path.resolve(__dirname, 'src'),
          use: {
            loader: 'swc-loader',
            options: {
              jsc: {
                parser: { syntax: 'typescript', tsx: true },
                transform: { react: { runtime: 'automatic', development: !isProd, refresh: !isProd } },
              },
              env: { targets: 'defaults' },
            },
          },
        },
        { test: /\.module\.css$/, use: cssLoaders(true) },
        { test: /\.css$/, exclude: /\.module\.css$/, use: cssLoaders(false) },
        {
          test: /\.(png|jpe?g|gif|webp|svg)$/i,
          type: 'asset',
          parser: { dataUrlCondition: { maxSize: 4 * 1024 } },
        },
        { test: /\.(woff2?|ttf)$/i, type: 'asset/resource' },
      ],
    },
    plugins: [
      new HtmlWebpackPlugin({ template: './public/index.html', scriptLoading: 'defer' }),
      new ForkTsCheckerWebpackPlugin({ async: !isProd }),   // 开发时不阻塞 HMR，生产时类型错误让构建失败
      isProd && new MiniCssExtractPlugin({
        filename: 'css/[name].[contenthash:8].css',
        chunkFilename: 'css/[name].[contenthash:8].css',
      }),
      !isProd && new ReactRefreshPlugin(),
    ].filter(Boolean),
    optimization: {
      runtimeChunk: 'single',
      moduleIds: 'deterministic',
      splitChunks: { chunks: 'all' },
      minimizer: [
        new TerserPlugin({ minify: TerserPlugin.swcMinify }),
        new CssMinimizerPlugin(),
      ],
    },
    cache: {
      type: 'filesystem',
      buildDependencies: { config: [__filename] },
    },
    devtool: isProd ? 'hidden-source-map' : 'eval-cheap-module-source-map',
    devServer: {
      port: 3000,
      hot: true,
      historyApiFallback: true,
      proxy: [{ context: ['/api'], target: 'http://localhost:8080', changeOrigin: true }],
    },
    performance: { hints: isProd ? 'warning' : false },
  }
}
```

**易错点**

- swc 的 `refresh` 在生产环境开启，产物中包含 Fast Refresh 的注册代码
- `include` 写成了 `exclude: /node_modules/`，在 Monorepo 中会漏掉 workspace 包或者误处理其他目录
- 没有 `buildDependencies`，修改配置后持久化缓存返回旧结果
- 生产环境用 `source-map` 并直接部署 `.map` 文件，导致源码泄露
- `postcss-loader` 需要配套的 `postcss.config.js`

---

### H11. 读代码找问题：为什么 Tree Shaking 没有生效？

- **难度**: mid
- **标签**: Tree Shaking, 诊断, 高频
- **面试官看什么**: 能否逐一识别失效原因并给出修复方式

**题干**

项目只使用了 `formatMoney`，但产物中包含了整个 lodash、echarts 和所有工具函数。找出问题。

```js
// babel.config.js
module.exports = { presets: [['@babel/preset-env', { modules: 'commonjs' }]] }

// src/utils/package.json 不存在；根目录 package.json 没有 sideEffects 字段

// src/utils/index.js
export * from './format'
export * from './chart'

// src/utils/chart.js
import * as echarts from 'echarts'
echarts.registerTheme('dark', darkTheme)              // 顶层副作用
export const renderChart = (el, option) => echarts.init(el).setOption(option)

// src/utils/format.js
import _ from 'lodash'
export const formatMoney = n => _.round(n, 2).toFixed(2)
export const PALETTE = createPalette()                 // 顶层函数调用
export const formatters = { date: formatDate, money: formatMoney }

// src/pages/order.js
import * as utils from '../utils'
const format = utils[`format${type}`]
```

**参考答案**

| 问题 | 原因 | 修复 |
|------|------|------|
| `modules: 'commonjs'` | Babel 把 ESM 转为 CommonJS，webpack 无法静态分析 | 删除该选项，使用默认值（babel-loader 下保留 ESM） |
| 没有声明 `sideEffects` | webpack 不能跳过 chart.js，echarts 被完整引入 | 在 package.json 中声明 `"sideEffects": ["*.css"]` |
| chart.js 有顶层副作用 | `registerTheme` 在模块加载时执行，即使声明了无副作用也属于错误声明 | 把注册放进函数中，在使用图表时调用 |
| `import _ from 'lodash'` | lodash 是 CommonJS，整个包都被引入 | 使用 `lodash-es` 并具名导入：`import { round } from 'lodash-es'` |
| `PALETTE = createPalette()` | 压缩器无法判断调用是否有副作用，不敢删除 | `/*#__PURE__*/ createPalette()`，或改为懒计算 |
| `formatters` 对象 | 对象引用了所有格式化函数，只要对象被保留，函数就都被保留 | 不导出聚合对象，或者确认没有使用 |
| `utils[\`format${type}\`]` | 命名空间被动态访问，webpack 只能认为所有导出都被使用 | 改为显式映射：`const map = { Money: formatMoney }` |

**易错点**

- 只修改了 Babel 配置，没有处理副作用和动态访问，仍然不生效
- 为了 Tree Shaking 把有副作用的模块也声明为无副作用，导致线上功能缺失（主题没有注册）
- 验证方式：开发模式 + `optimization.usedExports: true`，查看产物中的 `unused harmony export` 注释，或者对比 bundle-analyzer

---

### H12. Module Federation 配置：host、remote、动态 remote 与降级

- **难度**: senior
- **标签**: Module Federation, 微前端
- **面试官看什么**: 共享依赖配置、异步边界、动态 remote 和加载失败的处理

**题干**

1. `shop` 应用暴露 `Cart` 组件
2. `main` 应用加载 `Cart`，React 作为单例共享
3. remote 地址由配置中心在运行时下发（支持灰度和回滚）
4. remote 加载失败时显示降级 UI，不影响主应用

**思路口述**

1. remote 通过 `exposes` 暴露模块，生成 `remoteEntry.js`
2. 双方 `shared` 中 React 设置 `singleton` 和 `requiredVersion`
3. 入口使用 `import('./bootstrap')` 建立异步边界
4. 动态 remote：构建时不写死地址，运行时通过 MF 2.0 的运行时 API 注册
5. 用 ErrorBoundary + Suspense 处理加载失败和加载中

**参考实现**

```js
// shop/webpack.config.js（remote）
const { ModuleFederationPlugin } = require('@module-federation/enhanced/webpack')

module.exports = {
  output: { publicPath: 'auto', uniqueName: 'shop' },
  plugins: [
    new ModuleFederationPlugin({
      name: 'shop',
      filename: 'remoteEntry.js',
      exposes: { './Cart': './src/Cart.tsx' },
      shared: {
        react: { singleton: true, requiredVersion: '^19.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^19.0.0' },
      },
    }),
  ],
}
```

```js
// main/webpack.config.js（host）：不配置 remotes，运行时动态注册
new ModuleFederationPlugin({
  name: 'main',
  shared: {
    react: { singleton: true, requiredVersion: '^19.0.0' },
    'react-dom': { singleton: true, requiredVersion: '^19.0.0' },
  },
})
```

```ts
// main/src/index.ts
// 异步边界：共享依赖需要先完成版本协商，才能被同步使用
import('./bootstrap')
```

```tsx
// main/src/remotes.ts
import { registerRemotes, loadRemote } from '@module-federation/enhanced/runtime'

/** 从配置中心获取 remote 地址并注册，支持按版本灰度和回滚 */
export async function setupRemotes(): Promise<void> {
  const config = (await fetch('/api/remotes').then(r => r.json())) as Record<string, string>
  registerRemotes(Object.entries(config).map(([name, entry]) => ({ name, entry })))
}

/** 加载远程组件；失败时抛出，由 ErrorBoundary 显示降级 UI */
export function lazyRemote<T extends React.ComponentType<any>>(id: string) {
  return React.lazy(async () => {
    const mod = await loadRemote<{ default: T }>(id)
    if (!mod) throw new Error(`Remote module ${id} is not available`)
    return mod
  })
}
```

```tsx
// main/src/bootstrap.tsx
const Cart = lazyRemote('shop/Cart')

await setupRemotes()

createRoot(document.getElementById('root')!).render(
  <ErrorBoundary fallback={<p>购物车暂时不可用</p>}>
    <Suspense fallback={<Spinner />}>
      <Cart />
    </Suspense>
  </ErrorBoundary>,
)
```

运行时 API 的具体导出（如 `registerRemotes`、`loadRemote`、`createInstance`）随 MF 2.0 版本演进，以官方文档为准。

**易错点**

- 没有异步边界，报错 `Shared module is not available for eager consumption`
- React 没有设置 singleton，页面出现两份 React，Hooks 报错
- remote 的 `publicPath` 写死为相对路径，被 host 加载时资源地址错误 → 使用 `'auto'`
- 多个应用的 `uniqueName` 相同，JSONP 全局变量冲突
- remote 加载失败没有降级处理，整个页面白屏
- remote 暴露的模块就是公共 API，做不兼容修改前需要版本化发布
