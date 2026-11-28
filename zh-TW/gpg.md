# GPG 驗證

> 適用於 neca v1.5.1 及之後的正式 Release。發布簽章分兩條獨立鏈：Release 資產 `checksums.txt` 與 Git tag。兩條鏈使用不同金鑰，不能互相取代。

## 1. 先理解兩條簽章鏈

| 對象 | 簽章金鑰 | 指紋 | 用途 |
|---|---|---|---|
| Release 資產 `checksums.txt` | neca release signing (2026) | `E514EB7C153AA24A3E4241F846CC03EE21F66795` | 校驗全部發布二進位檔 |
| Git tag | 既有個人 neca 金鑰 | `4DE8259A9C3E832137506D3C9398D6D10FD5A903` | 校驗 tag 指向的提交 |

從 v1.5.1 起，Release 資產由專用 Ed25519 發布金鑰簽章；v1.5.0 及更早版本可能使用既有 neca 金鑰。Git tag 仍使用既有個人金鑰。

驗證時不要只看 gpg 輸出的 `Good signature`，還必須核對實際簽章者指紋。

## 2. 下載驗證材料

以目前穩定版 v1.6.0 為例；其他版本請替換版本號：

~~~bash
gh release download v1.6.0 --repo hyhdsl/neca \
  --pattern checksums.txt \
  --pattern checksums.txt.asc \
  --pattern neca-gpg-key.asc
~~~

Release 資產中的 `neca-gpg-key.asc` 與儲存庫中的 `keys/neca-release-signing.asc` 都是公鑰入口。建議同時透過兩個管道核對指紋。

## 3. 匯入公鑰並核對指紋

~~~bash
gpg --import neca-gpg-key.asc
gpg --fingerprint E514EB7C153AA24A3E4241F846CC03EE21F66795
~~~

預期看到：

~~~text
E514 EB7C 153A A24A 3E42 41F8 46CC 03EE 21F6 6795
neca release signing (2026) <release@neca.invalid>
~~~

如果指紋不一致，立即停止，不要執行下載的二進位檔。

## 4. 驗證 Release 資產

~~~bash
gpg --verify checksums.txt.asc checksums.txt
sha256sum -c checksums.txt
~~~

只有同時符合以下條件才算通過：

1. `checksums.txt.asc` 是有效簽章；
2. 簽章者指紋是 `E514EB7C153AA24A3E4241F846CC03EE21F66795`；
3. `sha256sum -c` 全部通過。

## 5. 驗證 Git tag

Git tag 使用既有個人 neca 金鑰，與 Release 資產簽章是兩條驗證鏈：

~~~bash
git fetch --tags
git tag -v v1.6.0
~~~

預期簽章者指紋：

~~~text
4DE8259A9C3E832137506D3C9398D6D10FD5A903
~~~

只驗證 Release 資產、不驗證 tag，不能說明建置來源與 tag 指向的提交完全一致。

## 6. 常見問題

### gpg 顯示 Good signature，但指紋不同

不要繼續。`Good signature` 只表示簽章與某個已匯入公鑰相符，不表示該公鑰就是 neca 發布金鑰。

### 無法匯入公鑰

確認下載的是 `neca-gpg-key.asc`，不是簽章檔。Windows 可使用 Kleopatra 匯入，也可使用 GnuPG 命令列；不同安裝方式的 `gpg` 路徑可能不同。

### sha256sum 失敗

刪除已下載二進位檔並重新下載，不要嘗試修復或忽略校驗失敗。

### 需要驗證舊版本

v1.5.0 及更早版本的資產簽章可能使用既有 neca 金鑰，請以對應 Release 的公鑰與說明為準。歷史 Release 不重新簽章。

## 7. 公鑰來源

- 儲存庫：`keys/neca-release-signing.asc`
- Release：`neca-gpg-key.asc`
- 公鑰指紋：`E514EB7C153AA24A3E4241F846CC03EE21F66795`

公鑰沒有 CA 憑證鏈；可信度來自儲存庫、Release 與指紋的跨管道核對。

## 相關頁面

- [安裝、升級與復原](install.md)
- [安全、權限與備份](security.md)
- [疑難排解](troubleshooting.md)
