# HTML 初级 Q&A（能用）

> 目标：标签和属性用对，知道浏览器默认行为。初级题也要答出「为什么」。

---

## 一、文档基础

### Q1. HTML 是什么？它和 CSS、JavaScript 的职责怎么区分？

- **难度**: junior
- **标签**: 概念

**答案**

HTML（HyperText Markup Language）描述页面的**结构和语义**；CSS 负责**表现**；JavaScript 负责**行为和状态变化**。

职责分离不是教条：例如 `<button>` 本身就带有可点击、可聚焦、键盘触发等行为。好的 HTML 能减少大量 JS 和 ARIA 补丁。

**口述模板（30–60 秒）**

1. 一句话结论：HTML 定义内容是什么，CSS 决定长什么样，JS 决定怎么变化。
2. 关键机制：浏览器把 HTML 解析成 DOM，再结合 CSSOM 渲染。
3. 注意点：原生 HTML 自带很多行为和语义。
4. 可追问时补充：语义 HTML 对 a11y 和 SEO 有直接影响。

**常见追问**

- `div` 能做所有事吗？→ 视觉上能，但会丢掉默认语义和交互，后续要自己补。

---

### Q2. `<!DOCTYPE html>` 有什么作用？

- **难度**: junior
- **标签**: DOCTYPE, 渲染模式

**答案**

它让浏览器使用**标准模式（Standards Mode）**。缺失或写错 DOCTYPE 时，浏览器可能进入**怪异模式（Quirks Mode）**，盒模型、行高、表格等布局行为会模拟老浏览器，导致样式表现异常。

`<!DOCTYPE html>` 不是 HTML 标签，也不区分大小写。

**口述模板（30–60 秒）**

1. 一句话结论：DOCTYPE 决定浏览器使用标准模式。
2. 关键机制：浏览器解析开头的 DOCTYPE 来选择渲染模式。
3. 注意点：缺失会进入怪异模式，导致 CSS 行为不一致。
4. 可追问时补充：可用 `document.compatMode` 检查，`CSS1Compat` 表示标准模式。

**常见追问**

- `BackCompat` 是什么？→ 表示怪异模式。

---

### Q3. 一个标准 HTML 页面最基本的结构是什么？

- **难度**: junior
- **标签**: 文档结构

**答案**

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>页面标题</title>
  </head>
  <body>
    <main>
      <h1>主标题</h1>
    </main>
  </body>
