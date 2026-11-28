# 子命令参考

> 适用于 **neca v1.5.2**。本文覆盖全部内置子命令、参数、交互确认与退出码。
> 全局开关可放在命令任意位置；在任一命令后加 `--help` 可查看该层用法。
> 写操作建议先加 `--dry-run`，确认输出后再实际执行。

## 全局开关

| 开关 | 作用 |
|---|---|
| `--json` | 输出结构化 JSON；交互式命令在 JSON 模式下通常要求同时给 `--yes` |
| `--dry-run` | 只打印将要执行的写命令，不执行、不修改文件 |
| `--yes` / `-y` | 跳过 Y/N 确认；审计记录会标记为显式确认 |
| `-q` / `--quiet` | 仅输出错误；与 `--json` 并用时以 JSON 为准 |
| `--lang zh-CN\|zh-TW\|en-US` | 覆盖本次命令语言；未指定时读取配置 |
| `--config <文件>` | 本次命令使用指定配置文件（等价设置 `NECA_CONFIG`） |
| `--reload-mode creds\|restart` | 覆盖 CRL 重载方式（等价设置 `NECA_RELOAD_MODE`） |
| `--version` / `-V` | 显示版本、提交号、构建时间与平台 |
| `-h` / `--help` | 显示当前层帮助 |

`--dry-run` 只拦截会写状态的命令：cert 的 issue/reissue/revoke/unrevoke/trash/platform/package/export，pki index sync、pki downloads clean，trash restore/purge，以及 web 的 readonly on/off、idle-timeout 修改、export on/off、lock、lockdown。只读命令不受影响。`cert import` 是例外：`--dry-run` 由命令自身执行只读校验并输出计划，不写入。

**退出码**：`0` 成功（允许附带 WARN）；`1` 用法错误；`2` 运行期失败；`3` 前置条件不满足（证书已存在、缺 CA、helper 不可用、权限不足等）。

## 顶层命令

### neca version

~~~bash
neca version
neca --json version
~~~

显示版本、提交号、构建时间、目标平台。

### neca help

~~~bash
neca help
neca cert --help
neca cert issue --help
~~~

不带参数运行 `neca` 会进入交互向导；`neca wizard` 也可显式进入。向导顶部显示版本信息和 `neca help` 提示；一级菜单为证书管理、导入 / 导出、回收站、维护与诊断、帮助。导入 / 导出下可导入 p12、导入 PEM 证书 + 私钥、导出已签发 CSV、导出已吊销 CSV。每个动作执行前会打印等价 CLI 命令。

### neca completion

~~~bash
source <(neca completion bash)
source <(neca completion zsh)   # 需要先启用 compinit
neca completion fish | source
neca completion powershell | Out-String | Invoke-Expression
~~~

生成 bash、zsh、fish、PowerShell 补全脚本，输出到 stdout，不修改 shell 配置。补全覆盖顶层命令、命名空间子命令、别名与主要开关；PowerShell 也可使用简写 `pwsh`。持久安装时按所用 shell 的补全目录保存脚本即可。

## config

### config init

~~~bash
neca config init
neca config init --force
neca config init --from-env
~~~

- 写入配置文件；已存在时需 `--force`。
- 新文件权限为 0600；覆盖时保留原权限。
- `--from-env` 以当前环境/默认值生成配置，而不是示例模板。

### config show / validate / path

~~~bash
neca config show
neca config validate
neca config path
~~~

- `show`：输出当前生效配置的 JSON 快照，用于核对路径和 Web 设置。
- `validate`：加载并校验配置；通过时输出配置文件路径。
- `path`：只输出当前配置文件路径。

### config export-env

~~~bash
neca config export-env
~~~

输出可直接交给 shell 的环境变量赋值。输出可能包含敏感配置，请在受控终端使用，不要写入日志、工单或公开聊天。

## cert：证书管理

### cert list

~~~bash
neca cert list
neca --json cert list
~~~

列出 CN、状态、序列号、平台、到期与备注，并显示 CRL 吊销条目数。不接受额外位置参数。

### cert expiring

~~~bash
neca cert expiring
neca cert expiring --days 60
neca --json cert expiring --days 30
~~~

只读列出未来 N 天内到期的有效证书，默认 30 天；输出 CN、到期时间与剩余天数，
并显示已过期证书数量。仅包含 `valid` 状态，已吊销/未索引不进入提醒。

### cert verify

~~~bash
neca cert verify
neca cert verify alice
neca cert verify alice bob
~~~

