# src/backtesting/* + src/backtester.py + src/cli/input.py + src/data/* + src/tools/api.py 深度解析

> 文件：`src/backtesting/{engine,controller,trader,portfolio,valuation,metrics,benchmarks,output,types,cli}.py`（10 个）+ `src/backtester.py` + `src/cli/input.py` + `src/data/{cache,models}.py` + `src/tools/api.py`
> 地位：**v1 回测的完整实现 + CLI 解析 + 数据层 + API 客户端**

## 总览

v1 的回测比 v2 复杂得多——拆成 10 个模块协作。这反映 v1 的"过度工程"风格。

## `src/backtesting/engine.py` —— 协调器

### 文件作用一句话

**v1 BacktestEngine 是"协调器"——把 agent 调用、trade execution、valuation、metrics 拼起来，按日循环跑。**

### 8 个组件注入

```python
class BacktestEngine:
    def __init__(self, *, agent, tickers, start_date, end_date, initial_capital, model_name, model_provider, selected_analysts, initial_margin_requirement):
        ...
        self._portfolio = Portfolio(...)
        self._executor = TradeExecutor()
        self._agent_controller = AgentController()
        self._perf = PerformanceMetricsCalculator()
        self._results = OutputBuilder(initial_capital=self._initial_capital)
        self._benchmark = BenchmarkCalculator()
        ...
```

**6 个组件**——每个都是单独模块（看下面）。

### 主流程（推测）

```python
def run_backtest(self):
    for date in trading_days:
        # 1. 调 agent
        agent_output = self._agent_controller.run_agent(
            self._agent,
            tickers=self._tickers,
            start_date=self._start_date,
            end_date=date,
            portfolio=self._portfolio.get_snapshot(),
            ...
        )
        
        # 2. 执行交易
        for ticker, decision in agent_output["decisions"].items():
            self._executor.execute_trade(ticker, decision["action"], decision["quantity"], current_price, self._portfolio)
        
        # 3. 记录组合价值
        portfolio_value = calculate_portfolio_value(self._portfolio, current_prices)
        self._portfolio_values.append({
            "Date": date,
            "Portfolio Value": portfolio_value,
            ...
        })
    
    # 4. 算 metrics
    self._performance_metrics = self._perf.compute_metrics(self._portfolio_values, self._benchmark)
    return self._performance_metrics
```

### 与 v2 的对比

| 维度 | v1 engine | v2 engine |
|------|-----------|-----------|
| 组件数 | 6 个 | 1 个 |
| agent 调用 | 每天调 N 个 LLM | 不调 LLM（quant）或缓存（LLM） |
| trade execution | 4 个 action（buy/sell/short/cover） | long/short 二选一 |
| metrics | 6 个指标 | 10 个指标 |
| benchmark | S&P 500 对比 | 没实现 |

## `src/backtesting/controller.py` —— AgentController

### 文件作用一句话

**调 agent + 把 agent 输出"标准化"成 BacktestEngine 期望的格式。**

### 关键代码（line 30-58）

```python
output = agent(
    tickers=list(tickers),
    start_date=start_date,
    end_date=end_date,
    portfolio=portfolio_payload,
    ...
)

# Normalize outputs
decisions_in = dict(output.get("decisions", {}))
analyst_signals_in = dict(output.get("analyst_signals", {}))

normalized_decisions = {}
for ticker in tickers:
    d = decisions_in.get(ticker, {})
    action = d.get("action", "hold")
    qty = d.get("quantity", 0)
    
    # Coerce qty to float
    try:
        qty_val = float(qty)
    except Exception:
        qty_val = 0.0
    
    # Coerce action to enum
    try:
        action = Action(action).value
    except Exception:
        action = Action.HOLD.value
    
    normalized_decisions[ticker] = {"action": action, "quantity": qty_val}
```

**关键防御**：

- ❌ LLM 输出的 `action` 不合法 → 强制 `"hold"`
- ❌ `quantity` 不是数字 → 强制 0
- ✅ 默认 fallback 防止 LLM 输出异常让 backtest 崩

## `src/backtesting/trader.py` —— TradeExecutor

### 文件作用一句话

**根据 agent 的 decision 调 portfolio 的 4 个 action 方法之一。**

### 4 个 action 映射

```python
if action == BUY:    portfolio.apply_long_buy(ticker, qty, price)
if action == SELL:   portfolio.apply_long_sell(ticker, qty, price)
if action == SHORT:  portfolio.apply_short_open(ticker, qty, price)
if action == COVER:  portfolio.apply_short_cover(ticker, qty, price)
# hold / unknown → no-op
```

