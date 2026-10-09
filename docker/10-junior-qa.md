# Docker 初级 Q&A（能用）

> 目标：术语准确、常用命令顺手。答案偏「是什么 / 怎么用」。

---

## 一、基本概念

### Q1. 什么是 Docker？

- **难度**: junior
- **标签**: 概念

**答案**

Docker 是一套用于**构建、分发、运行**容器化应用的平台与工具链。核心是把应用及其依赖打成**镜像（Image）**，再以**容器（Container）**形式在宿主机上隔离运行。日常接触最多的是 Docker Engine（守护进程 dockerd）和 CLI。

**口述模板（30–60 秒）**

1. 一句话结论：Docker 用镜像打包应用，用容器隔离运行。  
2. 关键机制：CLI → dockerd → 拉镜像/建容器 → 进程在隔离环境里跑。  
3. 对比：比虚拟机更轻，共享宿主机内核。  
4. 可追问：和 containerd/runc 的关系可再说一层。

**常见追问**

- Docker 和 Docker Desktop 区别？→ Desktop 是桌面发行版（含 VM/GUI）；服务器上常用 Engine。  
- 必须用 Linux 吗？→ 容器本质跑 Linux；Windows/Mac 上通过虚拟机提供 Linux 内核。

---

### Q2. 镜像、容器、仓库分别是什么？

- **难度**: junior
- **标签**: Image, Container, Registry

**答案**

- **Image**：只读模板，含文件系统分层和应用元数据（默认命令、环境变量等）。  
- **Container**：镜像的运行实例；在镜像只读层上加一层**可写层**，再配上网络、资源限制等。  
- **Registry**：存放与分发镜像的服务，如 Docker Hub、Harbor、云厂商镜像仓库。

一个镜像可启动多个容器；容器删了，未持久化的可写层数据会丢。

**口述模板（30–60 秒）**

1. 镜像是只读模板，容器是运行实例，仓库是分发中心。  
2. 启动 = 镜像层 + 可写层 + 运行配置。  
3. 注意：容器内写的数据默认不持久。  
4. 可追问：tag、digest、manifest 再展开。

**常见追问**

- 镜像 ID 和 tag？→ tag 是可变指针；生产更稳用 digest（sha256）。

---

### Q3. 容器和虚拟机有什么区别？

- **难度**: junior
- **标签**: 对比, VM

**答案**

| | 容器 | 虚拟机 |
|--|------|--------|
| 内核 | 共享宿主机 | 每台 Guest OS 自有内核 |
| 启动 | 秒级（起进程） | 通常更慢 |
| 隔离 | Namespace + cgroup 等 | 硬件虚拟化，隔离更强 |
| 体积 | 通常更小 | 更大 |

容器不是「轻量级虚拟机」的完整替代：隔离边界不同，内核漏洞影响面不同。

**口述模板（30–60 秒）**

1. 容器共享内核，VM 自带 Guest OS。  
2. 所以容器更轻、更快、密度更高。  
3. 代价是隔离弱于 VM，安全要额外加固。  
4. 可追问：Kata/ Firecracker 等强隔离方案。

**常见追问**

- 能否在容器里再跑 Docker？→ 可以但危险（尤其挂 docker.sock）；生产谨慎，常用 DinD 或独立构建节点。

---

### Q4. Docker 的典型应用场景有哪些？

- **难度**: junior
- **标签**: 场景

**答案**

- 开发环境一致：「在我机器上能跑」问题  
- CI 构建与测试环境标准化  
- 微服务拆分部署与依赖打包  
- 快速试用中间件（Redis、MySQL、Nginx）  
- 作为 K8s 等编排的运行时镜像载体  

**口述模板（30–60 秒）**

1. 核心价值是环境一致性与交付标准化。  
2. 开发、CI、部署一条链用同一镜像。  
3. 有状态中间件要额外谈存储与运维。  
4. 可追问：和 Serverless/裸机部署对比。

**常见追问**

- 数据库适不适合上容器？→ 可以，但要认真做存储、备份、资源与运维；很多团队数据库仍用托管或专用集群。

---

## 二、安装与架构直觉

