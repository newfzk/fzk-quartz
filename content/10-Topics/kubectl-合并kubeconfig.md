---
tags:
  - topic/K8s
  - topic/kubectl
status: to-review
---

# kubectl 合并 kubeconfig

通过 `KUBECONFIG` 环境变量合并多个 kubeconfig 文件，再用 `--flatten` 生成自包含配置。

## 命令

```bash
KUBECONFIG=~/.kube/config:/path/to/target-kubeconfig.yaml kubectl config view --flatten > /tmp/merged-config && mv /tmp/merged-config ~/.kube/config
```

## 说明

| 步骤 | 作用 |
|:---|:---|
| `KUBECONFIG=...` | 将多个 kubeconfig 路径用 `:` 分隔，合并为一个临时视图 |
| `--flatten` | 内联所有引用（如证书文件内容），生成自包含配置 |
| 写回 `~/.kube/config` | 用合并后的配置覆盖默认配置 |

## 切换集群

```bash
kubectl config get-contexts          # 查看所有 context
kubectl config use-context <context> # 切换目标集群
```

## 相关笔记

- [[K8s-DNS-故障排查方法论]]
