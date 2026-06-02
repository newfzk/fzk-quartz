---
title: Linux cgroup 控制组
date: 2026-06-02
tags:
  - topic/Linux
  - topic/VFS
  - topic/容器
  - topic/资源管理
status: evergreen
aliases:
  - cgroup
  - cgroup2
  - 控制组
  - Control Group
  - 资源隔离
---

# Linux cgroup 控制组

## 名字由来：Control Group

**cgroup** = **Control**（控制）+ **Group**（组）

| 部分 | 含义 |
|------|------|
| **c** | Control（控制）—— 对资源进行限制、统计、隔离 |
| **group** | 组 —— 把相关进程归为一组统一管理 |

合起来就是 **"把进程分组，然后控制每组的资源使用"**。

> 给进程画一个个"笼子"：这个笼子里的进程最多用 512MB 内存，那个笼子里的进程最多用 2 个 CPU 核心。

## cgroup 版本

当前主流是 **cgroup v2**（Linux 4.5+ 引入，多数现代发行版默认使用）：

| 版本 | 特点 | 典型系统 |
|------|------|---------|
| **v1** | 每个资源类型独立层级，管理复杂 | 旧版 CentOS 7 |
| **v2** | 统一层级（unified hierarchy），更简洁 | Ubuntu 22.04+、Fedora、Debian 12+ |

> cgroup v2 挂载在 `/sys/fs/cgroup/`，所以它常被归类为"虚拟文件系统"——你通过文件操作来配置资源限制。

## 核心概念

```
cgroup 的工作流程：

1. 创建组     mkdir /sys/fs/cgroup/my-group
              └── 在内存中创建一个"控制组"目录

2. 设置限制   echo 512M > /sys/fs/cgroup/my-group/memory.max
              └── 写入限制参数（就像写配置文件）

3. 加入进程   echo 12345 > /sys/fs/cgroup/my-group/cgroup.procs
              └── 把 PID 12345 加入这个组

4. 自动管控   ── 内核现在限制该进程的内存使用
              超过 512MB → OOM 杀死
```

> **把 cgroup 想象成酒店的房间分配**：
> - 创建组 = 开一个房间
> - 设置限制 = 定房规（最多住2人、不能开派对）
> - 加入进程 = 客人入住
> - 超限惩罚 =违规清退

## 可限制的资源类型

```bash
ls /sys/fs/cgroup/
# 你会看到很多控制文件：

# CPU 控制
cpu.max              # CPU 使用上限（如 "200000 100000" = 2 核）
cpu.weight           # CPU 权重（相对优先级）

# 内存控制
memory.max           # 内存使用上限（字节）
memory.current       # 当前内存使用量
memory.high          # 内存软限制（超过后开始回收）
memory.low           # 内存保障（不低于此值不会被回收）
memory.oom.group     # OOM 时是否杀死整个组

# I/O 控制
io.max               # 磁盘 I/O 上限
io.weight            # I/O 优先级

# 进程控制
pids.max             # 最大进程/线程数
cgroup.procs         # 组内的进程列表
cgroup.controllers   # 可用的控制器
```

## 经典例子

### 1. 限制进程内存使用

```bash
# 创建一个控制组
sudo mkdir /sys/fs/cgroup/demo-group

# 设置最大内存 256MB
echo 268435456 | sudo tee /sys/fs/cgroup/demo-group/memory.max

# 启动一个测试程序（不断申请内存）
# 先查看当前 shell 的 PID
echo $$
# 输出: 12345

# 把当前 shell 加入控制组
echo 12345 | sudo tee /sys/fs/cgroup/demo-group/cgroup.procs

# 现在在这个 shell 中运行任何命令，总内存不能超过 256MB
# 如果程序试图超过这个限制，会被 OOM 杀死
```

### 2. 限制进程的 CPU 使用

```bash
# 创建 CPU 控制组
sudo mkdir /sys/fs/cgroup/cpu-demo

# 限制最多使用 0.5 个 CPU 核心
# 格式: "配额 周期"，周期通常 100000μs = 100ms
echo "50000 100000" | sudo tee /sys/fs/cgroup/cpu-demo/cpu.max

# 运行压力测试
# 在另一个终端启动: sha1sum /dev/zero &
# 找到 PID 并加入控制组
echo <PID> | sudo tee /sys/fs/cgroup/cpu-demo/cgroup.procs

# 用 top 观察，该进程 CPU 使用率不会超过 50%
```

### 3. 限制进程数（防 fork 炸弹）

```bash
# 创建 PIDs 控制组
sudo mkdir /sys/fs/cgroup/pids-demo

# 最多允许 10 个进程
echo 10 | sudo tee /sys/fs/cgroup/pids-demo/pids.max

# 把当前 shell 加进去
echo $$ | sudo tee /sys/fs/cgroup/pids-demo/cgroup.procs

# 现在在这个 shell 中，最多只能创建 10 个进程
# 第 11 个 fork() 会返回 EAGAIN 错误
```

## cgroup 与现代基础设施

cgroup 是容器技术的**底层基础**：

```
Docker 容器
    │
    ├── 创建 cgroup → docker/<容器ID>/
    │
    ├── 设置限制 → memory.max = 512M
    │              cpu.max = 100000 100000（1核）
    │              pids.max = 100
    │
    └── 放入进程 → 容器的 init 进程
                     │
                     └── 所有子进程自动继承

所以当你运行:
  docker run --memory=512m --cpus=1 nginx

Docker 内部做的是:
  1. mkdir /sys/fs/cgroup/docker/<容器ID>/
  2. echo 536870912 > memory.max
  3. echo 100000 100000 > cpu.max
  4. echo <PID> > cgroup.procs
```

类似地，**Kubernetes**、**systemd**、**lxc** 等都用 cgroup 做资源管理。

## 为什么 cgroup 是个"文件系统"？

```bash
# 因为它的操作方式完全是"文件操作"
cd /sys/fs/cgroup

# 增 创建组 = 创建目录
mkdir my-group

# 删 删除组 = 删除目录
rmdir my-group    # 需要先清空组内进程

# 改 修改限制 = 写入文件
echo "512M" > my-group/memory.max

# 查 查看用量 = 读取文件
cat my-group/memory.current
```

这种设计让 **shell 脚本就能管理资源**，不需要专门的命令行工具（虽然也有 `cgcreate`、`cgset` 等工具）。

## 相关笔记

- [[Linux-虚拟文件系统VFS]] — VFS 抽象层，cgroup2 是具体实现之一
- [[Linux-ext4文件系统]] — 真实文件系统对比
- [[Linux-挂载选项详解]] — cgroup 也涉及挂载命名空间等概念
