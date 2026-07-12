# src/agents/12 personas + risk_manager 深度解析（合并）

> 文件：`src/agents/{warren_buffett, charlie_munger, ben_graham, peter_lynch, phil_fisher, cathie_wood, michael_burry, bill_ackman, stanley_druckenmiller, rakesh_jhunjhunwala, mohnish_pabrai, nassim_taleb, aswath_damodaran}.py` + `risk_manager.py`
> 地位：**v1 的 12 个 LLM persona + 1 个 risk manager agent**——每个都遵循同一模板

## 12 personas 总览

| Persona | 风格 | 关键 metric | 关键 file 行数 |
|---------|------|-------------|---------------|
| Warren Buffett | 价值 + 长期 | ROE, moat, margin, DCF | 826 |
| Charlie Munger | 理性 + 质量 | ROE, FCF, moat | 较短 |
| Ben Graham | 深度价值 | P/B, NCAV | 较短 |
| Peter Lynch | 增长可懂 | PEG, "买你懂的" | 较短 |
| Phil Fisher | scuttlebutt | R&D, 管理层 | 较短 |
| Cathie Wood | 颠覆创新 | R&D %, TAM, 增长 | 较长 |
| Michael Burry | 逆向 | 估值, 资产负债表 | 较短 |
| Bill Ackman | 激进股东 | 估值 + 治理 | 较短 |
| Stanley Druckenmiller | 宏观 | 利率 / 增长 | 较短 |
| Rakesh Jhunjhunwala | 印度大牛 | 长期 + 增长 | 较短 |
| Mohnish Pabrai | Dhandho | 极少交易 + 集中 | 较短 |
| Nassim Taleb | 反脆弱 | 凸性 / 尾风险 | 较短 |
| Aswath Damodaran | 学术估值 | DCF + WACC | 较长 |

## 共同模板（所有 12 personas）

每个 persona 文件遵循**完全相同**的结构：

```python
class XxxSignal(BaseModel):
    signal: Literal["bullish", "bearish", "neutral"]
    confidence: int / float
    reasoning: str


def xxx_agent(state: AgentState, agent_id: str = "xxx_agent"):
    data = state["data"]
    end_date = data["end_date"]
    tickers = data["tickers"]
    api_key = get_api_key_from_state(state, "FINANCIAL_DATASETS_API_KEY")
    
    analysis_data = {}
    xxx_analysis = {}
    
    for ticker in tickers:
        # 1. 拿数据
        progress.update_status(agent_id, ticker, "Fetching financial metrics")
        metrics = get_financial_metrics(ticker, end_date, period="ttm", limit=8, api_key=api_key)
        
        progress.update_status(agent_id, ticker, "Gathering financial line items")
        line_items = search_line_items(ticker, [...], end_date, period="ttm", limit=8, api_key=api_key)
        
        # 2. 手算多个 sub-analysis
        progress.update_status(agent_id, ticker, "Analyzing X")
        x_analysis = analyze_X(metrics, line_items)
        ...
        
        # 3. 综合打分
        total_score = ...
        
        # 4. 调 LLM 拿最终判断
        xxx_output = generate_xxx_output(ticker=..., analysis_data=..., state=state, agent_id=agent_id)
        
        # 5. 写回 state
        xxx_analysis[ticker] = {
            "signal": xxx_output.signal,
            "confidence": xxx_output.confidence,
            "reasoning": xxx_output.reasoning,
        }
    
    # 6. 写回全局 state
    message = HumanMessage(content=json.dumps(xxx_analysis), name=agent_id)
    if state["metadata"]["show_reasoning"]:
        show_agent_reasoning(xxx_analysis, agent_id)
    state["data"]["analyst_signals"][agent_id] = xxx_analysis
    
    return {"messages": [message], "data": state["data"]}
```

**所有 12 个 persona 都是这个模板**——只有"手算哪些指标"和"LLM 的 system prompt"不同。

## 共同问题

### ❌ 1. 没有 filing_date 过滤

```python
metrics = get_financial_metrics(ticker, end_date, period="ttm", limit=8, api_key=api_key)
```

`get_financial_metrics`（v1 版）不传 `filing_date_lte=end_date`——**所有 12 个 persona 都有 lookahead 漏洞**。

