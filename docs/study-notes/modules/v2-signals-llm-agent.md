# v2/signals/llm_agent.py 深度解析

> 文件：`v2/signals/llm_agent.py` · 173 行 · 1 个 ABC + 5 个 method
> 地位：**v2 LLM investor agent 的基类**——所有 persona（Buffett / Munger / Lynch ...）只需要写 `name` + `get_system_prompt()`，其余全自动

## 文件作用一句话

**"Persona = 名字 + system prompt"**——所有 LLM 投资 agent 共享同一份机制：snapshot → prompt → LLM → parse → Signal。失败语义分两层（数据失败 propagate，LLM 失败 abstain）。

## 关键哲学（docstring line 16-21）

```python
"""LLMAgent — base class for LLM investor agents (the second AlphaModel flavor).

An LLMAgent reasons over a point-in-time FundamentalsSnapshot in a persona's
voice and emits the same Signal every quant model does. The base class owns
all the machinery; a persona is just a name + a system prompt:

    class BuffettAgent(LLMAgent):
        @property
        def name(self) -> str:
            return "buffett"

        def get_system_prompt(self) -> str:
            return "You are Warren Buffett..."

Failure contract (locked decisions):
- Data-layer errors PROPAGATE (fail loud — a broken snapshot must never
  silently become a neutral view).
- LLM call/parse failures ABSTAIN: Signal(value=0.0, metadata.abstained=True).
- Every LLM decision persists its exact prompt + response (via PromptCache),
  and an unchanged snapshot never pays for a second LLM call.
"""
```

**3 条锁定的不妥协原则**：

| 失败类型 | 处理 | 为什么 |
|---------|------|--------|
| **数据层失败**（FDClientError） | propagate raise | 一旦数据出错，**必须让 backtest 崩**，绝不能静默 |
| **LLM call 失败**（timeout / auth） | 返回 abstain Signal | LLM 临时不可用，不应该让 backtest 崩 |
| **Parse 失败**（LLM 输出非 JSON） | 返回 abstain Signal + 写盘 | 调试时需要 raw response |

**这是 v2 设计哲学最集中的体现**：**该崩的崩，不该崩的不崩**。

## `predict` 主流程（line 54-95）

```python
def predict(self, ticker, date, data_client):
    # 1. 数据快照
    try:
        snapshot = self.build_snapshot(ticker, date, data_client)
    except InsufficientData as exc:
        return self._abstain(ticker, date, f"insufficient data: {exc}")
    # 2. 其他 data 异常（如 FDClientError）→ 直接 propagate，不在这里吞
    
    # 3. 准备 prompt + cache key
    system = self.get_system_prompt()
    user = self.build_user_prompt(snapshot)
    key = prompt_key(self.name, self._llm.model, system, user)
    
    # 4. cache 命中直接返回
    cached = self._cache.get(key)
    if cached is not None and "parsed" in cached:
        return self._to_signal(ticker, date, cached["parsed"], key, snapshot, cached=True)
    
    # 5. 调 LLM
    try:
        response = self._llm.complete(system, user)
    except Exception as exc:
        logger.warning(...)
        return self._abstain(ticker, date, f"LLM call failed: {exc}")
    
    # 6. 解析 + 持久化
    record = {
        "agent": self.name,
        "model": self._llm.model,
        "ticker": ticker,
        "as_of": date,
        "snapshot_hash": snapshot.content_hash,
        "system": system,
        "user": user,
        "response": response,
    }
    try:
        parsed = self._parse(response)
    except Exception as exc:
        # parse 失败也写盘！
        self._cache.put(key, {**record, "parse_error": str(exc)})
        logger.warning(...)
        return self._abstain(ticker, date, f"parse failed: {exc}")
    
    # 7. 成功路径：写盘 + 返回 Signal
    self._cache.put(key, {**record, "parsed": parsed})
    return self._to_signal(ticker, date, parsed, key, snapshot, cached=False)
```

### 7 步流程图

```
predict(ticker, date, data_client)
    ↓
[1] build_snapshot ── InsufficientData → abstain
    ↓ ok
    ↓ (其他异常 propagate)
[2] get_system_prompt + build_user_prompt(snapshot)
    ↓
[3] prompt_key = hash(agent, model, system, user)
    ↓
[4] cache.get(key) → 命中 → to_signal(cached=True) ✓ 退出
    ↓ miss
[5] llm.complete(system, user) ── 异常 → abstain
    ↓ ok
[6] parse(response) ── 异常 → 写盘(parse_error) → abstain
    ↓ ok
[7] 写盘(parsed) + to_signal(cached=False) ✓ 退出
```

## 失败处理的两层抽象

