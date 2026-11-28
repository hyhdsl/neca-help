# 子命令參考

> 適用於 **neca v1.6.0**。本文涵蓋全部內建子命令、參數、互動確認與結束代碼。
> 全域旗標可放在指令的任意位置；在任一指令後加 `--help` 可查看該層用法。
> 寫入操作建議先加 `--dry-run`，確認輸出後再實際執行。

## 全域旗標

| 旗標 | 作用 |
|---|---|
| `--json` | 輸出結構化 JSON；互動式指令在 JSON 模式下通常要求同時給 `--yes` |
| `--dry-run` | 只列印將要執行的寫入指令，不執行、不修改檔案 |
| `--yes` / `-y` | 跳過 Y/N 確認；稽核紀錄會標記為明確確認 |
| `-q` / `--quiet` | 僅輸出錯誤；與 `--json` 並用時以 JSON 為準 |
| `--lang zh-CN\|zh-TW\|en-US` | 覆寫本次指令語言；未指定時讀取設定 |
| `--config <檔案>` | 本次指令使用指定的設定檔（等同設定 `NECA_CONFIG`） |
| `--reload-mode creds\|restart` | 覆寫 CRL 重新載入方式（等同設定 `NECA_RELOAD_MODE`） |
| `--version` / `-V` | 顯示版本、提交號、建置時間與平台 |
| `-h` / `--help` | 顯示目前層級的說明 |

`--dry-run` 只攔截會寫入狀態的指令：cert 的 issue/reissue/revoke/unrevoke/trash/platform/package/export，pki index sync、pki downloads clean，trash restore/purge，以及 web 的 readonly on/off、idle-timeout 修改、export on/off、lock、lockdown。唯讀指令不受影響。

**結束代碼**：`0` 成功（允許附帶 WARN）；`1` 用法錯誤；`2` 執行期失敗；`3` 前置條件不滿足（憑證已存在、缺少 CA、helper 不可用、權限不足等）。

## 頂層指令

### neca version

~~~bash
neca version
neca --json version
~~~

顯示版本、提交號、建置時間、目標平台。

### neca help

~~~bash
neca help
neca cert --help
neca cert issue --help
~~~

不帶參數執行 `neca` 會進入互動精靈；`neca wizard` 也可明確進入。精靈頂部顯示版本資訊與 `neca help` 提示；一級選單為憑證管理、匯入 / 匯出、回收桶、維護與診斷、說明。匯入 / 匯出下可匯入 p12、匯入 PEM 憑證 + 私鑰、匯出已簽發 CSV、匯出已撤銷 CSV。每個動作執行前會列印等值的 CLI 指令。

### neca completion

~~~bash
source <(neca completion bash)
source <(neca completion zsh)   # 需要先啟用 compinit
neca completion fish | source
neca completion powershell | Out-String | Invoke-Expression
~~~

產生 bash、zsh、fish、PowerShell 自動補齊指令碼，輸出到 stdout，不修改 shell 設定。自動補齊涵蓋頂層指令、命名空間子命令、別名與主要旗標；PowerShell 也可使用簡寫 `pwsh`。要長期安裝時，依所用 shell 的補齊目錄保存指令碼即可。

## config

### config init

~~~bash
neca config init
neca config init --force
neca config init --from-env
~~~

- 寫入設定檔；已存在時需 `--force`。
- 新檔案權限為 0600；覆寫時保留原權限。
- `--from-env` 以目前環境／預設值產生設定，而不是使用範例範本。
- 產生 `schemaVersion: 2` 設定；v1/0 設定仍可讀取。
- 升級舊設定先執行 `neca config migrate` 檢查；確認後用 `--write` 遷移，寫入前自動產生 0600 備份。

### config migrate

~~~bash
neca config migrate --check
neca config migrate --write
~~~

