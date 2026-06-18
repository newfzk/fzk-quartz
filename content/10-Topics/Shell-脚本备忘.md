---
title: Shell 脚本备忘
type: basic-note
date: 2026-06-03
tags:
  - language/shell
  - topic/Linux/命令
status: to-review
---

# Shell 脚本备忘

## 字符串操作

### 拼接

```shell
your_name="runoob"
# 双引号拼接（推荐，变量可解析）
greeting="hello, "$your_name" !"
greeting_1="hello, ${your_name} !"
# 单引号拼接（变量不解析）
greeting_2='hello, '$your_name' !'
```

### 长度

```shell
string="abcd"
echo ${#string}  # 输出 4
```

### 截取

```shell
string="runoob is a great site"
echo ${string:1:4}  # 输出 unoo
```

### 查找字符首次出现位置

```shell
string="runoob is a great site"
echo $(expr index "$string" io)  # 输出 4
```

## 数组

### 定义

```shell
# 方式1
array_name=(value0 value1 value2 value3)

# 方式2（多行）
array_name=(
value0
value1
value2
)

# 方式3（按索引）
array_name[0]=value0
array_name[1]=value1
```

### 读取

```shell
${arr[5]}         # 单个元素
${arr[@]}         # 所有元素（用于循环）
${#arr[@]}        # 数组长度
${#arr[n]}        # 单个元素长度
```

## Map（关联数组）

> 需要 Bash 4.0+

### 定义

```shell
declare -A m
m["zh"]="中国"

# 或一行定义
declare -A m=(["zh"]="中国" ["cn"]="美国")
```

### 读取

```shell
${m[key]}         # 获取某个元素
${!m[@]}          # 获取所有 key
${m[@]}           # 获取所有 value
${#m[@]}          # 获取长度
```

### 遍历

```shell
# 遍历 key
for key in ${!m[@]}; do
    echo "key:$key, val:${m[$key]}"
done

# 遍历 value
for val in ${m[@]}; do
    echo "val:$val"
done
```

## 函数返回值

`return` 必须是数字 `[0,255]`，0 表示成功。返回字符串/列表使用 `echo`：

```shell
function get_name() {
    echo "hello"
}
result=$(get_name)
```

## 跨文件函数调用

```shell
source ./log.sh  # 引入其他脚本
LOG_NOTICE "NOTICE"  # 调用其中的函数
```

## Debug 调试

### set -x：显示命令及输出

```shell
#!/bin/bash -x           # 方式1：文件头
bash -x shell.sh         # 方式2：命令行

set -x                   # 方式3：脚本内部启用
echo "foo"
set +x                   # 关闭调试
```

### set -u：禁止忽略未定义变量

```shell
#!/bin/bash
set -u
echo $foo  # 报错而非静默跳过
```

### set -e：报错后立刻停止

```shell
#!/bin/bash
set -e
foo  # 执行失败，脚本停止
echo "bar"  # 不会执行
```

### set -eo pipefail：识别管道中的子命令故障

```shell
#!/bin/bash
set -eo pipefail
foo | echo "a"  # foo 失败，即使 echo 成功，脚本也停止
echo "bar"       # 不会执行
```

## 文件读取

### mapfile 按行读入数组

```shell
mapfile -t array < "$file"
for elem in "${array[@]}"; do
    echo "$elem"
done
```

### CSV 文件读取

```shell
IFS=,
row=0
cat above-csv-file.csv | while read -r col1 col2 col3; do
    if [[ $row -eq 0 ]]; then  # 跳过表头
        row=$((row+1))
        continue
    fi
    echo "col1 = $col1"
    row=$((row + 1))
done
```

## 相关笔记

- [[Linux-常用命令速查]]