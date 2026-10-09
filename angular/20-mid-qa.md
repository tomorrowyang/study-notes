# Angular 中级 Q&A

> 目标：能讲清变更检测、Zoneless、Signals、RxJS、依赖注入的原理和常见坑，能处理路由、表单、SSR、测试中的中等复杂问题。

---

## 一、变更检测

### Q1. Angular 的变更检测是怎么工作的？

- **难度**: 中级
- **标签**: 变更检测

**答案**

**变更检测（Change Detection）**：把组件状态同步到 DOM 的过程。

**流程**：

1. 某个事件「通知」Angular 需要检查（Zone.js 拦截到异步任务完成，或 Zoneless 下的显式通知）
2. Angular 从**根组件开始，自上而下**遍历组件树
3. 对每个需要检查的组件，执行编译后的模板函数的更新部分，**比较每个绑定表达式的新旧值**，不同时更新 DOM
4. 开发模式下会再检查一次，验证值没有在检查后被修改

**编译产物**：AOT 编译器把模板编译成创建（create）和更新（update）两段指令代码，更新时只比较绑定的表达式，不做虚拟 DOM 的 diff。

**哪些组件会被检查**：

- `Eager`（原 `Default`）策略：每次都检查
- `OnPush` 策略：只有被标记为脏时才检查（v22 起新组件默认 `OnPush`）
- Signals：模板读取的信号变化时，**只标记读取它的视图**，可以实现更细粒度的刷新

**口述模板（30–60 秒）**

1. 一句话结论：变更检测是从根到叶遍历组件树，比较模板绑定的新旧值并更新 DOM。
2. 关键机制：模板被编译成更新指令，逐个比较绑定表达式，不使用虚拟 DOM。
3. 对比或注意点：OnPush 组件只有被标记为脏时才检查，Eager 每次都检查。
4. 可追问时补充：开发模式会再检查一次，检测单向数据流是否被破坏。

**常见追问**

- Angular 和 React 的更新方式有什么区别？→ React 重新执行组件函数生成虚拟 DOM 再 diff；Angular 执行编译好的更新指令，直接比较绑定值。

---

### Q2. Zone.js 的原理是什么？有什么问题？

- **难度**: 中级
- **标签**: Zone.js

**答案**

**原理**：Zone.js 通过**猴子补丁（monkey patch）**替换浏览器的异步 API（`setTimeout`、`Promise`、`addEventListener`、XHR 等），在异步任务执行前后插入钩子。Angular 在 `NgZone` 中运行应用，任务完成且微任务队列清空时（`onMicrotaskEmpty`），自动触发变更检测。

**优点**：开发者不需要手动通知，状态变化后视图自动更新。

**问题**：

- **过度检测**：任何异步事件（包括与 UI 无关的定时器、第三方库的事件）都会触发整个应用的检查
- **性能**：补丁本身的开销；需要 `runOutsideAngular` 规避高频事件
- **包体积**：约几十 KB
- **兼容性**：原生 `async/await` 无法被补丁（需要降级为 Promise），新的浏览器 API 需要额外补丁
- **调试**：调用栈复杂

这些问题推动了 **Zoneless**（v21 起新应用默认）。

**口述模板（30–60 秒）**

1. 一句话结论：Zone.js 通过猴子补丁拦截所有异步 API，在任务完成后自动触发变更检测。
2. 关键机制：NgZone 在微任务队列清空时通知 Angular 检查。
3. 对比或注意点：问题是过度检测、性能开销、包体积和原生 async/await 不兼容。
4. 可追问时补充：v21 起新应用默认 Zoneless，依靠 Signals 和显式通知。

**常见追问**

- `runOutsideAngular` 的用途？→ 在 Zone 外执行高频事件（`mousemove`、`scroll`、动画帧），避免每次都触发变更检测，需要更新视图时再 `run` 回来。

---

### Q3. `OnPush` 组件在什么情况下会被检查？

- **难度**: 中级
- **标签**: OnPush

**答案**

`OnPush` 组件只有在被**标记为脏**（及其祖先路径）时才会被检查：

1. **输入引用变化**（`===` 比较）：父组件传入新的对象引用
2. **组件或其子组件中的模板事件**（点击、输入等绑定的事件）
3. **模板中读取的 Signal 变化**
4. **`async` 管道收到新值**（内部调用 `markForCheck`）
5. **手动调用 `markForCheck()`**
6. `@defer` 块状态变化等框架内部事件

**常见坑**：

```ts
// ❌ 修改对象内部，引用不变，OnPush 子组件不会更新
this.user.name = 'new';
// ✅ 不可变更新
this.user = { ...this.user, name: 'new' };
// ✅ 或使用 signal
this.user.update((u) => ({ ...u, name: 'new' }));
```

**v22 起新组件默认 `OnPush`**，原 `Default` 策略更名为 `ChangeDetectionStrategy.Eager`。

**口述模板（30–60 秒）**

1. 一句话结论：OnPush 组件只有被标记为脏时才检查。
2. 关键机制：触发条件是输入引用变化、模板事件、读取的 Signal 变化、async 管道和 markForCheck。
3. 对比或注意点：修改对象内部属性不会触发，要用不可变更新或 Signal。
4. 可追问时补充：v22 起新组件默认 OnPush，原 Default 策略更名为 Eager。

**常见追问**

- `setTimeout` 中修改普通属性，`OnPush` 组件会更新吗？→ 不会（即使有 Zone.js 触发了检测，该组件也没被标记为脏）；改用 Signal 或 `markForCheck`。

---

### Q4. `markForCheck` 和 `detectChanges` 有什么区别？

- **难度**: 中级
- **标签**: 变更检测

**答案**

| 方法 | 行为 |
|------|------|
| `markForCheck()` | 把当前组件**及所有祖先**标记为脏，**等待下一次**变更检测（异步、调度式） |
| `detectChanges()` | **立即同步**检查当前组件及其子组件 |
| `detach()` / `reattach()` | 把组件从变更检测树中分离 / 重新接入 |

**选择**：

- 绝大多数情况用 `markForCheck`（或者直接用 Signals）
- `detectChanges` 用于需要立即同步更新 DOM 的场景（例如更新后马上测量尺寸），或已经 `detach` 的组件手动刷新
- 高频更新的组件（实时行情）可以 `detach` 后按固定频率 `detectChanges`

