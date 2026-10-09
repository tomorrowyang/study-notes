# 前端工程化手写题

> 目标：通过手写理解 Monorepo 任务调度、缓存、版本解析、微前端沙箱和加载器、发布相关工具的核心原理。代码以 TypeScript 为主，Node 20+。

## 使用方式

- **Monorepo 原理**：H1 任务调度、H2 缓存键、H3 受影响分析、H12 模块边界
- **版本与发布**：H4 semver、H10 变更日志
- **微前端**：H5 沙箱、H6 HTML Entry、H7 通信、H11 远程加载
- **CI/CD 与发布**：H8 工作流、H9 灰度分桶

---

### H1. 实现带并发限制的拓扑任务调度器

- **难度**: 高级
- **标签**: 任务图, 拓扑排序
- **面试官看什么**: 依赖图、环检测、并发控制、失败处理

**题干**

给定任务和依赖（`build` 依赖其依赖包的 `build`），按依赖顺序执行，最多同时执行 N 个任务；有环时报错；某个任务失败时，依赖它的任务不再执行。

**思路口述**

记录每个任务的未完成依赖数（入度），入度为 0 的任务进入就绪队列；每完成一个任务，把依赖它的任务入度减 1。用正在运行的数量控制并发。

**参考实现**

```ts
/** 任务图中的一个任务 */
export interface Task {
  id: string;
  deps: string[];
  run: () => Promise<void>;
}

/** 执行结果：成功、失败、因上游失败而跳过 */
export interface RunResult {
  succeeded: string[];
  failed: string[];
  skipped: string[];
}

/**
 * 按拓扑顺序执行任务，最多同时执行 concurrency 个。
 * 任务失败时，所有直接或间接依赖它的任务都会被跳过。
 */
export async function runTaskGraph(tasks: Task[], concurrency = 4): Promise<RunResult> {
  const byId = new Map(tasks.map((t) => [t.id, t]));
  const pending = new Map<string, number>();
  const dependents = new Map<string, string[]>();

  for (const t of tasks) {
    pending.set(t.id, t.deps.length);
    for (const d of t.deps) {
      if (!byId.has(d)) throw new Error(`任务 ${t.id} 依赖不存在的任务 ${d}`);
      dependents.set(d, [...(dependents.get(d) ?? []), t.id]);
    }
  }

  assertNoCycle(tasks, dependents, pending);

  const result: RunResult = { succeeded: [], failed: [], skipped: [] };
  const ready = tasks.filter((t) => t.deps.length === 0).map((t) => t.id);
  let running = 0;

  return new Promise((resolve) => {
    const finishedCount = () =>
      result.succeeded.length + result.failed.length + result.skipped.length;

    // 失败的传播要递归：A 失败 → B 跳过 → 依赖 B 的 C 也要跳过
    const skipDependents = (id: string) => {
      for (const dep of dependents.get(id) ?? []) {
        if (result.skipped.includes(dep)) continue;
        result.skipped.push(dep);
        skipDependents(dep);
      }
    };

    const schedule = () => {
      while (running < concurrency && ready.length > 0) {
        const id = ready.shift()!;
        running++;
        byId
          .get(id)!
          .run()
          .then(
            () => {
              result.succeeded.push(id);
              for (const dep of dependents.get(id) ?? []) {
                const left = pending.get(dep)! - 1;
                pending.set(dep, left);
                // 已经因为其他依赖失败而被跳过的任务，不能再进入就绪队列
                if (left === 0 && !result.skipped.includes(dep)) ready.push(dep);
              }
            },
            () => {
              result.failed.push(id);
              skipDependents(id);
            },
          )
          .finally(() => {
            running--;
            if (finishedCount() === tasks.length) resolve(result);
            else schedule();
          });
      }
    };

    if (tasks.length === 0) resolve(result);
    else schedule();
  });
}

/** Kahn 算法检测环：能出队的任务数小于总数说明存在环 */
function assertNoCycle(
  tasks: Task[],
  dependents: Map<string, string[]>,
  pending: Map<string, number>,
) {
  const indegree = new Map(pending);
  const queue = tasks.filter((t) => indegree.get(t.id) === 0).map((t) => t.id);
  let visited = 0;
  while (queue.length) {
    const id = queue.shift()!;
    visited++;
    for (const dep of dependents.get(id) ?? []) {
      const left = indegree.get(dep)! - 1;
      indegree.set(dep, left);
      if (left === 0) queue.push(dep);
    }
  }
  if (visited !== tasks.length) {
    const cyclic = tasks.filter((t) => indegree.get(t.id)! > 0).map((t) => t.id);
    throw new Error(`任务图存在环：${cyclic.join(', ')}`);
  }
}
```

