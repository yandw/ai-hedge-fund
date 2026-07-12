# 模块索引（持续更新）

> 总览：本项目所有 Python 模块的清单、分类与深读状态。
> 配套：[STUDY_PLAN.md](../STUDY_PLAN.md) 是学习路径；本文是完成度追踪。
> 详细解析放在 [`./modules/`](./modules/) 目录下。

## 状态图例

- ✅ 已深读（有 `./modules/<file>.md`）
- ⏳ 待深读
- 🚧 占位/空文件（仅 README / docstring / 占位 stub）

---

## v2/ —— 核心引擎（29 个 .py 文件）

### 顶层入口

| 文件 | 状态 | 行数 | 作用一句话 |
|------|------|------|-----------|
| `v2/__init__.py` | ⏳ | | |
| `v2/models.py` | ✅ | 72 | 全流水线数据类型（Signal/QuantSignals/PortfolioTarget/TradeOrder/ExecutionResult） |
| `v2/analyze.py` | ⏳ | | |
| `v2/conftest.py` | ⏳ | | |

### v2/data/ —— 数据层

| 文件 | 状态 | 行数 | 作用 |
|------|------|------|------|
| `v2/data/__init__.py` | ⏳ | | |
| `v2/data/protocol.py` | ✅ | 89 | `DataClient` Protocol（结构化 typing，零继承） |
| `v2/data/models.py` | ✅ | 277 | FD API 响应 Pydantic 模型（60+ 字段） |
| `v2/data/client.py` | ✅ | 277 | `FDClient` 真实实现，fail-loud + filing_date 强制 |
| `v2/data/cached.py` | ✅ | 158 | `CachedDataClient` 磁盘缓存包装 |
| `v2/data/test_*.py` | ⏳ | | 3 个测试文件 |

### v2/llm/ —— LLM 层

| 文件 | 状态 | 作用 |
|------|------|------|
| `v2/llm/__init__.py` | ⏳ | |
| `v2/llm/client.py` | ✅ | `LLMClient` Protocol + `AnthropicLLM` + `extract_json` |
| `v2/llm/cache.py` | ✅ | `PromptCache`（缓存+审计+调试三合一） |

### v2/features/ —— Feature 层

| 文件 | 状态 | 作用 |
|------|------|------|
| `v2/features/snapshot.py` | ✅ | `FundamentalsSnapshot` + `build_snapshot` |
| `v2/features/test_snapshot.py` | ⏳ | |

### v2/signals/ —— Alpha 模型层

| 文件 | 状态 | 作用 |
|------|------|------|
| `v2/signals/base.py` | ✅ | `AlphaModel` / `QuantModel` 抽象 |
| `v2/signals/pead.py` | ✅ | PEAD 量化模型模板 |
| `v2/signals/llm_agent.py` | ✅ | LLM agent 基类 |
| `v2/signals/buffett.py` | ✅ | 巴菲特 persona（~50 行） |
| `v2/signals/test_signals.py` | ⏳ | |
| `v2/signals/test_llm_agents.py` | ⏳ | |

### v2/backtesting/ —— 回测引擎

| 文件 | 状态 | 作用 |
|------|------|------|
| `v2/backtesting/engine.py` | ⏳ | `BacktestEngine` 主引擎 |
| `v2/backtesting/models.py` | ⏳ | `Trade` / `PerformanceMetrics` |
| `v2/backtesting/__main__.py` | ⏳ | CLI 入口 |
| `v2/backtesting/test_backtest.py` | ⏳ | |

### v2/event_study/ —— 事件研究

| 文件 | 状态 | 作用 |
|------|------|------|
| `v2/event_study/engine.py` | ⏳ | 市场模型 CAR 引擎 |
| `v2/event_study/models.py` | ⏳ | |
| `v2/event_study/stats.py` | ⏳ | t-test / bootstrap |
| `v2/event_study/plot.py` | ⏳ | |
| `v2/event_study/__main__.py` | ⏳ | |
| `v2/event_study/test_event_study.py` | ⏳ | |

### v2/demo/ —— 演示

| 文件 | 状态 | 作用 |
|------|------|------|
| `v2/demo/backtest.py` | ⏳ | PEAD 回测 demo（rich 终端） |

### v2/pipeline/, v2/portfolio/, v2/risk/, v2/validation/ —— 规划中

| 文件 | 状态 | 备注 |
|------|------|------|
| `v2/pipeline/__init__.py` | 🚧 |  |
| `v2/pipeline/execution.py` | ⏳ | |
| `v2/portfolio/__init__.py` | 🚧 | |
| `v2/portfolio/optimizer.py` | ⏳ | |
| `v2/risk/__init__.py` | 🚧 | |
| `v2/risk/manager.py` | 🚧 | 已知是 6 行 docstring |
| `v2/validation/__init__.py` | 🚧 | 空目录 |

---

## src/ —— v1 LangGraph 应用（~50 个 .py 文件）

### 顶层入口

| 文件 | 状态 | 作用 |
|------|------|------|
| `src/__init__.py` | ⏳ | |
| `src/main.py` | ⏳ | LangGraph StateGraph 入口 |
| `src/backtester.py` | ⏳ | 回测 CLI |

### src/agents/ —— 19 个分析师

