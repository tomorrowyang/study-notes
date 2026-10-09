# Vite 高级 Q&A

> 目标：能讲清 Vite 的源码架构和核心算法（插件容器、模块图、HMR 传播、预构建），理解 Rolldown、Bundled Dev Mode 和 Environment API 的设计动机，能排查线上和开发问题，能主导插件设计、迁移和选型。

---

## 一、原理深入

### Q1. 从源码角度讲讲 Vite 开发服务器的启动过程

- **难度**: 高级
- **标签**: 源码, 架构

**答案**

```text
createServer(inlineConfig)
  1. resolveConfig
     - 加载 vite.config（用 Rolldown 打包配置文件后执行，所以配置中可以写 TS 和 ESM）
     - 按 enforce 排序插件，执行 config → configResolved 钩子
     - 加载 .env，确定 mode
  2. 创建 HTTP 服务器、WebSocket 服务器、chokidar watcher
  3. 为每个 environment（client、ssr 等）创建：
     - PluginContainer（模拟 Rollup 插件上下文）
     - EnvironmentModuleGraph
     - DepsOptimizer（client 环境）
  4. 执行插件的 configureServer 钩子（收集后置中间件）
  5. 注册内置中间件：
     代理 → base → 静态文件（public）→ transformMiddleware → HTML fallback → indexHtml → 后置中间件 → 404
  6. listen 之前：调用 buildStart；启动依赖扫描和预构建
  7. server.warmup 中配置的文件开始预转换
```

关键设计：**启动阶段不转换源码**，只做配置解析和依赖预构建，源码在请求时才转换。

**口述模板（30–60 秒）**

1. 一句话结论：启动过程主要是解析配置、创建服务器组件、注册中间件和启动预构建。
2. 关键机制：每个环境有独立的插件容器和模块图，源码转换延迟到请求时。
3. 对比或注意点：配置文件本身也会被打包后执行，所以支持 TS。
4. 可追问时补充：中间件顺序决定了 configureServer 中注册的中间件在哪个位置生效。

**常见追问**

- 修改 `vite.config.ts` 后会怎样？→ Vite 监听配置文件及其依赖，变化后自动重启服务器。

---

### Q2. PluginContainer 是怎么实现的？

- **难度**: 高级
- **标签**: 源码, 插件

**答案**

开发时没有真正的 Rollup / Rolldown 构建，但插件依赖 Rollup 的上下文（`this.resolve`、`this.emitFile`、`this.warn`、`this.getModuleInfo` 等）。PluginContainer 模拟了这个上下文：

- **`resolveId`**：按插件顺序依次调用，**第一个返回非空值的插件生效**（first 模式）
- **`load`**：同样是 first 模式
- **`transform`**：**依次串行**调用所有插件，上一个的输出作为下一个的输入（sequential 模式），同时合并 source map 链
- **`buildStart` 等**：并行调用（parallel 模式）
- 上下文中的 `this.resolve` 调用会回到容器自身，支持 `skipSelf` 避免死循环
- `this.getModuleInfo` 从模块图中获取信息
- `this.environment`（Vite 6+）获取当前环境

限制：开发时不支持的能力（例如 `moduleParsed`、输出钩子），以及 `this.emitFile` 不会真正写出文件。

**口述模板（30–60 秒）**

1. 一句话结论：PluginContainer 在开发时模拟 Rollup 的插件上下文，让 Rollup 插件能在不打包的情况下运行。
2. 关键机制：resolveId 和 load 是 first 模式，transform 是串行模式并合并 source map。
3. 对比或注意点：只模拟了部分能力，输出相关的钩子在开发时不可用。
4. 可追问时补充：this.resolve 支持 skipSelf，用于插件包装其他解析逻辑。

**常见追问**

- 为什么 transform 要合并 source map？→ 每个插件都可能改变代码，只有把所有映射串起来，才能从最终代码映射回源码。

---

### Q3. ModuleGraph 记录了哪些信息？

- **难度**: 高级
- **标签**: 源码, 模块图

**答案**

**索引**：

- `urlToModuleMap`：浏览器请求的 URL → 模块
- `idToModuleMap`：解析后的 id → 模块
- `fileToModulesMap`：文件路径 → 模块集合（一个文件可能对应多个模块，例如 `a.vue` 和 `a.vue?type=style`）

**每个 ModuleNode**：

- `url`、`id`、`file`、`type`（js / css）
- `importers`：谁导入了我
- `importedModules`：我导入了谁
- `acceptedHmrDeps`：我接受哪些依赖的更新
- `isSelfAccepting`：是否自身接受更新
- `transformResult`：转换结果缓存
- `lastHMRTimestamp`、`lastInvalidationTimestamp`

**作用**：

- 缓存转换结果，避免重复转换
- 文件变化时，通过 `fileToModulesMap` 找到受影响的模块
- HMR 时，通过 `importers` 向上查找边界
- 导入分析时，给 URL 添加 `?t=` 时间戳

**口述模板（30–60 秒）**

1. 一句话结论：模块图记录模块的索引、导入关系、HMR 接受信息和转换缓存。
2. 关键机制：三套索引分别服务于请求、解析和文件监听，importers 用于 HMR 向上查找。
3. 对比或注意点：一个文件可能对应多个模块，例如 Vue SFC 的不同部分。
4. 可追问时补充：Vite 6 起每个环境有独立的模块图。

**常见追问**

