# NestJS 真实面试题汇总

> 根据近年公开面经、社区高频题和国内 Node 后端 / 全栈岗位面试趋势整理。`来源倾向` 表示题目常见的出现场景，不代表某公司的固定题库。题干尽量保留面试中的真实问法。

---

## 一、基础与核心概念

### R1. NestJS 和 Express / Koa 有什么区别？为什么要用 Nest？

- **难度**: junior
- **标签**: 框架定位
- **来源倾向**: 一面开场必问，常追问「Nest 有什么缺点」

**答案**

Express / Koa 是 HTTP 库，只提供路由和中间件，项目结构由团队自己约定；Nest 是构建在 Express（默认）或 Fastify 之上的应用框架，提供模块化、依赖注入、守卫/管道/拦截器/过滤器等 AOP 机制，以及微服务、WebSocket、GraphQL 的统一抽象，适合中大型团队项目。缺点：学习曲线陡、装饰器和元数据较「魔法」、启动和抽象有开销；简单脚本、极致性能的小服务或 Serverless 冷启动敏感场景不一定适合。

详见：[10-junior-qa Q1](10-junior-qa.md)、[10-junior-qa Q24](10-junior-qa.md)、[30-senior-qa Q25](30-senior-qa.md)

---

### R2. 说一下 Nest 的请求生命周期，一个请求进来依次经过哪些环节？

- **难度**: mid
- **标签**: 生命周期
- **来源倾向**: 中级以上几乎必问，常让手画顺序或追问「过滤器为什么是反的」

**答案**

中间件（全局 → 模块）→ 守卫（全局 → 控制器 → 路由）→ 拦截器前半段（全局 → 控制器 → 路由）→ 管道（全局 → 控制器 → 路由 → 参数）→ 处理器 → 拦截器后半段（路由 → 控制器 → 全局，与进入顺序相反）→ 任一环节抛异常时进入异常过滤器（路由 → 控制器 → 全局，只执行第一个匹配的）。

**口述模板（30–60 秒）**

1. **一句话结论**：中间件 → 守卫 → 拦截器 → 管道 → 处理器 → 拦截器（返回）→ 异常过滤器。
2. **关键机制**：守卫、拦截器、管道都按全局、控制器、路由的顺序执行；拦截器基于 Observable，是洋葱模型，所以返回阶段的顺序相反。
3. **注意点**：过滤器是唯一从最具体开始查找的组件，路由级优先，并且只有第一个匹配的过滤器会执行，异常不会继续传给下一个过滤器。
4. **可追问时补充**：管道在拦截器内部执行，所以拦截器的 `catchError` 能捕获管道和处理器的异常；守卫在拦截器之前，所以守卫拒绝的请求不会经过拦截器的日志逻辑。

详见：[20-mid-qa Q1](20-mid-qa.md)、[20-mid-qa Q2](20-mid-qa.md)、[40-coding-handwritten H11](40-coding-handwritten.md)

---

### R3. 中间件、守卫、拦截器、管道、过滤器分别适合做什么？鉴权为什么放守卫而不是中间件？

- **难度**: mid
- **标签**: 组件职责
- **来源倾向**: 高频，考察是否真正用过

**答案**

中间件：与路由无关的通用处理（requestId、CORS、body 解析）；守卫：决定是否放行（认证、授权）；拦截器：在处理器前后附加逻辑（日志、响应包装、缓存、超时）；管道：参数校验和转换；过滤器：把异常转换为响应。鉴权放守卫是因为守卫可以拿到 `ExecutionContext`，知道即将执行哪个处理器，从而读取 `@Roles()`、`@Public()` 等元数据；中间件不知道目标处理器。

**口述模板（30–60 秒）**

1. **一句话结论**：按「能拿到什么信息」和「在哪个阶段」来分配职责。
2. **关键机制**：中间件最早执行但只有 req/res；守卫有执行上下文，可以读取处理器元数据；拦截器包裹处理器，能处理前后两端；管道只关心单个参数。
3. **注意点**：鉴权放守卫；参数校验放管道；响应包装放拦截器；错误格式统一放过滤器。
4. **可追问时补充**：完整的访问日志放中间件监听 `finish` 事件，否则守卫拒绝和 404 的请求会漏记。

详见：[20-mid-qa Q22](20-mid-qa.md)、[10-junior-qa Q13](10-junior-qa.md)

---

### R4. Nest 的依赖注入是怎么实现的？`@Injectable()` 到底做了什么？

- **难度**: senior
- **标签**: DI, 原理
- **来源倾向**: 高级岗原理题，常追问「为什么 interface 不能直接注入」

