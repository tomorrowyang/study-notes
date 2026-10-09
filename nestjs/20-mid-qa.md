# NestJS 中级 Q&A（机制与工程实践）

> 目标：讲清完整的请求生命周期、执行上下文与元数据、自定义 Provider 与作用域、动态模块、循环依赖、鉴权授权、校验与序列化、事务、缓存、消息通信和测试。中级题的关键是「为什么这样设计」和「踩过什么坑」。

---

## 一、请求生命周期

### Q1. 一个 HTTP 请求在 Nest 中的完整执行顺序是什么？

- **难度**: mid
- **标签**: 生命周期, 高频

**答案**

```text
请求进入
 │
 ├─ 1. 中间件：全局（app.use）→ 模块绑定（按 configure 中的绑定顺序）
 ├─ 2. 守卫：全局 → 控制器 → 路由
 ├─ 3. 拦截器（进入阶段，next.handle() 之前）：全局 → 控制器 → 路由
 ├─ 4. 管道：全局 → 控制器 → 路由 → 参数级
 ├─ 5. 控制器方法 → Service ...
 ├─ 6. 拦截器（返回阶段）：路由 → 控制器 → 全局（与进入阶段相反）
 └─ 7. 异常过滤器（只在出现异常时）：路由 → 控制器 → 全局，只执行第一个匹配的
 │
响应
```

细节：

- 同一级别的多个组件按**绑定顺序**执行（如 `@UseGuards(A, B)` 先 A 后 B）
- 拦截器是嵌套的（洋葱模型），所以返回阶段顺序相反
- 管道在参数级别的执行顺序：官方文档说明，方法有多个带管道的参数时，**从最后一个参数到第一个参数**依次执行（单个参数上的多个管道仍按书写顺序）
- 任一环节抛出异常，后续环节都不执行，直接进入异常过滤器

记忆方法：**中间件做通用预处理 → 守卫决定能不能进 → 拦截器包住整个执行 → 管道准备参数 → 处理器干活**。

**口述模板（30–60 秒）**

1. 一句话结论：中间件 → 守卫 → 拦截器前半段 → 管道 → 处理器 → 拦截器后半段，出错走过滤器。
2. 关键机制：每类组件按全局、控制器、路由的顺序执行；拦截器返回阶段倒序。
3. 注意点：守卫在管道之前，拿到的参数还没有被转换。
4. 可追问时补充：过滤器从路由级开始找，只执行第一个匹配的。

**常见追问**

- 守卫为什么放在拦截器之前？→ 鉴权失败的请求不应该进入任何业务包装逻辑（如缓存、计时），越早拒绝开销越小。

---

### Q2. 不同阶段抛出的异常分别会怎样被处理？

- **难度**: mid
- **标签**: 异常, 生命周期

**答案**

| 抛出位置 | 处理方式 |
|---------|---------|
| 中间件 | 被 Nest 的异常层捕获，交给全局过滤器（中间件不在路由上下文中，路由级和控制器级过滤器不生效） |
| 守卫 | 进入过滤器链；拦截器尚未执行，拦截器的 `catchError` 看不到 |
| 管道 | 进入过滤器链；**拦截器的 `catchError` 能看到**（管道在拦截器内部执行） |
| 控制器 / Service | 先经过拦截器的 `catchError`（可以转换异常），再进入过滤器 |
| 拦截器自身 | 进入过滤器链 |
| 过滤器内部 | 由 Nest 内置的兜底处理返回 500 |

```ts
// 拦截器中把数据库唯一约束错误映射为 409
return next.handle().pipe(
  catchError(err => {
    if (err?.code === '23505') throw new ConflictException('duplicate')
    throw err
  }),
)
```

额外注意：

- **在回调或未被 await 的 Promise 中抛出的异常**不会进入 Nest 的异常层，会变成 `unhandledRejection`
- 使用 `@Res()` 手动响应后再抛异常，可能出现「响应头已发送」错误

**口述模板（30–60 秒）**

1. 一句话结论：除了中间件只走全局过滤器，其他阶段的异常都进入过滤器链。
2. 关键机制：管道和处理器的异常能被拦截器的 catchError 捕获，守卫的异常不能。
3. 注意点：没有 await 的异步异常不会被 Nest 捕获。
4. 可追问时补充：拦截器适合把底层错误转换为业务异常，过滤器负责统一输出格式。

**常见追问**

- 为什么推荐在拦截器里转换异常，而在过滤器里统一格式？→ 职责分离：转换是业务语义，格式化是协议层表现；过滤器只执行一个，难以叠加多种转换逻辑。

---

### Q3. `ArgumentsHost` 和 `ExecutionContext` 有什么区别？

- **难度**: mid
- **标签**: 执行上下文

**答案**

- `ArgumentsHost`：对处理器参数的抽象，能按上下文类型取出对象：
  - HTTP：`switchToHttp()` → `getRequest()` / `getResponse()`
  - 微服务：`switchToRpc()` → `getData()` / `getContext()`
  - WebSocket：`switchToWs()` → `getClient()` / `getData()`
  - `getType()` 返回 `'http' | 'rpc' | 'ws'`（GraphQL 返回 `'graphql'`）
- `ExecutionContext` **继承** `ArgumentsHost`，额外提供：
  - `getHandler()`：即将执行的处理器方法
  - `getClass()`：所属的控制器类

用途：守卫和拦截器拿到 `ExecutionContext`，可以读取处理器和类上的元数据；过滤器只拿到 `ArgumentsHost`。

同一个守卫要同时支持 HTTP、微服务、GraphQL 时：

```ts
canActivate(ctx: ExecutionContext) {
  const type = ctx.getType<'http' | 'rpc' | 'graphql'>()
  const req = type === 'graphql'
    ? GqlExecutionContext.create(ctx).getContext().req
    : ctx.switchToHttp().getRequest()
}
```

**口述模板（30–60 秒）**

1. 一句话结论：ArgumentsHost 封装参数，ExecutionContext 在此基础上能拿到处理器和类。
2. 关键机制：getHandler 和 getClass 用来读取装饰器元数据。
3. 注意点：同一组件跨协议使用时要按 getType 分支处理。
4. 可追问时补充：GraphQL 需要用 GqlExecutionContext 转换。

**常见追问**

- 过滤器为什么拿不到 `getHandler`？→ 过滤器可能处理任何地方抛出的异常（包括中间件），此时不一定有确定的处理器。

---

### Q4. 如何用元数据实现 `@Public()` 和 `@Roles()`？`Reflector` 的几个方法有什么区别？

- **难度**: mid
- **标签**: 元数据, 鉴权, 高频

**答案**

```ts
export const IS_PUBLIC = 'isPublic'
export const Public = () => SetMetadata(IS_PUBLIC, true)

// Nest 10 起推荐：类型安全的元数据装饰器
export const Roles = Reflector.createDecorator<Role[]>()

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly reflector: Reflector, private readonly jwt: JwtService) {}

  async canActivate(ctx: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC, [
      ctx.getHandler(),
      ctx.getClass(),
    ])
    if (isPublic) return true
    // ... 校验 token
    const roles = this.reflector.getAllAndOverride(Roles, [ctx.getHandler(), ctx.getClass()])
    if (!roles?.length) return true
    const user = ctx.switchToHttp().getRequest().user
    return roles.some(r => user.roles.includes(r))
  }
}
```

