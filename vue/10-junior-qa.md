# Vue 初级 Q&A

> 目标：熟练使用 Vue 3 的模板语法、组件、Composition API、内置组件、Pinia 和 Vue Router，能说清每个 API 的作用和常见坑。示例默认使用 `<script setup>` + TypeScript。

---

## 一、基础

### Q1. Vue 是什么？核心特点有哪些？

- **难度**: 初级
- **标签**: 基础

**答案**

Vue 是一个用于构建用户界面的**渐进式**框架。

- **声明式渲染**：用模板描述 UI 与状态的关系，状态变化时自动更新 DOM
- **响应式**：自动追踪数据依赖，数据变化时精确更新依赖它的视图
- **组件化**：单文件组件（SFC，`.vue`）把模板、逻辑、样式放在一起
- **渐进式**：可以只用核心库增强页面，也可以搭配 Router、Pinia、Nuxt 构建完整应用
- **编译时 + 运行时**：模板在构建时编译为渲染函数，编译器做了大量优化

**口述模板（30–60 秒）**

1. 一句话结论：Vue 是渐进式的前端框架，核心是声明式渲染和响应式系统。
2. 关键机制：模板编译为渲染函数，响应式系统追踪依赖，数据变化时自动更新。
3. 对比或注意点：与 React 相比，Vue 是细粒度响应式，React 是状态变化后重新渲染组件。
4. 可追问时补充：Vue 的编译器知道模板的结构，可以做静态优化。

**常见追问**

- 「渐进式」是什么意思？→ 可以按需逐步引入能力：从在页面中使用，到组件化，再到路由、状态管理、SSR。

---

### Q2. Vue 3 相比 Vue 2 有哪些主要变化？

- **难度**: 初级
- **标签**: 基础, 版本

**答案**

| 方面 | Vue 2 | Vue 3 |
|------|-------|-------|
| 响应式 | `Object.defineProperty` | `Proxy` |
| API | Options API | Composition API + Options API |
| 性能 | — | 编译优化（PatchFlag、Block Tree、静态提升），更新更快 |
| 体积 | 整体引入 | 支持 Tree-shaking |
| 根节点 | 必须单根 | 支持多根（Fragment） |
| 新组件 | — | Teleport、Suspense |
| TypeScript | 支持一般 | 源码用 TS 重写，类型推导好 |
| 创建应用 | `new Vue()` | `createApp()`，避免全局配置污染 |
| v-model | `value` + `input` | `modelValue` + `update:modelValue`，支持多个 v-model |
| 移除 | — | 过滤器（filter）、`$on` / `$off`、`.sync` |

Vue 2 已于 2023-12-31 停止维护。

**口述模板（30–60 秒）**

1. 一句话结论：Vue 3 在响应式、API 设计、性能和 TypeScript 支持上全面升级。
2. 关键机制：Proxy 响应式，Composition API 更好地复用逻辑，编译优化让更新更快。
3. 对比或注意点：移除了过滤器、事件总线 API，v-model 的默认 prop 和事件名变了。
4. 可追问时补充：Vue 2 已停止维护，存量项目需要规划迁移。

**常见追问**

- 为什么移除 `$on` / `$off`？→ 事件总线难以追踪数据流，可以使用 mitt 等库，或者用 Pinia、provide / inject 替代。

---

### Q3. 常用的指令有哪些？

- **难度**: 初级
- **标签**: 模板

**答案**

| 指令 | 作用 |
|------|------|
| `v-bind`（`:`） | 绑定属性 |
| `v-on`（`@`） | 绑定事件，支持修饰符 `.stop`、`.prevent`、`.once`、`.self`、`.passive`、按键修饰符 |
| `v-model` | 双向绑定，修饰符 `.lazy`、`.number`、`.trim` |
| `v-if` / `v-else-if` / `v-else` | 条件渲染 |
| `v-show` | 通过 `display` 切换显示 |
| `v-for` | 列表渲染 |
| `v-slot`（`#`） | 插槽 |
| `v-html` | 插入 HTML（有 XSS 风险） |
| `v-text` | 设置文本内容 |
| `v-once` | 只渲染一次 |
| `v-memo` | 依赖不变时跳过更新 |
| `v-pre` | 跳过编译 |
| `v-cloak` | 编译完成前隐藏（配合 CSS） |

**口述模板（30–60 秒）**

1. 一句话结论：常用的指令包括绑定、事件、双向绑定、条件、列表和插槽。
2. 关键机制：指令在编译时被转换为渲染函数中的代码。
3. 对比或注意点：v-html 有 XSS 风险，不能用于用户输入。
4. 可追问时补充：v-once 和 v-memo 用于性能优化。

**常见追问**

- 怎么写自定义指令？→ 定义包含 `mounted`、`updated`、`unmounted` 等钩子的对象，例如自动聚焦、权限控制。

---

### Q4. `v-if` 和 `v-show` 有什么区别？

- **难度**: 初级
- **标签**: 模板

**答案**

| 对比项 | `v-if` | `v-show` |
|--------|--------|----------|
| 实现 | 条件为假时不渲染（销毁 / 创建元素和组件） | 始终渲染，切换 `display: none` |
| 初始开销 | 条件为假时没有开销 | 总是渲染 |
| 切换开销 | 高（创建、销毁，触发生命周期） | 低 |
| 适用 | 很少切换、条件初始为假 | 频繁切换 |
| 支持 `<template>` | 支持 | 不支持 |

**口述模板（30–60 秒）**

1. 一句话结论：v-if 是真正的条件渲染，v-show 只切换 CSS。
2. 关键机制：v-if 会创建和销毁组件，触发生命周期；v-show 只修改 display。
3. 对比或注意点：频繁切换用 v-show，很少切换用 v-if。
4. 可追问时补充：v-show 不能用于 template，也不支持 v-else。