</html>
```

`lang` 影响读屏发音、翻译、断词等；`charset` 应尽早出现在文档开头。

**口述模板（30–60 秒）**

1. 一句话结论：DOCTYPE、html、head、body 是骨架。
2. 关键机制：head 放元信息，body 放用户可见内容。
3. 注意点：要写 `lang`、`charset`、`viewport` 和有意义的 title。
4. 可追问时补充：每页通常只有一个 `main`。

**常见追问**

- `lang` 写错会怎样？→ 读屏可能按错误语言发音，翻译和搜索理解也会受影响。

---

### Q4. `<meta charset="utf-8">` 为什么要放在前面？

- **难度**: junior
- **标签**: head, 编码

**答案**

浏览器需要知道字节如何转换为字符。字符编码声明应尽早出现，规范要求通过 meta 声明时位于文档前 1024 字节内。放太后可能导致前面内容按错误编码解析，出现乱码甚至需要重新解析。

服务端更推荐通过响应头声明：

```http
Content-Type: text/html; charset=utf-8
```

**口述模板（30–60 秒）**

1. 一句话结论：先确定编码，后续字符才能正确解析。
2. 关键机制：HTML 下载的是字节，解析前需要解码。
3. 注意点：meta charset 要尽早出现。
4. 可追问时补充：HTTP 响应头比 meta 更早生效。

**常见追问**

- 为什么推荐 UTF-8？→ 覆盖多语言、生态统一、减少编码转换问题。

---

### Q5. viewport meta 是做什么的？

- **难度**: junior
- **标签**: 移动端, head

**答案**

```html
<meta name="viewport" content="width=device-width, initial-scale=1" />
```

它告诉移动浏览器布局视口宽度使用设备宽度、初始缩放为 1。缺少时，移动浏览器可能用较宽的虚拟视口渲染页面，再整体缩小，导致文字很小。

不建议随意写 `user-scalable=no` 或 `maximum-scale=1`，会影响视力障碍用户缩放。

**口述模板（30–60 秒）**

1. 一句话结论：viewport 控制移动端布局视口。
2. 关键机制：`width=device-width` 让 CSS 断点按设备宽度工作。
3. 注意点：不要禁止缩放。
4. 可追问时补充：响应式布局依赖它。

**常见追问**

- 为什么媒体查询在手机上不生效？→ 常见原因之一是没有 viewport meta。

---

### Q6. `head` 里常见的元信息有哪些？

- **难度**: junior
- **标签**: head, SEO

**答案**

常见内容：

- `title`：浏览器标签页、搜索结果标题、收藏夹名称
- `meta name="description"`：搜索结果摘要候选
- `link rel="canonical"`：声明规范 URL，减少重复内容
- `meta name="robots"`：控制索引和跟踪链接
- `link rel="icon"`：站点图标
- `meta property="og:*"`：社交分享卡片
- 样式表、预加载、预连接等资源声明

`meta keywords` 对主流搜索引擎价值很低，不要把它当 SEO 重点。

**口述模板（30–60 秒）**

1. 一句话结论：head 告诉浏览器、搜索引擎和分享平台页面元信息。
2. 关键机制：title、description、canonical 和 robots 最常用。
3. 注意点：每个页面 title 和 description 应该有区分度。
4. 可追问时补充：资源提示也放在 head 中。

**常见追问**

- description 一定会显示在搜索结果吗？→ 不一定，搜索引擎可能重写摘要。

---

## 二、常用元素与语义

### Q7. 什么是 HTML 语义化？

- **难度**: junior
- **标签**: 语义化

**答案**

语义化是根据内容的含义选择合适元素，例如导航用 `nav`、主要内容用 `main`、文章用 `article`、按钮用 `button`。

收益：

- 读屏器能识别页面区域和控件
- 搜索引擎更容易理解结构
- 浏览器提供默认行为，如按钮可聚焦和键盘触发
- 代码可读性更好

**口述模板（30–60 秒）**

1. 一句话结论：语义化是让元素表达内容含义。
2. 关键机制：浏览器把语义映射到无障碍树，并提供默认行为。
3. 注意点：不是为了标签名好看，而是为了机器和人都能理解。
4. 可追问时补充：语义化和 ARIA 的优先级。

**常见追问**

- 语义标签会自动提升 SEO 排名吗？→ 不会直接保证排名，但能帮助搜索引擎理解内容。

---

### Q8. HTML5 常见的结构语义元素有哪些？

- **难度**: junior
- **标签**: landmark

**答案**

| 元素 | 用途 |
|------|------|
| `header` | 页面或区块的头部 |
| `nav` | 主要导航链接集合 |
| `main` | 当前页面的主要内容 |
| `article` | 可独立分发的内容，如文章、帖子、商品卡片 |
| `section` | 有主题的内容分组，通常带标题 |
| `aside` | 与主内容相关但可分离的补充内容 |
| `footer` | 页面或区块的页脚 |
| `search` | 搜索或过滤区域 |

**口述模板（30–60 秒）**

1. 一句话结论：这些元素描述页面区域。
2. 关键机制：读屏用户可以按 landmark 快速跳转。
3. 注意点：不要为了语义把所有 `div` 都替换掉。
4. 可追问时补充：`article` 和 `section` 的区别。

**常见追问**

- `header` 只能出现一次吗？→ 不是；页面和多个 article 都可以有自己的 header。

---

### Q9. `div` 和 `span` 有什么区别？

- **难度**: junior
- **标签**: 元素

**答案**

二者都没有特定语义。`div` 是通用流内容容器，默认 `display: block`；`span` 是通用短语内容容器，默认 `display: inline`。

「块级 / 行内」主要是 CSS 显示层面的描述；HTML 规范更关注内容模型，例如某个元素允许包含哪些子元素。

**口述模板（30–60 秒）**

1. 一句话结论：div 和 span 都是无语义容器。
2. 关键机制：默认 display 不同。
3. 注意点：有合适语义元素时优先语义元素。
4. 可追问时补充：内容模型比块级 / 行内更准确。

**常见追问**

- `a` 里面能放 `div` 吗？→ 现代 HTML 中 `a` 可以是透明内容模型，在合法上下文中可包裹块内容，但不能嵌套交互内容。

---

### Q10. 标题 `h1`–`h6` 应该怎么用？

- **难度**: junior
- **标签**: 标题, a11y

**答案**

标题表达内容层级，不是用来控制字号。应从主标题开始，按结构依次往下，避免为了样式跳级。读屏用户常用标题导航快速浏览页面。

视觉字号由 CSS 控制：

```html
<h2 class="title-lg">账户设置</h2>
```

**口述模板（30–60 秒）**

1. 一句话结论：标题表达层级，不表达字号。
2. 关键机制：辅助技术会生成标题列表。
3. 注意点：避免跳级和用 `div` 冒充标题。
4. 可追问时补充：一个页面能否多个 h1 取决于结构，但通常保留清晰主标题。

**常见追问**

- 一个页面能有多个 `h1` 吗？→ 技术上可以，但多数业务页面保持一个页面主标题更清晰。

---

### Q11. `strong` / `b`、`em` / `i` 有什么区别？

- **难度**: junior
- **标签**: 文本语义

**答案**

- `strong`：内容很重要、严重或紧急
- `b`：仅需要视觉上区分，例如产品名、关键词，没有额外重要性
- `em`：强调语气，可能改变句意
- `i`：另一种语气或特殊术语，例如外文词、思想内容

不要只因为想加粗或斜体而选择元素，样式用 CSS。

**口述模板（30–60 秒）**

1. 一句话结论：strong/em 表达语义，b/i 更偏文本类别。
2. 关键机制：语义元素可能影响辅助技术和内容理解。
3. 注意点：样式用 CSS 控制。
4. 可追问时补充：不同读屏器对强调的播报不一定一致。

**常见追问**

- 读屏一定会读出 strong 吗？→ 不一定，所以重要提示不能只依赖粗体。

---

### Q12. 列表 `ul`、`ol`、`dl` 怎么选？

- **难度**: junior
- **标签**: 列表

**答案**

- `ul`：无序列表，顺序不影响含义
- `ol`：有序列表，顺序有意义，如步骤、排名
- `dl`：描述列表，由 `dt` 术语和 `dd` 描述组成，适合键值信息

```html
<dl>
  <dt>订单号</dt>
  <dd>20261009001</dd>
