---
title: Dockerfile — 多阶段构建
tags:
  - topic/Docker
  - topic/容器
status: to-review
---

## 核心思想

`FROM ... AS builder` 创建中间阶段，最终镜像只 `COPY --from=builder` 取编译产物——**编译器、源码、build 缓存全都不带到最终镜像**。

```dockerfile
# Stage 1: build
FROM golang:1.22-alpine AS builder
# ... 编译 ...

# Stage 2: runtime
FROM scratch
COPY --from=builder /app /app
```

---

## Go 示例：1.2GB → 12MB

**普通版（单阶段）**：1.2GB（带整个 Go 工具链）

```dockerfile
FROM golang:1.22
WORKDIR /src
COPY . .
RUN go build -o app
CMD ["/src/app"]
```

**优化版（多阶段 + scratch）**：12MB

```dockerfile
# Stage 1: build
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app

# Stage 2: runtime
FROM scratch
COPY --from=builder /app /app
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/app"]
```

`-ldflags="-s -w"` 去掉调试信息再瘦 30%。

---

## Node 示例：950MB → 180MB

**普通版**：950MB

```dockerfile
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
CMD ["node", "dist/server.js"]
```

**优化版**：180MB

```dockerfile
# Stage 1: build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --production

# Stage 2: runtime
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

关键点：
- `npm ci` 比 `npm install` 严格、可复现
- `npm prune --production` 删除 devDependencies
- 多阶段隔离开发依赖

---

## Java 示例：720MB → 200MB

**普通版**：720MB

```dockerfile
FROM eclipse-temurin:17-jdk
COPY . .
RUN ./mvnw package
CMD ["java", "-jar", "target/app.jar"]
```

**优化版（多阶段 + JRE + Spring Boot layertools）**：200MB

```dockerfile
# Stage 1: build
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /build
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline
COPY src src
RUN ./mvnw clean package -DskipTests
RUN java -Djarmode=layertools -jar target/*.jar extract

# Stage 2: runtime
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /build/dependencies/ ./
COPY --from=builder /build/spring-boot-loader/ ./
COPY --from=builder /build/snapshot-dependencies/ ./
COPY --from=builder /build/application/ ./
RUN addgroup -S app && adduser -S app -G app
USER app
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

关键点：
- JDK 镜像编译，JRE 镜像运行（JRE 比 JDK 小 100MB+）
- Spring Boot 的 `layertools` 把 jar 拆成依赖层 + 应用层，依赖不变时层能复用
- 进一步瘦：用 `jlink` 定制 JRE（只含用到的模块），能再砍一半
