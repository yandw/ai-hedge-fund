# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **仅用于教育用途。** 本项目用于 AI/量化交易研究，**不构成投资建议**，也不应用于真实交易。详见 [README.md](./README.md)、[VISION.md](./VISION.md)、[ROADMAP.md](./ROADMAP.md)。

## 语言风格

- **始终使用中文回复用户**，包括所有解释、注释和沟通内容
- 代码中的标识符（变量名、函数名、类名、文件名）保持英文
- 技术术语（如 API、SDK、HTTP、JSON、LangGraph、Pydantic 等）在中文文本中保留英文原样
- 路径、命令、配置项一律英文，不翻译

---

## 项目概览

本仓库**同时存在两套系统**，编辑前请先判断目标在哪一边：

- **`src/` + `app/`** — **v1**，已发布。基于 LangGraph 的 agent 图 + FastAPI/React Web 应用；正向 v2 收敛。
- **`v2/`** — **v2**，开发中。围绕 `AlphaModel` + `Signal` 重建，point-in-time 诚实，目标是一条 `run_cycle` 流水线。

最终目标（见 ROADMAP）是**一条流水线**（`run_cycle`）以 backtest / paper / live 三种模式运行；v2 在建，v1 是当前在跑的对外形态。新增 analyst 工作请走 v2 的 `AlphaModel` 接口，除非明确在改已发布的应用。

## 常用命令

依赖（Poetry，Python ≥3.11）：

```bash
poetry install
cp .env.example .env   # 然后填 FINANCIAL_DATASETS_API_KEY + 至少一个 LLM key
```

### v1 — LangGraph 应用（已发布）

```bash
# CLI 跑一次（所有 analyst → risk → portfolio manager）
poetry run python src/main.py --ticker AAPL,MSFT,NVDA
poetry run python src/main.py --ticker AAPL,MSFT,NVDA --start-date 2024-01-01 --end-date 2024-03-01
poetry run python src/main.py --ticker AAPL,MSFT,NVDA --ollama

# v1 回测（在日期区间内循环 run_hedge_fund；输出指标 + PNG）
poetry run python src/backtester.py --ticker AAPL,MSFT,NVDA

# Web 应用（后端 :8000，前端 :5173）
cd app && bash run.sh
```

### v2 — alpha-model 引擎（开发中）

```bash
poetry run python -m v2.demo.backtest              # PEAD 跨 25 只股票，约 20s
poetry run python -m v2.demo.backtest --refresh    # 重建数据缓存
poetry run python -m v2.analyze NVDA               # analyst 对单只 ticker 的观点
poetry run python -m v2.analyze NVDA --date 2024-06-01 --agent pead
```

### 测试

```bash
poetry run pytest v2/          # v2 单元 + 契约测试
poetry run pytest tests/       # v1 测试（CLI alias、缓存、回测指标/执行等）
poetry run pytest tests/backtesting/integration/   # long-only / short-only / long-short 集成
```

### Lint / Format

`pyproject.toml` 已配置 `black`（line-length 420 — 是的，非常宽）、`isort`（black profile）、`flake8`。仓库**没有提交** `make` / `tox` / CI 配置，需要手动运行。

## 架构

### v1 — LangGraph（`src/`）

- 入口：`src/main.py`（CLI）、`src/backtester.py`（在日期区间内循环 agent）。
- 图：`src/graph/state.py` 定义 `AgentState`（`messages` + `data` + `metadata`）。`src/main.py::create_workflow` 构建一个 `StateGraph`：`start_node` → 选中的 analysts → `risk_management_agent` → `portfolio_manager`。
- Analyst 注册表：`src/utils/analysts.py::ANALYST_CONFIG` — 19 个 analyst 节点的**唯一真源**（`src/agents/*.py`）。每个 agent 是一个节点函数，往 `state["data"]["analyst_signals"]` 里写信号。
- 数据：`src/data/cache.py`（磁盘缓存） + `src/tools/api.py`（Financial Datasets REST）。
- 回测 harness：`src/backtesting/engine.py`（`BacktestEngine`）、`controller.py`、`trader.py`、`portfolio.py`、`metrics.py`、`valuation.py`、`benchmarks.py`、`output.py`。CLI 包装在 `src/backtesting/cli.py`（同时以 `backtester` poetry script 暴露）。
- Web 应用：FastAPI（`app/backend/main.py`） + React/Vite（`app/frontend/`），用 alembic 迁移 + SQLAlchemy 模型（`app/backend/database/`、`app/backend/models/`、`app/backend/repositories/`）。

