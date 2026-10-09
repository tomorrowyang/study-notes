# HTML 手写与现场编码

> 使用方式：先自己写结构并讲思路，再对照参考实现。HTML 手写题重点看语义、可访问性、浏览器默认行为和边界处理。

---

### H1. 写一个语义化的文章详情页骨架

- **难度**: junior
- **标签**: 语义化, landmark
- **面试官看什么**: 是否能合理选择 landmark、标题层级和文章结构

**题干**

页面包含顶部导航、文章正文、作者信息、相关推荐和页脚。请写出 HTML 骨架。

**思路口述**

1. 页面级 `header`、`nav`、`main`、`aside`、`footer`
2. 正文是可独立分发内容，用 `article`
3. 文章内部用标题、时间、段落等语义元素
4. 提供跳到主要内容的链接

**参考实现**

```html
<body>
  <a class="skip-link" href="#main">跳到主要内容</a>

  <header>
    <a href="/" aria-label="返回首页">
      <img src="/logo.svg" width="120" height="32" alt="技术博客" />
    </a>
    <nav aria-label="主导航">
      <ul>
        <li><a href="/frontend">前端</a></li>
        <li><a href="/backend">后端</a></li>
      </ul>
    </nav>
  </header>

  <main id="main" tabindex="-1">
    <article>
      <header>
        <h1>HTML 解析与首屏性能</h1>
        <p>
          作者 <a href="/authors/zhangsan">张三</a>
          发布于 <time datetime="2026-10-09">2026 年 10 月 9 日</time>
        </p>
      </header>

      <section aria-labelledby="parse-title">
        <h2 id="parse-title">浏览器如何解析 HTML</h2>
        <p>...</p>
      </section>

      <footer>
        <p>标签：<a href="/tags/html">HTML</a></p>
      </footer>
    </article>
  </main>

  <aside aria-labelledby="related-title">
    <h2 id="related-title">相关推荐</h2>
    <ul>
      <li><a href="/posts/css-rendering">CSS 渲染路径</a></li>
    </ul>
  </aside>

  <footer>
    <p>&copy; 2026 技术博客</p>
  </footer>
</body>
```

**易错点**

- 全部用 `div`
- 使用多个无标签 `nav`，读屏用户难以区分
- 标题层级只按字号选择
- 时间没有机器可读的 `datetime`

---

### H2. 写一个可访问的登录表单

- **难度**: mid
- **标签**: 表单, a11y
- **面试官看什么**: label、autocomplete、错误提示、提交行为

**题干**

实现手机号 + 密码登录表单。手机号必填且为 11 位数字；提交失败时显示错误，并让读屏用户感知。

**思路口述**

1. 每个控件用 `label` 命名
2. 手机号用 `type="tel"` 和 `inputmode="numeric"`
3. 正确设置 `autocomplete`
4. 错误通过 `aria-describedby` 关联，并设置 `aria-invalid`
5. 提交失败时聚焦第一个错误字段

**参考实现**

```html
<form id="login-form" novalidate>
  <h1>登录</h1>

  <div>
    <label for="phone">手机号</label>
    <input
      id="phone"
      name="phone"
      type="tel"
      inputmode="numeric"
      autocomplete="tel"
      required
      pattern="1\d{10}"
      aria-describedby="phone-hint phone-error" />
    <p id="phone-hint">请输入 11 位中国大陆手机号</p>
    <p id="phone-error" class="error" hidden></p>
  </div>

  <div>
    <label for="password">密码</label>
    <input
      id="password"
      name="password"
      type="password"
      autocomplete="current-password"
      required
      minlength="8"
      aria-describedby="password-error" />
    <p id="password-error" class="error" hidden></p>
  </div>

  <button type="submit">登录</button>
</form>
```

```js
const form = document.querySelector('#login-form')

function showFieldError(input, message) {
  const error = document.getElementById(`${input.id}-error`)
  input.setAttribute('aria-invalid', 'true')
  error.textContent = message
  error.hidden = false
}

function clearFieldError(input) {
  const error = document.getElementById(`${input.id}-error`)
  input.removeAttribute('aria-invalid')
  error.textContent = ''
  error.hidden = true
}

form.addEventListener('submit', (event) => {
  event.preventDefault()
  const fields = [...form.elements].filter((el) => el instanceof HTMLInputElement)
  let firstInvalid = null

  for (const field of fields) {
    clearFieldError(field)
    if (!field.validity.valid) {
      showFieldError(field, field.validationMessage)
      firstInvalid ??= field
    }
  }

  if (firstInvalid) {
    firstInvalid.focus()
    return
  }

  // 提交到接口
})
```

