# Docker 学习路线（高级面试向）

> 目标：能在国内互联网高级工程师 / DevOps / 后端面试中讲清容器原理、镜像构建、网络存储、安全与生产排障，并能做方案取舍。

## 面试画像

- **岗位侧重**：后端工程化、DevOps、SRE、平台；前端偶尔考「会用 Compose / 多阶段构建」
- **假定基础**：会 Linux 基础命令；有一门后端或脚本语言；了解进程、端口、文件系统概念
- **学完应能**：
  - 从 Namespace / cgroup / overlay2 讲到 `docker run` 发生了什么
  - 写出可缓存、可安全、可复现的生产级 Dockerfile
  - 排查容器秒退、网络不通、磁盘打满、OOM
  - 说清 Compose 与 K8s 边界，以及单机容器化基础设施怎么搭

## 版本说明（本文默认语境）

- **Docker Engine 23+**：Linux 镜像构建默认 **BuildKit**；建议安装 Buildx
- **Compose V2**：命令为 `docker compose`（空格）；`version` 字段已过时可省略
- **存储驱动**：生产常见仍为 **overlay2**；关注联合文件系统与写时复制（Copy-on-Write）
- **cgroup**：主流发行版默认 **cgroup v2**
- 编排延伸：Swarm 了解即可；生产集群重点在 **Kubernetes** 边界对比

## 知识地图

- **基础概念**
  - 镜像 Image / 容器 Container / 仓库 Registry
  - 客户端 CLI ↔ `dockerd` ↔ containerd ↔ runc
  - 与虚拟机对比（共享内核 vs Guest OS）
- **镜像与构建**
  - 分层 Layer、缓存、`.dockerignore`
  - Dockerfile 指令、多阶段构建 Multi-stage
  - BuildKit：并行、cache mount、secret mount、多架构
- **运行与资源**
  - `docker run` 参数、重启策略、健康检查 Healthcheck
  - cgroup 限制 CPU/内存/PIDs；OOM 行为
  - PID 1、僵尸进程、`tini` / `--init`
- **网络**
  - bridge / host / none / 自定义 bridge / overlay（多机）
  - veth、docker0、iptables NAT、端口映射
  - 容器 DNS、服务发现（Compose 服务名）
- **存储**
  - 可写层 vs Volume vs Bind Mount vs tmpfs
  - 数据持久化、权限、备份
- **编排与工程**
  - Compose 多服务、depends_on + health、profiles
  - CI 构建推送、镜像签名/扫描（概念）、Harbor
- **安全与生产**
  - 非 root、只读根、capabilities、seccomp
  - 日志驱动与轮转、磁盘清理、镜像治理
  - 与 K8s 的职责边界
- **底层原理（高级必打）**
  - Namespace：pid / net / mnt / uts / ipc / user
  - cgroup v1/v2 差异（概念级）
  - overlay2：lower/upper/merged、whiteout

## 建议阅读顺序

1. [初级 Q&A](10-junior-qa.md) — 建立术语与命令肌肉记忆  
2. [中级 Q&A](20-mid-qa.md) — 原理、对比、常见坑  
3. [高级 Q&A](30-senior-qa.md) — 底层、性能、安全、生产  
4. [手写与现场题](40-coding-handwritten.md) — Dockerfile / Compose / 排障脚本  
5. [系统设计与场景](50-system-design.md) — 基础设施与发布方案  
6. [真实/高频面试题](90-interview-real.md) — 用面试问法串起来复习  
7. 边学边勾 [PROGRESS.md](PROGRESS.md)

## 高级面试必打点

1. 容器不是轻量 VM：共享内核，隔离靠 Namespace，配额靠 cgroup  
2. `docker run` 全链路：镜像拉取 → 创建可写层 → 配网络 → runc 启动  
3. 镜像分层与缓存失效规则；为何「依赖先于源码 COPY」  
4. 多阶段构建：最终镜像不含编译器与源码  
5. BuildKit vs 旧 builder：并行、secret、cache mount  
6. bridge 默认网络 vs 用户自定义 bridge（DNS 服务发现）  
7. 端口映射本质是 iptables DNAT（再加用户态转发细节可提 `docker-proxy`）  
8. Volume vs Bind Mount：生产持久化优先 Volume  
9. 容器层写数据会丢；有状态服务要外挂存储并谈备份  
10. 内存限制与宿主机 OOM Killer 关系  
11. PID 1 与信号处理、僵尸进程  
12. 非 root + 最小 capabilities + 不挂载 docker.sock 到业务容器  
13. 日志打满磁盘：`json-file` 轮转或集中式日志  
14. 镜像安全：固定 tag/digest、扫描、不把密钥打进层  
15. Compose 适合单机/开发联调；K8s 负责集群调度与自愈  
16. 容器秒退排障顺序：状态 → 退出码 → logs → 命令/环境 → 依赖/网络/权限  
17. 跨主机网络：overlay / 外置 CNI（常落到 K8s）  
18. Rootless Docker：更安全但有性能与功能代价  

## 关联技术

- **Linux**：进程、cgroup、iptables/nftables、文件系统、systemd  
- **Kubernetes**：Pod/Service 与「单容器 Docker」的映射；面试常问边界  
- **CI/CD**：GitHub Actions / GitLab CI 里 build → push → deploy  
- **可观测**：Prometheus + cAdvisor / 容器日志链路  
- **镜像仓库**：Harbor、云厂商 ACR/ECR  

## 自测清单（学完打勾）

- [ ] 能白板画出 Namespace + cgroup + overlay 与容器的关系  
- [ ] 能现场写多阶段 Dockerfile + Compose（含 healthcheck）  
- [ ] 能讲一次「磁盘满了」和一次「容器不断重启」的排查  
- [ ] 能对比 Docker Compose 与 K8s，并说明团队何时该迁  
