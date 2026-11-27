# 常見問題

## neca 是 VPN 伺服器嗎？

不是。neca 管理 strongSwan IKEv2 的使用者憑證與用戶端套件，不包含 VPN 伺服器本身。
strongSwan 的部署、防火牆與 IKEv2 設定不在本專案範圍內。

## lite 和 neca-web 一定要一起安裝嗎？

不必。lite 是預設形態；neca-web 是選用的面板套件。需要 Web 時再拉取同版本映像；
不啟用 Web 不影響 CLI。

## Web 預設可以寫入嗎？

不行。`readOnly=true` 是預設值。需要 helper 健康並明確執行 `neca web readonly off`，
再送出 SIGHUP（`docker kill --signal=HUP neca-web`），才會開放 issue/revoke/unrevoke/reissue。

## Web 可以刪除憑證嗎？

不行。刪除只在 CLI 中提供，且需要 `--yes`；已撤銷憑證還需要 `--purge-revoked`。
這是刻意的風險控制。

## 為什麼 Web 連接埠是 18080？

程式預設監聽 `127.0.0.1:18080`；以 `neca.json` 的 `web.listen` 為準。若連接埠被占用，
可改為其他回送連接埠，並重新啟動 `neca-web`。

## Web 可以直接暴露公網嗎？

不建議。預設只監聽回送位址，優先使用 OpenSSH 本機轉送。確需公網存取時，使用系統
原生 IKEv2 + 反向代理 TLS，並考慮 mTLS；不要把應用程式直接綁到 `0.0.0.0`。

## 支援多使用者／RBAC 嗎？

目前是單一管理員模型，不支援多使用者、角色或依憑證歸屬授權。多使用者是後續階段。

## 支援 Passkey 嗎？

目前只保留介面與 `passkey` 設定開關，未註冊登入路由。啟用需要 HTTPS、穩定網域名稱
與 RP ID，需另行設計與審查。

## 為什麼不用 SQLite？

M6 的唯讀查詢與簽章 Cookie 工作階段不需要資料庫；索引仍是 `index.txt`。SQLite 留到
有明確並行／多使用者需求時再評估。

## 資料在哪裡？

主要在 `/opt/ipsec`：

- `pki/`：CA、索引、meta、稽核；
- `clients/`：用戶端憑證套件；
- `swanctl/x509crl/`：CRL；
- `downloads/`：暫時 zip；
- `web-secrets/`：Web 工作階段金鑰與密碼雜湊。

## 升級會遺失憑證嗎？

不會，前提是保留 `/opt/ipsec` 資料並正確掛載。升級前仍應備份。不要執行刪除
資料目錄的指令，也不要在 lite／neca-web 之間混用不相容版本。

## 如何備份？

~~~bash
tar -C /opt/ipsec -czf /root/ipsec-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  pki clients swanctl neca.json web-secrets
~~~

備份含 CA 私鑰，必須加密並限制存取。

## 舊 bash 指令碼還能用嗎？

保留在 `legacy/`，用於復原、影子測試與歷史參考。新部署應使用 neca；舊指令碼的
遷移與復原說明見儲存庫 legacy/ 目錄與 v0.5.0 遷移表（內部紀錄，不隨儲存庫發布）。

## 如何參與貢獻？

請先透過 Issue 說明問題或變更範圍；提交 PR 前請閱讀開發政策與 AGPL 要求。維護者會
依範圍、測試與授權條款影響決定是否合併。

## 授權條款是什麼？

**AGPL-3.0-only**。允許商業使用，但散布、修改或透過網路提供服務時須履行原始碼
公開與授權保留義務。詳見 [LICENSE](https://github.com/hyhdsl/neca/blob/main/LICENSE)。

## 說明文件與程式碼是 AI 產生的嗎？

程式碼、測試與文件有 AI 程式設計助理參與；可能存在錯誤。使用前請審查、測試並自行
承擔風險。專案在 README 與 NOTICE 中有完整揭露。
