# webpack 初级 Q&A（30 题）

> 目标：能独立搭建和维护一个 webpack 项目，说清每个常用配置项的作用。示例基于 webpack 5。

---

## 一、核心概念

### Q1. webpack 是什么？解决了什么问题？

- **难度**: junior
- **标签**: 概念

**答案**

webpack 是一个**静态模块打包器**（static module bundler）。它从入口文件出发，递归分析所有依赖，构建出一张**模块依赖图**（dependency graph），再把这些模块打包成浏览器可以运行的一个或多个文件（bundle）。

它解决的问题：

1. **模块化**：浏览器早期不支持模块，webpack 让我们可以使用 ESM、CommonJS 组织代码
2. **一切皆模块**：通过 Loader，CSS、图片、字体、TS、Vue 单文件组件都可以被 `import`
3. **转译与兼容**：配合 Babel / SWC 把新语法转为目标浏览器支持的语法
4. **性能优化**：代码分割、按需加载、Tree Shaking、压缩、长效缓存
5. **开发体验**：开发服务器、热更新、source map

**口述模板（30–60 秒）**

1. 一句话结论：webpack 是以模块依赖图为核心的静态打包器。
2. 关键机制：从 entry 递归解析依赖，用 Loader 转换各种资源，用 Plugin 扩展构建流程，最终输出 bundle。
3. 对比或注意点：现在浏览器原生支持 ESM，但打包仍然有意义：减少请求数、Tree Shaking、兼容处理、资源优化。
4. 可追问时补充：Vite 开发时不打包是利用原生 ESM，但生产环境仍然打包。

**常见追问**

- 浏览器都支持 ESM 了，为什么还要打包？→ 大量细碎模块导致请求瀑布、无法做 Tree Shaking 和压缩合并、第三方包大多不是浏览器可直接使用的格式。

---

### Q2. webpack 有哪些核心概念？

- **难度**: junior
- **标签**: 概念, 配置

**答案**

| 概念 | 说明 |
|------|------|
| `entry` | 入口，构建依赖图的起点 |
| `output` | 输出位置、文件名规则、publicPath |
| `module.rules` | 配置 Loader，决定不同类型文件如何转换 |
| `plugins` | 插件，在构建生命周期中执行更广泛的任务（生成 HTML、抽取 CSS、注入变量） |
| `mode` | `development` / `production` / `none`，决定一组默认优化 |
| `resolve` | 模块如何被解析（别名、扩展名） |
| `devServer` | 开发服务器配置 |
| `optimization` | 压缩、代码分割、模块 ID 等优化配置 |
| `target` | 构建目标环境（`web`、`node`、`browserslist`） |

```js
// webpack.config.js
const path = require('node:path')
const HtmlWebpackPlugin = require('html-webpack-plugin')

module.exports = {
  mode: 'production',
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash:8].js',
    clean: true,                     // 每次构建前清空输出目录
  },
  module: {
    rules: [{ test: /\.css$/, use: ['style-loader', 'css-loader'] }],
  },
  plugins: [new HtmlWebpackPlugin({ template: './public/index.html' })],
}
```

**口述模板（30–60 秒）**

1. 一句话结论：entry、output、loader、plugin、mode 是五个核心概念。
2. 关键机制：entry 决定从哪开始，loader 决定文件如何转换，plugin 介入整个构建过程，output 决定输出到哪。
3. 对比或注意点：mode 不是简单的标记，它会开启一组默认配置。
4. 可追问时补充：还有 resolve、optimization、devServer、target 等常用配置。

**常见追问**

- webpack 4 之后可以零配置吗？→ 可以，默认 entry 是 `./src/index.js`，输出到 `dist/main.js`，但实际项目都需要配置。

---

### Q3. module、chunk、bundle 有什么区别？

- **难度**: junior
- **标签**: 概念, 高频

**答案**

- **module（模块）**：源码中的每个文件（JS、CSS、图片…）经过 Loader 处理后都是一个模块
- **chunk（代码块）**：webpack 打包过程中，按入口、动态 import、splitChunks 规则把模块分组形成的中间产物
- **bundle（产物）**：chunk 最终输出到磁盘的文件。通常一个 chunk 对应一个 JS bundle，但一个 chunk 也可能输出多个文件（如 JS + 抽取出的 CSS + source map）

关系：**多个 module 组成一个 chunk，一个 chunk 输出为一个或多个 bundle 文件。**

chunk 的来源：

1. 每个 entry 生成一个初始 chunk（initial chunk）
2. 每个动态 `import()` 生成一个异步 chunk（async chunk）
3. `splitChunks` 拆分出的公共 chunk
4. `runtimeChunk` 拆出的运行时 chunk

**口述模板（30–60 秒）**

1. 一句话结论：module 是源文件，chunk 是打包过程中模块的分组，bundle 是最终输出的文件。
2. 关键机制：chunk 来自入口、动态 import 和 splitChunks。
3. 对比或注意点：一个 chunk 可以输出多个文件，比如 JS 和抽取的 CSS。
4. 可追问时补充：webpack 5 内部用 ModuleGraph 和 ChunkGraph 分别管理模块关系和 chunk 关系。

**常见追问**

- initial chunk 和 async chunk 的区别？→ 前者在页面加载时通过 script 标签加载，后者在运行时按需加载。

---

### Q4. `mode` 的 development 和 production 分别做了什么？

- **难度**: junior
- **标签**: 配置

**答案**

| 方面 | development | production |
|------|-------------|------------|
| `process.env.NODE_ENV` | `'development'` | `'production'` |
| 压缩 | 不压缩 | TerserPlugin 压缩 JS |
| Tree Shaking | `usedExports` 不开启 | 开启 `usedExports`、`sideEffects`、`innerGraph` |
| Scope Hoisting | 关闭 | 开启 `concatenateModules` |
| 模块 / chunk ID | `named`（可读的路径） | `deterministic`（短数字 hash，稳定） |
| devtool 默认 | `eval` | 不生成 |
| 缓存 | 默认内存缓存 | 默认不缓存 |
| `realContentHash` | 关闭 | 开启 |

