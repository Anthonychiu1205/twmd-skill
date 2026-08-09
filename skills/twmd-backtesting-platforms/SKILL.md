---
name: twmd-backtesting-platforms
description: >-
  Use TW Market Data (TWMD) in backtesting frameworks and trading platforms —
  backtrader (PandasData feed), vectorbt, zipline, QuantConnect/LEAN (PythonData
  custom data), and MetaTrader 5 / MetaTrader 4 (Python export → custom symbol /
  offline chart). Use when a quant wants to backtest or chart Taiwan equities with
  TWMD as the data source, or asks "TWMD with backtrader / vectorbt / QuantConnect
  / LEAN / MT5 / MT4." Daily resolution; includes survivorship-bias & point-in-
  time cautions. Confirm TWMD field names.
---

# TWMD in backtesting & platforms

Feed TWMD into backtests and trading platforms. **Not investment advice.** `export TWMD_API_KEY=sk_live_...`. Daily resolution only.

## Core: TWMD → OHLCV DataFrame (everything builds on this)
```python
import os, requests, pandas as pd
_S=requests.Session(); _S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})
def twmd_ohlcv(symbol, start=None, end=None, ds="twse-daily-price"):
    p={"symbol":symbol}
    if start:p["start_date"]=start
    if end:p["end_date"]=end
    r=_S.get(f"https://api.twmarketdata.com/v2/datasets/{ds}",params=p,timeout=30); r.raise_for_status()
    df=pd.DataFrame(r.json().get("data",[]))
    # print(df.columns.tolist())  # confirm real field names once
    m={"date":"datetime","open":"open","high":"high","low":"low","close":"close","volume":"volume"}
    df=df.rename(columns={k:v for k,v in m.items() if k in df})
    df["datetime"]=pd.to_datetime(df["datetime"]); df=df.set_index("datetime").sort_index()
    for c in ("open","high","low","close","volume"):
        if c not in df: df[c]=pd.NA
    df["openinterest"]=0
    return df[["open","high","low","close","volume","openinterest"]]
```

## backtrader
```python
import backtrader as bt
class TWMDData(bt.feeds.PandasData):
    params=(("datetime",None),("open","open"),("high","high"),("low","low"),
            ("close","close"),("volume","volume"),("openinterest","openinterest"))
cerebro=bt.Cerebro()
cerebro.adddata(TWMDData(dataname=twmd_ohlcv("2330", start="2022-01-01")))
# cerebro.addstrategy(MyStrategy); cerebro.run()
```

## vectorbt / zipline
```python
# vectorbt:
close = twmd_ohlcv("2330", start="2022-01-01")["close"].astype(float)
# pf = vbt.Portfolio.from_signals(close, entries, exits, freq="1D")
# zipline: write a custom bundle whose ingest yields OHLCV from twmd_ohlcv per sid, then `zipline ingest -b twmd`.
```

## QuantConnect / LEAN (PythonData)
```python
from AlgorithmImports import *
import json, os
class TWMDDaily(PythonData):
    def GetSource(self, config, date, isLive):
        key=os.environ.get("TWMD_API_KEY","")
        url=f"https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol={config.Symbol.Value}&start_date=2015-01-01"
        return SubscriptionDataSource(url, SubscriptionTransportMedium.RemoteFile,
                                      FileFormat.UnfoldingCollection, [f"X-API-Key: {key}"])
    def Reader(self, config, line, date, isLive):
        if not line.strip().startswith("{"): return None
        out=[]
        for row in json.loads(line).get("data",[]):
            d=TWMDDaily(); d.Symbol=config.Symbol; d.Time=datetime.strptime(row["date"],"%Y-%m-%d")
            d.Value=float(row["close"]); d["close"]=float(row["close"]); d["volume"]=float(row.get("volume",0))
            out.append(d)
        return out
# in Initialize: self.AddData(TWMDDaily, "2330", Resolution.Daily)
```
LEAN's header/UnfoldingCollection behavior varies by version; robust fallback = pre-export TWMD to CSV (see the integration skill) and point GetSource at that.

## MetaTrader 5 (custom symbol — two-stage, honest)
MT5's Python package **can't** create custom symbols; that's MQL5. So: Python exports CSV → MQL5 imports via `CustomSymbolCreate`/`CustomRatesUpdate`.
```python
import csv, datetime as dt
def export_mt5_csv(symbol, start, path=None):
    r=_S.get("https://api.twmarketdata.com/v2/datasets/twse-daily-price",
             params={"symbol":symbol,"start_date":start,"end_date":dt.date.today().isoformat()},timeout=30)
    r.raise_for_status(); rows=sorted(r.json().get("data",[]),key=lambda x:x["date"])
    path=path or f"TWMD_{symbol}.csv"
    with open(path,"w",newline="") as f:
        w=csv.writer(f); w.writerow(["date","open","high","low","close","volume"])
        for x in rows: w.writerow([x["date"],x.get("open"),x.get("high"),x.get("low"),x.get("close"),x.get("volume")])
    return path   # put in <terminal>\MQL5\Files\ ; import with an MQL5 script (CustomSymbolCreate + CustomRatesUpdate)
```
MQL5 skeleton: read the CSV, fill an `MqlRates[]` (time=StringToTime(date), OHLC, tick_volume), then `CustomRatesUpdate(sym, rates)`; open its D1 chart / use in Strategy Tester. Good for daily charts/EAs; not real-time.

## MetaTrader 4 (legacy — prefer MT5)
MT4 has no clean custom-symbol API. Export the same CSV, then use an offline chart (History Center import) or an MQL4 EA that `FileOpen(...FILE_CSV...)` and reads the values. Fiddly — **use MT5 if you can**.

## Backtest-correctness cautions (these change results)
- **Survivorship bias**: include delisted names — TWMD keeps 311 delisted histories; don't restrict to currently-listed.
- **Point-in-time**: align fundamentals to their knowledge/disclosure date, never the fiscal period (see the quant-recipes skill).
- **Adjusted vs raw**: total-return backtests → `price-enhanced` factors; raw close has split/dividend jumps.
- **data_gaps**: drop/flag; don't zero-fill into the feed.
- **Costs**: apply slippage/commission before believing an edge.

## Honesty & confirm
Not investment advice. Daily only (no intraday/real-time). MT4/MT5 custom-symbol APIs vary by build — treat MQL as a pattern. TWSE verified baseline; TPEx beta. Confirm fields: `<id>.md`, `/openapi.json`, `/llms.txt`.
