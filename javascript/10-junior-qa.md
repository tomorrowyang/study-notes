# JavaScript 初级 Q&A（能用）

> 目标：类型、变量、函数、对象和数组 API 用对，并能解释常见坑的原因。

---

## 一、类型与比较

### Q1. JavaScript 有哪些数据类型？

- **难度**: junior
- **标签**: 类型

**答案**

8 种类型：

- **原始类型（Primitive）**：`undefined`、`null`、`boolean`、`number`、`bigint`、`string`、`symbol`
- **对象类型（Object）**：普通对象、数组、函数、Date、Map、Set 等都属于对象

原始值不可变；对象按引用访问。函数是可调用对象。

**口述模板（30–60 秒）**

1. 一句话结论：7 种原始类型 + Object。
2. 关键机制：原始值不可变，对象通过引用共享。
3. 注意点：数组和函数都属于对象。
4. 可追问时补充：BigInt 和 Symbol 的使用场景。

**常见追问**

- 字符串方法为什么能调用？→ 访问属性时会临时包装成 String 对象。

---

### Q2. 如何判断数据类型？`typeof null` 为什么是 `"object"`？

- **难度**: junior
- **标签**: 类型判断

**答案**

| 方法 | 适用 |
|------|------|
| `typeof` | 原始类型和函数；`null` 返回 `"object"` 是历史遗留 |
| `Array.isArray` | 判断数组 |
| `instanceof` | 判断原型链，跨 iframe / realm 可能失效 |
| `Object.prototype.toString.call(x)` | 通用类型标签，如 `[object Date]` |

```js
Object.prototype.toString.call(null) // "[object Null]"
Object.prototype.toString.call([])   // "[object Array]"
```

`typeof null` 来自早期实现中对象类型标签为 0、`null` 也表示为全 0 的历史 bug，为兼容保留。

**口述模板（30–60 秒）**

1. 一句话结论：typeof 判原始值，Array.isArray 判数组，toString 做通用判断。
2. 关键机制：instanceof 查原型链。
3. 注意点：`typeof null === 'object'`。
4. 可追问时补充：`Symbol.toStringTag` 可影响 toString 结果。

**常见追问**

- `typeof function(){}`？→ `"function"`。

---

### Q3. 原始值和引用值有什么区别？JS 是值传递还是引用传递？

- **难度**: junior
- **标签**: 内存模型

**答案**

JS 函数参数都是**按值传递**。对于对象，传递的是「引用的副本」（常称 Call by Sharing）：

```js
function update(obj) {
  obj.name = 'B'  // 修改共享对象，外部可见
  obj = { name: 'C' } // 只改变局部变量指向，外部不受影响
}
const user = { name: 'A' }
update(user)
user.name // 'B'
```

**口述模板（30–60 秒）**

1. 一句话结论：JS 只有值传递，对象传的是引用值的副本。
2. 关键机制：修改属性会影响共享对象，重新赋值不会影响外部。
3. 注意点：React 状态更新要新对象，避免原地修改。
4. 可追问时补充：浅拷贝和深拷贝。

**常见追问**

- `const obj = {}` 能改属性吗？→ 能，const 只保证绑定不变。

---

### Q4. `==` 和 `===` 有什么区别？`Object.is` 呢？

- **难度**: junior
- **标签**: 相等比较

**答案**

- `===`：严格相等，不做类型转换
- `==`：宽松相等，会进行隐式类型转换
- `Object.is`：类似 `===`，但 `Object.is(NaN, NaN)` 为 true，`Object.is(+0, -0)` 为 false

```js
0 == ''        // true
null == undefined // true
NaN === NaN    // false
Object.is(NaN, NaN) // true
```

工程中统一用 `===`；`x == null` 可同时判断 `null` 和 `undefined`。

**口述模板（30–60 秒）**