注意：`mode` 设置的 `process.env.NODE_ENV` 是**注入到打包代码中**的（通过 DefinePlugin），而不是设置 webpack 配置文件运行时 Node 进程的环境变量。

**口述模板（30–60 秒）**

1. 一句话结论：mode 是一组默认优化的开关。
2. 关键机制：production 开启压缩、Tree Shaking、Scope Hoisting、稳定的模块 ID；development 优先构建速度和可调试性。
3. 对比或注意点：mode 注入的 `NODE_ENV` 只在业务代码中生效，配置文件中读不到。
4. 可追问时补充：React、Vue 等库根据 `NODE_ENV` 删除开发环境的警告代码，所以生产包必须正确设置。

**常见追问**

- 配置文件中如何区分环境？→ 导出函数 `(env, argv) => argv.mode === 'production' ? … : …`，或使用 `cross-env` 设置环境变量。

---

### Q5. Loader 是什么？常用的 Loader 有哪些？

- **难度**: junior
- **标签**: Loader

**答案**

webpack 原生只能理解 JS 和 JSON（5.109 起对 CSS、HTML 有内置支持）。**Loader 是一个函数**，接收源文件内容，返回转换后的内容，让 webpack 能把其他类型的文件当作模块处理。

```js
module: {
  rules: [
    { test: /\.[jt]sx?$/, exclude: /node_modules/, use: 'babel-loader' },
    { test: /\.css$/, use: ['style-loader', 'css-loader', 'postcss-loader'] },
    { test: /\.s[ac]ss$/, use: ['style-loader', 'css-loader', 'sass-loader'] },
    { test: /\.(png|jpe?g|svg)$/, type: 'asset' },       // webpack 5 Asset Modules
  ],
}
```

常用 Loader：

| 类别 | Loader |
|------|--------|
| 转译 | `babel-loader`、`swc-loader`、`esbuild-loader`、`ts-loader` |
| 样式 | `css-loader`（解析 `@import` 和 `url()`）、`style-loader`（注入 `<style>`）、`postcss-loader`、`sass-loader`、`less-loader` |
| 框架 | `vue-loader` |
| 其他 | `thread-loader`（多进程）、`@svgr/webpack`（SVG 转组件） |

webpack 5 之前常用的 `file-loader`、`url-loader`、`raw-loader` 已被 **Asset Modules** 取代。

**口述模板（30–60 秒）**

1. 一句话结论：Loader 是文件转换器，输入源码，输出 webpack 能处理的模块代码。
2. 关键机制：通过 `module.rules` 的 `test` 匹配文件，`use` 中的 Loader 从右到左（从下到上）执行。
3. 对比或注意点：webpack 5 用 Asset Modules 替代了 file-loader、url-loader。
4. 可追问时补充：Loader 还有 pitch 阶段，style-loader 就利用了 pitch。

**常见追问**

- 为什么 `use: ['style-loader', 'css-loader']` 的顺序不能反？→ Loader 从右往左执行，css-loader 先把 CSS 转为 JS 模块，style-loader 再把它注入页面。

---

### Q6. Plugin 是什么？常用的插件有哪些？

- **难度**: junior
- **标签**: Plugin

**答案**

**Plugin 是一个带有 `apply(compiler)` 方法的对象**，通过在 webpack 构建生命周期的钩子上注册回调，完成 Loader 无法完成的任务，例如生成 HTML、抽取 CSS、压缩、注入变量、分析产物。

常用插件：

| 插件 | 作用 |
|------|------|
| `HtmlWebpackPlugin` | 生成 HTML 并自动注入打包后的 JS / CSS |
| `MiniCssExtractPlugin` | 把 CSS 抽取为独立文件（生产环境） |
| `DefinePlugin` | 在编译时替换全局常量 |
| `CopyWebpackPlugin` | 复制静态文件（5.111 有内置能力） |
| `TerserPlugin` / `CssMinimizerPlugin` | 压缩 JS / CSS（配置在 `optimization.minimizer`） |
| `ProvidePlugin` | 自动引入模块（如 `Buffer`） |
| `webpack-bundle-analyzer` | 产物体积可视化 |
| `ForkTsCheckerWebpackPlugin` | 在独立进程中做 TS 类型检查 |
| `ModuleFederationPlugin` | 模块联邦 |

**口述模板（30–60 秒）**

1. 一句话结论：Plugin 通过钩子介入整个构建生命周期。
2. 关键机制：webpack 启动时调用每个插件的 `apply(compiler)`，插件在 compiler 或 compilation 的钩子上注册回调。
3. 对比或注意点：Loader 只处理单个文件，Plugin 能访问整个构建过程和所有产物。
4. 可追问时补充：webpack 本身的大部分功能也是用插件实现的。

**常见追问**

- `new HtmlWebpackPlugin()` 为什么要 new？→ 插件是类的实例，构造函数接收选项，`apply` 方法会被 webpack 调用。

---

### Q7. Loader 和 Plugin 有什么区别？

- **难度**: junior
- **标签**: Loader, Plugin, 高频

**答案**

| 维度 | Loader | Plugin |
|------|--------|--------|
| 本质 | 函数：`(source) => newSource` | 对象：`{ apply(compiler) {} }` |
| 作用范围 | 单个文件的内容转换 | 整个构建生命周期 |
| 运行时机 | 构建模块时（make 阶段） | 任意阶段（通过钩子） |
| 配置位置 | `module.rules` | `plugins` |
| 典型用途 | 转译 TS、处理 CSS、图片 | 生成 HTML、抽取 CSS、压缩、分析 |

**口述模板（30–60 秒）**

1. 一句话结论：Loader 是转换器，Plugin 是扩展器。
2. 关键机制：Loader 在构建每个模块时被调用，只关心输入输出；Plugin 通过 Tapable 钩子在任意阶段读写编译状态。
3. 对比或注意点：有些功能需要两者配合，如 MiniCssExtractPlugin 同时提供 Loader 和 Plugin。
4. 可追问时补充：Loader 能做的 Plugin 基本都能做，但 Loader 更简单、可缓存、可并行。

