---
title: Docker — Exec 形式与 Shell 形式
date: 2026-07-01
updated: 2026-07-01
tags:
  - topic/Docker
  - topic/容器
status: reviewed
---

## 概述

Dockerfile 中的 `RUN`、`CMD`、`ENTRYPOINT` 指令均支持两种语法形式：**exec 形式**（JSON 数组）和 **shell 形式**（普通字符串）。两者的核心区别在于是否经过 `/bin/sh -c` 进程启动。

## Exec 形式（Exec Form）

### 语法

```dockerfile
# JSON 数组，双引号包裹
RUN ["executable", "param1", "param2"]
CMD ["executable", "param1", "param2"]
ENTRYPOINT ["executable", "param1", "param2"]
```

### 行为

- Docker 直接通过 **`execve()` 系统调用**启动指定进程，**不经过 shell**
- 容器主进程（PID 1）就是数组中的 `executable`
- 不会自动展开环境变量（`$HOME`、`$PATH` 等）

```dockerfile
# ❌ 不会打印环境变量值，而是输出字面量 "$HOME"
CMD ["echo", "$HOME"]

# ✅ 使用 exec 形式且需要环境变量时，手动调用 shell
CMD ["sh", "-c", "echo $HOME"]
```

### 信号处理

exec 形式下，容器主进程直接接收信号（`SIGTERM`、`SIGINT` 等），**信号穿透无损耗**：

```
PID 1 = executable（信号直达）
```

> [!tip] 最佳实践
> exec 形式是 **大多数场景的推荐选择**，尤其是当进程需要正确处理信号时（如 Nginx、Tomcat 等长期运行的服务）。

---

## Shell 形式（Shell Form）

### 语法

```dockerfile
# 普通字符串
RUN executable param1 param2
CMD executable param1 param2
ENTRYPOINT executable param1 param2
```

### 行为

- Docker 隐式调用 `/bin/sh -c "executable param1 param2"`
- 实际容器主进程（PID 1）是 `sh` 进程，而非 `executable`
- 自动支持环境变量展开、管道、通配符等 shell 特性

```dockerfile
# ✅ 自动展开环境变量
CMD echo $HOME

# ✅ 可以使用管道
RUN ps aux | grep nginx

# ✅ 可以使用通配符
RUN cp /tmp/*.log /var/log/
```

### 信号处理

Shell 形式下，shell 进程（`sh`）为主进程，需要 shell 本身支持信号转发：

```
PID 1 = sh -c "executable" → 子进程 executable（信号被 sh 截获）
```

部分 shell 不会将信号转发给子进程，可能导致容器优雅关闭失败。

> [!warning] 陷阱
> `sh -c` 默认不会传播信号。若容器 CMD 使用 shell 形式，执行 `docker stop` 发送 `SIGTERM` 时，**sh 进程不会转发给实际进程**，容器可能被强制 `SIGKILL` 杀掉。
>
> 解决方法：exec 形式 或 shell 形式中使用 `exec` 前缀：
> ```dockerfile
> CMD executable param1    # 等价于 exec 形式
> # 但更好的做法：
> CMD exec executable param1
> ```

---

## 对比总结

| 对比维度 | Exec 形式（JSON 数组） | Shell 形式（字符串） |
|:---------|:---------------------|:--------------------|
| **语法** | `["cmd", "-a"]` | `cmd -a` |
| **启动方式** | `execve()` 直接启动 | `/bin/sh -c "..."` |
| **容器 PID 1** | 目标进程 | `sh` 进程 |
| **环境变量展开** | ❌ 不自动展开 | ✅ 自动展开 |
| **Shell 特性**（管道、通配符） | ❌ 不支持 | ✅ 支持 |
| **信号直达** | ✅ 信号直达目标进程 | ❌ 信号被 sh 截获，需 `exec` 前缀 |
| **资源占用** | 极简（无额外进程） | 多一个 `sh` 进程 |
| **适用场景** | 生产服务、长期运行的进程 | 调试命令、一次性 Shell 操作 |

---

## 典型误区

### 误区一：JSON 数组写了单引号

```dockerfile
# ❌ 错误：CMD 会按字符串处理（shell 形式），不会生效
CMD ['nginx', '-g', 'daemon off;']

# ✅ 正确：必须使用双引号
CMD ["nginx", "-g", "daemon off;"]
```

Docker 将单引号列表解释为 **shell 形式字符串**（而非 JSON 数组），即相当于 `CMD ['nginx', '-g', 'daemon off;']` 被当作一个命令字符串执行，会报错。

### 误区二：在 exec 形式中需要 shell 功能时没有处理

```dockerfile
# ❌ 错误：不会展开 $HOME
CMD ["echo", "$HOME"]

# ✅ 正确：显式调用 shell
CMD ["sh", "-c", "echo $HOME"]
```

---

## 相关笔记

- [[Dockerfile-CMD指令]] — CMD 指令详解
- [[Dockerfile-ENTRYPOINT指令]] — ENTRYPOINT 指令详解
