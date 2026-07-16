---
tags:
  - topic/GitLab
  - topic/CI-CD
status: to-review
---

# GitLab CI include 变量限制

GitLab CI 的 `include` 关键字中，变量的使用存在严格限制。

## 核心限制

在 `include` 部分，**只有以下预定义变量可用**：

- `$CI_SERVER_PROTOCOL`
- `$CI_SERVER_HOST`
- `$CI_SERVER_PORT`
- `$CI_SERVER_URL`
- `$CI_PROJECT_ID`
- `$CI_PROJECT_PATH`
- `$CI_PROJECT_PATH_SLUG`
- `$CI_PROJECT_NAMESPACE`
- `$CI_PROJECT_NAMESPACE_ID`
- `$CI_PROJECT_NAME`
- `$CI_PROJECT_TITLE`
- `$CI_PROJECT_VISIBILITY`
- `$CI_PROJECT_DESCRIPTION`
- `$CI_PROJECT_ROOT_NAMESPACE`
- `$CI_PROJECT_REPOSITORY_LANGUAGES`
- `$CI_PROJECT_URL`
- `$CI_PROJECT_CLASSIFICATION_LABEL`
- `$CI_DEFAULT_BRANCH`
- `$CI_COMMIT_SHA`
- `$CI_COMMIT_SHORT_SHA`
- `$CI_COMMIT_REF_NAME`
- `$CI_COMMIT_REF_SLUG`
- `$CI_ENVIRONMENT_NAME`
- `$CI_JOB_ID`
- `$CI_JOB_TOKEN`
- `$CI_REGISTRY`
- `$CI_REGISTRY_IMAGE`
- `$CI_REGISTRY_USER`
- `$CI_REGISTRY_PASSWORD`
- `$CI_REPOSITORY_URL`
- `$CI_RUNNER_ID`
- `$CI_RUNNER_DESCRIPTION`
- `$CI_RUNNER_TAGS`
- `$CI_TEMPLATE_REGISTRY_HOST`
- `$CI_TEMPLATE_REGISTRY_USER`
- `$CI_TEMPLATE_REGISTRY_PASSWORD`

用户自定义变量（如 `$MY_VAR`）、`CI/CD Settings` 中设置的变量、`variables:` 关键字定义的变量 **在 `include` 处理阶段尚未解析**，因此无法使用。

## 示例

❌ **错误用法**：自定义变量用于 include 路径
```yaml
variables:
  COMPONENT_URL: "https://gitlab.com/my-group/my-component"

include:
  - component: $COMPONENT_URL
```

✅ **正确用法**：结合预定义变量
```yaml
include:
  - project: '$CI_PROJECT_PATH'
    file: '/templates/common.yml'
```

## 解决方案

在流水线逻辑内部（`before_script` / `script`）使用自定义变量，而非在 `include` 阶段。

> 详细参考：[GitLab Docs — Use variables with include](https://docs.gitlab.com/ci/yaml/includes/#use-variables-with-include)
