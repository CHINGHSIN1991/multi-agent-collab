# Day 16 — 前端應用：路由、訊息頁面、shadcn.io 元件

- 階段：3 訊息
- 對應 SPEC：§3.1（前端）、§5.2（登入方式）、§7.3、§7.7、§13（網頁路徑）、§14.4
- 對應資安發現：**S-05**（延續 Day 15，今天用完整的頁面驗證防護有效）
- 前置：[Day 15](day-15-frontend-foundation.md)

## 今日目標

在昨天的地基上做出真正可用的網頁：完整的登入流程、訊息流、發文、通知。同時把需要的 shadcn.io 元件加進來並理解它們底下的 Ark UI，並用一整套 XSS 測試確認「使用者寫的內容不會被當成 HTML 或腳本」在每個顯示位置都成立。

今天不改昨天定的任何安全規則。若某個功能在嚴格 CSP 下做不出來，先找不違規的做法，而不是放寬 CSP。

## 要做的事

### 上午：加入元件與應用骨架

- [ ] 讀 Ark UI 的入門（約 30 分鐘）。shadcn.io 的元件只是 Ark UI 加上 Tailwind 樣式，行為出問題時要回到 Ark UI 的文件查。
- [ ] 從 shadcn.io 把今天要用的元件複製進 `web/src/components/ui/`：`Button`、`Input`、`Field`、`Tabs`、`Dialog`、`Toast`、`Dropdown Menu`、`Avatar`、`Tooltip`、`Input OTP`。
- [ ] 每複製一個就做三件事：讀完它的原始碼；跑 `npm run lint`；在正式建置的版本上開一次，確認主控台沒有 CSP 與 Trusted Types 違規。有違規的元件先不用，記下來。
- [ ] shadcn.io 沒有的元件直接用 Ark UI，照同樣的方式用 Tailwind 寫樣式。QR Code 用 Ark UI 的 `QrCode`（先確認 shadcn.io 有沒有對應的元件）。
- [ ] 圖示用 `lucide-solid`，只匯入用到的圖示，不要整包匯入。
- [ ] 全域狀態：目前使用者與目前專案用 context 加 store。
- [ ] 資料讀取用 `createResource`，搭配 `Suspense` 與 `ErrorBoundary`。API 錯誤統一用 `Toast` 顯示。
- [ ] 路由（`@solidjs/router`）：`/p/:slug/:page`、`/cross/:page`、`/my-ai`、`/members`、`/invite/:code`。未登入時導向登入頁。各頁面用 `lazy()` 分開載入。
- [ ] i18n：介面文字以 key 查字典（一個 context 加兩份字典檔）。協定標籤、卡片狀態值不翻譯。

### 下午：頁面

- [ ] 登入頁完整版：用 `Tabs` 切換密碼、邀請、配對三種方式。
  - 配對：新裝置這一側顯示代碼與 QR Code（不需要 CDN）。已登入裝置輸入代碼的欄位用 `Input OTP`。
  - 啟用 TOTP 的帳號，密碼通過後進入第二步。
  - 錯誤訊息用 `Field` 的錯誤文字顯示。
- [ ] 邀請連結的處理：已登入時打開別人給的邀請或登入連結，用 `Dialog` 先詢問，不直接切換帳號。
- [ ] 訊息流：
  - 以游標分頁往上載入更早的訊息。
  - 每則訊息顯示作者（`Avatar` 加名稱）、時間、協定標籤、內文。
  - 內文中的網址若要變成連結，一律經過 `safeHref`，外部連結加 `rel="noopener noreferrer"`。
  - 清單用 `For`，以訊息 ID 為鍵。
