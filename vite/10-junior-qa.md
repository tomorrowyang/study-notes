# Vite 初级 Q&A

> 目标：掌握 Vite 的日常使用：配置、命令、环境变量、静态资源、CSS、TS、代理、代码分割和常见报错。每题包含答案、口述模板和常见追问，边看边记。

---

## 一、核心概念

### Q1. Vite 是什么？解决了什么问题？

- **难度**: 初级
- **标签**: 概念

**答案**

Vite 是一个前端构建工具，分为两部分：

- **开发服务器**：基于浏览器原生 ESM，**不打包**，浏览器请求哪个模块就编译哪个模块，启动快，HMR 快
- **生产构建**：用打包器（Vite 8 起是 Rolldown，之前是 Rollup）打包，输出经过优化的静态资源

它解决的是传统打包器（webpack）在大项目中**开发启动慢、HMR 慢**的问题：webpack 启动前要先构建整个依赖图并打包，项目越大越慢；Vite 启动时只做依赖预构建，源码按需编译。

此外，Vite 提供了开箱即用的体验：TS、JSX、CSS Modules、预处理器、静态资源、环境变量等都无需额外配置。

**口述模板（30–60 秒）**

1. 一句话结论：Vite 是开发时 no-bundle、生产时打包的构建工具。
2. 关键机制：开发时利用浏览器原生 ESM 按需编译，生产时用 Rolldown 打包优化。
3. 对比或注意点：相比 webpack，启动和 HMR 速度基本不受项目规模影响。
4. 可追问时补充：Vite 8 起开发和生产统一使用 Rolldown，减少了两边行为不一致。

**常见追问**

- 生产为什么还要打包？→ 未打包的 ESM 在生产环境请求太多，网络瀑布严重，也无法做 Tree Shaking、代码分割、压缩等优化。
- Vite 只能用于 Vue 吗？→ 不是，框架无关，React / Vue / Svelte / Solid 等都有官方或社区插件。

---

### Q2. 为什么 Vite 开发服务器启动快？

- **难度**: 初级
- **标签**: 原理, 性能

**答案**

1. **不打包源码**：启动时不构建整个依赖图，浏览器请求到某个模块时才编译它
2. **依赖预构建**：第三方依赖变化少，启动时用 Rolldown（Vite 8 之前用 esbuild）一次性预构建并缓存，之后直接复用
3. **原生工具链**：转换用 Oxc（Rust），打包用 Rolldown（Rust），单个文件编译极快
4. **HTTP 缓存**：依赖模块使用强缓存（`max-age=31536000,immutable`），源码使用协商缓存（304）

因此启动时间主要取决于首屏用到的模块，而不是整个项目的大小。

**口述模板（30–60 秒）**

1. 一句话结论：Vite 启动快是因为不打包源码，只按需编译。
2. 关键机制：依赖做一次预构建并缓存，源码由浏览器按 ESM 请求，请求到才编译。
3. 对比或注意点：webpack 启动前要把所有模块打包好，所以项目越大启动越慢。
4. 可追问时补充：模块数量极大时，浏览器请求过多也会变慢，Vite 8.1 的实验性 Bundled Dev Mode 就是为了解决这个问题。

**常见追问**

- 冷启动和页面首次加载是一回事吗？→ 不是。服务器启动快，但首次打开页面时浏览器需要请求大量模块，大项目可能仍然慢。
- 第二次启动为什么更快？→ 预构建结果缓存在 `node_modules/.vite`，依赖未变化时直接复用。

---

### Q3. 为什么说 index.html 是 Vite 的入口？

- **难度**: 初级
- **标签**: 概念

**答案**

在 Vite 中，`index.html` 放在项目根目录，是**源码的一部分**，也是模块图的入口：

```html
<!doctype html>
<html>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

- 开发时，Vite 处理 `index.html`：注入 HMR 客户端（`/@vite/client`），执行插件的 `transformIndexHtml`
- 构建时，Vite 从 HTML 中的 `<script type="module">` 和 `<link>` 找到入口，打包后把引用替换为带 hash 的产物路径

与 webpack 不同，webpack 的入口是 JS，HTML 由 `html-webpack-plugin` 生成。

**口述模板（30–60 秒）**

1. 一句话结论：Vite 把 index.html 当作入口和源码的一部分。
2. 关键机制：从 HTML 的 module script 开始构建模块图，开发时注入 HMR 客户端，构建时替换为产物路径。
3. 对比或注意点：webpack 以 JS 为入口，HTML 由插件生成。
4. 可追问时补充：多页面应用就是在 `build.rolldownOptions.input` 中配置多个 HTML 入口。

**常见追问**

- `<script>` 不加 `type="module"` 会怎样？→ Vite 不会处理它，作为普通脚本原样保留。
- HTML 中能用环境变量吗？→ 可以，使用 `%VITE_APP_TITLE%` 语法。

---

### Q4. `vite.config.ts` 常用配置有哪些？

- **难度**: 初级
- **标签**: 配置

**答案**

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  root: process.cwd(),          // 项目根目录（index.html 所在）
  base: '/',                    // 部署的公共基础路径
  plugins: [react()],
  resolve: {
    alias: { '@': '/src' },
  },
  server: {
    port: 5173,
    proxy: { '/api': 'http://localhost:3000' },
  },
  build: {
    outDir: 'dist',
    sourcemap: false,
    target: 'baseline-widely-available',
  },
})
```

