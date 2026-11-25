# Day 8 — 代理、HTTPS、安全標頭、限流

- 階段：1 認證
- 對應 SPEC：§14.1、§14.2、§5.2（每 IP 限流、Turnstile）
- 對應資安發現：**S-05**（CSP）、**S-11**（代理標頭）、**S-12**（HTTPS）、**S-13**（公開端點限流）
- 前置：Day 7

## 今日目標

把伺服器對外的那一圈做好：正確取得用戶端 IP、送出安全標頭、公開端點有限流。這些是認證機制能真正生效的前提。

## 要做的事

- [ ] 用戶端 IP：
  - 設定項改成 `TRUSTED_PROXY_HOPS`（整數，預設 0）。
  - 0 時只用連線本身的位址，忽略所有轉送標頭。
  - N 時把 `X-Forwarded-For` 以逗號切開，**從右邊數第 N 個**是用戶端 IP。
  - 不讀 `CF-Connecting-IP`，除非另外設了 `BEHIND_CLOUDFLARE=1`。原專案只要開了信任代理就優先採用這個標頭，站台不在 Cloudflare 後面時用戶端可以自己填。
  - 抽成一個純函式 `clientIp(socketAddr, xffHeader, hops)`，方便測試。
  - 連線位址用 `@hono/node-server/conninfo` 的 `getConnInfo(c).remote.address` 取得。我實測過：它回傳的是連線本身的位址，請求帶 `X-Forwarded-For` 或 `CF-Connecting-IP` 都不影響它。轉送標頭只由你自己的 `clientIp` 函式解讀。
  - 寫一個中介層把算出的用戶端 IP 放進 Context（`c.set('ip', …)`），限流與稽核都從這裡取，不各自讀標頭。
- [ ] 對外網址：
  - 設了 `APP_URL` 時，所有產生的連結都用它，不看 `Host` 標頭。
  - 沒設 `APP_URL` 且綁定的不是本機位址時，啟動時印警告。
- [ ] `src/security-headers.js`，寫成自己的中介層，不直接用 `hono/secure-headers` 的預設值。我實測過它的預設：不含 CSP、不論是否為 https 都送 HSTS、另外送出幾個這個專案用不到的舊標頭。這個專案的 CSP 要依設定動態組成（Turnstile、Trusted Types），自己寫比較清楚。要用 `secureHeaders` 也可以，但每個選項都明確指定，不依賴預設。
  - 頁面：`Content-Security-Policy`、`X-Content-Type-Options: nosniff`、`Referrer-Policy: no-referrer`、`Cross-Origin-Opener-Policy: same-origin`。
  - CSP 內容：`default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data: blob:; connect-src 'self'; object-src 'none'; base-uri 'none'; form-action 'self'; frame-ancestors 'none'; require-trusted-types-for 'script'`。
  - `script-src` 不放行任何 CDN 網域。QR Code 函式庫之後放在 `public/vendor/`。
  - 例外：設了 Turnstile 金鑰時，CSP 要加上 `script-src https://challenges.cloudflare.com` 與 `frame-src https://challenges.cloudflare.com`，否則驗證元件載不進來。沒設金鑰時不加。CSP 字串由設定動態組成。
  - `style-src` 不開 `'unsafe-inline'`。前端用的 SolidJS 與 Ark UI 是以 JavaScript 設定 `element.style`，不受這條限制；JSX 中不寫字串形式的 `style`。Day 15 會用瀏覽器實測。
  - Solid 用 `innerHTML` 建立靜態範本，所以 `require-trusted-types-for 'script'` 要搭配 `trusted-types solid-template`，只允許 Day 15 建立的那一個政策。今天先把這個指令加進 CSP。
  - `APP_URL` 是 https 時加 `Strict-Transport-Security: max-age=31536000`。
  - API 回應：`nosniff`、`Cache-Control: no-store`。
- [ ] 公開端點限流（`src/ratelimit.js` 擴充）：
  - §5.5 所列的每個公開端點都套用以 IP 為鍵的限流。
  - 註冊每 IP 每小時 `JOIN_LIMIT_PER_HOUR`。
  - 限流表的大小上限與清理沿用 Day 4 的實作。
- [ ] Turnstile（選配）：設了金鑰才啟用。驗證時由伺服器呼叫 Cloudflare 的驗證端點，逾時視為失敗。
- [ ] 沒有 CORS：不使用 `hono/cors`，不送任何 `Access-Control-*` 標頭。寫一支測試確認，包含對 `OPTIONS` 預檢請求的回應。
- [ ] 記下 `hono/csrf` 的存在：它檢查 `Origin` 標頭。Day 15 若決定把瀏覽器的登入憑證放 cookie，會用到它或同樣邏輯的自訂中介層。今天先不啟用。

