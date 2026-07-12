# v2/signals/base.py 深度解析

> 文件：`v2/signals/base.py` · 106 行 · 2 个 ABC + 5 个 helper
> 地位：**v2 alpha model 的"宪法"——所有 quant/LLM 共用的接口 + 共享数值工具**

## 文件作用一句话

**定义 `AlphaModel` 抽象基类，规定"alpha model 只形成观点"，并给 quant 子类提供数值工具（sigmoid / RSI / percentile 等）。**

## 核心哲学（来自 docstring line 9-16）

```python
"""Alpha models — the components that form views on what to hold.

An *alpha model* (Rishi Narang's term, *Inside the Black Box*) is anything
that produces a forecast / view on an asset. It's the "edge" component of
a quant fund. In v2, both quant signals (PEAD, regime) and LLM investor
agents (Buffett, Druckenmiller) are alpha models — they all implement this
interface and produce a `Signal` (a conviction in [-1, +1] + reasoning).

The alpha model only forms a *view*. It does NOT decide position mechanics
(timing, sizing, holding period) — that's the job of portfolio construction
and execution. This separation (views vs positions) is deliberate.
"""
```

**2 条不妥协的设计原则**：

1. **Alpha = view only** —— 不是交易决定
2. **Views ≠ positions** —— 时机、规模、持仓期由 portfolio / execution 决定

**术语来源**：Rishi Narang 的《Inside the Black Box》——业内对"alpha"的标准定义（任何能产生预测/观点的组件）。

## `AlphaModel` ABC（line 29-51）

```python
class AlphaModel(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...
    
    @abstractmethod
    def predict(
        self,
        ticker: str,
        date: str,
        data_client: DataClient,
    ) -> Signal: ...
```

**接口只有 2 个成员**：

| 成员 | 类型 | 必填 |
|------|------|------|
| `name` | property / str | ✅ |
| `predict(ticker, date, data_client)` | method | ✅ |

**`predict` 的关键约束**（来自 docstring line 46-50）：

```
MUST be point-in-time: only use data with date <= *date* (no
lookahead). Return a Signal with conviction in [-1, +1] — use
0.0 to express "no view" (abstain).
```

**3 条铁律**：

1. **PIT** —— 只能用 date 当天及之前的数据
2. **value ∈ [-1, +1]** —— conviction 必须在这个区间
3. **0.0 = abstain** —— 没观点时返 0.0，不要瞎猜

## `QuantModel` 子类（line 54-105）

```python
class QuantModel(AlphaModel):
    """Base for pure-math alpha models (no LLM)."""
    
    # 5 个 helper
    _safe_float(value, default=0.0) -> float
    _percentile_rank(value, values) -> float   # 0-100
    _normalize_to_signal(raw, low=-1.0, high=1.0) -> float
    _sigmoid(x, scale=5.0) -> float           # (-1, +1)
    _compute_rsi(prices, period=14) -> float   # 0-100
```

### 5 个 helper 详解

#### `_safe_float`（line 65-74）

```python
@staticmethod
def _safe_float(value, default=0.0):
    if value is None:
        return default
    try:
        f = float(value)
        return default if (np.isnan(f) or np.isinf(f)) else f
    except (ValueError, TypeError):
        return default
```

**作用**：把任意值转 float，None/NaN/Inf/异常 → default。

**调用场景**：Financial Datasets 60+ 字段里 90% 是 nullable，quant 模型要对每个字段算之前先 sanitize。

#### `_percentile_rank`（line 77-82）

```python
@staticmethod
def _percentile_rank(value, values):
    if not values:
        return 50.0
    below = sum(1 for v in values if v < value)
    return (below / len(values)) * 100.0
```

**作用**：返回 `value` 在 `values` 中的百分位排名。

**注意**：不返回 0-100 的标准百分位——而是"严格小于的比例"，所以**结果范围 [0, 100)**（不包括 100）。

#### `_normalize_to_signal`（line 85-87）

```python
@staticmethod
def _normalize_to_signal(raw, low=-1.0, high=1.0):
    return max(low, min(high, raw))
```

**作用**：clamp 到 [-1, +1]。

**用法**：`alpha_model.predict()` 返回前最后一步，确保 value 不越界。

#### `_sigmoid`（line 89-92）

```python
@staticmethod
def _sigmoid(x, scale=5.0):
    return float(np.tanh(x * scale))
```

**作用**：把无界值映射到 (-1, +1)。

**为什么是 `tanh` 不是 logistic sigmoid**：

- tanh 输出 ∈ (-1, +1) —— 直接对齐 Signal.value 约束
- logistic sigmoid 输出 ∈ (0, 1) —— 需要再 `(s-0.5)*2` 才能用

**scale=5.0 的意义**：控制"灵敏度"。scale 越大，输出越接近 ±1；越小，越接近 0。

#### `_compute_rsi`（line 95-105）

```python
@staticmethod
def _compute_rsi(prices, period=14):
    delta = prices.diff()
    gain = delta.where(delta > 0, 0.0).rolling(window=period).mean()
    loss = (-delta.where(delta < 0, 0.0)).rolling(window=period).mean()
    rs = gain / loss
    rsi = 100.0 - (100.0 / (1 + rs))
    latest = rsi.iloc[-1]
    if pd.isna(latest):
        return 50.0
    return float(latest)
```

**作用**：算最后一天的 RSI（14-day 默认）。

**注意**：返回 50.0 而不是 None 当数据不够——这避免了上层要处理 None。

