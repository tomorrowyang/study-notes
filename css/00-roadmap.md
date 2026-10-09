# CSS 学习路线（高级面试向）

> 目标：能在国内互联网高级前端面试中讲清盒模型、层叠、BFC、Flex/Grid、响应式、动画性能与现代 CSS 能力，并能落到组件库与大型项目工程实践。

## 面试画像

- **岗位侧重**：前端 / 全栈；高级岗会把 CSS 和浏览器渲染、CLS、设计系统一起问
- **假定基础**：会写页面，了解 HTML 语义，用过 React / Vue 任一框架
- **学完应能**：
  - 讲清盒模型、BFC、层叠上下文、选择器特异性，而不是只背属性
  - 对比 Flex / Grid / 定位，并说明选型取舍
  - 从 Layout → Paint → Composite 讲动画为什么优先 `transform` / `opacity`
  - 设计移动端适配、容器查询、主题与暗色方案
  - 了解 `:has()`、`@layer`、Nesting、Anchor Positioning、View Transitions 等现代能力与兼容策略

## 版本说明（本文默认语境）

- CSS 以 **CSS Working Group / MDN** 为准，按能力而不是「CSS3 版本号」理解
- 兼容性以 MDN Baseline 为准（约 2025–2026）：
  - **广泛可用**：Container Queries、`:has()`、Nesting、`@layer`、`@property`、`color-mix()`、`subgrid`、Logical Properties、`clamp()`
  - **Newly available / 渐进增强**：Anchor Positioning、Scroll-driven Animations、View Transitions（跨文档导航视浏览器而定）
- 生产原则：核心布局用稳定能力；新能力用 `@supports` 渐进增强

## 知识地图

- **基础**
  - 盒模型、`box-sizing`、替换元素
  - 选择器、特异性（Specificity）、层叠（Cascade）、`!important`
  - 继承、初始值、`unset` / `revert`
- **布局**
  - 普通流、浮动（历史）、定位
  - BFC / IFC / FFC / GFC
  - Flexbox、Grid、`subgrid`
  - 多列、Sticky、含粘性页脚等经典布局
- **视觉与响应**
  - 单位：`px` / `em` / `rem` / `%` / `vw` / `svh` / `dvh` / `cqw`
  - Media Queries、Container Queries
  - 逻辑属性（Logical Properties）
  - 颜色：RGB、HSL、OKLCH、`color-mix`
- **层叠与合成**
  - 层叠上下文、`z-index` 排障
  - 合成层、`will-change`、重排 / 重绘
- **动效**
  - `transition` / `animation` / `@keyframes`
  - Scroll-driven Animations、View Transitions（新）
- **工程化**
  - CSS Modules、CSS-in-JS、Tailwind、设计令牌
  - `@layer`、命名约定、主题切换
  - 关键 CSS、CLS、字体加载
- **现代能力**
  - `:has()`、Nesting、`@scope`
  - Anchor Positioning、Popover 定位
  - 可定制 `<select>` 等（查兼容）

## 建议阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 盒模型、选择器、定位、Flex 入门
2. [中级 Q&A](20-mid-qa.md) — BFC、层叠、Grid、响应式、动画
3. [高级 Q&A](30-senior-qa.md) — 渲染性能、设计系统、现代 CSS、架构取舍
4. [手写与现场编码](40-coding-handwritten.md) — 居中、圣杯、组件、动效
5. [系统设计与场景](50-system-design.md) — 主题、适配、组件库、CLS 治理
6. [真实/高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. 边学边勾 [PROGRESS.md](PROGRESS.md)

## 高级面试必打点

1. 盒模型：`content-box` vs `border-box`；标准做法 `* { box-sizing: border-box }`
2. BFC 是布局隔离；现代优先 `display: flow-root` / Flex / Grid，慎用「靠 overflow:hidden 清浮动」当唯一手段
3. Flex 一维，Grid 二维；页面骨架 Grid、组件内 Flex 很常见
4. 层叠上下文：`z-index` 只在同一上下文内比较；`opacity`/`transform`/`filter` 会新建上下文
5. 外边距折叠（Margin Collapse）只发生在普通流相邻块之间；Flex/Grid 子项不折叠
6. 特异性不是越大越好；用 `@layer` 和低特异性选择器控制可维护性
7. 动画优先改 `transform`/`opacity`，避免每帧改 `width`/`top` 触发布局
8. `will-change` 是提示不是加速开关，过度使用浪费内存
9. `rem` 相对根元素，`em` 相对父级；响应式字号常用 `clamp()`
10. 容器查询让组件按自身容器适配，而不是只听视口
11. `:has()` 是「父选择器」能力，但代价是样式匹配成本，避免滥用
12. 逻辑属性服务 RTL / 国际化：`margin-inline-start` 优于死写 `margin-left`
13. 暗色模式：系统偏好 + 手动切换；CSS 变量做主题令牌
14. CLS：图片宽高、字体换行、动态插入都要预留空间
15. 关键 CSS 与非关键 CSS 拆分，避免阻塞首屏渲染
16. Anchor Positioning 逐步替代「JS 算气泡位置」；生产需 `@supports` 兜底
17. View Transitions 可做 SPA / 跨文档过渡，但要注意可访问性与降级
18. CSS 工程选型（Modules / Tailwind / CSS-in-JS）看团队边界与性能，没有银弹

## 关联技术

- **HTML**：语义结构、`dialog` / Popover、图片尺寸与 CLS
- **浏览器原理**：Layout / Paint / Composite、合成层
- **JS**：ResizeObserver、IntersectionObserver、样式读取导致强制同步布局
- **设计系统**：Design Tokens、组件变体、无障碍对比度
- **构建**：PostCSS、Lightning CSS、CSS Modules、Tailwind

## 自测清单（学完打勾）

- [ ] 能白板画出 BFC 触发条件与三个经典应用
- [ ] 能解释「z-index 很大仍被盖住」的排查路径
- [ ] 能手写 Flex 双栏、Grid 圣杯、Sticky 表头
- [ ] 能讲清动画卡顿的定位方法与改法
- [ ] 能设计一套基于 CSS 变量的明暗主题方案
