# v2/event_study/plot.py + app/backend/ 深度解析（最终轮）

> 文件：`v2/event_study/plot.py`（179 行）+ `app/backend/{database,models,repositories,routes,services}.py`（约 15 文件）
> 地位：**v2 事件研究的可视化** + **v1 Web 应用后端的完整实现**

## 第一部分：`v2/event_study/plot.py` —— 学术风格 CAR 图表

### 文件作用一句话

**3 个 matplotlib 函数，把 EventStudyResult 转成学术风格的 CAR 图表——bar chart、histogram、time series。**

### 3 个函数

| 函数 | 图类型 | 回答什么问题 |
|------|--------|------------|
| `plot_car_by_source` | 分组柱状图 | "earnings 事件对价格有影响吗？按 filing 类型分组的 CAR 是多少？" |
| `plot_car_distribution` | 直方图 | "CAR 的分布是集中还是分散？有多大的方差？" |
| `plot_cumulative_ar` | 时间序列 | "PEAD 是真的存在吗？CAR 随时间怎么累积？" |

### `plot_car_by_source` 详解（line 15-65）

```python
def plot_car_by_source(result: EventStudyResult) -> Figure:
    fig, ax = plt.subplots(figsize=(10, 6))
    
    aggregates = [a for a in result.aggregates if a.windows]
    if not aggregates:
        ax.text(0.5, 0.5, "No data", ha="center", va="center", transform=ax.transAxes)
        return fig
    
    # X-axis: windows; bar group per source_type
    window_labels = sorted({w.window for a in aggregates for w in a.windows})
    source_types = [a.source_type for a in aggregates]
    x = np.arange(len(window_labels))
    width = 0.8 / len(source_types)
    
    for i, agg in enumerate(aggregates):
        window_map = {w.window: w for w in agg.windows}
        
        means = []
        errors_lo = []
        errors_hi = []
        for wl in window_labels:
            w = window_map.get(wl)
            if w:
                means.append(w.mean_car * 100)
                errors_lo.append((w.mean_car - w.ci.lower) * 100)
                errors_hi.append((w.ci.upper - w.mean_car) * 100)
            else:
                means.append(0)
                errors_lo.append(0)
                errors_hi.append(0)
        
        offset = (i - len(source_types) / 2 + 0.5) * width
        ax.bar(x + offset, means, width, yerr=[errors_lo, errors_hi], capsize=3, label=agg.source_type)
    
    ax.set_xticks(x)
    ax.set_xticklabels(window_labels)
    ax.set_ylabel("Mean CAR (%)")
    ax.set_title("Mean CAR by Source Type and Window")
    ax.axhline(0, color="black", linewidth=0.8)
    ax.legend()
    fig.tight_layout()
    return fig
```

**图表设计**：

- **X 轴**：3 个 window（[0,+1], [0,+5], [0,+20]）
- **每个 X 位置**：一组 bar（按 source_type 分组）
- **Y 轴**：mean CAR %
- **误差线**：bootstrap 95% CI（非对称）
- **零线**：axhline at 0（参考线）

**学术意义**：

- 看 8-K vs 10-Q 的 CAR 是否不同（filing 时效性差异）
- 看 (0, +5) / (0, +20) 的 CAR 是否持续 drift（PEAD 验证）

### `plot_car_distribution` 与 `plot_cumulative_ar`

（**plot.py 中其余函数**——同样 matplotlib pattern）

- **`plot_car_distribution`**：每个 source_type 一个子图，画 CAR 直方图 + mean / 0 参考线
- **`plot_cumulative_ar`**：画 day-by-day 累积 AR（PEAD 看到 day 5-20 的 drift 曲线）

### 与 v2 demo/backtest.py 对比

| 维度 | `v2/event_study/plot.py` | `v2/demo/backtest.py` |
|------|------------------------|---------------------|
| 库 | matplotlib | rich |
| 输出 | PNG / PDF 文件 | 终端 |
| 风格 | 学术论文 | 演示 |
| 用例 | 论文 / 报告 | demo / 演讲 |

