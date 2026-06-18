---
title: netfilter框架详解
date: 2026-06-09
updated: 2026-06-09
tags:
  - topic/Linux
  - topic/计算机网络
status: to-review
---

## 概述

**Netfilter** 是 Linux 内核中的一个**包处理框架**，它在内核网络协议栈的关键路径上插入一系列**钩子（Hooks）**，允许内核模块注册回调函数，对经过的数据包进行过滤、修改、跟踪或记录。

> iptables 只是 netfilter 的用户空间配置工具，真正干活的是内核中的 netfilter 框架。**netfilter 是机制（mechanism），iptables 是策略（policy）**。

```
用户空间:  ┌──────────┐  ┌──────────┐
           │ iptables │  │ nftables │
           └─────┬────┘  └────┬─────┘
                 │            │
内核空间:        ▼            ▼
           ┌──────────────────────────┐
           │    Netfilter 框架         │
           │  ┌──────────────────────┐ │
           │  │ Hook 点 + 回调链      │ │
           │  └──────────────────────┘ │
           │       ↕                   │
           │  ┌──────────────────────┐ │
           │  │ 协议栈（IP层处理）    │ │
           │  └──────────────────────┘ │
           └──────────────────────────┘
```

---

## 一、五大 Hook 点（嵌入协议栈的位置）

Netfilter 在 Linux 网络协议栈的 IP 层中插入了 **5 个钩子点**，对应数据包处理的不同阶段：

| Hook 点 | 宏定义 | 位置 | 方向 |
|---------|--------|------|------|
| **PREROUTING** | `NF_INET_PRE_ROUTING` | 数据包进入 IP 层后，**路由决策前** | 入站 |
| **LOCAL_IN** | `NF_INET_LOCAL_IN` | 路由决策后，**目的地是本机**所有经 INPUT 链的包 | 入站→本机 |
| **FORWARD** | `NF_INET_FORWARD` | 路由决策后，**目的地非本机**（需转发） | 经过 |
| **LOCAL_OUT** | `NF_INET_LOCAL_OUT` | **本机进程发出的包**，路由决策前 | 出站 |
| **POSTROUTING** | `NF_INET_POST_ROUTING` | 路由决策后，**即将发送到网卡前** | 出站 |

### 1.1 Hook 点在协议栈中的位置

```
                         ┌────────────────────┐
                         │  网卡驱动收包       │
                         │  (NAPI / 中断处理)  │
                         └─────────┬──────────┘
                                   │
                                   ▼
                         ┌────────────────────┐
                         │  ip_rcv()          │
                         │  (IP 层入口)       │
                         └─────────┬──────────┘
                                   │
                                   ▼
                    ┌────────────────────────────┐
                    │  NF_INET_PRE_ROUTING (Hook) │  ◄── raw/mangle/nat(DNAT)
                    └─────────────┬──────────────┘
                                  │
                                  ▼
                          ┌───────────────┐
                          │  路由决策      │
                          │  (ip_route_input)
                          └───────┬───────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
    ┌─────────────────────────┐     ┌──────────────────────────┐
    │  NF_INET_LOCAL_IN       │     │  NF_INET_FORWARD         │
    │  (目的地是本机)         │     │  (非本机，需转发)        │
    │  ◄── mangle/filter/     │     │  ◄── mangle/filter/      │
    │       security          │     │       security           │
    └────────────┬────────────┘     └─────────────┬────────────┘
                 │                                │
                 ▼                                ▼
    ┌──────────────────────┐     ┌────────────────────────────────┐
    │  ip_local_deliver()  │     │  ip_forward()                  │
    │  → 传输层(TCP/UDP)   │     │  → NF_INET_POST_ROUTING       │
    └──────────────────────┘     └────────────────────────────────┘
                 │                                │
    ┌────────────┘                                │
    ▼                                             │
    ┌─────────────────────────┐                   │
    │  本机进程产生数据包      │                   │
    │  (传输层 → IP层)        │                   │
    └────────────┬────────────┘                   │
                 │                                │
                 ▼                                │
    ┌────────────────────────────┐                │
    │  NF_INET_LOCAL_OUT         │                │
    │  ◄── raw/mangle/nat(DNAT)/ │                │
    │       filter/security      │                │
    └─────────────┬──────────────┘                │
                  │                               │
                  ▼                               │
          ┌───────────────┐                       │
          │  路由决策      │                       │
          └───────┬───────┘                       │
                  │                               │
                  ▼                               ▼
    ┌──────────────────────────────────────────────────┐
    │  NF_INET_POST_ROUTING                             │
    │  ◄── mangle/nat(SNAT)                             │
    └───────────────────────┬──────────────────────────┘
                            │
                            ▼
                  ┌──────────────────┐
                  │  ip_output()     │
                  │  → 网卡驱动发送  │
                  └──────────────────┘
```

