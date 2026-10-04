# 05 — UI DASHBOARD

> Mission Control 的設計規範。
> Color tokens、typography、組件庫、各面板 wireframe、即時推送機制。

---

## 1. 設計哲學

### 1.1 一句話定位

> Mission Control 是專業量化研究者的「駕駛艙」 — 不是給投資人看的 marketing page。

### 1.2 設計三原則

**1. 資訊密度優先於空間美學**
這不是 SaaS landing page。一個畫面能呈現更多有效資訊（不是更多裝飾），就更有價值。靈感：Bloomberg Terminal、TradingView Pro、Linear 命令列、Vercel dashboard。

**2. 狀態即顏色，狀態即位置**
工程師看一眼就要知道：哪個 agent offline、哪個 stage 卡住、哪個策略要 review。**顏色不是裝飾，是語意**。每個顏色都有專屬語意 token，不可混用。

**3. 暗色為主，光是焦點**
背景是「環境」，數據與互動元素才是「焦點」。暗色主題降低長時間使用的視覺疲勞，且符合金融專業工具的視覺語言。

---

## 2. Design Tokens

### 2.1 Color

```css
/* tokens/color.css */
:root {
  /* ── Background ───────────────────────────── */
  --bg-canvas:        #0a0e14;        /* 整體背景 */
  --bg-surface:       #11161e;        /* 卡片背景 */
  --bg-surface-hi:    #161c25;        /* 卡片內子區塊 */
  --bg-overlay:       rgba(10, 14, 20, 0.85);
  
  /* ── Border ───────────────────────────────── */
  --border-subtle:    #1f2632;
  --border-default:   #2a3340;
  --border-strong:    #3a4658;
  
  /* ── Text ─────────────────────────────────── */
  --text-primary:     #e6edf3;
  --text-secondary:   #9aa6b7;
  --text-tertiary:    #5b6878;
  --text-disabled:    #3a4555;
  
  /* ── Semantic (Agent / Stage status) ──────── */
  --status-idle:      #5b6878;        /* 灰 — idle */
  --status-working:   #4cc38a;        /* 綠 — working */
  --status-review:    #f5a524;        /* 橘 — review */
  --status-offline:   #e5484d;        /* 紅 — offline */
  --status-success:   #4cc38a;
  --status-warning:   #f5a524;
  --status-error:     #e5484d;
  --status-info:      #5eb1ff;
  
  /* ── Soul colors (agent role accent) ──────── */
  /* 每個 agent 有自己的「靈魂色」做卡片邊框、tag 著色 */
  --soul-conor:       #c084fc;        /* 紫 — orchestration */
  --soul-peter:       #5eb1ff;        /* 藍 — engineering */
  --soul-daemon:      #fb7185;        /* 粉紅 — research */
  --soul-terry:       #fbbf24;        /* 黃 — data */
  --soul-jimmy:       #4ade80;        /* 綠 — kb */
  --soul-helen:       #f5a524;        /* 橘 — gate */
  --soul-calvin:      #06b6d4;        /* 青 — portfolio */
  
  /* ── Chart palette ────────────────────────── */
  --chart-positive:   #4cc38a;
  --chart-negative:   #e5484d;
  --chart-neutral:    #9aa6b7;
  --chart-1:          #5eb1ff;
  --chart-2:          #c084fc;
  --chart-3:          #fbbf24;
  --chart-4:          #4ade80;
  --chart-5:          #fb7185;
}
```

### 2.2 Typography

```css
:root {
  /* 字型族 — 避免 Inter / Roboto 那種「AI 通用感」 */
  --font-display: 'Space Grotesk', 'Geist', sans-serif;     /* 標題、agent name */
  --font-body:    'Geist', 'Inter', sans-serif;             /* 內文 */
  --font-mono:    'JetBrains Mono', 'Geist Mono', monospace; /* 數據、代碼 */
  
  /* 規模 — 工具型介面字級緊湊 */
  --text-xs:   11px;     --leading-xs:   16px;
  --text-sm:   12px;     --leading-sm:   18px;
  --text-base: 13px;     --leading-base: 20px;
  --text-md:   15px;     --leading-md:   22px;
  --text-lg:   18px;     --leading-lg:   26px;
  --text-xl:   22px;     --leading-xl:   30px;
  --text-2xl:  28px;     --leading-2xl:  36px;
  --text-3xl:  36px;     --leading-3xl:  44px;
  
  /* 字重 */
  --weight-normal:    400;
  --weight-medium:    500;
  --weight-semibold:  600;
  --weight-bold:      700;
  
  /* 數字 tabular（金額對齊） */
  --num-features: "tnum" 1, "lnum" 1;
}

.num {
  font-family: var(--font-mono);
  font-feature-settings: var(--num-features);
}
```

