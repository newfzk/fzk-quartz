---
title: helm-chart模板函数和管道符
type: basic-note
date: 2026-01-15
tags: helm, chart, templates
---

# helm-chart模板函数和管道符

模板函数语法：`functionName arg1 ...`，如`{{ quote .Values.port }}

## 模板函数

- `quote`引用函数 可以将参数用引号括起来

helm可用函数其实就是 [Go模板语言](https://pkg.go.dev/text/template?utm_source=godoc) + [Sprig模板库](https://masterminds.github.io/sprig/) +

## 管道符

> 官方文档：<https://masterminds.github.io/sprig/>

## 相关笔记

- [[Helm-Chart模板概述]]