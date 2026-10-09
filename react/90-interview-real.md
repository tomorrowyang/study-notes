# React 真实面试题汇总

> 根据近年公开面经、社区高频题和大厂前端面试趋势整理。`来源倾向` 表示题目常见场景，不代表某公司固定题库。

---

### R1. 说说 React Fiber 架构，为什么要引入 Fiber？

- **难度**: senior
- **标签**: Fiber
- **来源倾向**: 大厂二面原理高频

**答案**

Stack Reconciler 递归不可中断；Fiber 把渲染拆成工作单元，用 child/sibling/return 链表 + 双缓冲，实现可中断、可恢复、带优先级的渲染。commit 阶段仍同步。

详见：[30-senior-qa Q1](30-senior-qa.md)、[Q2](30-senior-qa.md)

---

### R2. Hooks 为什么不能写在条件语句里？

- **难度**: mid
- **标签**: Hooks
- **来源倾向**: 一面必问

**答案**

Hook 状态以链表形式挂在 Fiber.memoizedState 上，按调用顺序匹配；条件调用会导致顺序错位。`use` 是例外。

详见：[30-senior-qa Q5](30-senior-qa.md)

---

### R3. setState 是同步还是异步？

- **难度**: mid
- **标签**: 批处理
- **来源倾向**: 经典题，考察版本差异

**答案**

setState 本身是同步入队，但状态「生效」在下一次渲染。React 18 起所有场景自动批处理；React 17 中 setTimeout / 原生事件内是同步更新。需要立即刷新可用 flushSync。

详见：[20-mid-qa Q3](20-mid-qa.md)、[Q4](20-mid-qa.md)

---

### R4. 输出题：连续调用 setCount

```jsx
const [count, setCount] = useState(0)
function onClick() {
  setCount(count + 1)
  setCount((c) => c + 1)
  setCount(count + 5)
  setCount((c) => c * 2)
}
```

- **难度**: mid
- **标签**: 快照
- **来源倾向**: 一面输出题

**答案**

结果为 10，只渲染一次。

详见：[20-mid-qa Q3](20-mid-qa.md)

---

### R5. React Diff 算法原理？key 的作用？为什么不能用 index？

- **难度**: mid
- **标签**: Diff, key
- **来源倾向**: 高频原理题

**答案**

三个假设降为 O(n)：同层比较、类型不同替换、key 标识身份。多节点两轮遍历 + lastPlacedIndex。index 作 key 在增删排序时导致状态错位。

详见：[30-senior-qa Q4](30-senior-qa.md)、[10-junior-qa Q10](10-junior-qa.md)

---

### R6. useEffect 和 useLayoutEffect 区别？

- **难度**: mid
- **标签**: Effect
- **来源倾向**: 高频

**答案**

useLayoutEffect 在 DOM 变更后、绘制前同步执行，适合测量布局；useEffect 通常在绘制后执行。

详见：[20-mid-qa Q7](20-mid-qa.md)

---

### R7. 什么是闭包陷阱？如何解决？

- **难度**: mid
- **标签**: 闭包
- **来源倾向**: Hooks 实践高频

**答案**

回调捕获旧渲染的 state。解决：函数式更新、正确依赖、ref 保存最新值、React 19.2 useEffectEvent。

详见：[20-mid-qa Q10](20-mid-qa.md)、[40-coding-handwritten H5](40-coding-handwritten.md)

---

### R8. useMemo、useCallback、React.memo 区别？什么时候用？

- **难度**: mid
- **标签**: 性能
- **来源倾向**: 性能优化必问

**答案**

memo 跳过组件渲染，useMemo 缓存值，useCallback 缓存函数。只在昂贵计算、传给 memo 组件或作为依赖时使用；React Compiler 可自动处理。

详见：[20-mid-qa Q13](20-mid-qa.md)、[Q14](20-mid-qa.md)

---

### R9. 手写一个 useState / 解释 Hooks 实现

- **难度**: senior
- **标签**: Hooks 原理
- **来源倾向**: 原理手写

**答案**

用数组 + 游标模拟 Hook 链表，setState 写入对应槽位并调度渲染。

详见：[40-coding-handwritten H1](40-coding-handwritten.md)

---

### R10. React 并发模式是什么？useTransition 怎么用？

- **难度**: senior
- **标签**: 并发
- **来源倾向**: React 18+ 必问

**答案**

更新带 Lane 优先级，低优先级渲染可被打断；useTransition 标记非紧急更新，useDeferredValue 延迟值。

详见：[20-mid-qa Q20](20-mid-qa.md)、[Q21](20-mid-qa.md)、[30-senior-qa Q7](30-senior-qa.md)

---

### R11. Scheduler 如何实现时间切片？为什么用 MessageChannel？

- **难度**: senior
- **标签**: Scheduler
- **来源倾向**: 源码深挖

**答案**

小顶堆存任务，MessageChannel 宏任务驱动，5ms 时间片后 shouldYield 让出。避开 setTimeout 4ms 限制和 rIC 不稳定。

详见：[30-senior-qa Q8](30-senior-qa.md)

---

### R12. React 合成事件机制？与原生事件执行顺序？

- **难度**: mid
- **标签**: 事件
- **来源倾向**: 原理高频

**答案**

React 17+ 在 root 容器委托事件，按 React 树模拟捕获冒泡。原生监听器在目标元素上会先于 root 上的 React 冒泡处理执行。

详见：[20-mid-qa Q6](20-mid-qa.md)

