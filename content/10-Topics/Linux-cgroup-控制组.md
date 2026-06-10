---
title: Linux cgroup 控制组
date: 2026-06-02
tags:
  - topic/Linux
  - topic/VFS
  - topic/容器
  - topic/资源管理
status: to-review
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

## 可限制的资源类型（cgroup v2 完整控制器清单）

cgroup v2 支持以下资源控制器（controller），每个对应 `/sys/fs/cgroup/` 下的一个子系统目录：

### 1. CPU — CPU 时间分配

```bash
cpu.max                  # CPU 硬限制："配额 周期"（如 "50000 100000" = 0.5 核）
cpu.weight               # CPU 权重（相对优先级，范围 [1, 10000]，默认 100）
cpu.weight.nice          # 映射为 nice 值 [-20, 19] 方便兼容
cpu.stat                 # CPU 统计（usage_usec、system_usec、user_usec、nr_periods、nr_throttled、throttled_usec）
cpu.pressure             # CPU 压力滞胀信息（PSI）
```

### 2. Memory — 内存分配

```bash
memory.max               # 硬限制：最大内存使用（字节，写 "max" 取消限制）
memory.high              # 软限制：超过后开始回收/节流，但不杀进程
memory.low               # 低保障：不低于此值不会被回收（best-effort 保护）
memory.min               # 硬保障：不低于此值内核绝不动用（绝对保护）
memory.current           # 当前内存使用量
memory.swap.max          # 交换空间上限
memory.swap.current      # 当前交换用量
memory.oom.group         # OOM 时是否杀死整个 cgroup（1 = 是，0 = 只杀违章进程）
memory.numa_stat         # NUMA 节点内存分布
memory.stat              # 详细内存统计（anon、file、kernel 等分类）
memory.pressure          # 内存压力滞胀信息（PSI）
memory.events            # 事件计数（high 触发次数、max 触发次数、oom 次数）
```

### 3. IO — 块设备 I/O 控制

```bash
io.max                   # I/O 带宽硬限制（按设备，支持 BPS 和 IOPS，区分读写）
io.weight                # I/O 权重（相对优先级，范围 [1, 10000]）
io.stat                  # I/O 统计（读写次数、字节数等）
io.pressure              # I/O 压力滞胀信息（PSI）
io.latency               # I/O 延迟目标（如果内核编译时启用）
io.cost.*                # I/O 成本模型（io.cost.model、io.cost.qos，按设备）
```

### 4. PIDs — 进程/线程数量控制

```bash
pids.max                 # 最大进程+线程数（写 "max" 无限制）
pids.current             # 当前进程+线程数
pids.events              # 事件计数（超过上限的触发次数）
```

### 5. Cpuset — CPU 和 NUMA 内存节点绑定

```bash
cpuset.cpus              # 允许使用的 CPU 核列表（如 "0-3,7"）
cpuset.cpus.effective    # 实际生效的 CPU 列表（考虑热插拔）
cpuset.cpus.partition    # 分区模式："member"、"root"（隔离 CPU 用于实时负载）
cpuset.mems              # 允许使用的内存节点列表（NUMA node）
cpuset.mems.effective    # 实际生效的内存节点列表
```

> cpuset 和 cpu.weight 是互补的：cpuset 管"能跑在哪些核上"，cpu.weight 管"在这些核上占多少比例"。

### 6. HugeTLB — 大页内存限制

```bash
hugetlb.<size>.max       # 某个大页尺寸的最大数量（如 hugetlb.2MB.max）
hugetlb.<size>.current   # 当前使用的大页数量
hugetlb.<size>.events    # 大页限制触发事件
hugetlb.<size>.rsvd.*    # 预留大页统计
```

### 7. RDMA — 远程直接内存访问限制

```bash
rdma.max                 # RDMA 资源上限（按设备，如 "mlx5_0 hca_handle=2 hca_object=2000"）
rdma.current             # 当前 RDMA 资源用量
```

### 8. Misc — 杂项资源限制

用于管理非标准资源（Intel RDT / AMD PQoS、GPU 等），按 name 区分子资源：

```bash
misc.max                 # 杂项资源上限（如 "intel_l3cache.max=1024"）
misc.current             # 当前使用量
misc.capacity            # 总容量
```

### 9. ZRAM — 压缩内存写回控制（较新内核）

```bash
zram.max                 # ZRAM 写回上限
zram.current             # 当前 ZRAM 使用量
```

### 10. Perf Event — 性能监控

没有独立的限制文件，仅启用 cgroup 范围内的 perf 事件监控，用于统计分组进程的性能计数器。

### 11. 核心 cgroup 功能（不属于特定控制器，但通用）

```bash
cgroup.procs             # 组内线程组列表（写 PID 将整个线程组移入）
cgroup.threads           # 组内线程列表（可单独移动线程）
cgroup.controllers       # 该 cgroup 可用的控制器列表（只读）
cgroup.subtree_control   # 控制子组启用哪些控制器（写 "+memory +cpu"）
cgroup.type              # 节点类型（domain / domain threaded / threaded）
cgroup.freeze            # 冻结/解冻 cgroup（1 = 冻结所有进程，0 = 解冻）
cgroup.kill              # 杀死 cgroup 内所有进程（写入 1）
```

### 汇总对比

| 控制器 | 作用 | 典型限制文件 | 常见场景 |
|--------|------|-------------|---------|
| **cpu** | CPU 时间 | `cpu.max`, `cpu.weight` | 容器 CPU 配额 |
| **memory** | 内存 | `memory.max`, `memory.high` | 容器内存上限 |
| **io** | 磁盘 I/O | `io.max`, `io.weight` | 磁盘带宽限制 |
| **pids** | 进程数 | `pids.max` | 防 fork 炸弹 |
| **cpuset** | CPU/内存绑定 | `cpuset.cpus` | 绑核、实时优化 |
| **hugetlb** | 大页 | `hugetlb.2MB.max` | 大页数据库优化 |
| **rdma** | RDMA | `rdma.max` | 高性能计算 |
| **misc** | 杂项 | `misc.max` | GPU / RDT |
| **zram** | 压缩内存 | `zram.max` | 内存压缩写回 |
| **perf_event** | 性能监控 | — | 性能分析 |
| **freeze** | 冻结进程 | `cgroup.freeze` | 快照、检查点 |

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

- [[Docker-cgroup-v2-兼容性问题]] — Docker 旧版与 cgroup v2 的兼容性排查
- [[Linux-虚拟文件系统VFS]] — VFS 抽象层，cgroup2 是具体实现之一
- [[Linux-ext4文件系统]] — 真实文件系统对比
- [[Linux-挂载选项详解]] — cgroup 也涉及挂载命名空间等概念
