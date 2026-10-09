# NestJS 系统设计题（8 道）

> 回答结构：澄清需求与量级 → 架构图 → 模块划分 → 核心流程 → 数据与一致性 → 扩展性与可靠性 → 可观测性 → 演进。
>
> NestJS 面试的系统设计重点不在「画大图」，而在于**能把设计落到 Nest 的具体机制上**：哪些逻辑放守卫/拦截器/管道，模块怎么切，哪些用队列，上下文怎么传。每道题都附有「Nest 落地」部分。

---

### S1. 设计一个基于 Nest 的 BFF 聚合层

- **难度**: senior
- **标签**: BFF, 聚合, 容错

**澄清问题（面试时先问）**

- 服务哪些端（Web、App、小程序）？每个端的数据需求差异大吗？
- 下游有多少个服务？协议是 HTTP、gRPC 还是混合？
- 峰值 QPS、P99 延迟目标？
- 是否需要 GraphQL，还是 REST 聚合接口即可？

**参考方案**

```text
Client (Web / App)
  ↓
API 网关：TLS、WAF、粗粒度限流
  ↓
Nest BFF（无状态，多实例）
  ├─ 全局：requestId 中间件 → JwtAuthGuard → 日志/超时拦截器 → 统一异常过滤器
  ├─ 聚合模块（按页面/场景划分）：HomeModule、OrderDetailModule …
  │    └─ 并行调用下游 + 部分失败降级
  ├─ 下游客户端模块（每个服务一个）：UserClient、ProductClient …
  │    └─ 超时、熔断、重试（仅幂等请求）、连接池
  └─ 缓存：进程内 LRU（配置类）+ Redis（共享数据）
  ↓
下游微服务（HTTP / gRPC）
```

**关键设计**

- **按场景组织模块**：BFF 的接口为页面服务，`HomeModule` 聚合首页所需数据；下游客户端是共享模块
- **部分失败降级**：核心数据失败返回错误，非核心（推荐、广告）失败返回空并标记 `degraded`
- **超时预算**：总超时 800ms，单个下游 300ms；用 `AbortSignal.timeout()` 传给 fetch
- **熔断**：下游客户端封装 opossum 等熔断器，避免故障下游拖垮 BFF
- **类型契约**：下游 OpenAPI / proto 生成客户端类型，避免手写
- **REST vs GraphQL**：端多、字段需求差异大时用 GraphQL（配合 DataLoader）；场景固定时 REST 聚合接口更简单、更容易缓存

**Nest 落地**

- 下游客户端通过动态模块 `HttpClientModule.forFeature({ name, baseUrl, timeout })` 统一创建
- 用 `Promise.allSettled` 并行调用，按结果组装响应
- 超时拦截器作为兜底，具体超时在客户端层控制
- 上下文（requestId、用户、灰度标记）通过 `nestjs-cls` 透传到下游请求头

**口述模板（2–3 分钟）**

BFF 的定位是为前端聚合和适配数据，不写领域逻辑。整体是 Nest 无状态多实例部署，前面有网关做 TLS 和粗粒度限流。BFF 内部按页面场景划分模块，每个下游服务封装成独立的客户端模块，统一处理超时、熔断和只对幂等请求重试。聚合接口用 `Promise.allSettled` 并行调用，核心数据失败直接报错，非核心降级为空。横切关注点放在 Nest 管线中：中间件生成 requestId，全局守卫做鉴权，拦截器负责日志和超时兜底，过滤器统一错误格式。上下文通过 CLS 透传到下游请求头，配合 OpenTelemetry 实现全链路追踪。如果多端字段需求差异大，会考虑 GraphQL 并配合 DataLoader 解决 N+1。

**扩展追问**

- BFF 成为瓶颈怎么办？→ 无状态水平扩展；按端或业务域拆分多个 BFF；热点数据缓存。
- BFF 由前端团队还是后端团队维护？→ 通常前端团队维护（贴近页面需求），后端提供稳定的领域接口。
- 如何避免 BFF 变成「第二个单体」？→ 只做聚合和适配，业务规则下沉到领域服务；定期审查。

