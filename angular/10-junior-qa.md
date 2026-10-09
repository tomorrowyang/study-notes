# Angular 初级 Q&A

> 目标：能用 Angular 独立开发功能，讲清组件、模板、数据绑定、指令、管道、生命周期、依赖注入、路由、表单和 HTTP 的基本用法。示例基于 Angular v20+ 的 Standalone + Signals 写法。

---

## 一、组件与模板

### Q1. Angular 是什么？和 React、Vue 有什么区别？

- **难度**: 初级
- **标签**: 概述

**答案**

Angular 是 Google 维护的**完整前端框架**（不是库），内置路由、表单、HTTP、依赖注入、测试、CLI、SSR 等，基于 TypeScript。

| 方面 | Angular | React | Vue |
|------|---------|-------|-----|
| 定位 | 全家桶框架，约定多 | UI 库，生态自选 | 渐进式框架 |
| 语言 | TypeScript（强制） | JS / TS | JS / TS |
| 模板 | HTML 模板 + 专用语法 | JSX | HTML 模板 / JSX |
| 响应式 | Signals（+ RxJS） | 状态 + 重新渲染 | Proxy 响应式 |
| 依赖注入 | **内置，核心机制** | 无（Context 替代） | provide / inject（简化版） |
| 编译 | AOT 编译模板为指令代码 | 运行时 + 编译器优化 | 模板编译 + 运行时 |
| 适合 | 大型企业应用、大团队 | 各类应用 | 各类应用 |

**口述模板（30–60 秒）**

1. 一句话结论：Angular 是一个完整的、基于 TypeScript 的前端框架，内置路由、表单、HTTP 和依赖注入。
2. 关键机制：模板经过 AOT 编译，响应式基于 Signals，依赖注入是核心机制。
3. 对比或注意点：React 是 UI 库需要自选生态，Angular 约定更多、一致性更强。
4. 可追问时补充：适合大型团队和长期维护的企业级应用。

**常见追问**

- AngularJS 和 Angular 是什么关系？→ AngularJS（1.x）是旧框架，2016 年的 Angular 2+ 完全重写，两者不兼容；AngularJS 已停止维护。

---

### Q2. 一个 Angular 组件由哪些部分组成？

- **难度**: 初级
- **标签**: 组件

**答案**

```ts
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-counter',          // 在模板中使用的标签名
  template: `
    <p>Count: {{ count() }}</p>
    <button (click)="increment()">+1</button>
  `,
  styles: `p { color: steelblue; }`, // 默认作用域隔离（Emulated）
})
export class Counter {
  count = signal(0);

  increment() {
    this.count.update((c) => c + 1);
  }
}
```

- **类**：状态和行为
- **模板**：视图（`template` 或 `templateUrl`）
- **样式**：默认通过属性选择器模拟封装（`ViewEncapsulation.Emulated`）
- **元数据**：`@Component` 装饰器的配置（selector、imports、providers 等）

v19 起组件**默认是 Standalone**，直接在 `imports` 中声明依赖的组件、指令和管道。

**口述模板（30–60 秒）**

1. 一句话结论：组件由类、模板、样式和装饰器元数据组成。
2. 关键机制：类保存状态和方法，模板通过绑定使用它们，样式默认按组件隔离。
3. 对比或注意点：v19 起组件默认是 Standalone，在 imports 中声明依赖。
4. 可追问时补充：样式封装有 Emulated、ShadowDom、None 三种模式。

**常见追问**

- 样式封装怎么实现的？→ Emulated 模式下编译器给元素添加 `_ngcontent-xxx` 属性，给样式选择器加上对应的属性选择器。

---

### Q3. Angular 的数据绑定有哪几种？

- **难度**: 初级
- **标签**: 模板, 绑定

**答案**

| 类型 | 语法 | 方向 |
|------|------|------|
| 插值 | `{{ value }}` | 组件 → 视图 |
| 属性绑定 | `[src]="url"`、`[disabled]="isBusy()"` | 组件 → 视图 |
| 特性（attribute）绑定 | `[attr.aria-label]="label"` | 组件 → 视图 |
| 类 / 样式绑定 | `[class.active]="on"`、`[style.width.px]="w"` | 组件 → 视图 |
| 事件绑定 | `(click)="save()"`、`(keyup.enter)="search()"` | 视图 → 组件 |
| 双向绑定 | `[(ngModel)]="name"`、`[(value)]="v"` | 双向 |

**属性（property）和特性（attribute）的区别**：属性是 DOM 对象上的值，特性是 HTML 标签上的值；没有对应 DOM 属性的（`aria-*`、`colspan`）要用 `attr.`。

双向绑定是语法糖：`[(x)]="v"` 等价于 `[x]="v" (xChange)="v = $event"`。

**口述模板（30–60 秒）**

1. 一句话结论：数据绑定有插值、属性绑定、事件绑定和双向绑定。
2. 关键机制：方括号表示从组件到视图，圆括号表示从视图到组件。
3. 对比或注意点：没有对应 DOM 属性的值要用 attr. 绑定特性。
4. 可追问时补充：双向绑定是属性绑定加 xxxChange 事件的语法糖。

**常见追问**

- 为什么叫「香蕉在盒子里」？→ `[()]` 的形状像香蕉在盒子里，方便记住括号的顺序。

---

### Q4. 新的控制流语法 `@if`、`@for`、`@switch` 怎么用？

- **难度**: 初级
- **标签**: 模板, 控制流

**答案**

