# AI-Trader 作业报告（中文版）


### 1) 项目背景

这是一次课程作业实验：我使用 **DeepSeek 模型** 在 AI-Trader 框架中执行美股交易回放，并分析策略指标表现。  
本次重点窗口：`2025-10-01 10:00:00` ~ `2025-10-02 15:00:00`，初始资金 `$10,000`。

---

### 2) 设置说明（环境、数据、密钥、计算）

#### 2.1 环境
- Python：`.python-version` 为 `3.14`
- 依赖管理：`uv`（`pyproject.toml` + `uv.lock`），也可使用 `requirements.txt`
- 关键依赖：`langchain==1.0.2`、`langchain-openai==1.0.1`、`langchain-mcp-adapters>=0.1.0`

#### 2.2 数据
- 交易日志与持仓：`data/agent_data/my-deepseek/`
- 对照（报告基线）持仓：`data/agent_data/deepseek-chat-v3.1/`
- 行情文件：`data/daily_prices_*.json`（本次有 12 个标的被更新为 compact 日线）

#### 2.3 密钥
在 `.env` 中配置：
- `OPENAI_API_BASE`
- `OPENAI_API_KEY`（可用于兼容网关）
- `ALPHAADVANTAGE_API_KEY`
- `JINA_API_KEY`

#### 2.4 计算与运行
1. 安装依赖：`uv sync`
2. 启动 MCP 服务：`python agent_tools/start_mcp_services.py`
3. 启动实验：`python main.py configs/my_deepseek_config.json`
4. 结果读取：`data/agent_data/my-deepseek/position/position.jsonl`

---

### 3) 复现实验目标（们）+ 指标定义

#### 3.1 复现实验目标
- 使用 DeepSeek 成功跑通小时级交易回放流程
- 与仓库报告基线（`deepseek-chat-v3.1`）在同时间窗进行对比
- 分析模型改动与数据改动对指标可复现性的影响

#### 3.2 指标定义
- **总收益率**：`(期末资产 - 期初资产) / 期初资产`
- **期末资产**：现金 + 各持仓按对应时点收盘价估值
- **最大回撤**：`(历史峰值 - 当前值) / 历史峰值` 的最大值
- **波动率（step）**：相邻估值点收益率的标准差
- **缺失价格计数**：估值时找不到对应价格的持仓条目数

---

### 4) 结果：您的数据与报告数据对比（表格）

> 注：为公平比较，主表采用“报告口径估值”（对本次被改写的 12 个行情文件使用改动前快照，消除回测窗口缺口）。

#### 4.1 报告口径（可比）

| 组别 | 期末资产 | 总收益率 | 最大回撤 | 波动率(step) | 交易行为（买/卖/不交易） |
|---|---:|---:|---:|---:|---:|
| 我的实验（my-deepseek） | 10094.64 | **+0.95%** | 0.06% | 0.094% | 11 / 0 / 0 |
| 报告基线（deepseek-chat-v3.1） | 9949.48 | -0.51% | 0.51% | 0.119% | 2 / 0 / 9 |
| QQQ 基准 | 10087.10 | +0.87% | 0.04% | - | - |

**结论（可比口径）**：  
- 相对报告基线：`+1.45` 个百分点  
- 相对 QQQ：`+0.08` 个百分点

#### 4.2 当前工作区口径（受数据改动影响）

| 组别 | 期末资产 | 总收益率 | 最大回撤 | 缺失价格计数 |
|---|---:|---:|---:|---:|
| 我的实验（my-deepseek） | 6028.41 | -39.72% | 52.87% | 61 |
| 报告基线（deepseek-chat-v3.1） | 4741.98 | -52.58% | 52.58% | 8 |

---

### 5) 我的修改内容 + 修改后的结果

| 文件 | 修改点 | 影响 |
|---|---|---|
| `agent/base_agent/base_agent.py` | 新增 DeepSeek 兼容处理：将消息 `content` 从 list 归一化为 string；保持 tool args 解析 | 减少 DeepSeek 接口格式不兼容导致的失败风险 |
| `configs/default_config.json` | 启用 `deepseek-chat-v3.1`，关闭 `gpt-5` | 默认实验入口切换到 DeepSeek |
| `configs/my_deepseek_config.json` | 新增自定义配置（小时级，签名 `my-deepseek`） | 可独立复现实验并写入专属目录 |
| `.python-version` / `pyproject.toml` / `uv.lock` | 引入 Python 3.14 + uv 依赖管理 | 环境更明确，但需要统一 Python 版本 |
| `data/daily_prices_*.json`（12 个） | 行情从 60min/full 切换为 Daily/Compact（最早到 `2025-10-06`） | 回测窗口 `2025-10-01~10-02` 出现价格覆盖缺口，直接影响估值与回测可比性 |
| `data/agent_data/my-deepseek/*` | 新增本次实验日志与持仓记录 | 提供了完整过程数据用于复盘 |

---

### 6) 调试日记：主要障碍与解决方法

1. **障碍：DeepSeek 请求格式兼容性**  
   - 现象：消息内容可能为 list block，接口期望 string。  
   - 解决：在 `DeepSeekChatOpenAI._get_request_payload` 中统一转为字符串。

2. **障碍：工具调用参数格式不一致**  
   - 现象：tool call 的 `arguments` 有时为 JSON 字符串。  
   - 解决：在 `_generate/_agenerate` 中做 JSON 反序列化兜底。

3. **障碍：估值数据覆盖不足**  
   - 现象：被改为 compact 的行情文件最早仅到 `2025-10-06`，与回测窗口不重合。  
   - 解决：报告比较时使用改动前行情快照口径；并建议正式复现时回退 full-size/小时级数据或调整回测日期。

4. **障碍：日志中出现重复 user 请求与末时点 assistant 缺失**  
   - 现象：`2025-10-02 10:00/14:00/15:00` 存在重复请求，最后时点无 assistant 回复。  
   - 解决：保留 `max_retries`，并建议增加“超时重试标记 + 断点恢复”机制。

---

### 7) 结论：哪些可重复，哪些不可重复，原因何在

#### 可重复
- DeepSeek 兼容改动（payload/content/tool args）可稳定复用。
- 在**同一数据快照 + 同一时间窗 + 同一配置**下，`my-deepseek` 相对基线更优（+1.45pp）可重复。

#### 不可直接重复
- 使用当前 compact 数据直接估值本实验窗口时，结果会显著偏差（大量缺失价格），不可与报告直接对齐。
- 末时点日志缺失与重试行为具有运行时偶然性，不保证每次一致。

#### 主要原因
- 数据时间覆盖与回测时间窗不匹配（核心原因）。
- 运行时重试/中断导致日志形态存在非确定性。

---

