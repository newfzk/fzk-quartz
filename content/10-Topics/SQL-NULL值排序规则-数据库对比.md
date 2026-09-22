---
title: NULL 值排序规则 — MySQL vs Oracle vs 达梦
aliases:
  - "NULL Sorting Rules"
  - "NULL 排序对比"
  - "NULLS FIRST / NULLS LAST"
  - "ORDER_BY_NULLS_FLAG"
tags:
  - topic/SQL
  - topic/数据库
  - topic/MySQL
  - topic/Oracle
  - topic/达梦
status: to-review
created: 2026-07-16
updated: 2026-07-16
---

# NULL 值排序规则 — MySQL vs Oracle vs 达梦

在 SQL 中，`NULL` 表示"未知值"（Unknown）。当对包含 `NULL` 的列进行排序时，各数据库对 `NULL` 的排序位置处理方式**不同**。如果不了解这些差异，在数据库迁移或多数据库开发中容易导致结果不一致。

## 一句话总结

| 数据库 | `ORDER BY col ASC` | `ORDER BY col DESC` | 支持 `NULLS FIRST/LAST` | 可配置默认行为 |
|-------|-------------------|--------------------|------------------------|:------------:|
| **MySQL** | `NULL` **最前** | `NULL` **最后** | ✅ 8.0.21+ | ❌ 无参数 |
| **Oracle** | `NULL` **最后** | `NULL` **最前** | ✅ 原生支持 | ❌ 无参数 |
| **达梦 DM8** | `NULL` **最前** ⚠️ | `NULL` **最前** ⚠️ | ✅ 支持 | ✅ `ORDER_BY_NULLS_FLAG` |
| **ANSI SQL 标准** | 未定义默认行为 | 未定义默认行为 | 可选特性 T611 | — |

> [!warning] 最关键的差异
> - MySQL 和 Oracle 在 `ASC`（升序）下默认行为**正好相反**
> - 达梦默认行为（`ORDER_BY_NULLS_FLAG=0`）是 NULL **永远排在最前**，与 MySQL 的 ASC 行为一致，但 DESC 也排最前——**既不是 Oracle 兼容也不是 MySQL 兼容**
> - 生产环境中达梦常设为 `ORDER_BY_NULLS_FLAG=1` 以兼容 Oracle

---

## ANSI SQL 标准

### SQL-92（基础定义）

ANSI SQL-92（X3.135-1992, Section 13.1）规定：

1. NULL 在 `ORDER BY` 中的比较行为是**实现定义**的（implementation-defined）
2. 所有 NULL 必须被**一致地**视为全部大于或全部小于非 NULL 值——不允许"某些 NULL 大于、某些 NULL 小于"
3. 但没有规定到底是"大于"还是"小于"

### SQL:2003+（可选语法）

SQL:2003（ISO/IEC 9075）引入 `NULLS FIRST` 和 `NULLS LAST` 作为 `ORDER BY` 的显式语法元素，定义在 `<null ordering>` 子句（Subclause 10.10）中。这是一个**可选特性**（Feature T611, "Elementary OLAP operations"），实现可选择不提供该语法。该语法在 SQL:2003、SQL:2008、SQL:2011、SQL:2016、SQL:2023 各版本中持续保留。

```sql
-- ANSI 标准语法（可选特性 T611）
ORDER BY col ASC NULLS FIRST   -- NULL 排最前
ORDER BY col DESC NULLS LAST   -- NULL 排最后
```

---

## MySQL

### 默认行为

MySQL 将 `NULL` 视为**小于所有非 NULL 值**（即 `NULL` 是最小值）。此行为自 MySQL 4.0.10 起一直保持一致。

| 排序方式 | NULL 位置 |
|---------|-----------|
| `ORDER BY col ASC` | NULL **最前**（first） |
| `ORDER BY col DESC` | NULL **最后**（last） |

```sql
CREATE TABLE t (a INT);
INSERT INTO t VALUES (1), (NULL), (3), (2), (NULL);

SELECT * FROM t ORDER BY a ASC;
-- 结果: NULL, NULL, 1, 2, 3

SELECT * FROM t ORDER BY a DESC;
-- 结果: 3, 2, 1, NULL, NULL
```

### NULLS FIRST / NULLS LAST 支持

- **MySQL 8.0.21+**：完整支持 `NULLS FIRST` 和 `NULLS LAST` 语法
- **MySQL 8.0.21 之前**：不支持，需用表达式变通

```sql
-- MySQL 8.0.21+
SELECT * FROM t ORDER BY a ASC NULLS LAST;   -- 1, 2, 3, NULL, NULL
SELECT * FROM t ORDER BY a DESC NULLS FIRST;  -- NULL, NULL, 3, 2, 1
```

