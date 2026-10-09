# Node.js 学习路线（高级工程师面试向）

## 面试画像

国内大厂对 Node.js 的考察分两类岗位：

- **前端 / 全栈（BFF、SSR、工具链）**：事件循环、模块系统、Stream、中间件框架、性能与稳定性
- **后端（服务端 Node）**：在上述基础上，加上高并发调优、数据库与缓存、消息队列、分布式一致性、可观测性、安全

高级岗不再问「Node 是什么」，而是问场景：

> 「发布后 P99 延迟翻倍，你怎么排查？」「服务内存缓慢上涨直到 OOM，怎么定位？」「如何不中断请求地发布？」

## 版本说明（截至 2026-10）

| 版本 | 状态 | 关键内容 |
|------|------|----------|
| Node 22 | Maintenance LTS（至 2027-04） | `require(esm)`、内置 WebSocket 客户端、`fs.glob`、`node --run`、watch 模式稳定、Stream 默认 highWaterMark 提升为 64 KiB |
| Node 24 | Active LTS → 2026-10-20 转 Maintenance（至 2028-04） | 全局 URLPattern、Permission Model 稳定、AsyncLocalStorage 基于 AsyncContextFrame、npm 11、TypeScript 类型擦除（type stripping）默认可用 |
| Node 26 | Current → 2026-10-28 进入 LTS（至 2029-04） | Temporal 默认启用、V8 14.6、Undici 8、`Map.getOrInsert`、`Iterator.concat`；移除 `_stream_*` 旧模块和 `writeHeader` |
| Node 27+ | 新发布节奏 | 每年 4 月发布一个大版本，10 月进入 LTS，**不再区分奇偶，每个版本都是 LTS**；2026-10 起有 Alpha 通道 |

> 生产只用 LTS；面试中能说出「我们的升级策略」是加分项。

## 知识地图

```text
Node.js
├─ 运行时：V8、libuv、bindings、线程池、单线程误区
├─ 事件循环：6 阶段、nextTick、微任务、setImmediate、阻塞检测
├─ 模块：CommonJS、ESM、require(esm)、exports 字段、双包问题
├─ 核心模块：fs、path、http、events、stream、buffer、crypto、child_process
├─ Stream：四种流、背压、pipeline、Web Streams、异步迭代
├─ 并发：cluster、worker_threads、child_process、线程池调优
├─ Web 框架：Express、Koa、Fastify、NestJS、中间件、洋葱模型
├─ 工程：TypeScript、测试（node:test）、日志、配置、调试
├─ 稳定性：错误处理、优雅关闭、超时、重试、熔断、限流
├─ 性能：CPU Profile、火焰图、内存分析、事件循环延迟、连接池
├─ 数据：数据库连接池、Redis 缓存、消息队列、分布式锁、幂等
├─ 安全：注入、原型污染、ReDoS、SSRF、路径穿越、供应链、Permission Model
└─ 架构：BFF、SSR 服务、微服务、Serverless、可观测性（OpenTelemetry）
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 运行时基础、模块、核心 API、包管理
2. [中级 Q&A](20-mid-qa.md) — 事件循环、Stream、并发模型、框架、错误处理、认证、缓存
3. [高级 Q&A](30-senior-qa.md) — libuv、内存与 CPU 分析、高并发调优、分布式、安全、架构
4. [手写与现场编码](40-coding-handwritten.md) — 13 道高频手写题
5. [系统设计与场景](50-system-design.md) — 短链、BFF、推送、任务队列、SSR 服务等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. 「Node 是单线程」的准确表述：JS 执行单线程 + libuv 线程池 + 系统异步 I/O + Worker
2. 事件循环 6 个阶段，nextTick / Promise / setImmediate / setTimeout 顺序
3. 哪些操作走线程池（fs、dns.lookup、crypto、zlib），线程池大小如何影响性能
4. 阻塞事件循环的常见原因与检测（事件循环延迟监控）
5. Stream 背压原理，为什么用 `pipeline`
6. CommonJS 加载原理、模块缓存、循环依赖；ESM 与 CJS 互操作
7. cluster、worker_threads、child_process 的区别与选型
8. 错误处理：同步/异步错误、`unhandledRejection`、进程崩溃策略
9. 优雅关闭：停止接收、处理存量请求、关闭连接池、超时强退
10. AsyncLocalStorage 实现请求上下文和链路追踪
11. 内存泄漏排查：heap snapshot 对比、常见泄漏源
12. CPU 性能分析：`--cpu-prof`、火焰图
13. HTTP keep-alive 与负载均衡超时配置（502 经典问题）
14. 缓存、限流、熔断、重试、幂等
15. 数据库连接池与 N+1 问题
16. 安全：注入、ReDoS、SSRF、路径穿越、原型污染、依赖供应链
17. 可观测性：结构化日志、指标、分布式追踪
18. 部署：容器内一进程、内存限制、健康检查、零停机发布

## 相关技术

- [JavaScript](../javascript/00-roadmap.md)：事件循环、Promise、模块
- [Docker](../docker/00-roadmap.md)：Node 镜像构建、容器内存限制、优雅关闭信号
- [React](../react/00-roadmap.md)：SSR / RSC 服务端渲染

## 自检清单

- [ ] 能画出 Node 架构图并解释一次 `fs.readFile` 的完整链路
- [ ] 能独立完成事件循环输出题
- [ ] 能用 pipeline 写一个处理 GB 级文件的脚本且内存平稳
- [ ] 能手写 Koa 中间件框架和限流中间件
- [ ] 能讲清一次线上内存泄漏或 CPU 飙高的排查过程
- [ ] 能设计一个支持水平扩展的 WebSocket 推送服务
