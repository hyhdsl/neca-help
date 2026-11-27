# 📦 安裝、升級與復原

> 本頁只包含兩種安裝路徑：**Docker Compose（方式 A）** 和 **Linux 二進位檔（方式 B）**。
> 請選擇一種路徑，不要混用。下面的 A1–A6／B1–B4 標示了每條路徑的起訖位置；
> 升級、復原與解除安裝屬於安裝後的共用流程（C1–C3）。

## 0. 總流程邊界

| 路徑 | 從哪裡開始 | 到哪裡結束 | 需要什麼 |
|---|---|---|---|
| 方式 A：Docker Compose | **A1 準備目錄與檔案** | **A6 驗證 Docker 安裝** | Docker、Compose、儲存庫中的 deploy 檔案 |
| 方式 B：Linux 二進位檔 | **B1 下載與校驗** | **B4 檢查設定／PKI 前置** | Linux amd64/arm64、OpenSSL；僅 CLI |
| 共用流程 | **C1 升級** | **C3 解除安裝** | 已完成的 A 或 B |

**整頁起點**：先讀本節並選擇 A 或 B。

**整頁終點**：完成 A6 或 B4。
**不要混用**：Docker 方式從 A1 到 A6；二進位方式從 B1 到 B4。二進位方式不包含 Web／helper。

## 方式 A：Docker Compose 安裝（A1–A6）

**路徑範圍**：A1–A6；A5 是選用的 Web 寫入能力。完成 A6 即完成 Docker 安裝。

### A1. 準備目錄與檔案

在儲存庫根目錄（或已取得 deploy 檔案）執行：

~~~bash
install -d -m 0755 /opt/ipsec
install -m 0600 deploy/neca.env.example /opt/ipsec/neca.env
install -m 0644 deploy/docker-compose.neca.yml /opt/ipsec/docker-compose.neca.yml
install -m 0755 deploy/init-web-secrets.sh /opt/ipsec/init-web-secrets.sh
install -d -m 0750 -o root -g neca-web /opt/ipsec/config
install -m 0600 deploy/neca.json.example /opt/ipsec/config/neca.json
~~~

`neca.env` 只保留 bootstrap：

~~~bash
NECA_CONFIG=/opt/ipsec/config/neca.json
~~~

真實的 `SERVER_ID`、`ORG`、`CA_CN` 寫入 `/opt/ipsec/config/neca.json`，不要提交到儲存庫。
設定目錄單獨掛載給 Web，SIGHUP 才能看到原子替換後的新檔案。

### A2. 安裝指令包裝

~~~bash
install -m 0755 deploy/neca-wrapper.sh /usr/local/bin/neca
neca version
neca config validate
~~~

包裝指令會在非互動環境自動使用 `docker compose run -T`，並把 `NECA_LANG` 傳入容器。

### A3. 啟動 lite

~~~bash
cd /opt/ipsec
docker compose -f docker-compose.neca.yml run --rm neca cert list
~~~

或先拉取固定版本映像：

~~~bash
docker pull ghcr.io/hyhdsl/neca:v1.2.0
~~~

### A4. 選用：啟動唯讀 Web

~~~bash
cd /opt/ipsec
NECA_IMAGE=ghcr.io/hyhdsl/neca:v1.2.0 ./init-web-secrets.sh
docker compose -f docker-compose.neca.yml --profile web up -d web
~~~

指令碼會以互動方式提示設定管理員密碼（不回顯、不寫入 shell 歷史），並建立
`/opt/ipsec/web-secrets/`。請立即把密碼保存到密碼管理工具；如果部署流程另外產生了
初始密碼檔，首次登入後刪除它。需要重設時重新產生 `password.hash`。

如果 `/opt/ipsec/pki/ca` 已存在，指令碼還會設定群組繼承並修正 `index.txt` 為
`root:neca-web 0640`，避免 Web 讀取索引時回報 permission denied。

預設連接埠是 `127.0.0.1:18080`。修改 `web.listen` 後必須重新啟動 `neca-web`。

### A5. 選用：開啟 Web 寫入能力

~~~bash
cd /opt/ipsec
docker compose -f docker-compose.neca.yml --profile web --profile web-write up -d web web-helper
neca web status
neca web readonly off
~~~

寫入能力不是預設值。`readonly off` 需要 helper 可用、socket 權限正確且 CA 私鑰存在。
詳見 [Web 面板](web.md) 與 [安全、權限與備份](security.md)。

