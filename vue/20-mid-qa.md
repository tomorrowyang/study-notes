# Vue 中级 Q&A

> 目标：讲清响应式系统、异步更新队列、虚拟 DOM 与 diff、编译优化、内置组件和生态库的原理，以及常见的坑。

---

## 一、响应式原理

### Q1. Vue 3 的响应式系统是怎么实现的？

- **难度**: 中级
- **标签**: 响应式

**答案**

**核心三步**：

1. **代理**：`reactive(obj)` 返回 `Proxy`，拦截 `get`、`set`、`has`、`deleteProperty`、`ownKeys`
2. **依赖收集（track）**：读取属性时，把**当前正在执行的副作用**（activeEffect，例如组件的渲染函数、computed、watch）记录为该属性的依赖
3. **触发更新（trigger）**：修改属性时，找到该属性的所有依赖，调度执行

**依赖存储结构**（概念模型）：

```text
targetMap: WeakMap<target, Map<key, Dep>>
                      │          └─ Dep：依赖这个属性的副作用集合
                      └─ 原始对象（WeakMap 让对象被回收时依赖也被回收）
```

**组件更新的链路**：

```text
组件挂载 → 创建渲染 effect → 执行渲染函数 → 读取 state.count → track
修改 state.count → trigger → 调度渲染 effect 入队 → 微任务中重新渲染 → patch DOM
```

**Reflect 的作用**：`Reflect.get(target, key, receiver)` 保证 getter 中的 `this` 指向代理对象，从而 getter 内访问的属性也能被追踪。

**口述模板（30–60 秒）**

1. 一句话结论：Vue 3 用 Proxy 拦截读写，读取时收集依赖，修改时触发更新。
2. 关键机制：依赖以 WeakMap、Map、Dep 三层结构存储；副作用包括渲染函数、computed、watch。
3. 对比或注意点：使用 Reflect 保证 getter 中 this 指向代理。
4. 可追问时补充：组件渲染本身就是一个副作用，数据变化时被调度重新执行。

**常见追问**

- 3.4 / 3.6 中依赖结构有变化吗？→ 3.4 起 Dep 与 effect 之间改为双向链表，便于高效地清理和复用；3.6 基于 alien-signals 进一步重构，但对外 API 不变。

---

### Q2. 为什么 Vue 3 用 Proxy 替代 `Object.defineProperty`？

- **难度**: 中级
- **标签**: 响应式

**答案**

| 问题 | `defineProperty`（Vue 2） | `Proxy`（Vue 3） |
|------|---------------------------|------------------|
| 新增属性 | 无法检测，需要 `Vue.set` | 可以检测 |
| 删除属性 | 无法检测，需要 `Vue.delete` | `deleteProperty` 拦截 |
| 数组索引和 length | 无法检测，重写 7 个数组方法 | 可以检测 |
| Map / Set | 不支持 | 支持（通过拦截方法调用） |
| 初始化 | 递归遍历所有属性，大对象初始化慢 | **懒代理**：访问到嵌套对象时才代理 |
| 内存 | 每个属性都有闭包和 Dep | 更省 |

**Proxy 的代价**：不支持 IE11；代理对象与原始对象不相等（`reactive(obj) !== obj`）。

**口述模板（30–60 秒）**

1. 一句话结论：Proxy 可以拦截整个对象的操作，解决了 defineProperty 的检测盲区。
2. 关键机制：能检测新增删除属性、数组索引，支持 Map 和 Set。
3. 对比或注意点：Proxy 是懒代理，访问时才代理嵌套对象，初始化更快。
4. 可追问时补充：代价是放弃了 IE11 支持。

**常见追问**

- Vue 2 为什么只重写了 7 个数组方法？→ push、pop、shift、unshift、splice、sort、reverse 是会修改原数组的方法；索引赋值因为性能原因没有拦截。

---

### Q3. `ref` 是怎么实现的？为什么在模板中不需要 `.value`？

- **难度**: 中级
- **标签**: 响应式

**答案**

```ts
class RefImpl<T> {
  private _value: T
  private _rawValue: T
  dep = new Dep()

  constructor(value: T) {
    this._rawValue = value
    // 对象值转为 reactive，基本类型保持原样
    this._value = isObject(value) ? reactive(value) : value
  }

  get value() {
    this.dep.track()
    return this._value
  }

  set value(newValue) {
    if (hasChanged(newValue, this._rawValue)) {
      this._rawValue = newValue
      this._value = isObject(newValue) ? reactive(newValue) : newValue
      this.dep.trigger()
    }
  }
}
```

**自动解包**：

- 模板中：setup 返回的对象被 `proxyRefs` 包装，访问顶层 ref 时自动返回 `.value`
- reactive 对象中的 ref 属性也会自动解包：`reactive({ count: ref(0) }).count` 直接得到数字
- **不会解包的情况**：数组和 Map 中的 ref；模板中非顶层的 ref（例如 `obj.count` 中 count 是 ref 时需要 `.value`，除非在插值中是最终值）

**口述模板（30–60 秒）**

1. 一句话结论：ref 是一个带有 value getter 和 setter 的对象，getter 收集依赖，setter 触发更新。
2. 关键机制：对象值会被转换为 reactive。
3. 对比或注意点：模板中通过 proxyRefs 自动解包顶层 ref；数组中的 ref 不会解包。
4. 可追问时补充：setter 中会比较新旧值，相同不触发。

**常见追问**

- `shallowRef` 和 `ref` 有什么区别？→ shallowRef 只追踪 .value 的替换，不深度代理内部；修改内部属性不会触发更新，需要整体替换或 `triggerRef`。

---

### Q4. `computed` 是怎么实现缓存的？

- **难度**: 中级
- **标签**: 响应式, computed

**答案**

**机制**：

- computed 既是**副作用**（依赖其他响应式数据），又是**依赖源**（被组件或其他 computed 依赖）
- 内部有一个 `dirty` 标记（3.4+ 使用版本号和标记位）
- **惰性**：只有被访问时才计算；依赖没有变化时直接返回缓存的值
- 依赖变化时，**不立即重新计算**，只标记为脏并通知自己的订阅者
- 订阅者（例如组件）重新渲染、再次访问 computed 时才重新计算

**3.4 的优化**：computed 重新计算后，如果结果与之前相同，不会触发依赖它的副作用。

```ts
const count = ref(1)
const isEven = computed(() => count.value % 2 === 0)
// count 从 1 改为 3：isEven 重新计算后仍为 false，3.4+ 不会触发组件更新
```

**口述模板（30–60 秒）**

1. 一句话结论：computed 是惰性求值的，依赖不变就返回缓存。
2. 关键机制：依赖变化时只标记为脏并通知订阅者，被访问时才重新计算。
3. 对比或注意点：3.4 之后计算结果不变时不会触发下游更新。
4. 可追问时补充：computed 既依赖别人，也被别人依赖。