**易错点**

- 失败只跳过直接依赖者，没有递归跳过间接依赖者
- 被跳过的任务在另一个依赖完成后又被放入就绪队列
- 没有处理空任务列表，Promise 永远不 resolve
- 环检测放在执行过程中，导致部分任务已经执行

---

### H2. 实现任务缓存键的计算

- **难度**: 中级
- **标签**: 缓存, 哈希
- **面试官看什么**: 输入的完整性、确定性（排序）、依赖的传递

**题干**

缓存键由以下内容计算：任务名、输入文件的内容、声明的环境变量、依赖包的缓存键。要求结果稳定（与文件遍历顺序无关）。

**参考实现**

```ts
import { createHash } from 'node:crypto';
import { readFile } from 'node:fs/promises';
import { glob } from 'node:fs/promises'; // Node 22+；更早版本可使用 fast-glob

/** 计算缓存键所需的输入 */
export interface HashInput {
  task: string;
  packageDir: string;
  inputGlobs: string[];
  envKeys: string[];
  depHashes: Record<string, string>;
  /** 全局依赖的哈希（锁文件中本包相关部分、根配置等） */
  globalHash: string;
}

/**
 * 计算任务的缓存键。
 * 所有集合都排序后再参与哈希，保证结果与遍历顺序无关。
 */
export async function computeTaskHash(input: HashInput): Promise<string> {
  const hash = createHash('sha256');
  hash.update(`task:${input.task}\n`);
  hash.update(`global:${input.globalHash}\n`);

  const files = new Set<string>();
  for (const pattern of input.inputGlobs) {
    for await (const f of glob(pattern, { cwd: input.packageDir })) {
      files.add(f.replaceAll('\\', '/')); // 统一路径分隔符，Windows 和 Linux 的哈希才一致
    }
  }
  for (const file of [...files].sort()) {
    const content = await readFile(`${input.packageDir}/${file}`);
    // 路径和内容都参与：文件改名也应该使缓存失效
    hash.update(`file:${file}:${sha256(content)}\n`);
  }

  for (const key of [...input.envKeys].sort()) {
    // 未设置和空字符串要区分
    const value = process.env[key];
    hash.update(`env:${key}=${value === undefined ? '<unset>' : sha256(value)}\n`);
  }

  for (const name of Object.keys(input.depHashes).sort()) {
    hash.update(`dep:${name}:${input.depHashes[name]}\n`);
  }

  return hash.digest('hex');
}

function sha256(data: string | Buffer): string {
  return createHash('sha256').update(data).digest('hex');
}
```

**易错点**

- 文件没有排序，不同机器上哈希不同
- 路径分隔符不统一
- 只哈希内容不哈希路径
- 环境变量的值直接写入（日志中可能泄露）；这里对值再做一次哈希
- 依赖包的哈希没有参与，依赖修改后命中旧缓存

---

### H3. 根据变更文件计算受影响的包

- **难度**: 中级
- **标签**: 受影响分析
- **面试官看什么**: 文件到包的映射、反向依赖的传递、全局文件

**参考实现**

```ts
/** 工作区中的一个包 */
export interface WorkspacePackage {
  name: string;
  dir: string; // 相对仓库根目录，例如 packages/ui
  deps: string[]; // 依赖的工作区内部包
}

/**
 * 计算受影响的包：直接修改的包 + 所有直接或间接依赖它们的包。
 * 修改了全局文件（锁文件、根配置）时视为全部受影响。
 */
export function getAffectedPackages(
  packages: WorkspacePackage[],
  changedFiles: string[],
  globalFiles: string[] = ['pnpm-lock.yaml', 'turbo.json', 'tsconfig.base.json'],
): Set<string> {
  const normalized = changedFiles.map((f) => f.replaceAll('\\', '/'));

  if (normalized.some((f) => globalFiles.includes(f))) {
    return new Set(packages.map((p) => p.name));
  }

  // 目录长的优先匹配，处理嵌套目录（packages/ui 和 packages/ui-kit）
  const sorted = [...packages].sort((a, b) => b.dir.length - a.dir.length);
  const directlyChanged = new Set<string>();
  for (const file of normalized) {
    const owner = sorted.find((p) => file === p.dir || file.startsWith(p.dir + '/'));
    if (owner) directlyChanged.add(owner.name);
  }

  const reverseDeps = new Map<string, string[]>();
  for (const p of packages) {
    for (const d of p.deps) {
      reverseDeps.set(d, [...(reverseDeps.get(d) ?? []), p.name]);
    }
  }

  const affected = new Set(directlyChanged);
  const queue = [...directlyChanged];
  while (queue.length) {
    const name = queue.shift()!;
    for (const dependent of reverseDeps.get(name) ?? []) {
      if (!affected.has(dependent)) {
        affected.add(dependent);
        queue.push(dependent);
      }
    }
  }
  return affected;
}
```

