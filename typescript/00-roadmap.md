# TypeScript 学习路线（高级工程师面试向）

## 面试画像

国内大厂对 TypeScript 的考察已经从「会写类型注解」升级为三层：

- **基础层**：`any / unknown / never`、`type` vs `interface`、类型收窄、泛型、工具类型 —— 几乎每轮必问
- **类型编程层**：条件类型、`infer`、映射类型、模板字面量类型、协变逆变、分布式条件类型 —— 中高级岗**现场手写工具类型**是常态
- **工程层**：tsconfig 与模块解析、`.d.ts` 与库发布、Monorepo 与项目引用、类型检查性能、大型项目 JS → TS 迁移、TS 6/7 升级 —— 高级岗区分度所在

高级岗典型问法：

> 「手写一个 `DeepPartial`，数组和函数怎么处理？」「为什么函数参数是逆变的？方法为什么又是双变？」「项目 `tsc` 要跑 5 分钟，怎么优化？」「怎么做前后端端到端类型安全？」「TS 7 用 Go 重写了，对你们项目有什么影响？」

## 版本说明（截至 2026-10）

| 版本 | 时间 | 关键内容 |
|------|------|----------|
| 4.9 | 2022-11 | `satisfies`、`in` 收窄未声明属性、`accessor` |
| 5.0 | 2023-03 | 标准装饰器（TC39 Stage 3）、`const` 类型参数、`moduleResolution: bundler`、`verbatimModuleSyntax`、枚举统一为联合枚举 |
| 5.2 | 2023-08 | `using` / `await using`（显式资源管理）、装饰器元数据 `Symbol.metadata` |
| 5.4 | 2024-03 | `NoInfer<T>`、闭包中保留收窄结果（最后一次赋值之后） |
| 5.5 | 2024-06 | 推断类型谓词（`filter(x => x !== undefined)` 自动收窄）、`isolatedDeclarations`、正则语法检查 |
| 5.6–5.7 | 2024 | 禁止恒真/恒假的条件检查、迭代器辅助方法类型、`--rewriteRelativeImportExtensions`、未初始化变量检查 |
| 5.8 | 2025-02 | `--erasableSyntaxOnly`（配合 Node 原生类型擦除）、`nodenext` 下支持 `require(esm)` |
| 5.9 | 2025-08 | `import defer`、`--module node20`、更精简的 `tsc --init` |
| **6.0** | 2026 | **最后一个 JS 实现的版本，过渡版本**：`strict` 默认 `true`、`module` 默认 `esnext`、`target` 默认最新 ES、`types` 默认 `[]`、`rootDir` 默认 `.`；废弃 `target: es5`、`moduleResolution: node/classic`、`baseUrl`、`amd/umd/systemjs`、`esModuleInterop: false` 等；新增 `--stableTypeOrdering`、Temporal 类型、`#/` 子路径导入 |
| **7.0** | 2026-07 | **Go 原生重写（原代号 Project Corsa / tsgo）**：全量构建快 8–12 倍、内存更省；并行解析/检查/产出（`--checkers`、`--builders`、`--singleThreaded`）；6.0 的废弃项全部变为硬错误；**暂无编译器 API**（计划 7.1 提供），Vue / Svelte / Astro 等依赖 API 的工具暂时仍需 6.0（可用 `@typescript/typescript6` 的 `tsc6` 并存） |

> 面试中能说清「6.0 是过渡、7.0 是原生重写、两者类型检查结果一致、我们的迁移路径是什么」是明显加分项。具体细节以官方 Release Notes 为准。

## 知识地图