**易错点**

- 用 placeholder 代替 label
- 关闭原生校验后没有自行提供可访问错误
- 错误只用红色边框表达
- 密码输入框没有正确 autocomplete

---

### H3. 使用原生 `dialog` 实现删除确认

- **难度**: mid
- **标签**: dialog, Promise
- **面试官看什么**: 模态语义、返回值、关闭行为、焦点

**题干**

点击删除按钮后弹出确认框。用户确认返回 `true`，取消或按 Esc 返回 `false`。

**思路口述**

1. 用 `showModal()` 打开模态框
2. 用 `form method="dialog"` 让按钮直接关闭弹窗
3. 通过 `returnValue` 判断用户选择
4. 关闭事件只监听一次，避免重复 resolve

**参考实现**

```html
<button type="button" id="delete-button">删除项目</button>

<dialog id="confirm-dialog" aria-labelledby="confirm-title">
  <form method="dialog">
    <h2 id="confirm-title">确认删除项目？</h2>
    <p>删除后不可恢复。</p>
    <button value="cancel" autofocus>取消</button>
    <button value="confirm">删除</button>
  </form>
</dialog>
```

```js
/** 打开确认框，并在关闭时返回用户是否确认。 */
function openConfirmDialog(dialog) {
  return new Promise((resolve) => {
    dialog.returnValue = ''
    dialog.addEventListener(
      'close',
      () => resolve(dialog.returnValue === 'confirm'),
      { once: true },
    )
    dialog.showModal()
  })
}

document.querySelector('#delete-button').addEventListener('click', async () => {
  const confirmed = await openConfirmDialog(document.querySelector('#confirm-dialog'))
  if (confirmed) {
    // 执行删除
  }
})
```

**易错点**

- 用 `show()` 却期望背景不可交互
- 不重置 `returnValue`，导致 Esc 关闭时读到上一次结果
- 危险操作默认聚焦「删除」
- 弹窗没有标题或可访问名称

---

### H4. 实现可访问的 Tabs

- **难度**: senior
- **标签**: ARIA, 键盘交互
- **面试官看什么**: role 关系、roving tabindex、方向键、隐藏面板

**题干**

实现「资料 / 安全 / 通知」三个标签页。要求鼠标点击可切换，键盘支持左右方向键、Home、End。

**思路口述**

1. `tablist` 包含多个 `tab`
2. tab 用 `aria-controls` 指向 panel
3. panel 用 `aria-labelledby` 指回 tab
4. 当前 tab `tabindex=0`，其他为 `-1`
5. 方向键移动并激活 tab

**参考实现**

```html
<div class="tabs">
  <div role="tablist" aria-label="账户设置">
    <button role="tab" id="tab-profile" aria-controls="panel-profile" aria-selected="true">
      资料
    </button>
    <button role="tab" id="tab-security" aria-controls="panel-security" aria-selected="false" tabindex="-1">
      安全
    </button>
    <button role="tab" id="tab-notify" aria-controls="panel-notify" aria-selected="false" tabindex="-1">
      通知
    </button>
  </div>

  <section role="tabpanel" id="panel-profile" aria-labelledby="tab-profile">资料设置</section>
  <section role="tabpanel" id="panel-security" aria-labelledby="tab-security" hidden>安全设置</section>
  <section role="tabpanel" id="panel-notify" aria-labelledby="tab-notify" hidden>通知设置</section>
</div>
```

```js
/** 为 Tabs 绑定点击和键盘交互。 */
function setupTabs(root) {
  const tabs = [...root.querySelectorAll('[role="tab"]')]

  function selectTab(nextTab) {
    for (const tab of tabs) {
      const selected = tab === nextTab
      const panel = document.getElementById(tab.getAttribute('aria-controls'))
      tab.setAttribute('aria-selected', String(selected))
      tab.tabIndex = selected ? 0 : -1
      panel.hidden = !selected
    }
    nextTab.focus()
  }

  root.addEventListener('click', (event) => {
    const tab = event.target.closest('[role="tab"]')
    if (tab && root.contains(tab)) selectTab(tab)
  })

  root.addEventListener('keydown', (event) => {
    const currentIndex = tabs.indexOf(document.activeElement)
    if (currentIndex === -1) return

    const nextIndexByKey = {
      ArrowRight: currentIndex + 1,
      ArrowLeft: currentIndex - 1,
      Home: 0,
      End: tabs.length - 1,
    }
    if (!(event.key in nextIndexByKey)) return

    event.preventDefault()
    const nextIndex = (nextIndexByKey[event.key] + tabs.length) % tabs.length
    selectTab(tabs[nextIndex])
  })
}

setupTabs(document.querySelector('.tabs'))
```