**口述模板（30–60 秒）**

1. 一句话结论：markForCheck 标记组件和祖先为脏，等待下一次检测；detectChanges 立即同步检查当前子树。
2. 关键机制：markForCheck 是调度式的，detectChanges 是同步执行的。
3. 对比或注意点：大多数情况用 markForCheck 或直接用 Signals。
4. 可追问时补充：高频更新的组件可以 detach 后定时 detectChanges。

**常见追问**

- 为什么 `markForCheck` 要标记祖先？→ 变更检测从根往下遍历，如果祖先是 `OnPush` 且没被标记，遍历不会到达当前组件。

---

### Q5. Zoneless 应用中，什么会触发变更检测？

- **难度**: 中级
- **标签**: Zoneless

**答案**

Zoneless（v21 起新应用默认，不再加载 Zone.js）下，Angular 依赖**显式的通知**来调度变更检测：

- 模板中读取的 **Signal 被更新**
- **模板或宿主的事件监听器**回调执行
- `ChangeDetectorRef.markForCheck()`（`async` 管道内部会调用）
- `ComponentRef.setInput()`
- 挂载一个被标记为脏的视图

**不会自动触发的**：`setTimeout`、`Promise.then`、`fetch` 回调、第三方库回调中修改**普通属性**。

**迁移要点**：

- 状态改为 Signals，或在异步回调后 `markForCheck`
- 移除 `NgZone.onStable`、`onMicrotaskEmpty`、`isStable` 的依赖（改用 `afterNextRender`）
- 响应式表单的 `setValue` 等不会自动调度检测，模板依赖表单状态时需要通知或转为 Signal
- 检查是否残留 `provideZoneChangeDetection`

**口述模板（30–60 秒）**

1. 一句话结论：Zoneless 下变更检测由显式通知触发，主要是 Signal 更新、模板事件和 markForCheck。
2. 关键机制：Angular 收到通知后调度一次变更检测，只检查被标记的视图。
3. 对比或注意点：setTimeout 和 Promise 回调中修改普通属性不会触发更新。
4. 可追问时补充：迁移时把状态改为 Signals，移除对 NgZone 稳定性 API 的依赖。

**常见追问**

- Zoneless 有什么好处？→ 包更小、启动更快、检测更精确、调用栈更干净、原生 async/await 不需要降级。

---

### Q6. `ExpressionChangedAfterItHasBeenCheckedError` 是怎么产生的？怎么解决？

- **难度**: 中级
- **标签**: 变更检测, 排错

**答案**

**原因**：开发模式下，Angular 在一轮变更检测后会再检查一次，如果某个绑定的值变了，说明**在检查过程中修改了已经检查过的数据**，违反了单向数据流。

**常见场景**：

- 子组件在 `ngAfterViewInit` 中修改父组件的状态
- 子组件在 `ngOnInit` 中通过服务修改了父组件已经渲染的值
- 模板中的 getter 每次返回新的值（如 `new Date()`、`Math.random()`）

**解决**：

- 调整数据流：在更早的时机（父组件中）准备好数据
- 使用 Signals / `computed` 派生值
- 不要在模板中调用每次返回不同值的方法
- 实在需要异步修改：`afterNextRender` 或 `queueMicrotask`（权宜之计）

**不要**用 `setTimeout` 掩盖问题而不理解原因。

**口述模板（30–60 秒）**

1. 一句话结论：这个错误说明在变更检测过程中修改了已经检查过的数据。
2. 关键机制：开发模式会再检查一次，发现值变化就报错。
3. 对比或注意点：常见原因是子组件在生命周期钩子中修改父组件状态，或模板中返回不稳定的值。
4. 可追问时补充：根本解决是调整数据流，用 computed 派生值。

**常见追问**

- 生产环境会报这个错吗？→ 不会（生产模式不做第二次检查），但界面可能显示不一致的数据。

---

## 二、Signals

### Q7. Signal 是什么？`signal`、`computed`、`effect` 怎么用？

- **难度**: 中级
- **标签**: Signals

**答案**

Signal 是一个**包装值的响应式容器**，读取时自动追踪依赖，变化时通知依赖方。

```ts
const count = signal(0);                       // 可写信号
const double = computed(() => count() * 2);    // 派生信号，惰性计算 + 缓存
effect(() => console.log('count =', count())); // 副作用，依赖变化时重新执行

count.set(1);
count.update((c) => c + 1);
```

| API | 特点 |
|-----|------|
| `signal` | 可写；默认用 `Object.is` 判断是否变化（可以传 `equal` 自定义） |
| `computed` | 只读；**惰性**（被读取时才计算）、**缓存**（依赖不变不重新计算）；依赖是动态的 |
| `effect` | 在注入上下文中创建，随组件销毁；异步调度执行（不是同步） |

**口述模板（30–60 秒）**

1. 一句话结论：Signal 是包装值的响应式容器，读取时自动追踪依赖。
2. 关键机制：signal 可写，computed 惰性计算并缓存，effect 在依赖变化时执行副作用。
3. 对比或注意点：signal 默认用 Object.is 比较，修改对象内部不会通知。
4. 可追问时补充：computed 的依赖是动态的，只追踪本次执行中实际读取的信号。

**常见追问**

- 为什么修改数组 `items().push(x)` 不会触发更新？→ 引用没变，`Object.is` 判断为相等；要用 `update((l) => [...l, x])`。

---

### Q8. `effect` 应该在什么时候用？有什么坑？

- **难度**: 中级
- **标签**: Signals, effect

**答案**

**适合的场景**（与非响应式世界同步）：

- 同步到 `localStorage`、日志、分析上报
- 操作第三方库（图表、地图）、Canvas
- 自定义的 DOM 行为

**不适合的场景**：

- **用 effect 去设置另一个 signal**（派生状态）→ 应该用 `computed` 或 `linkedSignal`
- 数据请求 → 用 `resource` / `httpResource`

**坑**：

- effect 中写信号可能导致循环和多余的渲染
- 读取信号即建立依赖；不想追踪的读取用 `untracked()`
- effect 是**异步调度**的，不会在 `set` 之后立即执行
- 在注入上下文之外创建需要传入 `injector`
- 清理：`effect((onCleanup) => { const id = setInterval(...); onCleanup(() => clearInterval(id)); })`

