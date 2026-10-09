# 浏览器原理高级 Q&A

> 目标：能讲清 RenderingNG、合成线程、JIT 分层、Spectre 与隔离、存储隔离演进、导航新能力等底层机制，并能用浏览器原理解释性能优化和疑难问题。适合高级 / 资深岗位面试。

---

## 一、渲染架构

### Q1. 什么是 RenderingNG？它带来了什么？

- **难度**: 高级
- **标签**: 渲染, 架构

**答案**

RenderingNG 是 Chrome 在 2014–2021 年间重构的新一代渲染架构，核心目标是**可靠性、可扩展性和性能**。

**主要组成**：

| 项目 | 作用 |
|------|------|
| **LayoutNG** | 重写布局引擎，输出不可变的 Fragment 树，修复大量布局 bug，支持块分片（多列、打印） |
| **BlinkNG** | 把渲染流水线划分为清晰的阶段，每个阶段有明确的输入和输出 |
| **CompositeAfterPaint** | 先绘制再决定分层，而不是在绘制前分层，减少层的数量和错误 |
| **属性树** | 用 transform、clip、effect、scroll 树描述视觉属性，替代旧的层树 |
| **OOP-R** | 光栅化在 GPU 进程中进行 |
| **Viz / OOP-D** | 显示合成在 GPU 进程中进行，统一合成页面、iframe 和浏览器 UI |
| **VideoNG** | 统一视频渲染流程 |

**带来的能力**：容器查询、`content-visibility`、跨站 iframe 的流畅滚动、更可靠的合成。

**口述模板（30–60 秒）**

1. 一句话结论：RenderingNG 是 Chrome 新一代渲染架构，把流水线划分为清晰的阶段。
2. 关键机制：LayoutNG 输出不可变的 Fragment，CompositeAfterPaint 先绘制再分层，属性树描述视觉属性。
3. 对比或注意点：光栅化和显示合成都移到了 GPU 进程。
4. 可追问时补充：容器查询、content-visibility 等特性依赖这些架构改进。

**常见追问**

- 为什么不可变的 Fragment 树很重要？→ 布局结果不可变，可以安全地缓存和复用，增量布局更可靠，也便于实现容器查询这种依赖布局结果的特性。

---

### Q2. 合成线程为什么能在主线程卡住时继续工作？

- **难度**: 高级
- **标签**: 渲染, 合成

**答案**

**原因**：

1. 主线程每次更新后把**层和属性树**提交给合成线程，合成线程持有一份独立的副本
2. 滚动只修改 scroll 树中的偏移，transform / opacity 动画只修改 transform 树和 effect 树中的值
3. 这些修改不需要重新布局、重新绘制，只需要用已有的光栅化图块重新合成
4. 合成线程直接处理输入事件：没有非 passive 监听的区域，滚动不需要询问主线程

**合成动画**：CSS 动画和 Web Animations API 中只修改 `transform`、`opacity`（以及部分 `filter`）的动画，会交给合成线程运行（Chrome 也在支持更多可在合成线程运行的属性，以官方为准）。

**无法在合成线程运行的情况**：

- 修改 width、top 等布局属性的动画
- JS 每帧修改样式的动画（rAF 动画）
- 滚动驱动的 JS 效果（`scroll` 监听器）→ 改用 CSS 滚动驱动动画（`animation-timeline: scroll()`）

**口述模板（30–60 秒）**

1. 一句话结论：合成线程持有层和属性树的副本，滚动和 transform 动画只需要修改属性树并重新合成。
2. 关键机制：不需要布局和绘制，用已光栅化的图块直接合成。
3. 对比或注意点：JS 驱动的动画和布局属性的动画依赖主线程。
4. 可追问时补充：CSS 滚动驱动动画可以把滚动效果移到合成线程。

**常见追问**

- 主线程卡住时为什么点击按钮没有反应，但页面能滚动？→ 点击需要主线程执行事件监听；滚动由合成线程直接处理。

---

### Q3. 什么是层爆炸？怎么排查？

- **难度**: 高级
- **标签**: 渲染, 性能

**答案**

**原因**：合成层的绘制顺序必须与层叠顺序一致。如果一个合成层下方有元素，而**上方**还有与它重叠的普通元素，后者也必须被提升为合成层（**隐式合成**），否则顺序会出错。连锁反应导致层数量爆炸。

**典型场景**：

- 列表底部有一个 `z-index` 较低的动画元素，列表中每一项都与它重叠，全部被提升
- 滥用 `will-change: transform`，给大量元素都加上
- 大量 `position: fixed` 元素

**后果**：每个层都需要 GPU 内存（宽 × 高 × 4 字节）和合成开销；移动设备上会导致卡顿、发热甚至崩溃。

**排查**：

- DevTools → **Layers** 面板：查看层数量、内存占用、提升原因（Compositing Reasons）
- Rendering → Layer borders

**解决**：

- 给动画元素设置较高的 `z-index`，让它位于其他元素之上，避免其他元素被隐式提升
- 只在动画期间设置 `will-change`，结束后移除
- 减少不必要的合成触发属性

**口述模板（30–60 秒）**

1. 一句话结论：层爆炸是隐式合成导致的连锁反应，大量元素被提升为合成层。
2. 关键机制：为保证绘制顺序，与合成层重叠且位于其上的元素必须也被提升。
3. 对比或注意点：每个层都占用 GPU 内存，移动设备上影响很大。
4. 可追问时补充：用 Layers 面板查看提升原因，提高动画元素的 z-index 可以避免。

**常见追问**

- CompositeAfterPaint 解决了层爆炸吗？→ 它改进了分层算法（可以合并层），缓解了部分问题，但重叠导致的必要合成仍然存在。

---

### Q4. 页面掉帧怎么从原理上分析？

- **难度**: 高级
- **标签**: 渲染, 性能

**答案**

**按流水线定位**（Performance 面板 Frames 轨道中的红色或部分呈现的帧）：

