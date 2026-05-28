---
title: Java线程池与内存管理
date: 2026-05-26
tags:
  - topic/线程池
  - topic/内存管理
  - topic/并发控制
  - language/java
status: to-review
updated: 2026-05-26 15:30:00
aliases:
  - ThreadPool
  - 线程池
  - 内存管理
related:
  - "[[Java并发集合-ConcurrentHashMap]]"
  - "[[Java并发集合-CopyOnWriteArrayList]]"
  - "[[锁机制实现详解]]"
  - "[[CAS-Compare-And-Swap]]"
---

# Java线程池与内存管理

## 一、线程池核心概念

### 1.1 什么是线程池

线程池是一种**线程复用机制**，通过预先创建一定数量的线程，避免频繁创建和销毁线程的开销，提高系统性能和稳定性。

#### 线程池的优势

| 特性 | 说明 |
|------|------|
| **降低开销** | 避免线程创建/销毁的频繁开销 |
| **提高响应** | 任务到达时无需等待线程创建 |
| **可控并发** | 限制最大并发数，防止资源耗尽 |
| **便于管理** | 统一管理线程生命周期和任务队列 |

### 1.2 ThreadPoolExecutor 核心参数

```java
public ThreadPoolExecutor(
    int corePoolSize,        // 核心线程数
    int maximumPoolSize,     // 最大线程数
    long keepAliveTime,      // 空闲线程存活时间
    TimeUnit unit,           // 时间单位
    BlockingQueue<Runnable> workQueue,  // 工作队列
    ThreadFactory threadFactory,         // 线程工厂
    RejectedExecutionHandler handler     // 拒绝策略
)
```

#### 参数详解

```
线程池工作流程：
┌────────────────────────────────────────────────────────────────────┐
│                      提交任务                                      │
│                          ↓                                        │
│              核心线程数是否已满？                                   │
│              ├── 否 → 创建新核心线程执行任务                        │
│              └── 是 → 工作队列是否已满？                           │
│                        ├── 否 → 将任务加入队列                      │
│                        └── 是 → 线程数是否达到最大？                │
│                                  ├── 否 → 创建非核心线程执行任务     │
│                                  └── 是 → 执行拒绝策略              │
└────────────────────────────────────────────────────────────────────┘
```

#### 工作队列类型

| 队列类型 | 特点 | 风险 |
|---------|------|------|
| `ArrayBlockingQueue` | 有界队列，固定容量 | 队列满时触发拒绝策略 |
| `LinkedBlockingQueue` | 默认无界（Integer.MAX_VALUE） | 可能导致OOM |
| `SynchronousQueue` | 直接移交，无缓冲 | 需要足够的最大线程数 |
| `PriorityBlockingQueue` | 优先级队列 | 无界，可能OOM |

#### 拒绝策略

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| `AbortPolicy` | 抛出异常（默认） | 需要快速失败的场景 |
| `CallerRunsPolicy` | 调用者线程执行 | 非核心任务，允许降级 |
| `DiscardPolicy` | 静默丢弃 | 不重要的任务 |
| `DiscardOldestPolicy` | 丢弃最旧任务 | 实时性要求高的任务 |

---

## 二、线程池创建方式

### 2.1 不推荐的方式（存在风险）

```java
// ❌ 危险：newCachedThreadPool 最大线程数为 Integer.MAX_VALUE
// 可能创建大量线程导致 OOM
ExecutorService cachedPool = Executors.newCachedThreadPool();

// ❌ 危险：newFixedThreadPool 使用无界队列
// 任务过多时队列无限增长导致 OOM
ExecutorService fixedPool = Executors.newFixedThreadPool(10);

// ❌ 危险：newSingleThreadExecutor 使用无界队列
ExecutorService singlePool = Executors.newSingleThreadExecutor();
```

### 2.2 推荐的方式（手动配置）

