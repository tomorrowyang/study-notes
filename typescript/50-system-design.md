# TypeScript 系统设计 / 场景题

## 通用答题框架

TS 的系统设计题通常是**工程架构题**，而不是分布式系统题。回答框架：

1. 澄清目标与范围：项目规模、团队人数、技术栈、发布方式
2. 约束：类型检查耗时、构建耗时、兼容性、迁移成本、团队水平
3. 总体方案：目录结构、配置分层、类型来源、工具链
4. 关键路径：开发体验、CI 流程、发布流程
5. 风险、度量与演进

---

### S1. 为 50+ 包的前端 Monorepo 设计 TypeScript 架构

- **难度**: senior
- **标签**: Monorepo, 构建性能

**澄清问题（面试时先问）**

- 包的类型：应用、内部组件库、工具库、需要发布到 npm 的包各有多少？
- 当前痛点：类型检查慢？编辑器卡？包之间互相引用混乱？
- 构建工具：Vite / Next.js / Rspack？包管理器是 pnpm 吗？
- 是否有非 TS 生态的约束（Vue、Svelte 需要 TS 6 的编译器 API）？

**参考方案**

```text
repo/
├─ tsconfig.base.json        # 统一严格选项与新默认值
├─ tsconfig.json             # 只包含 references，供 tsc -b 使用
├─ apps/
│  ├─ web/                   # Vite 应用：moduleResolution bundler，noEmit
│  └─ admin/
├─ packages/
│  ├─ ui/                    # 组件库：composite + isolatedDeclarations
│  ├─ utils/
│  ├─ api-types/             # 由 OpenAPI 生成的接口类型
│  └─ tsconfig/              # 共享的配置预设（web、node、library）
└─ tooling/
```

- **配置分层**：`base`（严格选项）→ 预设（`web` / `node` / `library`）→ 包级配置（只写 `include` 和 `references`）
- **包间引用**：内部包的 `exports` 使用 `customConditions: ["source"]` 指向源码，开发时无需预构建；需要发布的包另外产出 `dist` 与声明文件
- **类型检查**：
  - 编辑器：只加载当前包
  - CI：`tsc -b`，基于项目引用增量检查；结合任务编排工具（Turborepo / Nx）只检查受影响的包，并缓存 `.tsbuildinfo`
  - 升级 TS 7 后按 CI 机器的核数设置 `--checkers` 和 `--builders`
- **声明生成**：共享包开启 `isolatedDeclarations`，用 Oxc / tsdown 并行生成声明，不阻塞下游

**关键指标**

- 全量与增量的类型检查耗时、CI 中受影响包的检查耗时
- 编辑器打开项目到出现首个诊断的时间
- 包之间的依赖深度（越深越难并行）

**口述模板（2–3 分钟）**

1. 先问清包的类型和当前痛点，确认是否有 Vue 等依赖 TS 6 API 的生态。
2. 配置分三层：base、按场景的预设、包级配置，统一严格选项并显式写出 TS 6 改变的默认值。
3. 包间用 workspace 依赖加 customConditions 直连源码，发布包单独构建。
4. 检查用项目引用 + 增量 + 受影响范围计算，CI 缓存 tsbuildinfo。
5. 共享包开启 isolatedDeclarations 并行生成声明；升级 TS 7 后调整并行参数。
6. 最后用检查耗时和编辑器响应时间作为度量，持续优化。

**扩展追问**

- 包之间出现循环依赖怎么办？→ 项目引用不允许循环；把公共部分抽到更底层的包，并用 lint（如 dependency-cruiser）在 CI 中阻止循环依赖。
- 为什么不直接用 `paths` 别名？→ 包边界不清晰，运行时也要配置别名，且所有代码都在一个检查范围内，无法增量。

---

### S2. 设计前后端端到端类型安全的 API 层

- **难度**: senior
- **标签**: 端到端类型, 契约

**澄清问题（面试时先问）**

- 后端技术栈是 Node（TS）还是 Java / Go？
- 接口是否对外开放？有多少个客户端（Web、App、小程序）？
- 前后端是否在同一个仓库，是否同步发布？

**参考方案**

