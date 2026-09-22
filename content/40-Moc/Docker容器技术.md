---
title: Docker 容器技术（MOC）
date: 2026-07-01
tags:
  - topic/Docker
  - topic/容器
  - topic/MOC
status: to-review
aliases:
  - Docker 知识地图
  - Docker MOC
---

# Docker 容器技术 — 知识地图

> [!abstract] 本笔记为 MOC（Map of Content）
> Docker 容器技术从**镜像构建**到**网络通信**到**运行时兼容性**，形成一条完整的技术栈。本 MOC 汇总相关的原子笔记。

## Dockerfile 指令

| 主题 | 核心笔记 |
|:-----|:---------|
| **Exec 形式 vs Shell 形式** | [[Docker-Exec形式与Shell形式]] |
| **CMD 指令** | [[Dockerfile-CMD指令]] |
| **ENTRYPOINT 指令** | [[Dockerfile-ENTRYPOINT指令]] |

> [!tip] 核心公式
> 容器最终执行的命令 = `ENTRYPOINT`（固定可执行文件） + `CMD`（默认参数，可被 `docker run` 覆盖）
>
> 详见 [[Dockerfile-ENTRYPOINT指令#CMD 配合模式]]。

## 网络

| 主题 | 核心笔记 |
|:-----|:---------|
| **Bridge 网络模式** | [[Docker网络模式-bridge]] |
| **iptables 端口映射** | [[iptables端口转发]]（跨笔记关联） |

## 运行时与兼容性

| 主题 | 核心笔记 |
|:-----|:---------|
| **cgroup v2 兼容性** | [[Docker-cgroup-v2-兼容性问题]] |
| **iptables 模式切换导致链缺失** | [[案例-Docker-iptables模式切换导致链缺失]] |
| **Alpine 镜像不含 tzdata** | [[Alpine镜像不自带tzdata]] |
| **TZ 环境变量与时区数据** | [[TZ环境变量只是指针-时区数据才是本体]] |

> [!warning] 运行时资源缺失是「静默故障」的高发区
> 镜像里少一个包（如 `tzdata`、`ca-certificates`）通常**不导致启动失败**，而是让时区静默退回 UTC、TLS 静默握手失败。
> 对策是把隐式依赖显式化，并用构建期断言（`[ -f ... ]`）把问题提前到构建阶段暴露。

## 镜像分析与诊断

| 主题 | 核心笔记 |
|:-----|:---------|
| **docker history 是审计日志不是文件清单** | [[docker-history是审计日志不是文件清单]] |
| **包数据库与文件系统是两套真相** | [[apk数据库与文件系统是两套真相]] |
| **ENOENT 无法区分层级** | [[容器文件消失排查-ENOENT无法区分层级]] |

> [!tip] 排查「镜像里到底有没有某文件」的手段可信度
> 容器内 `ls`/`mount` ≈ `docker export` + `tar -t` **>** `.RootFS.Layers` diff_id 比对 **>** 包数据库 **>** `apk audit` **>** **`docker history`（最易误导）**

## 相关原子笔记

```dataview
TABLE
  file.tags as "标签"
FROM "10-Topics"
WHERE file.name IN ["Docker-Exec形式与Shell形式", "Dockerfile-CMD指令", "Dockerfile-ENTRYPOINT指令", "Docker网络模式-bridge", "Docker-cgroup-v2-兼容性问题", "案例-Docker-iptables模式切换导致链缺失", "Alpine镜像不自带tzdata", "TZ环境变量只是指针-时区数据才是本体", "docker-history是审计日志不是文件清单", "apk数据库与文件系统是两套真相", "容器文件消失排查-ENOENT无法区分层级", "K8s卷挂载是覆盖而非合并"]
SORT file.name ASC
```

## 外部关联

- [[OCI-协议规范]] — OCI 开放容器标准规范（Docker 镜像格式和运行时的底层标准）
- [[Linux-cgroup-控制组]] — Docker 容器资源限制的底层机制（cgroup v1/v2）
- [[Linux-IP转发与路由]] — Docker 网络依赖 IP 转发
- [[iptables详解]] — Docker 端口映射和 NAT 的核心实现
- [[Linux网络数据包处理]] — netfilter 框架与 Docker 网络的关系
- [[K8s卷挂载是覆盖而非合并]] — 容器文件系统被卷遮蔽的机制（与镜像层的关系）
- [[案例-K8s容器时区异常-PVC挂载遮蔽usr-share]] — 镜像缺包 vs 挂载遮蔽的经典对照案例
- [[K8s容器运维]] — K8s 视角的容器运行时知识地图
