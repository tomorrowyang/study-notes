# 跨端开发手写题

> 目标：通过手写理解 JSBridge、setData 的优化、Taro 运行时、热更新、多端适配层等核心原理。代码以 TypeScript 为主；小程序代码使用微信原生 API。

## 使用方式

- **混合开发**：H1 JSBridge
- **小程序原理与优化**：H2 请求封装、H3 setData 差量、H4 setData 合并、H5 虚拟列表计算、H10 安全路由、H12 曝光埋点
- **跨端框架原理**：H8 迷你模拟 DOM
- **React Native**：H6 热更新与启动保护、H7 手势滑动删除
- **多端架构**：H9 适配层、H11 离线同步队列

---

### H1. 实现 H5 侧的 JSBridge

- **难度**: 中级
- **标签**: JSBridge, 混合开发
- **面试官看什么**: 回调 ID 机制、超时、事件订阅、原生未就绪的处理

**题干**

H5 调用原生：`bridge.call('getUserInfo', params)` 返回 Promise；原生处理完后调用 `window.__bridgeCallback(id, error, result)`。原生主动推送事件：`window.__bridgeEmit(event, data)`。原生通过 `window.webkit.messageHandlers.bridge.postMessage`（iOS）或 `window.AndroidBridge.postMessage`（Android）接收消息。

**参考实现**

```ts
type Pending = {
  resolve: (v: unknown) => void;
  reject: (e: Error) => void;
  timer: ReturnType<typeof setTimeout>;
};

/** 原生返回的错误 */
export class BridgeError extends Error {
  constructor(public code: string, message: string) {
    super(message);
  }
}

/**
 * H5 侧的 JSBridge：负责把调用发送给原生，并通过回调 ID 把结果对应回 Promise。
 * 原生通过全局函数 __bridgeCallback / __bridgeEmit 回调页面。
 */
export class JSBridge {
  private seq = 0;
  private pending = new Map<string, Pending>();
  private listeners = new Map<string, Set<(data: unknown) => void>>();

  constructor(private defaultTimeout = 10_000) {
    const w = window as any;
    w.__bridgeCallback = (id: string, error: { code: string; message: string } | null, result: unknown) => {
      const p = this.pending.get(id);
      // 超时后原生才返回的结果直接丢弃
      if (!p) return;
      clearTimeout(p.timer);
      this.pending.delete(id);
      error ? p.reject(new BridgeError(error.code, error.message)) : p.resolve(result);
    };
    w.__bridgeEmit = (event: string, data: unknown) => {
      this.listeners.get(event)?.forEach((fn) => {
        try {
          fn(data);
        } catch (err) {
          console.error('[JSBridge] listener error', err);
        }
      });
    };
  }

  /** 是否运行在 App 的 WebView 中 */
  get available(): boolean {
    const w = window as any;
    return !!(w.webkit?.messageHandlers?.bridge || w.AndroidBridge);
  }

  /** 调用原生方法 */
  call<T = unknown>(method: string, params: unknown = {}, timeout = this.defaultTimeout): Promise<T> {
    if (!this.available) {
      return Promise.reject(new BridgeError('NOT_IN_APP', '当前不在 App 环境中'));
    }
    const id = `cb_${Date.now()}_${++this.seq}`;
    return new Promise<T>((resolve, reject) => {
      const timer = setTimeout(() => {
        this.pending.delete(id);
        reject(new BridgeError('TIMEOUT', `调用 ${method} 超时`));
      }, timeout);
      this.pending.set(id, { resolve: resolve as (v: unknown) => void, reject, timer });
      this.post({ id, method, params });
    });
  }

  /** 订阅原生推送的事件，返回取消订阅函数 */
  on(event: string, fn: (data: unknown) => void): () => void {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(fn);
    return () => this.listeners.get(event)?.delete(fn);
  }

  private post(message: object) {
    const w = window as any;
    // 两端都只接受字符串，统一序列化，避免 Android 侧收到 [object Object]
    const payload = JSON.stringify(message);
    if (w.webkit?.messageHandlers?.bridge) {
      w.webkit.messageHandlers.bridge.postMessage(payload);
    } else {
      w.AndroidBridge.postMessage(payload);
    }
  }
}

export const bridge = new JSBridge();
```

**易错点**

- 回调 ID 不唯一（只用时间戳）
- 没有超时，原生没有回调时 Promise 永远挂起
- 超时后的迟到回调再次 resolve
- 不在 App 中时直接报错崩溃，没有降级
- 原生侧不校验页面域名（安全问题，见高级 Q21）

---

### H2. 封装小程序请求：自动登录和并发请求共享登录

- **难度**: 中级
- **标签**: 小程序, 请求
- **面试官看什么**: Promise 化、401 处理、single-flight、防止无限重试

**参考实现**

