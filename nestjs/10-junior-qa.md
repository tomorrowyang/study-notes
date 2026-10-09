# NestJS 初级 Q&A（核心概念与常用写法）

> 目标：把模块、控制器、Provider、依赖注入、管道、守卫、拦截器、过滤器、中间件和常用官方模块讲清楚。初级题在高级面试中常作为开场，回答要准确，并能自然引向原理。

---

## 一、框架概览

### Q1. NestJS 是什么？解决了什么问题？

- **难度**: junior
- **标签**: 概览

**答案**

NestJS 是一个用 TypeScript 构建 Node.js 服务端应用的**框架**，借鉴 Angular 的设计：

- **模块化**：用 `@Module` 划分功能边界
- **依赖注入（DI）/ 控制反转（IoC）**：由容器创建和组装对象
- **装饰器驱动**：路由、参数、元数据都用装饰器声明
- **平台无关**：HTTP 层默认 Express，可切换为 Fastify；同一套代码也能写微服务、WebSocket、GraphQL、定时任务

解决的问题：Express / Koa 只提供路由和中间件，**大型项目的分层、依赖管理、横切关注点（鉴权、校验、日志、异常）没有统一约定**，每个团队都在重新发明架构。Nest 把这些约定内置到框架中。

代价：学习曲线较陡；装饰器和 DI 带来一定的「魔法感」；启动比轻量框架慢；依赖 TS 的旧版实验性装饰器。

**口述模板（30–60 秒）**

1. 一句话结论：Nest 是带模块化和依赖注入的 Node 服务端框架，底层可以用 Express 或 Fastify。
2. 关键机制：模块划分边界，IoC 容器管理依赖，装饰器声明路由和元数据。
3. 注意点：适合中大型团队项目；小工具用它会显得笨重。
4. 可追问时补充：守卫、管道、拦截器、过滤器把横切关注点从业务代码中拆出来。

**常见追问**

- 和 Express / Koa 的关系？→ Nest 不是替代，而是在它们之上的架构层；可以通过适配器拿到底层实例。
- 和 Midway、Egg 比呢？→ Egg 是约定式目录 + 插件，Midway 同样基于 IoC + 装饰器；Nest 生态和社区最大。

---

### Q2. 一个 Nest 应用由哪些核心部分组成？

- **难度**: junior
- **标签**: 架构

**答案**

```text
main.ts            NestFactory.create(AppModule) 启动应用
AppModule          根模块，组织整个依赖图
├─ Controller      处理请求，负责路由和参数，不写业务
├─ Provider        可被注入的类：Service、Repository、Helper、Factory
└─ imports         引入其他模块

横切组件：Middleware、Guard、Interceptor、Pipe、Exception Filter
```

```ts
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule)
  app.setGlobalPrefix('api')
  await app.listen(process.env.PORT ?? 3000)
}
bootstrap()
```

典型分层：Controller（协议层）→ Service（业务层）→ Repository（数据层）。

**口述模板（30–60 秒）**

1. 一句话结论：入口用 NestFactory 创建应用，模块组织 Controller 和 Provider。
2. 关键机制：Controller 处理请求，Provider 承载业务并被注入。
3. 注意点：Controller 只做参数提取和响应，不放业务逻辑。
4. 可追问时补充：横切逻辑通过守卫、管道、拦截器、过滤器挂到路由上。

**常见追问**

- `NestFactory.createApplicationContext` 是什么？→ 只创建 IoC 容器不启动 HTTP 服务，用于脚本、CLI 工具、定时任务进程。

---

### Q3. `@Module` 的四个属性分别是什么？

- **难度**: junior
- **标签**: 模块

**答案**

```ts
@Module({
  imports: [TypeOrmModule.forFeature([User]), AuthModule],   // 依赖的其他模块
  controllers: [UsersController],                            // 本模块的控制器
  providers: [UsersService],                                 // 本模块的 Provider
  exports: [UsersService],                                   // 对外暴露的 Provider
})
export class UsersModule {}
```

规则：

- 模块默认是**封装的**：Provider 只在本模块内可见，必须 `exports` 后，其他模块 `imports` 本模块才能注入
- 可以 `exports` 一个导入的模块（重新导出）
- Provider 在同一个模块作用域内是**单例**；同一个模块被多处导入时，只实例化一次

**口述模板（30–60 秒）**

1. 一句话结论：imports 引入依赖模块，controllers 和 providers 是本模块成员，exports 决定对外暴露什么。
2. 关键机制：模块默认封装，exports + imports 才能跨模块注入。
3. 注意点：报「无法解析依赖」通常是忘了 exports 或 imports。
4. 可追问时补充：模块被多次导入也只实例化一次，Provider 默认单例。

**常见追问**

- 把同一个 Service 放进两个模块的 `providers` 会怎样？→ 会创建两个实例，单例被打破，通常是 bug。应该放在一个模块中并导出。

---

### Q4. Controller 怎么写？常用的参数装饰器有哪些？

- **难度**: junior
- **标签**: 控制器

**答案**

```ts
@Controller('users')
export class UsersController {
  constructor(private readonly users: UsersService) {}

  @Get()
  list(@Query() query: ListUsersDto) { return this.users.list(query) }

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) { return this.users.findOne(id) }

  @Post()
  @HttpCode(201)
  create(@Body() dto: CreateUserDto, @Headers('x-request-id') reqId?: string) {
    return this.users.create(dto)
  }
}
```

常用装饰器：

| 装饰器 | 作用 |
|--------|------|
| `@Get/@Post/@Put/@Patch/@Delete` | 路由方法 |
| `@Param` / `@Query` / `@Body` / `@Headers` | 取请求数据 |
| `@Req` / `@Res` | 底层请求响应对象 |
| `@HttpCode` / `@Header` / `@Redirect` | 响应状态、头、重定向 |
| `@Ip` / `@HostParam` | IP、子域名参数 |

