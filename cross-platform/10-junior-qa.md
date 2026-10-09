# 跨端开发初级 Q&A

> 目标：了解跨端方案的分类，能用 React Native 和微信小程序开发常规页面，知道 Taro / uni-app、WebView 混合开发的基本概念。

---

## 一、跨端概览

### Q1. 跨端方案有哪些类型？各有什么优缺点？

- **难度**: 初级
- **标签**: 概览, 选型

**答案**

| 类型 | 代表 | 渲染方式 | 优点 | 缺点 |
|------|------|----------|------|------|
| **Web 容器** | H5 + WebView、Hybrid | 浏览器内核渲染 | 开发快、动态发布、一套代码多端 | 性能和体验不如原生，能力受限 |
| **JS + 原生渲染** | React Native、小程序（WebView 渲染之外的原生组件） | JS 描述 UI，映射到原生控件 | 体验接近原生、可热更新 | 有通信开销，平台差异需要处理 |
| **自绘引擎** | Flutter、小程序 Skyline | 自己的渲染引擎直接绘制 | 多端一致性好、性能高 | 包体积大，与原生控件的融合成本 |
| **编译到原生** | uni-app x、Kotlin Multiplatform | 编译成原生代码 | 性能接近原生 | 生态和动态性较弱 |

**口述模板（30–60 秒）**

1. 一句话结论：跨端方案按渲染方式分为 Web 容器、JS 加原生渲染、自绘引擎、编译到原生四类。
2. 关键机制：区别在于 UI 由谁来画——浏览器、原生控件还是自己的引擎。
3. 对比或注意点：越接近原生性能越好，但动态性和开发效率可能下降。
4. 可追问时补充：小程序默认是 WebView 渲染，Skyline 是自绘方式。

**常见追问**

- RN 和 Flutter 的本质区别？→ RN 把 UI 映射到平台原生控件；Flutter 用自己的引擎绘制所有控件，不依赖平台控件。

---

### Q2. React Native、小程序、H5 分别适合什么场景？

- **难度**: 初级
- **标签**: 选型

**答案**

| 方案 | 适合场景 |
|------|----------|
| React Native | 需要原生体验和原生能力的独立 App，团队熟悉 React |
| 小程序 | 依托超级 App 的流量（微信、支付宝），即用即走，获客成本低 |
| H5 | 活动页、营销页、需要快速迭代和分享传播的页面 |

实际中常组合使用：App 主体用原生或 RN，活动页用 H5，再做一个小程序版本获取微信流量。

**口述模板（30–60 秒）**

1. 一句话结论：RN 做 App，小程序借平台流量，H5 做快速迭代的页面。
2. 关键机制：选型取决于流量入口、体验要求和迭代速度。
3. 对比或注意点：小程序受平台规则和能力限制。
4. 可追问时补充：多端需求可以用 Taro 或 uni-app 一套代码覆盖小程序和 H5。

**常见追问**

- 为什么不全部用 H5？→ 长列表、复杂动画、手势的体验差；无法使用部分原生能力；启动依赖网络。

---

## 二、React Native 基础

### Q3. React Native 是什么？它是怎么运行的？

- **难度**: 初级
- **标签**: RN, 原理

**答案**

React Native 让开发者用 React 编写 App，UI 最终渲染为**平台原生控件**（iOS 的 UIView、Android 的 View），而不是 WebView。

**运行时的主要线程**：

- **JS 线程**：执行业务代码和 React 渲染（Hermes 引擎）
- **UI 线程（主线程）**：原生控件的绘制和用户交互
- **后台线程**：Yoga 计算布局（Flexbox → 坐标）

**流程**：React 计算出组件树 → 渲染器生成 Shadow Tree 并计算布局 → 提交给 UI 线程创建和更新原生控件。

**口述模板（30–60 秒）**

1. 一句话结论：RN 用 React 描述 UI，最终渲染成原生控件。
2. 关键机制：JS 线程运行业务代码，布局由 Yoga 计算，UI 线程负责绘制。
3. 对比或注意点：和 WebView 方案不同，它没有 DOM 和 CSS 引擎。
4. 可追问时补充：新架构用 JSI 让 JS 和原生直接通信，不再经过异步的 Bridge。

**常见追问**

- RN 中可以用 `div` 吗？→ 不可以，要用 `View`、`Text` 等 RN 组件；没有 DOM。

---

### Q4. RN 的核心组件有哪些？

- **难度**: 初级
- **标签**: RN, 组件

**答案**