**常见追问**

- 一个功能需要「转换文件 + 输出额外文件」怎么办？→ Loader 负责转换并记录信息，Plugin 在生成产物阶段统一输出，例如 MiniCssExtractPlugin。

---

## 二、常用配置

### Q8. 怎么配置多入口？输出文件名中的占位符有哪些？

- **难度**: junior
- **标签**: entry, output

**答案**

```js
module.exports = {
  entry: {
    main: './src/main.js',
    admin: './src/admin.js',
    // 共享依赖：两个入口共用一个 shared chunk
    // main: { import: './src/main.js', dependOn: 'shared' },
    // shared: ['react', 'react-dom'],
  },
  output: {
    filename: 'js/[name].[contenthash:8].js',
    chunkFilename: 'js/[name].[contenthash:8].chunk.js',  // 异步 chunk
    assetModuleFilename: 'assets/[name].[hash:8][ext]',
    publicPath: '/',
  },
}
```

常用占位符：

| 占位符 | 含义 |
|--------|------|
| `[name]` | chunk 名称（入口名或 webpackChunkName） |
| `[id]` | chunk ID |
| `[contenthash]` | 文件内容的 hash |
| `[chunkhash]` | chunk 的 hash |
| `[fullhash]` | 整个编译的 hash（webpack 4 的 `[hash]`） |
| `[ext]` | 资源扩展名（含 `.`） |

**口述模板（30–60 秒）**

1. 一句话结论：`entry` 写成对象即可多入口，每个入口生成一个初始 chunk。
2. 关键机制：`filename` 用于入口 chunk，`chunkFilename` 用于异步 chunk。
3. 对比或注意点：生产环境用 `[contenthash]` 做长效缓存。
4. 可追问时补充：`dependOn` 可以让多个入口共享依赖，避免重复打包。

**常见追问**

- 多页应用每个页面需要一个 HTML？→ 为每个入口创建一个 `HtmlWebpackPlugin` 实例，并用 `chunks` 指定注入哪些 chunk。

---

### Q9. hash、chunkhash、contenthash 有什么区别？

- **难度**: junior
- **标签**: 缓存, 高频

**答案**

- **fullhash（webpack 4 的 hash）**：整个编译的 hash，任意文件变化都会变，所有文件名都变，缓存全部失效
- **chunkhash**：根据 chunk 的内容计算，同一 chunk 中的 JS 和抽取出的 CSS 共享这个 hash；改了 JS，CSS 文件名也会变
- **contenthash**：根据**输出文件本身的内容**计算，只有文件内容变化时才变

生产环境推荐：JS 和 CSS 都使用 `[contenthash]`。

```js
output: { filename: '[name].[contenthash:8].js' },
plugins: [new MiniCssExtractPlugin({ filename: '[name].[contenthash:8].css' })],
```

**口述模板（30–60 秒）**

1. 一句话结论：粒度从粗到细是 fullhash → chunkhash → contenthash。
2. 关键机制：contenthash 只跟输出文件内容有关，最适合长效缓存。
3. 对比或注意点：chunkhash 会导致 JS 改动时同 chunk 的 CSS 缓存失效。
4. 可追问时补充：即使用了 contenthash，模块 ID 和运行时的变化也会导致无关文件 hash 变化，需要配合 `runtimeChunk` 和 `deterministic` ID。

**常见追问**

- 为什么只改了一个文件，好几个文件的 hash 都变了？→ 见 [20-mid-qa Q15](20-mid-qa.md)。

---

### Q10. CSS 在 webpack 中怎么处理？开发和生产有什么不同？

- **难度**: junior
- **标签**: CSS

**答案**

- `css-loader`：解析 CSS 中的 `@import` 和 `url()`，把 CSS 转换为 JS 模块
- `style-loader`：在运行时创建 `<style>` 标签把 CSS 注入页面（开发环境，支持 HMR）
- `MiniCssExtractPlugin.loader`：把 CSS 抽取为独立的 `.css` 文件（生产环境，可并行加载、可缓存）
- `postcss-loader`：autoprefixer、postcss-preset-env 等
- `sass-loader` / `less-loader`：预处理器

```js
const isProd = process.env.NODE_ENV === 'production'

module.exports = {
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          isProd ? MiniCssExtractPlugin.loader : 'style-loader',
          { loader: 'css-loader', options: { importLoaders: 1 } },   // @import 的文件也经过后面 1 个 loader
          'postcss-loader',
        ],
      },
    ],
  },
  plugins: isProd ? [new MiniCssExtractPlugin({ filename: '[name].[contenthash:8].css' })] : [],
  optimization: { minimizer: ['...', new CssMinimizerPlugin()] },    // '...' 保留默认的 Terser
}
```

webpack 5.109 起 `experiments.css` 默认为 `"auto"`：没有为 CSS 注册 loader 时启用内置 CSS 支持（含 CSS Modules）。现有项目配置了 css-loader 则不受影响。

**口述模板（30–60 秒）**

1. 一句话结论：css-loader 负责解析，style-loader 或 MiniCssExtractPlugin 负责输出。
2. 关键机制：开发用 style-loader 注入，支持热更新；生产用 MiniCssExtractPlugin 抽取独立文件，避免 JS 执行前无样式。
3. 对比或注意点：`importLoaders` 保证 `@import` 进来的文件也经过 postcss 等 loader。
4. 可追问时补充：压缩 CSS 需要在 `minimizer` 中添加 CssMinimizerPlugin，并用 `'...'` 保留默认的 JS 压缩。

**常见追问**

- 为什么生产环境要抽取 CSS？→ 可以与 JS 并行下载、单独缓存，避免 FOUC（无样式内容闪烁）。

---

### Q11. 图片、字体等资源怎么处理？Asset Modules 有哪几种类型？

- **难度**: junior
- **标签**: Asset Modules

**答案**

