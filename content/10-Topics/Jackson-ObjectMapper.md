---
title: Jackson ObjectMapper
type: basic-note
date: 2026-06-03
tags: java, jackson, json, 序列化
---

# Jackson ObjectMapper

## 是什么

- 来自 Jackson 库（`com.fasterxml.jackson.databind.ObjectMapper`）
- 核心职责：Java 对象 ↔ JSON 的双向转换
- Spring Boot 项目中会自动配置，可直接注入使用
- **线程安全**，应复用单例，无需每次 new

## 核心方法

### 序列化（Java → JSON）

| 方法 | 说明 | 返回值 |
|------|------|--------|
| `writeValueAsString(obj)` | 对象 → JSON 字符串 | `String` |
| `writeValueAsBytes(obj)` | 对象 → JSON 字节数组 | `byte[]` |
| `writeValue(File, obj)` | 对象 → JSON 写入文件 | `void` |
| `writerWithDefaultPrettyPrinter()` | 格式化输出的 Writer | `ObjectWriter` |

### 反序列化（JSON → Java）

| 方法 | 说明 | 返回值 |
|------|------|--------|
| `readValue(jsonStr, Class)` | JSON 字符串 → 指定类型 | `T` |
| `readValue(jsonStr, TypeReference)` | JSON 字符串 → 泛型类型（解决泛型擦除） | `T` |
| `readTree(jsonStr)` | JSON → `JsonNode` 树模型 | `JsonNode` |

## 常用配置

```java
// 忽略 JSON 中有但 Java 类没有的字段
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

// 仅序列化非 null 字段
objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);

// 日期格式化
objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

// 注册 Java 8 时间模块（LocalDateTime 等）
objectMapper.registerModule(new JavaTimeModule());
```

## 注意事项

- **循环引用**：默认会抛异常，可通过 `SerializationFeature.FAIL_ON_SELF_REFERENCES` 控制
- **泛型擦除**：反序列化泛型类型（如 `List<User>`）时必须用 `TypeReference`
- **Java 8 时间**：`LocalDateTime` 等需要 `JavaTimeModule`，否则序列化失败
- **线程安全**：`ObjectMapper` 是线程安全的，应作为单例复用

## 相关笔记

- [[Java-启动参数]]