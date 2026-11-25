# Day 7 — 救援、改密碼、稽核紀錄

- 階段：1 認證
- 對應 SPEC：§5.2（`rescue`、改密碼的連帶效果）、§12（主機救援）
- 對應資安發現：**S-06**（管理員沒有第二因子）、**S-15**（沒有稽核紀錄）
- 前置：Day 6

## 今日目標

補完帳號復原的路徑，建立稽核紀錄模組，並替管理員加上 TOTP。

## 要做的事

- [ ] `src/audit.js`：
  - `audit(event, { actor, target, ip, result, detail })` 寫入 `audit_log`。
  - 表只允許 INSERT。用資料庫觸發器擋掉 UPDATE 與 DELETE。
  - `detail` 不得包含 Token、密碼、代碼的原文。寫一個檢查函式，內容像憑證就丟出錯誤。
  - `GET /api/admin/audit`（管理員）：可依事件類型、成員、時間範圍查詢，分頁。
  - 回頭把 Day 5、6 中直接寫表的地方改成呼叫這個模組。
- [ ] 事件清單（先定義常數，之後每天新增功能時補進來）：`login.ok`、`login.fail`、`device.new`、`device.logout`、`password.change`、`rescue.use`、`invite.create`、`invite.use`、`totp.enable`、`totp.disable`、`admin.transfer`、`member.disable`。
- [ ] 改密碼 `PUT /api/me/password`：
  - 要求舊密碼。
  - 成功後登出其他所有裝置、作廢未完成的配對。
- [ ] 主機救援：
  - `scripts/admin-rescue.js`：直接操作資料庫，產生一次性連結，15 分鐘有效，只給管理員。
  - `POST /api/auth/rescue`：成功後登出該帳號其他所有裝置，30 分鐘內可不用舊密碼設新密碼。
  - 救援登入會略過 TOTP（否則遺失驗證器就永遠進不來），所以要寫稽核並通知。
- [ ] TOTP：
  - `POST /api/me/totp/setup` 回密鑰與 `otpauth://` 網址；`POST /api/me/totp/enable` 驗證一組代碼後啟用。
  - 用 `node:crypto` 的 HMAC-SHA1 自己實作（約 30 行），接受前後各一個時間窗。
  - 同一組代碼不能用兩次（記下最後用過的時間窗）。
  - 管理員必須啟用才能做敏感操作。密碼登入成功後若已啟用 TOTP，要求第二步。
  - 產生 10 組一次性備用碼，只存雜湊。
- [ ] `requireRecentAuth(me, minutes)`：敏感操作要求最近 N 分鐘內驗證過密碼或 TOTP。今天先套用在改密碼與停用 TOTP。

## 驗收標準

- 改密碼後，其他裝置的下一個請求回 401。
- 執行救援腳本得到連結，用它登入並設新密碼。
- 用驗證器 App 掃描後能通過 TOTP 登入。
- 稽核紀錄查得到今天做的每一件事。

## 濫用情境測試

- [ ] 對 `audit_log` 執行 UPDATE 或 DELETE：被資料庫拒絕。
- [ ] 一般成員呼叫 `GET /api/admin/audit`：回 403。
- [ ] 稽核紀錄的 `detail` 中搜尋今天測試用過的所有密碼與 Token：找不到。
- [ ] 改密碼時不帶舊密碼、或帶錯的：失敗。
- [ ] 救援連結用第二次：失敗。對非管理員帳號產生救援連結：腳本拒絕。
- [ ] 同一組 TOTP 代碼在 30 秒內用兩次：第二次失敗。
- [ ] TOTP 連錯 5 次：進入延遲（沿用 Day 5 的機制）。
- [ ] 已啟用 TOTP 的帳號，只通過密碼那一步時拿到的憑證不能呼叫任何其他 API。
- [ ] 距上次驗證超過 N 分鐘時做敏感操作：回 403 `REAUTH_REQUIRED`。

## 需要關注的知識點

- **稽核紀錄與一般日誌的差別**：稽核紀錄回答「誰做了什麼」，要完整、不可改、可查詢。一般日誌是給除錯用的。
- **不可否認性**：STRIDE 的 R。沒有紀錄時，出事後無從證明任何事。
- **日誌裡不能有機密**：日誌的存取控制通常比資料庫寬鬆。
- **復原路徑是最弱的一環**：攻擊者不需要破解密碼，只要走得通復原流程。救援連結的安全性來自「需要主機存取權」。
- **TOTP 的運作**：共享密鑰加時間，算出 6 位數。它防的是密碼外洩，防不了即時釣魚。
- **重新驗證（step-up）**：登入狀態不等於可以做任何事。高風險操作要求重新證明身分。
- **改密碼為什麼要登出其他裝置**：使用者改密碼往往是因為懷疑帳號被盜。

## 學習資源

- OWASP Logging Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- OWASP Logging Vocabulary Cheat Sheet（事件命名可以直接參考）：https://cheatsheetseries.owasp.org/cheatsheets/Logging_Vocabulary_Cheat_Sheet.html
- OWASP Multifactor Authentication Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html
- OWASP Forgot Password Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- RFC 6238 TOTP：https://www.rfc-editor.org/rfc/rfc6238
- RFC 4226 HOTP（TOTP 建立在它上面，附錄有測試向量）：https://www.rfc-editor.org/rfc/rfc4226
- RFC 2104 HMAC：https://www.rfc-editor.org/rfc/rfc2104
- SQLite 觸發器：https://www.sqlite.org/lang_createtrigger.html
- OWASP Top 10 A09 Security Logging and Monitoring Failures：https://owasp.org/Top10/

## 今日產出

`src/audit.js`、`src/auth/totp.js`、`scripts/admin-rescue.js`、改密碼與救援流程、`test/audit.test.js`、`test/totp.test.js`、`test/abuse/recovery.test.js`。
