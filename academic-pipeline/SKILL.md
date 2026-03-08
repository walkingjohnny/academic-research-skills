---
name: academic-pipeline
description: "Orchestrator for the full academic research pipeline: research -> write -> integrity check -> review -> revise -> re-review -> re-revise -> final integrity check -> finalize. Coordinates deep-research, academic-paper, and academic-paper-reviewer into a seamless 9-stage workflow with mandatory integrity verification, two-stage peer review, and reproducible quality gates. Triggers on: academic pipeline, 學術研究流程, research to paper, 論文 pipeline, 從研究到論文, full paper workflow, 完整論文流程, paper pipeline, 幫我做一篇論文, 從頭到尾寫一篇論文."
metadata:
  version: "2.4"
  last_updated: "2026-03-08"
  depends_on: "deep-research, academic-paper, academic-paper-reviewer"
---

# Academic Pipeline v2.4 — 學術研究全流程調度器

輕量級 orchestrator，管理從研究探索到論文完稿的完整學術 pipeline。不做實質工作，只負責偵測階段、推薦模式、調度 skill、管理轉場和追蹤狀態。

**v2.0 核心改進**：
1. **強制使用者確認 checkpoint** — 每個 stage 完成後必須等待使用者確認才進入下一步
2. **學術誠信驗證** — 論文完成後、送審前必須通過 100% 參考文獻與數據驗證
3. **兩階段審查** — 第一次完整審查 + 修訂後聚焦驗收審查
4. **最終誠信審查** — 修訂完成後再次驗證所有引用和數據 100% 正確
5. **可復現** — 標準化流程，每次執行產生一致的品質保證
6. **過程紀錄** — Pipeline 完成後自動生成「論文生成過程紀錄」PDF，記錄人機協作歷程

## Quick Start

**完整流程（從零開始）：**
```
我想做一篇關於 AI 對高教品保影響的研究論文
```
--> academic-pipeline 啟動，從 Stage 1 (RESEARCH) 開始

**中途進入（已有論文）：**
```
我已經有一篇論文，幫我審查
```
--> academic-pipeline 偵測 mid-entry，從 Stage 2.5 (INTEGRITY) 開始

**修訂模式（收到審稿意見）：**
```
我收到審稿意見了，幫我修改
```
--> academic-pipeline 偵測，從 Stage 4 (REVISE) 開始

**執行結果：**
1. 偵測使用者目前階段與已有材料
2. 推薦每個 stage 的最適 mode
3. 逐 stage 調度對應 skill
4. **每個 stage 完成後主動提示並等待使用者確認**
5. 全程追蹤進度，隨時可查看 Pipeline Status Dashboard

---

## Trigger Conditions

### 觸發關鍵詞

**中文**：學術研究流程, 論文 pipeline, 從研究到論文, 完整論文流程, 幫我做一篇論文, 從頭到尾寫一篇論文, 研究到出版, 全流程論文
**English**：academic pipeline, research to paper, full paper workflow, paper pipeline, end-to-end paper, research-to-publication, complete paper workflow

### 不觸發情境

| 情境 | 應使用的 Skill |
|------|---------------|
| 只需要查資料、做文獻回顧 | `deep-research` |
| 只需要寫論文（不需研究階段） | `academic-paper` |
| 只需要審查一篇論文 | `academic-paper-reviewer` |
| 只需要檢查引用格式 | `academic-paper` (citation-check mode) |
| 只需要轉換論文格式 | `academic-paper` (format-convert mode) |

### Trigger Exclusions

- 如果使用者只需要單一功能（只查資料、只檢查引用），不需要 pipeline，直接觸發對應的 skill
- 如果使用者已經在使用某個 skill 的特定 mode，不要強制進入 pipeline
- pipeline 是可選的，不是必要的

---

## Pipeline Stages (10 Stages)

