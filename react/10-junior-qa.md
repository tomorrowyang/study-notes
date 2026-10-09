# React 初级 Q&A（能用）

> 目标：能写出正确的组件、状态、事件、列表、表单和基础 Hooks，并知道常见坑的原因。

---

## 一、核心概念

### Q1. React 是什么？声明式 UI 有什么好处？

- **难度**: junior
- **标签**: 基础

**答案**

React 是用于构建用户界面的 JavaScript 库，核心思想：

- **声明式（Declarative）**：描述「状态对应的 UI 应该长什么样」，而不是一步步操作 DOM
- **组件化（Component-based）**：UI 拆分为可复用、可组合的组件
- **单向数据流**：数据从父组件经 props 向下流动

```text
UI = f(state)
```

好处：状态与 UI 的对应关系清晰，减少手动 DOM 操作导致的状态不一致。

**口述模板（30–60 秒）**

1. 一句话结论：React 是声明式、组件化的 UI 库。
2. 关键机制：状态变化后 React 计算新 UI，并最小化更新 DOM。
3. 注意点：React 只负责视图层，路由、数据请求需要生态配合。
4. 可追问时补充：渲染和提交两阶段。

**常见追问**

- React 是框架还是库？→ 官方定位是库；结合 Next.js 等才形成完整框架。

---

### Q2. JSX 的本质是什么？

- **难度**: junior
- **标签**: JSX

**答案**

JSX 是语法扩展，编译后变成创建 React 元素的函数调用：

```jsx
const el = <Button type="primary">提交</Button>

// 新 JSX Transform（React 17+）大致编译为：
import { jsx as _jsx } from 'react/jsx-runtime'
const el = _jsx(Button, { type: 'primary', children: '提交' })
```

React 元素是描述 UI 的普通对象，不是真实 DOM。

JSX 规则：

- 只能返回一个根节点（可用 Fragment）
- `class` 写成 `className`，`for` 写成 `htmlFor`
- 属性用 camelCase，如 `onClick`、`tabIndex`
- `{}` 中放 JS 表达式，不能放语句

**口述模板（30–60 秒）**

1. 一句话结论：JSX 是创建 React 元素的语法糖。
2. 关键机制：编译为 `jsx()` / `createElement()` 调用。
3. 注意点：新 Transform 下不需要手动 import React。
4. 可追问时补充：React 元素是不可变对象。

**常见追问**

- 为什么组件名要大写？→ 小写会被当作原生 HTML 标签。

---

### Q3. 函数组件和类组件有什么区别？

- **难度**: junior
- **标签**: 组件

**答案**

| | 函数组件 | 类组件 |
|--|----------|--------|
| 状态 | Hooks | `this.state` |
| 副作用 | useEffect 等 | 生命周期方法 |
| this | 无 | 需要处理 this 绑定 |
| 逻辑复用 | 自定义 Hook | HOC / Render Props |
| 新特性 | 全部支持 | 部分新特性不支持 |

现代项目以函数组件为主；错误边界目前仍需要类组件（或使用 react-error-boundary 库）。

**口述模板（30–60 秒）**

1. 一句话结论：函数组件 + Hooks 是主流写法。
2. 关键机制：Hooks 让函数组件具备状态和副作用能力。
3. 注意点：函数组件每次渲染都会重新执行。
4. 可追问时补充：错误边界仍依赖类组件。

**常见追问**

- 函数组件每次渲染都重新执行，状态为什么不丢？→ 状态保存在 React 内部的 Fiber 上，不在函数局部变量中。

---

### Q4. Props 和 State 有什么区别？

- **难度**: junior
- **标签**: Props, State

**答案**

| | Props | State |
|--|-------|-------|
| 来源 | 父组件传入 | 组件内部维护 |
| 可修改 | 只读 | 通过 setter 更新 |
| 作用 | 配置组件 | 记录随交互变化的数据 |

```jsx
function Counter({ step }) {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(count + step)}>{count}</button>
}
```

**口述模板（30–60 秒）**

1. 一句话结论：Props 是外部输入，State 是内部记忆。
2. 关键机制：两者变化都会触发重新渲染。
3. 注意点：不要直接修改 props。
4. 可追问时补充：状态提升。

