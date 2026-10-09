# 浏览器原理手写题

> 目标：通过手写代码加深对事件机制、事件循环、跨域、渲染原理、缓存、安全的理解。面试时先口述思路，再写代码，最后说明边界情况。

## 使用方式

- **事件与事件循环**：H1–H2，考察对事件流和任务调度的理解
- **跨域与通信**：H3–H4、H11，考察对同源策略和跨上下文通信的理解
- **渲染与解析**：H5–H7，考察对解析、选择器匹配和帧的理解
- **缓存与安全**：H8–H10、H12，考察对 HTTP 缓存、Service Worker、XSS、CSRF 的理解

---

### H1. 实现事件委托

- **难度**: 初级
- **标签**: 事件
- **面试官看什么**: 是否理解冒泡、`closest` 和边界处理

**题干**

实现 `delegate(container, eventType, selector, handler)`，在容器上监听事件，只有当事件目标匹配选择器时才调用处理函数，并返回取消监听的函数。

**思路口述**

利用事件冒泡，在容器上注册监听；事件触发时从 `event.target` 向上查找匹配选择器的元素，并且这个元素必须在容器内部。处理函数中的 `this` 和第二个参数是匹配到的元素。

**参考实现**

```js
/**
 * 在容器上委托监听匹配选择器的后代元素的事件。
 * @returns 取消监听的函数
 */
function delegate(container, eventType, selector, handler) {
  function listener(event) {
    // target 可能是文本节点（例如某些合成事件），先找到元素
    const target = event.target instanceof Element ? event.target : event.target.parentElement
    const matched = target?.closest(selector)
    // closest 可能找到容器外部的祖先，必须确认在容器内
    if (matched && container.contains(matched)) {
      handler.call(matched, event, matched)
    }
  }
  container.addEventListener(eventType, listener)
  return () => container.removeEventListener(eventType, listener)
}

const off = delegate(document.querySelector('#list'), 'click', 'li[data-id]', (event, li) => {
  console.log('点击了', li.dataset.id)
})
```

**易错点**

- 只判断 `event.target.matches(selector)`：点击 li 内部的 span 时无法匹配
- 没有检查匹配的元素是否在容器内部
- `focus`、`blur` 不冒泡，需要使用 `focusin`、`focusout` 或捕获阶段
- 忘记提供取消监听的方法

---

### H2. 事件循环输出题

- **难度**: 中级
- **标签**: 事件循环
- **面试官看什么**: 能否准确分析任务、微任务、async/await 的执行顺序

**题干**

写出以下代码的输出顺序，并说明原因。

```js
console.log('1')

setTimeout(() => {
  console.log('2')
  Promise.resolve().then(() => console.log('3'))
}, 0)

new Promise((resolve) => {
  console.log('4')
  resolve()
}).then(() => {
  console.log('5')
  setTimeout(() => console.log('6'), 0)
})

async function run() {
  console.log('7')
  await null
  console.log('8')
}
run()

requestAnimationFrame(() => console.log('9'))

Promise.resolve().then(() => console.log('10'))

console.log('11')
```

**思路口述**

先执行所有同步代码；然后清空微任务队列（按入队顺序）；然后执行下一个任务，每个任务结束后再清空微任务。rAF 在渲染步骤中执行，时机取决于下一帧，与定时器的先后顺序不确定。

**参考答案**

```text
1 4 7 11     同步代码
5 8 10       微任务（按入队顺序）
2 3          第一个 setTimeout，执行后清空其中的微任务
6            在微任务 5 中注册的 setTimeout
9            rAF：通常在下一帧，可能出现在 2 之前或之后的任意位置
```

**分析**：

- `new Promise` 的执行器同步执行，所以 4 在同步阶段输出
- `await null` 之后的代码相当于一个微任务，入队顺序在 5 之后、10 之前
- setTimeout 2 在同步阶段注册，setTimeout 6 在微任务阶段注册，所以 2 先于 6
- rAF 与 setTimeout 的顺序取决于下一次渲染时机，**不确定**

**易错点**

- 认为 rAF 一定在 setTimeout 之前或之后
- 忘记 Promise 执行器是同步的
- 认为任务中产生的微任务要等所有任务执行完才执行

---

### H3. 实现 JSONP

- **难度**: 初级
- **标签**: 跨域
- **面试官看什么**: 是否理解 JSONP 的原理、清理和超时处理

