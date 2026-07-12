# v2/demo/backtest.py 深度解析

> 文件：`v2/demo/backtest.py` · 274 行 · 1 个 CLI
> 地位：**v2 的"展示工程"代表作**——驱动真实引擎，但加上 25 只股票宇宙、pinned dates、replay 节奏、rich 终端 UI

## 文件作用一句话

**包一层 `BacktestEngine`，配 25 只美股 + 固定日期 + 磁盘缓存 + rich 终端 replay，做一个 ~20 秒的演示 dashboard。**

## 8 个核心常量（line 45-63）

```python
TICKERS = [25 只美股]            # 25 只覆盖多行业的 universe
START_DATE = "2023-07-01"        # 固定
END_DATE = "2026-06-13"          # 固定，pinned（不是 date.today()）
HOLDING_DAYS = 5                  # 5 天持仓
CAPITAL = 100_000.0               # 10 万初始
PER_TRADE = 10_000.0              # 1 万单笔
EARNINGS_LIMIT = 40               # 40 期财报历史
REPLAY_SECONDS = 18.0             # 18 秒播完
CURVE_HEIGHT = 6                  # equity 曲线高度 6 行
BLOCKS = " ▁▂▃▄▅▆▇█"             # 8 个 unicode 字符画图
```

### 关键：`START_DATE` 和 `END_DATE` 都是**写死**的

**注释 line 52-55 明示**：

> "Pinned (NOT date.today()): a moving end date would change the cache keys and the quoted stats between rehearsal and demo day. Chosen so the engine's padded price window (+ holding_days*2 + 10 days) also stays in the past."

**意义**：

- 每次跑出**完全一样的数字**——rehearsal 和 demo day 一致
- Cache key 也稳定——不会被 `today()` 污染
- 选 `2026-06-13` 让 padded window（+20 天）也在过去

**这是展示工程的精髓**——**reproducibility > freshness**。

## 3 个数据加载阶段（main 函数 line 235-269）

### Phase A：并行预热缓存（line 99-105）

```python
with ThreadPoolExecutor(max_workers=8) as pool:
    futures = [pool.submit(_prefetch_ticker, t, refresh) for t in TICKERS]
    for _ in as_completed(futures):
        progress.advance(task)
        live.refresh()
    for f in futures:
        f.result()    # 显式 surface 任何 fetch error
```

**为什么 8 个线程**：

- FD API 限流但允许并发
- 8 是个保守值（不激进触发 429）
- 25 只 ticker / 8 并发 ≈ 3 批 ≈ 5-10 秒

**为什么 `f.result()` 在循环末尾**：

- `as_completed` 只触发"完成"事件，但**不抛错**
- 必须 `f.result()` 才能 surface exception
- **fail-loud 保留**：fetch 错误会让 demo 中止，而不是吞掉

**`_prefetch_ticker` 函数**（line 77-83）：

```python
def _prefetch_ticker(ticker, refresh):
    with FDClient() as raw:
        fd = CachedDataClient(raw, refresh=refresh)
        fd.get_prices(ticker, START_DATE, _price_window_end())
        fd.get_earnings_history(ticker, limit=EARNINGS_LIMIT)
    return ticker
```

**精确预热**：

- `get_prices` 用 `START_DATE` 到 `_price_window_end()`——**和 `BacktestEngine._trade_ticker` 用一样的窗口**
- `get_earnings_history` 用 `EARNINGS_LIMIT = 40` —— PEAD 需要 8 期但 demo 多拿以防万一

**效果**：第一次跑完整 warm cache；之后完全离线。

### Phase B：跑引擎（line 240-246）

```python
engine = BacktestEngine(capital=CAPITAL, per_trade=PER_TRADE)
model = PEADModel(earnings_limit=EARNINGS_LIMIT)
with FDClient() as raw:
    fd = CachedDataClient(raw)
    result = engine.run_alpha(
        model, TICKERS, fd, START_DATE, END_DATE, holding_days=HOLDING_DAYS,
    )
```

**注意：第二次 `with FDClient()`**——和 Phase A 同一个 client 模式，但**独立 session**（`with` 块结束会关闭）。

### Phase C：replay 节奏（line 253-258）

```python
delay = max(0.02, min(0.35, REPLAY_SECONDS / len(trades)))
shown: list = []
for trade in trades:
    shown.append(trade)
    live.update(_frame(engine, shown, len(trades), curve_width))
    time.sleep(delay)
```

