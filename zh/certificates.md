# 证书管理

## 数据模型

neca 不把证书状态放进数据库，仍使用 OpenSSL 兼容文件：

| 路径 | 作用 |
|---|---|
| `/opt/ipsec/pki/ca/index.txt` | 证书索引；状态、到期、序列号的唯一事实源 |
| `/opt/ipsec/pki/meta/<CN>.env` | 平台、签发时间、到期时间、序列号等元数据 |
| `/opt/ipsec/clients/<CN>/` | 客户端证书、p12、平台包 |
| `/opt/ipsec/swanctl/x509crl/ca.crl.pem` | 供 strongSwan 加载的 CRL |
| `/opt/ipsec/pki/audit.log` | 追加式审计日志 |

不要手工编辑 `index.txt` 后不回读校验；需要补录时优先使用 `neca pki index sync`。

## CN 规则

CN 只允许：

~~~text
A-Z a-z 0-9 . _ -
~~~

CN 同时是 VPN 用户标识、目录名和证书 Common Name。不要使用空格、斜杠、中文或
`..`。示例：`alice`、`team.laptop-01`。

## 常用命令

| 命令 | 作用 |
|---|---|
| `neca cert list` | 列出证书 |
| `neca cert verify [CN]` | 检查 p12 算法、证书数量、密码 |
| `neca cert issue <CN> ...` | 签发证书与平台包 |
| `neca cert reissue <CN> ...` | 归档旧证书并重签 |
| `neca cert revoke <CN>` | 吊销并更新 CRL |
| `neca cert unrevoke <CN>` | 恢复吊销并更新 CRL |
| `neca cert trash <CN>` | 移入回收站（可恢复） |
| `neca trash list` / `restore <ID>` | 查看/恢复回收站条目 |
| `neca trash purge <ID>` | 后台物理清理（Y/N，默认 N） |
| `neca cert platform add <CN> ...` | 补平台包 |
| `neca cert platform set <CN> ...` | 修改平台标注 |
| `neca cert package <CN> ...` | 从现有证书重建平台包 |
| `neca cert export issued` / `neca cert export revoked` | 导出 CSV |
| `neca pki index sync` | 为手工放入的证书补索引与元数据 |

所有写命令支持 `--dry-run`；建议先执行：

~~~bash
neca cert issue alice --platforms windows,android,ios --dry-run
~~~

## 签发

~~~bash
neca cert issue alice \
  --platforms windows,linux,macos,android,ios \
  --days 365 \
  --installers windows \
  --p12-profile compat \
  --zip
~~~

参数：

- `--platforms`：逗号分隔；默认由配置决定。
- `--days`：有效期；默认 365。
- `--installers`：`none`、`all` 或平台列表；默认 `none`。
- `--p12-profile`：`compat`（默认，3DES/SHA1）或 `modern`（AES-256/SHA256）。
- `--zip`：签发后在 `downloads/` 生成带 TTL 的压缩包。

证书文件写入 `clients/alice/`；私钥只留在服务器，不进入压缩包。

## 查看与验证

~~~bash
neca cert list
neca cert list --json
neca cert verify alice
neca cert verify
~~~

`verify` 会检查 p12 是否只含客户端证书、是否使用预期的算法、密码长度是否合规。
它不替代客户端实机导入测试。

## 重签

~~~bash
neca cert reissue alice \
  --platforms windows,android,ios \
  --days 365 \
  --zip
~~~

重签会归档旧证书并生成新序列号；客户端必须重新导入新包。重签前建议备份
`clients/alice/` 与 `pki/meta/alice.env`。

## 吊销与恢复

~~~bash
neca cert revoke alice
neca cert unrevoke alice
~~~

两者都会重建 CRL。默认重载模式是 `creds`，通过 `swanctl --load-creds` 热加载，
不重启 strongSwan；若配置为 `restart`，则按旧行为重启容器。生产上先确认：

~~~bash
neca config show
~~~

## 回收站

`cert trash` 只把证书和相关文件移入回收站，不物理删除；Web 面板也只移入回收站。

~~~bash
neca cert trash alice
neca trash list
neca trash restore <ID>
~~~

回收站默认不限时保存；恢复已吊销证书不会自动取消吊销。物理清理：

~~~bash
neca trash purge <ID>
~~~

`trash purge` 需要一次 Y/N（默认 N）；后台自动化可用 `--yes`。保留期由 `trash.retention` 配置（默认 0=不限时），可用 `neca trash purge --expired` 清理过期条目。不再输入 `PURGE <CN>`，也不承诺 SSD/快照/备份上的安全擦除。清理前先备份：

~~~bash
tar -C /opt/ipsec -czf /root/ipsec-before-trash-$(date +%Y%m%d-%H%M%S).tar.gz \
  pki clients
~~~

## 平台包

补平台（不重签）：

~~~bash
neca cert platform add alice --platforms linux,macos
~~~

修改平台标注：

~~~bash
neca cert platform set alice --platforms windows,android
~~~

从现有证书重建全部包：

~~~bash
neca cert package alice --platforms windows,linux,macos,android,ios --zip
~~~

## 导出与索引同步

~~~bash
neca cert export issued --out /root/issued.csv
neca cert export revoked --out /root/revoked.csv
neca pki index sync
~~~

CSV 使用 UTF-8 BOM，便于 Excel 直接打开。`index sync` 只补索引与元数据，
不重签证书。

## 审计与 JSON

~~~bash
neca cert list --json
neca cert revoke alice --json
~~~

审计日志默认 `/opt/ipsec/pki/audit.log`，每行包含时间、操作者、动作、CN、
序列号与结果。JSON 输出适合脚本处理；不要把含密码的字段写进工单或日志。
