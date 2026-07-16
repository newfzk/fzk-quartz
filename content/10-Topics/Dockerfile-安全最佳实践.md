---
title: Dockerfile — 安全最佳实践
tags:
  - topic/Docker
  - topic/容器
status: to-review
---

## 永远用 non-root 用户

容器默认以 root 运行，存在严重安全风险。应始终创建专用用户：

```dockerfile
# Alpine
RUN addgroup -S app && adduser -S app -G app
USER app

# Debian/Ubuntu
RUN groupadd --system app && useradd --system -g app app
```

文件权限也要对应设置：

```dockerfile
COPY --chown=app:app . /app/
```

---

## K8s securityContext 加固

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true       # 根文件系统只读
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
```

应用要写文件就 emptyDir 挂载 `/tmp`。

---

## 密钥绝不进镜像

```dockerfile
# ❌ 千万别这么干
ENV DB_PASSWORD=xxx

# ❌ 也别这么干
COPY .env /app/

# ✅ 运行时注入
# docker run -e DB_PASSWORD=xxx ...
# 或 K8s Secret 挂载
```

非要在 build 期用敏感参数（拉私有依赖），用 **BuildKit secret**：

```dockerfile
# syntax=docker/dockerfile:1.4
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc ...
```

密钥不会写进任何镜像层，`docker history` 也看不到。

---

## 镜像扫描

```bash
# 扫描 CVE
trivy image myimg:tag

# 集成 CI，高危漏洞自动拦截发布
trivy image --severity HIGH,CRITICAL --exit-code 1 myimg:tag
```

把 trivy 加进 CI pipeline，高危漏洞自动拦截发布。

---

## FROM 钉具体版本

`latest` 会随时变，今天 build 和明天 build 可能完全不一样。**生产 Dockerfile 必须钉到小版本**。

```dockerfile
# ❌ 不钉版本
FROM node

# ✅ 钉具体版本
FROM node:20.11.1-alpine3.19
```
