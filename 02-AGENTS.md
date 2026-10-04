# 02 — AGENTS

> 9 個 agent 的完整規範：角色、模型、觸發、system prompt、工具、KPI、失敗模式。

---

## 1. Agent 團隊總覽

| # | Agent | 角色 | 模型 | 決策類型 | 狀態 |
|---|-------|------|------|---------|------|
| 0 | **orchestrator** | 任務調度 | DeepSeek V4 Pro | 推理 + 意圖 | ✅ 即 Hermes Agent 自身 |
| 1 | **proposer** | 因子提案 | Opus 4.7 | 推理 + 探索 | ✅ 已有 skill |
| 2 | **builder** | 因子實作 | Sonnet 4.6 | 執行 + 生成 | ✅ 已有 skill |
| 3 | **skeptic** | 五維批判 | Sonnet 4.6 | 執行 + 審核 | ✅ 已有 skill |
| 4 | **empiricist** | 實證驗證 | Sonnet 4.6 | 執行 + 計算 | ✅ 已有 skill |
| 5 | **arbiter** | 最終裁決 | Opus 4.7 | 推理 + 決策 | ❌ 待新建 |
| 6 | **librarian** | 知識庫管理 | DeepSeek V4 Flash | 執行 + 組織 | ✅ 已有 skill |
| 7 | **data_collector** | 資料同步 | Sonnet 4.6 | 執行 + 監控 | ❌ 待新建 |
| 8 | **portfolio_manager** | 組合管理 | Sonnet 4.6 | 執行 + 計算 | ❌ 待新建 |

---

## 2. 模型路由策略

### 2.1 分層原則

```
      判斷層 (Opus 4.7)      執行層 (Sonnet 4.6)      整理層 (DeepSeek V3)
      ────────────────       ─────────────────      ─────────────────────
      proposer               builder                librarian
      arbiter                skeptic
                             empiricist
                             data_collector
                             portfolio_manager

      調度層 (DeepSeek V4 Pro, 原生 API)
      ──────────────────────────────────
      orchestrator ← 即 Hermes Agent 自身，不需 spawn
```

### 2.2 路由決策樹

```
任務進來
  │
  ├── 需要多步推理、價值判斷、最終決策？
  │     → Opus 4.7（proposer / arbiter）
  │     註：orchestrator 本身就是 DeepSeek V4 Pro，直接處理調度邏輯
  │
  ├── 需要寫 code、做計算、跑回測、審核資料？
  │     → Sonnet 4.6（builder / skeptic / empiricist / data_collector / portfolio_manager）
  │
  └── 需要高頻整理、歸檔、frontmatter 補完？
         → DeepSeek V4 Flash（librarian，原生 API）
```

### 2.3 成本模型

| 模型 | 每 1M token | 適用 agent | 月呼叫次數（估）|
|------|------------|-----------|----------------|
| DeepSeek V4 Pro | DeepSeek 原生 API | 1 (orchestrator，即 Hermes Agent) | ∞（自身推理）|
| `anthropic/claude-opus-4.7` | $5 in / $25 out | 2 (proposer/arbiter) | 50-200 |
| `anthropic/claude-sonnet-4.6` | $3 in / $15 out | 5 (builder/skeptic/empiricist/data_collector/portfolio_mgr) | 100-500 |
| DeepSeek V4 Flash | DeepSeek 原生 API（免費） | 1 (librarian) | 200-1000 |

> 2026-05-26：orchestrator 由 DeepSeek V4 Pro 擔任（Hermes Agent 自身，走 DeepSeek 原生 API，不走 OpenRouter）。
> Librarian 改走 DeepSeek 原生 API V4 Flash（免費），避免 OpenRouter 上 V4 Flash free 版的 429 限流。
> Sonnet 4.6 同價 Sonnet 4（$3/$15），效能達 Opus 4.5 等級 — 執行層性價比極高。
> Opus 4.7 保留給判斷層 proposer/arbiter，確保深度推理品質。

**預估月費**：$15-55（不含 orchestrator 自身成本）

---

## 3. Agent 詳細規範

