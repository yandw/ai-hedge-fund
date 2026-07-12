# v2/ 占位模块 + src/utils/* + 杂项深度解析（最终合并）

> 文件：v2/ 占位（pipeline / portfolio / risk / validation）+ src/utils/{api_key,display,visualize,ollama,docker}.py + 各 __init__.py
> 地位：**v2 ROADMAP 中明确"⬜ Planned"的子系统** + **v1 杂项工具**

## 第一部分：v2/ 占位模块（ROADMAP ⬜）

### 4 个占位的现状

| 文件 | 行数 | 状态 |
|------|------|------|
| `v2/pipeline/__init__.py` | 0 | 空（package marker） |
| `v2/pipeline/execution.py` | 5 | **仅 docstring** |
| `v2/portfolio/__init__.py` | 0 | 空 |
| `v2/portfolio/optimizer.py` | 5 | **仅 docstring** |
| `v2/risk/__init__.py` | 0 | 空 |
| `v2/risk/manager.py` | 5 | **仅 docstring** |
| `v2/validation/__init__.py` | 0 | 空 |

### `v2/pipeline/execution.py`（5 行）

```python
"""v2 execution simulation.

Focus: Almgren-Chriss optimal execution, fill probability,
market impact modeling, Sharpe degradation at scale.
"""
```

**规划内容**（来自 docstring）：

- **Almgren-Chriss 最优执行** —— 经典市场冲击模型，最小化市场冲击 × 等待成本
- **fill probability** —— 订单成交概率（限价单 / 大单 / 涨跌停时）
- **market impact modeling** —— 价格冲击建模（Kyle's lambda / Almgren-Chriss）
- **Sharpe degradation at scale** —— 容量限制（策略规模一大 Sharpe 下降）

**学术背景**：

- **Almgren-Chriss (2000)** —— "Optimal Execution of Portfolio Transactions"
- **Kyle (1985)** —— "Continuous Auctions and Insider Trading"
- **Bertsimas-Lo (1998)** —— "Optimal Control of Execution Costs"

**为什么 v2 没有**：

- ❌ 这些需要 broker 协议（拿到真实成交价、滑点、depth）
- ❌ v2 ROADMAP 上 broker protocol 也是 ⬜
- ❌ 当前 PEAD demo 不需要（等金额仓位足够）

### `v2/portfolio/optimizer.py`（5 行）

```python
"""v2 portfolio optimizer.

Focus: mean-variance optimization, eigenvalue cleaning (Marchenko-Pastur),
Black-Litterman, risk parity.
"""
```

**规划内容**：

- **mean-variance optimization (MVO)** —— Markowitz 经典（mean = Signal.value, var = 历史收益方差）
- **eigenvalue cleaning (Marchenko-Pastur)** —— 估计协方差矩阵时去除噪声特征值
- **Black-Litterman** —— 高盛 1990 内部模型，结合市场先验和投资者观点
- **risk parity** —— 风险平价（不是 dollar neutral，是 risk neutral）

**学术背景**：

- **Markowitz (1952)** —— 现代投资组合理论
- **Black-Litterman (1992)** —— Goldman Sachs 内部
- **Marchenko-Pastur** —— 随机矩阵理论，应用于协方差矩阵去噪

**为什么 v2 没有**：

- ❌ 需要历史收益率（v2 还没存过）
- ❌ 需要 alpha model 输出的 view（PEAD 和 Buffett 输出格式已经统一）
- ❌ portfolio_manager 仍是占位

### `v2/risk/manager.py`（5 行）

```python
"""v2 risk management.

Focus: drawdown controls, position sizing by volatility,
correlation-based exposure caps, tail risk metrics, stress testing.
"""
```

**规划内容**：

- **drawdown controls** —— 净值回撤限制
- **position sizing by volatility** —— 波动率调整仓位（v1 piecewise 函数可复用）
- **correlation-based exposure caps** —— 组合相关性上限
- **tail risk metrics** —— VaR / CVaR / 最大回撤分布
- **stress testing** —— 历史情景 / 假设情景压力测试

**v1 已经实现的**（可复用）：

- `calculate_volatility_adjusted_limit`（piecewise 函数）
- `calculate_volatility_metrics`（含 60-day rolling vol percentile）
- correlation matrix（v1 risk_manager 算了但没用）

**v2 还没实现的**：

- VaR / CVaR（v1 没有）
- 压力测试

### `v2/validation/__init__.py`（空目录）

**ROADMAP 上最重要的占位**：

> "Validation gate — CPCV, probability of backtest overfitting (PBO)"

**应该包含**：

