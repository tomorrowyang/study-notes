# Docker 中级 Q&A（能讲清）

> 目标：原理、对比、常见坑。面试里这层出现频率最高。

---

## 一、镜像分层与构建

### Q1. 镜像分层（Layer）是怎么工作的？和缓存有什么关系？

- **难度**: mid
- **标签**: Layer, 缓存

**答案**

镜像由多个只读层叠成。Dockerfile 中多数指令产生层；构建时若某层输入未变，可复用缓存。  
**缓存失效会连带后面所有层重做**。因此要把「少变的步骤」（依赖安装）放前面，「常变的」（业务代码 COPY）放后面。

**口述模板（30–60 秒）**

1. 镜像是只读层叠加，构建靠层缓存加速。  
2. 某层变了，后续层全部失效。  
3. 所以依赖先于源码。  
4. 可追问：`--mount=type=cache` 与层缓存区别。

**常见追问**

- 为什么 `RUN apt-get update && apt-get install` 要写成一行？→ 避免只缓存了 update、install 时用过期索引；也减少层数与中间状态。

---

### Q2. 多阶段构建（Multi-stage）解决什么问题？

- **难度**: mid
- **标签**: Multi-stage

**答案**

用多个 `FROM`：前阶段编译，后阶段只拷贝产物到运行镜像。最终镜像不含编译器、源码、测试依赖 → **更小、更安全、可复现**。

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app

FROM gcr.io/distroless/static:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

**口述模板（30–60 秒）**

1. 编译环境与运行环境分离。  
2. 只把二进制/静态文件拷到最终镜像。  
3. 体积与攻击面双降。  
4. 可追问：BuildKit 并行构建多个 stage。

**常见追问**

- 只有一个阶段算多阶段吗？→ 不算；至少两个 FROM 才体现价值。

---

### Q3. `COPY` 与 `ADD` 如何选择？

- **难度**: mid
- **标签**: Dockerfile

**答案**

默认用 **COPY**：行为清晰。  
ADD 额外能力：本地 tar 自动解压、接受 URL（不推荐用 ADD 下 URL，网络与可复现性差）。需要解压时也可 `COPY` + `RUN tar`。

**口述模板（30–60 秒）**

1. 能 COPY 就 COPY。  
2. ADD 的魔法易让审查者困惑。  
3. 下载用明确 RUN curl/wget 并校验校验和更稳。  
4. 可追问：构建可复现性。

**常见追问**

- 官方最佳实践怎么说？→ 明确倾向 COPY。

---

### Q4. 构建缓存总被打穿，常见原因？

- **难度**: mid
- **标签**: 缓存, CI

**答案**

- 先 `COPY . .` 再装依赖 → 任意文件改动失效  
- `ARG` 影响其后的缓存键  
- 未用 BuildKit cache mount，包管理每次重下  
- CI 每次空 runner 无 `--cache-from` / registry cache  
- `RUN` 里用了 `date`、无固定版本的 `latest` 包  

**口述模板（30–60 秒）**

1. 先看 Dockerfile 指令顺序。  
2. 再看 ARG 与上下文是否误拷。  
3. CI 要导出/导入缓存。  
4. 可追问：Go/npm 的 cache mount 写法。

**常见追问**

- `--no-cache` 什么时候用？→ 怀疑缓存脏、或强制刷新安全补丁时。

---

### Q5. BuildKit 相比旧 builder 的关键收益？

- **难度**: mid
- **标签**: BuildKit

**答案**

自 Engine **23.0** 起，Linux 镜像构建默认走 BuildKit（经 Buildx）。收益：

- 独立 stage **并行**构建  
- `RUN --mount=type=cache` 持久化包缓存且不进镜像层  
- `RUN --mount=type=secret` 注入密钥不落层  
- 更好的前端输出与多平台 `buildx --platform`  

旧 classic builder 已处于弃用路径，新项目不要依赖它。

**口述模板（30–60 秒）**

1. BuildKit 是现代默认构建引擎。  
2. 并行、缓存挂载、密钥挂载是三大卖点。  
3. 配合 buildx 做多架构。  
4. 可追问：远程缓存到 Registry。

**常见追问**

- `DOCKER_BUILDKIT=0`？→ 退回旧构建，仅兼容遗留；应尽快迁移。

