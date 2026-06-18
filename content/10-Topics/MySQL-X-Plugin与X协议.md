---
title: MySQL X Plugin 与 X 协议
date: 2026-06-11
aliases:
  - MySQL X Plugin
  - X Protocol
  - mysqlx-bind-address
  - X DevAPI
tags:
  - language/sql
  - topic/MySQL
  - topic/计算机网络
status: to-review
---

## 核心概念

**X Plugin** 是 MySQL 8.0 引入的一个插件，提供基于 **X Protocol** 的通信方式。它与传统 MySQL 协议（3306 端口）独立，默认监听 **33060** 端口。

## 两个协议与两个配置

| 配置项 | 控制对象 | 默认端口 | 用途 |
|--------|----------|----------|------|
| `bind-address` | **传统 MySQL 协议** | 3306 | 被 `mysql` 命令行、JDBC 等客户端使用 |
| `mysqlx-bind-address` | **X 协议** (X Plugin) | 33060 | 被 X DevAPI 客户端使用 |

> [!important] 关键区别
> `mysqlx-bind-address` 配置的是 **MySQL X Plugin** 的监听地址，与传统 MySQL 连接无关。修改 `bind-address` 并不会影响 X Plugin 的行为，反之亦然。

## X Plugin 能做什么

X Plugin 提供基于 **X Protocol** 的通信方式，支持：

### 1. 文档存储（Document Store）

可以直接用 JSON 文档操作，不需要写 SQL：

```javascript
// 通过 X Protocol 操作（不走 SQL）
collection.find("name = :name").bind("name", "张三").execute()
```

### 2. X DevAPI

面向 NoSQL 风格的 CRUD 操作，支持多种编程语言：
- MySQL Connector/Node.js
- MySQL Connector/Python
- MySQL Connector/Java
- MySQL Connector/.NET

### 3. MySQL Shell 的高级模式

MySQL Shell 的 JavaScript/Python 模式（非 SQL 模式）使用 X Protocol 连接。

## 什么时候会用到 X Plugin

| 场景 | 说明 |
|------|------|
| 使用 MySQL Shell 的 JS/Python 模式 | `mysqlsh` 默认尝试使用 X Protocol |
| 使用 Connector/Node.js 通过 X DevAPI | Node.js 应用直接 CRUD |
| 使用 MySQL Document Store | 面向文档的 NoSQL 操作 |
| 普通 JDBC 应用 | ❌ 不需要，走传统 3306 端口 |

## 配置示例

```ini
[mysqld]
# 传统协议监听所有接口
bind-address = 0.0.0.0

# X Protocol 只监听本地（默认行为）
# mysqlx-bind-address = 127.0.0.1

# X Protocol 监听所有接口
# mysqlx-bind-address = 0.0.0.0
```

## 排查注意事项

当遇到 MySQL 连接问题时，首先要区分使用的是哪个协议：

```bash
# 查看传统协议端口监听
ss -tlnp | grep 3306

# 查看 X Protocol 端口监听
ss -tlnp | grep 33060

# 查看 X Plugin 是否启用
mysql -u root -p -e "SHOW PLUGINS;" | grep xplugin
# 或
mysql -u root -p -e "SHOW VARIABLES LIKE 'mysqlx%';"
```

## 参考链接

- [[MySQL-bind-address-配置详解]] — 传统 bind-address 配置详解（排查连接问题的核心）
- [[MySQL-配置文件体系]] — MySQL 配置文件目录结构与加载顺序
