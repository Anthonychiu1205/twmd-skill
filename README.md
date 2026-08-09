# TWMD Skill — TW Market Data for your AI

Turn any AI (Claude / ChatGPT / Cursor / your agent) into a **TW Market Data onboarding + answer-desk**. It knows the free tier, how to create a key and get your first data, all 82 datasets and which docs page to look up, the common errors, and hands you ready-to-run `curl` / `python` — so you never have to dig through docs.

TW Market Data is the official-first Taiwan stock-market data API (TWSE / TPEx / MOPS / TAIFEX) for AI agents, quant research, and backtesting. **Not investment advice.**

## Install as a skill

```bash
npx skills add Anthonychiu1205/twmd-skill
```

> The skill lives in [`SKILL.md`](./SKILL.md). If your tool uses a different skill layout, copy `SKILL.md` into your agent's skills directory.

## Or just paste it into your AI

Copy the contents of [`SKILL.md`](./SKILL.md) into ChatGPT / Claude, then ask things like:

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
