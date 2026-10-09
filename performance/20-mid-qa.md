# 前端性能优化中级 Q&A

> 目标：能把每个核心指标拆解为子阶段并针对性优化，理解关键渲染路径、渲染性能、缓存与预加载、框架层优化和内存问题。能回答「为什么」和「怎么定位」。

---

## 一、LCP 优化

### Q1. 怎么拆解 LCP？每个阶段怎么优化？

- **难度**: 中级
- **标签**: LCP

**答案**

LCP 可以拆为四个阶段（以 LCP 元素是图片为例）：

| 阶段 | 含义 | 理想占比 | 优化手段 |
|------|------|----------|----------|
| **TTFB** | 导航开始到 HTML 首字节 | ~40% | CDN、缓存、减少重定向、服务端优化、103 Early Hints |
| **资源加载延迟**（Load Delay） | TTFB 到 LCP 资源开始加载 | < 10% | 让资源在 HTML 中可被发现、preload、`fetchpriority="high"` |
| **资源加载耗时**（Load Duration） | 资源下载时间 | ~40% | 压缩、AVIF / WebP、合适的尺寸、CDN |
| **元素渲染延迟**（Render Delay） | 资源加载完到渲染出来 | < 10% | 减少阻塞渲染的 CSS / JS、避免 JS 渲染才插入元素 |

LCP 元素是文本时，没有资源加载阶段，主要看 TTFB 和渲染延迟（字体、阻塞资源）。

DevTools Performance 面板的 LCP Insights 和 `web-vitals/attribution` 都能直接给出这四段数据。

**口述模板（30–60 秒）**

1. 一句话结论：LCP 拆为 TTFB、资源加载延迟、资源加载耗时、元素渲染延迟四段。
2. 关键机制：先看哪段占比最高，再针对性优化。
3. 对比或注意点：很多项目的瓶颈在加载延迟——LCP 图片要等 JS 执行后才被发现。
4. 可追问时补充：理想情况下两个延迟阶段都应小于 10%。

**常见追问**

- 怎么判断是哪个阶段的问题？→ 用 web-vitals 的 attribution 上报四段耗时，看线上的 P75 分布。

---

### Q2. LCP 图片应该怎么优化？

- **难度**: 中级
- **标签**: LCP, 图片

**答案**

1. **可被预加载扫描器发现**：图片写在 HTML 中的 `<img src>`，而不是 JS 渲染后才插入或使用 CSS 背景图（CSS 下载解析后才发现）
2. **提高优先级**：`fetchpriority="high"`（图片默认是低优先级，进入视口后才提升）
3. **不要懒加载**：`loading="lazy"` 会延迟加载
4. **必要时 preload**：图片地址在 CSS 中或由 JS 决定时

```html
<link rel="preload" as="image" href="hero.avif" fetchpriority="high"
      imagesrcset="hero-480.avif 480w, hero-960.avif 960w" imagesizes="100vw" />
```

5. **减小体积**：AVIF / WebP、合适尺寸、合理的压缩质量
6. **同域或 preconnect**：避免额外的连接建立
7. **SSR**：CSR 页面的 LCP 图片要等 JS 下载、执行、请求数据之后才渲染

**口述模板（30–60 秒）**

1. 一句话结论：LCP 图片要尽早被发现、以高优先级加载、体积尽量小。
2. 关键机制：写在 HTML 中让预加载扫描器发现，加 fetchpriority high，不要懒加载。
3. 对比或注意点：CSS 背景图和 JS 插入的图片发现得晚，需要 preload。
4. 可追问时补充：CSR 页面的 LCP 往往需要 SSR 才能根本解决。

**常见追问**

- 什么是预加载扫描器？→ 浏览器在主解析器被阻塞时，用一个轻量的扫描器提前扫描 HTML 中的资源并开始下载。

---

### Q3. TTFB 慢怎么优化？

- **难度**: 中级
- **标签**: TTFB, 网络

**答案**

TTFB 包括：重定向 + Service Worker 启动 + DNS + TCP + TLS + 请求 + 服务端处理。

| 原因 | 优化 |
|------|------|
| 重定向（http → https、短链） | 减少重定向，直接使用最终地址；HSTS |
| 网络距离远 | CDN、边缘节点 |
| 连接建立慢 | HTTP/3（0-RTT / 1-RTT）、TLS 1.3、连接复用 |
| 服务端处理慢 | 缓存渲染结果、优化数据库查询、减少串行接口调用 |
| SSR 渲染慢 | 页面级缓存、组件级缓存、流式渲染（先发送 head） |
| 动态内容无法缓存 | 边缘缓存 + stale-while-revalidate |

**103 Early Hints**：服务端还在生成页面时，先返回 103 响应携带 `Link: preload / preconnect` 头，浏览器提前加载关键资源。

**口述模板（30–60 秒）**

1. 一句话结论：TTFB 由重定向、网络连接和服务端处理组成。
2. 关键机制：减少重定向，CDN 就近访问，缓存渲染结果，流式输出。
3. 对比或注意点：SSR 会增加服务端处理时间，需要缓存或流式渲染。
4. 可追问时补充：103 Early Hints 可以利用服务端的思考时间提前加载资源。

**常见追问**

- 流式渲染为什么能改善 TTFB？→ 不需要等整个页面渲染完成，先发送 head 和首屏 shell，浏览器可以提前加载 CSS 和 JS。

---

### Q4. 什么是关键渲染路径？哪些资源会阻塞渲染？

- **难度**: 中级
- **标签**: 关键渲染路径

**答案**

**关键渲染路径（Critical Rendering Path）**：HTML → DOM，CSS → CSSOM，合并生成渲染树 → 布局 → 绘制。

**阻塞情况**：

| 资源 | 阻塞解析 | 阻塞渲染 |
|------|---------|---------|
| `<link rel="stylesheet">`（head 中） | 否（但会阻塞后续脚本执行） | **是** |
| 普通 `<script>` | **是** | 是（间接） |
| `<script defer>` / `type="module"` | 否 | 否 |
| `<script async>` | 执行时会中断 | 否 |
| 图片 | 否 | 否 |

为什么 CSS 会阻塞脚本执行：脚本可能读取样式（`getComputedStyle`），所以必须等 CSSOM 构建完成。

**优化**：

- 关键 CSS 内联，其余 CSS 异步加载
- 按媒体查询拆分 CSS：`<link rel="stylesheet" media="print">` 不阻塞渲染
- 脚本使用 `defer`
- 减少关键资源的数量、体积和往返次数

