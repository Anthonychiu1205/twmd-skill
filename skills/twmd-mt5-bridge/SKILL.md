---
name: twmd-mt5-bridge
description: >-
  Bring TW Market Data (TWMD) Taiwan daily bars into MetaTrader 5 (MT5) as a
  custom symbol for charting and Strategy-Tester backtests. Use when the user
  wants TWMD data inside MT5, asks "how do I use TWMD with MT5 / MetaTrader" or
  "can I chart Taiwan stocks in MT5 from TWMD." Explains the honest two-stage
  architecture (Python exports TWMD → CSV; MQL5 imports via CustomSymbolCreate /
  CustomRatesUpdate), with the Python export code and an MQL5 skeleton. Sets
  expectations: daily bars only, no intraday/real-time, no live TW equity feed.
---

# TWMD → MetaTrader 5 (MT5)

Chart / backtest Taiwan daily bars from TWMD inside MT5. **Not investment advice.** Set `export TWMD_API_KEY=sk_live_...`.

## Set expectations first (be honest with the user)
- MT5 is a **broker-connected trading terminal**, not a REST data store. It does not natively ingest arbitrary web data.
- The `MetaTrader5` **Python package cannot create custom symbols** — that's MQL5-side (`CustomSymbolCreate` / `CustomRatesUpdate`). So the bridge is **two stages**: Python exports TWMD → a file; an MQL5 script imports it into a **custom symbol**.
- **What works well**: viewing TWMD **daily** bars on an MT5 chart and running EA/Strategy-Tester backtests on daily timeframe.
- **What does NOT work**: real-time TW equity quotes, intraday/minute bars, live order routing on TW equities — TWMD is daily + fundamentals, and MT5's live feed is your broker's, not TWMD's.

## Stage 1 — Python: export TWMD daily bars to CSV (MT5-friendly)
```python
import os, requests, csv, datetime as dt
S = requests.Session(); S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})

def export_mt5_csv(symbol, start, end=None, path=None, ds="twse-daily-price"):
    end = end or dt.date.today().isoformat()
    r = S.get(f"https://api.twmarketdata.com/v2/datasets/{ds}",
              params={"symbol": symbol, "start_date": start, "end_date": end}, timeout=30)
    r.raise_for_status()
    rows = r.json().get("data", [])
    # print(rows[0].keys())  # ← confirm real field names (date/open/high/low/close/volume)
    rows = sorted(rows, key=lambda x: x["date"])
    path = path or f"TWMD_{symbol}.csv"
    with open(path, "w", newline="") as f:
        w = csv.writer(f)
        w.writerow(["date","open","high","low","close","volume"])   # MT5 import order
        for x in rows:
            w.writerow([x.get("date"), x.get("open"), x.get("high"),
                        x.get("low"), x.get("close"), x.get("volume")])
    print("wrote", path, len(rows), "bars")
    return path

# export_mt5_csv("2330", start="2020-01-01")
# Put the CSV in your terminal's MQL5\Files folder (File > Open Data Folder).
```

## Stage 2 — MQL5: import CSV into a custom symbol (skeleton)
Create a script in MetaEditor (adjust to your MT5 build; API details can differ by version):
```cpp
// TWMD_Import.mq5  — run once as a Script in MT5. Reads MQL5\Files\TWMD_2330.csv
void OnStart() {
   string sym = "TWMD_2330";
   CustomSymbolCreate(sym, "Custom\\TWMD");          // create under a custom group
   CustomSymbolSetInteger(sym, SYMBOL_DIGITS, 2);

   int h = FileOpen("TWMD_2330.csv", FILE_READ|FILE_CSV|FILE_ANSI, ',');
   if (h == INVALID_HANDLE) { Print("no file"); return; }
   FileReadString(h); // skip header row (6 fields)
   MqlRates rates[]; int n = 0;
   while (!FileIsEnding(h)) {
      string d = FileReadString(h);
      if (d == "") break;
      double o=FileReadNumber(h), hi=FileReadNumber(h), lo=FileReadNumber(h),
             c=FileReadNumber(h); long v=(long)FileReadNumber(h);
      ArrayResize(rates, n+1);
      rates[n].time  = StringToTime(d);              // daily bar timestamp
      rates[n].open=o; rates[n].high=hi; rates[n].low=lo; rates[n].close=c;
      rates[n].tick_volume=v; rates[n].real_volume=v; rates[n].spread=0;
      n++;
   }
   FileClose(h);
   int added = CustomRatesUpdate(sym, rates);
   PrintFormat("imported %d bars into %s", added, sym);
}
```
Then in MT5: Market Watch → show the custom symbol → open its **D1** chart, or select it in the Strategy Tester.

## Refresh workflow
- Re-run Stage 1 periodically (e.g. daily after close) to append new bars, then re-run the MQL5 import (or make the EA read the file on init). TWMD gives daily updates.

## Confirm fields
`https://twmarketdata.com/en/datasets/twse-daily-price.md` · `/openapi.json`.

## Honesty
Not investment advice. Daily bars only — no intraday/real-time via TWMD. MT5 custom-symbol APIs vary by build; treat the MQL5 above as a pattern to adapt. TWSE verified baseline; TPEx beta.
