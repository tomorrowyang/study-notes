# TypeScript 中级 Q&A（类型编程与工程原理）

> 目标：讲清类型兼容规则、条件类型、infer、映射类型、模板字面量类型、协变逆变、模块解析、声明文件、装饰器和近两年的新特性。中级题的关键是「为什么」和「能写出来」。

---

## 一、类型关系

### Q1. 什么是结构化类型？和名义类型有什么区别？

- **难度**: mid
- **标签**: 类型系统, 高频

**答案**

- **结构化类型（Structural Typing）**：只看结构，结构兼容即可赋值。TS、Go 接口属于这一类
- **名义类型（Nominal Typing）**：看声明的名字或继承关系。Java、C#、Rust 属于这一类

```ts
interface Point { x: number; y: number }
class Vec { constructor(public x: number, public y: number) {} }

const p: Point = new Vec(1, 2)      // OK：结构兼容
```

赋值兼容规则：**源类型至少包含目标类型要求的所有成员**（多了可以，少了不行）。

结构化类型的问题：`UserId` 和 `OrderId` 都是 `string`，容易传错。解决办法是品牌类型（Branded Type）：

```ts
type Brand<T, B extends string> = T & { readonly __brand: B }
type UserId = Brand<string, 'UserId'>
type OrderId = Brand<string, 'OrderId'>

const toUserId = (s: string) => s as UserId

function getUser(id: UserId) {}
getUser(toUserId('u1'))            // OK
// getUser('u1' as OrderId)        // 报错
```

TS 中带 `private` / `#private` 成员的类、`unique symbol` 也表现为名义化。

**口述模板（30–60 秒）**

1. 一句话结论：TS 是结构化类型，只要结构兼容就能赋值。
2. 关键机制：源类型的成员必须覆盖目标类型的要求。
3. 注意点：同为 string 的不同业务 ID 会被混用。
4. 可追问时补充：用品牌类型模拟名义类型，在构造函数或校验函数中统一打标。

**常见追问**

- 为什么 TS 选结构化类型？→ JS 大量使用对象字面量和鸭子类型，结构化类型更贴合 JS 的实际写法。

---

### Q2. 协变、逆变、双变、不变是什么？在 TS 中怎么体现？

- **难度**: mid
- **标签**: 型变, 高频

**答案**

设 `Dog` 是 `Animal` 的子类型（`Dog <: Animal`）：

| 型变 | 含义 | TS 中的位置 |
|------|------|------------|
| 协变（Covariant） | `F<Dog> <: F<Animal>`，方向不变 | 返回值、只读属性、数组元素 |
| 逆变（Contravariant） | `F<Animal> <: F<Dog>`，方向反转 | 函数参数（开启 `strictFunctionTypes`） |
| 双变（Bivariant） | 两个方向都允许 | **方法语法**的参数、关闭 strictFunctionTypes 时的函数参数 |
| 不变（Invariant） | 都不允许 | 既读又写的位置 |

为什么参数是逆变的：

```ts
type Handler = (a: Animal) => void
let handleDog = (d: Dog) => d.bark()
let h: Handler = handleDog   // strictFunctionTypes 下报错
h(new Cat())                 // 如果允许，运行时 Cat 没有 bark
```

一个能处理 `Animal` 的函数，当然能处理 `Dog`；反过来不行。所以参数类型**更宽**的函数才是子类型。

为什么方法是双变的：为了让 `Array<Dog>` 能赋给 `Array<Animal>`（`push(item: T)` 是方法），这是有意保留的**不安全点**。

```ts
interface A { fn(x: Dog): void }     // 方法语法：双变
interface B { fn: (x: Dog) => void } // 属性语法：逆变，更安全
```

**口述模板（30–60 秒）**

1. 一句话结论：返回值协变，参数逆变，方法参数双变。
2. 关键机制：替换原则——能处理宽类型的函数可以替代处理窄类型的函数。
3. 注意点：方法语法和数组协变是 TS 故意保留的不安全点。
4. 可追问时补充：想要严格检查就用属性语法声明函数；TS 4.7 起可用 `in` / `out` 显式标注泛型型变。

**常见追问**

- 数组协变为什么不安全？→ `const animals: Animal[] = dogs; animals.push(new Cat())`，编译通过，但 `dogs` 里混进了猫。

---

### Q3. `never` 有哪些深入用法？

- **难度**: mid
- **标签**: never

**答案**

1. **穷尽检查**：

```ts
function assertNever(x: never): never {
  throw new Error(`Unexpected: ${JSON.stringify(x)}`)
}

function handle(s: Shape) {
  switch (s.kind) {
    case 'circle': return 1
    case 'rect': return 2
    default: return assertNever(s)   // 新增成员时这里会编译报错
  }
}
```

2. **在联合中消失**：`string | never` 等于 `string`。`Exclude` 就是利用这点过滤联合成员

3. **作为分布式条件类型的输入时返回 never**：

```ts
type IsString<T> = T extends string ? true : false
type R = IsString<never>     // never，而不是 false

type IsNever<T> = [T] extends [never] ? true : false   // 正确判断
```

原因：never 是空联合，分发时没有成员可以遍历，结果也是空联合。

4. **在映射类型中通过 `as never` 删除键**（见 Q8）

5. **交叉冲突**：`string & number` 为 never

**口述模板（30–60 秒）**

1. 一句话结论：never 是空集，用于穷尽检查、过滤联合和删除键。
2. 关键机制：联合中自动消失；作为分布式条件类型的输入时直接得到 never。
3. 注意点：判断 never 必须用元组包裹阻止分发。
4. 可追问时补充：穷尽检查能让新增的联合成员在所有 switch 处报错，是重构安全网。

**常见追问**

- `keyof never` 是什么？→ `string | number | symbol`，因为 never 是所有类型的子类型，可以认为它拥有任意键。

---

### Q4. 可辨识联合（Discriminated Union）是什么？为什么推荐？

- **难度**: mid
- **标签**: 可辨识联合, 建模

**答案**

联合的每个成员都有一个**相同名字、字面量类型不同**的字段（判别字段），据此收窄：

```ts
type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error }

function render(s: RequestState<User>) {
  if (s.status === 'success') return s.data.name   // 只有这个分支能访问 data
  if (s.status === 'error') return s.error.message
}
```

对比「全部可选字段」的写法：

```ts
interface Bad { loading: boolean; data?: User; error?: Error }
// 允许 loading=true 且同时有 data 和 error 这种非法状态
```

核心思想：**让非法状态无法表示（Make illegal states unrepresentable）**。

