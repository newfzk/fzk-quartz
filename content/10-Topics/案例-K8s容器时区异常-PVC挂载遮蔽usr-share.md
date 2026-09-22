---
title: 案例-K8s容器时区异常-PVC挂载遮蔽usr-share
date: 2026-09-22
updated: 2026-09-22
aliases:
  - K8s 时区异常排查
  - 容器 date 显示 UTC
  - PVC 挂载遮蔽
related:
  - "[[K8s卷挂载是覆盖而非合并]]"
  - "[[TZ环境变量只是指针-时区数据才是本体]]"
  - "[[Alpine镜像不自带tzdata]]"
  - "[[apk数据库与文件系统是两套真相]]"
  - "[[docker-history是审计日志不是文件清单]]"
  - "[[容器文件消失排查-ENOENT无法区分层级]]"
  - "[[K8s-DNS-故障排查方法论]]"
tags:
  - topic/K8s
  - topic/Docker
  - topic/故障排查
  - topic/案例
  - topic/时区
status: to-review
---

# 案例：K8s 容器时区异常 —— PVC 挂载遮蔽 /usr/share

> [!abstract] 一句话根因
> `business-dev2-v2` 的 **PVC 挂载到了 `/usr/share`**，卷挂载**遮蔽**了镜像中 `/usr/share` 的全部内容，导致 `/usr/share/zoneinfo` 不可见，时区数据解析失败，`TZ=Asia/Shanghai` 静默失效退回 UTC。

| 项目 | 内容 |
|---|---|
| 报告日期 | 2026-09-22 |
| 故障现象 | 容器内 `date` 显示 UTC，Deployment 已配 `TZ=Asia/Shanghai` 但完全不生效 |
| 涉及服务 | `common-gateway`、`business-dev2-v2`（异常）；`business-sys-v2`（对照正常） |
| 命名空间 | `tech-test` |
| 排查耗时 | 6 轮交互 |

## 一、问题现象

```console
user01@t-ubuntu16-01:~$ kubectl exec -it -n tech-test common-gateway-db49bdbdb-dbxpz -- sh
/ # date
Tue Sep 22 01:31:32 UTC 2026        # 期望：09:31 CST
```

Deployment 中已明确配置：

```yaml
Environment:
  TZ: Asia/Shanghai
```

排查中又发现第二个异常服务 `business-dev2-v2` 同样显示 UTC；而 `business-sys-v2` 时区正常（CST）。三个服务的基础镜像与 Dockerfile 高度相似，**构成关键对照组**。

## 二、排查过程时间线

### 阶段 1：怀疑镜像缺少 tzdata

证据（common-gateway）：

```
/ # date                                        → Tue Sep 22 01:31:32 UTC 2026
TZ=Asia/Shanghai                                → 变量存在且值干净
NAME="Alpine Linux"  VERSION_ID=3.8.2           → Alpine + musl libc
ls /usr/share/                                   → apk/ man/ misc/（无 zoneinfo）
/etc/localtime -> /usr/share/zoneinfo/Asia/Shanghai   → 悬空软链
```

**技术原理**：`TZ` 只是**字符串指针**，libc 拿到 `Asia/Shanghai` 后需去 `/usr/share/zoneinfo/Asia/Shanghai` 读真正的时区规则文件——**找不到就静默退回 UTC，不报错**。

**结论**：common-gateway 的镜像确实缺 `tzdata`。（✅ 对其成立）

### 阶段 2：对照组打破结论 —— sys 也是 Alpine 却正常

```
--- zoneinfo ---  -rw-r--r-- 5 root root 561 Mar 25 2025 /usr/share/zoneinfo/Asia/Shanghai  ✅
```

| | sys | common-gateway |
|---|---|---|
| 基础镜像 | `eclipse-temurin:21.0.7_6-jdk-alpine-3.20` | 自建，底层 **alpine 3.8.2** |
| `tzdata` 包 | **官方 Dockerfile 中已安装** | 没有 |
| `/usr/share/zoneinfo` | 存在 | 不存在 |

**关键认知：`Alpine ≠ 有时区数据`**。`tzdata` 是独立可选包，裸 `alpine:3.x` 的 `/usr/share/` 下只有 `apk/ man/ misc/`。Temurin 官方镜像显式装了 `tzdata`。

**副作用认知**：`ln -sf /usr/share/zoneinfo/... /etc/localtime` 在源文件不存在时**不报错**，照样创建悬空软链 → 构建成功、Pod 正常启动、时区静默错误。

