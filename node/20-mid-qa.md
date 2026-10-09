# Node.js 中级 Q&A（原理与工程实践）

> 目标：讲清事件循环、Stream、模块系统、并发模型、框架原理、错误处理和常见后端工程问题。

---

## 一、事件循环

### Q1. Node.js 事件循环有哪些阶段？

- **难度**: mid
- **标签**: 事件循环

**答案**

libuv 事件循环 6 个阶段：

```text
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout / setInterval 到期回调
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  上一轮延迟的部分系统 I/O 回调（如 TCP 错误）
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  内部使用
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  获取新 I/O 事件并执行回调；可能在此阻塞等待
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate 回调
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  socket.on('close') 等
   └───────────────────────────┘
```

每个回调执行完后，都会先清空 `process.nextTick` 队列，再清空 Promise 微任务队列（Node 11+ 行为，与浏览器一致）。

**口述模板（30–60 秒）**

1. 一句话结论：timers、pending、idle/prepare、poll、check、close 六阶段。
2. 关键机制：poll 负责 I/O，check 执行 setImmediate。
3. 注意点：每个回调之后清空 nextTick 和微任务。
4. 可追问时补充：poll 阶段的阻塞时长由最近定时器决定。

**常见追问**

- 没有任何任务时进程会怎样？→ 事件循环没有活跃句柄和请求时，进程退出。

---

### Q2. 输出题：nextTick、Promise、setTimeout、setImmediate

- **难度**: mid
- **标签**: 事件循环, 输出题

**答案**

```js
// CommonJS 文件（index.cjs）
const fs = require('node:fs')

console.log('1')

setTimeout(() => console.log('2'), 0)
setImmediate(() => console.log('3'))

Promise.resolve().then(() => {
  console.log('4')
  process.nextTick(() => console.log('5'))
})

process.nextTick(() => {
  console.log('6')
  Promise.resolve().then(() => console.log('7'))
})

fs.readFile(__filename, () => {
  setTimeout(() => console.log('8'), 0)
  setImmediate(() => console.log('9'))
})

console.log('10')
```

输出：

```text
1 10 6 4 7 5 2 3 9 8
```

说明：

- 同步：1、10
- nextTick 队列：6（并加入 Promise 微任务 7）
- Promise 队列：4（加入 nextTick 5），7
- 微任务清空后，再次检查 nextTick：5
- 主模块中 2 和 3 顺序**理论上不确定**（取决于进入循环时是否已过 1ms），常见输出为 2 3
- I/O 回调中：check 阶段在 timers 之前，所以 9 一定先于 8

> 加分点：同样的代码放在 ESM 文件中，顶层代码本身运行在微任务中，模块执行完会先清空 Promise 队列再处理 nextTick，前半段变为 `1 10 4 6 5 7`。面试题一般默认 CommonJS。

**口述模板（30–60 秒）**

1. 一句话结论：同步 → nextTick → Promise → 事件循环各阶段。
2. 关键机制：微任务执行中产生的 nextTick，会在当前微任务队列清空后执行。
3. 注意点：主模块 setTimeout 与 setImmediate 顺序不确定。
4. 可追问时补充：I/O 回调中 setImmediate 必然优先。

**常见追问**

- 为什么主模块中顺序不确定？→ setTimeout 最小 1ms，进入事件循环时可能未到期。

---

### Q3. libuv 线程池处理哪些任务？如何调优？

- **难度**: mid
- **标签**: libuv, 线程池

**答案**

走线程池的操作：

- 大部分 `fs` 操作
- `dns.lookup`（而 `dns.resolve*` 走网络异步，不占线程池）
- `crypto` 的 pbkdf2、scrypt、randomBytes（异步版）等
- `zlib` 异步压缩

**不走线程池**：TCP/UDP 网络 I/O（系统 epoll/kqueue/IOCP）。

默认 4 个线程，可通过环境变量调整（需在线程池首次使用前设置）：

```bash
UV_THREADPOOL_SIZE=16 node app.js
```

问题场景：大量 bcrypt/pbkdf2 或文件操作把 4 个线程占满，其他 fs 操作和 `dns.lookup` 排队，表现为「HTTP 请求变慢」。

**口述模板（30–60 秒）**

1. 一句话结论：文件、DNS lookup、加密、压缩走线程池，网络 I/O 不走。
2. 关键机制：默认 4 线程，任务排队执行。
3. 注意点：线程池饱和会让看似无关的操作变慢。
4. 可追问时补充：外部请求 DNS 解析排队导致超时。

**常见追问**

- 调大线程池一定好吗？→ 受 CPU 核数限制，过大增加上下文切换；CPU 密集任务应放 Worker 或独立服务。

---

