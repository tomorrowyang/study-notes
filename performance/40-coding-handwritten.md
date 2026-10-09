# 前端性能优化手写与现场编码

> 目标：能现场写出性能相关的常见工具函数和组件，并讲清楚设计考虑。

## 使用方式

- **基础工具类**（H1、H6、H7、H9）：高频手写题，要求边界处理完整
- **渲染与调度类**（H2、H3、H10、H12）：考察对主线程、帧和渲染的理解
- **工程类**（H4、H5、H8、H11）：考察实际项目经验

---

### H1. 手写防抖和节流（支持 leading / trailing / cancel）

- **难度**: 中级
- **标签**: 工具函数
- **面试官看什么**: this 和参数传递、边界选项、取消能力

**题干**

实现 `debounce(fn, wait, { leading, trailing })` 和 `throttle(fn, wait)`，返回的函数支持 `cancel()`。

**思路口述**

防抖每次调用都重置定时器，定时器到期时执行最后一次调用。leading 表示一轮的第一次调用立即执行。节流记录上次执行时间，间隔不足时安排一次尾部执行，保证最后一次调用不丢失。

**参考实现**

```ts
type AnyFn = (...args: any[]) => void

/** 带取消能力的防抖函数 */
export interface Debounced<T extends AnyFn> {
  (this: unknown, ...args: Parameters<T>): void
  cancel(): void
}

/** 防抖：停止调用 wait 毫秒后才执行 */
export function debounce<T extends AnyFn>(
  fn: T,
  wait: number,
  { leading = false, trailing = true } = {},
): Debounced<T> {
  let timer: ReturnType<typeof setTimeout> | undefined
  let lastArgs: Parameters<T> | undefined
  let lastThis: unknown

  const debounced = function (this: unknown, ...args: Parameters<T>) {
    const isNewRound = timer === undefined
    lastArgs = args
    lastThis = this

    if (timer !== undefined) clearTimeout(timer)

    if (leading && isNewRound) {
      fn.apply(this, args)
      // leading 已执行，本轮尾部不应重复执行同一次调用
      lastArgs = undefined
    }

    timer = setTimeout(() => {
      if (trailing && lastArgs) fn.apply(lastThis, lastArgs)
      timer = undefined
      lastArgs = undefined
    }, wait)
  } as Debounced<T>

  debounced.cancel = () => {
    if (timer !== undefined) clearTimeout(timer)
    timer = undefined
    lastArgs = undefined
  }

  return debounced
}

/** 节流：wait 毫秒内最多执行一次，并保证最后一次调用会被执行 */
export function throttle<T extends AnyFn>(fn: T, wait: number): Debounced<T> {
  let lastTime = 0
  let timer: ReturnType<typeof setTimeout> | undefined

  const throttled = function (this: unknown, ...args: Parameters<T>) {
    const now = Date.now()
    const remaining = wait - (now - lastTime)

    if (remaining <= 0) {
      if (timer !== undefined) {
        clearTimeout(timer)
        timer = undefined
      }
      lastTime = now
      fn.apply(this, args)
    } else if (timer === undefined) {
      timer = setTimeout(() => {
        lastTime = Date.now()
        timer = undefined
        fn.apply(this, args)
      }, remaining)
    }
  } as Debounced<T>

  throttled.cancel = () => {
    if (timer !== undefined) clearTimeout(timer)
    timer = undefined
    lastTime = 0
  }

  return throttled
}
```

**易错点**

- 箭头函数导致 this 丢失
- leading 和 trailing 同时开启时，单次调用执行两次
- 节流丢失最后一次调用（例如滚动停止时的最终位置）

---

### H2. 手写可让出主线程的任务调度器

- **难度**: 高级
- **标签**: 调度, INP
- **面试官看什么**: 时间切片、让出主线程、优先级、降级

**题干**

实现 `runTasks(tasks, { budget })`：依次执行大量任务，每执行约 `budget` 毫秒就让出主线程，让浏览器处理输入和渲染。支持取消。

**思路口述**

记录本轮开始时间，超过预算后 await 一次让出。让出优先使用 `scheduler.yield()`，因为续体会被优先调度；不支持时用 MessageChannel，比 setTimeout 没有 4ms 嵌套限制。取消用 AbortSignal。

**参考实现**

