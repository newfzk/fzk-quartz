---
title: Ubuntu 查看系统版本号
date: 2026-06-09
tags:
  - topic/Linux
  - topic/Ubuntu
  - topic/命令
status: to-review
aliases:
  - lsb_release
  - Ubuntu版本查询
---

# Ubuntu 查看系统版本号

## 常用命令汇总

| 命令 | 说明 | 输出示例 |
|------|------|---------|
| `lsb_release -a` | 显示完整版本信息（**最推荐**） | `Description: Ubuntu 24.04 LTS` |
| `lsb_release -r` | 仅显示版本号 | `Release: 24.04` |
| `cat /etc/os-release` | 查看 OS 标识与版本变量 | `VERSION_ID="24.04"` |
| `cat /etc/lsb-release` | 查看 LSB 兼容版本信息 | `DISTRIB_RELEASE=24.04` |
| `hostnamectl` | 查看系统主机名及 OS 信息 | `Operating System: Ubuntu 24.04 LTS` |

---

## 命令详解

### 1. `lsb_release` — 标准方法

安装 LSB（Linux Standard Base）工具集后可用，通常**预装**在桌面版 Ubuntu 中。

```bash
# 显示全部版本信息
lsb_release -a

# 仅显示版本号
lsb_release -r

# 仅显示发行版代号
lsb_release -c
```

> **Tip:** 如果提示 `command not found`，可安装 `lsb-release` 包：
> ```bash
> sudo apt install lsb-release
> ```

### 2. `/etc/os-release` — 通用方法

所有 systemd 系统均包含此文件，**无需额外安装任何包**，适用于脚本中解析。

```bash
cat /etc/os-release
```

典型输出：
```
PRETTY_NAME="Ubuntu 24.04 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04 LTS (Noble Numbat)"
VERSION_CODENAME=noble
ID=ubuntu
```

在脚本中获取版本号：
```bash
source /etc/os-release
echo "$VERSION_ID"   # 输出: 24.04
```

### 3. `/etc/lsb-release` — 传统方法

```bash
cat /etc/lsb-release
```

输出格式：
```
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=24.04
DISTRIB_CODENAME=noble
DISTRIB_DESCRIPTION="Ubuntu 24.04 LTS"
```

### 4. `hostnamectl` — 系统信息汇总

```bash
hostnamectl | grep "Operating System"
# 或直接执行 hostnamectl
```

---

## 版本号含义

Ubuntu 版本号格式为 **YY.MM**，即**发行年份.月份**：

| 版本 | 代号 | 说明 |
|------|------|------|
| 24.04 | Noble Numbat | 2024年4月发布，LTS（长期支持） |
| 22.04 | Jammy Jellyfish | 2022年4月发布，LTS |
| 20.04 | Focal Fossa | 2020年4月发布，LTS |

- LTS（Long Term Support）版本每两年发布一次，支持 5 年（含免费安全更新）
- 非 LTS 版本每 6 个月发布一次，仅支持 9 个月

---

## 场景与应用

| 场景 | 推荐命令 |
|------|---------|
| 日常查看 | `lsb_release -a` |
| Shell 脚本中获取版本号 | `source /etc/os-release && echo "$VERSION_ID"` |
| 确认是否 LTS | `cat /etc/os-release \| grep SUPPORT` |
| 快速查看代号 | `lsb_release -c` |
| Docker 容器内查看 | `cat /etc/os-release` |

---

## 关联笔记

- [[Linux-常用命令速查]] — Linux 系统管理命令合集
- [[stat-命令详解]] — 查看文件/文件系统元数据
- [[Docker-cgroup-v2-兼容性问题]] — Ubuntu 22.04 cgroup v2 迁移相关背景
- [[iptables详解]] — 涉及 Ubuntu 中 iptables 的包管理方式
