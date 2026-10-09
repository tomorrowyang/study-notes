# React 手写题（12 道高频）

> 建议：先说清「状态放哪、副作用如何清理、引用是否稳定」，再写代码。示例默认 React 19；用到 19.2+ API 时会注明回退写法。

---

### H1. 模拟实现 useState 和 useEffect

- **难度**: senior
- **标签**: Hooks 原理
- **面试官看什么**: 是否理解 Hook 按顺序存储、批量更新、effect 依赖比较和清理

**题干**

不使用 React，实现一个最简运行时：支持单个函数组件中的 `useState`、`useEffect`，状态更新后重新渲染。

**思路口述**

用数组模拟 Fiber 上的 Hook 链表，`cursor` 表示当前调用到第几个 Hook，每次渲染前重置为 0。setState 修改对应槽位并用微任务合并渲染。useEffect 比较依赖，变化时把「先清理旧 effect、再执行新 effect」的任务放到渲染后执行。

**参考实现**

```js
let hooks = []
let cursor = 0
let rootComponent = null
let pendingEffects = []
let renderScheduled = false

/**
 * 简化版 useState：按调用顺序在 hooks 数组中存取状态。
 */
function useState(initialValue) {
  const index = cursor++

  if (index >= hooks.length) {
    hooks[index] = typeof initialValue === 'function' ? initialValue() : initialValue
  }

  const setState = (action) => {
    const next = typeof action === 'function' ? action(hooks[index]) : action
    if (Object.is(next, hooks[index])) return
    hooks[index] = next
    scheduleRender()
  }

  return [hooks[index], setState]
}

/**
 * 简化版 useEffect：依赖变化时在渲染后执行，并先调用上一次的清理函数。
 */
function useEffect(effect, deps) {
  const index = cursor++
  const prev = hooks[index]
  const changed =
    !prev ||
    !deps ||
    deps.length !== prev.deps.length ||
    deps.some((dep, i) => !Object.is(dep, prev.deps[i]))

  if (!changed) return

  const record = { deps, cleanup: undefined }
  hooks[index] = record
  pendingEffects.push(() => {
    prev?.cleanup?.()
    record.cleanup = effect()
  })
}

function scheduleRender() {
  // 同一轮内多次 setState 只渲染一次，模拟批处理
  if (renderScheduled) return
  renderScheduled = true
  queueMicrotask(() => {
    renderScheduled = false
    render()
  })
}

function render() {
  cursor = 0
  const output = rootComponent()
  const effects = pendingEffects
  pendingEffects = []
  effects.forEach((run) => run())
  return output
}

/**
 * 挂载根组件并完成首次渲染。
 */
function mount(Component) {
  rootComponent = Component
  return render()
}
```

**易错点**

- 忘记每次渲染重置 cursor
- setState 后同步渲染，导致多次更新多次渲染
- effect 依赖变化时没有先执行旧清理函数
- 真实 React 的状态存在每个 Fiber 上，并支持多组件、优先级和更新队列，这里只是单组件模型

---

### H2. 实现 useDebouncedValue 和 useDebouncedCallback

- **难度**: mid
- **标签**: 自定义 Hook, 防抖
- **面试官看什么**: 定时器清理、最新回调、引用稳定

**题干**

1. `useDebouncedValue(value, delay)`：返回延迟更新的值
2. `useDebouncedCallback(fn, delay)`：返回稳定的防抖函数，调用时使用最新的 fn

**思路口述**

值防抖：value 变化时启动定时器，下一次变化或卸载时清除。回调防抖：用 ref 保存最新回调，防抖函数本身只依赖 delay 保持引用稳定；卸载时清理定时器。

**参考实现**

