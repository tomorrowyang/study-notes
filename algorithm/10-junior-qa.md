# 数据结构与算法初级 Q&A

> 目标：掌握复杂度分析和基础数据结构，能独立解决数组、字符串、链表、二叉树、排序、二分查找的简单题，理解递归、BFS / DFS、贪心和动态规划的基本思想。代码使用 TypeScript。

---

## 一、复杂度与基础数据结构

### Q1. 时间复杂度怎么分析？

- **难度**: 初级
- **标签**: 复杂度

**答案**

时间复杂度描述**输入规模 n 增长时，执行步骤数的增长趋势**，用大 O 表示（忽略常数和低阶项）。

**分析方法**：

- 单层循环 O(n)；嵌套循环相乘 O(n²)
- 每次规模减半（二分）O(log n)
- 递归：**递归调用次数 × 每次的工作量**；例如归并排序 T(n) = 2T(n/2) + O(n) = O(n log n)
- 注意**隐藏的循环**：`arr.includes`、`indexOf`、`shift`、`splice`、`slice` 都是 O(n)

| 复杂度 | 例子 | n = 10⁵ 时是否可行 |
|--------|------|-------------------|
| O(1) | 哈希表查找 | ✅ |
| O(log n) | 二分查找 | ✅ |
| O(n) | 遍历 | ✅ |
| O(n log n) | 排序 | ✅ |
| O(n²) | 双重循环 | ⚠️ 约 10¹⁰，超时 |
| O(2ⁿ) | 子集枚举 | 只适合 n ≤ 20 左右 |

**口述模板（30–60 秒）**

1. 一句话结论：时间复杂度描述执行步骤随输入规模增长的趋势。
2. 关键机制：循环看次数，嵌套相乘，递归看调用次数乘以每次的工作量。
3. 对比或注意点：要注意 includes、shift、splice 这些隐藏的 O(n) 操作。
4. 可追问时补充：可以根据数据规模反推需要的复杂度，例如 10⁵ 需要 O(n log n) 以内。

**常见追问**

- 「均摊复杂度」是什么？→ 偶尔一次操作很慢，但平均到每次是 O(1)，例如动态数组扩容后的 `push`。

---

### Q2. 空间复杂度是什么？递归的空间怎么算？

- **难度**: 初级
- **标签**: 复杂度

**答案**

空间复杂度是算法**额外**使用的内存随 n 的增长趋势（通常不计输入本身和输出）。

- 新建长度为 n 的数组：O(n)
- 哈希表存 n 个元素：O(n)
- **递归**：调用栈的最大深度 × 每层的空间。例如二叉树递归遍历是 O(h)，h 是树高（平衡时 log n，退化成链时 n）

**口述模板（30–60 秒）**

1. 一句话结论：空间复杂度衡量算法额外使用的内存。
2. 关键机制：新建的数组、哈希表和递归调用栈都要计算。
3. 对比或注意点：递归的空间等于最大递归深度。
4. 可追问时补充：很多题目可以用原地修改或双指针把空间降到 O(1)。

**常见追问**

- JS 递归太深会怎样？→ 调用栈溢出（Maximum call stack size exceeded），一般在一万层左右，因引擎和栈帧大小而异 → 改为迭代 + 显式栈。

---

### Q3. 数组和链表有什么区别？

- **难度**: 初级
- **标签**: 数组, 链表

**答案**

| 操作 | 数组 | 链表 |
|------|------|------|
| 按下标访问 | O(1) | O(n) |
| 头部插入 / 删除 | O(n)（移动元素） | O(1) |
| 尾部插入 | 均摊 O(1) | O(1)（有尾指针时） |
| 已知节点后插入 / 删除 | O(n) | O(1) |
| 内存 | 连续，缓存友好 | 分散，有指针开销 |

```ts
class ListNode {
  constructor(public val: number, public next: ListNode | null = null) {}
}
```

**口述模板（30–60 秒）**

1. 一句话结论：数组支持 O(1) 随机访问，链表支持 O(1) 插入删除。
2. 关键机制：数组内存连续，链表通过指针连接。
3. 对比或注意点：数组头部插入要移动所有元素。
4. 可追问时补充：数组内存连续，对 CPU 缓存更友好，实际性能常常更好。

**常见追问**

- JS 的数组是真正的连续数组吗？→ V8 对元素类型一致的密集数组用连续存储（快速元素）；稀疏数组或混合类型会退化为字典模式。

---

### Q4. 栈和队列在 JS 中怎么实现？

- **难度**: 初级
- **标签**: 栈, 队列

**答案**

- **栈**（后进先出）：数组的 `push` / `pop`，都是 O(1)
- **队列**（先进先出）：数组的 `push` / `shift` —— **`shift` 是 O(n)**，数据量大时用指针或链表实现

```ts
/** 基于数组 + 头指针的队列，出队 O(1) */
class Queue<T> {
  private items: T[] = [];
  private head = 0;

  enqueue(x: T) {
    this.items.push(x);
  }

  dequeue(): T | undefined {
    if (this.head >= this.items.length) return undefined;
    const x = this.items[this.head++];
    // 已出队的元素过多时整理一次，避免数组无限增长
    if (this.head > 1024 && this.head * 2 > this.items.length) {
      this.items = this.items.slice(this.head);
      this.head = 0;
    }
    return x;
  }

  get size() {
    return this.items.length - this.head;
  }
}
```

