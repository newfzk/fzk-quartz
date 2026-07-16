---
title: Lombok @RequiredArgsConstructor 注解
date: 2026-06-24
tags:
  - language/java
  - topic/java/基础
aliases:
  - RequiredArgsConstructor
  - Lombok 构造器注入
  - Lombok 构造器注解
status: to-review
---

`@RequiredArgsConstructor` 是 Lombok 提供的注解，用于**自动生成包含所有 `final` 字段的构造函数**。

### 作用

使用该注解可以避免手动编写依赖注入的有参构造函数，简化代码。

### 典型用法：构造器注入

在 Spring Boot 应用中，配合 `@Service`、`@Component` 等注解使用，实现推荐的构造器注入模式：

```java
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;

    // Lombok 自动生成：
    // public UserService(UserRepository userRepository, EmailService emailService) {
    //     this.userRepository = userRepository;
    //     this.emailService = emailService;
    // }
}
```

### 与 @Autowired 对比

| 方式 | 注解 | 优点 | 缺点 |
| --- | --- | --- | --- |
| **构造器注入**（推荐） | `@RequiredArgsConstructor` + `final` 字段 | 不可变、易测试、显式依赖 | 类字段较多时构造函数长 |
| **字段注入** | `@Autowired` | 代码简洁 | 不利于测试、隐藏依赖、违反单一职责 |

Spring 官方推荐构造器注入方式，`@RequiredArgsConstructor` 可以自动化这一过程。

### 相关注解

- `@NoArgsConstructor`：生成无参构造函数
- `@AllArgsConstructor`：生成全参构造函数
- `@Data`：包含 `@Getter`、`@Setter`、`@RequiredArgsConstructor`、`@ToString`、`@EqualsAndHashCode`

相关笔记：[[Lombok-Data与boolean-getter命名]]