**口述模板（30–60 秒）**

1. 一句话结论：关键渲染路径是从 HTML 和 CSS 到首次渲染的过程。
2. 关键机制：CSS 阻塞渲染，同步脚本阻塞解析，CSS 还会阻塞后续脚本执行。
3. 对比或注意点：defer 和 module 脚本不阻塞解析。
4. 可追问时补充：优化目标是减少关键资源的数量、体积和往返次数。

**常见追问**

- 为什么 CSS 要阻塞渲染？→ 否则会先渲染无样式内容，再闪烁成有样式内容（FOUC）。

---

### Q5. CSR、SSR、SSG、流式渲染对性能指标有什么影响？

- **难度**: 中级
- **标签**: 渲染模式

**答案**

| 模式 | TTFB | FCP / LCP | INP | 适用场景 |
|------|------|-----------|-----|----------|
| CSR | 快（静态 HTML） | 慢：要等 JS 下载执行、请求数据 | 水合前不存在问题，但 JS 体积大 | 中后台 |
| SSR | 慢（服务端渲染） | 快：HTML 直接包含内容 | 水合期间可能卡顿 | 内容站、电商 |
| SSG | 最快（CDN） | 最快 | 同 SSR | 文档、博客、营销页 |
| 流式 SSR | 快（先发送 shell） | 快，内容逐步出现 | 可选择性水合 | 有慢数据的页面 |
| ISR / 边缘缓存 | 快 | 快 | 同 SSR | 内容更新不频繁的动态页面 |

**SSR 不一定更快**：

- 服务端渲染慢时，TTFB 增加可能抵消收益
- 水合需要下载执行完整的 JS，期间页面看起来能用但无法交互
- 优化方向：流式渲染、部分水合、React Server Components 减少客户端 JS

**口述模板（30–60 秒）**

1. 一句话结论：SSR 和 SSG 改善 LCP，CSR 的 LCP 受 JS 和数据请求链影响。
2. 关键机制：SSR 的 HTML 直接包含内容，但会增加 TTFB 和水合成本。
3. 对比或注意点：SSR 不一定更快，服务端慢或水合重时效果有限。
4. 可追问时补充：流式渲染和 RSC 进一步优化 TTFB 和客户端 JS 体积。

**常见追问**

- 水合期间为什么交互没反应？→ 事件处理函数还没有绑定，或主线程正在执行水合的长任务。

---

### Q6. 浏览器怎么决定资源加载优先级？怎么调整？

- **难度**: 中级
- **标签**: 资源优先级

**答案**

Chrome 的默认优先级（简化）：

| 资源 | 默认优先级 |
|------|-----------|
| HTML、head 中的 CSS、字体 | Highest |
| head 中的同步脚本、preload | High |
| 视口内的图片 | 初始 Low，布局后进入视口时提升为 High |
| async / defer 脚本 | Low |
| prefetch | Lowest |

**调整方式**：

- `fetchpriority="high" | "low" | "auto"`：用于 `<img>`、`<link>`、`<script>`、`fetch()`
- `preload`：提前发现
- 降低不重要资源的优先级：轮播图非首张、首屏外的图片设置 `fetchpriority="low"`

```js
// 非关键接口降低优先级
fetch('/api/recommend', { priority: 'low' })
```

**口述模板（30–60 秒）**

1. 一句话结论：浏览器根据资源类型和位置分配优先级，可以用 fetchpriority 调整。
2. 关键机制：图片初始是低优先级，要等布局确认在视口内才提升。
3. 对比或注意点：preload 解决发现得晚的问题，fetchpriority 解决优先级低的问题。
4. 可追问时补充：fetch 也支持 priority 参数。

**常见追问**

- 所有关键资源都设置 high 可以吗？→ 不行，全部高优先级等于没有优先级，会互相抢占带宽。

---

## 二、INP 优化

### Q7. 怎么拆解 INP？

- **难度**: 中级
- **标签**: INP

**答案**

一次交互的延迟分为三段：

```text
用户点击 ──[输入延迟]── 事件处理开始 ──[处理耗时]── 处理结束 ──[呈现延迟]── 下一帧绘制
```

| 阶段 | 原因 | 优化 |
|------|------|------|
| **输入延迟**（Input Delay） | 主线程正忙于其他任务（长任务、第三方脚本、水合） | 拆分长任务，延迟非关键脚本 |
| **处理耗时**（Processing Duration） | 事件回调本身太慢 | 只做必要的同步工作，其余延后或移到 Worker |
| **呈现延迟**（Presentation Delay） | 样式计算、布局、绘制成本高；rAF 回调多 | 减少 DOM 规模，避免大范围重渲染，使用 `content-visibility` |

**口述模板（30–60 秒）**

1. 一句话结论：INP 拆为输入延迟、处理耗时和呈现延迟三段。
2. 关键机制：输入延迟来自其他任务占用主线程，处理耗时来自事件回调，呈现延迟来自渲染成本。
3. 对比或注意点：三段的优化方法不同，必须先定位是哪一段。
4. 可追问时补充：web-vitals 的 attribution 能直接给出三段耗时。

**常见追问**

- 页面加载期间的点击为什么特别慢？→ 加载期间有大量脚本执行和水合，输入延迟很高。

---

### Q8. 什么是长任务？怎么拆分？

- **难度**: 中级
- **标签**: INP, 长任务

**答案**

**长任务**：主线程上执行超过 **50ms** 的任务。期间浏览器无法响应输入、无法渲染。

**拆分方法**（核心思想：**让出主线程**，让浏览器有机会处理输入和渲染）：

```js
// 方式一：scheduler.yield()（Chromium 129+、Firefox，需要降级）
async function yieldToMain() {
  if (globalThis.scheduler?.yield) return scheduler.yield()
  return new Promise((resolve) => setTimeout(resolve, 0))
}

async function processItems(items) {
  let lastYield = performance.now()
  for (const item of items) {
    process(item)
    // 每处理约 50ms 让出一次，而不是每项都让出，减少调度开销
    if (performance.now() - lastYield > 50) {
      await yieldToMain()
      lastYield = performance.now()
    }
  }
}
```

