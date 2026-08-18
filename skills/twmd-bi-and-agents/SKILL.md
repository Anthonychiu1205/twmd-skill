---
name: twmd-bi-and-agents
description: >-
  Bring TW Market Data (TWMD) into BI tools and AI agents — Excel (Power Query),
  Google Sheets (Apps Script), Power BI (Power Query M), Tableau (via warehouse),
  and LLM-agent tooling (LangChain @tool, LlamaIndex FunctionTool, plain
  OpenAI/Anthropic function-calling), plus OpenBB. Use when the user wants TWMD in
  a spreadsheet/dashboard, or to give an AI agent/chatbot a Taiwan-market-data
  tool, or asks "TWMD in Excel/Sheets/Power BI/Tableau/LangChain/OpenBB." Never
  embed the API key in a shared sheet/report. MCP is LIVE (mcp.twmarketdata.com); REST covers more datasets.
---

# TWMD in BI tools & AI agents

Spreadsheets, dashboards, and agent tools. **Not investment advice.** **Never put the key in a shared cell/report** — use script properties / parameters / gateway credentials.

## What you can fully do for the user
- Get TWMD into their exact surface: an Excel Power Query, a Google Sheets function, a parameterized Power BI query, a Tableau-via-warehouse plan, or an agent tool — working, not a sketch.
- Ship a runnable LangChain/LlamaIndex tool or a function-calling schema, plus dataset-discovery so the agent covers the whole storefront catalog (84) and stays current.
- Keep keys safe (script properties / parameters / gateway; never in a shared cell/report/URL).
- MCP is LIVE but serves a subset (85 of the REST catalogue's 125), so reach for REST when coverage matters; hand warehouse setup to **twmd-integration**.

## Excel — Power Query (sends X-API-Key header)
Data → Get Data → Blank Query → Advanced Editor:
```m
let
    key = ApiKey,                       // define ApiKey as a Parameter, not inline
    url = "https://api.twmarketdata.com/v2/datasets/monthly-revenue?symbol=2330&limit=24",
    resp = Json.Document(Web.Contents(url, [Headers=[#"X-API-Key"=key]])),
    data = resp[data],
    tbl  = Table.FromList(data, Splitter.SplitByNothing()),
    cols = Table.ExpandRecordColumn(tbl, "Column1", Record.FieldNames(data{0}))
in  cols
```

## Google Sheets — Apps Script (keyed) / IMPORTDATA (no-key demo)
```javascript
function TWMD(dataset, symbol, limit) {
  const key = PropertiesService.getScriptProperties().getProperty('TWMD_API_KEY');
  const url = `https://api.twmarketdata.com/v2/datasets/${dataset}?symbol=${symbol}&limit=${limit||30}`;
  const rows = JSON.parse(UrlFetchApp.fetch(url,{headers:{'X-API-Key':key}}).getContentText()).data;
  if(!rows.length) return [['no data']];
  const cols = Object.keys(rows[0]);
  return [cols].concat(rows.map(r=>cols.map(c=>r[c])));
}   // cell:  =TWMD("twse-daily-price","2330",30)   ; set key in Project Settings → Script Properties
```
No-key demo (5 symbols) works with plain `=IMPORTDATA("https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=30")` (IMPORTDATA can't send headers → keyed data needs Apps Script).

## Power BI
Same Power Query M as Excel; store the key as a **Parameter** (or a Service gateway credential). For a universe, build `fnTWMD(symbol)` and invoke over a symbol table. Scheduled refresh in Power BI Service (daily after close).

## Tableau (via warehouse — reliable path)
Tableau's web-data-connector route is fragile. Best: load TWMD into a database (DuckDB/Postgres/BigQuery/Snowflake — see the integration skill), then connect Tableau natively to that DB. Keeps the key server-side and gives fast extracts.

## AI agent tool — core callable
```python
import os, requests
_S=requests.Session(); _S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})
def twmd_query(dataset:str, symbol:str="", limit:int=20, start_date:str="", end_date:str="") -> list:
    """Query one TWMD dataset (twse-daily-price, monthly-revenue, institutional-flow,
    income-statement, dividends, valuation-data, ...). symbol like '2330'. Daily + fundamentals."""
    p={"limit":limit}
    for k,v in (("symbol",symbol),("start_date",start_date),("end_date",end_date)):
        if v: p[k]=v
    r=_S.get(f"https://api.twmarketdata.com/v2/datasets/{dataset}",params=p,timeout=30)
    r.raise_for_status(); return r.json().get("data",[])
```
```python
# LangChain
from langchain_core.tools import tool
@tool
def taiwan_market_data(dataset:str, symbol:str="", limit:int=20, start_date:str="", end_date:str="")->list:
    """Query Taiwan market data (TWMD). Daily + fundamentals."""
    return twmd_query(dataset,symbol,limit,start_date,end_date)
# LlamaIndex
from llama_index.core.tools import FunctionTool
twmd_tool = FunctionTool.from_defaults(fn=twmd_query, name="twmd_query",
            description="Query Taiwan market data (TWMD). Daily + fundamentals.")
```
```json
// plain OpenAI/Anthropic function-calling schema
{"name":"twmd_query","description":"Query Taiwan market data (TWMD). Not investment advice.",
 "parameters":{"type":"object","properties":{
   "dataset":{"type":"string","description":"e.g. twse-daily-price, monthly-revenue, institutional-flow"},
   "symbol":{"type":"string"},"limit":{"type":"integer"},"start_date":{"type":"string"},"end_date":{"type":"string"}},
   "required":["dataset"]}}
```
Give the agent discovery: let it read `https://twmarketdata.com/llms.txt` (all ids) and `<id>.md` for fields. **MCP note**: TWMD's hosted MCP is **LIVE** at `mcp.twmarketdata.com` (server `tw-market-data`), exposing `list_datasets` / `describe_dataset` / `query_dataset` / `find_related`. It covers a SUBSET of the catalogue (85 vs the REST API's 125), so this REST tool with `X-API-Key` is still the wider path.

## OpenBB
- **Quick**: call `twmd_query`/a small fetch inside your OpenBB workflow (works today).
- **Provider extension**: build an `openbb` provider package with a `Fetcher` (`transform_query`/`extract_data`/`transform_data`) reading TWMD, register via the `openbb_provider_extension` entry point → `obb.equity.price.historical(symbol="2330", provider="twmd")`. Adjust field models to TWMD's real fields.

## Honesty & confirm
Not investment advice. Daily + fundamentals. Keep the key in script properties / parameters / gateway creds — never in a shared cell/report/URL. Agent rule: never treat data_gaps as 0; quotas/coverage at `/pricing` and dataset pages. Confirm ids at `/llms.txt`, fields at `<id>.md` / `/openapi.json`.
