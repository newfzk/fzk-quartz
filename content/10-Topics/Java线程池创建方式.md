---
title: Java 线程池创建方式
date: 2026-06-01
tags:
  - topic/并发控制
  - topic/线程池
  - language/java
aliases:
  - Executors
  - 线程池创建
  - ThreadPoolExecutor创建
  - 线程数配置
related:
  - "[[ThreadPoolExecutor核心参数]]"
  - "[[线程池拒绝策略]]"
  - "[[进程线程协程]]"
  - "[[Java虚拟线程]]"
  - "[[Runtime.availableProcessors]]"
---

# Java 线程池创建方式

> [!abstract]
> 创建线程池有**两种方式**：使用 `Executors` 工厂方法（不推荐）和手动 `new ThreadPoolExecutor`（推荐）。

## 不推荐的方式：Executors 工厂方法

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

### 为什么危险？

| 工厂方法 | 风险 |
|---------|------|
| `newCachedThreadPool()` | `maximumPoolSize = Integer.MAX_VALUE`，高并发下创建**数百万**线程 |
| `newFixedThreadPool(n)` | 使用 `LinkedBlockingQueue`（无界），队列**无限增长** |
| `newSingleThreadExecutor()` | 同上，使用无界队列 |
| `newScheduledThreadPool(n)` | `maximumPoolSize = Integer.MAX_VALUE` |

> [!danger] 《阿里巴巴 Java 开发手册》强制规定
> **线程池不允许使用 Executors 创建**，而是通过 `ThreadPoolExecutor` 手动配置。这样让开发者更明确线程池的运行规则，避免资源耗尽风险。

## 推荐的方式：手动创建 ThreadPoolExecutor

```java
// ✅ 推荐：手动配置线程池参数
int processors = Runtime.getRuntime().availableProcessors();
ExecutorService executor = new ThreadPoolExecutor(
    processors * 2,                // 核心线程数
    processors * 4,                // 最大线程数
    60L,                           // 空闲存活时间
    TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000),  // 有界队列（关键！）
    r -> {
        Thread t = new Thread(r, "custom-pool-" + System.currentTimeMillis());
        t.setDaemon(false);
        return t;
    },                             // 自定义线程工厂
    new ThreadPoolExecutor.CallerRunsPolicy()  // 拒绝策略（见 [[线程池拒绝策略]]）
);
```

> [!danger] ⚠️ K8s 环境陷阱：`availableProcessors()` 可能不是容器限定的 CPU 数
>
> `Runtime.getRuntime().availableProcessors()` 返回的是 **JVM 可见的 CPU 核心数**（详见 [[Runtime.availableProcessors]]），在容器化环境中存在版本依赖问题：
>
> | JDK 版本 | 行为 | 后果 |
> |---------|------|------|
> | **JDK 8u191 之前** | 返回 **宿主机节点**的总 CPU 数 | 若节点 32 核，容器 limit 2 核，仍返回 32 → **线程数过大，过度争抢** |
> | **JDK 8u191+ / JDK 10+** | 返回容器 cgroup 限定的 CPU 数 ✅ | 正确识别 `limits.cpu`，但需确保未禁用 `-XX:-UseContainerSupport` |
>
> ```bash
> # 显式指定 CPU 数（最稳妥，适用于所有 JDK 版本）
> java -XX:ActiveProcessorCount=2 -jar app.jar
>
> # 或在代码中通过环境变量覆盖
> int processors = Integer.parseInt(
>     System.getenv().getOrDefault("CPU_LIMIT", 
>         String.valueOf(Runtime.getRuntime().availableProcessors()))
> );
> ```
>
> **推荐做法**：在 K8s Deployment 中通过 `env` 注入 `limits.cpu`，代码读取环境变量，彻底规避 JDK 版本差异。

### 关键配置要点

1. **有界队列**：设置合理的队列容量，防止 OOM
2. **合理拒绝策略**：`CallerRunsPolicy` 提供天然降级
3. **命名线程工厂**：方便排查问题（`jstack` 时看到有意义的名字）
4. **不要使用 `Executors.defaultThreadFactory()`**：默认线程名无意义

## 线程数配置原则

```
CPU密集型 → corePoolSize = CPU核心数 + 1
I/O密集型 → corePoolSize = CPU核心数 * 2
混合任务  → 根据实际压测结果调整
```

