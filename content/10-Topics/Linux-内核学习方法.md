---
tags:
  - topic/Linux
  - topic/Linux/内核
  - learning/methodology
status: to-review
---

# Linux 内核学习方法

## 核心思路

Linux 内核学习不走"从头啃源码"的路线，而是 **先搭骨架，再以工作中遇到的具体概念为锚点，辐射式深挖相关子系统**。

---

## 一、五层地图（内核全景骨架）

把内核想象成五层大楼，每个新概念先归位：

| 层次 | 核心职责 | 日常工作关联 |
|------|----------|-------------|
| **系统调用层** | 用户程序与内核的交互入口 | `open()`、`socket()`、`ioctl()` |
| **进程管理** | 任务调度、信号、进程/线程生命周期 | `top`、`kill`、容器 namespace/cgroup |
| **内存管理** | 虚拟内存、物理页面分配、缺页处理 | OOM、`free -h`、mmap |
| **文件系统层 (VFS)** | 统一抽象各文件系统操作，**inode 属于此层** | ext4/xfs/proc/sysfs |
| **网络协议栈** | 套接字、TCP/IP、Netfilter 框架，**ipset 在此层** | iptables、容器网络 |

> 速记口诀：**进程调度扛 CPU，内存管理分页忙；文件系统看 inode，网络包走 Netfilter。**

---

## 二、锚点穿透法

以日常接触的概念为抓手，透视对应子系统：

### inode → 文件系统层

```
VFS 四核心对象：
  super_block — 挂载的文件系统（如 ext4）
  inode      — 文件实体（元数据 + 数据块指针）
  dentry     — 目录项（文件名 ↔ inode 的映射）
  file       — 打开的文件上下文（当前读写位置）
```

实验感知：`stat`、`ls -i`、`df -i` 查看 inode 使用情况。

### ipset → 网络栈

```
ipset = 内核中带哈希/树结构的 IP 地址集合
       → 被 Netfilter 使用
       → O(1) 哈希查找（远快于 iptables 线性匹配）
```

实验：`ipset create` → `iptables -m set` 引用。

---

## 三、阶梯式学习方案

### 第一周：建立全景 + 操作感
- 读 **《Linux 内核设计与实现》(LKD) 第3版** 前 4 章（进程管理、系统调用、VFS、网络栈简介）
- 搭建实验环境（虚拟机 + `kernel-devel` + `bcc-tools`）
- 编译最小内核模块（几十行代码，破除神秘感）
- 通读 `man proc`

### 第二周：概念卡法各个击破
工作中遇到新概念（如 cgroup、epoll、RCU），制作**概念卡片**：

1. 属于五层地图哪一层？
2. 解决什么问题？（一句话）
3. 用户空间表现（命令/系统调用）
4. 内核关键数据结构/函数（去 [elixir.bootlin.com](https://elixir.bootlin.com) 搜源码）

### 第三周起：图形化追踪
- 用 `bcc`/`bpftrace` 透视系统行为（`opensnoop`、`tcptracer`）
- 用 `strace` 跟踪系统调用链

---

## 四、推荐资料

| 资料 | 定位 |
|------|------|
| **《Linux 内核设计与实现》(LKD) 第三版** | 最佳全景入门，厚度适中 |
| **《深入 Linux 内核架构》** | LKD 之后进阶，深入子系统设计 |
| **《Linux 设备驱动程序》(LDD3) 前 4 章** | 理解内核模块与 /proc |
| **[elixir.bootlin.com](https://elixir.bootlin.com)** | 在线源码交叉引用，查结构体定义 |
| **Linux Kernel Labs** | 官方带练习的内核实验环境 |

---

## 五、微习惯起步计划

- **Day1**：`man proc` + `ls /proc` 浏览内核信息接口
- **Day2**：`man 7 inode` + `stat`/`df -i`，画出 VFS 四对象关系图
- **Day3**：`ipset` 创建集合并挂到 iptables，用 `-v` 看计数
- **Day4**：编译含 `printk` 的内核模块，看 `dmesg` 输出
- **Day5**：读 LKD 第 13 章（VFS）和第 18 章（网络）
- **持续**：遇到新概念 → 打开 elixir 看结构体定义 → 补概念卡片

> 一个月内可从"零散名词"进入"有骨架的知识体系"。

## 关联笔记

- [[Linux-Inode详解]] — inode 数据结构与使用
- [[netfilter框架详解]] — Netfilter 内核框架
- [[iptables详解]] — iptables 用户空间工具
- [[Linux-cgroup-控制组]] — 容器资源控制
