# 07 — IMPLEMENTATION ROADMAP

> 從既有 factor-lab 出發，逐步擴展為完整 Alpha Agent Platform。
> 每階段有可 demo 的成果，每階段可獨立停止而不浪費前期投入。

---

## 1. 起點：現有資產盤點

在開始之前，先確認我們已經有什麼：

| 資產 | 狀態 | 註解 |
|------|------|------|
| **Debate pipeline**（proposer→builder→skeptic→empiricist→arbiter→librarian） | ✅ 已跑通一次完整 e2e | Stage 0-2 已覆蓋 |
| **SQLite index.db**（8 tables, 含 factor_registry, paper_insights, evidence 等） | ✅ 已建表、已有資料 | |
| **Obsidian vault**（9+1 類、10,000+ 條目） | ✅ 既有 | |
| **CryptoAdapter**（Binance → Polars） | ✅ 已驗證 | |
| **回測引擎**（Polars + numpy） | ✅ 已驗證 | IC-IR 計算、分組回報 |
| **5 agent skills**（proposer / builder / skeptic / empiricist / librarian） | ✅ 已有 | 已在 factor-lab 中註冊 |
| **debate_flow.py** state machine | ✅ 已測試 | status / prepare / done / reset |
| **OpenRouter API**（Opus 4 / Sonnet 4） | ✅ 已配置 | |
| **Hermes-agent cronjob** | ✅ 已設定 | 5-min tick + daily TST |
| **SuperTrend 因子**（ST(10,3)+ADX SH=1.40） | ✅ 已實證 | 基準因子 |
| **HTML 互動報表**（Plotly 暗色主題） | ✅ 既有模板 | |

### 1.1 缺少什麼

| 缺少的 | 優先級 | 估時 |
|--------|--------|------|
| Agent model routing（每個 agent 用不同模型） | 🔴 高 | 2-3h |
| orchestrator agent（任務調度） | 🔴 高 | 3-4h |
| arbiter agent（目前是手動裁決） | 🔴 高 | 3-4h |
| Soul Questions 整合到 proposer | 🔴 高 | 2-3h |
| Stage 3 Full ADRS backtest（PBO/CV/WF/Sensitivity） | 🟡 中 | 8-12h |
| data_collector agent（自動資料同步） | 🟡 中 | 3-4h |
| portfolio_manager agent（配權重） | 🟢 低 | 3-4h |
| Stage 4A Paper Trade | 🟢 低 | 8-12h |
| Stage 4B/5 Live | ⚪ 未來 | TBD |

---

## 2. 分階段路線

### Phase 1 — 核心閉環完善（本週）

**目標**：讓既有 debate pipeline 全自動化 — 包括 arbiter（Opus 4 自動裁決）和 orchestrator（自然語言指令轉 agent 任務）。

| # | 任務 | 估時 | 產出 |
|---|------|------|------|
| 1.1 | **Model routing 實作**：為每個 agent skill 加上 model 參數 | 2h | proposer/arbiter → Opus 4, builder/empiricist → Sonnet 4, librarian → V4 Flash |
| 1.2 | **arbiter agent skill**：建立 `factor-lab-arbiter` skill | 3h | 自動裁決（PASS/REJECT/REWORK），含完整理由 |
| 1.3 | **Soul Questions 整合**：擴充 `factor-lab-proposer` skill，加入 6 題必答 | 2h | proposer 產出的候選筆記含 Soul Questions |
| 1.4 | **orchestrator agent skill**：建立接收自然語言、解析意圖、調度 debate 的 skill | 3h | 「幫我研究一個基於 funding rate 的均值回歸因子」→ 自動跑完整 debate |
| 1.5 | **端到端測試**：用 orchestrator 觸發一次完整 debate | 2h | 確認 model routing 正確、arbiter 自動裁決、librarian 自動歸位 |
| 1.6 | **arbiter 一致性測試**：對已知結果的因子（如已淘汰的 F3）跑裁決，確認 arbiter 給出相同結論 | 1h | F3 被 REJECT，理由與人類之前裁決一致 |

