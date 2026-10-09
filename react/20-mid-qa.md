# React 中级 Q&A（原理与正确用法）

> 目标：讲清渲染机制、Effect 心智模型、性能优化手段、React 18/19 新 API，并能做输出题和排查常见 bug。

---

## 一、渲染机制

### Q1. 一次更新从触发到页面变化经历了什么？

- **难度**: mid
- **标签**: 渲染流程

**答案**

三步：

1. **Trigger（触发）**：首次渲染，或 state 更新（setState、dispatch、外部 store 变化）
2. **Render（渲染）**：React 调用组件函数，得到新的元素树，与上次结果对比（Reconciliation）
3. **Commit（提交）**：把差异应用到 DOM，执行 ref 和 layout effect；绘制后执行 passive effect（useEffect）

```text
setState → 调度 → render（纯计算，可中断） → commit（改 DOM，不可中断） → 浏览器绘制 → useEffect
```

**口述模板（30–60 秒）**

1. 一句话结论：触发、渲染、提交三阶段。
2. 关键机制：render 是纯计算，commit 才改 DOM。
3. 注意点：「渲染」不等于「更新 DOM」，渲染结果相同则不改 DOM。
4. 可追问时补充：并发模式下 render 可中断。

**常见追问**

- 组件重新渲染一定会改 DOM 吗？→ 不会，只有差异部分才提交。

---

### Q2. 什么情况下组件会重新渲染？

- **难度**: mid
- **标签**: 渲染

**答案**

1. 自身 state 变化（且新值与旧值 `Object.is` 不同）
2. 父组件重新渲染（默认子组件都会重新渲染，不管 props 是否变化）
3. 使用的 Context value 变化
4. 订阅的外部 store 变化（useSyncExternalStore）

**props 变化不是重新渲染的原因**，父组件渲染才是。

**口述模板（30–60 秒）**

1. 一句话结论：state、父组件渲染、Context、外部 store 四种原因。
2. 关键机制：父渲染默认递归渲染所有子组件。
3. 注意点：memo 才会让子组件根据 props 跳过渲染。
4. 可追问时补充：React Compiler 自动记忆化。

**常见追问**

- setState 相同值会渲染吗？→ 一般会跳过；极少数情况下 React 可能仍渲染该组件一次但不渲染子组件。

---

### Q3. 输出题：连续 setState 后的结果

- **难度**: mid
- **标签**: 快照, 批处理, 输出题

**答案**

```jsx
function Counter() {
  const [count, setCount] = useState(0)

  function handleClick() {
    setCount(count + 1)
    setCount((c) => c + 1)
    setCount(count + 5)
    setCount((c) => c * 2)
  }
  // ...
}
```

点击一次后 `count` 为 **10**。

更新队列计算（本次快照 count = 0）：

1. `count + 1` → 替换为 1
2. `c => c + 1` → 2
3. `count + 5` → 替换为 5
4. `c => c * 2` → 10

**口述模板（30–60 秒）**

1. 一句话结论：普通值是「替换为」，函数是「基于上一个结果计算」。
2. 关键机制：同一事件中的更新进入队列，批处理后统一渲染一次。
3. 注意点：普通值引用的是当前渲染快照。
4. 可追问时补充：自动批处理。

**常见追问**

- 组件渲染了几次？→ 一次，所有更新被批处理。

---

### Q4. 什么是自动批处理？flushSync 有什么用？

- **难度**: mid
- **标签**: 批处理

**答案**

React 18 起，所有更新（事件、Promise、setTimeout、原生事件）都会自动批处理：同一时机的多次 setState 只触发一次渲染。

React 17 只在 React 事件处理函数中批处理。

需要立即同步更新 DOM 时：

```jsx
import { flushSync } from 'react-dom'

flushSync(() => {
  setItems((list) => [...list, item])
})
listRef.current.lastChild.scrollIntoView()
```

**口述模板（30–60 秒）**

1. 一句话结论：React 18 自动批处理所有更新。
2. 关键机制：同一任务中的更新合并为一次渲染。
3. 注意点：flushSync 会影响性能，少用。
4. 可追问时补充：createRoot 才启用新行为。

**常见追问**

- 使用旧的 ReactDOM.render 会怎样？→ 走旧模式，没有自动批处理和并发特性；React 19 已移除该 API。

---

### Q5. 状态在什么情况下会保留或重置？

- **难度**: mid
- **标签**: 状态保留

**答案**

React 按「组件在 UI 树中的位置 + 类型 + key」关联状态：

- 同位置、同类型、同 key → 保留状态
- 类型变化（如 `<Input>` 变 `<TextArea>`，或 `<div>` 变 `<section>`）→ 整个子树卸载重建
- key 变化 → 重置

