# 浏览器原理真实面试题

> 根据近年公开面经（社区面经帖、技术博客、面试复盘）整理的高频题。`来源倾向`表示这类题常见于哪类岗位或面试轮次，不代表某公司的固定题库。每题给出简短答案，并链接到详细讲解。

---

### R1. 从输入 URL 到页面显示发生了什么？

- **来源倾向**: 几乎所有前端面试，常作为开放题深挖

**答案**：URL 解析和缓存检查 → DNS → TCP + TLS → 发送请求 → 服务器响应 → 浏览器进程判断类型并提交给渲染进程 → 解析 HTML、构建 DOM 和 CSSOM → 样式计算 → 布局 → 分层、绘制 → 合成线程分块、光栅化 → 合成显示。

**口述模板**：

1. **网络**：先检查缓存，再 DNS 解析、建立 TCP 和 TLS 连接，发送 HTTP 请求。
2. **导航**：浏览器进程收到响应头后决定渲染还是下载，选择渲染进程并提交导航。
3. **解析**：解析 HTML 构建 DOM，遇到同步脚本会阻塞；CSS 阻塞渲染。
4. **渲染**：样式计算、布局、分层、绘制生成指令，合成线程分块光栅化后合成显示。
5. **追问点**：可以按面试官的兴趣深入 HTTP 缓存、TLS 握手、关键渲染路径或合成。

详见 [初级 Q5](10-junior-qa.md)、[中级 Q4](20-mid-qa.md)。

---

### R2. 浏览器有哪些进程？为什么要多进程？

- **来源倾向**: 中高级岗位一面

**答案**：浏览器进程、渲染进程、GPU 进程、网络服务、存储服务、插件 / 扩展进程等。多进程带来稳定性（一个页面崩溃不影响其他页面）、安全性（渲染进程沙箱 + 站点隔离）和并行性。代价是内存占用。

详见 [初级 Q3](10-junior-qa.md)、[中级 Q1–Q2](20-mid-qa.md)。

---

### R3. 讲讲事件循环，宏任务和微任务的区别

- **来源倾向**: 几乎所有前端面试，常配合输出题

**答案**：每轮循环执行一个任务，然后清空微任务队列，在合适时机渲染。微任务（Promise、queueMicrotask、MutationObserver）在当前任务结束后立即执行；任务（setTimeout、事件、网络回调）在后续的循环中执行。无限产生的微任务会阻塞渲染。

详见 [初级 Q14](10-junior-qa.md)、[中级 Q18–Q19](20-mid-qa.md)、[手写 H2](40-coding-handwritten.md)。

---

### R4. 事件循环输出题（async/await + Promise + setTimeout）

- **来源倾向**: 笔试、一面高频

**答案**：分析方法：先执行同步代码（包括 Promise 执行器和 async 函数中 await 之前的部分）；await 之后的代码相当于微任务；then 链中下一个回调要等上一个执行后才入队；任务中产生的微任务在该任务结束后立即执行。

**口述模板**：

1. **同步**：先标出所有同步输出，注意 Promise 执行器是同步的。
2. **微任务**：按入队顺序列出，await 之后的代码和 then 回调都是微任务。
3. **链式**：then 链的下一个回调在前一个执行完才入队。
4. **任务**：按定时器注册的顺序和延迟执行，每个任务后清空微任务。

详见 [中级 Q19](20-mid-qa.md)、[手写 H2](40-coding-handwritten.md)。

---

### R5. 重排和重绘是什么？怎么减少？

- **来源倾向**: 初中级岗位高频

**答案**：重排是重新计算布局（几何属性变化），重绘是重新绘制（视觉属性变化）。重排一定导致重绘。减少方式：批量修改 DOM、读写分离避免强制同步布局、动画使用 transform / opacity、减少 DOM 深度、使用 contain / content-visibility。

详见 [初级 Q13](10-junior-qa.md)、[性能优化中级 Q13](../performance/20-mid-qa.md)。

---

### R6. 为什么 transform 动画比 top / left 动画性能好？

- **来源倾向**: 中高级岗位，考察渲染原理

