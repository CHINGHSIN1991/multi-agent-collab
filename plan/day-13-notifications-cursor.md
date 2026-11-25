# Day 13 — 通知、游標分頁、搜尋

- 階段：3 訊息
- 對應 SPEC：§7.4、§4（`notifications`）
- 對應資安發現：無直接對應
- 前置：Day 12

## 今日目標

做出通知的產生與確認、訊息流的游標分頁、以及搜尋。

## 要做的事

- [ ] 通知：
  - 發訊息時，在同一個交易內為每位收件人建立一則通知。一則訊息對一位收件人最多一則。
  - 欄位：`reasons`（`mention`、`reply`、`all`、`dm`）、`ask`（`true`／`false`／`null`）、`wake`（知會類為 false）、`status`（`pending`／`acked`）。
  - `GET /api/notifications`：回自己待處理的通知。沒有游標，ACK 之後再拉才有下一批。
  - `GET /api/notifications/count`：支援 `wake=1` 只算需要叫醒的。
  - `POST /api/notifications/ack`：帶通知 ID 陣列。只能 ACK 自己的。可同時存訊息游標。
- [ ] 訊息流 `GET /api/messages`：
  - 游標分頁。游標是不透明字串（`cur_` 開頭），內容是排序鍵，由伺服器簽章或加密，用戶端不能自己組。
  - `after` 只接受伺服器發出的游標。格式不對或簽章不符回 400。
  - `limit` 有上限（例如 100）。
  - `brief=1`：與自己無關的訊息只回前 80 字。
  - 查詢套用 Day 10 的 `scopeSql`。
- [ ] `GET /api/messages/:id/around`：回某則訊息前後各 N 則，同樣套用隔離。
- [ ] 搜尋 `GET /api/search/messages`：
  - 每人每分鐘 10 次（429 `SEARCH_LIMIT`）。
  - 只搜最近 5000 則。
  - 關鍵字長度有下限與上限。`LIKE` 的萬用字元（`%`、`_`）要跳脫。
  - 套用隔離。
- [ ] 索引：為 `notifications(recipient_id, status)`、`messages(project, created_at)` 建索引，用 `EXPLAIN QUERY PLAN` 確認查詢有用到。

## 驗收標準

- Alice 發訊息指定 Carol，Carol 的通知數加一；ACK 後歸零。
- 連續拉訊息流三頁，沒有重複也沒有遺漏，即使中間有新訊息寫入。
- 搜尋能找到自己專案的訊息。

## 濫用情境測試

- [ ] 把訊息流、`around`、搜尋、通知列表都加入隔離測試工具。Alice 搜尋專案 B 訊息裡的獨特字串：零筆結果，且總筆數也不洩漏。
- [ ] Alice ACK Bob 的通知 ID：沒有效果，回應不透露該 ID 是否存在。
- [ ] 自己組一個游標（改數字、改簽章、用別人的游標）：回 400。把游標當成 HTML 或腳本字串送進去：回 400，且不會被原樣存進任何地方。（原專案出過「特製游標在團隊狀態頁執行」的漏洞。）
- [ ] `limit=1000000`：被上限截斷。
- [ ] 搜尋關鍵字為 `%`、`_`、`%%%%%%%%`：當成一般字元處理，不會變成全表掃描或回傳全部。
- [ ] 搜尋第 11 次：回 429。
- [ ] `brief=1` 時，與 Alice 無關的訊息在回應中不超過 80 字。
- [ ] 訊息被指定給 Alice 後，Alice 被移出專案：通知還在但讀不到原文；確認不會因此洩漏內容，也不會造成 500。
- [ ] 對同一則通知同時 ACK 兩次：結果一致，沒有錯誤。

## 需要關注的知識點

- **游標分頁與位移分頁**：`OFFSET` 在資料變動時會重複或遺漏，而且越後面越慢。游標是「上次看到哪裡」。
- **不透明游標**：用戶端不該知道也不該能改游標的內容。伺服器要驗證游標是自己發的。
- **凡是用戶端送回來的東西都是輸入**：即使那個值是伺服器剛才發出去的。游標、ID、隱藏欄位都一樣。
- **搜尋是隔離最容易破的地方**：結果、筆數、建議詞、回應時間都可能洩漏。
- **`LIKE` 的萬用字元**：不是注入，但能讓查詢變得很貴或回傳超出預期的結果。
- **至少一次的通知傳遞**：處理完才 ACK，所以用戶端可能收到重複的通知，處理邏輯要能容忍。
- **查詢計畫**：沒有索引的查詢在資料少時看不出問題。昂貴的查詢也是阻斷服務的入口。

## 學習資源

- Slack 工程部落格對游標分頁的說明：https://slack.engineering/evolving-api-pagination-at-slack/
- 《Use The Index, Luke》分頁章節：https://use-the-index-luke.com/no-offset
- SQLite `EXPLAIN QUERY PLAN`：https://www.sqlite.org/eqp.html
- SQLite `LIKE` 與 `ESCAPE`：https://www.sqlite.org/lang_expr.html
- SQLite 查詢規劃器：https://www.sqlite.org/queryplanner.html
- Node.js `crypto.createHmac`（簽章游標用）：https://nodejs.org/api/crypto.html
- OWASP API Security Top 10，API4 Unrestricted Resource Consumption：https://owasp.org/API-Security/
- OWASP Denial of Service Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html

## 今日產出

`src/notifications.js`、`src/cursor.js`、`src/search.js`、`test/notifications.test.js`、`test/cursor.test.js`、`test/abuse/feed.test.js`。