### 2.3 Spacing & Radius

```css
:root {
  --space-1:  4px;
  --space-2:  8px;
  --space-3:  12px;
  --space-4:  16px;
  --space-5:  20px;
  --space-6:  24px;
  --space-8:  32px;
  --space-10: 40px;
  --space-12: 48px;
  
  --radius-sm:  4px;
  --radius-md:  6px;
  --radius-lg:  10px;
  --radius-xl:  14px;
  --radius-full: 9999px;
}
```

### 2.4 Motion

```css
:root {
  --duration-instant: 80ms;
  --duration-fast:    150ms;
  --duration-default: 220ms;
  --duration-slow:    420ms;
  
  --ease-out:    cubic-bezier(0.22, 1, 0.36, 1);
  --ease-in-out: cubic-bezier(0.42, 0, 0.58, 1);
  --ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);
}

/* 預設過渡 */
* {
  transition: background-color var(--duration-fast) var(--ease-out),
              border-color var(--duration-fast) var(--ease-out),
              color var(--duration-fast) var(--ease-out);
}

/* 狀態變化（agent idle → working）用 spring */
.agent-card-status {
  transition: all var(--duration-default) var(--ease-spring);
}
```

---

## 3. Layout 結構

### 3.1 主視窗（桌面 1440 寬）

```
┌─────────────────────────────────────────────────────────────────────┐
│ TOP BAR (h=56)                                                       │
│ [Logo Cetus] [MISSION CONTROL] [v2.5] [LIVE]   [Health][FAM][AGM]   │
│                                                  [DRM] [time] [user]│
├──────┬──────────────────────────────────────────────────────────────┤
│      │                                                               │
│ SIDE │  CONTENT (depending on route)                                │
│ NAV  │                                                               │
│      │  ── Mission Control 主頁 ───────────────────────────────────  │
│      │   ┌──────────┬───────────────────────────────────────────┐   │
│      │   │SYSTEM    │ AGENT TEAM (7 卡片 grid)                  │   │
│      │   │HEALTH    │                                            │   │
│      │   │          ├───────────────────────────────────────────┤   │
│      │   │KNOWLEDGE │ STRATEGY PIPELINE (7 stage 卡片 row)      │   │
│      │   │BASE      │                                            │   │
│      │   │STATS     ├───────────────────────────────────────────┤   │
│      │   │          │ RESEARCH DAEMON LOG STREAM                │   │
│      │   │          │                                            │   │
│      │   └──────────┴───────────────────────────────────────────┘   │
└──────┴──────────────────────────────────────────────────────────────┘
```

| 區塊 | 寬度 | 行為 |
|------|------|------|
| Side nav | 220 px | 可摺疊到 56 px |
| System health | 240 px | 固定 |
| Knowledge base stats | 240 px | 固定（與 system health 同欄） |
| Agent team | 自適應 grid，最少 7 列 |
| Strategy pipeline | full width，7 卡片 row |
| Log stream | full width，至少 200 px 高 |

### 3.2 Side Nav 結構

```
COMMAND
  ▸ Mission Control
  ▸ AI Office

PIPELINE
  ▸ Pipeline Analytics
  ▸ Gate Review
  ▸ Backtest Lab
  ▸ Arsenal           (current portfolio)
  ▸ Backtest Panel

TRADING
  ▸ Paper Trade
  ▸ Live Trade
  ▸ Factor Sandbox
  ▸ Portfolio Optimizer
  ▸ Alpha Health
  ▸ Trading Terminal

KNOWLEDGE
  ▸ Alpha Lab
  ▸ KB Explorer
  ▸ Factor Explorer
  ▸ Factor Network
  ▸ Alpha Genealogy
  ▸ KB Network

SYSTEM
  ▸ Daemon Dashboard
```

---

## 4. 核心組件

### 4.1 Agent Card

```tsx
// components/agent-card.tsx
type AgentCardProps = {
  name: string;
  role: string;
  status: 'idle' | 'working' | 'review' | 'offline';
  model: 'v4-pro' | 'v4-flash';      // DeepSeek V4 Pro / V4 Flash
  capabilities: string[];     // ['orchestration', 'delegation']
  currentTask?: string;
  metrics: Record<string, string | number>;
  notificationCount?: number;
  soulColor: string;          // var(--soul-conor) 等
};
```

