# React 高级 Q&A（源码原理、并发、服务端、架构）

> 目标：能讲清 Fiber、Diff、Hooks、Lane、Scheduler 的实现思路，能设计 SSR/RSC 架构，能定位性能问题并做技术取舍。
>
> 说明：源码函数名以 React 18/19 为参考，内部实现可能随版本变化，面试时讲清「设计思路」比背函数名更重要。

---

## 一、Fiber 架构

### Q1. 为什么需要 Fiber？它解决了什么问题？

- **难度**: senior
- **标签**: Fiber

**答案**

React 15 的 Stack Reconciler 递归遍历组件树，一旦开始就无法中断。大组件树更新时长时间占用主线程，导致输入卡顿和掉帧。

Fiber（React 16+）将渲染工作拆分为以 Fiber 节点为单位的工作单元：

- 每个 Fiber 对应一个组件或 DOM 节点，是一个普通对象
- 树用 `child`、`sibling`、`return` 指针连接，遍历可通过循环完成，进度保存在堆上
- 工作单元之间可以检查是否需要让出主线程（`shouldYield`）
- 支持优先级、中断、恢复、丢弃

```js
// Fiber 关键字段（简化）
{
  tag, type, key,
  stateNode,                 // DOM 节点或类组件实例
  child, sibling, return,    // 树结构
  pendingProps, memoizedProps,
  memoizedState,             // 函数组件中为 Hook 链表
  updateQueue,
  flags, subtreeFlags,       // 副作用标记
  lanes, childLanes,         // 优先级
  alternate,                 // 指向另一棵树中的对应节点
}
```

**口述模板（30–60 秒）**

1. 一句话结论：Fiber 把不可中断的递归更新变成可中断的增量工作。
2. 关键机制：链表结构 + 工作循环 + 优先级。
3. 注意点：只有 render 阶段可中断；单个组件函数内部的长循环仍然会阻塞。
4. 可追问时补充：双缓冲和 Lane。

**常见追问**

- 有了 Fiber 就一定是并发渲染吗？→ 不是，同步 Lane 走 `workLoopSync`，只有 Transition 等并发更新才走可让出的循环。

---

### Q2. 什么是双缓冲（Double Buffering）？

- **难度**: senior
- **标签**: Fiber

**答案**

React 同时维护两棵 Fiber 树：

- **current 树**：对应当前屏幕显示的内容
- **workInProgress 树**：正在内存中构建的新树

两棵树的节点通过 `alternate` 互相指向。render 阶段基于 current 创建或复用 workInProgress 节点；commit 完成后，`root.current` 指向 workInProgress 树，完成切换。

好处：

- 中断的工作不会影响屏幕显示
- 复用节点对象，减少内存分配
- 切换只需改一个指针

**口述模板（30–60 秒）**

1. 一句话结论：两棵树交替，构建完成再一次性切换。
2. 关键机制：alternate 互指，root.current 切换。
3. 注意点：render 中断时 current 树不受影响，所以页面不会出现半成品。
4. 可追问时补充：类似图形学中的双缓冲。

**常见追问**

- render 被打断后之前的工作怎么办？→ 高优先级更新会让 React 从 root 重新构建 workInProgress，低优先级工作可能被丢弃重做。

---

### Q3. render 阶段和 commit 阶段分别做什么？

- **难度**: senior
- **标签**: 渲染流程

**答案**

**render 阶段（可中断，纯计算）**

- `beginWork`：向下遍历，执行组件函数，对比子节点（Diff），打上 flags
- `completeWork`：向上归并，创建/更新 DOM 实例（不挂载），收集 subtreeFlags

**commit 阶段（同步，不可中断）**

1. **Before Mutation**：读取 DOM 快照（getSnapshotBeforeUpdate）
2. **Mutation**：执行 DOM 插入、更新、删除；解绑旧 ref
3. 切换 `root.current`
4. **Layout**：执行 useLayoutEffect、componentDidMount/Update，绑定 ref
5. 之后调度 **Passive Effects**：useEffect 的清理和执行

