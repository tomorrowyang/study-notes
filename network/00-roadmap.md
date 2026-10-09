# 计算机网络学习路线（高级工程师面试向）

## 面试画像

计算机网络是前端、Node、全栈岗位的基础必考项，国内大厂面试中几乎每轮都会涉及，考察集中在三层：

- **协议基础**：分层模型、TCP / UDP、三次握手与四次挥手、DNS、HTTP 报文与状态码、HTTPS —— 初中级必问，**要能讲出「为什么」**（为什么三次握手、为什么 TIME_WAIT 是 2MSL）
- **协议深入**：TCP 可靠传输、流量控制、拥塞控制（Cubic、BBR）、HTTP/2 多路复用、HTTP/3 与 QUIC、TLS 1.3 握手、HTTP 缓存、WebSocket —— 中高级区分点
- **工程与排查**：弱网优化、CDN 架构、负载均衡、长连接推送系统、抓包与线上问题排查（超时、连接数、TIME_WAIT / CLOSE_WAIT）、网络安全（DDoS、请求走私、证书管理）—— 高级岗必谈

高级岗典型问法：

> 「从输入 URL 到页面显示，网络层面发生了什么？每一步的耗时怎么优化？」「服务器上大量 TIME_WAIT / CLOSE_WAIT 怎么办？」「HTTP/2 已经多路复用了，为什么还需要 HTTP/3？」「QUIC 为什么基于 UDP？」「接口偶发超时怎么排查？」「设计一个百万连接的推送系统」

## 版本与标准说明（截至 2026-10）

| 协议 / 技术 | 标准 | 现状 |
|-------------|------|------|
| HTTP 语义 / 缓存 | RFC 9110 / 9111（2022） | 取代 RFC 7230–7235 系列 |
| HTTP/2 | RFC 9113（2022） | 浏览器普遍支持；**Server Push 已被主流浏览器移除**，改用 103 Early Hints |
| HTTP/3 / QUIC | RFC 9114 / RFC 9000（2021–2022） | 主流浏览器和 CDN 普遍支持，占全网请求的可观比例（具体比例以 Cloudflare Radar 等为准） |
| TLS 1.3 | RFC 8446（2018） | 主流；TLS 1.0 / 1.1 已被浏览器弃用 |
| **后量子密钥交换** | X25519MLKEM768（混合方案） | Chrome 131、Firefox 132 起默认启用，Apple 在 iOS 26 / macOS Tahoe 中默认启用；Cloudflare 报告其大部分人类流量已使用后量子密钥交换。**后量子证书（身份认证）尚未标准化** |
| ECH（加密 Client Hello） | IETF 草案 / 推进中 | Chrome、Firefox 已支持，Cloudflare 等部署；隐藏访问的域名（SNI） |
| HTTPS / SVCB DNS 记录 | RFC 9460（2023） | 浏览器用于发现 HTTP/3 和 ECH 配置 |
| DoH / DoT | RFC 8484 / RFC 7858 | 浏览器和系统层面普遍支持 |
| 拥塞控制 | Cubic RFC 9438；BBR v3 | Linux 默认 Cubic；BBR 在 Google 等大规模部署 |
| WebTransport | W3C / IETF 推进中 | Chrome、Firefox 支持，Safari 推进中（以官方为准） |
| 证书有效期 | CA/B 论坛 SC-081 | 最长有效期逐步缩短：2026-03 起 200 天，2027 年 100 天，2029 年 47 天；**证书自动化（ACME）成为必需** |

> 以上细节以 RFC、IETF 和各浏览器、CDN 厂商的公告为准。面试时说清「标准」和「部署现状」的区别会加分。

## 知识地图

```text
计算机网络
├─ 分层模型：OSI 七层、TCP/IP 四层、封装与解封装
├─ 网络层：IP、子网与 CIDR、ARP、NAT、ICMP、IPv6、路由、MTU 与分片
├─ 传输层：
│   ├─ TCP：三次握手、四次挥手、TIME_WAIT、可靠传输（序号 / 确认 / 重传 / SACK）、
│   │        滑动窗口、流量控制、拥塞控制（慢启动 / Cubic / BBR）、Nagle 与延迟确认、Keepalive
│   └─ UDP：特点、适用场景、为什么 QUIC 选择 UDP
├─ DNS：解析流程、记录类型、缓存与 TTL、GSLB、DNSSEC、DoH / DoT、HTTPS 记录
├─ HTTP：报文、方法、状态码、头部、缓存、Cookie、内容协商、Range、
│        HTTP/1.1 → HTTP/2（帧、流、HPACK）→ HTTP/3（QUIC、QPACK、0-RTT、连接迁移）
├─ TLS：TLS 1.3 握手、证书链、前向安全、0-RTT、HSTS、后量子密钥交换、ECH
├─ 实时通信：轮询、长轮询、SSE、WebSocket、WebTransport、WebRTC
├─ 基础设施：代理（正向 / 反向）、CDN、负载均衡（L4 / L7）、Anycast、API 网关
├─ 性能：延迟构成、弱网优化、连接复用、预连接、压缩（gzip / br / zstd）
├─ 安全：DDoS、中间人、请求走私、DNS 劫持、证书管理
└─ 排查工具：ping、traceroute / mtr、dig、curl、tcpdump、Wireshark、ss / netstat、Chrome DevTools
```

