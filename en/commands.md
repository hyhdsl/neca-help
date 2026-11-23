# Command reference

> Applies to **neca v1.0.1**. This page covers every built-in subcommand, its arguments, confirmation behavior and exit codes.
> Global options may appear anywhere on the command line; append `--help` to any command to see that level's usage.
> Run write commands with `--dry-run` first.

## Global options

| Option | Meaning |
|---|---|
| `--json` | Emit structured JSON; interactive commands usually require `--yes` in JSON mode |
| `--dry-run` | Print the write command that would run; do not execute or modify files |
| `--yes` / `-y` | Skip the Y/N confirmation; the audit entry records explicit confirmation |
| `-q` / `--quiet` | Suppress normal output; `--json` takes precedence |
| `--lang zh-CN\|en-US` | Override the language for this invocation |
| `--config <file>` | Use a specific config file (sets `NECA_CONFIG`) |
| `--reload-mode creds\|restart` | Override CRL reload mode (sets `NECA_RELOAD_MODE`) |
| `--version` / `-V` | Show version, commit, build time and platform |
| `-h` / `--help` | Show help for the current level |

`--dry-run` intercepts only state-changing commands: cert issue/reissue/revoke/unrevoke/trash/platform/package/export, pki index sync, pki downloads clean, trash restore/purge, and web readonly on/off, idle-timeout changes, export on/off, lock and lockdown. Read-only commands run normally.

**Exit codes**: `0` success (warnings allowed); `1` usage error; `2` runtime failure; `3` precondition not met (certificate already exists, missing CA, helper unavailable, insufficient privileges).

## Top-level commands

### neca version

~~~bash
neca version
neca --json version
~~~

Shows version, commit, build time and target platform.

### neca help

~~~bash
neca help
neca cert --help
neca cert issue --help
~~~

Running `neca` with no arguments starts the interactive wizard; `neca wizard` opens it explicitly. Menu items 0–10 map to exit, issue, revoke, unrevoke, reissue, add platform, export issued, export revoked, list, index sync and move to recycle bin. The equivalent CLI command is printed before each action.

## config

### config init

~~~bash
neca config init
neca config init --force
neca config init --from-env
~~~

- Writes the config file; an existing file requires `--force`.
- New files are created with mode 0600; overwriting preserves the existing mode.
- `--from-env` writes the current effective configuration instead of the example template.

### config show / validate / path

~~~bash
neca config show
neca config validate
neca config path
~~~

- `show`: print a JSON snapshot of the effective configuration.
- `validate`: load and validate the configuration; prints the config path on success.
- `path`: print only the config file path.

### config export-env

~~~bash
neca config export-env
~~~

Prints shell variable assignments. The output may contain sensitive settings; use it in a controlled terminal and never paste it into logs, tickets or public chat.

## cert: certificate management

### cert list

~~~bash
neca cert list
neca --json cert list
~~~

Lists CN, status, serial, platforms, expiry and note, plus the CRL revoked count. It accepts no positional arguments.

### cert verify

~~~bash
neca cert verify
neca cert verify alice
neca cert verify alice bob
~~~

Checks client p12 files: client-only content, expected algorithms and password length. With no CN it checks every client under `clients/`. Warnings go to stderr while the exit code stays 0 (matching the legacy script); it does not replace an import test on the real device.

### cert issue

~~~bash
neca cert issue alice --platforms windows,android --days 365 --zip
neca cert issue alice --platforms windows,linux --installers windows --p12-profile compat --zip --dry-run
~~~

| Argument | Meaning |
|---|---|
| `<CN>` | Required; allowed characters: `A-Z a-z 0-9 . _ -` |
| `--platforms a,b` | Platform list; defaults to configuration when omitted |
| `--days N` | Validity in days; defaults to configuration when omitted |
| `--installers none\|all\|list` | Generate one-click installer scripts; defaults to configuration (usually none) |
| `--p12-profile compat\|modern` | p12 algorithms: compat uses 3DES/SHA1, modern uses AES-256/SHA256 |
| `--zip` | Create a download zip with an expiry time |
| `--no-reload` | Do not reload the CRL automatically |

Private keys stay on the server and are not included in the client zip. Issuance writes an audit entry.

### cert reissue

~~~bash
neca cert reissue alice --platforms windows,android --days 365 --zip
~~~

Same arguments as issue. Archives the old certificate and creates a new serial; clients must import the new package. Back up `clients/alice/` and `pki/meta/alice.env` before reissuing.

### cert revoke / cert unrevoke

~~~bash
neca cert revoke alice
neca cert unrevoke alice
neca cert revoke alice --no-reload
~~~

Revokes or restores a certificate and rebuilds the CRL; by default it hot-reloads via `swanctl --load-creds` without restarting strongSwan. If the certificate is already in the target state, the command reports it without treating it as an error. `--no-reload` skips the automatic reload.

