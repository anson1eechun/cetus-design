# 03 — PIPELINE

> 從 idea 到 live 的完整七階段管線。
> 既有 debate pipeline 覆蓋 Stage 0-2，Stage 3-5 為未來擴展。

---

## 1. Pipeline 全景

### 1.1 一句話

> 任何 alpha idea 必須走過 7 個 stage、7 個 gate，無例外。每個 gate 由 arbiter（Opus 4）把守，每個淘汰自動歸檔 deprecated。

### 1.2 狀態機總覽

```
                         ┌─────────────────┐
                         │     IDLE         │
                         │  (等待指令)       │
                         └────────┬────────┘
                                  │ orchestrator 解析意圖
                                  ▼
                         ┌─────────────────┐
                         │  STAGE 0        │  ← proposer: Soul Questions + Dedup
                         │  Alpha Idea     │  ← 產出：candidates/<name>.md
                         └────────┬────────┘
                                  │ arbiter Gate 0 審核
                                  ▼
                    ┌──────────────────────────┐
                    │           PASS?           │
                    └─────┬──────────┬─────────┘
                          │ YES      │ NO → deprecated
                          ▼          │
                         ┌─────────────────┐
                         │  STAGE 1        │  ← proposer(深度): 文獻 + 因子探索
                         │  Research       │  ← 產出：hypothesis.md + formula
                         └────────┬────────┘
                                  │ arbiter Gate 1 審核
                                  ▼
                          PASS → continue
                          NO → deprecated | rework (限一次)
                                  │
                                  ▼
                         ┌─────────────────┐
                         │  STAGE 2        │  ← builder: 寫 code
                         │  Factor Dev     │  ← skeptic: 五維批判
                         └────────┬────────┘  ← empiricist: 量化驗證
                                  │ arbiter Gate 2 審核
                                  ▼
                          PASS(IC-IR≥0.3) → continue
                          NO → deprecated
                                  │
                                  ▼
                         ┌─────────────────┐
                         │  STAGE 3        │  ← Full ADRS backtest
                         │  Full Backtest  │  ← PBO + CV + Walk-Forward + Sensitivity
                         └────────┬────────┘
                                  │ arbiter Gate 3 審核
                                  ▼
                          PASS(Sharpe>1.5, PBO<0.5) → continue
                          NO → deprecated
                                  │
                                  ▼
                         ┌─────────────────┐
                         │  STAGE 4A       │  ← Paper trade (≥4週)
                         │  Paper Trade    │  ← Live data, simulated fills
                         └────────┬────────┘
                                  │ arbiter Gate 4A 審核
                                  ▼
                          PASS(paper/backtest Sharpe衰減<50%) → continue
                          NO → deprecated
                                  │
                                  ▼
                         ┌─────────────────┐
                         │  STAGE 4B       │  ← Small capital (≤$1,000)
                         │  Small Capital  │  ← 真實滑點、真實成本
                         └────────┬────────┘
                                  │ arbiter Gate 4B 審核
                                  ▼
                          PASS(4週實盤PnL>0) → continue
                          NO → deprecated
                                  │
                                  ▼
                         ┌─────────────────┐
                         │  STAGE 5        │  ← Live portfolio
                         │  Live           │  ← portfolio_manager 配權
                         └────────┬────────┘
                                  │ rolling monitor
                                  ▼
                          衰減觸發 → deprecated
                          (rolling Sharpe < 0.5 或 drawdown > 30%)
```

---

## 2. Stage-by-Stage 規格

### Stage 0 — Alpha Idea

| 項目 | 內容 |
|------|------|
| **執行 agent** | proposer（Opus 4） |
| **對應 debate** | `proposer` stage |
| **觸發** | orchestrator 收到人類指令 或 cronjob 自主提案 |
| **產出** | `candidates/<factor-name>.md`，含完整 YAML frontmatter |

**proposer 的完整任務**（既有 skill `factor-lab-proposer`）：

1. 讀取 vault 上下文 + paper_insights 表
2. Dedup 檢查：查 `factor_registry` + `deprecated/`
3. 提出因子假說（可證偽假說 + 可執行配方）
4. 回答 Soul Questions 6 題
5. 寫入候選筆記

**Gate 0 審核**（arbiter）：
- Soul Questions 6 題全答 ✅
- Dedup 狀態 CLEAR（無高度重複既有因子）✅
- Q1 有結構性原因（非「市場無效率」）✅
- 若任一不通過 → deprecated

