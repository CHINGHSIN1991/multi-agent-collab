# Day 2 — HTTP 骨架（Hono）、錯誤格式、測試工具

- 階段：0 基礎
- 對應 SPEC：§3.1、§13（錯誤格式）、§14.1（請求大小上限、錯誤不讓程序結束）
- 對應資安發現：無
- 前置：Day 1
- 與 SPEC 的差異：SPEC 的伺服器不用框架、沒有任何相依套件。這裡改用 Hono，伺服器有兩個相依套件

## 今日目標

用 Hono 做出一個能接請求、回 JSON、出錯不會整個當掉的伺服器，並建好之後每天都要用的測試工具。

## 動手前先確認版本

2026-11-26 我實際安裝並測試過：

| 套件 | 版本 | 備註 |
|---|---|---|
| `hono` | 4.13.22 | 本身沒有其他相依套件 |
| `@hono/node-server` | 2.1.11 | 需要 Node 20 以上；本身沒有其他相依套件 |

兩個套件裝完，`node_modules` 裡只有這兩個。

## 要做的事

- [ ] 根目錄的 `package.json`：只列 `hono` 與 `@hono/node-server`，版本寫死（不用 `^`），提交 `package-lock.json`。`.npmrc` 設 `ignore-scripts=true`。之後一律用 `npm ci`。
- [ ] `src/errors.js`：
  - `ApiError(status, code, message, extra)`。所有錯誤回應格式固定為 `{ "error": { "code", "message" } }`。
  - `app.onError`：`ApiError` 轉成對應的狀態碼與格式；其他例外一律回 500 與固定訊息，細節只寫進伺服器日誌。
  - `app.notFound`：回 404 與同一個錯誤格式。Hono 預設的 404 是純文字，要換掉。
- [ ] `src/http.js`：`readJson(c)` 輔助函式。包住 `c.req.json()`，解析失敗時丟 `ApiError(400, 'INVALID_JSON')`。**我實測過：不合法的 JSON 會讓 `c.req.json()` 丟出例外，沒處理的話會落到 `onError` 變成 500。** 所有 handler 都用這個函式讀本文，不直接呼叫 `c.req.json()`。
- [ ] `src/config.js`：集中讀環境變數，啟動時驗證型別與範圍。數值不合法就拒絕啟動，不要默默用預設值。
- [ ] `src/app.js`：
  - `createApp(deps)` 建立並回傳一個 Hono 實例。資料庫等相依物件由外部傳入，方便測試替換。
  - 中介層的順序固定並寫成註解：請求大小上限 → 安全標頭（Day 8）→ 認證（Day 4）→ 限流（Day 4）→ handler。順序錯了會出漏洞，例如限流放在認證之前就沒有成員 ID 可以計數。
  - 請求大小上限：`hono/body-limit` 的 `bodyLimit({ maxSize, onError })`，套用在 `/api/*`。`onError` 回 413 與統一的錯誤格式。
  - 路由用 `app.get`、`app.post` 等註冊。各功能模組匯出 `register(app, deps)`，在這裡依序掛上。
  - 不使用 `hono/cors`。這個系統不需要跨來源存取（Day 8 說明原因）。
- [ ] `server.js`：
  - `serve({ fetch: app.fetch, port, hostname })`（`@hono/node-server`）。它回傳 Node 的 HTTP 伺服器物件，在上面設定 `headersTimeout`、`requestTimeout`。
  - 預設綁定 `127.0.0.1`。要對外提供時由設定明確指定。
  - 靜態檔用 `@hono/node-server/serve-static`，從兩個資料夾提供：`web/dist`（Day 15 建置出的網頁）與 `public/`（AI 端腳本）。今天 `web/dist` 還不存在，先放一個測試用的 `index.html`。
  - 沒有副檔名、不是 `/api/` 開頭的路徑回 `index.html`。
- [ ] `GET /api/health` 只回 `{ "ok": true }`（對應 S-01：不洩漏設定狀態）。
- [ ] `test/helpers.js`：建立使用記憶體資料庫的 app，提供 `call(method, path, { token, body })`。底下用 Hono 的 `app.request()`，不需要真的開埠。
- [ ] 另外寫 2–3 支會真的啟動伺服器的測試，專門測 `app.request()` 測不到的東西：逾時、大本文的串流行為、靜態檔。
- [ ] `package.json` 加 `"test": "node --test"`。
- [ ] 設定 CI（GitHub Actions）：`npm ci` 後跑測試。

## 驗收標準

- `npm test` 通過。
- `curl localhost:3000/api/health` 回 `{"ok":true}`。
- 不存在的路徑回 404 且格式符合錯誤格式（不是 Hono 預設的純文字）。
- `ls node_modules` 只有 `hono` 與 `@hono` 兩項。
- 你能指出中介層的執行順序，並說出每一層放在那個位置的理由。