**题干**

实现 `jsonp(url, params, options)`，返回 Promise，支持超时和清理。

**思路口述**

动态创建 script 标签，URL 中带上回调函数名；服务端返回 `callbackName(data)`，浏览器执行时调用全局回调。完成后删除 script 和全局函数；超时则 reject。

**参考实现**

```js
let jsonpId = 0

/**
 * 通过 script 标签发起 JSONP 请求。
 * 仅用于兼容只支持 JSONP 的旧接口；JSONP 会执行服务端返回的任意脚本，存在安全风险。
 */
function jsonp(url, params = {}, { callbackKey = 'callback', timeout = 10000 } = {}) {
  return new Promise((resolve, reject) => {
    const callbackName = `__jsonp_${Date.now()}_${jsonpId++}`
    const query = new URLSearchParams({ ...params, [callbackKey]: callbackName })
    const script = document.createElement('script')
    script.src = `${url}${url.includes('?') ? '&' : '?'}${query}`

    const timer = setTimeout(() => {
      cleanup()
      reject(new Error('JSONP 请求超时'))
    }, timeout)

    function cleanup() {
      clearTimeout(timer)
      script.remove()
      // 超时后响应仍可能到达，保留一个空函数，避免调用不存在的全局函数报错
      window[callbackName] = () => delete window[callbackName]
    }

    window[callbackName] = (data) => {
      cleanup()
      delete window[callbackName]
      resolve(data)
    }

    script.onerror = () => {
      cleanup()
      reject(new Error('JSONP 请求失败'))
    }

    document.head.appendChild(script)
  })
}
```

**易错点**

- 没有清理 script 标签和全局函数，导致内存泄漏
- 超时后直接删除全局函数，响应迟到时报 `xxx is not a function`
- 回调名没有唯一化，并发请求互相覆盖
- 不了解安全风险：只支持 GET；服务端被攻破时可以执行任意代码

---

### H4. 实现跨标签页通信

- **难度**: 中级
- **标签**: 通信
- **面试官看什么**: API 选择、降级方案、消息不回环

**题干**

实现 `createTabChannel(name)`，提供 `post(message)`、`subscribe(handler)` 和 `close()`。优先使用 BroadcastChannel，不支持时降级到 localStorage 的 storage 事件。

**思路口述**

BroadcastChannel 原生支持同源广播。降级时，写入 localStorage 会触发其他标签页的 storage 事件；为了保证相同内容也能触发事件，消息中带上时间戳和随机 ID；写入后立即删除，避免残留数据。

**参考实现**

```js
/**
 * 创建同源标签页之间的广播通道。
 * 优先使用 BroadcastChannel，不支持时降级为 storage 事件。
 */
function createTabChannel(name) {
  const handlers = new Set()
  const emit = (message) => handlers.forEach((handler) => handler(message))

  if ('BroadcastChannel' in window) {
    const channel = new BroadcastChannel(name)
    channel.onmessage = (event) => emit(event.data)
    return {
      post: (message) => channel.postMessage(message),
      subscribe(handler) {
        handlers.add(handler)
        return () => handlers.delete(handler)
      },
      close: () => channel.close(),
    }
  }

  const storageKey = `__tab_channel_${name}`
  function onStorage(event) {
    // removeItem 也会触发 storage 事件，此时 newValue 为 null，需要忽略
    if (event.key !== storageKey || !event.newValue) return
    try {
      emit(JSON.parse(event.newValue).message)
    } catch {
      // 忽略其他代码写入的非法数据
    }
  }
  window.addEventListener('storage', onStorage)

  return {
    post(message) {
      // 加上随机 ID，保证连续发送相同的消息时值也发生变化，从而触发事件
      const payload = JSON.stringify({ id: Math.random(), time: Date.now(), message })
      localStorage.setItem(storageKey, payload)
      localStorage.removeItem(storageKey)
    },
    subscribe(handler) {
      handlers.add(handler)
      return () => handlers.delete(handler)
    },
    close: () => window.removeEventListener('storage', onStorage),
  }
}

const channel = createTabChannel('auth')
channel.subscribe((message) => {
  if (message.type === 'logout') location.href = '/login'
})
channel.post({ type: 'logout' })
```

**易错点**