</dl>
```

**口述模板（30–60 秒）**

1. 一句话结论：按「顺序是否有意义」和「是否键值」选择列表。
2. 关键机制：读屏会播报列表项数量和位置。
3. 注意点：不要用多个 div 模拟列表。
4. 可追问时补充：CSS 去掉 list-style 可能影响部分 Safari 的列表语义。

**常见追问**

- `ol` 能倒序吗？→ 可以使用 `reversed`、`start` 等属性。

---

### Q13. 什么时候用 `table`？

- **难度**: junior
- **标签**: 表格

**答案**

只有展示二维数据时用 `table`，例如订单、成绩、价格对比。不要用表格布局页面。

可访问的表格至少应有：

```html
<table>
  <caption>订单列表</caption>
  <thead>
    <tr><th scope="col">订单号</th><th scope="col">金额</th></tr>
  </thead>
  <tbody>
    <tr><td>A001</td><td>99</td></tr>
  </tbody>
</table>
```

**口述模板（30–60 秒）**

1. 一句话结论：table 用于数据，不用于布局。
2. 关键机制：th、scope、caption 帮助辅助技术建立表头关系。
3. 注意点：复杂表头要额外设计关联。
4. 可追问时补充：移动端表格可以横向滚动或改卡片展示，但要保持信息关系。

**常见追问**

- `thead` 必须写吗？→ 不总是必须，但有表头时建议写清楚。

---

## 三、链接、图片与媒体

### Q14. `a` 标签有哪些常用属性？

- **难度**: junior
- **标签**: 链接

**答案**

- `href`：目标地址
- `target="_blank"`：新窗口 / 标签页打开
- `rel`：说明关系，如 `noopener`、`noreferrer`、`nofollow`、`ugc`、`sponsored`
- `download`：提示下载
- `hreflang`：目标资源语言

没有 `href` 的 `a` 不是可聚焦链接；需要执行动作时应使用 `button`。

**口述模板（30–60 秒）**

1. 一句话结论：a 负责导航和资源链接。
2. 关键机制：href 决定链接目标，rel 描述关系。
3. 注意点：不要用 `href="#"` 加 click 做按钮。
4. 可追问时补充：外链安全与 SEO 的 rel。

**常见追问**

- `download` 跨域文件一定生效吗？→ 不一定，跨域通常受限制。

---

### Q15. 为什么外链 `target="_blank"` 常加 `rel="noopener noreferrer"`？

- **难度**: junior
- **标签**: 安全, 链接

**答案**

旧浏览器中，新页面可通过 `window.opener` 修改原页面地址，形成反向标签页钓鱼（Reverse Tabnabbing）。`noopener` 切断 opener；`noreferrer` 同时不发送 Referer。

现代浏览器对 `target="_blank"` 默认采用 noopener 行为，但团队规范里显式写清仍能提高可读性和兼容性。

**口述模板（30–60 秒）**

1. 一句话结论：noopener 防止新页面控制原页面。
2. 关键机制：切断 `window.opener`。
3. 注意点：noreferrer 还会影响来源统计。
4. 可追问时补充：现代浏览器已有默认保护。

**常见追问**

- 所有外链都要 noreferrer 吗？→ 看隐私和统计需求；安全上 noopener 更关键。

---

### Q16. `img` 的 `alt` 应该怎么写？

- **难度**: junior
- **标签**: 图片, a11y

**答案**

`alt` 是图片的替代文本。图片加载失败、读屏用户、搜索引擎都会依赖它。

- 信息图片：描述图片传达的信息
- 链接图片：描述链接目的
- 装饰图片：`alt=""`
- 复杂图表：简短 alt + 页面中补充详细说明

不要写「图片」「icon」这类空洞内容。

**口述模板（30–60 秒）**

1. 一句话结论：alt 写图片的作用，不是文件名。
2. 关键机制：图片不可见时 alt 替代其信息。
3. 注意点：装饰图用空 alt，避免读屏噪音。
4. 可追问时补充：链接图片的 alt 应描述目标。

**常见追问**

- 可以不写 alt 吗？→ 对普通 img 应写；空字符串和缺失属性含义不同。

---

### Q17. 如何避免图片导致布局抖动？

- **难度**: junior
- **标签**: 图片, CLS

**答案**

给图片设置 `width` 和 `height`，浏览器可以在图片加载前计算宽高比并预留空间：

```html
<img src="cover.jpg" width="1200" height="630" alt="活动封面" />
```

CSS 可再设置 `max-width: 100%; height: auto;`。

**口述模板（30–60 秒）**

1. 一句话结论：提前提供尺寸，避免加载后撑开页面。
2. 关键机制：浏览器根据宽高计算 aspect ratio。
3. 注意点：响应式时仍然推荐写原始尺寸。
4. 可追问时补充：广告和动态内容也要预留空间。

**常见追问**

- 用 CSS `aspect-ratio` 可以吗？→ 可以，但 img 原生 width/height 更早被解析，通常更直接。

---

### Q18. `loading="lazy"` 是做什么的？

- **难度**: junior
- **标签**: 性能, 图片

**答案**

告诉浏览器延迟加载暂时不在视口附近的图片或 iframe，减少首屏网络与解码压力。

```html
<img src="list.jpg" loading="lazy" width="400" height="300" alt="商品图" />
```

首屏 LCP 图片不要懒加载。

**口述模板（30–60 秒）**

1. 一句话结论：原生懒加载非首屏图片和 iframe。
2. 关键机制：浏览器根据距离视口的阈值决定何时加载。
3. 注意点：不要懒加载首屏关键图。
4. 可追问时补充：配合尺寸避免 CLS。

**常见追问**

- 还需要 IntersectionObserver 吗？→ 普通图片多数不用；复杂预加载策略仍可用。

---

### Q19. `audio` 和 `video` 常用属性有哪些？

- **难度**: junior
- **标签**: 媒体

**答案**

常用：`controls`、`autoplay`、`muted`、`loop`、`preload`、`poster`、`playsinline`。

```html
<video controls playsinline poster="poster.jpg">
  <source src="demo.webm" type="video/webm" />
  <source src="demo.mp4" type="video/mp4" />
  <track kind="captions" src="zh.vtt" srclang="zh" label="中文" />
