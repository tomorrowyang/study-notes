# CSS 高级 Q&A（能扛追问）

> 目标：把 CSS 放到渲染管线、性能指标、设计系统、现代能力与架构取舍中讨论。

---

## 一、渲染管线与性能

### Q1. 从样式计算到上屏，CSS 在渲染管线里经历哪些阶段？

- **难度**: senior
- **标签**: 渲染管线

**答案**

简化流程：

1. **样式计算（Style / Recalc）**：匹配选择器，计算每个节点的 computed style
2. **布局（Layout）**：计算几何位置和尺寸
3. **绘制（Paint）**：生成绘制指令（文字、颜色、阴影等）
4. **合成（Composite）**：分层、光栅化、合成到屏幕

CSS 变化可能只触发 Style + Composite，也可能连带 Layout + Paint。`:has()`、复杂选择器、大 DOM 会放大样式计算成本。

**口述模板（30–60 秒）**

1. 一句话结论：CSS 影响 Style → Layout → Paint → Composite。
2. 关键机制：越靠前的阶段改动越贵。
3. 注意点：选择器复杂度也影响性能，不只是动画属性。
4. 可追问时补充：Chrome Performance 看 Recalculate Style 耗时。

**常见追问**

- 「样式计算慢」常见原因？→ 选择器过于宽泛、大量节点、频繁 class 切换、`:has()` 滥用。

---

### Q2. 什么是强制同步布局（Forced Synchronous Layout）？如何避免？

- **难度**: senior
- **标签**: 性能, Layout Thrashing

**答案**

JS 先改样式（使布局失效），紧接着读取布局属性（`offsetHeight`、`getBoundingClientRect`），浏览器必须立刻执行布局，打断批处理。

循环中读写交错会造成 **Layout Thrashing**。

避免：

- 先批量读，再批量写
- 用 `requestAnimationFrame` 安排写入
- 用 CSS 动画替代 JS 每帧改几何属性
- 必要时使用 ResizeObserver 而不是轮询读尺寸

**口述模板（30–60 秒）**

1. 一句话结论：读写交错迫使浏览器同步算布局。
2. 关键机制：样式脏后读布局会触发立即 Layout。
3. 注意点：循环里最容易出现。
4. 可追问时补充：FastDOM 类读写分离思想。

**常见追问**

- `getComputedStyle` 一定会触发布局吗？→ 读某些属性会触发，取决于是否需要最新几何信息。

---

### Q3. 合成层（Compositing Layer）过多会有什么问题？

- **难度**: senior
- **标签**: 合成层

**答案**

`transform`、`opacity` 动画、`will-change`、video、canvas 等可能提升为合成层。层过多导致：

- 显存/内存上涨
- 合成成本上升，低端机掉帧
- 层叠上下文增多，z-index 更难推理

用 DevTools Layers / Rendering 观察，按需提升，动画结束移除 `will-change`。

**口述模板（30–60 秒）**

1. 一句话结论：合成层不是越多越好。
2. 关键机制：每层需要纹理与合成计算。
3. 注意点：层爆炸常见于给大量列表项加 will-change。
4. 可追问时补充：隐式提层（overlap 导致的提层）。

**常见追问**

- 什么是隐式合成？→ 元素与已提层元素重叠，浏览器可能被迫给其他元素也提层。

---

### Q4. CSS 如何影响 CLS？系统性怎么治理？

- **难度**: senior
- **标签**: CLS, 性能

**答案**

常见 CSS/HTML 原因：

- 图片、视频、iframe 无尺寸
- 动态插入广告、横幅、Cookie 条
- Web 字体交换导致文字重排
- 骨架屏和真实内容尺寸不一致
- 异步注入的样式改变布局

治理：

- `width/height` + `aspect-ratio` 预留空间
- 为动态区块设置最小高度
- 字体：`size-adjust`、`font-display` 策略
- 动画用 `transform` 而不是改变布局尺寸
- RUM 监控 CLS 来源元素

**口述模板（30–60 秒）**