### Q4. 什么会阻塞事件循环？如何检测？

- **难度**: mid
- **标签**: 性能

**答案**

常见阻塞源：

- 同步 I/O：`readFileSync`、`execSync`
- 大 JSON 的 `JSON.parse` / `JSON.stringify`
- 复杂正则（ReDoS）
- 同步加密：`crypto.pbkdf2Sync`、`bcrypt` 同步版
- 大循环、大数组排序
- 大量同步日志

检测：

```js
import { monitorEventLoopDelay } from 'node:perf_hooks'

const histogram = monitorEventLoopDelay({ resolution: 20 })
histogram.enable()

setInterval(() => {
  logger.info({
    p99: histogram.percentile(99) / 1e6, // 纳秒转毫秒
    max: histogram.max / 1e6,
  }, 'event loop delay')
  histogram.reset()
}, 10_000)
```

也可用 `performance.eventLoopUtilization()`、Clinic.js Doctor、CPU Profile 定位。

**口述模板（30–60 秒）**

1. 一句话结论：同步 CPU 工作会阻塞所有请求。
2. 关键机制：主线程只有一个，事件循环延迟上升。
3. 注意点：监控事件循环延迟 P99 作为核心指标。
4. 可追问时补充：拆分任务、Worker、流式 JSON。

**常见追问**

- 怎么拆分 CPU 任务？→ 分批处理，每批之间 `await setImmediate()`（`node:timers/promises`），或放 Worker。

---

## 二、Stream

### Q5. Node.js 有哪几种 Stream？

- **难度**: mid
- **标签**: Stream

**答案**

| 类型 | 说明 | 示例 |
|------|------|------|
| Readable | 可读 | `fs.createReadStream`、`http.IncomingMessage` |
| Writable | 可写 | `fs.createWriteStream`、`http.ServerResponse` |
| Duplex | 可读可写，两端独立 | `net.Socket` |
| Transform | 可读可写，输出由输入转换而来 | `zlib.createGzip`、`crypto.createCipheriv` |

另有 `objectMode` 用于传递对象而不是 Buffer。

**口述模板（30–60 秒）**

1. 一句话结论：四种流：可读、可写、双工、转换。
2. 关键机制：都继承 EventEmitter，内部有缓冲区。
3. 注意点：Duplex 两端独立，Transform 输出依赖输入。
4. 可追问时补充：Web Streams 互操作。

**常见追问**

- HTTP 的 req 和 res 分别是什么流？→ 服务端 req 是 Readable，res 是 Writable。

---

### Q6. 什么是背压（Backpressure）？如何处理？

- **难度**: mid
- **标签**: Stream, 背压

**答案**

生产速度快于消费速度时，数据在内存中堆积，导致内存暴涨。

机制：

- 每个流有 `highWaterMark`（字节流默认 64 KiB（Node 22+），对象模式 16 个对象）
- `writable.write(chunk)` 返回 `false` 表示缓冲区已超过阈值，应停止写入
- 等待 `'drain'` 事件后继续

```js
import { once } from 'node:events'

for await (const chunk of source) {
  if (!dest.write(chunk)) {
    await once(dest, 'drain')
  }
}
dest.end()
```

推荐直接使用 `pipeline`，自动处理背压、错误和资源清理：

```js
import { pipeline } from 'node:stream/promises'
await pipeline(createReadStream('in.log'), createGzip(), createWriteStream('in.log.gz'))
```

**口述模板（30–60 秒）**

1. 一句话结论：背压是消费者通知生产者放慢速度的机制。
2. 关键机制：write 返回 false，等待 drain。
3. 注意点：忽略返回值会导致内存无限增长。
4. 可追问时补充：pipeline 自动处理。

**常见追问**

- write 返回 false 后继续写会怎样？→ 数据仍会被缓冲，不会丢失，但内存持续增长。

---

### Q7. pipe 和 pipeline 有什么区别？

- **难度**: mid
- **标签**: Stream

**答案**

`a.pipe(b).pipe(c)` 的问题：

- 错误不会沿链传播，需要每个流单独监听 error
- 某个流出错时其他流不会自动销毁，可能导致文件句柄和内存泄漏

`pipeline(a, b, c)`：

- 任一流出错，销毁所有流并返回错误
- 支持 Promise 版本（`node:stream/promises`）
- 支持异步生成器作为转换步骤
- 支持 AbortSignal

```js
await pipeline(
  createReadStream('data.csv'),
  async function* (source) {
    for await (const chunk of source) yield chunk.toString().toUpperCase()
  },
  createWriteStream('out.csv'),
)
```

**口述模板（30–60 秒）**

