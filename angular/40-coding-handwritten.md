# Angular 手写题

> 目标：通过手写核心机制的简化实现理解 Signals，练习面试中常见的 Angular 业务编码（自定义控件、拦截器、守卫、指令、Store、复合组件）。面试时先说思路，再写核心代码，最后说明边界情况。示例基于 Angular v20+。

## 使用方式

- **原理实现**：H1 迷你 Signals（signal / computed / effect）
- **表单与路由**：H2 标签输入控件（CVA + 校验）、H4 守卫、H9 配置驱动的动态表单
- **HTTP 与异步**：H3 刷新令牌的拦截器、H5 防抖搜索、H11 可见时轮询
- **指令与组件**：H6 权限结构指令、H8 无限滚动指令、H12 Tabs 复合组件
- **状态与错误**：H7 SignalStore、H10 全局错误处理

---

### H1. 实现迷你 Signals（signal / computed / effect）

- **难度**: 高级
- **标签**: Signals, 原理
- **面试官看什么**: 依赖追踪、推拉结合、computed 的惰性与缓存、动态依赖清理

**题干**

实现 `signal`、`computed`、`effect`：

- `computed` 惰性计算并缓存，依赖不变时不重新计算
- 依赖是动态的（分支切换后不再追踪旧依赖）
- `effect` 异步批量执行，同一轮多次修改只执行一次

**思路口述**

全局保存当前活跃的消费者。读取生产者时双向登记依赖，并记录读取时生产者的版本号。写入时生产者版本号加一，并向下游**推送脏标记**；computed 被读取时再**拉取**：逐个检查依赖的版本是否变化，变了才重新计算。effect 收到脏标记后放入微任务批量执行。

**参考实现**

```ts
interface Producer {
  version: number;
  consumers: Set<Consumer>;
  /** computed 在被检查版本前需要先把自己更新到最新 */
  refresh?(): void;
}

interface Consumer {
  /** 依赖的生产者 → 读取时的版本号 */
  producers: Map<Producer, number>;
  markDirty(): void;
}

let activeConsumer: Consumer | null = null;

function track(producer: Producer): void {
  if (!activeConsumer) return;
  producer.consumers.add(activeConsumer);
  activeConsumer.producers.set(producer, producer.version);
}

function notify(producer: Producer): void {
  // 先复制：markDirty 过程中可能修改集合
  for (const consumer of [...producer.consumers]) consumer.markDirty();
}

/** 依赖中是否有生产者的版本发生了变化 */
function producersChanged(consumer: Consumer): boolean {
  for (const [producer, seenVersion] of consumer.producers) {
    producer.refresh?.();
    if (producer.version !== seenVersion) return true;
  }
  return false;
}

/** 以 consumer 身份执行 fn，并重新收集依赖（解决分支切换留下的旧依赖） */
function runAsConsumer<T>(consumer: Consumer, fn: () => T): T {
  for (const producer of consumer.producers.keys()) producer.consumers.delete(consumer);
  consumer.producers.clear();
  const prev = activeConsumer;
  activeConsumer = consumer;
  try {
    return fn();
  } finally {
    activeConsumer = prev;
  }
}

export interface WritableSignal<T> {
  (): T;
  set(value: T): void;
  update(fn: (value: T) => T): void;
}

/**
 * 创建可写信号，值变化（Object.is 判断）时通知依赖方。
 */
export function signal<T>(initial: T): WritableSignal<T> {
  let value = initial;
  const node: Producer = { version: 0, consumers: new Set() };

  const read = (() => {
    track(node);
    return value;
  }) as WritableSignal<T>;

  read.set = (next) => {
    if (Object.is(value, next)) return;
    value = next;
    node.version++;
    notify(node);
  };
  read.update = (fn) => read.set(fn(value));
  return read;
}

/**
 * 创建惰性、带缓存的派生信号。
 */
export function computed<T>(fn: () => T): () => T {
  let value: T;
  let dirty = true;
  let initialized = false;

  const node: Producer & Consumer = {
    version: 0,
    consumers: new Set(),
    producers: new Map(),
    markDirty() {
      if (dirty) return; // 已经是脏的，下游也已经被通知过
      dirty = true;
      notify(node);
    },
    refresh() {
      if (!dirty) return;
      dirty = false;
      // 依赖的版本都没变（例如上游 computed 重新计算后值相同），不需要重新计算
      if (initialized && !producersChanged(node)) return;
      const next = runAsConsumer(node, fn);
      // 值没变时版本不变，下游据此跳过重新计算（避免无效的级联更新）
      if (!initialized || !Object.is(value, next)) {
        value = next;
        node.version++;
      }
      initialized = true;
    },
  };

  return () => {
    node.refresh!();
    track(node);
    return value;
  };
}

/**
 * 创建副作用，依赖变化后在微任务中批量重新执行。返回销毁函数。
 */
export function effect(fn: () => void): () => void {
  let scheduled = false;
  let destroyed = false;

  const node: Consumer = {
    producers: new Map(),
    markDirty() {
      if (scheduled || destroyed) return;
      scheduled = true;
      // 同一轮中的多次修改只执行一次
      queueMicrotask(run);
    },
  };

  function run() {
    scheduled = false;
    if (destroyed) return;
    runAsConsumer(node, fn);
  }

  run(); // 首次同步执行以收集依赖
  return () => {
    destroyed = true;
    for (const producer of node.producers.keys()) producer.consumers.delete(node);
    node.producers.clear();
  };
}
```

