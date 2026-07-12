# tests/* + app/* 深度解析（合并）

> 文件：`tests/`（9 个文件）+ `app/backend/`（~15 个文件）
> 地位：**v1 的测试套件 + v1 Web 应用的 FastAPI 后端**

## 第一部分：tests/ —— v1 测试套件

### 总览

```
tests/
├── __init__.py
├── test_api_rate_limiting.py            # 测试 API 限流
├── test_cache.py                        # 测试 src/data/cache.py
├── test_cli_ticker_alias.py             # 测试 CLI ticker 别名
├── backtesting/
│   ├── conftest.py                      # backtest fixtures
│   ├── test_controller.py               # AgentController
│   ├── test_execution.py                # TradeExecutor
│   ├── test_metrics.py                  # PerformanceMetricsCalculator
│   ├── test_portfolio.py                # Portfolio
│   ├── test_results.py                  # OutputBuilder
│   ├── test_valuation.py                # calculate_portfolio_value
│   └── integration/
│       ├── conftest.py
│       ├── mocks.py                     # MockConfigurableAgent
│       ├── test_integration_long_only.py
│       ├── test_integration_long_short.py
│       └── test_integration_short_only.py
└── fixtures/api/                        # API 响应样本
```

### 关键文件：`integration/mocks.py` —— MockConfigurableAgent

```python
class MockConfigurableAgent:
    """Mock agent that executes a predefined sequence of trading decisions."""
    
    def __init__(self, decision_sequence: list[dict], tickers: list[str]):
        self.decision_sequence = decision_sequence
        self.tickers = tickers
        self.call_count = 0
    
    def __call__(self, **kwargs) -> AgentOutput:
        tickers = kwargs.get("tickers", self.tickers)
        
        if self.call_count < len(self.decision_sequence):
            day_decisions = self.decision_sequence[self.call_count]
        else:
            day_decisions = {}
        
        self.call_count += 1
        
        decisions = {}
        for ticker in tickers:
            if ticker in day_decisions:
                decisions[ticker] = day_decisions[ticker]
            else:
                decisions[ticker] = {"action": "hold", "quantity": 0}
        
        return {"decisions": decisions, "analyst_signals": {}}
```

**Mock 智能设计**：

- **预定义决策序列** —— 测试可以"假装"agent 每天的决定
- **call_count 计数** —— 每次 invoke 返回下一个预设决策
- **default to hold** —— 任何 ticker 没显式说就是 hold

**对比 v2**：v2 没有 MockConfigurableAgent —— v2 tests 用 mock LLM 但 mock 的是 alpha model 直接。

### 3 个集成测试

| Test | 场景 |
|------|------|
| `test_integration_long_only.py` | 只做多 |
| `test_integration_long_short.py` | 多空都做 |
| `test_integration_short_only.py` | 只做空 |

**为什么 3 个独立集成测试**：

- ✅ **回归保护** —— long_only 的 bug 不会污染 long_short
- ✅ **行为差异** —— short 的 margin requirement、borrow cost 不同
- ✅ **覆盖率** —— 100% action combination

### 单元测试覆盖

| 测试文件 | 覆盖 |
|---------|------|
| `test_controller.py` | AgentController.run_agent() 的标准化逻辑 |
| `test_execution.py` | TradeExecutor.execute_trade() 4 个 action |
| `test_metrics.py` | 6 个 metrics 计算正确性 |
| `test_portfolio.py` | Portfolio.apply_long_buy/sell/short_open/short_cover |
| `test_results.py` | OutputBuilder 输出格式 |
| `test_valuation.py` | calculate_portfolio_value 准确性 |

**6 个单元测试覆盖 v1 backtest 的每个组件**——**v1 backtest 的可测试性其实很好**。

### v1 测试的不足

| 问题 | 表现 |
|------|------|
| **没有数据正确性测试** | 不测 `get_financial_metrics` 的 PIT 强制 |
| **没有 LLM 输出稳定性测试** | 不测 agent 输出在重复调用下的差异 |
| **没有对比测试** | 不测 v1 vs 其他策略的相对表现 |