`defineConfig` 只用于提供类型提示。配置也可以是函数，根据命令和 mode 返回不同配置：

```ts
export default defineConfig(({ command, mode }) => ({
  base: command === 'build' ? '/app/' : '/',
}))
```

**口述模板（30–60 秒）**

1. 一句话结论：常用配置是 base、plugins、resolve、server、build 这几块。
2. 关键机制：配置导出对象或函数，函数形式可以根据 command 和 mode 动态返回。
3. 对比或注意点：Vite 默认配置已经覆盖大部分需求，配置量比 webpack 少很多。
4. 可追问时补充：底层打包选项通过 `build.rolldownOptions` 透传给 Rolldown。

**常见追问**

- 配置文件中能读取 `.env` 吗？→ 不能直接通过 `import.meta.env` 读取，需要用 `loadEnv(mode, process.cwd())`。
- `root` 和 `base` 的区别？→ `root` 是本地项目目录，`base` 是部署后的 URL 前缀。

---

### Q5. `vite`、`vite build`、`vite preview` 分别做什么？

- **难度**: 初级
- **标签**: 命令

**答案**

| 命令 | 作用 | 默认 mode |
|------|------|-----------|
| `vite` / `vite dev` | 启动开发服务器 | `development` |
| `vite build` | 生产构建，输出到 `dist` | `production` |
| `vite preview` | 本地启动静态服务器预览 `dist` | `production` |

`preview` 只用来本地验证构建产物，**不是生产服务器**，正式部署应使用 Nginx、CDN 或对象存储。

常用参数：`--mode staging`、`--host`、`--port`、`--open`、`--force`（强制重新预构建）、`--debug`、`--profile`。

**口述模板（30–60 秒）**

1. 一句话结论：dev 开发、build 构建、preview 预览产物。
2. 关键机制：dev 和 build 默认 mode 不同，影响加载哪个 `.env` 文件。
3. 对比或注意点：preview 不能用于生产部署。
4. 可追问时补充：`--force` 可以在依赖缓存出问题时强制重新预构建。

**常见追问**

- mode 和 `NODE_ENV` 的关系？→ 两者是不同的概念。`--mode staging` 构建时 `NODE_ENV` 仍然是 production，除非在 `.env.staging` 中显式设置 `NODE_ENV`。

---

### Q6. Vite 的环境变量怎么用？

- **难度**: 初级
- **标签**: 环境变量

**答案**

**文件加载顺序**（后者优先级更高）：

```text
.env                # 所有情况
.env.local          # 所有情况，git 忽略
.env.[mode]         # 指定 mode
.env.[mode].local   # 指定 mode，git 忽略
```

**使用方式**：

```ts
console.log(import.meta.env.VITE_API_BASE)
console.log(import.meta.env.MODE)  // 当前 mode
console.log(import.meta.env.DEV)   // 是否开发环境
console.log(import.meta.env.PROD)
console.log(import.meta.env.BASE_URL)
```

**关键规则**：

- 只有 **`VITE_` 前缀**的变量会暴露给客户端代码（可通过 `envPrefix` 修改），防止误泄露服务器密钥
- 构建时**静态替换**为字面量，所以不要在客户端变量中放密钥，它们会出现在产物中
- TS 类型提示：在 `vite-env.d.ts` 中扩展 `ImportMetaEnv`

```ts
interface ImportMetaEnv {
  readonly VITE_API_BASE: string
}
```

**口述模板（30–60 秒）**

1. 一句话结论：Vite 用 `.env` 文件加 `import.meta.env` 管理环境变量。
2. 关键机制：按 mode 加载对应文件，只有 `VITE_` 前缀会暴露到客户端，构建时静态替换。
3. 对比或注意点：客户端变量最终会出现在产物中，不能放密钥。
4. 可追问时补充：配置文件中需要用 `loadEnv` 读取。

**常见追问**

- 想在运行时切换环境变量（一次构建，多环境部署）怎么办？→ 不能依赖构建时替换，可以在 HTML 中注入 `window.__CONFIG__` 或运行时请求配置接口。
- 和 webpack 的 `process.env` 有什么区别？→ Vite 默认不提供 `process.env`，统一使用 `import.meta.env`。

---

### Q7. Vite 中怎么处理静态资源？

- **难度**: 初级
- **标签**: 静态资源

**答案**

**1. 导入资源得到 URL**

```ts
import logoUrl from './logo.png'          // 构建后为 /assets/logo-abc123.png
import iconRaw from './icon.svg?raw'      // 字符串内容
import workerUrl from './worker.js?url'   // 显式获取 URL
import inlineSvg from './a.svg?inline'    // 内联为 base64
import MyWorker from './worker.ts?worker' // Web Worker 构造器
```

- 小于 `build.assetsInlineLimit`（默认 4KB）的资源会内联为 base64
- 动态拼接路径用 `new URL('./img/${name}.png', import.meta.url).href`