**易错点**

- `startsWith('packages/ui')` 会错误匹配 `packages/ui-kit/...`，要加 `/`
- 只计算直接依赖者，没有传递
- 忽略全局文件的变更

---

### H4. 实现 semver 比较和范围匹配（^、~）

- **难度**: 中级
- **标签**: semver
- **面试官看什么**: ^ 对 0.x 的特殊规则、预发布版本的比较

**参考实现**

```ts
/** 解析后的版本号 */
export interface SemVer {
  major: number;
  minor: number;
  patch: number;
  prerelease: (string | number)[];
}

/** 解析 `1.2.3` 或 `1.2.3-beta.1` 格式的版本号 */
export function parse(version: string): SemVer {
  const m = /^v?(\d+)\.(\d+)\.(\d+)(?:-([0-9A-Za-z.-]+))?(?:\+[0-9A-Za-z.-]+)?$/.exec(version.trim());
  if (!m) throw new Error(`无效的版本号：${version}`);
  return {
    major: Number(m[1]),
    minor: Number(m[2]),
    patch: Number(m[3]),
    prerelease: m[4] ? m[4].split('.').map((p) => (/^\d+$/.test(p) ? Number(p) : p)) : [],
  };
}

/** 比较两个版本：a < b 返回负数，相等返回 0，a > b 返回正数 */
export function compare(a: string, b: string): number {
  const va = parse(a);
  const vb = parse(b);
  for (const k of ['major', 'minor', 'patch'] as const) {
    if (va[k] !== vb[k]) return va[k] - vb[k];
  }
  // 有预发布标识的版本低于正式版本：1.0.0-beta < 1.0.0
  if (!va.prerelease.length || !vb.prerelease.length) {
    return vb.prerelease.length - va.prerelease.length;
  }
  const len = Math.max(va.prerelease.length, vb.prerelease.length);
  for (let i = 0; i < len; i++) {
    const x = va.prerelease[i];
    const y = vb.prerelease[i];
    if (x === undefined) return -1; // 1.0.0-beta < 1.0.0-beta.1
    if (y === undefined) return 1;
    if (x === y) continue;
    // 数字标识低于字符串标识
    if (typeof x === 'number' && typeof y === 'number') return x - y;
    if (typeof x === 'number') return -1;
    if (typeof y === 'number') return 1;
    return x < y ? -1 : 1;
  }
  return 0;
}

/**
 * 判断版本是否满足范围，支持精确版本、^、~、>=、<。
 * 简化：预发布版本只在范围本身也是同一 major.minor.patch 的预发布时才匹配（与 npm 一致）。
 */
export function satisfies(version: string, range: string): boolean {
  const r = range.trim();
  const v = parse(version);

  const op = /^(\^|~|>=|<)?(.+)$/.exec(r)!;
  const base = parse(op[2]);

  if (v.prerelease.length) {
    const sameTuple = v.major === base.major && v.minor === base.minor && v.patch === base.patch;
    if (!sameTuple || !base.prerelease.length) return false;
  }

  switch (op[1]) {
    case '^': {
      const upper =
        base.major > 0
          ? `${base.major + 1}.0.0`
          : base.minor > 0
            ? `0.${base.minor + 1}.0` // ^0.2.3 → <0.3.0
            : `0.0.${base.patch + 1}`; // ^0.0.3 → <0.0.4
      return compare(version, op[2]) >= 0 && compare(version, upper) < 0;
    }
    case '~':
      return compare(version, op[2]) >= 0 && compare(version, `${base.major}.${base.minor + 1}.0`) < 0;
    case '>=':
      return compare(version, op[2]) >= 0;
    case '<':
      return compare(version, op[2]) < 0;
    default:
      return compare(version, op[2]) === 0;
  }
}
```

