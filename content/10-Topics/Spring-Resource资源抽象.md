---
title: Spring Resource 资源抽象
date: 2026-06-08
tags:
  - topic/Spring
  - topic/Spring-Boot
  - language/java
status: reviewed
aliases:
  - ResourceLoader
  - ResourcePatternResolver
  - Spring资源加载
related:
  - "[[Ant风格-路径通配符匹配模式]]"
---

# Spring Resource 资源抽象

`org.springframework.core.io.Resource` 是 Spring 对底层资源的统一抽象接口，屏蔽了文件系统、classpath、URL、ServletContext 等不同来源的差异。

## 一、Resource 接口

核心方法：

| 方法 | 作用 |
|------|------|
| `exists()` | 资源是否存在 |
| `getInputStream()` | 打开输入流 |
| `getFile()` | 获取 `java.io.File` 句柄 |
| `getURI()` / `getURL()` | 获取 URI/URL 标识 |
| `getFilename()` | 获取文件名 |
| `contentLength()` / `lastModified()` | 元信息 |

常用实现类：

| 实现类 | 来源 | 前缀 |
|--------|------|------|
| `ClassPathResource` | classpath | `classpath:` |
| `FileSystemResource` | 文件系统 | `file:` |
| `UrlResource` | 任意 URL | `http:` `ftp:` |
| `ServletContextResource` | Web 应用根目录 | 无 |
| `ByteArrayResource` | 内存字节数组 | 无 |
| `InputStreamResource` | 输入流（一次性） | 无 |

## 二、ResourceLoader 接口

核心方法：`Resource getResource(String location)`，根据前缀决定返回哪种 `Resource` 实现。

Spring 中 `ApplicationContext` 实现了 `ResourceLoader`，可直接注入使用：

```java
@Autowired
private ResourceLoader resourceLoader;  // 或直接注入 ApplicationContext

Resource r1 = resourceLoader.getResource("classpath:data/config.json");
Resource r2 = resourceLoader.getResource("file:/opt/app/conf.yml");
```

## 三、ResourcePatternResolver — 批量模式匹配

`ResourceLoader` 一次只能加载一个资源。`ResourcePatternResolver` 是其子接口，支持 **Ant 风格通配符**批量扫描：

```java
public interface ResourcePatternResolver extends ResourceLoader {
    Resource[] getResources(String locationPattern) throws IOException;
}
```

| 通配符 | 含义 |
|--------|------|
| `?` | 匹配单个字符 |
| `*` | 匹配单层任意字符 |
| `**` | 匹配多层路径 |
| `classpath*:` | 扫描**所有** JAR 和 classpath 目录 |

### classpath: vs classpath*:

```java
// classpath:  — 只扫描第一个匹配的 classpath 根
resolver.getResources("classpath:META-INF/spring.factories");

// classpath*: — 扫描所有 classpath 根（所有 JAR + 目录），合并返回
resolver.getResources("classpath*:META-INF/spring.factories");
```

## 四、完整体系图

```
ResourceLoader            (接口: getResource 单个加载)
    └── ResourcePatternResolver   (子接口: getResources 批量加载 + Ant通配符)
            └── PathMatchingResourcePatternResolver (默认实现)

ApplicationContext  实现了 ResourcePatternResolver

Resource                  (接口: 统一资源抽象)
    ├── ClassPathResource
    ├── FileSystemResource
    ├── UrlResource
    ├── ServletContextResource
    └── ByteArrayResource / InputStreamResource
```

## 常见用法

```java
// 读取资源内容
Resource res = resourceLoader.getResource("classpath:sql/init.sql");
String sql = StreamUtils.copyToString(res.getInputStream(), StandardCharsets.UTF_8);

// 批量扫描 MyBatis Mapper XML
Resource[] mapperLocations = resolver.getResources("classpath*:mapper/**/*.xml");

// 条件判断资源是否存在
Resource res = resourceLoader.getResource("classpath:banner.txt");
if (res.exists()) { /* 自定义启动 banner */ }
```

## 参考链接

- [[Ant风格-路径通配符匹配模式]] — Ant 风格通配符详解