| 组件 | 作用 | 对应 Web |
|------|------|----------|
| `View` | 容器 | `div` |
| `Text` | 文本（**所有文字必须在 Text 中**） | `span` / `p` |
| `Image` | 图片 | `img` |
| `TextInput` | 输入框 | `input` |
| `ScrollView` | 可滚动容器（一次性渲染所有子元素） | 滚动的 `div` |
| `FlatList` / `SectionList` | 虚拟化长列表 | — |
| `Pressable` | 可点击区域（推荐，取代 Touchable 系列） | `button` |
| `Modal` | 弹窗 | — |
| `SafeAreaView` | 避开刘海和底部指示条（推荐使用 `react-native-safe-area-context`） | — |

**口述模板（30–60 秒）**

1. 一句话结论：RN 用 View、Text、Image 等组件代替 HTML 标签。
2. 关键机制：所有文字必须放在 Text 中。
3. 对比或注意点：ScrollView 一次性渲染全部子元素，长列表要用 FlatList。
4. 可追问时补充：点击用 Pressable，可以根据按下状态设置样式。

**常见追问**

- 为什么文字必须在 Text 里？→ 原生平台的文本由专门的文本控件渲染，View 无法直接显示文字。

---

### Q5. RN 的样式和 Web CSS 有什么区别？

- **难度**: 初级
- **标签**: RN, 样式

**答案**

- 用 JS 对象（`StyleSheet.create`），属性名用驼峰（`backgroundColor`）
- **没有层叠和继承**（只有 Text 嵌套时部分文字样式继承）
- 没有选择器、伪类；样式通过 `style` 属性直接作用
- 布局只有 **Flexbox**，且默认 `flexDirection: 'column'`（Web 默认 row）
- 尺寸单位是**与密度无关的像素**（dp / pt），不写单位
- `flex: 1` 的含义与 Web 略有不同（RN 中 flex 是单个数字）
- 没有 `display: grid`、`float`

```tsx
const styles = StyleSheet.create({
  container: { flex: 1, padding: 16, backgroundColor: '#fff' },
  title: { fontSize: 18, fontWeight: '600' },
});
```

**口述模板（30–60 秒）**

1. 一句话结论：RN 的样式是 JS 对象，只有 Flexbox 布局，没有层叠和选择器。
2. 关键机制：默认主轴方向是 column，尺寸没有单位。
3. 对比或注意点：文字样式只在嵌套的 Text 之间继承。
4. 可追问时补充：可以用 NativeWind 等库在 RN 中使用 Tailwind 风格。

**常见追问**

- 怎么适配不同屏幕？→ Flexbox 弹性布局为主；`useWindowDimensions` 获取尺寸；按比例计算；字体随系统缩放要测试。

---

### Q6. 怎么处理 iOS 和 Android 的平台差异？

- **难度**: 初级
- **标签**: RN, 平台差异

**答案**

```tsx
import { Platform } from 'react-native';

const styles = StyleSheet.create({
  card: {
    ...Platform.select({
      ios: { shadowColor: '#000', shadowOpacity: 0.1, shadowRadius: 8 },
      android: { elevation: 4 },
    }),
  },
});

if (Platform.OS === 'android') { /* ... */ }
```

- **平台扩展名**：`Button.ios.tsx`、`Button.android.tsx`，引用 `./Button` 时自动选择
- 常见差异：阴影（iOS shadow vs Android elevation）、返回键（Android 物理返回）、状态栏、键盘行为、权限、字体

**口述模板（30–60 秒）**

1. 一句话结论：用 Platform.OS、Platform.select 和平台扩展名处理差异。
2. 关键机制：差异较大的组件用 `.ios.tsx` / `.android.tsx` 分文件实现。
3. 对比或注意点：阴影、返回键、键盘、状态栏是常见的差异点。
4. 可追问时补充：Android 的物理返回键用 BackHandler 处理。

**常见追问**

- Android 返回键怎么处理？→ `BackHandler.addEventListener('hardwareBackPress', ...)`，返回 true 表示已处理；导航库一般已内置。

---

### Q7. ScrollView 和 FlatList 有什么区别？

- **难度**: 初级
- **标签**: RN, 列表

**答案**

| | ScrollView | FlatList |
|--|-----------|----------|
| 渲染 | 一次性渲染所有子元素 | **虚拟化**：只渲染可视区域附近的项 |
| 适用 | 内容少、固定的页面 | 长列表、数据量不确定 |
| 功能 | 基础滚动 | 下拉刷新、上拉加载、分隔线、头尾组件 |

```tsx
<FlatList
  data={items}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <Row item={item} />}
  onEndReached={loadMore}
  onEndReachedThreshold={0.5}
  refreshing={refreshing}
  onRefresh={refresh}
/>
```

**口述模板（30–60 秒）**

1. 一句话结论：ScrollView 一次性渲染全部内容，FlatList 只渲染可视区域附近的项。
2. 关键机制：FlatList 是虚拟化列表，适合长列表。
3. 对比或注意点：要提供稳定的 keyExtractor。
4. 可追问时补充：性能要求高时可以用 FlashList，它会复用列表项。

**常见追问**