**答案**

TS 开启 `emitDecoratorMetadata` 后，带装饰器的类会被输出 `design:paramtypes` 元数据，即构造函数参数的类型列表。Nest 启动时扫描模块树，为每个 Provider 创建 InstanceWrapper，再由 Injector 按参数类型（或 `@Inject` 指定的 Token）在当前模块及其导入模块中查找依赖，递归实例化并缓存。`@Injectable()` 的主要作用是让类有装饰器从而产生元数据，并标记为可注入。interface 在运行时不存在，元数据是 `Object`，所以必须使用字符串/Symbol Token 或抽象类。

**口述模板（30–60 秒）**

1. **一句话结论**：基于 TS 装饰器元数据 + 启动时扫描模块树的 IoC 容器。
2. **关键机制**：`design:paramtypes` 提供构造函数参数类型；Scanner 收集模块和 Provider，InstanceLoader 按依赖关系实例化，默认单例缓存。
3. **注意点**：interface 和联合类型无法作为 Token；循环 import 会导致元数据中类型为 undefined。
4. **可追问时补充**：模块有可见性边界，依赖必须在当前模块声明或从导入模块导出；我可以手写一个简化的容器演示。

详见：[10-junior-qa Q6](10-junior-qa.md)、[30-senior-qa Q1](30-senior-qa.md)、[40-coding-handwritten H1](40-coding-handwritten.md)

---

### R5. Provider 的作用域有哪些？请求作用域有什么性能问题？

- **难度**: mid
- **标签**: 作用域
- **来源倾向**: 高频，常追问「怎么拿到当前用户而不用请求作用域」

**答案**

DEFAULT（单例）、REQUEST（每个请求一个实例）、TRANSIENT（每个注入方一个实例）。请求作用域会沿依赖链向上冒泡：依赖它的 Service、Controller 都变成请求作用域，每个请求都要重新实例化整条链，增加延迟和 GC 压力。替代方案：用 AsyncLocalStorage（nestjs-cls）传递请求上下文，Provider 保持单例；多租户场景可以用 durable provider 按租户复用实例。

**口述模板（30–60 秒）**

1. **一句话结论**：默认单例，请求作用域是有传染性的性能陷阱。
2. **关键机制**：REQUEST 作用域会沿注入链冒泡，整条链按请求重建。
3. **注意点**：需要请求上下文时优先用 CLS，单例中注入请求作用域的依赖会让单例也变成请求作用域。
4. **可追问时补充**：durable provider 通过 `ContextIdStrategy` 把请求分组到租户维度的子树，同一租户复用实例。

详见：[20-mid-qa Q7](20-mid-qa.md)、[30-senior-qa Q3](30-senior-qa.md)、[30-senior-qa Q4](30-senior-qa.md)

---

### R6. 遇到循环依赖怎么办？

- **难度**: mid
- **标签**: 循环依赖
- **来源倾向**: 高频

**答案**

两类：模块之间循环导入、Provider 之间循环注入。临时方案是两侧都使用 `forwardRef()`；根本方案是重构：抽取公共逻辑到第三个 Service、改用事件解耦、或者用 `ModuleRef` 延迟获取。另外，文件之间的循环 import（如 barrel 文件 `index.ts`）也会导致元数据为 undefined，报错信息类似「Nest can't resolve dependencies ... at index [0]」。

详见：[20-mid-qa Q11](20-mid-qa.md)

---

### R7. 动态模块是什么？`forRoot` 和 `forFeature` 有什么区别？怎么写一个 `forRootAsync`？

- **难度**: mid
- **标签**: 动态模块
- **来源倾向**: 中高级常问，封装过基础库的候选人会被追问细节

**答案**

动态模块是在导入时由静态方法返回模块定义（`DynamicModule`），用于传入配置。约定：`forRoot` 在根模块配置一次（如数据库连接）；`forFeature` 在功能模块中按需注册（如某些实体的 Repository）；`register` 每次导入独立配置。`forRootAsync` 接收 `imports`、`inject`、`useFactory`，把工厂注册为选项 Provider。推荐用 `ConfigurableModuleBuilder` 自动生成。

详见：[20-mid-qa Q9](20-mid-qa.md)、[40-coding-handwritten H5](40-coding-handwritten.md)

---

### R8. 自定义 Provider 有哪几种？什么时候用 `useFactory`？

- **难度**: junior
- **标签**: Provider
- **来源倾向**: 一面常问

**答案**

