---
title: TCP 三次握手与可靠性机制
date: 2026-06-15
aliases:
  - 三次握手
  - SYN Flood
  - TIME_WAIT
  - 2MSL
  - SYN-ACK
tags:
  - topic/计算机网络
status: to-review
---

# TCP 三次握手与可靠性机制

> TCP 通过**三次握手**建立可靠连接，使用 **SYN Flood** 防御机制保护服务端资源，**2MSL** 等待确保全双工连接可靠关闭。

## 三次握手过程

```
Client                          Server
  |           SYN (seq=x)         |
  | ──────────────────────────>  |
  |       SYN-ACK (seq=y, ack=x+1)|
  | <──────────────────────────  |
  |     ACK (seq=x+1, ack=y+1)   |
  | ──────────────────────────>  |
```

1. **SYN**：客户端发送 SYN（同步序列号），进入 `SYN_SENT` 状态
2. **SYN-ACK**：服务端回复 SYN + ACK（确认号 = x+1），进入 `SYN_RCVD` 状态
3. **ACK**：客户端发送 ACK，连接建立，双方进入 `ESTABLISHED` 状态

## SYN-ACK 丢了客户端会怎样？

### 行为

- 客户端收不到 SYN-ACK，认为 SYN 包丢失
- **重传 SYN**，重传超时时间（RTO）按指数退避：
  - 第 1 次重传：1 秒后
  - 第 2 次重传：2 秒后
  - 第 3 次重传：4 秒后...
  - 直到达到 `tcp_syn_retries` 上限（默认 6 次，总耗时约 127 秒）

### 服务端状态

- 服务端已发送 SYN-ACK，进入 `SYN_RCVD` 状态
- 分配了连接资源（半连接队列）
- 服务端也会超时重传 SYN-ACK（由 `tcp_synack_retries` 控制）
- 如果客户端重传 SYN，服务端再次回复 SYN-ACK（仍用相同的 seq=y）

## SYN Flood 防御

### 攻击原理

攻击者伪造大量 SYN 包，不回复最终的 ACK，导致服务端半连接队列满（内存耗尽），正常连接无法建立。

### 防御技术

| 防御方法 | 原理 | 优点 | 缺点 |
|---------|------|------|------|
| **SYN Cookie** | 不分配连接资源，用 Cookie 作为初始序列号 | 零资源消耗 | 占用 CPU 计算 |
| **SYN Cache** | 有限哈希表存储半连接 | 控制内存占用 | 超过仍会丢弃 |
| **缩短 SYN Timeout** | 减小 `tcp_synack_retries` | 简单有效 | 可能误伤正常用户 |
| **tcp_max_syn_backlog** | 限制半连接队列长度 | 防止内存耗尽 | 超过阈值直接丢弃 |
| **反向探测** | 先发 SYN-ACK，收到 RST 说明真实 | 精确 | 增加握手延迟 |

### SYN Cookie 原理

1. 服务端收到 SYN 后，**不分配任何连接资源**（不进入半连接队列）
2. 计算 Hash(源IP, 源端口, 目的IP, 目的端口, 服务端密钥) 作为初始序列号（Cookie）
3. 回复 SYN-ACK（seq=Cookie）
4. 收到客户端 ACK 后，验证 ACK 中的确认号是否等于 Cookie+1
5. 验证通过才分配连接资源，进入 `ESTABLISHED`

> Linux 下 `net.ipv4.tcp_syncookies = 1` 默认开启，在半连接队列满时自动启用。

## TIME_WAIT 为什么等 2MSL？

### MSL（Maximum Segment Lifetime）

- 报文最大生存时间
- Linux 默认 **30 秒**（实际硬编码为 60 秒）
- 任何 TCP 报文在网络中的最大存活时间

### TIME_WAIT 作用

**1. 保证最后一个 ACK 到达服务端**

```
主动关闭方                   被动关闭方
   FIN_WAIT_1     FIN        CLOSE_WAIT
   ──────────────────────────>
   FIN_WAIT_2     ACK        CLOSE_WAIT
   <──────────────────────────
    TIME_WAIT     FIN        LAST_ACK
   <──────────────────────────
    TIME_WAIT     ACK        CLOSED
   ──────────────────────────>
       ↑ 如果这个 ACK 丢了，服务端重传 FIN
       ↑ 2MSL 确保客户端能再次 ACK
```

- 如果最后的 ACK 丢失，服务端会重传 FIN
- 2MSL 时间内，客户端可以收到重传的 FIN 并重新发送 ACK

**2. 防止旧连接数据包干扰新连接**

- 2MSL 足够网络中所有属于这个连接的数据包消失
- 避免同样四元组的新连接收到旧连接延迟到达的数据

### TIME_WAIT 过多的问题

- 占用文件描述符和端口资源
- 高并发短连接场景（如 HTTP 短连接）容易出现端口耗尽
- **优化**：打开 `tcp_tw_reuse`（重用 TIME_WAIT 端口，需要时间戳选项）

## 参考链接

- [[快手电商-一面-19题总结]] — Q7 TCP 三次握手
- [[iptables详解]] — 网络层/传输层过滤
- [[tcpdump-抓包命令详解]] — TCP 抓包分析
