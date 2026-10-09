# HTML 中级 Q&A（能讲清）

> 目标：讲清解析、加载、无障碍、SEO、安全和现代原生组件。面试高频区。

---

## 一、解析与资源加载

### Q1. 浏览器拿到 HTML 后大致如何处理？

- **难度**: mid
- **标签**: 解析, 渲染

**答案**

简化流程：

1. 接收字节，根据编码解码成字符
2. 词法分析（Tokenization）生成开始标签、结束标签、文本等 token
3. 树构建（Tree Construction）生成 DOM
4. CSS 解析成 CSSOM
5. DOM + CSSOM 生成渲染树（Render Tree）
6. 布局（Layout）、绘制（Paint）、合成（Composite）

HTML 解析是**增量**进行的，浏览器不必等完整 HTML 下载完成才开始解析。

**口述模板（30–60 秒）**

1. 一句话结论：HTML 被增量解析为 DOM，再和 CSSOM 一起参与渲染。
2. 关键机制：解码、分词、建树、样式计算、布局、绘制、合成。
3. 注意点：脚本和 CSS 会影响解析或渲染时机。
4. 可追问时补充：预加载扫描器会提前发现资源。

**常见追问**

- 渲染树包含 `display: none` 元素吗？→ 不包含，因为它不生成盒子。

---

### Q2. 为什么说 CSS 阻塞渲染，JS 阻塞解析？

- **难度**: mid
- **标签**: 关键渲染路径

**答案**

- **CSS 阻塞渲染**：没有 CSSOM，浏览器无法确定最终样式，通常不会首次绘制相关内容，避免无样式闪烁（FOUC）
- **普通同步 JS 阻塞解析**：脚本可能读取或修改 DOM / CSSOM，甚至 `document.write`，所以 HTML 解析器遇到它时要暂停
- **JS 还可能等待 CSS**：如果脚本前面有样式表，脚本执行前通常要等待 CSSOM 就绪，因为 JS 可能读取计算样式

**口述模板（30–60 秒）**

1. 一句话结论：CSS 卡首绘，同步 JS 卡 DOM 解析。
2. 关键机制：JS 可能修改 DOM，也可能读取样式。
3. 注意点：前置 CSS 也会间接延迟同步 JS。
4. 可追问时补充：用 defer、关键 CSS、拆分样式优化。

**常见追问**

- CSS 会阻塞 DOM 解析吗？→ 通常不会直接阻塞解析，但会阻塞后续同步脚本执行。

---

### Q3. `script` 普通、`async`、`defer` 的区别？

- **难度**: mid
- **标签**: script, 性能

**答案**

| 类型 | 下载 | 执行时机 | 顺序 |
|------|------|----------|------|
| 普通外链 | 暂停解析后下载 | 下载完立即执行 | 按文档顺序 |
| `async` | 并行下载 | 下载完尽快执行，可能打断解析 | 不保证 |
| `defer` | 并行下载 | HTML 解析完成后、`DOMContentLoaded` 前 | 按文档顺序 |

`async` 适合独立脚本，如统计；业务入口、依赖顺序的脚本优先 `defer` 或 `type="module"`。

**口述模板（30–60 秒）**

1. 一句话结论：普通脚本阻塞，async 下载完就跑，defer 解析完按顺序跑。
2. 关键机制：async/defer 都并行下载。
3. 注意点：async 不保证顺序，也不保证 DOM 完整。
4. 可追问时补充：内联经典脚本上的 async/defer 无效。

**常见追问**

- 同时写 async 和 defer？→ 支持 async 的浏览器优先 async；defer 主要是历史兼容回退。

---

### Q4. `type="module"` 的脚本加载有什么特点？

- **难度**: mid
- **标签**: ES Module, script

**答案**

模块脚本：

- 默认类似 `defer`，不阻塞 HTML 解析，按顺序在解析后执行
- 支持 `import` / `export`
- 有独立模块作用域，不会把顶层变量挂到全局
- 默认严格模式
- 同一 URL 模块只执行一次
- 跨域加载需要 CORS

