# v2/signals/buffett.py 深度解析

> 文件：`v2/signals/buffett.py` · **54 行** · 1 个 `LLMAgent` 子类
> 地位：**v2 "persona is just a name + system prompt" 哲学的最佳示范**——一个真实可用的 LLM investor agent 全靠这一段 prompt

## 文件作用一句话

**定义 `BuffettAgent`：名字 + 一段 Buffett 风格的 system prompt**。其他一切（snapshot 构建、cache、parse、Signal 生成）全部继承自 `LLMAgent`。

## 整个文件就 54 行

```python
class BuffettAgent(LLMAgent):
    @property
    def name(self):
        return "buffett"
    
    def get_system_prompt(self):
        return """..."""
```

**对比 v1**：`src/agents/warren_buffett.py` 是 **826 行**——手算 ROE/毛利/管理层质量/DCF，然后塞给 LLM。

**对比 v2**：54 行——纯 prompt。

**节省的 772 行是什么**：

| v1 的工作 | v2 在哪里做 |
|----------|------------|
| 手算 ROE 平均 | `FundamentalsSnapshot.roe_avg`（Python） |
| 手算毛利趋势 | `FundamentalsSnapshot.gross_margin_trend`（Python） |
| 手算 BVPS CAGR | `FundamentalsSnapshot.bvps_cagr`（Python） |
| 手算 debt/equity | `FundamentalsSnapshot.debt_to_equity_latest`（Python） |
| 拼 prompt 模板 | `FundamentalsSnapshot.render()`（Python） |
| 调 LLM + retry + parse | `LLMAgent.predict()`（Python） |
| 写 cache | `LLMAgent` + `PromptCache`（Python） |
| 错误处理 | `LLMAgent._abstain`（Python） |

**所有这些都被 v2 的抽象层接管了**——Buffett persona 只负责"说什么"。

## System prompt 的精妙结构（line 22-53）

```python
"""You are Warren Buffett, evaluating a single company as a
long-term business owner, not a trader.

Work through your checklist:
1. Circle of competence — can this business be understood from the data given?
2. Competitive moat — durable high returns on equity, stable or improving
   margins, pricing power.
3. Management quality — capital allocation visible in the numbers: book value
   compounding, sensible leverage, consistent free cash flow.
4. Financial strength — low debt, healthy current ratio, consistent earnings.
5. Valuation — is the price (market cap, P/E) sensible relative to the
   quality and growth of the business? A wonderful company at a fair price
   beats a fair company at a wonderful price.
6. Long-term prospects — would you be comfortable holding this for ten years?

Signal rules:
- bullish: a strong, durable business at a reasonable or better price.
- bearish: a weak or deteriorating business, or a price that demands
  perfection.
- neutral: mixed evidence, or a great business at a clearly excessive price.

Confidence scale (0-100): 90-100 exceptional conviction with strong evidence;
70-89 solid conviction; 40-69 mixed; 10-39 weak or speculative.

Hard rules:
- Reason ONLY from the data provided. Do not use any knowledge of what
  happened after the as-of date. Do not invent numbers.
- If the data is insufficient to judge, say so and go neutral.

Respond with JSON only, in exactly this schema:
{"signal": "bullish" | "bearish" | "neutral", "confidence": <0-100>,
 "reasoning": "<your thesis in Buffett's voice, 2-4 sentences>"}"""
```

### 5 段结构

#### 1. **身份声明**（line 22-23）

> "You are Warren Buffett, evaluating a single company as a long-term business owner, not a trader."

**3 个作用**：
- 设定 persona（Warren Buffett）
- 设定视角（business owner，不是 trader）
- **隐式排除日内/短线推理**

#### 2. **Checklist**（line 25-35）—— 6 步

