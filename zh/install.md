# 📦 安装、升级与回滚

> 本页只包含两种安装路径：**Docker Compose（方式 A）** 和 **Linux 二进制（方式 B）**。
> 请选择一种路径，不要混用。下面的 A1–A6 / B1–B4 标注了每条路径的起止位置；
> 升级、回滚与卸载属于安装后的共用流程（C1–C3）。

## 0. 总流程边界

| 路径 | 从哪里开始 | 到哪里结束 | 需要什么 |
|---|---|---|---|
| 方式 A：Docker Compose | **A1 准备目录与文件** | **A6 验证 Docker 安装** | Docker、Compose、仓库中的 deploy 文件 |
| 方式 B：Linux 二进制 | **B1 下载与校验** | **B4 检查配置/PKI 前置** | Linux amd64/arm64、OpenSSL；仅 CLI |
| 共用流程 | **C1 升级** | **C3 卸载** | 已完成的 A 或 B |

**整页起点**：先读本节并选择 A 或 B。

**整页终点**：完成 A6 或 B4。
**不要混用**：Docker 方式从 A1 到 A6；二进制方式从 B1 到 B4。二进制方式不包含 Web/helper。

## 0.1 运行资源基线

以下为 2026-11-28 在 Linux amd64、2 vCPU、1.6 GiB 内存环境下的低负载实测与安全余量；CLI 与 Web 分开给出，不包含宿主 Docker、strongSwan、反向代理或高并发/大证书库的额外开销。

| 组件 | 最低（低负载余量） | 建议 |
|---|---|---|
| `neca` CLI | 1 vCPU / 64 MiB 内存 / 256 MiB 可用磁盘 | 1 vCPU / 256 MiB / 1 GiB |
| `neca-web` + helper | 1 vCPU / 256 MiB / 512 MiB 可用磁盘 | 2 vCPU / 512 MiB / 2 GiB |

实测参考：CLI 在 16 MiB 容器限制下可执行 `version`，常驻约 2.8 MiB；Web/helper 常驻合计约 15.5 MiB，healthcheck 为 healthy；两个镜像分别约 126.5 MiB 与 135.7 MiB。Docker、OpenSSL 与浏览器前置按本页安装步骤执行；未单独列出的宿主细节以安装验证通过、可以正常工作为准。

下载发布资产后，请先按 [GPG 验证](gpg.md) 完成公钥指纹核对、checksums 校验与 Git tag 校验，再执行安装命令。

## 方式 A：Docker Compose 安装（A1–A6）

**路径范围**：A1–A6；A5 是可选的 Web 写能力。完成 A6 即完成 Docker 安装。

### A1. 准备目录与文件

在仓库根目录（或已取得 deploy 文件）执行：

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

真实 `SERVER_ID`、`ORG`、`CA_CN` 写入 `/opt/ipsec/config/neca.json`，不要提交仓库。
配置目录单独挂载给 Web，SIGHUP 才能看到原子替换后的新文件。

### A2. 安装命令包装

~~~bash
install -m 0755 deploy/neca-wrapper.sh /usr/local/bin/neca
neca version
neca config validate
~~~

包装命令会在非交互环境自动使用 `docker compose run -T`，并把 `NECA_LANG` 传入容器。

### A3. 启动 lite

~~~bash
cd /opt/ipsec
docker compose -f docker-compose.neca.yml run --rm neca cert list
~~~

或先拉取固定版本镜像：

~~~bash
docker pull ghcr.io/hyhdsl/neca:v1.5.2
~~~

### A4. 可选：启动只读 Web

~~~bash
cd /opt/ipsec
NECA_IMAGE=ghcr.io/hyhdsl/neca:v1.5.2 ./init-web-secrets.sh
docker compose -f docker-compose.neca.yml --profile web up -d web
~~~

脚本会以交互方式提示设置管理员密码（不回显、不写入 shell 历史），并创建
`/opt/ipsec/web-secrets/`。请立即把密码保存到密码管理器；如果部署流程另外生成了
初始密码文件，首次登录后删除它。需要重置时重新生成 `password.hash`。

如果 `/opt/ipsec/pki/ca` 已存在，脚本还会设置组继承并修正 `index.txt` 为
`root:neca-web 0640`，避免 Web 读取索引时报 permission denied。

默认端口是 `127.0.0.1:18080`。修改 `web.listen` 后必须重启 `neca-web`。

### A5. 可选：开启 Web 写能力

