# v2/data/cached.py 深度解析

> 文件：`v2/data/cached.py` · 158 行 · 1 个类
> 地位：**通用缓存装饰器**——把任意 `DataClient` 包一层磁盘缓存，让 backtest 重跑零成本

## 文件作用一句话

**给任意 `DataClient` 加一层"按方法名+参数哈希"的磁盘 JSON 缓存**，只缓存成功响应，保留 fail-loud 语义。

## 核心模式：Decorator

```python
fd = FDClient()                              # 真实 HTTP 调用
cached = CachedDataClient(fd)                 # 包一层缓存
prices = cached.get_prices("AAPL", ...)      # 第一次：HTTP
prices = cached.get_prices("AAPL", ...)      # 第二次：磁盘
```

**这是一个标准的 Decorator 模式应用**——`CachedDataClient` 实现了完整的 `DataClient` Protocol（8 个方法都重写），所以**它本身也是一个合法的 DataClient**，可以无限嵌套。

## 8 个公开方法（line 54-107）

每个方法都遵循同一个模式：

```python
def get_prices(self, ticker, start_date, end_date, interval="day", interval_multiplier=1):
    return self._cached_list(
        "get_prices", Price,
        {"ticker": ticker, "start_date": start_date, "end_date": end_date,
         "interval": interval, "interval_multiplier": interval_multiplier},
        lambda: self._client.get_prices(
            ticker, start_date, end_date, interval, interval_multiplier),
    )
```

**4 个关键参数**：

1. **方法名字符串**（`"get_prices"`）——作为 cache key 的一部分
2. **Pydantic model 类**（`Price`）——用于反序列化缓存里的 dict
3. **参数 dict**——组成 cache key
4. **fetch lambda**——真正调下层的方法

## 三个内部 helper：处理三种返回类型

| Helper | 返回类型 | 适用方法 | 序列化方式 |
|--------|---------|---------|-----------|
| `_cached_list` | `list[T]` | get_prices, get_news, get_insider_trades, get_earnings_history, get_financial_metrics | `[{...}, {...}]` |
| `_cached_item` | `T \| None` | get_company_facts, get_earnings | `{...}` 或 `null` |
| `_cached_scalar` | `float \| None` | get_market_cap | `1.5` 或 `null` |

**为什么不统一成一个 helper**？

每个返回类型的序列化/反序列化不同（list vs dict vs scalar），分成 3 个让每个 helper 的代码都 ≤ 7 行，更清晰。

## Cache key 设计

```python
def _key(self, method: str, params: dict) -> str:
    canonical = json.dumps(params, sort_keys=True)
    return hashlib.sha256(f"{method}|{canonical}".encode()).hexdigest()[:24]
```

**3 个细节**：

1. **`sort_keys=True`**——保证 `{"a": 1, "b": 2}` 和 `{"b": 2, "a": 1}` 生成同样 key
2. **`f"{method}|{canonical}"`**——method name 和 params 都进 key
3. **截取前 24 位 hex**——节省文件名长度

**key 示例**：

```
方法: get_prices
参数: {"ticker": "AAPL", "start_date": "2024-01-01", "end_date": "2024-12-31", ...}
↓
SHA256("get_prices|{...}")[:24]
↓
例如: a1b2c3d4e5f6g7h8i9j0k1l2
```

→ 文件名 `a1b2c3d4e5f6g7h8i9j0k1l2.json` 在 `.v2_cache/data/`

## 关键设计：只缓存成功响应

```python
def _cached_list(self, method, model_cls, params, fetch):
    key = self._key(method, params)
    hit = self._read(key)
    if hit is not None:
        return [model_cls(**row) for row in hit["data"]]
    result = fetch()                                    # 这一步会抛错（如果下层 raise）
    self._write(key, {"data": [r.model_dump() for r in result]})   # 成功后写缓存
    return result
```

**重点**：`_write` 只在 `fetch()` 成功后执行。如果 `fetch()` 抛 `FDClientError`：

- ❌ **不写缓存**
- ❌ **不吞异常**——直接往上抛
- ✅ 下次相同请求**仍然走 HTTP**

**为什么重要**：

如果缓存了"网络瞬时失败的空响应"，下次 backtest 看到"无数据"会判定"无信号"——这其实是数据层失败被误读成"中性"。**这正是 v2 fail-loud 哲学要避免的**。

## `refresh=True` 模式

```python
def __init__(self, client, cache_dir=DEFAULT_CACHE_DIR, refresh=False):
    ...
    self._refresh = refresh

def _read(self, key):
    if self._refresh:
        return None                    # 强制 miss，每次都 fetch
    path = self._dir / f"{key}.json"
    ...
```

