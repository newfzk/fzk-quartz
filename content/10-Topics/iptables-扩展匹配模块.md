---
tags:
  - topic/iptables
  - topic/计算机网络
  - topic/Linux
status: to-review
---

# iptables 扩展匹配模块

iptables 标准五元组（协议、入接口、出接口、源地址、目标地址）之外的扩展匹配能力。

## ADDRTYPE — 地址类型匹配

检查数据包地址的类型，无需知道具体 IP。

**语法**：`ADDRTYPE match <src|dst>-type <类型>`

常见于 K8s 规则中匹配目标是否为本机地址：

```shell
KUBE-NODE-PORT  all  --  *  *  0.0.0.0/0  0.0.0.0/0  ADDRTYPE match dst-type LOCAL
```

**`LOCAL` 类型**：目标 IP 是某块网卡上配置的地址。

> 为什么不用直接匹配 IP？因为节点 IP 在不同机器上不一样，`ADDRTYPE` 是通用的、"无论本机 IP 是什么都匹配"的方式。

**对比直觉**：`ADDRTYPE match dst-type LOCAL` = "这个包裹的收件地址是不是我家？"

## match-set — ipset 匹配

使用 ipset（Linux 内核 IP/端口集合数据结构）进行高效匹配，比逐条 iptables 规则性能更好。

**语法**：`match-set <set-name> <src|dst>,<src|dst>`

常见于 K8s 中匹配 ClusterIP:Port：

```shell
KUBE-MARK-MASQ  all  --  *  *  !172.20.0.0/16  0.0.0.0/0  match-set KUBE-CLUSTER-IP dst,dst
ACCEPT          all  --  *  *  0.0.0.0/0        0.0.0.0/0  match-set KUBE-CLUSTER-IP dst,dst
```

**`dst,dst` 含义**：两个字段用逗号分隔，格式 `src|dst,src|dst`：
- 第一个 `dst` — ipset 中的目的 IP 匹配数据包的目标 IP
- 第二个 `dst` — ipset 中的目的端口匹配数据包的目标端口

**对比直觉**：`match-set KUBE-CLUSTER-IP dst,dst` = "这个包裹的收件地址是不是在公司的通讯录里？"

## 对比总结

| 语法 | 模块 | 作用 | 匹配对象 |
|:---|:---|:---|:---|
| `ADDRTYPE match dst-type LOCAL` | `addrtype` | 目标地址是否为本机网卡 IP | 抽象地址类型，无需知道具体 IP |
| `match-set KUBE-CLUSTER-IP dst,dst` | `set` (ipset) | 目标 IP:Port 是否在 Service 列表里 | 具体的 ipset 集合条目 |

## 相关笔记

- [[iptables详解]]
- [[K8s-NodePort流量流转]]
- [[netfilter框架详解]]