| 方式 | 特点 |
|------|------|
| `scheduler.yield()` | 让出后**优先继续**（续体排在队列前面），不会被其他任务插队太久 |
| `setTimeout(fn, 0)` | 兼容性好，但续体排到队尾，嵌套多层后有 4ms 限制 |
| `scheduler.postTask(fn, { priority })` | 指定优先级：`user-blocking`、`user-visible`、`background` |
| `requestIdleCallback` | 空闲时执行，适合非紧急任务 |
| Web Worker | 完全移出主线程 |

**口述模板（30–60 秒）**

1. 一句话结论：长任务是超过 50 毫秒的任务，拆分的核心是定期让出主线程。
2. 关键机制：scheduler.yield 让出后优先继续，setTimeout 兼容性好但会排到队尾。
3. 对比或注意点：不需要每一项都让出，按时间片让出可以减少开销。
4. 可追问时补充：计算密集型任务直接移到 Worker。

**常见追问**

- 为什么是 50ms？→ RAIL 模型要求 100ms 内响应输入，留 50ms 给浏览器处理输入和渲染。

---

### Q9. 线上 INP 很差，怎么定位？

- **难度**: 中级
- **标签**: INP, 排查

**答案**

**1. 线上数据归因**

```js
import { onINP } from 'web-vitals/attribution'

onINP(({ value, attribution }) => {
  report({
    value,
    target: attribution.interactionTarget,      // 交互的元素选择器
    type: attribution.interactionType,          // pointer / keyboard
    inputDelay: attribution.inputDelay,
    processing: attribution.processingDuration,
    presentation: attribution.presentationDelay,
    loaf: attribution.longAnimationFrameEntries, // 相关的长动画帧及脚本来源
  })
})
```

按页面、交互元素、设备聚合，找出 P75 最差的交互。

**2. LoAF（Long Animation Frames）**：给出长帧中各脚本的来源（文件、函数、执行耗时），可以定位到第三方脚本还是业务代码。

**3. 本地复现**：Performance 面板开启 CPU 节流，录制相同交互，查看 Interactions 轨道和对应的长任务。

**口述模板（30–60 秒）**

1. 一句话结论：先用线上归因数据找到最差的交互，再本地复现定位代码。
2. 关键机制：web-vitals attribution 给出交互元素和三段耗时，LoAF 给出脚本来源。
3. 对比或注意点：开发机性能好，本地必须开启 CPU 节流才能复现。
4. 可追问时补充：按设备分组，低端设备通常是 P75 的主要贡献者。

**常见追问**

- Long Tasks API 和 LoAF 有什么区别？→ Long Tasks 只告诉你有长任务，LoAF 以帧为单位，包含渲染时间和脚本归因，更适合定位 INP。

---

### Q10. 事件处理函数中应该怎么写才能提升响应速度？

- **难度**: 中级
- **标签**: INP

**答案**

**原则**：先更新用户能看到的反馈，再做其他工作。

```js
button.addEventListener('click', async () => {
  // 1. 立即给出视觉反馈
  button.classList.add('loading')

  // 2. 让出主线程，让浏览器先绘制反馈
  await scheduler.yield?.() ?? new Promise((r) => setTimeout(r, 0))

  // 3. 再做重活
  const result = heavyCompute()
  render(result)

  // 4. 非关键工作（埋点、日志）放到空闲时
  requestIdleCallback(() => track('click'))
})
```

**其他要点**：

- 输入框高频事件：防抖，或用 React 的 `useDeferredValue`
- 避免在事件处理中读写交替导致强制同步布局
- 埋点、日志不要在交互路径上同步执行
- 大量 DOM 更新时只更新可见部分

**口述模板（30–60 秒）**

1. 一句话结论：先给出视觉反馈，再让出主线程，最后做重活。
2. 关键机制：INP 只计算到下一帧绘制，先绘制反馈就能让 INP 变好。
3. 对比或注意点：埋点和日志等非关键工作要延后。
4. 可追问时补充：这不是作弊，用户确实更早看到了反馈。

**常见追问**

- React 中怎么实现？→ 用 `useTransition` 把非紧急的更新标记为过渡，紧急的反馈状态先渲染。

---

### Q11. DOM 规模为什么会影响性能？

- **难度**: 中级
- **标签**: 渲染, INP

**答案**

DOM 节点多时：

- **样式计算**：选择器匹配和样式计算的成本与节点数相关
- **布局**：任意布局变化可能需要重新计算大量节点
- **内存**：每个节点都有内存开销
- **呈现延迟**：交互后的渲染成本高，INP 变差
- **查询**：`querySelectorAll` 变慢

Lighthouse 建议总节点数不超过约 1400 个、深度不超过 32 层（经验值）。

**优化**：

- 虚拟列表：只渲染可见区域
- `content-visibility: auto`：跳过屏幕外内容的渲染
- 懒渲染：折叠面板、标签页的内容在展开时才渲染
- 减少包装层级

**口述模板（30–60 秒）**

1. 一句话结论：DOM 越大，样式计算、布局和渲染成本越高。
2. 关键机制：每次交互后的渲染都可能涉及大量节点，导致呈现延迟增加。
3. 对比或注意点：虚拟列表和 content-visibility 可以减少需要渲染的节点。
4. 可追问时补充：隐藏的标签页内容也应该懒渲染。

**常见追问**

- `display: none` 的节点有成本吗？→ 不参与布局和绘制，但仍然占用内存，样式计算也有一定成本。

---

## 三、CLS 优化

### Q12. CLS 怎么系统地优化？

- **难度**: 中级
- **标签**: CLS

**答案**

| 原因 | 解决 |
|------|------|
| 图片、视频没有尺寸 | `width` / `height` 属性或 `aspect-ratio` |
| 广告、嵌入内容 | 预留固定尺寸的容器（`min-height`） |
| 动态插入内容（通知条、Cookie 横幅） | 放在视口外、使用覆盖层（`position: fixed`），或预留空间 |
| 字体替换 | `font-display: optional` 或用 `size-adjust` 调整备用字体 |
| 动画使用布局属性 | 改用 `transform` |
| 骨架屏尺寸与内容不一致 | 骨架屏使用与真实内容相同的尺寸 |
| 懒加载的内容 | 占位符预留高度 |

**定位**：

- DevTools Performance 面板的 Layout Shifts 轨道
- `web-vitals/attribution` 的 `largestShiftTarget`、`largestShiftSource`

**口述模板（30–60 秒）**

