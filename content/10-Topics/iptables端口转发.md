---
title: iptables端口转发
date: 2026-06-02
tags:
  - topic/计算机网络
  - topic/Linux
  - topic/运维
status: to-review
updated: 2026-06-02 12:00:00
---

## 核心概念

**iptables** 是 Linux 内核的包过滤框架，通过 **NAT（Network Address Translation）** 表实现端口转发。核心是修改网络包的目标地址（DNAT）或源地址（SNAT），实现跨网络通信。

## 关键链与表

```
数据包流向：
PREROUTING → [路由决策] → FORWARD → POSTROUTING
                  ↓
              INPUT（本机进程）
                  ↓
             [本地进程]
                  ↓
             OUTPUT → POSTROUTING
```

| 链 | 表 | 作用时机 | 典型用途 |
|----|-----|---------|---------|
| **PREROUTING** | nat | 路由决策前 | DNAT：修改目标IP/端口（入站转发） |
| **POSTROUTING** | nat | 路由决策后、出站前 | SNAT/MASQUERADE：修改源IP |
| **FORWARD** | filter | 路由决策后 | 允许/拒绝经过本机的转发流量 |
| **INPUT** | filter | 发往本机进程前 | 控制本机服务访问 |
| **OUTPUT** | nat | 本机进程发包时 | 本机发出流量的源地址修改 |

## DNAT（目标地址转换）

```bash
# 将宿主机 8080 端口 → 容器 80 端口
iptables -t nat -A PREROUTING -d <宿主机IP> -p tcp --dport 8080 \
  -j DNAT --to-destination 172.17.0.2:80

# 必须允许 FORWARD
iptables -A FORWARD -p tcp -d 172.17.0.2 --dport 80 -j ACCEPT
```

## SNAT / MASQUERADE（源地址转换）

```bash
# 固定源地址转换
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 -o eth0 \
  -j SNAT --to-source 192.168.1.100

# 动态源地址转换（自动取出口IP）
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 \
  -j MASQUERADE
```

| 对比 | SNAT | MASQUERADE |
|------|------|-----------|
| 目标地址 | 固定 `--to-source IP` | 自动取出口IP |
| 性能 | 较快 | 略慢（动态查IP） |
| 适用场景 | 静态公网IP | PPPoE/动态IP |

## 完整实例：跨主机容器访问

```bash
# 1. 开启转发
echo 1 > /proc/sys/net/ipv4/ip_forward

# 2. DNAT：外部→容器
iptables -t nat -A PREROUTING -p tcp --dport 80 \
  -j DNAT --to-destination 172.17.0.2:80

# 3. FORWARD：允许转发
iptables -A FORWARD -p tcp -d 172.17.0.2 --dport 80 -j ACCEPT

# 4. SNAT：容器回包能到达外部
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 \
  -j MASQUERADE
```

## 规则持久化

```bash
# 保存规则
iptables-save > /etc/iptables/rules.v4

# 重启自动加载（安装 iptables-persistent）
apt install iptables-persistent
netfilter-persistent save
```

## 面试要点

- DNAT 修改 **目标 IP**，在 PREROUTING 链完成；SNAT 修改 **源 IP**，在 POSTROUTING 链完成
- 使用 DNAT 时**必须**配合 FORWARD 规则，否则包被丢弃
- Docker 的 `-p` 参数本质是自动添加 PREROUTING DNAT + FORWARD ACCEPT 规则
- `MASQUERADE` 是 SNAT 的变体，自动获取出口网卡 IP（适合动态IP场景）
- ICMP 协议不能被 DNAT/端口转发代理，如需 ping 通需配置静态路由

## 参考链接

- [柠檬微趣-笔试-Q3-网络配置](../30-Secret-Questions/柠檬微趣-笔试-Q3-网络配置.md)
- [[Docker网络模式-bridge]]
- [[Linux-IP转发与路由]]
- [[Windows端口转发-proxy]]
