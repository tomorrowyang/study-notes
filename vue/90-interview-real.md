# Vue 真实面试题

> 根据近年公开面经（社区面经帖、技术博客、面试复盘）整理的高频题。`来源倾向`表示这类题常见于哪类岗位或面试轮次，不代表某公司的固定题库。每题给出简短答案，并链接到详细讲解。

---

### R1. Vue 3 的响应式原理是什么？为什么用 Proxy 替代 defineProperty？

- **来源倾向**: 几乎所有 Vue 岗位，一面必问

**答案**：Proxy 拦截读写，get 时 track 收集当前副作用，set 时 trigger 调度依赖的副作用；依赖存储为 WeakMap → Map → Dep。Proxy 能检测新增删除属性、数组索引，支持 Map 和 Set，并且是懒代理，初始化更快。

**口述模板**：

1. **机制**：Proxy 拦截读写，读取时收集依赖，修改时触发更新。
2. **结构**：WeakMap 以对象为键，Map 以属性为键，Dep 存储依赖它的副作用。
3. **对比**：defineProperty 检测不到新增删除属性和数组索引，初始化需要递归。
4. **演进**：3.4、3.5 优化了依赖结构，3.6 基于 alien-signals 重构。

详见 [中级 Q1–Q2](20-mid-qa.md)、[手写 H1](40-coding-handwritten.md)。

---

### R2. `ref` 和 `reactive` 的区别？为什么 ref 需要 `.value`？

- **来源倾向**: 初中级高频

**答案**：ref 适用于任意类型，reactive 只适用于对象；reactive 整体替换和解构会丢失响应式。基本类型无法被 Proxy 代理，需要包装为对象，通过 value 的 getter / setter 追踪。模板中顶层 ref 自动解包。

详见 [初级 Q8](10-junior-qa.md)、[中级 Q3](20-mid-qa.md)。

---

### R3. `computed` 和 `watch` 的区别？computed 是怎么缓存的？

- **来源倾向**: 初中级高频

**答案**：computed 派生值，有缓存；watch 执行副作用。computed 惰性求值，依赖变化时只标记为脏并通知订阅者，被访问时才重新计算；3.4 之后结果不变时不触发下游。

详见 [初级 Q6](10-junior-qa.md)、[中级 Q4](20-mid-qa.md)、[手写 H2](40-coding-handwritten.md)。

---

### R4. `nextTick` 的原理是什么？

- **来源倾向**: 中级高频

**答案**：数据变化后，组件更新任务进入去重的队列，通过 `Promise.resolve().then(flushJobs)` 在微任务中批量执行。nextTick 返回当前刷新的 Promise，回调在 DOM 更新后执行。Vue 2 有多级降级，Vue 3 只用 Promise。

**口述模板**：

1. **异步更新**：数据变化不立即更新 DOM，任务进入去重的队列。
2. **微任务**：用 Promise.then 安排一次刷新，同一 tick 内的修改合并。
3. **nextTick**：链在当前刷新的 Promise 之后。
4. **原因**：微任务在浏览器渲染前执行，用户看不到中间状态。

详见 [中级 Q10–Q11](20-mid-qa.md)、[手写 H4](40-coding-handwritten.md)。

---

### R5. Vue 3 的 diff 算法相比 Vue 2 有什么优化？

- **来源倾向**: 中高级高频

**答案**：Vue 2 使用双端比较；Vue 3 先处理首尾相同的节点，再对乱序部分用最长递增子序列最小化移动。更重要的是编译优化：PatchFlag 只比较动态部分，Block Tree 只遍历动态节点，静态提升跳过静态节点。

**口述模板**：

1. **算法**：首尾预处理，中间部分建 key 映射，用最长递增子序列确定不需要移动的节点。
2. **编译优化**：PatchFlag 标记动态属性，Block Tree 收集动态节点为扁平数组。
3. **结果**：diff 的工作量从模板大小降到动态节点数量。
4. **对比**：Vue 2 需要遍历整棵树，双端比较不能保证最少移动。

详见 [中级 Q13–Q14、Q17](20-mid-qa.md)、[手写 H5–H6](40-coding-handwritten.md)。

---

### R6. `v-for` 中 key 的作用？为什么不能用 index？

- **来源倾向**: 初中级高频

**答案**：key 是 diff 判断节点身份的依据。用 index 时插入或删除会导致 key 与数据错位，Vue 就地更新文本，但输入框和组件内部状态留在原位置，造成状态错乱，而且所有节点都要更新。

详见 [初级 Q5](10-junior-qa.md)。

---

### R7. Vue 3 的编译器做了哪些优化？

- **来源倾向**: 中高级

