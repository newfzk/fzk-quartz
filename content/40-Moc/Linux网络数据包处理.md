---
title: Linux 网络数据包处理（MOC）
date: 2026-06-10
tags:
  - topic/Linux
  - topic/计算机网络
  - topic/MOC
status: to-review
aliases:
  - Linux 网络栈
  - Linux Kernel Networking
  - Netfilter MOC
---

# Linux 网络数据包处理 — 知识地图

> [!abstract] 本笔记为 MOC（Map of Content）
> Linux 内核从网卡收包到用户态进程接收，中间经历了一条完整的**协议栈路径**。Netfilter 框架在这条路径上插入了 Hook 点，iptables 和 IPVS 则是在这些 Hook 点上注册的不同处理逻辑。本 MOC 组织相关的原子笔记。

## 数据包处理全景

```
网卡收包 ─→ 驱动层 ─→ 内核协议栈
                         │
                    ┌────▼────┐
                    │PREROUTING│  ← netfilter Hook
                    └────┬────┘
                         │
                    ┌────▼────┐
                    │ 路由判定 │
                    └────┬────┘
                    ┌────┴────┐
               ┌────────┐  ┌────────────┐
               │本机    │  │非本机      │
               │LOCAL_IN│  │FORWARD     │
               │  ↕     │  │            │
               │ IPVS   │  │ Linux-IP   │
               │ 截获   │  │ 转发与路由 │
               │ 并分发 │  └─────┬──────┘
               └───┬────┘        │
                    │     ┌────▼───────┐
                    │     │POSTROUTING  │
                    │     └────┬───────┘
               ┌────▼──┐      │
               │转发给RS│    出网卡
               └──┬────┘
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      Real     Real     Real
      Server 1 Server 2 Server 3
```

> 图中展示了 IPVS 在 LOCAL_IN 点截获目标为 VIP 的流量，按调度算法分发给 Real Server。iptables 则在各 Hook 点配置过滤/NAT 规则。

## 核心组件速览

| 组件 | 层次 | 定位 | 核心笔记 |
|------|:----:|------|---------|
| **Netfilter** | 内核 | 包处理**框架**——定义 Hook 点，提供注册机制 | [[netfilter框架详解]] |
| **iptables** | 用户态 | 防火墙策略工具——包过滤、NAT、包修改 | [[iptables详解]] |
| **Conntrack** | 内核 | 连接跟踪子系统——跟踪连接状态，为 NAT 和状态防火墙提供基础 | [[Linux-连接跟踪-conntrack详解]] |
| **IPVS** | 内核 | 四层负载均衡器——按调度算法分发请求到 Real Server | [[IPVS-IP虚拟服务器详解]] |
| **Director & RS** | 概念 | IPVS 架构中的调度器与真实服务器的角色划分与关系 | [[IPVS-Director与Real-Server]] |
| **ipvsadm** | 用户态 | IPVS 管理命令——增删查改虚拟服务和 Real Server | [[ipvsadm命令详解]] |
| **七层负载均衡（用户态）** | 用户态 | 七层负载均衡器——解析 HTTP 协议，按内容路由/SSL 卸载分发 | [[四层与七层负载均衡对比]] |

## 数据流场景

### 场景一：iptables 防火墙过滤

```
包进入 → PREROUTING → 路由判定 → FORWARD/INPUT → 匹配 filter 表规则 → ACCEPT/DROP
                                                                   ↑
                                                           iptables 在此配置规则
```

### 场景二：iptables NAT（端口转发）

```
包进入 → PREROUTING(nat表DNAT) → 路由判定 → FORWARD → POSTROUTING(nat表SNAT/MASQUERADE)
                                                              ↑
                                                     [[iptables端口转发]]
```

### 场景三：IPVS 负载均衡（NAT 模式）

```
包进入 → PREROUTING → 路由判定(目标=VIP) → IPVS HOOK → 选 RS → 改目标IP
                                                          ↓
                                                  FORWARD → POSTROUTING → RS
```

### 场景四：IPVS 负载均衡（DR 模式）

