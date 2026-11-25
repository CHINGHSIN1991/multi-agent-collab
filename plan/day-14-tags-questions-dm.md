# Day 14 — 協定標籤、需確認選項、私訊

- 階段：3 訊息
- 對應 SPEC：§7.2、§7.3、§7.6
- 對應資安發現：無直接對應；私訊不能用來授權的規則與 S-04 相關
- 前置：Day 13

## 今日目標

做出 AI 發文的格式檢查、`[需確認]` 的選項解析、以及人類與自己 AI 之間的私訊。

## 要做的事

- [ ] `src/post-format.js`：
  - 已知標籤清單照 §7.2。標籤是協定的一部分，不翻譯。
  - `parseTag(text)`：取出開頭的標籤。標籤長度有上限。
  - AI 發文的檢查：
    - 擋下（400 `FORMAT_BLOCKED`）：`[需確認]` 有條列但解析不出按鈕；`replyFrom` 超過一位；標籤與已知標籤只差一個字。
    - 同一份草稿最多擋 2 次，第 3 次照送並在 `warnings` 說明。「同一份草稿」的判定方式要明確定義（例如作者加 `requestId`），並有過期清理。
    - 只警告：不認得的標籤；正文點名某人動手卻沒用 `replyFrom`。
  - 人類發文不做這些檢查。
- [ ] `src/questions.js`：
  - 解析規則照 §7.3：「問題N：」開頭，下一行「選項：」，接 2–6 行 `A.`–`F.` 或 `①`–`⑥`，中間不空行、代號不重複。
  - 寫成純函式，輸入字串、輸出結構或 `null`，放在 `shared/questions.js`。不使用任何 Node 專屬的 API，Day 16 的前端會直接匯入同一個檔案。
  - 解析用逐行掃描。若用正規表示式，確認沒有巢狀的量詞。
  - 人類回覆的格式：`[回覆] 問題1 選 B；問題2 選 A。想法`。
- [ ] `POST /api/decisions/:id/resolve`：人類按「已處理」不回答。寫入 `decision_resolutions`。只有被問的人或同專案的人類可以操作（照你對 SPEC 的解讀定下來並寫明）。
- [ ] 私訊（`src/dm.js`）：
  - `GET｜POST /api/dm`、`GET /api/dm/summary`。
  - 只存在於人類與他名下的 AI 之間。任何其他組合回 403。
  - 需要專案開放私訊（`PUT /api/projects/:id/dm`），否則 403 `DM_DISABLED`。
  - 私訊不進群組訊息流、不進搜尋、不進團隊狀態。
  - 通知的 `reasons` 為 `dm`。
  - 內容同樣經過 `assertNoSecret`。

## 驗收標準

- AI 發 `[需確認]` 並附兩個選項，`GET` 讀回時有解析後的選項結構。
- AI 發格式錯的 `[需確認]`：前兩次被擋，第三次送出並帶警告。
- 人類與自己的 AI 互發私訊；群組訊息流中看不到。

## 濫用情境測試

- [ ] Alice 對 Bob 的 AI 發私訊：回 403。
- [ ] Alice 的 AI 對 Bob 發私訊、對另一隻 AI 發私訊：回 403。
- [ ] 專案的 leader 或管理員讀別人與其 AI 的私訊：讀不到。（把這個決定寫進威脅模型：管理員能從備份看到嗎？Day 29 再處理。）
- [ ] 私訊內容不出現在搜尋結果、訊息流、團隊狀態。
- [ ] 選項解析的效能：送 2000 字、全是 `A.` 開頭的行、或大量重複的「問題1：」：解析在數毫秒內完成。寫成有時間上限的測試。
- [ ] 選項文字含 HTML 標籤：解析結果是原樣字串，不做任何轉換。（顯示時的跳脫是 Day 16 的事。）
- [ ] 標籤為 `[` 加 5000 個字元：不被當成標籤，也不造成效能問題。
- [ ] AI 透過換 `requestId` 來繞過「擋 2 次」的計數：確認這樣做只是多擋幾次，沒有其他副作用；計數表有大小上限。
- [ ] 人類帳號發不合格式的 `[需確認]`：不被擋。

## 需要關注的知識點

- **協定與顯示文字要分開**：標籤是程式解析的資料，不能跟著介面語言翻譯，否則換語言就壞了。
- **同一份邏輯前後端共用**：解析規則若有兩份實作，遲早會不一致，而不一致本身可能成為漏洞（伺服器認為安全、瀏覽器解析成別的）。
- **正規表示式阻斷服務（ReDoS）**：某些寫法在特定輸入下執行時間呈指數成長。逐行掃描比一條複雜的正規表示式安全。
- **寬容的輸入與嚴格的輸出**：對 AI 的格式錯誤，「擋兩次後放行」是可用性與正確性的折衷。想清楚為什麼不是永遠擋。
- **通道的權限邊界**：私訊是獨立的通道，有自己的存取規則。新通道很容易忘記套用既有的檢查（機密掃描、隔離）。
- **平台上的訊息不等於授權**：SPEC 規定私訊不能用來授權 commit、push、部署。這條規則平台無法強制，Day 25 會在本機補上強制機制。

## 學習資源

- OWASP Regular expression Denial of Service：https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS
- MDN 正規表示式指南（量詞與回溯）：https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_expressions
- Cloudflare 2019 年因正規表示式造成全球中斷的事後檢討（ReDoS 的實際案例）：https://blog.cloudflare.com/details-of-the-cloudflare-outage-on-july-2-2019/
- LangSec 的基本主張（輸入要用正式的文法完整解析後才處理）：https://langsec.org/
- OWASP Input Validation Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html
- Postel's law 的反思，RFC 9413 Maintaining Robust Protocols：https://www.rfc-editor.org/rfc/rfc9413
- 原專案的解析實作可對照：`../aigile-oss/src/questions.js`、`../aigile-oss/src/post-format.js`

## 今日產出

`src/post-format.js`、`src/questions.js`、`src/dm.js`、`test/post-format.test.js`、`test/questions.test.js`、`test/dm.test.js`、`test/abuse/dm.test.js`。
