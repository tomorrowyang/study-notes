# 计算机网络手写题

> 目标：用 TypeScript / Node.js 实现网络相关的常见题目，体现对协议细节（消息边界、超时、重试、幂等、背压）的理解。

## 使用方式

- **协议实现**：H1 解析 HTTP/1.1 请求、H2 用 `net` 实现 HTTP 服务器、H3 长度前缀拆包
- **请求控制**：H4 超时 + 重试 + 退避的 fetch、H5 并发控制、H6 请求去重与令牌单飞刷新
- **实时通信**：H7 WebSocket 客户端（心跳 + 重连）、H8 SSE 服务端与客户端
- **HTTP 机制**：H9 解析 Cache-Control 并判断新鲜度、H10 断点续传下载
- **基础设施**：H11 令牌桶限流、H12 简易反向代理

先口述思路，再写代码，最后主动说易错点。

---

### H1. 解析 HTTP/1.1 请求报文

- **难度**: 中级
- **标签**: HTTP, 解析
- **面试官看什么**: 报文结构、头部大小写不敏感、Content-Length、不完整数据的处理

**题干**

实现 `parseRequest(buffer)`：解析请求行、头部和正文；数据不完整时返回 `null`（等待更多数据）；返回解析结果和消耗的字节数（支持一个缓冲区中有多个请求）。

**思路口述**

先找头部结束标记 `\r\n\r\n`，找不到说明头部还没收完。解析请求行和头部，头部名转为小写。根据 `Content-Length` 判断正文是否完整。为了避免请求走私，同时出现 `Transfer-Encoding` 和 `Content-Length` 时直接拒绝；这里简化为不支持分块编码。

**参考实现**

```ts
/** 解析后的 HTTP 请求 */
export interface ParsedRequest {
  method: string
  path: string
  version: string
  headers: Map<string, string>
  body: Buffer
}

const HEADER_END = Buffer.from('\r\n\r\n')
const MAX_HEADER_SIZE = 16 * 1024

/**
 * 从缓冲区解析一个 HTTP/1.1 请求。
 * 返回 null 表示数据不完整；consumed 表示本次消耗的字节数，剩余部分属于下一个请求。
 */
export function parseRequest(buffer: Buffer): { request: ParsedRequest; consumed: number } | null {
  const headerEnd = buffer.indexOf(HEADER_END)
  if (headerEnd === -1) {
    // 防止恶意客户端发送无限长的头部占用内存
    if (buffer.length > MAX_HEADER_SIZE) throw new Error('431 Request Header Fields Too Large')
    return null
  }

  const [requestLine, ...headerLines] = buffer.subarray(0, headerEnd).toString('latin1').split('\r\n')
  const [method, path, version] = requestLine.split(' ')
  if (!method || !path || !version?.startsWith('HTTP/1.')) throw new Error('400 Bad Request')

  const headers = new Map<string, string>()
  for (const line of headerLines) {
    const colon = line.indexOf(':')
    if (colon <= 0) throw new Error('400 Bad Request')
    const name = line.slice(0, colon).trim().toLowerCase()
    const value = line.slice(colon + 1).trim()
    // 重复的 Content-Length 值不一致是请求走私的常见手法
    if (name === 'content-length' && headers.has(name) && headers.get(name) !== value) {
      throw new Error('400 Bad Request')
    }
    headers.set(name, value)
  }

  if (headers.has('transfer-encoding')) {
    // 简化实现：不支持分块编码；同时出现两者时必须拒绝，避免请求走私
    throw new Error('501 Not Implemented')
  }

  const lengthHeader = headers.get('content-length') ?? '0'
  if (!/^\d+$/.test(lengthHeader)) throw new Error('400 Bad Request')
  const contentLength = Number(lengthHeader)

  const bodyStart = headerEnd + HEADER_END.length
  if (buffer.length < bodyStart + contentLength) return null

  return {
    request: {
      method,
      path,
      version,
      headers,
      body: buffer.subarray(bodyStart, bodyStart + contentLength),
    },
    consumed: bodyStart + contentLength,
  }
}
```

**易错点**

- 用字符串长度而不是字节长度计算正文（中文字符占多个字节）
- 头部名区分大小写
- 没有限制头部大小
- 不处理一个缓冲区中有多个请求（管线化或粘包）
- `Content-Length` 不校验格式（负数、非数字）

---

### H2. 用 `net` 模块实现一个支持 Keep-Alive 的 HTTP 服务器

- **难度**: 中级
- **标签**: TCP, HTTP
- **面试官看什么**: 字节流的累积和拆分、连接复用、空闲超时

**题干**

基于 H1 的解析函数，用 `net.createServer` 实现服务器：支持一个连接上处理多个请求，空闲 5 秒后关闭连接。

