# TypeScript 高级 Q&A（原理、架构与生产实践）

> 目标：能讲清编译器原理与 TS 7 的变化、推断与控制流分析的机制、类型系统的不安全点、库类型设计、类型检查性能、大型项目治理与升级。高级题回答要有「机制 + 取舍 + 落地经验」。

---

## 一、编译器原理与 TS 7

### Q1. TypeScript 编译器的整体架构是怎样的？

- **难度**: senior
- **标签**: 编译器, 原理

**答案**

```text
源码 ──Scanner──> Token ──Parser──> AST
                                     │
                               Binder（建立符号表 Symbol、作用域、控制流图 Flow Node）
                                     │
                               Checker（类型推断、赋值兼容检查、收窄 —— 最重的部分）
                                     │
                               Emitter（输出 .js / .d.ts / .map）

Program：管理所有源文件、模块解析结果、编译选项
Language Service：基于 Program 提供补全、跳转、重命名等编辑器能力（旧版 tsserver，TS 7 改为 LSP）
```

关键点：

- **Binder** 为每个声明创建 Symbol，并构建**控制流图**，供 Checker 做收窄分析
- **Checker 是懒计算的**：只在需要某个类型时才计算并缓存。编辑器中只检查当前打开的文件
- **转译和类型检查相对独立**：Emitter 大部分工作只是删除类型，这是 esbuild/SWC 能「只转译不检查」的基础
- **`.d.ts` 生成依赖 Checker**：需要推断未标注的返回类型，这也是 `isolatedDeclarations` 出现的原因

**口述模板（30–60 秒）**

1. 一句话结论：Scanner → Parser → Binder → Checker → Emitter，Checker 最重。
2. 关键机制：Binder 建符号表和控制流图，Checker 懒计算并缓存类型。
3. 注意点：转译几乎不依赖类型，声明文件生成却依赖类型推断。
4. 可追问时补充：TS 7 保持了同样的结构，用 Go 重写并实现并行。

**常见追问**

- 为什么编辑器里报错比 `tsc` 快？→ 语言服务只检查打开的文件及其依赖的必要部分，而 `tsc` 要检查全部文件。

---

### Q2. TS 7.0 为什么用 Go 重写？带来了哪些变化？

- **难度**: senior
- **标签**: TS 7, 高频

**答案**

背景：大型代码库（VS Code、大型 Monorepo）中 `tsc` 全量检查要几分钟，编辑器打开项目要十几秒。JS 实现受限于单线程和 V8 的内存模型。

为什么选 Go（官方说法概括）：

- 采用**逐行移植（port）而非重写（rewrite）**，Go 的编程模型与现有 TS 代码结构最接近（大量使用函数与数据结构，循环引用多），便于保持行为一致
- 有 GC，不需要像 Rust 那样重新设计所有权
- 原生代码 + **共享内存多线程**

结果（2026-07 发布，官方数据）：

- 全量构建快 **8–12 倍**（如 VS Code 125.7s → 10.6s），内存降低约 6%–26%
- 编辑器打开项目到出现首个错误：VS Code 仓库从约 17.5 秒降到约 1.3 秒
- 解析、检查、产出均可并行：`--checkers`（默认 4 个检查器）、`--builders`（项目引用并行构建）、`--singleThreaded`
- 语言服务改为 LSP 实现，官方数据显示失败命令和崩溃明显减少
- 6.0 中的废弃项全部变为硬错误；`stableTypeOrdering` 强制开启

限制：

- **7.0 没有编译器 API**（计划 7.1 提供新 API）。依赖 API 的工具（typescript-eslint 的类型感知规则、Vue / Svelte / Astro 的语言工具、Angular 模板检查）暂时需要 6.0，可通过 `@typescript/typescript6`（命令 `tsc6`）并存
- 不同 `--checkers` 数量在极少数情况下可能产生顺序相关的结果差异，团队应固定该值

**口述模板（30–60 秒）**

1. 一句话结论：TS 7 用 Go 逐行移植编译器，全量构建快 8–12 倍，类型检查结果与 6.0 一致。
2. 关键机制：原生代码 + 共享内存并行，检查器数量可调。
3. 注意点：7.0 没有编译器 API，Vue 等生态工具暂时留在 6.0。
4. 可追问时补充：我们的策略是 CLI 和 CI 先切 7.0，编辑器和 lint 按生态进度切换。

**常见追问**

- 为什么不选 Rust？→ 官方的考量是移植成本：Rust 的所有权模型需要重新设计大量循环引用的数据结构，不利于「行为完全一致」的目标。
- `tsgo` 是什么？→ 原生预览版 `@typescript/native-preview` 的命令名，7.0 起统一为 `tsc`。

---

### Q3. 并行类型检查为什么会有「类型顺序」问题？`stableTypeOrdering` 是什么？

- **难度**: senior
- **标签**: TS 7, 原理

**答案**

TS 内部给每个类型分配一个**类型 ID**，按遇到的先后顺序递增。联合类型的成员按 ID 排序，所以声明顺序会影响结果：

```ts
// 只有 foo 时，声明文件输出 100 | 500
// 在 foo 之前加一行 const x = 500，输出变成 500 | 100
export function foo(c: boolean) { return c ? 100 : 500 }
```

单线程时这个顺序是确定的；**并行检查**时，多个检查器访问类型的顺序不同，ID 不再确定，可能导致：

- 同样的代码生成不同的 `.d.ts`
- 极少数情况下，推断结果依赖顺序，导致错误时有时无

TS 7 的解决办法：按**类型内容**的确定性算法排序，与创建顺序无关。

TS 6.0 提供 `--stableTypeOrdering` 让 6.0 的行为与 7.0 对齐，用于**迁移对比**（会让检查变慢最多约 25%，不建议长期开启）。开启后出现的新错误，通常说明原来的推断「碰巧」依赖了顺序，应该补充显式的类型参数或类型标注。

**口述模板（30–60 秒）**

1. 一句话结论：类型 ID 按出现顺序分配，并行后顺序不确定，所以 TS 7 改为按内容排序。
2. 关键机制：联合成员和属性按 ID 排序，影响声明输出和少数推断。
3. 注意点：6.0 的 stableTypeOrdering 只用于迁移对比，有性能代价。
4. 可追问时补充：开启后出现的报错用显式泛型参数或变量标注修复。