### 阶段 3：矛盾升级 —— 同一 `FROM` 结果却不同

`business-dev2-v2` 的 Dockerfile 与 sys 几乎完全相同（同一基础镜像，仅多一行注释），但 `zoneinfo` 不存在、软链悬空。

**逻辑上的不可能**：同一 `FROM`、同一 tag、几乎相同的 Dockerfile，产生"一个有一个没有"的结果 → 说明 `FROM` 实际解析到了不同内容。

**当时的假设（后被推翻）**：Harbor 基础镜像 tag 被覆盖推送 / 构建机缓存不一致。

**`docker history` 带来的误导**：两个镜像的 history 中**都存在** `apk add ... tzdata ...` 那行。→ 见 [[docker-history是审计日志不是文件清单]]。

### 阶段 4：apk 数据库证据推翻"未安装"假设

```
=== apk 数据库里有没有 tzdata ===
P:tzdata    V:2025b-r0    S:196130    I:1552384    T:Timezone data
=== 同批安装的其它包是否健在 ===
coreutils ✅  openssl ✅  gnupg ✅  binutils ✅  musl-locales ✅  ca-certificates ✅
```

"apk db 有完整记录（含所有 `R:`/`Z:` 校验和）"与"文件在磁盘上不存在"同时成立，只有一种解释：

> 包已成功安装，文件是在安装之后被**移除**或**遮蔽**的，且该动作**绕过了 apk**。

（若是 `apk del tzdata`，db 记录会一并消失——与观察不符。）

**结论修正**：问题不是"没装 tzdata"，而是"**装了，但文件在运行时不可见**"。

### 阶段 5：决定性证据 —— `/usr/share` 整个目录为空

```console
# dev2（异常）
ls: cannot access '/usr/share/zoneinfo': No such file or directory
drwxr-xr-x 2 root root 4096 Sep  8 00:39 .      ← link count = 2，无任何子目录
total 8   ← /usr/share 下只剩 . 和 ..

# sys（正常）
drwxr-xr-x 18 root root 4096 Jul 16  2025 /usr/share/zoneinfo
子项数: 66        1.5M  /usr/share/zoneinfo
apk  ca-certificates  fontconfig  fonts  gnupg  i18n  locale  man  misc  p11-kit  udhcpc  xml  zoneinfo
```

**dev2 丢失的不只是时区**——连 `apk/`、`man/`、`misc/` 都不见了，而这些恰是**裸 alpine 镜像的标配**（正是最早在 common-gateway 看到的那三个目录）。整个 `/usr/share` 被"掏空"。

**当时推断（方向对了一半）**：这是"过度瘦身"事故，有人执行了 `rm -rf /usr/share/*`。

**`apk audit` 的沉默陷阱**：

```
U etc/hosts / D etc/secfixes.d/ / A etc/os-release / U etc/shadow / ... / A etc/localtime
```

输出里**全是 `etc/...`，没有一条 `usr/share/...`**，尽管 `/usr/share` 已整个消失。→ 见 [[apk数据库与文件系统是两套真相]]。

**当时推断（这一半是错的）**：由 `/usr/share` 的 mtime `Sep 8 00:39` 推断"删除发生在 Sep 8，早于业务镜像构建（Sep 17），因此是 Harbor 基础镜像被覆盖为瘦身版"。**后被证伪**。

### 阶段 6：根因定位

> **用户确认最终根因**：`business-dev2-v2` 的 **PVC 挂载到了 `/usr/share` 目录**，导致 `zoneinfo` 目录被覆盖。

**所有反常证据一次性闭环**：

| 观察到的现象 | 用"挂载遮蔽"解释 |
|---|---|
| `/usr/share` 显示完全为空 | 空 PVC 挂在 `/usr/share` 上，遮蔽了全部原内容 |
| `link count = 2`、只有 `.` 和 `..` | 典型**空挂载点**特征 |
| mtime 为 `Sep 8 00:39` | **是挂载点/PV 目录自身的属性**，与镜像构建时间无关 |
| apk db 中 tzdata 记录**完好无损** | 是**遮蔽**而非**删除**，镜像层与包数据库都没被改动 |
| `apk audit` 报不出问题 | 只审计 `/etc`，且 `/usr/share` 内容在镜像层里根本没丢 |
| `TZ` 变量正确却无效 | musl 去被遮蔽的 `/usr/share/zoneinfo` 找文件 → 失败 → 静默 UTC |
| `/etc/localtime` 悬空 | 软链目标被遮蔽 |
| sys 正常 | sys 只挂了 `/usr/logs`，未触碰 `/usr/share` |
| dev2 与 sys 同 `FROM` 却不同 | **差异不在镜像，而在挂载配置** |