## 第二部分：app/backend/ —— FastAPI 后端

### 总览

```
app/backend/
├── main.py                    # FastAPI app
├── alembic.ini                # 迁移配置
├── alembic/                   # 迁移文件
├── database/
│   ├── connection.py          # SQLAlchemy engine
│   └── models.py              # ORM models
├── models/
│   ├── events.py              # 事件 schema
│   └── schemas.py             # Pydantic schemas
├── repositories/              # 数据访问层
│   ├── api_key_repository.py
│   ├── flow_repository.py
│   └── flow_run_repository.py
├── routes/                    # 8 个 router
│   ├── api_keys.py
│   ├── flow_runs.py
│   ├── flows.py
│   ├── health.py
│   ├── hedge_fund.py
│   ├── language_models.py
│   ├── ollama.py
│   └── storage.py
└── services/                  # 业务逻辑
    ├── agent_service.py
    ├── api_key_service.py
    ├── backtest_service.py
    ├── graph.py
    ├── ollama_service.py
    └── portfolio.py
```

**结构**：典型 FastAPI 三层 — Routes → Services → Repositories → DB。

### `main.py` —— FastAPI 入口

```python
app = FastAPI(title="AI Hedge Fund API", description="Backend API for AI Hedge Fund", version="0.1.0")

Base.metadata.create_all(bind=engine)    # 自动建表

app.add_middleware(CORSMiddleware, allow_origins=["http://localhost:5173", ...])

app.include_router(api_router)

@app.on_event("startup")
async def startup_event():
    """Check Ollama availability."""
    status = await ollama_service.check_ollama_status()
    ...
```

**关键设计**：

- ✅ **CORS only 允许 frontend origin**（5173）—— 防止跨域攻击
- ✅ **auto-create_all** —— 启动时建表（开发友好，prod 应只用 alembic）
- ✅ **startup 事件** —— 检查 Ollama（可选依赖）

### 8 个路由

| Route | 端点 | 作用 |
|-------|------|------|
| `api_keys.py` | `/api-keys` | CRUD 用户的 API keys（FINANCIAL_DATASETS / OPENAI 等） |
| `flows.py` | `/flows` | CRUD "hedge fund flows"（一个 flow = 一次回测配置） |
| `flow_runs.py` | `/flow-runs` | 一次 flow 的实际运行记录 |
| `health.py` | `/health` | 健康检查 |
| `hedge_fund.py` | `/hedge-fund/run` | **核心端点** —— 触发 agent 跑 |
| `language_models.py` | `/language-models` | 列出可用 LLM |
| `ollama.py` | `/ollama/*` | Ollama 状态 / 模型管理 |
| `storage.py` | `/storage/*` | 文件存储（上传 / 下载） |

**核心数据模型**：

- **Flow** = 用户配置（哪些 tickers / 哪些 analysts / 哪个 model）
- **FlowRun** = 一次 Flow 的实际执行（agent signals / decisions / 时间戳）
- **APIKey** = 用户的 provider keys（加密存 DB）

### 6 个 service 层

| Service | 职责 |
|---------|------|
| `agent_service.py` | 调用 `run_hedge_fund`（src/main.py） |
| `api_key_service.py` | 加解密 API key 存 DB |
| `backtest_service.py` | 包装 `BacktestEngine`（src/backtesting） |
| `graph.py` | LangGraph StateGraph 构造 |
| `ollama_service.py` | 异步检测 Ollama daemon |
| `portfolio.py` | Portfolio 状态操作 |

### 典型请求流：`POST /hedge-fund/run`

```
Client → routes/hedge_fund.py
    ↓ 解析 request
    ↓ 拿用户 API keys (services/api_key_service)
    ↓ 构造 LangGraph workflow (services/graph)
    ↓ 调用 run_hedge_fund (services/agent_service)
    ↓ 跑 agent + LLM
    ↓ 记录 FlowRun (repositories/flow_run_repository)
    ↓ 返回 decisions + analyst_signals
```

