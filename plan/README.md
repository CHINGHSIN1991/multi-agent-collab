# 開發計畫總覽（31 天）

依 [SPEC.md](../SPEC.md) 重新實作，並把 [SECURITY-REVIEW.md](../SECURITY-REVIEW.md) 的 18 項發現排進對應的日子修正。

## 我替你做的假設

你沒有指定以下條件，我先選了預設值。任何一項不對，告訴我，我重排。

| 項目 | 假設 | 不同時怎麼調 |
|---|---|---|
| 目的 | 從零重寫一套，同時當作學習專案 | 若是要直接改 `aigile-oss`，只需要 Day 1、5、8、21–27、31 的資安部分 |
| 人力與時間 | 一人，每天 4–6 小時 | 每天 2 小時就把每個檔案當成兩天 |
| 後端技術 | Node.js 22、Hono 框架、內建 `node:sqlite`（你指定用 Hono；SPEC 原本不用框架、零相依）。伺服器只有 `hono` 與 `@hono/node-server` 兩個相依套件 | 細節與版本見 [day-02](day-02-http-skeleton.md) |
| 前端技術 | SolidJS，元件用 shadcn.io 的 Solid 版（底層是 Ark UI，樣式用 Tailwind CSS），用 Vite 建置（你指定的；SPEC 原本是無建置步驟的原生 JavaScript） | 細節與版本注意事項見 [day-15](day-15-frontend-foundation.md) |
| 執行環境 | 只做 Node＋SQLite | Postgres、Cloudflare Workers、Supabase 列為 31 天後的延伸 |
| 天數 | 31 個工作天 | 見下方「最小可用版本」 |

## 排序原則

1. **先威脅模型，再寫程式。** Day 1 產出的信任邊界圖決定之後每一天的測試要測什麼。
2. **認證與授權先於功能。** Day 4–11 先把「你是誰」「你能做什麼」做完並寫好測試，之後每個功能都直接用。
3. **每天都有濫用情境測試。** 除了驗收功能正確，還要寫「攻擊者這樣做時會被擋下」的測試。這些測試在 Day 31 整批重跑。
4. **AI 端放後面，並有自己的威脅模型日。** AI 端腳本、MCP、技能是風險最高的部分（S-02、S-03、S-04），放在平台穩定後做，Day 22 先重新畫一次威脅模型。

## 每日檔案