**关键**：**按 trade 数量动态调整 delay**——保证**总 replay 时长 ≈ 18 秒**。

| 交易数 | delay |
|--------|-------|
| 50 | 0.36s |
| 100 | 0.18s |
| 200 | 0.09s |
| 500 | 0.036s |
| 1000 | 0.018s（被 `max(0.02, ...)` 截断） |

**为什么**：

- 每次跑都是 ~18s → 演讲/演示节奏一致
- 太快看不清，太慢不耐烦

## 4 个 rich 面板

### `_banner`（line 112-118）

```python
def _banner():
    line = Text()
    line.append("  PEAD BACKTEST", style="bold white")
    line.append("  ·  long the beats, short the misses", style="cyan")
    line.append(f"  ·  {len(TICKERS)} stocks", style="dim")
    line.append("  ·  point-in-time earnings, no lookahead", style="dim")
    return line
```

**简单 banner**——标题 + 标语 + 元数据。

### `_stats_panel`（line 121-143）

```python
def _stats_panel(metrics, equity, n_total, shown):
    grid = Table.grid(expand=True)
    for _ in range(6):
        grid.add_column(justify="center")
    ...
    grid.add_row(
        cell("PORTFOLIO", f"${equity:,.0f}", "white"),
        cell("RETURN", f"{m.total_return_pct:+.2%}", ret_style),
        cell("SHARPE", f"{m.sharpe_ratio:.2f}", sharpe_style),
        cell("MAX DD", f"{m.max_drawdown_pct:.2%}", "red"),
        cell("WIN RATE", f"{m.win_rate:.0%}", "cyan"),
        cell("TRADES", f"{shown}/{n_total}  ({m.n_long}L·{m.n_short}S)", "white"),
    )
```

**6 个并排指标**：

| 指标 | 颜色规则 |
|------|---------|
| Portfolio | 白色 |
| Return | green (>=0) / red (<0) |
| Sharpe | green (>1) / yellow (>0) / red (<=0) |
| Max DD | 红色（永远） |
| Win Rate | 青色 |
| Trades | 白色 + 实时更新（replay 期间） |

### `_curve_panel`（line 146-176）—— Unicode sparkline

```python
def _curve_panel(curve, width):
    if len(curve) <= width:
        cols = curve
    else:
        step = (len(curve) - 1) / (width - 1)
        cols = [curve[round(i * step)] for i in range(width)]
    
    lo = min(min(cols), CAPITAL)
    hi = max(max(cols), CAPITAL)
    span = (hi - lo) or 1.0
    
    rows = []
    levels = [((v - lo) / span) * CURVE_HEIGHT * 8 for v in cols]
    for row in range(CURVE_HEIGHT - 1, -1, -1):
        line = Text()
        for v, lvl in zip(cols, levels):
            eighths = max(0, min(8, round(lvl - row * 8)))
            style = "green" if v >= CAPITAL else "red"
            line.append(BLOCKS[eighths], style=style)
        rows.append(line)
```

**8 级 unicode 字符**（` ▁▂▃▄▅▆▇█`）画 area chart。

**算法**：

1. resample 到 `width` 列
2. 归一化到 [0, 1]（以 `lo..hi` 为范围）
3. 放大到 `CURVE_HEIGHT × 8` 级（每行 8 个 unicode 字符）
4. 每个 cell 选 unicode 字符 + 颜色（>= CAPITAL green, < red）

**好处**：

- ✅ 纯字符画图（不需要 matplotlib / image）
- ✅ 终端原生支持
- ✅ 重绘快（每帧重算一次）

### `_tape_panel`（line 179-202）

```python
def _tape_panel(trades, limit=20):
    table = Table.grid(expand=True, padding=(0, 1))
    ...
    for t in list(reversed(trades))[:limit]:
        surprise = t.metadata.get("eps_surprise")
        badge = Text("BEAT", style="bold green") if surprise == "BEAT" else ...
        direction = Text("LONG ", style="green") if t.direction == "long" else ...
        ...
        table.add_row(
            Text(t.entry_date, style="dim"),
            Text(t.ticker, style="bold cyan"),
            direction,
            badge,
            ...
        )
```

**展示最近 20 笔交易**：

