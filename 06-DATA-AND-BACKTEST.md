# 06 — DATA & BACKTEST

> 資料源、儲存、回測引擎、ADRS 防呆管線。
> 輕量化：Binance 公開 API → Parquet → Polars → HTML 互動報表。

---

## 1. 資料層全景

```
                        ┌───────────────────────┐
                        │   EXTERNAL SOURCES     │
                        │   Binance Public API   │
                        │   (CCXT, FREE, no key) │
                        └───────────┬───────────┘
                                    │ HTTPS (REST)
                                    ▼
                        ┌───────────────────────┐
                        │   data_collector       │
                        │   (cronjob, 每 5 min)   │
                        │   CCXT → Polars        │
                        └───────────┬───────────┘
                                    │ Polars DataFrame
                                    ▼
        ┌──────────────────────────────────────────────┐
        │            STORAGE LAYER                      │
        │                                               │
        │  data/<symbol>_<tf>.parquet (zstd)           │
        │   ├── btc_1d.parquet       daily OHLCV       │
        │   ├── eth_1d.parquet                          │
        │   ├── btc_1h.parquet       hourly OHLCV       │
        │   └── ...                                     │
        │                                               │
        │  DuckDB (OLAP engine)                         │
        │   └── query Parquet files directly            │
        │                                               │
        │  index.db (SQLite)                            │
        │   └── factor_registry, evidence, etc.         │
        └───────────┬──────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │   Backtest Engine      │
        │   (Polars + numpy)     │
        │                       │
        │   Mini (sample slice)  │ → Stage 2
        │   Full ADRS           │ → Stage 3
        └───────────────────────┘
```

---

## 2. 資料源規格

### 2.1 Binance Futures（公開 API）

| 端點 | 資料 | 粒度 | 歷史深度 | 費用 | 筆記 |
|------|------|------|---------|------|------|
| `GET /fapi/v1/klines` | OHLCV | 1m/5m/15m/1h/4h/1d | 約 2-3 年 | FREE | 主力資料 |
| `GET /fapi/v1/fundingRate` | Funding Rate | 8h | ~200 筆 (~66d) | FREE | 歷史太短，ML 訓練不夠 |
| `GET /fapi/v1/openInterest` | Open Interest | current only | 僅可查當前值 | FREE | 歷史 404 |
| `GET /fapi/v1/takerBuySellVolume` | Taker Buy/Sell | 已聚合 | 30d | FREE | 太短 |

**資料限制（已知）**：
- Funding Rate 歷史僅 200 筆（約 66 天），不足以訓練 ML 模型
- Open Interest 無歷史端點
- CoinGlass 需要付費 API key
- 目前不適合做基於 OI/funding 的 ML 策略（除非用 Tick data）

### 2.2 已停用或未使用的來源

| 來源 | 原因 |
|------|------|
| Bybit / Hyperliquid | 目前不需要，Binance 流動性最充足 |
| Glassnode | 需要付費 key（Advanced plan $39/mo） |
| CoinGlass | 需要付費 API key |
| DefiLlama | 未整合 |

---

## 3. 資料處理管線

### 3.1 Ingestion Pipeline

```python
# data_collector 的核心流程（pseudocode）
import ccxt
import polars as pl
from pathlib import Path

exchange = ccxt.binance({"enableRateLimit": True})

def fetch_and_store(symbol: str, timeframe: str, out_path: Path):
    # 1. 讀取現有 parquet（若存在）
    existing = None
    if out_path.exists():
        existing = pl.read_parquet(out_path)

    # 2. 從最後一個 timestamp 開始拉新資料
    since = None
    if existing is not None and len(existing) > 0:
        since = existing["timestamp"].max()

    # 3. 拉取 OHLCV
    raw = exchange.fetch_ohlcv(symbol, timeframe, since=since, limit=1000)
    new_df = pl.DataFrame(raw, schema=["timestamp", "open", "high", "low", "close", "volume"])

    # 4. 合併、去重、排序
    if existing is not None:
        combined = pl.concat([existing, new_df]).unique(subset=["timestamp"]).sort("timestamp")
    else:
        combined = new_df.sort("timestamp")

    # 5. 寫入 parquet (zstd compression)
    combined.write_parquet(out_path, compression="zstd")
```

### 3.2 資料驗證

每次同步後自動檢查：

```python
def validate_ohlcv(df: pl.DataFrame, symbol: str, timeframe: str):
    """回報異常，但不停機"""
    issues = []

    # 1. 時間排序
    if not df["timestamp"].is_sorted():
        issues.append("timestamp not sorted")

    # 2. 時間間隔檢查
    expected_interval = timeframe_to_ms(timeframe)
    gaps = df["timestamp"].diff() > expected_interval * 1.5
    if gaps.any():
        missing = df.filter(gaps)
        issues.append(f"data gaps: {len(missing)} missing candles")

    # 3. OHLCV 邏輯
    if (df["high"] < df["low"]).any():
        issues.append("high < low")
    if (df["high"] < df["open"]).any():
        issues.append("high < open")

    # 4. 零成交量
    if (df["volume"] == 0).any():
        issues.append("zero volume candles")

    return issues
```