### Q5. `docker version` / `docker info` 能看出什么？

- **难度**: junior
- **标签**: 命令

**答案**

- `docker version`：Client / Server（Engine）版本是否匹配、API 版本。  
- `docker info`：存储驱动（如 overlay2）、运行时、cgroup 驱动、镜像与容器数量、Registry 配置等——排障第一步常看这里。

**口述模板（30–60 秒）**

1. version 看版本，info 看宿主与 daemon 配置。  
2. 排障先确认存储驱动、是否 rootless、资源概况。  
3. 客户端连不上 daemon 时两个命令都会失败。  
4. 可追问：context、远程 Docker host。

**常见追问**

- Client 和 Server 版本差很多行不行？→ 可能不兼容；尽量对齐大版本。

---

### Q6. Docker 客户端和守护进程是什么关系？

- **难度**: junior
- **标签**: 架构

**答案**

CLI（`docker`）通过 API（默认 Unix socket ` /var/run/docker.sock`，或 TCP）与 **dockerd** 通信。真正拉镜像、建容器由守护进程协调；底层再交给 containerd / runc 等。  
**拥有 docker.sock ≈ 接近宿主机 root 权限**，不要随便挂进不可信容器。

**口述模板（30–60 秒）**

1. CLI 只是客户端，干活的是 dockerd。  
2. 默认走本地 socket。  
3. socket 权限等于高危权限。  
4. 可追问：rootless、TLS 远程守护进程。

**常见追问**

- 为什么 CI 里挂 docker.sock 有风险？→ 容器内可控制宿主机 Docker，等价提权。

---

## 三、镜像操作

### Q7. 常用镜像命令有哪些？

- **难度**: junior
- **标签**: 命令, Image

**答案**

```bash
docker pull nginx:1.27          # 拉取
docker images                   # 列表（或 docker image ls）
docker rmi <image>              # 删除镜像
docker tag a:b registry/a:b     # 打标签
docker push registry/a:b        # 推送
docker history <image>          # 看分层历史
docker inspect <image>          # 元数据
```

搜索：`docker search`（用得少，更多直接查文档/Harbor）。

**口述模板（30–60 秒）**

1. 增删查：pull / images / rmi。  
2. 分发：tag + push。  
3. 排查：history / inspect。  
4. 可追问：悬空镜像 dangling、prune。

**常见追问**

- `docker image prune` 删什么？→ 未使用的虚悬镜像等；加 `-a` 更激进，生产慎用。

---

### Q8. 镜像 tag 里的 `latest` 是什么意思？有什么坑？

- **难度**: junior
- **标签**: tag

**答案**

`latest` 只是一个**普通标签名**，不保证「最新稳定版」，且会被维护者移动指向。生产应使用**明确版本**（如 `1.27.2`）或 **digest**，避免不可复现构建与静默升级。

**口述模板（30–60 秒）**

1. latest 不是魔法，只是默认 tag 名。  
2. 生产锁定版本或 digest。  
3. 否则同一份 Dockerfile 今天和明天可能不同。  
4. 可追问：语义化版本与滚动升级策略。

**常见追问**

- 不写 tag 会怎样？→ 默认当 `latest` 处理。

---

### Q9. 如何查看镜像里有哪些环境变量、默认启动命令？

- **难度**: junior
- **标签**: inspect, CMD, ENTRYPOINT

**答案**

```bash
docker image inspect nginx --format '{{.Config.Env}}'
docker image inspect nginx --format '{{.Config.Cmd}} {{.Config.Entrypoint}}'
# 或
docker inspect nginx
```

也可用 `docker run --rm nginx env` 看运行时环境（取决于镜像）。

**口述模板（30–60 秒）**

1. 用 inspect 看 Config 里的 Env/Cmd/Entrypoint。  
2. 运行时 `-e` / `--env-file` 可覆盖或追加。  
3. CMD 与 ENTRYPOINT 组合决定最终进程。  
4. 可追问：exec 形式与 shell 形式区别。

**常见追问**

- `docker run nginx bash` 发生了什么？→ 若 ENTRYPOINT 是 nginx，参数可能变成 nginx 的参数；要覆盖需 `--entrypoint`。

