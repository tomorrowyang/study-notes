# Node.js 手写题（13 道高频）

> 建议：先说边界（错误、超时、背压、资源释放、并发），再写代码。示例默认 ESM、Node 22+。

---

### H1. 实现 promisify

- **难度**: junior
- **标签**: 异步, 回调
- **面试官看什么**: 错误优先回调转换、this 保留、多返回值

**题干**

实现 `promisify(fn)`，把 `(…args, callback(err, result))` 风格函数转为返回 Promise 的函数。

**思路口述**

返回一个新函数，调用时返回 Promise；在参数末尾追加回调，err 存在则 reject，否则 resolve。用普通函数保留调用者的 this。

**参考实现**

```js
/**
 * 将错误优先回调风格的函数转为返回 Promise 的函数。
 */
function promisify(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn.call(this, ...args, (err, ...results) => {
        if (err) {
          reject(err)
          return
        }
        resolve(results.length > 1 ? results : results[0])
      })
    })
  }
}
```

**易错点**

- 用箭头函数返回，丢失方法调用时的 this
- 回调可能被多次调用（Promise 状态只改变一次，天然防御）
- 原生 `util.promisify` 还支持 `util.promisify.custom` 自定义实现（如 `setTimeout`）

---

### H2. 实现迷你 Koa 框架

- **难度**: senior
- **标签**: 中间件, http
- **面试官看什么**: 洋葱模型 compose、ctx 设计、统一错误处理、响应输出

**题干**

实现 `MiniKoa`，支持 `use(middleware)`、`listen(port)`，中间件签名 `async (ctx, next) => {}`，支持 body 为字符串、Buffer、Stream、对象（JSON）。

**思路口述**

compose 将中间件组合成洋葱模型；每个请求创建 ctx；执行中间件链后统一根据 body 类型输出响应。错误在顶层捕获，5xx 不向客户端暴露内部信息。

**参考实现**

```js
import http from 'node:http'
import { Readable } from 'node:stream'
import { pipeline } from 'node:stream/promises'

/**
 * 将中间件组合为洋葱模型执行函数。
 */
function compose(middlewares) {
  return function (ctx) {
    let lastIndex = -1

    function dispatch(index) {
      if (index <= lastIndex) {
        return Promise.reject(new Error('next() called multiple times'))
      }
      lastIndex = index

      const middleware = middlewares[index]
      if (!middleware) return Promise.resolve()

      try {
        return Promise.resolve(middleware(ctx, () => dispatch(index + 1)))
      } catch (error) {
        return Promise.reject(error)
      }
    }

    return dispatch(0)
  }
}

/**
 * 极简 Koa：中间件 + 上下文 + 统一响应。
 */
class MiniKoa {
  #middlewares = []

  use(middleware) {
    this.#middlewares.push(middleware)
    return this
  }

  listen(...args) {
    const handler = compose(this.#middlewares)
    const server = http.createServer((req, res) => this.#handle(handler, req, res))
    return server.listen(...args)
  }

  async #handle(handler, req, res) {
    const ctx = { req, res, state: {}, status: undefined, body: undefined }

    try {
      await handler(ctx)
    } catch (error) {
      ctx.status = error.status ?? 500
      // 5xx 不暴露内部错误信息给客户端
      ctx.body = { message: ctx.status >= 500 ? 'Internal Server Error' : error.message }
      console.error(error)
    }

    this.#respond(ctx)
  }

  #respond(ctx) {
    const { res, body } = ctx
    res.statusCode = ctx.status ?? (body === undefined ? 404 : 200)

    if (body === undefined || body === null) {
      res.end(http.STATUS_CODES[res.statusCode])
      return
    }

    if (typeof body === 'string' || Buffer.isBuffer(body)) {
      if (!res.hasHeader('Content-Type')) res.setHeader('Content-Type', 'text/plain; charset=utf-8')
      res.end(body)
      return
    }

    if (body instanceof Readable) {
      pipeline(body, res).catch((error) => console.error('stream response failed', error))
      return
    }

    res.setHeader('Content-Type', 'application/json; charset=utf-8')
    res.end(JSON.stringify(body))
  }
}
```

