# TypeScript 初级 Q&A（概念与常用写法）

> 目标：把基础类型、类型声明、收窄、泛型入门、常用工具类型、tsconfig 和声明文件讲清楚。初级题在高级面试中通常作为开场，回答要准确、简洁，并能自然引向深入点。

---

## 一、TypeScript 是什么

### Q1. TypeScript 是什么？解决了什么问题？

- **难度**: junior
- **标签**: 基础

**答案**

TypeScript 是 JavaScript 的**超集**，在 JS 之上加了**静态类型系统**。它的核心特点：

- **编译期检查，运行时擦除**：类型只在编译阶段存在，产出的是普通 JS。TS 不改变运行时行为（极少数例外：`enum`、`namespace`、参数属性、旧装饰器会生成代码）
- **结构化类型（Structural Typing）**：只要结构兼容就可赋值，不看名字
- **渐进式**：可以从 `allowJs` + `checkJs` 开始，逐步收紧

解决的问题：

1. 提前发现拼写错误、空值访问、参数类型不匹配
2. 类型即文档，IDE 补全、跳转、重构更可靠
3. 大型项目和多人协作下，重构的信心来自编译器

**口述模板（30–60 秒）**

1. 一句话结论：TS 是带静态类型的 JS 超集，编译期检查、运行时擦除。
2. 关键机制：结构化类型系统 + 类型推断 + 编译到 JS。
3. 注意点：类型不提供运行时保证，外部数据仍需校验。
4. 可追问时补充：现在 Node 可以直接擦除类型运行 TS，TS 7 用 Go 重写后编译快 8–12 倍。

**常见追问**

- TS 能保证运行时不出类型错误吗？→ 不能。`any`、`as` 断言、外部数据（接口返回、`JSON.parse`）都可能让类型「说谎」，边界处需要运行时校验。
- TS 和 JSDoc + `checkJs` 有什么区别？→ 后者也能用 TS 的检查器，但表达能力和可读性弱于 `.ts` 文件。

---

### Q2. TypeScript 有哪些基础类型？

- **难度**: junior
- **标签**: 基础类型

**答案**

| 类别 | 类型 |
|------|------|
| 原始类型 | `string`、`number`、`boolean`、`bigint`、`symbol`、`null`、`undefined` |
| 特殊类型 | `any`、`unknown`、`never`、`void` |
| 对象类型 | `object`、数组 `T[]` / `Array<T>`、元组 `[T, U]`、函数、类实例 |
| 字面量类型 | `'GET'`、`42`、`true`、模板字面量 `` `on${string}` `` |
| 组合类型 | 联合 `A \| B`、交叉 `A & B` |

```ts
let id: number = 1
let name = 'Tom'           // 推断为 string
const method = 'GET'       // 推断为字面量类型 'GET'
let tuple: [string, number] = ['a', 1]
let big: bigint = 10n
```

注意：`let` 声明会**拓宽（widening）**为 `string`，`const` 声明保留字面量类型。

**口述模板（30–60 秒）**

1. 一句话结论：7 种原始类型 + any/unknown/never/void + 对象、数组、元组、函数、字面量、联合、交叉。
2. 关键机制：大部分场景靠推断，不必处处标注。
3. 注意点：`let` 拓宽，`const` 保留字面量。
4. 可追问时补充：`as const` 可以让对象和数组也保留字面量并变成只读。

**常见追问**

- `Number` 和 `number` 有什么区别？→ 大写是包装对象类型，几乎不应该使用。

---

### Q3. `any`、`unknown`、`never`、`void` 的区别？

- **难度**: junior
- **标签**: 特殊类型, 高频

**答案**

| 类型 | 能接收什么 | 能赋给什么 | 能直接使用吗 | 典型场景 |
|------|-----------|-----------|-------------|---------|
| `any` | 任何值 | 任何类型 | 能，完全关闭检查 | 迁移期逃生舱，尽量避免 |
| `unknown` | 任何值 | 只能赋给 `unknown` / `any` | **不能**，必须先收窄 | 外部输入、`catch` 变量、`JSON.parse` 结果 |
| `never` | 没有任何值 | 任何类型 | — | 永不返回的函数、穷尽检查、过滤联合成员 |
| `void` | `undefined` | — | — | 函数没有有意义的返回值 |

```ts
function parse(input: unknown) {
  // input.trim()           // 报错：unknown 必须先收窄
  if (typeof input === 'string') return input.trim()
  throw new Error('invalid')
}

function fail(msg: string): never {
  throw new Error(msg)
}
```

在类型层级中：`unknown` 是**顶层类型（top type）**，`never` 是**底层类型（bottom type）**。`any` 同时表现为顶层和底层，所以它会「污染」类型系统。

**口述模板（30–60 秒）**

1. 一句话结论：any 关闭检查，unknown 是安全版 any，never 表示不可能的值，void 表示没有返回值。
2. 关键机制：unknown 必须收窄后才能用；never 是所有类型的子类型。
3. 注意点：any 会传染，一个 any 能让整条调用链失去检查。
4. 可追问时补充：never 在联合类型里会消失，常用于穷尽检查和类型过滤。

**常见追问**

- `() => void` 类型的函数可以返回值吗？→ 可以。函数类型的 `void` 返回表示「调用方不关心返回值」，所以 `arr.forEach(x => list.push(x))` 合法；但函数声明上直接写 `: void` 时返回值会报错。
- `catch (e)` 里 `e` 是什么类型？→ 开启 `useUnknownInCatchVariables`（`strict` 包含）后是 `unknown`。

---

### Q4. `type` 和 `interface` 有什么区别？什么时候用哪个？

- **难度**: junior
- **标签**: 类型声明, 高频

**答案**