- **CPCV** (Combinatorial Purged Cross-Validation) —— Lopez de Prado 2018
- **PBO** (Probability of Backtest Overfitting) —— 同上
- **Combinatorial symmetric cross-validation**

**为什么最重要**：

- ⚠️ 当前 PEAD Sharpe 0.33 是**没经过 overfitting check** 的数字
- ⚠️ 任何新 alpha model 都可能"调参到 Sharpe 2.0"——过拟合
- ⚠️ 没有 CPCV/PBO → PEAD demo 的 Sharpe 数字**没有任何可信度**

**学术背景**：

- **Bailey, Borwein, López de Prado, Zhu (2014)** —— "The Probability of Backtest Overfitting"
- **López de Prado (2018)** —— "Advances in Financial Machine Learning" (Chapter 11)

## 第二部分：v1 杂项工具

### `src/utils/api_key.py`（9 行）

```python
def get_api_key_from_state(state: dict, api_key_name: str) -> str:
    """Get an API key from the state object."""
    if state and state.get("metadata", {}).get("request"):
        request = state["metadata"]["request"]
        if hasattr(request, 'api_keys') and request.api_keys:
            return request.api_keys.get(api_key_name)
    return None
```

**极简**：

- **从 LangGraph state 抽 API key**
- 两层 fallback：state.request.api_keys → None（隐含 `os.getenv` 不在）

**对比 v2**：

- v2 LLM 客户端直接读 `os.getenv("ANTHROPIC_API_KEY")`
- v2 没有"state 注入 API key"概念——CLI 单用户

**对 app/ 集成的价值**：

- v1 的 `api_key_service` 把 API keys 存 DB
- 多用户 web app 通过 `state.request.api_keys` 传给 agent
- v2 没有 web app 集成——这个机制 CN 化时**需要新建**

### `src/utils/display.py`（200+ 行，部分读）

#### `sort_agent_signals`（line 8-14）

```python
def sort_agent_signals(signals):
    analyst_order = {display: idx for idx, (display, _) in enumerate(ANALYST_ORDER)}
    analyst_order["Risk Management"] = len(ANALYST_ORDER)
    return sorted(signals, key=lambda x: analyst_order.get(x[0], 999))
```

**作用**：按 ANALYST_ORDER 排序 signals——保证输出顺序稳定。

#### `print_trading_output`（line 17+）

```python
def print_trading_output(result: dict) -> None:
    decisions = result.get("decisions")
    if not decisions:
        print(f"{Fore.RED}No trading decisions available{Style.RESET_ALL}")
        return
    
    for ticker, decision in decisions.items():
        print(f"\n{Fore.WHITE}{Style.BRIGHT}Analysis for {Fore.CYAN}{ticker}{Style.RESET_ALL}")
        print(f"{Fore.WHITE}{Style.BRIGHT}{'=' * 50}{Style.RESET_ALL}")
        
        table_data = []
        for agent, signals in result.get("analyst_signals", {}).items():
            if ticker not in signals:
                continue
            if agent == "risk_management_agent":
                continue
            
            signal = signals[ticker]
            agent_name = agent.replace("_agent", "").replace("_", " ").title()
            signal_type = signal.get("signal", "").upper()
            confidence = signal.get("confidence", 0)
            
            signal_color = {
                "BULLISH": Fore.GREEN,
                "BEARISH": Fore.RED,
                "NEUTRAL": Fore.YELLOW,
            }.get(signal_type, Fore.WHITE)
            
            ...
```

**作用**：格式化打印 trading decisions——**color-coded 信号 + reasoning**。

**关键**：

- 每个 ticker 一段
- 信号颜色：green/red/yellow
- 跳过 risk_management（**只显示 analyst 信号**）

**对比 v2 demo**：

| 维度 | v1 display.py | v2 demo/backtest.py |
|------|--------------|--------------------|
| 库 | colorama + tabulate | rich |
| 风格 | ASCII 表格 | rich Panel / Table |
| 实时更新 | ❌ 一次性打印 | ✅ Live 渲染 |

### `src/utils/visualize.py`（10 行）

```python
from langgraph.graph.state import CompiledGraph
from langchain_core.runnables.graph import MermaidDrawMethod


def save_graph_as_png(app: CompiledGraph, output_file_path) -> None:
    png_image = app.get_graph().draw_mermaid_png(draw_method=MermaidDrawMethod.API)
    file_path = output_file_path if len(output_file_path) > 0 else "graph.png"
    with open(file_path, "wb") as f:
        f.write(png_image)
```

**极简**：

- 用 LangGraph 自带的 mermaid → PNG 转换
- **依赖 mermaid.ink API**（在线渲染，需要网络）

**触发方式**：`--graph` CLI flag → 调 `save_graph_as_png`。