**常见追问**

- 能用 props 初始化 state 吗？→ 可以，但后续 props 变化不会自动同步；需要同步时考虑受控或 key 重置。

---

### Q5. 什么是单向数据流？

- **难度**: junior
- **标签**: 数据流

**答案**

数据从父组件通过 props 流向子组件；子组件不能直接修改父数据，只能调用父组件传下来的回调：

```jsx
function Parent() {
  const [value, setValue] = useState('')
  return <Input value={value} onChange={setValue} />
}
```

好处：数据来源唯一，问题容易追踪。

**口述模板（30–60 秒）**

1. 一句话结论：数据向下传，事件向上传。
2. 关键机制：子组件通过回调通知父组件修改状态。
3. 注意点：props 是只读的。
4. 可追问时补充：与 Vue 双向绑定的对比。

**常见追问**

- 层级很深怎么办？→ 组合（children）、Context 或状态管理库。

---

## 二、State 与事件

### Q6. useState 怎么用？setState 后为什么立即读取还是旧值？

- **难度**: junior
- **标签**: useState

**答案**

```jsx
const [count, setCount] = useState(0)

function handleClick() {
  setCount(count + 1)
  console.log(count) // 仍是旧值
}
```

原因：每次渲染的 state 是一份快照。setState 只是请求 React 用新值重新渲染，本次函数执行中的 `count` 常量不会改变。

**口述模板（30–60 秒）**

1. 一句话结论：state 是渲染快照，setState 不会修改当前变量。
2. 关键机制：React 安排一次新的渲染，新渲染中拿到新值。
3. 注意点：需要基于新值计算时直接用计算结果或函数式更新。
4. 可追问时补充：批处理。

**常见追问**

- 初始值计算很昂贵怎么办？→ 传函数 `useState(() => compute())`，只在首次渲染执行。

---

### Q7. 什么时候需要函数式更新？

- **难度**: junior
- **标签**: useState

**答案**

新状态依赖旧状态时使用：

```jsx
// 只加 1：三次都基于同一个快照 count
setCount(count + 1)
setCount(count + 1)
setCount(count + 1)

// 加 3：每次基于上一次更新后的值
setCount((c) => c + 1)
setCount((c) => c + 1)
setCount((c) => c + 1)
```

在定时器、订阅等闭包中，函数式更新还能避免读取到旧状态。

**口述模板（30–60 秒）**

1. 一句话结论：依赖旧值时用 `setState(prev => next)`。
2. 关键机制：更新函数会按队列顺序依次执行。
3. 注意点：更新函数必须是纯函数。
4. 可追问时补充：stale closure。

**常见追问**

- 严格模式下更新函数为什么执行两次？→ 开发环境检查纯度，结果只用一次。

---

### Q8. React 中如何处理事件？

- **难度**: junior
- **标签**: 事件

**答案**

```jsx
<button onClick={handleClick}>保存</button>
<button onClick={() => remove(item.id)}>删除</button>
```

- 事件名使用 camelCase
- 传函数引用，不要写成 `onClick={handleClick()}`（会在渲染时立即调用）
- 阻止默认行为需显式 `e.preventDefault()`，不能 `return false`
- 事件对象是合成事件（SyntheticEvent），抹平浏览器差异

**口述模板（30–60 秒）**

1. 一句话结论：传函数引用给 camelCase 事件属性。
2. 关键机制：React 在根容器做事件委托。
3. 注意点：不要在 JSX 中直接调用处理函数。
4. 可追问时补充：合成事件机制。

**常见追问**

- 怎么拿原生事件？→ `e.nativeEvent`。

---

### Q9. 条件渲染有哪些写法？`&&` 有什么坑？

- **难度**: junior
- **标签**: 条件渲染

**答案**

```jsx
{isLogin ? <UserMenu /> : <LoginButton />}
{error && <ErrorTip message={error} />}
```

坑：左侧为数字 0 时会渲染出 `0`：

```jsx
{list.length && <List items={list} />} // 空数组时页面显示 0
{list.length > 0 && <List items={list} />} // 正确
```

复杂条件可以提前 return 或抽变量。