注意：解构后收窄可能丢失。TS 4.6 起对 `const` 解构的判别字段支持关联收窄，但跨函数传递后仍需重新判断。

**口述模板（30–60 秒）**

1. 一句话结论：用一个字面量字段区分联合成员，按它收窄。
2. 关键机制：switch 判别字段 + never 穷尽检查。
3. 注意点：比「一堆可选字段」更安全，能排除非法组合。
4. 可追问时补充：Redux action、请求状态、表单步骤、状态机都适合这样建模。

**常见追问**

- 判别字段可以是什么类型？→ 字面量类型（字符串、数字、布尔）、`null`、`undefined`。

---

## 二、条件类型与 infer

### Q5. 条件类型是什么？什么是分布式条件类型？怎么阻止分发？

- **难度**: mid
- **标签**: 条件类型, 高频

**答案**

条件类型：`T extends U ? X : Y`，如果 T 可赋给 U 则得到 X，否则得到 Y。

**分布式（Distributive）**：当 T 是**裸类型参数**（naked type parameter）且传入联合类型时，会对联合的每个成员分别计算再合并：

```ts
type ToArray<T> = T extends any ? T[] : never
type A = ToArray<string | number>   // string[] | number[]

type ToArray2<T> = [T] extends [any] ? T[] : never
type B = ToArray2<string | number>  // (string | number)[]
```

触发分发的条件：

1. 被检查的类型是**泛型参数本身**（不能是 `T[]`、`[T]`、`Promise<T>`）
2. 在**实例化**时传入的是联合类型

阻止分发：用元组包裹两侧 `[T] extends [U]`。

`Exclude` 就是利用分发实现的：

```ts
type MyExclude<T, U> = T extends U ? never : T
type C = MyExclude<'a' | 'b' | 'c', 'a'>   // 'b' | 'c'
```

**口述模板（30–60 秒）**

1. 一句话结论：条件类型是类型层面的三元表达式，裸类型参数遇到联合会逐个分发。
2. 关键机制：分发相当于对联合做 map，再把结果合并。
3. 注意点：用 `[T] extends [U]` 阻止分发；never 输入会得到 never。
4. 可追问时补充：Exclude、Extract、NonNullable 都依赖分发。

**常见追问**

- `boolean` 会被分发吗？→ 会，`boolean` 本质是 `true | false`。

---

### Q6. `infer` 怎么用？常见模式有哪些？

- **难度**: mid
- **标签**: infer, 高频

**答案**

`infer` 只能出现在条件类型的 `extends` 子句中，用于**声明一个待推断的类型变量**，相当于类型层面的模式匹配。

```ts
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never
type MyParameters<T> = T extends (...args: infer P) => any ? P : never
type ElementOf<T> = T extends readonly (infer E)[] ? E : never
type First<T> = T extends [infer F, ...any[]] ? F : never
type Last<T> = T extends [...any[], infer L] ? L : never
type UnwrapPromise<T> = T extends Promise<infer V> ? UnwrapPromise<V> : T

// 模板字面量中匹配
type GetPrefix<S> = S extends `${infer P}_${string}` ? P : never
type X = GetPrefix<'user_id'>   // 'user'

// infer + extends 约束（4.7+）
type FirstString<T> = T extends [infer S extends string, ...unknown[]] ? S : never
```

同一个类型变量多次出现时：

- **协变位置**（属性、返回值）：推断为**联合**
- **逆变位置**（函数参数）：推断为**交叉**

```ts
type Co<T> = T extends { a: infer U; b: infer U } ? U : never
type R1 = Co<{ a: string; b: number }>          // string | number

type Contra<T> = T extends { a: (x: infer U) => void; b: (x: infer U) => void } ? U : never
type R2 = Contra<{ a: (x: { id: 1 }) => void; b: (x: { name: 'a' }) => void }>   // { id: 1 } & { name: 'a' }
```

这正是 `UnionToIntersection` 的原理（见手写题 H4）。

**口述模板（30–60 秒）**

1. 一句话结论：infer 在条件类型里做模式匹配，把某个位置的类型捕获出来。
2. 关键机制：匹配成功走 true 分支并能使用捕获的类型。
3. 注意点：多处推断时，协变位置得到联合，逆变位置得到交叉。
4. 可追问时补充：`infer X extends string` 可以在捕获时加约束，避免再嵌套一层判断。

**常见追问**

- 函数有重载时 `ReturnType` 取哪个？→ 取**最后一个**重载签名。

---

### Q7. 怎么实现常用工具类型？

- **难度**: mid
- **标签**: 工具类型, 手写

**答案**

```ts
type MyPartial<T> = { [K in keyof T]?: T[K] }
type MyRequired<T> = { [K in keyof T]-?: T[K] }
type MyReadonly<T> = { readonly [K in keyof T]: T[K] }
type MyPick<T, K extends keyof T> = { [P in K]: T[P] }
type MyRecord<K extends keyof any, V> = { [P in K]: V }
type MyExclude<T, U> = T extends U ? never : T
type MyExtract<T, U> = T extends U ? T : never
type MyOmit<T, K extends keyof any> = MyPick<T, MyExclude<keyof T, K>>
type MyNonNullable<T> = T & {}
type MyAwaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F, ...args: any[]): any }
    ? F extends (value: infer V, ...args: any[]) => any ? MyAwaited<V> : never
    : T
```

要点：

- `-?` 去掉可选，`-readonly` 去掉只读
- `NonNullable` 在 TS 4.8 起改为 `T & {}`，因为 `{}` 排除了 null 和 undefined
- 官方 `Awaited` 匹配的是任意 thenable 对象，而不只是 `Promise`

**口述模板（30–60 秒）**

1. 一句话结论：对象类工具基于映射类型，联合类工具基于分布式条件类型，函数类工具基于 infer。
2. 关键机制：`keyof` 遍历键，`-?` 和 `-readonly` 去修饰符。
3. 注意点：Omit 的 K 不受约束；Awaited 要递归并处理 thenable。
4. 可追问时补充：可以写一个严格版 `StrictOmit<T, K extends keyof T>`。

**常见追问**

- 为什么 `Omit` 的 K 是 `keyof any` 而不是 `keyof T`？→ 设计上允许删除不存在的键，方便在泛型中使用；代价是拼写错误不会报错。

---

## 三、映射类型与模板字面量类型

### Q8. 映射类型有哪些高级用法？key remapping 是什么？

- **难度**: mid
- **标签**: 映射类型

**答案**

基本形式：`{ [K in Keys]: Type }`。高级用法：