1. 一句话结论：默认用 `===`。
2. 关键机制：`==` 会触发类型转换规则。
3. 注意点：NaN 不等于自身。
4. 可追问时补充：`== null` 的少数合理用法。

**常见追问**

- `[] == ![]` 为什么是 true？→ `![]` 为 false，转成 0；`[]` 转原始值为 `''` 再转 0，所以 `0 == 0`。

---

### Q5. 哪些值是假值（Falsy）？

- **难度**: junior
- **标签**: 类型转换

**答案**

假值：`false`、`0`、`-0`、`0n`、`''`、`null`、`undefined`、`NaN`（以及历史上的 `document.all`）。

其他都是真值，包括 `'0'`、`'false'`、`[]`、`{}`。

```js
const count = 0
count || 10 // 10，可能不是你想要的
count ?? 10 // 0
```

**口述模板（30–60 秒）**

1. 一句话结论：除少数假值外都是真值。
2. 关键机制：条件判断会转换为布尔值。
3. 注意点：空数组和空对象是真值。
4. 可追问时补充：`??` 只处理 null 和 undefined。

**常见追问**

- 表单默认值为什么用 `??`？→ 避免把 0 和空字符串误替换。

---

### Q6. 为什么 `0.1 + 0.2 !== 0.3`？如何处理？

- **难度**: junior
- **标签**: Number

**答案**

`number` 使用 IEEE 754 双精度浮点数，0.1 和 0.2 无法用二进制精确表示，相加后产生误差。

处理：

- 比较时使用误差范围：`Math.abs(a - b) < Number.EPSILON`
- 金额用整数「分」计算
- 精确十进制计算用 decimal.js 等库
- 超大整数用 `BigInt`

**口述模板（30–60 秒）**

1. 一句话结论：二进制浮点数无法精确表示部分十进制小数。
2. 关键机制：IEEE 754 存储有精度误差。
3. 注意点：金额不要直接用浮点数计算。
4. 可追问时补充：`Number.MAX_SAFE_INTEGER`。

**常见追问**

- `toFixed` 能解决吗？→ 只适合展示，且有舍入问题，不适合金额计算。

---

### Q7. `isNaN` 和 `Number.isNaN` 区别？

- **难度**: junior
- **标签**: Number

**答案**

- `isNaN(value)`：先转数字，再判断，`isNaN('abc')` 为 true
- `Number.isNaN(value)`：不转换，只判断值本身是否为 NaN

```js
isNaN('abc')        // true
Number.isNaN('abc') // false
```

**口述模板（30–60 秒）**

1. 一句话结论：Number.isNaN 更精确。
2. 关键机制：全局 isNaN 会类型转换。
3. 注意点：NaN 不等于自身。
4. 可追问时补充：`Number.isFinite` 同理。

**常见追问**

- 怎么判断安全整数？→ `Number.isSafeInteger`。

---

## 二、变量与函数

### Q8. `var`、`let`、`const` 有什么区别？

- **难度**: junior
- **标签**: 变量

**答案**

| | var | let | const |
|--|-----|-----|-------|
| 作用域 | 函数作用域 | 块级作用域 | 块级作用域 |
| 提升 | 提升并初始化为 undefined | 提升但处于 TDZ | 提升但处于 TDZ |
| 重复声明 | 允许 | 不允许 | 不允许 |
| 重新赋值 | 允许 | 允许 | 不允许 |

TDZ（Temporal Dead Zone，暂时性死区）：声明前访问 let/const 会抛 ReferenceError。

**口述模板（30–60 秒）**

1. 一句话结论：默认 const，需要重新赋值用 let，避免 var。
2. 关键机制：let/const 是块级作用域且有 TDZ。
3. 注意点：const 不代表对象不可变。
4. 可追问时补充：循环中 var 闭包问题。

**常见追问**

- 全局 var 和 let 有何不同？→ 全局 var 会成为 `globalThis` 属性，全局 let 不会。

---

