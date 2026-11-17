# 安装、升级与回滚

## 选择部署方式

推荐使用 Docker Compose：

- **lite**：只运行 `neca` CLI，适合只做证书管理。
- **neca-web**：额外运行 Web 面板，默认只读。
- **web-write**：额外运行 root helper，允许在 Web 中签发/吊销/重签。

Release 也提供 linux amd64/arm64 二进制；但 lite 依赖 `openssl` 与 Docker CLI，
容器化部署更省事。生产请固定版本标签，不要依赖 `latest`。

## 二进制安装（Linux CLI）

本路径已在 Debian bookworm 容器中验证下载、校验、安装和 `--version`/`help`；
它只安装 lite CLI，不启动 Web。完整证书操作仍需按部署文档准备配置、PKI 和 OpenSSL；
需要 Web 或 helper 时使用下面的 Docker Compose 方式。

~~~bash
VERSION=v0.7.0
curl -fLO "https://github.com/hyhdsl/neca/releases/download/${VERSION}/neca-linux-amd64"
curl -fLO "https://github.com/hyhdsl/neca/releases/download/${VERSION}/checksums.txt"
grep ' neca-linux-amd64$' checksums.txt > /tmp/neca-checksum
sha256sum -c /tmp/neca-checksum
sudo install -m 0755 neca-linux-amd64 /usr/local/bin/neca
neca --version
neca help
~~~

arm64 主机把 `neca-linux-amd64` 换成 `neca-linux-arm64`。安装前请确认 `openssl` 可用；
证书写入和 Docker 运维所需配置仍以完整部署文档为准。

## 1. 准备目录与文件

在服务器上：

~~~bash
install -d -m 0755 /opt/ipsec
install -m 0600 deploy/neca.env.example /opt/ipsec/neca.env
install -m 0644 deploy/docker-compose.neca.yml /opt/ipsec/docker-compose.neca.yml
install -m 0755 deploy/init-web-secrets.sh /opt/ipsec/init-web-secrets.sh
install -d -m 0750 -o root -g neca-web /opt/ipsec/config
install -m 0600 deploy/neca.json.example /opt/ipsec/config/neca.json
~~~

`neca.env` 只保留 bootstrap：

~~~bash
NECA_CONFIG=/opt/ipsec/config/neca.json
~~~

真实 `SERVER_ID`、`ORG`、`CA_CN` 写入 `/opt/ipsec/config/neca.json`，不要提交仓库。配置目录单独挂载给 Web，SIGHUP 才能看到原子替换后的新文件。

## 2. 安装命令包装

~~~bash
install -m 0755 deploy/neca-wrapper.sh /usr/local/bin/neca
neca version
neca config validate
~~~

包装命令会在非交互环境自动使用 `docker compose run -T`，并把 `NECA_LANG` 传入容器。

## 3. 启动 lite

~~~bash
cd /opt/ipsec
docker compose -f docker-compose.neca.yml run --rm neca cert list
~~~

或先拉取固定版本镜像：

~~~bash
docker pull ghcr.io/hyhdsl/neca:v0.7.0
~~~

## 4. 可选：启动只读 Web

~~~bash
cd /opt/ipsec
NECA_IMAGE=ghcr.io/hyhdsl/neca:v0.7.0 ./init-web-secrets.sh
docker compose -f docker-compose.neca.yml --profile web up -d web
~~~

脚本会以交互方式提示设置管理员密码（不回显、不写入 shell 历史），并创建
`/opt/ipsec/web-secrets/`。请立即把密码保存到密码管理器；如果部署流程另外生成了
初始密码文件，首次登录后删除它。需要重置时重新生成 `password.hash`。

默认端口是 `127.0.0.1:18080`。修改 `web.listen` 后必须重启 `neca-web`。

## 5. 可选：开启 Web 写能力

~~~bash
cd /opt/ipsec
docker compose -f docker-compose.neca.yml --profile web --profile web-write up -d web web-helper
neca web status
neca web readonly off
docker kill --signal=HUP neca-web
~~~

写能力不是默认值。`readonly off` 需要 helper 可用、socket 权限正确且 CA 私钥存在。
详见 [Web 面板](web.md) 与 [安全、权限与备份](security.md)。

## 6. 升级

升级前备份：

~~~bash
tar -C /opt/ipsec -czf /root/ipsec-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  pki clients swanctl neca.json web-secrets
~~~

然后：

~~~bash
docker pull ghcr.io/hyhdsl/neca:v0.7.0
docker pull ghcr.io/hyhdsl/neca-web:v0.7.0
# 更新 compose 中的镜像标签，或使用 deploy 目录中的当前版本
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web up -d web
~~~

修改 `readOnly` / `idleTimeout` 后发送 SIGHUP：

~~~bash
docker kill --signal=HUP neca-web
~~~

只执行 `up -d` 不会让已运行的 Web 进程重新读取配置；`web.listen` 等监听类配置仍需重启。

## 7. 回滚

1. 停止 Web/helper；
2. 恢复备份的 `pki`、`clients`、`swanctl`、`neca.json`、`web-secrets`；
3. 把 compose 镜像标签切回上一个版本；
4. 重新启动对应容器。

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web stop web
~~~

lite 与 neca-web 必须使用同一版本；不要混用旧 CLI 与新 Web。

## 8. 卸载

停止并删除容器不会删除 `/opt/ipsec` 数据。确认不再需要 Web 时：

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write down
~~~

不要执行会删除数据目录的命令；卸载前先备份。

## 深入阅读

- [服务器部署（完整版）](https://github.com/hyhdsl/neca/blob/main/docs/development/deploy.md)
- [安全、权限与备份](security.md)
