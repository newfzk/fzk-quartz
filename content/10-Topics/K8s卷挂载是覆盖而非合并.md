---
title: K8s卷挂载是覆盖而非合并
date: 2026-09-22
updated: 2026-09-22
aliases:
  - 挂载遮蔽
  - mount over
  - volumeMounts 覆盖
  - 挂载点遮蔽系统目录
related:
  - "[[案例-K8s容器时区异常-PVC挂载遮蔽usr-share]]"
  - "[[Linux-mount挂载机制]]"
  - "[[Docker容器技术]]"
tags:
  - topic/K8s
  - topic/Docker
  - topic/Linux
status: to-review
---

# K8s 卷挂载是"覆盖"而非"合并"

## 核心语义

`volumeMounts` 的语义是 **mount over（挂载覆盖）**，不是 merge：

1. **挂载点的原有内容会被整体遮蔽**，而不是合并进卷
2. **原内容仍完好存在于镜像层中**，只是在该挂载点路径下不可见
3. **卸载后原内容恢复**（删掉 volumeMount 重建 Pod 即可验证）

> [!important] 与"删除文件"的本质区别
> 两者现象高度相似（目录都显示为空），但本质完全不同：
>
> | | 删除 | 遮蔽 |
> |---|---|---|
> | 镜像层 | **被修改** | **完好无损** |
> | 包数据库（apk/dpkg） | 记录会变化 | **完全不变** |
> | 卸载卷后 | 文件不会回来 | **文件恢复** |
> | 文件系统审计 | 通常有痕迹（`D:` 记录） | **无任何痕迹** |
>
> **"包数据库记录完好 + 文件不可见"这个组合，就是遮蔽的指纹。**

## 核心原则

> [!danger] 挂载点必须是镜像里不存在的目录
> **只要挂载点目录在镜像中已有内容，就会被遮蔽。**
>
> 这一条对 **PVC、ConfigMap、Secret、hostPath 一律成立**——不只 PVC。

## 正确与错误写法

```yaml
# ✅ 正确：挂到镜像中不存在的独立目录
volumeMounts:
  - name: data
    mountPath: /data          # 镜像里没有 /data，不遮蔽任何东西

# ❌ 错误：遮蔽了 /usr/share 的全部内容
volumeMounts:
  - name: data
    mountPath: /usr/share

# ⚠️ 折中：必须用系统目录下的路径时，挂「镜像中不存在的子目录」
volumeMounts:
  - name: data
    mountPath: /usr/share/myapp-data
```

**绝对不要**挂 `/usr`、`/etc`、`/lib`、`/var`、`/bin`、`/sbin`、`/opt`、`/root`、`/home` 这类系统目录的**顶层**。

## 挂在 /usr/share 的爆炸半径

`/usr/share` 是 Linux 的**共享数据目录**，遮蔽它远不止影响时区：

| 被遮蔽的目录 | 影响 | 危险度 |
|---|---|---|
| `zoneinfo` | 时区、日志时间戳、定时任务 | 中 |
| `ca-certificates` | **TLS 信任链** | 🔴 高 |
| `p11-kit` | PKCS#11 / TLS 信任库 | 🔴 高 |
| `fonts` / `fontconfig` | Java AWT、字体渲染、图片/PDF | 🟡 中 |
| `locale` / `i18n` | 本地化、中文处理 | 🟡 中 |
| `xml` | XML 目录解析 | 🟡 中 |
| `gnupg` | GPG 数据 | 🟡 中 |

> [!warning] 最容易忽视的连带故障
> 时区只是**最容易被发现**的症状。同一个挂载配置还可能造成 **TLS 握手失败**（CA 证书被遮蔽）和**中文乱码**（locale 被遮蔽），这些故障往往被归因到别处，长期得不到解决。

## 空挂载点的识别特征

```console
$ ls -la /usr/share/
total 8
drwxr-xr-x 2 root root 4096 Sep  8 00:39 .      ← link count = 2
drwxr-xr-x 1 root root 4096 Sep 22 02:25 ..
```

- **link count = 2**：目录的硬链接数等于"子目录数 + 2"。值为 2 表示**没有任何子目录**，是典型的空挂载点特征
- **mtime 是挂载点/PV 目录自身的属性**，与镜像构建时间**毫无关系**，不能用来推断"文件何时被删"

## 诊断命令

```bash
# ① 一条命令排除本类问题（最关键）
mount | grep -E '/usr/share|/etc/localtime'
findmnt /usr/share
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].volumeMounts}' | jq .

# ② 全量排查所有 Pod 的危险挂载点
kubectl get pods -n <ns> -o json | jq -r '
  .items[] | .metadata.name as $p |
  .spec.containers[] | .name as $c |
  (.volumeMounts // [])[] | "\($p)\t\($c)\t\(.mountPath)"
' | grep -E '/(usr|etc|lib|var|bin|sbin|opt|home|root)(/|$)'
```

## 预防：准入校验

用 **OPA / Kyverno** 等准入策略在 Push 阶段拦截落在系统目录下的挂载点，从源头杜绝这类事故。

## 参考链接

- [[案例-K8s容器时区异常-PVC挂载遮蔽usr-share]] — 该语义导致的完整故障案例
- [[Linux-mount挂载机制]] — 挂载的底层原理
- [[Docker容器技术]] — 容器文件系统分层