**参考实现**

```ts
import net from 'node:net'
import { parseRequest, type ParsedRequest } from './parse-request'

const IDLE_TIMEOUT_MS = 5_000

function buildResponse(status: number, body: string, keepAlive: boolean): Buffer {
  const bodyBuffer = Buffer.from(body, 'utf8')
  const head = [
    `HTTP/1.1 ${status} ${status === 200 ? 'OK' : 'Error'}`,
    'Content-Type: text/plain; charset=utf-8',
    `Content-Length: ${bodyBuffer.length}`,
    `Connection: ${keepAlive ? 'keep-alive' : 'close'}`,
    '',
    '',
  ].join('\r\n')
  return Buffer.concat([Buffer.from(head, 'latin1'), bodyBuffer])
}

function handle(request: ParsedRequest): { status: number; body: string } {
  if (request.method === 'GET' && request.path === '/') return { status: 200, body: '你好' }
  return { status: 404, body: 'Not Found' }
}

const server = net.createServer((socket) => {
  let pending = Buffer.alloc(0)
  socket.setTimeout(IDLE_TIMEOUT_MS)
  socket.on('timeout', () => socket.end())

  socket.on('data', (chunk) => {
    pending = Buffer.concat([pending, chunk])
    try {
      // TCP 是字节流：一次 data 可能包含半个请求，也可能包含多个请求
      while (true) {
        const result = parseRequest(pending)
        if (!result) break
        pending = pending.subarray(result.consumed)

        const { request } = result
        const keepAlive = request.headers.get('connection')?.toLowerCase() !== 'close'
        const { status, body } = handle(request)
        socket.write(buildResponse(status, body, keepAlive))
        if (!keepAlive) {
          socket.end()
          return
        }
      }
    } catch (error) {
      const message = error instanceof Error ? error.message : '400 Bad Request'
      socket.end(buildResponse(Number.parseInt(message, 10) || 400, message, false))
    }
  })

  socket.on('error', () => socket.destroy())
})

server.listen(8080)
```

**易错点**

- 假设一次 `data` 事件就是一个完整的请求
- 处理完一个请求就关闭连接（没有复用）
- 没有空闲超时，空闲连接永远占用资源
- 没有处理 socket 的 `error` 事件，导致进程崩溃
- 写入时没有考虑背压（大响应时应该检查 `write` 的返回值并等待 `drain`）

---

### H3. 长度前缀协议的编码与拆包

- **难度**: 中级
- **标签**: TCP, 粘包
- **面试官看什么**: 字节流到消息的转换、缓冲区管理、最大长度限制

**题干**

自定义协议：每条消息 = 4 字节大端长度 + JSON 正文。实现编码函数和一个解码器（接收任意切分的数据块，输出完整的消息）。

**参考实现**

```ts
const HEADER_SIZE = 4
const MAX_MESSAGE_SIZE = 1024 * 1024

/** 把消息编码为「4 字节长度 + JSON」 */
export function encode(message: unknown): Buffer {
  const body = Buffer.from(JSON.stringify(message), 'utf8')
  const header = Buffer.alloc(HEADER_SIZE)
  header.writeUInt32BE(body.length, 0)
  return Buffer.concat([header, body])
}

/** 流式解码器：可以接收任意切分的数据块 */
export class FrameDecoder {
  private buffer = Buffer.alloc(0)

  /** 追加数据，返回本次可以解析出的所有完整消息 */
  push(chunk: Buffer): unknown[] {
    this.buffer = Buffer.concat([this.buffer, chunk])
    const messages: unknown[] = []

    while (this.buffer.length >= HEADER_SIZE) {
      const length = this.buffer.readUInt32BE(0)
      // 先校验长度再等待数据：否则恶意的超大长度会让缓冲区无限增长
      if (length > MAX_MESSAGE_SIZE) throw new Error(`消息过大: ${length}`)
      if (this.buffer.length < HEADER_SIZE + length) break

      const body = this.buffer.subarray(HEADER_SIZE, HEADER_SIZE + length)
      messages.push(JSON.parse(body.toString('utf8')))
      this.buffer = this.buffer.subarray(HEADER_SIZE + length)
    }

    return messages
  }
}
```

```ts
// 测试：把两条消息拼接后按任意位置切分
const data = Buffer.concat([encode({ id: 1 }), encode({ text: '你好' })])
const decoder = new FrameDecoder()
console.log(decoder.push(data.subarray(0, 3)))   // []，长度头还没收完
console.log(decoder.push(data.subarray(3, 15)))  // [{ id: 1 }]
console.log(decoder.push(data.subarray(15)))     // [{ text: '你好' }]
```

**易错点**

