# JavaScript 中级 Q&A（原理）

> 目标：讲清作用域、闭包、this、原型、事件循环、Promise、模块系统，并能做输出题。

---

## 一、作用域与闭包

### Q1. 什么是执行上下文和调用栈？

- **难度**: mid
- **标签**: 执行机制

**答案**

执行上下文（Execution Context）是代码运行时的环境，主要包括：

- 词法环境（Lexical Environment）：let、const、函数声明、外层环境引用
- 变量环境（Variable Environment）：var 声明
- this 绑定

调用栈（Call Stack）按后进先出管理执行上下文：

```js
function a() { b() }
function b() { console.trace() }
a() // 栈：global → a → b
```

递归过深会导致 `RangeError: Maximum call stack size exceeded`。

**口述模板（30–60 秒）**

1. 一句话结论：执行上下文是代码运行环境，调用栈管理函数调用顺序。
2. 关键机制：每调用函数就压入新上下文，返回时弹出。
3. 注意点：同步代码会阻塞调用栈。
4. 可追问时补充：异步回调要等调用栈清空后执行。

**常见追问**

- 函数调用时创建作用域吗？→ 作用域由代码书写位置决定，调用时创建执行上下文和环境记录。

---

### Q2. 词法作用域和作用域链是什么？

- **难度**: mid
- **标签**: 作用域

**答案**

JS 使用词法作用域（Lexical Scope）：变量访问规则由函数定义位置决定，而不是调用位置。

```js
const value = 'global'
function outer() {
  const value = 'outer'
  return function inner() {
    return value
  }
}
outer()() // 'outer'
```

查找变量时，从当前环境开始，沿外层环境引用逐层向上，直到全局，这条链就是作用域链。

**口述模板（30–60 秒）**

1. 一句话结论：变量查找由定义位置决定。
2. 关键机制：环境记录通过 outer 引用形成作用域链。
3. 注意点：this 不是词法作用域，箭头函数除外。
4. 可追问时补充：闭包就是作用域链被保留。

**常见追问**

- `eval` 会影响作用域吗？→ 直接 eval 可以访问和修改局部作用域，影响优化和安全，应避免。

---

### Q3. 什么是闭包？有哪些应用场景？

- **难度**: mid
- **标签**: 闭包

**答案**

闭包是函数和其定义时词法环境的组合。内部函数在外部函数返回后仍能访问外部变量。

```js
function createCounter() {
  let count = 0
  return {
    increment: () => ++count,
    get: () => count,
  }
}
```

应用：

- 数据私有
- 函数工厂
- 防抖、节流、memoize
- 模块模式
- React Hooks 依赖闭包读取当次渲染的状态

**口述模板（30–60 秒）**

1. 一句话结论：闭包让函数记住定义时的作用域。
2. 关键机制：被引用的外部变量不会被 GC 回收。
3. 注意点：长期持有大对象可能内存泄漏。
4. 可追问时补充：React stale closure 问题。

**常见追问**

- 闭包一定导致内存泄漏吗？→ 不会，只有不再需要却被长期引用才是泄漏。

---

### Q4. 循环中的 var 闭包问题如何解决？

- **难度**: mid
- **标签**: 闭包, 输出题