---

### R13. React 19 有哪些新特性？

- **难度**: mid
- **标签**: React 19
- **来源倾向**: 2025–2026 高频

**答案**

Actions、useActionState、useOptimistic、useFormStatus、use、ref 作为 prop、文档元数据、RSC 稳定；19.2 Activity、useEffectEvent；19.3 ViewTransition。

详见：[20-mid-qa Q23–Q26](20-mid-qa.md)、[30-senior-qa Q25](30-senior-qa.md)

---

### R14. React Compiler 是什么？会取代 useMemo 吗？

- **难度**: senior
- **标签**: Compiler
- **来源倾向**: 新技术视野

**答案**

构建期自动细粒度记忆化，大部分手动 memo 可移除；需代码遵循 React 规则，渐进启用。

详见：[30-senior-qa Q17](30-senior-qa.md)

---

### R15. SSR 原理？Hydration 是什么？出现 mismatch 怎么办？

- **难度**: senior
- **标签**: SSR
- **来源倾向**: Next.js 方向必问

**答案**

服务端生成 HTML，客户端 hydrateRoot 复用 DOM 绑定事件。mismatch 常因时间、随机数、浏览器 API；客户端专属内容延后渲染。

详见：[30-senior-qa Q11](30-senior-qa.md)、[Q12](30-senior-qa.md)

---

### R16. RSC 和 SSR 有什么区别？

- **难度**: senior
- **标签**: RSC
- **来源倾向**: 2025–2026 高频

**答案**

SSR 产出 HTML 解决首屏；RSC 产出序列化 Payload，组件代码不进客户端包，解决 JS 体积和就近取数，两者互补。

详见：[30-senior-qa Q14](30-senior-qa.md)、[Q15](30-senior-qa.md)

---

### R17. Server Actions / Server Functions 有哪些安全风险？

- **难度**: senior
- **标签**: 安全
- **来源倾向**: 全栈岗

**答案**

本质是公开端点，需鉴权、参数校验、防敏感数据序列化，及时升级 RSC 相关依赖。

详见：[30-senior-qa Q16](30-senior-qa.md)

---

### R18. Context 的性能问题怎么解决？

- **难度**: mid
- **标签**: Context
- **来源倾向**: 性能高频

**答案**

value 变化导致所有消费者渲染。稳定 value、按变化频率拆分、state/dispatch 分离、高频状态用外部 store + selector。

详见：[20-mid-qa Q15](20-mid-qa.md)

---

### R19. Redux、Zustand、Context 怎么选？

- **难度**: mid
- **标签**: 状态管理
- **来源倾向**: 项目经验追问

**答案**

先分类状态：服务端状态用 Query，URL 状态放路由，全局客户端状态用 Zustand/RTK，低频共享用 Context。

详见：[20-mid-qa Q28](20-mid-qa.md)、[50-system-design S5](50-system-design.md)

---

### R20. 什么是 useSyncExternalStore？为什么需要它？

- **难度**: senior
- **标签**: 并发
- **来源倾向**: 原理深挖

**答案**

并发渲染下外部 store 可能导致撕裂，useSyncExternalStore 保证读取一致快照，状态库都基于它。

详见：[20-mid-qa Q16](20-mid-qa.md)、[30-senior-qa Q9](30-senior-qa.md)

---

### R21. 手写一个 Zustand

- **难度**: senior
- **标签**: 状态管理
- **来源倾向**: 中高级手写

**答案**

发布订阅 store + useSyncExternalStore + selector。

详见：[40-coding-handwritten H10](40-coding-handwritten.md)

---

### R22. 实现一个虚拟列表

- **难度**: senior
- **标签**: 性能
- **来源倾向**: 业务场景手写高频

**答案**

占位撑出滚动高度，根据 scrollTop 计算起止索引，加 overscan，translateY 偏移。

详见：[40-coding-handwritten H7](40-coding-handwritten.md)

---

### R23. 错误边界能捕获哪些错误？

- **难度**: mid
- **标签**: 错误处理
- **来源倾向**: 稳定性追问

**答案**

捕获子树渲染和生命周期错误；捕获不到事件、异步、SSR 和自身错误。

详见：[20-mid-qa Q27](20-mid-qa.md)、[40-coding-handwritten H9](40-coding-handwritten.md)

---

### R24. 你在项目中做过哪些 React 性能优化？

- **难度**: senior
- **标签**: 性能
- **来源倾向**: 项目深挖必问

**答案**

用「指标 → 定位工具 → 原因 → 方案 → 数据结果」讲 1–2 个真实案例，如列表虚拟化、Context 拆分、路由分割、Transition 优化输入。

详见：[30-senior-qa Q18](30-senior-qa.md)、[50-system-design S8](50-system-design.md)

---

### R25. React 和 Vue 的区别？

- **难度**: senior
- **标签**: 框架对比
- **来源倾向**: 技术视野

**答案**

React 不可变 + 重渲染 + Compiler；Vue Proxy 依赖追踪 + 编译优化。React 有 Fiber 并发调度。

详见：[30-senior-qa Q23](30-senior-qa.md)

---

### R26. 设计一个 Schema 驱动的表单

- **难度**: senior
- **标签**: 系统设计
- **来源倾向**: 中后台方向系统设计

**答案**

Schema + 组件注册表 + 外部表单 store（字段级订阅）+ 依赖图联动 + 沙箱化表达式。

详见：[50-system-design S2](50-system-design.md)
