# Day 15 — 前端基礎：SolidJS 入門、專案設定、安全基礎

- 階段：3 訊息
- 對應 SPEC：§3.1（前端）、§14.1（CSP）
- 對應資安發現：**S-05**（Token 放 localStorage、CSP）
- 前置：Day 14
- 與 SPEC 的差異：SPEC 的前端是無建置步驟的原生 JavaScript。這裡改用 SolidJS，元件用 shadcn.io 的 Solid 版（底層是 Ark UI，樣式用 Tailwind CSS），需要建置步驟。前端的相依套件只在建置時使用，不進入伺服器的執行環境
- 前端分成兩天：今天打地基，[Day 16](day-16-frontend-app.md) 做實際的頁面

## 今日目標

學會 SolidJS 的基本用法，把前端專案架起來，並在寫任何實際頁面之前先把安全規則建好：Trusted Types、嚴格 CSP、lint 規則、憑證的存放方式。今天結束時只有一個能登入的最小頁面，但它是在所有防護都開啟的狀態下運作的。

先把規則建好再寫頁面，是因為反過來做的成本高得多：頁面寫完才打開嚴格 CSP，會一次冒出幾十個違規，很容易就妥協成放寬規則。

## 動手前先確認版本

2026-11-25 我實際安裝並檢查過以下組合，四個套件加起來共 164 個相依套件：

| 套件 | 版本 | 備註 |
|---|---|---|
| `solid-js` | 1.9.20 | npm 的 `latest` 標籤已經是 2.0.4，但與 Ark UI、`vite-plugin-solid` 一起安裝時，npm 解析出來的是 1.9.20 |
| `@ark-ui/solid` | 5.40.1 | 宣告的相依條件是 `solid-js >=1.6.0` |
| `vite` | 8.3.8 | |
| `vite-plugin-solid` | 2.11.17 | |

Solid 2 把瀏覽器端的執行環境搬到了另一個套件（`@solidjs/web`），`@solidjs/router` 的 2.x 也只支援 Solid 2。我沒有測試 Ark UI 能不能在 Solid 2 上運作。建議用 Solid 1.9.x，路由用 `@solidjs/router` 的 `v1-latest` 標籤；若想用 Solid 2，今天上午先做一個只有 Ark UI Dialog 的頁面確認能跑，再決定。

## 關於 shadcn.io 的 Solid 元件

2026-11-26 我讀了 https://www.shadcn.io/ui/solid 與其中三個元件頁（Dialog、Button、Select），確認了以下幾點：

- **它建在 Ark UI 上**。網站寫明「169 components for Solid, built on Ark UI and styled with Tailwind CSS」。所以先前對 Ark UI 的檢查（樣式用 JavaScript 設定、只有 Splitter 會插入 `<style>`）仍然適用。
- **它不是官方的 shadcn/ui**。網站自己寫明是獨立專案；官方的在 ui.shadcn.com，沒有 Solid 版。
- **元件是複製進你的程式庫的原始碼**，不是 npm 套件。放在 `web/src/components/ui/`，之後由你維護。
- **需要的套件**：`solid-js`、`@ark-ui/solid`、`lucide-solid`（圖示）、Tailwind CSS，以及元件檔案中用到的 `tailwind-variants`。
- **範例的匯入路徑是 `@/components/ui/...`**，所以要設定 `@/` 路徑別名。

我沒能確認的事，動手前要自己查：

- **安裝指令要登入才看得到**，而且網站說指令中「帶有你的 token」。我沒有看到實際的指令與登錄檔網址。
- **Tailwind 是 v3 還是 v4**、**支援的 Solid 版本**，頁面上都沒寫。
- **免費元件的授權條款**：網站只說可用於個人與商業專案、可修改，沒有標明是 MIT 或其他授權。Pro 區塊另有限制轉散布的授權，不要用到。
- **圖表、Toast 等其他元件**我沒有讀，是否符合嚴格 CSP 未知。

## 要做的事

### 上午：學 SolidJS（約 2–3 小時）

- [ ] 做完 Solid 官方的互動教學。沒用過 Solid 的話這一步不要跳過。
- [ ] 教學做完後，不看文件寫出一個小練習確認自己懂了：一個計數器、一個從 `fetch` 讀資料並顯示清單的元件、一個會在卸載時清掉計時器的元件。
- [ ] 特別確認三件事：為什麼元件函式只執行一次；為什麼不能解構 props；`For` 與 `Index` 的差別。

### 下午之一：專案設定

