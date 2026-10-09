# Vue 手写题

> 目标：通过手写核心机制的简化实现，加深对响应式、调度、diff、生态库原理的理解；同时练习常见的业务组件和 composable。面试时先说思路，再写核心代码，最后说明与真实实现的差异。

## 使用方式

- **原理实现**：H1–H6，迷你响应式、computed、watch、调度器、LIS、diff
- **生态原理**：H7–H8，迷你 Router、迷你 Pinia
- **业务编码**：H9–H12，composable、组件、指令

---

### H1. 实现迷你响应式系统（reactive + effect）

- **难度**: 中级
- **标签**: 响应式
- **面试官看什么**: 依赖收集的数据结构、activeEffect、嵌套 effect、依赖清理

**题干**

实现 `reactive`、`effect`，支持嵌套对象懒代理、嵌套 effect、分支切换时清理旧依赖，并支持 `scheduler` 选项。

**思路口述**

Proxy 的 get 中 track，set 中 trigger。用 `WeakMap<target, Map<key, Set<effect>>>` 存储依赖。effect 执行时设置全局 activeEffect，并用栈处理嵌套。每次执行前清空 effect 的旧依赖，重新收集，解决分支切换（`ok ? a : b`）留下的无效依赖。

**参考实现**

```ts
type EffectFn = (() => void) & {
  deps: Set<EffectFn>[]
  scheduler?: (fn: EffectFn) => void
}

const targetMap = new WeakMap<object, Map<PropertyKey, Set<EffectFn>>>()
let activeEffect: EffectFn | undefined
const effectStack: EffectFn[] = []

function cleanup(effectFn: EffectFn) {
  for (const dep of effectFn.deps) dep.delete(effectFn)
  effectFn.deps.length = 0
}

/** 创建一个副作用，执行期间访问的响应式属性会成为它的依赖。 */
export function effect(fn: () => void, options: { scheduler?: (fn: EffectFn) => void } = {}) {
  const effectFn: EffectFn = Object.assign(
    () => {
      // 每次执行前清空旧依赖，避免分支切换后不再访问的属性仍能触发它
      cleanup(effectFn)
      activeEffect = effectFn
      effectStack.push(effectFn)
      try {
        fn()
      } finally {
        // 嵌套 effect 执行完后恢复外层 effect
        effectStack.pop()
        activeEffect = effectStack[effectStack.length - 1]
      }
    },
    { deps: [] as Set<EffectFn>[], scheduler: options.scheduler },
  )
  effectFn()
  return effectFn
}

function track(target: object, key: PropertyKey) {
  if (!activeEffect) return
  let depsMap = targetMap.get(target)
  if (!depsMap) targetMap.set(target, (depsMap = new Map()))
  let dep = depsMap.get(key)
  if (!dep) depsMap.set(key, (dep = new Set()))
  dep.add(activeEffect)
  activeEffect.deps.push(dep)
}

function trigger(target: object, key: PropertyKey) {
  const dep = targetMap.get(target)?.get(key)
  if (!dep) return
  // 复制一份再遍历：执行 effect 会先删除再重新添加，直接遍历原 Set 会无限循环
  const effectsToRun = new Set(dep)
  for (const effectFn of effectsToRun) {
    // 避免 effect 中修改自身依赖的数据（如 count++）导致无限递归
    if (effectFn === activeEffect) continue
    if (effectFn.scheduler) effectFn.scheduler(effectFn)
    else effectFn()
  }
}

const proxyMap = new WeakMap<object, object>()

/** 返回对象的响应式代理，嵌套对象在访问时才代理。 */
export function reactive<T extends object>(target: T): T {
  const existing = proxyMap.get(target)
  if (existing) return existing as T

  const proxy = new Proxy(target, {
    get(obj, key, receiver) {
      track(obj, key)
      const result = Reflect.get(obj, key, receiver)
      return typeof result === 'object' && result !== null ? reactive(result) : result
    },
    set(obj, key, value, receiver) {
      const oldValue = Reflect.get(obj, key, receiver)
      const result = Reflect.set(obj, key, value, receiver)
      // NaN !== NaN，用 Object.is 避免 NaN 重复赋值时误触发
      if (!Object.is(oldValue, value)) trigger(obj, key)
      return result
    },
    deleteProperty(obj, key) {
      const hadKey = Object.prototype.hasOwnProperty.call(obj, key)
      const result = Reflect.deleteProperty(obj, key)
      if (hadKey && result) trigger(obj, key)
      return result
    },
  })
  proxyMap.set(target, proxy)
  return proxy
}

const state = reactive({ ok: true, text: 'hello', nested: { count: 0 } })
effect(() => console.log(state.ok ? state.text : 'not ok'))
state.ok = false
state.text = 'world'  // 分支切换后不再依赖 text，不会触发
```

**易错点**

