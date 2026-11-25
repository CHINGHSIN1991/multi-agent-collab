# Day 30 — Docker、部署、CI

- 階段：6 收尾
- 對應 SPEC：§14.3、§14.5
- 對應資安發現：**S-12**（沒有要求 HTTPS）
- 前置：Day 29

## 今日目標

把系統包裝成可以部署的形式，寫出安全的預設部署設定，並讓 CI 自動執行所有測試與檢查。

## 要做的事

- [ ] `Dockerfile`：
  - 基底映像釘住版本與摘要（`node:22-alpine@sha256:...`），不用浮動的標籤。
  - 多階段建置：第一階段在 `web/` 執行 `npm ci` 與 `npm run build`；第二階段在根目錄執行 `npm ci --omit=dev`，複製伺服器程式碼與 `web/dist`。最終映像的 `node_modules` 只有 `hono` 與 `@hono/node-server`；沒有 `web/node_modules`、前端原始碼、測試檔、`.git`、`.env`。
  - 以非 root 使用者執行（`USER node`）。
  - 資料放在 `/data` volume。
  - `HEALTHCHECK` 打 `/api/health`。
  - `.dockerignore`：排除 `data/`、`.env*`、`.git`、`test/`、`.aigile/`、簽章私鑰。
- [ ] `docker-compose.yml`（S-12）：
  - 預設包含一個反向代理（Caddy，會自動取得憑證），應用程式的埠不對外公開，只在內部網路。
  - 另附 `docker-compose.local.yml` 給本機開發，只綁定 `127.0.0.1`。
  - 容器設定：`read_only: true`（資料 volume 除外）、`cap_drop: [ALL]`、`security_opt: [no-new-privileges:true]`、記憶體與 CPU 上限。
  - 機密用環境變數檔或 Docker secrets，不寫在 compose 檔裡。
- [ ] 啟動時的檢查（`src/config.js`）：
  - 綁定的位址不是本機，且 `APP_URL` 不是 `https://` 開頭 → 印出醒目的警告；設 `REQUIRE_HTTPS=1` 時直接拒絕啟動。
  - `TRUSTED_PROXY_HOPS` 大於 0 而 `APP_URL` 沒設 → 警告。
  - 列印生效的安全相關設定（不含機密的值）。
- [ ] 正常關機：收到 `SIGTERM` 時停止接受新連線、關閉即時通道連線、等進行中的請求完成、關閉資料庫。設最長等待時間。
- [ ] 日誌：
  - 結構化（每行一個 JSON）。
  - 請求日誌記錄方法、路徑、狀態碼、耗時、成員 ID。**不記**：`Authorization` 標頭、請求本文、查詢字串中的值。
  - 寫一支測試：跑一遍主要流程，在日誌輸出中搜尋所有用過的憑證。
- [ ] CI（`.github/workflows/ci.yml`）：
  - 在 Linux、macOS、Windows 各跑一次測試。
  - 伺服器端的工作：根目錄 `npm ci` 後跑測試，並加一個步驟檢查根目錄 `package.json` 的 `dependencies` 只有 `hono` 與 `@hono/node-server`。多了任何一個就失敗，逼自己在加相依套件時停下來想。
  - 伺服器與前端各跑一次 `npm audit --omit=dev`。
  - 前端的工作：`npm ci`（不是 `npm install`）、`npm run lint`、`npm run build`、`npm audit --omit=dev`。lockfile 與 `package.json` 不一致時 `npm ci` 會失敗，這是刻意的。
  - 設定 Dependabot 或 Renovate 追蹤前端相依的更新；每次更新都要看變更內容，不自動合併。
  - 瀏覽器測試（Playwright）放在另一個工作。
  - 建置 Docker 映像並用掃描工具檢查（Trivy 或 Docker Scout）。
  - 機密掃描（gitleaks）。
  - 工作流程的安全：動作釘住完整的 commit SHA；`permissions` 設為最小（預設 `contents: read`）；不在 `pull_request_target` 中執行來自 fork 的程式碼。
- [ ] 發行流程（`.github/workflows/release.yml`）：
  - `v*` 標籤觸發。標籤必須與 `package.json` 的版本一致。
  - 建置 amd64 與 arm64 映像。
  - 產生建置來源證明（GitHub 的 artifact attestation）。
  - Day 25 的腳本簽章不在 CI 上做（私鑰不該在 CI 上）。在發行檢查清單中列為手動步驟。
- [ ] 文件：
  - `SECURITY.md`：回報漏洞的方式、部署檢查清單、設計上的安全假設、已知的剩餘風險（從 `docs/threat-model.md` 整理）。
  - `README.md` 的快速開始：預設的指令就是安全的那一種。不安全的捷徑要另外標明。
  - 環境變數全表。
- [ ] 資料庫備份的營運面：寫一份說明，用 SQLite 的線上備份方式定期備份 `switch.db`，並測試還原。

## 驗收標準

- `docker compose up` 後，用 https 連得上；直接連應用程式的埠連不上。
- `docker run --rm <映像> id` 顯示的不是 root。
- 在全新的機器上只裝 Node 22，放入已建置的 `web/dist`，根目錄執行 `npm ci --omit=dev`，`node server.js` 能啟動並提供網頁。
- CI 全綠。
- 送 `SIGTERM` 後，進行中的請求完成，程序在時限內結束。