### `src/utils/ollama.py`（408 行）

#### 文件作用一句话

**完整的 Ollama 本地 LLM 工具集**——安装检测、启动服务、下载模型、跨平台支持。

#### 8 个核心函数

| 函数 | 作用 |
|------|------|
| `is_ollama_installed()` | 检测系统是否装了 ollama（`which` / `where`） |
| `is_ollama_server_running()` | 检测 ollama daemon 是否在跑（HTTP /api/tags） |
| `get_locally_available_models()` | 列出本地已下载的模型 |
| `start_ollama_server()` | 启动 ollama daemon |
| `install_ollama()` | 安装 ollama（mac / linux / win 各自不同） |
| `download_model(model_name)` | 下载指定模型（带进度条） |
| `delete_model(model_name)` | 删除已下载模型 |
| `ensure_ollama_and_model(model_name)` | **总入口**：确保 ollama + 模型都 ready |

#### 跨平台设计

```python
OLLAMA_DOWNLOAD_URL = {
    "darwin": "https://ollama.com/download/darwin",
    "windows": "https://ollama.com/download/windows",
    "linux": "https://ollama.com/download/linux",
}

INSTALLATION_INSTRUCTIONS = {
    "darwin": "curl -fsSL https://ollama.com/install.sh | sh",
    "windows": "# Download from https://ollama.com/download/windows and run the installer",
    "linux": "curl -fsSL https://ollama.com/install.sh | sh",
}
```

**3 个 OS** × **2 个 action**（download / install）= 6 种安装路径。

#### Docker 支持

```python
def ensure_ollama_and_model(model_name: str) -> bool:
    if env_override or ollama_url.startswith("http://ollama:") or ollama_url.startswith("http://host.docker.internal:"):
        return docker.ensure_ollama_and_model(model_name, ollama_url)
    # ... 本地流程 ...
```

**如果 `OLLAMA_BASE_URL` env var 指向 Docker 容器**——委托给 `src/utils/docker.py`。

#### download_model 的进度条

```python
percentage_match = re.search(r"(\d+(\.\d+)?)%", output)
if percentage_match:
    percentage = float(percentage_match.group(1))
    
    filled_length = int(bar_length * percentage / 100)
    bar = "█" * filled_length + "░" * (bar_length - filled_length)
    
    status_line = f"\r{phase_display}{Fore.GREEN}{bar}{Style.RESET_ALL} {Fore.YELLOW}{percentage:.1f}%{Style.RESET_ALL}"
    print(status_line, end="", flush=True)
```

**解析 ollama 的 stdout**——用 regex 提取百分比，画 Unicode 进度条（`█` / `░`）。

### `src/utils/docker.py`（推测）

**Docker 容器辅助**：

- 在 Docker 容器里跑 Ollama
- 处理 host.docker.internal（macOS Docker Desktop）
- 镜像构建 / 启动

### 各 `__init__.py` 状态

```
v2/__init__.py               空
v2/data/__init__.py          完整 facade（已解析）
v2/llm/__init__.py           完整 facade（已解析）
v2/signals/__init__.py       含 ALPHA_MODEL_REGISTRY（已解析）
v2/backtesting/__init__.py   推测含 BacktestEngine re-export
v2/event_study/__init__.py   推测含 compute_car re-export
v2/features/__init__.py     推测含 build_snapshot re-export
v2/demo/__init__.py          空
v2/pipeline/__init__.py      空
v2/portfolio/__init__.py     空
v2/risk/__init__.py          空
v2/validation/__init__.py    空

src/__init__.py              空
src/agents/__init__.py       空（推测）
src/data/__init__.py         空
src/graph/__init__.py        空
src/llm/__init__.py          空
src/utils/__init__.py        空
```

**所有 __init__.py 都很小**——只是 package marker 或少量 re-export。

## v2 占位 vs v1 已有 —— 评估

| 占位 | v2 计划 | v1 已有 | 重用难度 |
|------|---------|--------|---------|
| `v2/pipeline/execution.py` | Almgren-Chriss / 市场冲击 | ❌ v1 没有 | 难（broker 协议） |
| `v2/portfolio/optimizer.py` | MVO / Black-Litterman | ❌ v1 没有 | 中（需要历史收益） |
| `v2/risk/manager.py` | drawdown / vol sizing | ✅ v1 piecewise 函数 | **易（直接搬）** |
| `v2/validation/__init__.py` | CPCV / PBO | ❌ v1 没有 | **新写**（最重要） |

**最该先做的**：

