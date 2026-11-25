# Day 9 — 專案、角色、權限矩陣

- 階段：2 授權
- 對應 SPEC：§2、§6.1、§12（專案管理）
- 對應資安發現：無（今天照 SPEC 做，重點是把權限寫成可測試的單一來源）
- 前置：Day 8

## 今日目標

把 SPEC §6.1 的權限矩陣寫成一個集中的判斷函式，所有路由都透過它檢查。之後任何權限問題都只需要看一個檔案。

## 要做的事

- [ ] 專案的基本操作：
  - `GET｜POST /api/projects`、`PATCH /api/projects/:id`、`PUT …/slug`、`/code`。
  - slug：2–40 字，小寫英數與 `-`。卡號代號同樣要驗證格式。
  - 每次異動寫 `project_events` 與 `audit_log`。
- [ ] 成員與專案的關係：
  - 新增 `project_members(project_id, participant_id, role, read_only)` 表。SPEC 用逗號分隔字串存在 `participants` 上，改成關聯表才能用外鍵與索引。
  - leader 存成 `project_members.is_leader`，不放在 `app_settings`。
  - `PUT /api/projects/:id/members`、`PUT /api/participants/:id/projects`、`/project-roles`。
- [ ] `src/authz/policy.js`：
  - 單一函式 `can(me, action, resource)` 回 `true` 或丟出帶有具體錯誤碼的 `ApiError`。
  - `action` 是字串常數，例如 `project.create`、`project.manage_members`、`message.post`、`card.update`。
  - 規則照 §6.1 的表逐列寫。唯讀檢查在這裡統一處理（403 `READ_ONLY`）。
  - 沒有列在規則裡的 `action` 一律拒絕。
- [ ] 特殊規則：
  - 管理員與該專案的 leader 不能被設為唯讀。
  - leader 只能是人類且是該專案成員。
  - AI 的專案不能超出所屬人類的專案（400 `OWNER_NOT_IN_PROJECT`）。
  - 人類被設為唯讀時，他名下的 AI 在該專案也唯讀。
  - 人類被移出專案時，他名下的 AI 一併移出。
- [ ] `test/authz-matrix.test.js`：用表格驅動。把 §6.1 的矩陣直接寫成資料（列＝動作，欄＝五種身分，值＝允許與否），測試逐格呼叫 `can()` 比對。

## 驗收標準

- 矩陣測試的每一格都通過。
- 在路由檔案中搜尋 `is_admin`、`is_leader`：除了 `policy.js` 之外沒有任何地方直接判斷身分。
- 新增一個沒寫規則的 `action` 呼叫 `can()`：被拒絕。

## 濫用情境測試

- [ ] leader 管理不是自己專案的成員：回 403。
- [ ] leader 把自己設為另一個專案的 leader：回 403。
- [ ] leader 把管理員設為唯讀：回 400。
- [ ] 一般成員把自己加進專案：回 403。
- [ ] AI 的 Token 呼叫任何專案或成員管理端點：回 403。
- [ ] 唯讀成員對該專案的任何寫入：回 403 `READ_ONLY`。把所有寫入類的 `action` 列出來逐一測試，不要只測一兩個。
- [ ] 唯讀成員名下的 AI 在該專案寫入：回 403 `READ_ONLY`。
- [ ] 在請求本文多塞 `"is_admin": true`、`"kind": "human"` 等欄位更新自己的資料：欄位被忽略。
- [ ] 把人類移出專案後，他的 AI 仍用舊的專案 ID 發請求：回 403 或 404。

## 需要關注的知識點

- **認證與授權是兩件事**：前者回答「你是誰」，後者回答「你能做這件事嗎」。大多數嚴重漏洞出在後者。
- **集中的授權判斷**：權限檢查散落在各個 handler 時，漏掉一個就是漏洞。集中後可以整體測試。
- **預設拒絕**：未知的動作要拒絕，不是放行。
- **表格驅動測試**：規格本身就是一張表時，測試也寫成同一張表，兩者可以直接對照。
- **批量賦值（mass assignment）**：把請求本文整包寫進資料庫，使用者就能改到不該改的欄位。要明確列出可接受的欄位。
- **衍生權限的一致性**：AI 的權限從所屬人類衍生。人類的權限變動時，AI 的要同步，否則會留下孤兒權限。
- **RBAC 與資源層級的檢查**：「是 leader」不夠，還要問「是這個專案的 leader 嗎」。

## 學習資源

- OWASP Authorization Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
- OWASP Authorization Testing Automation Cheat Sheet（今天的矩陣測試就是這篇的做法）：https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Testing_Automation_Cheat_Sheet.html
- OWASP Mass Assignment Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html
- OWASP Top 10 A01 Broken Access Control：https://owasp.org/Top10/
- OWASP API Security Top 10（API1 物件層級、API3 屬性層級、API5 功能層級授權）：https://owasp.org/API-Security/
- PortSwigger 存取控制（含垂直與水平提權的實作練習）：https://portswigger.net/web-security/access-control
- NIST 對 RBAC 的介紹：https://csrc.nist.gov/projects/role-based-access-control

## 今日產出

`src/authz/policy.js`、`src/projects.js`、`migrations/` 中的 `project_members`、`test/authz-matrix.test.js`、`test/projects.test.js`、`test/abuse/authz.test.js`。
