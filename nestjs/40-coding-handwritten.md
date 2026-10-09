# NestJS 手写与现场编码

## 使用方式

NestJS 的手写题主要有三类：

1. **原理题**：手写迷你 IoC 容器、手写请求处理管线，考察是否真正理解框架
2. **组件题**：守卫、拦截器、过滤器、管道、动态模块，考察 API 熟练度和边界处理
3. **业务基础设施题**：事务装饰器、幂等、分布式锁、多租户，考察生产经验

先自己写、边写边讲思路，再对照参考实现。下文代码基于 Nest 11/12 + TypeScript，第三方库以常见用法为例（ioredis、Prisma、TypeORM、Zod）。

---

### H1. 手写一个迷你 IoC 容器

- **难度**: senior
- **标签**: 原理, DI, 高频
- **面试官看什么**: 是否理解 `design:paramtypes`、Token 查找、单例缓存、循环依赖检测

**题干**

实现 `@Injectable()`、`@Inject(token)` 和 `Container`，支持类 Provider、`useValue`、`useClass`、`useFactory`，单例缓存，并能检测循环依赖。

**思路口述**

1. `@Injectable()` 本身可以什么都不做，它的作用是让 TS 编译器为构造函数输出 `design:paramtypes`
2. `@Inject(token)` 把「参数下标 → Token」记录在类的元数据上，优先级高于类型元数据
3. 解析时：查缓存 → 查 Provider → 按类型创建 → 递归解析依赖 → 缓存
4. 用一个「正在解析」集合检测循环依赖，报错时输出依赖链

**参考实现**

```ts
import 'reflect-metadata'

type Ctor<T = unknown> = new (...args: any[]) => T
type Token<T = unknown> = Ctor<T> | string | symbol
type Provider =
  | Ctor
  | { provide: Token; useClass: Ctor }
  | { provide: Token; useValue: unknown }
  | { provide: Token; useFactory: (...deps: any[]) => unknown; inject?: Token[] }

const INJECT_TOKENS = Symbol('inject:tokens')
const nameOf = (t: Token) => (typeof t === 'function' ? t.name : String(t))

/** 标记类可被容器管理；装饰器存在时 TS 才会为构造函数输出 design:paramtypes */
export function Injectable(): ClassDecorator {
  return () => {}
}

/** 显式指定构造函数某个参数的注入 Token，用于接口、字符串或 Symbol 依赖 */
export function Inject(token: Token): ParameterDecorator {
  return (target, _key, index) => {
    const tokens: Map<number, Token> = Reflect.getOwnMetadata(INJECT_TOKENS, target) ?? new Map()
    tokens.set(index, token)
    Reflect.defineMetadata(INJECT_TOKENS, tokens, target)
  }
}

/** 单例 IoC 容器：按 Token 解析 Provider 并递归注入依赖 */
export class Container {
  #providers = new Map<Token, Provider>()
  #instances = new Map<Token, unknown>()
  #resolving: Token[] = []

  register(...providers: Provider[]): this {
    for (const p of providers) this.#providers.set(typeof p === 'function' ? p : p.provide, p)
    return this
  }

  get<T>(token: Token<T>): T {
    if (this.#instances.has(token)) return this.#instances.get(token) as T

    const provider = this.#providers.get(token)
    if (!provider) throw new Error(`No provider for ${nameOf(token)}`)

    if (this.#resolving.includes(token)) {
      const chain = [...this.#resolving, token].map(nameOf).join(' -> ')
      throw new Error(`Circular dependency: ${chain}`)
    }

    this.#resolving.push(token)
    try {
      const instance = this.#create(provider)
      this.#instances.set(token, instance)
      return instance as T
    } finally {
      this.#resolving.pop()
    }
  }

  #create(p: Provider): unknown {
    if (typeof p === 'function') return this.#construct(p)
    if ('useValue' in p) return p.useValue
    if ('useClass' in p) return this.#construct(p.useClass)
    return p.useFactory(...(p.inject ?? []).map(t => this.get(t)))
  }

  #construct(cls: Ctor): unknown {
    const types: Array<Ctor | undefined> = Reflect.getMetadata('design:paramtypes', cls) ?? []
    const explicit: Map<number, Token> = Reflect.getOwnMetadata(INJECT_TOKENS, cls) ?? new Map()

    const deps = types.map((type, i) => {
      const token = explicit.get(i) ?? type
      // interface、联合类型在元数据中会变成 Object；循环 import 时类型为 undefined
      if (!token || token === Object) {
        throw new Error(`Cannot resolve argument at index [${i}] of ${cls.name}`)
      }
      return this.get(token)
    })
    return new cls(...deps)
  }
}

// 使用
const CONFIG = Symbol('CONFIG')

@Injectable()
class Db {
  constructor(@Inject(CONFIG) readonly cfg: { url: string }) {}
}

@Injectable()
class UserService {
  constructor(readonly db: Db) {}
}

const c = new Container().register(UserService, Db, { provide: CONFIG, useValue: { url: 'pg://' } })
c.get(UserService).db.cfg.url   // 'pg://'
```

**易错点**

- 忘记开启 `emitDecoratorMetadata`，或没有任何类装饰器，`design:paramtypes` 不存在
- 构造函数参数类型为 interface 时元数据是 `Object`，必须使用 `@Inject(token)`
- 参数装饰器作用于构造函数时，`target` 是**类本身**而不是原型
- 和 Nest 的差异：没有模块可见性（imports/exports）、作用域、异步工厂和 forwardRef