---

### Q6. 什么是镜像 digest？为什么生产要用？

- **难度**: mid
- **标签**: digest, 供应链

**答案**

Digest 是内容寻址标识（`repo@sha256:...`）。同一 digest 内容不变；tag 可被移动重指。  
生产部署、审计、回滚应用 digest 或不可变 tag 策略，避免「同名不同货」。

**口述模板（30–60 秒）**

1. tag 可变，digest 钉死内容。  
2. 生产发布钉 digest 更稳。  
3. 安全审计可对齐唯一产物。  
4. 可追问：镜像签名与验收。

**常见追问**

- 怎么看 digest？→ `docker buildx imagetools inspect` 或 registry API / inspect RepoDigests。

---

## 二、容器运行原理（中层）

### Q7. `docker run` 时大概发生了什么？

- **难度**: mid
- **标签**: 运行时, 原理

**答案**

简化链路：

1. CLI 请求 dockerd  
2. 本地无镜像则从 Registry pull  
3. 创建容器配置：可写层、挂载、网络 endpoint、环境变量  
4. 经 containerd → **runc** 按 OCI runtime spec：创建 namespaces、加入 cgroup、准备 rootfs（pivot_root）、降权、exec 启动进程  
5. runc 退出后由 shim 守护容器进程生命周期  

**口述模板（30–60 秒）**

1. 不只是「起个进程」，还有隔离与配额装配。  
2. dockerd/containerd/runc 分层协作。  
3. 最终是内核特性在干活。  
4. 可追问：shim 的作用、OCI 标准。

**常见追问**

- 为什么说容器是「进程级虚拟化」？→ 没有独立内核，隔离的是进程视图与资源。

---

### Q8. Namespace 解决什么？常见有哪些？

- **难度**: mid
- **标签**: Namespace

**答案**

Namespace **隔离「看见什么」**：

| Namespace | 隔离内容 |
|-----------|----------|
| pid | 进程树（容器内见自己的 PID 1） |
| net | 网卡、路由、iptables |
| mnt | 挂载点 |
| uts | hostname |
| ipc | System V IPC / POSIX MQ |
| user | UID/GID 映射（增强隔离） |
| cgroup | cgroup 视图（较新） |

**口述模板（30–60 秒）**

1. Namespace 做视图隔离。  
2. 常见 pid/net/mnt/uts/ipc/user。  
3. 和 cgroup「能用多少」互补。  
4. 可追问：`--pid=host` 破坏了什么。

**常见追问**

- 共享 PID namespace 的场景？→ 调试、sidecar 特殊需求；默认不要。

---

### Q9. cgroup 解决什么？`--memory` 超限会怎样？

- **难度**: mid
- **标签**: cgroup, OOM

**答案**

cgroup（Control Group）限制/计量 **CPU、内存、IO、PIDs** 等。  
内存超限时，内核 OOM Killer 可能杀掉容器内进程（常见退出码 **137** = 128+9 SIGKILL）。  
要注意：只设 `--memory` 时 swap 相关行为与 `memory-swap` 设置有关；理解「容器 OOM」与「宿主全局 OOM」都可能发生。

**口述模板（30–60 秒）**

1. cgroup 管配额与计量。  
2. 内存打满常被 SIGKILL，退出码 137。  
3. 要设 limit 并监控。  
4. 可追问：cpu.shares vs cpus 硬限制。

**常见追问**

- `--cpus=1.5` 含义？→ 在调度周期内最多使用约 1.5 核等价算力（实现落在 CFS 配额）。

---

### Q10. 为什么容器里的主进程最好当 PID 1 且正确处理信号？

- **难度**: mid
- **标签**: PID 1, 信号

**答案**

容器内 PID 1 承担特殊责任：回收僵尸进程、处理信号。若 PID 1 是不能转信号的 `sh -c`，`docker stop` 的 SIGTERM 可能无效，最后只能超时 SIGKILL。  
实践：用 **exec 形式 ENTRYPOINT**、或 `--init`（tini）作 init。

**口述模板（30–60 秒）**

1. PID 1 要会处理信号、收尸。  
2. shell 包装常导致停不优雅。  
3. exec 形式或 tini。  
4. 可追问：僵尸进程如何观察。

**常见追问**