1. 一句话结论：生产代码用 pipeline 替代 pipe。
2. 关键机制：统一错误传播和资源清理。
3. 注意点：pipe 出错时可能泄漏句柄。
4. 可追问时补充：异步生成器作为 Transform。

**常见追问**

- HTTP 响应用 pipeline 时客户端断开会怎样？→ res 被销毁，pipeline 报错并关闭源文件流。

---

### Q8. 可读流的两种模式是什么？如何消费？

- **难度**: mid
- **标签**: Stream

**答案**

- **流动模式（flowing）**：数据自动读取，通过 `'data'` 事件推送
- **暂停模式（paused）**：需要调用 `read()` 拉取

切换：监听 `data`、调用 `pipe()` 或 `resume()` 进入流动模式；`pause()` 回到暂停。

推荐用异步迭代消费，天然支持背压：

```js
import { createInterface } from 'node:readline'

const rl = createInterface({ input: createReadStream('big.log'), crlfDelay: Infinity })
for await (const line of rl) {
  if (line.includes('ERROR')) count++
}
```

**口述模板（30–60 秒）**

1. 一句话结论：流动模式推，暂停模式拉。
2. 关键机制：异步迭代按需拉取。
3. 注意点：监听 data 后若不处理背压可能堆积。
4. 可追问时补充：readline 逐行处理大文件。

**常见追问**

- 只监听 data 不监听 end 有问题吗？→ 可能遗漏结束和错误处理逻辑。

---

### Q9. Node Stream 和 Web Streams 有什么关系？

- **难度**: mid
- **标签**: Stream, Web 标准

**答案**

- Node Stream：Node 原生、基于 EventEmitter，生态成熟
- Web Streams（`ReadableStream`、`WritableStream`、`TransformStream`）：WHATWG 标准，浏览器、Deno、Bun、Edge Runtime 通用；`fetch` 的 body 是 Web Stream

互转：

```js
import { Readable, Writable } from 'node:stream'

const webStream = Readable.toWeb(nodeReadable)
const nodeStream = Readable.fromWeb(response.body)
```

跨运行时库（如 SSR 框架、Edge 函数）倾向 Web Streams。

**口述模板（30–60 秒）**

1. 一句话结论：Web Streams 是跨运行时标准，Node Stream 是传统实现，可互转。
2. 关键机制：fromWeb / toWeb。
3. 注意点：Node Stream 性能通常更好，生态更多。
4. 可追问时补充：React SSR 两种流 API。

**常见追问**

- fetch 下载大文件如何写入磁盘？→ `pipeline(Readable.fromWeb(res.body), createWriteStream(path))`。

---

### Q10. Buffer.alloc 和 Buffer.allocUnsafe 区别？Buffer 池是什么？

- **难度**: mid
- **标签**: Buffer

**答案**

- `Buffer.alloc(size)`：分配并清零，安全
- `Buffer.allocUnsafe(size)`：不清零，速度快，但可能包含旧内存数据（敏感信息泄漏风险）
- 小于 4 KiB 的 `allocUnsafe` / `Buffer.from` 会从预分配的 8 KiB 池中切片，减少分配开销

注意：池化 Buffer 的 `buf.buffer` 指向整个池，直接把 `buf.buffer` 传给其他 API 可能暴露其他数据，应使用 `byteOffset` 和 `length`。

**口述模板（30–60 秒）**

1. 一句话结论：alloc 安全清零，allocUnsafe 快但需立即填满。
2. 关键机制：小 Buffer 来自共享池。
3. 注意点：allocUnsafe 后必须完全覆盖再使用。
4. 可追问时补充：堆外内存体现在 external / arrayBuffers 指标。

**常见追问**

- Buffer 内存算在 heapUsed 中吗？→ 不算，计入 external / arrayBuffers。

---

## 三、模块系统

### Q11. CommonJS 模块加载原理是什么？

- **难度**: mid
- **标签**: 模块

**答案**

`require` 流程：

1. 解析路径（resolve）
2. 检查缓存 `Module._cache`，有则直接返回 `module.exports`
3. 创建 Module 对象并**先放入缓存**
4. 读取文件，用包装函数包裹后执行：

```js
(function (exports, require, module, __filename, __dirname) {
  // 模块代码
})
```

5. 返回 `module.exports`

循环依赖：A require B，B 又 require A 时，B 拿到的是 A **尚未执行完**的 `module.exports`（部分导出）。

`exports = {}` 无效，因为它只是改变了局部变量；必须 `module.exports = {}`。

**口述模板（30–60 秒）**

1. 一句话结论：解析、缓存、包装执行、返回 module.exports。
2. 关键机制：先缓存再执行，因此循环依赖得到部分导出。
3. 注意点：重新赋值 exports 无效。
4. 可追问时补充：模块缓存天然实现单例。

**常见追问**

