# TWMD Skill — TW Market Data for your AI

Turn any AI (Claude / ChatGPT / Cursor / your agent) into a **TW Market Data onboarding + answer-desk**. It knows the free tier, how to create a key and get your first data, all 82 datasets and which docs page to look up, the common errors, and hands you ready-to-run `curl` / `python` — so you never have to dig through docs.

TW Market Data is the official-first Taiwan stock-market data API (TWSE / TPEx / MOPS / TAIFEX) for AI agents, quant research, and backtesting. **Not investment advice.**

## Install as a skill

The [`npx skills add`](https://github.com/vercel-labs/agent-skills) CLI scans the `skills/` folder in this repo:

```bash
npx skills add https://github.com/Anthonychiu1205/twmd-skill
```

Or install just this skill by its install name (the `name:` in the SKILL frontmatter):

```bash
npx skills add https://github.com/Anthonychiu1205/twmd-skill --skill "twmd-taiwan-market-data"
```

> The skill lives in [`skills/twmd-taiwan-market-data/SKILL.md`](./skills/twmd-taiwan-market-data/SKILL.md). If your tool uses a different layout, just copy that `SKILL.md` into your agent's skills directory.

## Skills in this repo

`npx skills add` scans the `skills/` folder. Install all, or one by its **install name** (`--skill "<name>"`).

| Skill (folder) | Install name | For | What it gives |
| --- | --- | --- | --- |
| twmd-taiwan-market-data | `twmd-taiwan-market-data` | Everyone / first time | Onboarding + full answer-desk: free tier, key → first data, 82-dataset → docs map, errors, command playbook |
| twmd-quant-recipes | `twmd-quant-recipes` | Quants / researchers | Ready-to-run factor code: monthly-revenue YoY, 三大法人 flow momentum, valuation screen, multi-factor rank, point-in-time-safe backtest skeleton |
| twmd-api-integration | `twmd-api-integration` | Engineers | Production client: retries/backoff on 429/5xx, pagination, incremental fetch, async, DataFrame/Parquet/SQL, Node/TS, key management |
| twmd-migrate-from-finmind | `twmd-migrate-from-finmind` | FinMind users | FinMind→TWMD dataset map + drop-in shim (keep your FinMind call shape) |
| twmd-migrate-from-yfinance | `twmd-migrate-from-yfinance` | yfinance users | `Ticker("2330.TW").history()`-shaped shim → swap one import |
| twmd-backtrader-feed | `twmd-backtrader-feed` | Backtesters | backtrader PandasData feed (+ vectorbt/zipline notes), survivorship & PIT cautions |
| twmd-mt5-bridge | `twmd-mt5-bridge` | MT5 users | TWMD daily bars → MT5 custom symbol (Python export + MQL5 import), honest scope |
| twmd-migrate-from-tej | `twmd-migrate-from-tej` | TEJ users | TEJ table → TWMD map + tejapi.get-shaped shim |
| twmd-migrate-from-twstock | `twmd-migrate-from-twstock` | twstock users | `Stock('2330').price`-shaped shim |
| twmd-migrate-from-broker-apis | `twmd-migrate-from-broker-apis` | Shioaji / Fugle 富果 users | Replace historical fetch (keep broker for live/orders) |
| twmd-r-quantmod | `twmd-r-quantmod` | R users | getSymbols/tq_get-style helper → xts / tibble |
| twmd-excel-sheets | `twmd-excel-sheets` | Analysts / no-code | Excel Power Query + Google Sheets Apps Script |
| twmd-warehouse-sql | `twmd-warehouse-sql` | Data engineers | Load to DuckDB / Postgres / BigQuery / Snowflake + incremental |
| twmd-langchain-tool | `twmd-langchain-tool` | Agent builders | LangChain / LlamaIndex / function-calling tool (MCP is preview) |
| twmd-openbb-provider | `twmd-openbb-provider` | OpenBB users | Quick Python path + OpenBB Platform provider-extension fetcher |
| twmd-airflow-dbt | `twmd-airflow-dbt` | Data platforms | Airflow DAG + dbt sources/staging + Dagster asset, incremental |
| twmd-quantconnect-lean | `twmd-quantconnect-lean` | QuantConnect/LEAN | PythonData custom data class + algorithm usage |
| twmd-powerbi-tableau | `twmd-powerbi-tableau` | BI / dashboards | Power BI Power Query (M) + Tableau via warehouse |
| twmd-migrate-data-vendors | `twmd-migrate-data-vendors` | Global-vendor users | Swap pandas-datareader / Alpha Vantage / Tiingo / Quandl / EODHD |
| twmd-mt4-bridge | `twmd-mt4-bridge` | MT4 users | Legacy CSV/offline-chart path (prefer MT5) |
| twmd-clients-multilang | `twmd-clients-multilang` | Go/C#/Java/Julia/Ruby/PHP | Minimal starter clients per language |

### Coverage

Covers the common Taiwan-quant + developer ecosystem end to end: onboarding & answer-desk, quant factor recipes, production engineering client; **data-source migrations** from FinMind, yfinance, twstock, TEJ, broker APIs (Shioaji/Fugle), and global vendors (pandas-datareader / Alpha Vantage / Tiingo / Quandl / EODHD); **backtesting** (backtrader / vectorbt / zipline, QuantConnect/LEAN); **platforms** (MT5, MT4, Excel / Google Sheets, Power BI / Tableau); **data engineering** (SQL warehouses, Airflow / dbt / Dagster); **languages** (Python, JS/TS, R, Go, C#, Java, Julia, Ruby, PHP); and **AI agents** (LangChain / LlamaIndex / function-calling; MCP preview). Want another tool wired in? Open an issue.

```bash
npx skills add https://github.com/Anthonychiu1205/twmd-skill --skill "twmd-quant-recipes"
```

## Or just paste it into your AI

Copy any skill's `SKILL.md` (under [`skills/`](./skills)) into ChatGPT / Claude, then ask things like:

- "I've never used TWMD — walk me through getting my first Taiwan stock data."
- "Show me TSMC's monthly revenue YoY."
- "What can I use for free?"
- "I got a 402 — what does that mean?"

## Try it with zero setup (no key)

```bash
curl "https://api.twmarketdata.com/v2/datasets/twse-daily-price?symbol=2330&limit=1"
```

5 symbols work with no key: 2330, 2317, 2454, 0050, 2603.

## Links

- Site: https://twmarketdata.com · Pricing: https://twmarketdata.com/pricing · Dashboard: https://twmarketdata.com/dashboard
- Machine files: https://twmarketdata.com/llms.txt · https://twmarketdata.com/llms-full.txt · https://twmarketdata.com/openapi.json

## License

MIT — copy, adapt, share.