1. 一句话结论：CLS 的核心是给所有异步出现的内容预留空间。
2. 关键机制：图片设置尺寸，广告预留容器，通知用覆盖层，字体调整备用字体尺寸。
3. 对比或注意点：transform 动画不会产生布局偏移。
4. 可追问时补充：用 attribution 找到偏移最大的元素。

**常见追问**

- 为什么 transform 动画不算布局偏移？→ transform 不改变布局，元素在文档流中的位置不变。

---

## 四、渲染性能

### Q13. 什么是强制同步布局？怎么避免？

- **难度**: 中级
- **标签**: 渲染

**答案**

**正常流程**：JS 修改样式 → JS 执行完 → 浏览器在下一帧统一计算样式和布局。

**强制同步布局（Forced Synchronous Layout）**：JS 修改样式后**立即读取**布局属性，浏览器必须立即计算布局才能返回正确的值。

**布局抖动（Layout Thrashing）**：在循环中读写交替，每次循环都触发一次布局：

```js
// 不好：每次循环都强制布局
for (const el of items) {
  el.style.width = box.offsetWidth + 'px' // 读（强制布局） → 写（使布局失效）
}

// 好：先读后写
const width = box.offsetWidth
for (const el of items) {
  el.style.width = width + 'px'
}
```

**避免方法**：

- 读写分离：先批量读取，再批量写入
- 把读取放在 rAF 回调开始时（上一帧的布局仍然有效）
- 使用 FastDOM 等库调度读写
- 使用 ResizeObserver / IntersectionObserver 代替主动读取

**口述模板（30–60 秒）**

1. 一句话结论：修改样式后立即读取布局属性，会强制浏览器同步计算布局。
2. 关键机制：循环中读写交替会导致每次循环都重新布局，就是布局抖动。
3. 对比或注意点：解决方法是读写分离。
4. 可追问时补充：Performance 面板中 Layout 带红色警告就是强制同步布局。

**常见追问**

- 读取 `getComputedStyle` 也会强制布局吗？→ 会强制样式计算；读取布局相关的值（如 width）时还会强制布局。

---

### Q14. 什么是合成层？`will-change` 应该怎么用？

- **难度**: 中级
- **标签**: 渲染, 合成

**答案**

浏览器把页面分成多个**层**，每个层单独绘制和光栅化，然后由合成线程合成。提升为独立合成层的元素：

- 修改 `transform`、`opacity` 时**只需要重新合成**，不需要重排和重绘
- 动画可以在合成线程上运行，主线程卡顿时也流畅

**触发合成层**：`will-change: transform`、3D transform、`<video>`、`<canvas>`、有合成动画的元素、`position: fixed`（部分情况）等。

**`will-change` 的正确用法**：

- 在动画开始前添加，结束后移除
- 不要给大量元素添加

**层爆炸**：层过多时，每个层都占用 GPU 内存，合成成本增加，反而更卡。另外，元素与合成层重叠时，也可能被隐式提升为合成层。

DevTools 的 Layers 面板可以查看层和内存占用。

**口述模板（30–60 秒）**

1. 一句话结论：合成层单独光栅化，transform 和 opacity 变化只需要重新合成。
2. 关键机制：合成在合成线程中完成，不依赖主线程。
3. 对比或注意点：层过多会占用大量内存，导致层爆炸。
4. 可追问时补充：will-change 应该在动画前添加、结束后移除。

**常见追问**

- 什么是隐式合成？→ 元素在 z 轴上覆盖了一个合成层时，浏览器为了保证绘制顺序正确，也会把它提升为合成层。

---

### Q15. `content-visibility` 和 `contain` 有什么用？

- **难度**: 中级
- **标签**: 渲染, CSS

**答案**

**`contain`**：告诉浏览器元素的内部变化不影响外部，可以限制样式计算、布局和绘制的范围。

```css
.card { contain: layout paint; }
```

**`content-visibility: auto`**：屏幕外的元素**跳过渲染**（样式计算、布局、绘制），进入视口附近时再渲染：

```css
.section {
  content-visibility: auto;
  /* 未渲染时的占位尺寸，避免滚动条跳动和 CLS；auto 会记住上次渲染的尺寸 */
  contain-intrinsic-size: auto 600px;
}
```

适合长页面、长列表、复杂的卡片列表。比虚拟列表简单，但 DOM 节点仍然存在（内存不减少）。

**注意**：屏幕外内容仍可以被页面内查找（Ctrl+F）和无障碍工具访问；需要设置 `contain-intrinsic-size`，否则滚动条会跳动。

**口述模板（30–60 秒）**

1. 一句话结论：content-visibility auto 让屏幕外内容跳过渲染。
2. 关键机制：基于 contain 隔离元素，进入视口时才渲染。
3. 对比或注意点：比虚拟列表简单，但 DOM 仍然存在，内存不减少。
4. 可追问时补充：要设置 contain-intrinsic-size 作为占位尺寸。

**常见追问**

- 和虚拟列表怎么选？→ 几百到几千个复杂节点用 content-visibility；上万条数据用虚拟列表。

---

### Q16. 长列表怎么优化？

- **难度**: 中级
- **标签**: 渲染, 列表

**答案**

| 方案 | 原理 | 适合 |
|------|------|------|
| **分页 / 加载更多** | 每次只加载一部分 | 大部分业务场景（最简单） |
| **分片渲染** | 数据一次加载，分批渲染（每帧渲染一批） | 数据量中等，需要全部展示 |
| **虚拟列表** | 只渲染可见区域及缓冲区的节点 | 万级以上数据 |
| **`content-visibility`** | 跳过屏幕外节点的渲染 | 千级复杂节点 |

**虚拟列表原理**：

- 容器高度 = 总条数 × 行高（撑开滚动条）
- 根据 `scrollTop` 计算可见区域的起止索引
- 只渲染这些项，通过 `transform: translateY` 定位
- 不定高时：先预估高度，渲染后测量真实高度并缓存，更新位置

成熟实现：`@tanstack/react-virtual`、`react-window`、`vue-virtual-scroller`。

手写见 [手写 H3](40-coding-handwritten.md)。

**口述模板（30–60 秒）**

1. 一句话结论：长列表优先分页，必须全部展示时用虚拟列表或分片渲染。
2. 关键机制：虚拟列表只渲染可见区域的节点，用占位撑开滚动高度。
3. 对比或注意点：不定高的虚拟列表需要预估加测量。
4. 可追问时补充：虚拟列表的缺点是 Ctrl+F 搜不到未渲染的内容。