| Stage | 名稱 | 呼叫的 Skill / Agent | 可用 Modes | 產出物 |
|-------|------|---------------------|-----------|--------|
| 1 | RESEARCH | `deep-research` | socratic, full, quick | RQ Brief, Methodology, Bibliography, Synthesis |
| 2 | WRITE | `academic-paper` | plan, full | Paper Draft |
| **2.5** | **INTEGRITY** | **`integrity_verification_agent`** | **pre-review** | **誠信驗證報告 + 修正後的論文** |
| 3 | REVIEW | `academic-paper-reviewer` | full (含魔鬼代言人) | 5 份審查報告 + Editorial Decision + Revision Roadmap |
| 4 | REVISE | `academic-paper` | revision | Revised Draft, Response to Reviewers |
| **3'** | **RE-REVIEW** | **`academic-paper-reviewer`** | **re-review** | **驗收審查報告：修訂回應檢核 + 殘留問題** |
| **4'** | **RE-REVISE** | **`academic-paper`** | **revision** | **第二次修訂稿（如需要）** |
| **4.5** | **FINAL INTEGRITY** | **`integrity_verification_agent`** | **final-check** | **最終驗證報告（100% 通過方可放行）** |
| 5 | FINALIZE | `academic-paper` | format-convert | Final Paper（預設 MD + DOCX → 問 LaTeX → 確認無誤 → PDF） |
| **6** | **PROCESS SUMMARY** | **orchestrator** | **auto** | **論文生成過程紀錄 MD + LaTeX → PDF（中英文版）** |

---

## Pipeline State Machine

1. **Stage 1 RESEARCH** → 使用者確認 → Stage 2
2. **Stage 2 WRITE** → 使用者確認 → Stage 2.5
3. **Stage 2.5 INTEGRITY** → PASS → Stage 3（FAIL → 修正重驗，最多 3 輪）
4. **Stage 3 REVIEW** → Accept → Stage 4.5 / Minor|Major → Stage 4 / Reject → Stage 2 或結束
5. **Stage 4 REVISE** → 使用者確認 → Stage 3'
6. **Stage 3' RE-REVIEW** → Accept|Minor → Stage 4.5 / Major → Stage 4'
7. **Stage 4' RE-REVISE** → 使用者確認 → Stage 4.5（不再回到審查）
8. **Stage 4.5 FINAL INTEGRITY** → PASS（零問題）→ Stage 5（FAIL → 修正重驗）
9. **Stage 5 FINALIZE** → MD + DOCX → 詢問 LaTeX → 確認 → PDF → Stage 6
10. **Stage 6 PROCESS SUMMARY** → 詢問語言版本 → 生成過程紀錄 MD → LaTeX → PDF → 結束

完整狀態轉換定義見 `references/pipeline_state_machine.md`。

---

## Mandatory User Confirmation Checkpoints

**v2.0 核心規則：每個 stage 完成後必須主動提示使用者並等待確認。**

### Checkpoint 通知模板

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ Stage [X] [名稱] 完成
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

產出物：
• [材料 1]
• [材料 2]

下一步：Stage [Y] [名稱]
目的：[一句話說明]

要繼續嗎？你也可以：
1. 查看目前進度（說「進度」）
2. 調整下一步的設定
3. 暫停 pipeline（隨時可以回來繼續）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Checkpoint 規則

1. **不可自動跳過**：即使上一 stage 結果完美，也必須等待使用者確認
2. **使用者可調整**：在 checkpoint 時使用者可以修改下一步的 mode 或設定
3. **暫停友好**：使用者可以在任何 checkpoint 暫停，下次回來繼續
4. **簡化版提示**：如果使用者說「直接繼續」或「全自動」，之後的 checkpoint 改為簡化版（一行狀態 + 自動繼續），但仍然會通知

---

## Agent Team (3 Agents)

| # | Agent | 角色 | 檔案 |
|---|-------|------|------|
| 1 | `pipeline_orchestrator_agent` | 主調度器：偵測階段、建議 mode、觸發 skill、管理轉場 | `agents/pipeline_orchestrator_agent.md` |
| 2 | `state_tracker_agent` | 狀態追蹤器：記錄已完成階段、已產出材料、revision 循環次數 | `agents/state_tracker_agent.md` |
| 3 | `integrity_verification_agent` | 誠信驗證員：100% 參考文獻/引用/數據驗證 | `agents/integrity_verification_agent.md` |