---

### Stage 1 — Research

| 項目 | 內容 |
|------|------|
| **執行 agent** | proposer（Opus 4，深度模式） |
| **觸發** | Gate 0 通過 |
| **產出** | hypothesis.md + factor formula spec + 文獻引用 |

**任務**：
1. 深度文獻探索（arXiv + vault + web_search）
2. 精煉 hypothesis，補充對立假設與邊界條件
3. 產出精確的 factor formula（pseudocode）

**Gate 1 審核**（arbiter）：
- ≥ 2 篇文獻引用 ✅
- Formula 可執行（非模糊的自然語言）✅
- 假說有可證偽條件 ✅
- 若任一不通過 → rework（限一次）或 deprecated

---

### Stage 2 — Factor Dev

| 項目 | 內容 |
|------|------|
| **執行 agent** | builder → skeptic → empiricist |
| **對應 debate** | `builder` → `skeptic` → `empiricist` stages |
| **觸發** | Gate 1 通過 |
| **產出** | `core/signals/F<id>_<name>.py` + skeptic review + empiricist report |

**子階段**：

| 步驟 | Agent | 產出 |
|------|-------|------|
| 2a | builder (Sonnet 4) | Python factor function + smoke test |
| 2b | skeptic (Sonnet 4) | 五維批判 review |
| 2c | empiricist (Sonnet 4) | Rank-IC / IC-IR / quantile returns / corr |

**Gate 2 審核**（arbiter）：
- IC-IR ≥ 0.3 ✅
- Rank-IC 均值 ≥ 0.02 ✅
- 五維批判無致命缺陷（無前視偏差、無高資料探勘風險）✅
- 與既有因子相關性 < 0.7 ✅
- 交易次數 ≥ 30（2y 區間）✅

---

### Stage 3 — Full Backtest

| 項目 | 內容 |
|------|------|
| **執行 agent** | builder + empiricist（深度模式） |
| **觸發** | Gate 2 通過 |
| **產出** | Full ADRS report + equity curve HTML |

**ADRS 四重驗證**：

| 模組 | 內容 | 工具 |
|------|------|------|
| **A — Anti-overfitting** | PBO (López de Prado), purged CV (5-fold) | scikit-learn TimeSeriesSplit + 自建 PBO |
| **D — Distribution** | OLS filter (regress out known factors), residual alpha | scipy OLS |
| **R — Robustness** | Walk-forward (expanding/rolling windows), decay analysis | 自建 WF runner |
| **S — Sensitivity** | Monte Carlo parameter grid, bootstrap confidence bands | numpy random + optuna |

**Gate 3 審核**：
- PBO < 0.5 ✅
- Sharpe > 1.5 ✅
- WF Sharpe 衰減 < 50%（樣本外 vs 樣本內）✅
- 無單一參數過度敏感（±20% 參數變動 Sharpe 波動 < 30%）✅

---

### Stage 4A — Paper Trade

| 項目 | 內容 |
|------|------|
| **執行 agent** | data_collector（提供 live data） + 自建 paper trader |
| **觸發** | Gate 3 通過 |
| **產出** | 即時 simulated PnL + fill 記錄 |
| **期間** | ≥ 4 週 |

**規格**：
- 使用 live OHLCV（Binance 公開 API）
- Simulated fills（bid/ask spread 0.09% per side）
- 每日產出：PnL、Sharpe、max drawdown、win rate

**Gate 4A 審核**：
- 4 週 paper Sharpe / backtest Sharpe 衰減 < 50% ✅
- 4 週 PnL > 0 ✅
- Fill rate ≥ 95%（模擬成交率）✅

---

### Stage 4B — Small Capital

| 項目 | 內容 |
|------|------|
| **觸發** | Gate 4A 通過 |
| **期間** | ≥ 4 週 |
| **資金** | ≤ $1,000 USD |

**規格**：
- 真實交易所執行（需 API key with trade permission）
- 記錄真實滑點（expected vs actual fill price）
- 記錄真實手續費

**Gate 4B 審核**：
- 4 週實盤 PnL > 0 ✅
- 真實滑點 ≤ 2x 模擬滑點 ✅
- 無異常行為（flash crash 時沒爆倉、API error 時正確處理）✅

---

### Stage 5 — Live