**2. `public` 目录**

- 原样复制到产物根目录，不经处理，没有 hash
- 通过绝对路径引用：`/favicon.ico`
- 适用于 `robots.txt`、必须保留原文件名的资源

**口述模板（30–60 秒）**

1. 一句话结论：导入的资源会被处理并加 hash，public 目录中的资源原样复制。
2. 关键机制：导入资源得到 URL，小资源内联；通过 `?raw`、`?url`、`?inline`、`?worker` 等后缀控制导入方式。
3. 对比或注意点：public 中的资源没有 hash，需要注意缓存。
4. 可追问时补充：动态路径用 `new URL(..., import.meta.url)`，Vite 构建时能识别并处理。

**常见追问**

- `new URL` 中的路径能完全动态吗？→ 不能，需要是带静态部分的模板字符串，Vite 才能分析出可能的文件。

---

### Q8. Vite 中 CSS 怎么处理？

- **难度**: 初级
- **标签**: CSS

**答案**

- **直接导入**：`import './style.css'`，开发时注入 `<style>` 并支持 HMR，构建时抽取为独立 CSS 文件
- **CSS Modules**：文件名为 `*.module.css`，`import styles from './a.module.css'`
- **预处理器**：安装 `sass` / `less` / `stylus` 即可，无需配置 loader
- **PostCSS**：自动读取 `postcss.config.js`
- **Lightning CSS**：Vite 8 默认用它压缩 CSS；也可以设置 `css.transformer: 'lightningcss'` 替代 PostCSS 做转换
- **`?inline`**：`import css from './a.css?inline'` 得到字符串，不注入页面

全局变量注入：

```ts
css: {
  preprocessorOptions: {
    scss: { additionalData: `@use "@/styles/vars" as *;` },
  },
}
```

**口述模板（30–60 秒）**

1. 一句话结论：Vite 内置 CSS、CSS Modules、预处理器和 PostCSS 支持。
2. 关键机制：开发时 CSS 作为 JS 模块注入 style 标签，构建时抽取并按 chunk 分割。
3. 对比或注意点：不需要像 webpack 那样配置 css-loader、style-loader。
4. 可追问时补充：Vite 8 默认用 Lightning CSS 压缩。

**常见追问**

- 异步 chunk 的 CSS 怎么加载？→ 构建时 CSS 按 chunk 分割，加载异步 chunk 前会先加载对应的 CSS。

---

### Q9. Vite 中 TypeScript 是怎么处理的？会做类型检查吗？

- **难度**: 初级
- **标签**: TypeScript

**答案**

Vite **只转译、不做类型检查**：用 Oxc（Vite 8 之前用 esbuild）去掉类型，单文件编译极快。

所以：

- 需要在 `tsconfig` 中开启 `isolatedModules: true`，因为单文件转译无法处理 `const enum`、仅导出类型的重导出等依赖跨文件类型信息的语法
- 类型检查需要另外做：
  - 构建脚本：`"build": "tsc --noEmit && vite build"`（项目引用时用 `tsc -b`）
  - 开发时：IDE，或使用 `vite-plugin-checker` 在浏览器显示类型错误

**口述模板（30–60 秒）**

1. 一句话结论：Vite 只转译 TS，不做类型检查。
2. 关键机制：用 Oxc 单文件剥离类型，所以需要开启 `isolatedModules`。
3. 对比或注意点：类型检查交给 tsc 或 IDE，可以和构建并行。
4. 可追问时补充：Vite 8 内置支持 `emitDecoratorMetadata`，但 Oxc 不降级原生装饰器语法。

**常见追问**

- 为什么不在构建中检查类型？→ 类型检查需要整个项目的类型信息，很慢，会拖慢开发反馈；分离后转译快，检查可以并行。

---

### Q10. 路径别名怎么配置？

- **难度**: 初级
- **标签**: 配置

**答案**

**方式一：`resolve.alias`**

```ts
import { fileURLToPath, URL } from 'node:url'

export default defineConfig({
  resolve: {
    alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) },
  },
})
```

同时在 `tsconfig.json` 中配置 `paths`，让 TS 能识别。

**方式二（Vite 8+）：直接读取 tsconfig paths**

```ts
export default defineConfig({
  resolve: { tsconfigPaths: true },
})
```

不需要再安装 `vite-tsconfig-paths` 插件，只在 `tsconfig` 中维护一份。

**口述模板（30–60 秒）**

1. 一句话结论：用 `resolve.alias` 配置，并同步 `tsconfig` 的 `paths`。
2. 关键机制：alias 负责 Vite 的模块解析，`paths` 负责 TS 的类型解析。
3. 对比或注意点：两处要保持一致，否则会出现能运行但报类型错误的情况。
4. 可追问时补充：Vite 8 可以开启 `resolve.tsconfigPaths`，只维护一份配置。

**常见追问**

- alias 为什么要用绝对路径？→ 相对路径会相对于导入文件解析，结果不确定。

---

### Q11. 开发时怎么配置接口代理？

- **难度**: 初级
- **标签**: 开发服务器

**答案**