### 关键设计决策

#### 1. **API keys 存 DB（加密）**

- ✅ 用户在前端填 key，存 DB（不是 .env）
- ✅ 多用户支持
- ❌ 加密 / 解密逻辑的复杂度
- ❌ **安全风险**：DB 被脱 = 所有 key 失陷

**对比 v2 CLI**：v2 用 `.env` —— 单用户，单机。app/ 是多用户 SaaS 风格。

#### 2. **Flow + FlowRun 分离**

- **Flow** = 模板（"我想跑 AAPL/MSFT，用 Buffett + Munger"）
- **FlowRun** = 一次执行（"Flow X 在 2024-06-01 跑了一次，结果如下"）

**好处**：

- ✅ 用户可以**重跑同一 Flow**（同配置不同时间）
- ✅ 历史 FlowRun 可对比
- ✅ 审计追溯

#### 3. **Alembic 迁移**

```bash
alembic/versions/
├── 1b1feba3d897_add_data_column_to_hedge_fund_flows.py
├── 2f8c5d9e4b1a_add_hedgefundflowrun_table.py
├── 3f9a6b7c8d2e_add_hedgefundflowruncycle_table.py
├── 5274886e5bee_add_hedgefundflow_table.py
└── add_api_keys_table.py
```

**5 个迁移**——表演进历史：

1. `add_api_keys_table` —— 创建 api_keys 表
2. `add_hedgefundflow_table` —— 创建 hedgefund_flows 表
3. `add_hedgefundflowrun_table` —— 创建 flow_run 表
4. `add_hedgefundflowruncycle_table` —— 创建 cycle 表（每次 backtest 的某一天？）
5. `add_data_column_to_hedge_fund_flows` —— 给 flow 表加 data 列（存 analyst_signals JSON）

**演进逻辑**：从单表 → 拆成 3 表（flow / run / cycle）→ 加 JSON 列。

### `app/frontend/` —— React/Vite

**没读源码**——但从目录结构推断：

- **package.json / pnpm-lock** —— 前端依赖
- **vite.config.ts** —— Vite 构建配置
- **tailwind.config.ts** —— Tailwind CSS
- **components.json** —— shadcn/ui 配置
- **.eslintrc.cjs** —— ESLint

**判断**：现代 React + Vite + Tailwind + shadcn/ui stack。

## v1 vs v2 的全景对比

| 维度 | v1 | v2 |
|------|----|----|
| 架构 | LangGraph StateGraph + fan-out | 单 pipeline + alpha model 接口 |
| Agent | 12 personas + 7 quant = 19 agents | 2 agents（PEAD + Buffett） |
| Web app | ✅ 完整 FastAPI + React | ❌ 未集成 |
| Tests | ✅ 9 文件 + 6 unit + 3 integration | ✅ 6 文件 + contract test |
| CI/CD | ❌ 无 | ❌ 无 |
| 数据层 | 内存 cache + 429 重试 | 磁盘 cache + 429 重试 + PIT 强制 |
| 回测 | 每天调 LLM（19 agent） | 一次性 PEAD + cache 命中 |
| 数据正确性 | ❌ `report_period_lte` | ✅ `filing_date_lte` |
| LLM 决定仓位 | ❌ 是 | ✅ 不（确定性代码） |
| Portfolio 模拟 | ✅ 完整账户 | ❌ 简单 Trade 列表 |
| Benchmark 对比 | ✅ S&P 500 | ❌ 未实现 |

## 全项目最终评估

### v1 的价值

- ✅ **可运行的 Web 应用**——多用户可用
- ✅ **完整 backtest harness**——可以测策略
- ✅ **19 个 agent 的多样性**——可以对比不同投资哲学
- ✅ **多 LLM provider 支持**——灵活

### v1 的根本问题

