# CSS 初级 Q&A（能用）

> 目标：属性用对、布局能搭。初级题也要答出「为什么」。

---

## 一、基础概念

### Q1. CSS 是什么？三种引入方式有什么区别？

- **难度**: junior
- **标签**: 概念

**答案**

CSS（Cascading Style Sheets）描述文档的**表现**。引入方式：

| 方式 | 写法 | 特点 |
|------|------|------|
| 外链 | `<link rel="stylesheet" href="a.css">` | 可缓存、可复用，生产首选 |
| 内嵌 | `<style>...</style>` | 适合关键 CSS 小块，增加 HTML 体积 |
| 行内 | `style="color:red"` | 特异性高、难维护，尽量少用 |

**口述模板（30–60 秒）**

1. 一句话结论：CSS 管表现，外链优先。
2. 关键机制：浏览器解析样式生成 CSSOM，再参与渲染。
3. 注意点：行内样式难维护且特异性高。
4. 可追问时补充：`@import` 可能延迟加载，生产慎用。

**常见追问**

- `link` 和 `@import` 区别？→ `link` 可并行下载；`@import` 常需等父样式解析完，可能更慢。

---

### Q2. 盒模型是什么？`content-box` 和 `border-box` 有什么区别？

- **难度**: junior
- **标签**: 盒模型

**答案**

每个元素可看成盒子：`content` + `padding` + `border` + `margin`。

- **content-box（默认）**：`width` / `height` 只含内容区；加 padding/border 会把总尺寸撑大
- **border-box**：`width` / `height` 含 content + padding + border；布局更直观

```css
*, *::before, *::after { box-sizing: border-box; }
```

**口述模板（30–60 秒）**

1. 一句话结论：border-box 让设宽高更符合直觉。
2. 关键机制：width 是否包含 padding/border 不同。
3. 注意点：margin 从不计入 width。
4. 可追问时补充：替换元素、百分比高度的坑。

**常见追问**

- `box-sizing` 会继承吗？→ 不继承，所以常用通配选择器设置。

---

### Q3. `margin`、`padding`、`border` 分别做什么？

- **难度**: junior
- **标签**: 盒模型

**答案**

- **margin**：盒子外侧空隙，可塌陷（普通流块级）
- **padding**：内容与边框之间的内边距，背景会延伸到 padding
- **border**：边框，占用尺寸（border-box 下计入 width）

垂直方向相邻块的 margin 可能折叠取较大值。

**口述模板（30–60 秒）**

1. 一句话结论：margin 在外，padding 在内，border 是边。
2. 关键机制：背景覆盖到 border 内侧（含 padding）。
3. 注意点：垂直 margin 可能折叠。
4. 可追问时补充：负 margin 的用途与风险。

**常见追问**

- 行内元素垂直 padding 会撑开行高吗？→ 视觉上可能溢出，但不一定改变行框布局，行为易踩坑；块级或 inline-block 更可控。

---

### Q4. `display` 常见值有哪些？

- **难度**: junior
- **标签**: display

**答案**

常用：

- `block`：独占一行，可设宽高
- `inline`：随文字流动，宽高设置受限
- `inline-block`：行内排列，又可设宽高
- `none`：不生成盒子，不占位（通常也不进无障碍树）
- `flex` / `inline-flex`：弹性布局
- `grid` / `inline-grid`：网格布局
- `contents`：子元素「提升」，自身盒子消失（有无障碍坑）
- `flow-root`：创建 BFC，无 overflow 副作用

**口述模板（30–60 秒）**

1. 一句话结论：display 决定盒子如何参与布局。
2. 关键机制：块、行内、弹性、网格是不同格式化上下文。
3. 注意点：`none` 与 `visibility:hidden` / `opacity:0` 不同。
4. 可追问时补充：`contents` 对无障碍的影响。

**常见追问**

- `visibility: hidden` 和 `display: none`？→ 前者占位、通常仍在无障碍树；后者不占位。