---

## 四、容器生命周期

### Q10. 容器常见状态有哪些？常用生命周期命令？

- **难度**: junior
- **标签**: 命令, 生命周期

**答案**

常见状态：created、running、paused、exited、dead 等。

```bash
docker run -d --name web nginx:1.27   # 创建并后台运行
docker ps / docker ps -a                # 运行中 / 全部
docker stop web / docker kill web       # 优雅停 / 强杀
docker start web / docker restart web
docker rm web / docker rm -f web
docker pause web / docker unpause web
docker logs -f --tail 100 web
docker exec -it web sh                  # 进容器（镜像需有 shell）
```

**口述模板（30–60 秒）**

1. run/start/stop/rm 管生命周期。  
2. ps/logs/exec/inspect 管观察与调试。  
3. stop 先发信号再超时强杀。  
4. 可追问：重启策略 on-failure/unless-stopped。

**常见追问**

- `docker run` 和 `create`+`start`？→ run ≈ create + start；create 只创建不启动。

---

### Q11. `-d`、`-it`、`--name`、`--rm` 分别做什么？

- **难度**: junior
- **标签**: run 参数

**答案**

- `-d`：detached 后台运行  
- `-i`：保持 STDIN 打开；`-t`：分配伪终端；交互常用 `-it`  
- `--name`：指定容器名（便于引用）  
- `--rm`：容器退出后自动删除（适合一次性任务）

**口述模板（30–60 秒）**

1. -d 后台，-it 交互，--name 起名，--rm 用完即焚。  
2. 调试常用 `docker run -it --rm ... sh`。  
3. 生产服务多用 -d + 重启策略，而不是 --rm。  
4. 可追问：TTY 对日志/信号的影响。

**常见追问**

- 后台容器能不能再挂 TTY？→ 一般服务不需要；要用 `docker exec -it`。

---

### Q12. 端口映射 `-p` 怎么用？`-P` 呢？

- **难度**: junior
- **标签**: 网络, 端口

**答案**

```bash
docker run -d -p 8080:80 nginx    # 宿主机 8080 -> 容器 80
docker run -d -p 127.0.0.1:8080:80 nginx  # 仅本机访问
docker run -d -P nginx            # 随机映射 EXPOSE 的端口
```

格式：`hostPort:containerPort`，还可指定协议 `-p 53:53/udp`。

**口述模板（30–60 秒）**

1. -p 把容器端口暴露到宿主。  
2. 左边宿主，右边容器。  
3. -P 随机高端口，适合临时演示。  
4. 可追问：和宿主机防火墙/安全组的关系。

**常见追问**

- 容器内监听 80，没 -p 外面访问不到？→ bridge 模式下默认如此；同网络内容器可用 IP/服务名互访。

---

### Q13. 如何把环境变量和文件传进容器？

- **难度**: junior
- **标签**: 环境变量, 挂载

**答案**

```bash
docker run -e ENV=prod -e DEBUG=0 nginx
docker run --env-file ./env.list nginx
docker run -v /host/path:/container/path:ro nginx   # 绑定挂载只读
docker run -v mydata:/var/lib/mysql mysql           # 命名卷
```

密钥不要写进镜像层；构建期密钥用 BuildKit secret（中级再讲）。

**口述模板（30–60 秒）**

1. -e / --env-file 传配置。  
2. -v/-mount 传文件或持久化。  
3. 密钥与镜像分层隔离。  
4. 可追问：env 在 compose 里的写法。

**常见追问**

- 环境变量和配置文件哪个更好？→ 十二要素偏环境变量；大块配置常用挂载或配置中心。

---

### Q14. `docker logs` 常用技巧？

- **难度**: junior
- **标签**: 日志

**答案**

```bash
docker logs web
docker logs -f --tail 200 web
docker logs --since 30m web
docker logs --timestamps web
```

默认常见驱动是 `json-file`，日志落在宿主机；不轮转可能把磁盘写满。

**口述模板（30–60 秒）**

1. logs 看标准输出/错误。  
2. -f / --tail / --since 是日常三件套。  
3. 生产要配轮转或集中采集。  
4. 可追问：应用把日志只写文件不写 stdout 怎么办。