- [ ] 建立 `web/` 資料夾，前端有自己的 `package.json`。根目錄的伺服器維持沒有任何 npm 相依。
- [ ] 安裝：`solid-js`、`@ark-ui/solid`、`@solidjs/router`、`lucide-solid`、`tailwind-variants`（執行時）；`vite`、`vite-plugin-solid`、`tailwindcss` 與它的 Vite 外掛、`typescript`、`eslint`、`eslint-plugin-solid`、`@playwright/test`（開發時）。Tailwind 的版本依 shadcn.io 元件原始碼的寫法決定，先看一個元件檔再裝。
- [ ] 設定 Tailwind：只產生靜態的 CSS 檔，由 `<link>` 載入。這符合 `style-src 'self'`。
- [ ] 設定 `@/` 路徑別名（`vite.config.ts` 的 `resolve.alias` 與 `tsconfig.json` 的 `paths`），指向 `web/src`。
- [ ] 取得 shadcn.io 元件的方式，二選一：
  - 用它的 CLI：指令帶有你帳號的 token。只在自己的電腦上執行，不要把指令貼進程式庫、CI 設定、訊息或卡片。
  - 從元件頁手動複製原始碼。比較慢，但不需要 token，也強迫你把每個檔案看過一遍。建議用這個方式。
- [ ] 今天只取兩個元件：`Field`（或 `Input`）與 `Dialog`，放進 `web/src/components/ui/`。
- [ ] 審查複製進來的每個檔案，確認沒有：`innerHTML`、字串形式的 `style`、插入 `<style>` 元素、從外部網址載入資源、`eval`。這些檔案之後受同一套 ESLint 規則檢查，不設例外。
- [ ] 版本寫死（不用 `^`），提交 `package-lock.json`。之後一律用 `npm ci`。
- [ ] `web/.npmrc` 設 `ignore-scripts=true`：安裝時不執行套件的安裝腳本。
- [ ] `web/vite.config.ts`：輸出到 `web/dist`；開發伺服器把 `/api` 轉給 Node 伺服器；確認對外提供的 source map 中不含機密，或直接關閉。
- [ ] 伺服器（Day 2 的靜態檔邏輯）：網頁從 `web/dist` 提供，沒有副檔名的路徑回 `index.html`；AI 端腳本仍從 `public/` 提供，不經過打包工具（Day 25 的簽章針對的是原始檔案）。
- [ ] 帶雜湊檔名的資源（`/assets/*`）回 `Cache-Control: public, max-age=31536000, immutable`；`index.html` 回 `no-cache`。
- [ ] 伺服器端回 `index.html` 時把群組名稱填入 `<title>` 與連結預覽標籤，填入時做 HTML 跳脫。

### 下午之二：安全基礎

- [ ] **決定登入憑證放哪裡**，把決定與理由寫進 `docs/threat-model.md`：

  | 選項 | 優點 | 缺點 |
  |---|---|---|
  | A. `HttpOnly; Secure; SameSite=Strict` cookie，加 CSRF 檢查 | 頁面上的腳本讀不到憑證 | 需要處理 CSRF；與 SPEC「沒有 cookie」不同 |
  | B. 維持 localStorage 加 Bearer 標頭 | 與 SPEC 一致、沒有 CSRF | 任何 XSS 都能拿走憑證 |

  建議選 A。前端現在有 160 多個第三方套件，其中任何一個出問題都等於頁面上有惡意腳本。AI 仍然用 Bearer Token，只有瀏覽器改用 cookie。

- [ ] 若選 A，伺服器端要改：登入成功時 `Set-Cookie`；`authenticate` 同時接受 cookie 與 Bearer；所有非 GET 請求檢查自訂標頭（例如 `X-Requested-With`）與 `Origin`。
- [ ] **Trusted Types**：Solid 編譯後的範本是用 `innerHTML` 把靜態字串放進 `<template>` 建立的（`solid-js/web` 的 `template()`），所以 `require-trusted-types-for 'script'` 會擋下整個框架。Solid 1.9 提供 `setTrustedTypesPolicy()`：
  - 在 `web/src/index.tsx` 最前面建立一個具名的政策（`solid-template`），`createHTML` 原樣回傳字串，並交給 `setTrustedTypesPolicy`。
  - 這個政策可以原樣回傳，是因為傳進去的字串全部是編譯時產生的，不含執行時的資料。把這個前提寫成註解。
  - CSP 已在 Day 8 加上 `trusted-types solid-template`，只允許這一個政策名稱。
