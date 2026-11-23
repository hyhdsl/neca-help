# 🧰 故障排查

先保存现场：日志、`neca web status`、`docker compose ps`、审计与最近变更。
不要在公开渠道粘贴密码、Cookie、私钥或完整配置。

## `neca: command not found`

安装包装命令：

~~~bash
install -m 0755 deploy/neca-wrapper.sh /usr/local/bin/neca
which neca
neca version
~~~

若在非标准目录，检查 `PATH` 与 wrapper 中的 compose 路径。

## Docker 不可用 / permission denied

- 确认 Docker Desktop 或 daemon 已启动；
- 确认执行用户能访问 `/var/run/docker.sock`；
- lite 容器通过 `docker.sock` 触发 `swanctl --load-creds`；
- 不要把 `docker.sock` 暴露给 Web 容器。

## Web 启动失败：address already in use

`18080` 被占用时，修改 `neca.json`：

~~~json
{ "web": { "listen": "127.0.0.1:18080" } }
~~~

然后重启：

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web restart web
~~~

默认使用 18080；若端口已被其他服务占用，改为其他回环端口并重启 `neca-web`。

## 修改了 readOnly，但页面仍显示旧状态

只执行 `docker compose up -d` 不会重载已存在的 Web 进程。发送 SIGHUP：

~~~bash
docker kill --signal=HUP neca-web
neca web status
~~~

`web.listen` 变更仍需重启 web 容器。

## 页面显示“helper 不可用”

检查：

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write ps
neca web status
ls -l /run/neca/web-helper.sock
~~~

常见原因：

- 没有启动 `web-write` profile；
- helper 容器未 healthy；
- socket 属主/组不是 `root:neca-web`；
- Web 容器 UID/GID 不是 10001；
- CA 私钥不存在或不可读。

## `readonly off` 被拒绝

该命令要求：

1. root 执行；
2. Web 组件已安装；
3. helper 健康；
4. socket 可访问；
5. `ca.key` 存在。

先看 `neca web status`，再查 helper 日志。不要用“手工改 JSON 为 false”绕过
检查；helper 仍会拒绝，且可能造成状态不一致。

## 登录失败 / 被限流

- 确认密码与部署时设置的初始密码（或初始密码文件）一致；
- 检查系统时间是否准确；
- 等待限流窗口结束，避免连续暴力尝试；
- 忘记密码时，重新生成 `password.hash` 并重启 web；
- 检查 `web-secrets` 权限是否为 `0710`、单文件 `0640`。

## Web 能启动但无法读取证书

检查 `neca.json`、`clients`、`downloads` 的组权限：

~~~bash
chgrp neca-web /opt/ipsec/config/neca.json
chmod 0640 /opt/ipsec/config/neca.json
chgrp -R neca-web /opt/ipsec/clients /opt/ipsec/downloads
chmod -R g+rX /opt/ipsec/clients /opt/ipsec/downloads
~~~

Web 只需要 `pki/ca/index.txt`、`pki/meta`、`clients`、`downloads` 和公开 CRL；
不要为了省事把整个 `pki` 挂给 Web。

## p12 验证失败

~~~bash
neca cert verify alice
~~~

常见原因：

- 使用了 `modern` profile，但旧客户端只支持 `compat`；
- p12 密码文件与归档不一致；
- p12 中混入了 CA 证书或链条数量不对；
- 客户端手工改过 p12。

重新生成：

~~~bash
neca cert package alice --p12-profile compat --zip
~~~

## 吊销后 CRL 未生效

确认重载模式：

~~~bash
neca config show
docker exec ipsec-vpn swanctl --load-creds
docker logs ipsec-vpn --since 2m 2>&1 | grep -i crl
~~~

默认 `creds` 模式应热加载；若配置为 `restart`，需要重启容器。不要在未确认的
情况下反复重启 strongSwan，以免中断现有连接。

## 下载链接 404 / 410

- 410/404 通常表示过期、已清理或 ID 不存在；
- 默认 TTL 30m；
- 重新执行 `neca cert package <CN> --zip`；
- 检查 `neca pki downloads list`。

## 一键安装脚本没有生成

默认 `--installers none`。显式开启：

~~~bash
neca cert issue alice --platforms windows,linux --installers windows,linux
~~~

Linux/macOS 脚本未覆盖所有发行版；先在测试机验证。

## 升级后 Web 行为异常

- 确认 lite 与 neca-web 使用同一版本标签；
- 检查是否只更新了镜像但没有重建/重启容器；
- 查看 `docker compose logs` 与 `neca web status`；
- 回滚时恢复旧镜像与备份，不要只回滚其中一个组件。

## Web 显示读取索引 permission denied

如果 Web 报 `/opt/ipsec/pki/ca/index.txt: open ...: permission denied`，通常是 CA
索引被重写后组变成了 root，而 Web 进程使用 UID/GID 10001（组 `neca-web`）：

~~~bash
chgrp neca-web /opt/ipsec/pki/ca /opt/ipsec/pki/ca/index.txt
chmod 2755 /opt/ipsec/pki/ca
chmod 0640 /opt/ipsec/pki/ca/index.txt
docker exec neca-web sh -c 'head -c 1 /opt/ipsec/pki/ca/index.txt >/dev/null && echo OK'
~~~

`pki/ca` 的 setgid 位确保以后 CLI 原子重写 `index.txt` 时新文件仍继承
`neca-web` 组。修正后刷新 Web 页面即可，不需要重启 Web 或 strongSwan；重新运行
`init-web-secrets.sh` 也会修正该权限。

## 仍然无法解决

收集以下信息后提交 Issue：

~~~bash
neca version
neca web status
docker compose -f /opt/ipsec/docker-compose.neca.yml ps
~~~

同时说明：版本、部署形态（lite/web/web-write）、复现步骤、已脱敏的错误信息。