---

### Q5. 选择器有哪些常见类型？优先级怎么记？

- **难度**: junior
- **标签**: 选择器, 特异性

**答案**

常见：元素、类、ID、属性、伪类、伪元素、后代 / 子代 / 相邻 / 通用兄弟。

特异性（Specificity）粗记为四元组 `(a,b,c,d)`：

1. 行内 style → a
2. ID → b
3. 类 / 属性 / 伪类 → c
4. 元素 / 伪元素 → d

`!important` 另算；同等特异性时后者覆盖（层叠）。`:where()` 特异性为 0；`:is()` / `:has()` 取参数中最高者。

**口述模板（30–60 秒）**

1. 一句话结论：ID > 类 > 元素，再比源码顺序。
2. 关键机制：浏览器按特异性 + 层叠层 + 顺序解析冲突。
3. 注意点：别靠 `!important` 堆叠修样式。
4. 可追问时补充：`@layer` 可改变层间优先级。

**常见追问**

- `#app .btn` 和 `button.btn` 谁高？→ 前者含 ID，更高。

---

### Q6. 哪些属性可以继承？哪些不行？

- **难度**: junior
- **标签**: 继承

**答案**

通常**可继承**：`color`、`font-*`、`line-height`、`text-align`、`visibility`、`list-style` 等。

通常**不继承**：`margin`、`padding`、`border`、`width`、`height`、`display`、`position`、`background` 等。

可用 `inherit` / `initial` / `unset` / `revert` / `revert-layer` 控制。

**口述模板（30–60 秒）**

1. 一句话结论：文字相关多可继承，盒模型多不继承。
2. 关键机制：子元素拿父元素计算值。
3. 注意点：`a` 的颜色常被 UA 样式单独设定。
4. 可追问时补充：`all: unset` 的激进重置。

**常见追问**

- `line-height` 继承数字和百分比有何不同？→ 无单位数字会按子元素字号再算；百分比先算成父元素的绝对长度再继承。

---

## 二、定位与浮动

### Q7. `position` 各值是什么意思？

- **难度**: junior
- **标签**: 定位

**答案**

| 值 | 含义 |
|------|------|
| `static` | 默认，不定位，`top/z-index` 无效 |
| `relative` | 相对自身原位置偏移，仍占原空间 |
| `absolute` | 相对最近「定位祖先」（非 static）定位，脱离普通流 |
| `fixed` | 相对视口（或特定 containing block）定位 |
| `sticky` | 在阈值内像 relative，越过阈值后像 sticky 吸附 |

**口述模板（30–60 秒）**

1. 一句话结论：五种定位，核心是参照谁、是否脱离文档流。
2. 关键机制：absolute 找定位祖先；sticky 需要祖先可滚动且无 overflow hidden 误伤。
3. 注意点：sticky 失效常因父级 overflow。
4. 可追问时补充：`transform` 会改变 fixed 的包含块。

**常见追问**

- absolute 的包含块是什么？→ 最近非 static 祖先的 padding box（多数情况）。

---

### Q8. `z-index` 为什么有时「设很大也不生效」？

- **难度**: junior
- **标签**: z-index

**答案**

`z-index` 只对定位元素（及 flex/grid 子项等）有意义，且只在**同一层叠上下文**内比较。父级创建了新的层叠上下文后，子元素再大也无法「逃出」盖过另一整棵子树。

初级排查：祖先是否有 `opacity < 1`、`transform`、`filter`、`position + z-index` 等。

**口述模板（30–60 秒）**

1. 一句话结论：z-index 不是全局绝对分。
2. 关键机制：先比上下文，再比上下文内数字。
3. 注意点：往上找谁创建了上下文。
4. 可追问时补充：完整层叠顺序（背景、浮动、in-flow、定位）。

**常见追问**

- `z-index: auto` 和 `0` 区别？→ `auto` 通常不新建上下文；`0` 常会。

---