`useClass`（替换实现，如按环境切换）、`useValue`（常量、Mock）、`useFactory`（需要动态创建或异步初始化，可通过 `inject` 注入依赖，如根据配置创建数据库连接）、`useExisting`（别名）。Token 可以是类、字符串或 Symbol。

详见：[20-mid-qa Q6](20-mid-qa.md)

---

## 二、认证、校验与错误处理

### R9. 怎么实现「默认所有接口需要登录，个别接口公开」？角色怎么控制？

- **难度**: mid
- **标签**: 守卫, 元数据
- **来源倾向**: 高频，常要求现场写代码

**答案**

用 `APP_GUARD` 注册全局 JWT 守卫，`@Public()` 通过 `SetMetadata` 标记公开接口，守卫中用 `Reflector.getAllAndOverride` 同时读取方法和类上的元数据，命中则放行。角色用 `@Roles()` + `RolesGuard`，注册在认证守卫之后。未登录返回 401，无权限返回 403。

详见：[20-mid-qa Q4](20-mid-qa.md)、[40-coding-handwritten H2](40-coding-handwritten.md)

---

### R10. Access Token 和 Refresh Token 怎么设计？Token 被盗怎么办？

- **难度**: mid
- **标签**: JWT
- **来源倾向**: 高频

**答案**

Access Token 短期（如 15 分钟），无状态校验；Refresh Token 长期，存服务端（哈希后存储）可吊销，放 HttpOnly Cookie。刷新时轮换 Refresh Token，旧的立即失效；检测到已失效的 Refresh Token 被再次使用，说明可能被盗，吊销该用户的整个 Token 家族。退出登录、修改密码时吊销。

详见：[20-mid-qa Q13](20-mid-qa.md)

---

### R11. `ValidationPipe` 的 `whitelist`、`forbidNonWhitelisted`、`transform` 分别有什么作用？有什么坑？

- **难度**: junior
- **标签**: 校验
- **来源倾向**: 一面二面都常见

**答案**

`whitelist` 去掉 DTO 中未声明的字段（防止批量赋值漏洞）；`forbidNonWhitelisted` 遇到未声明字段直接报错；`transform` 把普通对象转为 DTO 实例并转换基本类型。坑：嵌套对象需要 `@ValidateNested()` + `@Type()`；query 参数都是字符串；数组参数单个值时不是数组；`transform` 有性能开销。Nest 12 可以选择 Standard Schema（Zod 等）方案。

详见：[10-junior-qa Q9](10-junior-qa.md)、[20-mid-qa Q15](20-mid-qa.md)、[20-mid-qa Q16](20-mid-qa.md)

---

### R12. 项目中怎么做统一的异常处理和响应格式？Service 里能抛 `HttpException` 吗？

- **难度**: mid
- **标签**: 错误处理
- **来源倾向**: 高频，高级岗会追问错误体系设计

**答案**

响应格式用拦截器统一包装；错误用全局异常过滤器统一格式，区分 HttpException、业务错误和未知错误，未知错误记录堆栈但只返回通用信息。Service 中直接抛 `HttpException` 会让业务层依赖 HTTP 协议，在消息消费者、定时任务中复用时语义不对；更好的做法是抛领域错误（带错误码），由过滤器映射为 HTTP 状态码。

详见：[10-junior-qa Q12](10-junior-qa.md)、[30-senior-qa Q6](30-senior-qa.md)、[40-coding-handwritten H4](40-coding-handwritten.md)

---

## 三、数据与性能

### R13. 一个业务操作涉及多个 Service，事务怎么做？

- **难度**: mid
- **标签**: 事务
- **来源倾向**: 中高级高频

**答案**

不要把事务对象层层传参。用 AsyncLocalStorage 保存当前事务，所有 Repository 从上下文中获取事务客户端；用 `@Transactional()` 装饰入口方法，嵌套调用自动加入同一事务。可以直接使用 `@nestjs-cls/transactional`。注意事务中不要调用外部服务或发消息，消息用 Outbox 模式在事务内写入。

详见：[20-mid-qa Q18](20-mid-qa.md)、[40-coding-handwritten H8](40-coding-handwritten.md)

---

### R14. 什么是 N+1 问题？在 Nest 中怎么解决？

- **难度**: mid
- **标签**: 性能, GraphQL
- **来源倾向**: 用过 GraphQL 或 ORM 的必问

**答案**

查询 N 条主数据后，对每条再查一次关联数据，总共 N+1 次查询。解决：ORM 中用 join / `include` 预加载；GraphQL 中用 DataLoader 把同一事件循环内的多次加载合并为一次批量查询，并且 DataLoader 必须是请求级实例（通过 CLS 或 context 创建），避免跨用户缓存。

