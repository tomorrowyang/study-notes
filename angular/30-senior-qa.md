# Angular 高级 Q&A

> 目标：能讲清 Ivy、变更检测、Signals、依赖注入、水合的底层实现，能排查大型应用的性能问题，能设计应用架构、选择状态管理方案、规划版本升级和迁移。

---

## 一、底层原理

### Q1. Ivy 编译器把模板编译成了什么？

- **难度**: 高级
- **标签**: Ivy, 编译

**答案**

Ivy（v9 起默认）把每个组件编译成**带静态定义的类**（`ɵcmp`），模板变成一个**模板函数**，分为两个阶段：

```ts
// 模板：<p>{{ name() }}</p><button (click)="inc()">+</button>
function Template(rf: RenderFlags, ctx: Counter) {
  if (rf & RenderFlags.Create) {          // 创建阶段：只执行一次
    ɵɵelementStart(0, 'p');
    ɵɵtext(1);
    ɵɵelementEnd();
    ɵɵelementStart(2, 'button');
    ɵɵlistener('click', () => ctx.inc());
    ɵɵtext(3, '+');
    ɵɵelementEnd();
  }
  if (rf & RenderFlags.Update) {          // 更新阶段：每次检查执行
    ɵɵadvance(1);
    ɵɵtextInterpolate(ctx.name());        // 与上一次的值比较，不同才更新 DOM
  }
}
```

**设计特点**：

- **局部性（Locality）**：每个组件只依赖自己的元数据编译，便于增量编译和库的独立发布
- **可摇树**：运行时功能以指令函数的形式按需引用，没用到的不会打包
- **增量 DOM 思想**：没有虚拟 DOM，直接用指令创建和更新 DOM；绑定值保存在数组（LView）中逐个比较
- 新控制流（`@if` / `@for`）由编译器直接生成指令，不需要结构指令的运行时开销

**口述模板（30–60 秒）**

1. 一句话结论：Ivy 把模板编译成分为创建和更新两个阶段的模板函数。
2. 关键机制：创建阶段执行一次生成 DOM，更新阶段逐个比较绑定值并更新 DOM。
3. 对比或注意点：没有虚拟 DOM，绑定值存在数组中按索引比较。
4. 可追问时补充：局部性让每个组件可以独立编译，指令函数可以被摇树。

**常见追问**

- 为什么 Angular 不用虚拟 DOM？→ 模板是静态可分析的，编译器知道哪些地方是动态绑定，只需要比较这些绑定值，内存和比较的开销都更小。

---

### Q2. 变更检测在运行时内部是怎么遍历的？

- **难度**: 高级
- **标签**: 变更检测, 原理

**答案**

**核心数据结构**：

- **TView**：每种组件模板一个，保存静态信息（节点结构、绑定数量、指令）
- **LView**：每个组件**实例**一个数组，保存运行时状态（DOM 节点引用、绑定的旧值、上下文、子视图、标志位）

**遍历过程**（`detectChanges` → `refreshView`）：

1. 检查视图的标志位：`Dirty`、`CheckAlways`（Eager 策略）、`RefreshView`、`HasChildViewsToRefresh`
2. 需要刷新的视图：执行模板函数的更新阶段，执行生命周期钩子
3. 递归处理嵌入视图（`@if` / `@for` 生成的）和子组件视图

**Signals 带来的变化（「局部变更检测」）**：

- 每个组件模板是一个**响应式消费者**（reactive consumer）
- 模板读取的信号变化时，只把**该视图**标记为 `RefreshView`，祖先标记为 `HasChildViewsToRefresh`（而不是标记为 Dirty）
- 遍历时，祖先的 `OnPush` 视图如果本身不脏，**只是路过**而不执行自己的模板函数，直接到达需要刷新的子视图

这让 Signals + OnPush 的应用可以做到**只刷新真正变化的组件**。

**口述模板（30–60 秒）**

1. 一句话结论：变更检测按照 LView 树递归遍历，根据标志位决定是否执行模板的更新阶段。
2. 关键机制：TView 保存模板的静态信息，LView 保存每个实例的绑定值和标志位。
3. 对比或注意点：Signal 变化只把读取它的视图标记为需要刷新，祖先只是路过不重新执行。
4. 可追问时补充：所以 Signals 加 OnPush 可以只刷新真正变化的组件。

**常见追问**

- `markForCheck` 和信号通知的区别？→ `markForCheck` 把自己和所有祖先都标记为 Dirty，祖先的模板也会重新执行；信号通知只标记读取它的视图。

---

### Q3. Zoneless 的变更检测调度器是怎么工作的？

- **难度**: 高级
- **标签**: Zoneless, 调度

**答案**

**调度器**（`ChangeDetectionScheduler`）：

1. 收到通知（信号更新、`markForCheck`、事件监听器、`setInput` 等）
2. 如果还没有安排过，**安排一次**变更检测（同一轮中的多次通知会合并）
3. 调度时机：使用 `setTimeout` 和 `requestAnimationFrame` 的**竞速**（哪个先到就用哪个），保证在下一帧渲染前完成，同时在后台标签页中也能执行
4. 执行 `ApplicationRef.tick()`：刷新视图、执行 `afterRender` 钩子

**和 Zone.js 的对比**：

| 方面 | Zone.js | Zoneless |
|------|---------|----------|
| 触发 | 任意异步任务完成 | 显式通知 |
| 范围 | 往往整个应用 | 只检查被标记的视图 |
| SSR 稳定性判断 | Zone 中没有挂起的任务 | `PendingTasks` 服务显式跟踪 |

**SSR 中的挑战**：服务端需要知道「应用何时稳定」才能序列化 HTML。Zoneless 下 HttpClient、Router、`resource` 等会自动注册**挂起任务**；自定义的异步操作需要用 `PendingTasks` 告诉 Angular。

