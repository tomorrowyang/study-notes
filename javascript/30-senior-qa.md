# JavaScript 高级 Q&A（引擎、性能、架构、新特性）

> 目标：能把 JS 运行机制和线上问题、性能优化、安全、新特性落地结合起来讲。

---

## 一、V8 与内存

### Q1. V8 如何执行 JavaScript？

- **难度**: senior
- **标签**: V8, JIT

**答案**

简化流程：

```text
源码 → Parser → AST → Ignition 字节码解释执行
                         ↓ 收集类型反馈
                  Sparkplug / Maglev / TurboFan 优化编译
                         ↓ 假设失效
                       Deoptimization
```

- **Ignition**：解释器，快速启动
- **Sparkplug**：快速基线编译
- **Maglev**：中层优化编译
- **TurboFan**：高度优化编译
- 类型反馈来自运行时观察到的对象形状和值类型

**口述模板（30–60 秒）**

1. 一句话结论：V8 先解释执行，热点代码再根据类型反馈分层编译优化。
2. 关键机制：优化依赖类型和对象形状稳定。
3. 注意点：假设失效会去优化。
4. 可追问时补充：Hidden Class 和 Inline Cache。

**常见追问**

- 为什么懒解析（Lazy Parsing）？→ 很多函数启动时不执行，先预解析可减少启动成本。

---

### Q2. Hidden Class 和 Inline Cache 是什么？如何写出对 V8 友好的代码？

- **难度**: senior
- **标签**: V8, 性能

**答案**

- **Hidden Class / Map / Shape**：描述对象属性布局
- **Inline Cache（IC）**：在属性访问点缓存对象形状和偏移
- 访问点只看到一种形状为单态（monomorphic），多种为多态，太多为超态（megamorphic）

友好写法：

```js
class Point {
  constructor(x, y) {
    this.x = x
    this.y = y
  }
}
```

建议：

- 构造时初始化所有字段，顺序一致
- 避免运行时频繁 `delete` 属性
- 热点数组保持元素类型一致，避免空洞数组
- 不要过早微优化，先用 Profiler 找热点

**口述模板（30–60 秒）**

1. 一句话结论：对象形状稳定，属性访问才能被 IC 优化。
2. 关键机制：Hidden Class 记录布局，IC 缓存访问路径。
3. 注意点：delete 和不同属性顺序会产生不同形状。
4. 可追问时补充：只有热点代码值得这样优化。

**常见追问**

- `obj.a = undefined` 和 `delete obj.a` 哪个好？→ 热点对象通常前者更利于保持形状，但语义不同：属性仍存在。

---

### Q3. V8 垃圾回收机制是什么？

- **难度**: senior
- **标签**: GC, V8

**答案**

V8 使用分代回收（Generational GC），依据「大多数对象朝生夕死」：

- **新生代（Young Generation）**：Scavenge 复制回收，存活对象复制到另一空间，多次存活晋升老生代
- **老生代（Old Generation）**：标记清除、标记整理（Mark-Sweep / Mark-Compact）
- **Orinoco**：并行、增量、并发 GC，降低主线程停顿

**口述模板（30–60 秒）**

1. 一句话结论：V8 分新生代和老生代，用不同算法回收。
2. 关键机制：新生代复制回收，老生代标记清除和整理。
3. 注意点：GC 仍可能造成停顿。
4. 可追问时补充：并发和增量标记。

**常见追问**

- 为什么频繁创建大量临时对象会卡？→ 增加 GC 压力，可能引发更频繁回收和晋升。

---

### Q4. 如何定位前端内存泄漏？

- **难度**: senior
- **标签**: 内存, DevTools

**答案**

流程：

1. 复现：反复打开/关闭页面或组件
2. Performance Monitor 观察 JS Heap、DOM Nodes、Listeners 是否持续增长
3. Memory 面板拍多次 Heap Snapshot，手动 GC 后对比
4. 查找 Detached DOM、Closure、Array、Map 增长
5. 看 Retainers 引用链，定位谁在持有对象
6. 修复后重复验证