| 列 | 例子 |
|----|------|
| Date | 2024-03-15 |
| Ticker | AAPL |
| Direction | LONG/SHORT |
| Beat/Miss | BEAT/MISS |
| Entry price | $180.23 |
| → | (separator) |
| Exit price | $182.45 |
| PnL | +1,234 +1.2% |

**replay 期间 trades 逐渐增加**——观众能看到 trade tape 实时累积。

## `_frame` 函数（line 205-216）

```python
def _frame(engine, trades_shown, n_total, width, footer=None):
    curve = engine._build_equity_curve(trades_shown)
    metrics = engine._compute_metrics(trades_shown, curve)
    parts = [
        _banner(),
        _stats_panel(metrics, curve[-1], n_total, len(trades_shown)),
        _curve_panel(curve, width),
        _tape_panel(trades_shown),
    ]
    if footer is not None:
        parts.append(footer)
    return Group(*parts)
```

**每帧 4 个面板**：banner + stats + curve + tape。

**注意**：`engine._build_equity_curve` 和 `engine._compute_metrics` **都是私有方法**（下划线开头）——demo 直接调私有方法。

**判断**：合理。**demo 知道内部结构**，用私有方法省去"开 public API"的开销。

## 设计取舍

### 1. rich vs matplotlib / plotext

✅ **rich**：

- 终端原生，重绘快
- 不需要 GUI / image export
- 跨平台

❌ **matplotlib**：需要 savefig → 显示图片，需要 X server / image viewer

### 2. 8 个 unicode 字符 vs 256 色块

✅ **8 字符**：足够看趋势，渲染快

❌ **256 色块**：每个 cell 一个 char，渲染慢

### 3. fixed dates vs sliding window

✅ **fixed**：

- 可复现
- demo 数字稳定

❌ **sliding window**：总是用最新数据，但每次跑数字不同

**判断**：demo 用 fixed。prod/回测框架用 sliding。

### 4. `f.result()` 显式 raise

```python
for f in futures:
    f.result()    # 显式 surface 任何 fetch error
```

✅ **显式 raise**：fail-loud

❌ **不调用 result**：exception 会被吞

**判断**：和 v2 fail-loud 哲学一致。

### 5. `ThreadPoolExecutor(max_workers=8)`

✅ **8**：保守并发

❌ **16/32**：激进，可能触发 429

**判断**：合理。

## 对 fork → CN 版本的启示

### A. CN demo 几乎可以**完全照抄**

**唯一改的**：

| 改 | 改成 |
|----|------|
| `TICKERS` | A 股 25 只（如 `600519.SH`, `000001.SZ` 等） |
| `START_DATE` / `END_DATE` | A 股某个 historical 区间 |
| `_MARKET_TICKER` | 沪深 300 |

**完全保留**：rich UI、replay 节奏、8 个 unicode 字符、panel 布局。

### B. 几个 CN 特有的 demo 增强

| 增强 | 原因 |
|------|------|
| 在 `_banner` 加"涨跌停内不成交"标注 | 强调 A 股约束 |
| `_tape_panel` 加"是否触发涨停"列 | 演示约束 |
| `_curve_panel` 用红色阈值 | 跟 A 股"跌"语义对齐 |

### C. CN data 加载可能更慢

Tushare 单 ticker 财报历史可能限速更严——可能要把 `max_workers` 调到 4 或 2。

## 深读问题（自检）

- [ ] demo 直接调 `engine._build_equity_curve`（私有方法）——如果将来 `BacktestEngine` 重构，这段代码会坏。怎么改更鲁棒？
- [ ] `REPLAY_SECONDS = 18.0` 是硬编码——如果 trade 数量变化（数据集变了），replay 节奏会变。如何让节奏自适应？
- [ ] `for f in futures: f.result()` 放在 `as_completed` 循环**外**——这样如果某个 future 抛错，**等到所有 future 完成才看到**。是不是应该**一抛错就 bail**？
- [ ] 8 个 unicode 字符的 sparkline 精度有限（每行 8 级）——100k → 200k 的 equity 曲线看不清细节。能不能换成 16 级？

---

*解析时间：2026-07-12 · 第九轮迭代*
*下次解析目标：`v2/__init__.py` + `v2/analyze.py` + `v2/conftest.py` + `v2/data/__init__.py` + `v2/llm/__init__.py` + `v2/signals/__init__.py` + 各 `__main__.py` 和 tests*