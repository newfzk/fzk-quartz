---
title: Linux-ext4文件系统
date: 2026-06-02
tags:
  - topic/Linux
  - topic/文件系统
status: to-review
updated: 2026-06-02 00:00:00
---

## 核心概念

ext4（Fourth Extended Filesystem）是 Linux 最广泛使用的日志文件系统，2008 年稳定，向后兼容 ext2/ext3。

## 关键特性

| 特性 | 说明 |
|------|------|
| 日志（Journal） | 记录元数据变更，崩溃后快速恢复，避免 fsck |
| 延迟分配（Delayed Allocation） | 先分配内存缓存，再批量写入磁盘，减少碎片 |
| Extent 树 | 用连续区间代替间接块，提升大文件性能 |
| 在线碎片整理 | `e4defrag` 支持在线碎片整理 |
| 文件系统最大 1EB | 单个文件最大 16TB |
| 纳秒级时间戳 | 比 ext3 的秒级更精确 |

## 挂载选项

```bash
/dev/sda1 on /boot type ext4 (rw,noatime)
```

- `noatime`：不更新文件访问时间，提升性能
- `relatime`：相对更新 atime，兼顾性能与 POSIX 兼容
- `errors=remount-ro`：出错时重新挂载为只读，保护数据

## 与 XFS 对比

| 维度 | ext4 | XFS |
|------|------|-----|
| 起源 | Linux 原生 | SGI IRIX 移植 |
| 大文件 | 支持但不如 XFS | 擅长超大文件 |
| 并发写入 | 一般 | 分配组（AG）并行写入 |
| 缩小文件系统 | 支持 | 不支持 |
| 默认场景 | 通用、启动分区 | 数据盘、日志盘 |

## 面试要点

- ext4 是 ext3 的升级，核心改进：Extent 树、延迟分配、在线碎片整理
- `/boot` 分区用 ext4 是因为 GRUB 对 ext4 兼容性最好
- `tune2fs -l /dev/sda1` 查看 ext4 文件系统超级块信息

## 参考链接

- [[柠檬微趣-笔试-Q2-mount命令输出解释|柠檬微趣-笔试-Q2-mount命令输出解释]]