```ts
/** 让出主线程，优先使用 scheduler.yield，否则降级为 MessageChannel */
export function yieldToMain(): Promise<void> {
  const scheduler = (globalThis as any).scheduler
  if (typeof scheduler?.yield === 'function') return scheduler.yield()

  return new Promise((resolve) => {
    // MessageChannel 不受 setTimeout 嵌套 4ms 最小延迟的限制
    const { port1, port2 } = new MessageChannel()
    port1.onmessage = () => {
      port1.close()
      resolve()
    }
    port2.postMessage(null)
  })
}

/** 任务执行选项 */
export interface RunTasksOptions {
  /** 每个时间片的预算（毫秒），默认 8ms，给浏览器留出渲染时间 */
  budget?: number
  signal?: AbortSignal
}

/** 分片执行任务，避免产生长任务 */
export async function runTasks<T>(
  tasks: Array<() => T>,
  { budget = 8, signal }: RunTasksOptions = {},
): Promise<T[]> {
  const results: T[] = []
  let sliceStart = performance.now()

  for (const task of tasks) {
    if (signal?.aborted) throw signal.reason ?? new DOMException('Aborted', 'AbortError')

    results.push(task())

    if (performance.now() - sliceStart >= budget) {
      await yieldToMain()
      sliceStart = performance.now()
    }
  }

  return results
}
```

使用：

```ts
const controller = new AbortController()
const rows = await runTasks(
  data.map((item) => () => transform(item)),
  { signal: controller.signal },
)
```

**易错点**

- 每个任务都让出一次，调度开销过大
- 用 `setTimeout(0)` 连续嵌套，触发 4ms 最小延迟
- 单个任务本身就超过预算时，切片无效（需要进一步拆分任务）

---

### H3. 手写虚拟列表（支持不定高）

- **难度**: 高级
- **标签**: 渲染, 列表
- **面试官看什么**: 可见区计算、缓冲区、不定高的预估与测量

**题干**

用 React 实现虚拟列表组件，支持 10 万条数据，行高不固定。

**思路口述**

先用预估高度计算每一项的偏移，渲染后用 ResizeObserver 测量真实高度并更新缓存。根据 scrollTop 用二分查找找到起始索引，向后累加直到超出视口，前后各加一个缓冲区。容器用总高度撑开滚动条，可见项用 translateY 定位。

**参考实现**

```tsx
import { useCallback, useLayoutEffect, useMemo, useRef, useState, type ReactNode } from 'react'

/** 虚拟列表的属性 */
export interface VirtualListProps<T> {
  items: T[]
  height: number
  estimateHeight: number
  overscan?: number
  renderItem: (item: T, index: number) => ReactNode
}

/** 二分查找第一个底部超过 scrollTop 的项 */
function findStartIndex(offsets: number[], scrollTop: number): number {
  let low = 0
  let high = offsets.length - 1
  while (low < high) {
    const mid = (low + high) >> 1
    if (offsets[mid + 1] <= scrollTop) low = mid + 1
    else high = mid
  }
  return low
}

/** 支持不定高的虚拟列表，只渲染可见区域和缓冲区 */
export function VirtualList<T>({
  items,
  height,
  estimateHeight,
  overscan = 5,
  renderItem,
}: VirtualListProps<T>) {
  const [scrollTop, setScrollTop] = useState(0)
  // 测量到的真实高度；用 ref 存储，测量变化时用 version 触发重新计算
  const heightsRef = useRef(new Map<number, number>())
  const [version, setVersion] = useState(0)

  // offsets[i] 为第 i 项的顶部位置，offsets[n] 为总高度
  const offsets = useMemo(() => {
    const result = new Array<number>(items.length + 1)
    result[0] = 0
    for (let i = 0; i < items.length; i++) {
      result[i + 1] = result[i] + (heightsRef.current.get(i) ?? estimateHeight)
    }
    return result
    // version 变化代表有新的测量结果
  }, [items.length, estimateHeight, version])

  const start = Math.max(0, findStartIndex(offsets, scrollTop) - overscan)
  let end = start
  while (end < items.length && offsets[end] < scrollTop + height) end++
  end = Math.min(items.length, end + overscan)

  const observerRef = useRef<ResizeObserver>()
  useLayoutEffect(() => {
    observerRef.current = new ResizeObserver((entries) => {
      let changed = false
      for (const entry of entries) {
        const index = Number((entry.target as HTMLElement).dataset.index)
        const measured = entry.borderBoxSize[0].blockSize
        if (heightsRef.current.get(index) !== measured) {
          heightsRef.current.set(index, measured)
          changed = true
        }
      }
      if (changed) setVersion((v) => v + 1)
    })
    return () => observerRef.current?.disconnect()
  }, [])

  const measureRef = useCallback((el: HTMLDivElement | null) => {
    if (el) observerRef.current?.observe(el)
  }, [])

  return (
    <div
      style={{ height, overflowY: 'auto', position: 'relative' }}
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
    >
      <div style={{ height: offsets[items.length], position: 'relative' }}>
        {items.slice(start, end).map((item, i) => {
          const index = start + i
          return (
            <div
              key={index}
              ref={measureRef}
              data-index={index}
              style={{ position: 'absolute', top: 0, left: 0, right: 0, transform: `translateY(${offsets[index]}px)` }}
            >
              {renderItem(item, index)}
            </div>
          )
        })}
      </div>
    </div>
  )
}
```