**简单 dispatcher**——所有"交易逻辑"都在 `Portfolio` 类里。

## `src/backtesting/portfolio.py` —— Portfolio（推测）

**Portfolio 类**管理：

- `cash` —— 现金
- `positions[ticker]` —— 多/空持仓 + 成本基础
- `margin_used` —— 用了多少保证金
- `realized_gains` —— 已实现收益

**4 个 apply 方法**：

- `apply_long_buy` —— 开多 / 加多
- `apply_long_sell` —— 平多 / 减多
- `apply_short_open` —— 开空 / 加空
- `apply_short_cover` —— 平空 / 减空

每个都更新 `cash`、`positions`、`margin_used`、`realized_gains`。

**对比 v2**：v2 没有 Portfolio 类——`Trade` 是不可变的 record，不维护状态。v1 是真"账户"模拟。

## `src/backtesting/valuation.py` —— calculate_portfolio_value

### 文件作用一句话

**根据当前市场价格算组合的总价值（含未实现 P&L）。**

```python
def calculate_portfolio_value(portfolio, current_prices):
    # Long positions
    for ticker, pos in portfolio.positions.items():
        long_value = pos["long"] * current_prices[ticker]
        total += long_value
        
        # Unrealized P&L
        unrealized = pos["long"] * (current_prices[ticker] - pos["long_cost_basis"])
    
    # Short positions
    for ticker, pos in portfolio.positions.items():
        short_value = pos["short"] * current_prices[ticker]
        total -= short_value  # short value subtracts
        
        # Unrealized gain (opposite of long)
        ...
    
    return total - portfolio.margin_used
```

**关键细节**：short 持仓的"value"是**负数**（因为将来要买回归还，亏了）。

## `src/backtesting/metrics.py` —— PerformanceMetricsCalculator

### 文件作用一句话

**算 6 个 metrics：sharpe / sortino / max_drawdown / long_short_ratio / gross_exposure / net_exposure。**

### 6 个 metrics 详解

| Metric | 公式 | 含义 |
|--------|------|------|
| `sharpe_ratio` | `(avg_return / std_return) × √(annual_factor)` | 风险调整收益 |
| `sortino_ratio` | 类似 Sharpe，但只算下行波动 | 区分好坏波动 |
| `max_drawdown` | 峰值到谷底最大跌幅 | 最坏情况 |
| `long_short_ratio` | long_exposure / short_exposure | 多空比例 |
| `gross_exposure` | \|long\| + \|short\| | 总风险敞口 |
| `net_exposure` | long + short | 净敞口（方向性） |

**对比 v2**：v2 只有 5 个（无 long_short_ratio / gross_exposure / net_exposure），多了 `win_rate` / `n_long` / `n_short` / `avg_holding_days`。

## `src/backtesting/types.py` —— TypedDict 定义

```python
class Action(str, Enum):
    BUY = "buy"
    SELL = "sell"
    SHORT = "short"
    COVER = "cover"
    HOLD = "hold"

class PositionState(TypedDict):
    long: int
    short: int
    long_cost_basis: float
    short_cost_basis: float
    short_margin_used: float

class PortfolioSnapshot(TypedDict):
    cash: float
    margin_used: float
    margin_requirement: float
    positions: dict[str, PositionState]
    realized_gains: dict[str, TickerRealizedGains]

class AgentDecision(TypedDict):
    action: ActionLiteral
    quantity: float

class AgentOutput(TypedDict):
    decisions: AgentDecisions
    analyst_signals: AgentSignals

class PortfolioValuePoint(TypedDict, total=False):
    Date: datetime
    Portfolio Value: float
    Long Exposure: float
    Short Exposure: float
    Gross Exposure: float
    Net Exposure: float
    Long/Short Ratio: float

class PerformanceMetrics(TypedDict, total=False):
    sharpe_ratio: Optional[float]
    sortino_ratio: Optional[float]
    max_drawdown: Optional[float]
    ...
```

**v1 vs v2 类型风格**：

| 维度 | v1 | v2 |
|------|----|----|
| 类型 | TypedDict + Enum | Pydantic BaseModel |
| 验证 | 静态检查 | 运行时验证 |
| Action | `class Action(str, Enum)` | `Literal["buy","sell",...]` |

**v2 改进**：用 Pydantic 而不是 TypedDict——**有运行时验证**。

## `src/backtesting/benchmarks.py` —— BenchmarkCalculator

**对比 S&P 500 (SPY) 的同期表现**：