| 步骤 | 评估什么 | 用的 snapshot 数据 |
|------|---------|-------------------|
| 1. Circle of competence | 业务可理解性 | sector/industry（用作"是不是我能理解的领域"） |
| 2. Competitive moat | ROE、毛利稳定性 | roe_avg, gross_margin_trend |
| 3. Management quality | BVPS 增长、杠杆、FCF | bvps_cagr, debt_to_equity_latest |
| 4. Financial strength | 低负债、流动性、盈利一致 | debt_to_equity, current_ratio, periods |
| 5. Valuation | P/E vs 质量 | price_to_earnings_ratio, market_cap |
| 6. Long-term prospects | 10 年视角 | bvps_cagr, revenue_growth |

**精妙之处**：每个 checklist item **正好对应 snapshot 里的一个字段**——LLM 用得上数据。

#### 3. **Signal rules**（line 37-41）

```
- bullish: a strong, durable business at a reasonable or better price.
- bearish: a weak or deteriorating business, or a price that demands perfection.
- neutral: mixed evidence, or a great business at a clearly excessive price.
```

**3 种信号的定义明确**——LLM 不会乱选。

#### 4. **Confidence scale**（line 43-44）

```
Confidence scale (0-100):
90-100: exceptional conviction with strong evidence
70-89:  solid conviction
40-69:  mixed
10-39:  weak or speculative
```

**关键**：**没有 0-9 区间**——避免 LLM 给出无意义低分。

#### 5. **Hard rules**（line 46-49）

```
- Reason ONLY from the data provided.
- Do not use any knowledge of what happened after the as-of date.
- Do not invent numbers.
- If the data is insufficient to judge, say so and go neutral.
```

**4 条铁律**——**对应 v2 的核心 PIT 原则**：

1. 只用提供的数据 → 防止 LLM 用预训练知识"补"未公开信息
2. 不用 as-of 之后的事 → PIT
3. 不编数字 → 防止 hallucination
4. 数据不够就 neutral → graceful degrade

#### 6. **JSON schema 强制**（line 51-53）

```json
{"signal": "bullish" | "bearish" | "neutral", "confidence": <0-100>,
 "reasoning": "<your thesis in Buffett's voice, 2-4 sentences>"}
```

**明确字段**：
- `signal`：3 选 1
- `confidence`：0-100 整数
- `reasoning`：2-4 句

**reasoning 限制 2-4 句**——避免 LLM 写小说（节省 token、审计更易读）。

## 没有 override 的部分（继承自 LLMAgent）

| 方法 | 来源 | 行为 |
|------|------|------|
| `predict(ticker, date, data_client)` | `LLMAgent` | 主流程 |
| `build_snapshot(...)` | `LLMAgent` → `build_snapshot()` | 用默认 `FundamentalsSnapshot` |
| `build_user_prompt(...)` | `LLMAgent` → `snapshot.render()` | 默认渲染 |
| `_parse(...)` | `LLMAgent` | 验证 JSON |
| `_to_signal(...)` | `LLMAgent` | 计算 value |
| `_abstain(...)` | `LLMAgent` | 返回 0.0 |

**BuffettAgent 一个都没 override**——纯粹是 system prompt + name。

## 对 fork → CN 版本的启示

### A. 加一个中国 persona 非常便宜

```python
class DuanYongpingAgent(LLMAgent):
    @property
    def name(self):
        return "duan_yongping"
    
    def get_system_prompt(self):
        return """你是段永平。'敢为天下后'，'做对的事情，把事情做对'，'Stop doing stupid things'。

评估清单：
1. 商业模式 — 这门生意是不是简单到你能看懂？
2. 文化 — 管理层有没有"做对的事情"的品格？
3. 差异化 — 长期有没有定价权？
4. 现金流 — 是否产生持续的、不需要持续融资的自由现金流？
5. 估值 — 当前价格相对于未来 10 年的现金流折现值是否便宜？

信号规则：
- bullish: 好的生意 + 合理的价格（用 4 年盈利估算回报率 > 6% 银行存款）
- bearish: 生意难懂 / 价格过高 / 需要持续融资
- neutral: 好生意但价格已经反映了 / 证据不足

信心 0-100：90+ 极强；70-89 强；40-69 中；10-39 弱。

硬规则：
- 只用提供的数据，不引用未公开信息，不编数字。
- 数据不足就说'数据不足'并给 neutral。

只返回 JSON：
{"signal": "bullish" | "bearish" | "neutral", "confidence": <0-100>,
 "reasoning": "<段永平风格的投资逻辑，2-4 句话>"}"""
```