**常见追问**

- `v-if` 切换时组件状态会保留吗？→ 不会，组件被销毁；需要保留时用 v-show 或 KeepAlive。

---

### Q5. `v-for` 为什么要加 `key`？为什么不推荐用 index？

- **难度**: 初级
- **标签**: 模板, 列表

**答案**

**key 的作用**：作为节点的唯一标识，diff 时判断新旧节点是否是同一个节点，从而**复用**和**移动**节点，而不是就地修改。

**用 index 的问题**：

```vue
<!-- 在列表头部插入一项后，所有项的 index 都变了 -->
<li v-for="(item, index) in list" :key="index">
  <input />{{ item.name }}
</li>
```

- 头部插入、删除、排序时，index 与数据的对应关系改变
- Vue 认为 key 相同的节点是同一个，于是**就地更新**文本，但 input 中输入的内容（DOM 状态）和组件内部状态留在原位置，导致**状态错位**
- 所有节点都要更新，性能更差

**规则**：使用数据中稳定且唯一的 ID 作为 key；列表只追加且没有状态时，index 也可以接受。

**口述模板（30–60 秒）**

1. 一句话结论：key 是 diff 时判断节点身份的依据，用于正确地复用和移动节点。
2. 关键机制：key 相同就认为是同一个节点，就地更新。
3. 对比或注意点：用 index 时插入或删除会导致 key 和数据错位，输入框和组件状态出错。
4. 可追问时补充：应该用数据的唯一 ID。

**常见追问**

- `v-if` 和 `v-for` 能一起用吗？→ Vue 3 中 v-if 优先级高于 v-for（与 Vue 2 相反），v-if 访问不到 v-for 的变量；应该用计算属性过滤，或者把 v-if 放在外层 template 上。

---

### Q6. `computed` 和 `watch` 有什么区别？

- **难度**: 初级
- **标签**: 响应式

**答案**

| 对比项 | `computed` | `watch` |
|--------|-----------|---------|
| 用途 | 根据已有状态**派生**新的值 | 状态变化时执行**副作用** |
| 返回值 | 有，可以在模板中使用 | 无 |
| 缓存 | 有，依赖不变不重新计算 | 无 |
| 异步 | 不应该包含异步 | 可以执行异步操作 |
| 执行时机 | 惰性：被访问时才计算 | 依赖变化后执行（默认在组件更新前） |

```ts
const fullName = computed(() => `${firstName.value} ${lastName.value}`)

watch(keyword, async (value) => {
  results.value = await search(value)
})
```

**口述模板（30–60 秒）**

1. 一句话结论：computed 用于派生值，watch 用于执行副作用。
2. 关键机制：computed 有缓存，依赖不变不重新计算；watch 在依赖变化后执行回调。
3. 对比或注意点：computed 中不要有副作用和异步操作。
4. 可追问时补充：能用 computed 就不要用 watch 加手动赋值。

**常见追问**

- computed 可以修改吗？→ 可以提供 `get` 和 `set`，但一般只读。

---

### Q7. `watch` 和 `watchEffect` 有什么区别？

- **难度**: 初级
- **标签**: 响应式

**答案**

| 对比项 | `watch` | `watchEffect` |
|--------|---------|---------------|
| 依赖 | 显式指定数据源 | 自动收集回调中访问的响应式数据 |
| 首次执行 | 默认不执行（`immediate: true` 时执行） | 立即执行一次 |
| 新旧值 | 可以获取 | 不能获取 |
| 适用 | 需要精确控制依赖、需要旧值 | 依赖多且明确就是回调中用到的数据 |

```ts
watch(
  () => route.params.id,
  (id, oldId) => fetchUser(id),
  { immediate: true },
)

watchEffect(() => {
  // 自动追踪 page.value 和 size.value
  fetchList(page.value, size.value)
})
```

**`watch` 的数据源**：ref、reactive 对象（自动深度监听）、getter 函数、以上的数组。

**注意**：`watchEffect` 只追踪**同步执行部分**访问的数据，`await` 之后访问的数据不会被追踪。

**口述模板（30–60 秒）**

1. 一句话结论：watch 显式指定依赖，watchEffect 自动收集依赖。
2. 关键机制：watchEffect 立即执行一次，执行过程中访问的响应式数据成为依赖。
3. 对比或注意点：watchEffect 只追踪同步部分，await 之后的访问不会被追踪。
4. 可追问时补充：需要旧值、需要精确控制依赖时用 watch。

**常见追问**

- 怎么停止侦听？→ 调用 watch 返回的函数；组件中同步创建的侦听器会在组件卸载时自动停止，异步创建的需要手动停止。

---

### Q8. `ref` 和 `reactive` 有什么区别？

- **难度**: 初级
- **标签**: 响应式

**答案**

| 对比项 | `ref` | `reactive` |
|--------|-------|-----------|
| 数据类型 | 任意类型 | 只能是对象（对象、数组、Map、Set） |
| 访问 | 脚本中用 `.value`，模板中自动解包 | 直接访问属性 |
| 重新赋值 | `count.value = 新值`，保持响应式 | 整体替换会丢失响应式 |
| 解构 | — | 解构后丢失响应式，需要 `toRefs` |

```ts
const count = ref(0)
count.value++

const state = reactive({ count: 0 })
state.count++

// 错误：替换后，模板引用的仍是原来的对象
let user = reactive({ name: 'a' })
user = reactive({ name: 'b' })

// 错误：解构后得到普通值
const { count: c } = state
// 正确
const { count: c2 } = toRefs(state)
```

**官方推荐**：优先使用 `ref`，避免 reactive 的替换和解构问题。

**口述模板（30–60 秒）**