**口述模板（30–60 秒）**

1. 一句话结论：effect 用于把 Signal 的变化同步到非响应式的世界，比如存储、日志、第三方库。
2. 关键机制：effect 在依赖变化后异步执行，支持 onCleanup 清理。
3. 对比或注意点：不要用 effect 设置其他信号，派生状态用 computed 或 linkedSignal。
4. 可追问时补充：不想追踪的读取用 untracked。

**常见追问**

- 为什么官方不鼓励 effect 写信号？→ 会产生隐式的数据流，难以追踪，并可能导致额外的变更检测轮次甚至循环。

---

### Q9. `linkedSignal` 解决什么问题？

- **难度**: 中级
- **标签**: Signals

**答案**

**场景**：一个状态**默认由其他信号派生，但用户也可以修改它**；源变化时重置。

例如：选项列表变化时，选中项重置为第一个；用户可以手动选择其他项。

```ts
options = input.required<Option[]>();

selected = linkedSignal({
  source: this.options,
  computation: (opts, previous) =>
    // 新列表中仍然存在之前选中的项时保留，否则选第一个
    opts.find((o) => o.id === previous?.value?.id) ?? opts[0],
});

choose(o: Option) {
  this.selected.set(o);   // 可写
}
```

简写：`selected = linkedSignal(() => this.options()[0]);`

**对比**：

- `computed`：只读，不能手动修改
- `effect` + `signal`：可以实现，但是隐式、容易出错

**口述模板（30–60 秒）**

1. 一句话结论：linkedSignal 是可写的派生信号，源变化时重新计算，用户也可以手动修改。
2. 关键机制：computation 可以拿到上一次的值，决定是保留还是重置。
3. 对比或注意点：computed 是只读的，用 effect 同步信号容易出错。
4. 可追问时补充：典型场景是选项变化时重置选中项。

**常见追问**

- `linkedSignal` 和 `model` 有什么关系？→ 不同的东西；`model` 是双向绑定的输入，`linkedSignal` 是组件内部的可写派生状态。

---

### Q10. `resource` 和 `httpResource` 是什么？

- **难度**: 中级
- **标签**: Signals, 异步

**答案**

把**异步数据**接入 Signals（v22 稳定）：

```ts
userId = input.required<string>();

// 通用的异步资源
user = resource({
  params: () => ({ id: this.userId() }),          // 依赖的信号，变化时重新加载
  loader: ({ params, abortSignal }) =>
    fetch(`/api/users/${params.id}`, { signal: abortSignal }).then((r) => r.json()),
});

// 基于 HttpClient（经过拦截器）
orders = httpResource<Order[]>(() => `/api/users/${this.userId()}/orders`);
```

**提供的状态**：`value()`、`hasValue()`、`isLoading()`、`error()`、`status()`；方法 `reload()`、`set()`（本地乐观更新）。

**行为**：

- 参数变化时**自动取消上一次请求**（`abortSignal`），避免竞态
- `params` 返回 `undefined` 时不加载
- `rxResource`：loader 返回 Observable

**注意**：`resource` 适合**读取**数据；修改数据（POST）仍然用 HttpClient 方法调用。

**口述模板（30–60 秒）**

1. 一句话结论：resource 和 httpResource 把异步数据请求接入 Signals。
2. 关键机制：参数信号变化时自动重新加载，并取消上一次请求，避免竞态。
3. 对比或注意点：提供 value、isLoading、error 等状态信号；适合读取，不适合修改操作。
4. 可追问时补充：httpResource 基于 HttpClient，会经过拦截器。

**常见追问**

- 怎么避免切换参数时界面闪成空白？→ 加载期间 `value()` 可能保留旧值（取决于状态），模板根据 `isLoading()` 显示加载指示而不是清空内容。

---

### Q11. Signals 和 RxJS 怎么分工？怎么互相转换？

- **难度**: 中级
- **标签**: Signals, RxJS

**答案**

| 方面 | Signals | RxJS |
|------|---------|------|
| 模型 | **同步的状态**（总有当前值） | **异步的事件流**（随时间的值序列） |
| 适合 | 组件状态、派生状态、模板绑定 | 事件处理、复杂异步编排（防抖、取消、重试、合并） |
| 学习成本 | 低 | 高 |
| 取消 | 不需要 | 需要管理订阅 |

**原则**：**状态用 Signals，事件和异步编排用 RxJS**，在边界处转换。

```ts
// Observable → Signal
results = toSignal(
  toObservable(this.query).pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap((q) => this.api.search(q)),
  ),
  { initialValue: [] },
);
```

- `toSignal`：订阅 Observable，自动在销毁时取消；没有 `initialValue` 时初始值为 `undefined`
- `toObservable`：基于 effect，**异步**发出值（不是每次 set 都同步发出，同一轮的多次修改只发出最后一次）

**口述模板（30–60 秒）**

1. 一句话结论：状态用 Signals，事件流和复杂的异步编排用 RxJS。
2. 关键机制：toSignal 把 Observable 转为 Signal，toObservable 把 Signal 转为 Observable。
3. 对比或注意点：toObservable 是异步发出的，同一轮的多次修改只发出最后的值。
4. 可追问时补充：典型组合是信号作为搜索词，经过 RxJS 防抖和 switchMap 后再转回信号。

**常见追问**

- Angular 会移除 RxJS 吗？→ RxJS 在 Angular 中会变成可选的，但 HttpClient、路由、表单仍然提供 Observable API；Signals 不能替代流式编排。

---

### Q12. `untracked` 有什么用？Signal 的依赖追踪是怎么实现的？

- **难度**: 中级
- **标签**: Signals, 原理

**答案**

**依赖追踪**：

- 全局有一个「当前活跃的消费者」（正在执行的 computed、effect 或模板）
- 读取信号时，信号把当前消费者记录为自己的依赖方，消费者也记录这个信号
- 信号变化时，通知依赖方「可能脏了」（**推送脏标记**）
- 依赖方被读取或调度时，检查依赖的版本号决定是否需要重新计算（**拉取值**）——这就是「推拉结合」

**`untracked`**：在函数中读取信号但**不建立依赖**。

```ts
effect(() => {
  const user = this.user();                       // 追踪
  untracked(() => this.logger.log(user, this.config()));  // config 变化不触发 effect
});
```

**口述模板（30–60 秒）**

