# Day 17 — 卡片基本操作、樂觀鎖

- 階段：4 卡片
- 對應 SPEC：§8.1、§8.3、§4（`cards`、`card_log`）、§4 識別碼段落（卡號）
- 對應資安發現：無直接對應
- 前置：Day 16

## 今日目標

做出卡片的建立、讀取、列表、修改，以及專案內的連號卡號與樂觀鎖。狀態轉換的細節留到明天。

## 要做的事

- [ ] 資料表：`cards`（含 `version`、`proj_no`、決策欄位先建好留空）、`card_log`、`card_msgs`。
- [ ] 卡號：
  - 對外一律用 `<專案代號>-<連號>`，例如 `MAIN-12`。
  - 連號在建卡的同一個交易內取得。用一張 `project_counters` 表，`UPDATE ... SET n = n + 1 ... RETURNING n`。
  - 內部主鍵仍是不可猜的亂數 ID。路由同時接受兩種。
  - 卡片移到另一個專案時換新卡號，舊卡號記在對照表中仍可查到。
- [ ] `POST /api/cards`：
  - 類型：`工作`、`阻塞`、`決策`、`人類紀錄`。今天先做前兩種。
  - 欄位長度上限照 §8.1，集中定義成一個常數物件。
  - AI 建卡時 `ownerId` 只能是自己或留空（403 `ASSIGN_OTHERS`）。
  - 負責人、等誰必須是同專案成員（Day 10 的 `assertSameProject`）。
  - 所有自由文字欄位都經過 `assertNoSecret`。
- [ ] `GET /api/cards`：列表，套用 `scopeSql`。支援依狀態、負責人篩選，以及 `since` 增量查詢。
- [ ] `GET /api/cards/:id`：套用 `assertVisible`。
- [ ] `PATCH /api/cards/:id`：
  - 必須帶 `version`。用 `UPDATE ... WHERE id = ? AND version = ?` 並檢查影響列數；不符回 409 `CONFLICT`，回應附上目前的版本。
  - 可修改的欄位用白名單。
  - AI 只能改自己負責的卡（403 `NOT_CARD_OWNER`）；可認領「未指派、待整理或待處理、非決策」的卡。
  - 類型的限制：決策卡不能改類型，其他類型也不能改成決策（400 `TYPE_LOCKED`）。
  - 每次修改寫一筆 `card_log`（誰、何時、改了哪些欄位）。
- [ ] 在 `policy.js` 加入卡片相關的 `action`，並擴充 Day 9 的矩陣測試。

## 驗收標準

- 在同一個專案連續建三張卡，卡號是連續的三個數字。
- 兩個請求帶同一個 `version` 修改同一張卡：一個成功、一個回 409。
- AI 改別人的卡回 403；改自己的卡成功。

## 濫用情境測試

- [ ] 把卡片的列表、單筆、增量查詢、修改都加入隔離測試工具。特別測 `GET /api/cards?since=`：回應中不得出現別專案的卡號或任何提示。（原專案的漏洞就在這個增量查詢。）
- [ ] 同時建立 50 張卡：卡號沒有重複、沒有跳號。
- [ ] 用舊卡號別名、新卡號、內部 ID 存取別專案的卡：三種都回 404。
- [ ] 卡片移到專案 B 後，只在專案 A 的成員用舊卡號查：回 404，不洩漏新卡號。
- [ ] `PATCH` 時不帶 `version`：回 400，不是「無條件覆寫」。
- [ ] `PATCH` 本文夾帶 `proj_no`、`created_by`、`version`、`project`：被忽略或回 400，資料沒有被改。
- [ ] AI 建卡指定別人為負責人：回 403。
- [ ] AI 把別人的卡的負責人改成自己（藉此取得修改權）：回 403。只有符合「可認領」條件的卡才行。
- [ ] 唯讀成員建卡、改卡：回 403 `READ_ONLY`。
- [ ] 標題 101 字、原因 501 字：回 400，錯誤訊息指出是哪個欄位。
- [ ] 把有效的 Token 貼進卡片的任一文字欄位：被擋下。

## 需要關注的知識點

- **樂觀鎖**：不鎖定資源，而是在寫入時檢查「我讀的時候的版本還是現在的版本嗎」。適合衝突少的情境。
- **遺失更新**：兩個人各自讀取、修改、寫回，後寫的蓋掉先寫的。人與 AI 同時操作卡片時很常見。
- **檢查與動作要是同一個敘述**：`SELECT` 版本再 `UPDATE` 之間有空隙。要把版本條件放在 `UPDATE` 的 `WHERE` 裡。
- **連號的產生**：`MAX(n) + 1` 在並行時會重複。要用計數器加交易。
- **對外 ID 與內部 ID**：連號方便人使用，但可列舉。可列舉的 ID 更需要嚴格的授權檢查。
- **別名是另一條存取路徑**：每多一種查到同一個資源的方法，就多一個要套用授權檢查的地方。
- **欄位白名單**：見 Day 9 的批量賦值。卡片的欄位多，特別容易漏。

## 學習資源

- Martin Fowler《Patterns of Enterprise Application Architecture》Optimistic Offline Lock：https://martinfowler.com/eaaCatalog/optimisticOfflineLock.html
- MDN `ETag` 與 `If-Match`（HTTP 標準的樂觀鎖機制，了解概念）：https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-Match
- RFC 9110 第 13 節 Conditional Requests：https://www.rfc-editor.org/rfc/rfc9110#name-conditional-requests
- SQLite `RETURNING`：https://www.sqlite.org/lang_returning.html
- SQLite 的隔離性：https://www.sqlite.org/isolation.html
- Jepsen 對一致性模型的整理（了解「遺失更新」在各種隔離級別下的行為）：https://jepsen.io/consistency
- OWASP Mass Assignment Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html
- PortSwigger 競爭條件：https://portswigger.net/web-security/race-conditions

## 今日產出

`src/cards.js`（基本操作）、`src/card-numbers.js`、`test/cards.test.js`、`test/card-numbers.test.js`、`test/abuse/cards.test.js`。
