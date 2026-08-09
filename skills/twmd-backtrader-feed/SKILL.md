---
name: twmd-backtrader-feed
description: >-
  Feed TW Market Data (TWMD) into backtesting frameworks — backtrader (PandasData
  feed), plus vectorbt and zipline notes. Use when a quant wants to backtest
  Taiwan equities in backtrader/vectorbt/zipline using TWMD as the data source,
  or asks "how do I use TWMD with backtrader / how do I make a data feed." Gives a
  TWMD→OHLCV DataFrame helper and a ready backtrader data feed, with
  survivorship-bias and point-in-time cautions. Confirm TWMD field names.
---

# TWMD → backtesting frameworks

Use TWMD as the data source for backtrader / vectorbt / zipline. **Not investment advice.** Set `export TWMD_API_KEY=sk_live_...`. Needs `pandas` (+ `backtrader`).

## Core: TWMD → OHLCV DataFrame (everything builds on this)
```python
import os, requests, pandas as pd
_S = requests.Session(); _S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})

def twmd_ohlcv(symbol, start=None, end=None, ds="twse-daily-price"):
    p = {"symbol": symbol}
    if start: p["start_date"] = start
    if end:   p["end_date"] = end
    r = _S.get(f"https://api.twmarketdata.com/v2/datasets/{ds}", params=p, timeout=30)
    r.raise_for_status()
    rows = r.json().get("data", [])
    df = pd.DataFrame(rows)
    # print(df.columns.tolist())   # ← confirm real TWMD field names once
    m = {"date":"datetime","open":"open","high":"high","low":"low","close":"close","volume":"volume"}
    df = df.rename(columns={k:v for k,v in m.items() if k in df.columns})
    df["datetime"] = pd.to_datetime(df["datetime"])
    df = df.set_index("datetime").sort_index()
    for c in ("open","high","low","close","volume"):
        if c not in df: df[c] = pd.NA         # backtrader tolerates NaN; fill/clean as needed
    df["openinterest"] = 0
    return df[["open","high","low","close","volume","openinterest"]]
```

## backtrader feed
```python
import backtrader as bt

class TWMDData(bt.feeds.PandasData):
    params = (("datetime", None), ("open","open"), ("high","high"),
              ("low","low"), ("close","close"), ("volume","volume"),
              ("openinterest","openinterest"))

# usage
cerebro = bt.Cerebro()
df = twmd_ohlcv("2330", start="2022-01-01", end="2024-12-31")
cerebro.adddata(TWMDData(dataname=df))
# cerebro.addstrategy(MyStrategy); cerebro.run()
```

## vectorbt
```python
import vectorbt as vbt
close = twmd_ohlcv("2330", start="2022-01-01")["close"].astype(float)
# entries/exits = your signal ...
# pf = vbt.Portfolio.from_signals(close, entries, exits, freq="1D")
```

## zipline (note)
Zipline needs a **bundle**: write a custom bundle `ingest` that yields OHLCV DataFrames from `twmd_ohlcv` per sid, register it, then `zipline ingest -b twmd`. More setup than backtrader — only if you're already on zipline.

## Backtest-correctness cautions (say these — they change results)
- **Survivorship bias**: include delisted names. TWMD keeps 311 delisted histories — use them, don't restrict to currently-listed tickers.
- **Point-in-time**: for fundamentals-driven strategies, align each fundamental to its knowledge/disclosure date (see the `twmd-quant-recipes` skill), never the fiscal period.
- **Adjusted vs raw**: for total-return backtests use `price-enhanced` adjustment factors; raw close has splits/dividends jumps.
- **data_gaps**: drop/flag; don't zero-fill into the feed.
- **Costs**: add slippage/commission before believing an edge.

## Confirm fields
`https://twmarketdata.com/en/datasets/twse-daily-price.md` · `/openapi.json` · full index `/llms.txt`.

## Honesty
Not investment advice. Daily frequency (no intraday). TWSE verified baseline; TPEx beta. Backtests are not forward returns.
