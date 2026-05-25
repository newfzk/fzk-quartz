---
title: MySQL 常用配置
date: 2026-05-25
tags:
  - topic/数据库事务
  - topic/并发控制
  - topic/数据库配置
  - language/sql
  - status/to-review
aliases:
  - MySQL配置
  - MySQL常用参数
---

# MySQL 常用配置

## 目录

1. [Undo Log 配置](#1-undo-log-配置)
2. [Redo Log 配置](#2-redo-log-配置)
3. [Binlog 日志配置](#3-binlog-日志配置)
4. [通用查询配置](#4-通用查询配置)
5. [最佳实践建议](#5-最佳实践建议)

---

> [!info] 关联阅读
> 详细的 Undo/Redo 日志原理说明请参考：[[Undo-Redo日志详解]]

---

## 1. Undo Log 配置

### 1.1 独立 Undo 表空间配置

```ini
# 启用独立 Undo 表空间（MySQL 5.6+）
innodb_undo_tablespaces = 3

# Undo 表空间文件路径（默认在 datadir 下）
# 文件命名格式：undo001, undo002, ...

# Undo 表空间大小（每个文件的初始大小）
innodb_undo_log_file_size = 1G

# 是否启用自动 truncate
innodb_undo_log_truncate = ON

# Undo 表空间最小保留大小
innodb_purge_rseg_truncate_frequency = 128
```

### 1.2 Undo 相关重要参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `innodb_undo_tablespaces` | 0 | 独立 Undo 表空间数量，0 表示共享表空间 |
| `innodb_undo_log_file_size` | 16MB | 每个 Undo 表空间文件大小 |
| `innodb_undo_log_truncate` | OFF | 是否允许自动收缩 Undo 表空间 |
| `innodb_max_undo_log_size` | 1GB | Undo 表空间触发收缩的阈值 |
| `innodb_purge_rseg_truncate_frequency` | 128 | purge 操作多少次后尝试收缩 |

### 1.3 配置示例

```ini
[mysqld]
# Undo 表空间配置
innodb_undo_tablespaces = 2
innodb_undo_log_file_size = 2G
innodb_undo_log_truncate = ON
innodb_max_undo_log_size = 4G
```

---

## 2. Redo Log 配置

### 2.1 Redo 日志文件配置

```ini
# Redo 日志文件数量（至少2个）
innodb_log_files_in_group = 2

# 每个 Redo 日志文件大小
innodb_log_file_size = 2G

# Redo 日志文件路径（默认在 datadir 下）
innodb_log_group_home_dir = ./

# 日志缓冲区大小
innodb_log_buffer_size = 64M
```

### 2.2 Redo 相关重要参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `innodb_log_files_in_group` | 2 | Redo 日志文件数量 |
| `innodb_log_file_size` | 48MB | 每个 Redo 日志文件大小 |
| `innodb_log_buffer_size` | 16MB | Redo 日志缓冲区大小 |
| `innodb_flush_log_at_trx_commit` | 1 | 日志刷新策略 |
| `innodb_flush_log_at_timeout` | 1 | 日志刷新间隔（秒） |

### 2.3 日志刷新策略

```ini
# 策略对比
# innodb_flush_log_at_trx_commit = 1  # 每次提交都刷盘（最安全，性能最低）
# innodb_flush_log_at_trx_commit = 2  # 每次提交写入 OS buffer，每秒刷盘
# innodb_flush_log_at_trx_commit = 0  # 每秒写入 OS buffer 并刷盘

# 生产环境推荐配置（兼顾安全和性能）
innodb_flush_log_at_trx_commit = 1
innodb_flush_log_at_timeout = 1
```

### 2.4 配置示例

```ini
[mysqld]
# Redo 日志配置
innodb_log_files_in_group = 4
innodb_log_file_size = 4G
innodb_log_buffer_size = 128M
innodb_flush_log_at_trx_commit = 1
```

---

## 3. Binlog 日志配置

### 3.1 Binlog 基础配置

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

### 3.2 Binlog 格式选择

```ini
# 三种格式对比
# binlog_format = STATEMENT  # 基于语句，记录SQL语句
# binlog_format = ROW       # 基于行，记录行变更（推荐）
# binlog_format = MIXED      # 混合模式

# 生产环境推荐
binlog_format = ROW
```

### 3.3 Binlog 相关重要参数

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

### 3.4 主从复制相关配置

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

### 3.5 配置示例

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

## 4. 通用查询配置

### 4.1 查询缓存配置（MySQL 8.0 已移除）

> [!warning] 版本说明
> MySQL 8.0 移除了查询缓存功能，以下配置仅适用于 MySQL 5.7 及以下版本。

```ini
# MySQL 5.7 及以下版本配置
# query_cache_type = ON
# query_cache_size = 64M
# query_cache_limit = 2M
```

### 4.2 查询优化器配置

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

### 4.3 连接与线程配置

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

### 4.4 查询相关重要参数

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

### 4.5 配置示例

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

## 5. 最佳实践建议

### 5.1 配置原则

| 场景 | Undo 配置建议 | Redo 配置建议 | Binlog 配置建议 |
|------|--------------|--------------|----------------|
| **OLTP（写密集）** | 增大 `innodb_undo_log_file_size` | 增大 `innodb_log_file_size` | `binlog_format = ROW` |
| **OLAP（读密集）** | 默认配置即可 | 默认配置即可 | 根据需要调整 |
| **高可用要求** | 启用 `innodb_undo_log_truncate` | `innodb_flush_log_at_trx_commit = 1` | `sync_binlog = 1` |
| **SSD 存储** | 适当增大缓冲区 | `innodb_log_file_size = 2G-4G` | 可增大 `max_binlog_size` |

### 5.2 生产环境推荐配置

> [!tip] 生产环境配置
> 以下配置适用于高并发写入的生产环境，可根据实际情况调整。

```ini
[mysqld]
# Undo 日志配置
innodb_undo_tablespaces = 3
innodb_undo_log_file_size = 2G
innodb_undo_log_truncate = ON
innodb_max_undo_log_size = 4G

# Redo 日志配置
innodb_log_files_in_group = 4
innodb_log_file_size = 4G
innodb_log_buffer_size = 128M
innodb_flush_log_at_trx_commit = 1

# Binlog 配置
log_bin = /var/log/mysql/mysql-bin
binlog_format = ROW
expire_logs_days = 7
max_binlog_size = 1G
sync_binlog = 1

# 主从复制配置
server-id = 1
gtid_mode = ON
enforce_gtid_consistency = ON

# 查询配置
max_connections = 500
wait_timeout = 60
thread_cache_size = 64
table_open_cache = 2048

# 慢查询日志
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

# 其他相关配置
innodb_flush_method = O_DIRECT
innodb_doublewrite = ON
```

### 5.3 修改配置后的操作

> [!danger] 重要操作
> 修改 `innodb_log_file_size` 后需要手动处理旧日志文件，否则 MySQL 无法启动。

```bash
# 修改 innodb_log_file_size 后需要手动删除旧日志文件
# 1. 停止 MySQL 服务
systemctl stop mysqld

# 2. 删除旧的 Redo 日志文件
rm -f /var/lib/mysql/ib_logfile*

# 3. 启动 MySQL 服务（会自动创建新的日志文件）
systemctl start mysqld
```

#### 常用查询命令

```bash
# 查看 Binlog 状态
mysql> SHOW BINARY LOGS;

# 查看慢查询日志状态
mysql> SHOW VARIABLES LIKE '%slow%';
```

---

> [!info] 关联阅读
> 详细的 Undo/Redo 日志原理说明请参考：[[Undo-Redo日志详解]]

---

**文档信息**：
- 创建日期：2026-05-25
- 适用读者：数据库开发者、DBA、后端工程师
- 难度级别：进阶

%% 本文档使用 Obsidian-flavored Markdown 编写 %%