webpack 5 内置 Asset Modules，不再需要 file-loader / url-loader：

| type | 作用 | 替代 |
|------|------|------|
| `asset/resource` | 输出独立文件，返回 URL | file-loader |
| `asset/inline` | 转为 Data URL 内联 | url-loader |
| `asset/source` | 返回文件源码字符串 | raw-loader |
| `asset` | 自动选择：小于阈值内联，否则输出文件（默认 8KB） | url-loader + limit |

```js
module: {
  rules: [
    {
      test: /\.(png|jpe?g|gif|webp|svg)$/i,
      type: 'asset',
      parser: { dataUrlCondition: { maxSize: 4 * 1024 } },
      generator: { filename: 'images/[name].[contenthash:8][ext]' },
    },
    { test: /\.(woff2?|ttf)$/i, type: 'asset/resource' },
  ],
}
```

5.109 起在 `futureDefaults` 下还支持 `import url from './a.png?url'`、`?raw`、`?inline` 等查询后缀。

**口述模板（30–60 秒）**

1. 一句话结论：webpack 5 用 Asset Modules 处理静态资源，有 resource、inline、source、asset 四种。
2. 关键机制：`asset` 根据大小自动选择内联或输出文件。
3. 对比或注意点：内联可以减少请求，但会增大 JS 体积且无法单独缓存，阈值不宜过大。
4. 可追问时补充：图片压缩可以用 image-minimizer-webpack-plugin，或者在 CDN 层处理。

**常见追问**

- 内联的阈值设多少合适？→ 通常 4–8KB；HTTP/2 下请求成本降低，可以适当调小。

---

### Q12. Babel 在 webpack 中怎么配置？polyfill 怎么处理？

- **难度**: junior
- **标签**: Babel, 兼容

**答案**

```js
// webpack.config.js
{ test: /\.[jt]sx?$/, exclude: /node_modules/, use: 'babel-loader' }
```

```js
// babel.config.js
module.exports = {
  presets: [
    ['@babel/preset-env', { useBuiltIns: 'usage', corejs: '3.40' }],
    ['@babel/preset-react', { runtime: 'automatic' }],
    '@babel/preset-typescript',
  ],
}
```

- `@babel/preset-env` 根据 `browserslist`（`.browserslistrc` 或 package.json）决定需要转换哪些语法
- **语法转换**（箭头函数、class）由 preset-env 完成；**API 补全**（`Promise`、`Array.prototype.includes`）需要 polyfill（core-js）
- `useBuiltIns: 'usage'`：按代码中实际使用的 API 自动引入 polyfill
- `useBuiltIns: 'entry'`：在入口 `import 'core-js'`，按目标浏览器全量替换为需要的模块

**口述模板（30–60 秒）**

1. 一句话结论：babel-loader 调用 Babel 转译，preset-env 按目标浏览器转换语法，core-js 补全 API。
2. 关键机制：browserslist 决定转换范围，`useBuiltIns: 'usage'` 按需引入 polyfill。
3. 对比或注意点：`exclude: /node_modules/` 可以提速，但如果第三方包使用了新语法，需要单独包含它。
4. 可追问时补充：现在很多项目用 swc-loader 或 esbuild-loader 替代 Babel，速度快很多。

**常见追问**

- 写库的时候 polyfill 怎么处理？→ 不要污染全局，使用 `@babel/plugin-transform-runtime`，或者不打 polyfill 交给使用方。详见 [20-mid-qa Q20](20-mid-qa.md)。

---

### Q13. TypeScript 项目怎么配置？ts-loader 和 babel-loader 怎么选？

- **难度**: junior
- **标签**: TypeScript

**答案**

| 方案 | 特点 |
|------|------|
| `ts-loader` | 使用 tsc 编译，**带类型检查**，慢；可开 `transpileOnly: true` 只转译 |
| `babel-loader` + `@babel/preset-typescript` | 只擦除类型，不检查，快；不支持部分 TS 特性（如 `const enum` 跨文件） |
| `swc-loader` / `esbuild-loader` | 只转译，最快 |
| webpack 5.107 `experiments.typescript` | 用 Node 内置的类型擦除，只支持可擦除语法，不支持 `.tsx` |

推荐做法：**转译和类型检查分离**。Loader 只负责转译（swc / babel / ts-loader 的 transpileOnly），类型检查交给 `ForkTsCheckerWebpackPlugin`（独立进程）或 CI 中的 `tsc --noEmit`。

```js
module: {
  rules: [{ test: /\.tsx?$/, exclude: /node_modules/, loader: 'swc-loader' }],
},
resolve: { extensions: ['.ts', '.tsx', '.js'] },
plugins: [new ForkTsCheckerWebpackPlugin()],
```

注意 `tsconfig.json` 中的 `paths` 别名 webpack 不会自动识别，需要在 `resolve.alias` 中配置，或使用 `tsconfig-paths-webpack-plugin`；webpack 5.105 起可以直接设置 `resolve.tsconfig: true` 替代该插件。

**口述模板（30–60 秒）**

1. 一句话结论：转译和类型检查分离，Loader 只转译，类型检查放到独立进程或 CI。
2. 关键机制：ts-loader 带类型检查最慢；babel、swc 只擦除类型，速度快。
3. 对比或注意点：`isolatedModules` 要开启，保证每个文件可以单独转译。
4. 可追问时补充：tsconfig 的 paths 需要同步到 webpack 的 resolve 配置。

**常见追问**

- 为什么要开启 `isolatedModules`？→ babel / swc 逐文件转译，无法处理需要跨文件类型信息的语法（如 re-export 类型、`const enum`）。

---

### Q14. `resolve` 常用的配置有哪些？

- **难度**: junior
- **标签**: resolve

**答案**