```ts
/** 请求选项 */
export interface RequestOptions {
  url: string;
  method?: 'GET' | 'POST' | 'PUT' | 'DELETE';
  data?: Record<string, unknown>;
  /** 是否需要登录态，登录接口本身为 false */
  auth?: boolean;
}

const BASE_URL = 'https://api.example.com';
const TOKEN_KEY = 'token';

let loginPromise: Promise<string> | null = null;

/**
 * 执行登录并返回 token。
 * 多个请求同时遇到 401 时共享同一次登录，避免并发调用 wx.login 导致 code 互相失效。
 */
function login(): Promise<string> {
  if (loginPromise) return loginPromise;
  loginPromise = new Promise<string>((resolve, reject) => {
    wx.login({
      success: ({ code }) => {
        rawRequest<{ token: string }>({ url: '/auth/login', method: 'POST', data: { code }, auth: false })
          .then(({ token }) => {
            wx.setStorageSync(TOKEN_KEY, token);
            resolve(token);
          })
          .catch(reject);
      },
      fail: reject,
    });
  }).finally(() => {
    loginPromise = null;
  });
  return loginPromise;
}

/** 请求错误，携带 HTTP 状态码 */
export class HttpError extends Error {
  constructor(public status: number, message: string) {
    super(message);
  }
}

function rawRequest<T>(opts: RequestOptions, token?: string): Promise<T> {
  return new Promise((resolve, reject) => {
    wx.request({
      url: BASE_URL + opts.url,
      method: opts.method ?? 'GET',
      data: opts.data,
      header: token ? { Authorization: `Bearer ${token}` } : {},
      success: (res) => {
        if (res.statusCode >= 200 && res.statusCode < 300) resolve(res.data as T);
        else reject(new HttpError(res.statusCode, (res.data as any)?.message ?? '请求失败'));
      },
      fail: (err) => reject(new HttpError(0, err.errMsg)),
    });
  });
}

/** 业务请求入口：自动带 token，401 时登录并重试一次 */
export async function request<T>(opts: RequestOptions): Promise<T> {
  if (opts.auth === false) return rawRequest<T>(opts);

  let token: string = wx.getStorageSync(TOKEN_KEY) || (await login());
  try {
    return await rawRequest<T>(opts, token);
  } catch (err) {
    // 只重试一次：如果重新登录后仍然 401，说明是权限问题而不是登录态过期，避免死循环
    if (err instanceof HttpError && err.status === 401) {
      wx.removeStorageSync(TOKEN_KEY);
      token = await login();
      return rawRequest<T>(opts, token);
    }
    throw err;
  }
}
```

**易错点**

- 多个请求同时 401，各自调用 `wx.login`
- 登录接口本身也带上了 401 重试逻辑，导致递归
- 无限重试
- `loginPromise` 失败后没有清空，之后永远失败

---

### H3. 实现 setData 的差量计算

- **难度**: 高级
- **标签**: 小程序, setData, 性能
- **面试官看什么**: 路径生成、数组和对象的处理、何时退化为整体替换

**题干**

给定旧数据和新数据，生成最小的 setData 路径对象，例如 `{ 'list[2].name': 'x', 'user.age': 20 }`。

**参考实现**

```ts
type Json = null | boolean | number | string | Json[] | { [k: string]: Json };

/**
 * 计算从 prev 到 next 的最小 setData 补丁。
 * 数组变短或类型变化时整体替换：setData 的路径语法无法删除数组元素或对象的键。
 */
export function diffForSetData(
  prev: Record<string, Json>,
  next: Record<string, Json>,
): Record<string, Json> {
  const patch: Record<string, Json> = {};
  for (const key of Object.keys(next)) {
    diffValue(prev[key], next[key], key, patch);
  }
  return patch;
}

function diffValue(a: Json | undefined, b: Json, path: string, patch: Record<string, Json>) {
  if (a === b) return;

  if (Array.isArray(a) && Array.isArray(b)) {
    if (b.length < a.length) {
      patch[path] = b;
      return;
    }
    for (let i = 0; i < b.length; i++) {
      diffValue(a[i], b[i], `${path}[${i}]`, patch);
    }
    return;
  }

  if (isPlainObject(a) && isPlainObject(b)) {
    const removed = Object.keys(a).some((k) => !(k in b));
    if (removed) {
      patch[path] = b;
      return;
    }
    for (const k of Object.keys(b)) {
      diffValue(a[k], b[k], `${path}.${k}`, patch);
    }
    return;
  }

  // 基础类型变化，或者类型不同（对象变数组、null 变对象）
  patch[path] = b;
}

function isPlainObject(v: unknown): v is Record<string, Json> {
  return typeof v === 'object' && v !== null && !Array.isArray(v);
}
```

**示例**

```ts
diffForSetData(
  { list: [{ id: 1, name: 'a' }, { id: 2, name: 'b' }], user: { age: 18 } },
  { list: [{ id: 1, name: 'a' }, { id: 2, name: 'c' }, { id: 3, name: 'd' }], user: { age: 20 } },
);
// { 'list[1].name': 'c', 'list[2]': { id: 3, name: 'd' }, 'user.age': 20 }
```

