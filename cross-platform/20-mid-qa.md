# 跨端开发中级 Q&A

> 目标：讲清 React Native 新架构和 Hermes、RN 的性能优化；讲清小程序的双线程模型、setData 的开销与优化、Skyline；讲清 Taro / uni-app 的原理和 WebView 混合开发的优化。

---

## 一、React Native 架构

### Q1. RN 旧架构（Bridge）有什么问题？

- **难度**: 中级
- **标签**: RN, 架构

**答案**

```text
JS 线程 ──JSON 序列化──▶ Bridge（异步消息队列，批量）──▶ 原生线程
        ◀──JSON 反序列化──                        ◀──
```

**问题**：

| 问题 | 影响 |
|------|------|
| **全部异步** | JS 无法同步获取原生的结果（例如同步测量布局），需要回调 |
| **JSON 序列化** | 每次通信都要序列化和反序列化，数据量大时开销高 |
| **批量传输** | 消息批量发送，延迟不可控；高频交互（滚动、手势、动画）卡顿 |
| **启动时初始化所有原生模块** | 模块越多启动越慢 |
| **缺乏类型约束** | JS 和原生的接口靠约定，容易出错 |
| 单线程 Shadow Tree | 无法支持 React 的并发特性 |

典型现象：快速滚动时出现白块（JS 来不及渲染）；手势跟随有延迟。

**口述模板（30–60 秒）**

1. 一句话结论：旧架构的 Bridge 是异步的 JSON 消息队列。
2. 关键机制：所有通信都要序列化并批量异步传输。
3. 对比或注意点：导致无法同步调用、高频交互卡顿、启动时要初始化所有模块。
4. 可追问时补充：新架构用 JSI 替代 Bridge，解决了这些问题。

**常见追问**

- 为什么最初要设计成异步？→ 让 JS 线程和 UI 线程解耦，JS 的卡顿不会直接阻塞 UI；但代价是无法同步交互。

---

### Q2. JSI 是什么？

- **难度**: 中级
- **标签**: RN, JSI

**答案**

**JSI（JavaScript Interface）** 是一层 C++ API，让 JS 引擎和 C++ 代码直接交互：

- JS 可以**直接持有 C++ 对象（HostObject）的引用**并调用其方法，**不需要序列化**
- 支持**同步调用**（也可以异步）
- 与具体的 JS 引擎解耦（Hermes、JSC 都实现了 JSI）

```text
旧：JS → JSON → Bridge 队列 → 原生
新：JS → JSI（C++ 函数调用）→ 原生
```

**带来的能力**：TurboModules 和 Fabric 都基于 JSI；MMKV、Reanimated、VisionCamera 等高性能库直接使用 JSI。

**口述模板（30–60 秒）**

1. 一句话结论：JSI 是一层 C++ 接口，让 JS 直接调用 C++ 对象的方法。
2. 关键机制：没有序列化，支持同步调用。
3. 对比或注意点：同步调用如果耗时长会阻塞 JS 线程，要谨慎使用。
4. 可追问时补充：TurboModules、Fabric 和许多高性能库都基于 JSI。

**常见追问**

- 同步调用一定好吗？→ 不一定；耗时操作同步调用会阻塞 JS 线程，应该异步。同步适合测量布局、读取小数据等快速操作。

---

### Q3. Fabric 是什么？解决了什么问题？

- **难度**: 中级
- **标签**: RN, Fabric

**答案**

Fabric 是 RN 的**新渲染系统**：

- 渲染逻辑用 **C++** 实现，iOS 和 Android 共享（一致性更好）
- **Shadow Tree 不可变**，可以在多个线程中安全地创建，支持 **React 18+ 的并发特性**（`startTransition`、Suspense）
- **同步布局**：可以在 UI 线程同步计算和提交（例如 `useLayoutEffect` 中测量布局后立即调整，不会闪烁）
- 通过 JSI 与 JS 通信，不需要序列化

**渲染流程**：

```text
Render（JS 线程：React 生成元素树 → C++ 创建 Shadow Tree）
  → Commit（后台线程：Yoga 计算布局）
  → Mount（UI 线程：对比新旧树，更新原生视图）
```

**口述模板（30–60 秒）**

1. 一句话结论：Fabric 是 RN 的新渲染系统，核心用 C++ 实现。
2. 关键机制：Shadow Tree 不可变，可以在多个线程中处理。
3. 对比或注意点：支持同步布局和 React 的并发特性。
4. 可追问时补充：渲染分为 Render、Commit、Mount 三个阶段。

**常见追问**

- `useLayoutEffect` 在新架构中有什么变化？→ 可以同步读取布局并在绘制前更新，避免旧架构中的闪烁。

---

### Q4. TurboModules 是什么？

- **难度**: 中级
- **标签**: RN, TurboModules

**答案**

TurboModules 是新架构的**原生模块系统**：

- **懒加载**：模块在第一次使用时才初始化（旧架构启动时初始化所有模块）→ 启动更快
- **基于 JSI**：直接调用，支持同步和异步，没有序列化
- **类型安全**：用 TypeScript 定义接口规范（spec），由 Codegen 生成原生的接口代码
- 可以用 C++ 实现跨平台共享的模块

**口述模板（30–60 秒）**

1. 一句话结论：TurboModules 是新架构的原生模块系统。
2. 关键机制：懒加载、基于 JSI 直接调用。
3. 对比或注意点：接口由 TypeScript 定义，Codegen 生成原生代码，保证类型一致。
4. 可追问时补充：模块可以用 C++ 实现，iOS 和 Android 共享。

**常见追问**

- 懒加载为什么能加快启动？→ 大型 App 有上百个原生模块，旧架构启动时全部初始化；懒加载只初始化首屏用到的模块。

