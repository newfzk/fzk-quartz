---
tags:
  - mybatis-plus
  - java
  - orm
status: to-review
---

# MyBatis-Plus-@TableField排除策略

`@TableField` 注解的 `insertStrategy` 和 `updateStrategy` 可控制字段在 SQL 生成时的包含策略。

## NEVER 策略

```java
@TableField(
    value = "ID",
    insertStrategy = FieldStrategy.NEVER,
    updateStrategy = FieldStrategy.NEVER
)
private Long id;
```

- **`insertStrategy = NEVER`** — `insert()` 时 SQL 中**不包含**该列，由数据库自增生成
- **`updateStrategy = NEVER`** — `updateById()` 时 SQL 中**不包含** `SET ID=?`，避免达梦等数据库报"试图修改自增列"

## 其他策略值

| 策略 | 行为 |
|------|------|
| `ALWAYS` | 始终包含该字段 |
| `NOT_NULL` | 字段非 null 时包含（默认） |
| `NOT_EMPTY` | 字段非 null 且非空时包含 |
| `DEFAULT` | 跟随全局策略 |
| `IGNORED` | 忽略该字段的 SQL 处理 |
| `NEVER` | 始终不包含该字段 |

> 适用于自增主键、数据库默认值字段等无需在应用层赋值的场景。
