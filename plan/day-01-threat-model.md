# Day 1 — 範圍與威脅模型

- 階段：0 基礎
- 對應 SPEC：§1、§2、§3.1、§5.5、§6
- 對應資安發現：無（今天的產出是之後每一天的依據）
- 前置：讀完 [SPEC.md](../SPEC.md) 與 [SECURITY-REVIEW.md](../SECURITY-REVIEW.md)

## 今日目標

寫程式之前，先畫出這個系統有哪些參與者、資料怎麼流、哪裡是信任邊界，並列出每條邊界上可能出的事。今天不寫任何產品程式碼。

## 要做的事

- [ ] 建立程式庫，設定 `.gitignore`（`data/`、`.env`、`.aigile/`）。
- [ ] 寫 `docs/threat-model.md`，包含以下四段：
  - **參與者**：匿名訪客、一般人類、唯讀人類、leader、管理員、AI、AI 的所屬人類、主機管理者。
  - **資產**：密碼雜湊、Token、訊息與卡片內容、專案文件、成員電腦上的程式庫與 `.env`、成員電腦的執行權。
  - **資料流圖**：瀏覽器 ↔ 伺服器 ↔ 資料庫；AI 端腳本 ↔ 伺服器；伺服器 → AI 端腳本（腳本下載、技能下載）；AI ↔ 本機檔案系統。
  - **信任邊界**：至少標出四條。網際網路／伺服器、伺服器／資料庫、伺服器／成員電腦、人寫的內容／AI 的指令。
- [ ] 對每條邊界用 STRIDE 六類各問一次，把答案填成表格。每一格寫「威脅、目前 SPEC 怎麼處理、夠不夠」。
- [ ] 把 SECURITY-REVIEW 的 18 項發現對回你的表格。有對不上的格子，代表你的模型漏了東西，或我的檢核漏了東西。
- [ ] 寫下明確不防護的事（例如：主機管理者本身是惡意的）。

## 驗收標準

- `docs/threat-model.md` 有資料流圖（用 Mermaid 或手繪照片都可以）與 STRIDE 表格。
- 表格中「伺服器 → 成員電腦」這條邊界至少有三項威脅。這是本系統最特別、也最容易被忽略的一條。
- 你能不看文件，口頭說出：「管理員帳號被盜之後，攻擊者最遠能走到哪裡？」

## 濫用情境測試

今天不寫測試。改成列出 10 條濫用情境（「身為某某，我想要某某，以便某某」的反面版本），存進 `docs/abuse-cases.md`。之後每天的濫用情境測試會從這份清單長出來。

範例：「身為只屬於專案 A 的成員，我想讀到專案 B 的卡片標題。」

## 需要關注的知識點

- **信任邊界**：資料從信任程度低的一側進到高的一側時，就是需要驗證的地方。
- **STRIDE**：冒用身分、竄改、否認、資訊洩漏、阻斷服務、提升權限。它是用來提醒你不要漏掉某一類，不是評分工具。
- **資產導向與攻擊者導向**：兩種思考起點。今天兩種都用一次，比較找到的東西有什麼不同。
- **伺服器不一定是受信任的一方**：一般 Web 應用只擔心用戶端騙伺服器。這個系統的伺服器會把程式送到成員電腦上執行，所以方向反過來也要想。

## 學習資源

- OWASP Threat Modeling Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html
- OWASP Abuse Case Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Abuse_Case_Cheat_Sheet.html
- Threat Modeling Manifesto（短，先讀這篇）：https://www.threatmodelingmanifesto.org/
- Microsoft 的 STRIDE 說明：https://learn.microsoft.com/en-us/azure/security/develop/threat-modeling-tool-threats
- 書：Adam Shostack《Threat Modeling: Designing for Security》第 1–3 章

## 今日產出

`docs/threat-model.md`、`docs/abuse-cases.md`、空的程式庫。
