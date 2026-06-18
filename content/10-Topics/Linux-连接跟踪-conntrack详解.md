---
title: Linux 连接跟踪 — conntrack 详解
date: 2026-06-10
aliases:
  - Connection Tracking
  - nf_conntrack
  - 连接跟踪
tags:
  - topic/Linux
  - topic/计算机网络
status: to-review
---

## 概述

**连接跟踪（Connection Tracking，简称 conntrack）** 是 Linux 内核 netfilter 框架中最核心的子模块之一。它跟踪所有经过系统的网络连接状态，为 **NAT**、**状态防火墙**、**连接数限制**等功能提供基础。

> conntrack 是许多网络功能的前提条件：NAT 依赖它来记录地址转换的映射关系，`-m state` 状态匹配依赖它来识别包属于哪个连接，`-m connlimit` 连接数限制也依赖它来计数。

---

## 一、工作原理

### 1.1 Hook 注册位置

conntrack 模块（`nf_conntrack.ko`）注册到 netfilter 的两个关键 Hook 点：

| Hook 点 | 时机 | 优先级 |
|---------|------|--------|
| `NF_INET_PRE_ROUTING` | 数据包进入 IP 层后、路由决策前 | -200 |
| `NF_INET_LOCAL_OUT` | 本机进程发出数据包时 | -200 |