## 濫用情境測試

- [ ] 設定不安全的組合（對外位址加 `http://` 的 `APP_URL`、`REQUIRE_HTTPS=1`）：拒絕啟動。
- [ ] 不設 `ADMIN_PASSWORD`、不設任何環境變數直接啟動：沒有可用的預設帳密，開放註冊是關的（回歸 Day 5 與 Day 11）。
- [ ] 映像內容檢查：`docker run --rm <映像> ls -la /app` 中沒有 `.env`、`data/`、`test/`、私鑰檔。寫成 CI 的一個步驟。
- [ ] 容器內嘗試寫入 `/app`：因唯讀檔案系統而失敗。
- [ ] 日誌中搜尋測試用過的密碼、Token、接入代碼、TOTP 密鑰：找不到。
- [ ] 在一個測試分支故意 commit 一個假的金鑰樣式：CI 的機密掃描失敗。
- [ ] 工作流程檔中搜尋以標籤（`@v4`）而非 SHA 引用的動作：列出並修正。
- [ ] 版本號在 `package.json` 與程式中的版本常數不一致：測試失敗。
- [ ] 刪掉 `web/node_modules` 後跑伺服器端的測試：全部通過（伺服器不依賴任何前端套件）。
- [ ] 最終映像的 `node_modules` 中只有 `hono` 與 `@hono` 兩項。
- [ ] 在根目錄 `package.json` 多加一個相依套件：CI 的檢查失敗。
- [ ] 在 `web/package.json` 把某個套件改成浮動版本（`^`）而不更新 lockfile：CI 失敗。

## 需要關注的知識點

- **預設安全的部署**：使用者會照著 README 的第一段指令做。那段指令的結果必須是安全的。把安全的做法放在「進階設定」裡，等於大多數人不會做。
- **TLS 終止**：應用程式不處理 TLS，由前面的代理處理。應用程式要知道自己在代理後面（Day 8 的設定）。
- **容器的最小權限**：非 root、唯讀檔案系統、丟掉 capabilities。容器不是安全邊界，但這些設定讓被入侵後能做的事變少。
- **釘住版本**：浮動的標籤（`latest`、`22-alpine`）今天與明天可能是不同的內容。摘要才是不變的。
- **少相依的代價與好處**：伺服器只有兩個第三方套件，要追蹤的漏洞與供應鏈風險就只有這兩個；代價是框架沒做的部分要自己實作並自己負責正確性。每加一個套件都是一次取捨，不是免費的。
- **建置時相依與執行時相依**：前端的 160 多個套件只在建置時存在，不進入伺服器的執行環境。但建置出來的 JavaScript 會在每位使用者的瀏覽器中執行，所以建置環境被入侵等於所有使用者被入侵。CI 上的 `npm ci`、lockfile、關閉安裝腳本都是為了這件事。
- **CI 是高價值目標**：CI 能存取程式碼、機密、發行管道。工作流程的權限要最小化。
- **釘住 Actions 的 SHA**：標籤可以被移動到另一個 commit。
- **機密不進日誌**：日誌會被送到各種地方，存取控制通常比資料庫寬鬆。
- **正常關機**：直接結束程序可能讓交易中斷、連線被重置。
- **來源證明**：讓使用者能驗證映像確實是從這個程式庫的這個 commit 建置的。
- **備份要測試還原**：沒測過還原的備份不算備份。

## 學習資源

- OWASP Docker Security Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html
- Docker 多階段建置：https://docs.docker.com/build/building/multi-stage/
- OWASP NPM Security Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/NPM_Security_Cheat_Sheet.html
- Docker 建置最佳實務：https://docs.docker.com/build/building/best-practices/
- Docker Compose 檔案參考（`read_only`、`cap_drop`、`security_opt`）：https://docs.docker.com/reference/compose-file/services/
- Node.js 官方的 Docker 最佳實務：https://github.com/nodejs/docker-node/blob/main/docs/BestPractices.md
- Caddy 自動 HTTPS：https://caddyserver.com/docs/automatic-https
- Mozilla SSL Configuration Generator（若改用 nginx）：https://ssl-config.mozilla.org/
- OWASP Transport Layer Security Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Security_Cheat_Sheet.html
- GitHub Actions 安全強化指南：https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions
- OWASP CI/CD Security Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/CI_CD_Security_Cheat_Sheet.html
- GitHub artifact attestations：https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations
- Trivy：https://trivy.dev/
- gitleaks：https://github.com/gitleaks/gitleaks
- The Twelve-Factor App（設定、日誌、可拋棄性三章）：https://12factor.net/
- Node.js `process` 的訊號事件：https://nodejs.org/api/process.html#signal-events
- SQLite 線上備份：https://www.sqlite.org/backup.html

## 今日產出

`Dockerfile`、`.dockerignore`、`docker-compose.yml`、`docker-compose.local.yml`、`Caddyfile`、`.github/workflows/ci.yml`、`.github/workflows/release.yml`、`SECURITY.md`、`README.md`、`docs/operations.md`、`test/config.test.js`、`test/abuse/logs.test.js`。
