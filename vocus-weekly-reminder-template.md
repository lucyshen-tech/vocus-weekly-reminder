# vocus 媒合週報提醒 — 通用排程腳本

給 CL（Creator Liaison / 媒合專員）每個人使用的自動提醒工具範本。設定好之後，Claude 每週會自動幫你檢查媒合進度表，把「逾期未交稿」「曝光已逾期」「尚未回傳數據」三種需要跟催的訂單整理成表格，發到你的 Slack。

## 這是什麼

vocus 內部媒合進度表（Google 試算表）裡有一大堆訂單，每個人都要自己盯著看有沒有卡關。這個腳本會自動幫你：

1. 讀取試算表的 Raw data
2. 篩選出「你負責」的訂單（依 S 欄「媒合負責人」比對你的名字）
3. 依三個規則挑出需要跟催的訂單
4. 整理成表格，發送 Slack 訊息通知你

## 怎麼設定（3 步驟）

1. 打開 Cowork，叫 Claude 幫你建立排程任務（可以直接說「幫我排程」或使用 `schedule` 這個 skill）。
2. 把下方「Prompt 內容」整段貼給 Claude，並且把裡面兩個變數換成你自己的：
   - `{{YOUR_NAME}}` → 你在 S 欄（媒合負責人）填寫的名字，例如 `Lucy`
   - `{{SLACK_CHANNEL_ID}}` → 你要收通知的 Slack 使用者 ID 或頻道 ID（在 Slack 個人資料頁可以查到，通常長這樣：`U0XXXXXXX`）
3. 告訴 Claude 你想要的執行頻率，例如「每週一早上 11:00」（cron：`0 11 * * 1`），或依自己需求調整（例如想改成每天，可用 `0 12 * * 1-5`）。

設定完成後，建議先手動按一次「Run now」讓 Claude 完成一次瀏覽器與 Slack 的權限授權，之後排程就會自動執行，不會卡在權限確認上。

## Prompt 內容（複製貼上使用）

```
你是 vocus 行銷媒合專員（{{YOUR_NAME}}）的自動助理。每週執行以下工作：

## 步驟一：讀取 Raw data

先用 Claude in Chrome 的 tabs_context_mcp（createIfEmpty:true）取得 tabId，只開一個分頁，然後導覽到：
https://docs.google.com/spreadsheets/d/1WFwJ8wlcc6p95axJEukdbLNZdkcodkFPBlXjkfw1_h4/edit

等頁面載入後，用 javascript_tool 點擊 Raw data 分頁（id=":1e"）：
document.getElementById(':1e').click()

確認切換成功後，再執行以下 fetch 取得資料（使用 gid=0）。注意：JavaScript 必須用 top-level await 的寫法（不要包成 async IIFE 再回傳 Promise，否則工具會回傳空物件），最後一行直接是要回傳的值：

const r = await fetch('https://docs.google.com/spreadsheets/d/1WFwJ8wlcc6p95axJEukdbLNZdkcodkFPBlXjkfw1_h4/gviz/tq?tqx=out:json&gid=0');
const text = await r.text();
const json = JSON.parse(text.replace("/*O_o*/\ngoogle.visualization.Query.setResponse(","").replace(");",""));
const rows = json.table.rows;
const getVal = (row, idx) => row.c[idx] && row.c[idx].v !== null ? row.c[idx].v : null;
const now = new Date();
const myRows = rows.filter(row => getVal(row, 18) === "{{YOUR_NAME}}");
function parseDate(s) { if (!s || !s.startsWith('Date(')) return null; const p = s.replace('Date(','').replace(')','').split(',').map(Number); return new Date(p[0], p[1], p[2]); }
const todayDate = new Date(now.getFullYear(), now.getMonth(), now.getDate());
const pickBase = (r) => ({ order_id: getVal(r,0), brand: getVal(r,2), creator_id: getVal(r,6), nick: getVal(r,7), ig: getVal(r,8) });
const getReason = (r) => { const status = getVal(r, 13); const storeReviewNeeded = getVal(r, 5); if (status === '店家審稿') return '店家尚未審完'; if (status === '我方審稿') return storeReviewNeeded === true ? '尚未提交給店家審稿' : '我方尚未審完'; if (status === '店家審完' || status === '審稿完成') return '創作者尚未曝光'; if (status === '創作者改稿') return '創作者需改稿'; return '原因未知（審稿狀態：' + (status || '無') + '）'; };
const missingDraft = myRows.filter(r => { const status = getVal(r,1); const expectedPost = parseDate(getVal(r,14)); return status === '已體驗/供稿中' && expectedPost && expectedPost < todayDate; }).map(r => ({ ...pickBase(r), expected_date: getVal(r,14) }));
const exposureOverdue = myRows.filter(r => { const status = getVal(r,1); const d = parseDate(getVal(r,14)); const link = getVal(r,15); return status !== '已體驗/供稿中' && d && d < todayDate && !link; }).map(r => ({ ...pickBase(r), expected_date: getVal(r,14), reason: getReason(r) }));
const dataNotReturned = myRows.filter(r => { const status = getVal(r,1); const expectedData = parseDate(getVal(r,16)); const dataReturned = getVal(r,17); return status === '已曝光' && expectedData && expectedData < todayDate && dataReturned !== '已回傳'; }).map(r => ({ ...pickBase(r), expected_data_date: getVal(r,16) }));
JSON.stringify({ missingDraft, exposureOverdue, dataNotReturned });

若資料筆數較多導致 JSON 字串在單次 javascript_tool 呼叫中被截斷，請改用 window 暫存變數（如 window.__exposureOverdue = ...）分批用 JSON.stringify(window.__exposureOverdue.slice(a,b)) 取出，確保每一筆資料都完整取得，不可遺漏或憑印象填寫。

欄位說明（0-based index，若欄位有異動請重新核對一次再執行，詳見下方「欄位參考」章節）：
A(0) 訂單編號 / B(1) 訂單狀態 / C(2) 商家名稱 / F(5) 商家要求審稿 / G(6) 創作者編號 / H(7) 創作者暱稱 / I(8) IG帳號 / M(12) 供稿截止日 / N(13) 審稿狀態 / O(14) 預計曝光貼文日 / P(15) 貼文連結 / Q(16) 預計數據回傳日 / R(17) 數據回傳 / S(18) 媒合負責人

日期格式為 Google Sheets Date 物件，例如 "Date(2026,5,4)" 代表 2026年6月4日（月份0-based），轉換為可讀格式時月份需 +1。

## 步驟二：三個通知類別

1. **逾期未交稿**：B欄為「已體驗/供稿中」，且今天已超過 O欄（預計曝光貼文日）。
2. **曝光已逾期**：今天已超過 O欄且 P欄為空，但排除 B欄為「已體驗/供稿中」的訂單（避免跟項目1重複）。逾期原因依 N欄與 F欄判斷（見下方規則），輸出時只顯示描述文字，不加編號前綴。
3. **尚未回傳數據**：B欄為「已曝光」，且今天已超過 Q欄（預計數據回傳日），且 R欄不是「已回傳」。

## 步驟三：日期轉換

"Date(YYYY,M,D)" → 月份 +1，例如 Date(2026,5,3) → 2026/06/03。

## 步驟四：傳送 Slack 通知

使用工具 mcp__c9364076-1525-43cd-8fdd-7e12b1acb0f4__slack_send_message，channel_id = {{SLACK_CHANNEL_ID}}

訊息格式：

📊 *每週創作者摘要 — YYYY/MM/DD*

*1️⃣ 逾期未交稿（N組）*
| 訂單編號 | 商家名稱 | 創作者編號 | 創作者暱稱 | IG帳號 | 預計曝光日 |
|---|---|---|---|---|---|
（無則顯示：無逾期未交稿）

---

*2️⃣ 曝光已逾期（N組）*
| 訂單編號 | 商家名稱 | 創作者編號 | 創作者暱稱 | IG帳號 | 應曝光日 | 逾期原因 |
|---|---|---|---|---|---|---|
（無則顯示：無曝光已逾期）

---

*3️⃣ 尚未回傳數據（N組）*
| 訂單編號 | 商家名稱 | 創作者編號 | 創作者暱稱 | IG帳號 | 預計回傳日 |
|---|---|---|---|---|---|
（無則顯示：無尚未回傳數據）

## 步驟五：關閉分頁

Slack 訊息成功送出後，用 Claude in Chrome 的 tabs_close_mcp 把步驟一開啟的分頁關閉。
```

