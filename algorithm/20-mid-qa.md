# 数据结构与算法中级 Q&A

> 目标：掌握堆、单调栈 / 队列、Trie、并查集、LRU 等数据结构，能解决 Hot 100 中的经典中等题，熟练使用回溯、动态规划和图的算法模板。

---

## 一、进阶数据结构

### Q1. 堆是什么？怎么用数组实现？

- **难度**: 中级
- **标签**: 堆

**答案**

**堆**是一棵完全二叉树，最小堆中每个节点 ≤ 子节点，堆顶是最小值。用数组存储：下标 i 的子节点是 `2i+1`、`2i+2`，父节点是 `(i-1) >> 1`。

| 操作 | 复杂度 |
|------|--------|
| 查看堆顶 | O(1) |
| 插入（上浮） | O(log n) |
| 弹出堆顶（末尾元素放到堆顶，下沉） | O(log n) |
| 建堆（从最后一个非叶子节点向前下沉） | O(n) |

**用途**：优先队列、Top K、合并 K 个有序链表、任务调度（React Scheduler）、Dijkstra。

完整实现见 [40-coding-handwritten.md H2](./40-coding-handwritten.md)。

**口述模板（30–60 秒）**

1. 一句话结论：堆是用数组存储的完全二叉树，堆顶是最值。
2. 关键机制：插入时上浮，弹出时把末尾元素放到堆顶再下沉。
3. 对比或注意点：插入和弹出都是 O(log n)，建堆是 O(n)。
4. 可追问时补充：JS 没有内置的堆，面试中需要手写。

**常见追问**

- 为什么建堆是 O(n) 而不是 O(n log n)？→ 大部分节点在底层，下沉的距离很短；求和后是 O(n)。

---

### Q2. Top K 问题怎么解？

- **难度**: 中级
- **标签**: 堆, 快速选择

**答案**

求数组中最大的 K 个数：

| 方法 | 复杂度 | 适用 |
|------|--------|------|
| 排序 | O(n log n) | 简单 |
| **大小为 K 的最小堆** | O(n log k) | **数据流、n 很大** |
| **快速选择**（partition） | 平均 O(n) | 数据全在内存，可以修改数组 |

**最小堆思路**：维护 K 个元素的最小堆；新元素大于堆顶时替换堆顶并下沉；最后堆中就是最大的 K 个。

**第 K 大元素（快速选择）**：

```ts
function findKthLargest(nums: number[], k: number): number {
  const target = nums.length - k; // 第 k 大 = 升序后下标 n-k
  let lo = 0, hi = nums.length - 1;
  while (true) {
    const p = partition(nums, lo, hi); // 随机基准的 partition，见初级 Q22
    if (p === target) return nums[p];
    if (p < target) lo = p + 1;
    else hi = p - 1;
  }
}
```

**口述模板（30–60 秒）**

1. 一句话结论：Top K 可以用大小为 K 的最小堆，或者快速选择。
2. 关键机制：最小堆的堆顶是当前第 K 大，新元素比它大时替换。
3. 对比或注意点：堆适合数据流，快速选择平均 O(n) 但需要全部数据。
4. 可追问时补充：前 K 个高频元素先用哈希表计数，再用堆或桶排序。

**常见追问**

- 为什么求最大的 K 个用最小堆？→ 堆顶是 K 个中最小的，是「门槛」，比它小的直接丢弃。

---

### Q3. 单调栈适合解决什么问题？

- **难度**: 中级
- **标签**: 单调栈

**答案**

**适用**：找每个元素**左边 / 右边第一个更大（或更小）的元素**。O(n)。

**例：每日温度**（多少天后出现更高温度）：

```ts
function dailyTemperatures(t: number[]): number[] {
  const res = new Array(t.length).fill(0);
  const stack: number[] = []; // 存下标，对应温度单调递减
  for (let i = 0; i < t.length; i++) {
    // 当前温度比栈顶高：栈顶找到了「右边第一个更高」
    while (stack.length && t[i] > t[stack[stack.length - 1]]) {
      const j = stack.pop()!;
      res[j] = i - j;
    }
    stack.push(i);
  }
  return res;
}
```

**其他应用**：柱状图中最大的矩形、接雨水、去除重复字母（字典序最小）。

**口述模板（30–60 秒）**

1. 一句话结论：单调栈用于找每个元素左右两侧第一个更大或更小的元素。
2. 关键机制：栈中保持单调，新元素破坏单调性时弹出的元素就找到了答案。
3. 对比或注意点：栈中通常存下标，便于计算距离。
4. 可追问时补充：每个元素最多入栈出栈一次，所以是 O(n)。

**常见追问**

- 找左边第一个更小的元素？→ 维护单调递增栈，入栈前栈顶就是答案。

---

### Q4. 单调队列：滑动窗口最大值

- **难度**: 中级
- **标签**: 单调队列

**答案**

维护一个**下标的双端队列**，对应的值**单调递减**；队首是当前窗口最大值。

```ts
function maxSlidingWindow(nums: number[], k: number): number[] {
  const deque: number[] = [];
  let head = 0; // 用头指针模拟双端队列的队首，避免 shift 的 O(n)
  const res: number[] = [];
  for (let i = 0; i < nums.length; i++) {
    // 比当前元素小的元素以后不可能成为最大值，从队尾移除
    while (deque.length > head && nums[deque[deque.length - 1]] <= nums[i]) deque.pop();
    deque.push(i);
    // 队首已经滑出窗口
    if (deque[head] <= i - k) head++;
    if (i >= k - 1) res.push(nums[deque[head]]);
  }
  return res;
}
```

O(n) 时间。

**口述模板（30–60 秒）**

1. 一句话结论：用单调递减的双端队列维护窗口中的候选最大值。
2. 关键机制：新元素入队前，从队尾移除比它小的元素，因为它们不可能再成为最大值。
3. 对比或注意点：队首元素滑出窗口时要移除。
4. 可追问时补充：每个元素最多进出一次，整体 O(n)。

**常见追问**

- 用堆能做吗？→ 可以，O(n log n)，堆顶不在窗口内时延迟删除。

