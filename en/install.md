# Install, upgrade and rollback

> This page has exactly two installation paths: **Docker Compose (Method A)** and **Linux binary (Method B)**.
> Choose one path; do not mix them. A1–A6 and B1–B4 mark the start and end of each path.
> Upgrade, rollback and uninstall are shared post-install operations (C1–C3).

## 0. Process boundaries

| Path | Starts at | Ends at | Requires |
|---|---|---|---|
| Method A: Docker Compose | **A1 Prepare directories and files** | **A6 Verify the Docker install** | Docker, Compose and the repo deploy files |
| Method B: Linux binary | **B1 Download and verify** | **B4 Check config/PKI prerequisites** | Linux amd64/arm64 and OpenSSL; CLI only |
| Shared flow | **C1 Upgrade** | **C3 Uninstall** | A completed A or B install |

**Whole-page start:** choose A or B in this section.

**Whole-page end:** complete A6 or B4.
**Do not mix paths:** Docker runs A1–A6; binary runs B1–B4. The binary path does not include Web or helper.

## Method A: Docker Compose installation (A1–A6)

**Scope:** A1–A6; A5 is the optional Web write capability. Completing A6 completes the Docker installation.

### A1. Prepare directories and files

From the repository root (or after obtaining the deploy files):

~~~bash
install -d -m 0755 /opt/ipsec
install -m 0600 deploy/neca.env.example /opt/ipsec/neca.env
install -m 0644 deploy/docker-compose.neca.yml /opt/ipsec/docker-compose.neca.yml
install -m 0755 deploy/init-web-secrets.sh /opt/ipsec/init-web-secrets.sh
install -d -m 0750 -o root -g neca-web /opt/ipsec/config
install -m 0600 deploy/neca.json.example /opt/ipsec/config/neca.json
~~~

`neca.env` carries only bootstrap configuration:

~~~bash
NECA_CONFIG=/opt/ipsec/config/neca.json
~~~

Put real `SERVER_ID`, `ORG` and `CA_CN` in `/opt/ipsec/config/neca.json`; never commit them.
Mount the config directory separately so SIGHUP sees atomic replacements.

### A2. Install the command wrapper

~~~bash
install -m 0755 deploy/neca-wrapper.sh /usr/local/bin/neca
neca version
neca config validate
~~~

The wrapper uses `docker compose run -T` in non-interactive environments and forwards `NECA_LANG`.

### A3. Run lite

~~~bash
cd /opt/ipsec
docker compose -f docker-compose.neca.yml run --rm neca cert list
~~~

Pull a pinned image if needed:

~~~bash
docker pull ghcr.io/hyhdsl/neca:v0.7.0
~~~

### A4. Optional read-only Web

~~~bash
cd /opt/ipsec
NECA_IMAGE=ghcr.io/hyhdsl/neca:v0.7.0 ./init-web-secrets.sh
docker compose -f docker-compose.neca.yml --profile web up -d web
~~~

The script interactively asks for the administrator password and creates `/opt/ipsec/web-secrets/`.
Save the password in a password manager. If your deployment generated a temporary initial password file, delete it after the first login.

The default port is `127.0.0.1:18080`. Restart `neca-web` after changing `web.listen`.

### A5. Optional Web writes

~~~bash
cd /opt/ipsec
docker compose -f docker-compose.neca.yml --profile web --profile web-write up -d web web-helper
neca web status
neca web readonly off
~~~

Write access is not the default. `readonly off` requires a healthy helper, correct socket permissions and `ca.key`.
See [Web panel](web.md) and [Security, permissions and backups](security.md).

### A6. Verify the Docker install (Method A ends here)

After A6, the Docker installation path is complete:

~~~bash
neca version
neca config validate
neca web status
neca doctor
~~~

If Web is enabled, open it through OpenSSH local forwarding and confirm the login page, certificate list and download entry.

## Method B: Linux binary installation (B1–B4)

**Scope:** B1–B4. This installs the lite CLI only; it does not install Web or helper.
Certificate operations still require configuration, PKI and OpenSSL.

### B1. Download and verify the binary

~~~bash
VERSION=v0.7.0
curl -fLO "https://github.com/hyhdsl/neca/releases/download/${VERSION}/neca-linux-amd64"
curl -fLO "https://github.com/hyhdsl/neca/releases/download/${VERSION}/checksums.txt"
grep " neca-linux-amd64$" checksums.txt > /tmp/neca-checksum
sha256sum -c /tmp/neca-checksum
~~~

On arm64, replace `neca-linux-amd64` with `neca-linux-arm64`.
This path was verified in a Debian bookworm container for download, checksum, install, `--version` and `help`;
it has not been verified as a direct replacement for the Docker deployment on a production host.

### B2. Install to a system path

~~~bash
sudo install -m 0755 neca-linux-amd64 /usr/local/bin/neca
~~~

Confirm that `openssl` is available. The binary path does not provide a Web panel; return to Method A if you need Web or helper.

### B3. Verify the binary install

~~~bash
neca --version
neca help
neca --lang en-US help
~~~

### B4. Check configuration and PKI prerequisites (Method B ends here)

The binary installs only the CLI. Before issuing or revoking certificates, prepare the configuration,
PKI directories and OpenSSL as described in [Security, permissions and backups](security.md). Start with:

~~~bash
neca config validate
neca doctor
~~~

Completing B4 completes the binary installation path.

## Shared flow: upgrade, rollback and uninstall (C1–C3)

**Scope:** C1–C3; starts with an upgrade and ends with an uninstall. These steps apply to a completed A or B install.

### C1. Upgrade

Back up first:

~~~bash
tar -C /opt/ipsec -czf /root/ipsec-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  pki clients swanctl neca.json web-secrets
~~~

Docker path: update the compose image tags, then run:

~~~bash
docker pull ghcr.io/hyhdsl/neca:v0.7.0
docker pull ghcr.io/hyhdsl/neca-web:v0.7.0
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web up -d web
~~~

Binary path: repeat B1–B3 and replace `/usr/local/bin/neca`.
Changing `readOnly` or `idleTimeout` requires a SIGHUP:

~~~bash
docker kill --signal=HUP neca-web
~~~

`up -d` alone does not make a running Web process reread configuration; listener changes such as `web.listen` still require a restart.

### C2. Rollback

1. Stop Web/helper.
2. Restore `pki`, `clients`, `swanctl`, `neca.json` and `web-secrets`.
3. Switch the compose tags back to the previous release, or restore the previous binary.
4. Start the services or verify the restored binary with `neca --version`.

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web stop web web-helper
~~~

lite and neca-web must use the same version; do not mix an old CLI with a new Web.

### C3. Uninstall (shared flow ends here)

Removing containers does not delete `/opt/ipsec` data. To remove the services:

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write down
~~~

For the binary install, remove `/usr/local/bin/neca` manually. Back up before uninstalling; do not run commands that delete the data directory.

## Deep dive

- [Full server deployment guide](https://github.com/hyhdsl/neca/blob/main/docs/development/deploy.md)
- [Security, permissions and backups](security.md)