**口述模板（30–60 秒）**

1. 一句话结论：Zoneless 的调度器收到通知后安排一次变更检测，多次通知合并。
2. 关键机制：用 setTimeout 和 requestAnimationFrame 竞速，保证在下一帧前执行。
3. 对比或注意点：只检查被标记的视图，而 Zone.js 往往检查整个应用。
4. 可追问时补充：SSR 中用 PendingTasks 显式跟踪异步任务来判断应用是否稳定。

**常见追问**

- 自定义的异步数据加载在 SSR 中没有被渲染出来，为什么？→ 服务端没有等待它完成就序列化了 HTML；用 `PendingTasks.run()` 包裹，或使用 `resource`。

---

### Q4. 依赖注入在运行时是怎么快速查找的？

- **难度**: 高级
- **标签**: 依赖注入, 原理

**答案**

**元素注入器的查找优化**：

- 每个节点的注入器信息存储在 LView 中，并用一个**布隆过滤器（Bloom Filter）**记录该节点上提供了哪些令牌（令牌被哈希到位数组中）
- 向上查找时，先检查布隆过滤器：**如果过滤器说「没有」，一定没有**，直接跳过该节点；说「可能有」时才真正查找
- 这让沿 DOM 结构向上的查找在大部分节点上是 O(1) 的跳过

**环境注入器**：基于 `Map` 的 `R3Injector`，按令牌查找记录，懒创建实例并缓存。

**`inject()` 的实现**：依赖一个全局的「当前注入上下文」。构造函数、字段初始化执行时，框架设置当前的注入器；`inject()` 读取它。这也是为什么 `inject()` 只能在注入上下文中调用。

**口述模板（30–60 秒）**

1. 一句话结论：元素注入器用布隆过滤器快速判断节点上是否可能提供了某个令牌。
2. 关键机制：过滤器说没有就一定没有，可以快速跳过大部分节点。
3. 对比或注意点：环境注入器基于 Map 查找，实例懒创建并缓存。
4. 可追问时补充：inject() 依赖全局的当前注入上下文，所以只能在构造阶段调用。

**常见追问**

- 怎么在注入上下文之外使用 `inject`？→ `runInInjectionContext(injector, () => inject(X))`，或者提前在构造时注入并保存。

---

### Q5. 水合（Hydration）是怎么把服务端 DOM 和客户端组件对应起来的？

- **难度**: 高级
- **标签**: 水合, 原理

**答案**

**服务端**：

- 渲染完成后，为每个组件收集**水合注解**：嵌入视图（`@if` / `@for`）的数量和模板、需要跳过的节点、投影信息等
- 注解以 `ngh` 属性（指向页面中序列化的 JSON 数据）写入宿主元素
- HTTP 传输缓存、事件重放所需的信息也被序列化

**客户端**：

- 创建阶段的指令（`ɵɵelementStart` 等）不再创建新节点，而是**按照注解在现有 DOM 中定位并认领节点**
- 绑定、事件监听正常附加
- 如果找到的节点与预期不一致 → **水合不匹配错误**（开发模式下给出详细信息）

**增量水合**：`@defer (hydrate on viewport)` 的块在服务端完整渲染，客户端**不加载该块的代码**，直到触发条件满足；期间的交互通过事件重放（基于 JSAction 的事件委托）在水合后补上。

**`ngSkipHydration`**：对直接操作 DOM、无法水合的组件跳过水合（退化为重新渲染），是权宜之计。

**口述模板（30–60 秒）**

1. 一句话结论：服务端为组件生成水合注解，客户端按注解在已有 DOM 中认领节点而不是重新创建。
2. 关键机制：注解记录嵌入视图的数量和结构，客户端的创建指令据此定位节点。
3. 对比或注意点：DOM 不一致时会报水合不匹配错误。
4. 可追问时补充：增量水合在触发前不加载代码，期间的交互通过事件重放补上。

**常见追问**

- 浏览器扩展修改了 DOM 导致水合错误怎么办？→ 通常是扩展在 `<body>` 中注入内容；尽量让应用根元素不受影响，必要时对受影响的组件使用 `ngSkipHydration`。

---

## 二、性能

### Q6. 一个大型 Angular 应用交互卡顿，你怎么排查？

- **难度**: 高级
- **标签**: 性能, 排查

**答案**

**1. 定位**：

- **Angular DevTools 的 Profiler**：记录交互，查看每次变更检测的耗时、触发来源、**哪些组件被检查了多少次**
- **Chrome Performance 面板**：Long Tasks、脚本执行、样式计算、布局；v20+ 的 Angular 在 Performance 面板中有自定义轨道（以官方为准）
- 看是否是**变更检测太频繁 / 范围太大**，还是**单次渲染太重**，还是**非 Angular 的问题**（布局抖动、大图）

**2. 常见原因与处理**：

| 现象 | 原因 | 处理 |
|------|------|------|
| 每次事件整个应用被检查 | Eager 策略 + Zone.js | OnPush、Signals、Zoneless |
| 高频事件（scroll、mousemove）卡顿 | 每次都触发检测 | `runOutsideAngular`、节流、Signals |
| 模板中调用方法 | 每次检测都执行 | `computed`、纯管道 |
| 大列表 | DOM 节点多、`track` 不当 | `track` 唯一 ID、虚拟滚动、分页 |
| 第三方库触发检测 | 库的定时器、事件 | 在 Zone 外初始化 |
| 大量订阅 | 泄漏导致重复执行 | 检查取消订阅 |

**3. 验证**：Profiler 对比优化前后的检测次数和耗时；INP 指标。

**口述模板（30–60 秒）**

