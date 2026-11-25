# Day 18 — 狀態機、交接、驗收

- 階段：4 卡片
- 對應 SPEC：§8.2、§15（flows.md K4、K7 與程式不一致的部分，以 SPEC 為準）
- 對應資安發現：無直接對應；「完成者不能自己驗證」是職責分離的實作
- 前置：Day 17

## 今日目標

把卡片的狀態轉換寫成一張明確的表，並實作交接、驗收、退回、接手。

## 要做的事

- [ ] `src/card-states.js`：
  - 狀態：`待整理`、`待處理`、`進行中`、`待確認`、`受阻`、`已完成`、`已封存`。
  - 用資料描述轉換規則：`{ from, event, guard, to, effects }` 的陣列。`transition(card, event, actor)` 查表執行，表中沒有的轉換一律拒絕。
  - 把 §8.2 的事件表逐列寫進去。
- [ ] 事件與端點：
  - **認領**：負責人發 `[接單]`／`[進度]` 並帶 `cardId` 時，待整理／待處理改為進行中。在 Day 12 的發文流程中加入這個掛鉤。
  - **交接** `POST /api/cards/:id/handover`：
    - 一般交接只換負責人，狀態維持。三個例外：待整理改待處理；待確認改進行中並清掉等誰；AI 交接給人類視為請驗收，改待確認、等誰設為該人類。
    - 交接內容四欄，至少要有 `progress` 與 `nextStep`；也接受只寫 `note`。
    - AI 只能交接自己負責的卡。決策卡、已完成、已封存不能交接。
    - 交接對象必須是同專案、非唯讀的 active 成員。
  - **退回** `POST /api/cards/:id/reject`：只有人類能做。回到最近一則 `[交接]` 的 AI 作者，狀態改進行中。原因必填。
  - **標已完成**：經由 `PATCH`。記完成者與時間。人類標已完成同時記為已驗證。
  - **驗證** `POST /api/cards/:id/verify`：卡須為已完成；完成者不能自己驗證（403 `SELF_VERIFY`）。
  - **離開已完成或已封存**：清掉完成與驗證標記。
  - **接手** `POST /api/cards/:id/takeover`：改認領人、留紀錄、通知原認領人。別人認領中時 `reason` 必填。不改負責人與狀態。
  - **受阻**：任何未結案狀態可進出，寫 `blocker` 與 `waitFor`。
- [ ] 所有轉換都帶 `version` 走樂觀鎖，並寫 `card_log`。
- [ ] 設為唯讀時的連帶處理（接上 Day 9）：該成員與名下 AI 負責的未結案卡退回待處理。

## 驗收標準

- 走完一條完整路徑：建卡 → 交接給 AI → AI 接單 → AI 交接給人類 → 人類退回 → AI 再交接 → 人類標已完成。每一步的狀態與 `card_log` 都正確。
- 用測試把轉換表印成矩陣（狀態 × 事件），對照 §8.2 的圖，每一格都有測試。

## 濫用情境測試

- [ ] 對每個狀態送出每一種事件：表中沒有的組合全部回 4xx，卡片不變。這是一個迴圈就能寫完的測試。
- [ ] AI 完成自己的卡之後自己呼叫 `verify`：回 403 `SELF_VERIFY`。
- [ ] 人類 A 的 AI 完成、人類 A 驗證：依 SPEC 是允許的。想一下這算不算自己驗自己，把結論寫進威脅模型。
- [ ] AI 用 `PATCH` 直接把狀態從進行中改成已封存：被拒絕。
- [ ] AI 呼叫 `reject`：回 403。
- [ ] AI 交接別人負責的卡：回 403。
- [ ] 交接給另一個專案的成員、唯讀成員、已停用的成員：回 400。
- [ ] 用 `takeover` 取得修改權：接手後仍不是負責人，`PATCH` 仍回 403。
- [ ] 接手別人認領中的卡而不寫原因：回 400。
- [ ] 兩個人同時交接同一張卡給不同的人：一個成功、一個回 409，`card_log` 只有一筆。
- [ ] 退回時找不到任何 `[交接]` 紀錄的卡：回明確的錯誤，不是 500。
- [ ] 把所有新端點加入隔離測試工具。

## 需要關注的知識點

- **用資料描述狀態機**：轉換規則寫成表，比散落在各個 handler 的 `if` 容易檢查、測試、對照規格。
- **非法轉換預設拒絕**：狀態機最常見的漏洞是「沒想到有人會從這個狀態直接跳到那個狀態」。
- **業務邏輯漏洞**：每個請求單獨看都合法，但組合起來跳過了某個步驟（例如沒人驗收就完成）。這類問題掃描工具找不到。
- **職責分離**：做的人不能是驗的人。這條規則的價值在於它讓單一帳號被盜或單一 AI 被注入時，仍需要第二方才能完成整件事。
- **AI 與所屬人類算不算同一方**：從職責分離的角度，這是需要明確決定的事。
- **稽核軌跡**：`card_log` 要能重建一張卡的完整歷史。每個轉換都要有「誰、何時、從什麼到什麼、為什麼」。
- **衍生狀態的清理**：離開已完成時要清掉完成與驗證標記。忘了清，之後的自動封存會做錯。

## 學習資源

- OWASP Web Security Testing Guide，Business Logic Testing 一章：https://owasp.org/www-project-web-security-testing-guide/
- PortSwigger 業務邏輯漏洞（含實作練習）：https://portswigger.net/web-security/logic-flaws
- OWASP Top 10 A04 Insecure Design：https://owasp.org/Top10/
- NIST 詞彙表 Separation of Duty：https://csrc.nist.gov/glossary/term/separation_of_duty
- 狀態機的入門說明（Statecharts 網站）：https://statecharts.dev/
- XState 文件的概念章節（不用這個函式庫，但概念說明清楚）：https://stately.ai/docs/state-machines-and-statecharts
- 原專案的流程說明可對照，注意 SPEC §15 列出的差異：`../aigile-oss/docs/flows.md`

## 今日產出

`src/card-states.js`、`src/cards.js`（交接、退回、驗證、接手）、`test/card-states.test.js`、`test/card-flows.test.js`、`test/abuse/card-states.test.js`。