1. 一句话结论：读取信号时会把当前活跃的消费者记录为依赖方，untracked 用于读取但不追踪。
2. 关键机制：信号变化时向依赖方推送脏标记，依赖方被读取时再按版本号拉取新值。
3. 对比或注意点：这种推拉结合保证了 computed 的惰性和只计算一次。
4. 可追问时补充：effect 中调用的服务方法如果内部读取了信号，也会建立依赖，需要用 untracked。

**常见追问**

- 什么是「菱形依赖」问题？Signals 怎么解决？→ A 被 B、C 依赖，D 依赖 B 和 C；A 变化时 D 不应该计算两次或看到不一致的中间状态。推拉结合下 D 在被读取时才计算，此时 B、C 都已是最新值。

---

### Q13. 信号式输入 `input()` 有哪些高级用法？

- **难度**: 中级
- **标签**: Signals, 组件

**答案**

```ts
export class Avatar {
  size = input(32, { transform: numberAttribute });          // "48" → 48
  rounded = input(false, { transform: booleanAttribute });   // <app-avatar rounded>
  user = input.required<User>();
  label = input<string, string | undefined>('', { transform: (v) => v ?? '' });

  initials = computed(() => this.user().name.slice(0, 1).toUpperCase());
}
```

- `transform`：转换输入值（字符串属性转为数字和布尔）
- `alias`：对外使用不同的名称
- `input.required`：必须传入，读取未设置的值会报错
- 输入信号是**只读的**，组件内部需要修改时用 `model()` 或 `linkedSignal`

**口述模板（30–60 秒）**

1. 一句话结论：input 支持 required、transform 和 alias。
2. 关键机制：transform 可以把字符串属性转换为数字或布尔值。
3. 对比或注意点：输入信号是只读的，内部需要修改时用 model 或 linkedSignal。
4. 可追问时补充：输入信号可以直接用于 computed，不需要 ngOnChanges。

**常见追问**

- 怎么从 `@Input` 迁移？→ 官方迁移脚本 `ng generate @angular/core:signal-input-migration`。

---

## 三、RxJS

### Q14. `switchMap`、`mergeMap`、`concatMap`、`exhaustMap` 有什么区别？

- **难度**: 中级
- **标签**: RxJS

**答案**

都是把外部的值映射为内部的 Observable，区别在于**新的值到来时如何处理正在进行的内部 Observable**：

| 操作符 | 行为 | 典型场景 |
|--------|------|----------|
| `switchMap` | **取消**之前的，切换到新的 | 搜索建议、路由参数变化加载数据 |
| `mergeMap` | **并行**执行所有 | 并行上传、互不影响的请求 |
| `concatMap` | **排队**，前一个完成后执行下一个 | 必须按顺序执行的保存操作 |
| `exhaustMap` | **忽略**新的，直到当前的完成 | 防止重复提交、登录按钮 |

**口述模板（30–60 秒）**

1. 一句话结论：四个操作符的区别在于新值到来时如何处理正在进行的内部流。
2. 关键机制：switchMap 取消旧的，mergeMap 并行，concatMap 排队，exhaustMap 忽略新的。
3. 对比或注意点：搜索用 switchMap，防重复提交用 exhaustMap，顺序保存用 concatMap。
4. 可追问时补充：mergeMap 可以传并发数限制。

**常见追问**

- 用 `switchMap` 做保存操作有什么问题？→ 快速连续保存时前面的请求被取消（前端取消了，但服务端可能已经处理），导致状态不确定。

---

### Q15. Subject、BehaviorSubject、ReplaySubject 有什么区别？

- **难度**: 中级
- **标签**: RxJS

**答案**

| 类型 | 初始值 | 新订阅者收到 |
|------|--------|--------------|
| `Subject` | 无 | 只收到订阅之后的值 |
| `BehaviorSubject` | **必须有** | 立即收到**当前值** |
| `ReplaySubject(n)` | 无 | 收到最近 n 个值 |
| `AsyncSubject` | 无 | 完成时的最后一个值 |

**冷 vs 热**：

- **冷 Observable**（如 HttpClient）：每次订阅都重新执行生产者
- **热 Observable**（Subject、DOM 事件）：多个订阅者共享同一个生产者

**Signals 时代**：组件和服务中的状态更推荐用 Signal 代替 `BehaviorSubject`。

**口述模板（30–60 秒）**

1. 一句话结论：区别在于新订阅者能收到哪些值。
2. 关键机制：BehaviorSubject 有当前值，新订阅者立即收到；ReplaySubject 重放最近 n 个值。
3. 对比或注意点：冷 Observable 每次订阅重新执行，热 Observable 共享生产者。
4. 可追问时补充：状态管理现在更推荐 Signal 代替 BehaviorSubject。

**常见追问**

- 为什么不要对外暴露 Subject？→ 外部可以调用 `next`，破坏数据流；用 `asObservable()` 暴露只读的流。

---

### Q16. `shareReplay` 怎么用？有什么坑？

- **难度**: 中级
- **标签**: RxJS, 缓存

**答案**

把冷 Observable 变成共享的，并缓存最近的值：

```ts
readonly config$ = this.http.get<Config>('/api/config').pipe(
  shareReplay({ bufferSize: 1, refCount: true }),
);
```

**坑**：

- `shareReplay(1)` 默认 `refCount: false`：所有订阅者取消后，**源订阅仍然保持**（对于不会完成的源 → 内存泄漏）
- `refCount: true`：订阅者归零时取消源订阅，下次订阅会重新执行
- 错误不会被缓存（出错后再次订阅会重新请求）
- 需要手动失效缓存时，结合 Subject 触发重新请求

**口述模板（30–60 秒）**

1. 一句话结论：shareReplay 让多个订阅者共享同一个源订阅并重放最近的值。
2. 关键机制：常用于缓存 HTTP 请求结果，避免重复请求。
3. 对比或注意点：默认 refCount 为 false，订阅者归零后源订阅仍然保持，不会完成的源会泄漏。
4. 可追问时补充：需要失效缓存时，用 Subject 触发重新请求。

**常见追问**

- `share` 和 `shareReplay` 的区别？→ `share` 不重放值，晚到的订阅者只能收到之后的值。

---

### Q17. RxJS 中怎么处理错误和重试？

- **难度**: 中级
- **标签**: RxJS, 错误处理