| 瓶颈 | 表现 | 原因 |
|------|------|------|
| 主线程 JS | 长任务、事件处理时间长 | 计算密集、框架渲染 |
| 样式计算 | Recalculate Style 时间长 | 影响元素多、选择器复杂 |
| 布局 | Layout 时间长，或出现强制同步布局警告 | DOM 多、读写交替 |
| 绘制 | Paint 时间长 | 大面积重绘、复杂效果（阴影、滤镜） |
| 光栅化 | Raster 线程繁忙 | 大图、复杂绘制、高 DPR |
| 合成 / GPU | GPU 线程繁忙 | 层数量多、大面积透明叠加 |

**分析思路**：

1. 判断是主线程瓶颈还是合成 / GPU 瓶颈
2. 主线程：看具体耗时阶段和调用栈
3. 合成：看层数量、图块数量、是否有大面积 checkerboarding
4. 模拟低端设备（CPU 降速 4–6 倍）复现

**口述模板（30–60 秒）**

1. 一句话结论：按渲染流水线定位瓶颈在哪个阶段。
2. 关键机制：主线程看 JS、样式、布局、绘制；合成线程看光栅化和 GPU。
3. 对比或注意点：只看 JS 耗时可能漏掉绘制和光栅化的问题。
4. 可追问时补充：使用 CPU 降速模拟低端设备，才能复现真实用户的卡顿。

**常见追问**

- 部分呈现的帧（Partially presented frame）是什么？→ 合成线程按时输出了帧（例如滚动），但主线程的更新没赶上，只有部分内容更新。

---

### Q5. 事件循环中的「更新渲染」步骤有哪些细节？

- **难度**: 高级
- **标签**: 事件循环, 渲染

**答案**

**规范中的「更新渲染」步骤（简化）**：

1. 判断哪些文档有**渲染时机**（可见、需要更新、到达刷新时机）；可以跳过
2. 执行 resize 步骤
3. 执行 scroll 步骤（派发 scroll 事件）
4. 评估媒体查询，派发 change 事件
5. 更新动画，派发动画事件
6. 全屏相关步骤
7. **执行 rAF 回调**
8. 更新内容可见性（content-visibility）
9. **执行 ResizeObserver 回调**（可能循环多次，深度递增，防止无限循环）
10. 更新 IntersectionObserver
11. 更新渲染（样式、布局、绘制）

**微任务饥饿**：

```js
function loop() {
  Promise.resolve().then(loop)
}
loop()
```

微任务检查点要求清空微任务队列；如果微任务不断产生新的微任务，就永远无法到达渲染步骤，页面完全冻结（比 `while(true)` 更隐蔽）。

**ResizeObserver 循环错误**：回调中修改尺寸导致更深层的元素再次变化，超过限制时报 `ResizeObserver loop completed with undelivered notifications`。

**口述模板（30–60 秒）**

1. 一句话结论：更新渲染步骤依次处理 resize、scroll、媒体查询、动画、rAF、ResizeObserver、IntersectionObserver，最后渲染。
2. 关键机制：浏览器可以跳过没有渲染时机的帧；ResizeObserver 有深度限制防止无限循环。
3. 对比或注意点：无限产生的微任务会让页面永远无法渲染。
4. 可追问时补充：scroll 事件也是对齐到帧派发的。

**常见追问**

- ResizeObserver 的循环报错需要处理吗？→ 一般是良性的（延迟到下一帧处理），但如果频繁出现说明有尺寸反复变化的逻辑。

---

## 二、JS 引擎

### Q6. JIT 分层编译、反馈和 OSR 是怎么工作的？

- **难度**: 高级
- **标签**: V8, JIT

**答案**

**分层**：

| 层级 | 编译速度 | 代码质量 | 依据 |
|------|----------|----------|------|
| Ignition | 只生成字节码 | 解释执行 | 收集反馈 |
| Sparkplug | 很快 | 去除解释开销 | 字节码直接翻译 |
| Maglev | 较快 | 中等优化 | 基于反馈的推测 |
| TurboFan / Turboshaft | 慢 | 高度优化 | 基于反馈的激进优化（内联、逃逸分析等） |

**反馈向量**：每个函数有一个反馈向量，记录每个属性访问、调用、运算位置上看到的类型（隐藏类、Smi / Double 等）。

**触发升级**：每个函数有**中断预算**（interrupt budget），执行时消耗，预算耗尽时检查是否需要升级到下一层。

**OSR（栈上替换）**：一个长时间运行的循环中，函数本身只被调用一次，无法等待下次调用使用优化代码。OSR 在循环回边处把执行从解释器帧**切换到优化代码的帧**。

**去优化**：优化代码中插入检查（例如隐藏类检查），失败时根据保存的映射信息重建解释器帧，从字节码对应位置继续。

**Turboshaft**：新的基于控制流图（CFG）的编译器中间表示，替代 Sea of Nodes，编译速度更快；Turbolev 尝试从 Maglev 的图直接生成 Turboshaft 图（以 V8 官方进展为准）。

**口述模板（30–60 秒）**

1. 一句话结论：V8 用四层编译平衡启动和峰值性能，反馈向量提供推测依据，中断预算触发升级。
2. 关键机制：OSR 让长循环在执行中切换到优化代码；检查失败时去优化回到解释器。
3. 对比或注意点：优化编译的成本很高，所以只对热点代码进行。
4. 可追问时补充：顶层编译器正在迁移到 Turboshaft。

**常见追问**

- 为什么基准测试需要预热？→ 让代码经过分层编译达到稳定状态，否则测到的是解释执行或编译过程的耗时。

---

### Q7. 深入讲讲 V8 的堆结构和 GC 调优

- **难度**: 高级
- **标签**: V8, GC, 内存

**答案**

**堆空间**：

| 空间 | 内容 |
|------|------|
| 新生代（Young Generation） | 新创建的对象，Scavenge 回收 |
| 老生代（Old Space） | 存活较久的对象 |
| 代码空间（Code Space） | JIT 编译的机器码 |
| 大对象空间（Large Object Space） | 超过一定大小的对象，不移动 |
| 只读空间 | 不可变的内置对象 |

**Orinoco 的技术**：

