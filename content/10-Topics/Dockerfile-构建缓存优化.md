---
title: Dockerfile — 构建缓存优化
tags:
  - topic/Docker
  - topic/容器
status: to-review
---

## Docker 镜像分层原理

Docker 镜像由只读层叠加而成，每条 `RUN`、`COPY`、`ADD` 指令都会创建一个新层。**叠加文件系统的特性**：前面层产生的临时文件，即使后面层删除了，磁盘上仍然存在（层不可变）。因此必须**在同一条 `RUN` 内创建 + 清理**。

```dockerfile
# ❌ 三层，每层都留下 apt cache
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget

# ✅ 一层，清完 cache
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl wget && \
    rm -rf /var/lib/apt/lists/*
```

`--no-install-recommends` 跳过 apt 推荐依赖，瘦身利器。

---

## 依赖先 COPY、源码后 COPY

镜像层可复用缓存——前面层没变，后面层直接用缓存。**变动频率低的指令放前面**。

```dockerfile
# ❌ 反例：任何文件改动都会让下面所有层 cache 失效
COPY . .
RUN npm ci
RUN npm run build

# ✅ 正例：依赖文件先 COPY
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build
```

改业务代码 build 时间能从 2 分钟降到 10 秒。Go 的 `go.mod`/`go.sum`、Java 的 `pom.xml` 同理。

---

## .dockerignore 必须写

跟 `.gitignore` 同理——没写的话 `COPY . .` 会把 `node_modules`、`.git`、`.env` 全打包进去，**镜像直接膨胀几百 MB 还泄露密钥**。

```dockerfile
# .dockerignore 通用模板
.git
.gitignore
node_modules
npm-debug.log
.env
.env.*
*.log
coverage
.vscode
.idea
dist
build
target
__pycache__
*.pyc
.DS_Store
README.md
Dockerfile*
docker-compose*
```

---

## RUN --mount=type=cache（BuildKit 缓存挂载）

`--mount=type=cache` 把缓存目录挂载进来，**跨构建共享**——同一台机器多次 build 不用重新下载依赖。

```dockerfile
# syntax=docker/dockerfile:1.4
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production
COPY . .
```

Maven、pip、apt 都能用：

```dockerfile
RUN --mount=type=cache,target=/root/.m2 mvn package
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
RUN --mount=type=cache,target=/var/cache/apt apt-get install ...
```

需要开启 BuildKit（`DOCKER_BUILDKIT=1` 或 Docker 20.10+ 默认开）。

---

## RUN 合并

多条 RUN 合成一条 + 末尾清缓存（不同包管理器对应不同清理命令）：

- **apt**：`rm -rf /var/lib/apt/lists/*`
- **npm**：`npm cache clean --force`
- **pip**：`--no-cache-dir`

---

## 查镜像层大小

```bash
docker history myimg:tag --no-trunc
```

输出每层大小和创建命令，定位是哪一步把镜像撑大的。
