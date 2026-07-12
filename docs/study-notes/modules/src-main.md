# src/main.py 深度解析

> 文件：`src/main.py` · 180 行 · 1 个 LangGraph StateGraph 构造 + 1 个 CLI 入口
> 地位：**v1 LangGraph 应用的入口**——组装 StateGraph、运行 agent、返回决策

## 文件作用一句话

**用 LangGraph `StateGraph` 把 19 个 analyst → risk_manager → portfolio_manager 串成一条图，运行后输出 trading decisions。**

## v1 整体架构（与 v2 对比）

```
v1 (src/main.py):
  start_node → [analyst_1, analyst_2, ..., analyst_19]  ← fan-out
                  ↓       ↓              ↓
              risk_management_agent                       ← fan-in
                  ↓
              portfolio_manager                           ← final decision
                  ↓
                  END

v2 (v2/backtesting/engine.py):
  alpha.predict(ticker, date) for each ticker
                  ↓
              BacktestEngine (single-threaded loop)
                  ↓
              Trades + Metrics
```

**关键差别**：

| 维度 | v1 | v2 |
|------|----|----|
| 拓扑 | 19 路 fan-out + 1 路 fan-in | 串行 per-ticker |
| 决策者 | portfolio_manager (LLM) | 确定性代码 + alpha signal |
| 每次调用的"输入" | 多 ticker | 单 ticker + date |
| 输出 | trading decisions (LLM JSON) | Trade list |
| 中间产物 | `analyst_signals: dict` | 每个 ticker 一次 PEAD |

## `run_hedge_fund` 主函数（line 46-92）

```python
def run_hedge_fund(
    tickers,
    start_date,
    end_date,
    portfolio,
    show_reasoning=False,
    selected_analysts=[],
    model_name="gpt-4.1",
    model_provider="OpenAI",
):
    progress.start()
    try:
        workflow = create_workflow(selected_analysts if selected_analysts else None)
        agent = workflow.compile()
        
        final_state = agent.invoke({
            "messages": [HumanMessage(content="Make trading decisions based on the provided data.")],
            "data": {
                "tickers": tickers,
                "portfolio": portfolio,
                "start_date": start_date,
                "end_date": end_date,
                "analyst_signals": {},
            },
            "metadata": {
                "show_reasoning": show_reasoning,
                "model_name": model_name,
                "model_provider": model_provider,
            },
        })
        
        return {
            "decisions": parse_hedge_fund_response(final_state["messages"][-1].content),
            "analyst_signals": final_state["data"]["analyst_signals"],
        }
    finally:
        progress.stop()
```

### 关键设计点

#### 1. **`start_date` / `end_date` 进了 state 但实际不参与计算**

```python
"data": {
    "tickers": tickers,
    ...
    "start_date": start_date,
    "end_date": end_date,
    "analyst_signals": {},
},
```

**注意**：v1 的 analyst **不按日期调**——它只看最新数据。

**后果**：v1 是"现在怎么看"，**不是回测**——`backtester.py` 在外层循环里**每天调一次** `run_hedge_fund`。

**对比 v2**：v2 的 `predict(ticker, date, data_client)` 是 **point-in-time** 的。

#### 2. **state 里的 messages 实际不传递**

```python
"messages": [HumanMessage(content="Make trading decisions based on the provided data.")],
```

**意义**：LangGraph 的 messages reducer 是 `operator.add`——agent 之间**共享对话历史**。

但**实际**v1 的 analyst 把 reasoning 写到 `state["data"]["analyst_signals"]` 里——**messages 是空走过场**。

#### 3. `parse_hedge_fund_response` 把 LLM 输出转 dict

```python
"decisions": parse_hedge_fund_response(final_state["messages"][-1].content),
```

**为什么**：`messages[-1].content` 是 LLM 输出的字符串，要 JSON parse。

#### 4. `progress.start()` / `stop()` 包装

```python
try:
    ...
finally:
    progress.stop()
```

**保证**：即使 agent 出错，progress bar 也会被关闭——**不留终端残留**。

## `create_workflow` —— 构造图（line 100-130）

```python
def create_workflow(selected_analysts=None):
    workflow = StateGraph(AgentState)
    workflow.add_node("start_node", start)
    
    analyst_nodes = get_analyst_nodes()
    
    if selected_analysts is None:
        selected_analysts = list(analyst_nodes.keys())
    
    for analyst_key in selected_analysts:
        node_name, node_func = analyst_nodes[analyst_key]
        workflow.add_node(node_name, node_func)
        workflow.add_edge("start_node", node_name)
    
    workflow.add_node("risk_management_agent", risk_management_agent)
    workflow.add_node("portfolio_manager", portfolio_management_agent)
    
    for analyst_key in selected_analysts:
        node_name = analyst_nodes[analyst_key][0]
        workflow.add_edge(node_name, "risk_management_agent")
    
    workflow.add_edge("risk_management_agent", "portfolio_manager")
    workflow.add_edge("portfolio_manager", END)
    
    workflow.set_entry_point("start_node")
    return workflow
```