- **并行**：多个线程同时执行 GC 工作（新生代 Scavenge），主线程也参与
- **增量**：主线程上的工作拆分为小块
- **并发**：后台线程标记，主线程继续执行 JS；写屏障记录引用修改
- **空闲时 GC**：利用帧之间的空闲时间进行 GC
- **指针压缩**：64 位系统中用 32 位偏移表示指针，减少约 40% 的堆内存（代价是堆上限约 4GB）

**浏览器端的内存问题**：

- GC 停顿表现为 Performance 面板中的 Major GC / Minor GC 长任务
- 频繁的 Minor GC：热路径中创建大量临时对象
- 内存不断增长：泄漏（见性能优化的内存章节）

**Node 中的调优**：`--max-old-space-size`、`--max-semi-space-size`（增大新生代可以减少 Scavenge 次数）。

**口述模板（30–60 秒）**

1. 一句话结论：V8 堆分为新生代、老生代、代码、大对象等空间，Orinoco 通过并行、增量、并发减少停顿。
2. 关键机制：写屏障保证并发标记的正确性，指针压缩减少内存占用。
3. 对比或注意点：热路径中大量临时对象会导致频繁的新生代 GC。
4. 可追问时补充：Node 服务可以调整新生代大小，减少 Scavenge 次数。

**常见追问**

- 怎么判断性能问题是 GC 导致的？→ Performance 面板中看 GC 事件的频率和耗时；Memory 面板的分配时间线看对象分配热点。

---

## 三、安全与隔离

### Q8. Spectre 对浏览器带来了什么影响？COOP 和 COEP 是什么？

- **难度**: 高级
- **标签**: 安全, 隔离

**答案**

**Spectre**：利用 CPU 推测执行和缓存时序，读取**同一进程地址空间内**的任意数据。JS 可以利用高精度计时器（`performance.now`、`SharedArrayBuffer`）实施攻击。

**浏览器的应对**：

1. **站点隔离**：不同站点在不同进程中，攻击者进程中没有其他站点的数据
2. **降低计时器精度**
3. **禁用 SharedArrayBuffer**，只在**跨源隔离**的页面中重新启用
4. **ORB / CORB**：阻止跨站的敏感数据（JSON、HTML）通过 `<img>`、`<script>` 进入攻击者进程

**跨源隔离（Cross-Origin Isolation）**：

```http
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp    （或 credentialless）
```

- **COOP**：与跨源的打开者 / 被打开窗口断开关联（`window.opener` 为 null），让页面可以运行在独立的进程（浏览上下文组）中
- **COEP**：页面加载的所有跨源资源都必须明确允许被嵌入（CORS 或 `Cross-Origin-Resource-Policy` 头），`credentialless` 模式下不带凭证加载
- 满足后 `self.crossOriginIsolated === true`，可以使用 `SharedArrayBuffer` 和高精度计时器

**代价**：第三方资源（广告、统计、地图）需要支持 CORP / CORS；OAuth 弹窗需要 `opener`，可以使用 `same-origin-allow-popups`。

**口述模板（30–60 秒）**

1. 一句话结论：Spectre 可以读取同进程的内存，浏览器通过站点隔离、降低计时器精度、限制 SharedArrayBuffer 来应对。
2. 关键机制：COOP 断开跨源窗口关联，COEP 要求跨源资源明确允许嵌入，二者满足后才能使用 SharedArrayBuffer。
3. 对比或注意点：第三方资源需要配合 CORP 或 CORS 头。
4. 可追问时补充：ORB 防止敏感数据通过 img、script 等标签进入攻击者进程。

**常见追问**

- 哪些场景需要跨源隔离？→ FFmpeg.wasm、多线程 Wasm 应用、高性能计算等需要 SharedArrayBuffer 的场景。

---

### Q9. 浏览器的整体安全模型是怎样的？

- **难度**: 高级
- **标签**: 安全

**答案**

**多层防御**：

| 层级 | 机制 | 防御目标 |
|------|------|----------|
| 操作系统 | 渲染进程沙箱 | 渲染进程被攻破后无法访问文件、网络、系统 |
| 进程 | 站点隔离、ORB | 被攻破的渲染进程无法读取其他站点的数据 |
| Web 平台 | 同源策略 | 不同源的脚本不能互相读取 DOM 和数据 |
| 受控放开 | CORS、postMessage | 显式授权跨源访问 |
| 内容 | CSP、Trusted Types、Sandbox iframe | 限制页面自身可执行的内容 |
| 传输 | HTTPS、HSTS、证书透明度 | 防止窃听和篡改 |
| 用户 | 权限提示、Safe Browsing | 敏感权限和恶意网站 |

**Fetch Metadata（`Sec-Fetch-*` 请求头）**：浏览器告诉服务器请求的来源（`Sec-Fetch-Site`）、模式（`Sec-Fetch-Mode`）和目的地（`Sec-Fetch-Dest`），服务器可以据此拒绝不合理的跨站请求（资源隔离策略），防御 CSRF、XSSI 和部分侧信道攻击。

**口述模板（30–60 秒）**

1. 一句话结论：浏览器安全是多层防御：系统沙箱、进程隔离、同源策略、内容策略、传输安全。
2. 关键机制：沙箱限制被攻破的渲染进程，站点隔离防止跨站读取数据。
3. 对比或注意点：同源策略是 Web 层的边界，CORS 是受控的放开。
4. 可追问时补充：服务端可以用 Fetch Metadata 实施资源隔离策略。

**常见追问**

- 渲染进程被攻破后能做什么？→ 只能在沙箱中操作本站点的数据，需要通过 IPC 请求浏览器进程，浏览器进程会验证请求的合法性。

---

### Q10. Cookie 和存储隔离是怎么演进的？

- **难度**: 高级
- **标签**: 存储, 隐私

**答案**