**易错点**

- 不清理旧依赖：`ok() ? a() : b()` 切换后，`a` 变化仍会触发重新计算
- computed 在写入时立即重新计算（失去惰性，菱形依赖下会重复计算）
- computed 值没变也增加版本号 → 下游无谓地重新计算
- effect 同步执行 → 连续 `set` 多次导致多次执行
- 读取时没有处理嵌套的消费者（需要保存和恢复上一个 activeConsumer）

---

### H2. 实现标签输入控件（ControlValueAccessor + 校验）

- **难度**: 中级
- **标签**: 表单, CVA
- **面试官看什么**: CVA 四个方法的职责、touched / disabled、组件自带校验

**题干**

实现 `<app-tag-input>`：回车添加标签、点击删除，值为 `string[]`；可以用于 `formControlName`；组件自带「最多 N 个标签」的校验。

**参考实现**

```ts
import {
  Component, forwardRef, input, numberAttribute, signal,
} from '@angular/core';
import {
  AbstractControl, ControlValueAccessor, NG_VALIDATORS, NG_VALUE_ACCESSOR,
  ValidationErrors, Validator,
} from '@angular/forms';

/**
 * 标签输入控件：值为字符串数组，可接入响应式表单，并自带最大数量校验。
 */
@Component({
  selector: 'app-tag-input',
  template: `
    <ul class="tags">
      @for (tag of tags(); track tag) {
        <li>
          {{ tag }}
          <button type="button" [disabled]="disabled()" (click)="remove(tag)" [attr.aria-label]="'删除 ' + tag">×</button>
        </li>
      }
    </ul>
    <input
      [disabled]="disabled()"
      (keydown.enter)="add($event)"
      (blur)="onTouched()"
      placeholder="输入后回车"
    />
  `,
  providers: [
    { provide: NG_VALUE_ACCESSOR, useExisting: forwardRef(() => TagInput), multi: true },
    { provide: NG_VALIDATORS, useExisting: forwardRef(() => TagInput), multi: true },
  ],
})
export class TagInput implements ControlValueAccessor, Validator {
  max = input(10, { transform: numberAttribute });

  tags = signal<string[]>([]);
  disabled = signal(false);

  private onChange: (value: string[]) => void = () => {};
  protected onTouched: () => void = () => {};

  writeValue(value: string[] | null): void {
    // 表单 reset 时会传入 null；这里不能调用 onChange，否则会把控件标记为 dirty
    this.tags.set(value ?? []);
  }
  registerOnChange(fn: (value: string[]) => void): void { this.onChange = fn; }
  registerOnTouched(fn: () => void): void { this.onTouched = fn; }
  setDisabledState(isDisabled: boolean): void { this.disabled.set(isDisabled); }

  validate(control: AbstractControl<string[] | null>): ValidationErrors | null {
    const count = control.value?.length ?? 0;
    return count > this.max() ? { maxTags: { max: this.max(), actual: count } } : null;
  }

  add(event: Event): void {
    event.preventDefault(); // 回车不要提交外层表单
    const input = event.target as HTMLInputElement;
    const tag = input.value.trim();
    if (!tag || this.tags().includes(tag)) return;
    this.commit([...this.tags(), tag]);
    input.value = '';
  }

  remove(tag: string): void {
    this.commit(this.tags().filter((t) => t !== tag));
    this.onTouched();
  }

  private commit(next: string[]): void {
    this.tags.set(next);   // 不可变更新，外部拿到的是新数组
    this.onChange(next);
  }
}
```

使用：`<app-tag-input formControlName="skills" max="5" />`，错误：`form.controls.skills.hasError('maxTags')`。

**易错点**

- 在 `writeValue` 中调用 `onChange`
- 直接 `push` 修改数组（外部的值和内部共享引用）
- 忘记 `setDisabledState`、`onTouched`
- 回车触发外层表单提交
- 用 `NG_VALIDATORS` 提供校验时忘记 `multi: true`

---

### H3. 带令牌刷新的 HTTP 拦截器

- **难度**: 高级
- **标签**: HTTP, 拦截器, 并发
- **面试官看什么**: 并发 401 时只刷新一次、刷新请求本身不被拦截、失败后的处理

