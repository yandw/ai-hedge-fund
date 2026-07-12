# src/graph/state.py + src/utils/analysts.py 深度解析

> 文件：`src/graph/state.py`（52 行）+ `src/utils/analysts.py`（200+ 行，部分）
> 地位：**v1 LangGraph state 定义 + 19 个 analyst 的注册表（唯一真源）**

## `src/graph/state.py` —— LangGraph state schema

### 文件作用一句话

**定义 LangGraph `AgentState` TypedDict + reducer + 一个 helper `show_agent_reasoning` 用来 pretty-print。**

### `AgentState`（line 15-18）

```python
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]
    data: Annotated[dict[str, any], merge_dicts]
    metadata: Annotated[dict[str, any], merge_dicts]
```

**3 个字段 + 各自的 reducer**：

| 字段 | 类型 | Reducer | 语义 |
|------|------|---------|------|
| `messages` | `Sequence[BaseMessage]` | `operator.add` | 列表拼接 |
| `data` | `dict[str, any]` | `merge_dicts` | dict 合并 |
| `metadata` | `dict[str, any]` | `merge_dicts` | dict 合并 |

### 关键概念：LangGraph Reducer

```python
def merge_dicts(a: dict, b: dict) -> dict:
    return {**a, **b}
```

**当两个 node 写同一个 key 时**：

```python
# analyst_1 写：data["analyst_signals"]["warren_buffett"] = {...}
# analyst_2 写：data["analyst_signals"]["cathie_wood"] = {...}
# merge_dicts 把两个 dict 合并
# 合并后：data["analyst_signals"] = {"warren_buffett": {...}, "cathie_wood": {...}}
```

**意义**：

- ✅ 19 个 analyst 并行写 `state["data"]["analyst_signals"]`——**不会互相覆盖**
- ✅ **dict 的 key 合并**——`warren_buffett` 和 `cathie_wood` 同时存在
- ✅ `operator.add` 让 messages 列表拼接

**这是 LangGraph 的核心机制**——所有 node 函数 return 的是 "delta state"，reducer 决定怎么合并。

### `show_agent_reasoning`（line 21-51）

```python
def show_agent_reasoning(output, agent_name):
    print(f"\n{'=' * 10} {agent_name.center(28)} {'=' * 10}")
    
    def convert_to_serializable(obj):
        if hasattr(obj, "to_dict"):
            return obj.to_dict()
        elif hasattr(obj, "__dict__"):
            return obj.__dict__
        elif isinstance(obj, (int, float, bool, str)):
            return obj
        elif isinstance(obj, (list, tuple)):
            return [convert_to_serializable(item) for item in obj]
        elif isinstance(obj, dict):
            return {key: convert_to_serializable(value) for key, value in obj.items()}
        else:
            return str(obj)
    
    if isinstance(output, (dict, list)):
        serializable_output = convert_to_serializable(output)
        print(json.dumps(serializable_output, indent=2))
    else:
        try:
            parsed_output = json.loads(output)
            print(json.dumps(parsed_output, indent=2))
        except json.JSONDecodeError:
            print(output)
    
    print("=" * 48)
```

#### 关键设计点

**1. `convert_to_serializable` 递归处理**

- Pandas Series/DataFrame → `to_dict()`
- 自定义对象 → `__dict__`
- 基础类型 → 直通
- 容器 → 递归
- 其他 → `str(obj)` fallback

**目的**：agent 输出可能是 dict / list / 自定义对象——转成 JSON-safe 再 print。

**2. 输出格式**

```
==========        warren_buffett_agent        ==========
{
  "signal": "bullish",
  "confidence": 75,
  "reasoning": "Durable moat..."
}
================================================
```

**居中 agent 名 + 上下分隔线 + JSON 缩进**——**调试时易读**。

#### 对比 v2

v2 没有"show reasoning"——因为 v2 的 Signal 已经是结构化的 Pydantic model，**直接 print 就行**。