v17 引入的内置控制流，取代 `*ngIf`、`*ngFor`、`*ngSwitch`：

```html
@if (user(); as u) {
  <p>Hi, {{ u.name }}</p>
} @else if (loading()) {
  <app-spinner />
} @else {
  <a routerLink="/login">登录</a>
}

@for (item of items(); track item.id; let i = $index, last = $last) {
  <li [class.last]="last">{{ i + 1 }}. {{ item.name }}</li>
} @empty {
  <li>暂无数据</li>
}

@switch (status()) {
  @case ('pending')
  @case ('processing') { <span>处理中</span> }   <!-- v22 起支持多个 case 共享 -->
  @case ('done') { <span>完成</span> }
  @default { <span>未知</span> }
}
```

**相比结构指令的优点**：不需要导入 `CommonModule`；类型收窄更好；`@for` 强制 `track`，性能更好；`@empty` 内置。

**口述模板（30–60 秒）**

1. 一句话结论：v17 引入了内置的 @if、@for、@switch 控制流，取代结构指令。
2. 关键机制：@for 必须指定 track，用于复用 DOM 节点。
3. 对比或注意点：不需要导入 CommonModule，类型收窄更好，内置 @empty。
4. 可追问时补充：可以用 `ng generate @angular/core:control-flow` 自动迁移。

**常见追问**

- `track` 应该写什么？→ 数据中唯一且稳定的标识（如 `item.id`）；没有 ID 的静态列表可以用 `$index`，但会导致增删时的错误复用。

---

### Q5. 指令有哪几类？怎么写一个属性指令？

- **难度**: 初级
- **标签**: 指令

**答案**

- **组件**：带模板的指令
- **属性指令**：改变元素的外观或行为（`ngClass`、`ngStyle`、自定义高亮）
- **结构指令**：改变 DOM 结构（`*ngIf`、`*ngFor`，已被控制流取代，但自定义结构指令仍然有用）

```ts
import { Directive, ElementRef, inject, input } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  host: {
    '(mouseenter)': 'setColor(color())',
    '(mouseleave)': 'setColor(null)',
  },
})
export class Highlight {
  color = input('yellow', { alias: 'appHighlight' });
  private el = inject(ElementRef<HTMLElement>);

  setColor(value: string | null) {
    this.el.nativeElement.style.backgroundColor = value ?? '';
  }
}
```

使用：`<p appHighlight="lightblue">悬停高亮</p>`

**推荐使用 `host` 元数据**而不是 `@HostListener` / `@HostBinding`（官方风格指南的推荐）。

**口述模板（30–60 秒）**

1. 一句话结论：指令分为组件、属性指令和结构指令。
2. 关键机制：属性指令通过 selector 匹配元素，用 host 元数据监听事件和绑定属性。
3. 对比或注意点：内置结构指令已被控制流取代。
4. 可追问时补充：直接操作 DOM 时注意 SSR 兼容，可以用 Renderer2。

**常见追问**

- 什么是宿主指令（hostDirectives）？→ 在组件或指令的元数据中组合其他指令，实现行为复用（类似组合式的 mixin）。

---

### Q6. 管道（Pipe）是什么？纯管道和非纯管道的区别？

- **难度**: 初级
- **标签**: 管道

**答案**

管道在模板中转换显示的值：`{{ price | currency:'CNY' }}`、`{{ date | date:'yyyy-MM-dd' }}`。

```ts
@Pipe({ name: 'truncate' })
export class TruncatePipe implements PipeTransform {
  transform(value: string, max = 20): string {
    return value.length > max ? value.slice(0, max) + '…' : value;
  }
}
```

| 类型 | 何时重新计算 | 适用 |
|------|--------------|------|
| **纯管道**（默认） | 输入的**引用**变化时 | 绝大多数场景，有缓存效果 |
| 非纯管道（`pure: false`） | **每次变更检测**都执行 | 依赖对象内部变化（如 `async` 管道） |

**注意**：纯管道对于数组的 `push` 不会重新计算（引用没变）→ 用不可变更新。

**口述模板（30–60 秒）**

1. 一句话结论：管道用于在模板中转换显示的值。
2. 关键机制：纯管道只在输入引用变化时重新计算，相当于缓存。
3. 对比或注意点：非纯管道每次变更检测都执行，开销大。
4. 可追问时补充：模板中的复杂计算可以用纯管道或 computed 代替方法调用。

**常见追问**

- 为什么模板中调用方法可能有性能问题？→ 方法在每次变更检测时都执行；纯管道和 `computed` 有缓存。

---

### Q7. 组件的生命周期钩子有哪些？

- **难度**: 初级
- **标签**: 生命周期

**答案**

| 钩子 | 时机 | 常见用途 |
|------|------|----------|
| `constructor` | 创建实例 | 注入依赖（`inject()`），不要访问输入 |
| `ngOnChanges` | 输入属性变化（首次在 OnInit 之前） | 响应输入变化（信号输入可以用 computed 替代） |
| `ngOnInit` | 首次输入设置后，一次 | 初始化、请求数据 |
| `ngDoCheck` | 每次变更检测 | 自定义检测（慎用） |
| `ngAfterContentInit / Checked` | 投影内容初始化 / 检查后 | 访问 `contentChild` |
| `ngAfterViewInit / Checked` | 视图初始化 / 检查后 | 访问 `viewChild`、操作 DOM |
| `ngOnDestroy` | 销毁前 | 清理（定时器、订阅） |

**新的 API**：

