# 工作進度

## 目前任務
專案已啟動運作，但**雲端自動化排程（CCR cron trigger）目前整體故障**，已從「完全自動」降級為「使用者開口、Claude 互動處理」的人工觸發模式。本次對話（2026-06-30）主要做了兩件事：(1) 依使用者反饋大幅重新設計學習任務編排邏輯；(2) 排查並驗證雲端排程為何一直失敗，最終確認問題不在我方設計而是平台/帳號層級，暫時放棄全自動化。

## 已完成步驟

### A. 學習編排邏輯重新設計（已 commit+push，定案版本）
1. 第一版修正：把「Rust/Vue 同天混搭」改成「雙軌交替、單一技能聚焦」（commit `d5dcedf`）。
2. 第二版修正（**最終採用版本**）：依使用者進一步反饋，改成「**區塊式單軌前景 + 背景持續深化雙層結構**」（commit `ba473bb`）：
   - **前景 `current_focus`**：同一時間只學一項新技能，區塊式單軌——整條 Vue 技能線（9項：基礎→Composition API→Pinia→Router→TS→元件架構→Nuxt→Vitest→Vite）全部學完，才開始整條 Rust 技能線（9項），整合型技能（DevOps、全端架構、Code Review、開源貢獻）排最後。
   - **背景 `background_queue`**：技能前景畢業後不閒置，先進 `project_application` 階段（實際專案深化），證據導向畢業到 `optimization` 階段（效能/進階研究），之後持續輪替不退出佇列。
   - **每日任務** = 前景 1 個 + 背景最多 1 個（`background_rotation_index` 輪流挑，避免畢業技能越多任務越爆炸）。
   - **完成判斷**：證據導向、非時間導向——讀 `ongoing_notes_ref` 筆記比對是否有實際完成證據，不設固定天數。
   - **目標時程定位**：6 個月 cycle 重新定位為「週期性檢視點」，不是「達成大師級」的截止日。
3. 修正「day-1 不該白跑筆記比對」的邏輯漏洞（commit `7344d8a`）：只有 `assigned_task_ids` 裡存在「今天以前」指派的任務時才需要 clone 筆記，避免新技能剛起步就做無意義的證據查核。
4. 筆記改為依技能分資料夾管理（commit `338c25b`）：`ongoing_notes_ref.scope` 從「整個 vault」改成「依技能分資料夾」（例如 `Vue/`、`Rust/`），每日證據比對只掃對應資料夾，每月確認仍掃全部。
5. 記錄雲端排程已知限制（commit `05adce1`，見下方 C 節）。
6. 手動處理 2026-06-30 當天任務（commit `bf15cd2`）：發現 6/29 的三個 Calendar 事件被某次失敗的自動排程誤取消（status 變成 cancelled），重新建立 Vue 基礎任務的事件（順延，使用者確認尚未完成），更新 `progress-log.json`、`plan.json.current_focus.assigned_task_ids`。

### B. profile.json / plan.json 技能順序最終版（已生效）
`priority_skills` 區塊式順序：Vue 基礎語法 → Vue 3 Composition API → Pinia → Vue Router → TypeScript → 前端元件架構 → Nuxt → Vitest → Vite → Rust 基礎語法 → Rust 所有權/借用 → Rust 錯誤處理 → Async Rust → REST API設計 → Axum → sqlx → JWT/Session → Rust測試 → DevOps → 全端系統架構 → Code Review帶領 → 開源貢獻。

`current_focus` 目前狀態：`skill: "Vue 基礎語法"`，`assigned_task_ids: ["2026-06-30-1"]`，`status: "in_progress"`。`background_queue` 目前是空陣列（還沒有技能畢業）。

### C. 雲端排程除錯全紀錄（最終結論：放棄今日繼續嘗試）
依序試過以下方式，**全部失敗**，已用 `decisions.md` 第7大題記錄：
1. **prompt 內嵌 GitHub PAT**（原始設計，6/29 曾成功 push 過一次）→ 6/30 重試被安全分類器當成「憑證外洩風險」擋下。
2. **完全不給憑證、讓 session 自己判斷**（假設 `sources` 機制會自動提供 push 權限）→ session 自行掃描檔案系統（`/root/.claude.json`、環境變數等）找憑證，被當成「憑證探索」行為擋下。
3. **單純 `git clone`/`git push`，依賴環境內建 proxy**（觀察到 `GITHUB_TOKEN=proxy-injected` 環境變數與 git config 的 proxy 改寫規則）→ 仍然失敗，無 commit、無 calendar 事件。
4. **唯讀模式**（完全不碰 git，只讀 repo 內容 + 寫 Google Calendar）→ 跑了快 40 分鐘無任何結果，確認問題不限於 git push。
5. **最精簡測試**（單一 `PushNotification` 呼叫，完全不碰 GitHub/git/Calendar）→ 連續測試 3 次都沒有推播到使用者手機，**證實問題與 GitHub 權杖完全無關，是 CCR cron trigger 整體執行機制目前對這個帳號/環境失效**。

確認 `future-self-agent` repo 本身是**公開**的（anonymous API 查詢回 200），讀取從未真的需要憑證，只有寫回卡住——但連寫回都不需要的最簡單推播也失敗，所以問題範圍比「git push 被擋」更大。

