# HTML 高级 Q&A（能扛追问）

> 目标：把 HTML 放到浏览器原理、性能、安全、组件体系和工程治理中讨论。

---

## 一、解析、SSR 与 Hydration

### Q1. 为什么非法 HTML 嵌套会导致 SSR Hydration 不一致？

- **难度**: senior
- **标签**: SSR, Hydration, 解析器

**答案**

HTML 解析器具有容错能力。服务端输出的字符串如果嵌套不合法，浏览器会自动修正 DOM，导致实际 DOM 和框架期望的虚拟树不一致。

典型例子：

```html
<p><div>内容</div></p>
```

浏览器遇到 `div` 时会自动结束 `p`，最终 DOM 结构不是源码表面结构。React / Vue 在 Hydration 时就可能报 mismatch，甚至重新客户端渲染。

其他常见坑：

- `table` 内直接放 `div`，可能触发 foster parenting
- `a` 嵌套交互元素
- 服务端和客户端时间、随机数、浏览器专属 API 输出不同

**口述模板（30–60 秒）**

1. 一句话结论：浏览器会修正非法 HTML，修正后的 DOM 与框架预期不一致。
2. 关键机制：SSR 输出的是字符串，Hydration 对比的是浏览器真实 DOM。
3. 注意点：先修 HTML 合法性，再排查时间和随机值。
4. 可追问时补充：用 HTML validator、框架 dev 警告和 SSR 快照测试。

**常见追问**

- 为什么开发时看不出来？→ 视觉可能一样，但节点层级已改变。

---

### Q2. 什么是流式 HTML（Streaming HTML）？它对首屏有什么价值？

- **难度**: senior
- **标签**: SSR, 性能

**答案**

服务端不必等整页数据都准备完，而是先发送 HTML 头部和首屏壳，再分块发送后续内容。浏览器可以：

- 更早解析 head
- 更早发现 CSS、字体、首屏图片
- 更早绘制首屏框架

React Server Components / Suspense SSR、Vue SSR 等都可利用流式输出。

风险：

- 已发出的 HTTP 状态码和响应头无法再改
- 错误处理更复杂
- 中间代理或压缩配置可能缓冲整个响应
- 页面布局若未预留空间，可能造成 CLS

**口述模板（30–60 秒）**

1. 一句话结论：流式 HTML 让浏览器边收边解析。
2. 关键机制：先发 head 和首屏骨架，慢数据后续补。
3. 注意点：状态码、错误、代理缓冲、CLS 要设计。
4. 可追问时补充：结合 Early Hints 和 preload。

**常见追问**

- 流式输出一定提升 LCP 吗？→ 不一定，LCP 资源仍需尽早发现并获得高优先级。

---

### Q3. 如何从 HTML 层面系统优化 LCP？

- **难度**: senior
- **标签**: LCP, 性能

**答案**

先定位 LCP 元素和瓶颈阶段：TTFB、资源发现延迟、资源加载时间、渲染延迟。

HTML 层面优化：

1. **尽早发现**：LCP 图片直接写在 HTML，避免只由客户端 JS 创建
2. **提高优先级**：`fetchpriority="high"`，必要时 preload
3. **不要懒加载**：LCP 图片禁止 `loading="lazy"`
4. **选择合适尺寸**：`srcset/sizes`、AVIF/WebP
5. **减少渲染阻塞**：关键 CSS、拆分非关键 CSS、脚本 defer
6. **SSR / 流式**：尽早输出首屏 HTML
7. **字体**：关键字体 preload，`font-display` 控制渲染策略

**口述模板（30–60 秒）**

1. 一句话结论：LCP 优化先拆阶段，再对症处理。
2. 关键机制：资源要早发现、高优先级、尺寸合适、渲染不被阻塞。
3. 注意点：不能盲目 preload 一堆资源。
4. 可追问时补充：用 Chrome Performance 和 RUM 数据验证。

**常见追问**

- 背景图作为 LCP 怎么办？→ CSS 背景图发现较晚，可改为 img 或 preload 对应资源。

---

