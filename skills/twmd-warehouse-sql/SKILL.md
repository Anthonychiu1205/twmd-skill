---
name: twmd-warehouse-sql
description: >-
  Load TW Market Data (TWMD) into a database / warehouse — DuckDB, Postgres,
  BigQuery, Snowflake — and keep it fresh incrementally. Use when the user wants
  TWMD in SQL / a data warehouse / a lake, asks "load TWMD into Postgres/DuckDB/
  BigQuery/Snowflake," or is building an analytics data layer. Covers a
  fetch→DataFrame→table loader, provenance columns, an upsert/incremental
  pattern, and Parquet lake. Preserve source_role/freshness/data_gaps.
---

# TWMD → SQL warehouse / lake

Land TWMD in a database and keep it fresh. **Not investment advice.** `export TWMD_API_KEY=sk_live_...`. Needs `pandas` (+ your DB driver).

## Fetch → DataFrame with provenance
```python
import os, requests, pandas as pd
_S = requests.Session(); _S.headers.update({"X-API-Key": os.environ["TWMD_API_KEY"]})

def load_df(dataset, **params):
    r = _S.get(f"https://api.twmarketdata.com/v2/datasets/{dataset}", params=params, timeout=30)
    r.raise_for_status()
    env = r.json()
    df = pd.DataFrame(env.get("data", []))
    df["_dataset"] = dataset
    df["_source_role"] = env.get("source_role")
    df["_freshness"]   = env.get("freshness")
    df["_trace_id"]    = env.get("lineage", {}).get("trace_id")
    return df

df = load_df("twse-daily-price", symbol="2330", start_date="2024-01-01")
```

## DuckDB (zero-setup local warehouse)
```python
import duckdb
con = duckdb.connect("twmd.duckdb")
con.execute("CREATE TABLE IF NOT EXISTS prices AS SELECT * FROM df WHERE 1=0")
con.register("df", df)
con.execute("INSERT INTO prices SELECT * FROM df")
con.execute("SELECT symbol, count(*) FROM prices GROUP BY 1").df()
```

## Postgres (SQLAlchemy) with upsert-friendly staging
```python
from sqlalchemy import create_engine
eng = create_engine(os.environ["DATABASE_URL"])
df.to_sql("twmd_prices_stage", eng, if_exists="replace", index=False)
# then MERGE/UPSERT into the target on (symbol, date):
# INSERT INTO twmd_prices SELECT * FROM twmd_prices_stage
#   ON CONFLICT (symbol, date) DO UPDATE SET close=EXCLUDED.close, ...;
```

## Incremental refresh (only pull the tail)
```python
def refresh(con_query_max_date, dataset, symbol):
    last = con_query_max_date(dataset, symbol)          # e.g. "2024-12-31" from your DB
    import datetime as dt
    start = (dt.date.fromisoformat(last) + dt.timedelta(days=1)).isoformat()
    return load_df(dataset, symbol=symbol, start_date=start)
# schedule daily after close; append only new rows. Immutable history need not be refetched.
```

## BigQuery / Snowflake
- **BigQuery**: `df.to_gbq("dataset.twmd_prices", if_exists="append")` (pandas-gbq), or write Parquet to GCS + external table.
- **Snowflake**: `snowflake.connector.pandas_tools.write_pandas(conn, df, "TWMD_PRICES")`.

## Parquet data lake
```python
df.to_parquet(f"lake/{'twse-daily-price'}/symbol=2330/part.parquet", index=False)
# partition by dataset/symbol/date; query with DuckDB/Spark/Athena over the lake.
```

## Modeling tips
- Natural key: `(dataset, symbol, date)` (or `(symbol, fiscal_period)` for fundamentals).
- Keep `_source_role/_freshness/_trace_id` for audit; store `data_gaps` in a side table, **not** as zero rows.
- For point-in-time correctness, also persist each row's knowledge/disclosure date.

## Honesty
Not investment advice. Daily + fundamentals. Confirm dataset ids/fields at `https://twmarketdata.com/llms.txt` and `<id>.md`. data_gaps ≠ 0.