## 三、为什么这个故障特别隐蔽

1. **时区失效是静默的**：musl/libc 找不到时区文件时不报错，直接 UTC，无异常日志
2. **`apk` 不报错**：包数据库认为一切正常，`apk add` 也会认为"已满足"而跳过安装
3. **`apk audit` 是盲区**：默认只审计 `/etc`
4. **构建期无感知**：`ln -sf` 对不存在的源文件不报错

## 四、挂在 `/usr/share` 的爆炸半径（远不止时区）

| 被遮蔽的目录 | 影响 | 危险度 |
|---|---|---|
| `zoneinfo` | 时区、日志时间戳、定时任务 | 中 |
| `ca-certificates` | **TLS 信任链** | 🔴 高 |
| `p11-kit` | PKCS#11 / TLS 信任库 | 🔴 高 |
| `fonts` / `fontconfig` | Java AWT、字体渲染、图片/PDF | 🟡 中 |
| `locale` / `i18n` | 本地化、中文处理 | 🟡 中 |
| `xml` | XML 目录解析 | 🟡 中 |
| `gnupg` | GPG 数据 | 🟡 中 |
| `apk` | apk 共享数据 | 低 |

> [!warning] 很可能已经在别处出过故障
> `business-dev2-v2` 可能早已出现**难以解释的 TLS 握手失败、中文乱码、字体报错**，只是尚未与挂载配置联系起来。修时区只是顺带解决的第一个症状。

## 五、修复方案

### 5.1 立即修复：挂到不遮蔽系统目录的位置

```yaml
      volumeMounts:
        - name: data
          mountPath: /data          # 镜像中不存在 /data，不会遮蔽任何东西
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: <原来的 pvc 名>
```

然后修改业务配置中原来指向 `/usr/share/xxx` 的路径为 `/data`。

> [!important] 核心原则
> **挂载点应当是镜像里不存在的空目录。** 只要挂载点目录在镜像中已有内容，就会被遮蔽——对 **PVC、ConfigMap、Secret、hostPath 一律成立**。

### 5.2 若业务必须用 `/usr/share` 下的路径

挂**子目录**而非 `/usr/share` 本身，且该子目录在镜像里不存在：

```yaml
        - name: data
          mountPath: /usr/share/myapp-data    # 确认镜像中无此目录
```

⚠️ **绝对不要**挂 `/usr/share`、`/usr`、`/etc`、`/lib`、`/var` 这类系统目录的**顶层**。

### 5.3 兜底（不推荐）：initContainer 预填充

```yaml
      initContainers:
        - name: seed-share
          image: <同一镜像>
          command:
            - sh
            - -c
            - |
              [ -d /seed/zoneinfo ] || cp -a /usr/share/zoneinfo /seed/
          volumeMounts:
            - name: share
              mountPath: /seed
```

会引入运维复杂度与"卷内容与镜像版本漂移"风险，**仅作过渡**。

### 5.4 配套加固

**① Dockerfile 显式声明运行时依赖 + 构建期断言**（两个服务都建议改）：

```dockerfile
RUN apk add --no-cache tzdata \
 && ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
 && echo "Asia/Shanghai" > /etc/timezone \
 && [ -f /usr/share/zoneinfo/Asia/Shanghai ] \
 && [ -f /etc/localtime ] \
 && TZ=Asia/Shanghai date
ENV TZ=Asia/Shanghai
```

- `apk add --no-cache tzdata`：把隐式依赖变显式（Temurin 当前自带，近似 no-op，但防将来换基础镜像重演）
- `-snf`：加 `-n`，避免目标已是目录时软链被建到目录**内部**
- `[ -f ... ]` 断言：`test -f` 会**跟随软链**，悬空即失败，**让问题在构建阶段暴露**
- 最后一行在构建日志打印真实时间，肉眼可核

**② 基础镜像用 digest 固定**：`FROM ...@sha256:<digest>`（tag 可变，digest 不可变）

**③ Harbor 配置 Tag Immutability**，禁止基础镜像 tag 被覆盖推送

