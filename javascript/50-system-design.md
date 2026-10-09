# JavaScript 系统设计题（8 道）

> 前端系统设计重点：先澄清需求，再讲模块划分、数据流、异常、性能、可观测性和演进方案。

---

### S1. 设计前端统一请求层

- **难度**: senior
- **标签**: 网络, 架构

**澄清问题**

- 是否支持 REST 和 GraphQL？
- 鉴权方式是 Cookie 还是 Token？
- 是否需要缓存、重试、取消、请求去重？
- 是否需要 SSR 复用？

**参考方案**

```text
业务 API
  ↓
Request Client
  ├─ Request Interceptors：baseURL、headers、traceId、鉴权
  ├─ Dedupe：相同 GET 请求复用 in-flight Promise
  ├─ Cache：短期缓存 / SWR
  ├─ Transport：fetch + AbortSignal.timeout
  ├─ Retry：幂等请求、网络错误、5xx、指数退避
  ├─ Response Interceptors：解析、错误码映射
  └─ Telemetry：耗时、失败率、错误上报
```

**关键设计**

- 请求配置标记 `idempotent`，只有幂等请求自动重试
- in-flight Map 的 key 包含 method、URL、query 和用户上下文
- Token 刷新时使用单飞（single-flight），避免多个请求同时刷新
- 业务错误、HTTP 错误、网络错误、取消错误分类型
- 上传进度场景保留 XHR 适配器

**口述模板（2–3 分钟）**

我会把请求层设计成可组合的中间件管线。请求前统一处理 baseURL、鉴权和 traceId；GET 请求可做 in-flight 去重和短期缓存。传输层使用 fetch，组合用户取消和超时 signal。失败时只对幂等请求、网络错误和 5xx 做指数退避重试。响应层统一解析数据并映射错误类型，最后上报耗时和错误率。Token 过期时用单飞刷新，其他请求等待刷新结果后重放。

**扩展追问**

- 如何避免 401 风暴？→ 共享一个 refresh Promise，刷新失败统一登出。
- 缓存什么时候失效？→ 写操作后按资源 key 失效，或使用 TanStack Query 管理。

---

### S2. 设计前端错误监控 SDK

- **难度**: senior
- **标签**: 监控, SDK

**澄清问题**

- 采集 JS 错误、资源错误、请求错误还是性能？
- 日活和上报量多大？
- 是否有隐私合规要求？
- 是否需要会话回放？

**参考方案**

```text
Collectors
  ├─ error / unhandledrejection
  ├─ resource error（capture）
  ├─ fetch / XHR wrapper
  ├─ framework error boundary
  └─ breadcrumbs：路由、点击、请求
        ↓
Processor：标准化 → 脱敏 → 指纹 → 采样 → 去重
        ↓
Queue：内存队列 + IndexedDB 持久化
        ↓
Transport：sendBeacon / fetch keepalive / 批量压缩
        ↓
Server：source map 还原 → 聚合 → 告警
```

**关键设计**

- 错误指纹：错误类型 + 归一化消息 + 栈顶帧
- 同一错误短时间限频，避免死循环刷爆服务端
- source map 只上传到监控平台，不公开部署
- 用户输入、URL query、token 等敏感信息脱敏
- SDK 自身异常必须吞掉，不能影响业务

**口述模板（2–3 分钟）**

SDK 分为采集、处理、队列和传输四层。采集层监听全局错误、Promise rejection、资源错误，并封装请求和框架错误边界；同时记录路由、点击、请求作为面包屑。处理层统一数据结构、脱敏、生成指纹、采样和去重。传输层批量发送，页面隐藏时使用 sendBeacon 或 keepalive。服务端通过版本号找到 source map 还原堆栈，再聚合、告警和关联发布。

**扩展追问**

- 如何区分发布导致的错误？→ 上报 release 版本，比较新旧版本错误率。
- 为什么监听资源错误要用捕获阶段？→ 资源 error 事件不冒泡。

---

### S3. 设计埋点 SDK

- **难度**: senior
- **标签**: 埋点, 性能

**澄清问题**

- 代码埋点、可视化埋点还是全埋点？
- 是否需要实时统计？
- 是否允许数据丢失？
- 用户同意策略如何？

