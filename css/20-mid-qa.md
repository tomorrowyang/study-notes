# CSS 中级 Q&A（能讲清）

> 目标：BFC、层叠、Grid、响应式、渲染性能、工程化。面试最高频区。

---

## 一、格式化上下文与外边距

### Q1. 什么是 BFC？如何触发？解决什么问题？

- **难度**: mid
- **标签**: BFC

**答案**

BFC（Block Formatting Context，块级格式化上下文）是一块独立布局区域：内部块级盒按普通流规则排布，与外部浮动、外边距等相互隔离。

常见触发：

- `display: flow-root`（现代首选，无副作用）
- `overflow` 不为 `visible`（`hidden`/`auto`/`scroll`/`clip` 中部分组合）
- `float` 不为 `none`
- `position: absolute | fixed`
- `display: inline-block | table-cell | flex | grid` 等（Flex/Grid 是 FFC/GFC，但同样有隔离效果）
- `contain: layout | content | paint`

经典用途：

1. 包含浮动，父元素高度不塌陷
2. 阻止相邻块垂直外边距折叠（父子或兄弟）
3. 自适应两栏：BFC 区域不与浮动重叠

**口述模板（30–60 秒）**

1. 一句话结论：BFC 是布局隔离区。
2. 关键机制：内部规则自洽，与外部浮动/margin 隔离。
3. 注意点：优先 `flow-root`，慎用 `overflow:hidden` 裁内容。
4. 可追问时补充：Flex/Grid 子项为何不折叠 margin。

**常见追问**

- `display: flex` 算 BFC 吗？→ 严格说是 Flex 格式化上下文，但同样包含浮动、阻止子项 margin 与外部折叠类问题。

---

### Q2. 外边距折叠（Margin Collapse）在什么情况下发生？

- **难度**: mid
- **标签**: margin

**答案**

主要发生在**普通流**中**垂直方向**相邻块级盒之间：

1. 相邻兄弟的上下 margin
2. 父元素与第一个/最后一个子元素的 margin（父级无 border/padding/inline 内容/BFC 隔开时）
3. 空块自身上下 margin

结果取较大值（正负混合有规则）。不发生在：Flex/Grid 子项、浮动、绝对定位、`inline-block`、BFC 边界等。

**口述模板（30–60 秒）**

1. 一句话结论：普通流垂直相邻 margin 会合并。
2. 关键机制：父子穿透是常见坑。
3. 注意点：Flex/Grid 天然不折叠。
4. 可追问时补充：用 padding、border、flow-root 断开。

**常见追问**

- 水平 margin 会折叠吗？→ 不会。

---

### Q3. IFC、FFC、GFC 分别指什么？

- **难度**: mid
- **标签**: 格式化上下文

**答案**

- **IFC（Inline）**：行内盒在行框中水平排列，处理文字基线、行高
- **FFC（Flex）**：Flex 容器内部的一维布局规则
- **GFC（Grid）**：Grid 容器内部的二维网格规则

面试关键：不同上下文有不同算法；把元素放进 Flex/Grid 会「切换」子元素布局规则。

**口述模板（30–60 秒）**

1. 一句话结论：不同 display 进入不同格式化上下文。
2. 关键机制：上下文决定子元素怎么排。
3. 注意点：浮动在 Flex 子项上失效。
4. 可追问时补充：行内对齐 `vertical-align` 只在 IFC 有效。

**常见追问**

- 为何 `vertical-align` 对 div 无效？→ div 默认块级，不在行内格式化流里对齐。

---

## 二、层叠上下文与 z 轴

### Q4. 层叠上下文（Stacking Context）如何创建？层叠顺序是什么？

- **难度**: mid
- **标签**: 层叠上下文

**答案**

常见创建条件（非穷举）：

- 根元素 `html`
- `position` 非 `static` 且 `z-index` 不为 `auto`
- Flex/Grid 子项 `z-index` 不为 `auto`
- `opacity < 1`
- `transform`、`filter`、`backdrop-filter`、`perspective` 不为 none
- `will-change` 指定上述属性
- `isolation: isolate`
- `mix-blend-mode` 不为 normal
- `contain: layout | paint`、部分 `container-type`
- `position: fixed | sticky`（在多数现代实现中常新建）

同一上下文中，大致从下到上：背景与边框 → 负 z-index → 普通流块 → 浮动 → 普通流行内 → `z-index: 0/auto` 的定位 → 正 z-index。