- 預設只檢查；`--check` 明確只讀，不修改檔案。
- `--write` 將 v1/0 遷移到 schema v2，保留原檔案權限並建立帶時間戳的備份。
- v2 嚴格拒絕未知欄位、重複鍵、非法列舉／範圍、相對路徑與 Web 安全邊界；v1 保持寬容讀取。
- 含 v2 不識別欄位的 v1 設定會拒絕遷移，避免靜默遺失欄位。

### config show / validate / path

~~~bash
neca config show
neca config validate
neca config path
~~~

- `show`：輸出目前生效設定的 JSON 快照，包含 `schemaVersion`、遷移狀態、路徑與 Web 設定。
- `validate`：載入並驗證設定；v2 嚴格檢查未知欄位與非法值，通過時輸出設定檔路徑。
- `path`：只輸出目前設定檔路徑。

### config export-env

~~~bash
neca config export-env
~~~

輸出可直接交給 shell 的環境變數指派。輸出可能包含敏感設定，請在受控終端使用，不要寫入日誌、工單或公開聊天。

## cert：憑證管理

### cert list

~~~bash
neca cert list
neca --json cert list
~~~

列出 CN、狀態、序號、平台、到期與備註，並顯示 CRL 撤銷條目數。不接受額外的位置參數。

### cert expiring

~~~bash
neca cert expiring
neca cert expiring --days 60
neca --json cert expiring --days 30
~~~

唯讀列出未來 N 天內到期的有效憑證，預設 30 天；輸出 CN、到期時間與剩餘天數，
並顯示已過期憑證數量。僅包含 `valid` 狀態，已撤銷／未索引不進入提醒。

### cert verify

~~~bash
neca cert verify
neca cert verify alice
neca cert verify alice bob
~~~

檢查用戶端 p12：是否只含用戶端憑證、演算法是否符合預期、密碼長度是否合規，並使用 CA 憑證校驗用戶端憑證鏈。省略 CN 時檢查 `clients/` 下的全部用戶端。CA 缺失或鏈校驗失敗會寫入 stderr 警告，但結束代碼仍為 0（與既有指令碼一致）；它不能取代用戶端實機匯入測試。`--json` 的每個 report 額外包含 `chainChecked`、`chainOK` 與 `chainMessage`。鏈校驗只驗證信任鏈與有效期限，不檢查 CRL 撤銷狀態。

### cert issue

~~~bash
neca cert issue alice --platforms windows,android --days 365 --zip
neca cert issue alice --platforms windows,linux --installers windows --p12-profile compat --zip --dry-run
~~~

| 參數 | 說明 |
|---|---|
| `<CN>` | 必填；只允許 `A-Z a-z 0-9 . _ -` |
| `--platforms a,b` | 平台清單；省略時使用設定預設值 |
| `--days N` | 有效期限天數；省略時使用設定預設值 |
| `--installers none\|all\|清單` | 產生一鍵安裝指令碼；省略時使用設定預設值（通常為 none） |
| `--p12-profile compat\|modern` | p12 演算法：compat 為 3DES/SHA1，modern 為 AES-256/SHA256 |
| `--zip` | 在下載目錄產生帶有效期限的 zip |
| `--no-reload` | 不自動重新載入 CRL，由你手動載入 |

私鑰只留在伺服器，不進入用戶端 zip。簽發會寫入稽核紀錄。

### cert reissue

~~~bash
neca cert reissue alice --platforms windows,android --days 365 --zip
~~~

參數與 issue 相同。會封存舊憑證並產生新序號；用戶端必須重新匯入新套件。重新簽發前建議備份 `clients/alice/` 與 `pki/meta/alice.env`。

### cert revoke / cert unrevoke

~~~bash
neca cert revoke alice
neca cert unrevoke alice
neca cert revoke alice --no-reload
~~~

撤銷或取消撤銷，並重建 CRL；預設透過 `swanctl --load-creds` 熱重載，不重新啟動 strongSwan。對已處於目標狀態的憑證會給出提示，不視為錯誤。`--no-reload` 跳過自動重新載入。