```java
// ✅ 推荐：手动配置线程池参数
ExecutorService executor = new ThreadPoolExecutor(
    Runtime.getRuntime().availableProcessors() * 2,  // 核心线程数
    Runtime.getRuntime().availableProcessors() * 4,  // 最大线程数
    60L,                                            // 空闲存活时间
    TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000),                // 有界队列（关键！）
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.CallerRunsPolicy()        // 拒绝策略
);
```

### 2.3 线程数配置原则

```
配置策略：
┌────────────────────────────────────────────────────┐
│              任务类型                               │
├────────────────────────────────────────────────────┤
│ CPU密集型 → corePoolSize = CPU核心数 + 1           │
│ I/O密集型 → corePoolSize = CPU核心数 * 2           │
│ 混合任务 → 根据实际压测结果调整                      │
└────────────────────────────────────────────────────┘
```

---

## 三、批量计算门店销售实战

### 3.1 需求分析

**场景**：每日闭店后批量计算数千家门店的销售额

**挑战**：
- 数据量大（数千家门店 × 大量订单）
- 内存限制（避免一次性加载所有数据）
- 性能要求（在规定时间内完成计算）

### 3.2 解决方案：分批次并行处理

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

public class StoreSalesCalculator {
    private final ExecutorService executor;
    private final SalesRepository salesRepository;
    
    public StoreSalesCalculator(SalesRepository salesRepository) {
        this.salesRepository = salesRepository;
        int coreThreads = Runtime.getRuntime().availableProcessors() * 2;
        
        this.executor = new ThreadPoolExecutor(
            coreThreads,
            coreThreads * 2,
            60L,
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(100),  // 有界队列防止OOM
            new ThreadFactory() {
                private int counter = 0;
                @Override
                public Thread newThread(Runnable r) {
                    Thread t = new Thread(r, "sales-calculator-" + counter++);
                    t.setDaemon(true);
                    return t;
                }
            },
            new ThreadPoolExecutor.CallerRunsPolicy()  // 调用者执行，避免任务丢失
        );
    }
    
    // 批量计算所有门店销售额
    public List<StoreSalesResult> calculateAllStores(List<Long> storeIds) throws InterruptedException {
        int batchSize = 100;  // 每批处理100家门店
        List<Future<StoreSalesResult>> futures = new ArrayList<>();
        
        // 分批次提交任务
        for (int i = 0; i < storeIds.size(); i += batchSize) {
            int end = Math.min(i + batchSize, storeIds.size());
            List<Long> batch = storeIds.subList(i, end);
            
            futures.add(executor.submit(() -> calculateBatch(batch)));
        }
        
        // 收集结果
        List<StoreSalesResult> results = new ArrayList<>();
        for (Future<StoreSalesResult> future : futures) {
            try {
                results.add(future.get(5, TimeUnit.MINUTES));  // 设置超时
            } catch (ExecutionException | TimeoutException e) {
                // 处理异常
                e.printStackTrace();
            }
        }
        
        return results;
    }
    
    // 计算单个批次的销售额（流式处理，避免OOM）
    private StoreSalesResult calculateBatch(List<Long> storeIds) {
        StoreSalesResult result = new StoreSalesResult();
        
        for (Long storeId : storeIds) {
            // 分页查询订单，流式处理
            long totalSales = 0;
            int pageNum = 0;
            int pageSize = 1000;  // 每页1000条订单
            
            while (true) {
                List<Order> orders = salesRepository.findOrdersByStoreId(storeId, pageNum, pageSize);
                
                if (orders.isEmpty()) {
                    break;
                }
                
                // 边读边累加，不保留全部数据
                for (Order order : orders) {
                    totalSales += order.getAmount();
                }
                
                pageNum++;
            }
            
            result.addStoreSales(storeId, totalSales);
        }
        
        return result;
    }
    