---

### Q5. Trie（前缀树）是什么？

- **难度**: 中级
- **标签**: Trie

**答案**

每个节点代表一个字符，从根到某节点的路径是一个前缀。插入和查找的复杂度是 O(单词长度)，与单词数量无关。

```ts
class TrieNode {
  children = new Map<string, TrieNode>();
  isEnd = false;
}

class Trie {
  private root = new TrieNode();

  insert(word: string) {
    let node = this.root;
    for (const ch of word) {
      if (!node.children.has(ch)) node.children.set(ch, new TrieNode());
      node = node.children.get(ch)!;
    }
    node.isEnd = true;
  }

  search(word: string) {
    return this.find(word)?.isEnd ?? false;
  }

  startsWith(prefix: string) {
    return this.find(prefix) !== null;
  }

  private find(s: string): TrieNode | null {
    let node = this.root;
    for (const ch of s) {
      const next = node.children.get(ch);
      if (!next) return null;
      node = next;
    }
    return node;
  }
}
```

**应用**：搜索框自动补全、敏感词过滤、路由匹配（前缀树路由）、单词搜索 II。

**口述模板（30–60 秒）**

1. 一句话结论：Trie 是按字符组织的树，用于高效的前缀查找。
2. 关键机制：插入和查找的复杂度只与单词长度有关。
3. 对比或注意点：要用 isEnd 区分完整单词和前缀。
4. 可追问时补充：前端常用于自动补全和路由匹配。

**常见追问**

- Trie 的缺点？→ 内存占用大（每个节点一个子节点表）；可以用压缩前缀树（Radix Tree）优化。

---

### Q6. 并查集是什么？

- **难度**: 中级
- **标签**: 并查集

**答案**

维护若干**不相交集合**，支持：`find(x)` 找所属集合的代表，`union(x, y)` 合并两个集合。

**优化**：**路径压缩**（find 时把节点直接挂到根上）+ **按秩合并**（矮树挂到高树下）→ 均摊接近 O(1)。

**应用**：连通分量数量、判断图中是否有环、朋友圈、冗余连接、账户合并。

完整实现见 [40-coding-handwritten.md H4](./40-coding-handwritten.md)。

**口述模板（30–60 秒）**

1. 一句话结论：并查集用于管理不相交集合的合并和查询。
2. 关键机制：每个集合是一棵树，根节点是代表。
3. 对比或注意点：路径压缩和按秩合并让操作均摊接近 O(1)。
4. 可追问时补充：常用于连通性和判断环的问题。

**常见追问**

- 并查集和 DFS 求连通分量的区别？→ 并查集适合**动态**添加边的场景，每次加边后都能快速查询连通性。

---

### Q7. LRU 缓存怎么实现 O(1)？

- **难度**: 中级
- **标签**: LRU, 设计

**答案**

**要求**：`get` 和 `put` 都是 O(1)；容量满时淘汰**最久未使用**的。

**方法 1：利用 Map 的插入顺序**（JS 中最简洁）：

```ts
class LRUCache {
  private map = new Map<number, number>();
  constructor(private capacity: number) {}

  get(key: number): number {
    if (!this.map.has(key)) return -1;
    const value = this.map.get(key)!;
    // 删除后重新插入，移到「最近使用」的末尾
    this.map.delete(key);
    this.map.set(key, value);
    return value;
  }

  put(key: number, value: number): void {
    this.map.delete(key);
    this.map.set(key, value);
    if (this.map.size > this.capacity) {
      // Map 的第一个键就是最久未使用的
      this.map.delete(this.map.keys().next().value!);
    }
  }
}
```

**方法 2：哈希表 + 双向链表**（面试官要求不用 Map 顺序时，见 [H1](./40-coding-handwritten.md)）。

**口述模板（30–60 秒）**

1. 一句话结论：LRU 需要 O(1) 的查找和 O(1) 的顺序调整。
2. 关键机制：哈希表负责查找，双向链表维护使用顺序。
3. 对比或注意点：JS 中 Map 保持插入顺序，删除后重新插入就能移到末尾。
4. 可追问时补充：面试官可能要求手写双向链表版本。

**常见追问**

- LRU 在前端的应用？→ 路由组件缓存（Vue 的 `keep-alive` 的 `max`）、请求缓存、图片缓存。

---

### Q8. 图怎么表示？怎么遍历？

- **难度**: 中级
- **标签**: 图

**答案**

| 表示 | 空间 | 适用 |
|------|------|------|
| **邻接表** | O(V + E) | 稀疏图（大多数题目） |
| 邻接矩阵 | O(V²) | 稠密图、需要 O(1) 判断两点是否相连 |