~~~bash
cd /opt/ipsec
docker compose -f docker-compose.neca.yml --profile web --profile web-write up -d web web-helper
neca web status
neca web readonly off
~~~

写能力不是默认值。`readonly off` 需要 helper 可用、socket 权限正确且 CA 私钥存在。
详见 [Web 面板](web.md) 与 [安全、权限与备份](security.md)。

### A6. 验证 Docker 安装（方式 A 结束）

完成 A6 后，Docker 安装路径结束：

~~~bash
neca version
neca config validate
neca web status
neca doctor
~~~

可选 Web 已启动时，通过 OpenSSH 本地转发打开面板并确认登录页、证书列表和下载入口。

## 方式 B：Linux 二进制安装（B1–B4）

**路径范围**：B1–B4。只安装 lite CLI，不含 Web/helper；完整证书操作仍需配置、PKI 和 OpenSSL。

### B1. 下载与校验二进制

~~~bash
VERSION=v1.5.2
curl -fLO "https://github.com/hyhdsl/neca/releases/download/${VERSION}/neca-linux-amd64"
curl -fLO "https://github.com/hyhdsl/neca/releases/download/${VERSION}/checksums.txt"
grep " neca-linux-amd64$" checksums.txt > /tmp/neca-checksum
sha256sum -c /tmp/neca-checksum
~~~

arm64 主机把 `neca-linux-amd64` 换成 `neca-linux-arm64`。
本路径已在 Debian bookworm 容器中验证下载、校验、安装、`--version` 和 `help`；
尚未在生产主机直接替换 Docker 部署。

### B2. 安装到系统路径

~~~bash
sudo install -m 0755 neca-linux-amd64 /usr/local/bin/neca
~~~

确认 `openssl` 可用。二进制方式不提供 Web 面板；需要 Web/helper 时返回方式 A。

### B3. 验证二进制安装

~~~bash
neca --version
neca help
neca --lang en-US help
~~~

### B4. 检查配置与 PKI 前置（方式 B 结束）

二进制只负责 CLI 本身；在签发/吊销前，按 [安全、权限与备份](security.md) 准备配置、
PKI 目录和 `openssl`。可先运行：

~~~bash
neca config validate
neca doctor
~~~

完成 B4 后，二进制安装路径结束。

## 共用流程：升级、回滚与卸载（C1–C3）

**路径范围**：C1–C3；从升级开始，到卸载结束。以下流程适用于已完成的 A 或 B。

### C1. 升级

升级前备份：

~~~bash
umask 077
tar -C /opt/ipsec -czf - config pki clients downloads swanctl web-secrets \
  docker-compose.neca.yml .env neca.env \
  | gpg --symmetric --cipher-algo AES256 \
      --output /root/ipsec-backup-$(date +%Y%m%d-%H%M%S).tar.gz.gpg
~~~

GPG 口令不要写入命令、Git 或普通文档；恢复与更多细节见
[部署与日常运维](operations.md#备份与恢复)。

Docker 方式：更新 compose 中的镜像标签后执行：

~~~bash
docker pull ghcr.io/hyhdsl/neca:v1.5.2
docker pull ghcr.io/hyhdsl/neca-web:v1.5.2
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web up -d web
~~~

二进制方式：重新执行 B1–B3，替换 `/usr/local/bin/neca`。
修改 `readOnly` / `idleTimeout` 后发送 SIGHUP：

~~~bash
docker kill --signal=HUP neca-web
~~~

只执行 `up -d` 不会让已运行的 Web 进程重新读取配置；`web.listen` 等监听类配置仍需重启。

### C2. 回滚

1. 停止 Web/helper；
2. 恢复备份的 `pki`、`clients`、`swanctl`、`neca.json`、`web-secrets`；
3. 把 compose 镜像标签切回上一个版本，或恢复旧二进制；
4. 重新启动对应容器或执行 `neca --version` 验证。

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web stop web web-helper
~~~

lite 与 neca-web 必须使用同一版本；不要混用旧 CLI 与新 Web。

### C3. 卸载（共用流程结束）

停止并删除容器不会删除 `/opt/ipsec` 数据。确认不再需要 Web 时：

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write down
~~~

二进制安装的 CLI 可手动删除 `/usr/local/bin/neca`；不要执行会删除数据目录的命令，卸载前先备份。

## 深入阅读

- 部署、升级与回滚以本安装页为准；内部设计文档由维护者本地保存，不公开。
- [安全、权限与备份](security.md)
