# Vite 手写与现场编码

> 目标：能现场写出 Vite 核心机制的简化实现和常见插件，并能讲清楚思路。

## 使用方式

- **原理实现类**（H1、H7、H8）：手写迷你版本，重点讲清楚核心流程，不追求完整
- **插件类**（H2–H6、H10）：最常考，要能写出钩子、过滤逻辑和 HMR 处理
- **配置与工程类**（H9、H11、H12）：考察实际项目经验

---

### H1. 手写迷你 Vite 开发服务器

- **难度**: 高级
- **标签**: 原理, 开发服务器
- **面试官看什么**: 是否理解「不打包 + 按需编译 + 裸模块重写」

**题干**

实现一个简化的开发服务器：返回 index.html；请求 JS 时重写裸模块导入；`/@modules/xxx` 从 node_modules 加载依赖的 ESM 入口；CSS 转为注入 style 的 JS 模块。

**思路口述**

浏览器原生支持 ESM，所以服务器只需要在返回 JS 前处理导入路径。裸模块重写为 `/@modules/` 前缀，服务器收到后从 `node_modules` 中找到包的 ESM 入口返回。CSS 不能被 import，所以包装成一段插入 style 标签的 JS。

**参考实现**

```ts
import http from 'node:http'
import fs from 'node:fs/promises'
import path from 'node:path'
import { init, parse } from 'es-module-lexer'
import MagicString from 'magic-string'

const root = process.cwd()

/** 把源码中的裸模块导入重写为 /@modules/ 前缀，浏览器才能请求 */
async function rewriteImports(code: string): Promise<string> {
  await init
  const [imports] = parse(code)
  const s = new MagicString(code)
  for (const { n: specifier, s: start, e: end } of imports) {
    // n 为 undefined 表示动态 import 的参数不是字面量，无法静态重写
    if (!specifier) continue
    const isBare = !specifier.startsWith('.') && !specifier.startsWith('/')
    if (isBare) s.overwrite(start, end, `/@modules/${specifier}`)
  }
  return s.toString()
}

/** 根据包的 package.json 找到 ESM 入口文件 */
async function resolvePackageEntry(name: string): Promise<string> {
  const pkgDir = path.join(root, 'node_modules', name)
  const pkg = JSON.parse(await fs.readFile(path.join(pkgDir, 'package.json'), 'utf-8'))
  // 真实 Vite 会解析 exports 条件；这里简化为 module 字段，CJS 包需要预构建（见 H8）
  return path.join(pkgDir, pkg.module ?? pkg.main)
}

/** 把 CSS 包装成注入 style 标签的 JS 模块 */
function cssToModule(css: string, id: string): string {
  return `
const id = ${JSON.stringify(id)}
let style = document.querySelector(\`style[data-id="\${id}"]\`)
if (!style) {
  style = document.createElement('style')
  style.dataset.id = id
  document.head.appendChild(style)
}
style.textContent = ${JSON.stringify(css)}
`
}

const server = http.createServer(async (req, res) => {
  const url = req.url ?? '/'
  try {
    if (url === '/') {
      res.setHeader('Content-Type', 'text/html')
      res.end(await fs.readFile(path.join(root, 'index.html'), 'utf-8'))
      return
    }

    if (url.startsWith('/@modules/')) {
      const entry = await resolvePackageEntry(url.slice('/@modules/'.length))
      res.setHeader('Content-Type', 'application/javascript')
      res.end(await rewriteImports(await fs.readFile(entry, 'utf-8')))
      return
    }

    const filePath = path.join(root, url.split('?')[0])
    if (url.endsWith('.js')) {
      res.setHeader('Content-Type', 'application/javascript')
      res.end(await rewriteImports(await fs.readFile(filePath, 'utf-8')))
      return
    }

    if (url.endsWith('.css')) {
      res.setHeader('Content-Type', 'application/javascript')
      res.end(cssToModule(await fs.readFile(filePath, 'utf-8'), url))
      return
    }

    res.statusCode = 404
    res.end()
  } catch (err) {
    res.statusCode = 500
    res.end(String(err))
  }
})

