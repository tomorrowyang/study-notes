# HTML 学习路线（高级面试向）

> 目标：能在国内互联网高级前端面试中讲清 HTML 语义、可访问性、文档解析与资源加载、SEO、安全和现代原生能力，并能把它们落到性能和工程方案上。

## 面试画像

- **岗位侧重**：前端 / 全栈 / Node SSR；高级岗位会把 HTML 和浏览器渲染、性能、安全一起问
- **假定基础**：写过页面，会基础 CSS/JS，用过 React / Vue 等任一框架
- **学完应能**：
  - 说清语义化、无障碍（Accessibility, a11y）和 SEO 的关系，而不是只背标签列表
  - 从 HTML 字节到首屏讲清解析、阻塞、预加载扫描器（Preload Scanner）和关键渲染路径
  - 正确使用 `async` / `defer` / `module` / `preload` / `prefetch` / `fetchpriority`
  - 写出可访问的表单、弹窗、Tabs，并知道什么时候不该加 ARIA
  - 讲清 XSS、iframe 隔离、CSP、Trusted Types 等 HTML 相关安全边界
  - 了解 `dialog`、Popover、Invoker Commands、Declarative Shadow DOM 等现代原生能力

## 版本说明（本文默认语境）

- HTML 以 **WHATWG HTML Living Standard** 为准，不再按「HTML5 一次性版本」理解
- 兼容性以 MDN Baseline 为准：
  - Popover API：Baseline 2025（较新）
  - Invoker Commands（`command` / `commandfor`）：Baseline 2025 末（较新，生产需兜底）
  - `dialog closedby`、`popover="hint"`、`interestfor`、可定制 `<select>`：以 Chromium 为先，生产使用前查兼容表
- 默认面向现代浏览器；IE 兼容不作为重点

## 知识地图

- **文档基础**
  - DOCTYPE、标准模式 / 怪异模式（Quirks Mode）
  - `head` 元信息：charset、viewport、title、description、canonical
  - 元素分类：内容模型、块级 / 行内只是 CSS 显示概念
- **语义与结构**
  - landmark：`header` / `nav` / `main` / `aside` / `footer` / `search`
  - `article` vs `section`、标题层级、列表、表格
  - 链接与按钮的职责边界
- **表单**
  - `label`、`fieldset` / `legend`、`input type`
  - 原生校验（Constraint Validation）、`autocomplete`
  - 表单提交、`FormData`、按钮默认 `submit`
- **可访问性**
  - 可访问名称（Accessible Name）、角色、状态
  - 键盘与焦点管理、`tabindex`、`inert`
  - ARIA 第一规则：能用原生 HTML 就不用 ARIA
- **解析与加载**
  - HTML 解析器容错、DOM 构建、`document.write`
  - CSS 阻塞渲染、脚本阻塞解析
  - `async` / `defer` / `type="module"`、资源提示（Resource Hints）
  - 图片响应式、懒加载、LCP 优化
- **SEO 与分享**
  - 语义结构、canonical、robots、结构化数据 JSON-LD
  - Open Graph、SSR / 预渲染
- **安全**
  - XSS 与上下文转义、`innerHTML` 风险
  - iframe `sandbox`、`allow`、点击劫持防护
  - CSP、`rel="noopener noreferrer"`、Trusted Types
- **现代原生能力**
  - `dialog`、Popover、Invoker Commands
  - `details` / `summary`、`template`
  - Web Components：Custom Elements、Shadow DOM、`slot`、表单关联
- **页面生命周期**
  - `DOMContentLoaded` / `load` / `pageshow` / `pagehide` / `visibilitychange`
  - 往返缓存（bfcache）

## 建议阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 标签、表单、head、基础语义
2. [中级 Q&A](20-mid-qa.md) — 加载、解析、a11y、SEO、安全
3. [高级 Q&A](30-senior-qa.md) — 渲染链路、性能指标、Web Components、架构取舍
4. [手写与现场编码](40-coding-handwritten.md) — 可访问表单 / 弹窗 / Tabs / 组件
5. [系统设计与场景](50-system-design.md) — SSR 页面、设计系统、UGC、安全
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. 边学边勾 [PROGRESS.md](PROGRESS.md)

## 高级面试必打点

1. 语义化的价值是「机器可理解」：辅助技术、搜索引擎、浏览器默认行为都依赖它
2. `button` 负责动作，`a` 负责导航；用 `div` 模拟按钮要补键盘、焦点、角色、状态
3. ARIA 只能改变无障碍树里的语义，**不会自动带来行为**
4. 每个可交互控件都要有可访问名称；表单控件优先用 `label`
5. CSS 阻塞渲染，同步脚本阻塞解析，并且会等待前面 CSS 的 CSSOM
6. `defer` 按顺序执行，在解析完成后、`DOMContentLoaded` 前；`async` 下载完就执行、不保证顺序
7. 模块脚本默认延迟执行；`async` 也可用于模块脚本
8. `preload` 用于当前页关键资源，`prefetch` 用于未来导航；都只是浏览器提示
9. LCP 图片不能懒加载，可用 `fetchpriority="high"` 和合适的 `srcset/sizes`
10. 预加载扫描器能提前发现 HTML 中的资源；JS 动态插入的资源通常发现得更晚
11. `innerHTML` 拼接用户输入容易 XSS；输出要按 HTML / 属性 / URL / JS 等上下文处理
12. 第三方 iframe 要最小权限：`sandbox`、`allow`、CSP、来源校验
13. `target="_blank"` 的反向窗口风险：现代浏览器默认 noopener，但显式写清更稳
14. `dialog` 适合模态交互，Popover 适合轻量浮层；不要混用语义
15. Web Components 适合跨框架基础组件；样式隔离、SSR、表单关联和可访问性是难点
16. SEO 需要服务端可见的语义 HTML，客户端渲染不是不能收录，但可靠性和时效性更差
17. 页面生命周期优先 `pagehide` / `visibilitychange`，避免依赖 `unload`
18. HTML 的最终目标是可用、可读、可维护，而不是减少标签数量

## 关联技术

- **CSS**：渲染阻塞、层叠、布局与 CLS
- **JavaScript / DOM**：事件、焦点、DOM API、模块加载
- **浏览器原理**：解析器、渲染流水线、网络优先级、缓存
- **React / Vue**：SSR、Hydration、Portal、表单和可访问组件
- **Web 安全**：XSS、CSP、点击劫持、同源策略
- **性能**：Core Web Vitals（LCP、INP、CLS）

## 自测清单（学完打勾）

- [ ] 能不用 ARIA 写出可访问的登录表单
- [ ] 能画出 `script` 普通 / `async` / `defer` / `module` 的执行时间线
- [ ] 能解释一个首屏 LCP 图片为什么慢，并给出 HTML 层面的修复
- [ ] 能讲清 `dialog` 的焦点、Esc、背景不可交互与关闭后焦点恢复
- [ ] 能讲清富文本 / UGC 页面如何防 XSS