**易错点**

- `^0.2.3` 的上界是 `0.3.0` 而不是 `1.0.0`
- 预发布版本的比较规则（数字 vs 字符串、长度）
- 按字符串比较版本号（`"1.10.0" < "1.9.0"`）

---

### H5. 实现快照沙箱和代理沙箱

- **难度**: 高级
- **标签**: 微前端, 沙箱
- **面试官看什么**: 两种沙箱的原理、代理沙箱的 has / 函数 this 处理

**参考实现**

```ts
/** 沙箱的统一接口 */
export interface Sandbox {
  proxy: WindowProxy | Window;
  active(): void;
  inactive(): void;
}

/**
 * 快照沙箱：激活时记录 window 快照，失活时 diff 并恢复。
 * 只能同时运行一个子应用；遍历 window 开销大，用于不支持 Proxy 的环境。
 */
export class SnapshotSandbox implements Sandbox {
  proxy = window;
  private snapshot = new Map<PropertyKey, unknown>();
  private modified = new Map<PropertyKey, unknown>();

  active() {
    this.snapshot.clear();
    for (const key of Object.keys(window)) {
      this.snapshot.set(key, (window as any)[key]);
    }
    // 重新激活时恢复该子应用上次的修改
    this.modified.forEach((value, key) => {
      (window as any)[key] = value;
    });
  }

  inactive() {
    this.modified.clear();
    for (const key of Object.keys(window)) {
      const current = (window as any)[key];
      if (current !== this.snapshot.get(key)) {
        this.modified.set(key, current);
        (window as any)[key] = this.snapshot.get(key);
      }
    }
    // 子应用新增的属性：快照中没有，需要删除
    for (const key of Object.keys(window)) {
      if (!this.snapshot.has(key)) delete (window as any)[key];
    }
  }
}

/**
 * 代理沙箱：子应用写入 fakeWindow，读取时优先 fakeWindow 再回退到真实 window。
 * 支持多个子应用同时运行，彼此的全局变量互不影响。
 */
export class ProxySandbox implements Sandbox {
  proxy: WindowProxy;
  private running = false;
  private fakeWindow: Record<PropertyKey, unknown> = Object.create(null);

  constructor() {
    const fake = this.fakeWindow;
    const boundFnCache = new WeakMap<Function, Function>();

    this.proxy = new Proxy(window, {
      get: (target, key) => {
        // 子应用中的 window.window / self / globalThis 也要指向代理，否则会逃逸
        if (key === 'window' || key === 'self' || key === 'globalThis') return this.proxy;
        if (key in fake) return fake[key];
        const value = (target as any)[key];
        // fetch、setTimeout 等原生函数用代理作为 this 调用会报 Illegal invocation，需要绑定真实 window。
        // 只绑定非构造函数（没有 prototype 的箭头函数 / 原生方法），避免破坏 class 和构造函数
        if (typeof value === 'function' && !isConstructable(value)) {
          let bound = boundFnCache.get(value);
          if (!bound) {
            bound = value.bind(target);
            boundFnCache.set(value, bound!);
          }
          return bound;
        }
        return value;
      },
      set: (_target, key, value) => {
        if (this.running) fake[key] = value;
        return true;
      },
      has: (target, key) => key in fake || key in target,
      deleteProperty: (_target, key) => {
        delete fake[key];
        return true;
      },
    }) as WindowProxy;
  }

  active() {
    this.running = true;
  }

  inactive() {
    this.running = false;
  }
}

/** 粗略判断是否是构造函数：class 或带有非空 prototype 的普通函数 */
function isConstructable(fn: Function): boolean {
  const proto = (fn as any).prototype;
  return !!proto && proto.constructor === fn && Object.getOwnPropertyNames(proto).length > 1;
}

/** 在沙箱中执行子应用代码：用 with 把全局变量的查找指向代理 */
export function execInSandbox(code: string, sandbox: Sandbox) {
  // with 在严格模式下不可用，所以外层函数不能是严格模式；子应用的代码本身可以是严格模式
  const fn = new Function('window', 'self', 'globalThis', `with(window){;${code}\n}`);
  fn.call(sandbox.proxy, sandbox.proxy, sandbox.proxy, sandbox.proxy);
}
```