server.listen(3000)
```

**易错点**

- 依赖内部也有裸模块导入，同样需要重写
- 没有预构建的话，CJS 包无法在浏览器中运行
- 真实 Vite 还有转换缓存、模块图、插件容器、HMR，可以在讲完核心流程后补充

---

### H2. 手写虚拟模块插件：注入构建信息

- **难度**: 中级
- **标签**: 插件, 虚拟模块
- **面试官看什么**: resolveId / load 的配合，`\0` 约定，类型声明

**题干**

实现 `virtual:build-info`，导出版本号、构建时间和 git commit。

**思路口述**

resolveId 拦截虚拟 id，返回加 `\0` 前缀的内部 id；load 根据内部 id 生成代码。数据在 `buildStart` 中计算一次，避免每次 load 都执行 git 命令。

**参考实现**

```ts
import { execSync } from 'node:child_process'
import type { Plugin } from 'vite'

const VIRTUAL_ID = 'virtual:build-info'
// \0 前缀告诉其他插件这是虚拟模块，不要尝试从磁盘读取或做 CJS 转换
const RESOLVED_ID = '\0' + VIRTUAL_ID

/** 提供 virtual:build-info 模块，导出版本号、构建时间和 git commit */
export function buildInfoPlugin(version: string): Plugin {
  let info: Record<string, string> = {}

  return {
    name: 'build-info',
    buildStart() {
      let commit = 'unknown'
      try {
        commit = execSync('git rev-parse --short HEAD').toString().trim()
      } catch {
        // 非 git 环境（例如从压缩包构建）不应该导致构建失败
      }
      info = { version, commit, buildTime: new Date().toISOString() }
    },
    resolveId(id) {
      if (id === VIRTUAL_ID) return RESOLVED_ID
    },
    load(id) {
      if (id !== RESOLVED_ID) return
      return Object.entries(info)
        .map(([key, value]) => `export const ${key} = ${JSON.stringify(value)}`)
        .join('\n')
    },
  }
}
```

类型声明：

```ts
// env.d.ts
declare module 'virtual:build-info' {
  export const version: string
  export const commit: string
  export const buildTime: string
}
```

**易错点**

- 忘记 `\0` 前缀
- 值没有 `JSON.stringify`，生成的代码语法错误
- 忘记写类型声明，TS 报错

---

### H3. 手写 transform 插件：生产环境移除 console

- **难度**: 中级
- **标签**: 插件, transform
- **面试官看什么**: 钩子过滤器、apply、source map、AST 与正则的取舍

**题干**

实现插件：生产构建时移除 `console.log` 和 `console.debug`，保留 `console.error`；需要正确的 source map。

**思路口述**

只在构建时生效（`apply: 'build'`）。用钩子过滤器只处理源码中的 JS/TS 并且包含 `console.` 的文件。正则无法正确处理嵌套括号和字符串，所以用 AST 定位调用表达式，再用 magic-string 删除，生成 source map。

**参考实现**

```ts
import type { Plugin } from 'vite'
import MagicString from 'magic-string'
import { parseSync, Visitor } from 'oxc-parser'