可以写 `async`，此时模块及依赖准备好后尽快执行。

**口述模板（30–60 秒）**

1. 一句话结论：module 脚本默认延迟、模块作用域、严格模式。
2. 关键机制：浏览器先构建模块依赖图再执行。
3. 注意点：跨域需要 CORS，依赖链过深会产生瀑布请求。
4. 可追问时补充：`modulepreload` 可提前加载依赖。

**常见追问**

- `nomodule` 有什么用？→ 早期给不支持 ES Module 的旧浏览器提供回退脚本。

---

### Q5. 什么是预加载扫描器（Preload Scanner）？

- **难度**: mid
- **标签**: 解析, 性能

**答案**

当主解析器被同步脚本阻塞时，浏览器可以用预加载扫描器继续扫描后面的 HTML，提前发现 `img`、`link`、`script` 等资源并发起请求。

影响：

- 写在 HTML 中的资源更容易被早发现
- JS 动态插入的资源、CSS 背景图通常更晚被发现
- 首屏关键图片若只由客户端 JS 渲染，LCP 可能变慢

**口述模板（30–60 秒）**

1. 一句话结论：预加载扫描器在主解析阻塞时提前发现资源。
2. 关键机制：它扫描 HTML 标记，不执行 JS。
3. 注意点：动态资源无法被它提前发现。
4. 可追问时补充：SSR 和 preload 对 LCP 的帮助。

**常见追问**

- `document.write` 为什么不好？→ 可能让扫描器的预测失效并阻塞解析，现代页面应避免。

---

### Q6. `preload`、`prefetch`、`preconnect`、`dns-prefetch` 区别？

- **难度**: mid
- **标签**: Resource Hints

**答案**

| 写法 | 用途 |
|------|------|
| `rel="preload"` | 当前页面很快需要的关键资源 |
| `rel="prefetch"` | 未来导航可能需要的资源，低优先级 |
| `rel="preconnect"` | 提前建立 DNS、TCP、TLS 连接 |
| `rel="dns-prefetch"` | 只提前做 DNS 解析 |

```html
<link rel="preload" href="/fonts/a.woff2" as="font" type="font/woff2" crossorigin />
<link rel="preconnect" href="https://cdn.example.com" />
```

它们是提示，不保证浏览器一定执行；滥用会抢占带宽。

**口述模板（30–60 秒）**

1. 一句话结论：preload 管当前页，prefetch 管未来，preconnect 管连接。
2. 关键机制：提前发现或建立网络准备。
3. 注意点：preload 要写正确 `as`，字体常需 crossorigin。
4. 可追问时补充：用 Performance 面板验证收益。

**常见追问**

- preload 了但没使用会怎样？→ 浪费带宽，Chrome 控制台会警告。

---

### Q7. `fetchpriority` 有什么用？

- **难度**: mid
- **标签**: 性能, LCP

**答案**

`fetchpriority` 向浏览器提示资源相对优先级，可用于 `img`、`link`、`script`、`iframe` 等：

```html
<img src="hero.webp" fetchpriority="high" width="1200" height="600" alt="..." />
```

常用于首屏 LCP 图片提权，或降低轮播里非当前图片优先级。它是提示，不是强制命令；不要给很多资源都设 high。

**口述模板（30–60 秒）**

1. 一句话结论：fetchpriority 调整资源相对优先级。
2. 关键机制：影响浏览器网络调度。
3. 注意点：高优先级资源太多等于没有优先级。
4. 可追问时补充：和 preload 可以配合，但解决的问题不同。

**常见追问**

- LCP 图片能 lazy 吗？→ 不应 lazy，否则会推迟加载。

---

### Q8. `srcset`、`sizes`、`picture` 怎么选？

- **难度**: mid
- **标签**: 响应式图片

**答案**