**Phase 1 完成標準**：
- 說一句話 → 自動跑完整 debate（proposer → builder → skeptic → empiricist → arbiter → librarian）
- arbiter 的裁決與人類之前對 F1/F2/F3 的裁決一致
- 每個 agent 使用正確的模型（確認 cost 記錄中有 Opus/Sonnet/Flash 三種）

**Phase 1 總估時**：~13 小時（約 3-4 個晚上）

---

### Phase 2 — 回測品質升級（下週）

**目標**：補上 Stage 3 Full ADRS backtest，確保過擬合防護到位。

| # | 任務 | 估時 | 產出 |
|---|------|------|------|
| 2.1 | **Purged Cross-Validation**：用 `TimeSeriesSplit` 實作 5-fold purged CV | 3h | 函數 `purged_cv(df, factor_fn, n_splits=5)` |
| 2.2 | **PBO 實作**：López de Prado (2016) 的組合方法 | 3h | PBO score |
| 2.3 | **Walk-Forward**：expanding/rolling window runner | 3h | IS/OOS Sharpe 對照 |
| 2.4 | **Sensitivity Analysis**：optuna + Monte Carlo parameter grid | 3h | 參數敏感度熱圖 |
| 2.5 | **OLS Filter**：對既有因子做 residual alpha 計算 | 2h | 獨立 alpha 的顯著性 |
| 2.6 | **整合到 debate pipeline**：在 empiricist 階段之後自動跑 ADRS（當 Stage 3 啟用時） | 2h | debate_flow.py 支援 stage_3 |

**Phase 2 完成標準**：
- 對既有 SuperTrend 因子跑完整 ADRS，產出報告
- PBO < 0.5、CV Sharpe 穩定、WF 衰減 < 50%

**Phase 2 總估時**：~16 小時（約 5-6 個晚上）

---

### Phase 3 — 資料與監控自動化

**目標**：建立 data_collector + portfolio_manager，讓系統有自我維護能力。

| # | 任務 | 估時 | 產出 |
|---|------|------|------|
| 3.1 | **data_collector skill**：建立自動拉取 OHLCV + 補洞 + health check 的 agent | 3h | cronjob 定時執行 |
| 3.2 | **資料健康儀表板**（HTML） | 2h | 各 symbol 覆蓋率、缺失日、最新 timestamp |
| 3.3 | **portfolio_manager skill**：基於 Sharpe + correlation 的配權邏輯 | 3h | 自動配權重、總曝險控制 |
| 3.4 | **退場監控**：rolling Sharpe decay detection | 2h | 自動標記衰減策略 |

**Phase 3 完成標準**：
- `data_collector` cronjob 自動同步資料，異常時回報
- `portfolio_manager` 能讀取 validated 因子，計算建議權重

**Phase 3 總估時**：~10 小時（約 3-4 個晚上）

---

### Phase 4 — Paper Trade & Beyond（未來）

**目標**：Stage 4A/4B 的真實市場驗證。

這部分先列需求，不設時間線 — 前提是 Phase 1-3 都有產出有價值的因子。

| # | 任務 | 說明 |
|---|------|------|
| 4.1 | Paper trade daemon | 用 live OHLCV + simulated fills，每日 PnL 記錄 |
| 4.2 | Stage 4A Gate | 4 週後 arbiter 審核 paper/backtest Sharpe 衰減 |
| 4.3 | Small capital execution | 真金 ≤ $1,000，真實交易所 API |
| 4.4 | Live monitoring | Rolling Sharpe + drawdown + 自動退場 |

---

## 3. Phase 1 詳細計畫

Phase 1 是最關鍵的 — 它讓既有 debate pipeline 從半自動變成全自動。

### 3.1 任務 1.1：Model Routing

為既有 5 個 agent skill 加上 model 參數：

