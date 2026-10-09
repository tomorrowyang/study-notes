# Node.js 初级 Q&A（能用）

> 目标：理解 Node 运行时特点，会用模块、核心 API、包管理和基础 Web 服务，知道常见坑。示例默认 ESM + `node:` 前缀。

---

## 一、运行时基础

### Q1. Node.js 是什么？有什么特点？

- **难度**: junior
- **标签**: 基础

**答案**

Node.js 是基于 V8 引擎的 JavaScript 服务端运行时：

- **事件驱动、非阻塞 I/O**：I/O 不阻塞主线程，适合高并发 I/O 密集型场景
- **单线程执行 JS**：通过事件循环处理并发请求
- **跨平台**：底层 libuv 屏蔽操作系统差异
- **npm 生态**：庞大的包生态

适合：API 服务、BFF、SSR、实时通信、CLI 工具、构建工具。  
不擅长：长时间 CPU 密集计算（需 Worker 或其他语言服务）。

**口述模板（30–60 秒）**

1. 一句话结论：Node 是基于 V8 和 libuv 的 JS 运行时，擅长 I/O 密集型服务。
2. 关键机制：事件循环 + 非阻塞 I/O。
3. 注意点：CPU 密集任务会阻塞事件循环。
4. 可追问时补充：worker_threads 处理计算任务。

**常见追问**

- Node 是语言吗？→ 不是，它是运行时；语言是 JavaScript / TypeScript。

---

### Q2. Node.js 真的是单线程吗？

- **难度**: junior
- **标签**: 线程模型

**答案**

准确说法：**JavaScript 代码默认在一个主线程（事件循环线程）上执行**，但 Node 进程本身是多线程的：

- **libuv 线程池**（默认 4 个）：处理文件系统、`dns.lookup`、部分 crypto、zlib
- **操作系统异步 I/O**：网络 I/O 通过 epoll / kqueue / IOCP，不占线程池
- **V8 后台线程**：GC、JIT 编译
- **worker_threads**：可以显式创建线程并行执行 JS

**口述模板（30–60 秒）**

1. 一句话结论：JS 执行是单线程，Node 进程是多线程。
2. 关键机制：网络 I/O 靠系统事件通知，文件等 I/O 靠线程池。
3. 注意点：主线程被阻塞时，所有请求都会受影响。
4. 可追问时补充：UV_THREADPOOL_SIZE 调优。

**常见追问**

- 单线程怎么处理成千上万连接？→ 连接等待期间不占线程，I/O 就绪后回调进入事件循环处理。

---

### Q3. Node.js 和浏览器中的 JavaScript 有什么区别？

- **难度**: junior
- **标签**: 运行环境

**答案**

| | 浏览器 | Node.js |
|--|--------|---------|
| 全局对象 | window（globalThis） | global（globalThis） |
| API | DOM、BOM、Web API | fs、http、process、Buffer 等 |
| 模块 | ESM | ESM + CommonJS |
| 事件循环 | HTML 规范定义，有渲染步骤 | libuv 实现，有阶段划分 |
| 安全模型 | 沙箱、同源策略 | 完整系统权限（可开启 Permission Model） |

现代 Node 也内置了很多 Web API：`fetch`、`URL`、`AbortController`、`Web Streams`、`structuredClone`、`WebSocket`（客户端）。

**口述模板（30–60 秒）**

1. 一句话结论：语言相同，宿主 API 和事件循环不同。
2. 关键机制：Node 提供系统级能力。
3. 注意点：Node 代码有完整文件和网络权限，安全要求更高。
4. 可追问时补充：Web 标准 API 正在统一。

**常见追问**

- Node 有 DOM 吗？→ 没有，需要 jsdom 等库模拟。

---

### Q4. CommonJS 和 ESM 在 Node 中如何使用？

- **难度**: junior
- **标签**: 模块

**答案**

```js
// CommonJS
const fs = require('node:fs')
module.exports = { read }

// ESM
import fs from 'node:fs'
export function read() {}
```

