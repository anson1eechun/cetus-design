# Cetus Quant — 系統設計文檔

> 一座屬於個人量化研究者的「Alpha 工廠」。
> 由多代理人 AI 團隊驅動、強制 stage gate 審核、知識庫沉澱、Anti-overfitting 防呆，從 idea 一路走到 paper trade 與小資金實盤。
> Cetus 取自鯨魚星座，致敬此架構的精神原型 — Balaena Quant。

---

## 為何存在這份文檔

量化研究的失敗模式幾乎都是相同的：
- 隨手做一個 backtest 看到 Sharpe 1.5 就上線 → 過擬合
- 不知道自己過去研究過什麼 → 重複造輪子
- 沒有「為什麼這個 alpha 會存在」的解釋 → 訊號失效時無法歸因
- 一個人開發、一個人審核、一個人上線 → 沒有任何制衡

Cetus Quant 把這些失敗模式工程化地消滅：**用多代理人分工取代單一研究員、用 stage gate 強制淘汰機制取代「自己覺得 OK」、用 Graveyard 與 dedup 取代失憶、用 Soul Questions 強制思辨。**

---

## 文檔結構

| # | 檔案 | 內容 | 讀者優先級 |
|---|------|------|------|
| 00 | [OVERVIEW](./00-OVERVIEW.md) | 設計哲學、術語、整體流程 | ⭐⭐⭐ 必讀 |
| 01 | [ARCHITECTURE](./01-ARCHITECTURE.md) | 技術棧、服務拓撲、目錄結構、部署 | ⭐⭐⭐ 必讀 |
| 02 | [AGENTS](./02-AGENTS.md) | 7 個 AI agent 完整規範 | ⭐⭐⭐ 必讀 |
| 03 | [PIPELINE](./03-PIPELINE.md) | Stage gate、Soul Questions、Graveyard | ⭐⭐⭐ 必讀 |
| 04 | [OBSIDIAN KB](./04-KNOWLEDGE-BASE.md) | Obsidian vault 結構、frontmatter、檢索、genealogy | ⭐⭐ 核心 |
| 05 | [UI DASHBOARD](./05-UI-DASHBOARD.md) | Mission Control 規範、組件庫 | ⭐⭐ 核心 |
| 06 | [DATA & BACKTEST](./06-DATA-AND-BACKTEST.md) | 資料層、回測引擎、ADRS、PBO | ⭐⭐⭐ 必讀 |
| 07 | [ROADMAP](./07-IMPLEMENTATION-ROADMAP.md) | 12 週實作路線圖、成本估算、風險 | ⭐⭐⭐ 必讀 |
| 08 | [MVP ↔ factor-lab](./08-MVP-FACTOR-LAB-BRIDGE.md) | 設計與既有 factor-lab 實作的對接方式 | ⭐⭐ 核心 |

---

## 推薦閱讀順序

**第一輪（理解系統 / 約 2 小時）**
00 → 02 → 03

**第二輪（決定動工 / 約 3 小時）**
01 → 06 → 07

**第三輪（開始實作時）**
04 → 05（搭配實作進度逐章對照）

---

## 設計信念（One-liner）

> **量化研究是工程問題，不是直覺問題。每一步都應該可重現、可審計、可淘汰、可繼承。**

---

## 系統範圍宣告

**Cetus Quant 是什麼**
- 個人量化研究者的全流程作業系統
- 標的：加密貨幣永續合約（BTC / ETH / 主流 alt-coin）
- **不限策略頻率**：從日線級趨勢到 sub-second HFT 皆支援，差別在資料層與回測引擎的選擇
- 規模：個人單機（一台主機 Docker Compose 跑全部）
- 模型：**DeepSeek V4 Pro**（判斷/執行類 agent）+ **DeepSeek V4 Flash**（整理類 agent）
- 編排：**Hermes-agent** 運行調度（agent state、tool routing、checkpoint）
- 知識庫：**Obsidian vault**（人類與 agent 共用、git 版本化、純 markdown）

**Cetus Quant 不是什麼**
- 不是 SaaS 多租戶平台（單用戶設計）
- 不是黑盒策略商城（強調思辨與審計過程）
- 不是「給我 alpha」的魔法盒子（過濾率 80%+ 是常態）

---

## 實作現況

這份是設計文件，不含程式碼。實際落地的是輕量 MVP：

- 研究引擎：[factor-lab](https://github.com/anson1eechun/factor-lab)（辯論流程、stage gate、search ledger、forward paper trade）
- 控制台：[balaena-quant](https://github.com/anson1eechun/balaena-quant)（Mission Control + KB Explorer）

與設計的主要差異：Stage 4b / 5（小資金與實盤）沒有實作，系統只做研究不下單；Docker Compose、pgvector 等基礎設施以 SQLite 與詞袋去重代替。

---

## 維護者

**李鈞盛（負責人）**
逢甲大學資工系 / Taichung, Taiwan

文檔版本：v1.0 — 2026-05-25
