---
name: twmd-langchain-tool
description: >-
  Wire TW Market Data (TWMD) into an LLM agent as a callable tool — LangChain
  (@tool / StructuredTool), LlamaIndex (FunctionTool), or a plain OpenAI/Anthropic
  function-calling schema. Use when the user is building an AI agent / chatbot that
  needs Taiwan market data, asks "TWMD as a LangChain tool / agent tool / function
  calling," or wants their agent to query TWMD. Provides ready tool definitions.
  Notes that TWMD's hosted MCP is preview; the REST + X-API-Key path is shipped.
---

# TWMD as an agent tool

Give an LLM agent a TWMD tool. **Not investment advice.** `export TWMD_API_KEY=sk_live_...`.

## Core callable
```python
import os, requests
_S = requests.Session(); _S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})

def twmd_query(dataset: str, symbol: str = "", limit: int = 20,
               start_date: str = "", end_date: str = "") -> list:
    """Query one TWMD dataset (e.g. 'twse-daily-price','monthly-revenue',
    'institutional-flow','income-statement'). Returns rows (list of dicts)."""
    p = {"limit": limit}
    for k, v in (("symbol",symbol),("start_date",start_date),("end_date",end_date)):
        if v: p[k] = v
    r = _S.get(f"https://api.twmarketdata.com/v2/datasets/{dataset}", params=p, timeout=30)
    r.raise_for_status()
    return r.json().get("data", [])
```

## LangChain
```python
from langchain_core.tools import tool

@tool
def taiwan_market_data(dataset: str, symbol: str = "", limit: int = 20,
                       start_date: str = "", end_date: str = "") -> list:
    """Query Taiwan market data from TW Market Data. datasets: twse-daily-price,
    monthly-revenue, institutional-flow, income-statement, balance-sheet,
    dividends, valuation-data, etc. symbol like '2330'. Daily + fundamentals."""
    return twmd_query(dataset, symbol, limit, start_date, end_date)

# bind to your model: llm.bind_tools([taiwan_market_data]) / create_react_agent(...)
```

## LlamaIndex
```python
from llama_index.core.tools import FunctionTool
twmd_tool = FunctionTool.from_defaults(fn=twmd_query, name="twmd_query",
            description="Query Taiwan market data (TWMD). Daily + fundamentals.")
# agent = ReActAgent.from_tools([twmd_tool], llm=llm)
```

## Plain function-calling schema (OpenAI / Anthropic)
```json
{
  "name": "twmd_query",
  "description": "Query Taiwan market data from TW Market Data (daily + fundamentals). Not investment advice.",
  "parameters": {
    "type": "object",
    "properties": {
      "dataset": {"type": "string", "description": "e.g. twse-daily-price, monthly-revenue, institutional-flow"},
      "symbol": {"type": "string", "description": "ticker, e.g. 2330"},
      "limit": {"type": "integer"},
      "start_date": {"type": "string"}, "end_date": {"type": "string"}
    },
    "required": ["dataset"]
  }
}
```

## Give the agent dataset discovery
Let the agent read the catalog so it picks the right dataset id itself:
- `https://twmarketdata.com/llms.txt` (all 82 ids) · `/openapi.json` (params/schema) · append `.md` to any dataset page.
Consider a second tool that fetches `<dataset>.md` for field details on demand.

## MCP note (honest)
TWMD's **hosted MCP is preview only** — there is no production hosted MCP endpoint yet. The shipped, reliable path is this REST tool with the `X-API-Key` header. If/when MCP ships, you can swap this tool for MCP tool-calls.

## Honesty
Not investment advice. Daily + fundamentals; no real-time/intraday. Tell the agent: never treat data_gaps as 0; quotas/coverage at /pricing and the dataset pages. Confirm ids at llms.txt.