**54 行**——和你刚读的 BuffettAgent 一模一样的结构。

### B. CN persona 的法律红线

⚠️ **必须保持 VISION 的 "stylized approximation" 措辞**：

```python
class BuffettAgent(LLMAgent):
    """Reasons over fundamentals in Warren Buffett's voice."""
    
    def get_system_prompt(self):
        return """..."""
```

**没有"endorsement by"字样，没有模仿实际决策**——只是 persona 名字 + 公开投资哲学。

**对 CN 投资者同样适用**：

- 段永平、但斌、林园等公开发言/书很多——基于公开哲学不算侵权
- 但不能用其"未经授权的肖像 / 真实决策记录"
- **prompt 里**建议加 disclaimer："这是一个基于公开投资哲学的风格化 AI 决策，不是该投资者的真实决策"

### C. CN persona 设计建议

| Persona | 哲学关键词 | 适合的 snapshot |
|---------|----------|----------------|
| 段永平 | 商业模式 + 文化 + 长期 | fundamentals + qualitative |
| 林园 | 消费垄断 + 长期持有 | fundamentals（高 ROE） |
| 但斌 | 长期伟大公司 + 行业龙头 | fundamentals + revenue_growth |
| 冯柳 | 困境反转 + 左侧买入 | fundamentals + 趋势反转信号 |

**共同点**：所有 CN 价值投资者都看基本面 + 长期——**用 `FundamentalsSnapshot` 完全够用**，不需要重写 snapshot。

## 为什么 Buffett persona 不继承 BuffettAgent 自定义 snapshot？

因为 Buffett 风格 = 价值投资 + 长期持有 + 看基本面——**正好是默认 `FundamentalsSnapshot` 的用途**。

**什么时候该 override `build_snapshot`**（注释 line 109-111）：

> "Override for personas that reason over different data (macro, news); when a second snapshot TYPE exists, extract the implicit interface into a Protocol — not before."

**举例**：

- **宏观投资者**（Ray Dalio）→ 需要 GDP / CPI / 利率 → 单独 `MacroSnapshot`
- **事件驱动投资者**（Peter Thiel）→ 需要 M&A / 监管事件 → 单独 `EventSnapshot`

**v2 当前只有 Buffett persona**——还没到抽象第二个 Protocol 的时候。

## 深读问题（自检）

- [ ] BuffettAgent 的 prompt 里**没明确要求 LLM 看哪些数字**（如 "compute ROE from history"）——为什么？（提示：`render()` 已经给出 ROE avg）
- [ ] "Reasoning 2-4 句"——如果 LLM 输出了 1 句话或 5 句话，会发生什么？（提示：`_parse` 只验证类型，不验证长度）
- [ ] System prompt 是"public investment philosophy"——如果未来想用 Buffett 实际持仓/访谈构建 prompt，要注意什么法律风险？
- [ ] BuffettAgent 假设 ROE high + price reasonable = bullish——但 Buffett 实际**对 "great business at fair price" 和 "fair business at great price" 哪个更倾向？**（提示：看 prompt line 33-34 "wonderful company at a fair price beats fair company at a wonderful price"）

---

*解析时间：2026-07-12 · 第六轮迭代*
*下次解析目标：`v2/backtesting/{engine,models}.py` + `v2/event_study/` + `v2/demo/backtest.py`（剩余 v2/ 文件）*