返回值会被自动序列化：对象和数组转 JSON，Promise 和 Observable 会等待结果。

注意：Nest 11 使用 Express 5，路由通配符必须命名，如 `@Get('files/*path')`。

**口述模板（30–60 秒）**

1. 一句话结论：Controller 用路由装饰器定义接口，用参数装饰器取数据。
2. 关键机制：返回值自动序列化，支持 Promise 和 Observable。
3. 注意点：Controller 不写业务；使用 `@Res` 会接管响应。
4. 可追问时补充：Nest 11 起路由语法遵循 Express 5 的规则。

**常见追问**

- 使用 `@Res()` 有什么副作用？→ 进入「库特定模式」，Nest 不再自动发送响应，拦截器对返回值的处理会失效；需要时用 `@Res({ passthrough: true })`。

---

### Q5. 什么是 Provider？`@Injectable()` 做了什么？

- **难度**: junior
- **标签**: Provider, DI

**答案**

Provider 是**可以被 Nest 容器创建并注入**的东西：Service、Repository、Factory、Helper，甚至一个配置对象。

```ts
@Injectable()
export class UsersService {
  constructor(private readonly repo: UsersRepository) {}
}
```

`@Injectable()` 的作用：

1. 标记这个类可以被容器管理
2. 配合 TS 的 `emitDecoratorMetadata`，让编译器为构造函数输出 `design:paramtypes` 元数据，容器据此知道要注入什么

如果一个类自己没有依赖，不加 `@Injectable()` 也能被注入；但只要有构造函数依赖，就必须有某个类装饰器触发元数据输出。规范上 Provider 一律加上。

**口述模板（30–60 秒）**

1. 一句话结论：Provider 是由容器创建和注入的对象，Service 是最常见的 Provider。
2. 关键机制：@Injectable 触发编译器输出构造函数参数类型的元数据。
3. 注意点：Provider 必须注册在模块的 providers 中。
4. 可追问时补充：除了类，还可以用值、工厂、别名注册 Provider。

**常见追问**

- 为什么 Nest 需要 `emitDecoratorMetadata`？→ JS 运行时没有类型信息，只有通过编译器输出的元数据才知道构造函数参数的类型。

---

### Q6. 什么是依赖注入？有什么好处？

- **难度**: junior
- **标签**: DI, IoC

**答案**

- **控制反转（IoC）**：对象不再自己 `new` 依赖，而是由容器创建并交给它
- **依赖注入（DI）**：IoC 的实现方式，依赖通过构造函数（或属性）传入

```ts
// 没有 DI：强耦合，难以替换和测试
class OrderService { private repo = new MysqlOrderRepo() }

// 有 DI：只依赖抽象，由容器决定具体实现
@Injectable()
class OrderService { constructor(private readonly repo: OrderRepo) {} }
```

好处：

1. **解耦**：依赖可以替换（MySQL → Mongo、真实实现 → Mock）
2. **可测试**：测试中用 `overrideProvider` 注入假实现
3. **生命周期统一管理**：单例、请求级实例由容器负责
4. **依赖关系显式**：构造函数即依赖清单

**口述模板（30–60 秒）**

1. 一句话结论：依赖注入是由容器创建依赖并传给使用者，而不是使用者自己 new。
2. 关键机制：容器根据 Token 找到 Provider，实例化后注入构造函数。
3. 注意点：依赖过多的构造函数说明类职责过多。
4. 可追问时补充：最大的收益在测试，可以轻松替换为 Mock。

**常见追问**

- 构造函数注入和属性注入的区别？→ 构造函数注入依赖显式、实例创建后即完整；属性注入（`@Inject()` 写在属性上）用于继承场景或解决部分循环依赖。

---

## 二、请求处理组件

### Q7. 中间件（Middleware）怎么用？和 Express 中间件有什么关系？

- **难度**: junior
- **标签**: 中间件

**答案**

Nest 的中间件**等同于 Express 中间件**，在路由处理之前执行，能访问 `req`、`res`、`next`。

```ts
@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    req.headers['x-request-id'] ??= crypto.randomUUID()
    next()
  }
}

@Module({})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(RequestIdMiddleware)
      .exclude({ path: 'health', method: RequestMethod.GET })
      .forRoutes('{*splat}')     // Express 5 语法：匹配所有路径（包括根路径）
  }
}

// 全局函数式中间件
app.use(helmet())
```

特点：

- **不知道**接下来要执行哪个处理器（拿不到 `ExecutionContext`），所以不适合做基于元数据的鉴权
- 可以使用 DI（类中间件）
- 适合：请求 ID、日志、CORS、Helmet、body 解析、会话

**口述模板（30–60 秒）**

1. 一句话结论：Nest 中间件就是 Express/Fastify 中间件，在路由匹配之后、守卫之前执行。
2. 关键机制：在模块的 configure 中按路由绑定，或用 app.use 全局注册。
3. 注意点：中间件不知道目标处理器，拿不到装饰器元数据。
4. 可追问时补充：需要元数据的鉴权应该用守卫。

**常见追问**

- 中间件里能抛 HttpException 吗？→ 能，会被异常层处理；但更推荐在守卫中做拦截判断。

---

### Q8. 管道（Pipe）是什么？有哪些内置管道？

- **难度**: junior
- **标签**: 管道

**答案**

管道在**处理器调用之前**处理参数，有两个用途：

1. **转换（Transformation）**：如字符串转数字
2. **校验（Validation）**：不合法就抛异常，处理器不会执行

内置管道：`ValidationPipe`、`ParseIntPipe`、`ParseFloatPipe`、`ParseBoolPipe`、`ParseArrayPipe`、`ParseUUIDPipe`、`ParseEnumPipe`、`ParseDatePipe`（v11）、`DefaultValuePipe`、`ParseFilePipe`；v12 新增 `StandardSchemaValidationPipe`。