```jsx
// 切换联系人时重置表单
<ChatForm key={contact.id} contact={contact} />
```

坑：在组件内部定义组件，每次渲染都是新类型，导致子组件状态丢失：

```jsx
function Parent() {
  function Child() { /* ... */ } // 错误：每次渲染都是新的函数
  return <Child />
}
```

**口述模板（30–60 秒）**

1. 一句话结论：状态绑定在树中位置上，不是绑定在 JSX 变量上。
2. 关键机制：位置 + 类型 + key 决定复用。
3. 注意点：不要嵌套定义组件。
4. 可追问时补充：用 key 主动重置。

**常见追问**

- 条件渲染 `{a ? <X/> : <X/>}` 状态会保留吗？→ 会，同位置同类型；需要重置就加不同 key。

---

### Q6. React 合成事件机制是什么？

- **难度**: mid
- **标签**: 事件

**答案**

- React 17+ 将事件委托到 **root 容器**（React 16 是 document）
- 包装原生事件为 SyntheticEvent，抹平差异
- 按 React 树模拟捕获和冒泡
- 不同事件有不同优先级：离散事件（click、input）优先级高，连续事件（scroll、mousemove）次之
- React 17 起移除了事件池

为什么改为 root：多个 React 版本共存、微前端场景下避免互相干扰。

**口述模板（30–60 秒）**

1. 一句话结论：React 在根容器统一委托事件，并按 React 树分发。
2. 关键机制：合成事件 + 事件优先级。
3. 注意点：原生 stopPropagation 与 React 事件混用时顺序容易出错。
4. 可追问时补充：Portal 事件按 React 树冒泡。

**常见追问**

- `onScroll` 冒泡吗？→ React 17 起不冒泡，与浏览器行为一致。

---

## 二、Effect 心智模型

### Q7. useEffect、useLayoutEffect、useInsertionEffect 区别？

- **难度**: mid
- **标签**: Effect

**答案**

| Hook | 时机 | 用途 |
|------|------|------|
| useInsertionEffect | DOM 变更前 | CSS-in-JS 库插入样式 |
| useLayoutEffect | DOM 变更后、浏览器绘制前，同步 | 测量布局、同步调整位置，避免闪烁 |
| useEffect | 绘制后（通常） | 订阅、请求、日志等 |

```jsx
useLayoutEffect(() => {
  const { height } = tooltipRef.current.getBoundingClientRect()
  setTooltipHeight(height)
}, [])
```

useLayoutEffect 会阻塞绘制，滥用影响性能；SSR 中不执行。

**口述模板（30–60 秒）**

1. 一句话结论：需要在绘制前测量和调整布局用 useLayoutEffect，其他用 useEffect。
2. 关键机制：layout effect 在 commit 阶段同步执行。
3. 注意点：layout effect 会阻塞绘制。
4. 可追问时补充：useInsertionEffect 只给样式库用。

**常见追问**

- 由用户点击触发的更新，useEffect 一定在绘制后吗？→ 不一定，离散输入引起的更新，React 可能在绘制前同步刷新 effect。

---

### Q8. 哪些情况下你不需要 useEffect？

- **难度**: mid
- **标签**: Effect, 最佳实践

**答案**

1. **派生数据**：渲染时直接计算

```jsx
// 错误
useEffect(() => setFullName(first + ' ' + last), [first, last])
// 正确
const fullName = first + ' ' + last
```

2. **昂贵计算**：useMemo（或交给 Compiler）
3. **props 变化重置状态**：用 key
4. **响应用户事件的逻辑**：放在事件处理函数里，而不是 effect 中监听状态变化
5. **通知父组件**：在事件中同时调用 onChange
6. **数据请求**：优先框架数据加载或 TanStack Query

Effect 是「逃生舱」，用于和 React 之外的系统同步。

**口述模板（30–60 秒）**

1. 一句话结论：effect 只用于同步外部系统。
2. 关键机制：多余的 effect 会导致额外渲染和数据不一致。
3. 注意点：用户行为引起的逻辑放事件处理函数。
4. 可追问时补充：effect 链导致的级联渲染。

**常见追问**

- 「发送购买请求」应该放 effect 吗？→ 不应该，它是由点击导致的，放事件处理函数。

---

### Q9. Effect 中请求数据如何避免竞态？

- **难度**: mid
- **标签**: Effect, 竞态

**答案**

```jsx
useEffect(() => {
  const controller = new AbortController()

  fetch(`/api/user/${userId}`, { signal: controller.signal })
    .then((res) => res.json())
    .then(setUser)
    .catch((error) => {
      if (error.name !== 'AbortError') setError(error)
    })

  return () => controller.abort()
}, [userId])
```

或使用 ignore 标记：

