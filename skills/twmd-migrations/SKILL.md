---
name: twmd-migrations
description: >-
  Migrate an existing Taiwan-stock data pipeline to TW Market Data (TWMD) with
  minimal code change — drop-in shims and dataset maps for FinMind, yfinance,
  twstock, TEJ (tejapi), broker APIs (Shioaji 永豐 / Fugle 富果), and global vendors
  (pandas-datareader, Alpha Vantage, Tiingo, Quandl/Nasdaq Data Link, EOD
  Historical). Use when the user already pulls TW data from any of these and wants
  to switch to TWMD, or asks "TWMD equivalent of <vendor> / how do I port my
  <vendor> code." Column names differ per source — confirm against the TWMD docs.
---

# Migrate any source → TWMD

Make TWMD a near drop-in for whatever you use now. **Not investment advice.** `export TWMD_API_KEY=sk_live_...`. General shape: their `dataset`→TWMD `id`, their `ticker`→`symbol`, token→`X-API-Key` header, dates 1:1. TWMD envelope `{data:[...], source_role, freshness, data_gaps}`. **After any mapping, `print(list(rows[0].keys()))` — column names differ.**

## What you can fully do for the user
- Port their existing code with the smallest possible diff: identify their current source, give the exact dataset map + a drop-in shim, and show the before/after.
- Confirm every column mapping live (`print(keys)` → `<id>.md` / openapi.json) so nothing silently breaks.
- Say honestly what does NOT map (real-time/intraday/orders/non-TW) and what to keep the old vendor for.
- Cover FinMind, yfinance, twstock, TEJ, Shioaji/Fugle, and pandas-datareader / Alpha Vantage / Tiingo / Quandl / EODHD.

## Shared helper
```python
import os, requests
_S = requests.Session(); _S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})
def twmd(dataset, **p):
    r=_S.get(f"https://api.twmarketdata.com/v2/datasets/{dataset}", params=p, timeout=30)
    r.raise_for_status(); return r.json().get("data", [])
```

## FinMind
`GET api.finmindtrade.com/api/v4/data?dataset=<Name>&data_id=<t>&token=..` → TWMD id + symbol.
Map: TaiwanStockPrice→`twse-daily-price` (OTC→`tpex-daily-price`) · adjusted→`price-enhanced` · TaiwanStockMonthRevenue→`monthly-revenue` · FinancialStatements→`income-statement` · BalanceSheet→`balance-sheet` · CashFlowsStatement→`cash-flow-statement` · InstitutionalInvestorsBuySell→`institutional-flow` · MarginPurchaseShortSale→`margin-short` · Dividend→`dividends` · PER→`valuation-data` · News→`company-news` · Info→`issuer-profile` · TradingDate→`trading-calendar`.
```python
_FM = {"TaiwanStockPrice":"twse-daily-price","TaiwanStockMonthRevenue":"monthly-revenue",
       "InstitutionalInvestorsBuySell":"institutional-flow","TaiwanStockDividend":"dividends"}  # extend
def finmind_data(dataset, data_id=None, start_date=None, end_date=None, **_):
    p={}; 
    if data_id: p["symbol"]=data_id
    if start_date: p["start_date"]=start_date
    if end_date: p["end_date"]=end_date
    return twmd(_FM[dataset], **p)
```

## yfinance  (`2330.TW` / `6488.TWO`)
```python
import pandas as pd
class TWMDTicker:
    def __init__(self,s):
        s=s.upper()
        self.symbol,self.ds=(s[:-4],"tpex-daily-price") if s.endswith(".TWO") else \
                            (s[:-3] if s.endswith(".TW") else s,"twse-daily-price")
    def history(self, start=None, end=None, limit=None):
        p={"symbol":self.symbol}
        if start:p["start_date"]=start
        if end:p["end_date"]=end
        if limit:p["limit"]=limit
        df=pd.DataFrame(twmd(self.ds,**p))
        m={"date":"Date","open":"Open","high":"High","low":"Low","close":"Close","volume":"Volume"}
        df=df.rename(columns={k:v for k,v in m.items() if k in df})
        if "Date" in df: df=df.assign(Date=pd.to_datetime(df["Date"])).set_index("Date").sort_index()
        return df[[c for c in ["Open","High","Low","Close","Volume"] if c in df]]
# literal swap: class yf: Ticker = TWMDTicker   → yf.Ticker("2330.TW").history(...) runs on TWMD
```
No intraday/real-time/non-TW. Adjusted ≈ `price-enhanced` (separate dataset, not a flag).