**口述模板（30–60 秒）**

1. 一句话结论：z-index 只在同一层叠上下文里比大小。
2. 关键机制：子上下文作为整体参与父上下文排序。
3. 注意点：排障要沿祖先找谁建了上下文。
4. 可追问时补充：`isolation: isolate` 主动隔离。

**常见追问**

- 弹窗被盖住怎么修？→ 把弹窗挂到顶层（Portal / `dialog` top layer），或调整祖先上下文，而非无限加 z-index。

---

### Q5. `transform` 对 `position: fixed` 有什么影响？

- **难度**: mid
- **标签**: 定位, transform

**答案**

祖先若有 `transform`（以及 `filter`、`perspective`、`contain` 等部分属性），会成为 `fixed` 后代的**包含块**，`fixed` 不再相对视口，而相对该祖先定位。常见于「全屏遮罩在动画容器里不全屏」。

**口述模板（30–60 秒）**

1. 一句话结论：transform 祖先会「劫持」fixed 的参照。
2. 关键机制：包含块被改写。
3. 注意点：弹窗、Toast 应挂到 body 或 top layer。
4. 可追问时补充：同样影响 absolute 参照链与 stacking。

**常见追问**

- `will-change: transform` 也会吗？→ 会，同样可能创建包含块与层叠上下文。

---

## 三、层叠（Cascade）与特异性进阶

### Q6. 完整的层叠优先级大致是怎样的？

- **难度**: mid
- **标签**: Cascade

**答案**

简化顺序（高优先在后）：

1. 来源与重要性：UA 样式、用户样式、作者样式；`!important` 会反转部分层级
2. **`@layer` 层**：后声明的层优先级更高（未分层样式通常高于已分层，视规范与写法）
3. 特异性
4. 源码顺序

`!important` 在层中也有特殊规则（important 层的顺序与普通层相反）。工程上用 `@layer` 规定 reset → base → components → utilities，减少特异性战争。

**口述模板（30–60 秒）**

1. 一句话结论：来源/重要性 → 层 → 特异性 → 顺序。
2. 关键机制：`@layer` 把「谁覆盖谁」显式化。
3. 注意点：滥用 important 会难以覆盖。
4. 可追问时补充：第三方库包进低层。

**常见追问**

- 为什么 Tailwind 常配合 layer？→ 让 utilities 稳定覆盖组件默认样式。

---

### Q7. `:is()`、`:where()`、`:not()`、`:has()` 的特异性怎么算？

- **难度**: mid
- **标签**: 选择器

**答案**

- `:is(a, .b)`：特异性取参数里**最高**者
- `:where(...)`：特异性**恒为 0**，适合写可被轻松覆盖的默认样式
- `:not(...)`：取参数中最高特异性（不是 0）
- `:has(...)`：取参数中最高特异性

```css
:where(.card) .title { color: gray; } /* 易被覆盖 */
```

**口述模板（30–60 秒）**

1. 一句话结论：where 零特异性，is/not/has 跟参数走。
2. 关键机制：组合器内选择器参与计算。
3. 注意点：用 where 写 reset 更利于覆盖。
4. 可追问时补充：`:has()` 性能注意。

**常见追问**

- `:is` 和逗号分组区别？→ 无效选择器在逗号列表里会导致整条失效；`:is()` 容错更好。

---

## 四、Grid 布局

### Q8. Grid 和 Flex 有什么区别？怎么选？

- **难度**: mid
- **标签**: Grid, Flex

**答案**

| | Flex | Grid |
|--|------|------|
| 维度 | 一维（行或列） | 二维（行和列） |
| 驱动 | 内容优先，沿主轴分布 | 网格轨道优先 |
| 适合 | 导航、工具栏、卡片内排列 | 页面骨架、卡片墙、复杂对齐 |

实践：页面骨架 Grid，组件内部 Flex；二者常混用。

**口述模板（30–60 秒）**

1. 一句话结论：Flex 一维，Grid 二维。
2. 关键机制：Grid 先定义轨道再放置。
3. 注意点：不要为了 Grid 而 Grid。
4. 可追问时补充：subgrid 对齐子网格。

**常见追问**

- 能用 Flex 做二维吗？→ 能 wrap，但行间对齐控制弱于 Grid。

---