```ts
// 1. 修饰符增删
type Mutable<T> = { -readonly [K in keyof T]: T[K] }

// 2. key remapping（4.1+）：用 as 改写键名
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K]
}
type G = Getters<{ name: string }>   // { getName: () => string }

// 3. 通过 as never 过滤键
type PickByValue<T, V> = {
  [K in keyof T as T[K] extends V ? K : never]: T[K]
}
type Fns = PickByValue<{ a: 1; b: () => void }, Function>   // { b: () => void }

// 4. 遍历联合（不一定是 keyof）
type EventMap<E extends { type: string }> = {
  [Ev in E as Ev['type']]: (e: Ev) => void
}
```

**同态映射类型（Homomorphic Mapped Type）**：形如 `{ [K in keyof T]: ... }` 时，会保留 T 原有的可选和只读修饰符；作用于数组或元组时，结果仍然是数组或元组。

`string & K` 的原因：`keyof T` 可能包含 `number` 和 `symbol`，而模板字面量只接受 `string | number | bigint | boolean | null | undefined`。

**口述模板（30–60 秒）**

1. 一句话结论：映射类型遍历键生成新类型，as 可以改名或过滤。
2. 关键机制：`as never` 删除键，`-?`/`-readonly` 去修饰符。
3. 注意点：同态映射保留修饰符，并能正确处理数组和元组。
4. 可追问时补充：可以用联合中的对象作为遍历源，把事件联合转成处理器映射。

**常见追问**

- `Partial<string[]>` 的结果是什么？→ 同态映射作用于数组，得到 `(string | undefined)[]`。

---

### Q9. 模板字面量类型能做什么？

- **难度**: mid
- **标签**: 模板字面量

**答案**

TS 4.1+ 支持在类型中使用模板字符串，可以拼接、匹配和解析字符串字面量：

```ts
type Lang = 'zh' | 'en'
type Page = 'home' | 'about'
type Route = `/${Lang}/${Page}`         // 4 种组合，联合会做笛卡尔积

type EventName<T extends string> = `on${Capitalize<T>}`   // 'click' → 'onClick'

// 解析路由参数
type Params<S extends string> =
  S extends `${string}:${infer P}/${infer Rest}`
    ? P | Params<Rest>
    : S extends `${string}:${infer P}`
      ? P
      : never

type P = Params<'/user/:id/post/:postId'>   // 'id' | 'postId'

// 约束 CSS 单位
type Px = `${number}px`
const w: Px = '12px'
```

内置字符串工具类型：`Uppercase`、`Lowercase`、`Capitalize`、`Uncapitalize`。

注意：

- 联合做笛卡尔积，组合数过多会导致检查变慢甚至报「联合类型过于复杂」
- TS 7.0 起，`${infer H}${infer T}` 按 Unicode 码点拆分，emoji 不再被拆成两个代理项

**口述模板（30–60 秒）**

1. 一句话结论：模板字面量类型可以在类型层拼接和解析字符串。
2. 关键机制：联合做笛卡尔积，配合 infer 做模式匹配。
3. 注意点：组合爆炸会拖慢检查。
4. 可追问时补充：常用于事件名、路由参数、i18n key、CSS 值约束。

**常见追问**

- `` `${number}` `` 能匹配什么？→ 能被解析为数字的字符串，如 `'1'`、`'1.5'`、`'1e3'`。

---

### Q10. 递归类型有什么限制？「类型实例化过深」怎么处理？

- **难度**: mid
- **标签**: 递归类型, 性能

**答案**

类型可以递归引用自身：

```ts
type Json = string | number | boolean | null | Json[] | { [k: string]: Json }

type DeepReadonly<T> = T extends (...args: any[]) => any
  ? T
  : T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T
```

限制：

- 普通条件类型递归深度约 **50 层**；TS 4.5 起对**尾递归**的条件类型优化到约 **1000 层**
- 超出会报 `Type instantiation is excessively deep and possibly infinite`（TS2589）

尾递归写法：递归调用必须直接处于条件类型的分支位置，把中间结果放到累加器参数中。

```ts
// 非尾递归：递归结果被包在元组里
type Rev1<T extends unknown[]> = T extends [infer H, ...infer R] ? [...Rev1<R>, H] : []

// 尾递归：用累加器
type Rev2<T extends unknown[], Acc extends unknown[] = []> =
  T extends [infer H, ...infer R] ? Rev2<R, [H, ...Acc]> : Acc
```

处理思路：改为尾递归、限制递归深度参数、在业务中放弃过于精确的类型。

**口述模板（30–60 秒）**

1. 一句话结论：类型可以递归，但有深度限制，尾递归能到约 1000 层。
2. 关键机制：用累加器把递归调用放到分支的直接位置。
3. 注意点：深递归会显著拖慢 IDE。
4. 可追问时补充：给递归加深度计数器，超过就返回宽松类型。

**常见追问**

- 怎么在类型中计数？→ 用元组长度：`[...Acc, unknown]['length']`。

---

## 四、泛型进阶与新特性

### Q11. 泛型推断是怎么工作的？什么时候推断不出来？

- **难度**: mid
- **标签**: 泛型, 推断

**答案**

调用泛型函数时，编译器从**实参**中收集类型参数的候选，再选出最佳结果：

```ts
function pair<T>(a: T, b: T) { return [a, b] }
pair(1, 2)          // T = number
pair(1, 'a')        // 报错：'a' 不能赋给 number（以第一个候选为准）

function useState<S>(init: S | (() => S)) {}
useState(null)      // S = null，后续 set 对象会报错 → 需要 useState<User | null>(null)
```

常见推断失败或不符合预期的场景：

1. **初始值不完整**：`useState(null)`、`useState([])`（推断为 `never[]`）
2. **字面量被拓宽**：期望 `'a' | 'b'`，得到 `string` → 用 `const` 类型参数（Q12）
3. **从不希望的位置推断**：默认值参数干扰推断 → 用 `NoInfer`（Q13）
4. **上下文相关函数（contextually sensitive）**：参数无标注的回调会被延后推断；TS 6.0 起，不使用 `this` 的方法语法不再算作上下文敏感，推断更符合直觉
5. **只出现在返回值的类型参数**：无法从实参推断，只能显式传入

**口述模板（30–60 秒）**

1. 一句话结论：编译器从实参收集候选再选最佳类型，回调的参数会延后推断。
2. 关键机制：上下文类型（contextual typing）让回调参数从泛型推断结果中拿到类型。
3. 注意点：初始值不完整时需要显式传泛型。
4. 可追问时补充：`const T` 保留字面量，`NoInfer` 屏蔽某个位置的推断。

**常见追问**

