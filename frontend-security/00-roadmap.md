# 前端安全学习路线（高级工程师面试向）

## 面试画像

前端安全在大厂面试中通常不是单独一轮，而是穿插在浏览器、框架、Node、工程化和系统设计中考察，集中在三层：

- **经典漏洞与防御**：XSS（三种类型）、CSRF、点击劫持、Cookie 安全属性、同源策略与 CORS、安全响应头 —— 初中级必问，**要能说出完整的防御组合，而不是单点手段**
- **机制与认证**：CSP（nonce / hash / strict-dynamic）、Trusted Types、SameSite 细节、JWT 与 Session、OAuth 2.0 + PKCE、OIDC、Passkeys、原型污染、SSRF、XS-Leaks、COOP / COEP —— 中高级区分点
- **体系与工程**：供应链安全（npm 蠕虫、依赖治理、可信发布）、SSR / RSC 框架漏洞、安全响应流程、CSP 大规模落地、第三方脚本治理、微前端与沙箱隔离、AI 应用的提示词注入、合规 —— 高级岗必谈

高级岗典型问法：

> 「React 已经自动转义了，为什么还会有 XSS？」「SameSite=Lax 是不是就不用 CSRF Token 了？」「JWT 放 localStorage 还是 Cookie？」「SPA 的 OAuth 登录为什么要用 PKCE 或 BFF？」「如果你们的 npm 依赖被投毒了，你怎么发现、怎么止损？」「怎么在一个大型老项目里落地 CSP？」

> **本专题只讲防御**：漏洞原理讲到能理解防御为止，不提供攻击载荷和利用细节。

## 标准与事件说明（截至 2026-10）

**OWASP Top 10:2025**（与 2021 版对比）：

| 排名 | 2025 | 变化 |
|------|------|------|
| A01 | Broken Access Control（失效的访问控制） | 保持第一，**SSRF 并入此类**；CSRF、开放重定向也在此类 |
| A02 | Security Misconfiguration（安全配置错误） | 从第五上升 |
| A03 | **Software Supply Chain Failures（软件供应链失效）** | 由「易受攻击和过时的组件」扩展而来，社区调查排名第一 |
| A04 | Cryptographic Failures（加密失效） | |
| A05 | Injection（注入，**包括 XSS**） | |
| A06 | Insecure Design（不安全的设计） | |
| A07 | Authentication Failures（认证失效） | |
| A08 | Software or Data Integrity Failures（软件或数据完整性失效） | |
| A09 | Security Logging & Alerting Failures（日志与告警失效） | 强调「告警」 |
| A10 | **Mishandling of Exceptional Conditions（异常情况处理不当）** | **新增**：错误处理、失败时放行（fail open） |

**供应链事件**（面试高频）：

- **polyfill.io**（2024）：域名易主后，CDN 脚本被注入恶意代码，影响大量引用该脚本的网站
- **chalk / debug 等包被投毒**（2025-09）：维护者被钓鱼，热门包被发布恶意版本（劫持加密货币交易）
- **Shai-Hulud**（2025-09）：首个在 npm 上**自我传播的蠕虫**，通过安装脚本窃取令牌并用受害者的令牌发布更多被感染的包；**Shai-Hulud 2.0**（2025-11）改用 `preinstall`，规模更大
- **Mini Shai-Hulud**（2026-05）：利用 **OIDC 可信发布配置范围过宽**（信任了仓库中的任意工作流或分支）发布恶意版本，说明「没有长期令牌」不等于安全
- **React2Shell / CVE-2025-55182**（2025-12，CVSS 10.0）：React Server Components 反序列化导致未鉴权远程代码执行（见 [Next.js 高级 Q15](../nextjs/30-senior-qa.md)）

**npm 平台的加固**（以 GitHub / npm 官方公告为准）：

- 2025-12-09：**永久吊销所有 classic token**；`npm login` 改为短期会话；新包默认开启 2FA；写权限的细粒度令牌最长 90 天
- 推荐 **OIDC 可信发布（Trusted Publishing）** + provenance；未验证的可信发布配置 48 小时后过期
- **npm v12**（2026-07）：**默认不再执行依赖的生命周期脚本**（`preinstall` / `install` / `postinstall`），需要显式允许 —— 直接针对 Shai-Hulud 的传播路径
- pnpm 10 起也默认不执行依赖的生命周期脚本，并提供 `minimumReleaseAge`（延迟安装刚发布的版本）

**浏览器**：

- Chrome 在 2024–2025 年**放弃了全面淘汰第三方 Cookie 的计划**，保留现状；Safari、Firefox 默认限制第三方 Cookie
- Trusted Types 已在 Chromium、Safari 26、Firefox 中逐步支持（以 MDN 兼容表为准）
- Passkeys 在主流平台普及

## 知识地图

