# 前端安全真实面试题

> 根据近年公开面经和技术社区讨论整理，`来源倾向`表示这类题目常出现的场景，不代表某公司的固定题库。答案为要点，详细内容见对应链接。只讨论防御。

---

### R1. XSS 有哪几种？分别怎么防御？

- **来源倾向**: 必问，几乎每家公司

**答案**

- 存储型（恶意内容存入数据库，所有访问者受影响）、反射型（URL 参数原样输出）、DOM 型（前端 JS 把不可信数据写入危险汇）
- 核心防御：按上下文输出编码；避免危险汇（`innerHTML`、`v-html`、`dangerouslySetInnerHTML`）；富文本用 DOMPurify 白名单清洗；URL 校验协议
- 纵深防御：严格 CSP（nonce + strict-dynamic）、Trusted Types、HttpOnly Cookie

**口述模板（30–60 秒）**

1. **一句话结论**：XSS 分为存储型、反射型和 DOM 型，本质都是不可信数据被当作代码执行。
2. **关键机制**：核心防御是按上下文编码和避免危险汇，富文本使用白名单清洗。
3. **对比或注意点**：框架默认转义文本，但 `v-html`、`href` 等仍然需要处理。
4. **可追问时补充**：CSP 和 Trusted Types 作为纵深防御，HttpOnly 降低令牌被带走的风险。

详见：[10-junior Q2–Q3](./10-junior-qa.md)、[20-mid Q1–Q3](./20-mid-qa.md)

---

### R2. CSRF 的原理和防御？SameSite 能完全防住吗？

- **来源倾向**: 必问

**答案**

- 原理：浏览器跨站请求自动携带 Cookie，攻击者借用用户的身份
- 防御：SameSite Cookie、CSRF 令牌（同步令牌 / 签名的双重提交）、校验 Origin、Fetch Metadata（`Sec-Fetch-Site`）
- SameSite=Lax 不能完全防住：GET 请求有副作用时、同站的子域名（同站不等于同源）被攻破时、Lax 的兼容期例外

**口述模板（30–60 秒）**

1. **一句话结论**：CSRF 利用浏览器自动携带 Cookie 的特性，借用用户身份发起请求。
2. **关键机制**：SameSite Cookie 加 CSRF 令牌或校验 Origin。
3. **对比或注意点**：SameSite 按「站」判断，子域名被攻破时无效；GET 不能有副作用。
4. **可追问时补充**：Fetch Metadata 让服务端可以直接拒绝跨站的状态修改请求。

详见：[10-junior Q4](./10-junior-qa.md)、[20-mid Q7–Q8](./20-mid-qa.md)、[40 H5](./40-coding-handwritten.md)

---

### R3. Cookie 有哪些安全相关的属性？

- **来源倾向**: 高频

**答案**

- `HttpOnly`：JS 无法读取，降低 XSS 窃取的风险
- `Secure`：只通过 HTTPS 发送
- `SameSite`：Strict / Lax / None，控制跨站请求是否携带
- `Domain`：不设置时只发给当前主机（更安全）；`Path` 不是安全边界
- `__Host-` 前缀：强制 Secure、无 Domain、Path=/，防止子域名覆盖
- `Partitioned`（CHIPS）：第三方 Cookie 按顶级站点分区

详见：[10-junior Q6](./10-junior-qa.md)

---

### R4. CSP 是什么？怎么配置？为什么推荐 nonce 而不是域名白名单？

- **来源倾向**: 高频（中高级）

**答案**

- CSP 通过响应头限制页面可以加载和执行的资源
- 推荐的严格策略：`script-src 'nonce-xxx' 'strict-dynamic'; object-src 'none'; base-uri 'none'`
- 域名白名单容易被绕过：白名单中的域名（CDN、Google 等）上存在 JSONP 接口或可以托管任意脚本
- 落地：先 Report-Only 收集报告，改造内联脚本，再强制执行

**口述模板（30–60 秒）**