**答案**

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i))
}
// 3 3 3
```

`var` 只有一个函数级变量，回调执行时循环已结束。

解决：

```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i))
}
// 0 1 2
```

`let` 在每次迭代创建新的绑定。旧方案可使用 IIFE 或 `setTimeout` 第三个参数传值。

**口述模板（30–60 秒）**

1. 一句话结论：var 共享同一个变量，let 每轮新绑定。
2. 关键机制：回调在循环结束后执行。
3. 注意点：问题本质是作用域，不是 setTimeout。
4. 可追问时补充：IIFE 方案。

**常见追问**

- for 循环头部的 let 是怎么每轮复制的？→ 规范会为每次迭代创建新环境，并复制上轮值。

---

### Q5. React 中的 stale closure 是什么？

- **难度**: mid
- **标签**: 闭包, React

**答案**

函数捕获了旧渲染周期的状态，后续执行时读取不到最新值：

```js
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1) // count 永远是首次渲染的值
  }, 1000)
  return () => clearInterval(id)
}, [])
```

解决：

- 函数式更新：`setCount((c) => c + 1)`
- 正确填写依赖
- 用 ref 保存最新值
- React 新 API `useEffectEvent` 用于读取最新值的事件逻辑（按版本确认可用性）

**口述模板（30–60 秒）**

1. 一句话结论：回调捕获了旧状态。
2. 关键机制：每次渲染都是新的函数作用域。
3. 注意点：空依赖 effect 中最常见。
4. 可追问时补充：函数式更新和 ref。

**常见追问**

- 为什么不能都用 ref？→ ref 修改不触发渲染，滥用会让数据流难以追踪。

---

## 二、this、原型与继承

### Q6. this 的绑定规则有哪些？

- **难度**: mid
- **标签**: this

**答案**

优先级从高到低：

1. `new` 绑定：this 指向新实例
2. 显式绑定：`call`、`apply`、`bind`
3. 隐式绑定：`obj.fn()` 中 this 指向 obj
4. 默认绑定：严格模式为 undefined，非严格为全局对象

箭头函数不适用这些规则，它使用外层词法 this。

```js
const fn = obj.method
fn() // 隐式绑定丢失
```

**口述模板（30–60 秒）**

1. 一句话结论：this 由调用方式决定。
2. 关键机制：new > 显式 > 隐式 > 默认。
3. 注意点：方法赋值或作为回调传递时会丢失 this。
4. 可追问时补充：箭头函数词法 this。

**常见追问**

- bind 后再 new 会怎样？→ new 优先，绑定的 this 被忽略，预置参数保留。

---

### Q7. call、apply、bind 有什么区别？

- **难度**: mid
- **标签**: this

**答案**

| | 执行时机 | 参数 |
|--|----------|------|
| `call` | 立即执行 | 逐个传 |
| `apply` | 立即执行 | 数组传 |
| `bind` | 返回新函数 | 可预置参数 |

```js
fn.call(ctx, 1, 2)
fn.apply(ctx, [1, 2])
const bound = fn.bind(ctx, 1)
bound(2)
```

bind 后的函数再次 bind 不能改变 this，只能继续追加参数。

**口述模板（30–60 秒）**

1. 一句话结论：call/apply 立即调用，bind 返回函数。
2. 关键机制：都用于指定 this。
3. 注意点：bind 只第一次绑定的 this 生效。
4. 可追问时补充：手写 bind 处理 new。

**常见追问**

- 数组最大值怎么用 apply？→ `Math.max.apply(null, arr)`，现代写 `Math.max(...arr)`，大数组可能爆栈，应循环。

---

### Q8. `new` 的执行过程是什么？

- **难度**: mid
- **标签**: new

**答案**

1. 创建新对象
2. 将新对象原型指向构造函数的 `prototype`
3. 以新对象为 this 执行构造函数
4. 如果构造函数返回对象或函数，则返回它；否则返回新对象

```js
function myNew(Ctor, ...args) {
  const obj = Object.create(Ctor.prototype)
  const result = Ctor.apply(obj, args)
  return result !== null && (typeof result === 'object' || typeof result === 'function')
    ? result
    : obj
}
```

**口述模板（30–60 秒）**

1. 一句话结论：创建对象、连接原型、执行构造、处理返回值。
2. 关键机制：实例通过原型访问方法。
3. 注意点：构造函数返回对象会覆盖实例。
4. 可追问时补充：class 构造函数不能用 apply 调用。

**常见追问**

- 怎么判断函数是否被 new 调用？→ `new.target`。

---

### Q9. 原型和原型链是什么？

- **难度**: mid
- **标签**: 原型

**答案**

- 函数有 `prototype` 属性，用作其实例的原型
- 对象有内部槽 `[[Prototype]]`，可通过 `Object.getPrototypeOf` 访问
- 属性查找时先查自身，再沿原型链向上，直到 `null`

```js
function Person() {}
const p = new Person()

