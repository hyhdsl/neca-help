# Command cheat sheet

> Applies to neca v1.0.0.
> Run write commands with `--dry-run` first. Interactive confirmation defaults to N; automation may pass `--yes` (audited).

## Version, config and diagnostics

~~~bash
neca version
neca config validate
neca config show
neca doctor
~~~

## List and verify certificates

~~~bash
neca cert list
neca cert list --json
neca cert verify alice
neca cert verify alice --json
~~~

## Issue and reissue

~~~bash
neca cert issue alice --platforms windows,android --zip --dry-run
neca cert issue alice --platforms windows,android --zip
neca cert reissue alice --platforms windows,android --days 365
~~~

## Revoke and unrevoke

~~~bash
neca cert revoke alice --yes
neca cert unrevoke alice --yes
~~~

## Recycle bin

~~~bash
neca cert trash alice
neca trash list
neca trash restore <entry-id>
neca trash purge <entry-id> --yes
~~~

Purge expired entries:

~~~bash
neca trash purge --expired --yes
~~~

## Platforms and index

~~~bash
neca cert platform add alice --platforms linux
neca cert platform set alice --platforms windows,android,linux
neca pki index sync
~~~

## Web panel

~~~bash
neca web status
neca web readonly status
neca web readonly off
neca web readonly on
neca web lock --yes
neca web lockdown --yes
~~~

- `readonly off` requires root, a healthy helper and `ca.key`.
- `lock` sets read-only, increments sessionEpoch and invalidates old sessions.
- `lockdown` stops only Web/helper; it does not touch SSH, strongSwan or the VPN.

## Export and maintenance

~~~bash
neca cert export issued
neca cert export revoked
neca pki downloads list
neca doctor
~~~

## Language, JSON and automation

~~~bash
neca --lang en-US help
neca --lang en-US cert list
neca --json cert list
neca --dry-run cert issue alice --platforms windows
~~~

> Deployment, upgrades and rollback: [Install, upgrade and rollback](install.md). Web permissions and backups: [Security, permissions and backups](security.md).