- `docker stop` 默认等多久？→ 默认约 10 秒后 kill（可改）。

---

### Q11. 退出码 137 和 139 常见含义？

- **难度**: mid
- **标签**: 排障

**答案**

约定：退出码 = 128 + 信号编号（被信号杀死时）。

- **137**：128+9 → **SIGKILL**，常见 OOM Kill 或 `docker kill`、强制杀  
- **139**：128+11 → **SIGSEGV**，段错误，多是应用/本地库 bug  

不要只背数字，要结合 `dmesg`/宿主日志、容器内存限制、应用栈。

**口述模板（30–60 秒）**

1. 137 多看内存与强杀。  
2. 139 多看崩溃与本地扩展。  
3. 结合宿主 OOM 日志验证。  
4. 可追问：退出码 0/1/2 的区别。

**常见追问**

- 如何确认是 OOM？→ 宿主 `dmesg`/`journalctl` 搜 OOM；看容器 memory limit 与使用量。

---

## 三、网络深入

### Q12. 默认 bridge 与用户自定义 bridge 的关键区别？

- **难度**: mid
- **标签**: bridge, DNS

**答案**

- **默认 bridge（docker0）**：老行为下容器间多用 IP，**无内置 DNS 按容器名解析**（历史 `--link` 已过时）。  
- **用户自定义 bridge**：自带嵌入式 DNS，可用**容器名/服务名**互访；更好的隔离与可控网段；生产/Compose 默认走这类网络。

**口述模板（30–60 秒）**

1. 自定义网络才有好用的名字解析。  
2. 默认 bridge 仅适合凑合演示。  
3. Compose 会自动建项目网络。  
4. 可追问：网络隔离与暴露面。

**常见追问**

- 跨 compose 项目通信？→ 外部网络 external network 共享。

---

### Q13. 端口映射在底层大致怎么实现？

- **难度**: mid
- **标签**: iptables, NAT

**答案**

bridge 模式下容器有独立 net ns 与 veth 连到网桥。`-p 8080:80` 通常在宿主 **iptables/NAT** 做 DNAT，把到宿主 8080 的流量转到容器 IP:80；出站做 MASQUERADE。  
用户态还可能有 **docker-proxy** 协助监听（实现细节随版本变化）。排障时查：容器 IP、`iptables -t nat -L`、防火墙是否拦、是否只绑了 127.0.0.1。

**口述模板（30–60 秒）**

1. 映射本质是宿主转发到容器网栈。  
2. 关键看 NAT 规则与防火墙。  
3. 同网络内容器互访不需要 -p。  
4. 可追问：host 网络为何不用 -p。

**常见追问**

- 为什么容器能访问外网？→ 源地址伪装/NAT 出站。

---

### Q14. host 网络模式优缺点？

- **难度**: mid
- **标签**: host

**答案**

**优点**：无 NAT/端口映射开销，性能与网络调试更直接，端口即宿主端口。  
**缺点**：网络隔离几乎没有；端口冲突；多实例难；安全边界变差。  
Linux 上常用；Docker Desktop（Mac/Win）上 host 语义与真 Linux 不同，勿想当然。

**口述模板（30–60 秒）**

1. host 共用宿主网络命名空间。  
2. 性能好、隔离差。  
3. 仅在明确需要时用。  
4. 可追问：与 `--network=container:` 对比。

**常见追问**

- 微服务都用 host 行吗？→ 通常不行，端口与安全都难管。

---

### Q15. Compose 里 `depends_on` 为什么不够？怎么保证依赖就绪？

- **难度**: mid
- **标签**: Compose, healthcheck

**答案**

裸 `depends_on` 只保证**依赖容器已启动**（被创建/启动），不保证应用端口就绪。  
正确做法：给依赖加 **healthcheck**，并用：

```yaml
depends_on:
  db:
    condition: service_healthy
```

应用侧仍建议重试连接（防御性）。迁移任务可用 `service_completed_successfully`。

**口述模板（30–60 秒）**

1. depends_on ≠ ready。  
2. healthcheck + condition。  
3. 应用重试仍然要。  
4. 可追问：healthcheck 命令怎么写。

**常见追问**

- 健康检查失败会怎样？→ 状态 unhealthy；配合编排决定是否摘流/重启（Compose/K8s 行为不同）。

---

