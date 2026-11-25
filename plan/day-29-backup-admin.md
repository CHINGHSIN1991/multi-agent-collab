# Day 29 — 備份、管理功能

- 階段：6 收尾
- 對應 SPEC：§12
- 對應資安發現：**S-15**（稽核紀錄補完）、**S-17**（備份檔明文）
- 前置：Day 28

## 今日目標

完成管理員的工具：備份與匯入、專案的封存與刪除、示範資料、更新檢查。並回頭確認稽核紀錄涵蓋了所有該涵蓋的事件。

## 要做的事

- [ ] 備份（`src/backup.js`）：
  - `GET /api/admin/backup`、`/backup/status`、`/backup/rows`、`/backup/doc`：分批匯出成單一 JSON。
  - **不含**：Token 雜湊、密碼雜湊、TOTP 密鑰、門鈴金鑰、裝置、邀請、配對、救援連結、接入代碼、限流紀錄。把「匯出哪些表的哪些欄位」寫成明確的允許清單，新增的表預設不匯出。
  - 私訊是否匯出：做一個決定並寫明。建議匯出，但在備份畫面明確告知管理員備份內含私訊。
  - 加密（S-17）：匯出時要求管理員輸入通行碼。用 scrypt 從通行碼衍生金鑰，以 AES-256-GCM 加密。檔頭記錄演算法與參數。可選擇不加密，但畫面要顯示警告。
  - 要求近期重新驗證。寫 `audit_log`。
  - 備份檔帶版本號。
- [ ] 匯入：
  - `POST /api/admin/restore/begin`、`/rows`、`/doc`、`/finish`。
  - 只在空站可用（沒有任何訊息、卡片、其他成員）。在伺服器端檢查，並在整個匯入過程中持續成立。
  - 先預覽數量再確認。
  - 不能匯入較新版本的備份。
  - 匯入的每一列都經過與正常寫入相同的結構驗證：型別、長度、列舉值、外鍵。備份檔是不可信的輸入。
  - 整個匯入在一個交易內，任何錯誤全部還原。
  - 匯入者成為備份中的管理員。匯入的成員都沒有憑證，需要管理員重新發邀請。
- [ ] 專案管理的剩餘功能：
  - `POST /api/projects/:id/archive`、`/unarchive`。
  - `GET …/delete-preview`：列出會受影響的成員、卡片、訊息、文件數量。
  - `POST …/delete`：要求近期重新驗證，並要求在本文中帶專案名稱作為確認。只能刪已封存的專案。
  - `PUT …/repo`：程式庫連結。只接受 `https:`；拒絕含帳密、含 Token 樣式、含 `@` 的網址。
- [ ] 示範資料（`src/demo.js`）：一鍵載入虛構專案。`request_id` 以 `demo:` 開頭；清除時只刪示範部分。示範資料中的 AI 沒有 Token。
- [ ] 更新檢查（`src/update.js`）：
  - 管理員開管理頁時查詢最新版本，一天最多一次。`UPDATE_CHECK=0` 關閉。
  - 請求的網址是寫死的，不接受任何使用者輸入。
  - 設逾時；失敗時靜默。
  - 回應只取版本號並驗證格式，不把遠端回傳的任何文字直接顯示在頁面上。
  - 請求中不帶任何能識別這個站的資訊。
- [ ] 稽核紀錄盤點（S-15）：
  - 列出所有會改變權限、憑證、成員資格、設定的端點。
  - 寫一支測試：逐一呼叫這些端點，確認每次都有一筆對應的 `audit_log`。
  - 補上缺的事件：`ai.create`、`ai.token.reissue`、`ai.grants.change`、`skill.review`、`skill.allow`、`project.delete`、`backup.export`、`backup.restore`、`group.settings.change`、`member.readonly.change`、`leader.change`。
  - 管理頁加上稽核紀錄的檢視。
  - 稽核紀錄的保存期限與清理方式：做一個決定並寫明。

## 驗收標準

- 匯出加密的備份，在另一個全新的站用通行碼匯入，訊息、卡片、文件都在；所有成員都需要重新取得登入方式。
- 用文字編輯器打開加密的備份檔，看不到任何明文內容。
- 打開未加密的備份檔，搜尋 `scrypt$`、`dv_`、已知的 Token：找不到。
- 稽核盤點測試通過。

## 濫用情境測試

