# JavaScript 手写题（15 道高频）

> 建议：先口述思路，再写代码，最后主动说边界情况和测试用例。

---

### H1. 实现防抖 debounce（支持 leading / trailing / cancel）

- **难度**: mid
- **标签**: 闭包, 定时器
- **面试官看什么**: this 和参数是否保留、leading/trailing 语义、能否取消

**题干**

实现 `debounce(fn, wait, options)`，连续触发时只在停止触发 `wait` 毫秒后执行。

**思路口述**

每次调用都清除旧定时器并重新计时；记录最后一次的 this 和参数。leading 模式下，空闲期第一次调用立即执行；trailing 只有在等待期间又有新调用时才执行，避免同一次调用执行两次。

**参考实现**

```js
/**
 * 创建防抖函数：连续触发时只在空闲 wait 毫秒后执行。
 */
function debounce(fn, wait, { leading = false, trailing = true } = {}) {
  let timer = null
  let pendingArgs = null
  let pendingThis = null

  function invoke() {
    const args = pendingArgs
    const context = pendingThis
    pendingArgs = null
    pendingThis = null
    return fn.apply(context, args)
  }

  function debounced(...args) {
    pendingArgs = args
    pendingThis = this
    const isIdle = timer === null

    clearTimeout(timer)
    timer = setTimeout(() => {
      timer = null
      // leading 已执行且期间无新调用时 pendingArgs 为空，避免重复执行
      if (trailing && pendingArgs) invoke()
    }, wait)

    if (leading && isIdle) invoke()
  }

  debounced.cancel = () => {
    clearTimeout(timer)
    timer = null
    pendingArgs = null
    pendingThis = null
  }

  return debounced
}
```

**易错点**

- 用箭头函数包装 `debounced` 导致 this 丢失
- leading + trailing 时单次调用执行两次
- 组件卸载时没有 cancel
- 搜索场景只防抖不处理请求竞态

---

### H2. 实现节流 throttle

- **难度**: mid
- **标签**: 闭包, 定时器
- **面试官看什么**: 时间戳和定时器结合、最后一次调用是否执行

**题干**

实现 `throttle(fn, wait)`，`wait` 内最多执行一次，并保证最后一次触发能执行。

**思路口述**

记录上次执行时间。剩余时间小于等于 0 就立即执行；否则保存最后一次参数，并设置一个剩余时间后的 trailing 定时器。

**参考实现**

```js
/**
 * 创建节流函数：wait 毫秒内最多执行一次，并保留最后一次调用。
 */
function throttle(fn, wait) {
  let lastTime = 0
  let timer = null
  let pendingArgs = null
  let pendingThis = null

  function invoke(context, args) {
    lastTime = Date.now()
    fn.apply(context, args)
  }

  function throttled(...args) {
    const remaining = wait - (Date.now() - lastTime)

    if (remaining <= 0) {
      clearTimeout(timer)
      timer = null
      pendingArgs = null
      invoke(this, args)
      return
    }

    pendingArgs = args
    pendingThis = this
    if (!timer) {
      timer = setTimeout(() => {
        timer = null
        if (!pendingArgs) return
        const context = pendingThis
        const latestArgs = pendingArgs
        pendingArgs = null
        pendingThis = null
        invoke(context, latestArgs)
      }, remaining)
    }
  }

  throttled.cancel = () => {
    clearTimeout(timer)
    timer = null
    lastTime = 0
    pendingArgs = null
    pendingThis = null
  }

  return throttled
}
```

**易错点**

- 只用时间戳会丢失最后一次调用
- 只用定时器首次调用会延迟
- 视觉滚动更新更适合 requestAnimationFrame

---

### H3. 实现深拷贝 deepClone

- **难度**: mid
- **标签**: 递归, WeakMap
- **面试官看什么**: 循环引用、特殊对象、Symbol 键、原型

**题干**

实现深拷贝，支持普通对象、数组、Date、RegExp、Map、Set 和循环引用。

**思路口述**

原始值直接返回；用 WeakMap 记录已拷贝对象，遇到循环引用直接返回已创建结果。特殊对象分别处理，普通对象保留原型并复制自身可枚举属性，包括 Symbol 键。

