# neca

**Neko's Efficient Certificate Administrator（neca）**：面向 strongSwan IKEv2 的轻量证书管理工具。CLI + 可选 Web 面板 + 受控 root helper。

- [中文文档](https://neca.nekobiglazyer.work/zh/)
- [English docs](https://neca.nekobiglazyer.work/en/)
- [项目仓库](https://github.com/hyhdsl/neca)
- [v1.1.0 Release](https://github.com/hyhdsl/neca/releases/tag/v1.1.0)

默认只读；Web 写操作通过 root helper 的命令白名单执行。生产环境优先使用 OpenSSH
本地转发，不直接暴露 Web 端口。

## 适用场景

- 单机或少量 strongSwan IKEv2 网关，管理员希望用 CLI 维护证书。
- 需要默认只读 Web、清晰权限边界、审计与 Docker 部署。
- 不需要多租户、数据库或复杂高可用集群的部署。
- **不适合**：大规模 PKI/多租户、无人值守自动签发，或未经访问设计直接暴露公网。

许可证：AGPL-3.0-only。详见仓库根目录 LICENSE 与 AI-DISCLOSURE.md。
