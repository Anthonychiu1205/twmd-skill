---
name: twmd-quant-recipes
description: >-
  Quant factor cookbook for TW Market Data (TWMD) — ready-to-run Python recipes
  for Taiwan-equity research: monthly-revenue YoY growth factor, institutional
  (三大法人) flow momentum, valuation screens (PER/PBR/yield), cross-sectional
  factor ranking, multi-factor combination, and a point-in-time-safe backtest
  skeleton. Use when the user (a quant/researcher already past onboarding) wants
  to BUILD a factor, screen, rank, or backtest with TWMD data, or asks "how do I
  compute X factor / screen for Y / avoid look-ahead bias" on api.twmarketdata.com.
  Assumes a key is set. For first-time setup use twmd-taiwan-market-data; for
  production client/pipeline engineering use twmd-api-integration.
---

# TWMD quant factor cookbook

Ready-to-run Taiwan-equity factor recipes on TWMD. **Not investment advice.** Assumes `export TWMD_API_KEY=sk_live_...`.

## What you can fully do for the user
- Build any factor end-to-end and hand a complete runnable script: growth (revenue YoY), flow (三大法人), value (PER/PBR/yield), quality, momentum, and multi-factor composites.
- Screen/rank a universe cross-sectionally; z-score and combine signals.
- Stand up a point-in-time-safe backtest that avoids look-ahead and survivorship bias.
- Resolve exact field names live (llms.txt → `<id>.md` → openapi.json) before computing — never invent a column.
- Hand off plumbing (pagination, async, warehouse, scheduling) to **twmd-integration**; framework wiring (backtrader/QuantConnect) to **twmd-backtesting-platforms**.

## Ground rules (state to the user, they matter for correct research)
- **Confirm field names before computing.** This skill does not hard-code every dataset's field names — they differ per dataset. Each recipe prints `list(rows[0].keys())` first; pick the real field, or read the dataset's docs page markdown (`https://twmarketdata.com/en/datasets/<id>.md`) or `/openapi.json`.
- **Point-in-time / no look-ahead.** Every row carries `knowledge_date` / `freshness`. Fundamentals (revenue, financials) are known only *after* their disclosure date — when backtesting, align a factor to the date it became known, not the fiscal period it describes. Never join a fiscal-period value onto a trading date earlier than its disclosure.
- **data_gaps are signals, not zeros.** Drop/flag missing values; never impute silently.
- **Coverage is partial.** TWSE is the verified baseline; check `production_ready` (★ datasets in the onboarding skill) and per-dataset coverage. Don't assume full universe/history.
- **Join on ticker**, not company name.

## Reusable client (paste once)
```python
import os, time, requests

BASE = "https://api.twmarketdata.com/v2/datasets"
KEY  = os.environ["TWMD_API_KEY"]
S = requests.Session()
S.headers.update({"X-API-Key": KEY})

def twmd(dataset, **params):
    """GET one dataset; returns the list under 'data'. Retries once on 429."""
    for attempt in range(3):
        r = S.get(f"{BASE}/{dataset}", params=params, timeout=30)
        if r.status_code == 429:
            time.sleep(2 ** attempt)          # back off on rate limit
            continue
        r.raise_for_status()
        body = r.json()
        return body.get("data", body)         # envelope: {dataset, source_role, freshness, data:[...]}
    raise RuntimeError("rate-limited after retries")
```

## Recipe 1 — Monthly-revenue YoY growth factor (free-open dataset)
Taiwan's monthly revenue is a unique high-frequency fundamental. YoY growth is a classic momentum/quality signal.
```python
def revenue_yoy(symbol, months=24):
    rows = twmd("monthly-revenue", symbol=symbol, limit=months)
    if not rows: return []
    print("fields:", list(rows[0].keys()))     # ← confirm the revenue + period field names once
    key_period = "date" if "date" in rows[0] else "period"
    field_rev  = next((f for f in ("revenue","monthly_revenue","net_revenue") if f in rows[0]), None)
    rows = sorted(rows, key=lambda x: x[key_period])
    out = []
    for i in range(12, len(rows)):
        now, prior = rows[i].get(field_rev), rows[i-12].get(field_rev)
        if now and prior:
            out.append((rows[i][key_period], (now/prior - 1) * 100))
    return out

for period, yoy in revenue_yoy("2330")[-6:]:
    print(period, f"YoY {yoy:+.1f}%")
```
Cross-sectional version — rank a universe by latest YoY:
```python
universe = ["2330","2317","2454","2603","2412","2882"]   # your ticker list
scores = {}
for s in universe:
    y = revenue_yoy(s, months=14)
    if y: scores[s] = y[-1][1]                 # latest YoY
ranked = sorted(scores.items(), key=lambda kv: kv[1], reverse=True)
print("Revenue-YoY leaders:", ranked)
```