**常见追问**

- 升级 7.0 前怎么验证？→ 在 6.0 上开启 `stableTypeOrdering`、去掉 `ignoreDeprecations`，确认无报错后再对比 6.0 与 7.0 的 `.d.ts` 输出。

---

### Q4. 大型项目如何从 TS 5.x 升级到 6.0 / 7.0？

- **难度**: senior
- **标签**: 升级, 生产实践

**答案**

**第一步：升级到 6.0 并清理废弃项**

| 变化 | 处理 |
|------|------|
| `strict` 默认 true | 原来没开的项目显式写 `"strict": false` |
| `types` 默认 `[]` | 显式列出 `["node", "vitest/globals"]` 等，否则全局类型丢失 |
| `rootDir` 默认 `.` | tsconfig 在 `src` 外的项目显式写 `"rootDir": "./src"`，否则输出目录结构变化 |
| `module` 默认 esnext、`target` 默认最新 ES | 依赖旧默认值的项目显式配置 |
| 废弃 `moduleResolution: node/classic` | 打包项目改 `bundler`，Node 项目改 `nodenext` |
| 废弃 `baseUrl` | `paths` 改为相对 tsconfig 的路径 |
| 废弃 `target: es5`、`downlevelIteration` | 需要 ES5 时交给 Babel/SWC 降级 |
| 废弃 `esModuleInterop: false` | 检查默认导入的写法 |
| 废弃 `import ... assert {}` | 改为 `with {}` |
| 废弃 `module Foo {}` 命名空间写法 | 改为 `namespace Foo {}` |

可临时用 `"ignoreDeprecations": "6.0"` 压住报错，但 7.0 不再支持。

**第二步：为 7.0 做准备**

1. 开启 `stableTypeOrdering`，修复因顺序变化出现的报错
2. 去掉 `ignoreDeprecations`
3. 盘点依赖编译器 API 的工具：typescript-eslint（类型感知规则）、Vue/Svelte 语言工具、自定义 transformer、ts-morph、文档生成工具

**第三步：分层切换**

- CI 的 `tsc --noEmit` 与构建先切 7.0（收益最大、风险最低）
- 编辑器：VS Code 安装 TS 7 扩展；Vue 等项目暂时继续用 6.0
- lint：继续用 6.0 API（通过 npm 别名让 `typescript` 指向 `@typescript/typescript6`）
- 对比 6.0 与 7.0 的错误列表和 `.d.ts` 输出，在 CI 中固定 `--checkers` 数值

**口述模板（30–60 秒）**

1. 一句话结论：先升 6.0 清理废弃项和新默认值，再开 stableTypeOrdering 对齐，最后分层切 7.0。
2. 关键机制：6.0 是过渡版本，7.0 与它的检查结果保持一致。
3. 注意点：types 和 rootDir 的新默认值最容易造成意外；依赖编译器 API 的工具要留在 6.0。
4. 可追问时补充：CI 先切收益最大；用 npm 别名让 6.0 和 7.0 并存。

**常见追问**

- Monorepo 里各包进度不同怎么办？→ 根目录统一 TS 版本；先在共享 tsconfig 基础配置中显式写好新默认值，再逐个包清理。

---

### Q5. `isolatedDeclarations` 是什么？解决什么问题？

- **难度**: senior
- **标签**: 声明文件, 构建性能

**答案**

问题：生成 `.d.ts` 需要知道导出函数的返回类型等信息，未标注时必须运行完整的类型检查。在 Monorepo 中，下游项目必须**等上游完成类型检查**才能拿到声明，构建只能串行。

`isolatedDeclarations: true`（5.5+）要求**导出的 API 有足够的显式标注**，使得每个文件可以**不做类型检查、单独生成声明**：

```ts
export function add(a: number, b: number) { return a + b }          // 报错：需要显式返回类型
export function add2(a: number, b: number): number { return a + b }  // OK

export const config = { port: 3000 }   // OK：简单字面量可以直接推断
```

收益：

- 声明文件可以由 **oxc、SWC 等工具并行快速生成**
- 项目引用的构建不再被上游的类型检查阻塞
- 公开 API 的类型显式可见，代码审查更容易发现破坏性变更

代价：导出的函数和复杂常量都需要写类型标注（编辑器有快速修复）。

**口述模板（30–60 秒）**

1. 一句话结论：要求导出 API 显式标注，使声明文件能单文件生成，不依赖类型检查。
2. 关键机制：去掉声明生成对 Checker 的依赖，从而可以并行。
3. 注意点：增加标注成本，适合库和 Monorepo 的共享包。
4. 可追问时补充：和 isolatedModules 是同一思路——让单文件工具能做更多事。

**常见追问**

- 业务应用需要开吗？→ 一般不需要；应用不对外发布声明，收益小。

---

## 二、类型系统深层机制

### Q6. 控制流分析（CFA）是怎么工作的？为什么在回调中收窄会失效？

- **难度**: senior
- **标签**: 控制流分析, 原理

**答案**

Binder 阶段为每个函数构建**控制流图**，节点记录赋值、条件判断、分支合并等。Checker 读取某个变量的类型时，从引用位置**沿控制流图向上回溯**，应用路径上的收窄条件，得到当前位置的类型。

收窄在回调中失效的原因：

```ts
function f(x: string | null) {
  if (x !== null) {
    setTimeout(() => x.length)   // 5.4 之前：x 可能为 null
  }
  x = null
}
```

回调**何时执行无法确定**，执行时变量可能已被重新赋值，所以编译器保守地丢弃收窄结果。

TS 5.4 的改进：如果变量在**创建闭包之后不再被赋值**，闭包内保留收窄结果。`const` 变量和参数一直可以保留。

其他常见失效场景：

```ts
if (obj.a !== undefined) {
  doSomething()             // 函数调用后，属性收窄仍然保留（TS 有意的不安全取舍）
  obj.a.toFixed()
}

if (map[key] !== undefined) map[key].x    // 动态 key：只有 key 为 const 或字面量时才能收窄
```

函数调用不会使属性收窄失效，这是为了可用性做出的不安全取舍（否则几乎所有收窄都要重复判断）。

**口述模板（30–60 秒）**