---

### Q5. Codegen 是什么？

- **难度**: 中级
- **标签**: RN, Codegen

**答案**

Codegen 在**构建时**读取 TypeScript（或 Flow）写的接口规范，生成 C++ / Objective-C / Java 的接口代码：

```ts
// NativeCalculator.ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  add(a: number, b: number): Promise<number>;
  multiplySync(a: number, b: number): number;
}

export default TurboModuleRegistry.getEnforcing<Spec>('NativeCalculator');
```

原生只需要实现生成的接口；JS 和原生的类型不一致时**编译时报错**，而不是运行时崩溃。

**口述模板（30–60 秒）**

1. 一句话结论：Codegen 根据 TypeScript 接口规范生成原生的接口代码。
2. 关键机制：原生实现生成的接口，JS 调用时类型一致。
3. 对比或注意点：类型不一致在编译时就会发现。
4. 可追问时补充：原生组件的属性也通过 Codegen 生成。

**常见追问**

- 旧架构的原生模块能直接在新架构中运行吗？→ 有兼容层（Interop Layer）可以运行大多数旧模块，但建议迁移到 TurboModules。

---

### Q6. 怎么把一个 RN 项目迁移到新架构？

- **难度**: 中级
- **标签**: RN, 迁移

**答案**

RN 0.82 起只支持新架构，所以升级到新版本就意味着迁移：

1. **检查依赖**：第三方库是否支持新架构（React Native Directory 有标记）；不支持的寻找替代或使用兼容层
2. **逐步升级 RN 版本**（Upgrade Helper 对比原生工程的差异）；Expo 项目升级 SDK
3. **自研原生模块**迁移到 TurboModules / Fabric 组件（或 Expo Modules API）
4. **测试重点**：布局（同步布局的时序变化）、原生组件、动画、启动
5. 灰度发布，监控崩溃率

**常见问题**：老的第三方库不再维护；依赖旧 Bridge 私有 API 的代码失效。

**口述模板（30–60 秒）**

1. 一句话结论：先检查第三方库的兼容性，再逐步升级 RN 版本，最后迁移自研原生模块。
2. 关键机制：兼容层可以运行大多数旧模块，作为过渡。
3. 对比或注意点：不再维护的库要寻找替代。
4. 可追问时补充：迁移后重点测试布局、动画和原生组件，并灰度发布。

**常见追问**

- 为什么推荐用 Expo Modules API 写原生模块？→ 用 Swift / Kotlin 编写，API 简洁，自动支持新架构。

---

### Q7. Hermes 为什么能提升启动速度？Hermes V1 有什么变化？

- **难度**: 中级
- **标签**: RN, Hermes

**答案**

**启动提升的原因**：

- 传统引擎在启动时**下载 → 解析 → 编译** JS；Hermes 在**构建时**把 JS 编译为**字节码**，启动时直接执行，跳过解析和编译
- 字节码可以**按需加载**（内存映射），不需要一次性读入
- 为移动端调优的 GC，内存占用低

**Hermes V1**（RN 0.84 起默认）：新一代的 Hermes，提升执行性能和现代 JS 特性的支持（具体以官方说明为准）。

**注意**：Hermes 不支持 `eval` 动态执行源码等少量特性；字节码与 Hermes 版本绑定（热更新的 bundle 要用同版本编译）。

**口述模板（30–60 秒）**

1. 一句话结论：Hermes 在构建时把 JS 编译成字节码，启动时省去解析和编译。
2. 关键机制：字节码可以按需加载，减少内存占用。
3. 对比或注意点：字节码与 Hermes 版本绑定。
4. 可追问时补充：RN 0.84 起 Hermes V1 是默认引擎。

**常见追问**

- 热更新的 bundle 要注意什么？→ 用与 App 中 Hermes 版本一致的编译器生成字节码，否则无法加载。

---

## 二、React Native 性能

### Q8. RN 页面卡顿怎么排查？

- **难度**: 中级
- **标签**: RN, 性能

**答案**

**先判断是哪个线程卡**：性能监视器显示 **JS FPS** 和 **UI FPS**。

| 现象 | 原因 | 方向 |
|------|------|------|
| JS FPS 低 | JS 线程繁忙：重渲染、大计算、同步 JSON 解析 | React Profiler 找重渲染；memo；拆分计算；移到后台 |
| UI FPS 低 | 原生渲染繁重：视图层级深、过度绘制、大图 | 减少嵌套；图片尺寸；避免透明度叠加 |
| 动画卡 | 动画在 JS 线程驱动 | Reanimated / 原生驱动 |
| 列表白块 | 渲染跟不上滚动 | 列表优化（Q9、Q10） |

**工具**：React Native DevTools（Profiler）、Perfetto（Android）、Instruments（iOS）、Flashlight（性能评分）。

**注意**：在 **Release 包**中测量，开发模式有大量额外开销。

**口述模板（30–60 秒）**

1. 一句话结论：先看 JS FPS 和 UI FPS 判断是哪个线程卡。
2. 关键机制：JS 线程卡多是重渲染和大计算，UI 线程卡多是视图复杂和大图。
3. 对比或注意点：必须在 Release 包中测量。
4. 可追问时补充：用 React Profiler 定位重渲染，动画移到 UI 线程。

**常见追问**

- 为什么开发模式下特别卡？→ 开发模式有额外的检查、警告和调试支持，不代表真实性能。

---

### Q9. FlatList 怎么优化？

- **难度**: 中级
- **标签**: RN, 列表

**答案**