**常见追问**

- 快速滚动时出现白屏怎么办？→ 增加缓冲区（overscan），或者在滚动时显示占位。

---

### Q17. `requestAnimationFrame` 和 `requestIdleCallback` 有什么区别？

- **难度**: 中级
- **标签**: 调度

**答案**

| 对比项 | `requestAnimationFrame` | `requestIdleCallback` |
|--------|------------------------|----------------------|
| 执行时机 | 每帧渲染前 | 帧结束后有空闲时间时 |
| 频率 | 与屏幕刷新率一致 | 不确定，主线程忙时可能很久不执行 |
| 用途 | 视觉更新、动画、读取布局 | 非紧急任务：埋点、预加载、缓存清理 |
| 页面隐藏 | 暂停 | 可能执行 |
| 兼容性 | 全部支持 | Safari 支持较晚，需要降级 |

```js
function runIdleTasks(deadline) {
  while (deadline.timeRemaining() > 0 && tasks.length) {
    tasks.shift()()
  }
  if (tasks.length) requestIdleCallback(runIdleTasks, { timeout: 2000 })
}

// timeout：最多等待 2 秒，避免主线程一直繁忙时任务永远不执行
requestIdleCallback(runIdleTasks, { timeout: 2000 })
```

**注意**：rIC 中不要修改 DOM（会导致帧意外延长），DOM 修改应该放到 rAF 中。

**口述模板（30–60 秒）**

1. 一句话结论：rAF 在每帧渲染前执行，rIC 在帧的空闲时间执行。
2. 关键机制：rAF 用于视觉更新，rIC 用于非紧急任务。
3. 对比或注意点：rIC 可能长时间不执行，要设置 timeout。
4. 可追问时补充：React 的调度器不使用 rIC，而是用 MessageChannel 实现时间切片。

**常见追问**

- React 为什么不用 rIC？→ rIC 的执行频率不稳定（可能每秒只有 20 次）、兼容性差，React 需要更可控的调度。

---

## 五、缓存与网络

### Q18. 怎么设计前端资源的缓存策略？

- **难度**: 中级
- **标签**: 缓存

**答案**

**文件名策略**：

- 构建产物使用**内容 hash**（`main.a1b2c3.js`），内容不变 hash 不变
- 稳定的依赖（框架）单独分包，业务代码修改不影响依赖的缓存

**响应头策略**：

```text
index.html               Cache-Control: no-cache
/assets/*.[hash].js      Cache-Control: public, max-age=31536000, immutable
/api/*                   Cache-Control: no-store（或按业务设置短缓存）
```

**发布策略**：

- **先上传资源，后更新 HTML**，避免 HTML 引用的资源还不存在
- 旧资源保留一段时间（或永久保留），避免用户的旧页面加载旧 chunk 失败
- CDN 上的 HTML 缓存时间要短，或在发布时主动刷新

**`stale-while-revalidate`**：`Cache-Control: max-age=60, stale-while-revalidate=3600`，过期后先返回旧内容，后台更新。

**口述模板（30–60 秒）**

1. 一句话结论：HTML 协商缓存，带 hash 的资源永久强缓存。
2. 关键机制：内容 hash 保证内容变化时 URL 变化，从而绕过缓存。
3. 对比或注意点：发布时先传资源后更新 HTML，并保留旧资源。
4. 可追问时补充：stale-while-revalidate 适合可以接受短时间旧数据的场景。

**常见追问**

- 为什么不用 query 参数 `?v=123` 做版本？→ 部分 CDN 和代理忽略 query 参数做缓存；而且只要版本号变化所有文件都会失效。

---

### Q19. Service Worker 有哪些缓存策略？

- **难度**: 中级
- **标签**: 缓存, PWA

**答案**

| 策略 | 流程 | 适合 |
|------|------|------|
| Cache First | 先查缓存，没有再请求网络 | 带 hash 的静态资源、字体 |
| Network First | 先请求网络，失败时用缓存 | HTML、需要最新的接口 |
| Stale While Revalidate | 立即返回缓存，同时后台更新 | 头像、不频繁变化的接口 |
| Network Only | 只用网络 | 支付、实时数据 |
| Cache Only | 只用缓存 | 预缓存的离线资源 |

**注意**：

- Service Worker 启动需要时间，可能增加 TTFB；可以启用 **Navigation Preload**，让 SW 启动和导航请求并行
- 更新机制：新 SW 安装后默认等待旧页面关闭才激活，`skipWaiting` 要谨慎（新旧资源混用）
- 缓存需要清理，避免存储无限增长
- 推荐使用 Workbox

**口述模板（30–60 秒）**

1. 一句话结论：SW 常用缓存优先、网络优先和 stale-while-revalidate 三种策略。
2. 关键机制：拦截 fetch 请求，按资源类型选择策略。
3. 对比或注意点：SW 启动本身有成本，要用 Navigation Preload。
4. 可追问时补充：更新机制要处理好，避免新旧资源混用。

**常见追问**

- SW 出问题导致线上故障怎么办？→ 准备一个「自杀」版本的 SW：清空缓存并注销自己。

---

### Q20. 什么是 bfcache？哪些情况会让页面无法进入 bfcache？

- **难度**: 中级
- **标签**: 缓存, 导航

**答案**

**bfcache（往返缓存）**：用户离开页面时，浏览器把**整个页面的内存状态**（包括 JS 堆）保存起来，点击后退 / 前进时**瞬间恢复**，不需要重新加载。

**常见阻止因素**：

- 使用了 `unload` 事件监听（Chrome 正在逐步废弃 unload）
- 页面有未关闭的 WebSocket、WebRTC 连接、IndexedDB 事务
- `Cache-Control: no-store` 的页面（Chrome 在部分条件下正在放宽）
- 被其他页面通过 `window.opener` 引用

**优化**：

- 用 `pagehide` / `visibilitychange` 代替 `unload`
- 在 `pagehide` 中关闭连接，`pageshow` 中恢复
- 检查：DevTools → Application → Back/forward cache 测试

```js
window.addEventListener('pageshow', (event) => {
  if (event.persisted) {
    // 从 bfcache 恢复：刷新可能过期的数据
  }
})
```

**口述模板（30–60 秒）**

1. 一句话结论：bfcache 缓存整个页面状态，后退时瞬间恢复。
2. 关键机制：页面不被销毁，而是冻结在内存中。
3. 对比或注意点：unload 监听、未关闭的连接、no-store 会阻止进入 bfcache。
4. 可追问时补充：恢复时通过 pageshow 的 persisted 判断，刷新过期数据。

