# 前端性能优化学习路线（高级工程师面试向）

## 面试画像

性能优化是国内高级前端面试**几乎必考**的综合题，通常不会只问「有哪些优化手段」，而是围绕一条闭环追问：

- **指标**：LCP / INP / CLS / TTFB / FCP 是什么，阈值多少，怎么采集，实验室数据和真实用户数据的区别 —— 一面基础
- **定位**：Network 瀑布、Performance 面板、长任务、LoAF、Lighthouse、RUM 分位数 —— 中级必问
- **优化**：加载链路（网络、资源、关键渲染路径）、渲染（重排重绘、合成层）、JS 执行（长任务、调度、Worker）、框架层（React / Vue）—— 中高级必问
- **治理**：性能监控平台、性能预算、CI 门禁、防劣化、用数据讲项目收益 —— 高级岗必谈

高级岗典型问法：

> 「你做过的性能优化项目，指标从多少到多少？怎么证明是你的优化带来的？」「首屏 LCP 4 秒，你怎么排查？」「页面点击卡顿，INP 很差，怎么定位？」「如何设计一个性能监控系统？」「长列表 10 万条数据怎么渲染？」「SSR 一定比 CSR 快吗？」

## 版本说明（截至 2026-10）

| 时间 | 变化 |
|------|------|
| 2020-05 | Google 提出 Web Vitals：LCP、FID、CLS |
| 2021-06 | CLS 改为「最大会话窗口」计算，长时间停留的页面不再无限累加 |
| **2024-03** | **INP 取代 FID** 成为 Core Web Vitals |
| 2024-03 | Chrome 123 推出 **Long Animation Frames（LoAF）API**，比 Long Tasks 更适合定位 INP 问题 |
| 2024-09 | Chrome 129 推出 **`scheduler.yield()`**；Firefox 在 2025 年跟进，Safari 暂不支持（以 MDN 为准） |
| 2024 | `fetchpriority` 成为 Baseline；跨文档 View Transitions（Chrome 126+、Safari 18.2+） |
| 2024-01 / 2025-10 / 2025-12 | Firefox 122 支持 LCP；Firefox 144 支持 INP；**Safari 26.2 支持 LCP 和 INP** → LCP、INP 成为 Baseline Newly available |
| 2026-07 | Chrome 151 正式支持 **Soft Navigations**（SPA 路由切换单独统计 LCP / INP / CLS），web-vitals 库发布 v6 支持软导航（以官方为准） |
| 现状 | 阈值未变：**LCP ≤ 2.5s、INP ≤ 200ms、CLS ≤ 0.1**，按真实用户 **P75** 评估；**CLS 仍只有 Chromium 支持** |

> 面试中如果还只说 FID、TTI，会显得资料陈旧。TTI 已不再是 Lighthouse 的主要评分项，实验室中用 TBT 作为 INP 的代理指标。Speculation Rules（预渲染）目前只有 Chromium 支持，需要特性检测。

## 知识地图

