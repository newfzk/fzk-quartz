---
title: 保留最新 N 个日志文件的三种方案对比
date: 2026-09-28
tags:
  - topic/Linux
  - topic/Linux/定时任务
  - topic/日志
status: to-review
aliases:
  - 日志只保留最近7个
  - 保留最新7个日志文件
  - 定时任务日志保留策略
---

# 保留最新 N 个日志文件的三种方案对比

## 结论

需求：某个定时任务的日志**只保留最新 7 个文件**。三种可行方案：

| 方案 | 实现方式 | 推荐度 | 适用场景 |
|:-----|:---------|:------:|:---------|
| A. 脚本 + cron | 日志文件名带日期，脚本末尾用 `find` 删除旧文件 | 推荐 | 日志需要按天独立成文件、便于单日检索 |
| B. 一行 crontab | 把 A 的逻辑压成一行 | 不推荐 | 临时应急，无法维护 |
| C. logrotate | 写入固定文件名，交给 logrotate 轮转 | 最标准 | 长期运行的生产服务 |

**选择依据：日志是否需要按天独立成文件。** 需要 → 方案 A；不需要 → 方案 C（更标准，无需自己写清理逻辑）。

---

## 方案 A：脚本 + cron（推荐）

脚本 `/usr/local/bin/docker-image-prune.sh`：

```bash
#!/usr/bin/env bash

LOG_DIR="/var/log/docker-image-prune"
KEEP=7
mkdir -p "$LOG_DIR"

LOG_FILE="$LOG_DIR/image.clean.log.$(date +%F)"

{
  echo "===== $(date '+%F %T') start docker image prune ====="
  /usr/bin/docker image prune -af
  rc=$?
  echo "===== $(date '+%F %T') end, exit=$rc ====="
} >> "$LOG_FILE" 2>&1

find "$LOG_DIR" -maxdepth 1 -type f -name 'image.clean.log.*' -printf '%f\n' \
  | sort -r \
  | tail -n +$((KEEP + 1)) \
  | while IFS= read -r old; do
      rm -f -- "$LOG_DIR/$old"
    done

exit "$rc"
```

赋权与注册：

```bash
chmod +x /usr/local/bin/docker-image-prune.sh
```

```cron
0 22 * * * /usr/local/bin/docker-image-prune.sh
```

### 为什么这样写

| 写法 | 目的 |
|:-----|:-----|
| 逻辑写进脚本，crontab 只调用脚本 | 脚本内 `date +%F` **不需要转义 `%`**，避开 [[Linux-cron定时任务常见陷阱]] 中的头号陷阱 |
| `LOG_DIR` 用绝对路径 | cron 的工作目录是 HOME，相对路径会导致文件落在意外位置 |
| `{ ... } >> "$LOG_FILE" 2>&1` | 同时捕获 stdout 与 stderr，`>>` 追加不覆盖 |
| `rc=$?` 并 `exit "$rc"` | 保留真实退出码，便于上层监控告警 |
| `find -maxdepth 1 -type f -name` | 只在当前目录按前缀匹配，避免误删子目录内容 |
| `sort -r` + `tail -n +$((KEEP+1))` | 文件名含 `YYYY-MM-DD`，字典序倒排即时间倒序；跳过前 7 个（要保留的），删除其余 |
| `IFS= read -r` + `rm -f --` | 正确处理文件名中的空白与特殊字符 |

> [!warning] 依赖文件名可排序
> 该方案能成立的前提是**文件名中的日期格式是零填充的**（`2026-09-28` 而非 `2026-9-28`）。否则字典序与时间序不一致，会删错文件。`date +%F` 输出零填充日期，安全。

### 只清理 dangling 镜像的变体

```bash
/usr/bin/docker image prune -f                                   # 只删 <none> 镜像
/usr/bin/docker image prune -af --filter "until=168h"            # 只删超过 7 天未使用的镜像
```

删除范围的取舍见 [[Docker-image-prune镜像清理]]。

---

## 方案 B：一行 crontab（不推荐，但可行）