**易错点**

- 所有 tab 都能 Tab 聚焦，导致键盘路径过长
- 只改样式不改 `aria-selected`
- 面板只做视觉隐藏，读屏仍会读到
- 没有为 tablist 提供名称

---

### H5. 写一个首屏 Hero 图（LCP 优化）

- **难度**: mid
- **标签**: 图片, LCP, 性能
- **面试官看什么**: 格式回退、尺寸、优先级、CLS

**题干**

首页首屏有一张全宽 Banner，是 LCP 元素。请写 HTML，兼顾 AVIF/WebP/JPEG 回退和不同屏幕尺寸。

**思路口述**

1. 图片直接写在 HTML 中，便于预加载扫描器发现
2. `picture` 提供格式回退
3. `srcset/sizes` 选择合适尺寸
4. `width/height` 预留空间
5. `fetchpriority="high"`，不使用 lazy

**参考实现**

```html
<picture>
  <source
    type="image/avif"
    srcset="/hero-800.avif 800w, /hero-1200.avif 1200w, /hero-1600.avif 1600w"
    sizes="100vw" />
  <source
    type="image/webp"
    srcset="/hero-800.webp 800w, /hero-1200.webp 1200w, /hero-1600.webp 1600w"
    sizes="100vw" />
  <img
    src="/hero-1200.jpg"
    srcset="/hero-800.jpg 800w, /hero-1200.jpg 1200w, /hero-1600.jpg 1600w"
    sizes="100vw"
    width="1600"
    height="800"
    fetchpriority="high"
    alt="双十一数码产品低至五折" />
</picture>
```

如果图片由客户端 JS 才插入，可在 head 中补 preload：

```html
<link
  rel="preload"
  as="image"
  type="image/avif"
  imagesrcset="/hero-800.avif 800w, /hero-1200.avif 1200w, /hero-1600.avif 1600w"
  imagesizes="100vw"
  fetchpriority="high" />
```

**易错点**

- LCP 图片 `loading="lazy"`
- `sizes` 与真实布局不一致
- 文字放在图片里，导致 SEO 和可访问性差
- preload 与真实请求 URL 不一致，造成重复下载

---

### H6. 手写 HTML 转义与安全模板标签

- **难度**: mid
- **标签**: XSS, 字符串处理
- **面试官看什么**: 转义字符完整性、上下文边界

**题干**

实现 `escapeHTML(value)` 和模板标签 `html`，让插值默认按 HTML 文本转义。

**思路口述**

1. 转义 `&`、`<`、`>`、`"`、`'`
2. `&` 必须被正确处理，避免实体混淆
3. 模板标签只转义插值，不转义静态模板
4. 说明它只适合 HTML 文本和带引号的属性值

**参考实现**

```js
const HTML_ESCAPE_MAP = {
  '&': '&amp;',
  '<': '&lt;',
  '>': '&gt;',
  '"': '&quot;',
  "'": '&#39;',
}

/** 将任意值转换为可安全放入 HTML 文本或带引号属性值的字符串。 */
export function escapeHTML(value) {
  return String(value).replace(/[&<>"']/g, (char) => HTML_ESCAPE_MAP[char])
}

/** 模板插值默认转义的 HTML 模板标签。 */
export function html(strings, ...values) {
  return strings.reduce((result, part, index) => {
    const value = index < values.length ? escapeHTML(values[index]) : ''
    return result + part + value
  }, '')
}

const name = '<img src=x onerror=alert(1)>'
const markup = html`<p title="${name}">${name}</p>`
```

**易错点**

- 只转义 `<` 和 `>`
- 认为这个函数能用于 URL、JS、CSS 上下文
- 对已经转义过的内容重复转义

---

### H7. 实现一个 `<copy-button>` Web Component

- **难度**: senior
- **标签**: Custom Elements, Shadow DOM
- **面试官看什么**: 生命周期、事件、可访问状态、样式封装

**题干**

实现 `<copy-button text="要复制的内容">复制</copy-button>`。点击复制，成功或失败都要让读屏用户感知，并向外派发事件。

**思路口述**

