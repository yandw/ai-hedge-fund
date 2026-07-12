# v2/data/client.py 深度解析

> 文件：`v2/data/client.py` · 277 行 · `FDClient` + `FDClientError`
> 地位：**v2 默认数据源实现**——把 Financial Datasets HTTP API 包成结构化、强类型、带语义错误的 Python 类

## 文件作用一句话

**用 `requests.Session` 调 Financial Datasets REST API，把 HTTP 错误码翻译成 Python 异常，把 JSON 响应转成 v2/data/models 里的 Pydantic 模型。**

## 两个类

### `FDClientError`（line 24-33）—— 自定义异常

```python
class FDClientError(Exception):
    """An API request failed for infrastructure reasons (auth, rate limit,
    server error, network). Distinct from "no data exists" — that returns
    empty. A backtest must crash on this, not treat it as no-data.
    """
    def __init__(self, message, *, status_code=None, path=None):
        super().__init__(message)
        self.status_code = status_code
        self.path = path
```

**关键设计**：

1. **区别于"无数据"**——`FDClientError` 只在**基础设施失败**时抛出
2. **携带 `status_code` 和 `path`**——调试时能立刻看出"哪个 endpoint、什么 HTTP 状态"
3. **继承 `Exception`**（不是 `RuntimeError` / `IOError`）——故意用宽泛基类，让上层捕获时不绑定特定类型

### `FDClient`（line 36-276）—— 主类

**构造**（line 48-56）：

```python
def __init__(self, api_key=None, timeout=30.0):
    self._api_key = api_key or os.environ.get("FINANCIAL_DATASETS_API_KEY", "")
    self._timeout = timeout
    self._session = requests.Session()
    self._session.headers["X-API-Key"] = self._api_key
```

- API key 从参数或环境变量拿
- 用 `requests.Session` 而不是 `requests.get`——复用 TCP 连接 + 默认 headers
- `X-API-Key` 是 FD 的认证 header 风格

**8 个 public 方法对应 Protocol 的 8 个方法**：

| 方法 | 行号 | 关键参数 / 行为 |
|------|------|----------------|
| `get_prices` | 76-92 | `/prices/`，支持 `interval` / `interval_multiplier` |
| `get_financial_metrics` | 98-120 | `/financial-metrics/`，**强制 `filing_date_lte`** |
| `get_news` | 126-138 | `/news/` |
| `get_insider_trades` | 144-156 | `/insider-trades/`，**强制 `filing_date_lte`** |
| `get_company_facts` | 162-168 | `/company/facts/`（**注意：是 `_request` 不是 `_get`**） |
| `get_earnings` | 174-180 | `/earnings/`（latest） |
| `get_earnings_history` | 182-197 | `/earnings/`（history） |
| `get_market_cap` | 203-211 | **组合**：先 facts 后 metrics |
| `get_market_cap` 内部 | 205-211 | 自己组合 facts + metrics，无独立 endpoint |

## 核心私有方法：`_request`（line 229-276）—— fail-loud 的灵魂

```python
def _request(self, method, path, **kwargs):
    url = self.BASE_URL + path
    for attempt, delay in enumerate((*self._RETRY_DELAYS, None)):
        try:
            resp = self._session.request(method, url, timeout=self._timeout, **kwargs)
        except requests.RequestException as exc:
            raise FDClientError(f"{method} {path} failed: {exc}", path=path) from exc
        
        if resp.status_code == 429 and delay is not None:
            logger.info("Rate limited (429), retrying in %ds (attempt %d/%d)", 
                        delay, attempt + 1, len(self._RETRY_DELAYS))
            time.sleep(delay)
            continue
        
        if resp.status_code == 404:
            return None
        
        if resp.status_code >= 400:
            raise FDClientError(
                f"{method} {path} returned {resp.status_code}: {resp.text[:200]}",
                status_code=resp.status_code, path=path,
            )
        
        return resp
    
    raise FDClientError(
        f"{method} {path} rate limited (429) after {len(self._RETRY_DELAYS)} retries",
        status_code=429, path=path,
    )
```

