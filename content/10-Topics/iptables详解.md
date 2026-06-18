---
title: iptables详解
date: 2026-06-03
updated: 2026-06-03
tags:
  - topic/Linux
  - topic/计算机网络
status: to-review
---

## 概述

**iptables** 是 Linux 内核 **netfilter** 框架的用户空间命令行工具，用于配置包过滤（防火墙）、网络地址转换（NAT）、包修改等网络层功能。

> iptables 是**用户空间命令**，真正干活的是内核中的 **netfilter 框架**。之所以叫 iptables，是因为 IPv4 用 `iptables`，IPv6 用 `ip6tables`。

---

## 一、核心架构：五表五链

iptables 的组织方式是 **"表（Table）包含链（Chain），链包含规则（Rule）"**。

### 1.1 五张表（Tables）

| 表            | 功能                | 优先级   | 典型用途                              |
| ------------ | ----------------- | ----- | --------------------------------- |
| **raw**      | 绕过连接跟踪（conntrack） | 1（最高） | 不对某些流量做状态追踪，提升性能                  |
| **mangle**   | 修改数据包头部字段         | 2     | 修改 TTL、TOS、Mark 标记                |
| **nat**      | 网络地址转换            | 3     | DNAT（端口转发）、SNAT/MASQUERADE（源地址转换） |
| **filter**   | 包过滤（防火墙）          | 4     | 允许/拒绝流量（ACCEPT/DROP/REJECT）       |
| **security** | SELinux 安全标记      | 5（最低） | 基于强制访问控制的标记（较少用）                  |

> [!info] "四表五链" → "五表五链"
> 早期 iptables 只有 raw、mangle、nat、filter 四张表，所以传统上称"四表五链"。**security 表** 是 Linux 2.6.37+（2011年）才加入的。老教材和面试题仍会沿用"四表五链"这个说法，知道这个渊源即可。

### 1.2 表名释义（为什么叫这个名）

理解英文原意有助于记忆每个表的职责：

| 表名 | 英文原意 | 命名隐喻 | 一句话助记 |
|------|---------|---------|-----------|
| **raw** | **生的、未加工的**（raw material） | 让数据包保持 **"原始状态"** 通过，不对它做 conntrack 登记加工 | **"别碰我，保持 raw"** |
| **mangle** | **碾压/撕碎/弄变形**（如 `clothes got mangled` 衣服被绞烂） | 你要对数据包 **"动手动脚"** 改头部字段，就像在蹂躏它 | **"对包动手动脚 → mangle"** |
| **nat** | **N**etwork **A**ddress **T**ranslation 缩写 | 纯技术缩写，没有隐喻 | **"NAT 就是改地址"** |
| **filter** | **过滤器** | 只做 **"过/不过"** 的筛选，不改内容 | **"filter 只管拦，不改"** |
| **security** | **安全** | SELinux **安全标记** | **"security 盖安全章"** |

> **关键区分**：filter 只决定"让不让过"（不修改包），mangle 是真的**修改包的内容**，所以用了 mangle（蹂躏）这个强烈的词。

> [!tip] 记忆技巧
> **"raw 不管，mangle 改，nat 转，filter 拦，security 标签"** — 按处理顺序记忆。
> 字母顺序反过来更容易记：**security → filter → nat → mangle → raw**
> 按"对包动手程度"排序：**raw(最轻/不动) → filter(筛选) → nat(改地址) → mangle(改内容) → security(盖章)**

### 1.3 五条内置链（Chains）

每个"钩子点"是一条链，数据包在特定时机经过特定链：

| 链 | 触发时机 | 方向 |
|----|---------|------|
| **PREROUTING** | 数据包进入协议栈后，**路由决策前** | 入站 |
| **INPUT** | 路由决策后，**目的地是本机** | 入站→本机 |
| **FORWARD** | 路由决策后，**目的地不是本机**（需转发） | 经过 |
| **OUTPUT** | **本机进程发出的包**，路由决策前 | 出站 |
| **POSTROUTING** | 路由决策后，**即将发送到网卡前** | 出站 |