### Q9. 什么是变量提升（Hoisting）？

- **难度**: junior
- **标签**: 执行机制

**答案**

代码执行前，引擎会先处理声明：

- 函数声明整体提升
- `var` 声明提升并初始化为 undefined
- `let`、`const`、`class` 也被登记，但在初始化前不可访问

```js
console.log(a) // undefined
var a = 1

foo() // 可调用
function foo() {}

console.log(b) // ReferenceError
let b = 2
```

**口述模板（30–60 秒）**

1. 一句话结论：声明在执行前被登记，但初始化时机不同。
2. 关键机制：创建执行上下文时处理声明。
3. 注意点：let/const 有 TDZ。
4. 可追问时补充：函数表达式不会整体提升。

**常见追问**

- 函数和变量同名谁优先？→ 函数声明先登记，var 同名不会覆盖已存在的函数绑定，赋值执行后才覆盖。

---

### Q10. 函数声明、函数表达式、箭头函数有什么区别？

- **难度**: junior
- **标签**: 函数

**答案**

- **函数声明**：整体提升
- **函数表达式**：遵循变量提升规则
- **箭头函数**：
  - 没有自己的 `this`、`arguments`、`super`、`new.target`
  - 不能作为构造函数
  - 没有 `prototype`

```js
const obj = {
  value: 1,
  normal() { return this.value },
  arrow: () => this.value,
}
```

**口述模板（30–60 秒）**

1. 一句话结论：箭头函数最大的区别是词法 this。
2. 关键机制：this 继承外层作用域。
3. 注意点：对象方法和构造函数不要用箭头函数。
4. 可追问时补充：回调中箭头函数很方便。

**常见追问**

- 箭头函数能 call 改 this 吗？→ 不能，call/apply/bind 只能传参。

---

### Q11. 默认参数、剩余参数、展开运算符怎么用？

- **难度**: junior
- **标签**: 函数

**答案**

```js
function request(url, { method = 'GET', timeout = 5000 } = {}) {}

function sum(...nums) {
  return nums.reduce((a, b) => a + b, 0)
}

const next = { ...prev, name: 'new' }
const merged = [...a, ...b]
```

- 默认参数只在实参为 `undefined` 时生效
- 剩余参数是真数组，替代 `arguments`
- 展开是浅拷贝

**口述模板（30–60 秒）**

1. 一句话结论：默认参数处理缺省，rest 收集参数，spread 展开值。
2. 关键机制：默认值只对 undefined 生效。
3. 注意点：展开只做浅拷贝。
4. 可追问时补充：参数解构要给默认空对象。

**常见追问**

- 传 `null` 会用默认值吗？→ 不会。

---

### Q12. 解构赋值有哪些常见用法和坑？

- **难度**: junior
- **标签**: 解构

**答案**

```js
const { name, age = 18, profile: { city } = {} } = user
const [first, , third = 0] = list
;[a, b] = [b, a]
```

坑：

- 解构 `null` / `undefined` 会报错
- 嵌套解构要提供默认对象
- 以 `[` 开头的语句前缺分号可能出问题

**口述模板（30–60 秒）**

1. 一句话结论：解构用于按结构提取值。
2. 关键机制：默认值同样只对 undefined 生效。
3. 注意点：源对象可能为空时要兜底。
4. 可追问时补充：交换变量。

**常见追问**

- 怎么重命名？→ `const { name: userName } = user`。

---

### Q13. 可选链 `?.` 和空值合并 `??` 怎么用？

- **难度**: junior
- **标签**: ES2020

**答案**

```js
const city = user?.profile?.city
const result = callback?.()
const pageSize = options.pageSize ?? 20
```

- `?.`：左侧为 null / undefined 时短路返回 undefined
- `??`：左侧为 null / undefined 时取右侧，不会误处理 0、false、空字符串

不能把可选链用于赋值左侧：`obj?.a = 1` 非法。