**易错点**

- 中间件忘记 await next，后置逻辑提前执行
- 错误信息原样返回，泄漏内部实现
- Stream 用 pipe 而不处理错误，客户端断开导致文件句柄泄漏
- 响应已发送后又写入（真实 Koa 用 `res.headersSent` 判断）

---

### H3. 实现支持路径参数的路由

- **难度**: mid
- **标签**: 路由, 正则
- **面试官看什么**: 路径参数解析、正则转义、未匹配处理

**题干**

为 H2 的 MiniKoa 实现 Router：

```js
router.get('/users/:id/orders/:orderId', (ctx) => {
  ctx.body = ctx.params // { id: '1', orderId: '2' }
})
app.use(router.routes())
```

**思路口述**

注册时把路径编译为正则：先转义正则特殊字符，再把 `:param` 替换为捕获组并记录参数名。匹配时按方法和正则查找，提取参数并 decode；未匹配交给下一个中间件。

**参考实现**

```js
/**
 * 简易路由：支持方法匹配和 :param 路径参数。
 */
class Router {
  #routes = []

  #add(method, path, handler) {
    const keys = []
    const escaped = path.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
    const pattern = escaped.replace(/:(\w+)/g, (_, key) => {
      keys.push(key)
      return '([^/]+)'
    })

    this.#routes.push({ method, regex: new RegExp(`^${pattern}/?$`), keys, handler })
    return this
  }

  get(path, handler) {
    return this.#add('GET', path, handler)
  }

  post(path, handler) {
    return this.#add('POST', path, handler)
  }

  routes() {
    return async (ctx, next) => {
      const { pathname } = new URL(ctx.req.url, 'http://localhost')

      for (const route of this.#routes) {
        if (route.method !== ctx.req.method) continue

        const match = route.regex.exec(pathname)
        if (!match) continue

        try {
          ctx.params = Object.fromEntries(
            route.keys.map((key, i) => [key, decodeURIComponent(match[i + 1])]),
          )
        } catch {
          // 非法的百分号编码（如 %E0%A4%A）会让 decodeURIComponent 抛错
          ctx.status = 400
          ctx.body = { message: 'Malformed URL' }
          return
        }

        return route.handler(ctx, next)
      }

      return next()
    }
  }
}
```

**易错点**

- 路径中的 `.` 等字符未转义，被当作正则通配符
- 未处理非法 URL 编码导致 500
- 线性匹配在路由很多时较慢，生产框架使用 radix tree（find-my-way）
- Node 24+ 可使用全局 `URLPattern` 做匹配

---

### H4. 实现并发受限的批量处理 mapConcurrent

- **难度**: mid
- **标签**: 并发, Promise
- **面试官看什么**: 并发上限、结果顺序、失败快速停止、大输入内存

**题干**

实现 `mapConcurrent(iterable, limit, mapper)`：最多同时执行 limit 个 mapper，结果顺序与输入一致；任一失败则停止派发新任务并抛错。适用于 10 万个 URL 抓取。

**思路口述**

启动 limit 个「工人」，共享同一个迭代器，每个工人循环取下一个元素处理。这样不会一次性为 10 万个元素创建 Promise。用 failed 标记在出错后停止派发。

**参考实现**

```js
/**
 * 以最多 limit 个并发执行 mapper，结果按输入顺序返回。
 */
async function mapConcurrent(iterable, limit, mapper) {
  const iterator = iterable[Symbol.iterator]()
  const results = []
  let index = 0
  let failed = false

  async function worker() {
    // 多个 worker 共享同一个同步迭代器；JS 单线程保证 next() 调用不会交错
    for (let step = iterator.next(); !step.done && !failed; step = iterator.next()) {
      const current = index++
      try {
        results[current] = await mapper(step.value, current)
      } catch (error) {
        failed = true
        throw error
      }
    }
  }

  await Promise.all(Array.from({ length: Math.max(1, limit) }, worker))
  return results
}
```

**易错点**