---

### S2. 设计多租户 SaaS 后端

- **难度**: senior
- **标签**: 多租户, 数据隔离

**澄清问题（面试时先问）**

- 租户数量和规模分布？（几千个小租户 + 少数大客户？）
- 隔离要求：是否有客户要求独立数据库或独立部署（合规）？
- 是否允许租户自定义字段、配置、域名？
- 计费方式：按席位、按用量？

**参考方案**

```text
请求 → 租户识别（子域名 / JWT claim）
     → JwtAuthGuard（验证用户属于该租户）
     → TenantGuard：写入 CLS { tenantId, plan }
     → 业务模块
          └─ Repository：自动加 tenant_id 条件
               └─ 数据库：PostgreSQL RLS 兜底

数据隔离分层：
  小租户：共享库共享表 + tenant_id + RLS
  大客户：独立 schema 或独立库（连接按租户路由）
```

**关键设计**

- **租户识别**：可信来源是已验证的 JWT 中的 tenantId；子域名只用于登录前定位租户，登录后必须与身份一致
- **三层防线**：仓储基类自动加条件 → 数据库 RLS（`SET app.tenant_id`）→ 测试中的跨租户用例
- **混合隔离**：默认共享表，大客户可迁移到独立库；通过「租户 → 数据源」路由表决定连接
- **限额与公平性**：按租户限流、队列按租户分组，防止大租户影响其他租户（noisy neighbor）
- **租户配置**：功能开关、套餐限额缓存到 Redis，变更时主动失效

**Nest 落地**

- 默认用单例 Provider + `nestjs-cls` 传递租户，不使用请求作用域，避免性能问题
- 独立数据源的租户：用 durable provider + `ContextIdStrategy` 按租户复用连接，或自己维护「tenantId → DataSource」的连接缓存（需要上限和 LRU 淘汰）
- 队列任务和定时任务中，从消息中恢复租户并调用 `cls.run()`

**口述模板（2–3 分钟）**

多租户最核心的是隔离和成本的权衡。我会采用混合模型：大多数小租户共享数据库和表，通过 tenant_id 区分；对合规要求高的大客户使用独立库。租户 ID 只信任已验证的 JWT，子域名仅用于登录前定位。在 Nest 中，认证之后由 TenantGuard 把租户写入 CLS，所有仓储继承一个自动加租户条件的基类，并在 PostgreSQL 开启 RLS 作为兜底，即使有人写原生 SQL 漏了条件，也查不到其他租户数据。独立库的租户通过连接路由访问，连接按租户缓存并有上限。另外要防止吵闹邻居：限流、队列和缓存都按租户维度隔离。异步任务没有 HTTP 上下文，需要从消息中恢复租户。

**扩展追问**

- 如何把一个租户从共享库迁移到独立库？→ 双写或导出导入 + 短暂只读窗口，切换路由表，校验后清理旧数据。
- 跨租户的统计报表怎么做？→ 走单独的数据仓库（CDC 同步），不在业务库上跨租户查询。
- 租户自定义字段？→ JSONB 扩展字段 + 元数据定义 + 校验；查询需求高时建表达式索引。

---

### S3. 设计 RBAC + ABAC 权限系统

- **难度**: senior
- **标签**: 权限, 安全

**澄清问题（面试时先问）**

- 权限粒度：菜单/接口级，还是数据行级、字段级？
- 是否需要「只能编辑自己创建的文档」「只能看本部门数据」这类条件？
- 角色是否可以由租户自定义？
- 权限变更多快生效？

**参考方案**