commit 不能中断，因为 DOM 必须一次性保持一致，否则用户会看到不一致的界面。

**口述模板（30–60 秒）**

1. 一句话结论：render 算差异，commit 改 DOM。
2. 关键机制：beginWork 向下，completeWork 向上；commit 三个子阶段。
3. 注意点：render 可能执行多次，所以必须纯。
4. 可追问时补充：useEffect 在 commit 后异步执行。

**常见追问**

- 为什么 render 阶段不能有副作用？→ render 可能被中断、重试或丢弃，副作用会重复或泄漏。

---

### Q4. React Diff 算法是怎样的？

- **难度**: senior
- **标签**: Diff, Reconciliation

**答案**

传统树 Diff 复杂度 O(n³)，React 基于三个假设降到 O(n)：

1. 只做同层比较，不跨层移动
2. 类型不同直接销毁旧子树、创建新子树
3. 用 key 标识同层元素的身份

**单节点 Diff**：key 和 type 都相同则复用，否则删除旧节点创建新节点。

**多节点 Diff（两轮遍历）**：

1. 第一轮：按索引顺序比较新旧节点，key 相同就复用并继续；遇到 key 不同立即停止
2. 第二轮：
   - 新节点遍历完：删除剩余旧节点
   - 旧节点遍历完：插入剩余新节点
   - 都有剩余：把旧节点放入 `Map<key, fiber>`，遍历新节点查找复用；用 `lastPlacedIndex` 判断是否需要移动

```text
旧：A B C D
新：D A B C
React 会保留 D 不动，把 A、B、C 标记为移动（右移）
```

所以把末尾元素移到开头代价较高。

**口述模板（30–60 秒）**

1. 一句话结论：三个假设把 Diff 降为 O(n)。
2. 关键机制：同层比较、类型判断、key 复用；多节点两轮遍历 + lastPlacedIndex。
3. 注意点：React 只向右移动，头部插入/移动可能导致大量移动。
4. 可追问时补充：Vue 3 的最长递增子序列优化对比。

**常见追问**

- React 和 Vue 3 Diff 差别？→ Vue 3 双端比较 + 最长递增子序列，最小化移动；React 基于单向遍历和 lastPlacedIndex。

---

### Q5. Hooks 底层如何实现？为什么不能条件调用？

- **难度**: senior
- **标签**: Hooks, 原理

**答案**

- 函数组件 Fiber 的 `memoizedState` 指向 Hook 单向链表
- 每个 Hook 节点保存：`memoizedState`（值）、`queue`（更新队列）、`next`
- React 在渲染时安装不同的 Dispatcher：
  - 首次渲染：`HooksDispatcherOnMount`，创建 Hook 节点
  - 更新：`HooksDispatcherOnUpdate`，按顺序复用旧节点并计算新状态
- 渲染期间维护「当前 Hook 指针」，每调用一个 Hook 指针后移

```text
Fiber.memoizedState → useState → useRef → useEffect → null
```

条件调用会让某次渲染的调用顺序与链表不一致，导致状态错配或抛出 Hook 数量错误。

effect 不存在于这条链表的「值」中直接执行，而是生成 effect 对象挂到 Fiber 的 updateQueue，commit 后统一处理。

**口述模板（30–60 秒）**

1. 一句话结论：Hook 状态是 Fiber 上按调用顺序排列的链表。
2. 关键机制：Mount/Update 两套 Dispatcher，按顺序取节点。
3. 注意点：顺序变化即错位。
4. 可追问时补充：`use` 不依赖链表位置，所以可条件调用。

**常见追问**

- 在组件外调用 Hook 为什么报错？→ 此时没有安装 Dispatcher（或为抛错版本）。

---

### Q6. setState 之后内部发生了什么？

- **难度**: senior
- **标签**: 更新链路

**答案**

以函数组件为例：

