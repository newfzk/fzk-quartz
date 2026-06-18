---
title: MySQL 配置文件体系
date: 2026-06-11
aliases:
  - MySQL my.cnf 结构
  - MySQL 配置加载顺序
  - MySQL config groups
  - MySQL 配置目录
tags:
  - language/sql
  - topic/MySQL
  - topic/数据库/配置
status: to-review
---

## 核心概念

MySQL 的配置文件采用**分层目录结构 + 配置组隔离**的机制。了解文件加载顺序和配置组的作用范围，是正确配置 MySQL 的基础。

## 文件布局（Ubuntu/Debian 系）

```
/etc/mysql/
├── my.cnf                     # 主入口文件（所有 MySQL 程序读取）
├── conf.d/                    # 【通用】对所有 MySQL 程序生效
│   ├── mysql.cnf              # 客户端配置（[client] 组）
│   └── mysqldump.cnf          # mysqldump 配置（[mysqldump] 组）
└── mysql.conf.d/              # 【服务端】仅对 MySQL 服务器生效
    ├── mysql.cnf              # mysql CLI 配置（[mysql] 组）
    └── mysqld.cnf             # 服务器核心配置（[mysqld] 组）⭐
```

### 各文件作用

| 文件 | 适用程序 | 典型内容 |
|------|----------|----------|
| `/etc/mysql/my.cnf` | **所有** MySQL 程序 | 主入口，通过 `!includedir` 引入子目录 |
| `conf.d/mysql.cnf` | 客户端 + 服务端 | `[client]` 组默认参数（如 default-character-set） |
| `conf.d/mysqldump.cnf` | mysqldump | `[mysqldump]` 组参数（如 quick、max_allowed_packet） |
| `mysql.conf.d/mysql.cnf` | mysql CLI | `[mysql]` 组参数（如 prompt、auto-rehash） |
| `mysql.conf.d/mysqld.cnf` | **mysqld 服务端** | `[mysqld]` 组核心配置（port、datadir、bind-address、log-bin）⭐ |

## 加载机制与优先级

### 加载顺序

MySQL 通过 `!includedir` 指令按**字母顺序**加载配置目录中的 `.cnf` / `.ini` 文件：

```
my.cnf 读取顺序：
  1. /etc/mysql/my.cnf                      ← 最先读取，定义全局参数
  2. /etc/mysql/conf.d/                     ← 按字母序加载（mysql.cnf → mysqldump.cnf）
  3. /etc/mysql/mysql.conf.d/               ← 按字母序加载（mysql.cnf → mysqld.cnf）
  4. ~/.my.cnf                              ← 用户家目录配置（可选，优先级最高）
```

### 优先级规则

**同一配置组（section）内，后加载的文件覆盖先加载的相同参数。**

> 示例：若 `my.cnf` 中设了 `port=3306`，而 `mysql.conf.d/mysqld.cnf` 中设了 `port=3307`，则最终生效的是 `3307`。

## 配置组的隔离机制

MySQL 配置文件使用 `[group]` 标记来区分生效范围：

| 配置组 | 生效的程序 | 解释 |
|--------|-----------|------|
| `[client]` | 所有客户端程序（mysql、mysqldump 等） | 共享连接参数 |
| `[mysql]` | mysql CLI 客户端 | 仅命令行客户端 |
| `[mysqldump]` | mysqldump 备份工具 | 仅导出工具 |
| `[mysqld]` | **MySQL 服务器进程** | 核心服务参数 |
| `[mysqld_safe]` | mysqld_safe 启动脚本 | 日志、pid 文件等 |

## 实战意义

### 1. 配置排查时先定位正确的文件

`bind-address` 属于 `[mysqld]` 组，应去 `mysql.conf.d/mysqld.cnf` 查找，而非在客户端配置文件中浪费时间。

### 2. 追加配置时选择合适的位置

- **服务端参数**（bind-address、max_connections）→ 追加到 `mysql.conf.d/mysqld.cnf`
- **通用客户端参数**（default-character-set）→ 追加到 `conf.d/mysql.cnf`

### 3. 避免被覆盖

服务端专属配置**不要写在 `conf.d/` 目录下**，否则可能被 `mysql.conf.d/` 中的同名参数覆盖。

### 4. Ansible 模板策略

如果使用配置模板管理 MySQL，应针对 `mysql.conf.d/mysqld.cnf` 进行管理（而非 `my.cnf`），保持与发行版默认布局兼容。

## 验证当前生效配置

```bash
# 查看 mysqld 当前加载了哪些配置文件
mysqld --verbose --help | grep -A 1 "Default options"

# 查看所有生效的配置参数
mysql -u root -p -e "SHOW VARIABLES;"

# 查看特定参数
mysql -u root -p -e "SHOW VARIABLES LIKE 'bind_address';"
mysql -u root -p -e "SHOW VARIABLES LIKE 'port';"
```

## 参考链接

- [[MySQL-bind-address-配置详解]] — bind-address 与 Unix socket/TCP/IP 连接
- [[MySQL-X-Plugin与X协议]] — mysqlx-bind-address 与 X Plugin 配置
- [[Mysql常用配置]] — MySQL 常用配置参数汇总