### ❌ 2. 手算 + LLM 综合 = 重复劳动

```python
# 手算 5-6 个 sub-score
fundamentals_score = ...
consistency_score = ...
moat_score = ...
management_score = ...
pricing_power_score = ...

# 综合
total_score = sum([...])

# 再调 LLM 决定
result = call_llm(prompt, XxxSignal, ...)
```

**为什么浪费**：

- Python 算分但 LLM 也"理解"分——重复
- LLM 直接看数字应该能做得一样好

### ❌ 3. persona 之间高度相似

**12 个 persona 大部分是"价值 + 长期"的变体**：

- Warren Buffett：高质量 + 合理价格
- Charlie Munger：高质量 + 长期
- Ben Graham：低估
- Mohnish Pabrai：Dhandho（= 集中 + 低估）
- Aswath Damodaran：DCF 学术

**只有少数明显不同**：

- Cathie Wood：颠覆 / 增长
- Michael Burry：逆向
- Stanley Druckenmiller：宏观
- Nassim Taleb：反脆弱

**v1 的 12 个 persona 制造"个性化错觉"——但底层代码都一样**。

### ✅ 共同优点

- **统一的 state 接口**——LangGraph fan-out 跑得起来
- **每个 persona 有清晰的投资哲学 prompt**——LLM 能区分
- **输出格式一致**（Pydantic `XxxSignal`）——portfolio_manager 能消费

## 各 persona 的差异点（细节）

### Warren Buffett（826 行）—— 最复杂

8 个 sub-analysis：

| 分析 | 评分维度 |
|------|---------|
| `analyze_fundamentals` | ROE / Debt / Margins / Current Ratio |
| `analyze_consistency` | Earnings growth trend |
| `analyze_moat` | ROIC / Operating margin / Asset turnover |
| `analyze_management_quality` | Buybacks / Dividends |
| `analyze_pricing_power` | Gross margin trend |
| `calculate_owner_earnings` | NI + D&A - CapEx - ΔWC |
| `analyze_book_value_growth` | BVPS CAGR |
| `calculate_intrinsic_value` | 3-stage DCF |

**这是 v1 persona 复杂度的天花板**——其他 persona 都类似但少几个 analysis。

### Cathie Wood（较长）—— 颠覆投资

3 个 sub-analysis（不同于 Buffett）：

| 分析 | 评分维度 |
|------|---------|
| `analyze_disruptive_potential` | R&D 占比、行业趋势 |
| `analyze_innovation_growth` | Revenue 增长、毛利率趋势 |
| `analyze_cathie_wood_valuation` | 高增长 scenario DCF |

**特点**：R&D 投入占比权重高——这是颠覆型投资的关键信号。

### Aswath Damodaran（较长）—— 学术估值

DCF + WACC + 多个 scenario（保守 / 基准 / 乐观）——**学术派的严谨**。

### 其他 9 个 persona

- Ben Graham：Net Net 估值
- Peter Lynch：PEG + "买你懂的"
- Phil Fisher：scuttlebutt（管理层访谈）
- Michael Burry：逆向 + 资产负债表深度分析
- Bill Ackman：质量 + 治理
- Charlie Munger：高质量 + 长期
- Stanley Druckenmiller：宏观 + 利率敏感
- Mohnish Pabrai：Dhandho（集中 + 低估）
- Rakesh Jhunjhunwala：印度大牛（high growth + long-term）
- Nassim Taleb：反脆弱（凸性 / tail risk）

**每个的"投资哲学"都是 LLM system prompt 里的核心**——Python 代码层面**只是手算几个指标**。

## 共同输出格式

每个 persona 输出 `XxxSignal`：

```python
class XxxSignal(BaseModel):
    signal: Literal["bullish", "bearish", "neutral"]
    confidence: int    # 或 float
    reasoning: str
```

**3 个字段**——和 v2 `Signal` 不一样：

| 维度 | v1 XxxSignal | v2 Signal |
|------|-------------|-----------|
| 方向 | `signal: str` | `value: float ∈ [-1, +1]` |
| 信心 | `confidence: int` | (通过 value 隐含) |
| 论据 | `reasoning: str` | `reasoning: str \| None` |

**v1 的 signal 是 enum + 数字**，v2 是单一连续值——**v2 整合度更高**。

