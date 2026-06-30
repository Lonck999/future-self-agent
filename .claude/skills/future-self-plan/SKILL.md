---
name: future-self-plan
description: 透過訪談了解使用者想成為的人，調查所需技能/資格、評估現況、產出技能差距排序與規劃週期，寫入 data/profile.json 與 data/plan.json。用於專案初始設定，或使用者要重新規劃時。
---

# Future Self — 初始規劃 Skill

此 Skill 對應 [decisions.md](../../../decisions.md) 第1、2、3、6大題的決定。執行時依序完成以下步驟，每一步都要實際完成才進入下一步，不要省略。

## 步驟

### 1. 訪談「想成為的人」
- 請使用者自由描述想成為的人（混合式輸入，見決策 1-1）。
- 從描述中追問補齊以下欄位（缺哪個才問哪個，不要逐條照搬問卷）：
  - `domain`：領域/職業/角色
  - `timeframe_months`：預期達成的時間範圍
  - `success_criteria`：怎麼算達成
- 範疇限定在**職業/技能**（決策 1-3 現階段範圍，不含生活習慣/價值觀）。
- 一律視為**單一身分**（決策 1-2），若使用者描述了多個目標，提醒對方目前只支援單一身分，請選一個先做。
- 把結果整理成結構化摘要，向使用者確認無誤後才繼續。

### 2. 現況評估
- 依決策 2-2、5-1，現況與後續進度評估都依賴使用者現有資料（履歷/筆記/行事曆歷史）。詢問使用者這些資料在哪裡（檔案路徑、Google Drive 連結等），實際讀取後再做評估，不要憑空假設使用者水平。
- 把讀取過的資料來源記錄到 `profile.json.current_state.sources_consulted`。
- 整理出 `current_state.summary` 與 `current_state.skills`（目前已具備技能與大致水平）。
- 詢問使用者平常會持續記錄進度的筆記位置（檔案路徑或連結），寫入 `profile.json.ongoing_notes_ref`。這個欄位之後每日（證據導向換軌判斷）與每月確認 routine 都會用來判斷進度（決策 4-4、5-1），務必確認清楚，不要留空。
  - 建議使用者把筆記**依技能分資料夾**整理（例如 `Vue/`、`Rust/`），寫進 `ongoing_notes_ref.scope` 的說明裡——這樣每日證據比對才能只掃對應資料夾，不用每次都翻整個筆記庫。不是強制要求，但要主動建議。

### 3. 調查目標所需技能/資格
- 依決策 2-1：先用你自身知識列出該目標角色/領域通常需要的技能與資格。
- 只有當使用者目標涉及**具體職位、具體證照、具體公司要求**等會隨時間變動的資訊時，才用 WebSearch 查證最新狀況；其餘情況不要為了「保險」而搜尋。

### 4. 計算技能差距並排序
- 對每項需要但使用者尚不具備（或不足）的技能，給出：
  - `importance`（1-5）：對達成目標的關鍵程度
  - `current_level`（0-5）：使用者目前水平
  - `gap_score = importance * (5 - current_level)`
  - `track`：技能所屬主線，`backend`／`frontend`／`integration`（需多條主線都有基礎才適合進行的整合型技能，例如部署/DevOps、全端架構設計、Code Review 帶領、開源貢獻）。若目標只涉及單一主線（例如純後端），可以不需要 `integration` 分類，依實際情況判斷。
- **排序規則是區塊式單軌，不是單純依 `gap_score` 排序**（決策 4-4）：同一條 `track` 內部依 `gap_score` 由高到低排，但**整條主線排完才接下一條主線**（例如全部 `frontend` 排完才接全部 `backend`），`integration` 排在所有其他主線之後。哪條主線先排，依使用者自己提到的順序或對話中問清楚的優先意願決定，不要自行假設。若技能間有**前置依賴關係**（例如某技能是另一技能的基礎），依賴關係優先於排序規則，可以用 `prerequisite_for` 欄位標記。
- 填入 `priority_rank`（依上述區塊式規則排序後的最終順位）。
- 寫入 `data/profile.json` 的 `skill_gaps[]`，並更新 `created_at`/`updated_at`。

### 5. 建立規劃週期
- 預設 `cycle.length_months = 3`（決策 2-4），`start_date` 為今天，`end_date` 為 +3 個月。
- `cycle.note` 要寫清楚：這個 cycle 是**週期性檢視/調整點**，不是「達成 `success_criteria` 的截止日」（決策 4-4）。
- `priority_skills` 取 `profile.json.skill_gaps` 依 `priority_rank` 排序的清單。
- 初始化 `plan.json.current_focus`：取 `priority_skills` 第一項，`started_date` 為今天，`assigned_task_ids: []`，`status: "in_progress"`。
- 初始化 `plan.json.background_queue: []`、`background_rotation_index: 0`（技能完成前景學習後才會被加入，見 [ROUTINES.md](../../../ROUTINES.md)）。

### 6. 連結 Google Calendar
- 因現有 `Google_Calendar` MCP 工具無法建立新日曆（決策 6-4 限制），**請使用者先手動在 Google Calendar 建立一個新日曆**（建議名稱：「未來自己 - <domain>」），並提供日曆名稱。
- 用 `list_calendars` 工具找到對應的 calendar ID，確認後寫入 `plan.json.calendar.calendar_id` 與 `calendar_name`。
- 若使用者尚未建立日曆，暫停在這一步，不要假造 ID。

### 7. 寫入並啟用
- 把以上結果寫入 `data/profile.json`、`data/plan.json`，並將 `plan.json.status` 設為 `active`。
- 向使用者總結：本期目標、優先技能清單、週期起訖、對應日曆，並告知接下來每日/每月排程會依此自動執行（見 [ROUTINES.md](../../../ROUTINES.md)）。

## 注意事項
- 所有讀寫只發生在本機專案檔案內（決策 9-2），不外傳第三方。
- 不要在這個 Skill 裡建立每日任務或行事曆事件——那是每日 routine 的工作，這個 Skill 只負責產出 `profile.json` / `plan.json`。