/** 生产构建时移除指定的 console 方法调用 */
export function removeConsolePlugin(methods = ['log', 'debug']): Plugin {
  return {
    name: 'remove-console',
    apply: 'build',
    transform: {
      // 在 Rust 侧过滤，避免为每个模块调用 JS 钩子
      filter: {
        id: { include: /\.[jt]sx?$/, exclude: /node_modules/ },
        code: 'console.',
      },
      handler(code, id) {
        const { program } = parseSync(id, code)
        const s = new MagicString(code)

        new Visitor({
          ExpressionStatement(node) {
            const expr = node.expression
            if (
              expr.type === 'CallExpression' &&
              expr.callee.type === 'MemberExpression' &&
              expr.callee.object.type === 'Identifier' &&
              expr.callee.object.name === 'console' &&
              expr.callee.property.type === 'Identifier' &&
              methods.includes(expr.callee.property.name)
            ) {
              // 只删除独立的语句；作为表达式使用的 console（如 a && console.log()）保留，避免破坏语法
              s.remove(node.start, node.end)
            }
          },
        }).visit(program)

        if (!s.hasChanged()) return
        return { code: s.toString(), map: s.generateMap({ hires: true }) }
      },
    },
  }
}
```

> `oxc-parser` 的 Visitor API 以其文档为准；面试中用 `@babel/parser` + `@babel/traverse` 写同样合格。实际项目中也可以直接用压缩器的 `drop_console` / `pure_funcs` 类选项实现。

**易错点**

- 用正则替换，遇到 `console.log(fn(a, b))` 或字符串中的括号就会出错
- 不返回 source map，导致 source map 链断裂
- 处理了 node_modules，增加构建耗时

---

### H4. 手写 Mock 插件：开发服务器中间件

- **难度**: 中级
- **标签**: 插件, configureServer
- **面试官看什么**: configureServer 的使用、中间件顺序、热更新 mock 文件

**题干**

实现插件：读取 `mock/*.ts` 中定义的接口，开发时拦截匹配的请求返回 mock 数据；修改 mock 文件后不重启即可生效。

**思路口述**

在 configureServer 中注册中间件，并且放在内置中间件之前，这样能在代理之前拦截。mock 文件用 `server.ssrLoadModule` 加载，复用 Vite 的转换（支持 TS），文件变化时让模块失效，下次请求自动加载新版本。

**参考实现**

```ts
import type { Plugin, ViteDevServer } from 'vite'
import { glob } from 'node:fs/promises'

/** mock 文件导出的单条接口定义 */
export interface MockRoute {
  method?: 'GET' | 'POST' | 'PUT' | 'DELETE'
  url: string
  response: (ctx: { query: URLSearchParams; body: unknown }) => unknown
  delay?: number
}

async function loadRoutes(server: ViteDevServer, pattern: string): Promise<MockRoute[]> {
  const routes: MockRoute[] = []
  for await (const file of glob(pattern)) {
    // ssrLoadModule 复用 Vite 的转换管线，mock 文件可以直接写 TS
    const mod = await server.ssrLoadModule('/' + file)
    routes.push(...(mod.default as MockRoute[]))
  }
  return routes
}

function readBody(req: import('node:http').IncomingMessage): Promise<unknown> {
  return new Promise((resolve) => {
    let raw = ''
    req.on('data', (chunk) => (raw += chunk))
    req.on('end', () => {
      try {
        resolve(raw ? JSON.parse(raw) : undefined)
      } catch {
        resolve(raw)
      }
    })
  })
}

