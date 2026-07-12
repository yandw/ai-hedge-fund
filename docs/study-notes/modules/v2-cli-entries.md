# v2/__init__.py + 各 __init__ + CLI 入口深度解析

> 文件：`v2/__init__.py` + `v2/data/__init__.py` + `v2/llm/__init__.py` + `v2/signals/__init__.py` + `v2/analyze.py` + `v2/backtesting/__main__.py` + `v2/event_study/__main__.py` + `v2/conftest.py`
> 总计：8 个文件，每个都很小（10-200 行）

## 这次解析的目标

把 v2 里所有"小文件"集中解析：模块注册表（`__init__.py`）、CLI 入口（`__main__.py`、`analyze.py`）、测试钩子（`conftest.py`）。

## `__init__.py` 系列：模块注册表

### `v2/__init__.py`

**空文件**——只做 package 标记，**没有公开任何 API**。

**意义**：v2 不希望被 `from v2 import *` 整个拉进来——用户必须显式 `from v2.signals import PEADModel`。

### `v2/data/__init__.py`（33 行）

```python
from v2.data.cached import CachedDataClient
from v2.data.client import FDClient, FDClientError
from v2.data.models import (
    CompanyFacts, CompanyNews, Earnings, EarningsData,
    EarningsRecord, Filing, FinancialMetrics, InsiderTrade, Price,
)
from v2.data.protocol import DataClient

__all__ = [...]    # 全部公开符号
```

**设计意图**：

- `v2.data` 是一个**完整的 facade**——用户只用 `from v2.data import FDClient`
- 内部有 `cached` / `client` / `models` / `protocol` 4 个子模块，但**外部看不出来**
- 这是 v2 的"API 边界"——CN 化时**新增 client 在这里也加一行**就 OK

### `v2/llm/__init__.py`（15 行）

```python
from v2.llm.cache import PromptCache, prompt_key
from v2.llm.client import DEFAULT_MODEL, AnthropicLLM, LLMClient, LLMParseError, extract_json

__all__ = [...]
```

**7 个公开符号**——LLM 层完整 surface。

**CN 化时**：在 `v2/llm/client.py` 加 `DeepSeekLLM`，然后这里加 `from v2.llm.client import DeepSeekLLM` 和 `__all__` 加一项即可。

### `v2/signals/__init__.py`（27 行）—— 关键

```python
from v2.signals.base import AlphaModel, QuantModel
from v2.signals.buffett import BuffettAgent
from v2.signals.llm_agent import LLMAgent
from v2.signals.pead import PEADModel

ALPHA_MODEL_REGISTRY: dict[str, type[AlphaModel]] = {
    "pead": PEADModel,
    "buffett": BuffettAgent,
}
```

**关键设计**：`ALPHA_MODEL_REGISTRY` —— **字典映射"agent 名 → 类"**。

**意义**：

- CLI 可以用字符串选择 agent：`--agent pead` / `--agent buffett`
- 添加新 agent 不需要改 CLI，**只在这里加一行**

**CN 化**：

```python
ALPHA_MODEL_REGISTRY: dict[str, type[AlphaModel]] = {
    "pead": PEADModel,
    "buffett": BuffettAgent,
    "cn_pead": CN_PEADModel,           # ← 新加
    "duan_yongping": DuanYongpingAgent,  # ← 新加
}
```

### `v2/conftest.py`（5 行）

```python
"""Load .env for v2 tests so FINANCIAL_DATASETS_API_KEY is available."""
from dotenv import load_dotenv
load_dotenv()
```

**5 行的关键作用**：

- pytest 启动时自动调用 `load_dotenv()`
- 测试可以直接用 `os.getenv("FINANCIAL_DATASETS_API_KEY")`
- **不需要在每个测试里手动加载 .env**

**注意**：pytest 的 `conftest.py` 在**包级别**自动被所有子目录的测试发现——这是 pytest 的魔法。

## CLI 入口

### `v2/analyze.py`（95 行）—— 单 ticker 观点查询

**用法**：

