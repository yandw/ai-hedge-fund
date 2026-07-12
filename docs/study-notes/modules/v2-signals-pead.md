# v2/signals/pead.py 深度解析

> 文件：`v2/signals/pead.py` · 139 行 · 1 个 QuantModel 子类 + 2 个常量
> 地位：**v2 quant model 的样板工程**——PEAD（Post-Earnings Announcement Drift）是 quant 与 LLM 共享同一接口的最好示例

## 文件作用一句话

**实现 Post-Earnings Announcement Drift 策略：财报 BEAT 后看多 / MISS 后看空，只在 SEC 8-K filing 后短窗口内触发，严格 PIT。**

## PEAD 是什么（5 行简介）

**学术结论**（Ball & Brown 1968 + Bernard & Thomas 1989）：市场对 earnings surprise **反应不足**——BEAT 的股票后续几天到几周继续上涨，MISS 继续下跌。

**v2 实现**：每次 SEC 公布 8-K / 10-Q（filing_date）后 4 天内：

- BEAT → 多（`Signal.value = +1.0`）
- MISS → 空（`Signal.value = -1.0`）
- 其他 → 中性（`Signal.value = 0.0`）

## 2 个关键常量

```python
_RETROSPECTIVE_CUTOFF_DAYS = 45   # 过滤"远期回填"的数据
_SOURCE_PRIORITY = {"8-K": 0, "10-Q": 1, "10-K": 2, "20-F": 3}  # 同 report_period 选最早期
```

### `_RETROSPECTIVE_CUTOFF_DAYS = 45`（line 21）

**含义**：filing_date 与 report_period 的差 ≤ 45 天才纳入 PEAD 信号。

**为什么**：

- 8-K 通常在 quarter-end 后 2-4 周发布（filing_date - report_period ≈ 14-30 天）
- 10-Q / 10-K 在 quarter-end 后 4-8 周发布（30-60 天）
- **如果 filing_date 离 report_period > 45 天**：可能是 10-Q 包含了"上季度对比数据"，看起来像 Q4 event 实际是 Q1 数据的 retrospective 锚定——**必须剔除**

**注释里的真实例子**（在 `v2/event_study/engine.py` 里提到）：

> GS: report_period=2025-12-31, filing_date=2026-04-13 → 103 天 → 剔除

### `_SOURCE_PRIORITY`（line 22）

```python
{"8-K": 0, "10-Q": 1, "10-K": 2, "20-F": 3}
```

**含义**：同一个 report_period 可能有多个 filing（8-K 最早 + 10-Q 后续）。

**PEAD 用 8-K（最早那份）**：因为 PEAD 假设"市场第一次知道 surprise 就开始 drift"——8-K 是首次公告。

## `predict` 主函数（line 50-84）

```python
def predict(self, ticker, date, data_client):
    as_of = _parse_date(date)
    events = self._qualifying_events(ticker, data_client)
    
    # 1. PIT: 只看 <= date 的事件
    past = [e for e in events if _parse_date(e["filing_date"]) <= as_of]
    if not past:
        return self._neutral(ticker, date)
    
    # 2. 最近一次
    event = max(past, key=lambda e: e["filing_date"])
    filed = _parse_date(event["filing_date"])
    
    # 3. 必须在 signal window 内（默认 4 天）
    if (as_of - filed).days > self._signal_window_days:
        return self._neutral(ticker, date)
    
    # 4. BEAT → +1, MISS → -1
    surprise = event["surprise"]
    value = 1.0 if surprise == "BEAT" else -1.0
    return Signal(
        model_name=self.name,
        ticker=ticker,
        date=date,
        value=value,
        reasoning=(
            f"{surprise} on {event['report_period']} earnings "
            f"(filed {event['filing_date']}, {event['source_type']})"
        ),
        metadata={
            "eps_surprise": surprise,
            "source_type": event["source_type"],
            "report_period": event["report_period"],
            "filing_date": event["filing_date"],
        },
    )
```

### 4 个核心步骤

#### Step 1: PIT 过滤（line 55）

```python
past = [e for e in events if _parse_date(e["filing_date"]) <= as_of]
```

**关键**：`filing_date` 而非 `report_period`——避免 lookahead。

#### Step 2: 找最近一次事件（line 60）

```python
event = max(past, key=lambda e: e["filing_date"])
```

PEAD 是**事件驱动**——只看最近的 BEAT/MISS 事件，不是历史均值。

#### Step 3: 4 天窗口（line 64）

```python
if (as_of - filed).days > self._signal_window_days:
    return self._neutral(ticker, date)
```

