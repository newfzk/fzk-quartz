---
title: Linux proc 文件系统
date: 2026-06-02
aliases:
  - procfs
  - /proc
  - 进程文件系统
tags:
  - topic/Linux
  - topic/Linux/文件系统
status: to-review
---

# Linux proc 文件系统

## 名字由来：Proc = Process

**procfs** 的名字直接来自 **Process**（进程）。它最初的设计目的就是**暴露内核中进程相关的数据结构**，让用户态通过"读文件"的方式来查看进程和系统状态。

> 把内核想象成一座大楼。procfs 就是大楼的**状态面板**——你想知道 CPU 温度、内存余量、每个房间的人在做什么，看一眼面板就有，不需要挨个敲门去问。

## 核心本质

| 维度 | 说明 |
|------|------|
| **数据来源** | 内核动态生成，**不存在磁盘上** |
| **持久化** | 不持久，重启消失 |
| **访问方式** | 和普通文件一样 `read()` / `write()` |
| **内容** | 进程信息、系统状态、内核参数 |

```c
// 用户态代码视角：读 /proc 和读普通文件完全一样
// 但内核实现完全不同：
// ext4 的 read() → 从磁盘读数据块
// proc 的 read() → 现场查内核变量 → 拼成文字返回
```

## 为什么需要它？

在 procfs 出现之前，获取进程信息需要专门的系统调用或解析 `/dev/kmem`（内核内存设备），非常麻烦。Bruce Perens 提出了"文件系统接口"的想法：**把调试信息以文件形式呈现，用通用的文件操作就可以获取**。

## 经典例子

### 1. 查看进程信息

```bash
# 每个运行中的进程在 /proc 下有自己的目录
ls /proc/          # 一堆数字目录，每个数字 = PID

# 查看当前 shell 的信息
cat /proc/$$/status
# Name:   bash
# Pid:    12345
# PPid:   12300              ← 父进程 PID
# Uid:    1000  1000  1000  1000
# Gid:    1000  1000  1000  1000
# VmSize:  52100 kB          ← 虚拟内存大小
# VmRSS:   3248 kB           ← 物理内存占用（常驻集大小）
# Threads: 1                 ← 线程数
# 
# 注意：这些数据是内核现场从 task_struct 结构体里读出来的！

# 进程环境变量
cat /proc/$$/environ | tr '\0' '\n' | head -5
# 输出你的环境变量（以 \0 分隔，所以需要 tr 转换）

# 进程打开的文件描述符
ls -la /proc/$$/fd/
```

### 2. 查看系统全局状态

```bash
# 系统运行时间
cat /proc/uptime
# 输出: 12345.67 98765.43
#        ↑系统总运行秒数  ↑总空闲秒数（所有 CPU 累加）

# 内存使用（比 free 命令更详细）
cat /proc/meminfo
# MemTotal:       16384120 kB
# MemFree:         8214560 kB
# Buffers:          123456 kB
# Cached:          3456789 kB
# SwapTotal:       2097152 kB
# ...

# CPU 信息
cat /proc/cpuinfo
# processor   : 0
# vendor_id   : GenuineIntel
# model name  : Intel(R) Core(TM) i7-xxxx
# cpu MHz     : 2400.000
# cache size  : 8192 KB
# ...

# 系统负载
cat /proc/loadavg
# 0.15 0.20 0.10 2/345 67890
#  │    │    │    │     └── 最新 PID
#  │    │    │    └────── 运行中/总线程数
#  └────┴────┴────────── 1/5/15 分钟平均负载

# 当前网络连接（类似 netstat）
cat /proc/net/tcp | head
```

### 3. 修改内核参数（谨慎！）

```bash
# /proc 下有些文件是可写的，用于动态调整内核参数
# 注意：这只是临时修改，重启复原

# 开启 IP 转发（临时）
echo 1 > /proc/sys/net/ipv4/ip_forward

# 修改系统最大打开文件数
echo 100000 > /proc/sys/fs/file-max
```

## 实战：用 /proc 排障

```bash
# 场景1：磁盘空间明明够了，但报"磁盘满"
# 可能是一个已删除文件仍被进程持有
ls -la /proc/*/fd/* 2>/dev/null | grep '(deleted)'

# 场景2：找哪个进程占用 CPU 最高
cat /proc/stat | grep cpu
# 然后配合 /proc/PID/stat 分析每个进程

# 场景3：系统突然变慢，看是否内存不足
cat /proc/meminfo | grep -E "^(MemTotal|MemFree|SwapTotal|SwapFree)"
```

## proc 与 sysfs 的职责分工

| 维度 | procfs | sysfs |
|------|--------|-------|
| 主要关注 | **进程**和内核通用统计 | **硬件设备**和驱动 |
| 典型路径 | `/proc/PID/`、`/proc/meminfo` | `/sys/class/`、`/sys/block/` |
| 引入时机 | 早期内核（1992） | 2.6 内核（2003），分担 proc 的硬件信息职责 |

> 早期 `/proc` 什么信息都往里塞（包括硬件信息），后来信息太多太乱，Linux 2.6 推出 [[Linux-sysfs-文件系统\|sysfs]] 专门管理硬件，proc 回归"进程"本职。

## 相关笔记

- [[Linux-虚拟文件系统VFS]] — VFS 抽象层，procfs 是其中的一个具体实现
- [[Linux-sysfs-文件系统]] — 与 proc 互补的硬件信息虚拟文件系统
- [[Linux-文件描述符fd详解]] — `/proc/PID/fd/` 是查看文件描述符的核心入口
