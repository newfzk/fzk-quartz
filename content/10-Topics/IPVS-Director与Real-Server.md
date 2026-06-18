---
title: IPVS — Director 与 Real Server 概念详解
date: 2026-06-10
tags:
  - topic/计算机网络
  - topic/负载均衡
status: to-review
---

## 概述

在 **IPVS/LVS 架构**中，有两个核心角色：

| 角色 | 英文 | 别称 | 职责 |
|------|------|------|------|
| **调度器** | **Director** | Load Balancer、NLB、LVS Director | 接收客户端请求，按调度算法分发给 RS |
| **真实服务器** | **Real Server (RS)** | Backend Server、Upstream Server | 实际处理请求并返回响应的后端服务 |

```
                  Client
                    │
                    ▼ 请求到达 VIP
            ┌───────┴───────┐
            │   Director     │  ← 运行 IPVS，负责调度
            │  (IPVS/NLB)    │
            └───────┬───────┘
                    │ 分发请求（按调度算法）
               ┌────┼────┐
               ▼    ▼    ▼
            ┌────┐┌────┐┌────┐
            │ RS ││ RS ││ RS │  ← 真实处理请求
            └────┘└────┘└────┘
```

> Client 只知 VIP，不知 RS 存在。Director 屏蔽了后端拓扑。

---

## 一、Director（调度器）

### 1.1 定义

**Director** 是运行 **IPVS 内核模块**的服务器，是整个负载均衡系统的**入口节点**。它绑定一个**虚拟 IP（VIP）**供客户端访问，收到请求后根据配置的调度算法将请求分发到某台 Real Server。

### 1.2 核心职责

| 职责 | 说明 |
|------|------|
| **接收流量** | 在 VIP 上监听来自客户端的请求 |
| **调度分发** | 根据调度算法（rr、wlc、sh 等）选择一台 RS |
| **连接跟踪** | 维护 IPVS CT 表，保证同一连接的所有包发往同一 RS |
| **健康检查配合** | 配合 keepalived 等工具，自动摘除故障 RS |
| **高可用（可选）** | 主备 Director 间通过 keepalived/VRRP 实现 VIP 漂移 |

### 1.3 Director 的视角

```
┌──────────────────────────────────────────────┐
│                 Director                       │
│                                                │
│  eth0: VIP = 192.168.1.100/24                  │
│  lo:   Management IP = 192.168.1.1/24          │
│                                                │
│  ┌─────────┐   ┌──────────┐   ┌─────────────┐ │
│  │ IPVS    │   │ 连接跟踪  │   │ 调度算法     │ │
│  │ 规则表   │   │ (IPVS CT) │   │ (wlc/rr/...) │ │
│  └─────────┘   └──────────┘   └─────────────┘ │
│         │            │                │         │
│         ▼            ▼                ▼         │
│   ┌─────────────────────────────┐              │
│   │   VS 192.168.1.100:80       │              │
│   │   ├── RS 10.0.0.10:80  w=1  │              │
│   │   ├── RS 10.0.0.11:80  w=2  │              │
│   │   └── RS 10.0.0.12:80  w=1  │              │
│   └─────────────────────────────┘              │
└────────────────────────────────────────────────┘
```

### 1.4 不同模式下的 Director 行为

| 模式 | Director 对包的处理 |
|:----:|-------------------|
| **NAT** | **修改目标 IP:Port**（DNAT），响应回来再改源 IP（SNAT） |
| **DR** | **只改目标 MAC 地址**，IP 头不变 |
| **TUN** | 在原始 IP 包外**封装一层 IP 头**，通过 IPIP 隧道发送 |

> Director 在 NAT 模式下是**流量瓶颈**（进出都经过它），在 DR/TUN 模式下仅入口经过它（响应直返客户端）。

---

## 二、Real Server（RS，真实服务器）

### 2.1 定义

**Real Server** 是实际运行后端应用程序（Web 服务、API、数据库等）的服务器。Client 不会直接访问 RS，RS 对客户端是**透明**的。

### 2.2 核心职责

| 职责 | 说明 |
|------|------|
| **处理请求** | 接收 Director 转发来的请求，执行实际业务逻辑 |
| **返回响应** | 根据工作模式，响应可能经 Director（NAT）或直接返回客户端（DR/TUN）|
| **无状态设计（推荐）** | 理想的 RS 应该是无状态的，方便水平扩缩容 |

