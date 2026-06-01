---
title: Linux Inode 详解
date: 2026-06-01
tags:
  - topic/Linux
  - topic/文件系统
status: evergreen
aliases:
  - inode
  - 索引节点
  - index node
  - inode 节点
---

# Linux Inode 详解

## 什么是 Inode？

**Inode**（Index Node，索引节点）是 Linux/Unix 文件系统中用于存储**文件元信息（metadata）** 的数据结构。每个文件都有一个唯一的 inode。

> 如果把文件比作一本书：**文件名 = 书脊上的书名**，**inode = 书的版权页（作者、页数、出版日期...）**，**数据块 = 书的内容**。

### Inode 存了什么

| 类别 | 字段 |
|---|---|
| **类型与权限** | 文件类型（普通文件/目录/设备...）、`rwx` 权限位 |
| **链接计数** | [[Linux文件链接计数-link-count|link count]]，即有多少个文件名指向此 inode |
| **拥有者** | `uid`、`gid` |
| **时间戳** | `atime`（访问时间）、`mtime`（修改时间）、`ctime`（状态变更时间） |
| **大小** | 文件字节大小 |
| **数据块指针** | 指向实际数据块的地址（直接块 + 间接块） |
| **扩展属性** | ACL、selinux 上下文等 |

### Inode 没存什么

- **文件名** — 文件名存放在**[[Linux-Dentry目录项详解|目录项（dentry）]]** 中，不在 inode 里
- **文件内容** — 内容存放在数据块（data block）中，inode 只存指向它们的指针

> 一个文件名可以不存在 inode 里？是的——正因为这样，才能实现[[Linux文件链接计数-link-count|硬链接]]：多个文件名指向同一个 inode。

## 查看 Inode

```bash
# 查看文件的 inode 编号
ls -i 文件名

# 查看文件的 inode 详细信息
[[stat-命令详解|stat]] 文件名
```

`stat` 输出示例：

```
$ stat test.txt
  文件：test.txt
  大小：1024      块：8          IO 块：4096   普通文件
设备：fd01h/64769d  Inode：654321    硬链接：1
权限：(0644/-rw-r--r--)  Uid：( 1000/  user)   Gid：( 1000/  user)
访问时间：2026-06-01 10:00:00
修改时间：2026-06-01 09:00:00
状态更改时间：2026-06-01 08:00:00
```

其中 `Inode：654321` 就是该文件的 inode 编号，**硬链接：1** 就是 link count。

## 文件名 → Inode → 数据块的映射

```
ls -l 输出中的"文件名"
        │
        ▼  （通过目录项 dentry 查找）
    ┌─────── inode ───────┐
    │ • 权限 / 类型        │
    │ • link count = 1     │
    │ • 数据块指针 ──┼──────────────┐
    │ • 时间戳             │            │
    └──────────────────────┘            ▼
                                   ┌────────┐
                                   │ 数据块  │
                                   │ 实际内容 │
                                   └────────┘
```

**三步走：**
1. 系统从**目录文件**中根据文件名找到对应的 inode 编号
2. 从 inode 表中读取 inode 结构体，获取权限、大小、数据块指针等信息
3. 通过指针读取**数据块**中的实际内容

## Inode 编号与文件系统

```bash
# 查看整个文件系统的 inode 使用情况
df -i

# 示例输出
Filesystem       Inodes  IUsed    IFree IUse% Mounted on
/dev/sda1       655360  654321    1039  100% /
```

**⚠️ 常见陷阱：inode 耗尽**

即使磁盘还有剩余空间，若 **inode 被用光**（`IUse% = 100%`），也无法创建新文件。常见场景：
- 存放大量小文件（如 Git 仓库、缓存目录、邮件队列）
- 每个文件消耗一个 inode，小文件过多耗尽 inode 池

```bash
# 排查哪个目录文件最多
find / -xdev -type f | cut -d/ -f2 | sort | uniq -c | sort -rn | head
```

## Inode 与 Link Count 的关系

[[Linux文件链接计数-link-count|link count]] 存储在 inode 中，表示有多少个目录项（硬链接）指向这个 inode：

```
创建文件 test.txt
  test.txt → inode #100       link count = 1

创建硬链接 link.txt
  test.txt → inode #100       link count = 2  ← 递增
  link.txt → inode #100      

删除 test.txt（unlink）
  link.txt → inode #100       link count = 1  ← 递减
  (inode 和数据还在，因 link count ≠ 0)

删除 link.txt（unlink）
  无文件指向 inode #100       link count = 0  ← 回收 inode 和数据块
```

> 注意：**符号链接（symlink）有自己的独立 inode**，不影响目标文件的 link count。

## Inode 与文件描述符（fd）的关系

当进程 `open()` 一个文件时：
1. 内核通过 inode 找到文件
2. 增加 inode 的**内存引用计数**（不是 link count）
3. 返回文件描述符（fd）给进程

即使 `unlink()` 使 link count = 0，只要进程 fd 未关闭，inode 和数据块就保留：

```
rm file.log（unlink）
  → link count = 0（文件名消失，`ls` 看不到）
  → 引用计数仍 > 0（进程还在读写 fd）
  → inode 和数据块不释放 → 磁盘空间不释放！

进程 close(fd) 或进程终止
  → 引用计数 = 0
  → 内核回收 inode 和数据块 → 空间释放 ✓
```

诊断工具：[[lsof-文件诊断工具|lsof +L1]] 可定位这类场景。

## 相关笔记

- [[Linux文件链接计数-link-count|Linux 文件链接计数（Link Count）]] — link count 存储在 inode 中，二者直接相关
- [[lsof-文件诊断工具|lsof 文件诊断工具]] — 通过 `NODE` 字段查看 inode 编号
- [[stat-命令详解|stat 命令详解]] — inode 查看的标准工具
- [[Linux-Dentry目录项详解|Dentry 目录项详解]] — dentry 是 inode 的"门牌号"
- [[Linux-文件描述符fd详解|文件描述符（fd）详解]] — 进程通过 fd 引用 inode
- [[Linux进程持有已删除文件句柄导致磁盘空间不释放]] — inode 引用未释放的实际排障案例