1. 一句话结论：编译器沿控制流图回溯，应用路径上的判断条件得到当前类型。
2. 关键机制：回调执行时机不确定，变量可能被重新赋值，所以默认丢弃收窄。
3. 注意点：5.4 起闭包创建后不再赋值的变量可以保留收窄。
4. 可追问时补充：函数调用后属性收窄仍保留，这是有意的不安全取舍。

**常见追问**

- 怎样稳妥地在回调里使用收窄后的值？→ 先赋给 `const` 局部变量。

---

### Q7. 类型推断的过程是怎样的？什么是上下文类型？

- **难度**: senior
- **标签**: 推断, 原理

**答案**

两个方向：

1. **自底向上**：从表达式推断类型（`const a = [1, 'x']` → `(string | number)[]`），涉及字面量拓宽和「最佳公共类型」
2. **自顶向下（上下文类型，contextual typing）**：从所在位置的期望类型反推表达式类型，比如回调参数

```ts
window.addEventListener('click', e => e.clientX)   // e 从事件签名得到 MouseEvent
```

泛型调用的推断步骤（简化）：

1. 跳过**上下文敏感**的实参（参数未标注类型的函数），先从其他实参收集类型参数的候选
2. 根据候选确定类型参数（考虑优先级：直接推断 > 来自返回值 > 默认值和约束）
3. 用确定的类型参数为上下文敏感的函数提供参数类型，再从这些函数的返回值继续推断
4. 用最终的类型参数检查所有实参

TS 6.0 的改进：方法语法的函数如果**不使用 `this`**，就不再视为上下文敏感，因此对象字面量中属性顺序不再影响推断。

推断失败的常见修复：显式标注回调参数、调整参数顺序（让可推断的参数在前）、拆分为柯里化调用、使用 `NoInfer`。

**口述模板（30–60 秒）**

1. 一句话结论：推断分自底向上和上下文类型两个方向，泛型调用会先跳过上下文敏感的回调。
2. 关键机制：先从普通实参确定类型参数，再把结果作为上下文类型提供给回调。
3. 注意点：回调参数没标注时，推断依赖其他实参和顺序。
4. 可追问时补充：6.0 起不使用 this 的方法语法不再是上下文敏感的。

**常见追问**

- 为什么 `useState([])` 推断为 `never[]`？→ 空数组字面量作为泛型实参时没有元素可供推断，在 strictNullChecks 下得到 `never[]`，需要显式传入 `useState<Item[]>([])`。而 `const a = []` 这种局部变量是「演进数组（evolving array）」，类型会随后续 push 变化。

---

### Q8. TypeScript 类型系统有哪些已知的不安全点？

- **难度**: senior
- **标签**: 类型安全, 高频

**答案**

TS 的设计目标**不是可靠（sound）**，而是在可用性和安全之间取舍。已知不安全点：

| 不安全点 | 例子 |
|---------|------|
| `any` | 关闭所有检查并会传染 |
| 类型断言 | `x as T` |
| 方法参数双变 | `interface { fn(x: Dog): void }` |
| 数组协变 | `const a: Animal[] = dogs; a.push(cat)` |
| 索引访问不带 undefined | 未开启 `noUncheckedIndexedAccess` 时 |
| 可选属性与 undefined | 未开启 `exactOptionalPropertyTypes` 时 |
| 函数调用后属性收窄保留 | 被调用函数可能修改了属性 |
| 只读属性可赋给可变类型 | `const m: { x: number } = readonlyObj` |
| 类型谓词与断言函数 | 实现错误不会被检查 |
| 参数数量兼容 | 参数少的函数可以赋给参数多的函数类型（这一点是安全的，但常被误解） |
| 外部数据 | `JSON.parse`、`res.json()` 返回 any |

工程对策：

- 开启 `strict` + `noUncheckedIndexedAccess` + `exactOptionalPropertyTypes`
- 用属性语法声明回调类型
- 用 lint 禁止 any 和不安全的断言
- 在信任边界做运行时校验

**口述模板（30–60 秒）**

1. 一句话结论：TS 不追求完全可靠，any、断言、方法双变、数组协变、索引访问是主要漏洞。
2. 关键机制：这些都是为了兼容 JS 习惯和可用性而做的取舍。
3. 注意点：类型检查通过不代表运行时安全。
4. 可追问时补充：严格选项 + lint + 边界校验是工程上的补救组合。

**常见追问**

- 为什么不默认修复数组协变？→ 会让大量正常的只读用法报错；如果需要安全，应该用 `readonly` 数组（只读数组的协变是安全的）。

---

### Q9. 泛型的型变标注 `in` / `out` 是什么？编译器怎么判断型变？

- **难度**: senior
- **标签**: 型变, 性能

**答案**

比较 `Box<Dog>` 和 `Box<Animal>` 时，编译器不想每次都展开比较结构，而是先**测量**类型参数的型变（用特殊的标记类型代入，观察可赋值关系），之后直接比较类型参数。

测量可能不准确（例如遇到条件类型、递归类型），或者开销很大。TS 4.7 起可以**显式标注**：

```ts
interface Producer<out T> { get(): T }                 // 协变
interface Consumer<in T> { set(v: T): void }           // 逆变
interface Store<in out T> { get(): T; set(v: T): void } // 不变
```

作用：

1. **性能**：跳过型变测量，复杂泛型类型的比较更快
2. **文档与检查**：标注与实际使用不一致时报错

使用建议：只在**库作者遇到性能或推断问题**时使用；标注错误会导致不安全，编译器只做有限的验证。

**口述模板（30–60 秒）**

1. 一句话结论：in/out 显式声明泛型参数的型变，out 协变、in 逆变、in out 不变。
2. 关键机制：编译器默认测量型变并缓存，标注后可以跳过测量。
3. 注意点：主要用于性能优化和复杂类型，业务代码很少需要。
4. 可追问时补充：标注错误会造成不安全的赋值，需谨慎。

**常见追问**

- 型变测量什么时候不准？→ 类型参数用在条件类型、映射类型中时，测量结果可能被标记为「不可靠」，编译器会退回结构比较。

---

### Q10. 品牌类型和「解析而不是校验」的建模思想是什么？

- **难度**: senior
- **标签**: 领域建模

**答案**

**Parse, don't validate（解析而不是校验）**：校验函数返回 `boolean`，类型信息随即丢失，后续代码还要重复判断；解析函数返回**更精确的类型**，把「已校验」这个事实写进类型。

