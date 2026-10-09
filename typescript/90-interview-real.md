# TypeScript 真实面试题汇总

> 根据近年公开面经、社区高频题（含 type-challenges 变体）和国内前端 / 全栈岗位面试趋势整理。`来源倾向` 表示题目常见的出现场景，不代表某公司的固定题库。题干尽量保留面试中的真实问法。

---

### R1. `any`、`unknown`、`never` 有什么区别？

- **难度**: junior
- **标签**: 特殊类型
- **来源倾向**: 一面必问

**答案**

any 关闭检查且会传染；unknown 能接收任何值，但使用前必须收窄；never 是空集，用于永不返回的函数、穷尽检查和过滤联合。unknown 是顶层类型，never 是底层类型。

详见：[10-junior-qa Q3](10-junior-qa.md)

---

### R2. `type` 和 `interface` 的区别？你们项目里怎么选？

- **难度**: junior
- **标签**: 类型声明
- **来源倾向**: 一面必问，常追问「声明合并的实际用途」

**答案**

interface 只描述对象、支持声明合并、用 extends 继承且冲突时报错；type 能表达联合、元组、条件类型等任意类型。对象结构和可扩展的公开类型用 interface，类型运算用 type。大型项目中 interface 继承的检查性能更好。

详见：[10-junior-qa Q4](10-junior-qa.md)、[20-mid-qa Q32](20-mid-qa.md)

---

### R3. 手写 `Pick` 和 `Omit`

- **难度**: junior
- **标签**: 手写工具类型
- **来源倾向**: 中高级前端手写高频

**答案**

```ts
type MyPick<T, K extends keyof T> = { [P in K]: T[P] }
type MyOmit<T, K extends PropertyKey> = { [P in keyof T as P extends K ? never : P]: T[P] }
```

追问「内置 Omit 有什么问题」：K 不受 `keyof T` 约束，拼错不报错；作用于联合时不分发，会丢失非共有的键。

详见：[40-coding-handwritten H1](40-coding-handwritten.md)、[20-mid-qa Q31](20-mid-qa.md)

---

### R4. 手写 `ReturnType`，讲讲 `infer`

- **难度**: mid
- **标签**: infer
- **来源倾向**: 高频手写

**答案**

```ts
type MyReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : never
```

infer 在条件类型的 extends 子句中声明待推断的类型变量，相当于模式匹配。函数有重载时取最后一个签名。

详见：[20-mid-qa Q6](20-mid-qa.md)

---

### R5. 什么是分布式条件类型？下面的结果是什么？

- **难度**: mid
- **标签**: 条件类型, 输出题
- **来源倾向**: 原理深挖

**答案**

```ts
type A<T> = T extends string ? 'y' : 'n'
type R1 = A<string | number>     // 'y' | 'n'   分发
type R2 = A<never>               // never       空联合
type B<T> = [T] extends [string] ? 'y' : 'n'
type R3 = B<string | number>     // 'n'         不分发
type R4 = A<any>                 // 'y' | 'n'   any 同时走两个分支
```

裸类型参数遇到联合时逐个计算再合并；用元组包裹阻止分发。

详见：[20-mid-qa Q5](20-mid-qa.md)

---

### R6. 协变和逆变是什么？为什么函数参数是逆变的？

- **难度**: mid
- **标签**: 型变
- **来源倾向**: 中高级高频，常追问「方法和函数属性的区别」

**答案**

返回值协变、参数逆变：能处理 Animal 的函数一定能处理 Dog，所以 `(a: Animal) => void` 是 `(d: Dog) => void` 的子类型。`strictFunctionTypes` 只对函数属性语法生效，方法语法仍是双变，这是为了让数组协变可用而保留的不安全点。

详见：[20-mid-qa Q2](20-mid-qa.md)

---

### R7. 实现 `UnionToIntersection`，并解释原理

- **难度**: senior
- **标签**: infer, 逆变
- **来源倾向**: 高级岗类型体操

**答案**

```ts
type UnionToIntersection<U> =
  (U extends unknown ? (x: U) => void : never) extends (x: infer I) => void ? I : never
```

先分发，把每个成员放到函数参数（逆变位置）；同一个 infer 变量在逆变位置有多个候选时推断为交叉。

详见：[40-coding-handwritten H4](40-coding-handwritten.md)

---

### R8. `satisfies` 和类型注解、`as` 有什么区别？

- **难度**: junior
- **标签**: satisfies
- **来源倾向**: 近两年新增高频

**答案**

注解会把变量类型拓宽为声明的类型；`as` 不做真正的检查；`satisfies` 检查是否符合，但保留表达式的精确推断。常写作 `as const satisfies Config`。

详见：[10-junior-qa Q26](10-junior-qa.md)

---

### R9. 枚举编译后是什么？现在还推荐用吗？