检查客户端 p12：是否只含客户端证书、算法是否符合预期、密码长度是否合规，并使用 CA 证书校验客户端证书链。省略 CN 时检查 `clients/` 下全部客户端。CA 缺失或链校验失败会写入 stderr 告警，但退出码仍为 0（与历史脚本一致）；它不能替代客户端实机导入测试。`--json` 的每个 report 额外包含 `chainChecked`、`chainOK` 与 `chainMessage`。链校验只验证信任链与有效期，不检查 CRL 吊销状态。

### cert issue

~~~bash
neca cert issue alice --platforms windows,android --days 365 --zip
neca cert issue alice --platforms windows,linux --installers windows --p12-profile compat --zip --dry-run
~~~

| 参数 | 说明 |
|---|---|
| `<CN>` | 必填；只允许 `A-Z a-z 0-9 . _ -` |
| `--platforms a,b` | 平台列表；省略时用配置默认值 |
| `--days N` | 有效期天数；省略时用配置默认值 |
| `--installers none\|all\|列表` | 生成一键安装脚本；省略时用配置默认值（通常为 none） |
| `--p12-profile compat\|modern` | p12 算法：compat 为 3DES/SHA1，modern 为 AES-256/SHA256 |
| `--zip` | 在下载目录生成带有效期的 zip |
| `--no-reload` | 不自动重载 CRL，由你手动加载 |

私钥只留在服务器，不进入客户端 zip。签发会写审计记录。

### cert reissue

~~~bash
neca cert reissue alice --platforms windows,android --days 365 --zip
~~~

参数与 issue 相同。会归档旧证书并生成新序列号；客户端必须重新导入新包。重签前建议备份 `clients/alice/` 与 `pki/meta/alice.env`。

### cert revoke / cert unrevoke

~~~bash
neca cert revoke alice
neca cert unrevoke alice
neca cert revoke alice --no-reload
~~~

吊销或恢复证书并重建 CRL；默认通过 `swanctl --load-creds` 热加载，不重启 strongSwan。对已处于目标状态的证书会给出提示，不视为错误。`--no-reload` 跳过自动重载。

### cert trash

~~~bash
neca cert trash alice
~~~

把证书、客户端包与元数据移入回收站，不物理删除，可恢复。

### cert export

~~~bash
neca cert export issued
neca cert export revoked
neca cert export issued --out /tmp/issued.csv
~~~

导出 CSV，类型必须是 `issued` 或 `revoked`；`--out` 省略时使用默认导出路径。`issued`/`revoked` 与 `--out` 的顺序可交换。

### cert platform

~~~bash
neca cert platform add alice --platforms linux --installers linux --zip
neca cert platform set alice --platforms windows,android,linux
~~~

- `add`：为已有证书补生成新平台包；支持 `--installers` 与 `--zip`。
- `set`：只修改平台标注，不重新生成包；只接受 `--platforms`。

### cert package

~~~bash
neca cert package alice --platforms windows,android --installers windows --p12-profile compat --zip
~~~

从已有证书重建平台包，不签发新证书、不更换密钥。参数与 issue 的平台/安装器/p12 部分相同。

### cert import

~~~bash
neca cert import alice --p12 alice.p12 --password-file alice.p12.pass
neca cert import alice --cert alice.crt --key alice.key
neca cert import alice --cert alice.crt --key alice.key --key-password-file alice.key.pass --dry-run
~~~

v1.4 支持两种互斥来源：由当前 CA 签发、能通过证书链校验的 p12；或 PEM 格式的单张客户端证书 + PEM 私钥。p12 或 cert/key 必须匹配并恰好包含一个客户端证书和一个私钥；CN、序列号、clients 目录、meta 或 CRL 冲突时拒绝，不覆盖、不 repair。密码文件与私钥文件必须是权限 0600 的普通文件（Windows 仅记录 ACL 限制）；加密私钥需要 --key-password-file。

过期证书默认拒绝；使用 `--allow-expired` 并完成 Y/N 二次确认后，以 index 状态 E 导入，并提示随后执行 `neca cert reissue <CN>` 或 `neca cert revoke <CN>`。JSON 模式需要同时提供 `--allow-expired`（仅过期时）和 `--yes`。`--zip` 可在导入后生成下载包；`--platforms`/`--installers`/`--p12-profile` 与 issue 语义一致。

### cert import-batch

~~~bash
neca cert import-batch ./batch --password-dir ./secrets --dry-run
neca cert import-batch ./batch --password-dir ./secrets --yes
~~~

批次目录优先，不需要必填清单。每个 CN 一个目录，文件名为固定约定：

~~~text
batch/
  alice/client.p12                 # p12 来源
  bob/client.crt                   # 或 cert+key 来源
  bob/client.key
  bob/entry.json                   # 可选覆盖
secrets/
  alice.pass                       # p12 密码
  bob.key.pass                     # 加密私钥密码（如需要）
