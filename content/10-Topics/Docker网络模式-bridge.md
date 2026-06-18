---
title: Docker网络模式-bridge
date: 2026-06-02
updated: 2026-06-02 12:00:00
tags:
  - topic/Docker
  - topic/计算机网络
status: to-review
---

## 核心概念

Docker 的 **bridge 网络模式**（默认模式）使用 Linux 内核的网桥（bridge）功能，在宿主机上创建一个虚拟网桥 `docker0`，容器通过 `veth pair` 连接到该网桥上，形成私有子网。

### 网络拓扑

```
       宿主机（Docker Host）
    ┌──────────────────────────┐
    │    docker0 (bridge)      │
    │     172.17.0.1/16        │
    │   ┌─────┴──────┐         │
    │ veth0        veth1       │
    │   │            │         │
    │   ▼            ▼         │
    │ Ctr-A       Ctr-B        │
    │172.17.0.2  172.17.0.3    │
    └──────────────────────────┘
           │
      eth0/eth1 (物理/虚拟网卡)
           │
       外部网络
```

### 关键特性

| 特性 | 说明 |
|------|------|
| **自动 DNS** | 容器间可通过容器名通信（需 user-defined bridge） |
| **端口映射** | `-p 宿主机端口:容器端口` 自动添加 iptables DNAT 规则 |
| **隔离性** | 容器间默认可互通，但与宿主机外部隔离 |
| **出站 NAT** | Docker 自动添加 MASQUERADE 规则，容器可访问外网 |

## 通信路径

### 容器 → 外部网络
```
容器eth0 → docker0 → 宿主机路由 → eth1 → iptables MASQUERADE → 外部
```

### 外部 → 容器（需配置）
```
外部 → 宿主机IP:端口 → iptables PREROUTING DNAT → docker0 → 容器
```

## docker0 与用户自定义 bridge

| 对比项 | 默认 docker0 | 用户自定义 bridge |
|--------|-------------|-----------------|
| DNS 解析 | ❌ 需 `--link` | ✅ 自动 DNS |
| 隔离 | 所有容器互通 | 仅同一网络容器互通 |
| 自定义 | 使用默认子网 | 可指定 subnet/gateway |

```bash
# 创建用户自定义 bridge
docker network create --driver bridge --subnet 10.10.0.0/16 my-net

# 容器连接到自定义网络
docker network connect my-net container_name
```

## 面试要点

- 默认 bridge 模式下，**外部无法直接访问容器**，需端口映射或 iptables DNAT 转发
- Docker 自动管理 iptables 规则：`-p` 参数等价于手动添加 PREROUTING DNAT
- `--net=host` 模式跳过网络隔离，容器直接使用宿主机网络栈
- `--net=none` 模式容器无网络，适用于完全隔离场景

## 参考链接

- [柠檬微趣-笔试-Q3-网络配置](../30-Secret-Questions/柠檬微趣-笔试-Q3-网络配置.md)
- [[iptables端口转发]]
- [[Linux-IP转发与路由]]