## 逾期原因判斷規則（曝光已逾期用）

依 N欄「審稿狀態」與 F欄「商家要求審稿」判斷：

| N欄（審稿狀態） | F欄（商家要求審稿） | 顯示的逾期原因 |
|---|---|---|
| 店家審稿 | — | 店家尚未審完 |
| 我方審稿 | TRUE | 尚未提交給店家審稿 |
| 我方審稿 | 不為 TRUE | 我方尚未審完 |
| 店家審完 / 審稿完成 | — | 創作者尚未曝光 |
| 創作者改稿 | — | 創作者需改稿 |
| 其他（空白 / 供稿完成） | — | 原因未知（審稿狀態：實際值） |

## 欄位參考（0-based index）

> ⚠️ 這是 2026/07/05 核對過的欄位對照。試算表擁有者可能會新增/調整欄位順序，若篩選結果異常（例如篩不到自己的資料），請重新核對一次欄位再使用，不要照抄舊的對照表。

| 欄位 | 內容 |
|---|---|
| A(0) | 訂單編號 |
| B(1) | 訂單狀態 |
| C(2) | 商家名稱 |
| F(5) | 商家要求審稿（boolean） |
| G(6) | 創作者編號 |
| H(7) | 創作者暱稱 |
| I(8) | IG帳號 |
| J(9) | 體驗日期 |
| K(10) | 體驗時間 |
| L(11) | 體驗人數 |
| M(12) | 供稿截止日 |
| N(13) | 審稿狀態 |
| O(14) | 預計曝光貼文日 |
| P(15) | 貼文連結（有值=已曝光） |
| Q(16) | 預計數據回傳日 |
| R(17) | 數據回傳（null 或「已回傳」） |
| S(18) | 媒合負責人 |

## 版本紀錄

- **2026/07/05**：初版為每日（週一到週五 12:00）摘要，含「今日體驗」「曝光已逾期」「逾期未交稿」三類。
- **2026/07/05（更新）**：改為每週一 11:00 執行；通知類別改為「逾期未交稿」「曝光已逾期」「尚未回傳數據」三類，並調整判斷邏輯避免「逾期未交稿」與「曝光已逾期」重複列出同一筆訂單。移除「今日體驗」與舊版「尚未回傳數據」（Q/R欄）邏輯的初版差異。