```ts
server: {
  proxy: {
    '/api': {
      target: 'http://localhost:3000',
      changeOrigin: true,
      rewrite: (path) => path.replace(/^\/api/, ''),
    },
    '/socket.io': {
      target: 'ws://localhost:3000',
      ws: true,
    },
  },
},
```

- `changeOrigin`：把请求头中的 `Host` 改为目标地址，后端有虚拟主机或校验 Host 时需要
- `rewrite`：重写路径
- `ws: true`：代理 WebSocket

代理只在开发服务器（以及 `preview`）中生效，生产环境需要 Nginx 或网关转发。

**口述模板（30–60 秒）**

1. 一句话结论：通过 `server.proxy` 把请求转发到后端，解决开发时跨域。
2. 关键机制：基于 http-proxy，支持路径重写、改 Host、代理 WebSocket。
3. 对比或注意点：只在开发时生效，生产需要 Nginx 或网关。
4. 可追问时补充：也可以用 `configure` 回调拿到 proxy 实例，监听事件打印日志。

**常见追问**

- 为什么代理能解决跨域？→ 浏览器请求的是同源的开发服务器，服务器之间的请求没有同源限制。

---

### Q12. Vite 的 HMR 怎么用？需要自己写代码吗？

- **难度**: 初级
- **标签**: HMR

**答案**

大多数情况下**不需要**：

- CSS 文件天然支持 HMR
- 框架插件自动注入 HMR 代码：`@vitejs/plugin-react`（React Fast Refresh）、`@vitejs/plugin-vue`（Vue 组件热更新）

只有非框架代码需要手动处理时，才用 `import.meta.hot`：

```ts
if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    // 用新模块更新状态
  })
  import.meta.hot.dispose(() => {
    // 清理副作用：定时器、事件监听
  })
}
```

`if (import.meta.hot)` 在生产构建中会被移除。

**口述模板（30–60 秒）**

1. 一句话结论：框架插件已经处理好 HMR，通常不需要手写。
2. 关键机制：底层是 `import.meta.hot` API，模块接受更新后浏览器重新请求该模块。
3. 对比或注意点：没有模块接受更新时，会整页刷新。
4. 可追问时补充：dispose 用来清理副作用，否则多次更新会累积定时器和监听器。

**常见追问**

- 改了一个工具函数为什么整页刷新？→ 它的导入链上没有模块接受更新（没有 HMR 边界）。

---

### Q13. React 和 Vue 项目分别用哪些官方插件？

- **难度**: 初级
- **标签**: 插件, 框架

**答案**

| 框架 | 插件 | 说明 |
|------|------|------|
| React | `@vitejs/plugin-react` | v6 起使用 Oxc 处理 JSX 和 React Refresh，不再依赖 Babel |
| React | `@vitejs/plugin-react-swc` | 用 SWC 处理，适合需要 SWC 插件的场景 |
| Vue 3 | `@vitejs/plugin-vue` | 处理 SFC |
| Vue JSX | `@vitejs/plugin-vue-jsx` | |
| 旧浏览器 | `@vitejs/plugin-legacy` | 生成兼容旧浏览器的产物和 polyfill |

React Compiler：`plugin-react` v6 移除 Babel 后，需要配合 `@rolldown/plugin-babel` 使用 `reactCompilerPreset`（具体写法以官方文档为准）。

**口述模板（30–60 秒）**

1. 一句话结论：React 用 plugin-react，Vue 用 plugin-vue。
2. 关键机制：插件负责 JSX / SFC 转换和 HMR 集成。
3. 对比或注意点：plugin-react v6 已不依赖 Babel，需要 Babel 插件时要另外配置。
4. 可追问时补充：React Compiler 需要单独接入 Babel 插件。

**常见追问**

- 为什么移除 Babel？→ Babel 是 JS 实现，慢；Oxc 是 Rust 实现，速度快很多。

---

### Q14. `import.meta.glob` 是什么？

- **难度**: 初级
- **标签**: API

**答案**

Vite 提供的批量导入 API，**构建时**根据 glob 展开为具体的导入语句：

```ts
// 懒加载：值是返回 Promise 的函数
const pages = import.meta.glob('./pages/*.tsx')
// 结果类似 { './pages/a.tsx': () => import('./pages/a.tsx') }

// 立即导入
const modules = import.meta.glob('./locales/*.json', { eager: true })

// 只导入某个导出
const metas = import.meta.glob('./pages/*.tsx', { import: 'meta', eager: true })

// 以字符串或 URL 形式导入
const raws = import.meta.glob('./docs/*.md', { query: '?raw', import: 'default' })
```

常见用途：约定式路由、自动注册组件、批量加载国际化文件。

**口述模板（30–60 秒）**

1. 一句话结论：`import.meta.glob` 用 glob 批量导入模块，构建时展开。
2. 关键机制：默认生成懒加载函数，`eager` 时变成静态导入。
3. 对比或注意点：参数必须是字面量，不能是变量。
4. 可追问时补充：对应 webpack 的 `require.context`。

**常见追问**

- 为什么参数必须是字面量？→ 需要在构建时静态分析，变量无法分析。

---

### Q15. Vite 中怎么做代码分割？