- `Promise.all(items.map(fn))` 无并发上限，瞬间打满下游和文件句柄
- 先创建全部 Promise 再限流，内存仍随输入线性增长
- 失败后仍继续派发任务
- 需要取消正在执行的任务时，传入 AbortSignal 给 mapper

---

### H5. 统计 10GB 访问日志中访问量 Top N 的 IP

- **难度**: mid
- **标签**: Stream, readline
- **面试官看什么**: 流式逐行读取、内存控制、边界处理

**题干**

日志每行格式：`<ip> - - [time] "GET /path" 200 ...`。在内存有限的机器上统计访问量前 N 的 IP。

**思路口述**

用 readline 逐行读取，内存只保存 IP 计数 Map。最后对 Map 排序取前 N（IP 数量远小于行数）。若 IP 种类也极多，可分片哈希到多个临时文件再分别统计（MapReduce 思路）。

**参考实现**

```js
import { createReadStream } from 'node:fs'
import { createInterface } from 'node:readline'

/**
 * 流式统计日志中访问次数最多的 N 个 IP。
 */
async function topIps(file, n = 10) {
  const counts = new Map()
  const rl = createInterface({
    input: createReadStream(file),
    crlfDelay: Infinity,
  })

  for await (const line of rl) {
    const spaceIndex = line.indexOf(' ')
    if (spaceIndex <= 0) continue

    const ip = line.slice(0, spaceIndex)
    counts.set(ip, (counts.get(ip) ?? 0) + 1)
  }

  return [...counts]
    .sort((a, b) => b[1] - a[1])
    .slice(0, n)
    .map(([ip, count]) => ({ ip, count }))
}
```

**易错点**

- `readFile` 整体读入导致 OOM
- 按 chunk 处理时没有处理跨 chunk 的半行
- 空行或格式异常行导致解析错误
- N 很小而 IP 很多时，可用大小为 N 的小顶堆替代全量排序

---

### H6. 用 Transform 流把 CSV 转为 NDJSON

- **难度**: senior
- **标签**: Stream, Transform
- **面试官看什么**: chunk 边界、多字节字符、objectMode、pipeline

**题干**

把一个几 GB 的 CSV（首行为表头）转换为 NDJSON（每行一个 JSON），内存保持平稳。

**思路口述**

拆成两个 Transform：LineSplitter 负责把字节流切成行（处理跨 chunk 半行和多字节字符），CsvToJson 负责把行转成 JSON 字符串。用 pipeline 串联读文件、转换、写文件，自动背压和错误清理。

**参考实现**

```js
import { createReadStream, createWriteStream } from 'node:fs'
import { Transform } from 'node:stream'
import { pipeline } from 'node:stream/promises'
import { StringDecoder } from 'node:string_decoder'

/**
 * 将字节流切分为逐行字符串（对象模式输出）。
 */
class LineSplitter extends Transform {
  #rest = ''
  // 多字节字符可能被 chunk 截断，StringDecoder 会暂存不完整字节
  #decoder = new StringDecoder('utf8')

  constructor() {
    super({ readableObjectMode: true })
  }

  _transform(chunk, _encoding, callback) {
    const lines = (this.#rest + this.#decoder.write(chunk)).split(/\r?\n/)
    this.#rest = lines.pop()
    for (const line of lines) {
      if (line) this.push(line)
    }
    callback()
  }

  _flush(callback) {
    const last = this.#rest + this.#decoder.end()
    if (last) this.push(last)
    callback()
  }
}

/**
 * 将 CSV 行转换为 NDJSON 行，首行作为表头。
 */
function csvToNdjson() {
  let headers = null

  return new Transform({
    objectMode: true,
    transform(line, _encoding, callback) {
      const cells = line.split(',')
      if (!headers) {
        headers = cells
        callback()
        return
      }

      const record = Object.fromEntries(headers.map((header, i) => [header, cells[i] ?? '']))
      callback(null, `${JSON.stringify(record)}\n`)
    },
  })
}

await pipeline(
  createReadStream('users.csv'),
  new LineSplitter(),
  csvToNdjson(),
  createWriteStream('users.ndjson'),
)
```

**易错点**