- 没有清理旧依赖，分支切换后仍然触发
- 遍历 Set 时直接执行 effect，effect 重新添加自身导致无限循环
- 没有处理 effect 中修改自身依赖导致的无限递归
- 没有缓存代理，同一个对象多次 reactive 返回不同的代理
- 可以补充：真实实现还处理数组方法、`has` / `ownKeys` 拦截、Map / Set、只读代理

---

### H2. 实现 computed

- **难度**: 中级
- **标签**: 响应式, computed
- **面试官看什么**: 惰性求值、dirty 标记、computed 被其他 effect 依赖时的通知

**题干**

基于 H1 的 `effect`，实现 `computed(getter)`：惰性计算、依赖不变时返回缓存值、依赖变化时通知依赖 computed 的 effect。

**思路口述**

用带 scheduler 的 effect 包装 getter，但不立即执行（lazy）。dirty 为 true 时访问 value 才重新计算。依赖变化时 scheduler 只把 dirty 设为 true，并手动触发依赖 computed.value 的 effect。

**参考实现**

```ts
// 在 H1 的基础上给 effect 增加 lazy 选项：lazy 为 true 时不立即执行，返回 effectFn

/** 创建一个惰性求值、带缓存的计算属性。 */
export function computed<T>(getter: () => T) {
  let value: T
  let dirty = true

  const runner = effect(
    () => {
      value = getter()
    },
    {
      lazy: true,
      scheduler() {
        if (!dirty) {
          dirty = true
          // 依赖变化时不立即计算，只通知读取过 computed 的 effect
          trigger(computedRef, 'value')
        }
      },
    },
  )

  const computedRef = {
    get value() {
      if (dirty) {
        runner()
        dirty = false
      }
      track(computedRef, 'value')
      return value
    },
  }
  return computedRef
}

const state = reactive({ price: 10, count: 2 })
const total = computed(() => state.price * state.count)
effect(() => console.log('total', total.value))  // total 20
state.count = 3                                    // total 30
```

对应 effect 的修改：

```ts
export function effect(fn: () => void, options: { scheduler?: (fn: EffectFn) => void; lazy?: boolean } = {}) {
  // ...同 H1
  if (!options.lazy) effectFn()
  return effectFn
}
```

**易错点**

- 依赖变化时立即重新计算，失去惰性
- 没有在读取 value 时 track，依赖 computed 的 effect 不会更新
- 在 scheduler 中重复 trigger（dirty 已经为 true 时不需要再次通知）
- 可以补充：3.4+ 在重新计算后比较新旧值，相同则不通知下游

---

### H3. 实现 watch

- **难度**: 中级
- **标签**: 响应式, watch
- **面试官看什么**: 数据源归一化、深度遍历、新旧值、immediate、清理函数

**题干**

基于 H1 实现 `watch(source, callback, { immediate, deep })`：source 可以是 getter 或 reactive 对象；支持 onCleanup 处理竞态；返回停止函数。

**思路口述**

把 source 统一为 getter：reactive 对象则递归遍历所有属性（读取即收集依赖）。用 lazy effect 执行 getter，scheduler 中重新执行 getter 得到新值，调用回调。每次调用回调前先执行上一次注册的清理函数。

**参考实现**

```ts
function traverse(value: unknown, seen = new Set<unknown>()) {
  // 已访问过的对象直接返回，避免循环引用导致无限递归
  if (typeof value !== 'object' || value === null || seen.has(value)) return value
  seen.add(value)
  for (const key in value) traverse((value as Record<string, unknown>)[key], seen)
  return value
}

type WatchCallback<T> = (value: T, oldValue: T | undefined, onCleanup: (fn: () => void) => void) => void

/** 侦听数据源变化并执行回调，返回停止侦听的函数。 */
export function watch<T>(
  source: (() => T) | object,
  callback: WatchCallback<T>,
  options: { immediate?: boolean; deep?: boolean } = {},
) {
  const isGetter = typeof source === 'function'
  const baseGetter = isGetter ? (source as () => T) : () => source as T
  // reactive 对象默认深度侦听；getter 只有显式 deep 时才深度遍历
  const getter = !isGetter || options.deep ? () => traverse(baseGetter()) as T : baseGetter

  let oldValue: T | undefined
  let cleanup: (() => void) | undefined
  const onCleanup = (fn: () => void) => {
    cleanup = fn
  }

  const job = () => {
    const newValue = runner()
    // 在执行新的回调前清理上一次的副作用（例如取消过期的请求）
    cleanup?.()
    callback(newValue, oldValue, onCleanup)
    oldValue = newValue
  }

  const runner = effect(() => getter(), { lazy: true, scheduler: job }) as unknown as () => T

  if (options.immediate) job()
  else oldValue = runner()

  return () => {
    cleanup?.()
    // 从所有依赖中移除，之后数据变化不再触发
    for (const dep of (runner as unknown as EffectFn).deps) dep.delete(runner as unknown as EffectFn)
  }
}
```

