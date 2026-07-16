---
title: Dockerfile — 基础镜像选型
tags:
  - topic/Docker
  - topic/容器
status: to-review
---

## 各基础镜像对比

| 镜像 | 大小 | 包管理器 | 适合场景 |
|------|------|----------|----------|
| `ubuntu:22.04` | ~77MB | apt | 兼容性最强，体积大 |
| `debian:bookworm-slim` | ~75MB | apt | 类似 Ubuntu 但更精简 |
| `alpine:3.19` | ~7MB | apk | **绝大多数场景首选** |
| `gcr.io/distroless/base` | ~20MB | 无 | 无 shell，最安全 |
| `gcr.io/distroless/static` | ~2MB | 无 | 仅静态二进制 |
| `scratch` | 0 | 无 | 只能跑静态二进制 |

**FROM 必须钉具体版本**，不能写 `latest`。

```dockerfile
# ❌ 不钉版本
FROM node

# ❌ 只钉大版本
FROM node:20

# ✅ 钉小版本，可复现
FROM node:20.11.1-alpine3.19
```

---

## Alpine 的坑（必须了解）

Alpine 使用 **musl libc**，不是大多数发行版的 glibc，行为差异：

- **DNS 解析行为不同**：musl 不支持 `/etc/hosts` 某些复杂场景，超时处理也不一样。**容器里 DNS 偶发慢/失败时，第一个怀疑 musl**
- **Java 早期版本**：Java 8/11 早期 Alpine 镜像有兼容问题，`eclipse-temurin:17-alpine` 已稳定
- **Go cgo**：Go 默认静态编译没事，开了 cgo（用到 sqlite、某些 C 库）必须用 `apk add gcompat` 或换 debian-slim
- **Python 编译慢**：很多 pip 包没有 musl 的预编译 wheel，要在容器里现编译，build 巨慢——这种情况用 `python:3-slim`（Debian）更好

---

## Distroless

Google 出品的「无 shell」镜像，连 `sh` 都没有，只有应用运行所需的最小依赖。

- **优点**：镜像极小（20-50MB），攻击面小，几乎没有可被 RCE 的工具
- **缺点**：无法 `docker exec ... sh` 调试——需用 `gcr.io/distroless/xxx:debug` tag（带 busybox shell）或 K8s `kubectl debug` 临时挂调试容器

---

## Scratch

完全空白镜像，**只能放静态编译的二进制**。Go 是绝配：

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app

FROM scratch
COPY --from=builder /app /app
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/app"]
```

注意：
- `CGO_ENABLED=0` 关闭 cgo 才能真静态编译
- SSL 根证书要手动拷过去（scratch 啥都没有）

---

## 选型建议

- **Web 服务 / Go / Node** → Alpine
- **Python / Java 老版本 / 有 C 扩展** → Debian-slim
- **纯 Go 静态二进制** → scratch
