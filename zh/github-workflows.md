# GitHub 工作流说明

本项目在 GitHub 上只有三个工作流，分别负责质量门禁、Release 资产和容器镜像。它们都由仓库内的 YAML 定义，不在本地手工模拟发布。

## 1. `ci.yml`：质量门禁

- 触发：推送到 `main`、`feat/**`、`fix/**`、`chore/**`，以及所有 Pull Request。
- 权限：只读仓库内容（`contents: read`），不使用发布密钥。
- 作业：`build-and-test`。
- 检查内容：
  - `gofmt` 无差异；
  - `go vet`；
  - `govulncheck` 依赖漏洞扫描；
  - 平台脚本模板 ASCII 校验；
  - 中英文语言包 key 一致性；
  - Markdown 链接、代码围栏与帮助站结构；
  - `go test -race ./...`；
  - Linux amd64/arm64 交叉编译；
  - 新旧实现影子对比；
  - `version` 冒烟测试。
- 失败时不要合并；`main` 的分支保护要求 `build-and-test` 通过。

## 2. `release.yml`：GitHub Release

- 触发：推送任意 `v*` 标签。
- 权限：`contents: write`，只用于创建 Release 和上传资产。
- 流程：检出标签对应提交 → 构建 Linux amd64/arm64 的 `neca`、`necad`、`neca-web-helper` → 生成 `checksums.txt` → 创建 Release。
- 资产：6 个二进制 + `checksums.txt`，共 7 个。
- 预发布规则：标签名包含 `-`（例如 `v1.3.0-preview.1`）时自动标记为 Pre-release；稳定标签不标记。
- 注意：发布内容来自标签指向的提交；不要在标签创建后改代码再期待 Release 自动更新。

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
2. 创建 annotated 标签并推送，例如 `v1.2.0` 或 `v1.3.0-preview.2`。
3. 观察 `release.yml` 与 `docker.yml` 均成功。
4. 核对 Release 的 7 个资产、checksums 和 GHCR 镜像标签。
5. 部署与回滚由另行批准的运维流程执行；本文不记录任何具体服务器信息。

## 常见问题

- **只有 Release 成功、Docker 失败**：检查 GHCR 包权限或 registry 登录；Release 资产不依赖 Docker。
- **preview 更新了 latest**：不要手工覆盖；检查标签是否包含 `-` 以及 metadata 的启用条件。
- **CI 影子对比失败**：先看是否是新旧输出文本差异；如是预期变更，更新对比归一化规则，不要直接跳过门禁。