1. 一句话结论：ref 适用于任意类型，reactive 只适用于对象。
2. 关键机制：ref 通过 .value 访问，对象值内部也会转为 reactive；reactive 返回对象的 Proxy。
3. 对比或注意点：reactive 整体替换和解构都会丢失响应式。
4. 可追问时补充：官方推荐优先使用 ref。

**常见追问**

- 为什么 ref 需要 `.value`？→ JS 无法拦截基本类型变量的读写，需要包装为对象，通过 value 属性的 getter / setter 追踪。

---

### Q9. 组件的生命周期有哪些？

- **难度**: 初级
- **标签**: 组件, 生命周期

**答案**

| Composition API | Options API | 时机 |
|-----------------|-------------|------|
| `setup` 本身 | `beforeCreate` / `created` | 组件实例创建 |
| `onBeforeMount` | `beforeMount` | 首次渲染前 |
| `onMounted` | `mounted` | 挂载完成，可以访问 DOM |
| `onBeforeUpdate` | `beforeUpdate` | 响应式数据变化，更新 DOM 前 |
| `onUpdated` | `updated` | DOM 更新后 |
| `onBeforeUnmount` | `beforeUnmount` | 卸载前 |
| `onUnmounted` | `unmounted` | 卸载后，清理定时器、监听 |
| `onActivated` / `onDeactivated` | `activated` / `deactivated` | KeepAlive 缓存的组件激活 / 失活 |
| `onErrorCaptured` | `errorCaptured` | 捕获后代组件的错误 |
| `onServerPrefetch` | `serverPrefetch` | SSR 渲染前获取数据 |

**父子组件顺序**：

- 挂载：父 beforeMount → 子 beforeMount → 子 mounted → 父 mounted
- 卸载：父 beforeUnmount → 子 beforeUnmount → 子 unmounted → 父 unmounted

**口述模板（30–60 秒）**

1. 一句话结论：生命周期包括创建、挂载、更新、卸载四个阶段，以及 KeepAlive 的激活和失活。
2. 关键机制：setup 替代了 beforeCreate 和 created；DOM 操作在 onMounted 之后进行。
3. 对比或注意点：父组件的 mounted 在所有子组件 mounted 之后。
4. 可追问时补充：onUnmounted 中清理定时器和事件监听，防止内存泄漏。

**常见追问**

- 请求数据放在哪个钩子？→ setup 中或 onMounted 中都可以；SSR 时需要用 onServerPrefetch 或框架提供的数据获取方式。

---

### Q10. `<script setup>` 有什么好处？

- **难度**: 初级
- **标签**: Composition API

**答案**

`<script setup>` 是在 SFC 中使用 Composition API 的编译时语法糖：

- 顶层变量、函数、导入的组件**自动暴露给模板**，不需要 `return`
- 更简洁，更好的 TypeScript 推导（`defineProps<{...}>()`）
- **性能更好**：模板编译为与 setup 同作用域的渲染函数，不需要通过代理访问

**编译器宏**（不需要导入）：

```ts
const props = defineProps<{ title: string; count?: number }>()
const emit = defineEmits<{ change: [value: number] }>()
const model = defineModel<string>()
defineExpose({ reset })
defineOptions({ name: 'MyComponent', inheritAttrs: false })
const slots = defineSlots<{ default(props: { item: Item }): any }>()
```

**注意**：`<script setup>` 的组件默认是**关闭**的，父组件通过 ref 获取的实例上没有内部属性，需要 `defineExpose` 暴露。

**口述模板（30–60 秒）**

1. 一句话结论：script setup 是 Composition API 的编译时语法糖，代码更简洁，性能更好。
2. 关键机制：顶层绑定自动暴露给模板，提供 defineProps、defineEmits 等编译器宏。
3. 对比或注意点：组件默认关闭，需要 defineExpose 暴露给父组件。
4. 可追问时补充：编译器宏在编译时处理，不需要导入。

**常见追问**

- 编译器宏为什么不需要导入？→ 它们不是运行时函数，编译时被编译器识别并替换为对应的组件选项。

---

## 二、组件

### Q11. 组件通信有哪些方式？

- **难度**: 初级
- **标签**: 组件, 通信

**答案**

| 方式 | 方向 | 场景 |
|------|------|------|
| `props` / `emit` | 父 → 子 / 子 → 父 | 最常用 |
| `v-model`（`defineModel`） | 双向 | 表单类组件 |
| `ref` + `defineExpose` | 父调用子的方法 | 调用表单校验、重置等 |
| 插槽（作用域插槽） | 子 → 父的渲染内容 | 自定义渲染 |
| `$attrs` | 透传 | 封装组件，透传属性和事件 |
| `provide` / `inject` | 祖先 → 后代 | 跨层级（主题、表单上下文） |
| Pinia | 任意 | 全局共享状态 |
| 事件总线（mitt） | 任意 | 不推荐，数据流难以追踪 |
| 路由参数、URL | 任意 | 可分享的状态 |

**口述模板（30–60 秒）**

1. 一句话结论：父子用 props 和 emit，跨层级用 provide / inject，全局用 Pinia。
2. 关键机制：props 向下传数据，emit 向上传事件，保证单向数据流。
3. 对比或注意点：事件总线难以追踪，不推荐。
4. 可追问时补充：封装组件时用 $attrs 透传属性。

**常见追问**

- 兄弟组件怎么通信？→ 状态提升到共同的父组件，或者使用 Pinia。

---

### Q12. 为什么子组件不能直接修改 props？

- **难度**: 初级
- **标签**: 组件, 单向数据流

**答案**

**单向数据流**：props 由父组件传给子组件，父组件更新时 props 会被重新设置。如果子组件修改了 props：

- 数据流向变得难以追踪
- 父组件更新时，子组件的修改会被覆盖

Vue 在开发模式下会警告修改 props。

