# Copy Trading Architecture

## Overview

The scanner (`app/main.py`) runs every **15 minutes** via cron job `1979b309d4db` (changed from 30 min 2026-06-26). Each run: discovers wallets from the Polymarket leaderboard, scores them on closed-position performance, fetches their latest activity, filters candidates through tiered thresholds with staleness guard, and opens paper copy trades.

## Wallet Discovery

`leaderboard_wallets()` (line 528) scrapes `https://polymarket.com/leaderboard` for wallet addresses via regex `/profile/(0x[a-fA-F0-9]{40})`. Returns top 25 unique addresses. Falls back to `SEED_WALLETS` hardcoded list if scraping fails.

**Limitation**: Only sees wallets on the public leaderboard (~25). Private or low-volume wallets are invisible. Seed wallets provide a safety net for 3 known high-performers.

## Wallet Scoring

`wallet_stats()` (line 557) queries Polymarket's `/closed-positions` API:

```
GET /closed-positions?user=0x...&limit=100
```

Computes:
1. `closed_count` = total closed positions
2. `wins` = count where `realizedPnl > 0`
3. `losses` = count where `realizedPnl < 0`  
4. `win_rate` = wins / closed_count
5. `score` = win_rate × 100 (capped at 50 if closed_count < 20)
6. `realized_pnl` = sum of all realizedPnl values

**Critical distinction**: The `/closed-positions` API shows BOTH wins and losses — it's Polymarket's settlement ledger. Losses show as `realizedPnl < 0`. The `/activity` feed (used for copy signals) only shows BUY/SELL/REDEEM — losing positions produce no follow-up activity.

**Score = win_rate × 100**. Realized PnL ($) is NOT factored into the score. A wallet with $14M PnL and one with $3M PnL both score 100 if they're both 50/50. Volume, consistency, and drawdown are not considered.

**20-trade minimum**: Wallets with fewer than 20 closed trades are capped at score 50 regardless of win rate. `0x3f87...` (2/2 wins, $7.6M PnL) scores 50 — 2 trades is not a reliable sample.

## Tiered Copy Trade Thresholds (implemented 2026-06-26)

Single thresholds (0.67 min price, 25 min size) were tuned for FIFA markets where winners trade at 0.55-0.80 with sizes of 50+. Tennis/eSports/WNBA trade at 0.15-0.60 with sizes of 5-20. A single threshold either blocks good tennis bets or allows bad FIFA bets.

**Tiered approach** (in `smart_copy_candidate()`, line 590+):

| Filter | Score 70–89 wallets | Score 90+ wallets | Why |
|--------|-------------------|-------------------|-----|
| `min_price` | 0.67 (env: `COPY_MIN_PRICE`) | 0.10 (env: `COPY_HIGH_SCORE_MIN_PRICE`) | High-score wallets value-bet at low prices |\n| `min_size` | 25 (env: `COPY_MIN_SIZE`) | 5 (env: `COPY_HIGH_SCORE_MIN_SIZE`) | High-score wallets' micro-bets still have signal |
| `max_price` | 0.93 | 0.93 | Block near-certain outcomes (both tiers) |
| Wallet win rate | 70% | 70% | Same |
| Wallet closed count | 50 | 50 | Same |
| Market horizon | ≤3 days | ≤3 days | Same |
| Blocked terms | exact score, halftime, 1st half, both teams to score | Same | Avoid degenerate props |

## Staleness Guard (added 2026-06-26)

`COPY_MAX_AGE_MINUTES = 15` — compares `activity.timestamp` (when the wallet placed the trade) vs `now()`. Skip signals older than 15 minutes. Prevents copying a 2-hour-old trade after scanner downtime.

**Why**: If the scanner pauses (maintenance, Docker restart), it could see a wallet's trade from hours ago in its "last 5" activity filter. Copying a stale trade means the market odds have moved and the edge is gone. The staleness guard prevents this.

**Current state**: All wallet signals arrive within 0-7 minutes of placement. The guard is forward protection.

## Share Minimums (all copy strategies, 2026-06-26)

All paper trades now have minimum 2 shares. Scanner INSERTs hardcode `shares=2` for copy wallet, copy consensus, and arbitrage. No single-share trades — every signal gets at least 2 shares of conviction.

## Environment variables in `docker-compose.yml`

