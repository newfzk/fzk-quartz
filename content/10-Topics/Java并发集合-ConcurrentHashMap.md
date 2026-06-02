---
title: ConcurrentHashMap 原理与实现
date: 2026-05-26
tags:
  - topic/并发控制
  - topic/Java集合
  - language/java
status: to-review
updated: 2026-05-26 15:30:00
aliases:
  - ConcurrentHashMap
  - CHM原理
  - Java并发容器
related:
  - "[[CAS-Compare-And-Swap]]"
  - "[[synchronized机制详解]]"
  - "[[锁机制实现详解]]"
  - "[[乐观锁]]"
  - "[[volatile关键字详解]]"
  - "[[transient关键字详解]]"
  - "[[Java关键字-transient与volatile]]"
  - "[[红黑树面试常考知识点]]"
  - "[[Java并发集合-CopyOnWriteArrayList]]"
---

# ConcurrentHashMap 原理与实现

## 核心原理

`ConcurrentHashMap` 是 Java 并发包提供的线程安全哈希表实现，JDK 1.8 后采用 **CAS + [[synchronized机制详解|synchronized]]** 机制实现细粒度锁。

### 数据结构

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

### 锁机制

| 操作类型 | 锁策略               | 性能特点         |
| ---- | ----------------- | ------------ |
| 读操作  | 无锁（volatile可见性）   | O(1)，无阻塞     |
| 写操作  | synchronized 锁定桶头 | 只锁单个桶，不影响其他桶 |
| 扩容   | CAS + 分段迁移        | 并发扩容，不阻塞读写   |

## 核心实现

### 初始化与扩容

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

### 写入操作

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

### 读操作（无锁）

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

## 链表与红黑树的转换机制

### 转换阈值

| 条件 | 动作 | 原因 |
|------|------|------|
| 桶中节点 ≥ 8 **且** table.length ≥ 64 | 链表 → 红黑树（树化） | 树化以提升查找性能 O(n)→O(log n) |
| 桶中节点 ≥ 8 **但** table.length < 64 | 优先扩容（resize） | 扩容比树化更优，能从根本上降低冲突 |
| 桶中节点 ≤ 6 | 红黑树 → 链表（退化） | 退化以节省红黑树维护开销 |

### 为什么树化阈值是 8？

```java
/**
 * 理想随机哈希下，桶中节点数量的泊松分布概率（负载因子 0.75）：
 * 0:    0.60653066
 * 1:    0.30326533
 * 2:    0.07581633
 * 3:    0.01263606
 * 4:    0.00157952
 * 5:    0.00015795
 * 6:    0.00001316
 * 7:    0.00000094
 * 8:    0.00000006     ← 亿分之六，概率极低
 * 更多：小于千万分之一
 */
```

- **选择 8**：在良好哈希下链表长度超过 8 的概率不到亿分之一，8 是安全阈值
- **选择 6**：退化阈值留出 2 的缓冲区间（6 ↔ 8），避免节点在阈值附近震荡时频繁树化/退化

详细红黑树知识见：[[红黑树面试常考知识点]]

### 树化过程

```
putVal 中判断树化：
┌──────────────────┐
│  binCount ≥ 8?   │──否──→ 继续使用链表
└────────┬─────────┘
         │ 是
         ↓
┌──────────────────────┐
│  table.length < 64?  │──是──→ 扩容（resize）
└────────┬─────────────┘        让节点分散到更多桶
         │ 否
         ↓
┌──────────────────────┐
│  treeifyBin(tab, i)  │
│  ① 遍历链表，构建 TreeNode 双向链表
│  ② 用 TreeNode 构建红黑树
│  ③ 用 TreeBin 封装树，替换桶头
└──────────────────────┘
```

### ConcurrentHashMap 中的 TreeBin

`ConcurrentHashMap` 没有直接用 `TreeNode` 作为桶头，而是用 `TreeBin` 封装了一层：

```java
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;      // 红黑树根节点
    volatile TreeNode<K,V> first; // 保留了原始链表顺序
    volatile Thread waiter;  // 等待写锁的线程
    volatile int lockState;  // 锁状态：0=无锁，1=写锁，2=等待
}
```

**关键设计**：

1. **读写分离**：`first` 保留了原始链表引用，读操作走链表遍历（无需等待树旋转）；写操作操作树结构
2. **轻量级锁**：`lockState` 实现写写互斥，但读读不互斥，读操作完全无锁
3. **find() 实现**：读方法遍历 `first` 链表而非树，不受树结构调整影响

```java
// TreeBin.find() 读操作走链表，无需加锁
final Node<K,V> find(int h, Object k) {
    for (Node<K,V> e = first; e != null; ) {
        if (e.hash == h && ((ek = e.key) == k || (ek != null && k.equals(ek))))
            return e;
        e = e.next;
    }
    return null;
}
```

## 适用场景

| 场景 | 推荐度 | 说明 |
|------|--------|------|
| 高并发读写缓存 | ✅ 强烈推荐 | 细粒度锁，读无锁 |
| 读多写少的配置缓存 | ✅ 推荐 | 写操作只锁单个桶 |
| 频繁遍历的场景 | ⚠️ 谨慎 | 遍历时不保证快照一致性 |

```java
// 遍历时返回快照，避免 ConcurrentModificationException
public ArrayList<Product> getProductList() {
    return new ArrayList<>(cache.values());
}
```

## 相关链接

- [[Java并发集合-CopyOnWriteArrayList]] - 写时复制列表，另一种并发容器
- [[CAS-Compare-And-Swap]] - CAS 机制详解
- [[synchronized机制详解]] - synchronized 同步机制
- [[锁机制实现详解]] - 锁机制原理
- [[乐观锁]] - 乐观并发控制方式
- [[红黑树面试常考知识点]] - 红黑树详细考点
- [[Java并发集合-ConcurrentHashMap与CopyOnWriteArrayList]] - 并发容器对比与选型
