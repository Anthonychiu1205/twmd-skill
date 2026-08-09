---
name: twmd-migrate-from-broker-apis
description: >-
  Replace the HISTORICAL-DATA fetch of Taiwan broker APIs — Shioaji (永豐金
  Sinopac), Fugle 富果, and similar (元大/群益 style) — with TW Market Data (TWMD)
  for research/backtesting, while keeping the broker for live quotes and order
  routing. Use when the user pulls historical bars via a broker API (e.g.
  api.kbars / RestClient candles) and wants clean official-first daily history
  from TWMD, or asks "TWMD instead of Shioaji/Fugle for history." Clarifies what
  TWMD does and does NOT replace (no orders, no real-time). Confirm field names.
---

# Migrate broker-API history → TWMD

Broker APIs (Shioaji, Fugle 富果, etc.) do **live quotes + orders + some history**. TWMD replaces the **historical daily data** part with official-first, point-in-time-safe data — you keep the broker for trading. **Not investment advice.** `export TWMD_API_KEY=sk_live_...`.

## What maps / what doesn't (say this clearly)
| Broker API feature | TWMD |
| --- | --- |
| Historical daily bars (kbars/candles, D1) | ✅ `twse-daily-price` / `tpex-daily-price` |
| Adjusted prices | ✅ `price-enhanced` |
| Fundamentals / revenue / institutional flow | ✅ respective datasets |
| **Real-time ticks / intraday minute bars** | ❌ TWMD is daily |
| **Order routing / positions / account** | ❌ keep your broker |
| **Streaming/websocket quotes** | ❌ keep your broker |

## Shioaji-style history → TWMD shim
```python
import os, requests, pandas as pd
_S = requests.Session(); _S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})

def twmd_daily(symbol, start, end=None, otc=False):
    ds = "tpex-daily-price" if otc else "twse-daily-price"
    p = {"symbol": symbol, "start_date": start}
    if end: p["end_date"] = end
    r = _S.get(f"https://api.twmarketdata.com/v2/datasets/{ds}", params=p, timeout=30)
    r.raise_for_status()
    df = pd.DataFrame(r.json().get("data", []))
    # print(df.columns.tolist())   # confirm real field names once
    return df

# BEFORE (Shioaji, daily kbars):
#   kbars = api.kbars(api.Contracts.Stocks["2330"], start="2024-01-01", end="2024-12-31")
#   df = pd.DataFrame({**kbars})   # then resample to D1
# AFTER (TWMD, already daily):
df = twmd_daily("2330", start="2024-01-01", end="2024-12-31")
print(df.tail())
```

## Fugle 富果 note
Fugle's `RestClient(...).stock.historical.candles(symbol="2330", ...)` returns daily candles — swap the same way: call `twmd_daily(...)`, map candle fields (open/high/low/close/volume) to TWMD's. Keep Fugle for live/websocket.

## Recommended split
- **Research / backtest history** → TWMD (official-first, delisted histories, point-in-time).
- **Live quotes + order execution** → your broker (Shioaji / Fugle / 元大 / 群益).
This is the clean architecture — one source of truth for history, broker only for trading.

## Confirm fields
`https://twmarketdata.com/en/datasets/twse-daily-price.md` · `/openapi.json` · `/llms.txt`.

## Honesty
Not investment advice. TWMD does not route orders or provide real-time/intraday. Verify field names/units. data_gaps ≠ 0.