---

### 3.0 Orchestrator — 任務調度

| 項目 | 內容 |
|------|------|
| **角色** | CEO — 接收人類自然語言指令、解析意圖、調度 agent 執行 |
| **模型** | DeepSeek V4 Pro（DeepSeek 原生 API，即 Hermes Agent 自身）|
| **觸發** | 人類輸入自然語言 |
| **工具** | terminal (spawn hermes)、file、read_file、delegate_task |

#### System Prompt 核心

```
你是 Alpha Agent Platform 的任務調度者（orchestrator）。

你的工作是：
1. 接收人類的自然語言指令（可能是中文或英文）
2. 解析意圖：使用者想做什麼？是研究新因子？回顧舊因子？還是系統狀態查詢？
3. 將意圖拆解為可執行的 agent 任務，按順序調度
4. 每個任務完成後，讀取結果並決定下一步
5. 最終彙總所有結果，以簡潔中文回報給使用者

你可調度的 agent（透過 terminal spawn）：

  proposer (Opus 4.7):
    hermes -s factor-lab-proposer \
      -q '<具體任務描述，含 vault 路徑 /home/anson/Documents/Obsidian Vault/量化交易/>' \
      -m anthropic/claude-opus-4.7 --provider openrouter

  builder (Sonnet 4.6):
    hermes -s factor-lab-builder \
      -q '<具體任務描述，含候選因子路徑>' \
      -m anthropic/claude-sonnet-4.6 --provider openrouter

  skeptic (Sonnet 4.6):
    hermes -s factor-lab-skeptic \
      -q '<具體任務描述，含候選因子路徑和 builder 產出的 code 路徑>' \
      -m anthropic/claude-sonnet-4.6 --provider openrouter

  empiricist (Sonnet 4.6):
    hermes -s factor-lab-empiricist \
      -q '<具體任務描述，含候選因子路徑、因子函數 module 路徑、資料目錄 /home/anson/factor-lab/data/>' \
      -m anthropic/claude-sonnet-4.6 --provider openrouter

  arbiter (Opus 4.7):
    hermes -s factor-lab-arbiter \
      -q '<具體任務描述，含候選筆記路徑（已含 skeptic+empiricist 報告）>' \
      -m anthropic/claude-opus-4.7 --provider openrouter

  librarian (DeepSeek V4 Flash):
    hermes -s factor-lab-librarian \
      -q '<具體任務描述，含裁決後的因子筆記路徑、vault 根路徑>' \
      -m deepseek-chat --provider deepseek

  data_collector (Sonnet 4.6):
    hermes -s factor-lab-proposer \
      -q '執行 data_sync.py 同步 Binance 資料' \
      -m anthropic/claude-sonnet-4.6 --provider openrouter

調度原則：
- 新因子研究 → proposer → builder → skeptic → empiricist → arbiter → librarian
- 狀態查詢 → 直接讀取 debate_status.json 或 index.db
- 每個 spawn 用 terminal(timeout=600)，等結果回來再決定下一步
- 不確定的情況優先問使用者，不要自己猜

輸出格式：以繁體中文說明調度計畫和預計步驟。
```

#### 輸出產物
- agent 調度記錄（寫入 `debate_sessions/`）
- 最終彙總報告（回覆使用者）

#### KPI
- 意圖解析正確率 ≥ 90%（不會把「回顧舊因子」解析成「新建因子」）
- 調度順序正確率 100%

#### 失敗模式
- 調度過多 agent（過度解讀）→ 浪費成本
- 調度不足（跳過必要步驟）→ 結果不完整
- 不問使用者就自己決定參數 → 產出不符合需求

---

### 3.1 Proposer — 因子提案者

| 項目 | 內容 |
|------|------|
| **角色** | Researcher — 提出新 alpha 假設、做深度文獻探索 |
| **模型** | `anthropic/claude-opus-4.7` (OpenRouter) |
| **觸發** | orchestrator 調度 或 cron 定時觸發 |
| **工具** | terminal、file、web_search、web_extract、read_file、write_file、search_files |