```text
模型：
  User ─┬─ UserRole ─ Role ─ RolePermission ─ Permission(action, subject, conditions?)
        └─ 属性（部门、租户、职级）

请求流程：
  JwtAuthGuard → PermissionGuard
                   ├─ 读取 @RequirePermission('update', 'Article')
                   ├─ 加载用户权限（Redis 缓存，版本号失效）
                   ├─ 构建 Ability（CASL）
                   └─ 粗粒度判断：能否对该类资源执行该操作
  Service 层 → 加载资源后细粒度判断：ability.can('update', article)
  Repository → 数据范围：把 ability 转换为查询条件（accessibleBy）
```

**关键设计**

- **两层判断**：守卫做接口级（不加载资源），Service 加载资源后做实例级（条件判断）
- **数据范围过滤**：列表查询把权限条件转为 SQL 条件，而不是查出来再过滤（分页会错）
- **缓存与失效**：用户权限缓存在 Redis，角色变更时递增版本号；JWT 中不放完整权限，避免变更不生效
- **默认拒绝**：未声明权限的接口默认拒绝，或启动时扫描检查所有路由都有声明
- **审计**：敏感操作记录操作人、资源、前后值

**Nest 落地**

- `@RequirePermission()` 用 `Reflector.createDecorator` 实现
- `PermissionGuard` 通过 `APP_GUARD` 全局注册，排在认证守卫之后
- 构建 Ability 的工厂作为单例 Provider，用户维度结果按请求缓存在 CLS
- 启动时用 `DiscoveryService` 扫描所有路由，未声明权限的路由打印警告或启动失败

**口述模板（2–3 分钟）**

我会用 RBAC 作为基础，再叠加 ABAC 条件处理「只能编辑自己的文章」这类需求，实现上可以用 CASL。权限判断分两层：全局 PermissionGuard 读取接口上声明的 action 和 subject，做不依赖具体资源的粗粒度判断；Service 加载资源后再用 ability 判断具体实例。列表接口的数据范围不能查出来再过滤，否则分页会错，需要把权限条件转为查询条件。用户权限缓存在 Redis，通过版本号失效，JWT 只放用户 ID 和角色版本，保证权限变更能及时生效。安全上采用默认拒绝，启动时用 DiscoveryService 扫描所有路由，未声明权限的直接报警。敏感操作写审计日志。

**扩展追问**

- 字段级权限？→ 序列化阶段按 ability 过滤字段（CASL `permittedFieldsOf`），写入时同样校验。
- 权限检查的性能？→ 权限预加载 + 缓存；Ability 按请求构建一次复用。
- 多个服务共享权限？→ 独立的权限服务或策略引擎（如 OPA），各服务通过 SDK 本地评估缓存的策略。

---

### S4. 设计订单服务（下单、支付回调、超时取消）

- **难度**: senior
- **标签**: 一致性, 幂等, 状态机

**澄清问题（面试时先问）**

- 下单峰值 QPS？是否有秒杀场景？
- 库存在本服务还是独立的库存服务？
- 支付渠道有哪些？回调是否会重复、乱序？
- 未支付订单多久取消？

**参考方案**

```text
下单：
  POST /orders (Idempotency-Key)
    → 幂等拦截器
    → OrdersService.place()  [事务]
         ├─ 条件更新扣减库存（stock >= qty）或预占库存
         ├─ 创建订单 status=PENDING_PAYMENT
         └─ 写 outbox: OrderCreated
    → Outbox 转发器 → MQ
    → BullMQ 延迟任务：30 分钟后检查支付状态

支付回调：
  POST /payments/callback
    → 验签 → 按支付流水号幂等
    → 条件更新：UPDATE orders SET status='PAID' WHERE id=? AND status='PENDING_PAYMENT'
    → 写 outbox: OrderPaid

超时取消：
  延迟任务触发 → 条件更新为 CANCELLED（仅当仍为 PENDING_PAYMENT）→ 释放库存
  兜底：定时任务扫描超时订单（防止延迟任务丢失）
```

**关键设计**

