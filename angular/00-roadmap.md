# Angular 学习路线（高级工程师面试向）

## 面试画像

国内使用 Angular 的多为企业级中后台、金融、外企和大型 B 端产品，面试考察集中在三层：

- **框架基础**：组件、模板语法（新控制流 `@if` / `@for`）、数据绑定、依赖注入、服务、路由、表单、管道、生命周期 —— 初中级必问
- **原理与新范式**：变更检测（Zone.js → Zoneless、`OnPush`）、Signals（`signal` / `computed` / `effect` / `linkedSignal` / `resource`）、RxJS 与 Signals 的配合、依赖注入的层级与解析规则、Standalone 组件、`@defer`、SSR 与增量水合 —— 中高级区分点
- **工程与架构**：大型应用的模块划分与状态管理（NgRx / SignalStore）、性能排查、版本升级（`ng update`）、从 NgModule / Zone.js / Reactive Forms 迁移、Monorepo（Nx）、微前端 —— 高级岗必谈

高级岗典型问法：

> 「Angular 的变更检测是怎么工作的？Zoneless 之后靠什么触发？」「Signals 和 RxJS 怎么选？」「依赖注入的解析规则是什么？`providedIn: 'root'` 和组件级 providers 有什么区别？」「一个大型 Angular 应用卡顿，你怎么排查？」「你怎么把一个老 Angular 项目升级到最新版本？」

## 版本说明（截至 2026-10）

| 版本 | 时间 | 关键变化 |
|------|------|----------|
| v14–v16 | 2022–2023 | Standalone 组件（v14 预览、v15 稳定）、`inject()`、Signals 预览（v16）、类型化表单 |
| v17 | 2023-11 | 新控制流（`@if` / `@for` / `@switch`）、`@defer`、基于 esbuild + Vite 的 application builder 成为默认 |
| v18 | 2024-05 | Zoneless 实验、`@let` |
| v19 | 2024-11 | **Standalone 成为默认**、`linkedSignal`、`resource`（实验）、增量水合（预览） |
| v20 | 2025-05 | Signals 核心 API（`effect`、`linkedSignal`、`toSignal`）稳定、增量水合稳定、Zoneless 开发者预览 |
| v21 | 2025-11 | **Zoneless 成为新应用默认**、Signal Forms（实验）、**Vitest 成为默认测试运行器**、Angular Aria |
| **v22** | 2026-06 | **Signal Forms、`resource` / `httpResource`、Angular Aria 稳定**；**新应用默认 `OnPush`**，原 `Default` 策略更名为 `Eager`；`@Service` 装饰器、`injectAsync` 异步注入；模板支持箭头函数、展开语法、`@switch` 多 case 与穷尽检查；路由集成浏览器 Navigation API（实验）；**Webpack 相关构建器弃用** |
| 预告 | 2026 Q3 | `@boundary` 模板错误边界（开发者预览） |