**题干**

实现函数式拦截器：附加访问令牌；遇到 401 时刷新令牌并重试原请求；多个请求同时 401 时只发一次刷新请求；刷新失败则跳转登录。

**参考实现**

```ts
import { HttpClient, HttpContext, HttpContextToken, HttpErrorResponse, HttpInterceptorFn, HttpRequest } from '@angular/common/http';
import { Injectable, inject, signal } from '@angular/core';
import { Router } from '@angular/router';
import { Observable, catchError, finalize, map, shareReplay, switchMap, tap, throwError } from 'rxjs';

/** 标记不需要附加令牌和处理 401 的请求（例如刷新请求本身） */
export const SKIP_AUTH = new HttpContextToken<boolean>(() => false);

/**
 * 管理访问令牌，并保证同一时间只有一个刷新请求在进行。
 */
@Injectable({ providedIn: 'root' })
export class AuthService {
  private http = inject(HttpClient);
  readonly accessToken = signal<string | null>(null);
  private refreshing$: Observable<string> | null = null;

  refresh(): Observable<string> {
    // 并发的 401 共享同一个刷新请求，避免刷新令牌被轮换后其他请求用旧令牌刷新失败
    this.refreshing$ ??= this.http
      .post<{ accessToken: string }>('/api/auth/refresh', null, {
        withCredentials: true, // 刷新令牌在 HttpOnly Cookie 中
        context: new HttpContext().set(SKIP_AUTH, true),
      })
      .pipe(
        map((res) => res.accessToken),
        tap((token) => this.accessToken.set(token)),
        // 完成或失败后清空，下一次 401 才会发起新的刷新
        finalize(() => (this.refreshing$ = null)),
        shareReplay({ bufferSize: 1, refCount: false }),
      );
    return this.refreshing$;
  }
}

function withToken(req: HttpRequest<unknown>, token: string | null): HttpRequest<unknown> {
  return token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req;
}

/**
 * 附加访问令牌；401 时刷新令牌并重试一次，刷新失败跳转登录页。
 */
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  if (req.context.get(SKIP_AUTH)) return next(req);

  const auth = inject(AuthService);
  const router = inject(Router);

  return next(withToken(req, auth.accessToken())).pipe(
    catchError((error: unknown) => {
      if (!(error instanceof HttpErrorResponse) || error.status !== 401) {
        return throwError(() => error);
      }
      return auth.refresh().pipe(
        // 只重试一次：重试后的 401 会进入下面的 catchError，而不会再次刷新
        switchMap((token) => next(withToken(req, token))),
        catchError((retryError: unknown) => {
          auth.accessToken.set(null);
          router.navigate(['/login'], { queryParams: { redirect: router.url } });
          return throwError(() => retryError);
        }),
      );
    }),
  );
};

// provideHttpClient(withInterceptors([authInterceptor]))
```

**易错点**

- 刷新请求本身也被拦截 → 401 时无限循环
- 每个 401 各自刷新 → 并发刷新，刷新令牌轮换时互相使旧令牌失效
- 重试后仍然 401 时再次刷新 → 死循环
- 刷新失败后没有清空状态
- 把令牌附加到第三方域名的请求（应该只对自己的 API 附加，可以按 URL 前缀判断）

---

### H4. 未保存提示守卫和基于角色的路由匹配

- **难度**: 中级
- **标签**: 路由, 守卫
- **面试官看什么**: 函数式守卫、canDeactivate 的类型约束、canMatch 的用途

**题干**

1. 表单页有未保存的修改时，离开前弹窗确认；刷新和关闭标签页也要提示
2. 根据路由 `data.roles` 限制访问，无权限时跳转 403，并且懒加载的代码不应该被下载

**参考实现**

```ts
import { inject } from '@angular/core';
import { CanDeactivateFn, CanMatchFn, Router } from '@angular/router';

/** 需要离开确认的组件实现此接口 */
export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

/**
 * 有未保存修改时，离开路由前请求用户确认。
 */
export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) => {
  if (!component.hasUnsavedChanges()) return true;
  // ConfirmDialog.confirm 返回 Observable<boolean> 或 Promise<boolean>，守卫都支持
  return inject(ConfirmDialog).confirm('有未保存的修改，确定离开吗？');
};

/**
 * 按路由 data.roles 判断是否匹配；用 canMatch 而不是 canActivate，
 * 这样无权限时懒加载的代码不会被下载。
 */
export const roleGuard: CanMatchFn = (route) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (!auth.isLoggedIn()) return router.createUrlTree(['/login']);

  const required = (route.data?.['roles'] as string[] | undefined) ?? [];
  const userRoles = auth.roles();
  return required.length === 0 || required.some((r) => userRoles.includes(r))
    ? true
    : router.createUrlTree(['/403']);
};
```