**参考方案**

```text
track(event, properties)
  ↓
Context Enricher：用户匿名 ID、会话、页面、设备、版本
  ↓
Validator：事件 schema 校验
  ↓
Batch Queue：数量/时间阈值
  ↓
Storage：IndexedDB 离线缓存
  ↓
Sender：requestIdleCallback / visibilitychange 时 flush
```

**关键设计**

- 事件命名和字段由 schema registry 管理
- 曝光埋点使用 IntersectionObserver
- 点击埋点使用事件委托和 `data-track-*`
- 页面隐藏时 flush，不依赖 unload
- 支持用户同意前不采集或匿名采集

**口述模板（2–3 分钟）**

埋点 SDK 提供 track、identify 和 page 等 API。每个事件先补充上下文，再根据 schema 校验字段，进入批量队列。队列达到数量或时间阈值时发送，离线时写入 IndexedDB，恢复网络后补发。曝光使用 IntersectionObserver，点击通过事件委托读取 data 属性。页面隐藏时使用 sendBeacon flush。整个 SDK 要控制体积和主线程开销，并遵守用户同意策略。

**扩展追问**

- 如何保证不重复？→ 每个事件生成 eventId，服务端幂等去重。

---

### S4. 设计大数据处理与长任务调度方案

- **难度**: senior
- **标签**: 性能, Worker

**澄清问题**

- 数据量多大？计算是否 CPU 密集？
- 是否需要实时进度和取消？
- 结果是否要渲染成大表格或图表？

**参考方案**

```text
UI Thread
  ├─ 输入参数和取消控制
  ├─ 进度展示
  └─ 虚拟列表 / Canvas 渲染
          ↕ postMessage / Transferable
Worker Pool
  ├─ 数据分块
  ├─ 解析 / 过滤 / 聚合
  └─ 增量返回结果
```

**关键设计**

- CPU 密集任务放 Worker，主线程只负责交互和渲染
- 大型 ArrayBuffer 使用 Transferable，避免复制
- 分块处理并增量返回，展示进度
- 使用任务 ID 处理取消和过期结果
- 主线程不可避免的任务按时间片拆分，并调用 `scheduler.yield` 或回退方案

**口述模板（2–3 分钟）**

我会先判断瓶颈是计算还是渲染。计算密集任务放 Worker Pool，按数据块处理，结果分批返回，主线程展示进度并支持取消。传输大二进制数据时使用 Transferable。渲染侧使用虚拟列表或 Canvas，避免一次创建大量 DOM。主线程中的非关键工作拆成小任务并主动让出，确保输入响应和 INP 指标。

**扩展追问**

- Worker 数量设多少？→ 参考 `navigator.hardwareConcurrency`，并留出主线程和浏览器资源。

---

### S5. 设计一个轻量状态管理库

- **难度**: senior
- **标签**: 状态管理, 发布订阅

**澄清问题**

- 是否需要框架无关？
- 是否支持 selector 和中间件？
- 是否支持 SSR 和 DevTools？

**参考方案**

```text
createStore(initializer)
  ├─ state
  ├─ getState()
  ├─ setState(partial | updater)
  ├─ subscribe(listener)
  ├─ middleware：logger / persist / devtools
  └─ framework adapter：React useSyncExternalStore
```

**关键设计**

- 状态不可变更新，便于引用比较
- subscribe 返回取消函数
- selector + equality function 减少无效渲染
- React 适配使用 `useSyncExternalStore`，避免并发渲染撕裂
- SSR 每个请求创建独立 store，避免用户数据串号

**口述模板（2–3 分钟）**

核心 store 是一个框架无关的发布订阅模型：保存状态，setState 生成新状态并通知订阅者。组件通过 selector 订阅局部状态，只有 selector 结果变化才更新。React 适配层使用 useSyncExternalStore，保证并发渲染下快照一致。中间件包装 setState，实现日志、持久化和 DevTools。SSR 时按请求创建 store，客户端再用初始快照水合。

**扩展追问**

- 为什么 selector 返回新对象会导致频繁渲染？→ 引用每次不同，需要 shallow equal 或 memo selector。

---

### S6. 设计插件系统

- **难度**: senior
- **标签**: 插件, 架构

