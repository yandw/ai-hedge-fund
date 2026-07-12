# v2/event_study/stats.py + models.py 深度解析

> 文件：`v2/event_study/stats.py`（120 行）+ `v2/event_study/models.py`（104 行）
> 地位：**事件研究的"数学库"** + **数据结构**——`stats.py` 跑数学（OLS / t-test / bootstrap），`models.py` 定义返回类型

## 文件作用一句话

**`stats.py` 提供纯函数（无副作用）：OLS 市场模型拟合、异常收益计算、CAR 求和、t 检验、bootstrap 置信区间。`models.py` 定义事件研究的 5 层数据结构。**

## 两文件协作关系

```
v2/event_study/engine.py    ← 主流程
        ↓ 调用
v2/event_study/stats.py      ← 数学
        ↓ 操作
v2/event_study/models.py     ← 数据类型
```

## stats.py 5 个函数

### `fit_market_model` —— OLS 市场模型（line 16-44）

```python
def fit_market_model(stock_returns, market_returns):
    n = len(stock_returns)
    X = np.column_stack([np.ones(n), market_returns])
    coeffs, _, _, _ = np.linalg.lstsq(X, stock_returns, rcond=None)
    alpha, beta = coeffs[0], coeffs[1]
    
    predicted = alpha + beta * market_returns
    ss_res = np.sum((stock_returns - predicted) ** 2)
    ss_tot = np.sum((stock_returns - stock_returns.mean()) ** 2)
    r_squared = 1.0 - ss_res / ss_tot if ss_tot > 0 else 0.0
    
    return MarketModelFit(alpha=alpha, beta=beta, r_squared=r_squared, n_obs=n)
```

**OLS 拟合**：`R_stock = α + β × R_market`

**4 个返回字段**：

| 字段 | 含义 | 用途 |
|------|------|------|
| `alpha` | 截距——market return = 0 时的股票日均收益 | 衡量股票"基线超额收益" |
| `beta` | 斜率——市场涨 1%，股票涨多少 | 衡量股票"市场敏感度" |
| `r_squared` | R²——股票方差被市场解释的比例 | 衡量模型好坏（高 = 市场因素主导） |
| `n_obs` | 估计用的天数 | 估计稳定性 |

**关键细节**：

- 用 `np.linalg.lstsq` 而不是 `scipy.stats.linregress`——**lstsq 不需要 scipy.stats**，更通用
- `rcond=None` —— 用机器精度做 QR 分解的 rank 阈值
- `ss_tot > 0` 防御 —— 全 0 收益时不除 0

### `compute_abnormal_returns` —— AR 计算（line 47-60）

```python
def compute_abnormal_returns(stock_returns, market_returns, alpha, beta):
    return stock_returns - (alpha + beta * market_returns)
```

**最简单也最核心的函数**——一行数学。

**公式**：`AR_t = R_stock,t - (α + β × R_market,t)`

**含义**：

- "实际收益"减"市场模型预测的收益"
- 剩下的 = "市场因素无法解释的部分" = 异常收益

### `sum_car` —— CAR 求和（line 63-69）

```python
def sum_car(daily_ar, start, end):
    return float(np.sum(daily_ar[start : end + 1]))
```

**简单切片求和**：

- 窗口 [0, +1]：包含 day 0 和 day 1（共 2 天）
- 窗口 [0, +5]：包含 day 0 到 day 5（共 6 天）
- 窗口 [0, +20]：包含 day 0 到 day 20（共 21 天）

**`[start:end+1]` 是 Python slice 习惯**：包含 end。

### `ttest_cars` —— 单样本 t 检验（line 72-84）

```python
def ttest_cars(cars):
    if len(cars) < 2:
        return 0.0, 1.0
    t_stat, p_value = sp_stats.ttest_1samp(cars, popmean=0.0)
    return float(t_stat), float(p_value)
```

**H0**：`mean CAR = 0`（事件对价格无影响）

