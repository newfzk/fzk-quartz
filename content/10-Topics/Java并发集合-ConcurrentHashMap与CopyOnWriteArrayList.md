---
title: Java并发集合 - ConcurrentHashMap与CopyOnWriteArrayList
date: 2026-05-26
tags:
  - topic/并发控制
  - topic/Java集合
  - language/java
status: to-review
updated: 2026-05-26 15:30:00
aliases:
  - ConcurrentHashMap
  - CopyOnWriteArrayList
  - Java并发容器
related:
  - "[[CAS-Compare-And-Swap]]"
  - "[[锁机制实现详解]]"
  - "[[乐观锁]]"
---

# Java并发集合

## 一、ConcurrentHashMap

### 1.1 核心原理

`ConcurrentHashMap` 是 Java 并发包提供的线程安全哈希表实现，JDK 1.8 后采用 **CAS + synchronized** 机制实现细粒度锁。

#### 数据结构

```
ConcurrentHashMap 结构（JDK 1.8+）：
┌─────────────────────────────────────────────────────────┐
│                      Table                             │
│  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐   │
│  │  0   │  1   │  2   │  3   │  4   │ ...  │  n-1 │   │  ← Node数组（volatile）
│  └──┬───┴──┬───┴──┬───┴──┬───┴──┬───┴──────┴──┬───┘   │
│     ↓      ↓      ↓      ↓      ↓              ↓       │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐    ┌─────┐  │
│  │Node │→│Node │→│Node │→│Node │→│Node │... │Node │  │  ← 链表/红黑树
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘    └─────┘  │
└─────────────────────────────────────────────────────────┘
```

#### 锁机制

| 操作类型 | 锁策略 | 性能特点 |
|---------|--------|---------|
| 读操作 | 无锁（volatile可见性） | O(1)，无阻塞 |
| 写操作 | synchronized 锁定桶头 | 只锁单个桶，不影响其他桶 |
| 扩容 | CAS + 分段迁移 | 并发扩容，不阻塞读写 |

### 1.2 核心实现

#### 初始化与扩容

```java
public class ConcurrentHashMap<K, V> {
    // 核心数组，volatile保证可见性
    transient volatile Node<K,V>[] table;
    
    // 扩容时的新数组
    private transient volatile Node<K,V>[] nextTable;
    
    // 控制标识符：负数表示正在初始化/扩容
    private transient volatile int sizeCtl;
    
    // 初始化或扩容
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            // CAS设置为-1，表示正在初始化
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // 已有线程在初始化，让出CPU
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[16];
                        table = nt;
                        sc = 16 >>> 2; // 0.75 * capacity
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
}
```

