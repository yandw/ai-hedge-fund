# AI Hedge Fund 项目学习计划

> 目的：在决定 fork → CN 化改造之前，**把项目吃透**，同时把周边知识（LLM agent / 量化基础 / 国内数据生态）建立起来。
> 方式：模块化 Phase，每个 Phase 独立可停；混读——你读完反馈，我给下一份推荐 + 重点。
> 时间：未定，按节奏走，每个 Phase 末尾我会帮你"收口"。

---

## 进度总览

- [ ] Phase 0：本会话已完成（环境 + 概览 + 深度解读）
- [ ] Phase 1：v2 逐行阅读
- [ ] Phase 2：v1 完整阅读
- [ ] Phase 3：app/ 略读（可选）
- [ ] Phase 4：周边知识并行
- [ ] Phase 5：决策 checkpoint

---

## Phase 0：已完成（复盘）

**本会话已经做的**：

- [x] Poetry + venv + .gitignore 配好
- [x] `.env` 配 `FINANCIAL_DATASETS_API_KEY`（账户余额 0，需付费或换源）
- [x] 项目结构与 v1/v2 并存关系（[CLAUDE.md](../CLAUDE.md)）
- [x] 深度解读报告（[DEEP_READ.md](./DEEP_READ.md)）

**复盘要做的（30 min）**：

- [ ] 通读 [DEEP_READ.md](./DEEP_READ.md)，对每节打 1-5 分"我理解了吗"——低于 4 分的就是 Phase 1-3 要补的地方
- [ ] 通读 [VISION.md](../VISION.md) 和 [ROADMAP.md](../ROADMAP.md) —— 这两份是项目设计者的意图宣言
- [ ] 通读 [CLAUDE.md](../CLAUDE.md) 顶部几节（语言风格 / 项目概览 / 常用命令）

**收口问题**（请用自己的话回答，写到 `docs/study-notes/phase0-reflection.md`）：

1. 这个项目的**真实定位**是什么？（不是 README 表层的"AI fund"）
2. v1 和 v2 的根本差别在哪？为什么作者要重写？
3. 我现在对 v2 的 4 个核心原则（point-in-time / LLM-only-views / one-pipeline-three-modes / one-AlphaModel-interface）能用自己的话讲清楚吗？

---

## Phase 1：v2 逐行阅读（5–10 小时）

### 目标

读完 v2/ 每一个 .py 文件（除 __init__.py 外），能回答：每个文件的**单一职责是什么、对外契约是什么、内部最值得讨论的 2-3 行代码是什么**。

### 阅读顺序（按依赖从内到外）

#### 1.1 数据契约（30 min）

- [ ] `v2/models.py` —— 全流水线数据类型
  - Signal / QuantSignals / PortfolioTarget / TradeOrder / ExecutionResult
  - 理解 `Signal.value ∈ [-1, +1]` 这个约束的意义
- [ ] `v2/data/models.py` —— Financial Datasets 响应的 Pydantic 模型
  - 重点：`filing_date`、`filing_datetime`、`source_type`（8-K/10-Q/10-K/20-F）

#### 1.2 数据层（1.5 小时）

- [ ] `v2/data/protocol.py` —— `DataClient` Protocol
  - 理解 `@runtime_checkable` 的意义
  - 列出 8 个方法，每个的语义和"为什么需要它"
- [ ] `v2/data/client.py` —— `FDClient` 实现
  - 重点：`FDClientError` 的 fail-loud 设计
  - 重点：`get_financial_metrics` 的 `filing_date_lte` 强制
  - 重点：404 vs 429 vs 5xx 的处理差异
- [ ] `v2/data/cached.py` —— `CachedDataClient`
  - 为什么只缓存成功响应？
  - `extra="ignore"` 跟这里的兼容性

#### 1.3 LLM 层（1 小时）

- [ ] `v2/llm/client.py` —— `LLMClient` Protocol + `AnthropicLLM` + `extract_json`
  - 重点：作者"刻意不用 langchain 结构化输出"的注释
  - `extract_json` 的三段降级解析（fenced → whole-string → first balanced）