---

## Orchestrator Workflow

### Step 1: INTAKE & DETECTION

```
pipeline_orchestrator_agent 分析使用者的輸入：

1. 使用者有什麼材料？
   - 無材料           --> Stage 1 (RESEARCH)
   - 有研究資料       --> Stage 2 (WRITE)
   - 有論文草稿       --> Stage 2.5 (INTEGRITY)
   - 有已驗證的論文   --> Stage 3 (REVIEW)
   - 有審查意見       --> Stage 4 (REVISE)
   - 有修訂稿         --> Stage 3' (RE-REVIEW)
   - 有最終稿要轉格式 --> Stage 5 (FINALIZE)

2. 使用者的目標？
   - 完整流程（從研究到出版）
   - 部分流程（只需要某幾個 stage）

3. 判斷進入點，向使用者確認
```

### Step 2: MODE RECOMMENDATION

```
根據進入點和使用者偏好，推薦每個 stage 的 mode：

使用者類型判斷：
- 新手 / 想被引導 --> socratic (Stage 1) + plan (Stage 2) + guided (Stage 3)
- 老手 / 要直接產出 --> full (Stage 1) + full (Stage 2) + full (Stage 3)
- 時間有限 --> quick (Stage 1) + full (Stage 2) + quick (Stage 3)

推薦時說明每個 mode 的差異，讓使用者選擇
```

### Step 3: STAGE EXECUTION

```
呼叫對應的 skill（不自己做事，純粹調度）：

1. 告知使用者即將進入哪個 Stage
2. 載入對應 skill 的 SKILL.md
3. 以推薦的 mode 啟動 skill
4. 監控 stage 完成狀態

完成後：
1. 彙整產出物清單
2. 更新 pipeline state（呼叫 state_tracker_agent）
3. 【強制】主動提示 checkpoint，等待使用者確認
```

### Step 4: TRANSITION

```
使用者確認後：

1. 將上一 stage 的產出作為下一 stage 的輸入
2. 觸發 handoff protocol（已定義在各 skill 的 SKILL.md 中）：
   - Stage 1  --> 2：deep-research handoff（RQ Brief + Bibliography + Synthesis）
   - Stage 2  --> 2.5：將完整論文交給 integrity_verification_agent
   - Stage 2.5 --> 3：將驗證通過的論文交給 reviewer
   - Stage 3  --> 4：將 Revision Roadmap 交給 academic-paper revision mode
   - Stage 4  --> 3'：將修訂稿和 Response to Reviewers 交給 reviewer
   - Stage 3' --> 4'：將新的 Revision Roadmap 交給 academic-paper revision mode
   - Stage 4/4' --> 4.5：將修訂完成的論文交給 integrity_verification_agent（最終驗證）
   - Stage 4.5 --> 5：將驗證通過的最終稿交給 format-convert mode
3. 開始下一 stage
```

---

## Integrity Review Protocol（v2.0 新增）

### Stage 2.5：首次誠信審查（Pre-Review Integrity）

**觸發**：Stage 2 (WRITE) 完成後、Stage 3 (REVIEW) 之前
**目的**：在送審前確保所有參考文獻和數據沒有捏造或錯誤

```
執行步驟：
1. integrity_verification_agent 對論文執行 Mode 1（首次驗證）
2. 驗證範圍：
   - Phase A：100% 參考文獻存在性 + 書目正確性 + 幽靈引用
   - Phase B：≥ 30% 引用脈絡抽查
   - Phase C：100% 統計數據驗證
3. 結果處理：
   - PASS → checkpoint → Stage 3
   - FAIL → 產出修正清單 → 逐筆修正 → 重新驗證修正項
   - 修正後 PASS → checkpoint → Stage 3
   - 3 輪仍 FAIL → 通知使用者，列出無法驗證項
```