1. **一句话结论**：CSP 限制页面能执行的脚本来源，是 XSS 的纵深防御。
2. **关键机制**：每个请求生成随机 nonce，只有带正确 nonce 的脚本才能执行，strict-dynamic 信任它们加载的脚本。
3. **对比或注意点**：域名白名单容易通过白名单中的 JSONP 或公共 CDN 绕过。
4. **可追问时补充**：先 Report-Only 再强制执行，带 nonce 的页面不能被共享缓存。

详见：[10-junior Q22](./10-junior-qa.md)、[20-mid Q4–Q6](./20-mid-qa.md)、[30-senior Q2](./30-senior-qa.md)、[40 H4](./40-coding-handwritten.md)

---

### R5. 令牌存在 localStorage 还是 Cookie 里？

- **来源倾向**: 必问

**答案**

- localStorage：XSS 可以直接读取并带走，长期有效
- HttpOnly Cookie：JS 无法读取，但需要防 CSRF
- 内存 + 刷新令牌（HttpOnly Cookie）：折中
- BFF：令牌完全不进浏览器，安全性最好
- 关键认识：XSS 可以在页面中借用身份，任何存储方式都挡不住；首先要防 XSS

**口述模板（30–60 秒）**

1. **一句话结论**：推荐 HttpOnly Cookie 或 BFF，不推荐 localStorage。
2. **关键机制**：localStorage 中的令牌能被 XSS 读取并长期使用，HttpOnly Cookie 不能被读取。
3. **对比或注意点**：Cookie 方案要配合 SameSite 和 CSRF 防护。
4. **可追问时补充**：任何方式都挡不住 XSS 在页面内借用身份，所以根本是防 XSS。

详见：[10-junior Q9](./10-junior-qa.md)、[20-mid Q9–Q10](./20-mid-qa.md)、[30-senior Q6](./30-senior-qa.md)

---

### R6. JWT 的结构是什么？有什么缺点？怎么实现注销？

- **来源倾向**: 高频

**答案**

- 结构：header.payload.signature（base64url），载荷**没有加密**，只是签名
- 优点：无状态，服务端不需要查询会话
- 缺点：无法主动失效、体积大、载荷可见
- 注销方案：短有效期 + 刷新令牌（刷新令牌可以撤销）；黑名单（jti）；令牌版本号
- 校验：服务端固定算法、校验签名和 exp / iss / aud

详见：[10-junior Q12](./10-junior-qa.md)、[40 H9](./40-coding-handwritten.md)

---

### R7. OAuth 2.0 授权码流程是怎样的？为什么要 PKCE？

- **来源倾向**: 高频（中高级）

**答案**

- 流程：跳转到授权服务器 → 用户同意 → 带授权码回调 → 后端用授权码换取令牌
- PKCE：客户端生成随机的 verifier，授权时发送它的哈希（challenge），换令牌时发送 verifier；授权码被截获也无法使用
- `state` 防登录 CSRF
- 隐式流程已经不推荐（令牌出现在 URL 中）
- OAuth 是授权协议；登录（认证）使用 OIDC（ID Token）

详见：[10-junior Q13](./10-junior-qa.md)、[20-mid Q10–Q11](./20-mid-qa.md)、[40 H6](./40-coding-handwritten.md)

---

### R8. 跨域是怎么回事？CORS 配置有哪些安全问题？

- **来源倾向**: 必问

**答案**

- 同源策略限制读取跨源响应；CORS 是服务端声明允许哪些源读取
- 安全问题：反射请求的 Origin 并允许凭证（等于允许任意网站读取用户数据）；信任 `null` 源；用正则或 `endsWith` 校验时被 `evil-example.com` 绕过
- 正确做法：精确匹配的白名单；`Vary: Origin`
- CORS 不是 CSRF 防护：简单请求仍然会被发送

详见：[10-junior Q7](./10-junior-qa.md)、[浏览器原理中级](../browser/20-mid-qa.md)

---

### R9. 点击劫持是什么？怎么防？

- **来源倾向**: 常见

**答案**

- 攻击者用透明的 iframe 嵌入目标页面，诱导用户点击
- 防御：`Content-Security-Policy: frame-ancestors 'none'` 或 `'self'`；旧的 `X-Frame-Options: DENY`
- 不要依赖 JS 防嵌入（frame busting 可以被 sandbox 阻止）