**易错点**

- 数组变短时只更新前面的项，渲染层的数组长度不变
- 对象删除键时无法通过路径删除
- 键名中包含 `.` 或 `[` 时路径会被错误解析（实际项目中需要校验或整体替换）
- 路径过多时，补丁本身可能比整体数据还大 → 可以设置阈值，超过时整体替换

---

### H4. 合并同一轮中的多次 setData

- **难度**: 中级
- **标签**: 小程序, setData
- **面试官看什么**: 微任务合并、回调的处理、页面卸载

**参考实现**

```ts
/**
 * 为页面或组件实例创建合并后的 setData：同一轮同步代码中的多次调用合并为一次跨线程通信。
 * 注意：调用后立即读取 this.data 拿不到新值，需要读取的场景应使用原始 setData。
 */
export function createBatchedSetData(ctx: WechatMiniprogram.Page.TrivialInstance) {
  let buffer: Record<string, unknown> | null = null;
  let callbacks: (() => void)[] = [];
  let destroyed = false;

  const flush = () => {
    if (!buffer || destroyed) return;
    const data = buffer;
    const cbs = callbacks;
    buffer = null;
    callbacks = [];
    ctx.setData(data, () => cbs.forEach((cb) => cb()));
  };

  return {
    setData(data: Record<string, unknown>, cb?: () => void) {
      if (destroyed) return;
      if (!buffer) {
        buffer = {};
        Promise.resolve().then(flush);
      }
      for (const [key, value] of Object.entries(data)) {
        // 先删除同名路径和它的子路径再插入，保证合并后的应用顺序与原来多次调用的效果一致。
        // 例如依次设置 a、a.b、a 时，若直接覆盖，a 会保留在最前面，随后 a.b 又改写了最后一次的 a
        for (const existing of Object.keys(buffer)) {
          if (existing === key || existing.startsWith(key + '.') || existing.startsWith(key + '[')) {
            delete buffer[existing];
          }
        }
        buffer[key] = value;
      }
      if (cb) callbacks.push(cb);
    },
    /** 在 onUnload / detached 中调用，页面卸载后不再更新 */
    destroy() {
      destroyed = true;
      buffer = null;
      callbacks = [];
    },
  };
}
```

**易错点**

- 合并后 `this.data` 的读取时机改变（需要说明）
- 页面卸载后仍然 setData
- 回调丢失
- 直接 `Object.assign` 合并：同名键保留原来的插入位置，父路径和子路径交替设置时，最终结果与逐次调用不一致

---

### H5. 实现虚拟列表的可视范围计算（支持不定高度）

- **难度**: 高级
- **标签**: 虚拟列表, 算法
- **面试官看什么**: 前缀和、二分查找、预估高度与真实高度的修正

**参考实现**

```ts
/**
 * 不定高度虚拟列表的位置计算器。
 * 未测量的项使用预估高度，测量后修正；通过前缀和 + 二分查找定位可视范围。
 */
export class VirtualListLayout {
  private heights: number[];
  private offsets: number[] = []; // offsets[i] = 第 i 项的顶部位置
  private dirtyFrom = 0; // 从这一项开始的 offsets 需要重新计算

  constructor(count: number, private estimatedHeight: number) {
    this.heights = new Array(count).fill(estimatedHeight);
  }

  /** 数据长度变化时调用（例如分页加载） */
  setCount(count: number) {
    const old = this.heights.length;
    if (count > old) {
      this.heights.push(...new Array(count - old).fill(this.estimatedHeight));
    } else {
      this.heights.length = count;
    }
    this.dirtyFrom = Math.min(this.dirtyFrom, old);
  }

  /** 列表项渲染后上报真实高度 */
  setHeight(index: number, height: number) {
    if (this.heights[index] === height) return;
    this.heights[index] = height;
    this.dirtyFrom = Math.min(this.dirtyFrom, index);
  }

  /** 列表总高度，用于撑开滚动容器 */
  get totalHeight(): number {
    this.ensureOffsets();
    const n = this.heights.length;
    return n ? this.offsets[n - 1] + this.heights[n - 1] : 0;
  }

  getOffset(index: number): number {
    this.ensureOffsets();
    return this.offsets[index] ?? 0;
  }

  /** 计算需要渲染的范围，buffer 为上下额外渲染的项数，减少快速滚动时的白屏 */
  getRange(scrollTop: number, viewportHeight: number, buffer = 5): { start: number; end: number } {
    const n = this.heights.length;
    if (!n) return { start: 0, end: -1 };
    this.ensureOffsets();
    const first = this.findIndex(scrollTop);
    const last = this.findIndex(scrollTop + viewportHeight);
    return { start: Math.max(0, first - buffer), end: Math.min(n - 1, last + buffer) };
  }

  /** 二分查找：最后一个 offset <= y 的项 */
  private findIndex(y: number): number {
    let lo = 0;
    let hi = this.heights.length - 1;
    while (lo < hi) {
      const mid = (lo + hi + 1) >> 1;
      if (this.offsets[mid] <= y) lo = mid;
      else hi = mid - 1;
    }
    return lo;
  }

  /** 惰性更新：只从第一个变化的位置开始重算，避免每次测量都全量计算 */
  private ensureOffsets() {
    const n = this.heights.length;
    if (this.dirtyFrom >= n && this.offsets.length === n) return;
    this.offsets.length = n;
    for (let i = this.dirtyFrom; i < n; i++) {
      this.offsets[i] = i === 0 ? 0 : this.offsets[i - 1] + this.heights[i - 1];
    }
    this.dirtyFrom = n;
  }
}
```