### Q16. 容器 DNS 解析失败怎么排查？

- **难度**: mid
- **标签**: DNS, 排障

**答案**

1. 是否同网络、名字是否服务名/容器名  
2. `docker exec` 里 `getent hosts svc` / `nslookup`（镜像需有工具）  
3. 看 `/etc/resolv.conf` 是否指向嵌入式 DNS（常 127.0.0.11）再转发  
4. 宿主 DNS/公司内网 DNS 是否可达  
5. `network_mode: host` 时行为不同  

自定义 `dns:` 字段可指定。

**口述模板（30–60 秒）**

1. 先确认网络与名字。  
2. 再看 resolv.conf 链路。  
3. 最后查宿主/内网 DNS。  
4. 可追问：跨网络别名 alias。

**常见追问**

- 偶发解析失败？→ 查 conntrack 打满、DNS 超时、并发与重试。

---

## 四、存储

### Q17. Volume、Bind Mount、tmpfs 怎么选？

- **难度**: mid
- **标签**: 存储

**答案**

| 类型 | 特点 | 适用 |
|------|------|------|
| named volume | Docker 管理，可移植 | 生产持久化优先 |
| bind mount | 绑宿主路径 | 开发代码/配置 |
| tmpfs | 内存文件系统 | 临时敏感文件、高性能临时数据 |

容器可写层：临时、难备份、拖垮磁盘与镜像层差量。

**口述模板（30–60 秒）**

1. 持久化用 volume，开发用 bind，临时用 tmpfs。  
2. 别把重要数据只写可写层。  
3. 权限与 UID 映射要设计。  
4. 可追问：NFS/云盘做 volume driver。

**常见追问**

- 删 `docker compose down` 会删卷吗？→ 默认不删；`-v` 才删。

---

### Q18. 为什么说「数据库容器 + overlay 可写层」很危险？

- **难度**: mid
- **标签**: 数据库, 存储

**答案**

数据库需要持久化与可靠 fsync。数据若只在容器层：删容器即丢；且联合文件系统对大量小文件随机写、fsync 路径的性能与语义可能不理想。  
正确：数据目录挂 **volume/块存储**，谈备份、恢复、磁盘性能（IOPS/latency），并理解编排迁移时的数据跟着走还是外置。

**口述模板（30–60 秒）**

1. DB 数据必须外挂。  
2. 还要备份与磁盘性能。  
3. 容器化 DB ≠ 免运维。  
4. 可追问：有状态与无状态工作负载差异。

**常见追问**

- 生产一定要用托管数据库吗？→ 不一定，但要有对等的运维能力。

---

### Q19. 挂载权限问题（Permission denied）常见原因？

- **难度**: mid
- **标签**: 权限

**答案**

- 容器内进程 UID 与卷内文件所有者不一致  
- 只读挂载 `:ro` 却要写  
- SELinux/AppArmor 标签（尤其 Fedora/RHEL 上 `:Z`/`:z`）  
- user namespace 重映射导致宿主 UID 对不上  

修复思路：统一运行 USER、初始化入口改属主、或 CI/镜像里固定 UID（如 1000）。

**口述模板（30–60 秒）**

1. 先对 UID/GID 与读写标志。  
2. 再看安全模块标签。  
3. 用同一 UID 策略贯穿镜像与卷。  
4. 可追问：fsGroup 在 K8s 的类比。

**常见追问**

- 用 root 跑图省事？→ 能跑但不安全；高级面试减分。

---

## 五、日志与可观测

### Q20. 为什么生产要对 `json-file` 做日志轮转？

- **难度**: mid
- **标签**: 日志

**答案**

默认 json-file 可能无限增长，单容器日志即可打满磁盘，进而导致所有容器/宿主异常。  
daemon 或容器级配置 `max-size` / `max-file`，或改用 journald/syslog + 集中式采集（ELK/Loki 等）。应用坚持打 **stdout/stderr**。

**口述模板（30–60 秒）**

1. 不轮转必踩磁盘坑。  
2. 本地轮转 + 集中采集。  
3. 应用别只写死容器内文件。  
4. 可追问：多行日志与 JSON 日志。

**常见追问**

- 如何临时定位最大日志？→ 查 `/var/lib/docker/containers/*/*-json.log` 大小（路径因环境而异）。

---

