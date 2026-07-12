# v2/llm/client.py 深度解析

> 文件：`v2/llm/client.py` · 110 行 · 1 个 Protocol + 1 个 Provider + 1 个解析器
> 地位：**LLM provider 的"协议 + 默认实现 + 响应解析"**——v2 默认用 Anthropic，文档明示"刻意不用 langchain 的结构化输出"

## 文件作用一句话

**定义 LLM provider 接口（Protocol），给出 Anthropic 的具体实现，提供一个能容忍 LLM 各种"非 JSON 输出"的 JSON 解析器。**

## 三个组件

| 组件 | 行号 | 作用 |
|------|------|------|
| `LLMParseError` | 22-23 | 自定义异常（JSON 解析失败） |
| `LLMClient` Protocol | 27-37 | LLM provider 接口（结构化 typing） |
| `AnthropicLLM` | 40-75 | Anthropic 实现（langchain-anthropic 包装） |
| `extract_json` | 78-110 | 三段降级 JSON 解析器 |

## 关键哲学（来自 docstring line 4-11）

```python
"""LLM provider protocol + the Anthropic implementation.

Mirrors the DataClient pattern (v2/data/protocol.py): agents depend on the
`LLMClient` protocol, never a concrete provider. Any class with a
`complete(system, user) -> str` method plugs in — community providers welcome.

We deliberately do NOT use langchain's structured-output machinery: its
forced-tool mode breaks on Anthropic reasoning models (v1 carries the same
workaround). We ask for JSON in the prompt and parse it ourselves.
"""
```

**三条原则**：

1. **Provider 跟 DataClient 一样用 Protocol**——任何实现 `complete(system, user) -> str` 的类都是合法 provider
2. **刻意不用 langchain 的 `with_structured_output`**——在 Anthropic reasoning 模型上会坏（v1 也有同样 workaround）
3. **prompt 里要 JSON，自己 parse**——不依赖框架的"魔法"

## `LLMClient` Protocol（line 27-37）

```python
@runtime_checkable
class LLMClient(Protocol):
    """Protocol all LLM providers must satisfy.
    
    complete() returns the model's raw text. Providers should raise on
    transport failure — the LLMAgent layer decides to abstain, not the
    provider.
    """
    
    model: str
    
    def complete(self, system: str, user: str) -> str: ...
```

**接口最小化**：

- `model: str` —— 模型标识（用于 cache key）
- `complete(system, user) -> str` —— 一个方法搞定

**注意**：

- 返回的是 **raw text**——不带 JSON、不带结构化
- 解析是 `LLMAgent` 的事，不是 provider 的事
- provider 应该 raise（不是返回空）——abstain 决策在 agent 层

## `AnthropicLLM`（line 40-75）

```python
class AnthropicLLM:
    """Anthropic provider via the existing langchain-anthropic dependency
    (transport only — no structured-output magic)."""
    
    def __init__(self, model=None, timeout=60.0, max_tokens=1024):
        api_key = os.getenv("ANTHROPIC_API_KEY")
        if not api_key:
            raise ValueError("ANTHROPIC_API_KEY not found...")
        from langchain_anthropic import ChatAnthropic
        
        self.model = model or os.getenv("V2_LLM_MODEL", DEFAULT_MODEL)
        self._chat = ChatAnthropic(
            model=self.model,
            api_key=api_key,
            timeout=timeout,
            max_retries=1,
            max_tokens=max_tokens,
        )
    
    def complete(self, system, user):
        result = self._chat.invoke([("system", system), ("human", user)])
        content = result.content
        if isinstance(content, list):
            content = "".join(
                block.get("text", "") if isinstance(block, dict) else str(block)
                for block in content
            )
        return content
```

### 关键实现细节

#### 1. **API key 检查在构造时**

```python
if not api_key:
    raise ValueError("ANTHROPIC_API_KEY not found...")
```

- ✅ **fail-fast**——没 key 立刻报错，而不是第一次调用时才崩
- ❌ 没有 key 时无法 import 这个类

#### 2. **lazy import langchain**

```python
from langchain_anthropic import ChatAnthropic
```

放在方法体内 / 函数体内（其实在 `__init__` 里不算最 lazy，但避免了模块级别的 import）。