**④ 构建时加 `--pull`**（若必须用 tag）：`docker build --pull -t ...`

**⑤ 增加启动自检**，把静默错误变成"起不来"：

```yaml
        readinessProbe:
          exec:
            command:
              - sh
              - -c
              - test -f /usr/share/zoneinfo/Asia/Shanghai && test -s /etc/ssl/certs/ca-certificates.crt
```

或应用启动时自检 `TimeZone.getDefault()` 是否等于 `Asia/Shanghai`，不符则拒绝启动。

**⑥ 单独处理 common-gateway（独立缺陷 2）**：其镜像基于 **Alpine 3.8.2（2020 年已 EOL）**，确实未装 `tzdata`。按 ① 改 Dockerfile；无法重建时可用 hostPath 临时补回：

```yaml
      volumes:
        - name: tzdata
          hostPath:
            path: /usr/share/zoneinfo
            type: Directory
      containers:
        - name: common-gateway
          volumeMounts:
            - name: tzdata
              mountPath: /usr/share/zoneinfo      # 镜像中该目录不存在，不构成遮蔽
              readOnly: true
```

> [!note] 同一个手段，方向相反
> - common-gateway 是"挂载**补回**缺失目录"（镜像里本就没有，**安全**）
> - dev2 是"挂载**遮蔽**已有目录"（有内容，**危险**）

## 六、复盘：走弯路的三个关键点

### 弯路 1：把"遮蔽"误判为"删除"

由 mtime `Sep 8 00:39` 推断"删除发生在构建之前，因此是 Harbor tag 被瘦身"，并据此建议追查 Harbor 推送历史。

**错在哪**：那个 mtime 是**挂载点/PV 目录自身的属性**，与镜像构建时间无关。而"package db 记录完好 + 文件不可见"这个组合本该更早指向**遮蔽**——因为**删除总会留下痕迹**（apk db 变化、或 `apk audit` 报 `D:`），而遮蔽不会。

> [!important] 教训
> **排查容器内文件"消失"问题时，必须先排除挂载遮蔽，再怀疑镜像内容。**

### 弯路 2：`docker history` 被当作文件清单

### 弯路 3：验证命令的盲区没有被及时识别

| 命令 | 盲区 | 后果 |
|---|---|---|
| `ls -l /usr/share/zoneinfo/Asia/Shanghai` 报 ENOENT | **无法区分**是目录不存在还是文件不存在 | 一度误判目录整个不存在 |
| `apk audit` | 默认只审计 `/etc` | 对 `/usr/share` 的损失完全沉默 |
| `find / -name "zoneinfo*"` | `/proc/zoneinfo` 是内核 NUMA 文件，是噪音 | 干扰判读 |

## 七、快速诊断清单（下次直接用）

### 7.1 容器内时区不对 —— 四步定位

```bash
# ① 先排除挂载遮蔽（最关键，一条命令秒杀本类问题）
mount | grep -E '/usr/share|/etc/localtime|/etc/timezone'
cat /proc/mounts | grep /usr/share
findmnt /usr/share 2>/dev/null
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].volumeMounts}' | jq .

# ② 逐级确认目录本身（不能用带全路径的 ls，它无法区分层级）
ls -ld /usr/share /usr/share/zoneinfo 2>&1
ls -la /usr/share/ | head -20

# ③ 确认 TZ 变量干净
echo "TZ raw:[$TZ]"; env | grep -i '^TZ='

# ④ 确认包是否真的装过
grep -A5 '^P:tzdata' /lib/apk/db/installed || echo "未安装 tzdata"
```

### 7.2 读数判定表

| 观察结果 | 结论 | 处理 |
|---|---|---|
| `mount` 显示挂载点覆盖了 `/usr/share` 或其子目录 | **挂载遮蔽** | 改挂载点（5.1 / 5.2） |
| apk db 有 tzdata、文件不可见、`mount` 干净 | 被绕过 apk 的删除操作移除 | `apk fix` 重建文件 |
| apk db 无 tzdata 记录 | 镜像确实未安装 | Dockerfile 加 `apk add --no-cache tzdata` |
| `TZ` 变量为空 | 未注入 / Pod 是旧版本 | 检查 Pod spec，`rollout restart` |
| `TZ` 有值、文件都在、`date` 仍 UTC | 变量脏（含 `\r`/空格）或启动脚本覆盖 | `env \| grep TZ \| od -c` |

