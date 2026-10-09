# Vue 高级 Q&A

> 目标：能从源码层面解释响应式和渲染机制，讲清 Vapor Mode、alien-signals、SSR 与 hydration 的细节，能处理大型项目的性能、架构和迁移问题。

---

## 一、响应式深挖

### Q1. Vue 3.4 / 3.5 的响应式系统做了哪些重构？

- **难度**: 高级
- **标签**: 响应式, 源码

**答案**

**3.4 之前的问题**：

- 每次 effect 重新执行前，要先把它从所有 Dep 中**全部移除**，执行时再重新收集，开销大
- computed 的依赖变化后，即使计算结果不变，也会触发下游

**3.4 的改进**：

- 引入**版本号**和**脏检查**：computed 重新计算后结果不变，不触发下游 effect
- 依赖的清理改为增量方式，避免每次全部重建

**3.5 的改进**（内存约 -56%）：

- Dep 与订阅者之间使用**双向链表**（`Link` 节点同时存在于 Dep 的订阅者链表和 effect 的依赖链表中）
- 重新执行 effect 时，按顺序**复用** Link 节点：依赖没有变化时不需要分配新对象；执行结束后删除未被复用的 Link
- **全局版本号**：任何响应式数据变化时递增；computed 检查全局版本号未变化时，直接跳过依赖检查
- computed 在没有订阅者时不再持有依赖（可以被垃圾回收）

**口述模板（30–60 秒）**

1. 一句话结论：3.4 和 3.5 重构了依赖追踪的数据结构，减少了内存分配和不必要的触发。
2. 关键机制：用双向链表连接 Dep 和 effect，重新执行时复用链表节点；用版本号判断是否需要重新计算。
3. 对比或注意点：computed 结果不变时不再触发下游。
4. 可追问时补充：3.6 进一步基于 alien-signals 重构。

**常见追问**

- 为什么双向链表能节省内存？→ 依赖关系通常是稳定的，复用 Link 节点避免了每次执行都创建和销毁 Set 中的条目。

---

### Q2. alien-signals 是什么？Vue 3.6 为什么采用它？

- **难度**: 高级
- **标签**: 响应式, 3.6

**答案**

**alien-signals** 是一个高性能的 Signal（信号）库，作者同时也是 Vue 团队成员。Vue 3.6 把 `@vue/reactivity` 的核心按 alien-signals 的算法重构（`ref`、`reactive`、`computed`、`watch` 的 API 不变）。

**核心思路**（推拉结合）：

- **推（push）**：数据变化时，沿订阅链表向下**只标记**状态（脏 / 待检查），不执行计算
- **拉（pull）**：effect 执行或 computed 被读取时，才向上检查依赖是否真的变化，按需重新计算
- 状态用**位标记**表示，遍历使用迭代而不是递归（避免深层依赖链的栈溢出）
- 尽量少分配对象，链表节点复用

**效果**：更快的更新和更低的内存占用；在第三方基准测试中属于最快的 Signal 实现之一。

**行业背景**：Signal 已成为主流框架的共同方向（Solid、Preact Signals、Angular Signals、Svelte 5 runes），TC39 也有 Signals 提案（Stage 1）。

**口述模板（30–60 秒）**

1. 一句话结论：alien-signals 是高性能的 Signal 实现，Vue 3.6 用它的算法重构了响应式核心。
2. 关键机制：变化时只向下标记状态，读取时才向上检查并按需计算，推拉结合。
3. 对比或注意点：对外 API 不变，是内部实现的升级。
4. 可追问时补充：Signal 是各框架的共同方向，TC39 有相关提案。

**常见追问**

- 为什么不在变化时直接重新计算？→ 中间的 computed 可能不会被读取，提前计算是浪费；推拉结合只计算真正被使用的部分，也能避免菱形依赖导致的重复计算。

---

### Q3. 响应式系统如何处理数组和集合类型？

- **难度**: 高级
- **标签**: 响应式, 源码

**答案**

**数组**：

- `includes`、`indexOf`、`lastIndexOf` 被重写：先在代理数组中查找，找不到再用**原始值**查找（因为数组中的对象元素被访问时会被包装为代理，`arr.includes(rawObj)` 会失败）
- `push`、`pop`、`shift`、`unshift`、`splice` 被重写：执行期间**暂停依赖追踪**，因为这些方法内部会读取 `length`，如果追踪会导致在 effect 中 push 时产生无限循环
- 修改 `length` 会触发被截断的索引的依赖
- 3.5 起对迭代方法（`forEach`、`map`、`filter` 等）做了优化：只追踪一次迭代，而不是每个索引

**Map / Set**：

- Proxy 无法直接拦截内部槽位的方法（`map.get` 需要 `this` 是原始 Map），所以 Vue 使用**专门的 handler**：拦截 `get` 时返回包装后的方法（`get`、`set`、`has`、`add`、`delete`、`clear`、`forEach`、迭代器）
- 包装方法内部调用原始对象的方法，并执行 track / trigger
- 迭代器追踪特殊的 `ITERATE_KEY`；`Map.keys()` 追踪 `MAP_KEY_ITERATE_KEY`（只有新增和删除键才触发）

**口述模板（30–60 秒）**

1. 一句话结论：数组和集合需要特殊处理才能正确追踪依赖。
2. 关键机制：数组的查找方法同时查找代理和原始值；修改长度的方法执行期间暂停追踪，避免无限循环。
3. 对比或注意点：Map 和 Set 的方法依赖内部槽位，必须通过包装方法调用原始对象。
4. 可追问时补充：迭代操作通过特殊的 ITERATE_KEY 追踪。

**常见追问**

- 为什么 `push` 要暂停追踪？→ push 内部会读取 length。如果两个 effect 都对同一个数组 push，读取 length 会让彼此成为依赖，互相触发形成无限循环。

---

### Q4. Vue 的响应式系统有哪些性能陷阱？

- **难度**: 高级
- **标签**: 响应式, 性能

**答案**

