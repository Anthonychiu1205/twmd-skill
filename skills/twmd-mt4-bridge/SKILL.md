---
name: twmd-mt4-bridge
description: >-
  Bring TW Market Data (TWMD) Taiwan daily bars into MetaTrader 4 (MT4) for
  offline charting / EA backtests. Use when the user specifically needs MT4 (not
  MT5) with TWMD data, or asks "TWMD in MT4 / MetaTrader 4." Explains the honest
  legacy path (Python exports CSV → MT4 offline chart via history file / CSV read
  in MQL4) and strongly recommends MT5 (twmd-mt5-bridge) where possible. Daily
  only; no real-time.
---

# TWMD → MetaTrader 4 (MT4)

Get TWMD daily bars into MT4. **Not investment advice.** `export TWMD_API_KEY=sk_live_...`.

## Read this first (honest)
- **MT4 is legacy.** It has no clean custom-symbol API like MT5's `CustomSymbolCreate`. Getting external data in means **offline charts** (importing an `.hst`/CSV history) or an EA that reads a CSV — clunkier than MT5.
- **Prefer MT5** (`twmd-mt5-bridge` skill) if you have any choice — the integration is far cleaner.
- **Daily only**; no real-time/intraday via TWMD.

## Stage 1 — Python: export CSV (same as MT5)
```python
import os, requests, csv, datetime as dt
S = requests.Session(); S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})

def export_csv(symbol, start, end=None, path=None):
    end = end or dt.date.today().isoformat()
    r = S.get("https://api.twmarketdata.com/v2/datasets/twse-daily-price",
              params={"symbol": symbol, "start_date": start, "end_date": end}, timeout=30)
    r.raise_for_status()
    rows = sorted(r.json().get("data", []), key=lambda x: x["date"])
    path = path or f"TWMD_{symbol}.csv"
    with open(path, "w", newline="") as f:
        w = csv.writer(f)
        for x in rows:   # MT4 offline import order: date time open high low close volume
            w.writerow([x["date"], "00:00", x.get("open"), x.get("high"),
                        x.get("low"), x.get("close"), x.get("volume")])
    return path
# put the file in MT4  <terminal>\MQL4\Files\
```

## Stage 2 — MT4 options
- **Offline chart**: use MT4's History Center / an import script to build an offline chart from the CSV, then open it (File → Open Offline). Good enough for eyeballing + EA `Strategy Tester` on D1.
- **EA reads CSV**: an MQL4 EA can `FileOpen(...FILE_CSV...)` and drive its own logic/indicators off the CSV values (without a native symbol). Simplest if you only need the numbers in code, not a broker symbol.

## Honesty
Not investment advice. MT4 external-data integration is legacy and fiddly — **use MT5 if you can**. Daily bars only; no real-time. Confirm TWMD fields at `<id>.md`.
