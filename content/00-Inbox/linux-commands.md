# Linux Commands

> linux 常用命令备忘

- [Linux Commands](#linux-commands)
  - [1 文件目录相关](#1-文件目录相关)
    - [1.1 ls](#11-ls)
    - [1.2 cp](#12-cp)
    - [1.3 basename](#13-basename)
    - [1.3 dirname](#13-dirname)
    - [1.3 stat](#13-stat)
    - [1.3 查看本内核支持哪些文件系统](#13-查看本内核支持哪些文件系统)
  - [2 系统性能](#2-系统性能)
    - [2.1 uptime 查看系统负载信息](#21-uptime-查看系统负载信息)
    - [2.2 vmstat 显示虚拟内存状态](#22-vmstat-显示虚拟内存状态)
    - [2.3 dmesg 查看系统启动信息](#23-dmesg-查看系统启动信息)
  - [3 进程监控](#3-进程监控)
  - [4  网络](#4--网络)
    - [4.1 netstat 查看网络系统状态信息](#41-netstat-查看网络系统状态信息)
      - [查看端口是否被占用](#查看端口是否被占用)
      - [网络连接状态详解](#网络连接状态详解)
      - [常见标志位](#常见标志位)
  - [5 用户、权限](#5-用户权限)
    - [newgrp 以新的组身份打开新shell](#newgrp-以新的组身份打开新shell)
    - [id 用户信息和组信息输出](#id-用户信息和组信息输出)
  - [6 日志](#6-日志)
  - [7 系统信息](#7-系统信息)
    - [7.1 uname 打印系统信息](#71-uname-打印系统信息)
    - [lscpu 查看cpu信息](#lscpu-查看cpu信息)
  - [常用环境变量](#常用环境变量)
    - [BASHPID](#bashpid)

## 1 文件目录相关

### 1.1 ls

- `-a` 全部文件，包括隐藏文件、`.`、`..`；
- `-A` 全部文件，不包括`.`、`..`；
- `-F` 根据文件类型，在文件名后添加相应字符
  - `*` 表示可执行文件，如 `hello.sh*`；
  - `/` 表示目录，如`dir-test/`；
  - `=` 表示 socket 文件；
  - `|` 表示 FIFO 文件；
- `-i` 列出各文件的索引；
- `-n` 类似于`-l` 只不过用 UID 和 GID （数字）表示用户和组；
- `-r` 反向排序输出结果，例如默认文件名由小到大，反向则是由大到小；
- `-R` 与子目录内容一起列出，不好用，更推荐用`tree`；
- `-S` 按文件大小排序，large first；
- `-t` 按修改时间排序 newest first ；
- `--full-time` 显示完整时间 包含年月日时分；
- `--time={atime,ctime}` 默认显示内容更改时间（`modification time`），`atime` 显示访问时间（`access time`），`ctime` 显示更改时间（`change time`）；
- `-l --time-style=long-iso` 以更舒适的格式显示时间，如`2024-10-14 22:54`；
- `--color={always,auto,never}` 显示颜色，`always` 总是显示颜色，`auto` 根据终端自动判断，`never` 不显示颜色；

### 1.2 cp

> - `cp [options] source destination` # 复制源文件并重命名为目标文件名
>
> - `cp [options] source1 [source2 source3 ...] directory` # 复制一个或多个文件至目标目录中

- `-a` 相当于`-pdr` 保留文件属性 若源文件为链接文件则只复制链接文件属性 递归复制；
- `-d` 如果原文件为链接文件的属性，则复制链接文件属性而非文件本身；
  - 默认复制链接文件时，会直接得到软连接执行的目标文件，而不是链接文件属性，即链接文件的复制必须通过`-a`或`-d`进行；
- `-l` 建立硬链接文件（hard link），而非复制文件本身；
- `-s` 建立软连接文件（symbolic link），即快捷方式；
- `-p` 与文件属性一起复制，而不是默认属性；
- `-u` update 更新式复制，进原文件比目标文件更新，或目标文件不存在时，会进行复制；copy only when the SOURCE file is newer than the destination file or when the destination file is missing
- `-i` 如果目标文件已经存在，覆盖前会先询问是否覆盖;
- `-f` 强制，忽略重复或其他问题；
- `-r` 递归复制，复制目录及其子目录；

> 软连接（`file1.s` `file1.s1`） 硬链接（`file.l` `file.l1`）区别如下：
>
> - 软链接类似于快捷方式，新文件以路径的方式指向目标文件
>
> - 硬链接类似于原文件的别名，当删除原文件或新硬链接文件之一时，另一个可以正常使用；

```sh
~/test.d ⌚ 20:42:08
$ ls -alFi --time-style=long-iso
total 20
542495 drwxr-xr-x  2 root root 4096 2024-10-15 20:42 ./
130818 drwx------ 15 root root 4096 2024-10-15 20:42 ../
542972 -rw-r--r--  2 root root    5 2024-10-15 20:38 file1
542972 -rw-r--r--  2 root root    5 2024-10-15 20:38 file1.l
542975 lrwxrwxrwx  1 root root    5 2024-10-15 20:37 file1.s -> file1
543893 lrwxrwxrwx  1 root root    5 2024-10-15 20:37 file1.s1 -> file1
```

### 1.3 basename 

get file name without dir name.

> we also can get basename from url.
> just like `http://nginx.org/download/nginx-1.18.9.tar.gz`, it's basename is `nginx-1.18.9.tar.gz`

### 1.3 dirname

get dir name without file name.

### 1.3 stat

see file's state, just like access time, modify time, change time.


### 1.3 查看本内核支持哪些文件系统

```sh
~/files ⌚ 13:34:39  # uname -r 查看内核版本（release）
$ ls /lib/modules/`uname -r`/kernel/fs        
9p    affs  autofs  bfs             btrfs       ceph  coda    dlm  erofs  f2fs  freevxfs  fuse  hfs      hpfs   jffs2  ksmbd  minix  nfs         nfsd    nls   ntfs3  omfs      overlayfs  qnx4  quota     romfs       smbfs_common  ubifs  ufs     xfs
adfs  afs   befs    binfmt_misc.ko  cachefiles  cifs  cramfs  efs  exfat  fat   fscache   gfs2  hfsplus  isofs  jfs    lockd  netfs  nfs_common  nilfs2  ntfs  ocfs2  orangefs  pstore     qnx6  reiserfs  shiftfs.ko  sysv          udf    vboxsf  zonefs
```

## 2 系统性能

vmstat mpstat pidstat iostat free sar 等

### 2.1 uptime 查看系统负载信息

```sh
~ ⌚ 16:16:34
$ uptime
 16:11:08 up 15 days, 13:24,  2 users,  load average: 1.00, 0.64, 0.41
# 现在时间 系统已经运行了多长时间 目前有多少登陆用户 系统在过去的1分钟 5分钟 15分钟内的平均负载
```

> 如果每个CPU内核的当前活动进程数不大于3的话，那么系统的性能是良好的。如果每个CPU内核的任务数大于5，那么这台机器的性能有严重问题。
>
> 如果你的linux主机是1个双核CPU的话，当Load Average 为6的时候说明机器已经被充分使用了。

### 2.2 vmstat 显示虚拟内存状态

### 2.3 dmesg 查看系统启动信息

```sh
# 查看系统启动信息
~ ⌚ 16:16:34
$ dmesg | head
[    0.000000] Linux version 5.15.0-117-generic (buildd@lcy02-amd64-102) (gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #127-Ubuntu SMP Fri Jul 5 20:13:28 UTC 2024 (Ubuntu 5.15.0-117.127-generic 5.15.158)
[    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-5.15.0-117-generic root=UUID=a9699f99-5614-4444-be92-d2ef6cfdbaf6 ro vga=792 console=tty0 console=ttyS0,115200n8 net.ifnames=0 noibrs iommu=pt crashkernel=0M-1G:0M,1G-4G:192M,4G-128G:384M,128G-:512M nvme_core.io_timeout=4294967295 nvme_core.admin_timeout=4294967295
[    0.000000] KERNEL supported cpus:
[    0.000000]   Intel GenuineIntel
[    0.000000]   AMD AuthenticAMD
[    0.000000]   Hygon HygonGenuine
[    0.000000]   Centaur CentaurHauls
[    0.000000]   zhaoxin   Shanghai  
[    0.000000] BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009ffff] usable

# 查看硬盘信息
~ ⌚ 16:19:34
$ dmesg |grep vda 
[    1.046864] virtio_blk virtio1: [vda] 83886080 512-byte logical blocks (42.9 GB/40.0 GiB)
[    1.123699]  vda: vda1 vda2 vda3
[    6.522625] EXT4-fs (vda3): mounted filesystem with ordered data mode. Opts: (null). Quota mode: none.
[    7.361488] EXT4-fs (vda3): re-mounted. Opts: (null). Quota mode: none.
```

## 3 进程监控

ps top lsof

## 4  网络

ping netstat iptables ipvs

### 4.1 netstat 查看网络系统状态信息

netstat 查看网络系统状态信息 示例如下：

```sh
~ ⌚ 16:45:20
$ netstat                 
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0    164 fzk:121                 124.202.193.6:13737     ESTABLISHED
tcp        0      0 fzk:43330               100.100.30.25:http      ESTABLISHED
tcp        0      0 fzk:43010               100.100.18.120:http     TIME_WAIT  
udp        0      0 fzk:56457               100.100.2.138:domain    ESTABLISHED
udp        0      0 fzk:60707               100.100.2.136:domain    ESTABLISHED
udp        0      0 fzk:52944               100.100.2.138:domain    ESTABLISHED
Active UNIX domain sockets (w/o servers)
Proto RefCnt Flags       Type       State         I-Node   Path
unix  2      [ ]         DGRAM                    629848   /run/user/0/systemd/notify
unix  2      [ ]         DGRAM      CONNECTED     20932    /run/chrony/chronyd.sock
unix  3      [ ]         DGRAM      CONNECTED     17648    /run/systemd/notify
unix  2      [ ]         DGRAM                    17276    /run/systemd/journal/syslog
unix  10     [ ]         DGRAM      CONNECTED     17663    /run/systemd/journal/dev-log
unix  8      [ ]         DGRAM      CONNECTED     17280    /run/systemd/journal/socket
unix  3      [ ]         STREAM     CONNECTED     20621    
unix  3      [ ]         STREAM     CONNECTED     261581   
```

常用参数：

```text
-a (--all) 列出所有的Socket（连线情况） （tcp的 udp的 unix的 监听状态的 其他状态的）
-t (--tcp) 列出TCP协议的Socket
-u (--udp) 列出UDP协议的Socket
-x (--unix) 列出所有UNIX的Socket信息
-l (--listening) 列出处于监听（LISTEN）状态的Socket
-n (--numeric) 直接使用ip地址和端口号，而不是使用主机名和服务名称（默认）
-p (--programs) 显示正在使用Socket的程序识别码(pid)和程序名称(Program name)；

-r (--route) 显示核心路由信息 (-rn 以数字格式显示 不查询主机名)
```

#### 查看端口是否被占用

```sh
~ ⌚ 17:36:09
$ netstat -alnp | grep ':80' 
tcp        0      0 172.16.80.122:43330     100.100.30.25:80        ESTABLISHED 76772/AliYunDun     
tcp        0      0 172.16.80.122:44758     100.100.18.120:80       TIME_WAIT   -  
```

#### 网络连接状态详解

共有12中可能的状态，前面11种是按照TCP连接建立的三次握手和TCP连接断开的四次挥手过程来描述的：

- LISTEN：首先服务端需要打开一个socket进行监听，状态为 LISTEN，侦听来自远方TCP端口的连接请求 ；
- SYN_SENT：客户端通过应用程序调用connect进行active open，于是客户端tcp发送一个SYN以请求建立一个连接，之后状态置为 SYN_SENT，在发送连接请求后等待匹配的连接请求；
- SYN_RECV：服务端应发出ACK确认客户端的 SYN，同时自己向客户端发送一个SYN，之后状态置为，在收到和发送一个连接请求后等待对连接请求的确认；
- ESTABLISHED：代表一个打开的连接，双方可以进行或已经在数据交互了， 代表一个打开的连接，数据可以传送给用户；
- FIN_WAIT1：主动关闭(active close)端应用程序调用close，于是其TCP发出FIN请求主动关闭连接，之后进入FIN_WAIT1状态， 等待远程TCP的连接中断请求，或先前的连接中断请求的确认；
- CLOSE_WAIT：被动关闭(passive close)端TCP接到FIN后，就发出ACK以回应FIN请求(它的接收也作为文件结束符传递给上层应用程序)，并进入CLOSE_WAIT， 等待从本地用户发来的连接中断请求；
- FIN_WAIT2：主动关闭端接到ACK后，就进入了 FIN-WAIT-2，从远程TCP等待连接中断请求；
- LAST_ACK：被动关闭端一段时间后，接收到文件结束符的应用程 序将调用CLOSE关闭连接，这导致它的TCP也发送一个 FIN,等待对方的ACK.就进入了LAST-ACK，等待原来发向远程TCP的连接中断请求的确认；
- TIME_WAIT:在主动关闭端接收到FIN后，TCP 就发送ACK包，并进入TIME-WAIT状态，等待足够的时间以确保远程TCP接收到连接中断请求的确认；
- CLOSING: 比较少见，等待远程TCP对连接中断的确认；
- CLOSED: 被动关闭端在接受到ACK包后，就进入了closed的状态，连接结束，没有任何连接状态；
- UNKNOWN：未知的Socket状态；

#### 常见标志位

- SYN: (同步序列编号,Synchronize Sequence Numbers)该标志仅在三次握手建立TCP连接时有效。表示一个新的TCP连接请求。
- ACK: (确认编号,Acknowledgement Number)是对TCP请求的确认标志,同时提示对端系统已经成功接收所有数据。
- FIN: (结束标志,FINish)用来结束一个TCP回话.但对应端口仍处于开放状态,准备接收后续数据。

## 5 用户、权限

### newgrp 以新的组身份打开新shell

newgrp 命令通过更改用户真实有效的组 ID，将用户登录到新组。用户保留登录状态，当前目录不变。

**执行 newgrp 时始终会将当前 shell 替换成新的 shell** 因此，脚本中通常不使用次命令。

```shell
newgrp docker
```

### id 用户信息和组信息输出

```shell
root@tmanager:/etc/ansible/tools# newgrp docker
root@tmanager:/etc/ansible/tools# id -a
uid=0(root) gid=1001(docker) groups=1001(docker),0(root)

root@tmanager:/etc/ansible/tools# id -n
id: cannot print only names or real IDs in default format
root@tmanager:/etc/ansible/tools# id -un
root
root@tmanager:/etc/ansible/tools# id -gn
docker
root@tmanager:/etc/ansible/tools# id -Gn
docker root

root@tmanager:/etc/ansible/tools# id -r
id: cannot print only names or real IDs in default format
root@tmanager:/etc/ansible/tools# id -ru
0
root@tmanager:/etc/ansible/tools# id -gr
1001
root@tmanager:/etc/ansible/tools# id -Gr
1001 0

root@tmanager:/etc/ansible/tools# id --help
Usage: id [OPTION]... [USER]
Print user and group information for the specified USER,
or (when USER omitted) for the current user.

  -a             ignore, for compatibility with other versions
  -u, --user     print only the effective user ID
  -g, --group    print only the effective group ID
  -G, --groups   print all group IDs
  -n, --name     print a name instead of a number, for -ugG
  -r, --real     print the real ID instead of the effective ID, with -ugG
  -Z, --context  print only the security context of the process
  -z, --zero     delimit entries with NUL characters, not whitespace;
                   not permitted in default format
      --help     display this help and exit
      --version  output version information and exit
```

## 6 日志

## 7 系统信息

### 7.1 uname 打印系统信息

```sh
~/files ⌚ 13:34:23
$ uname --help
Usage: uname [OPTION]...
Print certain system information.  With no OPTION, same as -s.

  # -a 所有信息 
  -a, --all                print all information, in the following order,
                             except omit -p and -i if unknown:
  # -s 内核名称 如 Linux
  -s, --kernel-name        print the kernel name
  # hostname
  -n, --nodename           print the network node hostname
  # 内核发行版本号 如 5.15.0-117-generic 主版本号、次版本号：5.15.0 修订号：117 generic：发行版标识
  -r, --kernel-release     print the kernel release
  # 内核版本字符串-内核版本的更详细信息 #127-Ubuntu SMP Fri Jul 5 20:13:28 UTC 2024
  # 包含内核版本号：#127 系统对称多处理（SMP）内核：Ubuntu SMP 编译日期：Fri Jul 5 20:13:28 UTC 2024 
  -v, --kernel-version     print the kernel version
  # 机器硬件架构 通常是CPU架构
  -m, --machine            print the machine hardware name
  # 处理器类型 可能会输出 i386 i686 哪怕系统已经是x86_64
  -p, --processor          print the processor type (non-portable)
  # 硬件平台信息 通常与硬件架构相关
  -i, --hardware-platform  print the hardware platform (non-portable)
  # 输出操作系统名称
  -o, --operating-system   print the operating system
      --help     display this help and exit
      --version  output version information and exit

GNU coreutils online help: <https://www.gnu.org/software/coreutils/>
Full documentation <https://www.gnu.org/software/coreutils/uname>
or available locally via: info '(coreutils) uname invocation'

# 不同参数的输出示例如下：
~/files ⌚ 13:38:23
$ uname   
Linux

~/files ⌚ 13:35:45
$ uname -a    
Linux fzk 5.15.0-117-generic #127-Ubuntu SMP Fri Jul 5 20:13:28 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux

~/files ⌚ 13:37:04
$ uname -s
Linux

~/files ⌚ 13:37:42
$ uname -n
fzk

~/files ⌚ 13:37:51
$ uname -r
5.15.0-117-generic

~/files ⌚ 13:37:55
$ uname -v
#127-Ubuntu SMP Fri Jul 5 20:13:28 UTC 2024

~/files ⌚ 13:37:59
$ uname -m
x86_64

~/files ⌚ 13:38:07
$ uname -p
x86_64

~/files ⌚ 13:38:12
$ uname -i
x86_64

~/files ⌚ 13:38:18
$ uname -o
GNU/Linux
```

### lscpu 查看cpu信息

```shell
root@rscloud:/home/rs10user/sbc/5-docker-images# lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                2
On-line CPU(s) list:   0,1
Thread(s) per core:    1
Core(s) per socket:    1
Socket(s):             2
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 85
Model name:            Intel(R) Xeon(R) Silver 4116 CPU @ 2.10GHz
Stepping:              4
CPU MHz:               2095.078
BogoMIPS:              4190.15
Hypervisor vendor:     VMware
Virtualization type:   full
L1d cache:             32K
L1i cache:             32K
L2 cache:              1024K
L3 cache:              16896K
NUMA node0 CPU(s):     0,1
Flags:                 fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts nopl xtopology tsc_reliable nonstop_tsc pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single ibrs ibpb stibp kaiser fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 invpcid rtm mpx avx512f rdseed adx smap clflushopt clwb avx512cd xsaveopt xsavec arat pku arch_capabilities
```

## 常用环境变量

### BASHPID

`BASHPID` 是一个特殊变量，用于表示 当前 Bash 进程的进程 ID（PID）。它的行为与 `$$` 变量类似但有重要区别，尤其在涉及子 Shell 时。以下是详细解释：

- BASHPID 用于获取当前 Bash 进程的实际 PID，在子 Shell 或并发任务中更精确。
- $$ 表示脚本或 Shell 的初始 PID，在子 Shell 中不会变化。

```shell
root@tmanager:/etc/ansible/tools# echo $BASHPID
22758
```