**常见追问**

- 容器删了日志还在吗？→ 默认随容器；要持久需卷或集中日志。

---

### Q15. `docker exec` 和 `docker attach` 区别？

- **难度**: junior
- **标签**: 调试

**答案**

- **exec**：在运行中的容器里**再起一个进程**（如 `sh`），最常用。  
- **attach**：接到容器**主进程**的标准输入输出；主进程退出容器也停；误操作风险更高。

调试优先 `exec`。

**口述模板（30–60 秒）**

1. exec 开新进程，attach 挂主进程。  
2. 日常排障用 exec。  
3. attach 要小心 Ctrl 序列和退出行为。  
4. 可追问：没有 shell 的 distroless 镜像怎么调试。

**常见追问**

- distroless/scratch 没有 shell？→ 用临时 sidecar、`docker create`+拷贝、或 ephemeral debug 容器（高级实践）。

---

## 五、Dockerfile 入门

### Q16. Dockerfile 是什么？最常见指令有哪些？

- **难度**: junior
- **标签**: Dockerfile

**答案**

Dockerfile 是构建镜像的文本脚本。常见指令：

| 指令 | 作用 |
|------|------|
| `FROM` | 基础镜像 |
| `WORKDIR` | 工作目录 |
| `COPY` / `ADD` | 拷贝文件（一般优先 COPY） |
| `RUN` | 构建时执行命令 |
| `ENV` / `ARG` | 环境变量 / 构建参数 |
| `EXPOSE` | 声明端口（文档性，不自动映射） |
| `CMD` / `ENTRYPOINT` | 默认启动 |
| `USER` | 切换用户 |
| `VOLUME` | 声明卷挂载点 |

构建：`docker build -t myapp:1.0 .`

**口述模板（30–60 秒）**

1. Dockerfile 描述怎么把上下文打成镜像。  
2. FROM → 装依赖 → COPY 代码 → CMD。  
3. EXPOSE 不是 -p。  
4. 可追问：每条指令一层与缓存。

**常见追问**

- ADD 和 COPY？→ COPY 更明确；ADD 多 tar 自动解压、URL 等能力，易踩坑。

---

### Q17. `CMD` 和 `ENTRYPOINT` 有什么区别？

- **难度**: junior
- **标签**: CMD, ENTRYPOINT

**答案**

- **ENTRYPOINT**：容器的「主命令」，较固定。  
- **CMD**：默认参数；可被 `docker run` 后的参数覆盖。  

常见组合：`ENTRYPOINT ["nginx"]` + `CMD ["-g","daemon off;"]`。  
推荐 **exec 形式** JSON 数组，信号能正确传给进程。

**口述模板（30–60 秒）**

1. ENTRYPOINT 定命令，CMD 定默认参数。  
2. run 后面的参数默认替换 CMD。  
3. 用 exec 形式避免变成 shell 子进程。  
4. 可追问：`--entrypoint` 覆盖。

**常见追问**

- shell 形式有什么问题？→ PID 1 可能是 `sh`，信号与退出码行为怪异。

---

### Q18. 构建上下文（build context）和 `.dockerignore` 是什么？

- **难度**: junior
- **标签**: build

**答案**

`docker build` 会把指定目录（常见 `.`）作为上下文发给 daemon。上下文太大 → 构建慢、易误 COPY 进密钥/node_modules。  

用 `.dockerignore` 排除：`.git`、`node_modules`、测试产物、`.env` 等（类似 `.gitignore`）。

**口述模板（30–60 秒）**

1. 上下文是发送给 daemon 的文件集合。  
2. 无关文件用 .dockerignore 丢掉。  
3. 既加速又防泄密。  
4. 可追问：为什么不要 `COPY . .` 却不 ignore。

**常见追问**

- Dockerfile 必须在上下文根目录吗？→ 可用 `-f` 指定路径，但上下文目录仍要选对。

---

### Q19. 写一个最简单的 Nginx 静态站 Dockerfile 长什么样？

- **难度**: junior
- **标签**: Dockerfile, 示例

**答案**