### Q4. HTML 如何影响 INP？

- **难度**: senior
- **标签**: INP, 性能

**答案**

INP（Interaction to Next Paint）衡量交互到下一帧呈现的延迟。HTML 影响主要来自：

- DOM 太大：样式计算、布局、事件委托和查询成本上升
- 原生控件被大量自定义组件替代，增加 JS 工作
- 首屏 Hydration 太重，用户交互排队
- 隐藏但仍在 DOM 中的大量内容增加样式和布局负担
- 第三方脚本同步加载或长任务

优化：

- 减少 DOM 深度和节点数
- 长列表虚拟化，但保持可访问性
- 优先原生控件
- 拆分 Hydration、渐进 Hydration
- 非关键内容延迟渲染或使用 `content-visibility`

**口述模板（30–60 秒）**

1. 一句话结论：HTML 规模和结构会放大交互后的计算成本。
2. 关键机制：交互触发 JS、样式、布局、绘制，DOM 越复杂越慢。
3. 注意点：INP 不只是事件函数慢。
4. 可追问时补充：用 Performance 面板看 input delay、processing、presentation。

**常见追问**

- DOM 节点多少算多？→ 没有绝对阈值，要结合设备、更新频率和长任务分析。

---

### Q5. 如何治理第三方脚本对 HTML 加载的影响？

- **难度**: senior
- **标签**: 第三方脚本, 性能

**答案**

治理思路：

1. 建立清单：每个脚本的业务价值、负责人、加载页面
2. 加载策略：统计和广告优先 `async` 或交互后加载；非必要不放 head
3. 隔离：高风险组件放 iframe 或 Worker（视能力而定）
4. 安全：CSP、SRI（固定版本资源）、权限策略
5. 性能预算：限制主线程时间、请求数、体积
6. 监控：RUM 中关联第三方脚本长任务和错误

避免使用 `document.write` 注入第三方脚本。

**口述模板（30–60 秒）**

1. 一句话结论：第三方脚本要按资产治理，而不是随手插入。
2. 关键机制：控制加载时机、权限和性能预算。
3. 注意点：async 只能改变加载，不消除执行开销。
4. 可追问时补充：标签管理平台需要审批和回滚机制。

**常见追问**

- SRI 能用于动态更新的第三方 SDK 吗？→ 内容变了 hash 会失败，适合固定版本资源。

---

## 二、可访问性工程

### Q6. 什么是无障碍树（Accessibility Tree）？

- **难度**: senior
- **标签**: a11y, 浏览器

**答案**

浏览器会基于 DOM、CSS 和 ARIA 生成无障碍树，暴露给操作系统的无障碍 API，再由读屏器等辅助技术读取。

无障碍节点通常包含：

- 角色（Role）
- 名称（Name）
- 描述（Description）
- 状态（State），如 checked、expanded、disabled
- 层级和关系

CSS 也可能影响无障碍树，例如 `display: none` 会移除；某些 display 修改可能影响表格或列表语义。

**口述模板（30–60 秒）**

1. 一句话结论：读屏器读的是无障碍树，不是像素。
2. 关键机制：浏览器从 DOM、ARIA、CSS 计算角色、名称和状态。
3. 注意点：视觉正确不代表辅助技术正确。
4. 可追问时补充：DevTools Accessibility 面板可查看。

**常见追问**

- ARIA 修改 DOM 吗？→ 不修改 DOM 行为，主要影响无障碍树暴露的信息。

---

### Q7. 设计系统中，什么时候用原生控件，什么时候自研？

- **难度**: senior
- **标签**: 设计系统, a11y

**答案**

优先原生：

- button、checkbox、radio、select、input、details、dialog
- 原生控件自带键盘、焦点、表单提交、自动填充、移动端适配

考虑自研：

- 原生能力无法满足复杂交互，如可搜索 Combobox、多选标签、虚拟化选择器
- 设计要求无法用 CSS 合理实现

自研代价：

- 完整键盘模型
- 焦点管理
- ARIA 角色、状态、关系
- 读屏兼容测试
- 表单提交和校验集成
- 移动端和输入法问题