**参考实现**

```js
/**
 * 深拷贝常见内置对象和普通对象，支持循环引用。
 */
function deepClone(value, seen = new WeakMap()) {
  if (value === null || typeof value !== 'object') return value
  if (seen.has(value)) return seen.get(value)

  if (value instanceof Date) return new Date(value)
  if (value instanceof RegExp) return new RegExp(value.source, value.flags)

  if (value instanceof Map) {
    const result = new Map()
    seen.set(value, result)
    value.forEach((item, key) => {
      result.set(deepClone(key, seen), deepClone(item, seen))
    })
    return result
  }

  if (value instanceof Set) {
    const result = new Set()
    seen.set(value, result)
    value.forEach((item) => result.add(deepClone(item, seen)))
    return result
  }

  const result = Array.isArray(value) ? [] : Object.create(Object.getPrototypeOf(value))
  // 先登记再递归，循环引用才能拿到同一个结果对象
  seen.set(value, result)

  for (const key of Reflect.ownKeys(value)) {
    const descriptor = Object.getOwnPropertyDescriptor(value, key)
    if (descriptor.enumerable) {
      result[key] = deepClone(value[key], seen)
    }
  }

  return result
}
```

**易错点**

- 递归后才写入 WeakMap，导致循环引用爆栈
- 忽略 Symbol 键
- 函数通常直接复用引用，不做拷贝
- 生产中优先 `structuredClone`，但它不保留类原型

---

### H4. 实现 call、apply、bind

- **难度**: mid
- **标签**: this
- **面试官看什么**: this 绑定、参数处理、bind 后 new

**题干**

实现 `myCall`、`myApply`、`myBind`。

**思路口述**

call 把函数临时挂到上下文对象上执行，并用 Symbol 避免覆盖属性。bind 返回新函数，普通调用使用绑定 this；通过 new 调用时忽略绑定 this，用原函数构造实例。

**参考实现**

```js
Function.prototype.myCall = function (context, ...args) {
  if (typeof this !== 'function') {
    throw new TypeError('myCall must be called on a function')
  }

  // 模拟非严格模式：null/undefined 指向全局对象，原始值装箱
  const target = context == null ? globalThis : Object(context)
  const key = Symbol('fn')
  target[key] = this

  try {
    return target[key](...args)
  } finally {
    delete target[key]
  }
}

Function.prototype.myApply = function (context, args = []) {
  return this.myCall(context, ...args)
}

Function.prototype.myBind = function (context, ...boundArgs) {
  if (typeof this !== 'function') {
    throw new TypeError('myBind must be called on a function')
  }

  const targetFn = this

  function boundFn(...args) {
    const finalArgs = [...boundArgs, ...args]
    if (new.target) {
      return Reflect.construct(targetFn, finalArgs, new.target)
    }
    return targetFn.apply(context, finalArgs)
  }

  if (targetFn.prototype) {
    boundFn.prototype = Object.create(targetFn.prototype)
  }

  return boundFn
}
```

**易错点**

- 用固定字符串属性名覆盖原对象属性
- 没有在 finally 中删除临时属性
- bind 后 new 仍使用绑定 this
- 冻结对象无法挂临时属性，原生 call 不受此限制

---

### H5. 实现 new 和 instanceof

- **难度**: mid
- **标签**: 原型
- **面试官看什么**: 原型连接、构造函数返回值、原型链遍历

**题干**

实现 `myNew(Ctor, ...args)` 和 `myInstanceOf(obj, Ctor)`。

**思路口述**

new：创建原型为 `Ctor.prototype` 的对象，执行构造函数；若返回对象或函数则使用返回值。instanceof：从对象原型开始向上查找 `Ctor.prototype`。

**参考实现**