```text
前端性能优化
├─ 指标体系：Core Web Vitals（LCP / INP / CLS）、TTFB、FCP、TBT、业务指标（首屏可用、白屏率）
├─ 度量工具：Lighthouse、PageSpeed Insights、CrUX、DevTools（Network / Performance / Insights）、WebPageTest、web-vitals
├─ 网络层：DNS / TCP / TLS、HTTP/2 / HTTP/3、CDN、压缩（Brotli / zstd）、缓存（强缓存 / 协商缓存 / Service Worker）、103 Early Hints
├─ 资源层：体积（Tree Shaking、代码分割、依赖治理）、图片（AVIF / WebP、响应式、懒加载）、字体、第三方脚本
├─ 关键渲染路径：阻塞资源、preload / preconnect / fetchpriority、关键 CSS、SSR / SSG / 流式渲染、预渲染
├─ 渲染性能：重排 / 重绘 / 合成、强制同步布局、合成层、content-visibility、动画
├─ JS 执行：长任务、任务拆分、scheduler.yield、Web Worker、内存泄漏
├─ 框架层：React（memo、并发特性、Compiler、RSC）、Vue（响应式粒度、v-memo）、虚拟列表
└─ 工程治理：RUM 监控平台、性能预算、CI 门禁、A/B 验证、性能文化
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 指标定义、常用工具、常见优化手段
2. [中级 Q&A](20-mid-qa.md) — 各指标的拆解与优化、关键渲染路径、缓存、渲染性能、长任务、框架优化
3. [高级 Q&A](30-senior-qa.md) — 指标的计算细节、LoAF 定位、RUM 数据分析、复杂场景（SSR、微前端、低端机）、治理体系
4. [手写与现场编码](40-coding-handwritten.md) — 12 道题：防抖节流、虚拟列表、任务调度、图片懒加载、指标采集 SDK 等
5. [系统设计与场景](50-system-design.md) — 性能监控平台、首屏优化、大数据量表格、性能预算体系等
6. [真实 / 高频面试题](90-interview-real.md)
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. 三个 Core Web Vitals 的定义、阈值、P75 评估方式，以及 INP 取代 FID 的原因
2. 实验室数据（Lighthouse）与真实用户数据（RUM、CrUX）的区别和各自用途
3. LCP 的四段拆解：TTFB → 资源加载延迟 → 资源加载耗时 → 元素渲染延迟
4. INP 的三段拆解：输入延迟 → 事件处理耗时 → 呈现延迟
5. CLS 的计算方式（影响比例 × 距离比例，会话窗口）和常见原因
6. 关键渲染路径：CSS 阻塞渲染、JS 阻塞解析、`async` / `defer` / `module`
7. 资源提示：`preload`、`preconnect`、`dns-prefetch`、`prefetch`、`fetchpriority`、Speculation Rules
8. HTTP 缓存策略设计：HTML 协商缓存、带 hash 资源长期强缓存
9. 重排、重绘、合成；强制同步布局；只触发合成的动画属性
10. 长任务的危害和拆分方法：`scheduler.yield`、`setTimeout`、`requestIdleCallback`、Worker
11. 虚拟列表、分片渲染、`content-visibility`
12. 图片优化：格式、尺寸、`srcset`、懒加载、LCP 图片不能懒加载
13. 代码体积治理：分割、Tree Shaking、依赖替换、按需加载
14. SSR / SSG / 流式渲染 / 预渲染对各指标的影响和取舍
15. React 性能：避免无效渲染、并发特性、React Compiler
16. 内存泄漏的常见原因和排查（Heap Snapshot）
17. 性能监控系统设计：采集、上报、聚合、告警、归因
18. 性能预算与 CI 门禁，用数据证明优化收益

## 相关技术

- [浏览器原理](../browser/00-roadmap.md)：渲染流水线、事件循环、HTTP 缓存、合成层 —— 性能优化的底层依据，强烈建议对照阅读
- [JavaScript](../javascript/00-roadmap.md)：事件循环、闭包与内存
- [CSS](../css/00-roadmap.md)：重排重绘、动画、`contain`
- [HTML](../html/00-roadmap.md)：资源提示、图片属性、`<script>` 加载方式
- [React](../react/00-roadmap.md)：渲染优化、并发特性、SSR
- [webpack](../webpack/00-roadmap.md) / [Vite](../vite/00-roadmap.md)：代码分割、产物体积、缓存
- [Node.js](../node/00-roadmap.md)：SSR 服务端性能

## 自检清单

- [ ] 能说出三个 Core Web Vitals 的定义、阈值和优化思路
- [ ] 能把 LCP 和 INP 拆分为子阶段，并给出每个阶段的优化手段
- [ ] 能用 Performance 面板定位长任务和强制同步布局
- [ ] 能设计 HTTP 缓存策略
- [ ] 能手写虚拟列表和任务分片调度
- [ ] 能设计一个性能监控 SDK 和平台
- [ ] 能用 STAR 结构讲一个带数据的性能优化项目