- **难度**: junior
- **标签**: enum
- **来源倾向**: 一面常见，近年追问「和 Node 类型擦除的关系」

**答案**

数字枚举编译为双向映射对象，字符串枚举单向映射，const enum 内联。enum 会生成运行时代码，与 isolatedModules、`erasableSyntaxOnly`、Node 原生类型擦除不兼容，推荐 `as const` 对象加联合类型。

详见：[10-junior-qa Q13](10-junior-qa.md)

---

### R10. TypeScript 有哪些类型收窄方式？类型守卫怎么写？

- **难度**: junior
- **标签**: 收窄
- **来源倾向**: 一面高频

**答案**

typeof、instanceof、in、判等、真值、可辨识联合、类型谓词 `x is T`、断言函数 `asserts x is T`。类型谓词的正确性由开发者保证。TS 5.5 起简单的过滤函数能自动推断谓词。

详见：[10-junior-qa Q10、Q11](10-junior-qa.md)

---

### R11. 怎么用 `never` 做穷尽检查？

- **难度**: mid
- **标签**: never, 可辨识联合
- **来源倾向**: 中级高频

**答案**

在 switch 的 default 分支把值赋给 `never` 类型的参数。新增联合成员但没有处理时，该值不再是 never，编译报错。

```ts
default: { const _exhaustive: never = action; throw new Error('unreachable') }
```

详见：[20-mid-qa Q3、Q4](20-mid-qa.md)

---

### R12. 手写 `DeepReadonly` / `DeepPartial`，函数和数组怎么处理？

- **难度**: mid
- **标签**: 递归类型
- **来源倾向**: 高频手写

**答案**

递归映射类型，终止条件为原始类型、函数、Date 等内置对象；同态映射会自动保留数组和元组结构；Map / Set 单独处理。

详见：[40-coding-handwritten H3](40-coding-handwritten.md)

---

### R13. 下面这段代码为什么没报错？（多余属性检查）

- **难度**: junior
- **标签**: 类型兼容
- **来源倾向**: 一面「找问题」题

**答案**

```ts
interface Opts { width: number }
const tmp = { width: 1, heigth: 2 }
const o: Opts = tmp          // 不报错
```

结构化类型允许多余属性；多余属性检查只对直接赋值的「新鲜」对象字面量生效，经过变量中转后不再检查。

详见：[10-junior-qa Q29](10-junior-qa.md)

---

### R14. 泛型是什么？`K extends keyof T` 怎么用？

- **难度**: junior
- **标签**: 泛型
- **来源倾向**: 一面必问

**答案**

泛型是类型参数，用来在复用代码的同时保留输入与输出之间的类型关系。`function get<T, K extends keyof T>(o: T, k: K): T[K]` 是最经典的组合。

详见：[10-junior-qa Q14](10-junior-qa.md)

---

### R15. 实现一个类型安全的 EventEmitter

- **难度**: mid
- **标签**: 泛型类
- **来源倾向**: 中高级「运行时 + 类型」手写

**答案**

事件映射定义为「事件名 → 参数元组」，方法用 `E extends keyof Events` 关联事件名与参数，`emit(event, ...args: Events[E])`。实现中要处理 once 的自我移除，以及遍历期间修改监听器集合的问题。

详见：[40-coding-handwritten H10](40-coding-handwritten.md)

---

### R16. tsconfig 中 `moduleResolution` 怎么配？为什么 ESM 中要写 `.js` 后缀？

- **难度**: mid
- **标签**: 模块解析
- **来源倾向**: 工程化追问

**答案**

打包器项目用 `bundler`，Node 运行和发布的库用 `nodenext`。TS 不改写导入路径，Node ESM 要求完整的文件路径，所以源码中写编译后的 `.js`。TS 6.0 起 `node`（node10）和 `classic` 已废弃。

详见：[20-mid-qa Q17](20-mid-qa.md)

---

### R17. 怎么给没有类型的第三方库写声明？怎么给 `Window` 加属性？

- **难度**: mid
- **标签**: 声明文件
- **来源倾向**: 工程实践

**答案**

先找 `@types`，再按使用到的 API 手写 `declare module`。全局扩充需要在模块文件中写 `declare global { interface Window { ... } }`。

详见：[10-junior-qa Q21、Q30](10-junior-qa.md)、[20-mid-qa Q21](20-mid-qa.md)

---

### R18. 编译时类型都被擦除了，怎么保证接口返回数据的类型安全？

- **难度**: mid
- **标签**: 运行时校验
- **来源倾向**: 中高级高频，常延伸到端到端类型安全

**答案**

在信任边界做运行时校验（Zod 等），并用 `z.infer` 从 schema 推导类型，保证单一数据源。全栈 TS 可用 tRPC，跨语言后端用 OpenAPI 生成类型和校验器。

详见：[10-junior-qa Q23](10-junior-qa.md)、[30-senior-qa Q15](30-senior-qa.md)

