---
name: twmd-openbb-provider
description: >-
  Use TW Market Data (TWMD) inside OpenBB — either as a quick Python data source
  in an OpenBB workflow, or as a proper OpenBB Platform provider extension. Use
  when the user works in OpenBB (openbb / OpenBB Platform / Terminal) and wants
  Taiwan market data from TWMD, or asks "TWMD as an OpenBB provider / data
  source." Gives the quick path and the provider-extension fetcher skeleton.
  Confirm TWMD field names; daily + fundamentals.
---

# TWMD in OpenBB

Two paths. **Not investment advice.** `export TWMD_API_KEY=sk_live_...`.

## Path A — quick (call TWMD inside OpenBB / Python)
Simplest: use TWMD directly alongside OpenBB in the same notebook/script.
```python
import os, requests, pandas as pd
def twmd(dataset, **p):
    r = requests.get(f"https://api.twmarketdata.com/v2/datasets/{dataset}", params=p,
                     headers={"X-API-Key": os.environ["TWMD_API_KEY"]}, timeout=30)
    r.raise_for_status(); return pd.DataFrame(r.json().get("data", []))
df = twmd("twse-daily-price", symbol="2330", start_date="2024-01-01")
# then use OpenBB's charting/analytics on df (obb.technical.*, etc.)
```

## Path B — proper OpenBB Platform provider extension
OpenBB v4 discovers providers via a Python extension package exposing a `Provider` with `fetcher_dict`. Skeleton:
```python
# openbb_twmd/models/daily_price.py
from openbb_core.provider.abstract.fetcher import Fetcher
from openbb_core.provider.abstract.query_params import QueryParams
from openbb_core.provider.abstract.data import Data
import os, requests

class TWMDDailyQuery(QueryParams):
    symbol: str
    start_date: str | None = None
    end_date: str | None = None

class TWMDDailyData(Data):
    date: str
    close: float | None = None
    # add open/high/low/volume per TWMD's real fields

class TWMDDailyFetcher(Fetcher):
    @staticmethod
    def transform_query(params): return TWMDDailyQuery(**params)
    @staticmethod
    def extract_data(query, credentials, **kw):
        key = (credentials or {}).get("twmd_api_key") or os.environ["TWMD_API_KEY"]
        p = {"symbol": query.symbol}
        if query.start_date: p["start_date"] = query.start_date
        if query.end_date:   p["end_date"]   = query.end_date
        r = requests.get("https://api.twmarketdata.com/v2/datasets/twse-daily-price",
                         params=p, headers={"X-API-Key": key}, timeout=30)
        r.raise_for_status(); return r.json().get("data", [])
    @staticmethod
    def transform_data(query, data, **kw): return [TWMDDailyData(**d) for d in data]
```
```python
# openbb_twmd/__init__.py
from openbb_core.provider.abstract.provider import Provider
from .models.daily_price import TWMDDailyFetcher
twmd_provider = Provider(
    name="twmd", website="https://twmarketdata.com",
    credentials=["api_key"], fetcher_dict={"EquityHistorical": TWMDDailyFetcher},
)
```
Register it as an entry point (`openbb_provider_extension`) in your package's `pyproject.toml`, `pip install -e .`, then `obb.equity.price.historical(symbol="2330", provider="twmd")`.

## Honesty
Not investment advice. The provider API surface follows OpenBB's current version — check OpenBB docs and adjust field models to TWMD's real fields (`/openapi.json`, `<id>.md`). Daily + fundamentals only. Path A works today with zero scaffolding.
