---
title: helm-chart-templates
type: basic-note
date: 2026-01-08
tags: helm, chart, templates
---

# helm-chart-templates

> helm chart 模板和文件和values解释

Helm Chart 模板是按照[Go](https://pkg.go.dev/text/template)模板语言书写， 增加了50个左右的附加模板函数来自 Sprig库 和一些其他指定的函数。

所有模板文件存储在chart的 `templates/` 文件夹。 当Helm渲染chart时，它会通过模板引擎遍历目录中的每个文件。

模板的Value通过两种方式提供：

- Chart开发者可以在chart中提供一个命名为 `values.yaml` 的文件。这个文件包含了默认值。
- Chart用户可以提供一个包含了value的YAML文件。可以在命令行使用 helm install命令时提供。

当用户提供自定义value时，这些value会覆盖chart的`values.yaml`文件中value。

- helm 内置对象：[[helm-chart-builtin-objects]]
- 模板函数和管道符 [[helm-chart-functions-and-pipelines]]

## 参考链接

- [Chart模板指南](https://helm.sh/zh/docs/v3/chart_template_guide/getting_started)
- [helm chart Templates and Values](https://helm.sh/zh/docs/v3/topics/charts/#templates-and-values)

[helm-chart-builtin-objects]: helm-chart-builtin-objects.md "helm-chart内置对象"
[helm-chart-functions-and-pipelines]: helm-chart-functions-and-pipelines.md "helm-chart模板函数和管道符"
