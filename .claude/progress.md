# 工作進度

## 目前任務
正在執行 `future-self-plan` Skill（初始訪談規劃），目標身分：大師級全端工程師（Vue 系列 + Rust），第一期週期 6 個月（2026-06-23 ~ 2026-12-23）。目前卡在步驟「補齊 profile.json.ongoing_notes_ref」，等待使用者回覆筆記位置，之後才能把 `plan.json.status` 從 `inactive` 轉成 `active`。

## 已完成步驟
1. 與使用者逐項討論並定案 [decisions.md](../decisions.md) 全部 9 大題、28 個決策點。
2. 建立專案資料夾結構：`data/`、`archive/`、`.claude/skills/future-self-plan/`。
3. 設計並寫入資料 schema 說明 [data/SCHEMA.md](../data/SCHEMA.md)。
4. 建立初始訪談規劃 Skill：[.claude/skills/future-self-plan/SKILL.md](skills/future-self-plan/SKILL.md)。
5. 撰寫每日/每月排程邏輯說明：[ROUTINES.md](../ROUTINES.md)（尚未實際設定 `/schedule` cron routine，這是最後一步，待 plan 啟用後才設定）。
6. 完成訪談步驟 1（人格/目標定義）：寫入 `profile.json.target_identity`。
7. 完成訪談步驟 2（現況評估，口述方式）：寫入 `profile.json.current_state`（Vue、Rust 皆仍在看教學階段，無整合/部署經驗）。
8. 完成訪談步驟 3、4（技能調查 + 差距排序）：寫入 `profile.json.skill_gaps`（20 項技能，依 importance x 現況落差排序，已給 priority_rank）。
9. 完成訪談步驟 5（規劃週期）：寫入 `plan.json.cycle`（6 個月）與 `plan.json.priority_skills`。
10. 完成訪談步驟 6（連結 Google Calendar）：使用者已手動建立日曆「future-self」，用 `list_calendars` 查到 ID 並寫入 `plan.json.calendar`。

## 已寫入的檔案
- `data/profile.json` — 已填入 `target_identity`、`current_state`、`skill_gaps`（20項）。**`ongoing_notes_ref` 仍是 `null`，待補。**
- `data/plan.json` — 已填入 `cycle`、`priority_skills`、`calendar`（calendar_id: `a0f91305d642a598c5a345624eea531d03f2bdf709a8a9adccfd57a3963993cd@group.calendar.google.com`，calendar_name: `future-self`）。**`status` 仍是 `inactive`**，要等 `ongoing_notes_ref` 補齊後才轉 `active`。
- `data/progress-log.json` — 仍是空模板，未開始累積每日紀錄（要等 plan 啟用、每日 routine 開始跑才會有資料）。
- `data/SCHEMA.md`、`ROUTINES.md`、`.claude/skills/future-self-plan/SKILL.md` — 架構文件，已完成不需再動。

## 還沒做完的事
1. **取得 `ongoing_notes_ref`**：使用者目前的學習筆記放在哪裡（檔案路徑/Notion/Google Doc），尚未回覆。← 卡在這裡。
2. 把 `ongoing_notes_ref` 寫入 `profile.json`，並將 `plan.json.status` 改成 `active`。
3. 設定實際的每日/每月 `/schedule` cron routine（依 [ROUTINES.md](../ROUTINES.md) 的邏輯），這是整個 TodoWrite 清單最後一項待辦，目前仍是 `pending`。
4. 啟用後跑第一次每日 routine，驗證任務能正確寫入 Google Calendar（`create_event`）並記錄進 `progress-log.json`。

## 對話中已收集但尚未寫入檔案的資料
（目前沒有遺漏項——所有已收集到的回答都已寫入對應 JSON 檔案，只剩上面「還沒做完」第1項仍在等待使用者回覆。）

## 待確認的背景決策（供繼續對話時參考）
完整定案在 [decisions.md](../decisions.md)，關鍵幾項：
- 單一身分、職業/技能範疇（暫不含生活習慣）
- 現況評估與每月進度確認都依賴使用者的「現有資料/筆記」，不是問卷
- 每日任務具體可執行、自動排程、未完成自動順延並提高優先度
- 月度調整只提建議、需使用者確認才套用；月度報告主動推送通知
- Google Calendar 沒有「建立新日曆」API，已用使用者手動建立的日曆解決
- 排程失敗自動重試（因事件用唯一 ID 比對更新，重試不會重複建立）
- 支援 `status: active/paused` 暫停機制