## 第二部分：`app/backend/` —— v1 Web 应用完整后端

### 数据库层

#### 4 个 SQLAlchemy ORM model

**`HedgeFundFlow`** —— 用户保存的"flow 配置"：

```python
class HedgeFundFlow(Base):
    __tablename__ = "hedge_fund_flows"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(200))
    description = Column(Text)
    nodes = Column(JSON)      # React Flow 节点
    edges = Column(JSON)      # React Flow 边
    viewport = Column(JSON)   # 视口状态
    data = Column(JSON)       # 节点内部状态
    is_template = Column(Boolean)
    tags = Column(JSON)
```

**关键**：**`nodes` 和 `edges` 是 React Flow 的 JSON**——前端 React Flow 编辑器直接序列化存 DB。

**`HedgeFundFlowRun`** —— 一次 flow 的实际运行：

```python
class HedgeFundFlowRun(Base):
    __tablename__ = "hedge_fund_flow_runs"
    
    id = Column(Integer, primary_key=True)
    flow_id = Column(Integer, ForeignKey("hedge_fund_flows.id"))
    status = Column(String)        # IDLE / IN_PROGRESS / COMPLETE / ERROR
    trading_mode = Column(String)  # one-time / continuous / advisory
    schedule = Column(String)
    request_data = Column(JSON)    # 输入参数
    initial_portfolio = Column(JSON)
    final_portfolio = Column(JSON)
    results = Column(JSON)         # 输出结果
    error_message = Column(Text)
    run_number = Column(Integer)
```

**3 个 state**：`request_data` (input) / `initial_portfolio` (start state) / `final_portfolio` + `results` (output)。

**`HedgeFundFlowRunCycle`** —— 一个 run 内的单次 cycle（每个 trading day）：

```python
class HedgeFundFlowRunCycle(Base):
    __tablename__ = "hedge_fund_flow_run_cycles"
    
    id = Column(Integer, primary_key=True)
    flow_run_id = Column(Integer, ForeignKey("hedge_fund_flow_runs.id"))
    cycle_number = Column(Integer)
    
    started_at = Column(DateTime)
    completed_at = Column(DateTime)
    
    analyst_signals = Column(JSON)
    trading_decisions = Column(JSON)
    executed_trades = Column(JSON)
    portfolio_snapshot = Column(JSON)
    performance_metrics = Column(JSON)
    
    llm_calls_count = Column(Integer)
    api_calls_count = Column(Integer)
    estimated_cost = Column(String)
```

**关键**：

- **每个 cycle 都记录所有状态** —— 时间序列审计
- **cost tracking** —— llm_calls / api_calls / estimated_cost
- **trigger_reason** —— scheduled / manual / market_event

**`ApiKey`** —— 用户的 provider keys：

```python
class ApiKey(Base):
    __tablename__ = "api_keys"
    
    id = Column(Integer, primary_key=True)
    provider = Column(String)   # ANTHROPIC_API_KEY / OPENAI_API_KEY / ...
    key_value = Column(Text)     # 加密（注释提到）
    is_active = Column(Boolean)
    last_used = Column(DateTime)
```

**注释明示**：`# The actual API key (encrypted in production)` —— **生产应该加密，但当前是明文**。

### 关键关系图

```
HedgeFundFlow (1) ──< (N) HedgeFundFlowRun
                              │
                              └──< (N) HedgeFundFlowRunCycle

ApiKey (独立表) ──> 在 request 时被注入到 graph state
```

### 路由层

#### `routes/hedge_fund.py` —— 核心端点

**两个 POST 端点**：

| 端点 | 作用 | 关键差异 |
|------|------|---------|
| `POST /hedge-fund/run` | 跑一次 agent 决策 | 一次性，结果即时返回 |
| `POST /hedge-fund/backtest` | 跑连续 backtest | 每天循环，streaming |

