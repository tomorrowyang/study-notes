# Vite 中级 Q&A

> 目标：理解开发服务器请求处理流程、依赖预构建、HMR、插件机制、生产构建、开发与生产差异、SSR 与 Environment API 基础、Vite 8 迁移。能回答「为什么」和「怎么实现」。

---

## 一、开发服务器原理

### Q1. Vite 开发服务器由哪些部分组成？

- **难度**: 中级
- **标签**: 原理, 架构

**答案**

`createServer` 主要创建以下部分：

| 组成 | 作用 |
|------|------|
| **resolvedConfig** | 合并用户配置、插件 `config` 钩子、默认值 |
| **HTTP 服务器 + connect 中间件** | 处理请求：代理、CORS、静态文件、HTML 转换、模块转换 |
| **PluginContainer** | 在开发时模拟 Rollup 插件上下文，按顺序调用插件钩子 |
| **ModuleGraph** | 记录模块之间的导入关系、转换结果缓存、HMR 信息 |
| **Watcher（chokidar）** | 监听文件变化，触发 HMR |
| **WebSocket 服务器** | 向浏览器推送 HMR 消息和错误 |
| **Optimizer** | 依赖预构建与运行时发现新依赖 |

Vite 6 起，模块图和插件管线按 **Environment**（client / ssr 等）划分，见 Q25。

**口述模板（30–60 秒）**

1. 一句话结论：开发服务器由中间件、插件容器、模块图、文件监听、WebSocket 和依赖优化器组成。
2. 关键机制：请求进入中间件，经过插件容器转换，结果记录到模块图；文件变化时由 watcher 和模块图计算 HMR 并通过 WebSocket 推送。
3. 对比或注意点：插件容器让 Rollup 插件在没有打包的开发环境中也能运行。
4. 可追问时补充：Vite 6 之后这些结构按环境划分。

**常见追问**

- `configureServer` 注册的中间件在什么位置？→ 默认在内置中间件之前；返回一个函数的话，函数中注册的中间件在内置中间件之后（HTML 处理之前）。

---

### Q2. 浏览器请求 `/src/main.tsx` 时，Vite 做了什么？

- **难度**: 中级
- **标签**: 原理, 请求流程

**答案**

```text
GET /src/main.tsx
  → transformMiddleware
  → 检查模块图缓存（有缓存且 etag 匹配 → 304）
  → pluginContainer.resolveId('/src/main.tsx')  得到文件绝对路径
  → pluginContainer.load(id)                    读取文件（或虚拟模块）
  → pluginContainer.transform(code, id)         Oxc 转换 TSX、框架插件处理
  → importAnalysis 插件：
       - 用 es-module-lexer 找出所有 import
       - 裸模块重写：'react' → '/node_modules/.vite/deps/react.js?v=abc123'
       - 相对路径规范化，添加 ?t= 时间戳（HMR 后）
       - 更新模块图中的导入关系
       - 注入 import.meta.hot 相关代码
  → 返回 Content-Type: application/javascript
```

缓存策略：

- 源码模块：协商缓存（etag / 304）
- 预构建依赖：强缓存 `max-age=31536000,immutable`，URL 带 `?v=hash`，依赖更新后 hash 变化

**口述模板（30–60 秒）**

1. 一句话结论：请求经过 resolveId、load、transform 和导入分析后返回。
2. 关键机制：导入分析把裸模块重写为预构建路径，同时更新模块图。
3. 对比或注意点：源码用协商缓存，依赖用强缓存加版本号。
4. 可追问时补充：转换结果缓存在模块图中，文件变化时失效。

**常见追问**

- 为什么不对源码使用强缓存？→ 源码经常变化，需要确保拿到最新版本。

---

### Q3. 依赖预构建的完整流程是什么？

- **难度**: 中级
- **标签**: 预构建

**答案**

**1. 扫描**

- 从入口（默认是项目中的 HTML，可通过 `optimizeDeps.entries` 指定）开始，快速扫描源码中的裸模块导入
- 加上 `optimizeDeps.include` 中的依赖，去掉 `exclude` 中的依赖

**2. 打包**

- 用 Rolldown（Vite 8，之前用 esbuild）把每个依赖打包成 ESM，输出到 `node_modules/.vite/deps`
- 多个依赖之间的公共代码抽取为共享 chunk，保证单例（例如 React 只有一份）
- 生成 `_metadata.json`，记录依赖映射和 hash

**3. 缓存**

- hash 由 lockfile、相关配置、`patches` 目录等计算
- 下次启动时 hash 未变则直接复用

**4. 运行时发现新依赖**

- 浏览器加载过程中发现扫描时漏掉的依赖，会重新预构建
- 默认（`holdUntilCrawlEnd: true`）等首屏模块爬取完再统一处理，减少整页刷新

**口述模板（30–60 秒）**

1. 一句话结论：预构建分为扫描、打包、缓存和运行时发现四个阶段。
2. 关键机制：扫描入口找到依赖，打包成 ESM 并抽取公共 chunk，按 lockfile 和配置计算 hash 缓存。
3. 对比或注意点：运行时发现新依赖会重新预构建，可能导致页面刷新。
4. 可追问时补充：通过 `optimizeDeps.include` 提前声明可以避免运行时发现。

**常见追问**

- 为什么要抽取共享 chunk？→ 如果 `react-dom` 和 `react` 各自包含一份 React，就会出现多实例问题。

---

### Q4. 为什么要重写裸模块导入？

- **难度**: 中级
- **标签**: 原理

**答案**

浏览器的原生 ESM 只支持相对路径、绝对路径和完整 URL，**不认识** `import React from 'react'` 这种裸模块说明符（除非使用 import map）。

Vite 的 `importAnalysis` 插件会：

1. 用 `es-module-lexer` 快速找出所有导入语句的位置（不做完整 AST 解析，速度快）
2. 对每个说明符调用 `resolveId`
3. 依赖重写为预构建路径：`/node_modules/.vite/deps/react.js?v=abc123`
4. 未预构建的依赖（如 linked 包）重写为 `/@fs/...` 或 `/node_modules/...`
5. 用 `magic-string` 替换字符串，保持 source map

**口述模板（30–60 秒）**