**说明**：可视区域上方的项高度修正后，内容会跳动 → 需要同时调整 `scrollTop`（滚动锚定）。

**易错点**

- 每次滚动都全量计算前缀和（O(n)）
- 线性查找可视范围
- 没有缓冲区，快速滚动出现白屏
- 上方项的高度变化导致内容跳动

---

### H6. RN 热更新：检查更新与启动保护

- **难度**: 高级
- **标签**: RN, 热更新
- **面试官看什么**: 版本匹配、完整性校验、崩溃回滚的状态机

**题干**

实现 JS 侧的更新逻辑（原生能力通过接口注入）：检查更新、下载并校验、下次启动生效；新 bundle 启动后如果在 N 秒内没有确认成功，下次启动回滚。

**参考实现**

```ts
/** 原生侧提供的能力（通过原生模块实现） */
export interface NativeUpdater {
  runtimeVersion: string; // 原生版本，bundle 必须与之匹配
  currentBundleId: string | null;
  download(url: string): Promise<string>; // 返回本地文件路径
  sha256(path: string): Promise<string>;
  verifySignature(hash: string, signature: string): Promise<boolean>;
  /** 设置下次启动加载的 bundle */
  setPendingBundle(bundleId: string, path: string): Promise<void>;
  /** 回退到上一个确认可用的 bundle（或内置 bundle） */
  rollback(): Promise<void>;
  /** 持久化的 KV，用于记录启动状态 */
  storage: { get(k: string): Promise<string | null>; set(k: string, v: string): Promise<void> };
}

/** 服务端返回的更新信息 */
export interface UpdateManifest {
  bundleId: string;
  runtimeVersion: string;
  url: string;
  sha256: string;
  signature: string;
}

const LAUNCH_KEY = 'ota_launch_state';

/**
 * 启动保护：在 App 入口最早处调用。
 * 原理：加载新 bundle 前标记为 "trying"；首屏成功后标记为 "ok"。
 * 如果启动时发现上一次还是 "trying"，说明上次启动后崩溃了，执行回滚。
 */
export async function guardLaunch(native: NativeUpdater): Promise<'ok' | 'rolled-back'> {
  const state = await native.storage.get(LAUNCH_KEY);
  if (state === `trying:${native.currentBundleId}`) {
    await native.rollback();
    await native.storage.set(LAUNCH_KEY, 'rolled-back');
    return 'rolled-back';
  }
  await native.storage.set(LAUNCH_KEY, `trying:${native.currentBundleId}`);
  return 'ok';
}

/** 首屏渲染成功且稳定运行一段时间后调用，确认当前 bundle 可用 */
export function confirmLaunch(native: NativeUpdater, delayMs = 5_000) {
  setTimeout(() => {
    void native.storage.set(LAUNCH_KEY, `ok:${native.currentBundleId}`);
  }, delayMs);
}

/** 检查并下载更新，下次启动生效 */
export async function checkForUpdate(native: NativeUpdater, endpoint: string): Promise<boolean> {
  const res = await fetch(
    `${endpoint}?runtime=${encodeURIComponent(native.runtimeVersion)}&current=${native.currentBundleId ?? ''}`,
  );
  if (res.status === 204) return false;
  const manifest: UpdateManifest = await res.json();

  // 服务端出错下发了不匹配的 bundle 时，客户端仍要拒绝：不匹配的 JS 调用不存在的原生接口会崩溃
  if (manifest.runtimeVersion !== native.runtimeVersion) return false;
  if (manifest.bundleId === native.currentBundleId) return false;

  const path = await native.download(manifest.url);
  const hash = await native.sha256(path);
  if (hash !== manifest.sha256) throw new Error('bundle 哈希不一致');
  if (!(await native.verifySignature(hash, manifest.signature))) {
    throw new Error('bundle 签名校验失败');
  }
  await native.setPendingBundle(manifest.bundleId, path);
  return true;
}
```

**使用**

```tsx
// index.js 最早执行
await guardLaunch(NativeUpdater);

function App() {
  useEffect(() => {
    confirmLaunch(NativeUpdater);
    checkForUpdate(NativeUpdater, 'https://ota.example.com/check').catch(reportError);
  }, []);
  // ...
}
```