#### 3. **支持 env var override 模型**

```python
self.model = model or os.getenv("V2_LLM_MODEL", DEFAULT_MODEL)
```

- 默认模型：`DEFAULT_MODEL = "claude-sonnet-5"`（line 19）
- 可以通过 `V2_LLM_MODEL` env var 覆盖（如 `V2_LLM_MODEL=claude-fable-5`）

#### 4. **处理 reasoning 模型的 content blocks**

```python
content = result.content
if isinstance(content, list):
    content = "".join(
        block.get("text", "") if isinstance(block, dict) else str(block)
        for block in content
    )
return content
```

**为什么需要这段**：

Anthropic reasoning 模型（如 Claude extended thinking）返回的 content 可能是 list of blocks：

```python
[
  {"type": "thinking", "text": "...思考过程..."},
  {"type": "text", "text": "...最终输出..."}
]
```

**`complete()` 只返回 text 部分**——`thinking` 部分被丢弃（不影响 JSON 解析，但 audit 时会丢思考过程）。

**改进空间**：可以返回 `(thinking, text)` 元组，但当前 API 是 `-> str`，所以简化为只保留 text。

## `extract_json` —— 三段降级解析器（line 78-110）

```python
def extract_json(text: str) -> dict:
    """Pull the first JSON object out of an LLM response.
    
    Tries: ```json fence -> whole string -> first balanced {...} block.
    Raises LLMParseError if nothing parses.
    """
    # 1. ```json fence
    fence = re.search(r"```(?:json)?\s*(\{.*?\})\s*```", text, re.DOTALL)
    if fence:
        try:
            return json.loads(fence.group(1))
        except json.JSONDecodeError:
            pass
    
    # 2. whole string
    try:
        return json.loads(text.strip())
    except json.JSONDecodeError:
        pass
    
    # 3. first balanced {...}
    start = text.find("{")
    if start != -1:
        depth = 0
        for i, ch in enumerate(text[start:], start):
            if ch == "{":
                depth += 1
            elif ch == "}":
                depth -= 1
                if depth == 0:
                    try:
                        return json.loads(text[start:i + 1])
                    except json.JSONDecodeError:
                        break
    
    raise LLMParseError(f"no JSON object found in response: {text[:200]!r}")
```

### 三段降级（v1 也做了类似的事，但 v2 更紧凑）

#### 1. **Markdown 代码块 fence**

```
```json
{"signal": "bullish", ...}
```
```

匹配 ` ```json{...}``` ` 或 ````{...}``` `。**`re.DOTALL` 让 `.` 匹配换行**。

#### 2. **整段就是 JSON**

LLM 有时直接输出 `{"signal": "bullish", ...}` 而不带 fence。

#### 3. **找第一个平衡的 `{...}`**

LLM 输出 `"Here's my analysis:\n\n{...json...}\n\nLet me know if..."` 时，前两段都失败，第三段能找到。

**手写平衡匹配的逻辑**：

```python
depth = 0
for i, ch in enumerate(text[start:], start):
    if ch == "{":
        depth += 1
    elif ch == "}":
        depth -= 1
        if depth == 0:    # 回到 0 = 找到一个完整的 top-level {...}
            ...
```

**不处理**：
- 字符串里的 `{` `}`（JSON 里有 `"key": "{value}"` 时会算错）——但作为**降级策略**够用
- 嵌套的 JSON object 在 JSON 字符串里——同上

#### 失败抛 `LLMParseError`

```python
raise LLMParseError(f"no JSON object found in response: {text[:200]!r}")
```

**带 raw text 头 200 字符**——debug 时直接看到 LLM 输出了什么。

## 与其他模块的关系

```
v2/llm/client.py::AnthropicLLM
       ↓ (构造)
v2/signals/llm_agent.py::LLMAgent.__init__
       ↓ (调用 complete())
ChatAnthropic.invoke([(system, msg), (human, msg)])
       ↓ (返回 content list / str)
self._chat.invoke(...)
       ↓ (处理 content)
return content
       ↓ (传给 extract_json)
v2/llm/client.py::extract_json
       ↓ (返回 dict)
LLMAgent._parse() 验证 signal/confidence 字段
       ↓ (构造 Signal)
Signal(metadata={...})
```

## 设计取舍

### 1. 不用 langchain `with_structured_output`

✅ **手写 prompt + JSON + 自己 parse**：

- 跨模型兼容（不同模型对 tool_use 支持不同）
- reasoning 模型不破坏（forced-tool 在 reasoning 模型上会坏）
- 调试透明（你可以看到完整 prompt 和 raw response）

❌ **代价**：要自己处理 JSON 解析的边界 case

**判断**：合理。**LLM 输出本来就不可靠，自己 parse 更可控**。

### 2. `LLMClient.model: str` 必须是字段

```python
class LLMClient(Protocol):
    model: str
    def complete(self, system, user) -> str: ...
```

- ✅ `prompt_key` 需要 model 字段（v2/llm/cache.py）
- ❌ 不是所有 LLM provider 都有"模型名"概念（如自训练模型？）

**判断**：合理。**v2 假设的 LLM 是 commercial API 模型，都有 model name**。

### 3. `complete` 不返回 token 用量

- ❌ 不能统计 LLM 成本
- ✅ API 简单

**判断**：合理。**成本统计可以在 `LLMAgent.predict` 层面加 hook**，不必污染 provider 接口。

## 对 fork → CN 版本的启示

### A. 加 `DeepSeekLLM` 很简单

```python
class DeepSeekLLM:
    model = "deepseek-chat"
    
    def __init__(self, model=None, ...):
        api_key = os.getenv("DEEPSEEK_API_KEY")
        if not api_key:
            raise ValueError("DEEPSEEK_API_KEY not found...")
        from langchain_openai import ChatOpenAI   # DeepSeek 是 OpenAI 兼容
        
        self.model = model or os.getenv("V2_LLM_MODEL", "deepseek-chat")
        self._chat = ChatOpenAI(
            model=self.model,
            api_key=api_key,
            base_url="https://api.deepseek.com/v1",
            ...
        )
    
    def complete(self, system, user):
        result = self._chat.invoke(...)
        return result.content
```

**完全复制 `AnthropicLLM` 的结构，改 4 处**：

1. env var name（`DEEPSEEK_API_KEY`）
2. 默认 model
3. `ChatAnthropic` → `ChatOpenAI`
4. 加 `base_url`（DeepSeek 兼容 OpenAI 但 base_url 不同）

### B. CN LLM 特有的坑

1. **DeepSeek 返回 content 直接是字符串**（不像 Anthropic 有时是 list）——所以 `if isinstance(content, list)` 那段**对 DeepSeek 是无害的**（只走 false 分支直接返回 string）
2. **DeepSeek 中文输出**——`extract_json` 三段降级**对中文 content 同样适用**（不依赖语言）
3. **Moonshot/Qwen** 同理

### C. extract_json 的中文 robustness

LLM 在中文 prompt 下可能输出：

```
好的，根据您的要求，我的分析如下：

```json
{
  "signal": "bullish",
  "confidence": 75,
  "reasoning": "公司护城河深，估值合理"
}
```

希望对您有帮助！
```

**三段降级全部能处理**：

- 段 1：匹配 `` ```json{...}``` `` ✓
- 段 2：如果 LLM 整段只输出 JSON ✓
- 段 3：找 `{...}` 平衡块 ✓

**不需要改 `extract_json`**。

## 深读问题（自检）

- [ ] `DEFAULT_MODEL = "claude-sonnet-5"` ——这个模型名真的存在吗？（提示：截至 2026-07，Anthropic 的模型家族是什么？）
- [ ] `extract_json` 的第三段（找平衡块）有没有 bug？（提示：想想 JSON 字符串里嵌入 `{` 的情况）
- [ ] `complete` 失败时（网络/auth/timeout），`AnthropicLLM` 现在的行为是 raise——LLMAgent 收到 exception 会怎么做？（提示：看 `llm_agent.py::_predict` 的 try/except）
- [ ] 如果你想加 DeepSeekLLM，需要改 `v2/signals/llm_agent.py` 吗？还是只动 `v2/llm/client.py`？

---

*解析时间：2026-07-12 · 第四轮迭代*
*下次解析目标：`v2/features/snapshot.py` + `v2/signals/base.py`*