```ts
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
) {}

// 自定义管道
@Injectable()
export class TrimPipe implements PipeTransform {
  transform(value: unknown, metadata: ArgumentMetadata) {
    return typeof value === 'string' ? value.trim() : value
  }
}
```

**口述模板（30–60 秒）**

1. 一句话结论：管道在处理器之前转换和校验参数。
2. 关键机制：实现 PipeTransform 的 transform 方法，可绑定在参数、方法、控制器或全局。
3. 注意点：管道抛出异常时处理器不会执行。
4. 可追问时补充：最常用的是全局 ValidationPipe 配合 DTO。

**常见追问**

- `ArgumentMetadata` 里有什么？→ `type`（body/query/param/custom）、`metatype`（参数的类型）、`data`（装饰器参数，如 `'id'`）；v12 还可能带 `schema`。

---

### Q9. DTO 和 `ValidationPipe` 怎么配合使用？

- **难度**: junior
- **标签**: 校验, DTO

**答案**

```ts
export class CreateUserDto {
  @IsEmail() email!: string
  @IsString() @Length(2, 20) name!: string
  @IsOptional() @IsInt() @Min(0) age?: number
}

// main.ts
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,             // 去掉 DTO 中没有声明的属性
  forbidNonWhitelisted: true,  // 有未声明属性时直接报错
  transform: true,             // 把普通对象转换成 DTO 实例，并做基础类型转换
}))
```

原理：`ValidationPipe` 通过 `metatype` 拿到 DTO 类 → 用 class-transformer 把普通对象转成类实例 → 用 class-validator 读取属性上的校验装饰器并执行。

注意：

- DTO 必须是 **class**，不能是 interface（运行时不存在）
- `whitelist` 能防止「批量赋值（mass assignment）」漏洞，比如用户在请求体中多传 `role: 'admin'`
- 嵌套对象需要 `@ValidateNested()` + `@Type(() => AddressDto)`

**口述模板（30–60 秒）**

1. 一句话结论：DTO 用 class + 校验装饰器描述输入，全局 ValidationPipe 自动校验。
2. 关键机制：通过参数的类型元数据找到 DTO 类，转实例再校验。
3. 注意点：DTO 必须是类；建议开启 whitelist 防止批量赋值。
4. 可追问时补充：v12 可以改用 Zod 等 Standard Schema 库。

**常见追问**

- 为什么 DTO 不能用 interface？→ interface 编译后不存在，ValidationPipe 拿不到可以读取装饰器的类。

---

### Q10. 守卫（Guard）是什么？怎么实现鉴权？

- **难度**: junior
- **标签**: 守卫, 鉴权

**答案**

守卫决定**请求能否进入处理器**，返回 `true` 放行，返回 `false` 时 Nest 抛出 `ForbiddenException`（403）；也可以自己抛 `UnauthorizedException`（401）。

```ts
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly jwt: JwtService) {}

  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    const req = ctx.switchToHttp().getRequest<Request>()
    const token = req.headers.authorization?.replace(/^Bearer /, '')
    if (!token) throw new UnauthorizedException()
    try {
      req.user = await this.jwt.verifyAsync(token)
      return true
    } catch {
      throw new UnauthorizedException()
    }
  }
}

@UseGuards(AuthGuard)
@Controller('orders')
export class OrdersController {}
```

守卫和中间件的区别：守卫拿得到 `ExecutionContext`，知道**接下来执行哪个处理器**，可以读取处理器上的元数据（如 `@Roles('admin')`）。

**口述模板（30–60 秒）**

1. 一句话结论：守卫在处理器之前决定是否放行，用于认证和授权。
2. 关键机制：实现 canActivate，通过 ExecutionContext 读取请求和处理器元数据。
3. 注意点：返回 false 得到 403，未登录应主动抛 401。
4. 可追问时补充：配合 Reflector 读取 @Roles、@Public 元数据实现声明式权限。

**常见追问**

- 守卫执行时管道执行了吗？→ 没有。守卫在管道之前，所以守卫中拿到的路由参数是原始字符串，没有经过 ParseIntPipe。

---

### Q11. 拦截器（Interceptor）能做什么？

- **难度**: junior
- **标签**: 拦截器

**答案**

拦截器包裹处理器的执行，可以在**之前和之后**加逻辑，基于 RxJS：

```ts
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const start = Date.now()
    return next.handle().pipe(
      map(data => ({ code: 0, data })),                     // 统一响应结构
      tap(() => console.log(`cost ${Date.now() - start}ms`)),
    )
  }
}
```

典型用途：

- 统一响应包装
- 记录耗时、日志
- 缓存（命中时不调用 `next.handle()`，直接返回）
- 超时（`timeout(5000)`）
- 异常映射（`catchError`）
- 序列化（`ClassSerializerInterceptor`）

**口述模板（30–60 秒）**

1. 一句话结论：拦截器在处理器前后加逻辑，可以改写返回值或异常。
2. 关键机制：next.handle() 返回 Observable，用 RxJS 操作符处理结果。
3. 注意点：不调用 next.handle() 处理器就不会执行。
4. 可追问时补充：多个拦截器像洋葱一样嵌套，返回阶段顺序相反。

**常见追问**

- 拦截器和中间件都能计时，选哪个？→ 拦截器能拿到处理器信息和返回值，计时更准确；中间件适合与路由无关的通用处理。

---

### Q12. 异常过滤器（Exception Filter）怎么用？

- **难度**: junior
- **标签**: 异常

**答案**

Nest 有内置的全局异常层：未处理的 `HttpException` 会被转为对应的响应；其他异常返回 500 `Internal server error`。