1. 一句话结论：浏览器不认识裸模块，所以 Vite 要把它们重写为真实 URL。
2. 关键机制：用 es-module-lexer 找到导入位置，resolve 后用 magic-string 替换。
3. 对比或注意点：es-module-lexer 只做词法扫描，比完整 AST 解析快很多。
4. 可追问时补充：`/@fs/` 前缀用于访问项目根目录之外的文件，受 `server.fs.allow` 限制。

**常见追问**

- 为什么不直接用 import map？→ 依赖还需要预构建（CJS 转换），import map 不能解决这个问题。

---

### Q5. Vite HMR 的完整流程是什么？

- **难度**: 中级
- **标签**: HMR

**答案**

```text
1. chokidar 监听到 src/Button.tsx 变化
2. 模块图中找到对应模块，使其转换缓存失效
3. 插件的 hotUpdate（旧版为 handleHotUpdate）钩子可以过滤或修改受影响模块
4. 从变化的模块向上查找 HMR 边界：
   - 模块自身 accept（self-accepting）→ 它就是边界
   - 导入方 accept 了这个依赖 → 导入方是边界
   - 一直找到入口都没有边界 → 整页刷新
5. 通过 WebSocket 发送：
   { type: 'update', updates: [{ type: 'js-update', path, acceptedPath, timestamp }] }
6. 浏览器中 /@vite/client：
   - 执行旧模块的 dispose 回调
   - import(`${acceptedPath}?t=${timestamp}`) 请求新模块
   - 调用 accept 回调
```

时间戳参数用来绕过浏览器的 ESM 缓存（同一个 URL 的 ESM 只会执行一次）。

**口述模板（30–60 秒）**

1. 一句话结论：文件变化后，Vite 在模块图中查找 HMR 边界，再通过 WebSocket 通知浏览器重新导入。
2. 关键机制：向上找 accept 的模块作为边界，找不到就整页刷新；浏览器带时间戳重新 import 新模块。
3. 对比或注意点：只处理变化的模块，不需要重新打包，所以速度与项目规模无关。
4. 可追问时补充：时间戳参数用于绕过浏览器的模块缓存。

**常见追问**

- 边界以上的模块会重新执行吗？→ 不会，只有边界及其到变化模块之间的模块会重新执行。

---

### Q6. `import.meta.hot` 有哪些 API？

- **难度**: 中级
- **标签**: HMR, API

**答案**

| API | 作用 |
|-----|------|
| `accept(cb)` | 自身接受更新（self-accepting） |
| `accept(dep, cb)` / `accept([deps], cb)` | 接受依赖的更新 |
| `dispose(cb)` | 模块被替换前清理副作用 |
| `prune(cb)` | 模块不再被导入时清理 |
| `data` | 在新旧模块实例之间传递数据 |
| `invalidate(message?)` | 自己无法处理更新，继续向上传播 |
| `on(event, cb)` / `off` | 监听内置事件（`vite:beforeUpdate` 等）或自定义事件 |
| `send(event, data)` | 向服务器发送自定义事件 |

```ts
let count = import.meta.hot?.data.count ?? 0
const timer = setInterval(() => count++, 1000)

if (import.meta.hot) {
  import.meta.hot.dispose((data) => {
    clearInterval(timer)
    data.count = count
  })
  import.meta.hot.accept()
}
```

注意：`import.meta.hot.accept(` 必须原样出现在代码中（Vite 通过静态分析识别），不能解构或改名调用。

**口述模板（30–60 秒）**

1. 一句话结论：核心是 accept 接受更新、dispose 清理副作用、data 传递状态。
2. 关键机制：accept 定义 HMR 边界，invalidate 把更新继续向上传播。
3. 对比或注意点：调用必须写成静态形式，Vite 才能识别。
4. 可追问时补充：on / send 可以实现自定义 HMR 事件，例如服务端数据变化时推送。

**常见追问**

- `invalidate` 的使用场景？→ 模块接受了更新，但检查后发现无法安全处理（例如导出发生变化），调用 invalidate 让上层处理。

---

### Q7. React Fast Refresh 在 Vite 中是怎么工作的？

- **难度**: 中级
- **标签**: HMR, React

**答案**

`@vitejs/plugin-react` 转换组件文件时：

1. 为每个组件注册 `$RefreshReg$`，并记录 Hooks 签名 `$RefreshSig$`
2. 文件末尾检查导出：**只导出 React 组件**时，模块变成 self-accepting（HMR 边界）
3. 更新时调用 React Refresh 运行时 `performReactRefresh`，保留组件状态重新渲染
4. Hooks 签名变化（增删 Hook）时，重新挂载组件而不是保留状态

失效情况：

- 文件同时导出组件和非组件（常量、工具函数）→ 调用 `invalidate`，向上传播，可能整页刷新
- 匿名默认导出组件无法注册

v6 起使用 Oxc 实现 Refresh 转换，不再依赖 Babel。

**口述模板（30–60 秒）**

1. 一句话结论：plugin-react 把只导出组件的文件变成 HMR 边界，用 React Refresh 运行时保留状态更新。
2. 关键机制：注册组件和 Hooks 签名，签名不变则保留状态。
3. 对比或注意点：混合导出非组件会导致 HMR 失效。
4. 可追问时补充：ESLint 插件 `eslint-plugin-react-refresh` 可以检查这个问题。

**常见追问**

- 为什么混合导出就不行？→ 非组件导出变化后，使用它的模块不会更新，可能拿到旧值，所以只能交给上层处理。

---

### Q8. Vite 开发和生产中 CSS 的处理有什么不同？

- **难度**: 中级
- **标签**: CSS

**答案**

**开发时**

- `import './a.css'` → 请求返回一个 **JS 模块**，内容是调用 `updateStyle(id, css)` 插入 `<style>` 标签
- 这个 JS 模块是 self-accepting 的，CSS 变化时直接替换 style 内容，无需刷新
- HTML 中 `<link>` 引用的 CSS 通过 `?direct` 返回真正的 CSS

**生产时**

- CSS 被抽取为独立文件
- **CSS 代码分割**：每个异步 chunk 的 CSS 单独抽取，加载 chunk 前先加载对应 CSS（通过 preload helper）
- 用 Lightning CSS 压缩（Vite 8 默认）
- `build.cssCodeSplit: false` 时合并为一个 CSS 文件

**口述模板（30–60 秒）**

