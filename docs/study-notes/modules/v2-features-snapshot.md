# v2/features/snapshot.py 深度解析

> 文件：`v2/features/snapshot.py` · 187 行 · 4 个 Pydantic model + 4 个 helper + 1 个 builder
> 地位：**LLM investor agent 的"事实底座"——把数据变成 LLM 能直接推理的紧凑文本**

## 文件作用一句话

**为某个 ticker 在某个 as_of 日期，组装一个 point-in-time 的"事实快照"（历史财务 + 派生聚合），喂给 LLM agent 做判断，避免 LLM 做算术。**

## 4 个核心组件

| 组件 | 行号 | 作用 |
|------|------|------|
| `MIN_PERIODS` | 24 | 至少需要 4 期 TTM 数据 |
| `InsufficientData` | 27-28 | 数据不够时的异常 |
| `PeriodFundamentals` | 31-47 | 单期数据（compact） |
| `FundamentalsSnapshot` | 50-102 | 完整快照（含派生聚合） |
| `build_snapshot` | 105-150 | builder 函数 |
| `_avg` / `_trend` / `_cagr` / `_fmt` | 157-186 | 数值/格式化 helpers |

## 关键设计：`PeriodFundamentals` —— 字段选择是有取舍的

```python
class PeriodFundamentals(BaseModel):
    report_period: str
    filing_date: str | None = None
    market_cap: float | None = None
    price_to_earnings_ratio: float | None = None
    return_on_equity: float | None = None
    gross_margin: float | None = None
    operating_margin: float | None = None
    net_margin: float | None = None
    debt_to_equity: float | None = None
    current_ratio: float | None = None
    revenue_growth: float | None = None
    earnings_per_share: float | None = None
    book_value_per_share: float | None = None
    free_cash_flow_per_share: float | None = None
```

**13 个字段——故意不全选**：

`FinancialMetrics` 有 60+ 字段，这里只挑了 13 个。**为什么**：

✅ **这 13 个恰好覆盖 Buffett/Lynch/Munger 等价值投资者的核心问题**：
- 估值：P/E
- 盈利：gross/op/net margin, ROE, EPS
- 增长：revenue_growth, BVPS
- 偿债：debt_to_equity, current_ratio
- 现金流：FCF/share
- 规模：market_cap

❌ **舍弃的字段**：asset_turnover、inventory_turnover、ROIC 等——对 LLM 推理来说信息密度不够高

**取舍原则**：**LLM context 是稀缺资源**，多给一个字段 = 多烧 token = 不一定更准。13 个是经验最优。

## `FundamentalsSnapshot` —— 不只是数据，还有"已算好的派生指标"

```python
class FundamentalsSnapshot(BaseModel):
    ticker: str
    as_of: str
    sector: str | None = None
    industry: str | None = None
    periods: list[PeriodFundamentals]    # 原始 13 字段历史
    
    # 派生聚合（Python 算好，LLM 不用算）
    roe_avg: float | None = None
    net_margin_avg: float | None = None
    gross_margin_trend: float | None = None
    bvps_cagr: float | None = None
    debt_to_equity_latest: float | None = None
    market_cap_latest: float | None = None
    
    @property
    def content_hash(self) -> str:
        canonical = self.model_dump_json()
        return hashlib.sha256(canonical.encode()).hexdigest()[:24]
```

### 关键点 1：`content_hash` 是 LLM cache key 的关键

```python
@property
def content_hash(self) -> str:
    canonical = self.model_dump_json()
    return hashlib.sha256(canonical.encode()).hexdigest()[:24]
```

**作用**：snapshot 内容变了 → hash 变了 → LLM cache key 变了 → 重新调 LLM

**结果**：**同一只股票 + 同一日期**如果基本面没变，**不会重复调 LLM**。

### 关键点 2：派生指标 = 不让 LLM 做算术

| 派生 | 公式 | 为什么不让 LLM 算 |
|------|------|------------------|
| `roe_avg` | 平均 ROE | LLM 容易把 4 个 ROE 求和而不是平均 |
| `gross_margin_trend` | 最新 - 最旧 | LLM 容易看反方向 |
| `bvps_cagr` | `(latest/oldest)^(1/years) - 1` | CAGR 计算 LLM 几乎一定错 |
| `net_margin_avg` | 平均净利率 | 同 ROE |

**这是工程上非常聪明的设计**：**LLM 强项是判断，弱项是算术**。把算术算好，LLM 只做"高 ROE + 低 PE + 好趋势 → 看多"这种判断。

## `render()` —— 把 snapshot 变 LLM prompt