| 方法 | 行为 |
|------|------|
| `get(key, target)` | 只读一个目标 |
| `getAllAndOverride(key, [handler, class])` | 按顺序找，返回**第一个非 undefined** 的值 → 方法级覆盖类级 |
| `getAllAndMerge(key, [handler, class])` | 合并所有目标的值（数组拼接、对象合并）→ 方法级和类级都生效 |

模式：**全局守卫默认要求登录，`@Public()` 显式豁免**（默认安全），比在每个控制器上加守卫更不容易遗漏。

**口述模板（30–60 秒）**

1. 一句话结论：SetMetadata 或 createDecorator 写元数据，守卫中用 Reflector 读取。
2. 关键机制：getAllAndOverride 让方法级覆盖类级，getAllAndMerge 合并两者。
3. 注意点：默认安全，用全局守卫加 Public 豁免。
4. 可追问时补充：Reflector.createDecorator 提供类型安全，避免字符串 key。

**常见追问**

- 元数据存在哪里？→ 通过 `Reflect.defineMetadata` 存在目标函数或类上，`Reflector` 是对 `Reflect.getMetadata` 的封装。

---

### Q5. `applyDecorators` 有什么用？

- **难度**: mid
- **标签**: 装饰器组合

**答案**

把多个装饰器组合成一个，减少重复并统一规范：

```ts
export function Auth(...roles: Role[]) {
  return applyDecorators(
    Roles(roles),
    UseGuards(JwtAuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: '未登录' }),
  )
}

@Auth('admin')
@Delete(':id')
remove() {}
```

好处：权限、文档、限流等「总是一起出现」的配置集中定义，修改一处即可全局生效。

**口述模板（30–60 秒）**

1. 一句话结论：applyDecorators 把多个装饰器合并为一个语义化的装饰器。
2. 关键机制：依次应用传入的装饰器。
3. 注意点：组合装饰器要保持单一职责，避免隐藏过多行为。
4. 可追问时补充：常用于把鉴权、角色和 Swagger 文档合在一起。

**常见追问**

- 装饰器的执行顺序？→ 同一目标上的装饰器从下往上应用；但守卫的执行顺序由 `UseGuards` 的参数顺序决定。

---

## 二、依赖注入进阶

### Q6. 自定义 Provider 有哪几种？Injection Token 有哪些形式？

- **难度**: mid
- **标签**: Provider, 高频

**答案**

```ts
providers: [
  UsersService,                                                  // 简写，等同于 useClass
  { provide: PaymentGateway, useClass: process.env.MOCK ? MockGateway : AlipayGateway },
  { provide: 'APP_NAME', useValue: 'shop' },
  {
    provide: REDIS,                                              // Symbol Token
    useFactory: async (cfg: ConfigService) => new Redis(cfg.getOrThrow('REDIS_URL')),
    inject: [ConfigService],                                     // 工厂的依赖
  },
  { provide: 'LegacyLogger', useExisting: AppLogger },           // 别名：指向同一实例
]

export const REDIS = Symbol('REDIS')
constructor(@Inject(REDIS) private readonly redis: Redis) {}
```

| 类型 | 用途 |
|------|------|
| `useClass` | 按环境切换实现、面向抽象编程 |
| `useValue` | 常量、配置、测试 Mock |
| `useFactory` | 需要异步初始化或依赖其他 Provider（数据库连接、SDK 客户端） |
| `useExisting` | 别名，两个 Token 指向同一个实例 |

Token 可以是：**类**、**字符串**、**Symbol**。

**为什么 interface 不能作为 Token**：interface 编译后不存在，运行时没有值可以作为 key。解决办法：

1. 使用 **abstract class** 作为 Token（运行时存在，同时能描述类型）
2. 使用 Symbol 常量 + `@Inject(TOKEN)`

**口述模板（30–60 秒）**

1. 一句话结论：四种自定义 Provider：useClass、useValue、useFactory、useExisting。
2. 关键机制：Token 是查找 Provider 的 key，可以是类、字符串、Symbol。
3. 注意点：interface 运行时不存在，不能作为 Token，常用抽象类代替。
4. 可追问时补充：useFactory 支持异步，容器会等待它完成再继续启动。

**常见追问**

- 字符串 Token 有什么问题？→ 容易拼错且可能冲突，推荐 Symbol 或抽象类。

---

### Q7. Provider 有哪些作用域？请求作用域有什么问题？

- **难度**: mid
- **标签**: 作用域, 高频

**答案**

| 作用域 | 实例 | 场景 |
|--------|------|------|
| `DEFAULT`（单例） | 整个应用一个实例 | 绝大多数情况 |
| `REQUEST` | 每个请求一个实例，请求结束后回收 | 需要请求级状态（多租户连接、请求级缓存如 DataLoader） |
| `TRANSIENT` | 每个注入者一个独立实例 | 需要不共享状态的工具，如带上下文名称的 Logger |

```ts
@Injectable({ scope: Scope.REQUEST })
export class TenantContext {
  constructor(@Inject(REQUEST) private readonly req: Request) {}
}
```

**请求作用域的问题**：

1. **向上冒泡**：依赖请求作用域 Provider 的类也会变成请求作用域，一直冒泡到控制器。一个底层 Provider 改为请求作用域，可能导致整条依赖链在每个请求中都重新创建
2. **性能**：每个请求都要实例化整条链，增加延迟和 GC 压力（官方文档提示会影响性能）
3. **不能用于生命周期钩子**，也不能在全局增强器中直接注入

替代方案：

- 使用 **AsyncLocalStorage**（如 `nestjs-cls`）保存请求上下文，Provider 保持单例，需要时从上下文中读取
- 使用 **durable provider** 按租户复用实例（见高级题）

**口述模板（30–60 秒）**

1. 一句话结论：默认单例、请求级、瞬态三种作用域。
2. 关键机制：请求作用域沿注入链向上冒泡。
3. 注意点：冒泡会导致整条链每个请求都重新实例化，性能差。
4. 可追问时补充：优先用 AsyncLocalStorage 保存请求上下文，Provider 保持单例。

**常见追问**

- 单例 Service 中可以保存请求相关的数据吗？→ 不能。单例在所有并发请求间共享，存成员变量会导致数据串号。

---

### Q8. 瞬态作用域（TRANSIENT）有什么典型用途？

- **难度**: mid
- **标签**: 作用域

**答案**

典型用途：**每个使用者需要自己独立状态**的 Provider，例如带上下文名称的 Logger：

```ts
@Injectable({ scope: Scope.TRANSIENT })
export class ContextLogger extends ConsoleLogger {}

@Injectable()
export class OrdersService {
  constructor(private readonly logger: ContextLogger) {
    this.logger.setContext(OrdersService.name)    // 每个注入者有自己的实例，互不影响
  }
}
```

还可以注入 `INQUIRER` 拿到「谁在注入我」：

```ts
@Injectable({ scope: Scope.TRANSIENT })
export class NamedLogger {
  constructor(@Inject(INQUIRER) private readonly parent: object) {}
  log(msg: string) { console.log(`[${this.parent?.constructor?.name}] ${msg}`) }
}
```

注意：瞬态 Provider **不会向上冒泡**（注入者如果是单例，那么它拿到的瞬态实例也只创建一次）；也不会触发生命周期钩子。