- 用字符串的长度而不是字节长度
- 不限制最大长度
- 在 UTF-8 多字节字符中间切分后，对半个数据块直接 `toString`（应该在拿到完整消息后再解码）
- 每次 `Buffer.concat` 在高吞吐时开销大，可以使用块列表或环形缓冲区优化

---

### H4. 带超时、重试和指数退避的 fetch

- **难度**: 中级
- **标签**: 请求, 重试
- **面试官看什么**: 只重试幂等请求、退避加抖动、Retry-After、外部取消

**题干**

实现 `fetchWithRetry(url, options)`：每次尝试有超时；网络错误、5xx、429 时重试；指数退避加随机抖动；遵守 `Retry-After`；支持调用方取消。

**参考实现**

```ts
/** 重试选项 */
export interface RetryOptions extends RequestInit {
  retries?: number
  timeoutMs?: number
  baseDelayMs?: number
  maxDelayMs?: number
}

const IDEMPOTENT_METHODS = new Set(['GET', 'HEAD', 'OPTIONS', 'PUT', 'DELETE'])
const RETRYABLE_STATUS = new Set([408, 429, 500, 502, 503, 504])

function sleep(ms: number, signal?: AbortSignal | null): Promise<void> {
  return new Promise((resolve, reject) => {
    if (signal?.aborted) return reject(signal.reason)
    const timer = setTimeout(resolve, ms)
    signal?.addEventListener('abort', () => {
      clearTimeout(timer)
      reject(signal.reason)
    }, { once: true })
  })
}

/** 解析 Retry-After：可能是秒数，也可能是 HTTP 日期 */
function parseRetryAfter(value: string | null): number | undefined {
  if (!value) return undefined
  const seconds = Number(value)
  if (Number.isFinite(seconds)) return seconds * 1000
  const date = Date.parse(value)
  return Number.isNaN(date) ? undefined : Math.max(0, date - Date.now())
}

/** 带超时和重试的 fetch；非幂等请求需要携带 Idempotency-Key 才会重试 */
export async function fetchWithRetry(url: string, options: RetryOptions = {}): Promise<Response> {
  const { retries = 3, timeoutMs = 10_000, baseDelayMs = 300, maxDelayMs = 10_000, signal, ...init } = options
  const method = (init.method ?? 'GET').toUpperCase()
  const hasIdempotencyKey = new Headers(init.headers).has('Idempotency-Key')
  const canRetry = IDEMPOTENT_METHODS.has(method) || hasIdempotencyKey

  for (let attempt = 0; ; attempt++) {
    // 每次尝试独立超时，同时响应调用方的取消
    const attemptSignal = signal
      ? AbortSignal.any([signal, AbortSignal.timeout(timeoutMs)])
      : AbortSignal.timeout(timeoutMs)

    let retryAfterMs: number | undefined
    try {
      const response = await fetch(url, { ...init, signal: attemptSignal })
      if (!RETRYABLE_STATUS.has(response.status) || !canRetry || attempt >= retries) return response
      retryAfterMs = parseRetryAfter(response.headers.get('Retry-After'))
      // 释放响应体，避免连接无法复用
      await response.body?.cancel()
    } catch (error) {
      // 调用方主动取消时立即停止，不再重试
      if (signal?.aborted) throw error
      if (!canRetry || attempt >= retries) throw error
    }

    // 全抖动：在 [0, 指数上限] 之间随机，避免大量客户端同时重试
    const exponential = Math.min(maxDelayMs, baseDelayMs * 2 ** attempt)
    const delay = retryAfterMs ?? Math.random() * exponential
    await sleep(Math.min(delay, maxDelayMs), signal)
  }
}
```

**易错点**

- 对 POST 无条件重试（可能重复下单）
- 对 4xx（除 408、429 外）重试
- 退避没有抖动，造成重试风暴
- 调用方取消后仍然重试
- 重试前没有消费或取消响应体
- 请求体是流（`ReadableStream`）时无法重复发送

---

### H5. 并发控制：限制同时进行的请求数

- **难度**: 中级
- **标签**: 并发
- **面试官看什么**: 队列、完成一个补充一个、结果顺序、错误处理

**题干**

实现 `pLimit(concurrency)`，返回一个包装函数；以及 `mapWithConcurrency(items, concurrency, fn)`，结果按输入顺序返回，单个失败不影响其他任务。

**参考实现**