</video>
```

浏览器通常限制有声自动播放；字幕 `track` 是无障碍要点。

**口述模板（30–60 秒）**

1. 一句话结论：媒体元素自带播放控制和多格式回退。
2. 关键机制：浏览器选择第一个能播放的 source。
3. 注意点：自动播放常需 muted，视频应提供字幕。
4. 可追问时补充：preload 影响带宽。

**常见追问**

- `preload="none"` 有什么用？→ 用户可能不播放时节省流量。

---

## 四、表单基础

### Q20. `label` 的作用是什么？

- **难度**: junior
- **标签**: 表单, a11y

**答案**

`label` 为表单控件提供可访问名称，并扩大点击区域。

```html
<label for="email">邮箱</label>
<input id="email" name="email" type="email" />
```

也可以包裹：

```html
<label>邮箱 <input name="email" type="email" /></label>
```

`placeholder` 不能代替 label。

**口述模板（30–60 秒）**

1. 一句话结论：label 给控件命名，并让文字可点击。
2. 关键机制：通过 `for` 关联 id 或直接包裹。
3. 注意点：placeholder 不是标签。
4. 可追问时补充：可访问名称计算。

**常见追问**

- 视觉上不想显示 label 怎么办？→ 用视觉隐藏类，而不是删掉 label。

---

### Q21. 常见 `input type` 有哪些？为什么不要全写成 `text`？

- **难度**: junior
- **标签**: 表单

**答案**

常见：`email`、`tel`、`url`、`number`、`password`、`search`、`date`、`checkbox`、`radio`、`file`。

不同 type 会影响：

- 移动端键盘
- 原生校验
- 自动填充
- 浏览器控件与辅助技术

注意：手机号、银行卡号这类「由数字组成但不参与计算」的值通常不建议用 `number`，可用 `text` + `inputmode="numeric"`。

**口述模板（30–60 秒）**

1. 一句话结论：type 决定输入语义和默认体验。
2. 关键机制：浏览器据此提供键盘、校验和控件。
3. 注意点：不是所有数字都适合 number。
4. 可追问时补充：autocomplete 和 inputmode。

**常见追问**

- `number` 的坑？→ 可能出现滚轮修改、科学计数、前导零丢失等问题。

---

### Q22. `name`、`id`、`value` 在表单中的区别？

- **难度**: junior
- **标签**: 表单

**答案**

- `id`：文档内唯一，用于 label、锚点、CSS/JS 选择
- `name`：表单提交时的字段名；单选组靠相同 name 分组
- `value`：提交值或控件当前值

没有 `name` 的控件通常不会进入表单提交数据。

**口述模板（30–60 秒）**

1. 一句话结论：id 管引用，name 管提交，value 管值。
2. 关键机制：浏览器按 name/value 生成表单数据。
3. 注意点：缺 name 会导致提交漏字段。
4. 可追问时补充：radio 用相同 name 分组。

**常见追问**

- `disabled` 字段会提交吗？→ 不会；`readonly` 字段会提交。

---

### Q23. `button` 在表单中默认是什么类型？

- **难度**: junior
- **标签**: 表单, button

**答案**

表单内 `<button>` 默认 `type="submit"`。如果只是打开弹窗、切换密码可见等操作，应显式写：

```html
<button type="button">显示密码</button>
```

否则可能意外提交表单。

**口述模板（30–60 秒）**

1. 一句话结论：表单里的 button 默认提交。
2. 关键机制：点击 submit button 会触发表单提交流程。
3. 注意点：普通操作按钮写 `type="button"`。
4. 可追问时补充：`reset` 类型很少推荐使用。

**常见追问**

- 回车为什么会提交表单？→ 浏览器有隐式提交行为，通常依赖可用的提交按钮。

---

### Q24. 原生表单校验有哪些属性？

- **难度**: junior
- **标签**: 表单校验

**答案**

常用：`required`、`minlength`、`maxlength`、`min`、`max`、`step`、`pattern`，以及 `type="email"` 等内置格式。

```html
<input name="username" required minlength="3" maxlength="20" />
```

前端校验只是体验优化，后端必须再次校验。

**口述模板（30–60 秒）**

1. 一句话结论：HTML 提供声明式校验。
2. 关键机制：提交前浏览器执行约束校验。
3. 注意点：前端校验不能替代后端安全校验。
4. 可追问时补充：Constraint Validation API 可自定义提示。

**常见追问**

- 如何关闭浏览器默认校验？→ form 上加 `novalidate`，但仍应自行实现可访问提示。

---

### Q25. `checkbox` 和 `radio` 怎么正确分组？

- **难度**: junior
- **标签**: 表单

**答案**

- `radio`：多选一，同组使用相同 `name`
- `checkbox`：可多选，可共享 name 提交多个值

一组选项建议使用 `fieldset` + `legend`：

```html
<fieldset>
  <legend>配送方式</legend>
  <label><input type="radio" name="ship" value="express" /> 快递</label>
  <label><input type="radio" name="ship" value="pickup" /> 自提</label>