| 手段 | 说明 |
|------|------|
| `keyExtractor` | 稳定的 key |
| **`getItemLayout`** | 固定高度时提供，跳过测量，`scrollToIndex` 更准 |
| **列表项 `memo`** | 避免父组件更新导致所有项重渲染；`renderItem` 用 `useCallback` |
| `initialNumToRender` | 首屏渲染数量 |
| `windowSize` | 渲染窗口（屏幕数），越小内存越低但越容易白块 |
| `maxToRenderPerBatch` / `updateCellsBatchingPeriod` | 每批渲染数量 |
| `removeClippedSubviews` | 移除屏幕外的原生视图（Android 效果明显） |
| 列表项轻量 | 减少嵌套、图片使用缩略图和缓存 |

**根本限制**：FlatList 在滚动时**创建和销毁**列表项，渲染跟不上就出现白块 → 使用 FlashList（复用）。

**口述模板（30–60 秒）**

1. 一句话结论：FlatList 优化的重点是减少重渲染和控制渲染窗口。
2. 关键机制：固定高度时提供 getItemLayout，列表项用 memo。
3. 对比或注意点：windowSize 越小内存越低，但越容易出现白块。
4. 可追问时补充：FlatList 会销毁和创建列表项，长列表推荐 FlashList。

**常见追问**

- `renderItem` 中写匿名函数有什么问题？→ 每次渲染都是新的函数和新的元素，配合 memo 的列表项也会失效。

---

### Q10. FlashList 为什么比 FlatList 快？

- **难度**: 中级
- **标签**: RN, 列表

**答案**

- **视图复用（回收）**：离开屏幕的列表项不销毁，而是用新数据**复用**组件实例（类似原生的 RecyclerView / UICollectionView）→ 避免频繁的创建和销毁
- 更少的白块，更低的内存
- FlashList v2 基于新架构，不再需要手动估算尺寸（以官方文档为准）

**注意**：复用意味着组件内部的 **state 会被带到新的数据项上** → 列表项中的状态要根据 item 重置（或者不在列表项中保存与 item 相关的 state）。

**口述模板（30–60 秒）**

1. 一句话结论：FlashList 通过复用列表项提升性能。
2. 关键机制：离开屏幕的项不销毁，换上新数据继续使用。
3. 对比或注意点：复用会带着组件内部的 state，要注意重置。
4. 可追问时补充：异构列表要提供 getItemType，同类型的项之间才复用。

**常见追问**

- 为什么要提供 `getItemType`？→ 不同类型的列表项结构不同，复用时只在同类型之间复用效率才高。

---

### Q11. RN 的动画怎么做才流畅？

- **难度**: 中级
- **标签**: RN, 动画

**答案**

**原则**：动画要在 **UI 线程**执行，不依赖 JS 线程（JS 繁忙时动画仍然流畅）。

| 方案 | 说明 |
|------|------|
| `Animated` + `useNativeDriver: true` | 动画配置一次性发送给原生执行；**只支持 transform 和 opacity** 等非布局属性 |
| **Reanimated** | `useSharedValue` + `useAnimatedStyle`；动画逻辑写成 **worklet**，在 UI 线程的 JS 运行时中执行；可以动画布局属性 |
| Gesture Handler | 手势在 UI 线程识别，与 Reanimated 配合实现跟手动画 |
| LayoutAnimation | 简单的布局变化动画 |

```tsx
const offset = useSharedValue(0);
const style = useAnimatedStyle(() => ({ transform: [{ translateX: offset.value }] }));
const pan = Gesture.Pan().onChange((e) => {
  offset.value += e.changeX; // 在 UI 线程执行，不经过 JS 线程
});
```

**口述模板（30–60 秒）**

1. 一句话结论：流畅动画的关键是在 UI 线程执行。
2. 关键机制：Reanimated 的 worklet 在 UI 线程的 JS 运行时中运行。
3. 对比或注意点：Animated 的原生驱动只支持 transform 和 opacity。
4. 可追问时补充：手势用 Gesture Handler 在 UI 线程识别，配合 Reanimated 跟手。

**常见追问**

- worklet 中能调用普通 JS 函数吗？→ 不能直接调用，需要用 `runOnJS`（新版本中为 `scheduleOnRN` 等 API，以官方为准）回到 JS 线程。

---

### Q12. RN App 的启动怎么优化？

- **难度**: 中级
- **标签**: RN, 启动

**答案**

```text
原生初始化 → 加载 JS bundle → 执行 JS → 首屏渲染 → 数据请求 → 可交互
```

| 阶段 | 优化 |
|------|------|
| 原生初始化 | 延迟非必要的 SDK 初始化；TurboModules 懒加载 |
| 加载 bundle | **Hermes 字节码**；减小 bundle（移除无用依赖）|
| 执行 JS | **延迟 require**（inline requires）：模块用到时才执行；避免入口文件中的重量级 import |
| 首屏 | 骨架屏；首屏数据缓存（先显示缓存再刷新）；减少首屏组件 |
| 数据 | 并行请求；原生层提前发起请求（与 JS 加载并行） |

**度量**：TTI（从点击图标到可交互）；冷启动和热启动分开统计。

**口述模板（30–60 秒）**

1. 一句话结论：启动优化覆盖原生初始化、加载 bundle、执行 JS、首屏渲染和数据请求。
2. 关键机制：Hermes 字节码、inline requires 和 TurboModules 懒加载。
3. 对比或注意点：延迟初始化非首屏需要的 SDK。
4. 可追问时补充：首屏数据先显示缓存，原生层可以和 JS 加载并行预取数据。

**常见追问**

- inline requires 是什么？→ Metro 把模块顶层的 `import` 转换为用到时才执行的 `require`，减少启动时执行的代码。

---

### Q13. RN 中常见的内存问题？

