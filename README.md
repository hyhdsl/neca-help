# neca

面向 strongSwan IKEv2 的轻量证书管理工具。CLI + 可选 Web 面板 + 受控 root helper。

- [中文文档](https://neca.nekobiglazyer.work/zh/)
- [English docs](https://neca.nekobiglazyer.work/en/)
- [项目仓库](https://github.com/hyhdsl/neca)
- [v0.4.0 Release](https://github.com/hyhdsl/neca/releases/tag/v0.4.0)

默认只读；Web 写操作通过 root helper 的命令白名单执行。生产环境优先使用 OpenSSH
本地转发，不直接暴露 Web 端口。

许可证：AGPL-3.0-only。详见仓库根目录 LICENSE 与 AI-DISCLOSURE.md。
