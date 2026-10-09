# 前端工程化学习路线（Monorepo、CI/CD、微前端）

## 面试画像

工程化是高级前端岗位与初中级拉开差距的核心领域，面试不再只问「会不会用」，而是问「怎么为团队设计和落地」：

- **基础**：包管理器（npm / pnpm）、锁文件、语义化版本、代码规范工具链、Git 工作流、CI/CD 基本概念、部署与缓存策略、微前端的基本方案 —— 初中级必问
- **原理与方案对比**：Monorepo 的任务图与缓存、受影响分析、Turborepo vs Nx、版本发布（Changesets）；CI 加速、灰度发布、回滚；微前端的 JS 沙箱、样式隔离、HTML Entry、Module Federation 的共享依赖 —— 中高级区分点
- **体系建设**：从零搭建工程化体系、Polyrepo 迁移到 Monorepo、大规模仓库的性能、CI/CD 供应链安全、发布平台与灰度系统、微前端的治理与演进、研发效能度量 —— 高级岗必谈

高级岗典型问法：

> 「你们为什么用 Monorepo？遇到了什么问题？」「Turborepo 的缓存是怎么判断命中的？」「CI 从 30 分钟优化到 5 分钟，你会怎么做？」「前端怎么做灰度发布？出问题怎么回滚？」「qiankun 的 JS 沙箱是怎么实现的？能防住什么？」「Module Federation 共享依赖的版本冲突怎么处理？」「微前端和 Monorepo 怎么选？」

## 版本与现状（截至 2026-10）

| 领域 | 工具 / 事件 | 现状 |
|------|-------------|------|
| 包管理 | **pnpm 11**（2026-04） | 要求 Node.js 22+；存储索引改为 SQLite；配置从 `.npmrc` 迁移到 `pnpm-workspace.yaml`；`allowBuilds` 取代原来的构建脚本白名单设置；默认启用依赖的发布冷却期等供应链防护（细节以官方为准） |
| 包管理 | npm | 2025-12 起 classic token 被吊销，推荐可信发布（OIDC）；npm v12 起默认不执行依赖的安装脚本（详见[前端安全专题](../frontend-security/00-roadmap.md)） |
| Monorepo | Turborepo 2.x、Nx 22.x | Turborepo 专注任务编排与缓存；Nx 提供生成器、模块边界、分布式执行；pnpm `catalogs` 统一依赖版本 |
| 版本发布 | Changesets | Monorepo 包发布的主流方案 |
| 微前端 | **Module Federation 2.0 稳定** | 运行时与构建工具解耦，支持 Webpack、Rspack、Rollup、Rolldown、Rsbuild、Vite、Metro；动态类型提示；Node.js 支持 |
| 微前端 | qiankun 3.0 | 仍处于 RC 阶段（loader / sandbox / shared 拆分、流式加载）；生产中大量使用的仍是 2.x |
| 微前端 | wujie、micro-app、Native Federation | wujie 基于 iframe + Web Components；Native Federation 基于导入映射（Angular 生态常用） |
| CI 安全 | GitHub Actions | 2025-03 `tj-actions/changed-files` 标签被篡改（CVE-2025-30066）→ **第三方 Action 固定到完整提交 SHA**；2026-06 `actions/checkout` v7 默认拒绝在 `pull_request_target` 中检出 fork 的代码（pwn request 防护），并回溯到旧主版本 |
| 构建 | Vite、Rspack、Rolldown | 详见 [Vite](../vite/00-roadmap.md)、[Webpack](../webpack/00-roadmap.md) 专题 |

> 版本和政策变化很快，以各项目的官方文档和变更日志为准。

## 知识地图

