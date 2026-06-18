---
title: ipvsadm命令详解
date: 2026-06-10
updated: 2026-06-10
tags:
  - topic/计算机网络
  - topic/负载均衡
  - topic/Linux/命令
status: to-review
---

## 概述

**ipvsadm** 是 Linux IPVS（IP Virtual Server）的用户空间管理工具，用于配置和管理内核中的 LVS 负载均衡规则。

> ipvsadm 之于 IPVS，就如同 iptables 之于 netfilter：**ipvsadm 是用户空间命令，真正干活的是内核中的 IPVS 模块**。

---

## 一、语法结构

```
ipvsadm -A|E|D -t|u|f 服务地址 [-s 调度算法]
ipvsadm -a|e|d -t|u|f 服务地址 -r 服务器地址 [-g|i|m] [-w 权重]
ipvsadm -L|l [选项]
ipvsadm -C
ipvsadm -S|-R
```

### 常用命令分类

| 操作  | 命令           | 作用对象                    |
| --- | ------------ | ----------------------- |
| 增   | `-A, -a`     | 添加虚拟服务 / 添加 Real Server |
| 改   | `-E, -e`     | 修改虚拟服务 / 修改 Real Server |
| 删   | `-D, -d`     | 删除虚拟服务 / 删除 Real Server |
| 查   | `-L`（或 `-l`） | 列出当前配置和统计数据             |

### 服务地址类型

| 选项 | 含义 | 示例 |
|------|------|------|
| `-t` | TCP 虚拟服务 | `192.168.1.100:80` |
| `-u` | UDP 虚拟服务 | `192.168.1.100:53` |
| `-f` | firewall mark（防火墙标记） | `-f 100`，用于将多端口服务编组 |

### 转发模式（Real Server）

| 选项 | 模式 | 全称 |
|------|:----:|------|
| `-g` | **DR** | Direct Routing（默认） |
| `-i` | **TUN** | IP Tunneling |
| `-m` | **NAT** | Network Access Translation |

> [!note] 默认转发模式
> 不指定 `-g` / `-i` / `-m` 时，ipvsadm **默认使用 DR 模式**（`-g`）。

---

## 二、虚拟服务管理

### 2.1 添加虚拟服务

```bash
# TCP 虚拟服务，加权轮询
ipvsadm -A -t 192.168.1.100:80 -s wrr

# UDP 虚拟服务，最少连接
ipvsadm -A -u 192.168.1.100:53 -s lc

# 防火墙标记，将 80 和 443 编组到同一服务
iptables -t mangle -A PREROUTING -d 192.168.1.100 -p tcp -m multiport --dports 80,443 -j MARK --set-mark 100
ipvsadm -A -f 100 -s rr
```

### 2.2 修改虚拟服务

```bash
# 修改调度算法为加权最少连接
ipvsadm -E -t 192.168.1.100:80 -s wlc
```

### 2.3 删除虚拟服务

```bash
# 删除单个虚拟服务（同时删除其下的所有 Real Server）
ipvsadm -D -t 192.168.1.100:80

# 清空所有 IPVS 配置
ipvsadm -C
```

---

## 三、Real Server 管理

### 3.1 添加 Real Server

```bash
# NAT 模式，权重 1
ipvsadm -a -t 192.168.1.100:80 -r 10.0.0.10:8080 -m -w 1

# DR 模式，权重 2
ipvsadm -a -t 192.168.1.100:80 -r 10.0.0.11:80 -g -w 2

# 无权重添加（默认权重 1）
ipvsadm -a -t 192.168.1.100:80 -r 10.0.0.12:80 -m
```

### 3.2 修改 Real Server

```bash
# 调整权重
ipvsadm -e -t 192.168.1.100:80 -r 10.0.0.10:8080 -m -w 3
```

### 3.3 删除 Real Server

```bash
ipvsadm -d -t 192.168.1.100:80 -r 10.0.0.10:8080
```

---

## 四、查看与监控

### 4.1 查看虚拟服务列表

