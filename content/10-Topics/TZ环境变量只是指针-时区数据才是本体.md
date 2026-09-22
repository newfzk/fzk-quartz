---
title: TZ环境变量只是指针-时区数据才是本体
date: 2026-09-22
updated: 2026-09-22
aliases:
  - TZ 环境变量
  - 容器时区原理
  - etc-localtime
  - zoneinfo
related:
  - "[[案例-K8s容器时区异常-PVC挂载遮蔽usr-share]]"
  - "[[Alpine镜像不自带tzdata]]"
  - "[[K8s卷挂载是覆盖而非合并]]"
tags:
  - topic/Linux
  - topic/时区
  - topic/容器
status: to-review
---

# TZ 环境变量只是指针，时区数据才是本体

## 核心机制

**`TZ=Asia/Shanghai` 只是一个字符串指针，不是时区数据本身。**

libc 拿到这个值后的流程：

```
读到 TZ=Asia/Shanghai
      ↓
去读 /usr/share/zoneinfo/Asia/Shanghai   ← 真正的时区规则文件
      ↓
┌─ 找到 → 按上海时区正确格式化
└─ 找不到 → 静默退回 UTC（不报错、不告警）⚠️
```

> [!danger] 最危险的地方：找不到时不报错
> libc 在时区文件缺失时**不会抛异常、不会打日志**，直接使用 UTC。
>
> 结果就是：程序正常运行、Pod 正常启动、`date` 输出了一个**看起来合法但错误**的时间。这是典型的**静默故障**，只能靠对比预期值才能发现。

## Alpine / musl 的具体行为

musl libc（Alpine 使用）在 `/usr/share/zoneinfo` 不存在时：

1. 会把 `Asia/Shanghai` 当作 **POSIX 格式**尝试硬解析（如 `CST-8`）
2. 解析失败
3. 直接使用 UTC

同时 `/etc/localtime` 若是**悬空软链**（指向不存在的 `/usr/share/zoneinfo/...`），只读该文件的程序同样退回 UTC。

## 两个独立的失效点

时区能否正确生效，取决于**两处同时正常**：

| 失效点 | 表现 | 检查命令 |
|---|---|---|
| `TZ` 变量本身 | 未注入、为空、含 `\r`/空格等脏字符 | `echo "TZ raw:[$TZ]"` |
| `/usr/share/zoneinfo/<TZ>` | 文件不存在（镜像缺 tzdata 或被遮蔽） | `ls -l /usr/share/zoneinfo/Asia/Shanghai` |
| `/etc/localtime` | 悬空软链 | `ls -l /etc/localtime` |

> [!tip] 快速反证技巧
> `TZ=Asia/Shanghai date` 单独执行一次：
> - **若输出 CST** → 时区数据存在，问题在环境变量注入
> - **若仍输出 UTC** → 时区数据缺失或不可见（镜像缺包 / 挂载遮蔽）

## ⚠️ OS 时区 ≠ JVM 时区

**这两个必须分别验证**，很容易出现"`date` 是 UTC 但 Java 日志是 CST"的情况：

| 层级 | 时区数据来源 | 优先级 |
|---|---|---|
| **OS（`date`）** | `/usr/share/zoneinfo` + `TZ` / `/etc/localtime` | `TZ` → `/etc/localtime` |
| **JVM（Java 日志）** | JDK 自带 `tzdb.dat`（`$JAVA_HOME/lib/tzdb.dat`） | `-Duser.timezone` > `TZ` > `/etc/timezone`、`/etc/localtime` |

**关键差异**：JDK **自带**一份独立的时区数据库。因此即使容器的 `/usr/share/zoneinfo` 被遮蔽、`date` 显示 UTC，Java 应用仍可能通过 `tzdb.dat` 拿到正确的 `Asia/Shanghai`。

> [!important] 排查含义
> - **不能因为 Java 日志时间对，就认为容器时区没问题**
> - 反过来，**Java 日志时间错，也不一定是 OS 时区错**，要单独查 `TimeZone.getDefault()`
> - 排查时区问题必须**分层验证**：OS 层与 JVM 层各查一遍

## 容器内时区全量体检

```bash
kubectl exec -n <ns> <pod> -- sh -c '
  echo "== date ==";   date; date -u
  echo "== TZ ==";     echo "[$TZ]"
  echo "== 目录 ==";   ls -ld /usr/share /usr/share/zoneinfo 2>&1
  echo "== 文件 ==";   ls -l /etc/localtime /usr/share/zoneinfo/Asia/Shanghai 2>&1
  echo "== 反证 ==";   TZ=Asia/Shanghai date
'
```

## 正确的 Dockerfile 写法

```dockerfile
# 显式安装 + 重建软链 + 构建期断言，让问题在构建阶段暴露
RUN apk add --no-cache tzdata \
 && ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
 && echo "Asia/Shanghai" > /etc/timezone \
 && [ -f /usr/share/zoneinfo/Asia/Shanghai ] \
 && [ -f /etc/localtime ] \
 && TZ=Asia/Shanghai date
ENV TZ=Asia/Shanghai
```

- `ln -snf` 的 **`-n`**：避免目标已是目录时，软链被建到目录**内部**
- `[ -f ... ]`：`test -f` 会**跟随软链**，悬空即返回失败 → **构建期就失败，而不是上线后由日志时间戳倒查**
- 最后一行把真实时间打印进构建日志，肉眼可核

## 检查清单

| 观察结果 | 结论 |
|---|---|
| `TZ` 变量为空 | 环境变量未注入 / Pod 是旧版本 → `rollout restart` |
| `TZ` 有值，目录/文件都在，`date` 仍 UTC | 变量脏（含 `\r`）或启动脚本覆盖了 TZ |
| `TZ` 有值，`/usr/share/zoneinfo` 不存在 | 镜像缺 `tzdata`，或被 volumeMounts 遮蔽 |
| `mount` 输出显示 `/usr/share` 被挂载 | **挂载遮蔽** → 改挂载点 |

## 参考链接

- [[案例-K8s容器时区异常-PVC挂载遮蔽usr-share]] — 本机制的完整实战案例
- [[Alpine镜像不自带tzdata]] — 时区数据为何会缺失
- [[K8s卷挂载是覆盖而非合并]] — 时区数据为何会"不可见"
