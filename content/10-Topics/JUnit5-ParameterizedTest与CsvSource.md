---
title: JUnit 5 @ParameterizedTest 与 @CsvSource
type: basic-note
date: 2026-06-12
tags:
  - language/java
  - topic/测试
status: to-review
---

# JUnit 5 @ParameterizedTest 与 @CsvSource

## 是什么

`@ParameterizedTest` 和 `@CsvSource` 是 **JUnit 5（Jupiter）** 提供的参数化测试注解，允许同一个测试方法使用 **多组不同的入参** 执行多次，避免为每个测试用例编写重复的测试方法。

### @ParameterizedTest

标记测试方法为参数化测试。被标注的方法必须至少有一个参数来源（如 `@CsvSource`、`@ValueSource`、`@MethodSource` 等）。

```java
@ParameterizedTest
@ValueSource(strings = {"apple", "banana", "cherry"})
void testFruit(String fruit) { ... }
```

#### name 属性 — 自定义显示名称

`name` 用于定制每个测试用例的报告名称，支持占位符替换：

| 占位符 | 含义 | 示例值 |
|--------|------|--------|
| `{0}` | 第一个参数 | `BigInt` |
| `{1}` | 第二个参数 | `Int` |
| `{2}` | 第三个参数 | `true` |
| `{index}` | 当前调用索引（从1开始） | `1`, `2`, `3` |
| `{arguments}` | 所有参数的完整字符串 | `BigInt, Int, true` |
| `{argumentsWithNames}` | 带参数名的完整字符串 | `from=BigInt, to=Int, compatible=true` |

示例中的 `name = "#14/15 类型 {0}→{1} 兼容={2}"` 会生成：
- `#14/15 类型 BigInt→Int 兼容=true`
- `#14/15 类型 Int→BigInt 兼容=false`

### @CsvSource

以 **CSV（逗号分隔值）** 格式提供内联测试数据，每行字符串对应一次测试调用。

```java
@CsvSource({
    "BigInt, Int, true",
    "Int, BigInt, false",
})
```

- 默认分隔符为逗号 `,`
- 可通过 `delimiter` 或 `delimiterString` 自定义分隔符
- 空值用空字符串表示，可用 `nullValues` 指定代表 null 的字符串

## 常用参数来源对比

| 注解 | 适用场景 | 支持多参数 | 数据来源 |
|------|---------|:--------:|---------|
| `@ValueSource` | 单参数简单类型 | ❌ | 内联 |
| `@CsvSource` | 少量多参数数据 | ✅ | 内联 CSV |
| `@CsvFileSource` | 大量多参数数据 | ✅ | 外部 CSV 文件 |
| `@MethodSource` | 复杂参数/动态生成 | ✅ | 工厂方法返回值 |
| `@EnumSource` | 枚举类型参数 | ❌ | 枚举常量 |
| `@ArgumentsSource` | 自定义参数来源 | ✅ | 自定义 `ArgumentsProvider` |

## 典型使用场景

### 1. 边界值与等价类测试

```java
@ParameterizedTest
@CsvSource({
    "0,    true",   // 最小值边界
    "1,    true",   // 正常值
    "100,  true",   // 最大值边界
    "-1,   false",  // 小于最小值
    "101,  false",  // 大于最大值
})
void testValidateScore(int score, boolean expected) { ... }
```

### 2. 类型兼容性矩阵（Schema Evolution）

如题目中的示例，测试不同类型之间的向前/向后兼容性：

```java
@ParameterizedTest(name = "类型 {0}→{1} 兼容={2}")
@CsvSource({
    "BigInt, Int, true",     // 向前兼容：宽度扩大
    "Int, BigInt, false",    // 不兼容：宽度缩小
    "Str, Int, false",       // 跨类型不兼容
    "Str, Str, true",        // 同类型兼容
})
void testTypeCompatibility(String from, String to, boolean expected) { ... }
```

详见 [[类型兼容性矩阵测试模式]]。

### 3. 业务规则验证

```java
@ParameterizedTest
@CsvSource({
    "PREMIUM,  100, 0.8",
    "NORMAL,   100, 1.0",
    "VIP,      100, 0.6",
})
void testDiscountRate(String level, double amount, double expectedRate) { ... }
```

## 注意事项

1. **@ParameterizedTest 替代 @RunWith(Parameterized.class)** — JUnit 4 风格已过时，JUnit 5 的 `@ParameterizedTest` 更简洁
2. **参数类型转换** — JUnit 5 内置常见类型转换（String → int/long/boolean/Enum 等），复杂类型需自定义 `ArgumentConverter`
3. **空值处理** — `@CsvSource` 中空字符串可通过 `nullValues = "N/A"` 映射为 null
4. **显示名称唯一性** — 善用 `name` 属性，让 IDE 报告一眼看出哪个用例失败

## 相关笔记

- [[类型兼容性矩阵测试模式]] — 基于此注解的 Schema Evolution 测试模式
- [[Java-Stream-终端操作]] — 同为 Java 技术栈测试相关
