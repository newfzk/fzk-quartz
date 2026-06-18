---
title: stat 命令详解
date: 2026-06-01
aliases:
  - stat
  - stat 命令
tags:
  - topic/Linux
  - topic/Linux/命令
  - topic/Linux/文件系统
status: to-review
---

# stat 命令详解

`stat` 是 Linux 下用于查看文件或文件系统**元数据（metadata）** 的命令。它从 [[Linux-Inode详解|inode]] 中读取信息并格式化输出。

## 基本用法

```bash
stat [选项] 文件...
```

## 输出解读

```bash
$ stat test.txt
```

```
  文件：test.txt
  大小：1024          块：8          IO 块：4096    普通文件
设备：fd01h/64769d    Inode：654321    硬链接：1
权限：(0644/-rw-r--r--)  Uid：( 1000/  user)   Gid：( 1000/  user)
访问时间：2026-06-01 10:00:00.123456789 +0800
修改时间：2026-06-01 09:00:00.987654321 +0800
状态更改时间：2026-06-01 08:00:00.000000000 +0800
创建时间：2026-05-30 14:00:00.000000000 +0800
```

| 字段 | 含义 | 关联知识点 |
|---|---|---|
| **大小** | 文件实际内容的字节数 | — |
| **块** | 文件占用的磁盘块数（通常 1 块 = 512 字节） | 与"大小"不同，涉及 [[Linux-Inode详解|inode]] 块指针 |
| **IO 块** | 文件系统的逻辑块大小（通常 4096） | 文件系统块大小 |
| **设备** | 设备编号（`主:次` 或 `十六进制/十进制`） | 该文件所在的分区 |
| **Inode** | inode 编号 | [[Linux-Inode详解|Inode 详解]] |
| **硬链接** | [[Linux文件链接计数-link-count\|link count]] | 指向此 inode 的目录项数量 |
| **权限** | 数字 + 符号形式的权限位 | `0644` = `rw-r--r--` |
| **Uid / Gid** | 用户 ID / 组 ID 及名称 | 文件拥有者 |
| **访问时间** | `atime`，最后一次读取内容的时间 | 频繁访问会更新此时间 |
| **修改时间** | `mtime`，最后一次修改内容的时间 | `ls -l` 默认显示此时间 |
| **状态更改时间** | `ctime`，最后一次修改 inode 元数据的时间 | 改权限/重命名/增减链接数都会更新 |
| **创建时间** | `btime`（Birth time），文件创建时间 | 并非所有文件系统支持（ext4 支持） |

## 常用选项

### `-f`：查看文件系统信息

```bash
$ stat -f /

文件："/"
  ID：a1b2c3d4e5f6a7b8 文件名长度：255     块大小：4096
  块总数：48857088     可用块：10485760     可用块(非root)：9437184
  Inode 总数：12222464  可用 Inode：10000000
```

用于排查：磁盘空间不足、[[Linux-Inode详解#Inode 编号与文件系统|inode 耗尽]]等问题。

### `-c` / `--format`：自定义输出格式

```bash
# 只看 inode 和 link count
stat -c '%i %h %n' test.txt
# 输出：654321 1 test.txt

# 只查看权限（八进制）
stat -c '%a' test.txt
# 输出：644

# 只查看文件大小
stat -c '%s' test.txt
# 输出：1024
```

### 常用格式占位符

| 占位符 | 含义 |
|---|---|
| `%i` | inode 编号 |
| `%h` | [[Linux文件链接计数-link-count\|link count]] |
| `%s` | 文件大小（字节） |
| `%a` | 权限（八进制） |
| `%A` | 权限（可读格式，如 `rw-r--r--`） |
| `%n` | 文件名 |
| `%F` | 文件类型（"普通文件"、"目录"等） |
| `%y` | 最后修改时间（mtime） |
| `%x` | 最后访问时间（atime） |
| `%z` | 最后状态更改时间（ctime） |
| `%w` | 创建时间（btime） |

### `-t`：简洁模式（terse）

```bash
$ stat -t test.txt
test.txt 1024 8 81a4 1000 1000 fd01 654321 1 0 0 1717200000 1717196400 1717192800 1717056000 4096
```

各字段依次为：文件名、大小、块数、权限(八进制)、uid、gid、设备号、inode、link count、... 时间戳（unix timestamp）。适合脚本解析。

## 与 `ls -l` 的对比

| 对比项 | `ls -l` | `stat` |
|---|---|---|
| 信息量 | 精简（大小、权限、mtime、文件名） | 完整（含 atime、ctime、inode、块信息） |
| 格式灵活性 | 固定格式 | 可通过 `-c` 自定义 |
| 时间精度 | 秒级 | 纳秒级 |
| 文件系统信息 | 不支持 | 支持 `-f` |
| 批量处理 | 支持目录列表 | 逐个文件或路径 |

```bash
# stat 可同时查看多个文件
stat file1.txt file2.txt file3.txt
```

## 典型应用场景

### 1. 查看 [[Linux文件链接计数-link-count|link count]]

```bash
# 文件是否有硬链接？
stat -c '%h %n' *.txt
# 输出 link count > 1 表示有硬链接指向同一 inode
```

### 2. 确认文件是否被修改

```bash
# 监控 mtime 和 ctime
stat --printf='mtime: %y\nctime: %z\n' /etc/passwd
```

### 3. 排查文件系统 inode 使用率

```bash
# 检查 /var 分区的 inode 是否耗尽
stat -f /var | grep -E 'Inode|可用'
```

### 4. 脚本中使用

```bash
# 获取文件大小（比 wc -c 更快，不读取内容）
size=$(stat -c '%s' largefile.bin)
```

## 注意事项

- **atime 的更新行为**：现代 Linux 使用 `relatime` 挂载选项，atime 不会每次读取都更新，避免性能损耗
- **btime（创建时间）**：并非所有文件系统支持——ext4、xfs、btrfs 支持，但 tmpfs、NFS 可能不支持
- **ctime ≠ 创建时间**：`ctime` 是 inode 状态变更时间（改权限、重命名、改链接数等），`btime` 才是真正的创建时间

## 相关笔记

- [[Linux-Inode详解|Inode 详解]] — `stat` 读取的数据来源
- [[Linux文件链接计数-link-count|Linux 文件链接计数（Link Count）]] — 用 `stat` 查看 link count
- [[lsof-文件诊断工具|lsof 文件诊断工具]] — 同样是文件诊断的基础工具
