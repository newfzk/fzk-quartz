---
title: Shell 脚本执行方式 — source vs bash vs 直接执行
type: basic-note
date: 2026-07-02
tags:
  - language/shell
  - topic/Linux/命令
  - topic/shell/variable
status: to-review
---

# Shell 脚本执行方式 — source vs bash vs 直接执行

## 对比总览

| 方式 | 进程模型 | 变量/状态变更是否影响当前 Shell | 执行权限 | shebang 作用 |
|------|----------|:------------------------------:|:--------:|:-----------:|
| `source ./脚本.sh` / `. ./脚本.sh` | 当前 Shell 进程内执行 | ✅ **影响** | 不需要 | 被忽略 |
| `bash ./脚本.sh` | 启动新的 Bash 子进程 | ❌ 不影响 | 不需要 | 被忽略 |
| `./脚本.sh` | 启动新的子进程（由 shebang 指定解释器） | ❌ 不影响 | **需要** | ✅ 决定使用哪个解释器 |

---

## 详细说明

### 1. `source ./脚本.sh`（或简写 `. ./脚本.sh`）

在当前 Shell 进程中直接读取并执行脚本，**不创建子进程**。

```bash
# test.sh 内容
export NAME='sourced'
pwd_val=$(pwd)
cd /tmp

# 在主 Shell 中执行
source ./test.sh
echo $NAME       # 输出: sourced（变量保留）
pwd              # 输出: /tmp（目录变更保留）
```

**关键特性**：
- 脚本中的 `cd`、变量赋值、`export`、函数定义等**全部影响当前 Shell**
- 脚本中的 `exit` 会**直接退出当前 Shell**（危险！应用 `return` 替代）
- 常用于加载配置文件（如 `~/.bashrc`、`~/.profile`）
- 不需要可执行权限（只需读权限）

### 2. `bash ./脚本.sh`

启动一个新的 Bash 子进程来执行脚本，当前 Shell 等待其结束后继续。

```bash
# test.sh 内容
export NAME='child'
cd /tmp

# 在主 Shell 中执行
bash ./test.sh
echo $NAME       # 输出:（空，子进程的变量不保留）
pwd              # 保持原来的路径（目录变更不保留）
```

**关键特性**：
- **完全隔离**：脚本内的任何变更都不影响当前 Shell
- 脚本中的 `exit` **只退出子进程**，不会影响当前 Shell（安全）
- 不需要可执行权限（只需读权限）
- 可以通过**返回值** (`$?`) 获取脚本的执行结果状态
- 是一种**安全的沙箱执行**方式，适合测试脚本

```bash
# 子进程可以通过 stdout 向父进程传数据
output=$(bash ./脚本.sh)        # 捕获 stdout
bash ./脚本.sh && echo "成功"   # 判断返回值
```

### 3. `./脚本.sh`

通过操作系统的 `execve()` 系统调用执行脚本。内核读取脚本的第一行（shebang）确定解释器。

```bash
#!/bin/bash          # shebang 行，指定解释器
echo "Hello from $0"
```

```bash
chmod +x ./脚本.sh   # 需要先赋予可执行权限
./脚本.sh
```

**关键特性**：
- 与 `bash ./脚本.sh` 本质上**行为相同**（也是子进程执行），区别在于：
  - **依赖 shebang**：脚本第一行的 `#!/bin/bash` 决定用哪个解释器
  - **需要执行权限**：`chmod +x`
  - **文件路径查找**：`./脚本.sh` 明确指定当前目录；仅写 `脚本.sh` 会从 `$PATH` 查找
- 如果 shebang 写的是 `#!/usr/bin/python3`，则用 Python 解释执行

> [!warning] 注意
> 如果脚本没有 shebang 行，`./脚本.sh` 会报错 "No such file or directory" 或 "Permission denied"；而 `bash ./脚本.sh` 则正常执行。

---

## 进程模型图解

```
当前 Shell 进程 (PID=1000)
├── source ./脚本.sh     →  在 PID=1000 内直接执行（无新进程）
├── bash ./脚本.sh       →  fork 子进程 PID=1001 → 执行 → 退出
└── ./脚本.sh            →  fork 子进程 PID=1002 → exec shebang 解释器 → 退出
```

---

## 实践场景

### 加载配置 → 用 `source`

```bash
source ~/.bashrc               # 重新加载别名、环境变量
source ./env.sh                # 加载项目环境变量
. ./venv/bin/activate          # 激活 Python 虚拟环境
```

### 执行任务脚本 → 用 `bash` 或 `./`

```bash
bash ./deploy.sh               # 安全执行，不影响当前环境
./build.sh                     # 使用 shebang 指定的解释器
bash -x ./debug.sh             # 调试模式执行
```

### 在脚本中调用另一个脚本

```bash
#!/bin/bash
source ./utils.sh              # 加载工具函数（函数对当前脚本可见）
bash ./独立任务.sh             # 运行独立任务（完全隔离）
```

---

## 与变量声明的关系

这三种执行方式的差异，本质是 [[Shell-变量声明与作用域]] 中**进程作用域**的延伸：

- `source` → 在当前进程内执行，等价于在命令行逐条输入命令
- `bash ./脚本.sh` → 启动子进程，脚本内的 `export` 变量**不会**传回父进程
- 子进程只能通过 `stdout`（命令替换 `$()`）或 `exit code` 向父进程回传信息

---

## 相关笔记

- [[Shell-变量声明与作用域]] — 变量作用域与 export 机制是本篇的基础
- [[Shell-脚本备忘]] — Shell 脚本综合备忘（调试、函数、数组）