**答案**

```ts
this.http.get<Data>('/api/data').pipe(
  retry({ count: 3, delay: (err, i) => timer(2 ** i * 500) }),   // 指数退避
  catchError((err) => {
    this.toast.error('加载失败');
    return of(FALLBACK);           // 返回替代值，流正常完成
  }),
);
```

**关键点**：

- 错误会**终止** Observable；在 `switchMap` 内部 `catchError` 可以保护外部流（例如搜索框的流不会因为一次请求失败而永久停止）

```ts
query$.pipe(
  switchMap((q) => this.api.search(q).pipe(catchError(() => of([])))),   // 内部捕获
);
```

- 全局的错误处理：HTTP 拦截器、`ErrorHandler`

**口述模板（30–60 秒）**

1. 一句话结论：用 retry 重试，用 catchError 捕获错误并返回替代值。
2. 关键机制：错误会终止 Observable，所以在高阶操作符内部捕获可以保护外部流。
3. 对比或注意点：重试只适用于幂等请求，并要加退避。
4. 可追问时补充：全局错误处理用 HTTP 拦截器和 ErrorHandler。

**常见追问**

- 搜索框输入一次请求失败后再也不搜索了，为什么？→ 错误终止了外部的流；把 `catchError` 放在 `switchMap` 内部。

---

### Q18. 常见的 RxJS 内存泄漏和性能问题有哪些？

- **难度**: 中级
- **标签**: RxJS

**答案**

| 问题 | 原因 | 解决 |
|------|------|------|
| 订阅泄漏 | 长期存在的流（interval、Store、valueChanges）没有取消 | `takeUntilDestroyed`、`async`、`toSignal` |
| 嵌套订阅 | 在 subscribe 中再 subscribe | 用高阶映射操作符 |
| 重复请求 | 多次订阅冷 Observable | `shareReplay`、转为 Signal |
| 竞态 | 慢请求的结果覆盖新请求的结果 | `switchMap` |
| `takeUntilDestroyed` 位置不对 | 放在 `switchMap` 之前 | 放在最后 |
| 高频事件 | scroll、mousemove 未节流 | `throttleTime`、`auditTime`、`animationFrameScheduler` |

**口述模板（30–60 秒）**

1. 一句话结论：常见问题是订阅泄漏、嵌套订阅、重复请求和竞态。
2. 关键机制：用自动取消的方式管理订阅，用高阶映射操作符替代嵌套。
3. 对比或注意点：takeUntilDestroyed 要放在 pipe 的最后。
4. 可追问时补充：高频事件用节流操作符。

**常见追问**

- 怎么发现订阅泄漏？→ Chrome 内存快照对比，查找已销毁组件的残留实例（Detached 节点、组件实例数量）。

---

## 四、依赖注入进阶

### Q19. 依赖注入的解析规则是什么？解析修饰符有哪些？

- **难度**: 中级
- **标签**: 依赖注入

**答案**

**两棵注入器树**：

- **元素注入器（Element Injector）**：由组件和指令的 `providers` 创建，跟随 DOM 结构
- **环境注入器（Environment Injector）**：根注入器、路由注入器（`providers`）、平台注入器

**解析过程**：

1. 从当前组件 / 指令的元素注入器开始，**沿 DOM 结构向上**查找
2. 到达根组件后，转到**环境注入器**树，从最近的（例如懒加载路由的）向上到根、平台
3. 都找不到 → 报错 `NullInjectorError`

**解析修饰符**：

```ts
inject(Logger, { optional: true });   // 找不到返回 null
inject(Logger, { self: true });       // 只在当前元素注入器查找
inject(Logger, { skipSelf: true });   // 从父级开始查找
inject(Logger, { host: true });       // 查找到宿主组件为止
```

**口述模板（30–60 秒）**

1. 一句话结论：先沿元素注入器树向上查找，再沿环境注入器树向上查找。
2. 关键机制：元素注入器跟随 DOM 结构，环境注入器包括根注入器和路由注入器。
3. 对比或注意点：optional、self、skipSelf、host 修饰符控制查找范围。
4. 可追问时补充：skipSelf 常用于嵌套的同类组件获取父级实例，例如嵌套菜单。

**常见追问**

- 懒加载路由中的组件注入服务，查找顺序是什么？→ 组件的元素注入器 → 祖先组件 → 路由的环境注入器 → 根注入器 → 平台注入器。

---

### Q20. `providedIn: 'root'` 为什么可以摇树？`InjectionToken` 怎么提供默认值？

- **难度**: 中级
- **标签**: 依赖注入, 摇树

**答案**

**摇树**：`providedIn: 'root'` 把提供信息放在**服务自身**，而不是在模块的 `providers` 数组中引用服务。如果没有任何地方 `inject` 这个服务，它就不会被引用，构建工具可以移除。

**`InjectionToken` 的工厂**：

```ts
export const WINDOW = new InjectionToken<Window>('WINDOW', {
  providedIn: 'root',
  factory: () => inject(DOCUMENT).defaultView!,
});

export const PAGE_SIZE = new InjectionToken<number>('PAGE_SIZE', {
  providedIn: 'root',
  factory: () => 20,   // 默认值，使用方可以覆盖
});
```

**用途**：抽象浏览器对象（便于 SSR 和测试）、配置项、插件机制（`multi: true`）。

**口述模板（30–60 秒）**

1. 一句话结论：providedIn root 把提供信息放在服务自身，没被注入就不会被打包。
2. 关键机制：模块 providers 数组会直接引用服务，即使没被使用也无法移除。
3. 对比或注意点：InjectionToken 可以通过 factory 提供默认值，使用方可以覆盖。
4. 可追问时补充：常用来抽象 window 等浏览器对象，便于 SSR 和测试。

**常见追问**

- 为什么不直接用全局的 `window`？→ SSR 中没有 `window`；通过令牌注入便于在服务端提供替代实现和在测试中模拟。

---

### Q21. 组件级 providers 有哪些实际用途？

- **难度**: 中级
- **标签**: 依赖注入, 架构

**答案**

1. **组件局部状态**：每个组件实例一个 Store（列表页的筛选条件、复杂表单的状态）

```ts
@Component({ providers: [OrderListStore] })
export class OrderList {
  store = inject(OrderListStore);   // 组件销毁时 Store 也销毁
}
```