Node 判断模块类型：

- `.mjs` → ESM，`.cjs` → CommonJS
- `.js` → 看最近 package.json 的 `"type"`：`"module"` 为 ESM，否则 CommonJS

新项目推荐 ESM（`"type": "module"`）。

**口述模板（30–60 秒）**

1. 一句话结论：Node 同时支持 CJS 和 ESM，由扩展名和 type 字段决定。
2. 关键机制：CJS 同步 require，ESM 静态 import。
3. 注意点：ESM 中没有 `require`、`__dirname`、`module.exports`。
4. 可追问时补充：`require(esm)` 互操作。

**常见追问**

- ESM 中想用 require 怎么办？→ `import { createRequire } from 'node:module'`。

---

### Q5. require 的模块查找规则是什么？

- **难度**: junior
- **标签**: 模块

**答案**

`require(x)`：

1. 内置模块（`fs`、`node:fs`）直接返回
2. 以 `./`、`../`、`/` 开头：按文件路径查找，依次尝试 `x`、`x.js`、`x.json`、`x.node`，再尝试目录下的 `package.json` main 或 `index.js`
3. 否则按包名查找：从当前目录的 `node_modules` 开始逐级向上
4. 包中优先使用 `exports` 字段，其次 `main`

模块首次加载后会缓存（`require.cache`），再次 require 返回同一个对象。

**口述模板（30–60 秒）**

1. 一句话结论：内置模块 → 路径 → node_modules 逐级向上。
2. 关键机制：模块加载后缓存。
3. 注意点：exports 字段会限制可导入的子路径。
4. 可追问时补充：模块缓存导致的单例行为。

**常见追问**

- 两个包依赖同一库的不同版本会怎样？→ 各自 node_modules 中安装不同版本，可能出现多实例问题。

---

### Q6. npm 版本号 `^` 和 `~` 是什么意思？lockfile 有什么用？

- **难度**: junior
- **标签**: 包管理

**答案**

语义化版本 `MAJOR.MINOR.PATCH`：

- `^1.2.3`：允许 `>=1.2.3 <2.0.0`
- `~1.2.3`：允许 `>=1.2.3 <1.3.0`
- `1.2.3`：精确版本
- `^0.2.3`：允许 `>=0.2.3 <0.3.0`（0.x 时 minor 视为破坏性）

lockfile（package-lock.json / pnpm-lock.yaml / yarn.lock）锁定完整依赖树的精确版本，保证不同环境安装结果一致。

CI 使用 `npm ci` / `pnpm install --frozen-lockfile`。

**口述模板（30–60 秒）**

1. 一句话结论：`^` 锁大版本，`~` 锁小版本，lockfile 锁全部。
2. 关键机制：lockfile 记录解析后的版本和完整性哈希。
3. 注意点：lockfile 必须提交到仓库。
4. 可追问时补充：供应链安全。

**常见追问**

- npm install 和 npm ci 区别？→ ci 严格按 lockfile 安装，删除 node_modules，lockfile 不一致直接报错。

---

### Q7. npm、yarn、pnpm 有什么区别？

- **难度**: junior
- **标签**: 包管理

**答案**

| | npm | yarn (classic) | pnpm |
|--|-----|----------------|------|
| node_modules | 扁平化 | 扁平化 | 内容寻址存储 + 符号链接 |
| 磁盘占用 | 高 | 高 | 低（全局硬链接复用） |
| 幽灵依赖 | 有 | 有 | 默认避免 |
| Monorepo | workspaces | workspaces | workspaces（体验好） |

幽灵依赖（Phantom Dependency）：代码使用了未在 package.json 声明、但因扁平化被提升到根目录的包。

**口述模板（30–60 秒）**

1. 一句话结论：pnpm 节省磁盘且严格依赖。
2. 关键机制：全局存储 + 硬链接 + 符号链接。
3. 注意点：扁平化导致幽灵依赖。
4. 可追问时补充：Corepack 锁定包管理器版本。