**答案**：top / left 修改会触发布局和绘制，在主线程执行；transform 只修改属性树中的变换，合成线程用已光栅化的图块重新合成，不需要布局和绘制，主线程繁忙时也能流畅运行。

详见 [中级 Q16](20-mid-qa.md)、[高级 Q2](30-senior-qa.md)。

---

### R7. CSS 和 JS 会阻塞页面渲染吗？

- **来源倾向**: 中级岗位高频

**答案**：CSS 不阻塞 DOM 解析，但阻塞渲染，并阻塞其后同步脚本的执行。同步 JS 阻塞 DOM 解析。defer 脚本在解析完成后按顺序执行；async 脚本下载完立即执行。预加载扫描器可以在解析阻塞时提前下载资源。

详见 [中级 Q11–Q12](20-mid-qa.md)。

---

### R8. 强缓存和协商缓存有什么区别？

- **来源倾向**: 几乎所有前端面试

**答案**：强缓存在新鲜期内直接使用，不发请求（Cache-Control: max-age、Expires）；过期后带验证器（If-None-Match / If-Modified-Since）请求，未修改返回 304（协商缓存）。ETag 优先于 Last-Modified。no-cache 是每次都验证，no-store 才是不缓存。

**口述模板**：

1. **强缓存**：Cache-Control 的 max-age 内直接使用缓存，不发请求。
2. **协商缓存**：过期后带上 ETag 或 Last-Modified 请求，未修改返回 304。
3. **区别**：no-cache 每次验证，no-store 完全不缓存。
4. **实践**：带 hash 的资源设置长期缓存 + immutable，HTML 设置 no-cache。

详见 [中级 Q5](20-mid-qa.md)、[手写 H12](40-coding-handwritten.md)。

---

### R9. 跨域有哪些解决方案？CORS 预检什么时候触发？

- **来源倾向**: 几乎所有前端面试

**答案**：CORS（主流）、开发代理 / 网关同源代理、postMessage（窗口间通信）、JSONP（只支持 GET，已过时）、WebSocket。非简单请求触发预检：方法不是 GET / HEAD / POST，或带自定义请求头，或 Content-Type 是 application/json 等。

详见 [初级 Q23](10-junior-qa.md)、[中级 Q9](20-mid-qa.md)、[手写 H11](40-coding-handwritten.md)。

---

### R10. XSS 和 CSRF 是什么？怎么防御？

- **来源倾向**: 几乎所有前端面试

**答案**：XSS 是注入恶意脚本在用户浏览器中执行，防御靠输出转义、避免危险 API、CSP、Trusted Types、HttpOnly Cookie。CSRF 是利用用户的登录态伪造请求，防御靠 SameSite Cookie、CSRF Token、Origin 校验，以及保证 GET 无副作用。

**口述模板**：

1. **区别**：XSS 是在用户浏览器中执行攻击者的代码，CSRF 是借用用户的身份发请求。
2. **XSS 防御**：按上下文转义，避免 innerHTML，严格 CSP 作为纵深防御。
3. **CSRF 防御**：SameSite=Lax 作为基础，状态修改请求校验 Token 或 Origin。
4. **注意**：HttpOnly 防止 Cookie 被窃取，但不能阻止 XSS 本身。

详见 [中级 Q26–Q27](20-mid-qa.md)、[手写 H9–H10](40-coding-handwritten.md)。

---

### R11. Cookie 有哪些属性？SameSite 的三个值有什么区别？

- **来源倾向**: 中级岗位

**答案**：Domain、Path、Expires / Max-Age、Secure、HttpOnly、SameSite、Partitioned。SameSite：Strict 完全不在跨站请求中发送；Lax（Chrome 默认）只在跨站的顶级导航 GET 中发送；None 都发送，但必须配合 Secure。

详见 [初级 Q21](10-junior-qa.md)。

---

### R12. Chrome 是不是已经禁用了第三方 Cookie？

- **来源倾向**: 中高级岗位，考察是否跟进最新变化