- **同一张图不同分辨率**：`img srcset sizes`
- **不同格式回退**：`picture` + 多个 `source type`
- **艺术设计（Art Direction）**：不同屏幕显示不同裁剪，用 `picture` + `media`

```html
<img
  src="card-800.jpg"
  srcset="card-400.jpg 400w, card-800.jpg 800w, card-1200.jpg 1200w"
  sizes="(max-width: 600px) 100vw, 33vw"
  width="800" height="600" alt="商品图" />
```

`sizes` 写错会导致浏览器选错图片尺寸。

**口述模板（30–60 秒）**

1. 一句话结论：srcset 让浏览器选合适尺寸，picture 控制格式或裁剪。
2. 关键机制：浏览器结合 sizes、DPR 和网络选择资源。
3. 注意点：sizes 要描述真实布局宽度。
4. 可追问时补充：AVIF / WebP 回退。

**常见追问**

- `x` 和 `w` 描述符区别？→ `x` 面向固定显示尺寸的不同像素密度，`w` 面向响应式宽度。

---

### Q9. `DOMContentLoaded` 和 `load` 有什么区别？

- **难度**: mid
- **标签**: 页面生命周期

**答案**

- `DOMContentLoaded`：初始 HTML 解析完成，且 `defer` 脚本执行完成；不等待图片、iframe 等资源
- `load`：页面及依赖资源（图片、样式、iframe 等）加载完成

`async` 脚本不一定在 `DOMContentLoaded` 前执行。业务初始化通常不需要等待 `load`。

**口述模板（30–60 秒）**

1. 一句话结论：DCL 关注 DOM，load 关注全部资源。
2. 关键机制：defer 在 DCL 前执行。
3. 注意点：async 时序不确定。
4. 可追问时补充：现代性能指标更关注 LCP、INP、CLS。

**常见追问**

- 样式表会延迟 DCL 吗？→ 不直接等待，但如果样式阻塞了同步脚本，可能间接延迟。

---

### Q10. 为什么不推荐使用 `unload` 事件？

- **难度**: mid
- **标签**: 生命周期, bfcache

**答案**

`unload` 在移动端不可靠，并且会影响页面进入往返缓存（Back/Forward Cache, bfcache）。

推荐：

- 保存状态 / 上报：`visibilitychange` 到 `hidden` 时处理
- 页面离开：`pagehide`
- 从 bfcache 恢复：`pageshow` 且 `event.persisted === true`
- 上报数据用 `navigator.sendBeacon` 或 `fetch(..., { keepalive: true })`

**口述模板（30–60 秒）**

1. 一句话结论：unload 不可靠且影响 bfcache。
2. 关键机制：现代浏览器会冻结或缓存页面。
3. 注意点：用 visibilitychange/pagehide 代替。
4. 可追问时补充：bfcache 恢复后需要刷新过期数据。

**常见追问**

- `beforeunload` 能用吗？→ 仅在确实有未保存变更时临时注册，用完移除。

---

## 二、可访问性

### Q11. 可访问性（a11y）在 HTML 层面最重要的是什么？

- **难度**: mid
- **标签**: a11y

**答案**

优先级：

1. 语义正确：标题、landmark、列表、表格、按钮、链接
2. 每个控件有可访问名称
3. 键盘可用，焦点可见、顺序合理
4. 状态可感知：展开、选中、错误、加载
5. 图片、视频有替代内容
6. 颜色不是唯一信息来源

原生 HTML 是 a11y 的地基，ARIA 只是补充。

**口述模板（30–60 秒）**

1. 一句话结论：先用对 HTML，再补 ARIA。
2. 关键机制：浏览器把 DOM 映射为无障碍树。
3. 注意点：键盘、焦点、名称、状态是核心。
4. 可追问时补充：自动化检测只能发现部分问题。

**常见追问**

- Lighthouse 满分就无障碍了吗？→ 不是，还需要键盘和读屏实测。

---

### Q12. 什么是 ARIA？ARIA 第一规则是什么？

- **难度**: mid
- **标签**: ARIA

