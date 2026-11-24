# 🧰 Troubleshooting

Preserve the scene first: logs, `neca web status`, `docker compose ps`, audit data and
recent changes. Do not paste passwords, cookies, private keys or full configuration
into public channels.

## `neca: command not found`

Install the wrapper command:

~~~bash
install -m 0755 deploy/neca-wrapper.sh /usr/local/bin/neca
which neca
neca version
~~~

If it is in a non-standard directory, check `PATH` and the compose path inside the
wrapper.

## Docker unavailable / permission denied

- Confirm Docker Desktop or the daemon is running.
- Confirm the executing user can access `/var/run/docker.sock`.
- The lite container triggers `swanctl --load-creds` through `docker.sock`.
- Do not expose `docker.sock` to the Web container.

## Web fails to start: address already in use

If `18080` is in use, edit `neca.json`:

~~~json
{ "web": { "listen": "127.0.0.1:18080" } }
~~~

Then restart:

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web restart web
~~~

The default is 18080. If another service owns the port, choose another loopback port
and restart `neca-web`.

## readOnly was changed, but the page still shows the old state

Running `docker compose up -d` does not reload an existing Web process. Send SIGHUP:

~~~bash
docker kill --signal=HUP neca-web
neca web status
~~~

A `web.listen` change still requires restarting the web container.

## The page reports "helper unavailable"

Check:

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write ps
neca web status
ls -l /run/neca/web-helper.sock
~~~

Common causes:

- the `web-write` profile was not started;
- the helper container is not healthy;
- the socket owner/group is not `root:neca-web`;
- the Web container UID/GID is not 10001;
- the CA private key is missing or unreadable.

## `readonly off` is rejected

The command requires:

1. execution as root;
2. the Web component to be installed;
3. a healthy helper;
4. an accessible socket;
5. `ca.key` to exist.

Start with `neca web status`, then inspect the helper logs. Do not bypass the checks by
editing the JSON to `false`; the helper will still reject the change and the state may
become inconsistent.

## Login failure / rate limiting

- Confirm the password matches the initial password set during deployment (or the
  initial password file).
- Check that the system time is accurate.
- Wait for the rate-limit window to end; avoid repeated brute-force attempts.
- If the password is lost, regenerate `password.hash` and restart web.
- Check that `web-secrets` is `0710` and each file is `0640`.

## Web starts but cannot read certificates

Check the group permissions on `neca.json`, `clients` and `downloads`:

~~~bash
chgrp neca-web /opt/ipsec/config/neca.json
chmod 0640 /opt/ipsec/config/neca.json
chgrp -R neca-web /opt/ipsec/clients /opt/ipsec/downloads
chmod -R g+rX /opt/ipsec/clients /opt/ipsec/downloads
~~~

Web only needs `pki/ca/index.txt`, `pki/meta`, `clients`, `downloads` and the public
CRL; do not mount the entire `pki` directory into Web just for convenience.

## p12 verification fails

~~~bash
neca cert verify alice
~~~

Common causes:

- the `modern` profile was used, but the old client only supports `compat`;
- the p12 password file does not match the archive;
- the p12 contains the CA certificate or the chain length is wrong;
- the p12 was modified manually on the client.

Regenerate it:

~~~bash
neca cert package alice --p12-profile compat --zip
~~~

## The CRL does not take effect after revocation

Confirm the reload mode:

~~~bash
neca config show
docker exec ipsec-vpn swanctl --load-creds
docker logs ipsec-vpn --since 2m 2>&1 | grep -i crl
~~~

The default `creds` mode should hot-load the CRL. If the configuration uses `restart`,
restart the container. Do not repeatedly restart strongSwan without confirming the
cause, because that interrupts existing connections.

## Download link returns 404 / 410

- 410/404 usually means the link expired, was cleaned, or the ID does not exist.
- The default TTL is 30 minutes.
- Run `neca cert package <CN> --zip` again.
- Check `neca pki downloads list`.

## One-click installer script was not generated

The default is `--installers none`. Enable it explicitly:

~~~bash
neca cert issue alice --platforms windows,linux --installers windows,linux
~~~

Linux/macOS scripts do not cover every distribution; verify them on a test machine
first.

## Web behaves abnormally after an upgrade

- Confirm lite and neca-web use the same version tag.
- Check whether only the image was updated without recreating/restarting the container.
- Inspect `docker compose logs` and `neca web status`.
- When rolling back, restore both the old image and the backup; do not roll back only
  one component.

## Web reports permission denied while reading the index

If Web reports `/opt/ipsec/pki/ca/index.txt: open ...: permission denied`, the CA index
was usually rewritten with group `root`, while the Web process runs as UID/GID 10001
(group `neca-web`):

~~~bash
chgrp neca-web /opt/ipsec/pki/ca /opt/ipsec/pki/ca/index.txt
chmod 2755 /opt/ipsec/pki/ca
chmod 0640 /opt/ipsec/pki/ca/index.txt
docker exec neca-web sh -c 'head -c 1 /opt/ipsec/pki/ca/index.txt >/dev/null && echo OK'
~~~

The setgid bit on `pki/ca` ensures that future CLI atomic rewrites of `index.txt`
inherit the `neca-web` group. After correction, refresh the Web page; restarting Web
or strongSwan is not required. Re-running `init-web-secrets.sh` also corrects the
permissions.

## Still unresolved

Collect the following before opening an Issue:

~~~bash
neca version
neca web status
docker compose -f /opt/ipsec/docker-compose.neca.yml ps
~~~

Also include: version, deployment shape (lite/web/web-write), reproduction steps and
redacted error messages.