### 3.1 Proposer — 提案
#### 核心任務

1. **讀取知識庫上下文**：從 vault + paper_insights 表撈相關論文洞見
2. **檢查既有因子**：避免重複造輪子（查 factor_registry + deprecated）
3. **提出因子假設**：含可證偽假說 + 可執行配方（分離推理與實作）
4. **回答 Soul Questions 6 題**

#### Soul Questions（必答）

```
## Q1 — 為什麼這個 alpha 現在會存在？
[必須答出結構性原因，不能是「backtest 賺錢」或「市場無效率」
 範例：ETH 永續合約的 8h funding rate 結算機制
       造成結算前 2h 有系統性套利壓力]

## Q2 — 什麼會殺死它？
[必須列具體失效條件
 範例：market maker 套利容量增加使結算前溢價消失
       CME ETH futures 推出後現貨與永續價差結構改變]

## Q3 — 對手方是誰？
[必須具名某類交易者，不能寫「散戶」「市場」
 範例：使用被動槓桿的永續多頭（不關注 funding cost）]

## Q4 — 一句話描述
[≤ 30 字
 範例：多空 funding rate z-score > 2 的極端偏離回歸均值]

## Q5 — 資料來源
[列出具體 endpoint / table 名稱
 範例：Binance GET /fapi/v1/fundingRate (public)
       Binance GET /fapi/v1/openInterest (public)]

## Q6 — Alpha 衰減速度
[weeks / months / years / unknown，並說明理由
 範例：months — 結構性資金費套利不會太快消失，
       但隨著做這種策略的人增多，利潤會逐漸收窄]
```

#### 輸出產物

`candidates/<factor-name>.md`，含完整 YAML frontmatter：

```yaml
---
factor_id: F4_Funding_Basis_Mean_Reversion
title: Funding-Basis Residual Mean Reversion
author: proposer
created: 2026-05-26T10:00:00+08:00
status: candidate
direction: long-short
target: ETH-PERP
timeframe: 1h
soul_score: null                 # arbiter 填
dedup_status: clear
related: []
---
```

#### KPI
- Soul Questions 完整度：6 題全答，每題 ≥ 50 字
- hypothesis 可證偽：能說出「如果在資料上看不到 X，就否定這假設」
- Dedup 準確率 ≥ 90%：不會提一個與 deprecated 高度重複的因子
- 每次提案引用 ≥ 2 篇 vault 既有研究

#### 失敗模式
- Soul Questions 用模板式回答敷衍（如 Q3 寫「散戶」）
- hypothesis 太模糊無法實作（builder 看不懂）
- 忘了查 deprecated 就提重複假設
- 假說與配方不分離（推理過程與實作細節混雜）

---

### 3.2 Builder — 因子建構者

| 項目 | 內容 |
|------|------|
| **角色** | CTO — 把因子假設轉換為可執行 Python 函數 |
| **模型** | `anthropic/claude-sonnet-4.6` (OpenRouter) |
| **觸發** | debate state machine 進入 "builder" 階段 |
| **工具** | terminal、file、read_file、write_file、patch |
| **既有 skill** | `factor-lab-builder` |

#### 核心任務

1. **讀取候選筆記**：從 `candidates/<factor-name>.md` 讀取 proposer 的假設與 formula
2. **實作因子函數**：寫入 `core/signals/F<id>_<name>.py`
3. **防前視偏差**：嚴格檢查 — forward returns 計算是否用了未來資料
4. **防缺失值**：因子輸出含明確的 NaN 處理（forward-fill + zero-fill 策略）
5. **排名實作**：跨資產可比性用 cross-sectional rank（rank 而非 raw value）
6. **Smoke test**：用 CryptoAdapter 取內建 BTC/ETH sample data 跑一次驗證

#### 輸出產物

`core/signals/F<id>_<name>.py`，含：

