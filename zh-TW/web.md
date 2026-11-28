# 🖥️ Web 面板

## 定位

Web 面板是選用元件，預設**唯讀**，用於檢視憑證、驗證 p12、查看下載與管理工作階段。
它不取代 strongSwan，也不預設暴露公網。

| 元件 | 角色 | 權限 |
|---|---|---|
| `necad` | Web 伺服器 | 非 root；唯讀掛載設定、索引、meta、clients、downloads |
| `web-helper` | 寫入操作代理 | root；只接受 issue/revoke/unrevoke/reissue 允許清單 |
| lite CLI | 管理指令 | root；`neca web ...` |

Web 容器沒有 `ca.key`、用戶端私鑰或 `docker.sock`；helper 才掛載這些資產。

## 啟用唯讀 Web

### 1. 設定

在 `/opt/ipsec/config/neca.json` 中：

~~~json
{
  "schemaVersion": 2,
  "web": {
    "enabled": true,
    "listen": "127.0.0.1:18080",
    "readOnly": true,
    "idleTimeout": "30m",
    "adminCNs": [],
    "passkey": { "enabled": false, "rpId": "", "rpName": "neca Admin" }
  }
}
~~~

- `idleTimeout` 允許 5m–24h；絕對工作階段上限固定 24h。
- `readOnly` 預設 true。
- `adminCNs` 僅在反向代理 mTLS 情境使用。

### 2. 初始化 secrets

~~~bash
cd /opt/ipsec
NECA_IMAGE=ghcr.io/hyhdsl/neca:v1.6.0 ./init-web-secrets.sh
~~~

指令碼以互動方式讀取管理員密碼並建立 `web-secrets/`、`session.key`、
`password.hash`、`web-audit.log`，並確保 `neca-web` 群組 GID=10001。密碼不回顯，
也不寫入 `neca.env`；請保存到密碼管理工具。若部署流程另外產生了初始密碼檔，
首次登入後刪除它。

### 3. 啟動

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web up -d web
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web logs -f web
~~~

### 4. 存取

預設透過系統原生 OpenSSH 本機轉送：

~~~bash
ssh -L 18080:127.0.0.1:18080 root@<SERVER>
# 瀏覽器開啟 http://127.0.0.1:18080
~~~

不要把 `127.0.0.1:18080` 直接改成 `0.0.0.0`
暴露到公網；如確需反向代理，見下一節。

## 登入與工作階段

- 管理員密碼以 bcrypt 雜湊保存，不保存明文。
- 工作階段 Cookie 使用 HMAC 簽章；HttpOnly、SameSite=Lax，HTTPS 下加 Secure。
- 閒置逾時預設 30 分鐘；登入後絕對上限 24 小時。
- 登入失敗有速率限制；寫入操作強制 CSRF。
- 登出後工作階段失效；重新啟動 necad 也會使記憶體中的工作階段失效（除非實作持久化）。

## 到期概覽

登入後的概覽頁顯示 30/60/90 天累計到期卡片；點選卡片進入憑證頁並套用對應
到期區間。憑證頁可組合搜尋、狀態與到期區間篩選。

概覽與篩選都是唯讀操作：不會自動續期、撤銷或修改憑證。已撤銷／未索引憑證
不進入到期提醒；已過期憑證單獨計數。

## 寫入能力（選用）

預設唯讀。需要在 Web 內簽發／撤銷／重新簽發時：

~~~bash
cd /opt/ipsec
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write up -d web web-helper
neca web status
neca web readonly off
docker kill --signal=HUP neca-web
~~~

關閉寫入能力：

~~~bash
neca web readonly on
docker kill --signal=HUP neca-web
~~~

**重要**：`readOnly` / `idleTimeout` 修改後送出 SIGHUP 即可讓執行中的 necad 重新載入：
`docker kill --signal=HUP neca-web`。`web.listen`、`adminCNs`、passkey 變更仍需
重新啟動 web 容器。

Web 寫入操作包含 issue/revoke/unrevoke/reissue，以及「移入資源回收筒」；簽發頁平台使用核取方塊（windows/linux/macos/android/ios），並可單獨勾選「一鍵安裝指令碼」；預設不產生，僅 Windows/Linux/macOS 支援，由使用者本次簽發決定，不受全域 `NECA_INSTALLERS` 自動影響。設定頁僅支援允許清單中的高風險項目（readOnly/idleTimeout/匯出）；**不支援實體 purge 或 package**。
helper 的 `readOnly` 檢查是第二道關卡，不能只依賴 UI 隱藏按鈕。

