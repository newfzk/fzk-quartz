---
title: MySQL 查询用户信息（mysql.user 表）
date: 2026-06-11
aliases:
  - mysql.user 表详解
  - MySQL 查询用户
  - authentication_string
related:
  - "[[MySQL-GRANT授权语句]]"
tags:
  - language/sql
  - topic/MySQL
  - topic/安全
status: to-review
---

# MySQL 查询用户信息（mysql.user 表）

`mysql.user` 是 MySQL 系统数据库中的核心权限表，存储所有用户账户信息。通过查询该表可以查看当前 MySQL 实例中的用户列表、登录主机限制、密码认证信息等。

## 基本查询

```sql
-- 查看所有用户、允许登录的主机及密码哈希
SELECT user, host, authentication_string FROM mysql.user;
```

输出示例：

| user | host | authentication_string |
|------|------|----------------------|
| root | localhost | *6C387FC9C5F... |
| mysql.sys | localhost | *THISISNOTAVALIDPASSWORD... |
| admin | % | $A$005$?[caching_sha2_hash]... |

## 核心字段详解

### user — 用户名

- MySQL 账户的登录名
- 同一个用户名可对应多个 `host` 条目（视为不同账户）

### host — 允许连接的主机

| 值 | 含义 |
|----|------|
| `localhost` | 仅允许本地 Socket 连接（Unix socket / Windows named pipe） |
| `127.0.0.1` | 仅允许本地 TCP 回环连接 |
| `%` | 允许任意远程主机连接（**不包含** localhost） |
| `192.168.1.%` | 允许特定网段连接 |
| `::1` | IPv6 本地地址 |

> [!tip] host 匹配规则
> MySQL 使用**精确匹配优先**策略：`localhost` > `192.168.1.%` > `%`。当多个条目匹配时，选择匹配度最高的。所以 `'admin'@'localhost'` 和 `'admin'@'%'` 是**两个不同的账户**，拥有独立的权限和密码。

### authentication_string — 密码认证信息

- MySQL 5.7+ 中替代旧的 `password` 列，存储经过哈希处理的密码
- 不同认证插件产生不同格式的哈希值：

| 插件 | 格式特征 | 示例 |
|------|---------|------|
| `mysql_native_password` | 以 `*` 开头，41 字符 | `*6C387FC9C5F...` |
| `caching_sha2_password` | 以 `$A$005$` 开头 | `$A$005$?[hash]...` |
| `sha256_password` | 以 `$5$` 或 `$6$` 开头 | `$5$[hash]...` |

> [!warning] 密码哈希不可逆向
> `authentication_string` 存储的是**哈希值而非明文密码**，无法通过此字段还原原始密码。这是基本的安全设计。

## 扩展查询

```sql
-- 查看完整用户信息（含认证插件、账户锁定状态）
SELECT user, host, plugin, authentication_string, account_locked
FROM mysql.user;

-- 查看用户的全局权限
SELECT user, host, Select_priv, Insert_priv, Update_priv, Delete_priv,
       Create_priv, Drop_priv, Super_priv
FROM mysql.user;

-- 查看各认证插件的使用分布
SELECT plugin, COUNT(*) AS count
FROM mysql.user
GROUP BY plugin;

-- 查找密码为空的危险账户
SELECT user, host FROM mysql.user
WHERE authentication_string = '' OR authentication_string IS NULL;

-- 查找拥有远程访问权限的账户
SELECT user, host FROM mysql.user
WHERE host = '%' OR host LIKE '192.168.%';
```

## 其他重要字段

| 字段 | 说明 |
|------|------|
| `plugin` | 认证插件（MySQL 8.0 默认 `caching_sha2_password`） |
| `account_locked` | 账户是否锁定（MySQL 8.0+，`Y`/`N`） |
| `password_expired` | 密码是否已过期（`Y`/`N`） |
| `password_last_changed` | 密码最后修改时间 |
| `password_lifetime` | 密码有效期（天） |
| `Select_priv`, `Insert_priv`... | 全局权限标记（`Y`/`N`） |
| `max_questions`, `max_updates`... | 资源限制（每小时查询/更新次数） |

## MySQL 5.7 vs 8.0 差异

| 特性 | MySQL 5.7 | MySQL 8.0 |
|------|-----------|-----------|
| 默认认证插件 | `mysql_native_password` | `caching_sha2_password` |
| authentication_string 格式 | `*` 开头（native hash） | `$A$005$` 开头（SHA-256 缓存哈希） |
| `password` 列 | 已弃用但保留 | 已移除 |
| `account_locked` | 不支持 | 支持 |
| `password_expired` | 支持 | 支持（增强） |

> [!tip] 8.0 兼容旧客户端
> 如果客户端驱动不支持 `caching_sha2_password`，可以修改用户使用 `mysql_native_password` 插件：
> ```sql
> ALTER USER 'user'@'host' IDENTIFIED WITH mysql_native_password BY 'password';
> ```

## 与 GRANT 的关系

`mysql.user` 表是 MySQL 权限体系的基础：

1. 执行 `CREATE USER` → 在 `mysql.user` 中插入一条记录
2. 执行 `GRANT ALL PRIVILEGES` → 更新 `mysql.user` 中的全局权限字段
3. 执行 `DROP USER` → 从 `mysql.user` 中删除记录

> [!example] 实践建议
> 排查 MySQL 连接问题时，先查 `mysql.user` 确认用户是否存在、host 是否匹配、认证插件是否兼容：
> ```sql
> SELECT user, host, plugin, account_locked, password_expired
> FROM mysql.user
> WHERE user = 'your_user';
> ```

## 相关笔记

- [[MySQL-GRANT授权语句]] — 用户授权与权限管理
- [[Mysql常用配置]] — MySQL 基础配置
- [[MySQL-8-allowPublicKeyRetrieval]] — 认证插件相关连接问题
