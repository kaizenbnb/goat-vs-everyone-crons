# goat-vs-everyone ingestion crons

Scheduled pings that refresh the public data behind
[goat-vs-everyone.vercel.app](https://goat-vs-everyone.vercel.app).

No application code or data lives here. This repo is timer-only infrastructure.
The Next.js app and its data layer live in the private application repository.
These workflows call authenticated ingestion endpoints on a schedule:

- **ingest-fast** - hourly GitHub run with internal ticks: Binance spot + futures,
  Hyperliquid perps, CEX reserves and flows.
- **ingest-hourly** - minute :07 of each hour: CoinGecko exchange volumes,
  BTC price, daily series, derivatives, BNB data, news, pulse, stables,
  exchanges and terminal data.
- **ingest-6h** - every 6 hours: asset leadership, Binance Research, recent
  listings and ESMA MiCA public registers.
- **smartmoney** - every 6 hours: Hyperliquid smart-money leaderboard snapshots.

Required repository secrets:

- `CRON_SECRET`
- `SITE_URL`
