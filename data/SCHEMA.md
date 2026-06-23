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
- `ongoing_notes_ref`：使用者持續記錄進度的筆記位置（檔案路徑或連結）。每月確認 routine 會讀取這裡的內容判斷進度（決策 5-1），初始訪談時需向使用者確認此欄位。
- `skill_gaps[]`：每項缺口技能，欄位：
  - `skill`：技能名稱
  - `importance` (1-5)：對達成目標的關鍵程度
  - `current_level` (0-5)：目前水平
  - `gap_score`：`importance * (5 - current_level)`，分數越高越優先
  - `priority_rank`：依 `gap_score` 排序後的順位

## plan.json
目前生效的規劃週期與行事曆對應資訊，由初始訪談 Skill 建立，每日/每月排程 Agent 讀寫。

- `status`：`active` | `paused` | `inactive`（尚未開始）。排程執行前需檢查此欄位，非 `active` 則跳過當次執行。
- `cycle`：本期規劃的起訖日與長度（預設 3 個月一期）。
- `priority_skills`：本期排序後要學習的技能（取自 `profile.json.skill_gaps`，依 `priority_rank` 排序）。
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