```bash
poetry run python -m v2.analyze NVDA
poetry run python -m v2.analyze NVDA --date 2024-06-01
poetry run python -m v2.analyze AAPL --agent pead
```

**主流程**（line 27-90）：

```python
def main():
    load_dotenv()
    parser = argparse.ArgumentParser(...)
    parser.add_argument("ticker")
    parser.add_argument("--date", default=_date.today().isoformat())
    parser.add_argument("--agent", default="buffett", choices=sorted(ALPHA_MODEL_REGISTRY))
    args = parser.parse_args()
    
    model = ALPHA_MODEL_REGISTRY[args.agent]()
    ticker = args.ticker.upper()
    console = Console(stderr=True)
    
    started = time.time()
    with FDClient() as raw:
        fd = CachedDataClient(raw)
        with console.status(f"[cyan]building snapshot...", spinner="dots") as status:
            if isinstance(model, LLMAgent):
                try:
                    snapshot = model.build_snapshot(ticker, args.date, fd)
                    status.update(f"[bold cyan]{args.agent}[/] is reading...")
                except InsufficientData:
                    pass
            signal = model.predict(ticker, args.date, fd)
    
    elapsed = time.time() - started
    ...
    print(bar)
    print(f"  {args.agent.upper()} on {signal.ticker}   (as of {signal.date})")
    print(f"  view:        {direction}  ({signal.value:+.2f})")
    if signal.metadata.get("confidence") is not None:
        print(f"  confidence:  {signal.metadata['confidence']:.0f}/100")
    if signal.metadata.get("model"):
        cached = "  [cached]" if signal.metadata.get("cached") else ""
        print(f"  model:       {signal.metadata['model']}{cached}  ({elapsed:.1f}s)")
```

#### 关键设计点

**1. stderr 用作 spinner，stdout 用作结果**

```python
console = Console(stderr=True)    # spinner on stderr
print(...)    # results stay on stdout
```

**好处**：

- ✅ spinner 不污染 stdout → **可以 pipe**：
  ```bash
  poetry run python -m v2.analyze NVDA > result.txt
  ```
- ✅ spinner 不会和最终结果混在一起

**2. 两阶段 spinner**

```python
with console.status(f"[cyan]building snapshot...", spinner="dots") as status:
    if isinstance(model, LLMAgent):
        try:
            snapshot = model.build_snapshot(...)
            status.update(f"[bold cyan]{args.agent}[/] is reading...")   # ← 第二阶段
        except InsufficientData:
            pass
    signal = model.predict(...)
```

**LLM agent 两步**：
- 第一步：build_snapshot（拿数据）
- 第二步：predict（调 LLM）

**Quant agent 一步**：直接 predict（PEAD 用 earnings history）

**3. 显式 `try/except InsufficientData`**

```python
try:
    snapshot = model.build_snapshot(ticker, args.date, fd)
    status.update(...)
except InsufficientData:
    pass    # predict() will produce the abstain signal
```

**为什么 catch 而不是 let it propagate**：

- build_snapshot 是**为 spinner 信息准备的**——catch 它不会让 predict 失败
- predict 自己会处理 InsufficientData → abstain signal

**4. 输出格式**

```
──────────────────────────────
  BUFFETT on NVDA   (as of 2024-06-01)
──────────────────────────────
  view:        BULLISH  (+0.75)
  confidence:  75/100
  model:       claude-sonnet-5  (2.3s)
──────────────────────────────
  Durable moat, consistent ROE, fair price...
```

**6 行简洁输出**——一眼看懂。

### `v2/backtesting/__main__.py`（191 行）—— 100 只 ticker 回测

**特点**：

- **100 只 ticker universe**（按行业分组注释）
- 用 ANSI 颜色（不是 rich）
- 串行回测（不是 demo 的 replay）
- **清屏 + 重打**模拟"实时"dashboard

**关键代码**：

```python
TICKERS = [
    # Tech (21)
    "AAPL", "MSFT", "AMZN", ...
    # Financials (15)
    "JPM", "GS", ...
    ...
]
```

**100 只分类清楚**——tech / financials / healthcare / energy / consumer / industrials / media / other。

**主流程**（line 124-184）：

