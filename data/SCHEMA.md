# 資料 Schema 說明

> 真相來源是 `data/*.json`（純 JSON，依 [decisions.md](../decisions.md) 第3大題）。本檔案是給人看的欄位說明，不是程式解析對象。

## profile.json
使用者的「目標身分」與現況評估，由初始訪談 Skill 產生與更新。

- `target_identity.raw_description`：使用者原始自由描述。
- `target_identity.domain`：訪談追問補齊的領域（例：後端架構師）。
- `target_identity.timeframe_months`：使用者預期達成的時間範圍。
- `target_identity.success_criteria`：衡量是否達成的標準。
- `current_state.sources_consulted`：本次現況評估讀取過的資料來源（履歷檔案路徑、筆記檔案路徑、行事曆查詢範圍等）。
- `current_state.skills`：使用者目前已具備技能與水平的列表。
- `ongoing_notes_ref`：使用者持續記錄進度的筆記位置。每月確認 routine 會讀取這裡的內容判斷進度（決策 5-1），初始訪談時需向使用者確認此欄位。物件欄位：
  - `type`：來源類型，例如 `git_repo`（目前唯一已實作的類型）
  - `url`：可存取的位置（例：git 倉庫 URL）
  - `scope`：要讀取的範圍說明（例：整個 vault，或特定資料夾/檔名）
- `skill_gaps[]`：每項缺口技能，欄位：
  - `skill`：技能名稱
  - `importance` (1-5)：對達成目標的關鍵程度
  - `current_level` (0-5)：目前水平
  - `gap_score`：`importance * (5 - current_level)`，分數越高越優先（僅供參考，非最終排序依據）
  - `priority_rank`：最終學習順位。優先依 `gap_score`，但當技能間有**前置依賴關係**時（例如沒寫過基礎語法就不該排非同步框架），依賴關係優先於分數，基礎/前置技能排到依賴它的技能之前；整體排序是**區塊式單軌**（見 `track`）：整條 `frontend` 技能線排完才接整條 `backend` 技能線，`integration` 排最後
  - `prerequisite_for`（可選）：此技能是哪些技能的前置條件，標記出依賴鏈，避免任務生成時跳級
  - `track`：技能所屬主線，`backend`（Rust）｜`frontend`（Vue）｜`integration`（需兩條主線都有基礎才適合進行的整合型技能，排在最後階段）

## plan.json
目前生效的規劃週期與行事曆對應資訊，由初始訪談 Skill 建立，每日/每月排程 Agent 讀寫。

- `status`：`active` | `paused` | `inactive`（尚未開始）。排程執行前需檢查此欄位，非 `active` 則跳過當次執行。
- `cycle`：本期規劃的起訖日與長度（預設 3 個月一期）。`note` 說明 cycle 是週期性檢視點，不是達成目標的截止日。
- `priority_skills`：本期排序後要學習的技能（取自 `profile.json.skill_gaps`，依 `priority_rank` 排序，雙軌交替）。
- `current_focus`：**前景（foreground）**——目前正在「初次學習新概念」的單一技能，同一時間只會有一項。欄位：
  - `skill` / `track`：目前聚焦的技能名稱與所屬主線
  - `started_date`：開始聚焦此技能的日期
  - `assigned_task_ids`：已經為此技能出過的任務 ID 清單（對應 `progress-log.json` 的任務 `id`），每日規劃需讀使用者筆記比對這些任務是否都有完成證據
  - `status`：`in_progress`（進行中）｜`completed`（已找到完整證據，待下次規劃推進到下一項技能）
  - 證據導向判斷：全部 `assigned_task_ids` 都在筆記中找到完成證據，才視為完成——該技能會被移入 `background_queue`（stage 從 `project_application` 開始），`current_focus` 推進到 `priority_skills` 中下一項技能；否則維持原技能繼續出下一步任務，不設時間上限
- `background_queue`：**背景（background）**——已經從 `current_focus` 畢業、但仍持續深化的技能清單，與目前的 `current_focus` 同時並存、每日都可能各出一點任務。每筆欄位：
  - `skill` / `track`：技能名稱與所屬主線
  - `stage`：`project_application`（用實際專案應用/複習深化該技能）｜`optimization`（效能/進階優化研究，畢業後持續留在此階段，不會退出佇列）
  - `entered_stage_date`：進入目前 stage 的日期
  - `assigned_task_ids`：在目前 stage 已出過的任務 ID，證據導向比對，全部找到完成證據才從 `project_application` 畢業到 `optimization`（`optimization` 之後沒有下一階段，持續輪替深化）
- `background_rotation_index`：每日規劃用來在 `background_queue` 中輪流挑一項技能出當天背景任務的指標，避免畢業技能越多、背景任務越爆炸——每天前景出 1 個任務、背景最多再出 1 個（輪到誰就出誰）。
- `calendar`：對應的 Google Calendar 資訊（獨立的「未來自己」日曆，不寫入主行事曆）。
- `history[]`：每期/每月的回顧紀錄，欄位：
  - `month`：對應月份（YYYY-MM）
  - `summary`：依使用者筆記產出的當月進度摘要
  - `adjustments_proposed`：系統提出的調整建議
  - `adjustments_applied`：使用者確認後實際套用的調整（若未確認則為空）

## progress-log.json
每日任務的執行紀錄，由每日排程 Agent 寫入，每月由月度 Agent 摘要後歸檔。

- `entries[]`：每日一筆，欄位：
  - `date`：YYYY-MM-DD
  - `tasks[]`：當天任務列表，欄位：
    - `id`：任務唯一 ID
    - `title` / `description` / `duration_minutes`：具體可執行任務內容
    - `event_id`：對應的 Google Calendar event ID（用於更新避免重複建立）
    - `status`：`pending` | `carried_over`（自動順延）
    - `carried_over_from`：若為順延任務，記錄原始日期
    - `priority`：優先度，順延任務會被提高
- `archive[]`：已歸檔的月份索引，欄位 `month`、`file`（指向 `archive/YYYY-MM.json`）。歸檔後，`entries[]` 中對應月份的細項會從本檔移除，只留歸檔索引。