- **难度**: 中级
- **标签**: RN, 内存

**答案**

| 问题 | 处理 |
|------|------|
| 大图 | 按显示尺寸请求；图片库的内存缓存上限 |
| 订阅未清理 | `useEffect` 返回清理函数（事件、定时器、监听） |
| 页面栈中保留大量页面 | 栈导航中旧页面不卸载 → 控制栈深度；不可见时释放资源 |
| 列表 | 虚拟化；FlashList 复用 |
| 全局缓存无限增长 | LRU 限制 |

**工具**：Hermes 的堆快照（DevTools Memory 面板）、Android Studio Profiler、Xcode Instruments。

**口述模板（30–60 秒）**

1. 一句话结论：RN 的内存问题主要来自大图、未清理的订阅和页面栈。
2. 关键机制：栈导航中旧页面不会卸载，会一直占用内存。
3. 对比或注意点：useEffect 中的订阅必须清理。
4. 可追问时补充：用 Hermes 的堆快照对比定位泄漏。

**常见追问**

- 页面不可见时怎么释放资源？→ `useFocusEffect` 在页面失去焦点时暂停视频、停止轮询。

---

### Q14. RN 的手势冲突怎么处理？

- **难度**: 中级
- **标签**: RN, 手势

**答案**

场景：横向滑动的轮播在竖向滚动列表中、可拖动卡片在 ScrollView 中、侧滑返回与页面内横滑。

**react-native-gesture-handler** 提供组合方式：

- `Gesture.Simultaneous(a, b)`：同时识别
- `Gesture.Exclusive(a, b)`：优先识别 a，失败后识别 b
- `Gesture.Race(a, b)`：先激活的胜出
- `activeOffsetX` / `failOffsetY`：根据方向决定是否激活（例如横向移动超过 10 才激活，竖向移动超过 10 就失败）
- `simultaneousWithExternalGesture` / `requireExternalGestureToFail`：与外部手势的关系

**口述模板（30–60 秒）**

1. 一句话结论：用 Gesture Handler 的组合 API 和激活条件处理手势冲突。
2. 关键机制：根据移动方向设置激活和失败的阈值。
3. 对比或注意点：Exclusive 和 Race 决定手势的优先级。
4. 可追问时补充：与外部手势的关系可以用 requireExternalGestureToFail 声明。

**常见追问**

- 为什么不用 RN 自带的 PanResponder？→ 它在 JS 线程中处理，跟手延迟大，也难以和原生滚动协调。

---

## 三、React Native 工程

### Q15. 怎么开发一个 RN 原生模块？

- **难度**: 中级
- **标签**: RN, 原生模块

**答案**

**方式 1：Expo Modules API**（推荐，Swift / Kotlin）：

```swift
// iOS
public class DeviceInfoModule: Module {
  public func definition() -> ModuleDefinition {
    Name("DeviceInfo")
    Function("getBatteryLevel") { () -> Float in
      UIDevice.current.isBatteryMonitoringEnabled = true
      return UIDevice.current.batteryLevel
    }
    AsyncFunction("fetchSomething") { (id: String) -> String in /* ... */ }
    Events("onChange")
  }
}
```

**方式 2：Turbo Native Module**：写 TypeScript spec → Codegen → 实现原生接口（Objective-C++ / Kotlin）。

**设计要点**：

- 耗时操作用异步（不阻塞 JS 线程）
- 事件用事件发射器（例如传感器、下载进度）
- 错误转换为 JS 异常
- iOS 和 Android 接口一致

**口述模板（30–60 秒）**

1. 一句话结论：推荐用 Expo Modules API，也可以用 TypeScript spec 加 Codegen 写 TurboModule。
2. 关键机制：接口由 TypeScript 定义，原生用 Swift 或 Kotlin 实现。
3. 对比或注意点：耗时操作必须异步。
4. 可追问时补充：持续的数据用事件推送给 JS。

**常见追问**

- 原生 UI 组件怎么封装？→ Fabric 原生组件（Codegen 生成属性接口）或 Expo Modules 的 `View` 定义。

---

### Q16. RN 热更新的原理是什么？

- **难度**: 中级
- **标签**: RN, 热更新

**答案**

**原理**：RN 的业务逻辑在 JS bundle 中，App 启动时从本地加载 bundle → **下载新的 bundle 替换旧的**，下次启动（或立即重载）时加载新 bundle。

```text
App 启动 → 检查更新（当前版本、原生版本、渠道）
         → 有更新：下载 bundle（全量或差量）→ 校验哈希 / 签名 → 保存
         → 下次启动加载新 bundle；启动失败则回滚到上一个版本
```

**限制**：**只能更新 JS 和资源**；原生代码变化必须发布新的 App 版本 → bundle 要与**原生版本（runtime version）**匹配。

**方案**：EAS Update（Expo）；CodePush 已随 App Center 停止服务（2025-03），需要迁移；自建。

**合规**：苹果允许通过热更新修复问题，但不能显著改变 App 的主要功能或绕过审核；国内应用商店也有相关要求。

**口述模板（30–60 秒）**

1. 一句话结论：RN 热更新就是下载新的 JS bundle 替换本地的 bundle。
2. 关键机制：检查更新、下载、校验、下次启动加载，失败时回滚。
3. 对比或注意点：只能更新 JS 和资源，bundle 要与原生版本匹配。
4. 可追问时补充：要遵守应用商店的规则，不能用热更新改变主要功能。

**常见追问**

- 热更新后 App 崩溃怎么办？→ 启动时标记「正在尝试新版本」，成功渲染后清除标记；如果下次启动发现标记还在，说明崩溃了，回滚到旧版本。

---