~~~

- p12 密码文件支持 CN.pass；兼容旧命名的 CN.p12.pass。加密 key 使用 CN.key.pass。
- 可选 entry.json 仅覆盖 platforms、installers、p12Profile、allowExpired；不要写密码或任意路径。
- 单条失败不会回滚其他成功项；退出码 3 表示至少一条失败，逐项结果仍会输出。
- 过期证书需要 allowExpired/--allow-expired；实际导入仍必须显式 --yes，--dry-run 不写文件。
- CN 目录只放固定文件；未知非隐藏文件、子目录或符号链接会让该条失败。
- 保持 CLI-only；Web/helper 不接收 p12、私钥或口令。本版本不在向导中增加批量入口。

## pki：索引与下载目录

### pki index sync

~~~bash
neca pki index sync
~~~

为手工放入的证书补索引与元数据。不接受额外参数。

### pki downloads list

~~~bash
neca pki downloads list
~~~

列出下载目录中的 zip、大小与过期时间。

### pki downloads clean

~~~bash
neca pki downloads clean
neca pki downloads clean --all
~~~

默认只删除已过期的 zip；`--all` 删除全部下载文件。该命令不会触碰证书或平台包。

## trash：回收站

~~~bash
neca trash list
neca trash restore <条目ID>
neca trash purge <条目ID>
neca trash purge --expired
~~~

- `list`：显示条目 ID、CN 与移入时间。
- `restore`：按条目 ID 恢复证书与文件。
- `purge`：物理删除；交互确认默认 N，自动化可加 `--yes`。`--expired` 按配置的保留期批量清理。
- `--json trash purge` 必须同时给 `--yes`，否则返回用法错误。

## web：Web 面板管理

### web status

~~~bash
neca web status
neca web
~~~

显示 Web 是否启用、监听地址、只读/可写模式、sessionEpoch、CSV 导出、空闲超时、Passkey 与 helper 状态。

### web readonly

~~~bash
neca web readonly status
neca web readonly on
neca web readonly off
~~~

- `on`：切换为只读（需要 root）。
- `off`：切换为可写（需要 root）；还要求 helper 可用、socket 权限正确、CA 私钥存在，避免“配置可写但没有写边界”的半状态。
- 修改后按提示发送 SIGHUP 让运行中的 Web 重载。

### web idle-timeout

~~~bash
neca web idle-timeout status
neca web idle-timeout 30m
neca web idle-timeout 1h
~~~

查看或修改空闲超时。允许范围 5m–24h，默认 30m；绝对会话上限为 24h。修改需要 root，并需 SIGHUP 重载。

### web export

~~~bash
neca web export on
neca web export off
~~~

开启或关闭 Web 的 CSV 导出（默认关闭）。需要 root；修改后需 SIGHUP 重载。

### web lock

~~~bash
neca web lock --yes
~~~

强制只读、递增 `sessionEpoch` 使全部旧会话失效，并发送 SIGHUP。需要 root；交互确认默认 N，JSON 模式必须给 `--yes`。

### web lockdown

~~~bash
neca web lockdown --yes
~~~

只停止明确命名的 `neca-web` 与 `neca-web-helper` 容器，不触碰 SSH、strongSwan 或 VPN。需要 root；交互确认默认 N，JSON 模式必须给 `--yes`。

### web hash-password

~~~bash
read -r -s pw
printf '%s' "$pw" | neca web hash-password
~~~

从 stdin 读取密码，输出 bcrypt 哈希（cost 12）；密码至少 8 个字符。不读取配置、不要求 root，供生成 `NECA_WEB_PASSWORD_HASH` 使用；密码不会进入 argv 或日志。

## doctor：诊断

~~~bash
neca doctor
neca --json doctor
~~~

检查配置、CA 证书、CA 私钥、索引、CRL、Web 配置、docker 可执行文件、审计文件与 helper。`WARN` 不影响退出码；任一 `FAIL` 返回退出码 2。lite 环境没有 CA/Web 文件时出现 FAIL 属于预期，请按部署形态判断。

## 常见组合

~~~bash
# 先预演再签发
neca --dry-run cert issue alice --platforms windows,android --zip
neca cert issue alice --platforms windows,android --zip

# 脚本读取结构化列表
neca --json cert list

# 吊销并立即热加载 CRL
neca cert revoke alice

# 自动化清理过期回收站条目
neca trash purge --expired --yes

# 排障
neca doctor
neca web status
~~~

> 安装、升级和回滚见 [安装、升级与回滚](install.md)；证书数据模型见 [证书管理](certificates.md)；Web 权限边界见 [Web 面板](web.md) 与 [安全、权限与备份](security.md)。