**易错点**

- 没有代理 `window.window`，子应用通过它逃逸
- 原生函数的 this 问题（Illegal invocation）
- 把所有函数都 bind，破坏了构造函数和函数上的静态属性
- 子应用中 `var a = 1` 在 `with` 内执行时声明在函数作用域中，而不是 `window.a`；qiankun 的处理方式也有这个局限

---

### H6. 实现简化的 HTML Entry 加载器

- **难度**: 高级
- **标签**: 微前端, 加载
- **面试官看什么**: 解析 HTML、相对路径处理、执行顺序

**参考实现**

```ts
/** 解析出的子应用资源 */
export interface AppAssets {
  template: string;
  scripts: { src?: string; content?: string }[];
  styles: string[];
}

/**
 * 获取子应用的 HTML，提取脚本和样式，返回去掉脚本后的模板。
 * 相对路径以 entry 为基准解析，因为子应用部署在另一个域名或路径下。
 */
export async function importHtml(entry: string): Promise<AppAssets> {
  const html = await (await fetch(entry)).text();
  const doc = new DOMParser().parseFromString(html, 'text/html');
  const resolve = (url: string) => new URL(url, entry).href;

  const scripts: AppAssets['scripts'] = [];
  doc.querySelectorAll('script').forEach((el) => {
    // ES Module 脚本无法用 eval / with 放进沙箱执行，这里不支持（这也是 qiankun 与 Vite 不兼容的原因）
    if (el.type === 'module') {
      console.warn('[importHtml] 不支持 type="module" 的脚本：', el.src || '(inline)');
    } else if (el.src) {
      scripts.push({ src: resolve(el.getAttribute('src')!) });
    } else {
      scripts.push({ content: el.textContent ?? '' });
    }
    el.remove();
  });

  const styles: string[] = [];
  const styleTasks: Promise<void>[] = [];
  doc.querySelectorAll('link[rel="stylesheet"], style').forEach((el, i) => {
    if (el.tagName === 'LINK') {
      const href = resolve(el.getAttribute('href')!);
      // 保持样式的原始顺序，先占位再填充
      styleTasks.push(fetch(href).then((r) => r.text()).then((css) => void (styles[i] = css)));
    } else {
      styles[i] = el.textContent ?? '';
    }
    el.remove();
  });
  await Promise.all(styleTasks);

  doc.querySelectorAll('img[src], a[href]').forEach((el) => {
    const attr = el.tagName === 'IMG' ? 'src' : 'href';
    el.setAttribute(attr, resolve(el.getAttribute(attr)!));
  });

  return { template: doc.body.innerHTML, scripts, styles: styles.filter((s) => s !== undefined) };
}

/** 按顺序下载并执行脚本：并行下载，串行执行，保证与浏览器的执行顺序一致 */
export async function execScripts(
  scripts: AppAssets['scripts'],
  exec: (code: string, url?: string) => void,
) {
  const contents = await Promise.all(
    scripts.map((s) => (s.src ? fetch(s.src).then((r) => r.text()) : Promise.resolve(s.content!))),
  );
  contents.forEach((code, i) => exec(code, scripts[i].src));
}
```

**易错点**

- 相对路径以主应用为基准解析
- 脚本并行执行，破坏了依赖顺序
- 样式的顺序被打乱
- 没有意识到 ESM 脚本无法放进沙箱
- CSS 中的 `url()` 相对路径同样需要改写（这里省略）

---

### H7. 实现带命名空间的全局状态和事件总线

- **难度**: 中级
- **标签**: 微前端, 通信
- **面试官看什么**: 订阅清理、命名空间、防止误修改

**参考实现**