---

### H2. 实现基于元数据的认证与角色守卫

- **难度**: mid
- **标签**: 守卫, Reflector, 高频
- **面试官看什么**: 默认安全的设计、Reflector 的用法、401 与 403 的区分

**题干**

实现全局 `JwtAuthGuard`（默认所有接口需要登录，`@Public()` 豁免）和 `RolesGuard`（`@Roles('admin')`，方法级覆盖类级）。

**思路口述**

1. 两个装饰器写元数据：`@Public()` 用 `SetMetadata`，`@Roles()` 用类型安全的 `Reflector.createDecorator`
2. 认证守卫先检查 Public，再校验 token，并把用户挂到 `req.user`
3. 角色守卫读取角色元数据，没有声明就放行
4. 用 `APP_GUARD` 注册，按顺序执行：先认证后授权

**参考实现**

```ts
export type Role = 'admin' | 'editor' | 'user'
export interface AuthUser { id: string; roles: Role[] }

export const IS_PUBLIC = 'auth:isPublic'
/** 标记接口无需登录 */
export const Public = () => SetMetadata(IS_PUBLIC, true)
/** 声明访问接口所需的角色，满足任一即可 */
export const Roles = Reflector.createDecorator<Role[]>()

/** 全局认证守卫：默认所有接口需要登录 */
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private readonly reflector: Reflector, private readonly jwt: JwtService) {}

  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC, [ctx.getHandler(), ctx.getClass()])
    if (isPublic) return true

    const req = ctx.switchToHttp().getRequest<Request & { user?: AuthUser }>()
    const [type, token] = req.headers.authorization?.split(' ') ?? []
    if (type !== 'Bearer' || !token) throw new UnauthorizedException('missing token')

    try {
      const payload = await this.jwt.verifyAsync<{ sub: string; roles: Role[] }>(token)
      req.user = { id: payload.sub, roles: payload.roles }
      return true
    } catch {
      throw new UnauthorizedException('invalid token')
    }
  }
}

/** 角色守卫：方法级声明覆盖类级声明 */
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(ctx: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride(Roles, [ctx.getHandler(), ctx.getClass()])
    if (!required?.length) return true

    const user = ctx.switchToHttp().getRequest<{ user?: AuthUser }>().user
    // Public 接口上误加了 Roles 时 user 不存在，此时应拒绝而不是报错
    if (!user) throw new UnauthorizedException()
    if (!required.some(r => user.roles.includes(r))) throw new ForbiddenException('insufficient role')
    return true
  }
}

@Module({
  providers: [
    { provide: APP_GUARD, useClass: JwtAuthGuard },   // 注册顺序即执行顺序
    { provide: APP_GUARD, useClass: RolesGuard },
  ],
})
export class AuthModule {}

@Roles(['editor'])
@Controller('articles')
export class ArticlesController {
  @Public() @Get() list() {}
  @Roles(['admin']) @Delete(':id') remove() {}   // 覆盖类级的 editor
}
```

**易错点**

- 未登录应返回 401，已登录但无权限返回 403；守卫返回 `false` 默认得到 403
- `getAllAndMerge` 与 `getAllAndOverride` 混淆，导致方法级没有覆盖类级
- 非 HTTP 上下文（GraphQL、微服务）中 `switchToHttp()` 拿不到请求，需要按 `getType()` 分支处理

---

### H3. 实现统一响应包装与访问日志拦截器

- **难度**: mid
- **标签**: 拦截器, RxJS
- **面试官看什么**: RxJS 操作符、例外情况处理（文件流、跳过包装）、成功和失败都记录日志

**题干**

实现拦截器：成功响应包装为 `{ code: 0, data, requestId }`；文件流和标记了 `@RawResponse()` 的接口不包装；记录每个请求的方法、路径、状态码和耗时（失败也要记录）。

**思路口述**

1. 进入阶段记录开始时间和请求 ID
2. `map` 包装响应，遇到 `StreamableFile` 或元数据标记时跳过
3. 用 `finalize` 记录日志，保证成功和失败都会执行；状态码从响应对象读取
4. 失败时状态码由过滤器设置，在 `finalize` 中可能还未设置，所以失败的日志在 `catchError` 中根据异常计算

**参考实现**

```ts
export const RAW_RESPONSE = 'response:raw'
/** 标记接口返回原始数据，不做统一包装 */
export const RawResponse = () => SetMetadata(RAW_RESPONSE, true)

/** 统一包装成功响应，并为每个请求输出一条访问日志 */
@Injectable()
export class ResponseInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP')

  constructor(private readonly reflector: Reflector) {}

  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    if (ctx.getType() !== 'http') return next.handle()

    const req = ctx.switchToHttp().getRequest<Request>()
    const res = ctx.switchToHttp().getResponse<Response>()
    const requestId = (req.headers['x-request-id'] as string) ?? randomUUID()
    const start = performance.now()
    const raw = this.reflector.getAllAndOverride<boolean>(RAW_RESPONSE, [ctx.getHandler(), ctx.getClass()])

    const log = (status: number, err?: unknown) =>
      this.logger.log({
        requestId,
        method: req.method,
        path: req.originalUrl,
        status,
        costMs: Math.round(performance.now() - start),
        error: err instanceof Error ? err.message : undefined,
      })

    return next.handle().pipe(
      map(data => (raw || data instanceof StreamableFile ? data : { code: 0, data, requestId })),
      tap(() => log(res.statusCode)),
      catchError(err => {
        log(err instanceof HttpException ? err.getStatus() : 500, err)
        return throwError(() => err)
      }),
    )
  }
}
```