```
包进入 → PREROUTING → 路由判定(目标=VIP) → IPVS HOOK → 选 RS → 改MAC地址
                                                          ↓
                    包直接二层转发 → RS（lo 接口上绑定的 VIP 接收）
```

## 关键对比

### iptables vs IPVS

| 对比维度   | iptables               | IPVS                     |
| ------ | ---------------------- | ------------------------ |
| 用途     | 防火墙 + NAT              | 四层负载均衡                   |
| 工作位置   | 全部 5 个 Hook 点          | LOCAL_IN（优先级高于 iptables） |
| 规则复杂度  | 五表五链，线性/树形匹配           | 两层结构：VS → RS             |
| 性能     | 中等（规则越多越慢）             | 高（哈希查找 O(1)）             |
| 调度算法   | 无，纯条件匹配                | 10 种调度算法                 |
| K8s 模式 | kube-proxy iptables 模式 | kube-proxy IPVS 模式       |

### IPVS 三种模式

| 特性 | NAT | DR | TUN |
|------|:---:|:--:|:---:|
| 修改内容 | IP:Port | MAC 地址 | 外层 IP 头封装 |
| 响应路径 | 经 Director | 直返客户端 | 直返客户端 |
| 性能 | 中等 | 最高 | 高 |
| 端口映射 | ✅ | ❌ | ❌ |
| 跨网段 | ❌ 需同子网 | ❌ 需同二层 | ✅ 跨机房 |
| RS 配置 | 改网关 | lo 绑定 VIP + ARP 抑制 | 支持 IPIP 隧道 |

