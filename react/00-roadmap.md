# React 学习路线（高级前端面试向）

## 面试画像

国内大厂高级前端对 React 的考察通常分三层：

1. **会用**：组件、Props、State、Hooks、表单、列表 key、Context
2. **懂原理**：渲染与提交、Fiber、Diff、Hooks 链表、批处理、并发更新、Lane
3. **能落地**：性能优化、状态管理选型、SSR/RSC、错误边界、组件库与工程化、大型应用架构

高级岗回答要能把「API 用法 → 底层机制 → 线上问题 → 取舍」串起来。

## 版本说明（截至 2026-10）

| 版本 / 工具 | 关键内容 |
|-------------|----------|
| React 18 | createRoot、自动批处理、并发渲染、useTransition、useDeferredValue、Suspense SSR 流式 |
| React 19 | Actions、useActionState、useOptimistic、`use`、ref 作为 prop、文档元数据、Server Components / Server Functions 稳定 |
| React 19.2 | `<Activity>`、`useEffectEvent`、`cacheSignal`、Performance Tracks、部分预渲染（Partial Pre-rendering） |
| React 19.3（2026-09） | `<ViewTransition>` / `addTransitionType`、Transition 独立渲染、`browser()` 等（细节以官方 Release 为准） |
| React Compiler 1.0 | 稳定版，自动记忆化组件和 Hooks，支持 React 17+ |
| eslint-plugin-react-hooks v6+ | 默认 flat config，集成 Compiler 驱动规则 |

> 面试时说「我会按项目实际 React 版本选择方案」比背新 API 更加分。类组件生命周期仍可能被问，但重点已转向 Hooks、并发和 RSC。

## 知识地图

```text
React
├─ 基础：JSX、组件、Props、State、事件、条件渲染、列表 key、表单
├─ Hooks：useState / useReducer / useEffect / useLayoutEffect / useRef
│        useMemo / useCallback / useContext / 自定义 Hook / useEffectEvent
├─ 渲染机制：render → reconcile → commit、批处理、快照、纯函数
├─ 原理：Fiber、双缓冲、Diff、Hooks 链表、Lane、Scheduler
├─ 并发：useTransition、useDeferredValue、Suspense、Activity、ViewTransition
├─ React 19：Actions、useActionState、useOptimistic、use、ref as prop
├─ 服务端：SSR、Hydration、Streaming、RSC、Server Functions、Next.js App Router
├─ 性能：Compiler、memo、虚拟列表、代码分割、状态下沉、Profiler
├─ 状态管理：Context、Redux Toolkit、Zustand、Jotai、TanStack Query
└─ 工程：TypeScript、测试、错误边界、组件库设计、微前端
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — JSX、组件、State、Hooks 基础、表单、key
2. [中级 Q&A](20-mid-qa.md) — 渲染机制、Effect、闭包、批处理、Context、性能、React 18/19 API
3. [高级 Q&A](30-senior-qa.md) — Fiber、Diff、Lane、并发、SSR/RSC、Compiler、架构取舍
4. [手写与现场编码](40-coding-handwritten.md) — 12 道高频手写题
5. [系统设计与场景](50-system-design.md) — 组件库、表单引擎、大列表、状态管理、SSR 架构等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. 渲染（render）和提交（commit）两个阶段分别做什么
2. 每次渲染都是一份快照（Snapshot），state 更新不是立即修改变量
3. Hooks 为什么不能条件调用：Fiber 上的 Hook 链表按顺序匹配
4. useEffect 是「与外部系统同步」，不是生命周期替代品
5. useEffect 与 useLayoutEffect 时序差异
6. stale closure 成因与解决：函数式更新、依赖、ref、useEffectEvent
7. key 的作用和 index 作 key 的问题
8. Diff 三个假设：同层比较、类型不同直接替换、key 识别
9. Fiber：可中断工作单元、双缓冲、child/sibling/return
10. Lane 优先级模型和 Scheduler 时间切片
11. useTransition / useDeferredValue / Suspense 的适用场景
12. 自动批处理与 flushSync
13. Context 性能问题与拆分方案
14. memo / useMemo / useCallback 的正确使用，以及 React Compiler 的影响
15. SSR、Hydration、Streaming、Hydration Mismatch
16. RSC 和 Client Component 的边界、序列化限制、安全问题
17. 状态分类：服务端状态、客户端全局状态、URL 状态、表单状态
18. 错误边界、监控和性能分析（Profiler、Performance Tracks）

## 相关技术

- [JavaScript](../javascript/00-roadmap.md)：闭包、事件循环、不可变数据是 React 的基础
- [HTML](../html/00-roadmap.md)：语义化与可访问性
- [CSS](../css/00-roadmap.md)：样式方案、动画与 View Transitions

## 自检清单

- [ ] 能解释一次 setState 到页面更新的完整链路
- [ ] 能手写 useState / useEffect 简化实现并解释 Hook 链表
- [ ] 能讲清 Fiber 为什么能中断，commit 为什么不能中断
- [ ] 能定位并修复一次不必要的重复渲染
- [ ] 能为一个中后台项目选择状态管理方案并说明理由
- [ ] 能讲清 RSC 和 SSR 的区别