**易错点**

- 不校验原生版本，导致新 JS 调用旧原生中不存在的接口
- 只校验哈希不校验签名（哈希也来自服务端）
- 启动保护的「确认」太早（首屏渲染前），崩溃发生在之后时无法回滚
- 回滚后又下载同一个有问题的版本 → 服务端需要停止下发，客户端可以记录黑名单

---

### H7. RN：滑动删除的列表项（Gesture Handler + Reanimated）

- **难度**: 中级
- **标签**: RN, 手势, 动画
- **面试官看什么**: 动画在 UI 线程、方向判断避免与列表滚动冲突、回到 JS 线程的时机

**参考实现**

```tsx
import { StyleSheet, Text, View } from 'react-native';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  runOnJS,
  useAnimatedStyle,
  useSharedValue,
  withTiming,
} from 'react-native-reanimated';

const DELETE_THRESHOLD = -120;

/** 可以左滑删除的列表项 */
export function SwipeableRow({ title, onDelete }: { title: string; onDelete: () => void }) {
  const translateX = useSharedValue(0);
  const height = useSharedValue(64);

  const pan = Gesture.Pan()
    // 横向移动超过 10 才激活，竖向移动超过 10 就放弃，把手势让给列表滚动
    .activeOffsetX([-10, 10])
    .failOffsetY([-10, 10])
    .onChange((e) => {
      translateX.value = Math.min(0, translateX.value + e.changeX);
    })
    .onEnd(() => {
      if (translateX.value < DELETE_THRESHOLD) {
        translateX.value = withTiming(-500, { duration: 200 });
        height.value = withTiming(0, { duration: 200 }, (finished) => {
          // 动画回调在 UI 线程执行，修改 React 状态必须切回 JS 线程
          if (finished) runOnJS(onDelete)();
        });
      } else {
        translateX.value = withTiming(0);
      }
    });

  const rowStyle = useAnimatedStyle(() => ({ transform: [{ translateX: translateX.value }] }));
  const containerStyle = useAnimatedStyle(() => ({ height: height.value }));

  return (
    <Animated.View style={[styles.container, containerStyle]}>
      <View style={styles.deleteBg}>
        <Text style={styles.deleteText}>删除</Text>
      </View>
      <GestureDetector gesture={pan}>
        <Animated.View style={[styles.row, rowStyle]}>
          <Text>{title}</Text>
        </Animated.View>
      </GestureDetector>
    </Animated.View>
  );
}

const styles = StyleSheet.create({
  container: { overflow: 'hidden' },
  deleteBg: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: '#e53935',
    alignItems: 'flex-end',
    justifyContent: 'center',
    paddingRight: 24,
  },
  deleteText: { color: '#fff', fontWeight: '600' },
  row: { flex: 1, justifyContent: 'center', paddingHorizontal: 16, backgroundColor: '#fff' },
});
```

> Reanimated 4 中 `runOnJS` 逐步由 `scheduleOnRN`（react-native-worklets）替代，以所用版本的文档为准。

**易错点**

- 用 `useState` 保存位移，每一帧都触发 React 渲染
- 没有设置激活方向，与列表的竖向滚动冲突
- 在 worklet 中直接调用 JS 函数
- 删除时没有高度收起动画，列表突然跳动

---

### H8. 实现迷你版 Taro 运行时：模拟 DOM 生成 setData 路径

- **难度**: 高级
- **标签**: Taro, 原理
- **面试官看什么**: 节点路径、更新收集与合并、序列化结构

**题干**

实现 `MiniElement`，支持 `appendChild`、`removeChild`、`setAttribute`、文本节点修改；所有修改在同一轮中合并，生成 setData 的路径数据（结构为 `root.cn[0].cn[1].v` 这类路径）。

**参考实现**