- **状态机**：所有状态变更都是条件更新（带当前状态），天然处理回调和超时取消的并发冲突
- **幂等**：下单用 Idempotency-Key；回调用支付流水号唯一约束
- **库存**：普通场景数据库条件更新；秒杀场景 Redis 预扣（Lua 原子）+ 异步落库 + 对账
- **可靠事件**：Outbox 保证「订单写库」和「发事件」一致
- **延迟任务 + 定时扫描**：双保险；取消前查询支付渠道，防止「已付款但回调丢失」被误取消
- **对账**：每日与支付渠道对账，处理长尾异常

**Nest 落地**

- 订单模块内部分层：Controller → Application Service → Domain（状态机）→ Repository
- 事务用 `@nestjs-cls/transactional`，Outbox 写入在同一事务中
- 延迟取消用 `@nestjs/bullmq` 的 `delay` 任务，`jobId` 设为订单 ID 保证不重复
- 支付回调验签放在守卫中，回调接口加 `@Public()` 跳过 JWT

**口述模板（2–3 分钟）**

订单服务的核心是状态机和幂等。下单接口要求客户端传 Idempotency-Key，通过拦截器防重复；Service 在一个事务中条件扣减库存、创建待支付订单并写 outbox 记录，由转发器可靠地把事件发到 MQ。同时投递一个 30 分钟的 BullMQ 延迟任务用于超时取消，jobId 用订单号保证唯一。支付回调先验签，再按支付流水号去重，然后用带当前状态条件的 UPDATE 把订单改为已支付；超时取消同样是条件更新，所以两者并发时只有一个会成功。取消前还会主动查询支付渠道，避免回调丢失导致误取消。延迟任务之外还有定时扫描兜底，每日对账处理长尾问题。秒杀场景库存改为 Redis Lua 预扣，异步落库。

**扩展追问**

- 支付成功但订单已取消？→ 触发自动退款或人工处理流程，并告警。
- 库存是独立服务时怎么保证一致？→ 预占库存（TCC 的 Try）+ 超时释放，或 Saga 补偿。
- 订单表越来越大？→ 按用户 ID 分库分表；历史订单归档；查询走 ES。

---

### S5. 设计实时通知 / 消息推送服务

- **难度**: senior
- **标签**: WebSocket, 实时

**澄清问题（面试时先问）**

- 同时在线连接数？（10 万、100 万？）
- 消息类型：系统通知、聊天、协同编辑？
- 是否要求离线消息、已读回执、多端同步？
- 推送是否需要严格有序、不丢失？

**参考方案**

```text
Client ──WebSocket──→ LB（按连接粘性）→ Nest Gateway 实例 × N
                                         ├─ 握手鉴权（token）
                                         ├─ 加入房间：user:{id}、tenant:{id}
                                         └─ Redis Adapter（跨实例广播）

业务服务 → MQ（notification 主题）→ 推送服务消费者
                                    ├─ 写消息表（持久化，用于离线拉取）
                                    └─ server.to(`user:${id}`).emit(...)

客户端重连 → 带上 lastMessageId → 拉取期间缺失的消息
```

**关键设计**

- **推拉结合**：推送只负责实时性；可靠性由「消息持久化 + 客户端按游标补拉」保证
- **跨实例**：Socket.IO Redis Adapter（或 Redis Streams 版本）；连接数很大时按用户哈希路由到固定实例
- **鉴权**：握手时验证 token，token 过期后要求重连，不要在每条消息上重复验证
- **背压**：慢客户端的发送缓冲区设上限，超限断开
- **单向推送场景用 SSE**：通知类场景 SSE 更简单，基于 HTTP，自动重连
- **连接扩容**：调整文件描述符上限、心跳间隔；发布时优雅断开，客户端随机退避重连，避免重连风暴

**Nest 落地**