**这段代码是整个文件最重要的部分**。逐行解：

### 1. 重试机制（line 244-260）

```python
_RETRY_DELAYS = (5, 15, 30)    # line 46

for attempt, delay in enumerate((*self._RETRY_DELAYS, None)):
    # 4 次迭代：delay = 5, 15, 30, None
```

- **429 重试 3 次**：每次等 5/15/30 秒
- **第 4 次 `delay=None`**：表示不再重试，抛错
- **指数退避**（5→15→30）：不是严格的指数，但保证间隔递增

### 2. 三种 status_code 分支

```python
if resp.status_code == 429 and delay is not None:    # 限流 → 重试
    ...
if resp.status_code == 404:                            # 找不到 → 返 None（"无数据"）
    return None
if resp.status_code >= 400:                            # 其他错误 → raise
    raise FDClientError(...)
```

**404 → None 是关键设计**：

- **HTTP 404** 在金融数据 API 里通常意味着"这个 ticker/period 没有数据"
- 不是失败，是事实
- 与 `[]` 同义但更明确（"单个对象不存在"用 None，"列表为空"用 []）

### 3. 网络异常 raise

```python
except requests.RequestException as exc:
    raise FDClientError(f"{method} {path} failed: {exc}", path=path) from exc
```

**`from exc`**：保留原异常栈，调试时能看完整 traceback。

## 关键方法的实现细节

### `get_financial_metrics`（line 98-120）—— Point-in-time 的强制点

```python
def get_financial_metrics(self, ticker, end_date, period="ttm", limit=10):
    data = self._get("/financial-metrics/", {
        "ticker": ticker,
        "filing_date_lte": end_date,        # ← 关键
        "period": period,
        "limit": limit,
    }, response_key="financial_metrics")
    return [FinancialMetrics(**row) for row in data] if data else []
```

**`filing_date_lte=end_date`**：

- 这是 API 端的过滤参数
- **保证 server 只返回 SEC 受理日 ≤ end_date 的数据**
- 上层（alpha model）拿到的所有数据都"在 end_date 当天之前已经公开"

**对比 v1**：v1 `src/tools/api.py::get_financial_metrics` 调 FD 时**没有**传 `filing_date_lte`——这是 v2 修的"lookahead leak"。

### `get_company_facts`（line 162-168）—— 不用 `_get` 用 `_request`

```python
def get_company_facts(self, ticker: str) -> CompanyFacts | None:
    resp = self._request("GET", "/company/facts/", params={"ticker": ticker})
    if resp is None:
        return None
    facts_data = resp.json().get("company_facts")
    return CompanyFacts(**facts_data) if facts_data else None
```

**为什么不用 `_get`**：因为返回 JSON 的根 key 是 `company_facts`（嵌套对象），不是 list。`_get` 假设 `response_key` 指向数组。

**取舍**：写一个特例函数 vs 改 `_get` 接受复杂结构——选了前者，因为 company_facts 是**唯一**这种结构的 endpoint。

### `get_market_cap`（line 203-211）—— 组合实现

```python
def get_market_cap(self, ticker, end_date) -> float | None:
    facts = self.get_company_facts(ticker)
    if facts is not None and facts.market_cap is not None:
        return facts.market_cap
    metrics = self.get_financial_metrics(ticker, end_date, limit=1)
    if metrics and metrics[0].market_cap is not None:
        return metrics[0].market_cap
    return None
```

**注意**：

- **没有独立 endpoint**——market_cap 是从 company_facts 或 financial_metrics 里抽出来的
- **fallback 链**：facts → metrics → None
- **没有 PIT 强制**（因为 `end_date` 只传给 metrics 那个分支）

## 重试与退避的细节