- [ ] `v2/llm/cache.py` —— `PromptCache`
  - 一个文件三件事：缓存 / 审计 / 调试
  - `prompt_key` 用 `(agent, model, system, user)` 作 key 的好处

#### 1.4 Feature 层（30 min）

- [ ] `v2/features/snapshot.py` —— `FundamentalsSnapshot` + `build_snapshot`
  - 重点：`MIN_PERIODS = 4`（为什么 4？）
  - 重点：`content_hash` 作为 LLM cache key
  - 重点：`render()` 把派生指标算在 Python 里（不让 LLM 算术）

#### 1.5 Signal 层（1.5 小时）

- [ ] `v2/signals/base.py` —— `AlphaModel` / `QuantModel` 抽象
  - `predict(ticker, date, data_client) -> Signal` —— 三个参数为什么这样定
  - `QuantModel` 给的工具方法（哪些真用了？哪些是 YAGNI？）
- [ ] `v2/signals/pead.py` —— quant 模板
  - 重点：`_RETROSPECTIVE_CUTOFF_DAYS = 45` 和 `_SOURCE_PRIORITY` 为什么这么排
  - 怎么实现"按 filing_date 过滤 = point-in-time"
- [ ] `v2/signals/llm_agent.py` —— LLM agent 基类
  - 重点：失败语义分两层（数据失败 propagate，LLM 失败 abstain）
  - 重点：cache 命中时**不重发 LLM 请求**
- [ ] `v2/signals/buffett.py` —— persona 模板
  - 全文件 50 行而已，理解 "persona = 名字 + system prompt"

#### 1.6 Backtest 层（1 小时）

- [ ] `v2/backtesting/engine.py` —— `BacktestEngine`
  - 重点：`armed` 边沿触发器（只有模型回到中性才能再次开仓）
  - 仓位计算：`per_trade / entry_price` 等金额，**不考虑相关性/波动率**
  - 无摩擦（无交易成本、无滑点、无融资利率）
- [ ] `v2/backtesting/models.py` —— `Trade` / `PerformanceMetrics`
- [ ] `v2/backtesting/cli.py` / `__main__.py` —— 怎么从命令行调

#### 1.7 Event Study（1 小时）

- [ ] `v2/event_study/engine.py` —— 市场模型 CAR
  - 重点：estimation window `[-250, -11]` 为什么 -11 而不是 -1
  - 重点：`_filter_retrospective` —— GS 那个 103 天的真实例子
- [ ] `v2/event_study/stats.py` —— t-test / bootstrap CI
- [ ] `v2/event_study/models.py` —— 数据结构

#### 1.8 Demo + 入口（30 min）

- [ ] `v2/demo/backtest.py` —— 怎么驱动整个引擎
  - 重点：`START_DATE` / `END_DATE` 是写死的（pinned dates）
  - `rich.live` 终端展示
- [ ] `v2/analyze.py` —— 单只 ticker 观点 CLI

#### 1.9 测试（1 小时）

- [ ] `v2/data/test_client.py` + `test_client_contract.py` + `test_cached.py`
  - 重点：契约测试怎么写（任何 DataClient 都要满足的不变量）
- [ ] `v2/signals/test_signals.py` + `test_llm_agents.py`
- [ ] `v2/event_study/test_event_study.py`
- [ ] `v2/features/test_snapshot.py`
- [ ] `v2/backtesting/test_backtest.py`

### 收口问题（写到 `docs/study-notes/phase1-reflection.md`）

1. v2 的 `AlphaModel.predict()` 这个接口能加什么新东西而**不破坏现有调用**？给我举 3 个例子。
2. `LLMAgent` 的"失败分两层"设计，对应到 CN LLM（DeepSeek/Qwen）会有什么坑？
3. v2 的 BacktestEngine 的"无摩擦"假设，对 A 股 PEAD 影响大吗？（提示：A 股有 T+1、有印花税、有涨跌停）

### 完成后告诉我

读完 Phase 1 后告诉我：
- 你觉得 v2 哪些地方设计得好（值得借鉴）
- 哪些地方你觉得不够（想 fork 时改）
- 哪些地方你想先确认是 bug 还是 feature

我会基于你的反馈给 Phase 2 的具体调整。

