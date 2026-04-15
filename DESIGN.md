# TradingAgents — Detailed Design Document

> Generated from source analysis. Last updated: 2026-04-12.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Component Breakdown](#3-component-breakdown)
   - 3.1 [LLM Client Layer](#31-llm-client-layer)
   - 3.2 [Data Layer](#32-data-layer)
   - 3.3 [Agent Layer](#33-agent-layer)
   - 3.4 [Graph Orchestration Layer](#34-graph-orchestration-layer)
   - 3.5 [Memory System](#35-memory-system)
   - 3.6 [CLI Interface](#36-cli-interface)
4. [Data Flows](#4-data-flows)
5. [State Management](#5-state-management)
6. [Configuration System](#6-configuration-system)
7. [Design Patterns](#7-design-patterns)
8. [Key Interfaces and APIs](#8-key-interfaces-and-apis)
9. [Deployment and Infrastructure](#9-deployment-and-infrastructure)
10. [Extension Points](#10-extension-points)

---

## 1. System Overview

TradingAgents is a **multi-agent LLM framework** that simulates the analytical workflow of a professional trading firm. Rather than relying on a single model to make a trading decision, it deploys a team of specialized agents—analysts, researchers, risk managers, and a portfolio manager—who gather data, debate, and synthesize findings into a final investment decision.

### Design Goals

| Goal | How It Is Achieved |
|---|---|
| Comprehensive analysis | Four specialist analyst agents covering technical, sentiment, news, and fundamental dimensions |
| Balanced decision-making | Adversarial debate model (bull vs bear, aggressive vs conservative) before any decision is made |
| LLM-agnostic | Abstracted provider layer supporting OpenAI, Anthropic, Google, xAI, OpenRouter, Ollama, MLX |
| Data-vendor flexibility | Vendor abstraction with per-tool overrides and fallback chains |
| Learning over time | BM25-based in-memory storage of past decisions, retrieved at inference time |
| Interactive UX | Rich CLI with live progress display |

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          User Interface                              │
│              CLI (Typer + Rich)  /  Programmatic API                 │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                    TradingAgentsGraph.propagate()
                                │
┌───────────────────────────────▼─────────────────────────────────────┐
│                       Graph Orchestration                            │
│              LangGraph StateGraph  (setup.py, trading_graph.py)      │
│   ConditionalLogic  ·  Propagator  ·  Reflector  ·  SignalProcessor  │
└──────┬─────────────────────┬───────────────────┬────────────────────┘
       │                     │                   │
┌──────▼──────┐  ┌───────────▼───────┐  ┌────────▼───────────────────┐
│  LLM Client │  │   Agent Layer     │  │      Data Layer            │
│   Layer     │  │                   │  │                            │
│ factory.py  │  │ Analysts          │  │ interface.py (router)      │
│             │  │ Researchers       │  │ y_finance.py               │
│ OpenAI      │  │ Risk Debators     │  │ alpha_vantage.py           │
│ Anthropic   │  │ Trader            │  │ yfinance_news.py           │
│ Google      │  │ Research Manager  │  │ (data_cache/ on disk)      │
│ xAI         │  │ Portfolio Manager │  │                            │
│ OpenRouter  │  └───────────────────┘  └────────────────────────────┘
│ Ollama/MLX  │
└─────────────┘
```

---

## 3. Component Breakdown

### 3.1 LLM Client Layer

**Location**: `tradingagents/llm_clients/`

The LLM client layer provides a **provider-agnostic interface** that the rest of the system uses without caring which LLM backend is in use.

#### Files

| File | Purpose |
|---|---|
| `base_client.py` | Abstract base class `BaseLLMClient` |
| `factory.py` | `create_llm_client()` factory function |
| `model_catalog.py` | `MODEL_OPTIONS` dictionary and helper functions |
| `validators.py` | Model name validation per provider |
| `openai_client.py` | OpenAI (and OpenAI-compatible) implementation |
| `anthropic_client.py` | Anthropic implementation |
| `google_client.py` | Google Gemini implementation |

#### Class Hierarchy

```
BaseLLMClient (ABC)
├── OpenAIClient         # also used for xAI, OpenRouter, Ollama, MLX
├── AnthropicClient
└── GoogleClient
```

#### Factory Dispatch

```python
# tradingagents/llm_clients/factory.py
create_llm_client(provider, model, base_url, **kwargs) -> BaseLLMClient
```

| `provider` value | Routed to | Notes |
|---|---|---|
| `"openai"` | `OpenAIClient` | Default base URL |
| `"xai"` | `OpenAIClient` | `base_url="https://api.x.ai/v1"` |
| `"openrouter"` | `OpenAIClient` | OpenRouter proxy base URL |
| `"ollama"` | `OpenAIClient` | `base_url="http://localhost:11434/v1"` |
| `"mlx"` | `OpenAIClient` | `base_url="http://localhost:8000/v1"` |
| `"anthropic"` | `AnthropicClient` | |
| `"google"` | `GoogleClient` | |

#### Model Catalog

`MODEL_OPTIONS` in `model_catalog.py` organises models by provider and speed tier:

```
MODEL_OPTIONS[provider]["quick"]  →  [(display_name, model_id), ...]
MODEL_OPTIONS[provider]["deep"]   →  [(display_name, model_id), ...]
```

- **quick** models are used for fast, cheaper tasks (analyst agents, trader)
- **deep** models are used for complex reasoning (research manager, portfolio manager)

#### Thinking / Reasoning Support

Each provider has its own mechanism for extended reasoning:

| Provider | Parameter | Values |
|---|---|---|
| OpenAI | `openai_reasoning_effort` | `"low"`, `"medium"`, `"high"` |
| Anthropic | `anthropic_effort` | `"low"`, `"medium"`, `"high"` |
| Google | `google_thinking_level` | `"minimal"`, `"medium"`, `"high"` |

The `normalize_content()` utility function in `base_client.py` handles multi-typed content blocks returned by reasoning-capable models (e.g. `reasoning` + `text` blocks from OpenAI Responses API and Gemini 3).

---

### 3.2 Data Layer

**Location**: `tradingagents/dataflows/`

The data layer abstracts all market data retrieval behind a **vendor routing interface**, supporting hot-swappable vendors at category level or individual tool level.

#### Files

| File | Purpose |
|---|---|
| `interface.py` | Central router: `route_to_vendor()`, `get_vendor()` |
| `y_finance.py` | yfinance vendor implementation |
| `alpha_vantage.py` | Alpha Vantage API implementation |
| `yfinance_news.py` | News retrieval via yfinance |
| `config.py` | `get_config()` / `set_config()` (lazy global state) |
| `data_cache/` | Disk cache for API responses |

#### Vendor Categories

```python
TOOLS_CATEGORIES = {
    "core_stock_apis":      ["get_stock_data"],
    "technical_indicators": ["get_indicators"],
    "fundamental_data":     ["get_fundamentals", "get_balance_sheet",
                              "get_cashflow", "get_income_statement"],
    "news_data":            ["get_news", "get_global_news"],
}
```

#### Routing Logic

```
route_to_vendor(method, *args, **kwargs)
    └─ get_vendor(method)
          1. Check tool_vendors[method] for tool-level override
          2. Fall back to data_vendors[category] for category-level config
          └─ Returns vendor name → dispatches to VENDOR_METHODS[method][vendor]
```

Rate limit errors from Alpha Vantage (`AlphaVantageRateLimitError`) are caught and trigger fallback to the next vendor in the fallback chain.

#### yfinance Vendor Capabilities

| Function | Data Returned |
|---|---|
| `get_YFin_data_online()` | OHLCV price history |
| `get_stock_stats_indicators_window()` | SMA 50/200, EMA 10, MACD, RSI, Bollinger Bands, ATR, VWMA, MFI |
| `get_fundamentals()` | Company overview and key metrics |
| `get_balance_sheet()` | Balance sheet (annual or quarterly) |
| `get_cashflow()` | Cash flow statement |
| `get_income_statement()` | Income statement |
| `get_insider_transactions()` | Insider buy/sell activity |
| `get_news_yfinance()` | Company-specific news items |
| `get_global_news_yfinance()` | Macroeconomic and global market news |

---

### 3.3 Agent Layer

**Location**: `tradingagents/agents/`

There are **11 agents** in total, organised into five logical teams. Each agent is a LangGraph node that receives the full workflow state, calls LLM and/or tools, and writes its outputs back into specific state fields.

#### Agent Creation Pattern

Each agent is created by a factory function that:
1. Receives `llm`, tool list, and optional `memory` instance
2. Binds tools to the LLM via `.bind_tools()`
3. Returns an inner function `agent_node(state) -> dict` for use as a LangGraph node

```python
def create_market_analyst(llm, toolkit):
    @tool ...
    def agent_node(state: AgentState) -> dict:
        ...
        return {"messages": [result], "market_report": ...}
    return agent_node
```

#### 3.3.1 Analyst Team

These agents run **in parallel** during the first phase, each producing a specialist report.

##### Market Analyst
- **File**: `agents/analysts/market_analyst.py`
- **LLM tier**: quick
- **Tools**: `get_stock_data`, `get_indicators`
- **Output state field**: `market_report`
- **Indicators available**: SMA (50-day, 200-day), EMA (10-day), MACD, RSI, Bollinger Bands, ATR, VWMA, MFI
- **Prompt strategy**: Agent selects up to 8 complementary indicators, presents findings in markdown tables with trend interpretation

##### Social Media Analyst
- **File**: `agents/analysts/social_media_analyst.py`
- **LLM tier**: quick
- **Tools**: `get_news` (social + sentiment data)
- **Output state field**: `sentiment_report`
- **Prompt strategy**: Covers social sentiment, posts, recent company news, extracts actionable insights

##### News Analyst
- **File**: `agents/analysts/news_analyst.py`
- **LLM tier**: quick
- **Tools**: `get_news`, `get_global_news`
- **Output state field**: `news_report`
- **Prompt strategy**: Monitors global macro news and company-specific press with trading relevance

##### Fundamentals Analyst
- **File**: `agents/analysts/fundamentals_analyst.py`
- **LLM tier**: quick
- **Tools**: `get_fundamentals`, `get_balance_sheet`, `get_cashflow`, `get_income_statement`
- **Output state field**: `fundamentals_report`
- **Prompt strategy**: Full financial analysis including historical performance, growth metrics, and balance sheet health

---

#### 3.3.2 Research Team (Investment Debate)

The research phase runs an **adversarial debate** between Bull and Bear researchers, mediated by the Research Manager.

##### Bull Researcher
- **File**: `agents/researchers/bull_researcher.py`
- **LLM tier**: deep
- **Memory**: `bull_memory` (`FinancialSituationMemory`)
- **Input**: All four analyst reports + 2 most similar past situations from memory
- **Output state field**: `investment_debate_state.bull_history`
- **Prompt strategy**: Constructs growth-positive case — opportunities, competitive advantages, momentum, catalysts

##### Bear Researcher
- **File**: `agents/researchers/bear_researcher.py`
- **LLM tier**: deep
- **Memory**: `bear_memory` (`FinancialSituationMemory`)
- **Input**: All four analyst reports + 2 most similar past situations from memory
- **Output state field**: `investment_debate_state.bear_history`
- **Prompt strategy**: Identifies risks, weaknesses, challenges — directly counters bullish arguments with specific data

##### Research Manager
- **File**: `agents/managers/research_manager.py`
- **LLM tier**: deep (uses `deep_thinking_llm`)
- **Memory**: `invest_judge_memory`
- **Input**: Full bull/bear debate history
- **Output state field**: `investment_plan`
- **Output format**: Investment plan with `**BUY**`, `**SELL**`, or `**HOLD**` recommendation
- **Prompt strategy**: Critically evaluates both sides, chooses stance based on strongest evidence, provides clear rationale

---

#### 3.3.3 Trader Agent

##### Trader
- **File**: `agents/trader/trader.py`
- **LLM tier**: quick (uses `quick_thinking_llm`)
- **Memory**: `trader_memory`
- **Input**: `investment_plan` from Research Manager
- **Output state field**: `trader_investment_plan`
- **Output format**: Must end with `"FINAL TRANSACTION PROPOSAL: **BUY/HOLD/SELL**"`
- **Prompt strategy**: Translates strategic investment plan into a concrete trading action with position sizing rationale

---

#### 3.3.4 Risk Management Team

The risk phase runs a **three-way debate** between risk personalities, mediated by the Portfolio Manager.

##### Aggressive Debator
- **File**: `agents/risk_mgmt/aggressive_debator.py`
- **LLM tier**: deep
- **Output state field**: `risk_debate_state.aggressive_history`
- **Prompt strategy**: Champions high-reward, accepts high-risk — questions conservative caution, highlights missed upside

##### Conservative Debator
- **File**: `agents/risk_mgmt/conservative_debator.py`
- **LLM tier**: deep
- **Output state field**: `risk_debate_state.conservative_history`
- **Prompt strategy**: Asset protection, volatility minimisation — counters aggressive optimism with downside scenarios

##### Neutral Debator
- **File**: `agents/risk_mgmt/neutral_debator.py`
- **LLM tier**: deep
- **Output state field**: `risk_debate_state.neutral_history`
- **Prompt strategy**: Balanced risk/reward evaluation — challenges both extremes, advocates sustainable strategy

---

#### 3.3.5 Portfolio Manager

##### Portfolio Manager
- **File**: `agents/managers/portfolio_manager.py`
- **LLM tier**: deep (uses `deep_thinking_llm`)
- **Memory**: `portfolio_manager_memory`
- **Input**: Full risk debate history + trader proposal
- **Output state field**: `final_trade_decision`
- **Output format**: Five-tier rating: `BUY / OVERWEIGHT / HOLD / UNDERWEIGHT / SELL`
- **Report sections**: Rating, Executive Summary, Investment Thesis (with specific evidence), Risk Assessment, Position Sizing guidance

---

### 3.4 Graph Orchestration Layer

**Location**: `tradingagents/graph/`

The graph orchestration layer uses **LangGraph's `StateGraph`** to wire agents into a deterministic, conditional workflow.

#### Files

| File | Class/Function | Purpose |
|---|---|---|
| `trading_graph.py` | `TradingAgentsGraph` | Public API; exposes `propagate()` |
| `setup.py` | `GraphSetup` | Builds the compiled LangGraph |
| `conditional_logic.py` | `ConditionalLogic` | Edge routing functions |
| `propagation.py` | `Propagator` | State initialisation and graph invocation |
| `reflection.py` | `Reflector` | Post-trade memory updates |
| `signal_processing.py` | `SignalProcessor` | Extracts final signal from PM output |

#### Workflow Graph

```
START
  │
  ├─── market_analyst_node
  │        │  [tool_calls?]
  │        ├── market_tools_node ──► (loop back to market_analyst_node)
  │        └── (no tools) ──► next analyst
  │
  ├─── social_analyst_node
  │        └── social_tools_node (similar loop)
  │
  ├─── news_analyst_node
  │        └── news_tools_node (similar loop)
  │
  ├─── fundamentals_analyst_node
  │        └── fundamentals_tools_node (similar loop)
  │
  ├─── bull_researcher_node ◄─────────────────────────┐
  │                                                    │
  ├─── bear_researcher_node                           │ debate loop
  │        │                                          │ (max_debate_rounds × 2)
  │        └── [round < max?] ─────────────────────────┘
  │
  ├─── research_manager_node
  │
  ├─── trader_node
  │
  ├─── aggressive_debator_node ◄─────────────────────────┐
  │                                                       │
  ├─── conservative_debator_node                         │ risk loop
  │                                                       │ (max_risk_discuss_rounds × 3)
  ├─── neutral_debator_node                              │
  │        └── [round < max?] ─────────────────────────────┘
  │
  └─── portfolio_manager_node
           │
          END
```

#### Configurable Analyst Selection

Not all analysts need to run. The config key `"selected_analysts"` (list) controls which analyst nodes are added to the graph. `GraphSetup.setup_graph()` only adds nodes for selected analysts and chains them in order.

#### Conditional Edge Functions (`ConditionalLogic`)

| Function | Purpose |
|---|---|
| `should_continue_market(state)` | Route to `market_tools_node` if LLM emitted tool calls, else move forward |
| `should_continue_social(state)` | Same pattern for social analyst |
| `should_continue_news(state)` | Same pattern for news analyst |
| `should_continue_fundamentals(state)` | Same pattern for fundamentals analyst |
| `should_continue_debate(state)` | Alternate bull→bear until `2 × max_debate_rounds` total turns |
| `should_continue_risk_analysis(state)` | Cycle aggressive→conservative→neutral until `3 × max_risk_discuss_rounds` total turns |

#### Signal Processing

`SignalProcessor.process_signal(state)` uses regex to extract the final five-tier rating from the Portfolio Manager's output text:

```
BUY | OVERWEIGHT | HOLD | UNDERWEIGHT | SELL
```

#### Reflection / Memory Update

`Reflector` is called **after a trade result is known** (i.e., when returns data is available) to update agent memories. For each memory-carrying agent it:
1. Formats a prompt describing the original situation and the actual trade outcome
2. Invokes the LLM to synthesise a lesson
3. Appends the lesson to the agent's `FinancialSituationMemory`

---

### 3.5 Memory System

**Location**: `tradingagents/agents/utils/memory.py`

The memory system provides **long-term learning** across analysis sessions without requiring any external database or API.

#### `FinancialSituationMemory`

```python
class FinancialSituationMemory:
    def __init__(self, name: str, embedding_model=None)
    def add_memory(self, situation: str, analysis: str) -> None
    def get_memories(self, current_situation: str, n_matches: int = 2) -> str
    def reflect_and_remember(self, returns_losses: float) -> None
```

- **Storage**: In-process Python list of `(situation, analysis)` pairs
- **Retrieval**: BM25 (Best Match 25) lexical similarity — no vector embeddings, no external API
- **Retrieval call site**: At the start of each Bull/Bear/Trader/Manager agent invocation
- **Update call site**: In `Reflector` methods after trade outcomes are known
- **One instance per agent**: `bull_memory`, `bear_memory`, `trader_memory`, `invest_judge_memory`, `portfolio_manager_memory`

> **Note**: Memory is currently in-process only — it does not persist across program restarts unless the `TradingAgentsGraph` instance is reused.

---

### 3.6 CLI Interface

**Location**: `cli/`

The CLI provides an **interactive terminal UI** for running analyses without writing code.

#### Files

| File | Purpose |
|---|---|
| `main.py` | Typer app, `MessageBuffer` class, Rich layout |
| `utils.py` | Shared utility helpers |
| `stats_handler.py` | `StatsCallbackHandler` — token counts, timings |
| `announcements.py` | Display framework version announcements |

#### User Interaction Flow

```
1. Enter ticker symbol(s)
2. Enter analysis date
3. Select LLM provider
4. Select quick_think and deep_think models
5. Select which analysts to run
6. Configure debate rounds, risk rounds, etc.
7. Watch real-time Rich UI:
   - Header with ticker / date
   - Progress panel: current agent, phase completion
   - Messages panel: live agent outputs
   - Analysis display: completed report sections
8. View final Portfolio Manager report
```

#### `MessageBuffer`

Tracks UI state during a live run:

- `agent_status` — which agents have started/completed
- `report_sections` — accumulated text per section (market, sentiment, news, fundamentals, investment plan, etc.)
- `current_tool_calls` — tools being invoked right now
- `init_for_analysis(selected_analysts)` — resets display for new run

#### `StatsCallbackHandler`

LangChain callback that intercepts LLM events:

- Counts prompt/completion tokens per call
- Measures LLM inference duration and tool call duration
- Displays formatted stats summary at end of run

---

## 4. Data Flows

### 4.1 End-to-End Analysis Flow

```
User Input:
  ticker="AAPL", date="2025-01-15", config={...}
          │
          ▼
TradingAgentsGraph.propagate(ticker, date)
  └── Propagator.create_initial_state(ticker, date)
        → AgentState { company_of_interest: "AAPL",
                       trade_date: "2025-01-15",
                       messages: [],  reports: {} }
          │
          ▼
  LangGraph.invoke(state, config={recursion_limit: 100})
          │
    ┌─────▼──────────────────────────────────────────────┐
    │  PHASE 1: ANALYST                                   │
    │                                                     │
    │  market_analyst(state)                              │
    │    → LLM generates tool_calls                       │
    │    → market_tools executes: get_stock_data(AAPL)    │
    │                              get_indicators(AAPL)   │
    │    → LLM synthesises → state.market_report          │
    │                                                     │
    │  social_analyst → state.sentiment_report            │
    │  news_analyst   → state.news_report                 │
    │  fundamentals_analyst → state.fundamentals_report   │
    └─────────────────────────────────────────────────────┘
          │
    ┌─────▼──────────────────────────────────────────────┐
    │  PHASE 2: RESEARCH DEBATE                           │
    │                                                     │
    │  For i in range(max_debate_rounds):                 │
    │    bull_researcher(state) → state.bull_history      │
    │    bear_researcher(state) → state.bear_history      │
    │  research_manager(state) → state.investment_plan   │
    └─────────────────────────────────────────────────────┘
          │
    ┌─────▼──────────────────────────────────────────────┐
    │  PHASE 3: TRADING DECISION                          │
    │                                                     │
    │  trader(state) → state.trader_investment_plan       │
    └─────────────────────────────────────────────────────┘
          │
    ┌─────▼──────────────────────────────────────────────┐
    │  PHASE 4: RISK DEBATE                               │
    │                                                     │
    │  For i in range(max_risk_discuss_rounds):           │
    │    aggressive_debator → state.aggressive_history    │
    │    conservative_debator → state.conservative_history│
    │    neutral_debator → state.neutral_history          │
    │  portfolio_manager → state.final_trade_decision     │
    └─────────────────────────────────────────────────────┘
          │
          ▼
  SignalProcessor.process_signal(state)
    → "BUY" / "OVERWEIGHT" / "HOLD" / "UNDERWEIGHT" / "SELL"
          │
          ▼
  Result written to results/ as JSON
  Returned to caller as (state, signal)
```

### 4.2 Tool Call Data Flow (within an agent)

```
Agent LLM receives: [system_prompt, analyst_context, tool_definitions]
        │
        │  LLM emits ToolCall { name: "get_indicators",
        │                       args: {symbol, indicator, date, lookback} }
        ▼
ConditionalLogic.should_continue_X() → route to tools_node
        │
        ▼
ToolNode.invoke(tool_call)
  └── route_to_vendor("get_indicators", symbol, indicator, date, lookback)
        └── get_vendor("get_indicators")
              → config["tool_vendors"]["get_indicators"]   (tool-level)
              → config["data_vendors"]["technical_indicators"]  (category-level)
              → "yfinance"
        └── y_finance.get_stock_stats_indicators_window(symbol, indicator, date, lookback)
              → pd.DataFrame (OHLCV + indicator column)
              → format as string
        → ToolMessage added to state.messages
        │
        ▼
Agent LLM receives tool result, continues generating response
```

---

## 5. State Management

### 5.1 State Types

**Location**: `tradingagents/agents/utils/agent_states.py`

#### `AgentState` (primary workflow state)

```python
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]    # Full conversation + tool history
    company_of_interest: str                   # Ticker symbol
    trade_date: str                            # Analysis date YYYY-MM-DD

    # Analyst reports (populated during Phase 1)
    market_report: str
    sentiment_report: str
    news_report: str
    fundamentals_report: str

    # Research phase (populated during Phase 2)
    investment_debate_state: InvestDebateState
    investment_plan: str                       # Research Manager output

    # Trading phase (populated during Phase 3)
    trader_investment_plan: str                # Trader output

    # Risk phase (populated during Phase 4)
    risk_debate_state: RiskDebateState
    final_trade_decision: str                  # Portfolio Manager output
```

#### `InvestDebateState`

```python
class InvestDebateState(TypedDict):
    bull_history: str     # Concatenated bull researcher turns
    bear_history: str     # Concatenated bear researcher turns
    current_response: str
    judge_decision: str
    count: int            # Turn counter for debate loop
```

#### `RiskDebateState`

```python
class RiskDebateState(TypedDict):
    aggressive_history: str
    conservative_history: str
    neutral_history: str
    current_response: str
    judge_decision: str
    count: int            # Turn counter for risk loop
```

### 5.2 State Scoping

- Each agent reads whatever fields it needs from the full state
- Each agent writes only its designated output field(s)
- `messages` uses LangGraph's `add_messages` reducer to append (not overwrite)
- Debate turn counters (`count`) are incremented by the conditional logic functions

---

## 6. Configuration System

**Location**: `tradingagents/default_config.py`, `tradingagents/dataflows/config.py`

### Default Configuration

```python
DEFAULT_CONFIG = {
    # Paths
    "project_dir": <package_root>,
    "results_dir": "./results",
    "data_cache_dir": <package_root>/dataflows/data_cache,

    # LLM
    "llm_provider": "openai",
    "deep_think_llm": "gpt-5.4",
    "quick_think_llm": "gpt-5.4-mini",
    "backend_url": "https://api.openai.com/v1",

    # Provider-specific reasoning config (all default None = disabled)
    "google_thinking_level": None,      # "minimal" | "medium" | "high"
    "openai_reasoning_effort": None,    # "low" | "medium" | "high"
    "anthropic_effort": None,           # "low" | "medium" | "high"

    # Output
    "output_language": "English",

    # Workflow depth
    "max_debate_rounds": 1,
    "max_risk_discuss_rounds": 1,
    "max_recur_limit": 100,

    # Selected analysts (all four by default)
    "selected_analysts": ["market", "social", "news", "fundamentals"],

    # Data vendors (category-level)
    "data_vendors": {
        "core_stock_apis":      "yfinance",    # or "alpha_vantage"
        "technical_indicators": "yfinance",
        "fundamental_data":     "yfinance",
        "news_data":            "yfinance",
    },

    # Tool-level vendor overrides (takes precedence over data_vendors)
    "tool_vendors": {},    # e.g. {"get_stock_data": "alpha_vantage"}
}
```

### Configuration Access

```python
from tradingagents.dataflows.config import set_config, get_config

# Override at runtime
set_config({
    "llm_provider": "anthropic",
    "deep_think_llm": "claude-opus-4-6",
    "quick_think_llm": "claude-haiku-4-5-20251001",
})
```

### Environment Variables

| Variable | Provider |
|---|---|
| `OPENAI_API_KEY` | OpenAI, xAI (via different base URL), OpenRouter |
| `ANTHROPIC_API_KEY` | Anthropic |
| `GOOGLE_API_KEY` | Google Gemini |
| `XAI_API_KEY` | xAI Grok |
| `OPENROUTER_API_KEY` | OpenRouter |
| `ALPHA_VANTAGE_API_KEY` | Alpha Vantage data |

---

## 7. Design Patterns

### 7.1 Factory Pattern

Used throughout to defer construction and decouple consumers from concrete types.

```
create_llm_client(provider, model, ...)      → BaseLLMClient subclass
create_market_analyst(llm, toolkit)          → agent_node function
create_bull_researcher(llm, memory, toolkit) → agent_node function
```

### 7.2 Strategy Pattern

Bull/Bear/Aggressive/Conservative/Neutral agents all share the same structural interface (they are LangGraph nodes with the same signature) but implement **radically different analytical strategies** through their system prompts.

### 7.3 State Machine (via LangGraph)

The full workflow is a directed graph where each node is an agent and edges are routing functions. This is a formal state machine: transitions are explicit and deterministic given the state.

### 7.4 Chain of Responsibility (Vendor Fallback)

```
route_to_vendor()
  try primary vendor (e.g. alpha_vantage)
  on AlphaVantageRateLimitError:
    try next vendor (e.g. yfinance)
```

### 7.5 Adapter Pattern

`interface.py` adapts the disparate yfinance and Alpha Vantage APIs behind a uniform set of tool function signatures that agents call without knowing the underlying vendor.

### 7.6 Template Method Pattern

All analyst `agent_node` functions follow the same template:
1. Extract `company_of_interest` and `trade_date` from state
2. Format analyst-specific system prompt
3. Invoke LLM with tool bindings
4. Return `{"messages": [...], "<report_field>": ...}`

### 7.7 Memento Pattern

`FinancialSituationMemory` stores snapshots of past situations (mementos) and returns the most relevant ones given the current situation, enabling agents to learn from history without external state.

### 7.8 Observer Pattern

`StatsCallbackHandler` implements the LangChain `BaseCallbackHandler` interface, observing all LLM and tool events without agents needing to instrument themselves.

### 7.9 Dependency Injection

LLM clients and memory instances are constructed outside agents and **injected at construction time**:

```python
# In TradingAgentsGraph.__init__
deep_llm = create_llm_client(provider, deep_model, ...)
bull_memory = FinancialSituationMemory("bull")
bull_researcher = create_bull_researcher(deep_llm, bull_memory, toolkit)
```

---

## 8. Key Interfaces and APIs

### 8.1 Programmatic API

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph

# Instantiate with config
ta = TradingAgentsGraph(
    selected_analysts=["market", "fundamentals"],
    config={
        "llm_provider": "anthropic",
        "deep_think_llm": "claude-opus-4-6",
        "quick_think_llm": "claude-haiku-4-5-20251001",
        "max_debate_rounds": 2,
        "max_risk_discuss_rounds": 1,
    }
)

# Run analysis
state, decision = ta.propagate("AAPL", "2025-01-15")

# decision is one of: BUY, OVERWEIGHT, HOLD, UNDERWEIGHT, SELL
print(decision)

# Full state contains all reports
print(state["market_report"])
print(state["final_trade_decision"])
```

### 8.2 CLI API

```bash
tradingagents run              # interactive wizard
tradingagents run --ticker AAPL --date 2025-01-15
```

### 8.3 Tool Interface (Agent Tools)

All tools follow the LangChain `@tool` decorator pattern with typed signatures:

```python
@tool
def get_stock_data(symbol: str, start_date: str, end_date: str) -> str:
    """Get OHLCV price history for a symbol."""
    ...

@tool
def get_indicators(symbol: str, indicator: str,
                   curr_date: str, look_back_days: int) -> str:
    """Get technical indicator values for a symbol."""
    ...
```

### 8.4 Memory Interface

```python
memory = FinancialSituationMemory(name="bull")

# Retrieve similar past situations (called before LLM invoke)
context = memory.get_memories(current_situation_text, n_matches=2)

# Store a new situation+analysis pair (called after agent responds)
memory.add_memory(situation_text, analysis_text)

# Update memory with trade outcome (called by Reflector post-trade)
memory.reflect_and_remember(returns_percentage)
```

---

## 9. Deployment and Infrastructure

### Local Development

```bash
pip install -e .
cp .env.example .env    # add API keys
tradingagents run
```

### Docker

A `Dockerfile` and `docker-compose.yml` are provided for containerised deployment.

```bash
docker compose up
```

The container installs dependencies and exposes the CLI. Volume mounts can persist the `results/` and `data_cache/` directories across runs.

### Directory Layout at Runtime

```
TradingAgents/
├── tradingagents/
│   └── dataflows/
│       └── data_cache/        # Cached API responses (disk)
├── results/                   # JSON analysis results
├── reports/                   # Report outputs
└── .env                       # API keys
```

### Performance Notes

- **Cost control**: `quick_think_llm` handles all analyst/trader nodes; only `deep_think_llm` is used for debate/judgment nodes
- **Parallelism**: Analyst nodes run serially in the current graph (sequential edges), but are structurally independent and could be parallelised via `LangGraph` parallel branches
- **Caching**: Data vendor responses are cached on disk; a cache hit avoids all network calls for the same ticker/date combination
- **Recursion limit**: `max_recur_limit=100` caps total LangGraph steps to prevent runaway loops

---

## 10. Extension Points

### Adding a New LLM Provider

1. Subclass `BaseLLMClient` in `tradingagents/llm_clients/`
2. Implement `get_llm()` and `validate_model()`
3. Register in `factory.py` dispatch dict
4. Add model entries to `model_catalog.py`

### Adding a New Data Vendor

1. Implement the required functions (e.g. `get_stock_data`, `get_indicators`) in a new file under `tradingagents/dataflows/`
2. Register them in `VENDOR_METHODS` in `interface.py`
3. Add the vendor name to the relevant category in `TOOLS_CATEGORIES`
4. Set via `data_vendors` config key or `tool_vendors` for per-tool overrides

### Adding a New Agent

1. Create a factory function following the `create_<name>(llm, toolkit, ...)` pattern
2. Define a system prompt and bind the relevant tools
3. Return an `agent_node(state: AgentState) -> dict` function
4. Register the node in `GraphSetup.setup_graph()` with appropriate edges

### Adding a New Analyst Type

As above, plus:
1. Add the new analyst's report field to `AgentState`
2. Add a new tool set / tool node if the analyst uses different tools
3. Add the analyst name to `"selected_analysts"` in the config
4. Update `ConditionalLogic` with a `should_continue_<name>()` routing function

### Persisting Memory Across Runs

Currently `FinancialSituationMemory` is in-process only. To persist:
1. Serialise the memory list to JSON/SQLite in `add_memory()`
2. Load from storage in `__init__()`
3. This gives agents cumulative learning across all analysis sessions

---

*End of Design Document*
