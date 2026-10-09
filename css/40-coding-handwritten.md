# CSS 手写与现场编码

> 使用方式：先自己写并讲思路，再对照参考实现。面试官关注：方案是否现代、是否考虑边界、是否解释取舍。

---

### H1. 实现水平垂直居中（至少 4 种）

- **难度**: junior
- **标签**: 居中
- **面试官看什么**: 方案广度与适用场景

**题干**

子元素宽高未知，在父容器中水平垂直居中。

**思路口述**

1. Flex 最通用
2. Grid 最简短
3. absolute + transform 适合覆盖层
4. 定宽高时 absolute + margin auto

**参考实现**

```css
/* 1. Flex */
.parent { display: flex; justify-content: center; align-items: center; }

/* 2. Grid */
.parent { display: grid; place-items: center; }

/* 3. absolute + transform（宽高未知） */
.parent { position: relative; }
.child {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

/* 4. absolute + inset + margin（需宽高已知） */
.child {
  position: absolute;
  inset: 0;
  width: 200px;
  height: 100px;
  margin: auto;
}
```

**易错点**

- 父元素没有高度，垂直居中看不出
- transform 方案可能导致子元素模糊（亚像素）
- 只会一种方案

---

### H2. 左侧固定 240px，右侧自适应（多种实现）

- **难度**: junior
- **标签**: 两栏布局
- **面试官看什么**: Flex/Grid/BFC 理解

**题干**

左侧导航宽 240px，右侧内容占满剩余宽度，右侧内容过长时不撑破布局。

**参考实现**

```css
/* Flex */
.layout { display: flex; }
.sidebar { flex: none; width: 240px; }
.main { flex: 1; min-width: 0; }

/* Grid */
.layout {
  display: grid;
  grid-template-columns: 240px minmax(0, 1fr);
}

/* 浮动 + BFC（历史方案） */
.sidebar { float: left; width: 240px; }
.main { display: flow-root; }
```

**易错点**

- 忘记 `min-width: 0`，长单词或表格撑破
- 使用 `width: calc(100% - 240px)` 忽略 gap 与边框
- 历史浮动方案未说明 BFC 原理

---

### H3. 圣杯 / 双飞翼三栏布局（现代写法）

- **难度**: mid
- **标签**: Grid, 布局
- **面试官看什么**: 能否用现代方案替代旧 hack

**题干**

页面包含 header、左栏、主内容、右栏、footer。主内容优先显示，左右栏固定宽度，中间自适应；移动端改为单列。

**参考实现**

```css
.page {
  min-height: 100dvh;
  display: grid;
  grid-template-columns: 200px minmax(0, 1fr) 240px;
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header header header"
    "left   main   right"
    "footer footer footer";
  gap: 16px;
}
.header { grid-area: header; }
.left   { grid-area: left; }
.main   { grid-area: main; }
.right  { grid-area: right; }
.footer { grid-area: footer; }

@media (max-width: 767px) {
  .page {
    grid-template-columns: 1fr;
    grid-template-areas:
      "header"
      "main"
      "left"
      "right"
      "footer";
  }
}
```

**易错点**

- 视觉顺序和 DOM 顺序差异过大，影响键盘和读屏顺序
- 中间列不用 `minmax(0, 1fr)` 导致溢出
- 只背旧的负 margin 圣杯写法，不知道 Grid 方案

---

### H4. 响应式卡片墙 + 组件级容器查询

- **难度**: mid
- **标签**: Grid, Container Queries
- **面试官看什么**: auto-fill/minmax、容器查询

**题干**

卡片列表根据容器宽度自动分列；单张卡片在宽度足够时改为左右布局（图片左、文字右）。

**参考实现**

```css
.card-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(min(100%, 260px), 1fr));
  gap: 16px;
}

.card-wrap {
  container-type: inline-size;
}

.card {
  display: grid;
  gap: 12px;
}

.card img {
  width: 100%;
  aspect-ratio: 16 / 9;
  object-fit: cover;
}

@container (min-width: 420px) {
  .card {
    grid-template-columns: 160px 1fr;
    align-items: start;
  }
}
```

```html
<div class="card-list">
  <div class="card-wrap">
    <article class="card">
      <img src="cover.jpg" width="640" height="360" alt="课程封面" />
      <div>
        <h3>CSS 进阶</h3>
        <p>容器查询与现代布局</p>
      </div>
    </article>
  </div>
</div>
```

**易错点**

- 在卡片自身设置 `container-type` 后又用它查询自身尺寸，导致理解混乱
- `minmax(260px, 1fr)` 在窄屏溢出，可用 `min(100%, 260px)`
- 图片未设比例，引发 CLS