```jsx
useEffect(() => {
  let ignore = false
  getUser(userId).then((data) => {
    if (!ignore) setUser(data)
  })
  return () => { ignore = true }
}, [userId])
```

生产推荐 TanStack Query / SWR / 框架 loader，它们还处理缓存、去重、重试。

**口述模板（30–60 秒）**

1. 一句话结论：依赖变化时清理上一次请求结果。
2. 关键机制：清理函数在下一次 effect 前执行。
3. 注意点：effect 请求还有瀑布流、无缓存等问题。
4. 可追问时补充：数据请求库和 Suspense。

**常见追问**

- effect 请求为什么会造成瀑布？→ 子组件要等父组件渲染并请求完成后才开始请求。

---

### Q10. stale closure 是什么？useEffectEvent 怎么解决？

- **难度**: mid
- **标签**: 闭包, useEffectEvent

**答案**

```jsx
function ChatRoom({ roomId, theme }) {
  useEffect(() => {
    const conn = connect(roomId)
    conn.on('connected', () => showToast('已连接', theme))
    return () => conn.disconnect()
  }, [roomId, theme]) // theme 变化会导致重连，不合理
}
```

如果去掉 theme 依赖，回调读取的是旧 theme（stale closure）。

React 19.2 的 `useEffectEvent`：

```jsx
const onConnected = useEffectEvent(() => {
  showToast('已连接', theme) // 总能读到最新 theme
})

useEffect(() => {
  const conn = connect(roomId)
  conn.on('connected', onConnected)
  return () => conn.disconnect()
}, [roomId]) // Effect Event 不放进依赖
```

限制：只能在 effect 内调用，不能传给其他组件或在渲染期间调用。

**口述模板（30–60 秒）**

1. 一句话结论：effect 中有「需要最新值但不应触发重新同步」的逻辑时用 useEffectEvent。
2. 关键机制：Effect Event 非响应式，读取最新 props/state。
3. 注意点：不能用它来逃避 lint 依赖检查。
4. 可追问时补充：旧方案是 useRef 保存最新回调。

**常见追问**

- 没有 useEffectEvent 时怎么做？→ 用 ref 保存最新回调，在 layout effect 中更新 ref。

---

### Q11. effect 依赖是对象或函数时，为什么会无限执行？

- **难度**: mid
- **标签**: Effect, 依赖

**答案**

每次渲染创建新对象/函数，`Object.is` 比较总是不同：

```jsx
const options = { roomId } // 每次都是新对象
useEffect(() => {
  connect(options)
}, [options]) // 每次渲染都执行
```

解决：

- 把对象创建移到 effect 内部
- 依赖原始值：`[roomId]`
- 函数移到组件外或 effect 内
- 必要时 useMemo / useCallback（或 Compiler 自动处理）

若 effect 中还 setState，就会形成无限循环。

**口述模板（30–60 秒）**

1. 一句话结论：依赖用 Object.is 比较，新对象每次都不同。
2. 关键机制：effect 依赖响应式值的引用。
3. 注意点：优先移动代码，而不是加 useMemo。
4. 可追问时补充：Compiler 自动稳定引用。

**常见追问**

- 能用 JSON.stringify 做依赖吗？→ 能避免引用问题，但有性能开销且不能处理函数，不推荐作为通用方案。

---

## 三、Hooks 进阶

### Q12. useReducer 什么时候比 useState 合适？

- **难度**: mid
- **标签**: useReducer

**答案**

适合：

- 状态结构复杂，多个字段联动
- 下一个状态依赖多个旧状态
- 更新逻辑需要集中、可测试
- 需要把 dispatch 传给深层组件（dispatch 引用稳定）

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'added':
      return { ...state, todos: [...state.todos, action.todo] }
    case 'toggled':
      return {
        ...state,
        todos: state.todos.map((t) => (t.id === action.id ? { ...t, done: !t.done } : t)),
      }
    default:
      throw new Error(`Unknown action: ${action.type}`)
  }
}