### Stage 4.5：最終誠信審查（Post-Revision Final Check）

**觸發**：Stage 4' (RE-REVISE) 或 Stage 3' (RE-REVIEW, Accept) 完成後、Stage 5 (FINALIZE) 之前
**目的**：確認修訂後的論文 100% 正確，可以出版

```
執行步驟：
1. integrity_verification_agent 對修訂稿執行 Mode 2（最終驗證）
2. 驗證範圍：
   - Phase A：100% 參考文獻驗證（含修訂過程中新增的）
   - Phase B：100% 引用脈絡驗證（非抽查，全查）
   - Phase C：100% 統計數據驗證
3. 特別檢查：比對 Stage 2.5 的結果，確認之前的問題已修正
4. 結果處理：
   - PASS（零問題）→ checkpoint → Stage 5
   - FAIL → 修正 → 重新驗證 → PASS → Stage 5
5. **必須 PASS 且零問題才能進入 Stage 5**
```

---

## Two-Stage Review Protocol（v2.0 新增）

### Stage 3：第一次審查（Full Review）

- **輸入**：通過誠信審查的論文
- **審查團隊**：EIC + R1（方法論）+ R2（領域）+ R3（跨領域）+ Devil's Advocate
- **產出**：5 份審查報告 + Editorial Decision + Revision Roadmap + Socratic Revision Coaching
- **Decision 分支**：Accept → Stage 4.5 / Minor|Major → Revision Coaching → Stage 4 / Reject → Stage 2 或結束

審查流程細節見 `academic-paper-reviewer/SKILL.md`。

### Stage 3 → 4 轉場：Revision Coaching

EIC 以蘇格拉底式對話引導使用者理解審查意見並規劃修訂策略（最多 8 輪）。使用者可說「直接幫我改」跳過。

### Stage 3'：第二次審查（Verification Review）

- **輸入**：修訂稿 + Response to Reviewers + 原始 Revision Roadmap
- **模式**：`academic-paper-reviewer` re-review mode
- **產出**：修訂回應對照表 + 新問題清單 + 新 Editorial Decision
- **Decision 分支**：Accept|Minor → Stage 4.5 / Major → Residual Coaching → Stage 4'

驗收審查流程見 `academic-paper-reviewer/SKILL.md` Re-Review Mode。

### Stage 3' → 4' 轉場：Residual Coaching

EIC 引導使用者理解殘留問題並取捨（最多 5 輪）。使用者可說「直接改」跳過。

---

## Mid-Entry Protocol

使用者可以從任何 stage 進入。orchestrator 會：

1. **偵測材料**：分析使用者提供的內容，判斷已有什麼
2. **確認缺口**：檢查進入該 stage 需要什麼前置材料
3. **建議補做**：如果缺少關鍵材料，建議是否需要回到前面的 stage
4. **直接進入**：如果材料足夠，直接開始指定 stage

**重要：mid-entry 不可跳過 Stage 2.5**
- 如果使用者帶著論文直接進入，先進 Stage 2.5 (INTEGRITY) 再到 Stage 3 (REVIEW)
- 唯一例外：使用者能提供之前的誠信驗證報告且內容未修改

---

## Progress Dashboard

使用者隨時可以說「進度」「status」「pipeline 狀態」查看：

```
+=============================================+
|   Academic Pipeline v2.0 Status             |
+=============================================+
| Topic: AI 對高等教育品質保證的影響           |
+---------------------------------------------+

  Stage 1   RESEARCH          [v] Completed
  Stage 2   WRITE             [v] Completed
  Stage 2.5 INTEGRITY         [v] PASS (62/62 refs verified)
  Stage 3   REVIEW (1st)      [v] Major Revision (5 items)
  Stage 4   REVISE            [v] Completed (5/5 addressed)
  Stage 3'  RE-REVIEW (2nd)   [v] Accept
  Stage 4'  RE-REVISE         [-] Skipped (Accept)
  Stage 4.5 FINAL INTEGRITY   [..] In Progress
  Stage 5   FINALIZE          [ ] Pending
  Stage 6   PROCESS SUMMARY   [ ] Pending

+---------------------------------------------+
| Integrity Verification:                     |
|   Pre-review:  PASS (0 issues)              |
|   Final:       In progress...               |
+---------------------------------------------+
| Review History:                             |
|   Round 1: Major Revision (5 required)      |
|   Round 2: Accept                           |
+=============================================+
```

