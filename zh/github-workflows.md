# GitHub 工作流说明

本项目在 GitHub 上只有三个工作流，分别负责质量门禁、Release 资产和容器镜像。它们都由仓库内的 YAML 定义，不在本地手工模拟发布。

## 1. `ci.yml`：质量门禁

- 触发：推送到 `main`、`feat/**`、`fix/**`、`chore/**`，以及所有 Pull Request。
- 权限：只读仓库内容（`contents: read`），不使用发布密钥。
- 作业：`build-and-test`，以及依赖它的 `docker-smoke`。
- 检查内容：`gofmt`、`go vet`、`govulncheck`、平台脚本模板 ASCII、语言包 key 一致性、文档链接/围栏/帮助站结构、`go test -race ./...`、Linux amd64/arm64 交叉编译、影子对比与 `version` 冒烟测试；随后构建 lite/web 镜像并验证入口。
- 失败时不要合并；`main` 的分支保护要求 `build-and-test` 通过。

## 2. `release.yml`：GitHub Release

- 触发：推送任意 `v*` 标签。
- 预检：标签格式、tag 指向的 commit 是否位于 `main` 或 `release/x.y`，以及该 commit 的 `build-and-test` 是否成功。
- 发布 job：使用受保护的 `release-signing` environment；只有 `v*` 标签可进入。
- 流程：检出 → 构建 Linux amd64/arm64 的 `neca`、`necad`、`neca-web-helper` → 生成 `checksums.txt` → 在临时 GNUPGHOME 中导入发布私钥 → 签名并自验 → 导出公钥 → 创建 Release。
- 资产：6 个二进制 + `checksums.txt` + `checksums.txt.asc` + `neca-gpg-key.asc`，共 9 个。
- 预发布规则：标签名包含 `-`（例如 `v1.6.0-preview.1`）时自动标记为 Pre-release；稳定标签不标记。
- 注意：发布内容来自标签指向的提交；不要在标签创建后改代码再期待 Release 自动更新。验证方式见 [GPG 验证](gpg.md)。

## 3. `docker.yml`：GHCR 多架构镜像

- 触发：推送任意 `v*` 标签，或手动 `workflow_dispatch`。
- 权限：`contents: read`、`packages: write`。
- 流程：构建二进制 → QEMU/Buildx → 登录 GHCR → 构建并推送两个镜像：
  - `ghcr.io/hyhdsl/neca`（lite/CLI）；
  - `ghcr.io/hyhdsl/neca-web`（Web/helper）。
- 平台：`linux/amd64`、`linux/arm64`。
- 标签：镜像使用 Git 标签；稳定标签同时更新 `latest`，包含 `-` 的 preview 标签不会更新 `latest`。

## 发布顺序

1. 变更通过 PR 合入 `main`，并等待 `ci.yml` 的 `build-and-test` 成功。
2. 创建 annotated 标签并推送，例如 `v1.6.0`。
3. 观察 `release.yml` 与 `docker.yml` 均成功。
4. 核对 Release 的 9 个资产、checksums、签名与 GHCR 镜像标签。
5. 部署与回滚由另行批准的运维流程执行；本文不记录任何具体服务器信息。

## 常见问题

- **只有 Release 成功、Docker 失败**：检查 GHCR 包权限或 registry 登录；Release 资产不依赖 Docker。
- **preview 更新了 latest**：不要手工覆盖；检查标签是否包含 `-` 以及 metadata 的启用条件。
- **CI 影子对比失败**：先看是否是新旧输出文本差异；如是预期变更，更新对比归一化规则，不要直接跳过门禁。
