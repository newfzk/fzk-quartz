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

> [!tip] **回答要点**
>
> 详细解析见下方折叠块 👇

> [!info]- 🔹 底层原理详解
> **ArrayList（动态数组）**
> - 继承 `AbstractList`，底层是 `Object[]` 数组
> - 默认初始容量 10，**懒加载**（JDK 8+ 首次 add 才创建数组）
> - **扩容机制**：`grow()` 方法，新容量 = 旧容量 + 旧容量 >> 1（**1.5倍**），通过 `Arrays.copyOf()` 拷贝到新数组（本质是 `System.arraycopy()` 本地方法）
> - **随机访问** `get/set`：O(1)，数组下标直接寻址
> - **尾部插入** `add(E)`：均摊 O(1)，扩容时退化 O(n)
> - **中间插入/删除**：O(n)，需要批量移动元素
> - **内存特征**：连续内存 → CPU 缓存友好（空间局部性），仅存储元素数据
>
> **LinkedList（双向链表）**
> - 继承 `AbstractSequentialList`，底层是 Node 内部类：
>   ```java
>   private static class Node<E> {
>       E item;
>       Node<E> next;  // 后继指针
>       Node<E> prev;  // 前驱指针
>   }
>   ```
> - 维护 `first` 和 `last` 两个头尾指针
> - **随机访问** `get(i)`：O(n)，二分遍历以优化（`i < size/2` 从头找，否则从尾找）
> - **头尾插入/删除**：O(1)，只需修改指针
> - **中间插入/删除**：O(n) 查找位置 + O(1) 修改指针
> - **内存特征**：离散内存 → CPU 缓存不友好，每个节点多 24~32 字节引用开销
> - 实现了 **Deque** 接口，可当作队列/双端队列/栈使用

> [!question]- 🔹 面试官追问的 4 个坑
> 1. **"ArrayList 插入真的比 LinkedList 慢吗？"**
>    → **不一定！** 中间插入时 LinkedList 查找位置已经是 O(n)，且节点离散分配导致 CPU cache miss 率高。数据量大时 **ArrayList 的批量移动可能比 LinkedList 的逐个寻址更快**。实测：10 万级数据中间插入，ArrayList 完胜。
>
> 2. **"什么时候用 LinkedList？"**
>    → **只在头尾操作频繁且不需要随机访问时**。典型如队列（`offer/poll`）、双端队列、LRU 缓存雏形。日常开发 **90% 的场景用 ArrayList** 就够了。
>
> 3. **"ArrayList 扩容怎么避免性能损耗？"**
>    → **预估容量**。构造时指定初始大小：`new ArrayList<>(expectedSize)`。例：已知要存 1000 条数据就 `new ArrayList<>(1000)`，避免多次扩容。
>
> 4. **"LinkedList 能当作队列/栈用吗？"**
>    → 可以，它实现了 `Deque` 接口，`addFirst/addLast/removeFirst/removeLast/push/pop/offer/poll` 全是 O(1)。但**推荐使用 `ArrayDeque`**，它用循环数组实现，内存更紧凑、性能更好。

> [!summary]- 🔹 性能对比一览
> | 操作 | ArrayList | LinkedList |
> |------|-----------|------------|
> | `get(i)` 随机访问 | **O(1)** ✅ 数组下标直取 | O(n) ❌ 需遍历 |
> | `add(E)` 尾部插入 | **O(1)** 均摊（扩容时 O(n)） | O(1) ✅ |
> | `add(i, E)` 中间插入 | O(n) 移动元素 | O(n) 查找位置 + O(1) 改指针 |
> | `remove(i)` 删除 | O(n) 移动元素 | O(n) 查找位置 + O(1) 改指针 |
> | 内存占用 | **低** ✅ 仅存元素数据 | 高 ❌ 额外 2 引用/节点 |
> | CPU 缓存 | **友好** ✅ 连续内存 | 不友好 ❌ 离散内存 |
>
> **📌 一句话结论**：95% 的场景选 ArrayList，只有明确需要频繁头尾操作时才考虑 LinkedList。

> [!example]- 🔹 常用场景代码示例
> **场景 1：随机读取排行榜 → ArrayList**
> ```java
> List<Score> leaderboard = new ArrayList<>();
> leaderboard.add(new Score("Alice", 98));
> leaderboard.add(new Score("Bob", 95));
> // 查第 1 名 → O(1)
> Score top1 = leaderboard.get(0);
> ```
>
> **场景 2：消息队列（FIFO）→ LinkedList**
> ```java
> // 适合：频繁头部取出 + 尾部放入
> Queue<RobotCommand> cmdQueue = new LinkedList<>();
> cmdQueue.offer(new Command("前进"));   // 尾部入队 O(1)
> cmdQueue.offer(new Command("左转"));
> Command cmd = cmdQueue.poll();         // 头部出队 O(1)
> ```
>
> **场景 3：栈结构 → LinkedList / ArrayDeque**
> ```java
> Deque<String> stack = new LinkedList<>();
> stack.push("A");  // 头部压入
> stack.push("B");
> String top = stack.pop();  // 头部弹出 → "B"
> ```
>
> **场景 4：遍历全部元素 → ArrayList 更快**
> ```java
> // for-each 遍历时 ArrayList 利用 CPU 预读机制
> // 连续内存 → 效率是 LinkedList 的 2~5 倍
> List<Integer> list = new ArrayList<>();
> for (int v : list) { /* 缓存友好，速度更快 */ }
> ```

> 🔗 **相关知识**：[[Java集合-ArrayList与LinkedList底层原理]]

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