**H1**：`mean CAR ≠ 0`（事件对价格有影响）

**`len(cars) < 2` 防御**：1 个样本不能做 t 检验。

**输出**：

| 字段 | 含义 |
|------|------|
| `t_stat` | t 统计量 |
| `p_value` | 双侧 p-value |

**p-value < 0.05** → 拒绝 H0 → 事件显著影响价格。

### `bootstrap_ci` —— 非参数置信区间（line 87-119）

```python
def bootstrap_ci(cars, n_bootstrap=10_000, confidence=0.95, rng_seed=None):
    rng = np.random.default_rng(rng_seed)
    n = len(cars)
    boot_means = np.array([
        rng.choice(cars, size=n, replace=True).mean() for _ in range(n_bootstrap)
    ])
    lower_pct = (1 - confidence) / 2 * 100   # 2.5
    upper_pct = (1 + confidence) / 2 * 100   # 97.5
    lower, upper = np.percentile(boot_means, [lower_pct, upper_pct])
    return BootstrapCI(lower=float(lower), upper=float(upper), confidence=confidence, n_bootstrap=n_bootstrap)
```

**Percentile Bootstrap 算法**：

1. 从 `cars` 里**有放回**采样 n 个
2. 算这次采样的 mean
3. 重复 10,000 次
4. 取 2.5% 和 97.5% 分位数 → 95% CI

**为什么用 bootstrap 而不是 t 检验**：

- t 检验假设 CAR 分布**正态**
- Bootstrap **不假设分布**——更鲁棒
- 样本量小时更可靠

**CI 不包含 0 → 显著**。

**`rng_seed` 价值**：固定种子 → 同样数据同样 CI → 可复现。

## models.py 5 个数据模型

| 模型 | 含义 | 在 pipeline 中的位置 |
|------|------|---------------------|
| `MarketModelFit` | α/β/R²/n_obs | 单个事件 → 单次 OLS |
| `EventCAR` | 单个事件的 CAR | 单 ticker × 单 filing |
| `BootstrapCI` | bootstrap CI | WindowStats 的子字段 |
| `WindowStats` | 单窗口统计 | 单 source_type × 单窗口 |
| `AggregateResult` | 单 source_type 全窗口 | 跨事件聚合 |
| `EventStudyResult` | 顶层结果 | engine.compute_car 返回 |

### `MarketModelFit`（line 17-27）

```python
class MarketModelFit(BaseModel):
    alpha: float           # 截距
    beta: float            # 斜率
    r_squared: float       # R²
    n_obs: int             # 估计用天数
```

**简洁 4 字段**——OLS 的标准输出。

### `EventCAR`（line 31-48）

```python
class EventCAR(BaseModel):
    ticker: str
    event_date: str                           # 用 filing_date 当 anchor
    source_type: str                          # 8-K / 10-Q / 10-K / 20-F
    report_period: str                        # fiscal quarter end
    eps_surprise: str | None = None           # BEAT / MISS / MEET
    market_model: MarketModelFit              # 该事件用的 α/β
    daily_ar: list[float]                     # [0, +20] 每天的 AR
    car_0_1: float | None                     # 2 天 CAR
    car_0_5: float | None                     # 6 天 CAR
    car_0_20: float | None                    # 21 天 CAR
```

**3 个 CAR 字段是 Optional**——如果事件太近、+20 天还没到 → `None`。

### `BootstrapCI`（line 51-61）

```python
class BootstrapCI(BaseModel):
    lower: float
    upper: float
    confidence: float = 0.95
    n_bootstrap: int = 10_000
```

**保留 n_bootstrap**——审计时知道"CI 是基于多少 resample 算出来的"。

### `WindowStats`（line 64-77）

```python
class WindowStats(BaseModel):
    window: str                               # "[0,+1]"
    n_events: int
    mean_car: float
    std_car: float
    t_stat: float
    p_value: float
    ci: BootstrapCI
```

