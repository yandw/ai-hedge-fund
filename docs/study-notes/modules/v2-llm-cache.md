# v2/llm/cache.py 深度解析

> 文件：`v2/llm/cache.py` · 47 行 · 1 个函数 + 1 个类
> 地位：**LLM 决策的"持久化 + 缓存 + 调试"三合一记录层**——一次 LLM 调用 = 一个 JSON 文件

## 文件作用一句话

**把每一次 LLM 决策（prompt + response + 解析结果）持久化到磁盘 JSON 文件**，实现三件事：①backtest 重跑零成本 ②事后审计每个观点的来源 ③parse 失败也能复盘。

## 关键设计哲学（来自 docstring line 4-11）

```python
"""Prompt cache — one JSON file per LLM decision.

This is deliberately three things at once (a locked design decision):
1. a cache: a backtest re-running an agent over an unchanged snapshot costs $0;
2. the persistence record: the EXACT prompt + response behind every Signal,
   for replay and audit;
3. the debug trail: failed parses keep the raw response on disk.
"""
```

**三合一设计**：

| 用途 | 文件名 | 内容 |
|------|--------|------|
| 缓存 | `{key}.json` | parsed JSON 对象（response 已被解析） |
| 审计 | `{key}.json` | 完整 system + user + raw response + parsed |
| 调试 | `{key}.json` | parse 失败时也存（带 `parse_error` 字段） |

**为什么不分成三个文件**？

✅ 一个文件 = 一个决策——查找简单（`ls .v2_cache/llm/` 看一眼就知道有多少决策）  
✅ 写一次原子写（不会出现"缓存有但审计没写完"的中间态）  
❌ 文件较大（带 system + user + response，可能几十 KB）——但 LLM 决策量本来就小（一次 backtest 几百到几千次）

## 核心：`prompt_key` 函数

```python
def prompt_key(agent: str, model: str, system: str, user: str) -> str:
    """Cache key for one (agent, model, prompt) combination."""
    payload = f"{agent}|{model}|{system}|{user}"
    return hashlib.sha256(payload.encode()).hexdigest()[:24]
```

**4 个字段进 key**：

1. `agent` —— persona 名字（"buffett"）
2. `model` —— 实际用的模型（"claude-sonnet-5"）
3. `system` —— system prompt
4. `user` —— user prompt

**为什么 model 进 key**：

- **同一个 prompt 给两个不同模型会产生不同回答**
- 把 model 进 key 让 cache 自动按模型区分
- **避免"换了模型还拿到旧模型的答案"的隐蔽 bug**

**截取前 24 hex chars**——和 `CachedDataClient._key` 一致（Git short SHA 风格），碰撞概率 ~1/16M。

## `PromptCache` 类（47 行）

```python
class PromptCache:
    def __init__(self, cache_dir=DEFAULT_CACHE_DIR):
        self._dir = Path(cache_dir)

    def get(self, key) -> dict | None:
        path = self._dir / f"{key}.json"
        if not path.exists():
            return None
        try:
            return json.loads(path.read_text())
        except (JSONDecodeError, OSError):
            return None    # corrupt → miss, 会被覆盖

    def put(self, key, record):
        self._dir.mkdir(parents=True, exist_ok=True)
        record = {**record, "created_at": datetime.now(timezone.utc).isoformat()}
        path = self._dir / f"{key}.json"
        path.write_text(json.dumps(record, indent=2))
```

**两个方法**：

- `get(key) -> dict | None` —— 读完整 JSON
- `put(key, record)` —— 写入（**自动加 `created_at` 时间戳**）

**API 极简**——故意只暴露 `get` / `put`，不提供 list/delete/iterate 等"高级"操作。LLM agent 是唯一调用方，**调用约定由 agent 自己保证**。

## 实际写入的 JSON 长什么样

参考 `v2/signals/llm_agent.py::LLMAgent.predict`（已解析）的写入：

```json
{
  "agent": "buffett",
  "model": "claude-sonnet-5",
  "ticker": "AAPL",
  "as_of": "2024-06-01",
  "snapshot_hash": "a1b2c3d4e5f6...",
  "system": "You are Warren Buffett. Decide bullish, bearish, or neutral...",
  "user": "Ticker: AAPL\nFacts:\n{...}",
  "response": "<raw LLM output text>",
  "parsed": {
    "signal": "bullish",
    "confidence": 75,
    "reasoning": "Durable moat, fair price, consistent ROE..."
  },
  "created_at": "2026-07-12T18:23:45.123456+00:00"
}
```

**如果 parse 失败**：

```json
{
  "agent": "buffett",
  "model": "claude-sonnet-5",
  "ticker": "AAPL",
  ...
  "response": "<malformed output>",
  "parse_error": "no JSON object found in response: '...'",
  "created_at": "2026-07-12T..."
}
```

**注意**：`parse_error` 字段存在 = parse 失败。`llm_agent.py` 的代码逻辑：

```python
try:
    parsed = self._parse(response)
except Exception as exc:
    self._cache.put(key, {**record, "parse_error": str(exc)})   # 仍然写！
    return self._abstain(ticker, date, f"parse failed: {exc}")
```

