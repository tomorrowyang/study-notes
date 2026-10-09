# 跨端开发真实面试题

> 根据近年公开面经和技术社区讨论整理，`来源倾向`表示这类题目常出现的场景，不代表某公司的固定题库。答案为要点，详细内容见对应链接。

---

### R1. RN 的原理是什么？和 H5、Flutter 有什么区别？

- **来源倾向**: 一面、高频
- **难度**: 初级

**答案**

- RN：React 描述 UI，映射到平台原生控件；JS 线程运行业务，UI 线程绘制
- H5：浏览器内核渲染 DOM
- Flutter：自绘引擎，不使用平台控件

详见：[10-junior-qa.md Q1、Q3](./10-junior-qa.md)

**口述模板（30–60 秒）**

1. **一句话结论**：RN 用 React 描述 UI，最终渲染为原生控件。
2. **关键机制**：JS 线程运行业务代码，布局由 Yoga 计算，UI 线程绘制。
3. **对比或注意点**：H5 由浏览器渲染，Flutter 用自己的引擎绘制所有控件。
4. **可追问时补充**：新架构用 JSI 替代了异步的 Bridge。

---

### R2. RN 新架构解决了什么问题？JSI、Fabric、TurboModules 分别是什么？

- **来源倾向**: 二面、高频
- **难度**: 中级

**答案**

- 旧架构问题：Bridge 异步、JSON 序列化、批量传输、启动时初始化全部模块
- JSI：C++ 接口，JS 直接调用 C++ 对象，可同步、无序列化
- Fabric：C++ 渲染器，不可变 Shadow Tree，同步布局，支持并发特性
- TurboModules：懒加载、基于 JSI、Codegen 保证类型一致
- 0.82 起只支持新架构

详见：[20-mid-qa.md Q1–Q5](./20-mid-qa.md)、[30-senior-qa.md Q1–Q2](./30-senior-qa.md)

**口述模板（30–60 秒）**

1. **一句话结论**：新架构用 JSI 替代了异步的 JSON Bridge。
2. **关键机制**：Fabric 是新的渲染系统，TurboModules 是懒加载的原生模块系统，都基于 JSI。
3. **对比或注意点**：解决了无法同步调用、序列化开销和启动慢的问题。
4. **可追问时补充**：Codegen 根据 TypeScript 接口生成原生代码，保证类型一致。

---

### R3. Hermes 是什么？为什么启动快？

- **来源倾向**: 二面
- **难度**: 中级

**答案**

- RN 默认的 JS 引擎；构建时编译为字节码，启动时跳过解析和编译
- 字节码按需加载，内存占用低
- 0.84 起 Hermes V1 为默认

详见：[20-mid-qa.md Q7](./20-mid-qa.md)

---

### R4. RN 长列表卡顿怎么优化？

- **来源倾向**: 二面、高频
- **难度**: 中级

**答案**

- 先区分 JS FPS 和 UI FPS
- FlatList：`getItemLayout`、`keyExtractor`、列表项 memo、`windowSize` 等
- FlashList：复用列表项（注意 state 被带到新数据上）
- 图片使用缩略图和缓存；列表项轻量

详见：[20-mid-qa.md Q8–Q10](./20-mid-qa.md)

**口述模板（30–60 秒）**

1. **一句话结论**：先判断是 JS 线程还是 UI 线程卡，再针对性优化。
2. **关键机制**：FlatList 减少重渲染、提供 getItemLayout，调整渲染窗口。
3. **对比或注意点**：FlatList 会销毁和创建列表项，FlashList 通过复用减少开销。
4. **可追问时补充**：必须在 Release 包和低端机上测量。

---

### R5. RN 的动画怎么做到流畅？

- **来源倾向**: 二面
- **难度**: 中级

**答案**

- 动画在 UI 线程执行：Animated 原生驱动（只支持 transform / opacity）、Reanimated worklet
- 手势用 Gesture Handler 在 UI 线程识别
- 避免用 state 驱动每一帧