```text
方案 A：全栈 TS 且同仓库
  后端路由（Zod schema 定义输入输出）
      │  类型直接被前端导入（tRPC / Hono RPC / ts-rest）
      ▼
  前端调用：参数和返回值自动推导，无需代码生成

方案 B：后端为 Java / Go 或接口对外开放
  后端代码注解 ──> OpenAPI 文档（CI 产出，版本化）
      │
      ├─> openapi-typescript：生成类型
      ├─> orval / hey-api：生成请求函数 + TanStack Query hooks + Zod 校验器
      ▼
  前端 api-types 包（提交到仓库，变更走代码审查）
```

**关键设计**

1. **单一数据源**：类型来自后端定义或 OpenAPI 文档，禁止手写接口类型
2. **契约检查进入 CI**：后端变更后重新生成类型，若前端编译失败则说明有破坏性变更；可额外用 oasdiff 等工具检测 OpenAPI 的不兼容变更
3. **运行时校验分级**：后端必须校验输入；前端对关键接口校验响应，非关键接口在开发和测试环境校验、生产环境抽样上报
4. **兼容策略**：后端只做向后兼容的变更（新增可选字段）；删除字段先废弃（OpenAPI 的 `deprecated`），前端生成的类型会带上 `@deprecated` 提示
5. **错误类型**：统一的错误响应结构（`code` 的联合类型），前端用可辨识联合处理

**口述模板（2–3 分钟）**

1. 先确认后端技术栈和是否同仓库同发布，这决定用 RPC 推导还是 OpenAPI 生成。
2. 全栈 TS 用 tRPC 或类似方案，类型零生成；跨语言用 OpenAPI 生成类型、请求函数和校验器。
3. 核心原则是单一数据源，类型生成进入 CI，破坏性变更在合并前暴露。
4. 类型不等于运行时保证，按接口的重要程度分级做运行时校验。
5. 独立发布时，靠向后兼容的变更规则和废弃流程保证线上安全。

**扩展追问**

- tRPC 的缺点？→ 前后端技术栈强耦合；不适合对外开放的 API；大型路由表会增加类型检查耗时。
- 生成的代码要不要提交到仓库？→ 建议提交，便于审查接口变化；同时 CI 校验「重新生成后没有差异」。

---

### S3. 设计一个组件库的类型 API 与发布方案

- **难度**: senior
- **标签**: 组件库, 库发布

**澄清问题（面试时先问）**

- 使用方的 TS 版本和模块解析方式是否统一？是否需要支持 CJS？
- 是否需要多态组件（`as` 属性）、泛型组件（Table、Select）？
- 主题、图标等是否允许使用方扩展？

**参考方案**

- **Props 设计**：
  - 继承原生属性：`ComponentPropsWithoutRef<'button'>`，用 `Omit` 去掉冲突的键
  - 互斥属性用可辨识联合（`{ href: string } | { onClick: () => void }`）
  - 泛型组件：`Table<T>` 从 `data` 推断行类型，`columns` 的 `dataIndex` 约束为 `keyof T`
  - 多态组件只在少数基础组件（Box、Text）中使用，避免拖慢编辑器
- **可扩展点**：主题 token、图标名称通过 `interface` 暴露，使用方可通过模块扩充添加
- **类型稳定性**：只导出公开 API 的类型；导出函数和组件显式标注；开启 `isolatedDeclarations`
- **发布**：
  - 只发 ESM（Node 22+ 支持 `require(esm)`），或用 tsdown 产出双格式与对应的 `.d.ts` / `.d.cts`
  - `exports` 中 `types` 条件放在最前；`sideEffects` 标注样式文件
  - 用 `attw` 和 `publint` 在 CI 中检查产物
  - 声明文件中不能引用 devDependencies 里的类型
- **质量**：类型测试（`expectTypeOf`）覆盖泛型组件的推断；为 TS 的最低支持版本跑一次检查

**口述模板（2–3 分钟）**

1. 先问清使用方的 TS 版本、模块格式和扩展需求。
2. Props 继承原生属性，互斥属性用可辨识联合，泛型组件从数据推断。
3. 扩展点用 interface 暴露，允许使用方模块扩充。
4. 公开 API 显式标注并开启 isolatedDeclarations，保证类型稳定和声明生成快。
5. 发布时 exports 的 types 条件放最前，用 attw 检查；用类型测试防止推断回归。