> [!tip] 高优先级
> 优先级 -200 是所有 netfilter 功能中最高的之一，确保了 conntrack 在包过滤、NAT 等操作**之前**执行。详见 [[netfilter框架详解#2.4 优先级系统]]。

### 1.2 核心数据结构

#### nf_conn — 连接跟踪条目

```c
struct nf_conn {
    struct nf_conntrack          ct_general;  // 引用计数
    struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX]; // 双向元组（原始/应答）
    struct nf_conn_help          *helper;     // 协议辅助（如 FTP、SIP）
    struct nf_conn_nat           *nat;        // NAT 信息
    struct nf_conn_seqadj        *seqadj;     // TCP 序列号调整
    u32                          mark;        // ctmark
    unsigned long                status;      // 状态位
    unsigned long                timeout;     // 超时时间
};
```

#### nf_conntrack_tuple — 连接标识（五元组）

```c
struct nf_conntrack_tuple {
    struct nf_conntrack_man_proto src;     // 源端口
    struct nf_conntrack_man_proto dst;     // 目标端口
    union nf_inet_addr           src_addr; // 源 IP
    union nf_inet_addr           dst_addr; // 目标 IP
    u_int8_t                     protocol; // L4 协议（TCP/UDP/ICMP...）
};
```

### 1.3 处理流程

```
数据包到达
    │
    ▼
PRE_ROUTING（或 LOCAL_OUT）
    │
    ├──→ conntrack 进行状态查找 / 创建 nf_conn 条目
    │       │
    │       ├── NEW:       创建新条目（首包）
    │       ├── ESTABLISHED: 更新已有条目（后续包）
    │       ├── RELATED:   创建关联条目（如 FTP 数据通道）
    │       └── INVALID:   标记为无效
    │
    ▼
后续处理（路由、过滤、NAT...）
```

---

## 二、四大连接状态

| 状态 | 含义 | 典型场景 |
|------|------|---------|
| **NEW** | 新建连接 | 收到第一个包（通常是 TCP SYN），尚未确认连接是否完整建立 |
| **ESTABLISHED** | 已建立连接的后续包 | 双向都看到了包，连接已建立完成 |
| **RELATED** | 与已有连接相关的辅助连接 | FTP 数据通道（基于 FTP 控制通道创建）、ICMP 差错报文 |
| **INVALID** | 无法识别的包 | 状态异常、无法匹配任何已有连接的包，通常应丢弃 |

> [!tip] 状态防火墙的核心是 ESTABLISHED + RELATED
> 放行 `-m state --state ESTABLISHED,RELATED` 可以让所有合法回包通过，同时只放行白名单端口上的 NEW 连接，是最常用的防御手段。

---

## 三、在 iptables 中使用 conntrack

### 3.1 状态匹配（`-m state`）

```bash
# 允许已建立连接的后续包回包（最常用的状态防火墙规则）
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 只允许 SSH 新建连接
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -j ACCEPT

# 丢弃无效包
iptables -A INPUT -m state --state INVALID -j DROP
```

### 3.2 raw 表绕过 conntrack（NOTRACK）

对于高吞吐场景（如 DNS 服务器），conntrack 可能成为性能瓶颈。可以在 **raw 表** 中设置 `NOTRACK` 目标，跳过对特定流量的连接跟踪：

```bash
# 不对 DNS 流量做连接跟踪（提升性能）
iptables -t raw -A PREROUTING -p udp --dport 53 -j NOTRACK
iptables -t raw -A OUTPUT -p udp --dport 53 -j NOTRACK
```

> raw 表的优先级最高（优先级 1），在所有其他表（包括 conntrack）之前执行。设置 `NOTRACK` 后，该数据包不会进入 conntrack 系统，也不会被状态匹配。

---

## 四、查看与管理

### 4.1 查看连接跟踪表

```bash
# 通过 proc 文件系统查看所有连接
cat /proc/net/nf_conntrack

# 使用 conntrack 命令行工具（需安装 conntrack 包）
conntrack -L              # 列出所有连接
conntrack -S              # 查看统计信息
conntrack -D              # 清空连接跟踪表
conntrack -E              # 实时事件监听（类似 tcpdump 风格）
```

### 4.2 系统参数调优

```bash
# 查看所有 conntrack 相关参数
sysctl -a | grep conntrack

# 常用调优参数
net.netfilter.nf_conntrack_max = 262144           # 最大跟踪连接数（默认通常较小）
net.netfilter.nf_conntrack_buckets = 65536         # 哈希表大小
net.netfilter.nf_conntrack_tcp_timeout_established = 432000  # TCP 已建立超时（秒）
net.netfilter.nf_conntrack_udp_timeout = 30        # UDP 超时（秒）

# 查看当前 conntrack 模块加载状态
lsmod | grep nf_conntrack
```

> [!warning] 高流量场景注意
> 默认 `nf_conntrack_max` 通常只有 65536 或 262144，在高并发场景下可能耗尽，导致新连接被丢弃。需要根据实际并发连接数调整。当 conntrack 表满时，内核日志（`dmesg`）会出现 `nf_conntrack: table full, dropping packet` 的提示。

---

## 五、与 NAT 的关系

**NAT 引擎（`nf_nat`）依赖 conntrack 工作**，这是理解 NAT 的关键：

1. DNAT 或 SNAT 修改数据包地址后，conntrack 记录**修改前和修改后**的元组对应关系
2. 反向包到达时，conntrack 根据记录将地址**转换回原始值**
3. 这也是为什么 NAT 的优先级（-100）低于 conntrack（-200）—— NAT 需要 conntrack **先**建好条目

```
原始包: Client:12345 → VIP:80
               │
               ▼  PRE_ROUTING: DNAT (依赖 conntrack)
               │
               ▼
已修改:  Client:12345 → RS:8080
          (conntrack 记录了 Client:12345 → RS:8080 的映射)

回包:    RS:8080 → Client:12345
               │
               ▼  POST_ROUTING: SNAT (依赖 conntrack)
               │
               ▼
已修改:  VIP:80 → Client:12345
          (conntrack 根据记录将源地址改回 VIP)
```

---

## 六、IPVS 的连接跟踪（IPVS CT）

IPVS 维护**独立于 nf_conntrack 的连接跟踪表**，用于记录已建立的连接与分配的 Real Server 之间的映射关系，保证同一连接的所有包转发到同一台 RS：

```bash
# 查看 IPVS 连接表
ipvsadm -L -n -c
```

IPVS 连接状态（不与 nf_conntrack 的四大状态混淆）：

| IPVS 状态 | 说明 |
|-----------|------|
| `NONE` | 初始态 |
| `ESTABLISHED` | 连接已建立 |
| `SYN_SENT` | 收到 SYN |
| `SYN_RECV` | 收到 SYN+ACK |
| `FIN_WAIT` | 收到 FIN |
| `TIME_WAIT` | 收到 FIN+ACK |
| `CLOSE` | 连接关闭 |

> [!note] 两套独立的系统
> IPVS 的连接跟踪与 nf_conntrack 是两套独立的系统，互不干扰。在 Kubernetes 中，当 kube-proxy 使用 IPVS 模式时，IPVS 连接跟踪负责保证同源连接的一致性，而 nf_conntrack 可能仍被其他 iptables 规则使用。

---

## 七、最佳实践与注意事项

1. **高流量场景需调大 `nf_conntrack_max`**：默认值可能不足，dmesg 中出现 `table full, dropping packet` 就需要扩容
2. **NOTRACK 提升性能**：对高吞吐的 UDP 服务（DNS、NTP）或大量短连接场景，在 raw 表中使用 NOTRACK 绕过 conntrack
3. **INVALID 包通常应丢弃**：可能是扫描攻击、畸形包或配置错误导致，`-m state --state INVALID -j DROP` 是推荐做法
4. **conntrack 是状态防火墙的基石**：`-m state --state ESTABLISHED,RELATED -j ACCEPT` 是最核心的放行规则
5. **监控 conntrack 使用率**：通过 `conntrack -S` 或 Prometheus node_exporter 监控 conntrack 使用率，提前预警

## 相关内核模块

| 模块名 | 功能 | 依赖 |
|--------|------|------|
| `nf_conntrack` | 连接跟踪核心 | 无 |
| `nf_conntrack_ipv4` | IPv4 连接跟踪 | nf_conntrack |
| `nf_conntrack_ipv6` | IPv6 连接跟踪 | nf_conntrack |
| `nf_conntrack_proto_tcp` | TCP 协议跟踪 | nf_conntrack |
| `nf_conntrack_proto_udp` | UDP 协议跟踪 | nf_conntrack |
| `nf_nat` | NAT 引擎 | nf_conntrack |
| `xt_state` | iptables state 匹配 | nf_conntrack |
| `xt_conntrack` | iptables conntrack 匹配 | nf_conntrack |

## 关联笔记

- [[netfilter框架详解]] — conntrack 所属的 netfilter 框架（Hook 点和优先级系统）
- [[iptables详解]] — iptables 中使用 -m state 实现状态防火墙
- [[IPVS-IP虚拟服务器详解]] — IPVS 独立的连接跟踪（IPVS CT）
- [[iptables端口转发]] — DNAT/SNAT 端口转发（依赖 conntrack）
- [[Linux-IP转发与路由]] — 三层转发与路由配置
