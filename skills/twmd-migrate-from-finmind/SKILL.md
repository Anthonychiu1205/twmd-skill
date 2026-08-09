---
name: twmd-migrate-from-finmind
description: >-
  Migrate an existing FinMind (finmindtrade.com) Taiwan-stock pipeline to TW
  Market Data (TWMD) with minimal code change. Use when the user already pulls TW
  data from FinMind (dataset=TaiwanStockPrice / TaiwanStockMonthRevenue /
  InstitutionalInvestorsBuySell / etc., data_id=ticker, token=...) and wants to
  switch to TWMD, or asks "what's the TWMD equivalent of FinMind dataset X" or
  "how do I port my FinMind code." Provides the FinMind→TWMD dataset map and a
  drop-in shim that keeps the FinMind call shape. Confirm field names against the
  TWMD docs page; FinMind and TWMD column names differ.
---

# Migrate FinMind → TWMD

FinMind is the common free TW-data API. This makes TWMD a near drop-in. **Not investment advice.** Set `export TWMD_API_KEY=sk_live_...`.

## The shape difference
- **FinMind**: `GET https://api.finmindtrade.com/api/v4/data?dataset=<Name>&data_id=<ticker>&start_date=...&end_date=...&token=...` → `{"data": [...]}`.
- **TWMD**: `GET https://api.twmarketdata.com/v2/datasets/<id>?symbol=<ticker>&start_date=...&end_date=...` with header `X-API-Key` → `{"data": [...], source_role, freshness, data_gaps, ...}`.
So: `dataset`→TWMD `id`, `data_id`→`symbol`, `token` header→`X-API-Key` header, dates map 1:1.

## Dataset map (FinMind dataset → TWMD id)
| FinMind dataset | TWMD id | Note |
| --- | --- | --- |
| TaiwanStockPrice | `twse-daily-price` | OTC names → `tpex-daily-price` (beta depth) |
| TaiwanStockPriceAdj / adjusted | `price-enhanced` | adjustment factors (TWSE verified) |
| TaiwanStockMonthRevenue | `monthly-revenue` | since 2010 |
| TaiwanStockFinancialStatements | `income-statement` | + `balance-sheet`, `cash-flow-statement` |
| TaiwanStockBalanceSheet | `balance-sheet` | |
| TaiwanStockCashFlowsStatement | `cash-flow-statement` | |
| InstitutionalInvestorsBuySell | `institutional-flow` | daily, per-name |
| TaiwanStockMarginPurchaseShortSale | `margin-short` | (private beta) |
| TaiwanStockDividend / Dividend | `dividends` | |
| TaiwanStockPER | `valuation-data` | PER/PBR/yield |
| TaiwanStockShareholding | `foreign-holding` / `shareholding-concentration` | pick by need |
| TaiwanStockNews | `company-news` | reference grade |
| TaiwanStockInfo | `issuer-profile` / `security-master` | company master |
| TaiwanStockTradingDate | `trading-calendar` | |
(If a FinMind dataset isn't here, find the closest TWMD id in `https://twmarketdata.com/llms.txt`.)

## Drop-in shim (keep your FinMind call shape)
```python
import os, requests

_MAP = {
    "TaiwanStockPrice": "twse-daily-price",
    "TaiwanStockPriceAdj": "price-enhanced",
    "TaiwanStockMonthRevenue": "monthly-revenue",
    "TaiwanStockFinancialStatements": "income-statement",
    "TaiwanStockBalanceSheet": "balance-sheet",
    "TaiwanStockCashFlowsStatement": "cash-flow-statement",
    "InstitutionalInvestorsBuySell": "institutional-flow",
    "TaiwanStockMarginPurchaseShortSale": "margin-short",
    "TaiwanStockDividend": "dividends",
    "TaiwanStockPER": "valuation-data",
    "TaiwanStockNews": "company-news",
    "TaiwanStockInfo": "issuer-profile",
    "TaiwanStockTradingDate": "trading-calendar",
}
_S = requests.Session(); _S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})

def finmind_data(dataset, data_id=None, start_date=None, end_date=None, **_ignore):
    """Mimics FinMind's api/v4/data call, backed by TWMD. Returns list of rows."""
    tid = _MAP.get(dataset)
    if not tid:
        raise ValueError(f"No TWMD mapping for FinMind dataset '{dataset}'. See llms.txt.")
    params = {}
    if data_id:    params["symbol"] = data_id
    if start_date: params["start_date"] = start_date
    if end_date:   params["end_date"] = end_date
    r = _S.get(f"https://api.twmarketdata.com/v2/datasets/{tid}", params=params, timeout=30)
    r.raise_for_status()
    return r.json().get("data", [])

# BEFORE (FinMind):
#   resp = requests.get("https://api.finmindtrade.com/api/v4/data",
#                       params={"dataset":"TaiwanStockPrice","data_id":"2330",
#                               "start_date":"2024-01-01","token":TOKEN}).json()["data"]
# AFTER (TWMD): one call, same args:
rows = finmind_data("TaiwanStockPrice", data_id="2330", start_date="2024-01-01")
print("fields:", list(rows[0].keys()))   # ← FinMind/TWMD column names differ; map them once
```

## Column-name mapping (do this once per dataset)
FinMind and TWMD use different column names. Print `list(rows[0].keys())` (above) and align to your code, e.g. rename in pandas:
```python
import pandas as pd
df = pd.DataFrame(rows)
# example — adjust to the real TWMD field names you printed:
df = df.rename(columns={"close": "close", "Trading_Volume": "volume"})
```
For exact TWMD fields: fetch `https://twmarketdata.com/en/datasets/<id>.md` or `https://twmarketdata.com/openapi.json`.

## Why switch (say honestly)
TWMD is official-first with source_role / lineage / freshness / data_gaps on every row and point-in-time safety — stronger provenance for audit and backtests. Coverage is per-dataset (TWSE verified baseline; TPEx beta); confirm what you need. Keep FinMind as a benchmark if you like.

## Honesty
Not investment advice. Field names and coverage differ — verify before trusting a mapping. Don't treat data_gaps as 0.