Object.getPrototypeOf(p) === Person.prototype // true
Object.getPrototypeOf(Person.prototype) === Object.prototype // true
Object.getPrototypeOf(Object.prototype) === null // true
```

**口述模板（30–60 秒）**

1. 一句话结论：原型链是 JS 实现继承和属性查找的机制。
2. 关键机制：自身找不到时沿 [[Prototype]] 向上。
3. 注意点：`__proto__` 是历史访问器，不推荐业务使用。
4. 可追问时补充：class 的方法也在 prototype 上。

**常见追问**

- `Function.__proto__ === Function.prototype` 吗？→ 是，Function 本身也是函数。

---

### Q10. instanceof 的原理是什么？

- **难度**: mid
- **标签**: 原型

**答案**

`a instanceof B` 默认检查 `B.prototype` 是否出现在 `a` 的原型链上。

```js
function myInstanceOf(obj, Ctor) {
  if (obj === null || (typeof obj !== 'object' && typeof obj !== 'function')) return false
  let proto = Object.getPrototypeOf(obj)
  while (proto) {
    if (proto === Ctor.prototype) return true
    proto = Object.getPrototypeOf(proto)
  }
  return false
}
```

可通过 `Symbol.hasInstance` 自定义 instanceof 行为。

**口述模板（30–60 秒）**

1. 一句话结论：沿原型链查找构造函数 prototype。
2. 关键机制：比较对象引用。
3. 注意点：跨 iframe 的数组 instanceof Array 可能为 false。
4. 可追问时补充：Array.isArray 更可靠。

**常见追问**

- 原始值 instanceof 为什么是 false？→ 原始值不是对象，不会自动参与原型链判断。

---

### Q11. JS 有哪些继承方式？现在推荐哪种？

- **难度**: mid
- **标签**: 继承

**答案**

历史方式：原型链继承、构造函数继承、组合继承、寄生组合继承。

寄生组合继承是 ES5 最佳实践：

```js
function Child(name) {
  Parent.call(this, name)
}
Child.prototype = Object.create(Parent.prototype)
Child.prototype.constructor = Child
```

现在推荐：

```js
class Child extends Parent {
  constructor(name) {
    super(name)
  }
}
```

工程中更推荐组合优于继承（Composition over Inheritance），避免过深继承层次。

**口述模板（30–60 秒）**

1. 一句话结论：现代代码用 class extends，ES5 用寄生组合继承。
2. 关键机制：实例属性靠构造函数，方法靠原型链。
3. 注意点：继承过深会增加耦合。
4. 可追问时补充：组合优于继承。

**常见追问**

- 子类构造为什么要先 super？→ 派生类实例由父类构造函数创建，super 前 this 未初始化。

---

### Q12. 属性描述符和 Object.freeze 是什么？

- **难度**: mid
- **标签**: Object

**答案**

属性描述符：

- 数据属性：`value`、`writable`、`enumerable`、`configurable`
- 访问器属性：`get`、`set`、`enumerable`、`configurable`

```js
Object.defineProperty(obj, 'id', {
  value: 1,
  writable: false,
  enumerable: true,
  configurable: false,
})
```

| 方法 | 新增 | 删除 | 修改 |
|------|------|------|------|
| `preventExtensions` | 不可 | 可 | 可 |
| `seal` | 不可 | 不可 | 可 |
| `freeze` | 不可 | 不可 | 不可 |

freeze 是浅冻结。

**口述模板（30–60 秒）**

1. 一句话结论：描述符控制属性的可写、可枚举、可配置。
2. 关键机制：freeze 将属性设为不可写不可配置。
3. 注意点：freeze 只冻结一层。
4. 可追问时补充：Vue 2 用 defineProperty 实现响应式。

**常见追问**

- 严格模式修改冻结对象会怎样？→ 抛 TypeError；非严格模式静默失败。

---

## 三、事件循环与异步

### Q13. 浏览器事件循环是什么？

- **难度**: mid
- **标签**: Event Loop

**答案**

JS 主线程同一时间只执行一个调用栈。事件循环大致流程：

1. 从任务队列取一个任务执行，如 script、定时器、事件回调、消息
2. 当前调用栈清空后，清空微任务队列
3. 浏览器在合适时机执行渲染步骤：rAF、样式、布局、绘制
4. 进入下一轮

微任务：Promise 回调、`queueMicrotask`、MutationObserver。  
任务：setTimeout、setInterval、用户事件、MessageChannel、网络回调等。

**口述模板（30–60 秒）**

1. 一句话结论：每执行完一个任务，就清空所有微任务，然后可能渲染。
2. 关键机制：微任务优先级高于下一个任务。
3. 注意点：无限微任务会阻塞渲染。
4. 可追问时补充：rAF 在渲染前执行。

**常见追问**

- 每轮一定渲染吗？→ 不一定，浏览器按刷新率和是否需要更新决定。

---

### Q14. 经典输出题：同步、Promise、setTimeout、async/await

- **难度**: mid
- **标签**: Event Loop, 输出题

**答案**

```js
console.log('1')