**口述模板（30–60 秒）**

1. 一句话结论：`?.` 安全访问，`??` 处理空值默认。
2. 关键机制：只针对 null 和 undefined。
3. 注意点：不要用可选链掩盖本应存在的数据错误。
4. 可追问时补充：`??` 与 `||` 混用需要括号。

**常见追问**

- `a || b ?? c` 合法吗？→ 不合法，需要括号明确优先级。

---

## 三、对象与数组

### Q14. 数组哪些方法会修改原数组？

- **难度**: junior
- **标签**: Array

**答案**

会修改：`push`、`pop`、`shift`、`unshift`、`splice`、`sort`、`reverse`、`fill`、`copyWithin`。

不修改：`map`、`filter`、`slice`、`concat`、`flat`、`reduce` 等。

ES2023 新增不可变版本：

```js
const sorted = list.toSorted((a, b) => a - b)
const reversed = list.toReversed()
const removed = list.toSpliced(1, 1)
const replaced = list.with(0, 'new')
```

**口述模板（30–60 秒）**

1. 一句话结论：sort/splice/reverse 等会原地修改。
2. 关键机制：状态管理通常需要返回新数组。
3. 注意点：React 中原地 sort 可能导致更新问题。
4. 可追问时补充：toSorted 等不可变方法。

**常见追问**

- `sort` 默认怎么排序？→ 转字符串按 UTF-16 码元排序，所以数字要传比较函数。

---

### Q15. `map`、`forEach`、`filter`、`reduce` 怎么选？

- **难度**: junior
- **标签**: Array

**答案**

- `map`：一对一转换，返回新数组
- `forEach`：执行副作用，无返回值
- `filter`：筛选
- `reduce`：聚合为一个值或复杂结构
- `find` / `some` / `every`：查找或判断，可提前结束

`forEach` 不能用 `break` 中断；需要提前结束可用 `for...of` 或 `some`。

**口述模板（30–60 秒）**

1. 一句话结论：按返回结果选择数组方法。
2. 关键机制：map/filter 返回新数组，forEach 不返回。
3. 注意点：不要用 map 做纯副作用。
4. 可追问时补充：async 回调在 forEach 中不会被等待。

**常见追问**

- 如何顺序执行异步数组任务？→ `for...of` + `await`。

---

### Q16. `for...in` 和 `for...of` 有什么区别？

- **难度**: junior
- **标签**: 遍历

**答案**

- `for...in`：遍历对象的可枚举字符串键，包括原型链上的可枚举属性
- `for...of`：遍历可迭代对象的值，如数组、字符串、Map、Set

```js
for (const key in obj) {}
for (const value of array) {}
for (const [key, value] of Object.entries(obj)) {}
```

数组遍历不要用 `for...in`。

**口述模板（30–60 秒）**

1. 一句话结论：in 遍历键，of 遍历值。
2. 关键机制：of 依赖迭代器协议。
3. 注意点：for...in 可能遍历原型链属性。
4. 可追问时补充：对象默认不可迭代。

**常见追问**

- 怎么只遍历自身属性？→ `Object.keys` / `Object.entries`。

---

### Q17. 浅拷贝有哪些方式？

- **难度**: junior
- **标签**: 拷贝

**答案**

```js
const copy1 = { ...obj }
const copy2 = Object.assign({}, obj)
const arr1 = [...arr]
const arr2 = arr.slice()
```

浅拷贝只复制第一层，嵌套对象仍共享引用。

**口述模板（30–60 秒）**

1. 一句话结论：展开和 Object.assign 都是浅拷贝。
2. 关键机制：只复制第一层属性值。
3. 注意点：嵌套对象修改会互相影响。
4. 可追问时补充：structuredClone 深拷贝。

**常见追问**

- Object.assign 会复制 getter 吗？→ 会读取 getter 的值并写入目标，不复制 getter 本身。

---

