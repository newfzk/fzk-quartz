---
title: K8s Service DNS 域名解析规则
date: 2026-06-11
aliases:
  - Kubernetes Service DNS
  - K8s DNS 解析
  - Service 域名
tags:
  - topic/Kubernetes
  - topic/计算机网络
  - topic/DNS
status: to-review
---

## 核心概念

Kubernetes 中，每个 Service 会被自动分配一个 **DNS 名称**，集群内的 Pod 可以通过该域名访问 Service，无需感知 Service 的 Cluster IP。这套机制由 **CoreDNS**（替代了早期的 kube-dns）作为集群内的 DNS 服务器实现。

## Service DNS 完整格式

### 标准模板

```
<service-name>.<namespace>.svc.<cluster-domain>
```

默认集群域为 `cluster.local`，因此完整 FQDN 为：

```
<service-name>.<namespace>.svc.cluster.local
```

### FQDN（Fully Qualified Domain Name）是什么

> **FQDN（完全限定域名）** 是指域名树中从根（`.`）开始的完整域名路径，能够唯一标识一台主机或服务，不以歧义方式解析。

#### 通用 FQDN 格式

通用格式：

```
<hostname>.<domain>.<tld>.
```

以 `api.example.com.` 为例：
- `api` — 主机名（Hostname）
- `example` — 二级域名
- `com` — 顶级域（TLD）
- `.` — 根域（Root Domain），**末尾的点（trailing dot）是 FQDN 的标志**

> [!tip] 末尾的「.」为什么重要？
> 在 DNS 解析中，**末尾的点表示绝对路径**。不带点的域名是相对域名，DNS 解析器会依次拼接搜索域（search domain）去尝试解析。末尾的点告诉解析器："就是它了，不要再拼接任何后缀"。例如：
> - `api.example.com.` — FQDN，绝对路径，直接查询
> - `api.example.com` — 相对域名，可能被拼接搜索域解析成 `api.example.com.default.svc.cluster.local`

#### 在 K8s 中的对应关系

| 通用 DNS 组件 | K8s Service DNS 示例 | 说明 |
|:-------------:|:--------------------:|:----:|
| 根域 `.` | `.` | FQDN 末尾隐含 |
| TLD | `local` | `.local` 是保留 TLD |
| 二级域 | `cluster` | 集群域前缀 |
| 子域 | `svc` | 标识 Service 资源类型 |
| 命名空间 | `default` | K8s 命名空间 |
| 主机名 | `my-svc` | Service 名称 |

### 各场景下的可用域名

| 访问位置 | 可用域名 | 示例 |
|:--------:|:---------|:----:|
| **同命名空间** | 仅 Service 名 | `redis` |
| **跨命名空间** | `svc名.命名空间名` | `redis.production` |
| **同命名空间（略 svc）** | `svc名.svc` | `redis.svc` |
| **完整 FQDN** | 全部 | `redis.production.svc.cluster.local` |

> [!example] 实际场景
> 假设有一个 Service `redis` 部署在命名空间 `production` 中：
> ```
> redis                                # production 命名空间内的 Pod 可用
> redis.production                     # default 命名空间的 Pod 可用
> redis.production.svc                 # 可省略 cluster.local
> redis.production.svc.cluster.local   # 完整 FQDN，任何位置可用
> ```

## CoreDNS 解析流程

### 架构图

```
┌─────────────────────────────────────────────────────────┐
│                     Pod（客户端）                          │
│  /etc/resolv.conf                                        │
│  search  <namespace>.svc.<cluster-domain>  svc.<cluster-domain>  <cluster-domain>  │
│  nameserver 10.96.0.10                                    │
└────────────────────────┬────────────────────────────────┘
                         │ DNS 查询: "redis.production"
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   CoreDNS (kube-dns)                      │
│  Service: kube-dns/kube-system                           │
│  ClusterIP: 10.96.0.10                                    │
│                                                          │
│  1. 收到查询 "redis.production"                           │
│  2. 根据搜索域拼接尝试:                                    │
│     - redis.production.default.svc.cluster.local ❌       │
│     - redis.production.svc.cluster.local ✅ → A记录       │
│  3. 返回 redis.production.svc.cluster.local 的 Cluster IP │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   目标 Service: redis                     │
│  ClusterIP: 10.96.0.50                                    │
│  端口: 6379                                              │
└─────────────────────────────────────────────────────────┘
```