### 1.4 表与链的交叉关系

不是每个表都有所有链，下表展示哪些表在哪些链上注册了钩子（✅ 表示注册）：

| 表 \ 链 | PREROUTING | INPUT | FORWARD | OUTPUT | POSTROUTING |
|---------|:----------:|:-----:|:-------:|:------:|:-----------:|
| **raw** | ✅ | | | ✅ | |
| **mangle** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **nat** (DNAT) | ✅ | | | ✅ | |
| **nat** (SNAT) | | | | | ✅ |
| **filter** | | ✅ | ✅ | ✅ | |
| **security** | | ✅ | ✅ | ✅ | |

> [!tip] 关键记忆
> - **PREROUTING** 只有 `raw`、`mangle`、`nat(DNAT)` 三张表
> - **POSTROUTING** 只有 `mangle`、`nat(SNAT)` 两张表
> - **INPUT/FORWARD** 有 `mangle`、`filter`、`security` 三张表，**没有 nat**
> - **OUTPUT** 最复杂，经过了所有五张表

---

## 二、数据包完整流转路径

这是理解 iptables 最关键的一张图：

```
                      ┌─────────────────────────────────────────┐
                      │             数据包到达网卡               │
                      └─────────────────────────────────────────┘
                                        │
                                        ▼
                      ┌─────────────────────────────────────────┐
                      │  PREROUTING                             │
                      │    raw (raw表的PREROUTING链)             │
                      │    mangle (mangle表的PREROUTING链)       │
                      │    nat   (nat表的PREROUTING链 - DNAT)    │
                      └─────────────────────────────────────────┘
                                        │
                                        ▼
                               ╔══════════════╗
                               ║  路由决策     ║
                               ║  查路由表    ║
                               ╚══════════════╝
                                        │
                          ┌─────────────┴─────────────┐
                          ▼                           ▼
              ┌─────────────────────┐      ┌─────────────────────┐
              │ 目的地是本机         │      │ 目的地不是本机      │
              │                     │      │ （需要转发）        │
              │  INPUT              │      │  FORWARD            │
              │    mangle           │      │    mangle           │
              │    filter           │      │    filter           │
              │    security         │      │    security         │
              └─────────────────────┘      └─────────────────────┘
                          │                           │
                          ▼                           ▼
              ┌─────────────────────┐      ┌─────────────────────┐
              │    本地进程          │      │  POSTROUTING        │
              │    (应用程序)        │      │    mangle           │
              └─────────────────────┘      │    nat (SNAT)       │
                          │               └─────────────────────┘
                          ▼                           │
              ┌─────────────────────┐                  │
              │  OUTPUT             │                  │
              │    raw              │                  │
              │    mangle           │                  │
              │    nat (DNAT)       │                  │
              │    filter           │                  │
              │    security         │                  │
              └─────────────────────┘                  │
                          │                           │
                          ▼                           │
              ┌────────────────────────────────────────┘
              │
              ▼
    ┌─────────────────────────┐
    │  POSTROUTING            │
    │    mangle               │
    │    nat (SNAT)           │
    └─────────────────────────┘
              │
              ▼
       从网卡发送出去
```

### 简化的三种包路径

```bash
# 情况1：发给本机的包
网卡 → PREROUTING → 路由决策(目标=本机) → INPUT → 本机进程

# 情况2：经过本机转发的包（本机充当路由器）
网卡 → PREROUTING → 路由决策(目标≠本机) → FORWARD → POSTROUTING → 出网卡

# 情况3：本机发出的包
本机进程 → OUTPUT → 路由决策 → POSTROUTING → 出网卡
```

---

## 三、规则（Rules）基础

每条规则包含两个部分：**匹配条件（match）** + **动作（target）**。

### 3.1 匹配条件