组件中处理刷新和关闭：

```ts
@Component({
  selector: 'app-profile-edit',
  host: { '(window:beforeunload)': 'onBeforeUnload($event)' },
  // ...
})
export class ProfileEdit implements HasUnsavedChanges {
  form = inject(NonNullableFormBuilder).group({ name: [''] });

  hasUnsavedChanges(): boolean {
    return this.form.dirty;
  }

  onBeforeUnload(event: BeforeUnloadEvent): void {
    // 浏览器只显示通用提示文案，无法自定义
    if (this.hasUnsavedChanges()) event.preventDefault();
  }

  save(): void {
    // 保存成功后标记为 pristine，避免跳转时再次提示
    this.form.markAsPristine();
  }
}
```

路由：`{ path: 'admin', canMatch: [roleGuard], data: { roles: ['admin'] }, loadChildren: ... }`

**易错点**

- 用 `canActivate` 做权限 → 懒加载代码仍然被下载
- 保存后没有 `markAsPristine`，跳转时仍然提示
- 只处理路由离开，没处理刷新和关闭
- 把前端守卫当作权限控制（服务端必须校验）

---

### H5. 防抖搜索（Signals + RxJS）

- **难度**: 中级
- **标签**: RxJS, Signals
- **面试官看什么**: switchMap 防竞态、内部 catchError、加载状态、Signal 与 Observable 互转

**题干**

实现用户搜索组件：输入后 300ms 防抖，少于 2 个字符不搜索，展示加载中、结果和错误状态；慢请求的结果不能覆盖新请求的结果。

**参考实现**

```ts
import { Component, computed, inject, signal } from '@angular/core';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { Observable, catchError, debounceTime, distinctUntilChanged, map, of, startWith, switchMap } from 'rxjs';

type SearchState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; items: User[] }
  | { status: 'error' };

/**
 * 用户搜索：防抖、最短长度、加载与错误状态，并通过 switchMap 丢弃过期请求。
 */
@Component({
  selector: 'app-user-search',
  template: `
    <input [value]="query()" (input)="query.set($any($event.target).value)" placeholder="搜索用户" />
    @switch (state().status) {
      @case ('loading') { <p>搜索中…</p> }
      @case ('error') { <p>搜索失败，请重试</p> }
      @case ('success') {
        @for (u of items(); track u.id) { <li>{{ u.name }}</li> } @empty { <p>无结果</p> }
      }
    }
  `,
})
export class UserSearch {
  private api = inject(UserApi);
  query = signal('');

  state = toSignal(
    toObservable(this.query).pipe(
      map((q) => q.trim()),
      debounceTime(300),
      distinctUntilChanged(),
      switchMap((q): Observable<SearchState> =>
        q.length < 2
          ? of({ status: 'idle' })
          : this.api.search(q).pipe(
              map((items) => ({ status: 'success', items }) as const),
              startWith({ status: 'loading' } as const),
              // 必须在内部捕获：外层流出错会终止，之后的输入都不会再搜索
              catchError(() => of({ status: 'error' } as const)),
            ),
      ),
    ),
    { initialValue: { status: 'idle' } as SearchState },
  );

  items = computed(() => {
    const s = this.state();
    return s.status === 'success' ? s.items : [];
  });
}
```

**易错点**

- 用 `mergeMap` → 慢请求的结果覆盖新结果
- `catchError` 放在外层 → 一次失败后搜索永久失效
- 没有 `distinctUntilChanged` → 输入后又删回原值时重复请求
- 先 `debounceTime` 再 `trim` → 只增加空格也会触发请求
- `toSignal` 没有 `initialValue` → 类型包含 `undefined`

---

### H6. 权限控制结构指令

- **难度**: 中级
- **标签**: 指令, 结构指令
- **面试官看什么**: TemplateRef / ViewContainerRef、避免重复创建视图、else 模板、响应权限变化

**题干**

实现 `*appHasPermission="'order:edit'; else noAccess"`：用户拥有权限时渲染内容，否则渲染 else 模板；用户权限变化（重新登录）时自动更新。

**参考实现**

```ts
import { Directive, TemplateRef, ViewContainerRef, effect, inject, input } from '@angular/core';

/**
 * 根据当前用户的权限决定是否渲染模板，支持 else 模板。
 */
@Directive({ selector: '[appHasPermission]' })
export class HasPermission {
  appHasPermission = input.required<string | string[]>();
  appHasPermissionElse = input<TemplateRef<unknown> | null>(null);

  private tpl = inject(TemplateRef<unknown>);
  private vcr = inject(ViewContainerRef);
  private permissions = inject(PermissionService); // permissions() 是 Signal<Set<string>>
  private rendered: 'main' | 'else' | null = null;

  constructor() {
    effect(() => {
      const required = this.appHasPermission();
      const list = Array.isArray(required) ? required : [required];
      const owned = this.permissions.permissions();
      const allowed = list.every((p) => owned.has(p));
      const elseTpl = this.appHasPermissionElse();

      const target = allowed ? 'main' : elseTpl ? 'else' : null;
      // 状态没变时不重建视图，否则子组件的状态会丢失
      if (target === this.rendered) return;

      this.vcr.clear();
      if (target === 'main') this.vcr.createEmbeddedView(this.tpl);
      else if (target === 'else') this.vcr.createEmbeddedView(elseTpl!);
      this.rendered = target;
    });
  }
}
```