/** 开发环境 mock 插件：拦截 mock 目录中定义的接口 */
export function mockPlugin(options: { pattern?: string } = {}): Plugin {
  const pattern = options.pattern ?? 'mock/**/*.ts'

  return {
    name: 'mock',
    apply: 'serve',
    configureServer(server) {
      // 不返回函数：中间件注册在内置中间件之前，优先于 server.proxy 拦截
      server.middlewares.use(async (req, res, next) => {
        const url = new URL(req.url ?? '/', 'http://localhost')
        // 每次请求重新加载：ssrLoadModule 有缓存，文件变化后 Vite 会使对应模块失效
        const routes = await loadRoutes(server, pattern)
        const route = routes.find(
          (r) => r.url === url.pathname && (r.method ?? 'GET') === req.method,
        )
        if (!route) return next()

        const body = await readBody(req)
        if (route.delay) await new Promise((r) => setTimeout(r, route.delay))
        res.setHeader('Content-Type', 'application/json')
        res.end(JSON.stringify(route.response({ query: url.searchParams, body })))
      })
    },
  }
}
```

**易错点**

- 返回函数形式导致中间件在内置中间件之后注册，被 HTML fallback 抢先处理
- 用 `require` 加载 mock 文件，修改后因为 Node 模块缓存不生效
- 忘记 `apply: 'serve'`，构建时也执行

---

### H5. 手写 transformIndexHtml 插件：注入配置和监控脚本

- **难度**: 中级
- **标签**: 插件, HTML
- **面试官看什么**: transformIndexHtml 的返回格式、注入位置、开发与构建区分

**题干**

实现插件：构建时向 HTML 注入运行时配置 `window.__APP_CONFIG__`、CDN 的 preconnect，以及只在生产环境注入的监控 SDK。

**思路口述**

transformIndexHtml 可以返回标签描述数组，由 Vite 负责插入，比字符串替换更可靠。用 `ctx.server` 判断开发还是构建。

**参考实现**

```ts
import type { HtmlTagDescriptor, Plugin } from 'vite'

/** HTML 注入插件的配置 */
export interface HtmlInjectOptions {
  config: Record<string, unknown>
  cdnOrigin?: string
  monitorSrc?: string
}

/** 向 index.html 注入运行时配置、preconnect 和生产环境监控脚本 */
export function htmlInjectPlugin(options: HtmlInjectOptions): Plugin {
  return {
    name: 'html-inject',
    transformIndexHtml(html, ctx) {
      const tags: HtmlTagDescriptor[] = [
        {
          tag: 'script',
          // 配置必须在业务脚本之前可用，所以放在 head 最前面
          injectTo: 'head-prepend',
          // 转义 <，防止配置值中包含 </script> 导致 XSS
          children: `window.__APP_CONFIG__ = ${JSON.stringify(options.config).replace(/</g, '\\u003c')}`,
        },
      ]

      if (options.cdnOrigin) {
        tags.push({
          tag: 'link',
          attrs: { rel: 'preconnect', href: options.cdnOrigin, crossorigin: '' },
          injectTo: 'head',
        })
      }

      // ctx.server 只在开发时存在
      if (!ctx.server && options.monitorSrc) {
        tags.push({
          tag: 'script',
          attrs: { src: options.monitorSrc, async: true },
          injectTo: 'body',
        })
      }

      return { html, tags }
    },
  }
}
```

**易错点**

- JSON 直接拼进 script 标签，没有转义 `<` 导致 XSS
- 开发环境也注入监控脚本，污染监控数据
- 注入位置错误，业务脚本执行时配置还不存在

---

### H6. 手写自定义 HMR：国际化文件热更新

- **难度**: 高级
- **标签**: 插件, HMR
- **面试官看什么**: hotUpdate 钩子、自定义事件、阻止默认行为

**题干**

`locales/*.json` 修改后，不刷新页面，也不重新执行组件模块，只更新国际化文案。

**思路口述**

默认情况下修改 JSON 会沿导入链传播，可能导致整页刷新。在 hotUpdate 中拦截 locales 文件，读取新内容，通过自定义事件发给客户端，然后返回空数组阻止默认 HMR。客户端监听事件，更新 i18n 实例。

**参考实现**

```ts
import path from 'node:path'
import type { Plugin } from 'vite'

/** 国际化 JSON 文件的自定义热更新，避免整页刷新 */
export function i18nHmrPlugin(localesDir = 'src/locales'): Plugin {
  return {
    name: 'i18n-hmr',
    apply: 'serve',
    async hotUpdate({ file, read }) {
      // hotUpdate 会对每个环境调用；只需要通知浏览器端
      if (this.environment.name !== 'client') return
      if (!file.includes(localesDir) || !file.endsWith('.json')) return

      const locale = path.basename(file, '.json')
      const messages = JSON.parse(await read())

      this.environment.hot.send({
        type: 'custom',
        event: 'i18n:update',
        data: { locale, messages },
      })

      // 返回空数组：已经自行处理，阻止 Vite 默认的模块更新传播
      return []
    },
  }
}
```

客户端：

```ts
import i18n from './i18n'