**答案**

ARIA（Accessible Rich Internet Applications）通过角色、状态和属性向辅助技术描述复杂组件。

第一规则：**如果能使用原生 HTML 元素或属性实现所需语义和行为，就不要用 ARIA。**

例如用 `<button>`，不要用 `<div role="button">`。ARIA 不会自动增加键盘行为，你写了 role 就要负责实现对应交互。

**口述模板（30–60 秒）**

1. 一句话结论：ARIA 补语义，不补行为。
2. 关键机制：它影响无障碍树中的角色和状态。
3. 注意点：No ARIA is better than bad ARIA。
4. 可追问时补充：复杂组件参考 ARIA Authoring Practices。

**常见追问**

- `role="button"` 还要做什么？→ `tabindex="0"`、Enter/Space 触发、禁用状态、焦点样式。

---

### Q13. 可访问名称（Accessible Name）从哪里来？

- **难度**: mid
- **标签**: a11y

**答案**

常见来源（简化优先级）：

1. `aria-labelledby`
2. `aria-label`
3. 原生关联，如 `label`、`alt`、`caption`、`legend`
4. 元素文本内容，如 button 文本
5. `title` 等兜底

例子：

```html
<button aria-label="关闭弹窗">×</button>
```

可见文本存在时，尽量让可访问名称包含可见文本，方便语音控制用户。

**口述模板（30–60 秒）**

1. 一句话结论：可访问名称是辅助技术读出的控件名。
2. 关键机制：浏览器按规则从 aria、label、文本等计算。
3. 注意点：图标按钮必须补名称。
4. 可追问时补充：aria-labelledby 可引用多个元素。

**常见追问**

- placeholder 能当名称吗？→ 某些浏览器会兜底，但不可依赖。

---

### Q14. 什么是 landmark？为什么要写 skip link？

- **难度**: mid
- **标签**: a11y, 导航

**答案**

landmark 是页面中的大区域，如 `header`、`nav`、`main`、`aside`、`footer`、`search`。读屏用户可以按区域跳转。

Skip link 让键盘用户跳过重复导航：

```html
<a class="skip-link" href="#main">跳到主要内容</a>
<main id="main" tabindex="-1">...</main>
```

**口述模板（30–60 秒）**

1. 一句话结论：landmark 提供页面导航结构。
2. 关键机制：辅助技术可列出和跳转区域。
3. 注意点：skip link 服务键盘用户。
4. 可追问时补充：多个 nav 时用 aria-label 区分。

**常见追问**

- 为什么 main 加 `tabindex="-1"`？→ 方便跳转后程序化接收焦点，不进入普通 Tab 顺序。

---

### Q15. `aria-live` 用来解决什么问题？

- **难度**: mid
- **标签**: ARIA, 动态内容

**答案**

页面局部内容变化时，读屏用户可能不知道。live region 可以通知辅助技术：

```html
<div aria-live="polite" id="cart-status"></div>
```

- `polite`：等用户当前播报结束后通知
- `assertive`：尽快打断播报，只用于紧急错误
- `role="status"`：等价于 polite 状态区
- `role="alert"`：重要错误，通常 assertive

live region 最好提前存在于 DOM，再更新内容。

**口述模板（30–60 秒）**

1. 一句话结论：aria-live 播报动态变化。
2. 关键机制：辅助技术监听该区域内容变化。
3. 注意点：不要滥用 assertive。
4. 可追问时补充：表单错误和 Toast 场景。

**常见追问**

- 每次接口加载都播报吗？→ 不应该，会制造噪音，只播报对用户有意义的结果。

---

### Q16. `inert` 属性有什么用？

- **难度**: mid
- **标签**: 焦点, a11y

**答案**

`inert` 让元素及其子树不可聚焦、不可点击，并从无障碍树中隐藏。常用于弹窗打开时让背景内容不可交互。

```html
<main inert>...</main>
```

原生 `dialog.showModal()` 会自动让外部内容 inert，所以使用原生模态框时不必手动处理背景。

