---
title: Spring Boot — CommandLineRunner 启动后执行
date: 2026-06-08
aliases:
  - CommandLineRunner
  - ApplicationRunner
  - 启动后执行
tags:
  - language/java
  - topic/spring-boot
status: to-review
---

# Spring Boot — CommandLineRunner 启动后执行

## 基本概念

`CommandLineRunner` 是 Spring Boot 提供的函数式接口，在 Spring 容器初始化完成后、应用正式对外提供服务前，自动调用其 `run()` 方法。

```java
@FunctionalInterface
public interface CommandLineRunner {
    void run(String... args) throws Exception;
}
```

## 典型场景

- 数据字典自动建表/改表
- 初始化缓存
- 预加载配置
- 数据迁移脚本
- 启动后一次性校验

## 与 ApplicationRunner 的区别

| | `CommandLineRunner` | `ApplicationRunner` |
|---|---|---|
| 参数 | 原始 `String... args` | `ApplicationArguments args`（支持 `--key=value` 解析） |

功能上几乎等价，只是参数形式不同。

## 执行顺序控制

多个 Runner 通过 `@Order` 注解控制执行顺序：

```java
@Component
@Order(1)  // 数字越小越先执行
public class RunnerA implements CommandLineRunner { ... }

@Component
@Order(2)
public class RunnerB implements CommandLineRunner { ... }
```

## 示例

```java
@Component
public class AutoSchemaSyncRunner implements CommandLineRunner {

    @Override
    public void run(String... args) {
        // 服务启动后自动执行数据字典表结构比对与修复
        schemaSyncService.sync();
    }
}
```