- **难度**: 初级
- **标签**: 构建, 性能

**答案**

1. **动态导入自动分割**：`import('./Page')` 会生成独立 chunk

```tsx
const Settings = lazy(() => import('./pages/Settings'))
```

2. **手动分组**：Vite 8 使用 Rolldown 的 `codeSplitting`（取代 `manualChunks`）

```ts
build: {
  rolldownOptions: {
    output: {
      codeSplitting: {
        groups: [
          { name: 'react', test: /node_modules[\\/](react|react-dom)/ },
          { name: 'vendor', test: /node_modules/ },
        ],
      },
    },
  },
},
```

3. **CSS 代码分割**：异步 chunk 引入的 CSS 会单独抽取，随 chunk 加载

**口述模板（30–60 秒）**

1. 一句话结论：动态 import 自动分割，手动分组用 codeSplitting。
2. 关键机制：Rolldown 根据动态导入边界生成 chunk，groups 按规则把模块分到指定 chunk。
3. 对比或注意点：Vite 8 中 `manualChunks` 的对象形式已移除，函数形式已废弃。
4. 可追问时补充：拆得太细会增加请求数，需要结合 HTTP/2 和缓存策略权衡。

**常见追问**

- vendor 全部放一个包好吗？→ 不一定，任何依赖变化都会让整个 vendor 缓存失效。通常把稳定的大框架单独拆出来。

---

### Q16. 部署到子路径时需要注意什么？

- **难度**: 初级
- **标签**: 部署

**答案**

- 设置 `base: '/my-app/'`，构建产物中的资源路径会带上前缀
- 前端路由也要设置基础路径：`createBrowserRouter(routes, { basename: import.meta.env.BASE_URL })`
- `public` 目录资源要用 `import.meta.env.BASE_URL + 'favicon.ico'` 拼接
- 如果部署路径不确定，可以设置 `base: './'`（相对路径），但 history 路由下的深层路径可能有问题

服务器配置：history 路由需要把未命中的路径回退到 `index.html`。

**口述模板（30–60 秒）**

1. 一句话结论：部署到子路径要设置 base，并同步路由的 basename。
2. 关键机制：base 影响所有资源的引用路径，运行时用 `import.meta.env.BASE_URL` 获取。
3. 对比或注意点：相对路径 base 适合静态嵌入，不适合 history 路由。
4. 可追问时补充：资源放 CDN 时可以用 `experimental.renderBuiltUrl` 精细控制不同资源的 URL。

**常见追问**

- 部署后页面空白、资源 404？→ 优先检查 base 是否与实际部署路径一致。

---

### Q17. Vite 构建产物兼容哪些浏览器？怎么兼容旧浏览器？

- **难度**: 初级
- **标签**: 兼容性

**答案**

- 默认 `build.target: 'baseline-widely-available'`，Vite 8 中对应 Chrome / Edge 111、Firefox 114、Safari 16.4
- Vite **只做语法转换，不注入 polyfill**
- 需要兼容旧浏览器时：
  - 调低 `build.target`，例如 `'es2020'`（只能降级语法）
  - 使用 `@vitejs/plugin-legacy`：额外生成一套兼容产物和 polyfill，通过 `<script nomodule>` 给旧浏览器加载

注意：Vite 8 中 plugin-legacy 不支持转换到 ES5 及以下（以官方迁移指南为准），需要兼容非常旧的浏览器时要谨慎评估。

**口述模板（30–60 秒）**

1. 一句话结论：默认面向现代浏览器，只降级语法，不注入 polyfill。
2. 关键机制：通过 `build.target` 控制语法降级，通过 plugin-legacy 生成兼容产物。
3. 对比或注意点：plugin-legacy 会生成两套产物，构建时间和产物体积增加。
4. 可追问时补充：Vite 8 的默认 target 比 Vite 7 更高。

**常见追问**

- 开发环境兼容旧浏览器吗？→ 不兼容，开发时依赖原生 ESM 和较新的语法。

---

### Q18. Vite 怎么保证静态资源的缓存？

- **难度**: 初级
- **标签**: 缓存, 部署

**答案**

- 产物文件名默认带内容 hash：`assets/index-B2xkO1aJ.js`，内容不变 hash 不变
- 部署策略：
  - `index.html`：`Cache-Control: no-cache`，每次协商
  - `assets/*`：`Cache-Control: max-age=31536000, immutable`，长期缓存
- 发布时**先上传新资源，再更新 HTML**；旧资源保留一段时间，避免旧页面加载异步 chunk 时 404

**口述模板（30–60 秒）**

1. 一句话结论：资源带内容 hash 长期缓存，HTML 不缓存。
2. 关键机制：内容变化 hash 才变化，用户只需下载变化的文件。
3. 对比或注意点：发布时旧资源不能立即删除。
4. 可追问时补充：旧资源被删除后可能出现 chunk 加载失败，可以监听 `vite:preloadError` 兜底。

**常见追问**

- 修改一个文件，为什么很多文件 hash 都变了？→ 文件 hash 变化后，引用它的 chunk 内容也会变化，引起连锁变化。

---

