---
tags:
  - java
  - map
  - jdk8
  - stream
status: to-review
---

# Java-Map-merge方法

## 方法签名

```java
V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> remappingFunction)
```

## 逻辑

- **key 不存在** → 直接放入 `value`
- **key 已存在** → 用 `remappingFunction(oldValue, value)` 计算结果，覆盖旧值

## 典型用法：多行记录归约

JDBC `getIndexInfo()` 对多列索引返回多行，用 `merge` 聚合唯一性标记：

```java
indexUniqueMap.merge(indexName, !nonUnique, (a, b) -> a && b);
```

- 首次遇到索引名 → 初始化 `!nonUnique`
- 再次遇到同索引名 → `(old, new) -> old && new`，只要有一行非唯一，结果就为 `false`

## 等价写法对比

```java
// merge（简洁，一行搞定）
indexUniqueMap.merge(indexName, !nonUnique, (a, b) -> a && b);

// 传统写法（啰嗦）
if (indexUniqueMap.containsKey(indexName)) {
    indexUniqueMap.put(indexName, indexUniqueMap.get(indexName) && !nonUnique);
} else {
    indexUniqueMap.put(indexName, !nonUnique);
}
```

`merge` 将"不存在则插入，存在则合并"的双重判断压缩为一条语句，是 Java 8 流式归约场景的典型用法。
