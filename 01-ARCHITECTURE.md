# 01 — ARCHITECTURE

> 技術棧、目錄結構、運行方式。
> 個人單機、輕量化、零 Docker。

---

## 1. 技術棧

### 1.1 語言與核心依賴

```
Python 3.12          — 主語言
Hermes-agent         — Agent 編排框架（terminal spawn + skill preload）
OpenRouter API       — LLM provider（統一介面，模型對應如下）
Polars 1.x           — 資料處理（5-10x 快於 pandas、lazy eval）
DuckDB 1.x           — 本地 OLAP（Parquet native、零配置）
SQLite               — 結構化儲存（index.db、因子註冊、session 記錄）
CCXT 4.x             — 加密貨幣資料（Binance 公開 API）
```

#### OpenRouter 模型路由（2026-05-26 實測）

```
判斷層 (任務少但須最強推理)
  orchestrator    anthropic/claude-opus-4.7   $5/$25 per 1M  in/out
  proposer        anthropic/claude-opus-4.7   $5/$25
  arbiter         anthropic/claude-opus-4.7   $5/$25

執行層 (任務多、頻率高、需工具操作)
  builder         anthropic/claude-sonnet-4   $3/$15
  skeptic         anthropic/claude-sonnet-4   $3/$15
  empiricist      anthropic/claude-sonnet-4   $3/$15
  data_collector  anthropic/claude-sonnet-4   $3/$15
  portfolio_mgr   anthropic/claude-sonnet-4   $3/$15

整理層 (高頻但低智力需求)
  librarian       deepseek/deepseek-chat       $0.50/$2  (V3,歸檔專用)
```

> 備註：delegate_task 無法為不同 agent 指定不同模型，因此採用
> `terminal spawn` 方式 — orchestrator 透過 `hermes -s <skill> -q '...' -m <model> --provider openrouter`
> 啟動獨立 session，每個 agent 有自己的模型。

### 1.2 不做什麼

| 不做 | 原因 |
|------|------|
| PostgreSQL + TimescaleDB | DuckDB on Parquet 對個人級別資料量已經足夠；單機 OLAP 不需要 hypertable |
| Redis | 單機無分散式 pub/sub 需求；agent 之間透過檔案 + Hermes state machine 傳遞狀態 |
| FastAPI + Next.js | 目前 terminal + HTML 互動報表（Plotly）就夠用；未來有需要再加 |
| Docker Compose | 增加一層運維成本但沒有帶來收益；venv 直接跑即可 |
| Prometheus + Grafana + Loki | 單機直接 tail log 清楚更快；機器只有自己用 |
| Prefect / Airflow | Hermes-agent cronjob 已提供定時任務排程 |

---

## 2. 系統全景

```
                           ┌──────────────────────┐
                           │      User (Terminal)  │
                           │  natural language cmd │
                           └──────────┬───────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      Hermes-agent (Orchestration)                    │
│                                                                      │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐    ┌──────────┐   │
│   │orchestra-│────→│ proposer │────→│ builder  │───→│ skeptic  │   │
│   │   tor    │     │ (Opus 4) │     │(Sonnet 4)│    │(Sonnet 4)│   │
│   │ (Opus 4) │     └──────────┘     └──────────┘    └────┬─────┘   │
│   └──────────┘                                           │          │
│        │                                                 ▼          │
│        │          ┌──────────┐     ┌──────────┐    ┌──────────┐   │
│        └─────────→│empiricist│────→│ arbiter  │───→│librarian │   │
│                   │(Sonnet 4)│     │ (Opus 4) │    │(V4 Flash)│   │
│                   └──────────┘     └──────────┘    └──────────┘   │
│                                                                      │
│   ┌──────────┐     ┌──────────┐                                     │
│   │  data    │     │portfolio │                                      │
│   │collector │     │ manager  │                                      │
│   │(Sonnet 4)│     │(Sonnet 4)│                                      │
│   └──────────┘     └──────────┘                                      │
│                                                                      │
│   delegate_task() for subagent dispatch                              │
│   skill_view() for agent-specific instructions                       │
│   cronjob() for scheduled data sync + health checks                  │
└──────────┬──────────────┬──────────────┬─────────────────────────────┘
           │              │              │
           ▼              ▼              ▼
   ┌─────────────┐ ┌──────────────┐ ┌──────────────────────────┐
   │  Storage    │ │  Vault       │ │  External                │
   │             │ │              │ │                          │
   │ index.db    │ │ 量化交易/     │ │ Binance Public API       │
   │ (SQLite)    │ │ (Obsidian)   │ │ CCXT (free, no key)     │
   │             │ │              │ │                          │
   │ data/*.parq │ │ 9+1 classes  │ │ OHLCV + funding + OI    │
   │ (Parquet)   │ │ 10K+ entries │ │ Daily data (free tier)  │
   └─────────────┘ └──────────────┘ └──────────────────────────┘
```

### 2.1 資料流向