使用：

```html
<button *appHasPermission="'order:edit'; else readonly">编辑</button>
<ng-template #readonly><span>只读</span></ng-template>
```

**易错点**

- 每次检查都 `clear` + `create` → 闪烁、子组件状态丢失
- 不响应权限变化
- 微语法中 `else` 对应的输入名必须是「指令名 + Else」
- 前端隐藏按钮不等于权限控制

---

### H7. 用 NgRx SignalStore 实现待办 Store

- **难度**: 中级
- **标签**: 状态管理, SignalStore
- **面试官看什么**: 状态 / 派生 / 方法的划分、异步加载、乐观更新与回滚

**题干**

用 `@ngrx/signals` 实现待办 Store：加载列表、筛选、剩余数量、切换完成状态（乐观更新，失败回滚）。

**参考实现**

```ts
import { computed, inject } from '@angular/core';
import { patchState, signalStore, withComputed, withMethods, withState } from '@ngrx/signals';
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { tapResponse } from '@ngrx/operators';
import { firstValueFrom, pipe, switchMap, tap } from 'rxjs';

type Filter = 'all' | 'active' | 'done';

interface TodosState {
  todos: Todo[];
  filter: Filter;
  loading: boolean;
  error: string | null;
}

/**
 * 待办列表的状态管理：加载、筛选、乐观切换完成状态。
 */
export const TodosStore = signalStore(
  withState<TodosState>({ todos: [], filter: 'all', loading: false, error: null }),

  withComputed(({ todos, filter }) => ({
    visible: computed(() => {
      const f = filter();
      return f === 'all' ? todos() : todos().filter((t) => (f === 'done' ? t.done : !t.done));
    }),
    remaining: computed(() => todos().filter((t) => !t.done).length),
  })),

  withMethods((store, api = inject(TodoApi)) => ({
    setFilter(filter: Filter): void {
      patchState(store, { filter });
    },

    load: rxMethod<void>(
      pipe(
        tap(() => patchState(store, { loading: true, error: null })),
        switchMap(() =>
          api.list().pipe(
            tapResponse({
              next: (todos) => patchState(store, { todos, loading: false }),
              error: (e: Error) => patchState(store, { loading: false, error: e.message }),
            }),
          ),
        ),
      ),
    ),

    async toggle(id: string): Promise<void> {
      const flip = (list: Todo[]) => list.map((t) => (t.id === id ? { ...t, done: !t.done } : t));
      patchState(store, { todos: flip(store.todos()) });
      try {
        await firstValueFrom(api.toggle(id));
      } catch {
        // 只回滚这一项，而不是恢复整个快照：期间其他项的修改不应被覆盖
        patchState(store, { todos: flip(store.todos()), error: '操作失败，已撤销' });
      }
    },
  })),
);

// 组件级：@Component({ providers: [TodosStore] })；全局：signalStore({ providedIn: 'root' }, ...)
```

**易错点**

- 在组件中直接修改 `todos` 数组（应通过方法和 `patchState`）
- 加载用 `mergeMap` → 多次加载的结果乱序
- 回滚时恢复整个快照 → 覆盖期间的其他修改
- 错误没有在内部处理 → `rxMethod` 的流终止

---

### H8. 无限滚动指令

- **难度**: 中级
- **标签**: 指令, IntersectionObserver
- **面试官看什么**: IntersectionObserver、SSR 兼容、清理、加载完成后仍然可见时继续加载

**题干**

实现放在列表末尾的哨兵元素上的指令：进入视口时触发 `scrolled`；加载中（`disabled`）时不触发；加载完成后哨兵仍然可见（一页数据不满一屏）时继续触发。

**参考实现**

