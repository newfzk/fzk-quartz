---
tags:
  - idea
  - maven
  - troubleshooting
  - multi-module
status: to-review
---

# IDEA-Maven多模块依赖识别排查

多模块 Maven 项目中 IDEA 报"程序包不存在"的排查步骤（按推荐顺序）：

1. **Maven 命令行验证** — 执行 `mvn clean compile -pl business-sys -am`，区分是 Maven 问题还是 IDEA 问题
2. **检查模块是否在同一个 IDEA Project** — `File → Project Structure → Modules`，确认被依赖模块已导入
3. **Maven 重新导入** — `View → Tool Windows → Maven → Reload All Projects`
4. **Invalidate Caches** — `File → Invalidate Caches... → Invalidate and Restart`
5. **先安装依赖模块** — 在依赖模块目录执行 `mvn clean install -DskipTests`
6. **检查 packaging 类型** — 被依赖模块的 `<packaging>` 必须为 `jar`（非 `pom`）
7. **检查依赖 scope** — `<scope>` 不应为 `test` 或 `provided`
8. **清理 .idea 和 .iml** — 关闭 IDEA 后删除 `.idea` 目录和所有 `.iml` 文件，重新打开项目（最后手段）

> 核心原则：先通过 Maven 命令行确认依赖本身是否可用，再定位 IDEA 层面的缓存/配置问题。