```ts
type Listener<T> = (state: T, prev: T) => void;

/**
 * 微前端的全局状态：主应用创建，子应用按名称获取受限的操作句柄。
 * 子应用卸载时调用 dispose 清理该子应用的所有订阅，避免泄漏。
 */
export class GlobalStore<T extends Record<string, unknown>> {
  private state: T;
  private listeners = new Map<string, Set<Listener<T>>>();

  constructor(initial: T) {
    this.state = Object.freeze({ ...initial });
  }

  /** 为某个子应用创建操作句柄 */
  connect(appName: string) {
    if (!this.listeners.has(appName)) this.listeners.set(appName, new Set());
    const own = this.listeners.get(appName)!;

    return {
      getState: () => this.state,
      setState: (patch: Partial<T>) => this.setState(patch),
      onChange: (fn: Listener<T>, fireImmediately = false) => {
        own.add(fn);
        if (fireImmediately) fn(this.state, this.state);
        return () => own.delete(fn);
      },
      dispose: () => this.listeners.delete(appName),
    };
  }

  private setState(patch: Partial<T>) {
    const prev = this.state;
    const changed = Object.keys(patch).some((k) => !Object.is(prev[k], patch[k]));
    if (!changed) return;
    // 冻结状态，防止子应用直接修改对象而绕过通知
    this.state = Object.freeze({ ...prev, ...patch });
    for (const set of this.listeners.values()) {
      for (const fn of set) {
        try {
          fn(this.state, prev);
        } catch (err) {
          // 一个子应用的监听器报错不能影响其他子应用
          console.error('[GlobalStore] listener error', err);
        }
      }
    }
  }
}
```

**易错点**

- 子应用卸载后没有清理订阅
- 一个监听器报错导致后续监听器不执行
- 状态对象可以被直接修改

---

### H8. 写一个安全且高效的 GitHub Actions 工作流

- **难度**: 中级
- **标签**: CI, GitHub Actions
- **面试官看什么**: 最小权限、SHA 固定、缓存、受影响分析、并发取消

**参考实现**

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

# 默认只读，需要更多权限的 job 单独声明
permissions:
  contents: read

# 同一 PR 的新提交取消旧的运行，节省资源
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  check:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      # 第三方 Action 固定完整的 commit SHA（注释中记录版本），防止标签被篡改；SHA 以实际版本为准
      - uses: actions/checkout@<sha> # v5
        with:
          fetch-depth: 0 # 受影响分析需要与主干比较
          persist-credentials: false # 后续步骤不需要 git 凭证

      - uses: pnpm/action-setup@<sha> # v4

      - uses: actions/setup-node@<sha> # v5
        with:
          node-version-file: .nvmrc
          cache: pnpm

      # 锁文件不一致时失败；依赖的安装脚本由 pnpm 的 allowBuilds 白名单控制
      - run: pnpm install --frozen-lockfile

      - name: Lint / Typecheck / Test / Build（只运行受影响的包）
        run: pnpm turbo run lint typecheck test build --affected
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ vars.TURBO_TEAM }}
          # PR 只读远程缓存，防止缓存投毒；主干才写入
          TURBO_REMOTE_CACHE_READ_ONLY: ${{ github.event_name == 'pull_request' }}
```

**说明**：

- 发布 job 单独声明 `id-token: write`（OIDC），并使用 `environment: production` 要求审批
- 不要把 `${{ github.event.pull_request.title }}` 等不可信输入直接写进 `run:`，应通过 `env` 传入
- 来自 fork 的 PR 不能访问 secrets；TURBO_TOKEN 为空时 Turborepo 只使用本地缓存

**易错点**

- 使用 `@v4` 这类可变标签
- 顶层 `permissions: write-all`
- 在 `pull_request_target` 中检出并执行 PR 代码

---

### H9. 实现灰度分桶中间件

- **难度**: 中级
- **标签**: 灰度
- **面试官看什么**: 稳定分桶、白名单、粘性、缓存影响

**题干**

Node 中间件根据用户决定返回新版本还是旧版本的 HTML：白名单用户直接新版本；其余按用户 ID 哈希，比例由配置决定；同一用户的结果稳定。

**参考实现**

```ts
import { createHash } from 'node:crypto';
import type { Request, Response, NextFunction } from 'express';

/** 灰度规则配置（通常来自配置中心） */
export interface CanaryConfig {
  version: string; // 灰度版本号
  percentage: number; // 0–100
  allowList: string[];
  /** 每次灰度使用不同的盐，避免每次都是同一批用户先拿到新版本 */
  salt: string;
}

/** 根据用户 ID 计算 0–99 的桶号，同一个用户和盐的结果稳定 */
export function bucketOf(userId: string, salt: string): number {
  const digest = createHash('md5').update(`${salt}:${userId}`).digest();
  return digest.readUInt32BE(0) % 100;
}

/**
 * 灰度中间件：在 res.locals.version 中写入命中的版本。
 * 匿名用户用 Cookie 中的设备 ID，保证刷新后的版本一致。
 */