### Q9. 浮动（float）还有用吗？现代怎么清浮动？

- **难度**: junior
- **标签**: float

**答案**

浮动最初用于文字环绕图片。现代布局优先 Flex/Grid。仍会遇到历史代码或「文字环绕」需求。

父元素高度塌陷：浮动子元素脱离普通流。清浮动方式：

- 现代：父级 `display: flow-root`（推荐）
- 常见：`overflow: auto/hidden`（可能裁剪）
- 传统：clearfix 伪元素

**口述模板（30–60 秒）**

1. 一句话结论：浮动是历史方案，新布局少用。
2. 关键机制：浮动脱离流导致父高度塌陷。
3. 注意点：优先 flow-root / Flex / Grid。
4. 可追问时补充：BFC 包含浮动。

**常见追问**

- `clear: both` 作用？→ 元素自身下移到左右浮动之下。

---

### Q10. `overflow` 常见值与滚动容器？

- **难度**: junior
- **标签**: overflow

**答案**

`visible` / `hidden` / `scroll` / `auto` / `clip`。  
形成滚动容器后，子元素 `position: sticky`、滚动捕获等行为会变化。`overflow-x` 与 `overflow-y` 一个非 visible 时，另一个常被算成 `auto`。

**口述模板（30–60 秒）**

1. 一句话结论：overflow 控制溢出与是否可滚。
2. 关键机制：非 visible 常创建 BFC。
3. 注意点：hidden 可能裁掉下拉菜单。
4. 可追问时补充：`overscroll-behavior` 防滚动穿透。

**常见追问**

- 如何只允许纵向滚动？→ `overflow-x: hidden; overflow-y: auto;` 并注意双向耦合。

---

## 三、Flex 入门

### Q11. Flex 布局解决什么问题？常用属性有哪些？

- **难度**: junior
- **标签**: Flex

**答案**

Flex 是**一维**布局：主轴（main axis）与交叉轴（cross axis）。

容器常用：

- `display: flex`
- `flex-direction`、`flex-wrap`
- `justify-content`（主轴）、`align-items`（交叉轴）、`align-content`（多行）
- `gap`

项目常用：

- `flex-grow` / `flex-shrink` / `flex-basis`
- `align-self`
- `order`

```css
.row { display: flex; align-items: center; gap: 12px; }
```

**口述模板（30–60 秒）**

1. 一句话结论：Flex 管一行或一列的分布与对齐。
2. 关键机制：主轴分配剩余空间，交叉轴对齐。
3. 注意点：默认 `flex-shrink: 1`，子项可能被压扁。
4. 可追问时补充：`flex: 1` 等于 `1 1 0%`（视浏览器对 basis 解析）。

**常见追问**

- `space-between` 和 `gap` 一起用？→ 可以；gap 是现代首选间距。

---

### Q12. `flex: 1`、`flex: auto`、`flex: none` 分别什么意思？

- **难度**: junior
- **标签**: Flex

**答案**

简写 `flex-grow | flex-shrink | flex-basis`：

| 值 | 常见展开 | 含义 |
|------|----------|------|
| `flex: 1` | `1 1 0%` | 平分剩余空间，basis 从 0 起算 |
| `flex: auto` | `1 1 auto` | 按内容尺寸再伸缩 |
| `flex: none` | `0 0 auto` | 不伸缩，跟内容/设定宽 |

**口述模板（30–60 秒）**

1. 一句话结论：1 适合等分，auto 看内容，none 固定。
2. 关键机制：grow/shrink/basis 三者配合。
3. 注意点：内容过长仍可能溢出，需 `min-width: 0`。
4. 可追问时补充：为什么 flex 子项默认 `min-width: auto`。

**常见追问**

- 侧栏固定、主栏自适应怎么写？→ 侧栏 `flex: none; width: 240px;` 主栏 `flex: 1; min-width: 0;`。

---

### Q13. 如何用 Flex 做水平垂直居中？

- **难度**: junior
- **标签**: 居中