> [!tip] 为什么 CPU 密集型设置为核心数 +1？
> +1 是为了**补偿页缺失**等导致线程暂停的情况，保证 CPU 饱和时仍有线程可运行。

> [!warning] 容器环境核心数获取
> 上述公式依赖**正确的 CPU 核心数**。在 K8s 中，务必确认 JDK 版本 **≥ 8u191**（或 JDK 10+），`UseContainerSupport` 默认开启，JVM 才能正确读取 `limits.cpu`。低版本 JDK 会读到宿主机总核数导致线程池过大。
>
> 保险做法：通过环境变量注入 CPU 限额，见上方危险 callout 中的代码示例。

## 实战：批量门店销售计算

以下是一个实际业务场景 —— 闭店后批量计算数千家门店销售额，展示线程池的正确使用方式。

### 方案：分批次并行处理 + 流式查询

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
            r -> new Thread(r, "sales-calc-" + System.currentTimeMillis()),
            new ThreadPoolExecutor.CallerRunsPolicy()  // 降级
        );
    }

    // 批量计算所有门店销售额
    public List<StoreSalesResult> calculateAllStores(List<Long> storeIds) throws InterruptedException {
        int batchSize = 100;  // 每批处理100家门店
        List<Future<StoreSalesResult>> futures = new ArrayList<>();

        for (int i = 0; i < storeIds.size(); i += batchSize) {
            int end = Math.min(i + batchSize, storeIds.size());
            List<Long> batch = storeIds.subList(i, end);
            futures.add(executor.submit(() -> calculateBatch(batch)));
        }

        // 收集结果（设置超时）
        List<StoreSalesResult> results = new ArrayList<>();
        for (Future<StoreSalesResult> future : futures) {
            try {
                results.add(future.get(5, TimeUnit.MINUTES));
            } catch (ExecutionException | TimeoutException e) {
                log.error("批次计算超时或失败", e);
                // 可以记录失败的批次，后续重试
            }
        }
        return results;
    }

    // 计算单个批次（流式处理，避免OOM）
    private StoreSalesResult calculateBatch(List<Long> storeIds) {
        StoreSalesResult result = new StoreSalesResult();
        for (Long storeId : storeIds) {
            result.addStoreSales(storeId, calculateSingleStore(storeId));
        }
        return result;
    }

    private long calculateSingleStore(Long storeId) {
        long total = 0;
        int page = 0;
        while (true) {
            List<Order> orders = salesRepository.findOrdersByStoreId(storeId, page++, 1000);
            if (orders.isEmpty()) break;
            for (Order order : orders) {
                total += order.getAmount();
            }
        }
        return total;
    }

    public void shutdown() {
        executor.shutdown();
    }
}
```

> [!note] 设计要点
> - **线程池参数**：根据 CPU 核心数动态配置（见上方配置原则）
> - **有界队列**：`LinkedBlockingQueue<>(100)` 防止任务堆积导致 OOM
> - **超时机制**：`future.get(5, TimeUnit.MINUTES)` 防止单批次卡死
> - **流式查询**：分页查询订单，边读边累加，不保留全部数据

### 使用 CompletableFuture 实现更灵活的并行

```java
public class CompletableFutureSalesCalculator {
    private final ExecutorService executor;

    public CompletableFutureSalesCalculator() {
        this.executor = new ThreadPoolExecutor(
            Runtime.getRuntime().availableProcessors() * 2,
            Runtime.getRuntime().availableProcessors() * 4,
            60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(500),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }

    public CompletableFuture<List<StoreSales>> calculateAsync(List<Long> storeIds) {
        List<CompletableFuture<StoreSales>> futures = storeIds.stream()
            .map(storeId -> CompletableFuture.supplyAsync(
                () -> calculateSingleStore(storeId), executor))
            .collect(Collectors.toList());

        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList()));
    }

    private StoreSales calculateSingleStore(Long storeId) { /* 同上 */ }
}
```

> [!seealso] 相关笔记
> - [[ThreadPoolExecutor核心参数]] — 七大参数详解
> - [[线程池拒绝策略]] — 四种拒绝策略
> - [[Java虚拟线程]] — JDK 21 虚拟线程：无需池化的新选择
> - [[JVM内存溢出预防]] — 线程池相关的 OOM 预防
> - [[MapReduce并行模式]] — 分治法的另一种视角