```python
# 第一层：数据失败
try:
    snapshot = self.build_snapshot(ticker, date, data_client)
except InsufficientData as exc:
    return self._abstain(ticker, date, f"insufficient data: {exc}")
# 其他异常（如 FDClientError）→ 不捕获，往上抛

# 第二层：LLM 失败
try:
    response = self._llm.complete(system, user)
except Exception as exc:
    logger.warning(...)
    return self._abstain(ticker, date, f"LLM call failed: {exc}")
```

**对比 v1**：v1 没有这种分层——任何错误都可能被吞掉或直接返回默认值。v2 的语义明确：

| 异常来源 | 信号 value | 含义 |
|---------|-----------|------|
| 数据不够（`InsufficientData`） | 0.0 + `abstained: True` | 优雅降级 |
| 网络/服务失败（`FDClientError`） | **backtest 整体抛错** | 必须修 |
| LLM 失败 | 0.0 + `abstained: True` | 优雅降级 |
| Parse 失败 | 0.0 + `abstained: True` | 优雅降级 |

## `_parse` 验证逻辑（line 122-135）

```python
def _parse(self, response: str) -> dict:
    data = extract_json(response)
    signal = str(data.get("signal", "")).lower()
    if signal not in _SIGNAL_TO_SIGN:
        raise ValueError(f"invalid signal {data.get('signal')!r}")
    confidence = float(data.get("confidence", 0))
    if not 0 <= confidence <= 100:
        raise ValueError(f"confidence out of range: {confidence}")
    return {
        "signal": signal,
        "confidence": confidence,
        "reasoning": str(data.get("reasoning", "")),
    }
```

**4 个验证**：

1. `extract_json` —— 三段降级（见 `v2/llm/client.py`）
2. `signal ∈ {"bullish", "neutral", "bearish"}` —— 严格枚举
3. `0 ≤ confidence ≤ 100` —— 数值范围
4. `reasoning` —— 转 str，缺失变 ""

**严格验证的原因**：

LLM 经常"脑补"输出。比如输出 `{"signal": "bullishish"}`（拼错）——必须 raise，不能默认成 "neutral"。

## `_to_signal` 计算 conviction（line 137-162）

```python
def _to_signal(self, ticker, date, parsed, key, snapshot, cached):
    value = _SIGNAL_TO_SIGN[parsed["signal"]] * parsed["confidence"] / 100.0
    return Signal(
        model_name=self.name,
        ticker=ticker,
        date=date,
        value=value,
        reasoning=parsed["reasoning"],
        metadata={
            "signal": parsed["signal"],
            "confidence": parsed["confidence"],
            "model": self._llm.model,
            "prompt_key": key,
            "snapshot_hash": snapshot.content_hash,
            "cached": cached,
            "abstained": False,
        },
    )
```

**关键公式**：

```python
value = direction * confidence / 100
```

- `direction`: `bullish → +1`, `bearish → -1`, `neutral → 0`
- `confidence`: 0-100
- 结果 ∈ [-1, +1]

**举例**：

| signal | confidence | value |
|--------|-----------|-------|
| bullish | 100 | +1.0 |
| bullish | 75 | +0.75 |
| bullish | 50 | +0.5 |
| neutral | 50 | 0.0 |
| bearish | 80 | -0.8 |

**metadata 字段详解**：

| 字段 | 用途 |
|------|------|
| `signal` | 原始信号（bullish/bearish/neutral） |
| `confidence` | 原始 confidence 0-100 |
| `model` | 实际用的 LLM 模型名 |
| `prompt_key` | 缓存 key（hash 前 24 位）——审计用 |
| `snapshot_hash` | snapshot 内容 hash——审计用 |
| `cached: True` | 这次决策是从磁盘缓存读出来的，没真打 LLM |
| `cached: False` | 这次打了 LLM |
| `abstained: False` | 这是个真实观点，不是 abstain |

## `_abstain` 设计（line 164-172）

```python
def _abstain(self, ticker, date, reason):
    return Signal(
        model_name=self.name,
        ticker=ticker,
        date=date,
        value=0.0,
        reasoning=f"abstained: {reason}",
        metadata={"abstained": True, "abstain_reason": reason, "cached": False},
    )
```

**关键差异**：

| 字段 | 真实 Signal | Abstain Signal |
|------|-------------|----------------|
| `value` | 任意 ∈ [-1, +1] | 固定 0.0 |
| `reasoning` | LLM 给的论据 | `"abstained: <原因>"` |
| `metadata.abstained` | False | True |
| `metadata.cached` | 实际值 | **固定 False**（abstain 不写 cache） |

**为什么 abstain 不写 cache**：

cache 是为"LLM 调用昂贵"而设计的——abstain 没调 LLM，没必要写。

## 子类钩子（line 101-116）

