# Copy Trading Architecture

## Overview

The scanner (`app/main.py`) runs every 30 minutes via cron job `1979b309d4db`. Each run: discovers wallets from the Polymarket leaderboard, scores them on closed-position performance, fetches their latest activity, filters candidates through tiered thresholds, and opens paper copy trades.

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
| `min_price` | 0.67 (env: `COPY_MIN_PRICE`) | 0.30 (env: `COPY_HIGH_SCORE_MIN_PRICE`) | High-score wallets value-bet at low prices |
| `min_size` | 25 (env: `COPY_MIN_SIZE`) | 15 (env: `COPY_HIGH_SCORE_MIN_SIZE`) | High-score wallets' smaller bets still have signal |
| `max_price` | 0.93 | 0.93 | Block near-certain outcomes (both tiers) |
| Wallet win rate | 70% | 70% | Same |
| Wallet closed count | 50 | 50 | Same |
| Market horizon | ≤3 days | ≤3 days | Same |
| Blocked terms | exact score, halftime, 1st half, both teams to score | Same | Avoid degenerate props |

Environment variables in `docker-compose.yml`:
```yaml
COPY_MIN_PRICE: "0.67"
COPY_HIGH_SCORE_MIN_PRICE: "0.30"
COPY_MIN_SIZE: "25"
COPY_HIGH_SCORE_MIN_SIZE: "15"
COPY_MAX_PRICE: "0.93"
COPY_MIN_WALLET_CLOSED: "50"
COPY_MIN_WALLET_WIN_RATE: "0.70"
```

## Candidate Filtering Pipeline

1. Fetch last 10 activities per wallet from `/activity?wallet=X&limit=10`
2. Process 5 most recent (`acts[:5]`)
3. Skip non-BUY activities (REDEEM, SELL)
4. Skip already-processed (dedup via unique_key)
5. Apply tiered thresholds based on wallet score
6. Validate market: end ≥15 min away, resolve ≤3 days, no blocked terms
7. Open paper trade at wallet's exact price

## Copy Consensus

Shares the same `smart_copy_candidate()` filter as single-wallet. A candidate must pass the individual tiered filter FIRST, then gets checked for multi-wallet agreement:

1. Candidate enters `consensus_candidates` after passing `smart_copy_candidate()`
2. `open_smart_consensus_paper_trades()` groups by event_slug + asset
3. Opens trade only when 2+ wallets agree on same event + same side
4. Entry price is weighted by each wallet's trade size
5. Minimum total signal size: 100 USDC (env: `COPY_CONSENSUS_MIN_TOTAL_SIZE`)

Because consensus uses the same tiered filter as single-wallet, the 90+ score threshold relaxation (0.30 min price, 15 min size) also benefits consensus. A signal at 0.35 that was previously blocked will now pass into consensus candidates.

## Exit Rules (all copy strategies)

- **Take-profit**: +10% mark-to-market gain (env: `TAKE_PROFIT: "0.10"`)
- **Stop-loss**: −20% mark-to-market loss (env: `STOP_LOSS: "-0.20"`)
- **Resolution**: If trade reaches event end date, closed via Polymarket settlement data

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
