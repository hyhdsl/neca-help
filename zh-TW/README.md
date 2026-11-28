# 🏠 neca 說明中心

> 適用於 **neca v1.6.0**。**Neko's Efficient Certificate Administrator（neca）** 是 strongSwan IKEv2 使用者憑證管理工具，並非 VPN 伺服器本身；它負責簽發、撤銷、重新簽發憑證，產生用戶端套件，並維護 CRL。

> **AI 揭露**：本專案的程式碼撰寫、除錯、測試、CI／建置／部署指令碼、說明文件與說明網站內容由 AI 完成；人類維護者負責需求、審查、功能驗收與正式環境操作。完整說明見 [AI-DISCLOSURE.md](https://github.com/hyhdsl/neca/blob/main/AI-DISCLOSURE.md)。AI 輸出可能有誤，請自行審查並在正式上線前驗證。

## 🎯 適用情境

- 單機或少數 strongSwan IKEv2 閘道，管理者希望用 CLI 維護憑證。
- 需要預設唯讀的 Web、清楚的權限邊界、稽核與 Docker 部署。
- 不需要多租戶、資料庫或複雜高可用叢集的部署。
- **不適合**：大規模 PKI／多租戶、無人值守自動簽發，或未經存取設計就直接暴露公網。

## 🚀 先看這裡

- 第一次使用：從 [快速開始](getting-started.md) 走一遍最小流程；常用指令見 [常用指令速查](cheatsheet.md)。
- 查全部子命令、參數與結束代碼：看 [子命令參考](commands.md)。
- 已有伺服器：看 [安裝、升級與復原](install.md) 和 [部署與日常維運](operations.md)。
- 驗證 Release：看 [GPG 驗證](gpg.md)，核對公鑰指紋後再執行二進位檔。
- 管理憑證：看 [憑證管理](certificates.md)。
- 使用 Web 面板：看 [Web 面板](web.md)。
- 需要自動化：看 [Web API（實驗性）](api.md)。
- 出問題：看 [疑難排解](troubleshooting.md) 和 [常見問題](faq.md)。
- English docs: <https://neca.nekobiglazyer.work/en/>

## 🧩 執行形態

| 形態 | 內容 | 適用情境 |
|---|---|---|
| lite（預設） | `neca` CLI；不含 Web 伺服器 | 只用命令列管理憑證 |
| neca-web | `necad` Web 伺服器；預設唯讀 | 用瀏覽器檢視憑證、下載與驗證 |
| neca-web + helper | 額外的 root helper；可明確開放寫入能力 | 需要在 Web 中簽發／撤銷／重新簽發 |

lite 與 neca-web 使用同一個版本標籤；Web 預設只監聽回送位址，不直接暴露公網。

## 🛡️ 重要安全提醒

1. `ca.key` 是最高敏感度資產，必須保持 `0600 root:root`；不要交給 Web 容器。
2. Web 預設 `readOnly=true`。開啟寫入能力前，先確認 helper 健康並理解 root 邊界。
3. 正式環境的寫入操作先加 `--dry-run`，並保留 PKI 備份。
4. 不要把真實伺服器位址、組織名稱、CA 名稱或任何私鑰提交到儲存庫。
5. 本專案以 **AGPL-3.0-only** 發布；二次開發與網路服務須履行原始碼公開義務。

## 📦 版本與來源

- 專案：<https://github.com/hyhdsl/neca>
- 正式版：<https://github.com/hyhdsl/neca/releases/tag/v1.6.0>
- 授權條款：<https://github.com/hyhdsl/neca/blob/main/LICENSE>