**答案**

```css
.center {
  display: flex;
  justify-content: center;
  align-items: center;
}
```

或 Grid：`display: grid; place-items: center;`。

**口述模板（30–60 秒）**

1. 一句话结论：Flex/Grid 居中最省事。
2. 关键机制：主轴与交叉轴都居中。
3. 注意点：父级要有高度才看得出垂直居中。
4. 可追问时补充：不定宽高 absolute + translate。

**常见追问**

- `margin: auto` 在 Flex 里？→ 可以吸收额外空间，实现推进对齐。

---

## 四、文字、颜色、背景

### Q14. `em`、`rem`、`%`、`vw` 有什么区别？

- **难度**: junior
- **标签**: 单位

**答案**

- `px`：绝对像素（仍受缩放影响）
- `em`：相对**当前元素**字号（嵌套会累乘）
- `rem`：相对**根元素** `html` 字号
- `%`：相对包含块或属性定义的参照
- `vw` / `vh`：视口宽高 1%
- 现代移动端更推荐 `svh` / `dvh` / `lvh` 处理浏览器 UI 栏

**口述模板（30–60 秒）**

1. 一句话结论：rem 做全局节奏，em 做局部相对。
2. 关键机制：参照对象不同。
3. 注意点：em 嵌套易失控。
4. 可追问时补充：`clamp(1rem, 2vw, 1.5rem)` 流体字号。

**常见追问**

- 为什么移动端 `100vh` 不准？→ 移动浏览器地址栏显隐导致；用 `dvh`/`svh`。

---

### Q15. `line-height` 怎么设比较好？

- **难度**: junior
- **标签**: 文字

**答案**

推荐无单位数字，如 `line-height: 1.5`，子元素按**各自字号**计算。用长度值会先算死再继承，嵌套标题可能不合适。

行高影响行框与垂直对齐；按钮垂直居中常靠 Flex，而不是只调 line-height。

**口述模板（30–60 秒）**

1. 一句话结论：无单位行高更可继承。
2. 关键机制：数字 × 元素自己的 font-size。
3. 注意点：单行截断、多行截断要另写。
4. 可追问时补充：半行高导致图标对不齐。

**常见追问**

- 多行省略？→ `-webkit-line-clamp` + `display: -webkit-box`（兼容性需知）。

---

### Q16. 颜色写法有哪些？什么是透明度？

- **难度**: junior
- **标签**: 颜色

**答案**

`#rgb` / `#rrggbb` / `#rrggbbaa`、`rgb()` / `rgba()`、`hsl()`、`hwb()`，现代还有 `oklch()`、`color-mix()`。

透明度：

- `opacity`：作用于整个元素含子孙，并可能创建层叠上下文
- `rgba` / `/ alpha`：只作用于该颜色

**口述模板（30–60 秒）**

1. 一句话结论：改颜色透明度用颜色函数，改整层用 opacity。
2. 关键机制：opacity 影响整棵子树合成。
3. 注意点：半透明文字用颜色 alpha 更常见。
4. 可追问时补充：对比度与无障碍。

**常见追问**

- `#fff8` 合法吗？→ 现代浏览器支持 4/8 位十六进制含 alpha。

---

### Q17. 背景相关常用属性？

- **难度**: junior
- **标签**: 背景

**答案**

`background-color`、`background-image`、`background-size`（`cover`/`contain`）、`background-position`、`background-repeat`、`background-attachment`，简写 `background`。

多层背景用逗号分隔。装饰性图片用 CSS 背景；有信息的图片用 `<img>` + alt。

**口述模板（30–60 秒）**

1. 一句话结论：背景可多层，size/position 控制裁剪。
2. 关键机制：cover 裁切填满，contain 完整显示。
3. 注意点：信息图不要只用背景图。
4. 可追问时补充：渐变也是 background-image。

**常见追问**

- `background-size: cover` 和 img object-fit？→ 概念类似，一个用于背景，一个用于替换元素。

