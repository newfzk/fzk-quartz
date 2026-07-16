---
tags:
  - topic/Helm
  - topic/Harbor
  - topic/OCI
  - topic/DevOps
status: to-review
---

# Helm-push-OCI推送到Harbor

当前最佳实践是使用 **OCI 协议**将 Helm Chart 推送到 Harbor，使 Chart 像容器镜像一样被统一管理。

## 环境要求

| 工具 | 最低版本 |
|:---|:---:|
| Helm | >= 3.8.0 |
| Harbor | >= 2.6.0（推荐 2.8.0+ LTS） |

## 推送步骤

### 1. 登录
```bash
helm registry login <your-harbor-domain>
# 自签名证书加 --insecure（仅对登录生效）
```

> `--insecure` 仅对登录操作生效，**不会持久化到后续 `helm push` 命令**。

### 2. 打包
```bash
helm dependency update <chart-dir>   # 如有外部依赖
helm package <chart-dir>             # 生成 .tgz 文件
```

### 3. 推送
```bash
helm push <chart>.tgz oci://<harbor-domain>/<project>
```

## 自签名证书处理

`helm push` **不支持** `--insecure` 参数，需让系统信任 Harbor 的 CA 证书：

**Windows**（管理员 PowerShell）：
```powershell
certutil -addstore Root harbor-ca.crt
```

**Linux**：
```bash
sudo cp harbor-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

## 最佳实践

1. **版本号管理**：严格遵循语义化版本 `MAJOR.MINOR.PATCH`，**不加 `v` 前缀**。
2. **CI/CD 自动化**：将流程集成到流水线中（GitHub Actions / GitLab CI / Jenkins）。
3. **机器人账户**：为 CI/CD 创建专用 Robot Account，避免使用个人账户。
4. **内容信任**：如 Harbor 项目启用 Content Trust，Chart 需经 Cosign/Notation 签名。
5. **弃用 ChartMuseum**：Harbor v2.6.0+ 直接使用 OCI 方式，传统 ChartMuseum 已弃用。

## 相关笔记

- [[OCI-协议规范]]
- [[Helm-常用命令]]
- [[TLS-传输层安全协议]]