**易错点**

- 线性查找起始索引，10 万条数据时每次滚动都很慢
- 没有缓冲区，快速滚动时出现白屏
- 上方的项测量后高度变化，导致可见内容跳动（生产级实现需要在测量时修正 scrollTop）
- 卸载的节点没有 unobserve（这里依赖 disconnect；长时间滚动时生产实现应在节点卸载时 unobserve）

---

### H4. 手写图片懒加载组件

- **难度**: 中级
- **标签**: 图片, 加载
- **面试官看什么**: IntersectionObserver、占位防 CLS、原生能力优先

**题干**

实现 React 图片组件：首屏图片（`priority`）立即以高优先级加载；其他图片进入视口附近时加载；加载前显示占位，避免布局偏移；加载失败显示兜底图。

**思路口述**

优先使用原生 `loading="lazy"`，代码最少且对爬虫友好。priority 图片设置 `fetchpriority="high"` 并且不懒加载。用 width / height 和 aspect-ratio 预留空间，避免 CLS。

**参考实现**

```tsx
import { useState, type ImgHTMLAttributes } from 'react'

/** 图片组件属性 */
export interface ImageProps extends Omit<ImgHTMLAttributes<HTMLImageElement>, 'loading'> {
  src: string
  width: number
  height: number
  alt: string
  /** 首屏或 LCP 图片：不懒加载并提高优先级 */
  priority?: boolean
  fallback?: string
}

/** 带懒加载、占位和失败兜底的图片组件 */
export function Image({ src, width, height, alt, priority = false, fallback, style, ...rest }: ImageProps) {
  const [loaded, setLoaded] = useState(false)
  const [failed, setFailed] = useState(false)

  return (
    <img
      {...rest}
      src={failed && fallback ? fallback : src}
      width={width}
      height={height}
      alt={alt}
      // LCP 图片懒加载会推迟 LCP，所以首屏图片必须 eager
      loading={priority ? 'eager' : 'lazy'}
      fetchPriority={priority ? 'high' : 'auto'}
      decoding={priority ? 'sync' : 'async'}
      onLoad={() => setLoaded(true)}
      onError={() => setFailed(true)}
      style={{
        // 宽高比预留空间，避免图片加载后撑开布局导致 CLS
        aspectRatio: `${width} / ${height}`,
        height: 'auto',
        maxWidth: '100%',
        background: loaded ? undefined : '#f2f2f2',
        ...style,
      }}
    />
  )
}
```

需要自定义加载时机（例如背景图、组件懒加载）时，再使用 IntersectionObserver（见 [初级 Q18](10-junior-qa.md)）。

**易错点**

- 所有图片都懒加载，包括 LCP 图片
- 没有设置尺寸，加载后出现布局偏移
- 失败兜底图也失败时陷入无限 onError 循环（兜底图要确保可用，或判断已经使用过兜底）

---

### H5. 手写性能指标采集 SDK

- **难度**: 高级
- **标签**: 监控
- **面试官看什么**: 指标采集、上报可靠性、采样、批量、页面生命周期

**题干**

实现一个轻量的性能 SDK：采集 Core Web Vitals（带归因）、长动画帧；支持采样、批量上报；页面隐藏时可靠上报。

