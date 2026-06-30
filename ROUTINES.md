# Future Self Agent — 排程 Routine 邏輯

> 這份文件是兩個 `/schedule` cron routine（每日規劃、每月確認）的執行依據，對應 [decisions.md](decisions.md) 第4、5、6、7、8大題。設定排程時，把對應段落的步驟交給 routine 執行。

## 共用前置檢查

每次執行（每日或每月）一開始都要做：

1. 讀取 `data/plan.json`。若 `status != "active"`（即 `paused` 或 `inactive`），**直接跳過本次執行**，不做任何事，不通知（決策 7-3：暫停時靜默跳過）。
2. 若讀取或寫入任一步驟失敗（API 錯誤、額度問題等），**自動重試最多 3 次**（決策 7-2）。因事件更新採唯一 ID 比對（見下方），重試不會造成重複建立。重試仍失敗則記錄錯誤並用 `PushNotification` 通知使用者，本次執行終止，不影響下一次排程。

---

## 每日規劃 Routine（每天固定時間執行一次）

1. 前置檢查（見上）。
2. 讀取 `data/plan.json`（`priority_skills`、`current_focus`、`calendar.calendar_id`）與 `data/progress-log.json`。
3. **處理昨天未完成的任務**：檢查 `progress-log.json.entries[]` 中最近一天的任務，找出 `status` 仍是 `pending`（代表沒被標記完成）的任務 → 在今天的任務清單裡建立對應的順延任務：
   - `carried_over_from` = 原始日期
   - `status` = `carried_over`
   - `priority` 比原值提高一級
   - 不重新規劃內容，標題/描述沿用原任務。
4. **讀取使用者筆記一次，供本次執行的所有證據比對共用**：讀取 `profile.json.ongoing_notes_ref` 指向的筆記（`git clone --depth 1` 對應 repo，掃描整個 vault）。
5. **判斷前景技能（`current_focus`）是否完成**（決策 4-4，證據導向、非時間導向）：
   - 比對 `current_focus.assigned_task_ids` 裡每一個任務，是否能在筆記中找到對應的**實際完成證據**（使用者寫的具體成果、程式碼片段、心得描述）——只是「沒被順延」不算證據，一定要在筆記裡找到才算。
   - 若 `assigned_task_ids` 全部都找到證據 → 該技能視為完成：
     - 把該技能加入 `plan.json.background_queue`：`{skill, track, stage: "project_application", entered_stage_date: 今天, assigned_task_ids: []}`。
     - `current_focus` 推進到 `priority_skills` 中下一個尚未進入 `current_focus` 也尚未在 `background_queue` 裡的技能（依區塊式單軌順序：整條 frontend 排完才接整條 backend，最後才是 integration），`assigned_task_ids` 清空、`started_date` 改今天、`status` 改 `in_progress`。
   - 若還沒找到完整證據（或 `assigned_task_ids` 為空、是第一次規劃這個技能）→ `current_focus` 維持不變，不論已經過了幾天都一樣，沒有時間上限，也不可因為「時間到」就跳到下一項技能。
6. **判斷背景技能（`background_queue`）是否該換階段**：對 `background_queue` 中每一筆，比對其目前 `assigned_task_ids` 是否能在筆記中找到對應的**實際專案應用證據**（例如筆記提到把該技能用在某個專案/功能上、有具體產出）。
   - `project_application` 階段找到證據 → `stage` 改 `optimization`，`entered_stage_date` 改今天、`assigned_task_ids` 清空。
   - `optimization` 階段沒有下一階段，找到證據後留在 `optimization` 持續輪替深化即可（不需要再改變 `stage`，但仍可繼續累積新任務與證據比對，代表使用者持續在深化這項技能）。
   - 沒找到證據就維持原狀，繼續在同一 stage 出下一個任務，不設時間上限。
7. **產生今天的前景任務**：圍繞 `current_focus.skill` 出 1-2 個**具體可執行任務**（決策 4-2、4-4）：要有明確標題、描述、預估時長（`duration_minutes`），不要寫抽象提醒。
   - **難度必須對齊使用者目前真實程度**：對照 `profile.json.current_state.skills` 與該技能過去的任務歷史，任務只能是「目前程度的下一小步」，不能假設使用者已經會更進階的東西。例如使用者連 Rust/Vue 基礎語法都還沒寫過時，任務要從 `cargo new`、印出變數、寫一個函式這種最小步驟開始，不能直接出 async/框架/錯誤處理等進階任務。
   - 同一技能要按「先掌握語法 → 小練習 → 整合應用」由淺入深排，不可跳級。
   - 把今天新產生的任務 `id` 加進 `plan.json.current_focus.assigned_task_ids`。
8. **產生今天的背景任務（最多 1 個）**：若 `background_queue` 非空，用 `background_rotation_index % background_queue.length` 選出今天輪到的技能，出 1 個任務：
   - `project_application` 階段：任務內容是「用一個實際小專案/既有專案的功能去應用、複習、加深這項技能」，不是重複前景階段教過的基礎練習。
   - `optimization` 階段：任務內容是「針對這項技能做效能調校、進階用法、邊界案例或品質優化的研究與實作」。
   - 把任務 `id` 加進該筆 `background_queue` 項目的 `assigned_task_ids`，並把 `background_rotation_index` + 1（用於下次輪替）。
   - 若 `background_queue` 為空（例如剛起步、還沒有任何技能畢業），當天就只有前景任務，不用勉強生背景任務。
9. 合併「順延任務」+「前景新任務」+「背景新任務（如有）」成今天的任務清單。
10. **寫入 Google Calendar**（決策 6-2、6-3）：
    - 對每個任務呼叫 `create_event`，建立在 `plan.json.calendar.calendar_id` 對應的日曆（不是主行事曆）。
    - 把回傳的 event ID 存進該任務的 `event_id` 欄位。
    - 若任務是「順延」且舊任務已有 `event_id`，改用 `update_event` 調整時間/標題，不要刪除重建。
11. 把今天的 `{date, tasks[]}` 寫入 `progress-log.json.entries[]`，並把更新後的 `current_focus`、`background_queue`、`background_rotation_index` 寫回 `plan.json`。
12. 不主動通知使用者（每日規劃不需要推送，使用者直接看 Google Calendar 即可，決策 8-1）。

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
