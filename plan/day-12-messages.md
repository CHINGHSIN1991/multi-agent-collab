# Day 12 — 發文、冪等、機密檢查

- 階段：3 訊息
- 對應 SPEC：§7.1、§4（`messages`、`message_recipients`）
- 對應資安發現：無直接對應；今天的機密檢查是 S-04 的周邊防線
- 前置：Day 11

## 今日目標

做出發訊息與讀單則訊息。重點是冪等、收件人的規則、以及擋下貼進來的憑證。

## 要做的事

- [ ] 資料表：`messages`、`message_recipients`。`messages` 上建 `(author_id, request_id)` 唯一索引。
- [ ] `POST /api/messages`（`src/messages.js`）：
  - 驗證：`text` 必填、上限 2000 字；`requestId` 必填、限制長度與字元集；`project` 依 Day 10 的規則。
  - 編碼檢查：不合法的 UTF-8 或含控制字元回 400 `INVALID_ENCODING`。
  - 冪等：同作者同 `requestId` 且內容相同，回原本那則（200）；內容不同回 409 `IDEMPOTENCY_CONFLICT`。內容的比較用雜湊。
  - 收件人只來自三個來源：`recipientIds`、`audience: "all"`、回覆的原作者。正文中的 `@名字` 不產生收件人。
  - `replyFrom` 最多一位，必須在收件人之內。
  - 收件人必須是同專案的 active 成員；唯讀成員不能被指定（400 `READ_ONLY_RECIPIENT`）。
  - 回覆（`replyTo`）：原訊息必須是發文者看得到的；回覆不繼承原訊息的專案。
  - 訊息與收件人在同一個交易寫入。
- [ ] `src/secrets.js`：`assertNoSecret(text)`。
  - 平台自己的憑證：把文字中長得像 Token 的片段算雜湊，查資料庫是否為現行的 Token。是的話回 400 `SECRET_IN_TEXT`。
  - 接入連結的網址樣式。
  - 常見金鑰樣式：雲端服務金鑰、`-----BEGIN ... PRIVATE KEY-----`、GitHub 與 GitLab 的 Token 前綴。
  - 每則最多檢查 20 個候選片段。
  - 這個函式之後也用在卡片、私訊、工作記憶、技能。
- [ ] `GET /api/messages/:id`：套用 Day 10 的 `assertVisible`。
- [ ] 回應中的 `warnings` 陣列：不擋下、只提醒的事項放這裡。

## 驗收標準

- 發一則訊息，用回傳的 ID 讀回來內容一致。
- 用同一個 `requestId` 重送，資料庫中只有一則。
- 把一個有效的 AI Token 貼進正文，被擋下。

## 濫用情境測試

- [ ] 把訊息端點（發文、讀單則）加入 Day 10 的隔離測試工具。Alice 讀不到專案 B 的訊息。
- [ ] Alice 回覆專案 B 的訊息 ID：回 404，訊息沒有被寫進專案 B。（這是原專案出過的漏洞。）
- [ ] Alice 把 Bob 放進 `recipientIds`：回 400，且沒有產生任何紀錄。
- [ ] 同時送 20 個相同 `requestId` 的請求：只建立一則，其餘回相同的結果。
- [ ] 同一個 `requestId`、不同作者：互不影響。
- [ ] `text` 為 2001 字、空字串、只有空白、含 NUL 字元：都回 400。
- [ ] `recipientIds` 帶 1 萬個 ID：在處理前就被數量上限擋下。
- [ ] 用機密檢查當成探測工具：送一個隨機的 43 字元字串，回應不應與「不是 Token」有可區分的差異以外的資訊；確認一次請求最多檢查 20 個候選，無法用來大量驗證猜測。
- [ ] 把 Token 中間插入空白或換行再貼：記錄這種情況擋不住，寫進測試的註解與威脅模型。

## 需要關注的知識點

- **冪等**：網路不可靠，用戶端會重送。伺服器要能分辨「同一個請求再來一次」與「新的請求」。AI 用戶端特別容易重送。
- **冪等鍵的範圍**：鍵值要綁定作者，否則一個人可以用別人的鍵值造成衝突或讀到別人的結果。
- **唯一索引是冪等的基礎**：靠「先查再寫」會有競爭條件，要靠資料庫的唯一約束。
- **收件人由結構化欄位決定**：如果正文裡寫 `@某人` 就會通知，任何能影響正文的人（包含被注入的 AI）都能叫醒別的 AI。
- **機密掃描的定位**：這是最後一道防線，用樣式比對一定有漏網的。它不能取代「不要把機密放進對話」的習慣。
- **用雜湊比對而不存原文**：平台只有 Token 的雜湊，所以檢查時是把候選字串雜湊後比對。
- **驗證輸入的順序**：便宜的檢查（長度、數量）放前面，貴的檢查（查資料庫）放後面。

## 學習資源

- Stripe 的冪等請求說明（業界常被引用的設計）：https://docs.stripe.com/api/idempotent_requests
- IETF 草案 Idempotency-Key HTTP Header：https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- OWASP Input Validation Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html
- OWASP Secrets Management Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- GitHub secret scanning 支援的樣式（可參考有哪些常見金鑰格式）：https://docs.github.com/en/code-security/secret-scanning/introduction/supported-secret-scanning-patterns
- gitleaks 的規則檔（開源的金鑰樣式集合）：https://github.com/gitleaks/gitleaks
- MDN `TextDecoder`（`fatal: true` 用來驗證 UTF-8）：https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder
- SQLite `INSERT ... ON CONFLICT`：https://www.sqlite.org/lang_upsert.html

## 今日產出

`src/messages.js`、`src/secrets.js`、`test/messages.test.js`、`test/secrets.test.js`、`test/abuse/messages.test.js`。
