---
title: 挂载 mount
type: basic-note
date: 2025-06-09
tags: mount, linux, command
---

# 挂载 mount

## 目录挂载

> 参考示例：![[kubeasz-dev-on-local#kubeasz本地开发准备]]

### 临时挂载

```shell
mkdir -p <目标目录>
mount <源目录> <目标目录>
```

### 永久生效

```shell
# 修改配置文件
vim /etc/fstab
# 命令生效
mount -a
```

配置文件详细参考 ![[filesystem-table#filesystem-table]]