## 驗收標準

- 用 Google 的 CSP Evaluator 檢查 CSP，沒有高風險項目。
- 瀏覽器開發者工具的主控台沒有 CSP 違規。
- `TRUSTED_PROXY_HOPS=1` 時，在 nginx 後面取得的 IP 是真實的用戶端 IP。

## 濫用情境測試

- [ ] `TRUSTED_PROXY_HOPS=0`，請求帶 `X-Forwarded-For: 1.2.3.4`：取得的 IP 是連線位址。
- [ ] `TRUSTED_PROXY_HOPS=1`，標頭為 `偽造值, 真實值`：取得的是真實值。每次請求換一個偽造值，登入失敗的限流仍然生效。
- [ ] `TRUSTED_PROXY_HOPS=1`、沒設 `BEHIND_CLOUDFLARE`，請求帶 `CF-Connecting-IP: 1.2.3.4`：被忽略。
- [ ] 帶偽造的 `Host` 標頭要求產生邀請連結：連結的網域是 `APP_URL`。
- [ ] 從另一個來源用 `fetch` 呼叫 API 並帶 `Authorization`：瀏覽器的預檢請求失敗（因為沒有 CORS 標頭）。
- [ ] 回應標頭中的 CSP 字串不含 `unsafe-inline`、`unsafe-eval`、`*`、任何 CDN 網域。沒設 Turnstile 時不含任何外部網域；設了之後只多出 `challenges.cloudflare.com` 一個。
- [ ] 未登入對 `/api/group` 連打 1000 次：超過上限後回 429。
- [ ] Turnstile 驗證端點逾時：登入請求被拒絕，而不是放行。

## 需要關注的知識點

- **`X-Forwarded-For` 該信哪一段**：這個標頭是每一層代理往右邊附加。最左邊是用戶端自己可以填的，只有「你自己的代理加上去的那幾段」可信。
- **Host 標頭注入**：用請求的 Host 組連結時，攻擊者可以讓系統寄出指向他網站的連結。
- **CSP 的威脅模型**：它是 XSS 的第二道防線，假設第一道（輸出跳脫）已經失守。
- **為什麼放行整個 CDN 等於沒設**：CDN 上有可被當成跳板的函式庫。要放行就用雜湊或 nonce 指定到單一檔案。
- **Trusted Types**：讓瀏覽器直接拒絕把字串塞進 `innerHTML` 這類危險的介面。
- **HSTS**：瀏覽器記住「這個站只走 HTTPS」。第一次連線仍有風險。
- **CORS 是放寬而不是限制**：沒有 CORS 標頭時，瀏覽器預設就不讓別的網站讀你的回應。
- **框架的安全預設不等於你的需求**：內建的安全標頭中介層是通用的起點。知道它送了什麼、沒送什麼，再決定用它還是自己寫。
- **失敗時拒絕（fail closed）**：外部驗證服務掛掉時，該放行還是該擋。

## 學習資源

- MDN `X-Forwarded-For`（「Security and privacy concerns」與「Selecting an IP address」兩節）：https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For
- MDN Content Security Policy：https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
- OWASP Content Security Policy Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html
- web.dev Strict CSP：https://web.dev/articles/strict-csp
- Google CSP Evaluator（工具）：https://csp-evaluator.withgoogle.com/
- web.dev Trusted Types：https://web.dev/articles/trusted-types
- OWASP HTTP Security Response Headers Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html
- MDN Strict-Transport-Security：https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security
- Hono 的 Secure Headers 中介層（了解它的選項與預設值）：https://hono.dev/docs/middleware/builtin/secure-headers
- Hono 的 ConnInfo 輔助函式：https://hono.dev/docs/helpers/conninfo
- Hono 的 CSRF 中介層：https://hono.dev/docs/middleware/builtin/csrf
- MDN CORS：https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- PortSwigger Host 標頭攻擊：https://portswigger.net/web-security/host-header
- Cloudflare Turnstile 伺服器端驗證：https://developers.cloudflare.com/turnstile/get-started/server-side-validation/
- OWASP Denial of Service Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html

## 今日產出

`src/security-headers.js`、`src/net.js`（`clientIp`）、`src/ratelimit.js` 擴充、`test/headers.test.js`、`test/abuse/proxy.test.js`、`test/abuse/public-endpoints.test.js`。