## Recipe 2 — Institutional (三大法人) flow momentum
Foreign/investment-trust/dealer net buying is Taiwan's daily "smart money" signal (vs US 13F's 45-day lag). Sum recent net flow as a momentum factor.
```python
def inst_flow_momentum(symbol, days=20):
    rows = twmd("institutional-flow", symbol=symbol, limit=days)
    if not rows: return None
    print("fields:", list(rows[0].keys()))     # ← confirm net-buy field names (foreign/trust/dealer)
    # sum whatever net columns exist; adjust to real field names from the docs page:
    net_fields = [f for f in rows[0] if "net" in f.lower()]
    total = sum((r.get(f) or 0) for r in rows for f in net_fields)
    return total

for s in ["2330","2317","2454"]:
    print(s, "20d net inst flow:", inst_flow_momentum(s))
```

## Recipe 3 — Valuation screen (PER / PBR / yield)
```python
def valuation(symbol):
    rows = twmd("valuation-data", symbol=symbol, limit=1)
    if not rows: return None
    print("fields:", list(rows[0].keys()))     # ← confirm per / pbr / dividend_yield field names
    return rows[0]

# Screen: cheap + income. Adjust field names to the real ones printed above.
universe = ["2330","2317","2454","2603","2412","2882","1301","2882"]
picks = []
for s in universe:
    v = valuation(s)
    if not v: continue
    per = v.get("per") or v.get("pe_ratio")
    pbr = v.get("pbr") or v.get("pb_ratio")
    dy  = v.get("dividend_yield") or v.get("yield")
    if per and pbr and dy and per < 25 and pbr < 3 and dy > 3:
        picks.append((s, per, pbr, dy))
print("value+income picks:", picks)
```

## Recipe 4 — Cross-sectional multi-factor rank
Combine factors into one score by z-scoring each and averaging (equal-weight; adjust weights as you like).
```python
import statistics as st

def zscore(d):                                  # d: {ticker: value}
    vals = [v for v in d.values() if v is not None]
    if len(vals) < 2: return {k: 0.0 for k in d}
    mu, sd = st.mean(vals), st.pstdev(vals) or 1.0
    return {k: ((v - mu)/sd if v is not None else 0.0) for k, v in d.items()}

universe = ["2330","2317","2454","2603","2412","2882"]
f_growth = {s: (revenue_yoy(s, 14)[-1][1] if revenue_yoy(s,14) else None) for s in universe}
f_flow   = {s: inst_flow_momentum(s) for s in universe}
# higher YoY good, higher net-inflow good → both positive-is-good
zg, zf = zscore(f_growth), zscore(f_flow)
combined = {s: 0.5*zg[s] + 0.5*zf[s] for s in universe}
print("multi-factor rank:", sorted(combined.items(), key=lambda kv: kv[1], reverse=True))
```

## Recipe 5 — Point-in-time-safe backtest skeleton
The single most important correctness rule. Align each fundamental to its **disclosure/known date**, then form portfolios on dates when the data was actually available.
```python
# Pseudocode-level skeleton — fill in with real disclosure/knowledge_date fields.
# 1. Pull price history (twse-daily-price) for the universe over your window.
# 2. Pull the factor (e.g. monthly-revenue) WITH its knowledge_date / disclosure date.
# 3. For each rebalance date T:
#      - use ONLY factor rows whose knowledge_date <= T   ← prevents look-ahead
#      - rank universe, form long (top quantile) / short (bottom)
#      - hold to next rebalance, compute forward return from prices
# 4. Chain period returns; report CAGR, Sharpe, max drawdown, turnover.
#
# Guardrails:
# - Never use a fiscal-period value before its knowledge_date.
# - Include delisted names (twse-daily-price keeps 311 delisted histories) to avoid survivorship bias.
# - Drop rows flagged in data_gaps; don't zero-fill.
# - Costs: apply realistic slippage/commission before claiming edge.
```

## Where to confirm exact fields
- Dataset docs (markdown): `https://twmarketdata.com/en/datasets/<id>.md`
- OpenAPI (params + response schema): `https://twmarketdata.com/openapi.json`
- Full index of all 84 storefront datasets: `https://twmarketdata.com/llms.txt`

## Honesty
Not investment advice. Backtests are not forward returns. Coverage/history are partial and per-dataset; verify before drawing conclusions. TWSE is the verified baseline; TPEx/adjusted prices are beta/deferred.
