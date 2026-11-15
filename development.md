# 开发、许可与反馈

## 源码与文档

- 主仓库：<https://github.com/hyhdsl/neca>
- 开发文档：<https://github.com/hyhdsl/neca/blob/main/docs/neca-development.md>
- Web 设计：<https://github.com/hyhdsl/neca/blob/main/docs/web-panel.md>
- Web 实施：<https://github.com/hyhdsl/neca/blob/main/docs/web-panel-development.md>
- 部署文档：<https://github.com/hyhdsl/neca/blob/main/docs/deploy.md>
- 帮助文档源码：<https://github.com/hyhdsl/neca/tree/main/docs/help>

## 本地构建

开发在 Docker 中完成：

~~~bash
docker build -f Dockerfile.dev -t neca-dev .
docker compose run -T --rm dev make all
docker compose run -T --rm dev make shadow
~~~

常用命令：

| 命令 | 作用 |
|---|---|
| `make all` | gofmt + vet + ASCII + 测试 + 交叉编译 |
| `make shadow` | 旧 bash 与 neca 的影子对比 |
| `make check-locales` | zh-CN/en-US 语言包 key 一致性 |
| `make docker` | 构建 lite 与 neca-web 镜像 |

## 架构

- `internal/core`：业务核心，不依赖展示层；
- `internal/cli`：参数解析、子命令与输出；
- `internal/web`：Web 服务端、会话、模板与 API；
- `internal/helper`：root helper 协议与白名单；
- `cmd/neca`、`cmd/necad`、`cmd/neca-web-helper`：三个入口；
- `internal/core/ports`：可替换接口（索引、配置来源）。

lite 与 neca-web 共享同一 core 与数据目录；不发布 full 镜像。

## 发布流程

1. 合入 `main` 并通过 CI；
2. 打 tag `vX.Y.Z`；
3. `release.yml` 生成二进制与 checksums；
4. `docker.yml` 推送 GHCR；
5. 在服务器固定版本标签升级，保留回滚镜像。

预览标签（`vX.Y.Z-preview.N`）不是稳定版；旧预览只保留源代码归档。

## 帮助站点

本帮助站点的源文件在 `docs/help/`，使用 GitBook 兼容的
`README.md` + `SUMMARY.md` 结构，并由 Docsify 在 GitHub Pages 上静态渲染。

为什么不是 GitBook SaaS：GitBook 的自定义域名属于付费功能；本项目需要免费、
可版本化、可自动部署的方案。保留 GitBook 结构是为了将来可以无缝导入。

站点发布仓库：
<https://github.com/hyhdsl/deepseek-neca-help.github.io>

自定义域名：<https://neca.nekobiglazyer.work>

## 贡献与反馈

- 开发期暂不接受外部 Pull Request；
- 问题请提交 <https://github.com/hyhdsl/neca/issues>；
- 修改请先 fork，并遵守 AGPL 与 [二次开发要求](https://github.com/hyhdsl/neca/blob/main/docs/SECONDARY-DEVELOPMENT.md)；
- 安全问题时不要公开粘贴私钥、密码或可利用细节，先联系维护者。

## 许可

**AGPL-3.0-only**。完整文本：

<https://github.com/hyhdsl/neca/blob/main/LICENSE>

本项目还包含 AI 披露与免责声明，见 `NOTICE` 与 README。
