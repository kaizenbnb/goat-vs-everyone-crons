# goat-vs-everyone ingestion crons

Scheduled pings that refresh the public data behind
[goat-vs-everyone.vercel.app](https://goat-vs-everyone.vercel.app).

No application code or data lives here. This repo is timer-only infrastructure.
The Next.js app and its data layer live in the private application repository.
These workflows call authenticated ingestion endpoints on a schedule.

GitHub throttles high-frequency `schedule:` triggers (measured here: requested
hourly, delivered every ~4h). So the two freshness-critical workflows do not rely
on frequent short runs; they run as **long-lived, self-overlapping jobs**: each job
lives ~5h50 and ticks internally, and `cancel-in-progress` lets a newly granted run
relieve the previous one without ever opening a gap.

- **ingest-fast** - long-running job, ticks every 15 min: Binance spot + futures,
  Hyperliquid perps, CEX reserves and flows.
- **ingest-hourly** - long-running job, ticks every hour through: CoinGecko exchange
  volumes, BTC price, daily series, derivatives, BNB data, news, pulse, stables,
  exchanges and terminal data.
- **ingest-6h** - every 6 hours: asset leadership, Binance Research, recent
  listings and ESMA MiCA public registers.
- **smartmoney** - every 6 hours: Hyperliquid smart-money leaderboard snapshots.

Required repository secrets:

- `CRON_SECRET`
- `SITE_URL`