if (import.meta.hot) {
  import.meta.hot.on('i18n:update', ({ locale, messages }) => {
    i18n.addResourceBundle(locale, 'translation', messages, true, true)
    // 触发重新渲染
    i18n.changeLanguage(i18n.language)
  })
}
```

自定义事件的类型可以通过扩展 `vite/types/customEvent.d.ts` 中的 `CustomEventMap` 声明。

**易错点**

- 不判断环境，SSR 环境也发送事件
- 不返回空数组，默认传播仍然会触发刷新
- 客户端代码不在 `if (import.meta.hot)` 中，生产环境报错

---

### H7. 手写迷你 HMR 客户端

- **难度**: 高级
- **标签**: 原理, HMR
- **面试官看什么**: 理解 accept 注册、dispose、带时间戳重新导入

**题干**

实现 `createHotContext(ownerPath)`，支持 `accept`、`dispose`、`data`；连接 WebSocket，收到更新时执行 dispose、重新导入模块并调用 accept 回调。

**思路口述**

每个模块通过 createHotContext 拿到自己的 hot 对象，accept 时把回调注册到全局表中。服务端发来的更新包含边界路径和变化模块路径。客户端先执行旧模块的 dispose，再带时间戳 import 新模块（绕过浏览器 ESM 缓存），最后调用边界注册的回调。

**参考实现**

```ts
type AcceptCallback = (mod: unknown) => void

interface HotModule {
  callbacks: Array<{ deps: string[]; fn: (mods: unknown[]) => void }>
}

interface UpdatePayload {
  path: string          // 边界模块
  acceptedPath: string  // 被接受的模块（自身接受时等于 path）
  timestamp: number
}

const hotModules = new Map<string, HotModule>()
const disposeMap = new Map<string, (data: Record<string, unknown>) => void>()
const dataMap = new Map<string, Record<string, unknown>>()

/** 为指定模块创建 import.meta.hot 对象 */
export function createHotContext(ownerPath: string) {
  // 模块重新执行时会再次调用 createHotContext，需要清掉旧回调，否则会重复执行
  hotModules.delete(ownerPath)

  if (!dataMap.has(ownerPath)) dataMap.set(ownerPath, {})

  return {
    get data() {
      return dataMap.get(ownerPath)!
    },
    accept(depsOrCallback?: string | string[] | AcceptCallback, callback?: AcceptCallback) {
      const mod = hotModules.get(ownerPath) ?? { callbacks: [] }
      if (typeof depsOrCallback === 'function' || depsOrCallback === undefined) {
        mod.callbacks.push({
          deps: [ownerPath],
          fn: ([m]) => (depsOrCallback as AcceptCallback | undefined)?.(m),
        })
      } else {
        const deps = Array.isArray(depsOrCallback) ? depsOrCallback : [depsOrCallback]
        mod.callbacks.push({
          deps,
          fn: (mods) => callback?.(Array.isArray(depsOrCallback) ? mods : mods[0]),
        })
      }
      hotModules.set(ownerPath, mod)
    },
    dispose(cb: (data: Record<string, unknown>) => void) {
      disposeMap.set(ownerPath, cb)
    },
  }
}

async function fetchUpdate({ path, acceptedPath, timestamp }: UpdatePayload) {
  const boundary = hotModules.get(path)
  if (!boundary) return

  const dispose = disposeMap.get(acceptedPath)
  if (dispose) dispose(dataMap.get(acceptedPath)!)

  // 浏览器对同一 URL 的 ESM 只执行一次，必须加时间戳才能拿到新模块
  const newModule = await import(/* @vite-ignore */ `${acceptedPath}?t=${timestamp}`)

  for (const { deps, fn } of boundary.callbacks) {
    if (deps.includes(acceptedPath)) {
      fn(deps.map((dep) => (dep === acceptedPath ? newModule : undefined)))
    }
  }
}