### Q21. Healthcheck 怎么设计才有用？

- **难度**: mid
- **标签**: healthcheck

**答案**

检查「业务就绪」而非「进程还在」：HTTP `/health`、能连上依赖的浅检查。避免过重（别每次健康检查跑全量迁移）。  
控制 `interval`/`timeout`/`retries`/`start_period`，给启动留宽限期。

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -fsS http://127.0.0.1:8080/health || exit 1
```

无 curl 的镜像用专用小工具或应用自带 health 子命令。

**口述模板（30–60 秒）**

1. 探活要反映就绪。  
2. 设 start_period 避免误杀。  
3. 检查要轻。  
4. 可追问：liveness vs readiness（K8s 概念对照）。

**常见追问**

- 健康检查依赖外部 DB 是否合适？→ 过深会导致 DB 抖动时集体 unhealthy；常做浅检查 + 依赖独立告警。

---

## 六、安全（中级）

### Q22. 为什么不要把业务容器以 root 运行？

- **难度**: mid
- **标签**: 安全

**答案**

容器逃逸或应用漏洞时，root 容器对宿主威胁更大。用 `USER` 非特权用户；配合只读根文件系统、丢掉多余 capabilities。  
注意：非 root 后绑定 <1024 端口需要能力或改监听高端口由反向代理承接。

**口述模板（30–60 秒）**

1. 默认 root 是历史包袱。  
2. USER 非 root 是基线。  
3. 再叠加只读根与能力裁剪。  
4. 可追问：rootless Docker。

**常见追问**

- 官方镜像很多是 root？→ 是，生产要衍生加固或选非 root 变体。

---

### Q23. 为什么说挂载 `docker.sock` 极度危险？

- **难度**: mid
- **标签**: 安全, socket

**答案**

能访问 docker.sock 就能让 dockerd 起特权容器、挂载宿主根目录等，**基本等同宿主 root**。CI「方便构建」常这样干，但要把风险隔离到专用构建节点、权限与网络隔离，而不是塞进每个业务 Pod/容器。

**口述模板（30–60 秒）**

1. sock ≈ 宿主高权限 API。  
2. 业务负载严禁挂。  
3. 构建用专用环境收敛风险。  
4. 可追问：Kaniko/BuildKit remote 等替代。

**常见追问**

- 替代 DinD/sock 的构建方式？→ Kaniko、Buildah、BuildKit daemon 独立部署等。

---

### Q24. 镜像里误打进密钥怎么处理？

- **难度**: mid
- **标签**: 安全, 密钥

**答案**

层是可追溯的：即使后来 `rm` 密钥，**旧层仍可能含秘密**。必须：轮换密钥、重写历史（多阶段/secret mount 重建）、从仓库删除并视为已泄露。  
构建期用 BuildKit `--mount=type=secret`，运行期用环境注入/密钥管理，不把 `.env` COPY 进镜像。

**口述模板（30–60 秒）**

1. 删文件≠删历史层。  
2. 当泄露处理并轮换。  
3. secret mount / 运行时注入。  
4. 可追问：镜像扫描与门禁。

**常见追问**

- ARG 传密码安全吗？→ 仍可能进历史/`docker history`，不如 secret mount。

---

## 七、Compose 与工程化

### Q25. Compose V2 相对 V1 你该知道什么？

- **难度**: mid
- **标签**: Compose

**答案**

- 命令：`docker compose` 插件，V1 `docker-compose` 已不推荐  
- BuildKit 默认参与构建  
- 容器名分隔符等行为差异（可用兼容模式，但不建议依赖容器名）  
- `version` 字段可省略；健康条件 depends_on、profiles、watch 等能力更完善  

**口述模板（30–60 秒）**

1. 用 Compose V2 插件。  
2. 别再写死旧 version 迷信。  
3. 用 health condition、profiles。  
4. 可追问：watch 模式适用开发。

**常见追问**

- Compose 能上生产吗？→ 单机/小规模可以；要集群调度、自愈、多节点通常上 K8s/云编排。

---

### Q26. profiles 有什么用？

- **难度**: mid
- **标签**: Compose

**答案**

给服务打 profile，按需启动，例如仅开发启用 `adminer`、仅调试启用 `proxy-debug`：

```yaml
services:
  adminer:
    image: adminer
    profiles: ["dev"]
