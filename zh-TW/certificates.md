# 憑證管理

## 資料模型

neca 不把憑證狀態放進資料庫，仍使用 OpenSSL 相容的檔案：

| 路徑 | 作用 |
|---|---|
| `/opt/ipsec/pki/ca/index.txt` | 憑證索引；狀態、到期、序號的唯一真實來源 |
| `/opt/ipsec/pki/meta/<CN>.env` | 平台、簽發時間、到期時間、序號等中繼資料 |
| `/opt/ipsec/clients/<CN>/` | 用戶端憑證、p12、平台套件 |
| `/opt/ipsec/swanctl/x509crl/ca.crl.pem` | 供 strongSwan 載入的 CRL |
| `/opt/ipsec/pki/audit.log` | 附加式稽核日誌 |

不要手動編輯 `index.txt` 後不回讀校驗；需要補登時優先使用 `neca pki index sync`。

## CN 規則

CN 只允許：

~~~text
A-Z a-z 0-9 . _ -
~~~

CN 同時是 VPN 使用者識別、目錄名稱與憑證 Common Name。不要使用空格、斜線、中文或
`..`。範例：`alice`、`team.laptop-01`。

## 常用指令

| 指令 | 作用 |
|---|---|
| `neca cert list` | 列出憑證 |
| `neca cert verify [CN]` | 檢查 p12 演算法、憑證張數、密碼與憑證鏈 |
| `neca cert issue <CN> ...` | 簽發憑證與平台套件 |
| `neca cert reissue <CN> ...` | 封存舊憑證並重新簽發 |
| `neca cert revoke <CN>` | 撤銷並更新 CRL |
| `neca cert unrevoke <CN>` | 取消撤銷並更新 CRL |
| `neca cert trash <CN>` | 移入資源回收筒（可還原） |
| `neca trash list` / `restore <ID>` | 檢視／還原資源回收筒條目 |
| `neca trash purge <ID>` | 背景實體清理（Y/N，預設 N） |
| `neca cert platform add <CN> ...` | 補充平台套件 |
| `neca cert platform set <CN> ...` | 修改平台標註 |
| `neca cert package <CN> ...` | 從現有憑證重建平台套件 |
| `neca cert import <CN> --p12 ... --password-file ...` | 匯入目前 CA 簽發的 p12（過期需確認） |
| `neca cert import <CN> --cert ... --key ...` | 匯入 PEM 憑證與私鑰（可選 `--key-password-file`） |
| `neca cert import-batch <DIR>` | 依 CN 目錄批次匯入 p12 或 cert+key（可部分成功） |
| `neca cert export issued` / `neca cert export revoked` | 匯出 CSV |
| `neca pki index sync` | 為手動放入的憑證補索引與中繼資料 |

所有寫入指令都支援 `--dry-run`；建議先執行：

~~~bash
neca cert issue alice --platforms windows,android,ios --dry-run
~~~

## 簽發

~~~bash
neca cert issue alice \
  --platforms windows,linux,macos,android,ios \
  --days 365 \
  --installers windows \
  --p12-profile compat \
  --zip
~~~

參數：

- `--platforms`：以逗號分隔；預設由設定決定。
- `--days`：有效期限；預設 365。
- `--installers`：`none`、`all` 或平台清單；預設 `none`。
- `--p12-profile`：`compat`（預設，3DES/SHA1）或 `modern`（AES-256/SHA256）。
- `--zip`：簽發後在 `downloads/` 產生帶 TTL 的壓縮檔。

憑證檔案寫入 `clients/alice/`；私鑰只留在伺服器，不進入壓縮檔。

## 檢視與驗證

~~~bash
neca cert list
neca cert list --json
neca cert verify alice
neca cert verify
~~~

`verify` 會檢查 p12 是否只含用戶端憑證、是否使用預期的演算法、密碼長度是否合規，並使用 CA 憑證校驗用戶端憑證鏈。
鏈校驗只驗證信任鏈與有效期限，不檢查 CRL 撤銷狀態；它不能取代用戶端實機匯入測試。

## 重新簽發

~~~bash
neca cert reissue alice \
  --platforms windows,android,ios \
  --days 365 \
  --zip
~~~

重新簽發會封存舊憑證並產生新序號；用戶端必須重新匯入新套件。重新簽發前建議備份
`clients/alice/` 與 `pki/meta/alice.env`。

## 撤銷與取消撤銷

~~~bash
neca cert revoke alice
neca cert unrevoke alice
~~~

兩者都會重建 CRL。預設重新載入模式是 `creds`，透過 `swanctl --load-creds` 熱重載，
不重新啟動 strongSwan；若設定為 `restart`，則依舊行為重新啟動容器。正式環境請先確認：

~~~bash
neca config show
~~~

## 資源回收筒

`cert trash` 只把憑證和相關檔案移入資源回收筒，不實體刪除；Web 面板也僅移入資源回收筒。

~~~bash
neca cert trash alice
neca trash list
neca trash restore <ID>
~~~

資源回收筒預設不限時保存；還原已撤銷的憑證不會自動取消撤銷。實體清理：

~~~bash
neca trash purge <ID>
~~~

`trash purge` 需要一次 Y/N（預設 N）；背景自動化可用 `--yes`。保留期由 `trash.retention` 設定（預設 0＝不限時），可用 `neca trash purge --expired` 清理過期條目。不再需要輸入 `PURGE <CN>`，也不保證 SSD／快照／備份上的安全抹除。清理前先備份：

~~~bash
tar -C /opt/ipsec -czf /root/ipsec-before-trash-$(date +%Y%m%d-%H%M%S).tar.gz \
  pki clients
~~~

## 平台套件

補充平台（不重新簽發）：

~~~bash
neca cert platform add alice --platforms linux,macos
~~~

修改平台標註：

~~~bash
neca cert platform set alice --platforms windows,android
~~~

從現有憑證重建全部套件：

~~~bash
neca cert package alice --platforms windows,linux,macos,android,ios --zip
~~~

## 匯出與索引同步

~~~bash
neca cert export issued --out /root/issued.csv
neca cert export revoked --out /root/revoked.csv
neca pki index sync
~~~

CSV 使用 UTF-8 BOM，方便 Excel 直接開啟。`index sync` 只補索引與中繼資料，
不重新簽發憑證。

## 稽核與 JSON

~~~bash
neca cert list --json
neca cert revoke alice --json
~~~

稽核日誌預設為 `/opt/ipsec/pki/audit.log`，每行包含時間、操作者、動作、CN、
序號與結果。JSON 輸出適合指令碼處理；不要把含密碼的欄位寫進工單或日誌。