**常见追问**

- 如何统一团队包管理器版本？→ package.json 中 `packageManager` 字段 + Corepack。

---

### Q8. package.json 有哪些关键字段？

- **难度**: junior
- **标签**: 包管理

**答案**

```json
{
  "name": "my-lib",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  },
  "bin": { "my-cli": "./bin/cli.js" },
  "scripts": { "dev": "node --watch src/index.js", "test": "node --test" },
  "engines": { "node": ">=22" },
  "dependencies": {},
  "devDependencies": {},
  "peerDependencies": { "react": ">=18" }
}
```

- `exports`：定义包入口，可做条件导出，限制内部文件访问
- `peerDependencies`：由宿主安装（插件、组件库）
- `engines`：声明 Node 版本要求

**口述模板（30–60 秒）**

1. 一句话结论：type 定模块类型，exports 定入口，engines 定运行环境。
2. 关键机制：exports 优先于 main。
3. 注意点：dependencies 和 devDependencies 区分生产依赖。
4. 可追问时补充：条件导出的 import/require。

**常见追问**

- peerDependencies 有什么用？→ 避免插件自带一份宿主库导致多实例。

---

## 二、核心模块

### Q9. fs 模块有哪几种 API 风格？

- **难度**: junior
- **标签**: fs

**答案**

```js
import fs from 'node:fs'
import { readFile } from 'node:fs/promises'

// 同步：阻塞事件循环
const a = fs.readFileSync('a.txt', 'utf8')

// 回调：错误优先
fs.readFile('a.txt', 'utf8', (err, data) => {})

// Promise：推荐
const b = await readFile('a.txt', 'utf8')
```

- 服务运行期间不要用同步 API
- 启动阶段读配置可以用同步 API
- 大文件用 Stream

**口述模板（30–60 秒）**

1. 一句话结论：同步、回调、Promise 三种，服务中优先 Promise。
2. 关键机制：异步 fs 操作在 libuv 线程池执行。
3. 注意点：同步 API 会阻塞所有请求。
4. 可追问时补充：大文件用 Stream。

**常见追问**

- `fs.exists` 为什么不推荐？→ 检查后再操作有竞态（TOCTOU），直接操作并处理 ENOENT。

---

### Q10. ESM 中如何获取当前文件路径？path 模块常用方法？

- **难度**: junior
- **标签**: path

**答案**

```js
// Node 20.11+ / 21.2+
const dir = import.meta.dirname
const file = import.meta.filename

// 兼容写法
import { fileURLToPath } from 'node:url'
import path from 'node:path'
const __filename = fileURLToPath(import.meta.url)
const __dirname = path.dirname(__filename)
```

path 常用：

- `path.join`：拼接并规范化
- `path.resolve`：解析为绝对路径（从右向左直到形成绝对路径）
- `path.extname`、`path.basename`、`path.dirname`
- `path.relative`

**口述模板（30–60 秒）**

1. 一句话结论：ESM 用 `import.meta.dirname` 获取目录。
2. 关键机制：ESM 模块以 URL 标识。
3. 注意点：不要手动拼接字符串路径，跨平台分隔符不同。
4. 可追问时补充：路径穿越安全问题。

**常见追问**

- `process.cwd()` 和 `__dirname` 区别？→ cwd 是启动命令所在目录，`__dirname` 是当前文件所在目录。

---

### Q11. process 对象有哪些常用能力？

- **难度**: junior
- **标签**: process

**答案**

- `process.env`：环境变量
- `process.argv`：命令行参数
- `process.cwd()`：当前工作目录
- `process.exit(code)`：退出（会中断未完成的异步操作）
- `process.exitCode = 1`：设置退出码，等待自然退出（推荐）
- `process.on('SIGTERM', ...)`：信号处理
- `process.memoryUsage()`、`process.cpuUsage()`
- `process.nextTick()`

