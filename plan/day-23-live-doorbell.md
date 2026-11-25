# Day 23 — 即時通道與門鈴

- 階段：5 AI 端
- 對應 SPEC：§9.3、§7.7、§3.2（即時通知）
- 對應資安發現：**S-10**（Supabase 公開頻道）、**S-13**（即時連線可被佔滿）
- 前置：Day 22

## 今日目標

做出伺服器推送：網頁即時更新，AI 的 watcher 收到門鈴。只做內建的 SSE，不做 Supabase。

## 與 SPEC 的差異

| SPEC | 今天的做法 | 原因 |
|---|---|---|
| Workers 版用 Supabase Realtime 公開頻道 | 不做 | S-10；只做 Node 版就不需要 |
| 全站 500 條、每人 5 條 | 另外為 AI 的 watcher 保留名額；人類每人 3 條 | S-13 |
| 門鈴簽章的金鑰是 Token 的雜湊 | 每隻 AI 另有獨立的門鈴金鑰 | S-10：一把金鑰只做一件事（見下方說明） |

## 要做的事

- [ ] `src/live.js`：
  - `createLiveHub({ maxConnections, maxPerHuman, reservedForAi, heartbeatMs, maxAgeMs })`。
  - `open(me, { signal })` 回傳 `text/event-stream` 的 `Response`。
  - 連線上限：全站上限中保留一部分給 AI；人類連線只能用非保留的部分。同一成員超過上限時關掉最舊的。
  - 每 25 秒送一行心跳註解；每條連線最長 1 小時，到時送 `bye` 事件後關閉，由用戶端重連。
  - 寫入失敗或用戶端中斷時清理連線。確認沒有洩漏計時器。
  - `toPeople(ids, type, data)`：推給指定成員。
- [ ] `GET /api/live`：
  - 需要認證。用 `fetch` 讀串流（不用 `EventSource`，因為它不能帶 `Authorization` 標頭）。若 Day 15 選了 cookie，網頁可以用 `EventSource`。
  - 連線滿了回 503 `LIVE_FULL`，用戶端退回輪詢。
  - 成員被停用或 Token 被重發時，主動關閉他的連線。
- [ ] 推送的內容：
  - 只推「事件類型」與「資源 ID」，不推任何內容。用戶端收到後用自己的憑證呼叫 API 拿資料，隔離由 API 把關。
  - 推給誰由呼叫端決定：只推給看得到那則訊息、那張卡的人。重用 Day 10 的 `visibleProjects`。
  - 在訊息、通知、私訊、卡片的寫入流程中，交易成功後才呼叫推送。
- [ ] 門鈴（先讀這段說明）：只做內建 SSE 時，門鈴走的是已經用 Token 認證過的連線，簽章在安全上是多餘的。它在原設計中的用途是讓門鈴可以經過不受信任的第三方通道（Supabase）。這裡保留簽章有兩個理由：一是練習訊息驗證碼與防重播；二是將來要加第三方通道時不用改 watcher。若時間不夠，可以省略簽章，只留「收到後用自己的 Token 再確認」這一步。另外要知道：獨立金鑰若與 Token 雜湊存在同一個資料庫，資料庫外流時攻擊者一樣能偽造門鈴；金鑰分離防的是「用途混用」，不是資料庫外流。
- [ ] 門鈴的實作：
  - 訊息的收件人有 AI 且通知的 `wake` 為 true 時送出門鈴。
  - 內容：AI 的 ID、時間、隨機數、簽章。不含內文。
  - 簽章：HMAC-SHA256，訊息為 `doorbell|id|at|n`。金鑰是該 AI 專屬的門鈴金鑰，在接入交換時與 Token 一起發給 AI，伺服器端加密或以獨立欄位保存。Token 重發時一併更換。
  - 更新 Day 21 的 `exchange` 回應，加上門鈴金鑰。
- [ ] 網頁接上即時通道：連著時輪詢放慢到 60 秒；斷線時以遞增間隔重連並退回原本的輪詢節奏。

## 驗收標準