```js
/**
 * 模拟普通构造函数的 new 行为。
 */
function myNew(Ctor, ...args) {
  if (typeof Ctor !== 'function') {
    throw new TypeError('Ctor must be a function')
  }

  const instance = Object.create(Ctor.prototype)
  const result = Ctor.apply(instance, args)
  const isObjectLike = result !== null && (typeof result === 'object' || typeof result === 'function')

  return isObjectLike ? result : instance
}

/**
 * 判断构造函数原型是否出现在对象原型链上。
 */
function myInstanceOf(obj, Ctor) {
  if (typeof Ctor !== 'function') {
    throw new TypeError('Right-hand side must be callable')
  }
  if (obj === null || (typeof obj !== 'object' && typeof obj !== 'function')) {
    return false
  }

  const prototype = Ctor.prototype
  let proto = Object.getPrototypeOf(obj)

  while (proto !== null) {
    if (proto === prototype) return true
    proto = Object.getPrototypeOf(proto)
  }

  return false
}
```

**易错点**

- 构造函数返回原始值时应忽略
- class 构造函数不能 `Ctor.apply`，完整实现应使用 `Reflect.construct`
- 原生 instanceof 还会检查 `Symbol.hasInstance`

---

### H6. 实现 Promise.all / allSettled / race / any

- **难度**: mid
- **标签**: Promise
- **面试官看什么**: 结果顺序、空输入、非 Promise 值、失败语义

**题干**

实现 Promise 四个静态组合器。

**思路口述**

所有输入先用 `Promise.resolve` 包装，以兼容普通值和 thenable。all/allSettled 按索引写结果，计数完成后 resolve；空输入立即完成。any 需要收集错误，全失败时抛 AggregateError。

**参考实现**

```js
/** 全部成功才成功，任一失败立即失败。 */
function promiseAll(iterable) {
  return new Promise((resolve, reject) => {
    const results = []
    let total = 0
    let fulfilledCount = 0

    for (const item of iterable) {
      const index = total++
      Promise.resolve(item).then((value) => {
        results[index] = value
        if (++fulfilledCount === total) resolve(results)
      }, reject)
    }

    if (total === 0) resolve(results)
  })
}

/** 等待全部完成，返回每项状态。 */
function promiseAllSettled(iterable) {
  return promiseAll(
    Array.from(iterable, (item) =>
      Promise.resolve(item).then(
        (value) => ({ status: 'fulfilled', value }),
        (reason) => ({ status: 'rejected', reason }),
      ),
    ),
  )
}

/** 返回第一个完成的结果。 */
function promiseRace(iterable) {
  return new Promise((resolve, reject) => {
    for (const item of iterable) {
      Promise.resolve(item).then(resolve, reject)
    }
  })
}

/** 返回第一个成功结果，全部失败时抛 AggregateError。 */
function promiseAny(iterable) {
  return new Promise((resolve, reject) => {
    const errors = []
    let total = 0
    let rejectedCount = 0

    for (const item of iterable) {
      const index = total++
      Promise.resolve(item).then(resolve, (reason) => {
        errors[index] = reason
        if (++rejectedCount === total) {
          reject(new AggregateError(errors, 'All promises were rejected'))
        }
      })
    }

    if (total === 0) {
      reject(new AggregateError([], 'All promises were rejected'))
    }
  })
}
```

**易错点**

- 用 `push` 导致结果顺序和输入不一致
- 忘记处理空数组
- 只支持数组，不支持任意可迭代对象
- race 空输入会永远 pending，这是规范行为

---

### H7. 实现简化版 Promise（符合 Promise/A+ 核心规则）

- **难度**: senior
- **标签**: Promise, 微任务
- **面试官看什么**: 状态机、异步回调、then 链、thenable 递归解析

**题干**

实现 `MyPromise`，支持 `then`、`catch`、`finally`、`resolve`、`reject`。

**思路口述**

Promise 是只能从 pending 改变一次的状态机。then 返回新 Promise；回调必须异步执行，所以使用 queueMicrotask。resolve 遇到 thenable 时要读取 then 并递归跟随其状态，同时防止 thenable 多次调用回调，并检测自我解析循环。

**参考实现**

