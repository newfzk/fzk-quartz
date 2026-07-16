---
title: 唯一约束（UNIQUE CONSTRAINT）与唯一索引（UNIQUE INDEX）的区别 — Oracle / 达梦
aliases:
  - Unique Constraint vs Unique Index
  - 唯一约束与唯一索引
  - Oracle UNIQUE CONSTRAINT vs INDEX
tags:
  - topic/Oracle
  - topic/达梦
  - topic/数据库/索引
  - topic/数据库/约束
status: to-review
created: 2026-07-16
---

# 唯一约束（UNIQUE CONSTRAINT）与唯一索引（UNIQUE INDEX）的区别

在 Oracle 和达梦数据库中，`UNIQUE CONSTRAINT`（唯一约束）和 `UNIQUE INDEX`（唯一索引）虽然都能保证数据的唯一性，但两者是**不同层面**的对象，存在本质差异。

## 一句话概括

| 维度 | UNIQUE CONSTRAINT | UNIQUE INDEX |
|------|-------------------|--------------|
| **本质** | 逻辑约束（业务规则） | 物理索引（数据结构） |
| **数据字典** | `USER_CONSTRAINTS` 可见 | `USER_INDEXES` 可见 |
| **创建方式** | `ALTER TABLE ... ADD CONSTRAINT ... UNIQUE` | `CREATE UNIQUE INDEX ...` |
| **索引副作用** | 自动创建/重用唯一索引 | 本身就是索引 |
| **外键引用** | ✅ 可以被引用 | ❌ 不能被引用 |
| **Deferrable** | ✅ 支持 | ❌ 不支持 |
| **删除行为** | 删除约束可能保留索引（如果是独立创建） | 删除索引直接删除 |

> 核心区别：**约束是声明"数据必须唯一"的规则，索引是实现该规则的物理手段。**

---

## Oracle 中的行为详解

### 1. 创建 UNIQUE CONSTRAINT 时发生了什么？

```sql
-- 方式一：创建唯一约束
ALTER TABLE employees ADD CONSTRAINT uk_emp_email UNIQUE (email);
```

Oracle 内部会：

1. 在 `USER_CONSTRAINTS` 中注册一个约束对象（`CONSTRAINT_TYPE = 'U'`）
2. **检查是否存在合适的索引**：
   - 如果列上已经有索引（普通索引或唯一索引），Oracle **重用**该索引，不创建新索引
   - 如果没有合适索引，Oracle **自动创建一个同名的唯一索引**
3. 该约束出现在 `USER_CONS_COLUMNS` 中，可被外键引用

```sql
-- 方式二：先建索引，后加约束 —— 约束重用已有索引
CREATE INDEX idx_emp_email ON employees(email);
ALTER TABLE employees ADD CONSTRAINT uk_emp_email UNIQUE (email);
-- 此时 uk_emp_email 约束重用了 idx_emp_email 索引，不会创建新索引
```

### 2. 创建 UNIQUE INDEX 时发生了什么？

```sql
-- 直接创建唯一索引
CREATE UNIQUE INDEX idx_emp_email_uk ON employees(email);
```

Oracle 内部会：

1. 在 `USER_INDEXES` 中注册一个唯一索引（`UNIQUENESS = 'UNIQUE'`）
2. **不会**在 `USER_CONSTRAINTS` 中创建任何约束对象
3. 该索引**不可被外键引用**

### 3. Deferrable 约束的特殊性

```sql
-- 可延迟的唯一约束
ALTER TABLE employees ADD CONSTRAINT uk_emp_email UNIQUE (email)
  DEFERRABLE INITIALLY DEFERRED;
```

Oracle 为可延迟约束创建的索引是**非唯一索引**，因为需要在事务提交前临时允许重复值存在。这在 `CREATE UNIQUE INDEX` 中无法实现。

### 4. 删除行为的差异

```sql
-- 删除唯一约束（约束自动创建了索引）
ALTER TABLE employees DROP CONSTRAINT uk_emp_email;
-- ⚠️ 默认会同时删除自动创建的索引

-- 删除唯一约束（约束重用了已有索引）
ALTER TABLE employees DROP CONSTRAINT uk_emp_email;
-- ✅ 约束被删除，但索引仍保留（因为不是自动创建的）

-- 删除唯一索引
DROP INDEX idx_emp_email_uk;
-- 唯一性立即失效，且不受影响
```

---

## 达梦数据库中的行为

达梦数据库兼容 Oracle 的行为模式：

### 1. 唯一约束

```sql
-- 达梦中创建唯一约束
ALTER TABLE employees ADD CONSTRAINT uk_emp_email UNIQUE (email);
```