**口述模板（30–60 秒）**

1. 一句话结论：栈用数组的 push 和 pop，队列要注意 shift 是 O(n)。
2. 关键机制：队列可以用头指针记录出队位置，做到 O(1) 出队。
3. 对比或注意点：BFS 中数据量大时，用 shift 可能超时。
4. 可追问时补充：双端队列可以用环形数组实现。

**常见追问**

- 用两个栈实现队列？→ 入队压入栈 A；出队时如果栈 B 为空，把 A 全部倒入 B，再从 B 弹出；均摊 O(1)。

---

### Q5. Map、Set 和普通对象作为哈希表有什么区别？

- **难度**: 初级
- **标签**: 哈希表

**答案**

| | Object | Map | Set |
|--|--------|-----|-----|
| 键类型 | 字符串 / Symbol（数字会转字符串） | 任意值 | 任意值 |
| 顺序 | 整数键升序，其余按插入 | **按插入顺序** | 按插入顺序 |
| 大小 | `Object.keys().length` | `size` | `size` |
| 原型链键 | 可能冲突（`__proto__`、`constructor`） | 无 | 无 |
| 频繁增删 | 性能一般 | 优化更好 | 优化更好 |

增删查平均 O(1)。算法题中推荐用 **Map / Set**。

**口述模板（30–60 秒）**

1. 一句话结论：算法题中推荐用 Map 和 Set 作为哈希表。
2. 关键机制：Map 的键可以是任意类型，并且保持插入顺序。
3. 对比或注意点：对象的数字键会被转成字符串，还可能和原型上的属性冲突。
4. 可追问时补充：Map 保持插入顺序的特性可以用来实现 LRU。

**常见追问**

- 用数组作为 Map 的键会怎样？→ 按引用比较，两个内容相同的数组是不同的键；需要转成字符串（如 `${a},${b}`）。

---

## 二、数组与字符串

### Q6. 两数之和怎么做？

- **难度**: 初级
- **标签**: 哈希表

**答案**

题目：数组中找两个数，和为 target，返回下标。

**思路**：遍历时用 Map 记录「值 → 下标」；对每个数，查找 `target - x` 是否已出现。O(n) 时间，O(n) 空间。

```ts
function twoSum(nums: number[], target: number): [number, number] | [] {
  const seen = new Map<number, number>();
  for (let i = 0; i < nums.length; i++) {
    const j = seen.get(target - nums[i]);
    if (j !== undefined) return [j, i];
    // 先查再存，避免同一个元素使用两次
    seen.set(nums[i], i);
  }
  return [];
}
```

**口述模板（30–60 秒）**

1. 一句话结论：用哈希表记录已经遍历过的值，把查找降到 O(1)。
2. 关键机制：对每个数查找 target 减去它的值是否出现过。
3. 对比或注意点：先查再存，避免同一个元素用两次。
4. 可追问时补充：如果数组有序，可以用双指针做到 O(1) 空间。

**常见追问**

- 为什么判断 `j !== undefined` 而不是 `if (j)`？→ 下标 0 是假值。

---

### Q7. 双指针有哪些常见用法？

- **难度**: 初级
- **标签**: 双指针

**答案**

| 类型 | 例子 |
|------|------|
| **对撞指针**（两端向中间） | 有序数组两数之和、反转、回文、盛水最多的容器 |
| **快慢指针**（同向） | 原地删除重复元素、移动零、链表环 / 中点 |
| **两个数组各一个指针** | 合并有序数组 |

```ts
/** 原地移动零：慢指针指向下一个非零元素应放的位置 */
function moveZeroes(nums: number[]): void {
  let slow = 0;
  for (let fast = 0; fast < nums.length; fast++) {
    if (nums[fast] !== 0) {
      [nums[slow], nums[fast]] = [nums[fast], nums[slow]];
      slow++;
    }
  }
}

/** 合并两个有序数组到 nums1（nums1 尾部有足够空间）：从后往前填，避免覆盖 */
function merge(nums1: number[], m: number, nums2: number[], n: number): void {
  let i = m - 1, j = n - 1, k = m + n - 1;
  while (j >= 0) {
    nums1[k--] = i >= 0 && nums1[i] > nums2[j] ? nums1[i--] : nums2[j--];
  }
}
```

**口述模板（30–60 秒）**

1. 一句话结论：双指针分为对撞指针、快慢指针和两个序列各一个指针。
2. 关键机制：利用有序性或位置关系，每一步排除一部分可能，把 O(n²) 降到 O(n)。
3. 对比或注意点：原地修改时要注意写入位置不能覆盖未处理的数据。
4. 可追问时补充：合并有序数组从后往前填就是为了避免覆盖。

**常见追问**

- 合并时为什么只循环到 `j >= 0`？→ nums2 处理完后，nums1 剩余的元素已经在正确位置上。

---

### Q8. 滑动窗口怎么用？（无重复字符的最长子串）

- **难度**: 初级
- **标签**: 滑动窗口