```python
def main():
    n = len(TICKERS)
    engine = BacktestEngine(capital=CAPITAL, per_trade=PER_TRADE)
    model = PEADModel()
    
    # Phase 1: 串行回测
    sys.stdout.write(f"  Backtesting PEAD alpha... [0/{n}]")
    sys.stdout.flush()
    trades = []
    with FDClient() as fd:
        for i, ticker in enumerate(TICKERS):
            sys.stdout.write(f"\r  Backtesting PEAD alpha... [{i + 1}/{n}] {ticker:<6}")
            sys.stdout.flush()
            r = engine.run_alpha(model, [ticker], fd, START_DATE, END_DATE, holding_days=HOLDING_DAYS)
            trades.extend(r.trades)
    
    if not trades:
        print("  No trades generated.")
        return
    
    trades.sort(key=lambda t: t.entry_date)
    
    # Phase 2: 清屏 + replay
    equity = CAPITAL
    displayed = []
    for trade in trades:
        equity += trade.pnl
        displayed.append(trade)
        clear()    # ← 清屏
        running_curve = engine._build_equity_curve(displayed)
        running_metrics = engine._compute_metrics(displayed, running_curve)
        print_header(running_metrics, equity)
        print_table_header()
        for t in reversed(displayed):
            print_trade_row(t)
        time.sleep(0.4)
    
    print(f"  Done. {len(trades)} trades executed.")
```

#### 关键设计点

**1. ANSI 颜色 vs rich**

```python
GREEN = "\033[32m"
RED = "\033[31m"
...
print(f"{GREEN}LONG {RESET}")    # ← 直接拼接 ANSI 转义
```

**为什么不用 rich**：

- rich 比较重
- 这个 CLI 只要 basic 颜色
- ANSI 直接嵌入字符串简单

**2. 清屏 + 重打**

```python
clear()    # os.system("clear")
```

每笔交易后清屏——**模拟"实时 dashboard"效果**。

**3. `time.sleep(0.4)`**

每笔 0.4 秒——**演示节奏**（不是真的实时）。

**4. `_date.today()` vs pinned**

```python
END_DATE = date.today().isoformat()    # ← 这次用 today 了
```

**注意**：和 `v2/demo/backtest.py` 不同——这里**用 `today()`**（不是 pinned）。因为这是 CLI 回测**应该用最新数据**。

**对比**：

| 文件 | 日期 |
|------|------|
| `v2/demo/backtest.py` | pinned（演示一致性） |
| `v2/backtesting/__main__.py` | today（最新数据） |
| `v2/event_study/__main__.py` | today（最新数据） |

### `v2/event_study/__main__.py`（135 行）—— 事件研究 CLI

**特点**：

- **100 只 ticker 同样分类**（和 backtesting 一样的 universe）
- 计算所有 earnings event 的 CAR
- 按 ticker + date 排序打印
- `time.sleep(0.6)` 每行延迟——演示节奏

**关键代码**：

```python
for e in sorted(all_events, key=lambda x: (x.ticker, x.event_date)):
    eps = color_eps(e.eps_surprise)
    c1 = color_car(e.car_0_1)
    c5 = color_car(e.car_0_5)
    c20 = color_car(e.car_0_20)
    
    print(
        f"  {e.ticker:<6} {e.event_date:<12} {e.source_type:<6} {eps}"
        f"  {c1} {c5} {c20}"
        f"   {e.market_model.beta:5.2f} {e.market_model.r_squared:5.2f}"
    )
    time.sleep(0.6)
```

**输出每行 10 列**：

```
Ticker   Date         Type   EPS  CAR[0,1] CAR[0,5] CAR[0,20]   Beta   R2
AAPL     2023-08-04   8-K    BEAT    +0.5%    +1.2%     +2.3%   1.05  0.45
```

**每行 0.6 秒**——100 events = 60 秒完整重放。

## 三个 CLI 的对比