**口述模板（30–60 秒）**

1. 一句话结论：process 代表当前 Node 进程。
2. 关键机制：环境、参数、信号、资源信息都在其上。
3. 注意点：优先设置 exitCode 而不是直接 exit。
4. 可追问时补充：SIGTERM 优雅关闭。

**常见追问**

- 命令行参数怎么解析？→ 内置 `util.parseArgs`，复杂 CLI 用 commander。

---

### Q12. Buffer 是什么？

- **难度**: junior
- **标签**: Buffer

**答案**

Buffer 用于处理二进制数据，是 `Uint8Array` 的子类，内存分配在 V8 堆外。

```js
const buf = Buffer.from('你好', 'utf8')
buf.length // 6，字节数而不是字符数
buf.toString('base64')
Buffer.concat([buf1, buf2])
```

常见场景：文件、网络数据、加密、图片处理。

注意：按字节切分多字节字符（中文）会产生乱码，需要 `StringDecoder` 或 `setEncoding`。

**口述模板（30–60 秒）**

1. 一句话结论：Buffer 是 Node 处理二进制的字节数组。
2. 关键机制：堆外内存，Uint8Array 子类。
3. 注意点：length 是字节数。
4. 可追问时补充：alloc 和 allocUnsafe 区别。

**常见追问**

- `new Buffer()` 为什么废弃？→ 行为不一致且可能泄漏未初始化内存，用 `Buffer.from` / `Buffer.alloc`。

---

### Q13. EventEmitter 怎么用？

- **难度**: junior
- **标签**: events

**答案**

```js
import { EventEmitter, once } from 'node:events'

const bus = new EventEmitter()
bus.on('order', (order) => console.log(order.id))
bus.once('ready', () => {})
bus.emit('order', { id: 1 })

await once(server, 'listening') // Promise 方式等待一次事件
```

注意：

- `emit` 是同步调用监听器
- 未监听的 `'error'` 事件会抛出异常导致进程崩溃
- 同一事件监听器超过 10 个会警告（可能泄漏）

**口述模板（30–60 秒）**

1. 一句话结论：Node 核心的发布订阅机制。
2. 关键机制：emit 同步按注册顺序执行监听器。
3. 注意点：必须处理 error 事件。
4. 可追问时补充：Stream、HTTP 都继承 EventEmitter。

**常见追问**

- emit 是异步的吗？→ 同步；需要异步可在监听器内部 setImmediate / queueMicrotask。

---

### Q14. 如何用原生 http 模块创建服务器？

- **难度**: junior
- **标签**: http

**答案**

```js
import http from 'node:http'

const server = http.createServer((req, res) => {
  if (req.method === 'GET' && req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' })
    res.end(JSON.stringify({ ok: true }))
    return
  }
  res.statusCode = 404
  res.end('Not Found')
})

server.listen(3000, () => console.log('listening on 3000'))
```

- `req` 是可读流，`res` 是可写流
- 必须调用 `res.end()`，否则请求挂起

**口述模板（30–60 秒）**

1. 一句话结论：createServer 接收请求回调，req/res 都是流。
2. 关键机制：每个请求触发一次 request 事件。
3. 注意点：忘记 end 会导致请求超时。
4. 可追问时补充：框架在其上封装路由和中间件。

**常见追问**

- Node 26 移除了哪个 http API？→ `writeHeader()`，使用 `writeHead()`。

---

### Q15. Express 中间件是什么？

- **难度**: junior
- **标签**: Express

**答案**

中间件是 `(req, res, next) => {}` 函数，按注册顺序执行：

```js
app.use(express.json())

app.use((req, res, next) => {
  const start = Date.now()
  res.on('finish', () => console.log(req.method, req.url, Date.now() - start))
  next()
})

app.get('/users/:id', async (req, res) => {
  res.json(await getUser(req.params.id))
})

// 错误处理中间件：4 个参数
app.use((err, req, res, next) => {
  res.status(500).json({ message: 'Internal Error' })
})
```