setTimeout(() => console.log('2'), 0)

Promise.resolve().then(() => console.log('3'))

async function run() {
  console.log('4')
  await null
  console.log('5')
}
run()

console.log('6')
```

输出：`1 4 6 3 5 2`

分析：

- 同步：1、4、6
- 微任务：Promise then 先入队输出 3；await 后续入队输出 5
- 宏任务：setTimeout 输出 2

**口述模板（30–60 秒）**

1. 一句话结论：先同步，再微任务，最后下一个任务。
2. 关键机制：await 后面的代码相当于 Promise 微任务继续执行。
3. 注意点：async 函数中 await 前的代码同步执行。
4. 可追问时补充：await thenable 可能多消耗微任务轮次。

**常见追问**

- await 一个 setTimeout Promise 呢？→ 要等定时器任务 resolve 后才继续。

---

### Q15. Promise 有哪些状态？then 链如何传递值？

- **难度**: mid
- **标签**: Promise

**答案**

三种状态：`pending`、`fulfilled`、`rejected`，状态一旦确定不可变。

`then` 返回新 Promise：

- 回调返回普通值：新 Promise fulfilled 该值
- 返回 Promise / thenable：跟随其状态
- 抛出异常：新 Promise rejected
- 未提供回调：值或错误穿透

```js
Promise.resolve(1)
  .then((x) => x + 1)
  .then((x) => { throw new Error(String(x)) })
  .catch((e) => e.message) // '2'
```

**口述模板（30–60 秒）**

1. 一句话结论：Promise 状态单向不可逆，then 返回新 Promise。
2. 关键机制：回调返回值决定下一个 Promise 状态。
3. 注意点：忘记 return 会断开链。
4. 可追问时补充：catch 是 then(undefined, onRejected)。

**常见追问**

- catch 后还能继续 then 吗？→ 能，catch 返回新 Promise。

---

### Q16. Promise.all、allSettled、race、any 区别？

- **难度**: mid
- **标签**: Promise

**答案**

| 方法 | 成功条件 | 失败条件 | 场景 |
|------|----------|----------|------|
| `all` | 全部成功 | 任一失败立即失败 | 并行依赖数据 |
| `allSettled` | 全部完成 | 不会因单个失败而失败 | 批量结果统计 |
| `race` | 第一个完成者 | 第一个失败者 | 超时控制 |
| `any` | 第一个成功 | 全部失败，AggregateError | 多源竞速 |

注意：`Promise.all` 失败后，其他请求不会自动取消，需要 AbortController。

**口述模板（30–60 秒）**

1. 一句话结论：按「是否需要全部成功」选择组合器。
2. 关键机制：all 快速失败，allSettled 收集全部结果。
3. 注意点：Promise 本身不可取消。
4. 可追问时补充：手写 all 的顺序和空数组处理。

**常见追问**

- 空数组 Promise.all 返回什么？→ 立即 fulfilled 为空数组。

---

### Q17. async/await 的本质是什么？

- **难度**: mid
- **标签**: async/await

**答案**

async/await 是基于 Promise 的语法，形式上可理解为 Generator + 自动执行器：

- async 函数总返回 Promise
- `await x` 会把 x 转为 Promise，并暂停当前 async 函数
- 后续代码在 Promise 完成后作为微任务恢复执行

```js
async function load() {
  try {
    const user = await getUser()
    return user
  } catch (error) {
    report(error)
  }
}
```

并行请求不要串行 await：

```js
const [user, orders] = await Promise.all([getUser(), getOrders()])
```

**口述模板（30–60 秒）**

1. 一句话结论：async/await 是 Promise 的同步风格写法。
2. 关键机制：await 暂停函数，后续通过微任务恢复。
3. 注意点：无依赖请求要并行。
4. 可追问时补充：错误用 try/catch 捕获。

**常见追问**

- forEach 里 await 有效吗？→ forEach 不等待回调返回的 Promise。

---

### Q18. 未处理的 Promise 异常如何捕获？

- **难度**: mid
- **标签**: 错误处理

**答案**

局部：

```js
try {
  await request()
} catch (error) {}
```

全局：

```js
window.addEventListener('unhandledrejection', (event) => {
  report(event.reason)
})