- 显式传入部分类型参数可以吗？→ 不行，要么全传，要么全靠推断（部分推断提案尚未落地）；常见解法是柯里化拆成两次调用。

---

### Q12. `const` 类型参数解决什么问题？

- **难度**: mid
- **标签**: const 泛型, 5.0

**答案**

TS 5.0+：在类型参数前加 `const`，让推断像 `as const` 一样保留字面量和元组：

```ts
function routes<T extends readonly string[]>(r: T) { return r }
const a = routes(['home', 'about'])          // string[]

function routes2<const T extends readonly string[]>(r: T) { return r }
const b = routes2(['home', 'about'])         // readonly ['home', 'about']
```

注意：

- 约束最好写成 `readonly` 数组，否则 const 推断出的只读元组可能不满足约束而回退
- 只对**直接写在调用处的字面量**有效，传入变量不会改变变量的类型

典型场景：路由表、事件名列表、配置 DSL、状态机定义、`defineConfig` 类 API。

**口述模板（30–60 秒）**

1. 一句话结论：const 类型参数让调用方不用写 `as const` 也能推断出字面量。
2. 关键机制：对调用处的字面量做常量推断。
3. 注意点：约束要用 readonly 数组。
4. 可追问时补充：适合库作者设计 API，提升调用方体验。

**常见追问**

- 和让调用方自己写 `as const` 比有什么好处？→ API 更易用，不依赖使用者记得写断言。

---

### Q13. `NoInfer<T>` 解决什么问题？

- **难度**: mid
- **标签**: NoInfer, 5.4

**答案**

TS 5.4+：标记某个位置**不参与**类型参数推断。

```ts
function createFSM<S extends string>(states: S[], initial: S) {}
createFSM(['idle', 'run'], 'stop')       // S 被推断为 'idle' | 'run' | 'stop'，不报错

function createFSM2<S extends string>(states: S[], initial: NoInfer<S>) {}
createFSM2(['idle', 'run'], 'stop')      // 报错：'stop' 不在 'idle' | 'run' 中
```

原理：`NoInfer<T>` 是编译器内置的特殊类型，推断阶段忽略它，检查阶段等同于 `T`。

5.4 之前的替代写法：`initial: S & {}`，或者增加一个类型参数 `I extends S`。

**口述模板（30–60 秒）**

1. 一句话结论：NoInfer 让某个参数只用来检查，不用来推断。
2. 关键机制：推断时忽略，检查时等同于 T。
3. 注意点：用在「默认值」「初始值」这类应该被其他参数约束的位置。
4. 可追问时补充：库 API 设计中常见，比如 `defaultValue` 必须是 `options` 中的一项。

**常见追问**

- 不用 NoInfer 时为什么会被拓宽？→ 两个位置都是推断候选，编译器会取它们的联合（字符串字面量场景下）。

---

### Q14. `using` 声明是什么？

- **难度**: mid
- **标签**: using, 资源管理, 5.2

**答案**

TS 5.2+ 支持 TC39 显式资源管理提案：`using` 声明的变量在**离开作用域时自动调用** `[Symbol.dispose]()`，异步版本 `await using` 调用 `[Symbol.asyncDispose]()`。

```ts
function openFile(path: string) {
  const fd = fs.openSync(path, 'r')
  return {
    fd,
    [Symbol.dispose]() { fs.closeSync(fd) },
  }
}

function read() {
  using file = openFile('a.txt')
  // ... 无论正常返回还是抛错，退出作用域时都会关闭文件
}

async function query() {
  await using conn = await pool.connect()   // conn 实现 Symbol.asyncDispose
}
```

- 多个 `using` 按**声明的相反顺序**释放
- 释放过程中出错会被包装为 `SuppressedError`
- 需要 lib 包含 `esnext.disposable`（或较新的 `es` 版本），以及运行时支持或 polyfill

**口述模板（30–60 秒）**

1. 一句话结论：using 是语言级的 try/finally，离开作用域自动释放资源。
2. 关键机制：调用 `Symbol.dispose` / `Symbol.asyncDispose`，逆序释放。
3. 注意点：需要运行时支持或 polyfill。
4. 可追问时补充：适合文件句柄、数据库连接、锁、临时订阅、测试中的 mock 恢复。

**常见追问**

- 和 `try/finally` 比有什么好处？→ 多个资源时不需要嵌套，释放顺序和错误聚合由语言保证。

---

### Q15. 装饰器在 TS 5.0 之后有什么变化？

- **难度**: mid
- **标签**: 装饰器

**答案**

TS 中存在两套装饰器：

| 维度 | 旧版实验性装饰器 | 标准装饰器（5.0+） |
|------|---------------|------------------|
| 开启方式 | `experimentalDecorators: true` | 默认可用 |
| 标准 | TC39 早期草案 | TC39 Stage 3 |
| 函数签名 | `(target, key, descriptor)` | `(value, context)` |
| 参数装饰器 | 支持 | **不支持** |
| 元数据 | `emitDecoratorMetadata` + `reflect-metadata` | `context.metadata`（5.2+，`Symbol.metadata`） |

```ts
function logged<This, Args extends unknown[], R>(
  target: (this: This, ...args: Args) => R,
  context: ClassMethodDecoratorContext<This, (this: This, ...args: Args) => R>,
) {
  return function (this: This, ...args: Args): R {
    console.log(`call ${String(context.name)}`)
    return target.call(this, ...args)
  }
}

class Svc {
  @logged
  run(n: number) { return n * 2 }
}
```

现状：NestJS、TypeORM、Angular 等依赖依赖注入的框架长期使用实验性装饰器 + 参数装饰器 + `emitDecoratorMetadata`。标准装饰器不支持参数装饰器，迁移需要等框架跟进。

**口述模板（30–60 秒）**

1. 一句话结论：5.0 起支持标准装饰器，签名变为 `(value, context)`，与旧版不兼容。
2. 关键机制：装饰器返回替换值，通过 context 拿到名称、种类、addInitializer。
3. 注意点：标准版没有参数装饰器，依赖注入框架仍在用旧版。
4. 可追问时补充：`emitDecoratorMetadata` 依赖类型信息，单文件转译工具支持有限。

**常见追问**

- 装饰器的执行顺序？→ 同一目标上的多个装饰器从下往上（从近到远）应用。

---

### Q16. 推断类型谓词（5.5）是什么？`filter(Boolean)` 为什么不能收窄？

- **难度**: mid
- **标签**: 类型谓词, 5.5

**答案**

TS 5.5 起，编译器能从函数体自动推断出 `x is T`：

```ts
const xs = [1, 2, undefined]
const a = xs.filter(x => x !== undefined)   // 5.5+：number[]
```

