# src/agents/* quant 风格 agent + portfolio_manager 深度解析

> 文件：`src/agents/{valuation, sentiment, fundamentals, technicals, news_sentiment, growth_agent, portfolio_manager}.py`
> 地位：**v1 的 7 个核心 agent**——6 个 quant 风格（无人名）+ 1 个最终决策者

## 总览：7 个 quant agent

| 文件 | 行数 | 风格 | 输出 |
|------|------|------|------|
| `valuation.py` | ~500 | DCF + WACC + Owner Earnings | signal |
| `sentiment.py` | ~300 | 综合多个 LLM agent 的 signal | signal |
| `fundamentals.py` | ~200 | 基本面打分 | signal |
| `technicals.py` | ~200 | RSI / MACD / 趋势 | signal |
| `news_sentiment.py` | ~150 | 新闻情绪 | signal |
| `growth_agent.py` | ~150 | 增长指标 | signal |
| `portfolio_manager.py` | ~250 | LLM 决定仓位 | **decisions** |

## 共同模式（所有 v1 agent）

```python
def xxx_agent(state: AgentState, agent_id: str = "xxx_agent"):
    """标准 4 步：拿数据 → 算 → 调 LLM → 写回 state"""
    
    data = state["data"]
    end_date = data["end_date"]            # ← 注意：end_date 从 state 拿
    tickers = data["tickers"]
    api_key = get_api_key_from_state(state, "FINANCIAL_DATASETS_API_KEY")
    
    for ticker in tickers:
        progress.update_status(agent_id, ticker, "Fetching...")
        metrics = get_financial_metrics(ticker, end_date, ...)    # ← PIT？
        # ... 计算 ...
        # ... 调 LLM ...
        # ... 写回 state ...
```

**所有 v1 agent 的共同问题**：

### ❌ **没有 filing_date 过滤**

```python
metrics = get_financial_metrics(ticker, end_date, period="ttm", limit=8)
```

`get_financial_metrics`（v1 版）不传 `filing_date_lte=end_date`——**回测会泄露未来**。

**这是 v1 的根本问题**——也是 v2 commit `Fix lookahead leak` 修的源头。

### ❌ **end_date 在 state 里但 agent 不严格用它**

`state["data"]["end_date"]` 拿了之后**可能**只用最新数据——**没有"as of that date" 的语义**。

### ✅ **统一接口**

每个 agent 都是函数 `(state, agent_id) -> delta_state`——LangGraph 兼容。

## `src/agents/portfolio_manager.py` —— LLM 决定仓位

### 文件作用一句话

**v1 的最终决策 agent——拿所有 analyst signals + 风险限额，调 LLM 决定每个 ticker 的 buy/sell/short/cover/hold + quantity。**

### 关键 Pydantic models（line 13-21）

```python
class PortfolioDecision(BaseModel):
    action: Literal["buy", "sell", "short", "cover", "hold"]
    quantity: int
    confidence: int
    reasoning: str

class PortfolioManagerOutput(BaseModel):
    decisions: dict[str, PortfolioDecision]
```

**5 个 action**——比 v2 的 `TradeOrder` 多一个 `hold`。

### 主函数（line 25-93）

```python
def portfolio_management_agent(state, agent_id="portfolio_manager"):
    portfolio = state["data"]["portfolio"]
    analyst_signals = state["data"]["analyst_signals"]
    tickers = state["data"]["tickers"]
    
    # 1. 收集每个 ticker 的 risk limits
    position_limits = {}
    current_prices = {}
    max_shares = {}
    signals_by_ticker = {}
    for ticker in tickers:
        ...
        risk_data = analyst_signals.get(risk_manager_id, {}).get(ticker, {})
        position_limits[ticker] = risk_data.get("remaining_position_limit", 0.0)
        current_prices[ticker] = float(risk_data.get("current_price", 0.0))
        max_shares[ticker] = int(position_limits[ticker] // current_prices[ticker]) if current_prices[ticker] > 0 else 0
        
        # 2. 压缩 analyst signals
        ticker_signals = {}
        for agent, signals in analyst_signals.items():
            if not agent.startswith("risk_management_agent") and ticker in signals:
                sig = signals[ticker].get("signal")
                conf = signals[ticker].get("confidence")
                if sig is not None and conf is not None:
                    ticker_signals[agent] = {"sig": sig, "conf": conf}
        signals_by_ticker[ticker] = ticker_signals
    
    # 3. 调 LLM
    result = generate_trading_decision(
        tickers, signals_by_ticker, current_prices, max_shares,
        portfolio, agent_id, state,
    )
    
    # 4. 写回 state
    message = HumanMessage(
        content=json.dumps({ticker: decision.model_dump() for ticker, decision in result.decisions.items()}),
        name=agent_id,
    )
    return {"messages": state["messages"] + [message], "data": state["data"]}
```

