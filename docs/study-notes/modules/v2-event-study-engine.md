# v2/event_study/engine.py 深度解析

> 文件：`v2/event_study/engine.py` · 414 行 · 1 个公共 API + 3 个内部函数
> 地位：**事件研究引擎**——算 earnings 公告前后的异常收益（CAR），验证 PEAD 之类的 alpha 在历史上是否真的有效

## 文件作用一句话

**对一组 ticker 的 earnings events，跑市场模型（CAPM-style）算累积异常收益（CAR），按 source_type 分组输出统计指标。**

## 这是什么？

事件研究（event study）是金融学的经典工具：
1. 取一个事件（earnings 公告）
2. 在事件前后各取一段时间窗口
3. **估计**事件前股票和市场的关系（市场模型：`R_stock = α + β × R_market`）
4. 在事件窗口内，**观察**股票真实收益 vs 模型预测收益 → **异常收益 AR**
5. **累积** AR = CAR
6. 跨事件统计 CAR 的分布（t-test + bootstrap CI）

**v2 实现里**：CAR 是**PEAD 的学术基础**——如果 (0, +5) CAR 在 BEAT 后显著为正，PEAD 才真有戏。

## 7 个核心常量（line 53-61）

```python
_MARKET_TICKER = "SPY"             # 市场代理
_ESTIMATION_START = -250           # 估计窗口起点（事件前 250 交易日）
_ESTIMATION_END = -11              # 估计窗口终点（事件前 11 交易日）
_MIN_ESTIMATION_DAYS = 200         # 最少要 200 天估计数据
_MAX_EVENT_WINDOW = 20             # 事件后最多看 20 天
_RETROSPECTIVE_CUTOFF_DAYS = 45    # 45 天 retrospective filter
_CAR_WINDOWS = [(0, 1), (0, 5), (0, 20)]  # 3 个 CAR 窗口
```

### 关键常量的设计依据

#### `_ESTIMATION_START = -250` ≈ 1 年

**理由**：足够估计 α、β，又不会因太远而失效（β 漂移）。

#### `_ESTIMATION_END = -11` **不是 -1**

**为什么不是 -1**：

- 如果用 [-250, -1]，估计窗口包含事件前一天 → 可能有 announcement drift 渗入
- -11 给 10 天的"buffer"

**注释明示**（line 56-57）：

> "10-day buffer (-11 instead of -1) prevents pre-announcement drift / leakage from contaminating the alpha/beta estimates."

#### `_MIN_ESTIMATION_DAYS = 200`

**防御性**：理论上 -250 到 -11 是 240 天，但市场可能有停牌/数据缺失。少于 200 天就跳过这个事件。

#### `_CAR_WINDOWS = [(0, 1), (0, 5), (0, 20)]`

**3 个窗口**：

| 窗口 | 含义 | 学术对应 |
|------|------|---------|
| (0, +1) | 公告当天 + 次日 | 立即反应 |
| (0, +5) | 一周内 | 短期漂移 |
| (0, +20) | 一月内 | 中期漂移 |

PEAD 学术论文主要看 (0, +5) 和 (0, +20)。

## 主流程：`compute_car`（line 68-130）

```python
def compute_car(tickers, data_client, *, earnings_limit=12, market_ticker="SPY", n_bootstrap=10_000, rng_seed=None, require_eps_surprise=False):
    today = date.today().isoformat()
    
    # 1. 拿 SPY 一次
    spy_prices = data_client.get_prices(market_ticker, "2023-01-01", today)
    if not spy_prices:
        return EventStudyResult(skipped_tickers=list(tickers))
    spy_closes = {p.time[:10]: p.close for p in spy_prices}
    
    # 2. 每个 ticker 单独处理
    all_events = []
    skipped = []
    for ticker in tickers:
        events = _compute_ticker_events(ticker, data_client, spy_closes, earnings_limit=earnings_limit)
        if events:
            all_events.extend(events)
        else:
            skipped.append(ticker)
    
    # 3. 可选过滤
    if require_eps_surprise:
        all_events = [e for e in all_events if e.eps_surprise is not None]
    
    # 4. 聚合
    aggregates = _aggregate(all_events, n_bootstrap, rng_seed)
    
    return EventStudyResult(events=all_events, aggregates=aggregates, skipped_tickers=skipped)
```

