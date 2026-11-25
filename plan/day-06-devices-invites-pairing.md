# Day 6 — 裝置登入、邀請、配對

- 階段：1 認證
- 對應 SPEC：§5.1（裝置登入、登入碼）、§5.2（`invite`、`pair`、`code`）
- 對應資安發現：**S-06**（憑證沒有壽命上限）、**S-07**（配對碼可被全站作廢）
- 前置：Day 5

## 今日目標

完成人類的裝置登入管理，以及邀請連結與配對碼兩種登入方式。

## 與 SPEC 的差異

| SPEC | 今天的做法 | 原因 |
|---|---|---|
| 裝置登入使用中自動展延，無上限 | 閒置 30 天過期，另加 90 天絕對上限 | S-06 |
| 登入碼長期有效 | 不做登入碼 | S-06；邀請與配對已涵蓋同樣用途 |
| 新裝置輸入已登入裝置產生的 6 位數；全站錯 5 次作廢所有配對碼 | 流程反過來：新裝置顯示代碼，已登入裝置輸入確認 | S-07 |

## 要做的事

- [ ] `src/auth/devices.js`：
  - 資料表 `devices` 加 `created_at`（絕對上限用）與 `last_used_at`（閒置用）。
  - 裝置 Token 以 `dv_` 開頭，只存雜湊。
  - `authenticate` 中：超過閒置期限或絕對上限就回 401。`last_used_at` 最多每 60 秒寫一次。
  - `GET /api/devices`、`POST /api/devices/:id/logout`、`POST /api/auth/logout`。
- [ ] 邀請連結：
  - `POST /api/invites`（管理員）：可預帶專案與角色，效期 15 分鐘、1 小時、24 小時、7 天。
  - 邀請碼 32 位元組亂數，只存雜湊，一次性。
  - `POST /api/auth/invite/peek`：只回「有效與否、預帶的名稱」，不回專案名稱。
  - `POST /api/auth/invite`：在一個交易內「標記邀請已用」與「建立成員」。用 `UPDATE ... WHERE used_at IS NULL` 並檢查影響列數，確保同時兩個請求只有一個成功。
  - `POST /api/invites/:id/regenerate`、`/cancel`。
- [ ] 配對（反向流程）：
  - 新裝置 `POST /api/auth/pair/start`：伺服器產生一組配對 ID（32 位元組亂數，新裝置自己留著）與 8 碼顯示碼（去掉易混淆字元的英數，分成兩組顯示），5 分鐘有效。
  - `pair/start` 不需登入，所以要限制：每 IP 同時最多 3 筆未完成的配對，全站同時最多 200 筆，超過回 429。這是為了擋下面「預先佔位」的攻擊。
  - 已登入裝置 `POST /api/devices/pair/confirm`（需認證）：輸入顯示碼。伺服器先回傳該配對的發起資訊（瀏覽器、IP、時間），使用者再按一次確認才綁定。每位成員每 5 分鐘最多錯 5 次。
  - 新裝置 `POST /api/auth/pair/claim`：帶配對 ID 輪詢。已確認則發裝置登入，一次性。
- [ ] 所有新裝置登入都寫 `audit_log`，並寫一筆 `account_notices` 通知本人。

## 驗收標準

- 邀請連結能建立新成員並登入；第二次使用失敗。
- 兩台裝置完成配對，新裝置拿到自己的裝置登入。
- 裝置清單顯示兩台，可以從一台登出另一台。

## 濫用情境測試

- [ ] 同時送兩個相同邀請碼的請求：只建立一位成員。
- [ ] 過期、已取消、已用過的邀請：回應內容相同。
- [ ] 把裝置的 `created_at` 改成 91 天前：即使昨天才用過也回 401。
- [ ] 登出後用同一個裝置 Token：回 401。
- [ ] 未登入的人沒有任何端點可以猜顯示碼：`pair/confirm` 需要認證。
- [ ] **預先佔位攻擊**：攻擊者大量呼叫 `pair/start`，想讓受害者打錯一碼時剛好確認到攻擊者的配對。確認每 IP 與全站的上限生效；計算在上限 200 筆、8 碼顯示碼的條件下，打錯一次剛好命中的機率，寫進威脅模型。
- [ ] 已登入的攻擊者錯 5 次 `pair/confirm`：只有他自己被限制，其他人的配對不受影響。
- [ ] 攻擊者開了一個配對並誘騙受害者輸入代碼：確認畫面必須顯示新裝置的資訊（瀏覽器、IP 概略位置、時間），測試確認 API 有回這些欄位。把這個情境寫進 `docs/threat-model.md` 的已知剩餘風險。
- [ ] 沒帶正確配對 ID 的 `pair/claim`：回 404，不洩漏配對是否存在。

## 需要關注的知識點

- **閒置逾時與絕對逾時**：兩者防的事不同。只有滑動展延時，被偷的 Token 可以永遠有效。
- **一次性憑證的競爭條件**：「先查是否用過、再標記」之間有空隙。要用單一敘述完成檢查與標記。
- **短代碼的安全性來自哪裡**：6 位數只有一百萬種，它的安全性完全靠「次數限制＋短效期＋一次性」。
- **反向流程的新風險**：讓已登入的一方輸入代碼，擋掉了暴力猜測，但攻擊者可以改成大量開啟配對、等受害者打錯。所以代碼空間要夠大、同時存在的配對數要有上限。每換一種設計，都要重新問一次「攻擊者現在會怎麼做」。
- **誰來輸入代碼**：讓已登入的一方輸入，未登入的一方就沒有可以暴力嘗試的介面。這與電視登入串流服務的做法相同。
- **裝置授權流程的釣魚風險**：攻擊者自己發起流程，再騙受害者確認。防護靠確認畫面的資訊，無法完全消除。
- **撤銷**：無狀態的 Token 很難撤銷；存在資料庫的可以立刻失效。這是選擇不用 JWT 的理由之一。

## 學習資源

- OWASP Session Management Cheat Sheet（逾時的段落）：https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- RFC 8628 OAuth 2.0 Device Authorization Grant（今天的配對流程是它的簡化版，讀第 3 節與第 5 節的安全考量）：https://www.rfc-editor.org/rfc/rfc8628
- NIST SP 800-63B（重新驗證與 session 期限的段落）：https://pages.nist.gov/800-63-4/sp800-63b.html
- PortSwigger 競爭條件（「limit overrun」那一節正是邀請碼重複使用的情境）：https://portswigger.net/web-security/race-conditions
- OWASP Forgot Password Cheat Sheet（一次性連結的設計原則與邀請連結相同）：https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- SQLite `UPDATE` 與 `RETURNING`：https://www.sqlite.org/lang_returning.html

## 今日產出

`src/auth/devices.js`、`src/auth/login.js`（邀請與配對）、`test/devices.test.js`、`test/invites.test.js`、`test/pairing.test.js`、`test/abuse/session.test.js`。
