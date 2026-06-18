---
title: tcpdump 常见场景示例
date: 2026-06-03
aliases:
  - tcpdump 场景
  - tcpdump 实战
  - tcpdump 示例
tags:
  - topic/Linux
  - topic/Linux/命令
  - topic/计算机网络
status: to-review
---

# tcpdump 常见场景示例

> [!abstract] 场景索引
> - [TCP 三次握手排查](#tcp-三次握手排查)
> - [HTTP 请求调试](#http-请求调试)
> - [HTTPS/TLS 握手分析](#httpstls-握手分析)
> - [DNS 查询排查](#dns-查询排查)
> - [抓取特定进程流量](#抓取特定进程流量)
> - [Ping/ICMP 故障排查](#pingicmp-故障排查)
> - [端口不可达排查](#端口不可达排查)
> - [网络性能分析](#网络性能分析)
> - [DHCP 交互观察](#dhcp-交互观察)

## TCP 三次握手排查

> 场景：服务端端口已监听，但客户端连接超时，排查握手是否完成。

```bash
# 观察完整的握手交互（只抓 SYN/FIN/RST 标志包）
sudo tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0' and host 10.0.0.1 and host 10.0.0.2
```

```text
# 正常三次握手输出：
10:00:00.1 IP 10.0.0.1.54321 > 10.0.0.2.80: Flags [S], seq 1000
10:00:00.2 IP 10.0.0.2.80 > 10.0.0.1.54321: Flags [S.], seq 2000, ack 1001
10:00:00.3 IP 10.0.0.1.54321 > 10.0.0.2.80: Flags [.], ack 2001
```

> [!tip] 判断问题
> - 只看到 `[S]` 无响应 → 服务器未监听或防火墙拦截
> - 看到 `[S.]` 但客户端无 `[.]` → 客户端侧问题
> - 看到 `[R]` 回复 → 端口未开放（Connection Refused）

详见 [[tcpdump-过滤表达式#TCP 标志位过滤|TCP 标志位过滤]]。

## HTTP 请求调试

> 场景：确认客户端发送了什么请求，服务器返回了什么内容。

```bash
# 查看 HTTP GET/POST 请求行（-A 以 ASCII 显示）
sudo tcpdump -i any -nn -A 'tcp port 80 and host target.example.com'

# 更精简：只打印前 1500 字节，包含 HTTP 头部即可
sudo tcpdump -i any -nn -s 1500 -A 'tcp port 80 and host target.example.com'
```

```text
# 输出样例（-A 模式下可直接看到 HTTP 内容）：
GET /api/v1/users HTTP/1.1
Host: target.example.com
User-Agent: curl/7.68.0
...
HTTP/1.1 200 OK
Content-Type: application/json
...
```

> [!note] HTTP/2 提示
> HTTP/2 是二进制协议，`-A` 无法直接看到可读文本，建议用 `-X` 查看 HEX 内容，或配合 Wireshark 分析。

## HTTPS/TLS 握手分析

> 场景：诊断 TLS 版本、证书协商、密码套件问题。

```bash
# 只抓 Client Hello（TLS 握手开始）
sudo tcpdump -i any -nn -s 0 'tcp port 443 and host target.example.com'

# 抓取后保存，用 Wireshark 详细分析
sudo tcpdump -i any -nn -s 0 -w tls.pcap 'tcp port 443'
```

```text
# TLS Client Hello 关键字段
Client Hello
  Version: TLS 1.2
  Cipher Suites: ...  # 客户端支持的密码套件
  SNI: target.example.com  # 服务器名称指示
```

## DNS 查询排查

> 场景：域名解析慢或解析到错误 IP。

```bash
# 捕获 DNS 查询和响应
sudo tcpdump -i any -nn -s 0 'udp port 53'

# 只看特定域名的 DNS 查询
sudo tcpdump -i any -nn -s 0 'udp port 53 and host 8.8.8.8'
```

```text
# DNS 查询输出
10:00:00.1 IP 192.168.1.1.54321 > 8.8.8.8.53: 12345+ A? example.com. (30)
10:00:00.2 IP 8.8.8.8.53 > 192.168.1.1.54321: 12345 1/0/0 A 93.184.216.34 (46)
                      ^^^^^^^^ 事务 ID   ^ 回答数 ^ 解析到的 IP
```

> [!tip] DNS 输出解读
> - `A? example.com` — 查询 A 记录
> - `1/0/0` — `回答数/权威DNS数/附加记录数`
> - 高延迟的 DNS 响应通常是网络问题或上游 DNS 慢

## 抓取特定进程流量

> 场景：某个进程（如 nginx、java 进程）产生异常流量，需要定位。

tcpdump 本身不支持按进程过滤，需要组合 `lsof` 或 `netstat` 先确定端口：

```bash
# 1. 找到进程监听的端口
sudo lsof -i -P -n | grep java

# 2. 根据端口抓包
sudo tcpdump -i any -nn 'port 8080'
```

```bash
# 一行命令：获取进程 PID 并监听其所有连接
PID=12345
PORTS=$(sudo lsof -i -P -n -p $PID | awk '{print $9}' | grep -oP '\d+$' | sort -u | tr '\n' ' ')
sudo tcpdump -i any -nn "port $PORTS"
```

详见 [[lsof-文件诊断工具]]。

## Ping/ICMP 故障排查

> 场景：Ping 不通或者丢包率高。

```bash
# 捕获 ICMP Echo 请求和回复
sudo tcpdump -i any -nn icmp

# 只看特定主机的 ICMP
sudo tcpdump -i any -nn 'icmp and host 8.8.8.8'
```

```text
# ICMP 正常 Echo
10:00:00.1 IP 192.168.1.1 > 8.8.8.8: ICMP echo request, id 1000, seq 1, length 64
10:00:00.2 IP 8.8.8.8 > 192.168.1.1: ICMP echo reply, id 1000, seq 1, length 64

# 异常：收到了 ICMP 不可达
10:00:00.3 IP 10.0.0.1 > 192.168.1.1: ICMP host 203.0.113.1 unreachable
```

> [!tip] 常见 ICMP 类型
> - `echo request` (Type 8) — Ping 请求
> - `echo reply` (Type 0) — Ping 回复
> - `destination unreachable` (Type 3) — 目标不可达
> - `time exceeded` (Type 11) — TTL 超时（traceroute）

## 端口不可达排查

> 场景：telnet 某个端口失败，判断是防火墙还是服务问题。

```bash
# 抓端口 8080 上的流量，观察是否收到 RST
sudo tcpdump -i any -nn 'tcp port 8080'
```

```text
# 收到 RST → 端口未监听（Connection Refused）
10:00:00.1 IP 10.0.0.1.54321 > 10.0.0.2.8080: Flags [S], seq 1000
10:00:00.2 IP 10.0.0.2.8080 > 10.0.0.1.54321: Flags [R.], seq 0, ack 1001

# 无任何回复 → 防火墙丢弃（silent drop）
10:00:00.1 IP 10.0.0.1.54321 > 10.0.0.2.8080: Flags [S], seq 1000
# ... 然后客户端重试 ...
```

> [!tip] RST vs 无回复
> - **收到 RST** → 端口未监听（服务没启动）
> - **无响应**（超时）→ 防火墙拦截或路由不通

## 网络性能分析

> 场景：接口响应慢，需要看网络耗时分布。

```bash
# 用 -ttt 显示包间相对时间差（微秒），观察延迟
sudo tcpdump -i any -nn -ttt 'host 10.0.0.1'
```

```text
# 输出显示每个包与上一个包的时间差
00:00:00.000000 IP 10.0.0.1.54321 > 10.0.0.2.80: Flags [S], seq 1000
 00:00:00.123456 IP 10.0.0.2.80 > 10.0.0.1.54321: Flags [S.], seq 2000, ack 1001
 00:00:00.000015 IP 10.0.0.1.54321 > 10.0.0.2.80: Flags [.], ack 2001
 00:00:00.234567 IP 10.0.0.1.54321 > 10.0.0.2.80: Flags [P.], seq 1001:1100, ack 2001
 00:00:00.456789 IP 10.0.0.2.80 > 10.0.0.1.54321: Flags [.], ack 1100

# 解读：
# 0.123456s = SYN → SYN-ACK（服务器响应时间 + 网络延迟）
# 0.000015s = SYN-ACK → ACK（接近 0，纯客户端内核处理时间）
# 0.234567s = 请求发出到确认（应用层响应时间）
```

```bash
# 统计 TCP 重传（持续观察）
sudo tcpdump -i any -nn 'tcp[tcpflags] & tcp-syn != 0 or tcp[tcpflags] & tcp-rst != 0'
# 大量重传 → 网络丢包；大量 RST → 连接异常
```

## DHCP 交互观察

> 场景：客户端获取不到 IP 地址，排查 DHCP 流程。

```bash
# DHCP 使用 UDP 67（服务端）、68（客户端）
sudo tcpdump -i any -nn 'udp port 67 or udp port 68'
```

```text
# 正常 DHCP 四步握手
1. DISCOVER  — 客户端广播找 DHCP 服务器
2. OFFER     — 服务器提供 IP 地址
3. REQUEST   — 客户端确认请求
4. ACK       — 服务器最终确认
```

## 关联笔记

- [[tcpdump-抓包命令详解]] — 基础语法与选项
- [[tcpdump-过滤表达式]] — 过滤表达式详解
- [[tcpdump-抓包保存与读取]] — pcap 文件操作
- [[lsof-文件诊断工具]] — 根据进程找端口
- [[Linux-IP转发与路由]] — 网络层路由相关
