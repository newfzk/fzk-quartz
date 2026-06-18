---
title: Java Stream 惰性求值（Lazy Evaluation）
tags:
  - language/java
  - topic/java/stream
status: to-review
---

# Java Stream 惰性求值（Lazy Evaluation）

## 问题原因：Stream 缺少终端操作（Terminal Operation）

```java
// ❌ 错误写法：.map() 是惰性中间操作，没有终端操作不会执行
sqlSplitAndTrim(compatibleSql).stream().map(sql -> sqlInfoDtoList.add(new SqlInfoDto(sql)));
```

**`map()` 是 Stream 的中间操作（lazy intermediate operation），它不会执行 lambda 内部的代码——需要有终端操作（terminal operation）才能触发流处理链的执行。**

这段代码等价于：

```java
// 这行什么都没做，因为 .map() 是惰性的，没有终端操作就不执行
List<String> splitSql = sqlSplitAndTrim(compatibleSql); // ← 到这里就已经获取了SQL列表
Stream<String> stream = splitSql.stream();               // ← 创建了一个流
Stream<Boolean> mapped = stream.map(sql -> {
    return sqlInfoDtoList.add(new SqlInfoDto(sql));      // ← 但最后一行代码是 .map(...)
});                                 // ← 流上的映射从未被消费
```

### 对比能正常运行的代码

```java
// ✅ 正确写法：.collect() 是终端操作，触发执行
return sqlSplitAndTrim(createSql).stream()
    .map(SqlInfoDto::new)
    .collect(Collectors.toList());   // ← .collect() 是终端操作，触发执行
```

这里有 `.collect(Collectors.toList())` 作为终端操作，所以整个流水线会执行。

### 修复方案

将 `.map()` 改为 `.forEach()`（因为这里是要做副作用操作——往 list 里添加元素）：

```java
// ❌ 改前：
sqlSplitAndTrim(compatibleSql).stream().map(sql -> sqlInfoDtoList.add(new SqlInfoDto(sql)));
// ✅ 改后：
sqlSplitAndTrim(compatibleSql).forEach(sql -> sqlInfoDtoList.add(new SqlInfoDto(sql)));
```

### 总结

| 问题 | 修复 |
|------|------|
| `.stream().map(...)` 无终端操作 | 改为 `.forEach(...)` 或加上 `.collect(Collectors.toList())` |

## 相关笔记

- [[Java-Stream-终端操作]] — Stream 终端操作完整列表