1. 一句话结论：CLS 本质是未预留空间的布局变化。
2. 关键机制：可见元素位置在无用户输入时被推移。
3. 注意点：字体和广告是常见隐藏来源。
4. 可追问时补充：用 Layout Shift Regions 调试。

**常见追问**

- 用户点击后展开内容算 CLS 吗？→ 有用户输入后短时间内的位移通常不计入。

---

### Q5. 大型列表页的 CSS 性能如何优化？

- **难度**: senior
- **标签**: 性能

**答案**

- 减少 DOM 节点：虚拟列表
- 离屏区块：`content-visibility: auto` + `contain-intrinsic-size`
- 避免复杂后代选择器：`.list li .a .b span` 链过长
- 卡片 hover 只改 `transform`/`opacity`
- 避免全量 `:has()` 驱动样式
- 图片 lazy + 尺寸占位
- 限制阴影和 `filter` 在大量元素上同时动画

**口述模板（30–60 秒）**

1. 一句话结论：少节点、少选择器成本、少布局抖动。
2. 关键机制：样式计算和绘制随节点数线性甚至超线性增长。
3. 注意点：content-visibility 要防滚动跳动。
4. 可追问时补充：虚拟列表与无障碍的平衡。

**常见追问**

- 虚拟列表对 Ctrl+F 有影响吗？→ 有，未渲染内容搜不到，需要产品取舍。

---

## 二、现代 CSS 能力

### Q6. Anchor Positioning 解决什么问题？如何渐进增强？

- **难度**: senior
- **标签**: Anchor Positioning

**答案**

传统 tooltip、下拉菜单需要 JS 计算触发元素位置并处理视口翻转。Anchor Positioning 允许 CSS 声明锚点关系：

```css
.trigger { anchor-name: --menu-anchor; }
.menu {
  position: absolute;
  position-anchor: --menu-anchor;
  position-area: bottom span-right;
  position-try-fallbacks: flip-block;
}
```

配合 Popover 可减少定位 JS。它在 2025–2026 年逐步进入 Baseline，生产要 `@supports (anchor-name: --a)` 检测，并保留 Floating UI 等 JS 回退。

**口述模板（30–60 秒）**

1. 一句话结论：Anchor Positioning 让浮层用 CSS 绑定锚点。
2. 关键机制：anchor-name + position-anchor + 定位区域/fallback。
3. 注意点：新特性要特性检测和回退。
4. 可追问时补充：`anchor-scope` 解决多组件同名锚点冲突。

**常见追问**

- 和 Popover 什么关系？→ Popover 管显示和层级，Anchor 管定位，常组合。

---

### Q7. Scroll-driven Animations 是什么？适合什么？

- **难度**: senior
- **标签**: 动画

**答案**

让动画进度绑定滚动位置，而非时间：

```css
@keyframes grow { from { transform: scaleX(0); } to { transform: scaleX(1); } }
.progress {
  transform-origin: left;
  animation: grow linear both;
  animation-timeline: scroll(root);
}
```

适合阅读进度条、视差、元素进入视口的渐显（`view()`）。替代大量 `scroll` 事件监听，性能更好。需要 `@supports` 回退，并尊重 `prefers-reduced-motion`。

**口述模板（30–60 秒）**

1. 一句话结论：滚动驱动动画用滚动位置控制关键帧进度。
2. 关键机制：scroll() / view() 时间线。
3. 注意点：动效过强影响可访问性。
4. 可追问时补充：与 IntersectionObserver 方案对比。

**常见追问**

- 可以替代所有滚动 JS 吗？→ 不能，复杂业务逻辑仍需 JS。

---

### Q8. View Transitions 能做什么？有什么坑？

- **难度**: senior
- **标签**: View Transitions

**答案**

View Transitions 可在两个 DOM 状态间做过渡：

- 同文档 SPA：`document.startViewTransition(() => updateDOM())`
- 跨文档 MPA：`@view-transition { navigation: auto; }`（浏览器支持差异需查）

用 `view-transition-name` 让共享元素做形变动画。

坑：

- 需要稳定的唯一 `view-transition-name`
- 动画期间截图快照，内容复杂时可能有性能成本
- 尊重减少动效偏好
- 降级：不支持时直接更新 DOM

