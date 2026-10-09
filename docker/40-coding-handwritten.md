# Docker 手写与现场编码

> 使用方式：先自己写/讲思路，再对照参考要点。面试官看的是取舍与边界，不只是能否背指令。

---

### H1. 生产级多阶段 Dockerfile（Node/TS API）

- **难度**: senior
- **标签**: Dockerfile, Multi-stage
- **面试官看什么**: 缓存顺序、非 root、小镜像、无密钥进层、健康检查

**题干**

为 Node.js（或你熟悉的语言）API 写 Dockerfile：依赖安装可缓存、最终镜像不含 devDependencies 与源码构建工具、非 root 运行、提供 HEALTHCHECK。假设应用监听 `8080`，构建产物在 `dist/`。

**思路口述**

1. deps 阶段：只拷 package 清单装依赖  
2. build 阶段：拷源码编译  
3. runner 阶段：只带 production node_modules + dist  
4. USER、HEALTHCHECK、固定基础镜像 tag  

**参考实现要点**

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
COPY . .
RUN npm run build && npm prune --omit=dev

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -S app && adduser -S app -G app
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package.json ./
USER app
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --start-period=15s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8080/health || exit 1
CMD ["node", "dist/server.js"]
```

说明：若 alpine 无 wget，可换成应用内 health 或安装最小工具；也可用 distroless 变体（调试代价更高）。

**易错点**

- 先 COPY 全目录再 npm ci → 缓存永废  
- 最终阶段仍含 typescript  
- root 运行；健康检查打外部依赖过深  

---

### H2. Go 静态二进制 + Distroless

- **难度**: mid
- **标签**: Dockerfile, Go
- **面试官看什么**: CGO、多阶段、非 root、入口

**题干**

写一个尽可能小的 Go 服务镜像，目标架构 `linux/amd64`。

**思路口述**

1. 构建阶段官方 golang 镜像  
2. `CGO_ENABLED=0` 出静态文件  
3. 运行阶段 distroless/static 或 scratch  
4. 注意时区/证书：访问 HTTPS 需要证书包（distroless 常已带或用适当变体）

**参考实现要点**

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o /out/app ./cmd/app

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

**易错点**

- 需要 cgo 的库却禁用了 CGO  
- scratch 无 CA 证书导致 HTTPS 失败  

---

### H3. `.dockerignore` 清单

- **难度**: junior
- **标签**: build
- **面试官看什么**: 安全与速度意识

**题干**

列出一份通用后端项目的 `.dockerignore`，并说明为什么忽略 `.env`。

**思路口述**

1. 减上下文：git、依赖目录、构建产物、IDE  
2. 防泄密：.env、密钥、kubeconfig  
3. 防脏缓存：本地覆盖文件  

**参考实现要点**

```gitignore
.git
.gitignore
**/node_modules
**/dist
**/.env
**/.env.*
Dockerfile*
compose*.yaml
README.md
.idea
.vscode
coverage
*.log
```

**易错点**

- ignore 掉构建必需的文件  
- 以为不 COPY .env 就安全，却忘了上下文仍可能被其他指令使用  

---

### H4. Compose：API + Postgres + 健康依赖

- **难度**: mid
- **标签**: Compose, healthcheck
- **面试官看什么**: service_healthy、卷、网络、资源限制

**题干**

编写 `compose.yaml`：`api` 依赖 `db` 真正就绪后再启；Postgres 数据持久化；api 映射 `8080`；均设内存限制。

**思路口述**

1. db 写 HEALTHCHECK（pg_isready）  
2. depends_on condition: service_healthy  
3. named volume  
4. 密码用环境变量，勿写死进镜像  

**参考实现要点**

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10
    deploy:
      resources:
        limits:
          memory: 512M

  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://app:${POSTGRES_PASSWORD}@db:5432/app
    depends_on:
      db:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"

volumes:
  pgdata:
```

**易错点**

- 只写 depends_on 无 condition  
- 密码提交进 git；应用无重试仍可能偶发失败  

---

### H5. 只读根文件系统的 Compose 服务

- **难度**: senior
- **标签**: 安全
- **面试官看什么**: read_only + tmpfs/卷分工

**题干**

让服务 `read_only: true`，但应用需要写 `/tmp` 与本地上传目录 `/data/uploads`。

**思路口述**

1. 根只读防篡改  
2. tmpfs 给 /tmp  
3. volume 给上传持久化  
4. 确认应用路径可配  

**参考实现要点**

```yaml
services:
  api:
    image: registry.example.com/api:1.2.3
    read_only: true
    tmpfs:
      - /tmp
    volumes:
      - uploads:/data/uploads
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    user: "1000:1000"

volumes:
  uploads:
```

**易错点**

- 只读后忘了给运行时目录可写导致秒退  
- 上传目录用了 tmpfs 导致重启丢失  

---

### H6. BuildKit secret 安装私有依赖（概念手写）

- **难度**: senior
- **标签**: BuildKit, secret
- **面试官看什么**: 密钥不进层

**题干**