## 阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 分层模型、IP、TCP / UDP、DNS、HTTP 基础、HTTPS、WebSocket、CDN、工具
2. [中级 Q&A](20-mid-qa.md) — TCP 深入、HTTP/2、HTTP/3 与 QUIC、HTTP 缓存与传输细节、TLS 1.3、DNS 深入、实时通信、负载均衡
3. [高级 Q&A](30-senior-qa.md) — 全链路延迟分析、弱网优化、线上排查、CDN 与网关架构、长连接系统、网络安全、协议设计取舍
4. [手写与现场编码](40-coding-handwritten.md) — 12 道题：HTTP 服务器、重试与退避、并发控制、WebSocket 重连、SSE、拆包、限流、反向代理等
5. [系统设计与场景](50-system-design.md) — IM 长连接、大文件上传、CDN 加速、API 网关、实时推送、弱网优化等
6. [真实 / 高频面试题](90-interview-real.md) — 用面试问法串起来复习
7. [进度表](PROGRESS.md)

## 高级岗必会点（18 条）

1. TCP/IP 四层模型，每层的职责和代表协议
2. 三次握手的过程，**为什么是三次**（同步双方序号、防止历史连接）
3. 四次挥手的过程，**TIME_WAIT 为什么是 2MSL**，大量 TIME_WAIT / CLOSE_WAIT 的原因和处理
4. TCP 可靠传输：序号、确认、超时重传、快速重传、SACK
5. 流量控制（接收窗口）和拥塞控制（拥塞窗口）的区别
6. 拥塞控制：慢启动、拥塞避免、快速恢复；Cubic 与 BBR 的思路差异
7. TCP 和 UDP 的区别，QUIC 为什么基于 UDP
8. DNS 解析流程、缓存层级、DNS 负载均衡、DoH、HTTPS 记录
9. HTTP 报文、常用方法和状态码、幂等性
10. HTTP 缓存：强缓存、协商缓存、`no-cache` vs `no-store`、`Vary`、`stale-while-revalidate`
11. HTTP/1.1 的队头阻塞，HTTP/2 的帧和流、HPACK，**HTTP/2 仍然存在的 TCP 层队头阻塞**
12. HTTP/3 / QUIC：0-RTT / 1-RTT 建连、独立流、连接迁移、QPACK、用户态拥塞控制
13. TLS 1.3 握手（1-RTT）、证书链验证、前向安全、0-RTT 的重放风险
14. WebSocket 握手、心跳、重连；SSE、WebSocket、WebTransport 的选型
15. CDN 原理：DNS / Anycast 调度、边缘缓存、回源、缓存键
16. 负载均衡：L4 和 L7、常见算法、会话保持
17. 弱网优化：减少往返、连接复用、压缩、重试与退避、离线与降级
18. 线上网络问题的排查思路和工具（curl -w、dig、mtr、tcpdump、Wireshark、ss）

## 相关技术

- [浏览器原理](../browser/00-roadmap.md)：导航流程、HTTP 缓存（中级 Q5）、HTTP/2 与 HTTP/3（中级 Q6–Q7）、TLS（中级 Q8）、CORS（中级 Q9）、连接管理（中级 Q10）—— **本专题不重复这些内容，而是从协议层继续深入**
- [性能优化](../performance/00-roadmap.md)：TTFB、资源加载、预连接、CDN
- [Node.js](../node/00-roadmap.md)：`net`、`http`、流、事件循环与高并发连接
- [前端安全](../frontend-security/00-roadmap.md)：HTTPS、HSTS、CSP、CSRF、SSRF
- [Docker](../docker/00-roadmap.md)：容器网络、端口映射、DNS

## 自检清单

- [ ] 能画出三次握手和四次挥手的状态变化，并解释每个状态
- [ ] 能讲清 TIME_WAIT 的两个作用，以及服务器上 TIME_WAIT 过多的处理方法
- [ ] 能说明流量控制和拥塞控制的区别，以及 BBR 和 Cubic 的思路差异
- [ ] 能讲清 HTTP/2 解决了什么、没有解决什么，HTTP/3 又是怎么解决的
- [ ] 能画出 TLS 1.3 的握手过程，并说明 0-RTT 的风险
- [ ] 能从网络角度讲出「从输入 URL 到页面显示」的每一步耗时和优化手段
- [ ] 能用 curl、dig、mtr、tcpdump 定位一个接口超时问题
- [ ] 能设计一个支持百万连接的长连接推送系统