**扩展追问**

- 如何避免类型的破坏性变更？→ 用 api-extractor 生成公开 API 报告，变更时在代码审查中可见。
- 泛型组件在 React 19 之前怎么处理 ref？→ 用类型断言重新声明 `forwardRef` 的返回类型，或单独暴露 `ref` 属性。

---

### S4. 50 万行 JS 项目迁移到严格 TS 的方案

- **难度**: senior
- **标签**: 迁移, 治理

**澄清问题（面试时先问）**

- 业务能否接受迁移期间的开发速度下降？有没有专门的人力？
- 构建工具是否支持 TS？测试覆盖率如何？
- 有哪些高频线上问题（空值错误、接口字段错误）？

**参考方案**

```text
阶段 0：基础设施（1–2 周）
  tsconfig（allowJs、strict: false）+ 构建支持 + CI 跑 tsc --noEmit + lint 规则

阶段 1：边界优先（1–2 个月）
  接口类型（OpenAPI 生成）→ 全局状态 → 公共组件 Props → 工具函数
  新代码必须写 TS

阶段 2：按依赖图自底向上迁移（持续）
  叶子模块 → 业务模块 → 页面
  用 ts-migrate 类工具批量改名并插入 any / @ts-expect-error，再逐步清理

阶段 3：收紧严格选项
  noImplicitAny → strictNullChecks → strict → noUncheckedIndexedAccess
  每个选项：统计报错数 → codemod 批量修复 → 手工修复剩余 → 开启
```

**关键设计**

- **度量**：TS 文件占比、`type-coverage`、`@ts-expect-error` 数量、各严格选项的报错数
- **棘轮策略**：CI 中各项指标只能变好不能变差
- **严格度分区**：新模块目录使用更严格的 tsconfig
- **避免大爆炸**：每次合并的改动量可控，按模块分配负责人
- **收益验证**：对比迁移前后的空值类和接口字段类线上错误数量

**口述模板（2–3 分钟）**

1. 先确认人力、测试覆盖和主要线上问题，用问题驱动迁移优先级。
2. 先建设基础设施：allowJs、CI 检查、新代码必须用 TS。
3. 从边界开始：接口类型、全局状态、公共组件，收益最大。
4. 再按依赖图自底向上迁移，工具批量改名后逐步清理 any。
5. 逐个开启严格选项，每个选项都是「统计 → 批量修复 → 开启」。
6. 用棘轮指标保证不回退，用线上错误数据证明收益。

**扩展追问**

- 迁移过程中怎么处理大量的 `any`？→ 允许存在但必须可统计（统一使用带说明的 `@ts-expect-error` 或特定的 `TODO_ANY` 类型别名），按模块逐步清理。
- 如何防止迁移后又被 `as` 断言绕开？→ lint 限制断言写法，审查新增的断言，并在边界使用运行时校验。

---

### S5. 推动全公司从 TS 5.x 升级到 TS 7

- **难度**: senior
- **标签**: 升级, 工程效能

**澄清问题（面试时先问）**

- 有多少仓库和项目？各自使用的 TS 版本和构建工具？
- 有多少项目使用 Vue / Svelte / Angular，或依赖编译器 API 的工具（类型感知 lint、自定义 transformer、文档生成）？
- 是否有统一的 tsconfig 预设和脚手架？

**参考方案**

```text
1. 盘点：扫描所有仓库的 TS 版本、tsconfig 选项、依赖编译器 API 的工具
2. 统一预设：在共享 tsconfig 包中显式写出 6.0 改变的默认值（types、rootDir、strict 等）
3. 升级 6.0：处理废弃项（moduleResolution node、baseUrl、es5、esModuleInterop false 等）
   └─ 可以短期使用 ignoreDeprecations: "6.0"，但必须登记并设定清理期限
4. 对齐 7.0：开启 stableTypeOrdering，修复因推断顺序产生的错误，去掉 ignoreDeprecations
5. 分层切换 7.0：
   ├─ CI 检查与构建先切换（收益最大）
   ├─ 编辑器：React / Node 项目切换 TS 7 语言服务；Vue / Svelte 项目暂留 6.0
   └─ lint 与工具：通过 npm 别名让 typescript 指向 @typescript/typescript6，等待 7.1 API
6. 验证：对比 6.0 与 7.0 的错误列表与 .d.ts 输出；固定 --checkers 数值
```

