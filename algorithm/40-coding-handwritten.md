# 数据结构与算法手写题

> 目标：能在面试中完整、无 bug 地手写常见数据结构和经典算法，并说清复杂度和边界。代码使用 TypeScript。

## 使用方式

- **数据结构**：H1 LRU、H2 最小堆、H3 Trie 自动补全、H4 并查集
- **前端框架中的算法**：H5 LIS（Vue 3 diff）、H6 树结构工具
- **排序与搜索**：H7 三路快排与堆排序、H10 KMP、H11 合并 K 个有序链表
- **图**：H8 拓扑排序与环检测、H9 Dijkstra
- **数学与字符串**：H12 大数相乘

---

### H1. LRU 缓存（哈希表 + 双向链表）

- **难度**: 中级
- **标签**: LRU, 链表, 设计
- **面试官看什么**: 不依赖 Map 顺序时的 O(1) 实现、哨兵节点、指针操作的正确性

**题干**

实现 `LRUCache<K, V>`，`get` 和 `put` 都是 O(1)；超出容量时淘汰最久未使用的项。不能依赖 Map 的插入顺序。

**参考实现**

```ts
class DNode<K, V> {
  prev: DNode<K, V> | null = null;
  next: DNode<K, V> | null = null;
  constructor(public key: K, public value: V) {}
}

/**
 * LRU 缓存：哈希表负责 O(1) 查找，双向链表维护使用顺序（头部最新，尾部最旧）。
 */
export class LRUCache<K, V> {
  private map = new Map<K, DNode<K, V>>();
  // 头尾哨兵节点：插入和删除时不需要判断边界是否为空
  private head = new DNode<K, V>(null as K, null as V);
  private tail = new DNode<K, V>(null as K, null as V);

  constructor(private capacity: number) {
    if (capacity <= 0) throw new RangeError('capacity 必须大于 0');
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  get(key: K): V | undefined {
    const node = this.map.get(key);
    if (!node) return undefined;
    this.moveToFront(node);
    return node.value;
  }

  put(key: K, value: V): void {
    const existing = this.map.get(key);
    if (existing) {
      existing.value = value;
      this.moveToFront(existing);
      return;
    }
    const node = new DNode(key, value);
    this.map.set(key, node);
    this.insertAfterHead(node);
    if (this.map.size > this.capacity) {
      const lru = this.tail.prev!;
      this.unlink(lru);
      // 节点中保存 key 的原因：淘汰时需要同时从哈希表中删除
      this.map.delete(lru.key);
    }
  }

  get size() {
    return this.map.size;
  }

  private moveToFront(node: DNode<K, V>) {
    this.unlink(node);
    this.insertAfterHead(node);
  }

  private unlink(node: DNode<K, V>) {
    node.prev!.next = node.next;
    node.next!.prev = node.prev;
    node.prev = node.next = null;
  }

  private insertAfterHead(node: DNode<K, V>) {
    node.prev = this.head;
    node.next = this.head.next;
    this.head.next!.prev = node;
    this.head.next = node;
  }
}
```

**易错点**

- 节点中没有保存 key，淘汰时无法从哈希表删除
- `put` 已存在的 key 时没有移动到头部
- 指针操作顺序错误（先修改 `head.next` 再读取它）
- 没有哨兵节点，边界判断复杂易错

---

### H2. 通用最小堆（优先队列）

- **难度**: 中级
- **标签**: 堆
- **面试官看什么**: 上浮和下沉的正确性、比较函数、边界

**参考实现**

