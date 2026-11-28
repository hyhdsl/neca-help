# GitHub 工作流程說明

本專案在 GitHub 上只有三個工作流程，分別負責品質閘門、Release 資產與容器映像。三者都由倉庫內的 YAML 定義，不在本機手動模擬發布。

## 1. `ci.yml`：品質閘門

- 觸發：推送到 `main`、`feat/**`、`fix/**`、`chore/**`，以及所有 Pull Request。
- 權限：唯讀倉庫內容（`contents: read`），不使用發布密鑰。
- 作業：`build-and-test`，以及依賴它的 `docker-smoke`。
- 檢查內容：`gofmt`、`go vet`、`govulncheck`、平台範本 ASCII、語言包 key 一致性、文件連結/圍欄/說明站結構、`go test -race ./...`、Linux amd64/arm64 交叉編譯、影子比對與 `version` 冒煙測試；接著建置 lite/web 映像並驗證入口。
- 失敗時不要合併；`main` 的分支保護要求 `build-and-test` 通過。

## 2. `release.yml`：GitHub Release

- 觸發：推送任意 `v*` 標籤。
- 預檢：標籤格式、tag 指向的 commit 是否位於 `main` 或 `release/x.y`，以及該 commit 的 `build-and-test` 是否成功。
- 發布 job：使用受保護的 `release-signing` environment；只有 `v*` 標籤可進入。
- 流程：檢出 → 建置 Linux amd64/arm64 的 `neca`、`necad`、`neca-web-helper` → 產生 `checksums.txt` → 在暫存 GNUPGHOME 中匯入發布私鑰 → 簽章並自驗 → 匯出公鑰 → 建立 Release。
- 資產：6 個二進位檔 + `checksums.txt` + `checksums.txt.asc` + `neca-gpg-key.asc`，共 9 個。
- 預發布規則：標籤名稱包含 `-`（例如 `v1.6.0-preview.1`）時自動標記 Pre-release；穩定標籤不標記。
- 注意：發布內容來自標籤指向的提交；不要在標籤建立後改程式碼再期待 Release 自動更新。驗證方式見 [GPG 驗證](gpg.md)。

## 3. `docker.yml`：GHCR 多架構映像

- 觸發：推送任意 `v*` 標籤，或手動 `workflow_dispatch`。
- 權限：`contents: read`、`packages: write`。
- 流程：建置二進位檔 → QEMU/Buildx → 登入 GHCR → 建置並推送兩個映像：`ghcr.io/hyhdsl/neca`（lite/CLI）與 `ghcr.io/hyhdsl/neca-web`（Web/helper）。
- 平台：`linux/amd64`、`linux/arm64`。
- 標籤：映像使用 Git 標籤；穩定標籤同時更新 `latest`，包含 `-` 的 preview 標籤不會更新 `latest`。

## 發布順序

1. 變更通過 PR 合入 `main`，並等待 `ci.yml` 的 `build-and-test` 成功。
2. 建立 annotated 標籤並推送，例如 `v1.5.2`。
3. 觀察 `release.yml` 與 `docker.yml` 均成功。
4. 核對 Release 的 9 個資產、checksums、簽章與 GHCR 映像標籤。
5. 部署與復原由另行批准的維運流程執行；本文不記錄任何特定伺服器資訊。

## 常見問題

- **只有 Release 成功、Docker 失敗**：檢查 GHCR 套件權限或 registry 登入；Release 資產不依賴 Docker。
- **preview 更新了 latest**：不要手動覆寫；檢查標籤是否包含 `-` 與 metadata 啟用條件。
- **CI 影子比對失敗**：先看是否為新舊輸出文字差異；若是預期變更，更新比對正規化規則，不要直接跳過閘門。
