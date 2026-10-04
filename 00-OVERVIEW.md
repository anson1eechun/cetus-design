# 00 — OVERVIEW

> 在動工之前先理解：**為什麼這樣設計**。
> 最後更新：2026-05-26 — 從 factor-lab 既有基礎出發，輕量化落地。

---

## 1. 系統一句話

> Alpha Agent Platform 是一個由 **9 個 AI 代理人**（Claude Opus 4 / Sonnet 4 / DeepSeek V4 Flash）
> 驅動、**Hermes-agent 編排**、**debate pipeline + Soul Questions 雙層審核**、
> **Obsidian vault 知識沉澱**、**anti-overfitting 防呆**的個人量化 alpha 工廠。

> 從日線趨勢到 tick 級 HFT 皆可，差別在資料層與回測引擎的配置。

---

## 2. 起點：不是從零開始

本系統繼承並擴展既有的 **factor-lab** 辯論流程與知識庫。factor-lab 已具備：

| 既有資產 | 狀態 |
|---------|------|
| 5-agent debate pipeline（proposer → builder → skeptic → empiricist → librarian） | ✅ 已跑通一次完整 e2e |
| SQLite 資料庫（index.db，含 factor_registry、debate_sessions、evidence 等 8 表） | ✅ 已回填 |
| Obsidian vault（`量化交易/`，9 大分類） | ✅ 10,000+ 條目 |
| CryptoAdapter + Polars 回測（daily OHLCV，Binance 公開 API） | ✅ 已驗證 |
| debate_flow.py state machine | ✅ 已測試 |
| SuperTrend 等高 Sharpe 因子（ST(10,3)+ADX SH=1.40） | ✅ 已實證 |

**本系統不是取代 factor-lab，是在此基礎上加：**
- orchestrator（任務調度）+ data_collector（資料同步）+ portfolio_manager（配置風控）
- 完整 Stage 0-5 pipeline（含 paper trade / small capital / live）
- ADRS anti-overfitting 框架（PBO / CV / Walk-Forward / Sensitivity）
- 多模型路由（Opus 4 判斷、Sonnet 4 執行、V4 Flash 整理）

---

## 3. 七大設計原則

### 原則 1 — 把研究流程當軟體工程

每一筆 alpha 都是一個「物件」，有 ID、狀態、版本、審核者、生產日期。
整個流程是 state machine，不能跳級、不能繞過 gate、不能無紀錄修改。
失敗的策略不是消失，是進入 deprecated 留下完整檔案。

### 原則 2 — AI agent 是員工，不是萬能助手

每個 agent 有：明確職位、職責邊界、KPI、模型、所用工具。
**判斷層**（orchestrator / proposer / arbiter）用 **Claude Opus 4**：多步推理、價值判斷。
**執行層**（builder / skeptic / empiricist / data_collector / portfolio_manager）用 **Claude Sonnet 4**：code 撰寫、資料處理。
**整理層**（librarian）用 **DeepSeek V4 Flash**：歸檔、frontmatter 補完。

編排由 **Hermes-agent** 負責：agent state 機、tool routing、checkpoint、failure recovery。

### 原則 3 — 失敗是資產，不是垃圾

預期 80%+ 的 alpha idea 會在 Gate 0 被槍斃。這 80% 不丟掉，
全部進 `10-因子库/deprecated/` 保留完整檔案（hypothesis、failure reason、相關 factor）。
新 idea 進來第一步是 dedup check — 查 deprecated，看你是不是在重複造輪子。

### 原則 4 — Soul Questions + Debate 雙層審核

任何進入 backtest 前的策略必須通過兩層：
1. **Soul Questions 6 題**（proposer 自問自答）：為什麼存在、什麼會殺死它、對手方是誰……
2. **五維批判**（skeptic）：資料探勘、經濟直覺、前視偏誤、重複造輪子、成本流動性

答不出來 = 不准做。這是 Renaissance、Two Sigma 那種研究員 checklist 的工程化。

### 原則 5 — Anti-overfitting 是 pipeline 不是自律

每個策略強制走過：PBO (López de Prado) + CrossValidation + OLS filter + Sensitivity analysis。
**研究員的「我有做交叉驗證」不可信，系統強制做的才可信。**

### 原則 6 — 知識庫優先於程式碼

知識庫以 **Obsidian vault** 為單一真相來源：純 markdown + YAML frontmatter + wikilink，
人類與 agent 共用，git 版本化。Agent 直接讀寫 vault 中的 .md 檔，不經過任何專屬 UI；
人類隨時可開 Obsidian 看完整脈絡、做標註、補資料。