### 关键设计点

#### 1. **LLM 决定 quantity**

```python
class PortfolioDecision(BaseModel):
    action: Literal["buy", "sell", "short", "cover", "hold"]
    quantity: int    # ← LLM 决定股数
    confidence: int
    reasoning: str
```

**问题**：

- ❌ LLM 不懂"组合层面的相关性、波动率"
- ❌ 同一个 prompt 多次跑，**quantity 可能不同**——回测数学无效
- ❌ max_shares 只是"上限"——LLM 还是可能超

**对比 v2**：v2 完全不让 LLM 决定 quantity——只用 conviction 触发等金额仓位。

#### 2. **`max_shares` 是 risk-adjusted 上限**

```python
max_shares[ticker] = int(position_limits[ticker] // current_prices[ticker])
```

`position_limits` 来自 `risk_management_agent` 的 `remaining_position_limit`——这是 v1 唯一的"硬闸门"。

#### 3. **`signals_by_ticker` 压缩**

```python
ticker_signals[agent] = {"sig": sig, "conf": conf}
```

**只保留 sig + conf**——LLM 不看 reasoning（reasoning 是审计用）。

### 与 v2 的对比

| 维度 | v1 portfolio_manager | v2 (无对应，由 backtest engine 处理) |
|------|---------------------|-------------------------------------|
| 决定者 | LLM | 确定性代码 |
| quantity 来源 | LLM 输出 | 等金额 `per_trade / price` |
| 风险约束 | max_shares 软约束 | BacktestEngine hardcoded |
| 输出 | `decisions` dict | `Trade` 列表 |
| 可重复性 | ❌ LLM 不稳定 | ✅ 完全确定 |

**v1 的根本问题**：**让 LLM 做仓位决定**。这正是 v2 修复的。

## `src/agents/valuation.py` —— DCF + WACC 综合估值

### 文件作用一句话

**对每只 ticker 跑 4 种估值方法（Owner Earnings / DCF / P/E / P/B），加权综合，输出估值结论。**

### 4 种估值方法

| 方法 | 公式 | 用途 |
|------|------|------|
| Owner Earnings | `NI + D&A - CapEx - ΔWC` | Buffett 风格 |
| DCF (with WACC) | `Σ FCF / (1+WACC)^t + TV` | 经典现金流折现 |
| P/E Model | `EPS × fair_PE` | 相对估值 |
| P/B Model | `BPS × fair_PB` | 资产估值 |

### 代码组织

```python
def calculate_owner_earnings_value(net_income, depreciation, capex, working_capital_change, growth_rate):
    ...

def calculate_wacc(market_cap, total_debt, cash, interest_coverage, ...):
    ...

def calculate_dcf_value(fcf, wacc, growth_rate, terminal_growth):
    ...

def calculate_pe_value(eps, growth_rate):
    ...

def calculate_pb_value(book_value_per_share, roe):
    ...

# 主函数
def valuation_analyst_agent(state, agent_id="valuation_analyst_agent"):
    for ticker in tickers:
        metrics = get_financial_metrics(ticker, end_date, period="ttm", limit=8, api_key=api_key)
        # ... 调 4 个 valuation 方法 ...
        # ... 加权综合 ...
        # ... 调 LLM 总结 ...
```

### 关键观察

**v1 valuation 是"过度工程"样本**：

- 600 行 4 个估值模型
- 每个模型又手算了一遍 Python 早就能算的（DCF、WACC）
- **最后还是问 LLM "你怎么看"**