**常见追问**

- bfcache 对指标有影响吗？→ 从 bfcache 恢复的页面加载几乎是瞬间的，可以显著改善真实用户的 LCP 分布。

---

### Q21. HTTP/2 和 HTTP/3 对性能有什么影响？

- **难度**: 中级
- **标签**: 网络

**答案**

**HTTP/2**：

- 多路复用：一个 TCP 连接并行传输多个请求
- 头部压缩（HPACK）
- 问题：**TCP 队头阻塞**——一个 TCP 包丢失，所有流都要等待重传

**HTTP/3（基于 QUIC / UDP）**：

- **解决 TCP 队头阻塞**：每个流独立，丢包只影响对应的流
- **更快的连接建立**：传输和加密握手合并，1-RTT 建连，恢复连接可以 0-RTT
- **连接迁移**：网络切换（Wi-Fi 切到 4G）时连接不中断（基于连接 ID 而不是 IP 端口）
- 弱网和移动网络下收益最明显

**实践**：服务端和 CDN 开启 HTTP/3，通过 `Alt-Svc` 响应头告知浏览器支持 HTTP/3，浏览器后续请求升级。

**口述模板（30–60 秒）**

1. 一句话结论：HTTP/2 实现多路复用，HTTP/3 进一步解决了 TCP 队头阻塞。
2. 关键机制：HTTP/3 基于 QUIC，流之间独立，连接建立更快，支持连接迁移。
3. 对比或注意点：弱网和移动网络下 HTTP/3 的收益最明显。
4. 可追问时补充：首次访问通过 Alt-Svc 发现 HTTP/3，后续请求才会使用。

**常见追问**

- 0-RTT 有什么风险？→ 0-RTT 数据可能被重放攻击，只能用于幂等请求。

---

### Q22. 怎么让下一个页面打开得更快？

- **难度**: 中级
- **标签**: 预加载

**答案**

| 方式 | 效果 | 成本 |
|------|------|------|
| `<link rel="prefetch">` | 预先下载下一页的资源 | 低 |
| 悬停 / 进入视口时预加载 | 用户意图更明确，命中率高 | 低 |
| **Speculation Rules：prefetch** | 预先获取下一页的 HTML | 中 |
| **Speculation Rules：prerender** | 在后台完整渲染下一页，导航时瞬间切换 | 高（CPU、内存、带宽） |
| SPA 路由预加载 | 预加载下一个路由的 chunk 和数据 | 低 |

```html
<script type="speculationrules">
{
  "prerender": [{
    "where": { "href_matches": "/product/*" },
    "eagerness": "moderate"
  }]
}
</script>
```

`eagerness`：`immediate`、`eager`、`moderate`（悬停约 200ms）、`conservative`（按下鼠标时）。

**注意**：Speculation Rules 目前只有 Chromium 支持；预渲染页面中的埋点要在页面真正激活后再上报（`document.prerendering`、`prerenderingchange` 事件）。

**口述模板（30–60 秒）**

1. 一句话结论：根据用户意图预加载或预渲染下一个页面。
2. 关键机制：prefetch 下载资源，Speculation Rules 可以预获取 HTML 甚至在后台完整渲染。
3. 对比或注意点：预渲染成本高，要控制触发时机；埋点要等页面激活后上报。
4. 可追问时补充：Speculation Rules 只有 Chromium 支持，其他浏览器自动忽略。

**常见追问**

- 预渲染会不会浪费流量？→ 会，所以用 moderate 或 conservative，并排除登出、加购物车等有副作用的链接。

---

## 六、资源与构建

### Q23. 怎么治理 JS 包体积？

- **难度**: 中级
- **标签**: 资源, 体积

**答案**

**1. 分析**：`webpack-bundle-analyzer`、`rollup-plugin-visualizer`、Chrome Coverage 面板（查看未使用的代码比例）

**2. 常见问题与处理**：

| 问题 | 处理 |
|------|------|
| 大依赖 | `moment` → `dayjs`；`lodash` → `lodash-es` 按需导入或原生实现 |
| 整包导入 | 按需导入组件库和图标 |
| 重复依赖（多个版本） | 包管理器去重、统一版本 |
| 多余的 polyfill | 按目标浏览器注入，或使用现代构建 |
| 首屏包含非首屏代码 | 路由级和组件级分割 |
| 业务代码中的大常量（地区数据、配置） | 改为接口获取或懒加载 |

**3. 防劣化**：CI 中设置体积预算（`size-limit`、`bundlesize`），超出时阻止合并。

**口述模板（30–60 秒）**

1. 一句话结论：先分析产物组成，再替换大依赖、按需导入、分割代码。
2. 关键机制：用分析工具和 Coverage 面板找到大块和未使用的代码。
3. 对比或注意点：体积治理的关键是防止劣化，CI 中要有预算门禁。
4. 可追问时补充：JS 的成本不仅是下载，还有解析和执行，对低端机影响更大。

**常见追问**

- 100KB 的 JS 和 100KB 的图片成本一样吗？→ 不一样，JS 还需要解析、编译和执行，占用主线程；图片解码通常不阻塞主线程。

---

### Q24. 怎么避免给现代浏览器加载多余的 polyfill？

- **难度**: 中级
- **标签**: 资源, 兼容性

**答案**

**方案一：按目标浏览器配置**

- 使用 `browserslist` 精确配置目标浏览器，根据真实用户的浏览器分布设置
- Babel `useBuiltIns: 'usage'` + core-js：只注入用到的、目标浏览器缺少的 polyfill

**方案二：现代 / 传统双构建（module / nomodule）**

```html
<script type="module" src="/modern.js"></script>
<script nomodule src="/legacy.js"></script>
```

现代浏览器只加载现代产物（ES2020+，几乎没有 polyfill），旧浏览器加载传统产物。Vite 的 `@vitejs/plugin-legacy` 就是这个原理。

**方案三：运行时按需加载**：检测特性后再动态加载 polyfill。

**口述模板（30–60 秒）**

1. 一句话结论：根据真实用户的浏览器分布配置目标，或者构建现代和传统两套产物。
2. 关键机制：module / nomodule 让现代浏览器只加载现代产物。
3. 对比或注意点：配置过于保守的 browserslist 会给所有用户加载大量 polyfill。
4. 可追问时补充：现代语法本身也更紧凑，执行效率更高。