## 管理指令

~~~bash
neca web status
neca web readonly status
neca web readonly on
neca web readonly off
neca web idle-timeout status
neca web idle-timeout 30m
neca web hash-password
neca web export on
neca web export off
neca web lock
neca web lockdown
~~~

- `web lock`：原子設定唯讀並遞增 `web.sessionEpoch`，送出 SIGHUP；舊工作階段全部失效，服務保留。預設 Y/N 為 N；背景自動化可用 `--yes`。
- `web lockdown`：只停止明確命名的 `web` 與 `web-helper`，不觸碰 SSH/strongSwan/VPN；預設 Y/N 為 N。
- `web export on/off`：預設關閉；只提供 issued/revoked CSV。關閉時 `/export/*.csv` 與 `/api/v1/export/*.csv` 傳回 404。開啟後憑證頁顯示「匯出已簽發 CSV」與「匯出已撤銷 CSV」按鈕；也可直接存取對應 URL。

側邊欄「設定」可展開，下分為「一般」「安全性」「關於」。一般頁使用下拉式選單選擇主題與語言；關於頁顯示版本、儲存庫、說明、授權條款與著作權；右上角重複的下載中心按鈕已移除。高風險設定預設唯讀。若後端在設定中設定 `web.allowHighRiskSettings=true`，且 Web／helper 可寫，可在「安全性」頁用下拉式選單修改 readOnly、idleTimeout 與 CSV 匯出；點選「儲存設定」後，在確認對話框重新輸入管理員密碼、勾選 Y 並確認儲存。未儲存就離開頁面會觸發瀏覽器原生提醒。把唯讀改為「是」會立即關閉 Web 寫入模式與高風險修改，頁面隨後顯示唯讀狀態；恢復寫入模式只能由後端 root 執行。`adminCNs`、passkey 與 helper 設定不支援 Web 修改。

`readonly off` 只允許 root，並檢查：

1. helper 健康；
2. helper socket 權限；
3. `ca.key` 存在；
4. 設定原子寫入成功。

任何一步失敗都不應產生「半開啟」狀態。

## 反向代理與 mTLS（選用）

如果設定了 `web.adminCNs`，必須同時設定 `NECA_WEB_PROXY_SECRET`，由反向代理
轉送：

~~~text
X-Neca-Proxy-Secret: <共用金鑰>
X-Neca-Client-CN:   <已驗證的管理員憑證 CN>
~~~

necad 只接受來自回送位址代理的請求，並清除用戶端偽造標頭。TLS 終止後設定
`NECA_WEB_SECURE_COOKIES=1` 並重新啟動 web；以 OpenSSH 本機轉送 HTTP 時保持未設定。
本專案預設未啟用公網網域名稱／TLS／mTLS，保留為後續選項。

## 語言與主題

- 頁面支援 zh-CN / zh-TW / en-US 切換，使用 `neca_lang` Cookie。
- 預設語言跟隨 `neca.json` 的 `language`。
- 支援淺色／深色主題，選擇保存在瀏覽器 localStorage。
- 主題與語言切換位於「設定 → 一般」，不改變伺服器端權限或唯讀狀態。

## 安全邊界

- Web 容器非 root、唯讀根檔案系統、`no-new-privileges`、`cap_drop: ALL`。
- 不掛載 `ca.key`、用戶端私鑰、`docker.sock`。
- helper 是唯一的 root 寫入邊界；取得 helper 等同取得高權限，必須限制 socket 存取。
- helper 冪等快取預設寫入 `/run/neca/web-helper-cache.json`（0600，TTL 24h）；
  檔案損毀時忽略並重建，不會阻塞寫入操作。
- 預設不對外發布連接埠；優先使用 SSH 通道。
- Passkey 僅保留介面與設定開關，目前未註冊登入路由。
- SIGHUP 支援 readOnly/idleTimeout 熱重載；listen/adminCNs/passkey 變更需重新啟動。

## 疑難排解入口

- [疑難排解](troubleshooting.md)
- [安全、權限與備份](security.md)
- 部署、升級與復原見安裝頁。
