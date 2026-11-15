# Install, upgrade and rollback

## Choose a deployment

Docker Compose is recommended:

- **lite**: `neca` CLI only.
- **neca-web**: adds the Web panel; read-only by default.
- **web-write**: adds the root helper, allowing issue/revoke/reissue from the Web UI.

Linux amd64/arm64 binaries are also published, but lite needs `openssl` and the Docker
CLI; containers are simpler to operate. Pin a version tag in production; do not depend
on `latest`.

## 1. Prepare files

On the server:

~~~bash
install -d -m 0755 /opt/ipsec
install -m 0600 deploy/neca.env.example /opt/ipsec/neca.env
install -m 0644 deploy/docker-compose.neca.yml /opt/ipsec/docker-compose.neca.yml
install -m 0755 deploy/init-web-secrets.sh /opt/ipsec/init-web-secrets.sh
install -d -m 0750 -o root -g neca-web /opt/ipsec/config
install -m 0600 deploy/neca.json.example /opt/ipsec/config/neca.json
~~~

`neca.env` only carries bootstrap configuration:

~~~bash
NECA_CONFIG=/opt/ipsec/config/neca.json
~~~

Put real `SERVER_ID`, `ORG` and `CA_CN` in `/opt/ipsec/config/neca.json`; never commit them. Mount the config directory separately so SIGHUP sees atomic replacements.

## 2. Install the command wrapper

~~~bash
install -m 0755 deploy/neca-wrapper.sh /usr/local/bin/neca
neca version
neca config validate
~~~

The wrapper uses `docker compose run -T` in non-interactive environments and forwards
`NECA_LANG`.

## 3. Run lite

~~~bash
cd /opt/ipsec
docker compose -f docker-compose.neca.yml run --rm neca list
~~~

Pull a pinned image if needed:

~~~bash
docker pull ghcr.io/hyhdsl/neca:v0.4.1
~~~

## 4. Optional read-only Web

~~~bash
cd /opt/ipsec
NECA_IMAGE=ghcr.io/hyhdsl/neca:v0.4.1 ./init-web-secrets.sh
docker compose -f docker-compose.neca.yml --profile web up -d web
~~~

The script interactively asks for the administrator password and creates
`/opt/ipsec/web-secrets/`. Save the password in a password manager. If your deployment
generated a temporary initial password file, delete it after the first login.

The default port is `127.0.0.1:18080`. This production server uses `18080` because
18080 is occupied by other service. Restart `neca-web` after changing `web.listen`.

## 5. Optional writes

~~~bash
cd /opt/ipsec
docker compose -f docker-compose.neca.yml --profile web --profile web-write up -d web web-helper
neca web status
neca web readonly off
docker kill --signal=HUP neca-web
~~~

`readonly off` requires the helper to be healthy, the socket permissions to be correct
and `ca.key` to exist. The Web panel remains read-only until all checks pass.

## 6. Upgrade

Back up first:

~~~bash
tar -C /opt/ipsec -czf /root/ipsec-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  pki clients swanctl neca.json web-secrets
~~~

Then pull the pinned tags, update the compose image tags, and recreate the Web service:

~~~bash
docker pull ghcr.io/hyhdsl/neca:v0.4.1
docker pull ghcr.io/hyhdsl/neca-web:v0.4.1
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web up -d web
~~~

Changing `readOnly` or `idleTimeout` requires a SIGHUP:

~~~bash
docker kill --signal=HUP neca-web
~~~

Listener (`web.listen`) changes still require a restart.

## 7. Rollback

1. Stop Web/helper.
2. Restore `pki`, `clients`, `swanctl`, `neca.json` and `web-secrets`.
3. Switch the compose tags back to the previous release.
4. Start the services.

lite and neca-web must use the same version; do not mix old CLI with new Web.

## 8. Uninstall

Removing containers does not delete `/opt/ipsec` data:

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write down
~~~

Back up before uninstalling.

## Deep dive

- [Full server deployment guide](https://github.com/hyhdsl/neca/blob/main/docs/deploy.md)
- [Security, permissions and backups](security.md)