| 陷阱 | 原因 | 解决 |
|------|------|------|
| 大型数据深度代理 | 每次访问嵌套属性都经过 Proxy，创建代理对象 | `shallowRef` + 不可变更新，`markRaw` |
| 第三方实例被代理 | 图表、地图、编辑器实例被深度代理，访问慢且可能出错 | `markRaw` 或 `shallowRef` |
| 组件中依赖过多 | 渲染函数中访问大量响应式属性，每个都要追踪 | 拆分组件、使用 computed 汇总 |
| 在 computed 中返回新对象 | 每次计算都返回新引用，下游总是认为变化 | 返回原始值或稳定引用 |
| 在循环中频繁修改响应式数据 | 每次修改都触发 trigger 逻辑 | 先在普通变量中计算，最后一次赋值 |
| 深度 watch 大对象 | `deep: true` 需要遍历整个对象收集依赖 | 指定具体路径，3.5 支持 `deep: 数字` 限制深度 |
| 响应式对象作为 Map 的键或在 Set 中比较 | 代理和原始对象不相等 | 使用 `toRaw` 或 ID |

**口述模板（30–60 秒）**

1. 一句话结论：响应式的主要开销是 Proxy 访问和依赖追踪，数据越大越深开销越大。
2. 关键机制：大型数据用 shallowRef 加不可变更新，第三方实例用 markRaw。
3. 对比或注意点：深度 watch 会遍历整个对象。
4. 可追问时补充：批量修改先在普通变量中计算，最后一次性赋值。

**常见追问**

- 10 万条数据的表格应该怎么存？→ `shallowRef` 存储数组，更新时替换数组；配合虚拟滚动只渲染可见行。

---

## 二、Vapor Mode

### Q5. 什么是 Vapor Mode？原理是什么？

- **难度**: 高级
- **标签**: Vapor, 编译

**答案**

**Vapor Mode** 是 Vue 3.6 引入的新编译模式（截至 2026-10 处于 RC 阶段）：SFC 不再编译为生成 VNode 的渲染函数，而是编译为**直接创建和更新 DOM** 的代码，完全跳过虚拟 DOM。

```vue
<script setup vapor>
const count = ref(0)
</script>

<template>
  <button @click="count++">{{ count }}</button>
</template>
```

编译结果（概念示意）：

```js
const t0 = template('<button> </button>')

function setup() {
  const count = ref(0)
  const button = t0()           // 克隆静态模板
  const text = button.firstChild
  delegate(button, 'click', () => count.value++)
  // 每个动态绑定是一个独立的细粒度 effect
  renderEffect(() => setText(text, count.value))
  return button
}
```

**与虚拟 DOM 模式的区别**：

| 方面 | 虚拟 DOM 模式 | Vapor Mode |
|------|---------------|------------|
| 更新粒度 | 组件级：组件重新渲染 → diff | **绑定级**：每个动态绑定一个 effect，直接更新对应 DOM |
| 运行时 | 需要 VNode、diff、patch | 不需要，运行时体积更小 |
| 内存 | 每次渲染创建 VNode | 不创建 VNode |
| 组件 | 每次更新执行渲染函数 | setup 只执行一次，没有重新渲染 |

**性能**：官方说明在第三方基准中达到与 Solid、Svelte 5 相同的水平。

**口述模板（30–60 秒）**

1. 一句话结论：Vapor Mode 把模板编译为直接操作 DOM 的代码，跳过虚拟 DOM。
2. 关键机制：静态结构通过克隆模板创建，每个动态绑定是一个细粒度的 effect，数据变化时直接更新对应的 DOM。
3. 对比或注意点：更新粒度从组件级变为绑定级，没有 VNode 和 diff 的开销。
4. 可追问时补充：通过 `<script setup vapor>` 选择性启用，可以与虚拟 DOM 组件混用。

**常见追问**

- 为什么 Vue 能做到而 React 很难？→ Vue 有响应式系统可以精确知道哪个数据变化，模板也可以被编译器静态分析；React 是基于重新执行组件函数的模型，JSX 也难以静态分析。

---

### Q6. Vapor Mode 有哪些限制？怎么在项目中采用？

- **难度**: 高级
- **标签**: Vapor, 工程

**答案**

**限制**（以官方发布说明为准）：

- 只支持 `<script setup>` 的 SFC
- 支持现有 API 的**子集**；依赖 VNode 或组件公共实例代理的特性不可用（例如 Options API、`getCurrentInstance()` 的部分用法、渲染函数中操作 VNode）
- 不支持在纯 Vapor 应用中使用 Suspense（可以在虚拟 DOM 的 Suspense 中渲染 Vapor 组件）
- 依赖 VNode 的第三方组件库需要在混合模式下使用

**采用方式**：

1. **混合使用**：现有应用中，通过 `vaporInteropPlugin` 让 Vapor 组件和虚拟 DOM 组件互相嵌套
2. **纯 Vapor 应用**：`createVaporApp`，运行时体积最小，适合新的小型应用
3. **Nuxt 4.6+**：以兼容模式支持，应用根仍是虚拟 DOM，按组件或页面选择 Vapor

**官方建议的场景**：

- 现有应用中**已测量到性能瓶颈**的页面（例如大列表、高频更新的仪表盘）
- 全新的小型应用

**生产建议**：3.6 稳定之前，生产应用继续使用 3.5；对隔离的页面做实验。

**口述模板（30–60 秒）**

1. 一句话结论：Vapor 只支持 script setup，是现有 API 的子集，可以与虚拟 DOM 组件混用。
2. 关键机制：混合模式下通过互操作插件嵌套；纯 Vapor 应用体积最小。
3. 对比或注意点：依赖 VNode 的特性和部分组件库不可用。
4. 可追问时补充：官方建议用于已测量到瓶颈的页面和新的小型应用。

**常见追问**

- 迁移到 Vapor 需要重写组件吗？→ 使用 script setup 和 Composition API 的组件通常只需要加 vapor 属性；需要检查是否使用了不支持的特性。