常见根因：全局缓存无上限、事件和定时器未清理、闭包持有组件、第三方库实例未销毁。

**口述模板（30–60 秒）**

1. 一句话结论：先复现，再对比 Heap Snapshot，最后看 Retainers 链。
2. 关键机制：泄漏是无用对象仍可达。
3. 注意点：单次内存增长不一定是泄漏，要看反复操作后的趋势。
4. 可追问时补充：Allocation instrumentation timeline。

**常见追问**

- Shallow Size 和 Retained Size 区别？→ Shallow 是对象自身大小，Retained 是释放该对象可回收的总大小。

---

### Q5. WeakRef 和 FinalizationRegistry 能用来做什么？

- **难度**: senior
- **标签**: WeakRef

**答案**

```js
const ref = new WeakRef(bigObject)
const value = ref.deref() // 可能 undefined

const registry = new FinalizationRegistry((key) => {
  cache.delete(key)
})
registry.register(bigObject, 'cache-key')
```

适合高级缓存或资源辅助清理，但：

- GC 时机不可预测
- 回调可能不执行
- 不能用来释放关键资源，如文件、锁、连接

关键资源应显式 `close()`，或使用 `using` 资源管理。

**口述模板（30–60 秒）**

1. 一句话结论：WeakRef 弱引用对象，FinalizationRegistry 观察回收。
2. 关键机制：不阻止 GC。
3. 注意点：不可依赖执行时机和是否执行。
4. 可追问时补充：显式资源管理。

**常见追问**

- 普通业务应该用吗？→ 一般不应，优先 WeakMap 或显式生命周期管理。

---

## 二、调度与并发

### Q6. 微任务饥饿是什么？

- **难度**: senior
- **标签**: Event Loop

**答案**

微任务队列会被一次性清空。如果微任务不断产生新的微任务，浏览器就无法进入下一个任务和渲染阶段：

```js
function loop() {
  queueMicrotask(loop)
}
loop() // 页面卡死
```

大量计算应分片到任务中，或使用 Worker。

**口述模板（30–60 秒）**

1. 一句话结论：微任务无限产生会阻塞渲染和输入。
2. 关键机制：每轮任务后要清空全部微任务。
3. 注意点：Promise 不是让出主线程的方式。
4. 可追问时补充：scheduler.yield。

**常见追问**

- 为什么 Promise 循环不能拆分长任务？→ 它仍在同一个微任务检查点内持续执行。

---

### Q7. 如何优化长任务和 INP？

- **难度**: senior
- **标签**: 性能, INP

**答案**

长任务通常指主线程超过 50ms 的任务。INP（Interaction to Next Paint）衡量交互到下一帧绘制的延迟。

优化：

- 先更新必要 UI，再执行非关键工作
- 拆分任务并主动让出主线程
- 计算密集逻辑放 Worker
- 减少大范围 DOM 更新和强制同步布局
- 大列表虚拟化

```js
async function processItems(items) {
  for (const item of items) {
    process(item)
    if (navigator.scheduling?.isInputPending?.()) {
      await yieldToMain()
    }
  }
}

function yieldToMain() {
  return globalThis.scheduler?.yield?.() ?? new Promise((resolve) => setTimeout(resolve))
}
```

`scheduler.yield` 能让出主线程，并让后续任务保持较高优先级；需做兼容回退。

**口述模板（30–60 秒）**

1. 一句话结论：减少主线程连续占用，让浏览器及时响应和绘制。
2. 关键机制：长任务拆分并让出。
3. 注意点：Promise 微任务不能让出给渲染。
4. 可追问时补充：Worker 和虚拟列表。

**常见追问**

- isInputPending 还推荐吗？→ 兼容性和适用性有限，更推荐定期 yield。

---

### Q8. Web Worker、SharedArrayBuffer、Atomics 怎么用？