```js
resolve: {
  alias: { '@': path.resolve(__dirname, 'src') },   // 路径别名
  extensions: ['.ts', '.tsx', '.js'],              // 省略扩展名时依次尝试
  modules: ['node_modules'],                        // 查找第三方模块的目录
  mainFields: ['browser', 'module', 'main'],        // 读取 package.json 的入口字段顺序
  conditionNames: ['import', 'module', 'browser', 'default'],  // exports 条件导出
  symlinks: true,                                   // Monorepo 中软链接解析为真实路径
  fallback: { buffer: require.resolve('buffer/') }, // Node 核心模块的替代
}
```

注意：

- `extensions` 列表越长，查找越慢，应该把最常用的放前面，而且不要太多
- 第三方包有 `exports` 字段时，`mainFields` 不再生效，以 `exports` 的条件为准

**口述模板（30–60 秒）**

1. 一句话结论：resolve 决定 `import` 的路径如何找到真实文件。
2. 关键机制：常用的是 alias、extensions、mainFields 和 fallback。
3. 对比或注意点：extensions 过多会拖慢解析；package.json 的 exports 字段优先于 mainFields。
4. 可追问时补充：webpack 的解析由 enhanced-resolve 实现。

**常见追问**

- 为什么用 `mainFields` 优先 `module`？→ `module` 字段通常指向 ESM 版本，有利于 Tree Shaking。

---

### Q15. webpack-dev-server 常用配置有哪些？

- **难度**: junior
- **标签**: devServer

**答案**

```js
devServer: {
  port: 3000,
  hot: true,                       // 开启 HMR
  open: true,
  historyApiFallback: true,        // SPA 刷新时所有路由返回 index.html
  static: { directory: path.join(__dirname, 'public') },
  proxy: [
    {
      context: ['/api'],
      target: 'http://localhost:8080',
      changeOrigin: true,          // 修改请求头中的 Host
      pathRewrite: { '^/api': '' },
    },
  ],
  client: { overlay: { errors: true, warnings: false } },
}
```

- dev-server 把构建结果**保存在内存中**（webpack-dev-middleware + memfs），不写磁盘，所以速度快
- `proxy` 用于解决开发环境的跨域：浏览器请求同源的 dev-server，再由 dev-server 转发到后端
- webpack-dev-server v5 中 `proxy` 改为数组形式，要求 Node 18+

**口述模板（30–60 秒）**

1. 一句话结论：dev-server 提供开发服务器、热更新、代理和 SPA 路由回退。
2. 关键机制：构建结果在内存中，通过 WebSocket 与浏览器通信实现热更新。
3. 对比或注意点：proxy 解决跨域的原理是服务器之间请求没有同源限制。
4. 可追问时补充：historyApiFallback 是为了让 History 路由模式在刷新时不 404。

**常见追问**

- 为什么要设置 `changeOrigin`？→ 后端可能根据 Host 做虚拟主机或校验，需要把 Host 改为目标地址。

---

### Q16. source map 是什么？`devtool` 怎么选？

- **难度**: junior
- **标签**: source map, 高频

**答案**

source map 是一个映射文件，记录打包后代码的位置与源码位置的对应关系，用于调试和错误定位。

`devtool` 的关键字组合：

| 关键字 | 含义 |
|--------|------|
| `eval` | 每个模块用 `eval` 包裹，重新构建最快 |
| `source-map` | 生成独立的 `.map` 文件 |
| `cheap` | 只映射到行，不映射列 |
| `module` | 映射到 Loader 处理之前的源码（如 TS 源码） |
| `inline` | source map 以 Data URL 内联到文件中 |
| `hidden` | 生成 map 但不在产物中添加引用注释 |
| `nosources` | map 中不包含源码内容 |

推荐：

- 开发：`eval-cheap-module-source-map`（速度和质量的平衡）
- 生产：`hidden-source-map`（生成 map 上传到错误监控平台，但不暴露给用户）或者 `false`

**口述模板（30–60 秒）**

1. 一句话结论：source map 把产物位置映射回源码，devtool 决定生成方式和质量。
2. 关键机制：eval 最快、cheap 只到行、module 能映射到 Loader 处理前的源码。
3. 对比或注意点：生产环境不要把 source map 公开，避免源码泄露，用 hidden 并上传到 Sentry。
4. 可追问时补充：source map 的原理是 VLQ 编码的 mappings 字段。

**常见追问**

- 生产环境报错怎么定位到源码？→ 构建时生成 hidden source map，上传到监控平台后从产物中删除，平台根据 map 还原堆栈。

---

### Q17. HMR 是什么？怎么开启？

- **难度**: junior
- **标签**: HMR

**答案**

HMR（Hot Module Replacement，模块热替换）指在应用运行时替换修改过的模块，**不刷新整个页面**，从而保留页面状态（如表单输入、弹窗）。

开启方式：`devServer.hot: true`（webpack-dev-server v4+ 会自动添加 `HotModuleReplacementPlugin`）。

- CSS：style-loader 已内置热更新逻辑
- React：使用 `@pmmmwh/react-refresh-webpack-plugin` + `react-refresh/babel`，实现组件状态保留
- Vue：vue-loader 内置
- 普通 JS 模块：需要自己调用 `module.hot.accept()`，否则会冒泡到入口导致整页刷新

```js
if (module.hot) {
  module.hot.accept('./renderer.js', () => {
    render()   // 依赖更新后重新渲染
  })
}
```

**口述模板（30–60 秒）**

1. 一句话结论：HMR 在运行时替换模块，不刷新页面，保留状态。
2. 关键机制：dev-server 通过 WebSocket 通知浏览器，浏览器拉取更新的模块并执行 accept 回调。
3. 对比或注意点：没有模块处理更新时，会回退为整页刷新。
4. 可追问时补充：React 的 Fast Refresh 基于 HMR 实现组件级热更新。

**常见追问**

- `hot: true` 和 `hot: 'only'` 的区别？→ `'only'` 在热更新失败时不会自动刷新页面，方便看到错误。

---

### Q18. 怎么在代码中使用环境变量？

- **难度**: junior
- **标签**: 环境变量

**答案**

webpack 通过 **DefinePlugin 在编译时做文本替换**：

