---
title: SQL查询与优化（MOC）
date: 2026-09-22
updated: 2026-09-22
tags:
  - topic/数据库
  - topic/SQL
  - topic/SQL优化
  - topic/MOC
status: to-review
aliases:
  - SQL 知识地图
  - SQL MOC
  - 数据库查询优化
related:
  - "[[SQL-in和not-in的NULL陷阱]]"
  - "[[MySQL索引创建原则]]"
---

# SQL 查询与优化 — 知识地图

> [!abstract] 本笔记为 MOC（Map of Content）
> SQL 的坑分两类：**性能问题**（慢，但结果正确）和**语义陷阱**（快，但结果错误）。
> 后者更危险——不报错、无日志，只是悄悄给出错误答案。本 MOC 汇总两类知识。

## ⭐ 语义陷阱（静默错误，最危险）

| 主题 | 核心笔记 |
|:-----|:---------|
| **NOT IN 遇到 NULL 返回空结果** | [[SQL-in和not-in的NULL陷阱]] |
| **NULL 值排序规则（数据库对比）** | [[SQL-NULL值排序规则-数据库对比]] |
| **Oracle 列类型无效 1111** | [[Oracle列类型无效1111-jdbcTypeForNull]] |
| **DECIMAL 静默四舍五入** | [[DECIMAL精度的定义与数据库上限]] |

> [!danger] 共性：三值逻辑与类型推断
> 这几类问题的共同点是 **SQL 在语义模糊时倾向静默降级而非报错**：
> - `NOT IN (… NULL …)` → 整个表达式变 `unknown` → 整行被过滤
> - `null` 参数类型推断为 `OTHER` → Oracle 直接抛错（少数会报错的场景）
> - 小数超精度 → 四舍五入，不告警
> - 整数超范围 → 报错（唯一显式的）
>
> **写 SQL 时对 NULL 与精度要有显式预期，不能依赖默认行为。**

## 索引与性能

| 主题 | 核心笔记 |
|:-----|:---------|
| **索引创建原则** | [[MySQL索引创建原则]] |
| **索引类型** | [[MySQL索引类型]] |
| **联合索引** | [[MySQL联合索引]] |
| **深度分页优化** | [[MySQL深度分页优化]] |
| **全文索引 FULLTEXT** | [[MySQL-全文索引-FULLTEXT]] |
| **SELECT FOR UPDATE 锁粒度** | [[MySQL SELECT FOR UPDATE 锁粒度分析]] |

> [!tip] 性能问题的归因顺序
> 不要只盯着关键字（`IN` / `EXISTS`）。没有索引，写什么都慢。
> 看到全表扫描时，问题通常在**索引设计、字段选择性、数据分布和统计信息**，而非 SQL 写法。
> 判断依据永远是 `EXPLAIN` 的 `type` / `key` / `rows`，不是经验口号。

## 数据类型与建表

| 主题 | 核心笔记 |
|:-----|:---------|
| **DECIMAL 精度上限（达梦 38）** | [[DECIMAL精度的定义与数据库上限]] |
| **字符串存日期 vs 专用日期类型** | [[MySQL-字符串存储日期vs专用日期类型]] |

## 持久层框架

| 主题 | 核心笔记 |
|:-----|:---------|
| **jdbcTypeForNull 配置** | [[MyBatis-jdbcTypeForNull配置详解]] |
| **Page 分页对象** | [[MyBatis-Plus-Page分页对象详解]] |
| **MPJ 多表关联查询** | [[MyBatis-Plus-MPJLambdaWrapper多表关联查询]] |
| **@TableField 排除策略** | [[MyBatis-Plus-@TableField排除策略]] |

## 判断标准速记

| 场景 | 推荐写法 |
|---|---|
| 固定小集合（状态枚举） | `IN` |
| 大子查询判存在 | `EXISTS` |
| 判**不**存在（反关联） | **`NOT EXISTS`**（子查询可能含 NULL 时必选） |
| 必须用 `NOT IN` 时 | 子查询加 `WHERE x IS NOT NULL` |

## 参考链接

- [[MySQL知识点汇总]] — MySQL 专项知识地图