```ts
// 由边列表构建邻接表
function buildGraph(n: number, edges: [number, number][]): number[][] {
  const g: number[][] = Array.from({ length: n }, () => []);
  for (const [u, v] of edges) {
    g[u].push(v);
    g[v].push(u); // 无向图
  }
  return g;
}

function bfs(g: number[][], start: number): number[] {
  const dist = new Array(g.length).fill(-1);
  dist[start] = 0;
  let level = [start];
  while (level.length) {
    const next: number[] = [];
    for (const u of level) {
      for (const v of g[u]) {
        if (dist[v] === -1) {
          dist[v] = dist[u] + 1; // 入队时就标记，避免同一节点重复入队
          next.push(v);
        }
      }
    }
    level = next;
  }
  return dist;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：图通常用邻接表表示，用 DFS 或 BFS 遍历。
2. 关键机制：遍历时用 visited 标记避免重复访问。
3. 对比或注意点：BFS 要在入队时标记，否则同一节点可能多次入队。
4. 可追问时补充：BFS 求的是无权图的最短距离。

**常见追问**

- 有向图和无向图在建图时的区别？→ 无向图每条边加两次。

---

## 二、经典中等题

### Q9. 三数之和

- **难度**: 中级
- **标签**: 双指针, 排序

**答案**

找出所有和为 0 的不重复三元组。

**思路**：排序 → 固定第一个数 → 剩余部分用对撞双指针；**跳过重复值去重**。O(n²)。

```ts
function threeSum(nums: number[]): number[][] {
  nums.sort((a, b) => a - b);
  const res: number[][] = [];
  for (let i = 0; i < nums.length - 2; i++) {
    if (nums[i] > 0) break; // 最小的数大于 0，后面不可能和为 0
    if (i > 0 && nums[i] === nums[i - 1]) continue;
    let l = i + 1, r = nums.length - 1;
    while (l < r) {
      const sum = nums[i] + nums[l] + nums[r];
      if (sum < 0) l++;
      else if (sum > 0) r--;
      else {
        res.push([nums[i], nums[l], nums[r]]);
        while (l < r && nums[l] === nums[l + 1]) l++;
        while (l < r && nums[r] === nums[r - 1]) r--;
        l++;
        r--;
      }
    }
  }
  return res;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：排序后固定一个数，剩下两个数用对撞双指针。
2. 关键机制：和小于 0 左指针右移，大于 0 右指针左移。
3. 对比或注意点：三个位置都要跳过重复值来去重。
4. 可追问时补充：时间 O(n²)，排序后可以提前剪枝。

**常见追问**

- 四数之和？→ 再多一层循环，O(n³)；注意剪枝和整数溢出（JS 中一般无需考虑）。

---

### Q10. 最小覆盖子串

- **难度**: 中级
- **标签**: 滑动窗口

**答案**

找 s 中包含 t 所有字符的最短子串。

```ts
function minWindow(s: string, t: string): string {
  const need = new Map<string, number>();
  for (const ch of t) need.set(ch, (need.get(ch) ?? 0) + 1);
  let missing = t.length; // 还缺多少个字符（计重复）
  let left = 0, bestStart = 0, bestLen = Infinity;

  for (let right = 0; right < s.length; right++) {
    const ch = s[right];
    if ((need.get(ch) ?? 0) > 0) missing--;
    // 不在 t 中的字符也记录为负数，统一处理收缩逻辑
    need.set(ch, (need.get(ch) ?? 0) - 1);

    while (missing === 0) {
      if (right - left + 1 < bestLen) {
        bestLen = right - left + 1;
        bestStart = left;
      }
      const lc = s[left++];
      need.set(lc, need.get(lc)! + 1);
      if (need.get(lc)! > 0) missing++;
    }
  }
  return bestLen === Infinity ? '' : s.slice(bestStart, bestStart + bestLen);
}
```

**口述模板（30–60 秒）**

1. 一句话结论：用滑动窗口，右指针扩张直到覆盖所有字符，再收缩左指针求最短。
2. 关键机制：用计数表记录还需要的字符数量，missing 记录总共还缺几个。
3. 对比或注意点：收缩时某个字符的需求变为正数，说明窗口不再满足条件。
4. 可追问时补充：时间 O(n + m)。

**常见追问**

- 「找到字符串中所有字母异位词」？→ 固定长度的滑动窗口，比较计数。

---

### Q11. 合并区间

- **难度**: 中级
- **标签**: 排序, 区间

**答案**

```ts
function mergeIntervals(intervals: [number, number][]): [number, number][] {
  intervals.sort((a, b) => a[0] - b[0]);
  const res: [number, number][] = [];
  for (const [s, e] of intervals) {
    const last = res[res.length - 1];
    if (last && s <= last[1]) last[1] = Math.max(last[1], e);
    else res.push([s, e]);
  }
  return res;
}
```

**区间题的通用思路**：按起点（或终点）排序，然后线性扫描。

**口述模板（30–60 秒）**

1. 一句话结论：先按起点排序，再依次合并有重叠的区间。
2. 关键机制：当前区间的起点小于等于上一个区间的终点时合并。
3. 对比或注意点：合并时终点取两者的最大值，因为可能被完全包含。
4. 可追问时补充：时间 O(n log n)，瓶颈在排序。

**常见追问**

- 最少移除多少个区间使其不重叠？→ 按**终点**排序，贪心保留结束最早的区间。

---

### Q12. 接雨水

- **难度**: 中级
- **标签**: 双指针

**答案**

每个位置能接的水 = `min(左边最高, 右边最高) - 当前高度`。

**双指针**（O(n) 时间，O(1) 空间）：哪边的最高值更小，就处理哪边（这一边的水位已经确定）。

```ts
function trap(height: number[]): number {
  let l = 0, r = height.length - 1;
  let leftMax = 0, rightMax = 0, water = 0;
  while (l < r) {
    leftMax = Math.max(leftMax, height[l]);
    rightMax = Math.max(rightMax, height[r]);
    // 左边最高更小时，左指针位置的水位由 leftMax 决定（右边一定有不低于 rightMax 的柱子）
    if (leftMax < rightMax) water += leftMax - height[l++];
    else water += rightMax - height[r--];
  }
  return water;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：每个位置的水量等于左右最高柱子中较小的一个减去当前高度。
2. 关键机制：双指针从两端向中间，哪边的最高值更小就处理哪边。
3. 对比或注意点：较小一侧的水位已经确定，不受另一侧影响。
4. 可追问时补充：也可以用前后缀最大值数组或单调栈。

**常见追问**

- 用单调栈怎么做？→ 维护递减栈，遇到更高的柱子时弹出栈顶，按「凹槽」计算一层一层的水量。

---

### Q13. 二叉树的最近公共祖先

- **难度**: 中级
- **标签**: 二叉树, 递归

**答案**

```ts
function lowestCommonAncestor(
  root: TreeNode | null,
  p: TreeNode,
  q: TreeNode,
): TreeNode | null {
  if (!root || root === p || root === q) return root;
  const left = lowestCommonAncestor(root.left, p, q);
  const right = lowestCommonAncestor(root.right, p, q);
  // p、q 分别在左右子树中：当前节点就是最近公共祖先
  if (left && right) return root;
  return left ?? right;
}
```

**函数定义**：返回子树中 p 或 q 或它们的 LCA；都不在则返回 null。

**口述模板（30–60 秒）**

1. 一句话结论：后序遍历，左右子树分别查找 p 和 q。
2. 关键机制：左右都找到时，当前节点就是最近公共祖先。
3. 对比或注意点：只有一边找到时，把那一边的结果向上返回。
4. 可追问时补充：BST 中可以根据大小关系直接判断方向。

**常见追问**

- 在 DOM 树中找两个节点的最近公共祖先？→ 有 parent 指针时，把一个节点的祖先放进 Set，再从另一个节点向上找第一个在 Set 中的节点（类似链表相交）。

---

### Q14. 二叉树的序列化与反序列化

- **难度**: 中级
- **标签**: 二叉树, 设计

**答案**

前序遍历，空节点用 `#` 表示：

```ts
function serialize(root: TreeNode | null): string {
  const out: string[] = [];
  const dfs = (node: TreeNode | null) => {
    if (!node) {
      out.push('#');
      return;
    }
    out.push(String(node.val));
    dfs(node.left);
    dfs(node.right);
  };
  dfs(root);
  return out.join(',');
}

function deserialize(data: string): TreeNode | null {
  const tokens = data.split(',');
  let i = 0;
  const build = (): TreeNode | null => {
    const t = tokens[i++];
    if (t === '#') return null;
    const node = new TreeNode(Number(t));
    node.left = build();
    node.right = build();
    return node;
  };
  return build();
}
```

**口述模板（30–60 秒）**

1. 一句话结论：用前序遍历并记录空节点，就能唯一确定一棵树。
2. 关键机制：反序列化时按相同的前序顺序递归构建。
3. 对比或注意点：必须记录空节点，否则结构无法还原。
4. 可追问时补充：也可以用层序遍历实现。

**常见追问**

- 只有前序遍历（没有空节点标记）能还原吗？→ 不能；需要前序 + 中序（且值不重复）。

---

### Q15. 二叉树中的最大路径和

- **难度**: 中级
- **标签**: 二叉树, 递归

**答案**

路径可以从任意节点开始和结束。

**关键**：递归函数返回「从当前节点向下的单侧最大路径和」；在每个节点处，用「左单侧 + 根 + 右单侧」更新全局答案。

```ts
function maxPathSum(root: TreeNode | null): number {
  let best = -Infinity;
  const gain = (node: TreeNode | null): number => {
    if (!node) return 0;
    // 负的贡献不如不选
    const l = Math.max(0, gain(node.left));
    const r = Math.max(0, gain(node.right));
    best = Math.max(best, node.val + l + r);
    // 向上只能选择一侧，否则路径会分叉
    return node.val + Math.max(l, r);
  };
  gain(root);
  return best;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：递归返回单侧最大路径，在每个节点用左右两侧加根更新答案。
2. 关键机制：向上返回时只能选一侧，否则路径会分叉。
3. 对比或注意点：负数的贡献直接取 0。
4. 可追问时补充：答案初始化为负无穷，因为节点值可能全为负。

**常见追问**

- 「返回值」和「更新答案」不一致的递归模式还有哪些？→ 二叉树直径、最长同值路径，都是同样的模式。

---

### Q16. 搜索旋转排序数组

- **难度**: 中级
- **标签**: 二分查找

**答案**

有序数组在某处旋转（如 `[4,5,6,7,0,1,2]`），查找 target。

**关键**：二分后**至少有一半是有序的**；判断 target 是否在有序的那一半中。

```ts
function searchRotated(nums: number[], target: number): number {
  let lo = 0, hi = nums.length - 1;
  while (lo <= hi) {
    const mid = (lo + hi) >> 1;
    if (nums[mid] === target) return mid;
    if (nums[lo] <= nums[mid]) {
      // 左半有序
      if (nums[lo] <= target && target < nums[mid]) hi = mid - 1;
      else lo = mid + 1;
    } else {
      // 右半有序
      if (nums[mid] < target && target <= nums[hi]) lo = mid + 1;
      else hi = mid - 1;
    }
  }
  return -1;
}
```

**二分答案**：当答案具有单调性时（例如「最小的速度使得能在 H 小时内吃完」），对答案二分并检查可行性。

**口述模板（30–60 秒）**

1. 一句话结论：二分后至少有一半是有序的，判断 target 是否在有序的那一半。
2. 关键机制：比较 nums[lo] 和 nums[mid] 判断哪一半有序。
3. 对比或注意点：`nums[lo] <= nums[mid]` 中的等号处理 lo 等于 mid 的情况。
4. 可追问时补充：有重复元素时，`nums[lo] === nums[mid]` 无法判断，只能 lo++，最坏 O(n)。

**常见追问**

- 二分答案的典型题？→ 爱吃香蕉的珂珂、分割数组的最大值、在 D 天内送达包裹的能力。

---

### Q17. K 个一组翻转链表

- **难度**: 中级
- **标签**: 链表

**答案**

```ts
function reverseKGroup(head: ListNode | null, k: number): ListNode | null {
  const dummy = new ListNode(0, head);
  let groupPrev = dummy;
  while (true) {
    // 找到这一组的第 k 个节点，不足 k 个时保持原样
    let kth: ListNode | null = groupPrev;
    for (let i = 0; i < k && kth; i++) kth = kth.next;
    if (!kth) break;

    const groupNext = kth.next;
    // 反转本组：prev 初始为下一组的头，反转后本组尾部自然接上
    let prev: ListNode | null = groupNext;
    let cur = groupPrev.next;
    while (cur !== groupNext) {
      const next: ListNode | null = cur!.next;
      cur!.next = prev;
      prev = cur;
      cur = next;
    }
    const oldHead = groupPrev.next!;
    groupPrev.next = kth;
    groupPrev = oldHead;
  }
  return dummy.next;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：每次找到 k 个节点，反转后接回原链表。
2. 关键机制：反转时 prev 初始化为下一组的头节点，反转完自然接上。
3. 对比或注意点：不足 k 个的最后一组保持原样。
4. 可追问时补充：虚拟头节点让第一组的处理和其他组一致。

**常见追问**

- 两两交换链表节点？→ k = 2 的特例。

---

## 三、回溯

### Q18. 回溯的模板是什么？全排列怎么写？

- **难度**: 中级
- **标签**: 回溯

**答案**

**回溯** = 在决策树上 DFS：做选择 → 递归 → 撤销选择。

```text
function backtrack(路径, 选择列表):
  if 满足结束条件: 记录结果; return
  for 选择 in 选择列表:
    if 不合法: continue（剪枝）
    做选择
    backtrack(路径, 新的选择列表)
    撤销选择
```

```ts
function permute(nums: number[]): number[][] {
  const res: number[][] = [];
  const path: number[] = [];
  const used = new Array(nums.length).fill(false);
  const backtrack = () => {
    if (path.length === nums.length) {
      res.push([...path]); // 必须复制，path 之后还会被修改
      return;
    }
    for (let i = 0; i < nums.length; i++) {
      if (used[i]) continue;
      used[i] = true;
      path.push(nums[i]);
      backtrack();
      path.pop();
      used[i] = false;
    }
  };
  backtrack();
  return res;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：回溯是在决策树上做 DFS，做选择、递归、撤销选择。
2. 关键机制：全排列用 used 数组记录已经选择的元素。
3. 对比或注意点：记录结果时要复制路径。
4. 可追问时补充：复杂度通常是指数级，剪枝可以显著减少搜索量。

**常见追问**

- 有重复元素的全排列？→ 先排序；`if (i > 0 && nums[i] === nums[i-1] && !used[i-1]) continue`，保证相同元素按顺序使用。

---

### Q19. 子集和组合总和

- **难度**: 中级
- **标签**: 回溯

**答案**

**组合 / 子集**与排列的区别：**不考虑顺序** → 用 `start` 参数，只向后选择，避免 [1,2] 和 [2,1] 重复。

```ts
function subsets(nums: number[]): number[][] {
  const res: number[][] = [];
  const path: number[] = [];
  const backtrack = (start: number) => {
    res.push([...path]); // 每个节点都是一个子集
    for (let i = start; i < nums.length; i++) {
      path.push(nums[i]);
      backtrack(i + 1);
      path.pop();
    }
  };
  backtrack(0);
  return res;
}

/** 组合总和：元素可以重复使用，所以递归时传 i 而不是 i + 1 */
function combinationSum(candidates: number[], target: number): number[][] {
  candidates.sort((a, b) => a - b);
  const res: number[][] = [];
  const path: number[] = [];
  const backtrack = (start: number, remain: number) => {
    if (remain === 0) {
      res.push([...path]);
      return;
    }
    for (let i = start; i < candidates.length; i++) {
      if (candidates[i] > remain) break; // 已排序，后面更大，剪枝
      path.push(candidates[i]);
      backtrack(i, remain - candidates[i]);
      path.pop();
    }
  };
  backtrack(0, target);
  return res;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：组合和子集用 start 参数只向后选择，避免顺序不同的重复。
2. 关键机制：子集在每个节点都记录结果，组合在满足条件时记录。
3. 对比或注意点：元素可以重复使用时递归传 i，不可以时传 i + 1。
4. 可追问时补充：排序后可以在超过目标时直接 break 剪枝。

**常见追问**

- 子集有多少个？→ 2ⁿ 个，复杂度 O(n · 2ⁿ)。

---

### Q20. N 皇后问题与剪枝

- **难度**: 中级
- **标签**: 回溯, 剪枝

**答案**

逐行放置皇后，用三个集合记录被占用的**列、主对角线（r - c）、副对角线（r + c）**，O(1) 判断是否冲突。

```ts
function solveNQueens(n: number): string[][] {
  const res: string[][] = [];
  const queens: number[] = []; // queens[r] = 第 r 行皇后所在的列
  const cols = new Set<number>(), diag1 = new Set<number>(), diag2 = new Set<number>();

  const backtrack = (r: number) => {
    if (r === n) {
      res.push(queens.map((c) => '.'.repeat(c) + 'Q' + '.'.repeat(n - c - 1)));
      return;
    }
    for (let c = 0; c < n; c++) {
      if (cols.has(c) || diag1.has(r - c) || diag2.has(r + c)) continue;
      queens.push(c); cols.add(c); diag1.add(r - c); diag2.add(r + c);
      backtrack(r + 1);
      queens.pop(); cols.delete(c); diag1.delete(r - c); diag2.delete(r + c);
    }
  };
  backtrack(0);
  return res;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：逐行放置皇后，用集合记录被占用的列和两条对角线。
2. 关键机制：同一主对角线上 r - c 相同，副对角线上 r + c 相同。
3. 对比或注意点：O(1) 的冲突判断是剪枝的关键。
4. 可追问时补充：可以用位运算进一步优化。

**常见追问**

- 剪枝的常见方式？→ 可行性剪枝（不合法直接跳过）、最优性剪枝（当前已经不可能比最优解更好）、排序后提前终止。

---

## 四、动态规划

### Q21. 动态规划题目的解题步骤？

- **难度**: 中级
- **标签**: 动态规划

**答案**

1. **判断**：求最值、计数、可行性；问题可以拆分成重叠的子问题
2. **定义状态**：`dp[i]` / `dp[i][j]` 表示什么（最关键的一步）
3. **转移方程**：从「最后一步」思考，`dp[i]` 由哪些更小的状态得到
4. **初始化**：边界状态
5. **遍历顺序**：保证计算 `dp[i]` 时依赖的状态已经计算
6. **返回值**：`dp[n]` 还是 `max(dp)`
7. **空间优化**：只依赖前一行 / 前几项时压缩

**常见模型**：线性（打家劫舍）、子序列（LIS、LCS）、背包、区间（最长回文子序列）、状态机（股票）、树形 DP。

**口述模板（30–60 秒）**

1. 一句话结论：DP 的核心是定义状态和找到转移方程。
2. 关键机制：从最后一步思考，当前状态由哪些子状态得到。
3. 对比或注意点：遍历顺序要保证依赖的状态已经计算。
4. 可追问时补充：只依赖前一行时可以压缩成一维。

**常见追问**

- 怎么判断用贪心还是 DP？→ 能证明局部最优推出全局最优用贪心；否则需要考虑所有子问题，用 DP。

---

### Q22. 最长递增子序列（LIS）

- **难度**: 中级
- **标签**: 动态规划, 二分

**答案**

**O(n²) DP**：`dp[i]` = 以 `nums[i]` 结尾的 LIS 长度；`dp[i] = max(dp[j] + 1)`，其中 `j < i` 且 `nums[j] < nums[i]`。

**O(n log n) 贪心 + 二分**：`tails[k]` = 长度为 k+1 的递增子序列的**最小结尾**；对每个数二分找到第一个 ≥ 它的位置替换（或追加）。

```ts
function lengthOfLIS(nums: number[]): number {
  const tails: number[] = [];
  for (const x of nums) {
    let lo = 0, hi = tails.length;
    while (lo < hi) {
      const mid = (lo + hi) >> 1;
      if (tails[mid] < x) lo = mid + 1;
      else hi = mid;
    }
    // 结尾越小，后面越容易接上更多的数
    tails[lo] = x;
  }
  return tails.length;
}
```

**注意**：`tails` 本身**不是**一个合法的 LIS，只有长度是对的；需要序列时要额外记录前驱（Vue 3 diff 中的做法，见 [H5](./40-coding-handwritten.md)）。

**口述模板（30–60 秒）**

1. 一句话结论：LIS 可以用 O(n²) 的 DP，也可以用贪心加二分做到 O(n log n)。
2. 关键机制：tails 记录每个长度的递增子序列的最小结尾，每个数二分找到替换位置。
3. 对比或注意点：tails 数组不是真正的 LIS，求序列需要记录前驱。
4. 可追问时补充：Vue 3 的 diff 用 LIS 找出不需要移动的节点。

**常见追问**

- 为什么替换不影响长度的正确性？→ 替换只是让同样长度的子序列结尾更小，为后续扩展创造更好的条件，不改变已有的长度。

---

### Q23. 最长公共子序列和编辑距离

- **难度**: 中级
- **标签**: 动态规划

**答案**

**LCS**：`dp[i][j]` = `a` 前 i 个字符与 `b` 前 j 个字符的 LCS 长度。

```text
a[i-1] === b[j-1]：dp[i][j] = dp[i-1][j-1] + 1
否则：dp[i][j] = max(dp[i-1][j], dp[i][j-1])
```

**编辑距离**：`dp[i][j]` = 把 `a` 前 i 个字符变成 `b` 前 j 个字符的最少操作数。

```ts
function minDistance(a: string, b: string): number {
  const m = a.length, n = b.length;
  const dp = Array.from({ length: m + 1 }, (_, i) => {
    const row = new Array(n + 1).fill(0);
    row[0] = i; // 删除 i 个字符
    return row;
  });
  for (let j = 0; j <= n; j++) dp[0][j] = j; // 插入 j 个字符

  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (a[i - 1] === b[j - 1]) dp[i][j] = dp[i - 1][j - 1];
      else {
        dp[i][j] = 1 + Math.min(
          dp[i - 1][j], // 删除 a[i-1]
          dp[i][j - 1], // 插入 b[j-1]
          dp[i - 1][j - 1], // 替换
        );
      }
    }
  }
  return dp[m][n];
}
```

**前端应用**：文本 diff（基于 LCS，实际用 Myers 算法）、拼写纠错、模糊搜索的相似度。

**口述模板（30–60 秒）**

1. 一句话结论：两个字符串的 DP 通常定义 dp[i][j] 为两个前缀之间的答案。
2. 关键机制：最后一个字符相同时继承左上角，不同时从删除、插入、替换中取最小。
3. 对比或注意点：初始化第一行和第一列，表示和空字符串之间的距离。
4. 可追问时补充：只依赖上一行，可以压缩成一维。

**常见追问**

- 怎么输出具体的编辑步骤？→ 从 `dp[m][n]` 反向回溯，判断来自哪个方向。

---

### Q24. 背包问题

- **难度**: 中级
- **标签**: 动态规划, 背包

**答案**

**0-1 背包**（每个物品最多选一次）：`dp[j]` = 容量为 j 时的最大价值；**倒序**遍历容量，保证每个物品只用一次。

```ts
function knapsack01(weights: number[], values: number[], cap: number): number {
  const dp = new Array(cap + 1).fill(0);
  for (let i = 0; i < weights.length; i++) {
    for (let j = cap; j >= weights[i]; j--) {
      dp[j] = Math.max(dp[j], dp[j - weights[i]] + values[i]);
    }
  }
  return dp[cap];
}
```

**完全背包**（物品可以无限选）：**正序**遍历容量。

**例：零钱兑换**（最少硬币数，完全背包）：

```ts
function coinChange(coins: number[], amount: number): number {
  const dp = new Array(amount + 1).fill(Infinity);
  dp[0] = 0;
  for (const c of coins) {
    for (let j = c; j <= amount; j++) {
      dp[j] = Math.min(dp[j], dp[j - c] + 1);
    }
  }
  return dp[amount] === Infinity ? -1 : dp[amount];
}
```

**变体**：分割等和子集（0-1 背包可行性）、目标和、零钱兑换 II（组合数：先遍历物品；排列数：先遍历容量）。

**口述模板（30–60 秒）**

1. 一句话结论：0-1 背包每个物品选一次，完全背包可以选无限次。
2. 关键机制：一维 DP 中，0-1 背包倒序遍历容量，完全背包正序遍历。
3. 对比或注意点：倒序是为了保证使用的是上一轮（未包含当前物品）的状态。
4. 可追问时补充：求组合数先遍历物品，求排列数先遍历容量。

**常见追问**

- 分割等和子集怎么转成背包？→ 能否选出一些数使和为总和的一半，即容量为 sum/2 的 0-1 背包可行性问题。

---

### Q25. 打家劫舍和股票买卖（状态机 DP）

- **难度**: 中级
- **标签**: 动态规划

**答案**

**打家劫舍**：`dp[i] = max(dp[i-1], dp[i-2] + nums[i])`。

```ts
function rob(nums: number[]): number {
  let prev = 0, cur = 0;
  for (const x of nums) [prev, cur] = [cur, Math.max(cur, prev + x)];
  return cur;
}
```

**股票买卖（含冷冻期 / 手续费 / 最多 k 次）用状态机**：每天有「持有」和「不持有」两种状态。

```ts
/** 含手续费，可以多次交易 */
function maxProfitWithFee(prices: number[], fee: number): number {
  let hold = -prices[0]; // 当天结束时持有股票的最大收益
  let cash = 0; // 当天结束时不持有股票的最大收益
  for (let i = 1; i < prices.length; i++) {
    const newCash = Math.max(cash, hold + prices[i] - fee);
    hold = Math.max(hold, cash - prices[i]);
    cash = newCash;
  }
  return cash;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：打家劫舍是线性 DP，股票问题用状态机 DP。
2. 关键机制：股票问题每天有持有和不持有两种状态，分别转移。
3. 对比或注意点：冷冻期需要增加一个状态，最多 k 次交易需要增加交易次数的维度。
4. 可追问时补充：都只依赖前一天，可以用变量代替数组。

**常见追问**

- 打家劫舍 II（环形）？→ 分别计算「不偷第一家」和「不偷最后一家」两种情况取最大值。

---

### Q26. 最大子数组和

- **难度**: 中级
- **标签**: 动态规划

**答案**

**Kadane 算法**：`cur` = 以当前元素结尾的最大和；如果之前的和为负，就从当前元素重新开始。

```ts
function maxSubArray(nums: number[]): number {
  let cur = nums[0], best = nums[0];
  for (let i = 1; i < nums.length; i++) {
    cur = Math.max(nums[i], cur + nums[i]);
    best = Math.max(best, cur);
  }
  return best;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：用 Kadane 算法，维护以当前元素结尾的最大子数组和。
2. 关键机制：之前的和为负时，从当前元素重新开始。
3. 对比或注意点：答案要初始化为第一个元素，因为可能全为负数。
4. 可追问时补充：也可以用前缀和减去之前的最小前缀和。

**常见追问**

- 乘积最大子数组？→ 同时维护最大和最小乘积（负数乘负数变成最大）。

---

## 五、图

### Q27. 拓扑排序：课程表

- **难度**: 中级
- **标签**: 图, 拓扑排序

**答案**

有依赖关系的任务能否全部完成（有向图是否无环），并给出顺序。

**Kahn 算法（BFS）**：入度为 0 的节点入队 → 出队时把它指向的节点入度减 1 → 新的入度为 0 的节点入队；处理的节点数 < 总数说明有环。

```ts
function findOrder(n: number, prerequisites: [number, number][]): number[] {
  const graph: number[][] = Array.from({ length: n }, () => []);
  const indegree = new Array(n).fill(0);
  for (const [course, pre] of prerequisites) {
    graph[pre].push(course);
    indegree[course]++;
  }
  const queue: number[] = [];
  for (let i = 0; i < n; i++) if (indegree[i] === 0) queue.push(i);
  const order: number[] = [];
  for (let h = 0; h < queue.length; h++) {
    const u = queue[h];
    order.push(u);
    for (const v of graph[u]) {
      if (--indegree[v] === 0) queue.push(v);
    }
  }
  return order.length === n ? order : [];
}
```

**前端应用**：模块打包的依赖顺序、Monorepo 的任务调度、响应式系统中计算属性的更新顺序。

**口述模板（30–60 秒）**

1. 一句话结论：拓扑排序用于有依赖关系的任务排序，可以用 Kahn 算法。
2. 关键机制：不断取出入度为 0 的节点，并减少它指向的节点的入度。
3. 对比或注意点：最终处理的节点数小于总数说明有环。
4. 可追问时补充：也可以用 DFS 的后序遍历反转得到拓扑序。

**常见追问**

- DFS 怎么判断有环？→ 三色标记：未访问、访问中、已完成；遇到「访问中」的节点说明有环。

---

### Q28. 岛屿类问题的变体

- **难度**: 中级
- **标签**: 图, DFS / BFS

**答案**

| 题目 | 要点 |
|------|------|
| 岛屿数量 | DFS / BFS 淹没，计数（见初级 Q27） |
| 岛屿最大面积 | DFS 返回面积 |
| 被围绕的区域 | **从边界出发**标记不会被围绕的 O，其余的 O 翻转 |
| 腐烂的橘子 | **多源 BFS**：所有腐烂橘子同时入队，按层计时 |
| 太平洋大西洋水流 | 从两个海洋的边界**反向** DFS，取交集 |

**多源 BFS 模板**：

```ts
function orangesRotting(grid: number[][]): number {
  const m = grid.length, n = grid[0].length;
  let level: [number, number][] = [];
  let fresh = 0;
  for (let i = 0; i < m; i++) {
    for (let j = 0; j < n; j++) {
      if (grid[i][j] === 2) level.push([i, j]);
      else if (grid[i][j] === 1) fresh++;
    }
  }
  let minutes = 0;
  const dirs = [[1, 0], [-1, 0], [0, 1], [0, -1]];
  while (level.length && fresh > 0) {
    const next: [number, number][] = [];
    for (const [i, j] of level) {
      for (const [di, dj] of dirs) {
        const x = i + di, y = j + dj;
        if (x >= 0 && y >= 0 && x < m && y < n && grid[x][y] === 1) {
          grid[x][y] = 2;
          fresh--;
          next.push([x, y]);
        }
      }
    }
    level = next;
    minutes++;
  }
  return fresh === 0 ? minutes : -1;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：网格题本质是图的遍历，DFS 求连通，BFS 求最短步数。
2. 关键机制：多个起点同时扩散时用多源 BFS，所有起点一起入队。
3. 对比或注意点：「被围绕」类问题从边界反向出发更简单。
4. 可追问时补充：要原地标记或用 visited 数组避免重复访问。

**常见追问**

- 为什么要从边界出发？→ 与边界连通的区域不会被围绕，从边界标记它们只需要一次遍历。

---

### Q29. 最短路径：BFS 和 Dijkstra

- **难度**: 中级
- **标签**: 图, 最短路径

**答案**

| 场景 | 算法 | 复杂度 |
|------|------|--------|
| 无权图（或边权都相同） | **BFS** | O(V + E) |
| 边权为 0 或 1 | 0-1 BFS（双端队列） | O(V + E) |
| 非负权图 | **Dijkstra**（最小堆） | O((V + E) log V) |
| 有负权边 | Bellman-Ford | O(VE) |
| 所有点对 | Floyd | O(V³) |

**Dijkstra**：每次从最小堆取出距离最小的未确定节点，用它松弛邻居。负权边会破坏「取出时距离已经确定」的前提。

完整实现见 [40-coding-handwritten.md H9](./40-coding-handwritten.md)。

**口述模板（30–60 秒）**

1. 一句话结论：无权图用 BFS，非负权图用 Dijkstra。
2. 关键机制：Dijkstra 每次取出距离最小的节点，用它更新邻居的距离。
3. 对比或注意点：有负权边时 Dijkstra 不正确，要用 Bellman-Ford。
4. 可追问时补充：用最小堆实现，复杂度 O((V + E) log V)。

**常见追问**

- 单词接龙（每次改一个字母）？→ 无权图最短路径，BFS；可以双向 BFS 优化。

---

## 六、前端相关的算法题

### Q30. 数组扁平化和去重的复杂度

- **难度**: 中级
- **标签**: 数组, 前端

**答案**

```ts
/** 迭代版扁平化，避免深层嵌套时递归栈溢出 */
function flatten(arr: unknown[], depth = Infinity): unknown[] {
  const res: unknown[] = [];
  const stack: [unknown, number][] = arr.map((x) => [x, depth] as [unknown, number]).reverse();
  while (stack.length) {
    const [item, d] = stack.pop()!;
    if (Array.isArray(item) && d > 0) {
      for (let i = item.length - 1; i >= 0; i--) stack.push([item[i], d - 1]);
    } else {
      res.push(item);
    }
  }
  return res;
}
```

**去重**：

| 方法 | 复杂度 |
|------|--------|
| `[...new Set(arr)]` | O(n) |
| `filter` + `indexOf` | O(n²) |
| 对象数组按 id 去重 | Map，O(n) |

**口述模板（30–60 秒）**

1. 一句话结论：扁平化可以用递归或栈，去重用 Set 是 O(n)。
2. 关键机制：用栈迭代可以避免深层嵌套时的栈溢出。
3. 对比或注意点：filter 加 indexOf 去重是 O(n²)。
4. 可追问时补充：对象数组按某个字段去重用 Map。

**常见追问**

- `Set` 能去重 `NaN` 吗？→ 能（SameValueZero）；`indexOf` 找不到 `NaN`。

---

### Q31. 深拷贝如何处理循环引用？

- **难度**: 中级
- **标签**: 图, 前端

**答案**

对象之间的引用构成一个**图**（可能有环），深拷贝就是图的遍历：用 `WeakMap` 记录「原对象 → 拷贝」，遇到已访问的对象直接返回拷贝。

```ts
function deepClone<T>(value: T, seen = new WeakMap<object, unknown>()): T {
  if (value === null || typeof value !== 'object') return value;
  if (seen.has(value)) return seen.get(value) as T;

  if (value instanceof Date) return new Date(value) as T;
  if (value instanceof RegExp) return new RegExp(value.source, value.flags) as T;
  if (value instanceof Map) {
    const m = new Map();
    seen.set(value, m);
    value.forEach((v, k) => m.set(deepClone(k, seen), deepClone(v, seen)));
    return m as T;
  }
  if (value instanceof Set) {
    const s = new Set();
    seen.set(value, s);
    value.forEach((v) => s.add(deepClone(v, seen)));
    return s as T;
  }

  const copy = Array.isArray(value) ? [] : Object.create(Object.getPrototypeOf(value));
  // 先登记再递归：子属性指回自身时能拿到这份拷贝
  seen.set(value, copy);
  for (const key of Reflect.ownKeys(value)) {
    copy[key] = deepClone((value as any)[key], seen);
  }
  return copy;
}
```

**现代方案**：`structuredClone`（支持循环引用、Map、Set、Date，不支持函数和 DOM 节点）。

**口述模板（30–60 秒）**

1. 一句话结论：深拷贝是对象图的遍历，用 WeakMap 记录已拷贝的对象处理循环引用。
2. 关键机制：创建拷贝后先登记到 WeakMap，再递归拷贝属性。
3. 对比或注意点：Date、RegExp、Map、Set 需要特殊处理。
4. 可追问时补充：现代环境可以直接用 structuredClone。

**常见追问**

- 为什么用 WeakMap 而不是 Map？→ 不阻止原对象被垃圾回收。

---

### Q32. 大数相加

- **难度**: 中级
- **标签**: 字符串, 前端

**答案**

JS 的 Number 超过 `2^53 - 1` 会丢失精度。用字符串逐位相加：

```ts
function addStrings(a: string, b: string): string {
  let i = a.length - 1, j = b.length - 1, carry = 0;
  const res: number[] = [];
  while (i >= 0 || j >= 0 || carry) {
    const sum = (i >= 0 ? +a[i--] : 0) + (j >= 0 ? +b[j--] : 0) + carry;
    res.push(sum % 10);
    carry = Math.floor(sum / 10);
  }
  return res.reverse().join('');
}
```

**现代方案**：`BigInt`（`123n + 456n`）；但面试中通常要求手写。

**口述模板（30–60 秒）**

1. 一句话结论：大数相加用字符串从低位到高位逐位相加，处理进位。
2. 关键机制：循环条件包含进位，保证最高位的进位不丢失。
3. 对比或注意点：Number 超过 2^53 - 1 会丢失精度。
4. 可追问时补充：现代环境可以用 BigInt。

**常见追问**

- 大数相乘？→ 竖式乘法：`res[i + j + 1] += a[i] * b[j]`，最后统一处理进位（见 [H12](./40-coding-handwritten.md)）。