**正确做法**：

```ts
const props = defineProps<{ initialCount: number; size: string }>()

// 1. 作为初始值，复制到本地状态
const count = ref(props.initialCount)

// 2. 派生值使用 computed
const normalizedSize = computed(() => props.size.trim().toLowerCase())

// 3. 需要修改父组件数据：emit 事件或 v-model
const emit = defineEmits<{ 'update:size': [value: string] }>()
```

**注意**：props 是对象或数组时，修改其**内部属性**不会报警告（Vue 只做浅只读），但同样违反单向数据流。

**口述模板（30–60 秒）**

1. 一句话结论：遵循单向数据流，props 只能由父组件修改。
2. 关键机制：父组件重新渲染时会覆盖 props，子组件的修改会丢失，数据流也难以追踪。
3. 对比或注意点：对象类型的 props 修改内部属性不会报警告，但同样不应该修改。
4. 可追问时补充：需要修改时通过 emit 或 v-model 通知父组件。

**常见追问**

- 3.5 的响应式 props 解构是什么？→ `const { count = 0 } = defineProps<...>()`，解构出的变量保持响应式，编译器自动转换为 `props.count`。

---

### Q13. 组件上的 `v-model` 是怎么工作的？

- **难度**: 初级
- **标签**: 组件, v-model

**答案**

```vue
<!-- 父组件 -->
<MyInput v-model="text" />
<!-- 等价于 -->
<MyInput :modelValue="text" @update:modelValue="(value) => (text = value)" />

<!-- 多个 v-model -->
<UserForm v-model:name="name" v-model:age="age" />
```

**子组件（3.4+ 推荐 `defineModel`）**：

```vue
<script setup lang="ts">
const model = defineModel<string>({ required: true })
// 支持修饰符
const [title, modifiers] = defineModel<string>('title', {
  set(value) {
    return modifiers.capitalize ? value.charAt(0).toUpperCase() + value.slice(1) : value
  },
})
</script>

<template>
  <input v-model="model" />
</template>
```

`defineModel` 返回一个 ref，读取时是 prop 的值，赋值时触发 `update:modelValue` 事件。

**口述模板（30–60 秒）**

1. 一句话结论：组件上的 v-model 是 modelValue prop 加 update:modelValue 事件的语法糖。
2. 关键机制：defineModel 返回的 ref 读取 prop，赋值时 emit 更新事件。
3. 对比或注意点：Vue 3 支持多个 v-model（v-model:name），替代了 Vue 2 的 .sync。
4. 可追问时补充：可以通过 defineModel 的 set 处理自定义修饰符。

**常见追问**

- Vue 2 和 Vue 3 的 v-model 有什么区别？→ Vue 2 默认是 value 和 input，用 model 选项修改；Vue 3 是 modelValue 和 update:modelValue，支持多个。

---

### Q14. 插槽有哪些类型？

- **难度**: 初级
- **标签**: 组件, 插槽

**答案**

```vue
<!-- 子组件 Card.vue -->
<template>
  <div class="card">
    <header><slot name="header" /></header>
    <slot>默认内容</slot>
    <ul>
      <li v-for="item in items" :key="item.id">
        <!-- 作用域插槽：把数据传给父组件 -->
        <slot name="item" :item="item" :index="item.id" />
      </li>
    </ul>
  </div>
</template>

<!-- 父组件 -->
<Card :items="list">
  <template #header>标题</template>
  <p>默认插槽内容</p>
  <template #item="{ item }">
    <strong>{{ item.name }}</strong>
  </template>
</Card>
```

| 类型 | 作用 |
|------|------|
| 默认插槽 | 父组件传入内容 |
| 具名插槽 | 多个插入位置 |
| 作用域插槽 | 子组件提供数据，父组件决定如何渲染 |
| 动态插槽名 | `<template #[name]>` |

**作用域插槽的原理**：插槽被编译为函数，子组件调用时传入参数。

**口述模板（30–60 秒）**

1. 一句话结论：插槽让父组件向子组件传入渲染内容，分为默认、具名和作用域插槽。
2. 关键机制：作用域插槽让子组件提供数据，父组件决定渲染方式。
3. 对比或注意点：插槽内容在父组件的作用域中编译，只能访问父组件的数据。
4. 可追问时补充：插槽被编译为函数，子组件渲染时调用。

**常见追问**

- 怎么判断父组件是否传了某个插槽？→ `useSlots()` 或模板中的 `$slots.header`。

---

### Q15. `provide` / `inject` 怎么使用？

- **难度**: 初级
- **标签**: 组件, 通信

**答案**

```ts
// keys.ts：使用 InjectionKey 获得类型推导
export const ThemeKey: InjectionKey<Ref<'light' | 'dark'>> = Symbol('theme')

// 祖先组件
const theme = ref<'light' | 'dark'>('light')
provide(ThemeKey, readonly(theme))

// 后代组件
const theme = inject(ThemeKey, ref('light'))
```

**注意**：

- 提供 ref 或 reactive 对象才能保持响应式
- 推荐提供**只读**数据 + 修改函数，让修改集中在提供方
- 使用 Symbol 作为 key 避免冲突
- 应用级 provide：`app.provide(key, value)`，常用于插件

**适用场景**：组件库中的上下文（Form 和 FormItem、Tabs 和 TabPane）、主题、配置。

**口述模板（30–60 秒）**

1. 一句话结论：provide / inject 用于祖先组件向任意后代组件传递数据。
2. 关键机制：要提供 ref 或 reactive 才能保持响应式。
3. 对比或注意点：推荐提供只读数据加修改函数，使用 Symbol 作为 key。
4. 可追问时补充：组件库中 Form 和 FormItem 就是通过它通信的。

**常见追问**