輸出模板見 `templates/pipeline_status_template.md`。

---

## Revision Loop Management

- Stage 3 (首次審查) → Stage 4 (修訂) → Stage 3' (驗收審查) → Stage 4' (再修訂，如需要) → Stage 4.5 (最終驗證)
- **最多 1 輪 RE-REVISE**（Stage 4'）：如果 Stage 3' 判 Major，進入 Stage 4' 修訂後直接進 Stage 4.5（不再回到審查）
- **Pipeline 下覆蓋 academic-paper 的 max 2 revision 規則**：Pipeline 中修訂只有 Stage 4 + Stage 4'（各一輪），取代 academic-paper 的 max 2 rounds 規則
- 將未解決問題標記為 Acknowledged Limitations
- 提供累計的 revision history（每輪的 decision、處理項目數、未處理項目）

---

## Reproducibility（可復現性）

v2.0 的設計確保每次執行產生一致的品質保證：

### 標準化流程

| 保證項目 | 機制 |
|---------|------|
| 每次都會做誠信審查 | Stage 2.5 + Stage 4.5 是**強制** stage，不可跳過 |
| 審查角度一致 | EIC + R1/R2/R3 + Devil's Advocate 五角度固定 |
| 驗證方法一致 | integrity_verification_agent 使用標準化搜尋模板 |
| 品質閾值一致 | 誠信審查 PASS/FAIL 標準明確（零 SERIOUS + 零 MEDIUM） |
| 流程可追溯 | 每個 stage 的產出都有記錄，可回溯審計 |

### Audit Trail

Pipeline 結束時，state_tracker_agent 產出完整的審計軌跡：

```
Pipeline Audit Trail
====================
Topic: [主題]
Started: [時間]
Completed: [時間]
Total Stages: [X/9]

Stage 1 RESEARCH: [mode] → [產出數]
Stage 2 WRITE: [mode] → [字數]
Stage 2.5 INTEGRITY: [PASS/FAIL] → [refs verified] / [issues found → fixed]
Stage 3 REVIEW: [decision] → [items count]
Stage 4 REVISE: [items addressed / total]
Stage 3' RE-REVIEW: [decision]
Stage 4' RE-REVISE: [executed / skipped]
Stage 4.5 FINAL INTEGRITY: [PASS/FAIL] → [refs verified]
Stage 5 FINALIZE: Ask format style → MD + DOCX + LaTeX (apa7/ieee/etc.) → tectonic → PDF
Stage 6 PROCESS SUMMARY: Ask language → MD → LaTeX → PDF (zh/en)

Integrity Summary:
  Pre-review: [X] refs checked, [Y] issues found, [Y] fixed
  Final: [X] refs checked, [Y] issues found, [Y] fixed
  Overall: [CLEAN / ISSUES NOTED]
```

---

## Stage 6: Process Summary Protocol（v2.4 新增）

**觸發**：Stage 5 (FINALIZE) 完成後
**目的**：記錄人機協作的完整論文生成歷程，供使用者分享、報告或反思

### 流程

```
1. 詢問使用者語言偏好：
   「要先生成哪個語言版本的過程紀錄？」
   - 中文版（繁體中文）
   - 英文版
   - 兩個都要（預設先生成使用者對話主語言版本）

2. 回顧 session 歷史，彙整以下資訊：
   - 使用者的初始指令（原文引用）
   - 每個 stage 的關鍵決策點與使用者介入
   - 方向修正的時刻與原因
   - 迭代次數與審查結果摘要
   - 使用者提出的智識洞察（如催生新章節的提問）
   - 品質要求的演進（如格式、語感調整）
   - Pipeline 統計數據（stage 數、審查輪數、誠信驗證次數等）

3. 生成 Markdown 版本（paper_creation_process.md / paper_creation_process_en.md）

4. 轉為 LaTeX 並編譯 PDF：
   - pandoc MD → LaTeX body
   - 包裝完整 LaTeX 文件（含封面、目錄、頁首頁尾）
   - tectonic 編譯 PDF
   - 中文版需載入 xeCJK + Source Han Serif TC VF
```