| 阶段 | 变化 |
|------|------|
| 早期 | 第三方 Cookie 和存储在所有站点间共享，可以跨站追踪 |
| 2017+ | Safari ITP 逐步限制第三方 Cookie 和存储 |
| 2020 | Chrome 80 默认 `SameSite=Lax`；跨站 Cookie 需要 `SameSite=None; Secure` |
| 2020–2021 | Firefox Total Cookie Protection：按顶级站点分区 |
| 2023–2024 | Chrome **存储分区**：iframe 中的 localStorage、IndexedDB 等按顶级站点分区；**HTTP 缓存分区**（按顶级站点 + 帧站点）；CHIPS |
| 2025 | Chrome 宣布保留第三方 Cookie，不推出选择弹窗；Privacy Sandbox 大部分 API 退役 |

**HTTP 缓存分区的影响**：同一个 CDN 上的公共库（例如 jQuery CDN），不同站点**无法共享缓存**，所以「使用公共 CDN 共享缓存」的优化已经失效。

**反弹追踪缓解**：浏览器检测只通过重定向设置 Cookie 的站点，定期清除它们的状态。

**口述模板（30–60 秒）**

1. 一句话结论：存储隔离从全局共享演进到按顶级站点分区。
2. 关键机制：SameSite 默认 Lax，存储和 HTTP 缓存分区，CHIPS 提供分区 Cookie。
3. 对比或注意点：Chrome 仍默认允许第三方 Cookie，但存储和缓存已经分区。
4. 可追问时补充：缓存分区让公共 CDN 共享缓存的优化失效。

**常见追问**

- 为什么要分区 HTTP 缓存？→ 防止通过缓存命中的时序判断用户是否访问过某个网站（缓存探测攻击）。

---

## 四、网络与加载

### Q11. 浏览器的资源优先级是怎么决定的？

- **难度**: 高级
- **标签**: 网络, 加载

**答案**

**Chrome 的默认优先级（简化）**：

| 资源 | 优先级 |
|------|--------|
| 主文档 | 最高 |
| head 中的 CSS | 最高 |
| 字体（preload） | 高 |
| head 中的同步脚本 | 高 |
| 视口内的图片 | 加载时先为低，布局后发现在视口内提升为高 |
| async / defer 脚本 | 低 |
| 视口外的图片 | 低 |
| prefetch | 最低 |

**调整方式**：

- `fetchpriority="high" / "low"`（img、link、script、fetch 的 `priority` 选项）
- `preload` 提前发现
- `loading="lazy"` 推迟加载

**HTTP 层**：

- HTTP/2 中浏览器为每个流设置优先级，服务器按优先级调度（依赖服务器实现）
- RFC 9218 的 `Priority` 头（`u=0..7` 表示紧急程度，`i` 表示增量）用于 HTTP/2 和 HTTP/3

**Chrome 的紧张模式（Tight Mode）**：页面加载初期，在 body 之前的资源（阻塞资源）加载完成前，限制低优先级资源的并发请求，避免与关键资源争抢带宽。

**口述模板（30–60 秒）**

1. 一句话结论：浏览器根据资源类型、位置和是否在视口内决定优先级。
2. 关键机制：CSS 和主文档最高，视口外图片低；fetchpriority 可以调整。
3. 对比或注意点：服务器必须正确实现优先级调度，HTTP/2 优先级才有效。
4. 可追问时补充：加载初期浏览器会限制低优先级资源的并发。

**常见追问**

- 为什么 LCP 图片要加 `fetchpriority="high"`？→ 图片初始优先级为低，要等布局后才能发现在视口内并提升，加上后可以在发现时立即以高优先级请求。

---

### Q12. 预加载扫描器有哪些局限？怎么弥补？

- **难度**: 高级
- **标签**: 加载

**答案**

| 局限 | 弥补 |
|------|------|
| JS 渲染的内容（CSR） | SSR / SSG 输出 HTML |
| CSS 背景图 | 改为 `<img>`，或 `<link rel="preload" as="image">` |
| CSS 中的字体 | `<link rel="preload" as="font" crossorigin>` |
| `@import` | 改为 `<link>` |
| `data-src` 懒加载 | 首屏图片不懒加载 |
| JS 动态导入的模块 | `modulepreload`；打包器生成预加载 |
| 第三方脚本加载的子资源 | 无法提前发现；关键的可以 preconnect |
| 响应式图片的选择 | 需要视口信息，扫描器支持 `srcset`、`sizes` |

**103 Early Hints**：服务器在生成 HTML 之前，先返回 103 响应，带上 `Link: <...>; rel=preload` 头，浏览器提前加载关键资源，利用了服务器的「思考时间」。

**口述模板（30–60 秒）**

1. 一句话结论：预加载扫描器只能发现 HTML 中直接声明的资源。
2. 关键机制：CSS 背景图、字体、JS 动态加载的资源都需要额外的提示。
3. 对比或注意点：首屏图片不应该使用 data-src 懒加载。
4. 可追问时补充：103 Early Hints 可以在 HTML 生成之前预加载资源。

**常见追问**

- preload 用多了会怎样？→ 与其他关键资源争抢带宽；未使用的 preload 浪费流量，Chrome 会警告。

---

### Q13. 字体加载和文本渲染有哪些原理和问题？

- **难度**: 高级
- **标签**: 字体, 渲染

**答案**

**字体加载时机**：浏览器只有在构建渲染树时，发现某个文本使用了该字体，才会下载字体（所以字体在关键路径的后面）。

**`font-display`**：

| 值 | 阻塞期 | 交换期 | 效果 |
|----|--------|--------|------|
| `auto` / `block` | 约 3s | 无限 | 文本不可见（FOIT） |
| `swap` | 极短 | 无限 | 先显示后备字体（FOUT），可能引起 CLS |
| `fallback` | 约 100ms | 约 3s | 折中 |
| `optional` | 约 100ms | 无 | 网络慢时本次不使用 |

**减少 CLS**：使用 `size-adjust`、`ascent-override`、`descent-override` 调整后备字体的度量，使其接近 Web 字体。

**中文字体**：体积大（数 MB），可以按使用的字符**子集化**，或使用 `unicode-range` 分片加载。

**文本渲染流程**：文本整形（HarfBuzz：字符 → 字形）→ 断行 → 光栅化字形（字形缓存）。

**口述模板（30–60 秒）**