1. 一句话结论：开发时 CSS 是注入 style 的 JS 模块，生产时抽取为 CSS 文件。
2. 关键机制：开发时模块 self-accepting 实现 CSS HMR；生产按 chunk 分割，随 chunk 加载。
3. 对比或注意点：开发和生产 CSS 的加载顺序可能不同，可能导致样式优先级差异。
4. 可追问时补充：CSS Modules 的类名在开发和生产生成规则不同。

**常见追问**

- 为什么会出现开发正常、生产样式被覆盖？→ 生产 CSS 合并和分割后，规则顺序与开发时不同，依赖顺序的样式会出问题。

---

## 二、插件机制

### Q9. Vite 插件有哪些钩子？执行顺序是怎样的？

- **难度**: 中级
- **标签**: 插件

**答案**

**Rollup / Rolldown 兼容钩子**：`options`、`buildStart`、`resolveId`、`load`、`transform`、`buildEnd`、`closeBundle`，以及构建时的输出钩子（`renderChunk`、`generateBundle` 等）。

**Vite 特有钩子**：

| 钩子 | 时机与作用 |
|------|-----------|
| `config` | 修改配置，在解析配置前 |
| `configResolved` | 读取最终配置 |
| `configureServer` | 拿到开发服务器实例，添加中间件 |
| `configurePreviewServer` | 同上，用于 preview |
| `transformIndexHtml` | 转换 HTML |
| `hotUpdate`（Vite 6+）/ `handleHotUpdate` | 自定义 HMR 处理 |

**插件执行顺序**：

```text
alias 插件
→ 用户插件（enforce: 'pre'）
→ Vite 核心插件
→ 用户插件（无 enforce）
→ Vite 构建插件
→ 用户插件（enforce: 'post'）
→ Vite 后置构建插件（压缩、manifest、报告）
```

`apply: 'serve' | 'build'` 或函数可以控制插件只在开发或构建时生效。

**口述模板（30–60 秒）**

1. 一句话结论：Vite 插件包括 Rollup 兼容钩子和 Vite 特有钩子。
2. 关键机制：resolveId / load / transform 处理模块，config / configureServer / transformIndexHtml / hotUpdate 是 Vite 特有的。
3. 对比或注意点：用 enforce 控制插件顺序，用 apply 控制生效阶段。
4. 可追问时补充：需要在 Vite 核心插件之前处理源码时用 `enforce: 'pre'`。

**常见追问**

- 什么时候需要 `enforce: 'pre'`？→ 需要在 Vite 内置转换（如 TS 转换）之前拿到原始源码时。

---

### Q10. 开发时哪些 Rollup 钩子会被调用？

- **难度**: 中级
- **标签**: 插件

**答案**

| 时机 | 钩子 |
|------|------|
| 服务器启动 | `options`、`buildStart` |
| 每个模块请求 | `resolveId`、`load`、`transform` |
| 服务器关闭 | `buildEnd`、`closeBundle` |

**不会被调用的**：

- `moduleParsed`：开发时为了性能不做完整 AST 解析
- 输出相关钩子：`renderChunk`、`generateBundle`、`writeBundle` 等，因为开发时不打包

所以：依赖 `generateBundle` 生成文件的插件，开发时需要另外用 `configureServer` 中间件提供同样的内容。

**口述模板（30–60 秒）**

1. 一句话结论：开发时只调用启动、模块处理和关闭相关的钩子。
2. 关键机制：resolveId / load / transform 按请求调用，输出钩子不调用。
3. 对比或注意点：moduleParsed 在开发时不调用。
4. 可追问时补充：需要开发时生效的产物要通过中间件提供。

**常见追问**

- `this.emitFile` 在开发时有效吗？→ 发射 asset 时不会真正写出文件，需要通过中间件提供。

---

### Q11. 什么是虚拟模块？怎么实现？

- **难度**: 中级
- **标签**: 插件

**答案**

虚拟模块是**不存在于磁盘、由插件动态生成内容**的模块：

```ts
const virtualId = 'virtual:build-info'
const resolvedId = '\0' + virtualId

export function buildInfo(): Plugin {
  return {
    name: 'build-info',
    resolveId(id) {
      if (id === virtualId) return resolvedId
    },
    load(id) {
      if (id === resolvedId) {
        return `export const buildTime = ${JSON.stringify(new Date().toISOString())}`
      }
    },
  }
}
```

约定：

- 用户使用的说明符带 `virtual:` 前缀
- 内部解析后的 id 加 `\0` 前缀，告诉其他插件（如 commonjs、source map 处理）不要处理这个模块
- 浏览器中 `\0` 会被编码为 `/@id/__x00__`

应用：注入构建信息、约定式路由清单、`virtual:pwa-register` 等。

**口述模板（30–60 秒）**

1. 一句话结论：虚拟模块是由插件生成内容的模块。
2. 关键机制：resolveId 返回带 `\0` 前缀的 id，load 返回代码。
3. 对比或注意点：`\0` 是 Rollup 约定，避免其他插件处理它。
4. 可追问时补充：TS 需要声明 `declare module 'virtual:build-info'`。

**常见追问**

- 虚拟模块怎么支持 HMR？→ 依赖的数据变化时，在 hotUpdate 中找到虚拟模块并使其失效，再触发更新。

---

### Q12. 什么是钩子过滤器（hook filter）？为什么需要它？

- **难度**: 中级
- **标签**: 插件, 性能

**答案**

传统写法中，插件在钩子内部用 `if` 判断是否处理：

```ts
transform(code, id) {
  if (!id.endsWith('.md')) return
  // ...
}
```

在 Rolldown（Rust）中，**每次调用 JS 钩子都有跨语言通信开销**，即使大部分调用直接 return。

钩子过滤器（Rolldown、Rollup 4.38+、Vite 6.3+ 支持）让过滤在 Rust 侧完成：

```ts
transform: {
  filter: {
    id: /\.md$/,
    code: 'import.meta.glob', // 可选：代码包含某个字符串才调用
  },
  handler(code, id) {
    // 只有匹配的模块才会进入这里
  },
},
```

**口述模板（30–60 秒）**