- **难度**: senior
- **标签**: Worker, 并发

**答案**

- **Web Worker**：独立线程运行 JS，不能直接操作 DOM，通过 `postMessage` 通信
- **Transferable**：ArrayBuffer、MessagePort、OffscreenCanvas 可转移所有权，避免复制
- **SharedArrayBuffer**：多个线程共享内存，需要跨源隔离（COOP + COEP）
- **Atomics**：原子操作和等待通知，避免数据竞争

```js
worker.postMessage(buffer, [buffer]) // 转移后主线程 buffer 不可用
```

**口述模板（30–60 秒）**

1. 一句话结论：Worker 解决主线程计算阻塞。
2. 关键机制：消息传递默认结构化克隆，可用 Transferable 零拷贝。
3. 注意点：共享内存需要跨源隔离和原子操作。
4. 可追问时补充：Comlink 简化 RPC。

**常见追问**

- Worker 适合所有任务吗？→ 不适合小任务，通信和序列化有成本。

---

### Q9. Node.js 事件循环与浏览器有何不同？

- **难度**: senior
- **标签**: Node, Event Loop

**答案**

Node 基于 libuv，主要阶段：

```text
timers → pending callbacks → idle/prepare → poll → check → close callbacks
```

- `setTimeout` 在 timers 阶段
- `setImmediate` 在 check 阶段
- `process.nextTick` 队列优先于 Promise 微任务
- 每个回调执行后会处理 nextTick 和微任务

```js
setTimeout(() => console.log('timeout'))
setImmediate(() => console.log('immediate'))
```

主模块中二者顺序不确定；在 I/O 回调中 `setImmediate` 通常先于 `setTimeout`。

**口述模板（30–60 秒）**

1. 一句话结论：Node 有 libuv 阶段模型，并有 nextTick 优先队列。
2. 关键机制：timers、poll、check 是关键阶段。
3. 注意点：nextTick 递归会饿死事件循环。
4. 可追问时补充：I/O 回调中 setImmediate 顺序。

**常见追问**

- 现代 Node 中 Promise 和 nextTick 谁先？→ 同一检查点通常 nextTick 先。

---

### Q10. 前端请求竞态如何解决？

- **难度**: senior
- **标签**: 异步, 竞态

**答案**

场景：搜索框先发 A 再发 B，A 响应更慢，最终页面显示 A 的旧结果。

方案：

1. AbortController 取消旧请求
2. 请求序号，只接受最后一次请求结果
3. 使用 TanStack Query / SWR 等基于 key 管理请求
4. 后端支持幂等和版本号

```js
let controller
async function search(keyword) {
  controller?.abort()
  controller = new AbortController()
  const res = await fetch(`/api/search?q=${encodeURIComponent(keyword)}`, {
    signal: controller.signal,
  })
  return res.json()
}
```

**口述模板（30–60 秒）**

1. 一句话结论：旧请求晚到会覆盖新状态，要取消或丢弃。
2. 关键机制：AbortController 或请求版本号。
3. 注意点：取消前端请求不一定取消后端执行。
4. 可追问时补充：写操作要幂等。

**常见追问**

- 提交订单能简单取消吗？→ 不能依赖前端取消，后端需幂等键和状态查询。

---

## 三、元编程与模块

### Q11. Proxy 和 Reflect 如何实现响应式？

- **难度**: senior
- **标签**: Proxy, Vue

**答案**

核心：读时收集依赖，写时触发更新。

```js
const targetMap = new WeakMap()
let activeEffect = null

function reactive(target) {
  return new Proxy(target, {
    get(obj, key, receiver) {
      track(obj, key)
      const value = Reflect.get(obj, key, receiver)
      return value && typeof value === 'object' ? reactive(value) : value
    },
    set(obj, key, value, receiver) {
      const oldValue = obj[key]
      const result = Reflect.set(obj, key, value, receiver)
      if (!Object.is(oldValue, value)) trigger(obj, key)
      return result
    },
  })
}
```