**常见追问**

- computed 中能写异步吗？→ 不应该；computed 必须同步返回值，异步派生用 watch 加 ref，或者 VueUse 的 `computedAsync`。

---

### Q5. `watch` 的 `flush` 选项有什么区别？

- **难度**: 中级
- **标签**: 响应式, 调度

**答案**

| flush | 时机 | 场景 |
|-------|------|------|
| `'pre'`（默认） | 组件更新**之前** | 根据状态变化修改其他状态，修改会在同一次渲染中生效 |
| `'post'` | 组件更新**之后** | 需要访问更新后的 DOM（等价于 `watchPostEffect`） |
| `'sync'` | 数据变化时**同步**执行 | 极少使用；每次修改都执行，没有批处理 |

**批处理**：pre 和 post 的回调都会进入队列并去重，同一个 tick 中多次修改只执行一次回调。

```ts
watch(source, callback, { flush: 'post' })
watchPostEffect(() => {
  console.log(el.value?.offsetHeight)  // 更新后的 DOM
})
```

**口述模板（30–60 秒）**

1. 一句话结论：flush 控制回调执行的时机：组件更新前、更新后、或同步执行。
2. 关键机制：pre 和 post 都进入调度队列，同一 tick 内去重。
3. 对比或注意点：sync 没有批处理，每次修改都执行，有性能风险。
4. 可追问时补充：需要访问更新后的 DOM 时用 post。

**常见追问**

- 为什么默认是 pre？→ 让侦听器中修改的状态能在同一次渲染中生效，避免二次渲染。

---

### Q6. `watch` 中怎么处理异步竞态？

- **难度**: 中级
- **标签**: 响应式, 异步

**答案**

**问题**：快速切换搜索关键词时，先发出的请求可能后返回，覆盖新的结果。

**解决**：在侦听器重新执行或停止前，取消或忽略上一次的异步操作。

```ts
// 3.5+：onWatcherCleanup
watch(keyword, async (value) => {
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort())
  const response = await fetch(`/api/search?q=${value}`, { signal: controller.signal })
  results.value = await response.json()
})

// 旧写法：回调的第三个参数 onCleanup
watch(keyword, async (value, oldValue, onCleanup) => {
  let cancelled = false
  onCleanup(() => {
    cancelled = true
  })
  const data = await search(value)
  if (!cancelled) results.value = data
})
```

**注意**：`onWatcherCleanup` 必须在第一个 `await` **之前**同步调用（与当前侦听器关联）。

**口述模板（30–60 秒）**

1. 一句话结论：用清理函数在下一次执行前取消上一次的请求。
2. 关键机制：onWatcherCleanup 注册的函数会在侦听器重新执行或停止前调用。
3. 对比或注意点：必须在 await 之前调用。
4. 可追问时补充：取消请求用 AbortController，或者用标记位忽略旧结果。

**常见追问**

- 不取消会有什么问题？→ 结果乱序（显示的不是最后一次搜索的结果），以及浪费网络资源。

---

### Q7. `effectScope` 是什么？

- **难度**: 中级
- **标签**: 响应式

**答案**

`effectScope` 收集在其中创建的副作用（computed、watch、watchEffect），可以**一次性停止**。

```ts
const scope = effectScope()

scope.run(() => {
  const doubled = computed(() => counter.value * 2)
  watch(doubled, (value) => console.log(value))
  watchEffect(() => console.log('count', counter.value))
})

// 一次性停止所有副作用
scope.stop()
```

**用途**：

- 组件的 setup 本身就运行在一个 effect scope 中，卸载时自动停止
- 在组件**之外**使用响应式（例如 Pinia 的每个 store、VueUse 的 `createSharedComposable`）时管理副作用的生命周期
- `onScopeDispose`：在当前 scope 停止时执行清理，composable 中可以替代 `onUnmounted`（在组件外也能使用）

**口述模板（30–60 秒）**

1. 一句话结论：effectScope 收集一组副作用，可以一次性停止。
2. 关键机制：组件的 setup 运行在 scope 中，卸载时自动停止。
3. 对比或注意点：在组件外使用响应式时需要用它手动管理生命周期。
4. 可追问时补充：Pinia 的每个 store 都运行在独立的 effect scope 中。

**常见追问**

- `onScopeDispose` 和 `onUnmounted` 的区别？→ onUnmounted 只能在组件中使用；onScopeDispose 在任何 effect scope 中都能使用，更适合通用的 composable。

---

### Q8. 哪些操作会导致丢失响应式？

- **难度**: 中级
- **标签**: 响应式, 坑

**答案**

| 操作 | 原因 | 解决 |
|------|------|------|
| 解构 reactive 对象 | 得到普通值 | `toRefs`、`toRef` |
| 整体替换 reactive 变量 | 模板引用的是旧代理 | 使用 ref，或 `Object.assign` |
| 解构 props（3.5 之前） | 得到普通值 | `toRefs(props)` 或 3.5 的响应式解构 |
| 解构 Pinia store 的状态 | store 是 reactive | `storeToRefs` |
| 把响应式值传给普通函数后保存 | 保存的是值 | 传 ref 或 getter |
| `watch(state.count, ...)` | 传入的是数字 | `watch(() => state.count, ...)` |
| 修改 `markRaw` / `shallowReactive` 深层 | 有意不代理 | 整体替换或 `triggerRef` |
| composable 返回 reactive 对象后解构 | 同第一条 | 返回由 ref 组成的对象 |

**口述模板（30–60 秒）**

1. 一句话结论：丢失响应式的本质是拿到了值的副本，而不是响应式引用。
2. 关键机制：解构、整体替换、把属性值传给 watch 都会丢失追踪。
3. 对比或注意点：用 toRefs、storeToRefs、getter 函数保持响应式。
4. 可追问时补充：composable 返回 ref 组成的对象，调用方可以安全解构。

**常见追问**

- 为什么 `watch(state.count)` 不行？→ 传入时 state.count 已经求值为数字，watch 无法追踪一个数字；getter 函数在每次执行时读取属性，可以被追踪。

---

### Q9. `readonly`、`shallowReactive`、`markRaw`、`toRaw` 分别有什么用？

- **难度**: 中级
- **标签**: 响应式

**答案**

| API | 作用 | 场景 |
|-----|------|------|
| `readonly` | 返回只读代理，修改时警告 | provide 给后代、对外暴露不可修改的状态 |
| `shallowReactive` | 只有第一层是响应式 | 大对象只关心顶层属性 |
| `shallowRef` | 只追踪 `.value` 的替换 | 大列表、不可变数据 |
| `markRaw` | 标记对象永远不被代理 | 第三方实例（图表、地图、编辑器） |
| `toRaw` | 返回代理对应的原始对象 | 临时读取不需要追踪、传给第三方库 |
| `customRef` | 自定义 track 和 trigger 的时机 | 防抖的 ref |

