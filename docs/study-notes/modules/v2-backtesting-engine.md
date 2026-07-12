# v2/backtesting/engine.py 深度解析

> 文件：`v2/backtesting/engine.py` · 300 行 · 1 个主类（`BacktestEngine`）
> 地位：**v2 回测引擎的主类**——把 alpha model 的 Signal 转成 Trade + 统计指标

## 文件作用一句话

**对一组 ticker 跑一个 alpha model，按日调 `predict()`，触发开仓，固定持有 N 天，平仓，计算 Sharpe / drawdown 等指标。**

## 关键设计哲学（docstring line 5-12）

```python
"""Backtesting engine — simulate trading an alpha model's views over time.

IMPORTANT — separation of concerns (the "unify" decision):
  - The AlphaModel forms *views* (conviction in [-1, +1]).
  - This engine owns *mechanics* (entry timing, holding period, sizing).
These mechanics are intentionally simple for now (threshold + fixed holding
period + equal-dollar sizing). Week 8 portfolio construction will replace
this harness with real position sizing and risk-aware weighting.
"""
```

**关键诚实**：engine 是**简单 harness**，**Week 8 portfolio construction 会替换**——作者明示这是 v0。

## 主类 API

```python
class BacktestEngine:
    def __init__(self, *, capital=100_000.0, per_trade=10_000.0):
        ...
    
    def run_alpha(
        self,
        model: AlphaModel,
        tickers: list[str],
        data_client: DataClient,
        start_date: str,
        end_date: str,
        *,
        threshold: float = 0.0,
        holding_days: int = 5,
    ) -> BacktestResult:
        ...
```

**3 个核心参数**：

| 参数 | 默认 | 含义 |
|------|------|------|
| `threshold` | 0.0 | `|value|` 大于阈值才开仓 |
| `holding_days` | 5 | 持仓天数（trading days） |
| `capital` | 100k | 初始资金 |
| `per_trade` | 10k | 单笔交易等金额 |

## 主流程（line 53-92）

```python
def run_alpha(self, model, tickers, data_client, start_date, end_date, *, threshold, holding_days):
    trades = []
    for ticker in tickers:
        trades.extend(self._trade_ticker(...))
    
    if not trades:
        return BacktestResult()
    
    trades.sort(key=lambda t: t.entry_date)
    equity_curve = self._build_equity_curve(trades)
    metrics = self._compute_metrics(trades, equity_curve)
    return BacktestResult(trades=trades, metrics=metrics, equity_curve=equity_curve)
```

**4 个步骤**：

1. **每个 ticker 单独跑** `_trade_ticker`
2. **按 entry_date 排序**
3. **构造 equity curve**
4. **算 metrics**

## `_trade_ticker` —— 核心逻辑（line 98-155）

```python
def _trade_ticker(self, model, ticker, data_client, start_date, end_date, *, threshold, holding_days):
    # 1. 拿价格，padded 多拿几天
    end_padded = (_parse_date(end_date) + timedelta(days=holding_days * 2 + 10)).isoformat()
    today = date.today().isoformat()
    if end_padded > today:
        end_padded = today
    prices = data_client.get_prices(ticker, start_date, end_padded)
    if not prices:
        return []
    
    price_map = {p.time[:10]: p.close for p in prices}
    all_days = sorted(price_map)
    grid = [d for d in all_days if start_date <= d <= end_date]
    
    # 2. 边沿触发器扫描
    armed = True   # 仅在"模型回归中性"时重新开仓
    i = 0
    while i < len(grid):
        d = grid[i]
        signal = model.predict(ticker, d, data_client)
        
        if armed and abs(signal.value) > threshold:
            direction = "long" if signal.value > 0 else "short"
            entry_idx = all_days.index(d)
            exit_idx = entry_idx + holding_days
            if exit_idx >= len(all_days):
                break
            trade = self._build_trade(ticker, direction, d, all_days[exit_idx], price_map, holding_days, signal.reasoning, dict(signal.metadata))
            if trade is not None:
                trades.append(trade)
            armed = False
            # 跳到 exit 日期
            i = grid.index(all_days[exit_idx]) if all_days[exit_idx] in grid else len(grid)
            continue
        
        if abs(signal.value) <= threshold:
            armed = True
        i += 1
    
    return trades
```