---

## 三、渲染深挖

### Q7. 最长递增子序列算法在 diff 中是怎么实现的？

- **难度**: 高级
- **标签**: diff, 算法

**答案**

**输入**：`newIndexToOldIndexMap`，下标是新节点在乱序区间中的位置，值是对应旧节点的索引 + 1（0 表示新节点）。

**算法（贪心 + 二分 + 回溯）**：

```ts
function getSequence(arr: number[]): number[] {
  const predecessors = arr.slice()  // 记录每个元素在序列中的前驱
  const result = [0]                // 存放索引，对应的值递增
  for (let i = 0; i < arr.length; i++) {
    const value = arr[i]
    if (value === 0) continue       // 新节点不参与
    const lastIndex = result[result.length - 1]
    if (arr[lastIndex] < value) {
      predecessors[i] = lastIndex
      result.push(i)
      continue
    }
    // 二分查找第一个大于等于 value 的位置并替换（让序列末尾尽量小）
    let low = 0
    let high = result.length - 1
    while (low < high) {
      const mid = (low + high) >> 1
      if (arr[result[mid]] < value) low = mid + 1
      else high = mid
    }
    if (value < arr[result[low]]) {
      if (low > 0) predecessors[i] = result[low - 1]
      result[low] = i
    }
  }
  // 贪心替换会让 result 中的中间值不是真实的序列，需要通过前驱回溯
  let length = result.length
  let last = result[length - 1]
  while (length-- > 0) {
    result[length] = last
    last = predecessors[last]
  }
  return result
}
```

**使用**：从后向前遍历新节点，不在 LIS 中的已有节点需要移动，值为 0 的新建，LIS 中的节点不动。

**优化**：只有当检测到旧节点索引不是单调递增时（`moved = true`）才计算 LIS。

**口述模板（30–60 秒）**

1. 一句话结论：用贪心加二分在 O(n log n) 时间内求最长递增子序列，再通过前驱数组回溯。
2. 关键机制：维护长度为 k 的递增序列的最小末尾，替换时记录前驱。
3. 对比或注意点：贪心得到的 result 不是真实序列，需要回溯修正。
4. 可追问时补充：只有检测到需要移动时才计算 LIS。

**常见追问**

- 为什么从后向前遍历？→ 插入 DOM 需要参照节点（insertBefore）；从后向前时，后面的节点已经处理好，可以作为锚点。

---

### Q8. 自定义渲染器是什么？有什么用？

- **难度**: 高级
- **标签**: 渲染器, 跨平台

**答案**

Vue 的渲染器与平台无关，通过 `createRenderer` 传入平台相关的操作即可渲染到其他目标：

```ts
import { createRenderer } from '@vue/runtime-core'

const { createApp } = createRenderer<CanvasNode, CanvasContainer>({
  createElement: (type) => new CanvasNode(type),
  insert: (child, parent, anchor) => parent.insertBefore(child, anchor),
  remove: (child) => child.parent?.removeChild(child),
  patchProp: (el, key, prevValue, nextValue) => el.setProp(key, nextValue),
  createText: (text) => new CanvasText(text),
  setElementText: (el, text) => el.setText(text),
  parentNode: (node) => node.parent,
  nextSibling: (node) => node.nextSibling,
  createComment: () => new CanvasComment(),
  setText: (node, text) => node.setText(text),
})
```

**`@vue/runtime-dom`** 就是基于 `runtime-core` + DOM 操作实现的。

**应用**：

- 渲染到 Canvas / WebGL（TresJS 渲染 Three.js 场景）
- 小程序（uni-app 等跨端框架）
- 终端、PDF、原生（NativeScript-Vue）

**口述模板（30–60 秒）**

1. 一句话结论：自定义渲染器让 Vue 的组件和响应式系统可以渲染到任意平台。
2. 关键机制：runtime-core 与平台无关，通过 createRenderer 注入节点的创建、插入、属性更新等操作。
3. 对比或注意点：runtime-dom 就是 DOM 平台的实现。
4. 可追问时补充：Three.js、小程序等都有基于自定义渲染器的方案。

**常见追问**

- 这是虚拟 DOM 的优势吗？→ 是的，VNode 是平台无关的中间表示，这是虚拟 DOM 跨平台价值的体现。

---

## 四、SSR 与 hydration

### Q9. hydration 的过程和 mismatch 的排查方法？

- **难度**: 高级
- **标签**: SSR, hydration

**答案**

**hydration 过程**：

- 客户端用 `createSSRApp` 挂载时，渲染器以 hydration 模式运行
- 递归遍历 VNode 树，与已有的 DOM 节点**按顺序对应**：
  - 类型匹配：复用 DOM，把 `vnode.el` 指向它，绑定事件监听
  - 不匹配：开发环境警告；**修复**：删除不匹配的 DOM，客户端重新创建
- 建立组件的响应式和渲染 effect

**常见 mismatch 原因**：

| 原因 | 解决 |
|------|------|
| 时间、随机数 | 服务端生成后序列化，或放到 onMounted；3.5 `data-allow-mismatch` |
| 访问 `window`、`localStorage`、屏幕尺寸 | 在 onMounted 中读取；使用 `<ClientOnly>` |
| 不合法的 HTML 嵌套（`<p>` 中嵌套 `<div>`） | 浏览器解析时会修正结构，导致 DOM 与 VNode 不对应 |
| 自增 ID 不一致 | 3.5 `useId()` |
| 服务端和客户端数据不一致 | 确保状态正确序列化和恢复 |
| 浏览器扩展修改了 DOM | 无法控制，避免影响关键结构 |

**排查**：开发环境的警告会说明不匹配的节点和内容；生产环境可以开启 `__VUE_PROD_HYDRATION_MISMATCH_DETAILS__` 获取详情。

**口述模板（30–60 秒）**