- 直接 `chunk.toString()` 截断中文字符产生乱码
- 忘记 `_flush` 导致最后一行丢失
- 简单 `split(',')` 不能处理带引号和逗号的字段，生产用 csv-parse
- 用 pipe 代替 pipeline，出错时流不销毁

---

### H7. 实现 HTTP 服务优雅关闭

- **难度**: mid
- **标签**: 稳定性, 信号
- **面试官看什么**: 停止接收、存量请求、资源释放、超时兜底、重复信号

**题干**

实现 `setupGracefulShutdown(server, options)`：收到 SIGTERM / SIGINT 后，标记关闭中、停止接收新请求、等待存量请求、关闭资源，超时强制退出。

**思路口述**

用标志位防止重复关闭，供 readiness 探针返回 503。server.close 停止接收新连接并等待存量请求；关闭空闲 keep-alive 连接加速完成。最后关闭数据库等资源。超时定时器 unref，避免它本身阻止进程正常退出。

**参考实现**

```js
/**
 * 为 HTTP server 注册优雅关闭逻辑，返回关闭状态查询函数。
 */
function setupGracefulShutdown(server, { timeout = 10_000, onShutdown = async () => {} } = {}) {
  let shuttingDown = false

  async function shutdown(signal) {
    if (shuttingDown) return
    shuttingDown = true
    console.info(`received ${signal}, shutting down`)

    const forceExitTimer = setTimeout(() => {
      console.error('graceful shutdown timed out, forcing exit')
      process.exit(1)
    }, timeout)
    forceExitTimer.unref()

    try {
      await new Promise((resolve, reject) => {
        server.close((error) => (error ? reject(error) : resolve()))
        server.closeIdleConnections()
      })
      await onShutdown()
      process.exitCode = 0
    } catch (error) {
      console.error('shutdown failed', error)
      process.exitCode = 1
    }
  }

  process.once('SIGTERM', () => shutdown('SIGTERM'))
  process.once('SIGINT', () => shutdown('SIGINT'))

  return { isShuttingDown: () => shuttingDown }
}

const { isShuttingDown } = setupGracefulShutdown(server, {
  onShutdown: async () => {
    await db.end()
    await redis.quit()
  },
})

// 关闭期间对仍在 keep-alive 连接上到达的请求返回 Connection: close，让客户端不再复用连接
app.use((req, res, next) => {
  if (isShuttingDown()) res.setHeader('Connection', 'close')
  next()
})

// readiness 探针在关闭期间返回 503，让负载均衡摘除实例
app.get('/readyz', (req, res) => res.status(isShuttingDown() ? 503 : 200).end())
```

**易错点**

- 直接 `process.exit()`，存量请求被中断
- 忽略 keep-alive 连接，server.close 迟迟不回调
- 没有超时兜底，进程卡住直到被 SIGKILL
- 容器中通过 `npm start` 启动，信号未转发给 Node 进程

---

### H8. 实现令牌桶限流中间件

- **难度**: senior
- **标签**: 限流
- **面试官看什么**: 令牌补充计算、突发容量、内存清理、分布式扩展

**题干**

实现按 IP 限流的中间件：每个 IP 容量 capacity，每秒补充 refillPerSecond 个令牌，超限返回 429 和 `Retry-After`。

**思路口述**

不需要定时器补充令牌，访问时根据距上次更新时间计算应补充的令牌（惰性计算）。Map 按 key 保存桶，定期清理已满的桶防止内存无限增长。多实例部署时状态放 Redis，用 Lua 原子执行。

**参考实现**