- [ ] **樣式與 CSP**：Day 8 的 `style-src 'self'` 不開 `'unsafe-inline'`。我讀過原始碼，結論是這組技術應該能在這條規則下運作，今天與明天要用瀏覽器實測確認：
  - Solid 的 `style` 屬性傳物件時，是用 `element.style.setProperty` 設定，這不受 CSP 限制。Ark UI 底層的 Zag 傳給框架的樣式都是物件。
  - JSX 中寫死的 `style="..."` 字串會被編譯進範本的 HTML，這會被 CSP 擋下。規則：JSX 中不寫字串形式的 `style`，一律用 class。
  - Zag 只有 Splitter 元件會插入 `<style>` 元素（可設 nonce）。不用 Splitter 就沒有這個問題。
  - shadcn.io 的元件用 Tailwind 的 class 寫樣式，建置時編譯成靜態 CSS。不用執行時產生樣式的 CSS-in-JS 方案。Tailwind 的任意值寫法（例如 `w-[320px]`）也是編譯成 class，沒有問題；但不要為了動態值改用 `style` 字串。
- [ ] **ESLint 規則**（設為錯誤）：
  - `solid/no-innerhtml`：禁止 `innerHTML` 屬性。
  - 自訂 `no-restricted-syntax`：禁止 `outerHTML`、`insertAdjacentHTML`、`document.write`、`eval`、`new Function`、字串形式的 `style` 屬性。
  - `solid/reactivity`、`solid/no-destructure`。
- [ ] `web/src/lib/safe-url.ts`：`safeHref(url)` 只放行相對路徑與 `https:`，其他回傳 `undefined`。
- [ ] `web/src/lib/api.ts`：包裝 `fetch`，統一處理憑證、錯誤格式、401 時導回登入。
- [ ] 寫下兩條規則放進 `web/README.md`：連結一律經過 `safeHref`；不把來自 API 的物件直接展開到元素上（`<div {...data}>`）。

### 下午之三：最小頁面

- [ ] 只做一個頁面：帳號密碼登入，成功後顯示「你好，<名稱>」與登出按鈕。
- [ ] 這個頁面用剛複製進來的兩個 shadcn.io 元件（`Field` 加一個 `Dialog` 當登出確認），目的是驗證它們在嚴格 CSP 與 Trusted Types 下能運作。
- [ ] 設定 Playwright，寫第一支測試：對正式建置的版本登入成功。

## 驗收標準

- `npm run build` 產出 `web/dist`，由 Node 伺服器提供，能用帳號密碼登入並看到自己的名稱。
- 以**正式建置的版本**測試時，開發者工具主控台沒有任何 CSP 或 Trusted Types 違規，包含開啟 Dialog 的時候。
- 伺服器送出的 CSP 與 Day 8 定的相同，沒有加上 `unsafe-inline` 或 `unsafe-eval`。
- `npm run lint` 通過。
- 根目錄只裝伺服器的兩個相依（`npm ci`），沒有任何前端套件，`node server.js` 就能啟動並提供已建置的網頁。
- 你能不看文件解釋：為什麼 Solid 的元件函式只執行一次。

## 濫用情境測試

用 Playwright 對正式建置的版本跑。

- [ ] 把自己的名稱設為 `<img src=x onerror=alert(1)>`，登入後的歡迎訊息以純文字顯示，沒有對話框、沒有新增的 `<img>` 元素。
- [ ] 群組名稱設為 `</title><script>alert(1)</script>`：頁面原始碼中的 `<title>` 是跳脫後的文字。
- [ ] Trusted Types 生效：在頁面中執行 `document.body.innerHTML = '<b>x</b>'` 會丟出錯誤；執行 `trustedTypes.createPolicy('x', {})` 也被擋下。
- [ ] 在一個元件中故意寫 `innerHTML={userText}`：`npm run lint` 失敗。
- [ ] 在一個元件中故意寫 `style="color:red"`：`npm run lint` 失敗；暫時關掉 lint 規則後建置，瀏覽器主控台出現 CSP 違規。確認後復原。
- [ ] 若選 A：`document.cookie` 讀不到登入憑證；從另一個來源送出表單到寫入端點，請求被拒絕。
- [ ] 頁面被放進 `<iframe>`：瀏覽器拒絕顯示。
- [ ] 建置產物中搜尋測試用的 Token、`.env` 的內容、伺服器端的檔案路徑：找不到。
- [ ] `npm audit --omit=dev` 沒有高風險項目。

## 需要關注的知識點

### SolidJS

- **細粒度反應性**：元件函式只執行一次，之後只有用到 signal 的地方更新。這與 React 每次重新執行元件完全不同。
- **不要解構 props**：解構會在當下讀取值，失去反應性。用 `props.x` 或 `splitProps`。
- **signal、memo、effect 的分工**：狀態、衍生值、副作用。
- **控制流程元件**：`Show`、`For`、`Switch`、`Index`。`For` 以參考為鍵、`Index` 以位置為鍵。
- **清理**：計時器、事件監聽都要在 `onCleanup` 中釋放。
- **JSX 被編譯成什麼**：打開建置產物看一次。靜態的部分變成範本字串，動態的部分變成對特定節點的更新。看懂這個，才看得懂下面的安全議題。

