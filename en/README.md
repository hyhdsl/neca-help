# neca Help Center

> Applies to **neca v0.3.0** (2026-11-15). neca manages strongSwan IKEv2 user
> certificates and client packages; it is not the VPN server itself.

## Start here

- First run: [Quick start](getting-started.md)
- Server install/upgrade: [Install, upgrade and rollback](install.md)
- Certificates: [Certificate management](certificates.md)
- Browser panel: [Web panel](web.md)
- Hardening and backups: [Security, permissions and backups](security.md)
- Problems: [FAQ](faq.md)
- Automation: [Web API (experimental)](api.md)

## English coverage

This is the core English subset. Advanced pages such as package details, daily
operations, troubleshooting and development are currently Chinese-only; use the
language switch or [中文完整文档](https://neca.nekobiglazyer.work/) for those pages.

## Runtime forms

| Form | Contents | Typical use |
|---|---|---|
| lite (default) | `neca` CLI; no Web server | CLI-only certificate management |
| neca-web | `necad` Web server; read-only by default | Browser certificate review and downloads |
| neca-web + helper | root helper; write access can be enabled | issue/revoke/reissue from the browser |

lite and neca-web use the same release tag. The Web server listens on loopback by
default and is not exposed directly to the Internet.

## Safety

1. `ca.key` is the highest-value secret. Keep it `0600 root:root`; never give it to
   the Web container.
2. Web is `readOnly=true` by default. Enable writes only after the helper is healthy
   and you understand the root boundary.
3. Run write commands with `--dry-run` first and keep a PKI backup.
4. Never commit real server identifiers, private keys or passwords.
5. neca is licensed under **AGPL-3.0-only**; network use and redistribution carry
   source-disclosure obligations.

## Version and source

- Project: <https://github.com/hyhdsl/neca>
- Release: <https://github.com/hyhdsl/neca/releases/tag/v0.3.0>
- Help source: <https://github.com/hyhdsl/neca/tree/main/docs/help>
- License: <https://github.com/hyhdsl/neca/blob/main/LICENSE>

> AI disclosure: code, tests and documentation were produced with AI assistance.
> Review and test before production use.
