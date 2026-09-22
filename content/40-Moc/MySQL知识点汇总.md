---
title: MySQL 知识点汇总（MOC）
date: 2026-06-18
aliases:
  - MySQL 知识地图
  - MySQL MOC
  - MySQL 学习路线
tags:
  - topic/MySQL
  - topic/数据库
  - topic/MOC
  - language/sql
related:
  - "[[MySQL索引创建原则]]"
  - "[[MySQL索引类型]]"
  - "[[MySQL联合索引]]"
  - "[[Mysql常用配置]]"
  - "[[MySQL-配置文件体系]]"
  - "[[MySQL深度分页优化]]"
  - "[[MySQL-字符串存储日期vs专用日期类型]]"
  - "[[MySQL Binlog 日志配置]]"
  - "[[MySQL 通用查询日志配置]]"
  - "[[MySQL-GRANT授权语句]]"
  - "[[MySQL-查询用户信息-mysql.user表]]"
  - "[[MySQL-全文索引-FULLTEXT]]"
  - "[[MySQL-bind-address-配置详解]]"
  - "[[MySQL-8-allowPublicKeyRetrieval]]"
  - "[[MySQL-X-Plugin与X协议]]"
  - "[[Undo-Redo日志详解]]"
status: to-review
created: 2026-06-18
---

# MySQL 知识点汇总 — 知识地图

> [!abstract] 本笔记为 MOC（Map of Content）
> MySQL 知识体系梳理，涵盖索引、配置、权限、日志、数据类型、锁机制等核心主题。共收录 **16 条** 原子笔记。

## 索引与查询优化

| 笔记 | 核心内容 |
|------|---------|
| [[MySQL索引创建原则]] | 索引设计的最佳实践，选择性、覆盖索引、最左前缀原则 |
| [[MySQL索引类型]] | B+Tree、Hash、Full-Text、Spatial 索引原理对比 |
| [[MySQL联合索引]] | 联合索引的列顺序、索引下推（ICP）、失效场景 |
| [[MySQL 全文索引 — FULLTEXT]] | 全文索引的创建、查询模式与中文分词注意事项 |
| [[MySQL 深度分页优化]] | 延迟关联、游标分页、子查询优化大偏移 LIMIT |

## 配置与运维

| 笔记 | 核心内容 |
|------|---------|
| [[MySQL 常用配置]] | 常用 `my.cnf` 参数：连接数、缓存、字符集、时区 |
| [[MySQL 配置文件体系]] | 配置文件加载顺序、多文件合并机制 |
| [[MySQL bind-address 配置详解]] | 绑定地址与网络访问控制，安全配置 |
| [[MySQL 通用查询日志配置]] | 通用查询日志开启、文件位置与日志分析 |
| [[MySQL Binlog 日志配置]] | Binlog 格式（ROW/STATEMENT/MIXED）、过期策略、主从复制 |
| [[MySQL 8 — allowPublicKeyRetrieval]] | 连接参数说明，SHA2 认证插件兼容性 |

## 用户与安全管理

| 笔记 | 核心内容 |
|------|---------|
| [[MySQL GRANT 授权语句]] | 用户创建、权限授予、撤销与刷新 |
| [[MySQL 查询用户信息（mysql.user 表）]] | 用户表结构，认证插件与密码策略 |

## 数据类型与协议

| 笔记 | 核心内容 |
|------|---------|
| [[MySQL 字符串存储日期 vs 专用日期类型]] | DATE/DATETIME/TIMESTAMP 与 VARCHAR 存储对比 |
| [[MySQL X Plugin 与 X 协议]] | MySQL 8 的 X Protocol、文档存储、CRUD 操作 |

## 事务与锁机制

| 笔记 | 核心内容 |
|------|---------|
| [[MySQL SELECT FOR UPDATE 锁粒度分析]] | SELECT ... FOR UPDATE 的行锁、间隙锁、意向锁范围分析 |
| [[Undo-Redo日志详解]] | InnoDB 事务日志与 MVCC |

## 关联知识

- [[SQL-NULL值排序规则-数据库对比]] —— MySQL NULL 排序规则（与 Oracle、达梦对比）