详见：[20-mid-qa.md Q11](./20-mid-qa.md)、[40-coding-handwritten.md H7](./40-coding-handwritten.md)

---

### R6. RN 热更新的原理？有什么限制？

- **来源倾向**: 二面 / 三面、高频
- **难度**: 中级

**答案**

- 下载新的 JS bundle 替换本地 bundle，下次启动加载
- 只能更新 JS 和资源，bundle 要与原生版本匹配
- 签名校验、灰度、启动保护和回滚
- CodePush 已停止服务，可用 EAS Update 或自建
- 合规：不能用于发布重大功能变化

详见：[20-mid-qa.md Q16](./20-mid-qa.md)、[50-system-design.md S1](./50-system-design.md)、[40-coding-handwritten.md H6](./40-coding-handwritten.md)

**口述模板（30–60 秒）**

1. **一句话结论**：RN 热更新就是替换本地的 JS bundle。
2. **关键机制**：检查更新、下载、校验签名、下次启动加载。
3. **对比或注意点**：只能更新 JS，bundle 必须与原生版本匹配。
4. **可追问时补充**：要有启动保护，新 bundle 崩溃时自动回滚。

---

### R7. RN 和原生怎么通信？怎么写一个原生模块？

- **来源倾向**: 二面
- **难度**: 中级

**答案**

- 新架构：TurboModules（TS spec + Codegen）或 Expo Modules API（Swift / Kotlin）
- 原生 → JS：事件发射器
- 耗时操作异步；`jsi::Runtime` 只能在 JS 线程访问

详见：[20-mid-qa.md Q15](./20-mid-qa.md)、[30-senior-qa.md Q3](./30-senior-qa.md)

---

### R8. RN App 的启动怎么优化？

- **来源倾向**: 二面 / 三面
- **难度**: 高级

**答案**

- Hermes 字节码、inline requires、TurboModules 懒加载
- RN 实例预创建、拆包（基础包预加载）
- 首屏数据由原生并行预取、缓存先显示
- 延迟非必要 SDK 的初始化

详见：[20-mid-qa.md Q12](./20-mid-qa.md)、[30-senior-qa.md Q7](./30-senior-qa.md)

---

### R9. 小程序为什么设计成双线程？

- **来源倾向**: 一面 / 二面、高频
- **难度**: 中级

**答案**

- 逻辑层（JS 引擎）和渲染层（WebView）分离，通过 Native 中转通信
- 原因：管控和安全（逻辑层没有 DOM，无法随意操作页面和跳转）；JS 不阻塞渲染
- 代价：setData 跨线程通信开销

详见：[20-mid-qa.md Q20](./20-mid-qa.md)

**口述模板（30–60 秒）**

1. **一句话结论**：双线程主要是为了平台的管控和安全。
2. **关键机制**：逻辑层没有 DOM，只能通过 setData 更新视图。
3. **对比或注意点**：代价是逻辑层和渲染层之间的通信开销。
4. **可追问时补充**：WXS 和 Skyline 的 Worklet 都是为了减少跨线程通信。

---

### R10. setData 为什么慢？怎么优化？

- **来源倾向**: 二面、高频
- **难度**: 中级

**答案**

- 过程：序列化 → 跨线程传输 → 渲染层 diff 和更新
- 优化：只传变化的路径、不传无关数据、合并调用、降低频率、长列表分页追加、组件化

详见：[20-mid-qa.md Q21–Q22](./20-mid-qa.md)、[40-coding-handwritten.md H3–H4](./40-coding-handwritten.md)

**口述模板（30–60 秒）**

1. **一句话结论**：setData 慢是因为序列化和跨线程传输。
2. **关键机制**：开销与数据量和调用频率成正比。
3. **对比或注意点**：用路径语法只传变化的字段，不需要渲染的数据不放进 data。
4. **可追问时补充**：高频交互用 WXS 或 Worklet 在渲染层处理。

---

### R11. 小程序的分包怎么做？独立分包有什么用？

- **来源倾向**: 一面
- **难度**: 初级

**答案**