| 文件 | 状态 | 行数 | 风格 |
|------|------|------|------|
| `src/agents/warren_buffett.py` | ⏳ | 826 | 价值（最复杂） |
| `src/agents/charlie_munger.py` | ⏳ | | 价值 |
| `src/agents/ben_graham.py` | ⏳ | | 价值 |
| `src/agents/peter_lynch.py` | ⏳ | | 成长 |
| `src/agents/phil_fisher.py` | ⏳ | | 成长 |
| `src/agents/cathie_wood.py` | ⏳ | | 创新/颠覆 |
| `src/agents/michael_burry.py` | ⏳ | | 逆向 |
| `src/agents/bill_ackman.py` | ⏳ | | 激进 |
| `src/agents/stanley_druckenmiller.py` | ⏳ | | 宏观 |
| `src/agents/rakesh_jhunjhunwala.py` | ⏳ | | 印度（海外） |
| `src/agents/mohnish_pabrai.py` | ⏳ | | 价值 |
| `src/agents/nassim_taleb.py` | ⏳ | | 风险/反脆弱 |
| `src/agents/aswath_damodaran.py` | ⏳ | | 估值（学术） |
| `src/agents/fundamentals.py` | ⏳ | | 基本面（不挂 persona） |
| `src/agents/valuation.py` | ⏳ | | 量化估值 |
| `src/agents/technicals.py` | ⏳ | | 技术面 |
| `src/agents/sentiment.py` | ⏳ | | 情绪 |
| `src/agents/news_sentiment.py` | ⏳ | | 新闻情绪 |
| `src/agents/growth_agent.py` | ⏳ | | 增长 |
| `src/agents/risk_manager.py` | ⏳ | | 风险官（piecewise 函数） |
| `src/agents/portfolio_manager.py` | ⏳ | | 组合经理（最终决策） |

### src/backtesting/ —— v1 回测

| 文件 | 状态 | 作用 |
|------|------|------|
| `src/backtesting/engine.py` | ⏳ | BacktestEngine |
| `src/backtesting/controller.py` | ⏳ | |
| `src/backtesting/trader.py` | ⏳ | |
| `src/backtesting/portfolio.py` | ⏳ | |
| `src/backtesting/valuation.py` | ⏳ | |
| `src/backtesting/metrics.py` | ⏳ | |
| `src/backtesting/benchmarks.py` | ⏳ | |
| `src/backtesting/output.py` | ⏳ | |
| `src/backtesting/types.py` | ⏳ | |
| `src/backtesting/cli.py` | ⏳ | |

### src/data/, src/tools/, src/graph/, src/llm/, src/cli/, src/utils/

| 文件 | 状态 | 作用 |
|------|------|------|
| `src/data/cache.py` | ⏳ | v1 磁盘缓存 |
| `src/data/models.py` | ⏳ | v1 数据模型 |
| `src/tools/api.py` | ⏳ | v1 Financial Datasets 客户端（无 PIT 过滤） |
| `src/graph/state.py` | ⏳ | AgentState + reducer |
| `src/llm/models.py` | ⏳ | 13 家 provider 抽象 |
| `src/cli/input.py` | ⏳ | CLI 参数解析 |
| `src/utils/analysts.py` | ⏳ | 19 个 analyst 注册表 |
| `src/utils/llm.py` | ⏳ | call_llm 主入口 |
| `src/utils/progress.py` | ⏳ | rich 进度条 |
| `src/utils/visualize.py` | ⏳ | 状态图转 PNG |
| `src/utils/display.py` | ⏳ | 终端展示 |
| `src/utils/api_key.py` | ⏳ | API key 注入 |
| `src/utils/ollama.py` | ⏳ | Ollama 工具 |
| `src/utils/docker.py` | ⏳ | Docker 辅助 |

---

## tests/ —— 测试

| 文件 | 状态 |
|------|------|
| `tests/test_api_rate_limiting.py` | ⏳ |
| `tests/test_cache.py` | ⏳ |
| `tests/test_cli_ticker_alias.py` | ⏳ |
| `tests/backtesting/test_*.py` (6 个) | ⏳ |
| `tests/backtesting/integration/test_*.py` (3 个) | ⏳ |

---

## app/ —— v1 Web 应用（前端 + 后端）

### app/backend/

| 文件 | 状态 | 作用 |
|------|------|------|
| `app/backend/main.py` | ⏳ | FastAPI app |
| `app/backend/database/{connection,models}.py` | ⏳ | SQLAlchemy |
| `app/backend/alembic/env.py` + versions/ | ⏳ | 迁移 |
| `app/backend/models/{events,schemas}.py` | ⏳ | Pydantic schema |
| `app/backend/repositories/{api_key,flow,flow_run}_repository.py` | ⏳ | 数据访问 |
| `app/backend/routes/{api_keys,flow_runs,flows,health,hedge_fund,language_models,ollama,storage}.py` | ⏳ | 8 个 router |
| `app/backend/services/{agent,api_key,backtest,graph,ollama,portfolio}_service.py` | ⏳ | 业务逻辑 |

### app/frontend/ —— React/Vite

| 文件 | 状态 | 作用 |
|------|------|------|
| `app/frontend/src/` | ⏳ | 顶层结构扫读 |

---

## 完成度追踪

- **v2/**: 2/29 模块深读（~7%）
- **src/**: 0/40+ 模块深读（~0%）
- **tests/**: 0/9 文件深读（0%）
- **app/**: 0/15+ 文件深读（0%）

更新频率：每完成一个模块深读后更新此表。

---

*最后更新：2026-07-12 第一轮迭代*