- storage 事件不会在写入的标签页自身触发，BroadcastChannel 也不会发给自身，两种方式行为一致
- 没有处理 removeItem 触发的空值事件
- 消息内容必须可以序列化（BroadcastChannel 使用结构化克隆，storage 只能是字符串）

---

### H5. 实现简易 HTML 解析器

- **难度**: 高级
- **标签**: 解析
- **面试官看什么**: 是否理解词法分析（tokenizer）和用栈构建树

**题干**

实现 `parseHTML(html)`，支持开始标签（带属性）、结束标签、自闭合标签、文本，返回 DOM 树结构。不需要处理注释、脚本、容错。

**思路口述**

分两步：先用状态机或正则把字符串切分为 token（开始标签、结束标签、文本）；然后用栈构建树：遇到开始标签就创建节点，加入栈顶节点的子节点并入栈；遇到结束标签就出栈；文本节点加入栈顶的子节点。void 元素（img、br 等）不入栈。

**参考实现**

```js
const VOID_ELEMENTS = new Set(['img', 'br', 'hr', 'input', 'meta', 'link'])

/** 把 HTML 字符串切分为 token 序列。 */
function tokenize(html) {
  const tokens = []
  const tagPattern = /<\/?([a-zA-Z][\w-]*)([^>]*?)(\/?)>/g
  let lastIndex = 0
  let match

  while ((match = tagPattern.exec(html))) {
    const text = html.slice(lastIndex, match.index)
    if (text.trim()) tokens.push({ type: 'text', content: text.trim() })

    const [raw, tagName, attrString, selfClosing] = match
    if (raw.startsWith('</')) {
      tokens.push({ type: 'endTag', tagName: tagName.toLowerCase() })
    } else {
      tokens.push({
        type: 'startTag',
        tagName: tagName.toLowerCase(),
        attributes: parseAttributes(attrString),
        selfClosing: Boolean(selfClosing) || VOID_ELEMENTS.has(tagName.toLowerCase()),
      })
    }
    lastIndex = tagPattern.lastIndex
  }

  const rest = html.slice(lastIndex)
  if (rest.trim()) tokens.push({ type: 'text', content: rest.trim() })
  return tokens
}

function parseAttributes(attrString) {
  const attributes = {}
  const attrPattern = /([\w-]+)(?:\s*=\s*(?:"([^"]*)"|'([^']*)'|([^\s"']+)))?/g
  let match
  while ((match = attrPattern.exec(attrString))) {
    const [, name, doubleQuoted, singleQuoted, unquoted] = match
    // 没有值的布尔属性（如 disabled）记为空字符串，与 DOM 行为一致
    attributes[name] = doubleQuoted ?? singleQuoted ?? unquoted ?? ''
  }
  return attributes
}

/** 解析 HTML 字符串，返回以 document 为根的节点树。 */
function parseHTML(html) {
  const root = { type: 'document', children: [] }
  const stack = [root]

  for (const token of tokenize(html)) {
    const parent = stack[stack.length - 1]
    if (token.type === 'text') {
      parent.children.push({ type: 'text', content: token.content })
    } else if (token.type === 'startTag') {
      const element = {
        type: 'element',
        tagName: token.tagName,
        attributes: token.attributes,
        children: [],
      }
      parent.children.push(element)
      if (!token.selfClosing) stack.push(element)
    } else {
      // 结束标签：向下查找匹配的开始标签，中间未闭合的元素一并关闭（简化的容错）
      const index = stack.findLastIndex((node) => node.tagName === token.tagName)
      if (index > 0) stack.length = index
    }
  }
  return root
}

console.log(JSON.stringify(parseHTML('<div id="app"><p class="a">Hi<br/>there</p><img src="x.png"></div>'), null, 2))
```

**易错点**

- 没有处理 void 元素，导致后续元素被错误地嵌套
- 遇到不匹配的结束标签时直接出栈，打乱结构
- 属性值中包含 `>` 时正则会出错（真实的解析器使用状态机）
- 可以补充说明：真实的 HTML 解析器是规范定义的状态机，有大量的容错规则（例如自动补全 `</p>`）

---

### H6. 实现从右向左的选择器匹配

- **难度**: 高级
- **标签**: CSS, 选择器
- **面试官看什么**: 是否理解浏览器为什么从右向左匹配

**题干**

实现 `matches(element, selector)`，支持后代选择器（空格）和子选择器（`>`），每个简单选择器支持标签、`.class`、`#id` 的组合（例如 `div.nav > ul li.active`）。