```bash
# 按接口匹配
-i eth0          # 入站接口
-o eth1          # 出站接口

# 按来源/目标 IP 匹配
-s 192.168.1.0/24   # 源IP
-d 10.0.0.1         # 目标IP

# 按协议匹配
-p tcp        # TCP 协议
-p udp        # UDP 协议
-p icmp       # ICMP 协议

# 按端口匹配（仅 TCP/UDP）
--sport 80          # 源端口
--dport 443         # 目标端口

# 多端口匹配
-m multiport --dports 80,443,8080

# 按状态匹配（需 conntrack）
-m state --state NEW,ESTABLISHED,RELATED,INVALID

# 按 MAC 地址匹配
-m mac --mac-source 00:11:22:33:44:55

# 按时间匹配
-m time --timestart 09:00 --timestop 18:00 --weekdays Mon,Tue,Wed,Thu,Fri

# 按速率限制匹配
-m limit --limit 10/second --limit-burst 20

# 按包长度匹配
-m length --length 100:500

# 否定匹配（大部分条件前加 !）
-s ! 192.168.1.0/24   # 源IP不是这个网段
```

### 3.2 目标（Targets / 动作）

| 目标                |       表       | 含义        | 说明                                  |
| ----------------- | :-----------: | --------- | ----------------------------------- |
| **ACCEPT**        |    filter     | 放行        | 允许通过，不再检查本链后续规则                     |
| **DROP**          |    filter     | 丢弃        | 直接丢弃，不回复（客户端会超时）                    |
| **REJECT**        |    filter     | 拒绝        | 丢弃并回复错误（TCP RST 或 ICMP unreachable） |
| **LOG**           | filter/mangle | 记录日志      | 记录到内核日志（dmesg），继续匹配后续规则             |
| **DNAT**          |      nat      | 修改目标IP/端口 | 通常用于 PREROUTING 链（入站端口转发）           |
| **SNAT**          |      nat      | 修改源IP     | 通常用于 POSTROUTING 链（固定IP）            |
| **MASQUERADE** 伪装 |      nat      | 动态 SNAT   | 自动取出口网卡 IP（适合 DHCP/PPPoE）           |
| **REDIRECT** 转向   |      nat      | 重定向到本机    | 将包目标改为本机某端口（透明代理）                   |
| **RETURN**        |      任意       | 返回上级链     | 从子链返回到父链的下一条规则                      |
| **MARK**          |    mangle     | 打标记       | 为包设置一个标记值，用于高级路由策略                  |

> [!tip] DROP vs REJECT
> - **DROP**：静默丢弃，客户端**卡住等待超时**（对攻击者更隐蔽）
> - **REJECT**：**明确拒绝**，客户端立即收到错误（排错时推荐）
> - 生产环境：对陌生入站用 DROP（安全），对自己人用的服务端口用 REJECT（排错方便）

---

## 四、连接跟踪（Connection Tracking / conntrack）

### 4.1 四大状态

iptables 通过 `-m state` 匹配数据包的连接状态，这些状态由内核的 **连接跟踪（conntrack）** 模块维护：

| 状态 | 含义 |
|------|------|
| **NEW** | 新建连接（第一个包） |
| **ESTABLISHED** | 已建立连接的后续包 |
| **RELATED** | 与已有连接相关的辅助连接（如 FTP 数据通道） |
| **INVALID** | 无法识别的包（通常丢弃） |

### 4.2 状态匹配示例

```bash
# 允许已建立连接的后续包回包（典型状态防火墙规则）
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 只允许 SSH 新建连接
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -j ACCEPT

# 丢弃无效包
iptables -A INPUT -m state --state INVALID -j DROP
```

### 4.3 在 raw 表中绕过 conntrack（NOTRACK）

对于高流量场景（如 DNS 服务器），可在 raw 表中使用 `NOTRACK` 目标绕过连接跟踪，减少 conntrack 开销：