- `@WebSocketGateway()` + `IoAdapter` 子类集成 Redis Adapter
- 握手鉴权放在 `afterInit` 的 `server.use()` 中间件，或 `handleConnection` 中断开非法连接
- 业务服务不直接调用网关，而是发布事件（MQ 或 `@nestjs/event-emitter`），推送模块订阅后推送，降低耦合
- 消息处理器上使用 `WsExceptionFilter` 和 `ValidationPipe` 校验客户端消息

**口述模板（2–3 分钟）**

我会把实时推送拆成两件事：实时性靠 WebSocket 推送，可靠性靠消息持久化和客户端补拉。网关层是 Nest WebSocket Gateway 多实例部署，握手时验证 token，连接按用户加入房间，跨实例广播用 Socket.IO 的 Redis Adapter。业务服务不直接依赖网关，而是把通知发到 MQ，推送服务消费后先写消息表，再推给在线用户。客户端断线重连时带上最后收到的消息 ID，拉取缺失的消息，所以即使推送丢失也能补齐。连接数很大时要处理背压、心跳和发布时的重连风暴，客户端用随机退避重连。如果只是单向通知，我会优先考虑 SSE，实现和运维都更简单。

**扩展追问**

- 百万连接怎么支撑？→ 单机调优（fd、内存），按用户哈希分片到连接层，连接层与业务逻辑分离。
- 消息顺序？→ 同一会话按单调递增序号排序，客户端按序号去重和排序。
- 已读回执？→ 客户端上报已读位置（游标），而不是每条消息单独回执。

---

### S6. 设计异步任务平台（报表导出、批量处理）

- **难度**: mid
- **标签**: 队列, BullMQ

**澄清问题（面试时先问）**

- 任务类型和耗时：秒级还是小时级？
- 任务量峰值？是否有优先级？
- 是否需要进度、取消、重试？结果怎么交付（下载链接、邮件）？
- 多租户是否需要公平调度？

**参考方案**

```text
POST /exports → 校验 + 去重（相同参数复用进行中的任务）
             → 写任务表 status=QUEUED
             → BullMQ.add(jobId=taskId, { attempts, backoff })
             → 返回 202 + taskId

Worker 进程（独立部署，可单独扩容）
  → 流式读取数据库（游标分批）→ 流式写入文件 → 分片上传 OSS
  → job.updateProgress() → 更新任务表
  → 完成：生成签名下载链接 + 通知（站内信 / 邮件）

GET /exports/:id → 查询状态和进度（或 SSE 推送进度）
```

**关键设计**

- **API 与 Worker 分离部署**：同一代码库，不同入口；Worker 按队列积压量自动扩容
- **流式处理**：大数据量导出不能一次性加载到内存，使用游标 + Stream 管道
- **重试与幂等**：指数退避重试；任务以 taskId 幂等，重试时覆盖而不是追加
- **取消**：任务表标记取消，Worker 在分批循环中检查
- **公平性**：按租户限制并发（BullMQ Pro 的 group 或按租户拆队列）
- **失败处理**：超过重试次数进入失败列表，告警并支持手动重试；Bull Board 可视化

**Nest 落地**

- `@nestjs/bullmq`：`BullModule.registerQueue()`，`@Processor('export')` + `WorkerHost`
- Worker 入口用 `NestFactory.createApplicationContext()`，不启动 HTTP 服务
- 处理器中恢复 CLS 上下文（租户、traceId 从 job.data 中读取）
- 优雅关闭：`enableShutdownHooks()`，Worker 关闭时等待当前任务完成

**口述模板（2–3 分钟）**

长任务一律异步化：API 只负责校验、去重、写任务表并投递 BullMQ，立即返回 202 和任务 ID，前端轮询或通过 SSE 获取进度。Worker 和 API 是同一代码库的两个入口，Worker 用 `createApplicationContext` 启动，不监听 HTTP，可以按队列积压独立扩容。导出时用数据库游标分批读取，流式写文件并分片上传对象存储，内存占用恒定。任务支持指数退避重试，以任务 ID 保证幂等；支持取消和进度上报。多租户场景下按租户限制并发，避免大租户占满 Worker。失败任务进入失败列表并告警，通过 Bull Board 可以查看和手动重试。发布时 Worker 优雅关闭，等当前任务完成再退出。