v1 的 `show_agent_reasoning` 体现了**v1 的"agent 是黑盒"**哲学——agent 输出什么都可能，需要 helper 转 JSON。

### `state.py` 与 v2 的对比

| 维度 | v1 `state.py` | v2 |
|------|---------------|-----|
| State schema | TypedDict + LangGraph reducer | 函数参数 + return value |
| Concurrency | LangGraph 自动并行 | Python 单线程 for loop |
| Reasoning 输出 | helper 函数 | Signal 是 Pydantic |
| Messages | `Sequence[BaseMessage]` | 不存在 |

**v2 完全弃用了 LangGraph**——状态在函数调用间用普通 Python 传递。

## `src/utils/analysts.py` —— 19 个 analyst 注册表

### 文件作用一句话

**集中导入 19 个 analyst agent 函数，定义 `ANALYST_CONFIG` 注册表——`src/main.py::create_workflow` 直接读这个 dict。**

### `ANALYST_CONFIG` 结构

每个 entry：

```python
"warren_buffett": {
    "display_name": "Warren Buffett",
    "description": "The Oracle of Omaha",
    "investing_style": "Seeks high-quality businesses with durable competitive advantages...",
    "agent_func": warren_buffett_agent,
    "type": "analyst",
    "order": 10,
},
```

**5 个字段**：

| 字段 | 用途 |
|------|------|
| `display_name` | CLI 显示（`--analyst warren_buffett` → "Warren Buffett"） |
| `description` | 给 LLM 看的"人设" |
| `investing_style` | 投资哲学描述 |
| `agent_func` | agent 函数本身 |
| `type` | "analyst"（未来可能有 "risk" / "executor" 等） |
| `order` | CLI 显示排序 |

### 19 个 analyst 全清单（基于 ANALYST_CONFIG）

按 `order` 排序：

| # | Key | Display | 类型 |
|---|-----|---------|------|
| 0 | `aswath_damodaran` | Aswath Damodaran | 估值（学术） |
| 1 | `ben_graham` | Ben Graham | 价值 |
| 2 | `bill_ackman` | Bill Ackman | 激进股东 |
| 3 | `cathie_wood` | Cathie Wood | 增长 / 颠覆 |
| 4 | `charlie_munger` | Charlie Munger | 价值 + 理性 |
| 5 | `michael_burry` | Michael Burry | 逆向 |
| 6 | `mohnish_pabrai` | Mohnish Pabrai | 价值（Dhandho） |
| 7 | `nassim_taleb` | Nassim Taleb | 风险 / 反脆弱 |
| 8 | `peter_lynch` | Peter Lynch | 增长（"买你懂的"） |
| 9 | `phil_fisher` | Phil Fisher | 增长（深度研究） |
| 10 | `warren_buffett` | Warren Buffett | 价值（最经典） |
| 11 | `rakesh_jhunjhunwala` | Rakesh Jhunjhunwala | 印度大牛 |
| 12 | `stanley_druckenmiller` | Stanley Druckenmiller | 宏观 |
| 13 | `valuation` | Valuation | 量化估值（无人名） |
| 14 | `fundamentals` | Fundamentals | 基本面（无人名） |
| 15 | `sentiment` | Sentiment | 情绪（无人名） |
| 16 | `technicals` | Technicals | 技术（无人名） |
| 17 | `news_sentiment` | News Sentiment | 新闻情绪（无人名） |
| 18 | `growth_agent` | Growth | 增长（无人名） |

**19 个**——其中 12 个有人名 persona，7 个是 quant 风格。

### 与 v2 对比

| 维度 | v1 ANALYST_CONFIG | v2 ALPHA_MODEL_REGISTRY |
|------|-------------------|-------------------------|
| 字段数 | 6 个（display/desc/style/func/type/order） | 1 个映射（name → class） |
| 数量 | 19 | 2（PEAD + Buffett） |
| 增删难度 | 改 dict | 改 dict |
| 加载方式 | 顶部 import + dict 构造 | 顶部 import + dict 构造 |