2. **复合组件通信**：父组件提供服务，子组件注入（如 Tabs 和 Tab、Accordion）
3. **覆盖实现**：在某个子树中替换服务（如不同的日期格式、不同的主题）
4. **`viewProviders`**：只对组件自己的视图可见，投影进来的内容无法注入（库组件避免泄漏内部服务）

**口述模板（30–60 秒）**

1. 一句话结论：组件级 providers 用于局部状态、复合组件通信和局部覆盖实现。
2. 关键机制：每个组件实例有自己的服务实例，随组件销毁。
3. 对比或注意点：viewProviders 只对自身视图可见，投影的内容无法注入。
4. 可追问时补充：Tabs 和 Tab 这类复合组件通过父组件提供的服务协作。

**常见追问**

- 组件级服务中的 `ngOnDestroy` 会被调用吗？→ 会，组件级服务随组件销毁，可以在其中清理资源（或注入 `DestroyRef`）。

---

### Q22. v22 的 `@Service` 和 `injectAsync` 是什么？

- **难度**: 中级
- **标签**: 依赖注入, 新特性

**答案**

（v22 新增，细节以官方文档为准）

**`@Service()`**：`@Injectable({ providedIn: 'root' })` 的更直观写法，用于定义全局单例服务；需要更复杂配置时仍然使用 `@Injectable`。

**`injectAsync`**：**异步注入**，让大型服务也可以按需加载和代码拆分：

```ts
export class Report {
  private exporter = injectAsync(() => import('./report-exporter'), { prefetch: onIdle });

  async export() {
    const exporter = await this.exporter();   // 首次调用时才加载
    exporter.export();
  }
}
```

**场景**：体积大但只在少数操作中使用的服务（导出 Excel、PDF 生成、富文本处理）。

**口述模板（30–60 秒）**

1. 一句话结论：@Service 是根级单例服务的简写，injectAsync 支持服务按需加载。
2. 关键机制：injectAsync 接收动态 import，首次使用时才加载服务代码，可以设置预取。
3. 对比或注意点：之前只有组件和路由可以懒加载，服务不行。
4. 可追问时补充：适合导出、PDF 生成这类体积大但使用少的服务。

**常见追问**

- `injectAsync` 和直接在方法中 `import()` 有什么区别？→ `injectAsync` 仍然经过 DI，服务可以注入其他依赖，并支持预取策略。

---

## 五、路由与表单进阶

### Q23. 路由的预加载策略有哪些？怎么自定义？

- **难度**: 中级
- **标签**: 路由, 性能

**答案**

| 策略 | 行为 |
|------|------|
| 默认（不预加载） | 访问时才加载 |
| `PreloadAllModules` | 首屏加载后预加载所有懒加载路由 |
| 自定义策略 | 按路由配置、网络状况决定 |

```ts
@Injectable({ providedIn: 'root' })
export class SelectivePreload implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<unknown>) {
    const conn = (navigator as any).connection;
    if (conn?.saveData || /2g/.test(conn?.effectiveType ?? '')) return of(null);
    return route.data?.['preload'] ? load() : of(null);
  }
}
// provideRouter(routes, withPreloading(SelectivePreload))
```

其他方案：**鼠标悬停时预加载**（社区的 quicklink / 自定义指令）。

**口述模板（30–60 秒）**

1. 一句话结论：预加载策略决定懒加载路由是否在后台提前下载。
2. 关键机制：可以预加载全部，或者实现自定义策略按路由配置和网络状况决定。
3. 对比或注意点：弱网和省流量模式下不应该预加载。
4. 可追问时补充：也可以在链接悬停或进入视口时预加载。

**常见追问**

- `canMatch` 守卫和预加载的关系？→ 被 `canMatch` 拒绝的路由不会被预加载（`canActivate` 不阻止预加载）。

---

### Q24. 解析器（Resolver）和在组件中加载数据，怎么选？

- **难度**: 中级
- **标签**: 路由, 数据加载

**答案**

| 方面 | Resolver | 组件内加载（resource / httpResource） |
|------|----------|----------------------------------------|
| 时机 | **导航完成前**获取数据 | 组件渲染后获取 |
| 体验 | 数据到达前停留在旧页面（需要全局加载指示） | 立即切换页面，显示骨架屏 |
| 错误处理 | 可以在导航前重定向 | 在组件内展示错误 |
| 适合 | 必须有数据才能渲染、需要 404 重定向 | 大多数页面，体验更好 |

```ts
export const productResolver: ResolveFn<Product> = (route) => {
  const api = inject(ProductApi);
  const router = inject(Router);
  return api.get(route.paramMap.get('id')!).pipe(
    catchError(() => of(new RedirectCommand(router.parseUrl('/not-found')))),
  );
};
```

**口述模板（30–60 秒）**

1. 一句话结论：解析器在导航完成前获取数据，组件内加载则先切换页面再获取。
2. 关键机制：解析器适合必须有数据才能渲染、需要在导航前重定向的场景。
3. 对比或注意点：解析器会让用户停留在旧页面，体验上需要全局加载指示。
4. 可追问时补充：大多数页面推荐组件内加载配合骨架屏。

**常见追问**

- 解析器很慢怎么办？→ 改为组件内加载；或者解析器只获取关键的少量数据，其余在组件中加载。

---

### Q25. 怎么实现一个自定义表单控件（ControlValueAccessor）？

- **难度**: 中级
- **标签**: 表单, CVA

**答案**

让自定义组件可以使用 `formControlName` / `ngModel`：

```ts
@Component({
  selector: 'app-star-rating',
  template: `
    @for (s of stars; track s) {
      <button type="button" [disabled]="disabled()" (click)="select(s)" (blur)="onTouched()">
        {{ s <= value() ? '★' : '☆' }}
      </button>
    }
  `,
  providers: [{ provide: NG_VALUE_ACCESSOR, useExisting: forwardRef(() => StarRating), multi: true }],
})
export class StarRating implements ControlValueAccessor {
  stars = [1, 2, 3, 4, 5];
  value = signal(0);
  disabled = signal(false);
  private onChange: (v: number) => void = () => {};
  onTouched: () => void = () => {};

  writeValue(v: number | null) { this.value.set(v ?? 0); }       // 表单 → 组件
  registerOnChange(fn: (v: number) => void) { this.onChange = fn; }
  registerOnTouched(fn: () => void) { this.onTouched = fn; }
  setDisabledState(d: boolean) { this.disabled.set(d); }

  select(v: number) {
    this.value.set(v);
    this.onChange(v);                                             // 组件 → 表单
  }
}
```