**常见追问**

- 怎么知道目标浏览器该怎么配？→ 根据监控中的真实用户浏览器版本分布决定。

---

### Q25. 图片的响应式加载有哪些细节？

- **难度**: 中级
- **标签**: 图片

**答案**

**`srcset` + `sizes`**：浏览器根据显示宽度和设备像素比（DPR）选择合适的图片。

```html
<img
  srcset="p-400.avif 400w, p-800.avif 800w, p-1600.avif 1600w"
  sizes="(max-width: 768px) 100vw, 400px"
  src="p-800.avif" width="400" height="300" alt=""
/>
```

- `sizes` 必须准确，否则浏览器会选择错误的尺寸（默认 `100vw`，会下载过大的图片）
- 高 DPR 设备上，3x 图片的收益很小但体积很大，可以限制最大为 2x

**`<picture>` 用于艺术指导**：不同屏幕显示不同裁剪的图片。

**CDN 图片服务**：URL 参数指定宽度、格式、质量，根据 `Accept` 头自动返回 AVIF / WebP。

**口述模板（30–60 秒）**

1. 一句话结论：用 srcset 和 sizes 让浏览器选择合适尺寸的图片。
2. 关键机制：浏览器根据 sizes 计算显示宽度，再结合 DPR 选择图片。
3. 对比或注意点：sizes 不准确会下载过大的图片。
4. 可追问时补充：高 DPR 设备可以限制最大 2x，节省流量。

**常见追问**

- `sizes="auto"` 是什么？→ 用于懒加载图片，让浏览器根据布局后的实际宽度选择，需要配合 `loading="lazy"`（以浏览器支持为准）。

---

## 七、框架层优化

### Q26. React 应用怎么减少无效渲染？

- **难度**: 中级
- **标签**: React

**答案**

**原因**：父组件渲染会导致所有子组件重新渲染；Context 值变化导致所有消费者渲染。

**方法**：

1. **状态下沉**：状态放在真正使用它的组件中，减小渲染范围
2. **组件作为 children 传入**：children 由上层创建，状态变化时不会重新渲染
3. **`React.memo` + `useMemo` / `useCallback`**：保持 props 引用稳定
4. **拆分 Context**：频繁变化的值单独放一个 Context；或使用 selector（Zustand、Jotai）
5. **列表使用稳定的 key**
6. **React Compiler**（React 19 生态）：编译时自动做 memo，减少手动优化

**定位**：React DevTools Profiler 的「Highlight updates」和火焰图。

详见 [React 学习包](../react/00-roadmap.md)。

**口述模板（30–60 秒）**

1. 一句话结论：减少无效渲染的核心是缩小渲染范围和保持 props 引用稳定。
2. 关键机制：状态下沉、children 模式、memo、拆分 Context。
3. 对比或注意点：memo 本身有比较成本，不要无脑使用。
4. 可追问时补充：React Compiler 可以自动完成大部分 memo 工作。

**常见追问**

- 渲染次数多就一定慢吗？→ 不一定，关键是单次渲染的成本；先用 Profiler 确认问题再优化。

---

### Q27. React 的并发特性怎么改善 INP？

- **难度**: 中级
- **标签**: React, INP

**答案**

React 18+ 的并发渲染可以**中断**低优先级的渲染，让高优先级更新（用户输入）先执行。

**`useTransition`**：把非紧急更新标记为过渡

```tsx
const [isPending, startTransition] = useTransition()

function onTabClick(tab: string) {
  setActiveTab(tab)                // 紧急：立即更新标签高亮
  startTransition(() => {
    setTabContent(tab)             // 非紧急：重型内容可以被中断
  })
}
```

**`useDeferredValue`**：延迟某个值的更新

```tsx
const deferredQuery = useDeferredValue(query)
// 输入框使用 query（立即响应），列表使用 deferredQuery（可以延后）
const results = useMemo(() => filter(list, deferredQuery), [deferredQuery])
```

**原理**：React 以时间片（约 5ms）执行渲染工作，每个时间片后检查是否有更高优先级的任务，有则让出主线程。

**局限**：只对 React 的渲染阶段有效；事件处理函数中的同步计算、提交阶段的 DOM 操作不能被中断。

**口述模板（30–60 秒）**

1. 一句话结论：并发特性让低优先级渲染可以被中断，优先响应用户输入。
2. 关键机制：useTransition 标记非紧急更新，useDeferredValue 延迟值的更新，React 按时间片让出主线程。
3. 对比或注意点：只能中断渲染阶段，同步计算和提交阶段不能中断。
4. 可追问时补充：比防抖更好，因为不需要固定延迟，设备快时几乎没有延迟。

**常见追问**

- 和防抖有什么区别？→ 防抖固定延迟，在快设备上也会等待；useDeferredValue 根据设备能力自适应。

---

### Q28. Vue 应用有哪些性能优化手段？

- **难度**: 中级
- **标签**: Vue

**答案**

1. **响应式开销**：大型只读数据使用 `shallowRef` / `markRaw`，避免深层代理
2. **计算属性缓存**：使用 `computed` 而不是在模板中调用方法
3. **`v-memo`**：缓存列表项的渲染结果，依赖不变时跳过
4. **`v-once`**：只渲染一次的静态内容
5. **`v-show` vs `v-if`**：频繁切换用 `v-show`，很少显示用 `v-if`
6. **组件拆分**：Vue 的更新以组件为单位，拆分可以缩小更新范围
7. **`KeepAlive`**：缓存组件实例，避免重复创建
8. **异步组件**：`defineAsyncComponent` 按需加载
9. **编译时优化**：Vue 3 的静态提升、补丁标记、Block Tree 已自动完成；Vapor Mode（无虚拟 DOM 的编译模式，以官方发布状态为准）进一步降低运行时开销

**口述模板（30–60 秒）**

1. 一句话结论：Vue 优化的重点是减少响应式开销和缩小更新范围。
2. 关键机制：shallowRef 避免深层代理，组件拆分缩小更新范围，v-memo 跳过列表项更新。
3. 对比或注意点：Vue 3 编译器已经自动做了很多优化。
4. 可追问时补充：Vapor Mode 不使用虚拟 DOM，直接生成 DOM 操作。

**常见追问**

- 为什么大数据用 shallowRef？→ ref 会递归代理所有嵌套对象，数据量大时创建代理和依赖收集的成本很高。