**易错点**

- 用 `tap` 只能记录成功，失败的请求漏记
- 在 `catchError` 中返回 `of(...)` 会吞掉异常，过滤器不再执行
- 访问日志放在拦截器中不会记录被守卫拒绝的请求和 404；需要完整访问日志时放在中间件中（监听 `res.on('finish')`）
- SSE 接口返回的是事件流，包装后会破坏事件格式，应跳过

---

### H4. 实现统一的全局异常过滤器

- **难度**: mid
- **标签**: 过滤器, 错误处理
- **面试官看什么**: 区分异常类型、处理 ValidationPipe 的错误数组、不泄露内部信息、兼容平台

**题干**

实现 `AllExceptionsFilter`：输出统一结构 `{ code, message, details?, requestId, path }`；`HttpException` 使用其状态码；领域错误按映射表转换；未知异常返回 500 并记录堆栈，但不把堆栈返回给客户端。

**思路口述**

1. `@Catch()` 不带参数捕获所有异常
2. 按类型分支：HttpException → 领域错误 → 未知错误
3. `HttpException.getResponse()` 可能是字符串或对象，ValidationPipe 的 message 是数组
4. 通过 `HttpAdapterHost` 写响应，兼容 Express 和 Fastify

**参考实现**

```ts
/** 与协议无关的业务错误，由过滤器映射为 HTTP 状态码 */
export class DomainError extends Error {
  constructor(readonly code: string, message: string, readonly status = 422) {
    super(message)
  }
}

interface ErrorBody {
  code: string | number
  message: string
  details?: unknown
  requestId?: string
  path: string
}

/** 全局兜底异常过滤器：统一错误响应结构 */
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name)

  constructor(private readonly adapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // 只处理 HTTP；微服务和 WebSocket 由各自的过滤器处理
    if (host.getType() !== 'http') throw exception

    const { httpAdapter } = this.adapterHost
    const ctx = host.switchToHttp()
    const req = ctx.getRequest()
    const base = {
      requestId: req.headers?.['x-request-id'] as string | undefined,
      path: httpAdapter.getRequestUrl(req) as string,
    }

    const [status, body] = this.toResponse(exception, base)
    if (status >= 500) {
      this.logger.error({ ...base, err: exception }, exception instanceof Error ? exception.stack : undefined)
    }
    httpAdapter.reply(ctx.getResponse(), body, status)
  }

  private toResponse(e: unknown, base: Pick<ErrorBody, 'requestId' | 'path'>): [number, ErrorBody] {
    if (e instanceof HttpException) {
      const status = e.getStatus()
      const res = e.getResponse()
      const message = typeof res === 'string' ? res : (res as { message?: string | string[] }).message
      return [
        status,
        {
          ...base,
          code: status,
          // ValidationPipe 的 message 是字符串数组
          message: Array.isArray(message) ? 'Validation failed' : (message ?? e.message),
          details: Array.isArray(message) ? message : undefined,
        },
      ]
    }
    if (e instanceof DomainError) {
      return [e.status, { ...base, code: e.code, message: e.message }]
    }
    return [500, { ...base, code: 'INTERNAL_ERROR', message: 'Internal server error' }]
  }
}

// 注册：通过 APP_FILTER 使其可以注入 HttpAdapterHost
providers: [{ provide: APP_FILTER, useClass: AllExceptionsFilter }]
```

**易错点**

- 把未知异常的 `message` 或堆栈直接返回给客户端，泄露内部信息（如 SQL 语句）
- 直接调用 `res.status().json()`，切换到 Fastify 后失效
- 过滤器内部再抛异常会由 Nest 兜底处理为 500，过滤器本身要尽量简单可靠
- 4xx 不应打 error 级别日志，否则告警噪音过大

---

### H5. 用 ConfigurableModuleBuilder 实现 Redis 动态模块

- **难度**: mid
- **标签**: 动态模块
- **面试官看什么**: forRoot / forRootAsync 的实现方式、资源释放、全局选项

**题干**

实现 `RedisModule.forRoot(opts)` 和 `RedisModule.forRootAsync({ inject, useFactory })`，支持 `isGlobal`；应用关闭时断开连接。

**思路口述**

1. `ConfigurableModuleBuilder` 自动生成同步和异步的注册方法，以及选项 Token 和类型
2. 用 `setClassMethodName('forRoot')` 把默认的 `register` 改为 `forRoot`
3. 用 `setExtras` 增加 `isGlobal` 选项并映射到模块定义
4. 客户端 Provider 依赖选项 Token；模块类实现 `OnApplicationShutdown` 关闭连接

**参考实现**