```js
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

/**
 * 简化版 Promise：实现状态机、then 链和 thenable 解析。
 */
class MyPromise {
  #state = PENDING
  #value = undefined
  #handlers = []

  constructor(executor) {
    let called = false
    const resolve = (value) => {
      if (called) return
      called = true
      this.#resolveWith(value)
    }
    const reject = (reason) => {
      if (called) return
      called = true
      this.#settle(REJECTED, reason)
    }

    try {
      executor(resolve, reject)
    } catch (error) {
      reject(error)
    }
  }

  #resolveWith(x) {
    if (x === this) {
      this.#settle(REJECTED, new TypeError('Chaining cycle detected'))
      return
    }

    if (x !== null && (typeof x === 'object' || typeof x === 'function')) {
      let then
      try {
        // getter 只能读取一次，且读取过程可能抛错
        then = x.then
      } catch (error) {
        this.#settle(REJECTED, error)
        return
      }

      if (typeof then === 'function') {
        let called = false
        try {
          then.call(
            x,
            (y) => {
              if (called) return
              called = true
              this.#resolveWith(y)
            },
            (reason) => {
              if (called) return
              called = true
              this.#settle(REJECTED, reason)
            },
          )
        } catch (error) {
          if (!called) {
            called = true
            this.#settle(REJECTED, error)
          }
        }
        return
      }
    }

    this.#settle(FULFILLED, x)
  }

  #settle(state, value) {
    if (this.#state !== PENDING) return
    this.#state = state
    this.#value = value
    this.#handlers.forEach((handler) => this.#runHandler(handler))
    this.#handlers = []
  }

  #runHandler({ onFulfilled, onRejected, resolve, reject }) {
    queueMicrotask(() => {
      const isFulfilled = this.#state === FULFILLED
      const callback = isFulfilled ? onFulfilled : onRejected

      if (typeof callback !== 'function') {
        if (isFulfilled) resolve(this.#value)
        else reject(this.#value)
        return
      }

      try {
        resolve(callback(this.#value))
      } catch (error) {
        reject(error)
      }
    })
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const handler = { onFulfilled, onRejected, resolve, reject }
      if (this.#state === PENDING) this.#handlers.push(handler)
      else this.#runHandler(handler)
    })
  }

  catch(onRejected) {
    return this.then(undefined, onRejected)
  }

  finally(onFinally) {
    return this.then(
      (value) => MyPromise.resolve(onFinally?.()).then(() => value),
      (reason) => MyPromise.resolve(onFinally?.()).then(() => {
        throw reason
      }),
    )
  }

  static resolve(value) {
    return value instanceof MyPromise ? value : new MyPromise((resolve) => resolve(value))
  }

  static reject(reason) {
    return new MyPromise((_, reject) => reject(reason))
  }
}
```

**易错点**

- then 回调同步执行
- 没有返回新 Promise
- 未处理回调返回 Promise / thenable
- 未防止 thenable 多次调用 resolve/reject
- finally 修改了原值；规范中 finally 回调返回值不影响结果，抛错或 rejected 才覆盖

---

### H8. 实现并发控制调度器 Scheduler

- **难度**: mid
- **标签**: Promise, 并发
- **面试官看什么**: 队列、running 计数、任务失败后是否继续调度

**题干**

实现 `Scheduler`，最多同时执行 2 个异步任务：

```js
const scheduler = new Scheduler(2)
const timeout = (ms) => new Promise((resolve) => setTimeout(resolve, ms))
const addTask = (ms, label) => scheduler.add(() => timeout(ms)).then(() => console.log(label))

addTask(1000, '1')
addTask(500, '2')
addTask(300, '3')
addTask(400, '4')
// 输出：2 3 1 4
```

**思路口述**

add 返回 Promise，并把任务及其 resolve/reject 放入队列。每次尝试在 running 小于 limit 时出队执行；任务完成或失败都减少 running，再继续调度。

**参考实现**

```js
/**
 * 限制异步任务最大并发数，按添加顺序启动任务。
 */
class Scheduler {
  #limit
  #running = 0
  #queue = []

  constructor(limit) {
    if (!Number.isInteger(limit) || limit <= 0) {
      throw new RangeError('limit must be a positive integer')
    }
    this.#limit = limit
  }

  add(task) {
    return new Promise((resolve, reject) => {
      this.#queue.push({ task, resolve, reject })
      this.#runNext()
    })
  }

  #runNext() {
    while (this.#running < this.#limit && this.#queue.length > 0) {
      const { task, resolve, reject } = this.#queue.shift()
      this.#running++

      // 包一层 Promise，同步抛错也会进入 reject，避免 running 计数泄漏
      new Promise((innerResolve) => innerResolve(task()))
        .then(resolve, reject)
        .finally(() => {
          this.#running--
          this.#runNext()
        })
    }
  }
}
```

