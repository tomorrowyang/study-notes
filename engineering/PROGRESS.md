# 前端工程化学习进度

## 完成度

| 文件 | 内容 | 数量 | 状态 | 完成日期 |
|------|------|------|------|----------|
| [00-roadmap](./00-roadmap.md) | 路线图、工具现状（pnpm 11、MF 2.0 等） | - | ⬜ | |
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
| 第 1 周 | roadmap + 初级 Q1–Q30 | 掌握包管理、Monorepo 基础、代码规范、CI/CD 和微前端的基本概念 |
| 第 2 周 | 中级 Q1–Q17 + 手写 H1–H4、H8 | 讲清任务图、缓存、受影响分析、CI 安全与灰度；手写任务调度器和 semver |
| 第 3 周 | 中级 Q18–Q32 + 手写 H5–H7、H11 | 讲清沙箱、HTML Entry、wujie、Module Federation；手写沙箱和加载器 |
| 第 4 周 | 高级 Q1–Q26 + 手写 H9、H10、H12 | 体系建设、供应链安全、发布平台、微前端治理、效能度量 |
| 第 5 周 | 系统设计 S1–S8 + 真实面试题 R1–R25 | 模拟面试，准备 1–2 个工程化改进的项目案例（带量化数据） |

**建议配合实操**：用 pnpm workspace + Turborepo 搭一个包含 2 个应用和 2 个共享包的 Monorepo，配置 GitHub Actions（固定 SHA、最小权限、`--affected`），观察缓存命中；再用 qiankun 或 Module Federation 接入一个子应用，故意制造 React 版本不一致，观察 shared 的行为。

## 薄弱点记录

| 日期 | 题号 | 问题 | 补充学习 |
|------|------|------|----------|
| | | | |

## 下次复习

- [ ] 任务缓存的键与错误命中、远程缓存投毒（中级 Q2–Q3）
- [ ] 代理沙箱的实现与局限（中级 Q19–Q20、手写 H5）
- [ ] CI/CD 供应链安全（中级 Q11、高级 Q5）

## 模拟面试自检

- [ ] 能讲清 pnpm 的存储结构和它如何解决幽灵依赖
- [ ] 能讲清 Turborepo / Nx 的任务图、缓存键和受影响分析
- [ ] 能手写带并发限制的拓扑任务调度器
- [ ] 能写出安全的 GitHub Actions 工作流并解释每一项配置
- [ ] 能设计前端灰度发布和一键回滚
- [ ] 能手写代理沙箱，并说清它防不住什么
- [ ] 能比较 qiankun、wujie、Module Federation、iframe 并给出选型理由
- [ ] 能用 DORA 指标说明一次工程化改进的收益
- [ ] 能设计一个 50 个应用规模的 Monorepo 平台
