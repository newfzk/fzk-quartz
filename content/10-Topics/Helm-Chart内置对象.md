---
title: helm-chart内置对象
type: basic-note
date: 2026-01-15
tags: helm, chart, templates
---

# helm-chart内置对象

> 官方文档：<https://helm.sh/zh/docs/v3/chart_template_guide/builtin_objects>

- Release： Release对象描述了版本发布本身。包含了以下对象：
  - Release.Name： release名称
  - Release.Namespace： 版本中包含的命名空间(如果manifest没有覆盖的话)
  - Release.IsUpgrade： 如果当前操作是升级或回滚的话，该值将被设置为true
  - Release.IsInstall： 如果当前操作是安装的话，该值将被设置为true
  - Release.Revision： 此次修订的版本号。安装时是1，每次升级或回滚都会自增
  - Release.Service： 该service用来渲染当前模板。Helm里始终Helm
- Values： Values对象是从values.yaml文件和用户提供的文件传进模板的。默认为空
- Chart： Chart.yaml文件内容。 Chart.yaml里的所有数据在这里都可以可访问的。比如 {{ .Chart.Name }}-{{ .Chart.Version }} 会打印出 mychart-0.1.0 在[Chart 指南](https://helm.sh/zh/docs/v3/topics/charts#Chart-yaml-%E6%96%87%E4%BB%B6) 中列出了可获得属性（也就是Chart.yaml支持的属性）
- Files： 在chart中提供访问所有的非特殊文件的对象。你不能使用它访问Template对象，只能访问其他文件。 请查看这个[文件访问](https://helm.sh/zh/docs/chart_template_guide/accessing_files)部分了解更多信息
  - Files.Get 通过文件名获取文件的方法。 （.Files.Getconfig.ini）
  - Files.GetBytes 用字节数组代替字符串获取文件内容的方法。 对图片之类的文件很有用
  - Files.Glob 用给定的shell glob模式匹配文件名返回文件列表的方法
  - Files.Lines 逐行读取文件内容的方法。迭代文件中每一行时很有用
  - Files.AsSecrets 使用Base 64编码字符串返回文件体的方法
  - Files.AsConfig 使用YAML格式返回文件体的方法
- Capabilities： 提供关于Kubernetes集群支持功能的信息
  - Capabilities.APIVersions 是一个版本列表
  - Capabilities.APIVersions.Has $version 说明集群中的版本 (比如,batch/v1) 或是资源 (比如, apps/v1/Deployment) 是否可用
  - Capabilities.KubeVersion 和Capabilities.KubeVersion.Version 是Kubernetes的版本号
  - Capabilities.KubeVersion.Major Kubernetes的主版本
  - Capabilities.KubeVersion.Minor Kubernetes的次版本
  - Capabilities.HelmVersion 包含Helm版本详细信息的对象，和 helm version 的输出一致
  - Capabilities.HelmVersion.Version 是当前Helm语义格式的版本
  - Capabilities.HelmVersion.GitCommit Helm的git sha1值
  - Capabilities.HelmVersion.GitTreeState 是Helm git树的状态
  - Capabilities.HelmVersion.GoVersion 是使用的Go编译器版本
- Template： 包含当前被执行的当前模板信息
  - Template.Name: 当前模板的命名空间文件路径 (e.g. mychart/templates/mytemplate.yaml)
  - Template.BasePath: 当前chart模板目录的路径 (e.g. mychart/templates)

## 相关笔记

- [[Helm-Chart模板概述]]