**口述模板（30–60 秒）**

1. 一句话结论：默认原生，原生满足不了再自研。
2. 关键机制：原生控件提供大量浏览器和系统级能力。
3. 注意点：自研不是换皮，而是重新实现交互规范。
4. 可追问时补充：参考 ARIA Authoring Practices 并做读屏测试。

**常见追问**

- 自定义 select 风格怎么办？→ 先评估原生样式方案；可定制 select 是较新能力，需要兼容策略。

---

### Q8. SPA 路由切换时，无障碍需要处理什么？

- **难度**: senior
- **标签**: SPA, a11y

**答案**

传统页面跳转时，浏览器会刷新文档、焦点回到页面开始、读屏感知新页面。SPA 只局部替换 DOM，用户可能不知道页面已切换。

需要处理：

- 更新 `document.title`
- 焦点移动到新页面 `h1` 或主内容
- 必要时用 live region 通知路由变化
- 返回页面时恢复滚动和焦点
- 链接仍使用 `a href`

**口述模板（30–60 秒）**

1. 一句话结论：SPA 要手动补齐页面切换感知。
2. 关键机制：title、焦点、播报、滚动恢复。
3. 注意点：不要每次都把焦点乱跳到顶部。
4. 可追问时补充：不同路由类型策略不同，如 Tab 切换和页面跳转区分。

**常见追问**

- 焦点放 h1 需要 tabindex 吗？→ 通常加 `tabindex="-1"` 允许程序聚焦。

---

### Q9. 复杂组件（如 Combobox）为什么很难做好？

- **难度**: senior
- **标签**: ARIA, 组件

**答案**

Combobox 同时涉及输入框、弹出列表、选项状态、过滤、异步加载和键盘模型。

需要处理：

- 输入框角色和展开状态
- 列表与输入框关系
- 当前高亮项 `aria-activedescendant`
- 上下键、Enter、Esc、Home/End 等键盘行为
- 异步加载状态与结果数量播报
- 移动端虚拟键盘和输入法组合输入
- 读屏器之间的差异

**口述模板（30–60 秒）**

1. 一句话结论：Combobox 是多个交互模式叠加的组件。
2. 关键机制：焦点仍在输入框，同时要表达列表中的当前项。
3. 注意点：不同读屏器和浏览器组合差异大。
4. 可追问时补充：优先使用成熟组件库，并做真实辅助技术测试。

**常见追问**

- 为什么不让焦点直接移到选项？→ 用户还需要继续输入，常用 activedescendant 保留输入焦点。

---

### Q10. 如何把 a11y 纳入工程流程？

- **难度**: senior
- **标签**: 工程化, a11y

**答案**

分层治理：

1. **设计阶段**：对比度、焦点态、错误状态、交互规范
2. **组件阶段**：设计系统内置可访问性，提供使用约束
3. **开发阶段**：ESLint a11y 插件、组件单测检查角色和名称
4. **CI 阶段**：axe 等自动化扫描关键页面
5. **测试阶段**：键盘路径、读屏抽测、缩放和高对比
6. **线上反馈**：可访问性问题入口和缺陷分级

自动化工具只能发现部分问题，键盘和读屏人工测试必不可少。

**口述模板（30–60 秒）**

1. 一句话结论：a11y 要前移到设计系统和 CI。
2. 关键机制：组件内置能力，自动化扫基本错误，人工测交互。
3. 注意点：自动化不能证明可用。
4. 可追问时补充：建立关键用户路径的验收清单。

**常见追问**

- 测试 Library 中为什么优先 `getByRole`？→ 它更接近用户和辅助技术理解页面的方式。

---

## 三、Web Components

### Q11. Custom Elements 的生命周期有哪些？

- **难度**: senior
- **标签**: Web Components

**答案**

常用回调：

- `constructor`：初始化，不应读取属性或子元素，也不应添加属性
- `connectedCallback`：插入文档时
- `disconnectedCallback`：移出文档时，清理监听器
- `attributeChangedCallback`：观察的属性变化时
- `adoptedCallback`：移动到新文档时