**口述模板（30–60 秒）**

1. 一句话结论：三元、&&、提前返回都可用。
2. 关键机制：React 会渲染数字，但忽略 false、null、undefined。
3. 注意点：`&&` 左侧要是布尔值。
4. 可追问时补充：条件渲染会影响组件状态保留。

**常见追问**

- 返回 null 组件会卸载吗？→ 返回 null 的组件本身仍挂载，只是不输出 DOM。

---

### Q10. 列表渲染为什么需要 key？能用 index 吗？

- **难度**: junior
- **标签**: key, 列表

**答案**

key 帮助 React 在多次渲染之间识别同一个元素，从而正确复用组件和状态。

```jsx
{todos.map((todo) => <TodoItem key={todo.id} todo={todo} />)}
```

用 index 作 key 的问题：列表插入、删除、排序时，index 对应的数据变化，组件状态（如输入框内容、勾选态）会错位。

只有列表静态、不重排、不增删时 index 才相对安全。

**口述模板（30–60 秒）**

1. 一句话结论：key 是元素在兄弟节点中的稳定身份。
2. 关键机制：Diff 用 key 匹配新旧节点。
3. 注意点：不要用 index 或随机数作 key。
4. 可追问时补充：key 也可以用来重置组件。

**常见追问**

- 用 `Math.random()` 作 key 会怎样？→ 每次渲染都卸载重建，性能差且丢失状态。

---

## 三、表单

### Q11. 受控组件和非受控组件有什么区别？

- **难度**: junior
- **标签**: 表单

**答案**

- **受控组件**：值由 React state 控制，`value` + `onChange`
- **非受控组件**：值由 DOM 自己维护，通过 ref 或 FormData 读取，`defaultValue`

```jsx
// 受控
<input value={name} onChange={(e) => setName(e.target.value)} />

// 非受控
<input name="email" defaultValue="" ref={inputRef} />
```

受控适合即时校验、联动；非受控适合简单表单、文件上传、性能敏感的大表单（如 React Hook Form）。

**口述模板（30–60 秒）**

1. 一句话结论：受控由 state 驱动，非受控由 DOM 维护。
2. 关键机制：受控每次输入都触发渲染。
3. 注意点：受控组件 value 不能是 undefined，否则会在受控/非受控间切换警告。
4. 可追问时补充：React 19 form action 更偏向非受控 + FormData。

**常见追问**

- 文件输入能受控吗？→ 不能，`<input type="file">` 值只读，必须非受控。

---

### Q12. 多字段表单如何管理状态？

- **难度**: junior
- **标签**: 表单

**答案**

```jsx
const [form, setForm] = useState({ name: '', email: '' })

function handleChange(e) {
  const { name, value } = e.target
  setForm((prev) => ({ ...prev, [name]: value }))
}

<input name="name" value={form.name} onChange={handleChange} />
```

复杂表单（校验、动态字段、性能）推荐 React Hook Form + Zod。

**口述模板（30–60 秒）**

1. 一句话结论：用对象 state + 计算属性名统一处理。
2. 关键机制：不可变更新，展开旧值再覆盖。
3. 注意点：不要直接修改 form 对象。
4. 可追问时补充：表单库的非受控优化。

**常见追问**

- 为什么不能 `form.name = x; setForm(form)`？→ 引用没变，React 用 Object.is 判断后可能跳过渲染。

---

## 四、Hooks 基础

### Q13. useEffect 怎么用？依赖数组有哪几种情况？

- **难度**: junior
- **标签**: useEffect

**答案**

useEffect 用于让组件与外部系统同步（订阅、定时器、网络、非 React 组件）。

```jsx
useEffect(() => {
  // 每次渲染后执行
})

useEffect(() => {
  // 仅挂载后执行一次（严格模式开发环境会挂载-卸载-再挂载）
}, [])

useEffect(() => {
  // userId 变化后执行
}, [userId])
```

依赖必须包含 effect 中用到的所有响应式值（props、state、组件内函数）。

**口述模板（30–60 秒）**

