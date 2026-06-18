---
title: tcpdump 抓包保存与读取
date: 2026-06-03
aliases:
  - pcap 文件
  - tcpdump 保存
  - tcpdump -w
  - tcpdump -r
tags:
  - topic/Linux
  - topic/Linux/命令
  - topic/计算机网络
status: to-review
---

# tcpdump 抓包保存与读取

## 保存抓包到文件（-w）

```bash
sudo tcpdump -i any -w capture.pcap
```

- `-w` 将原始数据包写入文件，**不输出到终端**
- 文件格式为 **pcap**（packet capture），是 Wireshark、tshark 等工具的通用格式
- 配合过滤表达式只保存感兴趣的流量

### 分片保存（ring buffer）

长时间抓包时，用 `-C` + `-W` 控制文件大小和数量：

```bash
# 每个文件 64MB，最多 10 个文件，覆盖旧文件
sudo tcpdump -i any -w traffic.pcap -C 64 -W 10

# 加上 -G 按时间轮转（秒）
sudo tcpdump -i any -w traffic_%Y%m%d_%H%M%S.pcap -G 3600
```

| 选项 | 说明 |
|------|------|
| `-C <MB>` | 单个文件达到 <MB> 兆字节时切换新文件 |
| `-W <N>` | 最多保留 N 个文件，超过后覆盖最旧的 |
| `-G <秒>` | 每 <秒> 切换一个新文件（配合 `strftime` 命名） |
| `-z <命令>` | 文件关闭后自动调用压缩/处理命令（如 `-z gzip`） |

> [!example] 自动压缩历史包
> ```bash
> sudo tcpdump -i eth0 -w trace.pcap -C 100 -z gzip
> ```
> 每满 100MB 自动切分并压缩为 `trace.pcap.gz`（注意：压缩会消耗 CPU，高流量场景慎用）

## 读取 pcap 文件（-r）

```bash
tcpdump -r capture.pcap
```

读取时**可叠加过滤表达式**，只查看关心的包：

```bash
# 从保存的包中筛选 HTTP 流量
tcpdump -r capture.pcap 'tcp port 80'

# 查看 TCP 三次握手包
tcpdump -r capture.pcap 'tcp[tcpflags] & tcp-syn != 0'

# 读取时显示包内容
tcpdump -r capture.pcap -X
```

> [!tip] 读取时无须 root 权限
> `-r` 读取已有文件**不需要 `sudo`**，因为只是解析已保存的数据。

## 与 Wireshark 协作

### tcpdump 抓 → Wireshark 分析

```bash
# 1. 远程服务器上抓包并实时传输到本地
ssh user@server 'sudo tcpdump -i any -w - not port 22' | wireshark -k -i -

# 2. 抓包后用 scp 拉回本地分析
sudo tcpdump -i eth0 -w debug.pcap -c 10000
scp user@server:debug.pcap .
wireshark debug.pcap
```

### 合并多个 pcap 文件

```bash
# 使用 mergecap（Wireshark 工具包）
mergecap -w merged.pcap file1.pcap file2.pcap
```

## 性能注意事项

> [!warning] 高流量抓包建议
> 写入 pcap 文件时，tcpdump 本身也有性能开销：
> 1. **写不同磁盘**：将 pcap 写到与业务不相同的磁盘，减少 IO 竞争
> 2. **限制 snaplen**：用 `-s 128` 只抓每个包的头部（通常足够分析连接）
> 3. **优先用过滤**：`-w` 前先过滤，减少写入量
> 4. **后台运行**：配合 `nohup` 或 `screen` 避免终端中断

```bash
# 生产环境推荐的高效抓包
nohup sudo tcpdump -i eth0 -s 128 -w /data/capture.pcap \
  -C 256 -W 5 -z gzip \
  'not port 22 and not port 3306' > /dev/null 2>&1 &
```

## 关联笔记

- [[tcpdump-抓包命令详解]] — 基础语法与选项
- [[tcpdump-过滤表达式]] — 过滤表达式详解
- [[tcpdump-常见场景示例]] — 实战场景应用