### Q9. `fr`、`minmax()`、`repeat(auto-fill/auto-fit)` 怎么用？

- **难度**: mid
- **标签**: Grid

**答案**

```css
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 16px;
}
```

- `fr`：剩余空间份额
- `minmax(min, max)`：轨道尺寸范围
- `auto-fill`：尽可能多建轨道，空轨道也保留
- `auto-fit`：同上，但空轨道塌陷为 0，已有项目会拉伸填满

**口述模板（30–60 秒）**

1. 一句话结论：auto-fill/fit + minmax 做响应式卡片墙。
2. 关键机制：浏览器根据容器宽算列数。
3. 注意点：fill 与 fit 在内容少时表现不同。
4. 可追问时补充：`grid-auto-flow: dense`。

**常见追问**

- `1fr` 和 `auto` 区别？→ `fr` 分剩余空间；`auto` 依内容尺寸。

---

### Q10. `grid-template-areas` 有什么用？

- **难度**: mid
- **标签**: Grid

**答案**

用字符串画布局，可读性强：

```css
.layout {
  display: grid;
  grid-template-columns: 240px 1fr;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
}
.header { grid-area: header; }
```

改布局时只改 areas 字符串，适合骨架。

**口述模板（30–60 秒）**

1. 一句话结论：areas 用名字表达区块位置。
2. 关键机制：同名区块必须是矩形。
3. 注意点：断点里重写 areas 做响应式。
4. 可追问时补充：与 DOM 顺序不同带来的可访问性风险。

**常见追问**

- 视觉顺序和 DOM 顺序不一致有问题吗？→ 键盘 Tab 与读屏按 DOM，视觉乱序会造成困惑。

---

### Q11. `subgrid` 解决什么问题？

- **难度**: mid
- **标签**: Grid, subgrid

**答案**

子网格可以继承父网格的行或列轨道，使嵌套卡片里的标题、正文、按钮跨卡片对齐，而不必写死高度。

```css
.cards { display: grid; grid-template-columns: repeat(3, 1fr); }
.card { display: grid; grid-row: span 3; grid-template-rows: subgrid; }
```

`subgrid` 已在现代浏览器广泛可用。

**口述模板（30–60 秒）**

1. 一句话结论：subgrid 让嵌套元素对齐外层网格线。
2. 关键机制：子网格复用父轨道。
3. 注意点：需理解跨越行数。
4. 可追问时补充：替代 JS 测高度对齐。

**常见追问**

- 不用 subgrid 怎么对齐卡片按钮？→ 常用 flex column + `margin-top: auto`，但多列间行对齐仍难。

---

## 五、响应式与现代单位

### Q12. 容器查询（Container Queries）和媒体查询有何不同？

- **难度**: mid
- **标签**: Container Queries

**答案**

- **媒体查询**：看视口 / 设备特性
- **容器查询**：看**最近声明了 container 的祖先**尺寸

```css
.card-wrap { container-type: inline-size; container-name: card; }
@container card (min-width: 400px) {
  .card { display: grid; grid-template-columns: 120px 1fr; }
}
```

组件放在侧栏或主栏时可自适应，是组件库级响应式核心能力。

**口述模板（30–60 秒）**

1. 一句话结论：容器查询让组件看父容器，不看整个屏幕。
2. 关键机制：`container-type` 建立查询容器。
3. 注意点：会创建包含与布局约束，注意与 sizing 循环。
4. 可追问时补充：`cqw` / `cqi` 单位。

**常见追问**

- `inline-size` 和 `size`？→ 前者只查宽度方向，更常用且不易造成高度循环。

---

### Q13. `clamp()`、`min()`、`max()` 怎么用于流体排版？

- **难度**: mid
- **标签**: 响应式

**答案**

```css
h1 { font-size: clamp(1.5rem, 1rem + 2vw, 2.5rem); }
.container { width: min(100% - 32px, 1200px); }
```

`clamp(MIN, PREFERRED, MAX)` 在区间内随视口变化，避免小屏太小、大屏太大。

**口述模板（30–60 秒）**

1. 一句话结论：clamp 是流体尺寸的安全带。
2. 关键机制：首选值可混合 rem 与 vw。
3. 注意点：纯 vw 字号对用户缩放不友好，宜混合 rem。
4. 可追问时补充：用户字号偏好与可访问性。

**常见追问**

- 只用 vw 做字号有什么问题？→ 用户放大文字时可能不按预期放大。