注意：这里需要 H1 的 effect 在 lazy 模式下返回的函数执行后**返回 fn 的结果**（`const result = fn(); return result`）。

**易错点**

- 侦听 reactive 对象时没有深度遍历，修改深层属性不触发
- 深度遍历没有处理循环引用
- 清理函数时机错误：应该在下一次回调执行前，以及停止侦听时
- 侦听 reactive 对象时，新旧值是同一个对象（Vue 的真实行为也是如此）

---

### H4. 实现异步更新队列和 nextTick

- **难度**: 中级
- **标签**: 调度
- **面试官看什么**: 去重、排序、微任务、刷新期间新加入的任务

**题干**

实现 `queueJob(job)` 和 `nextTick(fn)`：同一 tick 内多次加入的同一个 job 只执行一次；按 job.id 从小到大执行（父组件先于子组件）；刷新过程中新加入的 job 也要执行。

**思路口述**

用数组作为队列，加入时检查是否已存在；用 Promise.resolve().then 安排一次刷新；刷新时先排序再依次执行。nextTick 链在当前刷新的 Promise 之后。刷新过程中加入的 job 需要插入到正确的位置（按 id），这里简化为加入后由循环继续执行。

**参考实现**

```ts
interface SchedulerJob {
  (): void
  id: number
}

const queue: SchedulerJob[] = []
let isFlushPending = false
let isFlushing = false
let flushIndex = 0
const resolvedPromise = Promise.resolve()
let currentFlushPromise: Promise<void> | null = null

/** 把任务加入更新队列，同一个任务在一次刷新中只执行一次。 */
export function queueJob(job: SchedulerJob) {
  // 刷新中只检查尚未执行的部分，已执行过的任务允许再次入队（例如在更新中又修改了数据）
  const searchStart = isFlushing ? flushIndex + 1 : 0
  if (queue.includes(job, searchStart)) return

  if (isFlushing) {
    // 按 id 插入到未执行部分的正确位置，保证父组件先于子组件
    let insertAt = queue.length
    for (let i = flushIndex + 1; i < queue.length; i++) {
      if (queue[i].id > job.id) {
        insertAt = i
        break
      }
    }
    queue.splice(insertAt, 0, job)
  } else {
    queue.push(job)
  }
  queueFlush()
}

function queueFlush() {
  if (!isFlushing && !isFlushPending) {
    isFlushPending = true
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}

function flushJobs() {
  isFlushPending = false
  isFlushing = true
  queue.sort((a, b) => a.id - b.id)
  try {
    for (flushIndex = 0; flushIndex < queue.length; flushIndex++) {
      queue[flushIndex]()
    }
  } finally {
    flushIndex = 0
    queue.length = 0
    isFlushing = false
    currentFlushPromise = null
  }
}

/** 在下一次队列刷新完成后执行回调。 */
export function nextTick(fn?: () => void): Promise<void> {
  const promise = currentFlushPromise ?? resolvedPromise
  return fn ? promise.then(fn) : promise
}
```

**易错点**

- 没有去重，同一组件更新多次
- 用 setTimeout 安排刷新：延迟大，且浏览器可能先渲染出中间状态
- 刷新过程中加入的任务没有执行，或者顺序错误
- 没有防止无限循环（真实实现中限制同一任务的递归次数为 100）

---

### H5. 实现最长递增子序列

- **难度**: 高级
- **标签**: 算法, diff
- **面试官看什么**: 贪心 + 二分、前驱回溯、返回索引

**题干**

实现 `getSequence(arr)`，返回最长递增子序列在原数组中的**索引**。数组中的 0 表示新节点，不参与计算。

**思路口述**

维护数组 result，result[k] 是长度为 k+1 的递增子序列的末尾元素索引，并保证末尾元素尽量小。遍历时，比末尾大就追加；否则二分查找第一个大于等于当前值的位置替换。替换会打乱序列，所以用 predecessors 记录每个元素加入时的前一个元素，最后从末尾回溯得到真实序列。

**参考实现**

```ts
/**
 * 返回最长递增子序列的索引数组。
 * 值为 0 的元素表示新增节点，不参与计算。
 */
export function getSequence(arr: number[]): number[] {
  const predecessors = new Array<number>(arr.length)
  const result: number[] = []

  for (let i = 0; i < arr.length; i++) {
    const value = arr[i]
    if (value === 0) continue

    if (result.length === 0 || arr[result[result.length - 1]] < value) {
      predecessors[i] = result.length > 0 ? result[result.length - 1] : -1
      result.push(i)
      continue
    }

    let low = 0
    let high = result.length - 1
    while (low < high) {
      const mid = (low + high) >> 1
      if (arr[result[mid]] < value) low = mid + 1
      else high = mid
    }
    if (value < arr[result[low]]) {
      predecessors[i] = low > 0 ? result[low - 1] : -1
      result[low] = i
    }
  }

  // 贪心替换后 result 只有长度和末尾正确，需要沿前驱链回溯得到真实序列
  let length = result.length
  let current = result[length - 1]
  while (length-- > 0) {
    result[length] = current
    current = predecessors[current]
  }
  return result
}

console.log(getSequence([3, 1, 5, 4, 2, 0, 6]))  // [1, 3, 6] → 值 1, 4, 6（结果之一）
```

