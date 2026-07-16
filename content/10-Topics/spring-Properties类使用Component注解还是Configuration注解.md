---
title: Spring @ConfigurationProperties — 使用 @Component 还是 @Configuration
date: 2026-06-24
tags:
  - language/java
  - topic/Spring
aliases:
  - ConfigurationProperties 注解选型
  - Component vs Configuration
  - SchemaSyncProperties
status: to-review
---

**`@Component` 更好，不要用 `@Configuration`。** 原因有三：

---

### 1. 语义不同

| 注解 | 语义 | 适用场景 |
| --- | --- | --- |
| `@Component` | 这是一个普通的 Spring Bean | 数据承载对象、服务、工具类 |
| `@Configuration` | 这是一个配置类，内含 `@Bean` 方法 | 定义和组装其他 Bean |

`SchemaSyncProperties` 只是存放配置值的 POJO，不产生任何 `@Bean`，用 `@Configuration` 在语义上是误导的。

### 2. `@Configuration` 会触发 CGLIB 代理

```java
@Configuration  // ← Spring 会对这个类创建 CGLIB 代理
public class SchemaSyncProperties { ... }
```

`@Configuration` 会让 Spring 对类进行 **CGLIB 代理增强**（为了拦截 `@Bean` 方法确保单例），但 `SchemaSyncProperties` 没有 `@Bean` 方法，这个代理是完全多余的开销。

`@Component` **不会创建 CGLIB 代理**，更轻量。

### 3. `@ConfigurationProperties` + `@Component` 是标准惯用法

Spring Boot 官方文档推荐的两种方式之一：

```java
// 方式 A：@Component 注册（Spring Boot 官方推荐惯用法之一）
@Component
@ConfigurationProperties(prefix = "rs.dictionary.auto-runner")
public class SchemaSyncProperties { ... }

// 方式 B：@EnableConfigurationProperties 注册
// 用于第三方类或不想侵入式的场景
```

方式 A 是完全正确的做法。

---

### 结论

```java
// SchemaSyncProperties.java — 正确用法
@Data
@ConfigurationProperties(prefix = "rs.dictionary.auto-runner")
@Component   // ✅ 正确：语义准、无代理开销、官方惯用法
public class SchemaSyncProperties {
    private boolean enabled = false;
    private List<String> tables = new ArrayList<>();
    private boolean dryRun = true;
    private SyncMode mode = SyncMode.SAFE;
    private boolean allowDropTable = false;
    private boolean logSql = true;
}
```

**一句话**：`@Component` 是"我是一个 Bean"，`@Configuration` 是"我定义其他 Bean"——你的类显然属于前者。