npm 私有仓库需要 token。写出使用 secret mount 的 `RUN` 片段，并说明为何不用 `ARG TOKEN`。

**思路口述**

1. CI 传 secret 给 buildkit  
2. RUN --mount=type=secret 读取  
3. ARG 会进历史，secret 不落层  

**参考实现要点**

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

构建：`docker build --secret id=npmrc,src=$HOME/.npmrc .`

**易错点**

- 把 token echo 进文件且 COPY 进最终镜像  
- 多阶段忘记只在 build 阶段用 secret  

---

### H7. 现场排障：根据线索写命令序列

- **难度**: mid
- **标签**: 排障
- **面试官看什么**: 命令顺序是否专业

**题干**

现象：`web` 容器状态 `Restarting`，映射 `8080:8080`。请写出你排查时会执行的命令序列（按顺序）。

**思路口述**

1. 看状态与退出码  
2. logs/inspect  
3. 停重启策略复现  
4. 资源与端口  

**参考实现要点**

```bash
docker ps -a --filter name=web
docker inspect web --format '{{.State.Status}} {{.State.ExitCode}} OOM={{.State.OOMKilled}} Error={{.State.Error}}'
docker logs --tail 200 web
docker inspect web --format '{{.HostConfig.Memory}} {{.HostConfig.RestartPolicy.Name}}'
# 必要时
docker update --restart=no web && docker start web
ss -lntp | grep 8080   # 宿主端口冲突
docker exec web wget -qO- http://127.0.0.1:8080/health  # 若能短暂起来
```

**易错点**

- 一上来 rm 容器丢掉现场  
- 只看 logs 不看 OOMKilled  

---

### H8. 清理脚本（安全版思维）

- **难度**: mid
- **标签**: 运维脚本
- **面试官看什么**: 是否盲目 prune

**题干**

写一个 bash 思路/伪代码：在生产节点清理磁盘，但**绝不**自动删除 named volume；先报告占用。

**思路口述**

1. df + docker system df  
2. 报最大 json 日志  
3. 清理构建缓存与悬空镜像  
4. volume 只列出不删  

**参考实现要点**

```bash
#!/usr/bin/env bash
set -euo pipefail
df -h
docker system df -v
# 列出大日志（路径示例，按系统调整）
du -h /var/lib/docker/containers/*/*-json.log 2>/dev/null | sort -h | tail -n 20
docker builder prune -f
docker image prune -f
# 明确不执行: docker volume prune / system prune -a --volumes
docker volume ls
```

**易错点**

- `prune --volumes` 误删数据  
- 不先看占用就全清  

---

### H9. 多架构 buildx 命令

- **难度**: mid
- **标签**: buildx
- **面试官看什么**: 平台与推送

**题干**

写出同时构建并推送 `linux/amd64,linux/arm64` 的命令，镜像名 `registry.example.com/team/api:1.4.0`。

**思路口述**

1. 确保 buildx builder 可用  
2. `--platform` 列表  
3. `--push` 直接推索引  

**参考实现要点**

```bash
docker buildx create --use --name multi || docker buildx use multi
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t registry.example.com/team/api:1.4.0 \
  --push .
```

**易错点**

- 只 `--load` 却指定多平台（load 通常单平台）  
- 未登录仓库  

---

### H10. 修复有问题的 Dockerfile（改 bug）

- **难度**: mid
- **标签**: 审查
- **面试官看什么**: 能否指出问题并改

**题干**

指出问题并给出修改版：

```dockerfile
FROM node:latest
COPY . .
RUN npm install
ENV PASSWORD=supersecret
EXPOSE 3000
CMD npm start
```

**思路口述**

1. latest 不钉版本  
2. COPY 顺序打穿缓存  
3. npm install 非 ci、含 dev  
4. 密钥进镜像  
5. shell 形式 CMD  

**参考实现要点**

- 钉 `node:20.11.1-alpine`（示例）  
- 先 copy 锁文件 `npm ci`  
- 密码改运行时注入  
- `CMD ["node","server.js"]`  
- 补 `.dockerignore`、非 root、多阶段  

**易错点**

- 只改一处就说「好了」  

---

### H11. 自定义 bridge 网络手写命令

- **难度**: junior
- **标签**: 网络
- **面试官看什么**: 基础实操

**题干**

不用 Compose，手动创建网络并让 `api` 用主机名访问 `redis`。

**参考实现要点**

```bash
docker network create appnet
docker run -d --name redis --network appnet redis:7-alpine
docker run -d --name api --network appnet -e REDIS_HOST=redis myapi:1.0
```

**易错点**

- 挂在默认 bridge 却期望名字解析  

---

### H12. 日志轮转 daemon.json 片段

- **难度**: mid
- **标签**: 日志
- **面试官看什么**: 生产意识

**题干**

给出 Docker daemon 日志轮转配置片段（json-file）。

**参考实现要点**

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
```

改后需安全重启 dockerd（注意 live-restore/业务影响）。也可在 compose 服务级 `logging` 覆盖。

**易错点**

- 只改新容器、旧容器仍无轮转（服务级需重建）  