| 维度 | `interface` | `type` |
|------|-------------|--------|
| 能描述什么 | 对象结构、函数、类契约 | 任意类型：联合、交叉、元组、原始类型别名、映射、条件类型 |
| 扩展方式 | `extends` | `&` 交叉 |
| 声明合并 | **支持**，同名自动合并 | 不支持，重名报错 |
| 冲突处理 | `extends` 时属性冲突**直接报错** | `&` 冲突时属性变成 `never`，不报错 |
| 编辑器显示 | 显示名字 | 可能展开成结构 |
| 检查性能 | `extends` 关系可缓存，大型项目更快 | 复杂交叉需反复计算 |

```ts
interface User { id: number }
interface User { name: string }    // 合并为 { id; name }

type Status = 'idle' | 'loading' | 'done'        // 只能用 type
type Point = [number, number]
type Handler = (e: Event) => void
```

实践建议：

- **对象结构、对外暴露、需要被扩展**的类型用 `interface`
- **联合、元组、工具类型、类型运算**用 `type`
- 团队统一即可，不必纠结

**口述模板（30–60 秒）**

1. 一句话结论：interface 只描述对象且可声明合并，type 能表达任意类型。
2. 关键机制：interface 用 extends 继承，type 用交叉组合。
3. 注意点：extends 冲突会报错，交叉冲突会静默变成 never；大型项目中 interface 继承的检查性能更好。
4. 可追问时补充：声明合并可以用来扩充第三方库类型，比如给 `Window` 加属性。

**常见追问**

- interface 能继承 type 吗？→ 能，只要 type 是对象类型（不能是联合）。
- 为什么官方性能建议推荐 interface？→ 交叉类型每次都要重新展开比较，interface 的继承关系可以被缓存。

---

### Q5. 联合类型和交叉类型分别是什么？

- **难度**: junior
- **标签**: 组合类型

**答案**

- **联合类型（Union）** `A | B`：值是 A **或** B。使用时只能访问**共有成员**，要访问特有成员必须先收窄
- **交叉类型（Intersection）** `A & B`：值同时满足 A **和** B，拥有两者全部成员

```ts
type Id = string | number
function show(id: Id) {
  // id.toUpperCase()   // 报错：number 上没有
  if (typeof id === 'string') id.toUpperCase()
}

type WithTime = { createdAt: Date }
type Post = { title: string } & WithTime
```

反直觉的点：从「值的集合」看，联合是并集、交叉是交集；但从「成员」看，联合的可用成员**更少**，交叉的可用成员**更多**。

```ts
type X = string & number   // never：不存在同时是两者的值
type Y = { a: string } & { a: number }   // { a: never }
```

**口述模板（30–60 秒）**

1. 一句话结论：联合是「或」，交叉是「且」。
2. 关键机制：联合要收窄才能用特有成员；交叉合并全部成员。
3. 注意点：原始类型交叉通常得到 never；同名属性类型冲突会变成 never。
4. 可追问时补充：用集合论理解——类型是值的集合，联合是并集、交叉是交集。

**常见追问**

- 函数类型的联合能调用吗？→ 能，但参数会取交叉。`((a: string) => void) | ((a: number) => void)` 调用时参数要求 `string & number`，即 `never`。

---

### Q6. 字面量类型和 `as const` 是什么？

- **难度**: junior
- **标签**: 字面量, as const

**答案**

字面量类型是把具体值当作类型：`'GET'`、`200`、`true`。配合联合可以表示有限集合：

```ts
type Method = 'GET' | 'POST'
function request(url: string, method: Method) {}

const opts = { method: 'GET' }
request('/a', opts.method)     // 报错：opts.method 被推断为 string

const opts2 = { method: 'GET' } as const
request('/a', opts2.method)    // OK：类型是 'GET'
```

`as const`（常量断言）的效果：

1. 字面量不拓宽
2. 对象属性全部变成 `readonly`
3. 数组变成**只读元组**

```ts
const ROLES = ['admin', 'user'] as const
type Role = (typeof ROLES)[number]    // 'admin' | 'user'
```

这是替代 `enum` 的常用写法：一份值同时得到运行时数组和类型。

**口述模板（30–60 秒）**

1. 一句话结论：字面量类型把具体值作为类型，`as const` 阻止拓宽并变为只读。
2. 关键机制：`let` 和对象属性默认拓宽，`as const` 冻结为最窄类型。
3. 注意点：`as const` 只影响类型，不会在运行时冻结对象。
4. 可追问时补充：`typeof ROLES[number]` 是从常量数组派生联合类型的标准写法。

**常见追问**

- `as const` 和 `Object.freeze` 的区别？→ 前者纯类型层面，后者运行时浅冻结；`Object.freeze` 的返回类型也是 `Readonly<T>`。

---

### Q7. 元组和数组有什么区别？

- **难度**: junior
- **标签**: 元组

**答案**

- 数组 `T[]`：长度不定，元素类型相同
- 元组 `[A, B]`：**长度和每个位置的类型都固定**

```ts
type Entry = [key: string, value: number]          // 具名元组，提升可读性
type Opt = [string, number?]                        // 可选元素
type Rest = [string, ...number[]]                   // 剩余元素
type RO = readonly [number, number]                 // 只读元组

function useToggle(): [boolean, () => void] {       // React Hook 常见返回
  return [true, () => {}]
}
```

注意：普通元组仍然有 `push` 方法，`push` 后长度改变不会报错；需要严格时使用 `readonly`。

**口述模板（30–60 秒）**

1. 一句话结论：元组是长度和位置类型固定的数组。
2. 关键机制：支持可选、剩余、具名元素和只读。
3. 注意点：可变元组的 push 不会被拦截，建议 readonly。
4. 可追问时补充：类型编程中元组的 `length` 是字面量，常用来做类型层面的计数。

**常见追问**

- `readonly string[]` 能赋给 `string[]` 吗？→ 不能，反过来可以。

---