### 過程紀錄必含內容

| 區段 | 內容 |
|------|------|
| 論文資訊 | 標題、最終產出物清單 |
| 各階段過程 | 每個 stage 的輸入/產出/關鍵決策，引用使用者原文 |
| 迭代細節 | 審查意見摘要、修訂項目、re-review 結果 |
| 互動模式總結 | 使用者角色、Claude 角色、介入次數、關鍵轉折點等統計表 |
| 使用者關鍵決策 | 按時序列出使用者做的每個重要決定 |
| 關鍵教訓 | 從過程中學到的可複用經驗 |
| **協作品質評估** | **最後一章：1-100 分評分 + 維度分析 + 改進建議**（見下方） |

### Collaboration Quality Evaluation（最後一章，必含）

過程紀錄的最後一章為「協作品質評估」，以誠實、建設性的語氣評估使用者在本次人機協作中的表現。格式參照 Claude Code CLI 的 `/insight` 功能。

#### 評分維度（每項 1-100，加權平均為總分）

```
┌─────────────────────────────────────────────────┐
│  Collaboration Quality Score: [XX]/100           │
├─────────────────────────────────────────────────┤
│                                                  │
│  Direction Setting          [██████████░░] XX    │
│  清晰度、時機、範圍界定                            │
│                                                  │
│  Intellectual Contribution  [████████████░] XX   │
│  洞察深度、原創提問、概念挑戰                       │
│                                                  │
│  Quality Gatekeeping        [█████████░░░] XX    │
│  視覺檢查、格式要求、品質標準                       │
│                                                  │
│  Iteration Discipline       [██████████░░] XX    │
│  適時修正方向、願意重跑 pipeline、不將就             │
│                                                  │
│  Delegation Efficiency      [███████░░░░░] XX    │
│  何時介入/何時放手、指令精確度、checkpoint 效率       │
│                                                  │
│  Meta-Learning              [████████████░] XX   │
│  將經驗回饋至 skill、要求記憶教訓、流程改進意識       │
│                                                  │
└─────────────────────────────────────────────────┘
```

#### 評分準則

| 分數區間 | 含義 |
|---------|------|
| 90-100 | 卓越——使用者的介入顯著提升了論文的智識品質，超越 AI 獨立產出的水準 |
| 75-89 | 優秀——使用者做出了正確的方向決策，有效利用了 pipeline 的迭代能力 |
| 60-74 | 良好——使用者完成了必要的決策，但有些機會未被把握 |
| 40-59 | 基本——使用者主要扮演「繼續」按鈕的角色，缺少實質性介入 |
| 1-39 | 待改進——使用者的介入可能干擾了流程或缺少關鍵品質把關 |

#### 必含子區段

1. **Overall Score**：總分 + 一句話評語
2. **What Worked Well**（做得好的）：2-4 項具體行為，引用使用者原文
3. **Missed Opportunities**（錯過的機會）：1-3 項使用者本可以做但沒做的事
4. **Recommendations for Next Time**（下次建議）：3-5 項具體、可操作的改進建議
5. **Human vs AI Value-Add**：明確指出哪些最終論文品質來自使用者介入（非 AI 獨立可達）

#### 評估原則

- **誠實優先**：不灌水、不客氣話。如果使用者只按「繼續」，就如實反映
- **證據為本**：每項評分都引用具體行為或對話紀錄
- **建設性**：批評必須附帶可操作的改進建議
- **承認不確定性**：如果某些維度無法評估（如 mid-entry 跳過研究階段），標註 N/A
- **雙向反思**：也坦誠指出 Claude 在過程中的不足（如需多次修正的地方）

