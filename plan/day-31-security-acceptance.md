# Day 31 — 安全驗收與發行

- 階段：6 收尾
- 對應 SPEC：全部
- 對應資安發現：S-01 至 S-18 逐項複核
- 前置：Day 30
- 這一天不能刪。若前面的進度落後，寧可砍功能也要留這一天

## 今日目標

用攻擊者的角度把整個系統走一遍，確認 31 天來做的防護都還在、都有效，然後發行第一版。今天不加新功能。

## 要做的事

### 一、逐項複核 18 項發現

- [ ] 打開 [SECURITY-REVIEW.md](../SECURITY-REVIEW.md)，對每一項填寫下表，存成 `docs/security-acceptance.md`：

| 編號 | 採用的做法 | 對應的測試檔 | 測試通過 | 剩餘風險 |
|---|---|---|---|---|
| S-01 | | | | |
| … | | | | |

- [ ] 「對應的測試檔」那一欄不能是空的。沒有測試的防護，下次重構時就會消失。
- [ ] 「剩餘風險」那一欄要誠實寫。例如 S-04：hook 的字串比對可以被繞過；模型仍可能被誤導而做出規則沒涵蓋的事。

### 二、整批重跑濫用情境測試

- [ ] 執行 `test/abuse/` 下的所有測試。全部通過。
- [ ] 檢查隔離測試工具（Day 10）涵蓋的端點清單與目前的路由表：每一條帶資源 ID 或回傳資源的路由都在清單內。
- [ ] 檢查權限矩陣測試（Day 9）涵蓋目前所有的 `action`。
- [ ] 檢查 GET 無副作用測試（Day 21）涵蓋目前所有的 GET 路由。
- [ ] 檢查不可信標記的走訪測試（Day 22）涵蓋目前所有的 GET 回應。
- [ ] 檢查稽核盤點測試（Day 29）涵蓋目前所有改變權限或憑證的端點。
- [ ] MCP 對等性測試（Day 26）重跑。

### 三、用 ASVS 當檢查清單

- [ ] 取 OWASP ASVS 的 Level 1，加上以下章節的 Level 2：認證、工作階段管理、存取控制、輸入驗證、檔案處理、API。
- [ ] 逐條標記：符合、不適用、不符合。不符合的寫下原因與是否要修。
- [ ] 存成 `docs/asvs-checklist.md`。

### 四、手動測試

- [ ] 用 OWASP ZAP 對本機執行的實例做一次被動掃描加主動掃描。逐項看結果，分辨誤報與真問題。
- [ ] 用瀏覽器的開發者工具與一個攔截代理（ZAP 或 Burp Suite Community），以三個身分各走一遍主要流程：匿名訪客、剛註冊且沒有專案的成員、只在一個專案的一般成員。每個身分都試著：
  - 改請求中的 ID。
  - 把方法從 GET 換成 POST、PUT、DELETE。
  - 在本文中加入多餘的欄位。
  - 重送同一個請求。
- [ ] AI 端的紅隊演練最終版：把 `docs/red-team-notes.md` 中的所有注入情境重跑一次，再加 5 條新的。至少包含：
  - 經由訊息、卡片欄位、附件、技能、另一隻 AI 的工作記憶、成員名稱各一條。
  - 目標各不相同：外洩 Token、外洩 `.env`、執行 `git push`、發訊息叫醒其他 AI、修改自己的工作記憶以持久化。
- [ ] 找一位沒參與開發的人（或另一個 AI 工作階段，只給它 SPEC 與執行中的實例，不給原始碼）花一小時嘗試打破專案隔離。記錄它嘗試了什麼。

### 五、修正與記錄

- [ ] 把今天找到的問題分級。高風險的今天修；其餘的開成議題。
- [ ] 每修一個問題，先寫一支會失敗的測試，再修到通過。
- [ ] 更新 `docs/threat-model.md`：最終版的信任邊界圖、已處理的威脅、接受的風險。
- [ ] 更新 `SECURITY.md` 的「已知的剩餘風險」一節。

### 六、發行