**关键设计**：

- 8-K filing 后第 1 天：signal 触发
- 第 2、3、4 天：仍然有效
- 第 5 天及之后：signal 过期

**为什么不更宽（10 天、20 天）？**

- 学术研究显示 drift 主要在 1-5 天内
- 5 天后 drift 减弱到接近 0
- 4 天是经验最优（避免过晚触发，也避免触发太频繁）

#### Step 4: 返回 Signal（line 67-84）

```python
value = 1.0 if surprise == "BEAT" else -1.0
```

**注意**：直接 ±1.0，**没有 confidence**——PEAD 的 conviction 是 100% 固定。

**文档说明**（line 30-31）：

> "Conviction magnitude is fixed ±1 for v0 — scaling by surprise size is a future enhancement."

## `_qualifying_events` 数据清洗（line 93-134）

```python
def _qualifying_events(self, ticker, data_client):
    """Return BEAT/MISS events for a ticker, deduped + retrospective-filtered."""
    
    # 1. 内存 cache（每个 ticker 只 fetch 一次）
    if ticker in self._cache:
        records = self._cache[ticker]
    else:
        records = data_client.get_earnings_history(ticker, limit=self._earnings_limit)
        self._cache[ticker] = records
    
    best: dict[str, tuple[int, EarningsRecord]] = {}
    for r in records:
        # 2. 必须有 filing_date + quarterly + surprise 标签
        if not r.filing_date or not r.quarterly:
            continue
        surprise = r.quarterly.eps_surprise
        if surprise not in ("BEAT", "MISS"):
            continue
        
        # 3. 45 天 retrospective filter
        lag = (_parse_date(r.filing_date) - _parse_date(r.report_period)).days
        if lag >= _RETROSPECTIVE_CUTOFF_DAYS:
            continue
        
        # 4. 同 report_period 选最早期
        priority = _SOURCE_PRIORITY.get(r.source_type, 99)
        if r.report_period not in best or priority < best[r.report_period][0]:
            best[r.report_period] = (priority, r)
    
    # 5. 转成 dict 列表
    return [
        {
            "filing_date": r.filing_date,
            "report_period": r.report_period,
            "source_type": r.source_type,
            "surprise": r.quarterly.eps_surprise,
        }
        for _, r in best.values()
    ]
```

### 5 个清洗步骤详解

#### Step 1: **实例级 cache**

```python
self._cache: dict[str, list[EarningsRecord]] = {}
```

**原因**（注释 line 42-44）：

> "Cache earnings history per ticker — predict is called once per trading day during a backtest, so we fetch each ticker only once."

**现实**：一个 backtest 跑 250 个交易日 × 25 只 ticker = 6250 次 `predict()` 调用——但 earnings_history **对每只 ticker 是不变的**，可以**实例级**cache。

#### Step 2: 字段完整性

```python
if not r.filing_date or not r.quarterly:
    continue
surprise = r.quarterly.eps_surprise
if surprise not in ("BEAT", "MISS"):
    continue
```

- 没 `filing_date` → 不知道何时公开 → 跳过
- 没 `quarterly.eps_surprise` 标签 → 没信号 → 跳过
- **只接 BEAT/MISS，不接 MEET**（中性，过滤掉）

#### Step 3: 45 天 retrospective filter

```python
lag = (_parse_date(r.filing_date) - _parse_date(r.report_period)).days
if lag >= _RETROSPECTIVE_CUTOFF_DAYS:
    continue
```

**这是从真实 bug 学来的**（注释 line 99-101）：

> "The extractor sometimes parses prior-quarter comparison data from a current 8-K, producing rows that look like a real Q4 event but are actually anchored on a Q1 filing date."

#### Step 4: 同 report_period 去重，选最早期

```python
priority = _SOURCE_PRIORITY.get(r.source_type, 99)
if r.report_period not in best or priority < best[r.report_period][0]:
    best[r.report_period] = (priority, r)
```

**举例**：AAPL Q3 2024 有两份 filing：

- 8-K（filed 2024-08-01，priority=0）
- 10-Q（filed 2024-08-08，priority=1）

→ 选 8-K（priority=0 较小）。

**逻辑关键**：`priority < best[...][0]` 而非 `<=`——**8-K 替换 10-Q 用严格小于**，第一次填进去时（`not in best`）也算"小于"。

#### Step 5: 转成 dict 返回

**为什么不用 Pydantic model**？

返回 dict 是为了**只暴露 PEAD 需要的字段**（4 个），隐藏其他 metadata。

