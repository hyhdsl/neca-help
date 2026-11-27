# 平台套件與用戶端安裝

## 平台矩陣

| 平台 | 預設產物 | 選用一鍵指令碼 |
|---|---|---|
| Windows | `p12`、`pass`、`ca.crt`、`使用说明.txt` | `install.ps1` / `uninstall.ps1` / `connect.ps1` + `.cmd` |
| Linux | 同上 | `install.sh` / `uninstall.sh` |
| macOS | 同上 + `<CN>.mobileconfig` | `install.command` / `uninstall.command` |
| Android | `p12`、`pass`、`ca.crt`、說明 | — |
| iOS | `<CN>.mobileconfig`（CA + 身分 + IKEv2） | — |

平台名稱：`windows`、`linux`、`macos`、`android`、`ios`；相容別名包括
`win`、`mac`、`darwin`、`osx`、`and`。

## 產生套件

簽發時直接產生：

~~~bash
neca cert issue alice --platforms windows,linux,macos,android,ios --zip
~~~

從既有憑證重建：

~~~bash
neca cert package alice --platforms windows,android,ios --zip
~~~

預設不產生一鍵安裝指令碼；需要時明確開啟：

~~~bash
neca cert issue alice --platforms windows,linux --installers windows,linux
~~~

`--installers all` 會為支援指令碼的平台產生指令碼，但 Linux／macOS 指令碼仍應在目標
發行版／版本上人工驗證。

## 壓縮檔與 TTL

`--zip` 會在 `/opt/ipsec/downloads/` 產生：

~~~text
<CN>-<YYYYmmdd-HHmmss>-<隨機>.zip
<同名>.json
~~~

預設 30 分鐘過期；Web 下載必須登入，並依 sidecar 中繼資料校驗，過期為 410/404。
查看與清理：

~~~bash
neca pki downloads list
neca pki downloads clean --expired
neca pki downloads clean --all
~~~

如果用戶端拿到的是過期連結，重新執行 `neca cert package <CN> --zip`。

## 套件內不應出現

- `<CN>.key`、`ca.key`、`index.txt`、`audit.log`；
- 任何絕對路徑、`..` 路徑或伺服器私鑰；
- 真實伺服器識別（範例設定使用預留值）。

壓縮檔包含 p12 與密碼檔，請視為敏感憑證管理；不要透過公開 IM 群組或未加密郵件傳送。

## 用戶端匯入要點

### Windows

1. 匯入 `ca.crt` 到「受信任的根憑證授權單位」；
2. 按兩下 `p12` 匯入「個人」憑證，輸入 `pass` 中的密碼；
3. 依組織要求設定 IKEv2 連線，或執行 `connect.ps1`（若已產生）。

### Linux / macOS

- Linux 可使用 strongSwan 的 `swanctl` 設定；指令碼支援 `NECA_DRY_RUN=1` 預覽。
- macOS 可匯入 p12 到登入鑰匙圈，並安裝 `.mobileconfig`。
- 一鍵指令碼預設關閉，開啟前先在測試機驗證。

### Android / iOS

- Android 透過系統「安裝憑證」匯入 p12；不同廠商選單不同。
- iOS 安裝 `.mobileconfig` 後，到「設定 → 一般 → VPN 與裝置管理」確認描述檔。
- 若用戶端提示密碼錯誤，先核對 `pass` 檔案與鍵盤輸入，不要反覆重新簽發。

## 常見問題

- **沒有產生安裝指令碼**：加 `--installers`；這是刻意的預設安全行為。
- **Linux／macOS 指令碼未驗證**：目標系統差異大，先 dry-run，再在正式環境推廣。
- **p12 在舊版 Android 打不開**：使用預設 `compat` profile；`modern` 不保證舊裝置相容。
- **壓縮檔下載 410/404**：已過期或已被清理；重新產生。
