# 🖥️ Web 面板

## 定位

Web 面板是可选组件，默认**只读**，用于查看证书、验证 p12、查看下载与管理会话。
它不替代 strongSwan，也不默认暴露公网。

| 组件 | 角色 | 权限 |
|---|---|---|
| `necad` | Web 服务端 | 非 root；只读挂载配置、索引、meta、clients、downloads |
| `web-helper` | 写操作代理 | root；只接受 issue/revoke/unrevoke/reissue 白名单 |
| lite CLI | 管理命令 | root；`neca web ...` |

Web 容器没有 `ca.key`、客户端私钥或 `docker.sock`；helper 才挂载这些资产。

## 启用只读 Web

### 1. 配置

在 `/opt/ipsec/config/neca.json` 中：

~~~json
{
  "web": {
    "enabled": true,
    "listen": "127.0.0.1:18080",
    "readOnly": true,
    "idleTimeout": "30m",
    "adminCNs": [],
    "passkey": { "enabled": false, "rpId": "", "rpName": "neca Admin" }
  }
}
~~~

- `idleTimeout` 允许 5m–24h；绝对会话上限固定 24h。
- `readOnly` 默认 true。
- `adminCNs` 仅在反向代理 mTLS 场景使用。

### 2. 初始化 secrets

~~~bash
cd /opt/ipsec
NECA_IMAGE=ghcr.io/hyhdsl/neca:v1.1.0 ./init-web-secrets.sh
~~~

脚本以交互方式读取管理员密码并创建 `web-secrets/`、`session.key`、
`password.hash`、`web-audit.log`，并确保 `neca-web` 组 GID=10001。密码不回显，
也不写入 `neca.env`；请保存到密码管理器。若部署流程另外生成了初始密码文件，
首次登录后删除它。

### 3. 启动

~~~bash
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web up -d web
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web logs -f web
~~~

### 4. 访问

默认通过系统原生 OpenSSH 本地转发：

~~~bash
ssh -L 18080:127.0.0.1:18080 root@<SERVER>
# 浏览器打开 http://127.0.0.1:18080
~~~

不要把 `127.0.0.1:18080` 直接改成 `0.0.0.0`
暴露到公网；如确需反向代理，见下一节。

## 登录与会话

- 管理员密码以 bcrypt 哈希保存，不保存明文。
- 会话 Cookie 使用 HMAC 签名；HttpOnly、SameSite=Lax，HTTPS 下加 Secure。
- 空闲超时默认 30 分钟；登录后绝对上限 24 小时。
- 登录失败有限流；写操作强制 CSRF。
- 退出登录后会话失效；重启 necad 也会使内存会话失效（除非实现持久化）。

## 到期概览

登录后的概览页显示 30/60/90 天累计到期卡片；点击卡片进入证书页并应用对应
到期窗口。证书页可组合搜索、状态与到期窗口筛选。

概览与筛选都是只读操作：不会自动续期、吊销或修改证书。已吊销/未索引证书
不进入到期提醒；已过期证书单独计数。

## 写能力（可选）

默认只读。需要 Web 内签发/吊销/重签时：

~~~bash
cd /opt/ipsec
docker compose -f /opt/ipsec/docker-compose.neca.yml --profile web --profile web-write up -d web web-helper
neca web status
neca web readonly off
docker kill --signal=HUP neca-web
~~~

关闭写能力：

~~~bash
neca web readonly on
docker kill --signal=HUP neca-web
~~~

**重要**：`readOnly` / `idleTimeout` 修改后发送 SIGHUP 即可让运行中的 necad 重载：
`docker kill --signal=HUP neca-web`。`web.listen`、`adminCNs`、passkey 变更仍需
重启 web 容器。