- 開兩個瀏覽器視窗，一邊發訊息，另一邊在一秒內更新。
- 用 `curl -N` 以 AI 的 Token 連 `/api/live`，指定該 AI 發訊息後收到一行 `doorbell` 事件。
- 關掉伺服器再啟動，網頁自動重連。

## 濫用情境測試

- [ ] Alice（只在專案 A）連著即時通道時，專案 B 有新訊息：Alice 的連線沒有收到任何事件。連事件的數量與時間點都不該洩漏。
- [ ] 推送的每一種事件的資料中只有 ID 與類型。寫一支測試檢查所有事件的結構，出現任何自由文字欄位就失敗。
- [ ] 100 個人類帳號各開滿連線：AI 的 watcher 仍然連得上（保留名額生效）。
- [ ] 同一個成員開第 4 條連線：最舊的一條被關閉。
- [ ] 開連線後不讀取資料（慢速用戶端）：伺服器的緩衝區有上限，超過就關閉連線，記憶體不會持續成長。
- [ ] 連線中的成員被停用：連線在數秒內被關閉。
- [ ] Token 重發後，用舊 Token 開的連線被關閉。
- [ ] 開了 1000 次連線又關掉：`conns` 的大小回到 0，計時器數量沒有增加。
- [ ] 門鈴簽章：改掉 ID、時間、隨機數任一個，驗證失敗。
- [ ] 用資料庫中存的 Token 雜湊嘗試計算門鈴簽章：驗證失敗（金鑰與 Token 無關）。
- [ ] 未認證的請求連 `/api/live`：回 401，沒有佔用連線名額。

## 需要關注的知識點

- **Server-Sent Events**：單向、建立在一般 HTTP 上、自動重連。與 WebSocket 的取捨。
- **推通知而不推內容**：推送通道只說「有東西變了」，內容仍經由有完整授權檢查的 API 取得。這樣推送通道即使出錯，洩漏的也只有「有動靜」。
- **中繼資料也是資料**：誰在什麼時候被叫醒，本身就透露了哪個專案在活動。
- **長連線是有限資源**：每條連線佔用記憶體與檔案描述符。上限、每人配額、保留名額都是為了不讓少數人佔滿。
- **背壓**：用戶端讀得比伺服器寫得慢時，資料會堆在伺服器的記憶體裡。
- **憑證撤銷與長連線**：請求層級的認證只在連線建立時檢查一次。撤銷憑證時要主動關掉既有的連線。
- **金鑰分離**：一把金鑰只做一件事。用 Token 的雜湊當簽章金鑰，等於讓「拿到資料庫的人」多了一項能力。
- **訊息驗證碼（MAC）**：證明訊息來自持有金鑰的一方且未被改動。隨機數與時間用來防重播。
- **資源清理**：長時間執行的程序中，沒清掉的計時器與監聽器會累積成記憶體洩漏。

## 學習資源

- MDN Using server-sent events：https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- WHATWG HTML 標準的 Server-sent events 章節（事件格式的正式定義）：https://html.spec.whatwg.org/multipage/server-sent-events.html
- MDN Streams API（`ReadableStream` 與背壓）：https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Concepts
- Node.js 的 Backpressuring in Streams：https://nodejs.org/en/learn/modules/backpressuring-in-streams
- RFC 2104 HMAC：https://www.rfc-editor.org/rfc/rfc2104
- MDN `SubtleCrypto.sign()`：https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/sign
- OWASP Cryptographic Storage Cheat Sheet（金鑰分離與金鑰管理）：https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html
- OWASP Denial of Service Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html
- OWASP WebSocket Security Cheat Sheet（長連線的認證與資源問題，概念通用）：https://cheatsheetseries.owasp.org/cheatsheets/WebSocket_Security_Cheat_Sheet.html
- 原專案的實作可對照：`../aigile-oss/src/live.js`

## 今日產出

`src/live.js`、`src/doorbell.js`、`GET /api/live`、網頁的即時更新、`test/live.test.js`、`test/doorbell.test.js`、`test/abuse/live.test.js`。
