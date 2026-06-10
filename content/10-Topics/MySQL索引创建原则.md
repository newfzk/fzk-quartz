---
title: MySQL索引创建原则
tags:
  - topic/数据库
  - topic/性能优化
  - language/sql
aliases:
  - 索引创建注意事项
  - 索引最佳实践
  - Index Best Practices
---

# MySQL索引创建原则

索引能极大加速查询，但**创建不当也会带来严重副作用**。以下是在 MySQL 中创建和设计索引时需遵循的核心原则与常见误区。

---

## 一、索引的代价

创建索引前必须意识到索引并非免费：

| 代价类型 | 说明 |
|---------|------|
| **空间代价** | 索引 B+Tree 占用磁盘空间，联合索引和长字段索引尤为明显 |
| **写入代价** | 每次 INSERT / UPDATE / DELETE 都需同步维护所有索引的 B+Tree 结构，写入变慢 |
| **查询优化器负担** | 索引过多时，优化器选择索引的代价增加，可能选错索引 |

> [!warning] 核心权衡
> **索引不是越多越好。** 每个索引对应一棵独立的 B+Tree，写操作时所有树都要同步更新。读写比例是决策关键：读多写少的表可适当多建索引，写密集的表应尽量减少索引。

---

## 二、选择索引列的原则

### 1. 高选择度（High Cardinality）

**选择度** = `COUNT(DISTINCT column) / COUNT(*)`。选择度越接近 1，索引越有效。

```sql
-- ✅ 好：性别列选择度低（~0.5），不适合单独建索引
-- ✓ 好：身份证号选择度 ≈ 1，非常适合建索引
```

> [!tip] 什么时候性别列值得建索引？
> 当查询中性别与其他字段作为 [[MySQL联合索引|联合索引]] 的前缀列，且过滤效果显著时（如只查询男性用户中的活跃用户），即使选择度低也有意义。

### 2. 优先选择查询频繁的列

```sql
-- 如果业务查询主要以 user_id 和 status 为条件
-- 应优先为这些列创建索引
CREATE INDEX idx_user_status ON order(user_id, status);
```

### 3. 优先选择短字段

- 索引列的==字段长度越小，B+Tree 每页可容纳的索引条目越多==，IO 效率越高
- `INT` / `BIGINT` 优于 `VARCHAR(255)`
- 长字符串考虑使用[[MySQL索引类型|前缀索引]]

---

## 三、联合索引设计原则

### 最左前缀原则

MySQL 联合索引的 B+Tree 按定义顺序逐级排序，查询**必须从最左列开始匹配**才能使用索引。

```sql
CREATE INDEX idx_a_b_c ON t(a, b, c);

-- ✅ 走索引
WHERE a = 1
WHERE a = 1 AND b = 2
WHERE a = 1 AND b = 2 AND c = 3
WHERE a = 1 AND c = 3   -- 用到 a，跳过 b 只部分走索引

-- ❌ 不走索引（未从最左列开始）
WHERE b = 2
WHERE c = 3
WHERE b = 2 AND c = 3
```

> [!info] "跳过"列的本质上
> `WHERE a = 1 AND c = 3` 中，索引只能用于过滤 `a = 1`，之后在 B+Tree 中无法跳过 b 列直接匹配 c 列——因为索引层级是按 `(a,b,c)` 逐层排序的。但 MySQL 8.0.13+ 的 **Skip Scan Range Access** 可以部分优化此场景。