**扩展追问**

- Redis 挂了任务会丢吗？→ 任务表是事实来源，可以从任务表恢复未完成的任务重新入队。
- 任务执行几小时，发布时怎么办？→ 拆成可断点续跑的小步骤，记录检查点。
- 定时任务多实例重复执行？→ BullMQ 的 repeatable job（Job Scheduler）天然只执行一次。

---

### S7. 设计开放平台 API（签名、限流、配额、Webhook）

- **难度**: senior
- **标签**: 开放平台, 安全, 限流

**澄清问题（面试时先问）**

- 调用方是第三方企业开发者？认证方式：API Key、HMAC 签名，还是 OAuth 2.0？
- 是否需要按套餐限流和计费？
- 需要向第三方推送事件（Webhook）吗？
- API 版本如何演进？

**参考方案**

```text
第三方 → 网关 → Nest Open API 服务
                 ├─ SignatureGuard：AppKey + 时间戳 + nonce + HMAC(body) 验签，防重放
                 ├─ QuotaGuard：按 AppKey 的令牌桶限流（Redis Lua）+ 日配额
                 ├─ 版本：URI 版本 /v1、/v2（enableVersioning）
                 ├─ 业务模块（复用内部领域服务）
                 └─ 用量拦截器：异步记录调用量 → 计费

Webhook：
  领域事件 → MQ → WebhookDispatcher
               ├─ HMAC 签名请求体
               ├─ 失败指数退避重试（最多 N 次，持续 24 小时）
               └─ 投递日志（可在控制台查看和重放）
```

**关键设计**

- **签名防重放**：时间戳窗口（如 5 分钟）+ nonce 去重（Redis SETNX，过期时间等于窗口）
- **限流**：按 AppKey + 接口维度令牌桶；响应头返回 `X-RateLimit-Remaining`，超限 429 + `Retry-After`
- **版本管理**：破坏性变更才升级大版本；旧版本标记 Deprecation 头，给出下线时间
- **Webhook 可靠性**：至少一次投递，事件带唯一 ID，由接收方去重；签名让接收方验证来源
- **SSRF 防护**：Webhook 地址校验，禁止内网地址
- **文档**：OpenAPI 自动生成 + SDK 生成

**Nest 落地**

- 签名守卫需要原始请求体：`NestFactory.create(AppModule, { rawBody: true })`，读取 `req.rawBody`
- 限流可基于 `@nestjs/throttler` 自定义存储（Redis）和 key 生成（AppKey）
- `@nestjs/swagger` 生成文档，只暴露开放接口的标签
- Webhook 投递用 BullMQ，`attempts` + 自定义退避策略

**口述模板（2–3 分钟）**

开放平台的关键是安全、限流和可靠的事件通知。认证使用 AppKey 加 HMAC 签名，签名内容包括时间戳、nonce 和请求体，在 Nest 中用守卫实现，需要开启 rawBody 拿到原始请求体；时间戳窗口加 nonce 去重防止重放。限流按 AppKey 和接口维度在 Redis 中做令牌桶，超限返回 429 和 Retry-After，同时按套餐有日配额。用量由拦截器异步记录用于计费。API 用 URI 版本管理，破坏性变更才升级大版本。向第三方推送事件用 Webhook：事件进 MQ，由投递服务签名后发送，失败指数退避重试，提供投递日志和手动重放；事件带唯一 ID 让接收方去重，同时校验回调地址防止 SSRF。

**扩展追问**

- AppSecret 泄露怎么办？→ 支持多密钥轮换（新旧同时有效一段时间），控制台一键吊销。
- 签名验证时 JSON 序列化不一致？→ 必须基于原始字节计算，而不是解析后再序列化。
- 第三方 Webhook 接口一直失败？→ 熔断该订阅，通知开发者，暂停投递并保留事件以便恢复后重放。