const socket = new WebSocket(`ws://${location.host}`, 'vite-hmr')

socket.addEventListener('message', async ({ data }) => {
  const payload = JSON.parse(data)
  switch (payload.type) {
    case 'update':
      // 多个更新需要按顺序执行，避免后一个更新先完成
      for (const update of payload.updates) await fetchUpdate(update)
      break
    case 'full-reload':
      location.reload()
      break
  }
})
```

**易错点**

- 不加时间戳，import 返回缓存的旧模块
- 模块重新执行时没有清理旧回调，导致回调累积
- 多个更新并发执行，顺序错乱

---

### H8. 手写简化的依赖预构建

- **难度**: 高级
- **标签**: 原理, 预构建
- **面试官看什么**: 理解扫描、打包、hash 缓存

**题干**

给定依赖列表，用打包器把它们打包为 ESM 输出到 `node_modules/.vite/deps`，根据 lockfile 内容计算 hash，hash 不变时跳过。

**思路口述**

先计算 hash，与缓存的 metadata 比较，相同则直接复用。否则把每个依赖作为入口一起打包：一起打包可以抽取公共 chunk，保证共享依赖（如 React）只有一份。最后写入 metadata，供导入重写时查找。

**参考实现**

```ts
import { createHash } from 'node:crypto'
import fs from 'node:fs/promises'
import path from 'node:path'
import { rolldown } from 'rolldown'

/** 预构建结果的元数据 */
export interface DepsMetadata {
  hash: string
  deps: Record<string, string> // 包名 → 产物文件
}

const cacheDir = path.resolve('node_modules/.vite/deps')
const metadataPath = path.join(cacheDir, '_metadata.json')

async function computeHash(deps: string[]): Promise<string> {
  const lockfile = await fs.readFile('pnpm-lock.yaml', 'utf-8').catch(() => '')
  return createHash('sha256')
    .update(lockfile)
    .update(JSON.stringify([...deps].sort()))
    .digest('hex')
    .slice(0, 8)
}