</fieldset>
```

**口述模板（30–60 秒）**

1. 一句话结论：radio 单选，checkbox 多选。
2. 关键机制：name 决定分组和提交字段。
3. 注意点：用 fieldset/legend 说明整组问题。
4. 可追问时补充：键盘方向键在 radio 组中移动。

**常见追问**

- 自定义 radio 样式会丢语义吗？→ 隐藏原生 input 的方式不当会丢，正确做法是保留原生控件可访问。

---

## 五、全局属性与其他

### Q26. `data-*` 属性有什么用？

- **难度**: junior
- **标签**: 属性

**答案**

`data-*` 用于在 HTML 上存放和页面脚本相关的自定义数据：

```html
<button data-product-id="123">加入购物车</button>
```

JS 通过 `element.dataset.productId` 读取。不要存放敏感信息，也不要替代真正的数据状态管理。

**口述模板（30–60 秒）**

1. 一句话结论：data 属性存放自定义元数据。
2. 关键机制：dataset 会把短横线转成驼峰。
3. 注意点：用户可见、可修改，不能放敏感数据。
4. 可追问时补充：可以用于事件委托定位业务 ID。

**常见追问**

- 可以用 data 属性写 CSS 选择器吗？→ 可以，但样式依赖业务数据要谨慎。

---

### Q27. `hidden`、`display: none`、`visibility: hidden` 的区别？

- **难度**: junior
- **标签**: 显示隐藏

**答案**

- `hidden`：HTML 属性，表示当前不相关，浏览器默认按 `display: none` 处理
- `display: none`：不渲染、不占位，通常也从无障碍树中隐藏
- `visibility: hidden`：不可见但占位，通常也不可交互

注意：CSS 可能覆盖 `hidden` 的默认显示效果。

**口述模板（30–60 秒）**

1. 一句话结论：hidden 是语义状态，display/visibility 是样式。
2. 关键机制：是否占位、是否进入无障碍树不同。
3. 注意点：自定义 CSS 可能让 hidden 失效。
4. 可追问时补充：视觉隐藏但读屏可读要用专门类。

**常见追问**

- `aria-hidden="true"` 呢？→ 只对辅助技术隐藏，元素仍可见，不能用于可聚焦元素。

---

### Q28. `tabindex` 的 0、-1、正数有什么区别？

- **难度**: junior
- **标签**: 焦点, a11y

**答案**

- `tabindex="0"`：进入自然 Tab 顺序
- `tabindex="-1"`：不进入 Tab 顺序，但可以通过 JS `focus()`
- 正数：强制改变 Tab 顺序，容易造成混乱，通常避免

优先使用原生可聚焦元素，例如 `button`、`a href`、`input`。

**口述模板（30–60 秒）**

1. 一句话结论：0 加入自然顺序，-1 只允许程序聚焦。
2. 关键机制：浏览器按 DOM 顺序和 tabindex 生成焦点顺序。
3. 注意点：避免正数 tabindex。
4. 可追问时补充：弹窗打开时如何管理焦点。

**常见追问**

- `div tabindex="0"` 就是按钮了吗？→ 不是，还缺角色、Enter/Space 行为和状态。

---

### Q29. `iframe` 是什么？有什么常见问题？

- **难度**: junior
- **标签**: iframe

**答案**

`iframe` 用于在当前页面嵌入另一个浏览上下文，例如地图、支付、第三方组件。

常见问题：

- 加载额外页面，影响性能
- 跨域通信复杂
- 高度自适应麻烦
- 安全风险：点击劫持、第三方权限
- 需要 `title` 提供可访问名称

```html
<iframe src="https://example.com/map" title="门店地图" loading="lazy"></iframe>
```

**口述模板（30–60 秒）**

1. 一句话结论：iframe 嵌入独立页面上下文。
2. 关键机制：它有自己的文档和 JS 环境。
3. 注意点：性能、安全、通信和可访问性都要考虑。
4. 可追问时补充：sandbox 和 postMessage。

**常见追问**

- 父页面能直接操作跨域 iframe DOM 吗？→ 不能，需要 postMessage。

---

### Q30. `details` / `summary` 有什么用？

- **难度**: junior
- **标签**: 原生组件

**答案**

它们提供原生展开 / 收起交互：

```html
<details>
  <summary>退货规则</summary>
  <p>签收后 7 天内可申请退货。</p>