---

### S8. 把 Nest 单体演进为模块化单体 / 微服务

- **难度**: senior
- **标签**: 架构演进, 微服务

**澄清问题（面试时先问）**

- 当前痛点是什么：发布冲突、性能瓶颈、团队协作，还是故障影响面？
- 团队规模和团队边界？
- 有没有完善的 CI/CD、监控、链路追踪？
- 数据库是否共享，表之间耦合程度如何？

**参考方案**

```text
阶段 1：模块化单体
  按业务域划分 Nest 模块（Order、Inventory、User …）
  ├─ 模块只通过公开的 Service 接口（exports）互相调用
  ├─ 禁止跨模块直接访问其他模块的表 / Repository
  ├─ 模块间异步交互通过领域事件（EventEmitter）
  └─ 依赖规则用 lint 工具强制（Nx module boundaries / dependency-cruiser）

阶段 2：按需拆分（Strangler 绞杀者模式）
  选择边界清晰、有独立扩展需求的模块（如通知、搜索）
  ├─ 进程内事件 → MQ 事件
  ├─ 进程内调用 → gRPC / HTTP（接口不变，替换实现）
  ├─ 数据：先逻辑隔离（独立 schema），再物理拆库
  └─ 网关按路由逐步切流量，可回滚
```

**关键设计**

- **先模块化再拆分**：边界不清时直接拆微服务，得到的是「分布式单体」
- **接口抽象**：模块之间依赖抽象 Token（如 `InventoryPort`），拆分时只替换 Provider 实现为远程客户端
- **数据拆分是最难的部分**：先消除跨模块 join，改为通过接口或数据冗余；用 CDC 同步需要的数据
- **一致性**：跨服务事务改为 Saga + Outbox
- **基础设施先行**：链路追踪、集中日志、契约测试、独立 CI/CD
- **不是所有模块都要拆**：拆分有成本，按收益决定

**Nest 落地**

- Nest Monorepo（或 Nx）：`apps/` 放各个服务，`libs/` 放共享库（DTO 契约、基础设施）
- `ClientsModule` + `@MessagePattern` / `@EventPattern` 实现服务间通信，传输层可从 TCP 换为 gRPC、Kafka
- 由于模块依赖的是抽象 Token，拆分时只需把 `{ provide: InventoryPort, useClass: LocalInventory }` 改为 `RemoteInventoryClient`
- 混合应用：`app.connectMicroservice()` 让同一进程同时提供 HTTP 和消息消费，便于过渡

**口述模板（2–3 分钟）**

我不会一开始就拆微服务，而是先把单体整理为模块化单体：按业务域划分 Nest 模块，模块之间只能通过导出的接口调用，不能直接访问其他模块的表，并用 lint 规则强制依赖边界。模块之间依赖抽象 Token 而不是具体实现，异步交互用领域事件。边界稳定后，再按收益选择拆分对象，比如有独立扩展需求或发布频率差异大的模块。拆分采用绞杀者模式：进程内事件改为 MQ，进程内调用改为 gRPC，由于依赖的是抽象 Token，只需替换 Provider 实现。数据拆分最难，先消除跨模块 join，再从独立 schema 过渡到独立库，跨服务一致性改用 Saga 加 Outbox。拆分前要先具备链路追踪、集中日志和契约测试，否则排查问题的成本会远超收益。

**扩展追问**

- 如何判断模块边界划得对不对？→ 看变更是否经常同时修改多个模块、是否存在大量跨模块调用和共享表。
- 拆分后接口契约怎么管理？→ 共享 proto / OpenAPI 定义，契约测试（Pact）在 CI 中验证。
- 拆分后性能变差？→ 粗粒度接口、批量接口、缓存；检查是否拆分粒度过细。
