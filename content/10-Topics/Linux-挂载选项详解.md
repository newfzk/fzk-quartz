---
title: Linux-挂载选项详解
date: 2026-06-02
tags:
  - topic/Linux
  - topic/mount
  - topic/安全
status: to-review
updated: 2026-06-02 00:00:00
---

## 核心概念

`mount` 命令的挂载选项（mount options）控制文件系统的访问行为和性能特征，分为通用选项和文件系统特定选项。

## 安全三剑客

| 选项 | 含义 | 安全作用 |
|------|------|----------|
| `nosuid` | 忽略 suid/sgid 位 | 防止用户通过 suid 程序提权 |
| `nodev` | 不解析字符/块设备文件 | 防止恶意创建设备节点访问物理设备 |
| `noexec` | 禁止执行二进制文件 | 防止上传恶意脚本/程序后执行 |

> 虚拟文件系统（`/proc`、`/sys`、`/run`）和用户数据分区（`/tmp`、`/home`）通常启用这些安全选项。

## atime 选项的性能取舍

| 选项 | 行为 | 性能 | 规范兼容 |
|------|------|------|----------|
| `atime`（默认） | 每次访问都更新 atime | 最慢 | 完全 POSIX 兼容 |
| `noatime` | 从不更新 atime | 最快 | 违反 POSIX |
| `relatime` | 条件更新 atime | 折中 | 兼顾 |
| `nodiratime` | 不更新目录 atime | 改善 | 部分兼容 |

**relatime 更新规则**：只有 atime 早于 mtime 或 ctime 时，或距上次更新超过 24 小时，才更新 atime。这是 Linux 内核默认行为。

## 其他常用选项

| 选项 | 含义 |
|------|------|
| `rw` / `ro` | 读写 / 只读 |
| `sync` / `async` | 同步 / 异步写入 |
| `errors=remount-ro` | 文件系统出错时重新挂载为只读 |
| `defaults` | 等效于 `rw,suid,dev,exec,auto,nouser,async` |
| `noauto` | 不在 `mount -a` 时自动挂载 |

## 面试要点

- `errors=remount-ro` 是根文件系统常用的保护选项，防止数据损坏扩散
- 临时目录（`/tmp`）应添加 `noexec` 防止恶意脚本执行
- `noatime` 对数据库、邮件服务器等随机读场景性能提升明显（可减少 20%~30% 的写 I/O）

## 参考链接

- [[柠檬微趣-笔试-Q2-mount命令输出解释|柠檬微趣-笔试-Q2-mount命令输出解释]]