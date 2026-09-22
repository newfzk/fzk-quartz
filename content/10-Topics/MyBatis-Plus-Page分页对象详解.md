---
title: MyBatis-Plus-Page分页对象详解
date: 2026-08-10
updated: 2026-09-22
aliases:
  - IPage
  - new Page
  - MyBatis-Plus 分页
related:
  - "[[MyBatis-Plus-MPJLambdaWrapper多表关联查询]]"
  - "[[MySQL深度分页优化]]"
tags:
  - topic/MyBatis
  - topic/java
status: to-review
---

# MyBatis-Plus Page 分页对象详解

## Page 是什么

`Page` 是 MyBatis-Plus 的核心分页模型类，全名：

```
com.baomidou.mybatisplus.extension.plugins.pagination.Page<T>
```

它的设计特点：**同时承载"请求参数"和"结果数据"**——你告诉它要哪一页、多少条，执行查询后结果和分页信息又封装回同一个对象。它实现了 `IPage<T>` 接口。

## 核心属性

| 属性 | 类型 | 说明 |
|---|---|---|
| `current` | `Long` | 当前页码，默认 `1` |
| `size` | `Long` | 每页条数，默认 `10` |
| `total` | `Long` | 总记录数，由 COUNT 查询自动填入 |
| `records` | `List<T>` | 当前页数据列表 |
| `orders` | `List<OrderItem>` | 排序字段信息 |
| `searchCount` | `boolean` | 是否执行 COUNT 查询，默认 `true` |

## ⚠️ 关键技巧：size = 0 只取总数不取数据

```java
Page<User> page = new Page<>(1, 0);
```

第二个参数 `size = 0` 时：

- **会执行 COUNT 查询**拿到 `total`
- **不查询数据列表**，`page.getRecords()` 返回空列表

> [!tip] 适用场景
> 只需要知道"满足条件的数据总量"、不需要具体数据的场景。比"查全量再 size()"高效得多。
>
> 注意区分：官方文档说的是 **`size < 0` 表示临时不分页**；传 `0` 的实际效果是只 COUNT 不取数。两者语义不同，`0` 才是"只要总数"的写法。

## 标准使用流程

```java
// 1. 创建分页对象：第 2 页，每页 5 条
Page<User> page = new Page<>(2, 5);

// 2. 构建查询条件（可选）
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
queryWrapper.ge("age", 18);

// 3. 执行分页查询
userMapper.selectPage(page, queryWrapper);

// 4. 从 Page 取结果
List<User> userList = page.getRecords();  // 当前页数据
long total = page.getTotal();             // 总记录数
long pages = page.getPages();             // 总页数
```

## 常用方法速查

| 方法 | 作用 |
|---|---|
| `getRecords()` | 当前页数据 |
| `getTotal()` | 总记录数 |
| `getPages()` | 总页数（由 total/size 计算） |
| `getCurrent()` | 当前页码 |
| `getSize()` | 每页条数 |

## 与多表关联查询配合

`Page` 可与 MPJ 的 wrapper 组合，执行带 JOIN 的分页查询：

```java
Page<SomeDTO> page = new Page<>(current, size);
IPage<SomeDTO> result = someMapper.selectJoinPage(page, wrapper);
```

## 参考链接

- [[MyBatis-Plus-MPJLambdaWrapper多表关联查询]] — 与 Page 配合的多表分页
- [[MySQL深度分页优化]] — 深翻页时 LIMIT 本身的性能问题