推断条件：函数没有显式返回类型、只有一个 return、不修改参数，且返回 true 时能收窄为 T、返回 false 时能排除 T。

`filter(Boolean)` 仍然不行：

```ts
const b = xs.filter(Boolean)   // (number | undefined)[]
```

原因：`Boolean` 是一个已经声明好的构造函数，签名是 `(value?: unknown) => boolean`，没有类型谓词。另外 `0` 也会被过滤掉，与「去掉 undefined」语义不同，所以编译器不能推断。

解决：

```ts
const isDefined = <T>(x: T): x is NonNullable<T> => x != null
xs.filter(isDefined)
```

**口述模板（30–60 秒）**

1. 一句话结论：5.5 起简单的判断箭头函数会自动推断类型谓词。
2. 关键机制：要求返回 true 和 false 时都能精确收窄。
3. 注意点：`filter(Boolean)` 没有谓词，也会误删 0 和空字符串。
4. 可追问时补充：项目中封装 `isDefined` 工具函数。

**常见追问**

- 为什么 `x => !!x` 不能推断？→ 返回 false 时不能确定排除了 number（0 也是 false），不满足双向条件。

---

## 五、模块与声明

### Q17. `moduleResolution` 的 `nodenext` 和 `bundler` 有什么区别？

- **难度**: mid
- **标签**: 模块解析, 高频

**答案**

| 维度 | `nodenext` / `node16` | `bundler` |
|------|----------------------|-----------|
| 对象 | 直接在 Node 中运行的代码、要发布的库 | 交给 Vite/webpack/Rolldown 打包的代码 |
| 相对导入扩展名 | ESM 中**必须写**（如 `./a.js`） | 可以不写 |
| `package.json` `exports` | 支持，区分 `import` / `require` 条件 | 支持，使用 `import` 条件 |
| 判断 ESM / CJS | 按 `.mts/.cts` 和 `package.json` 的 `type` 字段 | 不严格区分 |
| 搭配的 module | `nodenext` | `esnext` / `preserve`（6.0 起也可搭配 `commonjs`） |

ESM 中写 `.js` 的原因：TS **不改写导入路径**，输出的 JS 中路径原样保留，而 Node ESM 要求完整路径。所以源码里写的是编译后文件的扩展名。

TS 6.0 起 `moduleResolution: node`（即 node10）和 `classic` 已废弃，7.0 中直接报错。

**口述模板（30–60 秒）**

1. 一句话结论：Node 运行和发布库用 nodenext，打包器项目用 bundler。
2. 关键机制：nodenext 严格模拟 Node 的 ESM/CJS 解析规则，bundler 模拟打包器的宽松规则。
3. 注意点：nodenext 下 ESM 相对导入要写 `.js`，因为 TS 不改写路径。
4. 可追问时补充：旧的 node10 解析在 6.0 废弃、7.0 删除，升级时要处理。

**常见追问**

- 库项目可以用 `bundler` 吗？→ 不推荐，可能生成在 Node 中无法直接解析的产物；库应使用 nodenext 保证兼容。

---

### Q18. 发布一个 TS 库时，类型声明怎么配置？

- **难度**: mid
- **标签**: 库发布, exports

**答案**

```jsonc
{
  "name": "my-lib",
  "type": "module",
  "exports": {
    ".": {
      "import": {
        "types": "./dist/index.d.ts",
        "default": "./dist/index.js"
      },
      "require": {
        "types": "./dist/index.d.cts",
        "default": "./dist/index.cjs"
      }
    }
  },
  "files": ["dist"]
}
```

要点：

1. `types` 条件**必须放在每个条件对象的最前面**（条件按顺序匹配）
2. ESM 和 CJS 产物要配**各自的**声明文件（`.d.ts` / `.d.cts`），否则会出现「声明的模块格式与实际不符」（Masquerading as ESM/CJS）
3. 开启 `declaration`、`declarationMap`（支持跳转到源码）
4. 用 **are-the-types-wrong**（`attw`）和 `publint` 检查产物
5. 类型依赖的 `@types/*` 要放在 `dependencies` 或 `peerDependencies`，而不是 `devDependencies`

**口述模板（30–60 秒）**

1. 一句话结论：通过 exports 的 types 条件给 ESM 和 CJS 分别提供声明。
2. 关键机制：types 条件放最前，声明文件格式要与 JS 产物对应。
3. 注意点：用 attw 检查，类型依赖不能放在 devDependencies。
4. 可追问时补充：tsup / tsdown 可以一次产出双格式及对应声明。

**常见追问**

- 只有 `types` 顶层字段够吗？→ 有 `exports` 时，新版解析策略以 `exports` 为准，顶层 `types` 只作为旧解析的兜底。

---

### Q19. `isolatedModules` 是什么？为什么打包器项目需要开启？

- **难度**: mid
- **标签**: 转译

**答案**

Babel、esbuild、SWC 等工具**一次只处理一个文件**，不了解其他文件的类型信息。`isolatedModules: true` 会对「需要跨文件类型信息才能正确转译」的写法报错：

1. **重新导出类型**：`export { User } from './types'`，单文件工具不知道 User 是类型 → 改为 `export type`
2. **`const enum`** 跨文件使用：无法内联 → 避免使用
3. **非模块文件**：没有 import/export 的文件 → 加 `export {}`

相关配置的关系：

- `isolatedModules`：保证单文件转译安全
- `verbatimModuleSyntax`：更严格，导入导出按字面保留（隐含 isolatedModules 的大部分要求）
- `erasableSyntaxOnly`：更进一步，只允许能直接删除的类型语法（为 Node 原生运行 TS 设计）
- `isolatedDeclarations`：保证 `.d.ts` 也能单文件生成（见高级题）

**口述模板（30–60 秒）**

1. 一句话结论：isolatedModules 禁止需要跨文件类型信息才能转译的写法。
2. 关键机制：单文件转译工具不做类型分析。
3. 注意点：类型重导出要用 export type，避免 const enum。
4. 可追问时补充：从 isolatedModules 到 verbatimModuleSyntax 再到 erasableSyntaxOnly，约束逐级收紧。

**常见追问**

- Vite 项目必须开启吗？→ 建议开启，Vite 用 esbuild/Oxc 转译，不开启可能编译通过但运行时出错。

---

### Q20. ESM 与 CJS 互操作中，TS 需要注意什么？

- **难度**: mid
- **标签**: 模块互操作

**答案**