window.addEventListener('error', (event) => {
  report(event.error)
})
```

Node：`process.on('unhandledRejection')`，现代 Node 默认未处理 rejection 会使进程退出。

**口述模板（30–60 秒）**

1. 一句话结论：业务就近处理，全局做兜底上报。
2. 关键机制：unhandledrejection 捕获无 catch 的 Promise。
3. 注意点：全局监听不能替代业务恢复逻辑。
4. 可追问时补充：资源加载错误监听。

**常见追问**

- 为什么 setTimeout 里 throw 无法 try 捕获？→ 不在同一个调用栈。

---

### Q19. Generator 和迭代器协议是什么？

- **难度**: mid
- **标签**: Iterator, Generator

**答案**

迭代器协议：对象有 `next()`，返回 `{ value, done }`。  
可迭代协议：对象有 `[Symbol.iterator]()`，返回迭代器。

```js
class Range {
  constructor(start, end) {
    this.start = start
    this.end = end
  }
  *[Symbol.iterator]() {
    for (let i = this.start; i < this.end; i++) yield i
  }
}

[...new Range(1, 4)] // [1, 2, 3]
```

Generator 可暂停和恢复执行，适合惰性序列、状态机和流程控制。

**口述模板（30–60 秒）**

1. 一句话结论：Symbol.iterator 让对象可被 for...of 遍历。
2. 关键机制：next 返回 value 和 done。
3. 注意点：Generator 返回的是迭代器对象。
4. 可追问时补充：ES2025 Iterator helpers。

**常见追问**

- 提前 break 会通知迭代器吗？→ 会调用迭代器的 `return()` 方法做清理。

---

### Q20. AbortController 怎么使用？

- **难度**: mid
- **标签**: 取消, fetch

**答案**

```js
const controller = new AbortController()

fetch('/api/search', { signal: controller.signal })
  .catch((error) => {
    if (error.name === 'AbortError') return
    throw error
  })