```ts
function useDebouncedRef<T>(value: T, delay = 300) {
  let timer: ReturnType<typeof setTimeout>
  return customRef((track, trigger) => ({
    get() {
      track()
      return value
    },
    set(newValue) {
      clearTimeout(timer)
      timer = setTimeout(() => {
        value = newValue
        trigger()
      }, delay)
    },
  }))
}
```

**口述模板（30–60 秒）**

1. 一句话结论：这些 API 用于精细控制响应式的范围和行为。
2. 关键机制：shallow 系列只代理第一层，markRaw 跳过代理，readonly 禁止修改。
3. 对比或注意点：性能敏感的大数据和第三方实例应该避免深度代理。
4. 可追问时补充：customRef 可以实现防抖的 ref。

**常见追问**

- 大量数据使用 `shallowRef` 后怎么更新？→ 整体替换（不可变更新），或者修改后调用 `triggerRef`。

---

## 二、更新机制

### Q10. 修改数据后，DOM 是怎么更新的？

- **难度**: 中级
- **标签**: 更新机制, 调度

**答案**

```text
state.count++
  ↓ Proxy set → trigger
找到依赖 count 的副作用：组件的渲染 effect
  ↓ 不立即执行，调用调度器 scheduler
queueJob(组件更新任务)
  ├─ 按组件 ID 排序（父组件先于子组件更新）
  ├─ 去重（同一组件只入队一次）
  └─ queueFlush：用 Promise.resolve().then(flushJobs) 安排微任务
  ↓ 当前同步代码执行完毕
微任务：flushJobs
  1. 执行 pre 队列（flush: 'pre' 的侦听器）
  2. 执行组件更新：重新执行渲染函数 → 新 VNode → patch → 修改 DOM
  3. 执行 post 队列（flush: 'post' 的侦听器、mounted / updated 钩子）
```

**为什么父组件先更新**：父组件更新时可能修改子组件的 props，子组件随之更新；如果子组件先更新，可能更新两次或使用过期的 props。

**口述模板（30–60 秒）**

1. 一句话结论：数据变化后，组件更新任务被放入去重的队列，在微任务中批量执行。
2. 关键机制：队列按组件 ID 排序，父组件先更新；执行顺序是 pre 侦听器、组件更新、post 回调。
3. 对比或注意点：同一 tick 中多次修改只更新一次。
4. 可追问时补充：nextTick 返回的就是这次刷新的 Promise。

**常见追问**

- 为什么用微任务而不是宏任务？→ 微任务在当前任务结束后、浏览器渲染前执行，用户不会看到中间状态，延迟也最小。

---

### Q11. `nextTick` 的原理是什么？

- **难度**: 中级
- **标签**: 更新机制

**答案**

```ts
const resolvedPromise = Promise.resolve()
let currentFlushPromise: Promise<void> | null = null

function queueFlush() {
  if (!isFlushing && !isFlushPending) {
    isFlushPending = true
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}

export function nextTick(fn?: () => void) {
  const p = currentFlushPromise || resolvedPromise
  return fn ? p.then(fn) : p
}
```

- 有正在等待或进行中的刷新时，`nextTick` 链在这次刷新的 Promise 后面，保证回调在 DOM 更新后执行
- 没有待刷新的任务时，返回一个已 resolve 的 Promise

**Vue 2 的区别**：Vue 2 的 nextTick 有降级策略：Promise → MutationObserver → setImmediate → setTimeout；Vue 3 只使用 Promise。

**口述模板（30–60 秒）**

1. 一句话结论：nextTick 返回当前刷新任务的 Promise，回调会在 DOM 更新后执行。
2. 关键机制：数据变化时用 Promise.then 安排刷新；nextTick 链在这个 Promise 之后。
3. 对比或注意点：Vue 2 有多级降级，Vue 3 只使用 Promise。
4. 可追问时补充：在数据修改之前调用 nextTick，回调可能在 DOM 更新前执行。

**常见追问**

- 先调用 `nextTick` 再修改数据，回调能拿到新 DOM 吗？→ 不一定。nextTick 调用时还没有刷新任务，返回的是已 resolve 的 Promise，回调可能在刷新任务之前执行。

---

## 三、虚拟 DOM 与 diff

### Q12. 什么是虚拟 DOM？有什么优缺点？

- **难度**: 中级
- **标签**: 虚拟 DOM

**答案**

**虚拟 DOM**：用 JS 对象（VNode）描述 DOM 结构。

```ts
const vnode = {
  type: 'div',
  props: { class: 'card', onClick: handler },
  children: [{ type: 'span', children: 'hello' }],
  key: null,
  patchFlag: 0,
  el: null,  // 挂载后指向真实 DOM
}
```

**优点**：

- 声明式编程：开发者只描述状态对应的 UI，框架计算最小变更
- **跨平台**：同一套 VNode 可以渲染到 DOM、Canvas、小程序、原生（自定义渲染器 `createRenderer`）
- 批量更新，减少直接操作 DOM 的心智负担

**缺点**：

- 有额外开销：创建 VNode、diff 都需要时间和内存
- 「虚拟 DOM 比直接操作 DOM 快」是误解：精确的手动 DOM 操作一定更快，虚拟 DOM 是在可维护性和性能之间取得平衡

**这也是 Vapor Mode 的出发点**：编译器已经知道模板中哪些部分是动态的，可以直接生成精确更新 DOM 的代码，跳过虚拟 DOM。

**口述模板（30–60 秒）**

1. 一句话结论：虚拟 DOM 用 JS 对象描述 UI，通过 diff 计算最小的 DOM 变更。
2. 关键机制：状态变化时生成新的 VNode 树，与旧树比较后 patch。
3. 对比或注意点：虚拟 DOM 并不比精确的 DOM 操作快，它的价值是声明式和跨平台。
4. 可追问时补充：Vapor Mode 通过编译跳过虚拟 DOM。

**常见追问**

- Vue 为什么在有响应式的情况下还需要虚拟 DOM？→ Vue 的响应式是组件级的粒度：知道哪个组件需要更新，组件内部再用虚拟 DOM diff 找出具体变化，平衡了依赖追踪的开销和更新的精度。

---

### Q13. Vue 3 的 diff 算法是怎样的？

- **难度**: 中级
- **标签**: diff

**答案**

**整体策略**：同层比较；type 或 key 不同就认为是不同的节点，直接替换。

**带 key 的子节点数组的 diff（`patchKeyedChildren`）**：