```ts
/**
 * 基于数组的二叉堆。compare(a, b) < 0 表示 a 优先级更高（排在堆顶）。
 */
export class Heap<T> {
  private data: T[] = [];

  constructor(private compare: (a: T, b: T) => number) {}

  get size() {
    return this.data.length;
  }

  peek(): T | undefined {
    return this.data[0];
  }

  push(x: T): void {
    this.data.push(x);
    this.siftUp(this.data.length - 1);
  }

  pop(): T | undefined {
    const n = this.data.length;
    if (n === 0) return undefined;
    const top = this.data[0];
    const last = this.data.pop()!;
    // 只有一个元素时，pop 后数组已为空，不能再把 last 放回去
    if (n > 1) {
      this.data[0] = last;
      this.siftDown(0);
    }
    return top;
  }

  private siftUp(i: number) {
    const { data, compare } = this;
    while (i > 0) {
      const parent = (i - 1) >> 1;
      if (compare(data[i], data[parent]) >= 0) break;
      [data[i], data[parent]] = [data[parent], data[i]];
      i = parent;
    }
  }

  private siftDown(i: number) {
    const { data, compare } = this;
    const n = data.length;
    while (true) {
      const l = 2 * i + 1;
      const r = l + 1;
      let best = i;
      if (l < n && compare(data[l], data[best]) < 0) best = l;
      if (r < n && compare(data[r], data[best]) < 0) best = r;
      if (best === i) break;
      [data[i], data[best]] = [data[best], data[i]];
      i = best;
    }
  }
}

/** 示例：数据流中最大的 K 个数（大小为 K 的最小堆） */
export function topK(stream: Iterable<number>, k: number): number[] {
  const heap = new Heap<number>((a, b) => a - b);
  for (const x of stream) {
    if (heap.size < k) heap.push(x);
    else if (x > heap.peek()!) {
      heap.pop();
      heap.push(x);
    }
  }
  const res: number[] = [];
  while (heap.size) res.push(heap.pop()!);
  return res.reverse();
}
```

**易错点**

- 只有一个元素时 `pop` 把末尾元素又放回堆顶
- 下沉时只和一个子节点比较
- 比较函数方向搞反（最大堆和最小堆）

---

### H3. Trie 实现搜索框自动补全（按热度返回前 K 个）

- **难度**: 中级
- **标签**: Trie, 前端
- **面试官看什么**: 前缀定位、子树收集、排序与剪枝

**参考实现**

```ts
class TrieNode {
  children = new Map<string, TrieNode>();
  /** 以该节点结尾的完整词的热度；undefined 表示不是完整的词 */
  weight?: number;
}

/**
 * 自动补全：返回以 prefix 开头、热度最高的 k 个词。
 */
export class Autocomplete {
  private root = new TrieNode();

  add(word: string, weight = 1) {
    let node = this.root;
    for (const ch of word.toLowerCase()) {
      if (!node.children.has(ch)) node.children.set(ch, new TrieNode());
      node = node.children.get(ch)!;
    }
    node.weight = (node.weight ?? 0) + weight;
  }

  suggest(prefix: string, k = 10): string[] {
    let node: TrieNode | undefined = this.root;
    const p = prefix.toLowerCase();
    for (const ch of p) {
      node = node.children.get(ch);
      if (!node) return [];
    }
    const results: [string, number][] = [];
    // 迭代 DFS：词库很大时子树可能很深，避免递归栈溢出
    const stack: [TrieNode, string][] = [[node, p]];
    while (stack.length) {
      const [cur, word] = stack.pop()!;
      if (cur.weight !== undefined) results.push([word, cur.weight]);
      for (const [ch, child] of cur.children) stack.push([child, word + ch]);
    }
    results.sort((a, b) => b[1] - a[1] || a[0].localeCompare(b[0]));
    return results.slice(0, k).map(([w]) => w);
  }
}
```

**优化方向**：每个节点缓存子树中热度最高的 K 个词（插入时更新），查询变为 O(前缀长度)，代价是更新和内存开销。

**易错点**

- 用 `isEnd` 布尔值丢失了热度信息
- 前缀本身是完整的词时没有包含在结果中
- 大小写、全角半角等没有统一

---

### H4. 并查集（路径压缩 + 按秩合并）

- **难度**: 中级
- **标签**: 并查集
- **面试官看什么**: 两种优化、连通分量计数

**参考实现**

```ts
/** 并查集：维护 n 个元素的不相交集合 */
export class UnionFind {
  private parent: number[];
  private rank: number[];
  /** 当前连通分量的数量 */
  count: number;

  constructor(n: number) {
    this.parent = Array.from({ length: n }, (_, i) => i);
    this.rank = new Array(n).fill(0);
    this.count = n;
  }

  find(x: number): number {
    // 迭代式路径压缩：先找根，再把路径上的节点直接挂到根上，避免深递归
    let root = x;
    while (this.parent[root] !== root) root = this.parent[root];
    while (this.parent[x] !== root) {
      const next = this.parent[x];
      this.parent[x] = root;
      x = next;
    }
    return root;
  }

  /** 合并成功返回 true；已经在同一集合返回 false（可用于检测环） */
  union(a: number, b: number): boolean {
    let ra = this.find(a);
    let rb = this.find(b);
    if (ra === rb) return false;
    // 按秩合并：矮树挂到高树下，避免树变高
    if (this.rank[ra] < this.rank[rb]) [ra, rb] = [rb, ra];
    this.parent[rb] = ra;
    if (this.rank[ra] === this.rank[rb]) this.rank[ra]++;
    this.count--;
    return true;
  }

  connected(a: number, b: number): boolean {
    return this.find(a) === this.find(b);
  }
}

/** 示例：冗余连接——找出使无向图成环的那条边 */
export function findRedundantConnection(edges: [number, number][]): [number, number] | null {
  const uf = new UnionFind(edges.length + 1);
  for (const [a, b] of edges) {
    if (!uf.union(a, b)) return [a, b];
  }
  return null;
}
```

