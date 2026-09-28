---
title: logrotate — 配置结构与 rotate 语义
date: 2026-09-28
tags:
  - topic/Linux
  - topic/Linux/定时任务
  - topic/日志
status: to-review
aliases:
  - logrotate rotate 6 含义
  - logrotate 配置详解
  - rotate 6 是几个文件
---

# logrotate — 配置结构与 rotate 语义

## 结论

`logrotate` 是 Linux 上标准的日志轮转工具，由 [[Linux-logrotate-可用性与自动执行检查|systemd timer 或 cron.daily]] 触发。

**最容易理解错的一点：`rotate N` 表示保留 N 个归档，加上当前正在写的日志，实际共 N+1 个文件。**

---

## 配置块结构

系统级配置在 `/etc/logrotate.conf`，服务级配置放在 `/etc/logrotate.d/` 下（推荐，便于单独维护）：

```conf
/var/log/docker-image-prune/image.clean.log {
    daily
    rotate 6
    missingok
    notifempty
    compress
    copytruncate
}
```

语法规则：

- 第一行是**被管理的日志路径**，可写多个路径或使用通配符；路径可以带引号。
- 大括号内是该日志专属的指令。
- 大括号外的指令是全局默认值，块内指令覆盖全局值。

---

## 常用指令

| 指令 | 含义 |
|:-----|:-----|
| `daily` / `weekly` / `monthly` | 轮转周期 |
| `rotate N` | 保留 N 个归档（**不含当前日志**） |
| `missingok` | 日志文件不存在时不报错 |
| `notifempty` | 日志为空时不轮转 |
| `compress` | 归档使用 gzip 压缩（`.gz`） |
| `copytruncate` | 先复制日志再清空原文件，不改变 inode |
| `create` | 轮转后新建空日志文件（与 `copytruncate` 二选一） |
| `dateext` | 归档名使用日期后缀而非递增数字 |
| `dateformat -%Y-%m-%d` | 配合 `dateext` 指定日期格式 |
| `size 100M` | 按大小触发轮转（与时间条件可叠加） |

### `copytruncate` 与 `create` 的选择

| 方式 | 机制 | 适用场景 | 代价 |
|:-----|:-----|:---------|:-----|
| `copytruncate` | 复制日志内容到归档，然后截断原文件 | 写入进程**不支持重开文件**（无法接收信号或不会重新 `open`） | 复制与截断之间存在时间窗，窗口内写入的日志可能丢失 |
| `create` | 重命名旧文件，再创建同名新文件 | 写入进程能在收到信号后重新打开日志 | 需要配置 `postrotate` 通知进程重开文件 |

> [!warning] `copytruncate` 有丢日志窗口
> 它是「先 copy、后 truncate」两步操作，两步之间进程写入的内容会随截断一起消失。日志用于审计或计费时不要用它。

---

## `rotate N` 的精确语义

```conf
rotate 6
```

- 保留 **6 个轮转归档**：`.1`、`.2` …… `.6`
- 加上当前正在写的 `image.clean.log`，**磁盘上共 7 个文件**
- 第 7 个归档 `.7` 会被删除

| 目标（想看几个文件） | `rotate` 取值 |
|:---------------------|:--------------|
| 共 7 个（6 归档 + 当前） | `rotate 6` |
| 共 8 个（7 归档 + 当前） | `rotate 7` |

---

## 用 `dateext` 让归档名带日期

默认归档名是递增数字（`.1`、`.2`），看不出是哪天的日志。改用日期后缀：

```conf
/var/log/docker-image-prune/image.clean.log {
    daily
    rotate 6
    dateext
    dateformat -%Y-%m-%d
    missingok
    notifempty
    compress
    copytruncate
}
```

轮转后：

```text
image.clean.log
image.clean.log-2026-09-28.gz
image.clean.log-2026-09-27.gz
...
```

`dateformat` 支持的占位符与 `date` 命令不同：`%Y`（年）、`%m`（月）、`%d`（日）可用，但 **`%F` 在 logrotate 中代表「被轮转的原文件名」，不是 `date` 里的 `%Y-%m-%d`**。在 `dateformat` 中写 `%F` 会把原文件名拼进归档名，产生无意义甚至冲突的文件名，因此必须用 `-%Y-%m-%d` 显式书写。

---

## 关键前提：被管理日志必须是固定文件名

logrotate 通过**文件名精确匹配**（或通配符匹配）来管理日志。如果 cron 每天生成一个新文件：

```cron
# 每天生成 image.clean.log.2026-09-28 这样的新文件
0 22 * * * /usr/bin/docker image prune -af >> /var/log/docker-image-prune/image.clean.log.$(date +\%F) 2>&1
```

那么下面这个配置**管不到这些带日期的文件**：

```conf
/var/log/docker-image-prune/image.clean.log { ... }
```

结果就是两套日志并存、越积越多。

**正确做法**：使用 logrotate 时，写入方必须输出到固定文件名。

```cron
0 22 * * * /usr/bin/docker image prune -af >> /var/log/docker-image-prune/image.clean.log 2>&1
```

> [!tip] 「谁来切割日志」只能有一个
> 要么由写入方按日期命名、自己清理旧文件；要么由 logrotate 管理固定文件名的日志。
> 两者混用会产生重复文件，且清理逻辑互相看不见。

---

## 验证配置

配置写完后**必须**先做语法与行为验证，再做真实轮转：

```bash
logrotate -d /etc/logrotate.d/docker-image-prune    # dry-run，只打印，不改动
logrotate -vf /etc/logrotate.d/docker-image-prune   # 强制真实轮转，生产慎用
```

输出逐行解读见 [[Linux-logrotate-调试与归档查看]]。

---

## 关联知识点

- [[Linux-logrotate-调试与归档查看]] — `-d` 输出解读与归档命名规则
- [[Linux-logrotate-可用性与自动执行检查]] — 安装状态与自动触发机制检查
- [[保留最新N个日志文件的方案对比]] — logrotate 与「脚本 + cron」方案的对比
- [[Linux-cron定时任务常见陷阱]] — 配合 crontab 使用时的注意事项
- [[Linux进程持有已删除文件句柄导致磁盘空间不释放]] — 日志不轮转导致磁盘打满的相邻故障