```python
def compute_factor(df: pl.DataFrame) -> pl.Series:
    """
    因子：<因子名稱>

    假設：<一句話假設>
    作者：proposer
    日期：YYYY-MM-DD

    輸入 columns：open, high, low, close, volume, ...
    輸出：[-1, 1] 或 [-∞, ∞] 的標準化訊號
    前視偏差檢查：✅ 無 / ⚠️ 有（請說明）
    缺失值處理：forward-fill max 5 bars → zero-fill
    """
    ...
    return signal
```

#### KPI
- 無前視偏差（skeptic + empiricist 雙重檢查）— 100%
- Smoke test 通過（函數能被呼叫、不回 NaN 陣列、長度正確）
- Code 可讀（skeptic 能看懂邏輯並進行批判）

#### 失敗模式
- 前視偏差（用了當日 close 計算當日訊號）
- 缺失值沒處理（NaN → 後續回測崩潰）
- 把 proposer 的公式寫錯（需要人工 review）

---

### 3.3 Skeptic — 因子質疑者

| 項目 | 內容 |
|------|------|
| **角色** | Gate Keeper（質性審核）— 對候選因子進行五維批判 |
| **模型** | `anthropic/claude-sonnet-4.6` (OpenRouter) |
| **觸發** | debate state machine 進入 "skeptic" 階段 |
| **工具** | terminal、file、read_file、write_file、search_files |
| **既有 skill** | `factor-lab-skeptic` |

#### 五維批判框架

```
維度 1 — 資料探勘風險（Data Mining）
  檢查：參數是否從樣本內多輪調優得來？
        proposer 有沒有提供理論驅動的參數選擇？
        formula 中的 threshold 是否有文獻或市場結構支持？

維度 2 — 經濟直覺（Economic Intuition）
  檢查：為什麼這種市場失靈會持續存在？
        對手方為什麼不消滅它？
        有沒有合理的風險溢酬故事？

維度 3 — 前視偏差（Look-ahead Bias）
  檢查：factor 計算是否用到 t 時點尚不可得的資料？
        forward returns 的對齊是否正確（t+1 而非 t）？
        rolling window 邊界處理是否正確？

維度 4 — 重複造輪子（Reinventing）
  檢查：與 validated/ 和 deprecated/ 中既有因子比較
        是否有高度相關（corr > 0.7）的既有因子？
        與既有因子相比有什麼本質不同？

維度 5 — 成本與流動性（Cost & Liquidity）
  檢查：假設的交易成本是否合理？
        因子方向是否頻繁翻轉（→ 高交易成本）？
        對 ETH/BTC 流動性是否足夠支撐交易？
```

#### 輸出格式

在候選筆記末尾追加：

```markdown
# Skeptic Review

## 五維批判

### 維度 1 — 資料探勘
**評級**：LOW / MEDIUM / HIGH
**理由**：[具體說明]

### 維度 2 — 經濟直覺
**評級**：LOW / MEDIUM / HIGH
**理由**：[具體說明，若 HIGH 必須從 Q1 與 Q3 推導]

### 維度 3 — 前視偏差
**評級**：PASS / FAIL / UNCERTAIN
**理由**：[具體指出程式碼中的潛在問題或確認無誤]

### 維度 4 — 重複造輪子
**評級**：CLEAR / SIMILAR / DUPLICATE
**相似因子**：[列出既有因子]
**差異論證**：[若 SIMILAR，需說明本質不同之處]

### 維度 5 — 成本與流動性
**評級**：LOW / MEDIUM / HIGH
**理由**：[基於 daily rebalance 假設，估算交易成本占比]

## 總評
**是否建議繼續**：YES / YES_WITH_RESERVATION / NO
**致命缺陷**（若有）：[明確指出]
```

#### KPI
- 五維全評，無跳過維度 — 100%
- 對既有因子庫的引用 ≥ 1 個（證明真的有查）
- 前視偏差檢查命中率 ≥ 95%（不誤報、不漏報）

#### 失敗模式
- 所有因子都給 PASS（太寬鬆）→ 失去把關意義
- 所有因子都給 FAIL（太嚴格）→ 阻止潛在 alpha
- 批判只說「可能」而不給具體原因 → 對 arbiter 無用

---

### 3.4 Empiricist — 因子實證者