### Q19. Vite 怎么使用 Web Worker 和 WebAssembly？

- **难度**: 初级
- **标签**: Worker, Wasm

**答案**

**Web Worker**

```ts
// 方式一：标准写法（推荐）
const worker = new Worker(new URL('./worker.ts', import.meta.url), { type: 'module' })

// 方式二：Vite 后缀
import MyWorker from './worker.ts?worker'
const worker2 = new MyWorker()
```

Worker 产物格式通过 `worker.format` 配置（`'es'` 或 `'iife'`）。

**WebAssembly**

```ts
import init from './example.wasm?init'

const instance = await init({ imports: {} })
```

Vite 8 起 `?init` 也支持 SSR 环境。

**口述模板（30–60 秒）**

1. 一句话结论：Worker 用 `new URL(..., import.meta.url)` 或 `?worker`，Wasm 用 `?init`。
2. 关键机制：Vite 会把 Worker 单独打包成一个 chunk。
3. 对比或注意点：推荐标准写法，可移植性更好。
4. 可追问时补充：Worker 打包配置用 `worker.rolldownOptions`（Vite 8）。

**常见追问**

- Worker 能用 TS 吗？→ 可以，Vite 会编译 Worker 文件。

---

### Q20. `public` 目录和 `src/assets` 目录有什么区别？

- **难度**: 初级
- **标签**: 静态资源

**答案**

| 对比项 | `public/` | `src/assets/` |
|--------|-----------|---------------|
| 是否处理 | 不处理，原样复制 | 经过构建处理 |
| 文件名 | 保持原名 | 带 hash |
| 引用方式 | 绝对路径 `/logo.png` | `import` 导入或相对路径 |
| 未使用的文件 | 也会复制 | 不引用就不会打包 |
| 适合 | `robots.txt`、favicon、第三方静态脚本 | 业务图片、字体、图标 |

**口述模板（30–60 秒）**

1. 一句话结论：public 原样复制，assets 经过构建处理。
2. 关键机制：assets 中的资源带 hash，可以长期缓存，小文件还能内联。
3. 对比或注意点：public 中的资源没有 hash，更新后可能被缓存。
4. 可追问时补充：public 中的文件不要在 JS 中 import。

**常见追问**

- 代码中能 import public 下的文件吗？→ 不应该，直接用绝对 URL 引用。

---

## 二、常见场景与问题

### Q21. 依赖预构建是什么？为什么需要？

- **难度**: 初级
- **标签**: 预构建

**答案**

Vite 首次启动时，会把 `node_modules` 中用到的依赖预先打包，结果放在 `node_modules/.vite/deps`。原因有两个：

1. **兼容 CJS / UMD**：浏览器只认识 ESM，很多包仍然是 CommonJS，需要转换成 ESM
2. **减少请求**：例如 `lodash-es` 内部有 600 多个模块，不预构建的话，浏览器要发 600 多个请求；预构建后合并为一个模块

缓存失效条件：lockfile、`vite.config` 中的相关配置、`patches` 目录等变化，或者使用 `--force`。

**口述模板（30–60 秒）**

1. 一句话结论：依赖预构建把第三方依赖提前打包成 ESM。
2. 关键机制：解决 CJS 兼容问题和请求过多问题，结果缓存在 `node_modules/.vite`。
3. 对比或注意点：只针对依赖，源码不预构建。
4. 可追问时补充：Vite 8 起预构建也用 Rolldown，之前用 esbuild。

**常见追问**

- 修改了 node_modules 中的代码不生效？→ 预构建有缓存，需要 `--force` 或删除 `node_modules/.vite`。

---

### Q22. Vite 插件是什么？常用的有哪些？

- **难度**: 初级
- **标签**: 插件

**答案**

Vite 插件是一个带有钩子方法的对象，**兼容 Rollup / Rolldown 插件接口**，并扩展了 Vite 特有的钩子：

```ts
export default function myPlugin(): Plugin {
  return {
    name: 'my-plugin',
    transform(code, id) {
      if (!id.endsWith('.txt')) return
      return `export default ${JSON.stringify(code)}`
    },
  }
}
```

常用插件：

- 框架：`@vitejs/plugin-react`、`@vitejs/plugin-vue`
- 兼容：`@vitejs/plugin-legacy`
- 分析：`rollup-plugin-visualizer`、`vite-plugin-inspect`（Vite 8 也内置了 Devtools）
- 类型检查：`vite-plugin-checker`
- 自动导入：`unplugin-auto-import`、`unplugin-vue-components`
- SVG：`vite-plugin-svgr`
- 库类型声明：`vite-plugin-dts`

Vite 8 起官方提供插件目录 registry.vite.dev。

**口述模板（30–60 秒）**

1. 一句话结论：Vite 插件是带钩子的对象，兼容 Rollup 插件接口。
2. 关键机制：通过 resolveId / load / transform 等钩子介入模块处理，还有 Vite 特有的服务器和 HTML 钩子。
3. 对比或注意点：webpack 区分 loader 和 plugin，Vite 统一为插件。
4. 可追问时补充：unplugin 可以写一份代码同时支持 Vite、webpack、Rspack。

**常见追问**