```python
class BenchmarkCalculator:
    def __init__(self, benchmark_ticker="SPY"):
        self.benchmark_ticker = benchmark_ticker
    
    def compute_benchmark_returns(self, start_date, end_date):
        prices = get_prices(self.benchmark_ticker, start_date, end_date, ...)
        return [(prices[i+1] - prices[i]) / prices[i] for i in range(len(prices)-1)]
    
    def alpha(self, portfolio_returns, benchmark_returns):
        """Jensen's alpha: portfolio - benchmark × beta"""
        beta = self._compute_beta(portfolio_returns, benchmark_returns)
        return portfolio_returns.mean() - beta * benchmark_returns.mean()
```

**v1 有 benchmark 对比**——v2 没实现（占位）。

## `src/backtesting/output.py` —— OutputBuilder

**生成回测报告**——表格、图表、P&L summary。

```python
class OutputBuilder:
    def __init__(self, initial_capital):
        self.initial_capital = initial_capital
    
    def build_table_rows(self, ...):
        """生成 CSV/Excel 行"""
        ...
    
    def build_chart_data(self, portfolio_values):
        """生成 equity curve 数据"""
        ...
```

## `src/backtesting/cli.py` —— CLI wrapper

**v1 的 backtester CLI 包装**——主要是 argparse + 调用 BacktestEngine。

## `src/backtester.py` —— CLI 入口（67 行）

```python
from src.main import run_hedge_fund
from src.backtesting.engine import BacktestEngine
from src.cli.input import parse_cli_inputs

if __name__ == "__main__":
    inputs = parse_cli_inputs(
        description="Run backtesting simulation",
        require_tickers=False,
        default_months_back=1,
        include_graph_flag=False,
        include_reasoning_flag=False,
    )
    
    backtester = BacktestEngine(
        agent=run_hedge_fund,
        ...
    )
    
    performance_metrics = run_backtest(backtester)
```

**关键设计**：

- ✅ `agent=run_hedge_fund` —— **每天调一次 agent**（LangGraph workflow）
- ✅ KeyboardInterrupt 优雅退出 + 输出部分结果
- ❌ **每天调 LLM 19 个 agent**——**极慢 + 极贵 + 结果不稳定**

## `src/cli/input.py` —— CLI 参数解析（190+ 行）

### 文件作用一句话

**统一所有 v1 CLI 的参数解析：tickers / analysts / dates / model / ollama。**

### 4 个核心函数

| 函数 | 作用 |
|------|------|
| `add_common_args` | 添加 tickers / analysts / ollama / model 标志 |
| `add_date_args` | 添加 start_date / end_date（含 default_months_back） |
| `parse_tickers` | 拆分逗号分隔的 tickers |
| `select_analysts` | 选择 analysts（含 interactive prompt） |

### `select_analysts` 关键代码（line 75-80）

```python
def select_analysts(flags: dict | None = None) -> list[str]:
    if flags and flags.get("analysts_all"):
        return [a[1] for a in ANALYST_ORDER]
    
    if flags and flags.get("analysts"):
        return [a.strip() for a in flags["analysts"].split(",") if a.strip()]
```

**支持 3 种方式**：

1. `--analysts-all` 选全部
2. `--analysts burry,wood` 选指定
3. interactive questionary 选择

**对比 v2**：v2 CLI 更简单（没有 19 个 analyst 选择）——**这是 v1 的复杂度遗留**。

## `src/data/cache.py` —— 内存缓存（60+ 行）

### 文件作用一句话

**API 响应的内存缓存**——不是磁盘（v2 是磁盘）。

### 关键实现

```python
class Cache:
    def __init__(self):
        self._prices_cache: dict[str, list[dict]] = {}
        self._financial_metrics_cache: dict[str, list[dict]] = {}
        ...
    
    def _merge_data(self, existing, new_data, key_field):
        """Merge by key field (e.g., time, report_period)"""
        if not existing:
            return new_data
        existing_keys = {item[key_field] for item in existing}
        merged = existing.copy()
        merged.extend([item for item in new_data if item[key_field] not in existing_keys])
        return merged
```

### 与 v2 `CachedDataClient` 对比

| 维度 | v1 Cache | v2 CachedDataClient |
|------|----------|----------------------|
| 存储 | **内存 dict** | **磁盘 JSON 文件** |
| 进程间共享 | ❌ 重启进程缓存丢 | ✅ 重启后还在 |
| key 粒度 | ticker only | `(method, params)` hash |
| 失败语义 | 缓存任何响应 | 只缓存成功响应 |

**v2 优势**：

