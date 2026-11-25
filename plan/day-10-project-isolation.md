# Day 10 — 專案隔離

- 階段：2 授權
- 對應 SPEC：§6.2
- 對應資安發現：無。原專案在這裡出過兩個高風險漏洞（SECURITY.md 的「跨專案讀取」「跨專案寫入」），今天的目標是從結構上避免重蹈覆轍
- 前置：Day 9

## 今日目標

建立專案隔離的基礎設施與測試框架。今天還沒有訊息與卡片，先用專案與成員名單做出機制，之後每個功能都接上同一套測試。

## 要做的事

- [ ] `src/authz/isolation.js`：
  - `visibleProjects(me)`：回傳成員看得到的專案 ID 集合。跨專案角色與管理員看得到全部。
  - `scopeSql(me, column)`：回傳一段 SQL 條件與參數，限制查詢只回看得到的專案加上 `all`。所有「列出某類資源」的查詢都必須經過它。
  - `assertVisible(me, resource)`：看不到時丟 404，不是 403。
  - `assertSameProject(people, project, fieldLabel)`：被指定的對象（收件人、負責人）必須是同專案成員，否則 400 `PROJECT_MISMATCH`。
- [ ] 回應過濾：
  - 成員資料中的 `projects`、`projectRoles`，只留下觀看者自己也在的專案。
  - 在送出回應前統一處理，而不是每個 handler 各自處理。
- [ ] `GET /api/participants`：只回同專案的人與跨專案角色。leader 另外看得到尚未加入任何專案的人類。
- [ ] 多專案成員的規則：需要 `project` 的寫入若沒帶，回 400 `PROJECT_REQUIRED`。
- [ ] `test/abuse/isolation-kit.js`：一組可重複使用的測試工具。
  - 固定場景：專案 A、專案 B；Alice 只在 A，Bob 只在 B，Carol 兩邊都在；各有一隻 AI。
  - `assertInvisible(token, method, path)`：斷言回 404，且回應本文中不出現另一個專案的任何名稱、ID、內容。
  - `assertNoLeak(responseBody, forbiddenStrings)`：遞迴搜尋整個回應。
  - 之後每新增一類資源，就把它的列表端點、單筆端點、搜尋端點都丟進這套工具。
- [ ] 把所有需要資源 ID 的路由列成一張清單（放在測試檔內），每新增一條路由就要加一列。清單與路由表不一致時測試失敗。

## 驗收標準

- Alice 呼叫 `GET /api/participants` 看不到 Bob 與 Bob 的 AI。
- Carol 的資料在 Alice 眼中只顯示專案 A。
- 隔離測試工具可以用一行加入新的端點。

## 濫用情境測試

- [ ] Alice 用專案 B 的 ID 呼叫每一條帶專案 ID 的路由：全部回 404，且回應內容與「ID 根本不存在」時完全相同。
- [ ] Alice 的回應中搜尋專案 B 的名稱、slug、卡號代號、成員名稱：都找不到。
- [ ] Alice 把 Bob 設為收件人或負責人（今天先用成員管理端點模擬）：回 400 `PROJECT_MISMATCH`，且錯誤訊息不透露 Bob 屬於哪個專案。
- [ ] 用連號或猜測的 ID 掃描：ID 是不可猜的亂數，掃不到；即使猜中也回 404。
- [ ] Carol 寫入時不帶 `project`：回 400，不會自動選一個。
- [ ] 把 Alice 從專案 A 移除後，她原本快取的資源 ID 全部回 404。

## 需要關注的知識點

- **物件層級的授權（IDOR／BOLA）**：使用者改一下網址裡的 ID 就拿到別人的資料。這是 API 最常見的嚴重漏洞。
- **404 與 403 的差別**：回 403 等於承認「這個東西存在，只是你不能看」。對於不該知道存在的資源，要回 404。
- **列表查詢最容易漏**：單筆查詢通常記得檢查，但列表、搜尋、計數、增量同步、「相關項目」常常忘記。原專案的兩個漏洞都屬於這一類。
- **在查詢層過濾，不要在結果層過濾**：先撈全部再篩選，分頁與計數都會洩漏資訊。
- **旁路洩漏**：錯誤訊息、回應時間、計數、自動完成、通知，都可能洩漏另一個專案的存在。
- **不可猜的 ID 是輔助，不是防護**：ID 可能從別處洩漏，授權檢查仍然必須做。
- **負面測試**：測「看不到」比測「看得到」重要，而且要測整個回應本文，不只是狀態碼。

## 學習資源

- OWASP Insecure Direct Object Reference Prevention Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html
- OWASP API Security Top 10，API1 Broken Object Level Authorization：https://owasp.org/API-Security/
- OWASP Multi-Tenant Security Cheat Sheet（專案隔離就是多租戶隔離）：https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html
- PortSwigger IDOR：https://portswigger.net/web-security/access-control/idor
- PortSwigger 存取控制實作練習：https://portswigger.net/web-security/access-control
- 原專案的 SECURITY.md「公開部署前的安全檢查」表格前兩列：`../aigile-oss/SECURITY.md`。讀它描述的兩個漏洞，想想今天的結構是否能擋住

## 今日產出

`src/authz/isolation.js`、`test/abuse/isolation-kit.js`、`test/abuse/isolation.test.js`、路由清單測試。