```ts
declare const EmailBrand: unique symbol
type Email = string & { readonly [EmailBrand]: true }

function parseEmail(s: string): Email | null {
  return /^[^@\s]+@[^@\s]+$/.test(s) ? (s as Email) : null
}

function sendMail(to: Email) {}       // 只接受解析过的值
sendMail('a@b.com')                   // 报错：string 不能赋给 Email
```

在业务中的应用：

- 区分 ID 类型：`UserId` / `OrderId`
- 区分单位：`Cents` / `Yuan`、`Ms` / `Seconds`
- 区分状态：`SanitizedHtml`（已转义）vs `string`，防止 XSS
- 非空数组：`[T, ...T[]]`

Zod 的 `.brand<'Email'>()` 可以在运行时校验后得到品牌类型。

**口述模板（30–60 秒）**

1. 一句话结论：用品牌类型把「已经校验过」记录在类型里，后续不用重复判断。
2. 关键机制：交叉一个不存在于运行时的标记属性，只有解析函数能产出该类型。
3. 注意点：品牌只存在于类型层，构造入口要集中管理。
4. 可追问时补充：金额单位、已转义 HTML、各种 ID 都适合这样建模。

**常见追问**

- 为什么用 `unique symbol` 而不是字符串属性？→ 防止被其他同名品牌意外满足，也不会出现在属性补全中。

---

### Q11. 类型层面的编程有哪些边界？什么时候应该停下？

- **难度**: senior
- **标签**: 类型编程, 取舍

**答案**

TS 类型系统图灵完备，可以实现解析 SQL、JSON 甚至运行算法。但工程上要关注成本：

| 成本 | 表现 |
|------|------|
| 检查性能 | IDE 卡顿、`tsc` 变慢、TS2589 深度报错 |
| 错误信息 | 出错时展示几十行展开的类型，难以理解 |
| 维护 | 只有少数人看得懂，修改风险高 |
| 版本兼容 | 依赖编译器细节的写法可能在新版本中变化（如 TS 7 的 Unicode 模板字面量拆分） |

判断标准：

1. **调用方**是否因此获得了明显更好的体验（补全、错误提示）
2. 复杂度是否**集中在库内部**，而不是扩散到业务代码
3. 有没有**类型测试**覆盖（`expectTypeOf`、`tsd`）
4. 出错时信息是否可读（可以用约束或自定义错误字符串类型提示用户）

经验：

- 业务代码中，简单清晰的类型优于「完美精确」的类型
- 复杂类型放在基础库中，配类型测试和文档
- 超过 3 层嵌套的条件类型，考虑用代码生成（OpenAPI、GraphQL codegen、Prisma）替代

**口述模板（30–60 秒）**

1. 一句话结论：类型编程能力很强，但要权衡检查性能、错误可读性和维护成本。
2. 关键机制：复杂度要收敛在库内部，并有类型测试保护。
3. 注意点：业务代码追求清晰，不追求类型完美。
4. 可追问时补充：过于复杂的类型推导可以改为代码生成。

**常见追问**

- 怎么给复杂类型写测试？→ Vitest 的 `expectTypeOf` / `--typecheck`、`tsd`，或 `type Assert<T extends true>` 配合 `Equal` 工具类型。

---

## 三、性能与工程化

### Q12. 项目类型检查很慢，怎么排查和优化？

- **难度**: senior
- **标签**: 性能, 高频

**答案**

**排查工具**：

```sh
tsc --noEmit --extendedDiagnostics     # 看 Files、Types、Instantiations、Check time、Memory
tsc --noEmit --generateTrace ./trace   # 生成 trace，用 @typescript/analyze-trace 或 Perfetto 分析热点
tsc --listFilesOnly                    # 检查是否包含了不该包含的文件
```

重点指标：`Instantiations`（类型实例化次数）、`Types` 数量、`Check time`。

**常见原因与优化**：

| 原因 | 优化 |
|------|------|
| include 范围过大（含构建产物、node_modules） | 收紧 `include`、`exclude` |
| 检查第三方声明 | `skipLibCheck: true` |
| 大量交叉类型 | 改为 `interface extends` |
| 巨大的联合类型（数千个成员） | 拆分，或改用 `string` 加运行时校验 |
| 深递归条件类型、模板字面量笛卡尔积 | 限制深度，缓存中间类型为具名类型 |
| 复杂的导出函数未标注返回类型 | 显式标注，减少声明生成和比较开销 |
| 多态组件、样式库的复杂泛型 | 评估替换，或缩小使用范围 |
| 单一巨型项目 | 项目引用拆分 + 增量编译 |
| 全局 `@types` 过多 | 用 `types` 字段（TS 6 默认为空）显式指定 |

**结构性优化**：

- 转译与检查分离：开发用 Vite / Node 类型擦除，检查放在编辑器和 CI
- CI 中利用 `.tsbuildinfo` 缓存
- **升级 TS 7**：全量构建快 8–12 倍，在 CI 中根据机器核数调整 `--checkers`

**口述模板（30–60 秒）**

1. 一句话结论：用 extendedDiagnostics 看指标，用 generateTrace 找热点文件和类型。
2. 关键机制：慢的根源通常是实例化次数过多或检查范围过大。
3. 注意点：交叉类型、大联合、深递归、复杂泛型组件是常见元凶。
4. 可追问时补充：结构上靠项目引用、增量和 TS 7 并行，CI 中缓存 tsbuildinfo。

**常见追问**

- 编辑器卡但 `tsc` 不慢怎么办？→ 可能是某个文件中的复杂类型在悬停和补全时被反复计算；用 TS Server 日志或 trace 定位，必要时把复杂类型拆成具名类型。

---

### Q13. Monorepo 中 TS 有哪几种组织方式？怎么选？

- **难度**: senior
- **标签**: Monorepo, 架构

**答案**

| 方式 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| `paths` 别名 | 根 tsconfig 用 `paths` 指向各包源码 | 简单，跳转到源码 | 运行时和打包器也要配别名；包边界不清晰 |
| 内部包直接导出源码（Just-in-Time） | 包的 `exports` 指向 `.ts` 源码，由消费方的打包器转译 | 无需预构建，开发快 | 每个消费方都要检查全部源码；不能发布到 npm |
| 预构建包 | 每个包产出 `dist` + `.d.ts` | 边界清晰，可发布 | 需要 watch 构建，开发链路长 |
| 项目引用 | `composite` + `references` + `tsc -b` | 增量构建，检查范围小 | 配置复杂，需要维护引用关系 |