const [state, dispatch] = useReducer(reducer, { todos: [] })
```

**口述模板（30–60 秒）**

1. 一句话结论：复杂联动状态用 useReducer。
2. 关键机制：reducer 是纯函数，集中描述状态变化。
3. 注意点：reducer 中不能有副作用。
4. 可追问时补充：useReducer + Context 小型全局状态。

**常见追问**

- useState 和 useReducer 底层有关系吗？→ useState 可以看作内置 reducer 的 useReducer。

---

### Q13. useMemo 和 useCallback 有什么区别？什么时候用？

- **难度**: mid
- **标签**: 性能

**答案**

```jsx
const visibleTodos = useMemo(() => filterTodos(todos, tab), [todos, tab])
const handleSubmit = useCallback((data) => post(url, data), [url])
```

- useMemo 缓存计算结果
- useCallback 缓存函数引用，等价于 `useMemo(() => fn, deps)`

适合场景：

- 计算确实昂贵（可用 `console.time` 测量）
- 值/函数传给 memo 组件
- 值/函数作为其他 Hook 的依赖

React Compiler 启用后，大部分手动记忆化可以去掉；保留作为精确控制 effect 依赖的逃生舱。

**口述模板（30–60 秒）**

1. 一句话结论：useMemo 缓存值，useCallback 缓存函数。
2. 关键机制：依赖不变时返回上次结果。
3. 注意点：不配合 memo 组件时 useCallback 通常无收益。
4. 可追问时补充：Compiler 自动记忆化。

**常见追问**

- useMemo 能保证一定缓存吗？→ 不能，React 可能在某些情况丢弃缓存，只能作为性能优化，不能依赖其语义正确性。

---

### Q14. React.memo 为什么会失效？

- **难度**: mid
- **标签**: memo, 性能

**答案**

memo 对 props 做浅比较，以下情况会失效：

```jsx
<List
  style={{ color: 'red' }}        // 每次新对象
  onSelect={() => select(id)}     // 每次新函数
  items={data.filter(isVisible)}  // 每次新数组
>
  <Item />                        // children 是新元素
</List>
```

解决：

- 提取为常量或 useMemo / useCallback
- 传原始值而不是对象
- 把状态下沉、把内容作为 children 提升
- 启用 React Compiler

另外，组件内部使用的 Context 变化也会导致 memo 组件渲染。

**口述模板（30–60 秒）**

1. 一句话结论：memo 依赖 props 引用稳定。
2. 关键机制：浅比较每个 prop 的 Object.is。
3. 注意点：children 也是 prop。
4. 可追问时补充：结构调整比 memo 更根本。

**常见追问**

- 不用 memo 怎么减少子组件渲染？→ 状态下沉，或「内容提升」把稳定子树作为 children 传入。

---

### Q15. Context 有什么性能问题？如何优化？

- **难度**: mid
- **标签**: Context, 性能

**答案**

问题：Provider 的 value 变化时，所有消费该 Context 的组件都会重新渲染，即使只用了其中一个字段。

```jsx
<AppContext value={{ user, theme, setTheme }}> // 每次渲染都是新对象
```

优化：

1. value 用 useMemo 稳定引用
2. 按变化频率拆分 Context（UserContext、ThemeContext）
3. 拆分 state 和 dispatch 两个 Context
4. 高频变化状态使用外部 store（Zustand、Jotai）支持 selector 订阅
5. 使用 `use-context-selector` 等方案

**口述模板（30–60 秒）**

1. 一句话结论：Context 没有选择性订阅，value 变就全部消费者渲染。
2. 关键机制：比较 value 的 Object.is。
3. 注意点：Provider 的 value 对象要稳定。
4. 可追问时补充：拆分 Context 或外部 store。

**常见追问**

- Context 适合做全局状态管理吗？→ 适合低频数据；高频复杂状态更适合专用 store。

---

### Q16. useSyncExternalStore 解决什么问题？

- **难度**: mid
- **标签**: 外部状态, 并发

**答案**

用于安全订阅 React 外部的数据源（浏览器 API、全局 store），在并发渲染下避免「撕裂」（Tearing，同一次渲染不同组件读到不同版本数据）。

```jsx
function subscribe(callback) {
  window.addEventListener('online', callback)
  window.addEventListener('offline', callback)
  return () => {
    window.removeEventListener('online', callback)
    window.removeEventListener('offline', callback)
  }
}