```js
new webpack.DefinePlugin({
  'process.env.API_URL': JSON.stringify(process.env.API_URL),   // 值必须是代码字符串
  __VERSION__: JSON.stringify(require('./package.json').version),
})
```

- 值必须用 `JSON.stringify`，否则 `'https://x'` 会被替换为没有引号的标识符
- 替换是文本级别的，`process.env` 整个对象在浏览器中并不存在
- 也可以使用 `dotenv-webpack`，或 webpack 5.10x 新增的内置 `dotenv` 选项读取 `.env` 文件（具体用法以官方文档为准）
- webpack 5.10x 起支持 `import.meta.env`，并内置 `MODE`、`DEV`、`PROD` 等默认值

**注意**：所有注入到前端的变量都会出现在产物中，**不能放密钥**。

**口述模板（30–60 秒）**

1. 一句话结论：DefinePlugin 在编译时把标识符替换为常量值。
2. 关键机制：替换是文本级别的，值需要 `JSON.stringify`；配合压缩器可以删除死代码分支。
3. 对比或注意点：前端环境变量都是公开的，不能放密钥。
4. 可追问时补充：`if (process.env.NODE_ENV !== 'production')` 在生产包中会被替换为 `if (false)` 并被删除。

**常见追问**

- 同一个产物部署到多个环境怎么办？→ 编译时注入无法实现，改为运行时加载配置（如 `/config.json` 或服务端注入 `window.__CONFIG__`）。

---

### Q19. `output.publicPath` 有什么作用？

- **难度**: junior
- **标签**: publicPath

**答案**

`publicPath` 是**浏览器加载资源时使用的 URL 前缀**，影响：

- HtmlWebpackPlugin 注入的 script / link 地址
- 异步 chunk 的加载地址
- CSS 中 `url()` 和 Asset Modules 生成的地址

```js
output: {
  publicPath: process.env.NODE_ENV === 'production' ? 'https://cdn.example.com/app/' : '/',
}
```

- `'auto'`（webpack 5 默认）：运行时根据当前 script 的地址自动推断
- 运行时动态设置：在入口最前面设置 `__webpack_public_path__ = window.__CDN__`（微前端常用）

**口述模板（30–60 秒）**

1. 一句话结论：publicPath 是资源在浏览器中的访问前缀。
2. 关键机制：所有 chunk、图片、字体的 URL 都会拼接这个前缀。
3. 对比或注意点：path 是磁盘输出目录，publicPath 是访问路径，两者不要混淆。
4. 可追问时补充：需要运行时决定 CDN 地址时，用 `__webpack_public_path__`。

**常见追问**

- 部署到子路径后页面空白、chunk 404？→ 多数是 publicPath 配置错误。

---

### Q20. `externals` 有什么作用？

- **难度**: junior
- **标签**: externals

**答案**

`externals` 告诉 webpack 某些依赖**不打包**，在运行时从外部获取。

```js
externals: {
  react: 'React',           // import React from 'react' → 使用全局变量 window.React
  'react-dom': 'ReactDOM',
}
```

```html
<script src="https://cdn.example.com/react@19/umd/react.production.min.js"></script>
```

使用场景：

1. 应用中使用 CDN 加载大型库
2. **打包库**时把 peerDependencies 外部化，避免重复打包
3. Node 端构建时排除 `node_modules`（`webpack-node-externals`）

注意：React 19 已不再提供 UMD 构建，CDN 方式需要使用 ESM CDN（如 esm.sh）配合 `externalsType: 'module'`。

**口述模板（30–60 秒）**

1. 一句话结论：externals 让某些依赖不打包，运行时从外部获取。
2. 关键机制：import 被替换为读取全局变量或外部模块。
3. 对比或注意点：CDN 外部化需要保证版本一致，且失去了 Tree Shaking。
4. 可追问时补充：打包库时一定要把 react 等 peer 依赖外部化，否则使用方会有两份 React。

**常见追问**

- 用了 externals 还需要 Tree Shaking 吗？→ externals 的库整体加载，无法 Tree Shaking；体积敏感的库不适合 externals。

---

## 三、代码分割与优化入门

### Q21. 代码分割有哪几种方式？

- **难度**: junior
- **标签**: 代码分割, 高频

**答案**

1. **多入口**：每个 entry 一个 chunk（缺点：入口间的公共依赖会重复打包，需要配合 `dependOn` 或 splitChunks）
2. **动态 import**：`import('./Page')` 自动生成异步 chunk，按需加载
3. **splitChunks**：根据规则自动拆分公共模块和第三方库

```js
// 路由级懒加载
const Settings = React.lazy(() => import('./pages/Settings'))

// 第三方库单独拆分
optimization: {
  splitChunks: { chunks: 'all' },   // 默认只拆分异步 chunk，设为 all 同时处理初始 chunk
  runtimeChunk: 'single',
}
```

**口述模板（30–60 秒）**

1. 一句话结论：多入口、动态 import、splitChunks 三种方式。
2. 关键机制：动态 import 按需加载，splitChunks 提取公共代码和第三方库。
3. 对比或注意点：拆分的目的是减少首屏体积和提升缓存命中，拆得过细会增加请求数。
4. 可追问时补充：常见策略是框架库一个 chunk、其他第三方库一个 chunk、业务按路由拆分。

**常见追问**

- splitChunks 默认配置是什么？→ 见 [20-mid-qa Q14](20-mid-qa.md)。

---

### Q22. 动态 import 的魔法注释有哪些？prefetch 和 preload 有什么区别？

- **难度**: junior
- **标签**: 代码分割, 预加载

**答案**

```js
import(/* webpackChunkName: "chart" */ './Chart')
import(/* webpackPrefetch: true */ './Modal')        // 空闲时预取，用于将来可能用到的页面
import(/* webpackPreload: true */ './CriticalLib')   // 与父 chunk 并行加载，当前页面马上就要用
import(/* webpackMode: "lazy-once" */ `./locales/${lang}`)
import(/* webpackIgnore: true */ 'https://cdn.example.com/x.js')   // 不处理，保留原生 import
import(/* webpackExports: ["format"] */ 'date-fns')   // 只使用某些导出，帮助 Tree Shaking
```