**`/run` 端点**（line 26-160）：

```python
@router.post(path="/run", responses={...})
async def run(request_data: HedgeFundRequest, request: Request, db: Session = Depends(get_db)):
    try:
        # 1. 注入 API keys
        if not request_data.api_keys:
            request_data.api_keys = api_key_service.get_api_keys_dict()
        
        # 2. 构造 portfolio
        portfolio = create_portfolio(...)
        
        # 3. 构造 LangGraph
        graph = create_graph(graph_nodes=..., graph_edges=...)
        graph = graph.compile()
        
        # 4. StreamingResponse（SSE 协议）
        async def event_generator():
            progress_queue = asyncio.Queue()
            
            def progress_handler(agent_name, ticker, status, analysis, timestamp):
                event = ProgressUpdateEvent(...)
                progress_queue.put_nowait(event)
            
            progress.register_handler(progress_handler)
            
            try:
                run_task = asyncio.create_task(run_graph_async(...))
                yield StartEvent().to_sse()
                
                while not run_task.done():
                    try:
                        event = await asyncio.wait_for(progress_queue.get(), timeout=1.0)
                        yield event.to_sse()
                    except asyncio.TimeoutError:
                        pass
                
                result = await run_task
                yield CompleteEvent(data=...).to_sse()
            
            finally:
                progress.unregister_handler(progress_handler)
        
        return StreamingResponse(event_generator(), media_type="text/event-stream")
```

**3 个关键设计点**：

1. **SSE (Server-Sent Events) streaming** —— 不是 WebSocket
   - 简单 HTTP，浏览器 EventSource API
   - 前端可逐步看到进度

2. **`progress.register_handler` 模式** —— v1 的 progress 系统的多端支持
   - 注册 handler 接收所有 agent 更新
   - 翻译成 SSE event yield 出去

3. **`wait_for_disconnect`** —— 检测客户端断连
   - 如果用户关掉浏览器，cancel 后台任务
   - 防止资源泄漏

**`/backtest` 端点**（line 170-336）：

**与 `/run` 类似但更长**：

- 用 `BacktestService`（不是直接调 graph）
- 多一个 `progress_callback` 处理 backtest 特定更新（day-by-day 结果）
- 结果包含 `performance_metrics` + `final_portfolio` + `total_days`

**两个端点都用 SSE streaming**——支持长时间运行 + 实时进度。

### Service 层

**6 个 service 模块**：

| Service | 职责 |
|---------|------|
| `agent_service.py` | 调用 `run_hedge_fund`（src/main.py） |
| `api_key_service.py` | 加解密 API key 存 DB |
| `backtest_service.py` | 包装 `BacktestEngine`（src/backtesting） |
| `graph.py` | 构造 LangGraph StateGraph |
| `ollama_service.py` | 异步检查 Ollama daemon 状态 |
| `portfolio.py` | Portfolio 状态操作 |

**`graph.py::create_graph`** 关键代码（推测）：

```python
def create_graph(graph_nodes, graph_edges):
    """从 React Flow JSON 构造 LangGraph StateGraph"""
    workflow = StateGraph(AgentState)
    
    # React Flow 节点 → LangGraph nodes
    for node in graph_nodes:
        node_id = node["id"]
        node_type = node["data"]["type"]  # analyst / risk / portfolio
        
        if node_type == "analyst":
            agent_name = node["data"]["agent_name"]
            agent_func = get_agent_func(agent_name)
            workflow.add_node(node_id, agent_func)
    
    # React Flow 边 → LangGraph edges
    for edge in graph_edges:
        workflow.add_edge(edge["source"], edge["target"])
    
    return workflow
```

**意义**：**前端 React Flow 画图，后端自动转 LangGraph**——前端可视化编辑 agent 拓扑。

## v1 app 整体架构图