**視覺規範**：
- 邊框：`1px solid var(--border-default)`，hover 時換 `var(--soul-{name})`
- Status dot：左上角 8x8 px 圓點，色彩來自 `--status-*`
- Notification badge：右上角，紅色背景，白色數字
- Model badge：小型 chip，下方並列
- Capability tags：以 `--soul-{name}` 的半透明背景渲染
- 高度固定 200 px，padding 16 px

```tsx
<div className="agent-card" style={{ '--accent': soulColor }}>
  <div className="agent-card__header">
    <span className={`status-dot status-${status}`} />
    <h3 className="agent-card__name">{name}</h3>
    {notificationCount && <Badge>{notificationCount}</Badge>}
  </div>
  
  <p className="agent-card__role">{role}</p>
  
  <div className="agent-card__chips">
    <Chip variant="model">{model}</Chip>
    {capabilities.map(c => <Chip variant="capability" key={c}>{c}</Chip>)}
  </div>
  
  {currentTask && (
    <p className="agent-card__task">{currentTask}</p>
  )}
  
  <dl className="agent-card__metrics">
    {Object.entries(metrics).map(([k, v]) => (
      <div key={k}>
        <dt>{k}</dt><dd className="num">{v}</dd>
      </div>
    ))}
  </dl>
</div>
```

### 4.2 Stage Card

```tsx
type StageCardProps = {
  stageId: 'stage_0' | 'stage_1' | ... | 'stage_5' | 'graveyard';
  title: string;                  // 'Alpha Idea' | 'Research' | ...
  count: number;
  topItem?: { slug: string; date: string };
  pendingCount?: number;
  progressBar?: { done: number; total: number };
};
```

**視覺規範**：
- 寬度自適應（7 stage 平分 row）
- 大數字置中（`text-2xl` mono font）
- 漸進進度條（done / total）
- Top item 顯示最新進入該 stage 的 slug

```
┌──────────────────────────────┐
│ STAGE 0     [v4-pro] [info]  │  ← stage 標籤 + dropdown
│ Alpha Idea                   │
│                              │
│         5/58 (63)            │  ← active/pending (graveyard count)
│                              │
│ 2026-05-25-eth-funding...    │  ← top item slug
│ ────────────  6%             │  ← progress bar
└──────────────────────────────┘
```

### 4.3 KB Stat Card

```tsx
type KBStatProps = {
  label: string;                  // 'TOTAL' | 'ALPHA IDEAS' | ...
  count: number;
  delta?: number;                 // 24h 增量
};
```

**視覺規範**：
- 極簡：只有 label + 大數字 + 可選 delta
- Grid 排成 2 列（System Health 區塊內）
- Hover 時整張卡上提 2px

### 4.4 Log Stream

即時 SSE 串接，render daemon agent 的 log：

```
21:49  [INFO] [openclaw.daemon] [query-diag] 2026-04-06-btc-cvd-eth-lead msg #29
21:49  [INFO] [openclaw.daemon] Received signal 15, shutting down...
21:49  [INFO] [openclaw.daemon] Cancelling 20 in-flight tasks before shutdown...
21:49  [INFO] [openclaw.daemon] Task cleanup: 20 cancelled, 0 errors, 20 total
```

**視覺規範**：
- Mono font，行高緊湊
- Filter chips: `[All] [Info] [Warn] [Error]` + source filter
- Auto-scroll，hover 時暫停
- 顏色：INFO 灰、WARN 橘、ERROR 紅、DEBUG 藍

---

## 5. 主要頁面 Wireframes

### 5.1 Mission Control（主頁）

對應原版截圖 1。已在 §3.1 描述。重點：
- Top bar 健康指示器（Ingestion / FAM / AGM / DRM）即時更新
- Dispatch input：可直接打字給 Conor，按 Enter 送出
- Agent team grid 應呈現 7 個 agent，每張卡 200 px 高
- Strategy pipeline 7 stage 橫排
- Bottom log stream 至少 200 px

### 5.2 Pipeline Analytics

```
┌────────────────────────────────────────────────────────────┐
│ FILTER: [All Stages ▾] [Last 30d ▾] [All Authors ▾]       │
├────────────────────────────────────────────────────────────┤
│                                                             │
│      Funnel chart                                           │
│      ┌─────────────────┐                                    │
│      │ Stage 0 → 1: 18%│                                    │
│      │ Stage 1 → 2: 58%│                                    │
│      │ Stage 2 → 3: 42%│                                    │
│      │ Stage 3 → 4A:51%│                                    │
│      │ Stage 4A→4B:48% │                                    │
│      └─────────────────┘                                    │
│                                                             │
├────────────────────────────────────────────────────────────┤
│ Avg time per stage (chart)                                  │
├────────────────────────────────────────────────────────────┤
│ Author leaderboard (table)                                  │
│ Helen reject reasons distribution (donut chart)             │
└────────────────────────────────────────────────────────────┘
```

