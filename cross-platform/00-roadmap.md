# 跨端开发学习路线（React Native / 小程序）

## 面试画像

国内大厂的跨端岗位主要集中在两条线：**React Native（含 Expo）做 App**，**小程序（微信原生 / Taro / uni-app）做轻应用**，另外还有 H5 + WebView 混合开发和鸿蒙适配。面试层次：

- **基础**：RN 的组件和样式（Flexbox）、列表、导航、平台差异；小程序的目录结构、生命周期、`setData`、组件通信、分包 —— 初中级必问
- **原理与优化**：RN 新架构（JSI、Fabric、TurboModules、Codegen）、Hermes、列表性能、动画；小程序的**双线程模型**、`setData` 的开销与优化、Skyline 渲染引擎、Taro / uni-app 的编译和运行时原理；JSBridge —— 中高级区分点
- **体系与架构**：跨端技术选型、热更新体系、原生模块与混合栈、启动和包体积优化、多端统一的组件库、监控和稳定性、鸿蒙适配 —— 高级岗必谈

高级岗典型问法：

> 「RN 新架构解决了老架构的什么问题？JSI 是什么？」「为什么小程序要设计成双线程？`setData` 为什么慢？」「Taro 是怎么把 React 代码跑在小程序上的？」「RN 长列表卡顿怎么排查？」「RN 热更新怎么做？有什么合规风险？」「JSBridge 的原理？」「RN、Flutter、小程序怎么选？」

## 版本与现状（截至 2026-10）

| 技术 | 版本 / 事件 | 要点 |
|------|-------------|------|
| React Native | **0.82**（2025-10） | 只支持新架构（旧架构无法再开启）；Hermes V1 可选启用 |
| React Native | 0.83 | 升级到 React 19.2 |
| React Native | **0.84**（2026-02） | **Hermes V1 成为默认引擎**；iOS 默认使用预编译二进制（构建更快）；iOS 构建中移除旧架构代码；要求 Node.js 22.11+ |
| React Native | **0.86**（2026-06） | Android 15+ 默认 edge-to-edge（内容延伸到系统栏下）；无破坏性变更 |
| Expo | SDK 55（RN 0.83）→ **SDK 57**（2026-06，RN 0.86） | 不再支持旧架构；EAS Build / Update；Expo Router |
| 微信小程序 | **Skyline 渲染引擎** | 需要配置 `"renderer": "skyline"` 和 `"componentFramework": "glass-easel"`；Worklet 动画替代 WXS 动画；手势系统、共享元素动画 |
| 微信小程序 | glass-easel 组件框架 | 新的组件框架，基础库 3.8.12 起在 WebView 渲染下也可使用；Skyline 只支持 glass-easel |
| Taro | 4.x | 支持 React / Vue；支持 Skyline，4.0.8 起支持 Worklet；「半编译」模式提升性能；支持鸿蒙 |
| uni-app | uni-app / **uni-app x** | uni-app x 使用 UTS 语言编译为原生代码（Kotlin / Swift / ArkTS），不再依赖 JS 引擎 |
| 鸿蒙 | HarmonyOS NEXT | 不再兼容 Android 应用，需要 ArkTS / ArkUI 原生开发或跨端框架适配（RN 鸿蒙版、Taro、uni-app、Flutter 鸿蒙版） |
| Flutter | 3.x | 自绘引擎（Impeller），Dart 语言 |

> 版本信息变化快，以官方文档和发布说明为准。

## 知识地图

