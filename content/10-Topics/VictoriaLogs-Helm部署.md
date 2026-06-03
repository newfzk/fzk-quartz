---
title: VictoriaLogs Helm方式部署
type: basic-note
date: 2025-05-21
tags: victorialogs, helm, k8s
---

# VictoriaLogs Helm方式部署

> 官方文档： <https://docs.victoriametrics.com/victorialogs/quickstart/#helm-charts>

## 单节点部署

### 先决条件

git、kubectl、helm、helm-doc

> helm-doc 应该没用 我不需要他重新生成readme

### 图表详情

- victorialogs 单节点部署
- 用于从Pod中收集日志的部署向量

### 安装部署

[github helm-chart](https://github.com/VictoriaMetrics/helm-charts)

[离线文件](./helm-chart/victoria-logs-single-0.9.8.tgz)

#### 基础环境准备

依赖创建好的pvc（或storageClass 目前rs10-k8s集群没有）

1. `victorialogs-namespace.yaml`
2. `victorialogs-pvc.yaml`

#### 修改chart中的默认值

通过脚本修改参数（`victorialogs-install-by-helm.sh`）并执行`helm install`

##### install报错1：k8s版本过低

`Error: INSTALLATION FAILED: chart requires kubeVersion: >=1.25.0-0 which is incompatible with Kubernetes v1.18.3`

**强行**修改`Chart.yaml`中的`kubeVersion`要求 将其改为现在用的`v1.18`

##### install报错2: 命名空间参数位置错误导致命名空间识别错误

```log
部署命令为: 
helm install victorialogs-single . -n victorialogs 
    --set server.image.registry=harbor.registry.com 
    --set server.image.repository=library/victoria-logs 
    --set server.image.tag=v1.21.0-victorialogs 
    --set server.retentionDiskSpaceUsage=10 
    --set server.persistentVolume.name=victorialogs-pvc 
    --set server.persistentVolume.existingClaim=victorialogs-pvc 
    --set server.persistentVolume.size=10Gi 
    --set server.service.nodePort=39428 
    --set server.service.type=NodePort

Error: 
INSTALLATION FAILED: 
    Unable to continue with install: 
    could not get information about the resource Service 
        "victorialogs-single-victoria-logs-single-server" 
    in namespace 
        "victorialogs --set server.image.registry=harbor.registry.com --set server.image.repository=library/victoria-logs --set server.image.tag=v1.21.0-victorialogs --set server.retentionDiskSpaceUsage=10 --set server.persistentVolume.name=victorialogs-pvc --set server.persistentVolume.existingClaim=victorialogs-pvc --set server.persistentVolume.size=10Gi --set server.service.nodePort=39428 --set server.service.type=NodePort": invalid namespace "victorialogs --set server.image.registry=harbor.registry.com --set server.image.repository=library/victoria-logs --set server.image.tag=v1.21.0-victorialogs --set server.retentionDiskSpaceUsage=10 --set server.persistentVolume.name=victorialogs-pvc --set server.persistentVolume.existingClaim=victorialogs-pvc --set server.persistentVolume.size=10Gi --set server.service.nodePort=39428 --set server.service.type=NodePort": [may not contain '/']
```

##### install报错解决 改为直接使用helm命令 不在借助shell脚本

```shell
root@tmanager:~/victorialogs/victoria-logs-single# helm install -n victorialogs \
>     --set server.image.registry=harbor.registry.com \
>     --set server.image.repository=library/victoria-logs \
gs-pvc \
    --set server.persistentVolume.existingClaim=victori>     --set server.image.tag=v1.21.0-victorialogs \
>     --set server.retentionDiskSpaceUsage=10 \
>     --set server.persistentVolume.name=victorialogs-pvc \
>     --set server.persistentVolume.existingClaim=victorialogs-pvc \
>     --set server.persistentVolume.size=10Gi \
>     --set server.service.nodePort=39428 \
>     --set server.service.type=NodePort \
>     victorialogs-single .
NAME: victorialogs-single
LAST DEPLOYED: Mon May 26 11:28:23 2025
NAMESPACE: victorialogs
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The VictoriaLogs write api can be accessed via port 9428 on the following DNS name from within your cluster:
    victorialogs-single-victoria-logs-single-server-0.victorialogs-single-victoria-logs-single-server.victorialogs.svc.cluster.local.

Logs Ingestion:
  Get the Victoria Logs service URL by running these commands in the same shell:
    export NODE_PORT=$(kubectl get --namespace victorialogs -o jsonpath="{.spec.ports[0].nodePort}" services victorialogs-single-victoria-logs-single-server)
    export NODE_IP=$(kubectl get nodes --namespace victorialogs -o jsonpath="{.items[0].status.addresses[0].address}")
    echo http://$NODE_IP:$NODE_PORT

  Write URL inside the kubernetes cluster:
    http://victorialogs-single-victoria-logs-single-server.victorialogs.svc.cluster.local.:9428<protocol-specific-write-endpoint>

  All supported write endpoints can be found at https://docs.victoriametrics.com/victorialogs/data-ingestion/

Read Data:
  The following URL can be used to query data:
    http://victorialogs-single-victoria-logs-single-server.victorialogs.svc.cluster.local.:9428
```

##### 启动报错

> commitId `f2effe24e3402410e6aba5c1022137eaecda31ec`

```shell
root@tmanager:~/victorialogs/victoria-logs-single# kubectl logs -n victorialogs victorialogs-single-victoria-logs-single-server-0 --previous
{"ts":"2025-05-26T03:31:09.660Z","level":"info","caller":"VictoriaMetrics/lib/logger/flag.go:12","msg":"build version: victoria-logs-20250425-174411-tags-v1.21.0-victorialogs-0-g9d82de9385"}
{"ts":"2025-05-26T03:31:09.660Z","level":"info","caller":"VictoriaMetrics/lib/logger/flag.go:13","msg":"command-line flags"}
{"ts":"2025-05-26T03:31:09.660Z","level":"info","caller":"VictoriaMetrics/lib/logger/flag.go:20","msg":"  -envflag.enable=\"true\""}
{"ts":"2025-05-26T03:31:09.660Z","level":"info","caller":"VictoriaMetrics/lib/logger/flag.go:20","msg":"  -envflag.prefix=\"VM_\""}
{"ts":"2025-05-26T03:31:09.660Z","level":"info","caller":"VictoriaMetrics/lib/logger/flag.go:20","msg":"  -httpListenAddr=\":9428\""}
{"ts":"2025-05-26T03:31:09.660Z","level":"info","caller":"VictoriaMetrics/lib/logger/flag.go:20","msg":"  -loggerFormat=\"json\""}
{"ts":"2025-05-26T03:31:09.660Z","level":"info","caller":"VictoriaMetrics/lib/logger/flag.go:20","msg":"  -retention.maxDiskSpaceUsageBytes=\"10GiB\""}
{"ts":"2025-05-26T03:31:09.660Z","level":"info","caller":"VictoriaMetrics/lib/logger/flag.go:20","msg":"  -retentionPeriod=\"1\""}
{"ts":"2025-05-26T03:31:09.660Z","level":"info","caller":"VictoriaMetrics/lib/logger/flag.go:20","msg":"  -storageDataPath=\"/storage\""}
{"ts":"2025-05-26T03:31:09.660Z","level":"info","caller":"VictoriaMetrics/app/victoria-logs/main.go:42","msg":"starting VictoriaLogs at \"[:9428]\"..."}
{"ts":"2025-05-26T03:31:09.660Z","level":"info","caller":"VictoriaMetrics/app/vlstorage/main.go:111","msg":"opening storage at -storageDataPath=/storage"}
{"ts":"2025-05-26T03:31:09.661Z","level":"panic","caller":"VictoriaMetrics/lib/fs/fs.go:362","msg":"FATAL: cannot create lock file: cannot create lock file \"/storage/flock.lock\": open /storage/flock.lock: permission denied; make sure a single process has exclusive access to \"/storage\""}
```

##### 修改pv挂载文件夹的权限

查阅`values.yaml`可知 容器默认用户id为`1000` 具体如下：

```yaml
  # -- Pod's security context. Details are [here](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
  podSecurityContext:
    enabled: true
    fsGroup: 2000
    # 非root用户
    runAsNonRoot: true
    # 用户id
    runAsUser: 1000
```

据此 登陆 nfs服务器 修改对应文件夹的权限

```shell
cd /rscloud/nfs && chown -R 1000:1000 victorialogs-pv
```

至此 pod 启动成功 对外端口为 `39428`

尝试访问 `http://ip:39428` 访问成功 部署初步成功

## 相关笔记

- [[VictoriaLogs-安装]]
- [[VictoriaLogs-概述]]