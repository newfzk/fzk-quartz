---
title: helm upgrade
type: basic-note
date: 2025-06-14
tags: helm, command upgrade
---

# helm upgrade

`helm upgrade [RELEASE] [CHART] [flags]`

常用参数如下：

- `-f, --values strings` 通过 yaml 文件 或 url 指定一个或多个values值, 如 `-f my-values.yaml`
  - 可以有多个 重复的values配置值 最靠后的优先级最高
- `--set stringArray` 在命令行中制定配置项的值
  - 可以有多个 重复的values配置值 最靠后的优先级最高
- `--atomic` 如果设置，升级过程将回滚升级失败时所做的更改。如果使用了，将自动设置`--wait`
- `--wait` 如果设置，将等到 deploy、StatefulSet或ReplicaSet 的所有PVC、svc和最小数量的pod都处于就绪状态后，再将发布标记为成功。它将等待`--timeout`
- `--timeout 持续时间` 等待任何单个Kubernetes操作的时间(如钩子的作业)(默认为5分钟)
- `-i, --install` 如果此次要安装的release名称不存在，则执行`install`
- `--create-namespace` 依赖于`-i` 如果命名空间不存在 则先创建相应的命名空间

从命令中继承的参数

- `-n, --namespace string` 这次请求面向的命名空间