- `onEndReached` 被触发多次怎么办？→ 用加载中标记防止重复请求。

---

### Q8. RN 中怎么做页面导航？

- **难度**: 初级
- **标签**: RN, 导航

**答案**

- **React Navigation**：最常用的导航库；栈导航（Native Stack，使用原生导航控件）、Tab、抽屉
- **Expo Router**：基于文件系统的路由（类似 Next.js），构建在 React Navigation 之上，支持深链接

```tsx
// Expo Router: app/(tabs)/index.tsx、app/user/[id].tsx
import { Link, useLocalSearchParams } from 'expo-router';

<Link href="/user/42">查看用户</Link>;

export default function User() {
  const { id } = useLocalSearchParams<{ id: string }>();
  return <Text>用户 {id}</Text>;
}
```

**口述模板（30–60 秒）**

1. 一句话结论：RN 常用 React Navigation，Expo 项目常用基于文件的 Expo Router。
2. 关键机制：栈导航、Tab 导航、抽屉导航可以嵌套组合。
3. 对比或注意点：Native Stack 使用原生的导航控件，性能和手势更好。
4. 可追问时补充：深链接要配置 URL 和页面的映射。

**常见追问**

- 页面之间怎么传参？→ 路由参数只传简单值（ID），复杂数据通过状态管理或请求获取。

---

### Q9. Expo 是什么？和裸 RN 有什么区别？

- **难度**: 初级
- **标签**: RN, Expo

**答案**

Expo 是 RN 的框架和工具链（React Native 官方推荐用框架创建新项目）：

- **SDK**：相机、通知、文件等常用原生模块
- **Expo Router**：文件路由
- **开发构建（Development Build）**：可以包含任意自定义原生代码（不再是「只能用 Expo Go」）
- **Prebuild / Config Plugins**：从配置生成原生工程，无需手动维护 ios / android 目录
- **EAS**：云端构建（Build）、提交到商店（Submit）、热更新（Update）

**区别**：裸 RN 需要自己维护原生工程和工具；Expo 用配置管理原生工程，升级更容易。现在 Expo 也可以使用任意原生库，限制已经很少。

**口述模板（30–60 秒）**

1. 一句话结论：Expo 是 RN 的官方推荐框架，提供 SDK、路由、构建和热更新。
2. 关键机制：Prebuild 从配置生成原生工程，Config Plugins 修改原生配置。
3. 对比或注意点：开发构建可以包含任意原生代码，不受 Expo Go 限制。
4. 可追问时补充：EAS 提供云构建、商店提交和热更新。

**常见追问**

- Expo Go 和开发构建的区别？→ Expo Go 是预装了固定原生模块的 App，适合入门；开发构建是包含项目自身原生代码的调试版 App。

---

### Q10. RN 中图片和资源怎么处理？

- **难度**: 初级
- **标签**: RN, 图片

**答案**

```tsx
<Image source={require('./logo.png')} />              // 本地：打包时处理，自动选择 @2x / @3x
<Image source={{ uri: 'https://...' }} style={{ width: 100, height: 100 }} /> // 网络：必须指定尺寸
```

- 网络图片必须指定宽高（无法预知尺寸）
- 推荐使用 `expo-image` 或 `react-native-fast-image`：磁盘缓存、占位、渐显
- 大图要按显示尺寸请求合适大小，避免内存过高

**口述模板（30–60 秒）**

1. 一句话结论：本地图片用 require，网络图片用 uri 并且必须指定尺寸。
2. 关键机制：本地图片按设备密度自动选择 @2x / @3x。
3. 对比或注意点：图片缓存推荐使用 expo-image 等库。
4. 可追问时补充：大图会占用很多内存，要按显示尺寸请求。

**常见追问**

- 图片列表滚动时内存上涨怎么办？→ 请求缩略图；使用带缓存和回收的图片库；列表虚拟化。

---

### Q11. RN 中怎么请求网络和存储数据？

- **难度**: 初级
- **标签**: RN, 网络, 存储

**答案**

- **网络**：内置 `fetch`；也可以用 axios；数据层推荐 TanStack Query（缓存、重试、刷新）
- **存储**：
  - `AsyncStorage`：简单的键值存储（异步，不加密）
  - MMKV：高性能的同步键值存储
  - **敏感数据**（令牌）：`expo-secure-store` / Keychain（iOS）/ Keystore（Android）
  - 结构化数据：SQLite（`expo-sqlite`）、WatermelonDB

**注意**：Android 9+ 默认禁止明文 HTTP；iOS 有 ATS 限制。

**口述模板（30–60 秒）**

1. 一句话结论：网络请求用 fetch，存储按数据类型选择 AsyncStorage、MMKV、安全存储或 SQLite。
2. 关键机制：令牌等敏感数据要存到 Keychain / Keystore。
3. 对比或注意点：AsyncStorage 不加密，不能存敏感数据。
4. 可追问时补充：平台默认禁止明文 HTTP。