选择思路：

- 中小型 Monorepo：workspace 包 + 直接导出源码，CI 整体检查
- 大型、检查慢：项目引用 + `isolatedDeclarations` + TS 7 的 `--builders` 并行
- 需要对外发布的包：预构建，`attw` 检查产物

配套：共享的 `tsconfig.base.json`；用 `customConditions`（如 `"source"`）让开发时解析到源码、发布时解析到产物。

**口述模板（30–60 秒）**

1. 一句话结论：paths 别名、源码直连、预构建、项目引用四种方式，按规模和是否发布选择。
2. 关键机制：项目引用让下游依赖上游的声明文件，检查范围缩小。
3. 注意点：paths 只影响 TS，运行时也要配置别名。
4. 可追问时补充：customConditions 可以让同一个包在开发和发布时解析到不同入口。

**常见追问**

- 为什么不推荐只用 `paths`？→ 容易形成循环依赖和越界引用；`paths` 不会改写输出路径，Node 运行时无法识别。

---

### Q14. 转译工具（tsc / Babel / SWC / esbuild / Oxc）怎么选？

- **难度**: senior
- **标签**: 工具链

**答案**

| 工具 | 类型检查 | 速度 | 生成 .d.ts | 定位 |
|------|---------|------|-----------|------|
| tsc（6.0 JS 版） | 有 | 慢 | 有 | 官方检查与产出 |
| tsc（7.0 Go 版） | 有 | 快 8–12 倍 | 有 | 官方检查与产出 |
| Babel | 无 | 中 | 无 | 生态插件丰富，老项目 |
| SWC | 无 | 快 | 实验性（基于 isolatedDeclarations） | Next.js 等 |
| esbuild | 无 | 快 | 无 | Vite 开发态（旧版）、工具脚本 |
| Oxc | 无 | 很快 | 有（基于 isolatedDeclarations） | Rolldown / 新版 Vite |
| Node 类型擦除 | 无 | 快 | 无 | 直接运行脚本和服务 |

推荐架构：

- **转译**：打包器内置工具（Vite 用 Oxc/esbuild；Next.js 用 SWC）
- **检查**：编辑器实时检查 + CI 中 `tsc --noEmit`（TS 7）
- **声明**：库项目用 tsc 或 tsup / tsdown 生成；开启 `isolatedDeclarations` 后可用 Oxc 加速

约束：使用非 tsc 转译时需要 `isolatedModules` / `verbatimModuleSyntax`；不能依赖 `const enum` 跨文件内联和 `emitDecoratorMetadata`（SWC 有部分支持）。

**口述模板（30–60 秒）**

1. 一句话结论：转译交给快速的单文件工具，类型检查交给 tsc。
2. 关键机制：单文件工具只删除类型，所以需要 isolatedModules 一类的约束。
3. 注意点：装饰器元数据和 const enum 在非 tsc 工具中有兼容问题。
4. 可追问时补充：TS 7 让 tsc 检查本身变快，CI 瓶颈明显下降。

**常见追问**

- 既然 tsc 7 很快了，还需要其他转译工具吗？→ 需要。打包器还要做打包、HMR、压缩，转译只是其中一环，且单文件转译在 HMR 中仍然更快。

---

### Q15. 如何实现前后端端到端的类型安全？

- **难度**: senior
- **标签**: 端到端类型, 架构, 高频

**答案**

| 方案 | 原理 | 适用 |
|------|------|------|
| 共享类型包 | 前后端 Monorepo 共享 TS 类型 | 都是 TS，团队一致 |
| 共享 Zod schema | schema 同时用于后端校验、前端推导类型 | 都是 TS，需要运行时校验 |
| tRPC | 前端直接从后端路由定义推导类型，不生成代码 | 全栈 TS、内部系统 |
| OpenAPI codegen | 后端产出 OpenAPI 文档 → 生成 TS 类型与客户端（openapi-typescript、orval） | 后端非 TS、多端消费 |
| GraphQL codegen | 从 schema 和查询语句生成类型 | 已有 GraphQL |

关键设计点：

1. **单一数据源**：类型来自 schema 或后端定义，不手写两份
2. **运行时校验**：后端必须校验输入；前端对关键响应校验，或在开发环境中校验
3. **契约变更流程**：生成的类型提交到仓库或在 CI 中生成，接口不兼容的变更导致前端编译失败，从而在合并前发现
4. **版本兼容**：前后端独立发布时，后端需保持向后兼容（只新增可选字段），不能只依赖类型

tRPC 与 OpenAPI 的取舍：tRPC 开发体验最好，但耦合了前后端技术栈；OpenAPI 跨语言、可对外开放，但需要代码生成流程。

**口述模板（30–60 秒）**

1. 一句话结论：核心是单一数据源，类型从 schema 或后端定义推导或生成。
2. 关键机制：全栈 TS 用 tRPC 或共享 Zod；多语言后端用 OpenAPI codegen。
3. 注意点：类型不能替代运行时校验，也不能解决独立发布时的版本兼容。
4. 可追问时补充：在 CI 中生成类型，接口破坏性变更会直接导致前端编译失败。

**常见追问**

- 前端对每个响应都用 Zod 校验会影响性能吗？→ 大对象或列表会有开销；可以只校验关键接口，或在生产环境降级为抽样校验并上报。

---

### Q16. 如何设计一个类型友好的库 API？

- **难度**: senior
- **标签**: 库设计

**答案**

原则：

1. **让推断替用户工作**：调用处尽量不需要写泛型参数和类型标注
2. **推断来源明确**：用一个参数决定类型参数，其他参数用 `NoInfer` 只做检查
3. **保留字面量**：使用 `const` 类型参数
4. **错误信息可读**：约束放在参数上而不是返回值上，让错误出现在用户写错的位置
5. **导出关键类型**：用户需要标注变量时能引用（如 `type Options`、`InferInput<T>`）
6. **公开类型稳定**：内部复杂类型不导出，避免成为破坏性变更的来源