| 階段 | 日 | 檔案 | 主題 | 修正的發現 |
|---|---|---|---|---|
| 0 基礎 | 1 | [day-01](day-01-threat-model.md) | 範圍與威脅模型 | — |
| | 2 | [day-02](day-02-http-skeleton.md) | HTTP 骨架（Hono）、錯誤格式、測試工具 | — |
| | 3 | [day-03](day-03-database.md) | 資料庫、升級機制 | — |
| 1 認證 | 4 | [day-04](day-04-tokens-authn.md) | Token 與認證中介層 | — |
| | 5 | [day-05](day-05-password-first-run.md) | 密碼登入、首次設定 | S-01、S-07、S-09 |
| | 6 | [day-06](day-06-devices-invites-pairing.md) | 裝置登入、邀請、配對 | S-06、S-07 |
| | 7 | [day-07](day-07-recovery-audit.md) | 救援、改密碼、稽核紀錄 | S-06、S-15 |
| | 8 | [day-08](day-08-edge-hardening.md) | 代理、HTTPS、安全標頭、限流 | S-05、S-11、S-12、S-13 |
| 2 授權 | 9 | [day-09](day-09-authorization-matrix.md) | 專案、角色、權限矩陣 | — |
| | 10 | [day-10](day-10-project-isolation.md) | 專案隔離 | — |
| | 11 | [day-11](day-11-join-member-admin.md) | 加入流程、成員管理 | S-01 |
| 3 訊息 | 12 | [day-12](day-12-messages.md) | 發文、冪等、機密檢查 | — |
| | 13 | [day-13](day-13-notifications-cursor.md) | 通知、游標分頁、搜尋 | — |
| | 14 | [day-14](day-14-tags-questions-dm.md) | 協定標籤、需確認選項、私訊 | — |
| | 15 | [day-15](day-15-frontend-foundation.md) | 前端基礎：SolidJS 入門、專案設定、安全基礎 | S-05 |
| | 16 | [day-16](day-16-frontend-app.md) | 前端應用：路由、訊息頁面、shadcn.io 元件 | S-05 |
| 4 卡片 | 17 | [day-17](day-17-cards-crud.md) | 卡片基本操作、樂觀鎖 | — |
| | 18 | [day-18](day-18-card-state-machine.md) | 狀態機、交接、驗收 | — |
| | 19 | [day-19](day-19-decisions-suggestions.md) | 決策卡、人類紀錄卡、建議 | — |
| | 20 | [day-20](day-20-card-notices-team-status.md) | 卡片通知、排程、團隊狀態 | — |
| | 21 | [day-21](day-21-ai-onboarding.md) | AI 建立與接入（REST） | S-08 |
| | | | **── 最小可用版本到這裡 ──** | |
| 5 AI 端 | 22 | [day-22](day-22-ai-threat-model.md) | AI 端威脅模型、不可信標記、工作記憶 | S-04 |
| | 23 | [day-23](day-23-live-doorbell.md) | 即時通道與門鈴 | S-10、S-13 |
| | 24 | [day-24](day-24-watcher-wait.md) | watcher 與等待腳本 | S-12、S-14 |
| | 25 | [day-25](day-25-script-integrity.md) | 腳本發佈與完整性、本機強制 | S-02、S-04 |
| | 26 | [day-26](day-26-mcp.md) | MCP 端點與轉接 | — |
| | 27 | [day-27](day-27-skills.md) | 技能庫 | S-03 |
| 6 收尾 | 28 | [day-28](day-28-files-docs.md) | 檔案與文件區 | S-16、S-18 |
| | 29 | [day-29](day-29-backup-admin.md) | 備份、管理功能 | S-15、S-17 |
| | 30 | [day-30](day-30-deploy-ci.md) | Docker、部署、CI | S-12 |
| | 31 | [day-31](day-31-security-acceptance.md) | 安全驗收與發行 | 全部 |

## 最小可用版本

做到 Day 21 就有一套可以用的系統：人類登入、專案與權限、訊息、通知、卡片、AI 用 Token 透過 REST 輪詢參與。這時還沒有即時通知、watcher、MCP、技能、檔案、備份。

工作量提醒：Day 20 要做四個後端模組加三個前端頁面，是 31 天中最重的一天。做不完時把前端頁面移到 Day 21 的上午，Day 21 的 `poll.mjs` 可以延到 Day 24 一起做。

時間不夠時的刪減順序（先刪的在前）：Day 26 MCP → Day 27 技能 → Day 28 文件區（保留訊息附件）→ Day 19 的建議功能。Day 31 不能刪。

## 建議的程式碼配置

每日檔案中提到的檔案路徑以這個配置為準。這是建議，不是 SPEC 的要求。