1. 一句话结论：钩子过滤器在调用 JS 钩子之前就过滤掉无关模块。
2. 关键机制：过滤在 Rust 侧执行，避免无意义的跨语言调用。
3. 对比或注意点：在 Rolldown 中，插件不加过滤器会明显拖慢构建。
4. 可追问时补充：resolveId、load、transform 都支持过滤器。

**常见追问**

- 不支持过滤器的旧版本怎么兼容？→ 在 handler 中保留原来的 if 判断，过滤器作为优化。

---

## 三、生产构建

### Q13. Vite 生产构建的流程是什么？

- **难度**: 中级
- **标签**: 构建

**答案**

```text
vite build
  → resolveConfig（mode = production）
  → 创建 Rolldown 构建，输入为 index.html（或 build.lib / input 配置）
  → vite:build-html 插件从 HTML 中提取入口脚本和样式
  → Rolldown 打包：解析、转换（Oxc）、Tree Shaking、代码分割
  → Vite 构建插件：
       - CSS 抽取与代码分割
       - 资源处理与 hash
       - 动态导入预加载（modulepreload）
       - 写回 HTML，替换为产物路径
  → 压缩：Oxc（JS）、Lightning CSS（CSS）
  → 输出到 dist，可选生成 manifest（build.manifest）
```

**口述模板（30–60 秒）**

1. 一句话结论：构建时以 HTML 为入口，用 Rolldown 打包，再由 Vite 插件处理 CSS、资源、预加载和 HTML。
2. 关键机制：Rolldown 负责解析、Tree Shaking 和分割，Oxc 和 Lightning CSS 负责压缩。
3. 对比或注意点：Vite 8 之前用 Rollup 打包、esbuild 压缩。
4. 可追问时补充：后端渲染 HTML 时，可以用 manifest 找到入口对应的产物。

**常见追问**

- manifest 有什么用？→ 后端模板（如 Django、Rails）集成 Vite 时，根据 manifest 输出正确的 script 和 link 标签。

---

### Q14. Vite 8 中怎么配置代码分割？和 `manualChunks` 有什么区别？

- **难度**: 中级
- **标签**: 构建, 代码分割

**答案**

Vite 8 中 `manualChunks` 的对象形式已移除，函数形式已废弃，推荐使用 Rolldown 的 `codeSplitting`：

```ts
build: {
  rolldownOptions: {
    output: {
      codeSplitting: {
        groups: [
          {
            name: 'react',
            test: /node_modules[\\/](react|react-dom|scheduler)[\\/]/,
            priority: 20,
          },
          {
            name: 'vendor',
            test: /node_modules/,
            priority: 10,
            minSize: 20_000,
            maxSize: 500_000,
          },
        ],
      },
    },
  },
},
```

与 `manualChunks` 的区别：

- `manualChunks` 只把模块强制放进某个 chunk，**不处理依赖**，容易导致循环 chunk 依赖和执行顺序问题
- `codeSplitting.groups` 类似 webpack 的 `splitChunks.cacheGroups`：支持优先级、大小限制、共享次数等，更可控
- 使用 groups 时会生成一个 `runtime.js` chunk

**口述模板（30–60 秒）**

1. 一句话结论：Vite 8 用 Rolldown 的 codeSplitting groups 替代 manualChunks。
2. 关键机制：按 test 匹配模块，支持 priority、minSize、maxSize 等规则。
3. 对比或注意点：manualChunks 容易产生循环依赖和执行顺序问题。
4. 可追问时补充：配置方式和 webpack 的 cacheGroups 很像，迁移思路相通。

**常见追问**

- 为什么要 priority？→ 一个模块可能同时匹配多个分组，priority 决定它进入哪个分组。

---

### Q15. Vite 怎么做模块预加载？

- **难度**: 中级
- **标签**: 构建, 性能

**答案**

**入口 chunk**：HTML 中为入口依赖的 chunk 生成 `<link rel="modulepreload">`，让浏览器并行下载，而不是等解析入口后再发现。

**动态导入**：构建时把 `import('./Page')` 改写为：

```js
__vitePreload(() => import('./Page-abc.js'), ['assets/Page-abc.js', 'assets/shared-def.js', 'assets/Page-abc.css'])
```

加载 Page 时，**同时**预加载它依赖的 JS 和 CSS，避免请求瀑布（先加载 Page，发现依赖 shared，再加载 shared）。

相关配置：

- `build.modulePreload.polyfill`：为不支持 modulepreload 的浏览器注入 polyfill
- `build.modulePreload.resolveDependencies`：自定义预加载列表

预加载失败时触发 `vite:preloadError` 事件（见 [高级 Q15](30-senior-qa.md)）。

**口述模板（30–60 秒）**

1. 一句话结论：Vite 通过 modulepreload 和 `__vitePreload` 并行加载依赖，避免瀑布。
2. 关键机制：构建时已知每个 chunk 的依赖，动态导入时同时加载依赖的 JS 和 CSS。
3. 对比或注意点：没有预加载的话，深层依赖要逐层发现，延迟累加。
4. 可追问时补充：加载失败会触发 `vite:preloadError`，可以用来处理发版后的 chunk 404。

**常见追问**

- 预加载会不会浪费带宽？→ 只预加载即将执行的 chunk 的依赖，不会预加载所有 chunk。

---

### Q16. 开发和生产行为不一致的常见原因有哪些？

- **难度**: 中级
- **标签**: 排查

**答案**

| 原因 | 说明 |
|------|------|
| 打包与不打包 | 开发时是独立的 ESM 模块，生产时合并成 chunk，模块执行顺序、循环依赖表现可能不同 |
| 工具不同（Vite 8 之前） | 开发用 esbuild，生产用 Rollup，CJS 互操作行为不同 |
| CSS 顺序 | 开发时按导入顺序注入，生产时合并和分割后顺序可能变化 |
| Tree Shaking | 生产时移除了被认为无副作用的代码，`sideEffects` 配置错误会导致代码丢失 |
| 压缩 | 依赖函数名（`fn.name`）、class 名的代码在压缩后失效 |
| 环境变量 | `import.meta.env.DEV` 等分支不同 |
| base 和资源路径 | 开发时是根路径，生产时有 base 前缀 |

Vite 8 统一使用 Rolldown 后，第二类问题（工具不同）大幅减少，CJS 互操作在两边保持一致。

**排查方法**：用 `vite build --mode development` + `vite preview` 复现；关闭压缩（`build.minify: false`）二分定位；开启 source map。