```jsx
/**
 * 返回 value 在停止变化 delay 毫秒后的值。
 */
function useDebouncedValue(value, delay) {
  const [debounced, setDebounced] = useState(value)

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay)
    return () => clearTimeout(timer)
  }, [value, delay])

  return debounced
}

/**
 * 返回引用稳定的防抖回调，执行时总是调用最新的 callback。
 */
function useDebouncedCallback(callback, delay) {
  const callbackRef = useRef(callback)
  const timerRef = useRef(null)

  useLayoutEffect(() => {
    callbackRef.current = callback
  })

  useEffect(() => {
    return () => clearTimeout(timerRef.current)
  }, [])

  return useCallback(
    (...args) => {
      clearTimeout(timerRef.current)
      timerRef.current = setTimeout(() => callbackRef.current(...args), delay)
    },
    [delay],
  )
}
```

**易错点**

- 依赖 callback 导致每次渲染都生成新的防抖函数，防抖失效
- 卸载后定时器仍触发
- 在渲染期间写 `callbackRef.current = callback`，违反渲染纯度
- 搜索场景还要处理请求竞态

---

### H3. 实现 useFetch（支持取消和竞态处理）

- **难度**: mid
- **标签**: 数据请求, AbortController
- **面试官看什么**: 状态建模、竞态、错误处理、清理

**题干**

实现 `useFetch(url)`，返回 `{ status, data, error }`，url 变化时取消旧请求。

**思路口述**

用一个 status 字段代替多个布尔值，避免矛盾状态。effect 中创建 AbortController，清理函数中 abort，这样 url 变化或卸载时旧请求不会覆盖新结果。fetch 对 4xx/5xx 不 reject，需要手动检查 `res.ok`。

**参考实现**

```jsx
/**
 * 请求 url 对应的 JSON 数据，url 变化或卸载时取消旧请求。
 */
function useFetch(url) {
  const [state, setState] = useState({ status: 'idle', data: null, error: null })

  useEffect(() => {
    if (!url) return

    const controller = new AbortController()
    setState((prev) => ({ ...prev, status: 'loading', error: null }))

    fetch(url, { signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`)
        return res.json()
      })
      .then((data) => setState({ status: 'success', data, error: null }))
      .catch((error) => {
        if (error.name === 'AbortError') return
        setState({ status: 'error', data: null, error })
      })

    return () => controller.abort()
  }, [url])

  return state
}
```

**易错点**

- 没有取消导致旧响应覆盖新数据
- 用 `isLoading`、`isError`、`data` 多个状态，出现不可能组合
- 把 AbortError 当作业务错误上报
- 生产中缓存、去重、重试应交给 TanStack Query / SWR

---

### H4. 实现 usePrevious 和 useLatest

- **难度**: mid
- **标签**: 自定义 Hook, ref
- **面试官看什么**: 渲染纯度、ref 读写时机

**题干**

- `usePrevious(value)`：返回上一次不同的值
- `useLatest(value)`：返回一个 ref，`ref.current` 总是最新值，用于事件和定时器回调

**思路口述**

React 规则不建议在渲染中读写 ref，所以 usePrevious 用 state 保存「上次值」并在渲染中检测变化（官方推荐的保存上一次渲染信息的模式）。useLatest 在 layout effect 中更新 ref，保证事件回调执行时读到已提交的最新值。

**参考实现**

```jsx
/**
 * 返回 value 上一次变化前的值。
 */
function usePrevious(value) {
  const [current, setCurrent] = useState(value)
  const [previous, setPrevious] = useState(undefined)

  // 渲染期间根据 props 调整 state，React 会立即用新 state 重新渲染本组件
  if (!Object.is(value, current)) {
    setPrevious(current)
    setCurrent(value)
  }

  return previous
}

/**
 * 返回始终指向最新 value 的 ref，供事件和异步回调读取。
 */