Express 5 已稳定，async 处理函数抛错会自动传给错误中间件。

**口述模板（30–60 秒）**

1. 一句话结论：中间件按顺序处理请求，通过 next 传递。
2. 关键机制：错误中间件有 4 个参数。
3. 注意点：不调用 next 也不响应会导致请求挂起。
4. 可追问时补充：Koa 洋葱模型。

**常见追问**

- Express 4 中 async 错误怎么处理？→ 需要 try/catch 后 next(err)，或使用 express-async-errors。

---

### Q16. 如何管理环境变量和配置？

- **难度**: junior
- **标签**: 配置

**答案**

```bash
node --env-file=.env src/index.js       # Node 20.6+ 内置
node --env-file-if-exists=.env.local src/index.js
```

原则：

- 配置通过环境变量注入（12-Factor）
- `.env` 不提交，提供 `.env.example`
- 启动时用 schema（Zod）校验配置，缺失立即失败
- 密钥放密钥管理系统（Vault、K8s Secret），不打进镜像

```js
const env = EnvSchema.parse(process.env)
```

**口述模板（30–60 秒）**

1. 一句话结论：配置走环境变量，启动时校验。
2. 关键机制：Node 已内置 --env-file。
3. 注意点：密钥不能进仓库和镜像。
4. 可追问时补充：多环境配置管理。

**常见追问**

- `process.env` 的值是什么类型？→ 都是字符串，需要转换类型。

---

### Q17. 错误优先回调是什么？如何转为 Promise？

- **难度**: junior
- **标签**: 异步

**答案**

Node 早期回调约定：第一个参数是错误，第二个是结果。

```js
fs.readFile('a.txt', (err, data) => {
  if (err) return handle(err)
  use(data)
})
```

转 Promise：

```js
import { promisify } from 'node:util'
const sleep = promisify(setTimeout)

// 更推荐直接用 Promise 版本 API
import { setTimeout as delay } from 'node:timers/promises'
await delay(1000)
```

**口述模板（30–60 秒）**

1. 一句话结论：回调第一个参数是错误。
2. 关键机制：promisify 把回调风格包装为 Promise。
3. 注意点：现代 API 大多有 promises 版本。
4. 可追问时补充：手写 promisify。

**常见追问**

- 回调地狱怎么解决？→ Promise + async/await。

---

### Q18. setTimeout、setImmediate、process.nextTick 有什么区别？

- **难度**: junior
- **标签**: 事件循环

**答案**

- `process.nextTick`：当前操作完成后立即执行，优先级最高（先于 Promise 微任务）
- `Promise.then`：微任务
- `setTimeout(fn, 0)`：timers 阶段执行（最少 1ms）
- `setImmediate`：check 阶段执行（poll 之后）

```js
setTimeout(() => console.log('timeout'))
setImmediate(() => console.log('immediate'))
Promise.resolve().then(() => console.log('promise'))
process.nextTick(() => console.log('nextTick'))
// nextTick → promise → timeout/immediate（主模块中这两者顺序不确定）
```

**口述模板（30–60 秒）**

1. 一句话结论：nextTick > Promise > 事件循环阶段回调。
2. 关键机制：nextTick 队列在每个回调之后优先清空。
3. 注意点：递归 nextTick 会饿死 I/O。
4. 可追问时补充：I/O 回调中 setImmediate 一定先于 setTimeout。

**常见追问**

- 想让出事件循环用哪个？→ setImmediate，而不是 nextTick。

---

### Q19. 读取大文件为什么要用 Stream？

- **难度**: junior
- **标签**: Stream

**答案**

`readFile` 会把整个文件读入内存，GB 级文件可能导致内存暴涨甚至 OOM，且首字节响应慢。

Stream 分块处理：