```ts
export interface RedisModuleOptions {
  url: string
  keyPrefix?: string
}

export const {
  ConfigurableModuleClass,
  MODULE_OPTIONS_TOKEN,
} = new ConfigurableModuleBuilder<RedisModuleOptions>()
  .setClassMethodName('forRoot')
  .setExtras({ isGlobal: false }, (definition, extras) => ({ ...definition, global: extras.isGlobal }))
  .build()

export const REDIS = Symbol('REDIS')

/** 提供共享的 Redis 客户端，并在应用关闭时断开连接 */
@Module({
  providers: [
    {
      provide: REDIS,
      inject: [MODULE_OPTIONS_TOKEN],
      useFactory: (opts: RedisModuleOptions) => new Redis(opts.url, { keyPrefix: opts.keyPrefix }),
    },
  ],
  exports: [REDIS],
})
export class RedisModule extends ConfigurableModuleClass implements OnApplicationShutdown {
  constructor(@Inject(REDIS) private readonly redis: Redis) {
    super()
  }

  async onApplicationShutdown(): Promise<void> {
    await this.redis.quit()
  }
}

// 使用
@Module({
  imports: [
    RedisModule.forRootAsync({
      isGlobal: true,
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (cfg: ConfigService) => ({ url: cfg.getOrThrow('REDIS_URL'), keyPrefix: 'shop:' }),
    }),
  ],
})
export class AppModule {}
```

**易错点**

- 手写 `forRootAsync` 时忘记把 `options.imports` 合并到模块的 imports 中，导致 `ConfigService` 无法注入
- 只在 `onModuleDestroy` 中关闭连接，但没有调用 `app.enableShutdownHooks()`，关闭钩子不会被触发
- 用 `quit()`（等待命令执行完）而不是 `disconnect()`（立即断开），避免丢失正在执行的命令

---

### H6. 实现基于 Zod / Standard Schema 的校验管道

- **难度**: mid
- **标签**: 管道, 校验
- **面试官看什么**: 管道接口、错误格式、对 Standard Schema 的理解

**题干**

1. 实现 `ZodValidationPipe(schema)`（适用于 Nest 12 之前的版本）
2. 实现一个通用的 `SchemaPipe`，支持任何实现 Standard Schema 的库

**思路口述**

1. 管道接收原始值，校验失败抛 `BadRequestException`，成功返回解析后的值（可能被转换过）
2. 错误转换为 `{ path, message }` 列表，便于前端定位字段
3. Standard Schema 的统一入口是 `schema['~standard'].validate(value)`，返回值或 Promise，结果是 `{ value }` 或 `{ issues }`

**参考实现**

```ts
import type { StandardSchemaV1 } from '@standard-schema/spec'

/** 用 Zod schema 校验并转换参数 */
@Injectable()
export class ZodValidationPipe<T> implements PipeTransform<unknown, T> {
  constructor(private readonly schema: ZodType<T>) {}

  transform(value: unknown): T {
    const result = this.schema.safeParse(value)
    if (!result.success) {
      throw new BadRequestException({
        code: 'VALIDATION_FAILED',
        errors: result.error.issues.map(i => ({ path: i.path.join('.'), message: i.message })),
      })
    }
    return result.data
  }
}

/** 通用校验管道：接受 Zod、Valibot、ArkType 等任意 Standard Schema 实现 */
@Injectable()
export class SchemaPipe<S extends StandardSchemaV1> implements PipeTransform<unknown, Promise<StandardSchemaV1.InferOutput<S>>> {
  constructor(private readonly schema: S) {}

  async transform(value: unknown): Promise<StandardSchemaV1.InferOutput<S>> {
    const result = await this.schema['~standard'].validate(value)
    if (result.issues) {
      throw new BadRequestException({
        code: 'VALIDATION_FAILED',
        errors: result.issues.map(i => ({
          path: i.path?.map(p => (typeof p === 'object' ? p.key : p)).join('.') ?? '',
          message: i.message,
        })),
      })
    }
    return result.value
  }
}

const createUserSchema = z.object({ email: z.string().email(), age: z.coerce.number().int().min(0) })

@Post()
create(@Body(new SchemaPipe(createUserSchema)) body: z.infer<typeof createUserSchema>) {}
```

Nest 12 中可以直接使用内置方案：`@Body({ schema })` + 全局 `StandardSchemaValidationPipe`。

**易错点**

- 返回原始的 `value` 而不是解析后的值，导致 `z.coerce` 等转换不生效
- Standard Schema 的 `validate` 可能返回 Promise，要统一 `await`
- `path` 中的元素可能是 `{ key }` 对象，需要转换
- 管道实例通过 `new` 创建时无法注入依赖，需要依赖时把管道类交给容器创建

---

### H7. 实现基于 Redis 的幂等拦截器

- **难度**: senior
- **标签**: 幂等, 拦截器, Redis
- **面试官看什么**: 并发下的正确性、请求体校验、处理中状态、失败后的处理

**题干**

实现 `@Idempotent()`：要求请求头带 `Idempotency-Key`；同一用户、同一 key 的重复请求直接返回首次结果；首次请求处理中时返回 409；相同 key 但请求体不同时返回 422。

**思路口述**

1. 用 `SET key value NX EX ttl` 原子占位，只有一个请求能拿到执行权
2. 占位记录中保存请求体哈希，用于识别 key 被误用
3. 处理成功后把状态码和响应体写入记录（等写完再返回响应，避免竞态）
4. 处理失败时删除记录，让客户端可以用同一个 key 重试
5. key 加上用户维度，防止不同用户的 key 冲突

**参考实现**