    public void shutdown() {
        executor.shutdown();
    }
}
```

### 3.3 使用 CompletableFuture 实现更灵活的并行计算

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.stream.Collectors;

public class CompletableFutureSalesCalculator {
    private final ExecutorService executor;
    private final SalesRepository salesRepository;
    
    public CompletableFutureSalesCalculator(SalesRepository salesRepository) {
        this.salesRepository = salesRepository;
        this.executor = Executors.newFixedThreadPool(
            Runtime.getRuntime().availableProcessors() * 2
        );
    }
    
    public CompletableFuture<List<StoreSales>> calculateAsync(List<Long> storeIds) {
        // 并行处理每个门店
        List<CompletableFuture<StoreSales>> futures = storeIds.stream()
            .map(storeId -> calculateStoreSalesAsync(storeId))
            .collect(Collectors.toList());
        
        // 等待所有任务完成并合并结果
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList())
            );
    }
    
    private CompletableFuture<StoreSales> calculateStoreSalesAsync(Long storeId) {
        return CompletableFuture.supplyAsync(() -> {
            long total = 0;
            int page = 0;
            
            // 流式分页查询，避免一次性加载
            while (true) {
                List<Order> orders = salesRepository.findOrdersByStoreId(storeId, page, 500);
                if (orders.isEmpty()) break;
                
                for (Order order : orders) {
                    total += order.getAmount();
                }
                page++;
            }
            
            return new StoreSales(storeId, total);
        }, executor);
    }
}
```

---

## 四、内存溢出（OOM）预防策略

### 4.1 OOM 常见原因

| 类型 | 原因 | 症状 |
|------|------|------|
| **堆溢出** | 对象无法回收，内存耗尽 | `OutOfMemoryError: Java heap space` |
| **栈溢出** | 递归过深或线程过多 | `OutOfMemoryError: unable to create new native thread` |
| **元空间溢出** | 类加载过多 | `OutOfMemoryError: Metaspace` |

### 4.2 线程池相关的 OOM 预防

```java
// ✅ 正确配置示例
public class SafeThreadPoolConfig {
    public static ExecutorService createSafeThreadPool() {
        int corePoolSize = Runtime.getRuntime().availableProcessors() * 2;
        int maxPoolSize = corePoolSize * 2;
        
        return new ThreadPoolExecutor(
            corePoolSize,
            maxPoolSize,
            60L,
            TimeUnit.SECONDS,
            // 关键：使用有界队列
            new LinkedBlockingQueue<>(1000),
            Executors.defaultThreadFactory(),
            // 关键：使用合理的拒绝策略
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
}
```

### 4.3 数据库查询的内存优化

```java
// ✅ 使用流式查询避免一次性加载
public class StreamQueryExample {
    // 使用 JDBC 流式查询
    public void processOrdersStream(Long storeId) throws SQLException {
        String sql = "SELECT id, amount FROM orders WHERE store_id = ?";
        
        try (Connection conn = getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql, 
                 ResultSet.TYPE_FORWARD_ONLY, 
                 ResultSet.CONCUR_READ_ONLY)) {
            
            stmt.setLong(1, storeId);
            stmt.setFetchSize(Integer.MIN_VALUE);  // 启用流式获取
            
            try (ResultSet rs = stmt.executeQuery()) {
                long total = 0;
                while (rs.next()) {
                    total += rs.getLong("amount");
                    // 每处理一定数量后释放一次资源
                    if (total % 10000 == 0) {
                        System.gc();  // 提示GC
                    }
                }
            }
        }
    }
    
    // 使用 Spring Data JPA 分页查询
    public long calculateSalesWithPagination(Long storeId) {
        long total = 0;
        int pageNum = 0;
        int pageSize = 1000;
        
        while (true) {
            Page<Order> page = orderRepository.findByStoreId(
                storeId, 
                PageRequest.of(pageNum, pageSize)
            );
            
            if (page.isEmpty()) {
                break;
            }
            
            // 边处理边累加
            total += page.getContent().stream()
                .mapToLong(Order::getAmount)
                .sum();
            
            pageNum++;
        }
        
        return total;
    }
}
```