```js
class UserCard extends HTMLElement {
  static observedAttributes = ['name']

  attributeChangedCallback(name, oldValue, newValue) {
    this.render()
  }
}
customElements.define('user-card', UserCard)
```

**口述模板（30–60 秒）**

1. 一句话结论：Custom Element 生命周期围绕创建、连接、属性变化和断开。
2. 关键机制：浏览器在元素升级和 DOM 变化时调用回调。
3. 注意点：constructor 中不要依赖外部属性和子节点。
4. 可追问时补充：断开时清理资源避免泄漏。

**常见追问**

- 元素名有什么要求？→ 必须包含短横线，例如 `user-card`。

---

### Q12. 什么是 Custom Element 升级（Upgrade）？如何避免 FOUC？

- **难度**: senior
- **标签**: Web Components, 渲染

**答案**

HTML 中可以先出现 `<user-card>`，此时它只是未知元素。之后 `customElements.define` 执行，浏览器会把已有元素升级为自定义元素。

升级前可能出现未样式化或布局跳动。处理：

```css
user-card:not(:defined) {
  visibility: hidden;
}
```

或使用 Declarative Shadow DOM 让服务端直接输出 shadow root。

**口述模板（30–60 秒）**

1. 一句话结论：自定义元素可以先出现在 DOM 中，后续定义时升级。
2. 关键机制：浏览器在 define 后调用构造和连接逻辑。
3. 注意点：升级前样式和内容可能闪烁。
4. 可追问时补充：`customElements.whenDefined()` 可等待定义。

**常见追问**

- 隐藏未定义元素有什么风险？→ JS 加载失败时内容可能一直不可见，要考虑降级。

---

### Q13. Declarative Shadow DOM 解决什么问题？

- **难度**: senior
- **标签**: SSR, Shadow DOM

**答案**

传统 Shadow DOM 需要 JS 调用 `attachShadow`，服务端渲染不友好。Declarative Shadow DOM 允许 HTML 直接声明 shadow root：

```html
<user-card>
  <template shadowrootmode="open">
    <style>p { color: #333; }</style>
    <p><slot></slot></p>
  </template>
  张三
</user-card>
```

收益：

- SSR 首屏能直接显示组件内部结构
- 减少 JS 执行前的空白或闪烁
- 为 Web Components SSR 提供标准机制

**口述模板（30–60 秒）**

1. 一句话结论：DSD 让 shadow root 可以由 HTML 声明。
2. 关键机制：解析器遇到特定 template 时创建 shadow root。
3. 注意点：客户端组件仍需要正确接管已有 shadow root。
4. 可追问时补充：注意动态插入 HTML 时的解析 API 差异。

**常见追问**

- `innerHTML` 插入 DSD 会生效吗？→ 普通 innerHTML 有安全和解析限制，需用支持的解析 API。

---

### Q14. 自定义表单控件如何参与原生表单？

- **难度**: senior
- **标签**: ElementInternals, 表单

**答案**

使用 Form-associated Custom Elements：

```js
class RatingInput extends HTMLElement {
  static formAssociated = true

  constructor() {
    super()
    this.internals = this.attachInternals()
  }

  set value(next) {
    this._value = next
    this.internals.setFormValue(String(next))
  }
}
```

`ElementInternals` 可以：

- 设置提交值
- 设置校验状态
- 参与 form reset、disabled 等生命周期
- 暴露部分默认 ARIA 语义

**口述模板（30–60 秒）**

1. 一句话结论：formAssociated + ElementInternals 让组件像原生控件一样参与表单。
2. 关键机制：setFormValue 和 setValidity 接入表单系统。
3. 注意点：键盘、焦点、名称、状态仍要实现。
4. 可追问时补充：设计系统跨框架组件会用到。

**常见追问**

- 不用它会怎样？→ 要额外隐藏 input 同步值，容易状态不一致。

---

### Q15. 为什么 customized built-in elements（`is` 属性）很少在生产中用？

