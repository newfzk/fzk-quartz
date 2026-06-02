---
title: Linux 虚拟文件系统 VFS（MOC）
date: 2026-06-02
tags:
  - topic/Linux
  - topic/VFS
  - topic/MOC
status: evergreen
aliases:
  - VFS
  - 虚拟文件系统
  - Virtual File System
---

# Linux 虚拟文件系统 VFS — 总览

## 核心概念

VFS（Virtual File System，虚拟文件系统）是 Linux 内核中的一个抽象层，为上层应用提供统一的文件操作接口，屏蔽底层不同文件系统的差异。

> **一句话本质**：VFS 是一个"翻译官"——应用说 `read()`、`write()`、`open()`，VFS 把这些统一指令转译给 procfs、ext4、tmpfs 等具体实现去执行。应用不需要知道它在和内存还是磁盘打交道。

```
应用层        cat         ls          docker         top
              │           │           │              │
              ▼           ▼           ▼              ▼
系统调用层    open()    read()     write()       stat()
              │           │           │              │
              ▼           ▼           ▼              ▼
   ┌──────────────────────────────────────────────────────┐
   │              VFS 统一抽象接口层                        │
   │   "一切皆文件"——文件、设备、管道、socket、进程信息...   │
   └──────────────────────────────────────────────────────┘
              │       │         │           │
    ┌─────────┘       │         │           └─────────┐
    ▼                 ▼         ▼                     ▼
  procfs           sysfs     devtmpfs               tmpfs
  (进程信息)       (硬件拓扑)  (设备节点)          (内存存储)
    │                 │         │                     │
    ▼                 ▼         ▼                     ▼
 内核task_struct    kobject     硬件                 内存
 动态生成           设备模型   自动管理              临时存储
```

## 目录

```dataview
LIST
FROM "10-Topics"
WHERE contains(tags, "topic/VFS")
SORT file.name ASC
```

## 常见虚拟文件系统一览

| 文件系统 | 挂载点 | 名字含义 | 核心用途 | 原子笔记 |
|---------|--------|---------|---------|---------|
| [[Linux-proc-文件系统\|procfs]] | `/proc` | **Proc**ess | 进程与内核状态信息 | [[Linux-proc-文件系统]] |
| [[Linux-sysfs-文件系统\|sysfs]] | `/sys` | **Sys**tem **f**ile**s**ystem | 硬件设备拓扑与属性 | [[Linux-sysfs-文件系统]] |
| [[Linux-devtmpfs-文件系统\|devtmpfs]] | `/dev` | **Dev**ice **t**e**m**p**o**rary **f**ile**s**ystem | 设备节点自动管理 | [[Linux-devtmpfs-文件系统]] |
| [[Linux-tmpfs-文件系统\|tmpfs]] | `/run`、`/tmp` | **T**e**m**p**o**rary **f**ile**s**ystem | 内存中临时文件 | [[Linux-tmpfs-文件系统]] |
| [[Linux-cgroup-控制组\|cgroup2]] | `/sys/fs/cgroup` | **C**ontrol **group** v2 | 进程资源分组管控 | [[Linux-cgroup-控制组]] |

## 虚拟 vs 真实文件系统

| 特性 | 虚拟文件系统 | 真实文件系统（[[Linux-ext4文件系统\|ext4]] / [[Linux-XFS文件系统\|xfs]]） |
|------|-------------|------------------------|
| 存储位置 | 内存中 | 磁盘分区 |
| 持久化 | 重启消失 | 持久保存 |
| 数据来源 | 内核动态生成 | 用户写入 |
| I/O 操作 | 不涉及磁盘 I/O | 涉及磁盘读写 |
| 典型速度 | 内存级（极快） | 磁盘级（慢 1-2 个数量级） |

## 理解 VFS 的核心：名字故事

VFS 下属的每一个虚拟文件系统，**名字已经说明了它的全部**：

| 文件系统 | 名字拆解 | 记忆辅助 |
|---------|---------|---------|
| **proc** | **Proc**ess | "看进程的"，关注软件（进程/内核统计） |
| **sysfs** | **Sys**tem **f**ile**s**ystem | "看硬件的"，关注硬件（设备拓扑/驱动） |
| **devtmpfs** | **Dev**ice + **T**e**m**p**o**rary **f**ile**s**ystem | "设备+临时"，内核自动创建设备节点 |
| **tmpfs** | **T**e**m**p**o**rary **f**ile**s**ystem | "临时文件"，存内存，重启就没了 |
| **cgroup2** | **C**ontrol **group** v2 | "控制组 v2"，给进程画笼子管资源 |

## 实例感知：一条命令穿越多个 VFS

```bash
# 例子：在终端执行 ls，背后涉及的所有 VFS
ls -la /proc/self/fd/0

# 拆解：
# /proc    → procfs       → 提供 /proc/self/ 进程信息
# /proc/self/fd/0         → 当前进程的文件描述符（符号链接）
# /dev/pts/0              → devtmpfs      → 终端设备节点

# 一个命令，跨越 2 个 VFS 文件系统协作
```

## 安全限制

虚拟文件系统通常挂载时带有 `nosuid`、`nodev`、`noexec` 安全选项，防止恶意利用。详见 [[Linux-挂载选项详解]]。

## 面试要点

- VFS 是[[设计模式-策略模式|策略模式]]在内核中的经典应用：定义统一接口，各文件系统实现具体行为
- `df -h` 看到的虚拟文件系统不占磁盘空间
- 区别 [[Linux-tmpfs-文件系统\|tmpfs]]（内存，可换出到 swap）和 `ramfs`（纯内存，不可换出）
- [[Linux-proc-文件系统\|procfs]] 和 [[Linux-sysfs-文件系统\|sysfs]] 的分工："proc 看进程，sys 看设备"
- [[Linux-cgroup-控制组\|cgroup]] 是 Docker/K8s 容器资源限制的底层机制

## 相关笔记

### 上层概念
- [[Linux-mount挂载机制]] — 文件系统如何挂载到目录树
- [[Linux-挂载选项详解]] — nosuid、nodev、noexec 等安全选项
- [[Linux-文件描述符fd详解]] — VFS 之上应用看到的 fd 抽象

### VFS 内核组件
- [[Linux-Dentry目录项详解]] — 目录项缓存
- [[Linux-Inode详解]] — 文件元数据

### 真实文件系统对比
- [[Linux-ext4文件系统]]
- [[Linux-XFS文件系统]]

### 面试实践
- [[柠檬微趣-笔试-Q2-mount命令输出解释]] — 面试题：解释 mount 输出中的虚拟文件系统