| 項目 | 內容 |
|------|------|
| **角色** | Data Scientist — 用歷史資料量化驗證因子預測力 |
| **模型** | `anthropic/claude-sonnet-4.6` (OpenRouter) |
| **觸發** | debate state machine 進入 "empiricist" 階段 |
| **工具** | terminal、file、read_file、write_file |
| **既有 skill** | `factor-lab-empiricist` |

#### 核心任務

1. 從 CryptoAdapter 獲取 OHLCV 資料（≥ 2 年歷史）
2. 調用 builder 的 compute_factor 函數計算因子值
3. 對齊 forward returns（N-day fwd return, default N=1）
4. 計算驗證指標

#### 計算指標

| 指標 | 計算方式 | 閾值 |
|------|---------|------|
| Rank-IC 均值 | Spearman rank corr(因子值, fwd_return) 的時序均值 | ≥ 0.02 |
| IC-IR | IC 均值 / IC 標準差 | ≥ 0.3 |
| 分組回報 | 將因子值分 5 組，看 top/bottom quantile 的差異 | top > bottom |
| 相關性 vs 既有因子 | Pearson corr vs validated/ 中所有因子 | < 0.5（避免 redundancy） |
| 累積 IC 曲線 | 視覺化因子預測力的穩定性 | 不長期歸零 |

#### 輸出格式

在候選筆記末尾追加：

```markdown
# Empiricist Report

## 實證結果

| 指標 | 數值 | 閾值 | 通過 |
|------|------|------|------|
| Rank-IC 均值 | 0.043 | ≥ 0.02 | ✅ |
| IC-IR | 0.531 | ≥ 0.3 | ✅ |
| Top Q 回報 | 0.15% | > Bottom Q | ✅ |
| Bottom Q 回報 | -0.08% | — | — |
| vs SuperTrend corr | 0.12 | < 0.5 | ✅ |
| 交易次數（2y） | 85 | ≥ 30 | ✅ |

## 注意事項
- [任何資料異常、期間限制、方向反轉等觀察]
```

#### KPI
- 所有計算可在 ≥ 2 年資料上完成（無崩潰、無 NaN 陣列輸出）
- IC-IR 計算與手算一致（cross-check 通過）
- 報告含所有 5 項指標

#### 失敗模式
- 沒有做 forward return 對齊（前視偏差）
- 因子值與回報方向反轉（需要告知 arbiter）
- 只用 Pearson IC 不用 Rank-IC（對 outlier 敏感）

---

### 3.5 Arbiter — 最終裁決者

| 項目 | 內容 |
|------|------|
| **角色** | Judge — 綜合所有證據，做出 pass/reject 裁決 |
| **模型** | `anthropic/claude-opus-4.7` (OpenRouter) |
| **觸發** | debate state machine 進入 "arbiter" 階段 |
| **工具** | terminal、file、read_file、write_file、patch |

#### 裁決輸入

1. 候選筆記（含 proposer 的 Soul Questions + hypothesis）
2. Skeptic 五維批判
3. Empiricist 實證報告
4. 既有因子庫相關性報告

#### 裁決維度

| 維度 | 權重 | 評分方式 |
|------|------|---------|
| Soul Questions 完整度 | 15% | 6 題全答 + Q1/Q3 有實質內容 |
| 經濟直覺合理性 | 20% | 有結構性原因、有具體對手方 |
| 五維批判結果 | 15% | 無致命缺陷（無 HIGH 級資料探勘 / 前視偏差） |
| 實證結果 | 35% | IC-IR ≥ 0.3 + Rank-IC 顯著 |
| 與既有因子相關性 | 15% | 無 corr > 0.7 的 redundant 因子 |

#### 裁決結果

```
PASS              全條件滿足 → 移入 validated/
PASS_WITH_NOTES   基本條件滿足但有附帶條件 → validated/ + 標註
REWORK            有潛力但某維度不足 → 退回 proposer（限一次）
REJECT            致命缺陷或 IC-IR 不達標 → deprecated/
```

#### 輸出格式

在候選筆記末尾追加：