1. 一句话结论：字体在渲染树阶段才被发现，需要 preload；font-display 控制加载期间的显示。
2. 关键机制：swap 先显示后备字体，optional 网络慢时放弃。
3. 对比或注意点：字体交换引起的布局偏移可以用 size-adjust 等描述符缓解。
4. 可追问时补充：中文字体需要子集化或 unicode-range 分片。

**常见追问**

- 预加载字体为什么需要 `crossorigin`？→ 字体总是以 CORS 模式请求，不加 crossorigin 的预加载使用不同的连接和缓存键，无法被复用。

---

### Q14. Navigation API 和 View Transitions 是什么？

- **难度**: 高级
- **标签**: 导航, 新特性

**答案**

**Navigation API**（Chromium 102+，Safari 26.2+，Firefox 支持，以官方为准；属于 Interop 2026 重点）：

```js
navigation.addEventListener('navigate', (event) => {
  if (!event.canIntercept || event.hashChange) return
  const url = new URL(event.destination.url)
  event.intercept({
    async handler() {
      const html = await fetchPage(url)
      render(html)
    },
  })
})
```

- 统一拦截所有导航（链接点击、表单提交、前进后退、`location` 赋值）
- 相比 History API：不需要拦截每个 `<a>` 的点击；提供 `navigation.entries()`、`navigation.currentEntry` 等完整的历史记录访问；导航有 Promise 和中止信号
- 是 SPA 路由库的更好的底层

**View Transitions**：

- **同文档**（`document.startViewTransition(callback)`）：截取旧状态 → 执行 DOM 更新 → 截取新状态 → 用伪元素做过渡动画
- **跨文档**（MPA）：两个页面都声明 `@view-transition { navigation: auto; }`，同源导航时自动过渡（Chrome 126+、Safari 18.2+）
- `view-transition-name` 标记需要共享过渡的元素

**口述模板（30–60 秒）**

1. 一句话结论：Navigation API 统一拦截所有导航，View Transitions 提供页面状态之间的过渡动画。
2. 关键机制：navigate 事件的 intercept 处理 SPA 路由；View Transition 截取新旧状态，用伪元素做动画。
3. 对比或注意点：跨文档 View Transitions 让 MPA 也能实现 SPA 般的过渡。
4. 可追问时补充：Safari 26.2 支持 Navigation API 后，各主流浏览器都已支持。

**常见追问**

- View Transitions 会影响性能吗？→ 截取快照和动画在合成层进行；但过渡期间页面不可交互，过渡时间不宜过长。

---

### Q15. Speculation Rules 预渲染的内部原理是什么？

- **难度**: 高级
- **标签**: 导航, 加载

**答案**

**预渲染**：浏览器在**隐藏的渲染器**中完整加载并渲染目标页面（执行 JS、加载子资源），用户导航时直接**激活**（切换显示），几乎瞬间完成。

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

**页面需要处理的问题**：

```js
if (document.prerendering) {
  document.addEventListener('prerenderingchange', () => {
    sendPageView()  // 激活时才上报 PV
  }, { once: true })
} else {
  sendPageView()
}
```

- 预渲染期间某些 API 被推迟（权限请求、媒体播放、存储访问等）
- 统计、广告曝光需要在激活时才上报
- `Sec-Purpose: prefetch;prerender` 请求头让服务器识别预渲染请求

**限制**：

- Chromium 专有；内存和数量有上限（按 eagerness 不同）
- 跨源的同站预渲染需要目标页面返回 `Supports-Loading-Mode: credentialed-prerender`
- 低内存、省流量模式下不执行

**口述模板（30–60 秒）**

1. 一句话结论：预渲染在隐藏的渲染器中完整加载目标页面，导航时直接激活。
2. 关键机制：页面通过 document.prerendering 和 prerenderingchange 处理预渲染期间的逻辑。
3. 对比或注意点：统计上报要推迟到激活；预渲染会消耗用户的流量和服务器资源。
4. 可追问时补充：Chromium 专有，其他浏览器会忽略，属于渐进增强。

**常见追问**

- 预渲染期间状态变化怎么办？→ 例如用户在当前页登录，预渲染页面可能是未登录状态；可以在 prerenderingchange 时刷新状态，或在状态变化时使预渲染失效。

---

### Q16. bfcache 的内部原理是什么？

- **难度**: 高级
- **标签**: 导航, 缓存

**答案**

**原理**：离开页面时，不销毁页面，而是**冻结**整个页面（JS 堆、DOM、渲染状态），暂停定时器、任务队列；后退或前进时直接**恢复**，瞬间显示。

**生命周期事件**：

```js
window.addEventListener('pagehide', (event) => {
  if (event.persisted) {
    // 页面进入 bfcache
  }
})
window.addEventListener('pageshow', (event) => {
  if (event.persisted) {
    // 从 bfcache 恢复，刷新可能过期的数据
  }
})
```

**阻止进入 bfcache 的常见原因**：

- 使用 `unload` 事件（Chrome 正在逐步弃用 unload）
- 打开的 WebSocket、WebRTC 连接、IndexedDB 事务
- `Cache-Control: no-store` 的页面（Chrome 在逐步放开，以官方为准）
- 存在 `window.opener` 引用

**检查**：DevTools → Application → Back/forward cache，可以测试并列出阻止原因；`PerformanceNavigationTiming.notRestoredReasons` API 提供线上数据。

**口述模板（30–60 秒）**

1. 一句话结论：bfcache 冻结整个页面，后退时直接恢复。
2. 关键机制：通过 pagehide 和 pageshow 的 persisted 判断进出 bfcache。
3. 对比或注意点：unload、未关闭的连接、no-store 会阻止页面进入 bfcache。
4. 可追问时补充：notRestoredReasons API 可以收集线上无法使用 bfcache 的原因。

**常见追问**

- 从 bfcache 恢复后，数据过期怎么办？→ 在 pageshow 中判断 persisted，重新获取关键数据（例如登录态、购物车）。

---

## 五、Worker 与新能力

### Q17. Worker、SharedArrayBuffer 和 Atomics 是怎么协作的？

- **难度**: 高级
- **标签**: Worker, 并发