**易错点**

- 返回值而不是索引
- 没有回溯，直接返回贪心得到的 result（不是合法的子序列）
- 二分查找边界错误
- 能说明时间复杂度 O(n log n)，以及在 diff 中的用途

---

### H6. 实现带 key 的子节点 diff

- **难度**: 高级
- **标签**: diff
- **面试官看什么**: 首尾预处理、key 映射、卸载、LIS 移动、从后向前插入

**题干**

给定新旧两组带 key 的子节点，以及 `patch`、`mount`、`unmount`、`move` 操作，实现 Vue 3 风格的 `patchKeyedChildren`。

**思路口述**

先从头部、再从尾部比较相同 key 的节点并 patch。剩余部分：只有新节点则挂载，只有旧节点则卸载。中间乱序部分：建立新节点的 key → index 映射；遍历旧节点，找不到的卸载，找到的 patch 并记录到 newIndexToOldIndex；如果出现了逆序，计算 LIS；最后从后向前遍历新节点，未匹配的挂载，不在 LIS 中的移动。

**参考实现**

```ts
interface VNode {
  key: string | number
  el?: unknown
}

interface DomOps {
  patch(oldNode: VNode, newNode: VNode): void
  mount(node: VNode, anchor: VNode | null): void
  unmount(node: VNode): void
  move(node: VNode, anchor: VNode | null): void
}

/** Vue 3 风格的带 key 子节点 diff。anchor 为 null 表示插入到容器末尾。 */
export function patchKeyedChildren(oldChildren: VNode[], newChildren: VNode[], ops: DomOps) {
  let i = 0
  let oldEnd = oldChildren.length - 1
  let newEnd = newChildren.length - 1

  // 1. 头部相同的节点
  while (i <= oldEnd && i <= newEnd && oldChildren[i].key === newChildren[i].key) {
    ops.patch(oldChildren[i], newChildren[i])
    i++
  }
  // 2. 尾部相同的节点
  while (i <= oldEnd && i <= newEnd && oldChildren[oldEnd].key === newChildren[newEnd].key) {
    ops.patch(oldChildren[oldEnd], newChildren[newEnd])
    oldEnd--
    newEnd--
  }

  // 3. 旧节点处理完，挂载剩余的新节点
  if (i > oldEnd) {
    const anchor = newChildren[newEnd + 1] ?? null
    for (let j = i; j <= newEnd; j++) ops.mount(newChildren[j], anchor)
    return
  }
  // 4. 新节点处理完，卸载剩余的旧节点
  if (i > newEnd) {
    for (let j = i; j <= oldEnd; j++) ops.unmount(oldChildren[j])
    return
  }

  // 5. 中间乱序部分
  const newStart = i
  const toBePatched = newEnd - newStart + 1
  const keyToNewIndex = new Map<VNode['key'], number>()
  for (let j = newStart; j <= newEnd; j++) keyToNewIndex.set(newChildren[j].key, j)

  // 值为旧索引 + 1，0 表示没有对应的旧节点（需要新建）
  const newIndexToOldIndex = new Array<number>(toBePatched).fill(0)
  let moved = false
  let maxNewIndexSoFar = 0

  for (let j = i; j <= oldEnd; j++) {
    const oldChild = oldChildren[j]
    const newIndex = keyToNewIndex.get(oldChild.key)
    if (newIndex === undefined) {
      ops.unmount(oldChild)
      continue
    }
    newIndexToOldIndex[newIndex - newStart] = j + 1
    // 旧节点对应的新位置不是递增的，说明有节点需要移动
    if (newIndex >= maxNewIndexSoFar) maxNewIndexSoFar = newIndex
    else moved = true
    ops.patch(oldChild, newChildren[newIndex])
  }

  const stable = moved ? getSequence(newIndexToOldIndex) : []
  let stableIndex = stable.length - 1

  // 从后向前遍历，保证插入时的锚点（后一个节点）已经在正确位置
  for (let j = toBePatched - 1; j >= 0; j--) {
    const newIndex = newStart + j
    const node = newChildren[newIndex]
    const anchor = newChildren[newIndex + 1] ?? null
    if (newIndexToOldIndex[j] === 0) {
      ops.mount(node, anchor)
    } else if (moved) {
      if (stableIndex < 0 || j !== stable[stableIndex]) ops.move(node, anchor)
      else stableIndex--
    }
  }
}
```

**易错点**

- 锚点取错：应该是新数组中的下一个节点
- 从前向后遍历插入，导致锚点还没有就位
- 没有 `moved` 优化，所有情况都计算 LIS
- newIndexToOldIndex 存旧索引时没有 + 1，与「0 表示新节点」冲突