### 图的结构

```
        start_node
        /  |  \  \  ...  (19 路 fan-out)
       /   |   \
   warren  cathie  ... (analyst nodes)
       \   |   /
        \  |  /
   risk_management_agent   (fan-in)
            ↓
   portfolio_manager
            ↓
           END
```

### 关键设计点

#### 1. **fan-out + fan-in**

✅ **每个 analyst 都从 start_node 出发**

```python
for analyst_key in selected_analysts:
    ...
    workflow.add_edge("start_node", node_name)
```

✅ **每个 analyst 都连到 risk_management_agent**

```python
for analyst_key in selected_analysts:
    workflow.add_edge(node_name, "risk_management_agent")
```

**好处**：

- analyst 之间**并行执行**（LangGraph 默认并行 fan-out）
- analyst 之间**不互相依赖**

#### 2. **`start_node` 是个空函数**

```python
def start(state: AgentState):
    """Initialize the workflow with the input message."""
    return state
```

**意义**：LangGraph 需要**一个 entry point**——这个函数**什么都不做**只 return state。

#### 3. **`analyst_nodes` 从 `get_analyst_nodes()` 拿**

```python
analyst_nodes = get_analyst_nodes()
```

**指向**：`src/utils/analysts.py::ANALYST_CONFIG`（19 个 analyst 的注册表）。

**这是唯一**——`selected_analysts` 默认是注册表的所有 key。

#### 4. **selected_analysts == None vs [] 的处理**

```python
if selected_analysts is None:
    selected_analysts = list(analyst_nodes.keys())
```

✅ **`None` → 全部 analyst**

❌ **`[]` → 空集（没 analyst）**

**判断**：合理。`None` 显式表示"全选"。

## `parse_hedge_fund_response` —— JSON 解析（line 30-42）

```python
def parse_hedge_fund_response(response):
    try:
        return json.loads(response)
    except json.JSONDecodeError as e:
        print(f"JSON decoding error: {e}\nResponse: {repr(response)}")
        return None
    except TypeError as e:
        print(f"Invalid response type (expected string, got {type(response).__name__}): {e}")
        return None
    except Exception as e:
        print(f"Unexpected error while parsing response: {e}\nResponse: {repr(response)}")
        return None
```

**3 个 catch**：

1. `JSONDecodeError` —— 输出不是 JSON
2. `TypeError` —— `response` 不是 string
3. 通用 `Exception` —— 其他

**对比 v2 的 `extract_json`**：

| 维度 | v1 `parse_hedge_fund_response` | v2 `extract_json` |
|------|-------------------------------|-------------------|
| 三段降级 | ❌ 只有 `json.loads` | ✅ fence / whole / balanced |
| 失败处理 | print + return None | raise `LLMParseError` |
| 用法 | 顶层决策解析 | LLM agent 内部 |

**判断**：v1 实现比较脆弱，**只对"LLM 输出干净 JSON"工作**——一旦 LLM 输出带 markdown 或 prose 就崩。

**v2 的改进**：三段降级。

## CLI 入口（line 133-179）

```python
if __name__ == "__main__":
    inputs = parse_cli_inputs(
        description="Run the hedge fund trading system",
        require_tickers=True,
        default_months_back=None,
        include_graph_flag=True,
        include_reasoning_flag=True,
    )
    
    tickers = inputs.tickers
    selected_analysts = inputs.selected_analysts
    
    portfolio = {
        "cash": inputs.initial_cash,
        "margin_requirement": inputs.margin_requirement,
        "margin_used": 0.0,
        "positions": {
            ticker: {
                "long": 0,
                "short": 0,
                "long_cost_basis": 0.0,
                "short_cost_basis": 0.0,
                "short_margin_used": 0.0,
            }
            for ticker in tickers
        },
        "realized_gains": {
            ticker: {"long": 0.0, "short": 0.0}
            for ticker in tickers
        },
    }
    
    result = run_hedge_fund(...)
    print_trading_output(result)
```

### 关键设计点

#### 1. **`parse_cli_inputs` 抽到 `src/cli/input.py`**

- ✅ CLI 参数解析**独立于业务逻辑**
- ✅ 复用：`src/backtester.py` 也用同一个

#### 2. **portfolio 构造在 main 里**