- 和 Pinia 有什么区别？→ provide / inject 是组件树范围内的，可以每个子树提供不同的值；Pinia 是全局单例，有开发工具支持。

---

### Q16. `$attrs` 和属性透传是怎么工作的？

- **难度**: 初级
- **标签**: 组件

**答案**

**属性透传**：父组件传给子组件、但子组件没有声明为 props 或 emits 的属性（class、style、id、事件监听器等），会**自动添加到子组件的根元素上**。

```vue
<!-- MyButton.vue 的根元素是 button -->
<MyButton class="primary" @click="onClick" />
<!-- 渲染为 <button class="primary">，click 监听器也绑定在 button 上 -->
```

**控制透传**：

```vue
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
const attrs = useAttrs()
</script>

<template>
  <label class="wrapper">
    <!-- 透传到内部的 input，而不是根元素 -->
    <input v-bind="$attrs" />
  </label>
</template>
```

- 多根节点的组件没有自动透传，需要手动绑定 `$attrs`
- Vue 3 中 `$attrs` 包含 class、style 和事件监听器（Vue 2 的 `$listeners` 已合并）

**口述模板（30–60 秒）**

1. 一句话结论：未声明的属性和事件会自动透传到组件的根元素。
2. 关键机制：可以用 inheritAttrs: false 关闭，再手动用 v-bind="$attrs" 绑定到指定元素。
3. 对比或注意点：多根组件需要手动绑定；Vue 3 中 $listeners 合并到了 $attrs。
4. 可追问时补充：封装第三方组件时常用 $attrs 透传所有属性。

**常见追问**

- `$attrs` 是响应式的吗？→ 它总是最新的，但不是响应式对象，不能用 watch 侦听；需要时在 onUpdated 中处理。

---

### Q17. 什么是异步组件？

- **难度**: 初级
- **标签**: 组件, 性能

**答案**

异步组件在需要时才加载组件代码，用于代码分割：

```ts
const HeavyChart = defineAsyncComponent(() => import('./HeavyChart.vue'))

const AsyncDialog = defineAsyncComponent({
  loader: () => import('./Dialog.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorDisplay,
  delay: 200,      // 200ms 后才显示 loading，避免闪烁
  timeout: 10000,
})
```

**3.5 的懒 hydration**（SSR 场景）：

```ts
const Comments = defineAsyncComponent({
  loader: () => import('./Comments.vue'),
  hydrate: hydrateOnVisible(),  // 进入视口时才 hydrate
})
```

**路由懒加载**：`component: () => import('./views/User.vue')`。

**口述模板（30–60 秒）**

1. 一句话结论：异步组件在使用时才加载代码，实现代码分割。
2. 关键机制：defineAsyncComponent 接受返回 Promise 的加载函数，可以配置 loading、error、超时。
3. 对比或注意点：delay 避免加载很快时 loading 闪烁。
4. 可追问时补充：3.5 支持懒 hydration，SSR 页面中的组件可以在进入视口或空闲时才 hydrate。

**常见追问**

- 异步组件和 Suspense 的关系？→ 异步组件在 Suspense 中时，loading 状态由 Suspense 统一控制。

---

## 三、Composition API

### Q18. Composition API 解决了什么问题？

- **难度**: 初级
- **标签**: Composition API

**答案**

**Options API 的问题**：

- 一个功能的代码分散在 data、computed、methods、watch、生命周期等多个选项中，大组件难以维护
- 逻辑复用依赖 mixin：命名冲突、来源不清晰、隐式依赖
- TypeScript 推导困难（`this` 的类型）

**Composition API 的优势**：

- 按**功能**组织代码，相关逻辑放在一起
- 通过 **composable 函数**复用逻辑，来源清晰，没有命名冲突
- 完整的类型推导
- 更好的压缩（变量名可以被压缩，`this.xxx` 不行）

```ts
// composables/useMouse.ts
export function useMouse() {
  const x = ref(0)
  const y = ref(0)
  const update = (event: MouseEvent) => {
    x.value = event.pageX
    y.value = event.pageY
  }
  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))
  return { x, y }
}
```

**口述模板（30–60 秒）**

1. 一句话结论：Composition API 解决了大组件逻辑分散和 mixin 复用的问题。
2. 关键机制：按功能组织代码，通过 composable 函数复用逻辑。
3. 对比或注意点：mixin 有命名冲突和来源不清晰的问题，composable 都没有。
4. 可追问时补充：类型推导和压缩也更好。

**常见追问**

- Options API 还能用吗？→ 可以，Vue 3 两种都支持；简单组件用 Options API 也没问题。

---

### Q19. 怎么写一个好的 composable？

- **难度**: 初级
- **标签**: Composition API, 复用

**答案**

**约定**：

- 以 `use` 开头命名
- 在 setup 中**同步**调用（生命周期钩子和侦听器需要绑定到当前组件实例）
- 返回由 ref 组成的普通对象（便于解构，不丢失响应式）
- 在 `onUnmounted` 或 `onScopeDispose` 中清理副作用
- 参数接受 ref 或 getter 时，用 `toValue()` 统一处理

```ts
export function useFetch<T>(url: MaybeRefOrGetter<string>) {
  const data = ref<T | null>(null)
  const error = ref<Error | null>(null)
  const loading = ref(false)

  watchEffect(async (onCleanup) => {
    const controller = new AbortController()
    // URL 变化时取消上一次请求，避免旧请求的结果覆盖新结果
    onCleanup(() => controller.abort())
    loading.value = true
    error.value = null
    try {
      const response = await fetch(toValue(url), { signal: controller.signal })
      data.value = await response.json()
    } catch (err) {
      if ((err as Error).name !== 'AbortError') error.value = err as Error
    } finally {
      loading.value = false
    }
  })

  return { data, error, loading }
}
```

