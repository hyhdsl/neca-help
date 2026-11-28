# 部署與日常維運

## 日常檢查

~~~bash
neca version
neca cert list
neca web status
docker compose -f /opt/ipsec/docker-compose.neca.yml ps
~~~

關注：

- Web 是否仍為 `readOnly=true`（除非已確認開啟寫入能力）；
- helper 是否健康；
- 憑證到期時間與 CRL 條目數；
- `/opt/ipsec` 磁碟空間；
- 稽核日誌是否持續成長；
- `downloads` 是否積壓。

## Web 日誌

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web logs -f web
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write logs -f web-helper
~~~

日誌可能包含 CN、路徑與錯誤訊息；不要公開貼出含密碼、Cookie 或私鑰的片段。

## 備份

至少備份：

~~~bash
umask 077
tar -C /opt/ipsec -czf - config pki clients downloads swanctl web-secrets \
  docker-compose.neca.yml .env neca.env \
  | gpg --symmetric --cipher-algo AES256 \
      --output /root/ipsec-backup-$(date +%Y%m%d-%H%M%S).tar.gz.gpg
sha256sum /root/ipsec-backup-*.tar.gz.gpg
~~~

GPG 會互動式讀取口令；不要把口令寫進指令、Git 或一般文件。口令應存入密碼管理工具，
備份檔至少保留一份在異機。解密驗證：

~~~bash
gpg --decrypt /root/ipsec-backup-<時間>.tar.gz.gpg | tar -tzf - >/dev/null
~~~

備份包含 CA 私鑰與 Web 工作階段金鑰，必須依最高敏感度保護：

- 存放在加密磁碟或加密封存檔；
- 限制 root 讀取；
- 不要上傳到公開儲存庫、公開網路硬碟或 CI artifact；
- 定期做還原演練。

## 還原

1. 停止 Web／helper；
2. 還原備份中的 `pki`、`clients`、`swanctl`、`neca.json`、`web-secrets`；
3. 校驗 `index.txt` 與 CRL 雜湊；
4. 以唯讀模式啟動 Web；
5. 驗證登入、憑證清單與下載；
6. 再決定是否開啟寫入能力。

## 部署權限檢查

升級、遷移或還原後，在宿主機以 root 執行：

~~~bash
NECA_ROOT=/opt/ipsec bash scripts/check-deploy-permissions.sh
~~~

等值入口：`make check-permissions`。指令碼檢查 CA 私鑰、CA 索引、`pki/meta`、
`clients`、`downloads`、設定、Web secrets、匯出 CSV 與 `.env` 的權限／擁有者；
發現全世界可讀的私鑰、p12、中繼資料或匯出檔案時結束碼為 1。

## 升級與復原

固定版本標籤，不要用 `latest` 當作正式環境相依：

~~~bash
docker pull ghcr.io/hyhdsl/neca:v1.6.0
docker pull ghcr.io/hyhdsl/neca-web:v1.6.0
~~~

升級前備份；升級後驗證：

~~~bash
neca version
neca cert list
neca web status
curl -fsS http://127.0.0.1:18080/healthz
~~~

`healthz` 只用於回送位址健康檢查。復原時還原舊映像標籤與對應備份，不要混用版本。

## PKI 完整性

可在變更前後記錄：

~~~bash
sha256sum /opt/ipsec/pki/ca/index.txt
sha256sum /opt/ipsec/swanctl/x509crl/ca.crl.pem
~~~

在變更前後記錄雜湊，用於確認操作可解釋：

- `index.txt`
- CRL

雜湊變化不一定代表故障；簽發、撤銷、重新簽發都會改變它們。關鍵是變更可解釋、
有稽核、有備份。

## CRL 與 strongSwan

預設 `creds` 模式透過 `swanctl --load-creds` 熱重載，不重新啟動 strongSwan。除非
疑難排解明確需要，不要隨意重新啟動 `ipsec-vpn` 容器；重新啟動會中斷既有連線。

~~~bash
docker exec ipsec-vpn swanctl --list-crls
docker logs ipsec-vpn --since 5m 2>&1 | grep -i crl
~~~

## 清理

~~~bash
neca pki downloads clean --expired
~~~

定期檢查 `/opt/ipsec/downloads`。不要刪除 `pki`、`clients` 或 `swanctl` 中的
檔案來「清出空間」；先確認業務意義。

## 稽核

預設稽核日誌為 `/opt/ipsec/pki/audit.log`。建議：

- 僅附加，不手動編輯；
- 定期封存並限制存取；
- 與系統日誌／監控一起保留；
- 發生憑證異常時，先保存日誌再疑難排解。

## 事故處理順序

1. 停止進一步寫入操作（`neca web readonly on`，必要時停 Web）；
2. 保存日誌、稽核、目前 PKI 雜湊與備份；
3. 判斷是憑證、CRL、容器或權限問題；
4. 在測試環境重現；
5. 恢復服務後補做根本原因與稽核紀錄。

不要在沒有備份時執行 `delete`、`reissue` 或大範圍 `downloads clean --all`。