### Q17. Metro 是什么？和 Webpack 有什么不同？

- **难度**: 中级
- **标签**: RN, 打包

**答案**

Metro 是 RN 的打包工具：

- 解析模块依赖 → Babel 转换 → 打包为单个 bundle（加上 Hermes 编译为字节码）
- 支持平台扩展名解析（`.ios.js`、`.android.js`）
- 开发服务器、Fast Refresh
- inline requires、资源处理（`@2x`、`@3x`）

**与 Webpack 的不同**：

- 默认输出**单个 bundle**（移动端从本地加载，不需要按路由拆分以减少网络请求）
- 没有 Webpack 那样的插件和 loader 生态
- 支持 `package.json` 的 `exports`、符号链接（Monorepo）
- Module Federation 2.0 支持 Metro，可以实现 RN 中的远程模块

**口述模板（30–60 秒）**

1. 一句话结论：Metro 是 RN 的打包工具，负责解析、转换和打包 JS。
2. 关键机制：默认输出单个 bundle，支持平台扩展名。
3. 对比或注意点：移动端从本地加载 bundle，拆包不是为了网络而是为了按需执行和独立更新。
4. 可追问时补充：Module Federation 2.0 支持 Metro。

**常见追问**

- 什么时候需要拆包？→ 大型 App 中多个业务独立开发和更新，拆成基础包 + 业务包，减少热更新的体积。

---

### Q18. RN 中的状态管理和数据层怎么设计？

- **难度**: 中级
- **标签**: RN, 状态

**答案**

与 React Web 类似，按状态的类型选择：

| 类型 | 方案 |
|------|------|
| 服务端数据 | TanStack Query（缓存、重试、**焦点刷新改为 App 前台刷新**） |
| 全局客户端状态 | Zustand、Redux Toolkit、Jotai |
| 持久化 | MMKV + 状态库的持久化中间件 |
| 表单 | react-hook-form |
| 离线优先 | WatermelonDB、本地 SQLite + 同步 |

**移动端的特点**：

- App 前后台切换（`AppState`）时刷新数据
- 网络状态（`@react-native-community/netinfo`）：离线时暂停请求、恢复时重试
- 冷启动时从持久化恢复状态

**口述模板（30–60 秒）**

1. 一句话结论：服务端数据用 TanStack Query，客户端状态用 Zustand 或 Redux Toolkit。
2. 关键机制：持久化用 MMKV，离线优先用本地数据库加同步。
3. 对比或注意点：移动端要处理前后台切换和网络状态变化。
4. 可追问时补充：冷启动时从持久化恢复状态，提升首屏速度。

**常见追问**

- TanStack Query 在 RN 中要额外配置什么？→ 用 `focusManager` 监听 `AppState`，用 `onlineManager` 监听网络状态。

---

### Q19. RN 怎么做测试？

- **难度**: 中级
- **标签**: RN, 测试

**答案**

| 层次 | 工具 |
|------|------|
| 单元 / 组件 | Jest + React Native Testing Library（按用户可见的文本和角色查询） |
| E2E | **Maestro**（YAML 编写，简单稳定）、Detox（灰盒，自动等待） |
| 原生模块 | XCTest、JUnit |
| 视觉 | 截图对比 |

**注意**：组件测试中原生模块需要 mock；E2E 在 CI 中需要模拟器（耗时，选核心流程）。

**口述模板（30–60 秒）**

1. 一句话结论：组件测试用 Jest 和 Testing Library，E2E 用 Maestro 或 Detox。
2. 关键机制：组件测试按用户可见的内容查询元素。
3. 对比或注意点：原生模块在组件测试中需要 mock。
4. 可追问时补充：E2E 成本高，只覆盖核心流程。

**常见追问**

- Detox 和 Maestro 怎么选？→ Maestro 上手快、跨平台；Detox 与 App 同步更好（自动等待动画和请求），适合复杂场景。

---

## 四、小程序原理

### Q20. 小程序为什么采用双线程模型？

- **难度**: 中级
- **标签**: 小程序, 原理

**答案**

```text
┌──────────── 渲染层（WebView，每个页面一个）────────────┐
│  WXML / WXSS 渲染                                   │
└───────────────────────▲─────────────────────────────┘
                        │ setData / 事件（经 Native 中转）
                ┌───────┴───────┐
                │  Native（客户端）│  ← 原生 API、网络、存储
                └───────▲───────┘
┌───────────────────────┴─────────────────────────────┐
│  逻辑层（JS 引擎，独立线程，所有页面共享）               │
│  App / Page / Component 的 JS，没有 DOM / BOM        │
└─────────────────────────────────────────────────────┘
```

**原因**：

1. **管控和安全**：逻辑层没有 DOM 和 `window`，开发者无法通过 JS 操作页面、跳转外部链接、执行动态代码 → 平台可以审核和管控
2. **性能**：JS 执行不阻塞渲染；多个页面的 WebView 可以预加载
3. 统一的组件和能力由平台提供

**代价**：逻辑层和渲染层的通信需要**跨线程和序列化** → setData 有开销；交互响应有延迟（例如拖动跟手）。

**口述模板（30–60 秒）**

1. 一句话结论：小程序把逻辑层和渲染层分开，主要是为了管控和安全。
2. 关键机制：逻辑层运行在独立的 JS 引擎中，没有 DOM，只能通过 setData 更新视图。
3. 对比或注意点：代价是两层之间的通信需要跨线程和序列化。
4. 可追问时补充：WXS 和 Skyline 的 Worklet 就是为了减少跨线程通信。

**常见追问**

- 逻辑层在不同平台用什么引擎？→ iOS 用 JavaScriptCore，Android 用 V8 等（以官方为准）；开发者工具用 NW.js。