function useLatest(value) {
  const ref = useRef(value)

  useLayoutEffect(() => {
    ref.current = value
  })

  return ref
}
```

**易错点**

- ref 版 usePrevious 在渲染中读 `ref.current`，与 React Compiler 规则冲突
- useLatest 的 ref 不能在渲染中读取用于 UI
- React 19.2+ 中，effect 场景优先用 useEffectEvent 替代 useLatest

---

### H5. 实现 useInterval

- **难度**: mid
- **标签**: 定时器, 闭包
- **面试官看什么**: stale closure、delay 变化、暂停

**题干**

实现 `useInterval(callback, delay)`，callback 始终使用最新状态，delay 为 null 时暂停。

**思路口述**

直接在 effect 里 setInterval(callback) 会捕获旧闭包；若依赖 callback 又会每次渲染重建定时器。所以用 ref 保存最新回调，定时器只依赖 delay。

**参考实现**

```jsx
/**
 * 声明式 setInterval：delay 为 null 时暂停，回调总是最新的。
 */
function useInterval(callback, delay) {
  const savedCallback = useRef(callback)

  useLayoutEffect(() => {
    savedCallback.current = callback
  }, [callback])

  useEffect(() => {
    if (delay === null) return

    const timer = setInterval(() => savedCallback.current(), delay)
    return () => clearInterval(timer)
  }, [delay])
}
```

React 19.2+ 写法：

```jsx
function useInterval(callback, delay) {
  const onTick = useEffectEvent(callback)

  useEffect(() => {
    if (delay === null) return
    const timer = setInterval(() => onTick(), delay)
    return () => clearInterval(timer)
  }, [delay])
}
```

**易错点**

- 依赖 callback 导致定时器不断重建，间隔不准
- 不依赖 callback 又不用 ref，读取旧状态
- 计数场景也可以直接用函数式更新 `setCount(c => c + 1)`

---

### H6. 实现跨标签页同步的 useLocalStorage

- **难度**: senior
- **标签**: useSyncExternalStore, 存储
- **面试官看什么**: 外部数据源接入、SSR 快照、同页和跨页同步、异常处理

**题干**

实现 `useLocalStorage(key, initialValue)`，返回 `[value, setValue]`：

- 同一页面多个组件使用同一 key 时同步
- 其他标签页修改时同步
- SSR 时不报错

**思路口述**

localStorage 是外部数据源，用 useSyncExternalStore 接入。快照返回原始字符串，字符串按值比较天然稳定，避免每次 JSON.parse 生成新对象导致无限渲染。跨标签页通过 storage 事件；同页面 storage 事件不触发，需要自己通知订阅者。

**参考实现**

```jsx
const localListeners = new Set()

function subscribe(callback) {
  localListeners.add(callback)
  window.addEventListener('storage', callback)
  return () => {
    localListeners.delete(callback)
    window.removeEventListener('storage', callback)
  }
}

function parseStored(raw, fallback) {
  if (raw === null) return fallback
  try {
    return JSON.parse(raw)
  } catch {
    return fallback
  }
}

/**
 * 与 localStorage 双向同步的状态 Hook，支持跨标签页同步。
 */
function useLocalStorage(key, initialValue) {
  const raw = useSyncExternalStore(
    subscribe,
    () => localStorage.getItem(key),
    () => null,
  )

  const value = useMemo(() => parseStored(raw, initialValue), [raw, initialValue])

  const setValue = useCallback(
    (next) => {
      const current = parseStored(localStorage.getItem(key), initialValue)
      const resolved = typeof next === 'function' ? next(current) : next

      try {
        localStorage.setItem(key, JSON.stringify(resolved))
      } catch (error) {
        // 隐私模式或超出配额时 setItem 会抛错，不能让整个组件崩溃
        console.warn('localStorage write failed', error)
        return
      }

      // 同一页面不会触发 storage 事件，需要手动通知
      localListeners.forEach((listener) => listener())
    },
    [key, initialValue],
  )

  return [value, setValue]
}
```

**易错点**

- getSnapshot 返回 `JSON.parse` 结果，每次都是新对象，导致无限渲染
- 服务端访问 localStorage 报错
- initialValue 传字面量对象会让 useMemo 每次重算，调用方应传稳定值
- 只监听 storage 事件，同页面其他组件不同步

---

### H7. 实现固定高度虚拟列表

- **难度**: senior
- **标签**: 性能, 虚拟列表
- **面试官看什么**: 可视区计算、缓冲区、占位高度、key

**题干**

实现 `<VirtualList items itemHeight height renderItem />`，10 万条数据也能流畅滚动。

**思路口述**

外层容器固定高度并滚动；内层占位元素高度为总高度，撑出滚动条。根据 scrollTop 计算起止索引，加上 overscan 缓冲，只渲染可见项，并用 translateY 偏移到正确位置。

**参考实现**

```jsx
/**
 * 固定行高的虚拟列表，只渲染可视区域及缓冲区内的元素。
 */