```dockerfile
FROM nginx:1.27-alpine
COPY ./html /usr/share/nginx/html
EXPOSE 80
# 官方镜像已有合适 CMD，通常无需再写
```

构建运行：

```bash
docker build -t mysite:1.0 .
docker run -d -p 8080:80 mysite:1.0
```

**口述模板（30–60 秒）**

1. 选小基础镜像，COPY 静态文件。  
2. 用官方 nginx 的默认启动。  
3. 映射端口验证。  
4. 可追问：如何加健康检查、非 root。

**常见追问**

- 为什么用 alpine？→ 体积小；注意 musl 与 glibc 兼容差异。

---

## 六、数据卷入门

### Q20. 什么是 Volume？和往容器里直接写有何不同？

- **难度**: junior
- **标签**: Volume

**答案**

容器可写层随容器删除而丢失（除非 commit，生产不这么干）。  
**Volume** 由 Docker 管理的持久化存储，生命周期可独立于容器，适合数据库数据目录、上传文件等。

```bash
docker volume create mydata
docker run -v mydata:/data alpine
docker volume ls
docker volume inspect mydata
```

**口述模板（30–60 秒）**

1. 容器层数据默认不持久。  
2. Volume 把数据放在 Docker 管理的存储里。  
3. 删容器不一定删卷（看命令是否 -v）。  
4. 可追问：bind mount 对比。

**常见追问**

- `docker rm -v`？→ 会尝试删容器匿名卷。

---

### Q21. Bind Mount 是什么？开发时为什么常用？

- **难度**: junior
- **标签**: Bind Mount

**答案**

Bind Mount 把**宿主机某一路径**直接挂进容器，如 `-v $(pwd):/app`。改本地代码容器内立刻可见，适合开发热更新。  
缺点：依赖宿主路径，可移植性差；权限与安全要小心（别挂整个根目录）。

**口述模板（30–60 秒）**

1. Bind 是挂宿主目录。  
2. 开发方便，生产慎用。  
3. 生产持久化更偏向命名 Volume 或外部存储。  
4. 可追问：:ro 只读挂载。

**常见追问**

- Windows/Mac 路径挂载慢？→ 文件同步经过 VM，大项目要注意性能。

---

## 七、网络与 Compose 入门

### Q22. Docker 默认有哪些网络模式？

- **难度**: junior
- **标签**: 网络

**答案**

常见：

- **bridge**：默认；容器在 docker0 网桥后，出网做 NAT  
- **host**：共用宿主机网络命名空间（无独立 IP/端口映射概念）  
- **none**：无网络  
- **container:<id>**：共用另一容器网络命名空间  

自定义 bridge：`docker network create mynet` 后 `--network mynet`。

**口述模板（30–60 秒）**

1. 默认 bridge，另有 host/none。  
2. 自定义网络便于服务发现。  
3. host 性能好但隔离弱。  
4. 可追问：overlay 跨主机。

**常见追问**

- `docker network ls` / `inspect`？→ 看网络、容器 IP、网关。

---

### Q23. 两个容器怎么互相访问？

- **难度**: junior
- **标签**: 网络, 服务发现

**答案**

1. 放在**同一自定义 bridge 网络**，用**容器名**当主机名（内置 DNS）。  
2. 或用 `--link`（已过时，勿在新项目用）。  
3. 映射到宿主端口后通过 `host:port`（耦合宿主，不推荐作服务间调用）。

Compose 里同一 compose 项目默认同网络，用**服务名**访问。

**口述模板（30–60 秒）**

1. 同自定义网络 + 名字 DNS。  
2. Compose 用服务名。  
3. 别依赖过时的 link。  
4. 可追问：默认 bridge 为何不推荐。

**常见追问**

- 默认 bridge 能用名字解析吗？→ 老默认 bridge 对用户容器 DNS 不如用户定义网络好用；生产用自定义网络。

---

### Q24. Docker Compose 是做什么的？常用命令？

- **难度**: junior
- **标签**: Compose

**答案**

用 YAML 定义多容器应用（服务、网络、卷），一次拉起整套依赖。Compose V2：`docker compose`。

