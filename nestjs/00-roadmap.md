# NestJS 学习路线（高级工程师面试向）

## 面试画像

国内大厂考察 NestJS 的岗位主要是 **Node 全栈 / BFF / 中台后端**。面试通常分三层：

- **框架机制**：模块、依赖注入、Provider 作用域、请求生命周期（中间件 → 守卫 → 拦截器 → 管道 → 处理器 → 拦截器 → 异常过滤器）—— 几乎必问
- **原理**：IoC 容器如何基于 `reflect-metadata` 解析依赖、装饰器元数据、动态模块、循环依赖、生命周期钩子顺序 —— 中高级区分点
- **工程与架构**：鉴权与 RBAC、配置与校验、数据库事务、缓存与限流、微服务与消息队列、可观测性、测试、性能、多租户、优雅关闭 —— 高级岗必谈

高级岗典型问法：

> 「一个请求进来，在 Nest 里经过了哪些环节？抛异常后又怎么走？」「依赖注入是怎么实现的？为什么要开 `emitDecoratorMetadata`？」「请求作用域有什么性能问题？怎么做多租户？」「Service 之间循环依赖怎么解决？」「事务怎么跨多个 Service 传递？」

## 版本说明（截至 2026-10）

| 版本 | 时间 | 关键内容 |
|------|------|----------|
| v10 | 2023-06 | SWC 构建支持（`nest start -b swc`）、`CacheModule` 移到 `@nestjs/cache-manager`、Node 16+ |
| v11 | 2025-01 | **Node 20+**、**Express 5 / Fastify 5**（路由通配符必须命名：`*splat`，可选参数改用 `{}`）、`ConsoleLogger` 支持 JSON 输出、`cache-manager` v6 基于 Keyv（TTL 单位为毫秒）、`ParseDatePipe`、启动性能优化；11.1.4 起支持 `instrument` 选项 |
| **v12** | **2026-08** | **核心包改为 ESM 发布**（CJS 项目借助 Node 的 `require(esm)` 继续可用，不强制迁移）；要求 **Node 20.19+ / 22.12+**；**Standard Schema 支持**：`@Body({ schema })` + `StandardSchemaValidationPipe`、`StandardSchemaSerializerInterceptor`，可直接使用 Zod / Valibot / ArkType；官方可观测性 SDK **`@nestjs/observe`**；CLI 重构，webpack 废弃改用 **Rspack**；新建 ESM 项目默认 **Vitest + oxlint**；生命周期钩子按组件层级调用；`@nestjs/config` 通过 Standard Schema 校验；`ConsoleLogger` 默认开启结构化参数；NATS 改用 `@nats-io/transport-node`、GraphQL 订阅只支持 `graphql-ws`、默认 IDE 改为 GraphiQL |

> v11 已进入 npm 的 `legacy` 标签。面试中能说出「v11 的 Express 5 路由变化」和「v12 的 ESM 与 Standard Schema」是加分项。细节以官方迁移指南为准。

## 知识地图

