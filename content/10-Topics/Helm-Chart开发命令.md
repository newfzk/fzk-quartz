---
title: helmChart开发
type: basic-note
date: 2026-01-02
tags:
---

# helmChart开发

```sh
# 以项目名为deis-workflow 为例
# 创建初始模板在当前目录下的 deis-workflow 目录中
helm create deis-workflow

# 模板格式验证
helm lint

# helm chart 打包 得到 项目名-版本号.tgz 文件
helm package deis-workflow
```

## 官方文档

- <https://helm.sh/zh/docs/topics/charts>
- <https://helm.sh/zh/docs/intro/using_helm#creating-your-own-charts>

## 相关笔记

- [[Helm-Chart目录结构]]
- [[Helm-Chart模板概述]]