### 4 个步骤

1. **一次 SPY 调用**——共享给所有 ticker
2. **每个 ticker 单独处理**——`_compute_ticker_events`
3. **可选过滤**（只保留有 EPS surprise 标签的）
4. **跨 ticker 聚合**——`_aggregate`

## `_compute_ticker_events` —— 单 ticker 处理（line 137-205）

5 个步骤：

### Step 1: 拿 earnings history

```python
records = data_client.get_earnings_history(ticker, limit=earnings_limit)
```

### Step 2: 45 天 retrospective filter

```python
records = _filter_retrospective(records)
```

**详见 `_filter_retrospective`（line 369-391）**：

```python
def _filter_retrospective(records):
    kept = []
    for r in records:
        filing = _parse_date(r.filing_date)
        report = _parse_date(r.report_period)
        if (filing - report).days < _RETROSPECTIVE_CUTOFF_DAYS:
            kept.append(r)
    return kept
```

**和 `pead.py` 里的 `_RETROSPECTIVE_CUTOFF_DAYS = 45` 是同一个常量、同一个思想**——跨模块共享。

### Step 3: 一次性拿价格

```python
min_date = min(_parse_date(r.filing_date) for r in records)
max_date = max(_parse_date(r.filing_date) for r in records)
today = date.today()
price_start = (min_date - timedelta(days=400)).isoformat()
price_end = min(max_date + timedelta(days=35), today).isoformat()

stock_prices = data_client.get_prices(ticker, price_start, price_end)
```

**关键**：**一次 API 调用覆盖所有事件**——而不是每个事件单独 fetch。

**400 天**：足够最早的 event 的 estimation window（-250 交易日 ≈ -360 日历日）。
**35 天**：足够最近 event 的 post-event window（+20 交易日 ≈ +30 日历日）。

### Step 4: 对齐 trading days

```python
stock_closes = {p.time[:10]: p.close for p in stock_prices}
trading_days = sorted(set(stock_closes) & set(spy_closes))
```

**关键**：**取交集**——只保留 stock 和 SPY 都有交易的日期。

**为什么**：算 returns 时需要对齐，否则 `R_stock[t]` 和 `R_spy[t]` 不是同一日。

### Step 5: 算 returns

```python
stock_close_arr = np.array([stock_closes[d] for d in trading_days])
spy_close_arr = np.array([spy_closes[d] for d in trading_days])

stock_returns = np.diff(stock_close_arr) / stock_close_arr[:-1]
spy_returns = np.diff(spy_close_arr) / spy_close_arr[:-1]
return_days = trading_days[1:]
```

**注意细节**：

- `np.diff(closes) / closes[:-1]` —— 简单百分比收益
- `return_days[i]` 是 `returns[i]` 发生的**日期**（即 `trading_days[i+1]`）
- `len(returns) = len(trading_days) - 1`（第一天没有前一日对比）

## `_process_event` —— 单个事件处理（line 212-291）

```python
def _process_event(record, stock_returns, spy_returns, return_days, day_to_idx):
    event_date_str = record.filing_date
    
    # 1. 找 day 0（事件日，可能跨周末/节假日）
    event_idx = _find_event_idx(event_date_str, return_days, day_to_idx)
    if event_idx is None:
        return None
    
    # 2. 估计窗口 [-250, -11]
    est_start = event_idx + _ESTIMATION_START
    est_end = event_idx + _ESTIMATION_END
    if est_start < 0 or est_end < 0:
        return None
    
    stock_est = stock_returns[est_start : est_end + 1]
    spy_est = spy_returns[est_start : est_end + 1]
    if len(stock_est) < _MIN_ESTIMATION_DAYS:
        return None
    
    # 3. 拟合市场模型
    model = fit_market_model(stock_est, spy_est)
    
    # 4. 事件窗口 [0, +20]
    evt_start = event_idx
    evt_end = min(event_idx + _MAX_EVENT_WINDOW, len(stock_returns) - 1)
    stock_evt = stock_returns[evt_start : evt_end + 1]
    spy_evt = spy_returns[evt_start : evt_end + 1]
    
    # 5. 算异常收益
    daily_ar = compute_abnormal_returns(stock_evt, spy_evt, model.alpha, model.beta)
    
    # 6. 累积成 CAR
    cars = {}
    n_days = len(daily_ar)
    for start, end in _CAR_WINDOWS:
        if end < n_days:
            cars[f"car_{start}_{end}"] = sum_car(daily_ar, start, end)
        else:
            cars[f"car_{start}_{end}"] = None
    
    return EventCAR(...)
```

