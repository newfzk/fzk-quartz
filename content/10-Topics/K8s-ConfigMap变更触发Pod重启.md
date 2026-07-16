---
tags:
  - topic/K8s
  - topic/Helm
  - topic/DevOps
status: to-review
---

# K8s ConfigMap 变更触发 Pod 重启

ConfigMap 更新后，挂载它的 Pod **不会自动重启**。实现"ConfigMap 变更 → 自动重启对应工作负载"有三种主流方案。

## 方案一：Reloader 控制器（推荐）

[Reloader](https://github.com/stakater/Reloader) 专为解决此问题设计，监视 ConfigMap/Secret 变化，自动触发关联工作负载的滚动重启。

**安装**：
```bash
helm repo add stakater https://stakater.github.io/stakater-charts
helm install reloader stakater/reloader --namespace kube-system
```

**使用**：为 Deployment 添加注解即可。
```yaml
metadata:
  annotations:
    reloader.stakater.com/auto: "true"  # 监视引用的所有 ConfigMap/Secret
    # 或指定特定 ConfigMap
    configmap.reloader.stakater.com/reload: "my-configmap"
```

**优点**：完全自动、精准（只重启关联工作负载）、支持外部变更、不侵入 Chart 模板。
**缺点**：需额外安装控制器。

## 方案二：Helm Checksum 注解（轻量方案）

将 ConfigMap 内容散列值作为 Deployment 注解，`helm upgrade` 时 checksum 变化触发滚动更新。

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

**优点**：无需额外组件，Helm 社区标准模式。
**缺点**：仅 `helm upgrade` 时触发，不支持外部变更，需修改模板。

## 方案三：Wave 控制器

[Wave](https://github.com/wave-k8s/wave) 是类似 Reloader 的控制器。

```bash
helm repo add wave-k8s https://wave-k8s.github.io/wave/
helm install wave wave-k8s/wave
```

注解：`wave.k8s.io/hash: <configmap-name>`

## 方案对比

| 特性 | Reloader（推荐） | Checksum 注解 | Wave |
|:---|:---:|:---:|:---:|
| 自动化程度 | 完全自动 | 仅 `helm upgrade` | 完全自动 |
| 依赖 Helm | 否 | 是 | 否 |
| 修改 Chart | 仅加注解 | 需改模板 | 仅加注解 |
| 支持外部变更 | 支持 | 不支持 | 支持 |
| 额外组件 | 需安装控制器 | 不需要 | 需安装控制器 |

## 选择建议

- **推荐 Reloader**：一劳永逸，不侵入 Chart 逻辑，还能感知外部变更。
- **不想引入新控制器**：Checksum 注解是轻量替代方案，前提是 ConfigMap 变更总伴随 `helm upgrade`。

> 如果应用支持热加载（如 Nginx、Spring Cloud Config），也可通过挂载 Volume 让应用自行感知变化，但这需要应用本身支持。

## 相关笔记

- [[Helm-upgrade命令]]
- [[Helm-Chart模板概述]]
