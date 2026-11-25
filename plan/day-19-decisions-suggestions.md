# Day 19 — 決策卡、人類紀錄卡、建議

- 階段：4 卡片
- 對應 SPEC：§8.4、§8.5、§8.6、§6.1（決策卡不看身分）
- 對應資安發現：無直接對應；決策卡是「人類保留最終決定權」的核心機制，與 S-04 的防護目標一致
- 前置：Day 18

## 今日目標

完成剩下的兩種卡片類型與 AI 建議功能。三者的共同點是：AI 可以提出，但只有人類能決定。

## 要做的事

- [ ] 決策卡：
  - 建立即為待確認。欄位：`question`、`recommendation`、`options`（最多 4 個、每個 40 字）、`impact`、`afterNext`、`deciderId`。
  - `deciderId` 必須是人類。決策者不受「同專案」限制的話要明確寫出並測試；建議仍要求同專案或跨專案角色。
  - `POST /api/cards/:id/decide`：只有指定的決策者能呼叫（403 `NOT_DECIDER`），**管理員也不例外**。在 `policy.js` 中這條規則要放在管理員放行之前。
  - `result`：某個選項（沒設選項時為「同意」）、`不同意`、`要求補充`。後兩者 `note` 必填。
  - `要求補充`：等誰改為提案人；提案人補充後自動回到決策者。
  - 決定後狀態為已完成，內容不能再改（409 `DECIDED`）。不能用 `PATCH` 標已完成（400 `USE_DECIDE`）。
  - `POST …/defer`、`/resume`：決策者、提案人或建卡者可操作。暫緩以「受阻＋阻塞原因以『暫緩（』開頭」表示。
  - 離開已完成時決定作廢（接上 Day 18 的規則）。
- [ ] 人類紀錄卡：
  - 不發群組訊息。
  - AI 的一般列表、增量、計數、團隊狀態都不含這類卡。
  - AI 只能讀。唯一可做的修改是標「已完成」並帶 `mergedInto`（同專案的一般卡）。
  - 負責人與等誰只能是人類。
  - 內容上限 1000 字。
- [ ] 建議（`src/suggestions.js`）：
  - `POST /api/suggestions`：`kind` 為 `status` 或 `create`；`why` 必填、300 字內。
  - 每隻 AI 最多 20 則未處理（429 `TOO_MANY_SUGGESTIONS`）。
  - `GET /api/suggestions`、`POST …/:id/apply`、`/skip`、`POST /api/suggestions/batch`。只有人類能採用或略過。
  - 採用時以**採用者**的身分與權限執行對應的卡片操作，重新走一次完整的授權檢查。
  - 卡片在建議之後被修改過，建議標為過期，不能採用。
  - 批次操作有筆數上限；每一筆各自檢查權限。

## 驗收標準

- AI 建立決策卡，指定的人類選了選項 B，卡片變為已完成且內容鎖定。
- AI 建議把別人的卡改成受阻，人類採用後卡片狀態改變，`card_log` 記錄的操作者是那位人類並註明來自哪則建議。
- AI 的卡片列表中沒有人類紀錄卡。

## 濫用情境測試

- [ ] 管理員對不是指定給自己的決策卡呼叫 `decide`：回 403 `NOT_DECIDER`。
- [ ] AI 呼叫 `decide`：回 403。
- [ ] AI 建立決策卡時把 `deciderId` 設為另一隻 AI 或自己：回 400。
- [ ] 提案的 AI 在建卡後用 `PATCH` 改 `deciderId` 為對自己有利的人：想清楚這該不該允許。建議只允許人類改決策者，並寫測試。
- [ ] 決定之後修改 `options`、`question`、`recommendation`：回 409 `DECIDED`。
- [ ] 決定之前，提案人在決策者打開卡片後偷改選項內容，決策者按下的「B」變成別的意思：`decide` 必須帶 `version`，版本不符回 409。這是今天最重要的一條測試。
- [ ] `result` 送一個不在選項中的值：回 400。
- [ ] `不同意` 不帶 `note`：回 400。
- [ ] AI 修改人類紀錄卡的內容：回 403。AI 用 `mergedInto` 指向另一個專案的卡：回 400 或 404。
- [ ] AI 的 `GET /api/cards`、`?since=`、計數、團隊狀態中，逐一確認沒有人類紀錄卡的任何欄位。
- [ ] AI 建立第 21 則建議：回 429。
- [ ] 唯讀人類採用建議：回 403。人類採用一則會改到他沒有權限的卡的建議：回 403。
- [ ] 建議送出後卡片被改過，再採用：被拒絕並標為過期。
- [ ] AI 呼叫 `apply`：回 403。

## 需要關注的知識點

- **人在迴路中（human in the loop）**：關鍵決定保留給人。機制上要確保 AI 沒有任何路徑可以繞過，包含間接的路徑（改決策者、改選項內容）。
- **確認的是什麼**：決策者按下「同意」時，他同意的必須是他當時看到的內容。這就是為什麼 `decide` 要綁版本。這個問題在交易簽核的領域叫「所見即所簽」。
- **即使是最高權限也有做不到的事**：決策卡刻意不讓管理員代為決定。有些權限應該綁在特定的人身上，而不是角色上。
- **代理執行要用誰的權限**：採用建議時，動作是 AI 提的，但執行的是人。必須用人的權限重新檢查，否則 AI 可以借人的手做到自己做不到的事（confused deputy）。
- **過期的提案**：提案是基於當時的狀態。狀態變了，提案就不該再能執行。
- **對 AI 隱藏資料要全面**：人類紀錄卡不給 AI 看，就要檢查所有可能帶出卡片資料的端點，包含計數與統計。
- **配額**：限制未處理的數量，防止 AI 用大量建議淹沒人類，讓人類因疲勞而不看內容就全部採用。

## 學習資源

- OWASP Transaction Authorization Cheat Sheet（「所見即所簽」的原則在這裡）：https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html
- 原始論文 The Confused Deputy（短，值得讀）：https://css.csail.mit.edu/6.858/2015/readings/confused-deputy.html
- OWASP Top 10 for LLM Applications，Excessive Agency 一項：https://genai.owasp.org/llm-top-10/
- Anthropic〈Building effective agents〉（人類檢查點的設計）：https://www.anthropic.com/engineering/building-effective-agents
- NIST AI Risk Management Framework（人類監督的段落）：https://www.nist.gov/itl/ai-risk-management-framework
- OWASP Authorization Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
- PortSwigger 業務邏輯漏洞：https://portswigger.net/web-security/logic-flaws

## 今日產出

`src/cards.js`（決策與人類紀錄）、`src/suggestions.js`、`test/decisions.test.js`、`test/notes.test.js`、`test/suggestions.test.js`、`test/abuse/decisions.test.js`。