### 3.3 補洞（Backfill）

```bash
# 手動補洞 — 當發現資料缺失時
python /home/anson/factor-lab/scripts/data_sync.py \
    --backfill BTCUSDT \
    --timeframe 1d \
    --days 365
```

---

## 4. 儲存層

### 4.1 Parquet 檔案規格

```
data/
├── btc_1d.parquet         # schema: timestamp, open, high, low, close, volume
├── eth_1d.parquet
├── btc_1h.parquet
├── eth_1h.parquet
└── ...

# 單檔約 10-50MB（2 年 daily data）
# 壓縮比約 5:1（zstd level 3）
# DuckDB 可直接 query; Polars 原生讀取
```

### 4.2 DuckDB 用法

```python
import duckdb

# 直接查詢 parquet 檔案，無需 import/load
results = duckdb.sql("""
    SELECT
        timestamp,
        close,
        close / LAG(close) OVER (ORDER BY timestamp) - 1 AS ret
    FROM 'data/btc_1d.parquet'
    WHERE timestamp >= '2024-01-01'
    ORDER BY timestamp
""").pl()  # 直接轉 Polars DataFrame
```

### 4.3 為什麼不用 PostgreSQL + TimescaleDB

| 需求 | Parquet + DuckDB | PostgreSQL + TimescaleDB |
|------|-----------------|-------------------------|
| 單機查詢速度 | ✅ 極快（columnar） | ⚠️ 可接受但慢 |
| 部署複雜度 | ✅ 零（檔案系統） | ❌ Docker + 配置 + migration |
| 個人資料量（< 10GB） | ✅ 完美 | ⚠️ 殺雞用牛刀 |
| 查詢語法 | SQL（DuckDB） | SQL |
| 並發寫入 | ⚠️ 不需要（單機單寫） | ✅ 佳 |
| 備份 | 複製檔案 | pg_dump |

---

## 5. 回測引擎

### 5.1 架構

```
CryptoAdapter (CCXT → Polars)
    │
    ▼
Factor 函數 (core/signals/F<id>_<name>.py)
    │
    ▼
Signal 陣列
    │
    ▼
Forward Returns 對齊 (嚴格避免前視偏差)
    │
    ▼
Rank-IC / IC-IR / Quantile Returns
    │
    ▼
HTML 互動報表 (Plotly, 暗色主題)
```

### 5.2 既有實作

因子回測引擎已在 `/home/anson/factor-lab/core/` 下：

```python
# CryptoAdapter — 既有、已驗證
from core.adapter import CryptoAdapter
adapter = CryptoAdapter()

# 取 OHLCV
df = adapter.get_ohlcv("BTCUSDT", "1d", lookback_days=730)

# 計算因子
from core.signals.F1_Complexity_Gap_Regime import compute_factor
signal = compute_factor(df)

# 對齊 forward returns (t+1, 避免前視偏差)
fwd_returns = df["close"].pct_change().shift(-1)

# Rank-IC
from scipy.stats import spearmanr
ic, _ = spearmanr(signal.drop_nulls(), fwd_returns.drop_nulls())
```

### 5.3 回測規格

| 項目 | Mini (Stage 2) | Full ADRS (Stage 3) |
|------|---------------|---------------------|
| 資料期間 | ≥ 1 年 | ≥ 2 年 |
| 交易對 | 1-2 symbols | 3+ symbols |
| 費用假設 | 0.09% per side | 0.09% per side |
| 指標 | Rank-IC, IC-IR, quantile returns | + PBO, CV, WF, Sensitivity |
| 輸出 | 純文字報告 | HTML 互動報表 (K 線 + equity curve) |
| 執行時間 | < 5 秒 | < 60 秒 |

### 5.4 費用模型

```
Binance Futures: maker 0.02% + taker 0.04% = ~0.06% round trip
保守假設: 0.09% per side (0.18% round trip)
滑點假設: 0.01-0.05% (取決於流動性與下單量)
```

---

## 6. ADRS 防呆框架

### 6.1 為何需要 ADRS

Mini-backtest（Stage 2）給的是「最佳情況」。
Full ADRS（Stage 3）要回答的是「這東西在其他情況下還能活嗎？」

### 6.2 A — Anti-overfitting