详见：[10-junior Q5](./10-junior-qa.md)

---

### R10. 如何实现一个安全的富文本展示？

- **来源倾向**: 高频（有 UGC 的业务）

**答案**

- 白名单清洗（DOMPurify），而不是黑名单过滤
- 服务端存储前清洗 + 渲染时再清洗
- 链接校验协议，添加 `rel="noopener noreferrer"`
- 更好的方式：存储结构化的文档模型，由代码渲染
- CSP 和 Trusted Types 兜底；用户上传的文件放在独立域名

**口述模板（30–60 秒）**

1. **一句话结论**：使用白名单清洗，服务端和渲染端都清洗。
2. **关键机制**：DOMPurify 用浏览器解析器解析后只保留白名单中的标签和属性。
3. **对比或注意点**：黑名单总有遗漏，清洗后不要再修改 HTML。
4. **可追问时补充**：最好存储结构化的文档模型，从根本上避免 HTML 注入。

详见：[20-mid Q3](./20-mid-qa.md)、[40 H3](./40-coding-handwritten.md)、[50 S5](./50-system-design.md)

---

### R11. React / Vue 是不是就不会有 XSS 了？

- **来源倾向**: 高频

**答案**

- 框架默认转义文本插值，大幅减少 XSS
- 仍然存在的风险：`dangerouslySetInnerHTML` / `v-html`；`href` 中的 `javascript:`（React 19 会阻止，但其他协议和场景仍需处理）；直接操作 DOM（ref）；SSR 注入初始状态时没有转义 `</script>`；第三方组件；用户可控的模板（Vue 运行时编译）

详见：[20-mid Q2](./20-mid-qa.md)、[40 H1](./40-coding-handwritten.md)

---

### R12. 说说 HTTPS 为什么安全？中间人攻击怎么防？

- **来源倾向**: 高频

**答案**

- 证书链验证服务器身份，密钥交换（ECDHE）协商会话密钥，对称加密保证机密性和完整性
- 中间人攻击需要让用户信任伪造的证书 → 用户忽略警告、设备安装了恶意根证书
- 防御：HSTS（含 preload）、证书透明度监控、移动端可以考虑证书绑定（谨慎，有更新风险）

详见：[10-junior Q8](./10-junior-qa.md)、[计算机网络中级 Q20–Q23](../network/20-mid-qa.md)

---

### R13. 前端常见的安全响应头有哪些？

- **来源倾向**: 常见

**答案**

- `Content-Security-Policy`、`Strict-Transport-Security`、`X-Content-Type-Options: nosniff`
- `Referrer-Policy: strict-origin-when-cross-origin`、`Permissions-Policy`
- `Cross-Origin-Opener-Policy`、`Cross-Origin-Embedder-Policy`、`Cross-Origin-Resource-Policy`
- 移除 `Server`、`X-Powered-By`
- 通用的头在网关统一添加，CSP 由应用设置

详见：[10-junior Q21–Q25](./10-junior-qa.md)、[30-senior Q12](./30-senior-qa.md)

---

### R14. npm 供应链攻击你了解吗？团队怎么防？

- **来源倾向**: 高频（2025 年以后明显增多）

**答案**

- 事件：Shai-Hulud 蠕虫（窃取凭证并自动发布被感染的版本）、chalk / debug 维护者被钓鱼、polyfill.io 域名被收购后投毒
- 攻击方式：账号被盗、安装脚本、拼写近似的包、依赖混淆、CI 配置缺陷
- 防御：锁文件 + `npm ci`；禁用依赖的安装脚本（pnpm 10 默认、npm v12 默认，以官方为准）；依赖冷却期；私有代理仓库；SCA 扫描；可信发布
- 受影响后：轮换所有可能泄露的凭证

**口述模板（30–60 秒）**