**口述模板（30–60 秒）**

1. 一句话结论：瞬态作用域为每个注入者创建独立实例。
2. 关键机制：可以注入 INQUIRER 获取注入者。
3. 注意点：单例注入者拿到的瞬态实例只创建一次，并不是每次调用都新建。
4. 可追问时补充：最常见的用途是带上下文的 Logger。

**常见追问**

- 瞬态和请求作用域的区别？→ 瞬态按「注入者」区分实例，请求作用域按「请求」区分实例。

---

### Q9. 动态模块是什么？`forRoot`、`forRootAsync`、`register`、`forFeature` 有什么约定？

- **难度**: mid
- **标签**: 动态模块, 高频

**答案**

动态模块：一个静态方法返回 `DynamicModule` 对象，**根据参数决定模块提供哪些 Provider**，用于可配置的模块。

```ts
@Module({})
export class RedisModule {
  static forRoot(opts: RedisOptions): DynamicModule {
    return {
      module: RedisModule,
      global: true,
      providers: [{ provide: REDIS, useFactory: () => new Redis(opts) }],
      exports: [REDIS],
    }
  }
}
```

命名约定（社区惯例，非强制）：

| 方法 | 含义 |
|------|------|
| `forRoot` / `forRootAsync` | 只配置一次，整个应用共享（数据库连接、Redis） |
| `register` / `registerAsync` | 每次导入可以有不同配置（HTTP 客户端访问不同的服务） |
| `forFeature` | 在已有 forRoot 的基础上，为某个功能模块追加配置（`TypeOrmModule.forFeature([User])`） |

`Async` 版本：配置依赖其他 Provider（如 `ConfigService`）时使用 `useFactory` + `inject` + `imports`。

使用 `ConfigurableModuleBuilder` 可以自动生成 `register` / `registerAsync` 及其选项类型，避免样板代码（见手写题 H5）。

**口述模板（30–60 秒）**

1. 一句话结论：动态模块根据参数返回模块定义，实现可配置的模块。
2. 关键机制：forRoot 全局一次配置，register 每次导入配置，forFeature 追加功能级配置。
3. 注意点：配置依赖其他服务时使用 Async 版本。
4. 可追问时补充：ConfigurableModuleBuilder 自动生成这些方法和类型。

**常见追问**

- 同一个动态模块用不同参数 `register` 两次会怎样？→ 会得到两个独立的模块实例（Nest 根据模块的元数据生成不同的 key）。

---

### Q10. `@Global()` 模块的优缺点是什么？

- **难度**: mid
- **标签**: 模块

**答案**

`@Global()` 模块只需在根模块导入一次，其导出的 Provider 在所有模块中都能注入，无需再 `imports`。

优点：对配置、数据库连接、Logger、Redis 这类**几乎所有模块都需要**的基础设施，省去大量重复的 imports。

缺点：

- **依赖关系变隐式**：看模块的 imports 无法知道它依赖了什么
- 模块难以单独测试和复用（测试时要记得提供全局依赖）
- 滥用后模块边界失效

原则：只把真正的基础设施做成全局模块；业务模块必须显式 imports。

**口述模板（30–60 秒）**

1. 一句话结论：全局模块导入一次就到处可注入，适合基础设施。
2. 关键机制：导出的 Provider 进入全局可见范围。
3. 注意点：依赖变得隐式，滥用会破坏模块边界。
4. 可追问时补充：业务模块不应该是全局的。

**常见追问**

- `ConfigModule.forRoot({ isGlobal: true })` 是怎么实现的？→ 返回的动态模块对象中设置 `global: true`，效果与 `@Global()` 相同。

---

### Q11. 循环依赖是怎么产生的？怎么解决？

- **难度**: mid
- **标签**: 循环依赖, 高频

**答案**

两种循环：

1. **Provider 循环**：`AService` 依赖 `BService`，`BService` 依赖 `AService`
2. **模块循环**：`UsersModule` 导入 `OrdersModule`，`OrdersModule` 也导入 `UsersModule`

现象：启动报错「Nest can't resolve dependencies」，或注入的值是 `undefined`。另一个常见原因是 **文件之间的循环 import**（如通过 barrel 文件 `index.ts` 互相导入），导致装饰器执行时类还是 `undefined`。

`forwardRef` 解决：

```ts
@Injectable()
export class UsersService {
  constructor(@Inject(forwardRef(() => OrdersService)) private readonly orders: OrdersService) {}
}
@Injectable()
export class OrdersService {
  constructor(@Inject(forwardRef(() => UsersService)) private readonly users: UsersService) {}
}

@Module({ imports: [forwardRef(() => OrdersModule)] })
export class UsersModule {}
```

`forwardRef` 只是延迟引用，**实例化顺序不确定**，不能在构造函数中使用对方。

更好的解法（循环依赖往往说明设计有问题）：

1. **抽取公共部分**到第三个模块，两者都依赖它
2. **事件解耦**：A 发出事件，B 订阅，不再直接调用
3. **调整职责**：把互相调用的逻辑合并，或上移到更高层的编排服务（用例层）
4. 使用 `ModuleRef` 在运行时按需获取（只作为临时手段）

**口述模板（30–60 秒）**

1. 一句话结论：Provider 或模块互相依赖会形成循环，forwardRef 可以临时解决。
2. 关键机制：forwardRef 延迟解析引用，打破定义期的循环。
3. 注意点：实例化顺序不确定，构造函数里不能使用对方；文件级循环 import 也会导致问题。
4. 可追问时补充：根本解法是抽取公共模块、事件解耦或增加编排层。

**常见追问**

- 怎么发现文件级的循环 import？→ 使用 madge 或 dependency-cruiser 检查，并在 CI 中阻止。

---

### Q12. `ModuleRef` 有什么用？

- **难度**: mid
- **标签**: ModuleRef

**答案**

`ModuleRef` 让你在运行时从容器中获取实例：

```ts
@Injectable()
export class PaymentService {
  constructor(private readonly moduleRef: ModuleRef) {}

  // 策略模式：按支付渠道动态获取实现
  getGateway(channel: 'alipay' | 'wechat'): PaymentGateway {
    const token = channel === 'alipay' ? AlipayGateway : WechatGateway
    return this.moduleRef.get(token, { strict: false })   // strict: false 允许从其他模块获取
  }
}
```

| 方法 | 用途 |
|------|------|
| `get(token)` | 获取单例实例 |
| `resolve(token, contextId?)` | 获取请求作用域或瞬态实例（每次可能创建新实例） |
| `create(Class)` | 动态实例化一个未注册为 Provider 的类 |
| `registerRequestByContextId` | 在非 HTTP 场景（如队列消费）手动构造请求上下文 |

`LazyModuleLoader` 可以在运行时才加载某个模块，用于减少 Serverless 冷启动时间。

注意：大量使用 `ModuleRef` 相当于服务定位器模式，依赖关系变得隐式，应限制在策略分发、插件这类场景。

**口述模板（30–60 秒）**

1. 一句话结论：ModuleRef 在运行时从容器中获取或创建实例。
2. 关键机制：get 取单例，resolve 取作用域实例，create 动态实例化。
3. 注意点：滥用会让依赖隐式化。
4. 可追问时补充：适合策略模式分发，LazyModuleLoader 用于减少冷启动。

**常见追问**

