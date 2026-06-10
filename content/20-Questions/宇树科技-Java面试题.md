---
title: 宇树科技 Java 面试题
date: 2026-06-04
tags:
  - 面试
  - java
  - 机器人
  - 低延迟
  - status/to-review
type: question
---

# 宇树科技 Java 面试题

## 一、Java 核心基础

### 1. Java 集合核心原理

- ArrayList 和 LinkedList 底层原理与适用场景
- HashMap 底层实现、扩容机制与哈希冲突解决
- ConcurrentHashMap 1.7 和 1.8 的主要区别

> 相关笔记：[[Java并发集合-ConcurrentHashMap与CopyOnWriteArrayList]]、[[Java并发集合-ConcurrentHashMap]]、[[Java并发集合-CopyOnWriteArrayList]]、[[Java-Map-computeIfAbsent]]

### 2. Java 引用类型与 ThreadLocal

- Java 四种引用类型（强、软、弱、虚）的特点和使用场景
- ThreadLocal 为什么会出现内存泄漏，如何避免

> 相关笔记：[[Java线程池与内存管理]]、[[JVM内存溢出预防]]

### 3. Java IO 与 NIO 核心区别

- 传统 IO 和 NIO 的核心区别
- NIO 三大组件 Channel、Buffer、Selector 的作用

### 4. JVM 基础与垃圾回收

- JVM 类加载流程
- JVM 内存模型划分
- 常见垃圾回收算法
- G1 收集器的核心特点

> 相关笔记：[[JVM-堆内存分代模型]]、[[JVM-OOM排查指南]]、[[JVM内存溢出预防]]、[[jstat-命令详解]]

### 5. 线程池与并发基础

- 线程池七大核心参数
- 常见线程池类型及适用场景
- CAS 原理，ABA 问题如何解决

> 相关笔记：[[ThreadPoolExecutor核心参数]]、[[线程池拒绝策略]]、[[线程池监控与调优]]、[[Java线程池创建方式]]、[[CAS-Compare-And-Swap]]

---

## 二、Spring 框架

### 6. Spring IOC / AOP 原理

- Spring IOC 和 AOP 核心原理
- Spring Bean 的生命周期阶段

### 7. Spring 事务与失效场景

- Spring 事务的传播行为和隔离级别
- 日常开发中事务失效的常见场景

> 相关笔记：[[事务传播行为]]、[[隔离级别]]、[[事务ACID]]

---

## 三、数据库与缓存

### 8. MySQL 索引与事务隔离

- MySQL 索引为什么使用 B+ 树而不是 B 树
- 事务四大隔离级别分别解决了什么并发问题

> 相关笔记：[[MySQL索引类型]]、[[MySQL索引创建原则]]、[[MySQL联合索引]]、[[主键索引与唯一索引的区别]]、[[隔离级别]]、[[脏读-Dirty-Read]]、[[不可重复读-Non-repeatable-Read]]、[[幻读-Phantom-Read]]

### 9. Redis 缓存核心问题

- 缓存穿透、缓存击穿、缓存雪崩的概念和解决方案

---

## 四、计算机网络

### 10. TCP 与 HTTP

- TCP 三次握手和四次挥手流程
- HTTP 和 HTTPS 的核心区别
- TCP 粘包拆包如何解决

---

## 五、宇树科技深度题（机器人场景）

### 11. JVM 低延迟 GC 调优

- 机器人实时控制这类低延迟业务系统，JVM 如何调优
- ZGC 相比 G1 的优势

> 相关笔记：[[Java-启动参数]]、[[JVM-堆内存分代模型]]

### 12. AQS 与锁机制深度

- AQS 底层原理
- synchronized 和 ReentrantLock 的核心区别
- 如何检测与避免死锁

> 相关笔记：[[synchronized机制详解]]、[[锁机制实现详解]]、[[Java读写锁-ReadWriteLock]]、[[乐观锁]]、[[悲观锁]]

### 13. 消息队列可靠性保障（RocketMQ）

- 如何确保消息不丢失、不重复投递
- 消费者节点宕机后的消息重平衡

> 相关笔记：[[接口幂等方案设计]]

### 14. Redis 分布式锁实现

- 基于 Redis 实现分布式锁
- 锁超时、误删锁、锁续约等关键问题

> 相关笔记：[[分布式锁实现]]

### 15. 接口幂等性设计

- 机器人指令不能重复执行，如何设计接口幂等性
- 常用实现方案

> 相关笔记：[[接口幂等方案设计]]、[[状态机模式实现]]

### 16. 微服务限流熔断降级

- 限流、熔断、降级的实现
- Sentinel 的核心原理

### 17. 机器人软硬件通信可靠性

- 断连、丢包、超时场景下保证指令不丢失、不重复、不乱序

### 18. MySQL 主从复制与读写分离

- 数据库读写分离架构设计
- 主从延迟问题，实时数据查询一致性保证

> 相关笔记：[[MySQL Binlog 日志配置]]、[[Mysql常用配置]]

### 19. 机器人实时状态监控系统设计

- 低延迟、高可靠、数据不丢失的设计要点

### 20. 机器人 OTA 升级系统设计

- 断点续传、灰度发布、故障回滚
- 避免升级失败导致设备异常