- Rollup 插件能直接在 Vite 中用吗？→ 大多数可以，前提是没有依赖 Vite 不支持的钩子。

---

### Q23. 怎么配置多页面应用？

- **难度**: 初级
- **标签**: 配置

**答案**

```ts
import { resolve } from 'node:path'

export default defineConfig({
  build: {
    rolldownOptions: {
      input: {
        main: resolve(__dirname, 'index.html'),
        admin: resolve(__dirname, 'admin/index.html'),
      },
    },
  },
})
```

开发时直接访问 `/admin/index.html` 即可，不需要额外配置；构建时每个 HTML 是一个入口，公共依赖会自动抽取为共享 chunk。

**口述模板（30–60 秒）**

1. 一句话结论：在构建配置的 input 中配置多个 HTML 入口。
2. 关键机制：每个 HTML 是一个入口，Rolldown 自动抽取公共模块。
3. 对比或注意点：开发时按目录访问，构建时需要显式配置。
4. 可追问时补充：Vite 7 及之前写在 `build.rollupOptions.input`。

**常见追问**

- 页面很多时怎么办？→ 用 glob 扫描目录生成 input 对象。

---

### Q24. 什么是库模式？

- **难度**: 初级
- **标签**: 库模式

**答案**

库模式用于打包 npm 包或组件库，而不是应用：

```ts
export default defineConfig({
  build: {
    lib: {
      entry: 'src/index.ts',
      name: 'MyLib',              // UMD / IIFE 的全局变量名
      formats: ['es', 'cjs'],
      fileName: (format) => `my-lib.${format === 'es' ? 'mjs' : 'cjs'}`,
    },
    rolldownOptions: {
      external: ['react', 'react-dom', 'react/jsx-runtime'],
    },
  },
})
```

- 必须把 `peerDependencies` 配置为 external，否则 React 会被打进库中，导致多实例
- 类型声明需要 `vite-plugin-dts` 或单独执行 `tsc`
- 纯 TS 工具库可以考虑 `tsdown`（基于 Rolldown 的库打包工具）

**口述模板（30–60 秒）**

1. 一句话结论：库模式用 `build.lib` 配置入口和输出格式。
2. 关键机制：输出 ES / CJS 等格式，外部依赖需要 external。
3. 对比或注意点：peer 依赖不 external 会导致多实例问题。
4. 可追问时补充：类型声明需要额外的插件或 tsc。

**常见追问**

- 库模式下 CSS 怎么处理？→ 默认抽取为单独的 CSS 文件，使用方需要手动导入。

---

### Q25. Vite 中 source map 怎么配置？

- **难度**: 初级
- **标签**: 调试

**答案**

- 开发时默认启用，无需配置
- 生产构建：`build.sourcemap`
  - `false`（默认）：不生成
  - `true`：生成独立 `.map` 文件并在产物中添加注释
  - `'hidden'`：生成 `.map` 文件但不添加注释，适合上传到 Sentry 等平台，不暴露给用户
  - `'inline'`：内联到产物中

生产推荐 `'hidden'`：上传 source map 到监控平台，然后从部署产物中删除 `.map` 文件。

**口述模板（30–60 秒）**

1. 一句话结论：开发默认开启，生产通过 `build.sourcemap` 配置。
2. 关键机制：hidden 生成文件但不引用。
3. 对比或注意点：公开 source map 会暴露源码。
4. 可追问时补充：插件转换代码时应该返回 map，否则 source map 会断链。

**常见追问**

- 开启 source map 对构建速度影响大吗？→ 有一定影响，Rolldown 下影响较小。

---

### Q26. HTML 中怎么使用环境变量？

- **难度**: 初级
- **标签**: 环境变量

**答案**

```html
<title>%VITE_APP_TITLE%</title>
```

Vite 会把 `%变量名%` 替换为 `import.meta.env` 中对应的值；不存在的变量保持原样。

更复杂的 HTML 处理（注入脚本、根据环境插入标签）需要用插件的 `transformIndexHtml` 钩子。

**口述模板（30–60 秒）**

1. 一句话结论：HTML 中用 `%VITE_XXX%` 语法引用环境变量。
2. 关键机制：处理 HTML 时做字符串替换。
3. 对比或注意点：复杂逻辑用 transformIndexHtml。
4. 可追问时补充：同样只能使用暴露给客户端的变量。

**常见追问**

- 能写条件判断吗？→ 不能，需要使用插件。

---

### Q27. Vite 和 webpack 有什么区别？

- **难度**: 初级
- **标签**: 对比

**答案**

| 对比项 | Vite | webpack |
|--------|------|---------|
| 开发模式 | 不打包，原生 ESM 按需编译 | 先打包再提供服务 |
| 启动速度 | 快，基本与项目规模无关 | 项目越大越慢 |
| HMR | 只重新编译变化的模块 | 需要重新构建受影响的 chunk |
| 生产构建 | Rolldown（Vite 8） | webpack 自身 |
| 配置 | 开箱即用，配置少 | 灵活，但配置复杂 |
| 扩展 | 插件（兼容 Rollup） | loader + plugin |
| 生态 | 新项目默认选择 | 存量项目多，生态最全 |
| 模块联邦 | 需要社区插件 | 原生支持 |