```ts
/** 序列化给渲染层的节点结构：nn 为节点名，cn 为子节点，v 为文本 */
interface SerializedNode {
  nn: string;
  sid: string;
  v?: string;
  [attr: string]: unknown;
}

let sidSeq = 0;

/** 模拟 DOM 的节点基类 */
export class MiniNode {
  sid = `_n${++sidSeq}`;
  parent: MiniElement | null = null;

  constructor(public nodeName: string, protected root: MiniRoot | null = null) {}

  /** 当前节点在 setData 数据中的路径，例如 root.cn[0].cn[2] */
  get path(): string {
    if (!this.parent) return 'root';
    const index = this.parent.children.indexOf(this);
    return `${this.parent.path}.cn[${index}]`;
  }

  get ownerRoot(): MiniRoot | null {
    return this.parent ? this.parent.ownerRoot : this.root;
  }

  serialize(): SerializedNode {
    return { nn: this.nodeName, sid: this.sid };
  }
}

/** 文本节点 */
export class MiniText extends MiniNode {
  constructor(private _value: string) {
    super('#text');
  }

  get nodeValue() {
    return this._value;
  }

  set nodeValue(v: string) {
    this._value = v;
    // 文本变化只更新 v 字段，不需要重新传输整个节点
    this.ownerRoot?.enqueue(`${this.path}.v`, v);
  }

  serialize(): SerializedNode {
    return { nn: '#text', sid: this.sid, v: this._value };
  }
}

/** 元素节点 */
export class MiniElement extends MiniNode {
  children: MiniNode[] = [];
  attrs: Record<string, unknown> = {};

  setAttribute(name: string, value: unknown) {
    this.attrs[name] = value;
    this.ownerRoot?.enqueue(`${this.path}.${name}`, value);
  }

  appendChild(child: MiniNode) {
    child.parent?.removeChild(child);
    child.parent = this;
    this.children.push(child);
    // 新增子节点只追加该位置，不重传整个 cn 数组
    this.ownerRoot?.enqueue(child.path, child.serialize());
  }

  removeChild(child: MiniNode) {
    const i = this.children.indexOf(child);
    if (i < 0) return;
    this.children.splice(i, 1);
    child.parent = null;
    // 删除会让后续兄弟的下标变化，路径语法无法表达删除，只能整体替换 cn
    this.ownerRoot?.enqueue(`${this.path}.cn`, this.children.map((c) => c.serialize()));
  }

  serialize(): SerializedNode {
    return {
      nn: this.nodeName,
      sid: this.sid,
      ...this.attrs,
      cn: this.children.map((c) => c.serialize()),
    };
  }
}

/** 页面根节点：收集同一轮的修改，合并后调用一次 setData */
export class MiniRoot extends MiniElement {
  private updates = new Map<string, unknown>();
  private scheduled = false;

  constructor(private setData: (data: Record<string, unknown>) => void) {
    super('root');
  }

  get ownerRoot(): MiniRoot {
    return this;
  }

  enqueue(path: string, value: unknown) {
    // 已经有祖先路径的整体更新时，子路径的修改已包含在内，无需重复
    for (const existing of this.updates.keys()) {
      if (path.startsWith(existing + '.') || path.startsWith(existing + '[')) {
        this.updates.set(existing, this.resolve(existing));
        return;
      }
    }
    // 新路径是已有路径的祖先时，删除被覆盖的子路径
    for (const existing of [...this.updates.keys()]) {
      if (existing.startsWith(path + '.') || existing.startsWith(path + '[')) {
        this.updates.delete(existing);
      }
    }
    this.updates.set(path, value);
    if (!this.scheduled) {
      this.scheduled = true;
      Promise.resolve().then(() => this.flush());
    }
  }

  private flush() {
    this.scheduled = false;
    const data = Object.fromEntries(this.updates);
    this.updates.clear();
    this.setData(data);
  }

  /** 根据路径重新读取最新的序列化值，用于祖先路径的合并 */
  private resolve(path: string): unknown {
    const segments = path.replace(/^root\.?/, '').match(/cn\[\d+\]|[^.[\]]+/g) ?? [];
    let node: MiniNode = this;
    for (const seg of segments) {
      const m = /^cn\[(\d+)\]$/.exec(seg);
      if (m && node instanceof MiniElement) node = node.children[Number(m[1])];
      else if (seg === 'cn' && node instanceof MiniElement) {
        return node.children.map((c) => c.serialize());
      } else {
        return (node.serialize() as Record<string, unknown>)[seg];
      }
    }
    return node.serialize();
  }
}
```

**说明**：真实的 Taro 用 `sid` 结合 `CustomWrapper` 缩短路径，并处理事件、样式、模板层级等，这里只演示「模拟 DOM → 合并后的路径更新」这一核心思路。

**易错点**

- 每次修改都立即 setData
- 新增节点时重传整个子节点数组
- 祖先路径和子路径同时存在导致覆盖顺序错误
- 删除节点后后续兄弟节点的路径改变

---

### H9. 设计多端适配层（存储、请求、路由）

- **难度**: 中级
- **标签**: 多端架构
- **面试官看什么**: 依赖倒置、接口设计、平台实现的选择方式

**参考实现**

```ts
// packages/core/platform.ts
/** 键值存储的抽象 */
export interface IStorage {
  get<T>(key: string): Promise<T | null>;
  set<T>(key: string, value: T): Promise<void>;
  remove(key: string): Promise<void>;
}

/** 路由的抽象：业务代码只使用逻辑路由名，由各端映射到实际页面 */
export interface IRouter {
  push(route: string, params?: Record<string, string>): void;
  back(): void;
}

/** 平台能力的集合，由各端在启动时注入 */
export interface Platform {
  storage: IStorage;
  router: IRouter;
  name: 'weapp' | 'h5' | 'rn';
}

let current: Platform | null = null;

/** 在各端的入口文件中调用一次 */
export function setupPlatform(p: Platform) {
  current = p;
}

/** 业务代码获取平台能力 */
export function platform(): Platform {
  if (!current) throw new Error('platform 尚未初始化，请在入口调用 setupPlatform');
  return current;
}
```