**答案**

**postMessage 的代价**：

- 默认使用**结构化克隆**复制数据，大数据量时复制开销大
- **Transferable**（ArrayBuffer、MessagePort、OffscreenCanvas、ImageBitmap）：转移所有权，零拷贝，原线程中变为不可用

**SharedArrayBuffer**：多个线程共享同一块内存，需要跨源隔离（COOP + COEP）。

**Atomics**：保证共享内存操作的原子性和同步：

```js
// Worker 中等待
Atomics.wait(int32Array, index, expectedValue)
// 另一个线程中修改并唤醒
Atomics.store(int32Array, index, newValue)
Atomics.notify(int32Array, index, 1)
```

- `Atomics.add`、`compareExchange` 等原子操作，防止数据竞争
- `Atomics.wait` 会阻塞线程，**主线程不允许使用**（主线程使用 `Atomics.waitAsync`）

**应用场景**：Wasm 多线程（pthread）、音视频处理、大规模计算。

**口述模板（30–60 秒）**

1. 一句话结论：Worker 通过 postMessage 通信，SharedArrayBuffer 让线程共享内存，Atomics 保证同步。
2. 关键机制：结构化克隆会复制数据，Transferable 转移所有权，SharedArrayBuffer 真正共享。
3. 对比或注意点：SharedArrayBuffer 需要跨源隔离；主线程不能使用 Atomics.wait。
4. 可追问时补充：Wasm 多线程依赖 SharedArrayBuffer。

**常见追问**

- 什么时候值得使用 SharedArrayBuffer？→ 大量数据需要多线程频繁读写时；一般场景用 Transferable 就足够了。

---

### Q18. WebAssembly 在浏览器中是怎么执行的？

- **难度**: 高级
- **标签**: Wasm

**答案**

**执行流程**：

- **流式编译**：`WebAssembly.compileStreaming` / `instantiateStreaming`，边下载边编译
- **Liftoff**（基线编译器）：快速生成机器码，尽快开始执行
- **TurboFan**（优化编译器）：后台对热点函数重新编译为高度优化的代码，然后替换（动态分层）
- **代码缓存**：编译结果被缓存，二次加载时跳过编译

**为什么比 JS 快（在某些场景下）**：

- 静态类型，不需要类型反馈和去优化
- 二进制格式解码比 JS 解析快
- 线性内存，可预测的性能

**局限**：

- 操作 DOM 需要通过 JS 胶水代码，频繁跨边界调用有开销
- 包体积可能很大（引入了整个运行时）
- 并不总是比优化后的 JS 快

**新进展**：WasmGC（支持带 GC 的语言，如 Kotlin、Dart）、SIMD、多线程、异常处理、Memory64（以官方支持为准）。

**口述模板（30–60 秒）**

1. 一句话结论：Wasm 流式编译，先用 Liftoff 快速启动，热点函数再由 TurboFan 优化。
2. 关键机制：静态类型让性能更可预测，不需要去优化。
3. 对比或注意点：频繁调用 DOM 和跨边界传数据会抵消性能优势。
4. 可追问时补充：WasmGC 让 Kotlin、Dart 等带 GC 的语言也能高效编译到 Wasm。

**常见追问**

- 前端什么场景适合使用 Wasm？→ 图像和音视频处理、加密、物理引擎、复用已有的 C++ / Rust 库。

---

### Q19. GPU 进程和 WebGPU 是怎么工作的？

- **难度**: 高级
- **标签**: GPU, 渲染

**答案**

**GPU 进程**：

- 渲染进程在沙箱中不能直接访问 GPU，所有 GPU 命令都通过**命令缓冲区**发送到 GPU 进程执行
- 好处：隔离不稳定的 GPU 驱动（驱动崩溃只影响 GPU 进程，可以重启）；统一管理 GPU 资源
- Viz 合成、光栅化、WebGL、WebGPU 都在 GPU 进程中执行

**WebGL vs WebGPU**：

| 对比项 | WebGL | WebGPU |
|--------|-------|--------|
| 底层 | OpenGL ES 风格 | Vulkan / Metal / D3D12 风格 |
| 计算着色器 | 不支持 | **支持**（GPGPU、机器学习推理） |
| 状态管理 | 全局状态机 | 预先创建管线，开销更低 |
| 多线程 | 受限 | 更好的支持 |
| 支持情况 | 全面 | Chrome 113+，Safari 26，Firefox（部分平台）；以官方为准 |

**应用**：3D 渲染、浏览器端 AI 推理（例如 WebLLM、ONNX Runtime Web）、大规模数据可视化。

**口述模板（30–60 秒）**

1. 一句话结论：所有 GPU 操作都在 GPU 进程中执行，渲染进程通过命令缓冲区发送命令。
2. 关键机制：隔离不稳定的驱动，统一管理 GPU 资源。
3. 对比或注意点：WebGPU 支持计算着色器，状态管理开销更低。
4. 可追问时补充：WebGPU 让浏览器端 AI 推理成为可能。

**常见追问**

- GPU 进程崩溃会怎样？→ 浏览器会重启 GPU 进程，页面短暂黑屏或闪烁；多次崩溃后可能回退到软件渲染。

---

### Q20. 浏览器扩展的架构和影响是什么？

- **难度**: 高级
- **标签**: 扩展

**答案**

**Manifest V3 的组成**：

| 部分 | 运行环境 | 作用 |
|------|----------|------|
| Service Worker（后台） | 扩展进程 | 事件驱动的后台逻辑，空闲时被终止 |
| Content Script | 页面渲染进程，**隔离的世界**（Isolated World） | 与页面共享 DOM，但 JS 环境独立 |
| Popup / Options | 扩展页面 | UI |

**隔离的世界**：内容脚本和页面脚本的 JS 全局对象不同（不能直接访问页面的变量），但共享同一个 DOM。

**MV3 的变化**：后台页改为 Service Worker；`webRequest` 拦截改为声明式的 `declarativeNetRequest`；禁止远程代码。

**对前端开发的影响**：

