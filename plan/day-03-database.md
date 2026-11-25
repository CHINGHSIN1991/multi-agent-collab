# Day 3 — 資料庫、升級機制

- 階段：0 基礎
- 對應 SPEC：§3.2（轉接層介面）、§3.3、§4
- 對應資安發現：無
- 前置：Day 2

## 今日目標

建好資料庫轉接層與升級機制，並寫出第一版 `schema.sql`。之後每天新增資料表都走同一套流程。

## 要做的事

- [ ] `src/db.js`：包裝 `node:sqlite`，只對外提供四個方法：`get`、`all`、`run`、`batch`。
  - 所有方法只接受「SQL 字串＋參數陣列」。不提供任何會把值拼進 SQL 的輔助函式。
  - `batch` 在一個交易內執行多個敘述，任何一個失敗就全部還原。
  - 開啟 WAL 模式與外鍵檢查（`PRAGMA journal_mode=WAL`、`PRAGMA foreign_keys=ON`）。
- [ ] `src/schema.sql`：先寫今天到 Day 11 會用到的表：`participants`、`devices`、`credentials`、`login_fails`、`invites`、`pair_codes`、`rescue_links`、`projects`、`project_events`、`join_requests`、`app_settings`、`audit_log`、`schema_migrations`。
  - `audit_log` 是 SPEC 沒有、檢核報告 S-15 建議新增的表。
  - 每個欄位寫清楚 `NOT NULL` 與 `CHECK`。例如 `participants.kind` 只能是 `human` 或 `ai`。
- [ ] `src/migrations.js`：
  - 全新資料庫：執行 `schema.sql`，把現有升級檔全部標為已套用。
  - 既有資料庫：依編號套用未套用的，每個檔一個交易。
  - 資料庫的版本比程式新，或升級失敗時，印出原因並結束程序。
- [ ] `migrations/README.md`：寫下規則（三位數編號、只增不改、同步更新 `schema.sql`）。
- [ ] 資料庫檔案建立時權限設為 `0600`，`DATA_DIR` 設為 `0700`。

## 驗收標準

- 空的 `DATA_DIR` 啟動後有 `switch.db`，且 `schema_migrations` 有紀錄。
- 手動加一個 `migrations/001_test.sql`，重啟後被套用一次；再重啟不會重複套用。
- 故意寫一個會失敗的升級檔：程序結束，資料庫內容與升級前相同。

## 濫用情境測試

- [ ] 把 `'; DROP TABLE participants; --` 當作參數值寫入再讀回：值原樣存入，資料表還在。
- [ ] `batch` 的第二個敘述失敗：第一個敘述的寫入不存在。
- [ ] 違反 `CHECK` 的寫入（例如 `kind = 'robot'`）：被資料庫拒絕。
- [ ] 違反外鍵的寫入（AI 的 `owner_id` 指向不存在的成員）：被拒絕。
- [ ] 在程式碼中搜尋樣板字串內含 `${` 的 SQL：應該找不到任何一處把變數拼進 SQL。把這個搜尋寫成一支測試。

## 需要關注的知識點

- **參數化查詢**：SQL 注入的根本解法。為什麼「先跳脫再拼接」不可靠。
- **交易與原子性**：哪些操作必須在同一個交易裡（例如「用掉邀請」與「建立成員」）。
- **WAL 模式**：讀寫並行的行為，以及備份時要注意什麼。
- **資料庫層的約束**：為什麼應用程式檢查過了，資料庫還要再檢查一次。
- **升級只往前**：為什麼已發布的升級檔不能修改。
- **SQLite 的型別親和性**：欄位宣告為 `INTEGER` 仍可能存入字串，所以 `CHECK` 有用。

## 學習資源

- Node.js `node:sqlite`：https://nodejs.org/api/sqlite.html
- SQLite WAL：https://www.sqlite.org/wal.html
- SQLite 交易：https://www.sqlite.org/lang_transaction.html
- SQLite 外鍵：https://www.sqlite.org/foreignkeys.html
- SQLite 型別：https://www.sqlite.org/datatype3.html
- OWASP SQL Injection Prevention Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html
- OWASP Query Parameterization Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet.html
- OWASP Database Security Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Database_Security_Cheat_Sheet.html

## 今日產出

`src/db.js`、`src/schema.sql`、`src/migrations.js`、`migrations/README.md`、`test/db.test.js`、`test/migrations.test.js`、`test/abuse/sql.test.js`。