1. `dispatchSetState(fiber, queue, action)`
2. `requestUpdateLane(fiber)`：根据上下文选择 Lane（事件、Transition、默认）
3. 创建 update 对象，放入 Hook 的环形更新队列
4. 若当前无其他待处理更新，可尝试「急切计算」（eager state），新旧相同则直接跳过
5. `scheduleUpdateOnFiber(root, fiber, lane)`：标记 fiber 到 root 路径上的 lanes / childLanes
6. `ensureRootIsScheduled(root)`：根据最高优先级 Lane 决定同步调度还是通过 Scheduler 调度
7. 进入 render → commit

```text
setState → lane → update 入队 → 标记 root → 调度 → render → commit
```

**口述模板（30–60 秒）**

1. 一句话结论：setState 只是入队一个 update 并请求调度。
2. 关键机制：Lane 决定优先级，root 统一调度。
3. 注意点：同一时机多次更新会合并到一次调度。
4. 可追问时补充：eager state 跳过优化。

**常见追问**

- 更新为什么要标记到 root？→ React 总是从 root 开始调度和渲染，用 childLanes 跳过无更新的子树。

---

## 二、并发与调度

### Q7. Lane 模型是什么？

- **难度**: senior
- **标签**: Lane, 优先级

**答案**

Lane 用 31 位二进制位表示优先级，每一位称为一条车道：

```text
SyncLane              0b...0001   同步（离散输入等）
InputContinuousLane   ...         连续输入
DefaultLane           ...         默认更新
TransitionLanes       ...         多条 Transition 车道
IdleLane              ...         空闲
OffscreenLane         ...         隐藏内容
```

优点：

- 用位运算快速合并、判断、移除优先级
- 能表示「一批」更新（多个 lane 组合）
- 支持饥饿检测：长时间未执行的低优先级更新会被提升为过期

Lane（协调器优先级）和 Scheduler 优先级是两套体系，存在映射关系。

**口述模板（30–60 秒）**

1. 一句话结论：Lane 用位掩码表示更新优先级和批次。
2. 关键机制：位运算合并和选择最高优先级。
3. 注意点：低优先级会过期提升，避免饥饿。
4. 可追问时补充：相比旧 expirationTime 模型更灵活。

**常见追问**

- 为什么替换 expirationTime？→ expirationTime 难以表达「一批」独立更新，Lane 可以分离不同 Transition。

---

### Q8. Scheduler 如何实现时间切片？

- **难度**: senior
- **标签**: Scheduler

**答案**

- 任务按过期时间放入**小顶堆**（taskQueue），未到开始时间的放 timerQueue
- 浏览器中使用 **MessageChannel** 发起宏任务（不用 setTimeout 是为了避免 4ms 嵌套延迟；不用 rAF 是因为后台页暂停；不用 requestIdleCallback 是因为调用频率不稳定且兼容性差）
- 每个宏任务中循环执行任务，默认时间片约 5ms，超时 `shouldYield()` 返回 true，让出主线程
- 任务回调返回函数表示「还没做完」，下次继续

```text
MessageChannel 宏任务
  └─ workLoop：取堆顶任务执行
       └─ performConcurrentWorkOnRoot
            └─ workLoopConcurrent：while (wip && !shouldYield()) performUnitOfWork()
```

**口述模板（30–60 秒）**

1. 一句话结论：Scheduler 用小顶堆 + MessageChannel + 5ms 时间片实现可让出调度。
2. 关键机制：每个工作单元后检查 shouldYield。
3. 注意点：Node 环境使用 setImmediate。
4. 可追问时补充：浏览器原生 scheduler.postTask / yield。

**常见追问**

- 为什么不用微任务？→ 微任务会在同一轮持续执行，无法把控制权交还给浏览器渲染和输入。

---

### Q9. 并发渲染下的「撕裂」是什么？如何避免？

- **难度**: senior
- **标签**: 并发, Tearing

**答案**

并发渲染可能中途让出主线程。如果外部可变 store 在让出期间被修改，同一次渲染中前面的组件读到旧值，后面的组件读到新值，UI 不一致，这就是撕裂（Tearing）。

解决：

- 使用 `useSyncExternalStore` 订阅外部 store，React 检测到快照变化会以同步方式重新渲染
- 状态尽量放在 React state 中