| 項目 | 內容 |
|------|------|
| **執行 agent** | portfolio_manager（權重分配） |
| **觸發** | Gate 4B 通過 |
| **產出** | Live portfolio PnL |

**任務**：
- portfolio_manager 依 Sharpe + correlation 配權
- Rolling monitor：每週檢查 Sharpe、drawdown
- 退場觸發：rolling Sharpe < 0.5 或 drawdown > 30% → 自動移除 → deprecated

---

## 3. Debate Pipeline 對照表

既有 `debate_flow.py` 與 Cetus pipeline 的對應關係：

```
debate_flow 階段       Cetus Stage      Gate        Agent
─────────────────      ───────────      ─────       ──────
idle                    IDLE              —           —
prepare (讀取上下文)      —                 —           — (輔助用)
proposer                Stage 0/1        Gate 0/1    proposer (Opus 4)
builder                 Stage 2a          —           builder (Sonnet 4)
skeptic                 Stage 2b          —           skeptic (Sonnet 4)
empiricist              Stage 2c          —           empiricist (Sonnet 4)
arbiter                 Gate 2            Gate 2      arbiter (Opus 4)
librarian               — (歸檔)          —           librarian (V4 Flash)
—                       Stage 3           Gate 3      (未來)
—                       Stage 4A          Gate 4A     (未來)
—                       Stage 4B          Gate 4B     (未來)
—                       Stage 5           —           (未來)
```

**既有 debate pipeline 已完整覆蓋 Stage 0-2 + Gate 0-2。** Stage 3-5 為新增。

---

## 4. State Machine 實作

### 4.1 現有實作（debate_flow.py）

`/home/anson/factor-lab/scripts/debate_flow.py` 管理辯論狀態。

命令：
```bash
python debate_flow.py prepare          # 準備上下文
python debate_flow.py offer            # 手動提案
python debate_flow.py done <step>      # 完成某階段，推進狀態
python debate_flow.py status           # 查看目前狀態
python debate_flow.py reset            # 重置
```

狀態檔：`debate_status.json`

```json
{
  "current_step": "proposer",
  "history": [],
  "candidate_count": 0,
  "active_factors": 2,
  "deprecated_factors": 1,
  "context_hint": "..."
}
```

### 4.2 未來擴展

當 Stage 3+ 實作時，`debate_flow.py` 擴展為完整 pipeline state machine：

```python
VALID_TRANSITIONS = {
    "idle":        ["stage_0"],
    "stage_0":     ["gate_0"],
    "gate_0":      ["stage_1", "deprecated"],
    "stage_1":     ["gate_1"],
    "gate_1":      ["stage_2", "deprecated", "rework_stage_1"],
    "stage_2":     ["gate_2"],
    "gate_2":      ["stage_3", "deprecated"],
    "stage_3":     ["gate_3"],
    "gate_3":      ["stage_4a", "deprecated"],
    "stage_4a":    ["gate_4a"],
    "gate_4a":     ["stage_4b", "deprecated"],
    "stage_4b":    ["gate_4b"],
    "gate_4b":     ["stage_5", "deprecated"],
    "stage_5":     ["deprecated"],  # live → 衰減退場
    "deprecated":  [],               # terminal
}
```

---

## 5. Gate 決策邏輯

每個 gate 有三種輸出：

```
PASS               → 推進到下一 stage
REJECT             → 歸 deprecated（留下完整失敗記錄）
REWORK (限一次)     → 退回前一 stage 重做
```

**REWORK 的限制**：
- 同一個 stage 只能 REWORK 一次
- 第二次 REWORK 自動轉 REJECT
- 防止無限循環燒 LLM 成本

---

## 6. Pipeline 資料流

```
Stage 0-1:         candidates/<name>.md               ← 人類可讀
Stage 2:           core/signals/F<id>_<name>.py       ← 可執行
                   candidates/<name>.md + review      ← 註解追加
Stage 3:           candidates/<name>.md + ADRS report ← 註解追加
                   backtest/*.html                     ← 視覺化
Stage 4A/4B:       data/paper_pnl/<name>.parquet       ← 結構化 PnL
Stage 5:           portfolio_state.json               ← 即時權重
歸檔後:             10-因子库/validated/<name>.md         ← vault
                   或 10-因子库/deprecated/<name>.md
```

---

## 7. 接下來

繼續讀 [04-KNOWLEDGE-BASE.md](./04-KNOWLEDGE-BASE.md) 了解知識庫設計。