```ts
// 典型的 defineXxx 模式
export function defineRoutes<const R extends Record<string, RouteDef>>(routes: R): R {
  return routes
}

// 从定义中推导，而不是让用户手写
export type RouteName<R> = keyof R & string
```

需要注意：

- **声明文件生成**：导出函数的推断返回类型如果引用了未导出的内部类型，会报 TS2742 / TS4023 等错误，需要显式导出或标注
- **兼容多个 TS 版本**：`typesVersions` 可以为旧版本 TS 提供不同的声明
- **类型测试**：用 `expectTypeOf` 等测试推断结果，防止重构引入回归

**口述模板（30–60 秒）**

1. 一句话结论：让推断替用户工作，推断来源单一，错误出现在用户写错的位置。
2. 关键机制：const 类型参数保留字面量，NoInfer 控制推断来源。
3. 注意点：导出 API 的类型要稳定，并有类型测试。
4. 可追问时补充：defineXxx 模式是现代库（Vite、Vue、Zod、tRPC）的通用设计。

**常见追问**

- 什么是 TS2742？→ 声明文件中需要引用一个无法通过可移植路径访问的类型（常见于 pnpm 的深层依赖），解决方法是显式标注返回类型或导出该类型。

---

### Q17. 类型测试怎么写？

- **难度**: senior
- **标签**: 测试

**答案**

库和基础设施代码的类型推断本身就是 API，需要测试防止回归。

```ts
// 1. 手写断言工具（零依赖）
type Equal<X, Y> =
  (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false
type Expect<T extends true> = T

type _1 = Expect<Equal<ReturnType<typeof parse>, User | null>>

// 2. Vitest
import { expectTypeOf, test } from 'vitest'
test('infer', () => {
  expectTypeOf(defineRoutes({ home: { path: '/' } })).toHaveProperty('home')
  expectTypeOf(parse).parameter(0).toEqualTypeOf<string>()
})

// 3. 预期报错
// @ts-expect-error 不允许传入未知路由
navigate('nope')
```

`Equal` 用两个泛型函数比较的原因：普通的双向 `extends` 判断无法区分 `any` 与其他类型，也无法区分 `readonly` 等修饰符；这种写法利用了编译器对「延迟条件类型」的**同一性**比较，更严格。

工具：Vitest `--typecheck`、`tsd`、`tstyche`、`expect-type`。

**口述模板（30–60 秒）**

1. 一句话结论：用 Equal + Expect、expectTypeOf 或 ts-expect-error 测试类型推断。
2. 关键机制：类型测试在编译期执行，失败表现为类型错误。
3. 注意点：Equal 要用严格写法，否则 any 会骗过断言。
4. 可追问时补充：ts-expect-error 用来测试「应该报错」的场景。

**常见追问**

- 类型测试放在哪里运行？→ CI 中执行 `tsc --noEmit` 或 Vitest typecheck 模式，与单元测试一起运行。

---

### Q18. 大型代码库如何持续治理类型质量？

- **难度**: senior
- **标签**: 治理

**答案**

**度量**：

- `type-coverage`：非 any 标识符占比
- `@ts-expect-error` / `@ts-ignore` / `as any` 的数量
- strict 子选项开启情况
- `tsc` 检查耗时与 `Instantiations` 数量（性能也是质量）

**门禁**：

- CI：`tsc --noEmit` 必须通过；类型覆盖率**只升不降**（棘轮策略）
- lint：`no-explicit-any`、`no-unsafe-*`、`consistent-type-imports`、`ban-ts-comment`（要求说明）
- 代码审查：新增断言必须附原因

**基础设施**：

- 共享的 `tsconfig.base.json`，各项目只覆盖必要的选项
- 公共类型包（API 类型、领域模型、工具类型），统一维护
- 接口类型来自代码生成，禁止手写

**持续改进**：

- 每个季度挑选一个 strict 子选项推进（如 `noUncheckedIndexedAccess`），先统计报错数，用 codemod 批量处理
- 新代码目录（如 `src/features/new`）单独使用更严格的 tsconfig

**口述模板（30–60 秒）**

1. 一句话结论：度量 + 门禁 + 共享基础设施 + 渐进收紧。
2. 关键机制：棘轮策略保证不变差，codemod 批量改进。
3. 注意点：不要一次性强推，按模块和选项分批推进。
4. 可追问时补充：检查耗时也是质量指标，要纳入监控。

**常见追问**

- 怎么说服团队开启更严格的选项？→ 用线上事故数据说明（如空值错误占比），先在一个模块试点并展示收益。

---

## 四、框架与生态

### Q19. TS 中如何正确处理错误类型？

- **难度**: senior
- **标签**: 错误处理

**答案**

问题：JS 可以 `throw` 任何值，函数签名中**无法声明会抛出什么错误**，`catch` 中的 `e` 在 strict 下是 `unknown`。

```ts
try { await run() }
catch (e) {
  if (e instanceof HttpError) handleHttp(e)
  else if (e instanceof Error) log(e.message)
  else log(String(e))
}
```

两种风格：

1. **异常 + 错误类层次**：定义 `class AppError extends Error { code: ... }`，在边界统一捕获
2. **Result 类型**：把预期内的错误作为返回值，在类型中显式表达

```ts
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E }

function parseConfig(raw: string): Result<Config, 'INVALID_JSON' | 'MISSING_FIELD'> {
  // ...
}
const r = parseConfig(text)
if (!r.ok) {
  switch (r.error) { /* 必须处理每一种错误 */ }
}
```

取舍：

- **预期内的业务错误**（校验失败、余额不足）用 Result，调用方被迫处理
- **预期外的错误**（网络中断、程序 bug）用异常
- 可以使用 neverthrow、Effect 等库

注意：继承 `Error` 时，目标为 ES5 会导致原型链断裂（`instanceof` 失效），需要 `Object.setPrototypeOf`；TS 6 起已废弃 ES5 目标。

**口述模板（30–60 秒）**

1. 一句话结论：TS 无法声明抛出类型，catch 变量是 unknown，必须收窄。
2. 关键机制：业务错误可以用 Result 可辨识联合表达，强制调用方处理。
3. 注意点：预期内用 Result，预期外用异常。
4. 可追问时补充：自定义错误类在 ES5 目标下有原型链问题。

**常见追问**

- Promise 的 reject 类型能声明吗？→ 不能，`Promise<T>` 只有成功类型参数。

