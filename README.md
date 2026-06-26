# goat-vs-everyone ingestion crons

Scheduled GitHub Actions that keep the public data behind [goat-vs-everyone.vercel.app](https://goat-vs-everyone.vercel.app) fresh.

This repository is timer infrastructure only. There is no application code and no data here. The Next.js app and its data layer live in a separate private repository. What lives here are the workflows that call authenticated ingestion endpoints on a schedule, so the live index always has recent market data to show.

## What this repo is, and what it is not

- It is: a set of scheduled jobs that trigger data ingestion on a fixed cadence.
- It is not: the app, the database, or the business logic. Those are private.

The separation is deliberate. Secrets and product code stay in the private repo, while the public side is just the timing layer. Anyone can read exactly how the scheduling works without getting access to anything sensitive.

## The problem: GitHub throttles frequent cron

GitHub Actions `schedule:` triggers are not guaranteed to fire on time. Under load they are delayed, and high frequency schedules suffer the most. In practice, a workflow asked to run every hour was being delivered only every four hours or so. For a live market index, that is too stale.

A naive fix would be to schedule many short runs and hope enough of them fire. That wastes minutes and still leaves gaps.

## The solution: long-lived, self-overlapping jobs

The two freshness critical workflows do not rely on frequent short runs. Each one runs as a single long-lived job that lives for about five hours and fifty minutes and ticks internally on its own interval. Before one job reaches its time limit, the next scheduled run starts, and `cancel-in-progress` lets the new run relieve the old one. The handover happens without ever opening a gap in coverage.

```text
run A  |==============tick.tick.tick==============|
run B                                     |==============tick.tick.tick==============|
                                          ^ B starts and relieves A before A expires
                                          no gap in data freshness
```

This keeps the data flowing at the real cadence the index needs, instead of the cadence GitHub feels like delivering.

## The workflows

| Workflow | Cadence | Data it refreshes |
| --- | --- | --- |
| `ingest-fast` | long job, ticks every 15 min | Binance spot and futures, Hyperliquid perps, CEX reserves and flows |
| `ingest-hourly` | long job, ticks every hour | CoinGecko exchange volumes, BTC price, daily series, derivatives, BNB data, news, pulse, stables, exchanges and terminal data |
| `ingest-6h` | every 6 hours | asset leadership, Binance Research, recent listings and ESMA MiCA public registers |
| `smartmoney` | every 6 hours | Hyperliquid smart-money leaderboard snapshots |

`ingest-fast` and `ingest-hourly` are the two that use the long-lived, self-overlapping pattern. `ingest-6h` and `smartmoney` run on a plain schedule because a few hours of delay does not hurt that kind of data.

## How it fits together

```text
GitHub Actions (this repo)
        |
        |  scheduled trigger
        v
Authenticated ingestion endpoint   <-- protected by CRON_SECRET
        |
        v
Private app and data layer (Next.js)
        |
        v
Public index at goat-vs-everyone.vercel.app
```

The workflows never touch the data directly. They call an endpoint that does the work, and they prove they are allowed to by sending a shared secret.

## Configuration

These workflows expect two repository secrets:

| Secret | Purpose |
| --- | --- |
| `CRON_SECRET` | Shared token sent with each request so the ingestion endpoint can reject anything that is not these jobs. |
| `SITE_URL` | Base URL of the deployment the jobs should hit. |

No values are committed. Set them under Settings, Secrets and variables, Actions.

## Design notes

- Long jobs over frequent short jobs, because GitHub delivers them more reliably and the internal tick gives precise timing.
- `cancel-in-progress` so a fresh run cleanly takes over from the previous one without two jobs fighting over the same window.
- Public timing layer, private product, so the schedule is transparent while secrets and code stay protected.

## Scope and limitations

This repo only schedules and triggers ingestion. It does not store, transform or serve data. It also depends on GitHub runner availability, so the long-lived pattern is a mitigation for cron throttling, not a hard real time guarantee.

## Author

Built by Oliver Arjonilla as part of the data engineering behind a live crypto market index.