1. 一句话结论：hydration 按顺序把 VNode 与已有的 DOM 对应，复用 DOM 并绑定事件。
2. 关键机制：不匹配时会删除 DOM 在客户端重新创建，影响性能并可能闪烁。
3. 对比或注意点：常见原因是时间随机数、浏览器专属 API、不合法的 HTML 嵌套。
4. 可追问时补充：3.5 提供 useId 和 data-allow-mismatch 解决部分问题。

**常见追问**

- 为什么不合法的 HTML 嵌套会导致 mismatch？→ 服务端输出的是字符串，浏览器解析时会按 HTML 规范自动修正（例如关闭 p 标签），实际 DOM 结构与 VNode 不一致。

---

### Q10. Nuxt 的数据获取是怎么避免重复请求的？

- **难度**: 高级
- **标签**: Nuxt, SSR

**答案**

```vue
<script setup lang="ts">
const route = useRoute()
const { data, status, error, refresh } = await useFetch(`/api/products/${route.params.id}`, {
  key: `product-${route.params.id}`,
  pick: ['id', 'name', 'price'],  // 只序列化需要的字段，减小 payload
})
</script>
```

**机制**：

1. 服务端渲染时执行 `useFetch` / `useAsyncData`，结果按 **key** 存入 `nuxtApp.payload.data`
2. payload 被序列化到 HTML 中（或单独的 payload 文件）
3. 客户端 hydration 时，相同 key 的 `useFetch` 直接从 payload 中读取，**不再发起请求**
4. 客户端导航时才在客户端请求

**注意**：

- `$fetch` 直接在 setup 中调用会导致**服务端和客户端各请求一次**
- 数据很大时用 `pick` / `transform` 减小 payload
- `lazy: true` 不阻塞导航；`server: false` 只在客户端请求

**Nuxt 4 的改进**：数据默认使用 `shallowRef`；相同 key 的多个调用共享数据；组件卸载时自动清理数据。

**口述模板（30–60 秒）**

1. 一句话结论：Nuxt 在服务端获取数据后序列化到 payload，客户端按 key 复用，避免重复请求。
2. 关键机制：useFetch 和 useAsyncData 按 key 存储结果。
3. 对比或注意点：直接用 $fetch 会导致服务端和客户端各请求一次。
4. 可追问时补充：用 pick 减小 payload 的体积。

**常见追问**

- payload 过大有什么影响？→ HTML 体积变大，影响传输和解析；序列化的数据还需要在客户端反序列化，占用主线程。

---

### Q11. 懒 hydration 和流式 SSR 能带来什么？

- **难度**: 高级
- **标签**: SSR, 性能

**答案**

**全量 hydration 的问题**：页面所有组件的 JS 都要下载并执行 hydration，交互越复杂的页面，可交互时间越晚，INP 越差。

**懒 hydration（Vue 3.5）**：

```ts
const Comments = defineAsyncComponent({
  loader: () => import('./Comments.vue'),
  hydrate: hydrateOnVisible({ rootMargin: '100px' }),
})
```

- `hydrateOnIdle`：浏览器空闲时
- `hydrateOnVisible`：进入视口时
- `hydrateOnInteraction('click')`：用户交互时（交互事件会在 hydration 后重放）
- `hydrateOnMediaQuery`：媒体查询匹配时

服务端仍然完整渲染 HTML（SEO 和首屏不受影响），只推迟客户端 JS 的加载和执行。Nuxt 提供 `hydrate-on-visible` 等组件属性形式的封装。

**流式 SSR**：`renderToWebStream` / `renderToNodeStream` 边渲染边发送 HTML，浏览器提前开始解析和加载资源，降低 TTFB 和 FCP。Nuxt 4.5 引入了实验性的 SSR 流式模式。

**孤岛（Islands）**：Nuxt 的服务端组件（`.server.vue`、`<NuxtIsland>`）只在服务端渲染，不发送 JS。

**口述模板（30–60 秒）**

1. 一句话结论：懒 hydration 推迟非关键组件的 JS 执行，流式 SSR 让 HTML 更早到达。
2. 关键机制：服务端完整渲染，客户端按可见、空闲、交互等时机才 hydration。
3. 对比或注意点：SEO 和首屏不受影响，改善的是可交互时间和 INP。
4. 可追问时补充：Nuxt 的服务端组件可以完全不发送 JS。

**常见追问**

- 和 React Server Components 有什么区别？→ RSC 的服务端组件永远不发送代码到客户端，客户端组件按需 hydration；Vue 的懒 hydration 仍然会在某个时机发送并执行组件代码。

---

## 五、性能优化

### Q12. 大列表和大表格怎么优化？

- **难度**: 高级
- **标签**: 性能, 列表

**答案**

**分层优化**：

1. **数据层**：`shallowRef` 存储，不可变更新；避免对每行数据深度代理
2. **渲染层**：
   - **虚拟滚动**：只渲染可见区域的行（`@tanstack/vue-virtual`、`vue-virtual-scroller`），处理动态行高
   - 行组件拆分，每行是独立组件，单行变化只更新该行
   - `v-memo` 让未变化的行跳过更新
   - 单元格中避免复杂的组件嵌套，必要时使用函数式组件或纯文本渲染
3. **交互层**：
   - 筛选、排序、聚合放到 Web Worker
   - 输入防抖
   - 滚动时暂停非关键更新
4. **超大规模**：Canvas 渲染表格（例如在线表格产品），DOM 只用于编辑态

**口述模板（30–60 秒）**

1. 一句话结论：大列表从数据、渲染、交互三个层面优化，核心是虚拟滚动和减少响应式开销。
2. 关键机制：shallowRef 避免深度代理，虚拟滚动只渲染可见行，v-memo 跳过未变化的行。
3. 对比或注意点：计算密集的筛选排序放到 Worker。
4. 可追问时补充：超大规模的表格可以使用 Canvas 渲染。

**常见追问**

- 虚拟滚动中行高不固定怎么办？→ 先用估算高度，渲染后用 ResizeObserver 测量实际高度并缓存，更新位置索引。

---

