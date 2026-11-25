# Day 24 — watcher 與等待腳本

- 階段：5 AI 端
- 對應 SPEC：§9.2、§9.3（watcher 的驗章與確認）、§9.4、§9.5
- 對應資安發現：**S-14**（本機 Token 檔的權限）、**S-12**（AI 端拒絕明文連線）
- 前置：Day 23

## 今日目標

寫出在成員電腦上常駐的兩支腳本：`watcher.mjs` 收門鈴，`aigile-wait.mjs` 讓 AI 的對話等到有事才繼續。今天寫的程式會在別人的電腦上執行，標準要比伺服器端更嚴。

## 要做的事

- [ ] `public/lib-local.mjs`（共用的本機工具，之後每支腳本都用）：
  - `readToken(agent)`：讀 `.aigile/<代號>/.token`。讀之前檢查檔案權限，POSIX 系統上若群組或其他人可讀就拒絕並提示修正。
  - `writeSecret(path, content)`：以 `0600` 建立檔案（`fs.writeFileSync(path, content, { mode: 0o600 })`），目錄 `0700`。先寫暫存檔再改名。
  - `assertGitIgnored()`：執行 `git check-ignore .aigile`，沒被忽略就拒絕繼續。不在 git 程式庫內則略過。
  - `assertSafeBase(url)`：不是 `localhost`／`127.0.0.1` 時必須是 `https:`。
  - `apiFetch(path, opts)`：帶 Token、設逾時、不跟隨跨網域的轉址、錯誤訊息中不印 Token。
  - 代號的格式驗證（`^[A-Z]+_[A-Za-z0-9]{4}$` 之類），避免代號被拿來組成任意路徑。
- [ ] `public/watcher.mjs`：
  - 啟動：讀設定、檢查權限、寫 PID 檔（已有同代號的 watcher 在跑就結束）。
  - 連 `GET /api/live`，標頭帶 `X-Aigile-Source: watcher`。斷線以遞增間隔加隨機抖動重連。
  - 收到 `doorbell`：
    1. 用門鈴金鑰驗簽章（`crypto.timingSafeEqual`）。
    2. 檢查時間在可接受的範圍內、隨機數沒見過（保留最近 N 個）。
    3. 驗章通過後仍呼叫 `GET /api/notifications/count?wake=1`，有待處理才叫醒。
  - 合併：收到門鈴後等 25 秒，期間的門鈴併成一次。
  - 節流：每小時與每天的叫醒上限（§9.4 的表），計數存本機狀態檔。超過就不叫醒，只記日誌。
  - 叫醒的方式：寫入門鈴檔 `.aigile/<代號>/bell.json`。今天不執行任何外部命令。
  - 每 15 分鐘巡檢一次（兼心跳），即使沒收到門鈴也檢查待處理數。
  - 叫醒後呼叫 `POST /api/me/wake`。
  - 日誌寫到 `.aigile/<代號>/watcher.log`，有大小上限與輪替。日誌中不寫 Token 與訊息內容。
- [ ] `public/aigile-wait.mjs`：
  - 盯門鈴檔。有新門鈴就印出簡短提示並結束（結束即代表要處理）。
  - 啟動時向平台登記 `sid`（`POST /api/me/online`）。被更新的 `sid` 取代時印 `SUPERSEDED` 結束，不叫醒對話。
  - 結束碼明確定義並寫進文件。
  - 輸出的文字是固定的，不含任何來自平台的內容。AI 要看內容得自己呼叫 API。
- [ ] 伺服器端：`POST /api/me/online`、`POST /api/me/wake`、`POST /api/me/wake-cap`、`GET /api/me/watcher-env`。叫醒次數以固定時區切日，團隊狀態顯示當日次數。
- [ ] 測試：用 Node 的子程序啟動腳本，對著測試伺服器跑。

## 驗收標準

- 啟動 watcher，從網頁指定訊息給這隻 AI，25 秒後門鈴檔被更新，等待腳本結束。
- `ls -l .aigile/*/` 顯示 `.token` 的權限是 `-rw-------`。
- 拔掉網路再接回，watcher 自動重連。

## 濫用情境測試