controller.abort()
```

现代 API：

- `AbortSignal.timeout(ms)`：超时信号
- `AbortSignal.any([a, b])`：组合多个信号
- `addEventListener(type, fn, { signal })`：批量移除监听器

**口述模板（30–60 秒）**

1. 一句话结论：AbortController 是 Web 标准取消机制。
2. 关键机制：把 signal 传给可取消 API。
3. 注意点：取消错误通常不应上报为异常。
4. 可追问时补充：搜索竞态和组件卸载取消。

**常见追问**

- 超时错误名称是什么？→ `AbortSignal.timeout` 触发 `TimeoutError`。

---

### Q21. fetch 和 XMLHttpRequest 的错误处理区别？

- **难度**: mid
- **标签**: fetch

**答案**

fetch 只有网络错误、CORS 失败、取消等情况才 reject；HTTP 404、500 仍会 resolve。

```js
const response = await fetch(url)
if (!response.ok) {
  throw new Error(`HTTP ${response.status}`)
}
const data = await response.json()
```

fetch 优势：Promise API、Stream、AbortController。XHR 仍可方便获取上传进度。

**口述模板（30–60 秒）**

1. 一句话结论：fetch 不会因为 HTTP 错误状态码 reject。
2. 关键机制：要检查 response.ok。
3. 注意点：body 只能读取一次。
4. 可追问时补充：上传进度和 Stream。

**常见追问**

- body 要读两次怎么办？→ `response.clone()`。

---

### Q22. requestAnimationFrame 和 setTimeout 做动画有何区别？

- **难度**: mid
- **标签**: 渲染

**答案**

- `requestAnimationFrame`：在浏览器下次渲染前执行，与刷新率同步；后台页通常暂停
- `setTimeout`：时间不精确，可能与帧不同步导致掉帧

```js
function animate(time) {
  update(time)
  requestAnimationFrame(animate)
}
requestAnimationFrame(animate)
```

动画应基于时间差计算，而不是假定 60fps。

**口述模板（30–60 秒）**

1. 一句话结论：视觉动画优先 rAF 或 CSS 动画。
2. 关键机制：rAF 与渲染帧对齐。
3. 注意点：高刷屏不是 60fps。
4. 可追问时补充：避免在 rAF 中布局抖动。

**常见追问**

- rAF 是宏任务还是微任务？→ 都不是，属于渲染步骤中的回调。

---

## 四、模块、对象与性能基础

### Q23. ES Module 和 CommonJS 有什么区别？

- **难度**: mid
- **标签**: 模块化

**答案**

| | ESM | CommonJS |
|--|-----|----------|
| 语法 | import/export | require/module.exports |
| 加载 | 静态分析，异步加载 | 运行时同步加载 |
| 导出 | 实时绑定（live binding） | 导出值的拷贝/对象引用 |
| this | undefined | module.exports |
| Tree Shaking | 友好 | 困难 |

现代 Node 支持 `require(esm)`（同步 ESM，无顶层 await），但新项目优先 ESM。

**口述模板（30–60 秒）**

1. 一句话结论：ESM 是静态标准模块，CJS 是 Node 运行时模块。
2. 关键机制：ESM 导出是实时绑定。
3. 注意点：CJS 动态 require 不利于 Tree Shaking。
4. 可追问时补充：循环依赖差异。

**常见追问**

- ESM 中如何获得 `__dirname`？→ Node 新版本可用 `import.meta.dirname`，或通过 `import.meta.url` 计算。

---

### Q24. Tree Shaking 生效条件是什么？

- **难度**: mid
- **标签**: 构建优化

**答案**

条件：

- 使用 ESM 静态导入导出
- 构建工具开启生产优化
- 代码无副作用，或 package.json 正确声明 `sideEffects`
- 避免 `import * as` 后动态访问
- Babel 等工具不要提前转成 CommonJS

```json
{
  "sideEffects": ["*.css"]
}
```

**口述模板（30–60 秒）**

1. 一句话结论：Tree Shaking 依赖 ESM 静态结构和无副作用。
2. 关键机制：构建器标记未使用导出并删除。
3. 注意点：CSS 导入通常有副作用。
4. 可追问时补充：按需引入和动态 import。

**常见追问**

- 函数调用为什么可能无法删除？→ 构建器无法确认其无副作用，可用 `/*#__PURE__*/` 标注。

---

### Q25. 浅拷贝和深拷贝如何实现？

- **难度**: mid
- **标签**: 拷贝

**答案**

- 浅拷贝：`{ ...obj }`、`Object.assign`、`arr.slice()`
- 深拷贝：
  - `structuredClone`：标准方案，支持循环引用、Map、Set、Date
  - JSON 方法：只适合纯 JSON 数据
  - 手写递归：需处理循环引用、特殊对象、Symbol 键、原型

**口述模板（30–60 秒）**

1. 一句话结论：优先 structuredClone，特殊需求再手写。
2. 关键机制：深拷贝递归复制嵌套引用。
3. 注意点：循环引用要 WeakMap 缓存。
4. 可追问时补充：不可变数据结构共享。

**常见追问**

- 深拷贝有什么性能问题？→ 大对象复制成本高，状态管理更常用结构共享。

---

### Q26. WeakMap 和 WeakSet 有什么作用？

- **难度**: mid
- **标签**: WeakMap

**答案**

WeakMap 的键必须是对象（或非注册 Symbol），且对键是弱引用：键不再被其他地方引用时，可被 GC 回收。

应用：

- 给 DOM 节点或对象挂私有元数据
- 缓存对象计算结果
- 深拷贝记录访问过的对象

```js
const cache = new WeakMap()
function getMeta(obj) {
  if (!cache.has(obj)) cache.set(obj, compute(obj))
  return cache.get(obj)
}
```

WeakMap 不可遍历，也没有 size。

**口述模板（30–60 秒）**

1. 一句话结论：WeakMap 用于不阻止对象回收的关联数据。
2. 关键机制：对键弱引用，对值强引用。
3. 注意点：不可遍历，无法预测回收时机。
4. 可追问时补充：WeakRef 和 FinalizationRegistry。

**常见追问**

- 值引用键会阻止回收吗？→ WeakMap 采用 ephemeron 语义，值引用键不会单独阻止回收。

---

### Q27. 防抖和节流的区别？

- **难度**: mid
- **标签**: 性能

**答案**

- **防抖（Debounce）**：停止触发一段时间后才执行，适合搜索联想、窗口 resize 结束后计算
- **节流（Throttle）**：固定时间内最多执行一次，适合滚动、拖拽、埋点

关键细节：

- 保留 this 和参数
- 支持 leading / trailing
- 支持 cancel / flush
- 组件卸载时取消

**口述模板（30–60 秒）**

1. 一句话结论：防抖等停下来，节流控频率。
2. 关键机制：防抖重置定时器，节流记录上次执行时间。
3. 注意点：要处理 this、参数和取消。
4. 可追问时补充：搜索还要处理请求竞态。

**常见追问**

- 滚动动画用节流还是 rAF？→ 视觉更新优先 rAF。

---

### Q28. 垃圾回收的基本原理是什么？

- **难度**: mid
- **标签**: GC

**答案**

JS 自动内存管理基于可达性（Reachability）：从根对象（全局对象、调用栈、闭包等）出发能访问到的对象被保留，否则可回收。

常见算法：

- 标记清除（Mark-Sweep）
- 引用计数：无法处理循环引用，现代引擎不作为主要策略

常见泄漏：

- 全局变量
- 未清除的定时器
- 未移除的事件监听器
- 闭包持有大对象
- 已删除 DOM 仍被引用
- 无上限缓存

**口述模板（30–60 秒）**

1. 一句话结论：GC 回收不可达对象。
2. 关键机制：从根出发标记可达对象。
3. 注意点：可达但无用的对象就是泄漏。
4. 可追问时补充：V8 分代回收。

**常见追问**

- 循环引用会泄漏吗？→ 如果整个循环不可达，标记清除可以回收。

---

### Q29. Getter、Setter 和 Proxy 有什么区别？

- **难度**: mid
- **标签**: 元编程

**答案**

- getter/setter：定义单个属性的读取和赋值逻辑
- Proxy：拦截整个对象的操作，包括 get、set、has、deleteProperty、ownKeys 等

Vue 2 使用 `Object.defineProperty`，无法直接监听新增属性和数组索引；Vue 3 使用 Proxy，可拦截更多操作。

**口述模板（30–60 秒）**

1. 一句话结论：getter/setter 管属性，Proxy 管对象操作。
2. 关键机制：Proxy 使用 trap 拦截。
3. 注意点：Proxy 不等于原对象。
4. 可追问时补充：Vue 3 响应式原理。

**常见追问**

- Proxy 能拦截私有字段吗？→ 不能，私有字段直接作用于目标对象内部槽。

---

### Q30. 数组排序需要注意什么？

- **难度**: mid
- **标签**: Array

**答案**

- 默认按字符串排序
- `sort` 原地修改
- 现代规范要求 `Array.prototype.sort` 稳定排序
- 比较函数必须一致，返回负数、0、正数

```js
users.toSorted((a, b) => a.age - b.age || a.name.localeCompare(b.name))
```

中文排序使用 `Intl.Collator`：

```js
const collator = new Intl.Collator('zh-CN')
names.toSorted(collator.compare)
```

**口述模板（30–60 秒）**

1. 一句话结论：数字要传比较函数，状态数据用 toSorted。
2. 关键机制：sort 是原地修改。
3. 注意点：比较函数不能随机返回。
4. 可追问时补充：Intl.Collator 多语言排序。

**常见追问**

- 随机打乱用 sort(Math.random) 可以吗？→ 不均匀，应使用 Fisher-Yates 洗牌。

---

### Q31. 如何避免对象属性访问报错和数据不一致？

- **难度**: mid
- **标签**: 健壮性

**答案**

- 在边界层（API 响应）做数据校验，如 Zod / Valibot
- 内部数据使用明确类型
- 访问可选数据使用 `?.` 和 `??`
- 对必需数据缺失要显式报错，而不是静默兜底

```js
const user = UserSchema.parse(await response.json())
```

**口述模板（30–60 秒）**

1. 一句话结论：外部数据边界校验，内部依赖类型约束。
2. 关键机制：运行时校验弥补 TypeScript 编译期限制。
3. 注意点：可选链不要掩盖错误。
4. 可追问时补充：错误上报。

**常见追问**

- TypeScript 能保证接口数据正确吗？→ 不能，类型在运行时被擦除。

---

### Q32. 如何判断一个函数是纯函数？

- **难度**: mid
- **标签**: 函数式编程

**答案**

纯函数：

- 相同输入始终相同输出
- 不产生副作用：不修改外部变量、不请求网络、不操作 DOM

```js
const add = (a, b) => a + b // 纯
const push = (arr, x) => arr.push(x) // 不纯
```

纯函数更容易测试、缓存和并发处理。React render、reducer 也要求接近纯函数。

**口述模板（30–60 秒）**

1. 一句话结论：同输入同输出，且无副作用。
2. 关键机制：依赖只来自参数。
3. 注意点：Date.now、Math.random 让结果不确定。
4. 可追问时补充：副作用集中管理。

**常见追问**

- reducer 为什么不能修改 state？→ 依赖引用比较判断变化，原地修改会破坏更新和时间旅行调试。

---

### Q33. 事件监听为什么要清理？怎么批量清理？

- **难度**: mid
- **标签**: DOM, 内存

**答案**

监听器会引用回调闭包，闭包可能持有组件状态和 DOM，导致内存泄漏或重复执行。

```js
const controller = new AbortController()