```python
def get_system_prompt(self) -> str:
    """The persona — every subclass must define its voice."""
    raise NotImplementedError(f"{type(self).__name__} must define get_system_prompt()")

def build_snapshot(self, ticker, date, data_client) -> FundamentalsSnapshot:
    """What this persona is allowed to know. Default: the shared
    point-in-time fundamentals snapshot — right for value/quality
    personas. Override for personas that reason over different data
    (macro, news); when a second snapshot TYPE exists, extract the
    implicit interface (ticker/as_of/content_hash/render) into a
    Protocol — not before."""
    return build_snapshot(ticker, date, data_client)

def build_user_prompt(self, snapshot: FundamentalsSnapshot) -> str:
    """Default user prompt: the rendered snapshot. Override to enrich."""
    return snapshot.render()
```

**3 个可重写方法**：

1. **`get_system_prompt()`** —— **必填**，persona 的声音
2. **`build_snapshot()`** —— 默认 `FundamentalsSnapshot`；宏观/新闻 persona 应 override
3. **`build_user_prompt()`** —— 默认 `snapshot.render()`；可在 snapshot 基础上加额外信息

**注释里的重要警告**（line 109-111）：

> "when a second snapshot TYPE exists, extract the implicit interface (ticker/as_of/content_hash/render) into a Protocol — not before"

**关键洞见**：**不要预先抽象**——先有第二个 snapshot type（不是 1 个），再抽 Protocol。这避免了 YAGNI 抽象。

## 设计取舍

### 1. 为什么 `Exception` 不是 `LLMError`？

```python
except Exception as exc:
    logger.warning(...)
    return self._abstain(...)
```

✅ **捕获所有 Exception**：避免漏掉新的异常类型

❌ **捕获特定 LLMError**：可能漏掉 timeout / auth 等未分类的异常

**判断**：合理。**宁可 abstain 多，不可崩 backtest**。

### 2. parse 失败也写盘

```python
self._cache.put(key, {**record, "parse_error": str(exc)})
```

✅ **写盘**（带 `parse_error` 字段）——debug trail

❌ **不写盘**——下次同样 prompt 还会调 LLM 重新 parse

**判断**：合理。**audit 优先于效率**。

### 3. cache key 包含 model

```python
key = prompt_key(self.name, self._llm.model, system, user)
```

✅ **包含 model**：

- 换模型自动隔离缓存
- 避免"用 GPT-4 拿到 GPT-3.5 答案"

❌ **不包含**：模型升级会复用旧答案

**判断**：合理。

## 对 fork → CN 版本的启示

### A. 完全不动这个文件

LLMAgent 是抽象基类，CN 化**只改 get_system_prompt() 的内容**——比如：

```python
class BuffettAgent(LLMAgent):
    @property
    def name(self):
        return "buffett"
    
    def get_system_prompt(self):
        return "你是沃伦·巴菲特。基于以下事实给出买入/卖出/中性判断..."

class 段永平Agent(LLMAgent):
    @property
    def name(self):
        return "duan_yongping"
    
    def get_system_prompt(self):
        return "你是段永平。'敢为天下后'，'做对的事情，把事情做对'..."
```

### B. 失败语义对 CN LLM 的影响

| 失败 | 美 LLM | CN LLM |
|------|--------|--------|
| API timeout | raise → abstain | 同（一致） |
| 输出非 JSON | parse 失败 → abstain | **更常见**（CN LLM 喜欢啰嗦解释）→ 解析降级链要强 |
| 输出中文 reasoning | 不适用 | OK，已在 metadata.reasoning |

**CN LLM 容易出现的特殊 case**：

1. **包在 markdown 里**（` ```json ... ``` `）—— `extract_json` 第一段已覆盖
2. **混合中英文 key**（`{"signal": "bullish", "置信度": 75}`）—— 需要 `extract_json` 兼容
3. **过长 reasoning**（几千字）—— token cost 高，但不影响 parse

### C. metadata.abstained 的下游处理

下游（portfolio construction / backtest engine）看到 `abstained: True` 应该：

- **不要计入 composite score** —— 跳过
- **记录 abstain 率** —— 高 abstain 率说明 LLM 不稳定

**v2 当前没强制这点**——取决于 portfolio 实现。

## 深读问题（自检）

- [ ] `except InsufficientData` 后 `return _abstain` —— **但 line 59 注释说 "Any other data-layer exception propagates"**。如果 `build_snapshot` 抛 `FDClientError`，整个 backtest 崩——这个行为合理吗？
- [ ] `parse` 失败时记录了 `parse_error` 但没记录 `parsed: None`——下次同样 prompt，cache 命中后会进 `to_signal` 看到 `parsed=None` 怎么办？（提示：`"parsed" in cached` 这个判断）
- [ ] metadata 的 `cached: False` 在 abstain 时是**硬编码 False**——为什么 abstain 不能也 cached？
- [ ] LLMAgent 没暴露 `update_cache()` 或 `clear_cache()`——backtest 完想清理怎么办？

---

*解析时间：2026-07-12 · 第六轮迭代*
*下次解析目标：`v2/signals/buffett.py`*