Vault 位於 `/home/anson/Documents/Obsidian Vault/量化交易/`，包含：
- 10,000+ 條知識庫條目（論文、作者專題、思維框架、因子庫）
- 既有因子實驗室辯論記錄（`debate_sessions/`）
- deprecated 因子庫（= graveyard，最珍貴的失敗資料）

### 原則 7 — 輕量務實，個人單機

不做 Docker Compose、不做 PostgreSQL + TimescaleDB + Redis 全套微服務。
技術棧：**Python venv + SQLite/DuckDB + Parquet + Hermes-agent cronjob**。
所有東西跑在 RTX 2080 Ti 單機（22GB VRAM, 468G SSD）。
目標是自動化，不是 infra 炫技。

---

## 4. 從 Idea 到 Live 的完整旅程

```
        [orchestrator 收到指令 或 proposer 自主提案]
                       │
                       ▼
              ┌──────────────────┐
              │  Stage 0: Alpha  │  ← Soul Questions 必答
              │     Idea (.md)   │  ← Dedup check vs deprecated
              └────────┬─────────┘
                       │ arbiter Gate 0 審核
                       ▼   (~82% 在此被槍斃 → deprecated)
              ┌──────────────────┐
              │  Stage 1:        │  ← proposer 深度文獻 + 因子探索
              │  Research        │  ← 產出 hypothesis.md + factor candidates
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │  Stage 2:        │  ← builder 把 factor 寫成可執行 code
              │  Factor Dev      │  ← 跑 mini-backtest (sample slice)
              └────────┬─────────┘
                       │ skeptic 五維批判 → empiricist 實證驗證
                       ▼   arbiter Gate 2 審核 (IC-IR ≥ 0.3)
              ┌──────────────────┐
              │  Stage 3:        │  ← 完整 walk-forward backtest
              │  Full Backtest   │  ← ADRS framework 全套
              └────────┬─────────┘
                       │ arbiter Gate 3 審核
                       ▼
              ┌──────────────────┐
              │  Stage 4A:       │  ← Paper trade (live data, simulated fills)
              │  Paper Trade     │  ← 至少 4 週觀察期
              └────────┬─────────┘
                       │ arbiter Gate 4A 審核
                       ▼   (paper Sharpe / backtest Sharpe 衰減 < 50%)
              ┌──────────────────┐
              │  Stage 4B:       │  ← 真金 ≤ $1,000 USD
              │  Small Capital   │  ← 真實滑點、真實成本驗證
              └────────┬─────────┘
                       │ portfolio_manager 配權重
                       ▼
              ┌──────────────────┐
              │  Stage 5: Live   │  ← 進入 portfolio，按 Sharpe 配 weight
              └──────────────────┘
                       │
                       ▼ (任何時候衰減觸發退場)
                  [deprecated]
```

每一個箭頭背後都有一個 Gate，每一個 Gate 都有量化通過條件，
每一次淘汰都自動歸檔到 deprecated 並更新 dedup index。

---

## 5. Agent 團隊一覽

| Agent | 角色 | 模型 | 一句話描述 |
|-------|------|------|----------|
| **orchestrator** | 任務調度 | Opus 4 | 把人類自然語言指令翻譯成 agent 任務、總調度 |
| **proposer** | 因子提案 | Opus 4 | 提出新 alpha 假設、做深度文獻探索、答 Soul Questions |
| **builder** | 因子實作 | Sonnet 4 | 把因子假設寫成可執行 Python，維護回測引擎 |
| **skeptic** | 五維批判 | Sonnet 4 | 對候選因子進行五維批判（資料探勘/經濟直覺/前視偏誤/重複造輪子/成本流動性） |
| **empiricist** | 實證驗證 | Sonnet 4 | 量化驗證因子預測力（Rank-IC / IC-IR / 分組回報 / 相關性） |
| **arbiter** | 最終裁決 | Opus 4 | 綜合 Soul Questions + 五維批判 + 實證數據，做 pass/reject 裁決 |
| **librarian** | 知識庫管理 | V4 Flash | 把研究產出整理進 Obsidian vault、維護分類學、dedup index |
| **data_collector** | 資料同步 | Sonnet 4 | 資料源同步、ingestion、補洞、health monitoring |
| **portfolio_manager** | 組合管理 | Sonnet 4 | 配權重、控總曝險、風險指標監控 |

