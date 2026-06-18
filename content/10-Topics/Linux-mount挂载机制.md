---
title: Linux-mount挂载机制
date: 2026-06-02
updated: 2026-06-02 00:00:00
tags:
  - topic/Linux
  - topic/Linux/文件系统
status: to-review
---

## 核心概念

`mount` 命令将文件系统关联到目录树中的某个挂载点（mount point），使其内容可被访问。Linux 采用单一目录树结构，所有文件系统都挂载到 `/` 根文件系统下。

## mount 输出格式

```
设备/文件系统 on 挂载点 type 文件系统类型 (挂载选项)
```

| 列 | 含义 | 示例 |
|----|------|------|
| 第 1 列 | 挂载源（设备或虚拟文件系统名） | `/dev/sda1`、`tmpfs` |
| 第 2 列 | `on` | 固定分隔符 |
| 第 3 列 | 挂载点（目录路径） | `/`、`/boot`、`/home` |
| 第 4 列 | `type` | 固定分隔符 |
| 第 5 列 | 文件系统类型 | `ext4`、`xfs`、`tmpfs` |
| 第 6 列 | 括号内的挂载选项 | `rw,noatime` |

## 常见挂载类型

- **bind mount**：将已挂载的目录镜像到另一个位置
- **loop mount**：挂载 ISO 镜像文件
- **tmpfs mount**：挂载内存文件系统
- **NFS/CIFS mount**：网络文件系统挂载

## 挂载命名空间

Linux 支持挂载命名空间（mount namespace），每个容器拥有独立的挂载视图，是容器隔离的基础技术。

## 面试要点

- `/etc/fstab` 定义开机自动挂载的文件系统
- `mount -o remount,rw /` 可以重新挂载根文件系统为读写
- `findmnt` 命令以树形结构展示当前挂载关系（比 mount 更直观）

## 参考链接

- [[柠檬微趣-笔试-Q2-mount命令输出解释|柠檬微趣-笔试-Q2-mount命令输出解释]]