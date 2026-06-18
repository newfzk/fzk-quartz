---
title: Linux 进程持有已删除文件句柄导致磁盘空间不释放
date: 2026-06-01
aliases:
  - df与du显示不一致
  - 已删除文件空间不释放
  - 磁盘空间幽灵占用
  - deleted files taking up space
tags:
  - topic/Linux
  - topic/Linux/文件系统
  - topic/故障排查
status: to-review
---

# Linux 进程持有已删除文件句柄导致磁盘空间不释放

## 现象

`df -h` 显示磁盘使用率很高（甚至 100%），但 `du -h` 统计的实际文件总量远小于 `df` 的已用空间。

这是因为 `df` 统计文件系统层面的已分配块，而 `du` 只统计目录树可遍历到的文件。

## 根因

在 Linux 中，删除文件 (`rm` / `unlink()`) 并不立即释放磁盘空间，其流程为：

1. `rm` 仅删除**[[Linux-Dentry目录项详解|目录项（dentry）]]**，即文件名与 inode 的映射关系
2. 若有进程已通过 `open()` 打开了该文件，则 inode 的引用计数仍 > 0
3. inode 及其数据块只有在引用计数归零后才被回收
4. 若进程异常退出前未 `close()`，或进程仍在运行（如持续写日志），空间一直不释放

**公式：** `df ≈ du + 已删除但进程仍在引用的文件大小`

## 排查方法

使用 [[lsof-文件诊断工具|lsof]] 查找已删除但仍被进程打开的文件：

```bash
lsof +L1 /挂载点路径
lsof -nP | grep '(deleted)'
```

> 详细用法（端口诊断、进程筛选、输出解读等）请参见：[[lsof-文件诊断工具|lsof 文件诊断工具]]

若未安装 `lsof`，可通过 `/proc` 文件系统排查：

```bash
ls -la /proc/*/fd/ 2>/dev/null | grep '(deleted)'

# 更精确：只查找特定路径下的已删除文件
find /proc/*/fd -type l -lname '*/目标路径/*' 2>/dev/null | \
  while read link; do
    target=$(readlink "$link" 2>/dev/null)
    case "$target" in
      *' (deleted)') echo "$link -> $target" ;;
    esac
  done
```

## 解决方案

### 方式一：重启持有文件的进程（推荐）

```bash
systemctl restart ServiceName

# 或确认 PID 后向进程发送信号
kill -HUP <PID>    # SIGHUP，部分进程会重新加载配置
kill <PID>         # 正常终止进程，句柄自动释放
```

### 方式二：清空文件而非删除（预防性操作）

对于日志、审计等持续写入的文件，不应使用 `rm` 删除，而应**清空文件内容**：

```bash
: > /path/to/file.log
# 或
cat /dev/null > /path/to/file.log
# 或
truncate -s 0 /path/to/file.log
```

清空操作不改变 inode，不影响已打开的文件句柄，空间立即释放。

## 最佳实践

- 日志文件应使用 `truncate` 或 `: >` 清空，而非 `rm` 删除
- 配置日志轮转策略（如 `logrotate`），避免单个文件无限增长
- 设置磁盘使用率监控告警（如 80% 阈值）
- 生产环境操作日志/审计文件前，先确认是否有进程持有句柄

## 关联知识点

- [[lsof-文件诊断工具|lsof 文件诊断工具]] — 文件句柄诊断的核心工具
- [[Linux文件链接计数-link-count|Linux 文件链接计数（Link Count）]] — link count 与 inode 引用机制详解
- [[Linux-Inode详解|Inode 详解]] — inode 是文件元信息的核心结构
- [[Linux-Dentry目录项详解|Dentry 目录项详解]] — dentry 是文件名到 inode 的映射
- [[Linux-文件描述符fd详解|文件描述符（fd）详解]] — 进程持有文件句柄的本质
- [[案例-df与du磁盘空间不一致排查|案例：df与du磁盘空间不一致排查]] — 达梦数据库审计文件实战排查记录
- 待补充：logrotate 日志轮转配置