成熟的 composable 库：**VueUse**。

**口述模板（30–60 秒）**

1. 一句话结论：composable 以 use 开头，在 setup 中同步调用，返回 ref 组成的对象。
2. 关键机制：利用响应式 API 和生命周期钩子封装有状态的逻辑。
3. 对比或注意点：要清理副作用，处理异步竞态。
4. 可追问时补充：参数支持 ref 和 getter，用 toValue 统一处理。

**常见追问**

- 为什么要在 setup 中同步调用？→ 生命周期钩子要注册到当前活动的组件实例，await 之后当前实例已经丢失。

---

### Q20. `nextTick` 是什么？什么时候用？

- **难度**: 初级
- **标签**: 更新机制

**答案**

Vue 修改数据后**不会立即更新 DOM**，而是把更新任务放入队列，在下一个微任务中批量执行。`nextTick` 返回一个 Promise，在 DOM 更新完成后 resolve。

```ts
const list = ref<string[]>([])
const listEl = useTemplateRef<HTMLUListElement>('listEl')

async function addItem(item: string) {
  list.value.push(item)
  // 此时 DOM 还没有更新
  await nextTick()
  // DOM 已更新，可以滚动到底部
  listEl.value?.scrollTo({ top: listEl.value.scrollHeight })
}
```

**常见场景**：修改数据后需要读取更新后的 DOM（尺寸、滚动位置、聚焦新出现的输入框）。

**口述模板（30–60 秒）**

1. 一句话结论：nextTick 用于在 DOM 更新完成后执行代码。
2. 关键机制：Vue 把更新放入队列，在微任务中批量执行；nextTick 等待这次刷新完成。
3. 对比或注意点：修改数据后立即读取 DOM 拿到的是旧值。
4. 可追问时补充：也可以用 watch 的 flush: 'post' 在 DOM 更新后执行回调。

**常见追问**

- 为什么要异步更新？→ 同一个 tick 中多次修改数据只需要更新一次 DOM，避免重复渲染。

---

### Q21. 怎么获取 DOM 元素和组件实例？

- **难度**: 初级
- **标签**: 模板引用

**答案**

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'
import ChildForm from './ChildForm.vue'

// 3.5+ 推荐
const inputEl = useTemplateRef<HTMLInputElement>('input')
const formRef = useTemplateRef<InstanceType<typeof ChildForm>>('form')

onMounted(() => {
  inputEl.value?.focus()
  formRef.value?.validate()  // 需要子组件 defineExpose({ validate })
})
</script>

<template>
  <input ref="input" />
  <ChildForm ref="form" />
</template>
```

- 3.5 之前：声明一个与 ref 属性同名的 `ref(null)`
- 在 `onMounted` 之后才有值；v-if 为假时为 null
- v-for 中的 ref 是数组（顺序不保证与源数据一致）

**口述模板（30–60 秒）**

1. 一句话结论：通过模板 ref 获取 DOM 元素和子组件实例，3.5 推荐 useTemplateRef。
2. 关键机制：挂载后 ref 才有值；子组件需要 defineExpose 暴露方法。
3. 对比或注意点：v-if 为假时是 null，v-for 中是数组。
4. 可追问时补充：应该优先使用 props 和事件，只在必要时调用子组件的方法。

**常见追问**

- 为什么 3.5 引入 `useTemplateRef`？→ 不再依赖变量名与 ref 属性同名的隐式约定，类型推导更好，也可以在 composable 中使用。

---

## 四、内置组件

### Q22. `KeepAlive` 有什么作用？

- **难度**: 初级
- **标签**: 内置组件

**答案**

`KeepAlive` 缓存被切换掉的组件实例，而不是销毁它，再次显示时恢复状态（表单输入、滚动位置等）。

```vue
<RouterView v-slot="{ Component }">
  <KeepAlive :include="['UserList', 'OrderList']" :max="10">
    <component :is="Component" />
  </KeepAlive>
</RouterView>
```

- `include` / `exclude`：按组件的 `name` 匹配（`<script setup>` 中用 `defineOptions({ name })` 声明，3.2.34 起也会根据文件名推断）
- `max`：最多缓存的实例数，超过时按 **LRU** 淘汰最久未访问的
- 生命周期：`onActivated`（每次显示时）、`onDeactivated`（每次隐藏时）

**场景**：列表页进入详情页再返回，保留列表的筛选条件和滚动位置。

**口述模板（30–60 秒）**

1. 一句话结论：KeepAlive 缓存组件实例，切换回来时恢复状态。
2. 关键机制：被缓存的组件不会卸载，而是移动到隐藏的容器中；max 超出时按 LRU 淘汰。
3. 对比或注意点：include 按组件 name 匹配；激活时触发 onActivated。
4. 可追问时补充：缓存过多会占用内存，需要设置 max。

**常见追问**

- 被缓存的组件需要刷新数据怎么办？→ 在 onActivated 中重新请求，或者根据条件修改 include 让它重新创建。

---

### Q23. `Teleport` 和 `Suspense` 是什么？

- **难度**: 初级
- **标签**: 内置组件

**答案**

**Teleport**：把组件的一部分模板渲染到 DOM 中的其他位置，但逻辑上仍属于当前组件（props、事件、provide 都正常）。

```vue
<button @click="open = true">打开</button>
<Teleport to="body">
  <div v-if="open" class="modal">弹窗内容</div>
</Teleport>
```

- 场景：弹窗、提示框、通知，避免被父元素的 `overflow: hidden`、`z-index`、`transform` 影响
- `disabled`：在某些情况下不传送（例如移动端）
- 3.5 `defer`：等同一轮渲染中的其他内容挂载后再寻找目标，可以传送到同一组件中后面渲染的元素

**Suspense**（仍是实验性功能）：在异步依赖（`async setup`、异步组件）都完成前显示后备内容。

```vue
<Suspense>
  <UserProfile />  <!-- 内部有顶层 await -->
  <template #fallback>加载中...</template>
