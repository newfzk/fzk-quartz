---
title: docker-history是审计日志不是文件清单
date: 2026-09-22
updated: 2026-09-22
aliases:
  - docker history 误导
  - RootFS.Layers diff_id
  - 镜像层内容比对
related:
  - "[[案例-K8s容器时区异常-PVC挂载遮蔽usr-share]]"
  - "[[Docker容器技术]]"
tags:
  - topic/Docker
  - topic/故障排查
status: to-review
---

# docker history 是审计日志，不是文件清单

## 核心区别

| | `docker history` | 镜像层内容 |
|---|---|---|
| 记录内容 | **这一层执行了什么命令**（命令文本） | **这层最终在文件系统里留下了什么** |
| 性质 | 构建过程的**操作审计日志** | 真实的内容快照 |
| 能证明 | 命令被执行过 | 文件是否存在 |

> [!danger] 最常见的误用
> 看到 `docker history` 里有 `apk add --no-cache tzdata` 这一行，就断定"tzdata 装了"。
>
> **这是错的。** history 只记录命令文本，无法证明：
> - 该层最终留下了这些文件（**后续层可以删除前面层装的文件**）
> - 同一行命令在不同时间执行会产出相同内容（**依赖的外部源可能已变**）
> - 两个同名 tag 是否指向同一层内容

## 为什么 history 具有误导性

```console
$ docker history <image>
IMAGE   CREATED BY
xxxx    RUN /bin/sh -c set -eux; apk add --no-cache fontconfig ttf-dejavu gnupg \
        ca-certificates p11-kit-trust musl-locales binutils tzdata coreutils openssl ; \
        rm -rf /var/cache/apk/* # buildkit
```

这一行**只说明构建时执行过这条命令**。它可能：

1. 在**某个后续层**中被 `rm -rf /usr/share/*` 之类的操作移除
2. 在另一个**同名 tag、不同内容的镜像**里，因基础镜像不同而产生完全不同的结果
3. 因移除操作发生在**运行时挂载**层，导致文件"看起来"不存在，而 history 里毫无痕迹

## 正确做法：比对层 diff_id

要看**内容**是否相同，必须比对层的 **diff_id（内容哈希）**：

```bash
docker inspect --format '{{json .RootFS.Layers}}' <imageA> | jq -r '.[]' | nl > a.layers
docker inspect --format '{{json .RootFS.Layers}}' <imageB> | jq -r '.[]' | nl > b.layers
diff a.layers b.layers
```

- `.RootFS.Layers` 列出每个层的 **diff_id**（内容寻址哈希）
- diff_id 相同 = **层内容相同**；不同 = 内容有差异
- 这比对比 `IMAGE` ID 或 history 文本可靠得多

## 更彻底的验证：直接看最终文件系统

绕开 history 与包数据库的所有歧义，直接导出镜像的最终文件树：

```bash
# 看镜像里 /usr/share 下到底有什么
id=$(docker create <image>)
docker export "$id" | tar -t | grep "^usr/share/"
docker rm "$id"
```

> [!tip] 这是"文件清单级"的证据
> `docker export` 导出的是**容器最终的文件系统**，不含任何推断成分。当 history、apk db、`ls` 三者的结论互相矛盾时，用这个方法一锤定音。

## 三层证据的可信度排序

排查"镜像里到底有没有某文件"时，按下表选择手段：

| 手段 | 可信度 | 说明 |
|---|---|---|
| 容器内 `ls` / `mount` | ⭐⭐⭐⭐⭐ | 真实运行时视图（注意挂载遮蔽） |
| `docker export` + `tar -t` | ⭐⭐⭐⭐⭐ | 镜像静态内容，无歧义 |
| `.RootFS.Layers` diff_id 比对 | ⭐⭐⭐⭐ | 判断两个镜像是否内容相同 |
| 包数据库（apk db） | ⭐⭐⭐ | 反映"装过"，不反映"可见" |
| `apk audit` | ⭐⭐ | 默认只审计 `/etc`，盲区大 |
| **`docker history`** | ⭐ | **只是命令文本，最易误导** |

## 相关：镜像内容不一致的根本原因

若两个镜像 history 相同但内容不同，常见原因：

1. **同名 tag 被覆盖推送**（Harbor 未配 Tag Immutability）
2. **构建缓存不一致**（`docker build` 处理 `FROM` 时默认不检查 registry 上 tag 是否已更新）
3. 基础镜像本身被替换

对应加固措施：

```dockerfile
# 用 digest 固定基础镜像（tag 可变，digest 不可变）
FROM registry.example.com/library/image@sha256:<digest>
```

```bash
# 若必须用 tag，构建时强制拉取最新
docker build --pull -t ...
```

并在 Harbor 上配置 **Tag Immutability Rules**，禁止基础镜像 tag 被覆盖推送。

## 参考链接

- [[案例-K8s容器时区异常-PVC挂载遮蔽usr-share]] — 本认知被误用导致的排查弯路
- [[Docker容器技术]] — 镜像构建相关知识地图