- 策略模式还有更好的写法吗？→ 用一个 Token 注入所有实现的数组（自定义 Provider 用 `useFactory` 聚合），再按 key 查找，依赖更显式。

---

## 三、鉴权、校验与序列化

### Q13. JWT 认证在 Nest 中怎么设计？Access Token 和 Refresh Token 怎么配合？

- **难度**: mid
- **标签**: 认证, JWT

**答案**

两种实现：

1. **Passport**（`@nestjs/passport` + `passport-jwt`）：用策略（Strategy）封装验证逻辑，适合需要多种登录方式（本地、OAuth、SSO）的系统
2. **自定义守卫** + `@nestjs/jwt`：更直接，便于控制细节

双 Token 方案：

```text
登录 → access token（短期，如 15 分钟，放在内存或 Authorization 头）
     → refresh token（长期，如 7 天，HttpOnly + Secure + SameSite Cookie，服务端存储其 ID）

access 过期 → 用 refresh 换新的 access + 新的 refresh（轮换）
旧 refresh 被再次使用 → 判定为泄露，吊销该用户的整个 refresh 家族
登出 / 改密 → 吊销 refresh；access 在短期内自然过期（或加入黑名单）
```

要点：

- JWT 无状态，**无法主动失效**，所以 access token 要短；需要即时失效（如封禁账号）时，维护 Redis 黑名单或用户的 `tokenVersion`
- 载荷中不放敏感信息（JWT 只是 Base64 编码）
- 签名使用强密钥或非对称算法（RS256 / ES256，便于多服务只用公钥验签）
- 校验 `exp`、`iss`、`aud`，限制允许的算法

**口述模板（30–60 秒）**

1. 一句话结论：短期 access token 加长期 refresh token，refresh 轮换并存储在服务端。
2. 关键机制：守卫校验 access token，过期时用 refresh 换新。
3. 注意点：JWT 无法主动失效，需要黑名单或版本号；refresh 放 HttpOnly Cookie。
4. 可追问时补充：refresh 重放检测，发现复用即吊销整个令牌家族。

**常见追问**

- Session 和 JWT 怎么选？→ 单体 Web 应用用 Session 更简单、可即时失效；多服务、移动端、跨域场景用 JWT 更方便。

---

### Q14. 授权怎么做？RBAC 和 ABAC 有什么区别？

- **难度**: mid
- **标签**: 授权, RBAC

**答案**

| 模型 | 判断依据 | 实现 |
|------|---------|------|
| RBAC（基于角色） | 用户角色 → 权限 | `@Roles('admin')` + RolesGuard |
| ABAC（基于属性） | 用户属性、资源属性、环境 | CASL 等策略库 |

RBAC 的不足：无法表达「只能编辑**自己的**文章」「只能看本部门的订单」这类**资源级**权限。

ABAC（CASL）示例：

```ts
defineAbilityFor(user: User) {
  const { can, build } = new AbilityBuilder(createMongoAbility)
  if (user.role === 'admin') can('manage', 'all')
  else {
    can('read', 'Article', { published: true })
    can('update', 'Article', { authorId: user.id })   // 只能改自己的
  }
  return build()
}
```

分层落地：

1. **守卫**：做与资源无关的判断（是否登录、是否有某种角色或权限点）
2. **Service**：加载资源后做资源级判断（是否为作者、是否同租户）
3. **查询层**：列表查询时把权限条件转换为 where 条件（如 `tenantId = ?`），避免先查全部再过滤

**口述模板（30–60 秒）**

1. 一句话结论：RBAC 按角色判断，ABAC 按属性判断，可以表达资源级权限。
2. 关键机制：守卫做粗粒度判断，Service 做资源级判断。
3. 注意点：列表查询要把权限转成查询条件。
4. 可追问时补充：CASL 能把规则转换为数据库查询条件。

**常见追问**

- 为什么资源级权限不放在守卫里？→ 守卫在管道之前执行，还没有加载资源；在守卫中查库会重复查询，并把业务逻辑混入守卫。

---

### Q15. `ValidationPipe` 有哪些进阶用法和坑？

- **难度**: mid
- **标签**: 校验

**答案**

```ts
new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true,
  transform: true,
  transformOptions: { enableImplicitConversion: false },
  stopAtFirstError: false,
  exceptionFactory: errors => new BadRequestException(formatErrors(errors)),
})
```

常见坑：

1. **隐式转换**：`enableImplicitConversion: true` 会按类型元数据转换，`'false'` 转为 boolean 会得到 `true`（非空字符串）。查询参数建议显式使用 `@Type(() => Number)` 或 `@Transform`
2. **嵌套对象**：必须 `@ValidateNested()` + `@Type(() => Child)`，否则不会校验子对象
3. **数组参数**：`@Query('ids', new ParseArrayPipe({ items: Number, separator: ',' }))`
4. **泛型和联合类型**：TS 元数据无法表达，ValidationPipe 拿不到正确类型
5. **性能**：class-transformer 处理大对象较慢

DTO 复用（`@nestjs/mapped-types` 或 `@nestjs/swagger` 中的同名工具）：

```ts
export class UpdateUserDto extends PartialType(CreateUserDto) {}
export class LoginDto extends PickType(CreateUserDto, ['email', 'password'] as const) {}
```

这些工具类型会同时复制校验装饰器和 Swagger 元数据。

**口述模板（30–60 秒）**

1. 一句话结论：whitelist + transform 是基础配置，嵌套和数组需要额外声明。
2. 关键机制：class-transformer 转实例，class-validator 读装饰器校验。
3. 注意点：隐式转换会把字符串 'false' 转成 true。
4. 可追问时补充：PartialType 等工具复用 DTO 并保留校验规则。

**常见追问**

- 如何统一校验错误的返回格式？→ 使用 `exceptionFactory` 把错误数组转换为 `{ field, message }` 列表。

---

### Q16. Nest 12 的 Standard Schema 校验怎么用？和 class-validator 怎么选？

- **难度**: mid
- **标签**: 校验, v12

**答案**

Standard Schema 是 Zod、Valibot、ArkType 等校验库共同实现的标准接口。Nest 12 中：

```ts
const createUserSchema = z.object({
  email: z.string().email(),
  age: z.number().int().min(0).optional(),
})
type CreateUser = z.infer<typeof createUserSchema>

@Post()
create(@Body({ schema: createUserSchema }) body: CreateUser) {}

@Get(':id')
findOne(@Param('id', { schema: z.coerce.number().int().positive() }) id: number) {}

// 必须注册校验管道，否则 schema 只是挂在元数据上，不会被执行
app.useGlobalPipes(new StandardSchemaValidationPipe())
```

响应端可以用 `StandardSchemaSerializerInterceptor` + `@SerializeOptions({ schema })` 校验和裁剪输出。

原理：`schema` 选项只是把 schema 附加到参数的 `ArgumentMetadata` 上，由 `StandardSchemaValidationPipe` 读取并执行；其他管道会忽略它。

| 维度 | class-validator | Standard Schema（Zod 等） |
|------|----------------|--------------------------|
| 定义方式 | 类 + 装饰器 | schema 对象 |
| 类型来源 | 类本身 | `z.infer` 推导 |
| 依赖 | `emitDecoratorMetadata`、reflect-metadata | 不依赖装饰器元数据 |
| 前后端共享 | 难 | 容易，同一 schema 可在前端使用 |
| Swagger 集成 | 成熟 | 需要额外转换 |

