---
name: twmd-powerbi-tableau
description: >-
  Connect BI tools — Power BI and Tableau — to TW Market Data (TWMD). Use when the
  user builds dashboards/reports and wants Taiwan market data from TWMD, or asks
  "TWMD in Power BI / Tableau." Power BI uses a Power Query (M) web source with the
  X-API-Key header; Tableau's most reliable path is to load TWMD into a warehouse
  (see twmd-warehouse-sql) and connect Tableau to that DB. Never embed the key in
  a shared report.
---

# TWMD in Power BI & Tableau

**Not investment advice.** **Never publish a report with the key embedded** — use a parameter / gateway credential.

## Power BI — Power Query (M) with header
Get Data → Blank Query → Advanced Editor:
```m
let
    key = ApiKey,                       // define ApiKey as a Parameter (Manage Parameters), not inline
    url = "https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&start_date=2024-01-01",
    resp = Json.Document(Web.Contents(url, [Headers=[#"X-API-Key"=key]])),
    data = resp[data],
    tbl  = Table.FromList(data, Splitter.SplitByNothing()),
    cols = Table.ExpandRecordColumn(tbl, "Column1", Record.FieldNames(data{0}))
in
    cols
```
- Store the key as a **Parameter** (or a Power BI Service credential on the gateway), not in the query text.
- For a whole universe: build a function `fnTWMD(symbol)` and invoke it over a symbol table, then expand.
- Set a scheduled refresh in Power BI Service (respect a sane cadence — daily after close).

## Tableau — recommended path (via warehouse)
Tableau's web-data-connector route is fragile/deprecated. The reliable pattern:
1. Load TWMD into a database (DuckDB/Postgres/BigQuery/Snowflake) with the **`twmd-warehouse-sql`** skill (scheduled).
2. Connect Tableau to that database (native connector) and build extracts/dashboards on the tables.
This gives Tableau fast, governed access and keeps the API key server-side.

Alternative for ad-hoc: Tableau → use a Python (TabPy) script or pre-exported CSV/Parquet from TWMD.

## Honesty
Not investment advice. Daily + fundamentals. Keep the key in a parameter/gateway credential, never embedded in a shared workbook/report. Confirm dataset ids at `/llms.txt`, fields at `<id>.md`.
