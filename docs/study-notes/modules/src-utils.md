# src/utils/* 工具文件深度解析

> 文件：`src/utils/{llm,progress,display,api_key,visualize,ollama,docker}.py`
> 地位：**v1 的 7 个工具模块**——LLM 调用、进度条、显示、API key、可视化、Ollama、Docker

## 7 个工具概览

| 文件 | 行数 | 作用 |
|------|------|------|
| `llm.py` | ~200 | LLM 调用 + retry + JSON parse + 13 provider |
| `progress.py` | 117 | rich 实时进度条（多 agent 并行可视化） |
| `display.py` | ~200 | 终端格式化输出（trading output 表格） |
| `api_key.py` | ~50 | API key 从 state 抽取 |
| `visualize.py` | ~80 | LangGraph StateGraph → PNG |
| `ollama.py` | ~50 | Ollama 本地模型辅助 |
| `docker.py` | ~100 | Docker 容器辅助 |

## `src/utils/llm.py` —— LLM 调用 dispatcher

### 文件作用一句话

**统一 LLM 调用入口，支持 13 家 provider，retry 3 次，JSON 解析降级，Pydantic 结构化输出。**

### 主函数 `call_llm`（line 10-196）

```python
def call_llm(
    prompt,
    pydantic_model,
    agent_name=None,
    state=None,
    max_retries=3,
    default_factory=None,
) -> BaseModel:
    # 1. 拿 model 配置（从 state 或默认）
    if state and agent_name:
        model_name, model_provider = get_agent_model_config(state, agent_name)
    else:
        model_name = "gpt-4.1"
        model_provider = "OPENAI"
    
    # 2. 拿 API keys（从 state）
    api_keys = None
    if state:
        request = state.get("metadata", {}).get("request")
        if request and hasattr(request, 'api_keys'):
            api_keys = request.api_keys
    
    # 3. 构造 LLM 实例
    model_info = get_model_info(model_name, model_provider)
    llm = get_model(model_name, model_provider, api_keys)
    
    # 4. 决定用 langchain structured output 还是手写 JSON parse
    if not (model_info and not model_info.has_json_mode()):
        # has json mode → 用 langchain 的 with_structured_output
        llm = llm.with_structured_output(pydantic_model, method="json_mode")
    # else: 手写 parse（见 _extract_json_from_response）
    
    # 5. Retry 循环
    for attempt in range(max_retries):
        try:
            result = llm.invoke(prompt)
            
            # 非 JSON mode 模型：手动 parse
            if model_info and not model_info.has_json_mode():
                parsed_result = extract_json_from_response(result.content)
                if parsed_result:
                    return pydantic_model(**parsed_result)
            else:
                return result    # langchain 已经解析
        
        except Exception as e:
            if agent_name:
                progress.update_status(agent_name, None, f"Error - retry {attempt + 1}/{max_retries}")
            
            if attempt == max_retries - 1:
                # 最后一次也失败 → default_factory
                if default_factory:
                    return default_factory()
                return create_default_response(pydantic_model)
```

### 关键设计点

#### 1. **`model_name + model_provider` 从 state 拿**

```python
if state and agent_name:
    model_name, model_provider = get_agent_model_config(state, agent_name)
```

**意义**：每个 agent 可以**用不同的 LLM**——比如 Buffett 用 Claude，Sentiment 用 GPT。

**调用方**：

```python
# src/agents/warren_buffett.py
result = call_llm(
    prompt=...,
    pydantic_model=WarrenBuffettSignal,
    agent_name="warren_buffett_agent",
    state=state,
)
```

#### 2. **`has_json_mode()` 决定用哪种路径**

```python
# src/llm/models.py::LLMModel.has_json_mode()
def has_json_mode(self) -> bool:
    if self.is_deepseek() or self.is_gemini():
        return False
    if self.is_anthropic_reasoning():
        return False
    if self.is_ollama():
        return "llama3" in self.model_name or "neural-chat" in self.model_name
    if self.provider == ModelProvider.OPENROUTER:
        return True
    return True
```

