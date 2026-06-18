---
title: Linux sysfs 文件系统
date: 2026-06-02
aliases:
  - sysfs
  - /sys
  - 系统文件系统
  - System Filesystem
tags:
  - topic/Linux
  - topic/Linux/文件系统
status: to-review
---

# Linux sysfs 文件系统

## 名字由来：Sys = System

**sysfs** 的名称来自 **System Filesystem**（系统文件系统）。它的职责是**将内核的硬件设备模型（kobject）以目录和文件的形式暴露给用户态**。

> procfs 是"进程面板"，sysfs 是**"硬件拓扑地图"**——你打开 /sys，就能看到系统里所有硬件设备以及它们之间的连接关系。

## 历史背景：为什么要发明 sysfs？

早期 Linux 把所有信息塞进 [[Linux-proc-文件系统\|procfs]]，包括硬件信息（如 `/proc/pci`、`/proc/bus/`）。但这导致两个问题：

1. **职责混乱**：proc 本应是进程信息，却混杂了大量硬件数据
2. **结构混乱**：硬件之间存在树形关系（PCI 总线 → 设备 → 驱动），但 proc 是扁平文件

**Linux 2.6（2003 年）引入 sysfs**，把硬件信息从 proc 中剥离，用目录树清晰表达硬件拓扑。

> **记忆技巧**：`proc` = 进程（软件视角），`sys` = 系统硬件（硬件视角）。一个管进程，一个管设备。

## 核心结构

sysfs 的目录结构反映了内核对硬件的组织方式：

```
/sys/
├── block/        # 块设备（磁盘、分区）
│   ├── sda/
│   │   ├── sda1/
│   │   └── sda2/
│   └── sdb/
├── bus/          # 总线类型
│   ├── pci/
│   ├── usb/
│   └── i2c/
├── class/        # 设备分类
│   ├── net/      # 网络设备
│   ├── input/    # 输入设备
│   ├── sound/    # 音频设备
│   └── tty/      # 终端设备
├── devices/      # 所有设备的真实树形结构
└── kernel/       # 内核参数
```

这三个分类目录**从不同视角看同一套硬件**：

```
         硬盘 /dev/sda
         ┌─────────┐
         │         │
    /sys/block/sda    /sys/bus/pci/devices/...    /sys/class/block/sda
   （按设备类型）      （按总线连接关系）           （按功能分类）
```

## 经典例子

### 1. 查看网络设备

```bash
# 列出所有网络设备
ls /sys/class/net/
# 输出: enp3s0  lo  wlp2s0

# 查看网卡详细信息
cat /sys/class/net/enp3s0/speed     # 连接速度（Mbps）
# 输出: 1000

cat /sys/class/net/enp3s0/duplex    # 全双工/半双工
# 输出: full

cat /sys/class/net/enp3s0/address   # MAC 地址
# 输出: 00:1a:2b:3c:4d:5e

cat /sys/class/net/enp3s0/operstate # 运行状态
# 输出: up
```

### 2. 查看 CPU 详细信息

```bash
# 每个 CPU 核心一个目录
ls /sys/devices/system/cpu/
# cpu0  cpu1  cpu2  cpu3  ...

# 查看 CPU0 的缓存信息
cat /sys/devices/system/cpu/cpu0/cache/index0/size  # L1 缓存
# 输出: 32K
cat /sys/devices/system/cpu/cpu0/cache/index1/size  # L1 指令缓存
cat /sys/devices/system/cpu/cpu0/cache/index2/size  # L2 缓存
cat /sys/devices/system/cpu/cpu0/cache/index3/size  # L3 缓存（多核共享）
# 输出: 8192K

# 查看 CPU 频率
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

### 3. 查看块设备

```bash
# 查看硬盘参数
cat /sys/block/sda/size                     # 总扇区数
cat /sys/block/sda/queue/rotational         # 是否机械盘（1=HDD, 0=SSD）
cat /sys/block/sda/queue/scheduler          # I/O 调度器
# 输出: [mq-deadline] kyber  bfq

# 查看分区
ls /sys/block/sda/
# sda1  sda2  sda5  ...
```

### 4. 查看驱动与总线关系

```bash
# 查看 PCI 总线上的设备
ls /sys/bus/pci/devices/
# 0000:00:00.0  0000:00:02.0  0000:00:14.0 ...

# 查看某个 PCI 设备
cat /sys/bus/pci/devices/0000:00:02.0/vendor     # 厂商 ID
cat /sys/bus/pci/devices/0000:00:02.0/device     # 设备 ID
cat /sys/bus/pci/devices/0000:00:02.0/driver     # 绑定的驱动

# 查看 USB 设备
ls /sys/bus/usb/devices/
```

### 5. 实战：查看 GPU 信息

```bash
# 查看显卡（通常在 PCI 设备中）
ls /sys/class/drm/
# card0  card0-HDMI-A-1  card0-eDP-1  ...

cat /sys/class/drm/card0/device/vendor
cat /sys/class/drm/card0/device/device
```

## 关键特性：属性文件（Attribute）

sysfs 中的大部分文件是**属性文件（attribute）**，每次读取内核**实时生成**值：

```bash
# sysfs 中的属性文件通常只返回一行数据
cat /sys/class/net/lo/mtu
# 输出: 65536

# 有些属性文件可写，用于配置
echo 1500 > /sys/class/net/enp3s0/mtu    # 修改 MTU
```

这与 [[Linux-proc-文件系统\|procfs]] 不同——proc 更关注"进程级"的状态，sysfs 更关注"设备级"的属性和拓扑。

## sysfs 与 procfs 对比

| 维度 | sysfs | procfs |
|------|-------|--------|
| **名字含义** | System（系统硬件） | Process（进程） |
| **挂载点** | `/sys` | `/proc` |
| **数据内容** | 硬件拓扑、设备属性、驱动信息 | 进程信息、内核统计 |
| **数据结构** | 目录树（反映设备连接关系） | 目录（数字 PID + 系统文件） |
| **引入时间** | Linux 2.6（2003） | Linux 0.99（1992） |
| **典型路径** | `/sys/class/net/eth0/` | `/proc/12345/status` |

## 相关笔记

- [[Linux-虚拟文件系统VFS]] — VFS 抽象层，sysfs 是具体实现之一
- [[Linux-proc-文件系统]] — 互补的"进程"虚拟文件系统
- [[Linux-磁盘分区与设备命名]] — 块设备命名规则