## 设计取舍

### 1. ±1.0 固定 conviction vs 按 surprise 大小

✅ **±1.0 固定**：

- 简单
- 4 天窗口本来就只触发几次
- 不需要 calibration

❌ **按 surprise 大小**（如 EPS surprise 5% → +0.5，10% → +1.0）：

- 更"信息丰富"
- 但需要历史数据校准

**判断**：合理。文档明示 "future enhancement"——YAGNI 现在不做。

### 2. 4 天窗口 vs 20 天窗口

✅ **4 天**：

- 学术研究支持 1-5 天
- 减少误触发（避免一次财报被多次进入）

❌ **20 天**：

- 太长，会重叠多次 BEAT/MISS

**判断**：合理。4 是经验最优。

### 3. 实例级 cache vs 每次 fetch

✅ **实例级 cache**：

```python
self._cache: dict[str, list[EarningsRecord]] = {}
```

**关键观察**：backtest 里 predict() 调用频繁但底层数据不变。

**优势**：
- 250 个交易日 × 25 ticker = 6250 次调用，但 earnings history 只 fetch 25 次
- 快 250 倍

**风险**：
- **多 ticker PEAD 共享一个 instance** 时，cache 是共享的——OK
- **同一 PEAD instance 跑多个 backtest** 时，cache 可能脏——建议每次新 backtest 用新 instance

### 4. `signal_window_days` 参数化

```python
def __init__(self, *, earnings_limit: int = 8, signal_window_days: int = 4):
```

✅ **参数化**——可以实验不同窗口找最优。

## 对 fork → CN 版本的启示

### A. CN PEAD 必须重写——美股 PEAD 不能直接用

| 差异 | 美股 | A 股 | 处理 |
|------|------|------|------|
| **Filing 类型** | 8-K/10-Q/10-K/20-F | 业绩预告/快报/正式报告 | `_SOURCE_PRIORITY` 要重写 |
| **披露日** | SEC filing_date | cninfo 公告日 | 映射 |
| **BEAT/MISS 标签** | API 直接返回 | **Tushare 无原生标签** | 需自己算 `(actual-forecast)/|forecast|` |
| **T+1 制度** | T+0 现金 / T+1 持仓 | T+1 资金和持仓 | 信号触发次日才能买，window 调短 |
| **涨跌停** | 无 | ±10% / ±20% | 触发日涨停 → 无法买入，signal 浪费 |
| **业绩预告准确性** | 高 | 预告误差大 | surprise 阈值可能要更严格 |

### B. CN PEAD 的实现策略

```python
class CN_PEADModel(QuantModel):
    """A 股 PEAD：业绩超预期后的 drift。
    
    关键差异：
    - signal_window_days = 2（A 股 drift 比美股短）
    - 涨跌停过滤：触发日涨停则次日再看
    - 业绩预告标签自己计算（Tushare 没现成的）
    """
    
    @property
    def name(self):
        return "cn_pead"
    
    def predict(self, ticker, date, data_client):
        # 1. 拿业绩预告 + 实际数据
        # 2. 算 surprise = (actual - forecast) / |forecast|
        # 3. if surprise > 5%: BEAT; if surprise < -5%: MISS
        # 4. signal_window_days = 2
        # 5. 涨跌停检查
        ...
```

### C. 复用 v2.pead 的清洗逻辑

`_qualIFYING_EVENTS` 的"45 天 retrospective filter" + "同 report_period 选最早期"思想可以**直接复用**——只要把 `_SOURCE_PRIORITY` 改成 A 股 filing 类型。

## 深读问题（自检）

- [ ] `signal_window_days=4` 在 A 股是否合理？T+1 制度 + 涨跌停会让 drift 窗口变短还是变长？
- [ ] PEAD 的 cache 写在 `self._cache` 而不是用 `functools.lru_cache`——为什么？如果一个 backtest 跑完还要继续用同一个 model 跑 paper trading，cache 会"脏"吗？
- [ ] `_SOURCE_PRIORITY` 优先级数字越小越优先——如果以后 Tushare 加了一个 "X 公告" 类型（数字不在 dict 里），会发生什么？（提示：看 `priority = _SOURCE_PRIORITY.get(r.source_type, 99)`）
- [ ] PEAD 的 ±1.0 conviction + 4 天窗口 + 等金额仓位——这三个简化对 Sharpe 的影响是累加的还是相乘的？

---

*解析时间：2026-07-12 · 第六轮迭代*
*下次解析目标：`v2/signals/llm_agent.py` + `v2/signals/buffett.py`*