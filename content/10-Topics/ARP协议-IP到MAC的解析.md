---
title: ARP协议-IP到MAC的解析
date: 2026-08-05
updated: 2026-09-22
aliases:
  - ARP
  - 地址解析协议
  - ARP cache
  - Neighbor Discovery Protocol
source: "https://www.cnblogs.com/vamei/archive/2012/11/30/2794917.html"
related:
  - "[[IP包接力与路由表]]"
  - "[[IPv4地址分类与子网掩码]]"
tags:
  - topic/网络协议
  - topic/TCP-IP
status: to-review
---

# ARP 协议：IP 到 MAC 的解析

## 为什么需要 ARP

IP 包接力有一个隐含前提：**每台主机和路由器都知道局域网内 IP 地址与 MAC 地址的对应关系**。这是把 IP 包**封装（encapsulation）成帧**的基本条件——帧头部必须填 MAC 地址，而路由决策用的是 IP 地址。

这个对应关系由 **ARP 协议**传播到局域网中每台主机和路由器。

> [!info] 协议层次
> ARP 介于**连接层与网络层之间**。ARP 包本身需要被包裹在一个帧中才能发送。

## ARP 的工作过程

每台主机/路由器维护一个 **ARP cache**，存储局域网内 IP ↔ MAC 的映射：

1. 主机发出一个 ARP 包，包中包含**自己的 IP 和 MAC**
2. 以**广播**形式询问局域网内所有主机和路由器：
   > "我是 IP `xxxx`，我的 MAC 是 `xxxx`，有人知道 `199.165.146.4` 的 MAC 吗？"
3. **拥有该 IP 的主机**单播回复：
   > "我知道，这个 IP 属于我的一个 NIC，MAC 是 `xxxxxx`"
4. 由于请求是广播且**附带了发送方的 IP/MAC**，其他主机和路由器收到后会**同时检查并更新自己的 ARP cache**（不符合的也借机更新）
5. 经过几次请求后，ARP cache 达到稳定；局域网设备变动时重复上述过程

> [!tip] 一个精巧的设计
> ARP 通过"广播请求 + 附带自身信息"让**所有**监听者都能更新缓存，而不仅仅是被问的那一台。这让整个局域网的 ARP cache 能以较低成本快速收敛。

## 查看与排查

```bash
arp -a              # 查看 ARP 缓存（Linux / Windows 通用）
ip neigh            # Linux 现代写法

# 常见异常：同一 MAC 对应多个 IP（可能是 ARP 欺骗）
arp -a | sort -k4
```

## 适用范围

| 协议 | 地址解析机制 |
|---|---|
| **IPv4** | ARP |
| **IPv6** | **Neighbor Discovery Protocol（NDP）**，替代 ARP 的功能 |

> [!warning] ARP 只用于 IPv4
> IPv6 环境中不存在 ARP，排查时不要找 ARP 表，应查邻居表（`ip -6 neigh`）。

## 安全视角

ARP 协议**没有认证机制**，任何主机都可以声称"某 IP 的 MAC 是我"，从而：

- 冒充网关，截获整个局域网的出网流量（中间人）
- 造成大范围断网

因此生产网络常在交换机上启用 **DAI（Dynamic ARP Inspection）** 与 **IP-MAC 绑定**来缓解。这也是 [[MITM-中间人攻击]] 在局域网内的典型实现路径。

## 参考链接

- [[IP包接力与路由表]] — ARP 服务于 IP 包的帧封装
- [[IPv4地址分类与子网掩码]] — 同局域网的判定依据
