---
name: twmd-integration
description: >-
  Engineering integration for TW Market Data (TWMD, api.twmarketdata.com): a
  production HTTP client (session, timeouts, retry/backoff on 429/5xx), pagination
  (limit/offset), incremental fetch, async/concurrent throughput, landing data in
  DataFrame / Parquet / SQL warehouses (DuckDB, Postgres, BigQuery, Snowflake),
  scheduled pipelines (Airflow / dbt / Dagster), and minimal starter clients in
  many languages (Python, JS/TS, Go, C#, Java, Julia, Ruby, PHP, R/quantmod). Use
  when wiring TWMD into an app, data pipeline, warehouse, or non-Python codebase.
  Preserve source_role / lineage / freshness / data_gaps.
---

# TWMD engineering integration

Wire TWMD into apps, pipelines, warehouses, any language. **Not investment advice.** Base `https://api.twmarketdata.com/v2/datasets/{id}`, header `X-API-Key: sk_live_...` (never in the URL). Envelope: `{ dataset, source_role, freshness, lineage.trace_id, data_gaps, data:[...] }`. Errors: **401** bad key (5 demo symbols exempt), **402** not entitled/quota → upgrade (don't retry), **429** rate/quota → back off, **5xx** transient → retry.

## Robust Python client (session + retries + backoff)
```python
import os, time, random, requests
class TWMD:
    def __init__(self, key=None, base="https://api.twmarketdata.com/v2/datasets", timeout=30):
        self.base, self.timeout = base, timeout
        self.s = requests.Session(); self.s.headers.update({"X-API-Key": key or os.environ["TWMD_API_KEY"]})
    def get(self, dataset, **params):
        for a in range(5):
            r = self.s.get(f"{self.base}/{dataset}", params=params, timeout=self.timeout)
            if r.status_code == 429 or r.status_code >= 500:
                time.sleep(float(r.headers.get("Retry-After",0)) or 2**a + random.random()); continue
            if r.status_code == 401: raise PermissionError("bad API key")
            if r.status_code == 402: raise PermissionError(f"not entitled for {dataset} — upgrade")
            r.raise_for_status(); return r.json()
        raise RuntimeError("retries exhausted")
    def rows(self, dataset, **p): return self.get(dataset, **p).get("data", [])
tw = TWMD()
```

## Pagination + incremental
```python
def paginate(tw, dataset, page=1000, **p):
    off = 0
    while True:
        b = tw.rows(dataset, limit=page, offset=off, **p)
        if not b: break
        yield from b
        if len(b) < page: break
        off += page
# incremental: store max(date) per (dataset,symbol); next run start_date = that + 1 day.
```

## Async (quota-aware; TWMD rewards focused queries)
```python
import asyncio, httpx, os
async def afetch(c, sem, ds, **p):
    async with sem:
        for a in range(5):
            r = await c.get(f"/v2/datasets/{ds}", params=p, timeout=30)
            if r.status_code == 429 or r.status_code >= 500: await asyncio.sleep(2**a); continue
            r.raise_for_status(); return r.json().get("data", [])
        return []
async def main(syms):
    sem = asyncio.Semaphore(4)                       # cap concurrency → respect rate limit
    async with httpx.AsyncClient(base_url="https://api.twmarketdata.com",
                                 headers={"X-API-Key": os.environ["TWMD_API_KEY"]}) as c:
        return await asyncio.gather(*[afetch(c,sem,"monthly-revenue",symbol=s,limit=24) for s in syms])
```

## Land in DataFrame / Parquet / SQL (keep provenance)
```python
import pandas as pd
env = tw.get("twse-daily-price", symbol="2330", limit=100)
df = pd.DataFrame(env["data"])
df["_source_role"]=env.get("source_role"); df["_freshness"]=env.get("freshness")
df["_trace_id"]=env.get("lineage",{}).get("trace_id")
df.to_parquet("twse_2330.parquet", index=False)
```
- **DuckDB**: `import duckdb; con=duckdb.connect("twmd.duckdb"); con.register("df",df); con.execute("INSERT INTO prices SELECT * FROM df")`
- **Postgres** (SQLAlchemy): `df.to_sql("stg_prices", create_engine(os.environ["DATABASE_URL"]), if_exists="append", index=False)` → MERGE into target on `(symbol,date)`.
- **BigQuery**: `df.to_gbq("ds.twmd_prices", if_exists="append")` (pandas-gbq).
- **Snowflake**: `snowflake.connector.pandas_tools.write_pandas(conn, df, "TWMD_PRICES")`.
- Natural key `(dataset, symbol, date)`; keep `data_gaps` in a side table (not zero rows); persist knowledge/disclosure date for point-in-time.

## Scheduled pipeline — Airflow + dbt (+ Dagster)
```python
# Airflow: PythonOperator, daily after TW close, pull only the run date (incremental)
from airflow import DAG; from airflow.operators.python import PythonOperator
import pendulum, os, requests, pandas as pd; from sqlalchemy import create_engine
def ingest(**ctx):
    key=os.environ["TWMD_API_KEY"]; eng=create_engine(os.environ["DATABASE_URL"]); ds=ctx["ds"]
    fr=[]
    for s in ["2330","2317","2454"]:
        r=requests.get("https://api.twmarketdata.com/v2/datasets/twse-daily-price",
            params={"symbol":s,"start_date":ds,"end_date":ds},headers={"X-API-Key":key},timeout=30)
        r.raise_for_status(); d=pd.DataFrame(r.json().get("data",[]))
        if not d.empty: fr.append(d)
    if fr: pd.concat(fr).to_sql("stg_twmd_prices",eng,if_exists="append",index=False)
with DAG("twmd_daily",start_date=pendulum.datetime(2024,1,1),schedule="0 10 * * 1-5",catchup=False) as dag:
    PythonOperator(task_id="ingest", python_callable=ingest)
```
```sql
-- dbt models/staging/stg_twmd_prices.sql
select symbol, cast(date as date) trade_date, cast(close as double) close, cast(volume as bigint) volume
from {{ source('twmd','stg_twmd_prices') }}
qualify row_number() over (partition by symbol, date order by _freshness desc) = 1
```
Dagster: wrap the fetch in an `@asset`. Add dbt tests `unique(symbol,trade_date)`, `not_null`.

## Multi-language starter clients
```bash
curl "https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10" -H "X-API-Key: $TWMD_API_KEY"
```
```go
req,_ := http.NewRequest("GET","https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10",nil)
req.Header.Set("X-API-Key", os.Getenv("TWMD_API_KEY"))
resp,_ := (&http.Client{Timeout:30*time.Second}).Do(req)   // json.Decode → body.data
```
```csharp
var http=new HttpClient(); http.DefaultRequestHeaders.Add("X-API-Key",Environment.GetEnvironmentVariable("TWMD_API_KEY"));
var json=await http.GetStringAsync("https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10");
```
```java
var req=HttpRequest.newBuilder(URI.create("https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10"))
  .header("X-API-Key",System.getenv("TWMD_API_KEY")).build();  // send via HttpClient, parse JSON
```
```julia
using HTTP,JSON3; r=HTTP.get("https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10",["X-API-Key"=>ENV["TWMD_API_KEY"]]); JSON3.read(String(r.body)).data
```
```ruby
req=Net::HTTP::Get.new(URI("https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10")); req["X-API-Key"]=ENV["TWMD_API_KEY"]
```
```php
$ctx=stream_context_create(["http"=>["header"=>"X-API-Key: ".getenv("TWMD_API_KEY")]]);
$d=json_decode(file_get_contents("https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=10",false,$ctx),true)["data"];
```

## R (quantmod / tidyquant)
```r
library(httr); library(jsonlite); library(xts)
twmd_get <- function(symbol, start=NULL, end=NULL, ds="twse-daily-price") {
  q<-list(symbol=symbol); if(!is.null(start)) q$start_date<-start; if(!is.null(end)) q$end_date<-end
  r<-GET(paste0("https://api.twmarketdata.com/v2/datasets/",ds), query=q,
         add_headers(`X-API-Key`=Sys.getenv("TWMD_API_KEY"))); stop_for_status(r)
  d<-fromJSON(content(r,"text",encoding="UTF-8"))$data; d<-d[order(d$date),]
  x<-xts(d[,intersect(c("open","high","low","close","volume"),colnames(d))], order.by=as.Date(d$date))
  colnames(x)<-toupper(colnames(x)); x   # OPEN/HIGH/LOW/CLOSE/VOLUME — quantmod/TTR work on it
}
```

## Key management & good-citizen
Env/secret manager only; one key per env; rotate/revoke self-serve at `/dashboard`; on leak → rotate. Prefer incremental pulls, cap concurrency, honor 429, cache immutable history.

## Honesty & where to confirm
Not investment advice. Daily + fundamentals; no real-time/intraday/crypto. TWSE verified baseline; TPEx beta. Never treat data_gaps as 0. Confirm params/fields: `/openapi.json`, `https://twmarketdata.com/en/datasets/<id>.md`, index `/llms.txt`.