```ts
throw new NotFoundException(`user ${id} not found`)
throw new HttpException({ code: 'USER_DISABLED', message: '用户已禁用' }, 403)
```

自定义过滤器统一错误格式：

```ts
@Catch()   // 不传参数：捕获所有异常
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(private readonly adapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost) {
    const { httpAdapter } = this.adapterHost
    const ctx = host.switchToHttp()
    const status = exception instanceof HttpException ? exception.getStatus() : 500
    const body = {
      code: status,
      message: exception instanceof HttpException ? exception.message : 'Internal error',
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    }
    httpAdapter.reply(ctx.getResponse(), body, status)
  }
}
```

使用 `HttpAdapterHost` 而不是直接调用 `res.status().json()`，可以同时兼容 Express 和 Fastify。

**口述模板（30–60 秒）**

1. 一句话结论：过滤器捕获异常并生成响应，用于统一错误格式。
2. 关键机制：@Catch 指定异常类型，catch 方法通过 ArgumentsHost 拿到响应对象。
3. 注意点：非 HttpException 默认返回 500，需要记录日志且不能暴露内部细节。
4. 可追问时补充：通过 HttpAdapterHost 写响应，兼容不同的 HTTP 平台。

**常见追问**

- 多个过滤器都匹配时用哪个？→ 只执行一个：路由级优先，然后是控制器级、全局级。

---

### Q13. 管道、守卫、拦截器、过滤器、中间件分别用 `UseXxx` 怎么绑定？

- **难度**: junior
- **标签**: 绑定方式

**答案**

| 组件 | 路由 / 控制器级 | 全局（不能注入依赖） | 全局（可以注入依赖） |
|------|----------------|---------------------|---------------------|
| 守卫 | `@UseGuards()` | `app.useGlobalGuards()` | `{ provide: APP_GUARD, useClass }` |
| 拦截器 | `@UseInterceptors()` | `app.useGlobalInterceptors()` | `APP_INTERCEPTOR` |
| 管道 | `@UsePipes()` / 参数级 | `app.useGlobalPipes()` | `APP_PIPE` |
| 过滤器 | `@UseFilters()` | `app.useGlobalFilters()` | `APP_FILTER` |
| 中间件 | `configure(consumer)` | `app.use()` | — |

```ts
@Module({
  providers: [
    { provide: APP_GUARD, useClass: JwtAuthGuard },
    { provide: APP_INTERCEPTOR, useClass: LoggingInterceptor },
  ],
})
export class AppModule {}
```

推荐用 `APP_*` 的方式注册全局组件：它们在模块内部创建，**可以注入其他 Provider**（如 `Reflector`、`ConfigService`）。`app.useGlobalXxx(new X())` 在容器之外创建实例，无法注入依赖。

传类（`@UseGuards(AuthGuard)`）而不是实例，让容器负责创建并注入依赖。

**口述模板（30–60 秒）**

1. 一句话结论：每类组件都有路由、控制器、全局三种绑定方式。
2. 关键机制：APP_GUARD 等 Token 让全局组件进入 DI 容器。
3. 注意点：useGlobalXxx(new X()) 无法注入依赖。
4. 可追问时补充：多个 APP_GUARD 按注册顺序执行。

**常见追问**

- `APP_GUARD` 写在哪个模块都一样吗？→ 无论注册在哪个模块，都是全局生效的；建议放在根模块或核心模块中便于查找。

---

## 三、常用官方模块

### Q14. 配置管理怎么做？`@nestjs/config` 怎么用？

- **难度**: junior
- **标签**: 配置

**答案**

```ts
// database.config.ts
export default registerAs('database', () => ({
  url: process.env.DATABASE_URL,
  poolSize: Number(process.env.DB_POOL_SIZE ?? 10),
}))

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,                      // 全局可用，无需各处 imports
      load: [databaseConfig],
      envFilePath: ['.env.local', '.env'],
      validationSchema: envSchema,         // 启动时校验环境变量
    }),
  ],
})
export class AppModule {}

@Injectable()
export class DbService {
  constructor(
    @Inject(databaseConfig.KEY) private readonly db: ConfigType<typeof databaseConfig>,
  ) {}
}
```

要点：

- **启动时校验**，缺少关键配置直接启动失败（快速失败），不要等到运行时才报错
- 使用 `registerAs` + `ConfigType` 获得类型安全的配置对象，避免到处 `config.get('xxx')` 写字符串
- 敏感信息走环境变量或密钥管理服务，不提交到仓库
- v12 起配置校验基于 Standard Schema，可直接使用 Zod；继续使用 Joi 需要升级到 v18 并调整选项位置

**口述模板（30–60 秒）**

1. 一句话结论：用 ConfigModule 加载环境变量，用 registerAs 按命名空间组织配置。
2. 关键机制：启动时校验配置，缺失直接失败。
3. 注意点：用 ConfigType 注入强类型配置，避免字符串 key。
4. 可追问时补充：v12 的配置校验支持 Zod 等 Standard Schema 库。

**常见追问**

- 动态模块需要配置时怎么拿到 ConfigService？→ 使用 `forRootAsync({ imports: [ConfigModule], inject: [ConfigService], useFactory })`。

---

### Q15. Nest 中怎么连接数据库？常见 ORM 怎么选？

- **难度**: junior
- **标签**: 数据库

**答案**

| 方案 | 特点 |
|------|------|
| TypeORM（`@nestjs/typeorm`） | 官方集成最早，装饰器实体，Active Record / Data Mapper；类型安全一般 |
| Prisma | Schema 文件生成类型安全的客户端，开发体验好；复杂 SQL 需要原生查询 |
| Drizzle | SQL 风格、轻量、类型推导强，近两年流行 |
| MikroORM | Unit of Work + Identity Map，DDD 友好 |
| Mongoose（`@nestjs/mongoose`） | MongoDB |