---

## 五、伪类与伪元素

### Q18. 伪类和伪元素有什么区别？

- **难度**: junior
- **标签**: 选择器

**答案**

- **伪类**：选元素状态或结构位置，单冒号如 `:hover`、`:focus-visible`、`:nth-child`
- **伪元素**：创建「假元素」渲染盒，双冒号如 `::before`、`::after`、`::placeholder`

`::before` / `::after` 需要 `content`（可为空字符串）才会生成。

**口述模板（30–60 秒）**

1. 一句话结论：伪类选状态，伪元素造盒子。
2. 关键机制：伪元素可插图标、清浮动等。
3. 注意点：无障碍上伪元素内容可能被读或不被读，重要信息不只放 content。
4. 可追问时补充：`:is()` / `:where()` / `:has()`。

**常见追问**

- `:focus` 和 `:focus-visible`？→ 后者更适合「键盘才显示焦点环」。

---

### Q19. `:nth-child` 和 `:nth-of-type` 区别？

- **难度**: junior
- **标签**: 选择器

**答案**

- `:nth-child(2)`：父元素下第 2 个子节点（任何类型）且匹配前面选择器
- `:nth-of-type(2)`：父元素下同标签类型的第 2 个

中间若有别的标签，两者结果常不同。

**口述模板（30–60 秒）**

1. 一句话结论：child 数所有孩子，of-type 数同类型。
2. 关键机制：计数范围不同。
3. 注意点：列表里插了别的元素会踩坑。
4. 可追问时补充：`:nth-child(n of .item)` 较新语法。

**常见追问**

- 奇偶行斑马纹？→ `tr:nth-child(odd)`。

---

## 六、过渡动画入门

### Q20. `transition` 和 `animation` 有什么区别？

- **难度**: junior
- **标签**: 动效

**答案**

- **transition**：状态变化时平滑过渡（如 hover），一般有来有回
- **animation**：用 `@keyframes` 定义多关键时间轴，可循环、可延迟、可不依赖状态切换

```css
.btn { transition: transform 0.2s ease, background-color 0.2s; }
.pulse { animation: pulse 1s ease infinite; }
```

优先对 `transform`、`opacity` 做动画。

**口述模板（30–60 秒）**

1. 一句话结论：transition 跟状态，animation 跟时间轴。
2. 关键机制：都可插值可动画属性。
3. 注意点：`display` 等不可插值。
4. 可追问时补充：`prefers-reduced-motion`。

**常见追问**

- 所有属性都能 transition 吗？→ 只有可动画属性；否则瞬间跳变。

---

### Q21. 如何写一个简单的旋转加载动画？

- **难度**: junior
- **标签**: animation

**答案**

```css
@keyframes spin {
  to { transform: rotate(360deg); }
}
.loader {
  width: 24px;
  height: 24px;
  border: 3px solid #ddd;
  border-top-color: #333;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}
@media (prefers-reduced-motion: reduce) {
  .loader { animation: none; }
}
```

**口述模板（30–60 秒）**

1. 一句话结论：keyframes + transform 旋转。
2. 关键机制：linear infinite 匀速循环。
3. 注意点：尊重减少动效偏好。
4. 可追问时补充：用 SVG/SMIL 或纯 CSS 的替代。

**常见追问**

- 为什么不用 `left` 做位移动画？→ 触发布局，性能差。

---

## 七、响应式入门

### Q22. 什么是媒体查询？常用写法？

- **难度**: junior
- **标签**: 响应式

**答案**

```css
@media (max-width: 767px) {
  .nav { display: none; }
}
@media (min-width: 768px) and (hover: hover) {
  .card:hover { transform: translateY(-2px); }
}
```

还可查询 `prefers-color-scheme`、`prefers-reduced-motion`、`orientation` 等。

移动优先：先写小屏，再用 `min-width` 增强。

**口述模板（30–60 秒）**