1. **默认导入 CJS 模块**：CJS 的 `module.exports = fn` 没有 default。`esModuleInterop` 让 `import fn from 'cjs-lib'` 正常工作（生成辅助函数）。TS 6.0 起不能再把 `esModuleInterop` 和 `allowSyntheticDefaultImports` 设为 `false`
2. **`nodenext` 下 CJS 文件导入 ESM**：Node 22+ 支持 `require(esm)`，TS 5.8 起在 nodenext 下允许
3. **`import x = require()`**：TS 特有语法，用于 CJS 输出时保持 `module.exports =` 的精确类型
4. **双包问题（Dual Package Hazard）**：同一个包的 ESM 和 CJS 版本同时被加载，导致单例和 `instanceof` 失效，类型声明也要分别对应
5. **`.mts` / `.cts`**：显式指定单个文件的模块格式，对应输出 `.mjs` / `.cjs` 和 `.d.mts` / `.d.cts`

**口述模板（30–60 秒）**

1. 一句话结论：核心是 CJS 没有真正的 default 导出，靠 esModuleInterop 抹平。
2. 关键机制：nodenext 按文件扩展名和 type 字段判断模块格式。
3. 注意点：双包问题会破坏单例；声明文件格式要与实际产物一致。
4. 可追问时补充：新库建议只发 ESM，借助 Node 的 require(esm) 兼容 CJS 使用者。

**常见追问**

- `export =` 和 `export default` 的区别？→ 前者对应 CJS 的 `module.exports =`，后者对应 ESM 默认导出（在 CJS 中是 `exports.default`）。

---

### Q21. 如何为没有类型的第三方 JS 库补类型？

- **难度**: mid
- **标签**: 声明文件

**答案**

按优先级：

1. 查找 `@types/xxx`
2. 快速兜底：`declare module 'xxx'`（整个模块为 any，只适合临时）
3. 按实际用到的 API 手写声明：

```ts
// types/legacy-chart.d.ts
declare module 'legacy-chart' {
  export interface ChartOptions {
    el: HTMLElement
    data: Array<{ x: number; y: number }>
    theme?: 'light' | 'dark'
  }
  export class Chart {
    constructor(opts: ChartOptions)
    render(): void
    on(event: 'click', cb: (point: { x: number; y: number }) => void): this
    destroy(): void
  }
  export default Chart
}
```

4. 库本身有 JSDoc 时，可以用 `tsc --allowJs --declaration --emitDeclarationOnly` 生成初版声明
5. 长期使用的库，可以向上游或 DefinitelyTyped 提交 PR

注意：手写声明是**承诺而非事实**，要与运行时行为核对，并在库升级时同步更新。

**口述模板（30–60 秒）**

1. 一句话结论：先找 @types，再按用到的 API 手写 declare module。
2. 关键机制：声明文件只描述类型，需要被 tsconfig 包含。
3. 注意点：手写声明可能与实现不一致，要随版本维护。
4. 可追问时补充：用 `--emitDeclarationOnly` 从 JSDoc 生成初版。

**常见追问**

- 库的类型有错怎么临时修复？→ 用模块扩充补充，或用 `patch-package` 修补声明文件，并向上游报告。

---

## 六、工程实践

### Q22. 项目引用（Project References）和增量编译是什么？

- **难度**: mid
- **标签**: 构建, Monorepo

**答案**

**增量编译**：`incremental: true` 把上次的检查信息写入 `.tsbuildinfo`，下次只重新检查受影响的文件。

**项目引用**：把大项目拆成多个 TS 子项目，声明依赖关系，用 `tsc -b` 按依赖顺序构建：

```jsonc
// packages/app/tsconfig.json
{
  "compilerOptions": { "composite": true, "outDir": "dist" },
  "references": [{ "path": "../shared" }]
}
```

- `composite: true`：要求开启 `declaration`，所有文件必须被 include 覆盖
- 下游项目依赖上游的 **`.d.ts`**，而不是重新检查上游源码 → 检查范围变小
- `tsc -b` 只重新构建发生变化的项目
- TS 7.0 的 `--builders` 可以并行构建没有依赖关系的项目

代价：配置复杂；上游必须先生成声明；IDE 跳转需要 `declarationMap`。

**口述模板（30–60 秒）**

1. 一句话结论：增量编译缓存检查结果，项目引用把大项目拆成可以独立构建的子项目。
2. 关键机制：下游只读上游的 .d.ts，tsc -b 按依赖图构建。
3. 注意点：composite 要求生成声明，配置成本较高。
4. 可追问时补充：TS 7 的 builders 并行构建；isolatedDeclarations 能让声明生成不依赖类型检查。

**常见追问**

- Monorepo 一定要用项目引用吗？→ 不一定。很多团队用 workspace 包 + 打包器直接引用源码，只在 CI 中整体检查；项目引用适合规模很大、检查很慢的情况。

---

### Q23. 如何治理项目中的 `any`？

- **难度**: mid
- **标签**: 代码质量

**答案**

替代方案：

| 场景 | 替代 |
|------|------|
| 类型确实未知 | `unknown` + 收窄 |
| 通用函数 | 泛型 |
| 任意对象 | `Record<string, unknown>` |
| 第三方库无类型 | 写声明文件 |
| 确认类型正确但编译器推不出 | 局部断言并注释原因 |
| 预期报错 | `// @ts-expect-error 原因`（不再报错时会提醒删除） |

工具：

- typescript-eslint：`no-explicit-any`、`no-unsafe-assignment`、`no-unsafe-member-access`、`no-unsafe-call`、`no-unsafe-return`（后几条用于防止 any 扩散）
- `type-coverage` 统计类型覆盖率，在 CI 中设置**只升不降**的阈值（棘轮策略）
- 禁止 `@ts-ignore`，只允许带说明的 `@ts-expect-error`

隐式 any 来源：`JSON.parse`、`res.json()`、`catch` 变量（未开启 strict 时）、无类型的第三方库、`Function` 类型。

**口述模板（30–60 秒）**

1. 一句话结论：用 unknown、泛型、声明文件替代 any，用 lint 和覆盖率阈值防止新增。
2. 关键机制：no-unsafe 系列规则阻止 any 扩散到更多代码。
3. 注意点：ts-expect-error 优于 ts-ignore，因为问题修好后会提示删除。
4. 可追问时补充：存量治理用棘轮策略，只要求不变差，逐步修复。

**常见追问**

- `@ts-ignore` 和 `@ts-expect-error` 的区别？→ 下一行没有错误时，后者会报错，避免过期的忽略注释。

---

### Q24. 如何实现泛型 React 组件？多态组件（`as` 属性）怎么写类型？

- **难度**: mid
- **标签**: React, 泛型组件

**答案**

泛型组件：