**答案**：静态提升（静态节点提升到渲染函数外，连续的合并为字符串）、PatchFlag、Block Tree、事件处理函数缓存、`v-once` / `v-memo`。`<script setup>` 的模板内联在 setup 中，通过闭包直接访问变量。

详见 [中级 Q16–Q19](20-mid-qa.md)、[高级 Q21](30-senior-qa.md)。

---

### R8. 组件通信有哪些方式？

- **来源倾向**: 初级高频

**答案**：props / emit、v-model、ref + defineExpose、插槽、$attrs、provide / inject、Pinia、路由参数；事件总线不推荐。

详见 [初级 Q11](10-junior-qa.md)。

---

### R9. 父子组件的生命周期执行顺序？

- **来源倾向**: 初级高频

**答案**：挂载：父 beforeMount → 子 beforeMount → 子 mounted → 父 mounted。更新：父 beforeUpdate → 子 beforeUpdate → 子 updated → 父 updated。卸载：父 beforeUnmount → 子 beforeUnmount → 子 unmounted → 父 unmounted。

详见 [初级 Q9](10-junior-qa.md)。

---

### R10. `KeepAlive` 的原理是什么？

- **来源倾向**: 中级高频

**答案**：内部用 Map 缓存组件 VNode，用 Set 的顺序实现 LRU；卸载时不销毁，而是把 DOM 移到隐藏的容器中（deactivate），再次显示时移回（activate）并触发 onActivated。include / exclude 按组件 name 匹配。

详见 [初级 Q22](10-junior-qa.md)、[中级 Q20](20-mid-qa.md)。

---

### R11. Composition API 和 Options API 的区别？和 React Hooks 有什么不同？

- **来源倾向**: 中级高频

**答案**：Composition API 按功能组织代码，通过 composable 复用逻辑，解决了 mixin 的命名冲突和来源不清问题，类型推导更好。与 Hooks 相比，setup 只执行一次，没有调用顺序限制和过期闭包问题。

详见 [初级 Q18](10-junior-qa.md)、[中级 Q29](20-mid-qa.md)。

---

### R12. Vue 和 React 的区别？

- **来源倾向**: 中高级高频

**答案**：Vue 是响应式驱动的精确更新，React 是状态变化后重新渲染；Vue 的优化基本自动，React 需要 memo 或 React Compiler；Vue 使用可变数据和模板，React 使用不可变数据和 JSX；React 有并发渲染。

**口述模板**：

1. **更新机制**：Vue 自动追踪依赖，只更新相关组件；React 重新执行组件函数。
2. **优化**：Vue 子组件只在 props 变化时更新；React 需要手动或编译器 memo。
3. **心智模型**：Vue 的 setup 只执行一次；Hooks 每次渲染都执行。
4. **趋同**：React Compiler 自动 memo，Vue Vapor Mode 走向更细粒度的编译。

详见 [中级 Q29](20-mid-qa.md)。

---

### R13. Vue Router 的 hash 和 history 模式的区别和原理？

- **来源倾向**: 初中级高频

**答案**：hash 模式监听 hashchange，不需要服务器配置；history 模式使用 pushState 和 popstate，需要服务器把所有路径回退到 index.html。pushState 不触发 popstate，路由器需要自己更新状态。

详见 [初级 Q27](10-junior-qa.md)、[手写 H7](40-coding-handwritten.md)。

---

### R14. 导航守卫的执行顺序？怎么做权限控制？

- **来源倾向**: 中级

**答案**：离开守卫 → 全局 beforeEach → beforeRouteUpdate → beforeEnter → 解析异步组件 → beforeRouteEnter → beforeResolve → 确认 → afterEach。权限：动态路由、守卫校验、按钮权限组件，接口由后端鉴权。

详见 [初级 Q28](10-junior-qa.md)、[中级 Q25、Q27](20-mid-qa.md)。

---

### R15. Pinia 和 Vuex 的区别？Pinia 的原理？

- **来源倾向**: 中级

**答案**：Pinia 没有 mutation，每个 store 独立，类型推导完整，体积小。原理：首次调用 useStore 时在独立的 effectScope 中执行 setup，包装为 reactive 并缓存为单例；状态集中在 pinia.state 中，便于 SSR 序列化。

详见 [初级 Q25](10-junior-qa.md)、[中级 Q26](20-mid-qa.md)、[手写 H8](40-coding-handwritten.md)。

---

### R16. 哪些情况会丢失响应式？

- **来源倾向**: 中级

**答案**：解构 reactive 对象或 props（3.5 之前）、整体替换 reactive 变量、解构 Pinia store 的状态、`watch(state.count)` 传入值而不是 getter。解决：toRefs、storeToRefs、使用 ref、getter 函数。

详见 [中级 Q8](20-mid-qa.md)。