```bash
iptables -t raw -A PREROUTING -p udp --dport 53 -j NOTRACK
iptables -t raw -A OUTPUT -p udp --dport 53 -j NOTRACK
```

> 完整原理、数据结构和调优方法请见 [[Linux-连接跟踪-conntrack详解]]。

---

## 五、规则管理命令

### 5.1 增删改查

```bash
# ---- 查看规则 ----
iptables -L                    # 列出 filter 表规则（默认）
iptables -t nat -L             # 列出 nat 表规则
iptables -L -n -v              # 不解析域名(-n)，显示统计(-v)
iptables -L --line-numbers     # 显示行号（方便删除/插入）
iptables -S                    # 以命令格式显示规则（可复用导出）

# ---- 追加/插入/替换规则 ----
iptables -A INPUT -p tcp --dport 22 -j ACCEPT              # 追加到末尾
iptables -I INPUT 1 -p tcp --dport 22 -j ACCEPT            # 插入到第1行
iptables -R INPUT 3 -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT  # 替换第3行

# ---- 删除规则 ----
iptables -D INPUT 3                        # 删除第3条规则
iptables -D INPUT -p tcp --dport 22 -j ACCEPT   # 删除匹配此规则的条目

# ---- 清空规则 ----
iptables -F                # 清空 filter 表所有规则
iptables -t nat -F         # 清空 nat 表所有规则
iptables -X                # 删除用户自定义链
iptables -Z                # 计数器归零
```

### 5.2 策略（默认策略）

```bash
# 设置链的默认策略（DROP 更安全，ACCEPT 更方便）
iptables -P INPUT DROP       # 默认拒绝所有入站
iptables -P FORWARD DROP     # 默认拒绝所有转发
iptables -P OUTPUT ACCEPT    # 默认允许所有出站

# 典型默认拒绝 + 白名单模式
iptables -P INPUT DROP
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT  # 允许回包
iptables -A INPUT -i lo -j ACCEPT                                   # 允许本机
iptables -A INPUT -p tcp --dport 22 -j ACCEPT                      # 允许SSH
```

### 5.3 持久化

```bash
# 保存当前规则到文件
iptables-save > /etc/iptables/rules.v4
ip6tables-save > /etc/iptables/rules.v6

# 从文件恢复规则
iptables-restore < /etc/iptables/rules.v4

# 安装持久化工具（Debian/Ubuntu）
apt install iptables-persistent
netfilter-persistent save
netfilter-persistent reload

# CentOS/RHEL/Fedora
yum install iptables-services
service iptables save
```

---

## 六、常用场景与完整示例

### 场景 1：基础防火墙 — 只允许 SSH 和 Web

```bash
# === 初始清理（小心！会断开SSH）===
iptables -F
iptables -X
iptables -Z

# === 默认策略 ===
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# === INPUT 链 ===
# 允许回包（已建立连接）
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 允许本机回环
iptables -A INPUT -i lo -j ACCEPT

# 允许 SSH（限制来源网段更安全）
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -j ACCEPT

# 允许 HTTP/HTTPS
iptables -A INPUT -p tcp -m multiport --dports 80,443 -m state --state NEW -j ACCEPT

# 允许 ICMP Ping（排错用，可选）
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# 丢弃无效包
iptables -A INPUT -m state --state INVALID -j DROP

# === 日志（丢包前记录，方便排错）===
iptables -A INPUT -j LOG --log-prefix "[IPTABLES-DROP] " --log-level 4
```

### 场景 2：端口转发 — 将宿主机端口转发到内网服务

```bash
# 拓扑：外网用户 → 公网IP:8080 → 转发到 内网服务器:80

# 1. 开启 IP 转发
echo 1 > /proc/sys/net/ipv4/ip_forward

# 2. DNAT：修改目标地址
iptables -t nat -A PREROUTING -d 公网IP -p tcp --dport 8080 \
  -j DNAT --to-destination 192.168.10.100:80

# 3. 允许 FORWARD 流量
iptables -A FORWARD -p tcp -d 192.168.10.100 --dport 80 -j ACCEPT

# 4. SNAT：让内网服务器知道回包路径（回包经由本机转发）
iptables -t nat -A POSTROUTING -s 192.168.10.100 -j MASQUERADE
```