export function canaryMiddleware(getConfig: () => CanaryConfig, stableVersion: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    const config = getConfig();
    let id: string | undefined = (req as any).user?.id ?? req.cookies?.did;
    if (!id) {
      id = crypto.randomUUID();
      res.cookie('did', id, { httpOnly: true, sameSite: 'lax', maxAge: 365 * 864e5 });
    }

    const hit =
      config.allowList.includes(id) || bucketOf(id, config.salt) < config.percentage;
    res.locals.version = hit ? config.version : stableVersion;

    // 不同用户返回不同的 HTML，不能被 CDN 或共享缓存缓存
    res.setHeader('Cache-Control', 'private, no-cache');
    res.setHeader('X-App-Version', res.locals.version);
    next();
  };
}
```

**易错点**

- 用 `Math.random()` 分桶，用户每次刷新版本不同
- HTML 被 CDN 缓存，灰度失效
- 匿名用户没有稳定的标识
- 所有灰度用同一个哈希，导致总是同一批用户承担风险

---

### H10. 根据 Conventional Commits 生成变更日志

- **难度**: 初级
- **标签**: 发布, 变更日志
- **面试官看什么**: 提交解析、破坏性变更识别、版本号推断

**参考实现**

```ts
/** 解析后的提交 */
export interface ParsedCommit {
  type: string;
  scope?: string;
  subject: string;
  breaking: boolean;
  hash: string;
}

const HEADER = /^(\w+)(?:\(([^)]+)\))?(!)?: (.+)$/;

/** 解析一条提交，不符合规范的返回 null */
export function parseCommit(hash: string, message: string): ParsedCommit | null {
  const [header, ...body] = message.split('\n');
  const m = HEADER.exec(header.trim());
  if (!m) return null;
  return {
    type: m[1],
    scope: m[2],
    subject: m[4],
    // 破坏性变更有两种写法：类型后的 ! 或正文中的 BREAKING CHANGE:
    breaking: !!m[3] || body.some((l) => /^BREAKING[ -]CHANGE:/.test(l)),
    hash,
  };
}

/** 根据提交推断下一次的版本升级类型 */
export function inferBump(commits: ParsedCommit[]): 'major' | 'minor' | 'patch' | null {
  if (commits.some((c) => c.breaking)) return 'major';
  if (commits.some((c) => c.type === 'feat')) return 'minor';
  if (commits.some((c) => c.type === 'fix' || c.type === 'perf')) return 'patch';
  return null;
}

/** 生成 Markdown 格式的变更日志 */
export function renderChangelog(version: string, commits: ParsedCommit[]): string {
  const sections: [string, (c: ParsedCommit) => boolean][] = [
    ['⚠ BREAKING CHANGES', (c) => c.breaking],
    ['Features', (c) => c.type === 'feat' && !c.breaking],
    ['Bug Fixes', (c) => c.type === 'fix' && !c.breaking],
    ['Performance', (c) => c.type === 'perf' && !c.breaking],
  ];
  const lines = [`## ${version} (${new Date().toISOString().slice(0, 10)})`, ''];
  for (const [title, match] of sections) {
    const items = commits.filter(match);
    if (!items.length) continue;
    lines.push(`### ${title}`, '');
    for (const c of items) {
      lines.push(`- ${c.scope ? `**${c.scope}:** ` : ''}${c.subject} (${c.hash.slice(0, 7)})`);
    }
    lines.push('');
  }
  return lines.join('\n');
}
```

**易错点**

- 漏掉 `BREAKING CHANGE` 写在正文中的情况
- 0.x 版本的破坏性变更通常只升级 minor（需要额外处理）

---

### H11. 加载远程模块：超时、重试和降级

- **难度**: 中级
- **标签**: 微前端, 容错
- **面试官看什么**: 超时实现、指数退避、去重、降级

**参考实现**

```ts
/** 加载选项 */
export interface LoadOptions {
  timeout?: number;
  retries?: number;
  fallbackUrls?: string[];
}

const scriptCache = new Map<string, Promise<void>>();

/**
 * 以 script 标签加载远程入口文件（如 remoteEntry.js），带超时、重试和备用地址。
 * 相同 URL 的并发请求共享同一个 Promise，失败后从缓存删除以允许重试。
 */