</details>
```

自带键盘支持和展开状态。适合 FAQ、简单折叠内容；复杂 Accordion 需求要评估样式和交互一致性。

**口述模板（30–60 秒）**

1. 一句话结论：details/summary 是原生折叠组件。
2. 关键机制：summary 是可交互控件，details 保存 open 状态。
3. 注意点：复杂设计可能需要自定义。
4. 可追问时补充：`name` 属性可让同名 details 互斥打开（较新特性，查兼容）。

**常见追问**

- 怎么监听展开收起？→ 监听 details 的 `toggle` 事件。

---

### Q31. `template` 标签有什么用？

- **难度**: junior
- **标签**: template

**答案**

`template` 中的内容会被解析，但不会渲染、不会执行脚本、不会加载图片等资源。JS 可以克隆其 `content`：

```html
<template id="item-tpl">
  <li class="item"></li>
</template>
```

```js
const tpl = document.querySelector('#item-tpl')
const node = tpl.content.cloneNode(true)
```

它常用于 Web Components 和原生 DOM 模板。

**口述模板（30–60 秒）**

1. 一句话结论：template 保存惰性 HTML 片段。
2. 关键机制：内容存在 DocumentFragment 中，默认不渲染。
3. 注意点：插入后才真正生效。
4. 可追问时补充：Declarative Shadow DOM 也用 template 语法。

**常见追问**

- 和隐藏 div 的区别？→ 隐藏 div 内容已是活动 DOM，资源和脚本可能被处理。
