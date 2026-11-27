# ⚡ 快速開始

本頁用最短路徑完成一次「檢視憑證 → 簽發 → 驗證 → 下載」的流程。指令均以 root
在伺服器執行；範例 CN 使用 `alice`，請替換成真實使用者識別。

## 0. 前提

- 已按 [安裝、升級與復原](install.md) 部署 lite 映像與 `neca` 包裝指令。
- strongSwan 容器已存在並正常執行。
- `/opt/ipsec` 下已有 PKI；全新環境不要直接照搬正式環境指令。

~~~bash
neca version
neca config show
neca cert list
~~~

如果 `neca` 不存在，先安裝包裝指令：

~~~bash
install -m 0755 deploy/neca-wrapper.sh /usr/local/bin/neca
~~~

`deploy/neca-wrapper.sh` 預設位於原始碼儲存庫根目錄下的 `deploy/` 目錄；上面的指令
預設在儲存庫根目錄執行。若目前目錄不同，請改用該檔案的實際路徑。

## 1. 檢視目前憑證

~~~bash
neca cert list
neca cert list --json
~~~

`list` 顯示狀態、序號、平台與到期時間。憑證狀態以
`/opt/ipsec/pki/ca/index.txt` 為準。

## 2. 簽發一張憑證

先 dry-run，確認參數：

~~~bash
neca cert issue alice \
  --platforms windows,android,ios \
  --days 365 \
  --installers windows \
  --zip \
  --dry-run
~~~

確認後去掉 `--dry-run`：

~~~bash
neca cert issue alice \
  --platforms windows,android,ios \
  --days 365 \
  --installers windows \
  --zip
~~~

產物在 `/opt/ipsec/clients/alice/`；加 `--zip` 時還會在
`/opt/ipsec/downloads/` 產生帶到期時間的壓縮檔。

- 平台名稱：`windows`、`linux`、`macos`、`android`、`ios`，以逗號分隔。
- `--installers` 預設 `none`；需要一鍵指令碼時明確開啟。
- `--p12-profile compat` 是預設值，相容 MIUI／舊版 Android；`modern` 用於新用戶端。

## 3. 驗證 p12

~~~bash
neca cert verify alice
~~~

`verify` 檢查 p12 的演算法、憑證張數、密碼長度與用戶端憑證鏈。批次檢查：

~~~bash
neca cert verify
~~~

## 4. 檢視下載套件

~~~bash
neca pki downloads list
~~~

預設 TTL 為 30 分鐘；過期後下載會失敗。需要時可清理：

~~~bash
neca pki downloads clean --expired
~~~

## 5. 選用：開啟 Web 面板

Web 預設唯讀，只監聽回送位址。部署環境通常透過 SSH 本機轉送存取：

~~~bash
ssh -L 18080:127.0.0.1:18080 root@<SERVER>
# 瀏覽器開啟 http://127.0.0.1:18080
~~~

若修改過 `web.listen`，請按實際連接埠替換。詳見 [Web 面板](web.md)。

## 6. 輸出與結束代碼

- 輸出前綴：`[OK]`、`[INFO]`、`[WARN]`、`[ERR]`。
- 結束代碼：`0` 成功；`1` 作業失敗；`2` 用法錯誤；`3` 設定／環境錯誤。
- 所有指令都支援 `--json`；寫入指令支援 `--dry-run`（以實際指令說明為準）。

## 下一步

- 需要撤銷、重新簽發或刪除：[憑證管理](certificates.md)。
- 需要 Windows／Linux／macOS／Android／iOS 用戶端套件：[平台套件](packages.md)。
- 需要 Web 寫入能力：[Web 面板](web.md)。