**易错点**

- 合并时直接连接 a 和 b，而不是它们的根
- 只做路径压缩不维护 count
- 递归 find 在极端情况下栈溢出

---

### H5. 最长递增子序列（返回下标，Vue 3 diff 用）

- **难度**: 高级
- **标签**: LIS, 二分, Vue
- **面试官看什么**: 前驱数组、回溯还原、跳过新节点

**题干**

输入 `source`（新节点在旧列表中的位置，`-1` 表示新节点），返回构成最长递增子序列的**下标数组**。这些下标对应的节点不需要移动。

**参考实现**

```ts
/**
 * 返回 source 中最长严格递增子序列的下标（升序）。值为 -1 的位置（新节点）不参与。
 * O(n log n)：tails 存下标，维护「每个长度的递增子序列的最小结尾」；prev 记录前驱用于还原。
 */
export function getSequence(source: number[]): number[] {
  const prev = new Array<number>(source.length).fill(-1);
  const tails: number[] = [];

  for (let i = 0; i < source.length; i++) {
    const v = source[i];
    if (v === -1) continue;

    // 二分查找第一个 source[tails[mid]] >= v 的位置
    let lo = 0, hi = tails.length;
    while (lo < hi) {
      const mid = (lo + hi) >> 1;
      if (source[tails[mid]] < v) lo = mid + 1;
      else hi = mid;
    }
    if (lo > 0) prev[i] = tails[lo - 1];
    tails[lo] = i;
  }

  // tails 中间的下标可能被后来的元素替换过，不一定构成合法序列；
  // 但每个元素的前驱是在它加入时记录的，从最后一个元素沿前驱回溯一定是合法的
  const result = new Array<number>(tails.length);
  let k = tails.length ? tails[tails.length - 1] : -1;
  for (let i = tails.length - 1; i >= 0; i--) {
    result[i] = k;
    k = prev[k];
  }
  return result;
}
```

**示例**

```ts
getSequence([4, 2, 3, 1, 5]); // [1, 2, 4] → 值 [2, 3, 5]
getSequence([2, -1, 0, 1]); //   [2, 3]    → 值 [0, 1]，下标 1 是新节点被跳过
```

**在 diff 中的用法**：从后往前遍历新列表的乱序部分，下标在 LIS 中的节点不动；否则移动到下一个节点之前；`-1` 的节点新建。

**易错点**

- 直接返回 tails（不是合法序列）
- 二分条件用 `<=`，变成非严格递增
- 没有跳过新节点

---

### H6. 树结构工具：扁平转树、查找路径、过滤

- **难度**: 中级
- **标签**: 树, 前端
- **面试官看什么**: O(n) 转换、异常数据处理、保留祖先的过滤

**参考实现**