```js
/**
 * 基于内存的令牌桶限流中间件（单实例）。
 */
function createTokenBucketLimiter({
  capacity,
  refillPerSecond,
  keyOf = (req) => req.socket.remoteAddress,
}) {
  const buckets = new Map()

  // 桶已满等价于从未访问，删除以控制内存
  const cleanupTimer = setInterval(() => {
    const now = Date.now()
    for (const [key, bucket] of buckets) {
      const refilled = bucket.tokens + ((now - bucket.updatedAt) / 1000) * refillPerSecond
      if (refilled >= capacity) buckets.delete(key)
    }
  }, 60_000)
  cleanupTimer.unref()

  function take(key) {
    const now = Date.now()
    const bucket = buckets.get(key) ?? { tokens: capacity, updatedAt: now }
    const elapsedSeconds = (now - bucket.updatedAt) / 1000

    bucket.tokens = Math.min(capacity, bucket.tokens + elapsedSeconds * refillPerSecond)
    bucket.updatedAt = now

    const allowed = bucket.tokens >= 1
    if (allowed) bucket.tokens -= 1
    buckets.set(key, bucket)

    const retryAfter = allowed ? 0 : Math.ceil((1 - bucket.tokens) / refillPerSecond)
    return { allowed, retryAfter }
  }

  return (req, res, next) => {
    const { allowed, retryAfter } = take(keyOf(req))
    if (allowed) {
      next()
      return
    }
    res.statusCode = 429
    res.setHeader('Retry-After', String(retryAfter))
    res.end('Too Many Requests')
  }
}
```

分布式版本（Redis Lua，原子执行）：

```lua
-- KEYS[1]: 桶 key；ARGV: capacity, refillPerSecond, nowMs
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local data = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(data[1]) or capacity
local ts = tonumber(data[2]) or now

tokens = math.min(capacity, tokens + (now - ts) / 1000 * rate)
local allowed = 0
if tokens >= 1 then
  tokens = tokens - 1
  allowed = 1
end

redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('PEXPIRE', KEYS[1], math.ceil(capacity / rate * 1000))
return allowed
```

**易错点**

- 用 setInterval 给每个桶补充令牌，key 多时开销巨大
- Map 无清理，被大量随机 IP 打爆内存
- 反向代理后 `remoteAddress` 是代理 IP，需要可信地解析 `X-Forwarded-For`
- 多实例使用应用时间戳有时钟偏差，可在 Lua 中使用 `redis.call('TIME')`

---

### H9. 实现带 TTL 的 LRU 缓存和请求合并（Single Flight）

- **难度**: senior
- **标签**: 缓存, 并发
- **面试官看什么**: LRU 淘汰、过期处理、缓存击穿防护、失败不缓存

**题干**

1. 实现 `TtlLruCache({ max, ttl })`
2. 实现 `createCachedLoader(loader, cache)`：缓存未命中时，同一 key 的并发请求只调用一次 loader

**思路口述**

Map 保持插入顺序实现 LRU，get 命中时移到末尾；读取时检查过期（惰性删除）。请求合并用 inflight Map 保存进行中的 Promise，后续并发请求复用；完成后无论成功失败都删除 inflight，失败结果不写缓存。

**参考实现**

```js
/**
 * 带过期时间的 LRU 缓存。
 */
class TtlLruCache {
  #max
  #ttl
  #map = new Map()

  constructor({ max, ttl }) {
    this.#max = max
    this.#ttl = ttl
  }

  get(key) {
    const entry = this.#map.get(key)
    if (!entry) return undefined

    if (entry.expiresAt <= Date.now()) {
      this.#map.delete(key)
      return undefined
    }

    this.#map.delete(key)
    this.#map.set(key, entry)
    return entry.value
  }

  set(key, value, ttl = this.#ttl) {
    this.#map.delete(key)
    this.#map.set(key, { value, expiresAt: Date.now() + ttl })

    if (this.#map.size > this.#max) {
      this.#map.delete(this.#map.keys().next().value)
    }
  }
}

/**
 * 包装 loader：先查缓存，未命中时合并同 key 的并发请求。
 */
function createCachedLoader(loader, cache) {
  const inflight = new Map()

  return function load(key) {
    const cached = cache.get(key)
    if (cached !== undefined) return Promise.resolve(cached)

    const pending = inflight.get(key)
    if (pending) return pending

    const promise = (async () => {
      try {
        const value = await loader(key)
        cache.set(key, value)
        return value
      } finally {
        inflight.delete(key)
      }
    })()

    inflight.set(key, promise)
    return promise
  }
}
```

**易错点**

- 热点 key 过期瞬间大量请求同时回源（击穿）
- 失败结果也写入缓存，导致错误被放大
- 值为 undefined 时无法区分「未命中」，需要存包装对象
- 多实例下进程内缓存不一致，需要 Redis 或失效广播