### 4.4 JVM 参数配置建议

```bash
# 生产环境推荐配置
java -Xms4g \           # 初始堆大小
     -Xmx8g \           # 最大堆大小（根据服务器配置调整）
     -XX:MetaspaceSize=256m \
     -XX:MaxMetaspaceSize=512m \
     -XX:+UseG1GC \     # 使用 G1 垃圾收集器
     -XX:MaxGCPauseMillis=200 \  # 最大GC停顿时间
     -XX:+HeapDumpOnOutOfMemoryError \  # OOM时自动生成堆转储
     -XX:HeapDumpPath=/tmp/heapdump.hprof \
     -jar application.jar
```

---

## 五、分治法（Map-Reduce）并行模式

### 5.1 模式概述

```
分治法流程：
┌──────────────────────────────────────────────────────────────┐
│                        Map 阶段                             │
│  将大任务分解为多个子任务，并行执行                           │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                    │
│  │Task1 │  │Task2 │  │Task3 │  │Task4 │  ...               │
│  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘                    │
│     ↓         ↓         ↓         ↓                         │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                    │
│  │Res1  │  │Res2  │  │Res3  │  │Res4  │  ...               │
│  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘                    │
│                        │                                    │
│                        ↓                                    │
│                   Reduce 阶段                               │
│              将子结果合并为最终结果                           │
│                   ↓                                         │
│              ┌──────────┐                                   │
│              │ Final    │                                   │
│              │ Result   │                                   │
│              └──────────┘                                   │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 瑞幸门店销售计算示例

```java
import java.util.List;
import java.util.concurrent.*;
import java.util.stream.Collectors;

public class MapReduceSalesCalculator {
    private final ExecutorService executor;
    
    public MapReduceSalesCalculator() {
        this.executor = Executors.newFixedThreadPool(
            Runtime.getRuntime().availableProcessors() * 2
        );
    }
    
    // Map 阶段：并行计算各门店销售额
    private List<CompletableFuture<Long>> map(List<Long> storeIds) {
        return storeIds.stream()
            .map(storeId -> CompletableFuture.supplyAsync(
                () -> calculateSingleStore(storeId),
                executor
            ))
            .collect(Collectors.toList());
    }
    
    // Reduce 阶段：汇总所有门店销售额
    private long reduce(List<CompletableFuture<Long>> futures) {
        return futures.stream()
            .mapToLong(f -> {
                try {
                    return f.get();
                } catch (InterruptedException | ExecutionException e) {
                    throw new RuntimeException(e);
                }
            })
            .sum();
    }
    
    // 完整的 Map-Reduce 流程
    public long calculateTotalSales(List<Long> storeIds) {
        List<CompletableFuture<Long>> futures = map(storeIds);
        return reduce(futures);
    }
    
    private long calculateSingleStore(Long storeId) {
        // 分页查询该门店订单并累加
        long total = 0;
        int page = 0;
        
        while (true) {
            List<Order> orders = orderRepository.findByStoreId(storeId, page, 500);
            if (orders.isEmpty()) break;
            
            total += orders.stream()
                .mapToLong(Order::getAmount)
                .sum();
            
            page++;
        }
        
        return total;
    }
}
```

---

## 六、线程池监控与调优

### 6.1 监控指标

```java
import java.util.concurrent.*;

public class ThreadPoolMonitor {
    private final ThreadPoolExecutor executor;
    
    public ThreadPoolMonitor(ThreadPoolExecutor executor) {
        this.executor = executor;
    }
    