### 关键步骤详解

#### `_find_event_idx` —— 处理周末/节假日

```python
def _find_event_idx(event_date, return_days, day_to_idx):
    if event_date in day_to_idx:
        return day_to_idx[event_date]
    d = _parse_date(event_date)
    for offset in range(1, 5):
        candidate = (d + timedelta(days=offset)).isoformat()
        if candidate in day_to_idx:
            return day_to_idx[candidate]
    return None
```

**关键细节**：

- 8-K filing 通常在美东时间，工作日发布
- 如果是周末 → 找下一个交易日（最多 +4 天）
- A 股类似

**+4 天的 buffer**：覆盖长周末（如美国 MLK Day / Memorial Day 等）。

#### 估计窗口的 sanity check

```python
if est_start < 0 or est_end < 0:
    return None
if len(stock_est) < _MIN_ESTIMATION_DAYS:
    return None
```

**两道防线**：
1. 事件太早 → estimation window 超出数据范围
2. 数据本身有缺失 → estimation 不够

#### 拟合市场模型

```python
model = fit_market_model(stock_est, spy_est)
```

**调用 `v2/event_study/stats.py::fit_market_model`** —— `stats.py` 里有 OLS 实现。

#### 算异常收益

```python
daily_ar = compute_abnormal_returns(stock_evt, spy_evt, model.alpha, model.beta)
```

**公式**：`AR_t = R_stock_t - (α + β × R_spy_t)`

**含义**：股票**真实收益**减**模型预测收益** = "市场因素无法解释的部分"。

#### 累积 CAR

```python
for start, end in _CAR_WINDOWS:
    if end < n_days:
        cars[f"car_{start}_{end}"] = sum_car(daily_ar, start, end)
    else:
        cars[f"car_{start}_{end}"] = None
```

**3 个 CAR**：(0,+1) / (0,+5) / (0,+20)

**`None` if data 不够**：如果 event 是最近发生的，+20 还没到 → CAR(None)。

## `_aggregate` —— 跨 ticker 聚合（line 298-357）

```python
def _aggregate(events, n_bootstrap, rng_seed):
    if not events:
        return []
    
    groups = defaultdict(list)
    for e in events:
        groups[e.source_type].append(e)
    
    car_attr = {"[0,+1]": "car_0_1", "[0,+5]": "car_0_5", "[0,+20]": "car_0_20"}
    results = []
    
    for source_type in sorted(groups):
        group = groups[source_type]
        windows = []
        
        for window_label, attr in car_attr.items():
            values = [getattr(e, attr) for e in group if getattr(e, attr) is not None]
            if len(values) < 2:
                continue
            
            cars_arr = np.array(values)
            mean = float(cars_arr.mean())
            std = float(cars_arr.std(ddof=1))
            t, p = ttest_cars(cars_arr)
            ci = bootstrap_ci(cars_arr, n_bootstrap=n_bootstrap, rng_seed=rng_seed)
            
            windows.append(WindowStats(...))
        
        results.append(AggregateResult(source_type=source_type, n_events=len(group), windows=windows))
    
    return results
```

### 按 source_type 分组

| source_type | 含义 | 学术预期 |
|------------|------|---------|
| 8-K | 最早公告（earnings 公告） | CAR 最显著 |
| 10-Q | 季报 | CAR 弱一些（filing 滞后） |
| 10-K | 年报 | CAR 更弱 |
| 20-F | 外国公司年报 | 样本少，统计力弱 |

**注释明示**（line 311-313）：

