---
title: Spring Boot 自动配置 — spring.factories 机制
date: 2026-06-08
aliases:
  - spring.factories
  - Spring自动配置
  - EnableAutoConfiguration
related:
  - "[[Spring-Resource资源抽象]]"
tags:
  - language/java
  - topic/spring-boot
status: to-review
---

# Spring Boot 自动配置 — spring.factories 机制

## 问题背景

当一个 JAR 包需要被其他项目引用时，如何让 JAR 内部的 `@Configuration` 类**自动被加载**，而无需消费者手动添加 `@ComponentScan` 或 `@EnableConfigurationProperties`？

## 方案对比

### 方案 A：依赖 @ComponentScan（不推荐用于库）

在消费者项目的配置类上添加 `@EnableConfigurationProperties`：

```java
@Configuration
@EnableConfigurationProperties(SchemaSyncProperties.class)  // 消费者手动注册
@MapperScan("com.rs.cloud.business.**.mapper*")
public class MybatisPlusConfig { ... }
```

**问题**：
1. 依赖消费者的 `@ComponentScan` 覆盖到库的包路径，不是所有项目都会扫到
2. 换一个项目引用同一个 JAR，需要重复配置
3. 无法使用 `@ConditionalOnProperty` 按条件控制 Bean 加载

### 方案 B：spring.factories 自动配置（推荐）

JAR 内部自己管理配置的注册：

```
business-dictionary.jar
├── com/rs/cloud/business/dictionary/config/
│   └── DictionaryAutoConfiguration.java  ← @Configuration + @ConditionalOnProperty
├── META-INF/
│   └── spring.factories                  ← org.springframework.boot.autoconfigure.EnableAutoConfiguration=...
```

消费者只需在 `pom.xml` 中声明依赖，无需任何额外注解。

## spring.factories 格式

```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.rs.cloud.business.dictionary.config.DictionaryAutoConfiguration
```

## 核心优势

| 对比维度 | @ComponentScan 方式 | spring.factories 方式 |
|---------|-------------------|---------------------|
| 消费者配置 | 需要手动添加注解 | **零配置**，引入依赖即可 |
| 条件控制 | 无法使用 `@ConditionalOnProperty` | 支持条件注解按需加载 |
| 可移植性 | 绑定到特定项目结构 | 独立于消费者项目 |
| 标准性 | 非标准做法 | Spring Boot 官方推荐 |

## @ConditionalOnProperty 用法

```java
@Configuration
@ConditionalOnProperty(
    prefix = "rs.dictionary.auto-sync",
    name = "enabled",
    havingValue = "true"
)
public class DictionaryAutoConfiguration {
    // 只有当配置 rs.dictionary.auto-sync.enabled=true 时，才加载此配置类
}
```

如果条件不满足，**整个配置类都不加载**，相关 Bean 根本不会被创建，避免启动报错。