### Q8. 可选属性 `?:` 和 `| undefined` 有什么区别？

- **难度**: junior
- **标签**: 可选属性

**答案**

```ts
type A = { name?: string }               // 属性可以不存在
type B = { name: string | undefined }    // 属性必须存在，值可以是 undefined

const a: A = {}                          // OK
const b: B = {}                          // 报错：缺少 name
const b2: B = { name: undefined }        // OK
```

默认情况下 `name?: string` 也允许显式赋值 `undefined`。开启 `exactOptionalPropertyTypes` 后，`?:` 只表示「可以不存在」，不允许显式写 `undefined`，用于区分「没传」和「传了 undefined」（例如 `PATCH` 请求中 undefined 可能表示要清空字段）。

函数参数同理：`(x?: number)` 可以不传，`(x: number | undefined)` 必须传。

**口述模板（30–60 秒）**

1. 一句话结论：`?:` 表示键可以不存在，`| undefined` 表示键必须存在但值可以是 undefined。
2. 关键机制：默认情况下两者对赋值 undefined 都宽松。
3. 注意点：`exactOptionalPropertyTypes` 能区分「缺失」和「显式 undefined」。
4. 可追问时补充：`'name' in obj` 的结果在两种类型下含义不同。

**常见追问**

- 读取可选属性得到什么类型？→ `string | undefined`。

---

### Q9. 类型断言 `as`、非空断言 `!`、双重断言是什么？

- **难度**: junior
- **标签**: 类型断言

**答案**

类型断言是告诉编译器「我比你更清楚这个值的类型」，**不做任何运行时转换或检查**。

```ts
const el = document.getElementById('app') as HTMLDivElement
const input = document.querySelector('input')!     // 非空断言：去掉 null
const n = ('x' as unknown) as number               // 双重断言：绕过兼容性检查
```

规则：`as` 只允许在「有重叠」的类型间断言（一方能赋给另一方），否则需要先转成 `unknown`。

风险：断言错误不会报错，只会把问题推迟到运行时。优先级建议：

1. 类型收窄（typeof / in / 类型谓词）
2. `satisfies`
3. 必要时才用 `as`，并写清原因

**口述模板（30–60 秒）**

1. 一句话结论：断言是开发者向编译器担保类型，编译器不再检查。
2. 关键机制：只能在兼容方向断言；`!` 去掉 null 和 undefined。
3. 注意点：断言错了就是运行时 bug，优先用收窄。
4. 可追问时补充：`as const` 名字里有 as，但它是常量断言，是安全的。

**常见追问**

- `<T>value` 写法呢？→ 旧语法，在 `.tsx` 中会和 JSX 冲突，统一用 `as`。

---

## 二、类型收窄与函数

### Q10. TypeScript 有哪些类型收窄（Narrowing）方式？

- **难度**: junior
- **标签**: 收窄, 高频

**答案**

| 方式 | 示例 | 适用 |
|------|------|------|
| `typeof` | `typeof x === 'string'` | 原始类型、函数 |
| `instanceof` | `e instanceof Error` | 类实例 |
| `in` | `'swim' in animal` | 区分对象结构 |
| 判等 | `x === null`、`x == null` | 字面量、null/undefined |
| 真值 | `if (x)` | 排除 null/undefined/空值 |
| 可辨识联合 | `switch (action.type)` | 带公共字面量字段的联合 |
| 类型谓词 | `function isFish(x): x is Fish` | 自定义判断 |
| 断言函数 | `function assert(x): asserts x is T` | 失败即抛错 |
| `Array.isArray` | `Array.isArray(v)` | 数组 |

```ts
type Shape =
  | { kind: 'circle'; r: number }
  | { kind: 'rect'; w: number; h: number }

function area(s: Shape) {
  switch (s.kind) {
    case 'circle': return Math.PI * s.r ** 2
    case 'rect': return s.w * s.h
  }
}
```

**口述模板（30–60 秒）**

1. 一句话结论：typeof、instanceof、in、判等、真值、可辨识联合、类型谓词、断言函数。
2. 关键机制：编译器做控制流分析（Control Flow Analysis），根据分支缩小类型。
3. 注意点：真值收窄会把 `0` 和 `''` 一起排除，可能误伤。
4. 可追问时补充：业务中最推荐可辨识联合，配合 never 做穷尽检查。

**常见追问**

- `typeof null` 是什么？→ `'object'`，所以 `typeof x === 'object'` 收窄后仍可能是 `null`。

---

### Q11. 自定义类型守卫 `is` 和断言函数 `asserts` 怎么写？

- **难度**: junior
- **标签**: 类型谓词

**答案**

```ts
interface Fish { swim(): void }
interface Bird { fly(): void }

// 类型谓词：返回 true 时，参数被收窄为 Fish
function isFish(pet: Fish | Bird): pet is Fish {
  return 'swim' in pet
}

// 断言函数：函数正常返回，即说明条件成立
function assertIsString(v: unknown): asserts v is string {
  if (typeof v !== 'string') throw new TypeError('not string')
}

function handle(v: unknown) {
  assertIsString(v)
  v.toUpperCase()   // v 已是 string
}
```

注意：**类型谓词的正确性由开发者保证**，函数体写错了编译器不会发现，本质上和 `as` 一样有风险。

TS 5.5 起可以自动推断类型谓词：

```ts
const list = [1, undefined, 2].filter(x => x !== undefined)   // number[]
```

**口述模板（30–60 秒）**

1. 一句话结论：`x is T` 让布尔函数带上收窄能力，`asserts x is T` 让函数通过即收窄。
2. 关键机制：编译器信任返回值，在调用处缩小类型。
3. 注意点：函数体逻辑写错不会被检查。
4. 可追问时补充：5.5 起简单的过滤函数能自动推断谓词。

**常见追问**