### v2 — alpha-model 引擎（`v2/`）

唯一抽象：**`AlphaModel`** 形成观点，返回 **`Signal`**（`[-1, +1]` 的 conviction + 论据）。Quant 和 LLM analyst 共用同一接口。

```text
data → alpha models → portfolio → risk → execution → ledger   (= run_cycle，三种模式)
```

关键契约（详见 [v2/README.md](./v2/README.md) 与源码）：

- `v2/models.py` — `Signal`、`QuantSignals`、`PortfolioTarget`、`TradeOrder`、`ExecutionResult`（Pydantic，唯一真源）。
- `v2/signals/base.py` — `AlphaModel` ABC；`QuantModel` 子类提供共享工具（`_safe_float`、`_percentile_rank`、`_normalize_to_signal`、`_sigmoid`、`_compute_rsi`）。
- `v2/data/protocol.py` — `DataClient` 协议（结构化类型，`@runtime_checkable`）。方法：`get_prices`、`get_financial_metrics`（必须按 `end_date` point-in-time）、`get_news`、`get_insider_trades`、`get_company_facts`、`get_earnings`、`get_earnings_history`、`get_market_cap`。
- `v2/data/client.py` — Financial Datasets REST 客户端（fail-loud）。
- `v2/data/cached.py` — 给任意 `DataClient` 包一层磁盘缓存（缓存目录 `.v2_cache/`，已 gitignore）。
- `v2/llm/client.py`、`v2/llm/cache.py` — LLM provider 协议 + prompt 缓存。
- `v2/features/snapshot.py` — point-in-time 基本面快照。
- `v2/backtesting/` — 在 alpha model 观点之上的回测引擎（入口：`python -m v2.backtesting`）。
- `v2/event_study/` — 市场模型异常收益（CAR）。
- `v2/demo/backtest.py` — 跑真实引擎的展示 demo。
- `v2/pipeline/` — `run_cycle`（规划中，见 ROADMAP）。
- `v2/portfolio/`、`v2/risk/`、`v2/validation/` — 占位；CPCV/PBO 验证闸门是引擎下一个硬依赖。

## 项目原则（来自 VISION.md，v2 强制执行）

1. **Point-in-time 诚实。** 在任何模拟日期上，只能看到当时确实已公开的数据。按 `filing_date` 过滤，**绝不**按 `report_period`。`DataClient.get_financial_metrics` 契约强制此规则。数据层在失败时若静默返回空，会污染回测（缺失数据 = 假"无信号"）— **基础设施失败必须 raise**。
2. **LLM 不碰交易。** LLM 只形成*观点*和*叙述*。仓位规模和下单由确定性代码完成；风控限额是硬闸门，agent 不可突破。
3. **回测即上线。** 一条流水线、三种模式（`BACKTEST` / `PAPER` / `LIVE`），变的只有时钟和券商。
4. **一个接口适配所有 analyst。** 实现 `AlphaModel.predict(ticker, date, data_client) -> Signal` 即可接入引擎。模板见 `v2/signals/pead.py`（quant）和 `v2/signals/buffett.py`（LLM persona）。

## 值得记一笔的约定