**口述模板（30–60 秒）**

1. 一句话结论：inert 让一块内容暂时不可交互。
2. 关键机制：同时影响鼠标、焦点和无障碍树。
3. 注意点：别把当前活动区域设成 inert。
4. 可追问时补充：自定义模态框可用 inert 替代复杂焦点陷阱的一部分。

**常见追问**

- 和 `aria-hidden` 区别？→ aria-hidden 只影响辅助技术，inert 还阻止交互。

---

### Q17. `button` 和 `a` 怎么选？

- **难度**: mid
- **标签**: 语义, a11y

**答案**

- 改变 URL / 跳转页面 / 下载资源：用 `a href`
- 执行当前页动作：提交、打开弹窗、切换、删除：用 `button`

错误例子：

```html
<a href="#" onclick="openDialog()">打开</a>
```

问题：语义错误、可能修改 URL、中键 / 新窗口行为奇怪。

**口述模板（30–60 秒）**

1. 一句话结论：导航用链接，动作用按钮。
2. 关键机制：两者默认键盘和浏览器行为不同。
3. 注意点：不要只看外观选元素。
4. 可追问时补充：SPA 路由跳转仍然应是链接。

**常见追问**

- 看起来像按钮的跳转呢？→ 仍用 `a`，样式做成按钮。

---

### Q18. 表单错误提示如何做得可访问？

- **难度**: mid
- **标签**: 表单, a11y

**答案**

要点：

- 错误文字可见，不只靠红色
- 控件设置 `aria-invalid="true"`
- 用 `aria-describedby` 关联错误
- 提交失败后把焦点移到错误汇总或首个错误字段

```html
<label for="phone">手机号</label>
<input id="phone" aria-invalid="true" aria-describedby="phone-error" />
<p id="phone-error">请输入 11 位手机号</p>
```

**口述模板（30–60 秒）**

1. 一句话结论：错误要可见、可关联、可感知。
2. 关键机制：aria-invalid 表达状态，describedby 提供说明。
3. 注意点：不要只靠颜色。
4. 可追问时补充：提交时焦点管理。

**常见追问**

- 错误应实时提示吗？→ 视字段而定，避免用户刚输入一个字符就报错。

---

## 三、SEO 与分享

### Q19. HTML 层面能做哪些 SEO 优化？

- **难度**: mid
- **标签**: SEO

**答案**

- 唯一且准确的 `title`
- 合理的 `meta description`
- 语义结构与清晰标题层级
- 可抓取的 `a href`，不要用 JS click 伪链接
- 图片 alt
- `canonical` 处理重复 URL
- `robots` 控制索引
- 结构化数据 JSON-LD
- 服务端输出关键内容，提升抓取可靠性

**口述模板（30–60 秒）**

1. 一句话结论：HTML SEO 核心是可抓取、可理解、去重复。
2. 关键机制：搜索引擎读取文档结构、链接和元信息。
3. 注意点：标签不是排名作弊工具。
4. 可追问时补充：CSR 页面收录风险。

**常见追问**

- keywords 有用吗？→ 主流搜索引擎基本不依赖。

---

### Q20. canonical 是做什么的？

- **难度**: mid
- **标签**: SEO

**答案**

`canonical` 声明一组重复或高度相似页面中的首选 URL：

```html
<link rel="canonical" href="https://example.com/products/123" />
```

适用于参数排序、追踪参数、移动 / PC 重复页面等。它是强信号，但搜索引擎不一定完全遵循。

**口述模板（30–60 秒）**

1. 一句话结论：canonical 告诉搜索引擎哪个 URL 是主版本。
2. 关键机制：合并重复页面信号。
3. 注意点：写绝对 URL，避免指向错误页面。
4. 可追问时补充：分页不一定都 canonical 到第一页。

**常见追问**

- 301 和 canonical 区别？→ 301 是用户和爬虫都跳转；canonical 只是索引偏好提示。

---