### 1.2 三种数据包路径

```c
// 情况 1：发给本机的包
网卡 → ip_rcv() → PRE_ROUTING → 路由决策(目标=本机) → LOCAL_IN → 本机进程

// 情况 2：经过本机转发的包
网卡 → ip_rcv() → PRE_ROUTING → 路由决策(目标≠本机) → FORWARD → POST_ROUTING → 出网卡

// 情况 3：本机发出的包
本机进程 → LOCAL_OUT → 路由决策 → POST_ROUTING → ip_output() → 出网卡
```

---

## 二、Hook 注册机制

### 2.1 核心数据结构

Kernel 模块通过 `nf_hook_ops` 结构体注册钩子回调：

```c
struct nf_hook_ops {
    struct list_head    list;       // 链表节点，将多个 ops 串起来
    nf_hookfn           *hook;     // 回调函数指针（核心！）
    struct net_device   *dev;      // 绑定到特定网卡（NULL=所有网卡）
    void                *priv;     // 私有数据
    u_int8_t            pf;        // 协议族（AF_INET, AF_INET6, AF_BRIDGE...）
    u_int8_t            hooknum;   // 挂载到哪个 Hook 点
    int                 priority;  // 优先级（数字越小优先级越高）
};
```

### 2.2 回调函数签名

```c
// 返回值决定了数据包的命运
typedef unsigned int nf_hookfn(
    void              *priv,       // 注册时的私有数据
    struct sk_buff    *skb,        // 数据包（socket buffer）
    const struct nf_hook_state *state  // 钩子状态（含 net, dev, pf 等）
);
```

### 2.3 回调返回值（裁决结果）

| 返回值 | 含义 | 行为 |
|--------|------|------|
| `NF_ACCEPT` (1) | 放行 | 继续正常处理，走下一个钩子或协议栈 |
| `NF_DROP` (0) | 丢弃 | 立即释放 skb，不再处理 |
| `NF_QUEUE` (3) | 排队 | 将包发到用户空间处理（如 IPSec、防病毒） |
| `NF_STOLEN` (4) | 接管 | 钩子接管了包，协议栈不再处理（如用来实现透明代理） |
| `NF_REPEAT` (2) | 重做 | 让本钩子再处理一次（用于动态规则） |

```
                     数据包到达 Hook 点
                           │
                           ▼
                 ┌───────────────────┐
                 │ 遍历钩子回调链    │
                 │ 按优先级排序      │
                 └───────┬───────────┘
                           │
                    ┌──────┴──────┐
                    │             │
                    ▼             ▼
              ┌──────────┐  ┌──────────┐
              │ 回调函数  │  │ 回调函数  │  ...
              │ #1 (最高  │  │ #2       │
              │  优先级)  │  │          │
              └─────┬────┘  └─────┬────┘
                    │             │
         ┌──────────┴──┐         │
         │ 返回值       │         │
         └──────┬──────┘         │
                │                │
       ┌────────┼────────┐       │
       │        │        │       │
       ▼        ▼        ▼       │
   NF_DROP  NF_ACCEPT  NF_QUEUE  │
   (丢弃)   (继续)     (排队)    │
                                  │
                            NF_ACCEPT
                                  │
                                  ▼
                         协议栈继续处理
```

### 2.4 优先级系统

多个模块可以在同一个 Hook 点注册，通过优先级决定执行顺序：

```c
// 枚举优先级（高到低，数字越小优先级越高）
enum nf_ip_hook_priorities {
    NF_IP_PRI_FIRST         = INT_MIN,  // 最高优先级
    NF_IP_PRI_RAW_BEFORE    = -450,     // raw 表
    NF_IP_PRI_CONNTRACK     = -200,     // 连接跟踪
    NF_IP_PRI_MANGLE        = -150,     // mangle 表
    NF_IP_PRI_NAT_DST       = -100,     // NAT (DNAT)
    NF_IP_PRI_FILTER        = 0,        // filter 表
    NF_IP_PRI_SECURITY      = 50,       // security 表
    NF_IP_PRI_NAT_SRC       = 100,      // NAT (SNAT)
    NF_IP_PRI_RAW_AFTER     = 300,      // raw 表后期
    NF_IP_PRI_LAST          = INT_MAX,  // 最低优先级
};
```

