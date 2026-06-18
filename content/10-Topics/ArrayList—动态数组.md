---
title: ArrayList — 动态数组
tags:
  - language/java
  - topic/java/集合
status: to-review
---

### ArrayList — 动态数组

```java
// 简化示意
public class ArrayList<E> {
    transient Object[] elementData;  // 存放元素的数组
    private int size;                // 实际元素个数
    
    // 扩容：新容量 = 旧容量 + 旧容量 >> 1（1.5倍，[[Java-移位运算符|移位运算符]]详解）
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
}
```

**关键特性：**
- **初始容量**：默认 10（JDK 8+ 懒加载，首次 add 才创建数组）
- **扩容因子**：1.5 倍，`old + (old >> 1)`，位运算高效
- **扩容代价**：O(n) 数组拷贝，可通过预分配容量避免
- **数据结构**：连续内存空间 → CPU 缓存友好（空间局部性）