Redux、Zustand 等库已基于 useSyncExternalStore 实现。

**口述模板（30–60 秒）**

1. 一句话结论：并发渲染中外部数据变化导致同一帧 UI 不一致。
2. 关键机制：useSyncExternalStore 保证读取一致快照。
3. 注意点：自己写的全局 store 也要用它接入。
4. 可追问时补充：代价是外部 store 更新无法享受 Transition。

**常见追问**

- React state 为什么没有撕裂问题？→ React 管理其版本，一次渲染使用一致的状态。

---

### Q10. Transition 的中断、复用与 19.3 的变化？

- **难度**: senior
- **标签**: Transition

**答案**

- Transition 更新在 TransitionLane 上渲染，可被同步输入打断
- 被打断后，React 先处理高优先级更新，再基于新状态重新渲染 Transition
- 已显示的内容在 Transition 中不会回退到 Suspense fallback，而是保留旧 UI 等新内容就绪
- 旧版本中多个 Transition 可能被「纠缠（entangled）」在一次渲染里，一个慢 Transition 拖住其他
- React 19.3 起 Transition 独立渲染，慢 Transition 不再阻塞无关 Transition

**口述模板（30–60 秒）**

1. 一句话结论：Transition 是可中断、可重做的低优先级渲染。
2. 关键机制：Lane 区分优先级，打断后重新渲染。
3. 注意点：Transition 中的渲染必须纯，因为可能重复执行。
4. 可追问时补充：19.3 独立渲染改进。

**常见追问**

- Transition 能处理网络慢吗？→ 它管理渲染优先级和 pending 状态，配合 Suspense / Actions 处理异步。

---

## 三、服务端渲染

### Q11. SSR 和 Hydration 的流程是什么？

- **难度**: senior
- **标签**: SSR, Hydration

**答案**

1. 服务端执行组件，生成 HTML（`renderToPipeableStream` / `renderToReadableStream`）
2. 浏览器收到 HTML 直接展示（首屏快、SEO 友好）
3. 加载 JS 后 `hydrateRoot` 进行水合：在已有 DOM 上构建 Fiber 树、绑定事件，使页面可交互

问题：

- 水合前页面可见但不可交互（TTI 延迟）
- 传统 SSR 需要整页数据就绪、整页 JS 下载完才能水合

**口述模板（30–60 秒）**

1. 一句话结论：服务端出 HTML，客户端水合使其可交互。
2. 关键机制：hydrateRoot 复用已有 DOM。
3. 注意点：服务端和客户端首次渲染结果必须一致。
4. 可追问时补充：流式 SSR 和选择性水合。

**常见追问**

- SSR 一定更快吗？→ 首屏内容更快可见，但服务器开销增加，交互时间取决于 JS 体积。

---

### Q12. Hydration Mismatch 的原因和处理？

- **难度**: senior
- **标签**: SSR

**答案**

常见原因：

- 使用 `Date.now()`、`Math.random()`、时区相关格式化
- 渲染时读取 `window`、`localStorage`
- 浏览器扩展修改 DOM
- 无效 HTML 嵌套（如 `<p>` 里放 `<div>`）被浏览器自动修正

处理：

- 客户端专属内容在 effect 后渲染（`useEffect` 设置 mounted），或使用 `useSyncExternalStore` 的 serverSnapshot
- 时间等值由服务端生成并传入
- 不可避免的差异用 `suppressHydrationWarning`（只作用于单层文本）
- 修复 HTML 结构

React 19 提供更清晰的 mismatch diff 报错信息。

**口述模板（30–60 秒）**

1. 一句话结论：服务端与客户端首次渲染结果不一致。
2. 关键机制：水合假设 DOM 与客户端渲染结果匹配。
3. 注意点：不一致时 React 会回退到客户端重新渲染，影响性能。
4. 可追问时补充：React 19.3 的 `browser()` 可标记仅浏览器渲染子树。

**常见追问**

- 主题切换闪烁怎么处理？→ 在 head 内联脚本提前设置 class，并对 html 元素用 suppressHydrationWarning。

