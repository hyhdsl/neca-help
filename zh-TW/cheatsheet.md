# 常用指令速查

> 適用於 neca v1.5.2。
> 寫入操作先加 `--dry-run`；互動確認預設為 N，自動化可明確加 `--yes`（會留下稽核紀錄）。

## 版本、設定與診斷

~~~bash
neca version
neca config validate
neca config show
neca doctor
~~~

## 檢視憑證

~~~bash
neca cert list
neca cert list --json
neca cert expiring --days 30
neca cert verify alice
neca cert verify alice --json
~~~

## 簽發與重新簽發

~~~bash
neca cert issue alice --platforms windows,android --zip --dry-run
neca cert issue alice --platforms windows,android --zip
neca cert reissue alice --platforms windows,android --days 365
~~~

## 撤銷與取消撤銷

~~~bash
neca cert revoke alice --yes
neca cert unrevoke alice --yes
~~~

## 資源回收筒

~~~bash
neca cert trash alice
neca trash list
neca trash restore <條目ID>
neca trash purge <條目ID> --yes
~~~

清理過期條目：

~~~bash
neca trash purge --expired --yes
~~~

## 平台與索引

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

- `readonly off` 需要 root、helper 可用和 CA 私鑰存在。
- `lock` 會設定唯讀、遞增 sessionEpoch，使舊工作階段失效。
- `lockdown` 只停止 Web／helper，不觸碰 SSH、strongSwan 或 VPN。

## 匯出與日常維護

~~~bash
neca cert export issued
neca cert export revoked
neca pki downloads list
neca doctor
~~~

## 語言、JSON 與自動化

~~~bash
neca --lang en-US help
neca --lang en-US cert list
neca --json cert list
neca --dry-run cert issue alice --platforms windows
~~~

> 完整參數與全部子命令見 [子命令參考](commands.md)；正式部署、升級和復原見 [安裝、升級與復原](install.md)；Web 權限與備份見 [安全、權限與備份](security.md)。