function VirtualList({ items, itemHeight, height, overscan = 5, getKey, renderItem }) {
  const [scrollTop, setScrollTop] = useState(0)

  const totalHeight = items.length * itemHeight
  const startIndex = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan)
  const endIndex = Math.min(
    items.length,
    Math.ceil((scrollTop + height) / itemHeight) + overscan,
  )
  const visibleItems = items.slice(startIndex, endIndex)

  return (
    <div
      style={{ height, overflowY: 'auto' }}
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>
        <div style={{ transform: `translateY(${startIndex * itemHeight}px)` }}>
          {visibleItems.map((item, i) => (
            <div key={getKey(item)} style={{ height: itemHeight }}>
              {renderItem(item, startIndex + i)}
            </div>
          ))}
        </div>
      </div>
    </div>
  )
}
```

**易错点**

- 用 index 作 key，滚动时列表项状态错位
- 没有 overscan，快速滚动出现白屏
- 不定高度需要测量（ResizeObserver）并缓存每项高度，用二分查找起始索引
- 生产推荐 TanStack Virtual

---

### H8. 实现可访问的 Modal

- **难度**: mid
- **标签**: 可访问性, dialog
- **面试官看什么**: 焦点管理、Esc 关闭、遮罩点击、受控 API

**题干**

实现受控 `<Modal open onClose title>`，满足：打开时焦点进入、Tab 不跑出弹窗、Esc 关闭、关闭后焦点回到触发按钮、点击遮罩关闭。

**思路口述**

优先使用原生 `<dialog>` 的 `showModal()`：自带 top layer、焦点约束、Esc 关闭、关闭后焦点恢复，比手写焦点陷阱可靠。React 负责把 `open` prop 同步到 dialog 的命令式 API。

**参考实现**

```jsx
/**
 * 基于原生 dialog 的受控模态框。
 */
function Modal({ open, onClose, title, children }) {
  const dialogRef = useRef(null)
  const titleId = useId()

  useEffect(() => {
    const dialog = dialogRef.current
    if (open && !dialog.open) dialog.showModal()
    if (!open && dialog.open) dialog.close()
  }, [open])

  function handleClick(e) {
    // dialog 自身 padding 为 0，点击到 dialog 元素本身即点击了遮罩区域
    if (e.target === dialogRef.current) onClose()
  }

  return (
    <dialog
      ref={dialogRef}
      aria-labelledby={titleId}
      onClose={onClose}
      onClick={handleClick}
      style={{ padding: 0 }}
    >
      <div style={{ padding: 24 }}>
        <h2 id={titleId}>{title}</h2>
        {children}
        <button type="button" onClick={onClose}>关闭</button>
      </div>
    </dialog>
  )
}
```

**易错点**

- 用 div 模拟弹窗却没有焦点陷阱和 Esc 处理
- dialog 有 padding 时点击内容边缘也会被误判为遮罩
- 只用 CSS 隐藏，读屏仍可访问背景内容
- 需要兼容自定义动画时，可结合 Radix Dialog 等 Headless 库

---

### H9. 实现可重置的 ErrorBoundary

- **难度**: mid
- **标签**: 错误边界
- **面试官看什么**: 类组件生命周期、降级 UI、重置机制、异步错误接入

**题干**

实现 `<ErrorBoundary fallback onError resetKeys>`，出错显示降级 UI，提供 reset，resetKeys 变化时自动恢复。另实现 Hook 让异步错误也能进入错误边界。

**思路口述**

`getDerivedStateFromError` 在渲染阶段更新 state 显示降级 UI；`componentDidCatch` 在提交阶段上报。resetKeys 变化（如路由变化）时清空错误。异步错误捕获不到，可以在 setState 的更新函数中 throw，让错误在渲染阶段抛出。

**参考实现**

```jsx
function arrayChanged(a = [], b = []) {
  return a.length !== b.length || a.some((item, i) => !Object.is(item, b[i]))
}