```
React Flow (前端)
       │ (保存 flow 配置)
       ▼
HedgeFundFlow (DB)
       │
       │ (用户触发 run)
       ▼
routes/hedge_fund.py::/run
       │
       ├── ApiKeyService.get_api_keys_dict()
       │       │
       │       ▼
       │   ApiKey (DB, 加密的 key)
       │
       ├── create_graph(graph_nodes, graph_edges)
       │       │
       │       ▼
       │   LangGraph StateGraph
       │
       └── run_graph_async(graph)
               │
               ├── progress_handler 注册到 progress
               │
               ├── streaming SSE response
               │       │
               │       └── EventGenerator → EventSource (浏览器)
               │
               ▼
           HedgeFundFlowRun / FlowRunCycle 持久化
```

## v1 app 的亮点

### ✅ 1. **完整的 SaaS 架构**

- 多用户
- Flow 模板（`is_template=True`）
- 历史 run 记录
- 成本跟踪（llm_calls / api_calls / estimated_cost）

### ✅ 2. **SSE streaming**

- 实时进度反馈
- 断连检测
- 比 WebSocket 简单

### ✅ 3. **React Flow 集成**

- 前端可视化编辑 agent 拓扑
- JSON 存 DB
- 后端自动转 LangGraph

### ✅ 4. **3 表设计**（flow / run / cycle）

- 每个 cycle 都有完整快照
- 时间序列审计
- 可重跑

## v1 app 的问题

### ❌ 1. **API key 存明文**

```python
key_value = Column(Text, nullable=False)  # The actual API key (encrypted in production)
```

**注释说"encrypted in production"——但代码是明文**。生产部署会泄露所有用户 key。

### ❌ 2. **没有 v2 集成**

- 只支持 v1 LangGraph graph
- v2 alpha model 接口**没有接入**
- 这是 v1 app 的最大局限

### ❌ 3. **estimated_cost 字段是 String**

```python
estimated_cost = Column(String(20), nullable=True)
```

**为什么不是 Numeric / Float**？——可能是为了避免小数精度问题。

### ❌ 4. **没有 rate limit / abuse detection**

- 用户可以无限触发 run
- 会爆 LLM API 配额

## 对 fork → CN 版本的启示

### A. v2 完全可以重新设计 app 层

v1 app 是为 v1 LangGraph 设计的。CN fork 用 v2：

- ❌ 不要直接 fork app/
- ✅ 写新的 `app_v2/` 用 FastAPI + v2 alpha model 接口
- ✅ 可以借鉴 v1 的 SSE streaming + Flow/Run/Cycle 表设计
- ✅ 必须加密 API keys（v1 没做）

### B. v1 的 3 表设计值得保留

```
Flow (用户配置)
    │
    └──< Run (单次执行)
            │
            └──< Cycle (每天的状态)
```

**CN 化可以扩展**：

- Flow + Run 复用 v1 设计
- Cycle 可以记录**v2 的 alpha model signal + 仓位变化**（更结构化）

### C. SSE streaming 模式可以复用

`routes/hedge_fund.py` 的 SSE streaming 模式（progress_handler + asyncio.Queue + StreamingResponse）——**可以直接搬到 v2 app**。

## 深读问题（自检）

- [ ] `plot.py` 返回 `matplotlib.figure.Figure` 而不是直接 `plt.show()`——为什么？好处是？
- [ ] `routes/hedge_fund.py` 同时支持 `/run` 和 `/backtest`——这两个端点的代码高度相似，是否应该用 **factory pattern** 共享逻辑？
- [ ] `HedgeFundFlow.nodes` 和 `edges` 直接存 React Flow JSON——如果 React Flow API 升级，DB 里旧数据会"破"。**版本控制怎么办？**
- [ ] `ApiKey.key_value` 注释说"encrypted in production"——**什么时间用什么方式加密**？是 v1 的设计遗漏还是 TODO？

---

*解析时间：2026-07-12 · 第二十轮迭代*
*全部模块解析 100% 完成。共 31 篇 deep dive。*