### Q18. `JSON.parse(JSON.stringify(obj))` 深拷贝有什么问题？

- **难度**: junior
- **标签**: JSON, 拷贝

**答案**

问题：

- 丢失 `undefined`、函数、Symbol
- Date 变成字符串
- Map、Set、RegExp 等无法正确保留
- `NaN`、`Infinity` 变成 null
- 循环引用直接报错
- 原型链丢失

现代可用 `structuredClone(obj)`，支持循环引用、Date、Map、Set 等，但不支持函数和 DOM 节点。

**口述模板（30–60 秒）**

1. 一句话结论：JSON 方法只适合纯数据。
2. 关键机制：只能序列化 JSON 支持的数据类型。
3. 注意点：循环引用会报错。
4. 可追问时补充：structuredClone 和手写深拷贝。

**常见追问**

- structuredClone 能拷贝 class 实例吗？→ 会变成普通对象，原型方法不保留。

---

### Q19. Map 和 Object、Set 和 Array 有什么区别？

- **难度**: junior
- **标签**: Map, Set

**答案**

**Map vs Object**

- Map 的键可以是任意类型
- Map 保持插入顺序，`size` 直接获取
- Map 适合频繁增删的字典
- Object 适合固定结构记录和 JSON 数据

**Set vs Array**

- Set 值唯一，`has` 查找通常为 O(1)
- Array 有序、可重复，适合列表

```js
const unique = [...new Set(list)]
```

**口述模板（30–60 秒）**

1. 一句话结论：动态键值用 Map，唯一集合用 Set。
2. 关键机制：Map 键不会被转字符串。
3. 注意点：JSON 不能直接序列化 Map/Set。
4. 可追问时补充：WeakMap 和 WeakSet。

**常见追问**

- Object 的键能是对象吗？→ 会被转成字符串，如 `"[object Object]"`。

---

## 四、常用内置能力

### Q20. 模板字符串有什么用？

- **难度**: junior
- **标签**: 字符串

**答案**

模板字符串支持插值、多行和标签模板：

```js
const message = `你好，${user.name}`

function html(strings, ...values) {
  return strings.reduce((out, str, i) => out + str + (values[i] ?? ''), '')
}
```

注意：模板字符串只是拼接字符串，插入 HTML 时仍需转义，否则可能 XSS。

**口述模板（30–60 秒）**

1. 一句话结论：模板字符串用于插值和多行文本。
2. 关键机制：标签模板可接管拼接逻辑。
3. 注意点：不会自动防 XSS。
4. 可追问时补充：GraphQL、styled-components 使用标签模板。

**常见追问**

- `String.raw` 是什么？→ 获取不处理转义序列的原始字符串。

---

### Q21. `try...catch...finally` 的执行规则？

- **难度**: junior
- **标签**: 错误处理

**答案**

- `try` 中出错进入 `catch`
- `finally` 无论是否出错都会执行
- `finally` 中的 `return` 会覆盖前面的返回值或异常，应避免

```js
try {
  JSON.parse(text)
} catch (error) {
  console.error(error)
} finally {
  loading = false
}
```

异步错误需要 `await` 才能被外层 try 捕获。

**口述模板（30–60 秒）**

1. 一句话结论：catch 处理异常，finally 做清理。
2. 关键机制：同步异常和 await 的 rejection 可被 try 捕获。
3. 注意点：finally 中不要 return。
4. 可追问时补充：全局 unhandledrejection。

**常见追问**

- setTimeout 回调里的异常能被外层 try 捕获吗？→ 不能，它在后续任务中执行。

---

### Q22. 严格模式（Strict Mode）有什么作用？

- **难度**: junior
- **标签**: 严格模式

**答案**

严格模式让一些静默错误变成显式错误：

- 未声明变量赋值报错
- 普通函数默认 `this` 为 undefined
- 禁止重复参数名
- 禁止删除不可删除属性
- 禁用 `with`