---

### Q20. React + TS 在大型项目中有哪些进阶问题？

- **难度**: senior
- **标签**: React, 进阶

**答案**

1. **Context 类型与空值**：

```ts
const Ctx = createContext<Store | null>(null)
export function useStore() {
  const v = useContext(Ctx)
  if (!v) throw new Error('useStore must be used within Provider')
  return v                                   // 返回类型为 Store，调用方无需判空
}
```

2. **组件 Props 的组合**：`ComponentPropsWithoutRef<'button'>` + 自定义 props + `Omit` 冲突字段；可辨识联合表达互斥 props：

```ts
type Props =
  | { mode: 'link'; href: string }
  | { mode: 'button'; onClick: () => void }
```

3. **泛型组件与 ref**：React 19 之前 `forwardRef` 丢失泛型；React 19 中 `ref` 作为普通 prop，泛型组件可以直接接收 ref
4. **React 19 的类型变化**：`useRef` 必须传参数、移除全局 `JSX` 命名空间（改用 `React.JSX`）、`ReactElement` 的 props 默认 `unknown`
5. **性能**：多态组件、样式库（CSS-in-JS 的复杂泛型）、巨大的联合 props 会拖慢编辑器
6. **状态库的类型推断**：Zustand 的 `create<State>()(...)` 柯里化写法，是为了绕过「无法部分指定类型参数」的限制

**口述模板（30–60 秒）**

1. 一句话结论：Context 用自定义 Hook 去掉空值，互斥 props 用可辨识联合。
2. 关键机制：柯里化调用绕过无法部分推断类型参数的限制。
3. 注意点：React 19 的 ref 和 JSX 命名空间类型有破坏性变化。
4. 可追问时补充：多态组件和样式库会拖慢编辑器，需要权衡。

**常见追问**

- 为什么 Zustand 要写 `create<T>()(fn)`？→ TS 不支持部分指定类型参数，柯里化后第一次调用显式传入 State，第二次调用的其他类型参数仍可推断。

---

### Q21. 扩充框架类型（模块扩充）在大型项目中有什么风险？

- **难度**: senior
- **标签**: 声明合并, 架构

**答案**

常见扩充：Express 的 `Request.user`、Vue 的 `ComponentCustomProperties`、Pinia 插件属性、`Window` 全局变量、环境变量 `ProcessEnv`。

风险：

1. **类型与运行时不一致**：扩充声明 `req.user` 始终存在，但只有经过认证中间件的路由才有。应声明为可选，或在路由层用泛型参数精确表达
2. **全局污染**：扩充是全局生效的，Monorepo 中一个包的扩充可能影响其他包
3. **加载依赖**：扩充文件没有被 include，在某些包中类型缺失，表现为「本地正常、CI 报错」
4. **版本升级**：被扩充的内部模块路径（如 `express-serve-static-core`）在大版本中可能变化

做法：

- 扩充集中在一个文件，并在 tsconfig 中显式包含
- 能用显式参数或泛型的地方优先不用扩充（如 Hono、Fastify 的类型化上下文）
- `ProcessEnv` 扩充需配合启动时的环境变量运行时校验

**口述模板（30–60 秒）**

1. 一句话结论：模块扩充方便，但会造成类型与运行时不一致和全局污染。
2. 关键机制：扩充是全局合并，无法限定作用域。
3. 注意点：只在部分路径存在的属性要声明为可选。
4. 可追问时补充：新框架倾向于用泛型显式传递上下文类型，而不是全局扩充。

**常见追问**

- 环境变量类型怎么做最可靠？→ 启动时用 Zod 解析 `process.env`，导出解析后的配置对象，业务代码不直接访问 `process.env`。

---

### Q22. Node 原生运行 TS 和 `erasableSyntaxOnly` 对团队有什么影响？

- **难度**: senior
- **标签**: Node, 类型擦除

**答案**

Node 22.18+ / 23.6+ 默认支持**类型擦除（Type Stripping）**：把类型注解替换为空白（保持行列号不变，不需要 source map），直接运行 `.ts` 文件。

不支持需要生成代码的 TS 语法：

| 语法 | 替代 |
|------|------|
| `enum` | `as const` 对象 + 联合类型 |
| `namespace`（含值） | ES 模块 |
| 构造函数参数属性 `constructor(private x)` | 显式声明字段 |
| `import x = require()` | 标准 import / `createRequire` |
| 旧版装饰器 | 标准装饰器（需要运行时支持） |

TS 5.8 的 `erasableSyntaxOnly` 会在编译期禁止上述语法。

对团队的影响：

- 脚本、工具、小型服务可以**去掉构建步骤**
- 编码规范需要统一转向「可擦除」写法，这与 TC39 的类型注解提案方向一致
- 仍然需要 `tsc --noEmit` 做类型检查
- 导入路径要写 `.ts` 扩展名（`allowImportingTsExtensions`），发布时可用 `rewriteRelativeImportExtensions` 改写为 `.js`
- `node_modules` 中的 `.ts` 文件不会被擦除，发布的库仍需构建为 JS

**口述模板（30–60 秒）**

1. 一句话结论：Node 可以直接擦除类型运行 TS，前提是只使用可擦除语法。
2. 关键机制：用空白替换类型，行列号不变。
3. 注意点：enum、namespace、参数属性不支持；类型检查仍需 tsc。
4. 可追问时补充：用 erasableSyntaxOnly 在编译期约束，团队规范逐步转向可擦除写法。

**常见追问**

- TC39 类型注解提案是什么？→ 让 JS 引擎把类型语法当作注释忽略。它和 Node 类型擦除方向一致，但目前仍处于早期阶段。

---

### Q23. 如何用类型建模复杂的业务状态？

- **难度**: senior
- **标签**: 建模, 状态机

**答案**

以订单为例：

```ts
type Order =
  | { status: 'created'; id: OrderId; items: Item[] }
  | { status: 'paid'; id: OrderId; items: Item[]; paidAt: Date; payId: string }
  | { status: 'shipped'; id: OrderId; items: Item[]; paidAt: Date; payId: string; trackingNo: string }
  | { status: 'cancelled'; id: OrderId; reason: string }

type OrderOf<S extends Order['status']> = Extract<Order, { status: S }>

// 状态转移函数只接受合法的源状态
function ship(o: OrderOf<'paid'>, trackingNo: string): OrderOf<'shipped'> {
  return { ...o, status: 'shipped', trackingNo }
}
```

