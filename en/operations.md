# Deployment and daily operations

## Daily checks

~~~bash
neca version
neca cert list
neca web status
docker compose -f /opt/ipsec/docker-compose.neca.yml ps
~~~

Pay attention to:

- whether Web is still `readOnly=true` (unless write access has been explicitly enabled);
- whether the helper is healthy;
- certificate expiry and the number of CRL entries;
- free disk space under `/opt/ipsec`;
- whether the audit log keeps growing;
- whether `downloads` is accumulating.

## Web logs

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web logs -f web
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write logs -f web-helper
~~~

Logs may contain CNs, paths and error details. Do not paste fragments containing
passwords, cookies or private keys into public channels.

## Backups

Back up at least:

~~~bash
tar -C /opt/ipsec -czf /root/ipsec-backup-$(date +%Y%m%d-%H%M%S).tar.gz pki clients swanctl neca.json web-secrets
~~~

The backup contains the CA private key and Web session secrets, so protect it at the
highest sensitivity level:

- store it on an encrypted disk or in an encrypted archive;
- restrict reads to root;
- never upload it to a public repository, public cloud drive or CI artifact;
- rehearse restoration regularly.

## Restore

1. Stop Web/helper.
2. Restore `pki`, `clients`, `swanctl`, `neca.json` and `web-secrets` from the backup.
3. Verify the `index.txt` and CRL hashes.
4. Start Web in read-only mode.
5. Verify login, the certificate list and downloads.
6. Only then decide whether to enable write access.

## Upgrade and rollback

Pin version tags; do not use `latest` as a production dependency:

~~~bash
docker pull ghcr.io/hyhdsl/neca:v1.0.1
docker pull ghcr.io/hyhdsl/neca-web:v1.0.1
~~~

Back up before upgrading. After upgrading, verify:

~~~bash
neca version
neca cert list
neca web status
curl -fsS http://127.0.0.1:18080/healthz
~~~

`healthz` is only for loopback health checks. To roll back, restore the old image tags
and the corresponding backup; do not mix versions.

## PKI integrity

Capture these values before and after a change:

~~~bash
sha256sum /opt/ipsec/pki/ca/index.txt
sha256sum /opt/ipsec/swanctl/x509crl/ca.crl.pem
~~~

Record the hashes before and after the change so the operation can be explained:

- `index.txt`
- CRL

A hash change does not automatically mean a fault; issuance, revocation and reissue all
change them. What matters is that the change is explainable, audited and backed up.

## CRL and strongSwan

The default `creds` mode hot-loads credentials through `swanctl --load-creds` without
restarting strongSwan. Do not restart the `ipsec-vpn` container unless troubleshooting
clearly requires it; a restart interrupts existing connections.

~~~bash
docker exec ipsec-vpn swanctl --list-crls
docker logs ipsec-vpn --since 5m 2>&1 | grep -i crl
~~~

## Cleanup

~~~bash
neca pki downloads clean --expired
~~~

Check `/opt/ipsec/downloads` regularly. Do not delete files under `pki`, `clients` or
`swanctl` just to free space; confirm the operational meaning first.

## Audit

The default audit log is `/opt/ipsec/pki/audit.log`. Recommended practices:

- append-only; do not edit it by hand;
- archive it regularly and restrict access;
- retain it with system logs/monitoring;
- if a certificate anomaly occurs, preserve the logs before troubleshooting.

## Incident response order

1. Stop further writes (`neca web readonly on`; stop Web if necessary).
2. Preserve logs, audit records, current PKI hashes and a backup.
3. Determine whether the cause is certificates, the CRL, containers or permissions.
4. Reproduce in a test environment.
5. After service is restored, complete the root-cause and audit records.

Do not run `delete`, `reissue` or a broad `downloads clean --all` without a backup.
