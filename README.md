# neca 帮助中心

> 适用于 **neca v0.3.0**（2026-11-15）。neca 是 strongSwan IKEv2 用户证书管理工具，
> 不是 VPN 服务端本身；它负责签发、吊销、重签证书，生成客户端包，并维护 CRL。

## 先看这里

- 第一次使用：从 [快速开始](getting-started.md) 走一遍最小流程。
- 已有服务器：看 [安装、升级与回滚](install.md) 和 [部署与日常运维](operations.md)。
- 管理证书：看 [证书管理](certificates.md)。
- 使用 Web 面板：看 [Web 面板](web.md)。
- 需要自动化：看 [Web API（实验性）](api.md)。
- 出问题：看 [故障排查](troubleshooting.md) 和 [常见问题](faq.md)。
- English core pages: <https://neca.nekobiglazyer.work/en/>

## 运行形态

| 形态 | 内容 | 适用场景 |
|---|---|---|
| lite（默认） | `neca` CLI；不含 Web 服务端 | 只用命令行管理证书 |
| neca-web | `necad` Web 服务端；默认只读 | 浏览器查看证书、下载与验证 |
| neca-web + helper | 额外 root helper；可显式开放写能力 | 需要在 Web 中签发/吊销/重签 |

lite 与 neca-web 使用同一版本标签；Web 默认只监听回环地址，不直接暴露公网。

## 重要安全提醒

1. `ca.key` 是最高敏感资产，必须保持 `0600 root:root`；不要交给 Web 容器。
2. Web 默认 `readOnly=true`。开启写能力前，先确认 helper 健康并理解 root 边界。
3. 生产写操作先加 `--dry-run`，并保留 PKI 备份。
4. 不要把真实服务器地址、组织名、CA 名称或任何私钥提交到仓库。
5. 本项目以 **AGPL-3.0-only** 发布；二次开发与网络服务须履行源码公开义务。

## 版本与来源

- 项目：<https://github.com/hyhdsl/neca>
- 正式版：<https://github.com/hyhdsl/neca/releases/tag/v0.3.0>
- 帮助文档源码：<https://github.com/hyhdsl/neca/tree/main/docs/help>
- 许可：<https://github.com/hyhdsl/neca/blob/main/LICENSE>

> AI 披露：本项目代码、测试与文档由 AI 编程助手辅助生成；请自行审查并在生产前验证。
