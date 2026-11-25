# Day 20 — 卡片通知、排程、團隊狀態

- 階段：4 卡片
- 對應 SPEC：§8.7、§8.2（自動封存）、§9.8、§1（要你處理）
- 對應資安發現：無直接對應
- 前置：Day 19

## 今日目標

把卡片與訊息系統接起來，做排程工作，並完成團隊狀態與卡片的網頁介面。

## 要做的事

- [ ] 卡片動作通知：
  - 建卡指定別人、改負責人或等誰、決定、暫緩、交接、退回、接手時，伺服器以動作者的身分代發 `[通知]`／`[交接]`／`[退回]` 訊息，並寫入 `card_msgs` 關聯到卡片。
  - `requestId` 由卡片 ID 與版本組成，重送不會重複發。
  - 卡片的修改與代發的訊息在同一個交易內。
  - 代發的訊息正文由伺服器用固定範本產生。範本中會帶入卡片標題等使用者寫的內容，這些內容仍是不可信的。
- [ ] 排程（`src/scheduler.js`）：
  - `server.js` 每 10 分鐘呼叫一次 `runScheduled()`。
  - 工作：已完成且已驗證滿 7 天的卡片自動封存（未驗證的不封存）；清除過期的邀請、配對、設定碼；清除過期的限流與登入失敗紀錄。
  - 每項工作各自 try／catch，一項失敗不影響其他。
  - 工作要可重入：同一項跑兩次結果相同。
  - 每次執行有處理筆數上限。
- [ ] 團隊狀態 `GET /api/team-status`：
  - 每位成員：在線與否、目前的工作、Token 到期時間、尚未回應的指派。
  - 目前工作的判定照 §9.8：有負責的未完成卡時取最急的一張（受阻 ＞ 待確認 ＞ 進行中 ＞ 待處理）；沒有卡片時依最新一則訊息的標籤。
  - 「可能斷線」「尚未回應」「快到期」的門檻照 §9.8，可用環境變數調整。
  - 快取 180 秒，寫入時清除。**快取的鍵值必須包含觀看者的可見範圍**，或是快取原始資料、每次回應時再依觀看者過濾。
  - 套用隔離：只列看得到的成員與卡片。
- [ ] 「要你處理」`GET /api/todo`（SPEC §1 的功能，§13 沒列端點，自行命名）：要回覆的通知、等驗收的卡、等決定的決策卡、待審核的加入申請。
- [ ] 網頁：
  - 卡片看板（依狀態分欄）、卡片詳情、交接與驗收的表單、決策卡的選項按鈕。
  - 團隊狀態頁、要你處理頁。
  - 沿用 Day 15 的規則與 Day 16 的元件包裝（不用 `innerHTML`、不寫字串 `style`、連結經過 `safeHref`）。看板的欄位切換用 `Tabs` 或 `Toggle Group`，卡片詳情用 `Dialog`，交接與驗收表單用 `Field`，決策選項用 `Radio Group`。這些都從 shadcn.io 複製進來，照 Day 16 的方式審查。
  - 更新節奏：卡片 30–90 秒、團隊狀態 60 秒。

## 驗收標準

- 交接一張卡，對方收到一則 `[交接]` 訊息與通知，訊息可連回卡片。
- 把一張已完成且已驗證的卡的時間改成 8 天前，執行排程後變為已封存；未驗證的不變。
- 團隊狀態頁顯示每個人正在做的卡。

## 濫用情境測試

- [ ] **快取洩漏**：Carol（兩個專案都在）先呼叫 `team-status`，接著 Alice（只在 A）呼叫：Alice 的回應中沒有專案 B 的任何成員、卡號、訊息 ID。反過來順序再測一次。（原專案在這裡洩漏過別專案的訊息 ID。）
- [ ] 把 `team-status`、`todo` 加入隔離測試工具。
- [ ] 卡片標題為 `[需確認] 問題1：...` 或開頭是另一個協定標籤：代發訊息的開頭標籤仍是伺服器決定的那一個，標題不會被解析成按鈕或影響狀態判定。
- [ ] 卡片標題含 HTML：在通知訊息、看板、團隊狀態頁都以純文字顯示（把這些位置加進 Day 16 的 XSS 測試陣列）。
- [ ] 同一個卡片動作重送：只有一則通知訊息。
- [ ] 卡片修改成功但代發訊息失敗（測試中模擬）：整個交易還原，卡片沒有變。
- [ ] 排程的某一項工作丟出例外：其他工作照常完成，程序沒有結束。
- [ ] 排程連續執行兩次：第二次沒有任何變更。
- [ ] 有 10 萬張符合封存條件的卡：一次排程只處理上限筆數，不會長時間佔住資料庫。
- [ ] 人類紀錄卡不出現在團隊狀態中任何人的「目前工作」。
- [ ] 唯讀成員不會被代發的通知指定為要回覆的對象。

## 需要關注的知識點

- **快取與授權**：快取的結果是「某個人看到的版本」。把它給另一個人，就繞過了授權。快取鍵要包含所有會影響結果的權限因素。
- **衍生資料繼承來源的敏感度**：團隊狀態、通知、摘要都是從訊息與卡片算出來的，隔離規則要跟著套用。
- **伺服器代發的內容仍然含有使用者輸入**：範本是可信的，填進去的值不是。
- **二次注入**：資料存入時無害，之後被另一段程式取出並放進不同的情境時才出事。卡片標題進入訊息正文就是這種情境。
- **交易邊界**：兩件事必須一起成功或一起失敗時，要放在同一個交易。
- **排程工作的冪等與可重入**：排程可能重複執行、可能中途失敗、可能兩個程序同時跑。
- **有界的工作量**：每次執行處理固定上限，避免一次處理太多資料而影響正常請求。

## 學習資源

- PortSwigger Web cache poisoning 與 cache deception（快取與授權互動的概念）：https://portswigger.net/web-security/web-cache-poisoning
- OWASP Multi-Tenant Security Cheat Sheet（快取隔離的段落）：https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html
- OWASP 對二次注入的說明（Testing for Stored XSS／SQL Injection 的相關段落）：https://owasp.org/www-project-web-security-testing-guide/
- Transactional outbox 模式（「改資料並發通知」的一般解法）：https://microservices.io/patterns/data/transactional-outbox.html
- Node.js Timers：https://nodejs.org/api/timers.html
- SQLite 交易：https://www.sqlite.org/lang_transaction.html
- Google SRE Book，Distributed Periodic Scheduling 一章（冪等與重入的概念）：https://sre.google/sre-book/distributed-periodic-scheduling/

## 今日產出

`src/card-notices.js`、`src/scheduler.js`、`src/team-status.js`、`src/todo.js`、卡片與團隊狀態的前端頁面、`test/card-notices.test.js`、`test/scheduler.test.js`、`test/team-status.test.js`、`test/abuse/cache.test.js`。
