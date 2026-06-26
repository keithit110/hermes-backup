# Polymarket Intel Scanner Pattern

Use this reference when building or extending read-only Polymarket/prediction-market monitoring systems.

## Goal

Build intelligence before execution:

1. Market scanner
2. Arbitrage detector
3. Smart-wallet tracker
4. Information monitor
5. Paper-trading ledger
6. Only later: manual/live execution with separate approvals and key handling

## Container-first implementation

For user-facing monitoring agents, prefer a containerized batch job:

- Project directory with `Dockerfile`, `docker-compose.yml`, app code, `data/`, `logs/`.
- Persist SQLite or Postgres outside the image via a bind mount.
- Keep it read-only by default: no private keys, no trading endpoints, no wallet credentials.
- Verify with `docker compose build` and `docker compose run --rm <service>` before scheduling.

Example structure:

```text
/root/polymarket-intel/
  Dockerfile
  docker-compose.yml
  requirements.txt
  app/main.py
  data/polymarket_intel.sqlite
  logs/latest_report.md
  run_once.sh
```

## Polymarket public data sources

- Gamma API: `https://gamma-api.polymarket.com` for event/market discovery.
- CLOB API: `https://clob.polymarket.com` for public books/prices/spreads.
- Data API: `https://data-api.polymarket.com` for public wallet/profile/activity/positions where available.
- WebSockets later for lower-latency price/orderbook monitoring.

If direct `urllib` requests get blocked by Cloudflare/403, retry with a normal `User-Agent` using `requests.Session`. Do not encode this as a permanent tool failure; it is an HTTP-header/API-access quirk.

## Market scanner filters

Useful initial filters:

- active=true, closed=false
- close resolution date; for Keith's Polymarket workflow default to instant/same-day/within 2 days and ignore longer-dated markets unless he explicitly asks otherwise
- minimum liquidity and/or 24h volume
- enable orderbook only
- store event slug, title, resolution source, condition ID, token IDs, outcomes, end date, liquidity, volume, top bid/ask/spread

For date-only same-day sports/events, do not treat the date as midnight at the start of the day; treat it as end-of-day unless the API provides a precise timestamp. Avoid only showing event titles; include outcome labels, otherwise multi-market events look duplicated.

## Arbitrage detector rules

Read-only first. No live trading until paper logs prove the edge.

### Binary YES/NO

```text
YES best ask + NO best ask < 1 - required_edge
```

Required edge should cover fees, stale data, slippage, partial fills, and a real profit buffer.

### Multi-outcome mutually exclusive events

For negative-risk/mutually exclusive events, check the YES leg of each visible outcome:

```text
sum(best ask for YES leg of each outcome) < 1 - required_edge
```

Skip incomplete baskets, unclear placeholder outcomes, and risky `Other` cases unless the resolution terms are manually reviewed.

### Sizing

Max balanced basket size is the minimum available size across required legs. Do not treat a large edge as tradable if one leg has tiny size.

## Smart-wallet tracker

Use copy trading as intelligence plus gated paper-trade input, not passive observation and not automatic live execution.

Track:

- leaderboard/profile wallets
- recent activity/trades
- realized vs unrealized P/L if available
- category-specific success if enough history exists
- position sizing and whether entry was early or after the move
- whether each activity was paper-copied and why/why not

Actionable copy-trade paper rules should be explicit. A conservative starting pattern:

- require enough closed-position history, e.g. 20+ closed positions
- require strong win rate/profitability, e.g. 70%+ win rate or better category-specific evidence
- prefer BUY activity with identifiable market/outcome/token/price
- require the market to pass the user's horizon filter, e.g. Keith's instant/same-day/≤2-day rule
- open paper trades only; no private keys or live trading unless separately authorized with execution guardrails

Report wallet activity with score/confidence and copy-decision fields, but do not blindly suppress lower-score activity during early discovery because historical scoring may be incomplete.

## Information monitor and research agent

Pair market data with official-resolution and news feeds:

- official market resolution sources when available
- CFTC/SEC/regulatory feeds
- Google News/RSS for Polymarket and prediction markets
- domain-specific official feeds for tracked categories

Deduplicate items by URL/title and store first-seen timestamps.

For a recurring research agent, keep it local-first and evidence-based:

1. Read latest stored info/news items and near-resolution markets.
2. Research only markets that pass the horizon filter.
3. Write plain-English notes with recommendation, confidence, reasoning, source URLs, and timestamp.
4. Insert paper-only research trades only when confidence is high and the market/token/price is unambiguous.
5. If no qualifying market exists, record `IGNORE` rather than forcing a recommendation.

## Paper trading ledger

Use SQLite/Postgres tables for:

- scanner runs
- markets/orderbook snapshots
- arbitrage opportunities
- paper trades
- wallet activity
- info items

Paper trade rules from Keith's requested pattern:

- Prefer close-resolution markets; avoid waiting weeks when possible.
- Open paper trades only from scanner-approved signals.
- Close paper trades at +10% mark-to-market before contract end.
- Otherwise evaluate at/near resolution.
- Report whether tracked paper trades are profitable, not just whether opportunities existed.

## Scheduling with Hermes cron

For `cronjob(no_agent=True, script=...)`, script paths must be relative to the active profile's scripts directory, e.g. `~/.hermes/profiles/keith/scripts/<name>.sh`; absolute script paths are rejected. Put a small wrapper there that `cd`s into the project directory and runs Docker Compose.

Example wrapper:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd /root/polymarket-intel
/usr/bin/docker compose run --rm scanner
```

Use `no_agent=True` for fixed-shape watchdog/report jobs where stdout is the exact Telegram report. Consider making recurring jobs quiet unless there is a new opportunity, new paper-trade open/close, or high-relevance alert.