### D. 雲端 trigger 現況（目前帳號下的 4 個 future-self-agent 相關 trigger）
- `trig_01Ms2KX5ArxGyMT6YQbga1z1`「future-self-agent 每日提醒」：每天 13:01 UTC（21:01 Asia/Taipei），只發 PushNotification 提醒使用者回訊息給 Claude，**測試 3 次均未送達，但先保留**，作為偵測平台是否恢復的探針。
- `trig_018eGPxG51Cz4W837hW2ntLe`「future-self-agent 每月提醒」：每月 1 號 01:00 UTC（09:00 Asia/Taipei），同上邏輯，未測試送達情況。
- 之前建立又刪除的多個版本（PAT 版、無憑證版、唯讀版的每日/每月 trigger）皆已刪除，不留垃圾設定。
- **目前運作模式**：使用者開口（例如「今天的任務」「做每月確認」）時，由 Claude 在互動對話中讀取 `ROUTINES.md` + `plan.json` 現狀，手動產生任務、寫入 Google Calendar、更新 JSON、commit+push（這個互動 session 的 git 環境是正常的，已驗證多次成功）。

## 已寫入的檔案
- `data/profile.json` — `target_identity`（含 `timeframe_note`）、`current_state`、`skill_gaps`（22項，含 `track` 欄位、區塊式 `priority_rank`）、`ongoing_notes_ref`（依技能分資料夾慣例）皆已完成。
- `data/plan.json` — `cycle`（含 `note`）、`priority_skills`（區塊式順序）、`current_focus`、`background_queue`（空）、`background_rotation_index`、`calendar`、`status: "active"` 皆已完成。`history[]` 仍是空陣列。
- `data/progress-log.json` — 已有 2026-06-29、2026-06-30 兩天的紀錄。
- `data/SCHEMA.md` — 已同步更新 `track`、`current_focus`、`background_queue` 等新欄位說明。
- `ROUTINES.md` — 每日規劃 Routine 已改寫為「前景判斷完成→背景判斷換階段→前景出任務→背景出任務」的新邏輯，含 day-1 跳過讀筆記的防呆規則。
- `decisions.md` — 第4大題新增「技能聚焦規則」「技能完成判斷」「目標時程定位」三項決策；第7大題新增雲端排程已知限制與唯讀模式折衷方案的記錄。

## 還沒做完的事
1. **雲端自動化恢復評估**：目前判斷是平台/帳號層級問題，不是設計問題。建議過幾天後重新測試 `trig_01Ms2KX5ArxGyMT6YQbga1z1` 這個最精簡的提醒 trigger 有沒有開始送達，若送達了可以考慮逐步把完整自動化邏輯加回去。
2. **`background_queue` 機制尚未實際驗證**：因為自動化卡住，還沒有任何技能真正透過證據導向判斷從 `current_focus` 畢業進入 `background_queue`，這條路徑的實際行為（尤其是筆記證據比對的判斷品質）還沒被測試過。
3. **每月確認 routine 從未成功執行過**：`plan.json.history[]` 仍是空陣列，沒有驗證過讀筆記評估進度、產出調整建議的實際效果。
4. **使用者目前每天需要主動開口**才能拿到任務（例如說「今天的任務」），這是降級後的運作方式，不是長期理想狀態。
5. 若使用者之後找到/設定好 CCR 環境真正可用的 GitHub 授權方式（例如官方 GitHub App 連接，而非 prompt 內嵌憑證），可以把雲端 routine 改回完整自動寫回模式（見 `decisions.md` 第7大題最後一條）。

## 對話中收集到但僅以口頭/記憶形式存在、需留意是否已寫入的資訊
- 使用者明確表示**目標是「最終達到大師級」優先於時間表**，6 個月 cycle 不是截止日 → 已寫入 `profile.json.target_identity.timeframe_note` 與 `plan.json.cycle.note`，**不需要再問**。
- 使用者確認 **6/29 的 Vue 任務（建立第一個 Vue 專案）截至 2026-06-30 對話當下尚未完成** → 已反映在 `progress-log.json`（2026-06-30 條目為順延任務，`carried_over_from: "2026-06-29"`）。
- 使用者的 Obsidian 筆記未來會**依技能分資料夾**整理（例如 `Vue/`、`Rust/` 資料夾），且**本機與 GitHub 都會有一份** → 已寫入 `profile.json.ongoing_notes_ref.scope` 的慣例說明，但**使用者本機 Obsidian vault 的實際路徑尚未提供**，若之後要在互動中直接讀本機筆記（不透過 git clone），需要詢問使用者本機路徑。
- 使用者手機**有裝 Claude App**，但測試 push 通知 3 次均未送達 → 不確定是 App 通知權限沒開、帳號未連動、還是平台問題，下次互動可詢問使用者是否要協助排查 App 端設定。
- 使用者提到希望用「Loop Engineering」方式（本機 session 自我定速檢查），但因為「電腦不可能一直開著」這個硬限制而放棄，改採人工開口模式 → 這個取捨已在對話中定案，不需要重新討論，除非使用者主動提起。

## 待確認的背景決策（完整定案見 decisions.md）
- 第1-3、5、6、8、9大題：維持先前版本未變動（人格/目標定義、Google Calendar 整合方式、互動通知分級、隱私邊界等）。
- 第4大題：**已更新**為「前景單一聚焦＋背景持續深化」「證據導向完成判斷」「目標時程是檢視點非截止日」三項新決策。
- 第7大題：**已新增**雲端排程已知限制與唯讀模式折衷方案的完整記錄（含三種失敗嘗試的細節），供未來除錯參考。