**常见追问**

- 为什么 MMKV 比 AsyncStorage 快？→ 基于内存映射文件，通过 JSI 同步调用，没有旧 Bridge 的异步序列化开销。

---

### Q12. RN 怎么调试？

- **难度**: 初级
- **标签**: RN, 调试

**答案**

- **React Native DevTools**（0.76 起的默认调试工具）：基于 Chrome DevTools 前端，支持断点、Console、React 组件树、Profiler
- 开发菜单：摇晃设备或快捷键打开；Fast Refresh（保存后保留状态刷新）
- 网络调试：DevTools 的网络面板（新版本支持）、代理工具（Charles / Proxyman）
- 原生层：Xcode、Android Studio（Logcat）
- 性能：性能监视器（FPS）、Perfetto / Instruments

**口述模板（30–60 秒）**

1. 一句话结论：RN 的 JS 层用 React Native DevTools 调试，原生层用 Xcode 和 Android Studio。
2. 关键机制：DevTools 支持断点、组件树和性能分析。
3. 对比或注意点：旧的远程调试（在 Chrome 中运行 JS）已经移除。
4. 可追问时补充：原生崩溃要看 Logcat 或 Xcode 的日志。

**常见追问**

- 为什么旧的 Chrome 远程调试被废弃？→ 它把 JS 放在电脑的 Chrome 中执行，与设备上的引擎行为不一致，并且无法使用 JSI 的同步调用。

---

### Q13. RN 中键盘遮挡和安全区怎么处理？

- **难度**: 初级
- **标签**: RN, 布局

**答案**

- **安全区**：`react-native-safe-area-context` 的 `SafeAreaView` 或 `useSafeAreaInsets()` 获取刘海、底部指示条的间距
- **键盘**：`KeyboardAvoidingView`（iOS 常用 `behavior="padding"`）；`react-native-keyboard-controller` 提供更一致的行为；滚动表单中点击外部收起键盘（`keyboardShouldPersistTaps`）
- RN 0.86 起 Android 15+ 默认 edge-to-edge：内容延伸到状态栏和导航栏下方，需要用安全区处理

**口述模板（30–60 秒）**

1. 一句话结论：安全区用 safe-area-context，键盘遮挡用 KeyboardAvoidingView 或 keyboard-controller。
2. 关键机制：useSafeAreaInsets 返回四个方向的安全间距。
3. 对比或注意点：iOS 和 Android 的键盘行为不同，要分别测试。
4. 可追问时补充：Android 15+ 默认 edge-to-edge，更依赖安全区处理。

**常见追问**

- 底部固定按钮被 Home 指示条遮挡怎么办？→ 底部加上 `insets.bottom` 的内边距。

---

### Q14. Hermes 是什么？有什么好处？

- **难度**: 初级
- **标签**: RN, Hermes

**答案**

Hermes 是 Meta 为 RN 开发的 JS 引擎，也是默认引擎：

- **构建时预编译为字节码**：App 启动时不需要解析和编译 JS → 启动更快
- 内存占用更低、包体积更小
- 为移动端优化的垃圾回收
- RN 0.84 起 **Hermes V1** 成为默认版本（更好的性能和现代 JS 支持）

**口述模板（30–60 秒）**

1. 一句话结论：Hermes 是 RN 的默认 JS 引擎，为移动端优化。
2. 关键机制：构建时把 JS 预编译成字节码，启动时直接执行。
3. 对比或注意点：相比 JSC，启动更快、内存更低。
4. 可追问时补充：RN 0.84 起 Hermes V1 是默认版本。

**常见追问**

- Hermes 有 JIT 吗？→ 传统 Hermes 以解释执行字节码为主，侧重启动速度和内存；具体能力以版本说明为准。

---

## 三、小程序基础

### Q15. 微信小程序的目录结构和文件类型？

- **难度**: 初级
- **标签**: 小程序, 基础

**答案**

```text
├── app.js          # 全局逻辑（App()）
├── app.json        # 全局配置：页面列表、窗口、tabBar、分包
├── app.wxss        # 全局样式
├── project.config.json
├── pages/
│   └── index/
│       ├── index.js    # 页面逻辑（Page()）
│       ├── index.json  # 页面配置
│       ├── index.wxml  # 模板（类似 HTML）
│       └── index.wxss  # 样式（类似 CSS，支持 rpx）
└── components/     # 自定义组件（Component()）
```

- **WXML**：模板语言，`{{ }}` 绑定、`wx:if`、`wx:for`
- **WXSS**：CSS 的子集 + rpx 单位
- **WXS**：可以在渲染层运行的脚本（用于过滤和简单交互）

**口述模板（30–60 秒）**