- 断言函数有什么限制？→ 必须用显式类型标注的函数声明来调用（箭头函数赋值给 `const` 时需要标注类型）。

---

### Q12. 函数类型怎么写？重载怎么用？

- **难度**: junior
- **标签**: 函数, 重载

**答案**

```ts
type Add = (a: number, b: number) => number

function greet(name: string, greeting = 'Hi', ...rest: string[]) {}

// this 参数：只用于类型检查，编译后会被删除
function onClick(this: HTMLButtonElement, e: MouseEvent) {
  this.disabled = true
}

// 重载：多个签名 + 一个实现
function format(v: string): string
function format(v: number, digits: number): string
function format(v: string | number, digits?: number): string {
  return typeof v === 'number' ? v.toFixed(digits) : v.trim()
}
```

重载要点：

- 实现签名对外**不可见**，调用方只能看到上面的重载签名
- 按顺序匹配，**具体的写在前面**
- 能用联合或泛型表达时优先不用重载

**口述模板（30–60 秒）**

1. 一句话结论：函数类型标注参数和返回值；重载用多个签名描述不同调用方式。
2. 关键机制：重载签名对外可见，实现签名必须兼容所有重载。
3. 注意点：顺序从具体到宽泛；简单场景用联合或泛型更清晰。
4. 可追问时补充：返回值依赖参数类型时，重载或泛型 + 条件类型才能表达。

**常见追问**

- 重载和联合参数怎么选？→ 返回值类型不随参数变化时用联合；返回值随参数变化时用重载或泛型。

---

### Q13. `enum` 是什么？现在还推荐用吗？

- **难度**: junior
- **标签**: 枚举

**答案**

```ts
enum Dir { Up, Down }              // 数字枚举：生成双向映射
// 编译为：Dir[Dir["Up"] = 0] = "Up" ...

enum Color { Red = 'RED' }         // 字符串枚举：单向映射

const enum Flag { A = 1 }          // const enum：编译时内联，不生成对象
```

问题：

1. `enum` 会生成运行时代码，不是纯类型擦除，**`--erasableSyntaxOnly` 和 Node 原生类型擦除都不支持**
2. `const enum` 与 `isolatedModules`（Babel/esbuild/SWC 单文件转译）配合有坑，跨文件无法内联
3. 数字枚举可接受任意 `number` 变量，约束较弱
4. 名义化：字符串枚举不能直接用字符串字面量赋值

推荐替代：

```ts
const Status = { Idle: 'idle', Done: 'done' } as const
type Status = (typeof Status)[keyof typeof Status]   // 'idle' | 'done'
```

**口述模板（30–60 秒）**

1. 一句话结论：enum 会生成运行时对象，现在更推荐 `as const` 对象加联合类型。
2. 关键机制：数字枚举双向映射，字符串枚举单向，const enum 内联。
3. 注意点：和单文件转译工具、Node 类型擦除不兼容。
4. 可追问时补充：老项目已有的枚举不必强改，新代码统一规范即可。

**常见追问**

- 为什么同名的 `const` 和 `type` 不冲突？→ 值空间和类型空间是两个命名空间。

---

## 三、泛型与工具类型

### Q14. 什么是泛型？为什么需要？

- **难度**: junior
- **标签**: 泛型, 高频

**答案**

泛型（Generics）是**类型参数**：在定义时不确定具体类型，在使用时由调用方传入或由编译器推断，从而在复用代码的同时保留类型关系。

```ts
function first<T>(arr: T[]): T | undefined {
  return arr[0]
}
const n = first([1, 2])          // T 推断为 number

// 约束
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
getProp({ a: 1 }, 'a')           // number
// getProp({ a: 1 }, 'b')        // 报错

// 默认值
interface ApiResp<T = unknown> { code: number; data: T }
```

如果用 `any` 代替，会丢失「输入和输出之间的关系」；泛型的价值就在于**描述类型之间的关系**。

**口述模板（30–60 秒）**

1. 一句话结论：泛型是类型参数，用来在复用的同时保留类型关系。
2. 关键机制：调用时推断，`extends` 约束，可以设默认值。
3. 注意点：只出现一次的类型参数通常没有意义，可以直接用具体类型或 unknown。
4. 可追问时补充：`K extends keyof T` + `T[K]` 是最经典的组合。

**常见追问**

- 箭头函数在 `.tsx` 里写泛型有什么问题？→ `<T>` 会被当成 JSX，写成 `<T,>` 或 `<T extends unknown>`。

---

### Q15. 常用的内置工具类型有哪些？

- **难度**: junior
- **标签**: 工具类型, 高频

**答案**

| 工具类型 | 作用 |
|---------|------|
| `Partial<T>` | 全部可选 |
| `Required<T>` | 全部必选 |
| `Readonly<T>` | 全部只读 |
| `Pick<T, K>` | 挑选属性 |
| `Omit<T, K>` | 去掉属性 |
| `Record<K, V>` | 键为 K、值为 V 的对象 |
| `Exclude<U, E>` | 从联合中去掉 |
| `Extract<U, E>` | 从联合中提取 |
| `NonNullable<T>` | 去掉 null 和 undefined |
| `ReturnType<F>` / `Parameters<F>` | 函数返回值 / 参数元组 |
| `InstanceType<C>` / `ConstructorParameters<C>` | 构造函数实例 / 参数 |
| `Awaited<T>` | 递归解开 Promise |
| `NoInfer<T>`（5.4+） | 阻止从该位置推断 |
| `Uppercase` / `Lowercase` / `Capitalize` / `Uncapitalize` | 字符串字面量变换 |

```ts
type UserForm = Partial<Omit<User, 'id'>>
type Fetcher = typeof fetchUser
type UserData = Awaited<ReturnType<Fetcher>>
```

**口述模板（30–60 秒）**

