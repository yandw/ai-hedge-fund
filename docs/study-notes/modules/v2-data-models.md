# v2/data/models.py 深度解析

> 文件：`v2/data/models.py` · 277 行 · 9 个 Pydantic 模型
> 地位：**Financial Datasets API 响应的结构化表示**——所有数据从 API 进来都要经过这些模型才能进入流水线。

## 文件作用一句话

**把 Financial Datasets API 的 JSON 响应**全部转成**强类型的 Pydantic 模型**，每个字段一个萝卜一个坑，对应 API 文档里的字段。

## 9 个模型一览

| 模型 | 行号 | 字段数 | 对应 API | 作用 |
|------|------|--------|---------|------|
| `Price` | 19-29 | 6 | `/prices/` | 单根 OHLCV 柱 |
| `FinancialMetrics` | 36-107 | 60+ | `/financial-metrics/` | 财务比率全套 |
| `InsiderTrade` | 113-132 | 17 | `/insider-trades/` | 内部人单笔交易 |
| `CompanyNews` | 148-152 | 5 | `/news/` | 公司新闻 |
| `CompanyFacts` | 158-174 | 14 | `/company/facts/` | 公司元数据 |
| `EarningsData` | 181-220 | 26 | （嵌套） | 单期财报数据 |
| `Earnings` | 224-233 | 5 | `/earnings/`（latest） | 最新财报 |
| `EarningsRecord` | 237-258 | 13 | `/earnings/`（history） | 历史财报清单（**PEAD 用这个**） |
| `Filing` | 265-275 | 9 | `/filings/` | SEC 申报元数据 |

## 所有模型的统一约定

```python
_IGNORE = {"extra": "ignore"}   # line 12

class Xxx(BaseModel):
    model_config = _IGNORE
    ...
```

**`extra="ignore"` 的意义**：

- ✅ 容忍上游 API 增加新字段——你的代码不会因为 FD 多返回一个字段就崩
- ✅ 不丢失上游信息——多出来的字段静默被丢掉（不会因为不认识就报错）
- ❌ 失去类型检查——上游字段名写错你不会知道

**判断**：合理。这是**外部数据适配器**的标准做法——**自己的代码严格，外部的代码宽松**。

## 关键模型详解

### `Price`（line 19-29）—— 最简单

```python
class Price(BaseModel):
    open: float
    close: float
    high: float
    low: float
    volume: int
    time: str    # ISO date
```

**6 个字段，无 nullable**——价格数据要么有要么没，没有"未知"状态。**`time` 用字符串**，原因同 `Signal.date`。

### `FinancialMetrics`（line 36-107）—— **最关键也最复杂**

60+ 字段，按用途分组：

```python
# Point-in-time 核心字段（line 47-53）
filing_date: str | None = None
filing_datetime: str | None = None   # ET 时区，精确到秒

# 估值（line 56-65）
market_cap, enterprise_value, pe_ratio, pb_ratio, ps_ratio,
ev_to_ebitda, ev_to_revenue, fcf_yield, peg_ratio

# 盈利（line 68-72）
gross_margin, operating_margin, net_margin,
roe, roa, roic

# 效率（line 75-80）
asset_turnover, inventory_turnover, receivables_turnover,
days_sales_outstanding, operating_cycle, working_capital_turnover

# 流动性（line 83-86）
current_ratio, quick_ratio, cash_ratio, operating_cash_flow_ratio

# 杠杆（line 89-91）
debt_to_equity, debt_to_assets, interest_coverage

# 增长（line 94-100）
revenue_growth, earnings_growth, book_value_growth,
eps_growth, fcf_growth, operating_income_growth, ebitda_growth

# 每股（line 103-106）
payout_ratio, eps, book_value_per_share, fcf_per_share
```

**两个关键设计点**：

1. **几乎所有比率字段都是 `float | None`**——NaN/Inf 在 API 响应里被服务端转成 null。
2. **`filing_date` / `filing_datetime`** 是 **point-in-time 的关键字段**——`_IGNORE` 之外，这两个字段的存在决定了能不能做严肃回测。

### `EarningsData`（line 181-220）—— 一期财报的完整数据

```python
class EarningsData(BaseModel):
    # 实际 vs 估计（line 184-189）
    revenue, estimated_revenue, revenue_surprise  # "BEAT" | "MISS" | "MEET"
    eps, estimated_eps, eps_surprise               # 同上
    
    # 利润表 / 资产负债表 / 现金流（约 20 个字段）
    net_income, gross_profit, operating_income,
    cash_and_equivalents, total_debt, total_assets, total_liabilities, shareholders_equity,
    net_cash_flow_from_operations, capital_expenditure,
    net_cash_flow_from_investing, net_cash_flow_from_financing,
    change_in_cash_and_equivalents,
    
    # 同比变化（line 214-219）
    revenue_chg, net_income_chg, operating_income_chg, gross_profit_chg, fcf_chg
```

**重点字段**：

