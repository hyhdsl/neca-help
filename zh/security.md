# 安全、权限与备份

## 资产分级

| 资产 | 敏感级别 | 要求 |
|---|---|---|
| `pki/ca.key` | 最高 | `0600 root:root`；不进入 Web 容器、不备份到公开位置 |
| 客户端私钥 / p12 | 高 | 仅授权用户可取；下载包有 TTL |
| `web-secrets/session.key` | 高 | 可伪造会话，按密钥管理 |
| `web-secrets/password.hash` | 高 | 只存 bcrypt 哈希，不存明文 |
| `pki/ca/index.txt`、CRL | 中 | 完整性重要；可公开 CRL |
| 审计日志 | 高 | 只追加，限制读取，保留足够时间 |

## 目录与权限

启用 Web 时的典型权限：

~~~text
/opt/ipsec/config/neca.json       root:neca-web 0640
/opt/ipsec/web-secrets/           root:neca-web 0710
  session.key                     root:neca-web 0640
  password.hash                   root:neca-web 0640
  web-audit.log                   root:neca-web 0660
/run/neca/web-helper.sock         root:neca-web 0660
~~~

Web 进程固定 UID/GID 10001，非 root。CLI 以 root 执行 CA 操作。lite 容器挂载
`docker.sock` 等价于宿主 root，必须限制谁能执行 `neca`。

## Web 的边界

- Web 容器只读挂载必要文件；不整体挂载 `/opt/ipsec/pki`。
- 不挂载 `ca.key`、客户端私钥、`docker.sock`。
- 写操作交给 root helper；helper 只接受白名单操作。
- Web 侧 `readOnly` 与 helper 侧 `readOnly` 双重检查。
- UI 隐藏按钮不是安全边界；服务端必须拒绝。
- 默认只监听 `127.0.0.1`，不发布公网端口。

## 网络访问

推荐顺序：

1. 系统原生 OpenSSH 本地转发；
2. 系统原生 IKEv2 连接 VPN 后访问；
3. 反向代理 TLS + 可选 mTLS。

不要引入 Tailscale/WireGuard 等第三方 VPN 客户端作为本项目的前提。若使用反向
代理，必须限制来源、校验代理共享密钥与客户端 CN，并防止伪造头。

## 会话与密钥轮换

- 空闲超时默认 30m；绝对上限 24h。
- 修改 `readOnly` / `idleTimeout` 后发送 SIGHUP（`docker kill --signal=HUP neca-web`）；`web.listen`/`adminCNs`/passkey 仍需重启。
- 轮换 `session.key` 会使现有会话失效；在维护窗口执行。
- 管理员密码通过 `neca web hash-password` 生成哈希，不要写进 `neca.env`。
- 若部署流程生成了初始密码文件，它只应短期存在；首次登录后删除或转移到密码管理器。

## 回收站与安全删除（规划）

- 计划中的 Web 删除只把已吊销证书**移入回收站**，物理清理只能由后台确认。
- 回收站默认不限时保存；保留期可由用户配置；恢复已吊销证书不会自动取消吊销状态。
- 后台 purge 需要二次确认。
- **不承诺安全擦除**：SSD、文件系统快照、备份和对象存储副本可能保留数据；回收站只用于隔离与防误删。

## 高风险设置（规划）

- 高风险安全设置默认只允许后台修改；Web 设置页只读展示。
- 即使后台开启 Web 修改开关，前后台修改高风险设置也需要 Y/N 二次确认、重新认证、CSRF 与审计。
- 建议 `adminCNs`、passkey 与 helper 相关设置永远保持后台-only。

## 备份安全

备份包含 CA 私钥，必须：

- 加密存储；
- 限制 root 读取；
- 与生产主机分离保存；
- 定期验证可恢复；
- 不进入公开仓库、公开对象存储或 CI artifact。

## 审计与监控

- 关注 `audit.log` 中的 issue/revoke/reissue/delete 与失败记录。
- 关注 Web 登录失败、CSRF 拒绝、helper 不可用和只读切换。
- 记录部署前后 `index.txt` / CRL 哈希，便于解释变化。
- 不记录密码、Cookie、私钥内容。

## 供应链与许可

- 固定镜像版本；生产不要用 `latest`。
- 只从官方仓库/GHCR 拉取；核对 checksums。
- 项目以 **AGPL-3.0-only** 发布；分发或网络服务须履行源码公开义务。
- 代码与文档含 AI 辅助生成内容；使用前自行审查。

## 上线前检查清单

- [ ] `ca.key` 仍为 `0600 root:root`
- [ ] Web 未挂载 `ca.key` / 客户端私钥 / `docker.sock`
- [ ] Web 未发布公网端口
- [ ] `readOnly=true` 或已明确开启写能力
- [ ] helper socket 仅 `root:neca-web` 可访问
- [ ] 初始密码文件已处理
- [ ] 备份与恢复演练完成
- [ ] 审计日志与监控已接入
- [ ] 版本标签固定，回滚镜像可用
