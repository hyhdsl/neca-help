# 常见问题

## neca 是 VPN 服务端吗？

不是。neca 管理 strongSwan IKEv2 的用户证书与客户端包，不包含 VPN 服务端本身。
strongSwan 的部署、防火墙和 IKEv2 配置不在本项目范围内。

## lite 和 neca-web 必须一起装吗？

不必。lite 是默认形态；neca-web 是可选面板包。需要 Web 时再拉取同版本镜像；
不启用 Web 不影响 CLI。

## Web 默认能写吗？

不能。`readOnly=true` 是默认值。需要 helper 健康并显式执行 `neca web readonly off`，
再发送 SIGHUP（`docker kill --signal=HUP neca-web`），才开放 issue/revoke/unrevoke/reissue。

## Web 能删除证书吗？

不能。删除只在 CLI 中提供，且需要 `--yes`；已吊销证书还需要 `--purge-revoked`。
这是有意的风险控制。

## 为什么 Web 端口是 18080？

代码默认监听 `127.0.0.1:18080`；以 `neca.json` 的 `web.listen` 为准。若端口被占用，
可改为其他回环端口，并重启 `neca-web`。

## Web 可以直接暴露公网吗？

不建议。默认只监听回环地址，优先 OpenSSH 本地转发。确需公网访问时，使用系统
原生 IKEv2 + 反向代理 TLS，并考虑 mTLS；不要把应用直接绑到 `0.0.0.0`。

## 支持多用户/RBAC 吗？

当前是单管理员模型，不支持多用户、角色或按证书归属授权。多用户是后续阶段。

## 支持 Passkey 吗？

当前只保留接口与 `passkey` 配置开关，未注册登录路由。启用需要 HTTPS、稳定域名
和 RP ID，需单独设计与评审。

## 为什么不用 SQLite？

M6 的只读查询和签名 Cookie 会话不需要数据库；索引仍是 `index.txt`。SQLite 留到
有明确并发/多用户需求时再评估。

## 数据在哪里？

主要在 `/opt/ipsec`：

- `pki/`：CA、索引、meta、审计；
- `clients/`：客户端证书包；
- `swanctl/x509crl/`：CRL；
- `downloads/`：临时 zip；
- `web-secrets/`：Web 会话密钥与密码哈希。

## 升级会丢证书吗？

不会，前提是保留 `/opt/ipsec` 数据并正确挂载。升级前仍应备份。不要执行删除
数据目录的命令，也不要在 lite/neca-web 之间混用不兼容版本。

## 如何备份？

~~~bash
tar -C /opt/ipsec -czf /root/ipsec-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  pki clients swanctl neca.json web-secrets
~~~

备份含 CA 私钥，必须加密并限制访问。

## 旧 bash 脚本还能用吗？

保留在 `legacy/`，用于回退、影子测试和历史参考。新部署应使用 neca；旧脚本的
使用说明见 [legacy-vpn-cert-manager.md](https://github.com/hyhdsl/neca/blob/main/docs/development/legacy-vpn-cert-manager.md)。

## 为什么外部 PR 暂不接受？

项目处于开发期，仅维护者内部合并；问题可通过 Issue 反馈，修改可先 fork 并遵守
AGPL。功能稳定后会重新评估开放 PR。

## 许可证是什么？

**AGPL-3.0-only**。允许商业使用，但分发、修改或通过网络提供服务时须履行源码
公开与许可保留义务。详见 [LICENSE](https://github.com/hyhdsl/neca/blob/main/LICENSE)。

## 文档和代码是 AI 生成的吗？

代码、测试与文档有 AI 编程助手参与；可能存在错误。使用前请审查、测试并自行
承担风险。项目在 README 与 NOTICE 中有完整披露。
