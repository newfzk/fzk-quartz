---
tags:
  - topic/K8s
  - topic/计算机网络
  - topic/iptables
  - topic/IPVS
status: to-review
---

# K8s NodePort 流量流转（IPVS 模式）

IPVS 模式下，K8s NodePort 的外部请求经过以下完整路径：

## 数据包流转路径

```
客户端 → 节点IP:NodePort
    ↓
① PREROUTING（nat 表） → KUBE-SERVICES
    ↓
② KUBE-SERVICES → KUBE-NODE-PORT（匹配 dst-type LOCAL）
    ↓
③ KUBE-NODE-PORT → KUBE-MARK-MASQ（ipset 匹配 NodePort 端口）
    ↓
   MARK set 0x4000（打 SNAT 标记）
    ↓
④ INPUT 链 → IPVS 截获 → DNAT（节点IP:Port → PodIP:80）
    ↓
⑤ POSTROUTING → KUBE-POSTROUTING → SNAT（源IP替换为节点IP）
    ↓
   到达 Pod
```

## 各阶段详解

### ① PREROUTING 链

所有入站包先进入 `PREROUTING`，跳转到 `KUBE-SERVICES` 自定义链。

实际环境中 PREROUTING 可能包含三个并列链：
| 链 | 来源 | 匹配条件 |
|---|---|---|
| `cali-PREROUTING` | Calico CNI | 所有流量（执行网络策略） |
| `KUBE-SERVICES` | kube-proxy | 所有流量（服务调度枢纽） |
| `DOCKER` | Docker daemon | `ADDRTYPE match dst-type LOCAL`（仅本机目标） |

`cali-PREROUTING` 和 `KUBE-SERVICES` 的包计数完全一致，说明两者串联执行且匹配所有流量。

### ② KUBE-SERVICES → KUBE-NODE-PORT

```shell
KUBE-NODE-PORT  all  --  *  *  0.0.0.0/0  0.0.0.0/0  ADDRTYPE match dst-type LOCAL
```

`ADDRTYPE match dst-type LOCAL` 匹配目标为本机 IP 的流量。外部请求目标正是节点 IP，因此匹配进入 `KUBE-NODE-PORT`。

### ③ KUBE-NODE-PORT → MARK 标记

通过 `ipset` 匹配 NodePort 端口，打上 `0x4000` 标记供后续 SNAT 使用。

```shell
# KUBE-NODE-PORT 链
KUBE-MARK-MASQ  all  --  *  *  0.0.0.0/0  0.0.0.0/0  match-set KUBE-NODE-PORT-TCP dst

# KUBE-MARK-MASQ 链
MARK       all  --  *  *  0.0.0.0/0  0.0.0.0/0  MARK set 0x4000
```

`KUBE-NODE-PORT-TCP` ipset 中存储所有 NodePort 端口号。

### ④ IPVS 执行 DNAT

核心步骤：IPVS 在 INPUT 链中截获数据包，执行 DNAT。

```shell
# ipvsadm -ln 输出
TCP  10.68.203.185:80 rr          # ClusterIP
  -> 10.244.1.5:80                Masq    1      0          0
TCP  192.168.1.10:30320 rr        # NodePort
  -> 10.244.1.5:80                Masq    1      0          0
```

**将 `节点IP:30320` 改写为 `PodIP:80`**，完成 DNAT。

### ⑤ POSTROUTING → SNAT

`KUBE-POSTROUTING` 链检查 `0x4000` 标记，执行 SNAT 将源 IP 替换为节点 IP，保证回包正确路由回客户端。

## 关键调试命令

| 阶段 | 命令 |
|:---|:---|
| PREROUTING | `iptables -t nat -L PREROUTING -n -v` |
| KUBE-SERVICES | `iptables -t nat -L KUBE-SERVICES -n -v` |
| KUBE-NODE-PORT | `iptables -t nat -L KUBE-NODE-PORT -n -v` |
| ipset 端口集合 | `ipset list KUBE-NODE-PORT-TCP` |
| IPVS 规则 | `ipvsadm -ln` |
| POSTROUTING | `iptables -t nat -L POSTROUTING -n -v` |
| 连接跟踪 | `conntrack -L \| grep <NodePort>` |

## 相关笔记

- [[iptables详解]]
- [[iptables-扩展匹配模块]]
- [[IPVS-IP虚拟服务器详解]]
- [[Linux-连接跟踪-conntrack详解]]