```ts
interface IdemRecord {
  state: 'PROCESSING' | 'DONE'
  bodyHash: string
  status?: number
  body?: unknown
}

const IDEMPOTENT_TTL = 'idempotent:ttl'

/** 声明接口需要幂等保护，ttl 为结果保留时间（秒） */
export const Idempotent = (ttlSeconds = 24 * 3600) =>
  applyDecorators(SetMetadata(IDEMPOTENT_TTL, ttlSeconds), UseInterceptors(IdempotencyInterceptor))

/** 基于 Idempotency-Key 的幂等拦截器 */
@Injectable()
export class IdempotencyInterceptor implements NestInterceptor {
  constructor(@Inject(REDIS) private readonly redis: Redis, private readonly reflector: Reflector) {}

  async intercept(ctx: ExecutionContext, next: CallHandler): Promise<Observable<unknown>> {
    const ttl = this.reflector.get<number>(IDEMPOTENT_TTL, ctx.getHandler())
    const req = ctx.switchToHttp().getRequest<Request & { user?: { id: string } }>()
    const res = ctx.switchToHttp().getResponse<Response>()

    const idemKey = req.header('idempotency-key')
    if (!idemKey) throw new BadRequestException('Idempotency-Key header is required')

    const key = `idem:${req.user?.id ?? 'anonymous'}:${idemKey}`
    const bodyHash = createHash('sha256').update(JSON.stringify(req.body ?? null)).digest('hex')
    const placeholder: IdemRecord = { state: 'PROCESSING', bodyHash }

    const acquired = await this.redis.set(key, JSON.stringify(placeholder), 'EX', ttl, 'NX')
    if (!acquired) return this.replay(key, bodyHash, res)

    return next.handle().pipe(
      // 先保存结果再返回响应：否则客户端立即重试时可能读到 PROCESSING
      mergeMap(async body => {
        const done: IdemRecord = { state: 'DONE', bodyHash, status: res.statusCode, body }
        await this.redis.set(key, JSON.stringify(done), 'EX', ttl)
        return body
      }),
      catchError(err => from(this.redis.del(key)).pipe(mergeMap(() => throwError(() => err)))),
    )
  }

  private async replay(key: string, bodyHash: string, res: Response): Promise<Observable<unknown>> {
    const raw = await this.redis.get(key)
    // 占位记录恰好过期：让客户端稍后重试，而不是在这里再次执行
    if (!raw) throw new ConflictException('please retry')

    const record = JSON.parse(raw) as IdemRecord
    if (record.bodyHash !== bodyHash) {
      throw new UnprocessableEntityException('Idempotency-Key reused with a different request body')
    }
    if (record.state === 'PROCESSING') throw new ConflictException('request is being processed')

    res.status(record.status ?? 200)
    return of(record.body)
  }
}

@Idempotent()
@Post('orders')
createOrder(@Body() dto: CreateOrderDto) {}
```

**易错点**

- 先 `GET` 判断再 `SET`，两步之间存在竞态；必须用 `NX` 原子占位
- 用 `tap` 异步保存结果，响应先于保存完成，导致重试读到 PROCESSING
- `JSON.stringify` 对键顺序敏感，客户端改变字段顺序会被判为不同请求体；严格场景使用规范化的序列化
- 业务失败（如余额不足）是否保存结果需要按业务决定；这里统一删除，允许客户端重试
- Redis 只是第一道防线，数据库唯一约束和状态机条件更新仍然需要

---

### H8. 用 AsyncLocalStorage 实现 `@Transactional()`

- **难度**: senior
- **标签**: 事务, AsyncLocalStorage
- **面试官看什么**: 事务上下文传播、嵌套事务的处理、装饰器替换方法时保留元数据

**题干**

基于 Prisma 实现事务上下文：`@Transactional()` 标记的方法在事务中执行；方法内部调用的其他 Service 自动加入同一事务；已经在事务中时不开启新事务。

**思路口述**

1. `TxHost` 内部持有 `AsyncLocalStorage<TransactionClient>`
2. `run(fn)`：已有事务就直接执行（加入，相当于 REQUIRED 传播）；否则开启 `$transaction` 并在 `als.run` 中执行
3. Repository 统一通过 `txHost.client` 访问数据库：有事务用事务客户端，没有则用普通客户端
4. 方法装饰器无法注入依赖，约定所在类有一个 `txHost` 属性
5. 替换 `descriptor.value` 后要复制原方法上的元数据，否则同一方法上的路由、事件监听等装饰器会失效

**参考实现**

