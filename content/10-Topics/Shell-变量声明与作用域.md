---
title: Shell 变量声明与作用域
type: basic-note
date: 2026-07-02
tags:
  - language/shell
  - topic/Linux/命令
  - topic/shell/variable
status: to-review
---

# Shell 变量声明与作用域

## 声明方式一览

| 方式 | 作用域 | 子进程继承 | 典型场景 |
|------|--------|:----------:|----------|
| `a='1'` | 当前 Shell 局部变量 | ❌ | 临时计算、循环计数器、函数内局部状态 |
| `export a='1'` | 环境变量（全局） | ✅ | 配置子进程行为（PATH、HOME、LANG） |
| `declare a=1` | 当前 Shell 变量 | ❌ | 显式声明，可配合 `-i` `-a` `-A` 等类型标志 |
| `local a=1` | 函数内局部变量 | ❌ | 函数内部使用，避免污染全局命名空间 |
| `readonly a=1` | 只读变量 | 取决于是否 export | 常量定义，不可修改 |
| `a='1' command` | 临时环境变量 | ✅（仅对 command 生效） | 单条命令的临时环境配置 |

---

## 详细说明

### 1. 基本赋值 `a='1'`

最基础的形式，变量仅在当前 Shell 进程内可见，子进程无法访问。

```bash
a='hello world'
echo $a               # 输出: hello world
bash -c 'echo $a'     # 输出:（空，子进程看不到）
```

### 2. 环境变量 `export a='1'`

将变量标记为环境变量，**当前 Shell 及所有子进程均可见**。

```bash
export PATH="/usr/local/bin:$PATH"   # 子进程继承 PATH
export a='hello'
bash -c 'echo $a'                    # 输出: hello（子进程可见）
```

`export` 的本质是两步操作：
```bash
a='hello'
export a         # 标记变量 a 为导出属性
```

### 3. 一次性环境变量 `a='1' command`

变量仅在命令的子进程中存在，不影响当前 Shell：

```bash
LANG=en_US.UTF-8 python3 script.py    # 仅该 Python 进程使用英文环境
echo $LANG                            # 当前 Shell 的 LANG 不受影响
```

### 4. `declare` 类型声明

Bash 内置命令，可显式声明变量并指定类型：

```bash
declare -i num=10        # 整数类型（算术运算无需 $(( ))）
declare -a arr           # 普通数组
declare -A map           # 关联数组（Bash 4.0+）
declare -r const=42      # 只读（等同 readonly）
declare -x env_var=foo   # 导出（等同 export）
declare -p variable      # 查看变量属性
```

### 5. `local` 函数内局部变量

仅在函数内部生效，函数退出后自动销毁：

```bash
myfunc() {
  local count=0         # 函数内局部变量
  count=$((count + 1))
  echo $count           # 输出: 1
}
echo $count             # 输出:（空，外部不可见）
```

### 6. `readonly` 只读变量

一旦赋值不可修改，常用于常量定义：

```bash
readonly MAX_CONNECTIONS=1000
readonly -p             # 列出所有只读变量
MAX_CONNECTIONS=2000    # 错误: bash: MAX_CONNECTIONS: readonly variable
```

---

## 重要细节

### 引号的区别

```bash
a=$HOME        # 变量 $HOME 展开为 /home/user
a="$HOME"      # 同上（推荐，语义清晰）
a='$HOME'      # 字面量字符串 "$HOME"，变量不展开
```

### `declare -x` vs `export`

两者功能等价，`export` 是 POSIX 标准，`declare -x` 是 Bash 扩展。推荐在脚本内用 `declare` 统一风格，交互式 Shell 用 `export`。

### 子进程视角

```bash
a='local'
export b='global'

bash -c 'echo "a=$a b=$b"'    # 输出: a= b=global
```

环境变量（`export`）通过 `fork()`/`exec()` 时的环境块传递给子进程；非导出变量仅存在于当前 Shell 的内存中。

### 查看变量

```bash
set          # 查看所有变量（含环境变量）
env          # 仅查看环境变量
printenv     # 等同于 env
export -p    # 查看所有导出的变量
declare -p   # 查看所有变量及其属性
```

---

## 相关笔记

- [[Shell-脚本备忘]] — Shell 脚本综合备忘（字符串、数组、函数、调试）
- [[Shell-脚本执行方式-source-vs-bash-vs-direct]] — source/bash/直接执行的进程模型差异
- [[Docker-Exec形式与Shell形式]] — Exec 形式不经过 Shell，环境变量展开方式不同
- [[Linux-常用命令速查]] — Linux 命令参考，包含 `BASHPID` 等特殊变量
- [[Linux-文件描述符fd详解]] — stdin/stdout/stderr 与 Shell 重定向 `2>&1`
- [[Spring-Boot-外部化配置优先级]] — 环境变量在 Spring Boot 配置中的优先级
