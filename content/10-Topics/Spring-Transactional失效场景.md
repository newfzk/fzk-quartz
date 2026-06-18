---
title: Spring @Transactional 失效场景
date: 2026-06-15
aliases:
  - 事务失效
  - Transactional失效
  - 声明式事务
related:
  - "[[事务ACID]]"
  - "[[事务传播行为]]"
  - "[[隔离级别]]"
tags:
  - language/java
  - topic/spring
  - topic/数据库/事务
status: to-review
---

# Spring @Transactional 失效场景

> `@Transactional` 是 Spring 声明式事务的核心注解，但有很多**隐式失效场景**，理解这些边界对正确使用事务至关重要。

## 失效场景详解

### 1. 自调用（类内部方法调用）

```java
@Service
public class UserService {
    
    public void parentMethod() {
        // 问题：this.childMethod() 不走 AOP 代理
        this.childMethod(); // 等同于直接调用，@Transactional 失效！
    }
    
    @Transactional
    public void childMethod() {
        // 事务不会生效
    }
}
```

**原因**：Spring AOP 基于代理，只有外部调用才会走代理。`this.childMethod()` 是直接调用目标对象的方法，没有经过代理对象的拦截器。

**解决方案**：
1. **注入自身代理**：`@Autowired UserService self` → `self.childMethod()`
2. **使用 AopContext**：`((UserService) AopContext.currentProxy()).childMethod()`
3. **分离到另一个 Bean**：将事务方法放到另一个 Service 中

### 2. private / static / final 方法

```java
@Transactional
private void privateMethod() { } // 不生效

@Transactional
static void staticMethod() { }   // 不生效

@Transactional
final void finalMethod() { }     // CGLIB 无法重写，不生效
```

- Spring AOP（JDK Proxy）只能拦截 public 方法
- CGLIB 通过生成子类重写方法实现，final 方法无法重写

### 3. 异常类型不匹配

```java
@Transactional
public void createUser() throws Exception {
    // 业务逻辑...
    throw new Exception("业务异常"); // checked exception → 不回滚！
}
```

- **默认回滚规则**：`RuntimeException` 和 `Error` 才会回滚
- **受检异常（Checked Exception）** 不回滚
- **解决方案**：`@Transactional(rollbackFor = Exception.class)`

### 4. 异常被捕获吞掉

```java
@Transactional
public void method() {
    try {
        // 业务逻辑...
        throw new RuntimeException();
    } catch (Exception e) {
        // 异常被吃掉，Spring 不知道有异常，不会回滚
        log.error("", e);
    }
}
```

- **事务回滚靠异常**，如果异常被捕获不抛出，Spring 认为执行成功
- **解决方案**：在 catch 中重新抛出异常

### 5. 数据库引擎不支持事务

```sql
-- MyISAM 引擎不支持事务，@Transactional 不生效
CREATE TABLE user_myisam (...) ENGINE=MyISAM;
```

- **解决方案**：确保表使用 InnoDB 引擎

### 6. 传播行为配置不当

```java
@Service
public class OuterService {
    @Autowired
    InnerService innerService;
    
    @Transactional
    public void outerMethod() {
        innerService.innerMethod(); // 内层事务抛异常
    }
}

@Service
public class InnerService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void innerMethod() {
        throw new RuntimeException(); // 内层事务回滚，但外层不一定会回滚
    }
}
```

- `REQUIRES_NEW` 会暂停外层事务，内层异常不会导致外层回滚（除非外层也检测到异常）

### 7. 多数据源事务（分布式事务）

```java
@Transactional
public void multiDataSourceMethod() {
    userDao.save(user);      // 数据源1
    orderDao.save(order);    // 数据源2
    // 一个 @Transactional 无法同时管理两个数据源的事务
}
```

- **解决方案**：使用 `@Transactional` + `Atomikos` / `Seata` / `TCC` 等分布式事务方案

### 8. 同事务内调用不同方法

- 同一个事务中，不管调多少次内部方法，都属于同一个数据库连接和事务
- 这通常不是"失效"，而是理解上的常见误区

## 正确使用检查清单

| 风险点 | 检查项 | 解决方法 |
|--------|--------|---------|
| 自调用 | 是否通过 `this.xxx()` 调用 | 注入自身代理或拆分 Bean |
| 方法可见性 | 是否是 private/static/final | 改为 public |
| 异常类型 | 抛出的异常是否被回滚规则覆盖 | 配置 `rollbackFor` |
| 异常捕获 | 异常是否被 try-catch 吞掉 | 重新抛出异常 |
| 引擎支持 | 数据表是否使用 InnoDB | 迁移到 InnoDB |
| 传播行为 | 内外层事务的交互是否符合预期 | 理解 Propagation 含义 |
| 多数据源 | 是否跨多个数据源 | 使用分布式事务 |

## 参考链接

- [[快手电商-一面-19题总结]] — Q14 @Transactional 失效场景
- [[事务ACID]] — 事务基础概念
- [[事务传播行为]] — 事务传播行为详解
- [[隔离级别]] — 事务隔离级别