1. **一句话结论**：供应链攻击通过被攻破的依赖在开发者电脑、CI 或用户浏览器中执行恶意代码。
2. **关键机制**：常见方式是维护者账号被盗后发布恶意版本，利用安装脚本窃取凭证。
3. **对比或注意点**：锁文件、禁用安装脚本和冷却期是最有效的三项措施。
4. **可追问时补充**：受影响后最重要的是轮换凭证，自己发布的包使用可信发布。

详见：[10-junior Q26–Q27](./10-junior-qa.md)、[20-mid Q21–Q24](./20-mid-qa.md)、[30-senior Q7–Q8](./30-senior-qa.md)

---

### R15. SRI 是什么？什么时候用？

- **来源倾向**: 常见

**答案**

- `<script src integrity="sha384-..." crossorigin="anonymous">`：浏览器校验资源的哈希，不匹配则拒绝执行
- 适用：从 CDN 加载的**固定版本**的资源
- 不适用：内容会变化的脚本（例如统计脚本的最新版本）
- 跨源资源需要 `crossorigin` 和 CORS 头

详见：[10-junior Q23](./10-junior-qa.md)、[20-mid Q24](./20-mid-qa.md)

---

### R16. 前端加密有意义吗？密码要不要在前端加密？

- **来源倾向**: 常见

**答案**

- 前端代码对攻击者完全可见，前端加密**不能替代 HTTPS**
- 前端哈希密码：哈希值本身就成了密码，攻击者截获后可以直接使用
- 可能的价值：防止明文密码出现在日志中、合规要求；但核心依然是 HTTPS + 服务端使用慢哈希（Argon2id、bcrypt）存储
- 代码混淆只能提高逆向成本

详见：[10-junior Q14、Q30](./10-junior-qa.md)

---

### R17. 什么是 SSRF？前端工程师为什么需要了解？

- **来源倾向**: 中高级、有 SSR / BFF 的团队

**答案**

- 服务端根据用户提供的 URL 发起请求，攻击者借此访问内网和云元数据
- 前端相关：SSR / BFF 的图片代理、链接预览、Webhook、截图服务
- 防御：协议和端口白名单、DNS 解析后检查所有 IP、固定连接的 IP（防 DNS 重绑定）、逐跳检查重定向、网络层隔离
- OWASP 2025 已把 SSRF 并入「访问控制失效」

详见：[20-mid Q16](./20-mid-qa.md)、[40 H10](./40-coding-handwritten.md)

---

### R18. 原型污染是什么？怎么防？

- **来源倾向**: 中级常见

**答案**

- 通过 `__proto__`、`constructor.prototype` 修改 `Object.prototype`，影响所有对象
- 常见入口：深合并、按路径设置值、查询字符串解析
- 防御：过滤危险键、只合并普通对象、使用 `Object.create(null)` 或 `Map`、用 `Object.hasOwn` 判断、`Object.freeze(Object.prototype)`（需要评估兼容性）

详见：[20-mid Q14](./20-mid-qa.md)、[40 H8](./40-coding-handwritten.md)

---

### R19. postMessage 怎么安全使用？

- **来源倾向**: 常见（有 iframe、微前端、SDK 场景）

**答案**

- 发送时指定准确的 `targetOrigin`，不使用 `'*'`
- 接收时精确校验 `event.origin`，必要时校验 `event.source`
- 校验数据结构，不把数据直接用于 `innerHTML`、跳转、`eval`

详见：[10-junior Q20](./10-junior-qa.md)、[40 H7](./40-coding-handwritten.md)

---

### R20. 如何设计一个安全的登录模块？

- **来源倾向**: 高频（中高级）

**答案**

- 传输：HTTPS + HSTS
- 存储：Argon2id / bcrypt
- 防撞库和暴力破解：限流（按账号和 IP）、按风险出现验证码、统一的错误提示（不暴露账号是否存在）
- MFA / Passkeys
- 会话：登录后重新生成会话 ID（防会话固定）、HttpOnly + Secure + SameSite
- 登录页：严格 CSP、`frame-ancestors 'none'`、不加载第三方脚本
- 跳转：校验 `redirect` 参数
- 审计与异常登录通知

**口述模板（30–60 秒）**