- 模块图中的导入关系是什么时候建立的？→ 在 importAnalysis 插件中，转换完成后根据导入语句更新。

---

### Q4. 详细讲讲 HMR 更新传播算法

- **难度**: 高级
- **标签**: HMR, 源码

**答案**

核心函数逻辑（简化）：

```text
propagateUpdate(node, traversed, boundaries):
  if node 在 traversed 中（循环导入）:
    return 需要整页刷新
  traversed.add(node)

  if node.isSelfAccepting:
    boundaries.add({ boundary: node, acceptedVia: node })
    // CSS 模块等：继续检查导入方中是否有 CSS 需要更新
    return 不需要刷新

  if node.importers 为空:
    return 需要整页刷新（到达入口仍然没有边界）

  for importer of node.importers:
    if importer.acceptedHmrDeps 包含 node:
      boundaries.add({ boundary: importer, acceptedVia: node })
      continue
    if propagateUpdate(importer, ...) 需要刷新:
      return 需要整页刷新

  return 不需要刷新
```

关键点：

- **所有路径都必须找到边界**，任意一条路径到达入口都会整页刷新
- **循环导入**会导致整页刷新（无法确定安全的更新顺序）
- 客户端收到更新后，对每个边界执行：旧模块 dispose → 带时间戳重新 import 边界与变化模块之间的模块 → 调用 accept 回调
- 边界**以上**的模块不重新执行，它们持有的仍是旧模块的引用（这也是为什么 Vite 文档说 HMR 不会替换原来导入的模块）

**口述模板（30–60 秒）**

1. 一句话结论：从变化的模块沿 importers 向上递归查找边界，所有路径都找到边界才能热更新。
2. 关键机制：self-accepting 或导入方 accept 了该依赖就是边界，到达入口或遇到循环就整页刷新。
3. 对比或注意点：边界以上的模块不会重新执行，持有的仍是旧引用。
4. 可追问时补充：循环导入是 HMR 整页刷新的常见原因。

**常见追问**

- 为什么循环导入要整页刷新？→ 循环中的模块互相依赖，重新执行顺序无法保证状态一致。

---

### Q5. 依赖预构建还有哪些细节？

- **难度**: 高级
- **标签**: 预构建, 源码

**答案**

**hash 设计**：

- `hash`：由 lockfile、配置（optimizeDeps、resolve 等影响预构建的部分）、`patches` 目录计算，决定是否重新预构建
- `browserHash`：在 hash 基础上加上依赖列表，作为 URL 的 `?v=` 参数；新增依赖时变化，让浏览器拿到新版本

**运行时发现新依赖**：

1. 导入分析时发现一个未预构建的依赖
2. 注册为待处理依赖，返回一个预期的路径
3. 防抖后重新执行预构建
4. 如果新的预构建导致已有依赖的 chunk 发生变化（例如公共 chunk 重新拆分），浏览器中已经加载的旧模块和新模块不兼容 → 发送整页刷新
5. 已发出的旧版本请求返回 `504 Outdated Optimize Dep`

**`holdUntilCrawlEnd`**（默认 true）：等首屏所有静态导入都处理完，再统一提交预构建结果，尽量只预构建一次，避免多次刷新。

**为什么要合并多个依赖一起打包**：保证共享依赖（React）只有一份实例。

**口述模板（30–60 秒）**

1. 一句话结论：预构建有两层 hash，并且会在运行时发现新依赖时重新构建。
2. 关键机制：hash 决定是否重建，browserHash 决定浏览器缓存；重新预构建导致 chunk 变化时整页刷新。
3. 对比或注意点：holdUntilCrawlEnd 等首屏爬取完再提交，减少刷新次数。
4. 可追问时补充：504 Outdated Optimize Dep 是旧版本依赖请求被拒绝。

**常见追问**

- 怎样彻底避免运行时发现？→ 用 `optimizeDeps.include` 列出所有动态加载的依赖，或者设置 `optimizeDeps.entries` 包含所有页面入口。

---

### Q6. Vite 8 为什么要换成 Rolldown？

- **难度**: 高级
- **标签**: Rolldown, 架构

**答案**

**Vite 8 之前的问题**：

1. **双打包器不一致**：开发时用 esbuild 预构建和转换，生产时用 Rollup 打包，两者在 CJS 互操作、Tree Shaking、代码分割上的行为不同，导致开发正常、生产出错
2. **生产构建慢**：Rollup 是 JS 实现，大项目构建需要几分钟
3. **重复处理**：同样的转换在两个工具中各做一遍，配置也要写两套（`esbuild`、`rollupOptions`）

**Rolldown 的设计**：

- Rust 实现，速度接近 esbuild
- **兼容 Rollup 插件 API**，现有生态可以继续使用
- 内置常用能力：CJS 处理、node resolve、JSON、TS 转换（Oxc）、压缩，减少 JS 插件
- 支持更细粒度的代码分割（codeSplitting groups）

**效果**：构建快 10–30 倍（官方数据），开发和生产使用同一个打包器，行为一致。

**代价**：

- JS 插件有跨语言调用开销，需要钩子过滤器
- 部分 Rollup 钩子和输出格式不支持
- 安装体积增加

**口述模板（30–60 秒）**

