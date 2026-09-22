---
title: SQL-in和not-in的NULL陷阱
date: 2026-07-16
updated: 2026-09-22
aliases:
  - NOT IN NULL
  - IN EXISTS 选择
  - SQL 三值逻辑
source: "https://mp.weixin.qq.com/s/dmxBGoAHcKJMPRmc2bJ1og"
related:
  - "[[SQL-NULL值排序规则-数据库对比]]"
  - "[[MySQL索引创建原则]]"
  - "[[DECIMAL精度的定义与数据库上限]]"
tags:
  - topic/数据库
  - topic/SQL
  - topic/SQL优化
status: to-review
---

# SQL 中 IN 与 NOT IN 的 NULL 陷阱

## 一句话结论

**`IN` 不是不能用，`NOT IN` 才是真正要谨慎的那个。**

决定 SQL 好坏的从来不是关键字本身，而是数据量、索引、`NULL` 值、优化器改写与最终执行计划。

## 核心陷阱：NOT IN 遇到 NULL 会静默返回空结果

```sql
-- 目标：找出从未下过单的用户
SELECT u.id, u.name
FROM users u
WHERE u.id NOT IN (
    SELECT o.user_id FROM orders o   -- 只要这里出现一个 NULL，结果可能直接为空
);
```

### 原理：SQL 三值逻辑（true / false / **unknown**）

`5 NOT IN (1, 2, NULL)` 等价于：

```
5 <> 1  AND  5 <> 2  AND  5 <> NULL
  true       true        unknown
```

`5 <> NULL` 的结果**既不是 true 也不是 false，而是 unknown**。整个表达式变为 unknown，而 `WHERE` 只保留 true 的行 → **该行被过滤掉**。

> [!danger] 最危险的地方：它不报错
> SQL 正常执行、无任何异常，只是**结果错了**。线上报表、权限过滤、风控名单这类场景，因为一个 `NULL` 返回空结果，排查极其隐蔽。这属于"静默语义错误"，比性能问题危险得多。

## 推荐做法：反关联查询默认用 NOT EXISTS

```sql
SELECT u.id, u.name
FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

`NOT EXISTS` 判断的是"是否存在匹配行"，**子查询中其他记录的 `user_id` 是否为 NULL 不影响当前行的判断**。

如果确实必须用 `NOT IN`，至少要显式排除 NULL：

```sql
WHERE u.id NOT IN (
    SELECT o.user_id FROM orders o WHERE o.user_id IS NOT NULL
);
```

## IN 与 EXISTS 的正确取舍

| 场景 | 推荐 | 理由 |
|---|---|---|
| 固定小集合常量（状态枚举） | **`IN`** | 语义清楚，优化器易处理，无需改成 EXISTS |
| 大子查询做存在性判断 | **`EXISTS`** | 语义更准，可优化为半连接（Semi Join），找到一条即停 |
| 反关联（判断"不存在"） | **`NOT EXISTS`** | 不受子查询 NULL 影响 |

```sql
-- 小集合：这样写完全没问题，别强行改 EXISTS
SELECT id, order_no, status FROM orders
WHERE status IN ('PAID', 'SHIPPED', 'FINISHED');

-- 存在性判断：EXISTS 更贴合语义
SELECT u.id, u.name FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.pay_status = 'SUCCESS'
);
```

> [!tip] 别把 EXISTS 神化
> 现代 MySQL / PostgreSQL / SQL Server 优化器都不弱，简单的 `IN` 子查询经常也能被改写成半连接。最终要看 `EXPLAIN`，不能靠"背口号"判断性能。

## 性能的关键其实是索引

不要只盯着关键字。没有索引，写什么都慢。

```sql
-- 关联字段必须有索引：user_id 用于关联，pay_status 用于继续过滤
CREATE INDEX idx_orders_user_status ON orders(user_id, pay_status);

EXPLAIN SELECT u.id, u.name FROM users u
WHERE EXISTS (SELECT 1 FROM orders o
              WHERE o.user_id = u.id AND o.pay_status = 'SUCCESS');
-- 重点看 type、key、rows
```

如果 `key` 为空、`rows` 很大或出现全表扫描，问题通常在**索引设计、字段选择性、数据分布和统计信息**上，而不是 `IN` 慢或 `EXISTS` 快。

## 判断标准速记

1. 固定小列表 → `IN`
2. 判断存在 → `EXISTS`
3. 判断不存在 → `NOT EXISTS`（尤其子查询字段可能为 NULL 时）
4. 看到 `NOT IN` 接子查询 → **第一反应不是改不改，而是先确认子查询结果里有没有 NULL**，再看执行计划

## 参考链接

- [[SQL-NULL值排序规则-数据库对比]] — NULL 在各数据库中的其它语义差异
- [[MySQL索引创建原则]] — 索引设计决定这类查询的真实性能