```python
def render(self) -> str:
    lines = [
        f"Company: {self.ticker}"
        + (f"  |  Sector: {self.sector}" if self.sector else "")
        + (f"  |  Industry: {self.industry}" if self.industry else ""),
        f"As of: {self.as_of} (all data below was publicly filed by this date)",
        "",
        "Summary:",
        f"  Market cap (latest filed): {_fmt(self.market_cap_latest)}",
        f"  ROE avg: {_fmt(self.roe_avg)}  |  Net margin avg: {_fmt(self.net_margin_avg)}",
        f"  Gross margin trend (latest-oldest): {_fmt(self.gross_margin_trend)}",
        f"  Book value/share CAGR: {_fmt(self.bvps_cagr)}",
        f"  Debt/equity (latest): {_fmt(self.debt_to_equity_latest)}",
        "",
        "History (trailing-twelve-month periods, newest first):",
        "period | filed | mktcap | P/E | ROE | gross_m | op_m | net_m | D/E "
        "| curr | rev_gr | EPS | BVPS | FCF/sh",
    ]
    for p in self.periods:
        lines.append(...)
    return "\n".join(lines)
```

**输出长这样**（简化）：

```
Company: AAPL  |  Sector: Technology  |  Industry: Consumer Electronics
As of: 2024-06-01 (all data below was publicly filed by this date)

Summary:
  Market cap (latest filed): 3.2T
  ROE avg: 0.45  |  Net margin avg: 0.25
  Gross margin trend (latest-oldest): 0.04
  Book value/share CAGR: 0.18
  Debt/equity (latest): 1.73

History (trailing-twelve-month periods, newest first):
period | filed | mktcap | P/E | ROE | gross_m | op_m | net_m | D/E | curr | rev_gr | EPS | BVPS | FCF/sh
2024-03-31 | 2024-05-02 | 3.0T | 28.5 | 0.45 | 0.46 | 0.30 | 0.25 | 1.73 | 1.05 | 0.05 | 6.30 | 4.30 | 6.50
2023-12-31 | 2024-02-01 | 2.9T | 27.0 | 0.44 | 0.45 | 0.30 | 0.25 | 1.80 | 1.10 | 0.06 | 6.10 | 4.10 | 6.20
...
```

**设计要点**：

1. **管道符分隔**——LLM 训练数据里表格常见，易解析
2. **"_fmt" 数字格式化**——`3.2T` 比 `3200000000000.0` 易读
3. **"as of ... was publicly filed"**——明示 PIT，避免 LLM 用未公开信息

## `build_snapshot()` —— builder（line 105-150）

```python
def build_snapshot(ticker, as_of, data_client, periods=20):
    metrics = data_client.get_financial_metrics(
        ticker, as_of, period="ttm", limit=periods,
    )
    if len(metrics) < MIN_PERIODS:
        raise InsufficientData(...)
    
    facts = data_client.get_company_facts(ticker)
    
    rows = [
        PeriodFundamentals(**m.model_dump(include=set(PeriodFundamentals.model_fields)))
        for m in metrics
    ]
    
    return FundamentalsSnapshot(
        ticker=ticker,
        as_of=as_of,
        sector=facts.sector if facts else None,
        industry=facts.industry if facts else None,
        periods=rows,
        roe_avg=_avg([m.return_on_equity for m in metrics]),
        ...
    )
```

### 关键设计点

#### 1. PIT 来自 data 层，不是这里

```python
metrics = data_client.get_financial_metrics(ticker, as_of, ...)
```

**`as_of` 透传给 data_client**——PIT 强制在 `FDClient.get_financial_metrics` 里（`filing_date_lte=as_of`），不在这里。

#### 2. `MIN_PERIODS = 4` raise 而不是默认

```python
if len(metrics) < MIN_PERIODS:
    raise InsufficientData(f"{ticker} as of {as_of}: only {len(metrics)} filed periods (need {MIN_PERIODS})")
```

- **不静默回退**——数据不够就 fail
- 异常类型 `InsufficientData` 让 `LLMAgent` 可以特别处理（返回 abstain signal）

#### 3. `model_dump(include=...)` 字段过滤

```python
PeriodFundamentals(**m.model_dump(include=set(PeriodFundamentals.model_fields)))
```

**Pydantic v2 的 `include` 参数**——只把目标 model 需要的字段从 dict 里挑出来。

**好处**：
- ✅ `FinancialMetrics` 有 60 字段，`PeriodFundamentals` 只有 13——不挑就多余字段被忽略（但浪费内存）
- ✅ 显式声明"我需要哪些"

#### 4. **故意不用** `get_market_cap()`（line 127 注释）

```python
# Market cap comes from the most recent FILED metrics row. Deliberately
# NOT data_client.get_market_cap(): that prefers company_facts.market_cap,
# which is latest-only — lookahead in a backtest.
market_cap_latest=metrics[0].market_cap,
```

**关键 PIT 修正**：`FDClient.get_market_cap` 优先用 `company_facts.market_cap`——**这是"现在的市值"，不是"as_of 当时的市值"**。

**正确做法**：用 `metrics[0].market_cap`（最近一期 filing 里带的市值）。

