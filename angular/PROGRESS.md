# Angular 学习进度

## 完成度

| 文件 | 内容 | 数量 | 状态 | 完成日期 |
|------|------|------|------|----------|
| [00-roadmap](./00-roadmap.md) | 路线图、版本说明（v14–v22） | - | ⬜ | |
| [10-junior-qa](./10-junior-qa.md) | 初级 Q&A | 30 | ⬜ | |
| [20-mid-qa](./20-mid-qa.md) | 中级 Q&A | 32 | ⬜ | |
| [30-senior-qa](./30-senior-qa.md) | 高级 Q&A | 26 | ⬜ | |
| [40-coding-handwritten](./40-coding-handwritten.md) | 手写题 | 12 | ⬜ | |
| [50-system-design](./50-system-design.md) | 系统设计 | 8 | ⬜ | |
| [90-interview-real](./90-interview-real.md) | 真实面试题 | 25 | ⬜ | |

状态：⬜ 未开始 / 🟨 进行中 / ✅ 完成 / 🔁 需要复习

## 建议节奏（5 周）

| 周 | 内容 | 目标 |
|----|------|------|
| 第 1 周 | roadmap + 初级 Q1–Q30 | 能用 Standalone + Signals 写出带路由、表单、HTTP 的小应用 |
| 第 2 周 | 中级 Q1–Q18 + 手写 H1、H5 | 讲清变更检测、Zoneless、Signals、RxJS；手写迷你 Signals |
| 第 3 周 | 中级 Q19–Q32 + 手写 H2–H4、H6–H8 | DI 进阶、路由与表单进阶、SSR、测试；写自定义控件、拦截器、指令 |
| 第 4 周 | 高级 Q1–Q26 + 手写 H9–H12 | 底层原理、性能排查、架构与状态管理、升级迁移 |
| 第 5 周 | 系统设计 S1–S8 + 真实面试题 R1–R25 | 模拟面试，准备 1–2 个 Angular 性能优化或架构改进的项目案例 |

**建议配合实操**：用 `ng new` 创建一个 v22 项目，用 Angular DevTools 的 Profiler 观察 OnPush 和 Signals 对变更检测的影响；尝试把一个 Zone.js + `@Input` 的组件迁移为 Signals + Zoneless。

## 薄弱点记录

| 日期 | 题号 | 问题 | 补充学习 |
|------|------|------|----------|
| | | | |

## 下次复习

- [ ] 变更检测、OnPush 的触发条件、Zoneless（中级 Q1–Q5）
- [ ] Signals 的原理与 effect 的使用边界（中级 Q7–Q12、手写 H1）
- [ ] 依赖注入的解析规则（中级 Q19–Q21）

## 模拟面试自检

- [ ] 能讲清变更检测的流程，以及 Signals 如何实现局部刷新
- [ ] 能解释 Zone.js 的原理和去掉它的理由
- [ ] 能手写一个带 computed 和 effect 的迷你 Signals
- [ ] 能根据场景选择 RxJS 的高阶映射操作符
- [ ] 能讲清依赖注入的解析过程和组件级 providers 的用途
- [ ] 能手写自定义表单控件、令牌刷新拦截器、结构指令
- [ ] 能给出大型应用卡顿的排查思路和优化手段
- [ ] 能规划一个老 Angular 项目的升级和现代化路线
- [ ] 能设计一个大型中后台平台的架构