## 濫用情境測試

- [ ] 送 30 MB 的本文：回 413。用真的啟動伺服器的測試確認伺服器記憶體用量沒有跟著漲 30 MB。
- [ ] 不帶 `Content-Length`、用分塊傳輸送超過上限的本文：同樣回 413。
- [ ] 送不合法的 JSON：回 400，不是 500。
- [ ] 送不合法的 `Host` 標頭：回 400，程序沒有結束。
- [ ] 請求 `/..%2f..%2fetc/passwd`、`/%2e%2e/server.js`、`/public/../server.js`：回 404，不回檔案內容。靜態檔的路徑處理交給了框架，這幾條測試就是在驗證它。
- [ ] 請求 `/package.json`、`/src/app.js`、`/data/switch.db`、`/.env`：回 404。只有 `web/dist` 與 `public/` 裡的檔案拿得到。
- [ ] handler 內丟出非 `ApiError` 的例外：回 500，回應內容不含堆疊或檔案路徑；下一個請求照常處理。
- [ ] 非同步的 handler 中丟出例外（`await` 之後才丟）：同樣回 500，程序沒有結束。
- [ ] 很慢地送標頭（一秒一個位元組）：連線在逾時後被關閉。
- [ ] 對同一條路徑用沒註冊的方法（例如對 `GET` 路由送 `DELETE`）：回 404 或 405，格式符合錯誤格式。

## 需要關注的知識點

- **Web 標準的 `Request`／`Response`**：Hono 建立在這兩個介面上。業務邏輯只依賴它們，之後換執行環境不用改。
- **Hono 的 Context**：`c.req` 讀請求，`c.json()` 回應，`c.set()`／`c.get()` 在中介層之間傳值（Day 4 用它傳遞已認證的成員）。
- **中介層的順序就是安全邊界的順序**：`app.use` 的註冊順序決定執行順序。寫成註解，並用測試固定下來。
- **框架的預設行為要逐一確認**：預設的 404 內容、JSON 解析失敗時的行為、沒設本文上限時的行為，都要自己測過，不要假設。今天的「不合法 JSON 變成 500」就是一個例子。
- **串流讀取與大小上限**：為什麼不能先把整個本文讀進記憶體再檢查長度；`Content-Length` 可以不帶或說謊。
- **路徑穿越**：把路徑處理交給框架不等於不用測。編碼過的 `..` 是經典的繞過方式。
- **錯誤訊息的兩個讀者**：給用戶端的要固定且不含內部細節，給維運的要完整。
- **慢速攻擊**：為什麼 HTTP 伺服器需要逾時設定。這是 Node 的 HTTP 伺服器層級的設定，框架不會替你設。
- **相依套件的取捨**：用框架省下路由與請求處理的程式碼，換來兩個要追蹤更新與安全公告的套件。鎖定版本、用 lockfile、關閉安裝腳本是基本的對策。
- **相依注入**：為什麼 `createApp` 要接收資料庫而不是自己建立。

## 學習資源

- Hono 文件（先讀 Getting Started 的 Node.js 一節與 Concepts）：https://hono.dev/docs/
- Hono 的 Node.js 轉接層：https://hono.dev/docs/getting-started/nodejs
- Hono 的 Context：https://hono.dev/docs/api/context
- Hono 的錯誤處理（`onError`、`notFound`、`HTTPException`）：https://hono.dev/docs/api/exception
- Hono 的 Body Limit 中介層：https://hono.dev/docs/middleware/builtin/body-limit
- Hono 的測試方式（`app.request()`）：https://hono.dev/docs/guides/testing
- Node.js `http` 模組（逾時設定）：https://nodejs.org/api/http.html
- Node.js 內建測試執行器：https://nodejs.org/api/test.html
- MDN Fetch API 的 Request：https://developer.mozilla.org/en-US/docs/Web/API/Request
- RFC 9110 HTTP Semantics（查狀態碼與方法語意用）：https://www.rfc-editor.org/rfc/rfc9110
- OWASP REST Security Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html
- OWASP Node.js Security Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Nodejs_Security_Cheat_Sheet.html
- OWASP Error Handling Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Error_Handling_Cheat_Sheet.html
- OWASP NPM Security Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/NPM_Security_Cheat_Sheet.html
- PortSwigger 路徑穿越：https://portswigger.net/web-security/file-path-traversal

## 今日產出

`package.json`、`package-lock.json`、`.npmrc`、`server.js`、`src/app.js`、`src/errors.js`、`src/http.js`、`src/config.js`、`test/helpers.js`、`test/http.test.js`、`test/server.test.js`（真的啟動伺服器）、`test/abuse/http.test.js`、CI 設定。