---

### Q14. 什么是逻辑属性（Logical Properties）？为什么要用？

- **难度**: mid
- **标签**: 国际化

**答案**

用书写方向相关属性替代物理方向：

| 物理 | 逻辑 |
|------|------|
| `margin-left` | `margin-inline-start` |
| `padding-top` | `padding-block-start` |
| `width` | `inline-size` |
| `left` | `inset-inline-start` |

在 `dir="rtl"` 或竖排时自动翻转，适合国际化与设计系统。

**口述模板（30–60 秒）**

1. 一句话结论：逻辑属性跟着书写方向走。
2. 关键机制：inline 是文字方向，block 是段落堆叠方向。
3. 注意点：图标箭头等仍可能需要单独镜像。
4. 可追问时补充：与 `dir` 配合。

**常见追问**

- `margin-inline: auto` 能居中吗？→ 在块级、宽度确定时可水平居中。

---

### Q15. 移动端 `100vh` 问题和 `dvh` / `svh` / `lvh` 怎么选？

- **难度**: mid
- **标签**: 视口单位

**答案**

移动浏览器地址栏伸缩导致视口高度变化：

- `svh`：最小视口（地址栏展开时）
- `lvh`：最大视口（地址栏收起）
- `dvh`：动态，跟随当前视口

全屏首屏常用 `min-height: 100dvh`；避免布局因地址栏抖动可用 `svh`。

**口述模板（30–60 秒）**

1. 一句话结论：移动端优先 dvh/svh 替代 100vh。
2. 关键机制：浏览器 UI 改变可见高度。
3. 注意点：dvh 可能随滚动变化引发重排。
4. 可追问时补充：`env(safe-area-inset-bottom)`。

**常见追问**

- iOS 底部 Home 条挡按钮？→ 加 `padding-bottom: env(safe-area-inset-bottom)`。

---

### Q16. 暗色模式怎么做？

- **难度**: mid
- **标签**: 主题

**答案**

```css
:root {
  color-scheme: light dark;
  --bg: #fff;
  --text: #111;
}
@media (prefers-color-scheme: dark) {
  :root { --bg: #0f1115; --text: #e8e8e8; }
}
[data-theme="dark"] { --bg: #0f1115; --text: #e8e8e8; }
body { background: var(--bg); color: var(--text); }
```

支持系统偏好 + 用户手动覆盖；`color-scheme` 让表单控件、滚动条随主题。

**口述模板（30–60 秒）**

1. 一句话结论：变量 + 媒体查询 + 手动开关。
2. 关键机制：主题切换改令牌而不改每个组件。
3. 注意点：图片、阴影、对比度要单独检查。
4. 可追问时补充：避免首屏闪烁（FOUC）用 head 内联脚本或 SSR。

**常见追问**

- 怎么避免刷新时闪白？→ 在首屏前确定主题（cookie / inline 脚本写 data-theme）。

---

## 六、渲染性能

### Q17. 重排（Reflow/Layout）和重绘（Repaint）是什么？

- **难度**: mid
- **标签**: 渲染性能

**答案**

- **重排**：几何信息变化（尺寸、位置、显示、字体），需要重新计算布局，开销大
- **重绘**：外观变化但不影响布局（颜色、背景、阴影部分情况）
- **合成（Composite）**：仅合成层变换，如 `transform`、`opacity`，通常最便宜

触发重排常见：改宽高、字号、增删 DOM、读取 `offsetWidth` 等布局属性（强制同步布局）。

**口述模板（30–60 秒）**

1. 一句话结论：布局变化最贵，合成最便宜。
2. 关键机制：Layout → Paint → Composite。
3. 注意点：读写交错会触发强制同步布局（Layout Thrashing）。
4. 可追问时补充：用 Performance 面板定位。

**常见追问**

- 改 class 一定会重排吗？→ 取决于新样式是否改几何属性。

---

### Q18. 为什么动画优先 `transform` 和 `opacity`？

- **难度**: mid
- **标签**: 动画性能

**答案**

它们通常可在合成线程处理，不触发布局和完整绘制；而 `top`、`left`、`width`、`height` 往往每帧重排。

```css
.slide { transition: transform 0.3s ease; }
.slide.active { transform: translateX(100%); }
```

**口述模板（30–60 秒）**

