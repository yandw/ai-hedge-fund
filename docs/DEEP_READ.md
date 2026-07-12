# AI Hedge Fund 项目深度解读报告

> 解读对象：[virattt/ai-hedge-fund](https://github.com/virattt/ai-hedge-fund)（本地 `ai-hedge-fund`，当前 `pyproject.toml` 版本 `2026.7.10`，git tag 末端 `2026.7.3`）。
> 解读时间：2026-07-12。
> 解读立场：一个读过 v1 全量、v2 核心实现、VISION/ROADMAP 的工程师视角。

---

## 一句话总结

**这是一个从"网红 demo"认真转型为"可被认真对待的量化研究系统"的项目**——v1 是给 GitHub 看的明星展柜，v2 是给真实工程原则看的内核；二者目前在同一仓库里共存，谁也没删掉谁。

---

## 一、项目的真实定位

表层是 README 上的"AI-powered hedge fund / educational use only"——19 个 LLM 投资者 persona 一起讨论，然后给买卖建议。但读 VISION.md、ROADMAP.md、最近几条 git commit（`Fix lookahead leak: query metrics by filing_date, fail-loud client`、`Add VISION.md doc`、`Bump version to 2026.7.3`），就能看出**这是一次严肃的自我重写**：

- 维护者（virattt）已经把 v1 的 LangGraph 拼盘视作历史包袱；
- v2 不是"再加一个特性"，是把 v1 的根本假设推翻重来——**alpha 模型只形成观点，不下单；确定性代码才下单**；
- 维护者明确写下"如果不能在回测里诚实表达，就不上线"作为原则。

把它当"另一个 AI trading 玩具"是误读。它在做的事，本质上是一个**面向公开开源社区的、量化研究系统的样板工程**——一个社区可以往里加 alpha model、加数据源、加组合策略的开放骨架。LLM 投资者 persona 反而不是重点，重点是 `AlphaModel` 接口和 `Signal` 的契约。

---

## 二、v1 → v2 的转向：到底转了什么

| 维度 | v1（`src/`，已发布） | v2（`v2/`，开发中） |
|------|---------------------|---------------------|
| 抽象 | 19 个 LLM agent 节点（persona + system prompt） | 1 个 `AlphaModel` 接口，persona 只是"名字 + 系统提示" |
| 数据流 | `start_node → analysts → risk_manager → portfolio_manager` LangGraph | `data → alpha models → portfolio → risk → execution → ledger`（单管线，三模式） |
| 信号形状 | 各 agent 自由 dict（`{signal, confidence, reasoning}`） | 强类型 `Signal`（`value ∈ [-1,+1]`、reasoning、components、metadata） |
| 数据层 | `src/tools/api.py` + `src/data/cache.py`，**没有 filing_date 过滤** | `v2/data/protocol.py`（Protocol）+ `FDClient`（**强制 `filing_date_lte`**）+ `CachedDataClient`（fail-loud） |
| 数据失败语义 | 静默默认值（"no price data → vol = 0.05"） | 基础设施失败 `raise FDClientError`；"无数据"返回空 |
| 回测引擎 | LangGraph 跑每日循环（`src/backtesting/engine.py`） | `v2/backtesting/engine.py`：`alpha.predict(ticker, date, client)` + 简单持仓/退出 |
| 下单 | LLM 在 portfolio_manager 里**直接决策**（LLM 决定 `buy/sell/short/cover/quantity`） | LLM 只出观点；仓位规模、风险上限、执行——全是确定性代码 |
| 风控 | 人工 piecewise 函数（vol < 15% → 25%；vol 15-30% → 线性…） | **占位**（`v2/risk/manager.py` 是个 docstring） |
| 校验 | 无 | **占位**（`v2/validation/` 是空目录） |
| 测试 | `tests/` + `tests/backtesting/integration/` | `v2/**/test_*.py`，契约测试（`test_client_contract.py`） |
| Web | FastAPI + React/Vite，已发布 | 未接入 |

转向的**内核**只有一句：**"LLM 不碰交易"**。这一句话被反复写进 `v2/llm/client.py`、`v2/signals/llm_agent.py`、`v2/README.md`、`VISION.md`。

附带一项**更正历史错误**：最近一次 commit `Fix lookahead leak: query metrics by filing_date` 修了 v1 数据层一个隐蔽 bug——v1 的回测**在用报告期（report_period）当作"何时可见"，而不是 filing_date**。这等于在回测里偷看了未来。v2 把这件事作为不妥协原则写进数据层契约。

---

## 三、架构解读

### 3.1 v2 是真核心：单接口、双实现

`v2/signals/base.py` 的 `AlphaModel` 是整库的脊柱：

```python
class AlphaModel(ABC):
    @abstractmethod
    def predict(self, ticker, date, data_client) -> Signal: ...
```

两个具体子类共享同一接口：

- **`QuantModel`（纯数学）**——典型：`v2/signals/pead.py::PEADModel`。Post-Earnings Announcement Drift：基于 SEC 8-K/10-Q filing date 做 long BEAT / short MISS。**只用 `data_client.get_earnings_history()` 按 filing date 过滤**。
- **`LLMAgent`（LLM persona）**——典型：`v2/signals/buffett.py::BuffettAgent`。**关键观察**：persona 真的**只是 `name` 和 `get_system_prompt()`**。所有机制——prompt 构造、缓存、失败语义——都在 `LLMAgent` 基类里。维护者特意在基类 docstring 里写了一句话：

> "a persona is just a name + a system prompt"

这是非常优雅的取舍：**避免 v1 那种"每个 agent 600 行启发式"的膨胀**，同时让加新 persona 变成 30 行代码。

### 3.2 数据层：fail-loud + point-in-time 是设计原则

`v2/data/protocol.py` 是结构性 Protocol（`@runtime_checkable`），**任何带 `get_prices` / `get_financial_metrics` / `get_news` / ... 方法的类都是合法 DataClient**——无需继承。这给社区提供了天然的扩展点（"我要加一个 yfinance / 巨潮 / 港交所 client"）。

`v2/data/client.py::FDClient` 是这个协议的真实实现。它做了三件值得记住的事：

1. **404 → None（"无数据"是事实，不是失败）**；
2. **429 / 5xx / 网络异常 → `raise FDClientError`**（基础设施失败必须 raise，**禁止静默**）；
3. **`get_financial_metrics` 强制 `filing_date_lte=end_date`**（docstring 写明：report_period 比 filing 早 3-6 周，按 report_period 过滤就是 lookahead）。

`v2/data/cached.py` 把任意 `DataClient` 包一层磁盘缓存（`.v2_cache/data/`，gitignored），**只缓存成功响应**——失败的请求**不写入缓存**，下次仍然走网络。这保证了缓存不会"锁住"一次短暂的网络错误。

后果很实际：`v2/demo/backtest.py` 把所有 API 调用预热一遍（25 只 ticker × 价格 + 财报），然后**离线、无成本、确定性**地重放回测——`demo` 的 dates 是**写死的**（`START_DATE = "2023-07-01"`, `END_DATE = "2026-06-13"`），不是 `date.today()`，注释里明说："a moving end date would change the cache keys and the quoted stats between rehearsal and demo day"。这是**演示工程做对了的样子**。

### 3.3 LLM 层：刻意绕开 langchain 的结构化输出

`v2/llm/client.py::AnthropicLLM` 的 docstring 说得很直接：

> "We deliberately do NOT use langchain's structured-output machinery: its forced-tool mode breaks on Anthropic reasoning models (v1 carries the same workaround). We ask for JSON in the prompt and parse it ourselves."

`extract_json` 是个**三段降级解析器**：```json``` 代码块 → 整段 `json.loads` → 第一个平衡 `{...}`。这就是 LLM 工程里真正的"production code"——降级路径全在。

`v2/llm/cache.py` 一个文件同时是**缓存、审计记录、调试追踪**：

> "a cache: a backtest re-running an agent over an unchanged snapshot costs $0;
> the persistence record: the EXACT prompt + response behind every Signal, for replay and audit;
> the debug trail: failed parses keep the raw response on disk."

key 是 `(agent, model, system, user)` 的 SHA256——**模型换了，缓存自动失效**，这避免了"换了 GPT-5 还拿到 GPT-4 答案"的隐蔽错误。

### 3.4 事件研究：工程上有一处显式的"被打过"

`v2/event_study/engine.py::_filter_retrospective`（lines 369-391）：

```python
# Drop records where filing_date is >45 days after report_period.
# The ER extractor sometimes parses prior-period comparison data from
# a current 8-K, producing rows that look like a real Q4 event but are
# actually anchored on a Q1 filing date (e.g., GS: report_period=2025-12-31,
# filing_date=2026-04-13 → 103 days, clearly retrospective).
```

注释里直接写了 Goldman Sachs 的具体例子（filing 比 report 晚 103 天）——**这是被真实 bug 咬过、然后把补丁连同案例一起留在代码里**。这种注释比 README 里的承诺值钱 10 倍。

### 3.5 v1 的 backtester：能跑，但数学不诚实

`src/backtesting/engine.py::_prefetch_data` 拉了 `get_financial_metrics(ticker, end_date, limit=10)`——没有 `start_date`，更没有 `filing_date_lte`。结合 v1 数据层不做 PIT 过滤，这就是 v2 commit 修的同一类 leak 的源头。

**v1 回测出的"夏普 1.x"在数学上不等同于真实可交易策略的夏普**。这一点维护者知道，所以 v2 才是未来。但 v1 仍在 `src/backtester.py` 提供入口，仍然会有人跑。

---

## 四、写得好的地方（值得学习的）

1. **`FDClientError` 把"无数据"和"基础设施失败"做成语义级区分**——`raise` vs `return None`。这是真正的工程纪律。
2. **Protocol + 结构性 typing**——`DataClient`、`LLMClient` 都不强制继承，duck-typing 进系统，社区贡献零门槛。
3. **persona = 名字 + system prompt**——优雅地避免了"每个 agent 600 行启发式"的膨胀。
4. **prompt cache 同时是审计记录**——`v2/llm/cache.py` 的设计文档里把"缓存、审计、调试"三件事的合并说得很清楚。
5. **Pydantic 模型单源**——`v2/models.py` 5 个 model 就是流水线全部数据类型；`v2/data/models.py` 用 `extra="ignore"` 给上游 schema 演化留口。
6. **`FundamentalsSnapshot.render()` 把派生指标算在 Python 里**——ROE 平均、毛利趋势、BVPS CAGR——LLM 不再做算术，只做判断。
7. **PEAD 与 Buffett 共享 `predict()` 接口**——任何分析师的实现确实只是接口适配，仓库的核心承诺（"alpha model 只形成观点"）是真的。
8. **demo 的 pinned date**——`v2/demo/backtest.py` 注释明说"用固定日期保证 rehearsal 和 demo 数字一致"。这种细节显示作者做过真实演示。

---

## 五、技术债与隐患

### 5.1 v1 / v2 并存的真实成本

仓库里**有两套完整的数据层、缓存、backtester、测试、文档、demo**：

- `src/data/` vs `v2/data/`
- `src/tools/api.py` vs `v2/data/client.py`
- `src/backtesting/` vs `v2/backtesting/`
- `tests/` vs `v2/**/test_*.py`

`pyproject.toml` 把 `src`、`v2`、`app` 三个都打成 package。新人第一次进仓库，要花半天判断"哪个是真的"。**没有 README 段落告诉贡献者"新人请直接走 v2"**——CLAUDE.md 现在补上了，但仓库其他位置没说。

### 5.2 v1 的 Buffett agent 是过度工程的样本

`src/agents/warren_buffett.py` 共 826 行。前 500 行手算了 ROE 评分、债务评分、运营利润率评分、毛利稳定性评分、管理层评分（看股票回购 vs 增发）、三阶段 DCF、内在价值、margin of safety。**然后把这些塞进 prompt，问 LLM "你怎么看？"**

两个问题：

1. **LLM 用这些 pre-computed score 的能力远不如直接给它原始数字**——你让它综合 7 个打分，不如让它直接读 P/E、ROE、自由现金流。
2. **所有这些数字的获取都没经过 filing_date 过滤**——v1 数据层的问题。所有这些 score 都隐含 lookahead。

这正是 v2 要清理的东西。

### 5.3 v1 risk_manager 是配置伪装的代码

`src/agents/risk_manager.py::calculate_volatility_adjusted_limit`（lines 270-298）：

```python
if annualized_volatility < 0.15:    vol_multiplier = 1.25    # 25%
elif annualized_volatility < 0.30:  vol_multiplier = 1.0 - (vol - 0.15) * 0.5
elif annualized_volatility < 0.50:  vol_multiplier = 0.75 - (vol - 0.30) * 0.5
else:                                vol_multiplier = 0.50    # 10%
vol_multiplier = max(0.25, min(1.25, vol_multiplier))
```

这东西要么该是个 YAML/JSON 配置表，要么该是个有原则的公式（如 inverse-vol、risk-parity、fractional Kelly）。**把它写成代码，让它看起来像有道理，是最糟的中间态**——既不可改，又不可验证。

### 5.4 `black` 的 line-length = 420

```toml
[tool.black]
line-length = 420
```

420 是"什么都行"的意思。要么 88/100（PEP 8 友好），要么不配。420 等于告诉所有 reviewer"我不 care"。CI 没装（仓库没有 GitHub Actions），lint 也没人跑——意味着 PR 进来没法判断风格。

### 5.5 占位文件假装存在

`v2/risk/manager.py` 是 6 行 docstring；`v2/validation/__init__.py` 是空目录。ROADMAP 也老实说"⬜ Planned"。但 README 的架构图把 risk / validation / portfolio 都画出来了——**新人找过去发现是空的，会困惑**。

### 5.6 没有 CI、没有 pre-commit

测试要靠人手动跑：

```bash
poetry run pytest v2/
poetry run pytest tests/
```

仓库无 GitHub Actions、无 tox、无 pre-commit 钩子。**v2 的契约测试（`test_client_contract.py`）是项目里最值钱的东西之一，但没人自动跑它**。

### 5.7 版本号不一致

- `pyproject.toml`: `version = "2026.7.10"`
- git tag / 最近 commit message: `2026.7.3`

CalVer 没问题，但不一致是 sloppy。

### 5.8 LLM 投资者 persona 的法律风险（轻度）

VISION 明说 "stylized approximations — not the actual individuals, and not endorsements"。这是必要的护盾。但当一个面向公众的系统持续以"Warren Buffett"、"Charlie Munger"等名字输出投资建议，**仍然可能在某些司法管辖区触及商标 / 名人形象权**。当前阶段（educational, not investment advice）应该是安全的；任何向商业化迈步的动作都需要真正的法律意见。

### 5.9 v2 `BacktestEngine` 仓位逻辑过于简单

`v2/backtesting/engine.py`：

- 等金额 `per_trade / entry_price` 仓位（不考虑现有持仓、相关性、波动率）；
- `holding_days` 固定 5 天（不可参数化 per-signal）；
- `armed` 边沿触发器——只有"模型回到中性"才允许再次开仓。这是个奇怪的约束；
- 不算交易成本、不算滑点、不算融资利率。

作者在 engine docstring 里承认了：

> "These mechanics are intentionally simple for now … Week 8 portfolio construction will replace this harness."

但这个 Week 8 还没到。在它到之前，**任何 v2 backtest 出的 Sharpe 都是过度乐观的**——和 v1 一样，只是乐观的方向不同（v1 是 lookahead，v2 是无摩擦）。

### 5.10 `_compute_rsi` 在 `QuantModel` 里但没人用

`v2/signals/base.py` 给 `QuantModel` 提供了 `_compute_rsi`、`_sigmoid`、`_percentile_rank`、`_safe_float`、`_normalize_to_signal` 等一堆工具方法。**只有 `_safe_float` 在 `pead.py` 里被实际使用**。其他都是"以后用得上"的脚手架——YAGNI 反对的典型。

---

## 六、哲学 / 价值观观察

把 README、VISION、ROADMAP 和代码注释合起来读，能看到维护者的几条隐含原则：

1. **诚实 > 漂亮**——修 lookahead leak 是个不声张的工作；v2 README 明说 "Fail loud"；`FDClientError` 的注释直接写"a silent empty would poison a backtest"。这种话不是从训练数据里学来的，是被现实教训过的。
2. **视图与机制分离**——`AlphaModel` 只出 `Signal`，不让它决定仓位。这是把"alpha 研究员"和"交易台"分开——和真实对冲基金的组织结构对应。
3. **展示工程也属于工程**——demo 的 pinned dates、rich 终端的 equity curve、replay 节奏，都是为了让数字在演示时**可复现、可讲述**。
4. **接口是贡献表面**——`DataClient`、`LLMClient`、`AlphaModel` 都设计成社区可扩展。README 和 ROADMAP 都明示"新人最容易的贡献就是新加一个 alpha model"。
5. **教育性 > 商业性**——README 顶部免责声明；VISION 重申"educational use only"；没有 broker integration、没有 PBO 闸门之前不会让你跑真钱。

---

## 七、对贡献者的实际建议

如果你打算往这个仓库交代码：

| 想做什么 | 走哪条路 |
|---------|---------|
| 加一个新投资者 persona（Graham, Lynch, Burry …） | v2 的 `v2/signals/<persona>.py`，继承 `LLMAgent`，写 `name` + `get_system_prompt()`。**别走 v1。** |
| 加一个 quant alpha（momentum, mean-reversion, regime） | v2 的 `v2/signals/<model>.py`，继承 `QuantModel`，写 `name` + `predict()` |
| 加一个数据源（yfinance、巨潮、IBKR 历史） | 实现 `DataClient` Protocol 的方法即可，注册到 `v2/data/__init__.py` |
| 改 v1 的 Web 应用 | 直接改 `app/`，知道它的回测数学不诚实即可 |
| 改 backtest engine | v2 的 `v2/backtesting/`；小心 `_trade_ticker` 的简单仓位逻辑 |
| 加 portfolio construction / risk / validation | 直接动 `v2/portfolio/`、`v2/risk/`、`v2/validation/`——它们是空的，你想填什么就填什么 |
| 加测试 | `v2/<module>/test_*.py`，参照 `v2/data/test_client_contract.py` 的契约测试风格 |

**永远不要**：

- 改 v1 数据层加 filing_date 过滤——这是大动作，应该单独开 PR 并修所有 v1 backtest 报告的数字；
- 在 `v2/risk/` 之外的地方写风控逻辑——除非你想做 ADR（架构决策记录）；
- 在没跑过 `poetry run pytest v2/` 和 `poetry run pytest tests/` 之前提 PR。

---

## 八、这个项目真正在做什么

最后一层观察：**这个项目的真正产品不是那个 fund，是一套"如何严肃地做 AI 量化"的样板**。

- 数据层的 fail-loud + point-in-time 契约
- alpha model 接口与 Signal 类型
- LLM 输出观点、确定性代码下单的纪律
- persona = 名字 + system prompt 的解耦
- prompt cache = 审计记录的设计
- 事件研究的 retrospective filter
- demo 的 pinned dates 保证可复现

这些单独看都不稀奇，**合起来是一套不常见的工程纪律**。大多数"AI trading"仓库到"接一个 LangChain、拼 19 个 prompt、跑一下"就停了；这个仓库在认真解决"回测不撒谎"的问题。

**风险**：维护者单人维护、v1/v2 双线、社区贡献入口分散、CI 缺位、没有真实资金链路。如果作者精力转移或兴趣转移，v2 完工度可能停在现状。

**机会**：validation gate（CPCV/PBO）一旦落地，配上 point-in-time 数据层和 alpha model 接口，这会是 GitHub 上**最值得 fork 的开源量化研究骨架之一**。

---

*skipped: 不再展开 broker 协议、allocator、scheduler——它们都还没代码，只在 ROADMAP；待它们真的出现时再补。*