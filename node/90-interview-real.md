# Node.js 真实面试题汇总

> 根据近年公开面经、社区高频题和大厂 Node / 全栈岗位面试趋势整理。`来源倾向` 表示题目常见场景，不代表某公司固定题库。

---

### R1. Node.js 是单线程的吗？如何处理高并发？

- **难度**: junior
- **标签**: 线程模型
- **来源倾向**: 一面必问

**答案**

JS 执行单线程，进程多线程：网络 I/O 由系统多路复用，文件等由 libuv 线程池处理，CPU 任务可用 worker_threads。

详见：[10-junior-qa Q2](10-junior-qa.md)

---

### R2. 讲讲 Node 事件循环的六个阶段

- **难度**: mid
- **标签**: 事件循环
- **来源倾向**: 高频原理

**答案**

timers → pending → idle/prepare → poll → check → close；每个回调后清空 nextTick 和 Promise 微任务。

详见：[20-mid-qa Q1](20-mid-qa.md)

---

### R3. 事件循环输出题（nextTick / Promise / setTimeout / setImmediate）

- **难度**: mid
- **标签**: 输出题
- **来源倾向**: 一面输出题

**答案**

同步 → nextTick → Promise → timers / check；I/O 回调中 setImmediate 先于 setTimeout；ESM 顶层顺序与 CJS 不同。

详见：[20-mid-qa Q2](20-mid-qa.md)

---

### R4. 哪些操作会使用 libuv 线程池？

- **难度**: mid
- **标签**: libuv
- **来源倾向**: 原理深挖

**答案**

fs、dns.lookup、部分 crypto、zlib；网络 I/O 不使用。默认 4 线程，可调 UV_THREADPOOL_SIZE。

详见：[20-mid-qa Q3](20-mid-qa.md)

---

### R5. 什么是 Stream 背压？如何处理？

- **难度**: mid
- **标签**: Stream
- **来源倾向**: 高频

**答案**

write 返回 false 时暂停，等待 drain；生产用 pipeline 自动处理背压与错误。

详见：[20-mid-qa Q6](20-mid-qa.md)、[Q7](20-mid-qa.md)

---

### R6. 如何读取并处理一个 10GB 的日志文件？

- **难度**: mid
- **标签**: Stream
- **来源倾向**: 场景手写

**答案**

createReadStream + readline 逐行处理，只在内存保留聚合结果。

详见：[40-coding-handwritten H5](40-coding-handwritten.md)

---

### R7. CommonJS 和 ESM 的区别？require 的加载原理？

- **难度**: mid
- **标签**: 模块
- **来源倾向**: 高频

**答案**

CJS 运行时同步加载、包装函数、缓存、循环依赖得到部分导出；ESM 静态、实时绑定；现代 Node 支持 require(esm)。

详见：[20-mid-qa Q11](20-mid-qa.md)、[Q12](20-mid-qa.md)

---

### R8. cluster 和 worker_threads 的区别？

- **难度**: mid
- **标签**: 并发
- **来源倾向**: 高频

**答案**

cluster 多进程共享端口扩展请求处理；worker_threads 同进程多线程处理 CPU 计算。容器中优先多副本。

详见：[20-mid-qa Q17](20-mid-qa.md)、[Q18](20-mid-qa.md)、[30-senior-qa Q9](30-senior-qa.md)

---

### R9. Koa 洋葱模型原理，手写 compose

- **难度**: mid
- **标签**: 中间件
- **来源倾向**: 前端转全栈高频

**答案**

dispatch 递归执行下一个中间件并返回 Promise，await next 实现前后置逻辑。

详见：[20-mid-qa Q14](20-mid-qa.md)、[40-coding-handwritten H2](40-coding-handwritten.md)

---

### R10. 如何捕获 Node 中的未处理异常？进程该不该继续运行？

- **难度**: mid
- **标签**: 错误处理
- **来源倾向**: 稳定性

**答案**

分层处理；uncaughtException 后记录日志并优雅退出，由进程管理器重启。

详见：[20-mid-qa Q15](20-mid-qa.md)

---

### R11. 如何实现优雅关闭？

- **难度**: mid
- **标签**: 部署
- **来源倾向**: 高级岗必问

**答案**

SIGTERM → readiness 失败 → server.close → 关闭空闲连接 → 关闭资源 → 超时强退。

详见：[20-mid-qa Q16](20-mid-qa.md)、[40-coding-handwritten H7](40-coding-handwritten.md)

---

### R12. Node 服务内存泄漏如何排查？

- **难度**: senior
- **标签**: 内存
- **来源倾向**: 高级岗必问

**答案**

监控确认趋势 → 复现 → 多次 heap snapshot 对比 → Retainers 定位 → 修复验证；区分堆内和堆外泄漏。

详见：[30-senior-qa Q4](30-senior-qa.md)、[Q5](30-senior-qa.md)

---

### R13. CPU 100% 如何排查？

