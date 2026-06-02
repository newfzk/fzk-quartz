# ObjectMapper 讲解计划

## 概述

针对 `DictionaryService.java#L163` 处的 `objectMapper.writeValueAsString(tableInfo)` 代码，系统性地讲解 Jackson 的 `ObjectMapper` 类。

## 讲解大纲

### 1. ObjectMapper 是什么

* 来自 Jackson 库（`com.fasterxml.jackson.databind.ObjectMapper`）

* 核心职责：Java 对象 ↔ JSON 的双向转换

* 在 Spring Boot 项目中的自动配置机制

### 2. 构造方式

* `new ObjectMapper()` — 手动创建

* Spring 自动注入（本项目中的使用方式，见 `DictionaryService.java#L39` 构造函数注入）

* `ObjectMapper` 的线程安全性说明

### 3. 核心方法详解（含本项目示例）

#### 序列化（Java → JSON）

| 方法                                 | 说明              | 返回值            |
| ---------------------------------- | --------------- | -------------- |
| `writeValueAsString(obj)`          | 对象 → JSON 字符串   | `String`       |
| `writeValueAsBytes(obj)`           | 对象 → JSON 字节数组  | `byte[]`       |
| `writeValue(File, obj)`            | 对象 → JSON 写入文件  | `void`         |
| `writeValue(OutputStream, obj)`    | 对象 → JSON 写入输出流 | `void`         |
| `writerWithDefaultPrettyPrinter()` | 格式化输出的 Writer   | `ObjectWriter` |

#### 反序列化（JSON → Java）

| 方法                                  | 说明                        | 返回值        |
| ----------------------------------- | ------------------------- | ---------- |
| `readValue(jsonStr, Class)`         | JSON 字符串 → 指定类型对象         | `T`        |
| `readValue(jsonStr, TypeReference)` | JSON 字符串 → 泛型类型           | `T`        |
| `readTree(jsonStr)`                 | JSON 字符串 → `JsonNode` 树模型 | `JsonNode` |
| `readValue(InputStream, Class)`     | JSON 输入流 → 对象             | `T`        |

### 4. 常用配置方法

* `configure(DeserializationFeature, boolean)` — 反序列化特性

* `configure(SerializationFeature, boolean)` — 序列化特性

* `setSerializationInclusion(Include)` — 序列化时包含规则（`Include.NON_NULL` 等）

* `disable(SerializationFeature)` / `enable()` — 快捷开关

* `setDateFormat(DateFormat)` — 日期格式

* `registerModule(Module)` — 注册模块（如 JavaTimeModule）

### 5. 常见配置示例

* 忽略未知字段：`configure(FAIL_ON_UNKNOWN_PROPERTIES, false)`

* 日期序列化为指定格式：`setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"))`

* 仅序列化非 null 字段：`setSerializationInclusion(JsonInclude.Include.NON_NULL)`

* 缩进输出：`writerWithDefaultPrettyPrinter().writeValueAsString(obj)`

### 6. 本项目中的使用场景分析

* `DictionaryService` 中通过构造函数注入 `ObjectMapper`

* `getTablesJsonFilesToZip()` 方法中使用 `writeValueAsString(tableInfo)` 将表信息对象序列化为 JSON 字符串

* 序列化后的 JSON 作为文件内容写入 ZIP 包的每个条目中

* 输出文件名为 `{tabName}.json`

### 7. 注意事项

* 循环引用处理（`SerializationFeature.FAIL_ON_SELF_REFERENCES`）

* 泛型擦除问题与 `TypeReference`

* 性能：`ObjectMapper` 是线程安全的，应复用单例

* Java 8 日期类型需要 `JavaTimeModule` 或 `JSR310Module`