**易错点**

- 任务失败后没有释放并发槽
- 添加任务时就直接执行 task，导致并发失控
- 大队列用 `shift` 是 O(n)，生产可改为索引指针或链表

---

### H9. 实现带超时、重试和取消的 fetch

- **难度**: senior
- **标签**: fetch, AbortController
- **面试官看什么**: 错误分类、指数退避、取消传播

**题干**

实现 `fetchWithRetry(url, options)`：

- 每次请求有超时
- 网络错误、超时和 5xx 自动重试
- 4xx 不重试
- 用户取消后立即停止

**思路口述**

每次请求组合用户 signal 和超时 signal。网络错误或超时可重试；用户 signal 已取消直接抛错。HTTP 5xx 才重试，4xx 是请求本身问题。重试之间使用指数退避加随机抖动，sleep 也要支持取消。

**参考实现**

```js
/**
 * 支持超时、5xx 重试、指数退避和外部取消的 fetch。
 */
async function fetchWithRetry(url, options = {}) {
  const {
    retries = 3,
    timeout = 5000,
    baseDelay = 300,
    signal,
    ...init
  } = options

  for (let attempt = 0; ; attempt++) {
    const timeoutSignal = AbortSignal.timeout(timeout)
    const requestSignal = signal ? AbortSignal.any([signal, timeoutSignal]) : timeoutSignal

    let response
    try {
      response = await fetch(url, { ...init, signal: requestSignal })
    } catch (error) {
      if (signal?.aborted || attempt >= retries) throw error
    }

    if (response) {
      if (response.ok) return response

      // 4xx 通常是参数、鉴权或资源问题，重试不会改变结果
      if (response.status < 500 || attempt >= retries) {
        throw new Error(`HTTP ${response.status}`)
      }
    }

    // 随机抖动避免大量客户端同时重试造成惊群
    const delay = baseDelay * 2 ** attempt + Math.random() * baseDelay
    await sleep(delay, signal)
  }
}

/**
 * 可取消的延时 Promise。
 */
function sleep(ms, signal) {
  return new Promise((resolve, reject) => {
    if (signal?.aborted) {
      reject(signal.reason)
      return
    }

    const onAbort = () => {
      clearTimeout(timer)
      reject(signal.reason)
    }

    const timer = setTimeout(() => {
      signal?.removeEventListener('abort', onAbort)
      resolve()
    }, ms)

    signal?.addEventListener('abort', onAbort, { once: true })
  })
}
```

**易错点**

- 非幂等 POST 自动重试，导致重复下单
- 用户取消后仍继续重试
- 没有退避和抖动
- 429 可读取 `Retry-After`，生产中应单独处理

---

### H10. 实现 EventEmitter

- **难度**: mid
- **标签**: 发布订阅
- **面试官看什么**: on/off/once/emit、once 移除、emit 时修改监听器列表

**题干**

实现 `on`、`off`、`once`、`emit`。

**思路口述**

用 Map 存事件名到监听器数组。once 包装原函数，执行前先移除自己；wrapper 保存原函数引用，支持 off 原函数。emit 时复制监听器数组，避免执行过程中 off 影响遍历。

**参考实现**

