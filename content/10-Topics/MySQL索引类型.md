---
title: MySQL索引类型
tags:
  - topic/数据库
  - language/sql
aliases:
  - MySQL Index Types
  - MySQL 索引分类
---

# MySQL索引类型

索引是数据库表中一种==用于加速数据检索的辅助数据结构==。MySQL 的索引在存储引擎层实现，不同存储引擎支持不同的索引类型。从不同维度可对索引进行多种分类。

## 按数据结构分类（物理实现）

### B+Tree 索引

**InnoDB 的默认索引结构**，也是 MySQL 最核心的索引类型。

- 所有==实际数据或主键值存储在叶子节点==，非叶子节点仅存储键值用于导航
- 叶子节点形成**有序双向链表**，支持高效的范围查询（`BETWEEN`、`>`、`<`）和排序
- 通常 3-4 层即可存储数千万行数据，查询 IO 次数稳定
- 支持**最左前缀匹配**（针对[[MySQL联合索引|联合索引]]）

> [!info] B+Tree vs B-Tree
> B+Tree 非叶子节点不存数据，可容纳更多键值，降低树高，减少 IO 次数。所有数据都在叶子节点，使得范围查询时只需遍历链表即可。

### Hash 索引

- ==精确匹配等值查询极快==（O(1) 时间复杂度）
- **不支持**范围查询、排序、最左前缀匹配
- Memory 存储引擎显式支持；InnoDB 有**自适应 Hash 索引（AHI）**，由引擎自动管理热页的 Hash 映射，无需人工干预
- 适用于等值查询密集的场景

### Full-Text 索引

- 专用于全文检索场景，对文本内容进行**分词 + 倒排索引**处理
- 使用 `MATCH ... AGAINST` 语法查询
- MyISAM 和 InnoDB 均支持，InnoDB 从 MySQL 5.6 开始支持
- 适用于搜索引擎、文章内容检索等场景

### R-Tree（空间索引）

- 用于地理空间数据类型（`GEOMETRY`、`POINT`、`LINESTRING` 等）
- MyISAM 和 InnoDB（MySQL 5.7+）均支持
- 适用于 GIS、位置服务等场景

## 按功能逻辑分类（SQL 语法）

### 普通索引（INDEX / KEY）

```sql
CREATE INDEX idx_name ON table(column);
```

仅加速查询，允许重复值和 NULL 值，无约束作用。

### 唯一索引（UNIQUE）

```sql
CREATE UNIQUE INDEX idx_name ON table(column);
```

- 不允许重复值，但==允许 NULL（且允许多个 NULL 值）==
- MySQL 中 NULL ≠ NULL，因此多个 NULL 不违反唯一约束
- 兼具查询加速和约束双重作用
- 详细对比见 → [[主键索引与唯一索引的区别]]

### 主键索引（PRIMARY KEY）

```sql
ALTER TABLE table ADD PRIMARY KEY (column);
```

- 特殊的唯一索引 + **InnoDB 聚簇索引**
- 不允许 NULL
- 一个表只能有一个主键
- 详细对比见 → [[主键索引与唯一索引的区别]]

### 全文索引（FULLTEXT）

```sql
CREATE FULLTEXT INDEX idx_fulltext ON table(column);
```

对应 Full-Text 索引，用于 `MATCH ... AGAINST` 全文检索。

### 空间索引（SPATIAL）

```sql
CREATE SPATIAL INDEX idx_spatial ON table(column);
```

对应 R-Tree 索引，用于地理空间数据类型。

## 按列数分类

- **单列索引** — 只包含一个列的索引
- **[[MySQL联合索引|联合索引（复合索引）]]** — 包含多个列的索引，遵循 ==最左前缀原则==

## 按存储方式分类（InnoDB 特性）

### 聚簇索引（Clustered Index）

- ==InnoDB 表中，主键索引就是聚簇索引==
- 叶子节点存储整行数据，数据即索引，索引即数据
- 如果没有定义主键，InnoDB 会隐式选择第一个 UNIQUE NOT NULL 列作为聚簇索引；若无，则生成隐藏的 `ROW_ID` 作为聚簇索引
- 一个表只能有一个聚簇索引

### 二级索引（Secondary Index，也称辅助索引/非聚簇索引）

- 除主键索引外的其他索引都是二级索引
- 叶子节点存储的是**主键值**，而非整行数据
- 通过二级索引查询时，需要先得到主键值，再回聚簇索引查找完整数据 → **回表查询**

```mermaid
graph LR
    subgraph 二级索引
        A["索引列值 → 主键ID"]
    end
    subgraph 聚簇索引
        B["主键ID → 整行数据"]
    end
    A -->|回表| B
```

> [!tip] 覆盖索引避免回表
> 如果二级索引的 B+Tree 中已经包含了查询所需的所有列，则无需回表。此时该二级索引称为**覆盖索引**。

## 特殊索引类型

### 前缀索引

- 只对字符串列的前 N 个字符建立索引，==减少索引体积==
- 适用于 `VARCHAR` / `TEXT` 等长字符串列
- 选择性（Cardinality）需要平衡：前缀太短区分度低，前缀太长浪费空间

```sql
CREATE INDEX idx_email_prefix ON user(email(10));
```

### 不可见索引（Invisible Index, MySQL 8.0+）

- 优化器不可见的索引，但索引数据仍会维护更新
- 用于灰度删除索引：先设为不可见观察性能，确认无影响后再 `DROP`

```sql
ALTER TABLE table ALTER INDEX idx_name INVISIBLE;
ALTER TABLE table ALTER INDEX idx_name VISIBLE;
```

### 降序索引（Descending Index, MySQL 8.0+）

- 允许在 `CREATE INDEX` 时指定列排序方向为 `DESC`
- 支持多列混合排序（一列升序一列降序），避免 `filesort`

```sql
CREATE INDEX idx_a_desc_b_asc ON table(col1 DESC, col2 ASC);
```

### 函数索引（Function-based Index / 虚拟列索引, MySQL 8.0.13+）

- 允许对表达式或函数结果创建索引，避免查询中函数导致索引失效
- 底层通过**虚拟生成列（Generated Column）实现**

```sql
-- MySQL 8.0.13+ 直接创建函数索引
CREATE INDEX idx_year ON employee((YEAR(birth_date)));
```

## 相关笔记

- [[主键索引与唯一索引的区别]]
- [[MySQL索引创建原则]]
- [[MySQL联合索引]]
- [[接口性能排查指南]] — 慢 SQL 优化中的索引使用和 EXPLAIN 分析
- [[锁机制实现详解]] — Next-Key Lock 依赖索引
- [[MVCC-多版本并发控制]] — 索引与 Read View 的配合