---

### R19. 项目 `tsc` 很慢 / 编辑器很卡，你怎么排查？

- **难度**: senior
- **标签**: 性能
- **来源倾向**: 高级岗场景题

**答案**

`--extendedDiagnostics` 看实例化数量和检查耗时，`--generateTrace` 找热点类型；常见原因有 include 过大、交叉类型、巨大联合、深递归、复杂泛型组件；结构上用项目引用和增量编译，升级 TS 7 可获得 8–12 倍提速。

详见：[30-senior-qa Q12](30-senior-qa.md)、[50-system-design S8](50-system-design.md)

---

### R20. TypeScript 7 有什么变化？对你们项目有什么影响？

- **难度**: senior
- **标签**: TS 7
- **来源倾向**: 2026 年新增热点

**答案**

TS 7.0（2026-07）是 Go 原生移植版，类型检查结果与 6.0 一致，全量构建快 8–12 倍，支持并行检查。6.0 的废弃项变为硬错误；7.0 暂无编译器 API，依赖 API 的工具（类型感知 lint、Vue / Svelte 语言工具）暂时留在 6.0，可通过 `@typescript/typescript6` 并存。

**口述模板（30–60 秒）**

1. 一句话结论：TS 7 是 Go 原生移植版，检查结果与 6.0 一致，速度快 8–12 倍。
2. 关键机制：原生代码加共享内存并行，`--checkers` 控制检查器数量。
3. 注意点：没有编译器 API，部分生态要等 7.1。
4. 可追问时补充：我们先在 CI 切 7.0，编辑器和 lint 按生态进度切换。

详见：[30-senior-qa Q2、Q3、Q4](30-senior-qa.md)

---

### R21. 如何把一个大型 JS 项目迁移到 TS？

- **难度**: senior
- **标签**: 迁移
- **来源倾向**: 高级岗项目深挖

**答案**

allowJs 起步，CI 加入类型检查，新代码必须用 TS；先做接口类型、全局状态、公共组件等边界；按依赖图自底向上迁移；逐个开启严格选项；用类型覆盖率和棘轮策略防止回退。

详见：[20-mid-qa Q30](20-mid-qa.md)、[50-system-design S4](50-system-design.md)

---

### R22. 实现路由参数解析类型：`'/user/:id/post/:pid'` → `{ id: string; pid: string }`

- **难度**: mid
- **标签**: 模板字面量
- **来源倾向**: 中高级类型体操

**答案**

用模板字面量匹配 `${string}:${infer P}/${infer Rest}` 并递归，最后一段单独匹配，再用映射类型把参数名联合转成对象。

详见：[40-coding-handwritten H9](40-coding-handwritten.md)

---

### R23. 如何在 TS 中模拟名义类型，防止 `UserId` 和 `OrderId` 混用？

- **难度**: mid
- **标签**: 品牌类型
- **来源倾向**: 中高级建模题

**答案**

使用品牌类型：`type UserId = string & { readonly __brand: 'UserId' }`，只在解析或构造函数中产出该类型。延伸到「解析而不是校验」：把已校验的事实记录在类型中。

详见：[20-mid-qa Q1](20-mid-qa.md)、[30-senior-qa Q10](30-senior-qa.md)

---

### R24. React 中泛型组件怎么写？`useState(null)` 为什么后面赋值报错？

- **难度**: mid
- **标签**: React
- **来源倾向**: 前端岗高频

**答案**

泛型组件：`function List<T>(props: ListProps<T>)`，从 items 推断 T。`useState(null)` 把状态类型推断为 `null`，需要显式写 `useState<User | null>(null)`。React 19 中 ref 可以作为普通 prop，泛型组件不再受 forwardRef 丢失泛型的困扰。

详见：[10-junior-qa Q27](10-junior-qa.md)、[20-mid-qa Q24](20-mid-qa.md)

---

### R25. TS 类型系统是可靠（sound）的吗？举几个类型检查通过但运行时出错的例子

- **难度**: senior
- **标签**: 类型安全
- **来源倾向**: 高级岗原理深挖

**答案**

不是，TS 刻意在可用性和可靠性之间取舍。例子：数组协变后 push 了不同的子类型；方法参数双变；未开启 `noUncheckedIndexedAccess` 时越界访问；函数调用后属性收窄仍保留；`as` 断言和错误的类型谓词；`JSON.parse` 返回 any。对策是严格选项、lint 和边界校验。

**口述模板（30–60 秒）**

1. 一句话结论：TS 不追求完全可靠，有意保留了一些不安全点。
2. 关键机制：为了兼容 JS 习惯（数组、方法、索引访问）做了取舍。
3. 注意点：类型检查通过不代表运行时安全。
4. 可追问时补充：严格选项、no-unsafe 系列 lint、边界运行时校验三者配合。

详见：[30-senior-qa Q8](30-senior-qa.md)