### 安全

- **框架預設會跳脫，但有出口**：Solid 的 JSX 把 `{變數}` 當成文字節點插入，不會解析成 HTML。危險的出口是 `innerHTML` 屬性、把資料當成 `href`、以及把不可信的物件展開成屬性。
- **框架自己也用 `innerHTML`**：Solid 用它建立靜態範本。Trusted Types 讓你能區分「框架放進去的編譯時字串」與「其他任何來源的字串」。
- **CSP 對樣式的兩種限制**：`<style>` 元素與 `style` 屬性由 `style-src` 管；用 JavaScript 設定 `element.style` 的屬性不受限制。知道這個差別，才能判斷一個元件庫能不能在嚴格 CSP 下運作。
- **開發模式與正式建置不同**：Vite 的開發伺服器會插入 `<style>` 元素並使用熱更新，在嚴格 CSP 下會違規。CSP 的測試一律對正式建置的版本做。
- **複製進來的程式碼也是你的程式碼**：shadcn 這類做法把元件原始碼放進你的程式庫，好處是每一行都看得到、改得了，也不會被上游的更新默默換掉；代價是沒有人替你更新，安全修補要自己追。複製前要看過，之後要像自己寫的程式一樣審查與測試。
- **第三方相依是攻擊面**：前端從沒有相依變成 160 多個套件，每一個都在使用者的瀏覽器中以你的網站的權限執行。對策是鎖定版本、用 lockfile、關掉安裝腳本、定期檢查已知漏洞、以及讓頁面上的腳本拿不到憑證。
- **XSS 與憑證儲存的關係**：localStorage 中的憑證可被任何腳本讀走；HttpOnly cookie 讀不走，但惡意腳本仍可在頁面內代替使用者操作。A 是減少損害，不是消除影響。
- **CSRF**：瀏覽器會自動帶 cookie，所以別的網站可以代替你送請求。SameSite 與 Origin 檢查是現代的防法。
- **先建規則再寫程式**：防護在第一天就開著，每一行新程式都在它底下通過。

## 學習資源

SolidJS：

- Solid 互動教學（上午做這個）：https://docs.solidjs.com/tutorial/
- Solid 文件首頁：https://docs.solidjs.com/ 。網站同時有 1.x 與 2.x 的文件（2.x 在 `/v2/` 底下），讀的時候確認版本與你安裝的一致。先讀「反應性」（reactivity）的概念章節；這一章的網址我沒找到穩定的深層連結，請從首頁的目錄進入。
- `eslint-plugin-solid` 的規則說明：https://github.com/solidjs-community/eslint-plugin-solid

元件與樣式：

- shadcn.io 的 Solid 元件總覽：https://www.shadcn.io/ui/solid
- shadcn.io 的 Dialog（看原始碼的寫法與相依）：https://www.shadcn.io/ui/solid/dialog
- Tailwind CSS 搭配 Vite 的安裝方式：https://tailwindcss.com/docs/installation/using-vite
- Ark UI 入門（頁面上方把框架切到 Solid；元件出問題時要回到這一層查）：https://ark-ui.com/docs/overview/getting-started

建置工具：

- Vite 指南：https://vite.dev/guide/
- Vite 的建置設定與靜態資源處理：https://vite.dev/guide/build
- npm `ci` 與 lockfile：https://docs.npmjs.com/cli/commands/npm-ci
- OWASP NPM Security Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/NPM_Security_Cheat_Sheet.html
- Playwright：https://playwright.dev/docs/intro

安全：

- MDN Trusted Types API：https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API
- MDN CSP `trusted-types` 指令：https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/trusted-types
- web.dev Trusted Types：https://web.dev/articles/trusted-types
- MDN CSP `style-src`（說明哪些設定樣式的方式受限制）：https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/style-src
- MDN `Set-Cookie`（`HttpOnly`、`Secure`、`SameSite`）：https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie
- OWASP Cross-Site Request Forgery Prevention Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- OWASP Third Party JavaScript Management Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Third_Party_Javascript_Management_Cheat_Sheet.html
- OWASP DOM based XSS Prevention Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html

## 今日產出

`web/package.json`、`web/package-lock.json`、`web/.npmrc`、`web/vite.config.ts`、`web/eslint.config.js`、`web/README.md`、`web/index.html`、`web/src/index.tsx`（含 Trusted Types 政策）、`web/src/lib/api.ts`、`web/src/lib/safe-url.ts`、`web/src/components/ui/`（Field、Dialog 兩個元件）、`web/src/pages/Login.tsx`（最小版）、`web/src/styles/`、`web/tests/login.spec.ts`、`web/tests/abuse-foundation.spec.ts`、更新的 `docs/threat-model.md`。