TypeORM 用法：

```ts
@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (cfg: ConfigService) => ({
        type: 'postgres',
        url: cfg.getOrThrow('DATABASE_URL'),
        autoLoadEntities: true,
        synchronize: false,      // 生产环境必须关闭，使用迁移
      }),
    }),
    TypeOrmModule.forFeature([User]),
  ],
})
export class UsersModule {}

@Injectable()
export class UsersService {
  constructor(@InjectRepository(User) private readonly repo: Repository<User>) {}
}
```

Prisma 通常封装为一个 `PrismaService extends PrismaClient`，在 `onModuleInit` 中连接。

**口述模板（30–60 秒）**

1. 一句话结论：官方集成 TypeORM、Mongoose，社区常用 Prisma 和 Drizzle。
2. 关键机制：用 forRootAsync 读取配置建立连接，用 forFeature 注册仓库。
3. 注意点：生产环境关闭 synchronize，使用迁移脚本。
4. 可追问时补充：选型看类型安全、复杂查询能力、团队熟悉度。

**常见追问**

- `synchronize: true` 为什么危险？→ 实体变化时会自动修改表结构，可能删除列和数据。

---

### Q16. 怎么生成 Swagger / OpenAPI 文档？

- **难度**: junior
- **标签**: 文档

**答案**

```ts
const config = new DocumentBuilder()
  .setTitle('Order API')
  .setVersion('1.0')
  .addBearerAuth()
  .build()
const document = SwaggerModule.createDocument(app, config)
SwaggerModule.setup('docs', app, document)
```

DTO 属性用 `@ApiProperty()` 描述。可以开启 `@nestjs/swagger` 的 **CLI 插件**，在编译期根据 TS 类型和注释自动补全 `@ApiProperty`，减少样板代码。

生产注意：文档接口需要鉴权或只在内网开放；可以把 OpenAPI 文档导出给前端生成类型和请求函数，实现前后端类型对齐。

**口述模板（30–60 秒）**

1. 一句话结论：用 SwaggerModule 从控制器和 DTO 元数据生成 OpenAPI 文档。
2. 关键机制：DocumentBuilder 描述全局信息，装饰器描述接口细节。
3. 注意点：开启 CLI 插件可以减少重复的 ApiProperty。
4. 可追问时补充：文档导出给前端代码生成，打通端到端类型。

**常见追问**

- 文档和实际实现不一致怎么办？→ 尽量让文档从代码元数据生成而不是手写；CI 中检查文档变更。

---

### Q17. 生命周期钩子有哪些？执行顺序是什么？

- **难度**: junior
- **标签**: 生命周期

**答案**

| 钩子 | 时机 |
|------|------|
| `onModuleInit()` | 模块的依赖解析完成后 |
| `onApplicationBootstrap()` | 所有模块初始化完成，开始监听连接之前 |
| `onModuleDestroy()` | 收到终止信号后 |
| `beforeApplicationShutdown(signal)` | onModuleDestroy 完成后，连接关闭之前 |
| `onApplicationShutdown(signal)` | 连接关闭之后 |

```ts
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnApplicationShutdown {
  async onModuleInit() { await this.$connect() }
  async onApplicationShutdown() { await this.$disconnect() }
}

// main.ts：关闭钩子默认不监听信号，需要手动开启
app.enableShutdownHooks()
```

顺序：模块按在导入图中离根模块的距离调用，**被导入得最深的模块（以及全局模块）最先**，根模块最后；关闭时相反。钩子返回 Promise 时，Nest 会等待它完成。v12 起改为按组件层级调用（依赖先于依赖方），升级时要检查相关的顺序假设。

注意：请求作用域和瞬态作用域的 Provider **不会触发**这些钩子。

**口述模板（30–60 秒）**

1. 一句话结论：启动有 onModuleInit、onApplicationBootstrap，关闭有 onModuleDestroy、beforeApplicationShutdown、onApplicationShutdown。
2. 关键机制：被依赖的模块先初始化，关闭顺序相反，异步钩子会被等待。
3. 注意点：关闭钩子需要 enableShutdownHooks；请求作用域的 Provider 不触发钩子。
4. 可追问时补充：v12 改为按组件层级调用，升级时检查顺序依赖。

**常见追问**

- `enableShutdownHooks` 有什么代价？→ 会为每个应用实例注册信号监听；同一进程中创建大量应用实例（如测试）时要注意监听器数量。

---

### Q18. Nest 中怎么做定时任务和异步队列？

- **难度**: junior
- **标签**: 定时任务, 队列

**答案**

定时任务（`@nestjs/schedule`）：

```ts
@Injectable()
export class ReportTask {
  @Cron('0 2 * * *', { timeZone: 'Asia/Shanghai' })
  async daily() { /* 生成日报 */ }

  @Interval(60_000)
  heartbeat() {}
}
```

问题：**多实例部署时每个实例都会执行**。需要分布式锁（Redis `SET NX PX`），或交给专门的调度服务 / 队列。

队列（`@nestjs/bullmq`，基于 Redis）：

```ts
@Module({ imports: [BullModule.registerQueue({ name: 'email' })] })
export class EmailModule {}

await this.queue.add('welcome', { userId }, { attempts: 3, backoff: { type: 'exponential', delay: 1000 } })

@Processor('email')
export class EmailProcessor extends WorkerHost {
  async process(job: Job<{ userId: string }>) { /* 发送邮件 */ }
}
```

队列适合：耗时任务异步化、削峰、重试、延迟任务。

**口述模板（30–60 秒）**

1. 一句话结论：定时任务用 @nestjs/schedule，异步任务用 BullMQ。
2. 关键机制：Cron 装饰器声明调度，队列由生产者添加任务、消费者处理。
3. 注意点：多实例下 Cron 会重复执行，需要分布式锁。
4. 可追问时补充：队列支持重试、退避、延迟和并发控制。

