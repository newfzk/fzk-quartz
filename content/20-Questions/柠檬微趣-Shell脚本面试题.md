---
title: 柠檬微趣 Shell 脚本面试题
date: 2026-06-04
tags:
  - 面试
  - shell
  - linux
  - 脚本
  - status/to-review
type: question
---

# 柠檬微趣 Shell 脚本面试题

## 1. 转移大于 100K 的文件

编写 shell 脚本将 `/usr/local/test` 目录下大于 100K 的文件转移到 `/tmp` 目录。

```bash
#!/bin/bash
# 遍历目录，find + -exec 或 while 循环均可

find /usr/local/test -type f -size +100k -exec mv {} /tmp/ \;
```

> 相关笔记：[[Shell-脚本备忘]]

## 2. Ping 扫描在线 IP

使用 shell 脚本判断 `192.168.1.0/24` 网络中当前在线的 IP，能 ping 通则认为在线，在线 IP 记录到文件中。

```bash
#!/bin/bash
# 并行 ping 扫描 /24 网段

for i in $(seq 1 254); do
    (ping -c 1 -W 1 192.168.1.$i &> /dev/null && echo "192.168.1.$i" >> online_ips.txt) &
done
wait
echo "扫描完成，在线 IP 已写入 online_ips.txt"
```

## 3. 监控进程存活

监控进程（启动命令：`/usr/local/pg/bin/postgres -D /data/pgdata`）是否存在，不存在就 echo 告警消息。

```bash
#!/bin/bash
# pgrep 比 ps + grep 更准确，避免匹配到 grep 自身

if ! pgrep -f "/usr/local/pg/bin/postgres -D /data/pgdata" &> /dev/null; then
    echo "告警：PostgreSQL 进程不存在！"
fi
```

> 相关笔记：[[Shell-脚本备忘]]、[[Linux-常用命令速查]]

## 4. `2>&1` 含义

`2>&1` 表示将**标准错误输出（fd 2）**重定向到**标准输出（fd 1）**。

```bash
# 将命令的正常输出和错误输出都重定向到同一个文件
command > /var/log/app.log 2>&1

# 等价写法（bash 简写）
command &> /var/log/app.log
```

- `0` = stdin（标准输入）
- `1` = stdout（标准输出）
- `2` = stderr（标准错误输出）
- `&` 表示后面跟的是文件描述符，不是文件名

> 相关笔记：[[Shell-脚本备忘]]

## 5. find 命令使用

```bash
# 按文件名查找
find /path -name "*.log"

# 按大小查找
find /path -size +1M          # 大于 1M
find /path -size -10k         # 小于 10K

# 按时间查找
find /path -mtime -7          # 7 天内修改过的文件
find /path -mtime +30         # 30 天前修改的文件

# 按类型查找
find /path -type f            # 普通文件
find /path -type d            # 目录

# 组合条件
find /path -name "*.log" -size +1M -mtime -7

# 执行操作
find /path -name "*.tmp" -delete
find /path -name "*.log" -exec gzip {} \;
find /path -type d -empty -exec rmdir {} \;
```

> 相关笔记：[[Linux-常用命令速查]]、[[Linux文本处理三剑客]]