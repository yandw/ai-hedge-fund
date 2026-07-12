# v2/models.py 深度解析

> 文件：`v2/models.py` · 72 行 · 全部 5 个 Pydantic 模型
> 地位：**全流水线的数据类型唯一真源**——alpha model 输出 Signal，portfolio 层组合成 PortfolioTarget，execution 层产生 TradeOrder + ExecutionResult。

## 文件作用一句话

**定义 v2 整条流水线流通的 5 个数据结构，全部用 Pydantic BaseModel。**

## 5 个模型一览

| 模型 | 在流水线中的位置 | 谁产出 | 谁消费 |
|------|-----------------|--------|--------|
| `Signal` | Alpha 模型输出 | `AlphaModel.predict()` | Portfolio / Backtest / LLM agent |
| `QuantSignals` | 一只 ticker 一日的所有信号聚合 | （目前未用） | Portfolio construction |
| `PortfolioTarget` | 投资组合目标权重 | Portfolio optimizer | Execution |
| `TradeOrder` | 单笔订单指令 | Execution | Broker |
| `ExecutionResult` | 订单执行结果 | Broker | Ledger |

## 关键模型逐个拆解

### `Signal`（line 14-27）—— 最核心

```python
class Signal(BaseModel):
    model_name: str    # e.g. 'pead', 'buffett'
    ticker: str
    date: str          # as-of date (YYYY-MM-DD)
    value: float       # [-1.0, +1.0]
    reasoning: str | None
    components: dict[str, float]
    metadata: dict[str, Any]
```

**关键设计**：

1. **`value: float ∈ [-1, +1]`**——这是 PEAD / Buffett / 任何 alpha model 都必须遵守的统一形状。conviction + 方向合一表达：
   - `+1.0`：强烈看多
   - `0.0`：中性（无观点）
   - `-1.0`：强烈看空
   - 中间任意实数
2. **`reasoning: str | None`**——LLM agent 用（persona 的"论据"），quant 可以为 None。
3. **`components: dict[str, float]`**——quant 用的分解（如 PEAD 可以写 `{"surprise_pct": 0.12, "days_since_filing": 2}`），LLM agent 通常空着。
4. **`metadata: dict[str, Any]`**——杂项自由字段（模型名、prompt_key、abstained 标志等）。注意是 `Any` 类型，最宽松的扩展点。
5. **`date: str`**——用 `YYYY-MM-DD` 字符串而非 `datetime.date`。原因：跨数据源（Financial Datasets、Tushare、AKShare）字符串最稳定，不用处理时区。

### `QuantSignals`（line 30-36）—— 多 signal 聚合

```python
class QuantSignals(BaseModel):
    ticker: str
    date: str
    signals: dict[str, Signal]  # model_name -> Signal
    composite_score: float | None
```

**当前状态**：定义了但**未被任何模块使用**——是 portfolio construction 的预备接口，等 portfolio layer 落地后会用。

### `PortfolioTarget`（line 43-49）—— 目标权重

```python
class PortfolioTarget(BaseModel):
    weights: dict[str, float]   # ticker -> weight (-1 to +1)
    expected_return: float | None
    expected_risk: float | None
```

**关键约束**：`weights` 也是 `[-1, +1]`，允许**多空双向**（负权重 = 做空）。预期收益/风险字段是 optional，等 optimizer 真正实现时填。

### `TradeOrder` + `ExecutionResult`（line 57-72）—— 执行

```python
class TradeOrder(BaseModel):
    ticker: str
    action: Literal["buy", "sell", "short", "cover"]
    shares: int = 0
    price: float = 0.0
    estimated_cost: float = 0.0
    reason: str = ""

class ExecutionResult(BaseModel):
    orders: list[TradeOrder]
    total_cost: float = 0.0
```

**4 个 action 的语义**（来自 Action enum）：

| action | 含义 |
|--------|------|
| `buy` | 开多 / 加多 |
| `sell` | 平多 |
| `short` | 开空 / 加空 |
| `cover` | 平空 |

