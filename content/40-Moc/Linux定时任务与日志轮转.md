---
title: Linux 定时任务与日志轮转（MOC）
date: 2026-09-28
updated: 2026-09-28
tags:
  - topic/Linux
  - topic/Linux/定时任务
  - topic/日志
  - topic/MOC
status: to-review
aliases:
  - 定时任务知识地图
  - 日志轮转 MOC
  - cron 与 logrotate
---

# Linux 定时任务与日志轮转 — 知识地图

> [!abstract] 本笔记为 MOC（Map of Content）
> 运维自动化的两条主线：**谁来定时执行**（cron / systemd timer）与**日志如何不撑爆磁盘**（logrotate / 自定义清理）。
> 二者的结合点是「定时任务必须把输出写到**固定文件名**，才能交给 logrotate 管理」。

---

## cron 定时任务

| 主题 | 核心笔记 |
|:-----|:---------|
| **4 个高频陷阱（`%` 转义 / 相对路径 / stderr / PATH）** | [[Linux-cron定时任务常见陷阱]] |
| **脚本写法参考** | [[Shell-脚本备忘]] |

> [!danger] 头号陷阱
> crontab 中**第一个未转义的 `%` 会变成换行符**，其后的内容当作 stdin。
> 因此 `$(date +%F)` 在 crontab 里必须写成 `$(date +\%F)`；写在**脚本文件内部**则不需要转义。
> 结论：把逻辑放进脚本，crontab 只调用脚本。

---

## logrotate 日志轮转

| 主题 | 核心笔记 |
|:-----|:---------|
| **配置结构与 `rotate N` 语义** | [[Linux-logrotate-配置与rotate语义]] |
| **`-d` 输出解读与归档查看** | [[Linux-logrotate-调试与归档查看]] |
| **可用性与自动执行检查** | [[Linux-logrotate-可用性与自动执行检查]] |

> [!important] 三个必须记准的点
> 1. **`rotate 6` = 6 个归档 + 1 个当前日志 = 7 个文件**（不是 6 个文件）。
> 2. **最新的日志是 `xxx.log` 本身**，`.1` 是上一次，编号越小越新。
> 3. **logrotate 只能管理固定文件名的日志**；写入方按日期命名时，logrotate 匹配不到。

> [!tip] `copytruncate` 与 `create` 怎么选
> 写入进程不会重新打开日志文件时用 `copytruncate`（代价：有极短丢日志窗口）；
> 进程能响应信号重开文件时用 `create` + `postrotate`（无丢日志窗口）。

---

## 方案选型

| 主题 | 核心笔记 |
|:-----|:---------|
| **保留最新 N 个日志的三种方案对比** | [[保留最新N个日志文件的方案对比]] |

选型判据：**日志是否需要按天独立成文件**。

- 需要 → 「脚本 + cron + 按日期命名 + `find` 清理旧文件」。
- 不需要 → logrotate（声明式，最标准，无需自己写清理逻辑）。

---

## 容器与镜像清理（实战场景）

| 主题 | 核心笔记 |
|:-----|:---------|
| **`docker image prune -f` 与 `-af` 的区别** | [[Docker-image-prune镜像清理]] |
| **镜像分层诊断** | [[docker-history是审计日志不是文件清单]] |

> [!warning] `-a` 的删除范围常被低估
> `docker image prune -af` 删除的是**所有未被容器使用的镜像**，不只是 `<none>` 镜像。
> CI Runner 上会把准备复用的基础镜像一起删掉，导致下次构建重新拉取。
> 保守选择：`docker image prune -f`；需要激进回收时加 `--filter "until=168h"` 限定只删 7 天前的。

---

## 相关故障模式

| 现象 | 可能原因 | 关联笔记 |
|:-----|:---------|:---------|
| 磁盘被日志打满 | 日志从未轮转 / 轮转配置未生效 | [[Linux进程持有已删除文件句柄导致磁盘空间不释放]] |
| 删了日志但空间不释放 | 进程仍持有已删除文件的句柄 | [[案例-df与du磁盘空间不一致排查]] |
| CI 机器磁盘缓慢增长 | dangling 镜像堆积 | [[Docker-image-prune镜像清理]] |
| 定时任务「执行了但没效果」 | `%` 未转义 / 路径相对 / 丢了 stderr | [[Linux-cron定时任务常见陷阱]] |

---

## 落地检查清单

新增一个「定时清理 + 日志保留」任务时，逐条确认：

1. 清理命令的**删除范围**是否符合预期（`-f` 还是 `-af`，是否加 `until` 过滤）。
2. 输出目录已创建，且使用**绝对路径**。
3. 输出重定向为 `>> ... 2>&1`（追加 + 合并 stderr）。
4. 若命令写在 crontab 内，所有 `%` 是否已转义；**更好的做法是写进脚本**。
5. 日志切割由谁负责，**只保留一套机制**（按日期命名 或 logrotate，二选一）。
6. logrotate 配置写完后用 `logrotate -d` 验证。
7. 确认系统存在自动触发机制（`logrotate.timer` 或 `/etc/cron.daily/logrotate`）。
8. 最终用 `ls -1 <日志前缀>* | wc -l` 核对文件数量。

---

## 外部关联

- [[Linux-虚拟文件系统VFS]] — 日志文件所在的文件系统层知识
- [[Linux-Inode详解]] — `copytruncate` 与 inode 不变的关系
- [[lsof-文件诊断工具]] — 排查「删了文件空间不释放」
- [[Docker容器技术]] — 容器镜像清理相关知识的上级入口
- [[K8s容器运维]] — 容器场景的日志采集方案（Fluent Bit / VictoriaLogs）
- [[VictoriaLogs-日志保留策略]] — 集中式日志系统的保留策略（与文件级轮转互补）