```bash
docker compose up -d
docker compose ps
docker compose logs -f api
docker compose exec api sh
docker compose down          # 停并删容器/网络（卷默认保留）
docker compose down -v       # 连卷也删（慎用）
```

现代文件常叫 `compose.yaml`，可省略 `version` 字段。

**口述模板（30–60 秒）**

1. Compose 声明式管理多容器。  
2. up/ps/logs/exec/down 日常够用。  
3. 用 V2 插件命令。  
4. 可追问：与 K8s 的边界。

**常见追问**

- `docker-compose` 带连字符？→ V1 独立项目，已淘汰趋势；新环境用 `docker compose`。

---

### Q25. 最小 `compose.yaml` 长什么样？

- **难度**: junior
- **标签**: Compose, 示例

**答案**

```yaml
services:
  web:
    image: nginx:1.27-alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
  api:
    build: ./api
    environment:
      - DB_HOST=db
    depends_on:
      - db
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: example
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

注意：仅 `depends_on` **不保证**数据库已可接受连接（中级用 healthcheck）。

**口述模板（30–60 秒）**

1. services 定义多容器。  
2. 网络默认打通，用服务名互联。  
3. 命名卷持久化 DB。  
4. 可追问：健康检查与启动顺序。

**常见追问**

- build 和 image 同时写？→ 可：本地 build，image 指定打出的名字/标签。

---

## 八、资源、清理与排障入门

### Q26. 如何限制容器 CPU 和内存？

- **难度**: junior
- **标签**: cgroup, 资源

**答案**

```bash
docker run -d --memory=512m --cpus=1.5 nginx
```

Compose 中常用：

```yaml
deploy:
  resources:
    limits:
      cpus: "1.5"
      memory: 512M