1. 一句话结论：useEffect 在提交后执行副作用。
2. 关键机制：依赖用 Object.is 比较，变化才重新执行。
3. 注意点：不要漏写依赖，用 lint 规则约束。
4. 可追问时补充：很多场景不需要 effect。

**常见追问**

- effect 在渲染期间执行吗？→ 不，在浏览器绘制后（通常）异步执行。

---

### Q14. useEffect 的清理函数什么时候执行？

- **难度**: junior
- **标签**: useEffect

**答案**

```jsx
useEffect(() => {
  const id = setInterval(tick, 1000)
  return () => clearInterval(id)
}, [])
```

清理函数执行时机：

1. 依赖变化、下一次 effect 执行之前
2. 组件卸载时

常见需要清理的：定时器、事件监听、订阅、WebSocket、请求取消。

**口述模板（30–60 秒）**

1. 一句话结论：清理在下次 effect 前和卸载时执行。
2. 关键机制：先清理旧 effect，再执行新 effect。
3. 注意点：严格模式开发下会额外执行一次「挂载-清理-挂载」检查清理是否完整。
4. 可追问时补充：AbortController 取消请求。

**常见追问**

- 为什么开发环境 effect 执行两次？→ StrictMode 刻意检测缺失的清理逻辑，生产不会。

---

### Q15. useRef 有哪些用途？

- **难度**: junior
- **标签**: useRef

**答案**

1. **获取 DOM**：

```jsx
const inputRef = useRef(null)
<input ref={inputRef} />
inputRef.current.focus()
```

2. **保存可变值且不触发渲染**：定时器 ID、上一次的值、最新回调

```jsx
const timerRef = useRef(null)
```

注意：渲染期间不要读写 `ref.current`（初始化除外），否则渲染不纯。

**口述模板（30–60 秒）**

1. 一句话结论：ref 用于 DOM 引用和跨渲染的可变值。
2. 关键机制：ref 对象在组件生命周期内保持同一个引用。
3. 注意点：修改 ref 不会触发渲染。
4. 可追问时补充：ref callback 和 React 19 ref 清理函数。

**常见追问**

- ref 和 state 怎么选？→ 需要显示到 UI 的用 state，只在事件/effect 中用的用 ref。

---

### Q16. React 19 中 ref 如何传给子组件？

- **难度**: junior
- **标签**: ref, React 19

**答案**

React 19 中函数组件可以直接把 `ref` 作为 prop 接收：

```jsx
function MyInput({ ref, ...props }) {
  return <input ref={ref} {...props} />
}

<MyInput ref={inputRef} />
```

React 18 及之前需要 `forwardRef`：

```jsx
const MyInput = forwardRef((props, ref) => <input ref={ref} {...props} />)
```

React 19 的 ref callback 还支持返回清理函数。

**口述模板（30–60 秒）**

1. 一句话结论：React 19 ref 可作为普通 prop，forwardRef 逐步淘汰。
2. 关键机制：函数组件 props 中直接拿到 ref。
3. 注意点：旧版本仍需 forwardRef。
4. 可追问时补充：useImperativeHandle 限制暴露的方法。

**常见追问**

- ref callback 返回清理函数有什么用？→ 元素卸载时清理，如解绑观察器。

---

### Q17. Hooks 的使用规则是什么？

- **难度**: junior
- **标签**: Hooks

**答案**

1. 只能在函数组件或自定义 Hook 的顶层调用
2. 不能在条件、循环、嵌套函数、try/catch 中调用
3. 自定义 Hook 以 `use` 开头

原因：React 按调用顺序把 Hook 与内部状态对应，顺序变化会导致状态错乱。

例外：React 19 的 `use()` 可以在条件和循环中调用。

**口述模板（30–60 秒）**

1. 一句话结论：Hooks 只能在顶层、按固定顺序调用。
2. 关键机制：Fiber 上用链表按顺序保存 Hook 状态。
3. 注意点：提前 return 之后再调用 Hook 也会导致顺序变化。
4. 可追问时补充：`use` 是例外。

**常见追问**

- 怎么保证遵守规则？→ eslint-plugin-react-hooks。

---

### Q18. 什么是自定义 Hook？

- **难度**: junior
- **标签**: 自定义 Hook

**答案**