---

### Q21. setData 的过程是什么？为什么慢？

- **难度**: 中级
- **标签**: 小程序, setData

**答案**

```text
逻辑层：setData(data)
  → 合并到 this.data
  → 序列化为字符串（JSON）
  → 经过 Native 传递到渲染层（evaluateJavascript）
渲染层：反序列化
  → 更新虚拟 DOM，diff
  → 更新真实 DOM，渲染
```

**开销来源**：

1. **序列化 / 反序列化**：与数据大小成正比
2. **跨线程传输**：每次调用都有固定开销，频繁调用累积
3. **渲染层更新**：diff 和 DOM 操作，与受影响的节点数量有关
4. 渲染层繁忙时 setData 会排队，用户交互的响应变慢

**口述模板（30–60 秒）**

1. 一句话结论：setData 要经过序列化、跨线程传输和渲染层的 diff 更新。
2. 关键机制：开销与数据大小和调用频率成正比。
3. 对比或注意点：频繁的小 setData 和一次大 setData 都会导致卡顿。
4. 可追问时补充：优化方向是减少数据量、减少调用次数、减少受影响的节点。

**常见追问**

- setData 是同步的吗？→ 修改 `this.data` 是同步的，视图更新是异步的；回调在渲染完成后执行。

---

### Q22. setData 怎么优化？

- **难度**: 中级
- **标签**: 小程序, 性能

**答案**

| 手段 | 说明 |
|------|------|
| **只传变化的数据** | 用路径更新：`this.setData({ 'list[3].liked': true })`，而不是整个 list |
| **不传与渲染无关的数据** | 不需要渲染的数据放在 `this` 的其他属性上（或纯数据字段 `pureDataPattern`） |
| **合并调用** | 同一时间的多次 setData 合并为一次 |
| **降低频率** | 滚动、拖动中节流；交互动画用 WXS / Worklet 在渲染层完成 |
| **长列表** | 分页追加：`'list[20]': newItem` 只追加新的项；分屏渲染；虚拟列表 |
| **组件化** | 数据下沉到组件，组件的 setData 只影响组件内部的节点 |
| 后台页面 | 页面在后台时不 setData |

**口述模板（30–60 秒）**

1. 一句话结论：setData 优化的核心是减少数据量和调用次数。
2. 关键机制：用路径语法只传变化的字段，不需要渲染的数据不放进 data。
3. 对比或注意点：高频交互用 WXS 或 Worklet 在渲染层处理。
4. 可追问时补充：长列表分页追加，用组件缩小更新范围。

**常见追问**

- 列表追加时为什么不直接 `setData({ list: [...old, ...new] })`？→ 会把整个列表重新序列化传输；用路径只传新增的项。

---

### Q23. 小程序的启动流程和优化？

- **难度**: 中级
- **标签**: 小程序, 启动

**答案**

```text
资源准备：下载代码包（首次 / 更新）、准备运行环境
→ 代码注入：执行逻辑层和渲染层的代码
→ 首屏渲染：页面 onLoad → 请求数据 → setData → 渲染
```

| 方向 | 手段 |
|------|------|
| **减小代码包** | 分包、移除无用代码和图片（图片放 CDN）、**按需注入**（`lazyCodeLoading: "requiredComponents"`） |
| 代码注入 | 减少启动时执行的同步逻辑；延迟非必要的初始化 |
| **首屏数据** | **数据预拉取**（微信后台提前请求）、**周期性更新**；缓存上次的数据先显示 |
| 首屏渲染 | 骨架屏；首屏数据量小；**初始渲染缓存** |
| 分包预下载 | 预测下一个页面 |

**口述模板（30–60 秒）**

1. 一句话结论：启动分为资源准备、代码注入和首屏渲染三个阶段。
2. 关键机制：减小代码包和按需注入加快前两个阶段。
3. 对比或注意点：首屏数据用预拉取和本地缓存，避免等待请求。
4. 可追问时补充：开启初始渲染缓存可以直接显示上次的渲染结果。

**常见追问**

- 怎么度量启动性能？→ 开发者工具的性能面板、`wx.getPerformance()`、We 分析的启动耗时数据。

---

### Q24. 小程序的长列表怎么优化？

- **难度**: 中级
- **标签**: 小程序, 列表

**答案**

1. **分页加载 + 路径追加**：每页只 setData 新的项
2. **分屏渲染**：把数据按屏分组（二维数组），每次只更新一组
3. **虚拟列表**：只渲染可视区域附近的项，屏幕外用占位高度（需要知道高度）
4. 屏幕外的节点隐藏或替换为空占位（`IntersectionObserver` 判断可见性）
5. 列表项组件化，减少单次更新影响的节点
6. 官方方案：Skyline 下的 `list-view` / `grid-view`（按需渲染）；Taro 的 `VirtualList`

**口述模板（30–60 秒）**

1. 一句话结论：长列表优化的核心是控制 setData 的数据量和渲染层的节点数量。
2. 关键机制：分页时用路径追加，渲染层用虚拟列表只保留可视区域附近的节点。
3. 对比或注意点：屏幕外的节点可以替换为固定高度的占位。
4. 可追问时补充：Skyline 提供按需渲染的列表组件。

**常见追问**

- 虚拟列表在小程序中的难点？→ 滚动事件需要传回逻辑层计算，再 setData，有延迟；需要缓冲区（多渲染几屏）减少白屏。

---

### Q25. WXS 是什么？什么时候用？

- **难度**: 中级
- **标签**: 小程序, WXS

**答案**

WXS 是**运行在渲染层**的脚本语言（语法类似 ES5 的子集）：

