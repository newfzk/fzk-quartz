---
title: Docker cgroup v2 兼容性问题
date: 2026-06-09
tags:
  - topic/Docker
  - topic/Linux
  - topic/容器
  - topic/故障排查
  - topic/cgroup
status: to-review
aliases:
  - cgroups mountpoint does not exist
  - Docker cgroup v2 compatibility
  - Ubuntu 22.04 Docker
  - Docker version compatibility
---

# Docker cgroup v2 兼容性问题

## 问题现象

在 **Ubuntu 22.04+** 系统上运行 `docker run`，容器启动失败，报错：

```
docker: Error response from daemon: cgroups: cgroup mountpoint does not exist: unknown.
```

Docker daemon 无法找到 cgroup 挂载点，容器无法启动。

## 根因分析

### cgroup 版本演变

| 版本 | 特点 | 默认系统 |
|------|------|---------|
| **cgroup v1** | 每个资源类型独立层级，管理复杂 | CentOS 7、Ubuntu 20.04 |
| **cgroup v2** | 统一层级（unified hierarchy），更简洁 | **Ubuntu 22.04+**、Fedora、Debian 12+ |

Ubuntu 22.04 默认使用 **cgroup v2** 作为资源管理子系统。

### Docker 版本兼容性

```
┌─────────────────────────────────────────────────────────┐
│  Docker 版本  │  cgroup v1  │  cgroup v2               │
├─────────────────────────────────────────────────────────┤
│  < 20.10      │    ✅       │   ❌ 不支持              │
│  20.10 ~ 24.x │    ✅       │   ✅ 默认支持            │
│  25+          │    ⚠️ 遗留  │   ✅ 完全转向 v2         │
└─────────────────────────────────────────────────────────┘
```

**核心问题**：Docker **20.10 以下版本**（如 19.03.8）不支持 cgroup v2，而 Ubuntu 22.04 默认使用 cgroup v2，导致 Docker daemon 找不到 cgroup 挂载点。

> 具体来说，Docker 19.03 的 containerd / runc 只实现了 cgroup v1 的管理逻辑，尝试在 sysfs 中查找 v1 的挂载点时，发现系统只挂了 v2 → 报错退出。

## Docker 19 如何管理资源（没有 v2 时）

Docker 19.03 使用的是 **cgroup v1**，它的资源管理方式与 v2 完全不同：

```
cgroup v1（Docker 19.03 所用的方式）:

  每个资源类型有独立的层级树:

  /sys/fs/cgroup/
    ├── cpu/              ← CPU 限制子系统
    │   └── docker/<容器ID>/
    ├── memory/           ← 内存限制子系统
    │   └── docker/<容器ID>/
    ├── blkio/            ← 块设备 I/O 限制
    ├── cpuset/           ← CPU 绑定
    └── pids/             ← 进程数限制

  Docker 需要分别操作每个子系统:
    echo 512M > /sys/fs/cgroup/memory/docker/<ID>/memory.limit_in_bytes
    echo 50000 > /sys/fs/cgroup/cpu/docker/<ID>/cpu.shares
```

**cgroup v1 的特点：**
- 每个子系统（cpu、memory、blkio...）有**独立的挂载点**
- 同一个进程需要在每个子系统下分别注册
- 管理复杂，容易产生不一致状态
- 这是 Docker 19.x 时代的标准方案

## 为什么 Ubuntu 16 可以正常跑？

```
Ubuntu 16.04 LTS  (Linux 4.4 / 4.15)
    │
    └── cgroup v1  ← 当时的默认配置
            │
            └── /sys/fs/cgroup/cpu/      ✅
            └── /sys/fs/cgroup/memory/   ✅
            └── /sys/fs/cgroup/blkio/    ✅
                    │
                    └── Docker 19.03 运行正常 ✓

Ubuntu 22.04 LTS  (Linux 5.15+)
    │
    └── cgroup v2  ← 新系统标配
            │
            └── /sys/fs/cgroup/  （统一层级，无子子系统目录）
                    │
                    └── Docker 19.03 运行时:
                        └── 找 /sys/fs/cgroup/cpu/    → 没有 ❌
                        └── 找 /sys/fs/cgroup/memory/ → 没有 ❌
                        └── 崩溃: "cgroup mountpoint does not exist"
```

**关键点：** 不是 Docker 19 的代码坏了，而是它**只"认识" cgroup v1 的目录布局**，遇到 v2 的统一布局就直接迷路了。

