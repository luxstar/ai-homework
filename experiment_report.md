# AI-Trader 复现实验报告：LLM 自主交易代理中推理步数对交易性能的影响

---

## 1. 项目概述与复现目标

### 1.1 项目介绍

AI-Trader（arXiv:2512.10971）是由 HKUDS 团队提出的 LLM 自主交易代理基准测试框架，旨在回答"AI 能否战胜市场？"这一核心问题。该框架让多个前沿大语言模型（GPT-5、Claude-3.7-Sonnet、DeepSeek-V3.1、Qwen3-Max、Gemini-2.5-Flash、MiniMax-M2）作为独立的交易代理，在相同起始资金条件下，对 NASDAQ-100、SSE-50 和 BITWISE-10 三个市场进行完全自主的交易决策。

系统采用 ReAct 风格的工具调用代理循环，基于 LangChain + MCP（Model Context Protocol）架构。每个交易会话中，LLM 接收当前持仓和市场价格信息，通过调用工具（查价格、搜新闻、买入、卖出、数学计算）进行推理和交易决策，直到输出终止信号 `<FINISH_SIGNAL>` 或达到最大推理步数上限。

### 1.2 复现目标

1. 使用 DeepSeek-Chat 模型复现美股（NASDAQ-100）小时级交易代理
2. 修改关键参数 `max_steps`（最大推理步数：30 → 10），量化其对交易性能的影响
3. 对比修改前后的评估指标（CR、SR、Vol、MDD）

---

## 2. 实验环境说明

### 2.1 硬件环境

| 项目 | 配置 |
|------|------|
| CPU | Intel Core i7-13700K (13th Gen) |
| 内存 | 32 GB |
| 操作系统 | macOS 14.5 (Build 23F79) |

### 2.2 软件环境

| 组件 | 版本 |
|------|------|
| Python | 3.11.14 |
| langchain | 1.0.2 |
| langchain-openai | 1.0.1 |
| langchain-mcp-adapters | 0.2.1 |
| fastmcp | 2.12.5 |
| Git Commit | `82174e5` (main branch) |

### 2.3 API 服务

| 服务 | 端点 |
|------|------|
| LLM API | DeepSeek API (`https://api.deepseek.com`)，模型 `deepseek-chat` |
| 新闻搜索 | Alpha Vantage News Sentiment API |
| 价格数据 | 本地 `data/merged.jsonl`（60分钟级 OHLCV 数据） |

### 2.4 MCP 微服务

| 服务 | 端口 | 功能 |
|------|------|------|
| Math | 8000 | 算术计算（add、multiply） |
| Search | 8001 | Alpha Vantage 新闻搜索 |
| Trade | 8002 | 股票买入/卖出执行 |
| Price | 8003 | 本地历史价格查询 |

---

## 3. 数据说明

### 3.1 市场数据

- **市场**：NASDAQ-100 美股
- **数据格式**：JSONL（每行一个股票的完整时间序列）
- **时间粒度**：60 分钟（每交易日 6 个时间点：10:00 ~ 15:00 EST）
- **数据范围**：2025-10-01 至 2025-11-10（29 个交易日，174 个时间戳）
- **股票数量**：100 只 NASDAQ-100 成分股 + CASH
- **字段**：开盘价（buy price）、最高价、最低价、收盘价（sell price）、成交量

### 3.2 防前视偏差机制

- 查询当日价格时，仅返回开盘价，屏蔽当日最高/最低/收盘/成交量
- 新闻搜索仅返回当前模拟日期之前的文章

### 3.3 实验使用的日期范围

| 实验 | 日期范围 | 交易会话数 |
|------|---------|-----------|
| Baseline (steps=30) | 2025-10-01 11:00 ~ 2025-10-02 15:00 | 11 |
| Experiment (steps=10) | 2025-10-01 11:00 ~ 2025-10-02 10:00 | 6 |

> 注：两组实验均因个别会话达到递归限制（LangGraph recursion_limit=100）导致部分后续会话失败而提前结束。实验组因 max_steps 更小，每次会话中可用的工具调用更少，导致代理更频繁地达到限制。

---

## 4. 复现目标与指标定义

### 4.1 指标定义

| 指标 | 全称 | 计算方法 |
|------|------|---------|
| **CR** | Cumulative Return | `(最终价值 - 初始价值) / 初始价值` |
| **SR** | Sortino Ratio | `超额收益 / 下行标准差 × √(年化周期数)` |
| **Vol** | Annualized Volatility | `收益率标准差 × √(年化周期数)` |
| **MDD** | Maximum Drawdown | 最大峰谷回撤幅度 |
| **Sharpe** | Sharpe Ratio | `超额收益 / 总标准差 × √(年化周期数)` |
| **Win Rate** | 胜率 | 正收益周期占总周期的比例 |

年化周期数：252 × 6.5 = 1638（小时级美股数据）。