> "Segmenting by source_type is the built-in robustness check: 8-K events have precise announcement dates, so CARs should be sharper. 10-Q/K filing dates lag by 0-45 days, diluting the signal."

### 4 个聚合统计

每个 (source_type, window) 组合算：

| 统计 | 用途 |
|------|------|
| `mean_car` | 平均 CAR |
| `std_car` | 标准差 |
| `t_stat` + `p_value` | t 检验显著性 |
| `ci` | bootstrap 置信区间 |

**`n_events < 2` 跳过**：至少 2 个事件才能算统计量。

## 关键设计决策

### 1. 为什么 SPY 不是 ^GSPC 或其他？

✅ **SPY**：流动性最好的 ETF，data 最干净

❌ **^GSPC**：S&P 500 指数本身——价格有"指数点位"的特性，不完全可比

### 2. 为什么估计窗口这么宽（-250）？

✅ **-250**：约 1 年的 trading days，足以稳定估计 β

❌ **-60**（只用最近 3 个月）：β 估计噪声大

### 3. 为什么用 OLS 市场模型而不是简单 `(0, +1) return`？

✅ **市场模型**：剥离市场 beta → "pure idiosyncratic return"

❌ **简单收益**：会被市场整体涨跌污染（Q1 系统性上涨 → 所有股票 CAR 都虚高）

### 4. 为什么保留 `rng_seed`？

✅ **可复现**：bootstrap CI 的随机数种子固定 → 同一数据同一结果

❌ **不固定**：每次跑 CI 不同，难以对比

## 对 fork → CN 版本的启示

### A. A 股市场模型必须改

| 维度 | 美股 | A 股 |
|------|------|------|
| 市场代理 | SPY | 000300.SH（沪深 300）或 510300.SH |
| 节假日 | 美国节假日 | 春节 / 国庆 / 两会等 |
| 涨跌停 | 无 | ±10% / ±20% |
| 交易时间 | 6.5 小时 | 4 小时（更短） |
| T+1 | 持仓 T+1 | 资金 + 持仓都 T+1 |

### B. CN event study 的差异

```python
# v2/event_study/engine.py 的修改
_MARKET_TICKER = "510300.SH"  # 沪深 300 ETF
```

- A 股节假日更多——`_find_event_idx` 的 +4 buffer 可能不够（春节 7 天 + 调休）
- 涨跌停可能让"事件日"延后（filing 后停牌）
- Tushare 应该有专门的 trading_cal 表——可以用它替代简单 buffer

### C. CN 特有事件

| 事件 | A 股对应 | CAR 预期 |
|------|---------|---------|
| earnings 公告 | 业绩预告 / 快报 / 正式报告 | 类似 PEAD |
| 增发 | 配股 / 定增 | 通常负（稀释） |
| 减持公告 | 大股东减持 | 通常负（信号） |
| 监管处罚 | 证监会立案 / ST | 通常负（弱化估值） |
| 重大合同 | 中标公告 | 通常正（增长信号） |

**v2 的 `compute_car` 可以重用到这些事件**——只要把"earnings history"换成"事件列表"。

## 深读问题（自检）

- [ ] 估计窗口 [-250, -11] 用了 240 天，但 `_MIN_ESTIMATION_DAYS = 200`——这 40 天的余量是给什么用的？
- [ ] `_CAR_WINDOWS = [(0, 1), (0, 5), (0, 20)]` 没包含 (-1, 0) 或 (-5, 0) 的"pre-event"窗口——PEAD 学术论文有时会看"公告前有没有 leak / drift"。为什么 v2 跳过？
- [ ] `compute_car` 串行 for loop 处理 tickers——25 只 ticker 串行慢，但 v2 没并发。并行的风险在哪？
- [ ] 异常收益用 OLS 市场模型——A 股中小盘股的 β 不稳定，怎么办？（提示：Fama-French 三因子 / 五因子模型）

---

*解析时间：2026-07-12 · 第七轮迭代*
*下次解析目标：`v2/event_study/{models,stats,plot}.py` + `v2/demo/backtest.py` + `v2/backtesting/{__main__,__init__}.py`*