### 輸出規格

- **檔名**：`paper_creation_process.md`（中文）/ `paper_creation_process_en.md`（英文）
- **PDF**：`paper_creation_process_zh.pdf` / `paper_creation_process_en.pdf`
- **LaTeX 模板**：`article` class, 12pt, A4, Times New Roman + Source Han Serif TC VF
- **含目錄**：`\tableofcontents`
- **頁首**：左=文件標題（斜體），右=日期
- **編譯**：tectonic（與 Stage 5 相同工具鏈）

---

## Quality Standards

| 維度 | 要求 |
|------|------|
| 階段偵測 | 正確識別使用者目前所在階段和已有材料 |
| Mode 推薦 | 根據使用者偏好和材料狀態推薦合適的 mode |
| 材料傳遞 | Stage 間的 handoff 材料完整、格式正確 |
| 狀態追蹤 | Pipeline state 即時更新、Progress Dashboard 準確 |
| **強制 checkpoint** | **每個 stage 完成後必須等待使用者確認** |
| **強制誠信審查** | **Stage 2.5 和 4.5 不可跳過，必須 PASS** |
| 不越權 | orchestrator 不做實質研究/寫作/審查，只做調度 |
| 不強制 | 使用者可以隨時暫停或退出 pipeline（但不可跳過誠信審查） |
| 可復現 | 相同的輸入在不同 session 中走相同的流程 |

---

## Error Recovery

| 階段 | 錯誤 | 處理 |
|------|------|------|
| Intake | 無法判斷進入點 | 詢問使用者已有什麼材料和目標 |
| Stage 1 | deep-research 未收斂 | 建議切換 mode（socratic --> full）或縮小範圍 |
| Stage 2 | 缺少研究基礎 | 建議回到 Stage 1 補做研究 |
| Stage 2.5 | 3 輪修正仍 FAIL | 列出無法驗證項，使用者決定是否繼續 |
| Stage 3 | Review 結果為 Reject | 提供選項：重大重構 (Stage 2) 或放棄 |
| Stage 4 | 修訂未完成所有 items | 列出未處理項目，詢問是否繼續 |
| Stage 3' | 驗收仍有 major issues | 進入 Stage 4' 做最後修訂 |
| Stage 4' | 修訂後仍有問題 | 標記為 Acknowledged Limitations，進入 Stage 4.5 |
| Stage 4.5 | 最終驗證 FAIL | 修正後重新驗證（最多 3 輪） |
| Any | 使用者中途離開 | 儲存 pipeline state，下次可從斷點續行 |
| Any | Skill 執行失敗 | 報告錯誤，建議重試或跳過 |

---

## Agent File References

| Agent | Definition File |
|-------|----------------|
| pipeline_orchestrator_agent | `agents/pipeline_orchestrator_agent.md` |
| state_tracker_agent | `agents/state_tracker_agent.md` |
| integrity_verification_agent | `agents/integrity_verification_agent.md` |

---

## Reference Files

| Reference | Purpose |
|-----------|---------|
| `references/pipeline_state_machine.md` | 完整狀態機定義：所有合法轉換、前置條件、動作 |
| `references/plagiarism_detection_protocol.md` | Phase D 原創性驗證協議 + 自我抄襲 + AI 文字特徵 |

---

## Templates

| Template | Purpose |
|----------|---------|
| `templates/pipeline_status_template.md` | Progress Dashboard 輸出模板 |

---

## Examples

| Example | Demonstrates |
|---------|-------------|
| `examples/full_pipeline_example.md` | 完整 pipeline 對話紀錄（Stage 1-5，含 integrity + 2-stage review） |
| `examples/mid_entry_example.md` | 從 Stage 2.5 中途進入的範例（已有論文 → 誠信審查 → 審查 → 修訂 → 完稿） |

---

## Output Language

跟隨使用者語言。學術術語保留英文。

---

## Integration with Other Skills

