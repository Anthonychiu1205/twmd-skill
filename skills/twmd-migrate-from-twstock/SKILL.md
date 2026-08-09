---
name: twmd-migrate-from-twstock
description: >-
  Migrate from the popular twstock Python library to TW Market Data (TWMD). Use
  when the user uses twstock (from twstock import Stock; Stock('2330').price /
  .fetch_from) for Taiwan daily prices and wants a TWMD-backed replacement with a
  similar interface, or asks "TWMD version of twstock." Provides a Stock-like shim
  returning price/date/high/low/close/capacity lists. TWMD is daily + official-
  first (no real-time BestFourPoint). Confirm TWMD field names.
---

# Migrate twstock → TWMD

`twstock` is a common free TW price library. This gives a TWMD-backed `Stock`-like shim. **Not investment advice.** `export TWMD_API_KEY=sk_live_...`. Needs `pandas` optional.

## Mapping
- `Stock('2330').price` / `.date` / `.high` / `.low` / `.close` / `.capacity` → columns from TWMD `twse-daily-price`.
- `.fetch_from(2020, 1)` → TWMD `start_date="2020-01-01"`.
- twstock realtime (`twstock.realtime.get`) has **no TWMD equivalent** (TWMD is daily, not real-time).

## Drop-in shim
```python
import os, requests
_S = requests.Session(); _S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})

class TWMDStock:
    def __init__(self, sid, otc=False):
        self.sid = sid
        self.ds = "tpex-daily-price" if otc else "twse-daily-price"
        self._rows = []
    def fetch_from(self, year, month):
        start = f"{year:04d}-{month:02d}-01"
        r = _S.get(f"https://api.twmarketdata.com/v2/datasets/{self.ds}",
                   params={"symbol": self.sid, "start_date": start}, timeout=30)
        r.raise_for_status()
        self._rows = sorted(r.json().get("data", []), key=lambda x: x["date"])
        # print(self._rows[0].keys())  # confirm real field names once
        return self._rows
    def _col(self, name):
        return [x.get(name) for x in (self._rows or self.fetch_from(2020,1))]
    @property
    def date(self):     return [x["date"] for x in self._rows]
    @property
    def price(self):    return self._col("close")     # twstock .price == close
    @property
    def close(self):    return self._col("close")
    @property
    def high(self):     return self._col("high")
    @property
    def low(self):      return self._col("low")
    @property
    def open(self):     return self._col("open")
    @property
    def capacity(self): return self._col("volume")    # twstock .capacity ~ volume

# BEFORE: from twstock import Stock; s = Stock('2330'); s.fetch_from(2024,1); s.price
# AFTER:
s = TWMDStock('2330'); s.fetch_from(2024, 1)
print(s.date[-3:], s.price[-3:])
```

## Differences (honest)
- **No real-time** (twstock realtime / BestFourPoint have no TWMD equivalent — TWMD is daily).
- twstock `.capacity` is board-lot volume; confirm TWMD's volume unit on the docs page.
- TWMD adds source_role / freshness / data_gaps and keeps delisted histories — better for backtests.
- Confirm fields: `https://twmarketdata.com/en/datasets/twse-daily-price.md`.

## Honesty
Not investment advice. Daily only. Verify field names/units. data_gaps ≠ 0.
