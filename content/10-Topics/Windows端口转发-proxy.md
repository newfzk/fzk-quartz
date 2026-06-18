---
title: Windows端口转发-proxy
date: 2026-06-02
updated: 2026-06-02 12:00:00
tags:
  - topic/计算机网络
  - topic/Windows
status: to-review
---

## 核心概念

**netsh interface portproxy** 是 Windows 内置的端口转发工具，工作于 **内核态（TCP/IP 协议栈层）**，可以在 IPv4 和 IPv6 之间建立连接中继。

与 Linux iptables DNAT 不同，portproxy 工作在应用层之下的传输层，不依赖额外的代理程序。

## 基本语法

```bash
# 添加转发规则
netsh interface portproxy add v4tov4 \
  listenport=<本地监听端口> \
  listenaddress=<本地绑定IP> \
  connectport=<目标端口> \
  connectaddress=<目标IP>

# 查看所有规则
netsh interface portproxy show all

# 删除规则
netsh interface portproxy delete v4tov4 \
  listenport=8080 listenaddress=172.21.0.1

# 清空所有规则
netsh interface portproxy reset
```

## 典型场景

### 场景1：手机访问 Docker 容器（跨网络桥接）

```
手机 → Windows(172.21.0.1:8080) → portproxy → VM(192.168.236.128:8080)
                                                 ↓ DNAT →
                                              容器(172.17.0.2:80)
```

```bash
netsh interface portproxy add v4tov4 \
  listenport=8080 listenaddress=172.21.0.1 \
  connectport=8080 connectaddress=192.168.236.128
```

### 场景2：IPv4 ↔ IPv6 桥接

```bash
# IPv6 服务通过 IPv4 访问
netsh interface portproxy add v4tov6 \
  listenport=443 listenaddress=0.0.0.0 \
  connectport=443 connectaddress=2001:db8::1
```

## 注意事项

| 问题 | 解决 |
|------|------|
| **防火墙拦截** | `netsh advfirewall firewall add rule name="xxx" dir=in action=allow protocol=TCP localport=8080` |
| **只支持 TCP** | portproxy 仅支持 TCP，不支持 UDP/ICMP |
| **绑定 IP 需存在** | listenaddress 必须在本地适配器上已配置 |
| **服务重启丢失** | 规则不持久化（重启后丢失），可编写 PowerShell 开机脚本 |
| **性能** | 内核态转发，性能优于应用层代理，但不及硬件路由 |

## 面试要点

- portproxy 是**内核态 TCP 中继**，不是应用层代理，开销更小
- 仅支持 **TCP 协议**，无法转发 UDP/ICMP
- 常用于 **跨网段桥接**：让一个网络中的设备通过 Windows 访问另一网络中的服务
- 与 Linux iptables DNAT 配合使用可实现**多层转发链路**
- 重启后规则丢失，生产环境建议用 `schtasks` 或 PowerShell profile 做持久化

## 参考链接

- [柠檬微趣-笔试-Q3-网络配置](../30-Secret-Questions/柠檬微趣-笔试-Q3-网络配置.md)
- [[iptables端口转发]]
- [[Linux-IP转发与路由]]
- [[Docker网络模式-bridge]]