### 4.2 实验假设

> **H1**：减少 `max_steps`（从 30 到 10）将限制代理在每个交易会话中的推理深度，导致信息获取不充分和决策质量下降，最终体现为更低的累计收益率和更差的风险调整指标。

---

## 5. 原始结果 vs 我的复现结果

| 指标 | 原始 DeepSeek-V3.1 (仓库数据) | 我的复现 (my-deepseek) |
|------|---:|---:|
| 日期范围 | 10/01 10:00 ~ 11/07 15:00 | 10/01 10:00 ~ 10/02 15:00 |
| 初始资金 | $10,000.00 | $10,000.00 |
| 最终价值 | $10,838.57 | $6,312.73 |
| CR | +8.39% | -36.87% |
| SR (Sortino) | 2.83 | -2.87 |
| Vol | 341.76% | 161.78% |
| MDD | -52.58% | -51.64% |
| Sharpe | 1.27 | -2.49 |
| 交易次数 | 208 | 141 |
| 胜率 | 48.39% | 48.23% |

**差异分析**：

1. **日期范围不同**：原始数据覆盖 37 个交易日（218 条记录），我的复现仅覆盖 2 个交易日（142 条记录），时间窗口不同是结果差异的主要原因。
2. **模型版本差异**：原始实验使用 `deepseek-chat-v3.1`（已下线），我的复现使用当前可用的 `deepseek-chat`（V3 版本）。
3. **LLM 非确定性**：大语言模型的输出具有随机性，即使参数完全一致，不同运行的交易决策也会不同。

---

## 6. 我的修改内容

### 6.1 修改描述

**单一参数修改**：将代理配置中的 `max_steps` 从 30 降低为 10。

`max_steps` 控制每个交易会话中 LLM 的最大推理迭代次数。在每次迭代中，代理可以：
- 调用工具获取价格数据
- 搜索市场新闻
- 执行买入/卖出操作
- 进行数学计算

减少该参数意味着代理在每个交易窗口内能执行的操作更少，可能导致：
- 信息收集不充分（无法查看足够多的股票价格）
- 交易执行不完整（无法完成计划的调仓）
- 分析深度不足（无法综合多维信息做决策）

### 6.2 修改位置

**文件**：`configs/my_deepseek_steps10_config.json`

**代码控制点**：`agent/base_agent/base_agent_hour.py:87`
```python
while current_step < self.max_steps:  # max_steps: 30 → 10
    current_step += 1
    response = await self._ainvoke_with_retry(message)
    ...
```

### 6.3 配置差异

```diff
  "agent_config": {
-   "max_steps": 30,
+   "max_steps": 10,
    "max_retries": 3,
    "base_delay": 1.0,
    "initial_cash": 10000.0,
    "verbose": true
  }
```

---

## 7. 修改后的结果

### 7.1 Baseline vs Experiment 对比

| 指标 | Baseline (steps=30) | Experiment (steps=10) | 差值 | 变化方向 |
|------|---:|---:|---:|:---:|
| 最终价值 | $6,312.73 | $5,788.52 | -$524.21 | 更差 |
| **CR** | **-36.87%** | **-42.11%** | **-5.24%** | 更差 |
| **SR (Sortino)** | **-2.87** | **-7.04** | **-4.17** | 更差 |
| **Vol** | **161.78%** | **212.64%** | **+50.86%** | 更高风险 |
| **MDD** | **-51.64%** | **-54.71%** | **-3.07%** | 更大回撤 |
| Sharpe | -2.49 | -5.93 | -3.44 | 更差 |
| 交易次数 | 141 | 60 | -81 | 减少 57% |
| 胜率 | 48.23% | 46.67% | -1.56% | 略降 |
| 平均盈利 | 1.77% | 2.20% | +0.43% | 略升 |
| 平均亏损 | -2.12% | -3.37% | -1.25% | 更大亏损 |

### 7.2 结果分析

**假设验证**：实验结果支持 H1。减少 `max_steps` 对所有核心指标产生了负面影响：

1. **累计收益率下降 5.24%**（-36.87% → -42.11%）：代理在有限步数内无法充分分析市场并做出最优决策。

2. **Sortino 比率恶化 145%**（-2.87 → -7.04）：风险调整后的收益显著下降，说明更少的推理步数不仅降低了收益，还增加了下行风险。

3. **波动率上升 31%**（161.78% → 212.64%）：推理步数不足导致决策质量不稳定，产生更大的收益波动。

4. **交易次数减少 57%**（141 → 60）：这是最直接的影响——每个会话中可用步数减少，代理能执行的买卖操作更少。

5. **平均亏损扩大 59%**（-2.12% → -3.37%）：虽然平均盈利略有上升（+0.43%），但平均亏损显著扩大（-1.25%），说明步数限制下代理更容易做出低质量的交易决策。

---

## 8. 调试日志

### 8.1 问题 1：模型名称不匹配

