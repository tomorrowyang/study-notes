# Angular 真实面试题

> 根据近年公开面经和技术社区讨论整理，`来源倾向`表示这类题目常出现的场景，不代表某公司的固定题库。答案为要点，详细内容见对应链接。

---

### R1. 讲一下 Angular 的变更检测机制

- **来源倾向**: 必问，几乎每次 Angular 面试

**答案**

- 变更检测把组件状态同步到 DOM：从根到叶遍历组件树，执行编译后的模板更新指令，比较绑定的新旧值
- 触发：Zone.js 拦截异步任务后自动触发；Zoneless（v21+ 默认）下由 Signal 更新、模板事件、`markForCheck` 等显式通知触发
- 策略：`Eager`（原 Default，每次都检查）和 `OnPush`（只在被标记为脏时检查，v22 起新组件默认）
- Signals 可以做到只刷新读取了变化信号的视图

**口述模板（30–60 秒）**

1. **一句话结论**：变更检测从根到叶遍历组件树，比较模板绑定的新旧值并更新 DOM。
2. **关键机制**：以前由 Zone.js 在异步任务后自动触发，现在默认 Zoneless，由 Signal 和事件等显式通知触发。
3. **对比或注意点**：OnPush 组件只有在输入引用变化、事件、Signal 变化或 markForCheck 时才检查。
4. **可追问时补充**：Signals 只标记读取它的视图，可以实现更细粒度的刷新。

详见：[20-mid Q1–Q5](./20-mid-qa.md)、[30-senior Q2–Q3](./30-senior-qa.md)

---

### R2. Zone.js 是做什么的？为什么要去掉它？

- **来源倾向**: 高频（中高级）

**答案**

- 通过猴子补丁拦截所有异步 API，在任务完成后通知 Angular 进行变更检测
- 问题：过度检测、补丁开销、包体积、原生 async/await 不兼容、调用栈复杂
- Zoneless：v21 起新应用默认，依赖 Signals 和显式通知

详见：[20-mid Q2、Q5](./20-mid-qa.md)、[30-senior Q17](./30-senior-qa.md)

---

### R3. OnPush 什么时候会触发更新？修改对象属性为什么不更新？

- **来源倾向**: 必问

**答案**

- 触发条件：输入引用变化、组件内的模板事件、读取的 Signal 变化、`async` 管道收到新值、`markForCheck`
- 修改对象内部属性时引用没变，`===` 比较认为没有变化
- 解决：不可变更新或使用 Signal

详见：[20-mid Q3](./20-mid-qa.md)

---

### R4. Signals 是什么？和 RxJS 有什么区别？

- **来源倾向**: 高频（2024 年以后几乎必问）

**答案**

- Signal 是包装值的响应式容器，读取时自动追踪依赖；`signal` / `computed` / `effect` / `linkedSignal` / `resource`
- Signals 表示**同步的状态**（总有当前值），RxJS 表示**异步的事件流**
- 分工：状态用 Signals，事件和复杂异步编排（防抖、取消、重试）用 RxJS
- 转换：`toSignal` / `toObservable`

**口述模板（30–60 秒）**

1. **一句话结论**：Signal 是有当前值的响应式状态，RxJS 是随时间变化的事件流。
2. **关键机制**：Signal 读取时自动追踪依赖，computed 惰性计算并缓存。
3. **对比或注意点**：状态用 Signals，防抖、取消、重试这类异步编排用 RxJS。
4. **可追问时补充**：两者通过 toSignal 和 toObservable 互相转换。

详见：[20-mid Q7–Q12](./20-mid-qa.md)、[40 H1](./40-coding-handwritten.md)

---

### R5. computed 和 effect 有什么区别？什么时候用 effect？

- **来源倾向**: 高频

**答案**

- `computed`：派生值，惰性、缓存、只读、无副作用
- `effect`：执行副作用（同步到存储、日志、第三方库、DOM），异步调度
- 不要用 effect 设置其他信号（用 `computed` / `linkedSignal`），不要用 effect 请求数据（用 `resource`）

详见：[20-mid Q7–Q9](./20-mid-qa.md)

---

### R6. 讲讲 Angular 的依赖注入

- **来源倾向**: 必问

**答案**