**常见追问**

- 队列任务要注意什么？→ 消费逻辑必须幂等，因为重试会导致重复执行。

---

### Q19. 怎么做限流？

- **难度**: junior
- **标签**: 限流, 安全

**答案**

使用 `@nestjs/throttler`：

```ts
@Module({
  imports: [ThrottlerModule.forRoot([{ name: 'default', ttl: 60_000, limit: 100 }])],
  providers: [{ provide: APP_GUARD, useClass: ThrottlerGuard }],
})
export class AppModule {}

@Throttle({ default: { limit: 5, ttl: 60_000 } })   // 登录接口更严格
@Post('login')
login() {}

@SkipThrottle()
@Get('health')
health() {}
```

注意：

- 默认存储在**内存**中，多实例时每个实例单独计数，需要换成 Redis 存储
- 在反向代理后面，要正确识别客户端 IP（`trust proxy`），否则所有请求都来自代理 IP
- 可以重写 `getTracker` 按用户 ID 或 API Key 限流

**口述模板（30–60 秒）**

1. 一句话结论：用 ThrottlerModule 加全局守卫实现限流，敏感接口单独收紧。
2. 关键机制：限流本质是一个守卫，按 tracker 计数。
3. 注意点：多实例要用 Redis 存储；代理后要正确获取真实 IP。
4. 可追问时补充：可按用户或 API Key 限流，网关层还可以再做一层。

**常见追问**

- 应用层限流和网关限流怎么分工？→ 网关做粗粒度的全局保护，应用层做与业务相关的细粒度限流（如登录、短信）。

---

### Q20. 怎么做缓存？

- **难度**: junior
- **标签**: 缓存

**答案**

使用 `@nestjs/cache-manager`（Nest 11 起基于 cache-manager v6 / Keyv）：

```ts
CacheModule.registerAsync({
  isGlobal: true,
  useFactory: () => ({
    stores: [createKeyv('redis://localhost:6379')],
    ttl: 60_000,                         // 单位：毫秒
  }),
})

// 1. 自动缓存 GET 接口
@UseInterceptors(CacheInterceptor)
@CacheTTL(30_000)
@Get('hot')
hot() {}

// 2. 手动缓存
constructor(@Inject(CACHE_MANAGER) private cache: Cache) {}
const cached = await this.cache.get<Product>(key)
```

注意：

- `CacheInterceptor` 默认只缓存 GET，且按 URL 作为 key，**需要按用户区分的数据不能直接用**
- 值会序列化为 JSON，类实例和 `Date` 读出后会变成普通对象和字符串
- 缓存穿透、击穿、雪崩仍需要自己处理（见中级题）

**口述模板（30–60 秒）**

1. 一句话结论：CacheModule 提供统一缓存接口，支持拦截器自动缓存和手动读写。
2. 关键机制：v11 起基于 Keyv，可配置多级存储，TTL 单位是毫秒。
3. 注意点：默认按 URL 缓存，个性化数据要自定义 key。
4. 可追问时补充：缓存三大问题需要业务层处理。

**常见追问**

- 怎么自定义缓存 key？→ 继承 `CacheInterceptor` 重写 `trackBy`，把用户 ID 或租户 ID 加入 key。

---

## 四、工程实践

### Q21. Nest CLI 常用命令有哪些？

- **难度**: junior
- **标签**: CLI

**答案**

```sh
nest new my-app                 # 创建项目（v12 会询问 CJS 还是 ESM）
nest g resource users           # 生成模块 + 控制器 + 服务 + DTO（CRUD 模板）
nest g module/controller/service/guard/interceptor/pipe xxx
nest start --watch              # 开发模式
nest start -b swc               # 使用 SWC 编译，速度快很多
nest build
nest upgrade --dry-run          # v12：升级所有 @nestjs/* 包并输出需要人工检查的报告
```

v12 的 CLI 变化：webpack 构建废弃，改为 `--builder rspack`；新建 ESM 项目默认使用 Vitest 和 oxlint，CJS 项目仍使用 Jest。

**口述模板（30–60 秒）**

1. 一句话结论：CLI 负责创建项目、生成代码、构建和启动。
2. 关键机制：`nest g resource` 一次生成完整的 CRUD 骨架。
3. 注意点：大型项目开发时建议用 SWC 编译提速。
4. 可追问时补充：v12 用 Rspack 替代 webpack，提供 nest upgrade 辅助升级。

**常见追问**

- SWC 编译有什么限制？→ 不做类型检查，需要单独运行 `tsc --noEmit`；需要正确配置装饰器元数据支持。

---

### Q22. 单元测试怎么写？

- **难度**: junior
- **标签**: 测试

**答案**

```ts
describe('UsersService', () => {
  let service: UsersService
  const repo = { findOneBy: vi.fn() }

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: getRepositoryToken(User), useValue: repo },   // 替换真实依赖
      ],
    }).compile()
    service = moduleRef.get(UsersService)
  })

  it('throws when user not found', async () => {
    repo.findOneBy.mockResolvedValue(null)
    await expect(service.findOne(1)).rejects.toThrow(NotFoundException)
  })
})
```

要点：

- `Test.createTestingModule` 创建一个只包含所需 Provider 的小型容器
- 用 `useValue` 或 `.overrideProvider(X).useValue(mock)` 替换依赖
- v12 新建 ESM 项目默认使用 Vitest，CJS 项目仍为 Jest

**口述模板（30–60 秒）**

1. 一句话结论：用 TestingModule 构建隔离的容器，Mock 掉外部依赖。
2. 关键机制：useValue 或 overrideProvider 替换 Provider。
3. 注意点：单元测试只测业务逻辑，不连接真实数据库。
4. 可追问时补充：e2e 测试用 supertest 测完整链路。