```yaml
COPY_MIN_PRICE: "0.67"\nCOPY_HIGH_SCORE_MIN_PRICE: "0.10"\nCOPY_MIN_SIZE: "25"\nCOPY_HIGH_SCORE_MIN_SIZE: "5"\nCOPY_MAX_PRICE: "0.93"\nCOPY_MIN_WALLET_CLOSED: "50"\nCOPY_MIN_WALLET_WIN_RATE: "0.70"\nCOPY_MAX_AGE_MINUTES: "15"\nTAKE_PROFIT: "0.20"\nSTOP_LOSS: "-0.20"
```

## Candidate Filtering Pipeline

1. Fetch last 10 activities per wallet from `/activity?wallet=X&limit=10`
2. Process 5 most recent (`acts[:5]`)
3. Skip non-BUY activities (REDEEM, SELL)
4. Skip already-processed (dedup via unique_key)
5. **Staleness check**: skip if activity age > `COPY_MAX_AGE_MINUTES`
6. Apply tiered thresholds based on wallet score
7. Validate market: end ≥15 min away, resolve ≤3 days, no blocked terms
8. Open paper trade at wallet's exact price (shares=2 minimum)

## Copy Consensus

Shares the same `smart_copy_candidate()` filter as single-wallet. A candidate must pass the individual tiered filter FIRST, then gets checked for multi-wallet agreement:

1. Candidate enters `consensus_candidates` after passing `smart_copy_candidate()`
2. `open_smart_consensus_paper_trades()` groups by event_slug + asset
3. Opens trade only when 2+ wallets agree on same event + same side
4. Entry price is weighted by each wallet's trade size
5. Minimum total signal size: 100 USDC (env: `COPY_CONSENSUS_MIN_TOTAL_SIZE`)
6. Minimum 2 shares

Because consensus uses the same tiered filter as single-wallet, the 90+ score threshold relaxation (0.10 min price, 5 min size) also benefits consensus. A signal at 0.11 that was previously blocked will now pass into consensus candidates.

## Immediate Resolution (added 2026-06-26)

When a copy trade expires (event end date passes), the scanner no longer immediately marks it as `closed_pending_final_result_sync`. Instead, it queries Gamma API `/events/slug/{slug}` to find the resolved outcome. If the event has a settled market (outcomePrice ≥ 0.99), the trade is resolved directly to `closed_won` or `closed_lost`. Only if Gamma doesn't have the result yet does it fall back to `closed_pending_final_result_sync` — which `sync_final_results()` will retry on subsequent scanner runs.

This avoids the "stuck pending" problem where trades sat in pending state overnight because the scanner's `closed_pending_final_result_sync` markers were never resolved.

## Exit Rules (all copy strategies)

- **Take-profit**: +20% mark-to-market gain (env: `TAKE_PROFIT: "0.20"`)
- **Stop-loss**: −20% mark-to-market loss (env: `STOP_LOSS: "-0.20"`)
- **Resolution**: If trade reaches event end date, resolved via Gamma API `/events/slug/{slug}` — immediate won/lost, not pending

## Proposal-Before-Implementation Rule

**Never change copy trade parameters (or any strategy parameters) without Keith's explicit approval.**

1. Do analysis → present findings with data
2. Propose specific changes with reasoning
3. Wait for "go ahead" or "let's do it"
4. Only then implement

This rule was added after Keith rejected unapproved changes to `COPY_MIN_PRICE` and `COPY_MIN_SIZE` on 2026-06-26.

## Verification

Check current wallet scores:
```bash
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT wallet, score, win_rate, closed_count, ROUND(realized_pnl,2) as pnl
   FROM wallet_scores ORDER BY score DESC LIMIT 10;"
```

Check copy trade distribution:
```bash
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT kind, COUNT(*) FROM paper_trades
   WHERE kind LIKE 'copy_%' GROUP BY kind;"
```

Check staleness (signals older than 15 min):
```bash
sqlite3 /root/polymarket-intel/data/polymarket_intel.sqlite \
  "SELECT copied_paper_trade_id, seen_at,
   CAST((julianday('now') - julianday(seen_at)) * 1440 AS INTEGER) as age_min
   FROM wallet_activity
   WHERE copied_paper_trade_id IS NULL
   ORDER BY seen_at DESC LIMIT 20;"
```
