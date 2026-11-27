# 🛡️ 安全、權限與備份

## 資產分級

| 資產 | 敏感度 | 要求 |
|---|---|---|
| `pki/ca.key` | 最高 | `0600 root:root`；不進入 Web 容器、不備份到公開位置 |
| 用戶端私鑰 / p12 | 高 | 僅授權使用者可取；下載套件有 TTL |
| `web-secrets/session.key` | 高 | 可偽造工作階段，請視為金鑰管理 |
| `web-secrets/password.hash` | 高 | 只存 bcrypt 雜湊，不存明文 |
| `pki/ca/index.txt`、CRL | 中 | 完整性重要；CRL 可公開 |
| 稽核日誌 | 高 | 僅附加，限制讀取，保留足夠時間 |

## 目錄與權限

啟用 Web 時的典型權限：

~~~text
/opt/ipsec/config/neca.json       root:neca-web 0640
/opt/ipsec/pki/ca/                root:neca-web 2755 (setgid)
  index.txt                       root:neca-web 0640
/opt/ipsec/clients/               root:neca-web 2750 (setgid)
/opt/ipsec/pki/meta/              root:neca-web 2750 (setgid)
  *.env                           root:neca-web 0640
/opt/ipsec/downloads/             root:neca-web 2750 (setgid)
/opt/ipsec/exports/               root:root     0700
  *.csv                           root:root     0600
/opt/ipsec/web-secrets/           root:neca-web 0710
  session.key                     root:neca-web 0640
  password.hash                   root:neca-web 0640
  web-audit.log                   root:neca-web 0660
/run/neca/web-helper.sock         root:neca-web 0660
~~~

Web 程序固定 UID/GID 10001，非 root。CLI 以 root 執行 CA 操作。lite 容器掛載
`docker.sock` 等同宿主 root，必須限制誰能執行 `neca`。

`pki/ca` 使用 setgid 與 `neca-web` 群組：CLI 原子重寫 `index.txt` 時，新檔案仍繼承
該群組；否則 Web 會回報 `讀取索引: ... permission denied`。

## Web 的邊界

- Web 容器以唯讀方式掛載必要檔案；不整體掛載 `/opt/ipsec/pki`。
- 不掛載 `ca.key`、用戶端私鑰、`docker.sock`。
- 寫入操作交給 root helper；helper 只接受允許清單中的操作。
- Web 側 `readOnly` 與 helper 側 `readOnly` 雙重檢查。
- UI 隱藏按鈕不是安全邊界；伺服器端必須拒絕。
- 預設只監聽 `127.0.0.1`，不對外發布連接埠。

## 網路存取

建議順序：

1. 系統原生 OpenSSH 本機轉送；
2. 連上系統原生 IKEv2 VPN 後存取；
3. 反向代理 TLS + 可選 mTLS。

不要引入 Tailscale／WireGuard 等第三方 VPN 用戶端作為本專案的前提。若使用反向
代理，必須限制來源、校驗代理共用金鑰與用戶端 CN，並防止偽造標頭。

## 工作階段與金鑰輪替

- 閒置逾時預設 30m；絕對上限 24h。
- 修改 `readOnly`／`idleTimeout` 後送出 SIGHUP（`docker kill --signal=HUP neca-web`）；`web.listen`／`adminCNs`／passkey 仍需重新啟動。
- Passkey 目前僅有設定與介面預留，未註冊登入路由。
- 輪替 `session.key` 會使既有工作階段失效；請在維護視窗執行。
- 管理員密碼透過 `neca web hash-password` 產生雜湊，不要寫進 `neca.env`。
- 若部署流程產生了初始密碼檔，它只應短期存在；首次登入後刪除或轉存到密碼管理工具。

## 資源回收筒與安全刪除（已實作）

- Web 刪除只把已撤銷憑證**移入資源回收筒**，實體清理只能由背景確認執行。
- 資源回收筒預設不限時保存；保留期可由使用者設定；還原已撤銷憑證不會自動取消撤銷狀態。
- 背景 purge 需要二次確認。
- **不保證安全抹除**：SSD、檔案系統快照、備份與物件儲存副本可能保留資料；資源回收筒只用於隔離與防止誤刪。

## 高風險設定（已實作）

- 高風險安全設定預設只允許由後端修改；Web 設定頁僅唯讀顯示。
- 即使後端開啟 Web 修改開關，前後端修改高風險設定也都需要 Y/N 二次確認、重新驗證、CSRF 與稽核。
- 建議 `adminCNs`、passkey 與 helper 相關設定永遠保持後端專用。

## 備份安全

備份包含 CA 私鑰，必須：

- 使用 GPG 等工具加密後再寫入磁碟；
- 口令存入密碼管理工具，不寫入指令、Git 或一般文件；
- 限制 root 讀取；
- 與正式主機分開保存；
- 定期驗證可還原；
- 不進入公開儲存庫、公開物件儲存或 CI artifact。

最小驗證：先解密到暫存目錄並檢查 `pki/ca/index.txt`、設定與 Web 金鑰可讀，
再清理暫存目錄；不要只檢查「檔案存在」。

## 稽核與監控

- 關注 `audit.log` 中的 issue/revoke/reissue/delete 與失敗紀錄。
- 關注 Web 登入失敗、CSRF 拒絕、helper 不可用與唯讀切換。
- 記錄部署前後 `index.txt` / CRL 雜湊，便於解釋變化。
- 不記錄密碼、Cookie、私鑰內容。

## 供應鏈與授權條款

- 固定映像版本；正式環境不要用 `latest`。
- 只從官方儲存庫／GHCR 拉取；核對 checksums。
- 專案以 **AGPL-3.0-only** 發布；散布或網路服務須履行原始碼公開義務。
- 程式碼與文件含 AI 輔助生成內容；使用前請自行審查。

## 上線前檢查清單

- [ ] `ca.key` 仍為 `0600 root:root`
- [ ] Web 未掛載 `ca.key` / 用戶端私鑰 / `docker.sock`
- [ ] Web 未對外發布連接埠
- [ ] `readOnly=true` 或已明確開啟寫入能力
- [ ] helper socket 僅 `root:neca-web` 可存取
- [ ] 初始密碼檔已處理
- [ ] 備份與還原演練完成
- [ ] 稽核日誌與監控已接入
- [ ] 版本標籤固定，可復原的映像可用
- [ ] `scripts/check-deploy-permissions.sh` 通過（或 `make check-permissions`）
