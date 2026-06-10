---
title: IPVS — IP Virtual Server 详解
date: 2026-06-10
tags:
  - topic/计算机网络
  - topic/Linux
  - topic/负载均衡
status: to-review
updated: 2026-06-10
---

## 概述

**IPVS**（IP Virtual Server）是 Linux 内核内置的**四层（传输层）负载均衡器**，属于 **LVS**（Linux Virtual Server）项目的核心组件。它在内核空间直接实现负载均衡调度，将到达 VIP（虚拟 IP）的请求按调度算法分发到一组真实服务器（Real Server）上。

> 一句话：**IPVS 是内核态的 L4 负载均衡器**，类似于 **F5 硬件负载均衡器的软件实现**，也常作为 Kubernetes 中 kube-proxy 的 backend（iptables/IPVS 两种模式之一）。

```
用户请求（VIP:Port）
        │
        ▼
┌──────────────────┐
│   IPVS 调度器     │  ← 内核态，基于 netfilter hook
│  (Director/NLB)   │
└────────┬─────────┘
         │ 分发请求（根据调度算法）
    ┌────┼────┐
    ▼    ▼    ▼
┌─────┐┌─────┐┌─────┐
│ RS1 ││ RS2 ││ RS3 │  ← Real Server（真实后端服务）
└─────┘└─────┘└─────┘
```

---

## 一、IPVS 实现原理

### 1.1 基于 Netfilter Hook

IPVS 注册到 netfilter 框架的 **LOCAL_IN** 钩子点（`NF_INET_LOCAL_IN`）上。当一个数据包的目标 IP = VIP（即本机地址之一）时，在路由判定"目的地是本机"之后、进入 LOCAL_IN 链之前，IPVS 截获这个包：

```
数据包到达 → PREROUTING → 路由判定（目标=VIP）
           → IPVS HOOK（LOCAL_IN 之前）
           → 根据调度算法选 RS
           → 修改目标（NAT 模式：改目标 IP:Port）
           → 发送到 FORWARD/POSTROUTING
           → 到达 Real Server
```

> [!tip] IPVS 与 iptables 的协作关系
> IPVS 的 hook 点优先级**高于** iptables 的 INPUT 链规则。只要 IPVS 接管了流量，iptables INPUT 链就不会处理该包。但如果 IPVS 配置为空（无 Real Server），包会回退到 iptables 处理。

### 1.2 IPVS 连接跟踪（IPVS CT）

IPVS 维护自己的**连接跟踪表**（非 nf_conntrack），用于记录已建立的连接与分配的 Real Server 之间的映射关系，保证**同一连接的所有包转发到同一台 RS**。

```bash
# 查看 IPVS 连接表
ipvsadm -L -n -c
```

| 参数 | 说明 |
|------|------|
| `-L` | 列出虚拟服务/连接表 |
| `-n` | 数字格式，不反解 DNS |
| `-c` | 显示连接跟踪表 |

连接表条目示例：

```
pro expire state       source             virtual            destination
TCP 00:59  ESTABLISHED 10.0.0.1:54321     10.0.0.100:80      10.0.0.10:80
TCP 01:55  ESTABLISHED 10.0.0.2:12345     10.0.0.100:80      10.0.0.11:80
```

