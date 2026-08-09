---
name: twmd-excel-sheets
description: >-
  Pull TW Market Data (TWMD) into Excel (Power Query / M) and Google Sheets (Apps
  Script) — for analysts and non-coders. Use when the user wants Taiwan market
  data in a spreadsheet, asks "TWMD in Excel / Google Sheets," or wants a
  refreshable table without writing a program. Covers keyed requests via Power
  Query headers and Apps Script UrlFetchApp, plus the no-key demo via IMPORTDATA.
  Never put the API key in a cell/URL that gets shared.
---

# TWMD in Excel & Google Sheets

Get TWMD into a spreadsheet, refreshable, no real coding. **Not investment advice.** **Never share a sheet with your key in it.**

## No-key demo (works with plain IMPORTDATA — 5 symbols)
Google Sheets:
```
=IMPORTDATA("https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=30&format=csv")
```
(If the endpoint returns JSON only, use the Apps Script method below. IMPORTDATA can't send an API-key header, so keyed datasets need Power Query or Apps Script.)

## Excel — Power Query (sends the X-API-Key header)
Data → Get Data → From Other Sources → Blank Query → Advanced Editor:
```m
let
    key = "sk_live_...",            // better: store in a hidden config sheet / parameter
    url = "https://api.twmarketdata.com/v2/datasets/monthly-revenue?symbol=2330&limit=24",
    resp = Json.Document(Web.Contents(url, [Headers=[#"X-API-Key"=key]])),
    data = resp[data],
    tbl  = Table.FromList(data, Splitter.SplitByNothing()),
    cols = Table.ExpandRecordColumn(tbl, "Column1", Record.FieldNames(data{0}))
in
    cols
```
Refresh with Data → Refresh All. Keep the key in a Query Parameter, not shared.

## Google Sheets — Apps Script (sends the header, keyed)
Extensions → Apps Script:
```javascript
function TWMD(dataset, symbol, limit) {
  const key = PropertiesService.getScriptProperties().getProperty('TWMD_API_KEY'); // set once, not in a cell
  const url = `https://api.twmarketdata.com/v2/datasets/${dataset}?symbol=${symbol}&limit=${limit||30}`;
  const resp = UrlFetchApp.fetch(url, { headers: { 'X-API-Key': key } });
  const rows = JSON.parse(resp.getContentText()).data;
  if (!rows.length) return [['no data']];
  const cols = Object.keys(rows[0]);
  return [cols].concat(rows.map(r => cols.map(c => r[c])));
}
// In a cell:  =TWMD("twse-daily-price","2330",30)
```
Set the key once: Project Settings → Script Properties → `TWMD_API_KEY`. **Never** paste the key into a cell.

## Honesty
Not investment advice. Daily + fundamentals only. Store the key in Script Properties / a Query Parameter — never in a shared cell or URL. Confirm dataset ids at `https://twmarketdata.com/llms.txt`.