```ts
type TxClient = Prisma.TransactionClient

/** 管理当前异步调用链中的 Prisma 事务客户端 */
@Injectable()
export class TxHost {
  readonly #als = new AsyncLocalStorage<TxClient>()

  constructor(private readonly prisma: PrismaService) {}

  /** 当前事务中的客户端；不在事务中时返回普通客户端 */
  get client(): TxClient {
    return this.#als.getStore() ?? this.prisma
  }

  run<T>(fn: () => Promise<T>): Promise<T> {
    // 已在事务中则加入当前事务，不嵌套开启新事务
    if (this.#als.getStore()) return fn()
    return this.prisma.$transaction(tx => this.#als.run(tx, fn), { timeout: 10_000 })
  }
}

/** 让方法在事务中执行；所在类必须注入名为 txHost 的 TxHost */
export function Transactional(): MethodDecorator {
  return (_target, _key, descriptor: PropertyDescriptor) => {
    const original = descriptor.value as (...args: unknown[]) => Promise<unknown>

    const wrapped = function (this: { txHost: TxHost }, ...args: unknown[]) {
      return this.txHost.run(() => original.apply(this, args))
    }
    // 复制原方法上的元数据：替换函数后，其他装饰器写在原函数上的元数据会丢失
    for (const key of Reflect.getMetadataKeys(original)) {
      Reflect.defineMetadata(key, Reflect.getMetadata(key, original), wrapped)
    }
    descriptor.value = wrapped
  }
}

@Injectable()
export class InventoryRepository {
  constructor(private readonly txHost: TxHost) {}

  async deduct(productId: string, qty: number): Promise<void> {
    const { count } = await this.txHost.client.product.updateMany({
      where: { id: productId, stock: { gte: qty } },     // 条件更新防止超卖
      data: { stock: { decrement: qty } },
    })
    if (count === 0) throw new DomainError('INSUFFICIENT_STOCK', '库存不足', 409)
  }
}

@Injectable()
export class OrdersService {
  constructor(readonly txHost: TxHost, private readonly inventory: InventoryRepository) {}

  @Transactional()
  async placeOrder(userId: string, productId: string, qty: number) {
    await this.inventory.deduct(productId, qty)                        // 自动加入同一事务
    return this.txHost.client.order.create({ data: { userId, productId, qty } })
  }
}
```

**易错点**

- 装饰器替换方法后没有复制元数据，同一方法上的 `@Get()`、`@OnEvent()` 等失效（装饰器执行顺序不同，结果也不同）
- 事务中执行 HTTP 调用或发送消息，导致长事务和无法回滚的副作用
- Prisma 交互式事务有默认超时，长事务会被中断
- 并发的 `Promise.all` 在同一个交互式事务中执行多条语句，部分数据库驱动不支持在同一连接上并发
- 生产中可直接使用 `@nestjs-cls/transactional`，它支持多种 ORM 和传播行为

---

### H9. 实现分布式锁，保证定时任务在多实例下只执行一次

- **难度**: mid
- **标签**: 分布式锁, 定时任务
- **面试官看什么**: 原子加锁、只释放自己的锁、锁过期的处理

**题干**

实现 `LockService.runExclusive(key, ttlMs, fn)`：拿到锁才执行 `fn`，拿不到直接跳过；执行结束后安全释放锁。并在 `@Cron` 任务中使用。

**思路口述**

1. 加锁：`SET lock:key token PX ttl NX`，token 为随机值，标识持有者
2. 释放：用 Lua 脚本比较 token 后再删除，保证原子性
3. 为什么要比较：锁可能已经过期并被其他实例获取，直接 `DEL` 会删掉别人的锁
4. 拿不到锁时返回 `undefined`，由调用方决定是跳过还是等待

**参考实现**

```ts
const RELEASE_SCRIPT = `
if redis.call('get', KEYS[1]) == ARGV[1] then
  return redis.call('del', KEYS[1])
end
return 0`

/** 基于 Redis 的简单分布式互斥锁 */
@Injectable()
export class LockService {
  private readonly logger = new Logger(LockService.name)

  constructor(@Inject(REDIS) private readonly redis: Redis) {}

  /** 获取锁后执行 fn；未获取到锁时返回 undefined，表示其他实例正在执行 */
  async runExclusive<T>(key: string, ttlMs: number, fn: () => Promise<T>): Promise<T | undefined> {
    const lockKey = `lock:${key}`
    const token = randomUUID()

    const acquired = await this.redis.set(lockKey, token, 'PX', ttlMs, 'NX')
    if (!acquired) return undefined

    const start = Date.now()
    try {
      return await fn()
    } finally {
      const released = await this.redis.eval(RELEASE_SCRIPT, 1, lockKey, token)
      if (released === 0) {
        // 任务执行时间超过了锁的有效期，期间可能有其他实例重复执行
        this.logger.warn(`lock ${key} expired before release, cost ${Date.now() - start}ms`)
      }
    }
  }
}

@Injectable()
export class OrderSyncTask {
  constructor(private readonly lock: LockService, private readonly sync: OrderSyncService) {}

  @Cron('0 */5 * * * *')
  async run() {
    await this.lock.runExclusive('order-sync', 4 * 60_000, () => this.sync.syncPending())
  }
}
```

**易错点**

- 先 `SETNX` 再 `EXPIRE` 两条命令，中间进程崩溃会导致锁永不过期
- 直接 `DEL` 释放，可能删掉其他实例的锁
- 锁的 TTL 小于任务执行时间，导致重复执行；需要续期（看门狗）或让任务本身幂等
- Redis 主从切换时锁可能丢失；对正确性要求极高时使用数据库唯一约束或带 fencing token 的方案

---

### H10. 实现多租户上下文与自动带租户条件的仓储

- **难度**: senior
- **标签**: 多租户, CLS
- **面试官看什么**: 租户来源是否可信、上下文传递、防止漏写租户条件

**题干**

实现：认证后从 JWT 中解析租户 ID 并写入请求上下文；提供 `TenantRepository` 基类，所有查询和写入自动带上 `tenantId`，缺少租户上下文时拒绝执行。

**思路口述**

1. 租户 ID 来自**已验证的 JWT**，不能直接信任请求头（否则用户可以伪造访问其他租户）
2. 使用 `nestjs-cls` 保存上下文，Provider 保持单例
3. 基类封装常用方法，统一注入租户条件；子类不直接访问底层仓储
4. 缺少租户上下文时直接抛错（快速失败），而不是查询全部数据