**两种用法**：

```bash
# 默认：命中缓存就不重抓
cached = CachedDataClient(fd)

# 强制刷新：忽略现有缓存，重新抓（并覆盖）
cached = CachedDataClient(fd, refresh=True)
```

**注意**：`refresh=True` 时**仍然会写新缓存**（line 138）——刷新完就更新。

## 缓存目录设计

```python
DEFAULT_CACHE_DIR = Path(".v2_cache/data")
```

- `.v2_cache/data/` —— 数据层缓存
- `.v2_cache/llm/` —— LLM 层缓存（见 `v2/llm/cache.py`）
- `.v2_cache/` 整体在 `.gitignore` 里（项目根 `.gitignore` line 70）

**好处**：

- 一次 warm cache → 之后完全离线
- 删除 `.v2_cache/` 即可重置
- git 仓库保持干净

## 损坏缓存的处理

```python
def _read(self, key):
    path = self._dir / f"{key}.json"
    if not path.exists():
        return None
    try:
        return json.loads(path.read_text())
    except (json.JSONDecodeError, OSError):
        return None    # corrupt entry -> miss; rewritten on fetch
```

**两种损坏情况**：

1. `JSONDecodeError` —— JSON 解析失败（文件被手动改坏？）
2. `OSError` —— 文件系统错（权限？磁盘满？）

**处理**：静默当 miss 处理，下次 fetch 后重写。

**判断**：合理。**损坏的缓存不应该让回测崩溃**。

## 嵌套的可能性

```python
fd = FDClient(api_key=...)
cached1 = CachedDataClient(fd, cache_dir=".v2_cache/data")     # 磁盘缓存
logged = LoggingClient(cached1)                                 # 日志包装
retried = RetryingClient(logged, max_tries=3)                   # 重试包装
final: DataClient = retried                                     # 任意层叠
```

**因为 `CachedDataClient` 满足 `DataClient` Protocol**，可以无限嵌套。生产环境常见组合：

```
真实数据源 → 重试层 → 日志层 → 缓存层 → 业务代码
```

## 设计取舍

### 1. 缓存粒度：方法级 vs 请求级

✅ **方法级**——每个 `(method, params)` 一条记录

- ✅ 简单
- ❌ 不能在 API 响应里"部分缓存"（比如同 ticker 不同 end_date 不能共享数据）

**判断**：合理。**金融数据的时间维度本身就是 key**，共享反而会破坏 PIT 语义。

### 2. JSON 序列化 vs pickle

✅ **JSON**：

- ✅ 可读、可调试、跨语言
- ❌ 不能存 `datetime` / `Decimal`（需先转字符串）

**判断**：合理。Pydantic 的 `model_dump()` 已经处理了 datetime → ISO string。

### 3. key 截取 24 hex chars

SHA256 完整是 64 hex chars。截前 24 → **碰撞概率 ~1/16M**——对个人 backtest 足够。

**判断**：合理。Git commit short hash 也是 7 位。

## 对 fork → CN 版本的启示

### A. `CachedDataClient` 不用改

任何 `DataClient` 实现（包括未来的 `TushareClient`、`AKShareClient`）都能直接被 `CachedDataClient` 包装——**缓存逻辑通用**。

### B. CN 数据源的缓存键

Tushare 的某些接口会用 `trade_date` 或 `ann_date` 作参数。**这些字段必须进 cache key**（已经在 `_key` 的 params dict 里），所以不会串数据。

### C. 缓存目录建议

CN 数据 + LLM 也走 `.v2_cache/{data,llm}/`——但**Tushare 的 API 调用本身可能要加 rate-limit-aware 缓存**（避免被 Tushare 限流）。

## 深读问题（自检）

- [ ] `_cached_list` 把 Pydantic model 用 `model_dump()` 转 dict 存盘，再 `model_cls(**row)` 还原——如果有自定义 validator，Pydantic v2 会重跑吗？
- [ ] `_key` 截取 24 位 hex，**理论上会碰撞**——碰到碰撞的概率大不大？需要拉长吗？
- [ ] 为什么 `get_market_cap` 用 `_cached_scalar` 而不是 `_cached_item`？（提示：float 不是 Pydantic model）
- [ ] 如果你想加一个"缓存过期"机制（比如 7 天后自动重抓），最干净的加法是什么？

---

*解析时间：2026-07-12 · 第三轮迭代*
*下次解析目标：`v2/llm/cache.py` + `v2/llm/client.py`*