- **难度**: senior
- **标签**: 性能
- **来源倾向**: 排障场景

**答案**

`--cpu-prof` / inspector 采集，火焰图找热点；常见 ReDoS、大 JSON、频繁 GC。

详见：[30-senior-qa Q6](30-senior-qa.md)

---

### R14. 发布后 P99 延迟翻倍，你怎么排查？

- **难度**: senior
- **标签**: 排障
- **来源倾向**: 2026 后端面试高频场景题

**答案**

先回滚止血；按事件循环、下游、资源（CPU throttling、GC）、线程池、连接池逐层排查，用 trace 定位最慢 span。

详见：[30-senior-qa Q7](30-senior-qa.md)

---

### R15. 偶发 502 是什么原因？

- **难度**: senior
- **标签**: HTTP
- **来源倾向**: 线上问题追问

**答案**

常见于 LB 空闲超时大于 Node keepAliveTimeout，Node 已关闭的连接被 LB 复用；调大 keepAliveTimeout 与 headersTimeout。

详见：[20-mid-qa Q21](20-mid-qa.md)

---

### R16. 什么操作会阻塞事件循环？如何监控？

- **难度**: mid
- **标签**: 性能
- **来源倾向**: 高频

**答案**

同步 I/O、大 JSON、ReDoS、同步加密；用 monitorEventLoopDelay、ELU 监控。

详见：[20-mid-qa Q4](20-mid-qa.md)

---

### R17. AsyncLocalStorage 有什么用？

- **难度**: mid
- **标签**: 上下文
- **来源倾向**: 中高级

**答案**

在异步链路中传递 traceId、用户信息等上下文；Node 24 基于 AsyncContextFrame 性能更好。

详见：[20-mid-qa Q20](20-mid-qa.md)、[30-senior-qa Q18](30-senior-qa.md)、[40-coding-handwritten H12](40-coding-handwritten.md)

---

### R18. 缓存穿透、击穿、雪崩分别怎么解决？

- **难度**: mid
- **标签**: 缓存
- **来源倾向**: 后端高频

**答案**

穿透：空值缓存 + 布隆过滤器；击穿：互斥 / 单飞 + 逻辑过期；雪崩：随机 TTL + 多级缓存 + 降级。

详见：[20-mid-qa Q25](20-mid-qa.md)、[40-coding-handwritten H9](40-coding-handwritten.md)

---

### R19. 实现一个限流器

- **难度**: senior
- **标签**: 限流
- **来源倾向**: 手写高频

**答案**

令牌桶惰性补充；分布式用 Redis Lua 原子操作。

详见：[40-coding-handwritten H8](40-coding-handwritten.md)

---

### R20. 如何实现分布式锁？有哪些坑？

- **难度**: senior
- **标签**: 分布式
- **来源倾向**: 后端岗

**答案**

SET NX PX + 唯一 token + Lua 解锁；注意过期续期、误删、主从切换丢锁，强一致用 fencing token。

详见：[30-senior-qa Q12](30-senior-qa.md)

---

### R21. JWT 和 Session 怎么选？token 存哪里？

- **难度**: mid
- **标签**: 认证
- **来源倾向**: 全栈高频

**答案**

需要即时吊销选 Session；JWT 短期 access + 可吊销 refresh；Web 端优先 HttpOnly Cookie。

详见：[20-mid-qa Q23](20-mid-qa.md)

---

### R22. Node 服务常见安全问题有哪些？

- **难度**: senior
- **标签**: 安全
- **来源倾向**: 高级岗

**答案**

注入、命令注入、路径穿越、SSRF、原型污染、ReDoS、供应链投毒；Permission Model 做纵深防御。

详见：[30-senior-qa Q15](30-senior-qa.md)、[Q16](30-senior-qa.md)、[Q17](30-senior-qa.md)

---

### R23. 如何设计一个支持百万连接的推送服务？

- **难度**: senior
- **标签**: 系统设计
- **来源倾向**: 系统设计高频

**答案**

连接网关与业务分离、路由表定向投递、心跳、seq 补拉、发布时避免重连风暴。

详见：[50-system-design S3](50-system-design.md)

---

### R24. 如何设计秒杀系统？

- **难度**: senior
- **标签**: 系统设计
- **来源倾向**: 后端经典系统设计

**答案**

层层削峰、Redis Lua 预扣库存、MQ 异步下单、数据库条件更新、幂等、延迟消息回补。

详见：[50-system-design S7](50-system-design.md)

---

### R25. Node 最新版本有哪些变化？你们如何升级？

- **难度**: senior
- **标签**: 版本
- **来源倾向**: 技术视野

**答案**

Node 26（2026-10 LTS）：Temporal 默认启用、V8 14.6、Undici 8；Node 27 起每年一个大版本且都进入 LTS。升级策略：CI 矩阵、原生模块检查、压测、灰度。

详见：[00-roadmap 版本说明](00-roadmap.md)、[30-senior-qa Q25](30-senior-qa.md)