---

### H5. 修复「父元素高度塌陷」与「margin 穿透」

- **难度**: mid
- **标签**: BFC, margin
- **面试官看什么**: BFC 实战与现代替代方案

**题干**

```html
<div class="parent">
  <div class="child">内容</div>
</div>
```

```css
.parent { background: #eee; }
.child { margin-top: 20px; float: left; }
```

问题 1：父元素高度为 0。问题 2：若去掉 float，子元素 margin-top 穿透到父外。请修复并解释。

**参考实现**

```css
/* 包含浮动、阻断 margin 穿透 */
.parent {
  display: flow-root;
  background: #eee;
}

/* 若不再需要浮动，更推荐直接改布局 */
.parent {
  display: flex;
  flex-direction: column;
  background: #eee;
}
.child { margin-top: 20px; }
```

**易错点**

- 只会 `overflow: hidden`，不知道可能裁剪下拉框
- 用空 div clear 而不解释原理
- 不知道 Flex 子项 margin 不折叠

---

### H6. 用 Anchor Positioning 实现 Tooltip（带回退）

- **难度**: senior
- **标签**: Anchor Positioning, 渐进增强
- **面试官看什么**: 新特性与渐进增强、键盘可用

**题干**

按钮悬停或键盘聚焦时显示提示；支持锚点定位的浏览器用 CSS 锚点定位并在空间不足时翻转，不支持时退化为按钮下方绝对定位。

**思路口述**

1. 基础版：包裹元素 `position: relative`，提示 `position: absolute`，用 `:hover` / `:focus-within` 显示
2. 增强版：`@supports` 检测锚点定位，用 `anchor-name` / `position-anchor` / `position-area` 定位
3. 用 `position-try-fallbacks` 处理视口边缘翻转
4. 用 `aria-describedby` 关联提示文字

**参考实现**

```html
<span class="tip-wrap">
  <button class="tip-trigger" type="button" aria-describedby="tip-save">保存</button>
  <span id="tip-save" class="tip" role="tooltip">保存草稿到云端</span>
</span>
```

```css
.tip-wrap {
  position: relative;
  display: inline-block;
}

.tip {
  position: absolute;
  top: calc(100% + 6px);
  left: 0;
  padding: 6px 8px;
  border-radius: 6px;
  background: #222;
  color: #fff;
  font-size: 12px;
  white-space: nowrap;
  opacity: 0;
  visibility: hidden;
  transition: opacity 0.15s;
}

.tip-wrap:hover .tip,
.tip-wrap:focus-within .tip {
  opacity: 1;
  visibility: visible;
}

@supports (anchor-name: --a) {
  .tip-trigger {
    anchor-name: --save-btn;
  }
  .tip {
    position-anchor: --save-btn;
    top: auto;
    left: auto;
    margin-top: 6px;
    position-area: bottom;
    position-try-fallbacks: flip-block;
  }
}
```

说明：如果改用 `popover`，弹层显示时会进入 top layer，包含块变为视口，基础版的「相对包裹元素绝对定位」不再成立；此时回退方案应改为 JS 计算位置（如 Floating UI）。`popover="hint"` + `interestfor` 是更新的声明式悬停方案，生产使用前查兼容表。

**易错点**

- 不做 `@supports` 回退
- Tooltip 只依赖 hover，键盘用户无法触发
- 给 tooltip 放可交互内容（应改为 popover 菜单或 dialog）
- 父级 `overflow: hidden` 裁掉基础版提示框

---

### H7. 用 CSS 变量实现明暗主题切换

- **难度**: mid
- **标签**: 主题, 变量
- **面试官看什么**: 令牌设计、系统偏好、避免闪烁

**题干**

支持跟随系统和手动切换明 / 暗主题，组件只使用语义变量。

**参考实现**

```css
:root {
  color-scheme: light dark;

  --surface: #ffffff;
  --text: #1a1a1a;
  --muted: #666;
  --border: #e5e5e5;
  --primary: oklch(55% 0.18 250);
}

@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --surface: #121212;
    --text: #f1f1f1;
    --muted: #a3a3a3;
    --border: #2a2a2a;
    --primary: oklch(72% 0.14 250);
  }
}

:root[data-theme="dark"] {
  --surface: #121212;
  --text: #f1f1f1;
  --muted: #a3a3a3;
  --border: #2a2a2a;
  --primary: oklch(72% 0.14 250);
}

body {
  background: var(--surface);
  color: var(--text);
}

.button {
  background: var(--primary);
  color: white;
  border: 1px solid var(--border);
}
```