**关键的 7 字段组合**——**一组事件的统计全在这一个对象里**。

### `AggregateResult`（line 80-90）

```python
class AggregateResult(BaseModel):
    source_type: str
    n_events: int
    windows: list[WindowStats]
```

**一个 source_type 一条结果**——多个窗口合并到一个 AggregateResult 里。

### `EventStudyResult`（line 93-103）

```python
class EventStudyResult(BaseModel):
    events: list[EventCAR] = Field(default_factory=list)
    aggregates: list[AggregateResult] = Field(default_factory=list)
    skipped_tickers: list[str] = Field(default_factory=list)
```

**3 个字段**：

- `events` —— 所有单个事件的 CAR 详情
- `aggregates` —— 按 source_type 分组的聚合统计
- `skipped_tickers` —— 因数据不够跳过的 ticker

## 设计取舍

### 1. 为什么自己写 stats 而不是用 statsmodels？

✅ **自己写 OLS**：

- 只用 numpy，依赖少
- 控制更多（不暴露 statsmodels 的复杂 API）

❌ **用 statsmodels**：

- 功能更全（HAC 稳健标准误等）
- 但引入大依赖

**判断**：合理。**当前需求简单，自己写够用**。

### 2. 为什么 bootstrap + t-test 两个都算？

✅ **两个都给**：

- t 检验是经典（论文常见）
- bootstrap 更鲁棒
- 不同读者偏好不同

❌ **只算一个**：简化但牺牲灵活性

**判断**：合理。**审计读者会 cross-validate**。

### 3. `daily_ar: list[float]` 而不是 `np.ndarray`？

✅ **`list[float]`**：

- Pydantic 友好
- JSON 可序列化

❌ **`np.ndarray`**：Pydantic v2 也支持但序列化麻烦

**判断**：合理。

### 4. `source_type` 用 str 不是 Literal？

✅ **`str`**：

- 接受未来加新类型（如 "X-公告"）
- 不破坏现有 JSON

**判断**：和 `v2/data/models.py::EarningsRecord.source_type` 一致。

## 对 fork → CN 版本的启示

### A. A 股事件研究需要的市场代理

```python
_MARKET_TICKER = "510300.SH"  # 沪深 300 ETF
# 或
_MARKET_TICKER = "000300.SH"  # 沪深 300 指数
```

### B. 估计窗口可能要短

**美股 [-250, -11]** 对 A 股可能太宽：

- A 股 IPO 频繁，新股票历史短
- A 股 β 不稳定（政策市），长窗口可能反而拖累

**调整建议**：

```python
_ESTIMATION_START = -180  # A 股可能用 180 更合适
```

### C. Bootstrap 在 A 股样本小

**A 股单 source_type 的事件数可能远少于美股**（如 8-K = A 股"业绩预告"）：

- Bootstrap 1000 次可能就够
- `n_bootstrap` 参数化已支持

## 深读问题（自检）

- [ ] `fit_market_model` 用 `np.linalg.lstsq` 而不是 `scipy.stats.linregress`——后者会顺便给 p-value 和 std_err，为什么不用？
- [ ] `bootstrap_ci` 的 `lower_pct = (1 - confidence) / 2 * 100`——95% CI 是 2.5 和 97.5，对吗？如果 confidence=0.99 呢？
- [ ] `MarketModelFit.r_squared = 0.0 if ss_tot == 0` —— 在什么场景下 ss_tot 会是 0？（提示：股票长期停牌后恢复交易）
- [ ] `daily_ar` 是 21 个 float（day 0 到 day 20），如果事件太近只算到 day 10，剩下 11 个怎么填？

---

*解析时间：2026-07-12 · 第八轮迭代*
*下次解析目标：`v2/event_study/{plot,__main__}.py` + `v2/demo/backtest.py` + `v2/backtesting/__main__.py` + `v2/backtesting/__init__.py` + 几个 __init__/conftest/analyze*