1. 继承 `HTMLElement`
2. 用 Shadow DOM 封装按钮和状态区域
3. `slot` 支持外部自定义按钮文本
4. 连接时绑定事件，断开时移除
5. 用 `role="status"` 播报结果

**参考实现**

```js
/** 将 text 属性复制到剪贴板的可复用按钮组件。 */
class CopyButton extends HTMLElement {
  #button
  #status

  constructor() {
    super()
    const root = this.attachShadow({ mode: 'open' })
    root.innerHTML = `
      <style>
        button { font: inherit; }
        .status { margin-inline-start: 8px; }
      </style>
      <button type="button" part="button"><slot>复制</slot></button>
      <span class="status" role="status"></span>
    `
    this.#button = root.querySelector('button')
    this.#status = root.querySelector('.status')
  }

  connectedCallback() {
    this.#button.addEventListener('click', this.#handleClick)
  }

  disconnectedCallback() {
    this.#button.removeEventListener('click', this.#handleClick)
  }

  #handleClick = async () => {
    const text = this.getAttribute('text') ?? ''

    try {
      await navigator.clipboard.writeText(text)
      this.#status.textContent = '已复制'
      this.dispatchEvent(
        new CustomEvent('copy-success', {
          bubbles: true,
          composed: true,
          detail: { text },
        }),
      )
    } catch (error) {
      this.#status.textContent = '复制失败，请手动复制'
      this.dispatchEvent(
        new CustomEvent('copy-error', {
          bubbles: true,
          composed: true,
          detail: { error },
        }),
      )
    }
  }
}

customElements.define('copy-button', CopyButton)
```

**易错点**

- 不清理事件监听
- 自定义事件没设置 `composed`，跨 Shadow DOM 传播受限
- 只显示视觉 Toast，没有状态播报
- 忽略 Clipboard API 需要安全上下文和用户手势

---

### H8. 判断简化 HTML 字符串标签是否闭合

- **难度**: mid
- **标签**: 栈, 字符串
- **面试官看什么**: 栈的应用、空元素处理、能否说明真实 HTML 解析的限制

**题干**

实现 `isBalancedHTML(html)`，判断简化 HTML 中普通标签是否正确闭合。需要支持 `br`、`img` 等空元素，以及 `<br />` 写法。

**思路口述**

1. 用正则依次提取开始标签和结束标签
2. 开始标签入栈，结束标签和栈顶比较
3. 空元素和自闭合写法不入栈
4. 最后栈为空才合法
5. 明确说明这不是完整 HTML 解析器

**参考实现**

```js
const VOID_ELEMENTS = new Set([
  'area', 'base', 'br', 'col', 'embed', 'hr', 'img',
  'input', 'link', 'meta', 'source', 'track', 'wbr',
])

/**
 * 校验简化 HTML 的标签闭合关系。
 * 不处理注释、脚本内容、属性值中的 > 等真实 HTML 解析细节。
 */
function isBalancedHTML(html) {
  const stack = []
  const tagPattern = /<\/?([a-zA-Z][\w-]*)(?=[\s/>])[^>]*?(\/?)>/g

  for (const match of html.matchAll(tagPattern)) {
    const [rawTag, rawName, selfClosingSlash] = match
    const tagName = rawName.toLowerCase()

    if (rawTag.startsWith('</')) {
      if (stack.pop() !== tagName) return false
      continue
    }

    if (!VOID_ELEMENTS.has(tagName) && !selfClosingSlash) {
      stack.push(tagName)
    }
  }

  return stack.length === 0
}

isBalancedHTML('<div><p>Hello<br /></p></div>') // true
isBalancedHTML('<div><span></div></span>') // false
```

**易错点**

- 把 `img`、`br` 入栈
- 标签名大小写不一致
- 声称正则可以完整解析 HTML

---

### H9. 审查并修复一段问题 HTML

- **难度**: mid
- **标签**: 代码审查, a11y
- **面试官看什么**: 能否发现语义、可访问性和安全问题

**题干**

找出问题并修复：

```html
<div class="header">
  <div class="title">商品列表</div>
  <a href="#" onclick="openFilter()">筛选</a>
</div>

<div class="list">
  <div class="item">
    <img src="phone.jpg" />
    <div class="name">新款手机</div>
    <span onclick="buy()">购买</span>
  </div>
</div>

<a href="https://partner.example.com" target="_blank">合作伙伴</a>
```

**问题**

