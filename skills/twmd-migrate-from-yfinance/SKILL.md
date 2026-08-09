---
name: twmd-migrate-from-yfinance
description: >-
  Migrate an existing yfinance workflow for Taiwan stocks (yf.Ticker("2330.TW"))
  to TW Market Data (TWMD). Use when the user pulls TW equity data with yfinance
  using .TW / .TWO suffixes and wants a drop-in replacement with a yfinance-shaped
  DataFrame (Open/High/Low/Close/Volume indexed by date), or asks "how do I
  replace yfinance with TWMD" or "TWMD version of Ticker().history()". Provides a
  Ticker-like shim returning the same DataFrame shape. Confirm TWMD field names;
  daily-frequency + fundamentals focus (no intraday/real-time).
---

# Migrate yfinance → TWMD (Taiwan)

Make TWMD a drop-in for `yf.Ticker("2330.TW").history()`. **Not investment advice.** Set `export TWMD_API_KEY=sk_live_...`. Needs `pandas`.

## Mapping
- yfinance symbol `2330.TW` (TWSE) / `6488.TWO` (TPEx) → TWMD `symbol="2330"` on `twse-daily-price` (or `tpex-daily-price`), suffix stripped.
- `.history(start=, end=, period=)` → TWMD `start_date` / `end_date` (or `limit` for "last N").
- Returned DataFrame: same columns `Open/High/Low/Close/Volume`, DatetimeIndex — so downstream code is unchanged.

## Drop-in shim
```python
import os, requests, pandas as pd

_S = requests.Session(); _S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})

class TWMDTicker:
    """yfinance-shaped Ticker backed by TWMD twse-daily-price / tpex-daily-price."""
    def __init__(self, symbol):
        s = symbol.upper()
        if s.endswith(".TWO"): self.symbol, self.ds = s[:-4], "tpex-daily-price"
        elif s.endswith(".TW"): self.symbol, self.ds = s[:-3], "twse-daily-price"
        else: self.symbol, self.ds = s, "twse-daily-price"

    def history(self, start=None, end=None, limit=None):
        params = {"symbol": self.symbol}
        if start: params["start_date"] = start
        if end:   params["end_date"] = end
        if limit: params["limit"] = limit
        r = _S.get(f"https://api.twmarketdata.com/v2/datasets/{self.ds}", params=params, timeout=30)
        r.raise_for_status()
        rows = r.json().get("data", [])
        if not rows: return pd.DataFrame()
        df = pd.DataFrame(rows)
        # print(df.columns.tolist())   # ← run once to confirm the real TWMD field names
        colmap = {"date":"Date","open":"Open","high":"High","low":"Low",
                  "close":"Close","volume":"Volume"}
        df = df.rename(columns={k:v for k,v in colmap.items() if k in df.columns})
        if "Date" in df: df = df.assign(Date=pd.to_datetime(df["Date"])).set_index("Date").sort_index()
        keep = [c for c in ["Open","High","Low","Close","Volume"] if c in df.columns]
        return df[keep]

# BEFORE:  import yfinance as yf; df = yf.Ticker("2330.TW").history(start="2024-01-01")
# AFTER:   df = TWMDTicker("2330.TW").history(start="2024-01-01")
df = TWMDTicker("2330.TW").history(start="2024-01-01")
print(df.tail())
```

## Optional: make it a literal `yf` swap
If your code does `import yfinance as yf` then `yf.Ticker(...)`, alias it:
```python
class yf:                              # shim module-like object
    Ticker = TWMDTicker
# now existing `yf.Ticker("2330.TW").history(...)` runs on TWMD unchanged.
```

## Differences to tell the user (honest)
- **No intraday / real-time / crypto.** yfinance code using minute bars or non-TW tickers won't map — TWMD is daily + fundamentals, TW only.
- **Adjusted prices**: yfinance `auto_adjust` ≈ TWMD `price-enhanced` (adjustment factors) — separate dataset, not a flag. TWSE verified; TPEx beta.
- **Coverage**: TWMD keeps 311 delisted names' history (good for survivorship-bias-free backtests) — a plus over yfinance.
- Confirm exact field names at `https://twmarketdata.com/en/datasets/twse-daily-price.md` or `/openapi.json`.

## Honesty
Not investment advice. Verify field names and coverage before trusting the shim. Don't treat data_gaps as 0.