**口述模板（30–60 秒）**

1. 一句话结论：View Transitions 原生实现页面状态过渡。
2. 关键机制：浏览器捕获新旧快照并动画。
3. 注意点：命名冲突、性能、降级。
4. 可追问时补充：路由框架集成。

**常见追问**

- 共享元素过渡需要什么？→ 新旧状态中同名的 `view-transition-name`。

---

### Q9. `@scope` 和 CSS Modules 的隔离有什么区别？

- **难度**: senior
- **标签**: @scope, 作用域

**答案**

`@scope` 是原生作用域规则：

```css
@scope (.card) to (.card-content) {
  img { border-radius: 8px; }
}
```

可限定根与下界（donut scope），并影响层叠中的「作用域邻近度」。

CSS Modules 是**构建期**哈希类名隔离，不依赖浏览器新特性。

对比：

- `@scope`：原生、可下界、仍需注意兼容性
- Modules：成熟、零运行时，但动态嵌套场景需约定

**口述模板（30–60 秒）**

1. 一句话结论：@scope 是原生作用域，Modules 是编译期隔离。
2. 关键机制：scope 限制选择器匹配范围。
3. 注意点：生产用 scope 前查兼容。
4. 可追问时补充：与 Shadow DOM 隔离强度不同。

**常见追问**

- Shadow DOM 和 @scope？→ Shadow DOM 隔离更强（DOM + 样式），@scope 只是样式匹配范围。

---

### Q10. `color-mix()`、OKLCH 在设计系统中的价值？

- **难度**: senior
- **标签**: 颜色, 设计系统

**答案**

- **OKLCH**：更接近人眼感知的颜色空间，调亮度和饱和度时色相更稳定，适合生成色阶
- **color-mix()**：原生混色，例如悬停色：

```css
.btn {
  --brand: oklch(55% 0.18 250);
  background: var(--brand);
}
.btn:hover {
  background: color-mix(in oklch, var(--brand), white 12%);
}
```

减少设计师手写大量 hex 色板，也便于主题派生。注意广色域显示器与对比度校验。

**口述模板（30–60 秒）**

1. 一句话结论：OKLCH 便于系统化生成色板，color-mix 便于派生状态色。
2. 关键机制：感知均匀的颜色插值。
3. 注意点：仍需 WCAG 对比度检查。
4. 可追问时补充：与 Design Tokens 结合。

**常见追问**

- 老浏览器怎么办？→ 提供 hex/rgb 回退或用构建工具降级。

---

## 三、设计系统与架构

### Q11. 如何设计一套 CSS Design Tokens 体系？

- **难度**: senior
- **标签**: 设计系统

**答案**

分层：

```
基础令牌（Primitive）：--blue-500、--space-4
语义令牌（Semantic）：--color-bg-surface、--color-text-primary
组件令牌（Component）：--button-bg、--button-radius
```

实践：

- 语义令牌对接业务含义，主题切换只换语义层映射
- 组件只消费语义 / 组件令牌，不直接用 `#fff`
- 尺寸、间距、圆角、阴影、动效时长都做令牌
- 用 `@property` 声明需动画的令牌类型

**口述模板（30–60 秒）**

1. 一句话结论：令牌分基础、语义、组件三层。
2. 关键机制：主题切换改映射，不改组件实现。
3. 注意点：避免组件里硬编码颜色。
4. 可追问时补充：Figma Tokens 同步代码。

**常见追问**

- 多品牌主题怎么做？→ 同一语义令牌指向不同基础色板。

---

### Q12. 组件库样式如何避免「样式污染」与「覆盖困难」？

- **难度**: senior
- **标签**: 组件库

**答案**

- **隔离**：CSS Modules / scoped / Shadow DOM / `@scope`
- **低特异性默认样式**：组件内部用 `:where()` 或单类名
- **层叠层**：组件库放 `@layer components`，业务在更高层覆盖
- **扩展点**：CSS 变量、`::part()`、className 透传
- **避免**：全局 reset 写进组件、使用 `!important`、深层后代选择器