function useOnlineStatus() {
  return useSyncExternalStore(
    subscribe,
    () => navigator.onLine,
    () => true, // 服务端快照
  )
}
```

getSnapshot 必须返回稳定值（数据未变时返回同一引用），否则无限渲染。

**口述模板（30–60 秒）**

1. 一句话结论：外部 store 接入 React 的标准方式。
2. 关键机制：订阅变化 + 读取快照，并保证并发一致性。
3. 注意点：getSnapshot 结果要缓存。
4. 可追问时补充：Zustand、Redux 都基于它。

**常见追问**

- subscribe 函数为什么要定义在组件外？→ 每次传新函数会导致重新订阅。

---

### Q17. useId 有什么用？

- **难度**: mid
- **标签**: useId, SSR

**答案**

生成在服务端和客户端一致的唯一 ID，用于无障碍属性：

```jsx
function Field({ label }) {
  const id = useId()
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} aria-describedby={`${id}-hint`} />
      <p id={`${id}-hint`}>至少 8 位</p>
    </>
  )
}
```

不要用于列表 key；不要用 `Math.random()` 生成 ID（SSR 会 hydration mismatch）。

**口述模板（30–60 秒）**

1. 一句话结论：useId 生成 SSR 安全的唯一 ID。
2. 关键机制：基于组件在树中的位置生成。
3. 注意点：不能作为 key。
4. 可追问时补充：React 19.2 生成格式改为下划线形式。

**常见追问**

- 多个 React 根如何避免冲突？→ createRoot / hydrateRoot 设置 `identifierPrefix`。

---

### Q18. useImperativeHandle 什么时候用？

- **难度**: mid
- **标签**: ref

**答案**

限制父组件通过 ref 能访问的能力，只暴露必要方法：

```jsx
function VideoPlayer({ ref, src }) {
  const videoRef = useRef(null)

  useImperativeHandle(ref, () => ({
    play: () => videoRef.current.play(),
    pause: () => videoRef.current.pause(),
  }), [])

  return <video ref={videoRef} src={src} />
}
```

适用：聚焦、滚动、播放等命令式操作。能用 props 表达的不要用 ref。

**口述模板（30–60 秒）**

1. 一句话结论：自定义 ref 暴露的命令式 API。
2. 关键机制：替换 ref.current 为自定义对象。
3. 注意点：命令式 API 是逃生舱，避免滥用。
4. 可追问时补充：React 19 ref 作为 prop。

**常见追问**

- 「打开弹窗」应该用 ref 调用 open 吗？→ 优先用 `isOpen` prop 声明式控制。

---

### Q19. HOC、Render Props、自定义 Hook 对比？

- **难度**: mid
- **标签**: 逻辑复用

**答案**

| | HOC | Render Props | 自定义 Hook |
|--|-----|--------------|-------------|
| 形式 | 包装组件 | 函数作为子元素 | 函数调用 |
| 问题 | 嵌套地狱、props 冲突、来源不清 | 回调嵌套 | 需遵守 Hooks 规则 |
| 现状 | 少数场景（权限、埋点包装） | 少用 | 主流 |

**口述模板（30–60 秒）**

1. 一句话结论：自定义 Hook 是主流逻辑复用方式。
2. 关键机制：Hook 复用逻辑，不增加组件层级。
3. 注意点：HOC 要透传 ref 和静态方法。
4. 可追问时补充：HOC 仍可用于横切关注点。

**常见追问**

- HOC 的 props 命名冲突怎么处理？→ 加命名空间或改用 Hook。

---

## 四、React 18/19 新 API

### Q20. useTransition 有什么用？

- **难度**: mid
- **标签**: 并发, Transition

**答案**

把非紧急更新标记为 Transition，使其可以被紧急更新（如输入）打断：

```jsx
const [isPending, startTransition] = useTransition()

function selectTab(nextTab) {
  startTransition(() => {
    setTab(nextTab)
  })
}
```

- 输入框受控值不能放在 transition 中（会卡顿或丢字）
- isPending 可展示加载态
- React 19 起 startTransition 可接收 async 函数（Action）
- React 19.3 起不同 Transition 独立渲染，慢的 Transition 不再拖住无关 Transition

**口述模板（30–60 秒）**

1. 一句话结论：Transition 是低优先级、可中断的更新。
2. 关键机制：Lane 优先级 + 可中断渲染。
3. 注意点：受控输入的值必须同步更新。
4. 可追问时补充：与防抖的区别。

**常见追问**

- 和防抖有什么区别？→ 防抖固定延迟；Transition 不延迟，设备快时立即完成，慢时可被打断。

---

### Q21. useDeferredValue 和 useTransition 区别？

- **难度**: mid
- **标签**: 并发

**答案**

- useTransition：能控制「发起更新的代码」时使用
- useDeferredValue：只能拿到「值」（如来自 props）时使用，获得一个延迟版本

```jsx
function SearchPage({ query }) {
  const deferredQuery = useDeferredValue(query)
  const isStale = query !== deferredQuery

  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      <SlowList query={deferredQuery} />
    </div>
  )
}
```

SlowList 需要 memo，才能在 deferredQuery 未变时跳过渲染。

**口述模板（30–60 秒）**

1. 一句话结论：Transition 包装更新，DeferredValue 包装值。
2. 关键机制：先用旧值渲染，后台用新值渲染。
3. 注意点：配合 memo 才有效果。
4. 可追问时补充：React 19 支持 initialValue。

**常见追问**

- 能用来防抖请求吗？→ 它不减少请求次数，需要请求库或防抖配合。

---

### Q22. Suspense 是什么？能做哪些事？

- **难度**: mid
- **标签**: Suspense

**答案**

Suspense 声明「子树加载中显示什么」：

```jsx
<Suspense fallback={<Skeleton />}>
  <Comments />