```ts
/** 创建并发限制器：同一时间最多执行 concurrency 个任务 */
export function pLimit(concurrency: number) {
  if (!Number.isInteger(concurrency) || concurrency < 1) throw new RangeError('concurrency 必须是正整数')
  let active = 0
  const queue: Array<() => void> = []

  const next = () => {
    if (active >= concurrency) return
    const run = queue.shift()
    if (run) run()
  }

  return function limit<T>(task: () => Promise<T>): Promise<T> {
    return new Promise<T>((resolve, reject) => {
      queue.push(() => {
        active++
        // 用 then 包装：task 同步抛错时也会变成 rejected，而不是让 active 永远不减
        Promise.resolve()
          .then(task)
          .then(resolve, reject)
          .finally(() => {
            active--
            next()
          })
      })
      next()
    })
  }
}

/** 按并发上限处理列表，结果顺序与输入一致 */
export async function mapWithConcurrency<T, R>(
  items: readonly T[],
  concurrency: number,
  fn: (item: T, index: number) => Promise<R>,
): Promise<PromiseSettledResult<R>[]> {
  const limit = pLimit(concurrency)
  return Promise.allSettled(items.map((item, index) => limit(() => fn(item, index))))
}
```

```ts
const results = await mapWithConcurrency(urls, 4, (url) => fetch(url).then((r) => r.json()))
```

**易错点**

- 分批执行（每批等全部完成再开始下一批），而不是完成一个补充一个
- 结果按完成顺序而不是输入顺序
- 一个失败导致整体中断（`Promise.all`）
- 直接调用 `task()`：任务同步抛错时 `active` 不会减少，限制器逐渐「卡死」

---

### H6. 请求去重与令牌单飞刷新

- **难度**: 高级
- **标签**: 并发, 认证
- **面试官看什么**: 共享 Promise、失败后清理、避免死循环

**题干**

1. 相同 key 的并发 GET 请求只发送一次
2. 多个请求同时返回 401 时，只刷新一次令牌，其他请求等待刷新完成后重放；刷新失败时全部失败并跳转登录

**参考实现**

```ts
const inflight = new Map<string, Promise<unknown>>()

/** 并发去重：相同 key 的请求在完成前共享同一个 Promise */
export function dedupe<T>(key: string, task: () => Promise<T>): Promise<T> {
  const existing = inflight.get(key)
  if (existing) return existing as Promise<T>
  const promise = task().finally(() => inflight.delete(key))
  inflight.set(key, promise)
  return promise
}

let accessToken: string | null = null
let refreshPromise: Promise<string> | null = null

/** 单飞刷新：同一时间只有一个刷新请求，其他调用者等待它 */
function refreshAccessToken(): Promise<string> {
  if (!refreshPromise) {
    refreshPromise = fetch('/api/auth/refresh', { method: 'POST', credentials: 'include' })
      .then(async (response) => {
        if (!response.ok) throw new Error('刷新失败')
        const { token } = (await response.json()) as { token: string }
        accessToken = token
        return token
      })
      .finally(() => {
        // 无论成功失败都清理，下一次过期时可以重新刷新
        refreshPromise = null
      })
  }
  return refreshPromise
}

/** 带认证的请求：401 时刷新令牌并重放一次 */
export async function authFetch(input: string, init: RequestInit = {}): Promise<Response> {
  const send = (token: string | null) =>
    fetch(input, {
      ...init,
      headers: { ...Object.fromEntries(new Headers(init.headers)), ...(token ? { Authorization: `Bearer ${token}` } : {}) },
    })

  const tokenUsed = accessToken
  const response = await send(tokenUsed)
  if (response.status !== 401) return response

  try {
    // 如果其他请求已经刷新过（令牌已变化），直接使用新令牌，不再刷新
    const token = accessToken && accessToken !== tokenUsed ? accessToken : await refreshAccessToken()
    // 只重放一次，避免刷新后仍然 401 导致死循环
    return await send(token)
  } catch (error) {
    window.location.assign('/login')
    throw error
  }
}
```

**易错点**

- 刷新完成后没有清理 `refreshPromise`，令牌下次过期时直接返回旧结果
- 刷新接口本身返回 401 时又触发刷新，形成死循环
- 去重时 Promise 失败后没有从 Map 中删除，后续请求一直拿到失败结果
- 对 POST 等非幂等请求去重（语义错误）
- 重放时请求体是流，已经被消费

---

### H7. WebSocket 客户端：心跳、断线重连、消息队列

- **难度**: 高级
- **标签**: WebSocket
- **面试官看什么**: 心跳超时检测、退避重连、手动关闭不重连、离线消息缓存

**题干**

封装一个 WebSocket 客户端：定时发送心跳，超时未收到响应则判定断线；断线后指数退避重连；连接期间的发送请求先排队；调用 `close()` 后不再重连。

**参考实现**