```js
/**
 * 简单事件发布订阅中心。
 */
class EventEmitter {
  #events = new Map()

  on(eventName, listener) {
    if (typeof listener !== 'function') {
      throw new TypeError('listener must be a function')
    }

    const listeners = this.#events.get(eventName) ?? []
    listeners.push(listener)
    this.#events.set(eventName, listeners)

    return () => this.off(eventName, listener)
  }

  once(eventName, listener) {
    const wrapper = (...args) => {
      this.off(eventName, wrapper)
      listener.apply(this, args)
    }
    wrapper.originalListener = listener
    return this.on(eventName, wrapper)
  }

  off(eventName, listener) {
    const listeners = this.#events.get(eventName)
    if (!listeners) return

    if (!listener) {
      this.#events.delete(eventName)
      return
    }

    const next = listeners.filter(
      (item) => item !== listener && item.originalListener !== listener,
    )

    if (next.length > 0) this.#events.set(eventName, next)
    else this.#events.delete(eventName)
  }

  emit(eventName, ...args) {
    const listeners = this.#events.get(eventName)
    if (!listeners) return false

    // 复制快照，避免 once/off 在遍历中修改数组导致漏执行
    for (const listener of [...listeners]) {
      listener.apply(this, args)
    }

    return true
  }
}
```

**易错点**

- once 无法通过原函数 off
- emit 遍历原数组时被修改
- 订阅后忘记取消导致内存泄漏

---

### H11. 实现 LRU Cache

- **难度**: mid
- **标签**: Map, 数据结构
- **面试官看什么**: O(1) get/put、访问后更新顺序

**题干**

实现容量为 `capacity` 的 LRU 缓存，`get` 和 `put` 平均 O(1)。

**思路口述**

Map 保持插入顺序。get 命中时删除再插入，移到最新；put 已存在也先删除再插入。超容量时删除 Map 第一个键，即最久未使用项。

**参考实现**

```js
/**
 * 基于 Map 插入顺序实现的最近最少使用缓存。
 */
class LRUCache {
  #capacity
  #cache = new Map()

  constructor(capacity) {
    if (!Number.isInteger(capacity) || capacity <= 0) {
      throw new RangeError('capacity must be a positive integer')
    }
    this.#capacity = capacity
  }

  get(key) {
    if (!this.#cache.has(key)) return undefined

    const value = this.#cache.get(key)
    this.#cache.delete(key)
    this.#cache.set(key, value)
    return value
  }

  put(key, value) {
    if (this.#cache.has(key)) this.#cache.delete(key)
    this.#cache.set(key, value)

    if (this.#cache.size > this.#capacity) {
      const oldestKey = this.#cache.keys().next().value
      this.#cache.delete(oldestKey)
    }
  }
}
```

**易错点**

- get 后没有更新顺序
- 用数组维护顺序导致 O(n)
- 值可能为 undefined 时，用 `has` 判断而不是 `get`

---

### H12. 实现柯里化 curry

- **难度**: mid
- **标签**: 函数式编程, 闭包
- **面试官看什么**: 参数收集、this、arity

**题干**

```js
const add = (a, b, c) => a + b + c
const curriedAdd = curry(add)
curriedAdd(1)(2)(3)
curriedAdd(1, 2)(3)
```

**思路口述**

闭包保存已收集参数。参数数量达到原函数参数个数时执行，否则返回继续收集参数的函数。

**参考实现**

```js
/**
 * 将函数转换为可分批传参的柯里化函数。
 */
function curry(fn, arity = fn.length) {
  return function curried(...args) {
    if (args.length >= arity) {
      return fn.apply(this, args)
    }

    return function (...rest) {
      return curried.apply(this, [...args, ...rest])
    }
  }
}
```

**易错点**

- 默认参数和 rest 参数会让 `fn.length` 不准确，需要显式传 arity
- 返回箭头函数会锁死第一次的 this，是否需要取决于题目要求

---

### H13. 实现数组扁平化和去重

- **难度**: junior
- **标签**: Array, 递归
- **面试官看什么**: depth、递归和迭代、对象去重

**题干**

实现 `flatten(arr, depth)`，以及按 key 去重。

**思路口述**

depth 大于 0 时，遇到数组就递归展开 depth - 1；否则保留原值。深层数组可改用栈，避免递归爆栈。去重原始值用 Set，对象按 key 用 Map。

**参考实现**

```js
/**
 * 按指定深度扁平化数组。
 */
function flatten(arr, depth = Infinity) {
  if (depth <= 0) return arr.slice()

  return arr.reduce((result, item) => {
    if (Array.isArray(item)) {
      result.push(...flatten(item, depth - 1))
    } else {
      result.push(item)
    }
    return result
  }, [])
}

/**
 * 按 key 去重，保留第一次出现的元素。
 */
function uniqueBy(list, getKey) {
  const map = new Map()
  for (const item of list) {
    const key = getKey(item)
    if (!map.has(key)) map.set(key, item)
  }
  return [...map.values()]
}
```