## twstock
```python
class TWMDStock:
    def __init__(self,sid,otc=False): self.sid=sid; self.ds="tpex-daily-price" if otc else "twse-daily-price"; self._r=[]
    def fetch_from(self,y,m): self._r=sorted(twmd(self.ds,symbol=self.sid,start_date=f"{y:04d}-{m:02d}-01"),key=lambda x:x["date"]); return self._r
    @property
    def date(self): return [x["date"] for x in self._r]
    @property
    def price(self): return [x.get("close") for x in self._r]     # .capacity ≈ volume, .high/.low/.open likewise
```
No real-time (twstock.realtime has no TWMD equivalent).

## TEJ (tejapi)
Match by content, confirm your TEJ code in your account: 日行情→`twse-daily-price` · 還原→`price-enhanced` · 月營收→`monthly-revenue` · 財報→`income-statement`/`balance-sheet`/`cash-flow-statement` · 三大法人→`institutional-flow` · 融資融券→`margin-short` · 股利→`dividends` · PER/PBR→`valuation-data`.
```python
_TEJ={"TWN/APRCD":"twse-daily-price","TWN/AIM1A":"monthly-revenue"}   # map YOUR codes
def tej_get(code, coid=None, start=None, end=None, mdate=None, **_):
    p={}
    if coid:p["symbol"]=coid
    if start:p["start_date"]=start
    if end:p["end_date"]=end
    if isinstance(mdate,dict): p["start_date"]=mdate.get("gte",p.get("start_date")); p["end_date"]=mdate.get("lte",p.get("end_date"))
    return twmd(_TEJ[code], **p)
```

## Broker APIs (Shioaji 永豐 / Fugle 富果)
TWMD replaces **historical daily** fetch only — keep the broker for **live quotes + orders**. Maps: daily kbars/candles→`twse-daily-price`/`tpex-daily-price`; adjusted→`price-enhanced`; fundamentals/flow→respective datasets. No real-time/intraday/order-routing via TWMD.
```python
import pandas as pd
def twmd_daily(symbol, start, end=None, otc=False):
    p={"symbol":symbol,"start_date":start}
    if end:p["end_date"]=end
    return pd.DataFrame(twmd("tpex-daily-price" if otc else "twse-daily-price", **p))
# Shioaji api.kbars(...D1) / Fugle historical.candles(...) → twmd_daily(...)
```

## Global vendors (pandas-datareader / Alpha Vantage / Tiingo / Quandl / EODHD)
```python
import pandas as pd
def get_data(symbol, start=None, end=None, dataset="twse-daily-price"):
    p={"symbol":symbol}
    if start:p["start_date"]=start
    if end:p["end_date"]=end
    df=pd.DataFrame(twmd(dataset,**p))
    if "date" in df: df=df.assign(date=pd.to_datetime(df["date"])).set_index("date").sort_index()
    return df
# web.DataReader("2330.TW","yahoo",...) | AlphaVantage get_daily("2330") | Tiingo get_dataframe("2330")
#   | quandl.get("XTAI/2330") | EODHD get_eod("2330.TW")   → get_data("2330", start, end)
```
Global vendors' TW coverage is thin/delayed/secondary; TWMD is official-first + keeps 311 delisted histories. But TWMD is **Taiwan-only + daily + fundamentals** — keep the other vendor for US/global/intraday.

## Honesty & confirm
Not investment advice. Field names/units differ per source — map after `print(columns)`, don't trust blindly. data_gaps ≠ 0. Confirm at `/openapi.json`, `<id>.md`, `/llms.txt`. Unmapped dataset → search `/llms.txt`.
