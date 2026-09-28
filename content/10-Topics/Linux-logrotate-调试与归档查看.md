---
title: logrotate — 调试输出与归档查看
date: 2026-09-28
tags:
  - topic/Linux
  - topic/Linux/定时任务
  - topic/日志
status: to-review
aliases:
  - logrotate -d 输出解读
  - 查看 logrotate 归档日志
  - logrotate dry-run
---

# logrotate — 调试输出与归档查看

## 结论

- **验证配置**用 `logrotate -d`（dry-run，只打印不改动）。
- **强制真实轮转**用 `logrotate -vf`，生产环境谨慎（会立即切割并生成归档）。
- **归档文件的新旧顺序**：`image.clean.log`（当前，最新）→ `.1` → `.2` …… **编号越小越新**。

---

## `logrotate -d`：dry-run 验证

```bash
logrotate -d /etc/logrotate.d/docker-image-prune
```

典型输出：

```text
WARNING: logrotate in debug mode does nothing except printing debug messages!
reading config file /etc/logrotate.d/docker-image-prune
Reading state from file: /var/lib/logrotate/status
Creating new state
Handling 1 logs
rotating pattern: /var/log/docker-image-prune/image.clean.log  after 1 days (6 rotations)
empty log files are not rotated, old logs are removed
considering log /var/log/docker-image-prune/image.clean.log
Creating new state
```

### 逐行解读

| 输出行 | 含义 | 是否正常 |
|:-------|:-----|:---------|
| `WARNING: logrotate in debug mode...` | 提示当前是调试模式 | 正常，非错误 |
| `reading config file ...` | 成功读取指定配置 | 正常 |
| `Reading state from file: /var/lib/logrotate/status` | 读取状态文件（记录上次轮转时间） | 正常 |
| `Creating new state` | 首次记录该日志状态 | 正常，首次运行必现 |
| `Handling 1 logs` | 匹配到 1 个日志文件 | 数量应与配置一致 |
| `rotating pattern: ... after 1 days (6 rotations)` | 周期 1 天，保留 6 个归档 | 应与配置一致 |
| `empty log files are not rotated` | 空日志不轮转（来自 `notifempty`） | 正常 |

### 判定标准

- 输出中出现 `error` 或 `syntax error` → 配置有问题，需修正。
- `Handling 0 logs` → 路径写错，或文件不存在且未配 `missingok` 的相关提示。
- **只有 WARNING + 正常行，没有 error → 配置可用。**

---

## `logrotate -vf`：强制真实执行

```bash
logrotate -vf /etc/logrotate.d/docker-image-prune
```

- `-f`：忽略周期判断，立即轮转。
- `-v`：输出详细信息。

> [!danger] `-f` 会真的动文件
> 生产环境不要随便跑。它的用途是：新配置上线后**在测试机验证一次完整轮转效果**，确认归档名、数量、权限都符合预期。

只强制某一个配置文件轮转，不要跑 `logrotate -vf /etc/logrotate.conf`（会影响系统全部日志）。

---

## 归档命名规则

默认命名（未启用 `dateext`）：

```text
image.clean.log          # 当前正在写，最新
image.clean.log.1        # 上一次轮转，第 2 新
image.clean.log.2.gz     # 更旧
image.clean.log.3.gz
image.clean.log.4.gz
image.clean.log.5.gz
image.clean.log.6.gz     # 最旧
```

规则：

1. `image.clean.log` 永远是**最新**的。
2. `.1` 是上一次轮转的内容，**不是最新**。
3. 编号越大越旧；`rotate 6` 时最大编号是 `.6`。
4. 启用 `compress` 后，新生成的归档为 `.gz`；`.1` 可能仍是未压缩的纯文本，这是正常的（压缩发生在上一次轮转的归档上）。

启用 `dateext` 后（见 [[Linux-logrotate-配置与rotate语义]]）：

```text
image.clean.log
image.clean.log-2026-09-28.gz
image.clean.log-2026-09-27.gz
```

---

## 查看归档日志

### 列出与计数

```bash
ls -lh /var/log/docker-image-prune/                          # 看大小和时间
ls -1 /var/log/docker-image-prune/image.clean.log* | sort -V # 按版本排序
ls -1 /var/log/docker-image-prune/image.clean.log* | wc -l   # 数个数
```

`wc -l` 的结果判读：

- 等于 7（`rotate 6` 配置下）→ 保留策略生效。
- 大于 7 → 有旧文件未清理，或目录里混入了其他文件（例如同时存在 `.log.2026-09-28` 这类日期命名文件）。

### 查看内容

```bash
# 未压缩
cat  /var/log/docker-image-prune/image.clean.log.1
less /var/log/docker-image-prune/image.clean.log.1

# 压缩归档
zcat  /var/log/docker-image-prune/image.clean.log.2.gz
zless /var/log/docker-image-prune/image.clean.log.2.gz
zgrep "error" /var/log/docker-image-prune/image.clean.log.*.gz

# 一次看全部（zcat -f 对未压缩文件也生效）
zcat -f /var/log/docker-image-prune/image.clean.log* | less
```

`zcat -f` 是关键：它同时对 `.gz` 和纯文本文件生效，不用区分是否压缩。

---

## 常见误区

1. **以为 `.1` 是最新** — 最新的永远是 `image.clean.log` 本身。
2. **以为 `rotate 6` 等于保留 6 个文件** — 实际是 6 个归档 + 1 个当前日志 = 7 个文件。
3. **同时使用日期命名与 logrotate** — 写入方产出 `xxx.log.2026-09-28`，logrotate 管理 `xxx.log`，两套逻辑互不可见，日志只会越积越多。
4. **看到 `.1` 没被压缩就以为配置失效** — `compress` 影响的是本次轮转产出的归档，`.1` 可能是上次轮转留下的未压缩文件。
5. **改了配置却不验证** — 必须用 `logrotate -d` 至少确认没有语法错误。

---

## 关联知识点

- [[Linux-logrotate-配置与rotate语义]] — 配置文件结构与 `rotate` 语义
- [[Linux-logrotate-可用性与自动执行检查]] — 确认 logrotate 会被自动触发
- [[保留最新N个日志文件的方案对比]] — 与其他日志保留方案的对比
- [[Linux-cron定时任务常见陷阱]] — cron 输出必须是固定文件名