- 如何热更新模块？→ 删除 `require.cache[path]` 后重新 require，但旧引用和副作用仍在，生产不推荐。

---

### Q12. ESM 与 CommonJS 如何互操作？什么是双包风险？

- **难度**: mid
- **标签**: 模块, ESM

**答案**

- **ESM 导入 CJS**：可以，`module.exports` 作为默认导出，部分命名导出可被静态分析识别
- **CJS 导入 ESM**：Node 22.12+ / 20.19+ 支持 `require(esm)`，前提是 ESM 图中无顶层 await；否则需要 `await import()`

条件导出发布双格式：

```json
{
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

**双包风险（Dual Package Hazard）**：同一进程中，部分代码通过 import 加载 ESM 版本，部分通过 require 加载 CJS 版本，导致同一个库存在两个实例（单例状态分裂、instanceof 失效）。

缓解：只发布 ESM（依赖 require(esm)）；或让两个入口共享同一份状态实现。

**口述模板（30–60 秒）**

1. 一句话结论：现代 Node 中 ESM 和 CJS 可双向互操作。
2. 关键机制：require(esm) 同步加载无顶层 await 的 ESM。
3. 注意点：双格式发布可能导致双实例。
4. 可追问时补充：库作者趋向只发布 ESM。

**常见追问**

- 顶层 await 为什么阻止 require(esm)？→ require 是同步的，无法等待异步模块求值。

---

## 四、框架与中间件

### Q13. Express、Koa、Fastify、NestJS 怎么选？

- **难度**: mid
- **标签**: 框架

**答案**

| 框架 | 特点 | 适用 |
|------|------|------|
| Express | 生态最大，线性中间件，Express 5 支持 async 错误 | 通用、存量项目 |
| Koa | 洋葱模型、async 中间件、极简 | 需要灵活定制的中间件链 |
| Fastify | 高性能、基于 JSON Schema 校验和序列化、插件封装 | 高吞吐 API |
| NestJS | 依赖注入、模块化、装饰器，类 Spring | 大型团队、企业后端 |
| Hono | Web 标准 API，跨 Node/Bun/Deno/Edge | Edge、多运行时 |

**口述模板（30–60 秒）**

1. 一句话结论：按团队规模、性能要求和运行环境选型。
2. 关键机制：Fastify 用 schema 编译加速序列化。
3. 注意点：框架性能差异通常小于数据库和业务逻辑耗时。
4. 可追问时补充：NestJS 的架构约束对大团队有价值。

**常见追问**

- Fastify 为什么快？→ 路由基于 radix tree，响应序列化按 schema 预编译，日志使用 pino。

---

### Q14. Koa 洋葱模型是什么？和 Express 有何不同？

- **难度**: mid
- **标签**: Koa, 中间件

**答案**

```js
app.use(async (ctx, next) => {
  const start = Date.now()
  await next()                     // 进入下游
  ctx.set('X-Response-Time', `${Date.now() - start}ms`) // 下游完成后回到这里
})
```

请求从外到内经过中间件，响应从内到外返回，像洋葱。

| | Express | Koa |
|--|---------|-----|
| 执行模型 | 线性，next 后通常不再处理 | 洋葱，await next 后可后置处理 |
| 异步 | 回调风格为主（5 已改善） | 原生 async |
| 上下文 | req / res | ctx |
| 内置 | 路由、静态文件等 | 几乎无内置 |

**口述模板（30–60 秒）**

1. 一句话结论：Koa 中间件 await next 后可以处理响应阶段。
2. 关键机制：compose 递归 dispatch 返回 Promise。
3. 注意点：忘记 await next 会导致下游异步未完成就返回。
4. 可追问时补充：手写 compose。

**常见追问**

- 洋葱模型适合什么？→ 计时、统一错误处理、响应包装、事务管理。

---

## 五、错误处理与稳定性

### Q15. Node 中如何全面处理错误？

- **难度**: mid
- **标签**: 错误处理

**答案**

分层处理：

1. **业务层**：async/await + try/catch，抛出带错误码的自定义错误
2. **框架层**：统一错误中间件，区分业务错误（4xx）和系统错误（5xx）
3. **事件层**：EventEmitter / Stream 的 `'error'` 事件必须监听
4. **进程层**：

```js
process.on('unhandledRejection', (reason) => {
  logger.error({ reason }, 'unhandled rejection')
  throw reason // 交给 uncaughtException 统一处理
})