- **难度**: senior
- **标签**: Web Components, 兼容性

**答案**

customized built-in elements 写法：

```html
<button is="fancy-button">提交</button>
```

它能继承原生 button 语义和行为，但 Safari 长期不支持该规范能力。因此跨浏览器组件库更常用自主自定义元素（Autonomous Custom Elements），如 `<fancy-button>`，再在内部使用原生 `button`。

**口述模板（30–60 秒）**

1. 一句话结论：`is` 能扩展原生元素，但兼容性限制大。
2. 关键机制：它继承内置元素语义和行为。
3. 注意点：Safari 不支持导致生产风险高。
4. 可追问时补充：可用 polyfill，但要评估复杂度。

**常见追问**

- Autonomous 元素如何保持按钮语义？→ 内部包裹原生 button，并转发必要属性。

---

### Q16. Web Components 适合做微前端或跨框架组件库吗？

- **难度**: senior
- **标签**: 架构, Web Components

**答案**

适合：

- 跨 React、Vue、Angular 共享基础 UI
- 需要长期稳定、低框架绑定的组件
- 页面局部嵌入组件

难点：

- 属性只能是字符串，复杂数据要用 property
- 框架事件绑定和类型支持差异
- SSR 和 Hydration
- 样式主题定制
- 表单和可访问性
- 版本冲突：同名 custom element 只能定义一次

微前端中它更适合作为「组件边界」，不天然提供路由、依赖共享、发布治理。

**口述模板（30–60 秒）**

1. 一句话结论：Web Components 适合跨框架组件，不等于完整微前端方案。
2. 关键机制：浏览器原生组件模型减少框架绑定。
3. 注意点：数据传递、事件、SSR、主题和版本冲突要设计。
4. 可追问时补充：可与 iframe、Module Federation 按场景组合。

**常见追问**

- 两个版本组件库都定义 `x-button` 怎么办？→ 需要命名空间、版本前缀或宿主统一依赖。

---

## 四、安全与隔离

### Q17. 如何安全展示用户提交的富文本 HTML？

- **难度**: senior
- **标签**: XSS, UGC

**答案**

完整方案：

1. **输入阶段**：编辑器只产生受控结构，避免任意 HTML
2. **服务端清洗**：使用白名单，限制标签、属性、URL 协议
3. **存储**：保存清洗后的版本，并记录原始内容以便审计
4. **输出阶段**：前端再次清洗或使用安全渲染 API
5. **CSP**：限制脚本来源，降低漏网影响
6. **Trusted Types**：在支持的浏览器中限制危险 DOM 注入点

注意 mXSS（Mutation XSS）：HTML 被浏览器解析、序列化、再解析时可能变形。不要自己用正则写 HTML 清洗器。

**口述模板（30–60 秒）**

1. 一句话结论：UGC HTML 要多层防护，核心是白名单清洗。
2. 关键机制：限制标签、属性、协议，并在输出时安全插入。
3. 注意点：正则清洗和黑名单都不可靠。
4. 可追问时补充：CSP 和 Trusted Types 是兜底。

**常见追问**

- 前端清洗就够吗？→ 不够，接口可能被绕过，也有多端展示需求。

---

### Q18. CSP 如何和 HTML 脚本加载配合？

- **难度**: senior
- **标签**: CSP, 安全

**答案**

CSP（Content Security Policy）通过响应头限制资源来源和脚本执行。

现代推荐思路：

```http
Content-Security-Policy:
  script-src 'nonce-随机值' 'strict-dynamic';
  object-src 'none';
  base-uri 'none';
```

HTML：

```html
<script nonce="随机值" src="/app.js"></script>
```

要点：

- 每次响应生成不可预测 nonce
- 避免 `unsafe-inline`、`unsafe-eval`
- `strict-dynamic` 允许受信任脚本加载的依赖
- 先用 Report-Only 观察，再强制执行

**口述模板（30–60 秒）**

