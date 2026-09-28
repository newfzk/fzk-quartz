---
title: cron 定时任务常见陷阱
date: 2026-09-28
tags:
  - topic/Linux
  - topic/Linux/定时任务
  - topic/故障排查
status: to-review
aliases:
  - crontab 百分号转义
  - cron 命令不执行
  - crontab % 转义
  - cron 工作目录
---

# cron 定时任务常见陷阱

## 结论

一条在终端里能跑通的命令，直接粘贴进 crontab 后**经常不生效**。原因是 crontab 不是 shell 脚本：它有独立的解析规则、独立的 `PATH`、独立的工作目录。

本文覆盖 4 个高频陷阱：`%` 未转义、相对路径、丢失 stderr、`PATH` 缺失。

---

## 陷阱 1：`%` 必须转义

crontab 中 `%` 有特殊含义：**第一个未转义的 `%` 会被转换成换行符，其后的所有内容作为命令的标准输入（stdin）**。

| 写法 | 结果 |
|:-----|:-----|
| `>> image.clean.log.$(date +%F)` | `%F` 被截断，命令异常或日志名错误 |
| `>> image.clean.log.$(date +\%F)` | 正确 |

```cron
# 错误
0 22 * * * /usr/bin/docker image prune -af > image.clean.log.$(date +%F)

# 正确
0 22 * * * /usr/bin/docker image prune -af > image.clean.log.$(date +\%F)
```

> [!warning] 转义只针对 crontab 层
> 脚本文件内部的 `date +%F` **不需要**转义，因为脚本由 shell 解析，不经过 crontab 的 `%` 规则。
> 这也是「把逻辑写进脚本、crontab 只调用脚本」比「把长命令塞进一行 crontab」更可靠的原因之一。

`%` 转义问题同样出现在命令中的其他 `%`，例如 `find -printf '%f\n'` 在 crontab 中必须写成 `\%f`。

---

## 陷阱 2：相对路径不可靠

cron 执行任务时的默认工作目录是**用户 HOME**（如 `/root`），不是脚本所在目录，也不是你当前所在的目录。

```cron
# 相对路径：文件实际落在 $HOME/image.clean.log.2026-09-28
0 22 * * * /usr/bin/docker image prune -af > image.clean.log.$(date +\%F)
```

**对策**：所有输入、输出路径都写绝对路径。

```cron
0 22 * * * /usr/bin/docker image prune -af >> /var/log/docker-image-prune/image.clean.log 2>&1
```

---

## 陷阱 3：只重定向 stdout 会丢掉错误

`>` 只重定向标准输出。命令报错时信息走 stderr，会被 cron 通过邮件（或直接丢弃）处理，日志文件里看不到原因。

```bash
# 只捕获 stdout，出错时日志是空的
> /var/log/app/task.log

# 正确：追加模式 + 合并 stderr
>> /var/log/app/task.log 2>&1
```

两个要点：

1. `2>&1` 必须写在重定向之后，顺序不能颠倒。
2. 定时任务建议用 `>>`（追加）而非 `>`（覆盖），否则每次执行都会清掉历史。

---

## 陷阱 4：环境变量与 PATH 缺失

cron 不加载 `/etc/profile`、`~/.bashrc`，环境极简：

| 项目 | cron 中的典型值 |
|:-----|:----------------|
| `PATH` | `/usr/bin:/bin`（部分发行版可配置） |
| `HOME` | 对应用户的 HOME |
| 自定义变量 | 不存在 |

后果：

- 自定义目录下的命令（如 `/usr/local/bin/xxx`、`~/.nvm/...`）找不到 → 必须写绝对路径，或在 crontab 顶部声明 `PATH=`。
- 脚本中依赖的变量（`JAVA_HOME`、代理设置）全部缺失 → 在脚本内部显式 `export`。

```cron
PATH=/usr/local/bin:/usr/bin:/bin
0 22 * * * /usr/local/bin/docker-image-prune.sh
```

---

## 排查顺序

按以下顺序逐条确认，可覆盖绝大多数「cron 不执行 / 执行了但没效果」的问题：

1. `%` 是否转义（有 `date` 格式化时必查）。
2. 是否全部使用绝对路径。
3. 是否合并了 stderr（`2>&1`）。
4. 依赖的命令是否在 cron 的 `PATH` 内。
5. cron 服务本身是否在运行：`systemctl status cron` 或 `systemctl status crond`。
6. 手动执行一遍同一条命令，对比行为差异。

---

## 关联知识点

- [[保留最新N个日志文件的方案对比]] — 脚本 + cron 方案中这些陷阱的具体规避写法
- [[Linux-logrotate-配置与rotate语义]] — 更标准的日志轮转方案，可与 cron 配合
- [[Docker-image-prune镜像清理]] — 本文示例命令对应的镜像清理知识点
- [[Shell-脚本备忘]] — 脚本内部的写法参考（`%` 无需转义）
- [[Linux-logrotate-可用性与自动执行检查]] — 判断系统是否具备自动触发机制