**常见追问**

- 怎么拿到请求作用域的 Provider？→ 使用 `moduleRef.resolve()` 而不是 `get()`。

---

### Q23. e2e 测试怎么写？

- **难度**: junior
- **标签**: 测试

**答案**

```ts
describe('Users (e2e)', () => {
  let app: INestApplication

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({ imports: [AppModule] })
      .overrideProvider(MailService).useValue({ send: vi.fn() })   // 外部服务替换掉
      .overrideGuard(JwtAuthGuard).useValue({ canActivate: () => true })
      .compile()
    app = moduleRef.createNestApplication()
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }))   // 与 main.ts 保持一致
    await app.init()
  })

  afterAll(() => app.close())

  it('POST /users validates body', () =>
    request(app.getHttpServer()).post('/users').send({ email: 'bad' }).expect(400))
})
```

要点：

- 在 `main.ts` 中注册的全局配置（管道、前缀、过滤器）**不会自动生效**，需要在测试中再设置一次。更好的做法是抽成 `setupApp(app)` 函数共用，或用 `APP_PIPE` 等方式注册到模块中
- 数据库使用独立的测试库或容器（Testcontainers），每个用例前清理数据

**口述模板（30–60 秒）**

1. 一句话结论：e2e 测试导入整个 AppModule，用 supertest 发真实请求。
2. 关键机制：overrideProvider / overrideGuard 替换外部依赖。
3. 注意点：main.ts 中的全局配置要在测试中复用。
4. 可追问时补充：用 Testcontainers 启动真实数据库，保证测试与生产一致。

**常见追问**

- 为什么推荐用 `APP_PIPE` 注册全局管道？→ 它属于模块的一部分，测试导入模块时自动生效，避免 main.ts 和测试配置不一致。

---

### Q24. 怎么切换到 Fastify？有什么好处和代价？

- **难度**: junior
- **标签**: 平台适配

**答案**

```ts
const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter())
await app.listen(3000, '0.0.0.0')     // 容器中需要显式监听 0.0.0.0
```

好处：Fastify 的路由和 JSON 序列化更快，吞吐量明显高于 Express。

代价：

- Express 中间件不一定兼容（需要使用 Fastify 插件版本，如 `@fastify/helmet`）
- 直接操作 `req` / `res` 的代码要改写
- 文件上传、会话等功能的用法不同

建议：新项目追求性能时可直接用 Fastify；业务代码尽量不直接依赖底层的 `req` / `res`，以便切换。

**口述模板（30–60 秒）**

1. 一句话结论：通过 FastifyAdapter 切换平台，换取更高的吞吐量。
2. 关键机制：Nest 通过 HttpAdapter 抽象屏蔽平台差异。
3. 注意点：Express 中间件和直接操作 req/res 的代码需要适配。
4. 可追问时补充：真实服务的瓶颈常在数据库和下游，框架切换的收益要实测。

**常见追问**

- Fastify 默认只监听 localhost，在容器中会怎样？→ 外部无法访问，需要监听 `0.0.0.0`。

---

### Q25. 自定义参数装饰器怎么写？

- **难度**: junior
- **标签**: 装饰器

**答案**

```ts
export const CurrentUser = createParamDecorator(
  (field: keyof JwtUser | undefined, ctx: ExecutionContext) => {
    const user = ctx.switchToHttp().getRequest().user as JwtUser
    return field ? user?.[field] : user
  },
)

@Get('me')
me(@CurrentUser() user: JwtUser, @CurrentUser('id') userId: string) {}
```

自定义参数装饰器同样会经过管道：如果开启了 `ValidationPipe` 的 `validateCustomDecorators`，也会对它的结果做校验。

**口述模板（30–60 秒）**

1. 一句话结论：用 createParamDecorator 从执行上下文中提取数据。
2. 关键机制：工厂函数接收装饰器参数和 ExecutionContext。
3. 注意点：依赖守卫先把 user 挂到 req 上。
4. 可追问时补充：可以组合多个装饰器为一个（applyDecorators）。

**常见追问**

- 参数装饰器能注入 Service 吗？→ 不能直接注入；需要配合管道（管道可以注入依赖）完成，比如 `@CurrentUser(LoadUserPipe)`。

---

### Q26. WebSocket 在 Nest 中怎么用？

- **难度**: junior
- **标签**: WebSocket

**答案**

```ts
@WebSocketGateway({ namespace: '/chat', cors: { origin: '*' } })
export class ChatGateway implements OnGatewayConnection {
  @WebSocketServer() server!: Server

  handleConnection(client: Socket) {
    // 握手阶段鉴权：校验 token，失败则断开
  }

  @SubscribeMessage('message')
  onMessage(@MessageBody() body: { room: string; text: string }, @ConnectedSocket() client: Socket) {
    this.server.to(body.room).emit('message', body)
  }
}
```

要点：

- 默认基于 Socket.IO，也可以用 `ws` 适配器
- 守卫、管道、拦截器、过滤器同样可以用于网关（异常要用 `WsException`）
- 多实例部署需要 Redis Adapter，让消息能广播到其他实例上的连接

**口述模板（30–60 秒）**

1. 一句话结论：用 @WebSocketGateway 定义网关，@SubscribeMessage 处理消息。
2. 关键机制：网关也是 Provider，可以注入 Service。
3. 注意点：在连接阶段鉴权，异常使用 WsException。
4. 可追问时补充：多实例时用 Redis Adapter 跨实例广播。

**常见追问**

- 负载均衡需要注意什么？→ Socket.IO 的长轮询降级需要会话粘滞（sticky session），或只允许 WebSocket 传输。

---

### Q27. 微服务模块是什么？怎么创建一个微服务？

- **难度**: junior
- **标签**: 微服务