### Q21. 结构化数据 JSON-LD 是什么？

- **难度**: mid
- **标签**: SEO, Schema.org

**答案**

JSON-LD 用 Schema.org 词汇描述页面实体，帮助搜索引擎理解商品、文章、组织、面包屑等信息，可能获得富结果展示。

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "机械键盘",
  "offers": { "@type": "Offer", "price": "399", "priceCurrency": "CNY" }
}
</script>
```

数据必须和页面可见内容一致。

**口述模板（30–60 秒）**

1. 一句话结论：JSON-LD 用结构化方式描述页面实体。
2. 关键机制：搜索引擎解析 Schema.org 类型和属性。
3. 注意点：不能伪造页面不存在的信息。
4. 可追问时补充：富结果不保证展示。

**常见追问**

- 为什么常用 JSON-LD 而不是 Microdata？→ 与 HTML 结构解耦，维护更方便。

---

### Q22. Open Graph 是做什么的？

- **难度**: mid
- **标签**: 分享

**答案**

Open Graph 元信息控制链接分享到社交平台时的标题、描述、图片：

```html
<meta property="og:title" content="活动标题" />
<meta property="og:description" content="活动简介" />
<meta property="og:image" content="https://example.com/share.jpg" />
<meta property="og:url" content="https://example.com/event" />
```

注意：多数抓取器不执行完整前端 JS，元信息需服务端输出；分享图通常需要绝对 URL。

**口述模板（30–60 秒）**

1. 一句话结论：OG 决定分享卡片内容。
2. 关键机制：平台抓取 HTML head。
3. 注意点：SPA 客户端动态修改 head 可能抓不到。
4. 可追问时补充：缓存刷新需要平台调试工具。

**常见追问**

- 微信分享也只靠 OG 吗？→ 国内平台可能有自己的规则或 JS-SDK，需要按平台文档处理。

---

## 四、安全

### Q23. HTML 中的 XSS 是怎么发生的？

- **难度**: mid
- **标签**: XSS, 安全

**答案**

XSS（跨站脚本）是攻击者把不可信内容注入页面，被浏览器当成代码执行。典型原因是把用户输入直接拼进 HTML：

```js
el.innerHTML = userInput
```

正确思路：

- 普通文本用 `textContent`
- 根据输出上下文编码：HTML 文本、属性、URL、JS、CSS 规则不同
- 富文本使用成熟白名单清洗库
- 配合 CSP 降低影响

**口述模板（30–60 秒）**

1. 一句话结论：XSS 是数据被当成代码执行。
2. 关键机制：不可信输入进入 HTML 或脚本上下文。
3. 注意点：转义必须按输出上下文进行。
4. 可追问时补充：CSP、Trusted Types、清洗库。

**常见追问**

- 框架会自动防 XSS 吗？→ 默认插值通常会转义，但 `v-html`、`dangerouslySetInnerHTML` 等仍危险。

---

### Q24. 为什么属性和 URL 也需要单独处理？

- **难度**: mid
- **标签**: XSS, 上下文转义

**答案**

不同位置的解析规则不同：

- HTML 文本：要处理 `<`、`&`
- 属性：还要处理引号
- URL：要限制协议，例如阻止 `javascript:`
- 事件属性：`onclick` 本身就是 JS 上下文，几乎不能安全拼接
- CSS：可能注入危险值

所以「把尖括号转义」不能解决所有 XSS。

**口述模板（30–60 秒）**

1. 一句话结论：转义依赖输出上下文。
2. 关键机制：HTML、属性、URL、JS 解析器不同。
3. 注意点：URL 要做协议白名单。
4. 可追问时补充：避免动态生成内联事件。

**常见追问**

- 只允许 `http` 和 `https` 链接够吗？→ 基本场景可行，还要考虑相对 URL、mailto 等业务需求。

---

### Q25. iframe `sandbox` 有什么用？

- **难度**: mid
- **标签**: iframe, 安全

**答案**

`sandbox` 对 iframe 默认施加大量限制，例如禁止脚本、表单提交、弹窗、顶层导航，并视为不透明源（Opaque Origin）。可以按需开放：

```html
<iframe
  src="https://widget.example.com"
  sandbox="allow-scripts allow-forms"
  title="第三方组件"></iframe>