**v1 配置更丰富**（display_name 等）——给 CLI 提供人机交互。  
**v2 配置更精简**——给 engine 提供 name → class 即可。

### `get_analyst_nodes()`（推测）

```python
def get_analyst_nodes():
    return {key: (config["agent_func"].__name__, config["agent_func"]) for key, config in ANALYST_CONFIG.items()}
```

**返回**：`{key: (node_name, node_func)}`——`src/main.py::create_workflow` 用来 `add_node(name, func)`。

### `ANALYST_ORDER`（推测）

按 `order` 排序的 key 列表——CLI 选择 analyst 时按这个顺序展示。

## 设计取舍

### 1. TypedDict vs Pydantic for state

✅ **TypedDict**：

- LangGraph 原生支持
- 静态检查 + IDE 自动补全
- 不强制验证

❌ **Pydantic BaseModel**：

- 严格验证
- 但 LangGraph state 需要 reducer 机制

**判断**：合理。LangGraph 生态推荐 TypedDict。

### 2. `merge_dicts` 自己实现 vs dict.update

✅ **`{**a, **b}`**：

- 显式
- 不依赖 dict.update 的语义变化

❌ **`a.update(b)`**：

- 修改 in place
- 在 reducer 里**不推荐**（应返回新对象）

**判断**：合理。

### 3. 19 个 analyst 都进注册表

✅ **全部强制 import**：

```python
from src.agents.warren_buffett import warren_buffett_agent
from src.agents.cathie_wood import cathie_wood_agent
...
```

- 任何 import 错误立刻看到

❌ **lazy import**：

- 按需加载
- 但失去静态检查

**判断**：合理。19 个 import 不慢。

### 4. `display_name` + `description` + `investing_style`

✅ **多字段**：

- CLI 选 analyst 时显示 description（人话）
- LLM 看 investing_style（agent 之间可能引用）

❌ **单一字段**：

- 简洁
- 但 LLM 难理解"这人在做什么"

**判断**：合理。

## 对 fork → CN 版本的启示

### A. v2 不需要这套

v2 的 `ALPHA_MODEL_REGISTRY` 已经简化了——**CN 化不需要"display_name"等字段**（v2 的 CLI 更精简）。

### B. CN 投资者 persona 的模板

如果加 CN persona，可以参考 v1 ANALYST_CONFIG 的字段：

```python
ALPHA_MODEL_REGISTRY = {
    "duan_yongping": {
        "display_name": "段永平",
        "description": "价值投资 + 长期持有",
        "investing_style": "...",
        "agent_func": DuanYongpingAgent,
        "order": 20,
    },
}
```

**v2 简化版**只需要 `name → class`——但展示层面可以照搬 v1 的多字段配置。

### C. v1 的人设字段对 LLM 决策有用

`investing_style` 是**给 LLM 看的**——**不是装饰**。它会被 prompt 引用："Warren Buffett agent 用 investing_style 字段的风格判断"。

**CN 化的 persona 也应该有**类似 investing_style 的元数据。

## 深读问题（自检）

- [ ] `merge_dicts` 用 `{**a, **b}` 而不是 `a | b`（Python 3.9+）——为什么？向后兼容？
- [ ] 19 个 analyst 的 `order` 字段——CLI 选多个时按 order 显示还是按 selected 顺序？LangGraph 内部执行顺序呢？
- [ ] `agent_func` 直接是函数——如果某个 agent 改签名（加新参数），注册表怎么强制对齐？
- [ ] `description` 和 `investing_style` 是给人看还是给 LLM 看？如果是给 LLM 看，prompt 里怎么引用？

---

*解析时间：2026-07-12 · 第十三轮迭代*
*下次解析目标：`src/utils/{llm,progress,display,api_key,visualize,ollama,docker}.py`（7 个工具文件）*