```
academic-pipeline 調度以下 skills（不自己做事）：

Stage 1: deep-research
  - socratic mode: 引導式研究探索
  - full mode: 完整研究報告
  - quick mode: 快速研究摘要

Stage 2: academic-paper
  - plan mode: 蘇格拉底式逐章引導
  - full mode: 完整論文撰寫

Stage 2.5: integrity_verification_agent (Mode 1: pre-review)
Stage 4.5: integrity_verification_agent (Mode 2: final-check)

Stage 3: academic-paper-reviewer
  - full mode: 完整 5 人審查（EIC + R1/R2/R3 + Devil's Advocate）

Stage 3': academic-paper-reviewer
  - re-review mode: 驗收審查（聚焦修訂回應）

Stage 4/4': academic-paper (revision mode)
Stage 5: academic-paper (format-convert mode)
  - Step 1：詢問使用者要用哪種學術格式（APA 7.0 / Chicago / IEEE 等）
  - Step 2：自動產出 MD + DOCX
  - Step 3：產出 LaTeX（使用對應 document class，如 apa7 class for APA 7.0）
  - Step 4：使用者確認內容無誤後，tectonic 編譯 PDF（最終版）
  - 字體：Times New Roman（英文）+ Source Han Serif TC VF（中文）+ Courier New（等寬）
  - PDF 必須從 LaTeX 編譯（禁止 HTML-to-PDF）
```

---

## Related Skills

| Skill | 關係 |
|-------|------|
| `deep-research` | 被調度（Stage 1 研究階段） |
| `academic-paper` | 被調度（Stage 2 撰寫、Stage 4/4' 修訂、Stage 5 格式化） |
| `academic-paper-reviewer` | 被調度（Stage 3 第一次審查、Stage 3' 驗收審查） |

---

## Version Info

| 項目 | 內容 |
|------|------|
| Skill 版本 | 2.4 |
| 最後更新 | 2026-03-08 |
| 維護者 | HEEACT |
| 相依 Skills | deep-research v2.0+, academic-paper v2.0+, academic-paper-reviewer v1.1+ |
| 角色 | 學術研究全流程調度器 |

---

## Changelog

| 版本 | 日期 | 變更 |
|------|------|------|
| 2.4 | 2026-03-08 | Stage 6 PROCESS SUMMARY: post-pipeline paper creation process record; asks user preferred language (zh/en/both); generates structured MD summarizing full human-AI collaboration history with user quotes, key decisions, iteration details, and lessons learned; mandatory final chapter: **Collaboration Quality Evaluation** (6 dimensions scored 1-100, bar chart visualization, What Worked Well / Missed Opportunities / Recommendations / Human vs AI Value-Add / Claude's Self-Reflection); compiles to PDF via LaTeX + tectonic; outputs `paper_creation_process_zh.pdf` + `paper_creation_process_en.pdf` |
| 2.3 | 2026-03-08 | Stage 5 FINALIZE: mandatory formatting style prompt (APA 7.0 / Chicago / IEEE); PDF must compile from LaTeX via tectonic (no HTML-to-PDF); APA 7.0 uses `apa7` document class (`man` mode) with XeCJK for bilingual support; font stack: Times New Roman + Source Han Serif TC VF + Courier New |
| 2.2 | 2025-03-05 | Checkpoint confirmation semantics (6 user commands with precise actions); mode switching rules (safe/dangerous/prohibited matrix); skill failure fallback matrix (per-stage degradation strategies); state ownership protocol (single source of truth with write access control); material version control (versioned artifacts with audit trail); cross-skill reference to `shared/handoff_schemas.md` |
| 2.1 | 2026-03 | Added plagiarism detection protocol (Phase D); enhanced integrity_verification_agent with originality verification (D1 WebSearch, D2 self-plagiarism); updated both verification modes |
| 2.0 | 2026-02 | 新增 Stage 2.5/4.5 誠信審查、兩階段審查、強制 checkpoint、魔鬼代言人、可復現性保證、integrity_verification_agent |
| 1.0 | 2026-02 | 初版：5+1 stage pipeline |