### 5.3 KB Explorer

對應原版截圖 4。

```
┌────────────────────────────────────────────────────────────┐
│ 1728 FILES   [Search ...]   [All Status ▾] [All Authors ▾] │
├────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────────────┐ [Read →]    │
│ │ 2026-03-01-cbpi-spike-btc-momentum         │             │
│ │ [gate-0-failed] [anson]                    │             │
│ │ knowledge-base/alpha-ideas/2026-03-01-...  │             │
│ └────────────────────────────────────────────┘             │
│ ... (more rows, virtualized)                                │
└────────────────────────────────────────────────────────────┘
```

雙欄分割版（點 Read → 展開）：

```
┌──────────────────────────┬─────────────────────────────────┐
│ List of KB files         │ Document detail                  │
│                          │                                  │
│ [filter / search]        │ ── Metadata ───────────────────  │
│                          │ slug, author, status, tags...    │
│ ▸ 2026-05-25-...         │                                  │
│ ▸ 2026-05-24-...         │ ── Body ───────────────────────  │
│ ▶ 2026-05-23-...  ←sel   │ # Hypothesis                     │
│ ▸ 2026-05-22-...         │ ...                              │
│ ▸ 2026-05-21-...         │ # Soul Questions                 │
│                          │ Q1 — ...                         │
└──────────────────────────┴─────────────────────────────────┘
```

### 5.4 Backtest Panel

對應原版截圖 7。

```
┌────────────────────────────────────────────────────────────┐
│ Backtest Panel                                              │
│ Historical analytics · Strategy performance · Pipeline      │
├────────────────────────────────────────────────────────────┤
│ ┌────┐ ┌────┐ ┌────┐ ┌─────┐ ┌────┐ ┌────┐ ┌────┐         │
│ │1814│ │ 16 │ │0.82│ │-42.7│ │ N/A│ │N/A │ │88% │         │
│ │TOT │ │BT  │ │SHA │ │MaxDD│ │WIN │ │TRD │ │HLT │         │
│ └────┘ └────┘ └────┘ └─────┘ └────┘ └────┘ └────┘         │
├────────────────────────────────────────────────────────────┤
│ STRATEGY                | STATUS | STAGE | SR | MDD | HLT │
│ 2026-03-20-c-funding... |stage-..| Reject|0.34|23.7%| red │
│ 2026-03-20-c-1d1w-...   |stage-..| Reject|0.34|     | red │
│ test-funding-rate-arb   |stage-..| small |1.87|     |green│
│ ...                                                         │
├────────────────────────────────────────────────────────────┤
│ Equity curve / Drawdown chart (overlay)                     │
│ Filters: [SHARPE ≥ 1.50] [MDD ≤ -10%] [L/S ≥ 1.00]         │
└────────────────────────────────────────────────────────────┘
```

### 5.5 Alpha Genealogy

Force-directed graph，使用 `d3-force` 或 `react-flow`：

- 節點：每個 strategy（含 graveyard）
- 邊：`inherits-from` 實線、`inspired-by` 虛線、`dual-of` 雙箭頭
- 節點大小：被引用次數
- 節點顏色：當前 stage（活著=綠、死=灰、live=金）
- Hover 顯示 metadata，click 跳到 KB Explorer

### 5.6 Alpha Lab（KB 詳情頁）

對應原版截圖 3、5、6。

```
┌────────────────────────────────────────────────────────────┐
│ 📄 2026-03-19-canary-026-realized-vol-3m-x-...            │
│ [Alpha Genealogy] [run_backtest.py][ic_research.py][...]   │
├──────────────────────────────┬─────────────────────────────┤
│ REPORT (.MD)  | FULL REPORT  │ STRATEGY CODE | CODE OUTPUT │
│                              │ BACKTEST CSV  | PERFORMANCE │
│ ▸ SECTIONS (35)              │                             │
│ ▸ FRONTMATTER (111)          │ 1  """                      │
│                              │ 2  OpenClaw Backtest        │
│ # Hypothesis                 │ 3  =================        │
│ ...                          │ 4  Auto-generated by ...    │
│                              │ 5  Anti-overfitting...      │
│ # Factor Formula             │ 6  """                      │
│ factor = realized_vol_3M *   │ 7  import json              │
│   supply_inflation_rate      │ 8  import logging           │
│                              │ 9  ...                      │
│ # Backtest Results           │                             │
│ Sharpe: 1.93                 │                             │
│ Calmar: 2.44                 │                             │
│ Max DD: -20.9%               │                             │
│                              │                             │
└──────────────────────────────┴─────────────────────────────┘
```