1. 一句话结论：换 Rolldown 是为了解决双打包器不一致和生产构建慢的问题。
2. 关键机制：Rolldown 用 Rust 实现、兼容 Rollup 插件 API，开发和生产统一使用。
3. 对比或注意点：JS 插件有跨语言开销，需要钩子过滤器优化。
4. 可追问时补充：这是 VoidZero 统一工具链（Rolldown + Oxc + Vitest）的一部分。

**常见追问**

- 为什么不直接用 esbuild 做生产打包？→ esbuild 的代码分割和插件能力有限，不兼容 Rollup 插件生态。

---

### Q7. Bundled Dev Mode 要解决什么问题？

- **难度**: 高级
- **标签**: 性能, Vite 8.1

**答案**

**不打包开发模式的瓶颈**：

- 模块数量巨大（上万个）时，浏览器请求数巨大，即使每个请求很快，总耗时也很可观
- 请求瀑布：深层依赖要一层层发现
- 网络层开销（尤其通过代理、远程开发环境时）

**Bundled Dev Mode（Vite 8.1 实验性，原名 Full Bundle Mode）**：

```ts
export default defineConfig({
  experimental: { bundledDev: true },
})
// 或命令行：vite --experimental-bundle
```

- 开发时也用 Rolldown 打包，利用 Rolldown 的速度和增量构建
- HMR 由 Rolldown 的运行时处理，只重新构建变化部分
- 官方数据：大型应用启动快约 3 倍、整页刷新快约 40%、请求数减少约 10 倍

**限制**：目前只支持浏览器端，部分插件不兼容，仍为实验性。

这说明业界的共识：**不打包的优势是启动快，但在超大规模下请求数成为新瓶颈**；足够快的打包器可以兼得。

**口述模板（30–60 秒）**

1. 一句话结论：Bundled Dev Mode 解决超大项目开发时请求过多的问题。
2. 关键机制：开发时也用 Rolldown 打包，增量构建保证速度。
3. 对比或注意点：仍是实验性，部分插件不兼容。
4. 可追问时补充：这说明打包器足够快以后，开发时打包反而更有优势。

**常见追问**

- 什么项目适合开启？→ 模块数量非常多、首次加载请求数上万的项目，可以先在部分开发者中试用。

---

### Q8. Module Runner 是什么？和 `ssrLoadModule` 有什么关系？

- **难度**: 高级
- **标签**: SSR, Environment API

**答案**

**`ssrLoadModule`（旧方式）**：

- 在 Vite 服务器所在的 Node 进程中，把模块转换为 SSR 格式（`import` 转换为 `__vite_ssr_import__` 调用），再用 `new AsyncFunction` 执行
- 只能在 Node 进程中运行

**Module Runner（Vite 6+）**：

- 把「模块转换」和「模块执行」分离：
  - 转换在 Vite 服务器中进行
  - 执行在 Module Runner 中进行，**Runner 可以运行在任意运行时**（Node、workerd、Deno）
- 两者通过**传输层**通信：Runner 请求模块，服务器返回转换结果
- Runner 也支持 HMR：服务器通知 Runner 模块失效

```ts
import { createServerModuleRunner } from 'vite'

const runner = createServerModuleRunner(server.environments.ssr)
const { render } = await runner.import('/src/entry-server.ts')
```

**意义**：开发时可以让代码运行在与生产一致的运行时中，例如 Cloudflare Workers 的代码直接运行在 workerd 中。

**口述模板（30–60 秒）**

1. 一句话结论：Module Runner 把模块转换和执行分离，执行可以在任意运行时。
2. 关键机制：通过传输层请求转换结果，并支持 HMR 失效。
3. 对比或注意点：ssrLoadModule 只能在 Vite 所在的 Node 进程中执行。
4. 可追问时补充：这是 Environment API 的基础，Vitest 也基于类似机制。

**常见追问**

- 为什么要转换成 `__vite_ssr_import__`？→ 便于 Runner 自己管理模块加载、缓存和失效，实现 HMR。

---

### Q9. CSS 处理有哪些深层问题？

- **难度**: 高级
- **标签**: CSS

**答案**

**1. 顺序问题**

- 开发时 CSS 按模块执行顺序注入 `<style>`
- 生产时同一 chunk 的 CSS 合并，按 chunk 加载顺序插入
- 依赖导入顺序决定优先级的样式（例如覆盖组件库样式）可能在生产中失效
- 解决：提高选择器优先级、使用 CSS Layers（`@layer`）、CSS Modules 或原子化 CSS

**2. 重复问题**

- 多个异步 chunk 都导入同一个 CSS，可能被分别抽取，导致重复
- 共享 CSS 最好在公共 chunk 中导入

**3. 异步 chunk 的 CSS 加载**

- 加载异步 chunk 前，`__vitePreload` 会先加载对应 CSS，等 CSS 加载完成再执行 JS，避免无样式闪烁

**4. Lightning CSS**

- 可以通过 `css.transformer: 'lightningcss'` 替代 PostCSS 做转换（前缀、降级、CSS Modules），速度更快
- 但如果依赖 PostCSS 插件（如 Tailwind 的部分旧版本插件），需要保留 PostCSS

**口述模板（30–60 秒）**

1. 一句话结论：CSS 的深层问题主要是开发和生产的顺序差异、重复抽取和异步加载。
2. 关键机制：生产时 CSS 按 chunk 合并和加载，顺序与开发时不同。
3. 对比或注意点：不要依赖导入顺序决定优先级，可以使用 CSS Layers。
4. 可追问时补充：Lightning CSS 可以替代 PostCSS，但要确认插件依赖。