建议：已有大量 DTO 的项目保持 class-validator；新项目、需要前后端共享 schema 或使用 Node 类型擦除的项目考虑 Standard Schema。

**口述模板（30–60 秒）**

1. 一句话结论：v12 支持在参数装饰器上传入 Zod 等 schema，由 StandardSchemaValidationPipe 执行校验。
2. 关键机制：schema 挂在参数元数据上，管道读取后校验。
3. 注意点：不注册该管道时 schema 不会生效。
4. 可追问时补充：schema 可以前后端共享，也摆脱了对装饰器元数据的依赖。

**常见追问**

- 两种方式能混用吗？→ 能。两个管道都注册时，各自只处理自己能识别的参数，渐进迁移时很有用。

---

### Q17. 怎么避免把敏感字段返回给前端？

- **难度**: mid
- **标签**: 序列化, 安全

**答案**

问题：直接返回数据库实体，可能把 `passwordHash`、`deletedAt`、内部字段返回给前端。

方案一：`ClassSerializerInterceptor` + class-transformer

```ts
export class UserEntity {
  id!: number
  email!: string
  @Exclude() passwordHash!: string
  @Expose({ groups: ['admin'] }) phone!: string
}

@UseInterceptors(ClassSerializerInterceptor)
@SerializeOptions({ groups: ['user'] })
@Get(':id')
async findOne(@Param('id') id: number) {
  return new UserEntity(await this.users.findOne(id))   // 必须返回类实例，装饰器才生效
}
```

方案二：**显式的响应 DTO**（推荐）：Service 返回实体，Controller 映射为响应 DTO，只包含允许返回的字段。

方案三（v12）：`StandardSchemaSerializerInterceptor` + 响应 schema，未声明的字段会被去掉。

要点：**白名单优于黑名单**。用 `@Exclude` 是黑名单，实体新增敏感字段时容易遗漏；响应 DTO 或 schema 是白名单。

**口述模板（30–60 秒）**

1. 一句话结论：不要直接返回实体，用序列化或响应 DTO 控制输出。
2. 关键机制：ClassSerializerInterceptor 根据 Exclude、Expose 处理类实例。
3. 注意点：返回普通对象时装饰器不生效；黑名单容易遗漏。
4. 可追问时补充：推荐白名单式的响应 DTO 或 v12 的响应 schema。

**常见追问**

- 为什么返回普通对象时 `@Exclude` 不生效？→ class-transformer 依赖类的元数据，普通对象没有原型上的装饰器信息。

---

## 四、数据与一致性

### Q18. 数据库事务在 Nest 中怎么做？如何跨多个 Service？

- **难度**: mid
- **标签**: 事务, 高频

**答案**

TypeORM 的基本写法：

```ts
await this.dataSource.transaction(async manager => {
  const order = await manager.save(Order, dto)
  await manager.decrement(Product, { id: dto.productId }, 'stock', dto.qty)
})
```

问题：要跨多个 Service 共享同一个事务，就得把 `manager` 一层层作为参数传下去，侵入所有方法签名。

方案对比：

| 方案 | 做法 | 缺点 |
|------|------|------|
| 显式传递 manager / tx | 每个方法多一个参数 | 侵入性强 |
| 请求作用域的 UnitOfWork | 每个请求一个事务上下文 | 作用域冒泡，性能差 |
| **AsyncLocalStorage**（推荐） | 开启事务时把 tx 存入异步上下文，Repository 自动读取 | 需要封装 |

`nestjs-cls` + `@nestjs-cls/transactional`：

```ts
@Injectable()
export class OrdersService {
  constructor(private readonly txHost: TransactionHost<TransactionalAdapterPrisma>) {}

  @Transactional()
  async placeOrder(dto: PlaceOrderDto) {
    await this.inventory.deduct(dto)       // 内部使用 txHost.tx，自动加入同一事务
    return this.txHost.tx.order.create({ data: dto })
  }
}
```

注意：

- 事务中**不要调用外部 HTTP 或发送消息**（长事务、无法回滚），用「事务提交后再发送」或 Outbox 模式
- 嵌套事务的传播行为（加入现有事务 / 新建事务）要明确

**口述模板（30–60 秒）**

1. 一句话结论：单个方法用 dataSource.transaction，跨 Service 推荐基于 AsyncLocalStorage 的事务上下文。
2. 关键机制：开启事务时把事务客户端存入异步上下文，下游自动读取。
3. 注意点：避免请求作用域方案；事务中不调用外部服务。
4. 可追问时补充：消息发送用 Outbox 模式保证与事务一致。

**常见追问**

- 为什么事务中不能发消息？→ 消息发出后事务可能回滚，导致下游处理了不存在的数据；或消息发送失败但事务已提交。

---

### Q19. GraphQL 或关联查询中的 N+1 问题怎么解决？

- **难度**: mid
- **标签**: 性能, GraphQL

**答案**

N+1：查询 N 篇文章后，对每篇文章再查一次作者，共 1 + N 次查询。

解决：

1. **ORM 预加载**：`relations: ['author']` 或 Prisma 的 `include`（REST 场景）
2. **DataLoader**（GraphQL 场景）：在同一事件循环周期内收集所有 key，合并为一次批量查询，并在请求内缓存

```ts
@Injectable({ scope: Scope.REQUEST })      // DataLoader 必须是请求级的，避免跨请求缓存
export class UserLoader {
  readonly byId = new DataLoader<number, User>(async ids => {
    const users = await this.repo.findBy({ id: In([...ids]) })
    const map = new Map(users.map(u => [u.id, u]))
    return ids.map(id => map.get(id) ?? new Error(`user ${id} not found`))   // 顺序必须和 ids 一致
  })
  constructor(private readonly repo: UsersRepository) {}
}

@ResolveField(() => User)
author(@Parent() post: Post) { return this.loader.byId.load(post.authorId) }
```

为了避免请求作用域冒泡带来的性能问题，可以把 Loader 实例放在 GraphQL 的 context 中，或用 AsyncLocalStorage 按请求创建。

**口述模板（30–60 秒）**

1. 一句话结论：REST 用预加载，GraphQL 用 DataLoader 合并查询。
2. 关键机制：DataLoader 在一个事件循环周期内收集 key 并批量查询。
3. 注意点：Loader 必须按请求隔离，返回结果的顺序要和 key 一致。
4. 可追问时补充：用 context 或 AsyncLocalStorage 创建 Loader，避免请求作用域冒泡。

**常见追问**

- DataLoader 为什么要求返回顺序和 key 一致？→ 它按下标把结果分发给每个 `load()` 调用。

---

### Q20. 缓存三大问题在 Nest 服务中怎么处理？

- **难度**: mid
- **标签**: 缓存

**答案**

| 问题 | 现象 | 处理 |
|------|------|------|
| 穿透 | 查询不存在的数据，每次都打到数据库 | 缓存空值（短 TTL）、参数校验、布隆过滤器 |
| 击穿 | 热点 key 过期瞬间大量请求打到数据库 | 互斥重建（分布式锁）、单飞（single flight）、逻辑过期 |
| 雪崩 | 大量 key 同时过期或缓存宕机 | TTL 加随机抖动、多级缓存、熔断降级 |

单实例内的请求合并：

