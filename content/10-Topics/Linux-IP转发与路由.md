---
title: Linux-IP转发与路由
date: 2026-06-02
updated: 2026-06-02 12:00:00
tags:
  - topic/Linux
  - topic/计算机网络
status: to-review
---

## 核心概念

**IP 转发**（IP Forwarding）是 Linux 内核将接收到的数据包根据路由表从一个网络接口转发到另一个网络接口的能力。此时 Linux 充当 **路由器** 角色。

## 启用 IP 转发

```bash
# 临时启用（立即生效，重启丢失）
echo 1 > /proc/sys/net/ipv4/ip_forward

# 持久化配置
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
# 或写入 /etc/sysctl.d/99-ipforward.conf
sysctl -p  # 立即生效
```

## Linux 三层转发流程

```
┌─ 数据包到达 eth1 ─────────────────────────┐
│                                            │
│  1. PREROUTING（DNAT 检查）                 │
│  2. 路由决策（查路由表，决定转发到哪）       │
│  3. FORWARD 链（过滤规则检查）              │
│  4. POSTROUTING（SNAT/MASQUERADE 检查）     │
│                                            │
│  5. 从 eth0/docker0 发送出去               │
└────────────────────────────────────────────┘
```

**核心检查点**：
- `ip_forward = 1`：必须开启，否则内核直接丢弃非本机目的地的包
- `iptables FORWARD 链`：默认可能为 DROP（Docker 会设置 ACCEPT），需显式放行
- `路由表`：必须知道目标网络从哪个接口/网关可达

## 路由操作

```bash
# 查看路由表
ip route show
# 或
route -n

# 添加静态路由
ip route add 172.17.0.0/16 via 192.168.236.128 dev eth0
route add -net 172.17.0.0 netmask 255.255.0.0 gw 192.168.236.128

# 删除路由
ip route del 172.17.0.0/16

# 永久路由（CentOS /etc/sysconfig/network-scripts/route-eth0）
# Ubuntu：写入 /etc/network/interfaces 或 netplan
```

## 容器通信中的路由场景

```
容器(172.17.0.2) ── docker0 ── VM ── eth1 ── Windows ── 手机
                    172.17.0.1    192.168.236.128  172.21.0.1/2
```

### 转发配置要点

1. **VM 必须开启 ip_forward**：允许包在 docker0 与 eth1 间流转
2. **Windows 需添加静态路由**：去往 172.17.0.0/16 的流量指向 VM
   ```bash
   route add -p 172.17.0.0 mask 255.255.0.0 192.168.236.128
   ```
3. **配置 iptables FORWARD ACCEPT**：
   ```bash
   iptables -A FORWARD -i docker0 -o eth1 -j ACCEPT
   iptables -A FORWARD -i eth1 -o docker0 -j ACCEPT
   ```

## 面试要点

- Linux 默认 **不转发** IP 包，`ip_forward=0` 时跨接口流量被直接丢弃（安全考量）
- 转发路径上的 **三个必要条件**：`ip_forward=1` + FORWARD 链 ACCEPT + 路由表正确
- Docker 自动管理 iptables 规则，但自定义转发需手动配置
- `ip route` 是现代的 `route` 命令替代品，功能更丰富
- 使用 `traceroute` 或 `tracepath` 可排查转发路径故障

## 参考链接

- [柠檬微趣-笔试-Q3-网络配置](../30-Secret-Questions/柠檬微趣-笔试-Q3-网络配置.md)
- [[iptables端口转发]]
- [[Docker网络模式-bridge]]
- [[Windows端口转发-proxy]]