1. 一句话结论：对象类有 Partial/Required/Readonly/Pick/Omit/Record，联合类有 Exclude/Extract/NonNullable，函数类有 ReturnType/Parameters。
2. 关键机制：对象类基于映射类型，联合类基于分布式条件类型，函数类基于 infer。
3. 注意点：`Omit` 的 K 不受约束，写错键名不会报错。
4. 可追问时补充：都能手写（见手写题 H1、H2）。

**常见追问**

- `Pick` 和 `Omit` 在实现上有什么区别？→ `Pick` 是同态映射，会保留可选和只读修饰符；`Omit` 由 `Pick<T, Exclude<keyof T, K>>` 实现，同样保留。

---

### Q16. `keyof`、`typeof`、索引访问类型 `T[K]` 分别是什么？

- **难度**: junior
- **标签**: 类型操作符

**答案**

```ts
const config = { port: 3000, host: 'localhost' }

type Config = typeof config          // 从值得到类型：{ port: number; host: string }
type Keys = keyof Config             // 'port' | 'host'
type Port = Config['port']           // number
type Values = Config[keyof Config]   // number | string

const arr = ['a', 'b'] as const
type Item = (typeof arr)[number]     // 'a' | 'b'
```

- `typeof`（类型位置）：**值 → 类型**
- `keyof`：对象类型 → 键的联合
- `T[K]`：按键取属性类型；`T[number]` 取数组元素类型

`keyof` 的特殊情况：带 `string` 索引签名时，`keyof` 是 `string | number`（因为 JS 数字键会被转成字符串）。

**口述模板（30–60 秒）**

1. 一句话结论：typeof 从值取类型，keyof 取键联合，T[K] 按键取值类型。
2. 关键机制：三者组合可以从一份运行时常量派生出全部类型。
3. 注意点：类型位置的 typeof 和 JS 的 typeof 运算符是两回事。
4. 可追问时补充：「单一数据源」思想——配置写一次，类型自动派生。

**常见追问**

- `keyof any` 是什么？→ `string | number | symbol`，即内置的 `PropertyKey`。

---

### Q17. 索引签名和 `Record` 有什么区别？

- **难度**: junior
- **标签**: 索引签名

**答案**

```ts
interface Dict { [key: string]: number }
type Dict2 = Record<string, number>          // 基本等价

type Score = Record<'math' | 'english', number>   // 键是有限集合：每个键都必须存在
```

区别：

- 索引签名的键只能是 `string`、`number`、`symbol`、模板字面量类型
- `Record<K, V>` 是映射类型，K 可以是字面量联合，表示「这些键都必须有」
- 索引签名的 interface 能声明合并，还能和具名属性共存（具名属性必须兼容索引签名的值类型）

常见坑：`Record<string, T>` 读取任意键得到 `T`，而不是 `T | undefined`。开启 `noUncheckedIndexedAccess` 后才会加上 `undefined`。

**口述模板（30–60 秒）**

1. 一句话结论：键是开放集合用索引签名或 `Record<string, V>`，键是固定集合用 `Record<联合, V>`。
2. 关键机制：Record 是映射类型。
3. 注意点：动态键读取默认不带 undefined，有空值风险。
4. 可追问时补充：推荐开启 `noUncheckedIndexedAccess`。

**常见追问**

- 想要「部分键」怎么写？→ `Partial<Record<K, V>>`。

---

### Q18. 类的访问修饰符有哪些？`private` 和 `#private` 有什么区别？

- **难度**: junior
- **标签**: 类

**答案**

```ts
abstract class Animal {
  // 参数属性：声明并赋值（会生成代码，不属于可擦除语法）
  constructor(protected readonly name: string) {}
  abstract speak(): string
}

class Dog extends Animal implements Serializable {
  private secret = 1        // TS 编译期私有
  #realSecret = 2           // JS 运行时私有
  static count = 0
  speak() { return `${this.name}: woof` }
  toJSON() { return { name: this.name } }
}
```

| 维度 | `private` | `#private` |
|------|-----------|------------|
| 层面 | 只在编译期检查 | 运行时真正私有 |
| 绕过 | `(obj as any).secret` 能访问 | 无法从外部访问 |
| 序列化 / 反射 | 仍出现在 `Object.keys` 中 | 不出现 |
| 子类同名 | 冲突报错 | 互不影响 |

其他：`protected` 子类可访问；`readonly` 只能在声明或构造函数中赋值；`abstract` 类不能实例化；`implements` 只检查类的实例部分是否满足接口。

**口述模板（30–60 秒）**

1. 一句话结论：public/private/protected/readonly 是编译期修饰符，`#` 是运行时私有。
2. 关键机制：TS 修饰符编译后被擦除。
3. 注意点：需要真正封装用 `#`；参数属性不是可擦除语法。
4. 可追问时补充：私有成员会让类在结构化类型中变得「名义化」。

**常见追问**

- 两个结构相同但都带 `private` 成员的类能互相赋值吗？→ 不能，私有成员必须来自同一个声明。

---

## 四、工程配置

### Q19. tsconfig 中最重要的配置有哪些？

- **难度**: junior
- **标签**: tsconfig, 高频

**答案**

```jsonc
{
  "compilerOptions": {
    "target": "es2022",                // 输出语法版本
    "lib": ["es2023", "dom"],          // 可用的内置 API 类型
    "module": "nodenext",              // 输出的模块格式
    "moduleResolution": "nodenext",    // 如何解析 import 路径
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "verbatimModuleSyntax": true,      // import type 必须显式写
    "isolatedModules": true,           // 保证单文件转译工具可用
    "skipLibCheck": true,              // 跳过 .d.ts 检查，提速
    "declaration": true,               // 生成 .d.ts（库项目）
    "sourceMap": true,
    "outDir": "dist",
    "rootDir": "src",
    "types": ["node"]
  },
  "include": ["src"]
}
```

常见组合：