**答案**

`@nestjs/microservices` 提供基于消息的通信抽象，支持 TCP、Redis、NATS、MQTT、RabbitMQ、Kafka、gRPC 等传输层。

```ts
// 服务端
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.TCP,
  options: { port: 4000 },
})

@Controller()
export class MathController {
  @MessagePattern({ cmd: 'sum' })          // 请求-响应
  sum(@Payload() nums: number[]) { return nums.reduce((a, b) => a + b, 0) }

  @EventPattern('user.created')            // 事件，无需响应
  onUserCreated(@Payload() data: UserCreated) {}
}

// 客户端
@Module({ imports: [ClientsModule.register([{ name: 'MATH', transport: Transport.TCP, options: { port: 4000 } }])] })
constructor(@Inject('MATH') private client: ClientProxy) {}
const total = await firstValueFrom(this.client.send({ cmd: 'sum' }, [1, 2]))
this.client.emit('user.created', user)
```

**口述模板（30–60 秒）**

1. 一句话结论：Nest 微服务用消息模式通信，传输层可插拔。
2. 关键机制：MessagePattern 是请求响应，EventPattern 是发布事件。
3. 注意点：send 返回冷 Observable，不订阅就不会发送。
4. 可追问时补充：一个应用可以同时提供 HTTP 和微服务（混合应用）。

**常见追问**

- `client.send()` 不加 `firstValueFrom` 会怎样？→ Observable 是惰性的，不订阅就不会发出请求。

---

### Q28. 日志怎么做？

- **难度**: junior
- **标签**: 日志

**答案**

内置 `Logger`：

```ts
private readonly logger = new Logger(OrdersService.name)
this.logger.log('order created', { orderId })

// v11 起支持 JSON 输出，便于日志平台采集
const app = await NestFactory.create(AppModule, { logger: new ConsoleLogger({ json: true }) })
```

v12 起 `ConsoleLogger` 默认开启结构化参数（`structuredParams`），附加的对象参数会作为结构化字段输出。

生产常用 **pino**（`nestjs-pino`）：性能好，自动记录请求日志，结合 AsyncLocalStorage 为每条日志带上请求 ID。

规范：

- 结构化日志（JSON），包含 `requestId`、`traceId`、`userId`
- 不记录密码、Token、身份证号等敏感信息（使用脱敏配置）
- 生产环境控制日志级别

**口述模板（30–60 秒）**

1. 一句话结论：开发用内置 Logger，生产用结构化的 JSON 日志，常用 pino。
2. 关键机制：日志带上请求 ID 和链路 ID，便于串联排查。
3. 注意点：敏感信息必须脱敏。
4. 可追问时补充：v11 内置 JSON 输出，v12 默认结构化参数。

**常见追问**

- 怎么让每条日志自动带上请求 ID？→ 用 AsyncLocalStorage（nestjs-cls 或 nestjs-pino 内置）保存请求上下文，日志库从中读取。

---

### Q29. 健康检查怎么做？

- **难度**: junior
- **标签**: 运维

**答案**

```ts
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    private memory: MemoryHealthIndicator,
  ) {}

  @Get('live')
  live() { return { status: 'ok' } }          // 存活探针：只检查进程本身

  @Get('ready')
  @HealthCheck()
  ready() {                                    // 就绪探针：检查关键依赖
    return this.health.check([
      () => this.db.pingCheck('database', { timeout: 1000 }),
      () => this.memory.checkHeap('heap', 512 * 1024 * 1024),
    ])
  }
}
```

要点：

- **存活（liveness）** 和 **就绪（readiness）** 要分开：存活检查不应依赖数据库，否则数据库故障会导致所有实例被反复重启
- 优雅关闭时先让就绪检查返回失败，让负载均衡摘除流量

**口述模板（30–60 秒）**

1. 一句话结论：用 @nestjs/terminus 提供存活和就绪两个检查接口。
2. 关键机制：就绪检查依赖项，存活检查只看进程。
3. 注意点：存活检查不能依赖外部服务。
4. 可追问时补充：关闭流程中先把就绪置为失败再停止服务。

**常见追问**

- 为什么存活检查不能查数据库？→ 数据库故障时所有实例的存活检查都会失败，导致集体重启，问题更严重。

---

### Q30. Nest 项目的目录结构怎么组织？

- **难度**: junior
- **标签**: 工程结构

**答案**

按**业务领域（feature）**而不是按技术类型组织：

```text
src/
├─ main.ts
├─ app.module.ts
├─ common/              # 通用组件：过滤器、拦截器、守卫、装饰器、工具
├─ config/              # 配置定义与校验
├─ infra/               # 基础设施：数据库、Redis、消息队列、外部 API 客户端
└─ modules/
   ├─ users/
   │  ├─ users.module.ts
   │  ├─ users.controller.ts
   │  ├─ users.service.ts
   │  ├─ users.repository.ts
   │  ├─ dto/
   │  └─ entities/
   └─ orders/
```

原则：

- 每个业务模块只通过 `exports` 暴露必要的服务
- 模块之间避免循环依赖；共享的部分下沉到更底层的模块
- 业务逻辑放在 Service，Controller 保持轻量
- 复杂业务可以进一步分为 application（用例）/ domain（领域模型）/ infrastructure（实现）三层

**口述模板（30–60 秒）**

1. 一句话结论：按业务领域划分模块，通用组件和基础设施单独存放。
2. 关键机制：模块通过 exports 控制边界。
3. 注意点：避免按 controllers/services 这种技术类型分目录，否则业务分散。
4. 可追问时补充：复杂领域可以引入分层架构或 DDD。

**常见追问**

- 大型项目怎么拆分？→ 使用 Nest 的 Monorepo 模式（apps + libs），或 Nx / pnpm workspace 管理多个应用和共享库。