自定义 Hook 是以 `use` 开头、内部调用其他 Hook 的函数，用于复用有状态逻辑：

```jsx
function useOnlineStatus() {
  const [online, setOnline] = useState(navigator.onLine)

  useEffect(() => {
    const update = () => setOnline(navigator.onLine)
    window.addEventListener('online', update)
    window.addEventListener('offline', update)
    return () => {
      window.removeEventListener('online', update)
      window.removeEventListener('offline', update)
    }
  }, [])

  return online
}
```

每个使用它的组件拥有独立状态，自定义 Hook 复用的是逻辑，不是状态。

**口述模板（30–60 秒）**

1. 一句话结论：自定义 Hook 复用有状态逻辑。
2. 关键机制：每次调用都有独立的 Hook 状态。
3. 注意点：不调用 Hook 的函数不要以 use 开头。
4. 可追问时补充：比 HOC 和 Render Props 更扁平。

**常见追问**

- 想共享状态怎么办？→ 状态提升、Context 或外部 store。

---

## 五、组件组织

### Q19. Fragment 有什么用？

- **难度**: junior
- **标签**: Fragment

**答案**

在不额外增加 DOM 节点的情况下返回多个元素：

```jsx
<>
  <dt>{term}</dt>
  <dd>{desc}</dd>
</>

{items.map((item) => (
  <Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.desc}</dd>
  </Fragment>
))}
```

列表中需要 key 时必须用 `<Fragment key>`，短语法不支持 key。

**口述模板（30–60 秒）**

1. 一句话结论：Fragment 让组件返回多个元素而不加包裹节点。
2. 关键机制：不生成真实 DOM。
3. 注意点：需要 key 时写完整形式。
4. 可追问时补充：避免破坏表格、Flex 布局结构。

**常见追问**

- 为什么不用 div 包？→ 会破坏 HTML 语义（如 table、dl）和布局。

---

### Q20. children 和组合（Composition）怎么用？

- **难度**: junior
- **标签**: 组合

**答案**

```jsx
function Card({ title, children, footer }) {
  return (
    <section className="card">
      <h2>{title}</h2>
      {children}
      {footer}
    </section>
  )
}

<Card title="订单" footer={<Button>查看</Button>}>
  <OrderList />
</Card>
```

组合优于继承。通过 children 或「插槽 props」可以避免 props 层层透传。

**口述模板（30–60 秒）**

1. 一句话结论：用 children 和插槽 props 组合组件。
2. 关键机制：父组件决定子内容，容器组件只负责布局。
3. 注意点：React 不推荐组件继承。
4. 可追问时补充：组合可缓解 props drilling。

**常见追问**

- 组合怎么减少 props drilling？→ 中间层组件直接接收已组装好的子元素，不需要知道深层数据。

---

### Q21. 什么是状态提升？

- **难度**: junior
- **标签**: State

**答案**

多个组件需要共享同一状态时，把状态移到它们最近的共同父组件，再通过 props 传递值和修改回调：

```jsx
function Accordion() {
  const [activeIndex, setActiveIndex] = useState(0)
  return (
    <>
      <Panel isActive={activeIndex === 0} onShow={() => setActiveIndex(0)} />
      <Panel isActive={activeIndex === 1} onShow={() => setActiveIndex(1)} />
    </>
  )
}
```

**口述模板（30–60 秒）**

1. 一句话结论：共享状态放到最近共同父组件。
2. 关键机制：单一数据源（Single Source of Truth）。
3. 注意点：提升过高会导致大范围重渲染。
4. 可追问时补充：状态下沉优化性能。

**常见追问**

- 状态应该放多高？→ 放在需要它的组件的最近公共祖先，不要更高。

---

### Q22. 组件之间有哪些通信方式？

- **难度**: junior
- **标签**: 通信

**答案**

| 场景 | 方式 |
|------|------|
| 父 → 子 | props |
| 子 → 父 | 回调 props |
| 兄弟 | 状态提升 |
| 跨层级 | Context |
| 全局 | Zustand / Redux / Jotai |
| 服务端数据 | TanStack Query / SWR |
| 父调用子方法 | ref + useImperativeHandle（少用） |
| URL 状态 | 路由 query / params |