```text
NestJS
├─ 核心概念：Module、Controller、Provider、DI 容器、装饰器
├─ 依赖注入：Token、useClass/useValue/useFactory/useExisting、@Inject、@Optional、作用域、forwardRef、ModuleRef
├─ 模块系统：imports/exports/providers、全局模块、动态模块（forRoot/forRootAsync/register）、ConfigurableModuleBuilder、懒加载模块
├─ 请求生命周期：Middleware → Guard → Interceptor(前) → Pipe → Handler → Interceptor(后) → Exception Filter
├─ 执行上下文：ArgumentsHost、ExecutionContext、Reflector、自定义装饰器、applyDecorators
├─ 数据：ValidationPipe + class-validator、Standard Schema（v12）、序列化、DTO、Swagger
├─ 数据库：TypeORM / Prisma / Drizzle / MikroORM、Repository、事务、CLS 上下文
├─ 安全：Passport、JWT、RBAC / ABAC（CASL）、Throttler 限流、Helmet、CORS、CSRF
├─ 基础设施：Config、Cache（Keyv）、Schedule、BullMQ 队列、EventEmitter、CQRS、Terminus 健康检查
├─ 通信：WebSocket Gateway、微服务（TCP/Redis/NATS/Kafka/RMQ/gRPC）、混合应用、GraphQL
├─ 原理：reflect-metadata、design:paramtypes、NestContainer、Injector、InstanceWrapper、模块扫描与实例化、生命周期钩子
├─ 工程：测试（TestingModule、Vitest/Jest、e2e）、日志、Monorepo、SWC / Rspack、ESM 迁移
└─ 生产：Fastify 适配器、请求作用域性能、优雅关闭、多租户、可观测性（OpenTelemetry / @nestjs/observe）、容器化
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 模块、控制器、Provider、DI 基础、管道守卫拦截器入门、配置、DTO 校验
2. [中级 Q&A](20-mid-qa.md) — 完整请求生命周期、执行上下文与元数据、自定义 Provider、作用域、动态模块、循环依赖、鉴权、数据库、缓存、队列、测试
3. [高级 Q&A](30-senior-qa.md) — IoC 容器原理、模块扫描与实例化、请求作用域性能与多租户、事务传播、微服务、可观测性、性能、架构取舍、v12 升级
4. [手写与现场编码](40-coding-handwritten.md) — 12 道题：迷你 IoC 容器、守卫、拦截器、动态模块、事务装饰器等
5. [系统设计与场景](50-system-design.md) — BFF 网关、多租户 SaaS、权限系统、订单服务、实时推送、异步任务平台等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. Nest 解决了什么问题：为 Node 提供 Angular 风格的模块化 + IoC 架构，底层可切换 Express / Fastify
2. 完整请求生命周期的顺序，以及每一层的职责边界
3. 守卫、拦截器、管道、过滤器分别在全局、控制器、路由三个级别的执行顺序
4. DI 原理：`emitDecoratorMetadata` 生成 `design:paramtypes`，容器据此按 Token 解析依赖
5. 四种自定义 Provider 与 Injection Token；接口无法作为 Token 的原因
6. 三种作用域（DEFAULT / REQUEST / TRANSIENT），请求作用域沿注入链向上冒泡，以及 durable provider
7. 动态模块 `forRoot` / `forRootAsync` / `register` 的区别，`ConfigurableModuleBuilder`
8. 循环依赖：成因、`forwardRef`、更好的解法（拆分、事件）
9. 生命周期钩子的顺序与 `enableShutdownHooks`；v12 改为按层级调用
10. `ExecutionContext` + `Reflector` 实现元数据驱动的鉴权（`@Public`、`@Roles`）
11. 异常过滤器的匹配规则，统一错误响应
12. 参数校验：`ValidationPipe` 的 `whitelist` / `transform`；v12 的 Standard Schema
13. 事务如何跨 Service 传递（CLS / AsyncLocalStorage）
14. 缓存、限流在多实例下的问题（Redis 存储）
15. 微服务：`@MessagePattern` 与 `@EventPattern` 的区别，传输层选型
16. 测试：`Test.createTestingModule`、`overrideProvider`、e2e
17. 性能：Fastify、避免请求作用域、SWC、懒加载模块；启动速度与 Serverless
18. 生产实践：优雅关闭、健康检查、结构化日志、链路追踪、多租户

## 相关技术

- [Node.js](../node/00-roadmap.md)：事件循环、Stream、AsyncLocalStorage、优雅关闭、性能排查
- [TypeScript](../typescript/00-roadmap.md)：装饰器（Nest 使用旧版实验性装饰器）、`emitDecoratorMetadata`、模块解析、ESM
- [Docker](../docker/00-roadmap.md)：Nest 服务的镜像构建、信号处理、健康检查

## 自检清单

- [ ] 能画出 Nest 的请求生命周期图，并说出抛异常后的走向
- [ ] 能解释 DI 容器如何拿到构造函数参数类型
- [ ] 能手写一个基于元数据的角色守卫和一个响应包装拦截器
- [ ] 能写出一个支持 `forRootAsync` 的动态模块
- [ ] 能讲清请求作用域的性能问题和多租户方案
- [ ] 能讲清事务跨 Service 传递和分布式场景下的一致性方案
- [ ] 能讲清 v11 / v12 的升级要点
