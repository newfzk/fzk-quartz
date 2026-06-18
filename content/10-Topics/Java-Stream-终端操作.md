---
title: Java Stream 终端操作（Terminal Operation）
type: basic-note
date: 2026-06-12
tags:
  - language/java
  - topic/java/stream
status: to-review
---

# Java Stream 终端操作（Terminal Operation）

## 概述：Stream 流水线模型

Java 8 引入的 Stream API 采用 **流水线（Pipeline）** 设计模式，一条完整的 Stream 操作链由三阶段组成：

```plaintext
数据源（Source） → 中间操作（Intermediate Operations） → 终端操作（Terminal Operation）
     ↓                    ↓                              ↓
  集合/数组/IO          filter/map/sorted...           collect/forEach/count...
```

## 中间操作 vs 终端操作

| 特性 | 中间操作（Intermediate） | 终端操作（Terminal） |
|------|------------------------|---------------------|
| 执行方式 | **惰性（Lazy）** | **即时（Eager）** |
| 返回值 | 返回一个新的 Stream | 返回结果或 void |
| 是否触发执行 | ❌ 不触发流水线执行 | ✅ 触发整个流水线执行 |
| 能否链式调用 | ✅ 可以继续链式调用 | ❌ Stream 被消费，不可再调用 |
| 常见示例 | `filter()`, `map()`, `sorted()`, `peek()`, `distinct()`, `flatMap()` | `collect()`, `forEach()`, `count()`, `reduce()`, `anyMatch()`, `findFirst()` |

### 核心理解：惰性求值（Lazy Evaluation）

**中间操作是惰性的**——调用 `filter()` 或 `map()` 时，不会立即执行 lambda 里的代码，而是"记录"下要执行的操作。只有当终端操作被调用时，所有中间操作才会按顺序执行。

```java
// ⚠️ 这段代码什么都不会发生——缺少终端操作
list.stream()
    .map(s -> System.out.println(s));  // map 是中间操作，不会触发执行

// ✅ 加上终端操作后，lambda 才会执行
list.stream()
    .map(s -> s.toUpperCase())
    .forEach(System.out::println);     // forEach 是终端操作，触发流水线执行
```

## 常见的终端操作分类

### 1. 收集类

| 终端操作 | 作用 | 示例 |
|---------|------|------|
| `.collect(Collector)` | 将流元素收集到容器 | `stream.collect(Collectors.toList())` |
| `.toList()` | Java 16+，直接转不可变 List | `stream.toList()` |
| `.toArray()` | 转数组 | `stream.toArray(String[]::new)` |

### 2. 遍历类

| 终端操作 | 作用 | 说明 |
|---------|------|------|
| `.forEach(Consumer)` | 对流中每个元素执行操作 | 不保证顺序（并行流） |
| `.forEachOrdered(Consumer)` | 按源顺序遍历 | 保证相遇顺序 |