详见 [[IPVS-IP虚拟服务器详解#二、三种工作模式]]。

### 四层 vs 七层 负载均衡

| 对比维度 | L4 四层（内核态） | L7 七层（用户态） |
|:---------|:----------------|:----------------|
| **OSI 层级** | 传输层（TCP/UDP） | 应用层（HTTP/HTTPS/gRPC） |
| **决策依据** | 五元组 IP:Port | URL、Header、Cookie |
| **典型实现** | IPVS、Nginx Stream、HAProxy TCP | Nginx HTTP、HAProxy HTTP、Envoy |
| **性能** | ⭐⭐⭐⭐⭐（内核态 O(1)） | ⭐⭐⭐（用户态解包封包） |
| **内容路由** | ❌ | ✅ 路径/域名/Header 路由 |
| **SSL 卸载** | ❌ | ✅ TLS termination |
| **协议范围** | TCP/UDP 全部 | HTTP/HTTPS/gRPC 等 |

> [!tip] 选型原则
> **能用 L4 就不用 L7**，L7 仅在需要应用层决策时引入。大型架构常见前端 L4（IPVS/NLB）做流量入口 + 后端 L7（Nginx/Envoy）做精细路由的**分层架构**。
>
> 详见 [[四层与七层负载均衡对比]]。

## 依赖关系

```
┌──────────────────────────────────┐
│          Netfilter 框架           │ ← 底层基础设施
│  定义 5 个 Hook 点 + 注册机制     │
└───────┬────────────────┬────────┘
        │                │
   ┌────▼────┐     ┌────▼────┐
   │ Conntrack│     │  IPVS   │
   │ 连接跟踪 │     │ 负载均衡 │ ← IPVS 有独立连接跟踪
   └────┬─────┘     └─────────┘
        │
   ┌────▼────┐
   │ iptables │     ← 依赖 conntrack 做状态匹配
   │ 防火墙   │
   │ NAT      │
   └────┬─────┘
        │
   ┌────▼────┐
   │iptables │     ← 用户空间管理工具
   │ 命令    │
   └─────────┘
```

> **一句话理解关系**：Netfilter 是"高速公路上的检查站"，conntrack 是"记录车辆信息的摄像头"，iptables 是"交警（拦车检查/放行）"，IPVS 是"调度中心（把车引导到不同出口）"。

## 基础协议层（网络层）

内核协议栈之上的**网络层基础协议**：地址结构决定路由判定依据，ARP 提供帧封装所需的目标 MAC，RIP/BGP 自动生成路由表。

| 主题 | 核心笔记 |
|:-----|:---------|
| **IPv4 地址分类与子网掩码** | [[IPv4地址分类与子网掩码]] |
| **IP 包接力与路由表** | [[IP包接力与路由表]] |
| **ARP：IP → MAC 解析** | [[ARP协议-IP到MAC的解析]] |
| **RIP 与 BGP：路由表自动生成** | [[RIP与BGP-路由表的自动生成]] |

> [!important] 与内核处理路径的衔接
> 上方流程图中的**「路由判定」**一步，依据就是 [[IP包接力与路由表]] 描述的路由表匹配规则（最长前缀匹配 + 默认路由）。
> 判定完成后要把 IP 包封装成帧发送时，需要目标 MAC 地址 —— 由 [[ARP协议-IP到MAC的解析]] 提供。
> 而路由表本身的来源，则可能是静态配置（[[Linux-IP转发与路由]]）或路由协议（[[RIP与BGP-路由表的自动生成]]）。

## 相关原子笔记

```dataview
TABLE
  file.tags as "标签"
FROM "10-Topics"
WHERE file.name IN ["netfilter框架详解", "iptables详解", "iptables端口转发", "iptables-扩展匹配模块", "IPVS-IP虚拟服务器详解", "IPVS-Director与Real-Server", "ipvsadm命令详解", "Linux-连接跟踪-conntrack详解", "Linux-IP转发与路由", "K8s-NodePort流量流转", "IPv4地址分类与子网掩码", "IP包接力与路由表", "ARP协议-IP到MAC的解析", "RIP与BGP-路由表的自动生成"]
SORT file.name ASC
```

## 外部关联

- [[Linux-IP转发与路由]] — IPVS NAT 模式和 iptables NAT 都需要开启 IP 转发
- [[Docker网络模式-bridge]] — Docker bridge 网络依赖 iptables NAT 实现容器端口映射
- [[案例-Docker-iptables模式切换导致链缺失]] — iptables nft/legacy 模式切换导致 Docker 链缺失的故障排查案例
- [[Linux-cgroup-控制组]] — 容器网络资源限制的底层机制
- [[四层与七层负载均衡对比]] — L4（IPVS）vs L7（Nginx/Envoy）负载均衡对比

## 面试要点

- **Netfilter → iptables / IPVS 的关系**：Netfilter 是框架（机制），iptables 和 IPVS 是具体实现（策略）。[[netfilter框架详解]]
- **Conntrack 是状态防火墙和 NAT 的基础**：它跟踪连接状态（NEW/ESTABLISHED/RELATED/INVALID），在优先级 -200 处注册，先于所有其他处理。[[Linux-连接跟踪-conntrack详解]]
- **iptables "四表五链" → 实际"五表五链"**：security 表是后来加入的。[[iptables详解#1.1 五张表（Tables）]]
- **IPVS 三种模式选型**：NAT 配置最简单但性能有瓶颈；DR 性能最高但需同二层；TUN 可跨机房但配置复杂。[[IPVS-IP虚拟服务器详解#二、三种工作模式]]
- **IPVS vs iptables 性能差异**：IPVS 使用哈希查找 O(1)，iptables 规则链随规则数增长性能下降。[[IPVS-IP虚拟服务器详解#五、IPVS-vs-iptables]]
- **keepalived + IPVS 高可用方案**：keepalived 负责 VIP 漂移和健康检查，自动管理 IPVS 规则。[[ipvsadm命令详解#6.2 健康检查配合（keepalived）]]
- **K8s 中 iptables 模式 vs IPVS 模式**：大规模 Service 场景下 IPVS 模式性能更优。[[IPVS-IP虚拟服务器详解#五、IPVS-vs-iptables]]
- **Director 与 RS 的职责划分**：Director 负责调度，RS 负责处理，不同模式下两者的交互路径不同。[[IPVS-Director与Real-Server]]
- **RS 的健康检查与权重**：通过 keepalived 监控 RS 状态，权重为 0 时 RS 被摘除。[[IPVS-Director与Real-Server#2.4 RS 的健康检查]]
