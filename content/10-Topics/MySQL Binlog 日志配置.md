---
title: MySQL Binlog 日志配置
date: 2026-05-26
tags:
  - topic/数据库事务
  - topic/数据库配置
  - language/sql
aliases:
  - Binlog配置
  - 二进制日志
status: to-review
updated: 2026-05-26 15:30:00
---

# MySQL Binlog 日志配置

## 目录

1. [Binlog 基础配置](#1-binlog-基础配置)
2. [Binlog 格式选择](#2-binlog-格式选择)
3. [Binlog 相关重要参数](#3-binlog-相关重要参数)
4. [主从复制相关配置](#4-主从复制相关配置)
5. [配置示例](#5-配置示例)

---

> [!info] 关联阅读
> 详细的 MySQL 配置说明请参考：[[MySQL常用配置]]

---

## 1. Binlog 基础配置

```ini
# 启用 Binlog
log_bin = ON

# Binlog 文件路径及前缀
log_bin = /var/log/mysql/mysql-bin

# Binlog 文件名后缀格式
binlog_format = ROW

# Binlog 过期时间（天）
expire_logs_days = 7

# 最大 Binlog 文件大小
max_binlog_size = 1G
```

## 2. Binlog 格式选择

```ini
# 三种格式对比
# binlog_format = STATEMENT  # 基于语句，记录SQL语句
# binlog_format = ROW       # 基于行，记录行变更（推荐）
# binlog_format = MIXED      # 混合模式

# 生产环境推荐
binlog_format = ROW
```

**格式对比：**

| 格式 | 特点 | 适用场景 |
|------|------|----------|
| **STATEMENT** | 记录执行的SQL语句 | 读密集场景，存储空间小 |
| **ROW** | 记录行级别的变更 | 写密集场景，数据一致性高（推荐） |
| **MIXED** | 混合使用两种格式 | 兼顾性能和一致性 |

## 3. Binlog 相关重要参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `log_bin` | OFF | 是否启用 Binlog |
| `binlog_format` | ROW | Binlog 格式：STATEMENT/ROW/MIXED |
| `expire_logs_days` | 0 | Binlog 自动删除天数，0 表示不自动删除 |
| `max_binlog_size` | 1GB | 单个 Binlog 文件最大大小 |
| `sync_binlog` | 1 | Binlog 同步刷盘策略，1 表示每次提交都刷盘 |
| `binlog_row_image` | FULL | 行镜像模式：FULL/MINIMAL/NONE |
| `binlog_cache_size` | 32KB | Binlog 缓存大小 |
| `max_binlog_cache_size` | 4GB | 最大 Binlog 缓存大小 |

## 4. 主从复制相关配置

```ini
# 服务器唯一 ID
server-id = 1

# 中继日志配置
relay_log = /var/log/mysql/relay-bin
relay_log_index = /var/log/mysql/relay-bin.index

# 只读模式（从库配置）
read_only = ON

# 超级权限用户是否也只读
super_read_only = ON
```

## 5. 配置示例

```ini
[mysqld]
# Binlog 基础配置
log_bin = /var/log/mysql/mysql-bin
binlog_format = ROW
expire_logs_days = 7
max_binlog_size = 1G
sync_binlog = 1

# 主从复制配置
server-id = 1
relay_log = /var/log/mysql/relay-bin

# GTID 配置（MySQL 5.6+）
gtid_mode = ON
enforce_gtid_consistency = ON
```

---

**文档信息**：
- 创建日期：2026-05-26
- 适用读者：数据库开发者、DBA、后端工程师
- 难度级别：进阶

%% 本文档使用 Obsidian-flavored Markdown 编写 %%
