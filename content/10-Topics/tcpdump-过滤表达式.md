---
title: tcpdump 过滤表达式
date: 2026-06-03
tags:
  - topic/tcpdump
  - topic/网络
  - topic/BPF
status: evergreen
aliases:
  - BPF 过滤
  - tcpdump 过滤
  - 伯克利包过滤器
  - Berkeley Packet Filter
---

# tcpdump 过滤表达式

tcpdump 使用 **BPF（Berkeley Packet Filter）** 表达式来精确筛选数据包。理解 BPF 语法是高效使用 [[tcpdump-抓包命令详解|tcpdump]] 的核心。

## 表达式结构

```bash
[协议] [方向] [类型] [值]
```

三个维度组合，缺省时表示"匹配所有"。

## 类型（Type）

指定过滤的目标类型：

| 关键字 | 说明 | 示例 |
|--------|------|------|
| `host` | 匹配主机（默认类型） | `host 192.168.1.1` |
| `net` | 匹配网段 | `net 192.168.0.0/16` |
| `port` | 匹配端口 | `port 80` |
| `portrange` | 匹配端口范围 | `portrange 8000-8080` |

## 方向（Direction）

指定数据流方向：

| 关键字 | 说明 | 示例 |
|--------|------|------|
| `src` | 源地址/端口 | `src host 10.0.0.1` |
| `dst` | 目的地址/端口 | `dst port 443` |
| `src or dst` | 源或目的（默认） | `src or dst port 53` |
| `src and dst` | 源且目的 | `src 10.0.0.1 and dst 10.0.0.2` |

## 协议（Protocol）

指定网络/传输层协议：

| 关键字 | 说明 |
|--------|------|
| `tcp` | TCP 协议 |
| `udp` | UDP 协议 |
| `icmp` | ICMP 协议 |
| `arp` | ARP 协议 |
| `ip` | IPv4 协议 |
| `ip6` | IPv6 协议 |

> [!tip] 协议过滤的底层原理
> 协议过滤实际是对 IP 头部 `protocol` 字段的匹配：
> - `tcp` → protocol = 6
> - `udp` → protocol = 17
> - `icmp` → protocol = 1

## 逻辑运算符

组合多个过滤条件：

| 运算符 | 替代写法 | 说明 | 示例 |
|--------|----------|------|------|
| `and` | `&&` | 与 | `tcp and port 80` |
| `or` | `\|\|` | 或 | `port 80 or port 443` |
| `not` | `!` | 非 | `not arp` |

> [!tip] 优先级
> 优先级：`not` > `and` > `or`。建议**始终用括号**明确优先级：
> ```bash
> tcpdump 'host 10.0.0.1 and (port 80 or port 443)'
> ```

## 高级过滤

### TCP 标志位过滤

```bash
# SYN 包
tcp[tcpflags] & tcp-syn != 0
# 简写
'tcp[tcpflags] == tcp-syn'

# RST 包
'tcp[tcpflags] & tcp-rst != 0'

# SYN+ACK 包（多个标志位）
'tcp[tcpflags] == (tcp-syn|tcp-ack)'

# 只含 SYN 不含 ACK
'tcp[tcpflags] & (tcp-syn) != 0 and tcp[tcpflags] & (tcp-ack) == 0'
```

### 包大小过滤

```bash
# 小于 100 字节的小包
'less 100'

# 大于 500 字节的大包
'greater 500'
```

### MAC 地址过滤

```bash
# 源 MAC
ether src aa:bb:cc:dd:ee:ff

# 目的 MAC
ether dst aa:bb:cc:dd:ee:ff
```

### VLAN 过滤

```bash
# VLAN ID 100
vlan 100
```

## 常用过滤组合

```bash
# 排除 SSH 连接（避免抓到自己的终端流量）
'not port 22'

# 特定主机间的 HTTP 通信
'host 10.0.0.1 and tcp port 80'

# DNS 查询与响应（排除端口 53 的 TCP）
'udp port 53'

# 广播和多播流量
'broadcast' 或 'multicast'

# 排除 ARP 和 DHCP
'not (arp or port 67 or port 68)'
```

> [!warning] 引号使用
> 包含 `()` 或 `!` 等特殊字符时，**必须用单引号包裹表达式**，防止 Shell 解释：
> ```bash
> # 正确
> tcpdump -i any 'tcp[tcpflags] & tcp-syn != 0'
> # 错误（Shell 会把 () 当作子进程）
> tcpdump -i any tcp[tcpflags] & tcp-syn != 0
> ```

## 关联笔记

- [[tcpdump-抓包命令详解]] — 基础语法与选项
- [[tcpdump-抓包保存与读取]] — 保存过滤后的结果
- [[tcpdump-常见场景示例]] — 实战中的过滤应用
- [[iptables端口转发]] — 同属网络排查工具链