</Suspense>
```

触发 Suspense 的来源：

- `lazy` 组件加载
- `use(promise)` 读取未完成的 Promise
- 支持 Suspense 的框架/库数据加载（Next.js、TanStack Query suspense 模式）
- SSR 流式渲染

注意：在 effect 或事件中请求数据不会触发 Suspense。已显示的内容在 Transition 中再次挂起时不会退回 fallback。

**口述模板（30–60 秒）**

1. 一句话结论：Suspense 统一声明异步加载态。
2. 关键机制：子组件抛出 Promise / 挂起，最近边界显示 fallback。
3. 注意点：边界粒度要设计好，避免整页闪烁。
4. 可追问时补充：流式 SSR 和选择性水合。

**常见追问**

- 多个 Suspense 嵌套怎么显示？→ 由外到内逐步揭示，React 会批量揭示避免闪烁。

---

### Q23. React 19 的 Actions 和 useActionState 是什么？

- **难度**: mid
- **标签**: React 19, 表单

**答案**

Action：在 Transition 中执行的 async 函数，React 自动管理 pending、错误和乐观更新。

```jsx
async function updateName(prevState, formData) {
  const name = formData.get('name')
  const error = await saveName(name)
  if (error) return { error }
  return { error: null }
}

function NameForm() {
  const [state, formAction, isPending] = useActionState(updateName, { error: null })

  return (
    <form action={formAction}>
      <input name="name" />
      <button disabled={isPending}>保存</button>
      {state.error && <p role="alert">{state.error}</p>}
    </form>
  )
}
```

- `<form action={fn}>` 提交成功后会自动重置非受控表单
- `useFormStatus` 让子组件读取父表单 pending 状态

**口述模板（30–60 秒）**

1. 一句话结论：Actions 把异步提交、pending、错误处理标准化。
2. 关键机制：基于 Transition 和 FormData。
3. 注意点：表单字段偏非受控。
4. 可追问时补充：配合 Server Functions 实现渐进增强。

**常见追问**

- useFormStatus 能在 form 同组件中读吗？→ 不能，它读取的是父级 form 状态，需放在子组件中。

---

### Q24. useOptimistic 怎么用？

- **难度**: mid
- **标签**: React 19, 乐观更新

**答案**

在异步操作完成前先显示预期结果：

```jsx
function LikeButton({ liked, onToggle }) {
  const [optimisticLiked, setOptimisticLiked] = useOptimistic(liked)

  async function action() {
    setOptimisticLiked(!optimisticLiked)
    await onToggle()
  }

  return (
    <form action={action}>
      <button>{optimisticLiked ? '已赞' : '点赞'}</button>
    </form>
  )
}
```

Action 结束后，乐观值自动回到真实 state；失败时自然回滚（真实 state 未变）。

**口述模板（30–60 秒）**

1. 一句话结论：useOptimistic 在 Action 期间显示乐观值。
2. 关键机制：Transition 结束后切回真实值。
3. 注意点：必须在 Action / Transition 中设置乐观值。
4. 可追问时补充：错误提示和重试。

**常见追问**

- 和手动 setState 乐观更新有何不同？→ 不需要手写回滚逻辑。

---

### Q25. React 19 的 `use` 有什么特点？

- **难度**: mid
- **标签**: React 19, use

**答案**

`use` 读取 Promise 或 Context：

```jsx
function Comments({ commentsPromise }) {
  const comments = use(commentsPromise) // 未完成时挂起到 Suspense
  return comments.map((c) => <p key={c.id}>{c.text}</p>)
}

function Button({ show }) {
  if (show) {
    const theme = use(ThemeContext) // 可以条件调用
  }
}
```

注意：

- Promise 应来自服务端组件或缓存，不能在客户端组件渲染时新建（每次渲染新 Promise 会无限挂起）
- 不能在 try/catch 中调用，错误由错误边界处理

**口述模板（30–60 秒）**

1. 一句话结论：use 能读 Promise 和 Context，且可条件调用。
2. 关键机制：未完成的 Promise 触发 Suspense。
3. 注意点：Promise 需要稳定和缓存。
4. 可追问时补充：RSC 向客户端传 Promise 流式渲染。

**常见追问**

- 为什么 use 能条件调用？→ 它不依赖 Hook 链表中的固定位置保存状态。

---

### Q26. Activity 组件是什么？

- **难度**: mid
- **标签**: React 19.2, Activity

**答案**

`<Activity mode="visible|hidden">` 用于隐藏 UI 但保留状态：

```jsx
<Activity mode={tab === 'settings' ? 'visible' : 'hidden'}>
  <Settings />