### Q13. 怎么排查 Vue 应用的性能问题？

- **难度**: 高级
- **标签**: 性能, 排查

**答案**

**工具**：

- **Vue DevTools**：Performance / Timeline 面板查看组件渲染次数和耗时；组件树中查看哪些组件频繁更新
- `app.config.performance = true`：在浏览器 Performance 面板中显示组件的 init、render、patch 耗时
- 浏览器 Performance 面板：长任务、布局、GC
- `onRenderTracked` / `onRenderTriggered`：调试**是哪个依赖**触发了组件更新

```ts
onRenderTriggered((event) => {
  console.log('触发更新', event.key, event.target, event.type)
})
```

**常见问题**：

| 现象 | 原因 |
|------|------|
| 一个输入导致整页更新 | 状态放在了太高的层级，或者大组件没有拆分 |
| 子组件频繁更新 | 传入内联对象 / 函数，每次都是新引用 |
| 页面切换越来越慢 | 内存泄漏：未清理的监听器、定时器、第三方实例、KeepAlive 缓存过多 |
| 首屏慢 | 包体积大，没有懒加载 |
| 滚动卡顿 | 列表没有虚拟化，或者滚动监听中有大量计算 |

**口述模板（30–60 秒）**

1. 一句话结论：用 Vue DevTools 定位频繁更新的组件，用 onRenderTriggered 找到触发更新的依赖。
2. 关键机制：开启 performance 选项可以在浏览器面板看到组件耗时。
3. 对比或注意点：常见原因是状态层级过高、内联对象、内存泄漏。
4. 可追问时补充：浏览器 Performance 面板分析长任务和布局。

**常见追问**

- 怎么排查内存泄漏？→ 反复进出页面后拍摄堆快照对比，查找 Detached 的 DOM 和未释放的组件实例，检查事件监听和第三方实例的销毁。

---

### Q14. Vue 应用的打包体积怎么优化？

- **难度**: 高级
- **标签**: 性能, 构建

**答案**

- **路由懒加载**和**异步组件**：按页面和重组件拆分
- **组件库按需引入**：`unplugin-vue-components` 自动按需导入组件和样式
- **Tree-shaking**：Vue 3 的 API 都是具名导出，未使用的 API（例如 Transition、KeepAlive）不会打包
- **编译时标志**：
  - `__VUE_OPTIONS_API__: false`：不使用 Options API 时移除相关代码
  - `__VUE_PROD_DEVTOOLS__: false`
- 使用运行时版本（不包含模板编译器），构建时预编译模板
- 依赖分析：`rollup-plugin-visualizer` 找到大依赖（例如完整的 lodash、moment）
- **Vapor Mode**（纯 Vapor 应用）：运行时更小

**口述模板（30–60 秒）**

1. 一句话结论：通过懒加载、按需引入、tree-shaking 和编译时标志减小体积。
2. 关键机制：Vue 3 的 API 具名导出，未使用的会被摇掉。
3. 对比或注意点：关闭 Options API 可以进一步减小运行时体积。
4. 可追问时补充：用可视化工具分析依赖，替换大体积的库。

**常见追问**

- 为什么要使用运行时版本？→ 模板在构建时已经编译，运行时不需要编译器，可以减少约 14KB（gzip）。

---

## 六、架构与工程

### Q15. 大型 Vue 项目的状态管理怎么设计？

- **难度**: 高级
- **标签**: 架构, 状态管理

**答案**

**状态分类**：

| 类型 | 方案 |
|------|------|
| 服务端状态（接口数据） | **TanStack Query（Vue Query）** 或 Pinia Colada：缓存、去重、重试、后台刷新、失效 |
| 全局客户端状态（用户、权限、主题） | Pinia |
| 页面 / 功能状态 | 组件状态，或功能级的 composable / store |
| 组件树上下文（表单、向导） | provide / inject |
| URL 状态（筛选、分页、标签页） | 路由 query |
| 表单状态 | 表单库（VeeValidate、FormKit） |

**原则**：

- 服务端状态与客户端状态分离：不要把所有接口数据都放进 Pinia 手动管理缓存
- 状态放在**尽量低**的层级
- store 按**领域**拆分，避免巨型 store
- store 之间的依赖保持单向，避免循环引用

**口述模板（30–60 秒）**

1. 一句话结论：按状态类型选择方案：服务端状态用 Vue Query，全局客户端状态用 Pinia，局部状态留在组件。
2. 关键机制：Vue Query 处理缓存、去重、刷新，减少手写的加载和缓存逻辑。
3. 对比或注意点：不要把所有数据都放进全局 store。
4. 可追问时补充：可分享的筛选条件放到 URL 中。

**常见追问**

- 为什么不把接口数据都放 Pinia？→ 需要手写加载状态、缓存失效、去重、重试、刷新等逻辑，而这些正是服务端状态库已经解决的问题。

---

### Q16. 怎么设计一个高质量的 Vue 组件库？

- **难度**: 高级
- **标签**: 架构, 组件库

**答案**

**API 设计**：

- 一致的命名和 props 约定（`size`、`disabled`、`v-model`）
- 受控与非受控：支持 `v-model`，同时在不传时内部维护状态
- 灵活的定制：插槽（作用域插槽）、`$attrs` 透传、`class` / `style`
- 完整的 TypeScript 类型：泛型组件、类型化插槽和事件

**技术实现**：

- **Headless 组件**：逻辑与样式分离（参考 Reka UI / Radix Vue），上层再提供默认样式
- 复合组件通信：`provide` / `inject`（Form 与 FormItem、Tabs 与 TabPane）
- 样式：CSS 变量支持主题定制；BEM 或原子化命名
- 无障碍：键盘导航、焦点管理、ARIA 属性
- SSR 兼容：避免在 setup 中访问 window，使用 `useId`

**工程化**：

- 按需引入：每个组件独立入口，`sideEffects` 配置，提供自动导入解析器
- 文档：VitePress + 交互示例
- 测试：Vitest + Vue Test Utils 单元测试，Playwright 视觉回归
- 版本管理：Changesets，语义化版本
- Monorepo：组件、工具函数、主题分包