### cert trash

~~~bash
neca cert trash alice
~~~

Moves the certificate, client packages and metadata to the recycle bin. Nothing is physically deleted and the entry can be restored.

### cert export

~~~bash
neca cert export issued
neca cert export revoked
neca cert export issued --out /tmp/issued.csv
~~~

Exports CSV; the type must be `issued` or `revoked`. Without `--out` the default export path is used. The type and `--out` may appear in either order.

### cert platform

~~~bash
neca cert platform add alice --platforms linux --installers linux --zip
neca cert platform set alice --platforms windows,android,linux
~~~

- `add`: build packages for additional platforms; supports `--installers` and `--zip`.
- `set`: change platform annotations only; accepts only `--platforms`.

### cert package

~~~bash
neca cert package alice --platforms windows,android --installers windows --p12-profile compat --zip
~~~

Rebuilds platform packages from an existing certificate without issuing a new certificate or changing keys. The platform/installer/p12 arguments match issue.

## pki: index and downloads

### pki index sync

~~~bash
neca pki index sync
~~~

Adds index and metadata entries for certificates placed manually. Accepts no extra arguments.

### pki downloads list

~~~bash
neca pki downloads list
~~~

Lists download zips with size and expiry time.

### pki downloads clean

~~~bash
neca pki downloads clean
neca pki downloads clean --all
~~~

Without flags it removes only expired zips; `--all` removes every download. It never touches certificates or platform packages.

## trash: recycle bin

~~~bash
neca trash list
neca trash restore <entry-id>
neca trash purge <entry-id>
neca trash purge --expired
~~~

- `list`: show entry ID, CN and time moved to trash.
- `restore`: restore a certificate and its files by entry ID.
- `purge`: physically delete; interactive confirmation defaults to N, automation may pass `--yes`. `--expired` purges entries beyond the configured retention.
- `--json trash purge` requires `--yes`, otherwise it returns a usage error.

## web: web panel management

### web status

~~~bash
neca web status
neca web
~~~

Shows whether the web panel is enabled, listen address, read-only/writable mode, sessionEpoch, CSV export, idle timeout, Passkey and helper status.

### web readonly

~~~bash
neca web readonly status
neca web readonly on
neca web readonly off
~~~

- `on`: switch to read-only (requires root).
- `off`: switch to writable (requires root); also requires a healthy helper, correct socket permissions and the CA key, preventing a half-state where configuration is writable without a write boundary.
- After a change, send SIGHUP as prompted so the running web server reloads.

### web idle-timeout

~~~bash
neca web idle-timeout status
neca web idle-timeout 30m
neca web idle-timeout 1h
~~~

View or change the idle timeout. Allowed range is 5m–24h, default 30m; the absolute session limit is 24h. Changes require root and a SIGHUP reload.

### web export

~~~bash
neca web export on
neca web export off
~~~

Enable or disable CSV export in the web panel (disabled by default). Requires root; send SIGHUP afterwards.

### web lock

~~~bash
neca web lock --yes
~~~

Forces read-only, increments `sessionEpoch` to invalidate all old sessions, and sends SIGHUP. Requires root; interactive confirmation defaults to N and JSON mode requires `--yes`.

### web lockdown

~~~bash
neca web lockdown --yes
~~~

Stops only the explicitly named `neca-web` and `neca-web-helper` containers; it does not touch SSH, strongSwan or the VPN. Requires root; interactive confirmation defaults to N and JSON mode requires `--yes`.

### web hash-password

~~~bash
read -r -s pw
printf '%s' "$pw" | neca web hash-password
~~~

Reads a password from stdin and prints a bcrypt hash (cost 12); the password must be at least 8 characters. It does not read the config or require root, and is intended for generating `NECA_WEB_PASSWORD_HASH`. The password never appears in argv or logs.

## doctor: diagnostics

~~~bash
neca doctor
neca --json doctor
~~~

Checks configuration, CA certificate, CA key, index, CRL, web configuration, the docker executable, the audit file and the helper. `WARN` does not affect the exit code; any `FAIL` returns exit code 2. FAILs for CA/web files are expected on a lite-only host; judge by deployment form.

## Common combinations

~~~bash
# Rehearse, then issue
neca --dry-run cert issue alice --platforms windows,android --zip
neca cert issue alice --platforms windows,android --zip

# Machine-readable list
neca --json cert list

# Revoke and hot-reload the CRL
neca cert revoke alice

# Automated expired-trash cleanup
neca trash purge --expired --yes

# Triage
neca doctor
neca web status
~~~

> Install, upgrade and rollback: [Install, upgrade and rollback](install.md). Certificate model: [Certificate management](certificates.md). Web permissions: [Web panel](web.md) and [Security, permissions and backups](security.md). Common examples: [Command cheat sheet](cheatsheet.md).