---

## Phase 2：v1 完整阅读（3–5 小时）

### 目标

理解 v1 的 LangGraph agent 模式，**重点看它和 v2 在同一概念上的不同处理**——这是 fork 时最有价值的对比材料。

### 阅读顺序

#### 2.1 入口 + 状态（30 min）

- [ ] `src/main.py` —— LangGraph StateGraph 怎么搭
- [ ] `src/graph/state.py` —— `AgentState` 和 reducer
- [ ] `src/utils/analysts.py` —— 19 个 analyst 的注册表

#### 2.2 LLM 调用层（30 min）

- [ ] `src/utils/llm.py` —— `call_llm` 的 retry + structured output
  - 对比 `v2/llm/client.py::extract_json`，看哪个更鲁棒
- [ ] `src/llm/models.py` —— 13 家 provider 怎么抽象

#### 2.3 三个 representative agent（2 小时）

不需要全读 19 个。读这 3 个就够看出模式：

- [ ] `src/agents/warren_buffett.py` —— 最复杂（826 行），看 pre-computed score + LLM 决策
- [ ] `src/agents/valuation.py` —— 看 quant 派分析师怎么写
- [ ] `src/agents/sentiment.py` —— 看 sentiment 派怎么写

**收口**：用一句话总结这三个 agent 的"输出形状"是不是统一的？

#### 2.4 Risk + Portfolio Manager（1 小时）

- [ ] `src/agents/risk_manager.py` —— 那个 piecewise vol limit 函数
- [ ] `src/agents/portfolio_manager.py` —— LLM 直接决策 buy/sell 的位置

#### 2.5 Backtester（1 小时）

- [ ] `src/backtester.py` —— 入口
- [ ] `src/backtesting/engine.py` —— `BacktestEngine`
- [ ] `src/backtesting/types.py` —— 类型
- [ ] `src/backtesting/portfolio.py` + `trader.py` + `valuation.py` + `metrics.py` + `benchmarks.py`
- [ ] `src/backtesting/integration/` 下的 mock 和 integration tests

### 收口问题（写到 `docs/study-notes/phase2-reflection.md`）

1. v1 的 19 个 agent 用同一 prompt 调 LLM 多次，是效率浪费还是设计选择？
2. v1 的 risk_manager piecewise 函数，如果换成"用 LLM 决定仓位上限"会更糟还是更好？
3. v1 backtester 的 `_prefetch_data` 是 v2 commit 修的 lookahead 源头。具体哪一行？修了之后效果差多少？（提示：用 git blame 找 commit）

---

## Phase 3：app/ 略读（1–2 小时，可选）

**只在你想做 web UI 时读**。

### 阅读顺序

- [ ] `app/backend/main.py` —— FastAPI app + CORS + startup
- [ ] `app/backend/database/models.py` + `connection.py` —— SQLAlchemy
- [ ] `app/backend/repositories/` —— 数据访问层
- [ ] `app/backend/routes/` —— 8 个 router 各自干嘛（只读 docstring）
- [ ] `app/backend/services/` —— ollama_service 之类
- [ ] `app/frontend/src/` —— 顶层结构扫一遍，不深读组件

### 收口

如果 CN 化要 web UI，A 股和美股在前端展示上有什么需要本地化的？（如交易时间、货币符号、涨跌停颜色、板块分类）

---

## Phase 4：周边知识（并行，5–10 小时）

**混读模式**：你告诉我现在读到了 Phase 1/2/3 的哪一步，我根据你的状态给下一份阅读推荐 + 重点要解决的问题。

### 4.1 LLM Agent 架构基础（2 小时）

如果对 LangChain / LangGraph 不熟，先读：

- [ ] **LangGraph 概念**：<https://langchain-ai.github.io/langgraph/concepts/> —— 看 "Why LangGraph?" 和 "StateGraph"
- [ ] **LLM agent 的失败模式**：Anthropic 的 <https://docs.anthropic.com/en/docs/build-with-claude/agent-loop> —— 看 tool use 的错误处理

### 4.2 Quant 基础（2 小时）