```bash
# 基本查看
ipvsadm -L -n

# 查看详细信息（包含每个 RS 的连接数和统计）
ipvsadm -L -n --stats

# 查看速率统计
ipvsadm -L -n --rate

# 查看超时配置
ipvsadm -L --timeout

# 查看 IPVS 连接跟踪表
ipvsadm -L -n -c

# 查看单线程模式下的同步守护进程状态
ipvsadm -L --daemon
```

### 4.2 输出字段说明

```bash
$ ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.1.100:80 wlc
  -> 10.0.0.10:80                 Masq    1      0          0
  -> 10.0.0.11:80                 Masq    2      0          0
```

| 字段 | 说明 |
|------|------|
| `Prot` | 协议（TCP/UDP） |
| `LocalAddress:Port` | VIP |
| `Scheduler` | 调度算法 |
| `RemoteAddress:Port` | 真实服务器地址 |
| `Forward` | 转发模式（Masq=NAT, Route=DR, Tunnel=TUN） |
| `Weight` | 权重 |
| `ActiveConn` | 当前活跃连接数 |
| `InActConn` | 当前非活跃连接数 |

### 4.3 --stats 统计字段

```bash
$ ipvsadm -L -n --stats
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
  -> RemoteAddress:Port
TCP  192.168.1.100:80                    5       40       60    40000   60000
  -> 10.0.0.10:80                        2       15       20    15000   20000
  -> 10.0.0.11:80                        3       25       40    25000   40000
```

| 字段 | 说明 |
|------|------|
| `Conns` | 总连接数（历史累计） |
| `InPkts` / `OutPkts` | 入/出包数 |
| `InBytes` / `OutBytes` | 入/出字节数 |

---

## 五、保存与恢复

```bash
# 保存当前所有 IPVS 配置到文件
ipvsadm-save > /etc/sysconfig/ipvsadm

# 从文件恢复 IPVS 配置
ipvsadm-restore < /etc/sysconfig/ipvsadm

# 配合 systemd 自动加载
systemctl enable ipvsadm
systemctl start ipvsadm
```

---

## 六、完整配置示例

### 6.1 NAT 模式负载均衡

```bash
# Director 配置
echo 1 > /proc/sys/net/ipv4/ip_forward

# 添加虚拟服务
ipvsadm -A -t 192.168.1.100:80 -s wlc

# 添加 Real Server（NAT 模式）
ipvsadm -a -t 192.168.1.100:80 -r 192.168.2.10:80 -m -w 1
ipvsadm -a -t 192.168.1.100:80 -r 192.168.2.11:80 -m -w 2

# 查看
ipvsadm -L -n
```

### 6.2 健康检查配合（keepalived）

生产环境中 IPVS **通常与 keepalived 配合使用**：

- **keepalived** 负责 VIP 漂移（高可用）+ **Real Server 健康检查**
- IPVS 规则由 keepalived 自动管理，无需手动 `ipvsadm` 操作

```bash
# keepalived 检测到 RS 宕机 → 自动 ipvsadm -d 摘除
# keepalived 检测到 RS 恢复 → 自动 ipvsadm -a 加入
```

### 6.3 查看超时与连接

```bash
# 查看超时时间
ipvsadm --set 120 10 60
# 格式：ipvsadm --set tcp tcpfin udp（单位：秒）
# 默认值：300 120 300

# 查看当前连接
ipvsadm -L -n -c | head -20
```

---

## 关联笔记

- [[IPVS-IP虚拟服务器详解|IPVS — IP Virtual Server 详解]] — IPVS 原理与三种工作模式
- [[netfilter框架详解|Netfilter 框架详解]] — IPVS 底层依赖的 netfilter Hook 机制
- [[iptables详解|iptables 详解]] — 另一方向上的 netfilter 用户态工具
- [[Linux-IP转发与路由|Linux IP 转发与路由]] — IPVS NAT 模式依赖 IP 转发

## 参考资料

- `man ipvsadm` — 本地 man page
- [LVS 官方文档](http://www.linuxvirtualserver.org/)
- [Keepalived 官网](https://www.keepalived.org/)
