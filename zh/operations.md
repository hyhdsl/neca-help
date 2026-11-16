# 部署与日常运维

## 日常检查

~~~bash
neca version
neca cert list
neca web status
docker compose -f /opt/ipsec/docker-compose.neca.yml ps
~~~

关注：

- Web 是否仍为 `readOnly=true`（除非已确认开启写能力）；
- helper 是否健康；
- 证书到期时间与 CRL 条数；
- `/opt/ipsec` 磁盘空间；
- 审计日志是否持续增长；
- `downloads` 是否积压。

## Web 日志

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web logs -f web
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write logs -f web-helper
~~~

日志可能包含 CN、路径和错误信息；不要公开粘贴包含密码、Cookie 或私钥的片段。

## 备份

至少备份：

~~~bash
tar -C /opt/ipsec -czf /root/ipsec-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  pki clients swanctl neca.json web-secrets
~~~

备份包含 CA 私钥与 Web 会话密钥，必须按最高敏感级别保护：

- 存放在加密磁盘或加密归档；
- 限制 root 读取；
- 不要上传到公开仓库、公开网盘或 CI artifact；
- 定期做恢复演练。

## 恢复

1. 停止 Web/helper；
2. 恢复备份中的 `pki`、`clients`、`swanctl`、`neca.json`、`web-secrets`；
3. 校验 `index.txt` 与 CRL 哈希；
4. 以只读模式启动 Web；
5. 验证登录、证书列表与下载；
6. 再决定是否开启写能力。

## 升级与回滚

固定版本标签，不使用 `latest` 作为生产依赖：

~~~bash
docker pull ghcr.io/hyhdsl/neca:v0.5.0
docker pull ghcr.io/hyhdsl/neca-web:v0.5.0
~~~

升级前备份；升级后验证：

~~~bash
neca version
neca cert list
neca web status
curl -fsS http://127.0.0.1:18080/healthz
~~~

`healthz` 只用于回环健康检查。回滚时恢复旧镜像标签与对应备份，不要混用版本。

## PKI 完整性

可在变更前后记录：

~~~bash
sha256sum /opt/ipsec/pki/ca/index.txt
sha256sum /opt/ipsec/swanctl/x509crl/ca.crl.pem
~~~

在变更前后记录哈希，用于确认操作可解释：

- `index.txt`
- CRL

哈希变化不一定代表故障；签发、吊销、重签都会改变它们。关键是变更可解释、
有审计、有备份。

## CRL 与 strongSwan

默认 `creds` 模式通过 `swanctl --load-creds` 热加载，不重启 strongSwan。除非
排障明确需要，不要随意重启 `ipsec-vpn` 容器；重启会中断现有连接。

~~~bash
docker exec ipsec-vpn swanctl --list-crls
docker logs ipsec-vpn --since 5m 2>&1 | grep -i crl
~~~

## 清理

~~~bash
neca pki downloads clean --expired
~~~

定期检查 `/opt/ipsec/downloads`。不要删除 `pki`、`clients` 或 `swanctl` 中的
文件来“清理空间”；先确认业务含义。

## 审计

默认审计日志 `/opt/ipsec/pki/audit.log`。建议：

- 只追加，不手工编辑；
- 定期归档并限制访问；
- 与系统日志/监控一起保留；
- 发生证书异常时，先保存日志再排障。

## 事故处理顺序

1. 停止进一步写操作（`neca web readonly on`，必要时停 Web）；
2. 保存日志、审计、当前 PKI 哈希与备份；
3. 判断是证书、CRL、容器还是权限问题；
4. 在测试环境复现；
5. 恢复服务后补做根因与审计记录。

不要在没有备份时执行 `delete`、`reissue` 或大范围 `downloads clean --all`。