```js
import { createReadStream } from 'node:fs'
import { pipeline } from 'node:stream/promises'

app.get('/download', async (req, res) => {
  res.setHeader('Content-Type', 'application/octet-stream')
  await pipeline(createReadStream('big.zip'), res)
})
```

优点：内存占用稳定、处理更早开始、自动背压。

**口述模板（30–60 秒）**

1. 一句话结论：Stream 分块处理，内存恒定。
2. 关键机制：每次读取 highWaterMark 大小的块。
3. 注意点：用 pipeline 处理错误和背压。
4. 可追问时补充：背压原理。

**常见追问**

- Stream 一次读多大？→ 文件流默认 64 KiB（highWaterMark）。

---

### Q20. Node 中如何发起 HTTP 请求？

- **难度**: junior
- **标签**: fetch

**答案**

Node 18+ 内置全局 `fetch`（基于 Undici）：

```js
const res = await fetch('https://api.example.com/users', {
  signal: AbortSignal.timeout(5000),
})
if (!res.ok) throw new Error(`HTTP ${res.status}`)
const users = await res.json()
```

- 一定要设置超时
- 高性能场景可直接使用 Undici 的连接池 API
- fetch 对 4xx/5xx 不会 reject

**口述模板（30–60 秒）**

1. 一句话结论：现代 Node 直接用内置 fetch。
2. 关键机制：底层是 Undici，自带连接池。
3. 注意点：默认无超时限制，需 AbortSignal.timeout。
4. 可追问时补充：服务间调用的重试与熔断。

**常见追问**

- 还需要 axios 吗？→ 不是必需，拦截器等需求可以自己封装或继续用 axios。

---

## 三、工程与工具

### Q21. child_process 的基本用法？

- **难度**: junior
- **标签**: 子进程

**答案**

```js
import { execFile, spawn } from 'node:child_process'
import { promisify } from 'node:util'

const execFileAsync = promisify(execFile)
const { stdout } = await execFileAsync('git', ['rev-parse', 'HEAD'])

const child = spawn('ffmpeg', ['-i', input, output])
child.stderr.on('data', (chunk) => log(chunk.toString()))
child.on('close', (code) => {})
```

安全：不要把用户输入拼进 `exec` 的命令字符串（命令注入），用 `execFile` / `spawn` 传参数数组。

**口述模板（30–60 秒）**

1. 一句话结论：子进程用于运行外部命令或程序。
2. 关键机制：spawn 流式输出，exec 缓冲输出。
3. 注意点：用户输入不能拼接 shell 命令。
4. 可追问时补充：fork 与 IPC。

**常见追问**

- exec 输出过大会怎样？→ 超过 maxBuffer 会报错，大输出用 spawn。

---

### Q22. 如何处理跨域（CORS）？

- **难度**: junior
- **标签**: HTTP, CORS

**答案**

CORS 是浏览器的安全机制，服务端通过响应头声明允许的来源：

```js
res.setHeader('Access-Control-Allow-Origin', 'https://app.example.com')
res.setHeader('Access-Control-Allow-Credentials', 'true')
res.setHeader('Access-Control-Allow-Methods', 'GET,POST,PUT,DELETE')
res.setHeader('Access-Control-Allow-Headers', 'Content-Type,Authorization')
```

- 非简单请求会先发 OPTIONS 预检
- 携带 Cookie 时，Origin 不能是 `*`
- 使用白名单，不要反射任意 Origin

框架中使用 `cors` 中间件或 `@fastify/cors`。

**口述模板（30–60 秒）**

1. 一句话结论：服务端用响应头告诉浏览器允许哪些跨域请求。
2. 关键机制：预检请求 OPTIONS。
3. 注意点：credentials 时不能用通配符。
4. 可追问时补充：同域部署或网关代理可避免 CORS。

**常见追问**

- 服务端之间调用有 CORS 吗？→ 没有，CORS 只在浏览器中生效。

---

### Q23. 开发时如何自动重启？如何调试？

- **难度**: junior
- **标签**: 工具

**答案**

自动重启：