**口述模板（30–60 秒）**

1. 一句话结论：主要来自打包与否、CSS 顺序、Tree Shaking、压缩和环境分支的差异。
2. 关键机制：开发时是独立模块，生产时打包合并，执行顺序和副作用处理不同。
3. 对比或注意点：Vite 8 统一打包器后，工具差异引起的问题少了很多。
4. 可追问时补充：用 preview 复现，再逐个关闭压缩、Tree Shaking 来二分定位。

**常见追问**

- 怎么快速判断是不是压缩导致的？→ 设置 `build.minify: false` 后重新构建，问题消失就是压缩导致。

---

### Q17. 环境变量的原理是什么？`define` 和 `import.meta.env` 有什么区别？

- **难度**: 中级
- **标签**: 环境变量

**答案**

**`import.meta.env`**

- 开发时：Vite 在模块中注入 `import.meta.env` 对象
- 构建时：`import.meta.env.VITE_XXX` 被**静态替换**为字面量，Tree Shaking 可以删除死分支

**`define`**

```ts
define: {
  __APP_VERSION__: JSON.stringify(pkg.version),
}
```

- 全局常量替换，构建时直接替换代码中的标识符
- 值必须是 JSON 可序列化的表达式字符串

**在配置文件中读取环境变量**：

```ts
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '') // 第三个参数为 '' 时加载所有变量，不限前缀
  return {
    server: { proxy: { '/api': env.API_TARGET } },
  }
})
```

**口述模板（30–60 秒）**

1. 一句话结论：两者都是构建时替换，import.meta.env 管理 .env 文件中的变量，define 定义任意全局常量。
2. 关键机制：静态替换后，死分支可以被 Tree Shaking 删除。
3. 对比或注意点：配置文件中要用 loadEnv 读取。
4. 可追问时补充：所有客户端可见的变量都会进入产物，不能放密钥。

**常见追问**

- 用 `import.meta.env[key]` 动态访问可以吗？→ 构建时无法静态替换，结果不可靠。

---

### Q18. 怎么根据不同命令和环境返回不同配置？

- **难度**: 中级
- **标签**: 配置

**答案**

```ts
export default defineConfig(({ command, mode, isSsrBuild, isPreview }) => {
  const isBuild = command === 'build'
  return {
    base: isBuild ? 'https://cdn.example.com/app/' : '/',
    build: {
      sourcemap: mode === 'staging' ? true : 'hidden',
    },
    define: {
      __ENABLE_MOCK__: JSON.stringify(mode === 'development'),
    },
  }
})
```

- `command`：`'serve'` 或 `'build'`
- `mode`：`development`、`production` 或自定义
- `isSsrBuild`：是否 SSR 构建
- `isPreview`：是否 preview

多套配置可以用 `mergeConfig` 合并基础配置：

```ts
import { mergeConfig } from 'vite'
import base from './vite.base'

export default mergeConfig(base, { server: { port: 3000 } })
```

**口述模板（30–60 秒）**

1. 一句话结论：配置导出函数，根据 command 和 mode 返回不同配置。
2. 关键机制：command 区分开发和构建，mode 区分环境。
3. 对比或注意点：多个配置用 mergeConfig 合并，而不是对象展开（展开会覆盖数组和嵌套对象）。
4. 可追问时补充：配置函数可以是 async 的。

**常见追问**

- 为什么不用对象展开合并？→ 浅合并会覆盖嵌套对象，插件数组也会被替换。

---

### Q19. Vite 怎么处理 CommonJS 依赖？

- **难度**: 中级
- **标签**: 模块化

**答案**

- **开发时**：CJS 依赖经过预构建转换为 ESM
- **生产时**：Rolldown 原生支持 CJS（Vite 8 之前用 `@rollup/plugin-commonjs`）

**默认导入的互操作问题**：

```js
// some-cjs-lib
module.exports = { foo: 1 }

// 使用方
import lib from 'some-cjs-lib' // lib 是 module.exports 还是 module.exports.default？
```

Vite 8 统一了规则：导入方是 `.mjs` / `.mts`、`type: module` 包，或者 CJS 模块没有设置 `__esModule: true` 时，`default` 就是 `module.exports`。旧行为可以用 `legacy.inconsistentCjsInterop` 恢复（作为过渡）。

**源码中写 CJS**：Vite 不支持在浏览器源码中使用 `require`，需要改为 ESM。

**口述模板（30–60 秒）**

1. 一句话结论：开发时预构建转换 CJS，生产时 Rolldown 原生处理。
2. 关键机制：最容易出问题的是默认导入的互操作。
3. 对比或注意点：Vite 8 之前开发和生产的互操作规则不同，是 dev/prod 不一致的常见原因。
4. 可追问时补充：Vite 8 统一了规则，升级后需要检查默认导入是否变化。

**常见追问**

- 升级 Vite 8 后某个库导入变成 `{ default: ... }` 了？→ 互操作规则变化导致，可以改用命名空间导入或临时开启 `legacy.inconsistentCjsInterop`。

---

### Q20. Monorepo 中链接的本地包，Vite 怎么处理？

- **难度**: 中级
- **标签**: Monorepo

**答案**

- 通过 workspace 链接的包（`node_modules` 中是软链接，真实路径不在 `node_modules` 中）**默认不预构建**，当作源码处理，修改后支持 HMR
- 如果链接的包是 CJS，需要加入 `optimizeDeps.include` 让它预构建
- 真实路径在项目根目录之外时，需要允许访问：`server.fs.allow: ['..']`（Vite 会自动检测 workspace 根目录）
- 多个包依赖同一个库（如 React）时，用 `resolve.dedupe: ['react', 'react-dom']` 保证只有一份

**口述模板（30–60 秒）**

1. 一句话结论：链接的本地包默认当作源码处理，不预构建。
2. 关键机制：这样修改本地包可以直接 HMR。
3. 对比或注意点：CJS 格式的本地包要加入 include，单例依赖要 dedupe。
4. 可追问时补充：注意 server.fs.allow 的访问范围。

**常见追问**

- 本地包导出的是构建产物还是源码？→ 开发时导出源码（通过 exports 条件或 alias）体验最好，发布时导出构建产物。

---

### Q21. Vite 开发服务器有哪些安全注意点？