    public void logStatus() {
        System.out.println("=== 线程池状态 ===");
        System.out.println("核心线程数: " + executor.getCorePoolSize());
        System.out.println("最大线程数: " + executor.getMaximumPoolSize());
        System.out.println("活跃线程数: " + executor.getActiveCount());
        System.out.println("池中线程数: " + executor.getPoolSize());
        System.out.println("队列任务数: " + executor.getQueue().size());
        System.out.println("已完成任务数: " + executor.getCompletedTaskCount());
        System.out.println("总任务数: " + executor.getTaskCount());
        System.out.println("=================");
    }
}
```

### 6.2 动态调整线程池参数

```java
public class DynamicThreadPool {
    private final ThreadPoolExecutor executor;
    
    public DynamicThreadPool() {
        this.executor = new ThreadPoolExecutor(
            4, 8, 60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(1000)
        );
    }
    
    // 根据负载动态调整核心线程数
    public void adjustCorePoolSize(int newSize) {
        executor.setCorePoolSize(newSize);
    }
    
    // 根据队列长度动态调整
    public void adjustBasedOnQueue() {
        int queueSize = executor.getQueue().size();
        
        if (queueSize > 800) {
            // 队列积压严重，增加线程数
            executor.setCorePoolSize(Math.min(
                executor.getMaximumPoolSize(),
                executor.getCorePoolSize() + 2
            ));
        } else if (queueSize < 200 && executor.getCorePoolSize() > 4) {
            // 队列空闲，减少线程数
            executor.setCorePoolSize(executor.getCorePoolSize() - 1);
        }
    }
}
```

---

## 七、选型建议与最佳实践

### 7.1 线程池选型指南

| 场景 | 推荐配置 | 说明 |
|------|---------|------|
| CPU密集型计算 | 核心线程数 = CPU核心数 + 1 | 减少线程切换开销 |
| I/O密集型操作 | 核心线程数 = CPU核心数 × 2 | 充分利用CPU等待时间 |
| 批量数据处理 | 有界队列 + CallerRunsPolicy | 防止OOM，允许降级 |
| 实时任务处理 | SynchronousQueue + 足够最大线程数 | 低延迟，快速响应 |

### 7.2 防OOM Checklist

- [ ] 使用有界队列（设置合理容量）
- [ ] 避免使用 `Executors.newCachedThreadPool()`
- [ ] 数据库查询使用分页/流式读取
- [ ] 及时释放不再使用的对象引用
- [ ] 设置合理的JVM堆内存大小
- [ ] 配置OOM自动堆转储

### 7.3 瑞幸场景最佳实践

| 场景 | 推荐方案 |
|------|---------|
| 每日批量计算 | 固定大小线程池 + 分页查询 + Map-Reduce |
| 实时订单处理 | 有界队列线程池 + CallerRunsPolicy |
| 异步任务处理 | CompletableFuture + 自定义线程池 |

---

## 八、总结

### 核心要点

1. **线程池配置**：手动配置优于使用 `Executors` 工厂方法，关键是使用有界队列和合理的拒绝策略。

2. **内存管理**：避免一次性加载大量数据，使用分页查询和流式处理，边读边处理。

3. **并行模式**：分治法（Map-Reduce）适合批量数据处理，主线程等待所有子任务完成后合并结果。

4. **监控调优**：定期监控线程池状态，根据实际负载动态调整参数。

### 关键代码模式

```java
// 核心模式：有界队列 + 合理拒绝策略
ExecutorService executor = new ThreadPoolExecutor(
    corePoolSize,
    maxPoolSize,
    keepAliveTime,
    TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(capacity),  // 有界队列
    threadFactory,
    new ThreadPoolExecutor.CallerRunsPolicy()  // 降级策略
);
```

---

## 参考链接

- [[Java并发集合-ConcurrentHashMap]] - ConcurrentHashMap 原理与实现
- [[Java并发集合-CopyOnWriteArrayList]] - CopyOnWriteArrayList 原理与实现
- [[Java并发集合-ConcurrentHashMap与CopyOnWriteArrayList]] - 并发容器对比与选型
- [[锁机制实现详解]] - 锁机制原理
- [[CAS-Compare-And-Swap]] - CAS机制详解