- `afterNextRender` / `afterEveryRender`：渲染后执行（只在浏览器中运行，适合 DOM 操作和第三方库初始化，对 SSR 友好）
- `DestroyRef.onDestroy`、`takeUntilDestroyed()`：替代 `ngOnDestroy` 进行清理

**口述模板（30–60 秒）**

1. 一句话结论：常用的钩子是 ngOnInit、ngAfterViewInit 和 ngOnDestroy。
2. 关键机制：OnInit 中输入已经可用，AfterViewInit 中视图子元素可用，OnDestroy 中清理资源。
3. 对比或注意点：操作 DOM 推荐 afterNextRender，它只在浏览器执行，兼容 SSR。
4. 可追问时补充：清理可以用 DestroyRef 或 takeUntilDestroyed。

**常见追问**

- 为什么在 `ngAfterViewInit` 中修改绑定的值会报 `ExpressionChangedAfterItHasBeenCheckedError`？→ 开发模式下 Angular 会再检查一次，发现值在检查后被修改，说明数据流不是单向的。

---

### Q8. 内容投影（ng-content）怎么用？

- **难度**: 初级
- **标签**: 组件, 投影

**答案**

类似 Vue 的插槽：

```ts
@Component({
  selector: 'app-card',
  template: `
    <header><ng-content select="[card-title]" /></header>
    <section><ng-content /></section>          <!-- 默认投影 -->
    <footer><ng-content select="app-card-actions" /></footer>
  `,
})
export class Card {}
```

```html
<app-card>
  <h2 card-title>标题</h2>
  <p>正文内容</p>
  <app-card-actions><button>确定</button></app-card-actions>
</app-card>
```

**注意**：`ng-content` 投影的内容**始终会被创建**（即使被放在 `@if` 中不显示）；需要按条件或多次渲染时，使用 `ng-template` + `ngTemplateOutlet`。

**口述模板（30–60 秒）**

1. 一句话结论：ng-content 把父组件传入的内容投影到子组件模板中。
2. 关键机制：select 属性按选择器把内容分配到不同位置。
3. 对比或注意点：投影的内容始终会被创建，条件渲染要用 ng-template。
4. 可追问时补充：通过 contentChild 可以在子组件中访问投影的内容。

**常见追问**

- `ng-template` 和 `ng-container` 的区别？→ `ng-template` 定义一个模板，不会直接渲染；`ng-container` 是不产生 DOM 元素的分组容器。

---

### Q9. 模板引用变量和 `viewChild` 怎么用？

- **难度**: 初级
- **标签**: 模板, 查询

**答案**

```ts
@Component({
  template: `
    <input #searchBox />
    <button (click)="search(searchBox.value)">搜索</button>
    <app-chart #chart />
  `,
})
export class Page {
  // 信号式查询（v17.2+）
  searchInput = viewChild.required<ElementRef<HTMLInputElement>>('searchBox');
  chart = viewChild(Chart);

  constructor() {
    afterNextRender(() => this.searchInput().nativeElement.focus());
  }
}
```

- `#name` 声明模板引用变量，可以在模板中直接使用
- `viewChild` / `viewChildren`：查询组件自己模板中的元素；`contentChild` / `contentChildren`：查询投影内容
- 信号式查询返回 Signal，结果变化时自动更新

**口述模板（30–60 秒）**

1. 一句话结论：模板引用变量用 # 声明，可以在模板中直接访问元素或组件实例。
2. 关键机制：viewChild 在类中查询视图元素，信号式查询返回 Signal。
3. 对比或注意点：viewChild 查询自己的模板，contentChild 查询投影内容。
4. 可追问时补充：视图渲染后才能访问 DOM，适合在 afterNextRender 中使用。

**常见追问**

- 为什么在 `@if` 中的元素查询结果可能是 undefined？→ 条件为假时元素不存在；信号式查询会在元素出现后更新。

---

### Q10. `@let` 和 `@defer` 是什么？

- **难度**: 初级
- **标签**: 模板

**答案**

**`@let`**（v18.1+）：在模板中声明局部变量。

```html
@let user = user$ | async;
@let fullName = user?.firstName + ' ' + user?.lastName;
<h1>{{ fullName }}</h1>
```

**`@defer`**（v17+）：延迟加载模板中的一部分，被延迟的组件会被**自动拆分为单独的代码块**。

```html
@defer (on viewport; prefetch on idle) {
  <app-comments [postId]="id()" />
} @placeholder (minimum 500ms) {
  <div class="skeleton"></div>
} @loading (after 100ms; minimum 300ms) {
  <app-spinner />
} @error {
  <p>加载失败</p>
}
```

触发条件：`on idle`（默认）、`on viewport`、`on interaction`、`on hover`、`on timer(2s)`、`on immediate`、`when 条件`。

**口述模板（30–60 秒）**

1. 一句话结论：@let 声明模板局部变量，@defer 延迟加载模板中的一部分。
2. 关键机制：@defer 中的组件会被自动拆分成独立的代码块，按触发条件加载。
3. 对比或注意点：@defer 支持 placeholder、loading、error 状态，以及 prefetch 预取。
4. 可追问时补充：SSR 时 @defer 渲染 placeholder，配合增量水合可以延迟水合。

**常见追问**

- `@defer` 中的组件在其他地方被直接引用了会怎样？→ 无法拆分，仍然会被打包到主块中。

---

## 二、组件通信

### Q11. 父子组件怎么通信？

- **难度**: 初级
- **标签**: 组件通信

**答案**