1. 一句话结论：先用 Angular DevTools 的 Profiler 定位是检测太频繁、范围太大，还是单次渲染太重。
2. 关键机制：Profiler 显示每次变更检测的触发来源和每个组件的耗时。
3. 对比或注意点：常见原因是 Eager 策略、高频事件、模板中调用方法和大列表。
4. 可追问时补充：处理手段是 OnPush、Signals、Zoneless、computed、虚拟滚动，最后用 INP 验证。

**常见追问**

- 怎么知道是哪个异步任务触发了变更检测？→ Profiler 中显示触发来源；Zone.js 应用可以在 `NgZone.onUnstable` 中打断点查看调用栈。

---

### Q7. 怎么优化 Angular 应用的包体积和首屏加载？

- **难度**: 高级
- **标签**: 性能, 加载

**答案**

**分析**：`ng build --stats-json` + esbuild 分析工具（或 source-map-explorer），查看初始包的组成。

**优化**：

| 手段 | 说明 |
|------|------|
| 路由懒加载 | 每个页面独立代码块 |
| `@defer` | 页面内的重组件延迟加载 |
| `injectAsync`（v22） | 大型服务按需加载 |
| 摇树 | `providedIn: 'root'`；避免导入整个库（lodash → lodash-es 或原生） |
| 移除 Zone.js | Zoneless 减少几十 KB |
| 预算 | `angular.json` 中的 `budgets`，超出时构建失败 |
| 图片 | `NgOptimizedImage`：自动 `srcset`、懒加载、LCP 图片优先级提示、尺寸检查 |
| SSR / SSG | 首屏 HTML 直出；增量水合减少水合的 JS |
| 字体 | 内联关键字体 CSS |

**口述模板（30–60 秒）**

1. 一句话结论：先分析初始包的组成，再用懒加载、@defer、摇树和移除 Zone.js 减小体积。
2. 关键机制：路由懒加载按页面拆分，@defer 按区块拆分，injectAsync 按服务拆分。
3. 对比或注意点：用 budgets 防止体积回退。
4. 可追问时补充：NgOptimizedImage 优化 LCP 图片，SSR 加增量水合优化首屏。

**常见追问**

- `NgOptimizedImage` 做了什么？→ 强制宽高（防 CLS）、生成 `srcset`、默认懒加载、为 `priority` 图片添加 `fetchpriority` 和预连接提示，并在开发模式下警告常见问题。

---

### Q8. 怎么排查 Angular 应用的内存泄漏？

- **难度**: 高级
- **标签**: 内存, 排查

**答案**

**排查**：

1. 重复执行「进入页面 → 离开页面」，Chrome Memory 面板拍堆快照对比
2. 搜索组件类名，看已销毁组件的实例数量是否持续增长
3. 查看保留路径（Retainers），找到谁持有引用

**常见原因**：

| 原因 | 处理 |
|------|------|
| 未取消的订阅（Store、interval、valueChanges、全局事件流） | `takeUntilDestroyed`、`toSignal` |
| 全局事件监听（window resize、document click） | 在 `DestroyRef.onDestroy` 中移除；使用 `host` 绑定或 Renderer2 的 `listen`（返回的清理函数） |
| 第三方库实例（图表、编辑器）未销毁 | 销毁时调用其 `destroy` |
| 根级服务中缓存了组件引用或大数据 | 只缓存必要的数据，设置上限 |
| `RouteReuseStrategy` 缓存的路由 | 设置上限，v22 的 `destroyDetachedRouteHandle` 用于正确销毁 |
| 路由级的注入器 | v22 实验特性 `withExperimentalAutoCleanupInjectors` 自动销毁不活跃路由的注入器 |
| 闭包中引用的 DOM（Detached 节点） | 清理引用 |

**口述模板（30–60 秒）**

1. 一句话结论：通过反复进出页面并对比堆快照，查找已销毁组件的残留实例。
2. 关键机制：查看保留路径找到持有引用的对象。
3. 对比或注意点：最常见的是未取消的订阅、全局事件监听和未销毁的第三方库实例。
4. 可追问时补充：路由复用策略缓存的组件需要正确销毁，v22 提供了相应的 API。

**常见追问**

- 根级服务中的订阅需要取消吗？→ 根级服务与应用同生命周期，订阅一般不会泄漏；但如果每次组件调用服务方法都创建新的订阅而不取消，就会累积。

---

### Q9. 大数据量的表格怎么优化？

- **难度**: 高级
- **标签**: 性能, 列表

**答案**

| 手段 | 说明 |
|------|------|
| **虚拟滚动** | CDK `cdk-virtual-scroll-viewport`，只渲染可见行 |
| **正确的 `track`** | 用唯一 ID，数据刷新时复用 DOM |
| **OnPush + 不可变数据 / Signals** | 只更新变化的行；行组件读取自己的信号 |
| 服务端分页 / 排序 / 筛选 | 前端只处理当前页 |
| 单元格中避免方法调用 | 用纯管道或预先计算 |
| 高频更新（行情） | 批量合并更新（每帧一次）；在 Zone 外接收 WebSocket 数据；行级信号 |
| 复杂单元格 | 进入视口后才渲染（`@defer on viewport`） |
| 计算密集的排序 / 聚合 | Web Worker |

**口述模板（30–60 秒）**

1. 一句话结论：大表格优化的核心是虚拟滚动、正确的 track 和细粒度更新。
2. 关键机制：虚拟滚动只渲染可见行，track 唯一 ID 让刷新时复用 DOM。
3. 对比或注意点：高频更新要批量合并，每帧最多更新一次。
4. 可追问时补充：大量计算放到 Web Worker，能放服务端的分页排序放服务端。

**常见追问**

- 虚拟滚动的行高不固定怎么办？→ CDK 的实验性自动尺寸策略，或预估高度 + 测量后修正；也可以考虑分页。

---

## 三、架构

### Q10. 怎么设计一个大型 Angular 应用的目录和模块结构？