**口述模板（30–60 秒）**

1. 一句话结论：按关系远近选通信方式。
2. 关键机制：React 推崇单向数据流。
3. 注意点：不要滥用全局状态和事件总线。
4. 可追问时补充：服务端状态单独管理。

**常见追问**

- 事件总线适合 React 吗？→ 一般不推荐，数据流隐式、难追踪。

---

### Q23. 如何不可变地更新对象和数组 state？

- **难度**: junior
- **标签**: 不可变数据

**答案**

```jsx
// 对象
setUser((u) => ({ ...u, profile: { ...u.profile, city: '上海' } }))

// 数组新增 / 删除 / 修改
setList((l) => [...l, item])
setList((l) => l.filter((x) => x.id !== id))
setList((l) => l.map((x) => (x.id === id ? { ...x, done: !x.done } : x)))

// 排序
setList((l) => l.toSorted((a, b) => a.order - b.order))
```

嵌套很深时可用 Immer：`produce(state, draft => { draft.a.b = 1 })`。

**口述模板（30–60 秒）**

1. 一句话结论：创建新对象/新数组，不修改原值。
2. 关键机制：React 用 Object.is 判断 state 是否变化。
3. 注意点：push、splice、sort 会修改原数组。
4. 可追问时补充：Immer 的 Proxy 草稿。

**常见追问**

- 原地修改后 setState 会发生什么？→ 引用相同可能跳过渲染，且破坏 memo 优化。

---

### Q24. StrictMode 有什么作用？

- **难度**: junior
- **标签**: StrictMode

**答案**

开发环境下帮助发现问题：

- 组件函数、初始化函数、更新函数执行两次，检测渲染是否纯
- effect 额外执行「挂载 → 清理 → 挂载」，检测清理是否完整
- ref callback 额外执行一次
- 警告废弃 API

生产环境不生效。

**口述模板（30–60 秒）**

1. 一句话结论：StrictMode 是开发期的纯度和清理检查工具。
2. 关键机制：故意重复调用暴露副作用。
3. 注意点：不要为了避免执行两次而删除 StrictMode。
4. 可追问时补充：为并发渲染和 Activity 做准备。

**常见追问**

- 请求发两次怎么办？→ 正确清理（取消或忽略结果），或使用数据请求库。

---

### Q25. React 中有哪些样式方案？

- **难度**: junior
- **标签**: 样式

**答案**

| 方案 | 特点 |
|------|------|
| 全局 CSS + className | 简单，但易冲突 |
| CSS Modules | 局部作用域，零运行时 |
| Tailwind CSS | 原子类，设计约束强 |
| CSS-in-JS（运行时） | 动态样式方便，但有运行时成本，与 RSC 兼容性差 |
| 零运行时 CSS-in-JS（vanilla-extract、Panda） | 编译期生成 CSS |
| inline style | 动态值适用，不支持伪类和媒体查询 |

新项目常见：Tailwind 或 CSS Modules；动态主题用 CSS 变量。

**口述模板（30–60 秒）**

1. 一句话结论：优先零运行时方案。
2. 关键机制：CSS Modules 编译期生成唯一类名。
3. 注意点：运行时 CSS-in-JS 在 RSC/流式 SSR 下有额外成本。
4. 可追问时补充：CSS 变量做主题切换。

**常见追问**

- 动态类名怎么拼？→ `clsx` / `classnames`。

---

### Q26. dangerouslySetInnerHTML 有什么风险？

- **难度**: junior
- **标签**: 安全

**答案**

React 默认会转义 JSX 中的字符串，防止 XSS。`dangerouslySetInnerHTML` 绕过转义：

```jsx
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(html) }} />
```

风险：不可信 HTML 会执行恶意脚本。必须先用 DOMPurify 等净化。

其他风险点：`href={userInput}` 可能是 `javascript:` URL（React 会警告并在新版本中阻止部分情况），需要校验协议。

**口述模板（30–60 秒）**

1. 一句话结论：React 默认转义，dangerouslySetInnerHTML 需要自行净化。
2. 关键机制：字符串被当作 HTML 解析。
3. 注意点：用户输入的 URL 也要校验协议。
4. 可追问时补充：CSP 和 Trusted Types。

