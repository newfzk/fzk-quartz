---
title: Lombok @Data 与 boolean 类型 getter 命名
date: 2026-06-08
aliases:
  - Lombok is方法
  - boolean getter命名
  - Data注解
tags:
  - language/java
  - topic/java/基础
status: to-review
---

# Lombok @Data 与 boolean 类型 getter 命名

`@Data` 是 Lombok 的组合注解，等价于 `@Getter` + `@Setter` + `@ToString` + `@EqualsAndHashCode` + `@RequiredArgsConstructor`，在**编译期**自动生成方法。

## boolean 类型的特殊处理

对于基本类型 `boolean` 字段，Lombok 遵循 **JavaBean 规范**，生成的 getter 方法名是 `isXxx()` 而不是 `getXxx()`：

```java
// @Data 在编译期自动生成如下方法：

public boolean isAllowDropTable() {   // boolean 类型遵循 isXxx() 命名
    return this.allowDropTable;
}

public void setAllowDropTable(boolean allowDropTable) {
    this.allowDropTable = allowDropTable;
}
```

### 注意

- 仅对**基本类型 `boolean`** 生效（不是包装类 `Boolean`）
- 如果字段名本身以 `is` 开头（如 `isActive`），生成的 getter 仍然是 `isActive()`（不会变成 `isIsActive()`）
- 包装类 `Boolean` 使用标准 `getXxx()` 命名