---

### H10. 实现熔断器 CircuitBreaker

- **难度**: senior
- **标签**: 稳定性, 熔断
- **面试官看什么**: 三状态转换、半开试探、超时与取消

**题干**

实现 `CircuitBreaker(action, options)`：

- 连续失败达到阈值 → OPEN，直接快速失败
- OPEN 持续 resetTimeout 后 → HALF_OPEN，只放行一个试探请求
- 试探成功 → CLOSED；失败 → OPEN
- 每次调用带超时

**思路口述**

状态机维护 state、失败次数和打开时间。调用前根据状态决定放行或拒绝；调用时传入超时 signal；根据结果更新状态。半开状态用标志位保证同时只有一个试探请求。

**参考实现**

```js
/**
 * 简易熔断器：CLOSED → OPEN → HALF_OPEN 状态机。
 * action 的第一个参数为超时 AbortSignal。
 */
class CircuitBreaker {
  #action
  #failureThreshold
  #resetTimeout
  #timeout
  #state = 'CLOSED'
  #failures = 0
  #openedAt = 0
  #probing = false

  constructor(action, { failureThreshold = 5, resetTimeout = 10_000, timeout = 3000 } = {}) {
    this.#action = action
    this.#failureThreshold = failureThreshold
    this.#resetTimeout = resetTimeout
    this.#timeout = timeout
  }

  get state() {
    return this.#state
  }

  async fire(...args) {
    if (this.#state === 'OPEN') {
      if (Date.now() - this.#openedAt < this.#resetTimeout) {
        throw new Error('Circuit is open')
      }
      this.#state = 'HALF_OPEN'
    }

    const isProbe = this.#state === 'HALF_OPEN'
    if (isProbe) {
      // 半开时只允许一个试探请求，避免恢复中的下游被瞬间打满
      if (this.#probing) throw new Error('Circuit is half-open')
      this.#probing = true
    }

    try {
      const result = await this.#action(AbortSignal.timeout(this.#timeout), ...args)
      this.#failures = 0
      this.#state = 'CLOSED'
      return result
    } catch (error) {
      this.#failures++
      if (isProbe || this.#failures >= this.#failureThreshold) {
        this.#state = 'OPEN'
        this.#openedAt = Date.now()
      }
      throw error
    } finally {
      if (isProbe) this.#probing = false
    }
  }
}

// 使用
const breaker = new CircuitBreaker((signal, id) => fetch(`${API}/users/${id}`, { signal }))
```

**易错点**

- 4xx 业务错误也计入失败，导致误熔断（应只统计超时、5xx、网络错误）
- 半开状态放行大量请求
- 只计连续失败次数，流量大时更适合滑动窗口失败率（opossum）
- 熔断时没有降级逻辑，直接把错误抛给用户

---

### H11. 实现 worker_threads 线程池

- **难度**: senior
- **标签**: worker_threads, 并发
- **面试官看什么**: 任务分发、有界队列、异常重建、关闭

**题干**

实现 `WorkerPool(file, { size, maxQueue })`，`run(data)` 返回 Promise；worker 崩溃时拒绝当前任务并自动补充新线程。

**思路口述**

预先创建 size 个 worker 放入空闲列表；任务入队后分发给空闲 worker，用 Map 记录 worker 正在执行的任务。message 时 resolve 并归还 worker；error / exit 时 reject 当前任务并重建线程。队列有上限，满了直接拒绝，形成背压。

**参考实现**