```bash
node --watch src/index.js          # Node 内置
node --watch-path=src src/index.js
```

调试：

```bash
node --inspect src/index.js        # Chrome 打开 chrome://inspect
node --inspect-brk src/index.js    # 第一行暂停
```

VS Code / Cursor 可直接附加调试器。

**口述模板（30–60 秒）**

1. 一句话结论：`--watch` 自动重启，`--inspect` 调试。
2. 关键机制：基于 V8 Inspector 协议。
3. 注意点：生产环境不要对公网开启 inspect 端口。
4. 可追问时补充：CPU / 内存分析也基于 Inspector。

**常见追问**

- 还需要 nodemon 吗？→ 简单场景不需要，内置 watch 已够用。

---

### Q24. Node 版本如何管理？LTS 是什么？

- **难度**: junior
- **标签**: 版本

**答案**

- 版本管理工具：nvm、fnm、Volta、mise
- 项目中用 `.nvmrc` / `engines` 声明版本
- **LTS（长期支持）**：生产应使用 LTS 版本

发布节奏（2026 起变化）：

- Node 26 及以前：每年 4 月偶数版、10 月奇数版，偶数版进入 LTS
- Node 27 起：每年 4 月发布一个大版本，10 月进入 LTS，每个版本都会成为 LTS，并新增 Alpha 测试通道

截至 2026-10：Node 24 为 LTS（即将转维护），Node 26 将于 10 月底进入 LTS。

**口述模板（30–60 秒）**

1. 一句话结论：生产只用 LTS，用工具锁定版本。
2. 关键机制：LTS 提供 30 个月左右的支持。
3. 注意点：CI、本地、生产版本保持一致。
4. 可追问时补充：升级前跑兼容性测试。

**常见追问**

- 为什么不用 Current 版本上生产？→ 可能有破坏性变化，稳定性和支持周期不如 LTS。

---

### Q25. Node 内置测试工具怎么用？

- **难度**: junior
- **标签**: 测试

**答案**

```js
import { test, describe, mock } from 'node:test'
import assert from 'node:assert/strict'

describe('sum', () => {
  test('adds numbers', () => {
    assert.equal(sum(1, 2), 3)
  })

  test('mock timers', (t) => {
    t.mock.timers.enable({ apis: ['setTimeout'] })
    // ...
  })
})
```

```bash
node --test
node --test --experimental-test-coverage
```

也可以继续使用 Vitest / Jest，生态更丰富。

**口述模板（30–60 秒）**

1. 一句话结论：Node 内置 node:test 和 assert。
2. 关键机制：零依赖，支持 mock、快照、覆盖率。
3. 注意点：大型项目可按团队习惯选 Vitest。
4. 可追问时补充：接口测试用 supertest 或直接 fetch。

**常见追问**

- 集成测试数据库怎么办？→ 用 Testcontainers 启动临时数据库容器。

---

### Q26. 如何在 Node 中直接运行 TypeScript？

- **难度**: junior
- **标签**: TypeScript

**答案**

现代 Node 支持类型擦除（Type Stripping），可以直接运行 `.ts`：

```bash
node src/index.ts
```

限制：

- 只擦除类型，不做类型检查（仍需 `tsc --noEmit`）
- 不支持需要代码转换的语法：`enum`、`namespace`、参数属性等（需 `--experimental-transform-types`）
- 导入时写 `.ts` 扩展名

其他方案：tsx、ts-node、先用 tsc / esbuild 编译。

**口述模板（30–60 秒）**

1. 一句话结论：新版 Node 可直接运行 TS，但只擦除类型。
2. 关键机制：去掉类型注解后执行。
3. 注意点：类型检查要单独跑。
4. 可追问时补充：推荐 `erasableSyntaxOnly` 配置。

**常见追问**

- 生产环境也直接跑 .ts 吗？→ 可以，但很多团队仍编译成 JS，便于 source map 和启动性能控制。

---