```text
跨端开发
├─ 跨端方案分类
│   ├─ Web 容器：H5 + WebView、JSBridge、离线包
│   ├─ JS + 原生渲染：React Native、Weex（历史）、小程序（双线程）
│   ├─ 自绘引擎：Flutter、Skyline（小程序）
│   └─ 编译到原生：uni-app x、Kotlin Multiplatform
├─ React Native
│   ├─ 基础：核心组件、StyleSheet / Flexbox、平台差异、列表（FlatList / FlashList）
│   ├─ 导航：React Navigation、Expo Router
│   ├─ 架构：旧架构（Bridge、异步 JSON）→ 新架构（JSI、Fabric、TurboModules、Codegen、Bridgeless）
│   ├─ 引擎：Hermes（字节码预编译、Hermes V1）
│   ├─ 性能：启动、列表、动画（Reanimated、原生驱动）、重渲染、内存
│   ├─ 原生：原生模块、原生组件、混合开发（原生 + RN 页面）
│   ├─ 工程：Expo / EAS、Metro、热更新（OTA）、CodePush 替代方案
│   └─ 质量：调试（React Native DevTools）、崩溃监控、测试
├─ 小程序
│   ├─ 基础：目录结构、WXML / WXSS / JS / JSON、生命周期、组件、路由
│   ├─ 原理：双线程模型（逻辑层 + 渲染层）、setData 通信、基础库
│   ├─ 性能：setData 优化、分包与预下载、启动优化、长列表
│   ├─ Skyline：渲染引擎、glass-easel、Worklet 动画、手势
│   ├─ 跨端框架：Taro（编译时 + 运行时）、uni-app、uni-app x
│   └─ 平台差异：微信、支付宝、抖音、百度
├─ 混合开发：WebView、JSBridge、离线包、首屏优化
└─ 鸿蒙：ArkTS / ArkUI、跨端框架适配
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 跨端方案分类、RN 基础、小程序基础、Taro / uni-app 入门
2. [中级 Q&A](20-mid-qa.md) — RN 新架构、Hermes、列表与动画性能、小程序双线程与 setData 优化、Skyline、JSBridge
3. [高级 Q&A](30-senior-qa.md) — 新架构底层、热更新体系、混合开发、Taro 原理、启动优化、跨端选型与架构
4. [手写与现场编码](40-coding-handwritten.md) — 12 道题：JSBridge、setData 差量更新、小程序请求封装、RN 虚拟列表、热更新检查等
5. [系统设计与场景](50-system-design.md) — 跨端组件库、热更新平台、混合 App 架构、小程序电商等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. 跨端方案的分类（Web 容器、JS + 原生渲染、自绘引擎、编译到原生）和各自的取舍
2. RN 的运行原理：JS 线程、UI 线程、后台线程（布局）
3. **旧架构的问题**：Bridge 异步、JSON 序列化、批量传输、启动时初始化所有原生模块
4. **新架构**：JSI（JS 直接持有 C++ 对象引用）、Fabric（C++ 渲染器、同步布局、并发特性）、TurboModules（懒加载、类型安全）、Codegen
5. Hermes：字节码预编译带来的启动提升、内存占用；Hermes V1
6. RN 列表性能：FlatList 的窗口化、`getItemLayout`、`keyExtractor`、FlashList 的复用
7. RN 动画：为什么要在 UI 线程执行（Reanimated worklet、`useNativeDriver`）
8. RN 热更新原理（替换 JS bundle）、方案（EAS Update 等）和应用商店的合规限制
9. 小程序**双线程模型**：逻辑层和渲染层分离的原因（安全、管控），带来的通信开销
10. **setData 的开销**：数据序列化 + 跨线程传输 + 渲染层 diff；优化手段（只传变化的路径、减少频率、不传无关数据）
11. 小程序的分包、独立分包、分包预下载；启动流程与优化
12. Skyline 的原理和收益（独立渲染线程、Worklet 动画、手势），与 WebView 渲染的区别
13. **Taro 的原理**：Taro 3+ 的运行时方案（在小程序中模拟 DOM / BOM，React 渲染到虚拟 DOM 树，再通过 setData 同步到模板）
14. uni-app 的编译原理与条件编译；uni-app x 的 UTS
15. **JSBridge 原理**：JS 调用原生（URL 拦截、注入 API、`postMessage`）和原生调用 JS（evaluateJavaScript）；回调 ID 机制
16. H5 混合开发的首屏优化：离线包、预加载 WebView、数据预取
17. RN 混合开发：原生页面与 RN 页面的路由、多 bundle 拆包、实例复用
18. 跨端技术选型（RN、Flutter、小程序、H5、原生）与鸿蒙适配

## 相关技术

- [React](../react/00-roadmap.md)：RN 和 Taro 的基础，Fiber、并发特性
- [Vue](../vue/00-roadmap.md)：uni-app、Taro 的 Vue 版本
- [JavaScript](../javascript/00-roadmap.md) / [TypeScript](../typescript/00-roadmap.md)：语言基础
- [浏览器原理](../browser/00-roadmap.md)：渲染流程，对比 WebView 与小程序渲染层
- [性能优化](../performance/00-roadmap.md)：启动、列表、动画的优化思路
- [前端工程化](../engineering/00-roadmap.md)：Monorepo 管理多端代码、CI/CD、热更新发布
- [前端安全](../frontend-security/00-roadmap.md)：WebView 安全、JSBridge 的权限控制

## 自检清单

- [ ] 能说清 RN、Flutter、小程序、H5 的渲染方式差异
- [ ] 能讲清 RN 旧架构的瓶颈以及新架构如何解决
- [ ] 能解释 JSI 是什么，为什么它让同步调用成为可能
- [ ] 能给出 RN 长列表卡顿的排查和优化方法
- [ ] 能解释小程序为什么是双线程，以及 setData 为什么慢
- [ ] 能说清 Taro 3 的运行时原理
- [ ] 能手写一个简单的 JSBridge（含回调机制）
- [ ] 能设计一个 RN 热更新方案并说明合规风险
- [ ] 能根据业务场景给出跨端技术选型并说明理由