- 磁盘缓存在 `.v2_cache/`（已 gitignore）— 跑过一次后即可离线、零成本复用。
- `backtester = "src.backtesting.cli:main"` 作为 poetry script 暴露。
- 图 PNG 输出（`*.png`）已 gitignore — 由 `src/utils/visualize.py::save_graph_as_png` 产生。
- 数据库文件（`*.db`、`*.sqlite*`）已 gitignore — 用户自己跑本地库；保留 `app/backend/alembic/` 的迁移文件。
- v1 的 `src/data/cache.py` API 响应缓存键在项目树下（重抓前先查）。
- 仓库**没有** Cursor 规则（`.cursor/`）或 Copilot 规则（`.github/copilot-instructions.md`）。

---

## 编码与工作流规范

### 1. Read Before You Write

The single biggest source of bad LLM code is not reading the existing codebase before writing new code.

Before writing anything:

- Read the files you're about to modify. Not skim. Read.
- Look at how similar things are done elsewhere in the project. If there's a pattern for API routes, follow that pattern. If there's a utility function that does half of what you need, use it.
- Check the imports at the top of the file. They tell you what libraries this project actually uses. Don't introduce axios if the project uses fetch everywhere.
- Look at the test files. They tell you what the expected behavior actually is.

If you're not sure how something is done in this project, say so. "I don't see a pattern for X in the codebase" is always better than guessing.

### 2. Think Before You Code

Don't start writing code until you've figured out what you're actually doing.

- **State your assumptions.** "I'm assuming you want JWT-based auth. If not, let me know." If you're wrong, you've lost 10 seconds. If you silently guess wrong, you've lost an hour.
- **Name the tradeoffs.** Caching trades memory for speed and introduces cache invalidation. Better to know before you write 200 lines.
- **If multiple approaches exist, present them briefly.** Two, maybe three. With a recommendation.
- **If something is confusing, stop.** Don't fill confusion with plausible-sounding code. Ask.

### 3. Simplicity

Write the minimum amount of code that solves the problem. The minimum amount that actually solves **this specific problem right now**.

- **Premature abstraction.** Strategy pattern for one email. Don't. Write `send_welcome_email(user)`.
- **Speculative error handling.** Wrap-in-try-catch for errors that can't happen. Every line of error handling is a line someone has to read.
- **Unnecessary configurability.** Parameters for values that will never change. Hardcode until there's a real reason not to.
- **Dead flexibility.** Interfaces with one implementation. Abstract base classes with one child. Wait for the second implementation.

The test: show your code to someone unfamiliar with the project. If they have to ask "why is this abstracted like this?" and the answer is "in case we need to..." — you've over-engineered it.

### 4. Surgical Changes

Your diff should be as small as possible.

- **Don't touch what you weren't asked to touch.** Fixing a bug in function A is not license to rename function B.
- **Match the existing style.** Single quotes → single quotes. `snake_case` → `snake_case`. Consistency within a file beats personal preference.
- **Clean up after yourself, not after others.** If YOUR change made an import unused, remove it. Pre-existing dead code is not your problem.
- **Don't reformat.** Don't run prettier on a file that wasn't formatted with prettier. Reformatting creates massive diffs that hide real changes.

The test: look at your diff. Can you justify every changed line with a direct connection to what was asked?

### 5. Verification

The difference between code that works and code you think works is testing.

- **Write the test first when fixing bugs.** Reproduce the bug first. Watch it fail. Fix it. Watch it pass.
- **Run existing tests before and after.** If tests passed before and fail after, you broke something. If tests were already failing before your change, say so — don't let your changes get blamed for pre-existing breakage.
- **Don't write tests for the sake of writing tests.** Test behavior, not implementation. Test the interesting cases.
- **If you can't write a test, say why.** "The DB calls are tightly coupled to business logic" is a signal something needs restructuring.

### 6. Goal-Driven Execution

Every task should have a clear success criterion before you start writing code.

- "Add validation" → "reject missing/invalid email, return 400 with a message, add tests for both cases"
- "Fix the bug" → "write a test that reproduces, make it pass, verify existing tests still pass"
- "Improve performance" → "profile, identify the bottleneck, fix that specific thing, measure again"