</Activity>
```

- hidden 时：子树隐藏（display: none）、effect 被卸载、更新以低优先级延迟处理
- visible 时：恢复显示并重新执行 effect，状态保留
- 可用于预渲染用户可能访问的页面、保留 Tab 切换状态

对比条件渲染：条件渲染卸载会丢失状态；CSS 隐藏不会清理 effect。

**口述模板（30–60 秒）**

1. 一句话结论：Activity 隐藏 UI 但保留状态，并暂停副作用。
2. 关键机制：hidden 时清理 effect，低优先级渲染。
3. 注意点：需要 effect 清理逻辑写正确。
4. 可追问时补充：类似 Vue 的 KeepAlive。

**常见追问**

- 和 Vue KeepAlive 区别？→ 理念相似，但 Activity 会卸载 effect，并能用于后台预渲染。

---

### Q27. 错误边界是什么？捕获不到哪些错误？

- **难度**: mid
- **标签**: 错误处理

**答案**

错误边界是实现了 `getDerivedStateFromError` / `componentDidCatch` 的类组件，捕获子树渲染、生命周期中的错误并显示降级 UI。

捕获不到：

- 事件处理函数中的错误
- 异步代码（setTimeout、Promise 回调）
- 服务端渲染错误
- 错误边界自身的错误

事件中的错误可 try/catch；也可以把错误放入 state 再 throw，让边界接管。

React 19 中 `createRoot` 提供 `onCaughtError`、`onUncaughtError`、`onRecoverableError` 用于统一上报。

**口述模板（30–60 秒）**

1. 一句话结论：错误边界捕获渲染阶段错误并降级。
2. 关键机制：类组件生命周期。
3. 注意点：事件和异步错误捕获不到。
4. 可追问时补充：react-error-boundary 的 reset。

**常见追问**

- 错误边界放在哪？→ 路由级 + 关键独立模块级，避免整页白屏。

---

## 五、生态与工程

### Q28. 状态管理库如何选型？

- **难度**: mid
- **标签**: 状态管理

**答案**

先对状态分类：

| 状态类型 | 推荐 |
|----------|------|
| 服务端数据（缓存、同步） | TanStack Query / SWR / RTK Query |
| 局部 UI 状态 | useState / useReducer |
| 跨组件低频全局 | Context |
| 跨组件高频全局 | Zustand / Jotai / Redux Toolkit |
| URL 状态（筛选、分页） | 路由 search params / nuqs |
| 表单状态 | React Hook Form |

选型要点：

- Redux Toolkit：规范强、DevTools、适合大团队
- Zustand：轻量、selector 订阅、样板少
- Jotai：原子化，细粒度依赖

**口述模板（30–60 秒）**

1. 一句话结论：先分类状态，再选工具。
2. 关键机制：服务端状态和客户端状态是两类问题。
3. 注意点：大部分「全局状态」其实是服务端缓存。
4. 可追问时补充：URL 状态便于分享和刷新恢复。

**常见追问**

- 为什么不推荐把接口数据都放 Redux？→ 需要自己处理缓存、过期、去重、重试，专用库更合适。

---

### Q29. TanStack Query 解决了什么问题？

- **难度**: mid
- **标签**: 数据请求

**答案**

```jsx
const { data, isPending, error } = useQuery({
  queryKey: ['user', userId],
  queryFn: ({ signal }) => fetchUser(userId, { signal }),
  staleTime: 60_000,
})
```

能力：

- 按 queryKey 缓存和去重
- stale-while-revalidate
- 自动重试、窗口聚焦刷新
- 取消请求和竞态处理
- 分页、无限加载
- mutation 与缓存失效、乐观更新
- SSR 预取和 hydration

**口述模板（30–60 秒）**

1. 一句话结论：它是服务端状态的缓存和同步层。
2. 关键机制：queryKey 标识数据，staleTime 控制新鲜度。
3. 注意点：queryKey 必须包含所有参数。
4. 可追问时补充：与 RSC 的分工。

**常见追问**

- staleTime 和 gcTime 区别？→ staleTime 决定何时视为过期需刷新，gcTime 决定无订阅后缓存保留多久。

---

### Q30. React + TypeScript 常见类型怎么写？

- **难度**: mid
- **标签**: TypeScript

**答案**

```tsx
type ButtonProps = React.ComponentProps<'button'> & {
  variant?: 'primary' | 'ghost'
}

function Button({ variant = 'primary', ...rest }: ButtonProps) {
  return <button data-variant={variant} {...rest} />
}

type ListProps<T> = {
  items: T[]
  renderItem: (item: T) => React.ReactNode
  getKey: (item: T) => React.Key
}

function List<T>({ items, renderItem, getKey }: ListProps<T>) {
  return <ul>{items.map((item) => <li key={getKey(item)}>{renderItem(item)}</li>)}</ul>
}