- **难度**: 高级
- **标签**: 架构

**答案**

**按领域（业务）划分**，而不是按技术类型（components / services）划分：

```text
apps/
  admin/                     # 应用壳：路由、布局、全局配置
libs/
  orders/
    feature-list/            # 页面级（智能组件、路由）
    feature-detail/
    data-access/             # API、Store
    ui/                      # 展示组件（纯输入输出）
    util/                    # 工具函数
  shared/
    ui/                      # 设计系统组件
    data-access/             # 认证、用户上下文
    util/
```

**分层规则**（用 Nx 的模块边界规则强制）：

- `feature` → 可以依赖 `data-access`、`ui`、`util`
- `ui` → 只能依赖 `ui`、`util`（不能依赖 `data-access`）
- 领域之间不能直接互相依赖，通过 `shared` 或明确的公共接口

**组件分层**：

- **容器（智能）组件**：注入服务、管理状态、处理路由
- **展示组件**：只有输入输出，`OnPush`，易复用和测试

**口述模板（30–60 秒）**

1. 一句话结论：按业务领域划分，每个领域分为 feature、data-access、ui、util 四层。
2. 关键机制：用 Nx 的模块边界规则强制依赖方向。
3. 对比或注意点：领域之间不直接依赖，ui 层不能依赖数据访问层。
4. 可追问时补充：组件分为容器组件和展示组件，展示组件只有输入输出。

**常见追问**

- 为什么按领域划分？→ 一个功能的修改集中在一个目录；便于团队按领域分工；便于将来拆分（微前端）。

---

### Q11. 状态管理怎么选：服务 + Signals、NgRx Store、NgRx SignalStore？

- **难度**: 高级
- **标签**: 状态管理

**答案**

| 方案 | 特点 | 适合 |
|------|------|------|
| **服务 + Signals** | 最简单，无额外依赖 | 中小型应用、局部状态 |
| **NgRx SignalStore** | 基于 Signals 的轻量 Store；`withState`、`withComputed`、`withMethods`、`withEntities`，可组合扩展 | 大多数新项目的推荐 |
| **NgRx Store（Redux 模式）** | Action、Reducer、Selector、Effect；严格的单向数据流；DevTools 时间旅行 | 大型团队、复杂的跨功能交互、需要审计每一次状态变化 |
| TanStack Query（Angular 版） | 服务端状态缓存、失效、重试 | 以服务端数据为主的应用 |

**选型原则**：

- 区分**服务端状态**（缓存的 API 数据）和**客户端状态**（UI 状态）
- 状态应该放在**尽可能低的层级**（组件 → 功能 → 全局）
- 不要因为「大项目」就默认上 Redux 模式，样板代码有成本

**口述模板（30–60 秒）**

1. 一句话结论：简单场景用服务加 Signals，大多数新项目推荐 SignalStore，复杂的大团队场景用 NgRx Store。
2. 关键机制：SignalStore 基于 Signals，可以组合扩展；NgRx Store 是 Redux 模式，有严格的单向数据流。
3. 对比或注意点：要区分服务端状态和客户端状态，状态放在尽可能低的层级。
4. 可追问时补充：以服务端数据为主时可以考虑 TanStack Query。

**常见追问**

- NgRx Store 的优势在哪里？→ 所有状态变化都是显式的 Action，可追踪、可重放；Effect 隔离副作用；适合多人协作的复杂交互。

---

### Q12. NgRx 的 Selector 和 Effect 有哪些要点？

- **难度**: 高级
- **标签**: NgRx

**答案**

**Selector**：

- **记忆化**：输入不变时返回缓存的结果，保证引用稳定（OnPush 友好）
- 组合：`createSelector(selectA, selectB, (a, b) => ...)`
- 带参数的 selector 用工厂函数，注意每次调用工厂创建新的记忆化实例
- 组件中：`store.selectSignal(selectX)` 得到 Signal

**Effect**：

- 监听 Action，执行副作用（请求），派发新的 Action
- 选择合适的映射操作符：加载用 `switchMap`，保存用 `concatMap`，删除用 `mergeMap`，防止重复用 `exhaustMap`
- **错误必须在内部捕获**，否则 Effect 的流终止，之后的 Action 都不会被处理

```ts
loadOrders$ = createEffect(() =>
  inject(Actions).pipe(
    ofType(OrdersActions.load),
    switchMap(({ filter }) =>
      this.api.list(filter).pipe(
        map((orders) => OrdersActions.loadSuccess({ orders })),
        catchError((error) => of(OrdersActions.loadFailure({ error }))),
      ),
    ),
  ),
);
```

**口述模板（30–60 秒）**

1. 一句话结论：Selector 是记忆化的派生状态，Effect 隔离副作用。
2. 关键机制：Selector 输入不变时返回缓存结果，保证引用稳定。
3. 对比或注意点：Effect 中的错误必须在内部捕获，否则整个 Effect 终止。
4. 可追问时补充：根据业务语义选择 switchMap、concatMap、mergeMap 或 exhaustMap。

**常见追问**

- 实体集合怎么管理？→ `@ngrx/entity`（或 SignalStore 的 `withEntities`）以规范化的方式（ids + entities 字典）存储，提供增删改的适配器方法。

---

### Q13. 怎么设计一个 Angular 组件库？

- **难度**: 高级
- **标签**: 组件库, 设计系统

**答案**

**分层**：

- **行为层**：CDK / Angular Aria（焦点、键盘、浮层、无障碍）
- **组件层**：设计系统的组件
- **主题层**：设计令牌（CSS 变量）

**API 设计**：