详细原理对比见 [webpack 初级 Q30](../webpack/10-junior-qa.md)。

**口述模板（30–60 秒）**

1. 一句话结论：Vite 开发不打包，webpack 开发要先打包。
2. 关键机制：Vite 利用浏览器 ESM 按需编译，所以启动和 HMR 快。
3. 对比或注意点：webpack 更灵活、生态更成熟，存量项目多。
4. 可追问时补充：Vite 8 后开发和生产统一使用 Rolldown，一致性也更好了。

**常见追问**

- 什么情况下不建议用 Vite？→ 深度依赖 webpack 特有能力（自定义 loader 链、模块联邦等）的存量项目，迁移成本高，可以考虑 Rspack。

---

### Q28. Vitest 和 Vite 是什么关系？

- **难度**: 初级
- **标签**: 测试, 生态

**答案**

Vitest 是基于 Vite 的测试框架：

- **复用 Vite 配置和插件**：alias、插件、TS / JSX 转换和应用一致，不需要像 Jest 那样单独配置 babel-jest、moduleNameMapper
- **原生 ESM**、速度快，watch 模式利用模块图只重跑受影响的测试
- API 与 Jest 基本兼容（`describe` / `it` / `expect` / `vi.fn`）
- 支持浏览器模式、组件测试

```ts
// vite.config.ts
/// <reference types="vitest/config" />
export default defineConfig({
  test: { environment: 'jsdom' },
})
```

**口述模板（30–60 秒）**

1. 一句话结论：Vitest 是基于 Vite 的测试框架。
2. 关键机制：复用 Vite 的配置、插件和转换管线。
3. 对比或注意点：相比 Jest，不需要重复配置转换规则，ESM 支持更好。
4. 可追问时补充：watch 模式基于模块图只重跑相关测试。

**常见追问**

- 从 Jest 迁移难吗？→ API 基本兼容，主要改 `jest.fn` 为 `vi.fn` 以及 mock 相关写法。

---

### Q29. 常见报错：`require is not defined`、`process is not defined` 怎么处理？

- **难度**: 初级
- **标签**: 排错

**答案**

**`require is not defined`**

- 原因：浏览器端代码中使用了 `require`，Vite 只认 ESM
- 处理：改为 `import`；如果是第三方包，检查它是否被正确预构建（`optimizeDeps.include`）

**`process is not defined`**

- 原因：代码中使用了 `process.env.XXX`，这是 webpack 时代的写法，Vite 不注入 `process`
- 处理：改为 `import.meta.env.VITE_XXX`；第三方库使用的话，通过 `define` 替换：

```ts
define: {
  'process.env.NODE_ENV': JSON.stringify('production'),
}
```

**`Failed to resolve import`**

- 原因：路径错误、alias 未配置或依赖未安装
- 处理：检查路径大小写、alias 配置和 `node_modules`

**口述模板（30–60 秒）**

1. 一句话结论：这两个报错大多来自 webpack 时代的 CJS / Node 写法。
2. 关键机制：Vite 浏览器端只认 ESM，也不注入 process。
3. 对比或注意点：源码改成 ESM 和 `import.meta.env`，第三方库用 define 替换。
4. 可追问时补充：define 是字符串替换，值要用 `JSON.stringify` 包裹。

**常见追问**

- 为什么 define 的值要用 JSON.stringify？→ define 是原样替换代码文本，不包裹的话会被当作标识符。

---

### Q30. Vite 8 带来了哪些主要变化？

- **难度**: 初级
- **标签**: 版本

**答案**

Vite 8（2026-03）最大的变化是**统一使用 Rolldown**：

- **之前**：开发时用 esbuild 预构建和转换，生产时用 Rollup 打包，两套工具行为不完全一致
- **现在**：Rolldown（Rust 实现，兼容 Rollup 插件 API）负责开发预构建和生产打包；Oxc 负责转换和 JS 压缩；Lightning CSS 负责 CSS 压缩

带来的效果：

- 生产构建快 10–30 倍
- 开发和生产行为更一致（例如 CJS 默认导入的互操作）
- 新功能：内置 Devtools、`resolve.tsconfigPaths`、`server.forwardConsole`（把浏览器日志转发到终端）等

迁移要点：`build.rollupOptions` → `build.rolldownOptions`、`esbuild` → `oxc`、`manualChunks` → `codeSplitting`，有兼容层自动转换大部分配置。详细内容见 [中级 Q32](20-mid-qa.md)。

**口述模板（30–60 秒）**

1. 一句话结论：Vite 8 用 Rolldown 统一了开发和生产的打包器。
2. 关键机制：Rolldown 负责打包，Oxc 负责转换和压缩，都是 Rust 实现。
3. 对比或注意点：构建大幅提速，开发和生产不一致的问题减少。
4. 可追问时补充：8.1 进一步推出实验性 Bundled Dev Mode，开发时也打包。

**常见追问**

- 现有 Rollup 插件还能用吗？→ 大部分可以，Rolldown 兼容 Rollup 插件 API，但少数钩子不支持。