```
package.json           伺服器的相依：只有 hono 與 @hono/node-server
server.js              進入點：啟動 Hono、靜態檔、逾時設定、排程
src/
  app.js               建立 Hono 實例、掛中介層（順序固定）與各模組的路由
  http.js              readJson 等請求處理的輔助函式
  errors.js            ApiError 與錯誤格式
  config.js            環境變數讀取與驗證
  db.js                資料庫轉接層（get、all、run、batch）
  migrations.js        升級機制
  schema.sql
  auth/                tokens.js、authenticate.js、password.js、login.js、devices.js
  authz/               policy.js（權限矩陣）、isolation.js
  audit.js             稽核紀錄
  ratelimit.js
  messages.js、notifications.js、post-format.js、questions.js、dm.js
  cards.js、card-states.js、suggestions.js
  ai.js                我的 AI、接入、工作記憶
  live.js              即時通道
  mcp.js、skills.js、files.js、docs.js、backup.js
migrations/            NNN_說明.sql
public/                AI 端腳本（不經過打包，Day 25 簽章的對象）
shared/                前後端共用的純函式（questions.js 等）
web/                   前端：SolidJS＋Ark UI，有自己的 package.json
  src/                 index.tsx、pages/、components/、lib/、styles/、i18n/
  src/components/ui/   從 shadcn.io 複製進來的元件原始碼（屬於你的程式庫，要審查）
  dist/                建置產物，由 server.js 提供
test/                  每個模組一支 *.test.js；abuse/ 放濫用情境測試
docs/threat-model.md   Day 1 產出，Day 22 更新
```

## 用 Hono 之後哪些事不變

框架處理路由、請求解析與中介層的串接。下列部分仍然要自己設計與測試，計畫中對應的日子沒有因為用了框架而減少：

- 認證的預設拒絕與公開路徑清單（Day 4）
- 用戶端 IP 的判定與 CSP 的組成（Day 8）
- 權限矩陣與專案隔離（Day 9、10）
- 冪等、游標、機密檢查（Day 12、13）
- 卡片狀態機（Day 18）
- 所有濫用情境測試

各日檔案中提到「在 `src/app.js` 註冊路由」或「handler」的地方，都是指 Hono 的 `app.get`／`app.post` 與它的 handler；讀本文一律用 Day 2 的 `readJson(c)`，取得目前成員用 `c.get('me')`。Day 3、5–7、9 之後的檔案沒有逐一改寫成 Hono 的寫法。

## 每日檔案的格式

每個檔案都有相同的段落：今日目標、要做的事、驗收標準、濫用情境測試、需要關注的知識點、學習資源。「濫用情境測試」每一條都要寫成自動化測試並放進 `test/abuse/`。

## 關於學習資源的連結

連結都選官方文件或長期穩定的來源（OWASP、MDN、RFC、Node.js、SQLite、MCP 規格）。

2026-11-24 我用程式對全部 214 個連結各發了一次請求：211 個回應正常（含轉址後正常）。這只確認頁面存在，我沒有逐頁閱讀內容是否仍與描述相符。有一個連不上的連結已經從 Day 25 移除。另外兩個保留，但回應不正常：

- Day 27 的 Medium 文章〈Dependency Confusion〉回 403。Medium 會擋自動化請求，用瀏覽器應可開啟；打不開就用標題搜尋。
- Day 30 的 Node.js Docker 最佳實務（GitHub 上的檔案）回 503，可能是 GitHub 暫時限制自動化請求。

2026-11-25 改用 SolidJS 與 Ark UI 後，Day 15、16 與 Day 30 新增的連結也同樣檢查過。其中三個回 404，已換成確認可開啟的頁面；Solid 的「反應性」與 Router 等章節沒找到穩定的深層連結，Day 15、16 改為請你從文件首頁的目錄進入。

有幾個連結會轉址到新位置：`docs.claude.com` 的 Claude Code 文件轉到 `code.claude.com`；MCP 規格的總入口轉到當時的最新版（查詢時是 `2026-07-28`，比 SPEC 列的三個版本都新）。

常用的總入口：

- OWASP Cheat Sheet Series：https://cheatsheetseries.owasp.org/
- OWASP ASVS（驗收清單，Day 31 用）：https://owasp.org/www-project-application-security-verification-standard/
- OWASP Top 10 for LLM Applications：https://genai.owasp.org/llm-top-10/
- PortSwigger Web Security Academy（有免費的實作練習）：https://portswigger.net/web-security
- Node.js 文件：https://nodejs.org/api/
- MDN：https://developer.mozilla.org/