- **前端（Vite 等打包器）**：`module: "preserve"` 或 `"esnext"` + `moduleResolution: "bundler"` + `noEmit: true`（只检查，由打包器转译）
- **Node 服务 / 库**：`module` 与 `moduleResolution` 都设为 `"nodenext"`

TS 6.0 默认值变化：`strict: true`、`module: esnext`、`target` 为最新 ES、`types: []`、`rootDir: "."`。升级时需要显式补上 `types` 和 `rootDir`。

**口述模板（30–60 秒）**

1. 一句话结论：target/lib 决定语法和 API，module/moduleResolution 决定模块，strict 决定检查严格度。
2. 关键机制：前端走 bundler 解析，Node 走 nodenext。
3. 注意点：TS 6 起 types 默认为空数组，全局类型（如 node、jest）要显式声明。
4. 可追问时补充：库项目要开 declaration，大型项目用 composite + 项目引用。

**常见追问**

- `target` 和 `lib` 有什么区别？→ `target` 决定语法降级，`lib` 决定类型声明；`target` 不会帮你 polyfill API。

---

### Q20. `strict` 包含哪些子选项？

- **难度**: junior
- **标签**: strict

**答案**

| 子选项 | 作用 |
|--------|------|
| `noImplicitAny` | 无法推断时不允许隐式 any |
| `strictNullChecks` | null/undefined 不再能赋给任何类型 |
| `strictFunctionTypes` | 函数参数逆变检查（方法语法除外） |
| `strictBindCallApply` | 检查 bind/call/apply 参数 |
| `strictPropertyInitialization` | 类属性必须在构造函数中初始化 |
| `noImplicitThis` | 不允许隐式 any 的 this |
| `useUnknownInCatchVariables` | catch 变量为 unknown |
| `alwaysStrict` | 输出 `"use strict"`（TS 6 起不能设为 false） |
| `strictBuiltinIteratorReturn` | 内置迭代器的返回类型更严格（5.6+） |

`strict` 之外推荐额外开启：`noUncheckedIndexedAccess`、`exactOptionalPropertyTypes`、`noImplicitOverride`、`noFallthroughCasesInSwitch`。

**口述模板（30–60 秒）**

1. 一句话结论：strict 是一组严格检查的总开关，最核心的是 noImplicitAny 和 strictNullChecks。
2. 关键机制：新版本 TS 新增的严格选项会自动纳入 strict。
3. 注意点：升级 TS 版本可能因此出现新报错。
4. 可追问时补充：老项目可以先开 strict 再对个别子项设 false，逐步收紧。

**常见追问**

- 为什么 `strictNullChecks` 最重要？→ 空值访问是 JS 最常见的运行时错误，开启后编译器强制处理。

---

### Q21. `.d.ts` 声明文件是什么？`declare` 怎么用？

- **难度**: junior
- **标签**: 声明文件

**答案**

`.d.ts` 只包含类型信息，不包含实现，用来描述 JS 代码的类型。来源：

1. 库自带（`package.json` 的 `types` 或 `exports` 中的 `types` 条件）
2. DefinitelyTyped 社区包 `@types/xxx`
3. 项目内手写

```ts
// global.d.ts
declare const __APP_VERSION__: string          // 构建工具注入的全局常量

declare module '*.svg' {                       // 让非 JS 资源可以 import
  const url: string
  export default url
}

declare module 'legacy-lib' {                  // 为无类型的第三方库补类型
  export function init(opts: { key: string }): void
}

// 扩充全局对象：所在文件必须是模块（有 import/export）
export {}
declare global {
  interface Window { __STORE__: unknown }
}
```

`declare` 表示「这个东西在别处存在，我只声明它的类型」。

**口述模板（30–60 秒）**

1. 一句话结论：.d.ts 只描述类型，declare 声明在别处实现的东西。
2. 关键机制：类型来源依次是库自带、@types、手写声明。
3. 注意点：`declare global` 必须在模块文件中；全局声明文件里不能有顶层 import，否则会变成模块。
4. 可追问时补充：模块扩充（module augmentation）可以给已有库加类型。

**常见追问**

- `skipLibCheck` 有什么风险？→ 跳过所有 `.d.ts` 的检查，可能掩盖声明文件之间的冲突，但通常是值得的提速。

---

### Q22. `import type` 和 `verbatimModuleSyntax` 是做什么的？

- **难度**: junior
- **标签**: 模块

**答案**

```ts
import type { User } from './types'              // 只导入类型，编译后完全删除
import { fetchUser, type Query } from './api'    // 内联 type 修饰符
export type { User }
```

为什么需要：单文件转译工具（Babel、esbuild、SWC、Node 类型擦除）**不做类型分析**，看到 `import { User }` 时不知道 `User` 是类型还是值，无法判断能不能删除这条导入。

`verbatimModuleSyntax: true` 的规则很简单：

- 带 `type` 修饰的导入/导出：删除
- 不带的：**原样保留**

这样所有工具的行为一致。它替代了旧的 `importsNotUsedAsValues` 和 `preserveValueImports`。

**口述模板（30–60 秒）**

1. 一句话结论：import type 显式标记纯类型导入，确保编译后被删除。
2. 关键机制：单文件转译工具无法判断一个名字是不是类型。
3. 注意点：开启 verbatimModuleSyntax 后，漏写 type 会导致运行时导入不存在的导出。
4. 可追问时补充：ESLint 的 `consistent-type-imports` 可以自动修复。

**常见追问**

- 不写 `type` 会有什么实际问题？→ 可能产生多余的副作用导入，或在 ESM 中导入不存在的导出而直接报错。

---

### Q23. 为什么说 TS 类型在运行时不存在？外部数据怎么保证类型安全？

- **难度**: junior
- **标签**: 运行时校验

**答案**

TS 编译后类型全部擦除，下面这些代码**编译通过但运行时可能出错**：