```ts
@Component({ selector: 'app-rating', template: `...` })
export class Rating {
  value = input.required<number>();          // 父 → 子
  max = input(5);
  changed = output<number>();                // 子 → 父
  select(v: number) { this.changed.emit(v); }
}
```

```html
<app-rating [value]="score()" (changed)="onRate($event)" />
```

| 方式 | 适用 |
|------|------|
| `input()` / `output()` | 父子 |
| `model()` | 父子双向绑定 |
| `viewChild` | 父组件直接调用子组件的方法 |
| 共享服务（Signals / RxJS） | 兄弟、跨层级 |
| 路由参数、状态管理库 | 跨页面 |

**口述模板（30–60 秒）**

1. 一句话结论：父传子用 input，子传父用 output，双向用 model。
2. 关键机制：input 返回只读 Signal，output 通过 emit 发出事件。
3. 对比或注意点：兄弟和跨层级通信用共享服务。
4. 可追问时补充：父组件可以通过 viewChild 调用子组件的方法，但耦合较高。

**常见追问**

- `input()` 和 `@Input()` 的区别？→ `input()` 返回 Signal，可以直接用于 `computed`，不需要 `ngOnChanges`；支持 `required` 和 `transform`。

---

### Q12. `model()` 是什么？

- **难度**: 初级
- **标签**: 组件通信

**答案**

`model()` 创建一个**可写的输入信号**，支持双向绑定：

```ts
@Component({
  selector: 'app-toggle',
  template: `<button (click)="checked.set(!checked())">{{ checked() ? 'ON' : 'OFF' }}</button>`,
})
export class Toggle {
  checked = model(false);
}
```

```html
<app-toggle [(checked)]="darkMode" />   <!-- darkMode 可以是 signal 或普通属性 -->
```

`model()` 自动生成 `checkedChange` 输出。

**口述模板（30–60 秒）**

1. 一句话结论：model 是可写的输入信号，用于实现双向绑定。
2. 关键机制：子组件调用 set 修改值，自动触发 xxxChange 事件通知父组件。
3. 对比或注意点：input 是只读的，model 可以在子组件中修改。
4. 可追问时补充：父组件可以直接绑定一个 signal。

**常见追问**

- 什么时候用 `model`？→ 自定义的表单类组件（开关、评分、日期选择器）需要双向绑定时。

---

### Q13. 兄弟组件或跨层级组件怎么共享状态？

- **难度**: 初级
- **标签**: 状态共享

**答案**

用**服务 + Signals**：

```ts
@Injectable({ providedIn: 'root' })
export class CartStore {
  private readonly _items = signal<CartItem[]>([]);
  readonly items = this._items.asReadonly();
  readonly total = computed(() => this._items().reduce((s, i) => s + i.price * i.qty, 0));

  add(item: CartItem) {
    this._items.update((list) => [...list, item]);
  }
}
```

组件中：`cart = inject(CartStore);` 模板中 `{{ cart.total() }}`。

**要点**：对外暴露只读信号，修改通过方法进行，保证数据流可追踪。

**口述模板（30–60 秒）**

1. 一句话结论：用注入的服务加 Signals 共享状态。
2. 关键机制：服务内部持有可写的 signal，对外暴露只读信号和 computed。
3. 对比或注意点：修改只能通过服务的方法，保证数据流可控。
4. 可追问时补充：更复杂的场景可以用 NgRx 或 SignalStore。

**常见追问**

- 服务的作用域怎么控制？→ `providedIn: 'root'` 全局单例；在组件或路由的 `providers` 中提供则是局部实例。

---

## 三、依赖注入与服务

### Q14. 什么是依赖注入？Angular 中怎么使用？

- **难度**: 初级
- **标签**: 依赖注入

**答案**

依赖注入（DI）：类不自己创建依赖，而是由框架（注入器）提供。好处是解耦、便于测试（替换为模拟实现）、控制实例的作用域。

```ts
@Injectable({ providedIn: 'root' })
export class UserApi {
  private http = inject(HttpClient);
  getUser(id: string) {
    return this.http.get<User>(`/api/users/${id}`);
  }
}

@Component({ /* ... */ })
export class Profile {
  private api = inject(UserApi);   // 推荐：inject() 函数
  // 或构造函数注入：constructor(private api: UserApi) {}
}
```

v22 新增 `@Service()` 装饰器，作为 `@Injectable({ providedIn: 'root' })` 的简写（以官方文档为准）。

**口述模板（30–60 秒）**

1. 一句话结论：依赖注入让类从注入器获取依赖，而不是自己创建。
2. 关键机制：用 @Injectable 声明可注入的服务，在组件中用 inject() 获取。
3. 对比或注意点：好处是解耦、便于测试、可以控制实例作用域。
4. 可追问时补充：v22 新增 @Service 作为根级单例服务的简写。

**常见追问**

- `inject()` 可以在哪里调用？→ 只能在**注入上下文**中：构造函数、字段初始化、工厂函数、`runInInjectionContext` 中。

---

### Q15. providers 有哪几种配置方式？

- **难度**: 初级
- **标签**: 依赖注入

**答案**

```ts
providers: [
  UserService,                                          // 简写，等于 useClass
  { provide: Logger, useClass: ConsoleLogger },         // 替换实现
  { provide: API_URL, useValue: 'https://api.example.com' },
  { provide: Cache, useFactory: () => new Cache(inject(API_URL)) },
  { provide: OldLogger, useExisting: Logger },          // 别名，同一个实例
]
```

**`InjectionToken`**：为非类的值（字符串、配置对象、函数）创建令牌。

