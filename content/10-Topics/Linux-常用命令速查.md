---
title: Linux 常用命令速查
type: basic-note
date: 2026-06-03
tags:
---

# Linux 常用命令速查

## 文件目录

### ls

- `-a` 全部文件，包括隐藏文件、`.`、`..`
- `-A` 全部文件，不包括 `.`、`..`
- `-F` 根据文件类型，在文件名后添加字符：`*` 可执行文件，`/` 目录，`=` socket，`|` FIFO
- `-i` 列出各文件的 inode 索引
- `-l` 长格式，显示权限、大小、时间等
- `-n` 类似 `-l`，但用 UID/GID 数字表示用户和组
- `-r` 反向排序
- `-S` 按文件大小排序
- `-t` 按修改时间排序
- `--full-time` 显示完整时间
- `--time={atime,ctime}` 指定显示访问时间或更改时间
- `-l --time-style=long-iso` 以更舒适的格式显示时间，如 `2024-10-14 22:54`
- `--color={always,auto,never}` 控制颜色显示

### cp

- `-a` 相当于 `-pdr`，保留文件属性，递归复制
- `-d` 复制链接文件属性而非文件本身
- `-l` 建立硬链接而非复制
- `-s` 建立软链接（符号链接）
- `-p` 与文件属性一起复制
- `-u` 更新式复制，仅当源文件比目标文件新或目标文件不存在时复制
- `-i` 覆盖前询问
- `-f` 强制覆盖
- `-r` 递归复制目录

> 软链接 vs 硬链接：软链接类似快捷方式，以路径指向目标文件；硬链接是原文件的别名，删除任意一个后另一个仍可用。

### basename / dirname

- `basename`：获取文件名（不含目录）
- `dirname`：获取目录名（不含文件名）

### stat

查看文件状态，包括访问时间、修改时间、更改时间等。

### 查看内核支持的文件系统

```shell
ls /lib/modules/`uname -r`/kernel/fs
```

## 系统性能

### uptime

查看系统负载信息：

```shell
$ uptime
16:11:08 up 15 days, 13:24,  2 users,  load average: 1.00, 0.64, 0.41
```

> 如果每个 CPU 内核的当前活动进程数不大于 3，系统性能良好。大于 5 则性能严重问题。

### vmstat

显示虚拟内存状态。

### dmesg

查看系统启动信息：

```shell
dmesg | head          # 查看启动信息
dmesg | grep vda      # 查看硬盘信息
```

## 进程监控

- `ps`：查看进程状态
- `top`：实时进程监控
- `lsof`：列出打开的文件

## 网络

### netstat

查看网络系统状态信息：

| 参数 | 含义 |
|------|------|
| `-a` | 列出所有 Socket |
| `-t` | TCP 协议 |
| `-u` | UDP 协议 |
| `-x` | UNIX Socket |
| `-l` | 监听状态的 Socket |
| `-n` | 直接使用 IP 和端口号（不解析主机名） |
| `-p` | 显示程序 PID 和名称 |
| `-r` | 显示路由信息 |

#### 查看端口是否被占用

```shell
netstat -alnp | grep ':80'
```

#### TCP 连接状态

| 状态 | 含义 |
|------|------|
| LISTEN | 服务端监听 |
| SYN_SENT | 客户端发送 SYN，等待匹配 |
| SYN_RECV | 服务端收到 SYN，已回复 SYN+ACK |
| ESTABLISHED | 连接已建立，可传输数据 |
| FIN_WAIT1/2 | 主动关闭端等待 |
| CLOSE_WAIT | 被动关闭端等待应用层关闭 |
| LAST_ACK | 被动关闭端发送 FIN，等待 ACK |
| TIME_WAIT | 主动关闭端收到 FIN 后等待 |
| CLOSED | 连接关闭 |

## 用户、权限

### newgrp

以新的组身份打开新 Shell：

```shell
newgrp docker
```

### id

查看用户和组信息：

```shell
id -a     # 所有信息
id -un    # 用户名
id -gn    # 组名
id -Gn    # 所有组名
id -ru    # 真实用户 ID
```

## 系统信息

### uname

打印系统信息：

| 参数 | 含义 |
|------|------|
| `-a` | 所有信息 |
| `-s` | 内核名称 |
| `-n` | 主机名 |
| `-r` | 内核发行版本号 |
| `-v` | 内核版本详细信息 |
| `-m` | 机器硬件架构 |
| `-o` | 操作系统名称 |

### lscpu

查看 CPU 信息：

```shell
lscpu
# 输出包括：Architecture, CPU(s), Model name, L1/L2/L3 cache 等
```

## 环境变量

### BASHPID

表示当前 Bash 进程的 PID，在子 Shell 中更精确（`$$` 在子 Shell 中不变）。

```shell
echo $BASHPID
```

## 相关笔记

- [[Linux-mount命令]]
- [[Linux-mount挂载机制]]
- [[lsof-文件诊断工具]]
- [[stat-命令详解]]
- [[Linux文本处理三剑客]]
- [[Linux-comm命令详解]]
- [[iptables端口转发]]