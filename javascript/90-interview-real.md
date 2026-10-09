# JavaScript 真实面试题汇总

> 根据近年公开面经、社区高频题和大厂前端面试趋势整理。`来源倾向` 表示题目常见场景，不代表某公司固定题库。

---

### R1. 说说闭包，以及闭包在项目中的实际应用

- **难度**: mid
- **标签**: 闭包
- **来源倾向**: 国内大厂一面高频

**答案**

闭包是函数和其词法环境的组合，可用于私有状态、函数工厂、防抖节流和缓存。项目中要注意长期持有大对象导致泄漏。

详见：[20-mid-qa Q3](20-mid-qa.md)

---

### R2. 解释 this 绑定规则，并说出输出

```js
const obj = {
  name: 'obj',
  normal() { return this.name },
  arrow: () => this?.name,
}
const fn = obj.normal
console.log(obj.normal(), fn())
```

- **难度**: mid
- **标签**: this
- **来源倾向**: 基础面试输出题

**答案**

`obj.normal()` 输出 `obj`。`fn()` 普通调用，在严格模式或模块中 this 为 undefined，会抛 TypeError；非严格脚本中 this 为全局对象，通常得到 `window.name`。

详见：[20-mid-qa Q6](20-mid-qa.md)

---

### R3. 原型链是什么？`Function` 和 `Object` 的原型关系怎么画？

- **难度**: mid
- **标签**: 原型
- **来源倾向**: 原理面高频

**答案**

```text
Object.__proto__ === Function.prototype
Function.__proto__ === Function.prototype
Function.prototype.__proto__ === Object.prototype
Object.prototype.__proto__ === null
```

详见：[20-mid-qa Q9](20-mid-qa.md)

---

### R4. 事件循环输出题

```js
async function async1() {
  console.log('async1 start')
  await async2()
  console.log('async1 end')
}
async function async2() {
  console.log('async2')
}
console.log('script start')
setTimeout(() => console.log('setTimeout'), 0)
async1()
new Promise((resolve) => {
  console.log('promise1')
  resolve()
}).then(() => console.log('promise2'))
console.log('script end')
```

- **难度**: mid
- **标签**: Event Loop
- **来源倾向**: 头部互联网一面经典题

**答案**

```text
script start
async1 start
async2
promise1
script end
async1 end
promise2
setTimeout
```

现代引擎优化后，`await` 原生 Promise 不再额外多两轮微任务，所以 `async1 end` 在 `promise2` 前。

详见：[20-mid-qa Q14](20-mid-qa.md)

---

### R5. 手写 Promise，至少实现 then 链

- **难度**: senior
- **标签**: Promise
- **来源倾向**: 中高级手写高频

**答案**

重点讲状态不可逆、then 返回新 Promise、回调微任务执行、thenable 解析和循环引用检测。

详见：[40-coding-handwritten H7](40-coding-handwritten.md)

---

### R6. 手写 Promise.all，并说明和 allSettled 区别

- **难度**: mid
- **标签**: Promise
- **来源倾向**: 手写题高频

**答案**

Promise.all 快速失败，allSettled 等待全部完成。实现时按索引存结果，处理空数组和普通值。

详见：[40-coding-handwritten H6](40-coding-handwritten.md)

---

### R7. 实现一个并发数为 2 的调度器

- **难度**: mid
- **标签**: 并发
- **来源倾向**: 字节系等前端面试常见题型

**答案**

维护任务队列和 running 计数。任务结束后释放并发槽并继续调度。

详见：[40-coding-handwritten H8](40-coding-handwritten.md)

---

### R8. 防抖和节流的区别，手写一个

- **难度**: mid
- **标签**: 性能
- **来源倾向**: 基础手写必考

**答案**

防抖等停止触发后执行，节流固定频率执行。要保留 this、参数，并支持取消。

详见：[40-coding-handwritten H1](40-coding-handwritten.md)、[H2](40-coding-handwritten.md)

---

### R9. 深拷贝怎么实现？循环引用怎么处理？

- **难度**: mid
- **标签**: 拷贝
- **来源倾向**: 高频手写

**答案**

使用 WeakMap 记录原对象和拷贝结果，在递归前登记。现代业务优先 structuredClone。

详见：[40-coding-handwritten H3](40-coding-handwritten.md)

---

### R10. 手写 bind，bind 后 new 会怎样？

- **难度**: mid
- **标签**: this
- **来源倾向**: 原理手写题

**答案**

new 调用时忽略绑定 this，但保留预置参数；实现可用 `new.target` 判断，并用 Reflect.construct 构造。

详见：[40-coding-handwritten H4](40-coding-handwritten.md)

---

### R11. ESM 和 CommonJS 有什么区别？循环依赖如何表现？

- **难度**: senior
- **标签**: 模块化
- **来源倾向**: 工程化和 Node 方向

**答案**

ESM 静态结构、实时绑定，适合 Tree Shaking；CJS 运行时同步加载。循环依赖中 CJS 可能拿到部分导出，ESM 可能触发 TDZ。

详见：[20-mid-qa Q23](20-mid-qa.md)、[30-senior-qa Q13](30-senior-qa.md)

---

### R12. 浏览器和 Node 的事件循环有什么不同？

- **难度**: senior
- **标签**: Node, Event Loop
- **来源倾向**: 全栈 / Node 方向

**答案**

Node 基于 libuv 阶段模型，有 timers、poll、check 等阶段；`process.nextTick` 优先于 Promise 微任务。

