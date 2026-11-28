# GPG 验证

> 适用于 neca v1.5.1 及之后的正式 Release。发布签名分两条独立的链：Release 资产的 `checksums.txt` 和 Git tag。两条链使用不同密钥，不能互相替代。

## 1. 先理解两条签名链

| 对象 | 签名密钥 | 指纹 | 用途 |
|---|---|---|---|
| Release 资产 `checksums.txt` | neca release signing (2026) | `E514EB7C153AA24A3E4241F846CC03EE21F66795` | 校验全部发布二进制 |
| Git tag | 既有个人 neca 密钥 | `4DE8259A9C3E832137506D3C9398D6D10FD5A903` | 校验 tag 指向的提交 |

从 v1.5.1 起，Release 资产由专用 Ed25519 发布密钥签名；v1.5.0 及更早版本可能使用既有 neca 密钥。Git tag 仍使用既有个人密钥。

验证时不要只看 gpg 输出中的 `Good signature`，还必须核对实际签名者指纹。

## 2. 下载验证材料

以当前稳定版 v1.6.0 为例；其他版本把版本号替换为该 Release 的 tag：

~~~bash
gh release download v1.6.0 --repo hyhdsl/neca \
  --pattern checksums.txt \
  --pattern checksums.txt.asc \
  --pattern neca-gpg-key.asc
~~~

Release 资产中的 `neca-gpg-key.asc` 与仓库中的 `keys/neca-release-signing.asc` 都是公钥入口。建议同时通过两个渠道核对指纹。

## 3. 导入公钥并核对指纹

~~~bash
gpg --import neca-gpg-key.asc
gpg --fingerprint E514EB7C153AA24A3E4241F846CC03EE21F66795
~~~

预期看到：

~~~text
E514 EB7C 153A A24A 3E42 41F8 46CC 03EE 21F6 6795
neca release signing (2026) <release@neca.invalid>
~~~

如果指纹不一致，立即停止，不要执行下载的二进制。

## 4. 验证 Release 资产

~~~bash
gpg --verify checksums.txt.asc checksums.txt
sha256sum -c checksums.txt
~~~

只有同时满足以下条件才算通过：

1. `checksums.txt.asc` 是有效签名；
2. 签名者指纹是 `E514EB7C153AA24A3E4241F846CC03EE21F66795`；
3. `sha256sum -c` 全部通过。

## 5. 验证 Git tag

Git tag 使用既有个人 neca 密钥，与 Release 资产签名是两条验证链：

~~~bash
git fetch --tags
git tag -v v1.6.0
~~~

预期签名者指纹：

~~~text
4DE8259A9C3E832137506D3C9398D6D10FD5A903
~~~

如果只验证 Release 资产，不验证 tag，不能说明构建来源与 tag 指向的提交完全一致。

## 6. 常见问题

### gpg 显示 Good signature，但指纹不同

不要继续。`Good signature` 只说明签名与某个已导入公钥匹配，不说明该公钥就是 neca 发布密钥。

### 无法导入公钥

确认下载的是 `neca-gpg-key.asc`，而不是签名文件。Windows 可使用 Kleopatra 导入，也可以使用 GnuPG 命令行；不同安装方式的 `gpg` 路径可能不同。

### sha256sum 失败

删除已下载二进制并重新下载，不要尝试修复或忽略校验失败。

### 需要验证旧版本

v1.5.0 及更早版本的资产签名可能使用既有 neca 密钥，请以对应 Release 的公钥和说明为准。历史 Release 不重签。

## 7. 公钥来源

- 仓库：`keys/neca-release-signing.asc`
- Release：`neca-gpg-key.asc`
- 公钥指纹：`E514EB7C153AA24A3E4241F846CC03EE21F66795`

公钥没有 CA 证书链；它的可信度来自仓库、Release 与指纹的跨渠道核对。

## 相关页面

- [安装、升级与回滚](install.md)
- [安全、权限与备份](security.md)
- [故障排查](troubleshooting.md)