- [ ] **Point-in-time 数据为什么重要**：搜 "lookahead bias backtest" 看 2-3 篇科普（QuantStart / Quantocracy 都行）
- [ ] **PEAD 文献**：Ball & Brown 1968 原文不必读，但搜一篇"Post-Earnings Announcement Drift 综述"博客
- [ ] **Sharpe / Sortino / Max Drawdown**：百科级别理解即可

### 4.3 CN 数据生态（3 小时，按顺序读）

| 顺序 | 资料 | 重点要回答 |
|------|------|-----------|
| 1 | Tushare Pro 官网文档的"数据接口"目录 | 哪些接口是免费的？积分怎么扣？有没有 ann_date 这种 point-in-time 字段？ |
| 2 | AKShare GitHub README + `stock_announcement_report_*` 章节 | 它和 Tushare 的字段差异在哪？数据来源稳定性？ |
| 3 | 巨潮资讯网 cninfo 主页 + 随便点一只股票 | 公告页面长什么样？爬虫拿什么字段？ |
| 4 | JoinQuant 官网"数据"页 | 它提供 API 还是有平台绑定？数据来源是它自己抓的还是聚合？ |
| 5 | 米筐 RiceQuant / 优矿 uqer 类似浏览 | 国内量化平台对比，心里有数 |

### 4.4 CN 法律环境（1 小时）

- [ ] 搜"非持牌投资咨询 处罚 2024" —— 了解国内监管口径
- [ ] 看证监会官网 "证券投资咨询业务" 的资质要求
- [ ] 看 VISION.md 里的 "stylized approximation" 免责声明，套到 CN 语境下够不够

### 收口

读完 4.3 后，请回答：

| 问题 | 答案要点 |
|------|---------|
| 哪个数据源的 **point-in-time** 能力最强？ | （CN 几乎没有 SEC EDGAR 那种严格的 filing_date，需要理解替代方案） |
| 哪个**性价比**最高（你 15,000 积分能跑多久）？ | |
| **法律风险**最大的是哪个组合（数据源 + LLM + persona）？ | |

---

## Phase 5：决策 checkpoint（1 小时）

读完 Phase 1-4 后，**重新评估 fork 方向**。

### 自检表

- [ ] 我能用一句话说清这个项目是干什么的
- [ ] 我能说清 v2 的 4 个核心原则
- [ ] 我能在不看代码的情况下，画出 v2 一次 PEAD 回测的数据流
- [ ] 我能列出 3 个 v2 的设计取舍（哪些是 feature，哪些可能是 bug）
- [ ] 我能说清 Tushare / AKShare / 巨潮的差异
- [ ] 我能估算 15,000 积分能跑多少次端到端 demo
- [ ] 我能说出国内用"段永平 AI"做投资建议的法律边界

### 决策产出

写 `docs/study-notes/phase5-decision.md`，包含：

1. **是否 fork**：是 / 否 / 部分 fork
2. **目标用户**：自己用 / 内部团队 / 公开仓库
3. **改造范围**：A / B / C（参考之前 brainstorm 的三档）
4. **数据源选择**：Tushare / AKShare / 巨潮 / JoinQuant
5. **LLM 选择**：DeepSeek / Qwen / Moonshot / 保留 Anthropic
6. **时间投入**：x 周，每天 y 小时
7. **风险评估**：列 3 个最可能踩的坑

完成后把这份给我看，我们再决定是否进入 writing-plans 阶段做实施 spec。

---

## 怎么用这个文档

- **每个 checkbox 是独立可勾的**，随时停下来都可以
- **每 Phase 末尾的"收口问题"必做**——能回答出来才算 Phase 真的过了
- **Phase 4 的具体阅读顺序是我给的初始建议**——你读的过程中觉得方向不对，告诉我，我帮你调
- **不要线性**——Phase 4 可以跟 Phase 1-3 并行做，比如你 Phase 1 读到一半想看 Tushare 文档，随时切过去

## 准备好后告诉我

读完任意 Phase 后 ping 我，我会基于你看到的细节给下一份推荐 + 重点要回答的问题。Phase 5 完成后我们正式进入 fork 决策。

---

*更新：2026-07-12 — v2.0 草稿；边读边调。*