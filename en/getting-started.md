# Quick start

This page completes a minimal "list → issue → verify → download" flow. Run commands
as root on the server. Replace `alice` with the real user identifier.

## Prerequisites

- A lite deployment and the `neca` wrapper command (see [install](install.md)).
- A running strongSwan container.
- A PKI under `/opt/ipsec`. Do not copy production commands into a fresh environment
  without reviewing them.

~~~bash
neca version
neca config show
neca cert list
~~~

If `neca` is missing, install the wrapper:

~~~bash
install -m 0755 deploy/neca-wrapper.sh /usr/local/bin/neca
~~~

## 1. Inspect certificates

~~~bash
neca cert list
neca cert list --json
~~~

`list` shows status, serial number, platforms and expiry. The source of truth is
`/opt/ipsec/pki/ca/index.txt`.

## 2. Issue a certificate

Dry-run first:

~~~bash
neca cert issue alice \
  --platforms windows,android,ios \
  --days 365 \
  --installers windows \
  --zip \
  --dry-run
~~~

Then remove `--dry-run`:

~~~bash
neca cert issue alice \
  --platforms windows,android,ios \
  --days 365 \
  --installers windows \
  --zip
~~~

Output goes to `/opt/ipsec/clients/alice/`. With `--zip`, a time-limited archive is
created under `/opt/ipsec/downloads/`.

- Platforms: `windows`, `linux`, `macos`, `android`, `ios`.
- `--installers` defaults to `none`; opt in explicitly.
- `--p12-profile compat` is the default for MIUI/older Android; use `modern` for new clients.

## 3. Verify the p12

~~~bash
neca cert verify alice
~~~

Batch verification:

~~~bash
neca cert verify
~~~

## 4. Check downloads

~~~bash
neca pki downloads list
~~~

The default TTL is 30 minutes. Clean expired files:

~~~bash
neca pki downloads clean --expired
~~~

## 5. Optional Web panel

The Web panel is read-only by default and listens on loopback. Production usually
uses an SSH tunnel:

~~~bash
ssh -L 18080:127.0.0.1:18080 root@<SERVER>
# Open http://127.0.0.1:18080
~~~

If the server still uses the default port, replace 18080 with 18080. See
[Web panel](web.md).

## Output and exit codes

- Prefixes: `[OK]`, `[INFO]`, `[WARN]`, `[ERR]`.
- Exit codes: `0` success; `1` business error; `2` usage; `3` config/environment.
- All commands support `--json`; write commands support `--dry-run` when applicable.

## Next

- Revoke, reissue or delete: [Certificate management](certificates.md).
- Browser operations: [Web panel](web.md).
- Automation: [Web API (experimental)](api.md).
