---
title: Linux tmpfs 文件系统
date: 2026-06-02
aliases:
  - tmpfs
  - 临时文件系统
  - 内存文件系统
  - 临时文件
tags:
  - topic/Linux
  - topic/Linux/文件系统
status: to-review
---

# Linux tmpfs 文件系统

## 名字由来：Temporary Filesystem

**tmpfs** 的名字就是 **Temporary Filesystem**（临时文件系统）的缩写。

| 部分 | 含义 |
|------|------|
| **tmp** | Temporary（临时的） |
| **fs** | Filesystem（文件系统） |

> 名字已经告诉你它的全部特性：**临时**（重启就消失）+ **文件系统**（像文件一样操作）。

## 核心本质

tmpfs 将文件存储在**内存（RAM）**中，而非磁盘上：

```
传统文件系统：      数据 → 磁盘（持久化，但慢）
tmpfs：            数据 → 内存（速度快，但重启消失）

      ┌─────────────────────┐
      │       tmpfs         │
      │                     │
      │  ┌───┐ ┌───┐ ┌───┐ │        物理内存（RAM）
      │  │a  │ │b  │ │c  │ │─────→  ┌───┬───┬───┬───┐
      │  └───┘ └───┘ └───┘ │        │ a │ b │   │ c │
      │  文件/目录          │        └───┴───┴───┴───┘
      └─────────────────────┘
            │
            ▼
      ❌ 重启后全部消失
```

## 动态大小特性

tmpfs 最大的特点是**动态分配**——它不像磁盘分区有固定大小：

```bash
# 挂载一个 tmpfs，指定最大 100MB
mount -t tmpfs -o size=100M mytmpfs /mnt/tmp

df -h /mnt/tmp
# 输出: tmpfs   100M     0   100M   0% /mnt/tmp

# 复制一个 1MB 的文件进去（现在只用了 1MB）
dd if=/dev/zero of=/mnt/tmp/test1 bs=1M count=1

df -h /mnt/tmp
# 输出: tmpfs   100M    1M    99M   1% /mnt/tmp
#        ↑大小没变      ↑已用空间只增加了 1MB

# 复制 50MB 进去
dd if=/dev/zero of=/mnt/tmp/test2 bs=1M count=50

df -h /mnt/tmp
# 输出: tmpfs   100M   51M    49M  51% /mnt/tmp
```

> **想象一个气球**：你往里面吹气（写入数据），它就胀大；把气放掉（删除数据），它就缩小。`size=100M` 不是"固定大小"，而是"最多能吹到 100M"。

## 系统默认使用的 tmpfs

```bash
# 你的系统实际上已经挂载了好几个 tmpfs
df -h | grep tmpfs
# 输出:
# tmpfs   7.8G  2.1M  7.8G  /run         ← 运行时数据（PID 文件、socket）
# tmpfs    16G  2.5G   14G  /dev/shm     ← 共享内存（进程间通信）
# tmpfs   7.8G   80K  7.8G  /tmp         ← 临时文件

# 为什么用 tmpfs 挂载 /tmp？
# 1. 内存速度远快于磁盘
# 2. 临时文件本来就不需要持久化
# 3. 减少磁盘 I/O，延长 SSD 寿命
```

## tmpfs vs 普通磁盘 vs ramfs

| 特性 | tmpfs | ext4（磁盘） | ramfs |
|------|-------|-------------|-------|
| **存储介质** | 内存（RAM） | 磁盘 | 内存 |
| **持久化** | 重启消失 | 持久保存 | 重启消失 |
| **大小限制** | 可设置上限（如 `size=100M`） | 固定分区大小 | **无上限**（可能耗尽所有内存） |
| **换出到 swap** | ✅ 可以（内存紧张时） | ❌ | ❌ **不能换出** |
| **速度** | 快 | 慢 | 最快 |

> **tmpfs vs ramfs 的关键区别**：
> - tmpfs 有容量上限，超出后可以换出到 swap —— 安全
> - ramfs 无上限，可以一直写直到**系统 OOM 崩溃** —— 危险
> 所以日常场景都用 tmpfs，几乎不用 ramfs。

## 经典例子

### 1. 加速编译和构建

```bash
# 把编译临时文件放到 tmpfs，大幅提升速度
mount -t tmpfs -o size=2G tmpfs /tmp

# 在 /tmp 下编译大型项目
cd /tmp
git clone huge-project
cd huge-project
make -j8
# 编译过程的文件读写都在内存中，速度极快
```

### 2. 测试磁盘性能的基线对比

```bash
# 测试 tmpfs（内存速度）
mount -t tmpfs -o size=1G tmpfs /mnt/tmp
dd if=/dev/zero of=/mnt/tmp/test bs=1M count=500 2>&1

# 测试 ext4（磁盘速度）
dd if=/dev/zero of=/home/user/test bs=1M count=500 2>&1

# 对比结果，tmpfs 通常快 5-50 倍
```

### 3. 敏感数据的安全处理

```bash
# 敏感数据只在内存中处理，不留磁盘痕迹
mount -t tmpfs -o size=10M tmpfs /mnt/secure
# 处理完敏感文件...
umount /mnt/secure
# 所有数据彻底消失，无法恢复！
```

### 4. 实战：tmpfs 耗尽会发生什么？

```bash
mount -t tmpfs -o size=100M tmpfs /mnt/tmp

# 试图写入 200MB（超过限制）
dd if=/dev/zero of=/mnt/tmp/bigfile bs=1M count=200
# 输出: dd: error writing '/mnt/tmp/bigfile': No space left on device
# 系统安全地拒绝写入，不会崩溃
```

## 相关笔记

- [[Linux-虚拟文件系统VFS]] — VFS 抽象层，tmpfs 是具体实现之一
- [[Linux-devtmpfs-文件系统]] — 同为"tmp"家族的设备文件系统
- [[Linux-挂载选项详解]] — tmpfs 挂载时的 size、mode 等选项