### 2.3 不同模式下的 RS 配置要求

| 模式 | RS 配置要求 |
|:----:|-----------|
| **NAT** | RS 的**默认网关必须指向 Director**（否则响应包会走错路） |
| **DR** | RS 的 lo 接口需绑定 VIP，且设置 ARP 抑制（`arp_ignore=1` + `arp_announce=2`）|
| **TUN** | RS 需开启 IPIP 隧道支持（`modprobe ipip`）|

> [!warning] DR 模式 ARP 抑制
> 如果不做 ARP 抑制，RS 也会响应 ARP who-has VIP 的请求，导致客户端直接给 RS 发请求，绕过 Director 的调度逻辑，破坏负载均衡。

### 2.4 RS 的健康检查

Director 需要实时感知 RS 的健康状态。通常通过 **keepalived** 实现：

```bash
# keepalived 定期检查 RS 是否存活
# 如果 RS 宕机，IPVS 自动将其摘除
# RS 恢复后自动加回
```

```bash
# 手动查看 RS 状态
ipvsadm -L -n

# 输出示例
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.1.100:80 wlc
  -> 10.0.0.10:80                 Route   1      0          0
  -> 10.0.0.11:80                 Route   2      1          3
  -> 10.0.0.12:80                 Route   1      0          5         ← 权重为 0 或 Route 标记异常
```

| 字段 | 说明 |
|------|------|
| `Forward` | 转发模式：`Route`(DR)/`Masq`(NAT)/`Tunnel`(TUN) |
| `Weight` | 权重，为 0 表示该 RS **不参与调度**（被摘除） |
| `ActiveConn` | 当前活跃连接数 |
| `InActConn` | 非活跃连接数（如 HTTP Keep-Alive 空闲连接） |

---

## 三、Director 与 RS 的关系

### 3.1 流量路径视角

```
请求方向（三种模式）:
  NAT:  Client → Director(DNAT) → RS → Director(SNAT) → Client
  DR:   Client → Director(MAC改) → RS → Client（直返）
  TUN:  Client → Director(IP封装) → RS → Client（直返）
```

### 3.2 调度与分发的协议

IPVS 在内核中维护一个 VS（Virtual Server）规则，关联一组 RS：

```
VS 192.168.1.100:80 (虚拟服务)
  ├── RS 10.0.0.10:80  Weight=1  ActiveConn=3
  ├── RS 10.0.0.11:80  Weight=2  ActiveConn=5
  └── RS 10.0.0.12:80  Weight=1  ActiveConn=2
```

- Director 根据调度算法（如 wlc）计算目标 RS
- Director 在 IPVS CT 表中记录"某客户端连接 → 某 RS"
- 同一连接的后续包直接查表转发，无需重新调度

### 3.3 常见问题

| 问题 | 原因 |
|------|------|
| RS 没收到请求 | RS 网关未指向 Director（NAT 模式）或 ARP 未抑制（DR 模式）|
| 请求到达 RS 但响应不了 | RS 路由表配置错误，响应包走错了路径 |
| Director 健康检查失败 | RS 上的服务未启动或防火墙拦截了检查包 |
| 某 RS 权重为 0 仍有连接 | 已有连接不会中断，新连接不再分发到该 RS |

---

## 四、架构类比

| 比喻 | Director | Real Server |
|------|----------|-------------|
| **餐厅** | 前台领位员 | 后厨厨师 |
| **快递** | 分拣中心 | 各片区快递员 |
| **客服中心** | 呼叫分配器（ACD）| 客服坐席 |
| **K8s** | kube-proxy (IPVS 模式) | Service 背后的 Pod |

---

## 关联笔记

- [[IPVS-IP虚拟服务器详解|IPVS — IP Virtual Server 详解]] — IPVS 的核心原理、模式、配置
- [[ipvsadm命令详解|ipvsadm 命令详解]] — IPVS 的管理命令
- [[四层与七层负载均衡对比|四层与七层负载均衡对比]] — L4 vs L7 负载均衡选型
- [[Linux-IP转发与路由|Linux IP 转发与路由]] — IPVS NAT 模式依赖 IP 转发
- [[netfilter框架详解|Netfilter 框架详解]] — IPVS 底层依赖的 Hook 机制