**模型分層理由**：
- **Opus 4**（3 agents）：最強推理能力，用於調度決策、假設建構、最終裁決。這些環節出錯成本最高，不能降級。
- **Sonnet 4**（5 agents）：code 撰寫、資料處理、實證計算。能力足夠、成本可控。
- **V4 Flash**（1 agent）：高頻整理類任務。容錯度大、成本最低。

> 註：既有 factor-lab 已實現 proposer / builder / skeptic / empiricist / librarian 五個 agent。
> 新增僅需 4 個：orchestrator / arbiter / data_collector / portfolio_manager。
> 若把 arbiter 合併到 orchestrator，實際新增僅 3 個。

---

## 6. 七階段 Pipeline 一覽

| Stage | 名稱 | 對應 debate 階段 | 進入條件 | 通過條件 | 失敗去處 |
|-------|------|-----------------|----------|---------|----------|
| 0 | Alpha Idea | proposer | Soul Questions 全答 + Dedup CLEAR | arbiter Gate 0 通過 | deprecated |
| 1 | Research | proposer（深度） | Stage 0 通過 | hypothesis.md + ≥1 factor formula | deprecated |
| 2 | Factor Dev | builder → skeptic → empiricist | Stage 1 通過 | IC-IR ≥ 0.3 + 五維批判無致命缺陷 | deprecated |
| 3 | Full Backtest | —（新增） | Stage 2 通過 | Full ADRS pass + Sharpe > 1.5 | deprecated |
| 4A | Paper Trade | —（新增） | Stage 3 通過 | 4 週 paper Sharpe 衰減 < 50% | deprecated |
| 4B | Small Capital | —（新增） | Stage 4A 通過 | 4 週實盤 PnL > 0 | deprecated |
| 5 | Live | —（新增） | Stage 4B 通過 | portfolio_manager 配權重 | Live → 衰減後 deprecated |

**Stage 0-2 對應既有 debate pipeline**。Stage 3-5 為新增，保留未來實作。

---

## 7. 知識庫分類學（Obsidian Vault）

沿用既有 vault（`/home/anson/Documents/Obsidian Vault/量化交易/`），9+1 大類，agent 與人類共用：

```
量化交易/                              ← Obsidian vault root
│
├── 01-交易策略/                        — 策略提案、回測報告
├── 02-市场数据/                        — 資料源規格、API 文件
├── 03-回测框架/                        — 引擎設計、工具文件
├── 04-风险管理/                        — 風控、配置、ratio 計算法
├── 05-编程技术/                        — Python/Polars 技巧、系統架構
├── 06-书籍论文/                        — 學術文獻、從業者文章全文
├── 07-交易心理/                        — 思維框架、Munger checklists
├── 08-工具资源/                        — 外部工具、API、資料源索引
├── 09-作者专题/                        — Renaissance / Two Sigma 等人物與機構
│
├── 10-因子库/                          — 因子資料庫
│   ├── validated/                      — 已驗證因子（active）
│   └── deprecated/                     — 已淘汰因子（= graveyard！）
│
├── debate_sessions/                    — 辯論記錄（每次 e2e 產一份）
│   └── <YYYY-MM-DD>-<name>.md
│
└── _meta/                              — vault 系統檔（可選）
```

**關鍵差異 vs 原 Cetus 設計**：
- 不新建 vault，延用現有 `量化交易/`。
- `deprecated/` 即 graveyard，已在運作。
- `debate_sessions/` 是我們獨有的，記錄每次辯論的完整過程。
- 12 類 → 9+1 類，更精簡。

---

## 8. 技術棧決策（精簡版）

| 層級 | 選型 | 理由 |
|------|------|------|
| 後端語言 | **Python 3.12** | 量化生態（polars / numpy）無可取代 |
| Agent 編排 | **Hermes-agent** | state machine、checkpoint、delegate_task |
| LLM Provider | **OpenRouter**（Opus 4 / Sonnet 4 / V4 Flash） | 多模型路由、統一 API |
| 資料處理 | **Polars 1.x** | 速度 5-10x、lazy eval、記憶體省 |
| 本地分析 | **DuckDB 1.x** | OLAP-first、Parquet native、零配置 |
| 結構化儲存 | **SQLite** | 單檔、零配置、已在使用（index.db） |
| 時序儲存 | **Parquet（zstd）** | 個人單機足夠、Polars 原生支援 |
| 知識庫 | **Obsidian vault**（markdown + frontmatter + wikilink） | 人類與 agent 共用、git 版本化 |
| 回測引擎 | **Polars + 自建** | 既有 CryptoAdapter 已驗證可用 |
| 加密資料 | **CCXT 4.x（Binance 公開 API）** | 免費、OHLCV + funding + OI |
| 排程 | **Hermes-agent cronjob** | 已在使用、無需額外依賴 |
| 前端 | **HTML 互動報表（Plotly）** | 已在使用、暗色主題、K 線 + 權益曲線 |

