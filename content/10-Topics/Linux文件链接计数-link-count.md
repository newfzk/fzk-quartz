---
title: Linux 文件链接计数（Link Count）
date: 2026-06-01
aliases:
  - link count
  - 链接计数
  - 硬链接计数
  - nlink
tags:
  - topic/Linux
  - topic/Linux/文件系统
status: to-review
---

# Linux 文件链接计数（Link Count）

## 什么是 Link Count？

Link Count（链接计数，又称 `nlink`）是 Linux 文件系统中 inode 的一个字段，记录**有多少个目录项（dentry）指向该 inode**。

```bash
# 查看文件的 link count
[[stat-命令详解|stat]] 文件名
# 输出中 "Links: N" 即为 link count
```

| Link Count | 含义 |
|---|---|
| ≥ 1 | 有文件名指向该 inode（文件"活着"） |
| = 0 | 无目录项指向该 inode（文件已被删除） |

## Link Count 的典型变化

### 创建文件

```
创建一个新文件 →
  文件名 → inode #100
  link count = 1
```

```bash
echo hello > test.txt
[[stat-命令详解|stat]] test.txt   # Links: 1
```

### 创建硬链接

硬链接实质上是**创建新的目录项指向同一个 inode**，link count 递增：

```
ln test.txt link.txt →
  test.txt → inode #100
  link.txt → inode #100    ← 同一 inode
  link count = 2
```

```bash
ln test.txt hardlink.txt
[[stat-命令详解|stat]] test.txt    # Links: 2  （两个文件名都返回 2）
```

> [!note] 硬链接的限制
> - 只能在同一文件系统内创建（不能跨分区）
> - 不能对目录创建硬链接（`.` 和 `..` 除外，由内核自动管理）
> - 区别于符号链接（symlink），符号链接有自己的独立 inode

### 删除文件（unlink）

删除文件本质是 `unlink()` 系统调用——移除目录项，link count 减 1：

```
rm test.txt →
  link.txt → inode #100
  link count = 1   ← 减 1，但还有 link.txt 引用，数据保留
```

```bash
rm test.txt
cat hardlink.txt   # 内容还在！
[[stat-命令详解|stat]] hardlink.txt  # Links: 1
```

只有当 **link count 归零** 时，内核才回收 inode 和数据块。

## 特殊场景：link count = 0 但进程仍持有

这是磁盘空间不泄露排查中最关键的知识点：

```
1. 进程打开文件 open() → 内核额外增加 inode 引用
   link count = 1（从目录项）
   引用计数 = 2（目录项 + 进程句柄）

2. rm file.log（unlink）→ 移除目录项
   link count = 0
   引用计数 = 1（只剩进程句柄）→ inode 保留，数据块不释放！

3. 进程 close() 或进程终止 → 引用计数归零
   内核回收 inode 和数据块 → 空间释放 ✓
```

简而言之：**link count 决定文件名是否可见，引用计数决定数据是否释放**。

要查找这类文件，使用 [[lsof-文件诊断工具|lsof]] 的 `+L1` 参数：

```bash
lsof +L1 /挂载点
```

`+L1` 的含义正是：**筛选 link count < 1（即 = 0）的文件**。

## 相关概念对比

| 概念                                  | 作用                          | 关联                      |
| ----------------------------------- | --------------------------- | ----------------------- |
| **Link Count**                      | 指向同一 inode 的目录项数量           | 文件被删除时归零                |
| [[Linux-Inode详解\|Inode]]            | 存储文件的元信息（权限、大小、位置）          | link count 是 inode 的字段  |
| **[[Linux-Dentry目录项详解\|Dentry]]**   | 文件名 → inode 的映射关系           | 目录项，unlink 后消失          |
| **[[Linux-文件描述符fd详解\|文件描述符 (fd)]]** | 进程对打开文件的引用                  | 即使 link count=0，fd 仍可读写 |
| **引用计数**                            | inode 活跃引用总数（目录项 + 进程 + 内核） | 归零时才真释放空间               |

## 相关笔记

- [[lsof-文件诊断工具]] — `+L1` 使用场景
- [[Linux进程持有已删除文件句柄导致磁盘空间不释放]] — 磁盘空间排查实战
- [[Linux-Inode详解|Inode 详解]] — 更深入的文件系统元数据结构
- [[stat-命令详解|stat 命令详解]] — 查看 link count、inode 等元信息的标准工具
- [[Linux-Dentry目录项详解|Dentry 目录项详解]] — 文件名与 inode 之间的映射层
- [[Linux-文件描述符fd详解|文件描述符（fd）详解]] — 即使 link count=0，fd 仍可读写