- [ ] `[需確認]` 的選項按鈕：從 `shared/questions.js` 匯入 Day 14 的解析函式，不另寫一份。按下後組成 `[回覆] 問題1 選 B；…` 的格式送出。
- [ ] 發文框：選擇收件人、標示要回覆的人、送出時帶 `requestId`（用 `crypto.randomUUID()`，重試時沿用同一個）。
- [ ] 通知數：顯示在導覽列；點開列出待處理的通知，可以 ACK。
- [ ] 更新節奏：訊息每 5 秒輪詢，背景分頁（`document.visibilityState`）60 秒。成員與通知數每 30 秒。用 `onCleanup` 清掉計時器。SSE 在 Day 23 才接上。
- [ ] 專案切換：用 `Dropdown Menu` 列出自己所在的專案。
- [ ] 用鍵盤走一遍所有頁面：Tab 順序合理、Dialog 開啟時焦點在裡面、Esc 能關閉。

## 驗收標準

- 能用三種方式登入、看到訊息流、發一則訊息並在 5 秒內出現在另一個瀏覽器視窗。
- AI 發的 `[需確認]` 訊息顯示成按鈕，按下後送出格式正確的回覆。
- 以正式建置的版本測試，把今天用到的每個元件都開過一次，主控台沒有任何 CSP 或 Trusted Types 違規。
- 伺服器送出的 CSP 與 Day 15 結束時完全相同。
- `npm run lint` 通過。
- 切換到背景分頁後，網路面板顯示輪詢間隔變成 60 秒。

## 濫用情境測試

用 Playwright 對正式建置的版本跑。把測試字串與顯示位置寫成兩個陣列，用迴圈跑完所有組合。

- [ ] 測試字串：
  - `<img src=x onerror=alert(1)>`
  - `"><script>alert(1)</script>`
  - `javascript:alert(1)`
  - `<svg onload=alert(1)>`
  - `{{constructor.constructor('alert(1)')()}}`
- [ ] 顯示位置：成員名稱、群組名稱、專案名稱、訊息內文、`[需確認]` 的問題與選項文字、通知的預覽文字。
- [ ] 同樣的字串放進元件內的位置：Dialog 標題、Toast 內容、Tooltip、Menu 項目、Avatar 的替代文字與縮寫。（原專案的 XSS 就出在視窗標題。）
- [ ] 每個組合的斷言：沒有對話框跳出、DOM 中沒有新增的 `<script>`、`<img>`、`<svg>` 元素、字串以原樣的文字出現在頁面上。
- [ ] 訊息中的網址：`javascript:`、`data:`、`vbscript:` 開頭的不會變成可點的連結；`https:` 的會，且帶有 `rel="noopener noreferrer"`。
- [ ] 已登入時開啟 `/invite/<別人的碼>`：出現確認，不會自動切換身分。
- [ ] 邀請碼出現在網址中：頁面載入後用 `history.replaceState` 把它從網址列移除；確認頁面沒有對外發出任何帶有邀請碼的請求（`Referrer-Policy` 已在 Day 8 設定）。
- [ ] API 回 401：導向登入頁，本機沒有殘留的使用者資料。
- [ ] API 回 500 或格式不對的 JSON：顯示錯誤提示，頁面沒有白畫面，錯誤訊息中沒有堆疊。
- [ ] 快速連按送出按鈕 10 次：只產生一則訊息（`requestId` 生效）。
- [ ] 登出後按瀏覽器的上一頁：看不到先前的訊息內容。
- [ ] 只屬於專案 A 的成員，手動把網址改成專案 B 的 slug：看到「找不到」，不是空白的專案 B 頁面，也不洩漏專案 B 的名稱。

## 需要關注的知識點

### shadcn.io 元件與 Ark UI

- **兩層的分工**：Ark UI 負責行為與無障礙；shadcn.io 的檔案在它上面加了 Tailwind 樣式與一些預設的組合方式。
- **元件變體**：用 `tailwind-variants` 的 `tv()` 定義尺寸與樣式的變體。看懂一個 Button 檔，其他的都類似。
- **Tailwind 與資料屬性**：`data-[state=open]:…` 這類寫法就是在用 Ark UI 輸出的狀態屬性。
- **組合式的結構**：每個元件拆成 `Root`、`Trigger`、`Content` 等部分。
- **狀態機**：底層的 Zag 用狀態機描述每個元件的行為，這與 Day 18 的卡片狀態機是同一種思路。
- **Portal**：Dialog、Menu 的內容會被搬到 `body` 底下，寫 CSS 與測試時要注意。
- **無障礙**：焦點管理、鍵盤操作、ARIA 屬性由元件處理。自己寫的部分要用鍵盤實際走一遍。

