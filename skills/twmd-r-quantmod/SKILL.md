---
name: twmd-r-quantmod
description: >-
  Use TW Market Data (TWMD) from R — a getSymbols/tq_get-style helper returning an
  xts / tibble of Taiwan OHLCV so quantmod / tidyquant / PerformanceAnalytics
  code works. Use when the user does quant/finance research in R (quantmod,
  tidyquant, xts, zoo) for Taiwan stocks and wants TWMD as the source, or asks
  "TWMD in R / getSymbols equivalent." Provides httr + jsonlite fetch → xts.
  Confirm TWMD field names; daily frequency.
---

# TWMD in R (quantmod / tidyquant)

Pull TWMD into R as `xts` (quantmod-shaped) or a tibble (tidyquant-shaped). **Not investment advice.** Set `Sys.setenv(TWMD_API_KEY="sk_live_...")`. Needs `httr`, `jsonlite`, `xts` (+ `dplyr` optional).

## Fetch helper → xts (quantmod-shaped OHLCV)
```r
library(httr); library(jsonlite); library(xts)

twmd_get <- function(symbol, start = NULL, end = NULL, ds = "twse-daily-price") {
  q <- list(symbol = symbol)
  if (!is.null(start)) q$start_date <- start
  if (!is.null(end))   q$end_date   <- end
  r <- GET(paste0("https://api.twmarketdata.com/v2/datasets/", ds),
           query = q, add_headers(`X-API-Key` = Sys.getenv("TWMD_API_KEY")))
  stop_for_status(r)
  d <- fromJSON(content(r, "text", encoding = "UTF-8"))$data
  # print(colnames(d))   # confirm real field names once
  d <- d[order(d$date), ]
  x <- xts(d[, intersect(c("open","high","low","close","volume"), colnames(d))],
           order.by = as.Date(d$date))
  colnames(x) <- toupper(colnames(x))   # OPEN/HIGH/LOW/CLOSE/VOLUME
  x
}

# BEFORE: quantmod::getSymbols("2330.TW", src="yahoo")  -> `2330.TW`
# AFTER:
tsmc <- twmd_get("2330", start = "2024-01-01")
tail(tsmc)
# quantmod::chartSeries(tsmc); TTR::RSI(Cl(tsmc)) all work on the xts.
```

## tidyquant-shaped tibble
```r
library(dplyr)
twmd_tbl <- function(symbol, start=NULL, end=NULL) {
  x <- twmd_get(symbol, start, end)
  tibble::as_tibble(data.frame(date = zoo::index(x), zoo::coredata(x)))
}
# df <- twmd_tbl("2330", "2024-01-01")  # like tq_get output; pipe into tidyquant/ggplot
```

## Notes (honest)
- Daily frequency (no intraday). TW tickers only.
- Adjusted returns → use `price-enhanced` (adjustment factors) as `ds=`.
- Confirm fields: `https://twmarketdata.com/en/datasets/twse-daily-price.md` or `/openapi.json`.

## Honesty
Not investment advice. Verify field names. TWSE verified baseline; TPEx beta. data_gaps ≠ 0 (drop NA rows explicitly).