详见：[20-mid-qa Q19](20-mid-qa.md)

---

### R15. 缓存穿透、击穿、雪崩在你的服务中怎么处理？

- **难度**: mid
- **标签**: 缓存
- **来源倾向**: 后端通用高频

**答案**

穿透：缓存空值、布隆过滤器、参数校验；击穿：热点 key 互斥重建（single-flight，进程内合并 + 分布式锁）或逻辑过期；雪崩：过期时间加随机值、多级缓存、限流降级。Nest 内置的 `CacheInterceptor` 只适合简单场景，复杂场景自己封装缓存服务。

详见：[20-mid-qa Q20](20-mid-qa.md)

---

### R16. 线上接口 P99 突然变高，你会怎么排查？

- **难度**: senior
- **标签**: 性能排查
- **来源倾向**: 高级岗场景题

**答案**

先确认范围（所有接口还是个别、所有实例还是部分、是否与发布时间吻合），再看链路追踪找到慢在哪一段：下游依赖、数据库（慢查询、连接池等待）、还是 Node 进程本身（事件循环延迟、GC、CPU）。进程问题用 CPU profile / 火焰图定位同步阻塞代码；常见原因有大 JSON 序列化、同步加解密、正则回溯、请求作用域 Provider 被引入。能回滚先回滚止血。

详见：[30-senior-qa Q13](30-senior-qa.md)、[30-senior-qa Q14](30-senior-qa.md)

---

### R17. Nest 应用出现内存泄漏，常见原因有哪些？

- **难度**: senior
- **标签**: 内存
- **来源倾向**: 高级岗

**答案**

单例中用 Map 缓存请求数据且无上限、事件监听未移除、定时器未清理、请求作用域对象被单例引用无法回收、DataLoader 被做成单例导致缓存无限增长、大量未完成的 Promise 堆积。排查：对比多份堆快照找增长对象及引用链，配合监控观察堆内存趋势。

详见：[30-senior-qa Q15](30-senior-qa.md)

---

## 四、分布式与架构

### R18. `@MessagePattern` 和 `@EventPattern` 有什么区别？传输层怎么选？

- **难度**: mid
- **标签**: 微服务
- **来源倾向**: 简历写了微服务必问

**答案**

`MessagePattern` 是请求-响应，调用方等待返回（`client.send`）；`EventPattern` 是发布事件，不需要响应（`client.emit`）。传输层：内部同步调用用 gRPC（强契约、性能好）；异步事件用 Kafka（高吞吐、可回放）或 RabbitMQ（灵活路由、ack 语义清晰）；TCP 传输只适合简单场景或开发。

详见：[20-mid-qa Q23](20-mid-qa.md)、[30-senior-qa Q10](30-senior-qa.md)

---

### R19. 下单后要发消息通知库存服务，怎么保证消息不丢、不重复？跨服务一致性怎么做？

- **难度**: senior
- **标签**: 一致性, MQ
- **来源倾向**: 高级后端必问

**答案**

不丢：Outbox 模式，在业务事务中写入 outbox 表，由转发器投递到 MQ，消费者处理成功后才 ack；不重复：至少一次投递无法避免重复，消费者按消息 ID 幂等（去重表或唯一约束）。跨服务一致性用 Saga（编排或协同）+ 补偿操作，资金类强一致场景考虑 TCC。

详见：[20-mid-qa Q24](20-mid-qa.md)、[30-senior-qa Q9](30-senior-qa.md)、[50-system-design S4](50-system-design.md)

---

### R20. 如何设计一个幂等的下单接口？

- **难度**: senior
- **标签**: 幂等
- **来源倾向**: 高级岗高频，常要求说出并发场景

**答案**

客户端生成 Idempotency-Key；服务端用 Redis `SET NX` 原子占位，记录请求体哈希；处理中再次请求返回 409，完成后再次请求返回首次结果，相同 key 不同请求体返回 422。Redis 只是第一层，数据库唯一约束和状态机条件更新是最终保障。在 Nest 中用拦截器 + 装饰器实现。

**口述模板（30–60 秒）**

1. **一句话结论**：客户端传幂等 key，服务端原子占位并保存结果，数据库约束兜底。
2. **关键机制**：`SET NX EX` 原子占位，状态分为处理中和已完成；完成后保存状态码和响应体，重复请求直接重放。
3. **注意点**：先保存结果再返回响应；失败时删除占位允许重试；key 加用户维度；校验请求体哈希。
4. **可追问时补充**：支付回调这类没有客户端 key 的场景，用渠道流水号做唯一约束，状态变更用带当前状态条件的 UPDATE。

