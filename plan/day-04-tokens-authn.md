# Day 4 — Token 與認證中介層

- 階段：1 認證
- 對應 SPEC：§2（成員、所屬人類）、§5.1、§14.2（每位成員的請求上限）
- 對應資安發現：無（今天照 SPEC 做；SPEC 這部分設計正確）
- 前置：Day 3

## 今日目標

做出「拿著 Token 的請求 → 認出是哪位成員」這一段，以及每位成員的請求頻率上限。今天還沒有登入流程，測試中直接在資料庫建成員與 Token。

## 要做的事

- [ ] `src/auth/tokens.js`：
  - `newToken()`：32 位元組的亂數（`crypto.getRandomValues`），轉成 base64url。
  - `hashToken(t)`：SHA-256。資料庫只存雜湊。
  - `randId(prefix)`：產生 `p_`、`msg_` 等識別碼。
- [ ] `src/auth/authenticate.js`，寫成 Hono 的中介層：
  - 認證通過後用 `c.set('me', 成員)` 把成員放進 Context，handler 用 `c.get('me')` 取得。不要放在模組層級的變數或全域物件上，那樣在並行的請求之間會互相覆蓋。
  - 用 `app.use('/api/*', authenticate)` 掛在所有 API 路由之前。中介層內部比對一份公開路徑清單，在清單內的才放行。
  - 只讀 `Authorization: Bearer <token>`。不讀網址參數，不讀 cookie。
  - 用雜湊查資料庫。查詢條件就是雜湊值本身，所以不需要另外做定時比對。
  - 成員狀態不是 `active` 回 401。
  - AI 的所屬人類不是 `active` 回 401。
  - AI 的 Token 過期回 401 `TOKEN_EXPIRED`。
  - 回傳成員物件，附上 `kind`、`authKind`。
- [ ] `src/ratelimit.js`：固定視窗計數器，鍵值為成員 ID，每分鐘 120 次，超過回 429 並帶 `Retry-After`。資料結構要有大小上限與過期清理（對應 S-13）。
- [ ] 公開路徑清單（`src/auth/open-routes.js`）：一個寫死的陣列，每項是「方法＋路徑樣式」，內容對應 SPEC §5.5。路由預設需要認證，只有在這份清單裡的不用。
- [ ] 用 `app.routes` 寫一支測試：列出所有已註冊的路由（我實測過，它會回傳每條路由的方法與路徑，中介層會以 `ALL /api/*` 的形式出現，要濾掉），對每一條不帶 Token 呼叫一次。回應不是 401 的路由集合，必須剛好等於公開路徑清單。之後任何人新增路由，這支測試都會自動涵蓋。
- [ ] 限流中介層接在認證之後（`c.get('me').id` 當鍵值）。
- [ ] `GET /api/me`：回自己的基本資料。
- [ ] `requireHuman(c)`、`requireAdmin(c)` 輔助函式，從 Context 取成員並檢查，不符就丟 `ApiError`。

## 驗收標準

- 帶有效 Token 呼叫 `GET /api/me` 回 200 與正確的成員。
- 資料庫的 `token` 欄位裡找不到任何 Token 原文。
- 第 121 次請求回 429，一分鐘後恢復。

## 濫用情境測試

- [ ] 沒帶標頭、標頭格式錯、Token 不存在：都回 401，且三者的回應內容相同（不讓人分辨是哪一種）。
- [ ] Token 放在 `?token=`：回 401。
- [ ] 停用的成員用原本的 Token：回 401。
- [ ] 所屬人類被停用後，他的 AI 用自己的 Token：回 401。
- [ ] 過期的 AI Token：回 401 `TOKEN_EXPIRED`。
- [ ] 新增一條路由但忘了想清楚是否公開：預設需要認證（測試中臨時註冊一條新路由，未帶 Token 呼叫回 401）。
- [ ] 對需要認證的路徑做變形：`/API/me`、`/api/me/`、`/api//me`、`/api/me%2f`：每一種要嘛回 401，要嘛回 404，不能回 200。
- [ ] 同時送出兩個不同成員的請求，其中一個 handler 故意延遲：兩個回應各自是自己的成員，沒有互換。
- [ ] 用 10 萬個不同的假成員 ID 觸發限流表：記憶體中的項目數不超過設定的上限。

## 需要關注的知識點

- **亂數的來源**：`Math.random()` 與密碼學安全亂數的差別。Token 有多少位元的熵才夠。
- **為什麼 Token 用 SHA-256 就好、密碼卻要慢雜湊**：差別在輸入的熵。256 位元的亂數無法被暴力猜測，人選的密碼可以。
- **Bearer Token 的性質**：持有即身分。所以它不能出現在網址、日誌、錯誤訊息裡。
- **預設拒絕**：路由預設需要認證，公開的要明確標示。反過來做（預設公開、需要的才加檢查）遲早會漏。用框架時特別容易犯這個錯：在某幾條路由上個別加 `authenticate`，新路由忘了加就是公開的。
- **中介層的比對範圍**：`app.use('/api/*', …)` 只涵蓋符合樣式的路徑。確認沒有任何 API 路由註冊在 `/api/` 以外，也確認路徑的大小寫、結尾斜線、重複斜線不會讓請求繞過中介層而仍然到達 handler。
- **每個請求的狀態放在 Context**：伺服器同時處理很多請求，放在共用變數上的「目前使用者」會被別的請求覆蓋，結果是用別人的身分執行操作。
- **錯誤回應不洩漏資訊**：「Token 不存在」與「Token 格式錯」回一樣的內容。
- **限流演算法**：固定視窗、滑動視窗、權杖桶的差別與各自的邊界行為。

## 學習資源

- RFC 6750 Bearer Token Usage：https://www.rfc-editor.org/rfc/rfc6750
- MDN `Crypto.getRandomValues()`：https://developer.mozilla.org/en-US/docs/Web/API/Crypto/getRandomValues
- MDN `SubtleCrypto.digest()`：https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/digest
- OWASP Authentication Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OWASP Session Management Cheat Sheet（Session ID 的熵與性質）：https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- OWASP API Security Top 10（API2 Broken Authentication、API4 Unrestricted Resource Consumption）：https://owasp.org/API-Security/
- Hono 的中介層寫法與執行順序：https://hono.dev/docs/guides/middleware
- Hono 的 Context（`c.set`／`c.get` 與型別）：https://hono.dev/docs/api/context
- Hono 的路由比對規則：https://hono.dev/docs/api/routing
- Cloudflare 對限流演算法的說明：https://blog.cloudflare.com/counting-things-a-lot-of-different-things/

## 今日產出

`src/auth/tokens.js`、`src/auth/authenticate.js`、`src/auth/open-routes.js`、`src/ratelimit.js`、`test/authn.test.js`、`test/abuse/authn.test.js`。