**跳過的**（個人單機不需要）：
- PostgreSQL / TimescaleDB（DuckDB + Parquet 足夠）
- Redis（單機無需 pub/sub）
- Docker Compose（直接 venv 跑）
- FastAPI / Next.js（HTML 報表 + terminal 已足夠）
- Prometheus / Grafana / Loki（單機看 log 就好）
- Prefect（Hermes cronjob 代替）

---

## 9. 術語表

| 術語 | 定義 |
|------|------|
| **Alpha** | 預期能產生超額報酬的訊號/策略 |
| **Gate** | 階段之間的審核關卡，由 arbiter 把守 |
| **Soul Question** | 任何 alpha 進 Stage 1 前必須回答的 6 題 |
| **五維批判** | skeptic 的五維度審查：資料探勘、經濟直覺、前視偏誤、重複造輪子、成本流動性 |
| **deprecated** | 已淘汰因子的存檔目錄（= graveyard），保留完整檔案供 dedup 使用 |
| **Dedup Check** | 新 idea 與 deprecated 的相似度檢查 |
| **ADRS** | Anti-overfitting / Distribution / Robustness / Sensitivity，回測四重驗證 |
| **PBO** | Probability of Backtest Overfitting (López de Prado, 2016) |
| **IC** | Information Coefficient，因子與報酬的相關係數 |
| **IC-IR** | IC 的資訊比率（均值/標準差），衡量因子預測力穩定度 |
| **Hermes-agent** | 本系統採用的 agent 編排框架（state、tool routing、checkpoint） |
| **Vault** | Obsidian 知識庫根目錄；agent 與人類共用，純 markdown |
| **Debate Pipeline** | proposer → builder → skeptic → empiricist → arbiter → librarian 六階段因子審核流程 |

---

## 10. Quick Start（給未來的自己）

### 10.1 系統現狀

| 項目 | 狀態 |
|------|------|
| Agent 編排（delegate_task） | ✅ 已可用 |
| Model 路由（OpenRouter Opus 4 / Sonnet 4） | ✅ 已配置 |
| Debate pipeline（6-stage） | ✅ 已跑通一次 e2e |
| Vault（量化交易/，9+1 類） | ✅ 10,000+ 條目 |
| 回測引擎（CryptoAdapter + Polars） | ✅ 已驗證 |
| 因子庫（SuperTrend SH=1.40 等） | ✅ 已實證 |
| 排程（cronjob） | ✅ 已設定 |

### 10.2 下一步優先級

1. **模型路由實作**：讓 factor-lab 每個 agent skill 帶上 model 參數（proposer/arbiter → Opus 4, builder/empiricist → Sonnet 4, librarian → V4 Flash），跑一次完整 debate 驗證效果。
2. **Soul Questions 整合**：在 proposer skill 加入 Soul Questions 6 題作為必答項目。
3. **orchestrator agent**：建立 orchestrator skill，接收人類自然語言指令，自動調度 debate pipeline。
4. **ADRS 框架**：補上 PBO + CV + Walk-Forward + Sensitivity（Stage 3）。
5. **data_collector**：資料 ingestion cronjob，自動同步 Binance OHLCV / funding / OI。

---

## 11. 成功與失敗的定義

**成功**
- 系統運轉 6 個月後，deprecated 累積 ≥ 200 個有完整檔案的死策略
- Live portfolio 至少有 1 個策略持續活著超過 3 個月
- 任何時刻你能在 60 秒內回答「我為什麼做這個策略？」「它什麼時候會死？」

**失敗**
- 系統運轉 3 個月後，沒有任何策略進到 Stage 3 以上 → 流程設計有問題
- deprecated 累積很多但同一個假設反覆出現 → Dedup 機制壞了
- Paper trade 表現好但實盤一直虧 → ADRS 沒有真的防到 overfitting，需要回頭檢查

---

## 12. 接下來

繼續讀 [01-ARCHITECTURE](./01-ARCHITECTURE.md) 了解技術棧與目錄結構。
