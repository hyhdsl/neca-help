# Web panel

## Purpose

The Web panel is optional and read-only by default. It provides certificate review,
p12 verification, protected downloads and session management. It does not replace
strongSwan and is not exposed directly to the Internet.

| Component | Role | Access |
|---|---|---|
| `necad` | Web server | Non-root; read-only mounts for config, index, meta, clients and downloads |
| `web-helper` | Write proxy | root; accepts only issue/revoke/unrevoke/reissue |
| lite CLI | Administration | root; `neca web ...` commands |

The Web container has no `ca.key`, client private keys or `docker.sock`. Only the helper
mounts those assets.

## Enable read-only Web

### Configuration

In `/opt/ipsec/config/neca.json`:

~~~json
{
  "web": {
    "enabled": true,
    "listen": "127.0.0.1:18080",
    "readOnly": true,
    "idleTimeout": "30m",
    "adminCNs": [],
    "passkey": { "enabled": false, "rpId": "", "rpName": "neca Admin" }
  }
}
~~~

- `idleTimeout` accepts 5m–24h; the absolute session limit is 24h.
- `readOnly` defaults to true.

### Initialize secrets

~~~bash
cd /opt/ipsec
NECA_IMAGE=ghcr.io/hyhdsl/neca:v0.4.1 ./init-web-secrets.sh
~~~

The script asks for the administrator password interactively, creates `web-secrets/`,
`session.key`, `password.hash` and `web-audit.log`, and ensures the `neca-web` group has
GID 10001. The password is not echoed and is not written to `neca.env`.

### Start

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web up -d web
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web logs -f web
~~~

### Access

Use an OpenSSH local forward:

~~~bash
ssh -L 18080:127.0.0.1:18080 root@<SERVER>
# Open http://127.0.0.1:18080
~~~

Do not bind to `0.0.0.0` or expose the port publicly.

## Authentication and sessions

- The administrator password is stored as a bcrypt hash.
- The session cookie is HMAC-signed, HttpOnly and SameSite=Lax; it becomes Secure over HTTPS.
- Idle timeout defaults to 30 minutes; absolute limit is 24 hours.
- Login failures are rate-limited; writes require CSRF.
- Logout revokes the session; restarting necad invalidates in-memory sessions.

## Enable writes

~~~bash
cd /opt/ipsec
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write up -d web web-helper
neca web status
neca web readonly off
docker kill --signal=HUP neca-web
~~~

Disable writes:

~~~bash
neca web readonly on
docker kill --signal=HUP neca-web
~~~

**Important:** send SIGHUP after changing `readOnly` / `idleTimeout`:
`docker kill --signal=HUP neca-web`. `web.listen`, `adminCNs` and passkey changes
still require a Web restart.

Web writes support issue/revoke/unrevoke/reissue only. They do not support delete,
package or settings editing. The helper checks `readOnly` independently; hiding buttons
is not a security boundary.

## CLI

~~~bash
neca web status
neca web readonly status
neca web readonly on
neca web readonly off
neca web idle-timeout status
neca web idle-timeout 30m
neca web hash-password
~~~

`readonly off` requires root, a healthy helper, correct socket permissions, `ca.key` and
a successful atomic config write. It must not leave a half-enabled state.

## Reverse proxy and mTLS (optional)

If `web.adminCNs` is configured, set `NECA_WEB_PROXY_SECRET` and have the reverse proxy
forward:

~~~text
X-Neca-Proxy-Secret: <shared secret>
X-Neca-Client-CN:   <verified administrator certificate CN>
~~~

necad accepts only loopback proxy requests and rejects forged headers. Set
`NECA_WEB_SECURE_COOKIES=1` after TLS termination. This production server currently uses
OpenSSH forwarding and has no public TLS/mTLS endpoint.

## Language and theme

- zh-CN/en-US switching is available via the `neca_lang` cookie.
- The default language follows `neca.json` `language`.
- Light/dark theme is stored in browser localStorage.
- Language and theme do not change server permissions or read-only state.

## Security boundaries

- Web runs as UID/GID 10001, read-only root filesystem, `no-new-privileges` and
  `cap_drop: ALL`.
- It does not mount `ca.key`, client private keys or `docker.sock`.
- The helper is the only root write boundary; protect its socket.
- The helper idempotency cache persists at `/run/neca/web-helper-cache.json`
  (0600, 24h TTL); corrupt files are ignored and rebuilt without blocking writes.
- No public port; prefer SSH tunneling.
- Passkey is only an interface/feature flag and has no login route yet.
- SIGHUP hot reload is implemented for readOnly/idleTimeout; web.listen/adminCNs/passkey changes require a restart.

## Related

- [Troubleshooting (Chinese)](../zh/troubleshooting.md)
- [Security, permissions and backups](security.md)
- [Full deployment guide](https://github.com/hyhdsl/neca/blob/main/docs/deploy.md)