```ts
export const API_URL = new InjectionToken<string>('API_URL');
const url = inject(API_URL);
```

**口述模板（30–60 秒）**

1. 一句话结论：providers 有 useClass、useValue、useFactory、useExisting 四种方式。
2. 关键机制：provide 指定令牌，后面指定如何创建值。
3. 对比或注意点：非类的值用 InjectionToken 作为令牌。
4. 可追问时补充：useExisting 是别名，和原令牌共享同一个实例。

**常见追问**

- `multi: true` 是什么？→ 同一个令牌注册多个值，注入时得到数组（例如多个拦截器、多个初始化函数）。

---

### Q16. 服务在哪里提供有什么区别？

- **难度**: 初级
- **标签**: 依赖注入, 作用域

**答案**

| 位置 | 作用域 | 说明 |
|------|--------|------|
| `@Injectable({ providedIn: 'root' })` | 应用级单例 | **可摇树**：没被使用时不会打包 |
| `bootstrapApplication` 的 providers | 应用级 | 全局配置（`provideRouter`、`provideHttpClient`） |
| 路由的 `providers` | 该路由及其子路由 | 懒加载模块级别的服务 |
| 组件的 `providers` | 该组件及其子组件，**每个组件实例一个** | 组件局部状态（如表单状态、列表页的筛选） |
| 组件的 `viewProviders` | 只对视图可见，投影内容不可见 | 库组件 |

**口述模板（30–60 秒）**

1. 一句话结论：providedIn root 是全局单例且可摇树，组件级 providers 每个组件实例一个。
2. 关键机制：注入器按层级组织，子级可以覆盖父级的提供者。
3. 对比或注意点：路由级 providers 在该路由范围内共享。
4. 可追问时补充：组件级服务会随组件销毁，适合页面局部状态。

**常见追问**

- 懒加载路由中的服务会创建新实例吗？→ 如果在路由的 `providers` 中提供，会为该路由创建独立的环境注入器和实例；`providedIn: 'root'` 的服务仍然是全局单例。

---

## 四、路由

### Q17. 怎么配置路由？怎么实现懒加载？

- **难度**: 初级
- **标签**: 路由

**答案**

```ts
export const routes: Routes = [
  { path: '', component: Home },
  { path: 'products/:id', loadComponent: () => import('./product/product').then((m) => m.Product) },
  { path: 'admin', loadChildren: () => import('./admin/admin.routes').then((m) => m.ADMIN_ROUTES),
    canMatch: [adminGuard] },
  { path: '**', component: NotFound },
];

bootstrapApplication(App, {
  providers: [provideRouter(routes, withComponentInputBinding())],
});
```

- `loadComponent`：懒加载单个组件
- `loadChildren`：懒加载一组子路由
- 模板中：`<router-outlet />`、`<a routerLink="/products/1" routerLinkActive="active">`
- 编程式导航：`inject(Router).navigate(['/products', id])`

**口述模板（30–60 秒）**

1. 一句话结论：用 Routes 数组配置路由，用 loadComponent 和 loadChildren 实现懒加载。
2. 关键机制：动态 import 让构建工具拆分代码块。
3. 对比或注意点：通配路由 ** 放在最后，因为路由按顺序匹配。
4. 可追问时补充：withComponentInputBinding 可以把路由参数直接绑定到组件的 input。

**常见追问**

- 路由匹配的顺序？→ 按数组顺序，第一个匹配的生效。

---

### Q18. 怎么获取路由参数？

- **难度**: 初级
- **标签**: 路由

**答案**

**方式一：绑定到组件输入（推荐）**

```ts
// provideRouter(routes, withComponentInputBinding())
export class Product {
  id = input.required<string>();     // 来自 :id
  tab = input<string>();             // 来自 ?tab=
  product = httpResource<Product>(() => `/api/products/${this.id()}`);
}
```

**方式二：ActivatedRoute**

```ts
private route = inject(ActivatedRoute);
id = toSignal(this.route.paramMap.pipe(map((p) => p.get('id'))));
```

**注意**：从 `/products/1` 导航到 `/products/2` 时**组件会被复用**，所以要用可观察 / 信号的方式响应参数变化，而不是只在 `ngOnInit` 中读取快照（`snapshot`）。

**口述模板（30–60 秒）**

1. 一句话结论：推荐开启组件输入绑定，把路由参数直接作为 input。
2. 关键机制：也可以注入 ActivatedRoute 订阅 paramMap。
3. 对比或注意点：同一路由参数变化时组件会被复用，不能只读一次快照。
4. 可追问时补充：查询参数和路由数据也可以通过输入绑定获取。

**常见追问**

- 怎么在路由中传递静态数据？→ 路由配置中的 `data` 字段，或通过解析器（resolver）预先获取数据。

---

### Q19. 路由守卫有哪些？怎么写？

- **难度**: 初级
- **标签**: 路由, 守卫

**答案**

| 守卫 | 作用 |
|------|------|
| `canActivate` | 能否进入路由 |
| `canActivateChild` | 能否进入子路由 |
| `canDeactivate` | 能否离开（未保存提示） |
| `canMatch` | 路由是否匹配（不匹配时继续尝试其他路由；懒加载的代码不会被下载） |
| `resolve` | 进入前预取数据 |

推荐**函数式守卫**：

```ts
export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.isLoggedIn()
    ? true
    : router.createUrlTree(['/login'], { queryParams: { redirect: state.url } });
};
```

