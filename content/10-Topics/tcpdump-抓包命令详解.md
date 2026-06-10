---
title: tcpdump 抓包命令详解
date: 2026-06-03
tags:
  - topic/tcpdump
  - topic/网络排查
  - topic/命令
  - topic/Linux
status: evergreen
aliases:
  - tcpdump
  - tcpdump 命令
  - 抓包命令
  - packet capture
---

# tcpdump 抓包命令详解

`tcpdump` 是 Linux/Unix 下最经典的**命令行网络抓包工具**，基于 [[tcpdump-过滤表达式|BPF（Berkeley Packet Filter）]] 实现高效的数据包过滤。它将网络接口设置为**混杂模式（Promiscuous Mode）** 以捕获所有经过的数据包。

## 基本语法

```bash
tcpdump [选项] [表达式]
```

- **选项**：控制抓包行为（接口、数量、输出格式等）
- **表达式**：BPF 过滤规则，只抓取匹配的数据包

## 常用选项速查

| 选项 | 说明 | 示例 |
|------|------|------|
| `-i <接口>` | 指定监听网卡 | `-i eth0` / `-i any`（所有接口） |
| `-n` | 不解析主机名（显示 IP） | `-n` |
| `-nn` | 不解析主机名和端口名 | `-nn` |
| `-c <数量>` | 抓取 N 个包后退出 | `-c 10` |
| `-s <长度>` | 设置 snaplen（每个包抓取字节数），`0` 表示完整包 | `-s 0` |
| `-v / -vv / -vvv` | 详细输出级别，越多越详细 | `-vv` |
| `-e` | 显示链路层头部（MAC 地址） | `-e` |
| `-X` | 以 **HEX + ASCII** 打印包内容 | `-X` |
| `-A` | 以 **ASCII** 打印包内容（适合 HTTP 等文本协议） | `-A` |
| `-xx` | 以 HEX 打印包内容（含链路层头） | `-xx` |
| `-q` | 精简输出 | `-q` |
| `-t` | 不打印时间戳 | `-t` |
| `-ttt` | 打印包间相对时间差（微秒） | `-ttt` |
| `-K` | 不验证 TCP 校验和 | `-K` |
| `-p` | 不启用混杂模式 | `-p` |
| `--time-stamp-precision <nano/micro>` | 时间戳精度（默认微秒，可设为纳秒） | `--time-stamp-precision nano` |

> [!tip] 抓包必备组合
> 绝大多数场景从以下组合开始：
> ```bash
> sudo tcpdump -i any -nn -c 100
> ```

## 输出解读

```bash
# 典型 TCP 包输出
12:34:56.789012 IP 192.168.1.1.443 > 192.168.1.100.54321: Flags [P.], seq 1:100, ack 200, win 65535, length 99
```

各部分含义：

| 字段 | 示例 | 说明 |
|------|------|------|
| 时间戳 | `12:34:56.789012` | 抓包时间（微秒精度） |
| 协议 | `IP` | 网络层协议（IP、IP6、ARP 等） |
| 源地址 | `192.168.1.1.443` | 源 IP.端口（`-n` 关闭反向解析） |
| 方向 | `>` | 数据流向 |
| 目的地址 | `192.168.1.100.54321` | 目的 IP.端口 |
| Flags | `[P.]` | TCP 标志位（见下方） |
| Seq | `seq 1:100` | 序列号范围 |
| Ack | `ack 200` | 确认号 |
| Win | `win 65535` | 窗口大小 |
| Length | `length 99` | 应用层数据长度 |

### TCP Flags 速记

```bash
[S]  = SYN      # 连接发起
[.]  = ACK      # 确认包
[P]  = PUSH     # 推送数据
[F]  = FIN      # 连接关闭
[R]  = RST      # 连接重置
[S.] = SYN+ACK  # 连接响应
[FP] = FIN+PUSH # 关闭并推送
```

## 注意事项

> [!warning] 权限要求
> tcpdump 需要 **root 权限**（Linux 需要 `CAP_NET_RAW` 和 `CAP_NET_ADMIN`），通常通过 `sudo` 执行。

> [!warning] 高流量环境
> 在流量大的接口上不加过滤地抓包，可能导致终端刷屏卡死。**始终先加过滤表达式**，或使用 `-c` 限制包数量。

## 关联笔记

- [[tcpdump-过滤表达式]] — BPF 过滤语法详解
- [[tcpdump-抓包保存与读取]] — pcap 文件操作
- [[tcpdump-常见场景示例]] — 实战场景集合
- [[lsof-文件诊断工具]] — 端口与进程诊断
- [[Linux-IP转发与路由]] — 网络层相关