- ❌ **数据层 lookahead leak**（已 commit 修，但生产部署的还是老版本）
- ❌ **LLM 决定仓位**——回测数学不诚实
- ❌ **整体架构不适合严肃量化研究**——太多"装饰性"特性

### v2 的价值

- ✅ **Point-in-time 诚实**
- ✅ **接口设计清晰**（AlphaModel / DataClient / LLMClient）
- ✅ **LLM-only-views** 哲学
- ✅ **失败语义明确**（fail-loud + abstain）

### v2 的未完成

- ❌ Portfolio construction 占位
- ❌ Risk management 占位
- ❌ **Validation gate (CPCV/PBO) 完全缺**——最关键
- ❌ Web app 未集成

## 对 fork → CN 版本的最终建议

### A. fork v2（不是 v1）

CN fork **直接基于 v2**，**不要碰 v1**——v1 的代码即使能用，回测数学不诚实。

### B. 加的东西

1. **`v2/data/tushare_client.py`** —— 实现 DataClient Protocol
2. **`v2/llm/client.py::DeepSeekLLM`** —— DeepSeek / Qwen provider
3. **`v2/signals/cn_pead.py`** —— A 股 PEAD（T+1 + 涨跌停）
4. **`v2/signals/duan_yongping.py`** —— 中国 persona
5. **`v2/risk/manager.py`** —— 实现 v1 risk_manager 的 piecewise vol limit（用配置表）

### C. 优先级

| 优先级 | 任务 | 价值 |
|--------|------|------|
| 🔴 P0 | TushareClient + A 股 PEAD | 跑通端到端 |
| 🟡 P1 | Validation gate (CPCV/PBO) | 防 overfitting |
| 🟡 P1 | 1-2 个中国 persona | demo 价值 |
| 🟢 P2 | DeepSeek/Qwen LLM provider | 中文语境更准 |
| 🟢 P2 | v2/risk/manager.py 实现 | 风险控制 |
| 🔵 P3 | Web app (v2 风格 FastAPI + React) | 多用户 |

### D. 不要做的事

- ❌ 把 v1 的 19 个 agent 都 port 到 v2 —— **v2 "persona = name + system prompt" 已经够了**
- ❌ 把 v1 的 LangGraph fan-out 搬到 v2 —— **v2 是串行 pipeline，不需要 fan-out**
- ❌ 把 v1 的 Portfolio 类搬到 v2 —— **v2 的 Trade 列表足够**
- ❌ 改 `v2/data/protocol.py` —— **Protocol 是契约，CN 化只加实现**

## 全项目最终统计

**已产出 27 篇 deep dive**（本次会话累计）：

| 范围 | 篇数 |
|------|------|
| v2/ 全套 | 19 |
| src/ 核心（main + utils + graph） | 4 |
| src/agents（quant + personas + risk） | 2 |
| src/backtesting + data + cli + tools | 1 |
| tests + app | 1 |
| 之前会话（CLAUDE.md + DEEP_READ.md + STUDY_PLAN.md） | 3 |

**总计约 30+ 篇 markdown 文档**，存放于 `docs/` 和 `docs/study-notes/modules/`。

## 整个解析旅程的总结

通过这次 `/loop` 调度，我们：

1. **从顶到底摸清了** v2 整套架构（从 Protocol 接口到 BacktestEngine）
2. **对比了** v1 vs v2 的根本差异（LLM 决定 vs 不决定仓位、PIT 强制 vs 不强制）
3. **发现了** v1 的根本问题（数据 lookahead、过度工程）
4. **形成了** CN fork 的具体路径（TushareClient + A 股 PEAD + 1-2 persona）

**下一步**：决定 CN fork 走 Path A/B/C 中的哪条——之前 brainstorm 阶段你暂停了。读完后重新决定。

---

*解析时间：2026-07-12 · 第十八轮迭代（最终）*
*全项目深度解析完成。返回 [INDEX.md](./INDEX.md) 查看完成度。*