**答案**

**适用**：求满足条件的**连续**子数组 / 子串的最长 / 最短 / 个数。

**模板**：右指针扩张窗口；不满足条件时左指针收缩；在合适的时机更新答案。

```ts
function lengthOfLongestSubstring(s: string): number {
  const lastIndex = new Map<string, number>();
  let left = 0;
  let best = 0;
  for (let right = 0; right < s.length; right++) {
    const prev = lastIndex.get(s[right]);
    // 只有重复字符在当前窗口内时才需要移动左边界；窗口外的旧位置不影响
    if (prev !== undefined && prev >= left) left = prev + 1;
    lastIndex.set(s[right], right);
    best = Math.max(best, right - left + 1);
  }
  return best;
}
```

O(n) 时间（每个字符最多进出窗口一次）。

**口述模板（30–60 秒）**

1. 一句话结论：滑动窗口用于求满足条件的连续子串或子数组。
2. 关键机制：右指针扩张，条件不满足时左指针收缩。
3. 对比或注意点：左指针只能向右移动，所以是 O(n)。
4. 可追问时补充：这道题记录字符最后出现的位置，可以让左指针直接跳过去。

**常见追问**

- 为什么要判断 `prev >= left`？→ 例如 "abba"：处理最后的 a 时，第一个 a 已经在窗口外，不能让 left 回退。

---

### Q9. 前缀和是什么？

- **难度**: 初级
- **标签**: 前缀和

**答案**

`prefix[i]` = 前 i 个元素的和，则区间 `[l, r]` 的和 = `prefix[r + 1] - prefix[l]`，O(1) 查询。

**经典题：和为 K 的子数组个数**（有负数，不能用滑动窗口）：