```text
前端工程化
├─ 依赖管理：npm / yarn / pnpm、锁文件、semver、依赖类型、幽灵依赖、overrides、
│           供应链防护（锁文件、禁止安装脚本、冷却期）
├─ Monorepo：
│   ├─ 工作区：pnpm workspace、workspace: 协议、catalogs
│   ├─ 任务编排：任务图、拓扑排序、并行、缓存（本地 / 远程）、受影响分析
│   ├─ 工具：Turborepo、Nx、Rush、Lerna（历史）
│   ├─ 架构：模块边界、内部包（源码引用 vs 预构建）、TS project references
│   └─ 发布：Changesets、固定 / 独立版本、CHANGELOG
├─ 代码质量：ESLint（flat config）、Prettier / Biome、TypeScript 严格模式、
│           Git Hooks（husky、lint-staged）、Conventional Commits、代码评审
├─ CI/CD：
│   ├─ 概念：持续集成、持续交付、持续部署
│   ├─ 流水线：安装 → 检查 → 测试 → 构建 → 部署；缓存、并行、分片
│   ├─ 部署：静态资源 + CDN、SSR 服务、容器化；缓存策略（哈希文件名、HTML 不缓存）
│   ├─ 发布策略：滚动、蓝绿、金丝雀 / 灰度、特性开关、回滚
│   ├─ 环境：预览环境、配置与密钥管理、构建一次多处部署
│   └─ 安全：Action 固定 SHA、最小权限、OIDC、pwn request、SLSA / provenance
├─ 微前端：
│   ├─ 方案：iframe、single-spa / qiankun、wujie、micro-app、Web Components、Module Federation
│   ├─ 核心问题：JS 沙箱、样式隔离、HTML Entry、路由、通信、共享依赖、预加载
│   └─ 治理：版本契约、独立发布、监控、降级
└─ 研发效能：DORA 指标、脚手架与生成器、codemod、可观测性（sourcemap、版本标记）
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 包管理、Monorepo 基础、代码规范、CI/CD 基础、部署缓存、微前端基础
2. [中级 Q&A](20-mid-qa.md) — 任务图与缓存、受影响分析、版本发布、CI 加速与安全、灰度发布、JS 沙箱、Module Federation 深入
3. [高级 Q&A](30-senior-qa.md) — 工程化体系建设、大规模 Monorepo、CI/CD 供应链安全、发布平台、微前端治理、效能度量
4. [手写与现场编码](40-coding-handwritten.md) — 12 道题：任务调度器、缓存键、受影响分析、semver、JS 沙箱、HTML Entry、灰度分桶、工作流配置等
5. [系统设计与场景](50-system-design.md) — Monorepo 平台、CI/CD 平台、灰度发布系统、微前端基座、远程缓存服务等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. npm / yarn / pnpm 的区别，pnpm 的硬链接 + 符号链接结构，**幽灵依赖**和**依赖分身**
2. 锁文件的作用，`npm install` 和 `npm ci` 的区别，semver 范围（`^` / `~`）
3. Monorepo 与 Polyrepo 的取舍，Monorepo 解决了什么、带来了什么
4. 任务图：依赖拓扑排序、并行执行、`dependsOn: ["^build"]` 的含义
5. **缓存的原理**：输入（源码、依赖、环境变量、配置）→ 哈希 → 命中则恢复输出和日志；远程缓存
6. 受影响分析（affected）：从变更文件推导受影响的项目
7. Turborepo 和 Nx 的区别与选型
8. Monorepo 的包发布：Changesets、固定版本 vs 独立版本
9. CI 加速：依赖缓存、任务缓存、并行与分片、只跑受影响的任务
10. 部署与缓存：**文件名带内容哈希 + 长期缓存，HTML 不缓存**；先传资源再切 HTML
11. 发布策略：蓝绿、金丝雀、灰度（按用户 / 比例 / 地域）、特性开关、快速回滚
12. CI/CD 安全：第三方 Action 固定 SHA、`GITHUB_TOKEN` 最小权限、OIDC 替代长期密钥、pwn request
13. 微前端解决什么问题，什么时候**不该**用
14. 微前端方案对比：iframe、qiankun、wujie、micro-app、Module Federation
15. **JS 沙箱**：快照沙箱、代理沙箱的实现与局限；样式隔离（Shadow DOM、作用域 CSS）
16. HTML Entry 的原理（拉取 HTML、解析脚本和样式、在沙箱中执行）
17. Module Federation 的共享依赖：`singleton`、`requiredVersion`、版本协商
18. 研发效能度量（DORA 四项指标）和工程化体系的落地路径

## 相关技术

- [Webpack](../webpack/00-roadmap.md) / [Vite](../vite/00-roadmap.md)：构建原理、Module Federation、插件
- [Node.js](../node/00-roadmap.md)：CLI 工具、脚本、模块解析
- [前端安全](../frontend-security/00-roadmap.md)：供应链安全、npm 可信发布、微前端的隔离边界
- [Docker](../docker/00-roadmap.md)：容器化部署、多阶段构建
- [性能优化](../performance/00-roadmap.md)：构建产物优化、缓存策略
- [Angular](../angular/00-roadmap.md)：Nx、Native Federation

## 自检清单

- [ ] 能解释 pnpm 的 node_modules 结构，以及它如何避免幽灵依赖
- [ ] 能讲清 Turborepo / Nx 的缓存是怎么判断命中的，哪些情况会导致错误的缓存命中
- [ ] 能给出一个 CI 提速方案并估算收益
- [ ] 能讲清静态资源部署的缓存策略，以及发布时如何避免新旧版本混用
- [ ] 能设计一个前端灰度发布方案，并说明如何回滚
- [ ] 能讲清 GitHub Actions 的主要安全风险和防护措施
- [ ] 能手写一个简单的代理沙箱，并说明它防不住什么
- [ ] 能讲清 Module Federation 的共享依赖机制和版本冲突的处理
- [ ] 能根据团队情况在 Monorepo 和微前端之间做出选择并说明理由
