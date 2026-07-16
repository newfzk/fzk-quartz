---
title: Dockerfile — ENTRYPOINT 指令
date: 2026-07-01
updated: 2026-07-01
tags:
  - topic/Docker
  - topic/容器
status: to-review
---

## 概述

`ENTRYPOINT` 指令将容器配置为**可执行程序**——容器启动后固定运行指定的命令，不易被外部覆盖。

与 `CMD` 的区别：
- `CMD` 提供**默认命令**，`docker run` 可覆盖
- `ENTRYPOINT` 提供**固定命令**，需 `--entrypoint` 参数才能覆盖

---

## 两种形式

### Exec 形式（推荐）

```dockerfile
ENTRYPOINT ["executable", "param1", "param2"]
```

- 容器 PID 1 为 `executable` 进程
- 信号处理正确
- 推荐用于生产环境

### Shell 形式

```dockerfile
ENTRYPOINT executable param1 param2
```

- 通过 `/bin/sh -c` 执行
- 容器 PID 1 为 `sh` 进程
- 信号处理存在问题
- **重要**：Shell 形式下 `CMD` 的默认参数功能**失效**

> [!danger] Shell 形式 + Docker stop 陷阱
> 使用 shell 形式的 `ENTRYPOINT` 时，`CMD` 的参数会被**完全忽略**（不会传递给 `sh -c` 启动的命令）。若需要 shell 特性，建议：
> ```dockerfile
> # 不推荐：CMD 参数被忽略
> ENTRYPOINT nginx -g
> CMD ["daemon off;"]       # ❌ 不会生效
>
> # 推荐：exec 形式，需 shell 时手动调用
> ENTRYPOINT ["/bin/sh", "-c", "nginx -g \"$@\"", "--"]
> CMD ["daemon off;"]       # ✅ 生效
> ```

详见 [[Docker-Exec形式与Shell形式]]。

---

## 核心行为

### 不可覆盖性

`ENTRYPOINT` 不会被 `docker run` 命令行参数覆盖：

```bash
# Dockerfile 中：ENTRYPOINT ["echo", "Hello"]
docker run myimage World        # 输出：Hello World（参数被追加）
docker run myimage               # 输出：Hello
```

如需覆盖，必须使用 `--entrypoint` 标志：

```bash
docker run --entrypoint ls myimage -la    # 执行：ls -la
```

### CMD 配合模式

`ENTRYPOINT` + `CMD` 的组合是最常见的 Dockerfile 模式：

```dockerfile
ENTRYPOINT ["nginx"]            # 固定主命令
CMD ["-g", "daemon off;"]       # 默认参数
```

运行时行为：

| `docker run` | 等效命令 |
|:------------|:---------|
| `docker run myimage` | `nginx -g "daemon off;"` |
| `docker run myimage -t` | `nginx -t` |
| `docker run --entrypoint bash myimage` | `bash`（完全覆盖） |

> [!tip] 理解公式
> 最终运行的命令 = `ENTRYPOINT` + `CMD`（或 `docker run` 参数）
>
> 即：**ENTRYPOINT 定义可执行文件，CMD 定义默认参数**

---

## 常见应用模式

### 模式一：固定命令 + 可变参数

```dockerfile
FROM python:3.11
COPY app.py /app/
ENTRYPOINT ["python", "/app/app.py"]
CMD []                    # 无默认参数，但允许 docker run 后追加参数
```

```bash
docker run myimage                     # python /app/app.py
docker run myimage --port 8080         # python /app/app.py --port 8080
```

### 模式二：初始化脚本包装

许多官方镜像使用 `ENTRYPOINT` 指向一个 shell 脚本，在脚本中完成初始化逻辑后再 `exec` 主进程：

```dockerfile
FROM mysql:8.0
COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["mysqld"]
```

```bash
# docker-entrypoint.sh 简化逻辑
#!/bin/bash
# 初始化数据目录、创建用户等...
exec "$@"    # 执行 CMD 传递进来的命令（此处为 mysqld）
```

> [!tip] `exec "$@"` 的技巧
> 包装脚本最后使用 `exec "$@"` 将进程替换为 `CMD` 指定的命令，确保信号能正确传递到目标进程（替换 sh, PID 不变）。
> 详见 [[Docker-Exec形式与Shell形式#对比总结]]。

### 模式三：作为默认命令

```dockerfile
ENTRYPOINT ["ping"]
CMD ["localhost"]
```

```bash
docker run myimage              # ping localhost
docker run myimage google.com   # ping google.com
```

---

## ENTRYPOINT vs CMD 对比

| 对比维度 | `ENTRYPOINT` | `CMD` |
|:---------|:------------|:------|
| **定位** | 固定的可执行文件 | 默认命令 / 默认参数 |
| **覆盖方式** | `--entrypoint` 参数 | `docker run` 命令行参数 |
| **覆盖难度** | ⭐⭐⭐ 难（需显式指定） | ⭐ 易（自动覆盖） |
| **配合关系** | 定义容器最终的可执行文件 | 提供默认参数，可被 docker run 覆盖 |
| **覆盖后行为** | 整个命令被替换 | CMD 被 `docker run` 参数覆盖，追加到 ENTRYPOINT 后 |
| **典型用途** | 固定容器用途（如 nginx、redis） | 提供灵活默认值（如 `--help`） |

---

## 面试要点

- `ENTRYPOINT` 与 `CMD` 的区别及配合关系
- Shell 形式下 `CMD` 参数被忽略的原因：`ENTRYPOINT node app.js` + `CMD ["--port=8080"]` 实际运行的是 `sh -c "node app.js"`，CMD 参数未传递
- 信号穿透问题：为何 shell 形式的 `ENTRYPOINT` 会导致 `docker stop` 无法优雅关闭
- `docker run --entrypoint "" myimage` 可以清空 `ENTRYPOINT`，让 `CMD` 作为主命令
- 官方镜像中 `ENTRYPOINT` 脚本的 `exec "$@"` 模式

---

## 相关笔记

- [[Docker-Exec形式与Shell形式]] — exec form vs shell form 详解
- [[Dockerfile-CMD指令]] — CMD 指令详解
- [[Docker网络模式-bridge]] — Docker 网络基础