> **关键理解**：不同的内核模块（如 conntrack、iptables、nftables）各自注册自己的 `nf_hook_ops`，它们通过优先级确定执行顺序。例如 `conntrack` 的优先级高于 `filter` 表，所以连接跟踪先于包过滤执行。

---

## 三、核心子模块架构

Netfilter 不是一个单一的模块，而是一组协同工作的内核模块：

```
                      ┌─────────────────────────────────┐
                      │         Netfilter 框架           │
                      │  (nf_hook_slow, 钩子注册API)     │
                      └─────────────────────────────────┘
                                    │
           ┌───────────┬────────────┼────────────┬──────────────┐
           │           │            │            │              │
           ▼           ▼            ▼            ▼              ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐
    │ 连接跟踪  │ │ 包过滤    │ │ NAT 引擎 │ │ 包日志   │ │ 其他扩展    │
    │ conntrack │ │ iptables │ │ nf_nat   │ │ nf_log   │ │ nf_queue    │
    │           │ │ nftables │ │          │ │          │ │ nfnetlink   │
    └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────────┘
```

### 3.1 连接跟踪（Connection Tracking / nf_conntrack）

**最核心的子模块**，许多其他功能（NAT、状态匹配、连接数限制）依赖于它。conntrack 在 `NF_INET_PRE_ROUTING` 和 `NF_INET_LOCAL_OUT` 两个 Hook 点以最高优先级（-200）注册，在所有包过滤和 NAT 之前执行。

> 完整原理、数据结构、调优方法及使用方式请见 [[Linux-连接跟踪-conntrack详解]]。

### 3.2 NAT 引擎（nf_nat）

- **功能**：修改数据包的源/目标地址和端口
- **核心机制**：在 conntrack 的基础上工作，通过修改 nf_conn 条目实现反向转换
- **Hook 注册点**：
  - DNAT：`NF_INET_PRE_ROUTING`（优先级 -100）和 `NF_INET_LOCAL_OUT`
  - SNAT：`NF_INET_POST_ROUTING`（优先级 100）
- **两种模式**：
  - `SNAT`：修改源地址，用于内网→外网
  - `DNAT`：修改目标地址，用于端口转发

### 3.3 包过滤引擎（iptables / nftables）

- **功能**：匹配数据包特征并执行动作（ACCEPT/DROP/REJECT 等）
- **内核模块**：
  - 传统：`ip_tables.ko` — x_tables 框架的 IPv4 部分
  - 新一代：`nf_tables.ko` — nftables 内核引擎
- **Hook 注册点**：`LOCAL_IN`, `FORWARD`, `LOCAL_OUT`（优先级 0）

### 3.4 包日志（nf_log）

- **功能**：记录被丢弃/匹配的数据包信息
- **用户空间**：通过 `dmesg` 或 `ulogd` 查看日志

---

## 四、数据包在内核中的数据结构

### 4.1 sk_buff — 网络数据的统一表示

`struct sk_buff`（简称 skb）是 Linux 网络子系统中最核心的数据结构，netfilter 的所有操作都是围绕着它进行的：

```c
struct sk_buff {
    /* 网络层信息 */
    union {
        struct iphdr  *iph;    // IPv4 头部
        struct ipv6hdr *ipv6h; // IPv6 头部
    };
    
    /* 传输层信息 */
    union {
        struct tcphdr *th;     // TCP 头部
        struct udphdr *uh;     // UDP 头部
        struct icmphdr *icmph; // ICMP 头部
    };
    
    /* 网络命名空间 */
    struct net       *dev->nd_net;
    
    /* 数据指针 */
    unsigned char    *data;    // 数据起始位置
    unsigned char    *head;    // 分配空间起始
    unsigned char    *end;     // 分配空间结束
    unsigned int     len;      // 实际数据长度
    unsigned int     truesize; // 实际占用内存
    
    /* 标记（hook 返回值等） */
    u32              mark;     // nfmark（包标记）
    
    /* ... 数百个字段，极其复杂的结构体 ... */
};
```

### 4.2 nf_conn — 连接跟踪条目

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

### 4.3 nf_conntrack_tuple — 连接标识（五元组）

```c
struct nf_conntrack_tuple {
    struct nf_conntrack_man_proto src;     // 源端口
    struct nf_conntrack_man_proto dst;     // 目标端口
    union nf_inet_addr           src_addr; // 源 IP
    union nf_inet_addr           dst_addr; // 目标 IP
    u_int8_t                     u_int8_t protocol;  // L4 协议
};
```