- 跨进程持久（backtest 重启能复用）
- hash key 更精确（不同参数不串）

## `src/data/models.py` —— v1 数据模型（推测）

类似 `v2/data/models.py`，但字段可能不一样。

## `src/tools/api.py` —— v1 Financial Datasets 客户端

### 关键差异（v1 vs v2）

```python
def get_financial_metrics(ticker, end_date, period="ttm", limit=10, api_key=None):
    # v1: 用 report_period_lte，不是 filing_date_lte
    url = f"https://api.financialdatasets.ai/financial-metrics/?ticker={ticker}&report_period_lte={end_date}&limit={limit}&period={period}"
```

**对比 v2**：

```python
# v2: 强制 filing_date_lte
url = f"https://api.financialdatasets.ai/financial-metrics/?ticker={ticker}&filing_date_lte={end_date}&period={period}&limit={limit}"
```

**v1 用 `report_period_lte`**——**这是 lookahead leak**！

- `report_period_lte=2024-06-01` → 返回 Q1 2024（period_end 3/31）的数据
- 但 Q1 2024 财报**实际**filing 在 2024-05（晚 1 个月）
- **回测在 4 月时不应该"知道"Q1 数据**

**v2 修了**：用 `filing_date_lte`——严格按 SEC 实际受理日。

### 429 retry：v1 vs v2

**v1**：
```python
delay = 60 + (30 * attempt)    # 60s, 90s, 120s, 150s
```

**v2**：
```python
_RETRY_DELAYS = (5, 15, 30)    # 5s, 15s, 30s
```

**v1 比 v2 慢**——但更保守（不容易再被限流）。

### 其他 v1 工具函数

- `get_company_news` —— 公司新闻
- `get_insider_trades` —— 内部人交易
- `search_line_items` —— 自定义财务 line items（**v1 特有**，v2 没有）
- `prices_to_df` —— Price list → pandas DataFrame
- `get_price_data` —— 别名 / 包装

`search_line_items` 是 **v1 特有**——v2 通过 `FundamentalsSnapshot` 直接消费 Pydantic model，没有"自定义 line items"概念。

## v1 vs v2 回测系统对比

| 维度 | v1 backtesting | v2 backtesting |
|------|---------------|----------------|
| 模块数 | 10 | 1 |
| 组件注入 | 6 个组件 | 无 |
| agent 调用 | 每天调 LLM（19 个 analyst） | cache 命中不调 LLM |
| 仓位决定 | LLM 决定 quantity | 确定性等金额 |
| Portfolio 状态 | 完整账户模拟 | 简单 Trade 列表 |
| Benchmark 对比 | S&P 500 | 没实现 |
| 数据 lookahead | ❌ `report_period_lte` | ✅ `filing_date_lte` |
| Cache | 内存（重启动丢） | 磁盘（持久） |

**结论**：v1 比 v2 复杂 10 倍，**但回测数学不诚实**（lookahead + LLM 不稳定）。v2 简化是因为**严肃化**。

## 对 fork → CN 版本的启示

### A. 完全不要 fork v1 backtesting

**直接用 v2 backtesting engine**——已深度覆盖 PEAD、Buffett 等。

### B. v1 的 risk_manager piecewise 函数可以搬到 v2/risk/manager.py

**有现成代码**——`calculate_volatility_adjusted_limit` 是真实可用的实现。

### C. v1 的 `search_line_items` 不需要移植

**v2 通过 Pydantic model + snapshot 已覆盖**——CN 化时直接用 snapshot.render()。

### D. CLI 参数解析可以借 v1 的成熟度

v2 CLI 当前非常简陋——v1 的 `parse_cli_inputs` 有：
- questionary interactive 选择
- tickers 验证
- date 默认值

CN 化可以**把 v1 的 CLI 风格搬到 v2**。

## 深读问题（自检）

- [ ] v1 backtest 每天调 LLM 19 个 agent——一个回测 250 天 × 19 agent × 12 LLM API call = 57000 calls。**这成本多少？**
- [ ] v1 Cache 是内存——多 backtest 跑需要重启服务吗？
- [ ] v1 `report_period_lte` 导致的 lookahead——v1 README 里说"这是 quant trading tool"——**这个 lookahead 在 v1 时代是 bug 还是"行业惯例"？**
- [ ] v1 的 Portfolio 类管理 margin_used 等状态——v2 的 Trade 列表是 immutable——**哪种更适合做"持久 ledger"**？

---

*解析时间：2026-07-12 · 第十七轮迭代*
*下次解析目标：tests/* + app/* —— 合并成 1 篇*