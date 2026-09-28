---
title: logrotate — 可用性与自动执行检查
date: 2026-09-28
tags:
  - topic/Linux
  - topic/Linux/定时任务
  - topic/日志
status: to-review
aliases:
  - 如何判断服务器能否使用 logrotate
  - logrotate 是否自动执行
  - logrotate 安装检查
---

# logrotate — 可用性与自动执行检查

## 结论

在新服务器上部署 logrotate 配置前，必须确认 **3 层**都成立，缺一层配置写对了也不会生效：

1. **命令已安装**（`logrotate` 可执行）。
2. **系统会自动触发**（systemd timer 或 `/etc/cron.daily/logrotate`，且 cron 服务在运行）。
3. **配置语法有效**（`logrotate -d` 无 error）。

---

## 一键检查脚本

```bash
echo "===== 1. logrotate 是否安装 ====="
command -v logrotate && logrotate --version

echo "===== 2. 包管理器确认 ====="
if command -v rpm >/dev/null 2>&1; then rpm -q logrotate; fi
if command -v dpkg >/dev/null 2>&1; then dpkg -s logrotate 2>/dev/null | grep -E 'Package|Status|Version'; fi
if command -v apk >/dev/null 2>&1; then apk info -e logrotate; fi

echo "===== 3. systemd 定时器 ====="
systemctl list-timers --all 2>/dev/null | grep -i logrotate || true
systemctl status logrotate.timer --no-pager 2>/dev/null || true

echo "===== 4. cron 每日任务 ====="
ls -l /etc/cron.daily/logrotate 2>/dev/null || true
systemctl status cron --no-pager 2>/dev/null || systemctl status crond --no-pager 2>/dev/null || true

echo "===== 5. 配置文件 ====="
ls -l /etc/logrotate.conf 2>/dev/null
ls -ld /etc/logrotate.d 2>/dev/null
```

判读要点：

| 检查项 | 通过的标志 |
|:-------|:-----------|
| 命令存在 | `command -v logrotate` 输出路径 |
| 包管理记录 | `rpm -q` / `dpkg -s` 显示已安装 |
| 自动触发（之一） | 存在 `logrotate.timer`，**或**存在 `/etc/cron.daily/logrotate` |
| 触发服务在运行 | cron/crond 服务处于 active，或 timer 处于 active |
| 配置目录存在 | `/etc/logrotate.d` 可读 |

---

## 各发行版安装方式

```bash
# Debian / Ubuntu
sudo apt-get update && sudo apt-get install -y logrotate

# RHEL / CentOS / Rocky / Alma
sudo yum install -y logrotate
# 或
sudo dnf install -y logrotate

# Alpine
sudo apk add logrotate
```

Alpine 等精简基础镜像默认不带 logrotate，且通常**没有 systemd 也没有 cron**，需要自行安装并额外提供触发机制。

---

## 自动触发机制说明

| 机制 | 检查方式 | 说明 |
|:-----|:---------|:-----|
| systemd timer | `systemctl status logrotate.timer`<br>`systemctl list-timers \| grep logrotate` | 主流发行版的默认方式 |
| cron.daily | `ls -l /etc/cron.daily/logrotate` | 传统方式，依赖 cron/crond 服务运行 |

两种机制**任一存在且服务在运行**即可。

> [!warning] 容器环境不适用
> 容器内通常没有 systemd，也不启动 cron。三种处理方式：
> 1. 日志目录挂载到宿主机，由**宿主机**的 logrotate 管理（最常见）。
> 2. 容器内安装 cron 并手动创建定时任务。
> 3. 由写入方（应用或脚本）自行轮转日志。
>
> 日志目录挂到宿主机时，宿主机 logrotate 配置里的路径要按**宿主机视角**书写，而不是容器内路径。

---

## 验证与执行

```bash
# 1. 语法与行为验证（dry-run，不改动文件）
logrotate -d /etc/logrotate.d/docker-image-prune

# 2. 强制真实轮转一次，确认效果（生产慎用）
logrotate -vf /etc/logrotate.d/docker-image-prune
```

输出解读见 [[Linux-logrotate-调试与归档查看]]。

注意：`logrotate -d` 显示的 `Creating new state` 等提示属于正常输出；只要没有 `error` / `syntax error`，配置即可用。

---

## 关联知识点

- [[Linux-logrotate-配置与rotate语义]] — 配置文件结构与 `rotate` 语义
- [[Linux-logrotate-调试与归档查看]] — 验证输出解读与归档查看
- [[保留最新N个日志文件的方案对比]] — 没有自动触发机制时的替代方案
- [[Linux-cron定时任务常见陷阱]] — 自行添加触发任务时的注意事项