1. 一句话结论：CSP 控制哪些脚本和资源能执行或加载。
2. 关键机制：nonce/hash 证明脚本可信。
3. 注意点：缓存页面时 nonce 也要每次变化。
4. 可追问时补充：Report-Only 灰度上线。

**常见追问**

- CSP 能替代转义吗？→ 不能，它是纵深防御。

---

### Q19. Trusted Types 解决什么问题？

- **难度**: senior
- **标签**: Trusted Types, XSS

**答案**

Trusted Types 用于防 DOM XSS。启用后，`innerHTML`、`outerHTML`、动态脚本 URL 等危险注入点不再接受普通字符串，只接受经过受控策略生成的 Trusted 类型对象。

```http
Content-Security-Policy: require-trusted-types-for 'script'
```

价值：

- 把危险注入点集中到少数策略中审计
- 防止开发者随手 `innerHTML = userInput`

兼容性以浏览器兼容表为准，生产要有降级和清洗策略。

**口述模板（30–60 秒）**

1. 一句话结论：Trusted Types 让危险 DOM 注入必须经过安全策略。
2. 关键机制：危险 sink 只接受受信任类型。
3. 注意点：策略本身仍要可靠。
4. 可追问时补充：适合大型代码库治理 DOM XSS。

**常见追问**

- 框架项目也有用吗？→ 有，尤其是存在第三方库或手写 DOM 注入的项目。

---

### Q20. iframe 微前端与组件化微前端如何选？

- **难度**: senior
- **标签**: iframe, 微前端

**答案**

| 方案 | 优点 | 缺点 |
|------|------|------|
| iframe | 隔离强、技术栈独立、故障边界清晰 | 通信复杂、性能开销、弹窗和路由同步难、SEO 差 |
| Web Components / JS 沙箱 | 体验一体、共享样式和状态方便 | 隔离弱、依赖和样式冲突风险 |
| 构建时组合 | 性能好、类型清晰 | 独立部署能力弱 |

第三方、支付、强隔离系统倾向 iframe；同一产品内高频交互模块倾向组件化。

**口述模板（30–60 秒）**

1. 一句话结论：iframe 换隔离，组件化换体验。
2. 关键机制：iframe 是独立浏览上下文。
3. 注意点：通信、路由、弹窗和性能要提前设计。
4. 可追问时补充：postMessage 协议、版本兼容和超时处理。

**常见追问**

- iframe 弹窗为何遮不住父页面？→ 它只能在 iframe 自身视口内渲染，通常需要父页面代为展示。

---

### Q21. `postMessage` 通信如何保证安全？

- **难度**: senior
- **标签**: iframe, 安全

**答案**

安全要点：

```js
window.addEventListener('message', (event) => {
  if (event.origin !== 'https://trusted.example.com') return
  if (!isValidMessage(event.data)) return
  handleMessage(event.data)
})
```

- 发送时指定目标 origin，避免 `*`
- 接收时校验 `event.origin`
- 校验 `event.source` 是否是预期窗口
- 校验消息结构和版本
- 不把消息内容直接插入 HTML 或执行
- 敏感操作需要额外鉴权

**口述模板（30–60 秒）**

1. 一句话结论：postMessage 要双向校验来源和数据。
2. 关键机制：浏览器提供 origin 和 source。
3. 注意点：不要信任 data 内容。
4. 可追问时补充：协议加版本号和请求 ID。

**常见追问**

- 只校验 origin 够吗？→ 不够，还应校验 source 和消息格式。

---

## 五、现代能力与架构取舍

### Q22. `command` / `commandfor` 和 Popover 的关系是什么？如何渐进增强？

- **难度**: senior
- **标签**: Invoker Commands, Popover, dialog

**答案**

`command` / `commandfor` 是声明式按钮控制机制：

```html
<button commandfor="edit-dialog" command="show-modal">编辑</button>
<dialog id="edit-dialog">
  <button commandfor="edit-dialog" command="close">关闭</button>
</dialog>
```

它可统一控制 popover 和 dialog，并支持自定义命令事件。它在 2025 年末才进入 Baseline，生产中应：

- 做特性检测
- 保留 JS 回退
- 对旧浏览器确保关键操作可用

