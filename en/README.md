# 🏠 neca Help Center

> Applies to **neca v1.0.1**. **Neko's Efficient Certificate Administrator (neca)** manages strongSwan IKEv2 user certificates and client packages; it is not the VPN server itself.

> **AI disclosure:** all code authoring, debugging, testing, CI/build/deploy scripting, documentation and this help site were produced by an AI agent. The human maintainer defines requirements, reviews changes, performs functional acceptance and handles production operations. See [AI-DISCLOSURE.md](https://github.com/hyhdsl/neca/blob/main/AI-DISCLOSURE.md) for scope and limits. AI output may contain errors; review and test before production use.

## 🎯 When to use neca

- A single or small number of strongSwan IKEv2 gateways maintained from the CLI.
- A deployment that wants a read-only web panel by default, clear privilege boundaries, audit logs and Docker packaging.
- A setup that does **not** need multi-tenancy, a database or a complex HA cluster.
- **Not suitable for**: large-scale or multi-tenant PKI, unattended automatic issuance, or direct public exposure without an access design.

## 🚀 Start here

- First run: [Quick start](getting-started.md); common commands: [Command cheat sheet](cheatsheet.md)
- Full subcommand reference: [Command reference](commands.md)
- Server install/upgrade: [Install, upgrade and rollback](install.md)
- Certificates: [Certificate management](certificates.md)
- Client packages: [Platform packages](packages.md)
- Browser panel: [Web panel](web.md)
- Daily operations: [Deployment and operations](operations.md)
- Hardening and backups: [Security, permissions and backups](security.md)
- Problems: [Troubleshooting](troubleshooting.md) and [FAQ](faq.md)
- Automation: [Web API (experimental)](api.md)
- Development: [Development, license and feedback](development.md)

## English coverage

English help pages cover the same topics as the Chinese documentation. Both are maintained manually; the Chinese documentation remains authoritative for behavior and release notes.

## 🧩 Runtime forms

| Form | Contents | Typical use |
|---|---|---|
| lite (default) | `neca` CLI; no Web server | CLI-only certificate management |
| neca-web | `necad` Web server; read-only by default | Browser certificate review and downloads |
| neca-web + helper | root helper; write access can be enabled | issue/revoke/reissue from the browser |

lite and neca-web use the same release tag. The Web server listens on loopback by
default and is not exposed directly to the Internet.

## 🛡️ Safety

1. `ca.key` is the highest-value secret. Keep it `0600 root:root`; never give it to
   the Web container.
2. Web is `readOnly=true` by default. Enable writes only after the helper is healthy
   and you understand the root boundary.
3. Run write commands with `--dry-run` first and keep a PKI backup.
4. Never commit real server identifiers, private keys or passwords.
5. neca is licensed under **AGPL-3.0-only**; network use and redistribution carry
   source-disclosure obligations.

## 📦 Version and source

- Project: <https://github.com/hyhdsl/neca>
- Release: <https://github.com/hyhdsl/neca/releases/tag/v1.0.1>
- License: <https://github.com/hyhdsl/neca/blob/main/LICENSE>
