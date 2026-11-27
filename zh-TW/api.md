# Web API（實驗性）

> **狀態：實驗性，不是穩定的公開 API。** 端點、欄位和驗證方式可能隨版本調整；
> 介面為實驗性，不承諾第三方相容；自動化前請先在測試環境驗證。

## 適用範圍

neca Web 面板提供一組管理員 JSON API，用於查詢憑證、驗證 p12、管理下載，以及執行
issue/revoke/unrevoke/reissue。它使用**登入工作階段 Cookie + CSRF**，不是 Bearer Token
API；沒有多使用者／RBAC，也沒有公開的 API Key。

**不要把說明網站的網域當成 API 位址。** `https://neca.nekobiglazyer.work` 是說明文件；
Web 面板通常透過 SSH 通道存取：

~~~text
http://127.0.0.1:18080
~~~

預設連接埠是 18080；若修改過 `web.listen`，請按實際連接埠替換。

## 驗證流程

API 需要管理員工作階段：

1. 先請求登入頁，保存 Cookie 並讀取隱藏的 `csrf_token`；
2. 提交密碼與 CSRF token 到 `POST /login`；
3. 後續請求攜帶工作階段 Cookie；寫入請求額外帶 `X-CSRF-Token`（值來自
   `neca_csrf` Cookie）。

### 登入並檢查能力

~~~bash
BASE=http://127.0.0.1:18080
JAR=$(mktemp)

# 1. 取得登入頁與 CSRF cookie
LOGIN_HTML=$(curl -sS -c "$JAR" "$BASE/login")
CSRF=$(printf '%s' "$LOGIN_HTML" \
  | sed -n 's/.*name="csrf_token" value="\([^"]*\)".*/\1/p' | head -1)

# 2. 登入；密碼從環境變數讀取，避免進入 shell 歷史
read -r -s -p 'password: ' NECA_WEB_PASSWORD; echo
curl -sS -b "$JAR" -c "$JAR" -X POST "$BASE/login" \
  --data-urlencode "csrf_token=$CSRF" \
  --data-urlencode "password=$NECA_WEB_PASSWORD" \
  -o /dev/null
unset NECA_WEB_PASSWORD

# 3. 登入後的 CSRF（用於寫入請求）
CSRF=$(awk '$6=="neca_csrf" {print $7}' "$JAR" | tail -1)

# 4. 查詢能力
curl -sS -b "$JAR" "$BASE/api/v1/capabilities" | jq .
~~~

`capabilities` 中的關鍵欄位：

~~~json
{
  "readOnly": true,
  "writable": false,
  "helper": { "available": true, "readOnly": true },
  "operations": ["read"]
}
~~~

只有 `readOnly=false`、helper 可用且 helper 本身不是唯讀時，寫入操作才可用。

## 端點

所有 `/api/v1/*` 都需要有效工作階段。寫入請求還需要 `X-CSRF-Token`。

| 方法 | 路徑 | 說明 |
|---|---|---|
| GET | `/api/v1/capabilities` | 唯讀／可寫能力、helper 狀態 |
| GET | `/api/v1/certs?q=&status=&expiry=` | 憑證清單／搜尋；`expiry=30\|60\|90` 只看對應到期區間 |
| GET | `/api/v1/certs/{cn}` | 憑證詳細資料 |
| GET | `/api/v1/certs/{cn}/verify` | p12 驗證結果 |
| GET | `/api/v1/downloads` | 下載清單 |
| GET | `/api/v1/downloads/{id}` | 下載檔案 |
| POST | `/api/v1/certs` | 簽發憑證 |
| POST | `/api/v1/certs/{cn}/revoke` | 撤銷 |
| POST | `/api/v1/certs/{cn}/unrevoke` | 取消撤銷 |
| POST | `/api/v1/certs/{cn}/reissue` | 重新簽發 |

`expiry` 只傳回未來區間內到期的 `valid` 憑證；非法值會被忽略。

不支援 delete、package 或設定編輯。

## 範例：簽發

~~~bash
curl -sS -b "$JAR" \
  -H "X-CSRF-Token: $CSRF" \
  -H 'Content-Type: application/json' \
  -d '{
    "requestId": "demo-issue-001",
    "cn": "alice",
    "days": 365,
    "platforms": ["windows", "android"],
    "p12Profile": "compat"
  }' \
  "$BASE/api/v1/certs" | jq .
~~~

`requestId` 也可以放在 `X-Request-ID` 要求標頭。相同 ID 的重複請求會命中 helper 的
冪等快取；快取預設寫入磁碟 `/run/neca/web-helper-cache.json`（0600，TTL 24h），
可由 `NECA_WEB_HELPER_CACHE_FILE` 覆寫，重新啟動後仍保留（檔案損毀時從空快取重建）。

## 範例：撤銷

~~~bash
curl -sS -b "$JAR" \
  -H "X-CSRF-Token: $CSRF" \
  -H 'Content-Type: application/json' \
  -d '{"requestId":"demo-revoke-001"}' \
  "$BASE/api/v1/certs/alice/revoke" | jq .
~~~

## 錯誤與限制

錯誤回應：

~~~json
{"error":{"code":"read_only","message":"Web 面板目前為唯讀模式"}}
~~~

常見狀態碼：

| 狀態碼 | 意義 |
|---|---|
| 400 | JSON／參數不合法 |
| 401 | 未登入或工作階段過期 |
| 403 | CSRF 失敗、唯讀或權限不足 |
| 409 | 前置條件衝突、requestId 衝突 |
| 503 | helper 不可用 |

- 工作階段預設閒置 30 分鐘；絕對上限 24 小時。
- 登入失敗有速率限制；API 不提供繞過登入的 Token。
- 不支援 CORS；指令碼應在同源瀏覽器內容中或透過 SSH 通道使用。
- 不要透過公網直接暴露 Web 連接埠；優先使用 OpenSSH 本機轉送。
- API 是管理員能力，等同高權限操作，需納入稽核與變更管理。
- 目前為實驗性介面；如需長期整合，請先與維護者確認介面約定。

## 相關

- [Web 面板](web.md)
- [安全、權限與備份](security.md)
- [疑難排解](troubleshooting.md)