```ts
private readonly inflight = new Map<string, Promise<Product | null>>()

async getProduct(id: string) {
  const key = `product:${id}`
  const cached = await this.cache.get<Product | 'NULL'>(key)
  if (cached !== undefined) return cached === 'NULL' ? null : cached

  // 同一实例内相同 key 只回源一次，其他请求等待同一个 Promise
  let p = this.inflight.get(key)
  if (!p) {
    p = this.repo.findOne(id)
      .then(async v => {
        const ttl = 60_000 + Math.floor(Math.random() * 10_000)
        await this.cache.set(key, v ?? 'NULL', v ? ttl : 5_000)
        return v
      })
      .finally(() => this.inflight.delete(key))
    this.inflight.set(key, p)
  }
  return p
}
```

多实例之间的击穿还需要 Redis 分布式锁。更新数据时采用「先更新数据库，再删除缓存」，配合延迟双删或订阅 binlog 删除缓存。

**口述模板（30–60 秒）**

1. 一句话结论：穿透缓存空值，击穿互斥重建，雪崩加随机 TTL 和多级缓存。
2. 关键机制：单飞合并同一实例内的并发回源。
3. 注意点：多实例需要分布式锁；更新时先写库再删缓存。
4. 可追问时补充：可以通过订阅 binlog 异步删除缓存，保证最终一致。

**常见追问**

- 为什么是删除缓存而不是更新缓存？→ 并发更新时，更新缓存的顺序可能和写库顺序不一致，导致缓存中是旧值；删除更简单可靠。

---

## 五、拦截器与通信

### Q21. 拦截器有哪些进阶用法？

- **难度**: mid
- **标签**: 拦截器, RxJS

**答案**

```ts
// 1. 超时
return next.handle().pipe(
  timeout(5000),
  catchError(err => err instanceof TimeoutError
    ? throwError(() => new RequestTimeoutException())
    : throwError(() => err)),
)

// 2. 缓存：命中时不调用处理器
const hit = await this.cache.get(key)
if (hit) return of(hit)
return next.handle().pipe(tap(data => this.cache.set(key, data)))

// 3. 幂等：根据 Idempotency-Key 返回第一次的结果
// 4. 审计日志：记录操作人、参数、结果
// 5. 埋点与指标：记录耗时并按路由上报
```

注意：

- 拦截器中的 `timeout` 只是**不再等待**，处理器中的数据库查询仍在继续执行。真正取消需要把 `AbortSignal` 传递到下游
- 只有返回 Observable 的部分可以被拦截器处理；使用 `@Res()` 手动响应时拦截器的返回值处理不生效
- SSE、流式响应要小心 `map` 操作符改变数据结构

**口述模板（30–60 秒）**

1. 一句话结论：拦截器常用于超时、缓存、幂等、审计、埋点。
2. 关键机制：基于 RxJS 操作符包裹处理器的执行。
3. 注意点：timeout 不会取消下游正在执行的操作。
4. 可追问时补充：真正的取消需要 AbortSignal 贯穿到底层客户端。

**常见追问**

- 拦截器的 `intercept` 可以是 async 吗？→ 可以返回 `Promise<Observable>`，比如先查缓存再决定是否调用处理器。

---

### Q22. 日志、鉴权、参数校验、响应包装分别应该放在哪个组件？为什么？

- **难度**: mid
- **标签**: 职责划分

**答案**

| 需求 | 组件 | 原因 |
|------|------|------|
| 请求 ID、访问日志、CORS、安全头 | 中间件 | 与具体路由无关，越早越好 |
| 认证、授权 | 守卫 | 需要处理器元数据，且要在业务逻辑前拒绝 |
| 参数转换、校验 | 管道 | 专门处理处理器参数 |
| 响应包装、耗时统计、缓存、超时 | 拦截器 | 需要处理器前后的逻辑和返回值 |
| 错误格式化、错误日志 | 过滤器 | 统一出口 |
| 业务规则、资源级权限 | Service | 需要业务数据 |

反例：

- 在中间件中做基于角色的鉴权（拿不到处理器元数据）
- 在拦截器中做鉴权（拦截器在守卫之后执行，已经晚了）
- 在守卫中做参数校验（守卫在管道之前，参数还没转换）

**口述模板（30–60 秒）**

1. 一句话结论：中间件做通用预处理，守卫做准入，管道处理参数，拦截器包装执行，过滤器处理异常。
2. 关键机制：每个组件在生命周期中的位置决定了它能拿到什么信息。
3. 注意点：鉴权放守卫，不要放在中间件或拦截器中。
4. 可追问时补充：资源级权限需要业务数据，放在 Service 中。

**常见追问**

- 访问日志要记录响应状态码和耗时，放中间件还是拦截器？→ 中间件监听 `res.on('finish')` 能覆盖所有请求（包括 404 和守卫拒绝的请求）；拦截器只覆盖进入处理器的请求。

---

### Q23. 微服务中 `@MessagePattern` 和 `@EventPattern` 有什么区别？传输层怎么选？

- **难度**: mid
- **标签**: 微服务

**答案**

| 维度 | `@MessagePattern` + `send()` | `@EventPattern` + `emit()` |
|------|-----------------------------|---------------------------|
| 模式 | 请求-响应 | 发布事件，不等待响应 |
| 返回 | 调用方等待结果 | 不返回结果 |
| 订阅者 | 一个 | 可以多个 |
| 适用 | 查询、需要结果的命令 | 领域事件通知 |

传输层选型：

| 传输层 | 特点 | 适用 |
|--------|------|------|
| TCP | 简单、无中间件 | 内部简单调用 |
| gRPC | 强类型契约（proto）、高性能、流式 | 跨语言服务间调用 |
| Redis | 简单发布订阅，**不持久化** | 低要求的通知 |
| NATS | 轻量、低延迟；JetStream 提供持久化 | 实时消息 |
| RabbitMQ | 确认机制、路由灵活、死信队列 | 可靠的任务和事件 |
| Kafka | 高吞吐、持久化、可回放、分区有序 | 事件流、日志、大数据管道 |

混合应用：`app.connectMicroservice(...)` + `app.startAllMicroservices()`，同一个应用既提供 HTTP 又消费消息。

**口述模板（30–60 秒）**

1. 一句话结论：MessagePattern 是请求响应，EventPattern 是发布事件。
2. 关键机制：传输层可插拔，同一套处理器可以换不同的传输层。
3. 注意点：Redis 发布订阅不持久化，消费者离线就会丢消息。
4. 可追问时补充：跨语言选 gRPC，可靠任务选 RabbitMQ，事件流选 Kafka。

**常见追问**

- Nest 12 中 NATS 有什么变化？→ 改为使用 `@nats-io/transport-node`（NATS v3），消息以 JSON 字符串序列化，自定义反序列化器会收到完整的消息对象。

---

### Q24. 消息消费如何保证可靠性？

- **难度**: mid
- **标签**: 消息队列, 可靠性

**答案**

以 RabbitMQ 为例：

```ts
app.connectMicroservice({
  transport: Transport.RMQ,
  options: { urls: [url], queue: 'orders', noAck: false, queueOptions: { durable: true } },
})

@EventPattern('order.paid')
async onPaid(@Payload() data: OrderPaid, @Ctx() ctx: RmqContext) {
  const channel = ctx.getChannelRef()
  const msg = ctx.getMessage()
  try {
    await this.handle(data)          // 处理逻辑必须幂等
    channel.ack(msg)
  } catch (e) {
    channel.nack(msg, false, false)  // 不重新入队，转入死信队列，避免无限重试
  }
}
```