function handleChange(e: React.ChangeEvent<HTMLInputElement>) {}
```

- children 类型：`React.ReactNode`
- 不推荐 `React.FC`（历史上隐式 children，现在可用但非必需）

**口述模板（30–60 秒）**

1. 一句话结论：Props 用 type 声明，复用原生属性用 ComponentProps。
2. 关键机制：泛型组件保证数据和渲染函数类型一致。
3. 注意点：事件类型要区分元素。
4. 可追问时补充：判别联合类型做互斥 props。

**常见追问**

- 互斥 props 怎么表达？→ 判别联合：`{ mode: 'link'; href: string } | { mode: 'button'; onClick: () => void }`。

---

### Q31. 如何测试 React 组件？

- **难度**: mid
- **标签**: 测试

**答案**

推荐 React Testing Library 理念：按用户使用方式测试，而不是测试实现细节。

```jsx
test('提交后显示成功提示', async () => {
  const user = userEvent.setup()
  render(<LoginForm />)

  await user.type(screen.getByLabelText('邮箱'), 'a@b.com')
  await user.click(screen.getByRole('button', { name: '登录' }))

  expect(await screen.findByText('登录成功')).toBeInTheDocument()
})
```

- 查询优先级：getByRole > getByLabelText > getByText > getByTestId
- 接口用 MSW 模拟
- 单测：Vitest / Jest；E2E：Playwright

**口述模板（30–60 秒）**

1. 一句话结论：像用户一样测试组件行为。
2. 关键机制：基于可访问性角色查询。
3. 注意点：不要断言内部 state。
4. 可追问时补充：MSW 拦截网络。

**常见追问**

- 为什么推荐 getByRole？→ 同时验证可访问性。

---

### Q32. 路由懒加载和权限路由怎么做？

- **难度**: mid
- **标签**: 路由

**答案**

```jsx
const routes = [
  {
    path: '/admin',
    lazy: () => import('./routes/admin'), // React Router 数据路由懒加载
    loader: requireRole('admin'),
  },
]
```

要点：

- 路由级代码分割
- 在 loader / 中间件中做鉴权和重定向，避免先渲染再跳转造成闪烁
- 菜单根据权限生成，但前端权限只是体验，后端必须校验
- 按钮级权限用组件或 Hook 封装

**口述模板（30–60 秒）**

1. 一句话结论：路由懒加载 + 加载前鉴权。
2. 关键机制：loader 在渲染前执行。
3. 注意点：前端权限不是安全边界。
4. 可追问时补充：动态路由表。

**常见追问**

- 401 怎么处理？→ 请求层统一拦截，跳转登录并记录回跳地址。

---

### Q33. 自定义 Hook 设计有哪些原则？

- **难度**: mid
- **标签**: 自定义 Hook

**答案**

- 命名表达用途：`useChatRoom`，而不是 `useMount` 这类生命周期封装
- 输入参数和返回值简单清晰，返回对象便于扩展
- 内部处理清理逻辑
- 返回的函数尽量引用稳定
- 不要把所有 effect 都封装成 Hook，关注「解决具体问题」
- 编写测试（`renderHook`）

```jsx
function useChatRoom({ serverUrl, roomId }) {
  useEffect(() => {
    const conn = createConnection(serverUrl, roomId)
    conn.connect()
    return () => conn.disconnect()
  }, [serverUrl, roomId])
}
```

**口述模板（30–60 秒）**

1. 一句话结论：自定义 Hook 封装具体业务问题。
2. 关键机制：每个调用者独立状态。
3. 注意点：避免 `useEffectOnce` 这类泛化生命周期 Hook。
4. 可追问时补充：返回数组还是对象。

**常见追问**

- 返回数组还是对象？→ 两个值且常重命名用数组，否则用对象。

---

### Q34. 如何排查组件为什么重复渲染？

- **难度**: mid
- **标签**: 性能, 调试

**答案**

工具：

- React DevTools Profiler：「Record why each component rendered」
- React DevTools「Highlight updates」
- Chrome Performance 面板中的 React Performance Tracks（19.2+）
- why-did-you-render

排查思路：

1. 确认是否真的有性能问题（渲染耗时，而不是次数）
2. 找到触发源：state、父组件、Context、store
3. 判断 props 引用是否变化
4. 优化：状态下沉、拆分组件、memo、稳定引用、Compiler

**口述模板（30–60 秒）**

1. 一句话结论：先 Profiler 测量，再定位渲染原因。
2. 关键机制：渲染原因只有 state、父组件、Context、store。
3. 注意点：渲染次数多不一定慢。
4. 可追问时补充：Performance Tracks。

**常见追问**

- 生产环境怎么分析？→ 使用 profiling 构建版本，或 `<Profiler onRender>` 采集数据。