```html
<script>
  // 在样式渲染前确定主题，避免首屏闪烁
  const saved = localStorage.getItem('theme')
  if (saved) document.documentElement.dataset.theme = saved
</script>
```

**易错点**

- 组件直接写死颜色
- 手动主题与系统主题优先级不清
- 主题脚本放到 body 底部导致闪白

---

### H8. 骨架屏 Shimmer 动画（只动 transform）

- **难度**: mid
- **标签**: 动画性能
- **面试官看什么**: 动画属性选择、减少重排

**题干**

实现列表骨架屏的闪光扫过效果，要求性能良好并尊重减少动效偏好。

**参考实现**

```css
.skeleton {
  position: relative;
  overflow: hidden;
  background: #e9e9e9;
  border-radius: 6px;
}

.skeleton::after {
  content: "";
  position: absolute;
  inset: 0;
  transform: translateX(-100%);
  background: linear-gradient(
    90deg,
    transparent,
    rgb(255 255 255 / 60%),
    transparent
  );
  animation: shimmer 1.2s ease-in-out infinite;
}

@keyframes shimmer {
  to { transform: translateX(100%); }
}

.skeleton-title { height: 20px; width: 60%; }
.skeleton-line { height: 14px; width: 100%; margin-top: 8px; }

@media (prefers-reduced-motion: reduce) {
  .skeleton::after { animation: none; }
}
```

**易错点**

- 用 `background-position` 动画大面积渐变（可能更耗绘制）
- 骨架尺寸与真实内容差异大，加载后 CLS
- 读屏用户不知道在加载，应配合 `aria-busy` 或状态文字

---

### H9. 自定义复选框样式且保留原生可访问性

- **难度**: mid
- **标签**: 表单, a11y
- **面试官看什么**: 不破坏原生 input

**题干**

把原生复选框改成品牌样式，要求可键盘操作、可读屏、可显示焦点。

**参考实现**

```html
<label class="checkbox">
  <input type="checkbox" name="agree" />
  <span class="checkbox-ui" aria-hidden="true"></span>
  <span>同意用户协议</span>
</label>
```

```css
.checkbox {
  position: relative;
  display: inline-flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
}

.checkbox input {
  position: absolute;
  opacity: 0;
  width: 1px;
  height: 1px;
}

.checkbox-ui {
  width: 18px;
  height: 18px;
  border: 2px solid #666;
  border-radius: 4px;
  display: grid;
  place-items: center;
}

.checkbox input:checked + .checkbox-ui {
  background: #2563eb;
  border-color: #2563eb;
}

.checkbox input:checked + .checkbox-ui::after {
  content: "";
  width: 5px;
  height: 9px;
  border: solid white;
  border-width: 0 2px 2px 0;
  transform: rotate(45deg);
}

.checkbox input:focus-visible + .checkbox-ui {
  outline: 2px solid #2563eb;
  outline-offset: 2px;
}

.checkbox input:disabled + .checkbox-ui {
  opacity: 0.5;
}
```

更现代的做法也可直接对原生控件使用 `appearance: none` 自绘，同时保留 input 本身。

**易错点**

- `display: none` 隐藏 input，导致不可聚焦、不可读屏
- 只有视觉勾选，没有 checked 状态
- 去掉焦点环

---

### H10. 审查并修复 z-index / 层叠问题

- **难度**: senior
- **标签**: 层叠上下文, 排障
- **面试官看什么**: 能否定位上下文根因

**题干**

```html
<div class="header">
  <div class="dropdown">下拉菜单</div>
</div>
<main class="content">
  <div class="card">卡片</div>
</main>
```

```css
.header { position: relative; z-index: 10; transform: translateZ(0); }
.dropdown { position: absolute; top: 100%; z-index: 9999; }
.content { position: relative; z-index: 20; }
```

现象：下拉菜单被 `.content` 盖住。请解释并修复。

**分析**

1. `.header` 有 `z-index: 10` 且定位，已创建层叠上下文
2. `transform` 也使 `.header` 成为层叠上下文
3. `.dropdown` 的 `9999` 只在 `.header` 内部比较
4. `.content` 在根上下文中 `z-index: 20 > 10`，整体盖住 header 及其子树

**参考修复**

```css
/* 方案 1：提高 header 所在层级，统一 z-index 令牌 */
.header { position: relative; z-index: 30; }
.content { position: relative; z-index: 20; }

/* 方案 2：移除不必要的 transform，避免意外上下文 */
.header { position: relative; z-index: 30; }

/* 方案 3：下拉菜单挂载到 body / 使用 popover top layer */
```

**易错点**

- 把 dropdown 改成 `99999`
- 不知道 transform 也会创建层叠上下文
- 没有团队级 z-index 规范