**parse 失败也写磁盘**——这就是"debug trail"的实现：复盘时直接看 raw response + parse_error 就能定位问题。

## Cache key 与 Pydantic snapshot 的关系

```python
# v2/signals/llm_agent.py
snapshot = self.build_snapshot(ticker, date, data_client)
key = prompt_key(self.name, self._llm.model, system, user)

cached = self._cache.get(key)
if cached is not None and "parsed" in cached:
    return self._to_signal(...)        # 命中：直接返回，不再调 LLM
```

**为什么用 SHA256(user_prompt) 而不用 `snapshot.content_hash`**？

`prompt_key` 的 user 参数就是 `snapshot.render()` 的结果——**render 已经把 snapshot 的内容融进去了**（hash 一致 → 内容一致）。所以**两者是等价的，但 prompt_key 把 model 也带进去了**，更安全。

## 与 `v2/data/cached.py` 的对比

| 维度 | `v2/data/cached.py` | `v2/llm/cache.py` |
|------|---------------------|-------------------|
| 包装对象 | DataClient | LLM 决策 |
| Cache key | `(method, params)` | `(agent, model, system, user)` |
| 存储格式 | 简化（只存 data） | 完整（存 raw + parsed + error） |
| 用途 | 性能（避免重抓） | 审计 + 性能 + 调试 |
| 目录 | `.v2_cache/data/` | `.v2_cache/llm/` |

**关键区别**：`data` cache 只关心"返回的数据本身"，`llm` cache 关心"决策的完整上下文"——后者**丢了 raw response 就丢了审计价值**。

## 设计取舍

### 1. JSON vs pickle vs sqlite

✅ **JSON**：

- 可读（编辑器直接打开看）
- 跨语言
- 一个决策一个文件，ls 即查

❌ **不能索引、不能按 ticker 查**——要查所有 AAPL 决策需要 grep 文件名

**判断**：合理。**审计性优先于查询性能**。

### 2. 一个 JSON 文件 vs SQLite 一行

✅ **JSON 文件**：

- ✅ 简单、无依赖
- ✅ 看一眼就知道有几个决策
- ❌ 文件多（1000 个 LLM 决策 = 1000 个文件）

**判断**：合理。LLM 决策量在 backtest 里可控（几千次级别）。

### 3. `created_at` 自动加

```python
record = {**record, "created_at": datetime.now(timezone.utc).isoformat()}
```

- ✅ 调用方不用记得加
- ❌ 写文件时调用方时间被覆盖

**判断**：合理。**审计的时间是"写入磁盘的时间"**，不是"LLM 实际响应的时间"（后者来自 response metadata，暂未记录）。

### 4. corrupt cache 静默当 miss

```python
except (JSONDecodeError, OSError):
    return None    # corrupt → miss, 会被覆盖
```

- ✅ 不让损坏文件阻塞 backtest
- ❌ 没有告警

**判断**：合理。可以加 logger.warning 但目前没加。

## 对 fork → CN 版本的启示

### A. CN LLM 用同一个 `PromptCache`

```python
# v2/llm/client.py 加 DeepSeekLLM 后
class DeepSeekLLM:
    model = "deepseek-chat"
    def complete(self, system, user): ...

# 在 LLMAgent 里
self._llm = llm if llm is not None else AnthropicLLM()  # 改成 DeepSeekLLM() 也行
```

**完全不动 `cache.py`**——`prompt_key` 已经把 model 进 key 了，换模型自动隔离缓存。

### B. CN 特有的审计挑战

- **CN 模型可能输出更长 reasoning**（中文比英文密度高）——JSON 文件会更大，但还能读
- **CN LLM 可能输出额外解释文字包着 JSON**——`extract_json` 的三段降级能处理
- **审计时需要看中文 reasoning**——可以加 `lang` 字段到 record 方便筛选

### C. 想加 query 能力？

如果将来想"查所有 AAPL 的 buffett 决策"，最干净的方法不是改 cache，而是**在 record 里加 ticker 索引**（已经有了：`"ticker": "AAPL"`）。

更高级的方案：另起一个 `v2/llm/ledger.py`，把已解析的 Signal 写入 SQLite，按 ticker/index 索引——cache 保持纯文件。

## 深读问题（自检）

- [ ] `prompt_key` 包含 4 个字段（agent/model/system/user），**如果 system prompt 里某天加了一个时间戳**（比如 `"Today is {date}"`），cache 还能命中吗？（提示：看 user 是否随之变化）
- [ ] `created_at` 是"写入磁盘的时间"，不是"LLM 响应时间"——这对审计够用吗？需要加 `responded_at` 吗？
- [ ] corrupt cache 静默当 miss——如果你怀疑某个 cache 文件被破坏了，怎么 debug？
- [ ] 假设你 backtest 1000 次 PEAD 决策，每个 LLM 决策 10KB JSON，**1000 个文件 = 10MB**——这规模下还有性能问题吗？什么时候需要切到 SQLite？

---

*解析时间：2026-07-12 · 第三轮迭代*
*下次解析目标：`v2/llm/client.py`*