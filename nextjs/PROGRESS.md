# Next.js 学习进度

## 完成度

| 文件 | 内容 | 数量 | 状态 | 完成日期 |
|------|------|------|------|----------|
| [00-roadmap](./00-roadmap.md) | 路线图、版本说明 | - | ⬜ | |
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
| 第 1 周 | roadmap + 初级 Q1–Q30 | 能用 App Router 搭建完整应用，讲清服务端 / 客户端组件 |
| 第 2 周 | 中级 Q1–Q17 | 讲清 RSC 原理、流式渲染、缓存体系（分版本）、Server Actions |
| 第 3 周 | 中级 Q18–Q32 + 手写 H1–H6 | 掌握鉴权、国际化、升级变化，写出安全的 Action 和缓存代码 |
| 第 4 周 | 高级 Q1–Q26 + 手写 H7–H12 | Cache Components 深入、自托管、安全事件、迁移和架构 |
| 第 5 周 | 系统设计 S1–S8 + 真实面试题 R1–R25 | 模拟面试，准备 2–3 个项目案例 |

## 薄弱点记录

| 日期 | 题号 | 问题 | 补充学习 |
|------|------|------|----------|
| | | | |

## 下次复习

- [ ] 缓存体系：旧模型四层 vs Cache Components（20-mid Q10–Q14）
- [ ] Server Actions 原理与安全（20-mid Q15–Q16）
- [ ] 两个安全事件（30-senior Q14–Q15）
- [ ] 自托管与版本偏差（30-senior Q11–Q12）

## 模拟面试自检

- [ ] 能在 1 分钟内讲清 RSC 和 SSR 的区别
- [ ] 回答缓存问题时会先说明版本
- [ ] 能画出一次请求从服务端渲染到 hydration 的流程
- [ ] 能讲清 updateTag、revalidateTag、refresh 的区别和使用场景
- [ ] 能写出带鉴权、校验、权限检查的 Server Action
- [ ] 能说出 CVE-2025-29927 和 React2Shell 的原理与教训
- [ ] 能为一个电商详情页设计渲染和缓存策略
- [ ] 能讲出自托管多实例需要注意的 4 件事
- [ ] 准备好一个 Next.js 性能或线上问题的项目案例