- **难度**: 中级
- **标签**: 安全

**答案**

- **`server.fs.allow` / `server.fs.deny`**：限制可以通过 `/@fs/` 访问的文件范围，`deny` 默认包含 `.env`、`*.{crt,pem}` 等敏感文件。Vite 历史上出现过多个绕过 `fs.deny` 的漏洞，需要及时升级补丁版本
- **`server.allowedHosts`**：防止 DNS 重绑定攻击（恶意网站通过 DNS 指向本机访问开发服务器），2025 年 1 月的安全修复中加入
- **`server.host`**：默认只监听 localhost，`--host` 暴露到局域网时要注意风险
- **CORS**：默认只允许 localhost 来源
- **不要把开发服务器暴露到公网**，也不要用 `vite preview` 做生产服务器

**口述模板（30–60 秒）**

1. 一句话结论：开发服务器能访问本地文件，需要限制文件访问范围和请求来源。
2. 关键机制：fs.allow/deny 限制文件访问，allowedHosts 防止 DNS 重绑定。
3. 对比或注意点：不要暴露到公网，及时升级安全补丁。
4. 可追问时补充：Vite 的支持策略中，旧版本只有安全补丁，要关注升级。

**常见追问**

- 用内网穿透给别人预览时需要注意什么？→ 需要把域名加入 allowedHosts，最好用构建产物而不是开发服务器。

---

### Q22. 大项目开发时首次打开页面很慢，为什么？怎么优化？

- **难度**: 中级
- **标签**: 性能

**答案**

**原因**：服务器启动快，但浏览器需要请求大量模块：

- 每个源码文件都是一个请求，几千个模块就有几千个请求
- 请求有**瀑布**：加载 A 才知道要加载 B
- barrel 文件（`index.ts` 重新导出几百个模块）会导致大量无用模块被加载
- 插件 transform 慢

**优化**：

1. **`server.warmup`**：提前转换常用文件

```ts
server: {
  warmup: { clientFiles: ['./src/main.tsx', './src/components/**/*.tsx'] },
},
```

2. 避免 barrel 文件，直接导入具体路径
3. 减少 `resolve.extensions` 和省略扩展名的导入（每次都要尝试多个扩展名）
4. 检查慢插件：`vite --debug plugin-transform`、`vite-plugin-inspect`、Vite Devtools
5. 页面按路由懒加载，减少首屏模块数
6. Vite 8.1 实验性 Bundled Dev Mode：开发时也打包

**口述模板（30–60 秒）**

1. 一句话结论：大项目首屏慢是因为请求数量多和请求瀑布。
2. 关键机制：每个模块一个请求，依赖层层发现。
3. 对比或注意点：优化方向是减少模块数（治理 barrel）、提前转换（warmup）和排查慢插件。
4. 可追问时补充：模块数极大时可以尝试 Bundled Dev Mode。

**常见追问**

- 为什么 barrel 文件影响这么大？→ 开发时没有 Tree Shaking，导入 barrel 会加载它重新导出的所有模块。

---

### Q23. 怎么分析 Vite 的性能问题？

- **难度**: 中级
- **标签**: 性能, 工具

**答案**

| 工具 | 用途 |
|------|------|
| `vite --debug` / `--debug plugin-transform` | 查看各阶段和插件转换耗时 |
| `vite --profile` | 生成 CPU profile，用 speedscope 或 Chrome 分析 |
| `vite-plugin-inspect` | 查看每个模块经过各插件转换后的结果和耗时 |
| Vite Devtools（Vite 8 内置，`devtools` 选项） | 可视化模块图、插件耗时、构建分析 |
| `rollup-plugin-visualizer` | 产物体积分析 |
| 浏览器 Network 面板 | 开发时请求数、瀑布 |

**口述模板（30–60 秒）**

1. 一句话结论：用 debug 日志、profile 和 inspect 定位慢点。
2. 关键机制：profile 看 CPU 时间分布，inspect 看每个插件的转换耗时。
3. 对比或注意点：开发慢多看 Network 面板的请求数，构建慢多看插件耗时。
4. 可追问时补充：Vite 8 内置的 Devtools 可以统一查看。

**常见追问**

- 发现某个插件很慢怎么办？→ 加钩子过滤器、缩小处理范围、缓存结果，或者寻找 Rust 实现的替代品。

---

## 四、SSR、Environment API 与其他

### Q24. Vite 的 SSR 是怎么工作的？

- **难度**: 中级
- **标签**: SSR

**答案**

**开发时（中间件模式）**：

```ts
const vite = await createServer({
  server: { middlewareMode: true },
  appType: 'custom',
})
app.use(vite.middlewares)

app.use('*', async (req, res) => {
  let template = fs.readFileSync('index.html', 'utf-8')
  template = await vite.transformIndexHtml(req.originalUrl, template)
  const { render } = await vite.ssrLoadModule('/src/entry-server.tsx')
  const html = template.replace('<!--app-html-->', await render(req.originalUrl))
  res.end(html)
})
```

- `ssrLoadModule` 在 Node 中加载模块，使用同一套插件转换，并支持 HMR 后失效
- Vite 6 起推荐使用 Module Runner（`createServerModuleRunner`）

**构建时**：

- 客户端：`vite build --outDir dist/client --ssrManifest`
- 服务端：`vite build --ssr src/entry-server.tsx --outDir dist/server`

**依赖处理**：

- SSR 时依赖默认 external（直接用 Node 的 `require` / `import` 加载）
- 需要转换的依赖（例如导入了 CSS 的组件库）配置 `ssr.noExternal`

**口述模板（30–60 秒）**

1. 一句话结论：开发时用中间件模式加 ssrLoadModule，构建时分别构建客户端和服务端。
2. 关键机制：服务端模块也经过 Vite 插件转换，所以可以直接使用 JSX、CSS 导入等。
3. 对比或注意点：依赖默认 external，需要转换的依赖要 noExternal。
4. 可追问时补充：Vite 6 起有 Module Runner 和 Environment API。

**常见追问**

- SSR manifest 有什么用？→ 记录模块对应的产物，渲染时可以输出当前页面需要的 preload 标签。

---

### Q25. 什么是 Environment API？

- **难度**: 中级
- **标签**: Environment API

**答案**

