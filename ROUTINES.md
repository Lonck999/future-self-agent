# Future Self Agent — 排程 Routine 邏輯

> 這份文件是兩個 `/schedule` cron routine（每日規劃、每月確認）的執行依據，對應 [decisions.md](decisions.md) 第4、5、6、7、8大題。設定排程時，把對應段落的步驟交給 routine 執行。

## 共用前置檢查

每次執行（每日或每月）一開始都要做：

1. 讀取 `data/plan.json`。若 `status != "active"`（即 `paused` 或 `inactive`），**直接跳過本次執行**，不做任何事，不通知（決策 7-3：暫停時靜默跳過）。
2. 若讀取或寫入任一步驟失敗（API 錯誤、額度問題等），**自動重試最多 3 次**（決策 7-2）。因事件更新採唯一 ID 比對（見下方），重試不會造成重複建立。重試仍失敗則記錄錯誤並用 `PushNotification` 通知使用者，本次執行終止，不影響下一次排程。

---

## 每日規劃 Routine（每天固定時間執行一次）

1. 前置檢查（見上）。
2. 讀取 `data/plan.json`（`priority_skills`、`calendar.calendar_id`）與 `data/progress-log.json`。
3. **處理昨天未完成的任務**：檢查 `progress-log.json.entries[]` 中最近一天的任務，找出 `status` 仍是 `pending`（代表沒被標記完成）的任務 → 在今天的任務清單裡建立對應的順延任務：
   - `carried_over_from` = 原始日期
   - `status` = `carried_over`
   - `priority` 比原值提高一級
   - 不重新規劃內容，標題/描述沿用原任務。
4. **產生新任務**：依 `priority_skills` 排序，挑選尚未被排進近期任務的技能，產出 1-3 個**具體可執行任務**（決策 4-2）：要有明確標題、描述、預估時長（`duration_minutes`），不要寫抽象提醒。
   - **難度必須對齊使用者目前真實程度**：對照 `profile.json.current_state.skills` 與該技能過去的任務歷史，任務只能是「目前程度的下一小步」，不能假設使用者已經會更進階的東西。例如使用者連 Rust/Vue 基礎語法都還沒寫過時，任務要從 `cargo new`、印出變數、寫一個函式這種最小步驟開始，不能直接出 async/框架/錯誤處理等進階任務。
   - 同一技能要按「先掌握語法 → 小練習 → 整合應用」由淺入深排，不可跳級；若某技能標記了 `prerequisite_for`，代表它是其他技能的前置，必須先在該技能上累積足夠任務並標記完成，才能開始排被依賴的技能。
5. 合併「順延任務」+「新任務」成今天的任務清單。
6. **寫入 Google Calendar**（決策 6-2、6-3）：
   - 對每個任務呼叫 `create_event`，建立在 `plan.json.calendar.calendar_id` 對應的日曆（不是主行事曆）。
   - 把回傳的 event ID 存進該任務的 `event_id` 欄位。
   - 若任務是「順延」且舊任務已有 `event_id`，改用 `update_event` 調整時間/標題，不要刪除重建。
7. 把今天的 `{date, tasks[]}` 寫入 `progress-log.json.entries[]`。
8. 不主動通知使用者（每日規劃不需要推送，使用者直接看 Google Calendar 即可，決策 8-1）。

---

## 每月確認 Routine（每月固定一天執行一次）

1. 前置檢查（見上）。
2. 讀取 `data/profile.json.ongoing_notes_ref` 指向的筆記內容，以及 `data/progress-log.json` 中本月的 `entries[]`。
3. **依使用者筆記內容判斷進度**（決策 5-1，不是檢查行事曆完成率、也不是問卷）：比對本月筆記提到的進展，與 `plan.json.priority_skills` / 本月任務清單，評估「做到哪裡」「哪些落後」。
4. 產出本月摘要 `summary`，若有明顯落後或落差，產出 `adjustments_proposed`（具體建議，例如調整優先順序、延長某項技能的時間）。**不要自動套用**，`adjustments_applied` 保持空陣列（決策 5-2、8-2：結構性調整一律先建議、等使用者確認）。
5. 把 `{month, summary, adjustments_proposed, adjustments_applied: []}` 加入 `plan.json.history[]`。
6. **歸檔**：把 `progress-log.json` 中已結束月份的 `entries[]` 摘要寫入 `archive/<YYYY-MM>.json`，從 `entries[]` 移除，並在 `progress-log.json.archive[]` 加入索引 `{month, file}`（決策 3-3：避免主檔案無限增長）。
7. **主動推送通知**（決策 5-3、8-1）：用 `PushNotification` 把本月摘要 + 調整建議發給使用者，請使用者之後在對話中明確回覆「同意/調整哪一項」。
8. 使用者確認調整後（這是後續一次獨立的對話，不在本次排程執行內）：才將對應建議寫入 `adjustments_applied`，並更新 `plan.json` 的 `priority_skills`/`cycle` 等欄位。

---

## 與 future-self-plan Skill 的分工

- [future-self-plan](.claude/skills/future-self-plan/SKILL.md) Skill：負責**第一次**建立 `profile.json`/`plan.json`，或使用者明確要求「重新規劃」時觸發。
- 本文件的兩個 routine：負責**已經有 active plan 之後**的日常自動執行，不重新做訪談或技能調查。
