# 前端安全学习进度

## 完成度

| 文件 | 内容 | 数量 | 状态 | 完成日期 |
|------|------|------|------|----------|
| [00-roadmap](./00-roadmap.md) | 路线图、OWASP 2025、供应链现状 | - | ⬜ | |
| [10-junior-qa](./10-junior-qa.md) | 初级 Q&A | 30 | ⬜ | |
| [20-mid-qa](./20-mid-qa.md) | 中级 Q&A | 32 | ⬜ | |
| [30-senior-qa](./30-senior-qa.md) | 高级 Q&A | 26 | ⬜ | |
| [40-coding-handwritten](./40-coding-handwritten.md) | 手写题 | 12 | ⬜ | |
| [50-system-design](./50-system-design.md) | 系统设计 | 8 | ⬜ | |
| [90-interview-real](./90-interview-real.md) | 真实面试题 | 25 | ⬜ | |

状态：⬜ 未开始 / 🟨 进行中 / ✅ 完成 / 🔁 需要复习

## 建议节奏（5 周）

| 周 | 内容 | 目标 |
|----|------|------|
| 第 1 周 | roadmap + 初级 Q1–Q30 | 能讲清 XSS、CSRF、Cookie 属性、CORS、令牌存储、安全响应头、npm 风险 |
| 第 2 周 | 中级 Q1–Q17 + 手写 H1–H6 | DOM XSS、CSP nonce、Trusted Types、认证（BFF、OIDC、WebAuthn）、注入类问题；实现转义、URL 校验、清洗、CSRF 令牌、PKCE |
| 第 3 周 | 中级 Q18–Q32 + 手写 H7–H12 | 跨源隔离、供应链、访问控制、SSR 和 AI 安全；实现 postMessage、深合并、JWT 校验、SSRF 防护 |
| 第 4 周 | 高级 Q1–Q26 | 安全体系建设、CSP 落地、SSO、供应链应急、微前端和沙箱、合规 |
| 第 5 周 | 系统设计 S1–S8 + 真实面试题 R1–R25 | 模拟面试，准备 1–2 个安全相关的项目案例 |

**建议配合实操**：用 [CSP Evaluator](https://csp-evaluator.withgoogle.com/) 检查自己项目的 CSP；给项目开启 CSP Report-Only 看一周的报告；对项目跑一次 `npm audit` / `pnpm audit` 并检查哪些依赖有安装脚本。

## 薄弱点记录

| 日期 | 题号 | 问题 | 补充学习 |
|------|------|------|----------|
| | | | |

## 下次复习

- [ ] XSS 三种类型与上下文编码（初级 Q2–Q3、中级 Q1–Q2）
- [ ] SameSite 的局限与 CSRF 令牌（中级 Q7–Q8）
- [ ] CSP nonce + strict-dynamic 与落地路径（中级 Q4–Q6、高级 Q2）
- [ ] 令牌存储的取舍与 BFF（初级 Q9、中级 Q10、高级 Q6）
- [ ] 供应链攻击与应急（中级 Q21–Q22、高级 Q7）

## 模拟面试自检

- [ ] 能讲清三种 XSS，并说出框架仍然存在 XSS 的场景
- [ ] 能讲清 CSRF 的原理，以及 SameSite=Lax 防不住的情况
- [ ] 能写出严格的 CSP，并说明为什么不用域名白名单
- [ ] 能对比令牌存储的几种方案，并说明 XSS 是根本问题
- [ ] 能画出 OAuth 授权码 + PKCE 的流程
- [ ] 能手写白名单清洗、安全跳转校验、防原型污染的深合并
- [ ] 能说出 npm 供应链攻击的典型方式和团队的防御措施
- [ ] 能设计一个 SSO 系统和一个第三方脚本管理方案
- [ ] 准备好一个安全相关的项目案例（STAR 结构，有量化结果）