### 7.3 容器内时区全量体检脚本

```bash
kubectl exec -n tech-test <pod> -- sh -c '
  echo "== mount ==";        mount | grep -E "/usr/share|/etc/localtime"
  echo "== date ==";         date; date -u
  echo "== TZ ==";           echo "[$TZ]"
  echo "== 逐级目录 ==";     ls -ld /usr/share /usr/share/zoneinfo 2>&1
  echo "== 关键文件 ==";     ls -l /etc/localtime /usr/share/zoneinfo/Asia/Shanghai 2>&1
  echo "== apk db ==";       grep -A3 "^P:tzdata" /lib/apk/db/installed || echo "无记录"
  echo "== 强制验证 ==";     TZ=Asia/Shanghai date
'
```

### 7.4 逐层比对镜像内容（真正的"文件清单"级证据）

```bash
docker inspect --format '{{json .RootFS.Layers}}' <imageA> | jq -r '.[]' | nl > a.layers
docker inspect --format '{{json .RootFS.Layers}}' <imageB> | jq -r '.[]' | nl > b.layers
diff a.layers b.layers

# 直接看镜像最终的容器文件系统（绕开 history 与 db 的所有歧义）
id=$(docker create <image>); docker export "$id" | tar -t | grep "^usr/share/"; docker rm "$id"
```

### 7.5 全量排查：其它服务是否也挂了系统目录

```bash
kubectl get pods -n tech-test -o json | jq -r '
  .items[] | .metadata.name as $p |
  .spec.containers[] | .name as $c |
  (.volumeMounts // [])[] | "\($p)\t\($c)\t\(.mountPath)"
' | grep -E '/(usr|etc|lib|var|bin|sbin|opt|home|root)(/|$)'
```

**凡是挂载点落在 `/usr`、`/etc`、`/lib`、`/var`、`/bin`、`/sbin`、`/opt`、`/root`、`/home` 之下的，都要逐一确认该目录在镜像中是否为空。**

## 八、经验总结

### 技术层面

1. **`TZ` 只是指针，时区数据才是本体**——缺 tzdata 时 `TZ` 与 `/etc/localtime` 双双静默失效
2. **`Alpine ≠ 自带时区数据`**——`tzdata` 是独立可选包
3. **K8s 卷挂载是覆盖而非合并**——挂在已有内容的目录上，原内容被整体遮蔽（**本事故根因**）
4. **`docker history` 是审计日志，不是文件清单**——判断层内容要看 `RootFS.Layers` 的 diff_id
5. **包数据库与文件系统是两套独立真相**——`apk db` 有记录 ≠ 文件可见；`apk audit` 默认只查 `/etc`
6. **`ENOENT` 无法区分层级**——`ls` 全路径报错时必须用 `ls -ld` 逐级确认
7. **`ln -sf` 对不存在的源不报错**——这类"空软链"是构建期的沉默杀手
8. **OS 时区 ≠ JVM 时区**——JDK 自带 `tzdb.dat`，优先读 `TZ`（顺序 `-Duser.timezone` > `TZ` > `/etc/timezone`、`/etc/localtime`）。因此可能出现"`date` 是 UTC 而 Java 日志是 CST"，**二者需分别验证**

### 方法论层面

1. **对照组是最高效的排查工具**——`business-sys-v2` 这个"同样技术栈却正常"的服务，是把问题从"镜像缺包"推进到"环境差异"的关键
2. **反常证据要当线索，不要当噪音**——"包记录完好但文件不见"这个矛盾，本应更早把方向从**删除**扭向**遮蔽**
3. **理解每条验证命令的能力边界**——知道 `apk audit` 只查 `/etc`、`ls` 报错不区分层级，能省掉大量无效推理
4. **优先怀疑配置层，再怀疑制品层**——可疑对象从"基础镜像"最终收敛到"Deployment 的 volumeMounts"：**离代码越近的配置改动，往往越容易被忽视**

### 流程改进建议

| 措施 | 目的 |
|---|---|
| 禁止将卷挂载到 `/usr`、`/etc`、`/lib`、`/var` 等系统目录（Push 时准入校验，如 OPA/Kyverno 策略） | 从源头杜绝本类事故 |
| 业务 Dockerfile 显式声明运行时依赖 + 构建期断言 | 消除对基础镜像隐式内容的依赖 |
| 基础镜像使用 digest 固定；Harbor 配置 Tag Immutability | 保证构建可复现 |
| 应用启动自检关键运行时资源（时区、CA 证书），不符则拒绝启动 | 把静默错误转为显式失败 |
| 镜像发布前跑一次"运行时资源冒烟测试"（时区、TLS、字符集） | 在交付前拦截 |