---

## 五、数据包处理流程的完整路径（含 netfilter 视角）

以**本机转发**数据包为例，netfilter 在协议栈中的完整旅程：

```
[网卡收包]
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 1. 网卡驱动 → GRO → 进入 IP 层                       │
│    __netif_receive_skb_core() → ip_rcv()             │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 2. NF_INET_PRE_ROUTING                              │
│    ● raw (NOTRACK)                                  │
│    ● conntrack (连接跟踪查找/创建)  ← 优先级最高     │
│    ● mangle (修改 TTL/TOS/Mark)                     │
│    ● nat DNAT (修改目标地址)                         │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 3. 路由决策（ip_route_input_slow/fast）              │
│    查找 FIB 路由表 → 确定出接口和下一跳               │
│    如果目标非本机，设置 skb->pkt_type = PACKET_OTHERHOST │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 4. NF_INET_FORWARD                                  │
│    ● mangle                                         │
│    ● filter (ACCEPT/DROP/REJECT)  ← 防火墙核心       │
│    ● security                                       │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 5. ip_forward_finish() → ip_output()                │
│    分片、路由缓存、邻居子系统查找                    │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 6. NF_INET_POST_ROUTING                             │
│    ● mangle                                         │
│    ● nat SNAT/MASQUERADE (修改源地址) ← 优先级最低   │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 7. 邻居子系统 → 网卡驱动 → 发送                     │
│    dev_queue_xmit() → ndo_start_xmit()              │
└─────────────────────────────────────────────────────┘
```

---

## 六、Netfilter 的发展历史

```
1998年前  — 内核 2.0/2.2：ipfwadm、ipchains（不成熟的早期方案）
          │
          ▼
1998-2001 — 内核 2.3.x → 2.4
          Rusty Russell 设计了 netfilter 框架
          首次引入"钩子 + 回调"架构
          iptables 作为用户空间管理工具
          │
          ▼
2003-2012 — 内核 2.6.x → 3.x
          引入连接跟踪（conntrack）
          引入 nf_nat 替代旧的 NAT 代码
          引入 nfnetlink 通信机制
          xtables 模块化扩展系统成熟
          │
          ▼
2014-至今 — 内核 3.13+
          引入 nf_tables（nftables 内核引擎）
          保留 x_tables 兼容
          逐步推动 nf_tables 成为默认
          │
          ▼
如今     — iptables 命令在 RHEL9/Debian12 上默认调用 nftables 内核
          nf_tables 成为主流推荐
```

---

## 七、Netfilter 与 iptables / nftables 的关系

```
┌──────────────────────────────────────────────────────┐
│ 用户空间                                              │
│  ┌──────────┐    ┌──────────┐                        │
│  │ iptables │    │ nft      │  ← 命令行工具            │
│  │ (旧)     │    │ (新)     │                          │
│  └────┬─────┘    └────┬─────┘                         │
│       │               │                               │
│  ┌────▼───────────────▼────┐                          │
│  │  Netlink 通信           │                          │
│  │  (nfnetlink / nlsock)   │                          │
│  └────┬────────────────────┘                          │
├───────┼──────────────────────────────────────────────┤
│ 内核空间 │                                            │
│  ┌────▼────────────────────┐                          │
│  │  x_tables (iptables内核) │  ───── 旧架构            │
│  │  /                       │                          │
│  │  nf_tables (nftables内核)│  ───── 新架构            │
│  │  均注册 nf_hook_ops       │                          │
│  └────┬────────────────────┘                          │
│       │                                               │
│  ┌────▼────────────────────┐                          │
│  │  Netfilter 核心          │                          │
│  │  ─ nf_hook_slow()       │                          │
│  │  ─ nf_hook_ops 链表      │                          │
│  │  ─ nf_conn 连接跟踪       │                          │
│  └────┬────────────────────┘                          │
│       │                                               │
│  ┌────▼────────────────────┐                          │
│  │ 内核网络协议栈            │                          │
│  │  (ip_rcv, ip_output...)  │                          │
│  └─────────────────────────┘                          │
└──────────────────────────────────────────────────────┘
```

### 对比总结