```ts
import {
  DestroyRef, Directive, ElementRef, afterNextRender, booleanAttribute,
  effect, inject, input, output, signal, untracked,
} from '@angular/core';

/**
 * 哨兵元素进入视口且未禁用时触发 scrolled，用于加载下一页。
 */
@Directive({ selector: '[appInfiniteScroll]' })
export class InfiniteScroll {
  disabled = input(false, { alias: 'infiniteScrollDisabled', transform: booleanAttribute });
  rootMargin = input('200px');
  scrolled = output<void>();

  private el = inject(ElementRef<HTMLElement>);
  private destroyRef = inject(DestroyRef);
  private visible = signal(false);

  constructor() {
    // 只在浏览器中执行：SSR 中没有 IntersectionObserver
    afterNextRender(() => {
      const observer = new IntersectionObserver(
        ([entry]) => this.visible.set(entry.isIntersecting),
        { rootMargin: this.rootMargin() },
      );
      observer.observe(this.el.nativeElement);
      this.destroyRef.onDestroy(() => observer.disconnect());
    });

    // 同时依赖 visible 和 disabled：
    // IntersectionObserver 只在相交状态变化时回调，若加载完成后哨兵仍在视口内，
    // 不会有新的回调；这里在 disabled 变回 false 时重新判断，从而继续加载
    effect(() => {
      if (this.visible() && !this.disabled()) {
        untracked(() => this.scrolled.emit());
      }
    });
  }
}
```

使用：

```html
@for (item of items(); track item.id) { <app-row [item]="item" /> }
<div appInfiniteScroll [infiniteScrollDisabled]="loading() || !hasMore()" (scrolled)="loadMore()"></div>
```

**易错点**

- 监听 `scroll` 事件并计算位置（性能差）
- 不传入 `disabled` → 加载中重复触发
- 加载完成后哨兵仍可见时不再触发
- 构造函数中直接使用 `IntersectionObserver` → SSR 报错
- 忘记 `disconnect`

---

### H9. 配置驱动的动态表单

- **难度**: 高级
- **标签**: 表单, 动态
- **面试官看什么**: 根据配置构建 FormGroup、条件显示字段、隐藏字段不参与校验和提交

**题干**

根据 JSON 配置渲染表单：支持文本、数字、下拉、复选框；支持必填；支持「当某字段等于某值时才显示」。隐藏的字段不参与校验，也不出现在提交的值中。

**参考实现**

```ts
import { Component, computed, effect, input, output } from '@angular/core';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { FormControl, FormGroup, ReactiveFormsModule, Validators } from '@angular/forms';
import { map, startWith, switchMap } from 'rxjs';

/** 单个字段的配置 */
export interface FieldConfig {
  key: string;
  label: string;
  type: 'text' | 'number' | 'select' | 'checkbox';
  required?: boolean;
  options?: { label: string; value: string }[];
  /** 只有当 key 字段的值等于 equals 时才显示 */
  visibleWhen?: { key: string; equals: unknown };
}

function buildGroup(fields: FieldConfig[]): FormGroup {
  const controls: Record<string, FormControl> = {};
  for (const f of fields) {
    const initial = f.type === 'checkbox' ? false : null;
    // checkbox 的必填含义是「必须勾选」
    const validators = f.required ? [f.type === 'checkbox' ? Validators.requiredTrue : Validators.required] : [];
    controls[f.key] = new FormControl(initial, validators);
  }
  return new FormGroup(controls);
}

/**
 * 根据字段配置渲染响应式表单，支持条件显示。
 */
@Component({
  selector: 'app-dynamic-form',
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form()" (ngSubmit)="submit()">
      @for (f of fields(); track f.key) {
        @if (visibleKeys().has(f.key)) {
          <label>
            {{ f.label }}
            @switch (f.type) {
              @case ('select') {
                <select [formControlName]="f.key">
                  @for (o of f.options ?? []; track o.value) { <option [value]="o.value">{{ o.label }}</option> }
                </select>
              }
              @case ('checkbox') { <input type="checkbox" [formControlName]="f.key" /> }
              @default { <input [type]="f.type" [formControlName]="f.key" /> }
            }
          </label>
        }
      }
      <button type="submit">提交</button>
    </form>
  `,
})
export class DynamicForm {
  fields = input.required<FieldConfig[]>();
  submitted = output<Record<string, unknown>>();

  // 配置变化时重建表单；表单是配置的派生物
  form = computed(() => buildGroup(this.fields()));

  // getRawValue 包含被禁用（隐藏）的字段，条件判断才不会因为字段被隐藏而失效
  private values = toSignal(
    toObservable(this.form).pipe(
      switchMap((g) => g.valueChanges.pipe(startWith(null), map(() => g.getRawValue()))),
    ),
    { initialValue: {} as Record<string, unknown> },
  );

  visibleKeys = computed(() => {
    const values = this.values();
    return new Set(
      this.fields()
        .filter((f) => !f.visibleWhen || values[f.visibleWhen.key] === f.visibleWhen.equals)
        .map((f) => f.key),
    );
  });

  constructor() {
    // 隐藏的字段禁用：被禁用的控件不参与校验，也不出现在 form.value 中
    effect(() => {
      const visible = this.visibleKeys();
      const group = this.form();
      for (const [key, control] of Object.entries(group.controls)) {
        const shouldEnable = visible.has(key);
        // emitEvent: false 避免触发 valueChanges 导致循环
        if (shouldEnable && control.disabled) control.enable({ emitEvent: false });
        if (!shouldEnable && control.enabled) control.disable({ emitEvent: false });
      }
    });
  }