---

### H7. 实现迷你 Vue Router

- **难度**: 高级
- **标签**: Router
- **面试官看什么**: 响应式的当前路由、History API、popstate、RouterView 和 RouterLink

**题干**

实现一个支持 history 模式的迷你路由：`createRouter({ routes })` 返回带 `push`、`replace`、`beforeEach`、`install` 的路由器；提供 `RouterView` 和 `RouterLink` 组件；支持动态参数 `/users/:id`。

**思路口述**

当前路由是一个 shallowRef，RouterView 读取它并渲染匹配的组件，所以路由变化自动触发重新渲染。push 调用 pushState 后手动更新 currentRoute（pushState 不触发 popstate）；监听 popstate 处理前进后退。路径匹配把 `:id` 转为正则捕获组。守卫按顺序执行，返回 false 取消，返回字符串重定向。

**参考实现**

```ts
import { defineComponent, h, inject, shallowRef, type App, type Component, type InjectionKey, type ShallowRef } from 'vue'

interface RouteRecord {
  path: string
  component: Component
}

interface RouteLocation {
  path: string
  params: Record<string, string>
  matched?: RouteRecord
}

type Guard = (to: RouteLocation, from: RouteLocation) => boolean | string | void | Promise<boolean | string | void>

const RouterKey: InjectionKey<Router> = Symbol('router')

/** 把 '/users/:id' 编译为正则和参数名列表。 */
function compilePath(path: string) {
  const keys: string[] = []
  const pattern = path.replace(/:(\w+)/g, (_, key: string) => {
    keys.push(key)
    return '([^/]+)'
  })
  return { regex: new RegExp(`^${pattern}$`), keys }
}

/** 创建一个基于 History API 的迷你路由器。 */
export function createRouter({ routes }: { routes: RouteRecord[] }) {
  const compiled = routes.map((record) => ({ record, ...compilePath(record.path) }))
  const guards: Guard[] = []

  function resolve(path: string): RouteLocation {
    for (const { record, regex, keys } of compiled) {
      const match = path.match(regex)
      if (match) {
        const params = Object.fromEntries(keys.map((key, index) => [key, decodeURIComponent(match[index + 1])]))
        return { path, params, matched: record }
      }
    }
    return { path, params: {} }
  }

  const currentRoute: ShallowRef<RouteLocation> = shallowRef(resolve(location.pathname))

  async function navigate(path: string, mode: 'push' | 'replace' | 'pop'): Promise<void> {
    const to = resolve(path)
    const from = currentRoute.value
    for (const guard of guards) {
      const result = await guard(to, from)
      if (result === false) {
        // 前进后退时 URL 已经改变，取消导航需要恢复 URL
        if (mode === 'pop') history.pushState(null, '', from.path)
        return
      }
      if (typeof result === 'string') return navigate(result, 'replace')
    }
    if (mode === 'push') history.pushState(null, '', path)
    else if (mode === 'replace') history.replaceState(null, '', path)
    currentRoute.value = to
  }

  // pushState 不会触发 popstate，只有前进后退才触发
  window.addEventListener('popstate', () => navigate(location.pathname, 'pop'))

  const router = {
    currentRoute,
    push: (path: string) => navigate(path, 'push'),
    replace: (path: string) => navigate(path, 'replace'),
    beforeEach(guard: Guard) {
      guards.push(guard)
      return () => guards.splice(guards.indexOf(guard), 1)
    },
    install(app: App) {
      app.provide(RouterKey, router)
      app.component('RouterView', RouterView)
      app.component('RouterLink', RouterLink)
    },
  }
  return router
}

type Router = ReturnType<typeof createRouter>

export function useRouter() {
  return inject(RouterKey)!
}

export const RouterView = defineComponent({
  setup() {
    const router = useRouter()
    // 渲染函数读取 currentRoute.value，路由变化时自动重新渲染
    return () => {
      const matched = router.currentRoute.value.matched
      return matched ? h(matched.component, { key: router.currentRoute.value.path }) : h('div', '404')
    }
  },
})

export const RouterLink = defineComponent({
  props: { to: { type: String, required: true } },
  setup(props, { slots }) {
    const router = useRouter()
    return () =>
      h(
        'a',
        {
          href: props.to,
          onClick: (event: MouseEvent) => {
            // 保留新标签页打开等浏览器默认行为
            if (event.metaKey || event.ctrlKey || event.shiftKey || event.button !== 0) return
            event.preventDefault()
            router.push(props.to)
          },
        },
        slots.default?.(),
      )
  },
})
```

**易错点**

- 以为 pushState 会触发 popstate
- currentRoute 不是响应式的，RouterView 不会更新
- RouterLink 拦截了所有点击，导致 Ctrl + 点击无法在新标签页打开
- 前进后退触发的导航被守卫取消时，没有恢复 URL
- 可以补充：真实实现支持嵌套路由（RouterView 通过 depth 注入）、命名路由、query、滚动行为