**思路口述**

把选择器解析为复合选择器数组和组合符；从最右边的复合选择器开始匹配当前元素；然后按组合符向上找：子选择器只检查父元素；后代选择器向上查找任意一个匹配的祖先。后代选择器匹配失败时需要回溯，尝试更上层的祖先。

**参考实现**

```js
/** 解析复合选择器，例如 'li.active#first' → { tag, id, classes }。 */
function parseCompound(compound) {
  const result = { tag: null, id: null, classes: [] }
  for (const part of compound.match(/[#.]?[\w-]+/g) ?? []) {
    if (part.startsWith('#')) result.id = part.slice(1)
    else if (part.startsWith('.')) result.classes.push(part.slice(1))
    else result.tag = part.toLowerCase()
  }
  return result
}

function matchesCompound(element, compound) {
  if (compound.tag && element.tagName.toLowerCase() !== compound.tag) return false
  if (compound.id && element.id !== compound.id) return false
  return compound.classes.every((name) => element.classList.contains(name))
}

/** 把选择器拆分为 [复合选择器, 组合符, 复合选择器, ...]，组合符为 '>' 或 ' '。 */
function parseSelector(selector) {
  const tokens = selector.trim().replace(/\s*>\s*/g, ' > ').split(/\s+/)
  const parts = []
  for (const token of tokens) {
    if (token === '>') {
      parts.push('>')
      continue
    }
    // 两个复合选择器相邻时，中间是被空白吞掉的后代组合符
    if (parts.length && parts[parts.length - 1] !== '>') parts.push(' ')
    parts.push(parseCompound(token))
  }
  return parts
}

/** 从右向左判断元素是否匹配选择器。 */
function matches(element, selector) {
  const parts = parseSelector(selector)

  function matchFrom(node, index) {
    if (!matchesCompound(node, parts[index])) return false
    if (index === 0) return true

    const combinator = parts[index - 1]
    const nextIndex = index - 2
    if (combinator === '>') {
      return node.parentElement !== null && matchFrom(node.parentElement, nextIndex)
    }
    // 后代选择器：任意祖先匹配即可，失败时继续尝试更上层的祖先（回溯）
    for (let ancestor = node.parentElement; ancestor; ancestor = ancestor.parentElement) {
      if (matchFrom(ancestor, nextIndex)) return true
    }
    return false
  }

  return matchFrom(element, parts.length - 1)
}

// matches(document.querySelector('li.active'), 'div.nav > ul li.active')
```

**易错点**

- 后代选择器匹配失败时没有回溯
- 子选择器向上查找了多层祖先
- 能解释为什么从右向左：最右边的选择器可以快速排除大部分元素，而从左向右需要遍历大量的后代

---

### H7. 实现 FPS 监控

- **难度**: 中级
- **标签**: 渲染, 监控
- **面试官看什么**: 是否理解 rAF 与帧的关系、长帧检测

**题干**

实现 `createFpsMonitor(onReport)`，每秒报告一次 FPS，同时报告该秒内超过 50ms 的长帧数量。支持停止。

**思路口述**

rAF 回调在每帧渲染前执行一次，每秒统计回调次数就是 FPS。相邻两次回调的时间差超过阈值就是长帧。页面隐藏时 rAF 会暂停，需要重置统计，避免把隐藏时间计为一个超长帧。

**参考实现**

```js
/**
 * 基于 requestAnimationFrame 监控页面帧率。
 * @returns 停止监控的函数
 */
function createFpsMonitor(onReport, { longFrameThreshold = 50 } = {}) {
  let rafId = 0
  let frames = 0
  let longFrames = 0
  let windowStart = performance.now()
  let lastFrameTime = windowStart

  function reset() {
    frames = 0
    longFrames = 0
    windowStart = performance.now()
    lastFrameTime = windowStart
  }

  function tick(now) {
    frames++
    if (now - lastFrameTime > longFrameThreshold) longFrames++
    lastFrameTime = now

    const elapsed = now - windowStart
    if (elapsed >= 1000) {
      onReport({ fps: Math.round((frames * 1000) / elapsed), longFrames })
      frames = 0
      longFrames = 0
      windowStart = now
    }
    rafId = requestAnimationFrame(tick)
  }

  // 页面隐藏时 rAF 暂停，恢复后的第一帧间隔会非常大，需要重置
  function onVisibilityChange() {
    if (document.visibilityState === 'visible') reset()
  }

  document.addEventListener('visibilitychange', onVisibilityChange)
  rafId = requestAnimationFrame(tick)

  return () => {
    cancelAnimationFrame(rafId)
    document.removeEventListener('visibilitychange', onVisibilityChange)
  }
}

const stop = createFpsMonitor(({ fps, longFrames }) => console.log(`FPS ${fps}，长帧 ${longFrames}`))
```