1. 一句话结论：transform/opacity 更容易走合成。
2. 关键机制：避开 Layout/Paint。
3. 注意点：不等于「绝对 GPU 加速」，也受层数与纹理影响。
4. 可追问时补充：will-change 与层爆炸。

**常见追问**

- `transform` 会不会触发重排？→ 一般不影响布局流，但可能产生新合成层与内存成本。

---

### Q19. `will-change` 应该怎么用？

- **难度**: mid
- **标签**: 性能

**答案**

提前告诉浏览器某属性即将变化，便于优化（如提升合成层）。

```css
.card:hover { will-change: transform; }
```

问题：

- 长期常驻会占内存（层爆炸）
- 会创建层叠上下文、改变 fixed 包含块

最佳：动画开始前加，结束后移除；不要全局 `* { will-change: transform }`。

**口述模板（30–60 秒）**

1. 一句话结论：will-change 是短期提示，不是万能加速。
2. 关键机制：浏览器可能预先分配层。
3. 注意点：滥用拖慢页面。
4. 可追问时补充：用 Layers 面板看层数。

**常见追问**

- `translateZ(0)` 黑科技呢？→ 类似强制提层，同样有内存与层叠副作用。

---

### Q20. `contain` 和 `content-visibility` 有什么用？

- **难度**: mid
- **标签**: 性能

**答案**

- `contain`：声明子树与外部的独立程度（`layout`、`paint`、`size`、`style`），减少布局影响范围
- `content-visibility: auto`：视口外内容跳过渲染工作，进入视口再渲染；常配 `contain-intrinsic-size` 防滚动条跳动

