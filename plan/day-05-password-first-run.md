# Day 5 — 密碼登入、首次設定

- 階段：1 認證
- 對應 SPEC：§5.1（帳號密碼）、§5.2（`password`）、§5.3
- 對應資安發現：**S-01**（預設 admin／admin）、**S-07**（鎖定可被濫用）、**S-09**（PBKDF2 次數）
- 前置：Day 4

## 今日目標

做出帳號密碼登入與新站的第一位管理員建立流程。今天有三處刻意不照 SPEC 做，改用檢核報告建議的寫法。

## 與 SPEC 的差異

| SPEC | 今天的做法 | 原因 |
|---|---|---|
| PBKDF2-SHA256 100,000 次 | `crypto.scrypt`，參數寫進雜湊字串 | S-09 |
| 沒設 `ADMIN_PASSWORD` 時密碼是 `admin` | 不建立預設帳號；啟動時在終端機印一次性設定連結 | S-01 |
| 同帳號連錯 5 次鎖 15 分鐘 | 以「帳號＋IP」為單位遞增延遲 | S-07 |

## 要做的事

- [ ] `src/auth/password.js`：
  - `hashPassword(pw)`：scrypt，每個密碼一組 16 位元組的隨機鹽。輸出格式 `scrypt$N$r$p$鹽$雜湊`。
  - 參數用 OWASP 目前建議的 `N=2^17, r=8, p=1`（約佔 128 MiB 記憶體）。Node 的 `crypto.scrypt` 預設 `maxmem` 是 32 MiB，直接用這組參數會丟出 `ERR_CRYPTO_INVALID_SCRYPT_PARAMS`，要把 `maxmem` 設到 256 MiB。我在 Node 24 上實測確認過這個行為。
  - 每次雜湊佔 128 MiB，所以同時進行的雜湊數量要有上限（例如 4 個），其餘排隊。否則同時送幾十個登入請求就能用光記憶體。
  - `verifyPassword(pw, stored)`：依字串中的參數計算，用 `crypto.timingSafeEqual` 比對。
  - `needsRehash(stored)`：參數低於現行設定時回 true。登入成功後若為 true 就重新雜湊並存回。
  - 帳號規則：4–30 字，英數與 `. _ -`，存小寫。密碼 12–128 碼，比對一份常見密碼清單（放 1 萬筆即可）。
- [ ] `src/auth/login.js`：`POST /api/auth/password`。
  - 先寫入「這次嘗試」再驗密碼，避免同時送多個請求繞過計數。
  - 帳號不存在時也跑一次 scrypt（對一個固定的假雜湊），讓回應時間與帳號存在時相近。
  - 失敗訊息固定為「帳號或密碼錯誤」。
  - 延遲規則：同一組「帳號＋IP」第 3 次失敗起，每次失敗後的等待時間加倍，上限 15 分鐘。另設每 IP 的總失敗上限（`AUTH_FAIL_LIMIT`）。
  - 成功後發一組裝置登入（Day 6 才做完整的裝置管理，今天先能發能用）。
- [ ] 首次設定：
  - 啟動時若沒有任何人類成員，產生一次性設定碼，雜湊存資料庫，原文印在終端機，15 分鐘有效。
  - `POST /api/setup/first-admin`：帶設定碼、帳號、密碼，建立管理員。帳號不能是 `admin`。
  - 設了 `ADMIN_PASSWORD` 時仍可沿用 SPEC 的流程（建立 `admin` 帳號並強制首次改帳密），但密碼不得為 `admin` 或少於 12 碼，否則拒絕啟動。
  - 開放註冊預設關閉。
- [ ] 每次登入成功與失敗都寫入 `audit_log`（表已在 Day 3 建好，完整的稽核模組在 Day 7）。

## 驗收標準

- 全新資料庫啟動後，終端機印出設定連結；用它建立管理員後可以登入。
- 設定連結用過一次或過了 15 分鐘就失效。
- 資料庫的密碼欄位以 `scrypt$` 開頭，同一個密碼存兩次得到不同的字串。

## 濫用情境測試

- [ ] 用 `admin`／`admin` 登入全新的站：失敗（帳號不存在）。
- [ ] `GET /api/health` 的回應不含任何能判斷「是否已完成首次設定」的欄位。
- [ ] 沒有設定碼就呼叫 `first-admin`：回 403。已經有管理員後再呼叫：回 403。
- [ ] 同時送 50 個錯誤密碼的請求：計數為 50，不是 1。
- [ ] 從 IP A 對某帳號連錯 20 次後，該帳號的本人從 IP B 用正確密碼登入：成功。
- [ ] 存在與不存在的帳號，登入失敗的回應內容相同，回應時間的差距在可接受範圍內（測試取多次平均）。
- [ ] 送 1 MB 的密碼字串：在雜湊之前就被長度檢查擋下。
- [ ] 同時送 100 個登入請求：同時進行的雜湊不超過上限，程序的記憶體用量有界。
- [ ] 把資料庫裡的雜湊參數改低，登入成功後雜湊被自動更新成現行參數。

## 需要關注的知識點

- **慢雜湊為什麼要慢**：攻擊者拿到資料庫後離線猜密碼，每次嘗試的成本決定他能猜多少。
- **PBKDF2、scrypt、Argon2id 的差別**：運算成本與記憶體成本。為什麼記憶體成本對抗 GPU 有用。
- **鹽的作用**：防的是預先算好的對照表與「一次算出所有相同密碼的人」，不是防暴力破解本身。
- **雜湊格式要帶參數**：之後才能在不影響舊使用者的情況下調高。
- **帳號列舉**：從錯誤訊息、回應時間、鎖定行為都可能分辨帳號是否存在。
- **鎖定與阻斷服務的取捨**：硬鎖帳號保護了密碼，但把「讓你登不進來」的能力給了任何知道你帳號的人。
- **預設安全（fail closed）**：缺少設定時應該拒絕運作，而不是退回一個不安全的預設值。
- **先計數再驗證**：競爭條件如何繞過次數限制。

## 學習資源

- OWASP Password Storage Cheat Sheet（各演算法的建議參數在這裡，實作前先核對現行數字）：https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
- OWASP Authentication Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OWASP Credential Stuffing Prevention Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Credential_Stuffing_Prevention_Cheat_Sheet.html
- NIST SP 800-63B（密碼長度、禁止清單、不強制定期更換的依據）：https://pages.nist.gov/800-63-4/sp800-63b.html
- Node.js `crypto.scrypt` 與 `crypto.timingSafeEqual`：https://nodejs.org/api/crypto.html
- RFC 7914 scrypt：https://www.rfc-editor.org/rfc/rfc7914
- RFC 9106 Argon2（了解另一個選項）：https://www.rfc-editor.org/rfc/rfc9106
- PortSwigger 認證弱點（含帳號列舉與暴力破解的實作練習）：https://portswigger.net/web-security/authentication
- PortSwigger 競爭條件：https://portswigger.net/web-security/race-conditions

## 今日產出

`src/auth/password.js`、`src/auth/login.js`（密碼部分）、首次設定流程、`test/password.test.js`、`test/first-run.test.js`、`test/abuse/login.test.js`。