**对比 v2**：v2 的 snapshot 只给 LLM 13 个数字 + 派生指标——LLM 自己判断"高 ROE + 低 PE"是不是好。**v2 把算术放 Python，把判断放 LLM**——比 v1 优雅。

## `src/agents/sentiment.py` —— 综合多个 LLM agent

### 文件作用一句话

**聚合 sentiment、technicals、fundamentals、valuation、news_sentiment 等 agent 的 signal，输出"综合情绪分"。**

### 关键代码

```python
def sentiment_analyst_agent(state, agent_id="sentiment_analyst_agent"):
    # 1. 拿所有 agent 的 signal
    sentiment_data = {}
    for ticker in tickers:
        ticker_signals = {}
        for agent_name, signals in analyst_signals.items():
            if ticker in signals:
                ticker_signals[agent_name] = signals[ticker]
        sentiment_data[ticker] = ticker_signals
    
    # 2. 调 LLM 综合
    prompt = ChatPromptTemplate.from_messages([
        ("system", "You are a sentiment analyzer..."),
        ("human", f"Ticker signals: {sentiment_data}"),
    ])
    
    result = call_llm(
        prompt=prompt,
        pydantic_model=SentimentOutput,
        agent_name=agent_id,
        state=state,
        default_factory=lambda: SentimentOutput(signal="neutral", confidence=50, reasoning="Insufficient data"),
    )
```

**本质**：`sentiment` agent 是**meta-agent**——它的 input 是其他 agent 的输出。

**问题**：

- ❌ LLM 看了 12 个其他 agent 的 signals——**计算量爆炸**
- ❌ 组合信号失真（"meta-LLM 看 LLM 看 LLM" 的递归）

**v2 的做法**：

- 没有 meta-agent
- portfolio construction 由代码做（v2/portfolio/ 占位）

## `src/agents/fundamentals.py` —— 基本面打分

### 文件作用一句话

**基于 ROE / margins / debt / growth 给每只 ticker 打分，调 LLM 综合。**

### 关键代码

```python
def fundamentals_analyst_agent(state, agent_id="fundamentals_analyst_agent"):
    for ticker in tickers:
        metrics = get_financial_metrics(ticker, end_date, period="ttm", limit=8, api_key=api_key)
        
        # 打分（手算）
        score = 0
        if metrics[0].return_on_equity and metrics[0].return_on_equity > 0.15:
            score += 2
        if metrics[0].debt_to_equity and metrics[0].debt_to_equity < 0.5:
            score += 2
        ...
        # 调 LLM
        result = call_llm(...)
```

**类似 valuation**——手算分 + 调 LLM 综合。

**和 v2 的对比**：

| v1 fundamentals | v2 snapshot.roe_avg |
|----------------|---------------------|
| 手算 score，LLM 综合 | **Python 算**（v2 不调 LLM 算术） |

**v2 更纯**——LLM 只看数字，不看分数。

## `src/agents/technicals.py` —— 技术指标

### 文件作用一句话

**用 RSI / MACD / 趋势指标判断 ticker。**

### 关键代码

```python
def technical_analyst_agent(state, agent_id="technical_analyst_agent"):
    for ticker in tickers:
        prices = get_prices(ticker, ...)
        prices_series = pd.Series([p.close for p in prices])
        
        # RSI
        rsi = calculate_rsi(prices_series, 14)
        
        # MACD
        macd, signal, histogram = calculate_macd(prices_series)
        
        # 趋势
        trend = determine_trend(prices_series)
        
        # 调 LLM 综合
        ...
```

**和 v2 的对比**：

- v2 的 `QuantModel._compute_rsi` 也是这个目的（但当前没人用）
- v1 是**多 agent**之一（用 RSI），v2 把 RSI 做成**共享 helper**

## `src/agents/news_sentiment.py` —— 新闻情绪

### 文件作用一句话

**拉最近的新闻，让 LLM 判断情绪。**

### 关键代码

```python
def news_sentiment_agent(state, agent_id="news_sentiment_agent"):
    for ticker in tickers:
        news = get_news(ticker, end_date, ...)    # ← PIT？end_date 没用对
        
        prompt = f"Analyze these news for {ticker}:\n{news}"
        result = call_llm(...)
```

**问题**：

