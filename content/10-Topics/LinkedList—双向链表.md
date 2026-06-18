---
tags:
  - language/java
  - topic/java/集合
status: to-review
---

### LinkedList — 双向链表

```java
// 简化示意
public class LinkedList<E> {
    private static class Node<E> {
        E item;        // 当前元素
        Node<E> next;  // 后继指针
        Node<E> prev;  // 前驱指针
    }
    
    transient Node<E> first;  // 头节点
    transient Node<E> last;   // 尾节点
    private int size;
}
```

**关键特性：**
- **非连续内存**：节点分散在堆中 → CPU 缓存不友好
- **额外开销**：每个节点多 24~32 字节（prev + next 引用）
- **双向遍历**：查找时二分优化，`index < size/2` 从头找，否则从尾找
- **双端操作**：头尾增删均为 O(1)，实现了 `Deque` 接口