## `src/agents/risk_manager.py` —— v1 唯一风控

### 文件作用一句话

**算每只 ticker 的波动率，给出 volatility-adjusted position limit，写入 `analyst_signals[risk_manager_id][ticker]` 让 portfolio_manager 消费。**

### 主流程

```python
def risk_management_agent(state, agent_id="risk_management_agent"):
    portfolio = state["data"]["portfolio"]
    data = state["data"]
    tickers = data["tickers"]
    
    # 1. 拿所有 ticker 的价格，算波动率
    for ticker in all_tickers:
        prices = get_prices(ticker, start_date, end_date, api_key=api_key)
        volatility_metrics = calculate_volatility_metrics(prices_df)
    
    # 2. 算 correlation matrix
    correlation_matrix = ...
    
    # 3. 算 current_price + total_portfolio_value
    ...
    
    # 4. 对每个 ticker 算 position limit（line 250+）
    for ticker in tickers:
        remaining_position_limit = calculate_volatility_adjusted_limit(
            volatility_data[ticker],
            total_portfolio_value,
            ...
        )
        risk_analysis[ticker] = {
            "remaining_position_limit": remaining_position_limit,
            "current_price": current_prices[ticker],
            ...
        }
    
    # 5. 写回 state
    state["data"]["analyst_signals"][agent_id] = risk_analysis
```

### `calculate_volatility_adjusted_limit`（核心，已在 DEEP_READ 里批评过）

```python
def calculate_volatility_adjusted_limit(
    volatility_data: dict,
    total_portfolio_value: float,
    current_price: float,
    portfolio_pct_portfolio_var: float,
    ...
) -> float:
    if annualized_volatility < 0.15:    vol_multiplier = 1.25
    elif annualized_volatility < 0.30:  vol_multiplier = 1.0 - (annualized_volatility - 0.15) * 0.5
    elif annualized_volatility < 0.50:  vol_multiplier = 0.75 - (annualized_volatility - 0.30) * 0.5
    else:                                vol_multiplier = 0.50
    
    vol_multiplier = max(0.25, min(1.25, vol_multiplier))
    
    return ...
```

**问题**：

- ❌ 4 段 piecewise 函数——没有理论依据
- ❌ magic numbers (`0.15`, `1.25`, `0.5`) 都是凭空选的
- ❌ 该函数**没有 LLM 调用**——纯代码——意味着应该搬到 v2 量化代码里

### `calculate_volatility_metrics`

```python
def calculate_volatility_metrics(prices_df):
    daily_returns = prices_df["close"].pct_change().dropna()
    daily_volatility = daily_returns.std()
    annualized_volatility = daily_volatility * np.sqrt(252)
    
    # 历史波动率的 percentile（用于"现在波动算不算高"）
    rolling_vol = daily_returns.rolling(window=60).std() * np.sqrt(252)
    volatility_percentile = percentileofscore(rolling_vol.dropna(), annualized_volatility)
    
    return {
        "daily_volatility": daily_volatility,
        "annualized_volatility": annualized_volatility,
        "volatility_percentile": volatility_percentile,
    }
```

**3 个指标**：

- `daily_volatility`：日波动率
- `annualized_volatility`：年化（× √252）
- `volatility_percentile`：当前波动率在历史上的 percentile

**用 `scipy.stats.percentileofscore`**——这是 v1 唯一用 scipy 的地方（其他都是 numpy）。

### 与 v2 的对比

| 维度 | v1 risk_manager | v2 risk |
|------|----------------|--------|
| 实现 | 完整 300+ 行 | **6 行 docstring 占位** |
| 位置 | LangGraph node | 没集成进 engine |
| 决定仓位 | 通过 position_limit（软约束） | 引擎直接等金额 |
| 波动率 | 算且用 | 算但没用（v2 risk 是占位） |
| Correlation matrix | 算（但只展示） | 没实现 |

**v1 实际上有"硬闸门"——portfolio_manager 必须尊重 position_limit**。但 portfolio_manager 是 LLM，**实际上 LLM 可能超**。

**v2 的策略**：v2 risk 是占位，但 v2 engine 用等金额仓位——**天然有"总仓位不超过 capital"的约束**——不需要 LLM 配合。