- **现象**：API 返回 `Error code: 400 - Model Not Exist`
- **原因**：仓库配置中使用 `deepseek/deepseek-chat-v3.1`，但 DeepSeek API 实际仅支持 `deepseek-chat` 和 `deepseek-reasoner`
- **解决方案**：将配置文件中的 `basemodel` 改为 `deepseek-chat`

### 8.2 问题 2：消息格式不兼容

- **现象**：API 返回 `Failed to deserialize the JSON body into the target type: messages[3]: invalid type: sequence, expected a string`
- **原因**：LangChain v1.0.2 的 `create_agent()` 生成的工具响应消息中，`content` 字段为数组格式（多内容块），但 DeepSeek API 仅接受字符串格式
- **解决方案**：在 `agent/base_agent/base_agent.py` 的 `DeepSeekChatOpenAI` 类中重写 `_get_request_payload()` 方法，将数组格式的 content 归一化为字符串：

```python
def _get_request_payload(self, input_, *, stop=None, **kwargs):
    payload = super()._get_request_payload(input_, stop=stop, **kwargs)
    if "messages" in payload:
        for msg in payload["messages"]:
            if isinstance(msg.get("content"), list):
                parts = []
                for block in msg["content"]:
                    if isinstance(block, dict) and "text" in block:
                        parts.append(block["text"])
                    elif isinstance(block, str):
                        parts.append(block)
                msg["content"] = "\n".join(parts) if parts else ""
    return payload
```

### 8.3 问题 3：Python 环境兼容性

- **现象**：`pip install` 报错 `No matching distribution found for langchain==1.0.2`
- **原因**：系统默认 Python 3.9.6 不支持 langchain 1.0.2（需要 Python ≥ 3.10）
- **解决方案**：使用 Python 3.11.14 创建虚拟环境

### 8.4 问题 4：LangGraph 递归限制

- **现象**：部分交易会话报错 `Recursion limit of 100 reached without hitting a stop condition`
- **原因**：代理在单次推理步骤中发起大量工具调用，LangGraph 内部的递归深度超过 100
- **影响**：该错误触发重试机制，部分会话在重试后成功，但极少数会话最终失败。Baseline 完成 11/17 个会话，Experiment 完成 6/17 个会话
- **可能的解决方案**（未实施）：增加 `recursion_limit` 参数，或减少系统提示中的股票数量以降低每步工具调用量

---

## 9. 结论

### 9.1 实验结论

本实验通过修改 AI-Trader 框架中 `max_steps` 参数（30 → 10），验证了推理步数对 LLM 交易代理性能的影响。实验结果表明：

1. **推理步数是影响交易性能的关键参数**：减少 67% 的推理步数导致累计收益下降 5.24%，Sortino 比率恶化 145%，交易次数减少 57%。
2. **步数限制影响决策质量而非决策方向**：胜率仅下降 1.56%，但平均亏损扩大 59%，说明代理在步数受限时仍会尝试交易，但决策质量显著下降。
3. **信息获取深度与交易表现正相关**：更多的推理步数允许代理查看更多股票价格、搜索更多新闻、执行更精细的调仓操作。

### 9.2 可重复性分析

| 因素 | 可控性 | 说明 |
|------|:------:|------|
| 价格数据 | ✅ 完全可控 | 本地 `data/merged.jsonl`，只读使用 |
| 代码版本 | ✅ 完全可控 | Git commit `82174e5` |
| 模型参数 | ✅ 完全可控 | 配置文件指定 `max_steps`, `initial_cash` 等 |
| API 端点 | ⚠️ 部分可控 | DeepSeek API 可用性可能变化 |
| LLM 随机性 | ❌ 不可控 | 模型输出具有非确定性，无可用 seed 参数 |

**主要限制**：
- LLM 的非确定性意味着相同配置的重复运行会产生不同的交易决策和结果
- 实验仅覆盖 2 个交易日的数据，样本量有限，结论的统计显著性需要更多重复实验验证
- `deepseek-chat-v3.1` 已下线，使用的是当前可用的 `deepseek-chat`（V3），与原始实验存在模型版本差异

### 9.3 未来工作

1. 扩大日期范围至完整的 29 个交易日，提高统计显著性
2. 测试更多 `max_steps` 值（如 5、15、20、50），绘制性能-步数曲线
3. 分析日志文件，统计每个会话中代理实际使用的步数分布
4. 对比不同 LLM 模型在相同步数限制下的性能差异

---

**附录**

- 项目仓库：https://github.com/HKUDS/AI-Trader
- Baseline 配置：`configs/my_deepseek_config.json`（max_steps=30, signature=my-deepseek）
- 实验配置：`configs/my_deepseek_steps10_config.json`（max_steps=10, signature=my-deepseek-steps10）
- 指标计算：`python tools/calculate_metrics.py <position.jsonl> --data-dir data --is-hourly`
