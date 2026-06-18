---
title: Java Map.computeIfAbsent
type: basic-note
date: 2026-06-03
tags:
  - language/java
  - topic/java/集合
status: to-review
---

# Java Map.computeIfAbsent

Java 8 引入的 `Map` 接口方法，用于在 key 不存在时计算并插入值。

## 方法签名

```java
V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction)
```

## 执行逻辑

1. 检查 Map 中是否已有该 key
   - **有** → 直接返回对应的 value
   - **没有** → 执行 `mappingFunction`，将返回值作为新 value 放入 Map，返回该 value

## 等价写法

```java
// 一行代码实现分组
detailsByTabName.computeIfAbsent(detail.getTabName(), k -> new ArrayList<>()).add(detail);

// 等价于以下三行
if (!detailsByTabName.containsKey(detail.getTabName())) {
    detailsByTabName.put(detail.getTabName(), new ArrayList<>());
}
detailsByTabName.get(detail.getTabName()).add(detail);
```

## 典型场景：按 key 分组

将扁平列表按某个字段分组，例如：

```java
// 输入: [EMP-ID, EMP-NAME, DEPT-ID, DEPT-CODE, DEPT-NAME]
// 输出: {"EMP" → [ID, NAME], "DEPT" → [ID, CODE, NAME]}
Map<String, List<String>> grouped = new HashMap<>();
list.forEach(item -> grouped
    .computeIfAbsent(item.getType(), k -> new ArrayList<>())
    .add(item.getValue()));
```

## 相关笔记

- [[Java并发集合-ConcurrentHashMap]]