# FAQ

## Is neca a VPN server?

No. neca manages strongSwan IKEv2 user certificates and client packages. The VPN server,
firewall and IKEv2 configuration are outside this project.

## Do I need both lite and neca-web?

No. lite is the default. neca-web is optional and must use the same version tag. Not
installing Web does not affect the CLI.

## Can the Web panel write certificates?

Not by default. `readOnly=true` is the default. You need a healthy helper, then
`neca web readonly off` and an explicit Web restart.

## Can Web delete certificates?

No. Delete is CLI-only and requires `--yes`; revoked certificates also require
`--purge-revoked`. This is intentional risk control.

## Why is the Web port 18080?

The code default is `127.0.0.1:18080`. This production server uses 18080 because 18080
is occupied by other service. Check `web.listen` in `neca.json`.

## Can I expose Web to the Internet?

Not recommended. Use OpenSSH forwarding. If public access is required, use native IKEv2
plus a TLS reverse proxy and consider mTLS; never bind directly to `0.0.0.0`.

## Is multi-user/RBAC supported?

No. The current model is a single administrator. Multi-user support is a later phase.

## Is Passkey supported?

Only the interface and feature flag exist. No login route is registered. Enabling it
requires HTTPS, a stable RP ID and a separate security review.

## Why not SQLite?

The current read-only queries and signed-cookie sessions do not need a database; index
state remains in `index.txt`. SQLite is deferred until concurrency or multi-user needs
are concrete.

## Where is data stored?

Under `/opt/ipsec`: `pki/` (CA/index/meta/audit), `clients/`, `swanctl/x509crl/`,
`downloads/` and `web-secrets/`.

## Does upgrade lose certificates?

No, provided `/opt/ipsec` is preserved and mounted. Still back up first. Do not mix
incompatible lite/neca-web versions.

## How do I back up?

~~~bash
tar -C /opt/ipsec -czf /root/ipsec-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  pki clients swanctl neca.json web-secrets
~~~

Backups contain the CA private key; encrypt and restrict them.

## Can I still use the old bash script?

It remains in `legacy/` for rollback and shadow tests. New deployments should use neca.

## Why are external PRs paused?

The project is still in its development phase and currently accepts maintainer merges
only. Use Issues, or fork under AGPL.

## What license applies?

**AGPL-3.0-only**. Commercial use is allowed, but distribution, modification and network
service require source-disclosure and license preservation. See
[LICENSE](https://github.com/hyhdsl/neca/blob/main/LICENSE).

## Was AI used?

Code, tests and documentation include AI-assisted output. Review and test before use;
see README and NOTICE for the full disclosure.