详见：[30-senior-qa Q18](30-senior-qa.md)、[40-coding-handwritten H7](40-coding-handwritten.md)、[50-system-design S4](50-system-design.md)

---

### R21. 服务部署了多个实例，`@Cron` 定时任务重复执行怎么办？

- **难度**: mid
- **标签**: 定时任务, 分布式锁
- **来源倾向**: 高频实战题

**答案**

方案：Redis 分布式锁（`SET NX PX` + token，Lua 脚本释放）；或改用 BullMQ 的重复任务，由队列保证只执行一次；或把定时任务拆到单独的单实例部署。无论哪种方案，任务本身都应该幂等，以应对锁过期等异常。

详见：[30-senior-qa Q16](30-senior-qa.md)、[40-coding-handwritten H9](40-coding-handwritten.md)

---

### R22. 多租户系统在 Nest 中怎么实现数据隔离？

- **难度**: senior
- **标签**: 多租户
- **来源倾向**: SaaS 方向高级岗

**答案**

隔离模型：共享表 + tenant_id、独立 schema、独立库，通常混合使用。租户 ID 来自已验证的 JWT；认证后写入 CLS；仓储基类自动加租户条件；PostgreSQL RLS 兜底；异步任务从消息中恢复租户上下文。避免用请求作用域传递租户，独立库场景用 durable provider 或连接缓存。

详见：[30-senior-qa Q8](30-senior-qa.md)、[40-coding-handwritten H10](40-coding-handwritten.md)、[50-system-design S2](50-system-design.md)

---

### R23. 应用关闭时怎么做优雅停机？生命周期钩子的顺序？

- **难度**: mid
- **标签**: 生命周期, 部署
- **来源倾向**: 常见，K8s 部署经验会被追问

**答案**

启动：`onModuleInit` → `onApplicationBootstrap`；关闭：`onModuleDestroy` → `beforeApplicationShutdown` → `onApplicationShutdown`。关闭钩子需要 `app.enableShutdownHooks()` 才会响应 SIGTERM。K8s 中收到 SIGTERM 后：先让就绪探针失败，等待负载均衡摘除流量，再停止接收新请求、等待进行中的请求和任务完成，最后关闭数据库、Redis、MQ 连接。

详见：[10-junior-qa Q17](10-junior-qa.md)、[20-mid-qa Q28](20-mid-qa.md)

---

### R24. 你会怎么组织一个大型 Nest 项目的代码结构？怎么落地分层或 DDD？

- **难度**: senior
- **标签**: 架构
- **来源倾向**: 高级岗开放题

**答案**

按业务域划分模块，模块内分层：Controller（协议适配）→ Application Service（用例编排、事务边界）→ Domain（实体、领域规则，不依赖 Nest）→ Infrastructure（Repository 实现、外部客户端）。领域层定义抽象类作为 Token，基础设施层提供实现。模块之间只通过导出的接口通信，用 lint 规则限制依赖方向。不要为了 DDD 而 DDD，简单 CRUD 模块保持简单。

详见：[30-senior-qa Q7](30-senior-qa.md)、[10-junior-qa Q30](10-junior-qa.md)、[50-system-design S8](50-system-design.md)

---

### R25. Nest 11、Nest 12 有哪些重要变化？升级要注意什么？

- **难度**: mid
- **标签**: 版本
- **来源倾向**: 近年新增，考察是否持续关注生态

**答案**

Nest 11：要求 Node 20+，默认 Express 5 / Fastify 5（通配符必须命名，如 `*splat`；包含根路径用 `{*splat}`）、`ConsoleLogger` 支持 JSON 输出、cache-manager v6（TTL 单位为毫秒）、`ParseDatePipe`。Nest 12：核心包改为 ESM 发布（CJS 项目借助 `require(esm)` 仍可使用）、要求 Node 20.19+ / 22.12+、内置 Standard Schema 校验（`@Body({ schema })`，可直接使用 Zod、Valibot）、官方可观测性 SDK `@nestjs/observe`、CLI 废弃 webpack 改用 Rspack、新 ESM 项目默认 Vitest。升级先读官方迁移指南，逐个大版本升级，`@nestjs/*` 包同步升级，跑完整测试。细节以官方迁移指南为准。

详见：[20-mid-qa Q30](20-mid-qa.md)、[20-mid-qa Q16](20-mid-qa.md)、[30-senior-qa Q21](30-senior-qa.md)
