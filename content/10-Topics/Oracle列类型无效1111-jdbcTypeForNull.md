---
title: Oracle列类型无效1111-jdbcTypeForNull
date: 2026-07-31
updated: 2026-09-22
aliases:
  - 无效的列类型 1111
  - ORA-17004
  - jdbcTypeForNull
related:
  - "[[MyBatis-jdbcTypeForNull配置详解]]"
tags:
  - topic/数据库
  - topic/Oracle
  - topic/MyBatis
  - topic/故障排查
status: to-review
---

# Oracle 列类型无效 1111：null 值的 JDBC 类型问题

## 问题现象

Spring Boot + MyBatis / MyBatis-Plus 操作 Oracle 时，只要插入或更新的字段值为 `null`，就可能抛异常：

```text
java.sql.SQLException: 无效的列类型: 1111
```

完整堆栈形式：

```text
Cause: org.apache.ibatis.type.TypeException: Error setting null for parameter #X
  with JdbcType OTHER . Try setting a different JdbcType for this parameter
  or a different jdbcTypeForNull configuration property.
Cause: java.sql.SQLException: ORA-17004: 列类型无效: 1111
```

## 根本原因

1. MyBatis 为 `null` 参数未显式指定 `jdbcType` 时，默认使用 `JdbcType.OTHER`
2. `JdbcType.OTHER` 对应的类型码是 **1111**
3. **Oracle 驱动不支持 `JdbcType.OTHER`**，于是抛"无效的列类型: 1111"

链条的起点是 MyBatis 的 `jdbcTypeForNull` 配置项**默认值就是 `OTHER`**。

> [!info] 为什么其他数据库不报错
> MySQL 等数据库对 `null` 的类型校验宽松，能接受 `OTHER`；Oracle 校验严格，必须给出确切类型。所以这是**Oracle 特有**的坑，从 MySQL 迁到 Oracle 时才会暴露。

## 解决方案

核心思路：**让 MyBatis 处理 `null` 时使用 `JdbcType.NULL`，而不是 `OTHER`**。

### 方案一：全局配置（推荐）

**MyBatis-Plus**（`application.yml`）：

```yaml
mybatis-plus:
  configuration:
    jdbc-type-for-null: 'null'   # 引号必需
```

**原生 MyBatis**（`mybatis-config.xml`）：

```xml
<settings>
    <setting name="jdbcTypeForNull" value="NULL" />
</settings>
```

> [!warning] YAML 引号陷阱
> 必须写成 `'null'` 或 `"null"`，**不能直接写裸 `null`**。裸 `null` 会被 YAML 解析成 Java 的 null 值，配置不生效，问题照旧。这个坑很隐蔽——配置看起来"写了"，实际没生效。

### 方案二：局部配置（逐个参数指定）

Mapper XML 中：

```xml
<insert id="insertUser">
    INSERT INTO users (name, age)
    VALUES (#{name, jdbcType=VARCHAR}, #{age, jdbcType=INTEGER})
</insert>
```

### 方案三：代码配置

```java
@Bean
public ConfigurationCustomizer mybatisPlusCustomizer() {
    return configuration -> configuration.setJdbcTypeForNull(JdbcType.NULL);
}
```

## 选择建议

- **优先方案一**：一处配置解决全项目所有同类问题，一劳永逸
- 方案二适合只修个别语句、不便改动全局配置的场景
- 方案三适合需要按环境动态决定的场景

## 参考链接

- [[MyBatis-jdbcTypeForNull配置详解]] — 该配置项的完整参数说明