#### 写入操作

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    
    // 计算哈希值（二次哈希减少冲突）
    int hash = spread(key.hashCode());
    int binCount = 0;
    
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        
        // 表未初始化，先初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        
        // CAS获取桶位置，如果为空则插入新节点
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;  // CAS成功，无需加锁
        }
        
        // 桶正在扩容，协助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        
        // 桶有数据，加锁写入（synchronized锁定桶头）
        else {
            V oldVal = null;
            synchronized (f) {
                // 双重检查确保桶头未变
                if (tabAt(tab, i) == f) {
                    // 链表结构
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }
                    // 红黑树结构
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            
            // 检查是否需要转为红黑树
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    
    // 更新size并检查是否需要扩容
    addCount(1L, binCount);
    return null;
}
```

### 1.3 读操作（无锁）

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    
    // 二次哈希
    int h = spread(key.hashCode());
    
    // 无锁读取
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        
        // 检查桶头是否匹配
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        
        // 桶正在扩容，从nextTable读取
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        
        // 遍历链表查找
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

### 1.4 适用场景

| 场景 | 推荐度 | 说明 |
|------|--------|------|
| 高并发读写缓存 | ✅ 强烈推荐 | 细粒度锁，读无锁 |
| 读多写少的配置缓存 | ✅ 推荐 | 写操作只锁单个桶 |
| 频繁遍历的场景 | ⚠️ 谨慎 | 遍历时不保证快照一致性 |

---

## 二、CopyOnWriteArrayList

### 2.1 核心原理

`CopyOnWriteArrayList` 采用 **写时复制** 策略，读操作不加锁，写操作复制整个数组。

#### 写时复制机制

```
CopyOnWriteArrayList 写操作流程：
┌─────────────────────────────────────────────────────────────┐
│                    初始状态                                  │
│  array = [A, B, C, D]  ← volatile引用                      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    写操作（添加元素E）                        │
│  1. 加锁                                                    │
│  2. 复制原数组：newArray = [A, B, C, D]                     │
│  3. 新数组尾部添加：newArray = [A, B, C, D, E]              │
│  4. 原子替换引用：array = newArray                          │
│  5. 解锁                                                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    读操作（无锁）                            │
│  读取的始终是旧数组的快照，不阻塞写操作                       │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 核心实现

#### 添加操作

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();  // 写操作必须加锁
    try {
        Object[] elements = getArray();
        int len = elements.length;
        
        // 复制整个数组（关键：写时复制）
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        
        // 原子替换数组引用
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

#### 读取操作

```java
public E get(int index) {
    // 读操作完全无锁
    return get(getArray(), index);
}

private E get(Object[] a, int index) {
    return (E) a[index];
}
```

#### 批量写入优化

```java
public boolean addAll(Collection<? extends E> c) {
    Object[] cs = c.toArray();
    if (cs.length == 0)
        return false;
    
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        
        // 只需复制一次，优化批量写入
        Object[] newElements = Arrays.copyOf(elements, len + cs.length);
        System.arraycopy(cs, 0, newElements, len, cs.length);
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

### 2.3 适用场景

| 场景 | 推荐度 | 说明 |
|------|--------|------|
| 配置列表（写极少，读频繁） | ✅ 强烈推荐 | 读操作零开销 |
| 事件监听器列表 | ✅ 推荐 | 遍历期间可安全修改 |
| 商品分类/标签列表 | ✅ 推荐 | 数据相对稳定 |
| 频繁写入的场景 | ❌ 不推荐 | 每次写都复制数组，性能差 |

---

## 三、并发读写问题对比

### 3.1 常见集合线程安全性

| 集合类型 | 线程安全 | 问题描述 |
|---------|---------|---------|
| `HashMap` | ❌ 不安全 | 多线程下可能死循环（JDK 1.7）、数据丢失 |
| `Hashtable` | ✅ 安全 | 方法级synchronized，性能差 |
| `Collections.synchronizedMap` | ✅ 安全 | 包装器模式，全局锁，性能差 |
| `ConcurrentHashMap` | ✅ 安全 | 细粒度锁，读无锁，高性能 |

### 3.2 HashMap 并发问题演示

```java
// 危险：多线程下HashMap可能死循环（JDK 1.7）
public class HashMapThreadIssue {
    private static HashMap<String, Integer> map = new HashMap<>();
    
    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 1000; i++) {
            final int num = i;
            new Thread(() -> {
                map.put("key" + num, num);
            }).start();
        }
        Thread.sleep(1000);
        System.out.println("Map size: " + map.size());
    }
}
```

**问题原因**：JDK 1.7 HashMap 扩容时链表反转可能形成环形链表，导致死循环。

---

## 四、缓存优化策略

### 4.1 本地缓存方案对比

| 方案 | 过期策略 | 最大容量 | 并发安全 | 适用场景 |
|------|---------|---------|---------|---------|
| ConcurrentHashMap | 手动实现 | 无限制 | ✅ | 简单缓存 |
| Caffeine | LRU/LFU/TTL | 支持 | ✅ | 高性能缓存 |
| Guava Cache | LRU | 支持 | ✅ | 通用缓存 |

### 4.2 Caffeine 缓存实现

```java
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import java.util.concurrent.TimeUnit;

public class ProductCache {
    private final Cache<String, Product> cache;
    
    public ProductCache() {
        this.cache = Caffeine.newBuilder()
            .maximumSize(1000)           // 最大缓存1000个条目
            .expireAfterWrite(10, TimeUnit.MINUTES)  // 写入后10分钟过期
            .expireAfterAccess(5, TimeUnit.MINUTES)  // 访问后5分钟过期
            .recordStats()               // 记录统计信息
            .build();
    }
    
    public Product get(String productId) {
        return cache.get(productId, this::loadProductFromDB);
    }
    
    private Product loadProductFromDB(String productId) {
        // 从数据库加载商品信息
        return productRepository.findById(productId);
    }
    
