# AI-Trader Assignment Report (English)


### 1) Background

This is a course assignment. I used a **DeepSeek model** to run trading replay in AI-Trader and evaluated performance metrics.  
Main window: `2025-10-01 10:00:00` to `2025-10-02 15:00:00`, initial capital `$10,000`.

---

### 2) Setup (environment, data, keys, computation)

#### 2.1 Environment
- Python: `.python-version` is `3.14`
- Dependency manager: `uv` (`pyproject.toml` + `uv.lock`), with `requirements.txt` as fallback
- Key packages: `langchain==1.0.2`, `langchain-openai==1.0.1`, `langchain-mcp-adapters>=0.1.0`

#### 2.2 Data
- My run logs/positions: `data/agent_data/my-deepseek/`
- Report baseline positions: `data/agent_data/deepseek-chat-v3.1/`
- Price files: `data/daily_prices_*.json` (12 symbols were updated to compact daily files)

#### 2.3 Keys
Set in `.env`:
- `OPENAI_API_BASE`
- `OPENAI_API_KEY`
- `ALPHAADVANTAGE_API_KEY`
- `JINA_API_KEY`

#### 2.4 Run & compute
1. Install deps: `uv sync`
2. Start MCP services: `python agent_tools/start_mcp_services.py`
3. Run experiment: `python main.py configs/my_deepseek_config.json`
4. Read results: `data/agent_data/my-deepseek/position/position.jsonl`

---

### 3) Reproduction targets + metric definitions

#### 3.1 Targets
- Successfully run hourly replay with DeepSeek
- Compare with repository report baseline (`deepseek-chat-v3.1`) in the same time window
- Analyze reproducibility impact from code/data modifications

#### 3.2 Metrics
- **Total Return**: `(final equity - initial equity) / initial equity`
- **Final Equity**: cash + marked-to-market value of holdings
- **Max Drawdown**: max of `(peak - current) / peak`
- **Step Volatility**: std of adjacent-step returns
- **Missing Price Count**: number of positions without available prices at valuation time

---

### 4) Results: your data vs report data (table)

> Note: The main comparison uses a “report-compatible valuation” (pre-change snapshot for the 12 updated price files) to remove coverage bias.

#### 4.1 Report-compatible comparison

| Group | Final Equity | Total Return | Max Drawdown | Step Volatility | Actions (buy/sell/no-trade) |
|---|---:|---:|---:|---:|---:|
| My run (`my-deepseek`) | 10094.64 | **+0.95%** | 0.06% | 0.094% | 11 / 0 / 0 |
| Report baseline (`deepseek-chat-v3.1`) | 9949.48 | -0.51% | 0.51% | 0.119% | 2 / 0 / 9 |
| QQQ benchmark | 10087.10 | +0.87% | 0.04% | - | - |

**Summary (comparable setting)**:  
- vs baseline: `+1.45` percentage points  
- vs QQQ: `+0.08` percentage points

#### 4.2 Current-workspace valuation (affected by data changes)

| Group | Final Equity | Total Return | Max Drawdown | Missing Price Count |
|---|---:|---:|---:|---:|
| My run (`my-deepseek`) | 6028.41 | -39.72% | 52.87% | 61 |
| Report baseline (`deepseek-chat-v3.1`) | 4741.98 | -52.58% | 52.58% | 8 |

---

### 5) My modifications + post-change outcomes

| File | Change | Impact |
|---|---|---|
| `agent/base_agent/base_agent.py` | Added DeepSeek compatibility: normalize message `content` list → string; keep tool-args parsing | Reduces API-format failure risk |
| `configs/default_config.json` | Enabled `deepseek-chat-v3.1`, disabled `gpt-5` | Default run now uses DeepSeek |
| `configs/my_deepseek_config.json` | Added custom hourly config with signature `my-deepseek` | Isolated and reproducible experiment run |
| `.python-version` / `pyproject.toml` / `uv.lock` | Introduced Python 3.14 + uv workflow | More explicit env; requires version alignment |
| `data/daily_prices_*.json` (12 files) | Switched from 60min/full to daily/compact (earliest date `2025-10-06`) | Creates valuation gaps for `2025-10-01~10-02`, affecting comparability |
| `data/agent_data/my-deepseek/*` | Added run logs and position records | Enables full run replay and analysis |

---

### 6) Debug diary: major blockers and fixes

1. **DeepSeek request payload mismatch**  
   - Issue: API expects string content, but content blocks can be lists.  
   - Fix: normalized content in `DeepSeekChatOpenAI._get_request_payload`.

2. **Tool-call argument format inconsistency**  
   - Issue: tool-call `arguments` may arrive as JSON strings.  
   - Fix: added JSON parsing fallback in `_generate/_agenerate`.

3. **Insufficient price coverage for valuation**  
   - Issue: compact files start at `2025-10-06`, outside replay window.  
   - Fix: used pre-change snapshot for report comparison; recommend full-size/hourly prices or aligned dates for strict reproduction.

4. **Repeated user prompts and missing final assistant output in logs**  
   - Issue: duplicated user entries at `2025-10-02 10:00/14:00/15:00`, and no assistant output at last slot.  
   - Fix: keep retry settings and add timeout/retry markers plus checkpoint resume.

---

### 7) Conclusion: reproducible vs non-reproducible

#### Reproducible
- DeepSeek compatibility fixes (payload/content/tool args) are reproducible.
- With same data snapshot + date window + config, my run outperforms baseline by +1.45pp.

#### Not directly reproducible
- Using current compact price files on this replay window yields biased valuation (many missing prices), so it cannot be directly compared to report numbers.
- Final-slot logging behavior is runtime-sensitive (retry/interruption), so logs may differ run-to-run.

#### Why
- Core cause: mismatch between data coverage and replay window.
- Secondary cause: non-deterministic retry/interruption behavior during runtime.