- 扩展注入的脚本和 DOM 修改可能导致 **hydration 不匹配**、白屏、性能问题
- 广告拦截扩展可能拦截包含 `ad`、`track` 等关键词的请求
- 监控中会出现大量来自扩展的错误（`chrome-extension://`），需要过滤
- 性能测试应该在无痕模式或干净的配置中进行

**口述模板（30–60 秒）**

1. 一句话结论：MV3 扩展由 Service Worker、内容脚本和扩展页面组成。
2. 关键机制：内容脚本运行在隔离的世界中，与页面共享 DOM 但不共享 JS 环境。
3. 对比或注意点：扩展会修改 DOM，可能导致 hydration 不匹配和监控噪声。
4. 可追问时补充：性能测试应在无痕模式中进行，排除扩展的影响。

**常见追问**

- 监控中的扩展错误怎么过滤？→ 根据错误堆栈中的 `chrome-extension://`、`moz-extension://` 过滤，只保留自身代码的错误。

---

## 六、疑难问题与工程

### Q21. 页面崩溃和内存溢出怎么诊断？

- **难度**: 高级
- **标签**: 稳定性, 内存

**答案**

**现象**：页面显示「喔唷，崩溃啦」（Aw, Snap!），错误码如 `Out of Memory`、`SIGILL`、`STATUS_ACCESS_VIOLATION`。

**常见原因**：

- 内存泄漏导致内存不断增长，超出渲染进程的上限
- 一次性创建大量对象（例如渲染几十万行数据、加载超大图片）
- 无限递归、死循环
- GPU 内存耗尽（层爆炸、大量大尺寸 Canvas）
- 浏览器 bug、驱动问题

**线上监测**：

- **Reporting API**：`crash` 报告类型（包括 OOM 原因），通过 `Reporting-Endpoints` 头配置上报地址
- **心跳检测**：页面定期向 Service Worker 发送心跳；SW 发现某个页面心跳中断，且没有正常关闭，就判断为崩溃并上报
- `performance.memory`（非标准）/ `measureUserAgentSpecificMemory()`（需要跨源隔离）监测内存

**本地排查**：Memory 面板对比堆快照；`chrome://crashes`；任务管理器查看进程内存。

**口述模板（30–60 秒）**

1. 一句话结论：页面崩溃常见原因是内存溢出、GPU 内存耗尽和死循环。
2. 关键机制：线上通过 Reporting API 的 crash 报告或 Service Worker 心跳检测崩溃。
3. 对比或注意点：崩溃时页面 JS 无法执行，必须依赖外部机制上报。
4. 可追问时补充：本地用堆快照对比排查泄漏。

**常见追问**

- 心跳检测怎么区分崩溃和正常关闭？→ 页面在 pagehide 时通知 SW 正常关闭；心跳中断且没有收到关闭通知，才判断为崩溃。

---

### Q22. 怎么处理浏览器兼容性问题？Baseline 和 Interop 是什么？

- **难度**: 高级
- **标签**: 兼容性, 工程

**答案**

**Baseline**：

- **Newly available**：所有核心浏览器（Chrome、Edge、Firefox、Safari）最新版本都支持
- **Widely available**：新可用之后 30 个月，可以放心使用
- MDN、caniuse 显示 Baseline 标记；构建工具和 Browserslist 可以基于 Baseline 年份配置目标

**Interop**：各浏览器厂商每年共同选择重点领域，对齐实现、修复差异（以 WPT 测试通过率衡量）。Interop 2026 的重点包括事件循环、Navigation API、同文档 View Transitions、锚点定位等。

**工程实践**：

1. **明确支持范围**：根据用户数据配置 Browserslist
2. **构建时**：Babel / SWC 转译语法，按需引入 polyfill，PostCSS 处理 CSS
3. **运行时**：特性检测（`'IntersectionObserver' in window`、`CSS.supports()`、`@supports`），渐进增强
4. **测试**：真机 / 云测试平台覆盖主要浏览器；WebView 单独测试
5. **监控**：按浏览器版本统计错误率

**口述模板（30–60 秒）**

1. 一句话结论：用 Baseline 判断特性是否可以使用，用 Interop 了解各浏览器对齐的方向。
2. 关键机制：构建时转译和 polyfill，运行时特性检测和渐进增强。
3. 对比或注意点：支持范围要基于用户数据，而不是凭感觉。
4. 可追问时补充：监控中按浏览器版本统计错误，发现兼容性问题。

**常见追问**

- 为什么推荐特性检测而不是 UA 检测？→ UA 可能被伪造或冻结（UA Reduction），特性检测直接判断能力是否存在。

---

### Q23. 移动端 WebView 与浏览器有什么不同？

- **难度**: 高级
- **标签**: 移动端, WebView

**答案**

| 对比项 | Android WebView | iOS WKWebView |
|--------|-----------------|---------------|
| 内核 | Chromium，独立更新（Android 7+ 通过应用商店更新） | WebKit，随 iOS 系统更新 |
| 多进程 | 部分版本单进程或渲染进程独立 | WebContent 进程独立 |
| 第三方 Cookie | 默认限制，App 可以配置 | 受 ITP 影响 |
| 存储 | 可能被 App 清除 | 系统可能在存储紧张时清除 |

**与浏览器的差异**：

- **版本碎片化**：Android 旧设备的 WebView 版本可能很老；iOS 跟随系统版本
- **JSBridge**：与原生通信（URL Scheme 拦截、注入对象、`postMessage`）
- **性能**：WebView 初始化有开销（几百毫秒），App 通常会预热 WebView
- **缓存**：App 可以配置离线包，拦截请求返回本地资源
- **调试**：Android 用 `chrome://inspect`，iOS 用 Safari Web Inspector
- 键盘弹出、安全区域（`env(safe-area-inset-*)`）、100vh 问题（使用 `dvh`）

**口述模板（30–60 秒）**

1. 一句话结论：Android WebView 基于 Chromium 独立更新，iOS WKWebView 随系统更新。
2. 关键机制：通过 JSBridge 与原生通信，App 可以预热 WebView 和配置离线包。
3. 对比或注意点：版本碎片化、存储被清除、键盘和安全区域是常见的问题。
4. 可追问时补充：iOS 上所有浏览器都使用 WebKit 内核（欧盟等地区有例外）。