```ts
/** 扁平数据项 */
export interface FlatItem {
  id: string;
  parentId: string | null;
  name: string;
}

/** 树节点 */
export interface TreeNode extends FlatItem {
  children: TreeNode[];
}

/**
 * 扁平数组转树，O(n)。
 * 父节点不存在的项（孤儿）作为根节点返回，避免数据异常时节点「消失」。
 */
export function toTree(items: FlatItem[]): TreeNode[] {
  const map = new Map<string, TreeNode>();
  for (const item of items) map.set(item.id, { ...item, children: [] });

  const roots: TreeNode[] = [];
  for (const node of map.values()) {
    const parent = node.parentId === null ? undefined : map.get(node.parentId);
    // 自己是自己的父节点属于异常数据，按根节点处理，避免形成环
    if (parent && parent !== node) parent.children.push(node);
    else roots.push(node);
  }
  return roots;
}

/** 查找从根到目标节点的路径（例如面包屑、展开父级菜单） */
export function findPath(roots: TreeNode[], targetId: string): TreeNode[] | null {
  const path: TreeNode[] = [];
  const dfs = (nodes: TreeNode[]): boolean => {
    for (const node of nodes) {
      path.push(node);
      if (node.id === targetId || dfs(node.children)) return true;
      path.pop();
    }
    return false;
  };
  return dfs(roots) ? [...path] : null;
}

/**
 * 按关键字过滤树：保留匹配的节点及其所有祖先（树形搜索的常见需求）。
 * 返回新树，不修改原数据。
 */
export function filterTree(roots: TreeNode[], predicate: (n: TreeNode) => boolean): TreeNode[] {
  const res: TreeNode[] = [];
  for (const node of roots) {
    const children = filterTree(node.children, predicate);
    if (predicate(node) || children.length) {
      res.push({ ...node, children });
    }
  }
  return res;
}
```

**易错点**

- 双重循环查找子节点（O(n²)）
- 父节点不存在的数据被丢弃
- 过滤时修改了原树
- `findPath` 中找到后没有立即返回，路径被后续回溯清空

---

### H7. 三路快排与堆排序

- **难度**: 中级
- **标签**: 排序
- **面试官看什么**: 大量重复元素的处理、原地堆排序

**参考实现**

```ts
/**
 * 三路快排：把数组分成 < pivot、= pivot、> pivot 三段，
 * 大量重复元素时避免退化为 O(n²)。
 */
export function quickSort3(arr: number[], lo = 0, hi = arr.length - 1): number[] {
  if (lo >= hi) return arr;
  const pivot = arr[lo + Math.floor(Math.random() * (hi - lo + 1))];
  // 不变式：[lo, lt) < pivot，[lt, i) = pivot，(gt, hi] > pivot
  let lt = lo, i = lo, gt = hi;
  while (i <= gt) {
    if (arr[i] < pivot) {
      [arr[lt], arr[i]] = [arr[i], arr[lt]];
      lt++;
      i++;
    } else if (arr[i] > pivot) {
      [arr[i], arr[gt]] = [arr[gt], arr[i]];
      gt--; // 换过来的元素还没检查，i 不动
    } else {
      i++;
    }
  }
  quickSort3(arr, lo, lt - 1);
  quickSort3(arr, gt + 1, hi);
  return arr;
}

/** 原地堆排序：先建最大堆，再依次把堆顶换到末尾。O(n log n)，O(1) 空间，不稳定 */
export function heapSort(arr: number[]): number[] {
  const n = arr.length;
  const siftDown = (i: number, size: number) => {
    while (true) {
      const l = 2 * i + 1, r = l + 1;
      let max = i;
      if (l < size && arr[l] > arr[max]) max = l;
      if (r < size && arr[r] > arr[max]) max = r;
      if (max === i) return;
      [arr[i], arr[max]] = [arr[max], arr[i]];
      i = max;
    }
  };
  // 从最后一个非叶子节点开始向前下沉，O(n) 建堆
  for (let i = (n >> 1) - 1; i >= 0; i--) siftDown(i, n);
  for (let end = n - 1; end > 0; end--) {
    [arr[0], arr[end]] = [arr[end], arr[0]];
    siftDown(0, end);
  }
  return arr;
}
```

**易错点**

- 三路快排中与 gt 交换后 i 也前进，漏检查换过来的元素
- 堆排序建堆从 0 开始向后（应该从最后一个非叶子节点向前）
- 下沉时 size 没有随已排序部分缩小

---

### H8. 拓扑排序：返回顺序或环的路径

- **难度**: 中级
- **标签**: 图, 拓扑排序
- **面试官看什么**: 三色标记、环路径的还原、清晰的报错信息

**题干**

给定模块的依赖关系（`deps[a] = [b, c]` 表示 a 依赖 b 和 c），返回构建顺序（依赖在前）；有循环依赖时返回环的路径，例如 `a → b → c → a`。

**参考实现**