> 对应知识：详细的 NAT 配置请见 [[iptables端口转发]]

### 场景 3：共享上网（NAT 路由器）

```bash
# 拓扑：Linux 双网卡，eth0 接外网，eth1 接内网
# 内网机器通过 Linux 上网

# 1. 开启 IP 转发
sysctl -w net.ipv4.ip_forward=1

# 2. 内网出站做 MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE

# 3. 允许 FORWARD
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT   # 内→外
iptables -A FORWARD -i eth0 -o eth1 -m state --state ESTABLISHED,RELATED -j ACCEPT  # 外→内（仅回包）

# 4. 可选：限制内网流量
iptables -A FORWARD -s 192.168.1.0/24 -p tcp --dport 80 -m limit --limit 1000/second -j ACCEPT
```

### 场景 4：透明代理（REDIRECT）

将经过本机的 HTTP 流量强制重定向到本地代理端口（如 Squid、mitmproxy）：

```bash
# 所有从内网来的 HTTP 请求，透明转发到本地 3128 代理端口
iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 80 \
  -j REDIRECT --to-port 3128

# 如果是本机发出的也可以
iptables -t nat -A OUTPUT -p tcp --dport 80 \
  -j REDIRECT --to-port 3128
```

### 场景 5：防端口扫描

```bash
# 限制 SSH 连接频率（每分钟最多4次新建连接，突发8次）
iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m recent --name SSH --set

iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m recent --name SSH --update --seconds 60 --hitcount 4 \
  -j DROP

# 限制 ICMP 洪水
iptables -A INPUT -p icmp --icmp-type echo-request \
  -m limit --limit 1/second --limit-burst 5 \
  -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
```

### 场景 6：负载均衡（statistic 模块）

将流量按比例分发到多台后端服务器：

```bash
# 50% 流量分到 192.168.1.10:80
iptables -t nat -A PREROUTING -p tcp --dport 80 \
  -m statistic --mode random --probability 0.5 \
  -j DNAT --to-destination 192.168.1.10:80

# 剩余 50% 流量分到 192.168.1.11:80
iptables -t nat -A PREROUTING -p tcp --dport 80 \
  -m statistic --mode random --probability 0.5 \
  -j DNAT --to-destination 192.168.1.11:80
```

### 场景 7：基于时间的访问控制

```bash
# 工作日 09:00-18:00 允许访问 Web 服务
iptables -A INPUT -p tcp --dport 80 \
  -m time --timestart 09:00 --timestop 18:00 \
  --weekdays Mon,Tue,Wed,Thu,Fri \
  -j ACCEPT

# 其他时间拒绝
iptables -A INPUT -p tcp --dport 80 -j DROP
```

### 场景 8：多端口转发（利用自定义链）

```bash
# 创建自定义链
iptables -t nat -N PORT_FORWARD

# 添加多条转发规则
iptables -t nat -A PORT_FORWARD -p tcp --dport 8080 -j DNAT --to-destination 10.0.0.2:80
iptables -t nat -A PORT_FORWARD -p tcp --dport 8443 -j DNAT --to-destination 10.0.0.2:443
iptables -t nat -A PORT_FORWARD -p tcp --dport 9090 -j DNAT --to-destination 10.0.0.3:9090

# 在 PREROUTING 中引用自定义链
iptables -t nat -A PREROUTING -i eth0 -j PORT_FORWARD
```

---

## 七、排错与调试

### 7.1 查看规则计数器

```bash
# -v 显示包计数和字节数（帮你看哪些规则被命中了）
iptables -L -n -v

# 清空计数器重新观察
iptables -Z
iptables -L -n -v
# 过一会儿再看，被命中次数多的规则会显现
iptables -L -n -v
```