**答案**：没有。Chrome 2025 年宣布保留第三方 Cookie，不推出单独的选择弹窗，Privacy Sandbox 大部分 API 已退役。但 Safari 和 Firefox 默认阻止或分区第三方 Cookie，Chrome 也已对存储和 HTTP 缓存进行了分区。依赖第三方 Cookie 的方案仍需要适配。

详见 [中级 Q30](20-mid-qa.md)、[高级 Q10](30-senior-qa.md)。

---

### R13. localStorage、sessionStorage、Cookie、IndexedDB 的区别

- **来源倾向**: 初中级岗位

**答案**：Cookie 约 4KB，随请求发送；localStorage 约 5MB，持久化，同源共享；sessionStorage 只在当前标签页会话有效；IndexedDB 容量大、异步、支持索引和事务。localStorage 是同步 API，大量读写会阻塞主线程。

详见 [初级 Q20](10-junior-qa.md)。

---

### R14. 浏览器怎么进行垃圾回收？

- **来源倾向**: 中高级岗位

**答案**：V8 使用分代回收。新生代使用 Scavenge 复制算法，存活两次晋升到老生代；老生代使用标记清除和标记整理。通过增量标记、并发标记、并行回收减少主线程停顿，写屏障保证并发标记的正确性。

详见 [初级 Q28](10-junior-qa.md)、[中级 Q24](20-mid-qa.md)、[高级 Q7](30-senior-qa.md)。

---

### R15. V8 是怎么执行 JS 代码的？什么是 JIT？

- **来源倾向**: 高级岗位、基础架构方向

**答案**：解析为 AST → Ignition 解释执行字节码并收集类型反馈 → 热点代码由 Sparkplug、Maglev、TurboFan 逐层编译为机器码。优化编译基于类型假设，假设失效时去优化。隐藏类和内联缓存加快属性访问。

**口述模板**：

1. **解析**：生成 AST，暂时不用的函数只做预解析。
2. **解释**：Ignition 生成字节码执行，同时收集类型反馈。
3. **编译**：热点代码逐层交给 Sparkplug、Maglev、TurboFan 编译。
4. **去优化**：类型假设失效时回退到解释器。
5. **实践**：保持对象结构和类型稳定，避免 delete 属性。

详见 [中级 Q22–Q25](20-mid-qa.md)、[高级 Q6](30-senior-qa.md)。

---

### R16. HTTP/1.1、HTTP/2、HTTP/3 的区别

- **来源倾向**: 中高级岗位

**答案**：HTTP/1.1 文本协议，每个连接同时只能处理一个请求（需要多个连接）；HTTP/2 二进制分帧、多路复用、HPACK 头部压缩，但仍有 TCP 层队头阻塞；HTTP/3 基于 QUIC（UDP），流之间独立、握手更快、支持连接迁移。

详见 [初级 Q10](10-junior-qa.md)、[中级 Q6–Q7](20-mid-qa.md)。

---

### R17. HTTPS 是怎么保证安全的？握手过程是怎样的？

- **来源倾向**: 中级岗位高频

**答案**：非对称加密（密钥交换）协商出对称密钥，之后使用对称加密传输；证书验证服务器身份，防止中间人攻击。TLS 1.3 握手 1-RTT：客户端发送支持的套件和密钥共享参数，服务器回复参数、证书和签名，双方计算出会话密钥。

详见 [初级 Q8](10-junior-qa.md)、[中级 Q8](20-mid-qa.md)。

---

### R18. requestAnimationFrame 和 setTimeout 做动画有什么区别？

- **来源倾向**: 中级岗位

**答案**：rAF 在每帧渲染前执行，与刷新率同步，页面隐藏时暂停；setTimeout 与帧无关，可能在一帧中执行多次或跳帧，造成卡顿，且后台仍会执行（虽然会被节流）。

详见 [初级 Q16](10-junior-qa.md)、[中级 Q17](20-mid-qa.md)。

---

### R19. 事件委托的原理和优缺点

- **来源倾向**: 初中级岗位，常配合手写