## 5 个 helper 的实际使用情况

| Helper | 在 PEAD 里用了？ | 在 Buffett 里用了？ | 实际用 |
|--------|----------------|-------------------|--------|
| `_safe_float` | ✅ (line 间接) | ❌ | **1/5** |
| `_percentile_rank` | ❌ | ❌ | 0/5 |
| `_normalize_to_signal` | ❌ | ❌ | 0/5 |
| `_sigmoid` | ❌ | ❌ | 0/5 |
| `_compute_rsi` | ❌ | ❌ | 0/5 |

**事实**：

- **只有 `_safe_float` 被实际使用**（`pead.py` 通过 `_qualifying_events` 间接依赖）
- 其他 4 个 helper 是**为未来 quant model 准备的脚手架**
- 反映 **YAGNI 原则的轻微违反**——预先写"总有一天用得上"的工具

**维护者取舍**：

✅ **优点**：未来加 `MomentumModel`、`MeanReversionModel` 时不用从零写
❌ **缺点**：增加文件长度、未被测试覆盖、未来 API 可能变了

## 设计取舍

### 1. 为什么不直接用 numpy / pandas 内置？

| Helper | 为什么自己写 |
|--------|------------|
| `_safe_float` | numpy 不直接处理 None + NaN + Inf 三种情况 |
| `_percentile_rank` | numpy 有 `percentile` 但语义不同（插值 vs 严格小于） |
| `_normalize_to_signal` | `np.clip` 也行，但一行 max/min 更直观 |
| `_sigmoid` | 直接 `np.tanh` 就行，wrapper 的价值是统一接口 |
| `_compute_rsi` | pandas-ta / talib 都有，但引入新依赖 |

**判断**：5 个 helper 里 2 个确实有价值（`_safe_float`、`_compute_rsi`），其他 3 个是过度工程。

### 2. `name` 为什么是 property 而不是字段？

```python
@property
@abstractmethod
def name(self) -> str:
    """Model identifier (e.g. 'pead', 'buffett')."""
```

✅ **property + abstractmethod** 的组合让子类**必须**实现 `name`，但可以是 property 或 field。

```python
class PEADModel(QuantModel):
    @property
    def name(self):
        return "pead"
```

**好处**：
- 调用方 `model.name` 统一（property vs field 在 Python 是透明的）
- 子类可以选择动态计算 `name`

### 3. ABC vs Protocol

✅ **ABC**（用了 `ABC` + `@abstractmethod`）：

- 实例化期强制子类实现接口（防止漏写 `name`）
- 适合**必须有继承**的接口

❌ **如果用 Protocol**：子类可以 duck-type，但漏写 `name` 不会立刻报错

**v2 signals vs v2 data** 的对比：

| 模块 | 用 Protocol 还是 ABC | 原因 |
|------|---------------------|------|
| `v2/data/protocol.py` | Protocol | 第三方 client 不想继承 |
| `v2/signals/base.py` | ABC | alpha model 在仓库内，必有继承 |

**判断**：合理。**社区贡献者写 DataClient、内部开发者写 AlphaModel**——不同治理策略。

## 对 fork → CN 版本的启示

### A. 完全不动这个文件

`AlphaModel` 接口是 v2 内核，CN 化不需改。**新加的 `MomentumModel`、`CN_PEADModel` 都继承 `QuantModel`**。

### B. CN 量化模型模板

```python
class CN_PEADModel(QuantModel):
    """中国版 PEAD：A 股的 BEAT/MISS 信号。
    
    与美股 PEAD 的差异：
    - T+1 制度 → signal_window_days 应缩短（次日就开始 drift）
    - 涨跌停 → 不能交易日的信号要跳过
    - 中文财报披露习惯 → 4-week retrospective cutoff 可能要调
    """
    
    @property
    def name(self):
        return "cn_pead"
    
    def predict(self, ticker, date, data_client):
        # 复用 v2.pead 的逻辑，但 Tushare 的 source_type 是 "季报"/"中报"/"年报" 不是 "10-Q"
        ...
```

### C. 5 个 helper 在 CN 都有用

| Helper | CN 用法 |
|--------|--------|
| `_safe_float` | Tushare 返回 NaN 比 FD 还多，必用 |
| `_percentile_rank` | A 股估值百分位排名（PE 分位） |
| `_normalize_to_signal` | 所有 quant model 最后一步 |
| `_sigmoid` | 涨跌停硬约束外的"软"信心表达 |
| `_compute_rsi` | A 股技术派必备 |

**结论**：CN 化时这些 helper **比在美股环境更有用**——可以保留。

## 深读问题（自检）

- [ ] `ABC` + `@abstractmethod` vs `Protocol` 的边界在哪？v2 两个核心文件一个用 ABC、一个用 Protocol，是有意分工还是历史偶然？
- [ ] `_compute_rsi` 用 `pd.Series.diff()` + `rolling()` 而不是 `talib.RSI`——如果上 prod，Tushare 数据可能有缺失日期，diff() 的 NaN 会传播多长？
- [ ] `_percentile_rank` 严格小于 vs 百分位插值——这两种语义对 quant 模型决策影响大吗？
- [ ] 5 个 helper 里 4 个没人用——如果让你删 4 个留 1 个，你留哪个？为什么？

---

*解析时间：2026-07-12 · 第六轮迭代*
*下次解析目标：`v2/signals/pead.py` + `v2/signals/llm_agent.py` + `v2/signals/buffett.py`*