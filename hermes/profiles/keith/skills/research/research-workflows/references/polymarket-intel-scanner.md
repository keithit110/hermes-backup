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
- close resolution date; for Keith's Polymarket workflow default to instant/same-day/within 3 days and ignore longer-dated markets unless he explicitly asks otherwise
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

Actionable copy-trade paper rules should be explicit and should prefer consensus over blind single-wallet copying. A conservative pattern:

- require enough closed-position history, e.g. 50+ closed positions for wallets considered qualified
- require strong win rate/profitability, e.g. 70%+ win rate or better category-specific evidence
- prefer BUY activity with identifiable market/outcome/token/price
- require the market to pass the user's horizon filter, e.g. Keith's instant/same-day/≤3-day rule
- block tail-risk/ambiguous props by default: exact score, halftime/1st-half, both-teams-to-score, leading-at-halftime, and similar markets until category-specific evidence justifies them
- require multiple qualified wallets to converge on the same event/token/outcome before opening a paper copy trade, e.g. 2+ wallets and combined copied size above a minimum such as 100
- use weighted average observed price as the paper entry for consensus trades
- open paper trades only; no private keys or live trading unless separately authorized with execution guardrails

Report wallet activity with score/confidence and copy-decision fields, but do not blindly suppress lower-score activity during early discovery because historical scoring may be incomplete. For consensus strategies, store wallet count, wallets, total copied size, weighted entry, filters applied, and signal rows in the paper-trade details JSON so later postmortems can distinguish true consensus from one wallet repeatedly trading.

## Information monitor and research agent

Pair market data with official-resolution and news feeds:

- official market resolution sources when available
- CFTC/SEC/regulatory feeds
- Google News/RSS for Polymarket and prediction markets
- domain-specific official feeds for tracked categories

Deduplicate items by URL/title and store first-seen timestamps.

For a recurring research agent, keep it local-first and evidence-based:

1. First fix candidate market discovery. Do not let the research agent trade from weak/stale market discovery; maintain an active near-resolution market table from Gamma markets, Data API activity/trades, CLOB books/prices, and category tags.
2. Read latest stored info/news items and near-resolution markets.
3. Prefer primary/official sources before commentary: league/team injury and lineup reports, official result/status pages, CFTC/SEC/Fed/filings, company/protocol blogs, court/government feeds, and reputable breaking-news wires.
4. Match news to markets using named entities, tags, title similarity, date proximity, and exact token/orderbook availability.
5. Score the edge before paper trading: source quality, freshness, direct relevance, liquidity/spread, price movement confirmation, smart-wallet/whale confirmation, uncertainty, stale-news penalty, and already-priced-in penalty.
6. Output one of `IGNORE`, `WATCH`, or `PAPER_BUY`, with confidence, source URLs, exact market/token/side/price, reasoning, and invalidation conditions.
7. Insert paper-only research trades only when confidence is high and the market/token/price is unambiguous; otherwise record `IGNORE`/`WATCH` rather than forcing a recommendation.
8. Preserve postmortem fields for every research trade: news URL, news first-seen time, trade-open time, expected catalyst, final outcome, whether the news was actually early, and resulting P/L.

## Paper trading ledger

Use SQLite/Postgres tables for:

- scanner runs
- markets/orderbook snapshots
- arbitrage opportunities
- paper trades
- wallet activity
- info items

Paper trade rules from Keith's requested pattern:

- Prefer close-resolution markets; default to instant/same-day/within 3 days unless Keith changes the horizon.
- Open paper trades only from scanner-approved signals.
- Close paper trades at +10% mark-to-market before contract end.
- Use a stop loss for paper-copy strategies, e.g. -20%, so tail-risk losses are visible before resolution.
- If the event end time passes before official final settlement data is synced, mark the trade `closed_pending_final_result_sync`; this is not a request for Keith to intervene, it means the bot needs/has not yet completed final-result synchronization.
- Later, final settlement sync should convert pending rows into `closed_won`, `closed_lost`, or `closed_void_or_unknown` based on official/resolution data.
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