/** 预构建依赖，命中缓存时直接返回已有的元数据 */
export async function optimizeDeps(deps: string[]): Promise<DepsMetadata> {
  const hash = await computeHash(deps)

  try {
    const cached: DepsMetadata = JSON.parse(await fs.readFile(metadataPath, 'utf-8'))
    if (cached.hash === hash) return cached
  } catch {
    // 没有缓存，继续预构建
  }

  await fs.rm(cacheDir, { recursive: true, force: true })

  // 包名中的 / 不能作为文件名
  const toEntryName = (dep: string) => dep.replace(/\//g, '_')
  const input = Object.fromEntries(deps.map((dep) => [toEntryName(dep), dep]))

  // 所有依赖一起打包，公共依赖会被抽取为共享 chunk，保证单例
  const bundle = await rolldown({ input, platform: 'browser' })
  await bundle.write({
    dir: cacheDir,
    format: 'esm',
    entryFileNames: '[name].js',
    chunkFileNames: 'chunk-[hash].js',
  })
  await bundle.close()

  const metadata: DepsMetadata = {
    hash,
    deps: Object.fromEntries(deps.map((dep) => [dep, `${toEntryName(dep)}.js`])),
  }
  await fs.writeFile(metadataPath, JSON.stringify(metadata, null, 2))
  return metadata
}
```

**易错点**

- 每个依赖单独打包，导致 React 等共享依赖出现多份
- hash 没有包含依赖列表，新增依赖后不重新构建
- 没有清理旧的缓存目录

---

### H9. 写一份生产级 `vite.config.ts`

- **难度**: 中级
- **标签**: 配置
- **面试官看什么**: 配置的完整性与每一项的理由

**题干**

React + TS 中后台项目：别名、代理、环境变量、CDN、分包、source map、体积分析、兼容性。

**参考实现**

```ts
import { defineConfig, loadEnv } from 'vite'
import react from '@vitejs/plugin-react'
import { visualizer } from 'rollup-plugin-visualizer'

export default defineConfig(({ command, mode }) => {
  // 第三个参数为空字符串：配置文件中需要读取不带 VITE_ 前缀的变量（如代理目标）
  const env = loadEnv(mode, process.cwd(), '')
  const isBuild = command === 'build'

  return {
    base: isBuild ? env.CDN_BASE || '/' : '/',
    plugins: [
      react(),
      env.ANALYZE === 'true' && visualizer({ open: true, gzipSize: true }),
    ],
    resolve: {
      tsconfigPaths: true,
    },
    server: {
      port: 5173,
      proxy: {
        '/api': { target: env.API_TARGET, changeOrigin: true },
      },
      warmup: { clientFiles: ['./src/main.tsx', './src/layouts/**/*.tsx'] },
    },
    build: {
      // 上传监控平台后删除 .map 文件，不暴露源码
      sourcemap: 'hidden',
      // 计算 gzip 大小很耗时，CI 中关闭
      reportCompressedSize: false,
      chunkSizeWarningLimit: 800,
      rolldownOptions: {
        output: {
          codeSplitting: {
            groups: [
              // 框架几乎不变，单独拆分以获得长期缓存
              { name: 'react', test: /node_modules[\\/](react|react-dom|react-router|scheduler)[\\/]/, priority: 30 },
              { name: 'antd', test: /node_modules[\\/](antd|@ant-design|rc-[^\\/]+)[\\/]/, priority: 20 },
              { name: 'vendor', test: /node_modules/, priority: 10, minSize: 20_000 },
            ],
          },
        },
      },
    },
  }
})
```

**易错点**

- 配置文件中用 `import.meta.env` 读取 `.env`（读不到）
- 把所有依赖放进一个 vendor，任何依赖更新都导致缓存失效
- 生产 source map 设置为 `true`，公开了源码

---

### H10. 用 `import.meta.glob` 实现约定式路由

- **难度**: 中级
- **标签**: API, 路由
- **面试官看什么**: glob 的懒加载与 eager 用法、路径转换

**题干**

`src/pages` 下的文件自动生成路由：`pages/index.tsx` → `/`，`pages/user/[id].tsx` → `/user/:id`；页面懒加载；页面可以导出 `meta`（同步获取，用于菜单）。

**参考实现**

```tsx
import { lazy, type ComponentType, type LazyExoticComponent } from 'react'

/** 页面文件导出的元信息，用于生成菜单 */
export interface PageMeta {
  title?: string
  hideInMenu?: boolean
}

/** 约定式路由生成的路由项 */
export interface FileRoute {
  path: string
  Component: LazyExoticComponent<ComponentType>
  meta: PageMeta
}

// 组件懒加载：每个页面生成独立 chunk
const pages = import.meta.glob<{ default: ComponentType }>('/src/pages/**/*.tsx')
// meta 同步导入：菜单渲染需要立即拿到；只导入 meta，不会把组件打进主包
const metas = import.meta.glob<PageMeta>('/src/pages/**/*.tsx', {
  import: 'meta',
  eager: true,
})

function filePathToRoute(file: string): string {
  const route = file
    .replace(/^\/src\/pages/, '')
    .replace(/\.tsx$/, '')
    .replace(/\/index$/, '')
    .replace(/\[([^\]]+)\]/g, ':$1')
  return route || '/'
}

