---
name: twmd-api-integration
description: >-
  Production integration guide for the TW Market Data (TWMD) REST API
  (api.twmarketdata.com). Use when an engineer wants to WIRE TWMD into an app or
  data pipeline: a robust HTTP client (session, timeouts, ret/backoff on 429/5xx),
  pagination via limit/offset, error handling (401/402/429 + envelope), preserving
  source_role / lineage / freshness / data_gaps, incremental/date-windowed fetch,
  async/concurrent throughput, writing to DataFrame / Parquet / SQL, key
  management and rotation, quota-friendly access patterns, and a Node/TypeScript
  equivalent. For first-time onboarding use twmd-taiwan-market-data; for research
  factor code use twmd-quant-recipes.
---

# TWMD production integration

Wire TWMD into apps/pipelines correctly. **Not investment advice.** Base `https://api.twmarketdata.com`, auth header `X-API-Key: sk_live_...` (never in the URL).

## Response envelope (build your parser around this)
```json
{ "dataset": "twse-daily-price", "source_role": "canonical",
  "freshness": "...", "lineage": { "trace_id": "..." },
  "data_gaps": [ ... ], "data": [ { ... } ] }
```
Preserve `source_role`, `freshness`, `lineage.trace_id`, `data_gaps` alongside every row you store — they're needed for audit, debugging latency, and correct research. **Never treat `data_gaps` as 0.**

## Errors to handle
- **401 missing_api_key** — key missing/invalid (5 demo symbols 2330/2317/2454/0050/2603 exempt on twse-daily-price).
- **402 not_entitled_for_dataset** — plan lacks the dataset / free quota used → surface an upgrade path, don't retry.
- **429** — rate or monthly-usage quota exceeded → back off (respect `Retry-After` if present), then retry.
- **5xx** — transient → retry with backoff.

## Robust Python client (session + retries + backoff)
```python
import os, time, random, requests

class TWMD:
    def __init__(self, key=None, base="https://api.twmarketdata.com/v2/datasets", timeout=30):
        self.base, self.timeout = base, timeout
        self.s = requests.Session()
        self.s.headers.update({"X-API-Key": key or os.environ["TWMD_API_KEY"]})

    def get(self, dataset, **params):
        for attempt in range(5):
            r = self.s.get(f"{self.base}/{dataset}", params=params, timeout=self.timeout)
            if r.status_code == 429 or r.status_code >= 500:
                wait = float(r.headers.get("Retry-After", 0)) or (2 ** attempt + random.random())
                time.sleep(wait); continue
            if r.status_code == 401:
                raise PermissionError("missing/invalid API key")
            if r.status_code == 402:
                raise PermissionError(f"not entitled for {dataset} (plan/quota) — upgrade at /pricing")
            r.raise_for_status()
            return r.json()                       # full envelope
        raise RuntimeError(f"{dataset}: retries exhausted")

    def rows(self, dataset, **params):
        return self.get(dataset, **params).get("data", [])

tw = TWMD()
print(tw.rows("twse-daily-price", symbol="2330", limit=5))
```

## Pagination (limit / offset)
```python
def paginate(tw, dataset, page=1000, **params):
    offset = 0
    while True:
        batch = tw.rows(dataset, limit=page, offset=offset, **params)
        if not batch: break
        yield from batch
        if len(batch) < page: break               # last page
        offset += page

rows = list(paginate(tw, "twse-daily-price", symbol="2330",
                     start_date="2024-01-01", end_date="2024-12-31"))
print(len(rows), "rows")
```

## Incremental / date-windowed fetch (only pull what's new)
```python
import datetime as dt

def fetch_since(tw, dataset, symbol, last_date):
    start = (dt.date.fromisoformat(last_date) + dt.timedelta(days=1)).isoformat()
    today = dt.date.today().isoformat()
    if start > today: return []
    return list(paginate(tw, dataset, symbol=symbol, start_date=start, end_date=today))

# store max(date) per (dataset, symbol) in your DB, then only fetch forward.
```