> 上图展示了 Pod 发起 DNS 查询时，搜索域如何自动补全域名后缀，最终由 CoreDNS 返回目标 Service 的 Cluster IP。

### Pod resolv.conf 示例

每个 Pod 启动时，K8s 会注入如下 DNS 配置：

```
nameserver 10.96.0.10
search <pod-namespace>.svc.<cluster-domain> svc.<cluster-domain> <cluster-domain>
options ndots:5
```

其中：
- `nameserver` — CoreDNS Service 的 Cluster IP
- `search` — DNS 搜索域列表，查询相对域名时按顺序拼接
- `ndots:5` — 域名中「.」数量 ≥ 5 时视为 FQDN 直接查询，否则先拼接搜索域

> [!tip] ndots:5 的意义
> Service 的短域名（如 `redis`）只有 0 个点，ndots=5 使得 K8s 先拼接搜索域尝试，而不是直接作为 FQDN 查询。这保证了在同命名空间内用 `redis` 就能访问到正确的 Service。

## DNS 记录类型

CoreDNS 为每个 Service 创建以下 DNS 记录：

| 记录类型 | 用途 | 示例 |
|:--------:|:----|:----:|
| **A 记录** | 将域名解析为 IPv4 Cluster IP | `redis.production.svc.cluster.local. → 10.96.0.50` |
| **AAAA 记录** | IPv6 解析 | `redis.production.svc.cluster.local. → fd00::50` |
| **SRV 记录** | 命名端口（Named Port）解析 | `_redis._tcp.production.svc.cluster.local. → 0 50 6379 redis.production.svc.cluster.local.` |

### Headless Service（ClusterIP = None）

对于 headless Service，DNS 解析行为不同：
- **A 记录** — 返回所有就绪 Pod 的 IP 地址列表（而非 Service Cluster IP）
- 适用于 StatefulSet 等需要直接访问 Pod 的场景

```
# Headless Service: my-stateful-svc
# 查询返回所有就绪 Pod 的 IP
my-stateful-svc.default.svc.cluster.local. → 10.1.0.1, 10.1.0.2, 10.1.0.3
```

> [!info] Pod 也有 DNS 记录
> K8s 也会为 Pod（非 headless 服务）创建 DNS 记录，格式为：
> ```
> <pod-ip-with-dashes>.<namespace>.pod.<cluster-domain>
> ```
> 例如 Pod IP `10.1.0.1` → `10-1-0-1.default.pod.cluster.local`

## 配置 Custom DNS

可以通过 Pod 的 `dnsConfig` 自定义 DNS 解析行为：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsConfig:
    nameservers:
      - 8.8.8.8        # 额外的 DNS 服务器
    searches:
      - custom.svc.local  # 额外的搜索域
    options:
      - name: ndots
        value: "3"
  dnsPolicy: "None"    # None 表示完全自定义（默认是 ClusterFirst）
```

`dnsPolicy` 可选值：
- **Default** — 继承节点 DNS 配置
- **ClusterFirst** — 优先使用集群 DNS（默认值）
- **ClusterFirstWithHostNet** — hostNetwork 模式下使用集群 DNS
- **None** — 全部使用 dnsConfig 自定义

## 面试要点

- **K8s Service DNS 完整格式**：`<service>.<namespace>.svc.<cluster-domain>`，默认域 `cluster.local`
- **FQDN 末尾的「.」**：表示绝对路径，K8s 中 `ndots:5` 控制搜索域拼接行为
- **搜索域机制**：跨命名空间必须带命名空间名，同命名空间可省略
- **Headless Service**：没有 Cluster IP，DNS 返回所有就绪 Pod IP
- **CoreDNS**：作为集群 DNS，动态监听 Service 和 Endpoint 变化，自动更新 DNS 记录
- **dnsPolicy**：ClusterFirst vs Default vs None 的区别

## 相关笔记

- [[四层与七层负载均衡对比]] — K8s Service 四层与七层负载均衡
- [[Docker网络模式-bridge]] — 容器网络基础，容器间通信与 DNS
- [[Linux-IP转发与路由]] — Pod 跨节点通信依赖 IP 转发