**口述模板（30–60 秒）**

1. 一句话结论：组件库要在 API 一致性、可定制性、类型、无障碍和按需引入上做好。
2. 关键机制：Headless 架构分离逻辑和样式；复合组件通过 provide / inject 通信。
3. 对比或注意点：同时支持受控和非受控；CSS 变量支持主题。
4. 可追问时补充：工程上需要文档、测试、版本管理和按需引入。

**常见追问**

- 怎么同时支持受控和非受控？→ 判断是否传入了 modelValue：传入时使用 prop 并 emit 更新事件；未传入时使用内部 ref。VueUse 的 `useVModel` 可以简化。

---

### Q17. Vue 2 项目如何迁移到 Vue 3？

- **难度**: 高级
- **标签**: 迁移, 工程

**答案**

**评估**：

- 统计使用的 Vue 2 特有 API（过滤器、`$on`、`.sync`、`$listeners`、`Vue.set`）
- 检查依赖的 Vue 3 兼容性：UI 组件库（Element UI → Element Plus，API 有差异）、Vuex → Pinia、Vue Router 3 → 4
- 构建工具：Vue CLI（维护模式）→ Vite

**迁移路径**：

1. **先升级到 Vue 2.7**：内置 Composition API 和 `<script setup>`，逐步用新写法重构，降低后续迁移成本
2. **使用迁移构建 `@vue/compat`**：在 Vue 3 上运行，大部分 Vue 2 行为保持兼容，控制台输出不兼容的警告，逐个修复
3. **按模块渐进替换**：或者使用微前端，新功能用 Vue 3 开发，旧模块逐步迁移
4. 替换构建工具为 Vite

**主要的破坏性变更**：

- `v-model` 的 prop 和事件名、`.sync` 移除
- `v-if` 优先级高于 `v-for`
- 过滤器移除（改用方法或 computed）
- `$on` / `$off` / `$once` 移除
- `$listeners` 合并到 `$attrs`
- 函数式组件的写法变化
- 全局 API 改为应用实例 API（`Vue.use` → `app.use`）

**口述模板（30–60 秒）**

1. 一句话结论：先评估依赖兼容性，推荐先升级到 2.7，再使用兼容构建逐步迁移。
2. 关键机制：@vue/compat 让 Vue 3 运行大部分 Vue 2 代码，并提示不兼容的地方。
3. 对比或注意点：最大的成本通常是 UI 组件库的替换。
4. 可追问时补充：大型项目可以用微前端渐进迁移。

**常见追问**

- 迁移收益怎么衡量？→ 性能指标（首屏、更新耗时）、包体积、开发效率、类型安全、安全维护（Vue 2 已停止维护，没有安全更新）。

---

### Q18. 微前端中使用 Vue 有哪些问题？

- **难度**: 高级
- **标签**: 架构, 微前端

**答案**

**常见方案**：qiankun（single-spa）、wujie（基于 iframe + Web Components）、Module Federation、micro-app。

**Vue 相关的问题**：

| 问题 | 解决 |
|------|------|
| 多个 Vue 实例、多个版本共存 | 每个子应用独立 `createApp`，Vue 3 没有全局状态污染；共享依赖需要版本一致 |
| 路由冲突 | 子应用使用 `base` 前缀；主应用路由切换时子应用同步 |
| 样式冲突 | scoped、CSS Modules、命名前缀；Shadow DOM 隔离（注意 Teleport 到 body 的弹窗样式会丢失） |
| 卸载不彻底 | 子应用 unmount 时调用 `app.unmount()`，清理全局监听、定时器 |
| 应用间通信 | 主应用下发 props、全局事件、共享的轻量状态；避免共享 Pinia 实例 |
| 弹窗挂载位置 | 子应用的 Teleport 目标需要在子应用容器内或明确约定 |

**口述模板（30–60 秒）**

1. 一句话结论：Vue 3 的应用实例隔离让微前端更容易，但路由、样式、卸载和通信需要处理。
2. 关键机制：子应用独立 createApp，卸载时调用 app.unmount 并清理副作用。
3. 对比或注意点：Shadow DOM 隔离样式时，Teleport 到 body 的内容会失去样式。
4. 可追问时补充：应用间通信尽量通过主应用下发的 props 或事件，避免共享状态实例。

**常见追问**

- 为什么 Vue 3 比 Vue 2 更适合微前端？→ Vue 2 的全局配置（`Vue.use`、`Vue.mixin`、`Vue.component`）会影响同一页面中所有的 Vue 实例；Vue 3 的配置都在应用实例上。

---

### Q19. 怎么设计 composable 的分层和复用？

- **难度**: 高级
- **标签**: 架构, composable

**答案**

**分层**：

```text
基础层（与业务无关）：useEventListener、useDebounce、useStorage（使用 VueUse）
  ↓
通用业务层：useRequest、usePagination、useTable、useForm、usePermission
  ↓
领域层：useOrderList、useUserProfile（组合通用层，包含业务规则）
  ↓
组件：只负责组装和渲染
```

**设计原则**：

- **单一职责**：一个 composable 解决一个问题，通过组合构建复杂逻辑
- **输入灵活**：接受 `MaybeRefOrGetter`，用 `toValue` 处理，参数变化自动响应
- **输出稳定**：返回 ref 组成的对象，可以安全解构
- **副作用可清理**：使用 `onScopeDispose`，在组件外也能工作
- **SSR 安全**：不在顶层访问浏览器 API
- **共享状态**：需要在多个组件间共享时，使用 `createSharedComposable` 或 Pinia，而不是模块级变量（SSR 下会跨请求共享）

**口述模板（30–60 秒）**

1. 一句话结论：composable 分为基础、通用业务、领域三层，组件只负责组装。
2. 关键机制：单一职责，输入支持 ref 和 getter，输出 ref 对象，副作用可清理。
3. 对比或注意点：不要用模块级变量共享状态，SSR 下会跨请求泄露。
4. 可追问时补充：基础能力优先使用 VueUse。