- 注入器层级：元素注入器（跟随 DOM）+ 环境注入器（根、路由、平台）
- 解析：沿元素注入器向上 → 环境注入器向上 → 找不到报错
- 提供方式：`useClass` / `useValue` / `useFactory` / `useExisting`，`InjectionToken`，`multi`
- 作用域：`providedIn: 'root'`（单例、可摇树）、路由级、组件级（每个实例一个）
- 修饰符：`optional`、`self`、`skipSelf`、`host`

**口述模板（30–60 秒）**

1. **一句话结论**：Angular 的依赖注入由分层的注入器组成，按层级向上查找依赖。
2. **关键机制**：先查元素注入器，再查环境注入器，providedIn root 是可摇树的全局单例。
3. **对比或注意点**：组件级 providers 每个组件实例一个，适合局部状态。
4. **可追问时补充**：内部用布隆过滤器加速元素注入器的查找。

详见：[10-junior Q14–Q16](./10-junior-qa.md)、[20-mid Q19–Q22](./20-mid-qa.md)、[30-senior Q4](./30-senior-qa.md)

---

### R7. switchMap、mergeMap、concatMap、exhaustMap 的区别和使用场景

- **来源倾向**: 必问

**答案**

- `switchMap`：取消旧的 → 搜索、参数变化加载
- `mergeMap`：并行 → 互不影响的并行请求
- `concatMap`：排队 → 顺序保存
- `exhaustMap`：忽略新的 → 防重复提交

**口述模板（30–60 秒）**

1. **一句话结论**：区别在于新值到来时如何处理进行中的内部流。
2. **关键机制**：取消旧的、并行、排队、忽略新的。
3. **对比或注意点**：搜索用 switchMap，提交用 exhaustMap，顺序保存用 concatMap。
4. **可追问时补充**：用 switchMap 做保存可能导致请求被取消而服务端已经处理。

详见：[20-mid Q14](./20-mid-qa.md)

---

### R8. 怎么避免 RxJS 的内存泄漏？

- **来源倾向**: 高频

**答案**

- `async` 管道、`takeUntilDestroyed()`、`toSignal()` 自动管理
- `takeUntilDestroyed` 放在 pipe 的最后
- 用高阶映射操作符替代嵌套订阅
- HttpClient 请求会自动完成，但取消订阅可以中止未完成的请求

详见：[10-junior Q24](./10-junior-qa.md)、[20-mid Q18](./20-mid-qa.md)、[30-senior Q8](./30-senior-qa.md)

---

### R9. 生命周期钩子有哪些？执行顺序？

- **来源倾向**: 高频（初中级）

**答案**

- `constructor` → `ngOnChanges` → `ngOnInit` → `ngDoCheck` → `ngAfterContentInit` → `ngAfterContentChecked` → `ngAfterViewInit` → `ngAfterViewChecked` → `ngOnDestroy`
- 新 API：`afterNextRender` / `afterEveryRender`（只在浏览器执行）、`DestroyRef`
- 信号输入可以用 `computed` 代替 `ngOnChanges`

详见：[10-junior Q7](./10-junior-qa.md)

---

### R10. ExpressionChangedAfterItHasBeenCheckedError 怎么产生、怎么解决？

- **来源倾向**: 常见

**答案**

- 开发模式下的二次检查发现值在检查后被修改，违反单向数据流
- 常见原因：子组件在 `ngAfterViewInit` 中修改父组件状态、模板中返回不稳定的值
- 解决：调整数据流、使用 `computed`；不要用 `setTimeout` 掩盖问题

详见：[20-mid Q6](./20-mid-qa.md)

---

### R11. 路由守卫有哪些？canActivate 和 canMatch 的区别？

- **来源倾向**: 高频

**答案**

- `canActivate`、`canActivateChild`、`canDeactivate`、`canMatch`、`resolve`
- `canMatch` 不通过时路由不匹配，**懒加载的代码不会被下载**，并继续尝试其他路由
- 推荐函数式守卫，返回 `UrlTree` 实现重定向
- 前端守卫只是体验，权限由服务端校验

详见：[10-junior Q19](./10-junior-qa.md)、[40 H4](./40-coding-handwritten.md)

---

### R12. 怎么实现一个自定义表单控件？

- **来源倾向**: 高频（中级）