**口述模板（30–60 秒）**

1. 一句话结论：组件库要「默认可用、业务易覆盖」。
2. 关键机制：隔离 + 低特异性 + 显式扩展点。
3. 注意点：滥用 important 会让下游失控。
4. 可追问时补充：版本升级的样式破坏性变更管理。

**常见追问**

- Ant Design 类库为什么常提供 token / ConfigProvider？→ 统一主题与降低业务覆盖成本。

---

### Q13. 运行时 CSS-in-JS 在 SSR 和性能上有哪些问题？

- **难度**: senior
- **标签**: CSS-in-JS, SSR

**答案**

问题：

- 渲染时序列化/注入样式，增加主线程工作
- SSR 需收集样式并内联，流式渲染更复杂
- 动态 props 生成大量类名，缓存命中差
- 与 React 并发特性、Server Components 集成成本高

趋势：零运行时方案（Vanilla Extract、Linaria、Panda、StyleX）或原子化 CSS、CSS Modules + CSS 变量。

**口述模板（30–60 秒）**

1. 一句话结论：运行时 CSS-in-JS 有渲染期成本和 SSR 复杂度。
2. 关键机制：样式在 JS 执行时才生成。
3. 注意点：小项目可接受，大规模应用要评估。
4. 可追问时补充：迁移策略逐步替换。

**常见追问**

- styled-components 还能用吗？→ 能用，但要在性能和架构上权衡。

---

### Q14. Tailwind 在大型团队中的优缺点与治理？

- **难度**: senior
- **标签**: Tailwind

**答案**

优点：开发快、约束设计系统、产物可按使用裁剪、减少命名成本。

缺点：模板类名冗长、可读性下降、设计令牌若不受控会乱用任意值。

治理：

- 配置主题令牌，限制随意 `w-[123px]`
- 提取组件封装重复组合
- `prettier-plugin-tailwindcss` 排序
- 与 `@layer` / 组件库样式边界清晰
- Code Review 关注语义 HTML 与可访问性，不只看类名

**口述模板（30–60 秒）**

1. 一句话结论：Tailwind 适合高迭代 + 有设计约束的团队。
2. 关键机制：原子类映射设计令牌。
3. 注意点：任意值失控会破坏一致性。
4. 可追问时补充：与 CSS Modules 混用边界。

**常见追问**

- Tailwind 会不会导致 CSS 很大？→ 现代 JIT 按使用生成，通常可控。

---

### Q15. 微前端中如何隔离子应用样式？

- **难度**: senior
- **标签**: 微前端

**答案**

| 方案 | 隔离强度 | 问题 |
|------|----------|------|
| 命名前缀 / BEM | 弱 | 靠约定 |
| CSS Modules | 中 | 全局样式仍可能冲突 |
| Shadow DOM | 强 | 弹窗、字体、全局主题穿透难 |
| iframe | 最强 | 体验和通信成本高 |
| 运行时 scoped 前缀 | 中 | qiankun 等需注意动态样式 |

建议：基座定义 reset 与设计令牌；子应用禁止修改全局 `html/body`；弹窗挂载点统一。

**口述模板（30–60 秒）**

1. 一句话结论：样式隔离按「冲突风险 vs 体验」选强度。
2. 关键机制：作用域越强，全局能力越难共享。
3. 注意点：弹窗和全局字体是常见坑。
4. 可追问时补充：设计令牌由基座统一下发。

**常见追问**

- 子应用卸载后样式残留？→ 动态插入的 style/link 要在卸载时清理。

---

## 四、复杂布局与边界

### Q16. 如何实现「卡片网格 + 底部按钮对齐 + 响应式」？

- **难度**: senior
- **标签**: Grid, 布局

**答案**

方案一：卡片内 Flex column，正文 `flex: 1`，按钮 `margin-top: auto`。

方案二：外层 Grid + 卡片 `subgrid` 对齐行轨道（跨卡片对齐标题/正文/按钮）。

