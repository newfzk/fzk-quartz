---
title: Ant 风格 — 路径通配符匹配模式
date: 2026-06-08
tags:
  - topic/Spring
  - topic/Java基础
  - topic/文件路径
  - language/java
status: reviewed
aliases:
  - Ant风格
  - Ant path pattern
  - 路径通配符
related:
  - "[[Spring-Resource资源抽象]]"
---

# Ant 风格路径匹配

**Ant 风格**源自 **Apache Ant**（一个 Java 构建工具）的路径通配符匹配模式，后被 Spring、Maven、Gradle 等广泛采用，成为 Java 生态中路径匹配的事实标准。

## 通配符规则

| 符号 | 含义 |
|------|------|
| `?` | 匹配文件名或目录名中的**单个字符** |
| `*` | 匹配文件名或目录名中的**零个或多个字符**（不跨越目录层级） |
| `**` | 匹配**零层或多层目录**（跨越目录层级） |

## 具体示例

假设目录结构：

```
src/
├── main/
│   ├── java/
│   │   └── com/demo/
│   │       ├── UserController.java
│   │       ├── UserService.java
│   │       └── UserMapper.java
│   └── resources/
│       └── application.yml
└── test/
    └── java/com/demo/
        └── UserServiceTest.java
```

| 模式 | 匹配结果 |
|------|---------|
| `**/*.java` | 所有层级的 `.java` 文件 |
| `src/main/**/*.java` | 仅 `main` 下的 Java 文件 |
| `**/*Service*.java` | 含 "Service" 的文件 |
| `src/main/java/com/demo/User*.java` | 该目录下 User 开头的文件 |
| `src/**/demo/*.java` | demo 目录下直接子文件 |

## 三个核心区别

```
*    不跨层    —  仅匹配当前目录层级下的文件名/目录名
**   跨层      —  可以跨越任意多级子目录
?    单字符    —  精确匹配一个字符
```

```java
"com/**/*.java"    // 匹配 com/ 及所有子目录下的 .java 文件
"com/*/*.java"     // 只匹配 com/ 下正好一层子目录里的 .java 文件
"com/*/Test?.java" // 匹配 com/下一层中名为 TestX.java 的文件（X为单字符）
```

## 与正则表达式的对比

| 概念 | Ant 风格 | 正则表达式 |
|------|---------|-----------|
| 任意字符（不跨 `/`） | `*` | `[^/]*` |
| 任意字符（跨 `/`） | `**` | `.*` |
| 单个字符 | `?` | `.` |

Ant 风格更直观，无需反斜杠转义，专门适合描述文件系统路径。

## Spring 中的应用

```java
// classpath*: 扫描所有模块
String scanPath = "classpath*:" + applicationName + "/dictionary/*.json";
Resource[] resources = resolver.getResources(scanPath);

// classpath: 仅扫描第一个匹配的 classpath 根
Resource[] resources = resolver.getResources("classpath:META-INF/spring.factories");
```

## 参考链接

- [[Spring-Resource资源抽象]] — Spring Resource 体系与 Ant 通配符的配合使用