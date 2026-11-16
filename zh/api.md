# Web API（实验性）

> **状态：实验性，不是稳定公开 API。** 端点、字段和认证方式可能随版本调整；
> 当前没有 OpenAPI 契约，也不承诺第三方兼容。生产自动化请先在测试环境验证。

## 适用范围

neca Web 面板提供一组管理员 JSON API，用于查询证书、验证 p12、管理下载和执行
issue/revoke/unrevoke/reissue。它使用**登录会话 Cookie + CSRF**，不是 Bearer Token
API；没有多用户/RBAC，也没有公开的 API Key。

**不要把帮助站点域名当作 API 地址。** `https://neca.nekobiglazyer.work` 是帮助文档；
Web 面板通常通过 SSH 隧道访问：

~~~text
http://127.0.0.1:18080
~~~

默认端口是 18080；若修改过 `web.listen`，按实际端口替换。

## 认证流程

API 需要管理员会话：

1. 先请求登录页，保存 Cookie 并读取隐藏的 `csrf_token`；
2. 提交密码和 CSRF token 到 `POST /login`；
3. 后续请求携带会话 Cookie；写请求额外带 `X-CSRF-Token`（值来自
   `neca_csrf` Cookie）。

### 登录并检查能力

~~~bash
BASE=http://127.0.0.1:18080
JAR=$(mktemp)

# 1. 取登录页与 CSRF cookie
LOGIN_HTML=$(curl -sS -c "$JAR" "$BASE/login")
CSRF=$(printf '%s' "$LOGIN_HTML" \
  | sed -n 's/.*name="csrf_token" value="\([^"]*\)".*/\1/p' | head -1)

# 2. 登录；密码从环境变量读取，避免进入 shell 历史
read -r -s -p 'password: ' NECA_WEB_PASSWORD; echo
curl -sS -b "$JAR" -c "$JAR" -X POST "$BASE/login" \
  --data-urlencode "csrf_token=$CSRF" \
  --data-urlencode "password=$NECA_WEB_PASSWORD" \
  -o /dev/null
unset NECA_WEB_PASSWORD

# 3. 登录后的 CSRF（用于写请求）
CSRF=$(awk '$6=="neca_csrf" {print $7}' "$JAR" | tail -1)

# 4. 查询能力
curl -sS -b "$JAR" "$BASE/api/v1/capabilities" | jq .
~~~

`capabilities` 中的关键字段：

~~~json
{
  "readOnly": true,
  "writable": false,
  "helper": { "available": true, "readOnly": true },
  "operations": ["read"]
}
~~~

只有 `readOnly=false`、helper 可用且 helper 自身不为只读时，写操作才可用。

## 端点

所有 `/api/v1/*` 都需要有效会话。写请求还需要 `X-CSRF-Token`。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/v1/capabilities` | 只读/可写能力、helper 状态 |
| GET | `/api/v1/certs?q=&status=` | 证书列表/搜索 |
| GET | `/api/v1/certs/{cn}` | 证书详情 |
| GET | `/api/v1/certs/{cn}/verify` | p12 验证结果 |
| GET | `/api/v1/downloads` | 下载列表 |
| GET | `/api/v1/downloads/{id}` | 下载文件 |
| POST | `/api/v1/certs` | 签发证书 |
| POST | `/api/v1/certs/{cn}/revoke` | 吊销 |
| POST | `/api/v1/certs/{cn}/unrevoke` | 恢复吊销 |
| POST | `/api/v1/certs/{cn}/reissue` | 重签 |

不支持 delete、package 或设置编辑。

## 示例：签发

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

`requestId` 也可以放在 `X-Request-ID` 请求头。相同 ID 的重复请求会命中 helper 的
幂等缓存；缓存默认落盘到 `/run/neca/web-helper-cache.json`（0600，TTL 24h），
可由 `NECA_WEB_HELPER_CACHE_FILE` 覆盖，重启后仍保留（文件损坏时从空缓存重建）。

## 示例：吊销

~~~bash
curl -sS -b "$JAR" \
  -H "X-CSRF-Token: $CSRF" \
  -H 'Content-Type: application/json' \
  -d '{"requestId":"demo-revoke-001"}' \
  "$BASE/api/v1/certs/alice/revoke" | jq .
~~~

## 错误与限制

错误响应：

~~~json
{"error":{"code":"read_only","message":"Web 面板当前为只读模式"}}
~~~

常见状态码：

| 状态码 | 含义 |
|---|---|
| 400 | JSON/参数非法 |
| 401 | 未登录或会话过期 |
| 403 | CSRF 失败、只读或权限不足 |
| 409 | 前置条件冲突、requestId 冲突 |
| 503 | helper 不可用 |

- 会话默认空闲 30 分钟；绝对上限 24 小时。
- 登录失败有限流；API 不提供绕过登录的 Token。
- 没有 CORS 支持；脚本应在同源浏览器上下文或通过 SSH 隧道使用。
- 不要通过公网直接暴露 Web 端口；优先 OpenSSH 本地转发。
- API 是管理员能力，等同高权限操作，需纳入审计和变更管理。
- 当前为实验性接口；如需长期集成，请先与维护者确认契约。

## 相关

- [Web 面板](web.md)
- [安全、权限与备份](security.md)
- [故障排查](troubleshooting.md)
