# Certificate management

## Data model

neca keeps OpenSSL-compatible files instead of a database:

| Path | Purpose |
|---|---|
| `/opt/ipsec/pki/ca/index.txt` | Certificate index and source of truth |
| `/opt/ipsec/pki/meta/<CN>.env` | Platforms, dates and serial metadata |
| `/opt/ipsec/clients/<CN>/` | Client certificate, p12 and platform packages |
| `/opt/ipsec/swanctl/x509crl/ca.crl.pem` | CRL loaded by strongSwan |
| `/opt/ipsec/pki/audit.log` | Append-only audit log |

Do not edit `index.txt` without validating it afterwards; use `neca pki index sync` for
manual imports.

## CN rules

CN may contain only:

~~~text
A-Z a-z 0-9 . _ -
~~~

CN is also the VPN identity, directory name and certificate Common Name. Do not use
spaces, slashes, non-ASCII characters or `..`.

## Commands

| Command | Purpose |
|---|---|
| `neca cert list` | List certificates |
| `neca cert verify [CN]` | Check p12 algorithm, certificate count and password |
| `neca cert issue <CN> ...` | Issue certificate and packages |
| `neca cert reissue <CN> ...` | Archive and reissue |
| `neca cert revoke <CN>` | Revoke and update the CRL |
| `neca cert unrevoke <CN>` | Restore a revoked certificate and update the CRL |
| `neca cert trash <CN>` | Move into the recycle bin (restorable) |
| `neca trash list` / `restore <ID>` | Inspect or restore trash entries |
| `neca trash purge <ID>` | Backend physical purge (Y/N, default N) |
| `neca cert platform add / set <CN> ...` | Add packages or update platform metadata |
| `neca cert package <CN> ...` | Rebuild packages from an existing certificate |
| `neca cert export issued / revoked` | Export CSV |
| `neca pki index sync` | Rebuild index/meta for manually installed certificates |

All write commands support `--dry-run`.

## Issue

~~~bash
neca cert issue alice \
  --platforms windows,linux,macos,android,ios \
  --days 365 \
  --installers windows \
  --p12-profile compat \
  --zip
~~~

- `--platforms`: comma-separated platform list.
- `--days`: validity period; default 365.
- `--installers`: `none`, `all` or a platform list; default `none`.
- `--p12-profile`: `compat` (default, 3DES/SHA1) or `modern` (AES-256/SHA256).
- `--zip`: create a TTL-limited archive under `downloads/`.

The private key remains on the server and is not included in packages.

## Inspect and verify

~~~bash
neca cert list
neca cert list --json
neca cert verify alice
neca cert verify
~~~

`verify` checks that the p12 contains only the client certificate, uses the expected
algorithm and has a valid password length. It does not replace an import test on the
target client.

## Reissue

~~~bash
neca cert reissue alice \
  --platforms windows,android,ios \
  --days 365 \
  --zip
~~~

Reissue archives the old certificate and creates a new serial. Clients must import the
new package. Back up `clients/alice/` and `pki/meta/alice.env` first.

## Revoke and unrevoke

~~~bash
neca cert revoke alice
neca cert unrevoke alice
~~~

Both regenerate the CRL. The default `creds` mode uses `swanctl --load-creds` without
restarting strongSwan; `restart` mode keeps the old behavior. Check the mode with
`neca config show`.

## Recycle bin

`cert trash` only moves the certificate and related files into the recycle bin; it does not delete them. The Web panel also only moves certificates there.

~~~bash
neca cert trash alice
neca trash list
neca trash restore <ID>
~~~

The recycle bin is retained indefinitely by default. Restoring a revoked certificate does not undo the revocation. Physical purge:

~~~bash
neca trash purge <ID>
~~~

`trash purge` requires one Y/N confirmation (default N); backend automation may use `--yes`. It no longer asks for `PURGE <CN>` and does not promise secure erasure on SSDs, snapshots or backups. Back up before purging.

## Packages and export

~~~bash
neca cert platform add alice --platforms linux,macos
neca cert platform set alice --platforms windows,android
neca cert package alice --platforms windows,linux,macos,android,ios --zip
neca cert export issued --out /root/issued.csv
neca cert export revoked --out /root/revoked.csv
neca pki index sync
~~~

CSV uses UTF-8 BOM for Excel. `index sync` only rebuilds index/meta; it does not reissue.

## Audit and JSON

~~~bash
neca cert list --json
neca cert revoke alice --json
~~~

The audit log records time, operator, action, CN, serial and result. Do not put
passwords or private keys into tickets or JSON logs.