```markdown
# Arbiter Verdict

## 裁決
**結果**：PASS / PASS_WITH_NOTES / REWORK / REJECT

## 評分矩陣
| 維度 | 評分 | 備註 |
|------|------|------|
| Soul Questions | 5/6 | Q1 結構性原因有說服力 |
| 經濟直覺 | 4/5 | 對手方分析可再加強 |
| 五維批判 | 4/5 | 無致命缺陷 |
| 實證結果 | 4/5 | IC-IR=0.531，顯著 |
| 相關性 | 5/5 | 與既有因子 corr < 0.3 |

## 理由
[≥ 100 字的完整裁決理由]

## 復活條件（僅 REJECT）
[若未來滿足何種條件可重新提案]
```

#### KPI
- 裁決不空洞：每個 REJECT 都有具體理由（不是「實證不達標」就結束）
- 一致性：兩個「相似 factor」不會一個 PASS 一個 REJECT（除非數據差異大）
- 可復現：同一個 factor 給不同 arbiter 同一個裁決（用 Opus 4 確保一致性）

#### 失敗模式
- 只看實證就裁決（忽略 soul questions / 經濟直覺）→ 過擬合
- 給 PASS 但實證根本沒達標 → 把關失職
- 給 REJECT 但實證明顯好 → 漏掉 alpha

---

### 3.6 Librarian — 知識庫管理員

| 項目 | 內容 |
|------|------|
| **角色** | Archivist — 把研究產出整理進 Obsidian vault |
| **模型** | DeepSeek V4 Flash（DeepSeek 原生 API）|
| **觸發** | debate state machine 進入 "librarian" 階段 |
| **工具** | terminal、file、read_file、write_file、patch、search_files |
| **既有 skill** | `factor-lab-librarian` |

#### 核心任務

1. **讀取裁決結果**：從候選筆記取得 arbiter 的最終裁決
2. **歸位因子**：
   - PASS / PASS_WITH_NOTES → `10-因子库/validated/<factor-name>.md`
   - REJECT → `10-因子库/deprecated/<factor-name>.md`
   - REWORK → 保留在 `candidates/`，標記 REWORK