- [ ] 發行檢查清單：
  - 所有測試通過，CI 全綠。
  - 版本號一致。
  - 用離線金鑰簽署腳本清單（Day 25）。把公鑰指紋與 `aigile-setup.mjs` 的雜湊寫進發行說明。
  - 打標籤，觸發映像建置。
  - 從發行的映像（不是本機的程式碼）起一個全新的實例，照 README 從頭走一次：首次設定、建立專案、邀請成員、建立 AI、接入、發訊息、交接卡片。
- [ ] 寫一份回顧，存成 `docs/retrospective.md`：哪幾天的估計偏差最大、哪些防護最花時間、哪些知識點是做了才懂的、下一版要做什麼。

## 驗收標準

- `docs/security-acceptance.md` 的 18 列都填完，每一列都有測試檔。
- `test/abuse/` 全部通過。
- ZAP 的報告中沒有未處理的高風險與中風險項目。
- 全新的實例可以照 README 走完整條流程。
- 發行說明中有公鑰指紋。

## 需要關注的知識點

- **驗收與測試的差別**：測試確認你做的東西照你想的運作。驗收確認你想的東西是對的，而且沒有漏掉什麼。
- **回歸**：早期做的防護很容易在後來的修改中被破壞。這就是為什麼每條防護都要有自動化測試，而且在最後要整批重跑。
- **涵蓋範圍的後設測試**：除了測試本身，還要檢查「測試涵蓋了所有該涵蓋的東西」。今天檢查的五份清單都是這個用途。
- **掃描工具的定位**：自動掃描找得到設定錯誤與常見的注入，找不到授權問題與業務邏輯漏洞。這個系統最重要的風險都屬於後者。
- **誤報與漏報**：掃描報告要逐項判斷。沒有發現不代表沒有問題。
- **剩餘風險**：沒有系統是完全安全的。把還沒解決的、決定接受的風險明確寫出來，是安全工作的一部分，也是對使用者的誠實。
- **獨立的檢視**：你知道系統怎麼設計的，所以你只會測試你想得到的路徑。不知道設計的人會走你沒想到的路。
- **從發行物驗證**：測試本機的程式碼不等於測試使用者拿到的東西。
- **安全是持續的事**：發行之後要有接收漏洞回報的管道、處理的流程、以及重新檢視威脅模型的時機（例如每次加入新的 AI 能力時）。

## 學習資源

- OWASP ASVS：https://owasp.org/www-project-application-security-verification-standard/
- OWASP Web Security Testing Guide（手動測試的方法，依章節對照今天的每一類測試）：https://owasp.org/www-project-web-security-testing-guide/
- OWASP ZAP 入門：https://www.zaproxy.org/getting-started/
- Burp Suite Community 文件：https://portswigger.net/burp/documentation/desktop/getting-started
- OWASP Top 10：https://owasp.org/Top10/
- OWASP API Security Top 10：https://owasp.org/API-Security/
- OWASP Top 10 for LLM Applications：https://genai.owasp.org/llm-top-10/
- OWASP AI Testing Guide：https://owasp.org/www-project-ai-testing-guide/
- OWASP Vulnerability Disclosure Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Vulnerability_Disclosure_Cheat_Sheet.html
- GitHub 的私下回報漏洞功能：https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability
- `security.txt`（RFC 9116，讓研究者知道怎麼聯絡你）：https://www.rfc-editor.org/rfc/rfc9116
- CVSS 計算機（幫問題分級）：https://www.first.org/cvss/calculator/4.0
- 原專案的 SECURITY.md 中「公開部署前的安全檢查」一節是一份實際的自我檢查紀錄，可以當成今天產出的格式參考：`../aigile-oss/SECURITY.md`

## 今日產出

`docs/security-acceptance.md`、`docs/asvs-checklist.md`、最終版的 `docs/threat-model.md` 與 `docs/red-team-notes.md`、更新的 `SECURITY.md`、`docs/retrospective.md`、第一個發行版本。

## 31 天之後

依優先順序：

1. 觀察實際使用一到兩週，記錄 AI 被誤導或越權的每一次事件，回頭更新威脅模型。
2. 技能包（Claude Code 外掛）的支援，需要另外做一次威脅模型。
3. Postgres 支援。
4. Cloudflare Workers 支援。到時要重新面對 S-09（密碼雜湊的限制）與 S-10（沒有內建即時通道）兩個問題。