/** 根据 src/pages 目录结构生成路由表 */
export const routes: FileRoute[] = Object.entries(pages).map(([file, loader]) => ({
  path: filePathToRoute(file),
  Component: lazy(loader),
  meta: metas[file] ?? {},
}))
```

**易错点**

- glob 参数使用变量，无法静态分析
- meta 使用懒加载，菜单无法同步渲染；或 meta 使用 eager 导入整个模块，所有页面被打进主包
- `[id]` 没有转换为 `:id`

---

### H11. 处理发版后的 chunk 加载失败

- **难度**: 中级
- **标签**: 线上问题
- **面试官看什么**: vite:preloadError、防止无限刷新、版本检测

**题干**

发版后，停留在旧页面的用户切换路由时 chunk 加载失败。要求自动恢复，并且不能无限刷新。

**参考实现**

```ts
const RELOAD_KEY = 'chunk-error-reload-at'
// 同一会话中两次刷新的最小间隔，防止新版本资源本身有问题时无限刷新
const RELOAD_COOLDOWN = 10_000

/** 监听 Vite 预加载失败事件，必要时刷新页面获取最新版本 */
export function setupChunkErrorRecovery(): void {
  window.addEventListener('vite:preloadError', (event) => {
    const last = Number(sessionStorage.getItem(RELOAD_KEY) ?? 0)
    if (Date.now() - last < RELOAD_COOLDOWN) {
      // 刚刷新过仍然失败，交给错误边界展示提示，而不是继续刷新
      return
    }
    // 阻止错误抛出，改为刷新
    event.preventDefault()
    sessionStorage.setItem(RELOAD_KEY, String(Date.now()))
    window.location.reload()
  })
}

/** 定时检查是否有新版本，提示用户刷新 */
export function watchNewVersion(onNewVersion: () => void, interval = 5 * 60_000): void {
  const current = import.meta.env.VITE_APP_VERSION
  setInterval(async () => {
    try {
      // 加时间戳绕过缓存；version.json 由构建时生成
      const res = await fetch(`${import.meta.env.BASE_URL}version.json?t=${Date.now()}`)
      const { version } = await res.json()
      if (version !== current) onNewVersion()
    } catch {
      // 网络错误时忽略，下次再检查
    }
  }, interval)
}
```

**易错点**

- 没有冷却机制，导致无限刷新
- 只依赖 preloadError，没有处理不经过 `__vitePreload` 的加载
- 根本原因（删除旧资源）没有解决

---

### H12. 库模式配置：React 组件库

- **难度**: 中级
- **标签**: 库模式
- **面试官看什么**: external、多入口、保留模块、类型、package.json

**题干**

用 Vite 构建 React 组件库：只输出 ESM，支持按需引入，样式单独导出，生成类型声明。

**参考实现**

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import dts from 'vite-plugin-dts'
import pkg from './package.json'

const externals = [
  ...Object.keys(pkg.peerDependencies ?? {}),
  ...Object.keys(pkg.dependencies ?? {}),
]

export default defineConfig({
  plugins: [react(), dts({ include: ['src'], exclude: ['**/*.test.tsx'] })],
  build: {
    lib: {
      entry: 'src/index.ts',
      formats: ['es'],
    },
    rolldownOptions: {
      // 用正则匹配子路径（如 react/jsx-runtime），否则子路径会被打包进库
      external: externals.map((name) => new RegExp(`^${name}(/.*)?$`)),
      output: {
        // 保留模块结构，使用方可以按需 Tree Shaking
        preserveModules: true,
        preserveModulesRoot: 'src',
        entryFileNames: '[name].js',
      },
    },
    cssCodeSplit: true,
  },
})
```

```json
{
  "name": "@company/ui",
  "type": "module",
  "sideEffects": ["**/*.css"],
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./style.css": "./dist/style.css"
  },
  "peerDependencies": {
    "react": ">=18",
    "react-dom": ">=18"
  }
}
```

**易错点**

- 只 external `react`，`react/jsx-runtime` 被打包进库
- `sideEffects: false` 导致使用方导入的 CSS 被 Tree Shaking 删除
- `exports` 中 `types` 没有放在第一位，TS 无法正确解析