**Signal Forms** 中对应的是 `FormValueControl` 接口（实现 `value` model 即可），并且可以和响应式表单互操作。

**口述模板（30–60 秒）**

1. 一句话结论：实现 ControlValueAccessor 接口并注册 NG_VALUE_ACCESSOR，让组件接入表单。
2. 关键机制：writeValue 接收表单的值，registerOnChange 注册的回调把组件的值传回表单。
3. 对比或注意点：要处理 touched 和 disabled 状态。
4. 可追问时补充：Signal Forms 中用 FormValueControl，只需实现 value 模型。

**常见追问**

- `writeValue` 中调用 `onChange` 会怎样？→ 会把值又传回表单，可能导致循环或把表单标记为 dirty；`writeValue` 中不应该调用。

---

### Q26. Signal Forms 是什么？和响应式表单相比有什么优势？

- **难度**: 中级
- **标签**: 表单, Signals

**答案**

v21 实验、**v22 稳定**的新表单 API：以 **signal 作为数据模型**，用声明式的 schema 定义校验和状态。

```ts
import { form, Field, required, email, minLength } from '@angular/forms/signals';

export class Signup {
  model = signal({ email: '', password: '' });

  signupForm = form(this.model, (p) => {
    required(p.email, { message: '请输入邮箱' });
    email(p.email);
    minLength(p.password, 8);
  });
}
```

```html
<input [field]="signupForm.email" />
@if (signupForm.email().invalid() && signupForm.email().touched()) {
  @for (e of signupForm.email().errors(); track e) { <small>{{ e.message }}</small> }
}
```

（模板绑定指令的具体名称以官方文档为准。）

**优势**：

- 模型就是 signal，**单一数据源**，不需要在表单值和组件状态之间同步
- 完全类型化，字段路径有类型推断
- 校验是声明式的，可以根据其他字段条件启用（`required(p.x, { when: ... })`）
- 与 Zoneless 和 OnPush 自然配合（响应式表单的 `setValue` 不会自动调度检测）

**迁移**：`compatForm` 可以把现有的 `FormGroup` 嵌入 Signal Form；`SignalFormControl` 可以把 Signal Form 的字段作为 `FormControl` 用在现有的响应式表单中，**渐进迁移**。

**口述模板（30–60 秒）**

1. 一句话结论：Signal Forms 以 signal 作为数据模型，用声明式 schema 定义校验，v22 起稳定。
2. 关键机制：表单状态（值、有效性、touched）都是信号，模型是单一数据源。
3. 对比或注意点：完全类型化，与 Zoneless 自然配合，响应式表单的修改不会自动触发检测。
4. 可追问时补充：可以通过 compatForm 和 SignalFormControl 与响应式表单互操作，渐进迁移。

**常见追问**

- Signal Forms 不支持哪些响应式表单的命令式 API？→ 例如动态 `addValidators`、`setErrors`、`enable()/disable()`；这些状态应该从信号和规则中派生。

---

## 六、渲染、测试与工程

### Q27. `@defer` 的原理和最佳实践？

- **难度**: 中级
- **标签**: @defer, 性能

**答案**

**原理**：编译器把 `@defer` 块中**只在该块中使用的**组件、指令、管道提取为动态 `import()`，生成单独的代码块；触发条件满足时加载并渲染。

**最佳实践**：

- 用于首屏以下的重组件（图表、评论、富文本编辑器、地图）
- `on viewport` + `prefetch on idle`：空闲时预取，进入视口时渲染
- `@placeholder` 的尺寸与实际内容一致，避免布局偏移（CLS）
- `@loading` 设置 `after` 和 `minimum`，避免闪烁
- 首屏的 LCP 元素**不要**放在 `@defer` 中
- 被延迟的组件必须是 Standalone，并且不能在块外被引用

**SSR 中**：服务端渲染 `@placeholder`；配合**增量水合**（`hydrate on viewport`）可以在服务端渲染完整内容，但延迟到触发时才水合。

**口述模板（30–60 秒）**

1. 一句话结论：@defer 由编译器把块中的依赖拆分成独立代码块，满足触发条件时加载。
2. 关键机制：常用 on viewport 加 prefetch on idle，空闲时预取、可见时渲染。
3. 对比或注意点：placeholder 尺寸要和实际内容一致，LCP 元素不能放在 @defer 中。
4. 可追问时补充：SSR 中配合增量水合，可以服务端渲染完整内容但延迟水合。

**常见追问**

- `@defer` 和路由懒加载的区别？→ 路由懒加载按页面拆分，`@defer` 按页面中的区块拆分，粒度更细。

---

### Q28. Angular SSR 和水合是怎么工作的？

- **难度**: 中级
- **标签**: SSR, 水合

**答案**

**SSR**（`@angular/ssr`）：服务端渲染出 HTML 返回给浏览器，提升首屏和 SEO。

**水合（Hydration，v16 起）**：客户端**复用服务端渲染的 DOM**，只附加事件监听和状态，而不是销毁后重新创建 DOM（旧版本的方式会导致闪烁）。

```ts
bootstrapApplication(App, {
  providers: [provideClientHydration(withEventReplay(), withIncrementalHydration())],
});
```

**相关特性**：

- **事件重放**：水合完成前用户的点击被记录，水合后重放
- **增量水合**（v20 稳定）：配合 `@defer (hydrate on viewport)`，按需水合页面的部分区域
- **HTTP 传输缓存**：服务端请求的数据序列化到页面中，客户端不再重复请求
- **混合渲染**：按路由配置 SSR、SSG（预渲染）或 CSR（`RenderMode`）

**水合失败的常见原因**：服务端和客户端渲染的 DOM 不一致；直接操作 DOM；无效的 HTML 结构（`<p>` 中嵌套 `<div>`）。

**口述模板（30–60 秒）**

1. 一句话结论：SSR 在服务端渲染 HTML，水合让客户端复用这些 DOM 并附加事件。
2. 关键机制：事件重放保证水合前的点击不丢失，HTTP 传输缓存避免重复请求。
3. 对比或注意点：增量水合可以按需水合页面的部分区域。
4. 可追问时补充：服务端和客户端 DOM 不一致、直接操作 DOM 会导致水合失败。