**常见追问**

- 为什么异步 chunk 要等 CSS 加载完再执行？→ 否则组件先渲染、样式后到，会出现无样式闪烁。

---

### Q10. Rolldown 的 Tree Shaking 和副作用判断是怎样的？

- **难度**: 高级
- **标签**: Tree Shaking

**答案**

- 基于 ESM 静态结构分析未使用的导出，结合 Oxc 的语义分析判断语句是否有副作用
- 副作用来源：
  - `package.json` 的 `sideEffects` 字段
  - `/*#__PURE__*/`、`/*#__NO_SIDE_EFFECTS__*/` 注释
  - 插件在 `resolveId` / `load` / `transform` 中返回 `moduleSideEffects`
  - `build.rolldownOptions.treeshake` 配置
- 常见失效原因：CJS 模块、顶层副作用（修改全局对象）、`sideEffects` 未声明、barrel 文件中的副作用导入

开发时**不做 Tree Shaking**，所以开发时加载的模块数比生产多。

**口述模板（30–60 秒）**

1. 一句话结论：Rolldown 基于 ESM 静态分析和副作用判断做 Tree Shaking。
2. 关键机制：副作用信息来自 sideEffects、PURE 注释和插件返回的 moduleSideEffects。
3. 对比或注意点：开发时不做 Tree Shaking，barrel 文件会加载所有模块。
4. 可追问时补充：原理与 webpack 相同，见 webpack 高级 Q4。

**常见追问**

- 插件生成的虚拟模块怎么声明无副作用？→ 在 load 返回值中设置 `moduleSideEffects: false`。

---

## 二、性能与排查

### Q11. 大型项目开发环境首次加载很慢，怎么系统排查？

- **难度**: 高级
- **标签**: 性能, 排查

**答案**

**1. 量化**

- Network 面板：请求数、总耗时、瀑布深度
- `vite --debug transform`：每个模块转换耗时
- Vite Devtools / `vite-plugin-inspect`：插件耗时排行

**2. 定位原因**

| 现象 | 原因 | 处理 |
|------|------|------|
| 请求数上万 | barrel 文件、图标库全量导入 | 直接导入具体路径；图标按需导入 |
| 某些插件耗时高 | 插件没有过滤、做了全量 AST 解析 | 加钩子过滤器、缓存、换 Rust 实现 |
| 瀑布很深 | 依赖层级深 | `server.warmup` 预热；路由懒加载 |
| 多次整页刷新 | 运行时发现新依赖 | `optimizeDeps.include` |
| 解析耗时高 | 省略扩展名、alias 过多 | 减少 `resolve.extensions`，导入时写扩展名 |

**3. 进一步**

- 尝试 Bundled Dev Mode
- 远程开发、容器内开发时检查文件系统性能和网络

**口述模板（30–60 秒）**

1. 一句话结论：先量化请求数和插件耗时，再按原因针对性优化。
2. 关键机制：请求数多就治理 barrel，插件慢就加过滤器，瀑布深就预热。
3. 对比或注意点：不要一上来就换工具，先找到真正的瓶颈。
4. 可追问时补充：模块数极大时 Bundled Dev Mode 是终极方案。

**常见追问**

- 怎么找到 barrel 文件导致的无用模块？→ 在 inspect 中查看哪些模块被加载但当前页面并未使用。

---

### Q12. 生产构建慢或内存占用高，怎么优化？

- **难度**: 高级
- **标签**: 性能

**答案**

Vite 8 下 Rolldown 本身很快，构建慢通常是 **JS 插件**导致的：

1. `vite build --profile` 查看 CPU 分布，找出耗时插件
2. 为插件加钩子过滤器，减少跨语言调用
3. 用 Rolldown 内置能力替代 JS 插件（CJS、JSON、alias、TS 转换）
4. 关闭非必要功能：
   - `build.reportCompressedSize: false`（计算 gzip 大小很耗时）
   - 非必要时关闭 source map
5. plugin-legacy 会使构建时间成倍增加，评估是否真的需要
6. CI 中缓存 `node_modules`，避免重复安装

内存问题：source map 和超大单文件是主要来源；可以拆分产物、关闭 source map 或增加 Node 内存。

**口述模板（30–60 秒）**

1. 一句话结论：Vite 8 的构建瓶颈主要在 JS 插件。
2. 关键机制：用 profile 定位，加过滤器、换内置能力。
3. 对比或注意点：reportCompressedSize 和 plugin-legacy 的开销经常被忽视。
4. 可追问时补充：Rolldown 内置了很多以前需要 JS 插件的能力。

**常见追问**

- 升级 Vite 8 后构建没有明显提速？→ 很可能被 JS 插件拖慢，检查插件耗时。

---

### Q13. 经常出现「new dependencies optimized, reloading page」，怎么解决？

- **难度**: 高级
- **标签**: 预构建, 排查

**答案**

**原因**：首屏之后访问某个页面时，动态导入了一个之前没有扫描到的依赖，Vite 需要重新预构建，导致整页刷新。

**排查**：日志会打印新发现的依赖名。

**解决**：

1. 把这些依赖加入 `optimizeDeps.include`
2. 用 `optimizeDeps.entries` 指定所有页面入口，让扫描阶段覆盖所有路由
3. 依赖被 exclude 但它的子依赖是 CJS 时，用 `include: ['my-lib > cjs-dep']` 形式
4. 检查是否有插件生成的导入（如自动导入插件），扫描阶段无法发现