1. 一句话结论：媒体查询按设备/偏好切换样式。
2. 关键机制：匹配时规则生效。
3. 注意点：断点应跟设计走，别抄死。
4. 可追问时补充：容器查询更适合组件级适配。

**常见追问**

- 断点写在设备像素还是 CSS 像素？→ 媒体查询用 CSS 像素（布局视口）。

---

### Q23. 移动端适配常见方案有哪些？（入门级）

- **难度**: junior
- **标签**: 移动端

**答案**

- 响应式 + rem/`clamp` 流体排版
- viewport meta（见 HTML 学习包）
- Flex/Grid 自适应
- 图片 `srcset`
- 安全区：`env(safe-area-inset-*)`

历史方案如整页 `transform: scale` 缩放、lib-flexible 等了解即可，新项目少用整页缩放。

**口述模板（30–60 秒）**

1. 一句话结论：现代以响应式布局 + 相对单位为主。
2. 关键机制：视口变化时重新布局。
3. 注意点：1px 线、安全区、横竖屏。
4. 可追问时补充：dpr 与 0.5px 边框技巧。

**常见追问**

- 禁止缩放好不好？→ 伤害无障碍，不推荐。

---

## 八、工程与调试入门

### Q24. 如何用开发者工具调试 CSS？

- **难度**: junior
- **标签**: 调试

**答案**

- Elements：看计算样式、盒模型、覆盖来源
- 切换伪类状态：`:hover` / `:focus`
- Layout：Flex/Grid 叠层可视化
- Rendering：刷图、层边界、FPS
- 删除/暂时注释规则定位来源

「样式不生效」检查：特异性、层叠层、是否被覆盖、选择器是否匹配、是否写错属性/单位。

**口述模板（30–60 秒）**

1. 一句话结论：先看计算样式和谁覆盖了谁。
2. 关键机制：DevTools 显示匹配规则链。
3. 注意点：区分「没匹配」和「被覆盖」。
4. 可追问时补充：强制同步布局的性能断点。

**常见追问**

- 计算样式里属性灰掉？→ 无效、被覆盖或不适用于该元素。

---

### Q25. CSS 变量（自定义属性）怎么用？

- **难度**: junior
- **标签**: 变量

**答案**

```css
:root {
  --color-text: #111;
  --space-md: 16px;
}
.title {
  color: var(--color-text);
  margin-bottom: var(--space-md, 12px); /* 回退值 */
}
.card {
  --color-text: #333; /* 局部覆盖，可继承 */
}
```

变量会继承；适合主题、间距刻度。用不存在的变量且无回退时，属性可能无效。

**口述模板（30–60 秒）**

1. 一句话结论：CSS 变量做主题和设计令牌。
2. 关键机制：运行时可继承、可覆盖。
3. 注意点：不是预处理器变量，选择器内动态有效。
4. 可追问时补充：`@property` 可规定类型与插值。

**常见追问**

- 能在 media 里改变量吗？→ 可以，很适合主题断点。

---

### Q26. `object-fit` 用在什么地方？

- **难度**: junior
- **标签**: 图片

**答案**

用于替换元素（`img`、`video`）在固定框内如何缩放：`fill` / `contain` / `cover` / `none` / `scale-down`。常配固定宽高或 aspect-ratio，避免拉伸变形。

**口述模板（30–60 秒）**

1. 一句话结论：object-fit 控制图片在盒子里怎么裁。
2. 关键机制：类似 background-size。
3. 注意点：要先有确定的内容盒尺寸。
4. 可追问时补充：`object-position`。

**常见追问**

- 和 background-size 怎么选？→ 需要 alt/SEO/内容图用 img；纯装饰可用背景。

---

### Q27. 隐藏元素有哪些方式？区别是什么？

- **难度**: junior
- **标签**: 显示隐藏

**答案**