1. 一句话结论：小程序由全局的 app 文件和每个页面的 js、json、wxml、wxss 四个文件组成。
2. 关键机制：app.json 配置页面路由、窗口和分包。
3. 对比或注意点：WXML 不是 HTML，只能使用小程序提供的组件。
4. 可追问时补充：WXS 运行在渲染层，可以减少跨线程通信。

**常见追问**

- 为什么不能直接操作 DOM？→ 逻辑层运行在独立的 JS 线程中，没有 DOM，只能通过 setData 更新视图（见中级）。

---

### Q16. 小程序的生命周期？

- **难度**: 初级
- **标签**: 小程序, 生命周期

**答案**

| 层级 | 生命周期 |
|------|----------|
| App | `onLaunch`（初始化，只执行一次）、`onShow`（启动或从后台进入前台）、`onHide`、`onError` |
| Page | `onLoad(query)`（获取参数）、`onShow`、`onReady`（首次渲染完成）、`onHide`、`onUnload`；以及 `onPullDownRefresh`、`onReachBottom`、`onShareAppMessage` |
| Component | `lifetimes`：`created`、`attached`（可以 setData）、`ready`、`detached`；`pageLifetimes`：所在页面的 show / hide |

**页面切换**：`navigateTo` 时前一个页面 `onHide`；`navigateBack` 时当前页面 `onUnload`。

**口述模板（30–60 秒）**

1. 一句话结论：小程序分为 App、Page、Component 三级生命周期。
2. 关键机制：onLoad 获取页面参数，onReady 表示首次渲染完成。
3. 对比或注意点：组件在 created 中不能 setData，要在 attached 之后。
4. 可追问时补充：页面返回时上一个页面触发 onShow，可以在这里刷新数据。

**常见追问**

- 在哪里请求页面数据？→ 一般在 `onLoad`；需要每次显示都刷新的数据在 `onShow`。

---

### Q17. 小程序怎么更新视图？setData 要注意什么？

- **难度**: 初级
- **标签**: 小程序, setData

**答案**

```js
Page({
  data: { count: 0, list: [] },
  add() {
    this.setData({ count: this.data.count + 1 });
    // 支持路径更新，只传变化的部分
    this.setData({ 'list[0].done': true });
  },
});
```

**注意**：

- 直接修改 `this.data.x` **不会更新视图**，必须调用 `setData`
- setData 会把数据从逻辑层传到渲染层，**数据越大越慢**
- 不要传与渲染无关的数据；不要频繁调用（例如滚动事件中）
- setData 是异步渲染的，可以传第二个参数回调

**口述模板（30–60 秒）**

1. 一句话结论：小程序通过 setData 更新视图，直接修改 data 不会生效。
2. 关键机制：setData 把数据从逻辑层传给渲染层。
3. 对比或注意点：数据量和调用频率直接影响性能。
4. 可追问时补充：用路径语法只更新变化的字段。

**常见追问**

- 为什么 setData 慢？→ 涉及序列化、跨线程传输和渲染层更新（见中级 Q）。

---

### Q18. 小程序的事件和传参？

- **难度**: 初级
- **标签**: 小程序, 事件

**答案**

```xml
<view bindtap="onTap" data-id="{{item.id}}">点击</view>
<view catchtap="onInner">阻止冒泡</view>
```

```js
onTap(e) {
  const { id } = e.currentTarget.dataset; // data-* 属性
}
```

- `bind`：冒泡；`catch`：阻止冒泡；`capture-bind`：捕获阶段；`mut-bind`：互斥
- 不能像 Vue / React 那样直接传参，通过 `data-*` 传递
- `currentTarget` 是绑定事件的元素，`target` 是触发事件的元素

**口述模板（30–60 秒）**

1. 一句话结论：小程序用 bind 和 catch 绑定事件，通过 data-* 传参。
2. 关键机制：catch 会阻止事件冒泡。
3. 对比或注意点：获取参数用 currentTarget.dataset，而不是 target。
4. 可追问时补充：Taro 和 uni-app 中可以像框架一样直接传参。

**常见追问**

- 为什么取不到 dataset？→ 点击到了子元素，用了 `target` 而不是 `currentTarget`。

---

### Q19. 小程序的自定义组件怎么通信？

- **难度**: 初级
- **标签**: 小程序, 组件

**答案**

```js
// 子组件
Component({
  properties: { title: { type: String, value: '' } },
  methods: {
    onClick() {
      this.triggerEvent('change', { value: 1 });
    },
  },
});
```

```xml
<!-- 父页面 -->
<my-comp title="标题" bind:change="onChange" />
```

| 方式 | 说明 |
|------|------|
| properties | 父 → 子 |
| `triggerEvent` | 子 → 父 |
| `this.selectComponent('#id')` | 父直接调用子组件的方法 |
| behaviors | 代码复用（类似 mixin） |
| 全局状态 | `getApp().globalData`、状态管理库（mobx-miniprogram） |