**CI / 团队**：

- 新成员第一次启动时缓存为空，可以在 README 中说明，或用脚本预先启动一次

**口述模板（30–60 秒）**

1. 一句话结论：这是运行时发现新依赖导致的重新预构建。
2. 关键机制：扫描阶段没有覆盖所有入口或依赖，访问新页面时才发现。
3. 对比或注意点：解决方法是 include 和 entries，让扫描阶段覆盖完整。
4. 可追问时补充：插件动态生成的导入在扫描阶段看不到，需要手动 include。

**常见追问**

- include 太多会影响启动吗？→ 首次预构建稍慢，但有缓存，后续不影响。

---

### Q14. HMR 不生效或总是整页刷新，怎么排查？

- **难度**: 高级
- **标签**: HMR, 排查

**答案**

**1. 看日志**：`vite --debug hmr` 会打印每次更新的边界和原因，例如「page reload」以及原因。

**2. 常见原因**：

| 原因 | 处理 |
|------|------|
| 文件混合导出组件和非组件 | 拆分文件 |
| 循环导入 | 用 `madge` 等工具找出循环，重构 |
| 修改的文件没有被任何边界接受 | 正常行为，或者手动添加 accept |
| 文件监听失效（Docker、WSL、网络磁盘） | `server.watch.usePolling: true` |
| 路径大小写不一致 | 统一导入路径大小写 |
| 通过反向代理访问，WebSocket 连接失败 | 配置 `server.hmr.clientPort` / `host` / `path` |
| 修改的文件在 node_modules 中 | 已预构建，不支持 HMR，需要 `--force` |

**口述模板（30–60 秒）**

1. 一句话结论：先用 debug hmr 看原因，再按原因处理。
2. 关键机制：常见原因是混合导出、循环导入、文件监听失效和 WebSocket 连接失败。
3. 对比或注意点：容器和 WSL 环境中文件监听经常失效，需要轮询。
4. 可追问时补充：通过 Nginx 代理开发服务器时，需要正确转发 WebSocket。

**常见追问**

- WSL 中为什么监听失效？→ 项目在 Windows 文件系统中时，Linux 侧收不到文件变化事件，要么把项目放在 WSL 文件系统中，要么开启轮询。

---

### Q15. 线上出现白屏或 chunk 加载失败，怎么处理？

- **难度**: 高级
- **标签**: 线上问题

**答案**

**常见原因**：

1. **发版后旧 chunk 被删除**：用户还停留在旧页面，点击路由加载旧 chunk 返回 404
2. **base 配置错误**：资源路径与实际部署路径不一致
3. **CDN 缓存了旧的 index.html**
4. **服务器把 JS 请求回退到 index.html**：返回 HTML 而不是 JS，报 MIME 类型错误

**Vite 提供的兜底事件**：

```ts
window.addEventListener('vite:preloadError', (event) => {
  // 阻止错误抛出，改为刷新页面拿到最新的 index.html
  event.preventDefault()
  window.location.reload()
})
```

需要防止无限刷新：在 sessionStorage 中记录刷新时间，短时间内只刷新一次。

**根本措施**：

- 发布时保留旧资源一段时间（或采用不删除的增量发布）
- `index.html` 不缓存
- 静态资源路径不存在时返回 404，不要回退到 index.html
- 监控 chunk 加载失败率

**口述模板（30–60 秒）**

1. 一句话结论：最常见的是发版后旧 chunk 被删除导致加载失败。
2. 关键机制：Vite 提供 `vite:preloadError` 事件，可以捕获后刷新页面。
3. 对比或注意点：要防止无限刷新，并且根本上要保留旧资源。
4. 可追问时补充：资源 404 时不要回退到 index.html，否则会报 MIME 错误。

**常见追问**

- `vite:preloadError` 能捕获所有失败吗？→ 只能捕获经过 `__vitePreload` 的动态导入，还需要在路由懒加载处配合错误边界。

---

### Q16. 开发和生产不一致，有什么系统排查方法？

- **难度**: 高级
- **标签**: 排查

**答案**

**1. 缩小范围**

- `vite build && vite preview` 本地复现
- `vite build --mode development`：排除环境变量分支的影响
- `build.minify: false`：排除压缩问题
- `build.rolldownOptions.treeshake: false`：排除 Tree Shaking 问题
- `build.cssCodeSplit: false`：排除 CSS 分割顺序问题

**2. 常见根因检查**

- 依赖的 `sideEffects` 声明是否正确
- CJS 依赖的默认导入（Vite 8 升级后尤其注意）
- 循环依赖导致的执行顺序问题（生产中合并后变量可能处于未初始化状态）
- 依赖 `fn.name` 或 `class.name` 的代码
- 动态拼接的资源路径
- 依赖导入顺序决定的 CSS 优先级

**3. 预防**

- E2E 测试跑在构建产物上
- CI 中对 preview 做冒烟测试

**口述模板（30–60 秒）**

1. 一句话结论：通过逐个关闭生产优化来二分定位。
2. 关键机制：依次排除环境变量、压缩、Tree Shaking、CSS 分割的影响。
3. 对比或注意点：常见根因是副作用声明、CJS 互操作、循环依赖和 CSS 顺序。
4. 可追问时补充：E2E 测试应该跑在构建产物上。