| Skill | 目前 | 改為 | 理由 |
|-------|------|------|------|
| `factor-lab-proposer` | 無指定（用 default） | `model: openrouter/claude-opus-4` | 推理 + 探索需要最強模型 |
| `factor-lab-builder` | 無指定 | `model: openrouter/claude-sonnet-4` | Code 生成 Sonnet 足夠 |
| `factor-lab-skeptic` | 無指定 | `model: openrouter/claude-sonnet-4` | 審核執行類 |
| `factor-lab-empiricist` | 無指定 | `model: openrouter/claude-sonnet-4` | 計算執行類 |
| `factor-lab-librarian` | 無指定 | `model: deepseek/deepseek-v4-flash` | 整理歸檔，最便宜 |

實作方式：在每個 skill 的 system prompt 開頭加上 `model:` 指示，
或在 `delegate_task` 調用時傳遞 model 參數。

### 3.2 任務 1.2：Arbiter Agent Skill

建立新 skill `factor-lab-arbiter`，取代目前手動裁決。

核心 prompt：
```
你是因子裁決者（arbiter）。你的工作只有一個：綜合所有證據，做出 pass/reject 裁決。

你收到的輸入包括：
1. 候選筆記（含 Soul Questions + hypothesis）
2. Skeptic 五維批判 review
3. Empiricist 實證報告

裁決條件（全部必須滿足才 PASS）：
- Soul Questions 6 題全答
- 五維批判無致命缺陷（無前視偏差、無高資料探勘風險）
- IC-IR ≥ 0.3
- Rank-IC 均值 ≥ 0.02
- 與既有因子相關性 < 0.7

輸出格式：
```yaml
verdict: PASS | PASS_WITH_NOTES | REWORK | REJECT
reasoning: |
  完整理由...
conditions: (僅 REWORK/REJECT)
  - 具體條件
```
```

### 3.3 任務 1.3：Soul Questions 整合

擴充 `factor-lab-proposer` skill：

在 system prompt 中加入 Soul Questions 6 題作為必答項目。
proposer 產出的候選筆記必須包含此 6 題，否則 arbiter Gate 0 直接 REJECT。

### 3.4 任務 1.4：Orchestrator Agent

建立新 skill `factor-lab-orchestrator`：

核心 prompt（參考 [02-AGENTS.md](./02-AGENTS.md) 中 orchestrator 的規範）。

觸發流程：
```
人類輸入：「幫我研究一個基於 funding rate 的均值回歸因子」

orchestrator (Opus 4):
  1. 解析意圖：「研究新因子」
  2. 調度順序：
     - delegate_task(factor-lab-proposer, context="funding rate mean reversion in crypto perps")
     - 等待結果
     - delegate_task(factor-lab-builder, context=<proposer 產出>)
     - 等待結果
     - ... (依序調度所有 agent)
  3. 彙總所有結果，回報使用者：「已完成。驗證通過 X 個因子、淘汰 Y 個因子。詳細報告在 vault。」
```

### 3.5 任務 1.5：端到端測試

用一個已知結果的因子（如 F3 Volatility Roughness，已被淘汰）觸發完整流程：

1. 開啟新的 debate session
2. orchestrator 觸發 proposer → 應正確提 3 個因子
3. builder → skeptic → empiricist 依序執行
4. arbiter → 應 REJECT F3（原因：IC-IR -0.203 < 門檻 0.3）
5. librarian → 應歸入 deprecated/

**預期結果**：arbiter 的裁決與人類之前一致。若不一致 → 調整 arbiter prompt。

---

## 4. 成本預估（Phase 1）

| Agent | 模型 | 每次呼叫 token 估 | 每次成本估 | 月呼叫次數估 |
|-------|------|-------------------|-----------|-------------|
| orchestrator | Opus 4 | ~2K input + ~500 output | ~$0.07 | 10-30 |
| proposer | Opus 4 | ~5K input + ~2K output | ~$0.25 | 10-20 |
| builder | Sonnet 4 | ~3K input + ~3K output | ~$0.05 | 10-20 |
| skeptic | Sonnet 4 | ~4K input + ~1K output | ~$0.03 | 10-20 |
| empiricist | Sonnet 4 | ~2K input + ~1K output | ~$0.02 | 10-20 |
| arbiter | Opus 4 | ~4K input + ~1K output | ~$0.15 | 10-20 |
| librarian | V4 Flash | ~2K input + ~500 output | ~$0.001 | 10-20 |

