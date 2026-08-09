---
name: twmd-migrate-data-vendors
description: >-
  Swap generic market-data vendors — pandas-datareader, Alpha Vantage, Tiingo,
  Quandl/Nasdaq Data Link, EOD Historical — to TW Market Data (TWMD) for Taiwan
  equities. Use when the user pulls TW data from one of these global vendors and
  wants a TWMD-backed drop-in returning a pandas DataFrame, or asks "TWMD instead
  of Alpha Vantage/Tiingo/Quandl/pandas-datareader." Provides a unified get_data
  adapter and per-vendor call mapping. Daily; confirm TWMD fields.
---

# Migrate generic vendors → TWMD (Taiwan)

Global vendors have thin/no Taiwan coverage; TWMD is TW-native official-first. This is a drop-in for the fetch. **Not investment advice.** `export TWMD_API_KEY=sk_live_...`. Needs `pandas`.

## Unified adapter (DataFrame out)
```python
import os, requests, pandas as pd
_S = requests.Session(); _S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})

def get_data(symbol, start=None, end=None, dataset="twse-daily-price"):
    """Vendor-agnostic: returns a DataFrame indexed by date (like most vendors)."""
    p = {"symbol": symbol}
    if start: p["start_date"] = start
    if end:   p["end_date"] = end
    r = _S.get(f"https://api.twmarketdata.com/v2/datasets/{dataset}", params=p, timeout=30)
    r.raise_for_status()
    df = pd.DataFrame(r.json().get("data", []))
    # print(df.columns.tolist())  # confirm real field names once
    if "date" in df: df = df.assign(date=pd.to_datetime(df["date"])).set_index("date").sort_index()
    return df
```

## Per-vendor call mapping
```python
# pandas-datareader:
#   web.DataReader("2330.TW", "yahoo", start, end)      →  get_data("2330", start, end)
# Alpha Vantage:
#   TimeSeries().get_daily("2330.TWO")                   →  get_data("2330", dataset="tpex-daily-price")
# Tiingo:
#   client.get_dataframe("2330", startDate=..., endDate=...)  →  get_data("2330", start, end)
# Quandl / Nasdaq Data Link:
#   quandl.get("XTAI/2330", start_date=..., end_date=...)     →  get_data("2330", start, end)
# EOD Historical:
#   get_eod("2330.TW", from_=..., to=...)                →  get_data("2330", start, end)
df = get_data("2330", start="2024-01-01")
print(df.tail())
```

## Why (honest)
Global vendors' TW coverage is often incomplete, delayed, or via secondary sources. TWMD is official-first (TWSE/TPEx/MOPS/TAIFEX) with source_role/lineage/freshness/data_gaps and keeps delisted histories. But TWMD is **Taiwan-only + daily + fundamentals** — if your pipeline also needs US/global or intraday, keep the other vendor for those and use TWMD for TW.

## Honesty
Not investment advice. Taiwan only; daily. Vendor field names differ — map after `print(columns)`. Confirm fields at `<id>.md` / `/openapi.json`. data_gaps ≠ 0.