> Angular 每 6 个月一个主版本，每个主版本有 18 个月的支持期（6 个月活跃 + 12 个月 LTS）。细节以 [angular.dev](https://angular.dev) 和官方博客为准。面试时说清「新应用默认」和「老项目迁移」的区别会加分。

## 知识地图

```text
Angular
├─ 基础：组件、模板、数据绑定（插值 / 属性 / 事件 / 双向）、指令、管道、生命周期
├─ 模板语法：新控制流（@if / @for track / @switch）、@let、@defer、箭头函数（v22）
├─ 组件通信：input() / output() / model()、viewChild()、内容投影、服务共享
├─ 依赖注入：注入器层级（Environment / Element）、providers 类型、解析修饰符、
│            InjectionToken、inject()、@Service（v22）、injectAsync（v22）
├─ 变更检测：Zone.js 与 NgZone、检测树、OnPush / Eager、Zoneless 调度、
│            markForCheck / detectChanges、Signals 驱动的局部刷新
├─ Signals：signal / computed / effect / linkedSignal / untracked、
│           resource / httpResource、toSignal / toObservable
├─ RxJS：Observable、操作符（switchMap / mergeMap / concatMap / exhaustMap）、
│        Subject 系列、取消订阅（takeUntilDestroyed）、高阶流
├─ 路由：懒加载（loadComponent / loadChildren）、函数式守卫与解析器、
│        输入绑定、预加载策略、RouteReuseStrategy
├─ 表单：模板驱动、响应式（类型化）、Signal Forms（v22 稳定）、自定义控件（CVA）
├─ HTTP：HttpClient、函数式拦截器、httpResource、错误处理与重试
├─ 渲染：SSR、水合、增量水合、事件重放、预渲染（SSG）
├─ 状态管理：服务 + Signals、NgRx Store / Effects、NgRx SignalStore
├─ 工程：Angular CLI、application builder（esbuild）、Vitest、ng update、Nx
└─ 性能与安全：OnPush、track、@defer、虚拟滚动、DomSanitizer、XSS 防护
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 组件、模板、绑定、指令、管道、生命周期、服务与 DI 基础、路由、表单、HTTP
2. [中级 Q&A](20-mid-qa.md) — 变更检测、Zoneless、Signals、RxJS 深入、DI 进阶、路由进阶、表单进阶、SSR、测试
3. [高级 Q&A](30-senior-qa.md) — 变更检测与 Signals 原理、编译器（Ivy）、性能排查、大型应用架构、状态管理选型、升级迁移、微前端
4. [手写与现场编码](40-coding-handwritten.md) — 12 道题：signal 实现、自定义控件、拦截器、守卫、虚拟列表、防抖搜索、SignalStore 等
5. [系统设计与场景](50-system-design.md) — 中后台平台、表单引擎、实时看板、组件库、大型应用迁移等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. 组件的组成（类 + 模板 + 样式 + 元数据），Standalone 组件与 NgModule 的区别
2. 新控制流 `@if` / `@for` / `@switch`，`@for` 为什么**必须写 `track`**
3. `input()` / `output()` / `model()` 信号式 API，与 `@Input` / `@Output` 的区别
4. 依赖注入的注入器层级和解析规则，`providedIn: 'root'`、组件级 providers、`InjectionToken`
5. 变更检测的流程：从根到叶检查、`OnPush` 的触发条件、`markForCheck` 与 `detectChanges`
6. Zone.js 的原理（猴子补丁异步 API），**Zoneless 之后靠什么通知 Angular 刷新**
7. Signals：`signal` / `computed` / `effect` / `linkedSignal`，依赖追踪的原理
8. `resource` / `httpResource`：异步数据如何进入 Signals 世界
9. Signals 和 RxJS 的分工与互转（`toSignal` / `toObservable`）
10. RxJS 高阶映射操作符的区别（`switchMap` / `mergeMap` / `concatMap` / `exhaustMap`）和取消订阅
11. 路由懒加载、函数式守卫与解析器、预加载策略
12. 响应式表单、自定义表单控件（ControlValueAccessor），Signal Forms 的思路
13. `@defer` 的触发条件和预取，与路由懒加载的区别
14. SSR、水合、增量水合、事件重放
15. 性能优化：`OnPush`、`track`、`@defer`、虚拟滚动、纯管道、避免模板中的昂贵计算
16. 状态管理：服务 + Signals、NgRx、SignalStore 的选型
17. 版本升级：`ng update`、官方迁移脚本（schematics）、从 Zone.js / NgModule 迁移
18. Angular 的安全模型：默认的上下文清理、`DomSanitizer`、`bypassSecurityTrust*` 的风险

## 相关技术

- [TypeScript](../typescript/00-roadmap.md)：装饰器、泛型、严格模式 —— Angular 重度依赖 TS
- [JavaScript](../javascript/00-roadmap.md)：事件循环（Zone.js 的基础）、Promise、Proxy
- [React](../react/00-roadmap.md) / [Vue](../vue/00-roadmap.md)：Signals 与 Vue 响应式、React 的对比是常见追问
- [NestJS](../nestjs/00-roadmap.md)：设计思想（模块、DI、装饰器）与 Angular 一脉相承
- [性能优化](../performance/00-roadmap.md)、[浏览器原理](../browser/00-roadmap.md)：渲染与加载性能
- [工程化](../engineering/00-roadmap.md)：Nx Monorepo、微前端
- [前端安全](../frontend-security/00-roadmap.md)：XSS、CSP、Trusted Types

## 自检清单

- [ ] 能讲清变更检测的流程，以及 `OnPush` 下哪些情况会触发检查
- [ ] 能解释 Zone.js 的原理，以及 Zoneless 应用中哪些操作会调度变更检测
- [ ] 能手写一个简化版的 `signal` / `computed` / `effect`
- [ ] 能讲清依赖注入的解析过程和各种解析修饰符
- [ ] 能根据场景选择 `switchMap` / `mergeMap` / `concatMap` / `exhaustMap`
- [ ] 能实现一个自定义表单控件并接入响应式表单
- [ ] 能给出一个大型 Angular 应用的性能排查和优化方案
- [ ] 能规划一个老项目（Zone.js + NgModule + Reactive Forms）的渐进式升级路线