```
Binance API ──→ data_collector (cron, tick) ──→ data/*.parquet (zstd)
                                                      │
                                                      ▼
                                               DuckDB query engine
                                                      │
                                                      ▼
                                          empiricist 取數驗證
                                                      │
                                                      ▼
                                          debate pipeline 使用
```

### 2.2 Agent 通訊

所有 agent 之間的協作透過以下機制：
- **delegate_task()**：主 agent 派生子 agent 處理子任務
- **檔案讀寫**：產出（筆記、code、回測結果）寫入 vault 或 candidates/，下一個 agent 讀取
- **debate_status.json**：state machine 追蹤目前階段與進度
- **index.db**：結構化資料（因子註冊、evidence 記錄）

**不經過 API、不經過 message queue**。這是刻意簡化：agent 之間靠檔案系統傳遞訊息，與人類讀寫 vault 的模式一致。

---

## 3. 目錄結構

```
/home/anson/
│
├── factor-lab/                              ← 系統主目錄
│   ├── config.yaml                          ← 全局配置（IC 閾值、交易對、費用…）
│   ├── .env                                 ← API keys（OpenRouter）
│   │
│   ├── scripts/
│   │   ├── debate_flow.py                   ← 辯論 state machine
│   │   ├── data_sync.py                     ← 資料同步 script（cron 定時跑）
│   │   └── backtest/
│   │       ├── run_mini.py                  ← mini-backtest runner
│   │       └── run_full.py                  ← full ADRS backtest
│   │
│   ├── core/
│   │   ├── memory.py                        ← SQLite 操作、index.db schema
│   │   ├── adapter.py                       ← CryptoAdapter（Binance → Polars）
│   │   └── signals/
│   │       ├── __init__.py
│   │       ├── F1_Complexity_Gap_Regime.py  ← 已驗證因子（均值回歸版）
│   │       ├── F2_Liquidity_Consumption_Tilt.py
│   │       └── ...                          ← 更多因子
│   │
│   ├── candidates/                          ← 待審核因子暫存區
│   │   └── <factor-name>.md
│   │
│   ├── debate_status.json                   ← state machine 狀態
│   ├── debate_sessions/                     ← 每次辯論的完整記錄（歸檔前暫存）
│   │
│   ├── data/                                ← Parquet files
│   │   ├── btc_1d.parquet                   ← BTCUSDT daily OHLCV
│   │   ├── eth_1d.parquet
│   │   └── ...
│   │
│   └── index.db                             ← 主資料庫（SQLite）
│       ├── factor_registry                  — 因子註冊表
│       ├── paper_insights                   — 論文洞見表
│       ├── evidence                         — 實證記錄表
│       ├── debate_sessions                  — 辯論 session 表
│       ├── agent_calls                      — Agent 呼叫記錄
│       └── ... (共 8 表)
│
├── alpha_agent_plantform/                   ← 設計文檔（本目錄）
│   ├── 00-OVERVIEW.md
│   ├── 01-ARCHITECTURE.md
│   ├── 02-AGENTS.md
│   ├── 03-PIPELINE.md
│   ├── 04-KNOWLEDGE-BASE.md
│   ├── 05-UI-DASHBOARD.md
│   ├── 06-DATA-AND-BACKTEST.md
│   └── 07-IMPLEMENTATION-ROADMAP.md
│
└── Documents/Obsidian Vault/量化交易/       ← Obsidian vault（共用）
    ├── 01-交易策略/
    ├── 02-市场数据/
    ├── 03-回测框架/
    ├── 04-风险管理/
    ├── 05-编程技术/
    ├── 06-书籍论文/
    ├── 07-交易心理/
    ├── 08-工具资源/
    ├── 09-作者专题/
    ├── 10-因子库/
    │   ├── validated/                       ← 已驗證因子
    │   └── deprecated/                      ← 已淘汰因子（= graveyard）
    ├── debate_sessions/                     ← 歸檔後的辯論記錄
    └── _meta/                               ← vault 系統檔（可選）
```

### 3.1 目錄設計原則

1. **一個事實來源**：vault 是唯一的因子描述 / 回測報告 / 文獻來源，不重複儲存。
2. **暫存區隔離**：`candidates/` 是暫存區，審核完成後由 librarian 移入 vault 的 validated/ 或 deprecated/。
3. **code 與 data 分離**：因子 code 在 `core/signals/`，因子描述/文獻在 vault，兩者透過 factor_id 關聯。
4. **無 vendor lock-in**：所有產出都是純文字（markdown / yaml / json / parquet），可以用任何工具讀寫。

---

## 4. Layer-by-Layer 細說

### 4.1 Orchestration Layer（Hermes-agent）

**職責**：
- 人類自然語言指令解析（orchestrator）
- debate pipeline 狀態管理（scripts/debate_flow.py）
- agent 協作編排（delegate_task）
- 定時任務排程（cronjob）