**判定逻辑**：

| 模型 | JSON mode |
|------|-----------|
| DeepSeek | ❌ |
| Gemini | ❌ |
| Anthropic reasoning（`claude-fable-*`） | ❌ |
| Ollama llama3 / neural-chat | ✅ |
| 其他 | ✅ |

**与 v2 对比**：v2 **永远不用** `with_structured_output`——总是 prompt 里要 JSON + 手 parse。

**v1 的策略**：

- JSON 模式模型用 langchain 的 `with_structured_output`（**框架魔法**）
- 非 JSON 模式用手 parse

**v2 的改进**：v2 不用框架魔法，统一手 parse——更可控，跨模型一致。

#### 3. **3 次 retry**

```python
for attempt in range(max_retries):
    try:
        result = llm.invoke(prompt)
        ...
    except Exception as e:
        if attempt == max_retries - 1:
            if default_factory:
                return default_factory()
            return create_default_response(pydantic_model)
```

**3 次 retry + 失败 fallback 到 default response**。

**default_factory 例子**（warren_buffett.py line 817-818）：

```python
def create_default_warren_buffett_signal():
    return WarrenBuffettSignal(signal="neutral", confidence=50, reasoning="Insufficient data")

return call_llm(
    prompt=prompt,
    pydantic_model=WarrenBuffettSignal,
    agent_name=agent_id,
    state=state,
    default_factory=create_default_warren_buffett_signal,
)
```

**优雅降级**：LLM 失败 → 返回 "neutral, 50 confidence, 'Insufficient data'"。

#### 4. **`create_default_response` 自动生成**

```python
def create_default_response(model_class: type[BaseModel]) -> BaseModel:
    default_values = {}
    for field_name, field in model_class.model_fields.items():
        if field.annotation == str:
            default_values[field_name] = "Error in analysis, using default"
        elif field.annotation == float:
            default_values[field_name] = 0.0
        elif field.annotation == int:
            default_values[field_name] = 0
        elif hasattr(field.annotation, "__origin__") and field.annotation.__origin__ == dict:
            default_values[field_name] = {}
        else:
            if hasattr(field.annotation, "__args__"):
                default_values[field_name] = field.annotation.__args__[0]
            else:
                default_values[field_name] = None
    
    return model_class(**default_values)
```

**自动推断**：

| 字段类型 | 默认值 |
|---------|--------|
| `str` | "Error in analysis, using default" |
| `float` | 0.0 |
| `int` | 0 |
| `dict` | {} |
| `Literal["a", "b"]` | "a"（第一项） |
| 其他 | None |

**意义**：agent 不用自己写 default factory——`create_default_response` 通用。

#### 5. `extract_json_from_response` —— v1 版 JSON parse

```python
def extract_json_from_response(content) -> dict | None:
    # 1. ```json fence
    json_start = content.find("```json")
    if json_start != -1:
        json_text = content[json_start + 7:]
        json_end = json_text.find("```")
        if json_end != -1:
            json_text = json_text[:json_end].strip()
            try:
                return json.loads(json_text)
            except json.JSONDecodeError:
                pass
    
    # 2. ``` 块（无 json 标签）
    json_start = content.find("```")
    ...
    
    # 3. 整段 JSON
    try:
        return json.loads(content.strip())
    except json.JSONDecodeError:
        pass
    
    # 4. 第一个 { ... } 平衡块
    brace_start = content.find("{")
    if brace_start != -1:
        depth = 0
        for i, char in enumerate(content[brace_start:], brace_start):
            if char == "{":
                depth += 1
            elif char == "}":
                depth -= 1
                if depth == 0:
                    try:
                        return json.loads(content[brace_start:i + 1])
                    except json.JSONDecodeError:
                        break
    
    return None
