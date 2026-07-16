---
tags:
  - yq
  - cli
  - devtools
  - yaml
status: to-review
---

# yq-命令简介

yq 是一个轻量级、可移植的命令行 **YAML/JSON/XML/INI/Properties/CSV/TSV** 处理器。

## 特点

- 采用类似 `jq` 的语法，但支持多种格式
- 使用 Go 编写，单二进制无依赖
- 支持通过包管理器、Docker、Podman 安装

## 基本用法

```bash
# 读取 YAML 文件中的某个值
yq '.key' file.yml

# 修改 YAML 值
yq '.key = "newValue"' -i file.yml

# 格式转换
yq -p json -o yaml file.json
```

## 与 jq 的关系

- 语法风格相似，但 yq 主要面向 YAML
- 尚未支持 `jq` 的全部功能，但已覆盖最常用的操作和函数
- 持续开发中

## 参考

- [官网](https://mikefarah.gitbook.io/yq)
- [GitHub: mikefarah/yq](https://github.com/mikefarah/yq)