```ts
type MessageHandler = (data: unknown) => void

/** 客户端配置 */
export interface ReconnectingSocketOptions {
  heartbeatIntervalMs?: number
  heartbeatTimeoutMs?: number
  maxReconnectDelayMs?: number
  maxQueueSize?: number
}

/** 带心跳和自动重连的 WebSocket 客户端 */
export class ReconnectingSocket {
  private ws: WebSocket | null = null
  private attempt = 0
  private manuallyClosed = false
  private heartbeatTimer?: ReturnType<typeof setInterval>
  private pongTimer?: ReturnType<typeof setTimeout>
  private reconnectTimer?: ReturnType<typeof setTimeout>
  private readonly queue: string[] = []
  private readonly handlers = new Set<MessageHandler>()
  private readonly options: Required<ReconnectingSocketOptions>

  constructor(private readonly url: string, options: ReconnectingSocketOptions = {}) {
    this.options = {
      heartbeatIntervalMs: 25_000,
      heartbeatTimeoutMs: 10_000,
      maxReconnectDelayMs: 30_000,
      maxQueueSize: 100,
      ...options,
    }
    this.connect()
  }

  onMessage(handler: MessageHandler): () => void {
    this.handlers.add(handler)
    return () => this.handlers.delete(handler)
  }

  send(data: unknown): void {
    const payload = JSON.stringify(data)
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(payload)
      return
    }
    // 断线期间排队，限制长度防止内存无限增长
    if (this.queue.length >= this.options.maxQueueSize) this.queue.shift()
    this.queue.push(payload)
  }

  close(): void {
    this.manuallyClosed = true
    clearTimeout(this.reconnectTimer)
    this.stopHeartbeat()
    this.ws?.close(1000, 'client closed')
  }

  private connect(): void {
    const ws = new WebSocket(this.url)
    this.ws = ws

    ws.onopen = () => {
      this.attempt = 0
      this.startHeartbeat()
      while (this.queue.length > 0 && ws.readyState === WebSocket.OPEN) {
        ws.send(this.queue.shift()!)
      }
    }

    ws.onmessage = (event) => {
      const data = JSON.parse(String(event.data)) as { type?: string }
      // 收到任何消息都说明连接存活
      clearTimeout(this.pongTimer)
      if (data.type === 'pong') return
      this.handlers.forEach((handler) => handler(data))
    }

    ws.onclose = () => {
      this.stopHeartbeat()
      if (!this.manuallyClosed) this.scheduleReconnect()
    }

    // error 之后一定会触发 close，重连统一在 onclose 中处理
    ws.onerror = () => ws.close()
  }

  private startHeartbeat(): void {
    this.stopHeartbeat()
    this.heartbeatTimer = setInterval(() => {
      if (this.ws?.readyState !== WebSocket.OPEN) return
      this.ws.send(JSON.stringify({ type: 'ping' }))
      // 移动网络断开时可能收不到 close 事件（半开连接），需要靠心跳超时主动发现
      this.pongTimer = setTimeout(() => this.ws?.close(4000, 'heartbeat timeout'), this.options.heartbeatTimeoutMs)
    }, this.options.heartbeatIntervalMs)
  }

  private stopHeartbeat(): void {
    clearInterval(this.heartbeatTimer)
    clearTimeout(this.pongTimer)
  }

  private scheduleReconnect(): void {
    const exponential = Math.min(this.options.maxReconnectDelayMs, 1000 * 2 ** this.attempt)
    // 抖动：服务恢复时避免所有客户端同时重连
    const delay = exponential / 2 + Math.random() * (exponential / 2)
    this.attempt++
    this.reconnectTimer = setTimeout(() => this.connect(), delay)
  }
}
```

**易错点**

- 手动关闭后仍然重连
- 只依赖 `onclose` 判断断线（半开连接收不到 close）
- 重连没有退避和抖动
- 心跳定时器没有清理，重连后存在多个定时器
- 重连后没有重新订阅和同步丢失的消息（真实系统需要带上最后的消息序号做增量同步）
- 页面进入后台时可以暂停心跳，回到前台立即检查连接

---

### H8. SSE：服务端推送和断点续传

- **难度**: 中级
- **标签**: SSE
- **面试官看什么**: 事件格式、Last-Event-ID、心跳注释、连接关闭时清理

**题干**

用 Node.js 实现 SSE 接口：推送带 ID 的事件；客户端重连时根据 `Last-Event-ID` 补发遗漏的事件；每 15 秒发送心跳注释；客户端断开时清理。

**参考实现**