## Concurrent throughput (async, quota-aware)
Use a semaphore so you stay under the rate limit; TWMD's model rewards **focused** queries (few tickers, incremental) over sweeping the whole universe.
```python
import asyncio, httpx, os

async def afetch(client, sem, dataset, **params):
    async with sem:
        for attempt in range(5):
            r = await client.get(f"/v2/datasets/{dataset}", params=params, timeout=30)
            if r.status_code == 429 or r.status_code >= 500:
                await asyncio.sleep(2 ** attempt); continue
            r.raise_for_status()
            return r.json().get("data", [])
        return []

async def main(symbols):
    headers = {"X-API-Key": os.environ["TWMD_API_KEY"]}
    sem = asyncio.Semaphore(4)                     # cap concurrency → respect rate limit
    async with httpx.AsyncClient(base_url="https://api.twmarketdata.com", headers=headers) as c:
        tasks = [afetch(c, sem, "monthly-revenue", symbol=s, limit=24) for s in symbols]
        return await asyncio.gather(*tasks)

# asyncio.run(main(["2330","2317","2454"]))
```

## Land it in a DataFrame / Parquet / SQL (keep provenance)
```python
import pandas as pd

env = tw.get("twse-daily-price", symbol="2330", limit=100)
df = pd.DataFrame(env["data"])
df["source_role"] = env.get("source_role")        # carry provenance columns
df["freshness"]   = env.get("freshness")
df["trace_id"]    = env.get("lineage", {}).get("trace_id")
df.to_parquet("twse_2330.parquet", index=False)

# SQL (SQLAlchemy): df.to_sql("prices", engine, if_exists="append", index=False)
# Recommended keys: unique (dataset, ticker, date). Store data_gaps separately, not as rows.
```

## Node / TypeScript equivalent
```ts
const BASE = "https://api.twmarketdata.com/v2/datasets";
async function twmd(dataset: string, params: Record<string,string|number>) {
  const qs = new URLSearchParams(params as any).toString();
  for (let attempt = 0; attempt < 5; attempt++) {
    const r = await fetch(`${BASE}/${dataset}?${qs}`, {
      headers: { "X-API-Key": process.env.TWMD_API_KEY! },
    });
    if (r.status === 429 || r.status >= 500) { await new Promise(s => setTimeout(s, 2**attempt*1000)); continue; }
    if (r.status === 401) throw new Error("missing/invalid API key");
    if (r.status === 402) throw new Error(`not entitled for ${dataset} — upgrade`);
    if (!r.ok) throw new Error(`${r.status}`);
    const body = await r.json();
    return body.data ?? body;                      // envelope { data: [...] }
  }
  throw new Error("retries exhausted");
}
// await twmd("twse-daily-price", { symbol: "2330", limit: 10 });
```

## Key management
- Store `TWMD_API_KEY` in env/secret manager — **never commit it, never in URLs/logs**.
- One key per environment (dev/stage/prod); rotate/revoke self-serve at `/dashboard`.
- On suspected leak: revoke that key, issue a new one — rotation is the fix, not an allowlist.

## Quota-friendly / good-citizen patterns
- Prefer incremental date-windowed pulls over refetching full history.
- Cap concurrency (semaphore) and honor 429 backoff.
- Cache immutable history locally; only fetch the tail forward.
- Query the tickers/datasets you actually need — focused usage is cheaper and faster than sweeping the universe (which also trips abuse protection).

## Confirm exact params/schema
- OpenAPI: `https://twmarketdata.com/openapi.json` (or `/openapi.yaml`) — params, response schema, per-endpoint.
- Dataset docs markdown: `https://twmarketdata.com/en/datasets/<id>.md`
- Full dataset index: `https://twmarketdata.com/llms.txt`

## Honesty / boundaries
Not investment advice. TWSE = verified baseline; TPEx/adjusted prices beta/deferred. Daily + fundamentals focus; no real-time quotes, no intraday minute bars, no crypto. MCP is preview — shipped path is REST + X-API-Key. Never claim roadmap features (webhooks, weekly reconciliation) are live.