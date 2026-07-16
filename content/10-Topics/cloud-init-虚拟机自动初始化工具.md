---
title: cloud-init — 虚拟机自动初始化工具
date: 2026-06-24
tags:
  - topic/Linux
  - topic/云原生
aliases:
  - cloud-init 模块
  - cloud-config
  - cloud-init 插件
status: to-review
---

初次接触 cloud-init，把它理解成云服务器或虚拟机的"**首次启动配置管家**"会比较直观。

cloud-init 本身是一个开源工具，而"cloud-init 插件"在它的语境里，官方名称是 **"模块 (Modules)"**。你可以把每个模块都看作一个**功能单一的"插件"**，专门负责一项具体的配置任务。

### 核心概念：模块即插件

cloud-init 由许多这样的"插件"（模块）组成。每个模块都是一个专门处理单个配置任务的 Python 脚本。比如，`set_hostname` 模块只负责设置主机名，`users_groups` 模块只负责创建用户和组。

它的最大优势在于**幂等性 (Idempotency)**。这意味着，无论一个模块执行一次还是多次，结果都是一样的。例如，`users_groups` 模块在实例每次启动时都会运行，如果用户已存在，它就不会重复创建，而是保持现状。这让配置过程非常可靠且可重复。

### 工作机制

cloud-init 的工作流程清晰且自动化：

1.  **启动触发**：在 Linux 实例**首次启动**时，cloud-init 服务会自动运行。
2.  **获取配置**：它会从云平台的**元数据服务**读取配置数据，主要是用户通过"用户数据 (User-Data)"传入的配置。
3.  **按序执行**：cloud-init 会按照预定义的**阶段 (Stages)** 和顺序来执行各个模块。例如，它通常会先创建用户和组 (`users_groups`)，然后安装软件包 (`packages`)，最后才运行用户自定义脚本 (`runcmd`)。
4.  **完成配置**：所有模块执行完毕后，一个配置好的、可投入生产的服务器就准备好了。

### 丰富的"插件"功能

cloud-init 的模块覆盖了系统初始化的大部分需求。以下是一些常见模块及其功能：

*   **`users_groups`**：配置用户和组。
*   **`package_update` / `package_install`**：更新并安装软件包。
*   **`write_files`**：向指定路径写入配置文件或脚本。
*   **`set_hostname` / `update_hostname`**：设置和更新主机名。
*   **`runcmd`**：执行任意 shell 命令。
*   **`ssh`**：配置 SSH 密钥等。
*   **`mounts`**：配置挂载点。
*   **`growpart` / `resizefs`**：自动扩展分区和文件系统以利用全部磁盘空间。

### 主要应用场景

cloud-init 的应用场景非常广泛，主要围绕云环境和虚拟化：

*   **云服务器（IaaS）初始化**：这是最典型的场景。在 AWS、Azure、阿里云等平台上创建 Linux 虚拟机时，通过传入 cloud-init 脚本，可以实现自动化配置。
*   **批量与自动化部署**：当你需要部署大量相同配置的服务器时，使用相同的 cloud-init 配置文件可以保证环境完全一致，避免手动操作的错误。
*   **本地虚拟化环境**：不仅限于公有云，在 KVM、Incus/LXD 等本地虚拟化环境中同样可以使用 cloud-init 来完成客户机系统的初始化。
*   **CI/CD 与测试环境**：在持续集成/持续部署（CI/CD）流程中，可以随时通过 cloud-init 拉起一个干净的测试环境，用完即毁，非常灵活。
*   **无状态与不可变基础设施**：在这种架构中，服务器从不被修改，而是被替换。cloud-init 确保了每个新实例在启动时都能自动完成所有必要的配置，成为理想的选择。

### 如何使用

使用 cloud-init 非常直接。你只需要创建一个以 `#cloud-config` 开头的 YAML 格式配置文件，并在其中声明你想要的系统最终状态。

例如，下面这个配置会在虚拟机首次启动时，更新软件包，安装 Nginx 和 Node.js，并写入一个简单的配置文件：

```yaml
#cloud-config
package_upgrade: true
packages:
  - nginx
  - nodejs
  - npm
write_files:
  - owner: www-data:www-data
    path: /etc/nginx/sites-available/default
    content: |
      server {
        listen 80;
        location / {
          proxy_pass http://localhost:3000;
        }
      }
runcmd:
  - service nginx restart
```

在创建虚拟机时，将这个文件内容作为"用户数据 (User-Data)"传入即可。

### 总结

可以把 cloud-init 想象成一个"**乐高积木套装**"，而每个"模块"（插件）就是一块功能单一的积木。你只需要通过一个简单的 YAML 文件，像看图纸一样，把需要的积木（模块）按顺序组合起来，就能快速、一致地搭建出你想要的任何"建筑"（服务器状态）。
