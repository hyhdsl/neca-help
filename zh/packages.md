# 平台包与客户端安装

## 平台矩阵

| 平台 | 默认产物 | 可选一键脚本 |
|---|---|---|
| Windows | `p12`、`pass`、`ca.crt`、`使用说明.txt` | `install.ps1` / `uninstall.ps1` / `connect.ps1` + `.cmd` |
| Linux | 同上 | `install.sh` / `uninstall.sh` |
| macOS | 同上 + `<CN>.mobileconfig` | `install.command` / `uninstall.command` |
| Android | `p12`、`pass`、`ca.crt`、说明 | — |
| iOS | `<CN>.mobileconfig`（CA + 身份 + IKEv2） | — |

平台名：`windows`、`linux`、`macos`、`android`、`ios`；兼容别名包括
`win`、`mac`、`darwin`、`osx`、`and`。

## 生成包

签发时直接生成：

~~~bash
neca issue alice --platforms windows,linux,macos,android,ios --zip
~~~

从已有证书重建：

~~~bash
neca package alice --platforms windows,android,ios --zip
~~~

默认不生成一键安装脚本；需要时显式开启：

~~~bash
neca issue alice --platforms windows,linux --installers windows,linux
~~~

`--installers all` 会为支持脚本的平台生成脚本，但 Linux/macOS 脚本仍应在目标
发行版/版本上人工验证。

## 压缩包与 TTL

`--zip` 会在 `/opt/ipsec/downloads/` 生成：

~~~text
<CN>-<YYYYmmdd-HHmmss>-<随机>.zip
<同名>.json
~~~

默认 30 分钟过期；Web 下载必须登录，并按 sidecar 元数据校验，过期为 410/404。
查看与清理：

~~~bash
neca downloads list
neca downloads clean --expired
neca downloads clean --all
~~~

如果客户端拿到的是过期链接，重新执行 `neca package <CN> --zip`。

## 包内不应出现

- `<CN>.key`、`ca.key`、`index.txt`、`audit.log`；
- 任何绝对路径、`..` 路径或服务器私钥；
- 真实服务器标识（示例配置使用占位值）。

压缩包包含 p12 与密码文件，按敏感凭据管理；不要通过公开 IM 群或未加密邮件发送。

## 客户端导入要点

### Windows

1. 导入 `ca.crt` 到“受信任的根证书颁发机构”；
2. 双击 `p12` 导入“个人”证书，输入 `pass` 中的密码；
3. 按组织要求配置 IKEv2 连接，或运行 `connect.ps1`（如已生成）。

### Linux / macOS

- Linux 可使用 strongSwan 的 `swanctl` 配置；脚本支持 `NECA_DRY_RUN=1` 预览。
- macOS 可导入 p12 到登录钥匙串，并安装 `.mobileconfig`。
- 一键脚本默认关闭，开启前先在测试机验证。

### Android / iOS

- Android 通过系统“安装证书”导入 p12；不同厂商菜单不同。
- iOS 安装 `.mobileconfig` 后，到“设置 → 通用 → VPN 与设备管理”确认描述文件。
- 若客户端提示密码错误，先核对 `pass` 文件与键盘输入，不要反复重签。

## 常见问题

- **没有生成安装脚本**：加 `--installers`；这是有意的默认安全行为。
- **Linux/macOS 脚本未验证**：目标系统差异大，先 dry-run，再在生产推广。
- **p12 在旧 Android 打不开**：使用默认 `compat` profile；`modern` 不保证旧设备兼容。
- **压缩包下载 410/404**：已过期或已被清理；重新生成。