| | prefetch | preload |
|---|---|---|
| 时机 | 父 chunk 加载完成后，浏览器空闲时 | 与父 chunk 并行 |
| 优先级 | 低 | 高 |
| 用途 | 将来可能需要的资源 | 当前页面马上需要的资源 |

**口述模板（30–60 秒）**

1. 一句话结论：魔法注释用来控制 chunk 名称、预加载方式和打包模式。
2. 关键机制：prefetch 生成 `<link rel="prefetch">`，在空闲时加载；preload 并行高优先级加载。
3. 对比或注意点：preload 滥用会抢占首屏关键资源的带宽。
4. 可追问时补充：webpackChunkName 相同的动态 import 会被合并为同一个 chunk。

**常见追问**

- 用户鼠标悬停时预加载？→ 在 `onMouseEnter` 中直接调用 `import()`，webpack 会缓存 Promise，真正点击时直接使用。

---

### Q23. Tree Shaking 是什么？需要满足什么条件？

- **难度**: junior
- **标签**: Tree Shaking, 高频

**答案**

Tree Shaking 是**删除未被使用的导出代码**（dead code elimination）的优化。

生效条件：

1. 使用 **ESM**（`import` / `export`），因为 ESM 是静态结构，编译时就能确定依赖关系
2. **production 模式**（开启 `usedExports` 标记和压缩删除）
3. Babel 不能把 ESM 转为 CommonJS（`@babel/preset-env` 的 `modules: false`，babel-loader 默认已处理）
4. 第三方库提供 ESM 版本，并在 package.json 中声明 `sideEffects`

```js
// math.js
export const add = (a, b) => a + b
export const sub = (a, b) => a - b

// index.js
import { add } from './math'   // sub 在生产包中会被删除
```

**口述模板（30–60 秒）**

1. 一句话结论：Tree Shaking 基于 ESM 的静态结构删除未使用的导出。
2. 关键机制：webpack 标记哪些导出没被使用，压缩器负责真正删除。
3. 对比或注意点：CommonJS 是动态的，基本无法 Tree Shaking；有副作用的模块不能整体删除。
4. 可追问时补充：`sideEffects: false` 让 webpack 可以跳过整个未使用的模块。

**常见追问**

- `import _ from 'lodash'` 能 Tree Shaking 吗？→ 不能，lodash 是 CommonJS；用 `lodash-es` 并按名称导入。

---

### Q24. production 模式默认开启了哪些优化？

- **难度**: junior
- **标签**: 优化

**答案**

- `optimization.minimize: true`：TerserPlugin 压缩 JS
- `optimization.usedExports: true`：标记未使用的导出
- `optimization.sideEffects: true`：读取 package.json 的 sideEffects
- `optimization.concatenateModules: true`：Scope Hoisting
- `optimization.innerGraph: true`：分析模块内部的变量引用
- `optimization.mangleExports: true`：压缩导出名
- `optimization.moduleIds / chunkIds: 'deterministic'`
- `optimization.realContentHash: true`
- `optimization.splitChunks`：默认对异步 chunk 拆分
- `performance.hints: 'warning'`：入口或资源超过 244KB 时警告
- `process.env.NODE_ENV = 'production'`

**口述模板（30–60 秒）**

1. 一句话结论：production 默认开启压缩、Tree Shaking、Scope Hoisting 和稳定 ID。
2. 关键机制：usedExports 和 sideEffects 负责标记，Terser 负责删除。
3. 对比或注意点：CSS 压缩需要自己添加 CssMinimizerPlugin。
4. 可追问时补充：splitChunks 默认只拆异步 chunk，通常要改为 `chunks: 'all'`。

**常见追问**

- 为什么开发环境不开启这些？→ 这些优化需要全局分析，会显著拖慢构建；开发时优先速度。

---

### Q25. 多环境的配置怎么组织？

- **难度**: junior
- **标签**: 配置

**答案**

方式一：`webpack-merge` 拆分为公共配置和环境配置：

```text
config/
├─ webpack.common.js
├─ webpack.dev.js     // merge(common, { mode: 'development', devtool: ..., devServer: ... })
└─ webpack.prod.js    // merge(common, { mode: 'production', optimization: ... })
```

方式二：导出函数，根据参数决定配置：

```js
module.exports = (env, argv) => {
  const isProd = argv.mode === 'production'
  return {
    devtool: isProd ? 'hidden-source-map' : 'eval-cheap-module-source-map',
    // ...
  }
}
```

**口述模板（30–60 秒）**

1. 一句话结论：公共配置 + 环境差异配置，用 webpack-merge 合并，或者用函数式配置。
2. 关键机制：webpack-merge 能智能合并数组（如 rules、plugins）。
3. 对比或注意点：环境差异应尽量少，避免开发和生产行为不一致。
4. 可追问时补充：大型团队通常把配置封装成脚手架或 Rsbuild 这类上层工具。

**常见追问**

- 「构建环境」和「部署环境」是一回事吗？→ 不是。mode 决定构建优化；测试、预发、生产的差异最好在运行时配置，产物只构建一次。

---

### Q26. 怎么分析打包产物的体积？

- **难度**: junior
- **标签**: 分析

**答案**

- `webpack-bundle-analyzer`：生成矩形树图，直观查看每个 chunk 包含哪些模块和大小
- `webpack --json > stats.json`：导出 stats，上传到 webpack 官方分析工具或 statoscope
- `performance.hints`：超过阈值时警告或报错
- webpack 5.110 新增的**性能提示**：报告重复包、循环依赖、未使用的 barrel 重导出、缺少 PURE 注解等

```js
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer')
plugins: [process.env.ANALYZE && new BundleAnalyzerPlugin()].filter(Boolean)
```

关注三个大小：stat（原始）、parsed（压缩后）、gzip / brotli（传输大小）。

**口述模板（30–60 秒）**