---

### H8. 实现迷你 Pinia

- **难度**: 中级
- **标签**: Pinia
- **面试官看什么**: 单例缓存、effectScope、通过 provide / inject 获取 pinia、`$reset` / `$subscribe`

**题干**

实现 `createPinia()` 和 `defineStore(id, setup)`，支持 Setup Store；store 是单例；提供 `$subscribe` 订阅状态变化和 `$dispose`。

**思路口述**

pinia 实例保存所有 store 和一个全局 effectScope。defineStore 返回 useStore 函数，首次调用时在独立的 effectScope 中执行 setup，把结果包装为 reactive 对象并缓存。获取 pinia 时优先 inject，在组件外使用活动的 pinia。

**参考实现**

```ts
import { computed, effectScope, getCurrentInstance, inject, reactive, ref, watch, type App, type EffectScope } from 'vue'

interface Pinia {
  install(app: App): void
  scope: EffectScope
  stores: Map<string, any>
}

const PiniaKey = Symbol('pinia')
let activePinia: Pinia | undefined

/** 创建 pinia 实例，所有 store 都在它的 effectScope 下创建。 */
export function createPinia(): Pinia {
  const pinia: Pinia = {
    scope: effectScope(true),
    stores: new Map(),
    install(app) {
      activePinia = pinia
      app.provide(PiniaKey, pinia)
    },
  }
  return pinia
}

/** 定义一个 Setup Store，返回获取单例 store 的函数。 */
export function defineStore<T extends Record<string, unknown>>(id: string, setup: () => T) {
  return function useStore() {
    // 组件内优先使用注入的 pinia；SSR 中每个请求的应用有各自的 pinia
    const pinia = (getCurrentInstance() && inject<Pinia>(PiniaKey)) || activePinia
    if (!pinia) throw new Error('请先调用 app.use(pinia)')

    if (!pinia.stores.has(id)) {
      // 每个 store 在独立的子 scope 中运行，便于 $dispose 时停止其中的 computed 和 watch
      const storeScope = pinia.scope.run(() => effectScope())!
      const setupResult = storeScope.run(setup)!

      const store = reactive({
        ...setupResult,
        $id: id,
        $subscribe(callback: (state: T) => void) {
          return storeScope.run(() => watch(() => store, () => callback(store as T), { deep: true }))!
        },
        $dispose() {
          storeScope.stop()
          pinia.stores.delete(id)
        },
      })
      pinia.stores.set(id, store)
    }
    return pinia.stores.get(id) as T & { $id: string; $subscribe: Function; $dispose: () => void }
  }
}

export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)
  const double = computed(() => count.value * 2)
  const increment = () => count.value++
  return { count, double, increment }
})
```

**易错点**

- 每次调用 useStore 都创建新 store，不是单例
- 没有用 effectScope，store 中的 computed 和 watch 无法统一停止
- 在模块顶层调用 useStore（此时 pinia 还没有安装）
- reactive 包装后，setup 返回的 ref 会被自动解包，所以 `store.count` 直接是数字，这也是解构需要 storeToRefs 的原因

---

### H9. 实现 useRequest composable

- **难度**: 中级
- **标签**: composable, 异步
- **面试官看什么**: 加载和错误状态、竞态处理、参数响应式、手动触发、取消

**题干**

实现 `useRequest(service, options)`：返回 `data`、`error`、`loading`、`run`、`cancel`；支持 `manual`（不自动执行）、`refreshDeps`（依赖变化时重新请求）；新请求发出后，旧请求的结果被忽略。

**思路口述**

用自增的请求编号处理竞态：每次 run 时编号加一，结果返回时编号不一致就丢弃。同时用 AbortController 取消网络请求。依赖变化时用 watch 触发。组件卸载时取消请求，避免卸载后修改状态。

**参考实现**

```ts
import { onScopeDispose, ref, shallowRef, watch, type WatchSource } from 'vue'

interface UseRequestOptions<P extends unknown[]> {
  manual?: boolean
  defaultParams?: P
  refreshDeps?: WatchSource[]
  onSuccess?: (data: unknown) => void
}

/** 封装异步请求的加载、错误、竞态和取消逻辑。 */
export function useRequest<T, P extends unknown[]>(
  service: (signal: AbortSignal, ...params: P) => Promise<T>,
  options: UseRequestOptions<P> = {},
) {
  const data = shallowRef<T>()
  const error = shallowRef<Error>()
  const loading = ref(false)

  let requestId = 0
  let controller: AbortController | undefined
  let lastParams = (options.defaultParams ?? []) as P

  function cancel() {
    controller?.abort()
    // 递增编号，让正在进行的请求返回后被忽略
    requestId++
    loading.value = false
  }

  async function run(...params: P): Promise<T | undefined> {
    cancel()
    const currentId = ++requestId
    controller = new AbortController()
    lastParams = params
    loading.value = true
    error.value = undefined

    try {
      const result = await service(controller.signal, ...params)
      // 有更新的请求发出后，丢弃过期的结果
      if (currentId !== requestId) return undefined
      data.value = result
      options.onSuccess?.(result)
      return result
    } catch (err) {
      if (currentId !== requestId || (err as Error).name === 'AbortError') return undefined
      error.value = err as Error
      return undefined
    } finally {
      if (currentId === requestId) loading.value = false
    }
  }

  const refresh = () => run(...lastParams)

  if (!options.manual) run(...lastParams)
  if (options.refreshDeps?.length) watch(options.refreshDeps, refresh)

  // 组件卸载时取消请求，避免卸载后修改状态
  onScopeDispose(cancel)

  return { data, error, loading, run, refresh, cancel }
}

const keyword = ref('')
const { data, loading } = useRequest(
  (signal) => fetch(`/api/search?q=${keyword.value}`, { signal }).then((r) => r.json()),
  { refreshDeps: [keyword] },
)
```

