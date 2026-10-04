# 04 — KNOWLEDGE BASE

> 知識庫設計：vault 結構、frontmatter 規範、檢索機制、分類學。
> 基於既有 Obsidian vault（`量化交易/`），不做重建。

---

## 1. 設計原則

### 1.1 一句話

> 知識庫是整個系統的「長期記憶」— 人類與 agent 共用同一個 Obsidian vault，
> 純 markdown + YAML frontmatter + wikilink，git 版本化，無 vendor lock-in。

### 1.2 三條鐵律

1. **單一真相來源**：任何因子、文獻、回測報告，只有 vault 一份。不複製到 DB。
2. **人類可讀優先**：agent 產出的 .md 要像人手寫的一樣可讀。人類在 Obsidian 中打開要能看懂。
3. **失敗是資產**：deprecated 不刪除。保留完整檔案（hypothesis、failure reason、相關聯結），以供未來 dedup。

### 1.3 為何不用 pgvector / RAG / vector DB

當前 vault 規模（~10,000 條目）：
- `search_files`（ripgrep 全文檢索）+ SQLite FTS5 已經夠用
- Obsidian 內建搜尋人類可以直接使用
- 加 pgvector 的維運成本 > 它帶來的價值

如果需要 semantic search（> 5,000 篇論文時）：
- 採用 `sqlite-vec`（SQLite extension）做本地向量檢索
- 或用 DuckDB array 做 cosine similarity
- 不加外部服務，保持零運維

---

## 2. Vault 結構（沿用現有）

```
/home/anson/Documents/Obsidian Vault/量化交易/
│
├── 01-交易策略/                       ← 策略提案、回測報告
│   ├── SuperTrend-ADX-BTC.md          — 既有因子
│   └── ...                            — 量化交易策略的完整文件
│
├── 02-市场数据/                       ← 資料源、API 文件、資料規格
│   ├── Binance-Futures-API.md
│   └── ...
│
├── 03-回测框架/                       ← 回測引擎設計、工具文件
│   ├── CryptoAdapter-design.md
│   └── ...
│
├── 04-风险管理/                       ← 風控方法、配置方法、ratio 計算
│   ├── Kelly-Criterion-notes.md
│   └── ...
│
├── 05-编程技术/                       ← Python/Polars 技巧、系統架構
│   └── ...
│
├── 06-书籍论文/                       ← 學術論文全文、從業者文章
│   ├── Lopez-de-Prado-Advances-in-Financial-ML.md
│   └── ...（因子提案的文獻來源）    ← paper_insights 表從此處索引
│
├── 07-交易心理/                       ← 思維框架、偏誤 checklist
│   ├── Munger-mental-models.md
│   └── ...
│
├── 08-工具资源/                       ← 外部工具、API、資料源索引
│   └── ...
│
├── 09-作者专题/                       ← 人物/機構專題（Renaissance, Two Sigma...）
│   └── ...
│
├── 10-因子库/                         ← ⭐ 核心：因子資料庫
│   ├── validated/                     ← 已驗證因子
│   │   ├── SuperTrend-ADX.md
│   │   ├── F1-Complexity-Gap.md
│   │   └── F2-Liquidity-Consumption.md
│   └── deprecated/                    ← = graveyard！失敗的因子
│       ├── F3-Volatility-Roughness.md
│       └── ...
│
├── debate_sessions/                   ← 每次辯論的完整記錄
│   └── 2026-05-26-funding-basis/
│       ├── 00-context.md              — 準備階段上下文
│       ├── 01-proposal.md             — proposer 提案
│       ├── 02-builder-output.md       — builder 產出
│       ├── 03-skeptic-review.md       — skeptic 批判
│       ├── 04-empiricist-report.md    — empiricist 實證
│       ├── 05-arbiter-verdict.md      — arbiter 裁決
│       └── 06-librarian-log.md        — librarian 歸檔記錄
│
└── _meta/                             ← vault 系統檔（可選）
    ├── frontmatter-schemas.yaml       — schema 定義
    └── tags.yaml                      — 合法 tag 字典
```

---

## 3. YAML Frontmatter 規範