达梦的行为与 Oracle 一致：
- 创建约束对象，在 `SYSOBJECTS` / `DBA_CONSTRAINTS` 中可见
- 自动创建对应的唯一索引（或重用已有索引）
- 约束可被外键引用

### 2. 唯一索引

```sql
-- 达梦中直接创建唯一索引
CREATE UNIQUE INDEX idx_emp_email_uk ON employees(email);
```

达梦的行为与 Oracle 一致：
- 仅创建索引对象，在 `DBA_INDEXES` 中可见
- **不创建约束对象**，在 `DBA_CONSTRAINTS` 中不可见
- 不能被外键引用

### 3. 达梦特有说明

达梦官方文档指出：

> 唯一约束是通过唯一索引来实现的。创建唯一约束时，达梦会自动生成一个对应的唯一索引来支持该约束。

达梦系统视图差异：

| 视图 | 唯一约束 | 唯一索引 |
|------|----------|----------|
| `DBA_CONSTRAINTS` | ✅ 可见 | ❌ 不可见 |
| `DBA_INDEXES` | ✅ 可见（自动生成的索引） | ✅ 可见 |
| 能否被外键引用 | ✅ | ❌ |

> [!note] 达梦与 Oracle 的兼容性
> 达梦对唯一约束/唯一索引的语义处理完全兼容 Oracle。如果你从 Oracle 迁移到达梦，关于这部分的 DDL 和 DML 行为无需修改。

---

## 实战场景对比

### 场景一：接口幂等

```sql
-- ✅ 使用唯一约束（明确业务规则："订单号必须唯一"）
ALTER TABLE idempotent_records ADD CONSTRAINT uk_order_no UNIQUE (order_no);

-- ❓ 使用唯一索引（也能保证唯一，但不表达业务意图）
CREATE UNIQUE INDEX idx_order_no ON idempotent_records(order_no);
```

**推荐**：幂等场景应使用唯一约束，因为：
- 自文档化，表达业务含义
- 可被外键引用（如果有子表记录关联）
- 具有标准化的约束管理接口

### 场景二：仅需加速查询

```sql
-- 如果列上已有其他索引，不需要再建约束
-- 已有普通索引，再加唯一约束会重用该索引
```

### 场景三：需定义可延迟约束

```sql
-- 唯一约束支持 DEFERRABLE
ALTER TABLE orders ADD CONSTRAINT uk_order_no UNIQUE (order_no)
  DEFERRABLE INITIALLY DEFERRED;
```

---

## 最佳实践（Tom Kyte 的建议）

根据 Oracle 首席架构师 Tom Kyte：

> **"The correct and proper way is to create the constraint and let the index be incidental."**
> （正确的方式是创建约束，让索引作为附带产物。）

```sql
-- ✅ 推荐：使用约束声明业务规则
ALTER TABLE employees ADD CONSTRAINT uk_emp_email UNIQUE (email);

-- ⚠️ 可选：如果明确需要特定索引类型，约束建好后显式创建/管理索引
CREATE UNIQUE INDEX idx_emp_email ON employees(email) TABLESPACE idx_tbs;
ALTER TABLE employees ADD CONSTRAINT uk_emp_email UNIQUE (email);
```

理由：
1. **约束是元数据** — 它是业务规则的声明，自文档化
2. **索引是实现细节** — 物理实现可以被 DBA 调整
3. **约束支持更多功能** — deferrable、外键引用等
4. **删除时更安全** — 约束和索引解耦，方便管理

---

## 常见面试题

> **Q: 如果一个表中已经有唯一索引（`CREATE UNIQUE INDEX`），再加唯一约束（`ADD CONSTRAINT ... UNIQUE`）会怎样？**

A: Oracle/达梦会检测到已有索引满足约束条件，直接重用该索引，不会创建新索引。约束对象被注册，索引被"借用"。

> **Q: 删除唯一约束时，自动创建的索引会被删除吗？**

A: 分两种情况：
- 如果索引是约束**自动创建**的 → 删除约束时索引**会**被删除
- 如果索引是**独立创建**后被约束重用的 → 删除约束时索引**不会**被删除（被保留）

> **Q: 唯一约束和唯一索引在 NULL 处理上有区别吗？**

A: 在 Oracle/达梦中，**两者行为一致**：都允许存在多个 NULL 值（因为 NULL != NULL，唯一定义不生效）。但在某些数据库（如 SQL Server）中，唯一索引只允许一个 NULL。

---

## 相关笔记

- [[主键索引与唯一索引的区别]] — MySQL 视角的主键 vs 唯一索引对比
- [[接口幂等方案设计]] — 唯一约束在幂等设计中的实际应用
- [[MySQL索引类型]] — 不同索引类型的全面分类
- [[MySQL索引创建原则]] — 索引设计与选择的最佳实践