- [ ] 一般成員、leader、AI 呼叫任何 `/api/admin/*`：都回 403。
- [ ] 距上次驗證超過時限的管理員匯出備份：回 `REAUTH_REQUIRED`。
- [ ] 在已有資料的站呼叫 `restore/begin`：被拒絕。
- [ ] 匯入進行到一半時，另一個請求在站上建立了資料：匯入失敗並還原。
- [ ] 竄改過的備份檔：
  - 加密檔改掉一個位元組 → 解密失敗（GCM 的驗證標籤），沒有任何資料被寫入。
  - 未加密檔中把某位成員改成 `is_admin: true` → 做一個決定：匯入時只承認一位管理員，並在預覽畫面明確顯示誰會成為管理員。
  - 加入不存在的表、多餘的欄位、超長的字串、不合法的列舉值、指向不存在成員的外鍵 → 被拒絕，全部還原。
  - 訊息內文含 HTML 與腳本 → 匯入後在網頁上以純文字顯示。
  - 檔案項目的儲存路徑含 `..` → 被拒絕。
- [ ] 版本號高於目前程式的備份：被拒絕。
- [ ] 10 GB 的匯入請求：受 `MAX_BODY_MB` 限制，分批機制正常運作。
- [ ] 刪除專案時確認名稱不符：被拒絕。刪除未封存的專案：被拒絕。
- [ ] `repo` 設為 `javascript:alert(1)`、`file:///etc/passwd`、`https://user:pass@host/`、含 `ghp_` 開頭字串的網址：被拒絕。
- [ ] 模擬更新檢查的遠端回傳 `<script>` 或超長字串當版本號：被格式驗證擋下，頁面上不顯示。
- [ ] 清除示範資料後，非示範的資料一筆都沒少。
- [ ] 新增一個會改權限的端點而沒寫稽核：盤點測試失敗。

## 需要關注的知識點

- **備份是資料外流的常見途徑**：備份檔通常放在保護較弱的地方（下載資料夾、雲端硬碟、電子郵件）。
- **匯出用允許清單**：列出要匯出的，而不是列出不匯出的。否則新增的敏感欄位會自動被匯出。
- **還原是特權操作**：能還原備份的人可以改寫整個系統的狀態。
- **備份檔是不可信輸入**：即使是自己匯出的，也可能被修改過。匯入時要當成來自外部的資料處理。
- **驗證式加密**：AES-GCM 同時提供保密與完整性。只加密不驗證的模式，內容可以被竄改而不被發現。
- **從通行碼衍生金鑰**：通行碼的熵不足以直接當金鑰，要經過慢速的衍生函式。
- **破壞性操作的防呆**：預覽影響、要求輸入名稱、要求重新驗證、只能刪已封存的。每一層都讓誤操作與被劫持的工作階段更難造成損害。
- **伺服器端請求偽造（SSRF）**：伺服器依使用者提供的網址發請求時，攻擊者可以讓它連到內部服務。更新檢查的網址寫死就是為了避免這個問題。
- **對外請求的隱私**：更新檢查會讓第三方知道這個站的存在與版本。
- **稽核的完整性要用測試保證**：靠人記得在每個地方寫稽核，一定會漏。

## 學習資源

- OWASP Cryptographic Storage Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html
- Node.js `crypto.createCipheriv`（AES-256-GCM 的用法與 `getAuthTag`）：https://nodejs.org/api/crypto.html
- NIST SP 800-38D（GCM 的規格；重點是 IV 不能重複使用）：https://csrc.nist.gov/pubs/sp/800/38/d/final
- OWASP Deserialization Cheat Sheet（處理不可信的結構化資料的原則）：https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html
- OWASP Server Side Request Forgery Prevention Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- PortSwigger SSRF：https://portswigger.net/web-security/ssrf
- OWASP Logging Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- SQLite 備份 API 與 `VACUUM INTO`（另一種備份方式，了解差異）：https://www.sqlite.org/backup.html
- OWASP Top 10 A08 Software and Data Integrity Failures：https://owasp.org/Top10/
- 原專案的實作可對照：`../aigile-oss/src/backup.js`

## 今日產出

`src/backup.js`、`src/backup-crypto.js`、`src/demo.js`、`src/update.js`、專案封存與刪除、管理頁的前端、`test/backup.test.js`、`test/restore.test.js`、`test/audit-coverage.test.js`、`test/abuse/backup.test.js`。