  submit(): void {
    const group = this.form();
    if (group.invalid) {
      group.markAllAsTouched();
      return;
    }
    this.submitted.emit(group.value); // 不含隐藏字段
  }
}
```

**易错点**

- 隐藏字段只是不显示，但仍然参与校验 → 表单永远无效
- 条件判断用 `value`（不含禁用字段）→ 被隐藏的条件字段导致判断出错
- `enable/disable` 触发 `valueChanges` → 循环
- checkbox 的必填用 `required` 而不是 `requiredTrue`
- 配置来自服务端时，要校验字段类型，避免渲染任意内容

---

### H10. 全局错误处理与代码块加载失败

- **难度**: 中级
- **标签**: 错误处理
- **面试官看什么**: ErrorHandler、未处理的 Promise、部署后的 ChunkLoadError、防止无限刷新

**题干**

实现全局错误处理：上报监控；发布新版本后旧的懒加载代码块不存在导致加载失败时，自动刷新一次页面（不能无限刷新）。

**参考实现**

```ts
import { ErrorHandler, Injectable, inject, provideBrowserGlobalErrorListeners } from '@angular/core';

const RELOAD_KEY = 'chunk-reload-at';

function isChunkLoadError(error: unknown): boolean {
  const message = error instanceof Error ? error.message : String(error);
  // 不同浏览器 / 构建工具的报错文案不同
  return /Loading chunk [\w-]+ failed|Failed to fetch dynamically imported module|Importing a module script failed|error loading dynamically imported module/i.test(message);
}

/**
 * 全局错误处理：代码块加载失败时自动刷新一次，其余错误上报监控。
 */
@Injectable()
export class AppErrorHandler implements ErrorHandler {
  // 不要在 ErrorHandler 中注入 Router 等依赖较多的服务：ErrorHandler 创建很早，容易产生循环依赖
  private monitor = inject(MonitorService);

  handleError(error: unknown): void {
    // Promise 中的错误可能被包装在 rejection 中
    const actual = (error as { rejection?: unknown })?.rejection ?? error;

    if (isChunkLoadError(actual) && this.reloadOnce()) return;

    this.monitor.capture(actual, { url: location.href });
    console.error(actual);
  }

  /** 10 秒内只自动刷新一次，防止新版本也加载失败时无限刷新 */
  private reloadOnce(): boolean {
    const last = Number(sessionStorage.getItem(RELOAD_KEY) ?? 0);
    if (Date.now() - last < 10_000) return false;
    sessionStorage.setItem(RELOAD_KEY, String(Date.now()));
    location.reload();
    return true;
  }
}

// app.config.ts
export const appConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(), // 把 window 的 error 和 unhandledrejection 交给 ErrorHandler
    { provide: ErrorHandler, useClass: AppErrorHandler },
  ],
};
```

**易错点**

- 无条件 `location.reload()` → 新版本也失败时无限刷新
- ErrorHandler 中注入 Router 等导致循环依赖
- 只处理 Angular 内部的错误，忽略 `window` 的未处理异常
- 根本解决还需要部署侧保留旧版本的代码块一段时间

---

### H11. 页面可见时轮询

- **难度**: 中级
- **标签**: RxJS, 轮询
- **面试官看什么**: 页面隐藏时暂停、请求不重叠、错误不终止轮询、SSR

**题干**

实现 `poll(request, intervalMs)`：立即请求一次，然后按间隔轮询；页面隐藏时暂停，恢复时立即请求；上一次请求未完成时跳过本次；单次失败不终止轮询；服务端渲染时不轮询。

**参考实现**

```ts
import { DOCUMENT, isPlatformBrowser } from '@angular/common';
import { PLATFORM_ID, inject } from '@angular/core';
import { EMPTY, Observable, catchError, distinctUntilChanged, exhaustMap, fromEvent, map, startWith, switchMap, timer } from 'rxjs';

/**
 * 页面可见时按间隔轮询，隐藏时暂停。需要在注入上下文中调用。
 */
export function poll<T>(request: () => Observable<T>, intervalMs: number): Observable<T> {
  // 服务端渲染时不轮询，否则应用永远不会稳定，HTML 无法输出
  if (!isPlatformBrowser(inject(PLATFORM_ID))) return EMPTY;
  const doc = inject(DOCUMENT);

  const visible$ = fromEvent(doc, 'visibilitychange').pipe(
    startWith(null),
    map(() => doc.visibilityState === 'visible'),
    distinctUntilChanged(),
  );

  return visible$.pipe(
    // 隐藏时切换到 EMPTY，自动取消计时器和进行中的请求
    switchMap((visible) =>
      visible
        ? timer(0, intervalMs).pipe(
            // 上一次请求未完成时忽略本次触发，避免请求堆积
            exhaustMap(() => request().pipe(catchError(() => EMPTY))),
          )
        : EMPTY,
    ),
  );
}