**常见追问**

- Markdown 渲染怎么做？→ 用 react-markdown 等生成 React 元素，或净化后再插入。

---

### Q27. React.lazy 和 Suspense 如何做代码分割？

- **难度**: junior
- **标签**: 代码分割

**答案**

```jsx
const Editor = lazy(() => import('./Editor'))

function Page() {
  return (
    <Suspense fallback={<Spinner />}>
      <Editor />
    </Suspense>
  )
}
```

- `lazy` 接收返回 Promise 的函数，模块需默认导出组件
- 加载中显示最近 Suspense 的 fallback
- `lazy` 要在模块顶层声明，不要在组件内部声明（否则每次渲染重建、状态丢失）

**口述模板（30–60 秒）**

1. 一句话结论：lazy + Suspense 实现组件按需加载。
2. 关键机制：动态 import 生成独立 chunk。
3. 注意点：加载失败需要错误边界兜底。
4. 可追问时补充：路由级分割和预加载。

**常见追问**

- 命名导出怎么 lazy？→ `lazy(() => import('./x').then((m) => ({ default: m.Editor })))`。

---

### Q28. createPortal 有什么用？

- **难度**: junior
- **标签**: Portal

**答案**

把子节点渲染到组件 DOM 层级之外，常用于弹窗、Tooltip、Toast：

```jsx
return createPortal(<Modal />, document.body)
```

特点：

- DOM 位置变了，但 React 树位置不变
- Context 正常传递
- 事件按 React 树冒泡，而不是 DOM 树

**口述模板（30–60 秒）**

1. 一句话结论：Portal 改变 DOM 挂载位置，避免 overflow 和 z-index 问题。
2. 关键机制：仍属于原 React 树。
3. 注意点：事件冒泡按 React 树，可能被父组件意外捕获。
4. 可追问时补充：原生 dialog / popover 的 top layer。

**常见追问**

- SSR 时能用 Portal 吗？→ 服务端没有 document，需在客户端挂载后再渲染。

---

### Q29. 类组件生命周期如何对应 Hooks？

- **难度**: junior
- **标签**: 生命周期

**答案**

| 类组件 | Hooks |
|--------|-------|
| constructor | useState 初始化 |
| componentDidMount | `useEffect(fn, [])` |
| componentDidUpdate | `useEffect(fn, [deps])` |
| componentWillUnmount | effect 返回的清理函数 |
| shouldComponentUpdate | React.memo |
| getDerivedStateFromProps | 渲染期间计算 / key 重置 |
| componentDidCatch | 无 Hook 等价，仍需类组件错误边界 |

注意：Hooks 不是生命周期的一一替代，应按「与什么同步」思考 effect。

**口述模板（30–60 秒）**

1. 一句话结论：大部分生命周期可用 useEffect 和 memo 表达。
2. 关键机制：effect 关注同步，不关注时机。
3. 注意点：错误边界仍需类组件。
4. 可追问时补充：「你可能不需要 effect」。

**常见追问**

- useEffect 空依赖完全等于 componentDidMount 吗？→ 不完全，effect 在绘制后执行，且严格模式会重复执行。

---

### Q30. Context 基础用法是什么？

- **难度**: junior
- **标签**: Context

**答案**

```jsx
const ThemeContext = createContext('light')

function App() {
  const [theme, setTheme] = useState('light')
  return (
    <ThemeContext value={theme}>
      <Page />
    </ThemeContext>
  )
}

function Button() {
  const theme = useContext(ThemeContext)
  return <button className={theme}>OK</button>
}
```

React 19 可直接用 `<ThemeContext value>`，旧版本写 `<ThemeContext.Provider value>`。

适合主题、语言、登录用户等低频变化的全局数据。

**口述模板（30–60 秒）**

1. 一句话结论：Context 解决跨层级传值。
2. 关键机制：Provider value 变化时，所有消费者重新渲染。
3. 注意点：高频变化数据放 Context 会有性能问题。
4. 可追问时补充：拆分 Context 和外部 store。

**常见追问**

- 没有 Provider 时读到什么？→ createContext 的默认值。