**易错点**

- 用 setInterval 计算 FPS：定时器与帧无关
- 没有处理页面隐藏
- 长期运行的 rAF 本身也有开销，生产环境应采样运行，或改用 Long Animation Frames API
- 高刷新率屏幕上 FPS 可能是 120，不应该写死 60

---

### H8. 实现 Service Worker 的缓存策略

- **难度**: 中级
- **标签**: Service Worker, 缓存
- **面试官看什么**: 是否掌握常见的缓存策略和版本清理

**题干**

在 Service Worker 中实现：安装时预缓存核心资源；激活时清理旧版本缓存；HTML 使用网络优先，带 hash 的静态资源使用缓存优先，API 使用 stale-while-revalidate。

**思路口述**

install 事件中预缓存；activate 中删除不是当前版本的缓存；fetch 事件中根据请求类型选择策略。网络优先适合需要最新内容的 HTML，离线时回退到缓存；缓存优先适合不会变化的带 hash 资源；SWR 先返回缓存，后台更新。

**参考实现**

```js
// sw.js
const VERSION = 'v3'
const STATIC_CACHE = `static-${VERSION}`
const RUNTIME_CACHE = `runtime-${VERSION}`
const PRECACHE_URLS = ['/', '/offline.html', '/assets/app.3f2a1c.js', '/assets/app.9b8e7d.css']

self.addEventListener('install', (event) => {
  event.waitUntil(caches.open(STATIC_CACHE).then((cache) => cache.addAll(PRECACHE_URLS)))
})

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(
        keys
          .filter((key) => key !== STATIC_CACHE && key !== RUNTIME_CACHE)
          .map((key) => caches.delete(key)),
      ),
    ),
  )
})

async function networkFirst(request) {
  const cache = await caches.open(RUNTIME_CACHE)
  try {
    const response = await fetch(request)
    if (response.ok) cache.put(request, response.clone())
    return response
  } catch {
    return (await cache.match(request)) ?? (await caches.match('/offline.html'))
  }
}

async function cacheFirst(request) {
  const cached = await caches.match(request)
  if (cached) return cached
  const response = await fetch(request)
  if (response.ok) {
    const cache = await caches.open(STATIC_CACHE)
    cache.put(request, response.clone())
  }
  return response
}

async function staleWhileRevalidate(event) {
  const cache = await caches.open(RUNTIME_CACHE)
  const cached = await cache.match(event.request)
  const networkPromise = fetch(event.request).then((response) => {
    if (response.ok) cache.put(event.request, response.clone())
    return response
  })
  // 有缓存时立即返回，但 SW 可能在后台更新完成前被终止，需要 waitUntil 延长生命周期
  event.waitUntil(networkPromise.catch(() => {}))
  return cached ?? networkPromise
}

self.addEventListener('fetch', (event) => {
  const { request } = event
  // 只处理同源的 GET 请求，POST 等请求无法缓存
  if (request.method !== 'GET' || new URL(request.url).origin !== self.location.origin) return

  if (request.mode === 'navigate') {
    event.respondWith(networkFirst(request))
  } else if (request.url.includes('/assets/')) {
    event.respondWith(cacheFirst(request))
  } else if (request.url.includes('/api/')) {
    event.respondWith(staleWhileRevalidate(event))
  }
})
```

**易错点**

- 缓存响应时没有 `clone()`：响应体只能读取一次
- 没有清理旧版本缓存，存储不断增长
- 缓存了错误响应（4xx / 5xx）或 POST 请求
- SWR 没有使用 waitUntil，后台更新可能被中断
- 涉及用户数据的 API 缓存要注意退出登录时清理

---

### H9. 实现 XSS 转义和 Trusted Types 策略

- **难度**: 中级
- **标签**: 安全, XSS
- **面试官看什么**: 是否知道不同上下文的转义方式和纵深防御

