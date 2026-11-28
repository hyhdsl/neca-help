# 🛡️ Security, permissions and backups

## Asset sensitivity

| Asset | Level | Requirement |
|---|---|---|
| `pki/ca.key` | Highest | `0600 root:root`; never in Web; never in public backups |
| Client private keys / p12 | High | Authorized users only; TTL on downloads |
| `web-secrets/session.key` | High | Can forge sessions; treat as a key |
| `web-secrets/password.hash` | High | bcrypt only; no plaintext |
| `pki/ca/index.txt` and CRL | Medium | Integrity matters; CRL may be public |
| Audit log | High | Append-only, restricted access, retained |

## Directory permissions

~~~text
/opt/ipsec/config/neca.json       root:neca-web 0640
/opt/ipsec/pki/ca/                root:neca-web 2755 (setgid)
  index.txt                       root:neca-web 0640
/opt/ipsec/web-secrets/           root:neca-web 0710
  session.key                     root:neca-web 0640
  password.hash                   root:neca-web 0640
  web-audit.log                   root:neca-web 0660
/run/neca/web-helper.sock         root:neca-web 0660
~~~

Web runs as UID/GID 10001, non-root. The CLI runs CA operations as root. The lite
container mounts `docker.sock`, which is equivalent to host root; restrict who can run
`neca`.

`pki/ca` uses setgid and the `neca-web` group so atomic `index.txt` rewrites keep the
group; otherwise the Web process reports `读取索引: ... permission denied`.

## Web boundary

- Mount only required files; never the whole `/opt/ipsec/pki`.
- Do not mount `ca.key`, client private keys or `docker.sock` into Web.
- Writes go through the root helper and its operation whitelist.
- Web-side and helper-side `readOnly` are both enforced.
- Hiding UI controls is not authorization; the server must reject.
- Web listens on `127.0.0.1` by default and publishes no port.

## Network access

Preferred order:

1. Native OpenSSH local forwarding.
2. Native IKEv2 VPN, then access the loopback service.
3. Reverse proxy with TLS and optional mTLS.

Do not make Tailscale/WireGuard a prerequisite. If a reverse proxy is used, restrict
sources, validate the proxy secret and client CN, and strip forged headers.

## Session and key rotation

- Idle timeout defaults to 30m; absolute limit is 24h.
- Changing `readOnly` / `idleTimeout` can be reloaded with SIGHUP; listen/adminCNs/passkey still require a restart.
- Passkey currently has configuration and reserved interfaces only; no login route is registered.
- Rotating `session.key` invalidates all sessions; do it in a maintenance window.
- Generate password hashes with `neca web hash-password`; never put plaintext in `neca.env`.
- If a deployment creates an initial password file, delete or transfer it after first login.

## Recycle bin and secure deletion (implemented)

- The Web delete action only **moves a revoked certificate to the recycle bin**; physical purge is backend-only.
- The recycle bin is retained indefinitely by default; users can configure a retention period. Restoring a revoked certificate does not automatically undo the revocation.
- Backend purge requires a second confirmation.
- **Secure erasure is not promised**: SSDs, filesystem snapshots, backups and object-storage replicas may retain data. The recycle bin is for isolation and mistake prevention only.

## High-risk settings (implemented)

- High-risk security settings remain backend-only by default; the Web settings page displays them read-only.
- Even when the backend enables Web editing, high-risk changes require a Y/N confirmation, re-authentication, CSRF protection and audit logging in both surfaces.
- `adminCNs`, passkey and helper settings should remain backend-only.

## Backup security

Backups contain the CA private key. Encrypt them, restrict access, keep them off the
deployment host, verify restores, and never upload them to public repositories, public
object storage or CI artifacts.

## Audit and monitoring

- Watch issue/revoke/reissue/delete and failures in `audit.log`.
- Watch login failures, CSRF rejects, helper outages and read-only changes.
- Record `index.txt` / CRL hashes around deployments.
- Never log passwords, cookies or private keys.

## Supply chain and license

- Pin image tags; do not use `latest` in deployed environments.
- Pull only from the official repository/GHCR; verify checksums and signer fingerprints with [GPG verification](gpg.md).
- **AGPL-3.0-only** applies; network use and distribution carry source-disclosure duties.
- Code and docs contain AI-assisted content; review before use.

## Pre-exposure checklist

- [ ] `ca.key` is `0600 root:root`
- [ ] Web does not mount `ca.key` / client keys / `docker.sock`
- [ ] No public Web port
- [ ] `readOnly=true` or deliberately enabled writes
- [ ] Helper socket restricted to `root:neca-web`
- [ ] Initial password file handled
- [ ] Backup and restore tested
- [ ] Audit/monitoring enabled
- [ ] Version pinned and rollback image available