**關鍵設定**（`~/.hermes/profiles/deepseek/config.yaml`）：
- provider: OpenRouter
- 預設 model: deepseek-v4-pro（主對話用）
- delegate_task 子 agent 依角色覆寫 model

### 4.2 Agent Layer

每個 agent 有獨立的 skill（skill file），內含：
- System prompt（角色定義、職責邊界、輸出格式）
- 工具列表（terminal / file / web_search / vault_read / vault_write）
- Model 指定（Opus 4 / Sonnet 4 / V4 Flash）
- KPI 與失敗模式

詳細見 [02-AGENTS.md](./02-AGENTS.md)。

### 4.3 Compute Layer

**資料處理**（Polars 1.x）：
```
CCXT → Polars DataFrame → clean/validate → write Parquet → DuckDB query
```

**回測**（自建 + Polars）：
```
load Parquet → compute factors → align with forward returns → rank IC → quantile equity
```

**統計**（numpy + scipy）：
- Rank-IC 計算
- Spearman/Pearson 相關性
- Quantile 分組回報
- Monte Carlo bootstrap

### 4.4 Storage Layer

| 儲存 | 格式 | 用途 |
|------|------|------|
| index.db | SQLite | 因子註冊、evidence、session 記錄（結構化查詢） |
| data/*.parquet | Parquet (zstd) | OHLCV 歷史資料（時序查詢） |
| DuckDB | ✅ 引擎 | 對 Parquet 做 OLAP 查詢 |
| Vault | Markdown + YAML | 因子描述、文獻、知識庫（人類與 agent 共用） |

**為什麼不用 PostgreSQL + TimescaleDB**：
- 個人單機、資料量不超過 10GB、不需要並發寫入
- DuckDB 的 Parquet 效能對個人規模秒殺 PostgreSQL
- SQLite 的全文搜尋（FTS5）已經夠用

**未來若需要升級**（≥ 100GB data 或 > 1 人使用）：
- SQLite → PostgreSQL（單機即可，不需 TimescaleDB）
- Parquet → 繼續用 Parquet（DuckDB 可直接讀取）

### 4.5 Knowledge Base Layer

詳見 [04-KNOWLEDGE-BASE.md](./04-KNOWLEDGE-BASE.md)。

**核心機制**：agent 直接讀寫 `.md` 檔案，人類在 Obsidian 中看同一份資料。
不需要 indexer、不需要 pgvector、不需要 RAG 框架。

**檢索方式**（目前）：
1. `search_files`（檔案系統全文檢索，rg/grep 等級）
2. `index.db` 中的 SQL 查詢（factor_registry、paper_insights）
3. Obsidian 內建搜尋（人類用）

**未來若需要 semantic search**（> 2,000 篇筆記後）：
- 可加 `sqlite-vec` extension 做本地 embedding vector search
- 或用 DuckDB 的 array 做 cosine similarity

---

## 5. 部署規格

| 項目 | 規格 |
|------|------|
| 主機 | RTX 2080 Ti 22GB VRAM, 468G SSD |
| OS | Linux (Ubuntu 24.04) |
| Python | 3.12（venv） |
| GPU | CUDA 12.x（llama.cpp 本地推理用） |
| LLM Provider | OpenRouter（API key） |
| 排程 | Hermes-agent cronjob |
| 備份 | Obsidian vault git repo（每日 auto-commit） |

---

## 6. 排程一覽

| 任務 | 頻率 | 觸發方式 |
|------|------|----------|
| 資料同步（Binance OHLCV） | 每 5 分鐘 | cronjob（Hermes） |
| 模擬交易 tick | 每 5 分鐘 | cronjob（Hermes） |
| 每日回測 (TST) | 每天 04:05 | cronjob（Hermes） |
| Vault git commit | 每天 02:00 | cronjob（Hermes） |
| Debate pipeline | 手動觸發（orchestrator） | delegate_task |
| Factor-Lab watcher（arXiv + RSS） | 每天 08:00 & 20:00 | cronjob（Hermes） |

---

## 7. 成本估算（簡化版）

| 項目 | 每月 | 備註 |
|------|------|------|
| OpenRouter API | $50-150 | Opus 4 + Sonnet 4 + V4 Flash 混合使用 |
| DeepSeek API | $0-30 | 替代方案，成本更低 |
| 電費 + 網路 | $15-30 | 單機 24/7 |
| **月合計** | **$65-210** | 無 hosting / software license 費用 |

**成本控制**：
- orchestrator / proposer / arbiter 用 Opus 4（重要但頻率低，每月可能 50-100 次呼叫）
- builder / skeptic / empiricist 用 Sonnet 4（頻率高但單價低）
- librarian 用 V4 Flash（最便宜，歸檔類任務容錯度大）
- 沒有 API server 費用、沒有資料庫 hosting 費用

---

## 8. 接下來

繼續讀 [02-AGENTS.md](./02-AGENTS.md) 了解每個 agent 的具體設計。