```ts
import http from 'node:http'

interface StoredEvent {
  id: number
  event: string
  data: unknown
}

const history: StoredEvent[] = []
const clients = new Set<http.ServerResponse>()
let nextId = 1

function format(event: StoredEvent): string {
  // data 中的换行要拆成多行 data: 字段，这里用 JSON 序列化避免换行
  return `id: ${event.id}\nevent: ${event.event}\ndata: ${JSON.stringify(event.data)}\n\n`
}

/** 广播事件并保留最近 1000 条用于断点续传 */
export function broadcast(event: string, data: unknown): void {
  const stored = { id: nextId++, event, data }
  history.push(stored)
  if (history.length > 1000) history.shift()
  for (const client of clients) client.write(format(stored))
}

const server = http.createServer((req, res) => {
  if (req.url !== '/events') {
    res.writeHead(404).end()
    return
  }

  res.writeHead(200, {
    'Content-Type': 'text/event-stream; charset=utf-8',
    'Cache-Control': 'no-cache, no-transform',
    Connection: 'keep-alive',
    // 告诉 Nginx 不要缓冲，否则事件会被攒起来延迟发送
    'X-Accel-Buffering': 'no',
  })
  // 建议客户端断线后 3 秒重连
  res.write('retry: 3000\n\n')

  const lastEventId = Number(req.headers['last-event-id'])
  if (Number.isFinite(lastEventId)) {
    for (const event of history) {
      if (event.id > lastEventId) res.write(format(event))
    }
  }

  clients.add(res)
  // 注释行作为心跳，防止代理和负载均衡因空闲而断开连接
  const heartbeat = setInterval(() => res.write(': ping\n\n'), 15_000)

  req.on('close', () => {
    clearInterval(heartbeat)
    clients.delete(res)
  })
})

server.listen(3000)
```

```ts
// 浏览器端：EventSource 自动重连并携带 Last-Event-ID
const source = new EventSource('/events')
source.addEventListener('order-updated', (event) => {
  const data = JSON.parse((event as MessageEvent).data)
  console.log(data)
})
source.onerror = () => {
  // readyState 为 CONNECTING 表示浏览器正在自动重连
  if (source.readyState === EventSource.CLOSED) console.warn('连接已关闭')
}
```

**易错点**

- 没有禁用代理缓冲或压缩，事件延迟到达
- 客户端断开后没有清理定时器和连接集合（内存泄漏）
- `data` 中直接包含换行
- 历史事件无限增长
- 原生 `EventSource` 不能设置自定义头部，需要认证时使用 Cookie，或使用基于 fetch 的实现

---

### H9. 解析 Cache-Control 并判断响应是否新鲜

- **难度**: 中级
- **标签**: HTTP 缓存
- **面试官看什么**: 指令解析、新鲜度计算（Age、Date）、优先级

**题干**

实现 `getFreshness(responseHeaders, storedAt, now, isSharedCache)`，返回 `fresh`、`stale`（需要验证）、`stale-while-revalidate`（可以先用旧内容）或 `no-store`。

**参考实现**

```ts
/** 解析 Cache-Control 为指令表，指令名不区分大小写 */
export function parseCacheControl(value: string | null): Map<string, string | true> {
  const directives = new Map<string, string | true>()
  if (!value) return directives
  for (const part of value.split(',')) {
    const [rawName, ...rest] = part.trim().split('=')
    if (!rawName) continue
    const rawValue = rest.join('=').trim().replace(/^"|"$/g, '')
    directives.set(rawName.toLowerCase(), rawValue === '' ? true : rawValue)
  }
  return directives
}

function seconds(value: string | true | undefined): number | undefined {
  if (typeof value !== 'string') return undefined
  const parsed = Number.parseInt(value, 10)
  return Number.isFinite(parsed) && parsed >= 0 ? parsed : undefined
}

export type Freshness = 'no-store' | 'fresh' | 'stale-while-revalidate' | 'stale'

/** 根据 RFC 9111 的简化规则判断缓存的新鲜度 */
export function getFreshness(headers: Headers, storedAt: number, now: number, isSharedCache: boolean): Freshness {
  const cc = parseCacheControl(headers.get('cache-control'))
  if (cc.has('no-store')) return 'no-store'
  if (isSharedCache && cc.has('private')) return 'no-store'
  if (cc.has('no-cache')) return 'stale'

  // 新鲜度的优先级：共享缓存的 s-maxage > max-age > Expires
  let lifetime = (isSharedCache ? seconds(cc.get('s-maxage')) : undefined) ?? seconds(cc.get('max-age'))
  if (lifetime === undefined) {
    const expires = Date.parse(headers.get('expires') ?? '')
    const date = Date.parse(headers.get('date') ?? '') || storedAt
    lifetime = Number.isNaN(expires) ? 0 : Math.max(0, (expires - date) / 1000)
  }

  // 当前年龄 = 响应到达时已有的 Age + 在本地存放的时间
  const initialAge = seconds(headers.get('age') ?? undefined) ?? 0
  const currentAge = initialAge + (now - storedAt) / 1000

  if (currentAge < lifetime) return 'fresh'

  const swr = seconds(cc.get('stale-while-revalidate'))
  if (!cc.has('must-revalidate') && swr !== undefined && currentAge < lifetime + swr) {
    return 'stale-while-revalidate'
  }
  return 'stale'
}
```