**思路口述**

指标采集复用 web-vitals 库，它处理了 bfcache、预渲染等边界情况。上报用队列批量发送，页面隐藏时通过 sendBeacon 一次性发送剩余数据（unload 在移动端不可靠，还会阻止 bfcache）。采样在会话开始时决定一次，保证同一会话的数据完整。

**参考实现**

```ts
import { onCLS, onINP, onLCP, onFCP, onTTFB, type Metric } from 'web-vitals/attribution'

/** SDK 初始化配置 */
export interface PerfSdkOptions {
  endpoint: string
  appVersion: string
  sampleRate?: number
  maxBatch?: number
}

interface PerfEvent {
  type: string
  value: number
  detail?: unknown
  page: string
  ts: number
}

/** 初始化性能采集 SDK */
export function initPerfSdk({ endpoint, appVersion, sampleRate = 0.1, maxBatch = 10 }: PerfSdkOptions): void {
  // 采样在会话开始时决定一次：按事件采样会导致同一会话的指标不完整
  if (Math.random() >= sampleRate) return

  const queue: PerfEvent[] = []
  const common = {
    appVersion,
    connection: (navigator as any).connection?.effectiveType,
    deviceMemory: (navigator as any).deviceMemory,
    cores: navigator.hardwareConcurrency,
  }

  const flush = () => {
    if (!queue.length) return
    const body = JSON.stringify({ ...common, events: queue.splice(0) })
    // sendBeacon 在页面卸载后仍能发送；失败时（例如超过大小限制）降级为 keepalive fetch
    if (!navigator.sendBeacon(endpoint, body)) {
      fetch(endpoint, { method: 'POST', body, keepalive: true }).catch(() => {})
    }
  }

  const push = (event: PerfEvent) => {
    queue.push(event)
    if (queue.length >= maxBatch) flush()
  }

  const onMetric = (metric: Metric) => {
    push({
      type: metric.name,
      value: metric.value,
      detail: { rating: metric.rating, attribution: metric.attribution, navigationType: metric.navigationType },
      page: location.pathname,
      ts: Date.now(),
    })
  }

  onLCP(onMetric)
  onINP(onMetric)
  onCLS(onMetric)
  onFCP(onMetric)
  onTTFB(onMetric)

  if (PerformanceObserver.supportedEntryTypes?.includes('long-animation-frame')) {
    new PerformanceObserver((list) => {
      for (const entry of list.getEntries() as any[]) {
        // 只上报明显的长帧，避免数据量过大
        if (entry.duration < 150) continue
        push({
          type: 'loaf',
          value: entry.duration,
          detail: entry.scripts.slice(0, 5).map((s: any) => ({
            url: s.sourceURL,
            fn: s.sourceFunctionName,
            invoker: s.invoker,
            duration: s.duration,
          })),
          page: location.pathname,
          ts: Date.now(),
        })
      }
    }).observe({ type: 'long-animation-frame', buffered: true })
  }

  // visibilitychange 是移动端最后可靠的时机，unload 经常不触发
  addEventListener('visibilitychange', () => {
    if (document.visibilityState === 'hidden') flush()
  })
}
```

**易错点**

- 在 `unload` 中上报（移动端不可靠，并且阻止 bfcache）
- 按事件随机采样，导致同一会话的数据缺失
- 上报不带版本、设备、网络等维度，无法分析
- SDK 本身在主线程做大量工作，影响性能

---

### H6. 手写并发请求控制

- **难度**: 中级
- **标签**: 网络, 异步
- **面试官看什么**: 并发控制、结果顺序、错误处理

**题干**

实现 `limitConcurrency(tasks, limit)`：同时最多执行 limit 个异步任务，返回结果顺序与输入一致；一个失败不影响其他任务。

**思路口述**

启动 limit 个 worker，每个 worker 循环从共享的索引中取下一个任务执行，直到任务取完。用索引保存结果，保证顺序。用 allSettled 风格的结果，单个失败不影响其他。

**参考实现**