### 7.2 临时放行所有流量（以防把自己锁在外面）

```bash
# 重置策略为 ACCEPT
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

# 清空所有规则
iptables -F
iptables -t nat -F
iptables -t mangle -F
```

### 7.3 跟踪数据包（TRACE 目标）

```bash
# 在 raw 表中添加 TRACE（会输出到 dmesg）
iptables -t raw -A PREROUTING -p tcp --dport 80 -j TRACE
iptables -t raw -A OUTPUT -p tcp --dport 80 -j TRACE

# 查看跟踪日志
dmesg -w | grep TRACE
```

### 7.4 常见排错场景

```
问题：配置了 DNAT 但访问不通

排查步骤：
1. 检查 ip_forward 是否开启
   cat /proc/sys/net/ipv4/ip_forward
   
2. 检查 DNAT 规则是否存在
   iptables -t nat -L PREROUTING -n -v
   
3. 检查 FORWARD 链是否放行
   iptables -L FORWARD -n -v
   
4. 检查目标主机是否可达
   ping 目标IP
   
5. 检查目标端口是否正确
   curl -v 目标IP:端口
   
6. 查看 conntrack 表确认连接状态
   conntrack -L | grep 目标IP
```

---

## 八、iptables vs nftables（未来趋势）

| 对比项 | iptables | nftables |
|--------|----------|----------|
| 内核框架 | 旧的 netfilter | 新的 nf_tables |
| 语法 | 复杂，表/链结构严格 | 更简洁，可读性好 |
| 性能 | 规则多时线性扫描较慢 | 使用 Set/Map 数据结构，性能更好 |
| 原子替换 | 需先清空再加载 | 支持原子替换（无窗口期） |
| 向后兼容 | - | `iptables` 命令调用 nftables 内核 |
| 趋势 | 长期维护，但不再发展 | 新一代，RHEL9/Debian12 默认 |

> 在 CentOS 8+/RHEL 9+/Debian 12+ 中，`iptables` 命令默认调用的是 nftables 内核模块（通过兼容层）。但命令语法和概念仍然相同。

**nftables 示例对比**：

```bash
# iptables 写法
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -j ACCEPT

# nftables 等价写法
nft add rule inet filter input tcp dport 22 ct state new accept
```

---

## 九、面试要点速记

1. **"五表五链"是最核心的框架** — 记住表的功能（raw/mangle/nat/filter/security）和链的流转顺序
2. **DNAT 在 PREROUTING 做，SNAT 在 POSTROUTING 做** — "改目标先于路由决策，改源在出站前最后一步"
3. **FORWARD 链是转发流量的"总闸门"** — 配了 DNAT 忘了放行 FORWARD 是最常见错误
4. **状态防火墙（stateful firewall）** — `-m state --state ESTABLISHED,RELATED -j ACCEPT` 是最常用的防御手段
5. **连接跟踪的代价** — 高流量场景 conntrack 表可能爆满（需要调大 `nf_conntrack_max` 或用 raw 表绕过），详见 [[Linux-连接跟踪-conntrack详解]]
6. **规则的顺序就是匹配顺序** — 先匹配到的规则生效（类似 ACL），重要规则放前面
7. **MASQUERADE vs SNAT** — MASQUERADE 用于动态 IP（自动取出口地址），SNAT 用于固定 IP（性能更好）

## 参考链接

- [[iptables端口转发]] — DNAT/SNAT/MASQUERADE 端口转发的具体配置
- [[Linux-IP转发与路由]] — 三层转发原理与路由配置
- [[netfilter框架详解]] — netfilter 内核框架详解（iptables 的底层基础）
- [[Linux-连接跟踪-conntrack详解]] — 连接跟踪完整原理与调优
- [[Docker网络模式-bridge]] — Docker bridge 网络与 iptables 的关系
