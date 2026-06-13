# goat-vs-everyone — ingestion crons

Scheduled pings that refresh the public data behind
[goat-vs-everyone.vercel.app](https://goat-vs-everyone.vercel.app).

No application code or data lives here — just timers. The Next.js app and its
data layer are in a separate private repository. These workflows call the app's
authenticated ingestion endpoints on a schedule:

- **ingest-fast** — every 5 min: Binance spot + futures, Hyperliquid perps.
- **ingest-hourly** — minute :07 of each hour: CoinGecko (exchange volumes,
  BTC price, daily series, derivatives).

Both require two repository secrets: `CRON_SECRET` and `SITE_URL`.
