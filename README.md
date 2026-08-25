# TWMD Skill — TW Market Data for your AI

Turn any AI (Claude / ChatGPT / Cursor / your agent) into a **TW Market Data onboarding + answer-desk**, and drop TWMD into whatever you already use — data vendors, backtesters, platforms, warehouses, BI, agents.

TW Market Data is the official-first Taiwan stock-market data API (TWSE / TPEx / MOPS / TAIFEX) for AI agents, quant research, and backtesting. **Not investment advice.**

## Install

```bash
npx skills add https://github.com/TW-Market-Data/twmd-skill
```

Install one by its install name:

```bash
npx skills add https://github.com/TW-Market-Data/twmd-skill --skill "twmd-migrations"
```

## Skills (6)

| Skill | Install name | For | What it gives |
| --- | --- | --- | --- |
| Onboarding / answer-desk | `twmd-taiwan-market-data` | Everyone / first time | Free tier, key → first data, full dataset index → docs map, errors, command playbook |
| Quant recipes | `twmd-quant-recipes` | Quants / researchers | Factor code: monthly-revenue YoY, 三大法人 flow, valuation screen, multi-factor rank, point-in-time-safe backtest |
| Integration | `twmd-integration` | Engineers | Production client (retries/backoff/pagination/async), warehouses (DuckDB/Postgres/BigQuery/Snowflake), Airflow/dbt/Dagster, clients in Go/C#/Java/Julia/Ruby/PHP/R |
| Migrations | `twmd-migrations` | Switching vendors | Drop-in shims for FinMind, yfinance, twstock, TEJ, broker APIs (Shioaji/Fugle), pandas-datareader / Alpha Vantage / Tiingo / Quandl / EODHD |
| Backtesting & platforms | `twmd-backtesting-platforms` | Backtesters / traders | backtrader / vectorbt / zipline, QuantConnect/LEAN, MT5, MT4 — with survivorship & point-in-time cautions |
| BI & agents | `twmd-bi-and-agents` | Analysts / agent builders | Excel / Google Sheets / Power BI / Tableau, and LangChain / LlamaIndex / function-calling / OpenBB (MCP is LIVE at mcp.twmarketdata.com) |

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

## Or skip the HTTP entirely

**Python + CLI**

```bash
pip install twmarketdata

twmd datasets --free-only
twmd get monthly_revenue --ticker 2330 --as-of 2024-06-30 --format csv
```

Omit `--as-of` and you get the latest revision — including values revised after the date you are
reasoning about. The CLI says so on stderr, because for a backtest that is a look-ahead leak and
the response otherwise looks completely normal.

**Agents — MCP**

`https://mcp.twmarketdata.com/mcp` — 34 tools. The catalogue, glossary, point-in-time methodology,
standards mapping, benchmark method and coverage windows all read with **no key at all**; the five
sample symbols above answer with no plan. Querying beyond those starts at the Pro plan, and
upgrading uses the same email you signed in with — no reconnect.

Full tool list and connection steps:
[TW-Market-Data/tw-market-data-mcp](https://github.com/TW-Market-Data/tw-market-data-mcp).

## Links

- Site: https://twmarketdata.com · Pricing: https://twmarketdata.com/en/pricing · Dashboard: https://twmarketdata.com/dashboard
- Machine files: https://twmarketdata.com/llms.txt · https://twmarketdata.com/llms-full.txt · https://twmarketdata.com/openapi.json
- Python SDK + CLI: [TW-Market-Data/twmarketdata](https://github.com/TW-Market-Data/twmarketdata) · [PyPI](https://pypi.org/project/twmarketdata/)
- MCP server: [TW-Market-Data/tw-market-data-mcp](https://github.com/TW-Market-Data/tw-market-data-mcp)

## License

MIT — copy, adapt, share.
