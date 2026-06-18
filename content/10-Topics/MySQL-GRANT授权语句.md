---
title: MySQL GRANT 授权语句
date: 2026-06-03
aliases:
  - MySQL 授权
  - GRANT 命令
  - MySQL 权限管理
related:
  - "[[Mysql常用配置]]"
tags:
  - language/sql
  - topic/MySQL
  - topic/安全
status: to-review
---

# MySQL GRANT 授权语句

## 基本语法

```sql
-- 允许 admin 从任意主机连接
GRANT ALL PRIVILEGES ON rs10_v2.* TO 'admin'@'%';

-- 如果只需要本地连接
GRANT ALL PRIVILEGES ON rs10_v2.* TO 'admin'@'localhost';

-- 如果用户不存在，需要先创建用户（MySQL 8.0+）
CREATE USER IF NOT EXISTS 'admin'@'%' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON rs10_v2.* TO 'admin'@'%';

-- 最后刷新权限
FLUSH PRIVILEGES;
```

## 参数详解

| 部分 | 含义 |
|------|------|
| `ALL PRIVILEGES` | 授予所有权限（SELECT, INSERT, UPDATE, DELETE, CREATE, DROP 等） |
| `rs10_v2.*` | 表示 rs10_v2 库下的所有表 |
| `'admin'@'%'` | `%` 表示允许从任意 IP 连接；`localhost` 仅限本机 |

## 同时允许本地和远程连接

```sql
GRANT ALL PRIVILEGES ON rs10_v2.* TO 'admin'@'localhost';
GRANT ALL PRIVILEGES ON rs10_v2.* TO 'admin'@'%';
FLUSH PRIVILEGES;
```

## 按需授予部分权限

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON rs10_v2.* TO 'admin'@'%';
```

## 常用权限列表

| 权限 | 说明 |
|------|------|
| `SELECT` | 查询数据 |
| `INSERT` | 插入数据 |
| `UPDATE` | 更新数据 |
| `DELETE` | 删除数据 |
| `CREATE` | 创建数据库/表 |
| `DROP` | 删除数据库/表 |
| `ALTER` | 修改表结构 |
| `INDEX` | 创建/删除索引 |
| `CREATE VIEW` | 创建视图 |
| `SHOW VIEW` | 查看视图定义 |
| `CREATE USER` | 创建用户 |
| `RELOAD` | 执行 FLUSH 操作 |
| `SUPER` | 超级权限（MySQL 8.0 已拆分） |

## 查看权限

```sql
-- 查看当前用户权限
SHOW GRANTS;

-- 查看指定用户权限
SHOW GRANTS FOR 'admin'@'%';
```

## 撤销权限

```sql
REVOKE ALL PRIVILEGES ON rs10_v2.* FROM 'admin'@'%';
REVOKE SELECT, INSERT ON rs10_v2.* FROM 'admin'@'%';
FLUSH PRIVILEGES;
```

## 相关笔记

- [[Mysql常用配置]]
- [[MySQL Binlog 日志配置]]