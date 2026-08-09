---
name: twmd-migrate-from-tej
description: >-
  Migrate an existing TEJ (台灣經濟新報 / tejapi) Taiwan-data pipeline to TW Market
  Data (TWMD). Use when the user pulls TW data via TEJ / TEJ+ / the tejapi Python
  package (tejapi.get('TWN/...', coid=...)) and wants a TWMD equivalent, or asks
  "TWMD version of TEJ table X." Gives a concept map (TEJ table → TWMD dataset) and
  a drop-in shim. TEJ table codes vary by subscription — confirm the user's exact
  TEJ code in their TEJ account; confirm TWMD field names in the docs.
---

# Migrate TEJ → TWMD

TEJ (台灣經濟新報) is the institutional paid incumbent; `tejapi.get('TWN/APRCD', coid='2330', ...)` style. This maps common TEJ tables to TWMD and provides a shim. **Not investment advice.** `export TWMD_API_KEY=sk_live_...`.

## Concept map (TEJ table → TWMD id)
TEJ codes differ by subscription; match by **content**, then confirm in your TEJ account.
| TEJ content | TWMD id |
| --- | --- |
| 日行情 / 未調整價 (e.g. TWN/APRCD) | `twse-daily-price` (OTC → `tpex-daily-price`) |
| 還原股價 / adjusted | `price-enhanced` |
| 月營收 (TWN/AIM1A-ish) | `monthly-revenue` |
| 財報 損益/資產/現金 (TWN/AIFINA-ish) | `income-statement` / `balance-sheet` / `cash-flow-statement` |
| 三大法人 | `institutional-flow` |
| 融資融券 | `margin-short` |
| 股利 | `dividends` |
| 本益比/淨值比 | `valuation-data` |
| 公司基本資料 | `issuer-profile` / `security-master` |
| 交易日曆 | `trading-calendar` |
(Others → search `https://twmarketdata.com/llms.txt`.)

## Drop-in shim (tejapi.get-shaped)
```python
import os, requests
_S = requests.Session(); _S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})
_MAP = {  # map YOUR TEJ codes → TWMD ids (fill from the table above)
    "TWN/APRCD": "twse-daily-price",
    "TWN/APRCDADJ": "price-enhanced",
    "TWN/AIM1A": "monthly-revenue",
    "TWN/AIFINA": "income-statement",
    # add the codes your subscription actually uses ...
}
def tej_get(code, coid=None, mdate=None, start=None, end=None, **_):
    """Mimic tejapi.get(code, coid=...). Backed by TWMD. Returns list of rows."""
    tid = _MAP.get(code)
    if not tid: raise ValueError(f"map TEJ code {code} → a TWMD id (see llms.txt)")
    p = {}
    if coid:  p["symbol"] = coid
    if start: p["start_date"] = start
    if end:   p["end_date"] = end
    if mdate and isinstance(mdate, dict):        # tejapi mdate={'gte':..,'lte':..}
        if mdate.get("gte"): p["start_date"] = mdate["gte"]
        if mdate.get("lte"): p["end_date"]   = mdate["lte"]
    r = _S.get(f"https://api.twmarketdata.com/v2/datasets/{tid}", params=p, timeout=30)
    r.raise_for_status()
    return r.json().get("data", [])

rows = tej_get("TWN/APRCD", coid="2330", start="2024-01-01")
print("fields:", list(rows[0].keys()))   # ← TEJ/TWMD column names differ; map once
```

## Why switch (honest)
TWMD is official-first with per-row source_role / lineage / freshness / data_gaps and point-in-time safety. Coverage is per-dataset (TWSE verified; TPEx beta) — confirm what you need vs your TEJ tables. TWMD may not cover every niche TEJ table; check llms.txt first.

## Honesty
Not investment advice. TEJ table codes and field names differ from TWMD — verify mapping before trusting it. data_gaps ≠ 0.