```text
TypeScript
├─ 基础：原始类型、对象/数组/元组、字面量、联合/交叉、any/unknown/never/void
├─ 类型声明：type vs interface、声明合并、索引签名、Record、可选与只读
├─ 函数：重载、this 参数、剩余参数、函数类型兼容性
├─ 类：访问修饰符、#private、抽象类、implements、参数属性
├─ 收窄：typeof/instanceof/in/判等/真值、可辨识联合、类型谓词、断言函数、控制流分析
├─ 泛型：约束、默认值、推断、const 类型参数、NoInfer
├─ 类型编程：keyof/typeof/索引访问、条件类型、分布式、infer、映射类型、key remapping、模板字面量、递归
├─ 类型关系：结构化类型、赋值兼容、协变/逆变/双变/不变、多余属性检查、品牌类型
├─ 模块与声明：ESM/CJS、import type、verbatimModuleSyntax、.d.ts、declare、模块扩充、@types
├─ 工程：tsconfig、strict 家族、moduleResolution、项目引用、增量编译、isolatedModules/isolatedDeclarations
├─ 工具链：tsc / Babel / SWC / esbuild / Node 类型擦除、类型检查与转译分离、TS 7 原生编译器
├─ 生态：React + TS、Zod / tRPC / OpenAPI 端到端类型、ESLint typescript-eslint、类型测试
└─ 架构：库类型设计与发布、Monorepo、any 治理、类型检查性能、大型迁移与升级
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 基础类型、type/interface、收窄、泛型入门、工具类型、tsconfig、声明文件
2. [中级 Q&A](20-mid-qa.md) — 条件类型、infer、映射类型、模板字面量、协变逆变、模块解析、装饰器、新特性
3. [高级 Q&A](30-senior-qa.md) — 编译器架构与 TS 7、推断与控制流原理、类型系统的不安全点、库设计、性能、迁移与治理
4. [手写与现场编码](40-coding-handwritten.md) — 14 道高频手写类型题（含运行时 + 类型结合题）
5. [系统设计与场景](50-system-design.md) — Monorepo 类型架构、端到端类型安全、组件库类型设计、大型迁移等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. 结构化类型（Structural Typing）的含义，以及如何用品牌类型模拟名义类型
2. `any / unknown / never / void` 的准确区别；`never` 在联合中消失、作为分布式条件类型输入返回 `never`
3. `type` vs `interface`：声明合并、扩展方式、性能差异（交叉类型 vs `extends`）
4. 类型收窄的全部方式，以及控制流分析在回调中失效的原因
5. 条件类型的分布式行为，以及用 `[T] extends [U]` 阻止分发
6. `infer` 的用法，协变位置推断出联合、逆变位置推断出交叉（`UnionToIntersection` 原理）
7. 映射类型的修饰符（`+/- readonly`、`+/- ?`）与 key remapping（`as`）
8. 手写常用工具类型：`Partial / Pick / Omit / Exclude / ReturnType / Awaited / DeepPartial`
9. 协变、逆变、双变：`strictFunctionTypes` 为什么不管方法语法
10. `satisfies` 与类型注解、`as` 断言的区别，配合 `as const`
11. 可辨识联合 + `never` 做穷尽检查
12. 模块解析：`nodenext` vs `bundler`；ESM 下导入要写 `.js` 扩展名的原因
13. `.d.ts`、`declare`、模块扩充（module augmentation）与全局扩充
14. 转译与类型检查分离：`isolatedModules`、`verbatimModuleSyntax`、`erasableSyntaxOnly`、Node 原生类型擦除
15. 项目引用、增量编译、`isolatedDeclarations` 与 Monorepo 构建
16. 类型检查性能诊断：`--extendedDiagnostics`、`--generateTrace`
17. 运行时边界校验（Zod 等）与端到端类型安全方案
18. TS 6.0 / 7.0 的变化与升级路径

## 相关技术

- [JavaScript](../javascript/00-roadmap.md)：TS 只做类型擦除，运行时行为仍是 JS（原型、`this`、模块、异步）
- [React](../react/00-roadmap.md)：组件 Props 泛型、Hooks 类型、多态组件
- [Node.js](../node/00-roadmap.md)：Node 原生运行 TS（类型擦除）、ESM/CJS 双包、`exports` 字段

## 自检清单

- [ ] 能在 3 分钟内手写 `Pick / Omit / Exclude / ReturnType / Awaited`
- [ ] 能讲清分布式条件类型，并写出 `IsNever`、`IsUnion`
- [ ] 能讲清函数参数为什么逆变，以及 TS 的双变漏洞
- [ ] 能写出 `UnionToIntersection` 并解释原理
- [ ] 能独立配置一个 Node ESM 项目和一个 Vite 前端项目的 tsconfig，并解释每一项
- [ ] 能讲清一次大型项目类型检查提速或 JS → TS 迁移的做法
- [ ] 能讲清 TS 7.0 的变化与升级风险