</Suspense>
```

**口述模板（30–60 秒）**

1. 一句话结论：Teleport 把内容渲染到其他 DOM 位置，Suspense 统一处理异步组件的加载状态。
2. 关键机制：Teleport 只改变 DOM 位置，逻辑关系不变。
3. 对比或注意点：弹窗用 Teleport 避免被父元素的 overflow 和层叠上下文影响。
4. 可追问时补充：Suspense 在 Vue 3 中仍是实验性功能。

**常见追问**

- Teleport 中的事件会冒泡到哪里？→ 原生 DOM 事件按实际 DOM 位置冒泡；Vue 组件事件按组件树传递。

---

### Q24. `Transition` 怎么使用？

- **难度**: 初级
- **标签**: 内置组件, 动画

**答案**

`Transition` 在元素插入、移除时自动应用 CSS 类名：

```vue
<Transition name="fade" mode="out-in">
  <component :is="current" :key="currentKey" />
</Transition>

<style>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
```

**类名**：`v-enter-from` → `v-enter-active` → `v-enter-to`；离开同理。

- `mode="out-in"`：先离开再进入，避免两个元素同时存在
- **JS 钩子**：`@before-enter`、`@enter(el, done)`、`@leave(el, done)`，用于 GSAP 等动画库
- **TransitionGroup**：列表动画，支持 `v-move` 类实现位置移动动画（FLIP 技术）
- 触发条件：`v-if`、`v-show`、动态组件、key 变化

**口述模板（30–60 秒）**

1. 一句话结论：Transition 在元素进入和离开时自动添加 CSS 类，实现过渡动画。
2. 关键机制：分为 enter-from、enter-active、enter-to 等阶段类名。
3. 对比或注意点：切换组件时使用 mode="out-in" 避免同时存在。
4. 可追问时补充：列表动画用 TransitionGroup，位置移动使用 FLIP 技术。

**常见追问**

- 什么是 FLIP？→ First（记录初始位置）、Last（记录最终位置）、Invert（用 transform 反向移动到初始位置）、Play（移除 transform 播放动画），只使用 transform，性能好。

---

## 五、生态

### Q25. Pinia 怎么使用？和 Vuex 有什么区别？

- **难度**: 初级
- **标签**: Pinia, 状态管理

**答案**

```ts
// Setup Store（推荐，与 Composition API 风格一致）
export const useCartStore = defineStore('cart', () => {
  const items = ref<CartItem[]>([])
  const total = computed(() => items.value.reduce((sum, item) => sum + item.price * item.count, 0))

  function add(item: CartItem) {
    items.value.push(item)
  }
  async function checkout() {
    await api.checkout(items.value)
    items.value = []
  }

  return { items, total, add, checkout }
})

// 组件中
const cart = useCartStore()
const { items, total } = storeToRefs(cart)  // 解构状态需要 storeToRefs
cart.add(item)                               // 方法可以直接解构或调用
```

| 对比项 | Vuex | Pinia |
|--------|------|-------|
| 修改状态 | mutation（同步）+ action（异步） | 直接修改或 action，没有 mutation |
| 模块 | 嵌套模块、命名空间 | 每个 store 独立，按需导入 |
| TypeScript | 支持差 | 完整类型推导 |
| 体积 | 较大 | 约 1KB |
| 状态 | Vuex 4 维护模式 | 官方推荐 |

**口述模板（30–60 秒）**

1. 一句话结论：Pinia 是 Vue 官方推荐的状态管理库，取代了 Vuex。
2. 关键机制：每个 store 独立定义，可以用 Setup 风格，直接修改状态或调用 action。
3. 对比或注意点：解构状态需要 storeToRefs，否则会丢失响应式。
4. 可追问时补充：没有 mutation，类型推导完整，体积很小。

**常见追问**

- 为什么解构需要 `storeToRefs`？→ store 是 reactive 对象，直接解构得到的是普通值；storeToRefs 只把状态和 getter 转为 ref，跳过方法。

---

### Q26. Vue Router 的基本使用有哪些？

- **难度**: 初级
- **标签**: Router

**答案**

```ts
const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/', component: Home },
    {
      path: '/users/:id',
      name: 'user',
      component: () => import('./views/User.vue'),  // 懒加载
      props: true,                                  // 参数作为 props 传入
      meta: { requiresAuth: true },
      children: [{ path: 'posts', component: UserPosts }],
    },
    { path: '/:pathMatch(.*)*', component: NotFound },
  ],
  scrollBehavior: (to, from, savedPosition) => savedPosition ?? { top: 0 },
})
```

```ts
// 组件中
const route = useRoute()      // 当前路由信息（params、query、meta）
const router = useRouter()    // 导航方法
router.push({ name: 'user', params: { id: 1 }, query: { tab: 'info' } })
router.replace('/login')
router.back()
```

**`route` 与 `router`**：route 是当前路由的状态（响应式），router 是路由器实例（提供导航方法）。

**口述模板（30–60 秒）**

1. 一句话结论：通过 createRouter 配置路由表，useRoute 读取当前路由，useRouter 进行导航。
2. 关键机制：支持动态参数、嵌套路由、命名路由、懒加载和 meta 信息。
3. 对比或注意点：同一组件内参数变化时组件被复用，需要 watch 参数变化。
4. 可追问时补充：scrollBehavior 控制导航后的滚动位置。

**常见追问**

- 从 `/users/1` 跳到 `/users/2` 组件会重新创建吗？→ 不会，组件被复用；需要 watch `route.params.id` 或使用 `onBeforeRouteUpdate`。

---

### Q27. hash 模式和 history 模式有什么区别？

- **难度**: 初级
- **标签**: Router

**答案**

| 对比项 | hash 模式 | history 模式 |
|--------|-----------|--------------|
| URL | `/#/users/1` | `/users/1` |
| 原理 | 监听 `hashchange`，hash 变化不请求服务器 | `history.pushState` + 监听 `popstate` |
| 服务器配置 | 不需要 | **需要**：所有路径回退到 index.html，否则刷新 404 |
| SEO | 差 | 好（配合 SSR） |
| 创建 | `createWebHashHistory()` | `createWebHistory()` |