**答案**

- 实现 `ControlValueAccessor`：`writeValue`、`registerOnChange`、`registerOnTouched`、`setDisabledState`
- 注册 `NG_VALUE_ACCESSOR`（`multi: true`）
- `writeValue` 中不要调用 `onChange`
- 自带校验：实现 `Validator`，注册 `NG_VALIDATORS`
- Signal Forms 中对应 `FormValueControl`

**口述模板（30–60 秒）**

1. **一句话结论**：实现 ControlValueAccessor 接口并注册为 NG_VALUE_ACCESSOR。
2. **关键机制**：writeValue 接收表单的值，onChange 回调把组件的值传回表单。
3. **对比或注意点**：要处理 touched 和 disabled，writeValue 中不要调用 onChange。
4. **可追问时补充**：组件可以同时实现 Validator 提供自带的校验。

详见：[20-mid Q25](./20-mid-qa.md)、[40 H2](./40-coding-handwritten.md)

---

### R13. 响应式表单和模板驱动表单的区别？Signal Forms 了解吗？

- **来源倾向**: 高频

**答案**

- 模板驱动：在模板中用 `ngModel` 定义，简单表单
- 响应式：在类中定义，显式、类型化，适合复杂和动态表单
- Signal Forms（v22 稳定）：以 signal 为模型，声明式 schema 校验，完全类型化；可以和响应式表单互操作渐进迁移

详见：[10-junior Q20](./10-junior-qa.md)、[20-mid Q26](./20-mid-qa.md)

---

### R14. Standalone 组件有什么好处？

- **来源倾向**: 常见

**答案**

- 组件自己声明依赖，不需要 NgModule；v19 起默认
- 依赖关系清晰、更容易懒加载和摇树、样板代码更少
- 官方迁移脚本：`ng generate @angular/core:standalone`

详见：[10-junior Q30](./10-junior-qa.md)

---

### R15. 纯管道和非纯管道的区别？为什么不建议模板中调用方法？

- **来源倾向**: 常见

**答案**

- 纯管道只在输入引用变化时重新计算，有缓存效果
- 非纯管道每次变更检测都执行
- 模板中的方法每次检测都执行 → 用纯管道或 `computed`

详见：[10-junior Q6](./10-junior-qa.md)

---

### R16. @defer 是什么？和路由懒加载有什么区别？

- **来源倾向**: 中高级常见

**答案**

- 编译器把 `@defer` 块中的组件拆为独立代码块，按触发条件（视口、交互、空闲、定时）加载
- 支持 `@placeholder`、`@loading`、`@error`、`prefetch`
- 路由懒加载按页面拆分，`@defer` 按页面中的区块拆分
- SSR 中配合增量水合

详见：[10-junior Q10](./10-junior-qa.md)、[20-mid Q27](./20-mid-qa.md)

---

### R17. Angular SSR 和水合是怎么工作的？

- **来源倾向**: 中高级常见

**答案**

- 服务端渲染 HTML；客户端水合时复用 DOM（按水合注解认领节点），只附加事件
- 事件重放、增量水合、HTTP 传输缓存、混合渲染（SSR / SSG / CSR 按路由）
- 常见问题：服务端访问浏览器 API、DOM 不一致导致水合失败

详见：[20-mid Q28](./20-mid-qa.md)、[30-senior Q5](./30-senior-qa.md)、[50 S7](./50-system-design.md)

---

### R18. 大型 Angular 应用卡顿，你会怎么排查和优化？

- **来源倾向**: 高级必问

**答案**

- 定位：Angular DevTools Profiler（检测次数、耗时、触发来源）、Chrome Performance
- 常见原因：Eager 策略 + Zone.js、高频事件、模板中调用方法、大列表、第三方库触发检测、订阅泄漏
- 优化：OnPush + Signals、Zoneless、`runOutsideAngular`、`computed` / 纯管道、虚拟滚动、`track`
- 验证：INP、Profiler 前后对比；用 ESLint 和 budgets 防止回退

**口述模板（30–60 秒）**

1. **一句话结论**：先用 Profiler 定位是检测太频繁、范围太大还是单次渲染太重。
2. **关键机制**：Profiler 显示每次检测的触发来源和各组件的耗时。
3. **对比或注意点**：常见原因是默认策略、高频事件和模板中调用方法。
4. **可追问时补充**：优化后用 INP 验证，并用规则和预算防止回退。