**易错点**

- `no-cache` 当作不缓存（它表示必须验证）
- 忽略 `Age` 头（CDN 已经缓存了一段时间）
- 共享缓存缓存了 `private` 响应
- 没有 `max-age` 和 `Expires` 时，真实的浏览器会使用启发式新鲜度（`Last-Modified` 距今时间的 10%），这里简化为 0

---

### H10. 断点续传下载

- **难度**: 中级
- **标签**: Range, 下载
- **面试官看什么**: Range 请求、If-Range 防止拼接损坏、206 与 200 的处理、流式写入

**题干**

用 Node.js 下载大文件：已经下载的部分保存在本地，再次运行时从断点继续；如果服务器上的文件已经变化，则重新下载。

**参考实现**

```ts
import { createWriteStream } from 'node:fs'
import { readFile, stat, writeFile } from 'node:fs/promises'
import { Readable } from 'node:stream'
import { pipeline } from 'node:stream/promises'

/** 支持断点续传的下载：用 ETag 保证续传的是同一个文件 */
export async function resumableDownload(url: string, filePath: string): Promise<void> {
  const metaPath = `${filePath}.meta.json`
  const downloaded = await stat(filePath).then((s) => s.size).catch(() => 0)
  const meta = await readFile(metaPath, 'utf8').then((text) => JSON.parse(text) as { etag?: string }).catch(() => ({}))

  const headers: Record<string, string> = {}
  if (downloaded > 0 && meta.etag) {
    headers.Range = `bytes=${downloaded}-`
    // 文件未变化时返回 206，变化时返回 200 完整内容，避免把新旧文件拼接在一起
    headers['If-Range'] = meta.etag
  }

  const response = await fetch(url, { headers })
  if (response.status === 416) return // 已经下载完整
  if (!response.ok || !response.body) throw new Error(`下载失败: ${response.status}`)

  const resuming = response.status === 206
  const etag = response.headers.get('etag')
  if (etag) await writeFile(metaPath, JSON.stringify({ etag }))

  // 流式写入，不把整个文件读入内存；206 时追加，200 时覆盖
  await pipeline(
    Readable.fromWeb(response.body as import('node:stream/web').ReadableStream),
    createWriteStream(filePath, { flags: resuming ? 'a' : 'w' }),
  )
}
```

**易错点**

- 没有使用 `If-Range`，文件更新后把新旧内容拼接成损坏的文件
- 服务器不支持 Range（返回 200）时仍然追加写入
- 使用弱 ETag（`W/` 开头）作为 `If-Range` 的值（规范要求强验证器）
- 把整个响应读入内存
- 下载完成后没有校验完整性（大小或哈希）

---

### H11. 令牌桶限流

- **难度**: 中级
- **标签**: 限流
- **面试官看什么**: 惰性补充令牌、突发容量、与漏桶和固定窗口的区别

**题干**

实现令牌桶：容量为 `capacity`，每秒补充 `refillPerSecond` 个令牌；`tryConsume(n)` 返回是否允许，以及需要等待的时间。并支持按 key（用户）分别限流。

**参考实现**

```ts
/** 单个令牌桶：允许不超过容量的突发流量，长期平均速率等于补充速率 */
export class TokenBucket {
  private tokens: number
  private lastRefill: number

  constructor(
    private readonly capacity: number,
    private readonly refillPerSecond: number,
    private readonly now: () => number = () => performance.now(),
  ) {
    this.tokens = capacity
    this.lastRefill = now()
  }

  tryConsume(count = 1): { allowed: boolean; retryAfterMs: number } {
    this.refill()
    if (this.tokens >= count) {
      this.tokens -= count
      return { allowed: true, retryAfterMs: 0 }
    }
    const missing = count - this.tokens
    return { allowed: false, retryAfterMs: Math.ceil((missing / this.refillPerSecond) * 1000) }
  }

  // 惰性补充：不使用定时器，在每次请求时按经过的时间计算
  private refill(): void {
    const current = this.now()
    const elapsedSeconds = (current - this.lastRefill) / 1000
    this.tokens = Math.min(this.capacity, this.tokens + elapsedSeconds * this.refillPerSecond)
    this.lastRefill = current
  }
}

/** 按 key 分别限流；长时间未使用的桶会被清理 */
export class KeyedRateLimiter {
  private readonly buckets = new Map<string, { bucket: TokenBucket; lastUsed: number }>()

  constructor(private readonly capacity: number, private readonly refillPerSecond: number) {}

  tryConsume(key: string, count = 1) {
    const now = Date.now()
    let entry = this.buckets.get(key)
    if (!entry) {
      entry = { bucket: new TokenBucket(this.capacity, this.refillPerSecond), lastUsed: now }
      this.buckets.set(key, entry)
    }
    entry.lastUsed = now
    return entry.bucket.tryConsume(count)
  }

  /** 定期调用：桶满后等价于新桶，可以安全删除，防止 Map 无限增长 */
  sweep(idleMs = 10 * 60_000): void {
    const now = Date.now()
    for (const [key, entry] of this.buckets) {
      if (now - entry.lastUsed > idleMs) this.buckets.delete(key)
    }
  }
}
```

