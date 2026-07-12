# v2/backtesting/models.py 深度解析

> 文件：`v2/backtesting/models.py` · 48 行 · 3 个 Pydantic model
> 地位：**回测结果的强类型表示**——`Trade` / `PerformanceMetrics` / `BacktestResult`

## 文件作用一句话

**定义回测引擎产出的 3 个数据结构：单笔交易、汇总指标、整体结果。**

## 3 个模型

| 模型 | 行号 | 作用 |
|------|------|------|
| `Trade` | 10-24 | 一笔完成的交易（entry + exit） |
| `PerformanceMetrics` | 27-39 | 一组交易的统计指标 |
| `BacktestResult` | 42-48 | 顶层结果（trades + metrics + equity curve） |

## `Trade`（line 10-24）

```python
class Trade(BaseModel):
    ticker: str
    direction: str                    # "long" or "short"
    entry_date: str
    exit_date: str
    entry_price: float
    exit_price: float
    shares: float
    pnl: float
    return_pct: float
    holding_days: int
    reasoning: str | None = None      # 来自 alpha Signal
    metadata: dict[str, Any]
```

**关键字段**：

| 字段 | 来源 | 用途 |
|------|------|------|
| `pnl` | `shares × (exit_price - entry_price)` | 美元盈亏 |
| `return_pct` | `(exit - entry) / entry` | 百分比盈亏 |
| `reasoning` | `Signal.reasoning` | **审计追溯**——为什么开仓 |
| `metadata` | `Signal.metadata` | alpha 模型上下文 |
| `holding_days` | `exit_date - entry_date` | 持仓时长 |

**特别价值**：`reasoning` 和 `metadata` 让每一笔交易**可追溯到具体的 alpha 模型信号**——这是回测审计的关键。

## `PerformanceMetrics`（line 27-39）

```python
class PerformanceMetrics(BaseModel):
    total_return_pct: float
    annualized_return_pct: float
    sharpe_ratio: float
    max_drawdown_pct: float
    win_rate: float
    n_trades: int
    n_long: int
    n_short: int
    avg_return_pct: float
    avg_holding_days: float
```

**10 个指标**——和 `v2/backtesting/engine.py::_compute_metrics` 一一对应：

| 指标 | 公式 | 解读 |
|------|------|------|
| `total_return_pct` | `(final - initial) / initial` | 累计收益 |
| `annualized_return_pct` | `(1 + total)^(1/years) - 1` | 年化 |
| `sharpe_ratio` | `mean / std × sqrt(trades_per_year)` | 风险调整收益 |
| `max_drawdown_pct` | 峰值到谷底最大跌幅 | 最坏情况 |
| `win_rate` | 正收益交易占比 | 胜率 |
| `n_trades` | 总交易数 | 统计基础 |
| `n_long` / `n_short` | 多/空头数 | 方向分布 |
| `avg_return_pct` | 平均单笔收益 | 期望值 |
| `avg_holding_days` | 平均持仓天数 | 换手率 |

## `BacktestResult`（line 42-48）

```python
class BacktestResult(BaseModel):
    trades: list[Trade] = Field(default_factory=list)
    metrics: PerformanceMetrics | None = None
    equity_curve: list[float] = Field(default_factory=list)
```

**3 个字段**：

- `trades` —— 所有交易明细
- `metrics` —— 汇总指标
- `equity_curve` —— `[100000, 100500, 99800, ...]` 每笔交易结算后的组合价值

**为什么 `equity_curve` 是 `list[float]` 而非 `list[Trade]`**？

- 一笔交易结算后才记录一个 equity point
- equity point **不含** 单笔交易的所有元数据（ticker / direction 等）——只看时序
- `trades` 列表单独保留完整信息

**对 demo 渲染的价值**：

`v2/demo/backtest.py` 把 `equity_curve` 直接画成 sparkline——这是它能快速展示的关键。

## 设计取舍

### 1. 为什么不用 `v2/models.py::TradeOrder`？

v2 顶层 `v2/models.py` 有 `TradeOrder`（line 57-66），但 `v2/backtesting/models.py::Trade` 是**独立的**——不复用。

**原因**：
- `TradeOrder` 是**计划**（执行前的指令）
- `Trade` 是**结果**（执行后的实际）

**语义不同，不能混用**。`Trade` 是 backtest 引擎的"内部账本"。

### 2. `direction` 用 `str` 而非 `Literal["long", "short"]`

```python
direction: str  # "long" or "short"
```

而不是：

```python
direction: Literal["long", "short"]
```

**判断**：保守。允许未来加 `spread` / `pair` 之类。

### 3. `reasoning: str | None = None`

✅ **optional**——alpha model 没给 reasoning（PEAD 是 quant）时为 None

❌ **required `""`** —— 空字符串还是语义不明的"无 reasoning"

**判断**：合理。

### 4. `equity_curve: list[float]` 没标 dict

❌ **如果用 dict** —— `[{date, equity}, ...]` 更明确

✅ **`list[float]`** —— 简单、序列化快

**判断**：demo 自己 join date 和 equity（`v2/demo/backtest.py::_build_equity_curve` 里）。

## 对 fork → CN 版本的启示

### A. 字段要扩展

A 股特有字段：

| 字段 | 含义 |
|------|------|
| `slippage_pct` | 滑点（A 股涨跌停硬约束可能导致 0 成交） |
| `limit_up_hit` | 触发日是否涨停 |
| `t1_locked` | T+1 解锁前的部分卖出限制 |
| `stamp_tax` | 印花税（A 股卖出收 0.05%） |

**怎么加**：在 `Trade` 加 optional 字段，CN 专属逻辑在 `v2/backtesting/engine.py` 里填。

### B. `metadata` 字段是好设计

CN 化的元数据（涨跌停状态、行业分类、限售解禁等）可以塞进 `metadata`，**不用改 Pydantic schema**。

## 深读问题（自检）

- [ ] `Trade` 没存 `entry_time` 和 `exit_time`，只有 `entry_date` 和 `exit_date`——如果想做 intraday 回测（港股美股常见），需要扩展吗？
- [ ] `PerformanceMetrics` 缺 Sortino / Calmar——为什么没加？什么时候该加？
- [ ] `equity_curve: list[float]` 不带日期——`v2/demo/backtest.py` 自己 join 出来；如果想画带时间轴的图，效率怎样？
- [ ] TradeOrder 和 Trade 在 v2 里共存——这是设计冗余还是有意分层？

---

*解析时间：2026-07-12 · 第七轮迭代*
*下次解析目标：`v2/backtesting/engine.py` + `v2/event_study/engine.py`*