```ts
// packages/adapters/weapp.ts
import type { Platform } from '@app/core/platform';

const ROUTES: Record<string, string> = {
  'order.detail': '/packageOrder/detail/index',
  home: '/pages/index/index',
};

/** 微信小程序的平台实现 */
export const weappPlatform: Platform = {
  name: 'weapp',
  storage: {
    async get(key) {
      const v = wx.getStorageSync(key);
      return v === '' ? null : v; // 小程序中不存在的键返回空字符串
    },
    async set(key, value) {
      wx.setStorageSync(key, value);
    },
    async remove(key) {
      wx.removeStorageSync(key);
    },
  },
  router: {
    push(route, params = {}) {
      const query = new URLSearchParams(params).toString();
      wx.navigateTo({ url: `${ROUTES[route]}${query ? `?${query}` : ''}` });
    },
    back() {
      wx.navigateBack();
    },
  },
};
```

```ts
// packages/adapters/h5.ts
import type { Platform } from '@app/core/platform';

/** H5 的平台实现 */
export const h5Platform: Platform = {
  name: 'h5',
  storage: {
    async get(key) {
      const raw = localStorage.getItem(key);
      return raw === null ? null : JSON.parse(raw);
    },
    async set(key, value) {
      localStorage.setItem(key, JSON.stringify(value));
    },
    async remove(key) {
      localStorage.removeItem(key);
    },
  },
  router: {
    push(route, params = {}) {
      const path = route.replace('.', '/');
      history.pushState(null, '', `/${path}?${new URLSearchParams(params)}`);
      dispatchEvent(new PopStateEvent('popstate'));
    },
    back() {
      history.back();
    },
  },
};
```

```ts
// 业务代码：与平台无关
export async function openLastOrder() {
  const id = await platform().storage.get<string>('lastOrderId');
  if (id) platform().router.push('order.detail', { id });
}
```

**易错点**

- 业务代码中散落 `if (isWeapp)` 判断
- 接口设计成某个平台的形状（例如同步接口），其他平台无法实现
- 小程序存储不存在时返回空字符串而不是 null
- 路由直接使用平台路径，业务代码与页面结构耦合

---

### H10. 小程序安全路由：处理页面栈上限和 tabBar

- **难度**: 初级
- **标签**: 小程序, 路由
- **面试官看什么**: 页面栈上限、tabBar 页面、防重复点击

**参考实现**

```ts
const MAX_STACK = 10;
const TAB_PAGES = new Set(['pages/index/index', 'pages/mine/index']);

let navigating = false;

/**
 * 统一的页面跳转：
 * - tabBar 页面使用 switchTab（navigateTo 无法打开 tabBar 页面）
 * - 页面栈接近上限时改用 redirectTo
 * - 跳转过程中忽略重复调用，防止快速连点打开多个相同页面
 */
export function navigate(url: string): Promise<void> {
  if (navigating) return Promise.resolve();
  navigating = true;

  const [path] = url.replace(/^\//, '').split('?');
  const done = () => {
    navigating = false;
  };

  return new Promise<void>((resolve, reject) => {
    const options = { url: url.startsWith('/') ? url : `/${url}`, success: () => resolve(), fail: reject, complete: done };

    if (TAB_PAGES.has(path)) {
      // switchTab 不支持参数，参数需要通过全局状态传递
      wx.switchTab({ ...options, url: `/${path}` });
    } else if (getCurrentPages().length >= MAX_STACK - 1) {
      wx.redirectTo(options);
    } else {
      wx.navigateTo(options);
    }
  });
}
```

**易错点**

- 用 navigateTo 打开 tabBar 页面
- 栈满时跳转失败但没有处理
- 快速连点打开多个页面
- 失败时没有重置标记，之后再也无法跳转（这里在 `complete` 中重置）

---

### H11. 离线操作队列（outbox）

- **难度**: 高级
- **标签**: 离线, 同步
- **面试官看什么**: 顺序、幂等、重试退避、持久化

**参考实现**