---

### R17. `v-if` 和 `v-for` 的优先级？

- **来源倾向**: 初级

**答案**：Vue 3 中 v-if 优先级高于 v-for（Vue 2 相反），v-if 访问不到 v-for 的循环变量。推荐用计算属性先过滤，或者把 v-if 放到外层的 template 上。

详见 [初级 Q5](10-junior-qa.md)。

---

### R18. Vue 3.5 有哪些新特性？

- **来源倾向**: 中高级，考察是否跟进版本

**答案**：响应式内存减少约 56%、响应式 props 解构、useTemplateRef、useId、懒 hydration、onWatcherCleanup、Teleport defer、data-allow-mismatch。

详见 [中级 Q32](20-mid-qa.md)。

---

### R19. Vapor Mode 是什么？

- **来源倾向**: 高级，考察技术视野

**答案**：Vue 3.6 引入的编译模式（截至 2026-10 处于 RC），模板编译为直接操作 DOM 的代码，跳过虚拟 DOM；每个动态绑定是一个细粒度的 effect。通过 `<script setup vapor>` 选择性启用，可与虚拟 DOM 组件混用，只支持 API 子集。

**口述模板**：

1. **定义**：无虚拟 DOM 的编译模式。
2. **原理**：静态结构克隆模板创建，动态绑定各自是一个 effect，数据变化时直接更新对应 DOM。
3. **优势**：没有 VNode 和 diff 的开销，运行时更小，性能接近 Solid。
4. **采用**：3.6 稳定前用于已测量到瓶颈的页面或新的小型应用。

详见 [高级 Q5–Q6](30-senior-qa.md)。

---

### R20. 什么是 SSR？hydration mismatch 怎么排查？

- **来源倾向**: 中高级，Nuxt 方向

**答案**：服务端渲染 HTML 和状态，客户端 hydration 复用 DOM 并绑定事件。mismatch 常见原因：时间随机数、浏览器专属 API、不合法的 HTML 嵌套、ID 不一致。解决：放到 onMounted、ClientOnly、useId、data-allow-mismatch。

详见 [中级 Q30](20-mid-qa.md)、[高级 Q9](30-senior-qa.md)。

---

### R21. 大列表怎么优化？

- **来源倾向**: 中高级，场景题

**答案**：数据使用 shallowRef；虚拟滚动只渲染可见行；行组件拆分；v-memo 跳过未变化的行；计算放到 Worker；超大规模使用 Canvas。

详见 [高级 Q12](30-senior-qa.md)、[手写 H10](40-coding-handwritten.md)。

---

### R22. 手写一个简易的响应式系统

- **来源倾向**: 中高级，现场编码

**答案**：Proxy 的 get 中 track、set 中 trigger；WeakMap → Map → Set 存储依赖；effect 执行时设置 activeEffect，用栈处理嵌套；执行前清理旧依赖解决分支切换问题。

详见 [手写 H1](40-coding-handwritten.md)。

---

### R23. `scoped` 样式的原理？怎么修改子组件的样式？

- **来源倾向**: 初级

**答案**：给元素添加 `data-v-hash` 属性，CSS 选择器加属性选择器。子组件根元素同时带有父组件的 hash。修改内部元素用 `:deep()`，插槽内容用 `:slotted()`。

详见 [初级 Q29](10-junior-qa.md)。

---

### R24. Vue 2 项目怎么迁移到 Vue 3？

- **来源倾向**: 高级，项目经验

**答案**：评估依赖兼容性（组件库是最大成本）；先升级到 2.7 使用 Composition API；用 @vue/compat 兼容构建逐步修复；大型项目用微前端渐进迁移；构建工具换为 Vite。

详见 [高级 Q17](30-senior-qa.md)。

---

### R25. 你在 Vue 项目中做过哪些性能优化？

- **来源倾向**: 中高级，项目深挖

**答案**：按「问题 → 定位 → 方案 → 结果」讲。加载：路由懒加载、按需引入、图片优化；渲染：虚拟列表、组件拆分、v-memo、KeepAlive；响应式：shallowRef、markRaw；SSR：懒 hydration。用 Vue DevTools 和 onRenderTriggered 定位，用指标证明效果。

**口述模板**：

1. **问题**：某页面切换筛选条件时卡顿约 1 秒。
2. **定位**：DevTools 发现整个表格组件重新渲染，onRenderTriggered 显示是深度代理的大数组触发。
3. **方案**：数据改为 shallowRef，行组件拆分，引入虚拟滚动。
4. **结果**：交互延迟从约 1 秒降到 100ms 以内，用 INP 数据验证。

详见 [初级 Q30](10-junior-qa.md)、[高级 Q12–Q14](30-senior-qa.md)。
