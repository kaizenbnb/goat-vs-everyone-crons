# goat-vs-everyone ingestion crons

This repository used to be the timer infrastructure for [goat-vs-everyone.vercel.app](https://goat-vs-everyone.vercel.app): scheduled GitHub Actions workflows calling authenticated ingestion endpoints on a schedule.

## 2026-11-05: moved off GitHub Actions `schedule:`

On 2026-08-25 GitHub silently disabled all 4 scheduled workflows in this repo because the repo had gone 60 days without a push (last push: 2026-06-26). That is a hard GitHub Actions policy for public and private repos alike: a `schedule:` trigger is auto-disabled after 60 days of repository inactivity, with no notification. The result was zero ingestion for over two months — the live index kept serving August data without anyone noticing.

This was the second reliability problem with GitHub Actions as the trigger for this project, not the first: `schedule:` is also throttled under load (a workflow asked to run every 30 min was observed firing roughly every 4h), which is why `ingest-fast` and `ingest-hourly` used to run as long-lived, self-overlapping jobs that ticked internally — a workaround for the throttling, not for the 60-day disable.

Both problems go away by not using `schedule:` at all:

- `ingest-fast`, `ingest-hourly`, `ingest-6h` are **retired**. Their job is now done by three aggregator routes in the Next.js app itself (`/api/cron/fast`, `/api/cron/hourly`, `/api/cron/sixhourly`), each fanning out in parallel to the relevant `/api/ingest/*` endpoints. An external pinger (cron-job.org, free tier) calls each one directly at its real cadence (15 min / hourly / 6h) with the `CRON_SECRET` bearer header. No GitHub repo, no 60-day clock, no schedule throttling.
- `smartmoney` **stays here**, because its leaderboard fetch (~32MB from Hyperliquid) and per-wallet position lookups are too heavy to run as a Vercel function. It now triggers on `workflow_dispatch` only (no `schedule:`), called every 6h by the same external pinger hitting the GitHub Actions API (`POST /repos/kaizenbnb/goat-vs-everyone-crons/actions/workflows/smartmoney.yml/dispatches`) with a fine-grained PAT scoped to `actions: write` on this repo. `workflow_dispatch` is not subject to the 60-day inactivity auto-disable — only `schedule:` is.

## What's left here

| Workflow | Trigger | Data it refreshes |
| --- | --- | --- |
| `smartmoney` | `workflow_dispatch`, called every 6h by the external pinger | Hyperliquid smart-money leaderboard snapshots |

Everything else lives in the private Next.js app's `/api/cron/*` routes, called directly by the external pinger. See that repo for `fast` / `hourly` / `sixhourly` group membership.

## Configuration

| Secret | Purpose |
| --- | --- |
| `CRON_SECRET` | Shared token the `smartmoney` script sends to `/api/ingest/smartmoney` so the endpoint can reject anything that is not this job. |
| `SITE_URL` | Base URL of the deployment `smartmoney` should POST its result to. |

No values are committed. Set them under Settings, Secrets and variables, Actions.

## Author

Built by Oliver Arjonilla as part of the data engineering behind a live crypto market index.