```js
import { Worker } from 'node:worker_threads'
import { availableParallelism } from 'node:os'

/**
 * 固定大小的 Worker 线程池，带有界队列与崩溃自愈。
 */
class WorkerPool {
  #file
  #maxQueue
  #idle = []
  #queue = []
  #tasks = new Map()
  #workers = new Set()
  #closed = false

  constructor(file, { size = Math.max(1, availableParallelism() - 1), maxQueue = 1000 } = {}) {
    this.#file = file
    this.#maxQueue = maxQueue
    for (let i = 0; i < size; i++) this.#spawn()
  }

  run(data) {
    return new Promise((resolve, reject) => {
      if (this.#closed) {
        reject(new Error('Worker pool is closed'))
        return
      }
      if (this.#queue.length >= this.#maxQueue) {
        reject(new Error('Worker pool queue is full'))
        return
      }
      this.#queue.push({ data, resolve, reject })
      this.#dispatch()
    })
  }

  async close() {
    this.#closed = true
    for (const task of this.#queue) task.reject(new Error('Worker pool is closed'))
    this.#queue = []
    await Promise.all([...this.#workers].map((worker) => worker.terminate()))
  }

  #spawn() {
    const worker = new Worker(this.#file)

    worker.on('message', (result) => {
      this.#settle(worker, (task) => task.resolve(result))
      this.#idle.push(worker)
      this.#dispatch()
    })

    worker.on('error', (error) => {
      this.#settle(worker, (task) => task.reject(error))
    })

    worker.on('exit', (code) => {
      this.#settle(worker, (task) => task.reject(new Error(`Worker exited with code ${code}`)))
      this.#workers.delete(worker)
      this.#idle = this.#idle.filter((item) => item !== worker)
      if (!this.#closed) {
        this.#spawn()
        this.#dispatch()
      }
    })

    this.#workers.add(worker)
    this.#idle.push(worker)
  }

  #settle(worker, callback) {
    const task = this.#tasks.get(worker)
    if (!task) return
    this.#tasks.delete(worker)
    callback(task)
  }

  #dispatch() {
    while (this.#idle.length > 0 && this.#queue.length > 0) {
      const worker = this.#idle.pop()
      const task = this.#queue.shift()
      this.#tasks.set(worker, task)
      worker.postMessage(task.data)
    }
  }
}
```

worker 文件：

```js
import { parentPort } from 'node:worker_threads'

parentPort.on('message', (data) => {
  parentPort.postMessage(heavyCompute(data))
})
```

**易错点**

- 每个请求 new Worker，线程创建开销远大于任务本身
- 无界队列让内存持续增长
- worker 崩溃后任务 Promise 永远 pending
- 大数据复制开销高，可用 Transferable 传 ArrayBuffer

---

### H12. 用 AsyncLocalStorage 实现带请求 ID 的日志

- **难度**: mid
- **标签**: 异步上下文, 日志
- **面试官看什么**: 上下文传播、请求 ID 透传与校验、无侵入日志

**题干**

实现中间件和 logger：每个请求生成或透传 `x-request-id`，业务任意位置调用 `logger.info()` 都自动带上该 ID，且无需传参。

**思路口述**

中间件中用 `als.run(context, next)` 包裹后续处理，logger 内部通过 `getStore()` 读取上下文。来自客户端的请求 ID 不可信，校验格式和长度后才透传。

**参考实现**

```js
import { AsyncLocalStorage } from 'node:async_hooks'
import { randomUUID } from 'node:crypto'

const requestContext = new AsyncLocalStorage()
const REQUEST_ID_PATTERN = /^[\w-]{1,128}$/

/**
 * 为每个请求建立上下文，并透传或生成请求 ID。
 */
function requestContextMiddleware(req, res, next) {
  const incoming = req.headers['x-request-id']
  // 客户端传入的值不可信，防止日志注入和超长字符串
  const requestId =
    typeof incoming === 'string' && REQUEST_ID_PATTERN.test(incoming) ? incoming : randomUUID()

  res.setHeader('x-request-id', requestId)
  requestContext.run({ requestId, startAt: performance.now() }, next)
}

function write(level, message, fields = {}) {
  const ctx = requestContext.getStore()
  const entry = {
    time: new Date().toISOString(),
    level,
    message,
    requestId: ctx?.requestId,
    ...fields,
  }
  process.stdout.write(`${JSON.stringify(entry)}\n`)
}

/** 自动附带请求上下文的结构化日志。 */
const logger = {
  info: (message, fields) => write('info', message, fields),
  warn: (message, fields) => write('warn', message, fields),
  error: (message, fields) => write('error', message, fields),
}

/**
 * 调用下游时透传请求 ID。
 */
function fetchWithContext(url, init = {}) {
  const ctx = requestContext.getStore()
  const headers = new Headers(init.headers)
  if (ctx) headers.set('x-request-id', ctx.requestId)
  return fetch(url, { ...init, headers })
}
```