Reflect 保持默认语义，并正确传递 receiver，避免 getter 中 this 指向问题。

**口述模板（30–60 秒）**

1. 一句话结论：Proxy 拦截读写，get 收集依赖，set 触发更新。
2. 关键机制：WeakMap → Map → Set 存储依赖。
3. 注意点：嵌套对象懒代理并缓存代理对象。
4. 可追问时补充：数组和 Map/Set 需要特殊处理。

**常见追问**

- 为什么用 Reflect.set？→ 返回布尔结果，并正确处理 receiver 和原型链 setter。

---

### Q12. Proxy 有哪些局限？

- **难度**: senior
- **标签**: Proxy

**答案**

- 代理对象和原对象身份不同：`proxy !== target`
- 私有字段 `#x` 不经过 Proxy，可能报错
- Map、Set、Date 等内置对象依赖内部槽，方法 this 指向代理可能失败
- 不变量限制：不可配置属性的 trap 返回值必须符合规范
- 性能比普通属性访问差，热点路径慎用

**口述模板（30–60 秒）**

1. 一句话结论：Proxy 很强，但不是透明对象替身。
2. 关键机制：内部槽和私有字段绕过 Proxy。
3. 注意点：身份比较会变化。
4. 可追问时补充：Vue 对集合类型的 instrumentation。

**常见追问**

- 如何代理 Map？→ 拦截方法访问并 bind 到原始对象，同时在 get/set/has 中收集和触发依赖。

---

### Q13. ESM 和 CommonJS 如何处理循环依赖？

- **难度**: senior
- **标签**: 模块化

**答案**

**CommonJS**：运行时执行，循环时可能拿到未执行完的 `module.exports`。

**ESM**：先链接导入导出绑定，再执行模块；导入是实时绑定。若执行时访问尚未初始化的 let/const/class，会触发 TDZ 错误。

建议：

- 抽出公共依赖
- 依赖倒置
- 延迟到函数调用时再访问
- 用 madge / dependency-cruiser 检查循环依赖

**口述模板（30–60 秒）**

1. 一句话结论：两者都允许循环，但都可能拿到未初始化状态。
2. 关键机制：CJS 是部分导出对象，ESM 是实时绑定。
3. 注意点：ESM 可能触发 TDZ。
4. 可追问时补充：架构上要消除循环。

**常见追问**

- 为什么 Webpack 打包后循环依赖可能报 undefined？→ 模块初始化顺序导致读取时导出尚未赋值。

---

### Q14. 动态 import 和代码分割要注意什么？

- **难度**: senior
- **标签**: 性能, 模块化

**答案**

```js
const { Chart } = await import('./Chart.js')
```

注意：

- 分割点放在路由、弹窗、编辑器、图表等大功能边界
- 预加载可能马上使用的模块：`<link rel="modulepreload">`
- 处理加载失败、重试和发布后旧 chunk 404
- 避免过细分割导致请求瀑布

**口述模板（30–60 秒）**

1. 一句话结论：动态 import 用于按需加载大功能。
2. 关键机制：构建器生成独立 chunk。
3. 注意点：过度分割会增加请求和瀑布。
4. 可追问时补充：预加载和版本发布兼容。

**常见追问**

- 发布后用户旧页面加载 chunk 失败怎么办？→ 保留旧资源一段时间，并捕获错误提示刷新。

---

### Q15. 如何设计前端错误监控？

- **难度**: senior
- **标签**: 监控, 错误处理

**答案**

采集：

- `window.onerror` / `error` 事件：运行时错误
- `unhandledrejection`：Promise 错误
- 资源错误：捕获阶段监听 `error`
- 框架错误边界：React Error Boundary、Vue errorHandler
- 请求错误：封装 fetch/XHR

上报：

- 采样、去重、限流
- 附带版本、页面、用户匿名 ID、浏览器、行为路径
- 使用 `sendBeacon` 或 `fetch(..., { keepalive: true })`

分析：