**易错点**

- 没有处理竞态，旧请求的结果覆盖新结果
- 取消后 loading 一直为 true
- 旧请求的 finally 把新请求的 loading 设为 false
- 卸载后仍然修改状态

---

### H10. 实现虚拟滚动列表组件

- **难度**: 高级
- **标签**: 组件, 性能
- **面试官看什么**: 可视区计算、缓冲区、占位高度、滚动性能

**题干**

实现固定行高的虚拟列表组件 `VirtualList`：props 为 `items`、`itemHeight`、`height`、`buffer`；通过作用域插槽渲染每一项。

**思路口述**

外层容器固定高度并可以滚动；内部一个占位元素撑开总高度；根据 scrollTop 计算起始索引和结束索引（加上缓冲区），只渲染这部分；用 transform 把渲染的列表移动到正确位置。滚动事件中只更新 scrollTop，计算交给 computed。

**参考实现**

```vue
<script setup lang="ts" generic="T">
import { computed, ref } from 'vue'

const props = withDefaults(
  defineProps<{
    items: T[]
    itemHeight: number
    height: number
    buffer?: number
  }>(),
  { buffer: 5 },
)

defineSlots<{ default(props: { item: T; index: number }): any }>()

const scrollTop = ref(0)

const totalHeight = computed(() => props.items.length * props.itemHeight)
const visibleCount = computed(() => Math.ceil(props.height / props.itemHeight))

// 上下各多渲染 buffer 项，避免快速滚动时出现空白
const startIndex = computed(() => Math.max(0, Math.floor(scrollTop.value / props.itemHeight) - props.buffer))
const endIndex = computed(() =>
  Math.min(props.items.length, Math.floor(scrollTop.value / props.itemHeight) + visibleCount.value + props.buffer),
)

const visibleItems = computed(() =>
  props.items.slice(startIndex.value, endIndex.value).map((item, offset) => ({
    item,
    index: startIndex.value + offset,
  })),
)

const offsetY = computed(() => startIndex.value * props.itemHeight)

function onScroll(event: Event) {
  scrollTop.value = (event.target as HTMLElement).scrollTop
}
</script>

<template>
  <div class="virtual-list" :style="{ height: `${height}px` }" @scroll.passive="onScroll">
    <div :style="{ height: `${totalHeight}px` }" />
    <div class="virtual-list__content" :style="{ transform: `translateY(${offsetY}px)` }">
      <div
        v-for="{ item, index } in visibleItems"
        :key="index"
        :style="{ height: `${itemHeight}px` }"
      >
        <slot :item="item" :index="index" />
      </div>
    </div>
  </div>
</template>

<style scoped>
.virtual-list {
  position: relative;
  overflow-y: auto;
}
.virtual-list__content {
  position: absolute;
  inset: 0 0 auto 0;
}
</style>
```

**易错点**

- 没有缓冲区，快速滚动时出现空白
- 用 `top` 定位而不是 `transform`，触发布局
- items 使用深度响应式，大数据时有额外开销（父组件应传入 shallowRef 的数据）
- 这里用 index 作为 key 是因为只渲染窗口内的项；如果每项有内部状态，应使用数据 ID
- 可以补充：动态行高需要测量和缓存每项的高度，用二分查找确定起始索引

---

### H11. 实现 v-lazy 图片懒加载指令

- **难度**: 初级
- **标签**: 指令
- **面试官看什么**: IntersectionObserver、清理、更新时处理新的地址

**题干**

实现 `v-lazy` 指令：`<img v-lazy="url">`，图片进入视口（提前 200px）时才加载；加载失败显示占位图；元素卸载时取消观察；绑定值变化时重新加载。

**思路口述**

所有元素共享一个 IntersectionObserver，用 WeakMap 记录元素对应的地址。进入视口后设置 src 并取消观察。updated 钩子中地址变化时重新观察。unmounted 中取消观察。

**参考实现**

