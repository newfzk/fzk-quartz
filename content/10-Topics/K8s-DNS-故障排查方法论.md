---
title: K8s DNS 故障排查方法论
date: 2026-06-11
aliases:
  - Kubernetes DNS 排查思路
  - CoreDNS 故障排查
tags:
  - topic/Kubernetes
  - topic/计算机网络
  - topic/故障排查
  - topic/DNS
status: to-review
---

## 核心概念

K8s DNS 故障排查应遵循**分层排查**的思路：从 DNS 组件自身逐步向下检查到内核网络层，避免在错误层面浪费时间。

## 分层排查模型

```
DNS 故障
   │
   ├── 第1层: DNS 组件自身状态
   │   ├── CoreDNS Pod 是否 Running
   │   ├── CoreDNS 日志有无异常
   │   ├── kube-dns Service 与 Endpoints 是否就绪
   │   └── 确认 DNS 不是问题本身
   │
   ├── 第2层: kubelet DNS 配置
   │   ├── cluster-dns 是否正确指向 CoreDNS ClusterIP
   │   ├── cluster-domain 是否正确 (默认 cluster.local.)
   │   └── 容器内 resolv.conf 是否被正确注入
   │
   ├── 第3层: 容器网络连通性
   │   ├── CoreDNS ClusterIP 是否可达 (telnet/curl)
   │   ├── CoreDNS Pod IP 是否可达
   │   ├── 跨节点 Pod 通信是否正常
   │   └── 确认是容器网络问题还是 DNS 服务本身问题
   │
   └── 第4层: 内核网络规则
       ├── iptables/nftables 规则是否正确
       ├── kube-proxy DNAT 规则是否写入
       ├── Flannel/Calico 等 CNI 组件状态
       └── conntrack 表是否正常
```

> [!tip] 核心洞察
> **DNS 故障不一定是 DNS 本身的问题。** CoreDNS 可能正常运行，但底层的容器网络（kube-proxy DNAT + CNI overlay）出了问题同样会导致 DNS 超时。

## 各层排查要点

### 第1层：CoreDNS 自身状态

```bash
# Pod 状态
kubectl get pod -n kube-system -o wide | grep dns

# 日志
kubectl logs -n kube-system -l k8s-app=kube-dns

# Service 与 Endpoints
kubectl get svc -n kube-system kube-dns
kubectl get endpoints -n kube-system kube-dns

# 资源限制（是否被 OOM Kill）
kubectl describe pod -n kube-system -l k8s-app=kube-dns | grep -A5 "Last State"
```

### 第2层：kubelet 配置

```bash
# 查看 kubelet 启动参数中的 cluster-dns
ps aux | grep kubelet | grep cluster-dns

# 或查看配置文件
cat /var/lib/kubelet/config.yaml | grep -A5 clusterDNS

# 容器内 resolv.conf
kubectl exec <pod> -- cat /etc/resolv.conf
```

K8s 会在每个 Pod 中注入如下 DNS 配置：
```
nameserver <coreDNS-clusterIP>
search <namespace>.svc.<cluster-domain> svc.<cluster-domain> <cluster-domain>
options ndots:5
```

### 第3层：容器网络连通性

使用诊断 Pod 测试：
```bash
# 创建测试 Pod（建议 busybox 1.36+，避免 nslookup 已知 bug）
kubectl run -it --rm dns-test --image=busybox:1.36 --restart=Never -- sh

# 容器内测试
cat /etc/resolv.conf
nslookup kubernetes.default.svc.cluster.local 10.68.0.2
ping 172.20.0.18  # CoreDNS Pod IP
```

### 第4层：内核网络规则

```bash
# 查看 iptables NAT 规则（确认 DNAT 是否存在）
iptables -t nat -S | grep <coreDNS-clusterIP>

# 查看 iptables 版本（判断是 legacy 还是 nft 模式）
iptables --version
# iptables v1.8.7 (legacy)  ← legacy 模式
# iptables v1.8.7 (nft)     ← nft 模式

# 检查连接跟踪
conntrack -L | grep <coreDNS-clusterIP>

# 检查 IP 转发
cat /proc/sys/net/ipv4/ip_forward
```

## Ubuntu 22.04 常见陷阱

### 陷阱1：iptables 双后端冲突

Ubuntu 22.04 默认使用 **nftables** 内核 API（iptables-nft），但 kube-proxy 和 Flannel 等组件可能写入 **legacy** 表，导致规则写入与规则查询不在同一张表中。

详见 [[Linux-update-alternatives命令详解#三、与 iptables 的关系]]。

### 陷阱2：systemd-resolved 导致 CoreDNS 转发失败

Ubuntu 22.04 的 `/etc/resolv.conf` 指向 `127.0.0.53`（systemd-resolved stub），CoreDNS 的 `forward . /etc/resolv.conf` 会读到这个地址。但 CoreDNS 容器内无法访问宿主机的 loopback，导致上游 DNS 转发失败。

详见 [[Ubuntu-systemd-resolved-机制]]。

## 排查工具对比

| 工具 | 适用场景 | 注意事项 |
|------|---------|---------|
| `nslookup` | 快速测试 DNS 解析 | busybox 1.28 有 bug，建议 1.36+ |
| `dig` | 详细 DNS 诊断 | 需镜像中安装 dnsutils |
| `telnet` / `nc` | 测试端口连通性 | 通用网络诊断 |
| `tcpdump` | 抓包分析 DNS 流量 | 最终手段 |

## 参考链接

- [[K8s-Service-DNS域名解析规则]] — K8s Service DNS 域名格式与 CoreDNS 解析流程
- [[Linux-update-alternatives命令详解]] — iptables-legacy/nft 切换方法
- [[Ubuntu-systemd-resolved-机制]] — Ubuntu 22.04 systemd-resolved 机制详解
- [[iptables详解]] — iptables 四表五链与包流转
- [[netfilter框架详解]] — netfilter 内核框架