**常见追问**

- composable 和 Pinia store 怎么选？→ 每个组件独立的状态用 composable；需要全局共享、需要开发工具调试、需要 SSR 序列化的状态用 Pinia。

---

### Q20. 怎么测试 Vue 组件？

- **难度**: 高级
- **标签**: 测试

**答案**

**工具**：Vitest + Vue Test Utils（或 Testing Library）；端到端使用 Playwright。

```ts
import { mount, flushPromises } from '@vue/test-utils'
import { createTestingPinia } from '@pinia/testing'

it('提交表单后调用接口', async () => {
  const wrapper = mount(UserForm, {
    props: { userId: 1 },
    global: {
      plugins: [createTestingPinia({ createSpy: vi.fn })],
      stubs: { RouterLink: true },
    },
  })

  await wrapper.find('input[name="name"]').setValue('张三')
  await wrapper.find('form').trigger('submit')
  await flushPromises()  // 等待异步操作完成

  expect(wrapper.emitted('saved')).toHaveLength(1)
})
```

**测试策略**：

- composable：直接调用测试；依赖生命周期的在测试组件中调用
- 组件：测试**行为**（用户操作 → 输出），不测试内部实现
- 异步：`await nextTick()` 等待 DOM 更新，`flushPromises()` 等待 Promise
- Pinia：`createTestingPinia` 模拟 store
- 路由：使用真实的 router（memory history）或 mock

**口述模板（30–60 秒）**

1. 一句话结论：使用 Vitest 和 Vue Test Utils 测试组件，Playwright 做端到端测试。
2. 关键机制：模拟用户操作，断言渲染结果和触发的事件。
3. 对比或注意点：测试行为而不是实现细节；注意等待异步更新。
4. 可追问时补充：用 createTestingPinia 隔离 store。

**常见追问**

- `nextTick` 和 `flushPromises` 的区别？→ nextTick 等待 Vue 的 DOM 更新；flushPromises 等待所有已排队的 Promise（例如 mock 的接口请求）完成。

---

## 七、深入问题

### Q21. 为什么 `<script setup>` 的组件性能更好？

- **难度**: 高级
- **标签**: 编译

**答案**

**普通 setup**：

- setup 返回对象 → 被 `proxyRefs` 包装 → 模板通过组件实例代理（`_ctx`）访问 → 代理的 `get` 依次查找 setupState、data、props、ctx
- 每次访问都要经过多层查找

**`<script setup>`**：

- 模板被编译为**内联在 setup 中的渲染函数**（生产环境）
- 模板直接通过**闭包**访问 setup 中的变量，不经过代理
- 编译器知道每个绑定的类型（ref、常量、reactive、props），可以生成最优的访问代码：
  - 已知是 ref：直接生成 `count.value`（或用 `unref`）
  - 已知是常量：直接访问，不需要追踪
- 更好的压缩：变量名可以被压缩

**口述模板（30–60 秒）**

1. 一句话结论：script setup 的模板编译为内联在 setup 中的渲染函数，通过闭包直接访问变量。
2. 关键机制：不需要经过组件实例代理的多层查找；编译器知道绑定类型，生成最优代码。
3. 对比或注意点：普通 setup 的返回值需要通过代理访问。
4. 可追问时补充：变量名可以被压缩，体积更小。

**常见追问**

- 开发环境也是内联的吗？→ 开发环境为了支持热更新，模板单独编译，通过 setup 返回的绑定对象访问。

---

### Q22. `defineModel` 和 props 解构是怎么由编译器实现的？

- **难度**: 高级
- **标签**: 编译, 宏

**答案**

**defineModel**：

```ts
const model = defineModel<string>()
```

编译为：

```js
export default {
  props: { modelValue: {}, modelModifiers: {} },
  emits: ['update:modelValue'],
  setup(__props) {
    const model = useModel(__props, 'modelValue')
    // useModel 返回一个 customRef：
    // get 读取 props.modelValue；set 时 emit('update:modelValue', value)
    // 父组件没有传 v-model 时，在本地维护值（同时支持非受控用法）
  },
}
```

**响应式 props 解构（3.5）**：

```ts
const { count = 0, msg } = defineProps<{ count?: number; msg: string }>()
watchEffect(() => console.log(count))
```

编译为：

```js
props: { count: { type: Number, default: 0 }, msg: { type: String, required: true } },
setup(__props) {
  watchEffect(() => console.log(__props.count))
}
```

编译器把所有对解构变量的访问**重写**为 `__props.xxx`，所以保持响应式；默认值转换为 props 的 default 选项。

**口述模板（30–60 秒）**

1. 一句话结论：这些宏都是编译器转换，最终生成标准的 props、emits 和运行时辅助函数。
2. 关键机制：defineModel 生成 prop 和更新事件，返回读取 prop、赋值时 emit 的 customRef。
3. 对比或注意点：props 解构后的变量访问被重写为 __props.xxx，因此保持响应式。
4. 可追问时补充：这也是为什么传给 watch 时需要包装为 getter。

**常见追问**

- 解构的 props 传给 composable 会怎样？→ 传入的是当前值，失去响应式；需要传 `() => count`，composable 中用 `toValue` 处理。

---

### Q23. Vue 是怎么处理组件的异步依赖和 Suspense 的？

- **难度**: 高级
- **标签**: Suspense, 异步

**答案**

**异步 setup**：

```vue
<script setup>
const user = await fetchUser()  // 顶层 await 使组件成为异步依赖
</script>
```

- 编译器把 setup 编译为 async 函数，并用 `withAsyncContext` 包装每个 await：在 await 之后**恢复** currentInstance，所以 await 之后仍然可以注册生命周期钩子
- 组件必须在 `<Suspense>` 中使用

**Suspense 的工作方式**：