```

注意：同时开放 `allow-scripts` 和 `allow-same-origin`，且 iframe 与父页面同源时，内部脚本可能移除 sandbox，隔离会失效。

**口述模板（30–60 秒）**

1. 一句话结论：sandbox 是 iframe 权限白名单。
2. 关键机制：默认全禁止，再按需 allow。
3. 注意点：同源 + 脚本 + same-origin 组合要谨慎。
4. 可追问时补充：`allow` 控制摄像头、支付等权限策略。

**常见追问**

- 第三方 iframe 怎么通信？→ postMessage，并校验 origin 和消息格式。

---

### Q26. 点击劫持是什么？HTML/HTTP 怎么防？

- **难度**: mid
- **标签**: 点击劫持, 安全

**答案**

攻击者把目标页面透明嵌入 iframe，诱导用户点击看似无害的位置，实际点了目标页面的敏感按钮。

防护主要靠响应头：

```http
Content-Security-Policy: frame-ancestors 'self'
```

旧方案是 `X-Frame-Options: DENY / SAMEORIGIN`。HTML 中的 meta CSP 不能设置 `frame-ancestors`。

**口述模板（30–60 秒）**

1. 一句话结论：点击劫持是透明 iframe 诱导点击。
2. 关键机制：浏览器根据响应头决定能否被嵌入。
3. 注意点：frame-ancestors 要用 HTTP 头。
4. 可追问时补充：敏感操作二次确认。

**常见追问**

- JS 防 frame busting 可靠吗？→ 不可靠，应使用响应头。

---

## 五、现代原生组件与 DOM

### Q27. `dialog` 元素有什么优势？

- **难度**: mid
- **标签**: dialog, a11y

**答案**

原生 `dialog` 提供：

- `show()` 非模态 / `showModal()` 模态
- 顶层（Top Layer）显示，避免 z-index 战争
- 模态时背景 inert
- Esc 关闭请求
- `::backdrop` 背景样式
- `method="dialog"` 表单关闭并返回值

```html
<dialog id="confirm">
  <form method="dialog">
    <p>确认删除？</p>
    <button value="cancel">取消</button>
    <button value="ok">确认</button>
  </form>