For anything multi-step, state the plan before executing. The plan lets the user catch mistakes in approach before you waste time implementing them.

### 7. Debugging

When something doesn't work, don't guess. Investigate.

- **Read the error message.** The whole thing. Including the stack trace. A TypeError could mean a hundred things.
- **Reproduce first.** If you can't reproduce it, you can't verify your fix.
- **Change one thing at a time.** If you change three things and the bug goes away, you don't know which change fixed it.
- **Don't add workarounds without understanding the root cause.** If a value is unexpectedly null, figure out **why**. The null check prevents a crash; the underlying bug is still there.
- **If you're stuck, say so.** "I've tried X and Y, neither worked, here's what I'm seeing, I think it might be Z but I'm not sure" — infinitely more useful than 20 silent iterations.

### 8. Dependencies

Every dependency you add is code you don't control that becomes permanent.

Before adding a package:

- Can you do this with what's already in the project?
- Can you do this with the standard library? You don't need lodash for `Array.prototype.map`. You don't need uuid if `crypto.randomUUID()` exists.
- Is this dependency actually maintained?
- How big is it?

When you do add a dependency, say why. Silently adding packages is not OK.

### 9. Communication

How you communicate about code matters as much as the code itself.

- **Say what you did and why.** "I moved the validation logic into a separate function because it was duplicated in three endpoints" — now the user understands without reading every line.
- **Flag concerns.** "This works but makes a DB call per item. If the list grows, this will be slow. Want me to batch it?"
- **Be precise about uncertainty.** "I'm not sure if this library supports streaming" is useful. "I think this should work" is not.
- **Don't explain things the user already knows.**
- **Commit messages matter.** "Fix null pointer in user lookup when email contains uppercase chars" > "Fix bug".

### 10. Common Failure Modes

Patterns to catch yourself doing:

- **The Kitchen Sink.** Asked to add one feature, restructured half the codebase. Don't. Do the one thing.
- **The Wrong Abstraction.** Generic solution to a problem that exists in one place. Duplication is far cheaper than the wrong abstraction. Copy-paste twice before you abstract.
- **The Invisible Decision.** Architectural choice (DB schema, API shape, auth strategy) made without flagging it. Hard to reverse; the user should know.
- **The Optimistic Path.** Handles the happy path, crashes on everything else. Think: API returns 500. File doesn't exist. User submits an empty form.
- **The Knowledge Hallucination.** Confidently uses an API that doesn't exist, a removed parameter, an imagined feature. If you're not 100% sure, check the docs / source.
- **The Style Drift.** Writes in "preferred" style instead of matching the project. Functional in OOP, classes in functional, TS in JS. Match the codebase.
- **The Runaway Refactor.** Fixing one thing touches another, which touches another. Twenty minutes, 15 files, original goal forgotten. If a fix is cascading, stop and get buy-in.

---

## 项目管理

### 文件组织规范

- **保持整洁**：尽量不要随便创建单独的文件，保持文件夹结构清晰
- **文档管理**：
  - 操作过程、分析结果、生成的报告统一放到 `docs/` 目录下（本项目尚无 `docs/`，按需创建）
  - 保持项目文档化，重要操作需要记录

### 版本控制规范

- **代码版本保护**：重大修改和变更必须提交 git 仓库
- **提交时机**：
  - 完成一个功能模块后
  - 进行重要的代码重构时
  - 修复关键 bug 后
  - 任何可能影响系统稳定性的变更
- **提交原则**：确保每次提交都是有意义的变更单元，便于回溯和追踪

---

## 贡献方向（按 ROADMAP）

最高杠杆的贡献是**新增 analyst**。从 [ROADMAP.md](./ROADMAP.md) 选一个未勾选项，实现 `AlphaModel.predict(...)` 返回 `Signal`，放到 `v2/signals/<your_model>.py`，在 `v2/signals/` 加一个测试，即可在回测器里跑起来，无需改动其他代码。同样形态适用于策略、allocator（CIO）、数据连接器、验证组件。