1. 一句话结论：用 bundle-analyzer 可视化，用 stats 做详细分析和对比。
2. 关键机制：找出大模块、重复依赖、本应异步却进入首屏的模块。
3. 对比或注意点：真正影响加载的是传输体积和解析执行时间。
4. 可追问时补充：可以在 CI 中对比每次提交的体积变化，设置体积预算。

**常见追问**

- 发现 moment 的语言包很大？→ 用 `IgnorePlugin` 或 `ContextReplacementPlugin` 只保留需要的语言，或者换用 dayjs。

---

### Q27. CSS Modules 和 PostCSS 怎么配置？

- **难度**: junior
- **标签**: CSS

**答案**

```js
{
  test: /\.module\.css$/,
  use: [
    'style-loader',
    {
      loader: 'css-loader',
      options: {
        modules: {
          localIdentName: isProd ? '[hash:base64:6]' : '[path][name]__[local]',
          namedExport: true,     // css-loader 7 默认开启：import * as styles / import { btn }
        },
        importLoaders: 1,
      },
    },
    'postcss-loader',
  ],
}
```

```js
// postcss.config.js
module.exports = { plugins: [require('postcss-preset-env')({ stage: 2 })] }
```

- CSS Modules 把类名编译为唯一名称，避免全局污染
- PostCSS 负责 autoprefixer、新语法降级、Tailwind 等
- 注意 css-loader 7 默认 `namedExport: true`，`import styles from './a.module.css'` 的默认导入会失效

**口述模板（30–60 秒）**

1. 一句话结论：css-loader 开启 modules 实现局部作用域，postcss-loader 处理兼容和扩展语法。
2. 关键机制：类名被编译为带 hash 的唯一名称，JS 中通过导入对象使用。
3. 对比或注意点：css-loader 7 的命名导出变化会导致升级后样式丢失。
4. 可追问时补充：webpack 内置 CSS 支持也支持 CSS Modules。

**常见追问**

- 怎么覆盖第三方组件的样式？→ 使用 `:global(.ant-btn)`。

---

### Q28. html-webpack-plugin 做了什么？

- **难度**: junior
- **标签**: 插件

**答案**

- 根据模板生成 HTML 文件
- 自动注入打包后的 JS 和 CSS（文件名带 hash 也能正确引用）
- 支持多页应用（多个实例 + `chunks` 选项）
- 支持压缩 HTML、注入变量（`templateParameters`）
- 提供钩子，供其他插件修改 HTML（如注入 preload、CSP nonce）

```js
new HtmlWebpackPlugin({
  template: './public/index.html',
  scriptLoading: 'defer',
  minify: isProd,
})
```

webpack 官方 2026 路线图计划把 HTML 入口内置到核心中（5.109 已有 `experiments.html`），未来可能不再需要这个插件。

**口述模板（30–60 秒）**

1. 一句话结论：根据模板生成 HTML，并自动注入带 hash 的产物。
2. 关键机制：在 compilation 中读取入口对应的 chunk 文件，生成 HTML 资源。
3. 对比或注意点：多页应用需要多个实例并指定 chunks。
4. 可追问时补充：它暴露了自己的钩子，其他插件可以在其基础上修改 HTML。

**常见追问**

- 为什么推荐 `scriptLoading: 'defer'`？→ 脚本不阻塞 HTML 解析，按顺序在 DOM 解析完成后执行。

---

### Q29. `watch` 模式和 dev-server 有什么区别？

- **难度**: junior
- **标签**: 开发

**答案**

| | `webpack --watch` | webpack-dev-server |
|---|---|---|
| 产物 | 写入磁盘 | 存在内存中 |
| 服务器 | 无 | 内置 HTTP 服务器 |
| 热更新 | 无 | 支持 HMR 和自动刷新 |
| 适用 | 库开发、需要其他服务读取产物（如后端模板） | 应用开发 |

两者底层都使用 Compiler 的 `watch` 方法：监听文件变化，增量重新编译。

**口述模板（30–60 秒）**

1. 一句话结论：watch 只负责增量编译写盘，dev-server 额外提供服务器和热更新。
2. 关键机制：两者都使用 compiler.watch 监听文件变化并增量构建。
3. 对比或注意点：dev-server 的产物在内存中，所以磁盘上看不到 dist。
4. 可追问时补充：需要写盘时可以设置 `devServer.devMiddleware.writeToDisk`。

**常见追问**

- 文件修改后不触发重新编译？→ 检查 Docker、WSL 或网络文件系统下文件监听是否生效，必要时开启 `watchOptions.poll`。

---

### Q30. webpack、Vite、Rspack 有什么区别？

- **难度**: junior
- **标签**: 选型

**答案**

| | webpack | Vite | Rspack |
|---|---|---|---|
| 实现语言 | JS | JS + Rust（Rolldown）/ esbuild | Rust |
| 开发模式 | 先打包再启动 | 不打包，浏览器原生 ESM 按需编译 | 先打包，但速度很快 |
| 生产构建 | webpack | Rollup / Rolldown | Rspack |
| 生态 | 最成熟 | 插件基于 Rollup 接口 | 兼容大部分 webpack loader 和插件 |
| 适用 | 存量项目、复杂定制 | 新项目、中小型应用 | 大型 webpack 项目提速 |

**口述模板（30–60 秒）**

1. 一句话结论：webpack 生态最成熟，Vite 开发体验好，Rspack 用 Rust 兼容 webpack 生态并大幅提速。
2. 关键机制：Vite 开发时不打包，冷启动快；Rspack 用 Rust 实现 webpack 的架构。
3. 对比或注意点：Vite 开发和生产使用不同的方式，存在不一致风险（Rolldown 正在统一）。
4. 可追问时补充：存量大型 webpack 项目迁移 Rspack 成本低，迁移 Vite 改动更大。

**常见追问**

- 为什么学了 Vite 还要学 webpack？→ 存量项目大量使用 webpack；模块图、chunk、Tree Shaking、HMR 等原理是通用的；Rspack 的配置和 webpack 基本一致。