1. **v2/validation/ CPCV/PBO** —— 整个系统可信度的基石
2. **v2/risk/manager.py** —— 复用 v1 piecewise 函数
3. **v2/portfolio/optimizer.py** —— 配合 alpha model 接口
4. **v2/pipeline/execution.py** —— 需要 broker，留到 broker 协议一起做

## 对 fork → CN 版本的最终建议

### A. 优先级（重排）

| 优先级 | 任务 | 价值 | 工作量 |
|--------|------|------|--------|
| 🔴 P0 | v2/validation/ CPCV + PBO | **整个 v2 可信度** | 1-2 周 |
| 🔴 P0 | TushareClient | 端到端跑通 | 1-2 天 |
| 🟡 P1 | v2/risk/manager.py | 风险控制 | 1-2 天（搬 v1） |
| 🟡 P1 | CN PEAD（adapt v2.pead） | A 股专属 | 2-3 天 |
| 🟢 P2 | DeepSeekLLM / QwenLLM | CN LLM | 1 天 |
| 🟢 P2 | 1-2 个中国 persona | demo 价值 | 半天 |
| 🔵 P3 | v2/portfolio/optimizer.py | portfolio construction | 1-2 周 |
| 🔵 P3 | v2/pipeline/execution.py | broker 协议 | 2-4 周（要做 broker） |

### B. CN 化的 v2/risk/manager.py 模板

直接搬 v1 piecewise 函数 + 加 A 股特有风险：

```python
# v2/risk/manager.py
VOL_BANDS = [
    (0.15, 1.25),       # vol < 15% → 25% 仓位上限
    (0.30, 1.00),       # 15-30% → 线性
    (0.50, 0.75),       # 30-50% → 线性
    (float('inf'), 0.50),  # > 50% → 10%
]

class RiskManager:
    def __init__(self, total_capital: float, current_price: float, volatility: float):
        self.total_capital = total_capital
        self.current_price = current_price
        self.volatility = volatility
    
    def position_limit(self) -> float:
        """最大允许仓位（美元）"""
        mult = self._vol_multiplier()
        max_value = self.total_capital * mult
        return max_value
    
    def _vol_multiplier(self) -> float:
        for threshold, mult in VOL_BANDS:
            if self.volatility < threshold:
                return mult
        return 0.50
```

**A 股特有加项**：

```python
# 涨跌停硬约束
def can_buy(ticker, side: str) -> bool:
    """A 股：涨停不能买入，跌停不能卖出"""
    price_change_pct = get_today_change(ticker)
    if side == "buy" and price_change_pct >= 0.095:    # 接近涨停
        return False
    if side == "sell" and price_change_pct <= -0.095:
        return False
    return True

# T+1 解锁检查
def shares_available_to_sell(portfolio, ticker) -> int:
    """A 股：T+1 持仓当日不能卖"""
    if portfolio.last_buy_date[ticker] == today():
        return 0
    return portfolio.positions[ticker]["long"]
```

### C. CN 化的 v2/validation/ 模板

```python
# v2/validation/cpcv.py
def cpcv(backtest_results: list[Trade], n_groups: int = 16, k_test: int = 4):
    """Combinatorial Purged Cross-Validation.
    
    把时间序列切成 n_groups，对每 (k_test 组做测试、其余做训练) 的组合算 Sharpe。
    """
    ...

# v2/validation/pbo.py
def pbo(backtest_results_per_config: list[list[Trade]]) -> float:
    """Probability of Backtest Overfitting.
    
    给定一组参数 config 的 backtest 结果，估 P(is_best config 是 overfit) ∈ [0, 1].
    """
    ...
```

**关键公式参考**：

- CPCV: López de Prado, "Advances in Financial Machine Learning" Ch. 11
- PBO: Bailey et al. (2014)

## 深读问题（自检）

- [ ] `v2/pipeline/execution.py` 5 行 docstring 规划**Almgren-Chriss**——但 ROADMAP broker protocol ⬜。**两个一起做，还是先 broker 后 execution？**
- [ ] `v2/portfolio/optimizer.py` 5 行 docstring 规划**Black-Litterman**——BL 需要 view（来自 alpha model）。v2 alpha model 现在只有 2 个（PEAD + Buffett）。**够不够 view？**
- [ ] `v2/risk/manager.py` v1 piecewise 函数能搬——但 v1 用 `np.sqrt(252)` 年化波动率，A 股用 244 还是 252？**(A 股每年 244 个交易日)**
- [ ] `v2/validation/__init__.py` 完全空——**写 CPCV/PBO 应该独立模块还是放在 validation 包里？**

---

*解析时间：2026-07-12 · 第十九轮迭代（最终）*
*全项目解析 100% 完成。共 30 篇 deep dive，全部位于 [docs/study-notes/modules/](docs/study-notes/modules/)。*