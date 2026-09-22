---
title: MySQL SELECT FOR UPDATE 锁粒度分析
date: 2026-07-22
updated: 2026-07-22
aliases:
  - FOR UPDATE锁范围
  - 行锁与间隙锁
  - 聚合函数加锁行为
related:
  - "[[悲观锁]]"
  - "[[MySQL 隔离级别与锁机制]]"
tags:
  - topic/数据库/MySQL
  - topic/数据库/事务
  - topic/数据库/锁
status: to-review
---

## 核心结论

**`SELECT MAX(xxx) FROM t WHERE condition FOR UPDATE` 锁的不是一行，而是所有扫描过程中触及的记录 + 间隙 + 表级意向锁。**

锁的范围取决于：
1. `WHERE` 条件是否使用索引
2. `xxx`（MAX 的目标列）是否有索引
3. 当前隔离级别（RC vs RR）

---

## 锁的种类拆解

### 1. 行锁（Record Lock）—— 锁住的是"扫描路径"上的行

```sql
-- 假设：id 是主键，amount 无索引
SELECT MAX(amount) FROM orders WHERE id BETWEEN 1 AND 100 FOR UPDATE;
```

- `id BETWEEN 1 AND 100` 通过主键索引定位到 100 行
- InnoDB 会锁住 **这 100 行** 对应的聚簇索引记录
- 但 **锁的并不只是 MAX 的那一行**，而是所有被扫描到的行

> 💡 `FOR UPDATE` 锁的是**读取经过的行**，而不是"最终结果行"

### 2. 间隙锁（Gap Lock）—— 阻止幻读的关键（仅在 RR 级别）

```sql
-- 假设 amount 列上有索引，当前最大 amount = 500
SELECT MAX(amount) FROM orders WHERE status = 'PAID' FOR UPDATE;
```

在 **REPEATABLE READ** 级别下，除了行锁，InnoDB 还会加上间隙锁：

- **记录之间的间隙锁**：锁住索引树中相邻记录之间的空隙
- **最大值的右间隙（Supremum Gap）**：==最关键！== 如果在 amount 索引上扫描最大值，会在最大记录后面的 "supremum" 伪记录上加上 **next-key lock**，防止其他事务插入比当前 MAX 更大的值

效果：
```sql
-- 事务 A
BEGIN;
SELECT MAX(amount) FROM orders FOR UPDATE;  -- 返回 500

-- 事务 B（被阻塞！）
INSERT INTO orders(amount) VALUES (600);    -- 等待间隙锁释放
```

### 3. 意向锁（Intention Lock）—— 表级锁，自动伴随行锁

任何 `SELECT ... FOR UPDATE` 都会自动在表上加 **IX（Intention Exclusive）锁**：

| 锁类型 | 级别 | 作用 |
|--------|------|------|
| IX（意向排他锁） | 表级 | 标记"该表有事务在持有行锁" |
| X（行锁） | 行级 | 实际保护该行数据 |

作用：**协调表级锁和行级锁**。当另一个事务想 `LOCK TABLES ... WRITE` 锁全表时，通过检查 IX 锁就知道有行锁存在，无需逐行扫描。

---

## 不同场景的具体锁范围

### 场景 A：WHERE 条件走索引（精准范围）

```sql
SELECT MAX(price) FROM products WHERE category_id = 10 FOR UPDATE;
```
- `category_id` 有索引
- **锁**：category_id = 10 的所有记录 + 这些记录之间的间隙锁（RR 下）
- **不锁**：category_id ≠ 10 的行

### 场景 B：WHERE 条件无索引（全表扫描）

```sql
SELECT MAX(price) FROM products WHERE name LIKE '%keyword%' FOR UPDATE;
```
- `name` 无索引 → 需要全表扫描
- **锁**：**聚簇索引的所有记录** + 所有间隙 → 实际效果接近表锁
- ⚠️ 这是生产事故的常见根源

### 场景 C：无 WHERE 条件

```sql
SELECT MAX(id) FROM orders FOR UPDATE;
```
- 如果 `id` 是主键，InnoDB 会在主键索引的 **最右端**（最大值处）加 **next-key lock**
- 如果 `MAX` 的列无索引 → 全表扫描 → 全表锁

### 场景 D：典型的"取号"模式（高并发陷阱）

```sql
-- 业务需求：获取下一个可用的顺序号
BEGIN;
SELECT MAX(seq_no) FROM orders FOR UPDATE;  -- 锁住大量行
-- 应用层计算新 seq_no = max + 1
INSERT INTO orders(seq_no, ...) VALUES (max+1, ...);
COMMIT;
```

问题：`MAX()` + `FOR UPDATE` 的锁范围可能远大于预期，在高并发下严重限制吞吐量。

---

## 总结表

| 场景 | 行锁 | 间隙锁（RR） | 意向锁 | 备注 |
|------|------|-------------|--------|------|
| WHERE 走索引 | ✅ 锁匹配行 | ✅ 锁间隙 | ✅ IX | 范围精准 |
| WHERE 无索引 | ✅ 锁所有行 | ✅ 全表间隙 | ✅ IX | ⚠️ 高危，≈ 表锁 |
| 无 WHERE + MAX走索引 | ✅ 锁索引叶节点 | ✅ 最大值右间隙 | ✅ IX | 影响插入新最大值 |
| 无 WHERE + MAX无索引 | ✅ 锁所有行 | ✅ 全表间隙 | ✅ IX | ⚠️ 高危 |

---

## 如何避免锁范围过大？

1. **给 WHERE 和聚合列建索引**
2. **考虑 RC 隔离级别**（无 gap lock，但注意业务是否能接受）
3. **考虑不使用 `FOR UPDATE`**，改用乐观锁或独立序列生成器（如 Redis incr、雪花算法）
4. **如果只是取 MAX 值用于插入**，考虑 `AUTO_INCREMENT` 或 `SEQUENCE`（MySQL 8.0+）

---

## 参考链接
- "[[悲观锁]]"