```tsx
interface ListProps<T> {
  items: T[]
  getKey: (item: T) => string
  renderItem: (item: T) => React.ReactNode
}

function List<T>({ items, getKey, renderItem }: ListProps<T>) {
  return <ul>{items.map(i => <li key={getKey(i)}>{renderItem(i)}</li>)}</ul>
}

<List items={users} getKey={u => u.id} renderItem={u => u.name} />   // T 推断为 User
```

多态组件：

```tsx
type PolymorphicProps<E extends React.ElementType, P = {}> = P & {
  as?: E
} & Omit<React.ComponentPropsWithoutRef<E>, keyof P | 'as'>

function Text<E extends React.ElementType = 'span'>(
  { as, ...rest }: PolymorphicProps<E, { size?: 'sm' | 'lg' }>,
) {
  const Comp = as ?? 'span'
  return <Comp {...rest} />
}

<Text as="a" href="/x" />        // href 合法
// <Text as="span" href="/x" />  // 报错
```

注意：

- `forwardRef` 会丢失泛型，旧版本需要类型断言；React 19 中 `ref` 是普通 prop，泛型组件更简单
- 多态组件的类型计算较重，组件库中大量使用会拖慢 IDE

**口述模板（30–60 秒）**

1. 一句话结论：泛型组件让 props 之间保持类型关联，多态组件根据 as 推导可用属性。
2. 关键机制：从 items 推断 T，回调参数自动得到类型。
3. 注意点：forwardRef 丢泛型；多态类型有性能成本。
4. 可追问时补充：React 19 的 ref 作为 prop 简化了这类组件。

**常见追问**

- 为什么要 `Omit<..., keyof P>`？→ 避免自定义 props 和原生属性同名时类型冲突（例如自定义的 `size` 与 input 的 `size`）。

---

### Q25. `readonly`、`as const`、`Readonly<T>`、`Object.freeze` 有什么区别？

- **难度**: mid
- **标签**: 只读

**答案**

| 写法 | 层面 | 深度 | 作用对象 |
|------|------|------|---------|
| `readonly` 修饰符 | 类型 | 浅 | 单个属性或数组类型 |
| `Readonly<T>` | 类型 | 浅 | 整个对象类型 |
| `ReadonlyArray<T>` / `readonly T[]` | 类型 | 浅 | 数组：移除 push 等修改方法 |
| `as const` | 类型 | **深**（对字面量递归生效） | 表达式 |
| `Object.freeze` | **运行时** | 浅 | 对象 |

```ts
const cfg = { db: { host: 'a' } } as const
// cfg.db.host = 'b'      // 报错：深层只读

const f = Object.freeze({ db: { host: 'a' } })
f.db.host = 'b'           // 类型上允许，运行时也允许（浅冻结）
```

注意：`readonly` 只是编译期约束。只读类型可以赋给可变类型的变量（对象属性层面 TS 不检查这一点），因此不能依赖它做安全隔离。

**口述模板（30–60 秒）**

1. 一句话结论：readonly 系列是编译期浅只读，as const 是深只读字面量，freeze 是运行时浅冻结。
2. 关键机制：ReadonlyArray 去掉修改方法。
3. 注意点：对象的只读属性可以通过赋值给可变类型被绕过。
4. 可追问时补充：需要深只读类型时手写 DeepReadonly。

**常见追问**

- `readonly string[]` 能传给 `string[]` 参数吗？→ 不能，因为数组类型会检查修改方法；但普通对象的 readonly 属性不会做这种检查。

---

### Q26. `abstract class` 和 `interface` 怎么选？

- **难度**: mid
- **标签**: 类, 设计

**答案**

| 维度 | `interface` | `abstract class` |
|------|-------------|------------------|
| 运行时 | 不存在 | 存在（生成类） |
| 实现代码 | 不能有 | 可以有共享实现 |
| 多继承 | 类可以 implements 多个 | 只能 extends 一个 |
| 构造函数和字段初始化 | 不能 | 可以 |
| `instanceof` | 不能 | 能 |

选择：

- 只需要**约定结构**：interface
- 需要**共享实现**（模板方法模式）或运行时判断：abstract class
- 依赖注入框架需要运行时令牌：abstract class 可以直接作为令牌，interface 需要额外的 Symbol 令牌

**口述模板（30–60 秒）**

1. 一句话结论：interface 是纯类型契约，抽象类是带部分实现的运行时类。
2. 关键机制：类可以实现多个接口，但只能继承一个类。
3. 注意点：interface 不能用于 instanceof。
4. 可追问时补充：DI 场景中抽象类可作为注入令牌。

**常见追问**

- `implements` 会影响类的类型推断吗？→ 不会，它只做检查。类中方法参数不会从接口获得上下文类型。

---

### Q27. 函数重载、联合参数、泛型 + 条件类型怎么选？

- **难度**: mid
- **标签**: API 设计

**答案**

```ts
// 1. 返回值不随参数变化：联合参数
function len(x: string | unknown[]): number { return x.length }

// 2. 返回值随参数变化，情况有限：重载
function get(id: string): User
function get(ids: string[]): User[]
function get(x: string | string[]) { /* ... */ }

// 3. 返回值随参数变化，需要通用表达：泛型 + 条件类型
type Result<T> = T extends string ? User : User[]
function get2<T extends string | string[]>(x: T): Result<T> {
  return null as any   // 实现内部通常需要断言
}
```

取舍：

- 重载：调用方提示清晰，但**不能接收联合参数**（`get(Math.random() ? 'a' : ['a'])` 无法匹配任何重载，需要再加一个联合重载）
- 泛型 + 条件类型：表达力强，但实现内部需要断言，错误信息也更难懂
- 能用联合就不用重载，能用重载就不用条件类型

**口述模板（30–60 秒）**

1. 一句话结论：返回值不变用联合，有限情况用重载，通用映射用泛型条件类型。
2. 关键机制：重载按顺序匹配，条件类型按类型参数计算。
3. 注意点：重载不接受联合实参；条件返回类型在实现内部需要断言。
4. 可追问时补充：优先考虑调用方的体验和错误信息的可读性。

**常见追问**

- 为什么条件返回类型在实现内要断言？→ 实现内 T 未确定，编译器无法把具体分支的值和未解析的条件类型对应起来（TS 后续版本在逐步改进）。

---

### Q28. `this` 类型和 `ThisType` 怎么用？

- **难度**: mid
- **标签**: this

**答案**

1. **`this` 参数**：声明函数中 this 的类型

```ts
function handler(this: HTMLElement) { this.classList.add('on') }
```

2. **多态 `this` 类型**：链式调用返回子类类型