```

（Compose 对 `deploy.resources` 的支持在 V2 已较完善，以你使用的版本文档为准。）

**口述模板（30–60 秒）**

1. --memory / --cpus 落到 cgroup。  
2. 防止单容器拖死宿主。  
3. 超限可能 OOM Kill。  
4. 可追问：cpu-shares 与 cpus 区别。

**常见追问**

- 不设 limit 会怎样？→ 默认可抢占大量宿主资源。

---

### Q27. 磁盘被 Docker 吃满了，初级怎么清理？

- **难度**: junior
- **标签**: 清理, 磁盘

**答案**

```bash
docker system df              # 看占用
docker system prune           # 清未用网络/停止容器/虚悬镜像等
docker system prune -a        # 更狠：含未用镜像（慎）
docker volume prune           # 未使用卷（更慎）
docker builder prune          # 构建缓存
```

先 `df`/`docker system df` 再决定，**生产禁止无确认地 prune -a/-v**。

**口述模板（30–60 秒）**

1. 先 df 看是镜像、容器层还是卷/日志。  
2. prune 分级别，先温和。  
3. 卷和构建缓存要单独评估。  
4. 可追问：json-file 日志轮转。

**常见追问**

- 如何看某容器可写层大小？→ `docker ps -s` 或 inspect / 系统工具。

---

### Q28. 容器启动立刻退出，第一步查什么？

- **难度**: junior
- **标签**: 排障

**答案**

1. `docker ps -a` 看 STATUS / 退出码  
2. `docker logs <容器>`  
3. 核对启动命令、WORKDIR、配置文件是否存在  
4. 交互复现：`docker run -it --rm 同样镜像 sh` 手动跑命令  
5. 看是否缺环境变量、端口冲突、架构不对（arm/amd64）

**口述模板（30–60 秒）**

1. 状态 → 日志 → 命令 → 环境。  
2. 用临时交互容器复现。  
3. 退出码给线索（1 通用错误，137 常和 OOM/SIGKILL 相关）。  
4. 可追问：137/139 含义。

**常见追问**

- 前台能跑后台不行？→ 看进程是否 daemon 化退出；容器要前台挂住主进程。

---

### Q29. 什么是重启策略（restart policy）？

- **难度**: junior
- **标签**: 稳定性

**答案**

```bash
docker run -d --restart=unless-stopped nginx
```

常见：`no`（默认）、`on-failure`、`always`、`unless-stopped`。  
用于进程崩溃后自动拉起；**不能替代**健康检查与依赖就绪。

**口述模板（30–60 秒）**

1. 重启策略管进程挂了怎么办。  
2. 服务常用 unless-stopped/always。  
3. 配不好会疯狂重启打日志。  
4. 可追问：和 healthcheck 组合。

**常见追问**

- 宿主重启后容器会起来吗？→ `always`/`unless-stopped` 会随 Docker 启动而拉起（daemon 启动时）。

---

### Q30. `EXPOSE` 和 `-p` 有什么区别？

- **难度**: junior
- **标签**: 网络, Dockerfile

**答案**

- `EXPOSE`：镜像元数据**声明**意图端口，文档/编排工具可读；**不会**在宿主打开端口。  
- `-p` / Compose `ports`：真正做端口发布/映射。

**口述模板（30–60 秒）**

1. EXPOSE 是声明，-p 才是映射。  
2. 只有 EXPOSE 外面仍访问不到。  
3. -P 会参考 EXPOSE 做随机映射。  
4. 可追问：发布端口与「容器间互访」无关。

**常见追问**

- 不 EXPOSE 能不能 -p？→ 能。

---

### Q31. 如何进入正在运行的容器改配置？（以及为什么生产不该靠这个）

- **难度**: junior
- **标签**: 实践

**答案**

```bash
docker exec -it web sh
# 或 bash
```

可以临时改文件做验证，但**可写层改动**在重建容器后消失；正确做法是改镜像/挂载配置/改编排后重建。生产排障可进容器看，但要用持久化手段固化修复。

**口述模板（30–60 秒）**

1. exec 进去看可以。  
2. 在可写层「修好」不等于交付修好。  
3. 最终要回到镜像与编排变更。  
4. 可追问：不可变基础设施 Immutable Infrastructure。

**常见追问**

- `docker commit` 救火？→ 能出临时镜像，但不可追溯、易脏；只作应急。

---

### Q32. 镜像仓库登录与推送怎么做？

- **难度**: junior
- **标签**: Registry

**答案**

```bash
docker login registry.example.com
docker tag myapp:1.0 registry.example.com/team/myapp:1.0
docker push registry.example.com/team/myapp:1.0
docker pull registry.example.com/team/myapp:1.0
```

私有仓库还要配证书/ insecure-registries（仅内网临时，有安全代价）。

**口述模板（30–60 秒）**

1. login → tag → push。  
2. 命名含仓库域名/项目路径。  
3. CI 用机器人账号与权限最小化。  
4. 可追问：Harbor 项目与扫描。

**常见追问**

- 推送被拒？→ 看权限、标签命名、配额、HTTPS 证书。

---

### Q33. Alpine、Debian slim、Distroless 基础镜像怎么选（入门级）？

- **难度**: junior
- **标签**: 基础镜像

**答案**

- **alpine**：小，包管理 apk；注意 musl。  
- **debian/ubuntu slim**：兼容性好，体积中等。  
- **distroless/scratch**：极小、无壳，攻击面小，调试难，适合编译型静态二进制。  

入门先保证「能跑、能排障」，再追求极致缩小。

**口述模板（30–60 秒）**

1. 按语言生态与安全要求选。  
2. 小镜像不等于没漏洞。  
3. 调试需求影响选择。  
4. 可追问：多阶段构建最终用 distroless。

**常见追问**

- 为什么 alpine 上跑某些 Go/glibc 应用出问题？→ C 库差异。

---

### Q34. `docker cp` 做什么用？

- **难度**: junior
- **标签**: 命令

**答案**

在宿主与容器间拷贝文件：

```bash
docker cp web:/var/log/app.log ./app.log
docker cp ./config.json web:/etc/app/config.json
```

适合应急取日志/塞配置；长期仍应挂载或打进镜像。

**口述模板（30–60 秒）**

1. cp 做双向文件拷贝。  
2. 应急好用。  
3. 不是配置管理的正规手段。  
4. 可追问：容器只读根时 cp 进系统路径可能失败。

**常见追问**

- 能 cp 正在写的数据库文件吗？→ 有一致性风险，应用官方备份工具。
