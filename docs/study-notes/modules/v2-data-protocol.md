# v2/data/protocol.py 深度解析

> 文件：`v2/data/protocol.py` · 89 行 · 1 个 Protocol + 8 个方法签名
> 地位：**v2 数据层的"宪法"——所有数据源（FD、Tushare、AKShare、yfinance）都必须实现这 8 个方法**

## 文件作用一句话

**用 Python 的结构化 typing 定义"什么算合法的 DataClient"，让任何类 duck-type 进入 v2 流水线，零继承。**

## 核心代码（简化）

```python
@runtime_checkable
class DataClient(Protocol):
    def get_prices(self, ticker, start_date, end_date, **kwargs) -> list[Price]: ...
    def get_financial_metrics(self, ticker, end_date, period='ttm', limit=10) -> list[FinancialMetrics]: ...
    def get_news(self, ticker, end_date, start_date=None, limit=1000) -> list[CompanyNews]: ...
    def get_insider_trades(self, ticker, end_date, start_date=None, limit=1000) -> list[InsiderTrade]: ...
    def get_company_facts(self, ticker) -> CompanyFacts | None: ...
    def get_earnings(self, ticker) -> Earnings | None: ...
    def get_earnings_history(self, ticker, limit=12) -> list[EarningsRecord]: ...
    def get_market_cap(self, ticker, end_date) -> float | None: ...
```

## 关键概念

### `@runtime_checkable` Protocol

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class DataClient(Protocol):
    ...
```

**含义**：

- `Protocol` 是 PEP 544 引入的**结构化子类型**——任何类只要方法签名匹配，就**自动满足**这个 Protocol，**不需要 `class Foo(DataClient):`** 显式继承。
- `@runtime_checkable` 让 `isinstance(obj, DataClient)` 在运行时可以工作（默认 Protocol 不支持 isinstance）。

**对 fork 用户的价值**：

```python
class YFinanceClient:           # 注意：没有继承任何东西
    def get_prices(self, ticker, start_date, end_date, **kwargs): ...
    def get_financial_metrics(self, ticker, end_date, ...): ...
    # ... 其他 6 个方法 ...

client: DataClient = YFinanceClient()       # ✅ 自动合法
isinstance(client, DataClient)              # ✅ True（runtime_checkable）
```

**这意味着 CN fork 时写 `TushareClient` 只需要写 8 个方法，零继承、零注册——直接喂给任何期望 `DataClient` 的函数。**

### 8 个方法详解

| # | 方法 | 返回 | 数据源调用 | 用途 |
|---|------|------|------------|------|
| 1 | `get_prices` | `list[Price]` | `/prices/` | 价格序列（回测） |
| 2 | `get_financial_metrics` | `list[FinancialMetrics]` | `/financial-metrics/` | 财务比率（**PIT 关键**） |
| 3 | `get_news` | `list[CompanyNews]` | `/news/` | 新闻（情绪） |
| 4 | `get_insider_trades` | `list[InsiderTrade]` | `/insider-trades/` | 内部人交易 |
| 5 | `get_company_facts` | `CompanyFacts \| None` | `/company/facts/` | 公司元数据（sector 等） |
| 6 | `get_earnings` | `Earnings \| None` | `/earnings/` (latest) | 最近一次财报 |
| 7 | `get_earnings_history` | `list[EarningsRecord]` | `/earnings/` (history) | **历史财报清单**（事件研究、PEAD） |
| 8 | `get_market_cap` | `float \| None` | derived | 当前市值 |

### 三种返回语义（重点！）

```python
# 1. 空列表 → "数据真的不存在"
return []   # 正常，例如这只股票没新闻

# 2. None → "数据不存在但应该是单个对象"
return None  # 例如这只股票没在 CompanyFacts 表里

# 3. raise → "基础设施失败"
raise SomeError(...)   # 401 / 429 / 网络异常 / 服务错误
```

**这是 v2 数据层"fail-loud"哲学的契约层**——**空 ≠ 失败**。backtest 看到空才能正确判断"无信号"，看到 raise 才会把整个回测崩掉（避免"假阴性"信号）。

### 关键约束：`get_financial_metrics` 的 point-in-time 语义

文件 docstring 第 41-43 行（核心约束）：

```python
"""
get_financial_metrics must be point-in-time: return only data that was
publicly filed by *end_date*, not data whose fiscal period ended by then.
"""
```

**含义**：

- ❌ **错**：返回所有 `report_period <= end_date` 的数据（看未来——Q3 财报 9 月发布，10 月回测时不应该"知道"）
- ✅ **对**：返回所有 `filing_date <= end_date` 的数据（SEC 实际受理日才是公开日）

**实现层**：`FDClient.get_financial_metrics` 强制 `filing_date_lte=end_date` 参数传给 API（见 [v2/data/client.py 待解析]）。

**对 CN fork 的影响**：A 股的 PIT 字段是 `ann_date`（公告日）/ `披露日期`——Tushare 和 AKShare 都有，但**用法要小心**：很多 API 默认按 `report_period`（报告期）返回，要显式过滤 `ann_date`。

## 为什么不用 `ABC` 抽象基类？

```python
# 方案 A（Protocol，本项目用）
class DataClient(Protocol):
    def get_prices(...): ...

class MyClient:        # ✅ 自动满足
    def get_prices(...): ...

# 方案 B（ABC，没用）
from abc import ABC, abstractmethod

class DataClient(ABC):
    @abstractmethod
    def get_prices(...): ...

