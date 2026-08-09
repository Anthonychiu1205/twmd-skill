---
name: twmd-airflow-dbt
description: >-
  Build a scheduled data pipeline that ingests TW Market Data (TWMD) into a
  warehouse and models it — Apache Airflow (or Dagster/Prefect) for orchestration
  plus dbt for transforms. Use when the user runs a data platform and wants TWMD
  on a daily schedule into Postgres/BigQuery/Snowflake with dbt staging models, or
  asks "TWMD Airflow DAG / dbt source / Dagster asset / daily ingestion." Provides
  a DAG, dbt sources + staging model, and a Dagster asset. Preserve provenance.
---

# TWMD → Airflow / dbt pipeline

Scheduled ingest + transform. **Not investment advice.** Set `TWMD_API_KEY` in your orchestrator's secrets (never in code). Pairs with the `twmd-warehouse-sql` skill for the loader.

## Airflow DAG (daily, incremental)
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
import pendulum, os, requests, pandas as pd
from sqlalchemy import create_engine

SYMS = ["2330","2317","2454"]  # or read your universe table

def ingest(**ctx):
    key = os.environ["TWMD_API_KEY"]; eng = create_engine(os.environ["DATABASE_URL"])
    ds = ctx["ds"]  # execution date YYYY-MM-DD
    frames = []
    for s in SYMS:
        r = requests.get("https://api.twmarketdata.com/v2/datasets/twse-daily-price",
                         params={"symbol": s, "start_date": ds, "end_date": ds},
                         headers={"X-API-Key": key}, timeout=30)
        r.raise_for_status(); env = r.json()
        df = pd.DataFrame(env.get("data", []))
        if not df.empty:
            df["_source_role"] = env.get("source_role"); df["_freshness"] = env.get("freshness")
            frames.append(df)
    if frames:
        pd.concat(frames).to_sql("stg_twmd_prices", eng, if_exists="append", index=False)

with DAG("twmd_daily", start_date=pendulum.datetime(2024,1,1),
         schedule="0 10 * * 1-5", catchup=False) as dag:   # after TW close, weekdays
    PythonOperator(task_id="ingest_prices", python_callable=ingest)
```

## dbt — source + staging model
```yaml
# models/sources.yml
sources:
  - name: twmd
    tables:
      - name: stg_twmd_prices
```
```sql
-- models/staging/stg_twmd_prices.sql
select
  symbol,
  cast(date as date)        as trade_date,
  cast(close as double)     as close,
  cast(volume as bigint)    as volume,
  _source_role, _freshness
from {{ source('twmd','stg_twmd_prices') }}
qualify row_number() over (partition by symbol, date order by _freshness desc) = 1  -- dedupe
```
Add tests (`unique` on `(symbol, trade_date)`, `not_null`) and downstream marts (factors, returns).

## Dagster (asset form)
```python
from dagster import asset
import os, requests, pandas as pd

@asset
def twmd_prices():
    key = os.environ["TWMD_API_KEY"]
    r = requests.get("https://api.twmarketdata.com/v2/datasets/twse-daily-price",
                     params={"symbol":"2330","limit":30}, headers={"X-API-Key":key}, timeout=30)
    r.raise_for_status(); return pd.DataFrame(r.json().get("data", []))
```

## Good-citizen scheduling
- Run once after TW market close; pull only the new date (incremental), not full history.
- Respect 429 with backoff (see `twmd-api-integration`); cap concurrency across symbols.
- Keep `data_gaps` in a side table; don't zero-fill. Persist knowledge/disclosure date for PIT.

## Honesty
Not investment advice. Daily + fundamentals. Confirm dataset ids/fields at `/llms.txt` and `<id>.md`. Secrets in the orchestrator, never in the DAG/repo.