```text
旧：a b [c d e] f g
新：a b [e c d h] f g

1. 从头部开始比较相同的节点：a、b 相同，patch 后继续
2. 从尾部开始比较相同的节点：g、f 相同
3. 如果旧节点处理完、新节点还有剩余：挂载剩余的新节点
4. 如果新节点处理完、旧节点还有剩余：卸载剩余的旧节点
5. 中间乱序部分 [c d e] → [e c d h]：
   a. 为新节点建立 key → 索引的 Map
   b. 遍历旧节点：在新节点中找不到的卸载；找到的 patch，并记录新旧索引的对应关系
   c. 计算对应关系数组的「最长递增子序列」（LIS）：c、d 的相对顺序不变
   d. 从后向前遍历新节点：不在 LIS 中的节点移动，没有对应旧节点的（h）新建
   结果：只移动 e，新建 h
```

**最长递增子序列的作用**：找出相对顺序没有变化的最多节点，这些节点不需要移动，从而**最小化 DOM 移动次数**。

**口述模板（30–60 秒）**

1. 一句话结论：Vue 3 的 diff 先处理首尾相同的节点，再对中间乱序部分用最长递增子序列最小化移动。
2. 关键机制：新节点建 key 映射，旧节点找对应关系，不存在的卸载，LIS 中的节点不动，其他节点移动或新建。
3. 对比或注意点：Vue 2 使用双端比较（四个指针），Vue 3 改为预处理加 LIS。
4. 可追问时补充：编译优化让大部分情况下只需要 diff 动态节点。

**常见追问**

- 最长递增子序列的时间复杂度？→ 贪心 + 二分查找，O(n log n)，并通过前驱数组回溯得到具体序列。

---

### Q14. Vue 2 和 Vue 3 的 diff 有什么区别？

- **难度**: 中级
- **标签**: diff

**答案**

| 方面 | Vue 2 | Vue 3 |
|------|-------|-------|
| 子节点 diff | 双端比较：新旧首尾四个指针，四种比较 + key 查找 | 首尾预处理 + 最长递增子序列 |
| 静态节点 | 有静态标记，但仍要遍历 | 静态提升，完全跳过 |
| 比较范围 | 递归比较整棵树 | Block Tree：只比较动态节点 |
| 属性比较 | 比较所有属性 | PatchFlag：只比较动态的属性 |
| 事件处理器 | 每次更新都比较 | 缓存事件处理器 |

**本质区别**：Vue 3 把大量优化放在**编译阶段**，运行时 diff 的工作量从「模板大小」降低到「动态节点数量」。

**口述模板（30–60 秒）**

1. 一句话结论：Vue 3 的 diff 算法改用最长递增子序列，并结合编译优化大幅减少比较范围。
2. 关键机制：Block Tree 只比较动态节点，PatchFlag 只比较动态属性。
3. 对比或注意点：Vue 2 的双端比较在运行时完成，需要遍历整棵树。
4. 可追问时补充：编译优化才是 Vue 3 性能提升的主要来源。

**常见追问**

- 双端比较有什么问题？→ 不能保证最少的移动次数；而且需要遍历所有节点。

---

### Q15. 组件的 patch 过程是怎样的？

- **难度**: 中级
- **标签**: 渲染器

**答案**

```text
patch(oldVNode, newVNode, container)
 ├─ old 为 null → 挂载（mount）
 ├─ type 或 key 不同 → 卸载旧节点，挂载新节点
 └─ 根据 type 分类处理：
     ├─ 文本 / 注释 → 直接更新内容
     ├─ Fragment → patch 子节点
     ├─ 元素 → patchElement
     │    ├─ 有 dynamicChildren（Block）→ 只 patch 动态子节点
     │    ├─ 否则 → patchChildren（完整 diff）
     │    └─ 根据 patchFlag 只更新动态的 class / style / props
     ├─ 组件 → updateComponent
     │    ├─ shouldUpdateComponent：比较 props、插槽是否变化
     │    ├─ 不需要更新 → 复用实例
     │    └─ 需要 → 更新 props，执行组件的渲染 effect
     └─ Teleport / Suspense / KeepAlive → 各自的 process 方法
```

**组件更新的特点**：父组件重新渲染时，子组件只有在 **props 或插槽变化**时才更新。这与 React 默认重新渲染所有子组件不同，Vue 不需要 `memo`。

**口述模板（30–60 秒）**

1. 一句话结论：patch 根据节点类型分别处理，类型或 key 不同直接替换。
2. 关键机制：元素节点根据 patchFlag 和 dynamicChildren 只更新动态部分；组件比较 props 和插槽决定是否更新。
3. 对比或注意点：Vue 的子组件只在 props 变化时更新，不需要手动 memo。
4. 可追问时补充：Teleport、Suspense、KeepAlive 有各自的处理逻辑。

**常见追问**

- 传给子组件的内联对象会导致子组件更新吗？→ 会，每次渲染都是新对象，props 比较不相等；Vue 的编译优化无法消除这种情况。

---

## 四、编译优化

### Q16. Vue 的模板编译流程是怎样的？

- **难度**: 中级
- **标签**: 编译

**答案**

```text
模板字符串
  ↓ parse（3.4 重写为基于状态机的 tokenizer，速度约 2 倍）
模板 AST
  ↓ transform（应用各种转换：v-if、v-for、v-model、事件、静态分析）
JS AST（带有 PatchFlag、静态提升、Block 信息）
  ↓ codegen
渲染函数代码
```

示例：

```vue
<div class="card">
  <span>静态文本</span>
  <p :class="active">{{ message }}</p>
</div>
```

编译结果（简化）：

```js
const _hoisted_1 = { class: 'card' }
const _hoisted_2 = /*#__PURE__*/ _createElementVNode('span', null, '静态文本', -1 /* HOISTED */)

function render(_ctx) {
  return (_openBlock(), _createElementBlock('div', _hoisted_1, [
    _hoisted_2,
    _createElementVNode('p', { class: _normalizeClass(_ctx.active) }, _toDisplayString(_ctx.message), 3 /* TEXT, CLASS */),
  ]))
}
```

**编译时机**：

- 使用 Vite / webpack 构建时预编译（推荐），运行时不需要编译器，体积更小
- 运行时编译需要完整版的 Vue（包含编译器），例如在 HTML 中直接写模板