**PBO (Probability of Backtest Overfitting)**：
```python
def compute_pbo(returns, n_trials=1000):
    """
    López de Prado (2016) PBO 實作
    若 PBO > 0.5: 回測結果很可能是過擬合
    """
    from itertools import combinations
    import numpy as np

    T = len(returns)
    M = T // 2  # in-sample size = half
    trials = []

    for _ in range(n_trials):
        # 隨機選 M/T 的組合做 IS
        idx = np.random.choice(T, M, replace=False)
        is_ret = returns[idx]

        # IS 最佳參數在 OOS 的表現
        is_sharpe = sharpe(is_ret)
        oos_sharpe = sharpe(returns[np.setdiff1d(range(T), idx)])
        trials.append(oos_sharpe < is_sharpe)  # OOS 比 IS 差

    return np.mean(trials)
```

**Purged Cross-Validation**：

使用 `sklearn.model_selection.TimeSeriesSplit` 避免時間洩漏。

### 6.3 D — Distribution

**OLS Filter（Residual Alpha）**：

將新因子的回報對既有因子做 OLS 回歸，取殘差：
```python
from scipy.stats import linregress

# 既有因子的訊號矩陣
known_factors = load_known_factors()

# OLS: new_factor_returns ~ known_factors
result = linregress(known_factors, new_factor_returns)
residual_alpha = result.intercept  # 獨立的 alpha
t_stat = ...  # 顯著性
```

### 6.4 R — Robustness

**Walk-Forward Analysis**：

```
Training Window    │ Test Window
[───── 1y ────]    │ [── 3m ──]
        [───── 1y ────]│[── 3m ──]
                [───── 1y ────]│[── 3m ──]
```

關鍵指標：
- WF Sharpe 衰減 = (IS Sharpe - OOS Sharpe) / IS Sharpe
- 目標：衰減 < 50%

### 6.5 S — Sensitivity

**參數敏感度分析**：
```python
# 對每個參數做 ±20% 網格搜索
import optuna

def objective(trial):
    param1 = trial.suggest_float("param1", lb * 0.8, lb * 1.2)
    param2 = trial.suggest_float("param2", lb * 0.8, lb * 1.2)
    return compute_sharpe(factor_fn(df, param1, param2))

study = optuna.create_study(direction="maximize")
study.optimize(objective, n_trials=100)
```

目標：參數 ±20% 變動，Sharpe 波動 < 30%（否則是過擬合）。

---

## 7. HTML 互動報表

### 7.1 規格

使用者偏好：**暗色主題 HTML 互動報表**（Plotly），直接開 Chrome 檢視。

含以下圖表：
1. **K 線圖 + 進出場標記**（Candlestick with markers）
2. **權益曲線**（Equity curve, log scale）
3. **回撤曲線**（Drawdown curve, filled area）
4. **滾動 Sharpe**（Rolling Sharpe, 90-day window）

### 7.2 模板

```python
import plotly.graph_objects as go
from plotly.subplots import make_subplots

fig = make_subplots(
    rows=4, cols=1,
    shared_xaxes=True,
    vertical_spacing=0.05,
    row_heights=[0.4, 0.25, 0.2, 0.15],
    subplot_titles=["Price & Signals", "Equity Curve", "Drawdown", "Rolling Sharpe"]
)

# 暗色主題
fig.update_layout(
    template="plotly_dark",
    paper_bgcolor="#0a0e14",
    plot_bgcolor="#11161e",
    font=dict(color="#e6edf3")
)

fig.write_html("/tmp/backtest_result.html")
```

### 7.3 開啟方式

```bash
google-chrome --new-window /tmp/backtest_result.html
```

---

## 8. 定時任務

| 任務 | 頻率 | Script | 說明 |
|------|------|--------|------|
| OHLCV 同步 | 每 5 min | `data_sync.py` | 拉最新 K 線，寫入 parquet |
| 每日回測 TST | 每天 04:05 | cronjob | 重新計算因子訊號 + 權益曲線 |
| 資料健康檢查 | 每天 03:00 | cronjob | 檢查缺失、異常、回報 |

---

## 9. 限制與已知問題

| 問題 | 影響 | 緩解 |
|------|------|------|
| Funding rate 歷史僅 66 天 | 無法做長週期基金融資率策略 | 等待累積資料 / 考慮付費資料源 |
| Open Interest 無歷史端點 | 無法做 OI-based 回測 | 等待 Binance API 更新 |
| 單一交易所 (Binance) | 資料中斷時無備援 | 未來可加 Bybit 作為備援 |
| 硬碟僅 468G | HFT tick data 不可行 | MVP 階段不用 tick data |
| CoinGlass 需付費 | 未整合 | 暫不納入 |

---

## 10. 接下來

繼續讀 [07-IMPLEMENTATION-ROADMAP.md](./07-IMPLEMENTATION-ROADMAP.md) 了解實作路線圖。