- 上传 source map 到私有平台
- 按错误指纹聚合
- 关联发布版本和告警

**口述模板（30–60 秒）**

1. 一句话结论：采集、上报、还原、聚合、告警形成闭环。
2. 关键机制：全局事件 + 框架边界 + 请求层封装。
3. 注意点：source map 不能公开暴露。
4. 可追问时补充：采样和隐私脱敏。

**常见追问**

- 跨域脚本只显示 Script error 怎么办？→ script 加 crossorigin，服务端返回 CORS 头。

---

## 四、安全

### Q16. 什么是原型污染（Prototype Pollution）？

- **难度**: senior
- **标签**: 安全

**答案**

攻击者通过 `__proto__`、`constructor.prototype` 等键，修改全局对象原型：

```js
merge({}, JSON.parse('{"__proto__":{"isAdmin":true}}'))
const victim = {}
victim.isAdmin // 不安全的 merge 下为 true
```

防御：

- 合并、set 路径时拒绝 `__proto__`、`prototype`、`constructor`
- 使用 `Object.create(null)` 或 Map 作为字典
- 输入使用 schema 校验
- 升级存在漏洞的依赖
- 可冻结关键原型，但需评估兼容性

**口述模板（30–60 秒）**

1. 一句话结论：恶意键修改原型，影响所有对象。
2. 关键机制：属性查找会沿原型链。
3. 注意点：深合并和路径 set 是高危点。
4. 可追问时补充：Map 和 null 原型对象。

**常见追问**

- JSON.parse 本身会污染吗？→ 通常只创建 `__proto__` 自有属性，危险在后续不安全合并。

---

### Q17. 前端 JS 中有哪些 XSS 危险 sink？

- **难度**: senior
- **标签**: 安全, XSS

**答案**

危险 sink：

- `innerHTML`、`outerHTML`、`insertAdjacentHTML`
- `document.write`
- `eval`、`new Function`、字符串形式 `setTimeout`
- `location = 'javascript:...'`
- 框架中的 `dangerouslySetInnerHTML`、`v-html`

防御：

- 默认使用 `textContent`
- 富文本使用 DOMPurify
- CSP：限制脚本来源，禁止 unsafe-inline / unsafe-eval
- Trusted Types：强制危险 sink 只能接收受信类型

**口述模板（30–60 秒）**

1. 一句话结论：把不可信字符串送入 HTML 或代码执行 sink 就有 XSS 风险。
2. 关键机制：浏览器把字符串解析为 HTML 或脚本。
3. 注意点：框架转义不覆盖危险 API。
4. 可追问时补充：CSP 和 Trusted Types。

**常见追问**

- URL 校验怎么做？→ 用 URL 解析并白名单协议，如只允许 http/https。

---

### Q18. eval、new Function 和沙箱有什么风险？

- **难度**: senior
- **标签**: 安全, 沙箱

**答案**

- `eval`：可访问局部作用域，影响优化，安全风险高
- `new Function`：只访问全局作用域，但仍执行任意代码
- `with + Proxy` 沙箱：只能隔离部分全局变量，无法作为安全边界

真正隔离不可信代码：

- 独立源 iframe + sandbox
- Worker + 严格通信协议
- 服务端隔离环境

微前端 JS 沙箱主要解决全局污染，不应被当作安全隔离。

**口述模板（30–60 秒）**

1. 一句话结论：同 Realm 的 JS 沙箱不是安全边界。
2. 关键机制：原型和全局对象可能被逃逸访问。
3. 注意点：CSP 会限制 eval。
4. 可追问时补充：ShadowRealm 仍在提案阶段。

**常见追问**

- qiankun 沙箱能防恶意代码吗？→ 不能，它主要防应用间全局状态冲突。

---

## 五、数值与数据

### Q19. 大数字和金额计算如何处理？

- **难度**: senior
- **标签**: Number, BigInt

**答案**