| 方式 | 占位 | 交互 | 无障碍 |
|------|------|------|--------|
| `display: none` | 否 | 否 | 通常移除 |
| `visibility: hidden` | 是 | 否 | 通常隐藏 |
| `opacity: 0` | 是 | 默认可点中 | 仍可能读到 |
| 移出屏幕 / `clip` | 视实现 | 视实现 | 视觉隐藏技巧需谨慎 |
| `hidden` 属性 | 否 | 否 | 语义隐藏 |

**口述模板（30–60 秒）**

1. 一句话结论：按是否占位、是否可读选隐藏方式。
2. 关键机制：display 不生成盒子。
3. 注意点：opacity:0 仍可能被聚焦点击。
4. 可追问时补充：仅视觉隐藏但读屏可读的 `.sr-only`。

**常见追问**

- 弹窗关闭用哪种？→ 从无障碍树移除用 `hidden`/`display:none`；若需过渡再配合。

---

### Q28. `gap` 和用 margin 做间距有什么不同？

- **难度**: junior
- **标签**: Flex, Grid

**答案**

`gap` 定义 Flex/Grid 项目之间的间距，不会在首尾产生多余空白，也不会像子项 margin 那样容易折叠或与对齐纠缠。现代布局优先 `gap`。

**口述模板（30–60 秒）**

1. 一句话结论：间距优先用 gap。
2. 关键机制：容器负责分发间隙。
3. 注意点：旧浏览器需核对兼容（现已广泛）。
4. 可追问时补充：`row-gap` / `column-gap`。

**常见追问**

- 普通块级流程有 gap 吗？→ 有较新的能力，但传统还是 margin；Flex/Grid 最稳。

---

### Q29. `min-width` / `max-width` / `min-height` 常见用途？

- **难度**: junior
- **标签**: 尺寸

**答案**

限制元素收缩或扩张范围。响应式常用：

```css
.container { width: min(100%, 1200px); margin-inline: auto; }
.card { max-width: 100%; }
```

Flex 子项文字溢出时设 `min-width: 0`（或 `min-height: 0`）允许收缩到小于内容固有尺寸。

**口述模板（30–60 秒）**

1. 一句话结论：min/max 给尺寸加边界。
2. 关键机制：参与约束求解。
3. 注意点：flex 溢出常靠 min-width:0。
4. 可追问时补充：`min()` / `max()` / `clamp()` 函数。

**常见追问**

- `width: 100%` 和 `max-width: 100%`？→ 后者常用于防止图片撑破容器。

---

### Q30. 伪元素实现三角形 / 小箭头的原理是什么？

- **难度**: junior
- **标签**: 技巧

**答案**

经典技巧：元素宽高为 0，用透明 border 拼出三角形：

```css
.arrow {
  width: 0;
  height: 0;
  border: 6px solid transparent;
  border-top-color: #333;
}
```

现代也可用 `clip-path` 或 SVG，更直观。

**口述模板（30–60 秒）**

1. 一句话结论：零尺寸 + 透明边框可拼三角。
2. 关键机制：相邻 border 斜接成角。
3. 注意点：可访问装饰用，重要图形用 SVG。
4. 可追问时补充：气泡框三角定位。

**常见追问**

- 为什么现在更推 clip-path？→ 语义更直观，形状更灵活。

---

### Q31. `cursor`、`pointer-events`、`user-select` 做什么用？

- **难度**: junior
- **标签**: 交互

**答案**

- `cursor`：鼠标指针样式，可点元素常用 `pointer`
- `pointer-events: none`：不接收指针事件，事件穿透
- `user-select: none`：禁止选中文本（慎用，影响可访问与复制）

**口述模板（30–60 秒）**

1. 一句话结论：三者管指针形态、命中、选中。
2. 关键机制：pointer-events 改变命中测试。
3. 注意点：不要全局禁用选择。
4. 可追问时补充：遮罩层吞事件与穿透。

**常见追问**

- 禁用按钮只靠 pointer-events？→ 不够，还要用 `disabled` 或 `aria-disabled` 并处理键盘。
