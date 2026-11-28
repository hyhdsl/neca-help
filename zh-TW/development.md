# 開發、授權條款與回饋

## 原始碼與文件

- 主儲存庫：<https://github.com/hyhdsl/neca>
- 安裝、升級與復原：<https://neca.nekobiglazyer.work/zh-TW/#/install>
- 內部開發文件與評估紀錄由維護者本機保存，不在儲存庫中發布

## 本機建置

開發在 Docker 中完成：

~~~bash
docker build -f Dockerfile.dev -t neca-dev .
docker compose run -T --rm dev make all
docker compose run -T --rm dev make shadow
~~~

常用指令：

| 指令 | 作用 |
|---|---|
| `make all` | gofmt + vet + ASCII + 測試 + 交叉編譯 |
| `make vuln` | govulncheck 相依套件漏洞掃描（需連網） |
| `make shadow` | 舊 bash 與 neca 的影子比對 |
| `make check-locales` | zh-CN／zh-TW／en-US 語言包 key 一致性 |
| `make docker` | 建置 lite 與 neca-web 映像 |

## 架構

- `internal/core`：業務核心，不依賴展示層；
- `internal/cli`：參數解析、子命令與輸出；
- `internal/web`：Web 伺服器、工作階段、範本與 API；
- `internal/helper`：root helper 協定與允許清單；
- `cmd/neca`、`cmd/necad`、`cmd/neca-web-helper`：三個進入點；
- `internal/core/ports`：可替換介面（索引、設定來源）。

lite 與 neca-web 共用同一個 core 與資料目錄；不發布 full 映像。

## 發布流程

1. 合併至 `main` 並通過 CI；
2. 建立標籤 `vX.Y.Z`；
3. `release.yml` 產生二進位檔與 checksums，並由專用發布金鑰簽章；
4. `docker.yml` 推送 GHCR；
5. 在伺服器升級到固定版本標籤，並保留可復原的映像。

預覽標籤（`vX.Y.Z-preview.N`）不是穩定版；舊預覽只保留原始碼封存。

## 多語言說明文件

簡體中文、繁體中文與英文說明頁涵蓋同一組主題，均為手動維護；簡體中文是行為與發布說明的權威版本。語言切換在頁面與側邊欄。

## 貢獻與回饋

- 問題或變更請先透過 <https://github.com/hyhdsl/neca/issues> 說明；
- GitHub 三個工作流程見 [工作流程說明](github-workflows.md)，影子比對細節見 [影子比對](shadow-comparison.md)；
- PR 請附上測試與授權條款影響說明，由維護者審查後合併；
- 修改請先 fork，並遵守 AGPL 與 [二次開發要求](https://github.com/hyhdsl/neca/blob/main/docs/SECONDARY-DEVELOPMENT.md)；
- 安全問題請不要公開貼出私鑰、密碼或可利用細節，先聯絡維護者。

## 授權條款

**AGPL-3.0-only**。完整條文：

<https://github.com/hyhdsl/neca/blob/main/LICENSE>

本專案還包含 AI 揭露與免責聲明，見 `NOTICE` 與 README。