Web 写操作包含 issue/revoke/unrevoke/reissue，以及 M11.1 的“移入回收站”；签发页平台使用勾选框（windows/linux/macos/android/ios），并可单独勾选“一键安装脚本”；默认不生成，仅 Windows/Linux/macOS 支持，由用户本次签发决定，不受全局 `NECA_INSTALLERS` 自动影响。设置页仅支持白名单高风险项（readOnly/idleTimeout/导出）；**不支持物理 purge 或 package**。
helper 的 `readOnly` 检查是第二道闸门，不能只依赖 UI 隐藏按钮。

## 管理命令

~~~bash
neca web status
neca web readonly status
neca web readonly on
neca web readonly off
neca web idle-timeout status
neca web idle-timeout 30m
neca web hash-password
neca web export on
neca web export off
neca web lock
neca web lockdown
~~~

- `web lock`：原子设置只读并递增 `web.sessionEpoch`，发送 SIGHUP；旧会话全部失效，服务保留。默认 Y/N 为 N；后台自动化可用 `--yes`。
- `web lockdown`：只停止明确命名的 `web` 与 `web-helper`，不触碰 SSH/strongSwan/VPN；默认 Y/N 为 N。
- `web export on/off`：默认关闭；只提供 issued/revoked CSV。关闭时 `/export/*.csv` 与 `/api/v1/export/*.csv` 返回 404。

侧栏“设置”可展开，下分为“通用”“安全”“关于”。通用页使用下拉框选择主题与语言；关于页展示版本、仓库、帮助、许可证与版权；右上角重复的下载中心按钮已移除。高风险设置默认只读。若后台在配置中设置 `web.allowHighRiskSettings=true`，且 Web/helper 可写，可在“安全”页用下拉框修改 readOnly、idleTimeout 与 CSV 导出；点击“保存设置”后，在确认对话框重新输入管理员密码、勾选 Y 并确认保存。未保存离开页面会触发浏览器原生提醒。把只读改为“是”会立即关闭 Web 写模式和高风险修改，页面随后显示只读状态；恢复写模式只能由后台 root 执行。`adminCNs`、passkey 与 helper 设置不支持 Web 修改。

`readonly off` 只允许 root，并检查：

1. helper 健康；
2. helper socket 权限；
3. `ca.key` 存在；
4. 配置原子写入成功。

任何一步失败都不应产生“半开启”状态。

## 反向代理与 mTLS（可选）

如果配置了 `web.adminCNs`，必须同时设置 `NECA_WEB_PROXY_SECRET`，由反向代理
转发：

~~~text
X-Neca-Proxy-Secret: <共享密钥>
X-Neca-Client-CN:   <已验证的管理员证书 CN>
~~~

necad 只接受来自回环代理的请求，并清除客户端伪造头。TLS 终止后设置
`NECA_WEB_SECURE_COOKIES=1` 并重启 web；OpenSSH 本地转发 HTTP 时保持未设置。
本次生产未启用公网域名/TLS/mTLS，保留为后续可选项。

## 语言与主题

- 页面支持 zh-CN / en-US 切换，使用 `neca_lang` Cookie。
- 默认语言跟随 `neca.json` 的 `language`。
- 支持浅色/深色主题，选择保存在浏览器 localStorage。
- 主题与语言切换位于“设置 → 通用”，不改变服务端权限或只读状态。

## 安全边界

- Web 容器非 root、只读根文件系统、`no-new-privileges`、`cap_drop: ALL`。
- 不挂载 `ca.key`、客户端私钥、`docker.sock`。
- helper 是唯一 root 写边界；拿到 helper 等价于拿到高权限，必须限制 socket 访问。
- helper 幂等缓存默认落盘到 `/run/neca/web-helper-cache.json`（0600，TTL 24h）；
  文件损坏时忽略并重建，不会阻塞写操作。
- 默认不发布端口；优先 SSH 隧道。
- Passkey 仅保留接口与配置开关，当前未注册登录路由。
- SIGHUP 支持 readOnly/idleTimeout 热重载；listen/adminCNs/passkey 变更需重启。

## 排障入口

- [故障排查](troubleshooting.md)
- [安全、权限与备份](security.md)
- 部署、升级与回滚见安装页。