export function loadScript(url: string, timeout = 10_000): Promise<void> {
  const cached = scriptCache.get(url);
  if (cached) return cached;

  const p = new Promise<void>((resolve, reject) => {
    const el = document.createElement('script');
    el.src = url;
    el.async = true;
    const timer = setTimeout(() => {
      cleanup();
      reject(new Error(`加载超时：${url}`));
    }, timeout);
    const cleanup = () => {
      clearTimeout(timer);
      el.onload = el.onerror = null;
    };
    el.onload = () => {
      cleanup();
      resolve();
    };
    el.onerror = () => {
      cleanup();
      el.remove(); // 移除失败的标签，重试时重新插入
      reject(new Error(`加载失败：${url}`));
    };
    document.head.appendChild(el);
  });

  scriptCache.set(url, p);
  p.catch(() => scriptCache.delete(url));
  return p;
}

/** 依次尝试主地址和备用地址，每个地址按指数退避重试 */
export async function loadRemoteEntry(url: string, opts: LoadOptions = {}): Promise<void> {
  const { timeout = 10_000, retries = 2, fallbackUrls = [] } = opts;
  let lastError: unknown;
  for (const candidate of [url, ...fallbackUrls]) {
    for (let attempt = 0; attempt <= retries; attempt++) {
      try {
        return await loadScript(candidate, timeout);
      } catch (err) {
        lastError = err;
        if (attempt < retries) await sleep(500 * 2 ** attempt);
      }
    }
  }
  throw lastError;
}

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));
```

**使用（React 中配合错误边界降级）**：

```tsx
const RemoteOrders = lazy(async () => {
  try {
    await loadRemoteEntry('https://cdn.example.com/orders/remoteEntry.js', {
      fallbackUrls: ['https://cdn-backup.example.com/orders/remoteEntry.js'],
    });
    return await loadModuleFromRemote('orders', './App');
  } catch {
    return { default: () => <ErrorPlaceholder name="订单" /> };
  }
});
```

**易错点**

- 超时后脚本仍然加载完成，又执行了一次 resolve（这里通过 cleanup 解除）
- 失败的 Promise 留在缓存中，之后永远失败
- 失败的 script 标签留在 DOM 中

---

### H12. 根据标签检查模块边界

- **难度**: 中级
- **标签**: 模块边界
- **面试官看什么**: 规则表达、依赖图检查、清晰的报错

**题干**

每个包有标签（`type:app`、`type:feature`、`type:ui`、`type:util`，`scope:orders` 等）。规则：`type:ui` 只能依赖 `type:util`；`scope:orders` 只能依赖 `scope:orders` 和 `scope:shared`。输出所有违规。

**参考实现**

```ts
/** 带标签的包 */
export interface TaggedPackage {
  name: string;
  tags: string[];
  deps: string[];
}

/** 一条边界规则：带有 sourceTag 的包只能依赖带有 allowed 中任意标签的包 */
export interface BoundaryRule {
  sourceTag: string;
  allowed: string[];
}

/** 一个违规 */
export interface Violation {
  from: string;
  to: string;
  rule: BoundaryRule;
}

/** 检查所有包的依赖是否满足边界规则，返回所有违规（不在第一个违规处停止） */
export function checkBoundaries(packages: TaggedPackage[], rules: BoundaryRule[]): Violation[] {
  const byName = new Map(packages.map((p) => [p.name, p]));
  const violations: Violation[] = [];

  for (const pkg of packages) {
    const applicable = rules.filter((r) => pkg.tags.includes(r.sourceTag));
    for (const depName of pkg.deps) {
      const dep = byName.get(depName);
      if (!dep) continue; // 外部依赖不在检查范围
      for (const rule of applicable) {
        if (!dep.tags.some((t) => rule.allowed.includes(t))) {
          violations.push({ from: pkg.name, to: depName, rule });
        }
      }
    }
  }
  return violations;
}

/** 格式化违规信息，告诉开发者为什么不允许 */
export function formatViolations(violations: Violation[]): string {
  return violations
    .map(
      (v) =>
        `✖ ${v.from} → ${v.to}：带有 "${v.rule.sourceTag}" 的包只能依赖 [${v.rule.allowed.join(', ')}]`,
    )
    .join('\n');
}
```

**易错点**

- 多条规则同时适用时，每条都要满足（这里逐条检查）
- 发现第一个违规就停止，开发者需要反复运行
- 报错信息不说明规则，开发者不知道怎么修