**易错点**

- 超大数组 `push(...items)` 可能超过参数数量限制
- `JSON.stringify` 做对象去重受键顺序影响
- 原生 `arr.flat(depth)` 会跳过稀疏数组空位

---

### H14. 实现 Koa 洋葱模型 compose

- **难度**: senior
- **标签**: 中间件, Promise
- **面试官看什么**: 递归 dispatch、next 多次调用检测、异常传播

**题干**

实现 `compose(middlewares)`，使中间件按洋葱模型执行：

```js
compose([
  async (ctx, next) => { ctx.logs.push(1); await next(); ctx.logs.push(4) },
  async (ctx, next) => { ctx.logs.push(2); await next(); ctx.logs.push(3) },
])({ logs: [] })
// logs: [1, 2, 3, 4]
```

**思路口述**

dispatch(i) 执行第 i 个中间件，并把 `() => dispatch(i + 1)` 作为 next 传入。用 index 记录已执行位置，同一中间件多次调用 next 时报错。所有结果统一转换为 Promise。

**参考实现**

```js
/**
 * 将中间件数组组合成洋葱模型执行函数。
 */
function compose(middlewares) {
  if (!Array.isArray(middlewares)) {
    throw new TypeError('middlewares must be an array')
  }

  return function composed(ctx, finalNext) {
    let lastIndex = -1

    function dispatch(index) {
      // 同一层多次调用 next 会让下游中间件重复执行
      if (index <= lastIndex) {
        return Promise.reject(new Error('next() called multiple times'))
      }
      lastIndex = index

      const middleware = index === middlewares.length ? finalNext : middlewares[index]
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
```

**易错点**

- 中间件没有 await next，导致后续逻辑提前执行
- 同步异常未转为 rejected Promise
- 未检测 next 重复调用

---

### H15. 实现安全的 get / set 路径访问

- **难度**: mid
- **标签**: 对象, 安全
- **面试官看什么**: 路径解析、默认值、自动创建数组/对象、原型污染防护

**题干**

实现：

```js
get(obj, 'a.b[0].c', defaultValue)
set(obj, 'a.b[0].c', value)
```

**思路口述**

先把路径统一转换成 key 数组。get 逐层读取，遇到 null/undefined 返回默认值。set 逐层创建中间对象；下一个 key 是数字时创建数组。必须拒绝 `__proto__`、`prototype`、`constructor`，防止原型污染。

**参考实现**

```js
const UNSAFE_KEYS = new Set(['__proto__', 'prototype', 'constructor'])

/**
 * 将 a.b[0].c 形式的路径转换为 key 数组。
 */
function toPath(path) {
  if (Array.isArray(path)) return path.map(String)
  return String(path)
    .replace(/\[(\w+)\]/g, '.$1')
    .split('.')
    .filter(Boolean)
}

/**
 * 按路径安全读取对象值。
 */
function get(obj, path, defaultValue) {
  let current = obj

  for (const key of toPath(path)) {
    if (current == null) return defaultValue
    current = current[key]
  }

  return current === undefined ? defaultValue : current
}

/**
 * 按路径写入对象值，并阻止原型污染。
 */
function set(obj, path, value) {
  const keys = toPath(path)
  let current = obj

  keys.forEach((key, index) => {
    if (UNSAFE_KEYS.has(key)) {
      throw new Error(`Unsafe path key: ${key}`)
    }

    if (index === keys.length - 1) {
      current[key] = value
      return
    }

    if (current[key] === null || typeof current[key] !== 'object') {
      current[key] = /^\d+$/.test(keys[index + 1]) ? [] : {}
    }
    current = current[key]
  })

  return obj
}
```

**易错点**

- 用 `||` 返回默认值，导致 0、false、空字符串被覆盖
- 未防护 `__proto__`，产生原型污染
- 路径中的 key 包含点号时，需要支持数组路径