### 3.1 必備欄位（所有筆記）

```yaml
---
title: "筆記標題"
date: 2026-05-26           # YYYY-MM-DD
type: factor | paper | strategy | note | session
tags: [tag1, tag2]
---
```

### 3.2 因子筆記（validated/ 和 deprecated/）

```yaml
---
title: "Complexity-Gap Regime Filter"
date: 2026-05-26
type: factor
factor_id: F1_Complexity_Gap_Regime
status: validated            # validated | deprecated
direction: long-short
target: BTC-PERP
timeframe: 1d

# 實證指標
ic_mean: 0.043
ic_ir: 0.531
top_q_return: 0.0015
bottom_q_return: -0.0008

# 關聯
derived_from: ["Complexity-Gap Theory (Maillard 2025)"]
similar_to: []
complements: ["F2_Liquidity_Consumption_Tilt"]
corr_with_supertrend: 0.12

# 中繼資料
author: proposer
created: 2026-05-26T10:00:00+08:00
validated_at: 2026-05-26T12:00:00+08:00
deprecated_at: null

# 裁決資訊
arbiter_verdict: PASS
arbiter_notes: "均值回歸版本，方向修正為空頭進場"

tags: [regime-filter, complexity, mean-reversion, btc]
---
```

### 3.3 論文筆記（06-書籍論文/）

```yaml
---
title: "The Probability of Backtest Overfitting"
date: 2016-05-15
type: paper
source: Journal of Investment Strategies
authors: ["Marcos López de Prado", "David H. Bailey"]
doi: 10.2139/ssrn.2326253
year: 2016
tags: [backtest, overfitting, pbo, research-methodology]
---
```

### 3.4 辯論 session 筆記（debate_sessions/）

```yaml
---
title: "Debate Session — Funding Basis Mean Reversion"
date: 2026-05-26
type: session
session_id: 2026-05-26-funding-basis
stages_completed: [proposer, builder, skeptic, empiricist, arbiter, librarian]
outcome: 2 validated, 1 deprecated
tags: [debate, funding-rate, mean-reversion]
---
```

---

## 4. Wikilink 慣例

代理人與人類使用一致的 wikilink 語法：

```markdown
## 關聯因子
- derived_from: [[Complexity-Gap Theory (Maillard 2025)]]
- complements: [[F2-Liquidity-Consumption-Tilt]]
- similar_to: [[SuperTrend-ADX]]
```

**關聯類型**（factor_registry 中結構化儲存、vault 中用 wikilink）：

| 關係 | 意義 | 方向 |
|------|------|------|
| `derived_from` | 此因子從該文獻/理論推導而出 | 單向（文獻不反向連結） |
| `similar_to` | 此因子與該因子的假設/機制相似 | 雙向（互為 similar_to） |
| `complements` | 此因子與該因子組合使用效果好 | 雙向（互為 complements） |
| `replaced_by` | 此因子被該因子取代 | 單向（從 deprecated → validated） |
| `competing` | 此因子與該因子爭取同一 alpha 來源 | 雙向 |

---

## 5. 檢索機制

### 5.1 全文檢索（search_files）

```bash
# agent 搜尋 vault 中的任何文字
search_files(pattern="funding rate mean reversion", path="/home/anson/Documents/Obsidian Vault/量化交易/")
```

### 5.2 結構化查詢（index.db）

```sql
-- 查所有通過 IC-IR ≥ 0.3 的因子
SELECT factor_id, name, ic_ir, rank_ic
FROM factor_registry
WHERE status = 'validated' AND ic_ir >= 0.3
ORDER BY ic_ir DESC;

-- 查與某因子相關的既有因子
SELECT factor_id, name, ic_ir
FROM factor_registry
WHERE factor_id IN (
    SELECT value FROM json_each(related_factors)
    WHERE factor_id = 'F1_Complexity_Gap_Regime'
);

-- Fuzzy dedup：搜尋 deprecated 中的相似描述
SELECT factor_id, name, notes
FROM factor_registry
WHERE status = 'deprecated'
  AND name LIKE '%complexity%';
```