**常见追问**

- H5 在 App 中打开慢怎么优化？→ WebView 预热、离线包、数据预取、骨架屏，详见性能优化 [S6](../performance/50-system-design.md)。

---

### Q24. 跨进程 iframe（OOPIF）带来了哪些复杂性？

- **难度**: 高级
- **标签**: 架构, iframe

**答案**

**OOPIF**：站点隔离后，跨站 iframe 运行在独立的渲染进程中。

**复杂性**：

- **帧树分布在多个进程中**：每个进程中有完整的帧树结构，本进程的帧是 LocalFrame，其他进程的帧是 RemoteFrame（代理）
- **跨进程 postMessage**：通过浏览器进程中转
- **渲染**：每个进程分别生成合成帧，由 Viz 合成到一起（SurfaceId 引用子帧的输出）
- **输入事件**：浏览器进程需要做命中测试，判断点击落在哪个进程的帧上
- **页面内查找、打印、无障碍树、DevTools**：都需要跨进程协调

**对开发者的影响**：

- 跨站 iframe 的 JS 执行与父页面**并行**，不再互相阻塞
- 父页面主线程卡住不影响跨站 iframe 的交互（反之亦然）
- 每个跨站 iframe 都有进程开销，大量嵌入第三方 iframe 会增加内存

**口述模板（30–60 秒）**

1. 一句话结论：OOPIF 让跨站 iframe 运行在独立进程中，帧树分布在多个进程中。
2. 关键机制：RemoteFrame 作为代理，postMessage 和输入事件由浏览器进程中转，Viz 合成各进程的输出。
3. 对比或注意点：跨站 iframe 和父页面并行执行，但每个都有进程开销。
4. 可追问时补充：同站 iframe 仍然与父页面在同一进程，互相阻塞。

**常见追问**

- 同站 iframe 会阻塞父页面吗？→ 会，同站 iframe 与父页面共享同一个渲染进程和主线程。

---

### Q25. 浏览器的发版周期和 Web 标准流程是怎样的？

- **难度**: 高级
- **标签**: 工程, 标准

**答案**

**Chrome 发布**：

- 每 4 周发布一个稳定版
- 渠道：Canary（每日）→ Dev → Beta → Stable；另有扩展稳定版（Extended Stable，8 周）
- 新特性的流程：Intent to Prototype → 实验（flag、Origin Trial）→ Intent to Ship → 发布
- **Origin Trial**：网站注册后，可以在线上真实用户中试用实验特性

**其他浏览器**：

- Firefox：每 4 周发布
- Safari：随 iOS / macOS 大版本发布（每年秋季），中间有小版本更新

**标准流程**：

- **WHATWG**：HTML、DOM、Fetch、URL 等「活标准」，持续更新
- **W3C**：CSS 工作组、WebApps 等，经历工作草案 → 候选推荐 → 推荐标准
- **TC39**：ECMAScript，提案分为 Stage 0–4，Stage 4 进入下一年的标准
- **IETF**：HTTP、TLS、QUIC 等网络协议，发布为 RFC

**跟进渠道**：Chrome Platform Status、MDN、WebKit 博客、Firefox 发布说明、Baseline。

**口述模板（30–60 秒）**

1. 一句话结论：Chrome 和 Firefox 每 4 周发布，Safari 跟随系统发布。
2. 关键机制：Chrome 新特性经过原型、Origin Trial、Intent to Ship 流程。
3. 对比或注意点：HTML 是 WHATWG 活标准，ECMAScript 由 TC39 按阶段推进。
4. 可追问时补充：Origin Trial 可以在真实用户中试用实验特性。

**常见追问**

- Safari 的发布节奏对前端有什么影响？→ 新特性在 Safari 中可用通常较晚，且 iOS 用户升级系统的速度决定了实际可用时间。

---

### Q26. 怎么用浏览器原理解释常见的性能优化？

- **难度**: 高级
- **标签**: 综合, 性能

**答案**

| 优化手段 | 对应的浏览器原理 |
|----------|------------------|
| 关键 CSS 内联 | CSS 阻塞渲染，内联省去请求 |
| 脚本使用 defer | 同步脚本阻塞解析 |
| LCP 图片写在 HTML 中 + fetchpriority | 预加载扫描器发现资源；图片默认低优先级 |
| preconnect | 提前完成 DNS + TCP + TLS |
| 带 hash 的资源 + 长缓存 | HTTP 强缓存跳过请求 |
| transform / opacity 动画 | 只需合成，可以在合成线程运行 |
| 读写分离 | 避免强制同步布局 |
| 拆分长任务 | 事件循环中长任务阻塞输入和渲染 |
| passive 事件监听 | 合成线程可以直接滚动 |
| content-visibility | 跳过屏幕外内容的布局和绘制 |
| Web Worker | 把计算移出主线程 |
| 避免 unload | 让页面可以进入 bfcache |
| Speculation Rules | 在隐藏渲染器中提前渲染 |
| 不使用公共 CDN 共享缓存 | HTTP 缓存已经按站点分区 |

**面试回答方式**：说出一个优化，就说明它对应的浏览器原理和要解决的瓶颈，以及怎么验证效果。

详见 [性能优化](../performance/00-roadmap.md)。

**口述模板（30–60 秒）**

1. 一句话结论：性能优化本质是减少关键路径上的工作，并利用浏览器的机制并行处理。
2. 关键机制：加载优化对应网络和解析，交互优化对应事件循环，动画优化对应合成。
3. 对比或注意点：每个优化都要有对应的指标来验证效果。
4. 可追问时补充：浏览器原理的变化会让某些旧的优化失效，例如缓存分区让公共 CDN 共享缓存失效。

**常见追问**

- 有哪些曾经有效、现在失效的优化？→ 公共 CDN 共享缓存（缓存分区）、域名分片（HTTP/2）、HTTP/2 服务器推送（已移除）、雪碧图的必要性降低（HTTP/2 多路复用）。
