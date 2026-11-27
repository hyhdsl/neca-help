# 🧰 疑難排解

先保存現場：日誌、`neca web status`、`docker compose ps`、稽核與最近變更。
不要在公開管道貼出密碼、Cookie、私鑰或完整設定。

## `neca: command not found`

安裝包裝指令：

~~~bash
install -m 0755 deploy/neca-wrapper.sh /usr/local/bin/neca
which neca
neca version
~~~

若在非標準目錄，檢查 `PATH` 與 wrapper 中的 compose 路徑。

## Docker 不可用 / permission denied

- 確認 Docker Desktop 或 daemon 已啟動；
- 確認執行使用者能存取 `/var/run/docker.sock`；
- lite 容器透過 `docker.sock` 觸發 `swanctl --load-creds`；
- 不要把 `docker.sock` 暴露給 Web 容器。

## Web 啟動失敗：address already in use

`18080` 被占用時，修改 `neca.json`：

~~~json
{ "web": { "listen": "127.0.0.1:18080" } }
~~~

然後重新啟動：

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web restart web
~~~

預設使用 18080；若連接埠已被其他服務占用，改為其他回送連接埠並重新啟動 `neca-web`。

## 修改了 readOnly，但頁面仍顯示舊狀態

只執行 `docker compose up -d` 不會重新載入既有的 Web 程序。送出 SIGHUP：

~~~bash
docker kill --signal=HUP neca-web
neca web status
~~~

`web.listen` 變更仍需重新啟動 web 容器。

## 頁面顯示「helper 不可用」

檢查：

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write ps
neca web status
ls -l /run/neca/web-helper.sock
~~~

常見原因：

- 沒有啟動 `web-write` profile；
- helper 容器未 healthy；
- socket 擁有者／群組不是 `root:neca-web`；
- Web 容器 UID/GID 不是 10001；
- CA 私鑰不存在或不可讀。

## `readonly off` 被拒絕

該指令要求：

1. 以 root 執行；
2. Web 元件已安裝；
3. helper 健康；
4. socket 可存取；
5. `ca.key` 存在。

先看 `neca web status`，再查 helper 日誌。不要用「手動把 JSON 改為 false」繞過
檢查；helper 仍會拒絕，且可能造成狀態不一致。

## 登入失敗 / 被限流

- 確認密碼與部署時設定的初始密碼（或初始密碼檔）一致；
- 檢查系統時間是否準確；
- 等待速率限制視窗結束，避免連續暴力嘗試；
- 忘記密碼時，重新產生 `password.hash` 並重新啟動 web；
- 檢查 `web-secrets` 權限是否為 `0710`、單一檔案 `0640`。

## Web 能啟動但無法讀取憑證

檢查 `neca.json`、`clients`、`downloads` 的群組權限：

~~~bash
chgrp neca-web /opt/ipsec/config/neca.json
chmod 0640 /opt/ipsec/config/neca.json
chgrp -R neca-web /opt/ipsec/clients /opt/ipsec/downloads
chmod -R g+rX /opt/ipsec/clients /opt/ipsec/downloads
~~~

Web 只需要 `pki/ca/index.txt`、`pki/meta`、`clients`、`downloads` 和公開 CRL；
不要為了省事把整個 `pki` 掛給 Web。

## p12 驗證失敗

~~~bash
neca cert verify alice
~~~

常見原因：

- 使用了 `modern` profile，但舊用戶端只支援 `compat`；
- p12 密碼檔與封存不一致；
- p12 中混入了 CA 憑證或憑證鏈數量不對；
- 用戶端手動改過 p12。

重新產生：

~~~bash
neca cert package alice --p12-profile compat --zip
~~~

## 撤銷後 CRL 未生效

確認重新載入模式：

~~~bash
neca config show
docker exec ipsec-vpn swanctl --load-creds
docker logs ipsec-vpn --since 2m 2>&1 | grep -i crl
~~~

預設 `creds` 模式應熱重載；若設定為 `restart`，需要重新啟動容器。不要在未確認的
情況下反覆重新啟動 strongSwan，以免中斷既有連線。

## 下載連結 404 / 410

- 410/404 通常表示已過期、已清理或 ID 不存在；
- 預設 TTL 30m；
- 重新執行 `neca cert package <CN> --zip`；
- 檢查 `neca pki downloads list`。

## 一鍵安裝指令碼沒有產生

預設 `--installers none`。請明確開啟：

~~~bash
neca cert issue alice --platforms windows,linux --installers windows,linux
~~~

Linux／macOS 指令碼未涵蓋所有發行版；先在測試機驗證。

## 升級後 Web 行為異常

- 確認 lite 與 neca-web 使用同一個版本標籤；
- 檢查是否只更新了映像但沒有重建／重新啟動容器；
- 查看 `docker compose logs` 與 `neca web status`；
- 復原時還原舊映像與備份，不要只復原其中一個元件。

## Web 顯示讀取索引 permission denied

如果 Web 回報 `/opt/ipsec/pki/ca/index.txt: open ...: permission denied`，通常是 CA
索引被重寫後群組變成 root，而 Web 程序使用 UID/GID 10001（群組 `neca-web`）：

~~~bash
chgrp neca-web /opt/ipsec/pki/ca /opt/ipsec/pki/ca/index.txt
chmod 2755 /opt/ipsec/pki/ca
chmod 0640 /opt/ipsec/pki/ca/index.txt
docker exec neca-web sh -c 'head -c 1 /opt/ipsec/pki/ca/index.txt >/dev/null && echo OK'
~~~

`pki/ca` 的 setgid 位確保之後 CLI 原子重寫 `index.txt` 時，新檔案仍繼承
`neca-web` 群組。修正後重新整理 Web 頁面即可，不需要重新啟動 Web 或 strongSwan；重新執行
`init-web-secrets.sh` 也會修正該權限。

## 仍然無法解決

收集以下資訊後提交 Issue：

~~~bash
neca version
neca web status
docker compose -f /opt/ipsec/docker-compose.neca.yml ps
~~~

同時說明：版本、部署形態（lite/web/web-write）、重現步驟、已去識別化的錯誤訊息。
