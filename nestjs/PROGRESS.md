# NestJS 学习进度

## 完成度

- [ ] 00-roadmap
- [ ] 10-junior-qa（30 题）
- [ ] 20-mid-qa（32 题）
- [ ] 30-senior-qa（26 题）
- [ ] 40-coding-handwritten（12 题）
- [ ] 50-system-design（8 题）
- [ ] 90-interview-real（25 题）

## 建议节奏（可改）

| 阶段 | 内容 | 预计 |
|------|------|------|
| 第 1 周 | roadmap + 10 全部，动手搭一个带 ValidationPipe、Swagger、TypeORM/Prisma 的 CRUD 项目 | |
| 第 2 周 | 20 的生命周期、元数据、Provider 与模块、作用域 + H2/H3/H4/H5 | |
| 第 3 周 | 20 的认证授权、校验、事务、缓存、微服务、工程实践 + H6/H8/H9/H12 | |
| 第 4 周 | 30 的容器原理、上下文、错误体系、架构、分布式一致性 + H1/H10/H11 | |
| 第 5 周 | 30 的可观测性、性能、稳定性、v12 升级 + H7 + 50 口述 + 90 模拟面试 | |

## 薄弱点（自己填）

- 

## 下次复习

- 日期：
- 重点：
- 重做错题：

## 模拟面试自检

- [ ] 能不看资料画出完整请求生命周期，并解释过滤器为什么从路由级开始查找
- [ ] 能讲清中间件、守卫、拦截器、管道、过滤器各自的职责和边界
- [ ] 能讲清 `design:paramtypes` 与 IoC 容器原理，并手写简化容器
- [ ] 能讲清三种作用域、请求作用域的冒泡问题，以及 CLS 和 durable provider 替代方案
- [ ] 能独立写出 `@Public()` + 全局 JWT 守卫 + `@Roles()` 角色守卫
- [ ] 能用 `ConfigurableModuleBuilder` 写一个支持 `forRootAsync` 的动态模块
- [ ] 能写出统一响应拦截器和全局异常过滤器，并说明领域错误与 HttpException 的取舍
- [ ] 能讲清跨 Service 事务的实现（AsyncLocalStorage + `@Transactional`）
- [ ] 能设计幂等下单接口，说清并发场景和兜底手段
- [ ] 能讲清 Outbox + 幂等消费 + Saga 的分布式一致性方案
- [ ] 能讲一次 P99 延迟或内存泄漏的完整排查过程
- [ ] 能讲清 Nest 11（Express 5）和 Nest 12（ESM、Standard Schema）的升级要点
- [ ] 能用 3 分钟讲清 BFF 或多租户 SaaS 的设计，并落到 Nest 的具体机制上
