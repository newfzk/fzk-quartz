---
title: MyBatis-Plus-MPJLambdaWrapper多表关联查询
date: 2026-08-10
updated: 2026-09-22
aliases:
  - MPJLambdaWrapper
  - mybatis-plus-join
  - selectAs 字段映射
related:
  - "[[MyBatis-Plus-Page分页对象详解]]"
tags:
  - topic/MyBatis
  - topic/java
status: to-review
---

# MyBatis-Plus MPJLambdaWrapper 多表关联查询

## MPJLambdaWrapper 是什么

来自第三方扩展 **mybatis-plus-join（MPJ）**，专门用于**多表连接查询（JOIN）**。

| 对比项 | 原生 `LambdaQueryWrapper` | `MPJLambdaWrapper` |
|---|---|---|
| 表数量 | 仅单表 | 支持 `LEFT JOIN`、`INNER JOIN` 等 |
| 结果映射 | 实体本身 | 可自动映射到 DTO / VO |
| 字段引用 | Lambda Getter | Lambda Getter（同样避免硬编码） |

优势：用 Lambda 引用实体 Getter，**避免硬编码字段名、编译期可检查**，同时保留 SQL 的灵活性。

## `selectAs` — 定制返回字段并映射到 DTO

```java
wrapper.selectAs(GenSysCompanyDO::getUpdCode,
                 Sys0030MygridpanelPQueryCompany2QueryDTO::getUpdCode);
// ... 共 8 个 selectAs 调用
wrapper.selectAs(GenSysCompanyDO::getInsCode,
                 Sys0030MygridpanelPQueryCompany2QueryDTO::getComCode2);
```

- **作用**：显式指定要查询的字段，并**把字段值映射到目标 DTO 的对应属性**
- **等价 SQL**：`SELECT upd_code AS updCode, rec_code AS recCode, ...`
- **第一个参数**：源实体类的字段引用（Getter）
- **第二个参数**：目标 DTO 中接收该字段的属性引用

> [!warning] 字段名不一致必须显式映射
> 若源字段名与 DTO 属性名不同，**必须用 `selectAs` 指定**，否则框架无法自动填充。上面最后一行把 `insCode` 映射到 DTO 的 `comCode2`，就是一次别名重映射。

意义：联查时只取需要的列，并灵活转成前端需要的 DTO 结构，减少无用字段传输。

## `eq` — 带非空判断的动态条件

```java
wrapper.eq(param.getComCode() != null,
           GenSysCompanyDO::getComCode, param.getComCode());
```

- **作用**：添加等值条件 `WHERE com_code = ?`
- **第一个参数**：布尔表达式，**仅当为 true 时才拼接该条件**
- **好处**：避免手写 `if` 判断，代码更简洁

## 外部服务注入业务逻辑

```java
wrapper = genAuthSys0030Service.CallPQueryCompany2(paramsMap, wrapper);
```

把已构建的 `wrapper` 交给 Service 层方法继续增强，典型用途：

- 添加**数据权限过滤**（如只查当前用户有权限的公司）
- 添加额外 JOIN 或 WHERE 条件

> [!tip] 设计价值
> 将业务规则（尤其是权限控制）抽离到 Service 层，保持 Mapper 层整洁。这是"查询构建"与"业务规则"解耦的常见做法。

## 与分页结合

```java
Page<SomeDTO> page = new Page<>(current, size);
IPage<SomeDTO> result = someMapper.selectJoinPage(page, wrapper);
```

`selectJoinPage` 执行带 JOIN 的分页查询，并根据 `selectAs` 的映射把结果填入 DTO。

## 与原生 MyBatis 对比

| 方式 | 缺点 / 优点 |
|---|---|
| 传统 MyBatis | 需手写 XML SQL + `<resultMap>` 配置字段映射，维护复杂易错 |
| MyBatis-Plus + MPJ | Java 链式调用，类型安全，同时保留 SQL 灵活性 |

## 组件职责速查

| 组件 | 作用 |
|---|---|
| `MPJLambdaWrapper` | 多表连接查询条件构建器 |
| `selectAs(...)` | 指定字段并映射到 DTO 属性 |
| `eq(条件, ...)` | 带非空判断的动态等值条件 |
| Service 注入方法 | 注入权限、附加条件等业务逻辑 |
| `selectJoinPage` | 执行带 JOIN 的分页查询 |

## 参考链接

- [[MyBatis-Plus-Page分页对象详解]] — 分页对象的属性与用法