### 关键机制：`armed` 边沿触发器

```python
armed = True
...
if armed and abs(signal.value) > threshold:
    # 开仓
    armed = False
    ...
if abs(signal.value) <= threshold:
    armed = True
```

**含义**：

- `armed=True` → 准备开仓
- 一旦开仓 → `armed=False`
- 只有 `|value| <= threshold`（中性）→ `armed=True`

**好处**：**避免同一信号触发多次**——比如 PEAD 一次 BEAT 只应该开一次仓。

**对比简单实现**：如果不用 `armed`，PEAD 在 filing 后 4 天每天都触发 `|value|=1.0 > 0`，会**每天开仓一次**——4 次交易、4 倍资金占用。

### 持仓期跳过逻辑

```python
i = grid.index(all_days[exit_idx]) if all_days[exit_idx] in grid else len(grid)
continue
```

开仓后**直接跳到 exit 日期**——不重复调 `model.predict()`，不重叠仓位。

### `all_days` vs `grid` 的区别

```python
all_days = sorted(price_map)
grid = [d for d in all_days if start_date <= d <= end_date]
```

- `all_days` —— 价格序列里所有有数据的交易日
- `grid` —— 在 [start_date, end_date] 范围内的交易日

**为什么要区分**：

- 价格 padded 拿的多（exit 日期可能在 end_date 之后）
- `grid` 是回测"扫描区间"
- `all_days` 是"实际能交易的全部日期"

## `_build_trade` —— 单笔交易构造（line 161-200）

```python
def _build_trade(self, ticker, direction, entry_date, exit_date, price_map, holding_days, reasoning, metadata):
    entry_price = price_map.get(entry_date)
    exit_price = price_map.get(exit_date)
    if entry_price is None or exit_price is None or entry_price <= 0:
        return None    # 数据不够
    
    shares = self._per_trade / entry_price
    
    if direction == "long":
        pnl = shares * (exit_price - entry_price)
        return_pct = (exit_price - entry_price) / entry_price
    else:    # short
        pnl = shares * (entry_price - exit_price)
        return_pct = (entry_price - exit_price) / entry_price
    
    return Trade(
        ticker=ticker,
        direction=direction,
        entry_date=entry_date,
        exit_date=exit_date,
        entry_price=entry_price,
        exit_price=exit_price,
        shares=round(shares, 4),
        pnl=round(pnl, 2),
        return_pct=round(return_pct, 6),
        holding_days=holding_days,
        reasoning=reasoning,
        metadata=metadata,
    )
```

### 关键计算

**仓位**：`shares = per_trade / entry_price`

- 等金额 = 1 万美元 / 100 美元 = 100 股
- 不是 Kelly / 波动率调整

**多空 P&L**：

| Direction | pnl | return_pct |
|-----------|-----|-----------|
| long | `shares × (exit - entry)` | `(exit - entry) / entry` |
| short | `shares × (entry - exit)` | `(entry - exit) / entry` |

**没有摩擦**：无手续费、无滑点、无印花税、无融资利率。

**rounding**：
- `shares` → 4 位
- `pnl` → 2 位（美元 cents）
- `return_pct` → 6 位（基点）

## `_compute_metrics` —— 3 个核心指标（line 226-291）

### Total Return（line 245-247）

```python
final_equity = equity_curve[-1]
total_return_pct = (final_equity - self._capital) / self._capital
```

### Annualized Return（line 249-255）

```python
first_entry = _parse_date(trades[0].entry_date)
last_exit = _parse_date(trades[-1].exit_date)
calendar_days = (last_exit - first_entry).days
years = max(calendar_days / 365.25, 0.01)
annualized = (1 + total_return_pct) ** (1 / years) - 1
```

**注意**：

- 用**实际日历天数**，不是交易日数（保守估计年化）
- `max(years, 0.01)` —— 防 0 除
- `** (1/years) - 1` —— 复利年化

### Sharpe Ratio（line 257-263）

```python
arr = np.array(returns)
avg = float(arr.mean())
std = float(arr.std(ddof=1)) if n > 1 else 1.0
trades_per_year = n / years if years > 0 else n
sharpe = (avg / std) * np.sqrt(trades_per_year) if std > 0 else 0.0
```

**Sharpe 公式**：

