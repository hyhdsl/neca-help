# ⚡ 快速开始

本页用最短路径完成一次“查看证书 → 签发 → 验证 → 下载”的流程。命令均以 root
在服务器执行；示例 CN 使用 `alice`，请替换成真实用户标识。

## 0. 前提

- 已按 [安装、升级与回滚](install.md) 部署 lite 镜像与 `neca` 包装命令。
- strongSwan 容器已存在并正常运行。
- `/opt/ipsec` 下已有 PKI；全新环境不要直接照搬生产命令。

~~~bash
neca version
neca config show
neca cert list
~~~

如果 `neca` 不存在，先安装包装命令：

~~~bash
install -m 0755 deploy/neca-wrapper.sh /usr/local/bin/neca
~~~

`deploy/neca-wrapper.sh` 默认位于源码仓库根目录下的 `deploy/` 目录；上面的命令
默认在仓库根目录执行。若当前目录不同，请改用该文件的实际路径。

## 1. 查看当前证书

~~~bash
neca cert list
neca cert list --json
~~~

`list` 显示状态、序列号、平台与到期时间。证书状态以
`/opt/ipsec/pki/ca/index.txt` 为准。

## 2. 签发一张证书

先 dry-run，确认参数：

~~~bash
neca cert issue alice \
  --platforms windows,android,ios \
  --days 365 \
  --installers windows \
  --zip \
  --dry-run
~~~

确认后去掉 `--dry-run`：

~~~bash
neca cert issue alice \
  --platforms windows,android,ios \
  --days 365 \
  --installers windows \
  --zip
~~~

产物在 `/opt/ipsec/clients/alice/`；加 `--zip` 时还会在
`/opt/ipsec/downloads/` 生成带过期时间的压缩包。

- 平台名：`windows`、`linux`、`macos`、`android`、`ios`，逗号分隔。
- `--installers` 默认 `none`；需要一键脚本时显式开启。
- `--p12-profile compat` 是默认值，兼容 MIUI/旧 Android；`modern` 用于新客户端。

## 3. 验证 p12

~~~bash
neca cert verify alice
~~~

`verify` 检查 p12 的算法、证书数量与密码长度。批量检查：

~~~bash
neca cert verify
~~~

## 4. 查看下载包

~~~bash
neca pki downloads list
~~~

默认 TTL 为 30 分钟；过期后下载会失败。需要时可清理：

~~~bash
neca pki downloads clean --expired
~~~

## 5. 可选：打开 Web 面板

Web 默认只读，监听回环地址。生产环境通常通过 SSH 本地转发访问：

~~~bash
ssh -L 18080:127.0.0.1:18080 root@<SERVER>
# 浏览器打开 http://127.0.0.1:18080
~~~

若修改过 `web.listen`，请按实际端口替换。详见 [Web 面板](web.md)。

## 6. 输出与退出码

- 输出前缀：`[OK]`、`[INFO]`、`[WARN]`、`[ERR]`。
- 退出码：`0` 成功；`1` 业务失败；`2` 用法错误；`3` 配置/环境错误。
- 所有命令支持 `--json`；写命令支持 `--dry-run`（以实际命令帮助为准）。

## 下一步

- 需要吊销、重签或删除：[证书管理](certificates.md)。
- 需要 Windows/Linux/macOS/Android/iOS 客户端包：[平台包](packages.md)。
- 需要 Web 写能力：[Web 面板](web.md)。