### cert trash

~~~bash
neca cert trash alice
~~~

把憑證、用戶端套件與中繼資料移入資源回收筒，不實體刪除，可還原。

### cert export

~~~bash
neca cert export issued
neca cert export revoked
neca cert export issued --out /tmp/issued.csv
~~~

匯出 CSV，類型必須是 `issued` 或 `revoked`；省略 `--out` 時使用預設匯出路徑。`issued`／`revoked` 與 `--out` 的順序可互換。

### cert platform

~~~bash
neca cert platform add alice --platforms linux --installers linux --zip
neca cert platform set alice --platforms windows,android,linux
~~~

- `add`：為既有憑證補充產生新平台套件；支援 `--installers` 與 `--zip`。
- `set`：只修改平台標註，不重新產生套件；只接受 `--platforms`。

### cert package

~~~bash
neca cert package alice --platforms windows,android --installers windows --p12-profile compat --zip
~~~

從既有憑證重建平台套件，不簽發新憑證、不更換金鑰。參數與 issue 的平台／安裝器／p12 部分相同。

### cert import

~~~bash
neca cert import alice --p12 alice.p12 --password-file alice.p12.pass
neca cert import alice --cert alice.crt --key alice.key
neca cert import alice --cert alice.crt --key alice.key --key-password-file alice.key.pass --dry-run
~~~

支援兩種互斥來源：由目前 CA 簽發且可通過憑證鏈校驗的 p12，或 PEM 用戶端憑證 + PEM 私鑰。p12 或 cert/key 必須匹配，且恰好包含一張用戶端憑證與一把私鑰；CN、序號、clients 目錄、meta 或 CRL 衝突時拒絕，不覆寫、不修復。密碼檔與私鑰檔必須是權限 0600 的一般檔案（Windows 僅有 ACL 限制）；加密私鑰需使用 `--key-password-file`。

過期憑證預設拒絕；使用 `--allow-expired` 並完成 Y/N 二次確認後，以 index 狀態 E 匯入，並提示 `neca cert reissue <CN>` 或 `neca cert revoke <CN>`。JSON 模式需同時提供 `--allow-expired`（僅過期時）與 `--yes`。`--zip`、`--platforms`、`--installers`、`--p12-profile` 與 issue 語意相同。

### cert import-batch

~~~bash
neca cert import-batch ./batch --password-dir ./secrets --dry-run
neca cert import-batch ./batch --password-dir ./secrets --yes
~~~

以批次目錄為準，不需要必填清單。每個 CN 一個目錄，檔名固定：

~~~text
batch/
  alice/client.p12
  bob/client.crt
  bob/client.key
  bob/entry.json
secrets/
  alice.pass
  bob.key.pass
~~~

- p12 密碼檔支援 `CN.pass`；也相容舊命名 `CN.p12.pass`。加密金鑰使用 `CN.key.pass`。
- 可選 `entry.json` 僅覆寫 `platforms`、`installers`、`p12Profile`、`allowExpired`；不要寫密碼或任意路徑。
- 單筆失敗不會回滾其他成功項目；結束代碼 3 表示至少一筆失敗，仍會輸出逐項結果。
- 過期憑證需要 `allowExpired`／`--allow-expired`；實際匯入仍必須明確 `--yes`；`--dry-run` 不寫入檔案。
- CN 目錄只放固定檔案；未知非隱藏檔案、子目錄或符號連結會讓該筆失敗。
- 維持 CLI-only；Web／helper 不接收 p12、私鑰或密碼。精靈沒有批次入口。

## pki：索引與下載目錄

### pki index sync

~~~bash
neca pki index sync
~~~

為手動放入的憑證補索引與中繼資料。不接受額外參數。

### pki downloads list

~~~bash
neca pki downloads list
~~~

列出下載目錄中的 zip、大小與過期時間。

### pki downloads clean