process.on('uncaughtException', (error) => {
  logger.fatal({ error }, 'uncaught exception')
  // 进程状态可能已不一致，记录后优雅退出，由进程管理器重启
  shutdown(1)
})
```

原则：`uncaughtException` 后不应继续运行业务，应退出重启。

**口述模板（30–60 秒）**

1. 一句话结论：业务就近捕获，框架统一处理，进程级兜底后退出。
2. 关键机制：未捕获异常意味着状态未知。
3. 注意点：Node 15+ 默认未处理 rejection 会使进程崩溃。
4. 可追问时补充：错误分类和告警。

**常见追问**

- 为什么不在 uncaughtException 中继续服务？→ 可能处于资源泄漏、事务中断等不一致状态。

---

### Q16. 如何实现优雅关闭（Graceful Shutdown）？

- **难度**: mid
- **标签**: 稳定性, 部署

**答案**

收到 SIGTERM（K8s / Docker 停止容器）后：

1. 标记不健康，让负载均衡停止转发新请求（readiness 失败）
2. `server.close()` 停止接收新连接，等待存量请求完成
3. 关闭空闲 keep-alive 连接：`server.closeIdleConnections()`
4. 停止消费消息队列、定时任务
5. 关闭数据库连接池、Redis、MQ 连接
6. 设置超时，超时后强制退出

```js
process.once('SIGTERM', () => shutdown(0))
process.once('SIGINT', () => shutdown(0))
```

容器中注意：Node 作为 PID 1 时要确保信号能被处理；`npm start` 可能不转发信号，推荐直接 `node app.js` 或使用 tini。

**口述模板（30–60 秒）**

1. 一句话结论：停流量、等请求、关资源、超时强退。
2. 关键机制：server.close + 连接池关闭 + 超时兜底。
3. 注意点：keep-alive 连接会阻止 close 完成。
4. 可追问时补充：K8s preStop 和 terminationGracePeriodSeconds。

**常见追问**

- 为什么需要 preStop 睡几秒？→ 等待 Endpoint 摘除传播到各节点，避免关闭期间仍有新请求进入。

---

## 六、并发模型

### Q17. cluster 模块原理是什么？

- **难度**: mid
- **标签**: cluster

**答案**

- 主进程 fork 多个工作进程（通常等于 CPU 核数）
- 工作进程共享同一端口：默认由主进程接收连接再轮询分发（Windows 除外，使用操作系统分配）
- 各进程内存独立，通过 IPC 通信
- 工作进程崩溃可由主进程重启

```js
import cluster from 'node:cluster'
import { availableParallelism } from 'node:os'

if (cluster.isPrimary) {
  for (let i = 0; i < availableParallelism(); i++) cluster.fork()
  cluster.on('exit', () => cluster.fork())
} else {
  startServer()
}
```

容器化部署中，更推荐「一个容器一个进程」，由 K8s 副本数横向扩展。

**口述模板（30–60 秒）**

1. 一句话结论：cluster 多进程共享端口利用多核。
2. 关键机制：主进程分发连接，工作进程独立运行。
3. 注意点：进程间不共享内存，会话和缓存需外置。
4. 可追问时补充：容器场景用副本替代 cluster。

**常见追问**

- WebSocket 在 cluster 下要注意什么？→ 连接分布在不同进程，广播需要 Redis Pub/Sub 等外部通道。

---

### Q18. worker_threads 适合什么场景？

- **难度**: mid
- **标签**: worker_threads

**答案**

worker_threads 在同一进程中创建线程，每个线程有独立 V8 Isolate 和事件循环。

适合：CPU 密集型 JS 计算（图片处理、加解密、压缩、复杂计算、大 JSON 处理）。  
不适合：I/O 操作（Node 已异步）。

```js
import { Worker } from 'node:worker_threads'

function runTask(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(new URL('./task.js', import.meta.url), { workerData: data })
    worker.once('message', resolve)
    worker.once('error', reject)
  })
}
```

生产中使用固定大小线程池（Piscina），避免每个请求创建线程。

**口述模板（30–60 秒）**

1. 一句话结论：worker_threads 处理 CPU 密集任务。
2. 关键机制：独立 Isolate，通过消息或 SharedArrayBuffer 通信。
3. 注意点：线程创建有成本，要用池。
4. 可追问时补充：Transferable 零拷贝。

**常见追问**

- 和 cluster 区别？→ cluster 是多进程扩展请求处理能力；Worker 是进程内并行计算。

---

### Q19. child_process 的 spawn、exec、execFile、fork 区别？

- **难度**: mid
- **标签**: 子进程

**答案**

| 方法 | 特点 | 场景 |
|------|------|------|
| `spawn` | 流式输出，不经过 shell（默认） | 长时间运行、大输出 |
| `exec` | 经过 shell，缓冲输出 | 简单 shell 命令 |
| `execFile` | 不经过 shell，缓冲输出 | 运行可执行文件，更安全 |
| `fork` | 启动 Node 子进程，内置 IPC | Node 多进程通信 |

**口述模板（30–60 秒）**

1. 一句话结论：大输出用 spawn，简单命令用 execFile，Node 子进程用 fork。
2. 关键机制：exec 通过 shell，存在注入风险。
3. 注意点：子进程要处理退出码和错误。
4. 可追问时补充：子进程资源限制和超时。

**常见追问**

- 如何防止子进程变成僵尸或孤儿？→ 监听父进程退出时 kill 子进程，并设置 timeout。

---

### Q20. AsyncLocalStorage 有什么用？

- **难度**: mid
- **标签**: 异步上下文

**答案**

在异步调用链中传递上下文（类似线程局部变量），无需层层传参：

```js
import { AsyncLocalStorage } from 'node:async_hooks'
import { randomUUID } from 'node:crypto'