- [ ] `.token` 的權限改成 `0644`：腳本拒絕啟動並說明如何修正。
- [ ] `.aigile/` 沒有被 git 忽略：腳本拒絕啟動。
- [ ] 平台網址為 `http://example.com`：腳本拒絕連線。`http://localhost:3000`：允許。
- [ ] 伺服器回 302 轉到另一個網域：腳本不跟隨，Token 沒有被送到另一個網域。
- [ ] 送偽造簽章的門鈴、重播舊的門鈴、時間是一小時前的門鈴：都不叫醒，並記入日誌。
- [ ] 簽章正確但平台回報沒有待處理：不叫醒。
- [ ] 一分鐘內送 100 個有效門鈴：只叫醒一次。
- [ ] 超過每小時上限：不再叫醒，直到下一個小時。
- [ ] 代號參數為 `../../etc`、`PG_1234/../../x`：被格式驗證擋下，沒有讀寫 `.aigile/` 以外的路徑。
- [ ] 門鈴事件的資料中夾帶額外欄位（例如 `cmd`、`text`）：被忽略，不出現在門鈴檔與輸出中。
- [ ] 檢查日誌檔與所有標準輸出：搜尋 Token 與門鈴金鑰，找不到。
- [ ] 伺服器回 401：腳本停止重試並提示所屬人類重發 Token，不進入快速重試的迴圈。
- [ ] 伺服器回 429 或 503：依 `Retry-After` 等待。
- [ ] 腳本中搜尋 `exec`、`spawn`、`eval`、`Function(`：今天的兩支腳本中零筆。

## 需要關注的知識點

- **在別人的電腦上執行的程式**：伺服器端的錯誤影響你的服務；用戶端腳本的錯誤影響每一位使用者的電腦。攻擊面是那台電腦上的所有東西。
- **檔案權限與 umask**：`writeFileSync` 的預設模式是 `0666` 再套用 umask，在多數系統上結果是其他使用者可讀。機密檔要明確指定 `0600`。
- **原子寫入**：先寫暫存檔再改名，避免讀到寫到一半的檔案。
- **本機機密的去處**：檔案、環境變數、作業系統的金鑰圈，各自的暴露面。Token 進了環境變數會被子程序繼承。
- **不信任推送，再確認一次**：門鈴只是提示。真正決定要不要叫醒的是用自己的憑證向伺服器查到的結果。這讓偽造門鈴最多只能造成一次多餘的查詢。
- **重播攻擊**：合法的訊息被錄下來再送一次。用隨機數與時間窗擋。
- **伺服器的回應也是不可信輸入**：腳本不該因為伺服器回了什麼就執行命令或把內容原樣印給 AI。這是 Day 1 畫的「伺服器 → 成員電腦」那條邊界。
- **轉址與憑證**：跟隨轉址時把 `Authorization` 標頭帶到另一個網域，憑證就外洩了。
- **重連的退避與抖動**：伺服器重啟時，所有 watcher 同時重連會造成尖峰。
- **叫醒是有成本的**：每次叫醒都花費模型用量。節流既是成本控制，也限制了攻擊者能觸發的次數。

## 學習資源

- Node.js `fs` 的檔案模式：https://nodejs.org/api/fs.html#file-modes
- Node.js `crypto.timingSafeEqual`：https://nodejs.org/api/crypto.html
- Node.js `child_process`（測試時啟動腳本用）：https://nodejs.org/api/child_process.html
- Node.js 權限模型（`--permission`，可用來限制腳本只能讀寫 `.aigile/`）：https://nodejs.org/api/permissions.html
- MDN `fetch` 的 `redirect` 選項：https://developer.mozilla.org/en-US/docs/Web/API/RequestInit#redirect
- OWASP Secrets Management Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- OWASP OS Command Injection Defense Cheat Sheet（明天會用到，今天先讀）：https://cheatsheetseries.owasp.org/cheatsheets/OS_Command_Injection_Defense_Cheat_Sheet.html
- AWS 架構部落格〈Exponential Backoff And Jitter〉：https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
- git `check-ignore`：https://git-scm.com/docs/git-check-ignore
- CWE-732 Incorrect Permission Assignment for Critical Resource：https://cwe.mitre.org/data/definitions/732.html
- 原專案的實作可對照（注意它沒有設檔案權限）：`../aigile-oss/public/watcher.mjs`、`../aigile-oss/public/aigile-wait.mjs`

## 今日產出

`public/lib-local.mjs`、`public/watcher.mjs`、`public/aigile-wait.mjs`、伺服器端的 `online`／`wake`／`watcher-env` 端點、`test/watcher.test.js`、`test/wait.test.js`、`test/abuse/local-scripts.test.js`。