**参考实现**

```ts
/** 在认证守卫之后执行：把已验证的租户 ID 写入请求上下文 */
@Injectable()
export class TenantGuard implements CanActivate {
  constructor(private readonly cls: ClsService) {}

  canActivate(ctx: ExecutionContext): boolean {
    const user = ctx.switchToHttp().getRequest<{ user?: { id: string; tenantId: string } }>().user
    if (!user?.tenantId) throw new ForbiddenException('tenant not resolved')
    this.cls.set('tenantId', user.tenantId)
    return true
  }
}

/** 所有操作自动限定在当前租户内的仓储基类 */
export abstract class TenantRepository<T extends { tenantId: string }> {
  constructor(
    protected readonly repo: Repository<T>,
    protected readonly cls: ClsService,
  ) {}

  protected get tenantId(): string {
    const id = this.cls.get<string | undefined>('tenantId')
    // 缺少租户上下文时拒绝执行，避免退化为查询所有租户的数据
    if (!id) throw new InternalServerErrorException('tenant context missing')
    return id
  }

  find(where: FindOptionsWhere<T> = {}): Promise<T[]> {
    return this.repo.find({ where: { ...where, tenantId: this.tenantId } as FindOptionsWhere<T> })
  }

  findOne(where: FindOptionsWhere<T>): Promise<T | null> {
    return this.repo.findOne({ where: { ...where, tenantId: this.tenantId } as FindOptionsWhere<T> })
  }

  save(entity: DeepPartial<T>): Promise<T> {
    // 覆盖调用方传入的 tenantId，防止跨租户写入
    return this.repo.save({ ...entity, tenantId: this.tenantId } as DeepPartial<T>)
  }
}

@Injectable()
export class ProjectRepository extends TenantRepository<Project> {
  constructor(@InjectRepository(Project) repo: Repository<Project>, cls: ClsService) {
    super(repo, cls)
  }
}
```

**易错点**

- 从请求头读取租户 ID 而不与用户身份绑定，导致越权
- `where` 为数组（OR 条件）时，简单展开无法给每个分支加条件；复杂查询要在 QueryBuilder 中统一处理
- 原生 SQL、批量更新、报表查询绕过了基类；需要数据库行级安全（RLS）兜底
- 队列消费者、定时任务中没有 HTTP 上下文，需要从消息中恢复租户并调用 `cls.run()`

---

### H11. 手写一个迷你请求处理管线

- **难度**: senior
- **标签**: 原理, 生命周期
- **面试官看什么**: 是否真正理解守卫、拦截器、管道、过滤器的执行顺序和异常流转

**题干**

不依赖 Nest，实现 `createRouteHandler`：按 Nest 的顺序执行守卫 → 拦截器（洋葱模型）→ 管道 → 处理器；任一环节抛异常时，按顺序查找第一个匹配的过滤器处理，没有匹配的则返回 500。

**思路口述**

1. 守卫依次执行，任一返回 false 就抛 Forbidden
2. 拦截器用 `reduceRight` 从内向外包裹：第一个拦截器在最外层，所以它的「前半段」最先执行、「后半段」最后执行
3. 最内层是「执行管道 + 调用处理器」：每个参数依次经过所有管道
4. 外层 `try/catch`，异常按「路由 → 控制器 → 全局」排好序的过滤器列表查找第一个匹配的

**参考实现**

```ts
interface Ctx {
  req: { params: Record<string, string>; body: unknown; headers: Record<string, string> }
}
type Guard = (ctx: Ctx) => boolean | Promise<boolean>
type Next = () => Promise<unknown>
type Interceptor = (ctx: Ctx, next: Next) => Promise<unknown>
type Pipe = (value: unknown, meta: { index: number }) => unknown
interface Filter {
  catches: (err: unknown) => boolean
  handle: (err: unknown, ctx: Ctx) => { status: number; body: unknown }
}

class ForbiddenError extends Error {
  readonly status = 403
}

/** 按 Nest 的生命周期顺序组合守卫、拦截器、管道、过滤器和处理器 */
export function createRouteHandler(opts: {
  guards: Guard[]
  interceptors: Interceptor[]       // 顺序：全局 → 控制器 → 路由
  pipes: Pipe[]
  filters: Filter[]                 // 顺序：路由 → 控制器 → 全局
  extractArgs: (ctx: Ctx) => unknown[]
  handler: (...args: unknown[]) => unknown
}) {
  return async (req: Ctx['req']): Promise<{ status: number; body: unknown }> => {
    const ctx: Ctx = { req }
    try {
      for (const guard of opts.guards) {
        if (!(await guard(ctx))) throw new ForbiddenError('Forbidden resource')
      }

      const invokeHandler: Next = async () => {
        const args: unknown[] = []
        for (const [index, raw] of opts.extractArgs(ctx).entries()) {
          let value = raw
          for (const pipe of opts.pipes) value = await pipe(value, { index })
          args.push(value)
        }
        return opts.handler(...args)
      }

      // reduceRight 让数组中的第一个拦截器位于最外层
      const run = opts.interceptors.reduceRight<Next>(
        (next, interceptor) => () => interceptor(ctx, next),
        invokeHandler,
      )
      return { status: 200, body: await run() }
    } catch (err) {
      const filter = opts.filters.find(f => f.catches(err))
      if (filter) return filter.handle(err, ctx)
      return { status: 500, body: { message: 'Internal server error' } }
    }
  }
}

// 验证执行顺序
const order: string[] = []
const handle = createRouteHandler({
  guards: [() => (order.push('guard'), true)],
  interceptors: [
    async (_c, next) => { order.push('global:before'); const r = await next(); order.push('global:after'); return r },
    async (_c, next) => { order.push('route:before'); const r = await next(); order.push('route:after'); return r },
  ],
  pipes: [v => (order.push('pipe'), Number(v))],
  filters: [],
  extractArgs: ctx => [ctx.req.params.id],
  handler: id => (order.push('handler'), { id }),
})
await handle({ params: { id: '1' }, body: null, headers: {} })
// order: guard, global:before, route:before, pipe, handler, route:after, global:after
```

