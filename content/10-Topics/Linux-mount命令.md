---
title: Linux mount 挂载命令
type: basic-note
date: 2025-06-09
tags: linux, mount, 命令
---

# Linux mount 挂载命令

## 临时挂载

```shell
mkdir -p <目标目录>
mount <源目录> <目标目录>
```

## 永久生效

```shell
# 修改配置文件
vim /etc/fstab

# 使配置生效
mount -a
```

## 相关笔记

- [[Linux-mount挂载机制]]
- [[Linux-挂载选项详解]]