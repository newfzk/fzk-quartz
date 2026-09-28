---
title: docker image prune — 镜像清理范围与过滤条件
date: 2026-09-28
tags:
  - topic/Docker
  - topic/容器
status: to-review
aliases:
  - docker image prune -a 与 -f 的区别
  - none 镜像清理
  - dangling 镜像
---

# docker image prune — 镜像清理范围与过滤条件

## 结论

| 命令 | 删除范围 | 风险 |
|:-----|:---------|:-----|
| `docker image prune -f` | 只删 **dangling（`<none>`）** 镜像 | 低 |
| `docker image prune -af` | 删除**所有未被容器使用**的镜像（含仍有 tag 的） | 高，可能删掉下次构建要复用的基础镜像 |

**优先用不带 `-a` 的 `docker image prune -f`，除非明确要回收所有未使用镜像。**

---

## dangling 镜像（`<none>`）从哪来

`docker images` 中显示为 `<none> <none>` 的镜像即 dangling 镜像：**有层数据，但没有 tag 也没有被任何镜像引用**。

常见来源：

- 同一个 tag 被重新构建并覆盖，旧镜像失去 tag。
- CI Runner（如 GitLab Runner）每次构建产生的临时层，构建结束后不再被引用。
- 构建中断（`docker build` 失败或 Ctrl+C）留下的中间产物。

这类镜像不影响运行，但会持续占用磁盘，是 CI 机器磁盘告警的常见根因。

查看：

```bash
docker images -f dangling=true
docker image ls -f dangling=true -q | wc -l   # 只统计数量
```

---

## `-a` 与默认行为的区别

```bash
# 只删 dangling 镜像
docker image prune -f

# 删除所有未被容器使用的镜像（dangling + 有 tag 但没容器用）
docker image prune -af
```

> [!danger] `-a` 的真实影响范围常被低估
> `-a` 不只是「多删一点 `<none>`」，而是**删除一切没有容器引用的镜像**。在 CI 机器上，这包括：
> - 手工 `docker pull` 下来、准备下次构建复用的基础镜像；
> - 已构建完成、等待部署的带 tag 业务镜像。
>
> 删掉之后下次构建会重新拉取，在网络受限的环境里意味着构建时间显著变长，甚至失败。

`-f` 只是**跳过交互确认**，不改变删除范围。定时任务中必须加，否则会卡在确认提示。

---

## 时间过滤：`--filter "until=..."`

避免「删得太新」，只清理创建时间足够久的镜像：

```bash
# 只删除创建超过 7 天（168 小时）的未使用镜像
docker image prune -af --filter "until=168h"
```

- `until` 的语义是「创建时间早于该时间点」。
- 支持 `24h`、`168h` 这类时长写法，也支持时间戳。
- 与 `-a` 组合使用，可把「删除范围」和「删除时机」两个维度分开控制。

其他可用过滤条件：`--filter "label=..."`、`--filter "dangling=true"`。

---

## 定时清理的推荐组合

```bash
# 保守方案：只清 dangling，无时间限制
docker image prune -f

# 激进方案：清理所有超过 7 天未使用的镜像
docker image prune -af --filter "until=168h"
```

配合 crontab 时注意 `2>&1`、绝对路径与 `%` 转义，详见 [[Linux-cron定时任务常见陷阱]]。

---

## 关联知识点

- [[Docker容器技术]] — Docker 知识地图（MOC）
- [[Linux-cron定时任务常见陷阱]] — 把清理命令放进 crontab 的注意事项
- [[保留最新N个日志文件的方案对比]] — 清理任务的日志保留策略
- [[Dockerfile-构建缓存优化]] — 构建缓存与镜像层复用（`-a` 删除会影响复用）
- [[docker-history是审计日志不是文件清单]] — 镜像分层诊断的相邻知识点
- [[Linux进程持有已删除文件句柄导致磁盘空间不释放]] — 另一种「空间没释放」的原因（镜像清理无效时的排查方向）
