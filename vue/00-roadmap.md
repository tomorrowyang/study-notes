# Vue 学习路线（高级工程师面试向）

## 面试画像

国内 Vue 技术栈的岗位非常多，面试考察集中在三层：

- **使用层**：Composition API、组件通信、生命周期、`v-model`、插槽、`watch` / `computed`、Pinia、Vue Router —— 初中级必问
- **原理层**：响应式系统（Proxy、依赖收集、effect 调度）、`nextTick` 与异步更新队列、虚拟 DOM 与 diff（最长递增子序列）、编译优化（PatchFlag、Block Tree、静态提升）—— 中高级区分点
- **工程与架构**：性能优化、SSR / Nuxt 与 hydration、大型项目状态管理、组件库设计、微前端、Vue 2 迁移、Vapor Mode —— 高级岗必谈

高级岗典型问法：

> 「Vue 3 的响应式是怎么实现的？为什么 Proxy 比 defineProperty 好？」「修改数据后 DOM 是什么时候更新的？」「Vue 3 的 diff 相比 Vue 2 做了哪些优化？」「编译器做了哪些优化？」「Vapor Mode 是什么？为什么能比虚拟 DOM 快？」

## 版本说明（截至 2026-10）

| 版本 | 时间 | 关键内容 |
|------|------|----------|
| Vue 2.7 | 2022-07 | 最后一个 2.x 小版本，内置 Composition API；**Vue 2 已于 2023-12-31 停止维护** |
| Vue 3.0 | 2020-09 | Proxy 响应式、Composition API、编译优化、Fragment / Teleport / Suspense |
| Vue 3.2 | 2021-08 | `<script setup>` 稳定、`v-memo`、`effectScope` |
| Vue 3.3 | 2023-05 | `defineOptions`、`defineSlots`、泛型组件、`defineModel`（实验） |
| Vue 3.4 | 2023-12 | 模板解析器重写（约 2 倍速）、响应式重构（computed 不再无效触发）、`defineModel` 稳定 |
| **Vue 3.5** | 2024-09 | 响应式内存优化（约 -56%）、**响应式 props 解构**、`useTemplateRef`、`useId`、**懒 hydration**、`onWatcherCleanup`、`Teleport defer` |
| **Vue 3.6** | **RC 阶段**（2026-09 为 rc.10） | **Vapor Mode**（无虚拟 DOM 的编译模式，`<script setup vapor>` 选择性启用）；响应式基于 **alien-signals** 重构，性能和内存进一步优化 |
| Pinia 3 | 2025-02 | 放弃 Vue 2 支持；Vuex 已进入维护模式，官方推荐 Pinia |
| Vue Router 5 | 2026 | 合并 `unplugin-vue-router`，内置基于文件的类型化路由（Nuxt 4.4 已升级） |
| Nuxt 4 | 2025-07 | 新目录结构（`app/`）、数据获取优化；**Nuxt 3 于 2026-07-31 停止维护**；Nuxt 4.6（2026-10）以兼容模式支持 Vapor |

> Vue 3.6 截至 2026-10 仍是 RC，稳定版是 3.5.x。Vapor Mode 官方推荐用于「现有应用中的性能敏感页面」和「全新的小型应用」。面试中能讲清 Vapor 的原理和适用范围是加分项，版本细节以官方发布说明为准。

## 知识地图