/**
 * 捕获子树渲染错误并显示降级 UI，resetKeys 变化时自动恢复。
 */
class ErrorBoundary extends Component {
  state = { error: null }

  static getDerivedStateFromError(error) {
    return { error }
  }

  componentDidCatch(error, info) {
    this.props.onError?.(error, info.componentStack)
  }

  componentDidUpdate(prevProps) {
    if (this.state.error && arrayChanged(prevProps.resetKeys, this.props.resetKeys)) {
      this.reset()
    }
  }

  reset = () => {
    this.setState({ error: null })
  }

  render() {
    if (this.state.error) {
      return this.props.fallback({ error: this.state.error, reset: this.reset })
    }
    return this.props.children
  }
}

/**
 * 把事件或异步中的错误抛给最近的错误边界。
 */
function useThrowToBoundary() {
  const [, setState] = useState()
  return useCallback((error) => {
    setState(() => {
      throw error
    })
  }, [])
}
```

**易错点**

- 以为错误边界能捕获事件和 Promise 错误
- reset 后未修复数据，立即再次报错进入循环
- 边界粒度太粗，一个模块错误导致整页白屏
- 生产推荐 react-error-boundary

---

### H10. 实现迷你 Zustand（createStore + useStore）

- **难度**: senior
- **标签**: 状态管理, useSyncExternalStore
- **面试官看什么**: 发布订阅、selector、并发安全、浅合并

**题干**

实现 `create(initializer)`，返回可在组件中使用的 Hook，支持 selector：

```jsx
const useCounter = create((set) => ({
  count: 0,
  inc: () => set((s) => ({ count: s.count + 1 })),
}))

const count = useCounter((s) => s.count)
```

**思路口述**

store 本身是框架无关的发布订阅：保存 state、setState 浅合并后通知订阅者。React 侧用 useSyncExternalStore 订阅，快照为 selector 的结果，只有 selector 结果变化（Object.is）才重新渲染，并天然避免并发撕裂。

**参考实现**

```jsx
/**
 * 框架无关的最小状态容器。
 */
function createStore(initializer) {
  let state
  const listeners = new Set()

  const getState = () => state

  const setState = (partial, replace = false) => {
    const next = typeof partial === 'function' ? partial(state) : partial
    if (Object.is(next, state)) return

    const prev = state
    const shouldMerge = !replace && typeof next === 'object' && next !== null
    state = shouldMerge ? { ...state, ...next } : next
    listeners.forEach((listener) => listener(state, prev))
  }

  const subscribe = (listener) => {
    listeners.add(listener)
    return () => {
      listeners.delete(listener)
    }
  }

  state = initializer(setState, getState)
  return { getState, setState, subscribe }
}

/**
 * 创建绑定 store 的 Hook，组件只在 selector 结果变化时更新。
 */