```ts
class Builder {
  set(k: string, v: unknown): this { return this }
}
class SqlBuilder extends Builder { where() { return this } }
new SqlBuilder().set('a', 1).where()   // OK：set 返回 SqlBuilder
```

3. **`ThisType<T>`**：给对象字面量中的方法指定 this 类型，常见于 Vue 2 Options API、Pinia 的 actions

```ts
type Store<S, A> = { state: S; actions: A & ThisType<S & A> }
function defineStore<S, A>(opts: Store<S, A>) { return opts }

defineStore({
  state: { count: 0 },
  actions: { inc() { this.count++ } },   // this 是 state & actions
})
```

`ThisType` 需要开启 `noImplicitThis`。

**口述模板（30–60 秒）**

1. 一句话结论：this 参数声明函数的 this，多态 this 支持链式调用，ThisType 给对象字面量方法指定 this。
2. 关键机制：this 参数编译后被删除。
3. 注意点：箭头函数没有自己的 this，不受这些声明影响。
4. 可追问时补充：Vue 和 Pinia 的类型推导大量使用 ThisType。

**常见追问**

- 把类方法作为回调传出去为什么会丢失 this？→ 运行时问题，TS 默认不检查；可用 `this` 参数或箭头函数属性避免。

---

### Q29. `noUncheckedIndexedAccess` 有什么作用？为什么推荐开启？

- **难度**: mid
- **标签**: 严格检查

**答案**

默认情况下，通过索引签名或数组下标读取的结果**不包含 undefined**：

```ts
const arr: number[] = []
const x = arr[0]                // number，但运行时是 undefined
const map: Record<string, User> = {}
map['nope'].name                // 编译通过，运行时报错
```

开启后：

```ts
const x = arr[0]                // number | undefined
for (const v of arr) {}         // v 仍为 number（遍历不受影响）
const [a] = arr                 // number | undefined
```

代价：已有代码会出现大量需要处理的地方。可以先在新模块中试点，再逐步推广。

**口述模板（30–60 秒）**

1. 一句话结论：让下标访问的结果带上 undefined，避免越界和不存在的键。
2. 关键机制：只影响索引访问，不影响 for...of 和已知键。
3. 注意点：存量项目开启的修改量大。
4. 可追问时补充：strict 不包含它，需要单独开启。

**常见追问**

- 确定下标存在时怎么写？→ 先判断长度或用 `!`；更推荐用 `at()` 加显式判断。

---

### Q30. 如何把 JS 项目渐进迁移到 TS？

- **难度**: mid
- **标签**: 迁移

**答案**

1. **准备**：加 tsconfig，`allowJs: true`、`checkJs: false`、`strict: false`；构建工具支持 TS
2. **先加检查后改代码**：对重点 JS 文件加 `// @ts-check`，或用 JSDoc 补类型
3. **从叶子模块开始改扩展名**：工具函数、常量、API 层 → 组件 → 页面
4. **先写边界类型**：API 响应、全局状态、公共组件 Props，收益最大
5. **逐步开启严格选项**：`noImplicitAny` → `strictNullChecks` → `strict`
6. **防止回退**：CI 中跑 `tsc --noEmit`；lint 禁止新增 any；新文件必须是 TS
7. **工具辅助**：`ts-migrate` 等批量迁移工具可以先自动改名并补 any，再逐步替换

**口述模板（30–60 秒）**

1. 一句话结论：allowJs 起步，从叶子模块和边界类型开始，逐步收紧 strict。
2. 关键机制：新旧代码共存，CI 保证不回退。
3. 注意点：不要一次性全量开启 strict，否则报错过多会让团队放弃。
4. 可追问时补充：用覆盖率和报错数量作为迁移进度指标。

**常见追问**

- 迁移期间怎么衡量进度？→ TS 文件占比、`type-coverage` 百分比、`@ts-expect-error` 数量、strict 子项开启情况。

---

### Q31. `Omit` 作用于联合类型时有什么坑？

- **难度**: mid
- **标签**: 工具类型, 联合

**答案**

`Omit` **不是分布式的**，作用于联合类型时会先取 `keyof` 联合（只保留**共有键**），导致可辨识联合被破坏：

```ts
type Shape =
  | { kind: 'circle'; r: number; id: string }
  | { kind: 'rect'; w: number; id: string }

type NoId = Omit<Shape, 'id'>
// 结果：{ kind: 'circle' | 'rect' }，r 和 w 都丢了

type DistributiveOmit<T, K extends PropertyKey> = T extends unknown ? Omit<T, K> : never
type NoId2 = DistributiveOmit<Shape, 'id'>
// { kind: 'circle'; r: number } | { kind: 'rect'; w: number }
```

原因：`keyof (A | B)` 等于 `keyof A & keyof B`，即只有共有键。

`Pick`、直接对联合取 `keyof`、以及非同态的映射类型都有同样的问题。

**口述模板（30–60 秒）**

1. 一句话结论：Omit 不分发，作用于联合会只剩共有键。
2. 关键机制：keyof 联合等于共有键。
3. 注意点：会破坏可辨识联合。
4. 可追问时补充：用 `T extends unknown ? Omit<T, K> : never` 包一层分发。

**常见追问**

- 为什么 `Partial<A | B>` 没问题？→ 同态映射类型作用于联合时会自动分发。

---

### Q32. `interface` 继承和交叉类型 `&` 在冲突时有什么区别？

- **难度**: mid
- **标签**: 类型组合

**答案**

```ts
interface A { x: string }
interface B extends A { x: number }        // 报错：属性 x 类型不兼容

type C = { x: string } & { x: number }     // 不报错，x 为 never，C 实际上无法构造出任何值

type D = { fn(a: string): void } & { fn(a: number): void }
// fn 变成重载：(a: string) => void & (a: number) => void
```

- `extends` 要求子接口属性是父接口属性的**子类型**，冲突立即报错
- `&` 对同名属性做交叉：原始类型冲突变成 never，函数类型变成重载
- 想要「覆盖」而不是合并：`Omit<A, 'x'> & { x: number }`

性能上，`extends` 的关系可以被缓存；大量交叉类型会在每次比较时重新计算。

**口述模板（30–60 秒）**

1. 一句话结论：extends 冲突直接报错，交叉冲突静默变成 never 或重载。
2. 关键机制：交叉是对每个属性再做交叉。
3. 注意点：交叉产生的 never 不易发现。
4. 可追问时补充：覆盖属性用 Omit 再交叉；大型项目优先 extends。

**常见追问**

- 怎么写一个通用的覆盖工具类型？→ `type Override<T, U> = Omit<T, keyof U> & U`。