- 主包只放启动页和公共代码，其他页面分包，进入时下载
- 独立分包不依赖主包，可以单独启动（活动页）
- 分包预下载

详见：[10-junior-qa.md Q23](./10-junior-qa.md)

---

### R12. 小程序的启动怎么优化？

- **来源倾向**: 二面
- **难度**: 中级

**答案**

- 减小主包、按需注入
- 数据预拉取、缓存先渲染、初始渲染缓存
- 骨架屏、分包预下载

详见：[20-mid-qa.md Q23](./20-mid-qa.md)

---

### R13. 小程序的登录流程？

- **来源倾向**: 一面、高频
- **难度**: 初级

**答案**

- `wx.login` 获取 code → 服务端 code2Session 换 openid / session_key → 服务端创建自己的登录态
- session_key 和 appsecret 只在服务端
- 并发请求共享一次登录

详见：[10-junior-qa.md Q21](./10-junior-qa.md)、[40-coding-handwritten.md H2](./40-coding-handwritten.md)

---

### R14. 小程序的长列表怎么优化？

- **来源倾向**: 二面
- **难度**: 中级

**答案**

- 分页时路径追加
- 虚拟列表（屏幕外替换为占位）
- 组件化缩小更新范围
- Skyline 的按需渲染列表组件

详见：[20-mid-qa.md Q24](./20-mid-qa.md)、[40-coding-handwritten.md H5](./40-coding-handwritten.md)

---

### R15. Skyline 是什么？和 WebView 渲染有什么区别？

- **来源倾向**: 二面、近年热点
- **难度**: 中级

**答案**

- 微信小程序的新渲染引擎，自研渲染管线，页面共享渲染线程
- 需要 glass-easel 组件框架；CSS 子集
- 新能力：Worklet 动画、手势、共享元素动画

详见：[20-mid-qa.md Q26–Q27](./20-mid-qa.md)、[30-senior-qa.md Q5](./30-senior-qa.md)

---

### R16. WXS 是什么？什么时候用？

- **来源倾向**: 二面
- **难度**: 中级

**答案**

- 运行在渲染层的脚本，用于模板中的格式化和直接响应触摸事件
- 避免跨线程通信，实现跟手交互
- Skyline 中推荐 Worklet

详见：[20-mid-qa.md Q25](./20-mid-qa.md)

---

### R17. Taro 是怎么把 React 跑在小程序上的？

- **来源倾向**: 二面 / 三面、高频
- **难度**: 高级

**答案**

- Taro 3 运行时方案：在逻辑层模拟 DOM / BOM，React 渲染到模拟 DOM
- DOM 的变化转换为 setData 的路径更新
- 渲染层用预先生成的通用模板递归渲染（固定层数）
- 优化：CustomWrapper、半编译、VirtualList

详见：[20-mid-qa.md Q29–Q30](./20-mid-qa.md)、[40-coding-handwritten.md H8](./40-coding-handwritten.md)

**口述模板（30–60 秒）**

1. **一句话结论**：Taro 3 在小程序的逻辑层模拟了一套 DOM，让 React 正常运行。
2. **关键机制**：模拟 DOM 的变化被合并为最小的 setData 路径更新。
3. **对比或注意点**：渲染层用通用模板递归渲染，层数有限制。
4. **可追问时补充**：运行时方案有额外开销，可以用 CustomWrapper 和半编译模式优化。

---

### R18. Taro 和 uni-app 的区别？

- **来源倾向**: 一面 / 二面
- **难度**: 中级

**答案**

- Taro：React / Vue，Taro 3 是运行时方案，不限制语法
- uni-app：Vue，小程序端以编译为主；App 端 WebView 或 nvue；uni-app x 编译为原生
- 选型：技术栈、是否需要 App、生态

详见：[10-junior-qa.md Q26–Q27](./10-junior-qa.md)、[20-mid-qa.md Q31](./20-mid-qa.md)

---

### R19. JSBridge 的原理？

- **来源倾向**: 一面 / 二面、高频
- **难度**: 中级

**答案**