```text
Vue
├─ 基础：模板语法、指令、计算属性、侦听器、class/style 绑定、条件与列表渲染、事件、表单 v-model
├─ 组件：props / emits、v-model（defineModel）、插槽（作用域插槽）、provide / inject、attrs 透传、异步组件
├─ Composition API：setup、ref / reactive、computed、watch / watchEffect、生命周期钩子、composable
├─ 内置组件：Transition、KeepAlive、Teleport、Suspense
├─ 响应式原理：Proxy / Reflect、track / trigger、effect、依赖存储结构、computed 惰性求值、调度器、effectScope、alien-signals（3.6）
├─ 渲染原理：虚拟 DOM、渲染器、patch、diff（双端 + 最长递增子序列）、key 的作用
├─ 编译原理：parse → transform → codegen、PatchFlag、Block Tree、静态提升、事件缓存、v-once / v-memo
├─ 更新机制：异步更新队列、nextTick、watch flush（pre / post / sync）
├─ 生态：Pinia、Vue Router、Nuxt、VueUse、Vite
├─ SSR：renderToString / 流式、hydration、hydration mismatch、懒 hydration、Nuxt 数据获取
├─ 工程：TypeScript（defineProps 泛型、类型化插槽）、测试（Vitest + Vue Test Utils）、组件库设计
└─ 演进：Vue 2 → 3 迁移、Vapor Mode
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 指令、组件通信、生命周期、Composition API 基础、Pinia、Router
2. [中级 Q&A](20-mid-qa.md) — 响应式原理、更新队列与 nextTick、diff、编译优化、内置组件原理、Router 和 Pinia 原理
3. [高级 Q&A](30-senior-qa.md) — 源码级细节、alien-signals、Vapor Mode、SSR 与 hydration、性能优化、大型项目架构、迁移
4. [手写与现场编码](40-coding-handwritten.md) — 12 道题：迷你响应式、computed、watch、diff、迷你 Router 和 Pinia 等
5. [系统设计与场景](50-system-design.md) — 组件库、中后台框架、大型表单、低代码平台、Nuxt SSR 站点等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. Vue 3 响应式原理：Proxy 拦截、`track` / `trigger`、`WeakMap → Map → Dep` 依赖结构
2. Proxy 相比 `Object.defineProperty` 解决了哪些问题，懒代理的意义
3. `ref` 与 `reactive` 的区别和选型；解构丢失响应式的原因和解决办法
4. `computed` 的惰性求值和缓存（dirty 标记）；3.4 之后的优化
5. `watch` 与 `watchEffect` 的区别；`flush` 时机；竞态清理（`onWatcherCleanup`）
6. 异步更新队列：同一 tick 内多次修改只更新一次；`nextTick` 的原理
7. 虚拟 DOM 的意义与代价；diff 的流程：预处理首尾 → 最长递增子序列移动
8. `key` 的作用，为什么不推荐使用 index
9. 编译优化：PatchFlag、Block Tree、静态提升、事件缓存
10. 组件通信的全部方式和适用场景
11. `KeepAlive` 的原理（缓存 VNode、LRU）、`Teleport`、`Suspense`
12. Vue Router 的原理（history 模式、导航守卫流程）与 Pinia 的原理
13. 自定义 composable 的设计原则，对比 React Hooks
14. SSR 与 hydration；hydration mismatch 的原因；3.5 的懒 hydration
15. 性能优化：减少响应式开销（`shallowRef`、`markRaw`）、`v-memo`、虚拟列表、异步组件
16. Vapor Mode 的原理和适用范围；alien-signals 重构
17. Vue 和 React 的设计差异（响应式 vs 不可变 + 重新渲染）
18. Vue 2 到 Vue 3 的迁移策略

## 相关技术

- [JavaScript](../javascript/00-roadmap.md)：Proxy / Reflect、事件循环（nextTick 基于微任务）、WeakMap
- [React](../react/00-roadmap.md)：设计思想对比，面试常问 Vue vs React
- [Vite](../vite/00-roadmap.md)：Vue 官方构建工具，`@vitejs/plugin-vue` 编译 SFC
- [TypeScript](../typescript/00-roadmap.md)：`defineProps` 泛型、类型化组件
- [浏览器原理](../browser/00-roadmap.md)、[性能优化](../performance/00-roadmap.md)：渲染、hydration、首屏
- [Next.js](../nextjs/00-roadmap.md)：与 Nuxt 的对比

## 自检清单

- [ ] 能手写一个包含 `reactive`、`effect`、`computed` 的迷你响应式系统
- [ ] 能说清修改数据到 DOM 更新的完整流程
- [ ] 能画出 Vue 3 diff 的流程，解释最长递增子序列的作用
- [ ] 能解释 PatchFlag 和 Block Tree 如何减少 diff 的工作量
- [ ] 能讲清 KeepAlive、Teleport、Suspense 的原理和使用场景
- [ ] 能讲清 SSR、hydration 和 mismatch 的排查
- [ ] 能讲清 Vapor Mode 和虚拟 DOM 模式的区别
- [ ] 能对比 Vue 和 React 的设计取舍