**口述模板（30–60 秒）**

1. 一句话结论：父传子用 properties，子传父用 triggerEvent。
2. 关键机制：父组件也可以通过 selectComponent 获取子组件实例。
3. 对比或注意点：跨页面共享状态用全局数据或状态管理库。
4. 可追问时补充：behaviors 用于组件之间复用代码。

**常见追问**

- 页面之间怎么传数据？→ URL 参数、`EventChannel`（navigateTo 返回）、全局状态。

---

### Q20. 小程序的路由和页面栈？

- **难度**: 初级
- **标签**: 小程序, 路由

**答案**

| API | 行为 |
|-----|------|
| `wx.navigateTo` | 保留当前页，打开新页（**页面栈最多 10 层**） |
| `wx.redirectTo` | 关闭当前页，打开新页 |
| `wx.navigateBack` | 返回上 N 页 |
| `wx.switchTab` | 跳转 tabBar 页面，关闭其他非 tab 页 |
| `wx.reLaunch` | 关闭所有页面，打开新页 |

**注意**：页面栈满了 `navigateTo` 会失败；tabBar 页面只能用 `switchTab`；`getCurrentPages()` 获取页面栈。

**口述模板（30–60 秒）**

1. 一句话结论：小程序有 navigateTo、redirectTo、navigateBack、switchTab、reLaunch 五种跳转。
2. 关键机制：navigateTo 会入栈，页面栈最多 10 层。
3. 对比或注意点：tabBar 页面只能用 switchTab 跳转。
4. 可追问时补充：栈满时可以改用 redirectTo，或者封装路由自动处理。

**常见追问**

- 页面栈超过 10 层怎么办？→ 封装跳转方法：栈深度接近上限时改用 `redirectTo`。

---

### Q21. 小程序的登录流程？

- **难度**: 初级
- **标签**: 小程序, 登录

**答案**

```text
1. 小程序：wx.login() 获取临时 code
2. 小程序 → 自己的服务端：发送 code
3. 服务端 → 微信接口（code2Session）：用 appid + secret + code 换取 openid、session_key
4. 服务端：创建自己的登录态（token），返回给小程序
5. 小程序：保存 token，后续请求携带
```

**注意**：

- `session_key` 和 `secret` **只能在服务端**，不能下发给前端
- 获取手机号需要用户主动点击按钮授权（`getPhoneNumber`），服务端解密或换取
- 用户信息接口已经收紧，头像昵称需要用户主动填写

**口述模板（30–60 秒）**

1. 一句话结论：小程序用 wx.login 获取 code，服务端用 code 换 openid 并创建自己的登录态。
2. 关键机制：服务端调用 code2Session 获取 openid 和 session_key。
3. 对比或注意点：session_key 和 appsecret 只能保存在服务端。
4. 可追问时补充：手机号需要用户主动点击按钮授权。

**常见追问**

- 登录态过期怎么处理？→ 请求返回 401 时重新执行登录流程并重试；多个并发请求共享同一次登录（见手写题）。

---

### Q22. 小程序的网络请求有什么限制？

- **难度**: 初级
- **标签**: 小程序, 网络

**答案**

- 使用 `wx.request`（不是 fetch / XHR）
- **域名必须在后台配置**（request 合法域名），且必须是 HTTPS
- 不存在浏览器的跨域问题（不在浏览器中），但受域名白名单约束
- 并发请求数有限制（以官方为准）
- 上传下载用 `wx.uploadFile`、`wx.downloadFile`；实时通信用 `wx.connectSocket`

**口述模板（30–60 秒）**

1. 一句话结论：小程序用 wx.request 请求，域名必须配置在白名单中且是 HTTPS。
2. 关键机制：小程序不在浏览器中运行，没有 CORS 问题。
3. 对比或注意点：开发时可以关闭域名校验，但上线必须配置。
4. 可追问时补充：一般会封装请求，统一处理登录态和错误。

**常见追问**

- 怎么封装请求？→ Promise 化、自动带 token、401 时刷新登录、统一错误提示（见手写题）。

---

### Q23. 小程序的分包是什么？为什么需要？

- **难度**: 初级
- **标签**: 小程序, 分包

**答案**

小程序的代码包有体积限制（主包和单个分包各有上限，总包也有上限，以官方为准）。**分包**把页面按功能拆到不同的包中：

```json
{
  "pages": ["pages/index/index"],
  "subPackages": [
    { "root": "packageOrder", "pages": ["list/index", "detail/index"] }
  ],
  "preloadRule": {
    "pages/index/index": { "network": "all", "packages": ["packageOrder"] }
  }
}
```

- 启动时只下载**主包**，进入分包页面时再下载
- **独立分包**：不依赖主包，可以单独启动（适合活动页）
- **分包预下载**：进入某个页面时提前下载可能访问的分包