**常见追问**

- 循环依赖为什么在生产中更容易出问题？→ 合并到同一个 chunk 后，模块执行顺序变化，某个变量可能在初始化前被访问（TDZ 错误）。

---

### Q17. 怎么用 Vite 做产物体积治理？

- **难度**: 高级
- **标签**: 性能, 体积

**答案**

**1. 分析**

- `rollup-plugin-visualizer` 或 Vite Devtools：查看每个 chunk 的组成
- 关注：重复依赖、超大依赖、首屏 chunk 体积

**2. 优化**

- 路由级懒加载
- 大依赖替换：`moment` → `dayjs`、`lodash` → `lodash-es` 或原生实现
- 按需导入组件库和图标
- `codeSplitting.groups` 拆分稳定的大依赖，提高缓存命中
- 检查重复依赖：同一个库的多个版本被打包（用包管理器的 dedupe、overrides）
- 检查 Tree Shaking 失效的依赖

**3. 防劣化**

- CI 中设置体积预算（`size-limit` 等），超出时阻止合并
- 每次 PR 对比产物体积变化

**口述模板（30–60 秒）**

1. 一句话结论：先分析产物组成，再做懒加载、依赖替换、按需导入和合理分包。
2. 关键机制：分包提高缓存命中，懒加载减少首屏体积。
3. 对比或注意点：治理的关键是在 CI 中防止体积劣化。
4. 可追问时补充：重复依赖经常被忽视，需要定期检查。

**常见追问**

- 首屏体积优化到什么程度？→ 根据业务指标（LCP、TTI）而不是单纯看体积。

---

## 三、工程与架构

### Q18. 怎样设计一个高质量的 Vite 插件？

- **难度**: 高级
- **标签**: 插件

**答案**

1. **选对钩子**：模块转换用 transform，生成模块用虚拟模块，开发服务器能力用 configureServer，HTML 处理用 transformIndexHtml
2. **性能**：
   - 使用钩子过滤器
   - 用 `es-module-lexer` 或字符串检查代替完整 AST 解析
   - 缓存转换结果（按内容 hash）
3. **正确性**：
   - 转换后返回 source map（用 `magic-string`）
   - 区分开发和构建（`apply` 或 `configResolved` 中读取 `command`）
   - 支持多环境（`this.environment`、`applyToEnvironment`）
4. **HMR**：数据变化时用 `hotUpdate` 使相关模块失效，或发送自定义事件
5. **配置**：提供合理默认值，用 `include` / `exclude` 让用户控制范围（`createFilter`）
6. **兼容性**：
   - 声明 `peerDependencies` 中支持的 Vite 版本
   - 跨构建工具需求可以用 unplugin
7. **测试**：用 Vite 的 `build` / `createServer` API 写集成测试

**口述模板（30–60 秒）**

1. 一句话结论：好的插件要选对钩子、性能好、source map 正确、支持 HMR 和多环境。
2. 关键机制：钩子过滤器和缓存保证性能，magic-string 保证 source map。
3. 对比或注意点：要区分开发和构建，开发时输出钩子不调用。
4. 可追问时补充：需要同时支持 webpack 时用 unplugin。

**常见追问**

- `hotUpdate` 和 `handleHotUpdate` 的区别？→ hotUpdate 是 Vite 6 新增的钩子，按环境调用，支持文件创建和删除事件；handleHotUpdate 是旧钩子，只针对 client 环境。

---

### Q19. 大型 webpack 项目迁移到 Vite，有哪些难点？

- **难度**: 高级
- **标签**: 迁移

**答案**

| 难点 | 处理 |
|------|------|
| `require` / `require.context` | 改为 `import` / `import.meta.glob` |
| `process.env.XXX` | 改为 `import.meta.env.VITE_XXX`，第三方库用 define |
| 自定义 loader | 改写为 Vite 插件的 transform，或找替代插件 |
| webpack 插件 | 找 Vite 替代品，或用 unplugin 重写 |
| 模块联邦 | `@module-federation/vite`，需要充分验证 |
| CSS `~` 前缀导入 | 改为 alias 或直接路径 |
| HTML 模板 | index.html 移到根目录，改为 module script |
| 隐式扩展名、大小写不敏感 | 补全路径 |
| 循环依赖 | 开发时 ESM 原生执行，循环依赖暴露更多问题 |

**策略**：

1. 评估 webpack 特有能力的依赖程度；依赖多时考虑 Rspack（迁移成本更低）
2. 先在新分支让开发环境跑起来，再处理构建
3. 两套构建并存一段时间，逐页面对比
4. 灰度上线，监控错误率

详见 [系统设计 S1](50-system-design.md)。

**口述模板（30–60 秒）**

1. 一句话结论：难点在于 CJS 写法、环境变量、自定义 loader 和插件、模块联邦。
2. 关键机制：逐项替换为 ESM、import.meta.env、Vite 插件。
3. 对比或注意点：依赖 webpack 特有能力多的项目，迁移 Rspack 更划算。
4. 可追问时补充：两套构建并存、逐页面对比、灰度上线。

**常见追问**

- 迁移收益怎么衡量？→ 开发启动时间、HMR 耗时、构建时间、开发者满意度。

---

### Q20. 大项目升级 Vite 8 应该怎么规划？

- **难度**: 高级
- **标签**: 迁移, 版本

**答案**

