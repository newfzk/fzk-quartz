---
title: MyBatis-jdbcTypeForNull配置详解
date: 2026-07-31
updated: 2026-09-22
aliases:
  - jdbcTypeForNull
  - null 值的 JDBC 类型
related:
  - "[[Oracle列类型无效1111-jdbcTypeForNull]]"
tags:
  - topic/MyBatis
  - topic/数据库
status: to-review
---

# MyBatis jdbcTypeForNull 配置详解

## 参数定义

`jdbcTypeForNull` 用于指定：**当没有为参数提供特定 JDBC 类型时，`null` 值应当使用哪个 JDBC 类型**。

| 项 | 值 |
|---|---|
| **默认值** | `OTHER` |
| **Oracle 推荐值** | `NULL` |
| 配置位置（原生） | `mybatis-config.xml` 的 `<settings>` |
| 配置位置（MP） | `application.yml` 的 `mybatis-plus.configuration` |

## 为什么需要有这个配置

`null` 本身没有类型信息，但 JDBC 的 `PreparedStatement.setNull(int, int)` **必须传入一个 SQL 类型码**。MyBatis 因此在参数为 `null` 时需要一个"兜底类型"，`jdbcTypeForNull` 就是这个兜底值。

> [!abstract] 官方说明要点
> 某些数据库驱动**需要**指定列的 JDBC 类型；另一些驱动则可以使用 `NULL`、`VARCHAR`、`OTHER` 等通用值。

Oracle 属于前者——它不接受通用类型 `OTHER`（类型码 1111），因此必须把这个兜底值改成 `NULL`。

## 配置写法

```yaml
# MyBatis-Plus（application.yml）
mybatis-plus:
  configuration:
    jdbc-type-for-null: 'null'
```

```xml
<!-- 原生 MyBatis（mybatis-config.xml） -->
<settings>
    <setting name="jdbcTypeForNull" value="NULL" />
</settings>
```

## ⚠️ 三个易错点

1. **YAML 必须加引号**：`jdbc-type-for-null: 'null'`。裸写 `null` 会被解析为 Java null，配置不生效且无任何提示。
2. **值建议统一大写**：`NULL`，配置值通常不区分大小写，但大写更规范、也与枚举名一致。
3. **数据库差异要心里有数**：Oracle 必需此项配置，MySQL 一般无需改动。跨库项目应确认目标库的严格程度。

## 排查经验

出现"无效的列类型 1111"时，**先确认配置是否真的生效**，再怀疑驱动版本。一个快速的验证方式是在配置类中直接断点或将 `JdbcTypeForNull` 打印出来——因为 YAML 引号问题会让配置"看起来配了、实际没配"。

## 参考链接

- [[Oracle列类型无效1111-jdbcTypeForNull]] — 该配置缺失时的完整故障现象与排查