> [!warning] forEach vs map 的误用
> `forEach` 是终端操作，会执行；`map` 是中间操作，不会执行。
> 如果目的是对每个元素执行副作用操作（如打印、添加到集合），应使用 `forEach`，而非 `map`。
> 详见 [[Java-Stream-终端操作#常见 Bug 模式：map 缺少终端操作|常见 Bug 模式]]

### 3. 匹配/查找类

| 终端操作 | 返回类型 | 作用 |
|---------|---------|------|
| `.anyMatch(Predicate)` | `boolean` | 任一元素匹配 → true |
| `.allMatch(Predicate)` | `boolean` | 所有元素匹配 → true |
| `.noneMatch(Predicate)` | `boolean` | 无元素匹配 → true |
| `.findFirst()` | `Optional<T>` | 返回第一个元素 |
| `.findAny()` | `Optional<T>` | 返回任意元素（并行流优化） |

### 4. 汇聚/归约类

| 终端操作 | 作用 |
|---------|------|
| `.count()` | 计数 |
| `.min(Comparator)` | 最小值 |
| `.max(Comparator)` | 最大值 |
| `.reduce(identity, accumulator)` | 自定义归约操作 |
| `.reduce(accumulator)` | 返回 Optional 的归约 |

### 5. 统计类

```java
// IntStream / LongStream / DoubleStream 专用
IntSummaryStatistics stats = stream.summaryStatistics();
stats.getSum();    // 总和
stats.getCount();  // 计数
stats.getMin();    // 最小值
stats.getMax();    // 最大值
stats.getAverage(); // 平均值
```

## 常见 Bug 模式：map 缺少终端操作

这是最常犯的 Stream 错误——用了 `map()` 但忘记加终端操作：

```java
// ❌ 错误的写法：map 是中间操作，add 不会执行
sqlSplitAndTrim(compatibleSql).stream()
    .map(sql -> sqlInfoDtoList.add(new SqlInfoDto(sql)));
// 这样写，add 永远不会被调用，stream 流没有任何效果

// ✅ 修复 1：改为 forEach（有副作用时）
sqlSplitAndTrim(compatibleSql).stream()
    .forEach(sql -> sqlInfoDtoList.add(new SqlInfoDto(sql)));

// ✅ 修复 2：用 collect 收集结果（无副作用时）
List<SqlInfoDto> list = sqlSplitAndTrim(compatibleSql).stream()
    .map(SqlInfoDto::new)
    .collect(Collectors.toList());
```

> [!tip] 最佳实践：函数式无副作用
> 函数式编程建议避免副作用，推荐用 `collect` 收集结果，而非用 `forEach` 修改外部集合。
> 即：优先采用「修复 2」而非「修复 1」。

## 完整示例：不同终端操作的效果

```java
List<String> list = List.of("apple", "banana", "cherry");

// collect — 收集为 List
List<String> collected = list.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());          // → [APPLE, BANANA, CHERRY]

// count — 计数
long count = list.stream()
    .filter(s -> s.startsWith("a"))
    .count();                                // → 1

// anyMatch — 判断是否存在匹配
boolean hasApple = list.stream()
    .anyMatch(s -> s.equals("apple"));       // → true

// reduce — 拼接字符串
String joined = list.stream()
    .reduce("", (a, b) -> a + "," + b);     // → ,apple,banana,cherry
```

## Stream 执行时机图解

```plaintext
创建 Stream
     │
     ▼
.filter(...)    ──── 仅记录操作，不执行
     │
     ▼
.map(...)       ──── 仅记录操作，不执行
     │
     ▼
.collect(...)   ──── 终端操作！
     │
     ▼
  ┌─ 触发整个流水线 ──────────────────┐
  │  1. 从数据源获取元素               │
  │  2. 执行 filter 过滤               │
  │  3. 执行 map 转换                 │
  │  4. 收集到容器（collect）          │
  └──────────────────────────────────┘
```

## 相关笔记

- [[Java-Stream-终端操作]] — 本文
- [[Java-Stream-惰性求值]] — Stream 惰性求值原理与常见 Bug 模式
- [[MapReduce并行模式]] — Stream 的并行流底层思想与 MapReduce 模式有相似之处
- [[Java集合-ArrayList与LinkedList底层原理]] — Stream 常操作的集合底层原理
- [[Java并发集合-CopyOnWriteArrayList]] — 了解在并发场景下 Stream 遍历时的集合选择

## 面试常考问题

1. **Stream 的中间操作和终端操作有什么区别？** — 中间操作为惰性求值，终端操作触发执行
2. **为什么 map 后不加 collect/forEach 代码不执行？** — 因为没有终端操作触发流水线
3. **forEach 和 for-each 循环有什么区别？** — Stream 的 forEach 允许并行执行，可链式调用
4. **一个 Stream 能调用两次终端操作吗？** — 不能，Stream 已被消费，再次调用会抛出 `IllegalStateException`