```python
_RETRY_DELAYS = (5, 15, 30)

for attempt, delay in enumerate((*self._RETRY_DELAYS, None)):
    if resp.status_code == 429 and delay is not None:
        time.sleep(delay)
        continue
    ...
raise FDClientError(... rate limited (429) after 3 retries ...)
```

**实际行为**：

| 尝试次数 | delay | 行为 |
|---------|-------|------|
| 1 | 5s | 失败等 5s |
| 2 | 15s | 失败等 15s |
| 3 | 30s | 失败等 30s |
| 4 | None | 不等了，raise |

**总等待时间上限**：5+15+30 = 50 秒。

**其他错误（5xx）不重试**——只重试 429（限流）。这是一个**取舍**：

- ✅ 简单，不会因为服务端 bug 反复打
- ❌ 5xx 瞬时错误也会让请求失败

## 设计取舍

### 1. 429 重试 vs 5xx 重试？

✅ **只重试 429**：

- 429 是可预期的（限流是反爬机制）
- 5xx 通常是 server bug，反复打可能加剧

**判断**：合理。生产级代码常这么做。

### 2. `_get` vs `_request` 两个 helper？

```python
def _get(self, path, params, response_key):       # 用于 list endpoint
    resp = self._request("GET", path, params=params)
    if resp is None: return None
    return resp.json().get(response_key)

def _request(self, method, path, **kwargs):       # 用于任意 endpoint
    # ... 重试 + error 处理
    return resp    # 或 None / raise
```

✅ **两个 helper**——`._get` 是 list endpoint 的快捷方式（80% 情况），`._request` 是底层（100% 情况）。

### 3. `requests.Session` vs 每次 `requests.get`？

✅ **Session**——TCP 连接复用、headers 复用、cookie 状态保持。

### 4. 异常类型：`Exception` vs `IOError` / `RuntimeError`？

✅ **`Exception`**——上层 `try/except` 时不绑定特定家族。

## 对 fork → CN 版本的启示

### A. `TushareClient` / `AKShareClient` 的实现骨架

**关键差异**：

| 维度 | FDClient | TushareClient |
|------|----------|---------------|
| HTTP | REST + `requests` | Python SDK |
| 重试 | 429 重试 | Tushare 内部重试 |
| PIT 字段 | `filing_date_lte` | `ann_date_lte` |
| 错误处理 | raise FDClientError | 应该 raise 同类异常 |
| 返回 | Pydantic models | 需映射成 Pydantic |

**建议**：CN client **继承同样的语义**——raise 同类异常 / 返回空列表 vs None 的语义保持一致。

### B. fail-loud 必须坚持

即使 Tushare SDK 默认返回空（错误吞掉），**你的 wrapper 必须 raise**。否则 v2 backtest 看到的"无信号"可能是真的没数据，也可能是 Tushare 服务挂了——不可区分。

### C. `_get` / `_request` 的模式可以复用

CN client 不需要 HTTP 层（Tushare 是 Python SDK），但**应该保留两层结构**：
- 底层：调 SDK，处理 SDK 异常 → translate 成 `FDClientError`（或新异常类）
- 上层：8 个方法对应 8 个 Protocol 方法

## 深读问题（自检）

- [ ] `_RETRY_DELAYS = (5, 15, 30)` 加起来 50 秒——50 秒内 429 都没解开，**这够不够**？什么场景下需要调？
- [ ] `get_market_cap` 没强制 PIT——这在回测里是 bug 吗？（提示：想想 market_cap 在 alpha model 里怎么用）
- [ ] 401（auth 失败）现在会走 `>= 400` 分支 raise——**该不该单独处理**？（比如启动时立刻报错，而不是回测中途崩）
- [ ] 如果写 `TushareClient`，Tushare 的积分用完了 SDK 会怎样？你的 wrapper 怎么翻译？

---

*解析时间：2026-07-12 · 第二轮迭代*
*下次解析目标：`v2/data/cached.py`（缓存包装器）+ `v2/llm/client.py`（LLM provider）*