- **过滤器 / 格式化**：在模板中格式化数据（价格、日期），不需要在逻辑层预处理
- **响应事件**：在渲染层直接处理触摸事件并修改样式（例如拖动、侧滑删除），**不经过逻辑层** → 跟手无延迟

```xml
<wxs module="fmt">
  module.exports.price = function (v) { return (v / 100).toFixed(2); };
</wxs>
<text>¥{{fmt.price(item.price)}}</text>
```

**限制**：不能调用小程序 API；语法受限；Skyline 中动画推荐用 Worklet 替代。

**口述模板（30–60 秒）**

1. 一句话结论：WXS 是运行在渲染层的脚本，用于格式化和响应交互。
2. 关键机制：在渲染层处理事件，不需要跨线程通信。
3. 对比或注意点：不能调用小程序 API，语法受限。
4. 可追问时补充：Skyline 中用 Worklet 动画替代 WXS 动画。

**常见追问**

- 为什么 WXS 能做到跟手？→ 触摸事件和样式修改都在渲染层，省去了「渲染层 → 逻辑层 → setData → 渲染层」的往返。

---

### Q26. Skyline 渲染引擎是什么？

- **难度**: 中级
- **标签**: 小程序, Skyline

**答案**

Skyline 是微信小程序的**新渲染引擎**，替代 WebView 渲染：

- **自研渲染管线**，不再是完整的浏览器内核 → 更轻量、启动和内存更好
- **独立的渲染线程**，布局和绘制不会被 JS 阻塞
- 页面之间**共享渲染线程**，不需要每个页面一个 WebView
- 新能力：**Worklet 动画**、手势系统、共享元素动画、自定义路由动画、按需渲染的列表组件
- 只支持 **glass-easel** 组件框架

**启用**：

```json
{
  "renderer": "skyline",
  "componentFramework": "glass-easel",
  "rendererOptions": { "skyline": { "defaultDisplayBlock": true } }
}
```

**注意**：只支持 WXSS 的子集（例如部分 CSS 特性不支持），需要兼容 WebView 时要做适配；可以按页面启用。

**口述模板（30–60 秒）**

1. 一句话结论：Skyline 是微信小程序的新渲染引擎，替代 WebView 渲染。
2. 关键机制：自研的轻量渲染管线，独立渲染线程，页面共享。
3. 对比或注意点：只支持 CSS 的子集，需要使用 glass-easel 组件框架。
4. 可追问时补充：带来了 Worklet 动画、手势和共享元素动画等能力。

**常见追问**

- 老项目怎么迁移到 Skyline？→ 按页面启用，从交互复杂的页面开始；检查样式兼容性；动画迁移到 Worklet。

---

### Q27. Worklet 动画是什么？

- **难度**: 中级
- **标签**: 小程序, Skyline, 动画

**答案**

Worklet 是可以在**渲染线程（UI 线程）中执行的函数**，概念与 RN Reanimated 的 worklet 相似：

```js
const { shared, timing } = wx.worklet;

Component({
  attached() {
    this.offset = shared(0); // 共享变量：两个线程都能访问
    this.applyAnimatedStyle('.box', () => {
      'worklet';
      return { transform: `translateX(${this.offset.value}px)` };
    });
  },
  methods: {
    move() {
      this.offset.value = timing(200, { duration: 300 });
    },
  },
});
```

- `'worklet'` 指令标记函数在 UI 线程运行
- 共享变量修改后，动画样式在 UI 线程更新，**不经过 setData**
- 手势回调也可以是 worklet，实现跟手效果

**口述模板（30–60 秒）**

1. 一句话结论：Worklet 是在 UI 线程执行的函数，用于实现流畅的动画和手势。
2. 关键机制：共享变量在两个线程间同步，动画样式在 UI 线程计算。
3. 对比或注意点：不需要 setData，避免了跨线程通信。
4. 可追问时补充：和 RN Reanimated 的 worklet 思路一致。

**常见追问**

- worklet 中能调用逻辑层的函数吗？→ 需要用 `runOnJS` 回到逻辑层。

---

### Q28. 小程序的原生组件和同层渲染是什么？

- **难度**: 中级
- **标签**: 小程序, 原生组件

**答案**

- **原生组件**：`video`、`map`、`camera`、`live-player`、`canvas`（旧版）等由客户端原生控件实现，不在 WebView 中渲染
- **旧问题**：原生组件盖在 WebView 之上，**层级最高**，`z-index` 无效，无法被其他元素覆盖，不能放在 `scroll-view` 中
- **同层渲染**：把原生组件**插入到 WebView 的渲染层级中**（iOS 利用 WKWebView 的 WKChildScrollView，Android 利用 Chromium 的扩展插件机制），使其可以和普通元素一样设置层级
- 现在大部分原生组件已经支持同层渲染

**口述模板（30–60 秒）**

1. 一句话结论：原生组件由客户端的原生控件实现，同层渲染把它们放进 WebView 的层级中。
2. 关键机制：同层渲染后，原生组件可以被普通元素覆盖。
3. 对比或注意点：旧版原生组件层级最高，z-index 无效。
4. 可追问时补充：iOS 和 Android 的同层渲染实现方式不同。

**常见追问**

- 早期怎么在 video 上覆盖内容？→ 使用 `cover-view` / `cover-image`（可以覆盖原生组件的特殊组件）。

---

## 五、跨端框架与混合开发

### Q29. Taro 3 的运行时原理是什么？

- **难度**: 中级
- **标签**: Taro, 原理

**答案**

**Taro 1 / 2（编译时）**：把 JSX 编译成 WXML 模板 → 受限于静态分析，很多 JSX 写法不支持。

**Taro 3+（运行时）**：

