---
name: twmd-quantconnect-lean
description: >-
  Use TW Market Data (TWMD) as a custom data source in QuantConnect / LEAN. Use
  when the user backtests on QuantConnect or the LEAN engine and wants Taiwan
  equity data from TWMD, or asks "TWMD custom data QuantConnect / PythonData LEAN
  / import Taiwan stocks into LEAN." Provides a PythonData subclass (GetSource /
  Reader) that reads TWMD and a usage snippet. Daily resolution; confirm fields.
---

# TWMD custom data in QuantConnect / LEAN

Add Taiwan daily bars to LEAN via a `PythonData` custom type. **Not investment advice.** In QuantConnect, store your key in the project (Organization → API keys / a parameter); locally set `TWMD_API_KEY`.

## Custom data class
```python
from AlgorithmImports import *
import json, os

class TWMDDaily(PythonData):
    def GetSource(self, config, date, isLive):
        # TWMD REST returns the series; LEAN calls Reader per line of the response.
        sym = config.Symbol.Value           # e.g. "2330"
        key = os.environ.get("TWMD_API_KEY", "")
        url = (f"https://api.twmarketdata.com/v2/datasets/twse-daily-price"
               f"?symbol={sym}&start_date=2015-01-01")
        # header auth via SubscriptionDataSource headers (LEAN supports header list):
        return SubscriptionDataSource(url, SubscriptionTransportMedium.RemoteFile,
                                      FileFormat.UnfoldingCollection,
                                      [f"X-API-Key: {key}"])

    def Reader(self, config, line, date, isLive):
        if not line.strip().startswith("{"):
            return None
        body = json.loads(line)
        out = []
        for row in body.get("data", []):
            d = TWMDDaily()
            d.Symbol = config.Symbol
            d.Time = datetime.strptime(row["date"], "%Y-%m-%d")
            d.Value = float(row["close"])          # primary value
            d["open"]  = float(row.get("open", row["close"]))
            d["high"]  = float(row.get("high", row["close"]))
            d["low"]   = float(row.get("low",  row["close"]))
            d["close"] = float(row["close"])
            d["volume"]= float(row.get("volume", 0))
            out.append(d)
        return out
```

## Use it in an algorithm
```python
class TWMDExample(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2020,1,1); self.SetCash(1_000_000)
        self.tsmc = self.AddData(TWMDDaily, "2330", Resolution.Daily).Symbol
    def OnData(self, data):
        if self.tsmc in data:
            self.Debug(f"{self.Time} 2330 close {data[self.tsmc].Value}")
```

## Notes (honest)
- **Daily resolution only** — no intraday/tick in TWMD.
- LEAN's `UnfoldingCollection` / header-source behavior varies by version; if the single-response approach doesn't fit, pre-export TWMD to CSV (see `twmd-warehouse-sql`) and point `GetSource` at the local/remote CSV instead — the robust fallback.
- Include delisted names for survivorship-bias-free tests; align fundamentals to knowledge date for PIT.

## Honesty
Not investment advice. Adjust to your LEAN version; validate the data type registers and reads. Confirm TWMD fields at `<id>.md` / `/openapi.json`. data_gaps ≠ 0.