**题干**

1. 实现 `escapeHTML`，用于把用户输入插入到 HTML 内容和属性中
2. 实现 `sanitizeURL`，只允许安全的协议
3. 创建一个 Trusted Types 策略，让 `innerHTML` 只能接受经过净化的内容

**思路口述**

HTML 内容和带引号的属性中，转义 `& < > " '` 五个字符即可。URL 需要解析后检查协议，防止 `javascript:`。Trusted Types 在 CSP 开启后，危险 API 只接受策略生成的类型；策略中使用 DOMPurify 净化。

**参考实现**

```js
const HTML_ESCAPE_MAP = { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }

/** 转义用于 HTML 内容或带引号属性值的字符串。 */
function escapeHTML(value) {
  return String(value).replace(/[&<>"']/g, (char) => HTML_ESCAPE_MAP[char])
}

const SAFE_PROTOCOLS = new Set(['http:', 'https:', 'mailto:'])

/** 只允许安全协议的 URL，其他情况返回安全的占位值。 */
function sanitizeURL(url) {
  try {
    // 用 URL 解析，避免 'JaVaScRiPt:'、前导空白、编码等绕过字符串前缀判断
    const parsed = new URL(url, location.origin)
    return SAFE_PROTOCOLS.has(parsed.protocol) ? parsed.href : 'about:blank'
  } catch {
    return 'about:blank'
  }
}

// CSP 头：Content-Security-Policy: require-trusted-types-for 'script'; trusted-types app-html
const appPolicy = window.trustedTypes?.createPolicy('app-html', {
  // DOMPurify 需要单独引入；RETURN_TRUSTED_TYPE 为 false，交给策略包装
  createHTML: (input) => DOMPurify.sanitize(input),
})

/** 把富文本内容安全地设置到元素上。 */
function setRichText(element, html) {
  element.innerHTML = appPolicy ? appPolicy.createHTML(html) : DOMPurify.sanitize(html)
}

const name = '<img src=x onerror=alert(1)>'
document.querySelector('#greeting').innerHTML = `你好，${escapeHTML(name)}`
document.querySelector('#link').href = sanitizeURL('javascript:alert(1)')
```

**易错点**

- 先转义 `<` 再转义 `&`，导致双重转义（使用单次替换可以避免）
- 用 `url.startsWith('javascript:')` 判断，被大小写、空白绕过
- 在 `<script>` 内、事件属性中、未加引号的属性中插入数据，HTML 转义不够
- 能说明：首选 `textContent`，根本不经过 HTML 解析

---

### H10. 实现 CSRF Token 中间件（Node）

- **难度**: 中级
- **标签**: 安全, CSRF, Node
- **面试官看什么**: Token 的生成与校验、安全比较、请求方法的区分

**题干**

用 Node 原生 `http` 实现双重提交 Cookie 方案：GET 请求时下发 CSRF Token Cookie；POST、PUT、DELETE 等请求必须在 `X-CSRF-Token` 头中带上相同的 Token，同时校验 `Origin`。

**思路口述**

Token 用加密安全的随机数生成，放在非 HttpOnly 的 Cookie 中（前端需要读取）；状态修改请求时比较请求头和 Cookie 中的值；攻击者能让浏览器带上 Cookie，但读不到 Cookie 的值，所以无法设置请求头。比较时使用定时安全的比较函数。再加一层 Origin 校验。

**参考实现**