| 维度 | x_tables (iptables) | nf_tables (nftables) |
|------|--------------------|---------------------|
| 内核模块 | `ip_tables.ko` | `nf_tables.ko` |
| 规则存储 | 线性链表（遍历慢） | Set/Map 数据结构（性能更优） |
| 原子替换 | 不支持（需清空→重载） | 支持（无窗口期） |
| 语法 | 复杂的表/链/规则 | 更简洁、可读性更好 |
| 扩展性 | 每个匹配/目标需加载内核模块 | 内建丰富的表达式系统 |
| 内核版本 | 2.4+ | 3.13+ |
| 现状 | 兼容模式，部分发行版仍在用 | 推荐方案 |

---

## 八、关键内核模块速查

| 模块名 | 功能 | 依赖关系 |
|--------|------|---------|
| `nf_conntrack` | 连接跟踪核心 | 被 NAT、state 匹配依赖 |
| `nf_conntrack_ipv4` | IPv4 连接跟踪 | 依赖 nf_conntrack |
| `nf_nat` | NAT 核心 | 依赖 nf_conntrack |
| `nf_nat_ipv4` | IPv4 NAT | 依赖 nf_nat |
| `ip_tables` | iptables IPv4 内核 | 依赖 x_tables |
| `iptable_filter` | filter 表 | 依赖 ip_tables |
| `iptable_nat` | nat 表 | 依赖 ip_tables + nf_nat |
| `iptable_mangle` | mangle 表 | 依赖 ip_tables |
| `iptable_raw` | raw 表 | 依赖 ip_tables |
| `nf_tables` | nftables 内核引擎 | 无 |

```bash
# 查看当前加载的 netfilter 相关模块
lsmod | grep -E "nf_|x_tables|ip_tables|iptable"

# 查看连接跟踪表
cat /proc/net/nf_conntrack

# 查看连接跟踪参数
sysctl -a | grep conntrack
```

---

## 九、Netfilter 的扩展能力

Netfilter 的扩展性体现在它允许用户空间程序和内核模块灵活介入包处理：

### 9.1 nfqueue — 将包发到用户空间

```bash
# 将 TCP 80 端口的包发送到用户空间的队列 #0
iptables -A INPUT -p tcp --dport 80 -j NFQUEUE --queue-num 0

# 用户空间的 Python 程序读取并裁决
```
```python
# 示例：使用 scapy 的 nfqueue 绑定
from netfilterqueue import NetfilterQueue

def process_packet(packet):
    data = packet.get_payload()
    # ... 解析、修改、决定 ...
    packet.accept()  # 或 packet.drop()

nfqueue = NetfilterQueue()
nfqueue.bind(0, process_packet)
nfqueue.run()
```

### 9.2 扩展匹配模块（xtables）

Netfilter 支持动态加载匹配模块（`-m` 参数对应内核模块）：

```bash
# 对应的内核模块
-m state        → xt_state.ko (依赖 nf_conntrack)
-m conntrack    → xt_conntrack.ko
-m limit        → xt_limit.ko
-m recent       → xt_recent.ko
-m connlimit    → xt_connlimit.ko
-m statistic    → xt_statistic.ko
-m time         → xt_time.ko
-m multiport    → xt_multiport.ko
-m length       → xt_length.ko
-m mac          → xt_mac.ko
```

---

## 十、面试要点速记

1. **Netfilter = 内核机制，iptables = 用户策略** — 理解这个分层是理解整个 Linux 网络的基础
2. **五大 Hook 点**必须记住：`PRE_ROUTING` → `LOCAL_IN` → `FORWARD` → `LOCAL_OUT` → `POST_ROUTING`
3. **优先级系统驱动执行顺序** — conntrack (-200) > mangle (-150) > DNAT (-100) > filter (0) > security (50) > SNAT (100)
4. **回调返回值决定包命运** — `NF_ACCEPT`, `NF_DROP`, `NF_QUEUE`, `NF_STOLEN`
5. **conntrack 是核心依赖** — NAT、state 匹配、连接限制都依赖于连接跟踪，详见 [[Linux-连接跟踪-conntrack详解]]
6. **nf_tables 是未来** — 性能更好、原子替换、语法简洁，新系统应使用 nftables
7. **Netfilter 的灵活性** — 不仅是防火墙框架，还能实现透明代理、负载均衡、流量统计、入侵检测等

## 参考链接

- [[iptables详解]] — iptables 命令使用和四表五链详解
- [[iptables端口转发]] — DNAT/SNAT 端口转发配置
- [[Linux-连接跟踪-conntrack详解]] — 连接跟踪完整原理与调优
- [[Linux-IP转发与路由]] — 三层转发与路由配置
- [[Docker网络模式-bridge]] — Docker 如何利用 netfilter/iptables 实现网络隔离