```
sharpe = (mean_return / std_return) × sqrt(trades_per_year)
```

**关键细节**：
- `std(ddof=1)` —— Bessel 校正（样本标准差）
- `sqrt(trades_per_year)` —— 年化（假设 trades 均匀分布）

**PEAD Sharpe ≈ 0.33**（注释 line 234）——作者明示"not tradable yet"。

### Max Drawdown（line 265-275）

```python
peak = equity_curve[0]
max_dd = 0.0
for val in equity_curve:
    if val > peak:
        peak = val
    dd = (peak - val) / peak
    if dd > max_dd:
        max_dd = dd
```

**标准实现**：O(n) 一次遍历，找"任何时点距离最近历史峰值的最大跌幅"。

### Win Rate（line 277-278）

```python
wins = sum(1 for r in returns if r > 0)
win_rate = round(wins / n, 4)
```

**注意**：返回 `wins / n`，**不是 `(wins - losses) / n`**——PEAD 报告的 win rate 是简单胜率。

## 设计取舍

### 1. 等金额 vs 波动率调整仓位

✅ **等金额**：

- 简单
- 不同 ticker 直接可比

❌ **波动率调整**：

- 高波动股仓位小
- 风险贡献均衡

**判断**：v0 简化。Week 8 portfolio construction 会替换。

### 2. 无摩擦 vs 含摩擦

✅ **无摩擦**：

- Sharpe 0.33 是"上界"
- 真实交易会更低

❌ **含摩擦**（A 股有印花税 0.05%，港股有 0.13%）：

- 更现实
- 但需要 broker 协议 + tax 规则

**判断**：v0 简化。ROADMAP ⬜ Brokers & execution。

### 3. `armed` 边沿触发器 vs 重叠仓位

✅ **`armed`**：

- 一次信号 → 一次开仓
- PEAD 之类的"事件触发"alpha 自然匹配

❌ **重叠仓位**：

- 同一信号可多次加仓
- 适合趋势跟踪（momentum）

**判断**：合理。**边沿触发**是 quant event-driven 的标准。

### 4. 仓位和资金不联动

✅ **`per_trade` 固定**：

- 不管组合当前 equity 多大
- 100k 时和 200k 时都是 10k 仓位

❌ **按 equity 比例**：

- 风险自动调整
- 但需要重新算 metrics

**判断**：v0 简化。Portfolio construction 应该接管。

## 对 fork → CN 版本的启示

### A. A 股特有约束（必须加）

| 约束 | 当前处理 | CN 必须改 |
|------|---------|----------|
| **T+1** | 模拟 T+0 | 加 `entry_t1_locked` 字段 |
| **涨跌停** | 不考虑 | 加 `exit_attempt_failed: bool` |
| **印花税** | 不扣 | `_build_trade` 加 `cost = pnl * 0.0005`（卖出时） |
| **停牌** | 价格数据缺失 → `exit_price is None` → trade 跳过 | 已隐式处理 ✓ |

### B. 复用的部分

- `_compute_metrics`（指标计算）—— **完全复用**
- `armed` 边沿触发 —— **复用**（PEAD 逻辑不变）
- `_build_equity_curve` —— **复用**

### C. 必须改的部分

- `_build_trade` 加印花税扣除
- `_trade_ticker` 处理涨跌停（如果 exit_price 是 None 表示 exit 当天无法成交——延后到下一个可交易日）
- 默认 `holding_days` 改 1-2（A 股 drift 短）

## 深读问题（自检）

- [ ] `armed=True` 后被设 `False` ——如果之后 **signal.value 还是 > threshold** 但模型"重复给同样观点"（比如 PEAD 同一事件持续触发）——是 bug 还是 feature？
- [ ] `_build_trade` 假设 `per_trade / entry_price` 永远买得起——如果 equity 已经亏光，`shares` 会是负数吗？
- [ ] Sharpe 公式 `mean / std × sqrt(trades_per_year)` 用 trades_per_year 而不是 252——为什么？哪种更准？
- [ ] `max_drawdown_pct` 用 `equity_curve` 而非 per-trade `return_pct`——这两种算法结果一样吗？

---

*解析时间：2026-07-12 · 第七轮迭代*
*下次解析目标：`v2/event_study/engine.py` + `v2/demo/backtest.py`*