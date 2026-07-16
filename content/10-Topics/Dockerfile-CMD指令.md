---
title: Dockerfile — CMD 指令
date: 2026-07-01
updated: 2026-07-01
tags:
  - topic/Docker
  - topic/容器
status: to-review
---

## 概述

`CMD`（Command）指令指定容器启动时的**默认命令**。它提供两种功能角色：

1. 单独使用时：提供容器启动时执行的**默认命令**
2. 配合 `ENTRYPOINT` 使用时：为 `ENTRYPOINT` 提供**默认参数**

`CMD` 会在 `docker run` 时被命令行参数**覆盖**。

---

## 三种形式

### 1. Exec 形式（推荐）

```dockerfile
CMD ["executable", "param1", "param2"]
```

- 直接执行可执行文件，不经过 shell
- 信号处理正确
- 详见 [[Docker-Exec形式与Shell形式]]

### 2. Shell 形式

```dockerfile
CMD executable param1 param2
```

- 通过 `/bin/sh -c` 执行
- 支持环境变量展开、管道等 shell 特性
- 信号穿透存在问题

### 3. 作为 ENTRYPOINT 的默认参数

```dockerfile
# 可执行部分在 ENTRYPOINT 中定义
ENTRYPOINT ["nginx"]

# CMD 仅提供默认参数
CMD ["-g", "daemon off;"]
```

此时 `CMD` 不包含可执行文件，仅提供参数列表。

---

## 核心行为

### 可覆盖性

`CMD` 定义的命令在 `docker run` 时可以被命令行参数覆盖：

```bash
# Dockerfile 中：CMD ["echo", "Hello"]
docker run myimage                    # 输出：Hello
docker run myimage echo Hi           # 输出：Hi（CMD 被覆盖）
docker run myimage ls -l             # 输出：当前目录列表（CMD 被覆盖）
```

### 只生效最后一个

Dockerfile 中若有多个 `CMD`，**只有最后一个生效**：

```dockerfile
FROM ubuntu
CMD ["echo", "first"]
CMD ["echo", "last"]       # ✅ 实际生效
```

### 与 ENTRYPOINT 的配合

| 配置 | 行为 |
|:-----|:-----|
| 仅有 `CMD` | `CMD` 作为默认命令，`docker run` 可覆盖 |
| 仅有 `ENTRYPOINT` | `ENTRYPOINT` 作为唯一命令，`docker run` 参数追加到命令后 |
| `ENTRYPOINT` + `CMD` | `CMD` 作为 `ENTRYPOINT` 的默认参数，`docker run` 参数覆盖 `CMD` |

详见 [[Dockerfile-ENTRYPOINT指令#CMD 配合模式]]。

---

## 三种应用模式

### 模式一：单独使用 CMD

适合简单镜像，不固定可执行文件：

```dockerfile
FROM alpine
CMD ["echo", "Hello Docker"]
```

用户可完全覆盖启动命令。

### 模式二：CMD 作为 ENTRYPOINT 的默认参数

适合固定主命令、允许调整参数：

```dockerfile
FROM nginx
ENTRYPOINT ["nginx"]                     # 固定主命令
CMD ["-g", "daemon off;"]                # 默认参数
```

```bash
docker run myimage                       # nginx -g "daemon off;"
docker run myimage -c /custom.conf       # nginx -c /custom.conf（覆盖 CMD）
```

### 模式三：结合 Shell 脚本的包装模式

```dockerfile
COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["--help"]                           # 默认显示帮助
```

这是许多官方镜像（如 MySQL、Redis）使用的模式。

---

## 面试要点

- `CMD` 和 `RUN` 的区别：`RUN` 在**构建阶段**执行，`CMD` 在**容器运行时**执行
- 多个 `CMD` 只有最后一个生效（Dockerfile 层叠机制）
- `CMD` 的 exec 形式必须使用**双引号**，单引号会被当作 shell 形式处理
- `CMD ["nginx", "-g", "daemon off;"]` 为什么用 daemon off？因为容器需要前台进程保持运行
- `docker run` 传递的参数只覆盖 `CMD`，不覆盖 `ENTRYPOINT`

---

## 相关笔记

- [[Docker-Exec形式与Shell形式]] — exec form vs shell form 详解
- [[Dockerfile-ENTRYPOINT指令]] — ENTRYPOINT 指令详解
- [[Docker网络模式-bridge]] — Docker 网络基础