可靠性的三个层面：

1. **生产端**：发布确认（publisher confirm）；业务数据与消息的一致性使用 **Outbox 模式**（事务中写入消息表，由独立进程投递）
2. **Broker**：队列和消息持久化、集群副本
3. **消费端**：手动 ack；**幂等**（消息 ID 去重表或业务唯一键）；有限次数重试 + 死信队列 + 告警

消息语义：实际工程中通常是「**至少一次**投递 + **幂等**消费 = 效果上的恰好一次」。

**口述模板（30–60 秒）**

1. 一句话结论：生产端确认加 Outbox，Broker 持久化，消费端手动 ack 加幂等。
2. 关键机制：至少一次投递配合幂等消费。
3. 注意点：失败消息不要无限重新入队，要进入死信队列。
4. 可追问时补充：Outbox 模式解决数据库事务和消息发送的一致性。

**常见追问**

- 幂等怎么实现？→ 消息带唯一 ID，消费前在数据库中插入去重记录（唯一索引），插入失败说明已处理过；或者依赖业务状态机（只有「待支付」状态才能变为「已支付」）。

---

### Q25. EventEmitter 模块和 CQRS 模块分别适合什么场景？

- **难度**: mid
- **标签**: 事件, CQRS

**答案**

**`@nestjs/event-emitter`**：进程内事件，用于模块解耦：

```ts
this.events.emit('user.registered', new UserRegistered(user.id))

@OnEvent('user.registered', { async: true })
async sendWelcome(e: UserRegistered) {}
```

注意：**进程内事件不持久化**，进程崩溃时事件丢失；监听器抛出的异常要单独处理，否则可能变成未处理的 Promise 拒绝。需要可靠性时改用消息队列或 Outbox。

**`@nestjs/cqrs`**：

- Command（写）→ CommandHandler
- Query（读）→ QueryHandler
- Event → EventHandler
- Saga：监听事件并触发新的命令，编排长流程

适用：领域复杂、读写模型差异大、需要事件溯源的系统。简单的 CRUD 使用 CQRS 会显著增加代码量。

**口述模板（30–60 秒）**

1. 一句话结论：EventEmitter 用于进程内解耦，CQRS 用于复杂领域的读写分离。
2. 关键机制：事件让发出方不依赖订阅方。
3. 注意点：进程内事件不持久化，可靠性要求高时使用消息队列。
4. 可追问时补充：简单 CRUD 不要上 CQRS。

**常见追问**

- 事件解耦能解决循环依赖吗？→ 能。A 发出事件、B 订阅，A 不再需要注入 B。

---

## 六、HTTP 进阶与工程

### Q26. API 版本控制怎么做？

- **难度**: mid
- **标签**: 版本管理

**答案**

```ts
app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' })

@Controller({ path: 'users', version: '2' })
export class UsersV2Controller {}

@Version(['1', '2'])
@Get('profile')
profile() {}
```

| 方式 | 示例 | 特点 |
|------|------|------|
| URI | `/v1/users` | 最直观，便于缓存和网关路由 |
| Header | `X-API-Version: 2` | URL 不变 |
| Media Type | `Accept: application/json;v=2` | 符合 REST 规范，使用门槛高 |
| 自定义 | 按任意规则提取 | 灵活 |

版本策略：只在**破坏性变更**时升级大版本；旧版本设置下线时间，通过日志统计旧版本调用量，通知调用方迁移。

**口述模板（30–60 秒）**

1. 一句话结论：Nest 支持 URI、Header、Media Type 和自定义四种版本控制方式。
2. 关键机制：enableVersioning 开启，控制器或方法上声明版本。
3. 注意点：只在破坏性变更时升级版本。
4. 可追问时补充：统计旧版本调用量，制定下线计划。

**常见追问**

- 能不能不做版本控制？→ 内部接口可以通过「只做向后兼容的变更」避免版本；对外开放的 API 必须有版本策略。

---

### Q27. 文件上传怎么处理？大文件怎么办？

- **难度**: mid
- **标签**: 文件上传

**答案**

```ts
@Post('avatar')
@UseInterceptors(FileInterceptor('file', { limits: { fileSize: 2 * 1024 * 1024 } }))
upload(
  @UploadedFile(new ParseFilePipe({
    validators: [
      new MaxFileSizeValidator({ maxSize: 2 * 1024 * 1024 }),
      new FileTypeValidator({ fileType: /^image\/(png|jpeg)$/ }),
    ],
  }))
  file: Express.Multer.File,
) {}
```

注意：

- 默认的 multer 内存存储会把整个文件放进内存，大文件会导致内存暴涨
- 文件类型校验不能只信任 `Content-Type`，应检查文件头（magic number）
- Fastify 平台需要使用 `@fastify/multipart`，用法不同

大文件方案：

1. **前端直传对象存储**：服务端只签发预签名 URL（STS 临时凭证），文件不经过 Node 服务
2. **分片上传 + 断点续传**：前端分片，服务端记录分片状态，最后合并
3. 必须经过服务端时使用**流式处理**，直接管道传输到对象存储

**口述模板（30–60 秒）**

1. 一句话结论：小文件用 FileInterceptor + ParseFilePipe，大文件用预签名直传。
2. 关键机制：multer 解析 multipart，校验器限制大小和类型。
3. 注意点：内存存储不适合大文件；类型校验要看文件头。
4. 可追问时补充：分片上传和断点续传配合对象存储的分片接口。

**常见追问**

- 预签名 URL 的安全怎么保证？→ 限制有效期、文件大小、类型和存储路径；上传完成后回调服务端校验。

---

### Q28. 优雅关闭怎么做？

- **难度**: mid
- **标签**: 生产实践, 高频

**答案**

```ts
app.enableShutdownHooks()

@Injectable()
export class ShutdownService implements BeforeApplicationShutdown {
  isShuttingDown = false

  async beforeApplicationShutdown(signal?: string) {
    this.isShuttingDown = true              // 就绪检查开始返回 503
    await sleep(5_000)                      // 等待负载均衡摘除本实例
  }
}
```

完整流程（Kubernetes 场景）：

1. Pod 进入 Terminating，从 Service 的 Endpoints 中移除（这一步是异步的，可能有几秒延迟）
2. 收到 SIGTERM → Nest 依次调用 `onModuleDestroy` → `beforeApplicationShutdown`
3. 就绪检查返回失败，等待几秒，让流量完全切走
4. HTTP 服务器停止接收新连接，等待存量请求完成
5. `onApplicationShutdown`：关闭数据库连接池、Redis、消息消费者（先停止拉取新消息，处理完当前消息）
6. `terminationGracePeriodSeconds` 到期未退出将被 SIGKILL

注意：

- 容器中 Node 进程要能收到信号（不要用 `npm start` 作为 PID 1，或使用 tini）
- 长连接（WebSocket、SSE）需要主动通知客户端重连
- 队列消费者要停止拉取，避免处理到一半被杀

**口述模板（30–60 秒）**

