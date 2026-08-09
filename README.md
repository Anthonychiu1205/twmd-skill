# TWMD Skill — TW Market Data for your AI

Turn any AI (Claude / ChatGPT / Cursor / your agent) into a **TW Market Data onboarding + answer-desk**, and drop TWMD into whatever you already use — data vendors, backtesters, platforms, warehouses, BI, agents.

TW Market Data is the official-first Taiwan stock-market data API (TWSE / TPEx / MOPS / TAIFEX) for AI agents, quant research, and backtesting. **Not investment advice.**

## Install

```bash
npx skills add https://github.com/Anthonychiu1205/twmd-skill
```

Install one by its install name:

```bash
npx skills add https://github.com/Anthonychiu1205/twmd-skill --skill "twmd-migrations"
```

## Skills (6)

| Skill | Install name | For | What it gives |
| --- | --- | --- | --- |
| Onboarding / answer-desk | `twmd-taiwan-market-data` | Everyone / first time | Free tier, key → first data, 82-dataset → docs map, errors, command playbook |
| Quant recipes | `twmd-quant-recipes` | Quants / researchers | Factor code: monthly-revenue YoY, 三大法人 flow, valuation screen, multi-factor rank, point-in-time-safe backtest |
| Integration | `twmd-integration` | Engineers | Production client (retries/backoff/pagination/async), warehouses (DuckDB/Postgres/BigQuery/Snowflake), Airflow/dbt/Dagster, clients in Go/C#/Java/Julia/Ruby/PHP/R |
| Migrations | `twmd-migrations` | Switching vendors | Drop-in shims for FinMind, yfinance, twstock, TEJ, broker APIs (Shioaji/Fugle), pandas-datareader / Alpha Vantage / Tiingo / Quandl / EODHD |
| Backtesting & platforms | `twmd-backtesting-platforms` | Backtesters / traders | backtrader / vectorbt / zipline, QuantConnect/LEAN, MT5, MT4 — with survivorship & point-in-time cautions |
| BI & agents | `twmd-bi-and-agents` | Analysts / agent builders | Excel / Google Sheets / Power BI / Tableau, and LangChain / LlamaIndex / function-calling / OpenBB (MCP is preview) |

`npx skills add` scans the `skills/` folder; each skill's rich `description` still triggers on its specific tools (FinMind, MT5, backtrader, LangChain, …), so one clean set of 6 covers the whole ecosystem.

## Or just paste it into your AI

Copy any skill's `SKILL.md` (under [`skills/`](./skills)) into ChatGPT / Claude and ask, e.g.:

- "I've never used TWMD — walk me through getting my first Taiwan stock data."
- "Port my FinMind pipeline to TWMD."
- "Give me a backtrader feed for Taiwan stocks from TWMD."
- "Show me TSMC's monthly revenue YoY."

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