**每月 total**：$50-100（假設每月 10-20 次 debate）

---

## 5. 成功指標（Phase 1 後）

| 指標 | 目標 | 量測方式 |
|------|------|---------|
| 全自動 e2e 成功率 | ≥ 90% | 10 次 triger 中 9 次完成全流程 |
| Arbiter 一致性 | ≥ 80% | 與人類裁決的對照 |
| Model routing 正確率 | 100% | agent_calls 表中 model 欄位正確 |
| 單次 debate 總成本 | < $1.50 | 加總所有 agent 呼叫的 cost_usd |
| 單次 debate 總時間 | < 30 min | 從 orchestrator 觸發到 librarian 完成 |

---

## 6. 潛在風險與緩解

| 風險 | 機率 | 衝擊 | 緩解 |
|------|------|------|------|
| Arbiter Opus 4 裁決不一致 | 中 | 中 | 每次 debate 後人工抽檢；可加 second-pass arbiter |
| Soul Questions 被 proposer 敷衍 | 中 | 中 | Arbiter Gate 0 專注審 Soul Questions 品質 |
| Opus 4 成本超預期 | 低 | 中 | 設每日成本上限 $10；超標自動降級到 Sonnet 4 |
| Model routing 配置錯誤 | 低 | 高 | Phase 1.1 完成後立刻檢查 agent_calls 表 |
| 既有 skill 不相容新 model | 低 | 低 | 先用 Opus 4 + Sonnet 4 測試既有 prompt，必要時微調 |

---

## 7. 放棄條件

如果 Phase 1 完成後出現以下任一情況，**暫停並重新評估方向**：

| 觸發 | 行動 |
|------|------|
| Arbiter 與人類裁決一致性 < 50% | 檢討 arbiter prompt 或回到手動裁決 |
| 單次 debate 成本 > $3.00 | 把部分 agent 降級（proposer → Sonnet 4 試試） |
| 全自動 e2e 成功率 < 50% | 回到半自動（orchestrator 只調度，人類介入審核） |
| 連續 3 次 debate 所有因子都被 arbiter REJECT | proposer 的假設品質有問題，回頭調整 |

**重新檢視不是放棄，是簡化。砍 30% 範圍繼續。**

---

## 8. Phase 1 開工 Checklist

- [ ] OpenRouter API key 已配置、餘額 ≥ $30
- [ ] 既有 5 個 agent skill 已讀取、確認內容
- [ ] debate_flow.py 可正常執行（status / prepare / done / reset）
- [ ] index.db 可正常查詢（factor_registry 等表有資料）
- [ ] vault 路徑正確（`/home/anson/Documents/Obsidian Vault/量化交易/`）
- [ ] CryptoAdapter 可正常取數（BTCUSDT/ETHUSDT daily）
- [ ] Phase 1.1（model routing）先做 — 這是所有後續任務的基礎
- [ ] 第一次 e2e 測試用已知結果的 F3 做 baseline
- [ ] 紀錄每次 debate 的成本與時間
- [ ] 不要追求一次完美，先求跑通再優化

---

## 9. 給未來的自己

這份路線圖的設計原則是「從已有的基礎出發」。我們不需要從零蓋 7 個 agent + PostgreSQL + Docker + Redis，那些是多餘的。

我們已經有 5 個能工作的 agent + debate pipeline + 回測引擎 + vault。
Phase 1 的目標很簡單：把這套半自動流程變成全自動，並確保自動裁決的品質不低於人類。

如果 Phase 1 成功，你就有一個**說一句話就能自動產出因子研究的系統**。
這本身就已經比 99% 的量化研究者更自動化了。

開工吧。

— 從 factor-lab 出發，往 Alpha Agent Platform 前進。
2026-05-26