```

**4 段降级**（比 v2 还多一段）：

1. ` ```json fence `
2. ```` ``` ```` 块（无 json 标签）
3. 整段 JSON
4. 平衡 `{...}`

**对比 v2**：

| 维度 | v1 | v2 |
|------|----|----|
| 降级路径 | 4 段 | 3 段 |
| 用 regex | ❌（find） | ✅（re.search） |
| 用法 | 顶层 fallback | LLM agent 内部 |

### v1 vs v2 LLM 调用对比

| 维度 | v1 `call_llm` | v2 `LLMAgent.predict` |
|------|--------------|---------------------|
| Provider 数量 | 13 家 | 1 家（Anthropic） |
| 重试 | 3 次 + default_factory | 1 次（无 retry） |
| 失败语义 | default response | abstain signal |
| Structured output | 混合（langchain + 手 parse） | 统一手 parse |
| 缓存 | ❌ | ✅ PromptCache |
| PIT 强制 | ❌ | ✅ |

**v2 简化了**：

- 1 个 provider（Anthropic）
- 不 retry（cache 命中率应该高）
- abstain 信号（更明确）
- 缓存优先

## `src/utils/progress.py` —— Rich 进度条

### 文件作用一句话

**多 agent 并行进度跟踪**——19 个 analyst 同时跑，每个有自己的状态行。

### `AgentProgress` 类（line 12-114）

**核心属性**：

```python
self.agent_status: Dict[str, Dict[str, str]] = {}    # 每个 agent 的状态
self.table = Table(show_header=False, box=None, padding=(0, 1))
self.live = Live(self.table, console=console, refresh_per_second=4)
self.update_handlers: List[Callable] = []
```

**4 个关键方法**：

#### `register_handler` / `unregister_handler`

```python
def register_handler(self, handler):
    self.update_handlers.append(handler)
    return handler
```

**支持装饰器风格**：

```python
@progress.register_handler
def my_handler(agent_name, ticker, status, analysis, timestamp):
    # 收到 agent 状态更新
    ...
```

#### `start` / `stop`

```python
def start(self):
    if not self.started:
        self.live.start()
        self.started = True

def stop(self):
    if self.started:
        self.live.stop()
        self.started = False
```

**保证 start/stop 配对**——`src/main.py` 用 `try/finally`。

#### `update_status`（核心）

```python
def update_status(self, agent_name, ticker=None, status="", analysis=None):
    if agent_name not in self.agent_status:
        self.agent_status[agent_name] = {"status": "", "ticker": None}
    
    if ticker:
        self.agent_status[agent_name]["ticker"] = ticker
    if status:
        self.agent_status[agent_name]["status"] = status
    if analysis:
        self.agent_status[agent_name]["analysis"] = analysis
    
    timestamp = datetime.now(timezone.utc).isoformat()
    self.agent_status[agent_name]["timestamp"] = timestamp
    
    for handler in self.update_handlers:
        handler(agent_name, ticker, status, analysis, timestamp)
    
    self._refresh_display()
```

**关键**：

- 更新 agent 状态 dict
- **通知所有注册的 handler**（让外部系统能订阅）
- **刷新显示**

#### `_refresh_display` —— 排序 + 渲染

```python
def _refresh_display(self):
    self.table.columns.clear()
    self.table.add_column(width=100)
    
    def sort_key(item):
        agent_name = item[0]
        if "risk_management" in agent_name:
            return (2, agent_name)
        elif "portfolio_management" in agent_name:
            return (3, agent_name)
        else:
            return (1, agent_name)
    
    for agent_name, info in sorted(self.agent_status.items(), key=sort_key):
        status = info["status"]
        ticker = info["ticker"]
        if status.lower() == "done":
            style = Style(color="green", bold=True)
            symbol = "✓"
        elif status.lower() == "error":
            style = Style(color="red", bold=True)
            symbol = "✗"
        else:
            style = Style(color="yellow")
            symbol = "⋯"
        
        ...
        self.table.add_row(status_text)