```text
React / Vue 代码
  → 框架正常运行，渲染到 Taro 实现的「模拟 DOM」（taro-runtime 中的 TaroElement 树，实现了 DOM / BOM 的子集）
  → Taro 记录 DOM 的变化，生成 setData 的路径数据（最小更新）
  → 小程序中预先生成的「通用模板」根据数据递归渲染节点
```

- **模板**：Taro 为每种组件生成模板，`<template is="tmpl_0_view" data="{{...}}">`；由于小程序模板不支持递归，Taro 生成固定层数的模板，超过层数时使用自定义组件中转
- **优点**：不限制框架语法，React / Vue 的生态可以使用
- **代价**：运行时开销（模拟 DOM、模板层级），setData 的数据结构较深

**口述模板（30–60 秒）**

1. 一句话结论：Taro 3 在小程序的逻辑层模拟了 DOM，让 React 和 Vue 正常运行。
2. 关键机制：模拟 DOM 的变化被转换为 setData 的最小路径更新。
3. 对比或注意点：渲染层用预先生成的通用模板递归渲染节点。
4. 可追问时补充：运行时方案不限制语法，但有额外的性能开销。

**常见追问**

- 为什么小程序模板需要「固定层数」？→ WXML 的 template 不能递归引用自身，所以 Taro 生成多层模板；节点层级过深时性能下降。

---

### Q30. Taro 的性能怎么优化？

- **难度**: 中级
- **标签**: Taro, 性能

**答案**

| 手段 | 说明 |
|------|------|
| **CustomWrapper** | 把组件包在自定义组件中，更新时 setData 的路径变短，只影响该组件 |
| **半编译模式** | 对静态的 JSX 结构在编译时生成 WXML，减少模板层级和运行时开销（Taro 3.6+ / 4.x） |
| 预渲染（prerender） | 首屏先渲染静态的骨架 |
| **VirtualList** | 长列表虚拟化 |
| `baseLevel` | 调整模板层数 |
| 减少不必要的重渲染 | memo、状态下沉 |
| 分包 | 与原生小程序相同 |

**口述模板（30–60 秒）**

1. 一句话结论：Taro 的优化重点是缩短 setData 的路径和减少模板层级。
2. 关键机制：CustomWrapper 让更新只影响包装的组件。
3. 对比或注意点：半编译模式在编译时为静态结构生成 WXML。
4. 可追问时补充：长列表用 VirtualList，React 层面减少重渲染。

**常见追问**

- 为什么 setData 的路径深会慢？→ 渲染层更新时需要沿着路径找到节点，且数据序列化后更大。

---

### Q31. uni-app 的原理是什么？

- **难度**: 中级
- **标签**: uni-app, 原理

**答案**

- **小程序端**：**编译时为主**——Vue 模板编译为 WXML 等平台模板，Vue 运行时（修改过的版本）在逻辑层运行，数据变化 diff 后调用 setData
- **H5**：普通的 Vue 应用
- **App 端（uni-app）**：WebView 渲染（vue 页面）或原生渲染（nvue，基于 Weex 思路）；逻辑层运行在 JS 引擎中，与渲染层分离
- **uni-app x**：UTS 编译为 Kotlin / Swift / ArkTS，**没有 JS 引擎**，性能接近原生；生态需要原生插件
- **条件编译**：编译时按平台裁剪代码

**与 Taro 3 的区别**：uni-app 的小程序端是编译模板（接近原生小程序的结构）；Taro 3 是运行时模拟 DOM 加通用模板。

**口述模板（30–60 秒）**

1. 一句话结论：uni-app 的小程序端以编译为主，Vue 模板编译为小程序模板。
2. 关键机制：Vue 运行时 diff 数据后调用 setData。
3. 对比或注意点：Taro 3 是运行时方案，uni-app 更接近编译时方案。
4. 可追问时补充：uni-app x 用 UTS 编译为原生代码，没有 JS 引擎。

**常见追问**

- 编译时方案的限制？→ 模板语法要能被静态编译到目标平台；动态的渲染函数支持有限。

---

### Q32. WebView 混合页面的首屏怎么优化？

- **难度**: 中级
- **标签**: Hybrid, 性能

**答案**

```text
点击 → 创建 WebView（几百毫秒）→ 请求 HTML → 下载 JS/CSS → 执行 → 请求数据 → 渲染
```

| 环节 | 优化 |
|------|------|
| WebView 初始化 | **WebView 预创建和复用**（App 启动后空闲时创建，放入池中） |
| HTML 和资源 | **离线包**：资源提前下载到本地，WebView 请求时拦截并返回本地文件；或者 HTTP 缓存 + CDN |
| 数据 | **数据预取**：原生在打开 WebView 的同时并行请求数据，H5 通过 JSBridge 获取 |
| 渲染 | SSR / 静态生成；骨架屏 |
| 连接 | DNS 预解析、连接复用 |

**离线包的要点**：版本管理和差量更新；拦截请求的方式（Android `shouldInterceptRequest`、iOS `WKURLSchemeHandler`）；失效时回退到线上资源。

**口述模板（30–60 秒）**

1. 一句话结论：混合页面的首屏优化覆盖 WebView 初始化、资源加载和数据请求。
2. 关键机制：WebView 预创建复用，离线包让资源从本地加载。
3. 对比或注意点：原生在打开 WebView 时并行预取数据。
4. 可追问时补充：离线包要有版本管理和回退到线上的机制。

**常见追问**

- 离线包和 Service Worker 有什么区别？→ 离线包由原生控制下载和拦截，首次打开即可命中；Service Worker 需要页面先访问一次才能注册，且 iOS WebView 的支持受限。
