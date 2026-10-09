# TypeScript 手写与现场编码

## 使用方式

国内中高级前端面试中，TS 手写题主要有两类：

1. **类型体操**：手写工具类型（来自 type-challenges 的变体最常见）
2. **运行时 + 类型**：写一个函数或类，并让它的类型推断「好用」

先自己写、边写边讲思路，再对照参考实现。可以在 [TS Playground](https://www.typescriptlang.org/play) 中验证。

下文多处使用的基础工具：

```ts
type Equal<X, Y> =
  (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false
type Expect<T extends true> = T
type Primitive = string | number | boolean | bigint | symbol | null | undefined
```

---

### H1. 实现 Partial / Required / Readonly / Pick / Omit / Record

- **难度**: junior
- **标签**: 映射类型
- **面试官看什么**: 映射类型语法、修饰符、约束是否写对

**题干**

不使用内置工具类型，实现 `MyPartial`、`MyRequired`、`MyReadonly`、`MyPick`、`MyOmit`、`MyRecord`，并额外实现严格版 `StrictOmit`（K 必须是 T 的键）和 `Mutable`。

**思路口述**

1. 对象类工具都是映射类型：遍历键，修改修饰符或值类型
2. `Pick` 的 K 要约束为 `keyof T`
3. `Omit` 先用 `Exclude` 求剩余的键，再 `Pick`

**参考实现**

```ts
type MyPartial<T> = { [K in keyof T]?: T[K] }
type MyRequired<T> = { [K in keyof T]-?: T[K] }
type MyReadonly<T> = { readonly [K in keyof T]: T[K] }
type Mutable<T> = { -readonly [K in keyof T]: T[K] }

type MyPick<T, K extends keyof T> = { [P in K]: T[P] }
type MyRecord<K extends PropertyKey, V> = { [P in K]: V }

type MyOmit<T, K extends PropertyKey> = MyPick<T, Exclude<keyof T, K>>
type StrictOmit<T, K extends keyof T> = MyPick<T, Exclude<keyof T, K>>

// 不借助 Pick 的写法：用 key remapping 过滤
type MyOmit2<T, K extends PropertyKey> = { [P in keyof T as P extends K ? never : P]: T[P] }

interface User { readonly id: number; name?: string; age: number }
type _t1 = Expect<Equal<MyOmit<User, 'age'>, { readonly id: number; name?: string }>>
```

**易错点**

- `Pick` 忘记约束 `K extends keyof T`，导致 `T[P]` 报错
- `-?` 写成 `?-`
- 认为 `Omit` 会报告不存在的键（内置版本不会，所以需要 `StrictOmit`）

---

### H2. 实现 Exclude / Extract / NonNullable / ReturnType / Parameters / Awaited

- **难度**: junior
- **标签**: 条件类型, infer
- **面试官看什么**: 分布式条件类型、infer、递归

**题干**

实现上述工具类型，以及 `InstanceType` 和 `ConstructorParameters`。

**思路口述**

1. 联合过滤类用分布式条件类型：返回 never 就是删除
2. 函数相关用 infer 捕获参数或返回值
3. `Awaited` 需要递归，并且要匹配 thenable，不只是 Promise

**参考实现**

```ts
type MyExclude<T, U> = T extends U ? never : T
type MyExtract<T, U> = T extends U ? T : never
type MyNonNullable<T> = T & {}

type MyReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : never
type MyParameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never

type MyInstanceType<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: any) => infer R ? R : never
type MyConstructorParameters<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: infer P) => any ? P : never

type MyAwaited<T> =
  T extends null | undefined ? T
  : T extends object & { then(onfulfilled: infer F, ...args: any[]): any }
    ? F extends (value: infer V, ...args: any[]) => any ? MyAwaited<V> : never
    : T

type _t1 = Expect<Equal<MyAwaited<Promise<Promise<number>>>, number>>
type _t2 = Expect<Equal<MyExclude<'a' | 'b' | 'c', 'a' | 'b'>, 'c'>>
```

**易错点**

- 约束写成 `(...args: any[]) => any` 也可以，但 `any` 比 `any[]` 更宽，能接受 `this` 参数等特殊签名
- `InstanceType` 加 `abstract` 才能接收抽象类
- `Awaited` 不递归时，嵌套 Promise 解不开

---

### H3. 实现 DeepPartial / DeepReadonly

- **难度**: mid
- **标签**: 递归类型
- **面试官看什么**: 递归终止条件、函数和数组的处理、对内置对象的考虑

**题干**

实现深层可选和深层只读。要求：函数类型保持不变；数组和元组正确处理；`Date`、`Map` 等内置对象不被拆开。

**思路口述**

1. 终止条件：原始类型、函数、`Date`、`RegExp` 原样返回
2. 对象和数组用**同态映射类型**递归：类型参数为数组时，映射结果仍是数组或元组
3. `Map` / `Set` 单独处理，递归它们的类型参数

**参考实现**

```ts
type Builtin = Primitive | Date | RegExp | Error | ((...args: any[]) => unknown)

type DeepPartial<T> =
  T extends Builtin ? T
  : T extends Map<infer K, infer V> ? Map<DeepPartial<K>, DeepPartial<V>>
  : T extends Set<infer E> ? Set<DeepPartial<E>>
  : { [K in keyof T]?: DeepPartial<T[K]> }

type DeepReadonly<T> =
  T extends Builtin ? T
  : T extends Map<infer K, infer V> ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>>
  : T extends Set<infer E> ? ReadonlySet<DeepReadonly<E>>
  : { readonly [K in keyof T]: DeepReadonly<T[K]> }

interface Conf {
  db: { host: string; ports: number[] }
  onError: (e: Error) => void
  createdAt: Date
}

type P = DeepPartial<Conf>
// { db?: { host?: string; ports?: (number | undefined)[] }; onError?: ...; createdAt?: Date }

type R = DeepReadonly<Conf>
// R['db']['ports'] 是 readonly number[]
```

**易错点**

- 不处理函数：函数会被映射成 `{}`，失去可调用性
- 不处理 `Date`：会把 `getTime` 等方法变成可选
- 同态映射作用于数组时，`DeepPartial` 会让数组**元素**变为可选（`(T | undefined)[]`）。如果只想让数组本身可选、元素不变，需要单独匹配 `readonly (infer E)[]`

---

### H4. 实现 UnionToIntersection、LastOfUnion、UnionToTuple

- **难度**: senior
- **标签**: infer, 逆变, 高频
- **面试官看什么**: 是否理解「逆变位置推断得到交叉」以及函数重载推断的规则

**题干**

1. `UnionToIntersection<A | B>` 得到 `A & B`
2. `LastOfUnion<'a' | 'b'>` 取联合的「最后一个」成员
3. `UnionToTuple<'a' | 'b'>` 得到 `['a', 'b']`

**思路口述**

1. 先用分布式条件类型把每个成员放到**函数参数位置**：`(x: A) => void | (x: B) => void`
2. 再用一个 infer 去匹配参数：同一个 infer 变量的多个候选处于逆变位置，推断结果为**交叉**
3. 函数类型的交叉相当于重载；对重载做 infer 时，取**最后一个签名**，由此得到联合的「最后一个」成员
4. 反复取出最后一个并排除，递归得到元组

**参考实现**

```ts
type UnionToIntersection<U> =
  (U extends unknown ? (x: U) => void : never) extends (x: infer I) => void ? I : never

type LastOfUnion<U> =
  UnionToIntersection<U extends unknown ? () => U : never> extends () => infer R ? R : never

type UnionToTuple<U, Last = LastOfUnion<U>> =
  [U] extends [never] ? [] : [...UnionToTuple<Exclude<U, Last>>, Last]

type _t1 = Expect<Equal<UnionToIntersection<{ a: 1 } | { b: 2 }>, { a: 1 } & { b: 2 }>>
type T2 = UnionToTuple<'a' | 'b' | 'c'>   // 通常为 ['a', 'b', 'c']
```

**易错点**

- 第一步必须是分布式的（裸类型参数），否则只得到一个函数类型
- `UnionToTuple` 的顺序取决于编译器内部的联合成员排序，**不应在业务中依赖**；TS 7 改为按内容稳定排序，结果可能与旧版本不同
- `UnionToIntersection<boolean>` 得到 `never`（`true & false`）

---

### H5. 实现 IsAny、IsNever、IsUnion、Equal

- **难度**: mid
- **标签**: 条件类型, 特殊类型
- **面试官看什么**: 对 any、never、分发行为的边界理解

**题干**

实现四个判断类型，返回 `true` 或 `false`。

**思路口述**

1. `IsAny`：any 和任何类型的交叉仍然是 any，而 `1 & T` 对其他 T 都不会是 0 的父类型
2. `IsNever`：必须用元组包裹，阻止 never 被分发成 never
3. `IsUnion`：分发时每个成员都和「完整联合」比较；如果完整联合不能赋给单个成员，说明有多个成员
4. `Equal`：利用延迟条件类型的同一性比较，能区分 any 和 readonly

**参考实现**

```ts
type IsAny<T> = 0 extends 1 & T ? true : false
type IsNever<T> = [T] extends [never] ? true : false
type IsUnion<T, U = T> =
  [T] extends [never] ? false
  : T extends unknown ? ([U] extends [T] ? false : true)
  : never

type Equal<X, Y> =
  (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false

type _t = [
  Expect<Equal<IsAny<any>, true>>,
  Expect<Equal<IsAny<unknown>, false>>,
  Expect<Equal<IsNever<never>, true>>,
  Expect<Equal<IsUnion<string | number>, true>>,
  Expect<Equal<IsUnion<string>, false>>,
  Expect<Equal<Equal<{ a: 1 }, { readonly a: 1 }>, false>>,
]
```

**易错点**

- `IsNever<T> = T extends never ? true : false` 对 never 返回 never
- 用双向 `extends` 判断相等时，`any` 与任何类型都「相等」
- `IsUnion<boolean>` 为 `true`，因为 boolean 是 `true | false`

---

### H6. 元组操作：First / Last / Length / Concat / Includes / Reverse / Flatten / Add

- **难度**: mid
- **标签**: 元组, 递归
- **面试官看什么**: 可变元组、infer 的位置、尾递归、用元组长度计数

**题干**

实现常见的元组工具类型；附加题：实现类型层面的加法 `Add<2, 3>` 得到 `5`。

**思路口述**

1. 用 `[infer H, ...infer R]` 拆分首尾
2. `Includes` 必须用严格的 `Equal`，否则 `Includes<[boolean], false>` 会误判
3. `Reverse` 使用累加器实现尾递归
4. 加法：构造对应长度的元组，拼接后取 `length`

**参考实现**

```ts
type First<T extends readonly unknown[]> = T extends readonly [infer F, ...unknown[]] ? F : never
type Last<T extends readonly unknown[]> = T extends readonly [...unknown[], infer L] ? L : never
type Length<T extends readonly unknown[]> = T['length']
type Concat<A extends readonly unknown[], B extends readonly unknown[]> = [...A, ...B]

type Includes<T extends readonly unknown[], U> =
  T extends readonly [infer F, ...infer R]
    ? Equal<F, U> extends true ? true : Includes<R, U>
    : false

type Reverse<T extends readonly unknown[], Acc extends unknown[] = []> =
  T extends readonly [infer H, ...infer R] ? Reverse<R, [H, ...Acc]> : Acc

type Flatten<T extends readonly unknown[]> =
  T extends readonly [infer H, ...infer R]
    ? H extends readonly unknown[] ? [...Flatten<H>, ...Flatten<R>] : [H, ...Flatten<R>]
    : []

type BuildTuple<N extends number, Acc extends unknown[] = []> =
  Acc['length'] extends N ? Acc : BuildTuple<N, [...Acc, unknown]>
type Add<A extends number, B extends number> = [...BuildTuple<A>, ...BuildTuple<B>]['length']

type _t = [
  Expect<Equal<Reverse<[1, 2, 3]>, [3, 2, 1]>>,
  Expect<Equal<Flatten<[1, [2, [3]]]>, [1, 2, 3]>>,
  Expect<Equal<Includes<[boolean], false>, false>>,
  Expect<Equal<Add<2, 3>, 5>>,
]
```

**易错点**

- 约束写成 `unknown[]` 时，`as const` 产生的只读元组传不进来
- `First<[]>` 应该返回 never，不要写成 `T[0]`（会得到 undefined）
- `BuildTuple` 只适用于非负整数且受递归深度限制（约 1000）

---

### H7. 字符串操作：Trim / Replace / Split / CamelCase / KebabCase

- **难度**: mid
- **标签**: 模板字面量
- **面试官看什么**: 模板字面量的模式匹配、递归、边界情况

**题干**

实现字符串类型的去空白、替换、分割和命名风格转换。

**思路口述**

1. 用 `` `${infer L}${X}${infer R}` `` 做模式匹配
2. `Replace` 要处理 `From` 为空字符串的情况，否则会死循环或结果错误
3. `Split` 要处理 `string` 这种非字面量输入，以及分隔符为空
4. `KebabCase` 判断下一个字符是否为大写：`T extends Uncapitalize<T>`

**参考实现**

```ts
type Space = ' ' | '\n' | '\t'
type TrimLeft<S extends string> = S extends `${Space}${infer R}` ? TrimLeft<R> : S
type TrimRight<S extends string> = S extends `${infer L}${Space}` ? TrimRight<L> : S
type Trim<S extends string> = TrimLeft<TrimRight<S>>

type Replace<S extends string, From extends string, To extends string> =
  From extends '' ? S
  : S extends `${infer L}${From}${infer R}` ? `${L}${To}${R}` : S

type ReplaceAll<S extends string, From extends string, To extends string> =
  From extends '' ? S
  : S extends `${infer L}${From}${infer R}` ? `${L}${To}${ReplaceAll<R, From, To>}` : S

type Split<S extends string, D extends string> =
  string extends S ? string[]
  : S extends `${infer H}${D}${infer T}` ? [H, ...Split<T, D>]
  : D extends '' ? [] : [S]

type CamelCase<S extends string> =
  S extends `${infer H}_${infer T}` ? `${H}${CamelCase<Capitalize<T>>}` : S

type KebabCase<S extends string> =
  S extends `${infer H}${infer T}`
    ? T extends Uncapitalize<T>
      ? `${Uncapitalize<H>}${KebabCase<T>}`
      : `${Uncapitalize<H>}-${KebabCase<T>}`
    : S

type _t = [
  Expect<Equal<Trim<'  hi \n'>, 'hi'>>,
  Expect<Equal<ReplaceAll<'a-b-c', '-', '_'>, 'a_b_c'>>,
  Expect<Equal<Split<'a,b,c', ','>, ['a', 'b', 'c']>>,
  Expect<Equal<Split<'abc', ''>, ['a', 'b', 'c']>>,
  Expect<Equal<CamelCase<'foo_bar_baz'>, 'fooBarBaz'>>,
  Expect<Equal<KebabCase<'FooBarBaz'>, 'foo-bar-baz'>>,
]
```

**易错点**

- `ReplaceAll` 递归时对 `${L}${To}` 再次替换，会在 To 包含 From 时出错，只应递归 R
- `Split<'', ','>` 按 JS 语义应为 `['']`
- TS 7.0 起 `${infer H}${infer T}` 按 Unicode 码点拆分，emoji 相关的结果与旧版不同

---

### H8. 实现对象路径类型 Path 与类型安全的 get

- **难度**: senior
- **标签**: 递归, 模板字面量, 实战
- **面试官看什么**: 递归映射、模板拼接、路径取值类型，以及能否把类型落到实际函数上

**题干**

实现 `Path<T>` 得到所有嵌套路径（如 `'user' | 'user.name'`），`PathValue<T, P>` 得到路径对应的值类型，并实现 `get(obj, path)`，要求路径错误时报错、返回值类型正确。（表单库、i18n、配置读取的常见需求）

**思路口述**

1. 终止条件：原始类型、函数、数组、`Date` 视为叶子
2. 对每个键生成 `K | ${K}.${子路径}`，再用 `[keyof T]` 取出值的联合
3. `PathValue` 用模板字面量按第一个 `.` 拆分，递归取值
4. 运行时用 `split('.')` + `reduce`

**参考实现**

```ts
type Leaf = Primitive | Date | ((...args: any[]) => unknown) | readonly unknown[]

type Path<T> = T extends Leaf ? never : {
  [K in keyof T & string]: T[K] extends Leaf ? K : K | `${K}.${Path<T[K]>}`
}[keyof T & string]

type PathValue<T, P extends string> =
  P extends `${infer K}.${infer Rest}`
    ? K extends keyof T ? PathValue<T[K], Rest> : never
    : P extends keyof T ? T[P] : never

/** 按点分路径读取嵌套属性，路径和返回值类型均由对象结构推导 */
export function get<T, P extends Path<T>>(obj: T, path: P): PathValue<T, P> {
  return path.split('.').reduce<any>((cur, key) => cur?.[key], obj)
}

const state = { user: { name: 'a', tags: ['x'] }, theme: 'dark' as const }
const name = get(state, 'user.name')     // string
const theme = get(state, 'theme')        // 'dark'
// get(state, 'user.age')                // 报错
```

**易错点**

- 可选属性：`T[K]` 是 `X | undefined`，`keyof (X | undefined)` 为 never，`PathValue` 会失效；需要用 `NonNullable<T[K]>` 递归并在结果上补 `| undefined`
- 不把数组当作叶子时，会把 `length`、`push` 等方法也展开为路径
- 嵌套很深或键很多时会触发深度限制并拖慢编辑器，可以加深度计数器

---

### H9. 从路由字符串中解析参数类型

- **难度**: mid
- **标签**: 模板字面量, 实战
- **面试官看什么**: infer 在模板字面量中的匹配规则，以及与运行时函数结合

**题干**

实现 `buildPath('/user/:id/post/:postId', { id: '1', postId: '2' })`，要求参数对象的键由路由字符串推导，缺少或多写参数都报错。

**思路口述**

1. 匹配 `:参数名/剩余部分`，递归处理剩余部分
2. 最后一段没有 `/`，单独匹配
3. 用映射类型把参数名联合转换为对象类型
4. 用 `const` 类型参数保留路由字面量

**参考实现**

```ts
type ParamKeys<S extends string> =
  S extends `${string}:${infer P}/${infer Rest}` ? P | ParamKeys<`/${Rest}`>
  : S extends `${string}:${infer P}` ? P
  : never

type RouteParams<S extends string> = { [K in ParamKeys<S>]: string }

/** 用参数填充路由模板，参数键名由模板字面量推导 */
export function buildPath<const S extends string>(pattern: S, params: RouteParams<S>): string {
  const values = params as Record<string, string>
  return pattern.replace(/:(\w+)/g, (_, key: string) => encodeURIComponent(values[key]))
}

buildPath('/user/:id/post/:postId', { id: '1', postId: '2' })   // OK
// buildPath('/user/:id', {})                                    // 报错：缺少 id

type _t = Expect<Equal<ParamKeys<'/a/:x/b/:y'>, 'x' | 'y'>>
```

**易错点**

- `pattern` 是 `string` 类型变量（非字面量）时，推导结果为 `{}`，失去约束
- 可选参数（`:id?`）需要额外匹配 `?` 并映射为可选键
- 运行时一定要对参数编码，否则特殊字符会破坏 URL

---

### H10. 实现类型安全的 EventEmitter

- **难度**: mid
- **标签**: 泛型类, 实战, 高频
- **面试官看什么**: 事件名与参数的类型关联、取消订阅、once 的实现细节

**题干**

实现 `TypedEmitter<Events>`，`on` / `off` / `once` / `emit` 的事件名和参数都受约束。

**思路口述**

1. 事件映射用「事件名 → 参数元组」表示，便于用剩余参数传递多个值
2. 方法使用泛型 `E extends keyof Events` 关联事件名和参数
3. `on` 返回取消订阅函数，方便在组件卸载时清理
4. `emit` 时先复制监听器集合，避免监听器在执行中被移除或新增导致遍历异常

**参考实现**

```ts
type Listener<A extends unknown[]> = (...args: A) => void

/** 事件名与参数类型强关联的事件发射器 */
export class TypedEmitter<Events extends Record<keyof Events, unknown[]>> {
  #listeners = new Map<keyof Events, Set<Listener<any>>>()

  on<E extends keyof Events>(event: E, fn: Listener<Events[E]>): () => void {
    let set = this.#listeners.get(event)
    if (!set) {
      set = new Set()
      this.#listeners.set(event, set)
    }
    set.add(fn)
    return () => this.off(event, fn)
  }

  off<E extends keyof Events>(event: E, fn: Listener<Events[E]>): void {
    this.#listeners.get(event)?.delete(fn)
  }

  once<E extends keyof Events>(event: E, fn: Listener<Events[E]>): () => void {
    const off = this.on(event, (...args) => {
      off()
      fn(...args)
    })
    return off
  }

  emit<E extends keyof Events>(event: E, ...args: Events[E]): void {
    // 复制一份：once 会在回调中删除自身，监听器也可能在回调中注册新的监听器
    for (const fn of [...(this.#listeners.get(event) ?? [])]) fn(...args)
  }
}

interface AppEvents {
  login: [user: { id: string }]
  logout: []
  error: [err: Error, retryable: boolean]
}

const bus = new TypedEmitter<AppEvents>()
const off = bus.on('error', (err, retryable) => {})   // 参数自动推断
bus.emit('login', { id: '1' })
// bus.emit('logout', 1)                               // 报错
off()
```

**易错点**

- 约束写成 `Record<string, unknown[]>` 时，`interface` 定义的事件映射无法满足（interface 没有隐式索引签名），所以用 `Record<keyof Events, unknown[]>`
- `once` 返回的取消函数要能在触发前取消
- 内部存储用 `Listener<any>`，是因为不同事件的参数类型不同，对外 API 已保证类型安全

---

### H11. 实现类型安全的 pipe

- **难度**: mid
- **标签**: 重载, 函数组合
- **面试官看什么**: 是否知道变长泛型链的局限，以及业界的做法

**题干**

实现 `pipe(value, f1, f2, ...)`，每个函数的参数类型是上一个函数的返回类型，最终返回最后一个函数的返回类型。

**思路口述**

1. 「每一步的输入等于上一步的输出」这种链式关系，TS 无法用一个可变长泛型签名优雅地表达
2. 业界（RxJS、fp-ts、Effect）的做法是**重载**：写到常用的长度（如 9 或 12 个函数）
3. 实现签名用 unknown，依赖重载保证对外类型安全

**参考实现**

```ts
/** 把值依次传给一组函数，类型由各函数签名逐级推导 */
export function pipe<A>(a: A): A
export function pipe<A, B>(a: A, ab: (a: A) => B): B
export function pipe<A, B, C>(a: A, ab: (a: A) => B, bc: (b: B) => C): C
export function pipe<A, B, C, D>(a: A, ab: (a: A) => B, bc: (b: B) => C, cd: (c: C) => D): D
export function pipe<A, B, C, D, E>(
  a: A, ab: (a: A) => B, bc: (b: B) => C, cd: (c: C) => D, de: (d: D) => E,
): E
export function pipe(a: unknown, ...fns: Array<(x: unknown) => unknown>): unknown {
  return fns.reduce((acc, fn) => fn(acc), a)
}

const r = pipe(
  ' 42 ',
  s => s.trim(),          // s: string
  s => Number(s),         // s: string
  n => n * 2,             // n: number
)                         // r: number
```

**易错点**

- 把初始值放在第一个参数，回调参数才能从左到右推断；`compose(f, g)(x)` 风格的推断更困难
- 用递归条件类型校验函数链是可行的，但错误信息难读，推断回调参数的效果也不如重载

---

### H12. 实现带类型的 Promise.all

- **难度**: mid
- **标签**: 映射类型, 元组, 异步
- **面试官看什么**: 用映射类型保持元组结构、Awaited、元组推断技巧

**题干**

实现 `promiseAll(values)`，传入 `[Promise<number>, string]` 时返回 `Promise<[number, string]>`。

**思路口述**

1. 参数约束写成 `readonly unknown[] | []`，促使编译器把数组字面量推断为**元组**（官方声明使用同样的技巧）
2. 返回值用同态映射类型遍历元组，对每一项取 `Awaited`
3. 运行时：计数器记录剩余数量，按下标写入结果，任一失败立即 reject，空数组直接 resolve

**参考实现**

```ts
type AwaitedAll<T extends readonly unknown[]> = { -readonly [K in keyof T]: Awaited<T[K]> }

/** 与 Promise.all 行为一致，并保留元组中每个位置的类型 */
export function promiseAll<T extends readonly unknown[] | []>(values: T): Promise<AwaitedAll<T>> {
  return new Promise((resolve, reject) => {
    const results: unknown[] = new Array(values.length)
    // 运行时数组无法被编译器证明符合映射后的元组类型，只能在这一处断言
    const finish = () => resolve(results as unknown as AwaitedAll<T>)
    let remaining = values.length
    if (remaining === 0) {
      finish()
      return
    }
    values.forEach((v, i) => {
      // Promise.resolve 统一处理普通值和 thenable
      Promise.resolve(v).then(val => {
        results[i] = val
        if (--remaining === 0) finish()
      }, reject)
    })
  })
}

const res = await promiseAll([Promise.resolve(1), 'a', fetchUser()])
// res: [number, string, User]
```

**易错点**

- 约束只写 `readonly unknown[]` 时，数组字面量会被推断为 `(Promise<number> | string)[]`，丢失位置信息
- 用 `push` 收集结果会打乱顺序，必须按下标写入
- 不单独处理空数组时，计数器永远不会归零，Promise 一直处于 pending

---

### H13. 实现迷你 Zod：schema 定义 + 运行时校验 + 类型推导

- **难度**: senior
- **标签**: 运行时校验, 类型推导, 实战
- **面试官看什么**: 运行时值与类型的统一、映射类型推导对象结构、错误路径

**题干**

实现 `s.string()`、`s.number()`、`s.optional()`、`s.object()`、`s.array()`，以及 `Infer<typeof schema>`，`parse` 失败时抛出带字段路径的错误。

**思路口述**

1. 每个 schema 是一个带 `parse(input: unknown): T` 的对象，T 就是它代表的类型
2. `Infer` 用 infer 从 `parse` 的返回类型中取出 T
3. `object` 接收由 schema 组成的对象，用映射类型对每个键做 `Infer`
4. 错误时在 message 前拼接字段路径，方便定位

**参考实现**

```ts
/** 运行时校验器，同时携带其校验结果的静态类型 */
export interface Schema<T> {
  parse(input: unknown, path?: string): T
}

export type Infer<S> = S extends Schema<infer T> ? T : never

const fail = (path: string, msg: string): never => {
  throw new TypeError(`${path || '(root)'}: ${msg}`)
}

/** 迷你 schema 构造器集合 */
export const s = {
  string: (): Schema<string> => ({
    parse: (v, path = '') => (typeof v === 'string' ? v : fail(path, 'expected string')),
  }),

  number: (): Schema<number> => ({
    parse: (v, path = '') =>
      typeof v === 'number' && !Number.isNaN(v) ? v : fail(path, 'expected number'),
  }),

  optional: <T>(inner: Schema<T>): Schema<T | undefined> => ({
    parse: (v, path) => (v === undefined ? undefined : inner.parse(v, path)),
  }),

  array: <T>(item: Schema<T>): Schema<T[]> => ({
    parse: (v, path = '') =>
      Array.isArray(v) ? v.map((x, i) => item.parse(x, `${path}[${i}]`)) : fail(path, 'expected array'),
  }),

  object: <Shape extends Record<string, Schema<unknown>>>(
    shape: Shape,
  ): Schema<{ [K in keyof Shape]: Infer<Shape[K]> }> => ({
    parse(v, path = '') {
      if (typeof v !== 'object' || v === null || Array.isArray(v)) return fail(path, 'expected object')
      const input = v as Record<string, unknown>
      const out: Record<string, unknown> = {}
      for (const key in shape) {
        out[key] = shape[key].parse(input[key], path ? `${path}.${key}` : key)
      }
      return out as { [K in keyof Shape]: Infer<Shape[K]> }
    },
  }),
}

const User = s.object({
  id: s.number(),
  name: s.string(),
  tags: s.array(s.string()),
  nickname: s.optional(s.string()),
})
type User = Infer<typeof User>
// { id: number; name: string; tags: string[]; nickname: string | undefined }

User.parse({ id: 1, name: 'a', tags: [1] })   // TypeError: tags[0]: expected string
```

**易错点**

- 结果中 `nickname` 是「必有键，值可为 undefined」，而不是可选键。真实的 Zod 会把这类键再映射为 `?:`：先用 `as` 子句筛出值包含 undefined 的键，再与其他键交叉
- `object` 默认会丢弃未声明的键（相当于 Zod 的 strip 模式），这是防止脏数据传播的有意设计
- `Schema<unknown>` 能作为约束，是因为 `parse` 的返回类型处于协变位置

---

### H14. 实现可链式调用的配置类型 Chainable

- **难度**: senior
- **标签**: 泛型累积, type-challenges
- **面试官看什么**: 通过返回新泛型实例累积类型，以及参数位置的条件约束

**题干**

```ts
const result = config
  .option('foo', 123)
  .option('name', 'type-challenges')
  .option('bar', { value: 'Hello' })
  .get()
// result: { foo: number; name: string; bar: { value: string } }
```

要求重复设置同一个键时报错。

**思路口述**

1. `Chainable<T>` 的 `option` 返回 `Chainable<T & Record<K, V>>`，每次调用都把新键累积到类型参数中
2. 用 `K extends keyof T ? never : K` 作为参数类型，键已存在时参数类型为 never，从而报错
3. 先用 `Omit<T, K>` 再交叉，保证类型干净
4. 运行时用一个普通的构建器类实现，再断言为 `Chainable`

**参考实现**

```ts
type Chainable<T = {}> = {
  option<K extends string, V>(
    key: K extends keyof T ? never : K,
    value: V,
  ): Chainable<Omit<T, K> & Record<K, V>>
  get(): T
}

class ConfigBuilder {
  #data: Record<string, unknown> = {}
  option(key: string, value: unknown): this {
    this.#data[key] = value
    return this
  }
  get() {
    return { ...this.#data }
  }
}

/** 创建一个逐步累积键值类型的配置构建器 */
export const createConfig = (): Chainable => new ConfigBuilder() as unknown as Chainable

const result = createConfig().option('foo', 123).option('name', 'x').get()
// result 的类型展开后为 { foo: number; name: string }
// createConfig().option('a', 1).option('a', 2)   // 报错
```

**易错点**

- 返回 `this` 无法累积类型，必须返回新的 `Chainable<...>`
- 悬停看到的是交叉类型，可以用 `Expand` 工具类型展开（见高级题 Q24）
- 运行时实现与类型声明是分离的，需要单元测试保证一致
