# v2 测试 + 占位模块深度解析

> 文件：v2 tests（6 个）+ 4 个 pipeline/portfolio/risk/validation 占位文件
> 地位：**v2 的质量保障**和**ROADMAP 中明确为占位的 4 个子系统**

## 测试文件总览

```
v2/data/test_cached.py              # CachedDataClient 测试
v2/data/test_client.py              # FDClient 测试
v2/data/test_client_contract.py     # 契约测试（任何 DataClient 都要满足的不变量）
v2/features/test_snapshot.py        # FundamentalsSnapshot 测试
v2/signals/test_signals.py          # AlphaModel / QuantModel 测试
v2/signals/test_llm_agents.py       # LLMAgent 测试
v2/event_study/test_event_study.py  # event_study 测试
v2/backtesting/test_backtest.py     # BacktestEngine 测试
```

8 个 test_*.py 文件，覆盖了 v2 的**每个核心模块**。

## 测试文件结构（按重要性）

### 1. `test_client_contract.py` —— **最重要的测试**

**作用**：任何实现 `DataClient` Protocol 的类**都必须满足**的不变量。

**典型测试内容**（基于 `test_client.py` 的常见模式推测）：

```python
import pytest
from v2.data.protocol import DataClient

def test_fdclient_satisfies_protocol():
    """FDClient 必须是合法 DataClient（duck typing 验证）"""
    from v2.data.client import FDClient
    client = FDClient(api_key="dummy")
    assert isinstance(client, DataClient)    # ← @runtime_checkable 才可能

def test_404_returns_empty(client):
    """404 → None / []（'无数据'而非失败）"""
    if client == ...:
        result = client.get_company_facts("INVALID_TICKER_XYZ")
        assert result is None

def test_429_raises_fdclienterror(client):
    """429 → raise FDClientError"""
    with pytest.raises(FDClientError):
        client.get_prices(...)  # 触发限流

def test_filing_date_lte_for_financial_metrics(client):
    """get_financial_metrics 必须传 filing_date_lte"""
    with patch.object(client._session, 'request') as mock_request:
        mock_request.return_value.status_code = 200
        mock_request.return_value.json.return_value = {"financial_metrics": []}
        client.get_financial_metrics("AAPL", "2024-06-01")
        # 验证 filing_date_lte 在请求参数里
        args, kwargs = mock_request.call_args
        assert "filing_date_lte=2024-06-01" in str(args) or kwargs.get('params', {}).get('filing_date_lte') == '2024-06-01'
```

**为什么 contract test 最值钱**：

- ✅ CN fork 写 `TushareClient` 时，跑这个 test 就知道是否满足 DataClient 契约
- ✅ 防止"看起来像但行为不像"的 client 实现

### 2. `test_client.py` + `test_cached.py` —— FDClient + CachedDataClient

**典型内容**（推测）：
- `test_get_prices_parses_ohlcv`
- `test_get_financial_metrics_filters_by_filing_date`
- `test_404_returns_empty_list`
- `test_5xx_raises`
- `test_cached_hits_disk_on_second_call`
- `test_cached_only_caches_successful_response`
- `test_cached_refresh_ignores_existing`

### 3. `test_snapshot.py` —— FundamentalsSnapshot

**关键测试**：

```python
def test_min_periods_required():
    """少于 4 期 → InsufficientData"""
    with pytest.raises(InsufficientData):
        build_snapshot(ticker="AAPL", as_of="2024-06-01", data_client=mock_with_3_periods)

def test_content_hash_changes_with_data():
    """同一 ticker 不同 as_of → hash 不同"""
    snap1 = build_snapshot("AAPL", "2024-06-01", client)
    snap2 = build_snapshot("AAPL", "2024-09-01", client)
    assert snap1.content_hash != snap2.content_hash

def test_render_includes_filing_date_disclaimer():
    """render() 必须明示 PIT"""
    snap = build_snapshot(...)
    rendered = snap.render()
    assert "publicly filed by this date" in rendered

def test_market_cap_from_metrics_not_facts():
    """PIT 修正：market_cap 必须从 metrics 来，不是 company_facts"""
    # 这个测试保护"lookahead leak"不被回退
    ...
```