## 附录 A：涉及的镜像与配置

**common-gateway（缺陷 1：镜像缺 tzdata）**

```yaml
Image: harbor.registry.com/tech-test/common-gateway:163-146348
Environment:
  TZ: Asia/Shanghai
```

底层：Alpine Linux 3.8.2（2020 年已 EOL）+ musl libc，**未安装 tzdata**。

**business-dev2-v2（缺陷 2：卷挂载遮蔽 /usr/share）—— 本次根因**

```dockerfile
FROM 192.168.168.55:8083/library/eclipse-temurin:21.0.7_6-jdk-alpine-3.20
VOLUME /tmp
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
ENV SERVER_PORT="80"
ENV LOG_HOME="/usr/logs"
ENV SPRING_PROFILES_ACTIVE="test"
ADD business-dev2-v2.jar app.jar
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -jar /app.jar" ]
```

**问题配置**：PVC 挂载到 `/usr/share`（应改为独立目录）。

**business-sys-v2（正常对照）**：挂载点仅 `/usr/logs`，**未触碰系统目录**，故一切正常。

## 参考链接

- [[K8s卷挂载是覆盖而非合并]]
- [[TZ环境变量只是指针-时区数据才是本体]]
- [[Alpine镜像不自带tzdata]]
- [[apk数据库与文件系统是两套真相]]
- [[docker-history是审计日志不是文件清单]]
- [[容器文件消失排查-ENOENT无法区分层级]]

## 附录 B：关键证据原始输出

保留原始输出以备回溯核对，其中 dev2 与 sys 的**「关键文件清单」对比**是遮蔽效应最直观的证据。

### B.1 common-gateway（Alpine 3.8.2 裸镜像，无 tzdata）

```
=== date ===        Tue Sep 22 01:41:32 UTC 2026
=== TZ env ===      TZ=Asia/Shanghai
=== os ===          NAME="Alpine Linux"  VERSION_ID=3.8.2
=== localtime ===   /etc/localtime -> /usr/share/zoneinfo/Asia/Shanghai
=== zoneinfo ===    ls: /usr/share/zoneinfo/Asia/Shanghai: No such file or directory
=== zoneinfo dir ===ls: /usr/share/zoneinfo: No such file or directory
=== libc ===        /lib/ld-musl-x86_64.so.1
```

### B.2 dev2 —— `/usr/share` 被挂载遮蔽

```
########## 1. 目录本身在不在 ##########
ls: cannot access '/usr/share/zoneinfo': No such file or directory
########## 4. /usr/share 全貌 ##########
total 8
drwxr-xr-x 2 root root 4096 Sep  8 00:39 .
drwxr-xr-x 1 root root 4096 Sep 22 02:25 ..
########## 5. 关键文件 ##########
  UTC 缺失 / Asia/Shanghai 缺失 / Asia/Tokyo 缺失 / Europe/London 缺失
  tzdata.zi 缺失 / iso3166.tab 缺失 / zone.tab 缺失
########## 6. apk audit（只覆盖 /etc）##########
U etc/hosts / D etc/secfixes.d/ / A etc/os-release / ... / A etc/localtime
########## 7. apk db 中 tzdata 记录（完整保留）##########
Z:Q1o/VN86AXw4Ym8EvZV2oKEWYzA/0=
R:Central
Z:Q1CgN/mF9voLOSyVx6+yR/FqOSWn4=
R:East-Indiana
...
```

### B.3 sys —— 正常基线

```
########## 1. 目录本身 ##########
drwxr-xr-x 18 root root 4096 Jul 16  2025 /usr/share/zoneinfo
########## 3. 条目数 / 体积 ##########
子项数: 66        1.5M  /usr/share/zoneinfo
########## 4. /usr/share 全貌 ##########
apk  ca-certificates  fontconfig  fonts  gnupg  i18n  locale
man  misc  p11-kit  udhcpc  xml  zoneinfo
########## 5. 关键文件 ##########
  UTC 存在 / Asia/Shanghai 存在 / Asia/Tokyo 存在 / Europe/London 存在
  tzdata.zi 缺失 / iso3166.tab 存在 / zone.tab 存在
```