1. 一句话结论：开启 shutdown hooks，先摘流量，再处理完存量请求，最后释放资源。
2. 关键机制：beforeApplicationShutdown 中让就绪检查失败并等待；onApplicationShutdown 中关闭连接。
3. 注意点：容器中要确保进程能收到 SIGTERM。
4. 可追问时补充：长连接和队列消费者需要特殊处理。

**常见追问**

- 为什么收到 SIGTERM 后还要等几秒？→ Endpoints 更新和负载均衡同步有延迟，立即关闭会导致仍被路由过来的请求失败。

---

### Q29. 守卫、拦截器、管道怎么单独做单元测试？

- **难度**: mid
- **标签**: 测试

**答案**

直接实例化并构造一个假的 `ExecutionContext`：

```ts
const createContext = (req: Partial<Request>, handler = () => {}) =>
  ({
    switchToHttp: () => ({ getRequest: () => req }),
    getHandler: () => handler,
    getClass: () => class {},
  }) as unknown as ExecutionContext

it('rejects users without role', async () => {
  const reflector = { getAllAndOverride: vi.fn().mockReturnValue(['admin']) } as unknown as Reflector
  const guard = new RolesGuard(reflector)
  expect(guard.canActivate(createContext({ user: { roles: ['user'] } } as any))).toBe(false)
})

it('wraps response', async () => {
  const interceptor = new TransformInterceptor()
  const next = { handle: () => of({ id: 1 }) }
  const result = await firstValueFrom(interceptor.intercept(createContext({}), next))
  expect(result).toEqual({ code: 0, data: { id: 1 } })
})
```

e2e 测试中可以 `.overrideGuard(JwtAuthGuard).useValue(...)` 跳过鉴权，专注测试业务。

测试分层：单元测试覆盖 Service 和横切组件的逻辑，e2e 测试覆盖关键链路（鉴权、校验、错误格式），用 Testcontainers 启动真实数据库。

**口述模板（30–60 秒）**

1. 一句话结论：直接 new 组件，Mock Reflector 和 ExecutionContext 进行测试。
2. 关键机制：拦截器测试中用 of() 模拟 next.handle 的返回。
3. 注意点：e2e 中可以 overrideGuard 跳过鉴权。
4. 可追问时补充：用 Testcontainers 在 e2e 中使用真实依赖。

**常见追问**

- 为什么不总是用 e2e 测试？→ e2e 慢且难以覆盖所有分支；单元测试快、定位准，两者配合。

---

### Q30. Nest 11 升级到 Express 5 后有哪些需要注意的变化？

- **难度**: mid
- **标签**: 版本升级, v11

**答案**

Nest 11（2025-01）的主要变化：

1. **Node 20+**
2. **Express 5 / Fastify 5**，路由匹配规则变化：
   - 通配符必须命名：`*` → `*splat`；要同时匹配根路径用 `{*splat}`
   - 可选参数不再使用 `?`，改用大括号：`/:file{.:ext}`
   - 不再支持正则字符；`()[]?+!` 等字符被保留，需要转义
3. **cache-manager v6 / Keyv**：存储配置方式变化，**TTL 单位为毫秒**
4. `ConsoleLogger` 支持 JSON 输出和更多格式选项
5. 新增 `ParseDatePipe`；`@Inject()` 的 Token 类型更严格
6. 启动性能优化（模块 key 的生成方式改进）

升级建议：先在测试环境运行全部 e2e 测试，重点关注通配符路由、中间件路径、缓存 TTL。

**口述模板（30–60 秒）**

1. 一句话结论：Nest 11 升级到 Express 5，路由语法和缓存模块变化最大。
2. 关键机制：通配符必须命名，可选参数改用大括号。
3. 注意点：缓存 TTL 改为毫秒，旧配置会导致缓存时间不对。
4. 可追问时补充：要求 Node 20 以上，并有启动性能优化。

**常见追问**

- 中间件 `forRoutes('*')` 在 v11 中还能用吗？→ Nest 内置了兼容转换并给出警告，建议改为 `{*splat}` 等新语法。

---

### Q31. 怎么实现 SSE 和流式响应？

- **难度**: mid
- **标签**: 流式响应

**答案**

SSE（Server-Sent Events）：

```ts
@Sse('progress/:taskId')
progress(@Param('taskId') taskId: string): Observable<MessageEvent> {
  return this.tasks.progress$(taskId).pipe(map(p => ({ data: p }) as MessageEvent))
}
```

文件流（不把整个文件读入内存）：

```ts
@Get('export')
export(@Res({ passthrough: true }) res: Response) {
  res.set({ 'Content-Disposition': 'attachment; filename="orders.csv"' })
  return new StreamableFile(this.reports.createCsvStream())
}
```

LLM 流式输出等场景：对模型返回的流逐块转换为 SSE 事件；客户端断开时要取消上游请求（监听 `req.on('close')` 并调用 `AbortController.abort()`），否则仍在消耗 token 和资源。

注意：

- 反向代理需要关闭缓冲（如 Nginx 的 `X-Accel-Buffering: no`），否则 SSE 会被攒批发送
- 多实例时，进度事件需要通过 Redis 发布订阅等方式跨实例传递

**口述模板（30–60 秒）**

1. 一句话结论：@Sse 返回 Observable 实现 SSE，StreamableFile 实现文件流。
2. 关键机制：流式传输避免把数据全部加载进内存。
3. 注意点：客户端断开要取消上游；代理要关闭缓冲。
4. 可追问时补充：多实例时事件需要通过消息系统广播。

**常见追问**

- SSE 和 WebSocket 怎么选？→ 只需服务端单向推送用 SSE（基于 HTTP、自动重连、简单）；需要双向通信用 WebSocket。

---

### Q32. 怎么防止常见的 Web 安全问题？

- **难度**: mid
- **标签**: 安全

**答案**

| 风险 | Nest 中的防护 |
|------|--------------|
| 批量赋值 | `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true })` |
| SQL 注入 | 使用 ORM 参数化查询；原生 SQL 必须使用参数绑定 |
| XSS | 输出转义由前端负责；接口返回 JSON 并设置正确的 Content-Type；Helmet 设置 CSP |
| CSRF | 使用 Cookie 鉴权时启用 SameSite 和 CSRF Token；使用 Authorization 头时风险较低 |
| 越权访问（IDOR） | Service 层校验资源归属；查询条件中带上用户或租户 ID |
| 暴力破解 | Throttler 限流 + 登录失败锁定 + 验证码 |
| 敏感信息泄露 | 响应 DTO 白名单；生产环境不返回错误堆栈 |
| SSRF | 服务端发起请求时校验目标地址，禁止访问内网 |
| 依赖漏洞 | `npm audit`、Dependabot、锁定版本 |
| 安全头 | `helmet()`；CORS 配置白名单，不要 `origin: '*'` 加 `credentials: true` |

**口述模板（30–60 秒）**

1. 一句话结论：whitelist 防批量赋值，参数化防注入，守卫加资源校验防越权，限流防爆破。
2. 关键机制：把安全措施做成全局默认配置。
3. 注意点：越权访问（IDOR）是业务系统中最常见的漏洞。
4. 可追问时补充：生产环境隐藏错误堆栈，依赖漏洞纳入 CI 扫描。

**常见追问**

- 为什么 IDOR 最常见？→ 框架只能解决「有没有权限调用接口」，无法自动判断「能不能访问这条数据」，需要业务代码逐个处理。