**注意**：前端守卫只是体验层面的控制，**权限必须在服务端校验**。

**口述模板（30–60 秒）**

1. 一句话结论：常用守卫有 canActivate、canDeactivate、canMatch 和 resolve。
2. 关键机制：推荐函数式守卫，返回布尔值或 UrlTree 实现重定向。
3. 对比或注意点：canMatch 不匹配时懒加载的代码不会被下载。
4. 可追问时补充：前端守卫只是体验，权限必须由服务端校验。

**常见追问**

- 守卫中返回 UrlTree 和调用 `router.navigate` 有什么区别？→ 返回 UrlTree 会取消当前导航并开始新导航，行为更可预测；在守卫中调用 `navigate` 可能导致竞态。

---

## 五、表单

### Q20. 模板驱动表单和响应式表单有什么区别？

- **难度**: 初级
- **标签**: 表单

**答案**

| 方面 | 模板驱动 | 响应式 | Signal Forms（v22 稳定） |
|------|----------|--------|---------------------------|
| 定义位置 | 模板（`ngModel`） | 类中（`FormGroup`） | 类中，基于 signal 模型 |
| 数据模型 | 隐式、可变 | 显式、类型化 | **signal 即模型** |
| 校验 | 指令 | 校验函数 | 声明式的 schema 规则 |
| 适合 | 简单表单 | 复杂、动态表单 | 新项目的推荐方向 |

响应式表单示例：

```ts
export class Signup {
  private fb = inject(NonNullableFormBuilder);
  form = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]],
  });

  submit() {
    if (this.form.invalid) return this.form.markAllAsTouched();
    const value = this.form.getRawValue();   // 类型为 { email: string; password: string }
  }
}
```

```html
<form [formGroup]="form" (ngSubmit)="submit()">
  <input formControlName="email" />
  @if (form.controls.email.touched && form.controls.email.hasError('email')) {
    <small>邮箱格式不正确</small>
  }
</form>
```

**口述模板（30–60 秒）**

1. 一句话结论：模板驱动表单在模板中定义，响应式表单在类中定义，Signal Forms 以 signal 作为模型。
2. 关键机制：响应式表单显式、类型化，适合复杂和动态的表单。
3. 对比或注意点：模板驱动适合简单表单。
4. 可追问时补充：v22 起 Signal Forms 稳定，可以与响应式表单互操作渐进迁移。

**常见追问**

- `value` 和 `getRawValue()` 的区别？→ `value` 不包含禁用的控件，`getRawValue()` 包含所有控件。

---

### Q21. 怎么写自定义校验器？异步校验器呢？

- **难度**: 初级
- **标签**: 表单, 校验

**答案**

```ts
// 同步校验器：返回 null 表示通过
export function forbiddenName(name: string): ValidatorFn {
  return (control) => (control.value === name ? { forbiddenName: { value: control.value } } : null);
}

// 跨字段校验：放在 FormGroup 上
export const passwordMatch: ValidatorFn = (group) => {
  const { password, confirm } = group.value;
  return password === confirm ? null : { mismatch: true };
};

// 异步校验器
export function uniqueEmail(api: UserApi): AsyncValidatorFn {
  return (control) =>
    timer(300).pipe(   // 防抖：值变化时上一次 timer 被取消
      switchMap(() => api.checkEmail(control.value)),
      map((taken) => (taken ? { emailTaken: true } : null)),
      catchError(() => of(null)),
    );
}
```

使用：`email: ['', { validators: [Validators.email], asyncValidators: [uniqueEmail(api)], updateOn: 'blur' }]`

**口述模板（30–60 秒）**

1. 一句话结论：校验器是函数，返回 null 表示通过，返回错误对象表示失败。
2. 关键机制：跨字段校验放在 FormGroup 上，异步校验器返回 Observable。
3. 对比或注意点：异步校验要防抖，可以设置 updateOn: 'blur' 减少请求。
4. 可追问时补充：异步校验在同步校验通过后才执行，期间状态为 PENDING。

**常见追问**

- 前端校验够吗？→ 不够，前端校验只是体验，服务端必须再次校验。

---

## 六、HTTP 与工程

### Q22. HttpClient 怎么用？拦截器怎么写？

- **难度**: 初级
- **标签**: HTTP

**答案**

```ts
bootstrapApplication(App, {
  providers: [provideHttpClient(withFetch(), withInterceptors([authInterceptor]))],
});

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();
  const authReq = token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req;
  return next(authReq);
};
```

- 请求对象是**不可变的**，修改需要 `clone`
- `HttpClient` 返回**冷的 Observable**：不订阅不会发请求，每次订阅都会发一次
- 推荐使用函数式拦截器
- 新的 `httpResource`（v22 稳定）：以 Signal 的方式获取数据

```ts
user = httpResource<User>(() => `/api/users/${this.id()}`);
// 模板：@if (user.hasValue()) { {{ user.value().name }} }
```

**口述模板（30–60 秒）**

1. 一句话结论：HttpClient 返回 Observable，拦截器用于统一处理请求和响应。
2. 关键机制：请求对象不可变，修改要 clone；函数式拦截器通过 next 传递。
3. 对比或注意点：Observable 是冷的，不订阅不发请求，多次订阅发多次请求。
4. 可追问时补充：httpResource 可以把请求结果直接作为 Signal 使用，参数变化自动重新请求。

**常见追问**

- 拦截器的执行顺序？→ 请求按注册顺序经过拦截器，响应按相反的顺序返回。

---

### Q23. `async` 管道有什么作用？