> IPVS 的连接跟踪独立于内核的 nf_conntrack，两者是两套不同的系统。详见 [[Linux-连接跟踪-conntrack详解#六、IPVS-的连接跟踪（IPVS-CT）]]。

---

## 二、三种工作模式

### 2.1 NAT 模式（最常用）

| 特性 | 说明 |
|------|------|
| **流程** | Director 做 DNAT + SNAT，修改请求和目标 IP/Port |
| **RS 网关** | 必须指向 Director 的 IP |
| **性能** | 中等（进出都经过 Director） |
| **网络要求** | Director 和 RS 在同一子网 |
| **端口映射** | 支持（VIP:Port1 → RS:Port2） |

```
请求方向:  Client → Director(VIP) → RS
响应方向:  RS → Director(VIP) → Client
```

### 2.2 DR 模式（Direct Routing，直接路由）

| 特性 | 说明 |
|------|------|
| **流程** | Director 只改写目标 MAC 地址，**不修改 IP 包首部** |
| **RS 网关** | 指向外部网关，**不需要**指向 Director |
| **性能** | 最高（响应直接回客户端，不经过 Director） |
| **网络要求** | Director 和 RS 在**同一二层网络**（LAN） |
| **端口映射** | 不支持 |
| **RS 配置** | RS 的 lo 接口需绑定 VIP，且抑制 ARP 响应 |

```
请求方向:  Client → Director(VIP) → RS（MAC 地址被改写）
响应方向:  RS → Client（直接返回，不经过 Director）
```

> [!note] DR 模式的 ARP 抑制
> RS 必须在 lo 上配置 VIP 且设置 `arp_ignore=1` 和 `arp_announce=2`，防止 RS 直接响应 ARP 请求导致客户端绕过 Director。

### 2.3 TUN 模式（IP Tunneling，IP 隧道）

| 特性 | 说明 |
|------|------|
| **流程** | Director 在原始 IP 包外**再封装一层 IP 头**，通过隧道发给 RS |
| **RS 网关** | 指向外部网关（同 DR） |
| **性能** | 高（响应直接回客户端） |
| **网络要求** | 支持**跨子网、跨机房**部署 |
| **端口映射** | 不支持 |
| **RS 配置** | RS 需要支持 IPIP 隧道协议 |

```
请求:  Client → Director(外层IP + 内层原IP) → RS
响应:  RS → Client（直接返回）
```

### 2.4 模式对比总结

| 特性 | NAT | DR | TUN |
|------|:---:|:--:|:---:|
| 响应路径 | 经 Director | 直返客户端 | 直返客户端 |
| 性能 | 中等 | 最高 | 高 |
| 端口映射 | ✅ 支持 | ❌ 不支持 | ❌ 不支持 |
| 二层要求 | 同一子网 | 同一 LAN | 无限制 |
| RS 修改 | 需改网关 | 需绑定 VIP + ARP 抑制 | 需支持 IPIP 隧道 |
| 带宽瓶颈 | 出入口 | 入口 | 入口 |

---

## 三、调度算法

IPVS 支持 **10 种调度算法**，通过 `ipvsadm -E -s <算法>` 指定：

| 算法 | 缩写 | 全称 | 说明 |
|------|:----:|------|------|
| **轮询** | **rr** | Round Robin | 依次分发，适合 RS 配置相同 |
| **加权轮询** | **wrr** | Weighted Round Robin | 按权重比例分配，权重越高分配越多 |
| **最少连接** | **lc** | Least Connections | 分配给当前活跃连接最少的 RS |
| **加权最少连接** | **wlc** | Weighted Least Connections | lc + 权重因子（**默认算法**） |
| **基于局部的最少连接** | **lblc** | Locality-Based Least Connections | 同一目标 IP 尽量发往同一 RS，适用于缓存集群 |
| **带复制的基于局部的最少连接** | **lblcr** | LBLCR | lblc 的改进，支持 RS 间复制缓存 |
| **目标哈希** | **dh** | Destination Hashing | 按目标 IP 哈希分配，保证同一客户端 IP 到同一 RS |
| **源哈希** | **sh** | Source Hashing | 按源 IP 哈希分配 |
| **最短期望延迟** | **sed** | Shortest Expected Delay | 期望延迟最短的 RS（(活动连接数+1)/权重）|
| **永不排队** | **nq** | Never Queue | sed 的改进，有空闲 RS 则直接分配 |

> [!tip] 默认算法 wlc
> 不指定 `-s` 时 IPVS **默认使用 wlc**（加权最少连接），这是大多数场景下表现最均衡的通用算法。

---

## 四、IPVS 连接处理状态机

IPVS 为每个 TCP 连接维护一个状态，与 iptables 的 conntrack 状态不同：

| IPVS 状态 | TCP 状态对应 | 说明 |
|-----------|-------------|------|
| `NONE` | - | 初始态 |
| `ESTABLISHED` | ESTABLISHED | 连接已建立 |
| `SYN_SENT` | SYN_SENT | 收到 SYN |
| `SYN_RECV` | SYN_RECV | 收到 SYN+ACK |
| `FIN_WAIT` | FIN_WAIT | 收到 FIN |
| `TIME_WAIT` | TIME_WAIT | 收到 FIN+ACK |
| `CLOSE` | CLOSE | 连接关闭 |

```bash
# 查看连接状态统计
ipvsadm -L -n --rate

# 查看超时配置
ipvsadm -L --timeout
```

---

## 五、IPVS vs iptables

两者都基于 netfilter，但定位和功能完全不同：

| 对比维度 | IPVS | iptables |
|---------|------|----------|
| **定位** | 四层负载均衡 | 防火墙 + NAT |
| **Hook 点** | LOCAL_IN（PREROUTING 之后） | 五表五链 |
| **性能** | 高（内核态哈希查找） | 中等（线性/树形规则匹配） |
| **调度策略** | 10 种调度算法 | 无，纯规则匹配 |
| **连接跟踪** | 独立 IPVS CT | nf_conntrack |
| **规则复杂度** | 简单（VS + RS 两层） | 复杂（五表五链嵌套） |
| **DPDK 友好** | 内核态，有局限 | 同左 |
| **K8s 支持** | kube-proxy IPVS 模式 | kube-proxy iptables 模式 |

> [!info] K8s 中的 IPVS
> Kubernetes 1.8+ 支持 kube-proxy 以 IPVS 模式运行。相比 iptables 模式，IPVS 模式在大规模 Service 场景下有 **O(1) 查找时间复杂度** 和更好的性能表现，且支持更丰富的负载均衡算法。

---

## 六、启用 IPVS 模块

```bash
# 查看当前 Linux 内核是否已加载 IPVS 模块
lsmod | grep ip_vs

# 手动加载 IPVS 相关模块
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_wlc
modprobe ip_vs_lc
modprobe ip_vs_sh
modprobe ip_vs_dh

# 持久化配置（/etc/modules-load.d/ipvs.conf）
cat > /etc/modules-load.d/ipvs.conf <<EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_wlc
ip_vs_lc
ip_vs_sh
ip_vs_dh
EOF
```

---

## 七、快速配置示例

以下示例在 Director 上配置一个 VIP `192.168.1.100:80`，分发到两台 Real Server：

```bash
# 1. 在 Director 上配置 VIP
ip addr add 192.168.1.100/24 dev eth0

# 2. 添加虚拟服务
ipvsadm -A -t 192.168.1.100:80 -s wlc

# 3. 添加 Real Server（NAT 模式需要 -m，DR 模式需要 -g）
ipvsadm -a -t 192.168.1.100:80 -r 10.0.0.10:80 -m -w 1
ipvsadm -a -t 192.168.1.100:80 -r 10.0.0.11:80 -m -w 2

# 4. 查看配置
ipvsadm -L -n

# 5. 保存配置
ipvsadm-save > /etc/sysconfig/ipvsadm
```

---

## 关联笔记

- [[netfilter框架详解|Netfilter 框架详解]] — IPVS 底层依赖的 netfilter Hook 机制
- [[iptables详解|iptables 详解]] — 同层的另一个 netfilter 用户态工具
- [[Linux-连接跟踪-conntrack详解|Linux 连接跟踪 — conntrack 详解]] — IPVS 有独立的连接跟踪（IPVS CT）
- [[Linux-IP转发与路由|Linux IP 转发与路由]] — IPVS 依赖 IP 转发
- [[ipvsadm命令详解|ipvsadm 命令详解]] — IPVS 的管理命令
- [[iptables端口转发|iptables 端口转发]] — iptables 实现端口转发与 IPVS 的对比
- [[Docker网络模式-bridge|Docker 网络模式 — bridge]] — K8s/Docker 环境中 IPVS 的典型应用场景

## 参考资料

- [LVS 官方文档](http://www.linuxvirtualserver.org/)
- [IPVS 内核文档](https://www.kernel.org/doc/Documentation/networking/ipvs-sysctl.txt)
- [Kubernetes IPVS 模式说明](https://kubernetes.io/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/)