## 12 personas + risk_manager 的共同设计教训

### 1. **Persona 是"prompt 工程"——不是代码工程**

**重要认知**：

- 12 个 persona 的**代码差异**很小（都是手算 3-8 个 sub-analysis）
- **真正差异在 LLM system prompt**
- v2 的 "persona = name + system prompt" 完全抓住这一点——**省掉 12 个 200-800 行的 .py 文件**

### 2. **没有 PIT 过滤——回测数学不诚实**

**所有 12 个 persona 的根本问题**——v2 commit `Fix lookahead leak` 修了。

### 3. **手算 + LLM 综合是浪费**

```python
# 浪费：先算 5 个 sub-score，再让 LLM 看 sub-score
total_score = sum of 5 sub_scores
result = call_llm(prompt_with_total_score, ...)

# 更好（v2 风格）：让 LLM 直接看数字
# Python 只算派生指标（均值、CAGR），不算"分数"
result = call_llm(prompt_with_raw_metrics, ...)
```

### 4. **`Literal` enum signal vs `value ∈ [-1, +1]`**

v1 用 enum → v2 用 float。

**v2 的优势**：可以做数学运算（`composite_score = 0.5 × signal_a + 0.5 × signal_b`）。

**v1 的 enum 只能"投票"**——更粗糙。

## 对 fork → CN 版本的启示

### A. CN 化的 personas 模板

```python
class DuanYongpingAgent(LLMAgent):    # ← v2 的基类
    @property
    def name(self):
        return "duan_yongping"
    
    def get_system_prompt(self):
        return """你是段永平。'做对的事情，把事情做对'。'敢为天下后'。Stop doing stupid things。

# 评估清单
1. 商业模式：简单到你能看懂吗？
2. 文化：管理层有'品格'吗？
3. 差异化：长期有没有定价权？
4. 现金流：是否产生持续的、不需融资的自由现金流？
5. 估值：用 4 年盈利估算的回报率 > 6% 银行存款吗？

# 信号
- bullish: 好的生意 + 合理价格
- bearish: 难懂的生意 / 价格过高 / 需融资
- neutral: 好生意但价格反映 / 数据不足

# 硬规则：只用提供的数据，不引用未公开信息，不编数字
"""
```

**50 行**——比 v1 的 600-800 行小一个数量级。

### B. v1 persona 改 CN 的工作量

| 改 | 估计时间 |
|----|---------|
| 12 个 persona 改成 v2 风格 | 1 天 |
| 加 PIT 过滤（data 层） | 0（v2 数据层已做） |
| risk_manager 移到 v2 risk | 1 天 |
| tests 复用 v2 contract test | 0.5 天 |

### C. v1 的 risk_manager 算法可以直接搬到 v2

`calculate_volatility_adjusted_limit` 虽然 piecewise 但**有现成代码**——可以**直接搬到 v2/risk/manager.py**。

但应该**用配置表替代**：

```python
# v2/risk/manager.py 未来实现
VOL_BANDS = [
    (0.15, 1.25),   # vol < 0.15 → multiplier 1.25
    (0.30, 1.00),   # vol 0.15-0.30 → linear
    (0.50, 0.75),   # vol 0.30-0.50 → linear
    (float('inf'), 0.50),  # vol > 0.50 → 0.50
]
```

## 深读问题（自检）

- [ ] 12 个 persona 的 LLM system prompt 多长？有什么共同模式？
- [ ] `signal: Literal["bullish", "bearish", "neutral"]` + `confidence: int` 这种**离散输出**——v2 用 `value: float ∈ [-1, +1]` 是进步还是退步？
- [ ] risk_manager 的 piecewise vol limit 函数——**如果以后用 Kelly / risk-parity 替代**——会破坏哪些 persona 的"预期"？
- [ ] 12 个 persona 的**英文 prompt**直接翻译成中文，能行吗？（中文 LLM 对中文 prompt 的反应更好）

---

*解析时间：2026-07-12 · 第十六轮迭代*
*下次解析目标：src/backtesting/* (8 个文件) + src/backtester.py + src/cli/input.py + src/data/* + src/tools/api.py —— 合并成 1-2 篇*
*下次解析目标：tests/ + app/ —— 各 1 篇*