- **难度**: 初级
- **标签**: RxJS, 模板

**答案**

```html
@if (user$ | async; as user) {
  <p>{{ user.name }}</p>
}
```

- 自动订阅 Observable / Promise，并返回最新值
- 组件销毁时**自动取消订阅**，避免内存泄漏
- 新值到来时调用 `markForCheck`，兼容 `OnPush`

**Signals 时代**：可以用 `toSignal(user$)` 转为 Signal，模板中直接 `user()`。

**口述模板（30–60 秒）**

1. 一句话结论：async 管道在模板中自动订阅 Observable 并在销毁时取消订阅。
2. 关键机制：新值到来时调用 markForCheck，兼容 OnPush。
3. 对比或注意点：同一个 Observable 多次使用 async 管道会多次订阅。
4. 可追问时补充：现在也可以用 toSignal 转为 Signal 使用。

**常见追问**

- 多次使用 `async` 导致多次请求怎么办？→ 用 `@if (x$ | async; as x)` 只订阅一次，或 `shareReplay`，或转为 Signal。

---

### Q24. RxJS 在 Angular 中的基本用法？怎么取消订阅？

- **难度**: 初级
- **标签**: RxJS

**答案**

Angular 中 RxJS 用于：HttpClient、路由事件、响应式表单的 `valueChanges`、事件流。

**取消订阅的方式**：

```ts
export class Search {
  private destroyRef = inject(DestroyRef);
  query = new FormControl('', { nonNullable: true });

  constructor() {
    this.query.valueChanges
      .pipe(debounceTime(300), distinctUntilChanged(), takeUntilDestroyed())
      .subscribe((q) => this.search(q));
  }
}
```

| 方式 | 说明 |
|------|------|
| `async` 管道 | 模板中自动管理 |
| `takeUntilDestroyed()` | 组件销毁时自动完成（注入上下文中调用，或传入 DestroyRef） |
| `toSignal()` | 自动在注入上下文销毁时取消订阅 |
| 手动 `unsubscribe` | `ngOnDestroy` 中调用 |

**不需要取消订阅的**：HttpClient 的请求（完成后自动结束）、`ActivatedRoute` 的 Observable（路由管理）。但请求未完成时组件已销毁，取消订阅可以中止请求。

**口述模板（30–60 秒）**

1. 一句话结论：长期存在的订阅必须在组件销毁时取消，否则会内存泄漏。
2. 关键机制：推荐 async 管道、takeUntilDestroyed 或 toSignal 自动管理。
3. 对比或注意点：HttpClient 请求完成后自动结束，但取消订阅可以中止未完成的请求。
4. 可追问时补充：takeUntilDestroyed 要放在 pipe 的最后。

**常见追问**

- 为什么 `takeUntilDestroyed` 要放在最后？→ 放在高阶操作符（如 `switchMap`）之前时，内部的订阅不会被取消。

---

### Q25. Angular CLI 常用命令有哪些？

- **难度**: 初级
- **标签**: 工程

**答案**

| 命令 | 作用 |
|------|------|
| `ng new my-app` | 创建项目 |
| `ng serve` | 开发服务器（基于 Vite） |
| `ng build` | 生产构建（esbuild） |
| `ng generate component user-list` | 生成组件（`g c`） |
| `ng generate service` / `guard` / `pipe` / `directive` | 生成其他类型 |
| `ng test` | 单元测试（v21 起默认 Vitest） |
| `ng update @angular/core @angular/cli` | 升级并自动执行迁移脚本 |
| `ng add @angular/material` | 添加库并执行配置脚本 |
| `ng generate @angular/core:signal-input-migration` | 官方迁移脚本示例 |

**口述模板（30–60 秒）**

1. 一句话结论：CLI 负责创建、开发、构建、测试、生成代码和升级。
2. 关键机制：构建基于 esbuild，开发服务器基于 Vite。
3. 对比或注意点：ng update 会自动执行官方的迁移脚本。
4. 可追问时补充：v21 起默认测试运行器是 Vitest，v22 弃用了 Webpack 相关构建器。

**常见追问**

- `ng add` 和 `npm install` 的区别？→ `ng add` 安装后会执行库的 schematics，自动修改配置和代码。

---

### Q26. Angular 中怎么处理样式？

- **难度**: 初级
- **标签**: 样式

**答案**

- **组件样式**默认隔离（Emulated）
- **全局样式**：`styles.scss`（在 `angular.json` 中配置）
- **`:host`**：选中组件宿主元素；`:host(.active)` 按宿主的类选择
- **`::ng-deep`**：穿透到子组件（**已弃用**，但仍然可用；推荐使用 CSS 变量或全局样式）
- 绑定：`[class.active]`、`[class]="{ a: x, b: y }"`、`[style.width.px]`
- 支持 Sass、Less、Tailwind

**口述模板（30–60 秒）**

1. 一句话结论：组件样式默认隔离，全局样式在 styles 文件中配置。
2. 关键机制：:host 选中宿主元素，类和样式可以绑定。
3. 对比或注意点：::ng-deep 已弃用，推荐用 CSS 变量定制子组件。
4. 可追问时补充：组件库常通过 CSS 变量暴露主题定制。

**常见追问**

- 为什么推荐 CSS 变量而不是 `::ng-deep`？→ CSS 变量可以穿透封装且是子组件显式暴露的定制点，不会依赖内部结构。

---

### Q27. 怎么在 Angular 中做国际化？

- **难度**: 初级
- **标签**: 国际化