```ts
function subarraySum(nums: number[], k: number): number {
  // 前缀和出现的次数；和为 0 的空前缀出现 1 次，用于统计从下标 0 开始的子数组
  const count = new Map<number, number>([[0, 1]]);
  let sum = 0;
  let result = 0;
  for (const x of nums) {
    sum += x;
    result += count.get(sum - k) ?? 0;
    count.set(sum, (count.get(sum) ?? 0) + 1);
  }
  return result;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：前缀和把区间求和变成两个前缀和相减。
2. 关键机制：子数组和为 k 等价于两个前缀和的差为 k。
3. 对比或注意点：有负数时窗口不具有单调性，不能用滑动窗口。
4. 可追问时补充：哈希表要初始化前缀和 0 出现一次。

**常见追问**

- 二维前缀和？→ `S[i][j] = S[i-1][j] + S[i][j-1] - S[i-1][j-1] + a[i][j]`，用于子矩阵求和。

---

### Q10. 有效的括号怎么判断？

- **难度**: 初级
- **标签**: 栈

**答案**

```ts
function isValid(s: string): boolean {
  if (s.length % 2 === 1) return false;
  const pairs: Record<string, string> = { ')': '(', ']': '[', '}': '{' };
  const stack: string[] = [];
  for (const ch of s) {
    if (ch in pairs) {
      if (stack.pop() !== pairs[ch]) return false;
    } else {
      stack.push(ch);
    }
  }
  return stack.length === 0;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：遇到左括号入栈，遇到右括号时弹出栈顶检查是否匹配。
2. 关键机制：栈的后进先出正好对应括号的嵌套关系。
3. 对比或注意点：最后要检查栈是否为空。
4. 可追问时补充：长度为奇数可以直接返回 false。

**常见追问**

- 只有一种括号时能 O(1) 空间吗？→ 用计数器代替栈，计数小于 0 时返回 false。

---

### Q11. 最长回文子串怎么求？

- **难度**: 初级
- **标签**: 字符串, 中心扩展

**答案**

**中心扩展**：回文以某个中心对称，中心有 2n - 1 个（单个字符或两个字符之间），从每个中心向两边扩展。O(n²) 时间，O(1) 空间。

```ts
function longestPalindrome(s: string): string {
  let start = 0;
  let maxLen = 0;
  const expand = (l: number, r: number) => {
    while (l >= 0 && r < s.length && s[l] === s[r]) {
      l--;
      r++;
    }
    // 循环结束时 l、r 已越过回文边界，实际长度是 r - l - 1
    if (r - l - 1 > maxLen) {
      maxLen = r - l - 1;
      start = l + 1;
    }
  };
  for (let i = 0; i < s.length; i++) {
    expand(i, i); // 奇数长度
    expand(i, i + 1); // 偶数长度
  }
  return s.slice(start, start + maxLen);
}
```

**口述模板（30–60 秒）**

1. 一句话结论：用中心扩展法，从每个可能的中心向两边扩展。
2. 关键机制：中心可以是一个字符（奇数长度）或两个字符之间（偶数长度）。
3. 对比或注意点：时间 O(n²)，空间 O(1)。
4. 可追问时补充：也可以用动态规划，或者用 Manacher 算法做到 O(n)。

**常见追问**

- 判断一个字符串是否是回文（忽略非字母数字）？→ 对撞双指针，跳过非字母数字字符，比较时统一小写。

---

## 三、链表

### Q12. 怎么反转链表？

- **难度**: 初级
- **标签**: 链表

**答案**

```ts
/** 迭代：O(n) 时间，O(1) 空间 */
function reverseList(head: ListNode | null): ListNode | null {
  let prev: ListNode | null = null;
  let cur = head;
  while (cur) {
    const next = cur.next; // 先保存，否则修改指针后就找不到后续节点
    cur.next = prev;
    prev = cur;
    cur = next;
  }
  return prev;
}

/** 递归：O(n) 空间（调用栈） */
function reverseListRec(head: ListNode | null): ListNode | null {
  if (!head || !head.next) return head;
  const newHead = reverseListRec(head.next);
  head.next.next = head;
  head.next = null;
  return newHead;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：用 prev 和 cur 两个指针逐个反转指向。
2. 关键机制：修改指针前先保存 next。
3. 对比或注意点：迭代是 O(1) 空间，递归需要 O(n) 的调用栈。
4. 可追问时补充：递归版本中，head.next.next = head 把后一个节点指回自己。

**常见追问**

- 反转链表的第 m 到 n 个节点？→ 用虚拟头节点找到 m 的前一个节点，然后用头插法反转区间。

---

### Q13. 怎么判断链表有环？找到环的入口？

- **难度**: 初级
- **标签**: 链表, 快慢指针

**答案**

**快慢指针**：慢指针每次一步，快指针每次两步；有环则必然相遇。

**入口**：相遇后，一个指针回到头部，两个指针都每次一步，再次相遇处就是入口。

```ts
function detectCycle(head: ListNode | null): ListNode | null {
  let slow = head;
  let fast = head;
  while (fast && fast.next) {
    slow = slow!.next;
    fast = fast.next.next;
    if (slow === fast) {
      let p = head;
      while (p !== slow) {
        p = p!.next;
        slow = slow!.next;
      }
      return p;
    }
  }
  return null;
}
```

**原因**：设头到入口距离 a，入口到相遇点 b，环长 c。相遇时快指针走了 2(a+b)，也等于 a+b+kc → a = kc - b，即从头走 a 步与从相遇点走 a 步会在入口相遇。

**口述模板（30–60 秒）**

1. 一句话结论：用快慢指针，有环则一定相遇。
2. 关键机制：相遇后一个指针回到头部，两者同速前进，再次相遇就是入口。
3. 对比或注意点：可以用数学推导证明 a = kc - b。
4. 可追问时补充：用 Set 记录访问过的节点也可以，但需要 O(n) 空间。

**常见追问**

- 找链表中点？→ 快慢指针，快指针到末尾时慢指针在中点。

---

### Q14. 怎么合并两个有序链表？

- **难度**: 初级
- **标签**: 链表

**答案**

```ts
function mergeTwoLists(a: ListNode | null, b: ListNode | null): ListNode | null {
  // 虚拟头节点：避免单独处理「结果链表为空时的第一个节点」
  const dummy = new ListNode(0);
  let tail = dummy;
  while (a && b) {
    if (a.val <= b.val) {
      tail.next = a;
      a = a.next;
    } else {
      tail.next = b;
      b = b.next;
    }
    tail = tail.next;
  }
  tail.next = a ?? b;
  return dummy.next;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：用虚拟头节点，每次取两个链表中较小的节点接到尾部。
2. 关键机制：一个链表用完后，把另一个剩余的部分直接接上。
3. 对比或注意点：虚拟头节点避免了对头节点的特殊处理。
4. 可追问时补充：合并 K 个有序链表可以用最小堆或两两分治合并。

**常见追问**

- `<=` 和 `<` 有区别吗？→ 用 `<=` 保证相等时先取 a，合并是稳定的。

---

### Q15. 怎么删除链表的倒数第 N 个节点？

- **难度**: 初级
- **标签**: 链表, 双指针

**答案**

快指针先走 n 步，然后快慢指针一起走，快指针到末尾时，慢指针在倒数第 n+1 个节点。

```ts
function removeNthFromEnd(head: ListNode | null, n: number): ListNode | null {
  // 虚拟头节点：要删除的是头节点时也能统一处理
  const dummy = new ListNode(0, head);
  let fast: ListNode | null = dummy;
  let slow: ListNode = dummy;
  for (let i = 0; i <= n; i++) fast = fast!.next;
  while (fast) {
    fast = fast.next;
    slow = slow.next!;
  }
  slow.next = slow.next!.next;
  return dummy.next;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：快指针先走 n 步，然后两个指针同时走。
2. 关键机制：快指针到达末尾时，慢指针正好在要删除节点的前一个。
3. 对比或注意点：用虚拟头节点处理删除头节点的情况。
4. 可追问时补充：只需要遍历一次。

**常见追问**

- 为什么快指针走 n+1 步？→ 从 dummy 出发，让慢指针停在被删除节点的前一个节点。

---

## 四、二叉树

### Q16. 二叉树的前中后序遍历（递归和迭代）？

- **难度**: 初级
- **标签**: 二叉树, 遍历

**答案**

```ts
class TreeNode {
  constructor(
    public val: number,
    public left: TreeNode | null = null,
    public right: TreeNode | null = null,
  ) {}
}

/** 递归：只需要调整 push 的位置 */
function preorder(root: TreeNode | null, out: number[] = []): number[] {
  if (!root) return out;
  out.push(root.val); // 前序；放在两个递归之间是中序，放在最后是后序
  preorder(root.left, out);
  preorder(root.right, out);
  return out;
}

/** 迭代前序：右子节点先入栈，保证左子节点先出栈 */
function preorderIter(root: TreeNode | null): number[] {
  const res: number[] = [];
  const stack = root ? [root] : [];
  while (stack.length) {
    const node = stack.pop()!;
    res.push(node.val);
    if (node.right) stack.push(node.right);
    if (node.left) stack.push(node.left);
  }
  return res;
}

/** 迭代中序：一路向左入栈，弹出时访问，再转向右子树 */
function inorderIter(root: TreeNode | null): number[] {
  const res: number[] = [];
  const stack: TreeNode[] = [];
  let cur = root;
  while (cur || stack.length) {
    while (cur) {
      stack.push(cur);
      cur = cur.left;
    }
    cur = stack.pop()!;
    res.push(cur.val);
    cur = cur.right;
  }
  return res;
}

/** 迭代后序：按「根 → 右 → 左」得到的序列反转即为「左 → 右 → 根」 */
function postorderIter(root: TreeNode | null): number[] {
  const res: number[] = [];
  const stack = root ? [root] : [];
  while (stack.length) {
    const node = stack.pop()!;
    res.push(node.val);
    if (node.left) stack.push(node.left);
    if (node.right) stack.push(node.right);
  }
  return res.reverse();
}
```

**口述模板（30–60 秒）**

1. 一句话结论：前中后序的区别是访问根节点的时机。
2. 关键机制：迭代版本用显式栈模拟递归。
3. 对比或注意点：中序迭代要先一路向左入栈，后序可以用「根右左」反转得到。
4. 可追问时补充：BST 的中序遍历是有序的。

**常见追问**

- 递归和迭代的复杂度？→ 都是 O(n) 时间、O(h) 空间；Morris 遍历可以做到 O(1) 空间。

---

### Q17. 二叉树的层序遍历？

- **难度**: 初级
- **标签**: 二叉树, BFS

**答案**

```ts
function levelOrder(root: TreeNode | null): number[][] {
  const res: number[][] = [];
  let level = root ? [root] : [];
  while (level.length) {
    res.push(level.map((n) => n.val));
    const next: TreeNode[] = [];
    for (const node of level) {
      if (node.left) next.push(node.left);
      if (node.right) next.push(node.right);
    }
    level = next;
  }
  return res;
}
```

用「当前层数组 → 下一层数组」代替队列，避免 `shift` 的 O(n) 开销，同时天然按层分组。

**口述模板（30–60 秒）**

1. 一句话结论：层序遍历是 BFS，按层处理节点。
2. 关键机制：每次处理当前层的所有节点，同时收集下一层。
3. 对比或注意点：用两个数组代替队列，避免 shift 的开销。
4. 可追问时补充：变体包括右视图、锯齿形遍历、每层最大值。

**常见追问**

- 二叉树的右视图？→ 层序遍历时取每层的最后一个节点。

---

### Q18. 二叉树的最大深度、是否平衡？

- **难度**: 初级
- **标签**: 二叉树, 递归

**答案**

**思路**：后序遍历，「子问题的返回值」组合出当前节点的答案。

```ts
function maxDepth(root: TreeNode | null): number {
  if (!root) return 0;
  return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}

/** 判断是否平衡：返回高度，不平衡时返回 -1 提前终止，避免重复计算高度 */
function isBalanced(root: TreeNode | null): boolean {
  const height = (node: TreeNode | null): number => {
    if (!node) return 0;
    const l = height(node.left);
    if (l === -1) return -1;
    const r = height(node.right);
    if (r === -1) return -1;
    return Math.abs(l - r) > 1 ? -1 : 1 + Math.max(l, r);
  };
  return height(root) !== -1;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：二叉树的递归题用「子树的返回值」组合出当前节点的答案。
2. 关键机制：最大深度等于左右子树深度的最大值加一。
3. 对比或注意点：判断平衡时在计算高度的同时判断，用 -1 表示不平衡，做到 O(n)。
4. 可追问时补充：如果先算高度再判断，会重复计算，变成 O(n log n) 到 O(n²)。

**常见追问**

- 二叉树的直径？→ 同样后序遍历，每个节点计算左深度 + 右深度，更新全局最大值。

---

### Q19. 二叉搜索树有什么性质？怎么验证？

- **难度**: 初级
- **标签**: BST

**答案**

**性质**：左子树所有节点 < 根 < 右子树所有节点；**中序遍历是升序**。查找、插入平均 O(log n)，退化成链时 O(n)。

```ts
/** 递归时传递上下界：只比较父子节点是不够的，左子树中的所有节点都必须小于根 */
function isValidBST(root: TreeNode | null, min = -Infinity, max = Infinity): boolean {
  if (!root) return true;
  if (root.val <= min || root.val >= max) return false;
  return isValidBST(root.left, min, root.val) && isValidBST(root.right, root.val, max);
}
```

**口述模板（30–60 秒）**

1. 一句话结论：BST 的左子树都小于根，右子树都大于根，中序遍历是升序。
2. 关键机制：验证时递归传递上下界。
3. 对比或注意点：只比较父子节点是错误的，要保证整棵子树都满足。
4. 可追问时补充：也可以中序遍历，检查是否严格递增。

**常见追问**

- BST 中第 K 小的元素？→ 中序遍历到第 k 个节点。

---

### Q20. 翻转二叉树和判断对称二叉树？

- **难度**: 初级
- **标签**: 二叉树

**答案**

```ts
function invertTree(root: TreeNode | null): TreeNode | null {
  if (!root) return null;
  [root.left, root.right] = [invertTree(root.right), invertTree(root.left)];
  return root;
}

function isSymmetric(root: TreeNode | null): boolean {
  const mirror = (a: TreeNode | null, b: TreeNode | null): boolean => {
    if (!a || !b) return a === b;
    return a.val === b.val && mirror(a.left, b.right) && mirror(a.right, b.left);
  };
  return mirror(root?.left ?? null, root?.right ?? null);
}
```

**口述模板（30–60 秒）**

1. 一句话结论：翻转是交换每个节点的左右子树，对称是比较左右子树是否镜像。
2. 关键机制：镜像比较时，左的左对应右的右，左的右对应右的左。
3. 对比或注意点：两个节点都为空时相等，只有一个为空时不相等。
4. 可追问时补充：都可以用 BFS 迭代实现。

**常见追问**

- 判断两棵树是否相同？→ 同样递归比较，但左对左、右对右。

---

## 五、排序与查找

### Q21. 常见排序算法怎么比较？

- **难度**: 初级
- **标签**: 排序

**答案**

| 算法 | 平均 | 最坏 | 空间 | 稳定 |
|------|------|------|------|------|
| 冒泡 / 插入 | O(n²) | O(n²) | O(1) | ✅ |
| 选择 | O(n²) | O(n²) | O(1) | ❌ |
| **快速排序** | O(n log n) | O(n²) | O(log n) | ❌ |
| **归并排序** | O(n log n) | O(n log n) | O(n) | ✅ |
| 堆排序 | O(n log n) | O(n log n) | O(1) | ❌ |
| 计数 / 桶 / 基数 | O(n + k) | | O(n + k) | ✅ |

- **稳定**：相等元素的相对顺序不变（例如先按时间排序，再按分数排序时需要稳定）
- 插入排序在**近乎有序**的数据上接近 O(n)
- 基于比较的排序下界是 O(n log n)

**口述模板（30–60 秒）**

1. 一句话结论：常用的 O(n log n) 排序是快排、归并和堆排序。
2. 关键机制：快排平均最快但最坏 O(n²)，归并稳定但需要 O(n) 空间。
3. 对比或注意点：稳定性在多关键字排序时很重要。
4. 可追问时补充：基于比较的排序下界是 O(n log n)，计数排序等非比较排序可以突破。

**常见追问**

- 什么时候用计数排序？→ 值域范围小（例如分数 0–100、年龄）时。

---

### Q22. 手写快速排序

- **难度**: 初级
- **标签**: 排序, 分治

**答案**

```ts
/** 原地快排：随机选择基准，避免有序输入退化为 O(n²) */
function quickSort(arr: number[], lo = 0, hi = arr.length - 1): number[] {
  if (lo >= hi) return arr;
  const p = partition(arr, lo, hi);
  quickSort(arr, lo, p - 1);
  quickSort(arr, p + 1, hi);
  return arr;
}

function partition(arr: number[], lo: number, hi: number): number {
  const r = lo + Math.floor(Math.random() * (hi - lo + 1));
  [arr[r], arr[hi]] = [arr[hi], arr[r]];
  const pivot = arr[hi];
  let i = lo; // [lo, i) 是小于 pivot 的区域
  for (let j = lo; j < hi; j++) {
    if (arr[j] < pivot) {
      [arr[i], arr[j]] = [arr[j], arr[i]];
      i++;
    }
  }
  [arr[i], arr[hi]] = [arr[hi], arr[i]];
  return i;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：快排选一个基准，把数组分成小于和大于基准的两部分，再递归排序。
2. 关键机制：partition 用一个指针维护小于基准的区域。
3. 对比或注意点：随机选择基准，避免有序数组退化为 O(n²)。
4. 可追问时补充：大量重复元素时用三路快排，把等于基准的元素放在中间。

**常见追问**

- 快排为什么实践中比归并快？→ 原地操作、缓存友好、常数小。

---

### Q23. 手写归并排序

- **难度**: 初级
- **标签**: 排序, 分治

**答案**

```ts
function mergeSort(arr: number[]): number[] {
  if (arr.length <= 1) return arr;
  const mid = arr.length >> 1;
  const left = mergeSort(arr.slice(0, mid));
  const right = mergeSort(arr.slice(mid));
  const res: number[] = [];
  let i = 0, j = 0;
  while (i < left.length && j < right.length) {
    // <= 保证相等时先取左边，排序稳定
    res.push(left[i] <= right[j] ? left[i++] : right[j++]);
  }
  while (i < left.length) res.push(left[i++]);
  while (j < right.length) res.push(right[j++]);
  return res;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：归并排序先递归排序两半，再合并两个有序数组。
2. 关键机制：每层合并 O(n)，共 log n 层，总共 O(n log n)。
3. 对比或注意点：需要 O(n) 额外空间，是稳定排序。
4. 可追问时补充：链表排序适合用归并，因为不需要随机访问。

**常见追问**

- 归并排序的应用？→ 求逆序对（合并时统计）、链表排序、外部排序（大文件）。

---

### Q24. 二分查找怎么写才不出错？

- **难度**: 初级
- **标签**: 二分查找

**答案**

**统一使用「左闭右开」写法找下界**（第一个 ≥ target 的位置）：

```ts
/** 返回第一个 >= target 的下标；所有元素都小于 target 时返回 nums.length */
function lowerBound(nums: number[], target: number): number {
  let lo = 0;
  let hi = nums.length; // 区间 [lo, hi)
  while (lo < hi) {
    const mid = lo + ((hi - lo) >> 1);
    if (nums[mid] < target) lo = mid + 1;
    else hi = mid;
  }
  return lo;
}

function search(nums: number[], target: number): number {
  const i = lowerBound(nums, target);
  return i < nums.length && nums[i] === target ? i : -1;
}

/** 最后一个 <= target 的位置 = 第一个 > target 的位置 - 1 = lowerBound(target + 1) - 1（整数时） */
```

**要点**：区间定义一致（循环条件、更新方式都由区间定义决定）；`mid` 计算避免溢出（JS 中数字大时也建议这样写）；确认循环一定能缩小区间。

**口述模板（30–60 秒）**

1. 一句话结论：二分查找的关键是保持区间定义的一致性。
2. 关键机制：左闭右开时，循环条件是 lo < hi，右边界更新为 mid。
3. 对比或注意点：用下界函数统一处理查找、左边界、右边界。
4. 可追问时补充：二分不只用于有序数组，只要答案具有单调性就能二分。

**常见追问**

- 查找目标的第一个和最后一个位置？→ `lowerBound(target)` 和 `lowerBound(target + 1) - 1`。

---

### Q25. JS 的 `Array.prototype.sort` 是怎么实现的？

- **难度**: 初级
- **标签**: 排序, JS

**答案**

- V8（Chrome 70+）使用 **TimSort**：归并排序 + 插入排序的混合，利用数据中已有的有序片段（run）；**稳定**；O(n log n)
- ES2019 起规范要求 `sort` 是**稳定的**
- **默认按字符串比较**：`[10, 9, 1].sort()` → `[1, 10, 9]` → 数字排序必须传比较函数 `(a, b) => a - b`
- 比较函数要返回数字（负数、0、正数），返回布尔值会导致不正确的结果
- 原地排序；ES2023 新增不修改原数组的 `toSorted`

**口述模板（30–60 秒）**

1. 一句话结论：V8 的 sort 使用 TimSort，是稳定的 O(n log n) 排序。
2. 关键机制：TimSort 结合了归并排序和插入排序，利用已有的有序片段。
3. 对比或注意点：默认按字符串比较，数字排序必须传比较函数。
4. 可追问时补充：toSorted 不修改原数组。

**常见追问**

- 比较函数返回 `a > b` 有什么问题？→ 返回布尔值，`false` 被当作 0（相等），排序结果不正确。

---

## 六、基础算法思想

### Q26. 递归怎么写？怎么避免栈溢出？

- **难度**: 初级
- **标签**: 递归

**答案**

**三要素**：

1. **函数定义**：明确函数的输入和返回值的含义
2. **终止条件**：最小的子问题直接返回
3. **递推关系**：用子问题的结果构造当前结果（相信函数定义，不要展开每一层）

**避免栈溢出**：

- 改为迭代 + 显式栈
- 记忆化减少重复调用（但深度不变）
- JS 引擎（除 Safari 外）基本不支持尾调用优化，不能依赖

**口述模板（30–60 秒）**

1. 一句话结论：写递归要先明确函数定义、终止条件和递推关系。
2. 关键机制：相信函数的定义，用子问题的结果构造当前结果。
3. 对比或注意点：递归深度过大会栈溢出，要改成迭代。
4. 可追问时补充：大部分 JS 引擎不支持尾调用优化。

**常见追问**

- 斐波那契递归为什么慢？→ 大量重复计算，O(2ⁿ) → 记忆化或迭代 O(n)。

---

### Q27. DFS 和 BFS 有什么区别？什么时候用哪个？

- **难度**: 初级
- **标签**: DFS, BFS

**答案**

| | DFS | BFS |
|--|-----|-----|
| 数据结构 | 栈 / 递归 | 队列 |
| 顺序 | 一条路走到底再回退 | 一层一层扩展 |
| 适用 | 所有路径、连通性、回溯、拓扑排序 | **无权图最短路径**、层序、最少步数 |
| 空间 | O(深度) | O(最宽的一层) |

**网格 DFS 模板（岛屿数量）**：

```ts
function numIslands(grid: string[][]): number {
  const m = grid.length, n = grid[0].length;
  let count = 0;
  const sink = (i: number, j: number) => {
    if (i < 0 || j < 0 || i >= m || j >= n || grid[i][j] !== '1') return;
    grid[i][j] = '0'; // 原地标记已访问
    sink(i + 1, j); sink(i - 1, j); sink(i, j + 1); sink(i, j - 1);
  };
  for (let i = 0; i < m; i++) {
    for (let j = 0; j < n; j++) {
      if (grid[i][j] === '1') {
        count++;
        sink(i, j);
      }
    }
  }
  return count;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：DFS 一条路走到底，BFS 一层一层扩展。
2. 关键机制：DFS 用栈或递归，BFS 用队列。
3. 对比或注意点：求无权图的最短路径和最少步数要用 BFS。
4. 可追问时补充：网格题要标记已访问，避免重复访问。

**常见追问**

- 网格很大时递归 DFS 会栈溢出吗？→ 可能（例如 1000×1000 全是陆地）→ 改用 BFS 或迭代 DFS。

---

### Q28. 贪心算法是什么？

- **难度**: 初级
- **标签**: 贪心

**答案**

每一步都做**当前看起来最好**的选择，期望得到全局最优。**适用前提**：局部最优能推出全局最优（需要证明或举反例验证）。

**例：买卖股票的最佳时机 II**（可以多次交易）：所有上涨的差价都赚到。

```ts
function maxProfit(prices: number[]): number {
  let profit = 0;
  for (let i = 1; i < prices.length; i++) {
    profit += Math.max(0, prices[i] - prices[i - 1]);
  }
  return profit;
}
```

**例：跳跃游戏**：维护能到达的最远位置。

```ts
function canJump(nums: number[]): boolean {
  let farthest = 0;
  for (let i = 0; i < nums.length; i++) {
    if (i > farthest) return false;
    farthest = Math.max(farthest, i + nums[i]);
  }
  return true;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：贪心每一步选择当前最优，期望得到全局最优。
2. 关键机制：前提是局部最优能推出全局最优。
3. 对比或注意点：不确定时要尝试举反例，贪心不成立时用动态规划。
4. 可追问时补充：常见题型有区间调度、跳跃游戏、分发饼干。

**常见追问**

- 零钱兑换能用贪心吗？→ 一般不能，例如硬币 [1, 3, 4] 凑 6：贪心得 4+1+1，最优是 3+3 → 用 DP。

---

### Q29. 动态规划入门：爬楼梯

- **难度**: 初级
- **标签**: 动态规划

**答案**

每次爬 1 或 2 级，到第 n 级有多少种方法。

**步骤**：

1. **状态**：`dp[i]` = 到第 i 级的方法数
2. **转移**：`dp[i] = dp[i-1] + dp[i-2]`（最后一步爬 1 级或 2 级）
3. **初始化**：`dp[0] = 1, dp[1] = 1`
4. **顺序**：从小到大

```ts
function climbStairs(n: number): number {
  // dp[i] 只依赖前两项，用两个变量代替数组，空间 O(1)
  let prev = 1, cur = 1;
  for (let i = 2; i <= n; i++) {
    [prev, cur] = [cur, prev + cur];
  }
  return cur;
}
```

**DP 与递归的关系**：自顶向下的记忆化递归 = 自底向上的 DP；DP 的核心是**重叠子问题**和**最优子结构**。

**口述模板（30–60 秒）**

1. 一句话结论：动态规划把问题拆成重叠的子问题，保存子问题的结果避免重复计算。
2. 关键机制：步骤是定义状态、写出转移方程、初始化、确定遍历顺序。
3. 对比或注意点：只依赖前几项时可以用变量代替数组，压缩空间。
4. 可追问时补充：记忆化递归和自底向上的 DP 本质相同。

**常见追问**

- 每次可以爬 1、2、3 级呢？→ `dp[i] = dp[i-1] + dp[i-2] + dp[i-3]`。

---

### Q30. 刷算法题有什么方法？面试时怎么作答？

- **难度**: 初级
- **标签**: 方法

**答案**

**刷题方法**：

- **按题型刷**（双指针、滑动窗口、二叉树、回溯、DP），总结每类的模板
- 以 **Hot 100** 为主，中等题为重点
- 每道题：先想 10–15 分钟 → 看题解 → 自己重写 → 隔几天再写一次
- 记录错题和「卡住的点」

**面试作答流程**：

1. **澄清**：输入范围、是否有重复、是否有序、空输入、返回什么
2. **举例**：用一两个例子确认理解
3. **说思路**：先说暴力解和复杂度，再说优化
4. **编码**：变量命名清晰，边写边解释
5. **测试**：用例子走一遍；检查边界（空、单元素、全相同、负数）
6. **复杂度**：主动说明

**口述模板（30–60 秒）**

1. 一句话结论：刷题按题型总结模板，面试时按「澄清、思路、编码、测试」作答。
2. 关键机制：先说暴力解，再逐步优化。
3. 对比或注意点：写完要主动用例子和边界测试。
4. 可追问时补充：卡住时说出思考过程，面试官可能会给提示。

**常见追问**

- 想不出最优解怎么办？→ 先写出正确的暴力解，再分析瓶颈在哪里，通常比什么都写不出来好。
