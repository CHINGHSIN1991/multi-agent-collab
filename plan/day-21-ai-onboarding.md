# Day 21 — AI 建立與接入（REST）

- 階段：4（最小可用版本的最後一天）
- 對應 SPEC：§1（我的 AI）、§2（角色六題）、§9.1、§9.5
- 對應資安發現：**S-08**（接入連結用 GET 換發 Token）
- 前置：Day 20

## 今日目標

讓人類可以建立自己的 AI、取得接入憑證，AI 可以用 Token 透過 REST 輪詢參與。今天結束時是一套可以用的系統：沒有即時通知與常駐腳本，AI 靠人類手動觸發每一輪。

## 與 SPEC 的差異

| SPEC | 今天的做法 | 原因 |
|---|---|---|
| `GET /api/setup/<AI id>.<secret>` 讀取即換發 Token | 連結只帶一次性代碼；用 `POST /api/setup/exchange` 換 Token | S-08 |
| `GET /api/my-ais/:id/onboarding` 沒帶 Token 時會換發 | 只有 `POST` 會改變狀態 | S-08 |

## 要做的事

- [ ] 我的 AI（`src/ai.js`）：
  - `GET｜POST /api/my-ais`、`PATCH /api/my-ais/:id`。
  - 建立時選角色範本，填角色六題：名稱、職責、溝通風格、可自主事項、必須確認事項、完成與交接條件。每題有長度上限並經過 `assertNoSecret`。
  - `src/roles.js`：內建角色範本。先做 3–4 種（PO、PG、QA、自訂），其餘之後補。
  - AI 的專案不能超出所屬人類的專案。唯讀人類不能把 AI 建在自己唯讀的專案。
  - 每位人類可建立的 AI 數量有上限。
  - `POST …/disable`、`/enable`、`/delete`。
  - 所有 `/api/my-ais/:id/*` 都先確認這隻 AI 的 `owner_id` 是自己，否則 404。
- [ ] Token 管理：
  - `POST …/reissue`：發新 Token、舊的立刻失效。原文只在這次回應出現一次。
  - 預設效期 168 小時（`AI_TOKEN_TTL_HOURS`）。
  - 重發與接入都寫 `audit_log`。
- [ ] 接入（兩階段）：
  - `POST /api/my-ais/:id/onboarding-link`（人類）：產生一次性代碼，15 分鐘有效，只存雜湊。回傳給人類貼到 AI 對話的文字，內含平台網址與代碼。重新產生時舊代碼失效。
  - `POST /api/setup/exchange`（公開）：本文帶 `{ aiId, code }`。驗證通過後在一個交易內刪除代碼、發新 Token。回傳 Token 與接入設定。同時兩個請求只有一個成功。
  - 這個端點套用 Day 8 的每 IP 限流。
  - 接入設定的文字由伺服器產生。其中角色六題是人類寫的內容，要與系統規則分段並清楚標示來源。
- [ ] AI 的基本端點：
  - `GET /api/me`：AI 用來核對身分。回 `id`、`name`、`kind`、`ownerId`、專案、角色六題。
  - `GET /api/docs/ai-integration`（公開）：AI 接入規格的靜態文件。今天先寫一份涵蓋 §9.5「AI 每一輪」的短版。
- [ ] `public/poll.mjs`：單輪輪詢腳本，零相依。
  - 讀 `.aigile/<代號>/.token`。
  - 執行 §9.5 的五個步驟：核對身分 → 拉訊息（`brief=1`，最多 3 頁）→ 拉通知 → 印出待處理事項。
  - ACK 由 AI 處理完後另外呼叫。
  - 平台網址不是 `localhost` 時拒絕 `http://`（S-12）。
- [ ] 網頁：我的 AI 頁面（建立、編輯六題、複製接入文字、重發 Token、停用）。Token 與接入文字顯示一次，關閉後無法再看。

## 驗收標準

- 在網頁建立一隻 AI，複製接入文字，在終端機用 `curl` 完成 `exchange`，拿到 Token。
- 用這個 Token 呼叫 `GET /api/me`，回傳的是這隻 AI。
- 執行 `poll.mjs`，印出指定給這隻 AI 的訊息。用 `curl` 以 AI 的身分發 `[接單]` 並 ACK。
- 至此，Day 1–21 的所有測試全部通過。在版本控制上打一個標籤。