- 信号式 `input` / `output` / `model`；`booleanAttribute` 等转换
- 支持 `ControlValueAccessor` / `FormValueControl`，接入表单
- 内容投影 + `ng-template` 提供定制点
- 复合组件通过父组件提供的服务协作（Tabs / Tab）
- `hostDirectives` 组合行为
- `OnPush`，兼容 Zoneless（不依赖 Zone.js 触发更新）
- 兼容 SSR（不在构造中访问浏览器 API）

**工程**：

- `ng-packagr` 构建（APF 格式），**次级入口点**（`@lib/ui/button`）便于摇树
- Storybook 文档、组件测试 Harness
- 版本策略：跟随 Angular 主版本，声明 peerDependencies 范围
- 提供 `ng add` 和 `ng update` 迁移脚本

**口述模板（30–60 秒）**

1. 一句话结论：组件库分为行为层、组件层和主题层。
2. 关键机制：行为基于 CDK 和 Angular Aria，主题用 CSS 变量。
3. 对比或注意点：组件要使用 OnPush、兼容 Zoneless 和 SSR，支持接入表单。
4. 可追问时补充：用次级入口点便于摇树，提供 Harness 和迁移脚本。

**常见追问**

- 为什么要次级入口点？→ 每个组件独立入口，使用方只导入需要的组件，也避免所有组件之间的循环依赖。

---

### Q14. Angular 应用怎么做微前端？

- **难度**: 高级
- **标签**: 微前端

**答案**

| 方案 | 说明 | 适合 |
|------|------|------|
| **Native Federation** | 基于 esbuild 和导入映射（import map）的模块联邦实现，与 application builder 兼容 | Angular 主应用 + Angular 子应用 |
| Module Federation 2.0 | 基于 Webpack / Rspack；v22 弃用了 Webpack 构建器，需要评估 | 已有 Webpack 方案的项目 |
| **Angular Elements** | 把组件打包为 Web Components，在任意框架中使用 | 跨框架嵌入 |
| qiankun / wujie / iframe | 运行时隔离 | 异构技术栈、强隔离 |
| Nx Monorepo + 构建时集成 | 不是运行时的微前端，但能解决大部分团队协作问题 | 同一团队可以统一发布 |

**Angular 特有的问题**：

- **多个 Angular 实例**：共享 `@angular/*` 依赖（单例），否则包体积翻倍、DI 和 Zone.js 冲突
- **版本对齐**：共享依赖要求主应用和子应用的 Angular 主版本一致
- **Zone.js**：多个应用都加载会冲突；统一迁移到 Zoneless 会简单很多
- 路由：主应用路由懒加载远程模块（`loadRemoteModule`）

**口述模板（30–60 秒）**

1. 一句话结论：Angular 应用之间推荐 Native Federation，跨框架嵌入用 Angular Elements，强隔离用 iframe 方案。
2. 关键机制：模块联邦共享 Angular 依赖，主应用路由懒加载远程模块。
3. 对比或注意点：共享依赖要求 Angular 主版本一致，多个 Zone.js 会冲突。
4. 可追问时补充：先评估 Monorepo 能否解决问题，微前端的运维成本很高。

**常见追问**

- 子应用升级 Angular 版本怎么办？→ 共享依赖时要求统一升级；无法统一时只能不共享（各自打包 Angular，体积增加）或改用隔离方案。

---

### Q15. 大型 Angular 应用的错误处理策略是什么？

- **难度**: 高级
- **标签**: 错误处理

**答案**

| 层次 | 手段 |
|------|------|
| HTTP | 拦截器：统一处理 401（刷新令牌 / 跳转登录）、403、5xx；重试幂等请求 |
| 全局 | 自定义 `ErrorHandler`：上报到监控（Sentry）、附带用户和路由上下文 |
| 未处理的 Promise | `provideBrowserGlobalErrorListeners()`（v20+）把 `window` 的错误和未处理的拒绝交给 ErrorHandler |
| 组件 | **`@boundary`**（v22 预告，2026 Q3 开发者预览）：模板中的错误边界，局部组件失败不影响整个页面 |
| 数据 | `resource` 的 `error()` 状态，组件展示错误和重试 |
| 路由 | 导航错误处理（`withNavigationErrorHandler`）、懒加载块加载失败（部署后旧版本的块不存在）→ 提示刷新 |

**块加载失败**：发布新版本后，用户还在旧页面，旧的代码块被删除 → 懒加载失败。处理：保留旧版本的产物一段时间；捕获 `ChunkLoadError` 后提示刷新；Service Worker 版本管理（`SwUpdate`）。

**口述模板（30–60 秒）**

1. 一句话结论：错误处理分为 HTTP 拦截器、全局 ErrorHandler、组件级和路由级几个层次。
2. 关键机制：拦截器统一处理认证和服务端错误，ErrorHandler 上报监控。
3. 对比或注意点：v22 预告的 @boundary 可以在模板中实现错误边界。
4. 可追问时补充：要处理部署后旧代码块不存在导致的懒加载失败。

**常见追问**

- Angular 之前没有错误边界，组件错误会怎样？→ 模板中的错误会交给 ErrorHandler，但可能导致视图处于不一致的状态；`@boundary` 就是为了解决这个问题。

---

## 四、升级与迁移

### Q16. 你怎么把一个老 Angular 项目（例如 v12）升级到最新版本？

- **难度**: 高级
- **标签**: 升级

**答案**

**原则**：**每次只升级一个主版本**（`ng update` 不支持跨多个主版本），每一步都要能构建、测试通过并发布。

**准备**：