**关键设计**

- **自动化**：编写 codemod 自动修改 tsconfig 和废弃语法（`assert` → `with`、`module` → `namespace`）
- **灰度**：先在几个代表性项目试点，记录问题清单，形成升级手册
- **度量收益**：CI 类型检查耗时、编辑器首次诊断时间、合并队列耗时
- **风险**：依赖 API 的工具链断档；`types` 默认为空导致全局类型丢失；`rootDir` 变化导致输出目录变化

**口述模板（2–3 分钟）**

1. 先盘点仓库、TS 版本、构建工具和依赖编译器 API 的工具。
2. 在共享预设中显式写出新默认值，再统一升级 6.0 并清理废弃项。
3. 用 stableTypeOrdering 对齐 7.0 的行为，修复推断顺序问题。
4. 分层切换：CI 先切，编辑器按生态切，lint 等工具通过别名继续使用 6.0。
5. 试点、写升级手册、用 codemod 批量处理，并用 CI 耗时等指标展示收益。

**扩展追问**

- 如何说服业务团队配合升级？→ 用试点项目的 CI 耗时和编辑器响应数据说明收益，并提供自动化工具降低成本。
- 7.0 和 6.0 结果不一致怎么办？→ 先确认是否开启了 stableTypeOrdering；仍不一致时补充显式类型，并向官方反馈。

---

### S6. 设计类型安全的国际化（i18n）方案

- **难度**: mid
- **标签**: 模板字面量, 实战

**澄清问题（面试时先问）**

- 文案数量级？是否按页面或模块拆分加载？
- 是否需要插值参数、复数、富文本？
- 翻译是否由外部平台管理？

**参考方案**

```ts
// 以默认语言的文案文件作为类型来源
const zh = {
  common: { ok: '确定', cancel: '取消' },
  order: { total: '共 {count} 件，合计 {amount} 元' },
} as const

type Messages = typeof zh
type Key = Path<Messages>                       // 'common.ok' | 'order.total' | ...

// 从文案中提取插值参数
type Vars<S extends string> =
  S extends `${string}{${infer V}}${infer Rest}` ? V | Vars<Rest> : never
type ParamsOf<K extends Key> = Vars<PathValue<Messages, K> & string>

declare function t<K extends Key>(
  key: K,
  ...args: [ParamsOf<K>] extends [never] ? [] : [Record<ParamsOf<K>, string | number>]
): string

t('common.ok')
t('order.total', { count: 2, amount: 10 })
// t('order.total')                // 报错：缺少参数
```

（`Path` / `PathValue` 见手写题 H8）

**关键设计**

- **类型来源**：默认语言文件是唯一的类型来源，其他语言文件用 `satisfies DeepPartial<Messages>` 或严格的结构约束检查缺失的键
- **按需加载**：按命名空间拆分文件，类型仍然由全部命名空间合并得到
- **性能**：文案达到数千条时，`Path` 生成的大联合会拖慢编辑器。可以按命名空间拆分（`t('order:total')` 先约束命名空间再约束键），或改为代码生成扁平的键联合
- **CI**：检查各语言的键是否齐全、是否存在未使用的键

**口述模板（2–3 分钟）**

1. 以默认语言文件作为唯一类型来源，用路径类型生成键的联合。
2. 用模板字面量从文案中提取插值参数，用条件元组决定是否需要第二个参数。
3. 其他语言用 satisfies 检查结构，CI 检查缺失和未使用的键。
4. 文案很多时按命名空间拆分或改为代码生成，避免巨大联合拖慢编辑器。

**扩展追问**

- 复数规则怎么处理？→ 使用 ICU 消息格式，参数类型中增加 `count: number`；复杂格式交给 i18n 库的解析器，类型只约束参数名。

---

### S7. 设计一个可扩展的插件系统，让插件能安全地扩展类型

- **难度**: senior
- **标签**: 插件, 声明合并, 架构

**澄清问题（面试时先问）**

- 插件是同仓库内的模块，还是第三方 npm 包？
- 插件需要扩展什么：配置项、上下文对象、事件、命令？
- 插件之间是否存在依赖关系？