---

### Q13. 流式 SSR 和选择性水合是什么？

- **难度**: senior
- **标签**: Streaming, Suspense

**答案**

**流式 SSR**：服务端先发送外壳 HTML，Suspense 边界内的慢内容先输出 fallback；数据就绪后再流式发送 HTML 片段和内联脚本替换。

**选择性水合（Selective Hydration）**：

- 各 Suspense 边界独立水合，不必等整页 JS
- 用户点击尚未水合的区域，React 会优先水合该区域

```jsx
<Layout>
  <Header />
  <Suspense fallback={<Spinner />}>
    <Comments />
  </Suspense>
</Layout>
```

React 19.2 还加入了部分预渲染（Partial Pre-rendering）的 resume API：静态部分预渲染，动态部分请求时补齐。

**口述模板（30–60 秒）**

1. 一句话结论：Suspense 让 SSR 能分块输出、分块水合。
2. 关键机制：fallback 先行，内容就绪后流式替换。
3. 注意点：Suspense 边界划分决定加载体验。
4. 可追问时补充：用户交互优先水合。

**常见追问**

- 流式 SSR 对 SEO 有影响吗？→ 搜索引擎能等待完整流；但关键 SEO 内容最好在外壳中。

---

### Q14. React Server Components 原理是什么？

- **难度**: senior
- **标签**: RSC

**答案**

RSC 是只在服务端（或构建时）运行的组件：

- 可直接访问数据库、文件系统、密钥
- 代码不打包到客户端，减少 JS 体积
- 不能使用 state、effect、浏览器 API

流程：

1. 服务端渲染 Server Component 树，输出 **RSC Payload**（一种可流式传输的序列化格式），包含渲染结果和 Client Component 的引用
2. 客户端 React 根据 Payload 重建树，加载并渲染 Client Component
3. 首屏还可以结合 SSR 输出 HTML

边界：

- `'use client'` 标记客户端模块入口，其导入的依赖都进入客户端包
- Server → Client 传的 props 必须可序列化（不能传普通函数，可传 Server Function）
- Client Component 可以通过 children 接收 Server Component

**口述模板（30–60 秒）**

1. 一句话结论：RSC 在服务端渲染并以序列化 Payload 传给客户端，组件代码不进入客户端包。
2. 关键机制：'use client' 划分边界，props 需可序列化。
3. 注意点：RSC 不是 SSR 的替代，两者可同时使用。
4. 可追问时补充：缓存与重新验证。

**常见追问**

- Server Component 能导入 Client Component 吗？→ 能；反过来 Client 不能直接导入 Server，只能通过 props/children 传入。

---

### Q15. RSC 和 SSR 有什么区别？

- **难度**: senior
- **标签**: RSC, SSR

**答案**

| | SSR | RSC |
|--|-----|-----|
| 产物 | HTML | RSC Payload |
| 目的 | 首屏可见、SEO | 减少客户端 JS、就近取数 |
| 组件代码 | 仍打包到客户端用于水合 | 不打包到客户端 |
| 执行时机 | 每次请求首屏 | 请求时或构建时，可局部刷新 |
| 交互 | 需水合 | 交互部分由 Client Component 负责 |

两者互补：Next.js App Router 中，RSC 生成 Payload，SSR 再把首屏渲染成 HTML。

**口述模板（30–60 秒）**

1. 一句话结论：SSR 解决首屏 HTML，RSC 解决组件在哪执行和 JS 体积。
2. 关键机制：RSC 组件不水合、不进客户端包。
3. 注意点：Client Component 也会被 SSR。
4. 可追问时补充：导航时只请求 RSC Payload。

**常见追问**

- Client Component 会在服务端执行吗？→ 会，SSR 阶段渲染一次生成 HTML，但不能访问浏览器 API。

---

### Q16. Server Functions 如何保证安全？

- **难度**: senior
- **标签**: RSC, 安全

**答案**

`'use server'` 标记的函数可被客户端调用，本质上是一个**公开的 HTTP 端点**。

安全要点：

