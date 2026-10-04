# 08 — MVP BRIDGE：factor-lab ⇄ Cetus Quant

> 把已跑通的輕量骨架 **factor-lab**（`/home/anson/factor-lab/`）對齊成 Cetus 的
> **MVP 實作層**（OVERVIEW §8.2 / ROADMAP M1–M3 的範圍）。
> 原則：**MVP 用檔案式 + 標準庫先跑通流程**，Cetus 的重量級技術棧（Postgres/TimescaleDB、
> FastAPI、Next.js、Hermes-agent、VectorBT Pro/Nautilus）延後到流程驗證後再逐步替換。

---

## 1. 為什麼這樣對齊

Cetus 設計完整但重（12 週、Docker stack、7 agent）。factor-lab 已經能跑「採集→KB→因子評估」。
與其重做，不如讓 **factor-lab 承載 Cetus 的 MVP（Stage 0–3、Conor/Daemon/Helen、3 類 KB）**，
把 Cetus 文檔當「目標終態規格」，MVP 當「第一條能 e2e 跑通的窄路徑」。

**MVP 成功定義**（對齊 OVERVIEW §8.2）：能跑通
`conor 接 idea → daemon 深研 + Soul Questions → helen Gate 審核 → 出報告`，
且每次淘汰自動進 graveyard、新 idea 先 dedup。

---

## 2. 概念對照表

| Cetus 概念 | factor-lab MVP 實作 | 狀態 |
|---|---|---|
| Obsidian vault（單一真相） | 既有 `量化交易/` vault（中文編號資料夾） | ✅ 沿用 |
| KB: alpha-ideas / research / graveyard | `11-alpha提案` / `12-深度研究` / `13-策略墳场` | ✅ 已建 |
| Stage 0–5 state machine | `core/stages.py`（檔案式，VALID_TRANSITIONS、rework 限一次、不能跳級） | ✅ MVP 到 Stage 3 |
| Soul Questions（6 題） | `templates/alpha-idea.md` + `core/gate.py:gate0_machine` | ✅ |
| Gate 0 審核 | 機器檢查(`cetus.py gate0`) + helen agent 語意判斷 | ✅ |
| Gate 2/3 回測門檻 | `core/gate.py:gate_backtest`（Sharpe/PBO/WF/trades） | ✅ 門檻就緒，回測引擎待接 |
| Dedup（pgvector 餘弦） | `core/dedup.py`（詞袋餘弦，中英 bi-gram；門檻同 config） | ✅ MVP 替代，可無痛換 embedding |
| Graveyard（cause_of_death/reusable） | `cetus.py bury` 寫入 + transitions 審計 | ✅ |
| Conor / Daemon / Helen | `.claude/agents/{conor,daemon,helen}.md` | ✅ MVP 三 agent |
| Peter（factor dev） | factor-lab 的 `builder` + `empiricist` + `core/metrics.py` | 🟡 部分（IC 已有，回測待接） |
| Jimmy（KB builder） | factor-lab 的 `archivist` + `librarian` + 採集子系統 | ✅ 已有 |
| Terry（data） | `core/adapters/`（CryptoAdapter，Binance REST/ccxt） | 🟡 行情有，funding/OI/onchain 待加 |
| Calvin（PM）、Stage 4A/4B/5 | — | ⛔ MVP 不做 |
| ADRS（PBO/WF/Sensitivity/MC） | `core/gate.py` 留好門檻欄位 | ⛔ 引擎待接（M 之後） |
| Hermes-agent 編排 | Claude Code sub-agents + slash commands | 🟡 MVP 用 Claude Code 代替 |
| Mission Control UI | `cetus.py log` + MOC | 🟡 CLI 代替，Next.js 延後 |

---

## 3. MVP 模型/編排決策（與 Cetus 終態的差異）

| 項目 | Cetus 終態 | MVP（factor-lab） | 切換時機 |
|---|---|---|---|
| 編排 | Hermes-agent state machine | Claude Code sub-agents + `cetus.py` | 流程穩定後 |
| LLM | DeepSeek V4 Pro/Flash | 當前 Claude Code（agent 定義與模型解耦） | 確認 DeepSeek API 後 |
| 結構化儲存 | Postgres+TimescaleDB | SQLite `index.db`（審計、dedup、IC 歷史） | 資料量 / 時序需求上來 |
| dedup | pgvector embedding | 詞袋餘弦 | 接 embedding 服務後 |
| 回測 | VectorBT Pro / Nautilus | （待接，門檻已就緒） | Stage 2/3 動工時 |

> ⚠️ **動工前先驗證**：`DeepSeek V4 Pro/Flash` 與 `Hermes-agent` 我無法確認存在。
> MVP 刻意讓 agent 定義與編排框架**解耦**——先用 Claude Code 跑通，確認上述兩者可用再遷移，
> 避免被不確定的依賴卡住。

---

## 4. 已驗證可跑的 MVP 指令

```bash
cd /home/anson/factor-lab
python cetus.py new   "ETH 8h Funding Rate Mean Reversion" --target ETH-PERP --timeframe 1h
python cetus.py dedup "<hypothesis 文字>"      # 比對 graveyard + active 提案
python cetus.py gate0 <slug>                   # Gate 0 機器檢查（題數/字數/dedup）
python cetus.py bury  <slug> --cause gate_0_reject --by helen
python cetus.py log   <slug>                   # 階段轉換審計
```
在 Claude Code：`/new-idea <一句話 alpha>`（daemon 產 idea → helen Gate 0）。

已 smoke-test 通過：建檔、dedup、Gate 0 機器檢查、graveyard 歸檔、rework 限一次、跳級攔截。

**已知限制**：MVP dedup 是詞袋餘弦，品質取決於 hypothesis 文字是否充實（對空模板會誤判 clear）；
接 embedding 後解決。

---

## 5. 銜接路線（補完 MVP → 逼近 Cetus）

| 步驟 | 內容 | 對應 Cetus |
|---|---|---|
| **M-a** | helen/daemon 跑一個**真實 idea** 走完 Stage 0→1（已可做） | Gate 0/1 |
| **M-b** | Terry 擴充：funding rate / OI ingestion（Binance 公開 API） | Stage 0 資料可得性 |
| **M-c** | 接最小回測（先 vectorbt 或自寫 walk-forward）餵 `gate_backtest` | Stage 2/3 |
| **M-d** | dedup 換 embedding（pgvector 或本地向量） | Dedup 升級 |
| **M-e** | 評估遷移 Postgres / Hermes / DeepSeek（確認可用後） | 終態技術棧 |

---

## 6. 檔案位置速查

- 設計（終態規格）：`/home/anson/alpha_agent_plantform/00–07`
- MVP 實作：`/home/anson/factor-lab/`
  - `core/stages.py`（狀態機）、`core/gate.py`（Gate）、`core/dedup.py`、`cetus.py`（CLI）
  - `.claude/agents/{conor,daemon,helen,...}.md`、`.claude/commands/new-idea.md`
  - `templates/alpha-idea.md`、`config.yaml` 的 `cetus:` 區
- KB（vault）：`量化交易/{11-alpha提案,12-深度研究,13-策略墳场,10-因子库,...}`