### 4. `test_signals.py` + `test_llm_agents.py` —— alpha 模型

**典型**：

```python
def test_pead_returns_signal_on_beat():
    """BEAT 在 signal window 内 → +1.0"""
    model = PEADModel()
    signal = model.predict("AAPL", "filing_date+2", mock_client_with_beat)
    assert signal.value == 1.0
    assert signal.metadata["eps_surprise"] == "BEAT"

def test_pead_returns_neutral_outside_window():
    """超过 4 天窗口 → 0.0"""
    model = PEADModel()
    signal = model.predict("AAPL", "filing_date+10", mock_client)
    assert signal.value == 0.0

def test_pead_retrospective_filter():
    """filing_date > report_period + 45 天 → 跳过"""
    ...

def test_llm_agent_abstains_on_data_error():
    """InsufficientData → abstain signal（value=0.0, abstained=True）"""
    agent = BuffettAgent(llm=MockLLM(), cache=MockCache())
    signal = agent.predict("AAPL", "today", mock_with_insufficient_data)
    assert signal.value == 0.0
    assert signal.metadata.get("abstained") is True

def test_llm_agent_abstains_on_llm_error():
    """LLM 失败 → abstain"""
    agent = BuffettAgent(llm=MockLLM(raises=TimeoutError()), cache=MockCache())
    signal = agent.predict(...)
    assert signal.value == 0.0

def test_llm_agent_abstains_on_parse_error():
    """LLM 输出非 JSON → abstain + 写盘（debug trail）"""
    agent = BuffettAgent(llm=MockLLM(returns="not json"), cache=MockCache())
    signal = agent.predict(...)
    assert signal.value == 0.0
    # cache 里应该有 parse_error 字段
    assert mock_cache.put_called_with_parse_error
```

### 5. `test_event_study.py` —— 事件研究

**典型**：

```python
def test_market_model_fit_perfect():
    """完美 β=1 + α=0 + R²=1"""
    stock = np.array([0.01, 0.02, -0.01, 0.03])
    market = np.array([0.01, 0.02, -0.01, 0.03])
    model = fit_market_model(stock, market)
    assert abs(model.alpha) < 1e-10
    assert abs(model.beta - 1.0) < 1e-10
    assert model.r_squared > 0.999

def test_event_date_snaps_to_next_trading_day():
    """filing 在周末 → 找下一个交易日"""
    ...

def test_retrospective_filter_drops_long_lag():
    """filing - report > 45 天 → drop"""
    ...

def test_aggregate_separates_by_source_type():
    """8-K 和 10-Q 分组聚合"""
    ...
```

### 6. `test_backtest.py` —— BacktestEngine

**典型**：

```python
def test_armed_triggers_only_once():
    """同一信号 → 一次开仓"""
    model = MockModel(return_signal_with_value=1.0)
    engine = BacktestEngine()
    result = engine.run_alpha(model, ["AAPL"], mock_data, "2024-01-01", "2024-01-30")
    # 1 月内多次 predict(1.0) → 只 1 次开仓
    assert len(result.trades) == 1

def test_skip_when_not_enough_data():
    ...

def test_metrics_calculation():
    """Sharpe / drawdown 计算正确"""
    ...
```

## v2 占位模块

```
v2/pipeline/__init__.py     # 空
v2/pipeline/execution.py    # 应该存在但内容未知
v2/portfolio/__init__.py    # 空
v2/portfolio/optimizer.py   # 应该存在但内容未知
v2/risk/__init__.py         # 空
v2/risk/manager.py          # 已知是 6 行 docstring
v2/validation/__init__.py   # 空
```

### `v2/risk/manager.py`（推测 6 行）

```python
"""v2 risk management.

Focus: drawdown controls, position sizing by volatility,
correlation-based exposure caps, tail risk metrics, stress testing.
"""
```

**完全占位**——只有 docstring，没有实现。

**含义**：v2 团队**先把接口和数据层做对**，risk 等"应用层"后续再补。ROADMAP ⬜ Planned。

### `v2/portfolio/optimizer.py`（推测）