可以在 [Vue Template Explorer](https://template-explorer.vuejs.org/) 查看编译结果。

**口述模板（30–60 秒）**

1. 一句话结论：模板经过解析、转换、代码生成三步编译为渲染函数。
2. 关键机制：转换阶段做静态分析，标记 PatchFlag、提升静态节点、建立 Block。
3. 对比或注意点：构建时预编译，运行时不需要编译器。
4. 可追问时补充：3.4 重写了解析器，速度提升约一倍。

**常见追问**

- 渲染函数和 JSX 能享受编译优化吗？→ 手写渲染函数不能；Vue 的 JSX 插件能做部分优化，但不如模板完整。

---

### Q17. 什么是 PatchFlag 和 Block Tree？

- **难度**: 中级
- **标签**: 编译优化

**答案**

**PatchFlag**：编译时给动态节点打上标记，说明它**哪些部分是动态的**，运行时只比较这些部分。

| 标记 | 值 | 含义 |
|------|-----|------|
| TEXT | 1 | 动态文本 |
| CLASS | 2 | 动态 class |
| STYLE | 4 | 动态 style |
| PROPS | 8 | 动态属性（附带动态属性名列表） |
| FULL_PROPS | 16 | 有动态 key 的属性，需要完整比较 |
| NEED_HYDRATION | 32 | 需要 hydration 的事件监听 |
| STABLE_FRAGMENT | 64 | 子节点顺序不变的 Fragment |
| KEYED_FRAGMENT / UNKEYED_FRAGMENT | 128 / 256 | v-for 的 Fragment |
| HOISTED | -1 | 静态节点 |
| BAIL | -2 | 退出优化，完整 diff |

使用位运算组合：`TEXT | CLASS = 3`。

**Block Tree**：

- **Block** 是一个特殊的 VNode，它把内部所有的**动态后代节点**收集到一个扁平数组 `dynamicChildren` 中
- 更新时直接遍历这个数组，**跳过所有静态节点**，不管它们嵌套多深
- 结构可能变化的节点（`v-if`、`v-for`）会创建新的 Block，因为这些节点的动态子节点数量和顺序可能变化

**效果**：diff 的复杂度从与模板大小相关，降低到与动态节点数量相关。

**口述模板（30–60 秒）**

1. 一句话结论：PatchFlag 标记节点的哪些部分是动态的，Block Tree 把动态节点收集为扁平数组。
2. 关键机制：更新时只遍历动态节点数组，只比较标记的动态部分。
3. 对比或注意点：v-if 和 v-for 会创建新的 Block，因为结构可能变化。
4. 可追问时补充：diff 的工作量从模板大小降低到动态节点数量。

**常见追问**

- 为什么 v-if 要单独作为 Block？→ 不同分支的动态节点不同，如果合并在父 Block 的扁平数组中，新旧数组无法一一对应。

---

### Q18. 静态提升和事件缓存是什么？

- **难度**: 中级
- **标签**: 编译优化

**答案**

**静态提升**：

- 完全静态的节点（以及静态的属性对象）被提升到渲染函数**外部**，只创建一次，每次渲染复用
- 连续多个静态节点会被合并为一个**静态字符串**（`createStaticVNode`），通过 innerHTML 一次性创建

**事件缓存**：

```vue
<button @click="count++">+1</button>
```

```js
// 没有缓存：每次渲染都创建新的函数，button 的 props 每次都不相等
// 有缓存：
_createElementVNode('button', {
  onClick: _cache[0] || (_cache[0] = ($event) => _ctx.count++),
}, '+1')
```

缓存后函数引用不变，这个节点可以被视为没有动态属性；传给子组件时也不会因为函数引用变化而导致子组件更新。

**其他**：`v-once` 只渲染一次（缓存 VNode）；`v-memo` 依赖不变时复用 VNode。

**口述模板（30–60 秒）**

1. 一句话结论：静态提升让静态节点只创建一次，事件缓存让处理函数的引用保持不变。
2. 关键机制：静态节点被提升到渲染函数外，连续的静态节点合并为字符串。
3. 对比或注意点：事件缓存避免了子组件因函数引用变化而更新。
4. 可追问时补充：v-once 和 v-memo 是手动的缓存手段。

**常见追问**

- React 中对应的手段是什么？→ React 需要手动使用 `useCallback`、`useMemo`、`memo`；React Compiler 可以自动完成类似的优化。

---

### Q19. `v-memo` 怎么使用？

- **难度**: 中级
- **标签**: 编译优化, 性能

**答案**

```vue
<div v-for="item in list" :key="item.id" v-memo="[item.id === selectedId]">
  <p>ID: {{ item.id }} - 选中: {{ item.id === selectedId }}</p>
  <ExpensiveRow :item="item" />
</div>
```

- 依赖数组中的值都不变时，跳过该节点及其子树的更新（复用上次的 VNode）
- 典型场景：大列表中切换选中项时，只有两项（旧选中和新选中）需要更新，其余项直接跳过
- `v-memo="[]"` 等价于 `v-once`

**注意**：

- 依赖数组必须包含所有可能影响渲染的值，否则会显示过期内容
- 一般不需要使用，只在**确实测量到性能问题**的大列表中使用

**口述模板（30–60 秒）**

1. 一句话结论：v-memo 在依赖不变时跳过节点子树的更新。
2. 关键机制：依赖数组中的值都相同时，复用上次的 VNode。
3. 对比或注意点：依赖不完整会导致显示过期内容。
4. 可追问时补充：适用于大列表中只有少数项变化的场景。

**常见追问**

- 和 `computed` 的缓存有什么区别？→ computed 缓存数据；v-memo 缓存渲染结果（VNode）。

---

## 五、内置组件原理

### Q20. `KeepAlive` 的原理是什么？

- **难度**: 中级
- **标签**: 内置组件

**答案**

**核心机制**：

- 内部维护 `cache: Map<key, VNode>` 和 `keys: Set<key>`
- **渲染时**：取默认插槽的组件 VNode，用 key（或组件类型）作为缓存键
  - 命中缓存：复用缓存中的组件实例，并把该 key 移到 Set 的末尾（**LRU**）
  - 未命中：加入缓存；超过 `max` 时删除 Set 中第一个（最久未使用的）
- 给 VNode 打上标记（`COMPONENT_SHOULD_KEEP_ALIVE`、`COMPONENT_KEPT_ALIVE`）
- **卸载时**：渲染器看到标记，不销毁组件，而是调用 `deactivate`，把 DOM 移动到一个**隐藏的容器**（一个不在文档中的 div）中
- **再次激活**：调用 `activate`，把 DOM 移回原位置，执行 `onActivated`

**include / exclude**：根据组件的 name 判断是否缓存；变化时清理不再匹配的缓存。

**口述模板（30–60 秒）**

1. 一句话结论：KeepAlive 用 Map 缓存组件 VNode，卸载时把 DOM 移到隐藏容器而不是销毁。
2. 关键机制：命中缓存时复用实例；用 Set 的顺序实现 LRU 淘汰。
3. 对比或注意点：渲染器根据标记调用 activate 和 deactivate，而不是 mount 和 unmount。
4. 可追问时补充：include 和 exclude 变化时会清理不匹配的缓存。

**常见追问**

- 被缓存的组件中的定时器还在运行吗？→ 在运行；需要在 onDeactivated 中暂停，onActivated 中恢复。

---

### Q21. `Teleport` 的原理是什么？

- **难度**: 中级
- **标签**: 内置组件

**答案**

- Teleport 是渲染器中的特殊类型，有自己的 `process` 方法
- **挂载时**：在原位置插入两个注释节点作为占位锚点；通过 `to` 找到目标容器，把子节点挂载到目标容器中
- **更新时**：子节点 diff 正常进行；如果 `to` 变化，把子节点**移动**到新的目标；`disabled` 变化时在原位置和目标之间移动
- **逻辑关系不变**：VNode 仍在组件树的原位置，所以 props、provide / inject、组件事件都正常

**SSR 注意**：Teleport 的内容在服务端渲染时会被收集到 `ctx.teleports` 中，需要手动注入到 HTML 的目标位置。

**口述模板（30–60 秒）**

1. 一句话结论：Teleport 在原位置留下锚点，把子节点挂载到目标容器中。
2. 关键机制：VNode 仍在组件树的原位置，只改变了真实 DOM 的挂载位置。
3. 对比或注意点：to 或 disabled 变化时会移动子节点。
4. 可追问时补充：SSR 时需要手动把 Teleport 内容注入到目标位置。

**常见追问**

- 为什么弹窗要 Teleport 到 body？→ 避免被祖先元素的 overflow、transform、z-index 等创建的层叠上下文和包含块影响。

---

## 六、组件机制

### Q22. 组件实例的创建和挂载过程是怎样的？

- **难度**: 中级
- **标签**: 组件, 渲染器

**答案**

```text
mountComponent(vnode)
  1. createComponentInstance：创建实例对象（props、slots、setupState、provides、生命周期数组等）
  2. setupComponent：
     - initProps：解析 props（校验、默认值），转为 shallowReactive
     - initSlots
     - 执行 setup()（期间设置 currentInstance，让生命周期钩子能注册到当前实例）
     - 处理 setup 返回值：函数作为渲染函数；对象用 proxyRefs 包装为 setupState
     - 没有 render 时编译模板（运行时编译版本）
  3. setupRenderEffect：
     - 创建渲染 effect，调度器为 queueJob
     - 首次执行：beforeMount → 执行渲染函数得到子树 VNode → patch 挂载 → mounted（放入 post 队列）
     - 后续更新：beforeUpdate → 重新渲染 → patch(旧子树, 新子树) → updated
```

**currentInstance**：`onMounted` 等钩子内部通过 `currentInstance` 找到当前组件，把回调放入实例的生命周期数组；所以必须在 setup 中同步调用。

**口述模板（30–60 秒）**

1. 一句话结论：组件挂载分为创建实例、执行 setup、创建渲染 effect 三步。
2. 关键机制：setup 执行期间设置 currentInstance，生命周期钩子注册到这个实例上。
3. 对比或注意点：渲染 effect 的调度器是 queueJob，所以更新是异步批量的。
4. 可追问时补充：setup 返回的对象被 proxyRefs 包装，模板中自动解包 ref。

**常见追问**

- 为什么 `await` 之后注册的生命周期钩子不生效？→ await 之后 setup 已经返回，currentInstance 被重置为 null。（`<script setup>` 中顶层 await 会被编译器特殊处理，恢复实例上下文）

---

### Q23. props 的更新是怎么触发子组件更新的？

- **难度**: 中级
- **标签**: 组件, props

**答案**

1. 父组件重新渲染，生成子组件的新 VNode（新的 props）
2. `updateComponent` → `shouldUpdateComponent`：
   - 有 PatchFlag 时只比较动态 props
   - 有动态插槽时一定更新
3. 需要更新时：`updateProps` 修改实例的 props（`shallowReactive`）
4. 子组件的渲染函数读取过这些 props → 触发子组件的渲染 effect
5. 父组件的更新任务中直接执行子组件的更新（`instance.update()`），并从队列中移除子组件的任务，避免重复更新

**props 是浅响应的**：传入对象后修改其内部属性，父组件中该对象本身是 reactive 的话，子组件也会响应（因为是同一个代理对象）。

**口述模板（30–60 秒）**

1. 一句话结论：父组件重新渲染时比较子组件的 props，变化时更新 props 并触发子组件渲染。
2. 关键机制：props 是 shallowReactive，子组件渲染时读取了 props，修改后触发它的渲染 effect。
3. 对比或注意点：父组件更新中直接更新子组件，并去除队列中重复的任务。
4. 可追问时补充：PatchFlag 让 props 比较只针对动态属性。

**常见追问**

- 父组件更新但 props 没变，子组件会更新吗？→ 不会，shouldUpdateComponent 返回 false，复用实例。

---

### Q24. 自定义指令的原理和常见场景是什么？

- **难度**: 中级
- **标签**: 指令

**答案**

```ts
// v-permission：没有权限时移除元素
export const vPermission: Directive<HTMLElement, string> = {
  mounted(el, binding) {
    const auth = useAuthStore()
    if (!auth.permissions.includes(binding.value)) {
      el.parentNode?.removeChild(el)
    }
  },
}

// v-click-outside
export const vClickOutside: Directive<HTMLElement & { _handler?: (e: Event) => void }, () => void> = {
  mounted(el, binding) {
    el._handler = (event) => {
      if (!el.contains(event.target as Node)) binding.value()
    }
    document.addEventListener('click', el._handler)
  },
  unmounted(el) {
    if (el._handler) document.removeEventListener('click', el._handler)
  },
}
```

**钩子**：`created`、`beforeMount`、`mounted`、`beforeUpdate`、`updated`、`beforeUnmount`、`unmounted`；参数 `binding` 包含 `value`、`oldValue`、`arg`、`modifiers`、`instance`。

**原理**：编译器把指令编译为 `withDirectives(vnode, [[dir, value, arg, modifiers]])`，渲染器在元素的对应生命周期中调用指令钩子。

**场景**：自动聚焦、权限控制、点击外部、懒加载图片、防重复点击、拖拽。

**口述模板（30–60 秒）**

1. 一句话结论：自定义指令用于封装底层的 DOM 操作，在元素的生命周期中执行。
2. 关键机制：编译为 withDirectives，渲染器在元素挂载、更新、卸载时调用钩子。
3. 对比或注意点：要在 unmounted 中清理事件监听。
4. 可追问时补充：权限指令移除元素只是体验控制，安全由后端保证。

**常见追问**

- 指令用在组件上会怎样？→ 作用于组件的根元素；多根组件会忽略并警告。

---

## 七、生态原理

### Q25. Vue Router 的导航流程是怎样的？

- **难度**: 中级
- **标签**: Router

**答案**

**完整的导航解析流程**：

1. 导航被触发
2. 失活组件中调用 `beforeRouteLeave`
3. 全局 `beforeEach`
4. 复用组件中调用 `beforeRouteUpdate`
5. 路由配置中的 `beforeEnter`
6. 解析异步路由组件
7. 被激活组件中调用 `beforeRouteEnter`
8. 全局 `beforeResolve`
9. 导航确认
10. 全局 `afterEach`
11. 触发 DOM 更新
12. 调用 `beforeRouteEnter` 中传给 `next` 的回调

**实现原理**：

- `currentRoute` 是一个 `shallowRef`，导航确认后替换，`RouterView` 读取它，所以路由变化会触发重新渲染
- `RouterView` 根据嵌套深度（通过 provide / inject 传递 depth）从 `matched` 数组中取出对应层级的组件
- 守卫被组装为一个 Promise 队列依次执行；任何一个守卫返回 false 或新地址，就中止当前导航
- 新的导航开始时，旧的未完成导航会被取消

**口述模板（30–60 秒）**

1. 一句话结论：导航依次经过离开守卫、全局前置、更新守卫、路由独享、组件进入、全局解析，确认后执行后置钩子。
2. 关键机制：currentRoute 是响应式的，RouterView 根据嵌套深度渲染 matched 中的组件。
3. 对比或注意点：守卫按 Promise 队列执行，返回 false 或新地址会中止导航。
4. 可追问时补充：新导航会取消未完成的旧导航。

**常见追问**

- 怎么实现动态路由（按权限加载菜单）？→ 登录后获取权限，用 `router.addRoute` 添加路由；注意刷新后要在守卫中重新添加，然后 `return to.fullPath` 重新导航。

---

### Q26. Pinia 的原理是什么？

- **难度**: 中级
- **标签**: Pinia

**答案**

- `createPinia()` 创建一个 pinia 实例，内部有一个全局 `effectScope` 和 `state: ref({})`（所有 store 的状态都存放在这里，按 id 索引）
- `app.use(pinia)` 时通过 `app.provide` 注入 pinia，并设置为活动的 pinia
- `defineStore(id, setup)` 返回 `useStore` 函数：
  - 第一次调用时创建 store：在独立的 effectScope 中执行 setup 函数，把返回的 ref 放入 `pinia.state.value[id]`，computed 作为 getter，函数作为 action（包装后支持 `$onAction` 订阅）
  - store 本身是一个 `reactive` 对象
  - 后续调用直接返回缓存的 store（单例）
- **Options Store** 内部也会转换为 Setup Store
- 插件：`pinia.use(plugin)`，在每个 store 创建时执行，可以添加属性、订阅变化（例如持久化插件）

**SSR**：每个请求需要创建新的 pinia 实例，避免状态在用户之间共享；服务端渲染后序列化 `pinia.state.value`，客户端激活时恢复。

**口述模板（30–60 秒）**

1. 一句话结论：Pinia 的每个 store 是在独立 effectScope 中执行 setup 得到的 reactive 对象，首次使用时创建并缓存。
2. 关键机制：所有 store 的状态集中在 pinia.state 中，便于 SSR 序列化和开发工具查看。
3. 对比或注意点：SSR 中每个请求必须创建新的 pinia 实例。
4. 可追问时补充：插件在每个 store 创建时执行，可以实现持久化。

**常见追问**

- 在组件外（例如路由守卫中）怎么使用 store？→ 确保在 `app.use(pinia)` 之后调用 `useStore()`；在守卫函数内部调用而不是在模块顶层。

---

### Q27. Vue 中怎么实现权限控制？

- **难度**: 中级
- **标签**: 工程, 权限

**答案**

| 层级 | 实现 |
|------|------|
| 路由权限 | 登录后根据权限生成路由，`router.addRoute` 动态添加；`beforeEach` 中校验 `meta.permission` |
| 菜单 | 根据有权限的路由生成 |
| 按钮权限 | 自定义指令 `v-permission`，或权限组件 `<Auth code="user:delete">`，或函数 `hasPermission()` |
| 接口权限 | **后端校验**；前端在响应拦截器中统一处理 401（跳转登录）和 403（提示无权限） |

**注意**：

- 前端权限只是**体验层面**的控制，真正的安全必须由后端保证
- 刷新页面时动态路由丢失：在 `beforeEach` 中判断是否已加载权限，没有则加载并添加路由后重新导航
- 退出登录时重置路由（删除动态添加的路由）

**口述模板（30–60 秒）**

1. 一句话结论：权限控制分为路由、菜单、按钮和接口四个层级。
2. 关键机制：动态添加路由，导航守卫校验，按钮用指令或组件控制。
3. 对比或注意点：前端权限只是体验控制，后端必须鉴权。
4. 可追问时补充：刷新时要重新加载权限并添加路由。

**常见追问**

- 用 `v-if` 还是指令移除元素？→ 推荐 v-if 或权限组件，与 Vue 的渲染机制一致；指令直接操作 DOM，权限变化时不会恢复。

---

### Q28. Vue 中如何处理全局错误？

- **难度**: 中级
- **标签**: 工程, 错误处理

**答案**

```ts
app.config.errorHandler = (err, instance, info) => {
  // info 说明错误来源，例如 'render function'、'setup function'
  reportError(err, { component: instance?.$options.name, info })
}

app.config.warnHandler = (msg, instance, trace) => {
  // 只在开发环境生效
}
```

**组件级错误边界**：

```ts
// ErrorBoundary.vue
const error = ref<Error | null>(null)
onErrorCaptured((err) => {
  error.value = err
  return false  // 阻止错误继续向上传播
})
```

**errorHandler 能捕获的**：组件渲染、setup、生命周期、侦听器回调、事件处理函数（通过模板绑定的）、自定义指令钩子中的错误。

**不能捕获的**：`setTimeout` 等异步回调中的错误、未处理的 Promise 拒绝（模板事件处理函数返回的 Promise 除外）→ 需要配合 `window.onerror` 和 `unhandledrejection`。

**口述模板（30–60 秒）**

1. 一句话结论：全局用 app.config.errorHandler，局部用 onErrorCaptured 实现错误边界。
2. 关键机制：Vue 捕获渲染、生命周期、侦听器、事件处理中的错误，并沿组件树向上传播。
3. 对比或注意点：定时器和未处理的 Promise 中的错误需要用 window 级别的监听。
4. 可追问时补充：onErrorCaptured 返回 false 阻止继续传播。

**常见追问**

- 渲染出错时整个应用会白屏吗？→ 未处理的渲染错误可能导致组件树卸载；错误边界可以显示降级 UI。

---

## 八、其他

### Q29. Vue 和 React 的核心设计有什么区别？

- **难度**: 中级
- **标签**: 对比

**答案**

| 方面 | Vue | React |
|------|-----|-------|
| 更新机制 | 响应式：自动追踪依赖，精确更新组件 | 状态变化后重新执行组件函数，默认子组件也重新渲染 |
| 优化方式 | 自动，基本不需要手动优化 | 需要 `memo`、`useMemo`、`useCallback`（React Compiler 可以自动化） |
| 模板 | 模板（也支持 JSX），编译器可做静态优化 | JSX，灵活但编译器难以分析 |
| 状态 | 可变（mutable）数据 | 不可变（immutable）数据 |
| 逻辑复用 | composable（setup 只执行一次） | Hooks（每次渲染都执行，有调用顺序规则） |
| 闭包问题 | 基本没有（setup 只执行一次，通过 ref 访问最新值） | 有过期闭包问题，需要正确设置依赖数组 |
| 生态 | 官方提供 Router、Pinia、Nuxt | 社区方案多样，Next.js 等 |
| 并发渲染 | 没有 | 有（Fiber、可中断渲染、Transition） |

**口述模板（30–60 秒）**

1. 一句话结论：Vue 是响应式驱动的精确更新，React 是状态驱动的重新渲染。
2. 关键机制：Vue 自动追踪依赖，React 需要手动或依靠编译器优化。
3. 对比或注意点：Vue 的 setup 只执行一次，没有 Hooks 的过期闭包问题；React 有并发渲染能力。
4. 可追问时补充：两者在趋同：React Compiler 自动 memo，Vue Vapor Mode 走向更细粒度的编译。

**常见追问**

- composable 和 Hooks 最大的区别？→ setup 只执行一次，composable 不受调用顺序和条件调用的限制；Hooks 每次渲染都执行，必须保持调用顺序。

---

### Q30. Vue 的 SSR 是怎么工作的？

- **难度**: 中级
- **标签**: SSR

**答案**

```text
服务端：
  每个请求创建新的 app 实例（createSSRApp）、router、pinia
  → router.push(url) 并等待就绪
  → renderToString(app) / renderToNodeStream（执行 setup、onServerPrefetch，生成 HTML）
  → 序列化状态（pinia.state）注入到 HTML 中
客户端：
  createSSRApp → 恢复状态 → app.mount('#app')
  → hydration：遍历已有的 DOM，与 VNode 对应，绑定事件，建立响应式，不重新创建 DOM
```

**注意**：

- **每个请求独立的实例**：避免跨请求状态污染（模块级的单例状态会在用户之间共享）
- 服务端只执行 `setup`、`onServerPrefetch`、`beforeCreate` / `created`，**不执行** `onMounted`；访问 `window`、`document` 的代码要放到 `onMounted` 中
- **hydration mismatch**：客户端首次渲染结果与服务端 HTML 不一致（时间、随机数、浏览器专属 API、HTML 结构不合法）

**实际项目**：通常使用 **Nuxt**，它封装了数据获取、路由、状态序列化、部署。

**口述模板（30–60 秒）**

1. 一句话结论：服务端渲染出 HTML 和状态，客户端通过 hydration 接管 DOM。
2. 关键机制：每个请求创建独立的应用实例；服务端不执行 onMounted。
3. 对比或注意点：首次渲染结果必须与服务端一致，否则出现 hydration mismatch。
4. 可追问时补充：实际项目一般使用 Nuxt。

**常见追问**

- 为什么不能有模块级的共享状态？→ Node 服务是长时间运行的，模块只加载一次，所有请求共享同一个模块状态，会导致用户之间的数据泄露。

---

### Q31. Vue 中如何使用 TypeScript？

- **难度**: 中级
- **标签**: TypeScript

**答案**

```vue
<script setup lang="ts" generic="T extends { id: number }">
// 泛型组件（3.3+）
const props = withDefaults(
  defineProps<{
    items: T[]
    selected?: T
    size?: 'small' | 'large'
  }>(),
  { size: 'small' },
)

// 3.5+ 可以用解构默认值替代 withDefaults
// const { items, size = 'small' } = defineProps<{ items: T[]; size?: 'small' | 'large' }>()

const emit = defineEmits<{
  select: [item: T]
  'update:selected': [item: T | undefined]
}>()

defineSlots<{
  default(props: { item: T }): any
}>()

const inputRef = useTemplateRef<HTMLInputElement>('input')
const user = ref<User | null>(null)
</script>
```

**工具**：

- `vue-tsc`：对 `.vue` 文件进行类型检查（构建时运行）
- Volar（Vue - Official 扩展）：编辑器支持
- `InjectionKey<T>`：provide / inject 的类型
- `ComponentPublicInstance`、`InstanceType<typeof Comp>`：组件实例类型

**口述模板（30–60 秒）**

1. 一句话结论：Vue 3 通过 defineProps、defineEmits、defineSlots 的泛型参数实现类型安全。
2. 关键机制：基于类型声明的 props 由编译器转换为运行时声明。
3. 对比或注意点：3.3 支持泛型组件，3.5 支持解构默认值。
4. 可追问时补充：使用 vue-tsc 在构建时做类型检查。

**常见追问**

- `defineProps` 的类型参数能引用外部导入的类型吗？→ 3.3 起支持导入的类型和部分复杂类型，但编译器需要能解析出属性列表，过于复杂的类型（如条件类型）不支持。

---

### Q32. Vue 3.5 有哪些重要的新特性？

- **难度**: 中级
- **标签**: 版本

**答案**

| 特性 | 说明 |
|------|------|
| 响应式重构 | 内存占用减少约 56%，大型深层数组操作更快 |
| **响应式 props 解构** | `const { count = 0 } = defineProps<...>()`，解构变量保持响应式；`watch(() => count)` 需要用 getter |
| `useTemplateRef` | 通过字符串 ID 获取模板引用，不依赖变量同名 |
| `useId` | 生成 SSR 安全的唯一 ID（客户端与服务端一致），用于表单 label 和 ARIA |
| **懒 hydration** | 异步组件的 `hydrate` 选项：`hydrateOnVisible`、`hydrateOnIdle`、`hydrateOnInteraction`、`hydrateOnMediaQuery` |
| `data-allow-mismatch` | 标记允许 hydration 不匹配的元素（例如时间） |
| `onWatcherCleanup` | 在侦听器中注册清理函数 |
| `Teleport defer` | 延迟到同一轮渲染完成后再查找目标 |
| 自定义元素增强 | `useHost`、`useShadowRoot`，`configureApp` |

**口述模板（30–60 秒）**

1. 一句话结论：Vue 3.5 主要在响应式性能、SSR 能力和开发体验上改进。
2. 关键机制：响应式内存大幅减少；props 解构保持响应式；支持懒 hydration。
3. 对比或注意点：解构出的 props 传给 watch 时要用 getter 包装。
4. 可追问时补充：useId 解决 SSR 下 ID 不一致导致的 hydration mismatch。

**常见追问**

- 为什么解构的 props 传给 `watch` 要用 getter？→ 编译器把 `count` 转换为 `props.count`，直接传给 watch 等于传入一个值；`() => count` 编译为 `() => props.count`，可以被追踪。