class MyClient(DataClient):    # 必须显式继承
    def get_prices(...): ...
```

**对比**：

| 维度 | Protocol | ABC |
|------|----------|-----|
| 继承要求 | 无（duck typing） | 必须 `class Foo(DataClient)` |
| `isinstance` 检查 | 需要 `@runtime_checkable` | 原生支持 |
| 方法定义 | 可写实现也可只声明 | `@abstractmethod` 装饰 |
| 第三方类适配 | 任何类都行 | 第三方必须修改源码 |
| 运行时错误时机 | 调用时 | 实例化时 |

**v2 选 Protocol 的理由**（社区贡献友好）：
- 用户写的第三方 client **不需要继承**任何东西
- 测试时可以用一个简化的 mock class，零侵入
- 数据源是动态集成的——Protocol 是**最松的契约**

## 与其他模块的关系

```
v2/data/protocol.py    ← DataClient 接口（duck typing）
       ↑
       │ 满足（实现）
       │
   ┌───┴──────────────────────────┐
   │                              │
v2/data/client.py::FDClient   v2/data/cached.py::CachedDataClient
（具体实现）                  （包装任意 DataClient 加缓存）
                                       │
                                       │ 包装
                                       ↓
                            yfinance / Tushare / AKShare client
                            （用户自己写）
```

**关键观察**：`CachedDataClient` 接受**任何 `DataClient`**——它本身就是一个合法的 DataClient（实现同样的 8 个方法），所以可以**多层嵌套**：

```python
fd = FDClient(api_key=...)              # 真实 FD
cached = CachedDataClient(fd)           # 加磁盘缓存
# 未来：再加一层 retry、logging、rate-limit，都不用改业务代码
```

## 设计取舍

### 1. Protocol vs ABC

✅ 选了 Protocol——为社区贡献者降低门槛
✅ 副作用：失去实例化期检查（写了 MyClient 但忘了实现 `get_news`，到调用时才发现）

**判断**：合理。**调用时报错**比"每个 client 必须继承"更友好。

### 2. 8 个方法——够不够？

✅ 覆盖了：
- 价格 / 财报 / 公司事实（基本面研究）
- 新闻 / 内部人交易（情绪、事件）
- 市值 / 历史财报（量化）

❌ 没覆盖：
- 期权数据
- 利率 / 汇率 / 大宗商品
- 宏观经济指标（GDP、CPI）
- 实时报价（last trade）

**判断**：8 个方法刚好覆盖 alpha model 的最小需求集合。更多数据源属于"v2.5"，不是 v2 核心。

### 3. `**kwargs` 只在 `get_prices` 上有

```python
def get_prices(self, ticker, start_date, end_date, **kwargs) -> list[Price]: ...
```

`get_prices` 允许 `interval` / `interval_multiplier` 等扩展参数（来自 FD API）。其他方法严格签名。

**判断**：合理。`get_prices` 是唯一一个有"interval"概念的方法（价格分时 vs 日 vs 周）。

## 对 fork → CN 版本的启示

### A. 写 `TushareClient` / `AKShareClient` 的模板

```python
class TushareClient:
    """实现 DataClient Protocol 的 Tushare 客户端。"""
    
    def __init__(self, token: str):
        import tushare as ts
        ts.set_token(token)
        self._pro = ts.pro_api()
    
    def get_prices(self, ticker, start_date, end_date, **kwargs):
        # ticker 是 A 股代码，如 '600519.SH'
        # Tushare 用 '600519.SH' 但 daily 接口要分开 market
        code = ticker.split('.')[0]
        market = ticker.split('.')[1]  # 'SH' / 'SZ'
        df = self._pro.daily(ts_code=ticker, start_date=start_date.replace('-', ''), 
                              end_date=end_date.replace('-', ''))
        return [Price(time=r['trade_date'], open=r['open'], ...) for _, r in df.iterrows()]
    
    def get_financial_metrics(self, ticker, end_date, period='ttm', limit=10):
        # 关键：用 ann_date 过滤，不要用 report_period！
        df = self._pro.fina_indicator(ts_code=ticker, 
                                       ann_date_lte=end_date.replace('-', ''),
                                       limit=limit)
        return [FinancialMetrics(**row) for _, row in df.iterrows()]
    
    # ... 其他 6 个方法
```

**两个 A 股特有的坑**：

1. **日期格式**：Tushare 用 `YYYYMMDD`（无连字符），FD 用 `YYYY-MM-DD`——client 层做转换。
2. **PIT 字段**：Tushare 的 `fina_indicator` / `disclosure_date` 是公告日；很多教程错误地用 `end_date`（报告期）做时间过滤——**会泄露未来**。

### B. 不要碰这个 Protocol 文件

CN 化**不修改 `protocol.py`**——它是契约，CN 化是给契约加实现，不是改契约本身。

## 深读问题（自检）

- [ ] 为什么 Protocol 比 ABC 更适合"社区贡献友好"的项目？
- [ ] `CachedDataClient` 自己也实现了 8 个方法，所以它**既是** DataClient 又**消费** DataClient——这种"自指"有什么好处和风险？
- [ ] `get_earnings` 返回 `Earnings | None` 而不是 `list[Earnings]`，为什么？
- [ ] 如果你想加 `get_macro_indicators(country, date) -> list[MacroIndicator]`，应该改 Protocol 还是另开一个 Protocol？（提示：interface segregation principle）

---

*解析时间：2026-07-12 · 第一轮迭代*
*下次解析目标：`v2/data/models.py`（FD 响应 Pydantic 模型）*