```ts
const user = JSON.parse(text) as User        // 断言，不校验
const res: User = await (await fetch(url)).json()   // json() 返回 any
```

解决方案：在**信任边界**做运行时校验，并从 schema 推导类型，保证只有一份定义。

```ts
import { z } from 'zod'

const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email().optional(),
})
type User = z.infer<typeof UserSchema>

const user = UserSchema.parse(await res.json())   // 失败会抛出带路径的错误
```

信任边界包括：接口响应、表单输入、`localStorage`、URL 参数、环境变量、消息队列、第三方 SDK 回调。

**口述模板（30–60 秒）**

1. 一句话结论：类型编译后擦除，外部数据必须在边界做运行时校验。
2. 关键机制：用 Zod 等库定义 schema，再用 infer 推导类型，单一数据源。
3. 注意点：不要用 as 断言接口返回值。
4. 可追问时补充：前后端可以共享 schema，或从 OpenAPI 生成类型和校验器。

**常见追问**

- 每个接口都校验会不会太重？→ 对核心链路和不可信来源校验；内部可信服务可以只在开发环境校验或抽样校验。

---

### Q24. `null`、`undefined` 和 `strictNullChecks` 怎么处理？

- **难度**: junior
- **标签**: 空值

**答案**

开启 `strictNullChecks` 后，`null` 和 `undefined` 是独立类型，必须显式处理：

```ts
function getName(u?: { name?: string }) {
  return u?.name ?? 'anonymous'      // 可选链 + 空值合并
}

const el = document.querySelector('#app')   // Element | null
if (el) el.textContent = 'hi'               // 收窄
```

常用工具：

- `?.` 可选链：遇到 null/undefined 短路返回 undefined
- `??` 空值合并：只对 null/undefined 生效（`||` 会把 `0`、`''` 也替换掉）
- `!` 非空断言：确定不为空时使用，慎用
- `NonNullable<T>`：类型层面去掉空值

**口述模板（30–60 秒）**

1. 一句话结论：开启 strictNullChecks 后空值必须显式处理。
2. 关键机制：收窄、可选链、空值合并。
3. 注意点：`??` 和 `||` 不同，`0` 和空字符串场景要用 `??`。
4. 可追问时补充：滥用 `!` 等于关掉了这项检查。

**常见追问**

- 项目里应该统一用 null 还是 undefined？→ 团队约定即可；常见做法是「缺失」用 undefined，「显式为空」（如数据库字段）用 null。

---

### Q25. `object`、`Object`、`{}` 有什么区别？

- **难度**: junior
- **标签**: 对象类型

**答案**

| 类型 | 含义 | 能接收原始值吗 |
|------|------|--------------|
| `object` | 所有非原始类型 | 不能 |
| `{}` | 除 null 和 undefined 之外的**任何值** | **能**（`const a: {} = 1` 合法） |
| `Object` | 和 `{}` 基本相同，还要求兼容 Object 原型方法 | 能 |

```ts
let a: object = { x: 1 }
// a = 1                   // 报错
let b: {} = 'str'          // OK
```

实践建议：

- 表示「任意对象」：`Record<string, unknown>` 或 `object`
- 表示「任意值」：`unknown`
- 表示「空对象」：`Record<string, never>`
- 不要用 `{}` 和 `Object`（typescript-eslint 默认禁止）

**口述模板（30–60 秒）**

1. 一句话结论：object 是非原始值，`{}` 是非空值，Object 几乎等同于 `{}`。
2. 关键机制：`{}` 表示「没有约束成员的结构」，所以原始值也满足。
3. 注意点：用 `{}` 表示空对象是常见误区。
4. 可追问时补充：泛型约束 `T extends {}` 可以用来排除 null 和 undefined。

**常见追问**

- `unknown` 和 `{} | null | undefined` 有什么关系？→ 在 strictNullChecks 下两者等价。

---

### Q26. `satisfies` 是什么？和类型注解有什么区别？

- **难度**: junior
- **标签**: satisfies, 高频

**答案**

`satisfies`（TS 4.9+）：**检查**表达式符合某个类型，但**保留**表达式自身更精确的推断类型。

```ts
type Theme = Record<'primary' | 'danger', string | [number, number, number]>

const a: Theme = { primary: '#fff', danger: [255, 0, 0] }
a.primary.toUpperCase()      // 报错：类型被拓宽为 string | [number...]

const b = { primary: '#fff', danger: [255, 0, 0] } satisfies Theme
b.primary.toUpperCase()      // OK：保留了 string
// 拼错键名、少写键都会报错
```

| 写法 | 校验 | 结果类型 |
|------|------|---------|
| `const x: T = v` | 校验 | 变成 T（丢失精确信息） |
| `const x = v as T` | 不真正校验 | 变成 T |
| `const x = v satisfies T` | 校验 | 保留 v 的推断类型 |

常与 `as const` 一起使用：`{ ... } as const satisfies Config`，既只读又精确又受约束。

**口述模板（30–60 秒）**

1. 一句话结论：satisfies 只校验不改变类型，注解会拓宽，as 不校验。
2. 关键机制：同时得到类型约束和精确推断。
3. 注意点：适合配置对象、路由表、主题这类「既要约束又要精确」的常量。
4. 可追问时补充：`as const satisfies T` 是配置类常量的常见写法。

**常见追问**

- `satisfies` 会触发多余属性检查吗？→ 会，对象字面量多写属性会报错。

---

## 五、实战入门

### Q27. React 项目中常见的 TS 写法有哪些？

- **难度**: junior
- **标签**: React

**答案**