- 查看 [update.angular.dev](https://update.angular.dev) 的升级指南
- 补充核心流程的 E2E 测试（升级的安全网）
- 盘点第三方依赖（Material、NgRx、组件库）在各版本的兼容性；没人维护的库要提前替换
- Node.js 和 TypeScript 版本要求

**执行**：

```bash
ng update @angular/core@13 @angular/cli@13   # 自动运行迁移脚本
# 修复、测试、提交
ng update @angular/core@14 @angular/cli@14
# ...
```

**关键节点**：

- v13：移除 View Engine，所有库必须是 Ivy
- v15–v17：类型化表单、Standalone、新控制流、application builder
- v21–v22：Zoneless 默认、OnPush 默认（只影响新组件）、Webpack 构建器弃用

**升级后的现代化**（可以分开进行）：Standalone 迁移、控制流迁移、信号输入迁移、`inject()` 迁移、application builder、Zoneless。

**口述模板（30–60 秒）**

1. 一句话结论：每次只升级一个主版本，每一步都构建、测试通过并发布。
2. 关键机制：ng update 会自动执行官方的迁移脚本。
3. 对比或注意点：提前盘点第三方依赖的兼容性，补充 E2E 测试作为安全网。
4. 可追问时补充：升级和现代化分开进行，现代化通过官方迁移脚本逐步完成。

**常见追问**

- 某个第三方库不支持新版本怎么办？→ 找替代库；fork 并自己维护；或者暂时用 `overrides` 绕过 peer 依赖检查（有风险）。

---

### Q17. 怎么把大型应用从 Zone.js 迁移到 Zoneless？

- **难度**: 高级
- **标签**: Zoneless, 迁移

**答案**

**步骤**：

1. **先全面 OnPush**：OnPush 兼容的组件基本也兼容 Zoneless（都依赖显式通知）；官方 MCP 工具有 `onpush_zoneless_migration`（v22）
2. **状态改为 Signals**：特别是在 `setTimeout`、Promise、第三方回调中修改的状态
3. **移除对 Zone 稳定性 API 的依赖**：`NgZone.onStable`、`onMicrotaskEmpty`、`isStable` → `afterNextRender`
4. **处理响应式表单**：模板依赖表单状态时，转为信号或在变化后通知
5. **SSR**：自定义异步任务用 `PendingTasks`
6. **测试**：`fixture.whenStable()` 代替 `fakeAsync` 中依赖 Zone 的写法；测试中启用 Zoneless
7. **开启**：v21+ 移除 `provideZoneChangeDetection`（或旧版本使用 `provideZonelessChangeDetection()`），从 polyfills 中移除 `zone.js`
8. **灰度**：开发环境开启检查（`provideCheckNoChangesConfig({ exhaustive: true })`），发现没有通知的更新

**第三方库**：依赖 Zone.js 触发更新的库需要升级或包装（在回调中更新信号）。

**口述模板（30–60 秒）**

1. 一句话结论：先全面 OnPush 并把状态改为 Signals，再移除 Zone.js。
2. 关键机制：OnPush 兼容的组件基本也兼容 Zoneless，因为都依赖显式通知。
3. 对比或注意点：要移除对 NgZone 稳定性 API 的依赖，SSR 中用 PendingTasks。
4. 可追问时补充：开发环境开启穷尽检查，发现遗漏通知的更新。

**常见追问**

- 迁移后有哪些收益？→ 包体积减少、启动更快、检测次数减少、调用栈干净、可以使用原生 async/await。

---

### Q18. 怎么渐进地把应用从 RxJS 主导迁移到 Signals？

- **难度**: 高级
- **标签**: Signals, 迁移

**答案**

**不是全部替换**，而是**重新划分职责**：

| 原来 | 迁移为 |
|------|--------|
| `BehaviorSubject` 保存状态 | `signal` |
| `combineLatest` + `map` 派生 | `computed` |
| 模板中 `async` 管道 | 信号（`toSignal`） |
| `@Input` + `ngOnChanges` | `input()` + `computed` |
| 简单的 GET 请求 | `httpResource` |
| 防抖、取消、重试、WebSocket 流 | **保留 RxJS** |

**顺序**：

1. 新代码使用 Signals
2. 用官方迁移脚本迁移输入、输出、查询
3. 在服务边界用 `toSignal` 向组件暴露信号，服务内部可以继续用 RxJS
4. 逐步替换 `BehaviorSubject` 状态服务为 Signals / SignalStore

**口述模板（30–60 秒）**

1. 一句话结论：不是全部替换，而是状态用 Signals，事件流和异步编排保留 RxJS。
2. 关键机制：BehaviorSubject 换成 signal，combineLatest 派生换成 computed。
3. 对比或注意点：防抖、取消、重试、WebSocket 流仍然适合 RxJS。
4. 可追问时补充：先用官方迁移脚本迁移输入输出，在服务边界用 toSignal 暴露信号。

**常见追问**

- 混用会不会很乱？→ 约定好边界：组件和模板只消费信号；复杂异步留在服务中用 RxJS，对外暴露信号。

---

### Q19. AngularJS 项目怎么迁移到 Angular？

- **难度**: 高级
- **标签**: 迁移, 遗留系统

**答案**

AngularJS 已于 2022 年停止维护，存在安全风险。

**方案对比**：

| 方案 | 说明 | 适合 |
|------|------|------|
| **重写** | 新 Angular 应用，按页面逐步替换，通过路由（反向代理）切换 | 应用不大，或者代码质量很差 |
| **ngUpgrade 混合运行** | 两个框架在同一页面运行，组件和服务可以互相降级 / 升级使用 | 大型应用需要逐步迁移 |
| **微前端（按页面）** | 新页面用新技术，旧页面保持，通过路由切换 | 只想控制风险，新功能用新技术 |

**ngUpgrade 的步骤**：

1. 准备：AngularJS 代码改为组件化、TypeScript、模块打包
2. 引导混合应用（`UpgradeModule`）
3. 从叶子组件开始逐个迁移，服务迁移后降级给 AngularJS 使用
4. 迁移路由
5. 移除 AngularJS

**注意**：混合应用中两套变更检测同时运行，性能较差；混合期不宜过长。

**口述模板（30–60 秒）**

1. 一句话结论：可以重写、用 ngUpgrade 混合运行，或者按页面用微前端方式逐步替换。
2. 关键机制：ngUpgrade 让两个框架在同一页面运行，组件和服务可以互相使用。
3. 对比或注意点：混合应用中两套变更检测同时运行，性能较差，混合期不宜过长。
4. 可追问时补充：从叶子组件开始迁移，最后迁移路由并移除 AngularJS。

**常见追问**

- 怎么说服业务方投入迁移？→ 安全风险（不再有安全更新）、招聘困难、开发效率；用按页面替换的方式降低一次性的投入。

---

## 五、安全、测试与选型

### Q20. Angular 应用怎么落地 CSP 和 Trusted Types？

- **难度**: 高级
- **标签**: 安全, CSP

**答案**

**CSP**：

- Angular 的**内联样式**（组件样式在运行时插入 `<style>`）需要 nonce：`provideNgCspNonce` 或在根元素上设置 `ngCspNonce` 属性，由服务端每次请求生成
- `autoCsp`（`angular.json` 的 security 配置）：构建时为内联脚本自动生成基于哈希的 CSP（适合静态部署，以官方为准）
- AOT 编译不需要 `unsafe-eval`

**Trusted Types**：

- Angular 内置支持：框架内部对 `innerHTML` 等危险汇的写入使用名为 `angular` 的策略
- CSP：`require-trusted-types-for 'script'; trusted-types angular angular#unsafe-bypass ...`
- `bypassSecurityTrust*` 使用 `angular#unsafe-bypass` 策略 → 如果不在白名单中，绕过调用会被浏览器拒绝，**可以借此禁止所有绕过**

**代码规范**：

- 禁止直接写 `nativeElement.innerHTML`
- `bypassSecurityTrust*` 必须经过审查（ESLint 规则）
- 用户 HTML 先用 DOMPurify 清洗

**口述模板（30–60 秒）**

1. 一句话结论：组件的内联样式需要 nonce，Angular 内置 Trusted Types 支持。
2. 关键机制：服务端为每个请求生成 nonce，通过 ngCspNonce 传给 Angular。
3. 对比或注意点：bypassSecurityTrust 使用单独的策略，不在白名单中时会被拒绝。
4. 可追问时补充：用 ESLint 规则限制直接操作 innerHTML 和绕过调用。

**常见追问**

- 为什么 Angular 应用对 XSS 相对安全？→ 默认上下文清理 + AOT 编译（模板不是运行时字符串）；主要风险来自绕过 API 和直接操作 DOM。

---

### Q21. 大型 Angular 项目的测试策略是什么？

- **难度**: 高级
- **标签**: 测试

**答案**

**测试金字塔**：

| 层次 | 工具 | 内容 |
|------|------|------|
| 单元 | Vitest（v21 默认） | 纯函数、服务、Store、管道 |
| 组件 | TestBed + Harness / Testing Library | 组件的行为和交互 |
| 集成 | TestBed + 路由 + HTTP 模拟 | 页面级流程 |
| E2E | Playwright / Cypress | 核心业务流程（登录、下单） |
| 视觉回归 | Storybook + Chromatic 等 | 组件库 |

**大型项目的要点**：

- **只运行受影响的测试**：Nx 的 `affected` 命令
- 测试**行为**而不是实现（避免测试私有方法、内部结构）
- 共享测试工具：模拟服务的工厂、测试数据构建器
- 稳定性：E2E 等待明确的条件，而不是固定的 sleep
- 覆盖率作为参考，关注核心业务

**口述模板（30–60 秒）**

1. 一句话结论：按测试金字塔分层，单元和组件测试为主，E2E 只覆盖核心流程。
2. 关键机制：单元测试用 Vitest，组件测试用 Harness 或 Testing Library。
3. 对比或注意点：测试行为而不是实现细节。
4. 可追问时补充：大型项目用 Nx affected 只运行受影响的测试。

**常见追问**

- Karma 迁移到 Vitest 要注意什么？→ 运行环境（jsdom / 浏览器模式）的差异；`fakeAsync` 等依赖 Zone 的工具在 Zoneless 下的替代写法；官方提供迁移脚本。

---

### Q22. 团队技术选型时，Angular 和 React 怎么选？

- **难度**: 高级
- **标签**: 选型

**答案**

| 考虑因素 | 倾向 Angular | 倾向 React |
|----------|--------------|------------|
| 团队规模 | 大团队、人员流动大，需要强约定 | 小团队、能自己制定规范 |
| 应用类型 | 复杂的企业级中后台、表单密集 | 各类应用，内容站、C 端 |
| 技术栈 | 团队熟悉 Java / C#（DI、类、强类型） | 函数式风格 |
| 生态需求 | 官方全家桶即可满足 | 需要丰富的第三方生态 |
| SSR / 全栈 | Angular SSR | Next.js 等成熟的全栈框架 |
| 招聘 | 国内相对少 | 国内多 |
| 长期维护 | 官方的升级路径和迁移脚本完善 | 依赖于所选的生态库 |

**回答思路**：没有绝对的好坏，基于团队、业务、生态和长期维护成本给出理由。

**口述模板（30–60 秒）**

1. 一句话结论：Angular 适合需要强约定的大团队和复杂的企业应用，React 生态更丰富、招聘更容易。
2. 关键机制：Angular 内置全家桶和依赖注入，升级有官方迁移脚本。
3. 对比或注意点：React 需要自己选择和维护生态。
4. 可追问时补充：选型要结合团队背景、业务类型和长期维护成本。

**常见追问**

- Signals 之后 Angular 和 Vue 很像了？→ 响应式模型相似（细粒度依赖追踪），但 Angular 仍然有编译期模板、依赖注入、完整的官方工具链等区别。

---

### Q23. Angular 的 i18n 在大型多语言应用中怎么落地？

- **难度**: 高级
- **标签**: 国际化

**答案**

**官方 `@angular/localize`**：

- 编译时替换，每种语言一个构建产物（`/en/`、`/zh/`），运行时零开销
- 提取：`ng extract-i18n` 生成 XLIFF，交给翻译平台
- 部署：按路径或域名分发；服务端根据 `Accept-Language` 重定向
- **问题**：构建时间乘以语言数量；切换语言需要加载另一个应用

**运行时方案（Transloco）**：

- 按需加载语言包（按功能拆分 scope）
- 可以不刷新切换
- 有运行时开销

**通用要点**：

- ICU 消息格式处理复数和性别
- 日期、数字、货币使用 Intl（管道）
- RTL 布局（CDK 的 `Directionality`）
- 翻译键的管理：命名规范、未使用键的检查、缺失翻译的告警

**口述模板（30–60 秒）**

1. 一句话结论：官方方案编译时替换，每种语言一个产物；运行时方案可以动态切换。
2. 关键机制：官方方案运行时零开销，但构建时间按语言数量增加。
3. 对比或注意点：需要动态切换时用 Transloco，按功能拆分语言包。
4. 可追问时补充：用 ICU 格式处理复数，注意 RTL 布局和翻译键的管理。

**常见追问**

- SSR 多语言怎么做？→ 官方方案每种语言一个服务端产物；服务端按路径前缀选择对应的渲染器。

---

### Q24. 怎么在 Angular 项目中使用 Nx？有什么收益？

- **难度**: 高级
- **标签**: Nx, 工程

**答案**

**Nx 为 Angular 提供**：

- **项目图和受影响分析**：`nx affected -t test build` 只构建和测试受影响的项目
- **计算缓存**（本地 + 远程）：相同输入直接复用结果
- **模块边界规则**：基于标签（`type:feature`、`scope:orders`）限制依赖方向
- **生成器**：统一创建库、组件的规范
- **迁移**：`nx migrate` 统一升级 Nx、Angular 和相关依赖
- **模块联邦**支持

**收益**：大型 Angular 应用拆分为多个库后，CI 时间显著下降；架构约束可以自动检查。

**代价**：学习成本、配置复杂度、对 Nx 的依赖。

**口述模板（30–60 秒）**

1. 一句话结论：Nx 为 Angular 提供受影响分析、计算缓存、模块边界和生成器。
2. 关键机制：受影响分析只构建和测试变更影响到的项目，远程缓存复用结果。
3. 对比或注意点：模块边界规则可以自动检查架构约束。
4. 可追问时补充：代价是学习成本和对 Nx 的依赖。

**常见追问**

- 详见[工程化专题](../engineering/00-roadmap.md)的 Monorepo 部分。

---

### Q25. 讲一个你做过的 Angular 性能优化或架构改进案例

- **难度**: 高级
- **标签**: 项目经验

**答案**

**STAR 结构参考**：

- **背景**：「一个 200+ 页面的中后台，切换菜单和表格筛选明显卡顿，INP 的 P75 超过 500ms」
- **分析**：Angular DevTools Profiler 发现每次输入都触发整个应用的变更检测；表格每行都在模板中调用格式化方法；一个第三方图表库的定时器在 Zone 中运行
- **行动**：
  - 共享组件全面 OnPush，状态迁移到 Signals
  - 格式化改为纯管道，表格改为虚拟滚动，`track` 改为 ID
  - 图表在 `runOutsideAngular` 中初始化
  - 用 ESLint 规则和 budgets 防止回退
  - 最终迁移到 Zoneless
- **结果**：INP P75 降到 150ms 以下，变更检测次数下降 80%，初始包减少若干 KB
- **复盘**：先度量再优化；规则和工具保证效果不回退

**口述模板（30–60 秒）**

1. 一句话结论：用一句话说明问题和结果。
2. 关键机制：说明如何用 Profiler 定位问题。
3. 对比或注意点：说明采取的措施和取舍。
4. 可追问时补充：给出量化的结果和防止回退的手段。

**常见追问**

- 推进中最大的阻力是什么？→ 例如老代码中大量直接修改对象的写法与 OnPush 不兼容；通过迁移脚本和分批改造解决。

---

### Q26. Angular 的发展趋势是什么？

- **难度**: 高级
- **标签**: 趋势

**答案**

（截至 2026-10，以官方为准）

- **Signals 成为核心**：输入、查询、表单（Signal Forms）、异步数据（`resource`）都基于 Signals；RxJS 变为可选
- **默认 Zoneless + OnPush**：性能默认优化
- **更轻的 DI 和懒加载**：`@Service`、`injectAsync`
- **模板语法现代化**：控制流、`@let`、箭头函数、展开语法、`@boundary` 错误边界
- **SSR**：增量水合、事件重放、混合渲染
- **工具链**：esbuild / Vite、Vitest；弃用 Webpack 构建器；关注 TypeScript 原生编译器（TSGo）
- **无障碍**：Angular Aria
- **AI 开发工具**：Angular MCP 服务器、Agent Skills、WebMCP 实验

**口述模板（30–60 秒）**

1. 一句话结论：Angular 的方向是以 Signals 为核心，默认 Zoneless 和 OnPush。
2. 关键机制：表单、异步数据、输入和查询都基于 Signals。
3. 对比或注意点：工具链转向 esbuild、Vite 和 Vitest，弃用 Webpack 构建器。
4. 可追问时补充：SSR 方面有增量水合和事件重放，还加入了 AI 开发工具支持。

**常见追问**

- 老项目需要跟进所有新特性吗？→ 不需要一次性跟进；按收益排序：安全更新（保持在支持的版本）> 性能相关（OnPush / Signals）> 语法现代化。