### Q27. 为什么生产环境不用 console.log 做日志？

- **难度**: junior
- **标签**: 日志

**答案**

问题：

- 无日志级别，难过滤
- 非结构化，难检索
- 大量输出时可能影响性能（输出到管道时为异步，但仍有开销）

推荐 pino（高性能 JSON 日志）：

```js
import pino from 'pino'
const logger = pino({ level: process.env.LOG_LEVEL ?? 'info' })

logger.info({ userId, orderId }, 'order created')
```

日志输出到 stdout，由容器 / 日志采集系统收集。

**口述模板（30–60 秒）**

1. 一句话结论：生产用结构化、分级日志。
2. 关键机制：JSON 日志便于检索聚合。
3. 注意点：不要记录密码、token 等敏感信息。
4. 可追问时补充：traceId 关联请求。

**常见追问**

- 日志写文件还是 stdout？→ 容器环境写 stdout，由平台采集。

---

### Q28. 什么是 globalThis？Node 中有哪些全局对象？

- **难度**: junior
- **标签**: 全局

**答案**

`globalThis` 是跨环境统一的全局对象引用，Node 中等于 `global`。

常见全局：`process`、`Buffer`、`console`、`setTimeout`、`setImmediate`、`queueMicrotask`、`structuredClone`、`fetch`、`URL`、`AbortController`、`TextEncoder`、`performance`。

CommonJS 中的 `require`、`module`、`exports`、`__dirname`、`__filename` 不是全局对象，而是模块包装函数的参数。

**口述模板（30–60 秒）**

1. 一句话结论：globalThis 是统一的全局对象。
2. 关键机制：CJS 中的 require 等是模块作用域变量。
3. 注意点：避免往全局对象上挂业务数据。
4. 可追问时补充：模块包装函数。

**常见追问**

- 为什么 ESM 中没有 `__dirname`？→ ESM 不经过 CJS 包装函数。

---

### Q29. 如何解析 JSON 请求体？有什么注意点？

- **难度**: junior
- **标签**: HTTP

**答案**

原生实现：

```js
async function readJson(req, limit = 1024 * 1024) {
  let size = 0
  const chunks = []
  for await (const chunk of req) {
    size += chunk.length
    if (size > limit) throw Object.assign(new Error('Payload Too Large'), { status: 413 })
    chunks.push(chunk)
  }
  return JSON.parse(Buffer.concat(chunks).toString('utf8'))
}
```

注意：

- 限制请求体大小，防止内存攻击
- 校验 Content-Type
- JSON.parse 失败返回 400
- 解析后用 schema 校验字段

**口述模板（30–60 秒）**

1. 一句话结论：读取请求流、限制大小、解析并校验。
2. 关键机制：req 是可读流，可异步迭代。
3. 注意点：不限制大小会被大请求打爆内存。
4. 可追问时补充：大 JSON 解析会阻塞事件循环。

**常见追问**

- 为什么先拼 Buffer 再转字符串？→ 逐块转字符串可能截断多字节字符。

---

### Q30. Node 适合哪些场景？不适合哪些？

- **难度**: junior
- **标签**: 选型

**答案**

适合：

- API 服务、BFF、网关
- SSR / RSC 渲染服务
- 实时通信（WebSocket、SSE）
- CLI 工具、构建工具
- Serverless 函数

不适合或需谨慎：

- 视频转码、图像处理、科学计算等 CPU 密集（可交给 Worker、原生模块或其他语言服务）
- 对强类型、复杂事务要求极高的核心交易系统（视团队能力，可用 Java/Go）

**口述模板（30–60 秒）**

1. 一句话结论：I/O 密集型适合，CPU 密集型需要额外方案。
2. 关键机制：单线程事件循环。
3. 注意点：选型要考虑团队技术栈。
4. 可追问时补充：前后端同构的效率优势。

**常见追问**

- 为什么前端团队常用 Node 做 BFF？→ 语言统一、可就近聚合接口和做 SSR。