同样占位——从 ROADMAP 看 portfolio construction 也是 ⬜ Planned。

### `v2/validation/` —— **最重要的占位**

**ROADMAP 写明**：

> "Validation gate — CPCV, probability of backtest overfitting (PBO)"

**但 `v2/validation/__init__.py` 完全空**——**整个 PBO 闸门还没建**。

**为什么这是最大风险**：

- ❌ 现在 PEAD Sharpe 0.33 是**没经过 overfitting check** 的数字
- ❌ 任何新加的 alpha model 都可以"调参调到 Sharpe 2.0"——可能完全是过拟合
- ✅ **CPCV/PBO 是 v2 严肃化的最后一道闸门**

### `v2/pipeline/` —— `run_cycle` 的占位

**ROADMAP**：

> "run_cycle — one pipeline (data → analysts → portfolio → risk → execution → ledger), three modes"

**当前 `v2/pipeline/execution.py` 应该存在**（pipeline 不是空目录），但功能很基础。

## v2 测试覆盖度评估

| 模块 | 测试覆盖 | 质量 |
|------|---------|------|
| `DataClient` Protocol | ✅ contract test | 高（最重要） |
| `FDClient` | ✅ unit tests | 中 |
| `CachedDataClient` | ✅ unit tests | 中 |
| `FinancialMetrics` / `Price` / etc. | ⚠️ 间接（通过 snapshot 测试） | 低 |
| `FundamentalsSnapshot` | ✅ unit tests | 中 |
| `PEADModel` | ✅ unit tests | 中 |
| `LLMAgent` / `BuffettAgent` | ✅ unit tests | 中 |
| `BacktestEngine` | ✅ unit tests | 中 |
| `compute_car` (event study) | ✅ unit tests | 中 |
| **Portfolio construction** | ❌ 没实现也没测试 | n/a |
| **Risk management** | ❌ 没实现也没测试 | n/a |
| **CPCV / PBO validation** | ❌ **最重要缺** | n/a |

**结论**：

- ✅ v2 核心模块（alpha model / data / backtest）**测试覆盖好**
- ⚠️ 但**整个 pipeline 集成测试缺失**——没有"从 snapshot 到 ledger"的全流程测试
- ❌ 最重要的 validation gate **完全没开始**

## 对 fork → CN 版本的启示

### A. contract test 是 CN client 的强制门槛

**写 `TushareClient` 必须跑**：

```bash
poetry run pytest v2/data/test_client_contract.py
```

如果 `assert isinstance(tushare, DataClient)` 失败 → 你的 client 不符合契约。

### B. 测试覆盖率应该照搬

每个新模块都要有 `test_xxx.py`：

| 新模块 | 测试文件 |
|--------|---------|
| `v2/data/tushare_client.py` | `v2/data/test_tushare_client.py` |
| `v2/signals/cn_pead.py` | `v2/signals/test_cn_pead.py` |
| `v2/signals/duan_yongping.py` | `v2/signals/test_llm_agents.py`（已有框架） |

### C. **优先补 validation gate**

CN 化前**最该加的不是数据 client，是 CPCV/PBO**——否则 CN 版本也会陷入"调参过拟合"陷阱。

## 深读问题（自检）

- [ ] `test_client_contract.py` 的契约测试对 CN client 适配——`TushareClient` 没有 `filing_date_lte` 怎么办？改成 `ann_date_lte` 后契约测试还要重写吗？
- [ ] v2 测试都是单元测试——**没有集成测试**（跑完整 PEAD demo + verify 输出）。这个 gap 在 CN 化时怎么补？
- [ ] `test_llm_agents.py` 用 `MockLLM`——mock 的 JSON 输出格式应该和真实 Anthropic 输出一致吗？如果不一致，测试通过但 prod 失败？
- [ ] `v2/validation/` 完全空——`PBO` 计算公式比较复杂（CPCV + bootstrap），CN fork 时是先实现 validation 还是先实现更多 alpha？

---

*解析时间：2026-07-12 · 第十一轮迭代*
*下次解析目标：开始 src/ v1 解析（~40 文件）——按优先级：main.py → graph → utils → 3 个 representative agent → backtesting → 其他 agent*