---
title: Linux Dentry（目录项）详解
date: 2026-06-01
aliases:
  - dentry
  - 目录项
  - directory entry
  - dcache
  - 目录项缓存
tags:
  - topic/Linux
  - topic/Linux/文件系统
status: to-review
---

# Linux Dentry（目录项）详解

## 什么是 Dentry？

**Dentry**（Directory Entry，目录项）是 Linux 文件系统中**文件名与 [[Linux-Inode详解|inode]] 之间的映射关系**。它本质上是一个数据结构，将人类可读的文件名转换为内核可识别的 inode 编号。

> 把文件系统想象成一个图书馆：**dentry = 图书索引卡（书名 → 书架编号）**，**inode = 书架编号对应的书架信息**。

### Dentry 存了什么

| 字段 | 含义 |
|---|---|
| **d_name** | 文件名（如 `test.txt`） |
| **d_parent** | 父目录的 dentry 指针 |
| **d_inode** | 指向关联 [[Linux-Inode详解|inode]] 的指针 |
| **d_op** | dentry 操作函数集 |
| **d_sb** | 所在文件系统的超级块 |

### Dentry 没存什么

- **文件内容** — 那是数据块的事
- **文件元信息（权限/大小/时间戳）** — 那些在 [[Linux-Inode详解|inode]] 里
- **inode 编号本身** — 它存的是指向 inode 结构体的指针

## Dentry 的关键设计：解耦

dentry 的存在实现了 **三层解耦**：

```
用户看到：  文件名          （dentry 管理）
                ↓
内核管理：  inode 编号      （inode 管理）
                ↓
实际存储：  数据块          （block 管理）
```

这种分离带来了两大能力：

### 1. 硬链接：两个 dentry 指向同一个 inode

```bash
ln file_a.txt file_b.txt
```

```
dentry "file_a.txt" ──┐
                       ├──→ inode #100 ──→ 数据块
dentry "file_b.txt" ──┘
```

两个文件名（两个 dentry）→ 同一个 inode → 同一份数据。这就是 [[Linux文件链接计数-link-count|link count = 2]] 的底层原理。

### 2. 删除与句柄分离：删除 dentry 不影响已打开的 fd

```bash
rm file.log  # 移除 dentry（目录项消失）
# 但进程还在读写 → inode 保留 → 空间不释放
```

[[Linux进程持有已删除文件句柄导致磁盘空间不释放|进程持有已删除文件句柄]] 问题的本质正是：**dentry 已删除，但 inode 引用未归零**。

## dcache：缓存 dentry 加速路径解析

Linux 内核维护了一个 **dcache（Directory Entry Cache，目录项缓存）**，用于缓存最近访问的 dentry。

### 为什么需要 dcache？

当你执行 `cat /usr/share/doc/test.txt` 时，内核需要逐级解析路径：

```
/               → 查找根目录的 dentry（已在 dcache）
usr             → 查找 / 下 usr 的 dentry
share           → 查找 usr 下 share 的 dentry
doc             → 查找 share 下 doc 的 dentry
test.txt        → 查找 doc 下 test.txt 的 dentry → 找到 inode
```

如果没有 dcache，每次访问文件都要从磁盘读取目录内容，性能极差。dcache 将解析结果缓存在内存中，下次访问同一路径直接命中。

### 查看 dcache

```bash
# 查看 dentry 缓存统计
slabtop -o | grep dentry

# 或
cat /proc/slabinfo | grep dentry
```

## Dentry 的生命周期

```
创建文件（open / creat）
  → 在目录中添加 dentry（文件名 → inode 映射）
  → dentry 加入 dcache
  → link count + 1

读取文件
  → dcache 查找 dentry
  → 命中：直接获取 inode
  → 未命中：从磁盘读取目录内容，构造 dentry 加入 dcache

删除文件（unlink / rm）
  → 从目录中移除 dentry
  → dcache 中标记为无效
  → link count - 1
  → 若 link count = 0 且无进程引用 → 回收 inode 和数据块

重命名文件（rename / mv）
  → 创建新 dentry（新文件名 → 原 inode）
  → 删除旧 dentry（旧文件名）
  → link count 不变
```

## Dentry vs Inode vs fd 一图流

```
进程                      文件系统
┌──────┐   fd 3    ┌──────────┐   dentry   ┌───────┐  指针   ┌─────────┐
│ 应用  │ ────────→ │ 文件描述符 │ ────────→ │ dentry│ ──────→ │  inode  │ ──→ 数据块
└──────┘            │  表      │            │  "a.txt"│         │  #100  │
                    └──────────┘            └──────────┘         └──────────┘
                                              ↑
                                           目录文件
                                           (磁盘上的目录内容)
```

| 层次 | 作用域 | 创建/删除方式 | 持久性 |
|---|---|---|---|
| **dentry** | 内核 VFS 层 | `open()` / `unlink()` | 内存（dcache）+ 磁盘（目录内容） |
| **inode** | 文件系统层 | `mkfs` 时创建 | 磁盘持久化 |
| **fd** | 进程层 | `open()` / `close()` | 仅进程存活期间 |

## 相关笔记

- [[Linux-Inode详解|Inode 详解]] — dentry 指向的目标，存储文件元信息
- [[Linux文件链接计数-link-count|Linux 文件链接计数（Link Count）]] — link count 就是指向同一 inode 的 dentry 数量
- [[Linux进程持有已删除文件句柄导致磁盘空间不释放]] — dentry 删除但 inode 未释放的排障案例
- [[lsof-文件诊断工具|lsof 文件诊断工具]] — 通过文件句柄反查 dentry 和 inode
