# 常用命令速查

> 适用于 neca v1.1.2。
> 写操作先加 `--dry-run`；交互确认默认 N，自动化可显式加 `--yes`（会审计）。

## 版本、配置与诊断

~~~bash
neca version
neca config validate
neca config show
neca doctor
~~~

## 查看证书

~~~bash
neca cert list
neca cert list --json
neca cert expiring --days 30
neca cert verify alice
neca cert verify alice --json
~~~

## 签发与重签

~~~bash
neca cert issue alice --platforms windows,android --zip --dry-run
neca cert issue alice --platforms windows,android --zip
neca cert reissue alice --platforms windows,android --days 365
~~~

## 吊销与恢复

~~~bash
neca cert revoke alice --yes
neca cert unrevoke alice --yes
~~~

## 回收站

~~~bash
neca cert trash alice
neca trash list
neca trash restore <条目ID>
neca trash purge <条目ID> --yes
~~~

清理过期条目：

~~~bash
neca trash purge --expired --yes
~~~

## 平台与索引

~~~bash
neca cert platform add alice --platforms linux
neca cert platform set alice --platforms windows,android,linux
neca pki index sync
~~~

## Web 面板

~~~bash
neca web status
neca web readonly status
neca web readonly off
neca web readonly on
neca web lock --yes
neca web lockdown --yes
~~~

- `readonly off` 需要 root、helper 可用和 CA 私钥存在。
- `lock` 会设置只读、递增 sessionEpoch，使旧会话失效。
- `lockdown` 只停止 Web/helper，不触碰 SSH、strongSwan 或 VPN。

## 导出与日常维护

~~~bash
neca cert export issued
neca cert export revoked
neca pki downloads list
neca doctor
~~~

## 语言、JSON 与自动化

~~~bash
neca --lang en-US help
neca --lang en-US cert list
neca --json cert list
neca --dry-run cert issue alice --platforms windows
~~~

> 完整参数与全部子命令见 [子命令参考](commands.md)；生产部署、升级和回滚见 [安装、升级与回滚](install.md)；Web 权限与备份见 [安全、权限与备份](security.md)。