**参考方案**

两种类型扩展方式：

```ts
// 方式一：声明合并（Vite、Vue、Fastify 的做法）——全局生效
export interface PluginRegistry {}           // 宿主暴露空接口

declare module 'my-host' {
  interface PluginRegistry {
    auth: { user: User | null; login(): Promise<void> }
  }
}

// 方式二：泛型累积（Hono、tRPC、Elysia 的做法）——局部生效、可推导
const app = createApp()
  .use(authPlugin)        // 返回 App<{ auth: AuthCtx }>
  .use(dbPlugin)          // 返回 App<{ auth: AuthCtx; db: DbCtx }>
app.handle(ctx => ctx.auth.user)
```

| 维度 | 声明合并 | 泛型累积 |
|------|---------|---------|
| 作用范围 | 全局 | 局部，随调用链 |
| 与运行时一致性 | 不保证（声明了但没注册插件也不报错） | 由调用链保证 |
| 类型检查开销 | 低 | 链越长越高 |
| 适用 | 第三方生态、配置扩展 | 应用内部组合 |

**关键设计**

- 插件定义使用 `definePlugin` 辅助函数，配合 `const` 类型参数保留插件名与配置的字面量
- 插件依赖：插件声明 `requires: ['auth']`，类型上要求宿主上下文已包含对应键
- 运行时也要校验插件的依赖和注册顺序，不能只靠类型

**口述模板（2–3 分钟）**

1. 先问插件来源和扩展内容。
2. 介绍两种方式：声明合并全局扩展，泛型累积局部推导。
3. 对比它们在作用范围、与运行时一致性和性能上的差异。
4. 第三方生态的配置扩展用声明合并，应用内部组合用泛型累积。
5. 插件依赖在类型和运行时两个层面同时校验。

**扩展追问**

- 泛型累积的链很长导致编辑器变慢怎么办？→ 在中间步骤显式标注类型并导出类型别名，切断推断链。

---

### S8. 大型仓库的类型检查耗时从 8 分钟降到 1 分钟以内

- **难度**: senior
- **标签**: 性能, CI

**澄清问题（面试时先问）**

- 8 分钟是全量检查还是增量检查？在 CI 还是本地？
- 当前 TS 版本？是否使用项目引用？
- CI 机器的核数和内存？

**参考方案**

```text
1. 度量：tsc --extendedDiagnostics 记录 Files / Types / Instantiations / Check time
2. 定位：--generateTrace + analyze-trace 找出最耗时的文件与类型
3. 快速收益：
   ├─ 收紧 include，排除产物与生成代码中不需要检查的部分
   ├─ skipLibCheck: true；types 显式列出
   └─ 修复热点类型：交叉改 interface、拆分巨型联合、给复杂导出函数加返回类型、
      替换或限制拖慢检查的库类型（某些 CSS-in-JS、表单库的复杂泛型）
4. 结构优化：
   ├─ 项目引用拆分 + 增量（CI 缓存 .tsbuildinfo）
   └─ 只检查受影响的项目（基于 git diff 与依赖图）
5. 升级 TS 7：全量检查快 8–12 倍；根据 CI 核数设置 --checkers / --builders
6. 防回退：在 CI 中记录检查耗时与 Instantiations，超过阈值报警
```

**关键设计**

- 先度量再优化，用 trace 数据而不是猜测
- 优化分三层：配置（立即生效）→ 类型写法（定点修复）→ 架构与工具（长期）
- 性能指标要持续监控，否则会因为新增复杂类型而回退

**口述模板（2–3 分钟）**

1. 先确认是全量还是增量、在哪里慢，并用 extendedDiagnostics 记录基线。
2. 用 generateTrace 找到热点文件和类型，定点修复。
3. 配置层面收紧 include、跳过库检查、显式声明 types。
4. 架构层面用项目引用、增量缓存和受影响范围检查。
5. 升级 TS 7 带来数量级的提升，再根据机器配置调整并行参数。
6. 最后把耗时和实例化次数纳入 CI 监控，防止回退。

**扩展追问**

- 并行检查器数量越多越好吗？→ 不是。每个检查器会重复部分工作并占用内存；`--checkers` 与 `--builders` 是相乘关系，要根据核数和内存实测。