`dialog closedby` 和 `popover="hint"`、`interestfor` 等更新能力也要按兼容性渐进使用。

**口述模板（30–60 秒）**

1. 一句话结论：command 让按钮声明式控制其他元素。
2. 关键机制：commandfor 指目标，command 指动作。
3. 注意点：新特性要特性检测和 JS 回退。
4. 可追问时补充：它能减少 JS 下载前不可交互的问题。

**常见追问**

- 和 `popovertarget` 有什么关系？→ command/commandfor 是更通用的新方案，能覆盖 popover 和 dialog 操作。

---

### Q23. CSR、SSR、SSG、ISR 对 HTML 和 SEO 有什么影响？

- **难度**: senior
- **标签**: 渲染模式, SEO

**答案**

| 模式 | HTML 内容 | 适用 |
|------|-----------|------|
| CSR | 初始 HTML 多为壳，JS 后渲染 | 后台、强交互应用 |
| SSR | 请求时生成完整 HTML | 个性化、SEO、首屏敏感页 |
| SSG | 构建时生成 HTML | 文档、博客、营销页 |
| ISR / 增量再生成 | 静态页按策略更新 | 大量内容页、更新不频繁 |

SEO 页面优先让核心内容、链接、title、description、结构化数据在服务端输出。

**口述模板（30–60 秒）**

1. 一句话结论：渲染模式决定首个 HTML 里有什么。
2. 关键机制：搜索和分享抓取器优先读取 HTML。
3. 注意点：SSR 也可能因 Hydration 太重导致交互慢。
4. 可追问时补充：按页面类型混合渲染。

**常见追问**

- CSR 能被搜索引擎收录吗？→ 可能，但抓取渲染时机、资源失败和社交平台抓取都更不可控。

---

### Q24. 国际化页面在 HTML 层面要注意什么？

- **难度**: senior
- **标签**: i18n

**答案**

- 正确 `lang`，局部外语内容也可设置 `lang`
- 阿拉伯语、希伯来语等使用 `dir="rtl"`
- 用户生成内容方向未知时可用 `dir="auto"`
- 多语言版本使用 `hreflang`
- 日期、货币、数字由 Intl 或服务端本地化
- 不要把文本拼接在 HTML 字符串中，语序会变化
- CSS 优先逻辑属性，如 `margin-inline-start`

```html
<link rel="alternate" hreflang="en" href="https://example.com/en/" />
```

**口述模板（30–60 秒）**

1. 一句话结论：i18n 不只是翻译文本，还包括语言、方向和格式。
2. 关键机制：lang、dir、hreflang 被浏览器和搜索引擎使用。
3. 注意点：RTL 布局要用逻辑属性。
4. 可追问时补充：字体和断词也要测试。

**常见追问**

- 混合中英文用户名如何显示？→ 用 `bdi` 或 `dir="auto"` 隔离双向文本。

---

### Q25. 渐进增强（Progressive Enhancement）在现代 SPA 中还有意义吗？

- **难度**: senior
- **标签**: 架构, 可靠性

**答案**

有意义。渐进增强的核心不是「完全不用 JS」，而是：

- 基础内容和关键操作尽量依赖可靠的 HTML / HTTP
- JS 增强体验，而不是成为唯一可用路径
- 网络慢、JS 加载失败、旧浏览器时仍有降级路径

现代实践：

- SSR 输出内容
- 表单可原生提交，JS 拦截后增强为无刷新
- 链接使用真实 URL
- 新 HTML 能力带特性检测和回退

**口述模板（30–60 秒）**

1. 一句话结论：渐进增强是可靠性策略，不是反 JS。
2. 关键机制：先保证 HTML 可用，再叠加 JS 体验。
3. 注意点：所有功能都无 JS 可用成本可能太高，要按关键路径取舍。
4. 可追问时补充：Remix / Next Server Actions 等也体现这一思想。

**常见追问**

- 后台系统也要做吗？→ 不必完全无 JS，但关键数据不丢、错误可恢复仍重要。
