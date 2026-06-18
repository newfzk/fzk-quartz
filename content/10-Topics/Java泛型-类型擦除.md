---
title: Java 泛型类型擦除
date: 2026-06-15
aliases:
  - Type Erasure
  - 类型擦除
  - 泛型擦除
related:
  - "[[Java集合-ArrayList与LinkedList底层原理]]"
tags:
  - language/java
  - topic/java/基础
  - topic/java/泛型
  - type/面试题
status: to-review
---

# Java 泛型类型擦除

> Java 泛型（Generics）通过**类型擦除（Type Erasure）** 机制实现：编译期进行类型检查，运行时移除泛型类型信息，保证向后兼容性。

## 回答

**类型擦除（Type Erasure）** 是指 Java 泛型在编译阶段检查类型安全，但在运行时移除所有泛型类型信息。`List<String>` 和 `List<Integer>` 在编译后都会变成原始类型 `List`（Raw Type），运行时无法区分。

**为什么这样设计？**
- 向后兼容性：Java 5 才引入泛型，为了兼容 Java 1.4 及之前的非泛型代码，JVM 层面不做泛型支持，只在编译期做类型检查。

**类型擦除导致的问题：**
1. **无法使用 `instanceof` 判断泛型类型**：`list instanceof List<String>` 编译错误
2. **无法创建泛型数组**：`new T[]` 不允许，因为运行时 T 信息丢失
3. **静态上下文中无法引用类型参数**：泛型类的静态成员不能使用类型参数
4. **桥接方法（Bridge Method）**：泛型继承时编译器生成桥接方法，可能引发反射问题
5. **重载冲突**：`List<String>` 和 `List<Integer>` 不能作为方法重载的区分

**关于类型擦除的误区**：Java 泛型并不是"完全擦除"，擦除后会在字节码中保留 Signature 属性（Signature Attribute）用于反射获取泛型信息。所以通过 `getGenericType()` 可以获取泛型类型。

## 核心原理

### 类型擦除机制

- Java 5 引入泛型，为了兼容 1.4 及以前的代码，JVM 层面不提供泛型支持
- 编译时将泛型类型参数替换为它们的上界（Bounded Type）或 `Object`
- 必要时插入强制类型转换（Cast）

```java
// 编译时
List<String> list = new ArrayList<>();
// 运行时（擦除后）
List list = new ArrayList();
```

### Signature 属性

类型擦除并非"完全擦除"，字节码中保留 **Signature 属性**：
- 存储在类文件的属性表中
- 可以通过 `Class.getGenericSuperclass()`、`Field.getGenericType()` 等反射 API 获取
- 使得 Spring、Jackson 等框架可以在运行时解析泛型信息

```java
// 利用匿名内部类保留泛型信息
Type type = new TypeToken<List<String>>() {}.getType();
// Java 匿名内部类在编译时会生成新的类，Signature 属性保留了泛型参数
```

## 类型擦除导致的问题

| 问题 | 说明 | 示例 |
|------|------|------|
| 无法 `instanceof` | 运行时泛型类型未知 | `list instanceof List<String>` ❌ |
| 无法创建泛型数组 | 数组需在运行时知道类型 | `new T[10]` ❌ |
| 静态成员不可用类型参数 | 静态成员属于类，类在运行时没有具体泛型 | `static T field` ❌ |
| 桥接方法（Bridge Method） | 泛型继承时编译器生成，反射可能混淆 | 方法签名不一致 |
| 重载冲突 | 擦除后方法签名相同 | `List<String>` 和 `List<Integer>` 不能做方法参数区分 |

### 桥接方法详解

```java
class Parent<T> {
    void method(T t) {}
}
class Child extends Parent<String> {
    @Override
    void method(String s) {}
}
// 编译器自动生成桥接方法：
// void method(Object o) { method((String) o); } — 保证多态性
```

## 面试要点

- Java 泛型是**编译期**机制，不是运行期机制
- 类型擦除不等于"完全不保留类型信息" — Signature 属性保留了泛型元数据
- 常用绕过方式：匿名内部类、`TypeReference`、`TypeToken`（Guava/Gson）
- JDK 8 的 `TypeVariable` API 可以在运行期间接获取泛型信息

## 参考链接

- [[快手电商-一面-19题总结]] — Q1 Java 泛型类型擦除
- [[Java集合-ArrayList与LinkedList底层原理]] — 集合框架中的泛型应用

