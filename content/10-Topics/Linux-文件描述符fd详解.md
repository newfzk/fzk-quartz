---
title: Linux 文件描述符（fd）详解
date: 2026-06-01
tags:
  - topic/Linux
  - topic/文件系统
  - topic/进程管理
status: evergreen
aliases:
  - fd
  - 文件描述符
  - file descriptor
  - 文件句柄
  - 进程文件描述符表
---

# Linux 文件描述符（fd）详解

## 什么是文件描述符？

**文件描述符（File Descriptor，简称 fd）** 是一个非负整数，是进程访问打开文件的**句柄**。它是 Linux "一切皆文件" 哲学的核心实现——进程通过 fd 读写普通文件、网络 socket、管道、设备等所有资源。

> 把进程想象成一个人：**fd = 手里的号码牌**。你不需要记住文件在哪、内容是什么，只要把号码牌递给内核，它帮你操作。

## 三层引用结构

```
进程 A                        内核
┌──────────────┐
│ 文件描述符表   │              ┌──────────────┐       ┌──────────┐       ┌─────────┐
│ ┌──────────┐ │─── fd 3 ───→ │  打开文件表    │──────→ │  dentry  │──────→ │  inode  │
│ │ fd 0:   │ │              │ (系统级)       │       │          │       │  #100   │
│ │ fd 1:   │ │              │ ┌──────────┐  │       │ "a.txt" │       └─────────┘
│ │ fd 2:   │ │              │ │ ref=2     │  │       └──────────┘           ↑
│ │ fd 3: ──┼─┼──────────────┼→│ offset=X  │  │                               │
│ │ ...     │ │              │ │ flags=... │  │                               │
│ └──────────┘ │              │ └──────────┘  │                               │
│              │              └──────────────┘                               │
│              │                                                             │
│ 进程 B       │              ┌──────────────┐       ┌──────────┐            │
│ ┌──────────┐ │─── fd 4 ───→ │ 打开文件表    │──────→ │  dentry  │───────────┘
│ │ fd 4: ──┼─┼──────────────┼→│ (同一个文件)  │       │          │
│ └──────────┘ │              │ ref=1       │       │ "a.txt" │
└──────────────┘              │ offset=Y    │       └──────────┘
                              └──────────────┘
```

| 层次 | 作用域 | 内容 | 关键字段 |
|---|---|---|---|
| **fd 表** | 进程私有 | 整数 → 打开文件表项 | 每个 fd 指向一个 file 结构体 |
| **打开文件表** | 系统全局 | 文件状态信息 | `f_count`（引用计数）、`f_pos`（读写偏移）、`f_flags` |
| **[[Linux-Dentry目录项详解\|dentry]]** | 文件系统 | 文件名 → inode 映射 | `d_name`、`d_inode` |
| **[[Linux-Inode详解\|inode]]** | 文件系统 | 文件元数据 | 权限、大小、[[Linux文件链接计数-link-count\|link count]]、数据块指针 |

> **为什么需要中间这层"打开文件表"？**
> 两个进程打开同一文件，各自持有自己的 fd，但可以共享偏移量（比如父进程 fork 后）或各自独立偏移。打开文件表管理这个"共享状态"。

## 标准文件描述符

所有进程启动时自动获得三个 fd：

| fd | 名称 | 默认指向 | 用途 |
|---|---|---|---|
| **0** | stdin | 终端输入 | 读取键盘/标准输入 |
| **1** | stdout | 终端输出 | 输出普通信息 |
| **2** | stderr | 终端输出 | 输出错误信息 |

```bash
# 重定向本质：改变 fd 指向的目标
echo "hello" > output.txt    # fd 1 → output.txt
./script.sh 2>&1             # fd 2 → fd 1 指向的地方
./read.sh < input.txt        # fd 0 → input.txt
```

## 文件描述符的生命周期

```bash
open("/path/file")  # 返回 fd 3（当前最小可用 fd）
     ↓
read(fd 3, buf, 1024)   # 通过 fd 读取
write(fd 3, buf, 1024)  # 通过 fd 写入
lseek(fd 3, 0, SEEK_SET) # 调整偏移量
     ↓
close(fd 3)  # 释放 fd，打开文件表引用计数 -1
```

### 创建场景