详见 → [[MySQL联合索引#最左前缀原则]]

### 列顺序策略

```sql
-- 原则1：高选择度列放前面（从过滤效率角度）
CREATE INDEX idx_status_create ON order(status, create_time);
-- 如果 status 只有 3 种值，create_time 几乎唯一
-- → create_time 放前面过滤性更好

-- 原则2：等值条件列放前面，范围条件列放后面
WHERE a = 1 AND b > 10  → 索引 (a, b)
-- a 的等值过滤缩小范围后，b 的范围查找在 B+Tree 中高效
```

详见 → [[MySQL联合索引#列顺序优化策略]]

---

## 四、常见注意事项

### 1. 避免对索引列进行函数操作或计算

```sql
-- ❌ 索引失效：函数包裹索引列
WHERE YEAR(create_time) = 2024

-- ✅ 改为范围查询，走索引
WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01'

-- ❌ 隐式类型转换也会导致索引失效
WHERE phone = 13800138000    -- phone 是 VARCHAR，右侧是数字

-- ✅ 类型一致
WHERE phone = '13800138000'
```

### 2. 避免使用 `LIKE '%keyword'` 前缀模糊查询

```sql
-- ✅ 走索引（前缀匹配，类似范围查询）
WHERE name LIKE '张%'

-- ❌ 不走索引（通配符在最前，无法在 B+Tree 中导航）
WHERE name LIKE '%张'
WHERE name LIKE '%张%'
```

> [!tip] 全文搜索替代方案
> 如需全文模糊搜索，应使用 [[MySQL索引类型#Full-Text 索引|全文索引]] 或搜索引擎（Elasticsearch），而非 `LIKE '%keyword%'`。

### 3. 避免 OR 导致索引失效

```sql
-- ❌ OR 可能使索引失效（取决于 MySQL 优化器判断）
WHERE a = 1 OR b = 2

-- ✅ 可改写为 UNION
SELECT * FROM t WHERE a = 1
UNION
SELECT * FROM t WHERE b = 2

-- 或确保所有 OR 条件列在同一索引中
-- MySQL 5.0+ 的 Index Merge 也可优化部分 OR 场景
```

### 4. 避免 `NOT IN`、`!=`、`<>`

- 这些操作符通常无法使用索引
- 优化方向：改写为 `IN` 或 `>`、`<` 的组合

### 5. 小表不需要索引

- 全表扫描比索引查找更快（索引有额外 IO 和随机访问开销）
- 通常表记录数少于数百行时，全表扫描最优

### 6. 考虑覆盖索引（Covering Index）

- 如果==索引中已经包含了查询所需的所有列==，无需回表，大大提高查询效率
- 用 `EXPLAIN` 查看 `Extra` 列是否显示 `Using index`

```sql
-- idx_a_b 覆盖了查询中的所有列
CREATE INDEX idx_a_b ON t(a, b);
SELECT a, b FROM t WHERE a = 1;  -- Extra: Using index
```

### 7. 使用 `EXPLAIN` 验证索引使用情况

```sql
EXPLAIN SELECT * FROM t WHERE a = 1\G
```

重点关注列：
- **`key`**：实际使用的索引
- **`rows`**：扫描行数估计值
- **`Extra`**：`Using index`（覆盖索引）、`Using where`（回表后过滤）、`Using filesort`（需优化）

详见 → [[接口性能排查指南]]

### 8. 监控索引使用频率（MySQL 8.0）

```sql
-- 查看索引使用统计
SELECT * FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'your_db';
```

- 长时间未使用的索引 → 考虑删除（降低写放大）

### 9. 唯一索引与普通索引的选择

- 业务上需要唯一约束 → 唯一索引
- 纯加速查询 → 普通索引
- 性能差异：==唯一索引在插入时需要额外唯一性检查==，但查询时几乎无差异

### 10. 冗余索引问题

```sql
-- idx_a_b 已经包含了 idx_a 的功能，idx_a 是冗余索引
CREATE INDEX idx_a ON t(a);
CREATE INDEX idx_a_b ON t(a, b);
```

> [!warning]
> 联合索引的最左列如果已存在单独的索引，则单独的索引通常是冗余的（除非有只查 a 列但不查 b 列的特定高频查询）。定期审查并删除冗余索引。

---

## 五、面试高频场景：什么情况下索引会失效

| 场景 | 原因 | 解决方案 |
|------|------|---------|
| 索引列使用函数 | B+Tree 中存储的是原值，函数值无法匹配 | 改为范围查询或建函数索引 |
| 隐式类型转换 | 比较时 MySQL 对字符串转数字，转换为对表达式求值 | 保持类型一致 |
| `LIKE '%keyword'` | 通配符在最前，B+Tree 无法从中间开始导航 | 改为前缀匹配或用全文索引 |
| 联合索引未用最左列 | 索引排序依赖最左列 | 调整查询条件或[[MySQL联合索引|联合索引]]列顺序 |
| OR 条件各列独立索引 | 优化器评估后可能选择全表扫描 | 用 UNION 改写或用 Index Merge |
| `!=` / `NOT IN` | 范围无法索引化 | 改写为具体范围或业务层面处理 |
| 数据量太小 | 全表扫描比索引查找更快 | 无需处理 |

## 相关笔记

- [[MySQL索引类型]] — 索引分类总览
- [[MySQL联合索引]] — 最左前缀原则与列顺序
- [[主键索引与唯一索引的区别]]
- [[接口性能排查指南]] — EXPLAIN 分析、慢 SQL 实战
