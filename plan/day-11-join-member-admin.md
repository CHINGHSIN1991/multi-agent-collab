# Day 11 — 加入流程、成員管理

- 階段：2 授權
- 對應 SPEC：§5.4、§12（群組設定、成員管理、轉交管理員）
- 對應資安發現：**S-01**（開放註冊預設開啟的部分）
- 前置：Day 10

## 今日目標

完成成員進出群組與專案的所有路徑：自己註冊、申請加入專案、停用與恢復、轉交管理員。

## 要做的事

- [ ] 群組設定：
  - `GET｜PUT /api/group/settings`（管理員）：群組名稱、開放註冊、群組密碼。
  - 群組密碼用 Day 5 的 `hashPassword` 存，不出現在任何回應與日誌。
  - `OPEN_JOIN`、`ROOM_CODE` 環境變數只當初始值。開放註冊的初始值預設為關閉。
  - `GET /api/group`（公開）只回群組名稱。
- [ ] 自己註冊 `POST /api/join`：
  - 關閉時回 403 `JOIN_CLOSED`。
  - 設了群組密碼而沒帶回 400 `ROOM_CODE_REQUIRED`；群組密碼錯誤的次數以 IP 計，錯 10 次等 10 分鐘。
  - 啟用 Turnstile 時一律要帶。
  - 名稱驗證：長度上限、去除控制字元與零寬字元、不得與現有成員重複（比對時先做 Unicode 正規化）。
  - 新成員沒有任何專案。
- [ ] 申請加入專案（`src/join.js`）：
  - `GET /api/join/projects`：只列出開放申請的專案。只回名稱，不回成員數與內容。
  - `POST /api/projects/:id/join-requests`、`GET /api/join-requests`、`POST …/approve`、`/reject`。
  - 審核者是該專案 leader；沒有 leader 時是管理員。核准時可指定角色與唯讀。
  - 每人同時最多 N 筆未處理的申請。
- [ ] 成員管理：
  - `POST /api/participants/:id/disable`、`/enable`（管理員）。
  - 停用在一個交易內完成：所有裝置登出、未用的配對與邀請作廢、名下 AI 的 Token 換成隨機值。
  - `POST /api/leave`：自己離開。管理員必須先轉交才能離開。
- [ ] 轉交管理員 `POST /api/admin/transfer`：
  - 要求近期重新驗證（Day 7 的 `requireRecentAuth`）。
  - 接手的人必須是 active 的人類。
  - 在一個交易內交換 `is_admin`。資料庫用部分唯一索引確保同時只有一位管理員。
- [ ] 以上每個動作都寫 `audit_log`。

## 驗收標準

- 預設狀態下 `POST /api/join` 回 403。
- 開放註冊後，新成員註冊、申請加入專案、leader 核准、看到專案內容，整條路徑走得通。
- 停用成員後，他與他的 AI 的所有請求都回 401；恢復後他需要重新登入。

## 濫用情境測試

- [ ] 同一個 IP 一小時內註冊第 21 次：回 429。
- [ ] 註冊名稱為 `<img src=x onerror=alert(1)>`：可以存（或被拒絕，二選一並寫明），但任何 API 回應都是原樣的 JSON 字串。Day 16 會測前端的顯示。
- [ ] 註冊名稱與管理員同名、或只差全形半形、或夾帶零寬字元：被拒絕。
- [ ] 剛註冊、沒有任何專案的成員：呼叫所有列表端點都拿到空的結果，拿不到任何專案內容。
- [ ] 一般成員核准自己的加入申請：回 403。
- [ ] 專案 A 的 leader 核准專案 B 的申請：回 403 或 404。
- [ ] 同一筆申請同時核准兩次：只生效一次。
- [ ] 不是管理員的人呼叫 `admin/transfer`：回 403。
- [ ] 轉交管理員後，原管理員立刻失去所有管理權限（用他的舊裝置 Token 呼叫管理端點回 403）。
- [ ] 資料庫層直接嘗試讓兩個人同時 `is_admin = 1`：被唯一索引擋下。
- [ ] 猜群組密碼錯 10 次：該 IP 被限制，其他 IP 不受影響。

## 需要關注的知識點

- **開放註冊是攻擊面的開關**：一旦開放，所有「已登入才能做」的事都變成「任何人都能做」。預設應該關閉。
- **同形字與正規化**：Unicode 中長得一樣的字元可以用來冒充他人。比對名稱前要正規化。
- **人機驗證的定位**：它提高自動化攻擊的成本，不是身分驗證。
- **撤銷要完整**：停用一個人，要同時處理他所有的憑證與他衍生出的身分（AI）。漏掉一項就留了後門。
- **資料庫層的不變量**：「只有一位管理員」這種規則，用唯一索引保證比用程式碼檢查可靠。
- **特權操作的交易性**：轉交管理員不能出現「兩個都是」或「兩個都不是」的中間狀態。
- **剛註冊的使用者能看到什麼**：這是最低權限的實際檢驗。

## 學習資源

- OWASP Authentication Cheat Sheet（註冊與使用者名稱的段落）：https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OWASP Input Validation Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html
- Unicode Technical Standard #39 Security Mechanisms（同形字問題的正式來源，讀第 4 節的概念即可）：https://www.unicode.org/reports/tr39/
- MDN `String.prototype.normalize()`：https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/normalize
- SQLite 部分索引：https://www.sqlite.org/partialindex.html
- Cloudflare Turnstile：https://developers.cloudflare.com/turnstile/
- OWASP Automated Threats to Web Applications（OAT-019 Account Creation）：https://owasp.org/www-project-automated-threats-to-web-applications/
- OWASP Top 10 A04 Insecure Design（「預設安全」的概念）：https://owasp.org/Top10/

## 今日產出

`src/group.js`、`src/join.js`、`src/members.js`、`test/join.test.js`、`test/members.test.js`、`test/abuse/join.test.js`。把新端點加入 Day 10 的隔離測試工具。