1. **一句话结论**：从传输、存储、防暴力破解、会话管理和页面安全几个方面设计。
2. **关键机制**：服务端慢哈希存储密码，按账号和 IP 限流，登录后重新生成会话 ID。
3. **对比或注意点**：错误提示不暴露账号是否存在，登录后的跳转要校验。
4. **可追问时补充**：支持 Passkeys 和 MFA，异地登录时通知用户。

详见：[10-junior Q14–Q15](./10-junior-qa.md)、[20-mid Q13](./20-mid-qa.md)、[40 H2](./40-coding-handwritten.md)

---

### R21. 越权（IDOR）问题怎么防？前端隐藏按钮够吗？

- **来源倾向**: 高频

**答案**

- 前端隐藏按钮、路由守卫只是体验，**权限必须在服务端检查**
- IDOR：修改请求中的 ID 访问别人的数据 → 服务端检查资源的归属，查询时带上当前用户的条件
- 使用不可预测的 ID（UUID）只能增加难度，不能替代授权检查
- OWASP 2025 中访问控制失效排名第一

详见：[20-mid Q25](./20-mid-qa.md)

---

### R22. Next.js / RSC 爆出过哪些安全漏洞？给你什么启示？

- **来源倾向**: 中高级、使用 Next.js 的团队

**答案**

- CVE-2025-29927：中间件授权可以通过内部请求头绕过 → 授权不能只放在中间件中，要在数据访问层检查
- React2Shell（CVE-2025-55182）：RSC 协议反序列化导致未鉴权的远程代码执行 → 新的服务端 / 客户端协议是新的攻击面
- 启示：快速升级的能力（SBOM、自动化测试）、纵深防御（授权在数据层）、WAF 作为临时缓解

详见：[30-senior Q10](./30-senior-qa.md)、[Next.js 高级 Q14–Q15](../nextjs/30-senior-qa.md)

---

### R23. 如何防止接口被刷（爬虫、薅羊毛、短信轰炸）？

- **来源倾向**: 常见（电商、营销业务）

**答案**

- 限流：按 IP、账号、设备、手机号多维度
- 人机识别：按风险出现的验证码、设备指纹、行为分析
- 业务规则：短信接口限制频率和每日上限、图形验证码前置、活动按账号和设备限制
- 前端的签名和混淆只能提高成本，不能作为主要防线
- 监控异常模式并实时调整策略

详见：[20-mid Q27](./20-mid-qa.md)、[30-senior Q21](./30-senior-qa.md)

---

### R24. 讲讲你在项目中做过的安全相关工作

- **来源倾向**: 中高级必问（行为面）

**答案**

- 使用 STAR 结构：背景（风险或事件）、任务（目标）、行动（技术方案和推进方式）、结果（量化）
- 可选的主题：落地 CSP、令牌存储改造、依赖治理、修复并复盘一次 XSS、Passkeys 接入、第三方脚本治理
- 要说出取舍和遇到的阻力

**口述模板（30–60 秒）**

1. **一句话结论**：用一句话说明做了什么、带来了什么结果。
2. **关键机制**：讲清技术方案，例如统一注入 nonce、Report-Only 收集、灰度强制执行。
3. **对比或注意点**：说出遇到的困难和取舍，例如第三方脚本和业务排期。
4. **可追问时补充**：给出量化的结果，例如覆盖率、拦截数量、漏洞数量变化。

详见：[30-senior Q1、Q25](./30-senior-qa.md)

---

### R25. AI 应用（聊天、Agent）的前端有哪些安全问题？

- **来源倾向**: 2025 年以后新增，AI 相关岗位常见

**答案**

- 提示词注入：网页、文档、邮件中的内容被模型当作指令 → 目前没有完全可靠的防御，需要限制影响
- 输出处理：模型输出是不可信的输入，Markdown 渲染禁用原始 HTML；图片和链接限制域名（防止通过图片 URL 外泄数据）
- 工具调用：最小权限、有副作用的操作需要用户确认
- 密钥：模型 API 密钥不能放在前端，通过 BFF 调用
- 成本：限流和额度

详见：[20-mid Q32](./20-mid-qa.md)、[30-senior Q16](./30-senior-qa.md)