```js
import { createServer } from 'node:http'
import { randomBytes, timingSafeEqual } from 'node:crypto'

const SAFE_METHODS = new Set(['GET', 'HEAD', 'OPTIONS'])
const ALLOWED_ORIGINS = new Set(['https://app.example.com'])
const TOKEN_COOKIE = 'csrf_token'

function parseCookies(header = '') {
  return Object.fromEntries(
    header.split(';').filter(Boolean).map((pair) => {
      const [key, ...rest] = pair.trim().split('=')
      return [key, decodeURIComponent(rest.join('='))]
    }),
  )
}

function safeEqual(a, b) {
  const bufferA = Buffer.from(a)
  const bufferB = Buffer.from(b)
  // timingSafeEqual 要求长度相同，长度不同直接判定不相等
  return bufferA.length === bufferB.length && timingSafeEqual(bufferA, bufferB)
}

/**
 * CSRF 防护：双重提交 Cookie + Origin 校验。
 * @returns 是否允许继续处理请求
 */
function csrfProtection(req, res) {
  const cookies = parseCookies(req.headers.cookie)

  if (SAFE_METHODS.has(req.method)) {
    if (!cookies[TOKEN_COOKIE]) {
      const token = randomBytes(32).toString('hex')
      // 不能设置 HttpOnly：前端需要读取 Cookie 放到请求头中
      res.setHeader('Set-Cookie', `${TOKEN_COOKIE}=${token}; Path=/; Secure; SameSite=Lax`)
    }
    return true
  }

  const origin = req.headers.origin
  if (origin && !ALLOWED_ORIGINS.has(origin)) return false

  const headerToken = req.headers['x-csrf-token']
  const cookieToken = cookies[TOKEN_COOKIE]
  return Boolean(headerToken && cookieToken && safeEqual(headerToken, cookieToken))
}

createServer((req, res) => {
  if (!csrfProtection(req, res)) {
    res.writeHead(403).end('CSRF 校验失败')
    return
  }
  res.writeHead(200, { 'Content-Type': 'application/json' }).end('{"ok":true}')
}).listen(3000)
```

前端：

```js
const token = document.cookie.match(/(?:^|;\s*)csrf_token=([^;]+)/)?.[1]
await fetch('/api/transfer', {
  method: 'POST',
  headers: { 'X-CSRF-Token': token, 'Content-Type': 'application/json' },
  body: JSON.stringify({ amount: 100 }),
})
```

**易错点**

- GET 请求有副作用时，CSRF 防护完全失效
- 使用 `===` 比较 Token，存在时序攻击风险（理论上）
- Token 放在 URL 中，会泄露到日志和 Referer
- 双重提交的弱点：子域可以写入父域的 Cookie；更安全的做法是用 HMAC 签名把 Token 与会话绑定

---

### H11. 实现 CORS 中间件（Node）

- **难度**: 中级
- **标签**: 跨域, Node
- **面试官看什么**: 预检请求处理、凭证与 Origin 白名单、Vary

**题干**

用 Node 原生 `http` 实现 CORS 中间件：支持 Origin 白名单、带凭证的请求、预检请求（含缓存时间），并正确设置 `Vary`。

**思路口述**

检查请求的 Origin 是否在白名单中，在则回显该 Origin（带凭证时不能用 `*`）。预检请求（OPTIONS 且带 `Access-Control-Request-Method`）直接返回 204，附带允许的方法、头部和缓存时间。因为响应随 Origin 变化，要设置 `Vary: Origin`。

**参考实现**

```js
import { createServer } from 'node:http'

const corsOptions = {
  origins: new Set(['https://app.example.com', 'https://admin.example.com']),
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-CSRF-Token'],
  exposedHeaders: ['X-Request-Id'],
  credentials: true,
  maxAge: 600,
}

/**
 * 处理 CORS 响应头。
 * @returns 是否已经结束响应（预检请求）
 */
function cors(req, res, options = corsOptions) {
  const origin = req.headers.origin
  // 响应内容随 Origin 变化，必须告诉缓存（CDN）按 Origin 区分
  res.setHeader('Vary', 'Origin')

  if (!origin || !options.origins.has(origin)) {
    // 不设置 CORS 头即可，浏览器会拦截响应；预检请求直接结束
    if (req.method === 'OPTIONS') {
      res.writeHead(204).end()
      return true
    }
    return false
  }

  res.setHeader('Access-Control-Allow-Origin', origin)
  if (options.credentials) res.setHeader('Access-Control-Allow-Credentials', 'true')
  if (options.exposedHeaders.length) {
    res.setHeader('Access-Control-Expose-Headers', options.exposedHeaders.join(', '))
  }

  const isPreflight = req.method === 'OPTIONS' && req.headers['access-control-request-method']
  if (isPreflight) {
    res.setHeader('Access-Control-Allow-Methods', options.methods.join(', '))
    res.setHeader('Access-Control-Allow-Headers', options.allowedHeaders.join(', '))
    res.setHeader('Access-Control-Max-Age', String(options.maxAge))
    res.writeHead(204).end()
    return true
  }
  return false
}

createServer((req, res) => {
  if (cors(req, res)) return
  res.writeHead(200, { 'Content-Type': 'application/json', 'X-Request-Id': '123' })
  res.end('{"ok":true}')
}).listen(3000)
```

