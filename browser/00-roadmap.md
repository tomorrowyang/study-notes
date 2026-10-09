# 浏览器原理学习路线（高级工程师面试向）

## 面试画像

浏览器原理是国内前端面试中**区分深度**的核心模块，经典问题「从输入 URL 到页面显示发生了什么」可以一路追问到网络、渲染、JS 引擎和安全。考察分三层：

- **基础**：URL 到页面的整体流程、渲染流程、事件循环、事件流、存储、同源策略与跨域、XSS / CSRF —— 一面必问
- **原理**：多进程架构、导航流程、HTTP 缓存、HTTP/2 / HTTP/3、CORS 细节、解析阻塞、样式计算、布局、分层与合成、一帧的生命周期、V8 执行与 GC —— 二面深挖
- **深度与实践**：RenderingNG 与合成线程、站点隔离与 Spectre、跨源隔离、Cookie 与存储隔离的演进、JIT 与去优化、用原理解释性能优化、线上崩溃与兼容性治理 —— 高级岗加分

高级岗典型问法：

> 「从输入 URL 到页面显示，越详细越好」「为什么 transform 动画不卡？主线程阻塞时页面还能滚动吗？」「Promise 和 setTimeout 的执行顺序，浏览器什么时候渲染？」「Chrome 为什么是多进程？站点隔离解决了什么问题？」「V8 是怎么执行 JS 的？为什么要隐藏类？」「SameSite 默认值是什么？第三方 Cookie 现在什么情况？」

## 版本说明（截至 2026-10）

| 时间 | 变化 |
|------|------|
| 2018 | Chrome 67 桌面端默认开启**站点隔离（Site Isolation）**，应对 Spectre |
| 2020 | Chrome 80 起 Cookie 默认 `SameSite=Lax` |
| 2021 | Chrome **RenderingNG** 架构基本完成（LayoutNG、合成线程改造、OOP 光栅化等）；V8 推出 **Sparkplug** 基线编译器 |
| 2022 | HTTP/3 发布为 RFC 9114 |
| 2023 | V8 推出 **Maglev** 中层优化编译器（Chrome 117） |
| 2024–2026 | V8 顶层优化编译器从 TurboFan 的 Sea of Nodes 迁移到 CFG 结构的 **Turboshaft**，并开发 Turbolev（用 Maglev 图作为前端） |
| 2025-04 / 2025-10 | Chrome 宣布**保留第三方 Cookie**（不再推出单独的选择弹窗）；2025-10 宣布退役大部分 Privacy Sandbox API（Topics、Protected Audience、Attribution Reporting 等），**CHIPS、FedCM、Storage Access** 等继续支持 |
| 2025-12 | Safari 26.2 支持 LCP、Event Timing 和 **Navigation API**，这些 API 在主流浏览器中均可用 |
| 2026 | Interop 2026 包括事件循环、Navigation API、同文档 View Transitions、锚点定位等的互操作性改进 |

> Safari（ITP）和 Firefox（Total Cookie Protection）默认限制第三方 Cookie，Chrome 保留第三方 Cookie 但推进存储分区。面试中说「Chrome 2024 年就禁用了第三方 Cookie」是错误的。V8 的编译管线细节以 v8.dev 为准。

## 知识地图

