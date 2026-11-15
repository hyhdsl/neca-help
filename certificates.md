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

不要手工编辑 `index.txt` 后不回读校验；需要补录时优先使用 `neca index sync`。

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
| `neca list` | 列出证书 |
| `neca verify [CN]` | 检查 p12 算法、证书数量、密码 |
| `neca issue <CN> ...` | 签发证书与平台包 |
| `neca reissue <CN> ...` | 归档旧证书并重签 |
| `neca revoke <CN>` | 吊销并更新 CRL |
| `neca unrevoke <CN>` | 恢复吊销并更新 CRL |
| `neca delete <CN> --yes` | 删除证书与客户端包 |
| `neca platform add <CN> ...` | 补平台包 |
| `neca platform set <CN> ...` | 修改平台标注 |
| `neca package <CN> ...` | 从现有证书重建平台包 |
| `neca export issued` / `neca export revoked` | 导出 CSV |
| `neca index sync` | 为手工放入的证书补索引与元数据 |

所有写命令支持 `--dry-run`；建议先执行：

~~~bash
neca issue alice --platforms windows,android,ios --dry-run
~~~

## 签发

~~~bash
neca issue alice \
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
neca list
neca list --json
neca verify alice
neca verify
~~~

`verify` 会检查 p12 是否只含客户端证书、是否使用预期的算法、密码长度是否合规。
它不替代客户端实机导入测试。

## 重签

~~~bash
neca reissue alice \
  --platforms windows,android,ios \
  --days 365 \
  --zip
~~~

重签会归档旧证书并生成新序列号；客户端必须重新导入新包。重签前建议备份
`clients/alice/` 与 `pki/meta/alice.env`。

## 吊销与恢复

~~~bash
neca revoke alice
neca unrevoke alice
~~~

两者都会重建 CRL。默认重载模式是 `creds`，通过 `swanctl --load-creds` 热加载，
不重启 strongSwan；若配置为 `restart`，则按旧行为重启容器。生产上先确认：

~~~bash
neca config show
~~~

## 删除

删除是不可逆操作：

~~~bash
neca delete alice --yes
~~~

已吊销证书默认不能直接删除；如确认要清理，使用：

~~~bash
neca delete alice --yes --purge-revoked
~~~

Web 面板**没有**删除入口。删除前先备份：

~~~bash
tar -C /opt/ipsec -czf /root/ipsec-before-delete-$(date +%Y%m%d-%H%M%S).tar.gz \
  pki clients
~~~

## 平台包

补平台（不重签）：

~~~bash
neca platform add alice --platforms linux,macos
~~~

修改平台标注：

~~~bash
neca platform set alice --platforms windows,android
~~~

从现有证书重建全部包：

~~~bash
neca package alice --platforms windows,linux,macos,android,ios --zip
~~~

## 导出与索引同步

~~~bash
neca export issued --out /root/issued.csv
neca export revoked --out /root/revoked.csv
neca index sync
~~~

CSV 使用 UTF-8 BOM，便于 Excel 直接打开。`index sync` 只补索引与元数据，
不重签证书。

## 审计与 JSON

~~~bash
neca list --json
neca revoke alice --json
~~~

审计日志默认 `/opt/ipsec/pki/audit.log`，每行包含时间、操作者、动作、CN、
序列号与结果。JSON 输出适合脚本处理；不要把含密码的字段写进工单或日志。