window.addEventListener('resize', onResize, { signal: controller.signal })
document.addEventListener('keydown', onKeydown, { signal: controller.signal })

controller.abort() // 批量移除
```

`removeEventListener` 需要传同一个函数引用和相同 capture 选项。

**口述模板（30–60 秒）**

1. 一句话结论：组件销毁时必须清理全局监听。
2. 关键机制：监听器和闭包保持对象可达。
3. 注意点：匿名函数无法 remove。
4. 可追问时补充：AbortSignal 批量清理。

**常见追问**

- passive 监听器有什么用？→ 声明不会 preventDefault，浏览器可更快滚动。

---

### Q34. 什么是柯里化（Currying）？

- **难度**: mid
- **标签**: 函数式编程

**答案**

把多参数函数转换为一系列单参数或部分参数函数：

```js
const add = (a) => (b) => (c) => a + b + c
add(1)(2)(3) // 6
```

应用：

- 参数复用
- 函数组合
- 配置预设

工程中常见的是部分应用（Partial Application），例如 `bind` 预置参数。

**口述模板（30–60 秒）**

1. 一句话结论：柯里化把参数分步传入。
2. 关键机制：闭包保存已收集参数。
3. 注意点：依赖 fn.length 时，默认参数和 rest 参数会影响。
4. 可追问时补充：手写 curry。

**常见追问**

- 柯里化有性能代价吗？→ 有额外闭包和函数调用，热点路径不应滥用。
