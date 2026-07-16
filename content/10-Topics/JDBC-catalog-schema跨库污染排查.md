---
tags:
  - jdbc
  - mysql
  - troubleshooting
  - database
status: to-review
---

# JDBC-catalog-schema跨库污染排查

## 现象

`SchemaMetadataService.getIndexes()` 读取索引时，多出数据库中不存在的幽灵索引。SQL 查询 `SHOW INDEX` 确认索引物理不存在，但 `getIndexInfo()` 返回了该索引。

## 根因

JDBC 元数据 API 的 `catalog`/`schema` 参数传 `null`，而 `null` 的语义是**"不过滤"**，返回所有数据库中同名表的信息。

```java
// 错误：返回所有库中同名表的索引
meta.getIndexInfo(null, null, tableName, false, false);
```

## 不同数据库的 catalog/schema 映射

| 数据库 | `conn.getCatalog()` | `conn.getSchema()` |
|--------|-------------------|-------------------|
| MySQL | 数据库名 | 被驱动忽略 |
| Oracle | null | 用户名/owner |
| 达梦 DM8 | null | 模式名 |

## 修复方案

同时传入 catalog 和 schema，各数据库驱动各取所需：

```java
String catalog = conn.getCatalog();
String schema = conn.getSchema();
meta.getIndexInfo(catalog, schema, tableName, false, false);
meta.getTables(catalog, schema, tableName, types);
meta.getColumns(catalog, schema, tableName, null);
meta.getPrimaryKeys(catalog, schema, tableName);
```

## 关键教训

- `null` 在 JDBC 元数据 API 中表示"不过滤"，而非"使用当前连接默认值"
- `""`（空串）匹配无 catalog/schema 的对象
- 排查方法论：SQL 层 vs JDBC 层分离验证 → 逐版本测试 → 输出 `TABLE_CAT` 原始数据定位来源 → 控制变量法确认根因
