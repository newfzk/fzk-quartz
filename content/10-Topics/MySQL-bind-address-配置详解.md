---
title: MySQL bind-address 配置详解
date: 2026-06-11
aliases:
  - MySQL bind_address
  - MySQL 连接协议
  - bind-address 默认值
  - Unix socket vs TCP/IP MySQL
tags:
  - language/sql
  - topic/MySQL
  - topic/计算机网络
status: to-review
---

## 核心概念

`bind-address` 是 MySQL 用于控制**服务端监听哪个网络接口**的配置参数。它决定了 MySQL 是否接受来自远程主机的 TCP 连接。

## 连接协议：Unix Socket vs TCP/IP

MySQL 客户端连接 MySQL 服务器有两种方式，理解其区别是排查连接问题的关键：

| 连接方式 | 触发条件 | 协议栈 | 是否受 bind-address 影响 |
|---------|---------|--------|:----------------------:|
| **Unix socket** | 不带 `-h` 参数（或 `-h localhost`）| 本地进程间通信，不经过 TCP/IP | ❌ 不受影响 |
| **TCP/IP** | 带 `-h` 参数指定 IP 地址 | 经过 TCP/IP 网络栈 | ✅ 受 bind-address 限制 |

> [!important] 关键认知
> MySQL 客户端不带 `-h` 时使用 Unix socket 连接（仅限本机），带 `-h` 时使用 TCP/IP 连接，**两者是独立的连接通道**。手动测试时必须带上 `-h` 才能复现远程连接场景。

### Unix socket 连接路径

```bash
# 成功（通过 Unix socket）
mysql -uadmin -p -P3306

# 也成功（localhost 通常也走 socket）
mysql -h localhost -uadmin -p
```

### TCP/IP 连接（可能失败）

```bash
# 可能失败（通过 TCP/IP）
mysql -h 127.0.0.1 -uadmin -p
mysql -h <实际IP> -uadmin -p
```

## bind-address 默认值

`bind-address` 的默认值因 **MySQL 版本** 和 **发行版** 而异：

| 环境 | 默认值 | 行为 |
|------|--------|------|
| MySQL 8.0 (Ubuntu 22.04 默认安装) | `127.0.0.1` | 仅监听本地回环，拒绝远程 TCP 连接 |
| MySQL 5.7 (CentOS 7 默认) | `*` 或 `0.0.0.0` | 监听所有接口 |
| 手动编译安装 | `*` | 监听所有接口 |

> 在编写自动化部署模板时，**应显式指定 `bind-address`**，避免依赖隐式默认值导致跨环境行为不一致。

## 常见错误：ERROR 2003 (HY000) Connection refused

### 错误信息

```
ERROR 2003 (HY000): Can't connect to MySQL server on '<host>:3306' (111)
```

错误码 **111 (Connection refused)** 表示 TCP 连接被服务器拒绝。

### 排查步骤

```bash
# 1. 确认 MySQL 是否在监听
ss -tlnp | grep 3306
# 若只监听 127.0.0.1:3306 → bind-address 限制

# 2. 查看当前 bind-address 值
mysql -u root -p -e "SHOW VARIABLES LIKE 'bind_address';"

# 3. 对比测试连接方式
mysql -uadmin -p -P3306                          # Unix socket → 可能成功
mysql -h <实际IP> -uadmin -p -P3306               # TCP/IP → 可能失败
```

## 配置方法

### 临时修改（立即生效，重启失效）

```bash
# 无需重启，但重启后丢失
mysql -u root -p -e "SET GLOBAL bind_address = '0.0.0.0';"
```

> [!warning] 注意
> `bind_address` 是只读变量，**不支持在线修改**。上述命令会报错，必须修改配置文件后重启。

### 永久修改

```ini
[mysqld]
# 监听所有网络接口
bind-address = 0.0.0.0

# 或指定特定 IP
# bind-address = 192.168.1.100
```

配置文件位置（Ubuntu/Debian）：
- `/etc/mysql/mysql.conf.d/mysqld.cnf` — 主要配置

修改后重启：
```bash
systemctl restart mysql
```

### 验证修改

```bash
# 查看监听地址
ss -tlnp | grep 3306
# 应显示 0.0.0.0:3306（监听所有接口）

# 或用 SQL 查看（不会实时更新，需重启后才准确）
mysql -u root -p -e "SHOW VARIABLES LIKE 'bind_address';"
```

## 安全建议

| 场景 | 推荐值 | 说明 |
|------|--------|------|
| 开发环境 | `0.0.0.0` | 方便远程连接调试 |
| 生产环境 | 具体 IP 或 `127.0.0.1` | 只暴露必要的接口，减少攻击面 |
| Kubernetes | `0.0.0.0` | 需要从其他 Pod 访问 |

## 参考链接

- [[MySQL-配置文件体系]] — MySQL 配置文件目录结构与加载顺序
- [[MySQL-X-Plugin与X协议]] — mysqlx-bind-address 与传统 bind-address 的区别
- [[Mysql常用配置]] — MySQL 常用配置参数汇总