**易错点**

- 用定时器给每个桶补充令牌（大量 key 时开销巨大）
- 补充时没有上限（容量）
- 使用 `Date.now()` 受系统时间调整的影响（单机可以用单调时钟）
- 按 key 的 Map 无限增长
- 多实例部署时每个实例单独计数 → 需要 Redis + 原子脚本（见 [Next.js 手写题 H11](../nextjs/40-coding-handwritten.md)）

---

### H12. 简易反向代理

- **难度**: 高级
- **标签**: 代理, HTTP
- **面试官看什么**: 流式转发、逐跳头部、X-Forwarded-*、超时、上游连接复用

**题干**

用 Node.js `http` 模块实现反向代理：把请求流式转发到上游；移除逐跳头部；设置 `X-Forwarded-For` 等头部；上游超时返回 504，上游错误返回 502；上游使用 Keep-Alive 连接池。

**参考实现**

```ts
import http from 'node:http'

const UPSTREAM = { host: '127.0.0.1', port: 3000 }
const UPSTREAM_TIMEOUT_MS = 30_000

// 逐跳头部只对单个连接有意义，不能转发
const HOP_BY_HOP = new Set([
  'connection', 'keep-alive', 'proxy-authenticate', 'proxy-authorization',
  'te', 'trailer', 'transfer-encoding', 'upgrade',
])

const agent = new http.Agent({ keepAlive: true, maxSockets: 256 })

function filterHeaders(headers: http.IncomingHttpHeaders): http.OutgoingHttpHeaders {
  // Connection 头中列出的字段也是逐跳的
  const extra = String(headers.connection ?? '').split(',').map((s) => s.trim().toLowerCase())
  const result: http.OutgoingHttpHeaders = {}
  for (const [name, value] of Object.entries(headers)) {
    if (!HOP_BY_HOP.has(name) && !extra.includes(name)) result[name] = value
  }
  return result
}

const server = http.createServer((clientReq, clientRes) => {
  const headers = filterHeaders(clientReq.headers)
  const clientIp = clientReq.socket.remoteAddress ?? ''
  // 追加而不是覆盖：保留更前面的代理链；下游只应信任可信代理追加的部分
  const prior = clientReq.headers['x-forwarded-for']
  headers['x-forwarded-for'] = prior ? `${prior}, ${clientIp}` : clientIp
  headers['x-forwarded-proto'] = 'http'
  headers['x-forwarded-host'] = clientReq.headers.host ?? ''

  const upstreamReq = http.request({
    ...UPSTREAM,
    method: clientReq.method,
    path: clientReq.url,
    headers,
    agent,
  })

  upstreamReq.setTimeout(UPSTREAM_TIMEOUT_MS, () => {
    upstreamReq.destroy(new Error('upstream timeout'))
  })

  upstreamReq.on('response', (upstreamRes) => {
    clientRes.writeHead(upstreamRes.statusCode ?? 502, filterHeaders(upstreamRes.headers))
    // pipe 自动处理背压：客户端读得慢时暂停读取上游
    upstreamRes.pipe(clientRes)
  })

  upstreamReq.on('error', (error) => {
    if (clientRes.headersSent) {
      clientRes.destroy(error)
      return
    }
    const status = error.message === 'upstream timeout' ? 504 : 502
    clientRes.writeHead(status, { 'Content-Type': 'text/plain' }).end(http.STATUS_CODES[status])
  })

  // 客户端提前断开时中止上游请求，避免浪费上游资源
  clientRes.on('close', () => {
    if (!clientRes.writableFinished) upstreamReq.destroy()
  })

  clientReq.pipe(upstreamReq)
})

// 代理自身的空闲超时要大于前面负载均衡的空闲超时，避免偶发 502
server.keepAliveTimeout = 65_000
server.headersTimeout = 66_000
server.listen(8080)
```

**易错点**

- 把请求体或响应体完整读入内存再转发（大文件、流式响应失效）
- 转发了逐跳头部（`Connection`、`Transfer-Encoding`）
- 覆盖 `X-Forwarded-For`，或者信任客户端传来的值做鉴权
- 没有上游超时，或超时后没有销毁上游请求
- 客户端断开后上游请求继续执行
- 响应头已发送后再尝试 `writeHead`（会抛错）
- 没有处理 WebSocket 升级（需要监听 `upgrade` 事件单独转发）
