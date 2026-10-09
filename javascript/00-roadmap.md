# JavaScript 学习路线（高级面试向）

> 目标：能在国内互联网高级前端 / 全栈面试中讲清语言机制（作用域、闭包、this、原型、异步、事件循环、内存），能手写高频工具函数和 Promise 相关题，并能把语言知识落到性能、稳定性和工程架构上。

## 面试画像

- **岗位侧重**：前端 / 全栈 / Node；高级岗位会把 JS 与浏览器、框架原理、性能和工程化串着问
- **假定基础**：写过业务代码，用过 React / Vue 任一框架，了解基本 DOM 操作
- **学完应能**：
  - 讲清执行上下文、作用域链、闭包、`this` 绑定规则和原型链
  - 准确判断事件循环输出顺序，并解释浏览器与 Node 的差异
  - 手写防抖节流、深拷贝、`call`/`bind`、`new`、Promise 组合方法、并发控制、EventEmitter
  - 定位内存泄漏、长任务、竞态请求等线上问题
  - 了解 ES2025 / ES2026 新能力，以及 Temporal、`using` 等即将进入标准的特性

## 版本说明（本文默认语境）

- 语言规范以 **ECMAScript（TC39）** 为准，运行环境以现代浏览器和 Node 22+ 为主
- **ES2025**：Iterator helpers、Set 集合运算方法、`Promise.try`、`RegExp.escape`、JSON modules / import attributes、`Float16Array`
- **ES2026**：`Array.fromAsync`、`Error.isError`、`Math.sumPrecise`、`Uint8Array` Base64/Hex、`Iterator.concat`、`Map.prototype.getOrInsert`、JSON.parse source access
- **已到 Stage 4、计划进入 ES2027**：Temporal、显式资源管理 `using` / `await using`
- 「进入规范」不等于「所有运行时都支持」，生产使用前查 MDN / caniuse，必要时 polyfill

## 知识地图

- **语言基础**
  - 数据类型、类型判断、类型转换、相等比较
  - `var` / `let` / `const`、暂时性死区（TDZ）
  - 函数声明 / 表达式 / 箭头函数
  - 对象、数组、解构、展开、可选链、空值合并
- **执行机制**
  - 执行上下文、词法环境、作用域链
  - 闭包与内存
  - `this` 绑定规则
  - 原型、原型链、`class`、继承
- **异步**
  - 回调、Promise、async/await、Generator
  - 事件循环：任务、微任务、渲染时机
  - Promise 组合：`all` / `allSettled` / `race` / `any`
  - 取消与超时：`AbortController`
- **内存与性能**
  - 垃圾回收（标记清除、分代）
  - 内存泄漏场景、`WeakMap` / `WeakRef`
  - 长任务、Web Worker、调度
- **模块与工程**
  - ES Modules vs CommonJS
  - Tree Shaking、动态 `import()`
  - 错误处理、严格模式
- **元编程**
  - `Proxy` / `Reflect`
  - `Symbol`、迭代器协议、属性描述符
- **现代特性**
  - Iterator helpers、Set 方法、`Promise.try`、`structuredClone`
  - `Map.getOrInsert`、`Array.fromAsync`
  - Temporal、`using`（即将标准化）

## 建议阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 类型、作用域、函数、对象数组 API
2. [中级 Q&A](20-mid-qa.md) — 闭包、this、原型、Promise、事件循环、模块
3. [高级 Q&A](30-senior-qa.md) — 引擎、GC、内存泄漏、并发、元编程、架构取舍
4. [手写与现场编码](40-coding-handwritten.md) — 15 道高频手写题
5. [系统设计与场景](50-system-design.md) — 请求层、埋点、调度器、错误监控等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. 边学边勾 [PROGRESS.md](PROGRESS.md)

## 高级面试必打点

1. JS 是单线程执行 + 事件循环调度，不是「只能做一件事」
2. 闭包 = 函数 + 其创建时的词法环境；风险是持有不再需要的大对象
3. `this` 由调用方式决定：`new` > 显式绑定 > 隐式绑定 > 默认绑定；箭头函数没有自己的 `this`
4. 原型链查找到 `null` 为止；`class` 是原型继承的语法与语义增强
5. 一个任务执行完，会清空全部微任务，再进入后续任务和渲染机会
6. `await` 之后的代码相当于在微任务中继续执行
7. Promise 状态不可逆；`then` 总返回新 Promise；未处理的 rejection 要监控
8. `Promise.all` 一个失败即失败，`allSettled` 等全部结束，`any` 一个成功即成功
9. 并发控制、取消、超时、重试是请求层高级题核心
10. `==` 的隐式转换规则复杂，工程中统一用 `===`，判断空值用 `== null` 是少数例外
11. 深拷贝要处理循环引用和特殊对象；能用 `structuredClone` 时优先它
12. 内存泄漏常见于全局变量、未清理的监听器 / 定时器、闭包缓存、分离 DOM
13. ESM 是静态结构，利于 Tree Shaking；CJS 是运行时加载
14. `Proxy` 是 Vue 3 响应式基础；`Reflect` 用于保持默认语义
15. 长任务阻塞交互，要拆分任务或放 Worker
16. 浮点数精度问题来自 IEEE 754，金额用整数分或十进制库
17. 新特性按「规范状态 + 运行时支持 + 降级成本」判断能否上生产
18. 高级回答要有边界：时间复杂度、错误处理、取消、并发和可测试性

## 关联技术

- **浏览器**：DOM 事件、渲染流水线、Web API、Worker
- **Node.js**：libuv 事件循环、Stream、Buffer
- **TypeScript**：类型系统建立在 JS 运行时语义之上
- **React / Vue**：闭包陷阱、响应式（Proxy）、批量更新与微任务
- **性能**：INP、长任务、内存分析

## 自测清单（学完打勾）

- [ ] 能画出执行上下文、作用域链和闭包的关系
- [ ] 能稳定答对 Promise + setTimeout + async/await 混合输出题
- [ ] 能手写 Promise 并发池、带取消的请求、深拷贝和 EventEmitter
- [ ] 能用 DevTools 定位一次内存泄漏
- [ ] 能说出 ES2025 / ES2026 的 3 个特性，并说明生产使用策略
