---
title: Linux devtmpfs 文件系统
date: 2026-06-02
tags:
  - topic/Linux
  - topic/VFS
  - topic/设备
status: evergreen
aliases:
  - devtmpfs
  - /dev
  - 设备文件系统
  - 设备节点管理
---

# Linux devtmpfs 文件系统

## 名字由来：Device + Temporary Filesystem

**devtmpfs** 的名称可以拆解为三部分：

| 部分 | 含义 | 说明 |
|------|------|------|
| **dev** | Device（设备） | 这个文件系统管理**设备节点** |
| **tmp** | Temporary（临时的） | 内容存储在**内存**中，重启消失 |
| **fs** | Filesystem（文件系统） | 以文件系统形式呈现 |

全称就是 **"设备节点的临时文件系统"**。

> 为什么有 "tmp"？因为设备节点是在内存中动态生成的，不是你手动写到磁盘上的。当硬件移除或系统重启，这些节点就消失了。

## 核心本质

`/dev` 目录下的每一个文件都代表一个**设备**：

```
/dev/
├── sda      # 第一块 SCSI 磁盘（块设备）
├── sda1     # 第一块磁盘的第一个分区
├── sdb      # 第二块磁盘
├── tty0     # 第一个终端（字符设备）
├── pts/0    # 伪终端（SSH 连接）
├── null     # 黑洞设备
├── zero     # 无限零设备
├── random   # 随机数设备
└── urandom  # 更快（但略不随机）的随机数设备
```

## 历史：它是怎么来的？

```
传统 /dev（静态） →  udev（动态） →  devtmpfs（内核直接管理）
                                                                        
Linux 2.4 及以前:     Linux 2.6 时代:       Linux 2.6.32+（2009）:
/dev 预创建数千个      udev 用户态守护进程    内核检测到硬件后
设备文件              监听内核事件            当场在 /dev 创建设备节点
占用大量 inode        按需创建/删除          延迟更低、更可靠
                                            
"先准备好所有可能      "有硬件了再创建"        "内核直接帮你创建，
  的设备文件"                                 不需要等用户态程序"
```

devtmpfs 的改进：当内核检测到新硬件时，**立即在 /dev 下创建对应的设备文件**，而不需要等待 udev 用户态程序响应。udev 仍然可以在此基础上做权限设置、命名规则等高级操作。

## 理解设备号

```bash
ls -l /dev/sda
# brw-rw---- 1 root disk 8, 0 Jun 2 10:00 /dev/sda
# ↑               ↑   ↑         ↑
# 块设备(首字母b)  主设备号 次设备号  设备名
# 字符设备则是 c

# 主设备号（8）→ 标识设备类型（SCSI 磁盘）
# 次设备号（0）→ 标识同类设备中的第几个（sda=0, sda1=1, sda2=2...）
```

> **类比**：主设备号 = 小区的**楼栋号**（8号楼），次设备号 = **房间号**（0=一楼大厅，1=101室，2=102室...）
> 内核看到 (8, 0) 就知道："这是第8栋楼1楼大厅，找 SCSI 磁盘驱动来处理"

## 经典例子

### 1. 常见设备文件

```bash
# 块设备（带缓冲，可随机访问）
ls -l /dev/sda      # 磁盘
ls -l /dev/loop0    # loop 设备（挂载 ISO 文件）
ls -l /dev/nvme0n1  # NVMe 固态硬盘

# 字符设备（不带缓冲，流式访问）
ls -l /dev/tty      # 当前终端
ls -l /dev/pts/0    # SSH 会话的伪终端
ls -l /dev/null     # 黑洞
ls -l /dev/random   # 随机数
```

### 2. /dev/null — 最著名的黑洞

```bash
# "把输出扔进黑洞"
find / -name "*.log" 2>/dev/null
# 错误信息 → /dev/null → 消失

# 清空文件而不删除
cat /dev/null > large_log.txt
# 等价于: : > large_log.txt

# 测试写性能（写入黑洞）
dd if=/dev/zero of=/dev/null bs=1M count=10000
# 这个操作极其快，因为数据根本没有写磁盘
```

### 3. /dev/random vs /dev/urandom

```bash
# random：阻塞式随机数（熵不够时会等）
dd if=/dev/random of=/tmp/rnd bs=1 count=32  # 可能卡住

# urandom：非阻塞式随机数（更快）
dd if=/dev/urandom of=/tmp/rnd bs=1M count=1  # 很快
```

### 4. 实战：设备热插拔

```bash
# 插入 U 盘前：
ls /dev/sd*

# 插入 U 盘后（内核自动创建）：
ls /dev/sd*
# 输出比之前多了 sdb、sdb1

# 这正是 devtmpfs 在工作：内核检测到 USB 存储设备
# → 分配主次设备号 → devtmpfs 创建 /dev/sdb 和 /dev/sdb1
# → 你可以直接用 mount /dev/sdb1 /mnt

# 拔掉 U 盘：
# devtmpfs 自动删除 /dev/sdb /dev/sdb1
```

## devtmpfs vs tmpfs

| 维度 | devtmpfs | tmpfs |
|------|----------|-------|
| **挂载点** | `/dev` | `/tmp`、`/run`、`/dev/shm` |
| **内容** | 设备节点（内核自动管理） | 用户临时文件 |
| **创建方式** | 内核检测到硬件时自动创建 | 用户应用程序创建 |
| **核心用途** | 设备访问入口 | 临时数据存储 |

> devtmpfs 虽然名字里带 tmpfs，但它的设备节点是内核管理的，和用户存临时文件的 tmpfs 是两回事。

## 相关笔记

- [[Linux-虚拟文件系统VFS]] — VFS 抽象层，devtmpfs 是具体实现之一
- [[Linux-tmpfs-文件系统]] — 同样是"tmp"家族，但用途不同
- [[Linux-磁盘分区与设备命名]] — 磁盘设备命名规则
- [[Linux-文件描述符fd详解]] — 设备文件通过 fd 被进程访问