另有 `createMemoryHistory()`，用于 SSR 和测试。

**Nginx 配置**：

```nginx
location / {
  try_files $uri $uri/ /index.html;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：hash 模式用 URL 的 hash 部分，history 模式用 History API。
2. 关键机制：hash 变化不会请求服务器；pushState 修改 URL 也不请求，但刷新时会请求真实路径。
3. 对比或注意点：history 模式需要服务器把所有路径回退到 index.html。
4. 可追问时补充：SSR 使用 memory history。

**常见追问**

- pushState 会触发 popstate 吗？→ 不会，只有浏览器前进后退（或 history.back）才触发，所以路由器在 push 时需要自己更新状态。

---

### Q28. 导航守卫有哪些？

- **难度**: 初级
- **标签**: Router

**答案**

| 类型 | API | 场景 |
|------|-----|------|
| 全局前置 | `router.beforeEach` | 登录校验、权限 |
| 全局解析 | `router.beforeResolve` | 所有组件内守卫和异步组件解析之后 |
| 全局后置 | `router.afterEach` | 修改标题、埋点、关闭进度条 |
| 路由独享 | `beforeEnter` | 特定路由的校验 |
| 组件内 | `onBeforeRouteUpdate`、`onBeforeRouteLeave` | 参数变化、离开前确认未保存的修改 |

```ts
router.beforeEach(async (to) => {
  const auth = useAuthStore()
  if (to.meta.requiresAuth && !auth.isLoggedIn) {
    return { path: '/login', query: { redirect: to.fullPath } }
  }
  // 返回 undefined 或 true 表示放行，false 表示取消
})
```

Vue Router 4 推荐**返回值**控制导航，而不是调用 `next()`。

**口述模板（30–60 秒）**

1. 一句话结论：导航守卫分为全局守卫、路由独享守卫和组件内守卫。
2. 关键机制：beforeEach 做登录和权限校验，返回路由地址表示重定向，返回 false 取消导航。
3. 对比或注意点：Vue Router 4 推荐用返回值代替 next。
4. 可追问时补充：onBeforeRouteLeave 用于离开前确认未保存的表单。

**常见追问**

- 前端路由守卫能保证安全吗？→ 不能，只是体验层面的控制；接口必须由后端鉴权。

---

### Q29. 样式隔离 `scoped` 是怎么实现的？

- **难度**: 初级
- **标签**: 样式

**答案**

```vue
<style scoped>
.title { color: red; }
</style>
```

编译后：

```html
<h1 class="title" data-v-7ba5bd90>
```

```css
.title[data-v-7ba5bd90] { color: red; }
```

- 编译器为组件的元素添加唯一的 `data-v-hash` 属性，CSS 选择器加上属性选择器
- 子组件的**根元素**会同时带有父组件的 hash，所以父组件可以修改子组件根元素的样式
- 修改子组件内部元素：`:deep(.inner)`；修改插槽内容：`:slotted(.item)`；全局样式：`:global(.x)`

**CSS Modules**：`<style module>`，类名被哈希化，通过 `$style.title` 使用。

**`v-bind()` in CSS**：`color: v-bind(themeColor)`，编译为 CSS 变量，随状态变化。

**口述模板（30–60 秒）**

1. 一句话结论：scoped 通过给元素添加唯一的 data 属性，CSS 选择器加上属性选择器实现隔离。
2. 关键机制：子组件根元素同时带有父组件的 hash。
3. 对比或注意点：修改子组件内部样式需要 :deep()。
4. 可追问时补充：也可以用 CSS Modules，或者在 CSS 中用 v-bind 绑定状态。

**常见追问**

- scoped 能完全隔离样式吗？→ 不能，全局样式仍然会影响组件；它只保证组件的样式不影响外部。

---

### Q30. Vue 项目中有哪些常见的性能优化手段？

- **难度**: 初级
- **标签**: 性能

**答案**

**加载**：

- 路由懒加载、异步组件
- 组件库按需引入
- 图片懒加载、资源压缩、CDN

**渲染**：

- `v-show` 用于频繁切换，`v-if` 用于很少切换
- 列表使用稳定的 key
- 长列表使用虚拟滚动（vue-virtual-scroller、`@tanstack/vue-virtual`）
- `v-once`（静态内容）、`v-memo`（依赖不变跳过更新）
- 拆分组件，缩小更新范围
- `KeepAlive` 缓存页面

**响应式**：

- 大型只读数据使用 `shallowRef` 或 `markRaw`（例如图表实例、地图实例、大列表）
- 使用 `computed` 缓存计算结果，避免在模板中调用方法

**口述模板（30–60 秒）**

1. 一句话结论：Vue 的优化分为加载、渲染和响应式三个方面。
2. 关键机制：懒加载减少首屏体积；虚拟列表和组件拆分减少渲染量。
3. 对比或注意点：大型数据和第三方实例使用 shallowRef 或 markRaw，避免深度代理。
4. 可追问时补充：v-memo 可以让大列表中未变化的项跳过更新。

**常见追问**

- 为什么图表实例要用 `markRaw`？→ ECharts 等实例结构复杂，被深度代理后访问开销大，而且内部的 this 可能变成代理对象导致异常。