| CLI | Universe | 输出风格 | 用途 |
|-----|---------|---------|------|
| `v2.analyze` | 单 ticker | rich spinner + 简洁 6 行 | 临时查一个观点 |
| `v2.backtesting` | 100 ticker | ANSI 清屏 + replay | 全市场 PEAD 回测 |
| `v2.event_study` | 100 ticker | ANSI + 表格 + replay | 学术风格 CAR 统计 |
| `v2.demo.backtest` | 25 ticker | rich 大 dashboard | 演示 + 演讲 |

## 设计取舍

### 1. 为什么 `__init__.py` 不全公开？

✅ **`__all__` 严格控制**：

- 防止"import 了一切导致 Pydantic 全加载"
- 让公共 API 显式

❌ **全部公开**：破坏封装，未来加内部 helper 会污染命名空间

**判断**：合理。

### 2. 为什么 `ALPHA_MODEL_REGISTRY` 而不是 decorator？

✅ **字典**：

- 显式
- 集中
- 看一眼就知道有哪些 agent

❌ **decorator（`@register_agent`）**：

- 自动注册
- 但分散在各个文件
- 调试时不直观

**判断**：合理。**字典注册表在 v2 这种规模刚好**。

### 3. `__main__.py` 在子模块里

```bash
poetry run python -m v2.backtesting
```

✅ **`-m v2.backtesting`**——直接执行子包

**对比 v1**：

```bash
poetry run python src/backtester.py
```

**v2 现代化**：用 `-m` 让 Python 把包当模块执行，符合 PEP 338。

### 4. `time.sleep()` 演示节奏

```python
time.sleep(0.4)    # backtesting
time.sleep(0.6)    # event_study
```

✅ **sleep 让观众跟上**

❌ **不 sleep**：结果瞬间刷完，观众眼花

**判断**：演示 CLI 的合理选择。**production CLI 不应 sleep**。

## 对 fork → CN 版本的启示

### A. `__init__.py` 系列 + 注册表完全照抄

**改的**：

```python
# v2/signals/__init__.py
ALPHA_MODEL_REGISTRY = {
    "pead": PEADModel,
    "buffett": BuffettAgent,
    "cn_pead": CN_PEADModel,                    # 新加
    "duan_yongping": DuanYongpingAgent,         # 新加
}
```

**`v2/data/__init__.py`** 加 `TushareClient`（如果 import）即可。

### B. CLI 的 ANSi/rich 可以本地化

CN 化的命令行输出可以加：

- "涨跌停"特殊标记
- 中文 tag（`"看多"`, `"看空"`）
- "T+1 解锁"日期提醒

### C. `conftest.py` 的 `load_dotenv()` 需要扩展

```python
# conftest.py 的扩展
from dotenv import load_dotenv
import os

load_dotenv()

# 验证 CN 必需 env var
for var in ["TUSHARE_TOKEN", "ANTHROPIC_API_KEY"]:
    if not os.getenv(var):
        import warnings
        warnings.warn(f"{var} not set — some tests may fail")
```

## 深读问题（自检）

- [ ] `v2/__init__.py` 是**完全空的**——为什么 `v2.data/__init__.py` 有内容，而顶层 v2 没有？是因为顶层没东西要公开，还是有意的"不导出"哲学？
- [ ] `ALPHA_MODEL_REGISTRY` 用 dict，但 `LLMAgent.predict()` 用 `isinstance` 区分——这是合理的分层吗？还是应该用 polymorphism？
- [ ] `v2.analyze` 的 `Console(stderr=True)` 让 spinner 不污染 stdout——但如果用户把 stderr 也 redirect 了（`2>/dev/null`），spinner 消失但结果还在。**这种情况要测试吗**？
- [ ] `v2.backtesting/__main__.py` 用 `time.sleep(0.4)` 让"demo"——production CLI 不应该 sleep，但 **`-m v2.backtesting` 是 production 入口还是 demo？**（提示：注释第一行说"screen-record friendly"——是 demo）

---

*解析时间：2026-07-12 · 第十轮迭代*
*下次解析目标：`v2/__init__.py`（空文件跳过）+ 各 tests 文件（6 个）+ v2/{pipeline,portfolio,risk,validation}（占位）+ 然后转 src/ v1（~40 个文件）*