详见：[30-senior-qa Q9](30-senior-qa.md)

---

### R13. V8 垃圾回收原理？如何排查内存泄漏？

- **难度**: senior
- **标签**: GC, 性能
- **来源倾向**: 高级岗原理与排障

**答案**

新生代 Scavenge，老生代标记清除和整理。排查时反复操作并对比 Heap Snapshot，重点看 Detached DOM 和 Retainers。

详见：[30-senior-qa Q3](30-senior-qa.md)、[Q4](30-senior-qa.md)

---

### R14. 什么是 Hidden Class？怎样写代码更利于 V8 优化？

- **难度**: senior
- **标签**: V8
- **来源倾向**: 高级性能面试

**答案**

保持对象形状稳定、字段初始化顺序一致、避免热点对象 delete 属性，优化前先 profile。

详见：[30-senior-qa Q2](30-senior-qa.md)

---

### R15. 如何解决搜索请求竞态问题？

- **难度**: senior
- **标签**: 异步
- **来源倾向**: 业务场景题

**答案**

取消旧请求，并使用请求版本号兜底丢弃过期响应；还可借助请求缓存库。

详见：[30-senior-qa Q10](30-senior-qa.md)

---

### R16. 如何设计前端错误监控？

- **难度**: senior
- **标签**: 监控
- **来源倾向**: 高级前端系统设计

**答案**

全局错误、Promise rejection、资源错误、请求错误和框架错误统一采集；脱敏、采样、去重后上报，服务端用 source map 还原。

详见：[50-system-design S2](50-system-design.md)

---

### R17. Vue 3 为什么用 Proxy 替代 defineProperty？

- **难度**: senior
- **标签**: Proxy, Vue
- **来源倾向**: Vue 技术栈高频

**答案**

Proxy 可以拦截新增、删除、in、遍历等更多操作，数组和动态属性支持更好，并可懒代理嵌套对象。

详见：[30-senior-qa Q11](30-senior-qa.md)

---

### R18. 什么是原型污染？怎么防？

- **难度**: senior
- **标签**: 安全
- **来源倾向**: 安全意识追问

**答案**

恶意 `__proto__` 等键修改原型，影响所有对象。合并和路径设置时过滤危险键，字典使用 Map 或 null 原型对象。

详见：[30-senior-qa Q16](30-senior-qa.md)、[40-coding-handwritten H15](40-coding-handwritten.md)

---

### R19. `0.1 + 0.2` 为什么不等于 `0.3`？大数字 ID 怎么处理？

- **难度**: junior
- **标签**: Number
- **来源倾向**: 基础 + 业务数据问题

**答案**

IEEE 754 精度误差；超安全整数 ID 用字符串传输，大整数计算用 BigInt。

详见：[10-junior-qa Q6](10-junior-qa.md)、[30-senior-qa Q19](30-senior-qa.md)

---

### R20. 手写 EventEmitter，支持 once

- **难度**: mid
- **标签**: 发布订阅
- **来源倾向**: 手写高频

**答案**

once 包装函数并在执行前 off；emit 遍历监听器快照。

详见：[40-coding-handwritten H10](40-coding-handwritten.md)

---

### R21. 实现 LRU 缓存

- **难度**: mid
- **标签**: 数据结构
- **来源倾向**: 算法与前端缓存结合

**答案**

利用 Map 插入顺序；get 命中时删除再插入，超容量时删除第一个 key。

详见：[40-coding-handwritten H11](40-coding-handwritten.md)

---

### R22. 实现 Koa compose 洋葱模型

- **难度**: senior
- **标签**: 中间件
- **来源倾向**: Node / 工程化方向

**答案**

递归 dispatch 下一个中间件，用 index 检测 next 重复调用，并统一返回 Promise。

详见：[40-coding-handwritten H14](40-coding-handwritten.md)

---

### R23. 长任务如何优化？INP 差怎么排查？

- **难度**: senior
- **标签**: 性能
- **来源倾向**: 性能优化专项

**答案**

用 Performance 面板找长任务，优先拆分任务、让出主线程、Worker 化计算、减少 DOM 和布局成本。

详见：[30-senior-qa Q7](30-senior-qa.md)

---

### R24. ES2025 以后你关注了哪些新特性？能否用于生产？

- **难度**: senior
- **标签**: 新特性
- **来源倾向**: 高级岗技术视野

**答案**

可以讲 Iterator helpers、Set 方法、Promise.try、Map.getOrInsert、Temporal、using。生产使用前确认 browserslist、Node 版本和 polyfill 成本。

详见：[30-senior-qa Q21](30-senior-qa.md)、[Q22](30-senior-qa.md)、[Q23](30-senior-qa.md)、[Q24](30-senior-qa.md)

---

### R25. 大文件上传如何设计断点续传和秒传？

- **难度**: senior
- **标签**: 系统设计
- **来源倾向**: 业务系统设计高频

**答案**

Worker 增量计算 hash，服务端查询已上传分片，前端并发上传剩余分片，最后合并校验。

详见：[50-system-design S8](50-system-design.md)

---

### R26. 实现带重试、超时、取消的请求函数

- **难度**: senior
- **标签**: fetch
- **来源倾向**: 网络层手写 / 场景题

**答案**

组合用户 signal 和超时 signal；4xx 不重试，5xx 和网络错误使用指数退避；非幂等请求不能盲目重试。

详见：[40-coding-handwritten H9](40-coding-handwritten.md)