    public void invalidate(String productId) {
        cache.invalidate(productId);
    }
    
    public void invalidateAll() {
        cache.invalidateAll();
    }
}
```

### 4.3 瑞幸商品列表缓存实战

```java
import java.util.ArrayList;
import java.util.concurrent.ConcurrentHashMap;

public class ProductListCache {
    private final ConcurrentHashMap<String, Product> cache = new ConcurrentHashMap<>();
    private volatile long lastUpdateTime = 0;
    private static final long CACHE_DURATION = 5 * 60 * 1000; // 5分钟
    
    // 获取商品列表快照（线程安全）
    public ArrayList<Product> getProductList() {
        // 检查缓存是否过期
        if (System.currentTimeMillis() - lastUpdateTime > CACHE_DURATION) {
            refreshCache();
        }
        
        // 返回快照，避免迭代期间修改
        return new ArrayList<>(cache.values());
    }
    
    // 刷新缓存（门店POS和小程序共用）
    public synchronized void refreshCache() {
        // 双重检查，避免重复刷新
        if (System.currentTimeMillis() - lastUpdateTime <= CACHE_DURATION) {
            return;
        }
        
        // 从数据库加载最新商品列表
        List<Product> products = productService.getAllProducts();
        
        // 清空旧缓存
        cache.clear();
        
        // 批量写入新数据
        for (Product product : products) {
            cache.put(product.getId(), product);
        }
        
        lastUpdateTime = System.currentTimeMillis();
    }
    
    // 单个商品更新
    public void updateProduct(Product product) {
        cache.put(product.getId(), product);
    }
}
```

---

## 五、选型建议

### 5.1 ConcurrentHashMap vs CopyOnWriteArrayList

| 特性 | ConcurrentHashMap | CopyOnWriteArrayList |
|------|------------------|---------------------|
| 数据结构 | 哈希表 | 动态数组 |
| 写操作开销 | 低（仅锁单个桶） | 高（复制整个数组） |
| 读操作开销 | 低（无锁） | 低（无锁） |
| 内存占用 | 适中 | 高（写时双倍内存） |
| 迭代一致性 | 弱一致性 | 快照一致性 |

### 5.2 选择指南

```
选择策略：
┌─────────────────────────────────────────────────────────┐
│                    数据访问模式                          │
├─────────────────────────────────────────────────────────┤
│ 读多写少 + 遍历频繁 → CopyOnWriteArrayList             │
│ 读多写少 + 随机访问 → ConcurrentHashMap                │
│ 写操作频繁 → ConcurrentHashMap（优于CopyOnWrite）       │
│ 需要遍历一致性 → CopyOnWriteArrayList                  │
└─────────────────────────────────────────────────────────┘
```

### 5.3 瑞幸场景推荐

| 场景 | 推荐容器 | 说明 |
|------|---------|------|
| 商品列表缓存 | ConcurrentHashMap | 支持并发刷新，写操作频繁 |
| 配置项列表 | CopyOnWriteArrayList | 写极少，遍历频繁 |
| 订单状态缓存 | ConcurrentHashMap | 高频读写 |
| 门店设备列表 | CopyOnWriteArrayList | 设备变更少，查询频繁 |

---

## 六、总结

### 核心要点

1. **ConcurrentHashMap**：JDK 1.8+ 使用 CAS + synchronized 实现细粒度锁，读操作无锁，适合高并发读写场景。

2. **CopyOnWriteArrayList**：写时复制策略，读操作完全无锁，适合写极少、遍历频繁的场景（如配置列表）。

3. **缓存优化**：配合 Caffeine 或 Guava Cache 设置过期时间和最大容量，避免内存无限增长。

4. **并发安全**：普通 HashMap 多线程下不安全，必须使用并发容器或同步包装器。

### 最佳实践

- 商品列表优先使用 `ConcurrentHashMap` 作为缓存容器
- 配置类数据使用 `CopyOnWriteArrayList`
- 配合本地缓存框架（Caffeine）实现自动过期策略
- 遍历集合时返回快照（如 `new ArrayList<>(cache.values())`）

---

## 参考链接

- [[CAS-Compare-And-Swap]] - CAS机制详解
- [[锁机制实现详解]] - 锁机制原理
- [[乐观锁]] - 乐观并发控制方式