**易错点**

- 使用 `enterWith` 导致上下文泄漏到其他请求
- 直接信任客户端请求 ID
- 自定义连接池 / 队列中回调丢失上下文，需要 `AsyncLocalStorage.bind`
- 生产用 pino + OpenTelemetry，而不是手写 logger

---

### H13. 实现安全的静态文件服务

- **难度**: mid
- **标签**: 安全, Stream
- **面试官看什么**: 路径穿越防护、流式响应、缓存头、错误处理

**题干**

实现 `createStaticHandler(rootDir)`：返回 http 请求处理函数，提供 rootDir 下的文件，目录返回 index.html，阻止访问 rootDir 外的文件，支持 `If-Modified-Since` 返回 304。

**思路口述**

URL 先解析 pathname 并 decode（非法编码返回 400，拒绝空字节）。用 `path.resolve` 规范化后校验仍在 root 下（以 `root + sep` 开头），防止 `../` 和编码绕过。stat 获取文件信息，设置 Content-Type、Content-Length、Last-Modified；用 pipeline 流式返回。

**参考实现**

```js
import { createReadStream } from 'node:fs'
import { stat } from 'node:fs/promises'
import path from 'node:path'
import { pipeline } from 'node:stream/promises'

const MIME_TYPES = {
  '.html': 'text/html; charset=utf-8',
  '.js': 'text/javascript; charset=utf-8',
  '.css': 'text/css; charset=utf-8',
  '.json': 'application/json; charset=utf-8',
  '.png': 'image/png',
  '.svg': 'image/svg+xml',
}

function sendStatus(res, statusCode) {
  res.statusCode = statusCode
  res.end()
}

/**
 * 创建限定在 rootDir 内的静态文件处理函数。
 */
function createStaticHandler(rootDir) {
  const root = path.resolve(rootDir)

  return async function handle(req, res) {
    let pathname
    try {
      pathname = decodeURIComponent(new URL(req.url, 'http://localhost').pathname)
    } catch {
      sendStatus(res, 400)
      return
    }
    if (pathname.includes('\0')) {
      sendStatus(res, 400)
      return
    }

    let filePath = path.resolve(root, `.${pathname}`)
    // 规范化后再校验前缀，阻止 ../ 和 %2e%2e 等编码绕过
    if (filePath !== root && !filePath.startsWith(root + path.sep)) {
      sendStatus(res, 403)
      return
    }

    let stats
    try {
      stats = await stat(filePath)
      if (stats.isDirectory()) {
        filePath = path.join(filePath, 'index.html')
        stats = await stat(filePath)
      }
    } catch {
      sendStatus(res, 404)
      return
    }

    const lastModified = stats.mtime.toUTCString()
    const ifModifiedSince = req.headers['if-modified-since']
    // HTTP 日期精度为秒，比较前要截掉毫秒
    if (ifModifiedSince && Math.floor(stats.mtimeMs / 1000) * 1000 <= Date.parse(ifModifiedSince)) {
      sendStatus(res, 304)
      return
    }

    res.setHeader('Content-Type', MIME_TYPES[path.extname(filePath)] ?? 'application/octet-stream')
    res.setHeader('Content-Length', stats.size)
    res.setHeader('Last-Modified', lastModified)

    try {
      await pipeline(createReadStream(filePath), res)
    } catch (error) {
      // 客户端中途断开属于正常情况，其余错误记录日志
      if (error.code !== 'ERR_STREAM_PREMATURE_CLOSE') console.error(error)
    }
  }
}
```

**易错点**

- 直接 `path.join(root, req.url)`，被 `../` 穿越
- 用 `startsWith(root)` 而不加分隔符，`/srv/app-secret` 会被误判在 `/srv/app` 内
- 符号链接可指向 root 外，需要 `realpath` 后再校验
- 生产中静态资源交给 Nginx / CDN，Node 只做兜底