ES Module 和 class 内部默认严格模式。

**口述模板（30–60 秒）**

1. 一句话结论：严格模式让代码更安全、更易优化。
2. 关键机制：限制历史上危险的语法和行为。
3. 注意点：模块默认严格。
4. 可追问时补充：this 默认绑定差异。

**常见追问**

- 普通脚本怎么开启？→ 文件或函数顶部写 `'use strict'`。

---

### Q23. `setTimeout(fn, 0)` 会立即执行吗？

- **难度**: junior
- **标签**: 定时器

**答案**

不会。它只是安排一个最早在约 0ms 后可执行的任务，必须等当前同步代码和微任务执行完。浏览器还可能对嵌套定时器设置最小延迟，后台标签页也会节流。

```js
setTimeout(() => console.log('timeout'), 0)
Promise.resolve().then(() => console.log('promise'))
console.log('sync')
// sync → promise → timeout
```

**口述模板（30–60 秒）**

1. 一句话结论：0ms 不等于立即执行。
2. 关键机制：定时器回调进入任务队列。
3. 注意点：微任务先于下一个任务。
4. 可追问时补充：后台页定时器节流。

**常见追问**

- setInterval 时间精确吗？→ 不精确，执行耗时和任务排队都会造成漂移。

---

### Q24. Symbol 有什么用？

- **难度**: junior
- **标签**: Symbol

**答案**

Symbol 是唯一值，常用于：

- 避免属性名冲突
- 定义内置协议：`Symbol.iterator`、`Symbol.asyncIterator`、`Symbol.toPrimitive`
- 元数据或半私有键

```js
const id = Symbol('id')
const user = { [id]: 1 }
```

Symbol 属性不会被 `for...in`、`Object.keys`、`JSON.stringify` 获取。

**口述模板（30–60 秒）**

1. 一句话结论：Symbol 提供唯一属性键。
2. 关键机制：每次 Symbol() 都不同。
3. 注意点：它不是安全私有，可用 Reflect.ownKeys 获取。
4. 可追问时补充：Symbol.for 全局注册。

**常见追问**

- 真正私有属性用什么？→ class 私有字段 `#field` 或闭包。

---

### Q25. class 基础语法有哪些？

- **难度**: junior
- **标签**: class

**答案**

```js
class User {
  static count = 0
  #password

  constructor(name, password) {
    this.name = name
    this.#password = password
    User.count++
  }

  checkPassword(input) {
    return input === this.#password
  }
}
```

- `constructor`：构造函数
- 实例字段、静态字段
- `#` 私有字段
- `extends` 继承，子类构造函数中先调用 `super()`

**口述模板（30–60 秒）**

1. 一句话结论：class 是基于原型的面向对象语法。
2. 关键机制：方法放在 prototype 上。
3. 注意点：class 默认严格模式，不能直接调用。
4. 可追问时补充：私有字段和继承。

**常见追问**

- class 会提升吗？→ 会登记但有 TDZ，声明前不能使用。

---

### Q26. `import` / `export` 基础用法？

- **难度**: junior
- **标签**: ES Module

**答案**

```js
export const API_URL = '/api'
export function request() {}
export default class Client {}

import Client, { API_URL, request } from './client.js'
import * as utils from './utils.js'
const module = await import('./heavy.js')
```

命名导出适合多个工具；默认导出适合模块主要内容。团队常偏好命名导出，便于重构和自动导入。

**口述模板（30–60 秒）**

1. 一句话结论：ESM 用 import/export 组织模块。
2. 关键机制：静态 import 在加载阶段解析依赖。
3. 注意点：动态 import 返回 Promise。
4. 可追问时补充：ESM 与 CommonJS 区别。

**常见追问**

- 浏览器原生 import 需要什么？→ `<script type="module">`。

---

## 五、浏览器基础

### Q27. DOM 事件的捕获、目标、冒泡阶段是什么？