> [!tip] MySQL 8.0.21 之前的兼容写法
> ```sql
> -- ASC 时想 NULL 放最后
> SELECT * FROM t ORDER BY ISNULL(a), a ASC;
> 
> -- DESC 时想 NULL 放最前
> SELECT * FROM t ORDER BY a IS NULL DESC, a DESC;
> ```

### 可配置性

MySQL **没有**任何服务器级或会话级参数来改变 NULL 排序的默认行为。`NULLS FIRST/LAST` 只支持在单条 SQL 语句中使用。

### 索引与 NULL 排序

- MySQL 的 B+Tree 索引中，**NULL 值在索引中排在非 NULL 值之前**（与默认排序行为一致）
- `IS NULL` 条件可以利用索引进行查找
- 混合排序方向（`ASC` 用于非 NULL 列、`DESC` 用于 NULL 判定表达式）可能导致 `Using filesort`

---

## Oracle

### 默认行为

Oracle 将 `NULL` 视为**大于所有非 NULL 值**（即 `NULL` 是最大值）。

| 排序方式 | NULL 位置 |
|---------|-----------|
| `ORDER BY col ASC` | NULL **最后**（last） |
| `ORDER BY col DESC` | NULL **最前**（first） |

```sql
SELECT * FROM t ORDER BY a ASC;
-- 结果: 1, 2, 3, NULL, NULL

SELECT * FROM t ORDER BY a DESC;
-- 结果: NULL, NULL, 3, 2, 1
```

### NULLS FIRST / NULLS LAST 支持

Oracle 早在 SQL:2003 标准之前就已原生支持 `NULLS FIRST` 和 `NULLS LAST` 语法。

```sql
-- 改变默认行为
SELECT * FROM t ORDER BY a ASC NULLS FIRST;   -- NULL, NULL, 1, 2, 3
SELECT * FROM t ORDER BY a DESC NULLS LAST;    -- 3, 2, 1, NULL, NULL
```

> [!quote] Oracle 官方文档
> "NULLS FIRST 和 NULLS LAST 可显式控制 NULL 在排序中的位置。省略时，NULLS LAST 用于 ASC，NULLS FIRST 用于 DESC。"

### 可配置性

Oracle **没有**全局配置参数来改变 NULL 排序的默认行为。只能通过逐条 SQL 的 `NULLS FIRST/LAST` 子句覆盖。

### 索引与 NULL 排序（重要）

Oracle B-tree 索引**不包含全部为 NULL 的行**（all-NULL rows are not indexed）。这一行为自 Oracle 8i 至 Oracle 23c 均保持一致。

这意味着：

- 对可空列（nullable column）进行 `ORDER BY` 时，优化器**无法直接使用索引来消除排序操作**（SORT ORDER BY），因为索引可能缺失那些全为 NULL 的行
- 解决方案：添加 `WHERE col IS NOT NULL` 约束后，优化器可以使用 `INDEX FULL SCAN` 替代排序
- 或者在列上添加 `NOT NULL` 约束（如果业务逻辑允许）

```sql
-- 以下无法利用索引消除排序（因为索引缺了全 NULL 行）
SELECT * FROM t ORDER BY a;

-- 添加 IS NOT NULL 后，优化器可用 INDEX FULL SCAN
SELECT * FROM t WHERE a IS NOT NULL ORDER BY a;
```

> [!note] 复合索引例外
> 如果索引包含多列，只要**任意一列非 NULL**，该行就会被包含在索引中。因此复合索引中，全 NULL 行的缺失问题只发生在**所有索引列均为 NULL** 的场景。

---

## 达梦 DM8

达梦的 NULL 排序规则是三款数据库中最灵活、也是最容易混淆的。

### 默认行为（`ORDER_BY_NULLS_FLAG=0`）

达梦 DM8 的**出厂默认**是 `ORDER_BY_NULLS_FLAG=0`，含义是 **NULL 永远排在所有非 NULL 值之前**，**与 ASC / DESC 方向无关**。

| 排序方式 | NULL 位置 |
|---------|-----------|
| `ORDER BY col ASC` | NULL **最前** |
| `ORDER BY col DESC` | NULL **最前**（⚠️ 与 Oracle 不同） |

```sql
-- 默认 ORDER_BY_NULLS_FLAG=0 时的结果
SELECT * FROM t ORDER BY a ASC;
-- 结果: NULL, NULL, 1, 2, 3

SELECT * FROM t ORDER BY a DESC;
-- 结果: NULL, NULL, 3, 2, 1  ← 注意这里 NULL 在最前面
```