| 系统调用 | 说明 |
|---|---|
| `open()` | 打开文件，返回新 fd |
| `socket()` | 创建网络 socket，返回新 fd |
| `pipe()` | 创建管道，返回两个 fd（读端 + 写端） |
| `dup()` / `dup2()` | 复制已有 fd |
| `accept()` | 接受网络连接，返回新 fd |
| `fcntl(fd, F_DUPFD)` | 复制 fd 到指定编号 |

### 删除场景

| 操作 | 说明 |
|---|---|
| `close(fd)` | 显式关闭 |
| 进程退出（exit / 被 kill） | 内核自动关闭该进程所有 fd |
| `execve()` 后未设置 `FD_CLOEXEC` | 执行新程序时关闭标记了 `CLOEXEC` 的 fd |

## 查看文件描述符

### 查看进程的 fd

```bash
# 查看 PID 为 12345 的进程所有 fd
ls -la /proc/12345/fd/

# 输出示例
lrwx------ 1 user user 64 6月 1 10:00 0 -> /dev/pts/0
lrwx------ 1 user user 64 6月 1 10:00 1 -> /dev/pts/0
lrwx------ 1 user user 64 6月 1 10:00 2 -> /dev/pts/0
lr-x------ 1 user user 64 6月 1 10:00 3 -> /var/log/app.log (deleted)  ← 已删除！
lrwx------ 1 user user 64 6月 1 10:00 4 -> socket:[12345]
```

`/proc/PID/fd/` 中的每个条目都是**符号链接**，指向真实的文件/资源。

### 查看 fd 上限

```bash
# 当前进程的 fd 软限制
ulimit -n

# 查看硬限制
ulimit -H -n

# 系统级 fd 上限
cat /proc/sys/fs/file-max

# 已使用的 fd 数量
cat /proc/sys/fs/file-nr
# 输出：已分配  已使用(空闲)  上限
```

### [[lsof-文件诊断工具|lsof]] 查看 fd

```bash
# 查看某进程的文件描述符
lsof -p 12345

# 查看某文件被哪些进程的哪些 fd 引用
lsof /var/log/syslog
```

## 关键场景：fd 导致磁盘空间不释放

这是 [[Linux进程持有已删除文件句柄导致磁盘空间不释放]] 的核心机制：

```
rm file.log
  → 移除 dentry（link count = 0）
  → 但进程 fd 3 仍指向该文件
  → inode 引用计数 > 0 → 数据块不释放！
```

```bash
# 定位：找到持有 fd 的进程
lsof +L1 /挂载点
# 或
ls -la /proc/*/fd/ 2>/dev/null | grep '(deleted)'

# 定位后：
# 1. 确认 PID
# 2. 查看 /proc/PID/fd/FD → 确认是哪个 fd
# 3. 清空或关闭

# 应急：清空文件内容（不重启进程）
: > /proc/PID/fd/FD
```

## fd 泄漏

当进程不断打开文件/网络连接而不 `close()`，fd 会用完：

```
代码缺陷：
while (true) {
    fd = open("/dev/null", O_RDONLY);  // 每次都分配新 fd
    // 忘记 close(fd)！
}
```

后果：
1. `ulimit -n` 上限耗尽 → `open()` 返回 `EMFILE`（Too many open files）
2. 系统 `/proc/sys/fs/file-max` 耗尽 → 所有进程都受影响

```bash
# 查看某进程当前 fd 数量
ls -la /proc/PID/fd/ | wc -l

# 查看 fd 最多的进程
find /proc/*/fd -maxdepth 1 -type l 2>/dev/null | awk -F/ '{print $3}' | sort | uniq -c | sort -rn | head
```

## 相关笔记

- [[Linux-Dentry目录项详解|Dentry 目录项详解]] — fd 通过 dentry 找到 inode
- [[Linux-Inode详解|Inode 详解]] — inode 引用计数因 fd 而 > 0
- [[Linux文件链接计数-link-count|Linux 文件链接计数（Link Count）]] — link count 归零但 fd 未释放
- [[lsof-文件诊断工具|lsof 文件诊断工具]] — 查看 fd 与文件映射的核心工具
- [[Linux进程持有已删除文件句柄导致磁盘空间不释放]] — fd 导致空间不释放的实战排障