- `Number.MAX_SAFE_INTEGER` 为 `2^53 - 1`
- 后端雪花 ID 等超安全整数必须用字符串传输
- 整数大数用 BigInt
- 金额用最小单位整数或 decimal 库
- 格式化用 `Intl.NumberFormat`

```js
const amount = 12345n
const price = new Intl.NumberFormat('zh-CN', {
  style: 'currency',
  currency: 'CNY',
}).format(123.45)
```

`Math.sumPrecise`（ES2026）可提高浮点求和精度，但不能替代十进制金额计算。

**口述模板（30–60 秒）**

1. 一句话结论：大 ID 用字符串，大整数用 BigInt，金额用整数或 decimal。
2. 关键机制：Number 是双精度浮点。
3. 注意点：JSON.parse 会丢失大整数精度。
4. 可追问时补充：JSON.parse source access 可读取原始数字文本。

**常见追问**

- BigInt 和 Number 能混算吗？→ 不能直接混算，需要显式转换。

---

### Q20. 不可变数据和结构共享是什么？

- **难度**: senior
- **标签**: 不可变数据, 状态管理

**答案**

不可变更新：修改时返回新对象，未变化部分复用原引用。

```js
const next = {
  ...state,
  user: {
    ...state.user,
    name: 'new',
  },
}
```

好处：

- 引用比较即可判断变化
- 支持时间旅行、撤销重做
- 避免共享可变状态

Immer 使用 Proxy 记录草稿修改，最终生成结构共享的新对象。

**口述模板（30–60 秒）**

1. 一句话结论：不可变数据通过新引用表达变化。
2. 关键机制：结构共享避免全量深拷贝。
3. 注意点：嵌套更新易写错，可用 Immer。
4. 可追问时补充：React memo 依赖引用相等。

**常见追问**

- 深拷贝一份再修改可以吗？→ 结果正确但成本高，并会破坏未变化部分的引用复用。

---

## 六、新特性与工程落地

### Q21. ES2025 Iterator Helpers 有什么价值？

- **难度**: senior
- **标签**: ES2025, Iterator

**答案**

迭代器支持链式惰性操作：

```js
function* naturals() {
  let i = 1
  while (true) yield i++
}

const result = naturals()
  .filter((n) => n % 2 === 0)
  .map((n) => n * n)
  .take(3)
  .toArray() // [4, 16, 36]
```

和数组方法区别：

- 惰性求值，按需生成
- 可处理无限序列
- 不创建中间数组

**口述模板（30–60 秒）**

1. 一句话结论：Iterator Helpers 让迭代器具备惰性链式处理能力。
2. 关键机制：每个值按需穿过 map/filter/take。
3. 注意点：迭代器只能消费一次。
4. 可追问时补充：ES2026 Iterator.concat。

**常见追问**

- 数组怎么使用？→ `array.values().map(...).toArray()`。

---

### Q22. ES2025/2026 有哪些值得关注的新 API？

- **难度**: senior
- **标签**: ES2025, ES2026

**答案**

ES2025：

- Set 方法：`union`、`intersection`、`difference`、`isSubsetOf` 等
- `Promise.try`
- `RegExp.escape`
- JSON modules 和 import attributes
- `Float16Array`

ES2026：

- `Array.fromAsync`
- `Error.isError`
- `Math.sumPrecise`
- `Uint8Array` Base64 / Hex 转换
- `Iterator.concat`
- `Map.prototype.getOrInsert` / `getOrInsertComputed`

```js
const counts = new Map()
counts.set(word, counts.getOrInsert(word, 0) + 1)
```

**口述模板（30–60 秒）**

1. 一句话结论：新特性主要补齐集合、异步、迭代、二进制和错误判断能力。
2. 关键机制：标准 API 替代工具函数。
3. 注意点：生产环境以目标浏览器和 Node 版本为准。
4. 可追问时补充：polyfill 和 browserslist。

**常见追问**

- `Promise.try` 有什么用？→ 把可能同步抛错或返回 Promise 的函数统一包装为 Promise。