- **难度**: junior
- **标签**: DOM 事件

**答案**

事件传播分三阶段：

1. 捕获阶段：从 window 向目标元素传播
2. 目标阶段：到达目标
3. 冒泡阶段：从目标向 window 传播

```js
el.addEventListener('click', handler)       // 默认冒泡阶段
el.addEventListener('click', handler, true) // 捕获阶段
```

`event.stopPropagation()` 阻止继续传播；`event.preventDefault()` 阻止默认行为。

**口述模板（30–60 秒）**

1. 一句话结论：事件先捕获，再到目标，再冒泡。
2. 关键机制：监听器可选择阶段。
3. 注意点：阻止传播和阻止默认行为不同。
4. 可追问时补充：事件委托。

**常见追问**

- 所有事件都冒泡吗？→ 不是，如 `focus`、`blur` 不冒泡，可用 `focusin`、`focusout`。

---

### Q28. 什么是事件委托？

- **难度**: junior
- **标签**: DOM 事件

**答案**

利用冒泡，在父元素统一监听子元素事件：

```js
list.addEventListener('click', (event) => {
  const button = event.target.closest('button[data-id]')
  if (!button || !list.contains(button)) return
  remove(button.dataset.id)
})
```

优点：减少监听器、支持动态元素。注意 `event.target` 可能是按钮内部图标，所以用 `closest`。

**口述模板（30–60 秒）**

1. 一句话结论：父元素统一处理子元素事件。
2. 关键机制：依赖事件冒泡。
3. 注意点：目标元素可能是内部节点。
4. 可追问时补充：React 事件系统也使用委托。

**常见追问**

- 不冒泡的事件怎么委托？→ 使用捕获阶段或可冒泡替代事件。

---

### Q29. Cookie、localStorage、sessionStorage 有什么区别？

- **难度**: junior
- **标签**: 存储

**答案**

| | Cookie | localStorage | sessionStorage |
|--|--------|--------------|----------------|
| 容量 | 约 4KB | 约 5MB 级 | 约 5MB 级 |
| 生命周期 | 可设置过期 | 持久保存 | 标签页会话 |
| 自动随请求发送 | 是（满足域名路径） | 否 | 否 |
| JS 可访问 | 非 HttpOnly 可访问 | 可访问 | 可访问 |

登录凭证优先 HttpOnly + Secure + SameSite Cookie，避免存在 localStorage 中被 XSS 读取。

**口述模板（30–60 秒）**

1. 一句话结论：Cookie 会随请求发送，Storage 只在本地。
2. 关键机制：生命周期和作用域不同。
3. 注意点：敏感 token 不要放 localStorage。
4. 可追问时补充：IndexedDB 存大量结构化数据。

**常见追问**

- Storage 是同步还是异步？→ 同步 API，大量读写可能阻塞主线程。

---

### Q30. Date 有哪些常见坑？

- **难度**: junior
- **标签**: Date

**答案**

- 月份从 0 开始：`new Date(2026, 0, 1)` 是 1 月
- 字符串解析格式不统一，可能受时区影响
- Date 对象可变
- 时区和夏令时处理困难

建议：

- 接口传 ISO 8601 / 时间戳
- 展示用 `Intl.DateTimeFormat`
- 复杂日期使用 date-fns / Day.js
- 关注 Temporal：已到 Stage 4，计划进入 ES2027，可先用 polyfill

**口述模板（30–60 秒）**

1. 一句话结论：Date API 易错，要统一格式和时区。
2. 关键机制：Date 本质是时间戳 + 本地时区解释。
3. 注意点：月份从 0 开始、对象可变。
4. 可追问时补充：Temporal 的不可变和时区模型。

**常见追问**

- 后端返回 `2026-10-09` 会有时区问题吗？→ 纯日期字符串在不同 API 中解析规则可能不同，业务日期最好按字符串处理或明确时区。
