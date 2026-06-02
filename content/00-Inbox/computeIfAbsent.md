当然，这个 `computeIfAbsent` 是 Java 8 引入的 `Map` 接口方法，我给您详细讲解一下。

> compute if absent 计算是否缺失

## `computeIfAbsent` 的作用

这行代码：

```java
detailsByTabName.computeIfAbsent(detail.getTabName(), k -> new ArrayList<>()).add(detail);
```

相当于以下三行代码的**简化写法**：

```java
// 如果 key 不存在，先创建一个空列表放进去
if (!detailsByTabName.containsKey(detail.getTabName())) {
    detailsByTabName.put(detail.getTabName(), new ArrayList<>());
}
// 获取这个列表，然后把当前明细加进去
detailsByTabName.get(detail.getTabName()).add(detail);
```

## 它的执行逻辑

`computeIfAbsent(key, mappingFunction)` 方法：

1. **先检查 Map 中是否已有这个 key**
   - **有** → 直接返回该 key 对应的 value，不做任何额外操作
   - **没有** → 执行第二个参数（`k -> new ArrayList<>()`），把返回值作为新 value 放入 Map，然后返回这个新 value
2. **链式调用 `.add(detail)`** → 往返回的列表中添加当前明细

所以整体效果就是：**按表名分组**——每张表对应一个 List，如果该表还没列表就新建一个，然后把当前明细行加进去。

## 用在这里的原因

`getTabDetailsByNames` 一次性查出了**所有表的所有明细行**，返回的是一个扁平的 `List<DevTabD>`。我们需要把这些明细按 `tabName` 重新分组，才能分别挂到对应的 `DevTabH` 头表对象上。

例如数据库返回了：

```
TAB_NAME   | FIE_NAME
-----------+---------
EMP        | ID
EMP        | NAME
DEPT       | ID
DEPT       | CODE
DEPT       | NAME
```

经过 `computeIfAbsent` 分组后，得到：

```
{
  "EMP"  → [ID, NAME],
  "DEPT" → [ID, CODE, NAME]
}
```

## 之前为什么没用过

之前的代码是**逐表查询**的——`getTabByName(String)` 每次只查一张表，直接拿到单个 `DevTabH` 和对应的 `List<DevTabD>`，不需要手动分组。而新的批量查询一次性拿回所有数据，就需要在 Java 内存中做这个分组组装工作。