```ts
/** 拓扑排序结果：成功返回顺序，失败返回循环依赖的路径 */
export type TopoResult = { ok: true; order: string[] } | { ok: false; cycle: string[] };

/**
 * DFS 三色标记：0 未访问，1 访问中（在当前递归路径上），2 已完成。
 * 遇到「访问中」的节点说明找到了环。
 */
export function topoSort(deps: Record<string, string[]>): TopoResult {
  const state = new Map<string, 0 | 1 | 2>();
  const order: string[] = [];
  const stack: string[] = []; // 当前递归路径，用于还原环

  const nodes = new Set([...Object.keys(deps), ...Object.values(deps).flat()]);

  const visit = (u: string): string[] | null => {
    state.set(u, 1);
    stack.push(u);
    for (const v of deps[u] ?? []) {
      const s = state.get(v) ?? 0;
      if (s === 1) {
        // 环是从 v 第一次出现在路径上的位置到当前节点，再回到 v
        return [...stack.slice(stack.indexOf(v)), v];
      }
      if (s === 0) {
        const cycle = visit(v);
        if (cycle) return cycle;
      }
    }
    stack.pop();
    state.set(u, 2);
    // 后序位置加入：所有依赖都已经在 order 中，所以依赖在前
    order.push(u);
    return null;
  };

  for (const n of nodes) {
    if ((state.get(n) ?? 0) === 0) {
      const cycle = visit(n);
      if (cycle) return { ok: false, cycle };
    }
  }
  return { ok: true, order };
}
```

**示例**

```ts
topoSort({ app: ['ui', 'utils'], ui: ['utils'], utils: [] });
// { ok: true, order: ['utils', 'ui', 'app'] }

topoSort({ a: ['b'], b: ['c'], c: ['a'] });
// { ok: false, cycle: ['a', 'b', 'c', 'a'] }
```

**易错点**

- 只用 visited 两种状态，无法区分「在当前路径上」和「已经完成」，导致误报环
- 没有包含只作为依赖出现的节点
- 依赖图很深时递归栈溢出（可以改为迭代或使用 Kahn 算法）

---

### H9. Dijkstra 最短路径（返回路径）

- **难度**: 高级
- **标签**: 图, 最短路径, 堆
- **面试官看什么**: 最小堆、延迟删除、路径还原

**参考实现**

```ts
import { Heap } from './heap'; // H2 中的 Heap

/** 带权有向图的邻接表：graph[u] = [[v, w], ...]，w 必须非负 */
export type WeightedGraph = Map<string, [string, number][]>;

/**
 * 求 start 到 end 的最短距离和路径。不可达时返回 null。
 */
export function dijkstra(
  graph: WeightedGraph,
  start: string,
  end: string,
): { distance: number; path: string[] } | null {
  const dist = new Map<string, number>([[start, 0]]);
  const prev = new Map<string, string>();
  const heap = new Heap<[string, number]>((a, b) => a[1] - b[1]);
  heap.push([start, 0]);

  while (heap.size) {
    const [u, d] = heap.pop()!;
    // 延迟删除：同一节点可能多次入堆，只处理距离最新的那一次
    if (d > (dist.get(u) ?? Infinity)) continue;
    if (u === end) break; // 出堆时距离已经确定，可以提前结束

    for (const [v, w] of graph.get(u) ?? []) {
      const nd = d + w;
      if (nd < (dist.get(v) ?? Infinity)) {
        dist.set(v, nd);
        prev.set(v, u);
        heap.push([v, nd]);
      }
    }
  }

  if (!dist.has(end)) return null;
  const path = [end];
  while (path[path.length - 1] !== start) path.push(prev.get(path[path.length - 1])!);
  return { distance: dist.get(end)!, path: path.reverse() };
}
```

**易错点**

- 没有跳过过期的堆元素，复杂度退化
- 在入堆时（而不是出堆时）判断到达终点，结果可能不是最短
- 存在负权边时仍使用 Dijkstra

---

### H10. KMP 字符串匹配

- **难度**: 高级
- **标签**: 字符串, KMP
- **面试官看什么**: next 数组的构建、匹配过程中模式指针的回退

**参考实现**

```ts
/** next[i] = p[0..i] 的最长相等真前缀和真后缀的长度 */
function buildNext(p: string): number[] {
  const next = new Array<number>(p.length).fill(0);
  for (let i = 1, len = 0; i < p.length; i++) {
    // 失配时回退到更短的相等前后缀继续尝试
    while (len > 0 && p[i] !== p[len]) len = next[len - 1];
    if (p[i] === p[len]) len++;
    next[i] = len;
  }
  return next;
}

/** 返回模式串 p 在文本 s 中所有出现的起始位置。O(n + m) */
export function kmpSearchAll(s: string, p: string): number[] {
  if (p.length === 0) return [];
  const next = buildNext(p);
  const res: number[] = [];
  for (let i = 0, j = 0; i < s.length; i++) {
    while (j > 0 && s[i] !== p[j]) j = next[j - 1];
    if (s[i] === p[j]) j++;
    if (j === p.length) {
      res.push(i - p.length + 1);
      // 完整匹配后继续查找可能重叠的下一次出现
      j = next[j - 1];
    }
  }
  return res;
}
```