3. **補完 frontmatter**：確認 YAML 前沿所有必填欄位齊全
4. **更新 factor_registry 表**（index.db）
5. **建立 Wikilink**：`derived_from`、`similar_to`、`complements` 等關聯
6. **清理 candidates/**：裁決完成後移除暫存筆記

#### 因子註冊 SQL

```sql
INSERT INTO factor_registry (factor_id, name, status, ic_ir, rank_ic,
    related_factors, validated_at, deprecated_at, notes)
VALUES ('F4_foo', 'Foo Factor', 'validated', 0.531, 0.043,
    '["SuperTrend", "F1_Complexity_Gap"]', '2026-05-26', NULL,
    'arbiter_notes: PASS with standard conditions');
```

#### KPI
- 歸位正確率 100%（PASS 不進 deprecated、REJECT 不進 validated）
- frontmatter 完整度 100%（缺欄位自動補）
- factor_registry 寫入率 100%

#### 失敗模式
- PASS 歸到 deprecated（把好因子埋了）
- 忘了更新 factor_registry（因子搜尋不到）
- Wikilink 指向不存在的目標

---

### 3.7 Data Collector — 資料收集者

| 項目 | 內容 |
|------|------|
| **角色** | Data Engineer — 同步、ingest、監控資料源 |
| **模型** | `anthropic/claude-sonnet-4.6` (OpenRouter) |
| **觸發** | cronjob（定時）或 orchestrator 手動調度 |
| **工具** | terminal、file、web_extract |
| **既有 skill** | `factor-lab-proposer`（data_collector 模式）|
#### 核心任務

1. **定時同步 OHLCV**：從 Binance 公開 API 拉取最新 K 線，寫入 `data/*.parquet`
2. **補洞檢查**：確認沒有遺漏的日期段，若有則自動 backfill
3. **資料健康回報**：定期回報各 symbol 的資料覆蓋率、缺失率
4. **新增 symbol**：當有需要時，自動建立新 symbol 的 Parquet 檔案

#### 資料同步命令（現有）

```bash
# 定時同步（每 5 分鐘）
python /home/anson/quant-btc-gold/sim/fetch_data.py

# 手動補洞
python /home/anson/factor-lab/scripts/data_sync.py --backfill BTCUSDT --days 30
```

#### 輸出產物
- `data/<symbol>_<timeframe>.parquet`（已排序、已去重、已補 NaN）

#### KPI
- 資料覆蓋率 ≥ 99.5%
- 資料延遲 < 10 分鐘（從 exchange timestamp 到本地寫入）
- 補洞成功率 100%（缺失日期能正確拉回）

#### 失敗模式
- 重複寫入（同一天同一 symbol 寫兩次）
- 補洞時 overwrite 正確資料
- 沒檢查 API rate limit → 被 Binance ban

---

### 3.8 Portfolio Manager — 組合管理者

| 項目 | 內容 |
|------|------|
| **角色** | PM — 多策略配權重、控總曝險、風險監控 |
| **模型** | `anthropic/claude-sonnet-4.6` (OpenRouter) |
| **觸發** | 新策略進 Stage 5 或定時 rebalance |
| **工具** | terminal、file、read_file、write_file |
| **既有 skill** | 無（待建置）|
#### 核心任務

1. **配權重**：依 Sharpe ratio + 策略相關性矩陣分配資金
2. **控總曝險**：單策略 ≤ 20%、總曝險 ≤ 80%（可配置）
3. **風險監控**：rolling drawdown alert、sharpe decay detection
4. **退場觸發**：策略連續虧損超過閾值 → 自動移除

#### 配權邏輯（簡化版）

```python
# 基於 Sharpe 的風險平價
weights = sharpe_ratios / sum(sharpe_ratios)
weights = clip(weights, max_per_strategy=0.20)
weights = weights / weights.sum()  # re-normalize

# 若總曝險超過上限，等比例縮放
if weights.sum() > 0.80:
    weights = weights * 0.80 / weights.sum()
```

#### KPI
- 組合 Sharpe > 單策略中位 Sharpe（portfolio 有改善）
- Max drawdown ≤ 30%
- 自動退場反應時間 < 24h（連續虧損觸發後）

#### 失敗模式
- 過度集中（一個策略占 > 50%）
- 相關性算錯（假設獨立但實際相關）
- 退場太慢（虧損擴大才移除）

---

## 4. Agent 間互動協議

### 4.1 檔案傳遞

所有 agent 之間的資訊傳遞透過以下檔案：
- `candidates/<factor-name>.md` — 因子從提案到裁決的全生命週期
- `debate_status.json` — state machine 狀態
- `core/signals/F<id>_<name>.py` — builder 產出的因子函數

### 4.2 調度順序

正常因子研究流程：
```
[人類指令]
    │
    ▼
orchestrator ──→ proposer ──→ builder ──→ skeptic ──→ empiricist ──→ arbiter ──→ librarian
                                                                                     │
                                                                                     ▼
                                                                           [vault 更新]
```

跳過規則：
- 資料同步失敗 → data_collector 阻斷全部後續（沒有資料不能做研究）
- arbiter REJECT → 跳過 librarian 的 validated 路徑，直接歸 deprecated
- arbiter REWORK → 跳回 proposer（限一次，再 REJECT 直接 deprecated）

### 4.3 成本追蹤

每次 agent 呼叫記錄寫入 `index.db` 的 `agent_calls` 表：

```sql
CREATE TABLE IF NOT EXISTS agent_calls (
    id INTEGER PRIMARY KEY,
    agent_name TEXT,
    model TEXT,
    task_type TEXT,
    input_tokens INTEGER,
    output_tokens INTEGER,
    cost_usd REAL,
    duration_seconds REAL,
    created_at TEXT DEFAULT (datetime('now'))
);
```

---

## 5. 接下來

繼續讀 [03-PIPELINE.md](./03-PIPELINE.md) 了解完整 pipeline 流程。