// 组件中：
// stats = toSignal(poll(() => this.api.getStats(), 5000));
```

**易错点**

- 用 `setInterval` → 页面隐藏时仍然请求，请求可能重叠
- 用 `mergeMap` → 慢请求堆积
- 错误没有在内部捕获 → 一次失败后轮询停止
- 服务端渲染时启动轮询 → SSR 永远不稳定
- 数据实时性要求高时，应该考虑 SSE / WebSocket 而不是轮询

---

### H12. Tabs 复合组件

- **难度**: 高级
- **标签**: 组件设计, 内容投影
- **面试官看什么**: contentChildren、ng-template 延迟渲染、无障碍、键盘导航

**题干**

实现 `<app-tabs>` 和 `<app-tab>`：支持双向绑定选中索引；面板内容支持懒渲染（未选中过的标签页不创建内容）；符合 ARIA 标签页模式，支持左右方向键切换。

**参考实现**

```ts
import { NgTemplateOutlet } from '@angular/common';
import {
  Component, Directive, TemplateRef, computed, contentChild, contentChildren,
  inject, input, model, viewChild,
} from '@angular/core';

/** 用于懒渲染面板内容：<ng-template appTabContent>...</ng-template> */
@Directive({ selector: 'ng-template[appTabContent]' })
export class TabContent {
  readonly tpl = inject(TemplateRef<unknown>);
}

/**
 * 单个标签页，提供标题和面板内容模板。
 */
@Component({
  selector: 'app-tab',
  // ng-content 投影的内容即使不显示也会被创建；需要懒渲染时使用 appTabContent
  template: `<ng-template #eager><ng-content /></ng-template>`,
})
export class Tab {
  label = input.required<string>();
  private lazy = contentChild(TabContent);
  private eager = viewChild.required<TemplateRef<unknown>>('eager');
  readonly content = computed(() => this.lazy()?.tpl ?? this.eager());
}

let nextId = 0;

/**
 * 标签页容器：管理选中状态、键盘导航和 ARIA 属性。
 */
@Component({
  selector: 'app-tabs',
  imports: [NgTemplateOutlet],
  template: `
    <div role="tablist" (keydown)="onKeydown($event)">
      @for (tab of tabs(); track tab; let i = $index) {
        <button
          role="tab"
          [id]="baseId + '-tab-' + i"
          [attr.aria-selected]="i === current()"
          [attr.aria-controls]="baseId + '-panel'"
          [tabIndex]="i === current() ? 0 : -1"
          (click)="selectedIndex.set(i)"
        >{{ tab.label() }}</button>
      }
    </div>
    <div role="tabpanel" [id]="baseId + '-panel'" [attr.aria-labelledby]="baseId + '-tab-' + current()">
      @if (activeTab(); as tab) {
        <ng-container [ngTemplateOutlet]="tab.content()" />
      }
    </div>
  `,
})
export class Tabs {
  selectedIndex = model(0);
  tabs = contentChildren(Tab);
  protected baseId = `tabs-${nextId++}`;

  // 标签页被动态删除时，索引可能越界
  protected current = computed(() =>
    Math.min(Math.max(this.selectedIndex(), 0), Math.max(this.tabs().length - 1, 0)),
  );
  protected activeTab = computed(() => this.tabs()[this.current()]);

  protected onKeydown(event: KeyboardEvent): void {
    const count = this.tabs().length;
    if (!count) return;
    const delta = event.key === 'ArrowRight' ? 1 : event.key === 'ArrowLeft' ? -1 : 0;
    if (!delta) return;
    event.preventDefault();
    const next = (this.current() + delta + count) % count;
    this.selectedIndex.set(next);
    // 漫游 tabindex：焦点跟随选中项
    (event.currentTarget as HTMLElement).querySelectorAll<HTMLElement>('[role="tab"]')[next]?.focus();
  }
}
```

使用：

```html
<app-tabs [(selectedIndex)]="tab">
  <app-tab label="概览"><app-overview /></app-tab>
  <app-tab label="日志">
    <ng-template appTabContent><app-heavy-log-viewer /></ng-template>  <!-- 懒渲染 -->
  </app-tab>
</app-tabs>
```

**易错点**

- 直接用 `@if` 包裹 `ng-content` 以为可以懒渲染（投影内容总会被创建）
- 选中索引越界
- 缺少 `role`、`aria-selected`、`aria-controls`，以及漫游 tabindex
- 用 `@for` 的 `$index` 作为 `track`，标签页增删时错误复用
- 实际项目中可以直接使用 Angular Aria / CDK 提供的标签页原语
