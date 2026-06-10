---
title: MySQL 8 — allowPublicKeyRetrieval 连接参数
date: 2026-06-08
tags:
  - topic/MySQL
  - topic/故障排查
  - language/sql
status: reviewed
aliases:
  - Public Key Retrieval is not allowed
  - MySQL caching_sha2_password
  - JDBC allowPublicKeyRetrieval
related:
  - "[[Mysql常用配置]]"
---

# MySQL 8 — allowPublicKeyRetrieval 连接参数

## 问题

应用连接 MySQL 时报错：

```
java.sql.SQLNonTransientConnectionException: Public Key Retrieval is not allowed
```

## 原因

MySQL 8.0+ 默认使用 `caching_sha2_password` 认证插件。当 JDBC 连接配置了 `useSSL=false` 时，驱动无法获取公钥来完成密码加密，除非显式启用 `allowPublicKeyRetrieval=true`。

## 修复

在 JDBC URL 中添加参数：

```
# 修复前
jdbc:mysql://127.0.0.1:3306/db?useSSL=false

# 修复后
jdbc:mysql://127.0.0.1:3306/db?useSSL=false&allowPublicKeyRetrieval=true
```

## 注意事项

- `allowPublicKeyRetrieval=true` 允许客户端从服务器获取公钥，在非 SSL 连接中存在**中间人攻击风险**
- 生产环境建议启用 SSL（`useSSL=true`）而不是使用此参数
- 如果使用 SSL，则不需要配置此参数，公钥通过安全通道传输

## 参考链接

- [[Mysql常用配置]] — MySQL 常用配置参数