**口述模板（30–60 秒）**

1. 一句话结论：分包把代码拆成主包和多个分包，启动时只下载主包。
2. 关键机制：进入分包页面时才下载分包，可以配置预下载。
3. 对比或注意点：独立分包可以不依赖主包单独运行。
4. 可追问时补充：分包能突破体积限制，也能加快启动。

**常见追问**

- 分包能引用主包的代码吗？→ 可以；但主包不能引用分包的代码；独立分包不能引用主包。

---

### Q24. 小程序和 H5 有什么区别？

- **难度**: 初级
- **标签**: 小程序, 对比

**答案**

| | 小程序 | H5 |
|--|--------|-----|
| 运行环境 | 宿主 App（微信）的双线程环境 | 浏览器 / WebView |
| DOM | 不能直接操作 | 可以 |
| 能力 | 宿主提供的丰富 API（支付、登录、定位） | 受浏览器限制，需要 JSSDK |
| 发布 | 需要审核 | 随时发布 |
| 加载 | 代码包下载到本地，启动快、有缓存 | 每次从网络加载（有缓存） |
| 入口 | 宿主内的搜索、分享、扫码 | URL |

**口述模板（30–60 秒）**

1. 一句话结论：小程序运行在宿主 App 的双线程环境中，H5 运行在浏览器中。
2. 关键机制：小程序不能操作 DOM，但可以使用宿主提供的能力。
3. 对比或注意点：小程序发布要审核，H5 可以随时发布。
4. 可追问时补充：小程序的代码包下载到本地，启动体验更好。

**常见追问**

- 小程序为什么要审核？→ 平台需要管控内容和能力的使用，这也是它不允许直接操作 DOM 的原因之一。

---

### Q25. rpx 是什么？小程序怎么适配屏幕？

- **难度**: 初级
- **标签**: 小程序, 适配

**答案**

- **rpx**：响应式像素，规定屏幕宽度为 **750rpx**，按屏幕宽度等比缩放
- 在 375 宽的 iPhone 上，`1rpx = 0.5px`
- 设计稿按 750 宽出图时，标注值直接作为 rpx 使用
- 字体也可以用 rpx，但大屏（平板）上会过大，需要限制
- 安全区：`env(safe-area-inset-bottom)`

**口述模板（30–60 秒）**

1. 一句话结论：rpx 把屏幕宽度定义为 750，按屏幕宽度等比缩放。
2. 关键机制：750 宽的设计稿可以直接使用标注值。
3. 对比或注意点：大屏设备上 rpx 会把内容放得过大。
4. 可追问时补充：底部安全区用 env(safe-area-inset-bottom)。

**常见追问**

- 1rpx 的边框在某些机型上不显示？→ 换算后小于 1 物理像素被舍去；细线用 1px 或 transform 缩放。

---

## 四、跨端框架与混合开发

### Q26. Taro 是什么？

- **难度**: 初级
- **标签**: Taro

**答案**

Taro 是京东开源的多端框架：用 **React 或 Vue** 编写，编译到微信 / 支付宝 / 抖音等小程序、H5、React Native、鸿蒙。

```tsx
import { View, Text } from '@tarojs/components';
import Taro, { useLoad } from '@tarojs/taro';

export default function Index() {
  useLoad(() => console.log('页面加载'));
  return (
    <View onClick={() => Taro.navigateTo({ url: '/pages/detail/index?id=1' })}>
      <Text>Hello</Text>
    </View>
  );
}
```

- 使用 Taro 的组件（`View`、`Text`）和 API（`Taro.xxx`，抹平各平台差异）
- Taro 3 起是**运行时方案**：不限制 React 语法（见中级）

**口述模板（30–60 秒）**

1. 一句话结论：Taro 让开发者用 React 或 Vue 编写一套代码，运行在多个小程序、H5 和 RN 上。
2. 关键机制：使用 Taro 统一的组件和 API，抹平平台差异。
3. 对比或注意点：Taro 3 是运行时方案，几乎不限制框架语法。
4. 可追问时补充：平台特有的逻辑用 `process.env.TARO_ENV` 区分。

**常见追问**

- Taro 的性能比原生小程序差吗？→ 运行时方案有额外开销，Taro 通过 diff 后的最小 setData、预渲染、半编译等手段优化。

---

### Q27. uni-app 是什么？条件编译怎么用？

- **难度**: 初级
- **标签**: uni-app

**答案**

uni-app 是 DCloud 的多端框架：用 **Vue** 编写，发布到小程序、H5、App（App 端使用 WebView 渲染或 nvue 原生渲染）、鸿蒙。uni-app x 用 UTS 语言编译为原生代码。

**条件编译**（编译时按平台保留代码）：

