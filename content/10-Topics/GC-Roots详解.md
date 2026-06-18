---
title: GC Roots 详解
date: 2026-06-15
aliases:
  - GC Root
  - 可达性分析
  - 类卸载
  - 静态变量生命周期
related:
  - "[[JVM-堆内存分代模型]]"
  - "[[JVM-OOM排查指南]]"
tags:
  - language/java
  - topic/JVM
status: reviewed
---

# GC Roots 详解

> GC Roots 是 Java 垃圾回收中**可达性分析（Reachability Analysis）** 的起点，从这些根节点向下搜索引用链，不可达的对象被判定为可回收。

## GC Roots 分类

| GC Root 类型            | 说明              | 示例                                   |
| --------------------- | --------------- | ------------------------------------ |
| **虚拟机栈引用**            | 栈帧中的局部变量表       | 方法中的局部对象引用                           |
| **方法区静态属性**           | 类的静态变量          | `private static Object obj`          |
| **方法区常量引用**           | 常量池中的引用         | 字符串常量引用、final 常量                     |
| **JNI 引用**            | 本地方法栈中的引用       | [[JNI详解\|Native 方法引用]]的对象           |
| **JVM 内部引用**          | 系统类加载器、Class 对象 | `ClassLoader.getSystemClassLoader()` |
| **synchronized 持有对象** | 被用作同步锁的对象       | `synchronized(obj){}`                |
| **JMXBean / JVMTI**   | 管理相关的引用         | MBeanServer 注册的对象                    |

[[JNI详解]]

## 局部变量为什么能作为 GC Root

- 局部变量存储在**虚拟机栈**的栈帧中
- 只要方法正在执行，该栈帧就处于"活跃"状态
- GC 从栈帧中的局部变量开始遍历对象引用图
- 当前方法可达的所有对象都必须存活
- **即使方法还没执行完的赋值语句**：
  ```java
  void method() {
      Object o = new Object(); // GC 开始时，o 还未初始化到 null
      // GC roots 不会包含未初始化的局部变量槽
  }
  ```

> HotSpot 使用 **OopMap** 记录栈帧中哪些位置是对象引用，精确引导 GC 遍历，而不是扫描栈上所有数据。

## 静态变量生命周期

### 创建时机

- 类加载的**准备阶段**分配内存并设置默认值
- **初始化阶段**执行 `<clinit>` 方法赋初始值
- 静态变量本身存储在**方法区（元空间）**

### 移除时机

静态变量作为 GC Root 的生命周期和类绑定：
- 静态变量指向的对象在类被卸载前不会被回收
- **类卸载条件（需同时满足）**：
  1. 该类的所有实例都已被回收
  2. 加载该类的 `ClassLoader` 已被回收
  3. 该类的 `java.lang.Class` 对象没有任何引用

### 典型场景

```java
// 静态变量引用的对象不会被 GC，直到类被卸载
class Holder {
    static Object CACHE = new Object();
    // CACHE 指向的对象一直存活，除非：
    // 1. CACHE = null 显式断开
    // 2. Holder 类被卸载
}
```

### 常见误区

- **静态变量不会像局部变量那样自动消失**，必须显式置 null 或类被卸载才能断开引用
- 这也是内存泄漏的常见来源：静态集合类长期持有对象引用
- **Tomcat 热部署**：每个 Web App 使用自己的 ClassLoader，重启时丢老 ClassLoader 触发类卸载

## 面试要点

- GC Roots 是**所有 GC 算法的前提**（只要是需要回收堆的语言/运行时都类似）
- 局部变量是 GC Root 但不是永久的 — 方法退出后栈帧出栈就不再是 Root
- 静态变量的引用生存期 = 类生命周期，容易导致内存泄漏
- **[[JNI详解#Global Reference|JNI Global Reference]]** 也是 GC Root，且要求手动释放（不释放导致泄漏）

## 参考链接

- [[快手电商-一面-19题总结]] — Q3 GC Roots
- [[JVM-堆内存分代模型]] — GC 回收与分代
- [[JVM-OOM排查指南]] — OOM 排查
- [[JVM-GC类型对比]] — 常见 GC 类型与回收器对比