- **`revenue_surprise` / `eps_surprise`**：API 已经算好 BEAT/MISS/MEET 标签，**`v2/signals/pead.py` 直接用这个**——不用自己算 surprise。
- **`weighted_average_shares` / `diluted`**：股本，DCF 时算每股价值需要。
- **同比变化字段**（`*_chg`）：API 直接给出环比变化百分比，不用自己算。

### `EarningsRecord`（line 237-258）—— **PEAD 的核心数据**

```python
class EarningsRecord(BaseModel):
    ticker: str
    report_period: str       # 财季末（如 2024-09-30）
    source_type: str         # "8-K" | "10-Q" | "10-K" | "20-F"
    filing_date: str | None  # SEC 实际受理日
    filing_datetime: str | None
    filing_window: str | None
    fiscal_period: str | None
    currency: str | None
    filing_url: str | None
    accession_number: str | None
    quarterly: EarningsData | None
    annual: EarningsData | None
```

**关键点**：

- 一次 SEC filing 一条 record——同一份财季可能会有 `8-K`（最早发布）+ `10-Q`（正式季报）两条 record。
- `filing_date` 决定 PEAD 何时"能看到"——比 `report_period` 晚 3-6 周。
- `filing_window` 是 API 给的"filing 实际覆盖的时间段"，用于异常 case 检测（如 GS 那个 103 天的 retrospective 案例）。

### `Filing`（line 265-275）—— SEC 申报元数据

```python
class Filing(BaseModel):
    ticker, cik, accession_number, filing_type, filing_date,
    report_period, document_count, is_xbrl, url
```

**这个模型在代码里目前没被任何业务模块消费**——是为 `/filings/` endpoint 预留的 schema。

## 设计取舍

### 1. 60+ 字段全列出来 vs 用嵌套 dict？

✅ **全列出来**：

- 类型安全（IDE 自动补全）
- 字段文档（description）
- 序列化清晰

❌ **代价**：新增字段要改这文件——但 `extra="ignore"` 让 FD 加字段时不破坏你的代码。

**判断**：合理。Pydantic 模型的"显式列举"哲学正是 v2 选择的。

### 2. 比率字段为什么不用 Enum？

```python
revenue_surprise: str | None = None  # "BEAT" | "MISS" | "MEET"
```

不是 `Literal["BEAT", "MISS", "MEET"]`，是 `str | None`。

**判断**：保守选择。FD API 文档可能加 `EXCEED` 之类的值，留 string 兼容未来。

### 3. `EarningsData` 和 `EarningsRecord` 都嵌套 EarningsData，但 `Earnings` 也嵌套——为什么？

- `Earnings`（latest 模式）：`quarterly: EarningsData | None; annual: EarningsData | None`
- `EarningsRecord`（history 模式）：同样嵌套

**原因**：FD API 的两种调用模式（`/earnings/` 单 ticker latest vs history）响应结构不一样——单条有 `quarterly/annual` 两个字段（Q 和 A 分别一组数据），历史是 flat list。

## 对 fork → CN 版本的启示

### 1. CN 数据源的字段映射

如果写 `TushareClient`，需要决定：保留 `FinancialMetrics` 字段名还是用 Tushare 字段名？

**两种策略**：

| 策略 | 优点 | 缺点 |
|------|------|------|
| A. 保留 v2 字段名 + 在 client 里映射 | 上层代码不动 | 写一个映射函数，维护成本 |
| B. 用 Tushare 字段名 + 改 v2/models | Tushare 直接对接 | 破坏 v2 一致性 |

**推荐 A**：保留 v2 字段名。alpha model 写一次，所有数据源通用。

### 2. A 股缺失字段怎么办？

| v2 字段 | A 股对应 | 处理 |
|---------|---------|------|
| `filing_date` | `ann_date`（Tushare） | 映射 |
| `market_cap` | `total_mv` | 映射 |
| `roe` | `roe`（Tushare `fina_indicator`） | 直接对应 |
| `revenue_surprise` | **无原生标签** | 自己算：`actual - estimate` 符号 |
| `filing_window` | **无对应** | 留 None 或不调用 |

**特别坑**：`revenue_surprise` 在 A 股要自己算 Tushare 没现成的——需要拿到 `forecast`（一致预期）数据自己比。

## 深读问题（自检）

- [ ] `filing_datetime` 用 ET 时区，对国内 fork 来说意味着什么？（提示：UTC+0 比较，国际化）
- [ ] `EarningsData.revenue_chg` 这种"变化百分比"字段，能不能用来算"超预期幅度"？（提示：要看 PEAD 实际怎么用）
- [ ] 为什么 `Filing` 定义了但没有 `client.get_filings()` Protocol 方法？（提示：检查 `protocol.py`）
- [ ] A 股"超预期"标签缺失时，怎么补这个能力？

---

*解析时间：2026-07-12 · 第二轮迭代*
*下次解析目标：`v2/data/client.py`*