- 每个 Server Function 内部做身份认证和权限校验，不能依赖页面是否展示按钮
- 参数用 schema（Zod）校验，参数来自不可信客户端
- 不要把敏感数据作为闭包变量或返回值暴露
- 使用 `server-only` 包防止服务端模块被导入客户端
- CSRF：框架通常校验 Origin，但自定义场景需确认
- 依赖及时升级：2025 年 12 月 React Server Components 曾曝出严重的远程代码执行漏洞，相关 react-server-dom-* 包和 Next.js 需升级到修复版本

**口述模板（30–60 秒）**

1. 一句话结论：Server Function 就是公开 API，要按接口标准做鉴权和校验。
2. 关键机制：客户端通过引用 ID 调用服务端函数。
3. 注意点：前端隐藏按钮不是权限控制。
4. 可追问时补充：server-only、taint API 和依赖安全升级。

**常见追问**

- Server Component 传给 Client 的 props 会泄漏什么？→ 所有 props 都会序列化到客户端，不要传整个数据库对象。

---

## 四、性能与架构

### Q17. React Compiler 原理是什么？如何落地？

- **难度**: senior
- **标签**: Compiler, 性能

**答案**

React Compiler 是构建期 Babel 插件，分析组件和 Hook 的数据流，自动插入细粒度记忆化：

- 自动缓存 JSX、计算值、回调函数
- 粒度比手写 useMemo 更细，可在条件分支后记忆化
- 依赖代码遵循 React 规则（渲染纯、props/state 不可变）

落地：

1. 开启 `eslint-plugin-react-hooks` 的 Compiler 相关规则，先修复违规代码
2. 逐目录启用（`compilationMode` / 注解 `'use memo'`、`'use no memo'` 排除）
3. 用 Profiler 对比效果
4. 保留必要的 useMemo/useCallback 控制 effect 依赖

React Compiler 1.0 已稳定，支持 React 17+（低版本需 react-compiler-runtime）。

**口述模板（30–60 秒）**

1. 一句话结论：Compiler 在构建期自动做记忆化。
2. 关键机制：静态分析数据流，生成缓存槽。
3. 注意点：违反 React 规则的代码可能行为变化，要渐进启用。
4. 可追问时补充：effect 依赖可能因记忆化变化而触发次数变化。

**常见追问**

- 有了 Compiler 还需要 memo 吗？→ 大部分不需要，作为精确控制的逃生舱保留。

---

### Q18. React 应用性能优化的方法论？

- **难度**: senior
- **标签**: 性能

**答案**

**先测量**：Lighthouse、Web Vitals（LCP、INP、CLS）、React Profiler、Performance Tracks。

**加载性能**：

- 路由/组件级代码分割、预加载
- RSC / SSR / 流式渲染
- 减少第三方依赖、按需引入
- 图片优化、关键资源优先级

**运行时性能**：

- 减少渲染范围：状态下沉、拆分组件、Context 拆分、selector
- 减少渲染成本：memo / Compiler、避免渲染中昂贵计算
- 大列表虚拟化
- 非紧急更新用 Transition / DeferredValue
- 计算密集逻辑放 Worker

**口述模板（30–60 秒）**

1. 一句话结论：先测量瓶颈，再分加载和运行时两类优化。
2. 关键机制：减少渲染次数、范围和单次成本。
3. 注意点：没有测量的优化可能是负优化。
4. 可追问时补充：INP 与并发特性。

**常见追问**

- 你做过的最有效优化？→ 用「问题指标 → 定位 → 方案 → 数据结果」讲一个真实案例。

---

### Q19. 大列表如何优化？

- **难度**: senior
- **标签**: 虚拟列表

**答案**

- **虚拟列表**：只渲染可视区域 + 缓冲区（TanStack Virtual、react-window）
- **分页 / 无限加载**
- **CSS `content-visibility: auto`**：浏览器跳过屏幕外渲染
- **memo 列表项** + 稳定 key 和回调
- 不定高度：先预估高度，渲染后测量修正

注意：虚拟列表对 Ctrl+F 搜索、无障碍和 SEO 不友好，需要权衡。