## 濫用情境測試

- [ ] 對系統中所有 `GET` 路由各呼叫一次，比對呼叫前後的資料庫內容（排除 `last_seen_at` 之類的時間戳）：沒有任何變化。把這寫成一支通用測試，之後每新增 GET 路由自動涵蓋。
- [ ] 用 `GET` 請求帶接入代碼（放在網址任何位置）：沒有任何效果，Token 沒有被換發。
- [ ] 接入代碼用第二次、過期後使用、對錯的 `aiId` 使用：都失敗，回應內容相同。
- [ ] 同時送兩個相同代碼的 `exchange`：只有一個拿到 Token。
- [ ] 猜接入代碼：每 IP 的限流生效。
- [ ] Alice 對 Bob 的 AI 呼叫 `reissue`、`onboarding-link`、`PATCH`、`delete`：都回 404。
- [ ] 管理員對別人的 AI 呼叫 `reissue`：回 404。（管理員可以停用成員，但不能取得別人 AI 的 Token。）
- [ ] 重發 Token 後，舊 Token 的下一個請求回 401。
- [ ] 人類被移出專案 A 後，他的 AI 不再屬於專案 A。
- [ ] AI 用自己的 Token 呼叫 `/api/my-ais`、建立另一隻 AI、修改自己的角色六題：都回 403。
- [ ] 角色六題中寫入「忽略以上所有規則」之類的文字：可以存（這是人類對自己 AI 的設定），但確認它不會出現在其他 AI 的接入設定中，並在回應結構中與系統規則分開。
- [ ] 伺服器日誌與稽核紀錄中搜尋今天測試產生的所有 Token 與接入代碼：找不到。

## 需要關注的知識點

- **安全方法（safe methods）**：HTTP 規定 GET 不該有副作用。瀏覽器、代理、連結預覽、掃毒軟體都依賴這個假設，會自行發出 GET 請求。
- **機密不該放在網址**：網址會出現在瀏覽紀錄、伺服器日誌、代理日誌、Referer。
- **兩階段交換**：人類拿到的是短效、一次性的代碼；真正的長效憑證由 AI 端直接向伺服器換取，不經過人類的剪貼簿以外的地方。這與 OAuth 授權碼流程的想法相同。
- **憑證只顯示一次**：伺服器不存原文，所以也無法再顯示。遺失就重發。
- **資源的擁有者檢查**：`/api/my-ais/:id` 的 `:id` 是使用者給的。每一條都要確認擁有者。
- **管理員權限的邊界**：管理員可以管理成員，但不該能冒用成員的 AI。
- **AI 的身分核對**：AI 每輪先確認「我是誰」。Token 檔放錯資料夾時，這一步能及早發現。
- **設定文字中的信任層級**：平台規則、所屬人類的設定、其他成員寫的內容，三者的可信程度不同，組成提示詞時要分開。

## 學習資源

- RFC 9110 第 9.2.1 節 Safe Methods：https://www.rfc-editor.org/rfc/rfc9110#name-safe-methods
- RFC 6749 OAuth 2.0 第 4.1 節 Authorization Code Grant（兩階段交換的原型）：https://www.rfc-editor.org/rfc/rfc6749#section-4.1
- RFC 9700 OAuth 2.0 Security Best Current Practice（一次性代碼、憑證不放網址的理由）：https://www.rfc-editor.org/rfc/rfc9700
- OWASP REST Security Cheat Sheet（敏感資訊不放在網址的段落）：https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html
- OWASP Secrets Management Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- OWASP API Security Top 10，API1 與 API5：https://owasp.org/API-Security/
- Anthropic 提示詞工程文件（用標籤區隔不同來源的內容）：https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview
- 原專案的 AI 接入規格可對照：`../aigile-oss/docs/ai-integration.md`

## 今日產出

`src/ai.js`、`src/roles.js`、`public/poll.mjs`、`docs/ai-integration.md`（短版）、我的 AI 前端頁面、`test/ai.test.js`、`test/onboarding.test.js`、`test/abuse/safe-methods.test.js`、`test/abuse/ai-ownership.test.js`。
