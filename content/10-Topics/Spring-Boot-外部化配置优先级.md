---
title: Spring Boot 外部化配置优先级
date: 2026-06-11
aliases:
  - Spring Boot 配置优先级
  - Spring Boot 外部化配置
  - Relaxed Binding
  - 环境变量映射
tags:
  - language/java
  - topic/spring-boot
status: to-review
---

## 核心概念

Spring Boot 支持从多种来源加载配置，并定义了明确的**优先级顺序**。优先级高的配置会覆盖优先级低的配置，这一机制称为**外部化配置（Externalized Configuration）**。

## 完整优先级排序（从低到高）

| 优先级 | 配置来源 | 说明 |
|:------:|----------|------|
| 1（最低） | Jar 包内的 `application.properties` / `application.yml` | 内置默认配置 |
| 2 | Jar 包内的 Profile 配置（`application-dev.yml`） | profile 专属配置 |
| 3 | 外部 `application.properties` / `application.yml` | jar 包外同目录 |
| 4 | 外部 Profile 配置（`application-dev.yml`） | 外部 profile 配置 |
| 5 | **OS 环境变量** | 如 `SPRING_DATASOURCE_URL` |
| 6 | Java 系统属性（`-D` 参数） | `-Dspring.datasource.url=xxx` |
| 7（最高） | 命令行参数 | `--spring.datasource.url=xxx` |

> [!tip] 实际意义
> **环境变量优先级高于配置文件**。这意味着你可以在不修改任何 YAML 文件的情况下，通过环境变量覆盖所有配置。这在容器化部署（Docker/K8s）中非常有用。

## 环境变量映射规则（Relaxed Binding）

Spring Boot 的 **Relaxed Binding（宽松绑定）** 机制允许环境变量名中的 `_` 映射为 YAML 中的 `.`，且大小写不敏感：

| 环境变量 | 等价 YAML 路径 |
|---------|----------------|
| `SPRING_DATASOURCE_URL` | `spring.datasource.url` |
| `SPRING_DATASOURCE_USERNAME` | `spring.datasource.username` |
| `SPRING_DATASOURCE_PASSWORD` | `spring.datasource.password` |
| `SPRING_DATASOURCE_DRIVER_CLASS_NAME` | `spring.datasource.driver-class-name` |

### 映射规则

```
环境变量名           →    YAML 属性路径
SPRING_DATASOURCE_URL  →  spring.datasource.url

转换过程：
  1. 全部转小写:            spring_datasource_url
  2. 下划线 → 点号:         spring.datasource.url（_ 到 .）
  3. 宽松匹配属性名:         spring.datasource.url（最终匹配）
```

> 注意：环境变量中使用 `_` 而不是 `.`，因为大多数操作系统不支持在环境变量名中使用点号。

## 实战示例：纯环境变量驱动数据源

假设有 `local.env`：

```env
SPRING_DATASOURCE_URL=jdbc:mysql://127.0.0.1:3306/mydb
SPRING_DATASOURCE_USERNAME=admin
SPRING_DATASOURCE_PASSWORD=pass
SPRING_DATASOURCE_DRIVER_CLASS_NAME=com.mysql.cj.jdbc.Driver

# Nacos 配置也可用环境变量覆盖
SPRING_CLOUD_NACOS_USERNAME=nacos
SPRING_CLOUD_NACOS_PASSWORD=nacos
```

此时可以删除 YAML 中的 `spring.datasource` 块，Spring Boot 会自动从环境变量读取。

## 与 Nacos 配置中心的优先级对比

当使用 Nacos Config 时，完整的优先级链为：

```
低优先级
   │
   ├── Nacos shared-configs（rs-datasource.yaml 等共享配置）
   ├── Nacos extension-configs（扩展配置）
   ├── Nacos 主配置（服务专属配置）
   ├── 本地 bootstrap.yml（启动上下文）
   ├── 本地 application.yml / application-dev.yml
   ├── OS 环境变量（如 SPRING_DATASOURCE_URL）  ⭐ 关键覆盖点
   ├── Java 系统属性（-D 参数）
   ├── 命令行参数（--xxx=yyy）
   │
高优先级
```

> [!important] 关键结论
> **OS 环境变量优先级高于 Nacos 和本地 YAML 配置。** 这意味着即使 Nacos 中配置了 Oracle 数据源，只要设置了 `SPRING_DATASOURCE_URL` 环境变量指向 MySQL，最终生效的就是 MySQL。

## 最佳实践

| 场景 | 配置方式 | 理由 |
|------|---------|------|
| 开发环境 | `application-dev.yml` | 方便本地修改 |
| 测试环境 | 环境变量 | 不影响代码仓库中的配置 |
| 生产环境（容器） | 环境变量 / ConfigMap | 云原生方式，不依赖打包配置 |
| 生产环境（非容器） | 外部配置 + 启动参数 | 灵活且可维护 |

## 参考链接

- [[Spring-Boot自动配置-spring-factories]] — Spring Boot 自动配置原理
- [[Spring-Boot-CommandLineRunner]] — Spring Boot 启动流程
- [[Spring-Boot-java.io.tmpdir临时目录]] — Spring Boot 临时目录配置
- [[Spring-Resource资源抽象]] — Spring 资源抽象机制