**这是 PIT 哲学在工程上的精确体现**——注释里直接写"lookahead in a backtest"。

#### 5. Sector/industry 是"可接受的近似"

```python
# Sector/industry are slow-moving company attributes; using latest
# facts here is an accepted, documented PIT approximation.
sector=facts.sector if facts else None,
```

**注释明示**：sector/industry 用最新值是**有意的 PIT 近似**——因为这两个属性变化慢（公司不会从"科技"跳到"金融"），且对判断影响小。

**这是诚实的工程取舍**：明确告诉读者"我们这里抄了近路，但理由充分"。

## 4 个数值 helpers（line 157-186）

```python
def _fmt(v): ...                          # 数字 → "3.2T" / "1.5M" / "1.23"
def _avg(values): ...                     # 平均（skip None）
def _trend(values): ...                   # 最新 - 最旧（line 173 注释明示）
def _cagr(values): ...                    # 年化增长率（line 178-186）
```

**特别看 `_cagr`**：

```python
def _cagr(values):
    xs = [v for v in values if v is not None]
    if len(xs) < 2 or xs[-1] is None or xs[-1] <= 0 or xs[0] <= 0:
        return None
    years = (len(xs) - 1) / 4  # quarter-spaced ttm periods
    if years <= 0:
        return None
    return round((xs[0] / xs[-1]) ** (1 / years) - 1, 4)
```

**3 个边界保护**：

1. `xs[-1] <= 0` ——最旧值 ≤ 0 时 CAGR 数学上无解（负数开根）
2. `xs[0] <= 0` ——最新值 ≤ 0 同理
3. `years <= 0` ——防御性

**`(len(xs) - 1) / 4`** ——假设是季度 TTM，4 个 period = 1 年。

## 设计取舍

### 1. 13 个字段 vs 60 个字段

✅ **13 个**：context 紧凑、LLM 关注度集中

❌ **60 个**：信息更全但 LLM 容易"挑花眼"

**判断**：合理。LLM context 是稀缺资源。

### 2. 派生聚合 vs 让 LLM 算

✅ **派生**：LLM 只判断不算

❌ **让 LLM 算**：prompt 更短但 LLM 容易算错

**判断**：**这是 v2 最重要的设计决策之一**。**算术在 Python、判断在 LLM**。

### 3. `MIN_PERIODS = 4`

✅ **4**：

- 4 期 TTM = 1 年数据
- 价值投资至少需要看 1 年趋势

❌ **2**：信息太少

❌ **8**：新股 / 新上市股票会被无理由排除

**判断**：合理。4 是经典价值投资的最小数据量。

## 对 fork → CN 版本的启示

### A. `PeriodFundamentals` 字段映射

CN 化时如果写 `TushareClient`，要确认这 13 个字段在 Tushare 都覆盖：

| 字段 | Tushare 对应 | 备注 |
|------|-------------|------|
| `price_to_earnings_ratio` | `pe_ttm` 或 `pe` | 直接有 |
| `return_on_equity` | `roe`（千分位！） | ⚠️ Tushare 的 `roe` 是百分数还是小数？需要确认 |
| `gross_margin` | `gross_profit_margin` | Tushare 字段名差异 |
| `revenue_growth` | `tr_yoy` | 同比，不是"趋势" |
| `book_value_per_share` | `bps` | 直接有 |
| `free_cash_flow_per_share` | **无** | 需要 `free_cf / total_share` 自己算 |

**CN 化的核心工作**就是这个映射表。

### B. PIT 修正还要做

`build_snapshot` 注释明示"不用 `get_market_cap()` 因为它是 latest-only"——**CN 数据源如果实现 `get_market_cap` 要遵守同样的 PIT 约束**。

### C. Sector/industry 分类

A 股 sector/industry 来自**申万 / 中信 / Wind** 行业分类——CN 化时 `PeriodFundamentals` 字段可能需要扩展（`industry_sw` / `industry_citic`）。

## 深读问题（自检）

- [ ] `PeriodFundamentals.model_fields` 用 set 包了一层——为什么要 `set(...)` 而不是直接传 `PeriodFundamentals.model_fields`？（提示：set 在 Pydantic 里的语义）
- [ ] `render()` 输出**管道符分隔的表格**——为什么不用 markdown 表格？token 效率差多少？
- [ ] `MIN_PERIODS = 4`——A 股新上市公司（如 IPO 不到 1 年）会全部 raise `InsufficientData`，对 PEAD demo（用 earnings history 而不是 fundamentals）有没有影响？
- [ ] `_cagr` 假设 `(len-1)/4` 年——如果数据不是 quarterly 而是 annual 呢？

---

*解析时间：2026-07-12 · 第五轮迭代*
*下次解析目标：`v2/signals/base.py` + `v2/signals/pead.py` + `v2/signals/llm_agent.py` + `v2/signals/buffett.py`*