```ts
/** 一个待同步的操作 */
export interface OutboxOp {
  /** 客户端生成的唯一 ID，服务端据此去重，保证重试不会重复提交 */
  id: string;
  type: string;
  payload: unknown;
  createdAt: number;
  attempts: number;
}

/** 持久化接口（SQLite / MMKV / 小程序 storage 的实现） */
export interface OutboxStore {
  load(): Promise<OutboxOp[]>;
  save(ops: OutboxOp[]): Promise<void>;
}

/**
 * 离线操作队列：按顺序同步，失败时退避重试。
 * 前一个操作未成功时不发送后续操作，因为后续操作可能依赖它（先创建再修改）。
 */
export class Outbox {
  private ops: OutboxOp[] = [];
  private syncing = false;

  constructor(
    private store: OutboxStore,
    private send: (op: OutboxOp) => Promise<void>,
    private isPermanentError: (err: unknown) => boolean = () => false,
  ) {}

  async init() {
    this.ops = await this.store.load();
  }

  async enqueue(type: string, payload: unknown) {
    this.ops.push({ id: crypto.randomUUID(), type, payload, createdAt: Date.now(), attempts: 0 });
    // 先持久化再返回，确保 App 被杀掉后操作不丢失
    await this.store.save(this.ops);
    void this.sync();
  }

  get size() {
    return this.ops.length;
  }

  /** 在网络恢复、App 回到前台时调用 */
  async sync(): Promise<void> {
    if (this.syncing) return;
    this.syncing = true;
    try {
      while (this.ops.length) {
        const op = this.ops[0];
        try {
          await this.send(op);
          this.ops.shift();
          await this.store.save(this.ops);
        } catch (err) {
          if (this.isPermanentError(err)) {
            // 永久错误（例如 400 校验失败）重试也不会成功，移出队列并交给业务处理，否则会阻塞后续所有操作
            this.ops.shift();
            await this.store.save(this.ops);
            this.onDeadLetter?.(op, err);
            continue;
          }
          op.attempts++;
          await this.store.save(this.ops);
          await sleep(Math.min(30_000, 1000 * 2 ** op.attempts));
          if (op.attempts >= 5) return; // 暂停本轮，等待下一次触发（网络恢复 / 前台）
        }
      }
    } finally {
      this.syncing = false;
    }
  }

  /** 无法同步的操作回调，例如提示用户或记录 */
  onDeadLetter?: (op: OutboxOp, err: unknown) => void;
}

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));
```

**易错点**

- 没有客户端 ID，网络超时重试导致重复提交
- 并发发送，顺序错乱
- 永久错误一直重试，阻塞整个队列
- 只保存在内存中，App 被杀掉后丢失

---

### H12. 小程序曝光埋点

- **难度**: 中级
- **标签**: 小程序, 埋点
- **面试官看什么**: IntersectionObserver、去重、批量上报、页面卸载时的处理

**参考实现**

```ts
/** 曝光事件 */
interface ExposureEvent {
  id: string;
  page: string;
  ts: number;
}

/**
 * 页面级曝光追踪：元素进入视口超过一半时记录一次曝光，批量上报。
 * 列表项需要带有 class="expose-item" 和 data-id。
 */
export class ExposureTracker {
  private observer: WechatMiniprogram.IntersectionObserver | null = null;
  private seen = new Set<string>();
  private queue: ExposureEvent[] = [];
  private timer: number | null = null;

  constructor(
    private page: WechatMiniprogram.Page.TrivialInstance,
    private report: (events: ExposureEvent[]) => void,
    private batchSize = 20,
    private flushInterval = 5_000,
  ) {}

  /** 列表数据更新后调用；重新创建 observer，因为新加入的节点不会被旧的 observer 观察 */
  observe() {
    this.observer?.disconnect();
    this.observer = this.page.createIntersectionObserver({ thresholds: [0.5], observeAll: true });
    this.observer.relativeToViewport().observe('.expose-item', (res) => {
      if (res.intersectionRatio < 0.5) return;
      const id = (res.dataset as { id?: string }).id;
      if (!id || this.seen.has(id)) return; // 同一个元素在本页面只曝光一次
      this.seen.add(id);
      this.queue.push({ id, page: this.page.route, ts: Date.now() });
      this.scheduleFlush();
    });
  }

  private scheduleFlush() {
    if (this.queue.length >= this.batchSize) {
      this.flush();
      return;
    }
    if (this.timer === null) {
      this.timer = setTimeout(() => this.flush(), this.flushInterval) as unknown as number;
    }
  }

  flush() {
    if (this.timer !== null) {
      clearTimeout(this.timer);
      this.timer = null;
    }
    if (!this.queue.length) return;
    const events = this.queue;
    this.queue = [];
    this.report(events);
  }

  /** 在 onHide 和 onUnload 中调用：页面离开时把剩余的事件上报，避免丢失 */
  destroy() {
    this.flush();
    this.observer?.disconnect();
    this.observer = null;
  }
}
```

**使用**

```ts
Page({
  onLoad() {
    this.tracker = new ExposureTracker(this, (events) => wx.request({ url: LOG_URL, method: 'POST', data: { events } }));
  },
  onListUpdated() {
    this.setData({ list: newList }, () => this.tracker.observe());
  },
  onHide() {
    this.tracker.flush();
  },
  onUnload() {
    this.tracker.destroy();
  },
});
```

**易错点**

- 新渲染的列表项没有被观察（需要在 setData 回调中重新观察）
- 同一元素滚动进出多次重复曝光
- 每次曝光立即上报，请求过多
- 页面离开时队列中的事件丢失
