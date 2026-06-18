---
title: MySQL 通用查询日志配置
date: 2026-05-26
aliases:
  - 查询日志配置
  - 查询优化配置
tags:
  - language/sql
  - topic/MySQL
  - topic/数据库/日志
  - topic/数据库/配置
status: to-review
---

# MySQL 通用查询日志配置

## 目录

1. [查询缓存配置](#1-查询缓存配置)
2. [查询优化器配置](#2-查询优化器配置)
3. [连接与线程配置](#3-连接与线程配置)
4. [查询相关重要参数](#4-查询相关重要参数)
5. [配置示例](#5-配置示例)

---

> [!info] 关联阅读
> 详细的 MySQL 配置说明请参考：[[MySQL常用配置]]

---

## 1. 查询缓存配置（MySQL 8.0 已移除）

> [!warning] 版本说明
> MySQL 8.0 移除了查询缓存功能，以下配置仅适用于 MySQL 5.7 及以下版本。

```ini
# MySQL 5.7 及以下版本配置
# query_cache_type = ON
# query_cache_size = 64M
# query_cache_limit = 2M
```

## 2. 查询优化器配置

```ini
# 优化器模式
optimizer_switch = index_merge=on,index_merge_union=on,index_merge_sort_union=on

# 是否开启慢查询日志
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
log_queries_not_using_indexes = ON

# 日志输出格式
log_output = FILE
```

**慢查询日志配置说明：**
- `slow_query_log`：启用慢查询日志
- `slow_query_log_file`：慢查询日志文件路径
- `long_query_time`：慢查询阈值（秒），超过此时间的查询会被记录
- `log_queries_not_using_indexes`：是否记录未使用索引的查询

## 3. 连接与线程配置

```ini
# 最大连接数
max_connections = 1000

# 最大并发连接数（推荐值）
max_connections = 500

# 连接超时时间
wait_timeout = 60
interactive_timeout = 60

# 线程缓存
thread_cache_size = 64

# 表缓存
table_open_cache = 2048
table_definition_cache = 2048
```

**连接超时说明：**
- `wait_timeout`：非交互连接超时时间（秒），适用于应用程序连接
- `interactive_timeout`：交互连接超时时间（秒），适用于客户端工具连接

## 4. 查询相关重要参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `max_connections` | 151 | 最大并发连接数 |
| `wait_timeout` | 28800 | 非交互连接超时时间（秒） |
| `interactive_timeout` | 28800 | 交互连接超时时间（秒） |
| `thread_cache_size` | 8 | 线程缓存大小 |
| `table_open_cache` | 2000 | 表打开缓存大小 |
| `slow_query_log` | OFF | 是否开启慢查询日志 |
| `long_query_time` | 10 | 慢查询阈值（秒） |
| `log_queries_not_using_indexes` | OFF | 是否记录未使用索引的查询 |

## 5. 配置示例

```ini
[mysqld]
# 连接配置
max_connections = 500
wait_timeout = 60
interactive_timeout = 60
thread_cache_size = 64

# 表缓存配置
table_open_cache = 2048
table_definition_cache = 2048

# 慢查询日志配置
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
log_queries_not_using_indexes = ON
```

---

**常用查询命令：**

```bash
# 查看慢查询日志状态
mysql> SHOW VARIABLES LIKE '%slow%';
```

---

**文档信息**：
- 创建日期：2026-05-26
- 适用读者：数据库开发者、DBA、后端工程师
- 难度级别：进阶

%% 本文档使用 Obsidian-flavored Markdown 编写 %%