```

`docker compose --profile dev up -d`

**口述模板（30–60 秒）**

1. profile 做可选组件开关。  
2. 一套文件多环境裁剪。  
3. 避免每人本地改 YAML。  
4. 可追问：override 文件 `compose.override.yaml`。

**常见追问**

- 和多个 compose 文件 `-f` 比？→ 可组合；override 适合本地覆盖。

---

### Q27. 开发与生产的 Compose/镜像策略如何分开？

- **难度**: mid
- **标签**: 工程化

**答案**

- 开发：bind mount、热重载、debug 端口、宽松命令  
- 生产：不可变镜像、无 bind 业务代码、资源 limit、只读根、健康检查、固定版本  
- 用多文件：`compose.yaml` + `compose.prod.yaml`，或 CI 只部署镜像+编排  

不要把开发用 root+volume 代码挂载直接当生产。

**口述模板（30–60 秒）**

1. 同一镜像产物，不同运行配置。  
2. 开发图快，生产图稳与安全。  
3. 用 override/多文件拆分。  
4. 可追问：十二要素配置。

**常见追问**

- 生产用 `compose up` 还是 Swarm/K8s？→ 看规模与组织能力；面试讲清边界即可。

---

## 八、排障方法论

### Q28. 容器不断重启，怎么系统排查？

- **难度**: mid
- **标签**: 排障

**答案**

1. `docker ps` 看重启次数与状态  
2. `docker inspect` 看 RestartCount、OOMKilled、Error  
3. `logs` 看崩溃栈/配置错误  
4. 暂时去掉 `--restart` 或 `restart: "no"` 便于观察  
5. 查内存 limit、健康检查是否误杀、依赖是否未就绪  
6. 确认入口命令是否立刻退出（后台化错误）  

**口述模板（30–60 秒）**

1. inspect + logs 优先。  
2. 区分 OOM、配置、依赖、命令退出。  
3. 先停重启策略再复现。  
4. 可追问：与 K8s CrashLoopBackOff 对照。

**常见追问**

- unhealthy 导致重启吗？→ Docker 单机默认不会因 unhealthy 自动杀；Swarm/K8s 策略不同。

---

### Q29. 「宿主机能 curl 通，容器内不能」怎么查？

- **难度**: mid
- **标签**: 网络, 排障

**答案**

- 容器 DNS 是否正常  
- 是否走代理环境变量（HTTP_PROXY）在容器未设  
- MTU 问题（尤其 VPN/云覆盖网）  
- 出站被公司防火墙按源地址策略拦截  
- 应用听在 `127.0.0.1` 而不是 `0.0.0.0`（反向：别人连不进你）  

对照：`docker exec` 里访问、宿主访问、看路由与 resolv。

**口述模板（30–60 秒）**

1. 对比宿主与容器网络路径。  
2. DNS/代理/MTU 是三大常见坑。  
3. 确认监听地址。  
4. 可追问：tcpdump 抓包位置（宿主 veth）。

**常见追问**

- 如何看容器 IP 与 veth？→ inspect NetworkSettings；宿主 `ip link`/`veth` 对。

---

### Q30. 镜像架构不匹配（amd64/arm64）会有什么现象？

- **难度**: mid
- **标签**: 多架构

**答案**

Apple Silicon / ARM 服务器上拉到 amd64 镜像可能：  
- 通过模拟运行（慢）  
- 或直接无法运行 / exec format error  

解决：多架构构建 `docker buildx build --platform linux/amd64,linux/arm64`，部署时选对平台；CI 标明目标架构。

**口述模板（30–60 秒）**

1. 镜像有 CPU 架构属性。  
2. 不匹配就慢或起不来。  
3. buildx 出 manifest list。  
4. 可追问：QEMU 模拟的性能代价。

**常见追问**

- `FROM --platform=$BUILDPLATFORM` 何时用？→ 交叉编译场景，区分构建机与目标架构。

---

### Q31. `docker diff` / `docker commit` 适合什么？不适合什么？

- **难度**: mid
- **标签**: 实践

**答案**

- `diff`：看可写层改了哪些文件，排障「谁改脏了容器」  
- `commit`：把可写层打成镜像——**仅应急**，无 Dockerfile 溯源、易含垃圾与密钥  

正规交付永远回到 Dockerfile/CI。

**口述模板（30–60 秒）**

1. diff 用于观察。  
2. commit 只做急救。  
3. 正式产物必须可重建。  
4. 可追问：不可变基础设施。

**常见追问**

- 线上热修了能否 commit 再推？→ 能救急，但要补 PR/Dockerfile 并轮换，否则下次发布冲掉且不可审计。

---

### Q32. 资源限制在 Compose 里写了不生效怎么办？

- **难度**: mid
- **标签**: Compose, 资源

**答案**

排查：Compose 版本是否 V2；字段是否写在正确位置（`deploy.resources` vs 旧版不支持）；是否其实部署到了忽略该字段的环境；Docker Desktop 资源上限是否太小；是否看错容器（scale 出多个）。  
用 `docker inspect` 看 HostConfig.Memory/NanoCpus 是否真正下发。

**口述模板（30–60 秒）**

1. 先 inspect 确认有没有应用到容器。  
2. 再查 Compose 版本与字段。  
3. 最后查 Desktop/宿主总资源。  
4. 可追问：Swarm 与 compose 对 deploy 字段历史差异。

**常见追问**

- K8s 里对应什么？→ requests/limits。

---

### Q33. 如何做「最小权限」容器运行（中级清单）？

- **难度**: mid
- **标签**: 安全

**答案**

- 非 root USER  
- `read_only: true` + 必要 tmpfs/卷  
- `cap_drop: [ALL]` 再按需 `cap_add`  
- `security_opt: [no-new-privileges:true]`  
- 不挂 docker.sock；敏感目录不 bind  
- 固定镜像版本；流水线漏洞扫描门禁  

**口述模板（30–60 秒）**

1. 用户、只读、能力、禁止提权四件套。  
2. 再加上供应链与挂载收敛。  
3. 用清单逐项落地。  
4. 可追问：seccomp/AppArmor 默认配置。

**常见追问**

- 丢掉 ALL 后常见还要加什么？→ 视应用；如 `NET_BIND_SERVICE` 等，按需最小加。

---

### Q34. 本地镜像与仓库镜像同 tag 不一致怎么避免踩坑？

- **难度**: mid
- **标签**: 工程化

**答案**

- 部署前 `docker pull` 明确 digest  
- CI 推送后用 digest 引用  
- 避免人手在节点上「刚好有旧 latest」  
- 镜像拉取策略在 K8s 为 IfNotPresent 时更易脏（对照了解）  

**口述模板（30–60 秒）**

1. 别信任同名 tag。  
2. 部署钉 digest。  
3. 节点缓存可能骗你。  
4. 可追问：滚动更新与镜像拉取策略。

**常见追问**

- 如何强制刷新？→ pull 新 digest 或删本地再拉；编排里设 Always（有代价）。

---

### Q35. 容器时间不对、时区问题怎么处理？

- **难度**: mid
- **标签**: 运维

**答案**

容器默认常 UTC。可：

- 环境变量 `TZ`  
- 挂载宿主 `/etc/localtime`（可移植性一般）  
- 镜像内安装时区数据  

更推荐应用统一用 UTC 存储，展示层转换。

**口述模板（30–60 秒）**

1. 容器时区独立于宿主观感。  
2. TZ 或时区数据包。  
3. 业务更推荐 UTC。  
4. 可追问：NTP 与宿主时间漂移。

**常见追问**

- 日志时间错乱？→ 统一时区与时间戳格式（ISO8601/UTC）。

---

### Q36. 如何理解「只读镜像层 + 可写容器层」（Union FS / overlay）？

- **难度**: mid
- **标签**: overlay2

**答案**

多个只读层与一个可写 upper 层联合成 merged 视图。读：从上到下找文件；写：Copy-on-Write 拷到 upper；删：whiteout 标记。  
好处：层共享省空间；坏处：大量写放大、inode/磁盘耗尽风险。这也是「数据别写容器层」的原理版解释。

**口述模板（30–60 秒）**

1. 只读层共享，可写层独享。  
2. 写时复制，删用 whiteout。  
3. 密集写不适合落在联合 FS 上。  
4. 可追问：overlay2 目录结构 lower/upper/work/merged。

**常见追问**

- 多容器同镜像为何省空间？→ 共享只读层。