```ts
/** 单个任务的执行结果 */
export type TaskResult<T> =
  | { status: 'fulfilled'; value: T }
  | { status: 'rejected'; reason: unknown }

/** 限制并发数执行异步任务，结果顺序与输入一致 */
export async function limitConcurrency<T>(
  tasks: Array<() => Promise<T>>,
  limit: number,
): Promise<TaskResult<T>[]> {
  const results = new Array<TaskResult<T>>(tasks.length)
  let nextIndex = 0

  async function worker() {
    // JS 单线程，读取并递增 nextIndex 是原子的，多个 worker 不会取到同一个任务
    while (nextIndex < tasks.length) {
      const index = nextIndex++
      try {
        results[index] = { status: 'fulfilled', value: await tasks[index]() }
      } catch (reason) {
        results[index] = { status: 'rejected', reason }
      }
    }
  }

  await Promise.all(Array.from({ length: Math.min(limit, tasks.length) }, worker))
  return results
}
```

**易错点**

- 传入 Promise 而不是返回 Promise 的函数（Promise 创建时就已经开始执行，无法控制并发）
- 结果按完成顺序 push，顺序与输入不一致
- 一个任务失败导致整体 reject

---

### H7. 手写带缓存和去重的请求函数

- **难度**: 中级
- **标签**: 网络, 缓存
- **面试官看什么**: 请求去重、缓存过期、stale-while-revalidate

**题干**

实现 `cachedFetch(key, fetcher, { ttl, staleTime })`：同一个 key 的并发请求只发一次；ttl 内直接返回缓存；超过 ttl 但在 staleTime 内时，返回旧数据并在后台刷新。

**参考实现**

```ts
interface CacheEntry<T> {
  data: T
  updatedAt: number
}

const cache = new Map<string, CacheEntry<unknown>>()
const inflight = new Map<string, Promise<unknown>>()

/** 缓存配置 */
export interface CacheOptions {
  /** 新鲜期：期间直接返回缓存 */
  ttl: number
  /** 可接受的过期时长：期间返回旧数据并后台刷新 */
  staleTime: number
}

function request<T>(key: string, fetcher: () => Promise<T>): Promise<T> {
  const existing = inflight.get(key)
  if (existing) return existing as Promise<T>

  const promise = fetcher()
    .then((data) => {
      cache.set(key, { data, updatedAt: Date.now() })
      return data
    })
    .finally(() => inflight.delete(key))

  inflight.set(key, promise)
  return promise
}

/** 带去重和 stale-while-revalidate 的请求 */
export async function cachedFetch<T>(
  key: string,
  fetcher: () => Promise<T>,
  { ttl, staleTime }: CacheOptions,
): Promise<T> {
  const entry = cache.get(key) as CacheEntry<T> | undefined
  const age = entry ? Date.now() - entry.updatedAt : Infinity

  if (entry && age < ttl) return entry.data

  if (entry && age < ttl + staleTime) {
    // 后台刷新失败不影响本次返回旧数据
    request(key, fetcher).catch(() => {})
    return entry.data
  }

  return request(key, fetcher)
}
```

**易错点**

- 请求失败后 inflight 没有清除，后续请求一直拿到失败的 Promise
- 后台刷新的错误未捕获，产生 unhandled rejection
- 缓存无限增长（生产中需要 LRU 或容量限制，见 H9）

---

### H8. 手写悬停预加载的链接组件

- **难度**: 中级
- **标签**: 预加载, 路由
- **面试官看什么**: 意图预测、去重、网络状况判断

**题干**

实现 `<PrefetchLink to loader>`：鼠标悬停约 100ms 或获得焦点时，预加载目标路由的代码；移动端在链接进入视口时预加载；用户开启省流量时不预加载。

**参考实现**

