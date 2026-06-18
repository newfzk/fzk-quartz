---
title: Java集合框架-HashSet
date: 2026-06-02
updated: 2026-06-02 00:00:00
tags:
  - language/java
  - topic/java/集合
status: to-review
---

## 核心概念

`HashSet` 基于 `HashMap` 实现，存储不重复元素，提供 O(1) 的 `add`/`remove`/`contains` 操作。

## 集合运算实现

```java
Set<String> onlyA = new HashSet<>(setA);
onlyA.removeAll(setB);  // 差集 A - B

Set<String> onlyB = new HashSet<>(setB);
onlyB.removeAll(setA);  // 差集 B - A
```

| 运算 | 方法 | 说明 |
|------|------|------|
| 差集 A-B | `a.removeAll(b)` | 从 a 中移除所有在 b 中的元素（原地修改） |
| 并集 | `a.addAll(b)` | 将 b 中元素加入 a（原地修改） |
| 交集 | `a.retainAll(b)` | 仅保留 b 中也存在的元素（原地修改） |

## 重要细节

- `removeAll` 是**原地操作**，先 `new HashSet<>(original)` 拷贝一份再操作
- 时间复杂度 O(n)，其中 n 是调用者集合的大小
- 从 Java 8+ 可用 Stream API：`setA.stream().filter(x -> !setB.contains(x)).collect(...)`

## 大文件优化

- 使用 `BufferedReader` 逐行读取，避免一次性加载全部到字符串
- 如果内存紧张，考虑分批处理或外部排序

## 面试要点

- `HashSet` vs `TreeSet`：HashSet O(1) 无序，TreeSet O(log n) 有序
- `removeAll` 等价于 `Set.difference`，底层遍历调用 contains
- 注意拷贝原集合，不要原地修改入参

## 参考链接

- [[柠檬微趣-笔试-Q1-文本文件去重|柠檬微趣-笔试-Q1-文本文件去重]]