```vue
<!-- #ifdef MP-WEIXIN -->
<button open-type="getPhoneNumber">微信获取手机号</button>
<!-- #endif -->
<!-- #ifdef H5 -->
<button @click="smsLogin">短信登录</button>
<!-- #endif -->
```

```js
// #ifdef APP-PLUS
plus.runtime.getProperty(/* ... */);
// #endif
```

**口述模板（30–60 秒）**

1. 一句话结论：uni-app 用 Vue 编写，发布到小程序、H5 和 App。
2. 关键机制：条件编译在编译时按平台保留或删除代码。
3. 对比或注意点：条件编译比运行时判断更好，不会把其他平台的代码打进包里。
4. 可追问时补充：uni-app x 用 UTS 编译为原生代码。

**常见追问**

- Taro 和 uni-app 怎么选？→ 团队技术栈（React → Taro；Vue → 两者都可）；是否需要 App 端；生态和插件市场。

---

### Q28. H5 和原生 App 怎么混合开发？

- **难度**: 初级
- **标签**: Hybrid, WebView

**答案**

原生 App 中通过 **WebView** 加载 H5 页面：

- iOS：`WKWebView`；Android：系统 WebView（基于 Chromium）
- H5 通过 **JSBridge** 调用原生能力（相机、支付、分享、获取用户信息）
- 原生负责导航栏、容器管理；H5 负责业务页面
- 适合活动页、频繁变化的业务页面

**常见问题**：白屏时间长（WebView 初始化 + 网络加载）、与原生体验不一致、调试不便。

**口述模板（30–60 秒）**

1. 一句话结论：混合开发是在原生 App 的 WebView 中加载 H5 页面。
2. 关键机制：H5 通过 JSBridge 调用原生能力。
3. 对比或注意点：主要问题是首屏白屏和体验不一致。
4. 可追问时补充：用离线包、WebView 预加载、数据预取优化首屏。

**常见追问**

- 怎么调试 App 中的 H5？→ Android 用 `chrome://inspect`（需要开启 WebView 调试）；iOS 用 Safari 的 Web 检查器；或者 vConsole。

---

### Q29. JSBridge 是什么？

- **难度**: 初级
- **标签**: JSBridge

**答案**

JSBridge 是 **WebView 中的 JS 与原生代码双向通信**的机制。

**JS 调用原生**：

- **注入对象**：原生向 WebView 注入对象（Android `addJavascriptInterface`、iOS `WKScriptMessageHandler` 的 `window.webkit.messageHandlers.xxx.postMessage`）
- **URL 拦截**：JS 发起 `jsbridge://method?params` 的请求，原生拦截解析（老方案）

**原生调用 JS**：执行一段 JS（Android `evaluateJavascript`、iOS `evaluateJavaScript`），调用页面上的全局函数。

**回调**：JS 调用时生成回调 ID，原生处理完后用这个 ID 调用 JS 回调（见手写题）。

**口述模板（30–60 秒）**

1. 一句话结论：JSBridge 是 WebView 中 JS 和原生的双向通信机制。
2. 关键机制：JS 通过注入的对象或 URL 拦截调用原生，原生通过执行 JS 回调页面。
3. 对比或注意点：调用是异步的，通过回调 ID 对应请求和结果。
4. 可追问时补充：要限制可以调用 Bridge 的域名，防止恶意页面调用原生能力。

**常见追问**

- 为什么要限制域名？→ 如果任意页面都能调用 Bridge，被注入或跳转到的恶意页面就能获取用户信息或执行敏感操作。

---

### Q30. 鸿蒙对前端跨端有什么影响？

- **难度**: 初级
- **标签**: 鸿蒙

**答案**

- **HarmonyOS NEXT** 不再兼容 Android 应用（APK），应用需要用 ArkTS / ArkUI 开发或者通过跨端框架适配
- **ArkTS**：基于 TypeScript 扩展的语言；**ArkUI**：声明式 UI 框架（类似 SwiftUI / Flutter 的写法）
- **跨端框架的适配**：React Native 鸿蒙版（社区和厂商维护）、Taro、uni-app / uni-app x、Flutter 鸿蒙版
- 微信小程序在鸿蒙版微信中也可以运行

**口述模板（30–60 秒）**

1. 一句话结论：鸿蒙 NEXT 不兼容 Android 应用，需要原生开发或跨端框架适配。
2. 关键机制：原生开发使用 ArkTS 和声明式的 ArkUI。
3. 对比或注意点：RN、Taro、uni-app、Flutter 都有鸿蒙适配方案。
4. 可追问时补充：已有的跨端代码可以通过这些框架复用到鸿蒙。

**常见追问**

- 前端学习 ArkTS 难吗？→ 语法接近 TypeScript，声明式 UI 的思路与 React / Vue 相近；主要是熟悉 ArkUI 组件和状态管理装饰器。