```cron
0 22 * * * mkdir -p /var/log/docker-image-prune && /usr/bin/docker image prune -af >> /var/log/docker-image-prune/image.clean.log.$(date +\%F) 2>&1; find /var/log/docker-image-prune -maxdepth 1 -type f -name 'image.clean.log.*' -printf '\%f\n' | sort -r | tail -n +8 | while IFS= read -r f; do rm -f -- "/var/log/docker-image-prune/$f"; done
```

要点与代价：

- **所有 `%` 都必须转义**：`date +\%F`、`find -printf '\%f\n'`。漏一个就整条命令失效。
- `tail -n +8` 表示从第 8 行开始删除，即保留最新 7 个。
- 代价：无法阅读、无法单独测试、无法版本管理。仅适合临时验证。

---

## 方案 C：logrotate（更标准）

前提：**cron 输出到固定文件名**（不带日期）。

```cron
0 22 * * * /usr/bin/docker image prune -af >> /var/log/docker-image-prune/image.clean.log 2>&1
```

`/etc/logrotate.d/docker-image-prune`：

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

要点：

- **`rotate 6` 不是 6 个文件**，而是 6 个归档 + 当前日志 = 7 个文件，正好满足「保留最新 7 个」。
- `copytruncate` 让 docker 命令继续往同一个 inode 追加，无需重启任何进程；代价是存在极短的丢日志窗口。
- 写入方必须是固定文件名，否则 logrotate 匹配不到带日期的文件。

细节见 [[Linux-logrotate-配置与rotate语义]]，验证见 [[Linux-logrotate-调试与归档查看]] 与 [[Linux-logrotate-可用性与自动执行检查]]。

---

## 两种思路的本质区别

| 维度 | 方案 A（按日期命名） | 方案 C（logrotate） |
|:-----|:---------------------|:--------------------|
| 谁负责切割 | 写入方（cron 命令里的 `date`） | logrotate |
| 谁负责清理 | 自己写 `find` 删除逻辑 | `rotate N` 声明式配置 |
| 日志文件名 | 每天一个新文件 | 始终同一个 + 编号归档 |
| 单日检索 | 直接 `cat` 对应日期文件 | 需要按归档编号推断日期（或用 `dateext`） |
| 压缩 | 需自行加 `gzip` | `compress` 一行搞定 |
| 配置可维护性 | 逻辑在脚本里，清晰 | 声明式配置，最清晰 |
| 失败表现 | 删除逻辑写错会误删或堆积 | 配置错误时 `logrotate -d` 可提前发现 |

> [!danger] 不要混用
> 「按日期命名」和「logrotate 管理固定文件」是两套互不可见的机制。同时启用会出现两套日志文件，且都以为自己在管理同一批日志，结果越积越多。

---

## 落地清单（方案 C 为例）

1. 创建日志目录：`mkdir -p /var/log/docker-image-prune`
2. 修改 crontab，输出到**固定文件名**并合并 stderr。
3. 新建 `/etc/logrotate.d/docker-image-prune`，写入 `rotate 6` 配置。
4. 确认自动触发机制存在（timer 或 `/etc/cron.daily/logrotate`）。
5. `logrotate -d` 验证语法。
6. 用 `ls -1 .../image.clean.log* | wc -l` 确认最终文件数为 7。

---

## 关联知识点

- [[Linux-cron定时任务常见陷阱]] — `%` 转义、绝对路径、stderr 合并
- [[Docker-image-prune镜像清理]] — 被定时执行的清理命令本身
- [[Linux-logrotate-配置与rotate语义]] — `rotate N` 的精确含义
- [[Linux-logrotate-调试与归档查看]] — 验证与查看归档
- [[Linux-logrotate-可用性与自动执行检查]] — 检查是否具备自动触发机制
- [[Shell-脚本备忘]] — 脚本语法参考
- [[Linux进程持有已删除文件句柄导致磁盘空间不释放]] — 日志未轮转导致磁盘打满的故障模式