详见：[30-senior Q6–Q9](./30-senior-qa.md)

---

### R19. NgRx 的核心概念？什么时候需要用 NgRx？

- **来源倾向**: 高频（使用 NgRx 的团队）

**答案**

- Store、Action、Reducer、Selector（记忆化）、Effect（副作用）
- SignalStore：基于 Signals 的轻量方案，大多数新项目推荐
- 选型：服务 + Signals 适合中小型；复杂跨功能交互、需要追踪每次状态变化的大团队用 NgRx Store
- Effect 中的错误必须在内部捕获

详见：[30-senior Q11–Q12](./30-senior-qa.md)、[40 H7](./40-coding-handwritten.md)

---

### R20. 怎么写一个 HTTP 拦截器实现令牌刷新？

- **来源倾向**: 高频（手写）

**答案**

- 函数式拦截器附加令牌，401 时刷新并重试
- 并发 401 共享同一个刷新请求（`shareReplay`）
- 刷新请求本身跳过拦截（`HttpContextToken`）
- 只重试一次，刷新失败跳转登录

详见：[40 H3](./40-coding-handwritten.md)

---

### R21. 结构指令怎么写？`*` 语法糖是什么？

- **来源倾向**: 常见

**答案**

- `*appX="expr"` 等价于 `<ng-template [appX]="expr">`
- 注入 `TemplateRef` 和 `ViewContainerRef`，按条件 `createEmbeddedView` / `clear`
- 状态不变时不要重建视图
- 内置的 `*ngIf` / `*ngFor` 已被 `@if` / `@for` 取代

详见：[40 H6](./40-coding-handwritten.md)

---

### R22. @for 为什么必须写 track？

- **来源倾向**: 常见

**答案**

- `track` 用于识别列表项，数据变化时复用对应的 DOM 和组件实例
- 没有稳定标识时，每次刷新都会销毁重建（性能差、状态丢失）
- 用唯一 ID；`$index` 只适合静态列表

详见：[10-junior Q4](./10-junior-qa.md)

---

### R23. Angular 和 React / Vue 相比有什么优缺点？

- **来源倾向**: 常见（选型、开放问题）

**答案**

- 优点：完整的官方全家桶、依赖注入、强约定、TypeScript 优先、官方升级工具
- 缺点：学习曲线陡、国内社区和招聘规模较小、灵活性低
- 适合大型团队和复杂的企业级应用
- Signals 之后响应式模型与 Vue 相似

详见：[10-junior Q1](./10-junior-qa.md)、[30-senior Q22](./30-senior-qa.md)

---

### R24. 你怎么把一个老 Angular 项目升级到最新版本？

- **来源倾向**: 高级常见

**答案**

- 每次升级一个主版本，`ng update` 执行官方迁移脚本，每步可发布
- 准备：E2E 测试、第三方依赖盘点
- 现代化：Standalone、控制流、信号 API、application builder、Vitest、OnPush → Signals → Zoneless
- 按收益排序，与业务并行

**口述模板（30–60 秒）**

1. **一句话结论**：每次只升级一个主版本，每一步都构建、测试通过并发布。
2. **关键机制**：ng update 自动执行官方的迁移脚本。
3. **对比或注意点**：提前补充 E2E 测试并盘点第三方依赖的兼容性。
4. **可追问时补充**：版本追上后再分阶段做现代化，最终迁移到 Signals 和 Zoneless。

详见：[30-senior Q16–Q18](./30-senior-qa.md)、[50 S5](./50-system-design.md)

---

### R25. Angular 的安全机制？怎么防 XSS？

- **来源倾向**: 常见

**答案**

- 默认按上下文清理和转义绑定的值；AOT 编译避免模板注入
- `bypassSecurityTrust*` 绕过保护，只能用于可信内容
- 不要直接操作 `nativeElement.innerHTML`
- 支持 CSP nonce 和 Trusted Types；HttpClient 内置 XSRF 支持

详见：[10-junior Q29](./10-junior-qa.md)、[30-senior Q20](./30-senior-qa.md)、[前端安全专题](../frontend-security/00-roadmap.md)