**问题**：Vite 5 及之前，开发服务器只有 client 和 ssr 两种隐含环境。但现代框架可能需要更多运行环境：浏览器、Node、边缘运行时（Cloudflare Workers）、React Server Components 的 server 环境等。

**Environment API（Vite 6 引入，实验性，计划在 Vite 9 稳定）**：

- 一个开发服务器中可以有**多个环境**，每个环境有独立的模块图、插件管线和配置
- 模块可以在**非 Node 运行时**中执行：通过 Module Runner 和传输层，在 workerd 等环境中运行

```ts
export default defineConfig({
  environments: {
    client: {},
    ssr: {},
    edge: {
      resolve: { noExternal: true },
      // 自定义 dev 环境，例如在 workerd 中运行
    },
  },
})
```

- 插件中可以通过 `this.environment` 获取当前环境，用 `applyToEnvironment` 决定插件对哪些环境生效

主要面向框架和平台作者；普通应用开发者通常不需要直接使用。

**口述模板（30–60 秒）**

1. 一句话结论：Environment API 让 Vite 在一个开发服务器中支持多个运行环境。
2. 关键机制：每个环境有独立的模块图和插件管线，通过 Module Runner 在目标运行时执行代码。
3. 对比或注意点：解决了以前只有 client 和 ssr、只能在 Node 中运行的限制。
4. 可追问时补充：Cloudflare 的 Vite 插件就是利用它在开发时直接运行 workerd。

**常见追问**

- 对普通开发者有什么意义？→ 开发环境更接近生产运行时，例如边缘函数在开发时就运行在真实的 workerd 中。

---

### Q26. 库模式有哪些进阶配置？

- **难度**: 中级
- **标签**: 库模式

**答案**

```ts
export default defineConfig({
  plugins: [dts({ rollupTypes: true })],
  build: {
    lib: {
      entry: {
        index: 'src/index.ts',
        hooks: 'src/hooks/index.ts',     // 多入口
      },
      formats: ['es'],
    },
    rolldownOptions: {
      external: [/^react/, /^@radix-ui\//],  // 正则 external 所有子路径
      output: {
        preserveModules: true,             // 保留模块结构，利于使用方 Tree Shaking
        preserveModulesRoot: 'src',
      },
    },
    cssCodeSplit: true,
  },
})
```

要点：

- 只输出 ESM 越来越常见（Node 20.19+ 支持 `require(esm)`）
- 依赖和 peer 依赖都应该 external，不要打包进库
- `preserveModules` 让使用方可以按需 Tree Shaking
- `package.json` 正确设置 `exports`、`types`、`sideEffects`
- 纯 TS 库也可以考虑 tsdown

**口述模板（30–60 秒）**

1. 一句话结论：库模式要注意多入口、external、保留模块结构和类型声明。
2. 关键机制：external 避免重复打包和多实例，preserveModules 利于使用方 Tree Shaking。
3. 对比或注意点：package.json 的 exports 和 sideEffects 同样重要。
4. 可追问时补充：组件库带样式时，需要设计 CSS 的导入方式。

**常见追问**

- `sideEffects` 应该怎么写？→ 只有 CSS 文件有副作用时写 `["**/*.css"]`。

---

### Q27. 怎么兼容旧浏览器？Vite 8 中有哪些限制？

- **难度**: 中级
- **标签**: 兼容性

**答案**

`@vitejs/plugin-legacy` 的原理：

- 生成**两套产物**：现代产物（`<script type="module">`）和兼容产物（SystemJS 格式，`<script nomodule>`）
- 兼容产物经过 Babel 转换，并根据 `targets` 注入 core-js polyfill
- 现代浏览器只加载现代产物，旧浏览器加载兼容产物
- `modernPolyfills` 可以为现代产物也注入必要的 polyfill

Vite 8 的限制（以官方迁移指南为准）：

- Oxc 不支持把语法降级到 ES5 及以下，plugin-legacy 转换到 ES5 存在限制
- Oxc 不降级原生装饰器语法，需要通过 Babel 或 SWC 插件处理

实际项目中应根据统计的用户浏览器占比决定是否需要兼容。

**口述模板（30–60 秒）**

1. 一句话结论：plugin-legacy 生成现代和兼容两套产物，通过 module / nomodule 分流。
2. 关键机制：兼容产物使用 Babel 转换并注入 polyfill。
3. 对比或注意点：构建时间和产物体积都会增加。
4. 可追问时补充：Vite 8 下降级到 ES5 有限制，需要评估。

**常见追问**

- 能不能只注入 polyfill 不降级语法？→ 可以用 `modernPolyfills` 或者在入口手动引入 core-js 的部分模块。

---

### Q28. Vite 中的 Web Worker 是怎么打包的？

- **难度**: 中级
- **标签**: Worker

**答案**

- 识别 `new Worker(new URL('./worker.ts', import.meta.url))` 或 `?worker` 导入
- Worker 文件**作为独立入口单独打包**（包括它导入的依赖），输出为独立文件
- `worker.format`：`'es'`（默认是 iife，`'es'` 需要浏览器支持 module worker）
- `worker.plugins`：Worker 打包使用的插件（与主应用分开）
- `worker.rolldownOptions`（Vite 8）：Worker 的打包选项
- `?worker&inline`：把 Worker 代码内联为 base64，适合库模式

开发时：Worker 文件同样按 ESM 请求，需要浏览器支持 module worker。

**口述模板（30–60 秒）**

1. 一句话结论：Worker 作为独立入口单独打包。
2. 关键机制：识别 new URL 或 ?worker 语法，单独构建一份产物。
3. 对比或注意点：Worker 的插件配置与主应用分开。
4. 可追问时补充：库中的 Worker 可以内联，避免使用方处理路径。

**常见追问**

- Worker 中的公共依赖会和主应用共享吗？→ 不会，Worker 运行在独立的上下文中，会单独打包一份。

---

### Q29. Vite 中的静态资源有哪些进阶处理？

- **难度**: 中级
- **标签**: 静态资源

**答案**

- **`build.assetsInlineLimit`**：默认 4096 字节，小于此值的资源内联为 base64；也可以传函数，按文件决定
- **`assetsInclude`**：把其他扩展名（如 `.gltf`）当作静态资源处理
- **`experimental.renderBuiltUrl`**：精细控制产物 URL，例如 JS 和图片使用不同 CDN，或运行时动态拼接：