```tsx
import { useEffect, useRef, type AnchorHTMLAttributes } from 'react'

const prefetched = new Set<string>()

function shouldPrefetch(): boolean {
  const connection = (navigator as any).connection
  // 尊重用户的省流量设置，弱网下预加载会抢占当前页面的带宽
  if (connection?.saveData) return false
  if (/2g/.test(connection?.effectiveType ?? '')) return false
  return true
}

/** 预加载链接属性 */
export interface PrefetchLinkProps extends AnchorHTMLAttributes<HTMLAnchorElement> {
  to: string
  /** 目标路由的懒加载函数，例如 () => import('./pages/Detail') */
  loader: () => Promise<unknown>
  hoverDelay?: number
}

/** 根据用户意图预加载目标路由代码的链接 */
export function PrefetchLink({ to, loader, hoverDelay = 100, children, ...rest }: PrefetchLinkProps) {
  const ref = useRef<HTMLAnchorElement>(null)
  const timer = useRef<ReturnType<typeof setTimeout>>()

  const prefetch = () => {
    if (prefetched.has(to) || !shouldPrefetch()) return
    prefetched.add(to)
    loader().catch(() => prefetched.delete(to)) // 失败后允许重试
  }

  useEffect(() => {
    // 触屏设备没有悬停，改为进入视口时预加载
    if (!matchMedia('(hover: none)').matches || !ref.current) return
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        prefetch()
        observer.disconnect()
      }
    })
    observer.observe(ref.current)
    return () => observer.disconnect()
  }, [to])

  return (
    <a
      {...rest}
      ref={ref}
      href={to}
      onMouseEnter={() => (timer.current = setTimeout(prefetch, hoverDelay))}
      onMouseLeave={() => clearTimeout(timer.current)}
      onFocus={prefetch}
    >
      {children}
    </a>
  )
}
```

**易错点**

- 鼠标划过就立即预加载，误触发太多
- 重复预加载同一个路由
- 不考虑省流量和弱网

---

### H9. 手写 LRU 缓存

- **难度**: 中级
- **标签**: 缓存, 数据结构
- **面试官看什么**: O(1) 的 get / set，利用 Map 的插入顺序

**参考实现**

```ts
/** 最近最少使用缓存，超过容量时淘汰最久未访问的项 */
export class LRUCache<K, V> {
  private map = new Map<K, V>()

  constructor(private readonly capacity: number) {}

  get(key: K): V | undefined {
    if (!this.map.has(key)) return undefined
    const value = this.map.get(key)!
    // Map 按插入顺序迭代：删除后重新插入，把它移到「最新」位置
    this.map.delete(key)
    this.map.set(key, value)
    return value
  }

  set(key: K, value: V): void {
    if (this.map.has(key)) this.map.delete(key)
    this.map.set(key, value)
    if (this.map.size > this.capacity) {
      // 第一个键就是最久未访问的
      const oldest = this.map.keys().next().value as K
      this.map.delete(oldest)
    }
  }

  has(key: K): boolean {
    return this.map.has(key)
  }
}
```

**易错点**

- get 时没有更新访问顺序
- 用数组维护顺序，get / set 变成 O(n)

---

### H10. 手写 DOM 读写批处理，避免布局抖动

- **难度**: 高级
- **标签**: 渲染
- **面试官看什么**: 理解强制同步布局，读写分离调度

**题干**

实现 `measure(fn)` 和 `mutate(fn)`：同一帧内所有的读操作先执行，所有的写操作后执行。

**思路口述**

把读和写分别放入两个队列，用 rAF 在下一帧开始时统一处理：先执行所有读（此时布局是有效的，不会强制重新计算），再执行所有写。写操作中产生的新读写会被安排到下一帧。

**参考实现**

```ts
type Job = () => void

const reads: Job[] = []
const writes: Job[] = []
let scheduled = false

function flush() {
  scheduled = false
  // 先读后写：读取时布局有效；写完之后布局失效，但本帧不再读取，就不会强制同步布局
  const currentReads = reads.splice(0)
  const currentWrites = writes.splice(0)
  for (const job of currentReads) job()
  for (const job of currentWrites) job()
  // 执行过程中新加入的任务安排到下一帧
  if (reads.length || writes.length) schedule()
}

function schedule() {
  if (scheduled) return
  scheduled = true
  requestAnimationFrame(flush)
}

/** 安排一次 DOM 读取，与同一帧的其他读取一起执行 */
export function measure(job: Job): void {
  reads.push(job)
  schedule()
}

/** 安排一次 DOM 写入，在同一帧的所有读取之后执行 */
export function mutate(job: Job): void {
  writes.push(job)
  schedule()
}
```

使用：

```ts
for (const el of items) {
  measure(() => {
    const width = container.offsetWidth
    mutate(() => {
      el.style.width = `${width}px`
    })
  })
}
```

**易错点**

- 读写在同一个回调中交替执行，仍然导致抖动
- 写操作中嵌套的读取在本帧执行，再次强制布局

---

### H11. 手写简化版 Worker RPC（类似 Comlink）