**口述模板（30–60 秒）**

1. 一句话结论：只渲染看得见的部分。
2. 关键机制：根据滚动位置计算起止索引和偏移。
3. 注意点：动态高度需测量和缓存。
4. 可追问时补充：content-visibility 低成本方案。

**常见追问**

- 滚动白屏怎么办？→ 增加 overscan 缓冲区，列表项保持轻量。

---

### Q20. 状态设计有哪些原则？

- **难度**: senior
- **标签**: 状态设计

**答案**

1. **最小化状态**：能计算出的数据不存 state
2. **避免冗余和重复**：存 `selectedId` 而不是 `selectedItem` 对象副本
3. **避免矛盾状态**：用 `status: 'idle' | 'loading' | 'success' | 'error'` 代替多个布尔值
4. **扁平化**：深层嵌套数据规范化为 `byId` + `ids`
5. **状态就近**：放在需要它的最低共同父组件
6. **分类管理**：服务端状态、URL 状态、表单状态、UI 状态分开

**口述模板（30–60 秒）**

1. 一句话结论：状态要最小、无冗余、无矛盾、就近。
2. 关键机制：派生数据渲染时计算。
3. 注意点：多个布尔值容易出现不可能状态。
4. 可追问时补充：状态机（XState）管理复杂流程。

**常见追问**

- 为什么不存 selectedItem？→ 原列表更新后副本不同步。

---

### Q21. React 应用中常见的内存泄漏？

- **难度**: senior
- **标签**: 内存

**答案**

- effect 中订阅、定时器、事件监听未清理
- 全局 store / 缓存无上限增长
- 闭包持有大对象（如大数组）被长期存活的回调引用
- 第三方实例（图表、地图、编辑器）未 destroy
- Portal 或手动创建的 DOM 未移除
- 卸载后仍在请求并更新（React 18 已移除警告，但请求资源仍在消耗）

排查：DevTools Memory 多次快照对比，关注 Detached DOM 和 FiberNode 数量。

**口述模板（30–60 秒）**

1. 一句话结论：泄漏多数源于副作用未清理。
2. 关键机制：effect 清理函数负责释放资源。
3. 注意点：StrictMode 能提前暴露缺失清理。
4. 可追问时补充：Heap Snapshot 排查流程。

**常见追问**

- React 18 为什么移除「卸载后 setState」警告？→ 大部分情况无害且误报多，真正的泄漏是订阅未清理。

---

### Q22. 如何设计一个高质量的组件库？

- **难度**: senior
- **标签**: 组件库, 架构

**答案**

- **API 设计**：受控与非受控双模式、一致的命名、合理默认值
- **组合模式**：Compound Components（`<Tabs><Tabs.List/><Tabs.Panel/></Tabs>`）、asChild / Slot
- **Headless 分层**：逻辑和可访问性（Radix、React Aria）与样式分离
- **可访问性**：键盘、焦点管理、ARIA、读屏测试
- **主题**：Design Tokens + CSS 变量
- **类型**：完整 TS 类型、泛型组件
- **工程**：Tree Shaking、按需样式、Storybook、视觉回归、语义化版本和变更日志
- **兼容**：RSC 下区分 Client Component

**口述模板（30–60 秒）**

1. 一句话结论：组件库核心是 API 一致、可组合、可访问、可主题化。
2. 关键机制：Headless + 样式层分离。
3. 注意点：受控/非受控要同时支持。
4. 可追问时补充：版本管理与迁移成本。

**常见追问**

- 受控与非受控如何同时支持？→ `value !== undefined` 时受控，否则内部 state，封装为 useControllableState。

---

### Q23. React 和 Vue 的响应式与更新机制有何不同？

- **难度**: senior
- **标签**: 框架对比

**答案**

| | React | Vue 3 |
|--|-------|-------|
| 响应式 | 不可变数据 + 显式 setState | Proxy 自动依赖追踪 |
| 更新粒度 | 组件级，默认重渲染子树 | 组件级，依赖精确追踪 |
| 优化方式 | memo / Compiler | 编译期静态提升、PatchFlag |
| 调度 | Fiber + 并发 | 异步批量队列，无时间切片 |
| 模板 | JSX，JS 表达力强 | 模板，编译优化空间大 |