### ORDER_BY_NULLS_FLAG 参数详解

达梦通过 `dm.ini` 中的 `ORDER_BY_NULLS_FLAG` 参数控制 NULL 排序默认行为，支持 **4 种模式**：

| 参数值 | 行为 | ASC 结果 | DESC 结果 | 适用场景 |
|:-----:|------|---------|----------|---------|
| **0** | NULL **永远最前**（出厂默认） | `NULL, NULL, 1, 2, 3` | `NULL, NULL, 3, 2, 1` | 默认值 |
| **1** | Oracle 兼容模式 | `1, 2, 3, NULL, NULL` | `NULL, NULL, 3, 2, 1` | **Oracle 迁移推荐** |
| **2** | MySQL 兼容模式 | `NULL, NULL, 1, 2, 3` | `3, 2, 1, NULL, NULL` | MySQL 迁移推荐 |
| **3** | 同 1 + 空串特殊处理 | 同 1 | 同 1 | 特殊兼容场景 |

> [!important] 生产环境建议
> 如果是从 **Oracle 迁移到达梦**，建议设置 `ORDER_BY_NULLS_FLAG=1`，否则默认行为会导致排序结果与 Oracle 不一致。许多达梦生产部署均采用值 1。

### 参数配置方式

达梦支持**三种粒度**修改 `ORDER_BY_NULLS_FLAG`：

#### 1. 系统级（dm.ini 或 SP_SET_PARA_VALUE）

```sql
-- 修改 dm.ini 并重启生效
SP_SET_PARA_VALUE(2, 'ORDER_BY_NULLS_FLAG', 1);
-- 参数：scope=2（永久生效，需重启）
```

#### 2. 会话级（sf_set_session_para_value）

```sql
-- 当前会话生效，不影响其他会话
SF_SET_SESSION_PARA_VALUE('ORDER_BY_NULLS_FLAG', 1);
```

#### 3. SQL 级（HINT）

```sql
-- 单条 SQL 生效，优先级最高
SELECT /*+ ORDER_BY_NULLS_FLAG(1) */ * FROM t ORDER BY a;
```

> [!tip] 优先级
> SQL HINT > 会话级设置 > 系统级设置

### NULLS FIRST / NULLS LAST 支持

达梦同时支持 ANSI 标准的 `NULLS FIRST` 和 `NULLS LAST` 语法，在 SQL 语句级别覆盖默认行为。即使 `ORDER_BY_NULLS_FLAG` 设为 0，查询中显式使用 `NULLS LAST` 也会生效。

```sql
-- 不管 ORDER_BY_NULLS_FLAG 设为多少，这条 SQL 都让 NULL 排最后
SELECT * FROM t ORDER BY a ASC NULLS LAST;
SELECT * FROM t ORDER BY a DESC NULLS LAST;
```

### 注意事项

1. **早期版本限制**：较早的 DM8 版本中，`ORDER_BY_NULLS_FLAG` 可能只影响 `ASC`，对 `DESC` 无效（`MAX_VALUE` 可能只有 1，即只支持值 0 和 1）。较新的版本（2023+）已完整支持 4 个值。
2. **NULL 之间的相对顺序**：当排序列有多个 NULL 时，它们**彼此之间的相对顺序是不确定的**（因为 NULL ≠ NULL）。如果结果集一致性很重要，建议在 `ORDER BY` 中添加排序列（如主键）作 tiebreaker。
3. **NULL peer-group 排序不确定性并非达梦独有**，MySQL、DuckDB、ClickHouse 等数据库中同样存在。

---

## 对比一览

```sql
-- 同一份数据: [1, NULL, 3, 2, NULL]

-- MySQL（默认）:
-- ASC  → NULL, NULL, 1, 2, 3
-- DESC → 3, 2, 1, NULL, NULL

-- Oracle（默认）:
-- ASC  → 1, 2, 3, NULL, NULL
-- DESC → NULL, NULL, 3, 2, 1

-- 达梦（ORDER_BY_NULLS_FLAG=0，出厂默认）:
-- ASC  → NULL, NULL, 1, 2, 3
-- DESC → NULL, NULL, 3, 2, 1

-- 达梦（ORDER_BY_NULLS_FLAG=1，Oracle兼容）:
-- ASC  → 1, 2, 3, NULL, NULL
-- DESC → NULL, NULL, 3, 2, 1

-- 达梦（ORDER_BY_NULLS_FLAG=2，MySQL兼容）:
-- ASC  → NULL, NULL, 1, 2, 3
-- DESC → 3, 2, 1, NULL, NULL
```