- JS → 原生：注入对象（`messageHandlers` / `addJavascriptInterface`）、URL 拦截
- 原生 → JS：`evaluateJavaScript`
- 回调 ID 对应请求和结果；超时
- 安全：域名白名单、接口分级

详见：[10-junior-qa.md Q29](./10-junior-qa.md)、[40-coding-handwritten.md H1](./40-coding-handwritten.md)、[30-senior-qa.md Q21](./30-senior-qa.md)

**口述模板（30–60 秒）**

1. **一句话结论**：JSBridge 是 WebView 中 JS 和原生的双向通信机制。
2. **关键机制**：JS 通过注入对象调用原生，原生通过执行 JS 回调。
3. **对比或注意点**：每次调用生成回调 ID，原生处理完后用 ID 找到对应的 Promise。
4. **可追问时补充**：原生要校验调用页面的域名，防止恶意页面调用敏感能力。

---

### R20. H5 页面在 App 中打开慢，怎么优化？

- **来源倾向**: 二面
- **难度**: 中级

**答案**

- WebView 预创建复用
- 离线包（本地资源拦截）
- 原生并行预取数据
- SSR / 骨架屏、DNS 预解析

详见：[20-mid-qa.md Q32](./20-mid-qa.md)、[50-system-design.md S5](./50-system-design.md)

---

### R21. 一套代码多端，架构怎么设计？

- **来源倾向**: 三面
- **难度**: 高级

**答案**

- 业务逻辑层（纯 TS）+ 适配层（平台接口 + 各端实现）+ 视图层
- 平台差异通过接口和能力检测处理
- Monorepo 管理

详见：[30-senior-qa.md Q15](./30-senior-qa.md)、[50-system-design.md S4](./50-system-design.md)、[40-coding-handwritten.md H9](./40-coding-handwritten.md)

---

### R22. RN、Flutter、小程序怎么选？

- **来源倾向**: 三面 / 主管面
- **难度**: 高级

**答案**

- 团队技能、是否需要热更新、UI 一致性、性能、流量入口、鸿蒙支持
- 前端团队做 App → RN；多端 UI 一致 → Flutter；微信生态 → 小程序

详见：[30-senior-qa.md Q17](./30-senior-qa.md)

**口述模板（30–60 秒）**

1. **一句话结论**：选型取决于团队技能、动态性、UI 一致性和流量入口。
2. **关键机制**：RN 适合前端团队做 App，可以热更新。
3. **对比或注意点**：Flutter 多端一致但需要学习 Dart，小程序依托平台流量但受平台限制。
4. **可追问时补充**：还要考虑鸿蒙的适配方案。

---

### R23. 原生 App 中怎么引入 RN？

- **来源倾向**: 三面
- **难度**: 高级

**答案**

- 路由中心 + RN 容器；共享 RN 实例
- 统一的原生能力模块
- 从试点页面开始，用数据推进

详见：[30-senior-qa.md Q11、Q18](./30-senior-qa.md)、[50-system-design.md S2](./50-system-design.md)

---

### R24. 鸿蒙适配你了解多少？

- **来源倾向**: 二面 / 三面、近年热点
- **难度**: 中级

**答案**

- HarmonyOS NEXT 不兼容 Android 应用
- ArkTS / ArkUI 原生开发；跨端框架适配（RN 鸿蒙版、Taro、uni-app、Flutter）
- 工作量在三方库和原生模块的适配

详见：[10-junior-qa.md Q30](./10-junior-qa.md)、[30-senior-qa.md Q19](./30-senior-qa.md)

---

### R25. 讲一个你做过的跨端性能优化

- **来源倾向**: 三面 / 主管面
- **难度**: 高级

**答案**

- 按 STAR：问题和数据 → 定位过程（工具、线程）→ 方案 → 量化结果 → 复盘
- 常见素材：RN 列表从 FlatList 迁移到 FlashList、启动时实例预创建、小程序 setData 优化、离线包

详见：[30-senior-qa.md Q6、Q25](./30-senior-qa.md)