export const requestContext = new AsyncLocalStorage()

app.use((req, res, next) => {
  const traceId = req.headers['x-trace-id'] ?? randomUUID()
  requestContext.run({ traceId }, next)
})

export function log(message) {
  const ctx = requestContext.getStore()
  logger.info({ traceId: ctx?.traceId }, message)
}
```

场景：请求 ID / traceId、用户信息、事务上下文、多租户。OpenTelemetry 也基于它。

**口述模板（30–60 秒）**

1. 一句话结论：ALS 在异步链路中自动传递请求上下文。
2. 关键机制：上下文随异步资源创建而传播。
3. 注意点：在 run 之外创建的回调拿不到上下文。
4. 可追问时补充：Node 24 基于 AsyncContextFrame 性能提升。

**常见追问**

- 某些库回调中丢失上下文怎么办？→ 使用 `AsyncResource.bind` 或 `AsyncLocalStorage.bind` 绑定。

---

## 七、HTTP 与数据

### Q21. HTTP keep-alive 和连接池如何配置？

- **难度**: mid
- **标签**: HTTP, 性能

**答案**

**客户端**：服务间调用复用连接，避免每次 TCP/TLS 握手：

- Node 19+ 全局 http Agent 默认开启 keep-alive
- fetch（Undici）自带连接池
- 配置最大连接数、空闲超时

**服务端**：

- `server.keepAliveTimeout`（默认 5s）：空闲连接保持时间
- `server.headersTimeout`：接收请求头超时
- `server.requestTimeout`：整个请求超时

经典问题：负载均衡（如 ALB 60s）的空闲超时大于 Node 的 keepAliveTimeout（5s）时，Node 先关闭连接，LB 仍复用该连接发送请求，产生偶发 502。

解决：Node 的 keepAliveTimeout 设置为大于 LB 的空闲超时。

**口述模板（30–60 秒）**

1. 一句话结论：客户端复用连接，服务端超时要与 LB 配合。
2. 关键机制：keep-alive 减少握手开销。
3. 注意点：服务端 keepAliveTimeout 应大于上游 LB 空闲超时。
4. 可追问时补充：偶发 502 排查。

**常见追问**

- 连接池太小会怎样？→ 请求排队，表现为延迟升高；太大则压垮下游。

---

### Q22. 文件上传如何处理？

- **难度**: mid
- **标签**: 上传, Stream

**答案**

- 使用 busboy / multer / @fastify/multipart 流式解析 multipart
- 限制文件大小、数量、类型（校验魔数，不只看扩展名）
- 不要把整个文件读入内存，直接流式写入对象存储
- 文件名重新生成，防止路径穿越和覆盖
- 大文件推荐前端直传 OSS/S3（服务端签发临时凭证），Node 只处理元数据

**口述模板（30–60 秒）**

1. 一句话结论：流式解析、限制大小、直传对象存储。
2. 关键机制：multipart 边界解析成流。
3. 注意点：用户文件名不可信。
4. 可追问时补充：分片上传与断点续传。

**常见追问**

- 为什么推荐直传 OSS？→ 降低 Node 带宽和内存压力，服务端只做鉴权签名。

---

### Q23. Session 和 JWT 认证如何选择？

- **难度**: mid
- **标签**: 认证

**答案**

| | Session | JWT |
|--|---------|-----|
| 状态 | 服务端存储（Redis） | 无状态，信息在 token 中 |
| 吊销 | 删除 session 即可 | 难，需黑名单或短有效期 |
| 扩展 | 需共享存储 | 天然支持多实例 |
| 体积 | Cookie 小 | token 较大 |

实践：

- Web 应用：HttpOnly + Secure + SameSite Cookie 承载 session 或 token
- JWT 使用短有效期 access token + 可吊销的 refresh token（轮换）
- 校验签名算法白名单，防止 `alg: none` 攻击
- 敏感信息不放 JWT payload（只是 Base64 编码）

**口述模板（30–60 秒）**

1. 一句话结论：需要即时吊销用 Session，分布式无状态用 JWT。
2. 关键机制：JWT 依靠签名防篡改。
3. 注意点：JWT 不加密，且吊销困难。
4. 可追问时补充：refresh token 轮换。

**常见追问**

- token 存 localStorage 安全吗？→ XSS 可读取，优先 HttpOnly Cookie 并防 CSRF。

---

### Q24. 数据库连接池为什么重要？N+1 问题是什么？

- **难度**: mid
- **标签**: 数据库

**答案**

**连接池**：

- 建立数据库连接成本高，池化复用
- 池大小 ≈ 数据库可承受连接数 / 实例数，不是越大越好
- 设置获取连接超时，防止请求无限等待
- 多实例时注意总连接数超过数据库上限

**N+1**：

```js
const orders = await db.order.findMany()           // 1 次
for (const o of orders) {
  o.user = await db.user.findUnique({ where: { id: o.userId } }) // N 次
}
```

解决：JOIN / `include`、批量 `IN` 查询、DataLoader 合并请求。

**口述模板（30–60 秒）**

1. 一句话结论：连接池复用连接，N+1 是循环查询导致的性能问题。
2. 关键机制：批量查询或 DataLoader 合并。
3. 注意点：Serverless 场景连接数易爆，需要连接代理。
4. 可追问时补充：慢查询分析和索引。

**常见追问**

- Serverless 如何处理数据库连接？→ 使用 RDS Proxy、PgBouncer 或 HTTP 型数据库驱动。

---

### Q25. 常见缓存策略和问题有哪些？

- **难度**: mid
- **标签**: 缓存, Redis

**答案**

**Cache-Aside（旁路缓存）**：

1. 读：先查缓存，未命中查数据库并写缓存
2. 写：先更新数据库，再删除缓存

三大问题：

| 问题 | 原因 | 解决 |
|------|------|------|
| 缓存穿透 | 查询不存在的数据 | 缓存空值、布隆过滤器、参数校验 |
| 缓存击穿 | 热点 key 过期瞬间大量请求 | 互斥锁 / 单飞（single flight）、逻辑过期 |
| 缓存雪崩 | 大量 key 同时过期或 Redis 宕机 | 随机过期时间、多级缓存、熔断降级 |

**口述模板（30–60 秒）**

1. 一句话结论：Cache-Aside + 先更新库后删缓存。
2. 关键机制：删除而不是更新缓存，避免并发写导致脏数据。
3. 注意点：穿透、击穿、雪崩分别处理。
4. 可追问时补充：延迟双删与 binlog 订阅。

**常见追问**

- 为什么删除缓存而不是更新缓存？→ 并发更新时更新顺序不确定，删除让下次读取重新加载最新值。

---

### Q26. 如何实现接口限流？

- **难度**: mid
- **标签**: 限流

**答案**

算法：

- **固定窗口**：实现简单，窗口边界可能突发 2 倍流量
- **滑动窗口**：更平滑，Redis ZSET 实现
- **令牌桶**：允许一定突发，平均速率受控
- **漏桶**：恒定速率输出

多实例部署时，计数需放 Redis，并用 Lua 脚本保证原子性。

维度：IP、用户、API Key、接口。超限返回 429 和 `Retry-After`。

**口述模板（30–60 秒）**

1. 一句话结论：单机内存计数，分布式用 Redis + Lua。
2. 关键机制：令牌桶允许突发，漏桶平滑输出。
3. 注意点：网关层限流优先，应用层做细粒度补充。
4. 可追问时补充：手写令牌桶。

**常见追问**

- 为什么用 Lua？→ 读取、判断、写入在 Redis 中原子执行，避免并发竞态。

---

## 八、工程化

### Q27. 如何设计结构化日志和链路追踪？

- **难度**: mid
- **标签**: 可观测性

**答案**

- 使用 pino 输出 JSON 日志：时间、级别、服务名、版本、traceId、spanId、userId、耗时
- 请求入口生成或透传 traceId（W3C `traceparent` 头）
- AsyncLocalStorage 自动注入上下文
- 下游调用透传 trace 头
- 敏感字段脱敏（pino `redact`）
- 使用 OpenTelemetry 统一采集日志、指标、追踪

**口述模板（30–60 秒）**

1. 一句话结论：结构化日志 + traceId 贯穿全链路。
2. 关键机制：ALS 传递上下文，W3C Trace Context 跨服务透传。
3. 注意点：日志脱敏。
4. 可追问时补充：OpenTelemetry 自动埋点。

**常见追问**

- 日志太多影响性能怎么办？→ 生产使用 info 级别，异步写入，采样调试日志。

---

### Q28. PM2 还需要吗？

- **难度**: mid
- **标签**: 部署

**答案**

PM2 提供：进程守护、cluster 模式、日志管理、零停机重载。

适用：物理机 / 虚拟机直接部署 Node。

容器化 / K8s 中：

- 进程重启由容器编排负责
- 横向扩展靠副本数
- 日志输出 stdout
- 一般不需要 PM2，避免两层进程管理导致信号处理和资源统计混乱

**口述模板（30–60 秒）**

1. 一句话结论：虚机部署用 PM2，容器部署交给编排平台。
2. 关键机制：PM2 守护进程并支持 cluster。
3. 注意点：容器中多进程会让内存限制和健康检查复杂化。
4. 可追问时补充：pm2-runtime。

**常见追问**

- 容器里 CPU 核多怎么利用？→ 增加副本数，或单 Pod 内使用 cluster，但需合理设置资源限制。

---

### Q29. 如何防止请求无限等待？超时如何设计？

- **难度**: mid
- **标签**: 稳定性

**答案**

每一层都要有超时：

- 入站：`server.requestTimeout`、`headersTimeout`
- 出站 HTTP：`AbortSignal.timeout`
- 数据库：查询超时、获取连接超时
- Redis：命令超时
- 总体请求预算（deadline）向下游传递，下游超时 < 上游超时

```js
const signal = AbortSignal.any([req.signal, AbortSignal.timeout(3000)])
const res = await fetch(url, { signal })
```

超时后要真正取消下游操作，而不只是不等待。

**口述模板（30–60 秒）**

1. 一句话结论：所有外部调用都要有超时，且下游超时小于上游。
2. 关键机制：AbortSignal 传递取消。
3. 注意点：没有超时的请求会耗尽连接池。
4. 可追问时补充：重试预算和熔断。

**常见追问**

- 超时多少合适？→ 参考下游 P99 延迟加余量，并结合用户体验总预算。

---

### Q30. 如何处理 CPU 密集型请求而不影响其他请求？

- **难度**: mid
- **标签**: 性能

**答案**

方案由简到繁：

1. 算法优化、缓存结果
2. 分片执行，`await setImmediate()` 让出事件循环
3. worker_threads 线程池（Piscina）
4. 异步任务队列（BullMQ），接口立即返回任务 ID，前端轮询或推送结果
5. 拆成独立服务（Go / Rust / Python）

**口述模板（30–60 秒）**

1. 一句话结论：不让 CPU 任务在主线程同步执行。
2. 关键机制：分片、线程池、异步队列。
3. 注意点：线程池需要限制队列长度，避免积压。
4. 可追问时补充：WASM / 原生模块。

**常见追问**

- 图片缩略图生成怎么做？→ 上传后投递队列，Worker 进程用 sharp 处理，完成后回写结果。

---

### Q31. 如何保证接口幂等？

- **难度**: mid
- **标签**: 幂等

**答案**

幂等：同一请求执行多次结果相同。

- 天然幂等：GET、PUT（全量更新）、DELETE
- POST 需要设计：
  - 客户端生成幂等键（Idempotency-Key），服务端用 Redis `SET NX` 记录处理状态和结果
  - 数据库唯一索引（订单号）
  - 状态机校验（只有「待支付」才能变「已支付」）
  - 乐观锁（version 字段）

**口述模板（30–60 秒）**

1. 一句话结论：幂等键 + 唯一约束 + 状态机。
2. 关键机制：重复请求返回首次结果。
3. 注意点：重试、消息重复投递都依赖幂等。
4. 可追问时补充：分布式锁。

**常见追问**

- 幂等键记录多久？→ 覆盖客户端最大重试窗口，如 24 小时。

---

### Q32. Node 应用如何做健康检查？

- **难度**: mid
- **标签**: 部署

**答案**

- **Liveness（存活）**：进程是否卡死，失败则重启。只检查自身，不检查依赖，避免依赖故障导致级联重启
- **Readiness（就绪）**：是否能接收流量；启动预热未完成、正在关闭、关键依赖不可用时返回失败
- **Startup**：启动慢的应用避免被 liveness 误杀

```js
app.get('/healthz', (req, res) => res.send('ok'))
app.get('/readyz', (req, res) => {
  res.status(isShuttingDown || !db.isReady() ? 503 : 200).end()
})
```

**口述模板（30–60 秒）**

1. 一句话结论：存活看自身，就绪看能否服务。
2. 关键机制：K8s 根据探针重启或摘流量。
3. 注意点：liveness 不要依赖下游。
4. 可追问时补充：与优雅关闭结合。

**常见追问**

- 事件循环卡死时 liveness 能检测到吗？→ 能，探针请求无法响应会超时失败。