1. 标题用 `div`，没有页面主标题
2. 打开筛选是动作，不应使用 `href="#"`
3. 图片缺少 alt 和尺寸
4. 购买用 `span`，不可聚焦、不可键盘触发
5. 列表没有列表语义
6. 外链缺少显式 `rel`

**参考实现**

```html
<header>
  <h1>商品列表</h1>
  <button type="button" id="open-filter">筛选</button>
</header>

<main>
  <ul class="product-list">
    <li>
      <article class="product-card">
        <img src="phone.jpg" width="320" height="320" alt="新款手机正面图" />
        <h2>新款手机</h2>
        <button type="button" data-product-id="phone-001">购买</button>
      </article>
    </li>
  </ul>
</main>

<a href="https://partner.example.com" target="_blank" rel="noopener noreferrer">
  合作伙伴（新窗口打开）
</a>
```

**易错点**

- 只把 `span` 改成 `div`
- 只补 ARIA，不改原生元素
- 新窗口打开没有给用户提示

---

### H10. 实现安全的 `renderToString`（简化 SSR）

- **难度**: senior
- **标签**: SSR, XSS, 递归
- **面试官看什么**: 递归渲染、文本和属性转义、空元素、危险属性和 URL 处理

**题干**

把如下虚拟节点渲染为 HTML 字符串：

```js
{
  tag: 'a',
  props: { href: '/user/1', title: '用户<1>' },
  children: ['查看 ', { tag: 'strong', children: ['张三'] }],
}
```

要求：文本和属性转义，支持布尔属性和空元素，拒绝事件属性和危险 URL 协议。

**思路口述**

1. 字符串和数字是文本节点，执行 HTML 转义
2. 元素节点校验标签名和属性名
3. `null`、`undefined`、`false` 属性跳过
4. `true` 输出布尔属性
5. 禁止 `onxxx` 属性
6. URL 属性只允许白名单协议
7. 空元素不输出闭合标签和子节点

**参考实现**

```js
const VOID_ELEMENTS = new Set([
  'area', 'base', 'br', 'col', 'embed', 'hr', 'img',
  'input', 'link', 'meta', 'source', 'track', 'wbr',
])
const URL_ATTRIBUTES = new Set(['href', 'src', 'action', 'formaction'])
const SAFE_URL_PROTOCOLS = new Set(['http:', 'https:', 'mailto:'])

function escapeHTML(value) {
  return String(value)
    .replaceAll('&', '&amp;')
    .replaceAll('<', '&lt;')
    .replaceAll('>', '&gt;')
    .replaceAll('"', '&quot;')
    .replaceAll("'", '&#39;')
}

function isSafeURL(value) {
  try {
    // 用 URL 解析而不是字符串前缀判断，可处理大小写和空白等变体。
    const url = new URL(String(value), 'https://example.invalid')
    return SAFE_URL_PROTOCOLS.has(url.protocol)
  } catch {
    return false
  }
}

function renderAttributes(props = {}) {
  return Object.entries(props)
    .filter(([, value]) => value != null && value !== false)
    .map(([name, value]) => {
      if (!/^[a-zA-Z_:][\w:.-]*$/.test(name) || /^on/i.test(name)) {
        throw new Error(`不允许的属性名: ${name}`)
      }
      if (URL_ATTRIBUTES.has(name.toLowerCase()) && !isSafeURL(value)) {
        throw new Error(`不允许的 URL: ${value}`)
      }
      return value === true ? ` ${name}` : ` ${name}="${escapeHTML(value)}"`
    })
    .join('')
}

/** 将简化虚拟节点渲染为转义后的 HTML 字符串。 */
function renderToString(node) {
  if (node == null || node === false) return ''
  if (typeof node === 'string' || typeof node === 'number') return escapeHTML(node)
  if (Array.isArray(node)) return node.map(renderToString).join('')

  const { tag, props, children = [] } = node
  if (!/^[a-zA-Z][\w-]*$/.test(tag)) {
    throw new Error(`不允许的标签名: ${tag}`)
  }

  const attributes = renderAttributes(props)
  if (VOID_ELEMENTS.has(tag.toLowerCase())) {
    return `<${tag}${attributes}>`
  }

  return `<${tag}${attributes}>${renderToString(children)}</${tag}>`
}
```

**易错点**

- 属性值没有加引号
- 只判断 `startsWith('javascript:')`
- 让调用方直接传入任意 HTML 字符串
- 忘记 `script`、`style` 等元素内部是不同解析上下文；生产 SSR 应交给成熟框架处理
