---
title: 案例：df 与 du 磁盘空间显示不一致排查
date: 2026-06-01
aliases:
  - df和du显示不一致排查案例
  - 达梦数据库审计文件磁盘空间排查
tags:
  - topic/Linux
  - topic/故障排查
  - topic/Linux/文件系统
status: to-review
---

# 案例：df 与 du 磁盘空间显示不一致排查

## 背景

- Linux 服务器，LVM 卷 `/dev/mapper/rsdata-dmdata` 挂载在 `/rscloud/dmdata`
- 已用 `rm -f` 删除 100G+ 达梦数据库审计文件
- `df -h` 仍显示 100%（186G used / 196G total）
- `du -h` 仅显示约 37G

## 根因

[[Linux进程持有已删除文件句柄导致磁盘空间不释放|进程持有已删除文件句柄导致磁盘空间不释放]]。

简言之：`rm` 删除了 [[Linux-Dentry目录项详解|dentry]]，但达梦数据库进程仍持有文件句柄（[[Linux-文件描述符fd详解|fd]]），导致 [[Linux-Inode详解|inode]] 和数据块未被回收。

## 排查步骤

### Step 1：确认是否存在被进程持有的已删除文件

```bash
# 查看指定挂载点下 link count = 0 的文件
lsof +L1 /rscloud/dmdata
```

或者更精确地限定路径：

```bash
lsof -nP | grep '(deleted)' | grep '/rscloud/dmdata'
```

### Step 2：若 lsof 不可用，使用 /proc 文件系统

```bash
# 精确查找指定路径下的已删除文件
find /proc/*/fd -type l -lname '*/rscloud/dmdata/* (deleted)' 2>/dev/null
```

### Step 3：确认是哪个进程占用了句柄

输出示例：

```
COMMAND   PID   USER   FD   TYPE  DEVICE   SIZE   NODE   NAME
dmserver 12345 dmdba  11w   REG   dm-0    100G  654321 /rscloud/dmdata/data/... (deleted)
```

重点关注：**PID**（进程ID）、**COMMAND**（进程名）、**FD**（文件描述符，`w` 表示写模式）。

### Step 4：释放磁盘空间

#### 方式一：重启持有文件的进程（推荐）

```bash
# 重启达梦数据库服务
systemctl restart DmService
```

或手动终止进程（谨慎）：

```bash
kill -HUP <PID>    # SIGHUP，优先尝试，部分进程会重载配置而不退出
kill <PID>         # 正常终止，句柄自动释放
```

#### 方式二：清空文件而非删除（预防性操作）

```bash
# 三种等价方式，均不改变 inode，句柄不受影响
: > /path/to/audit_file.log
cat /dev/null > /path/to/audit_file.log
truncate -s 0 /path/to/audit_file.log
```

### Step 5：验证空间是否释放

```bash
df -h /rscloud/dmdata
```

## 达梦数据库审计文件管理建议

1. **开启审计文件自动清理**：配置达梦数据库的定期清理策略，避免人工操作遗漏
2. **设置审计文件大小限制**：限制单个审计文件的最大大小，防止无限增长
3. **使用 truncate 方式清理**：避免 `rm -f` 后句柄残留问题
4. **监控磁盘使用率**：设置告警阈值（如 80%），及早发现异常

## 相关笔记

- [[Linux进程持有已删除文件句柄导致磁盘空间不释放|原理篇：进程持有已删除文件句柄导致磁盘空间不释放]]
- [[lsof-文件诊断工具|lsof 文件诊断工具]] — `lsof +L1` 的详细用法
- [[Linux-Dentry目录项详解|Dentry 目录项详解]] — 为什么 `rm` 删除的只是 dentry
- [[Linux-文件描述符fd详解|文件描述符（fd）详解]] — 进程持有文件句柄的底层机制
- [[Linux-Inode详解|Inode 详解]] — inode 引用计数与空间回收的关系