```text
前端安全
├─ 基础模型：同源策略、站点 vs 源、CORS、Cookie（HttpOnly / Secure / SameSite / 前缀）、HTTPS
├─ 注入类：
│   ├─ XSS：存储型、反射型、DOM 型；上下文编码；框架中的风险点；富文本清洗
│   ├─ 原型污染、ReDoS、模板注入、SQL / NoSQL 注入（全栈视角）
│   └─ SSR 状态序列化注入
├─ 请求伪造：CSRF、SSRF、开放重定向
├─ 界面与隔离：点击劫持、iframe sandbox、postMessage、COOP / COEP / CORP、XS-Leaks
├─ 防御机制：CSP（nonce / hash / strict-dynamic / report）、Trusted Types、SRI、安全响应头
├─ 认证与会话：Session vs Token、JWT、刷新令牌轮换、OAuth 2.0 + PKCE、OIDC、BFF、SSO、Passkeys、MFA
├─ 供应链：npm 投毒与蠕虫、锁文件、安装脚本、可信发布与 provenance、SBOM、第三方脚本、CDN
├─ 框架与服务端：React / Vue 的风险点、SSR / RSC 漏洞、Node 服务加固、Server Actions
├─ 数据保护：敏感数据存储、脱敏、source map、环境变量、前端加密的边界
├─ 新领域：AI 应用（提示词注入、输出渲染）、低代码沙箱、浏览器扩展、WebView JSBridge
└─ 体系：威胁建模、安全测试（SAST / DAST / SCA）、监控与告警、应急响应、合规（PIPL / GDPR / PCI DSS）
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 安全全景、XSS / CSRF / 点击劫持、Cookie、认证基础、输入校验、安全头、供应链基础
2. [中级 Q&A](20-mid-qa.md) — DOM XSS、框架风险、CSP 与 Trusted Types、CSRF 深入、JWT、OAuth + PKCE、Passkeys、原型污染、SSRF、XS-Leaks、供应链攻击手法
3. [高级 Q&A](30-senior-qa.md) — 安全体系建设、CSP 落地、认证架构、供应链事件复盘、框架漏洞应对、微前端隔离、AI 应用安全、应急响应、威胁建模
4. [手写与现场编码](40-coding-handwritten.md) — 12 道题：上下文转义、URL 校验、HTML 清洗、CSP nonce、CSRF Token、PKCE、安全的 postMessage、防原型污染、JWT 校验、SSRF 防护等
5. [系统设计与场景](50-system-design.md) — SSO、CSP 监控平台、供应链治理、第三方脚本管理、UGC 内容安全、支付页面等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. 同源策略的范围，「源」和「站点」的区别（SameSite 按站点判断）
2. XSS 三种类型，**按输出上下文**编码（HTML、属性、JS、URL、CSS）
3. 框架中的 XSS 风险点：`dangerouslySetInnerHTML` / `v-html`、`href="javascript:"`、SSR 状态注入、第三方库
4. CSP：nonce + `strict-dynamic` 的严格策略、`object-src 'none'`、`base-uri`、Report-Only 灰度上线
5. Trusted Types：从源头上消除 DOM XSS 的「危险汇点」
6. CSRF 的原理和防御组合：SameSite、CSRF Token、校验 Origin、关键操作二次确认
7. Cookie 的安全属性和 `__Host-` 前缀
8. 令牌存储的取舍：HttpOnly Cookie vs 内存 vs localStorage
9. JWT 的结构和常见错误（`alg: none`、算法混淆、不校验过期、无法吊销）
10. OAuth 2.0 授权码 + PKCE，为什么不再使用隐式模式；SPA 使用 BFF 模式
11. Passkeys / WebAuthn 的原理：为什么能防钓鱼
12. 原型污染的原理和防御
13. SSRF：服务端抓取用户提供的 URL 时的防护（DNS 重绑定）
14. 点击劫持、`frame-ancestors`；COOP / COEP；XS-Leaks 的概念
15. 供应链：锁文件、禁用安装脚本、依赖冷却期、可信发布与 provenance、SRI、第三方脚本治理
16. SSR / RSC 特有风险：数据泄露、Server Actions 鉴权、框架漏洞的应急
17. 安全响应头清单及其作用
18. 安全体系：威胁建模、SAST / DAST / SCA、监控告警、应急响应流程

## 相关技术

- [浏览器原理](../browser/00-roadmap.md)：同源策略、XSS（中级 Q26）、CSRF（中级 Q27）、CSP（中级 Q28）、点击劫持（中级 Q29）、第三方 Cookie（中级 Q30）、Spectre 与 COOP / COEP（高级 Q8）、浏览器安全模型（高级 Q9）、存储隔离（高级 Q10）—— **本专题不重复这些基础，而是从防御体系和工程落地继续深入**
- [计算机网络](../network/00-roadmap.md)：HTTPS、TLS、HSTS、证书、DDoS、请求走私
- [Next.js](../nextjs/00-roadmap.md)：Server Actions 安全、CVE-2025-29927、React2Shell
- [Node.js](../node/00-roadmap.md)：服务端安全、依赖管理
- [React](../react/00-roadmap.md)、[Vue](../vue/00-roadmap.md)：框架的转义机制和风险点
- [webpack](../webpack/00-roadmap.md)、[Vite](../vite/00-roadmap.md)：source map、构建产物中的密钥

## 自检清单

- [ ] 能说出 XSS 的三种类型，以及按上下文编码的规则
- [ ] 能说出 React / Vue 中仍然可能产生 XSS 的 5 个场景
- [ ] 能写出一个严格的 CSP 并解释每条指令，说明 Report-Only 上线流程
- [ ] 能讲清 SameSite 的三个值，以及为什么 Lax 不能完全替代 CSRF Token
- [ ] 能比较三种令牌存储方式的取舍并给出推荐方案
- [ ] 能画出 OAuth 2.0 授权码 + PKCE 的流程
- [ ] 能解释 Passkeys 为什么能防钓鱼
- [ ] 能说出 Shai-Hulud 的传播方式和团队的防御措施
- [ ] 能写出防原型污染的合并函数和带 DNS 校验的 SSRF 防护
- [ ] 能讲出一次安全事件的应急响应流程