- 在内存中（离屏容器）渲染默认插槽
- 收集所有异步依赖（异步 setup、异步组件），计数
- 全部完成前显示 `#fallback`；全部完成后把内容移到真实容器，触发 `resolve` 事件
- 内容中有新的异步依赖时，进入 pending 状态（可以通过 `timeout` 控制何时显示 fallback）

**状态**：Suspense 至今仍是实验性功能，API 可能变化；Vapor 纯应用中不支持。

**口述模板（30–60 秒）**

1. 一句话结论：顶层 await 让组件成为异步依赖，Suspense 等待所有异步依赖完成后再显示内容。
2. 关键机制：编译器用 withAsyncContext 在 await 后恢复组件实例上下文。
3. 对比或注意点：Suspense 先在离屏容器中渲染，完成后才移到真实位置。
4. 可追问时补充：Suspense 仍是实验性功能。

**常见追问**

- 不用 Suspense 怎么处理异步数据？→ 在 setup 中同步返回，用 loading 状态和 ref 在数据到达后更新；或者使用 Vue Query。

---

### Q24. Vue 的 SSR 中如何防止跨请求状态污染？

- **难度**: 高级
- **标签**: SSR, 安全

**答案**

**问题**：Node 服务中，模块只加载一次，所有请求共享模块级的状态：

```ts
// 危险：所有用户共享
const currentUser = ref<User | null>(null)
export function useCurrentUser() {
  return currentUser
}
```

用户 A 的数据可能被用户 B 看到，这是**严重的安全问题**。

**解决**：

- **每个请求创建新的应用实例**：`createSSRApp`、`createRouter`、`createPinia` 都放在工厂函数中
- 共享状态使用 Pinia（每个请求一个 pinia 实例）或 `app.provide`
- Nuxt 中使用 `useState`（按请求隔离）而不是模块级 ref
- composable 中的状态放在函数内部

```ts
export function createApp() {
  const app = createSSRApp(App)
  const pinia = createPinia()
  const router = createRouter({ history: createMemoryHistory(), routes })
  app.use(pinia).use(router)
  return { app, pinia, router }
}
```

**口述模板（30–60 秒）**

1. 一句话结论：SSR 中每个请求都要创建新的应用、路由和 Pinia 实例，不能有模块级的共享状态。
2. 关键机制：Node 模块只加载一次，模块级状态被所有请求共享。
3. 对比或注意点：这会导致用户数据泄露，是安全问题。
4. 可追问时补充：Nuxt 中用 useState 替代模块级 ref。

**常见追问**

- 第三方库有全局状态怎么办？→ 检查库是否支持 SSR；不支持时只在客户端使用，或者在每个请求中重新初始化。

---

### Q25. 怎么设计一个支持动态表单的渲染引擎？

- **难度**: 高级
- **标签**: 架构, 表单

**答案**

**Schema 驱动**：

```ts
interface FieldSchema {
  name: string
  component: 'input' | 'select' | 'date' | string
  label: string
  rules?: Rule[]
  props?: Record<string, unknown>
  visible?: (values: FormValues) => boolean    // 联动显示
  options?: Option[] | ((values: FormValues) => Promise<Option[]>)  // 联动选项
}
```

**实现要点**：

- **组件注册表**：`component` 字符串映射到实际组件，支持业务注册自定义组件
- 渲染：遍历 schema，用 `<component :is>` 渲染，`v-model` 绑定到 `values[name]`
- **联动**：`visible`、`options` 作为函数，在 computed 中执行，依赖自动追踪
- **校验**：字段级规则 + 表单级规则，异步校验处理竞态
- **性能**：
  - 每个字段一个组件，字段值变化只更新该字段和依赖它的字段
  - 表单值使用 `reactive`，每个字段只读取自己的值
  - 大表单（数百字段）：分组折叠、懒渲染
- **扩展**：布局（栅格、分组、标签页）、只读模式、草稿保存

**口述模板（30–60 秒）**

1. 一句话结论：用 Schema 描述表单，组件注册表映射渲染组件，联动规则用函数表达。
2. 关键机制：联动函数放在 computed 中，Vue 自动追踪依赖，相关字段变化时自动重新计算。
3. 对比或注意点：每个字段独立组件，避免一个字段变化导致整个表单更新。
4. 可追问时补充：异步选项和校验要处理竞态。

**常见追问**

- 联动函数中有异步请求怎么办？→ 使用 watch + 清理函数处理竞态，或者给异步选项加缓存。

---

### Q26. Vue 的未来发展方向是什么？

- **难度**: 高级
- **标签**: 趋势

**答案**

- **Vapor Mode**：3.6 提供无虚拟 DOM 的编译模式，可以与虚拟 DOM 组件混用，后续会继续扩大适用范围
- **响应式性能**：基于 alien-signals 持续优化
- **工具链**：Vite 8（Rolldown，Rust 打包器）、Vue Language Tools（Volar）、`oxc` 工具链
- **Router**：Vue Router 5 内置基于文件的类型化路由
- **Nuxt**：Nuxt 4 稳定，Nuxt 5 规划中；服务端组件、懒 hydration、流式 SSR
- **行业趋势**：Signals 成为各框架共识；编译器承担更多优化工作（Vue Vapor、Svelte 5、React Compiler）

**面试中的回答方式**：讲清楚 Vapor 和 Signals 的原理，说明它们解决的问题，以及在当前项目中如何评估和采用（先在已测量到瓶颈的页面试点）。

**口述模板（30–60 秒）**

1. 一句话结论：Vue 的方向是 Vapor Mode 和更快的响应式，以及基于 Rust 的工具链。
2. 关键机制：Vapor 跳过虚拟 DOM，alien-signals 提升响应式性能。
3. 对比或注意点：编译器承担更多优化是整个行业的趋势。
4. 可追问时补充：采用新特性要基于测量，先在隔离的页面试点。

**常见追问**

- 虚拟 DOM 会被淘汰吗？→ 短期不会：大量生态依赖 VNode，Vapor 是可选的；两种模式会长期共存。