```css
.article-section {
  content-visibility: auto;
  contain-intrinsic-size: auto 800px;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：contain 缩小影响面，content-visibility 延迟离屏渲染。
2. 关键机制：浏览器可跳过子树布局/绘制。
3. 注意点：无 intrinsic size 会导致滚动跳动。
4. 可追问时补充：对查找（Ctrl+F）、锚点有影响时需测试。

**常见追问**

- 和虚拟列表关系？→ 都减少工作量；虚拟列表减 DOM，content-visibility 减渲染。

---

## 七、字体与资源

### Q21. `font-display` 各值什么意思？FOIT 和 FOUT 是什么？

- **难度**: mid
- **标签**: 字体

**答案**

- **FOIT**：字体加载前文字不可见（Flash of Invisible Text）
- **FOUT**：先显示后备字体，加载后替换（Flash of Unstyled Text）

`font-display`：

- `auto`：浏览器决定
- `block`：短时间隐形再交换
- `swap`：立即用后备，加载完交换（常用，可能 CLS）
- `fallback` / `optional`：更短窗口，optional 可能不交换

**口述模板（30–60 秒）**

1. 一句话结论：font-display 控制等字体时的显示策略。
2. 关键机制：交换周期与阻塞周期。
3. 注意点：swap 可能引起布局偏移，可配 `size-adjust`。
4. 可追问时补充：关键字体 preload。

**常见追问**

- 如何减轻字体 CLS？→ `size-adjust`、`ascent-override` 等让后备与 webfont 度量接近。

---

### Q22. 关键 CSS（Critical CSS）是什么？

- **难度**: mid
- **标签**: 首屏性能

**答案**

把首屏渲染必需的少量 CSS 内联到 HTML，其余样式异步或延迟加载，减少渲染阻塞时间。

```html
<style>/* 首屏骨架关键样式 */</style>
<link rel="preload" href="/app.css" as="style" onload="this.rel='stylesheet'" />
```

需工具抽取或手写维护；注意 FOUC 与重复样式。

**口述模板（30–60 秒）**

1. 一句话结论：首屏 CSS 内联，其余延后。
2. 关键机制：CSS 阻塞渲染，减小阻塞包。
3. 注意点：内联过大反而拖慢 HTML。
4. 可追问时补充：现代框架 SSR 常自动处理。

**常见追问**

- CSS 拆包按路由可以吗？→ 可以，按路由懒加载样式减少首包。

---

## 八、工程化

### Q23. BEM、CSS Modules、CSS-in-JS、Tailwind 怎么对比？

- **难度**: mid
- **标签**: 工程化

**答案**

| 方案 | 思路 | 优点 | 缺点 |
|------|------|------|------|
| BEM | 命名约定 | 无构建依赖、可读 | 靠纪律，易冗长 |
| CSS Modules | 编译时局部类名 | 天然隔离、零运行时 | 动态主题需变量配合 |
| CSS-in-JS | JS 生成样式 | 动态 props 方便 | 运行时成本、SSR 复杂 |
| Tailwind | 原子类 | 开发快、一致 | HTML 冗长、需设计约束 |

选型看团队规模、SSR、主题需求、性能预算。

**口述模板（30–60 秒）**

1. 一句话结论：没有银弹，按隔离需求与运行时成本选。
2. 关键机制：命名约定、编译期作用域、运行时注入、原子化。
3. 注意点：大型项目要统一设计令牌。
4. 可追问时补充：零运行时 CSS-in-JS（如 Vanilla Extract）。

**常见追问**

- 为什么一些团队弃用运行时 CSS-in-JS？→ 渲染时插样式影响性能与 React 18 并发等场景。

---

### Q24. 原生 CSS Nesting 和 Sass 嵌套有何区别？

- **难度**: mid
- **标签**: Nesting

**答案**

原生嵌套已在现代浏览器可用：

```css
.card {
  padding: 16px;
  & .title { font-weight: 600; }
  &:hover { box-shadow: 0 2px 8px rgb(0 0 0 / 12%); }
}
```

区别：

- 原生在浏览器解析，无需编译（仍可能用工具做降级）
- 语法细节与 Sass 不完全相同（例如对 `&` 与元素选择器开头的规则）
- Sass 还提供变量运算、mixin、函数等预处理器能力

**口述模板（30–60 秒）**

1. 一句话结论：原生嵌套替代了大部分「为嵌套而用 Sass」的场景。
2. 关键机制：浏览器展开嵌套选择器。
3. 注意点：嵌套过深会产生高特异性。
4. 可追问时补充：`@layer`、`@scope` 一起用。

**常见追问**

- 嵌套越深越好吗？→ 不是，会增加特异性和耦合。

---

### Q25. `:has()` 能做什么？有什么注意点？

- **难度**: mid
- **标签**: :has()

**答案**

`:has()` 根据后代或兄弟条件选择元素：

```css
.form-row:has(input:invalid) { border-color: crimson; }
.card:has(img) { padding-top: 0; }
```

注意：

- 能力强大，可能影响样式重算性能，避免在大范围通配上滥用
- 渐进使用，必要时 `@supports selector(:has(*))`

**口述模板（30–60 秒）**

1. 一句话结论：`:has()` 实现了父选择器类需求。
2. 关键机制：子孙状态影响祖先样式。
3. 注意点：复杂 `:has()` 可能增加匹配成本。
4. 可追问时补充：表单校验、卡片布局切换。

**常见追问**

- 能替代所有 JS 状态类吗？→ 不能，复杂业务状态仍应在逻辑层。

---

### Q26. `@layer` 怎么用？解决什么问题？

- **难度**: mid
- **标签**: @layer

**答案**

```css
@layer reset, base, components, utilities;