**澄清问题**

- 插件是否可信？
- 插件生命周期有哪些？
- 是否需要异步钩子和顺序控制？

**参考方案**

```text
Core
  ├─ Plugin Registry
  ├─ Lifecycle Hooks：init / beforeRun / afterRun / dispose
  ├─ Hook Types：sync / async series / async parallel / waterfall
  ├─ Context API：受限能力暴露
  └─ Error Isolation：单插件异常处理
```

**关键设计**

- 插件声明 name、version、依赖和执行顺序
- Hook 类型参考 Tapable：串行、并行、瀑布流、熔断
- 插件只拿到受限 Context，不暴露全部内部状态
- dispose 时清理事件、定时器和资源
- 不可信插件放 iframe 或 Worker 隔离

**口述模板（2–3 分钟）**

插件系统由核心、注册中心和生命周期钩子组成。插件注册时声明名称、版本、依赖和顺序，核心在关键节点触发 hook。根据需求提供串行、并行和瀑布流 hook。插件只能通过受限 Context 调用核心能力。每个插件需要 dispose 生命周期释放资源，并对插件错误做隔离。如果插件来自第三方，不能只靠同 Realm 沙箱，要使用 iframe 或 Worker 隔离。

**扩展追问**

- 插件依赖冲突怎么办？→ 注册阶段做拓扑排序和版本范围校验。

---

### S7. 设计搜索联想组件

- **难度**: mid
- **标签**: 防抖, 竞态, 可访问性

**澄清问题**

- 数据来自本地还是服务端？
- 是否需要高亮、分组、历史记录？
- 是否支持键盘操作和读屏？

**参考方案**

```text
Input
  ↓ debounce 200–300ms
Query State
  ├─ Cache：keyword → results（LRU）
  ├─ Request：AbortController 取消旧请求
  └─ Version：丢弃过期结果
  ↓
Combobox UI：listbox / option / aria-activedescendant
```

**关键设计**

- 输入防抖，中文输入法组合期间不发请求
- AbortController 取消旧请求
- LRU 缓存最近关键词
- 结果高亮必须转义，避免 XSS
- 键盘支持上下选择、Enter 确认、Escape 关闭

**口述模板（2–3 分钟）**

输入框监听用户输入，在 compositionend 后再处理中文输入。查询使用防抖，并通过 AbortController 取消旧请求，同时用版本号兜底丢弃过期结果。最近查询放 LRU 缓存。UI 遵循 ARIA combobox 模式，支持键盘导航和读屏。关键词高亮时不能直接拼 HTML，要先转义或用 DOM 节点拼接。

**扩展追问**

- 为什么取消请求后还要版本号？→ 某些请求库或缓存层可能无法取消，版本号是最终防线。

---

### S8. 设计大文件分片上传

- **难度**: senior
- **标签**: 上传, Worker, 并发

**澄清问题**

- 文件大小上限？
- 是否需要断点续传、秒传、暂停？
- 存储是自有服务还是 OSS / S3？

**参考方案**

```text
File
  ↓ Worker 计算 hash（增量）
Init Upload：hash、size、name → uploadId / 已上传分片
  ↓
Chunk Queue：Blob.slice
  ↓
Concurrency Scheduler：并发 3–6，失败重试
  ↓
Complete Upload：服务端合并并校验
```

**关键设计**

- Worker 增量计算 hash，避免阻塞主线程
- 服务端返回已上传分片，实现断点续传
- 文件 hash 已存在时秒传
- 分片上传并发受控，失败指数退避
- 暂停通过 AbortController 取消进行中的分片
- 直传 OSS/S3 使用临时签名 URL

**口述模板（2–3 分钟）**

我会先用 Blob.slice 分片，并在 Worker 中增量计算文件 hash。初始化时向服务端查询该 hash 的上传状态，已存在就秒传，否则返回已上传分片。前端使用并发调度器上传剩余分片，失败分片按退避策略重试，暂停时取消进行中的请求。全部完成后通知服务端合并，并校验整体 hash。大文件可直接上传对象存储，服务端只负责签名和状态管理。

**扩展追问**

- 为什么不一次性读取整个文件？→ 内存占用过高，分片读取和增量 hash 更稳定。