### 5.3 Obsidian 內建（人類用）

- `Ctrl+O`：快速開啟檔案
- Graph view：視覺化因子關聯圖
- 全文搜尋：`Ctrl+Shift+F`

---

## 6. Dedup 機制

### 6.1 新因子提案時的檢查順序

1. **FTS5 全文搜尋**（vault + index.db）→ 找關鍵字相似的既有因子
2. **factor_registry 查詢**→ 找同類別、同標籤的因子
3. **deprecated/ 掃描**→ 找曾被淘汰的類似假設

### 6.2 Dedup 判定

| 相似度 | 判定 | 行動 |
|--------|------|------|
| 假設幾乎相同 | DUPLICATE | 直接拒絕，引用 deprecated |
| 假設相似但方法不同 | SIMILAR | 需在本質差異中論證（proposer 負責） |
| 假設無關 | CLEAR | 放行 |

### 6.3 Dedup 記錄

在 factor_registry 中保留 dedup 記錄：

```sql
-- 每個新因子記錄 dedup 檢查對象
ALTER TABLE factor_registry ADD COLUMN dedup_checked TEXT;
-- JSON array: ["F3.similar", "SuperTrend.clear"]
```

---

## 7. 分類學規則

### 7.1 因子放在哪個目錄

| 狀態 | 目錄 | 進入條件 |
|------|------|---------|
| 提案中 | `candidates/`（factor-lab 目錄下） | proposer 提案後 |
| 已驗證 | `10-因子库/validated/` | arbiter PASS → librarian 歸位 |
| 已淘汰 | `10-因子库/deprecated/` | arbiter REJECT → librarian 歸位 |
| 重做中 | `candidates/` | arbiter REWORK → 留在原處，標記 |

### 7.2 Tag 管理

**Tag 字典**（`_meta/tags.yaml`）：

```yaml
# 合法標籤清單，librarian 在歸檔時做校驗
tags:
  strategy_type: [trend-following, mean-reversion, carry, volatility, momentum, regime-filter]
  asset_class: [btc, eth, crypto-perp, crypto-spot]
  timeframe: [1m, 5m, 15m, 1h, 4h, 1d]
  method: [stat-arb, ml, econometric, market-microstructure]
  risk: [high-sharpe, high-drawdown, low-correlation]
```

**規則**：
- 新 tag 出現時，librarian 可以自動加入 tag 字典
- 每週檢視是否有過度增生（> 5 新 tag / 週 → 警告）
- deprecated 中的 tag 不清除（保留歷史）

---

## 8. Vault 維護

### 8.1 定期任務

| 任務 | 頻率 | 負責 |
|------|------|------|
| Git commit + push | 每天 02:00 | cronjob |
| Frontmatter 完整性檢查 | 每天 03:00 | librarian（V4 Flash） |
| Tag 增生檢查 | 每週日 | librarian |
| Deprecated 數量報告 | 每月初 | librarian → 人類 review |

### 8.2 備份策略

```
Obsidian vault → git repo（私有 GitHub）
               → 每天自動 commit
               → git push 到遠端

index.db       → 隨 vault 一起 git
               → .gitignore 中排除 data/*.parquet（太大）
```

---

## 9. 與 factor-lab 的知識庫整合

現有記憶模組（`core/memory.py`）提供的功能：

| 功能 | 對應 vault 操作 |
|------|----------------|
| `paper_insights` 表 | 從 `06-書籍论文/` 提取的洞見，結構化儲存供 proposer 使用 |
| `factor_registry` 表 | 對應 `10-因子库/` 中的結構化索引 |
| `design_constraints` | 從 validated 因子自動推導的約束條件 |
| `generate_context_prompt()` | 組裝當前 debate 的完整上下文供 agent 使用 |

**關鍵設計**：index.db 是 vault 的「索引」，不是 vault 的「替代」。
所有資料的 primary source 在 vault，index.db 提供結構化查詢加速。

---

## 10. 接下來

繼續讀 [05-UI-DASHBOARD.md](./05-UI-DASHBOARD.md) 了解 UI 設計（保留原文件，僅微調）。
