---
title: Spring Boot — java.io.tmpdir 临时目录配置
date: 2026-06-08
aliases:
  - java.io.tmpdir
  - Tomcat tempDir
  - Tomcat临时目录
related:
  - "[[Java-启动参数]]"
tags:
  - language/java
  - topic/spring-boot
  - topic/故障排查
status: to-review
---

# Spring Boot — java.io.tmpdir 临时目录配置

## 问题

Spring Boot 启动时报错：

```
Unable to create tempDir. java.io.tmpdir is set to C:\Windows\
Caused by: java.nio.file.AccessDeniedException: C:\Windows\tomcat.xxx
```

**原因**：`java.io.tmpdir` 被设置为 `C:\Windows\`，当前用户没有写权限。内嵌 Tomcat 需要临时目录来解压依赖和处理上传文件。

**根因**：运行环境缺少 `TMP`/`TEMP` 环境变量，JVM 回退到默认值 `C:\Windows\`。

## 修复方案

### 方案 A：JVM 参数（最快修复）

```bash
java -Djava.io.tmpdir="C:\Users\admin\AppData\Local\Temp" -jar app.jar
```

### 方案 B：application.yml 配置（代码级修复）

```yaml
server:
  tomcat:
    basedir: ./tmp  # 项目目录下创建 tmp 作为临时目录
```

### 方案 C：修复环境变量（全局修复）

```bash
set TMP=C:\Users\admin\AppData\Local\Temp
set TEMP=C:\Users\admin\AppData\Local\Temp
```

## 选型建议

| 方案 | 适用场景 |
|------|---------|
| A | 临时快速修复，不修改代码 |
| B | 团队级修复，对所有开发环境生效 |
| C | 系统级修复，影响所有 Java 应用 |

## 参考链接

- [[Java-启动参数]] — JVM 启动参数详解