### 快速记忆法

| 记忆口诀 | 含义 |
|---------|------|
| **MySQL 把 NULL 当最小值** | ASC 排最前，DESC 排最后 |
| **Oracle 把 NULL 当最大值** | ASC 排最后，DESC 排最前 |
| **达梦默认 NULL 永远冲第一** | 默认值 0：不论 ASC/DESC，NULL 都排最前 |
| **达梦值 1 变 Oracle** | ASC 最后、DESC 最前 |
| **达梦值 2 变 MySQL** | ASC 最前、DESC 最后 |
| **写可移植 SQL 就用 NULLS FIRST/LAST** | 显式声明最保险 |

---

## 迁移注意事项

### MySQL → Oracle

```sql
-- MySQL 写法（依赖 NULL 排最前）
ORDER BY a ASC;  -- NULL 在前

-- 迁移到 Oracle 需改为
ORDER BY a ASC NULLS FIRST;  -- 显式声明 NULL 在前
-- 或检查业务逻辑是否真的需要 NULL 在前
```

### MySQL → 达梦

```sql
-- MySQL 写法
ORDER BY a ASC;   -- NULL 在前（MySQL）
ORDER BY a DESC;  -- NULL 在后（MySQL）

-- 达梦（值 0 默认）：ASC 相同，DESC 不同！
ORDER BY a ASC;   -- NULL 在前 ✅ 与 MySQL 一致
ORDER BY a DESC;  -- NULL 也在前 ❌ 与 MySQL 相反！

-- 推荐：要么设 ORDER_BY_NULLS_FLAG=2，要么每条 SQL 加 NULLS LAST
SET SF_SET_SESSION_PARA_VALUE('ORDER_BY_NULLS_FLAG', 2);
```

### Oracle → 达梦

```sql
-- Oracle 写法
ORDER BY a ASC;   -- NULL 在后

-- 达梦（ORDER_BY_NULLS_FLAG=0）：NULL 在前 ❌ 不同！
-- 达梦（ORDER_BY_NULLS_FLAG=1）：NULL 在后 ✅ 一致

-- 迁移后一定记得设置参数！
SP_SET_PARA_VALUE(2, 'ORDER_BY_NULLS_FLAG', 1);
```

### 可移植写法（推荐）

```sql
-- 不论目标数据库，显式指定 NULL 排序位置
ORDER BY a ASC NULLS LAST;    -- 明确 NULL 放最后
ORDER BY a ASC NULLS FIRST;   -- 明确 NULL 放最前
```

---

## 与其他 NULL 语义的关联

NULL 在 SQL 中的特殊行为不仅影响排序，还影响：

- **[[SQL-in和not-in的NULL陷阱|NOT IN 与 NULL 的三值逻辑问题]]** — NULL 导致 NOT IN 返回空结果集的陷阱
- **[[主键索引与唯一索引的区别]]** — MySQL 中 NULL 在主键与唯一索引中的差异（主键不允许 NULL、唯一索引允许多个 NULL）
- **[[唯一约束与唯一索引的区别-Oracle-Dm]]** — Oracle/达梦中 NULL 在唯一约束中的行为
- 聚合函数（`COUNT(col)` 忽略 NULL，`COUNT(*)` 不忽略）
- 比较运算（`NULL = NULL` 结果为 Unknown，不是 True）

---

## 相关笔记

- [[唯一约束与唯一索引的区别-Oracle-Dm]] — Oracle/达梦中 NULL 在唯一约束中的行为
- [[主键索引与唯一索引的区别]] — MySQL 中 NULL 在主键与唯一索引中的差异
- [[MySQL知识点汇总]] — MySQL 知识总览

## 参考资料

- [MySQL 8.0 ORDER BY Optimization - NULLS FIRST/LAST](https://dev.mysql.com/doc/refman/8.0/en/order-by-optimization.html)
- [MySQL 8.0 Release Notes - NULLS FIRST/LAST](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-21.html)
- [Oracle Database SQL Language Reference - ORDER BY clause](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/ORDER-BY-clause.html)
- [达梦社区 - ORDER_BY_NULLS_FLAG 参数讨论](https://eco.dameng.com/community/question/9ff06fe097b08b5cec4cbb0c8f608868)
- [达梦社区 - ORDER_BY_NULLS_FLAG 取值说明](https://eco.dameng.com/community/question/ce8ed0b1085627e674dfedde5552b9a8)
- SQL:2003 标准 T611 — `NULLS FIRST`/`NULLS LAST` 可选特性
