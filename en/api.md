# Web API (experimental)

> **Status: experimental, not a stable public API.** Endpoints, fields and
> authentication may change; there is no OpenAPI contract and no third-party
> compatibility promise. Validate automation in a test environment first.

## Scope

The Web panel exposes an administrator JSON API for listing certificates, verifying
p12 files, managing downloads and issuing/revoking/reissuing certificates. It uses a
**session cookie + CSRF**, not a Bearer token. There is no multi-user/RBAC and no public
API key.

**Do not use the help-site domain as the API base.** `https://neca.nekobiglazyer.work` is
documentation. The Web panel is normally reached through an SSH tunnel:

~~~text
http://127.0.0.1:18080
~~~

The code default is 18080; this production server uses 18080.

## Authentication

1. Request the login page, keep cookies and extract the hidden `csrf_token`.
2. Submit the password and CSRF token to `POST /login`.
3. Send the session cookie on later requests; writes also require `X-CSRF-Token` from
   the `neca_csrf` cookie.

### Log in and inspect capabilities

~~~bash
BASE=http://127.0.0.1:18080
JAR=$(mktemp)

LOGIN_HTML=$(curl -sS -c "$JAR" "$BASE/login")
CSRF=$(printf '%s' "$LOGIN_HTML" \
  | sed -n 's/.*name="csrf_token" value="\([^"]*\)".*/\1/p' | head -1)

read -r -s -p 'password: ' NECA_WEB_PASSWORD; echo
curl -sS -b "$JAR" -c "$JAR" -X POST "$BASE/login" \
  --data-urlencode "csrf_token=$CSRF" \
  --data-urlencode "password=$NECA_WEB_PASSWORD" \
  -o /dev/null
unset NECA_WEB_PASSWORD

CSRF=$(awk '$6=="neca_csrf" {print $7}' "$JAR" | tail -1)
curl -sS -b "$JAR" "$BASE/api/v1/capabilities" | jq .
~~~

Key `capabilities` fields:

~~~json
{
  "readOnly": true,
  "writable": false,
  "helper": { "available": true, "readOnly": true },
  "operations": ["read"]
}
~~~

Writes work only when `readOnly=false`, the helper is available and the helper is not
itself read-only.

## Endpoints

All `/api/v1/*` endpoints require a valid session. Writes also require `X-CSRF-Token`.

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/capabilities` | Read/write capability and helper state |
| GET | `/api/v1/certs?q=&status=` | List/search certificates |
| GET | `/api/v1/certs/{cn}` | Certificate detail |
| GET | `/api/v1/certs/{cn}/verify` | p12 verification |
| GET | `/api/v1/downloads` | Download list |
| GET | `/api/v1/downloads/{id}` | Download a file |
| POST | `/api/v1/certs` | Issue a certificate |
| POST | `/api/v1/certs/{cn}/revoke` | Revoke |
| POST | `/api/v1/certs/{cn}/unrevoke` | Unrevoke |
| POST | `/api/v1/certs/{cn}/reissue` | Reissue |

Delete, package and settings editing are not exposed.

## Issue example

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

`requestId` may also be sent as `X-Request-ID`. Repeated requests with the same ID use
the helper's idempotency cache. The cache is persisted to
`/run/neca/web-helper-cache.json` (0600, 24h TTL), overridable with
`NECA_WEB_HELPER_CACHE_FILE`, and survives helper restarts; corrupt files are ignored
and rebuilt empty.

## Revoke example

~~~bash
curl -sS -b "$JAR" \
  -H "X-CSRF-Token: $CSRF" \
  -H 'Content-Type: application/json' \
  -d '{"requestId":"demo-revoke-001"}' \
  "$BASE/api/v1/certs/alice/revoke" | jq .
~~~

## Errors and limits

~~~json
{"error":{"code":"read_only","message":"Web panel is read-only"}}
~~~

| Status | Meaning |
|---|---|
| 400 | Invalid JSON or parameters |
| 401 | Missing/expired session |
| 403 | CSRF failure, read-only or insufficient permission |
| 409 | Precondition or requestId conflict |
| 503 | Helper unavailable |

- Idle session timeout defaults to 30 minutes; absolute limit is 24 hours.
- Login attempts are rate-limited; there is no token that bypasses login.
- No CORS support; run scripts in a same-origin context or through an SSH tunnel.
- Never expose the Web port publicly; prefer OpenSSH forwarding.
- The API is administrator-level and must be covered by audit and change control.
- This is experimental; confirm the contract with the maintainer before long-term use.

## Related

- [Web panel](web.md)
- [Security, permissions and backups](security.md)
- [FAQ](faq.md)