```text
浏览器原理
├─ 架构：浏览器进程、渲染进程、GPU 进程、网络服务、站点隔离；渲染进程中的主线程、合成线程、光栅线程
├─ 导航与网络：URL 解析、DNS、TCP / TLS、HTTP/1.1 / 2 / 3、HTTP 缓存、CORS、连接管理、资源优先级
├─ 解析：HTML 分词与树构建、预加载扫描器、CSS 解析、脚本阻塞
├─ 渲染流水线：样式计算 → 布局 → 分层 → 绘制 → 光栅化 → 合成 → 显示；一帧的生命周期
├─ 事件循环：任务队列、微任务、渲染时机、rAF、输入事件处理
├─ JS 引擎（V8）：解析、Ignition / Sparkplug / Maglev / TurboFan，隐藏类与内联缓存，去优化，分代 GC
├─ 事件与 DOM：事件流、事件委托、页面生命周期
├─ 存储：Cookie、Web Storage、IndexedDB、Cache Storage、配额、存储分区
├─ 安全：同源策略、CORS、XSS、CSRF、CSP、点击劫持、站点隔离、跨源隔离（COOP / COEP）
└─ 新能力：Service Worker、Worker、bfcache、Speculation Rules、Navigation API、View Transitions、WebAssembly、WebGPU
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 组成与内核、URL 到页面、网络基础、渲染流程、事件循环基础、事件、存储、同源与跨域、XSS / CSRF
2. [中级 Q&A](20-mid-qa.md) — 多进程与站点隔离、导航流程、HTTP 缓存、HTTP/2 / 3、TLS、CORS、解析阻塞、样式 / 布局 / 分层 / 合成、事件循环细节、V8、安全防御
3. [高级 Q&A](30-senior-qa.md) — RenderingNG、合成线程、事件循环与渲染时机、JIT 与 GC 深入、Spectre 与跨源隔离、存储隔离演进、预渲染、bfcache、崩溃排查、用原理解释性能优化
4. [手写与现场编码](40-coding-handwritten.md) — 12 道题：事件委托、事件循环输出题、跨标签页通信、迷你 HTML 解析器、选择器匹配、CORS / CSRF 中间件等
5. [系统设计与场景](50-system-design.md) — 简化渲染引擎、前端安全体系、跨域架构、离线与多标签页同步、稳定性监控等
6. [真实 / 高频面试题](90-interview-real.md)
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. 从输入 URL 到页面显示的完整流程，每一步能展开讲
2. Chrome 多进程架构，各进程职责，站点隔离的原因
3. 渲染进程中的线程：主线程、合成线程、光栅线程
4. HTTP 缓存：强缓存、协商缓存、启发式缓存、`Vary`、`no-cache` 与 `no-store`
5. HTTP/1.1 → HTTP/2 → HTTP/3 解决了什么问题（队头阻塞）
6. CORS 简单请求与预检请求，带凭证请求的限制
7. CSS 和 JS 如何阻塞解析和渲染，预加载扫描器
8. 渲染流水线各阶段的输入输出：DOM → 样式 → 布局树 → 分层 → 绘制记录 → 图块 → 合成帧
9. 为什么 transform / opacity 动画可以在合成线程运行
10. 事件循环：任务、微任务、渲染时机；`rAF` 在一帧中的位置
11. 浏览器与 Node.js 事件循环的区别
12. V8 的分层编译、隐藏类、内联缓存、去优化
13. V8 分代垃圾回收：Scavenge、Mark-Compact、并发与增量标记
14. XSS 的类型与防御（转义、CSP、HttpOnly、Trusted Types）
15. CSRF 的原理与防御（SameSite、Token、Origin 校验）
16. Cookie 属性与第三方 Cookie 的现状，CHIPS 与存储分区
17. Spectre 与站点隔离、COOP / COEP 与 SharedArrayBuffer
18. 能用浏览器原理解释常见的性能优化手段

## 相关技术

- [前端性能优化](../performance/00-roadmap.md)：浏览器原理的实践应用，强烈建议对照阅读
- [JavaScript](../javascript/00-roadmap.md)：事件循环、闭包、内存
- [HTML](../html/00-roadmap.md)：解析、`<script>` 加载、资源提示
- [CSS](../css/00-roadmap.md)：层叠、BFC、层叠上下文、合成
- [Node.js](../node/00-roadmap.md)：Node 事件循环、V8
- [React](../react/00-roadmap.md)：调度器与浏览器帧的关系

## 自检清单

- [ ] 能在 3 分钟内讲完 URL 到页面的流程，并能任意展开一个环节
- [ ] 能画出 Chrome 的进程模型和渲染进程的线程模型
- [ ] 能讲清 HTTP 缓存的判断流程
- [ ] 能讲清渲染流水线各阶段，以及重排、重绘、合成的区别
- [ ] 能做对事件循环的输出顺序题，并说明渲染发生在哪里
- [ ] 能讲清 V8 的执行流程和垃圾回收
- [ ] 能讲清 XSS、CSRF 的原理和防御，以及 Cookie 的现状