```ts
experimental: {
  renderBuiltUrl(filename, { hostType }) {
    if (hostType === 'js') {
      return { runtime: `window.__CDN__ + ${JSON.stringify(filename)}` }
    }
    return { relative: true }
  },
},
```

- **`build.assetsDir`**：资源输出子目录，默认 `assets`

**口述模板（30–60 秒）**

1. 一句话结论：通过内联阈值、assetsInclude 和 renderBuiltUrl 控制资源处理和 URL。
2. 关键机制：renderBuiltUrl 可以按资源类型输出不同 URL，或者运行时拼接。
3. 对比或注意点：内联会增加 JS 体积，阈值不宜过大。
4. 可追问时补充：运行时 CDN 地址可以实现一次构建、多环境部署。

**常见追问**

- SVG 内联有什么特殊处理？→ SVG 内联时使用 URL 编码而不是 base64，体积更小。

---

### Q30. Vitest 是怎么复用 Vite 的？

- **难度**: 中级
- **标签**: 测试

**答案**

- Vitest 内部启动一个 Vite 开发服务器（或复用转换管线），测试文件通过 **Vite 插件管线转换**，在 Node 中通过 Module Runner 执行
- 所以 alias、JSX、CSS 导入、虚拟模块等在测试中和开发时表现一致
- watch 模式利用**模块图**：文件变化时，只重新运行依赖该文件的测试
- 支持浏览器模式：测试在真实浏览器中运行
- 支持 in-source testing：

```ts
export function add(a: number, b: number) { return a + b }

if (import.meta.vitest) {
  const { it, expect } = import.meta.vitest
  it('add', () => expect(add(1, 2)).toBe(3))
}
```

**口述模板（30–60 秒）**

1. 一句话结论：Vitest 复用 Vite 的插件管线和模块图。
2. 关键机制：测试文件按 Vite 方式转换，在 Node 中通过模块运行器执行。
3. 对比或注意点：测试与开发环境的转换一致，配置只写一份。
4. 可追问时补充：watch 模式基于模块图只重跑受影响的测试。

**常见追问**

- 测试中需要关闭某些插件怎么办？→ 用 `process.env.VITEST` 判断，或者在 Vitest 配置中单独设置。

---

### Q31. Vite、webpack、Rspack 的原理有什么区别？

- **难度**: 中级
- **标签**: 对比, 选型

**答案**

| 对比项 | Vite 8 | webpack 5 | Rspack |
|--------|--------|-----------|--------|
| 开发模式 | 不打包（8.1 可选打包） | 打包 | 打包 |
| 实现语言 | Rolldown / Oxc（Rust）+ JS 插件 | JS | Rust |
| 生产打包 | Rolldown | webpack | Rspack |
| 插件接口 | Rollup 兼容 | webpack（Tapable） | 兼容大部分 webpack loader / plugin |
| HMR 速度 | 与项目规模无关 | 随项目增大变慢 | 快 |
| 大项目首次加载 | 请求多，可能慢 | 打包后请求少 | 打包后请求少 |
| 迁移成本 | webpack 项目迁移需改写配置和插件 | — | 低（接近 webpack） |

结论：新项目优先选 Vite；存量 webpack 大项目要提速，Rspack 迁移成本最低。详见 [webpack 高级 Q19 / Q21](../webpack/30-senior-qa.md)。

**口述模板（30–60 秒）**

1. 一句话结论：Vite 开发不打包，webpack 和 Rspack 开发时打包，Rspack 是 Rust 版 webpack。
2. 关键机制：Vite 的速度来自不打包加 Rust 工具链，Rspack 的速度来自 Rust 实现的打包。
3. 对比或注意点：Vite 在大项目中首次加载请求多，Rspack 迁移成本最低。
4. 可追问时补充：Vite 8.1 的 Bundled Dev Mode 也是在借鉴打包的优势。

**常见追问**

- 两者会趋同吗？→ 有趋同趋势：Vite 引入开发打包模式，打包器都在向 Rust 实现迁移。

---

### Q32. 从 Vite 7 升级到 Vite 8 需要注意什么？

- **难度**: 中级
- **标签**: 版本, 迁移

**答案**

**配置变化**（大部分有兼容层自动转换，但会提示废弃）：

| 旧 | 新 |
|----|----|
| `build.rollupOptions` | `build.rolldownOptions` |
| `worker.rollupOptions` | `worker.rolldownOptions` |
| `optimizeDeps.esbuildOptions` | `optimizeDeps.rolldownOptions` |
| `esbuild` | `oxc` |
| `build.minify: 'esbuild'` | 默认 Oxc（`'esbuild'` 已废弃，需要单独安装 esbuild） |
| `manualChunks` | `codeSplitting` |
| `build.commonjsOptions` | 无效，Rolldown 原生处理 CJS |

**行为变化**：

- CJS 默认导入的互操作规则统一（见 Q19）
- 移除 `browser` 和 `module` 字段之间的格式嗅探
- 外部依赖的 `require` 调用会被保留
- `build.target` 默认值升级为 Chrome 111、Firefox 114、Safari 16.4
- CSS 默认用 Lightning CSS 压缩

**插件作者需要注意**：

- `load` / `transform` 返回非 JS 内容转换的结果时，需要设置 `moduleType: 'js'`
- 部分 Rollup 钩子不支持：`shouldTransformCachedModule`、`resolveImportMeta`、`renderDynamicImport`、`resolveFileUrl`
- 不支持 `system`、`amd` 输出格式

**升级策略**：先升级到最新的 Vite 7，大项目可以先试用 `rolldown-vite`，再升级到 Vite 8。

**口述模板（30–60 秒）**

1. 一句话结论：主要变化是 rollupOptions 改名 rolldownOptions、esbuild 改为 oxc、manualChunks 改为 codeSplitting。
2. 关键机制：兼容层会自动转换大部分配置，但行为变化（CJS 互操作、target）需要测试。
3. 对比或注意点：插件作者需要检查不支持的钩子和 moduleType。
4. 可追问时补充：大项目可以先通过 rolldown-vite 渐进迁移。

**常见追问**

- 升级后安装体积变大了？→ 是的，约增加 15MB，主要来自 Lightning CSS 和 Rolldown 的原生二进制。