---

### Q23. Temporal 解决了 Date 的哪些问题？

- **难度**: senior
- **标签**: Temporal

**答案**

Temporal 已到 Stage 4，预计进入 ES2027。它提供：

- 不可变对象
- 区分纯日期、时间、日期时间、时间点、带时区日期时间
- 明确的时区和日历处理
- 可靠的日期加减和比较

```js
const date = Temporal.PlainDate.from('2026-10-09')
const next = date.add({ months: 1 })
```

落地：目标环境未支持时使用 polyfill，并封装日期工具层，避免业务代码散落调用。

**口述模板（30–60 秒）**

1. 一句话结论：Temporal 是现代日期时间 API，解决 Date 可变和时区模型混乱。
2. 关键机制：不同时间概念有不同类型。
3. 注意点：生产使用需确认兼容性。
4. 可追问时补充：PlainDate 和 ZonedDateTime 区别。

**常见追问**

- 生日应该用哪个类型？→ PlainDate，它没有时区和时间概念。

---

### Q24. `using` / `await using` 显式资源管理是什么？

- **难度**: senior
- **标签**: 资源管理

**答案**

作用域结束时自动释放资源：

```js
class FileHandle {
  [Symbol.dispose]() {
    this.close()
  }
}

{
  using file = new FileHandle()
  // ...
} // 自动调用 Symbol.dispose
```

异步资源实现 `Symbol.asyncDispose`，使用 `await using`。

价值：

- 统一资源释放协议
- 避免 finally 中遗漏释放
- 支持多资源按声明逆序释放

已到 Stage 4，预计进入 ES2027；生产使用依赖 TypeScript/Babel 编译和运行时支持。

**口述模板（30–60 秒）**

1. 一句话结论：using 是 JS 的自动资源释放语法。
2. 关键机制：离开作用域调用 dispose。
3. 注意点：需要目标环境或编译器支持。
4. 可追问时补充：DisposableStack。

**常见追问**

- 它和 FinalizationRegistry 区别？→ using 释放时机确定，FinalizationRegistry 不确定。

---

### Q25. 如何选择 polyfill 和语法转换策略？

- **难度**: senior
- **标签**: 工程化, 兼容性

**答案**

- **语法转换**：箭头函数、可选链、class fields 等由 Babel / SWC / esbuild 转换
- **API polyfill**：Promise、Array.prototype.at、Iterator helpers 等由 core-js 等补齐
- 用 browserslist 定义目标环境
- 现代浏览器和旧浏览器可分别构建
- 避免全量 polyfill 增大包体积

**口述模板（30–60 秒）**

1. 一句话结论：语法靠转译，API 靠 polyfill，目标环境由 browserslist 决定。
2. 关键机制：按需注入减少体积。
3. 注意点：有些能力无法 polyfill，如 Proxy。
4. 可追问时补充：差异化构建。

**常见追问**

- 第三方库需要转译吗？→ 若库发布了高版本语法且目标环境不支持，需要纳入转译范围。

---

### Q26. 观察者模式和发布订阅模式有什么区别？

- **难度**: senior
- **标签**: 设计模式

**答案**

- **观察者模式**：被观察者直接维护观察者列表，并直接通知
- **发布订阅模式**：发布者和订阅者通过事件中心通信，彼此不知道对方

```text
Observer: Subject → Observer
Pub/Sub:  Publisher → Event Bus → Subscriber
```

发布订阅解耦更强，但过度使用会让数据流难以追踪；大型应用应优先明确的状态管理和依赖关系。

**口述模板（30–60 秒）**

1. 一句话结论：区别在是否有中间事件中心。
2. 关键机制：观察者直接通知，发布订阅间接通知。
3. 注意点：事件总线滥用会导致隐式依赖。
4. 可追问时补充：手写 EventEmitter。

**常见追问**

- 事件总线怎么避免泄漏？→ 订阅返回取消函数，并在组件卸载时调用。