**常见追问**

- SSR 中访问 `window` 报错怎么办？→ 用 `afterNextRender` 包裹浏览器专用代码，或注入 `PLATFORM_ID` 判断，或通过令牌抽象浏览器对象。

---

### Q29. 指令组合 API（hostDirectives）解决什么问题？

- **难度**: 中级
- **标签**: 指令, 复用

**答案**

把多个指令的行为**组合**到一个组件或指令上，而不需要使用者在模板中手动添加：

```ts
@Component({
  selector: 'app-menu-item',
  hostDirectives: [
    { directive: CdkMenuItem, inputs: ['cdkMenuItemDisabled: disabled'], outputs: ['cdkMenuItemTriggered: triggered'] },
    FocusRing,
  ],
  template: `<ng-content />`,
})
export class MenuItem {}
```

- 宿主指令**静态**应用（编译时确定），可以选择暴露哪些输入和输出
- v22 起：匹配多次时自动去重；同一个输入或输出以多个名称暴露时报错

**场景**：组件库中的可复用行为（焦点管理、拖拽、键盘导航、无障碍属性），类似组合式的 mixin。

**口述模板（30–60 秒）**

1. 一句话结论：hostDirectives 把多个指令的行为组合到组件上，实现行为复用。
2. 关键机制：编译时静态应用，可以选择暴露哪些输入和输出。
3. 对比或注意点：使用者不需要在模板中手动添加指令。
4. 可追问时补充：组件库常用它组合焦点、键盘导航和无障碍行为。

**常见追问**

- 和继承相比的优点？→ 组合比继承更灵活，可以组合多个行为，避免深层继承链。

---

### Q30. 怎么测试使用了 Signals、HTTP 和路由的组件？

- **难度**: 中级
- **标签**: 测试

**答案**

```ts
describe('UserPage', () => {
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting(), provideRouter([])],
    });
    httpMock = TestBed.inject(HttpTestingController);
  });

  it('加载并显示用户', async () => {
    const fixture = TestBed.createComponent(UserPage);
    fixture.componentRef.setInput('id', '42');      // 设置信号输入
    await fixture.whenStable();

    httpMock.expectOne('/api/users/42').flush({ id: '42', name: 'Ada' });
    await fixture.whenStable();

    expect(fixture.nativeElement.textContent).toContain('Ada');
  });

  afterEach(() => httpMock.verify());   // 确认没有未处理的请求
});
```

- **路由测试**：`RouterTestingHarness`
- **组件库测试**：CDK 的**组件测试工具（Harness）**，通过语义化的 API 操作组件，不依赖内部 DOM 结构
- 服务的单元测试：`TestBed.inject` 或直接 `new`
- 测试 effect：`TestBed.tick()` 执行挂起的 effect

**口述模板（30–60 秒）**

1. 一句话结论：用 TestBed 配置依赖，HttpTestingController 模拟请求，setInput 设置信号输入。
2. 关键机制：用 whenStable 等待变更检测和异步任务完成。
3. 对比或注意点：路由用 RouterTestingHarness，组件库用 Harness 避免依赖内部结构。
4. 可追问时补充：afterEach 中调用 verify 确认没有遗漏的请求。

**常见追问**

- 为什么推荐 Harness？→ 组件库内部 DOM 变化时测试不需要修改；测试代码更接近用户的操作。

---

### Q31. Angular 的构建工具发生了什么变化？

- **难度**: 中级
- **标签**: 构建

**答案**

| 时间 | 变化 |
|------|------|
| v17 前 | Webpack（`@angular-devkit/build-angular:browser`） |
| v17 | **application builder（esbuild + Vite 开发服务器）**成为新项目默认，构建速度显著提升 |
| v21 | Vitest 成为默认测试运行器（取代 Karma） |
| **v22** | **Webpack 相关构建器（`@angular-devkit/build-angular`、`@ngtools/webpack`）弃用**；官方关注基于 TSGo（TypeScript 原生编译器）的支持 |

**迁移**：`ng update` 提供迁移到 application builder 的脚本；使用了自定义 Webpack 配置的项目需要找替代方案（esbuild 插件，或迁移到 Nx / Rspack 等方案）。

**口述模板（30–60 秒）**

1. 一句话结论：Angular 从 Webpack 迁移到了基于 esbuild 和 Vite 的 application builder。
2. 关键机制：v17 起是新项目默认，构建速度显著提升。
3. 对比或注意点：v22 弃用了 Webpack 相关构建器，有自定义 Webpack 配置的项目需要迁移。
4. 可追问时补充：测试运行器从 Karma 换成了 Vitest。

**常见追问**

- 用了 Module Federation（基于 Webpack）的项目怎么办？→ 可以使用 Native Federation（基于 esbuild 和导入映射）或 Module Federation 2.0 对其他构建工具的支持。

---

### Q32. Angular Material 和 CDK 是什么关系？

- **难度**: 中级
- **标签**: 组件库

**答案**

- **CDK（Component Dev Kit）**：**无样式的行为原语**——Overlay（浮层定位）、A11y（焦点陷阱、实时播报）、拖放、虚拟滚动、Portal、表格、Stepper
- **Angular Material**：基于 CDK、遵循 Material Design 的完整组件库
- **Angular Aria**（v21 引入，v22 稳定）：一组**无样式、可访问的指令**（菜单、列表框、标签页等），用于自己构建设计系统

**选型**：

- 使用 Material Design → Angular Material
- 自己的设计系统 → CDK + Angular Aria，或第三方（PrimeNG、ng-zorro-antd、Spartan）

**口述模板（30–60 秒）**

1. 一句话结论：CDK 提供无样式的行为原语，Material 是基于 CDK 的完整组件库。
2. 关键机制：CDK 提供浮层、无障碍、拖放、虚拟滚动等能力。
3. 对比或注意点：自己的设计系统可以基于 CDK 和 Angular Aria 构建。
4. 可追问时补充：国内常用的还有 ng-zorro-antd 和 PrimeNG。

**常见追问**

- 虚拟滚动怎么用？→ `cdk-virtual-scroll-viewport` + `*cdkVirtualFor`，需要固定的项目高度（或自定义滚动策略）。