趋势：React 用 Compiler 弥补细粒度更新，Vue 探索 Vapor Mode 去虚拟 DOM。

**口述模板（30–60 秒）**

1. 一句话结论：React 是「拉」模型重渲染，Vue 是「推」模型依赖追踪。
2. 关键机制：React 靠不可变和比较，Vue 靠 Proxy 追踪。
3. 注意点：没有绝对优劣，取决于团队和场景。
4. 可追问时补充：Signals 趋势。

**常见追问**

- React 会引入 Signals 吗？→ 官方方向是 Compiler 自动优化，而非 Signals API。

---

### Q24. 微前端中使用 React 需要注意什么？

- **难度**: senior
- **标签**: 微前端

**答案**

- 多个 React 实例：每个子应用独立 `createRoot`，卸载时 `root.unmount()`
- 依赖共享：Module Federation 共享 react / react-dom 单例，避免多份 React 导致 Hooks 报错
- 事件委托在 root 容器，多个版本可共存（React 17+）
- 样式隔离：CSS Modules、Shadow DOM、前缀
- 全局状态通信：通过基座提供的事件或 store，避免直接共享内部状态
- 路由协调：基座统一管理

**口述模板（30–60 秒）**

1. 一句话结论：重点是 React 实例生命周期、依赖共享和隔离。
2. 关键机制：createRoot 挂载、unmount 卸载。
3. 注意点：多份 React 共存会导致 Invalid hook call。
4. 可追问时补充：Module Federation singleton。

**常见追问**

- 「Invalid hook call」常见原因？→ 多份 React、违反 Hooks 规则、react 与 react-dom 版本不匹配。

---

### Q25. ViewTransition 组件的作用是什么？

- **难度**: senior
- **标签**: React 19.3, 动画

**答案**

`<ViewTransition>` 把浏览器 View Transition API 与 React 更新集成：

- 在 Transition、Suspense 揭示、Deferred 更新时自动触发视图过渡动画
- 可为元素命名实现共享元素过渡
- `addTransitionType` 为不同导航类型（前进、后退）指定不同动画

```jsx
<ViewTransition>
  <Page key={url} />
</ViewTransition>
```

只有在 Transition 中触发的更新才会动画，紧急更新不会动画。（API 细节以对应 React 版本官方文档为准，浏览器不支持时自动降级为无动画。）

**口述模板（30–60 秒）**

1. 一句话结论：React 声明式集成浏览器视图过渡。
2. 关键机制：基于 Transition 触发 document.startViewTransition。
3. 注意点：浏览器支持度和降级。
4. 可追问时补充：与 CSS view-transition-name 关系。

**常见追问**

- 为什么只有 Transition 触发？→ 紧急更新需要立即反馈，动画会增加延迟。

---

### Q26. 如何设计 React 应用的错误处理与监控？

- **难度**: senior
- **标签**: 错误处理, 监控

**答案**

分层：

1. **组件层**：路由级、模块级错误边界，提供降级 UI 和重试（react-error-boundary 的 resetKeys）
2. **异步层**：请求层统一错误、Query 的 error 状态、Actions 返回错误
3. **全局层**：
   - `createRoot(container, { onUncaughtError, onCaughtError, onRecoverableError })`
   - `window.onerror`、`unhandledrejection`
4. **上报**：版本号、路由、用户行为面包屑、组件栈（componentStack）
5. **还原**：source map 私有上传

**口述模板（30–60 秒）**

1. 一句话结论：错误边界降级 + 全局回调上报 + source map 还原。
2. 关键机制：React 19 root 级错误回调统一采集。
3. 注意点：错误边界粒度要合适。
4. 可追问时补充：恢复策略和重试。

**常见追问**

- onRecoverableError 是什么？→ React 能自动恢复的错误，如 hydration mismatch 后回退客户端渲染。