### A6. 驗證 Docker 安裝（方式 A 結束）

完成 A6 後，Docker 安裝路徑結束：

~~~bash
neca version
neca config validate
neca web status
neca doctor
~~~

選用 Web 已啟動時，透過 OpenSSH 本機轉送開啟面板，並確認登入頁、憑證清單與下載入口。

## 方式 B：Linux 二進位檔安裝（B1–B4）

**路徑範圍**：B1–B4。只安裝 lite CLI，不含 Web／helper；完整的憑證操作仍需設定、PKI 和 OpenSSL。

### B1. 下載與校驗二進位檔

~~~bash
VERSION=v1.2.0
curl -fLO "https://github.com/hyhdsl/neca/releases/download/${VERSION}/neca-linux-amd64"
curl -fLO "https://github.com/hyhdsl/neca/releases/download/${VERSION}/checksums.txt"
grep " neca-linux-amd64$" checksums.txt > /tmp/neca-checksum
sha256sum -c /tmp/neca-checksum
~~~

arm64 主機請把 `neca-linux-amd64` 換成 `neca-linux-arm64`。
本路徑已在 Debian bookworm 容器中驗證下載、校驗、安裝、`--version` 與 `help`；
尚未在正式主機上直接取代 Docker 部署。

### B2. 安裝到系統路徑

~~~bash
sudo install -m 0755 neca-linux-amd64 /usr/local/bin/neca
~~~

確認 `openssl` 可用。二進位方式不提供 Web 面板；需要 Web／helper 時請回到方式 A。

### B3. 驗證二進位安裝

~~~bash
neca --version
neca help
neca --lang en-US help
~~~

### B4. 檢查設定與 PKI 前置（方式 B 結束）

二進位檔只負責 CLI 本身；在簽發／撤銷前，請依 [安全、權限與備份](security.md) 準備設定、
PKI 目錄和 `openssl`。可先執行：

~~~bash
neca config validate
neca doctor
~~~

完成 B4 後，二進位安裝路徑結束。

## 共用流程：升級、復原與解除安裝（C1–C3）

**路徑範圍**：C1–C3；從升級開始，到解除安裝結束。以下流程適用於已完成的 A 或 B。

### C1. 升級

升級前備份：

~~~bash
umask 077
tar -C /opt/ipsec -czf - config pki clients downloads swanctl web-secrets \
  docker-compose.neca.yml .env neca.env \
  | gpg --symmetric --cipher-algo AES256 \
      --output /root/ipsec-backup-$(date +%Y%m%d-%H%M%S).tar.gz.gpg
~~~

GPG 口令不要寫入指令、Git 或一般文件；還原與更多細節見
[部署與日常維運](operations.md#還原)。

Docker 方式：更新 compose 中的映像標籤後執行：

~~~bash
docker pull ghcr.io/hyhdsl/neca:v1.2.0
docker pull ghcr.io/hyhdsl/neca-web:v1.2.0
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web up -d web
~~~

二進位方式：重新執行 B1–B3，取代 `/usr/local/bin/neca`。
修改 `readOnly`／`idleTimeout` 後送出 SIGHUP：

~~~bash
docker kill --signal=HUP neca-web
~~~

只執行 `up -d` 不會讓已執行的 Web 程序重新讀取設定；`web.listen` 等監聽類設定仍需重新啟動。

### C2. 復原

1. 停止 Web／helper；
2. 還原備份的 `pki`、`clients`、`swanctl`、`neca.json`、`web-secrets`；
3. 把 compose 映像標籤切回上一個版本，或還原舊二進位檔；
4. 重新啟動對應容器或執行 `neca --version` 驗證。

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web stop web web-helper
~~~

lite 與 neca-web 必須使用同一版本；不要混用舊 CLI 與新 Web。

### C3. 解除安裝（共用流程結束）

停止並刪除容器不會刪除 `/opt/ipsec` 資料。確認不再需要 Web 時：

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write down
~~~

二進位安裝的 CLI 可手動刪除 `/usr/local/bin/neca`；不要執行會刪除資料目錄的指令，解除安裝前先備份。

## 深入閱讀

- 部署、升級與復原以本安裝頁為準；內部設計文件由維護者本機保存，不對外公開。
- [安全、權限與備份](security.md)