## cgroup 版本确认命令（大全）

### 1. 🏆 最简洁 — stat 命令

```bash
stat -fc %T /sys/fs/cgroup/
```

| 输出值 | 含义 | 判断 |
|--------|------|------|
| `cgroup2fs` | cgroup v2 | ✅ 当前系统使用 cgroup v2 |
| `tmpfs` | cgroup v1 | ✅ 当前系统使用 cgroup v1 |

> `%T` 表示文件系统类型（File system Type），这是区分 v1/v2 的最快方法。

### 2. 查看挂载详情 — mount

```bash
mount | grep cgroup
```

**cgroup v2 系统上输出（只有一行）：**
```
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate...)
```

**cgroup v1 系统上输出（多个子系统各一行）：**
```
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,relatime,xattr,name=systemd)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,relatime,memory)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,relatime,blkio)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,relatime,pids)
...
```

### 3. 查看目录结构 — ls

```bash
ls -la /sys/fs/cgroup/
```

**cgroup v2**：直接看到 `memory.max`、`cpu.max` 等控制文件
```
-r--r--r--   1 root root 0 Jun  9 10:00 memory.max
-rw-r--r--   1 root root 0 Jun  9 10:00 memory.high
```

**cgroup v1**：看到的是子系统目录
```
dr-xr-xr-x  2 root root 0 Jun  9 10:00 cpu/
dr-xr-xr-x  2 root root 0 Jun  9 10:00 memory/
dr-xr-xr-x  2 root root 0 Jun  9 10:00 blkio/
```

### 4. 检查内核是否支持

```bash
# 查看内核是否编译了 cgroup v2
grep cgroup /proc/filesystems

# 输出包含:
# cgroup    ← 内核支持 cgroup v1
# cgroup2   ← 内核支持 cgroup v2
```

### 5. Docker 相关诊断

```bash
# Docker 版本
docker --version

# Docker 当前使用的 cgroup driver
docker info | grep -i cgroup
# cgroup v2 + Docker 20.10+ → Cgroup Driver: systemd
# cgroup v1                → Cgroup Driver: cgroupfs
```

## 解决方案

### 方案一：升级 Docker（推荐 ✅）

将 Docker Engine 升级到 **20.10+**（建议 24+），获得完整的 cgroup v2 支持：

```bash
# 卸载旧版本
sudo apt remove docker docker-engine docker.io containerd runc

# 安装最新 Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 验证版本
docker --version
# Docker version 24.x.x+

# 确认 cgroup driver
docker info | grep -i cgroup
# 预期: Cgroup Driver: systemd
```

### 方案二：回退到 cgroup v1（不推荐 ⚠️）

如果无法升级 Docker，可通过内核参数强制使用 cgroup v1：

```bash
# 编辑 GRUB 配置
sudo vim /etc/default/grub

# 在 GRUB_CMDLINE_LINUX 中添加：
# systemd.unified_cgroup_hierarchy=0

# 更新 GRUB 并重启
sudo update-grub
sudo reboot

# 验证
stat -fc %T /sys/fs/cgroup/
# 输出: tmpfs → cgroup v1
```

> **为什么不推荐？** cgroup v1 是旧方案，Ubuntu 22.04 的 systemd 和内核都已面向 v2 优化。长期来看所有容器工具都会抛弃 v1，升级 Docker 才是正道。

### 方案三：配置 Docker daemon 的 cgroup driver（辅助手段）

Docker 启动后还需确保 cgroup driver 与 systemd 匹配：

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

sudo systemctl restart docker
```

> ⚠️ 如果 Docker 版本本身 < 20.10，仅设置 driver 无法解决底层 containerd / runc 不支持 v2 的问题，必须先升级。

## 完整排查案例

```
场景: Ubuntu 22.04.2 + Docker 19.03.8
命令: docker run --detach --name kubeasz easzlab/kubeasz sleep 36000

报错: Error response from daemon: cgroups: cgroup mountpoint does not exist.

排查流程:
  1️⃣ docker --version → 19.03.8（已确认低版本）
  2️⃣ stat -fc %T /sys/fs/cgroup/ → cgroup2fs（确认 v2）
  3️⃣ 根因: Docker 19.03 不支持 cgroup v2
  4️⃣ 解决: 升级 Docker → 容器正常启动 ✅
```

## 相关笔记

- [[Linux-cgroup-控制组]] — cgroup 基础概念与 v1/v2 对比
- [[Docker网络模式-bridge]] — Docker 网络模式