1. **前置**：升级到 Node 20.19+ / 22.12+；先升级到最新的 Vite 7
2. **试用**：用 `rolldown-vite` 替换 vite 包，提前验证 Rolldown（这一步不改变其他 Vite 行为）
3. **插件审计**：
   - 检查插件是否使用不支持的钩子
   - 检查插件是否依赖 esbuild（`transformWithEsbuild` 已废弃）
   - 检查是否使用 `manualChunks` 对象形式
4. **配置迁移**：rollupOptions → rolldownOptions，esbuild → oxc，manualChunks → codeSplitting
5. **行为验证**：
   - CJS 默认导入的结果（重点）
   - `build.target` 变化导致的兼容性
   - 原生装饰器的降级
   - CSS 压缩结果
6. **对比产物**：体积、chunk 数量、关键页面的 E2E 测试
7. **灰度上线**

**口述模板（30–60 秒）**

1. 一句话结论：先升 Node 和 Vite 7，通过 rolldown-vite 渐进验证，再升级 Vite 8。
2. 关键机制：重点审计插件兼容性和 CJS 互操作变化。
3. 对比或注意点：兼容层会转换大部分配置，但行为变化需要测试验证。
4. 可追问时补充：对比产物和 E2E 测试是上线前的关键。

**常见追问**

- 有第三方插件不兼容怎么办？→ 查看插件是否有新版本；没有的话 fork 修复，或用 Rolldown 内置能力替代。

---

### Q21. 基于 Vite 设计 SSR 框架需要考虑什么？

- **难度**: 高级
- **标签**: SSR, 架构

**答案**

1. **开发**：中间件模式；用 Module Runner 加载服务端入口，支持服务端代码 HMR
2. **构建**：客户端和服务端两次构建（或使用 Environment API 的 `builder.buildApp` 统一构建）
3. **资源映射**：客户端 manifest 记录每个路由对应的 JS 和 CSS，渲染时输出 preload 和 stylesheet 标签，避免样式闪烁
4. **依赖处理**：`ssr.noExternal` 处理需要转换的依赖；部署到边缘时全部打包（`noExternal: true`）
5. **流式渲染**：React `renderToPipeableStream` / `renderToReadableStream`
6. **数据获取与注水**：服务端获取的数据序列化到 HTML，客户端注水时复用
7. **多运行时**：通过 Environment API 支持 Node 和边缘运行时
8. **错误处理**：`vite.ssrFixStacktrace(e)` 修正开发时的错误堆栈

实际例子：Nuxt、SvelteKit、React Router（框架模式）、Astro 都基于 Vite。

详见 [系统设计 S4](50-system-design.md)。

**口述模板（30–60 秒）**

1. 一句话结论：需要考虑开发加载、双端构建、资源映射、依赖处理、流式渲染和多运行时。
2. 关键机制：manifest 把路由映射到资源，Module Runner 支持服务端 HMR。
3. 对比或注意点：边缘运行时需要把依赖全部打包。
4. 可追问时补充：Environment API 让多运行时的框架实现更简单。

**常见追问**

- 为什么需要 manifest？→ 服务端渲染时需要知道当前页面对应哪些带 hash 的 JS 和 CSS 文件。

---

### Q22. Vite 项目怎么做微前端？

- **难度**: 高级
- **标签**: 微前端

**答案**

**问题**：qiankun 等方案通过 `fetch` + `eval` 执行子应用脚本，并用 Proxy 沙箱隔离全局变量。但 Vite 产物是 ESM，`import` 语句不能在 eval 中执行，开发时更是大量 ESM 请求。

**方案**：

| 方案 | 说明 |
|------|------|
| `vite-plugin-qiankun` 等社区插件 | 通过动态 import 加载子应用，生命周期挂到全局；JS 沙箱能力减弱 |
| 无界（wujie） | 用 iframe 运行 JS、Web Component 渲染 DOM，天然支持 ESM |
| micro-app | 支持 ESM 子应用（需要关闭部分沙箱） |
| 模块联邦 | `@module-federation/vite`，共享依赖、运行时加载远程模块 |
| iframe | 隔离最彻底，通信和体验较差 |

**选型**：

- 需要强隔离：无界、iframe
- 技术栈统一、需要共享依赖：模块联邦
- 已有 qiankun 体系：社区插件，接受沙箱能力减弱

**口述模板（30–60 秒）**

1. 一句话结论：Vite 的 ESM 产物与 qiankun 的 eval 沙箱不兼容。
2. 关键机制：可以用社区插件、无界、模块联邦等方案。
3. 对比或注意点：选择取决于隔离需求和是否需要共享依赖。
4. 可追问时补充：无界用 iframe 执行 JS，天然支持 ESM。

**常见追问**

- 为什么 ESM 不能 eval？→ `import` / `export` 是模块语法，只能在模块上下文中执行，eval 执行的是脚本。

---

### Q23. Monorepo 中使用 Vite 的最佳实践？

- **难度**: 高级
- **标签**: Monorepo

**答案**

1. **包管理**：pnpm workspace，`workspace:*` 协议
2. **内部包导出源码**：开发时直接引用源码（通过 `exports` 中的自定义条件或 `resolve.conditions`），HMR 跨包生效，不需要先构建内部包
3. **共享配置**：抽取 `@company/vite-config` 包，各应用用 `mergeConfig` 扩展
4. **依赖单例**：React 等用 `resolve.dedupe`，并在根目录统一版本
5. **构建编排**：Turborepo / Nx 按依赖拓扑构建并缓存
6. **测试**：Vitest 的 `projects`（旧称 workspace）统一运行多个包的测试
7. **类型**：TS 项目引用（`references`）+ `tsc -b`