```tsx
import type { ReactNode, ComponentProps, ChangeEvent } from 'react'

interface ButtonProps extends ComponentProps<'button'> {   // 继承原生属性
  variant?: 'primary' | 'ghost'
  children: ReactNode
}

function Button({ variant = 'primary', ...rest }: ButtonProps) {
  return <button data-variant={variant} {...rest} />
}

function Form() {
  const [user, setUser] = useState<User | null>(null)    // 初始值无法推断完整类型时显式传泛型
  const inputRef = useRef<HTMLInputElement>(null)
  const onChange = (e: ChangeEvent<HTMLInputElement>) => setName(e.target.value)
}
```

要点：

- Props 用 `interface` 或 `type`，不再推荐 `React.FC`（旧版本会隐式带 `children`）
- `children` 用 `ReactNode`
- 继承原生属性用 `ComponentProps<'button'>`
- React 19 起 `ref` 可作为普通 prop 传入，`useRef` 必须传初始参数

**口述模板（30–60 秒）**

1. 一句话结论：Props 写接口，状态初值不完整时显式泛型，事件用 React 提供的事件类型。
2. 关键机制：`ComponentProps` 复用原生属性。
3. 注意点：不推荐 React.FC；React 19 的 ref 类型有变化。
4. 可追问时补充：通用列表、表格组件用泛型组件实现（见中级题）。

**常见追问**

- `JSX.Element` 和 `ReactNode` 的区别？→ 前者只是 JSX 表达式的结果；后者还包括字符串、数字、null、数组等，作为 children 类型更合适。

---

### Q28. 怎么运行 TS 代码？tsc、tsx、esbuild、Node 原生运行有什么区别？

- **难度**: junior
- **标签**: 工具链

**答案**

| 方式 | 类型检查 | 速度 | 说明 |
|------|---------|------|------|
| `tsc` | 有 | TS 7 起大幅提升 | 官方编译器，检查 + 输出 JS / .d.ts |
| `tsc --noEmit` | 有 | — | 只做检查，常在 CI 中与其他转译工具配合 |
| esbuild / SWC / Babel | **无** | 极快 | 只删除类型，打包工具内部使用 |
| `tsx` / `ts-node` | tsx 无，ts-node 可选 | — | 开发时直接运行 |
| Node 原生（22.18+ / 23.6+ 默认开启） | **无** | 快 | 只做类型擦除，不支持 enum、namespace、参数属性等需要生成代码的语法 |

现代工程的通用模式：**转译和类型检查分离**。

- 开发 / 构建：Vite（esbuild/Rolldown）或 Node 原生直接擦除类型，速度快
- 质量保障：编辑器实时检查 + CI 中 `tsc --noEmit`

为配合 Node 原生运行，可开启 `erasableSyntaxOnly`（5.8+），禁止使用无法直接擦除的语法。

**口述模板（30–60 秒）**

1. 一句话结论：tsc 负责检查和产出，其他工具只负责快速删除类型。
2. 关键机制：转译与类型检查分离。
3. 注意点：只删除类型的工具不会报类型错误，CI 必须跑 `tsc --noEmit`。
4. 可追问时补充：Node 原生运行要求可擦除语法；TS 7 的 tsc 快 8–12 倍，检查环节的瓶颈明显下降。

**常见追问**

- 为什么 Node 原生运行 TS 时 import 要写 `.ts` 扩展名？→ Node 不改写路径；配合 `allowImportingTsExtensions` 或 `rewriteRelativeImportExtensions` 处理。

---

### Q29. 什么是多余属性检查（Excess Property Check）？

- **难度**: junior
- **标签**: 类型兼容

**答案**

结构化类型允许「多出来的属性」，但**直接把对象字面量赋给有类型的位置**时，TS 会额外检查多余属性，用来发现拼写错误：

```ts
interface Opts { width: number }

const o: Opts = { width: 1, heigth: 2 }   // 报错：heigth 是多余属性

const tmp = { width: 1, heigth: 2 }
const o2: Opts = tmp                      // 不报错：不是「新鲜」字面量
```

只对**新鲜的对象字面量（fresh object literal）**生效：一旦赋给中间变量，新鲜性就消失了。

绕过方式（需要时）：中间变量、类型断言、在类型中加索引签名。

**口述模板（30–60 秒）**

1. 一句话结论：对象字面量直接赋值时会检查多余属性，防止拼写错误。
2. 关键机制：只对新鲜字面量生效，是结构化类型的一个例外。
3. 注意点：通过变量中转就不会检查，这是常见的「为什么没报错」。
4. 可追问时补充：联合类型中的多余属性检查较宽松，只要属性在任一成员中存在就可能放过。

**常见追问**

- 函数参数直接传对象字面量会检查吗？→ 会，也属于新鲜字面量。

---

### Q30. 什么是声明合并和模块扩充？

- **难度**: junior
- **标签**: 声明合并

**答案**

**声明合并（Declaration Merging）**：同名的 `interface`、`namespace`、函数与 namespace、类与 interface 等会合并成一个声明。

**模块扩充（Module Augmentation）**：利用合并机制，给第三方库补充类型。

```ts
// express 请求对象上增加 user 字段
import 'express'
declare module 'express-serve-static-core' {
  interface Request {
    user?: { id: string; roles: string[] }
  }
}

// Vue 全局属性
declare module 'vue' {
  interface ComponentCustomProperties {
    $t: (key: string) => string
  }
}
```

要点：

- 扩充文件必须是模块（有 import 或 export）
- 只能**扩充已有声明**，不能新增顶层导出
- 要被 tsconfig 的 `include` 覆盖到

**口述模板（30–60 秒）**

1. 一句话结论：同名 interface 会合并；模块扩充利用这一点给第三方库加类型。
2. 关键机制：`declare module '库名'` 里写同名 interface。
3. 注意点：文件必须是模块，且要被 include。
4. 可追问时补充：扩充是全局生效的，滥用会导致类型与真实运行时不一致。

**常见追问**

- `type` 能被扩充吗？→ 不能，所以库作者对外暴露可扩展的类型时会用 interface。