function create(initializer) {
  const store = createStore(initializer)

  function useBoundStore(selector = (s) => s) {
    return useSyncExternalStore(
      store.subscribe,
      () => selector(store.getState()),
      () => selector(store.getState()),
    )
  }

  Object.assign(useBoundStore, store)
  return useBoundStore
}
```

**易错点**

- selector 返回新对象（如 `s => ({ a: s.a, b: s.b })`）每次快照都不同，导致无限渲染；需要 shallow 比较（Zustand 的 useShallow）
- 用 useState + useEffect 订阅，在并发渲染下可能撕裂
- SSR 中模块级 store 会在请求之间共享，需按请求创建

---

### H11. 实现简易 createElement 和 render

- **难度**: mid
- **标签**: 虚拟 DOM
- **面试官看什么**: JSX 编译结果、虚拟节点结构、属性和事件处理

**题干**

实现 `createElement(type, props, ...children)` 和 `render(element, container)`，支持原生标签、文本、函数组件、事件和 style。

**思路口述**

createElement 返回描述 UI 的普通对象，把字符串/数字子节点包装为文本节点，过滤 null 和布尔值。render 递归：函数组件先调用得到子元素；原生节点创建 DOM、设置属性和事件，再渲染子节点。

**参考实现**

```js
/**
 * 创建虚拟节点，结构与 JSX 编译产物类似。
 */
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children
        .flat()
        .filter((child) => child != null && typeof child !== 'boolean')
        .map((child) => (typeof child === 'object' ? child : createTextElement(child))),
    },
  }
}

function createTextElement(text) {
  return { type: 'TEXT_ELEMENT', props: { nodeValue: String(text), children: [] } }
}

/**
 * 把虚拟节点递归挂载到真实 DOM。
 */
function render(element, container) {
  if (typeof element.type === 'function') {
    const child = element.type(element.props)
    if (child != null) render(child, container)
    return
  }

  const dom =
    element.type === 'TEXT_ELEMENT'
      ? document.createTextNode('')
      : document.createElement(element.type)

  for (const [key, value] of Object.entries(element.props)) {
    if (key === 'children') continue

    if (key.startsWith('on') && typeof value === 'function') {
      dom.addEventListener(key.slice(2).toLowerCase(), value)
    } else if (key === 'style' && typeof value === 'object') {
      Object.assign(dom.style, value)
    } else if (key in dom) {
      dom[key] = value
    } else {
      dom.setAttribute(key, value)
    }
  }

  element.props.children.forEach((child) => render(child, dom))
  container.appendChild(dom)
}
```

**易错点**

- 数字 0 被当作假值过滤（React 会渲染 0）
- 子节点数组未展开（`map` 返回数组）
- 这里只有挂载；完整实现需要 Fiber、Diff、更新与卸载，可延伸讲 [30-senior-qa Q4](30-senior-qa.md) 的 Diff 思路

---

### H12. 实现无限滚动 Hook

- **难度**: mid
- **标签**: IntersectionObserver
- **面试官看什么**: 观察器生命周期、重复触发控制、最新回调

**题干**

实现 `useInfiniteScroll({ hasMore, loading, onLoadMore })`，返回挂在列表底部哨兵元素上的 ref。

**思路口述**

用 IntersectionObserver 观察底部哨兵，进入视口（提前 rootMargin）时加载下一页。loading 或没有更多时不观察，防止重复触发。onLoadMore 用 Effect Event 读取最新值，避免回调变化导致观察器重建。

**参考实现**

```jsx
/**
 * 底部哨兵进入视口时触发加载下一页。
 */
function useInfiniteScroll({ hasMore, loading, onLoadMore, rootMargin = '200px' }) {
  const sentinelRef = useRef(null)
  const loadMore = useEffectEvent(onLoadMore) // React 19.2+；旧版本可用 useLatest 替代

  useEffect(() => {
    const node = sentinelRef.current
    if (!node || !hasMore || loading) return

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) loadMore()
      },
      { rootMargin },
    )

    observer.observe(node)
    return () => observer.disconnect()
  }, [hasMore, loading, rootMargin])

  return sentinelRef
}
```

**易错点**

- 监听 scroll 事件并频繁计算位置，性能差
- loading 期间仍触发，导致重复请求同一页
- 首屏数据不足一屏时不会触发滚动；observer 在 loading 结束后重建会再次检测，正好解决
- 列表很长时需结合虚拟列表