```json
{
  "exports": {
    ".": {
      "development": "./src/index.ts",
      "default": "./dist/index.js"
    }
  }
}
```

**口述模板（30–60 秒）**

1. 一句话结论：开发时引用内部包源码，共享配置，依赖单例，用 Turborepo 编排构建。
2. 关键机制：通过 exports 条件让开发时指向源码，构建时指向产物。
3. 对比或注意点：React 等库要保证单例。
4. 可追问时补充：Vitest projects 可以统一运行所有包的测试。

**常见追问**

- 内部包导出源码有什么问题？→ 应用的构建需要能处理这些源码（TS、JSX），并且内部包的依赖需要正确声明。

---

### Q24. 2026 年怎么做前端构建工具选型？

- **难度**: 高级
- **标签**: 选型

**答案**

| 场景 | 推荐 |
|------|------|
| 新的 SPA / 组件库 / 中后台 | Vite |
| 使用 Next.js | Turbopack（框架内置） |
| 使用 Nuxt / SvelteKit / Astro / React Router 框架模式 | Vite（框架内置） |
| 存量 webpack 大项目，提速需求强 | Rspack / Rsbuild（迁移成本低） |
| 重度依赖模块联邦 | Rspack 或 webpack |
| 纯 TS 库 | tsdown |

**评估维度**：

- 开发体验：启动、HMR、首次加载
- 构建速度与产物质量
- 生态与插件可用性
- 迁移成本与团队熟悉度
- 长期维护：社区活跃度、商业支持

**口述模板（30–60 秒）**

1. 一句话结论：新项目选 Vite，框架项目跟随框架，存量 webpack 项目选 Rspack。
2. 关键机制：主要看迁移成本、生态和团队熟悉度。
3. 对比或注意点：模块联邦重度用户目前 Rspack 更成熟。
4. 可追问时补充：构建工具在向 Rust 实现收敛，差距在缩小。

**常见追问**

- 为什么不推荐所有项目都迁移到 Vite？→ 迁移有成本，存量项目如果瓶颈不在构建，收益不明显。

---

### Q25. 怎么理解 VoidZero 的统一工具链？

- **难度**: 高级
- **标签**: 生态

**答案**

VoidZero 的目标是提供统一的、Rust 驱动的 JS 工具链：

| 工具 | 定位 |
|------|------|
| Vite | 开发服务器与构建 |
| Rolldown | 打包器（取代 esbuild + Rollup） |
| Oxc | 解析、转换、压缩、lint（oxlint）、格式化（oxfmt）、解析器 |
| Vitest | 测试 |
| tsdown | 库打包 |

**好处**：

- 共享同一个解析器和 AST，避免重复解析
- 配置统一、行为一致
- 性能大幅提升

**背景**：2026 年 7 月 VoidZero 加入 Cloudflare，官方表示 Vite 保持开源和独立治理（以官方公告为准）。

面试中可以结合这个趋势讲：构建工具从「各种工具拼接」走向「统一工具链」。

**口述模板（30–60 秒）**

1. 一句话结论：VoidZero 在做统一的 Rust 前端工具链。
2. 关键机制：Vite、Rolldown、Oxc、Vitest 共享解析器和配置，减少重复工作。
3. 对比或注意点：以前需要 Babel、ESLint、Prettier、esbuild、Rollup 等多个工具拼接。
4. 可追问时补充：oxlint 可以替代 ESLint 的大部分规则，速度快很多。

**常见追问**

- oxlint 能完全替代 ESLint 吗？→ 覆盖了大部分常用规则，但部分插件生态（自定义规则）仍需要 ESLint。

---

### Q26. Vite 项目有哪些安全风险？

- **难度**: 高级
- **标签**: 安全

**答案**

| 风险 | 说明 | 措施 |
|------|------|------|
| 环境变量泄露 | `VITE_` 变量会进入产物 | 密钥只放在服务端；审查 `envPrefix` |
| source map 泄露源码 | 公开 `.map` 文件 | 使用 `'hidden'`，上传监控平台后删除 |
| 开发服务器暴露 | 任意文件读取漏洞 | 不暴露到公网，及时升级，配置 `fs.deny` 和 `allowedHosts` |
| 供应链攻击 | 依赖或插件被投毒 | 锁定版本、审计依赖、限制 postinstall 脚本 |
| 配置文件执行任意代码 | vite.config 和插件在 Node 中运行 | 审查第三方插件 |
| `define` 注入 | 构建时替换可能注入恶意代码 | 只使用可信数据 |

**口述模板（30–60 秒）**

1. 一句话结论：主要风险是环境变量泄露、source map 泄露、开发服务器暴露和供应链攻击。
2. 关键机制：客户端变量和 source map 都会进入产物，开发服务器可以读取本地文件。
3. 对比或注意点：插件和配置文件以完整权限在 Node 中运行。
4. 可追问时补充：关注 Vite 的安全公告，及时升级补丁版本。

**常见追问**

- CSP 怎么配合 Vite？→ 生产产物没有内联脚本问题；开发时注入了内联代码，可以通过 `html.cspNonce` 配置 nonce。