雙欄佈局，左 markdown 右 code，可切換 tab。

---

## 6. 即時推送機制

### 6.1 SSE Endpoint

```python
# apps/api/src/cetus_api/routers/stream.py
@router.get("/stream")
async def event_stream(request: Request, channels: list[str] = Query(...)):
    async def gen():
        pubsub = redis.pubsub()
        await pubsub.subscribe(*channels)
        async for msg in pubsub.listen():
            if msg["type"] == "message":
                yield f"event: {msg['channel']}\ndata: {msg['data']}\n\n"
            if await request.is_disconnected():
                break
    return StreamingResponse(gen(), media_type="text/event-stream")
```

### 6.2 前端訂閱

```tsx
// hooks/useEventStream.ts
function useEventStream(channels: string[], onEvent: (e: SSEvent) => void) {
  useEffect(() => {
    const url = `/api/stream?${channels.map(c => `channels=${c}`).join('&')}`;
    const es = new EventSource(url);
    es.onmessage = e => onEvent(JSON.parse(e.data));
    return () => es.close();
  }, [channels.join(',')]);
}

// 用法：Mission Control 主頁
useEventStream(
  ['agent.*.status', 'pipeline.*', 'data.alerts', 'cost.budget'],
  handleEvent,
);
```

### 6.3 Throttle 策略

UI 不應每秒 rerender 10 次。所有 high-frequency event 用 100 ms throttle：
- Agent status：100 ms
- Pipeline counter：500 ms
- Log stream：直接 append，不 throttle（但虛擬列表）

---

## 7. 響應式策略

優先級：

| 視窗 | 行為 |
|------|------|
| ≥ 1440 px（主場景） | 完整佈局 |
| 1024-1440 px | Side nav 自動摺疊；agent grid 維持 7 columns |
| 768-1024 px | Knowledge stats 移到主內容下方；agent grid 變 4 columns |
| < 768 px（手機） | 只顯示 dispatch + pipeline summary + log；agent 詳情走 modal |

**這個系統首要使用情境是桌面。手機只做 monitor + dispatch。**

---

## 8. Accessibility

- 所有色彩對比 ≥ WCAG AA（4.5:1 normal, 3:1 large）
- 所有互動元件 keyboard navigable
- Status 不只用顏色（加上 icon / text）— 色盲友善
- 重要警報除了視覺也要 toast notification

---

## 9. 命令面板（Cmd+K）

```
┌──────────────────────────────────────────────────┐
│ 🔍 Search agents, strategies, KB, commands...    │
├──────────────────────────────────────────────────┤
│ STRATEGIES                                        │
│   2026-05-25 ETH funding reversion  [stage_4a]   │
│   2026-05-24 BTC vol regime                       │
│                                                   │
│ COMMANDS                                          │
│   /pipeline status                                │
│   /kill <strategy_slug>                           │
│   /backtest <strategy_slug>                       │
│   /helen review <strategy_slug>                   │
│                                                   │
│ KB                                                │
│   "funding rate" — 38 results                     │
└──────────────────────────────────────────────────┘
```

實作：`cmdk` (Vercel) + shadcn/ui dialog。

---

## 10. 截圖到實作的對應表

對應原系統視覺，給未來實作者的對照清單：

| 原版區塊 | 本系統實作位置 | 組件 |
|---------|-------------|------|
| Top bar 健康指示 | `<TopBar />` | `<HealthChip />` × 4 |
| Dispatch input | `<DispatchBox />` | `<Input>` + `<Button>` |
| System Health | `<SystemHealth />` | `<HealthRow />` |
| Knowledge Base stats | `<KBStats />` | `<KBStatCard />` × 12 |
| Agent Team grid | `<AgentGrid />` | `<AgentCard />` × 7 |
| Strategy Pipeline row | `<PipelineRow />` | `<StageCard />` × 7 |
| Daemon Log Stream | `<LogStream />` | `<LogLine />` virtualized |
| Pipeline funnel | `<PipelineFunnel />` | Recharts FunnelChart |
| Alpha Genealogy | `<GenealogyGraph />` | react-flow |
| Backtest equity curve | `<EquityCurve />` | Lightweight Charts |

---

## 11. 接下來

繼續讀 [06-DATA-AND-BACKTEST](./06-DATA-AND-BACKTEST.md) 了解資料層與回測引擎。