```python
portfolio = {
    "cash": inputs.initial_cash,
    "margin_requirement": inputs.margin_requirement,
    "margin_used": 0.0,
    "positions": {...},
    "realized_gains": {...},
}
```

**这是 TypedDict 形状**（见 `src/backtesting/types.py::PortfolioSnapshot`）。

**为什么在 main 里而不是在 `run_hedge_fund` 里**：

- ✅ CLI 模式 vs backtester 模式可以不同构造（backtester 已经有持仓）
- ✅ 解耦 CLI 和业务

#### 3. `print_trading_output(result)` 输出

- 详见 `src/utils/display.py`
- 输出决策表格 + reasoning

## v1 vs v2 决策模式的根本差别

### v1: LLM 决定仓位

```python
# portfolio_manager agent 内部
# LLM 收到所有 analyst_signals，决定：
{
  "AAPL": {"action": "buy", "quantity": 100},
  "MSFT": {"action": "sell", "quantity": 50},
  ...
}
```

**问题**：

- LLM 的"quantity"决定**没原则**——同一个 prompt 多次跑结果不同
- **不可重复**——回测的 Sharpe 数学上无效
- **风险不可控**——LLM 不知道"组合层面的相关性、波动率、杠杆上限"

### v2: LLM 只出观点，代码决定仓位

```python
# v2/backtesting/engine.py
for ticker in tickers:
    signal = alpha.predict(ticker, date, data_client)
    if armed and abs(signal.value) > threshold:
        trade = open_position(signal, ticker, date)
```

**优势**：

- ✅ **确定性**——同样 input 同样 output
- ✅ **可重复**——回测数学有效
- ✅ **风险在代码层**——v2 的 risk manager（虽然还是占位）会接管

**这就是 VISION.md 反复强调的"LLM 不碰交易"。**

## 设计取舍

### 1. LangGraph fan-out + fan-in

✅ **并行 analyst**：

- LangGraph 默认并行跑 fan-out
- 19 个 LLM 调用**并行**

❌ **串行 analyst**：

- 慢 19 倍
- 但更稳定（不容易 rate limit）

**判断**：合理（用了 LangGraph 就接受它的并发模型）。

### 2. messages 字段几乎没用

✅ **保留**（LangGraph state 默认字段）：

- messages reducer 是 langgraph 的"标志"

❌ **删除**：简化 state

**判断**：当前 messages 是走过场，但**保留**给将来 agent 间对话留口。

### 3. `parse_hedge_fund_response` 简单粗暴

✅ **只 `json.loads`**：

- 简单
- 如果 LLM 输出有 markdown fence 会崩

❌ **用 v2 的 `extract_json`**：

- 三段降级
- 更鲁棒

**判断**：v1 是更早期的实现，**没享受到 v2 的 robustness**。

## 对 fork → CN 版本的启示

### A. **v1 这套架构本身就要被替换**

CN fork **不应该 fork v1**——直接 fork v2，加 CN LLM + CN data client 就好。

### B. v1 的一些工具可复用

- `src/utils/llm.py::call_llm` —— **13 家 provider**，比 v2 完善
- `src/utils/progress.py::progress` —— rich 进度条，v2 demo 没用到
- `src/cli/input.py::parse_cli_inputs` —— CLI 参数解析，比 v2 详尽

**CN 化策略**：

- ✅ fork v2 主线
- ✅ 把 v1 的 LLM provider / progress / CLI 工具"反向贡献"到 v2

### C. `parse_hedge_fund_response` 的脆弱是反面教材

CN 化的 LLM agent **必须用 v2 的 `extract_json`**——别重蹈 v1 覆辙。

## 深读问题（自检）

- [ ] LangGraph 默认并行跑 19 个 analyst——如果其中 8 个用 OpenAI、5 个用 Anthropic、6 个用 DeepSeek，**它们各自的 rate limit 怎么处理**？
- [ ] `state["data"]["analyst_signals"]` 是 dict——19 个 analyst 一起写，会不会**互相覆盖**？还是 langgraph 用 reducer 合并？
- [ ] `run_hedge_fund` 把 start_date / end_date 放进 state 但 analyst 不用——这是 dead code 还是有未来用途？
- [ ] `parse_hedge_fund_response` 不区分"是 portfolio_manager 的输出"——如果 LLM 输出 `"actions": {...}`（复数）vs `"decisions": {...}` 都能 parse 成功——但上层会怎么处理歧义？

---

*解析时间：2026-07-12 · 第十二轮迭代*
*下次解析目标：`src/graph/state.py` + `src/utils/{analysts,llm,progress}.py` + `src/utils/{display,api_key,visualize,ollama,docker}.py`*