```css
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  gap: 16px;
}
.card {
  display: grid;
  grid-template-rows: subgrid;
  grid-row: span 3;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：单卡对齐用 Flex，跨卡对齐用 subgrid。
2. 关键机制：subgrid 共享父行轨道。
3. 注意点：内容极长时仍要截断策略。
4. 可追问时补充：容器查询切换卡片布局。

**常见追问**

- 不能用 subgrid 时？→ 固定区块最小高度或 JS 测量（不推荐）。

---

### Q17. 粘性页脚（Sticky Footer）有哪些实现？

- **难度**: senior
- **标签**: 布局

**答案**

现代推荐：

```css
body {
  min-height: 100dvh;
  display: grid;
  grid-template-rows: auto 1fr auto;
}
```

或 Flex column + `main { flex: 1; }`。历史方案有负 margin、calc 等，可提但不推荐。

**口述模板（30–60 秒）**

1. 一句话结论：Grid/Flex 让主内容撑满剩余空间。
2. 关键机制：中间行 `1fr` 或 `flex:1`。
3. 注意点：移动端用 dvh。
4. 可追问时补充：与 Sticky 定位的区别。

**常见追问**

- `position: fixed` 页脚可以吗？→ 会遮挡内容，需要额外 padding，语义也不同。

---

### Q18. 文本溢出、多行截断、数字等宽的工程处理？

- **难度**: senior
- **标签**: 文本

**答案**

```css
.ellipsis { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.clamp-2 {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
.price { font-variant-numeric: tabular-nums; }
```

注意：Flex 子项截断需要 `min-width: 0`；截断要配合 `title` 或展开交互，避免信息丢失。

**口述模板（30–60 秒）**

1. 一句话结论：单行 ellipsis，多行 line-clamp，数字 tabular-nums。
2. 关键机制：需要限制宽度与溢出。
3. 注意点：可访问性不能只靠截断。
4. 可追问时补充：中文断行 `word-break` / `overflow-wrap`。

**常见追问**

- 倒计时数字跳动布局？→ 使用 tabular-nums 或固定宽度。

---

### Q19. 如何做无障碍友好的焦点样式？

- **难度**: senior
- **标签**: a11y

**答案**

- 不要全局 `outline: none` 且无替代
- 使用 `:focus-visible` 只在键盘导航时显示明显焦点环
- 焦点环对比度足够，不被 `overflow: hidden` 裁掉
- 自定义组件焦点要和视觉状态一致

```css
:focus-visible {
  outline: 2px solid var(--color-focus);
  outline-offset: 2px;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：焦点必须可见，优先 focus-visible。
2. 关键机制：键盘用户依赖焦点定位。
3. 注意点：去掉 outline 必须有等效替代。
4. 可追问时补充：WCAG 2.2 焦点外观要求。

**常见追问**

- 鼠标点击不想出现焦点框？→ 用 `:focus-visible`，不要粗暴删除焦点。

---

### Q20. `prefers-reduced-motion` 应如何在系统中落地？

- **难度**: senior
- **标签**: a11y, 动效

**答案**

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

更好：在动效令牌层提供「无动画/弱动画」变体，组件按需降级，而不是只用全局暴力覆盖（全局覆盖可作为兜底）。

**口述模板（30–60 秒）**

1. 一句话结论：尊重用户减少动效偏好。
2. 关键机制：媒体查询检测系统设置。
3. 注意点：关键状态变化仍需可感知（可用淡入代替位移）。
4. 可追问时补充：视差、自动轮播尤其要关。

**常见追问**

- 加载 spinner 也要停吗？→ 可改为静态或低幅度提示，但不能让用户不知道在加载。

---

## 五、排障与取舍

### Q21. 一个弹窗在部分页面被遮挡，你如何系统排查？

- **难度**: senior
- **标签**: 排障, z-index

**答案**

1. DevTools 看弹窗与遮挡元素是否同一层叠上下文
2. 沿祖先查找 `transform`、`opacity`、`filter`、`isolation`、`z-index`
3. 检查是否被 `overflow: hidden` 裁剪
4. 检查 fixed 是否因 transform 祖先改变包含块
5. 方案：Portal 到 body、原生 `dialog` top layer、统一 z-index 令牌表

**口述模板（30–60 秒）**

1. 一句话结论：先分是层叠问题还是裁剪问题。
2. 关键机制：上下文边界和包含块。
3. 注意点：不要无限加大 z-index。
4. 可追问时补充：设计系统 z-index 分层规范。

**常见追问**

- z-index 令牌怎么设？→ 如 base 0、dropdown 1000、modal 2000、toast 3000，团队统一。

---

### Q22. 动画卡顿如何定位与优化？

- **难度**: senior
- **标签**: 性能

**答案**

定位：

- Performance 录制，看 Long Tasks、Layout、Paint
- Rendering 面板开启 Paint flashing、Layer borders
- 检查动画属性是否触发布局

优化：

- 改用 `transform` / `opacity`
- 减少同时动画元素数量
- 避免大面积 `box-shadow` / `filter` 动画
- JS 动画用 rAF，避免读写交错
- 必要时降级动效

**口述模板（30–60 秒）**

1. 一句话结论：先录性能，再找 Layout/Paint 热点。
2. 关键机制：每帧预算约 16ms（60fps）。
3. 注意点：GPU 不是无限的。
4. 可追问时补充：低端安卓机专项测试。

**常见追问**

- CSS 动画一定比 JS 动画快吗？→ 不一定；关键是属性和是否阻塞主线程。

---

### Q23. 如何权衡「像素级还原」与「可维护性 / 响应式」？

- **难度**: senior
- **标签**: 工程取舍

**答案**

- 与设计约定**设计令牌**和组件规范，而不是每页 magic number
- 关键页面像素级，其余遵循系统组件
- 响应式优先保证可读与可用，而不是在所有尺寸像素一致
- 用视觉回归测试（Chromatic、Percy）防回归

**口述模板（30–60 秒）**

1. 一句话结论：像素级要在设计系统框架内实现。
2. 关键机制：令牌和组件减少一次性样式。
3. 注意点：硬编码尺寸是技术债来源。
4. 可追问时补充：设计验收流程。

**常见追问**

- 设计稿只有 375 一种尺寸？→ 和设计确认断点规则与流体策略。

---

### Q24. 打印样式、深色图片、高对比模式要注意什么？

- **难度**: senior
- **标签**: 媒体, 可访问性

**答案**

- 打印：`@media print` 隐藏导航、展开链接 URL、黑白可读
- 深色模式：图片和图标可能需要单独资源或 `filter`（慎用）
- Windows 高对比：`forced-colors` 媒体查询，避免只用背景色表达信息，使用系统颜色关键字

```css
@media (forced-colors: active) {
  .btn { border: 1px solid ButtonText; }
}
```

**口述模板（30–60 秒）**

1. 一句话结论：样式要考虑多种输出与用户偏好。
2. 关键机制：媒体查询区分打印、配色、强制颜色。
3. 注意点：forced-colors 下自定义颜色会被覆盖。
4. 可追问时补充：边框和图标保持可辨。

**常见追问**

- 只用 background 表示选中会怎样？→ 强制颜色模式下可能丢失状态。

---

### Q25. 如何看待 CSS 的未来：会替代多少 JS 交互？

- **难度**: senior
- **标签**: 趋势

**答案**

趋势：

- 定位：Anchor Positioning 替代部分 Floating UI
- 动效：Scroll-driven、View Transitions 替代部分动画库
- 结构：`:has()`、容器查询减少 JS 测量
- 组件：可定制 `select`、Popover 与 HTML 协同

但业务状态、数据请求、复杂手势、可访问组件逻辑仍需 JS。高级工程师应「能用 CSS 解决的交给 CSS，用 JS 做业务与降级」。

**口述模板（30–60 秒）**

1. 一句话结论：CSS 在吃掉表现层 JS，但不会替代业务逻辑。
2. 关键机制：浏览器原生能力更快、更稳定。
3. 注意点：新特性要渐进增强。
4. 可追问时补充：举一个你项目里用 CSS 替代 JS 的例子。

**常见追问**

- 团队如何引入新 CSS 特性？→ 设特性白名单、`@supports`、浏览器矩阵和灰度。