@layer reset { * { margin: 0; } }
@layer components { .btn { padding: 8px 12px; } }
@layer utilities { .p-0 { padding: 0; } }
```

后声明的层优先。第三方 UI 库放入低层，业务覆盖更可控，减少 `!important` 和长选择器。

**口述模板（30–60 秒）**

1. 一句话结论：@layer 显式管理样式优先级。
2. 关键机制：层顺序先于特异性参与层叠。
3. 注意点：未分层样式与分层样式的相对顺序要清楚。
4. 可追问时补充：import 第三方 CSS 到 layer。

**常见追问**

- `@import url(x.css) layer(vendor)`？→ 可把第三方样式直接放入层。

---

### Q27. `@property` 有什么用？

- **难度**: mid
- **标签**: @property

**答案**

注册自定义属性的类型、初始值、是否继承，使其可**插值动画**：

```css
@property --angle {
  syntax: '<angle>';
  inherits: false;
  initial-value: 0deg;
}
.spinner {
  background: conic-gradient(from var(--angle), red, blue);
  animation: rotate 2s linear infinite;
}
@keyframes rotate { to { --angle: 360deg; } }
```

**口述模板（30–60 秒）**

1. 一句话结论：@property 给 CSS 变量加类型。
2. 关键机制：浏览器知道如何插值。
3. 注意点：未注册变量动画多为跳变。
4. 可追问时补充：渐变边框动画常见。

**常见追问**

- 不支持怎么办？→ 用 `@supports` 降级为静态效果。

---

## 九、交互与细节

### Q28. `position: sticky` 为什么不生效？

- **难度**: mid
- **标签**: sticky

**答案**

常见原因：

1. 没有设置阈值，如 `top: 0`
2. 祖先有 `overflow: hidden/auto/scroll`，在错误的滚动容器内吸顶
3. 父元素高度不够，没有可粘滞空间
4. 表格里 sticky 需要作用到 `th` 等合适元素

**口述模板（30–60 秒）**

1. 一句话结论：sticky 依赖阈值和正确的滚动祖先。
2. 关键机制：在最近滚动容器内相对粘住。
3. 注意点：overflow 祖先最常坑。
4. 可追问时补充：表头 sticky 与 border-collapse。

**常见追问**

- 多个 sticky 叠加？→ 需各自 top 偏移，注意 z-index。

---

### Q29. 如何防止滚动穿透和弹窗滚动链？

- **难度**: mid
- **标签**: 滚动

**答案**

- 弹窗内容区：`overscroll-behavior: contain` 阻止滚动链到 body
- 打开模态时：锁定 body 滚动（注意 iOS 兼容与滚动位置恢复）
- 原生 `dialog` 模态有部分天然行为，仍要测试移动端

```css
.modal-body { overflow: auto; overscroll-behavior: contain; }
```

**口述模板（30–60 秒）**

1. 一句话结论：overscroll-behavior 控制滚动链。
2. 关键机制：到边界时是否把滚动传给祖先。
3. 注意点：iOS 锁滚动有历史坑。
4. 可追问时补充：滚动条宽度导致布局跳动用 `scrollbar-gutter`。

**常见追问**

- `scrollbar-gutter: stable`？→ 预留滚动条空间，避免出现时内容横跳。

---

### Q30. `scroll-snap` 适合什么场景？

- **难度**: mid
- **标签**: 滚动

**答案**

轮播、横向卡片列表、全屏分页：

```css
.track {
  display: flex;
  overflow-x: auto;
  scroll-snap-type: x mandatory;
}
.slide { scroll-snap-align: start; flex: 0 0 100%; }
```

比 JS 计算吸附更省事；注意可访问性与键盘控制。

**口述模板（30–60 秒）**

1. 一句话结论：scroll-snap 让滚动停在对齐点。
2. 关键机制：容器声明类型，子项声明对齐。
3. 注意点：mandatory 可能妨碍用户自由滚动。
4. 可追问时补充：配合 scroll-driven 动画做指示器。

**常见追问**

- 怎么做轮播自动播放？→ 仍需 JS 或滚动驱动方案，并提供暂停。

---

### Q31. 如何实现 1px 细线（高 DPR 屏）？

- **难度**: mid
- **标签**: 移动端

**答案**

常见方案：

- `transform: scaleY(0.5)` 伪元素
- `border-image` / 渐变背景模拟
- 部分环境直接 `0.5px`（兼容差异需测试）

```css
.hairline::after {
  content: "";
  position: absolute;
  left: 0; right: 0; bottom: 0;
  height: 1px;
  background: #ddd;
  transform: scaleY(0.5);
  transform-origin: bottom;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：高 DPR 下 CSS 1px 显得粗，需要缩放模拟物理 1px。
2. 关键机制：DPR 越大，1 CSS px 对应更多设备像素。
3. 注意点：容器需 `position: relative`。
4. 可追问时补充：设计系统统一 hairline 类。

**常见追问**

- 为何不用 `0.5px` 一刀切？→ 部分浏览器/系统渲染不稳定。

---

### Q32. `aspect-ratio` 解决什么？

- **难度**: mid
- **标签**: 尺寸, CLS

**答案**

用比例约束盒子尺寸，常用于视频、卡片封面、占位：

```css
.cover { aspect-ratio: 16 / 9; width: 100%; object-fit: cover; }
```

在图片加载前预留空间，配合 HTML `width`/`height` 改善 CLS。

**口述模板（30–60 秒）**

1. 一句话结论：aspect-ratio 固定宽高比。
2. 关键机制：一边确定时推导另一边。
3. 注意点：内容溢出时仍需 overflow 处理。
4. 可追问时补充：替代 padding-top 百分比 hack。

**常见追问**

- 和 padding-top: 56.25% 比？→ aspect-ratio 语义更清晰、可读性更好。