收益：

- 不同状态拥有的字段在类型上就被区分开，不会访问到不存在的数据
- 状态转移的合法性在编译期检查
- 新增状态时，所有 switch 通过 never 穷尽检查提示需要处理

延伸：

- 用映射类型定义状态转移表 `{ created: 'paid' | 'cancelled'; paid: 'shipped' | 'cancelled' }`，事件分发函数从表中推导合法的目标状态
- 复杂流程可以使用 XState，它提供基于 TS 的类型推导

注意：后端返回的数据必须先通过运行时校验（Zod 的 `discriminatedUnion`）才能信任状态与字段的对应关系。

**口述模板（30–60 秒）**

1. 一句话结论：用可辨识联合表达每种状态拥有的字段，用函数签名限定状态转移。
2. 关键机制：Extract 按状态取类型，never 穷尽检查兜底。
3. 注意点：外部数据要用 discriminatedUnion 校验后才能信任。
4. 可追问时补充：状态转移表可以用映射类型表达，复杂流程用 XState。

**常见追问**

- 字段很多时会不会重复？→ 用公共基础类型交叉或 `extends`，只在各状态中写差异字段。

---

### Q24. 如何调试复杂类型？

- **难度**: senior
- **标签**: 调试

**答案**

1. **展开类型**：悬停时显示的是别名，可以用工具类型展开：

```ts
type Expand<T> = T extends infer O ? { [K in keyof O]: O[K] } : never
type Debug = Expand<ComplexType>
```

2. **分步拆解**：把复杂条件类型拆为多个中间具名类型，逐个悬停查看
3. **Twoslash 注释**：在 TS Playground 中使用 `// ^?` 显示表达式的类型
4. **强制报错查看类型**：`const _: never = value`，错误信息会显示 value 的完整类型
5. **TS 7 编辑器的可展开悬停**（expandable hover）可以逐层展开类型
6. **性能问题**：用 `--generateTrace` 找到耗时的类型
7. **类型测试**：先写 `Expect<Equal<...>>` 断言，再迭代实现（类似 TDD）

**口述模板（30–60 秒）**

1. 一句话结论：展开类型、分步拆成中间类型、借助 Playground 和报错信息观察结果。
2. 关键机制：Expand 工具类型强制编译器计算出结构。
3. 注意点：复杂类型要先写测试再实现。
4. 可追问时补充：性能问题用 generateTrace 定位。

**常见追问**

- 为什么悬停只显示别名？→ 编译器为了可读性和性能，会尽量保留类型别名而不展开。

---

### Q25. 编辑器中的 TS 语言服务是怎么工作的？TS 7 有什么变化？

- **难度**: senior
- **标签**: 语言服务

**答案**

旧架构（TS 6 及以前）：

- 编辑器启动 **tsserver**（Node 进程），通过自定义协议通信
- 基于 Program 和**增量更新**：编辑一个文件时只重新解析该文件，类型按需计算
- 补全、跳转、重命名、快速修复都基于 Checker
- 插件机制（Vue、Angular、styled-components 插件）可以修改语言服务行为

常见问题：大型项目首次加载慢、内存高（单进程受 V8 堆限制）、偶发崩溃。

TS 7：

- 语言服务基于 **LSP（Language Server Protocol）** 实现，任意支持 LSP 的编辑器都可以直接使用
- 多线程处理并发请求
- 官方数据：VS Code 仓库从打开到出现首个错误从约 17.5 秒降到约 1.3 秒；失败命令减少 80% 以上、崩溃减少 60% 以上
- **插件 API 尚不可用**，依赖插件的框架（Vue / Svelte / Astro / MDX）暂时继续使用 TS 6

**口述模板（30–60 秒）**

1. 一句话结论：编辑器通过语言服务按需检查打开的文件，TS 7 改为多线程的 LSP 实现。
2. 关键机制：增量解析 + 懒计算类型。
3. 注意点：TS 7 暂无插件 API，部分框架编辑器支持需要等待。
4. 可追问时补充：大项目的编辑器卡顿问题在 TS 7 中明显改善。

**常见追问**

- 编辑器内存过高怎么办？→ 检查是否打开了包含过多文件的 tsconfig；使用项目引用拆分；关闭不需要的插件；升级 TS 7。

---

### Q26. 如果让你制定团队的 TypeScript 规范，你会包含哪些内容？

- **难度**: senior
- **标签**: 规范, 生产实践

**答案**

**编译配置**：

- 统一 `tsconfig.base.json`：`strict`、`noUncheckedIndexedAccess`、`noImplicitOverride`、`verbatimModuleSyntax`、`isolatedModules`
- 前端 `moduleResolution: bundler`，Node 和库 `nodenext`
- 显式写出 TS 6 改变的默认值（`types`、`rootDir`），避免升级时出现意外

**编码规范**：

- 优先推断，导出函数显式标注返回类型
- 禁止 `any`，用 `unknown` 并收窄；断言需说明原因
- 用 `as const` 对象代替 `enum`，使用可擦除语法
- 对象结构用 `interface`，组合和运算用 `type`
- 状态用可辨识联合建模，switch 必须穷尽
- 外部数据在边界用 schema 校验，类型从 schema 推导
- 使用 `import type`

**工具与流程**：

- typescript-eslint 的 `strict-type-checked` 预设（按需裁剪）
- CI 中 `tsc --noEmit`、类型覆盖率门禁
- 接口类型由代码生成
- 复杂工具类型放在公共包中，并配类型测试

**升级策略**：每个 TS 小版本在一周内评估；大版本（6.0 / 7.0）按 Q4 的步骤推进。

**口述模板（30–60 秒）**

1. 一句话结论：统一配置、编码约定、工具门禁和升级策略四部分。
2. 关键机制：严格配置 + lint + CI 保证规范落地。
3. 注意点：规范要有工具自动检查，不能只靠代码审查。
4. 可追问时补充：显式写出新版本改变的默认值，降低升级风险。

**常见追问**

- `strict-type-checked` 太严格怎么办？→ 先开启 recommended-type-checked，再按团队痛点逐条开启规则；对存量代码用 `--max-warnings` 和棘轮策略。