**答案**

| 方案 | 特点 |
|------|------|
| **官方 `@angular/localize`** | 编译时翻译，每种语言构建一个包；性能好，但切换语言需要刷新 / 跳转到对应的包 |
| **ngx-translate / Transloco** | 运行时翻译，加载 JSON；可以动态切换语言 |

```html
<!-- 官方 -->
<h1 i18n="@@welcome">欢迎</h1>
<!-- Transloco -->
<h1>{{ 'welcome' | transloco }}</h1>
```

日期、数字、货币管道会根据 `LOCALE_ID` 格式化。

**口述模板（30–60 秒）**

1. 一句话结论：官方方案是编译时翻译，社区方案（Transloco）是运行时翻译。
2. 关键机制：官方方案每种语言构建一个包，性能好；运行时方案可以动态切换。
3. 对比或注意点：需要不刷新切换语言时选运行时方案。
4. 可追问时补充：日期、货币管道根据 LOCALE_ID 格式化。

**常见追问**

- 选型建议？→ 内容站和需要 SEO 的多语言站点用官方方案；中后台需要动态切换时用 Transloco。

---

### Q28. 怎么写一个简单的组件单元测试？

- **难度**: 初级
- **标签**: 测试

**答案**

```ts
import { TestBed } from '@angular/core/testing';

describe('Counter', () => {
  it('点击后计数加一', async () => {
    const fixture = TestBed.createComponent(Counter);
    await fixture.whenStable();

    const button: HTMLButtonElement = fixture.nativeElement.querySelector('button');
    button.click();
    await fixture.whenStable();   // Zoneless 下等待调度的变更检测完成

    expect(fixture.nativeElement.querySelector('p').textContent).toContain('1');
  });
});
```

- `TestBed` 配置测试模块，替换依赖：`providers: [{ provide: UserApi, useValue: mockApi }]`
- HTTP 测试：`provideHttpClientTesting()` + `HttpTestingController`
- v21 起默认使用 Vitest；E2E 使用 Playwright / Cypress

**口述模板（30–60 秒）**

1. 一句话结论：用 TestBed 创建组件，通过 fixture 操作和断言 DOM。
2. 关键机制：依赖通过 providers 替换为模拟实现。
3. 对比或注意点：Zoneless 下用 whenStable 等待变更检测完成。
4. 可追问时补充：HTTP 测试用 HttpTestingController 模拟响应。

**常见追问**

- 测试应该测什么？→ 测试用户可见的行为（DOM 输出、交互结果），而不是内部实现细节。

---

### Q29. Angular 的安全机制是怎样的？

- **难度**: 初级
- **标签**: 安全

**答案**

- **默认清理**：插值和属性绑定中的值会按上下文（HTML、样式、URL、资源 URL）自动清理或转义
  - `[innerHTML]="html"` 会移除危险的内容（脚本、事件属性）
  - `[href]="url"` 中的 `javascript:` 会被处理为不安全
- **`DomSanitizer.bypassSecurityTrust*`**：明确告诉 Angular 信任某个值 → **绕过保护，只对可信内容使用**
- **AOT 编译**：模板不在运行时编译，防止模板注入
- **不要**直接用 `ElementRef.nativeElement.innerHTML` 赋值
- 支持 CSP（`autoCsp`、`ngCspNonce`）和 Trusted Types
- HttpClient 内置 XSRF 支持（读取 `XSRF-TOKEN` Cookie 并添加请求头）

**口述模板（30–60 秒）**

1. 一句话结论：Angular 默认对绑定的值按上下文进行清理和转义。
2. 关键机制：innerHTML 绑定会移除危险内容，URL 绑定会处理危险协议。
3. 对比或注意点：bypassSecurityTrust 会绕过保护，只能用于可信内容。
4. 可追问时补充：直接操作 nativeElement 不受保护；Angular 支持 CSP 和 Trusted Types。

**常见追问**

- 富文本需要展示用户提交的 HTML 怎么办？→ 先用 DOMPurify 等库按白名单清洗，再绑定到 `[innerHTML]`，不要用 `bypassSecurityTrustHtml` 处理未清洗的内容。

---

### Q30. Standalone 组件和 NgModule 有什么区别？

- **难度**: 初级
- **标签**: Standalone, NgModule

**答案**

**NgModule（旧）**：组件在模块的 `declarations` 中声明，通过模块的 `imports` / `exports` 组织依赖。

**Standalone（v15 稳定，v19 起默认）**：组件自己在 `imports` 中声明依赖，不需要 NgModule。

```ts
@Component({
  selector: 'app-user-list',
  imports: [RouterLink, DatePipe, UserCard],
  template: `...`,
})
export class UserList {}

bootstrapApplication(App, appConfig);   // 代替 AppModule
```

**优点**：依赖关系清晰（看组件本身就知道依赖什么）、更容易懒加载和摇树、减少样板代码、学习成本更低。

**迁移**：`ng generate @angular/core:standalone`。

**口述模板（30–60 秒）**

1. 一句话结论：Standalone 组件自己声明依赖，不需要 NgModule，v19 起是默认方式。
2. 关键机制：依赖在组件的 imports 中声明，应用通过 bootstrapApplication 启动。
3. 对比或注意点：依赖关系更清晰，更容易懒加载和摇树。
4. 可追问时补充：官方提供了迁移脚本，NgModule 仍然可以与 Standalone 混用。

**常见追问**

- NgModule 会被移除吗？→ 官方没有移除计划，但新代码推荐 Standalone。
