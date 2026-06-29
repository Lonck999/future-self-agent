# 工作進度

## 目前任務
`future-self-plan` Skill 的初始訪談規劃已全部完成，`plan.json.status` 已轉為 `active`，每日/每月 `/schedule` cron routine 已建立並上線。專案進入「已啟動、等待排程實際執行」階段。

## 已完成步驟
1. 與使用者逐項討論並定案 [decisions.md](../decisions.md) 全部 9 大題、28 個決策點。
2. 建立專案資料夾結構：`data/`、`archive/`、`.claude/skills/future-self-plan/`。
3. 設計並寫入資料 schema 說明 [data/SCHEMA.md](../data/SCHEMA.md)。
4. 建立初始訪談規劃 Skill：[.claude/skills/future-self-plan/SKILL.md](skills/future-self-plan/SKILL.md)。
5. 撰寫每日/每月排程邏輯說明：[ROUTINES.md](../ROUTINES.md)。
6. 完成訪談步驟 1（人格/目標定義）：寫入 `profile.json.target_identity`。
7. 完成訪談步驟 2（現況評估）：寫入 `profile.json.current_state`。
8. 完成訪談步驟 3、4（技能調查 + 差距排序）：寫入 `profile.json.skill_gaps`（20 項）。
9. 完成訪談步驟 5（規劃週期）：寫入 `plan.json.cycle`、`plan.json.priority_skills`。
10. 完成訪談步驟 6（連結 Google Calendar）：寫入 `plan.json.calendar`。
11. 補齊 `profile.json.ongoing_notes_ref`：使用者的學習筆記是 Obsidian vault，透過 Git 倉庫同步（公開倉庫 `https://github.com/Lonck999/obsidian`，範圍是整個 vault，沒有固定資料夾），已寫入 `{type: "git_repo", url, scope}`。
12. 同步更新 `data/SCHEMA.md` 對 `ongoing_notes_ref` 物件結構的說明。
13. 把 `plan.json.status` 從 `inactive` 改為 `active`。
14. 用 `/schedule` 建立兩個雲端 cron routine：
    - **每日規劃**（`trig_01LkeqGRHmBFAM5ZbCKTTB1D`）：每天 21:00 Asia/Taipei（UTC `0 13 * * *`），依 [ROUTINES.md](../ROUTINES.md) 的「每日規劃 Routine」執行，使用 Google Calendar MCP 連接器寫入事件，完成後自動 commit+push 回 `origin/main`。
    - **每月確認**（`trig_0155etSpV4SDg3K6ijYkjHev`）：每月 1 號 09:00 Asia/Taipei（UTC `0 1 1 * *`），依「每月確認 Routine」執行，用 Bash `git clone` 公開讀取 Obsidian vault 筆記判斷進度，完成後 commit+push 並用 PushNotification 推送摘要。

## 已寫入的檔案
- `data/profile.json` — `target_identity`、`current_state`、`skill_gaps`（20項）、`ongoing_notes_ref`（已補齊）皆已完成。
- `data/plan.json` — `cycle`、`priority_skills`、`calendar`、`status: "active"` 皆已完成。`history[]` 仍是空陣列，等第一次每月確認 routine 執行後才會有資料。
- `data/progress-log.json` — 仍是空模板，等每日 routine 第一次執行（2026-06-29 21:00 Asia/Taipei）才會開始累積。
- `data/SCHEMA.md`、`ROUTINES.md`、`.claude/skills/future-self-plan/SKILL.md` — 架構文件，已完成不需再動。

## 還沒做完的事
1. 等待每日規劃 routine 第一次實際執行，確認任務能正確寫入 Google Calendar、`progress-log.json`，並且 commit+push 成功（本機 `git pull` 能看到變更）。
2. 等待 2026-07-01 每月確認 routine 第一次執行，確認能讀到 Obsidian vault 筆記內容、產出摘要與調整建議、正確歸檔，並透過 PushNotification 通知使用者。
3. 若每月確認 routine 推送調整建議，需要在後續對話中等使用者明確回覆「同意/調整哪一項」，才能把 `adjustments_applied` 寫回 `plan.json`（這是一次獨立對話，不在排程執行內，見 [ROUTINES.md](../ROUTINES.md) 最後一節）。

## 待確認的背景決策（供繼續對話時參考）
完整定案在 [decisions.md](../decisions.md)，關鍵幾項：
- 單一身分、職業/技能範疇（暫不含生活習慣）
- 現況評估與每月進度確認都依賴使用者的「現有資料/筆記」（Obsidian vault，git 同步），不是問卷
- 每日任務具體可執行、自動排程、未完成自動順延並提高優先度
- 月度調整只提建議、需使用者確認才套用；月度報告主動推送通知
- Google Calendar 沒有「建立新日曆」API，已用使用者手動建立的日曆解決
- 排程失敗自動重試（因事件用唯一 ID 比對更新，重試不會重複建立），重試仍失敗才通知、且不 push 部分完成的變更
- 支援 `status: active/paused` 暫停機制
- 雲端 routine 執行後的資料變更會 commit+push 回 `origin/main`，本機需要 `git pull` 才能看到最新狀態
