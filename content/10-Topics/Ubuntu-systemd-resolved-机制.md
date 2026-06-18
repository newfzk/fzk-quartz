---
title: Ubuntu systemd-resolved 机制
date: 2026-06-11
aliases:
  - systemd-resolved
  - 127.0.0.53 stub
  - systemd-resolved stub resolver
  - resolved stub
tags:
  - topic/Linux
  - topic/计算机网络
  - topic/DNS
status: to-review
---

## 核心概念

**systemd-resolved** 是 systemd 提供的 DNS 解析服务，在 Ubuntu 22.04+ 中默认启用。它在 `127.0.0.53:53` 上监听作为 **stub（桩）解析器**，所有本地程序的 DNS 请求都先经过它，再转发到真正的上游 DNS 服务器。

## 工作原理

```
你的程序（nslookup/dig/curl 等）
    ↓ 查询 "google.com"
127.0.0.53:53  ← systemd-resolved 的 stub 监听地址
    ↓ 查询转发
systemd-resolved 根据自身配置（/etc/systemd/resolved.conf、netplan、DHCP 等）
    ↓ 转发查询
真正的上游 DNS 服务器（如 8.8.8.8、114.114.114.114 或内网 DNS）
```

## 为什么设计 stub 解析器

| 目的 | 说明 |
|------|------|
| **DNS 缓存** | 缓存 DNS 查询结果，减少重复查询，加速域名解析 |
| **多链路 DNS 管理** | VPN、有线、无线使用不同 DNS 时，systemd-resolved 自动路由到正确的上游 |
| **一致性接口** | 无论上游 DNS 如何变化，程序始终只需访问 `127.0.0.53` |

## /etc/resolv.conf 的变化

### systemd-resolved 启用时

```bash
$ cat /etc/resolv.conf
nameserver 127.0.0.53
options edns0 trust-ad
```

| 字段 | 含义 |
|------|------|
| `nameserver 127.0.0.53` | DNS 请求指向本机的 systemd-resolved stub |
| `options edns0` | 启用 EDNS0 扩展（支持更大的 DNS 报文） |
| `options trust-ad` | 信任 DNS 响应中的 AD（Authentic Data）标志位 |

### 查看真正的上游 DNS

```bash
# 方法1：查看 systemd-resolved 实际使用的 DNS
resolvectl status
# 输出中会显示每个网卡实际使用的 DNS 服务器

# 方法2：查看 systemd-resolved 的全局配置
cat /etc/systemd/resolved.conf

# 方法3：查看 netplan 配置（如果是 netplan 管理网络）
cat /etc/netplan/*.yaml

# 方法4：直接查看 resolved 维护的完整 resolv.conf（推荐）
cat /run/systemd/resolve/resolv.conf
# 这个文件包含的是真实的（real）上游 DNS 地址，不是 127.0.0.53
```

## 对容器/K8s 的影响

### 问题场景

在 **Docker 容器或 K8s CoreDNS** 中，如果程序读取了宿主机的 `/etc/resolv.conf` 指向 `127.0.0.53`，会因为容器无法访问宿主机 loopback 地址而导致 DNS 解析失败：

```
CoreDNS 容器内                    宿主机
┌──────────────┐               ┌──────────────────┐
│ forward .     │──→ 127.0.0.53  │  systemd-resolved  │
│ /etc/resolv.conf│  ✗ 连接失败   │  监听 127.0.0.53:53 │
└──────────────┘               └──────────────────┘
    ↑                            ↑
  容器内的 127.0.0.53          宿主机的 127.0.0.53
  （没有监听）                 （有监听但容器访问不到）
```

### 典型故障链

```
CoreDNS 配置 forward . /etc/resolv.conf
    │
    ├── 读取到 nameserver 127.0.0.53
    │
    ├── CoreDNS 尝试连接容器内 127.0.0.53:53 → 失败
    │
    └── 所有集群内 DNS 解析超时 → 服务发现异常
```

### 修复方法

在 CoreDNS ConfigMap 中直接指定真实上游 DNS，而非使用 `/etc/resolv.conf`：

```yaml
# CoreDNS ConfigMap 中的 forward 配置
forward . 8.8.8.8 114.114.114.114 {
    policy sequential
}
```

> 优先使用内网 DNS 地址（如公司内部 DNS），其次用公网 DNS 作为 fallback。

## 如何判断当前系统是否使用 systemd-resolved

```bash
# 方法1：查看 /etc/resolv.conf 内容
cat /etc/resolv.conf
# 如果 nameserver 是 127.0.0.53 → systemd-resolved 正在使用

# 方法2：查看 resolved 服务状态
systemctl status systemd-resolved

# 方法3：查看 /etc/resolv.conf 是哪个文件管理的
ls -la /etc/resolv.conf
# 如果是符号链接指向 ../run/systemd/resolve/stub-resolv.conf → systemd-resolved

# 方法4：查看 resolvectl 状态
resolvectl status
```

## 参考链接

- [[K8s-Service-DNS域名解析规则]] — K8s Service DNS 域名格式与搜索域机制
- [[K8s-DNS-故障排查方法论]] — K8s DNS 系统排查思路（含 systemd-resolved 陷阱）
- [[Linux-update-alternatives命令详解]] — iptables-legacy/nft 切换（Ubuntu 22.04 另一个常见陷阱）
- [[Docker-cgroup-v2-兼容性问题]] — Ubuntu 22.04 另一个常见兼容性问题
