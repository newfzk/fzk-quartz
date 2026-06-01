---
title: lsof 文件诊断工具
date: 2026-05-29
tags:
  - topic/Linux
  - topic/故障排查
  - topic/磁盘文件管理
status: to-review
aliases:
  - lsof
  - List Open Files
---

# lsof 文件诊断工具

`lsof`（List Open Files）是 Linux 下用于列出当前系统打开文件的工具。在 Linux 中，"一切皆文件"——普通文件、目录、网络套接字、管道、设备等都被视为文件，因此 `lsof` 是系统诊断的利器。

## 常用命令速查

| 用途              | 命令                             | 辅助记忆                                                                                                   |
| --------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------ |
| 查找已删除但仍被进程持有的文件 | `lsof +L1 /挂载点`                | `+LN` 代表 [[Linux文件链接计数-link-count\|link count]] < N，此处为`link count < 1`，即`link count = 0` 即已被删除但仍被进程占用 |
| 全局搜索已删除的文件      | `lsof -nP \| grep '(deleted)'` |                                                                                                        |
| 查看某端口被哪个进程占用    | `lsof -i :端口号`                 |                                                                                                        |
| 查看所有网络连接        | `lsof -i`                      |                                                                                                        |
| 查看某用户打开的文件      | `lsof -u 用户名`                  |                                                                                                        |
| 查看某进程打开的文件      | `lsof -p PID`                  |                                                                                                        |
| 查看某命令打开的文件      | `lsof -c 命令名`                  |                                                                                                        |
| 递归查看目录下被打开的文件   | `lsof +D /路径`                  |                                                                                                        |
| 查看某文件被哪些进程打开    | `lsof /path/to/file`           |                                                                                                        |

## 常见用法详解

### 1. 查找已删除但仍被进程持有的文件

这是排查磁盘空间不释放问题的核心用法：

```bash
# 查看挂载点下所有 link count = 0 的文件
lsof +L1 /挂载点路径

# 全局搜索所有已删除的文件
lsof -nP | grep '(deleted)'
```

`+L1` 表示筛选 [[Linux文件链接计数-link-count|link count]] < 1 的文件，即已被删除但仍有进程持有句柄的文件。

### 2. 网络诊断：查看端口占用

```bash
# 查看 8080 端口被哪个进程占用
lsof -i :8080

# 查看所有 TCP 监听端口
lsof -iTCP -sTCP:LISTEN

# 查看所有 UDP 连接
lsof -iUDP
```

### 3. 按进程筛选

```bash
# 查看 PID 为 12345 的进程打开的所有文件
lsof -p 12345

# 查看某个命令（如 nginx）打开的文件
lsof -c nginx

# 组合：查看 nginx 进程的网络连接
lsof -c nginx -i
```

### 4. 按用户筛选

```bash
# 查看某个用户打开的所有文件
lsof -u root

# 排除某个用户
lsof -u ^root
```

### 5. 按文件/目录筛选

```bash
# 查看某个文件被哪些进程打开
lsof /var/log/syslog

# 递归查看某个目录下被打开的文件
lsof +D /var/log
```

### 6. 查看特定类型的文件

```bash
# 查看所有打开的正则文件（REG）
lsof -a -d 0-999 -c bash

# 查看所有 IPv4 网络连接
lsof -i4

# 查看所有 IPv6 网络连接
lsof -i6
```

## 输出解读

```
COMMAND   PID   USER   FD   TYPE  DEVICE   SIZE   NODE   NAME
dmserver 12345 dmdba   11w   REG   dm-0    100G  654321 /path/to/file.log (deleted)
```

| 字段 | 含义 |
|------|------|
| COMMAND | 进程名称（如 `dmserver`、`nginx`） |
| PID | 进程 ID |
| USER | 运行进程的用户 |
| FD | 文件描述符编号和模式（`w` 写、`r` 读、`u` 读写） |
| TYPE | 文件类型（`REG` 普通文件、`DIR` 目录、`IPv4` 网络等） |
| DEVICE | 设备编号 |
| SIZE | 文件大小（若已删除则显示仍占用的空间） |
| NODE | inode 节点号 |
| NAME | 文件路径（后跟 `(deleted)` 表示已删除） |

## 常用参数说明

| 参数 | 说明 |
|------|------|
| `-n` | 禁止 DNS 反向解析（加快速度） |
| `-P` | 禁止端口号到服务名的转换（加快速度） |
| `-i` | 显示网络连接 |
| `-u` | 按用户筛选 |
| `-p` | 按 PID 筛选 |
| `-c` | 按命令名筛选 |
| `+D` | 递归搜索目录 |
| `+L1` | 显示 [[Linux文件链接计数-link-count|link count]] < 1 的文件（已删除） |
| `-a` | AND 组合多个条件（默认是 OR） |

## 无 lsof 时的备用方案：/proc 文件系统

当系统中未安装 `lsof` 时，可通过 `/proc` 文件系统达到类似效果：

```bash
# 查看所有进程的文件描述符，筛选标记为 (deleted) 的项
ls -la /proc/*/fd/ 2>/dev/null | grep '(deleted)'

# 查看某进程的所有打开文件描述符
ls -la /proc/PID/fd/

# 查看某进程打开的某个文件内容（即使文件已被删除，只要句柄存在即可读取）
cat /proc/PID/fd/文件描述符编号
```

> [!tip] /proc 的妙用
> 即使文件已被删除，通过 `/proc/PID/fd/FD` 仍可读取文件内容，这在恢复日志等场景中非常有用。

## 相关笔记

- [[Linux进程持有已删除文件句柄导致磁盘空间不释放|进程持有已删除文件句柄导致磁盘空间不释放]]
- [[Linux文件链接计数-link-count|Linux 文件链接计数（Link Count）]] — 理解 `+L1` 背后的 link count 机制
- [[Linux-Inode详解|Inode 详解]] — `NODE` 字段输出的 inode 编号详解
- [[Linux-文件描述符fd详解|文件描述符（fd）详解]] — `FD` 字段输出的文件描述符详解