注意：**LLM agent 完全不接触 TradeOrder**——`Signal.value` 是观点，下单是 Execution 层从 `PortfolioTarget` 转过来的。

## 设计取舍（值得讨论）

### 1. 为什么用字符串日期而不是 `datetime`？

✅ **优势**：跨数据源兼容（FD 用 ISO 字符串，Tushare 也是），序列化简单
❌ **代价**：失去类型保护（`"2024-13-99"` 不会报错）

**判断**：合理。Pydantic v2 可以加 `field_validator` 加强检查，但当前没加——遵循 YAGNI，等真有 bug 再补。

### 2. 为什么 `metadata: dict[str, Any]` 而不是严格 schema？

✅ **优势**：每个 alpha model 可以塞自己想塞的元数据（缓存 key、abstain 原因、模型名），不破坏 Signal 的统一形状
❌ **代价**：失去类型检查（写错字段名不会报错）

**判断**：合理。alpha model 的元数据**不该**影响主流程，应该自由扩展。

### 3. 为什么 `Signal.value` 用 `[-1, +1]` 而不是更精细的 `[-100, +100]`？

✅ **优势**：单位一致性（百分比和 raw 都映射到 [-1, +1]），直觉清晰
❌ **代价**：精度损失（int 100 步 vs float 无限）

**判断**：完全合理。这是 alpha model 的**输出协议**，不是内部计算用的。

### 4. `QuantSignals` 定义了但没人用——是 YAGNI 吗？

⚠️ **争议点**：YAGNI 原则下应该删掉。**但**：
- 它是 Portfolio Construction 阶段的明确输入契约
- Pipeline 演进路线图需要它
- 字段少（4 个），不维护成本几乎为零

**判断**：保留。**接口契约先于实现**是合理的工程纪律。但应该在 docstring 里加一行"current: unused; reserved for portfolio layer"以免混淆。

## 与其他模块的关系

```
v2/signals/base.py::AlphaModel.predict()  →  Signal
                                            ↓
                                   v2/backtesting/engine.py
                                            ↓
                                   TradeOrder (current path: 直接生成)
                                            ↓
                                   ExecutionResult

PortfolioTarget / QuantSignals → 当前未消费（pipeline/portfolio 占位）
```

**当前实际数据流**：`Signal` → `Trade`（v2/backtesting/models.py 自己定义，不是这里的 TradeOrder）→ 回测指标。

`TradeOrder` / `ExecutionResult` / `PortfolioTarget` 是**预留的 pipeline 接口**，v2.pipeline / v2.portfolio 落地后会激活。

## 一句话总结

v2/models.py 是流水线的**宪法**——5 个 Pydantic 模型定义了"信号是什么、目标是什么、订单是什么"。文件 72 行，是 v2 哲学的最精炼表达：**alpha model 出观点、portfolio 出权重、execution 出订单、ledger 记账**，LLM 在第一步之后就不碰了。

## 对 fork → CN 版本的启示

1. **完全不动这个文件**——它是 v2 架构的内核，CN 化不需要改。
2. CN 化要加的 LLM agent（DeepSeek/Qwen）输出仍然是 `Signal`，**不需要新 model 类**。
3. A 股特有的执行约束（T+1、涨跌停、印花税单向）应该加在 **`v2/pipeline/execution.py`** 而不是改 TradeOrder——TradeOrder 的语义保持通用，execution layer 处理市场特殊性。

## 深读问题（自检）

回答不出就回去重读：

- [ ] `Signal.value` 必须是 `[-1, +1]`，**谁在 enforce**？（提示：没有——靠 alpha model 自律）
- [ ] LLM agent 的 `Signal.metadata` 实际填了哪几个 key？（提示：看 `v2/signals/llm_agent.py::_to_signal`）
- [ ] `PortfolioTarget.weights` 的 key 可以是空吗？空 dict 意味着什么？
- [ ] `TradeOrder.action` 为什么不支持 `hold`？（提示：`Action` enum 在 `src/backtesting/types.py` 里有 `HOLD`，v2 没继承）

---

*解析时间：2026-07-12 · 第一轮迭代*
*下次解析目标：`v2/data/protocol.py`*