**示例**

```ts
kmpSearchAll('abababab', 'abab'); // [0, 2, 4]
```

**易错点**

- 失配时用 `if` 而不是 `while` 回退
- 完整匹配后把 j 重置为 0，漏掉重叠的匹配
- next 数组的定义和使用时下标差一（`next[j - 1]` 还是 `next[j]`）

---

### H11. 合并 K 个有序链表

- **难度**: 中级
- **标签**: 链表, 堆, 分治
- **面试官看什么**: 两种解法的复杂度对比

**参考实现**

```ts
import { Heap } from './heap'; // H2 中的 Heap

class ListNode {
  constructor(public val: number, public next: ListNode | null = null) {}
}

/** 解法一：最小堆，堆中保存每个链表的当前头节点。O(N log k)，N 为节点总数 */
export function mergeKListsHeap(lists: (ListNode | null)[]): ListNode | null {
  const heap = new Heap<ListNode>((a, b) => a.val - b.val);
  for (const head of lists) if (head) heap.push(head);
  const dummy = new ListNode(0);
  let tail = dummy;
  while (heap.size) {
    const node = heap.pop()!;
    tail.next = node;
    tail = node;
    if (node.next) heap.push(node.next);
  }
  return dummy.next;
}

/** 解法二：分治两两合并。同样 O(N log k)，不需要额外的堆结构 */
export function mergeKListsDivide(lists: (ListNode | null)[]): ListNode | null {
  if (lists.length === 0) return null;
  let step = 1;
  while (step < lists.length) {
    for (let i = 0; i + step < lists.length; i += step * 2) {
      lists[i] = mergeTwo(lists[i], lists[i + step]);
    }
    step *= 2;
  }
  return lists[0];
}

function mergeTwo(a: ListNode | null, b: ListNode | null): ListNode | null {
  const dummy = new ListNode(0);
  let tail = dummy;
  while (a && b) {
    if (a.val <= b.val) { tail.next = a; a = a.next; }
    else { tail.next = b; b = b.next; }
    tail = tail.next;
  }
  tail.next = a ?? b;
  return dummy.next;
}
```

**易错点**

- 依次合并（第一个和第二个合并，结果再和第三个合并）是 O(N·k)
- 空链表入堆导致报错

---

### H12. 大数相乘

- **难度**: 中级
- **标签**: 字符串, 数学
- **面试官看什么**: 竖式乘法的位置关系、进位、前导零

**参考实现**

```ts
/** 两个非负整数字符串相乘。O(m·n) */
export function multiply(a: string, b: string): string {
  if (a === '0' || b === '0') return '0';
  const m = a.length, n = b.length;
  // m 位数乘 n 位数，结果最多 m + n 位
  const res = new Array<number>(m + n).fill(0);

  for (let i = m - 1; i >= 0; i--) {
    for (let j = n - 1; j >= 0; j--) {
      // a[i] * b[j] 的结果落在 res[i + j]（进位）和 res[i + j + 1]（个位）
      const sum = Number(a[i]) * Number(b[j]) + res[i + j + 1];
      res[i + j + 1] = sum % 10;
      res[i + j] += Math.floor(sum / 10);
    }
  }

  // 去掉前导零（最多一个，因为结果至少 m + n - 1 位）
  const start = res[0] === 0 ? 1 : 0;
  return res.slice(start).join('');
}
```

**示例**

```ts
multiply('123', '456'); // '56088'
multiply('99999999999999999999', '2'); // '199999999999999999998'
```

**易错点**

- 位置关系写错（`i + j` 和 `i + j + 1`）
- 进位只处理一次（这里 `res[i + j]` 可能暂时 ≥ 10，会在下一次计算 `res[i + j + 1]` 时被处理，最高位不会超过 9）
- 没有处理 `'0'` 的特殊情况导致输出 `'000'`
- 直接用 Number 相乘丢失精度