**答案**：利用事件冒泡在父元素上统一处理子元素的事件。优点：减少监听器数量、支持动态添加的元素。缺点：不冒泡的事件（focus、blur）需要用 focusin、focusout；层级太深时判断成本略高；在 document 上注册非 passive 的触摸监听会影响滚动。

详见 [初级 Q18](10-junior-qa.md)、[手写 H1](40-coding-handwritten.md)。

---

### R20. 多个标签页之间怎么通信？

- **来源倾向**: 中级岗位

**答案**：同源：BroadcastChannel（首选）、storage 事件、SharedWorker、Service Worker；跨源：持有窗口引用时使用 postMessage。只让一个标签页执行任务时使用 Web Locks API。

详见 [中级 Q32](20-mid-qa.md)、[手写 H4](40-coding-handwritten.md)。

---

### R21. Service Worker 是什么？生命周期是怎样的？

- **来源倾向**: 中高级岗位，PWA / H5 方向

**答案**：运行在独立线程中的可编程网络代理，可以拦截请求、缓存资源、实现离线和推送。生命周期：注册 → 安装（预缓存）→ 等待（旧 SW 仍在控制）→ 激活（清理旧缓存）→ 控制页面；空闲时被终止。

详见 [中级 Q31](20-mid-qa.md)、[手写 H8](40-coding-handwritten.md)、[系统设计 S4](50-system-design.md)。

---

### R22. 什么是合成层？怎么排查层爆炸？

- **来源倾向**: 高级岗位、性能方向

**答案**：合成层单独光栅化，由合成线程合成，transform / opacity 变化不需要重绘。与合成层重叠的元素可能被隐式提升，导致层爆炸，占用大量 GPU 内存。用 DevTools Layers 面板查看层数量和提升原因；提高动画元素的 z-index、只在动画期间使用 will-change。

详见 [中级 Q15](20-mid-qa.md)、[高级 Q3](30-senior-qa.md)。

---

### R23. 什么是站点隔离？COOP / COEP 解决什么问题？

- **来源倾向**: 高级岗位、安全方向

**答案**：站点隔离让不同站点运行在不同的渲染进程中，防御 Spectre 读取同进程内存。COOP 与跨源窗口断开关联，COEP 要求跨源资源明确允许嵌入，二者同时满足后页面进入跨源隔离状态，才能使用 SharedArrayBuffer 和高精度计时器。

详见 [中级 Q2](20-mid-qa.md)、[高级 Q8](30-senior-qa.md)。

---

### R24. 页面白屏怎么排查和监控？

- **来源倾向**: 中高级岗位，稳定性方向

**答案**：排查：JS 报错（框架渲染失败）、关键资源加载失败、接口异常、兼容性问题。监控：加载后在多个采样点用 elementsFromPoint 判断是否只有容器元素，附带路由、错误和资源上下文；崩溃用 Reporting API 或 Service Worker 心跳检测。

**口述模板**：

1. **原因分类**：JS 错误、资源失败、接口异常、兼容性、崩溃。
2. **检测**：加载后采样点检测是否只有容器元素。
3. **上下文**：上报时附带路由、错误日志、资源加载情况和版本号。
4. **兜底**：框架错误边界显示降级 UI，资源加载失败时重试或切换 CDN。

详见 [高级 Q21](30-senior-qa.md)、[系统设计 S5](50-system-design.md)。

---

### R25. 浏览器的新特性你关注了哪些？

- **来源倾向**: 高级岗位，考察技术视野

**答案**：可以讲：INP 替代 FID 成为核心指标、Safari 26.2 支持 LCP 和 INP；Navigation API 和 View Transitions 改变了 SPA 路由和页面过渡；Speculation Rules 实现预渲染；容器查询、`:has()`、锚点定位；Chrome 保留第三方 Cookie 但存储分区；V8 的 Maglev 和 Turboshaft。重点是讲清楚**为什么**以及对项目的影响，并用 Baseline 判断能否使用。

详见 [高级 Q14–Q15](30-senior-qa.md)、[高级 Q22](30-senior-qa.md)、[路线图](00-roadmap.md)。