- ❌ 没有限制 news 数量 / 时间窗口
- ❌ LLM 看到一堆新闻直接判断——没有时间权重

## `src/agents/growth_agent.py` —— 增长指标

### 文件作用一句话

**看 revenue / earnings / book value 增长率。**

### 关键代码

```python
def growth_analyst_agent(state, agent_id="growth_analyst_agent"):
    for ticker in tickers:
        metrics = get_financial_metrics(...)
        line_items = search_line_items(...)
        
        revenue_growth = (line_items[0].revenue - line_items[1].revenue) / line_items[1].revenue
        earnings_growth = ...
        
        # 调 LLM
        ...
```

**和 v2 的对比**：

- v2 的 `FundamentalsSnapshot` 已经算出 `revenue_growth` + `bvps_cagr`
- v1 还在 agent 里手算

## 7 个 quant agent 的共同教训

| v1 模式 | v2 替代 |
|--------|--------|
| 每个 agent 自己手算指标 | 共享 `FundamentalsSnapshot`（Python 算好） |
| 每个 agent 调 LLM 综合 | LLM 只看 `snapshot.render()` + 决策 |
| 每个 agent 用 `end_date` 但 PIT 不严格 | `predict(ticker, date, data_client)` + data 层强制 PIT |
| 7 个 quant agent 都用 `get_financial_metrics` 同一接口 | 单接口 `DataClient.get_financial_metrics` |
| 信号进入 `analyst_signals` dict | 信号是 Pydantic `Signal` 对象 |

**v1 的 7 个 quant agent 是"一个模型一个 LLM 调用"**——v2 抽象成"一个接口多种实现"。

## 对 fork → CN 版本的启示

### A. CN 化的 quant agent 必须改的：

| 改 | 原因 |
|----|------|
| **加 PIT 过滤** | `get_financial_metrics(..., filing_date_lte=...)`（A 股是 `ann_date_lte`） |
| **不用 LLM 算术** | 让 Python 算（参考 v2 `snapshot.py`） |
| **signal 用 Pydantic** | `value ∈ [-1, +1]`，不用 dict |
| **不调 LLM 决定仓位** | 用 `BacktestEngine` 替代 portfolio_manager |

### B. CN 化的 quant agent 模板：

```python
class CN_TechnicalsModel(QuantModel):
    @property
    def name(self):
        return "cn_technicals"
    
    def predict(self, ticker, date, data_client):
        prices = data_client.get_prices(ticker, ...)
        prices_series = pd.Series([p.close for p in prices])
        
        rsi = self._compute_rsi(prices_series)
        ...
        
        # 用 v2 的 _sigmoid 把 RSI 转 [-1, +1]
        if rsi > 70:
            value = -0.3 * self._sigmoid(rsi - 70)    # 超买 → 看空
        elif rsi < 30:
            value = 0.3 * self._sigmoid(30 - rsi)    # 超卖 → 看多
        else:
            value = 0.0
        
        return Signal(
            model_name=self.name,
            ticker=ticker,
            date=date,
            value=value,
            reasoning=f"RSI={rsi:.1f}",
            components={"rsi": rsi},
        )
```

**50 行 vs v1 的 600 行**——**直接复用 v2 的 `QuantModel` helper**。

## 深读问题（自检）

- [ ] `portfolio_manager` 的 `quantity` 是 LLM 输出——同一 prompt 多次跑 quantity 会不会变？
- [ ] `valuation` 4 种方法加权综合——权重是硬编码还是 LLM 决定？
- [ ] `sentiment` 是 meta-agent——它看所有其他 agent 的 signal，会不会**计算爆炸**（token 太多）？
- [ ] 所有 quant agent 都**没用 filing_date 过滤**——这意味着什么？（答案：回测数学不诚实）

---

*解析时间：2026-07-12 · 第十五轮迭代*
*下次解析目标：剩余 v1 agents（12 个 personas + risk_manager）— 集中一个 deep dive*
*下次解析目标：src/backtesting/{engine,controller,trader,portfolio,valuation,metrics,benchmarks,output,types,cli}.py + src/backtester.py — 集中一个 deep dive*
*下次解析目标：tests/* + app/* — 集中一个 deep dive*