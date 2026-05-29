# 排查 `df -h` 与 `du -h` 磁盘空间显示不一致的问题

## 背景

- Linux 服务器，LVM 卷 `/dev/mapper/rsdata-dmdata` 挂载在 `/rscloud/dmdata`
- 已用 `rm -f` 删除 100G+ 达梦数据库审计文件
- `df -h` 仍显示 100%（186G used / 196G total）
- `du -h` 仅显示约 37G

## 根因分析

**核心原因：进程持有已删除文件的文件句柄（File Descriptor）。**

在 Linux 中：
1. `rm -f` 仅删除**目录项（directory entry）**，即文件名与 inode 的映射关系
2. 如果有进程已经打开了该文件（open()），则该文件的 **inode 及数据块仍被引用**
3. 磁盘空间只有在所有持有该文件句柄的进程都 close() 后才会真正释放
4. `df` 统计的是文件系统层面的**已分配块（allocated blocks）**，包括被删除但仍有进程引用的文件
5. `du` 统计的是通过目录树可遍历到的文件，看不到已删除但仍被引用的文件

所以：**df ≈ du + 已删除但进程仍在引用的文件大小**。

## 排查步骤

### Step 1：确认是否存在被进程持有的已删除文件

```bash
# 查看 /rscloud/dmdata 下所有被进程打开但已删除的文件
lsof +L1 /rscloud/dmdata
```

或者：

```bash
# 更通用的方式
lsof -nP | grep '(deleted)' | grep '/rscloud/dmdata'
```

### Step 2：若 lsof 不可用，检查 /proc 文件系统

```bash
# 查看所有进程的 fd 信息
find /proc/*/fd -type l -lname '*/rscloud/dmdata/* (deleted)' 2>/dev/null
```

或更精确地：

```bash
ls -la /proc/*/fd/ 2>/dev/null | grep '/rscloud/dmdata.*(deleted)'
```

### Step 3：确认是哪个进程占用了句柄

在 Step 1/2 的输出中，会看到类似：

```
COMMAND   PID   USER   FD   TYPE  DEVICE   SIZE   NODE   NAME
dmserver  12345 dmdba  11w   REG   dm-0     100G   654321 /rscloud/dmdata/data/... (deleted)
```

重点关注：
- **PID**：进程 ID
- **COMMAND**：进程名称
- **FD**：文件描述符（w 表示写模式打开）

### Step 4：释放磁盘空间

有两种方式：

#### 方式一：重启持有文件的进程（推荐）

```bash
# 如果确认是达梦数据库进程
systemctl restart DmService     # 或对应的达梦服务名
```

或者手动 kill（谨慎使用）：

```bash
# 先确认 PID，然后向进程发送信号
kill -HUP <PID>     # 优先尝试 SIGHUP，部分进程会重新加载而不退出
# 或
kill <PID>          # 正常终止进程
```

#### 方式二：清空文件而非删除（恢复后生产环境建议）

如果未来仍有类似问题，正确的做法是**清空审计文件**而非删除：

```bash
# 清空文件内容而非删除文件
: > /path/to/audit_file.log
# 或
cat /dev/null > /path/to/audit_file.log
# 或
truncate -s 0 /path/to/audit_file.log
```

这样文件 inode 不变，进程句柄不受影响，空间立即释放。

### Step 5：验证空间是否释放

```bash
df -h /rscloud/dmdata
```

## 达梦数据库审计文件管理建议（预防措施）

1. **开启审计文件自动清理**：配置达梦数据库的定期清理策略
2. **设置审计文件大小限制**：避免单个审计文件无限增长
3. **使用 truncate 方式清理**：避免 `rm -f` 后句柄残留
4. **监控磁盘使用率**：设置告警阈值（如 80%），及早发现

## 参考资料

- `lsof`：List Open Files，查看进程打开的文件列表
- `/proc/*/fd/`：每个进程的文件描述符目录
- Linux 文件删除原理：`rm` 删除目录项 → 文件引用计数归零 → 空间真正释放