- **难度**: 高级
- **标签**: Worker
- **面试官看什么**: 消息匹配、Promise 化、错误传递、Transferable

**题干**

实现 `wrap(worker)` 返回一个代理对象，调用 `api.method(...args)` 返回 Promise；Worker 端用 `expose(obj)` 暴露方法。

**参考实现**

```ts
// rpc.ts
interface RpcRequest {
  id: number
  method: string
  args: unknown[]
}

interface RpcResponse {
  id: number
  result?: unknown
  error?: { message: string; stack?: string }
}

/** 把 Worker 包装为可以直接调用方法的代理对象 */
export function wrap<T extends Record<string, (...args: any[]) => any>>(worker: Worker) {
  let nextId = 0
  const pending = new Map<number, { resolve: (v: unknown) => void; reject: (e: Error) => void }>()

  worker.addEventListener('message', (event: MessageEvent<RpcResponse>) => {
    const { id, result, error } = event.data
    const handlers = pending.get(id)
    if (!handlers) return
    pending.delete(id)
    if (error) {
      const err = new Error(error.message)
      err.stack = error.stack
      handlers.reject(err)
    } else {
      handlers.resolve(result)
    }
  })

  return new Proxy({} as { [K in keyof T]: (...args: Parameters<T[K]>) => Promise<Awaited<ReturnType<T[K]>>> }, {
    get(_, method: string) {
      return (...args: unknown[]) =>
        new Promise((resolve, reject) => {
          const id = nextId++
          pending.set(id, { resolve, reject })
          // ArrayBuffer 参数作为 Transferable 转移所有权，避免大数据的结构化克隆拷贝
          const transfer = args.filter((a): a is ArrayBuffer => a instanceof ArrayBuffer)
          worker.postMessage({ id, method, args } satisfies RpcRequest, transfer)
        })
    },
  })
}

/** 在 Worker 中暴露对象的方法，供主线程调用 */
export function expose(api: Record<string, (...args: any[]) => unknown>): void {
  self.addEventListener('message', async (event: MessageEvent<RpcRequest>) => {
    const { id, method, args } = event.data
    try {
      const result = await api[method](...args)
      self.postMessage({ id, result } satisfies RpcResponse)
    } catch (e) {
      // Error 对象的部分属性无法被结构化克隆完整保留，显式传递 message 和 stack
      const err = e as Error
      self.postMessage({ id, error: { message: err.message, stack: err.stack } } satisfies RpcResponse)
    }
  })
}
```

**易错点**

- 没有用 id 匹配请求和响应，并发调用时结果错乱
- Worker 中的错误没有传回主线程，Promise 永远不 resolve
- 大数据使用结构化克隆，传输成本高

---

### H12. 分片渲染大量 DOM 节点

- **难度**: 中级
- **标签**: 渲染
- **面试官看什么**: rAF 分批、DocumentFragment、可取消

**题干**

不使用虚拟列表，把 10 万条数据渲染到页面上，要求页面不卡死，首批内容尽快出现。

**参考实现**

```ts
/** 分帧渲染大量列表项，每帧渲染一批，返回取消函数 */
export function renderInChunks<T>(
  container: HTMLElement,
  items: T[],
  createNode: (item: T) => HTMLElement,
  chunkSize = 200,
): () => void {
  let index = 0
  let frameId = 0

  const renderChunk = () => {
    // DocumentFragment 先在内存中组装，一次性插入，只触发一次布局
    const fragment = document.createDocumentFragment()
    const end = Math.min(index + chunkSize, items.length)
    for (; index < end; index++) {
      fragment.appendChild(createNode(items[index]))
    }
    container.appendChild(fragment)

    if (index < items.length) frameId = requestAnimationFrame(renderChunk)
  }

  frameId = requestAnimationFrame(renderChunk)
  return () => cancelAnimationFrame(frameId)
}
```

**口述补充**：分片渲染解决了「一次性渲染导致长任务」的问题，但最终 DOM 仍有 10 万个节点，内存和后续交互成本很高；数据量大时应该优先使用虚拟列表，或者配合 `content-visibility: auto`。

**易错点**

- chunkSize 过大，单帧仍然是长任务
- 逐个 appendChild 到已挂载的容器
- 组件卸载时没有取消，继续向已移除的容器插入节点