</dialog>
```

**口述模板（30–60 秒）**

1. 一句话结论：dialog 是原生弹窗，减少自研焦点和层级问题。
2. 关键机制：showModal 进入 top layer 并让背景不可交互。
3. 注意点：关闭后焦点恢复与标题命名仍需关注。
4. 可追问时补充：closedby 和 command 是较新增强。

**常见追问**

- dialog 一定要用 showModal 吗？→ 需要模态时用 showModal；非模态浮层可考虑 show 或 Popover。

---

### Q28. Popover API 和 dialog 怎么选？

- **难度**: mid
- **标签**: Popover, dialog

**答案**

- **dialog 模态**：确认、编辑表单、必须处理的任务；背景不可交互
- **Popover**：菜单、下拉、提示卡片、轻量面板；通常非模态、可轻关闭（Light Dismiss）

```html
<button popovertarget="menu">菜单</button>
<div id="menu" popover>...</div>
```

Popover API 在 2025 年进入 Baseline。较新的 `command` / `commandfor` 能统一声明式控制 popover 和 dialog。

**口述模板（30–60 秒）**

1. 一句话结论：要阻断用户就 dialog，要轻量浮层就 popover。
2. 关键机制：两者都可进入 top layer。
3. 注意点：popover 不等于 dialog 语义，菜单/列表还要正确角色和键盘。
4. 可追问时补充：定位可结合 CSS Anchor Positioning。

**常见追问**

- `popover="manual"` 是什么？→ 不会被点击外部或其他 auto popover 自动关闭，需要手动控制。

---

### Q29. `contenteditable` 有哪些坑？

- **难度**: mid
- **标签**: 富文本, 安全

**答案**

`contenteditable` 可以让元素内容可编辑，但：

- 浏览器生成的 HTML 不一致
- 粘贴可能带入复杂样式和危险 HTML
- 选区、撤销、中文输入法组合事件复杂
- 可访问性需要额外处理
- 保存的 HTML 必须清洗

复杂富文本编辑器通常使用 ProseMirror、Lexical、Slate、TipTap 等基于模型的方案。

**口述模板（30–60 秒）**

1. 一句话结论：contenteditable 简单能用，生产富文本很复杂。
2. 关键机制：浏览器直接修改 DOM。
3. 注意点：粘贴和保存 HTML 必须清洗。
4. 可追问时补充：编辑器维护文档模型而不是直接信任 DOM。

**常见追问**

- `contenteditable="plaintext-only"` 有用吗？→ 可减少富文本，但兼容性和业务需求要验证。

---

### Q30. 原生表单的 `FormData` 有什么用？

- **难度**: mid
- **标签**: 表单, DOM API

**答案**

`FormData` 可以按浏览器提交规则收集表单字段，支持文件上传：

```js
form.addEventListener('submit', async (event) => {
  event.preventDefault()
  const data = new FormData(form)
  await fetch('/api/profile', { method: 'POST', body: data })
})
```

它会自动处理 checkbox、多选、文件等字段。直接作为 body 时不要手动设置 `Content-Type`，浏览器会生成 multipart boundary。

**口述模板（30–60 秒）**

1. 一句话结论：FormData 复用原生表单序列化规则。
2. 关键机制：根据 name/value、禁用状态、控件类型收集数据。
3. 注意点：上传时不要手写 multipart Content-Type。
4. 可追问时补充：可转成 URLSearchParams 或对象。

**常见追问**

- 多个同名 checkbox 怎么取？→ `formData.getAll(name)`。

---

### Q31. `autocomplete` 为什么重要？

- **难度**: mid
- **标签**: 表单, 体验

**答案**

`autocomplete` 让浏览器和密码管理器正确自动填充：

```html
<input name="email" type="email" autocomplete="email" />
<input name="password" type="password" autocomplete="current-password" />
<input name="newPassword" type="password" autocomplete="new-password" />
<input name="code" inputmode="numeric" autocomplete="one-time-code" />
```

它提升转化率，也帮助认知障碍或行动不便的用户。

**口述模板（30–60 秒）**

1. 一句话结论：autocomplete 告诉浏览器字段语义。
2. 关键机制：自动填充器根据 token 匹配用户数据。
3. 注意点：不要因安全幻想全部关闭。
4. 可追问时补充：登录、注册、修改密码 token 不同。

**常见追问**

- `autocomplete="off"` 一定生效吗？→ 浏览器可能忽略，尤其是密码字段。

---

### Q32. 什么是 Shadow DOM？它和普通 DOM 有什么区别？

- **难度**: mid
- **标签**: Web Components

**答案**

Shadow DOM 给元素挂载一个隐藏的子 DOM 树，提供：

- 样式封装：内部样式默认不影响外部
- DOM 封装：外部普通选择器不能直接选中内部
- `slot` 内容分发

```js
const root = host.attachShadow({ mode: 'open' })
root.innerHTML = '<button><slot></slot></button>'
```

它不是安全边界；open shadow root 可以通过 `host.shadowRoot` 访问。

**口述模板（30–60 秒）**

1. 一句话结论：Shadow DOM 提供组件级 DOM 和样式封装。
2. 关键机制：host 挂 shadow root，slot 投影外部内容。
3. 注意点：封装不是安全隔离。
4. 可追问时补充：CSS 变量和 `::part` 可穿透定制。

**常见追问**

- closed 模式更安全吗？→ 只能降低误访问，不能作为安全手段。