### SolidJS

- **`createResource`**：把非同步資料變成 signal，搭配 `Suspense` 與 `ErrorBoundary`。
- **store**：巢狀狀態的細粒度更新，適合訊息清單這種會局部變動的資料。
- **context**：全域狀態的傳遞。
- **路由與程式碼分割**：`lazy()` 讓每個頁面各自載入。
- **輪詢與清理**：元件卸載時沒清掉的計時器會持續發請求。

### 安全

- **每一個顯示位置都是一個測試案例**：XSS 不是「有沒有做跳脫」的問題，而是「有沒有哪一處漏了」的問題。用資料驅動的測試涵蓋所有組合。
- **元件庫也是顯示位置**：把資料傳給第三方元件的屬性時，要確認它是當成文字還是當成 HTML 處理。
- **網址也是注入點**：`javascript:` 協定讓一個連結變成執行腳本。
- **網址中的機密**：邀請碼在網址裡時，會留在瀏覽紀錄，也可能經由 Referer 外流。讀取後立刻從網址列移除。
- **前端的權限檢查只是體驗**：隱藏按鈕不是防護。專案隔離由伺服器把關，前端只負責把「找不到」顯示得合理。
- **用戶端的冪等**：重試時沿用同一個 `requestId`，伺服器才認得出是同一個請求。
- **錯誤狀態也是攻擊面**：錯誤訊息常常直接把伺服器回的字串顯示出來。
- **縱深防禦**：框架的跳脫是第一層，lint 規則是第二層，Trusted Types 是第三層，CSP 是第四層。今天的測試確認第一層沒有漏洞；後面三層是為了你沒測到的地方。

## 學習資源

元件：

- shadcn.io 的 Solid 元件總覽（每個元件頁有原始碼與用法）：https://www.shadcn.io/ui/solid
- Ark UI 入門（頁面上方把框架切到 Solid）：https://ark-ui.com/docs/overview/getting-started
- Ark UI 的樣式指南（了解資料屬性）：https://ark-ui.com/docs/guides/styling
- Tailwind CSS 文件：https://tailwindcss.com/docs
- Lucide 圖示：https://lucide.dev/icons/
- Zag 文件（了解底層的狀態機）：https://zagjs.com/
- WAI-ARIA Authoring Practices（每種元件該有的鍵盤行為）：https://www.w3.org/WAI/ARIA/apg/patterns/

SolidJS：

- Solid 文件首頁：https://docs.solidjs.com/ 。從目錄進入 `createResource`、store、context 與 Solid Router 的章節，確認讀的是與你安裝的版本相符的文件。
- Solid Testing Library（若要寫元件層級的測試）：https://github.com/solidjs/solid-testing-library

安全：

- OWASP Cross Site Scripting Prevention Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
- OWASP DOM based XSS Prevention Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html
- PortSwigger XSS（含實作練習）：https://portswigger.net/web-security/cross-site-scripting
- PortSwigger XSS cheat sheet（測試字串的來源）：https://portswigger.net/web-security/cross-site-scripting/cheat-sheet
- OWASP HTML5 Security Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html
- Playwright 的測試寫法（參數化測試）：https://playwright.dev/docs/test-parameterize

## 今日產出

`web/src/components/ui/`（從 shadcn.io 複製並審查過的元件）、`web/src/state/`（使用者與專案的 context）、`web/src/pages/`（Login 完整版、Messages、Invite）、`web/src/components/`（MessageList、Composer、QuestionButtons、NotificationMenu、ProjectMenu）、`web/src/i18n/`、`web/tests/messages.spec.ts`、`web/tests/abuse-xss.spec.ts`、`test/abuse/xss.test.js`（從根目錄呼叫前端的 XSS 測試）。