~~~bash
neca pki downloads clean
neca pki downloads clean --all
~~~

預設只刪除已過期的 zip；`--all` 刪除全部下載檔案。此指令不會觸碰憑證或平台套件。

## trash：資源回收筒

~~~bash
neca trash list
neca trash restore <條目ID>
neca trash purge <條目ID>
neca trash purge --expired
~~~

- `list`：顯示條目 ID、CN 與移入時間。
- `restore`：依條目 ID 還原憑證與檔案。
- `purge`：實體刪除；互動確認預設為 N，自動化可加 `--yes`。`--expired` 依設定的保留期批次清理。
- `--json trash purge` 必須同時給 `--yes`，否則傳回用法錯誤。

## web：Web 面板管理

### web status

~~~bash
neca web status
neca web
~~~

顯示 Web 是否啟用、監聽位址、唯讀／可寫模式、sessionEpoch、CSV 匯出、閒置逾時、Passkey 與 helper 狀態。

### web readonly

~~~bash
neca web readonly status
neca web readonly on
neca web readonly off
~~~

- `on`：切換為唯讀（需要 root）。
- `off`：切換為可寫（需要 root）；還要求 helper 可用、socket 權限正確、CA 私鑰存在，避免「設定可寫但沒有寫入邊界」的半開啟狀態。
- 修改後依提示送出 SIGHUP，讓執行中的 Web 重新載入。

### web idle-timeout

~~~bash
neca web idle-timeout status
neca web idle-timeout 30m
neca web idle-timeout 1h
~~~

查看或修改閒置逾時。允許範圍 5m–24h，預設 30m；絕對工作階段上限為 24h。修改需要 root，並需 SIGHUP 重新載入。

### web export

~~~bash
neca web export on
neca web export off
~~~

開啟或關閉 Web 的 CSV 匯出（預設關閉）。需要 root；修改後需 SIGHUP 重新載入。

### web lock

~~~bash
neca web lock --yes
~~~

強制唯讀、遞增 `sessionEpoch` 使全部舊工作階段失效，並送出 SIGHUP。需要 root；互動確認預設為 N，JSON 模式必須給 `--yes`。

### web lockdown

~~~bash
neca web lockdown --yes
~~~

只停止明確命名的 `neca-web` 與 `neca-web-helper` 容器，不觸碰 SSH、strongSwan 或 VPN。需要 root；互動確認預設為 N，JSON 模式必須給 `--yes`。

### web hash-password

~~~bash
read -r -s pw
printf '%s' "$pw" | neca web hash-password
~~~

從 stdin 讀取密碼，輸出 bcrypt 雜湊（cost 12）；密碼至少 8 個字元。不讀取設定、不要求 root，供產生 `NECA_WEB_PASSWORD_HASH` 使用；密碼不會進入 argv 或日誌。

## doctor：診斷

~~~bash
neca doctor
neca --json doctor
~~~

檢查設定、CA 憑證、CA 私鑰、索引、CRL、Web 設定、docker 可執行檔、稽核檔案與 helper。`WARN` 不影響結束代碼；任一 `FAIL` 傳回結束代碼 2。lite 環境沒有 CA／Web 檔案時出現 FAIL 屬於預期，請依部署形態判斷。

## 常見組合

~~~bash
# 先預演再簽發
neca --dry-run cert issue alice --platforms windows,android --zip
neca cert issue alice --platforms windows,android --zip

# 指令碼讀取結構化清單
neca --json cert list

# 撤銷並立即熱重載 CRL
neca cert revoke alice

# 自動化清理過期資源回收筒條目
neca trash purge --expired --yes

# 疑難排解
neca doctor
neca web status
~~~

> 安裝、升級和復原見 [安裝、升級與復原](install.md)；憑證資料模型見 [憑證管理](certificates.md)；Web 權限邊界見 [Web 面板](web.md) 與 [安全、權限與備份](security.md)。