```

**排序规则**：
1. analyst 排第一组
2. risk_management 排第二组
3. portfolio_management 排第三组

**每个 agent 一行**：

```
✓ Warren Buffett          [AAPL] Done
⋯ Cathie Wood             [MSFT] Fetching financials
✗ Peter Lynch             [TSLA] Error
```

**3 种状态**：✓ done（绿）/ ⋯ working（黄）/ ✗ error（红）

### 全局实例

```python
progress = AgentProgress()
```

**单例**——整个进程用一个 progress 实例。

## 其他 5 个 utils 工具（简评）

### `src/utils/display.py` —— 终端输出

- `print_trading_output(result)` —— 表格化打印 trading decisions
- 类似 v2 的 demo 输出但 ANSI 颜色

### `src/utils/api_key.py` —— API key 抽取

```python
def get_api_key_from_state(state, key_name):
    request = state.get("metadata", {}).get("request")
    if request and hasattr(request, 'api_keys'):
        return request.api_keys.get(key_name)
    return os.getenv(key_name)
```

**两层 fallback**：state.request.api_keys → os.getenv

### `src/utils/visualize.py` —— StateGraph → PNG

```python
def save_graph_as_png(workflow, output_path):
    """Render the LangGraph StateGraph to a PNG file."""
    # 用 mermaid 或 graphviz
    ...
```

**CLI 触发**：`poetry run python src/main.py --graph` 会调这个。

### `src/utils/ollama.py` —— Ollama 本地模型

辅助函数检测 ollama 是否在跑、列出本地模型。

### `src/utils/docker.py` —— Docker 容器

辅助函数构建/运行 Docker 镜像（Web app 的容器化部署）。

## 设计取舍

### 1. v1 `call_llm` 复杂但通用

✅ **13 家 provider + 3 次 retry + 4 段 JSON 解析**：

- 灵活
- 但调用栈深（langchain → wrapper → retry → parse）

❌ **简化（v2 风格）**：

- 只支持 1 家
- 不 retry

**判断**：v1 早期设计，**优先兼容多种 provider**——后来发现 v2 风格更可维护。

### 2. `progress.update_handlers` 模式

✅ **观察者模式**：

- 外部可以订阅 agent 状态
- Web app / API 可以接入

❌ **直接调外部**：耦合

**判断**：合理——**为多端接入留口**。

### 3. retry 3 次 vs 1 次

✅ **3 次**：网络抖动救一下

❌ **1 次**：失败就 abstain

**v2 选择 1 次**：cache 命中率应该高，retry 不必要。

**判断**：v1 旧世界（无 cache），retry 有用。

## 对 fork → CN 版本的启示

### A. v2 的 LLM 简化是进步

CN 化应该走 v2 风格——**不需要 13 家 provider**。

### B. 但 v1 的 progress / display / visualize 可复用

CN demo 可以加：

```python
from src.utils.progress import progress
progress.update_status("cn_pead", "600519.SH", "BEAT", analysis="...")
```

### C. v1 的 API key state 抽取机制合理

```python
api_keys = state.get("metadata", {}).get("request").api_keys
```

**CN 化可以保留**——web app 调用时把 key 放在 state 里。

## 深读问题（自检）

- [ ] `call_llm` 调 `llm.invoke(prompt)` —— `prompt` 是 LangChain 的 `ChatPromptTemplate` 还是 string？
- [ ] `default_factory` vs `create_default_response` —— 两者都生成 default Pydantic model，什么时候用哪个？
- [ ] `progress.update_handlers` 是同步还是异步调用？web app 接入需要异步吗？
- [ ] `ollama.py` 做什么具体事？是检测 daemon 还是 pull 模型？

---

*解析时间：2026-07-12 · 第十四轮迭代*
*下次解析目标：`src/agents/{valuation,sentiment,fundamentals,technicals,news_sentiment,growth_agent}.py`（6 个无人名 quant agent）+ src/backtesting/*（8 个文件）*