**易错点**

- 带凭证时使用 `Access-Control-Allow-Origin: *`，浏览器会拒绝
- 直接回显任意 Origin 且允许凭证：相当于关闭了同源保护
- 没有设置 `Vary: Origin`，CDN 把 A 源的响应缓存后返回给 B 源
- 预检请求走到了业务逻辑（例如鉴权中间件拒绝了没有 Token 的 OPTIONS 请求）
- 能说明：CORS 只是浏览器的读取限制，服务端仍需要做鉴权

---

### H12. 实现 HTTP 缓存协商（ETag / 304）

- **难度**: 中级
- **标签**: 缓存, HTTP, Node
- **面试官看什么**: ETag 生成、条件请求判断、缓存头组合

**题干**

用 Node 原生 `http` 实现一个静态文件服务：带 hash 的文件设置长期强缓存；HTML 设置 `no-cache`；支持 ETag 和 Last-Modified 的条件请求，未修改时返回 304。

**思路口述**

根据文件名判断缓存策略。ETag 用文件大小和修改时间生成（弱 ETag，成本低），或用内容 hash 生成（强 ETag）。请求带 `If-None-Match` 时优先比较 ETag；否则比较 `If-Modified-Since`。匹配时返回 304 且不带响应体。

**参考实现**

```js
import { createServer } from 'node:http'
import { createReadStream } from 'node:fs'
import { stat } from 'node:fs/promises'
import { extname, join, normalize } from 'node:path'

const ROOT = join(process.cwd(), 'dist')
const MIME_TYPES = { '.html': 'text/html; charset=utf-8', '.js': 'text/javascript', '.css': 'text/css' }
const HASHED_FILE = /\.[0-9a-f]{8,}\.(js|css|png|jpg|webp|woff2)$/

function getCacheControl(filePath) {
  if (HASHED_FILE.test(filePath)) return 'public, max-age=31536000, immutable'
  // HTML 每次都要验证，保证用户能拿到最新的资源引用
  return 'no-cache'
}

function isNotModified(req, etag, lastModified) {
  const ifNoneMatch = req.headers['if-none-match']
  // 同时存在时以 If-None-Match 为准，忽略 If-Modified-Since（RFC 9110）
  if (ifNoneMatch) {
    return ifNoneMatch.split(',').some((tag) => tag.trim() === etag || tag.trim() === '*')
  }
  const ifModifiedSince = req.headers['if-modified-since']
  if (ifModifiedSince) {
    // HTTP 日期只精确到秒，比较前去掉毫秒
    return Math.floor(lastModified.getTime() / 1000) <= Math.floor(Date.parse(ifModifiedSince) / 1000)
  }
  return false
}

createServer(async (req, res) => {
  const pathname = new URL(req.url, 'http://localhost').pathname
  // normalize 后检查前缀，防止 ../ 路径穿越读取 dist 外的文件
  const filePath = normalize(join(ROOT, pathname === '/' ? 'index.html' : pathname))
  if (!filePath.startsWith(ROOT)) {
    res.writeHead(403).end()
    return
  }

  let fileStat
  try {
    fileStat = await stat(filePath)
  } catch {
    res.writeHead(404).end('Not Found')
    return
  }

  const etag = `W/"${fileStat.size.toString(16)}-${fileStat.mtimeMs.toString(16)}"`
  const lastModified = fileStat.mtime
  const headers = {
    ETag: etag,
    'Last-Modified': lastModified.toUTCString(),
    'Cache-Control': getCacheControl(filePath),
  }

  if (isNotModified(req, etag, lastModified)) {
    res.writeHead(304, headers).end()
    return
  }

  res.writeHead(200, {
    ...headers,
    'Content-Type': MIME_TYPES[extname(filePath)] ?? 'application/octet-stream',
    'Content-Length': fileStat.size,
  })
  createReadStream(filePath).pipe(res)
}).listen(3000)
```

**易错点**

- 304 响应带了响应体
- `If-None-Match` 和 `If-Modified-Since` 同时存在时，没有以 ETag 为准
- Last-Modified 比较时没有处理秒级精度
- HTML 设置了长期缓存，导致用户拿到旧的 HTML
- 路径穿越漏洞：没有校验规范化后的路径
- 多台服务器部署时，基于 mtime 的 ETag 可能不一致，应使用内容 hash