---

## 八、内存

### Q29. 前端内存泄漏的常见原因有哪些？

- **难度**: 中级
- **标签**: 内存

**答案**

| 原因 | 示例 | 解决 |
|------|------|------|
| 未移除的事件监听 | 组件卸载后 window 上的监听仍在 | 卸载时移除，或使用 `AbortController` |
| 未清理的定时器 | `setInterval` 引用了组件数据 | 卸载时 `clearInterval` |
| 游离的 DOM（Detached DOM） | 节点已从文档移除，但 JS 变量仍引用 | 释放引用 |
| 闭包 | 长生命周期的闭包引用了大对象 | 缩小闭包作用域 |
| 全局变量和缓存 | 缓存无限增长 | 使用 LRU，或 WeakMap / WeakRef |
| 未关闭的订阅 | WebSocket、EventSource、观察者 | 卸载时关闭 |
| 第三方库实例 | 图表、地图实例未销毁 | 调用 `dispose` / `destroy` |

```js
const controller = new AbortController()
window.addEventListener('resize', onResize, { signal: controller.signal })
window.addEventListener('scroll', onScroll, { signal: controller.signal })
// 卸载时一次移除所有监听
controller.abort()
```

**口述模板（30–60 秒）**

1. 一句话结论：内存泄漏主要来自未清理的监听、定时器、游离 DOM 和无限增长的缓存。
2. 关键机制：长生命周期的对象引用了短生命周期的对象，导致无法回收。
3. 对比或注意点：SPA 中页面不刷新，泄漏会持续累积。
4. 可追问时补充：AbortController 可以一次移除多个监听。

**常见追问**

- WeakMap 为什么能避免泄漏？→ 它的键是弱引用，键对象没有其他引用时可以被回收，对应的值也随之释放。

---

### Q30. 怎么排查内存泄漏？

- **难度**: 中级
- **标签**: 内存, 排查

**答案**

**1. 确认泄漏**

- Performance 面板勾选 Memory，反复执行操作（打开关闭弹窗、切换路由），观察 JS Heap 是否持续增长（手动 GC 后仍然增长）
- Performance monitor 实时观察堆大小和 DOM 节点数

**2. 定位**：Memory 面板的 **Heap Snapshot**

- 三快照法：操作前拍快照 1 → 执行操作 → 快照 2 → 再执行 → 快照 3
- 对比快照：在快照 3 中筛选「Objects allocated between Snapshot 1 and Snapshot 2」
- 搜索 `Detached` 查找游离的 DOM 节点
- 查看对象的 **Retainers**（引用链），找到是谁持有了它

**3. Allocation instrumentation on timeline**：查看内存分配随时间的变化，蓝色柱表示未被回收的分配。

**口述模板（30–60 秒）**

1. 一句话结论：先观察堆内存趋势确认泄漏，再用堆快照对比定位。
2. 关键机制：多次快照对比找到持续增长的对象，通过 Retainers 引用链找到持有者。
3. 对比或注意点：要手动触发 GC 后再观察，排除尚未回收的对象。
4. 可追问时补充：搜索 Detached 可以快速找到游离 DOM。

**常见追问**

- 线上怎么监控内存？→ `performance.measureUserAgentSpecificMemory()`（需要跨源隔离）或在 Chromium 中读取 `performance.memory`，按会话时长观察趋势。

---

## 九、综合场景

### Q31. 移动端和弱网环境下有哪些特别的优化？

- **难度**: 中级
- **标签**: 移动端, 网络

**答案**

**设备层面**（低端安卓 CPU 比开发机慢 5–10 倍）：

- 减少 JS 体积和执行时间是首要任务
- 减少动画和复杂效果，可以根据 `navigator.hardwareConcurrency`、`navigator.deviceMemory` 降级

**网络层面**：

- 减少关键请求数和往返次数
- 请求超时和重试策略
- 根据网络状况自适应：`navigator.connection.effectiveType`、`saveData`（Chromium 支持）
- 离线缓存（Service Worker）
- 图片按网络质量降级
- HTTP/3 在弱网下收益明显

**WebView（App 内）**：

- 离线包：HTML、JS、CSS 预先下载到 App 中，加载时直接读取本地
- WebView 预创建和复用
- 数据预取：App 在打开 WebView 前并行请求页面数据

**口述模板（30–60 秒）**

1. 一句话结论：移动端优化的重点是减少 JS 执行和网络往返。
2. 关键机制：根据设备能力和网络状况自适应降级。
3. 对比或注意点：开发机测试会严重低估移动端的问题。
4. 可追问时补充：App 内 H5 可以用离线包和数据预取，接近原生体验。

**常见追问**

- 离线包怎么更新？→ App 启动时检查版本并后台下载新包，下次打开时生效；同时保留线上兜底。

---

### Q32. SPA 路由切换的性能怎么衡量和优化？

- **难度**: 中级
- **标签**: SPA, 指标

**答案**

**衡量的问题**：传统的 LCP、CLS 只统计第一次页面加载；SPA 中的路由切换（软导航）不会重新统计。

**新变化**：Chrome 151 正式支持 **Soft Navigations**：用户操作触发、URL 变化、有内容绘制时，浏览器会把它识别为一次软导航，并为它单独统计 LCP、INP、CLS（以官方文档为准）。web-vitals v6 支持软导航的上报。

**自定义衡量**：路由切换开始时 `performance.mark`，目标页面主体内容渲染后再 mark，计算差值。

**优化**：

- 路由 chunk 预加载（悬停时）
- 数据预取：路由切换和代码加载并行请求数据，而不是组件挂载后才请求
- 保持旧页面直到新页面准备好（React 的 `startTransition`），避免闪烁
- 缓存已访问页面的数据（SWR、React Query）
- View Transitions API 平滑过渡

**口述模板（30–60 秒）**

1. 一句话结论：SPA 的路由切换以前无法用 Web Vitals 衡量，需要自定义打点；Chrome 现在支持软导航。
2. 关键机制：软导航为每次路由切换单独统计指标。
3. 对比或注意点：数据请求不要等组件挂载后才发起，要和代码加载并行。
4. 可追问时补充：React Router 和 TanStack Router 的 loader 就是为了并行加载数据和代码。

**常见追问**

- 为什么数据请求要提前？→ 组件挂载后才请求会形成「代码加载 → 渲染 → 请求数据 → 再渲染」的瀑布。
