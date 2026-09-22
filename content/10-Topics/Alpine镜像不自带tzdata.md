---
title: Alpine镜像不自带tzdata
date: 2026-09-22
updated: 2026-09-22
aliases:
  - Alpine tzdata
  - 裸 alpine 镜像
  - musl 时区
related:
  - "[[TZ环境变量只是指针-时区数据才是本体]]"
  - "[[案例-K8s容器时区异常-PVC挂载遮蔽usr-share]]"
  - "[[Docker容器技术]]"
tags:
  - topic/Docker
  - topic/Linux
  - topic/时区
status: to-review
---

# Alpine 镜像不自带 tzdata

## 核心认知

> [!important] `Alpine ≠ 有时区数据`
> `tzdata` 是一个**独立的可选软件包**，裸 `alpine:3.x` 镜像**不包含**它。

裸 alpine 镜像的 `/usr/share/` 下只有三个目录：

```
apk/  man/  misc/
```

**没有 `zoneinfo`。** 因此 `TZ=Asia/Shanghai` 在这种镜像里必然静默失效（见 [[TZ环境变量只是指针-时区数据才是本体]]）。

## 为什么基于 Alpine 的官方镜像却正常

关键区别：**官方镜像不是"裸 alpine"**，其 Dockerfile 里显式安装了运行期依赖。

以 `eclipse-temurin:21.0.7_6-jdk-alpine-3.20` 为例，其 Dockerfile 中有：

```dockerfile
RUN apk add --no-cache fontconfig ttf-dejavu gnupg ca-certificates \
    p11-kit-trust musl-locales musl-locales-lang binutils tzdata coreutils openssl
```

| | 官方 Temurin 镜像 | 自建裸 alpine 镜像 |
|---|---|---|
| 基于 | alpine 3.20 | alpine 3.8.2 |
| `tzdata` 包 | **已安装** | 没有 |
| `/usr/share/zoneinfo` | 存在 | **不存在** |
| `date` 结果 | CST ✅ | UTC ❌ |

> [!tip] 排查启示
> "同样是 Alpine，为什么它正常？"——答案往往不在基础镜像的名字，而在**该镜像的 Dockerfile 装了哪些包**。遇到这类矛盾，优先对比两个镜像的 `apk` 包列表，而不是怀疑构建缓存或 registry。

## 隐式依赖的脆弱性

基于"基础镜像自带 tzdata"来写业务 Dockerfile，是一种**隐式依赖**：

```dockerfile
# ⚠️ 脆弱写法：依赖基础镜像恰好装了 tzdata
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
ENV TZ=Asia/Shanghai
```

问题在于：**`ln -sf` 在源文件不存在时不报错**，照样创建一条**悬空软链**。于是：

- 构建成功 ✅
- 镜像推送成功 ✅
- Pod 正常启动 ✅
- 时区静默错误 ❌（无人发现，直到日志时间戳对不上）

一旦将来基础镜像升级、tag 被替换、换了更"精简"的镜像，问题就突然爆发，且很难定位。

## 正确写法：显式依赖 + 构建期断言

```dockerfile
RUN apk add --no-cache tzdata \
 && ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
 && echo "Asia/Shanghai" > /etc/timezone \
 && [ -f /usr/share/zoneinfo/Asia/Shanghai ] \
 && [ -f /etc/localtime ] \
 && TZ=Asia/Shanghai date
ENV TZ=Asia/Shanghai
```

要点：

| 片段 | 作用 |
|---|---|
| `apk add --no-cache tzdata` | 把**隐式依赖变显式**（即使当前基础镜像自带，此命令近似 no-op，但能防止将来重演） |
| `ln -snf` 的 `-n` | 避免目标已是目录时，软链被建到目录**内部** |
| `[ -f ... ]` 断言 | `test -f` **跟随软链**，悬空即失败 → **让问题在构建阶段暴露** |
| `TZ=Asia/Shanghai date` | 在构建日志打印真实时间，**肉眼可核** |

## 其它常见基础镜像的 tzdata 情况

| 基础镜像 | 是否自带 tzdata |
|---|---|
| `alpine`（裸） | ❌ 需手动装 |
| `eclipse-temurin:*-alpine` | ✅ 官方已装 |
| `debian` / `ubuntu` | ✅ 通常自带 |
| `distroless` | ❌ 刻意极简，需自行处理 |
| `busybox` | ❌ 需手动装 |
| 大多数官方语言运行时镜像（*-slim） | ⚠️ 不一，需实测 |

> [!warning] 不要靠记忆判断
> 最可靠的方法是在容器里直接查：`ls /usr/share/zoneinfo` 或 `apk info -e tzdata`。把断言写进 Dockerfile 才能真正免于记忆。

## 参考链接

- [[TZ环境变量只是指针-时区数据才是本体]] — 缺 tzdata 时的静默失效机制
- [[案例-K8s容器时区异常-PVC挂载遮蔽usr-share]] — 缺包与遮蔽两种成因的对比案例
- [[Docker容器技术]] — 镜像构建相关知识地图
