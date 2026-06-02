# shell脚本备忘

## 复杂数据结构操作

参考链接：[shell基础-字符串 数组 map](https://www.cnblogs.com/tkzc2013/p/15349686.html)

### 1 字符串

#### 1.1 字符串拼接

```sh
your_name="runoob"
# 使用双引号拼接
greeting="hello, "$your_name" !"
greeting_1="hello, ${your_name} !"
echo $greeting  $greeting_1
# 使用单引号拼接
greeting_2='hello, '$your_name' !'
greeting_5='hello, $your_name !' # 无效
greeting_3='hello, ${your_name} !' #无效
echo $greeting_2  $greeting_3 $greeting_5

#输出结果
hello, runoob ! hello, runoob !
hello, runoob ! hello, ${your_name} ! hello, $your_name !
```

#### 1.2 字符串长度 ${#str}

```sh
string="abcd"
echo ${#string} #输出 4
```

#### 1.3 截取部分字符串 ${str:1:4}

`${str:a:b}` a,b均为非负整数，闭区间（`[a,b]`），索引从0开始算起

```sh
string="runoob is a great site"
echo ${string:1:4} # 输出 unoo
```

#### 1.4 查找字符（首次出现索引）

> 查看字符 i 或 o 的位置(哪个字母先出现就计算哪个)

```sh
string="runoob is a great site"
echo `expr index "$string" io`  # 输出 4
```

### 2 数组

#### 2.1 定义

```sh
#方法1
array_name=(value0 value1 value2 value3)

#方法2
array_name=(
value0
value1
value2
value3
)

#方法3
array_name[0]=value0
array_name[1]=value1
array_name[n]=valuen
```

#### 2.2 读取

单个元素：`${arr[5]}`

全部（用于循环）:`${arr[@]}` 或 `${arr[*]}`

数组长度：`${#arr[@]}` 或 `${#arr[*]}` （使用`#`获取长度）

获取数组中单个元素长度：`${#arr[n]}`

### 3 map

> 关联数组 `declare -A` 4.0 以上的 bash 才支持 `bash --version`

#### 3.1 定义

```sh
#方式1
declare -A m
m["zh"]="中国"

#方式2
declare -A m=(["zh"]="中国" ["cn"]="美国")
```

#### 3.2 读取

获取某个元素：`${m[key]}`

获取所有key：`${!m[@]}`  带有`!` 这里的结果是一个元组

获取所有value：`${m[@]}` 不带`!` 这里的结果是一个元组

获取长度：`${#m[@]}`

#### 3.3 遍历

```sh
# 遍历key
for key in ${!m[@]}; do
    echo "key:$key"
    echo "key:$key, val:${m[$key]}"
done

# 遍历value
for val in ${m[@]}; do
    echo "val:$val"
done
```

## return

shell return 必须是数字 [0,255] 0表示成功 其他代表失败；

如果没有return **函数将以最后一条命令的运行结果作为返回值** 因此，如果需要返回**字符串**、**列表值**等，都可以以echo作为返回值（仅最后一个echo）

参考：[shell函数返回值-csdn](https://blog.csdn.net/m0_45406092/article/details/134420004)

## 跨文件函数调用

通过`source shell_file`完成其他shell脚本文件的函数定义，后续即可正常调用，示例如下：

log.sh文件内容如下：

```sh
#!bin/bash

function LOG_NOTICE()
{
    echo -e "\033[34m${1}\033[0m"
}
```

调用的脚本文件内容如下：

```sh
#!/bin/bash

source ./log.sh # 同目录下

LOG_NOTICE "NOTICE"
```

参考链接：[shell 跨文件函数调用](https://blog.csdn.net/younger_china/article/details/51921631)

## debug

> 参考博客：[debug in shell script - opstree](https://opstree.com/blog/2021/11/23/debugging-in-shell-script/)

### `set -x` 显示命令及其输出

最常见的是-x选项，它将在调试模式下运行脚本。Set -x显示命令以及它们在终端上的输出，这样你就可以知道哪个命令的输出是什么。使用方式如下：

1. 执行脚本时：`bash -x shell.sh`
2. 在文件头：`#!/bin/bash -x`
3. 脚本内部部分调试:

```sh
set -x # enable debugging
echo "foo"
set +x # disable debugging
```

输出如下：

```text
root@localhost$ ./script.sh 
+ echo foo
foo
+ set +x
```

### `set -U` 禁止忽略未注册变量

默认情况为：

```sh
#!/bin/bash
echo $foo
echo "bar"
```

输出如下：

```sh
root@localhost$ ./script.sh 

bar
```

在上面的脚本，我们的脚本没有抛出任何错误，而是忽略了未定义的变量，并继续执行脚本的其余部分。这是不应该的，因此可以使用`set -u`，示例如下

```sh
#!/bin/bash
set -u
echo $foo
echo "bar"
```

输出如下：

```sh
root@localhost$ ./script.sh 
./script.sh: line 3: foo: unbound variable
```

### `set -e` 报错后立刻停止

默认发生报错后仍会继续执行脚本，示例如下：

```sh
#!/bin/bash
foo
echo "bar"
```

输出如下：

```sh
root@localhost$ ./script.sh 
./script.sh: line 2: foo: command not found
bar
```

使用`set -e`可以避免跳过报错，导致错误累计（无法及时发现错误），示例如下：

```sh
#!/bin/bash
set -e
foo
echo "bar"
```

输出如下

```sh
root@localhost$ ./script.sh 
./script.sh: line 3: foo: command not found
```

`set -e`是通过返回值判断脚本是执行成功还是失败，对于特殊命令（返回非零值，但是不希望运行失败）可以临时关闭`set -e`（即使用`set +e`），特殊命令执行完毕后，重新启用`set -e`

### `set -eo pipefail` 识别管道故障（子命令故障）

`set -e`仅会通过最后一个命令的返回值判断是否执行成功，这就会导致如下现象：

```sh
#!/bin/bash
set -e
foo | echo "a"    # foo 执行失败 但因为 echo "a" 执行成功 因此 set -e 判断命令执行成功
echo "bar"
```

输出如下

```sh
vikas.b4_ote$ ./script.sh 
a
./script.sh: line 3: foo: command not found
bar
```

解决示例：

```sh
#!/bin/bash
set -eo pipefail
foo | echo "a"
echo "bar"
```

输出如下

```sh
root@localhost$ ./script.sh 
a
./script.sh: line 3: foo: command not found
```

### 其他选项查看/配置 set -o

使用`set -o`可以显示所有选项（和其当前状态），可以使用想要使用的选项，使脚本更健壮，示例如下：

```text
root@localhost$ set -o
allexport       off
braceexpand     on
emacs           on
errexit         off
errtrace        off
functrace       off
hashall         on
histexpand      on
history         on
ignoreeof       off
interactive-comments on
keyword         off
monitor         on
noclobber       off
noexec          off
noglob          off
nolog           off
notify          off
nounset         off
onecmd          off
physical        off
pipefail        off
posix           off
privileged      off
verbose         off
vi              off
xtrace          off
```

## 文件读取

`cat filen_name | while read -r <变量名> ; do` 细节待学习补充

### 读取文件 按行存入数组

#### mapfile

读取文件并将每行内容存入数组变量，示例如下：

```shell
#!/bin/bash

# 文件名
file="yourfile.txt"

# 使用mapfile或readarray读取文件内容到数组中
mapfile -t array < "$file"

# 打印数组内容以验证
echo "数组内容如下："
for elem in "${array[@]}"
do
    echo "$elem"
done
```

#### readarray

### csv文件读取

csv文件(`above-csv-file.csv`)示例如下：

```csv
containers-per-node,map-mem,map-vcores,reduce-mem,reduce-vcores,am-mem,am-vcores,mappers
4,15360,8,15360,8,15360,8,127
8,7680,4,7680,4,7680,4,255
12,5120,3,5120,4,5120,4,383
16,3840,2,7680,4,7680,4,510
```

csv文件读取脚本如下：

```sh
IFS=,
row=0
cat above-csv-file.csv | while read -r CONTAINERS_PER_NODE MAP_MEM MAP_VCORES REDUCE_MEM REDUCE_VCORES AM_MEM AM_VCORES MAPPERS; do
    # skip first header line
    if [[ $row -eq 0 ]]; then
        row=$((row+1))
        continue
    fi
    echo "mapreduce.map.memory.mb = $MAP_MEM"
    echo "mapreduce.map.cpu.vcores = $MAP_VCORES"
    echo "mapreduce.reduce.memory.mb = $REDUCE_MEM"
    echo "mapreduce.reduce.cpu.vcores = $REDUCE_VCORES"
    echo "yarn.app.mapreduce.am.resource.mb = $AM_MEM"
    echo "yarn.app.mapreduce.am.resource.cpu-vcores = $AM_VCORES"
    echo "total containers per node: $CONTAINERS_PER_NODE"
    echo "total containers for map: $MAPPERS"
    row=$((row + 1))
done

```