**易错点**

- 用 `reduce` 而不是 `reduceRight`，拦截器嵌套顺序反了
- 管道放在拦截器外面执行，导致拦截器的 `catchError` 捕获不到管道异常
- 过滤器全部执行或从全局开始查找；Nest 只执行第一个匹配的，且路由级优先
- 真实的 Nest 中拦截器基于 Observable，这里用 Promise 简化，但嵌套语义相同

---

### H12. 为一个 Service 和接口编写单元测试与 e2e 测试

- **难度**: mid
- **标签**: 测试
- **面试官看什么**: 测试模块的构建、依赖替换、全局配置一致性、关键场景覆盖

**题干**

为 `OrdersService.cancel(orderId, userId)` 编写单元测试（订单不存在、不属于当前用户、状态不允许取消、成功取消），并为 `POST /orders/:id/cancel` 编写 e2e 测试（未登录 401、参数非法 400、成功 200）。

**思路口述**

1. 单元测试只测业务规则：用 Mock 替换仓储，断言抛出的错误和调用参数
2. e2e 测试导入 `AppModule`，替换外部依赖；全局管道、过滤器通过共用的 `setupApp` 函数配置，保证和生产一致
3. 鉴权测试不要整体跳过守卫，而是签发测试用的 token，这样 401 场景也能覆盖
4. 数据库使用 Testcontainers 启动的真实实例，每个用例前清理数据

**参考实现**

```ts
// orders.service.spec.ts
describe('OrdersService.cancel', () => {
  let service: OrdersService
  const repo = { findById: vi.fn(), updateStatus: vi.fn() }

  beforeEach(async () => {
    vi.resetAllMocks()
    const moduleRef = await Test.createTestingModule({
      providers: [OrdersService, { provide: OrderRepository, useValue: repo }],
    }).compile()
    service = moduleRef.get(OrdersService)
  })

  it('throws when order does not exist', async () => {
    repo.findById.mockResolvedValue(null)
    await expect(service.cancel('o1', 'u1')).rejects.toMatchObject({ code: 'ORDER_NOT_FOUND' })
  })

  it('rejects cancelling another user\'s order', async () => {
    repo.findById.mockResolvedValue({ id: 'o1', userId: 'u2', status: 'PENDING' })
    await expect(service.cancel('o1', 'u1')).rejects.toMatchObject({ code: 'ORDER_NOT_FOUND' })
  })

  it('rejects when order is already shipped', async () => {
    repo.findById.mockResolvedValue({ id: 'o1', userId: 'u1', status: 'SHIPPED' })
    await expect(service.cancel('o1', 'u1')).rejects.toMatchObject({ code: 'ORDER_STATE_INVALID' })
  })

  it('cancels a pending order', async () => {
    repo.findById.mockResolvedValue({ id: 'o1', userId: 'u1', status: 'PENDING' })
    await service.cancel('o1', 'u1')
    expect(repo.updateStatus).toHaveBeenCalledWith('o1', 'PENDING', 'CANCELLED')
  })
})

// orders.e2e-spec.ts
describe('POST /orders/:id/cancel', () => {
  let app: INestApplication
  let token: string

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({ imports: [AppModule] })
      .overrideProvider(PaymentClient).useValue({ refund: vi.fn() })
      .compile()
    app = moduleRef.createNestApplication()
    setupApp(app)                                   // 与 main.ts 共用全局配置
    await app.init()
    token = await app.get(JwtService).signAsync({ sub: 'u1', roles: ['user'] })
  })

  afterAll(() => app.close())

  it('returns 401 without token', () =>
    request(app.getHttpServer()).post('/orders/o1/cancel').expect(401))

  it('returns 400 for malformed id', () =>
    request(app.getHttpServer())
      .post('/orders/not-a-uuid/cancel')
      .set('Authorization', `Bearer ${token}`)
      .expect(400))

  it('cancels own pending order', async () => {
    const order = await seedOrder({ userId: 'u1', status: 'PENDING' })
    await request(app.getHttpServer())
      .post(`/orders/${order.id}/cancel`)
      .set('Authorization', `Bearer ${token}`)
      .expect(200)
  })
})
```

**易错点**

- 「订单不属于当前用户」返回 403 会泄露订单存在，常见做法是返回与「不存在」相同的错误
- e2e 中没有复用全局配置，测试通过但生产行为不同
- 用 `overrideGuard` 整体跳过鉴权后，无法覆盖 401 和越权场景
- 用例之间共享数据导致顺序依赖，并行执行时不稳定