```ts
import type { Directive } from 'vue'

const PLACEHOLDER = 'data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7'
const FALLBACK = '/images/load-failed.png'

const pendingSrc = new WeakMap<HTMLImageElement, string>()

// 所有图片共享一个观察者，比每个元素创建一个更省资源
const observer =
  typeof IntersectionObserver !== 'undefined'
    ? new IntersectionObserver(
        (entries) => {
          for (const entry of entries) {
            if (!entry.isIntersecting) continue
            const img = entry.target as HTMLImageElement
            const src = pendingSrc.get(img)
            if (src) img.src = src
            observer?.unobserve(img)
          }
        },
        { rootMargin: '200px' },
      )
    : null

function load(img: HTMLImageElement, src: string) {
  pendingSrc.set(img, src)
  if (!observer) {
    // 不支持 IntersectionObserver 时直接加载
    img.src = src
    return
  }
  img.src = PLACEHOLDER
  observer.observe(img)
}

/** 图片进入视口附近时才加载真实地址。 */
export const vLazy: Directive<HTMLImageElement, string> = {
  mounted(img, binding) {
    img.addEventListener('error', () => {
      // 避免占位图本身加载失败时无限触发 error
      if (img.src !== new URL(FALLBACK, location.href).href) img.src = FALLBACK
    })
    load(img, binding.value)
  },
  updated(img, binding) {
    if (binding.value !== binding.oldValue) load(img, binding.value)
  },
  unmounted(img) {
    observer?.unobserve(img)
    pendingSrc.delete(img)
  },
}
```

**易错点**

- 每个元素创建一个 IntersectionObserver
- 卸载时没有取消观察
- 错误处理中设置的占位图也失败时，无限循环
- SSR 环境中访问 IntersectionObserver 报错
- 能说明：现代浏览器可以直接使用原生的 `loading="lazy"`，指令适用于需要更多控制的场景

---

### H12. 实现可撤销 / 重做的 composable

- **难度**: 中级
- **标签**: composable
- **面试官看什么**: 快照、历史栈、容量限制、避免撤销本身被记录

**题干**

实现 `useHistory(source, { capacity })`：source 是一个 ref；每次修改自动记录快照；提供 `undo`、`redo`、`canUndo`、`canRedo`、`clear`；撤销和重做本身不应该被记录为新的历史。

**思路口述**

用 watch 侦听 source（深度），变化时把旧值的快照压入撤销栈，清空重做栈。撤销时把当前值压入重做栈，从撤销栈弹出并赋值。赋值会再次触发 watch，所以用一个标记跳过。快照使用 structuredClone 深拷贝，避免引用共享。

**参考实现**

```ts
import { computed, ref, shallowRef, toRaw, watch, type Ref } from 'vue'

/** 为 ref 提供撤销和重做能力。 */
export function useHistory<T>(source: Ref<T>, { capacity = 50 } = {}) {
  const undoStack = shallowRef<T[]>([])
  const redoStack = shallowRef<T[]>([])
  // 撤销和重做时修改 source 会触发 watch，用标记跳过这一次记录
  let isTraveling = false

  const snapshot = (value: T): T => structuredClone(toRaw(value))

  let lastSnapshot = snapshot(source.value)

  watch(
    source,
    (value) => {
      if (isTraveling) {
        isTraveling = false
        lastSnapshot = snapshot(value)
        return
      }
      const nextUndo = [...undoStack.value, lastSnapshot]
      // 超出容量时丢弃最早的记录
      undoStack.value = nextUndo.length > capacity ? nextUndo.slice(-capacity) : nextUndo
      redoStack.value = []
      lastSnapshot = snapshot(value)
    },
    // flush: 'sync' 保证每次修改都被记录，而不是同一 tick 中的多次修改合并为一次
    { deep: true, flush: 'sync' },
  )

  function travel(from: Ref<T[]>, to: Ref<T[]>) {
    if (from.value.length === 0) return
    const target = from.value[from.value.length - 1]
    from.value = from.value.slice(0, -1)
    to.value = [...to.value, snapshot(source.value)]
    isTraveling = true
    source.value = snapshot(target)
  }

  return {
    undo: () => travel(undoStack, redoStack),
    redo: () => travel(redoStack, undoStack),
    canUndo: computed(() => undoStack.value.length > 0),
    canRedo: computed(() => redoStack.value.length > 0),
    clear() {
      undoStack.value = []
      redoStack.value = []
    },
  }
}

const doc = ref({ title: '', content: '' })
const { undo, redo, canUndo } = useHistory(doc)
```

**易错点**

- 快照没有深拷贝，历史记录与当前值共享引用
- 撤销操作本身被记录为新历史
- 直接对 reactive 代理调用 structuredClone 报错
- 没有容量限制，内存不断增长
- 能说明：编辑器场景中通常需要合并连续的输入（例如 500ms 内的修改合并为一条记录），这时用 `flush: 'pre'` 或防抖
