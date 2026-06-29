# Copy Wallet Probation System — Design & Flow

## Overview

Every wallet starts in probation. The system requires wallets to prove profitability in paper trading before getting full access. This prevents blindly copying wallets based on Polymarket's misleading "win rate" score.

## Three-tier flow

```
New wallet scored ≥70 (leaderboard)
           │
           ▼
    PROBATION (strict filters: 0.67/25)
           │
           ├── Accumulate settled trades in our paper_trades table
           │
           ▼
    ≥5 settled AND net P/L > $0?  ──Yes──→  ✅ GRADUATED (full tiered access)
           │
           No (stay in probation)
           │
           ▼
    Net P/L < -$5/day (last 24h, 3+ trades)?  ──Yes──→  ❌ BLOCKED (skipped entirely)
           │
           No
           │
           ▼
    Continue probation (strict filters apply each scan)
```

## Functions

### `get_probation_wallets(store: Store) -> set[str]`

Returns wallets that should use strict probation filters. Two cases:

**Case 1**: Wallets with 1+ settled copy trades but < N settled OR net P/L ≤ 0.
```sql
SELECT w, COUNT(*) as n, SUM(pnl_pct * entry_cost * shares) as total_pnl
FROM paper_trades
WHERE kind = 'copy_single_high_win_rate_wallet'
  AND status LIKE 'closed%'
GROUP BY w
HAVING n < ? OR total_pnl <= 0
```

**Case 2** (added 2026-06-28): Wallets scored ≥70 with ZERO settled trades.
```sql
SELECT DISTINCT ws.wallet
FROM wallet_scores ws
WHERE ws.score >= 70
  AND NOT EXISTS (
    SELECT 1 FROM paper_trades pt
    WHERE pt.kind = 'copy_single_high_win_rate_wallet'
      AND pt.status LIKE 'closed%'
      AND LOWER(json_extract(pt.details_json, '$.copied_wallet')) = LOWER(ws.wallet)
  )
```

### `get_blocked_wallets(store: Store) -> set[str]`

Returns wallets to skip entirely. Two sources:

1. **Env var blacklist**: `COPY_BLOCKED_WALLETS` (comma-separated)
2. **Daily loss cap**: wallets with net P/L < -$5 in last 24h (≥3 trades)

Both checks run every scanner tick (15 min). Blocked wallets are skipped before `wallet_stats()` is called — no API cost.

### `smart_copy_candidate(..., probation: bool = False)`

When `probation=True`, overrides tiered thresholds with strict filters regardless of Polymarket score:

```python
if probation:
    min_price = 0.67   # COPY_MIN_PRICE
    min_size = 25      # COPY_MIN_SIZE
elif score >= 90:
    min_price = 0.30   # COPY_HIGH_SCORE_MIN_PRICE
    min_size = 15      # COPY_HIGH_SCORE_MIN_SIZE
else:
    min_price = 0.67
    min_size = 25
```

## Edge cases resolved

### 1. Zero-settled wallets (2026-06-28)
**Bug**: GROUP BY on `paper_trades` only returns wallets with ≥1 row. New wallets (0 settled) never appeared in probation results → got graduated-tier access despite zero track record.

**Fix**: Added Case 2 query against `wallet_scores` for scored (≥70) wallets absent from paper_trades.

### 2. Slow-bleed graduation (2026-06-28)
**Bug**: Original HAVING `n < ?` let wallets graduate at ≥N settled trades regardless of P/L. A wallet with 10 settled and -$3 overall would graduate.

**Fix**: Changed to `n < ? OR total_pnl <= 0`. Both conditions must be false for graduation: settled ≥ N AND total_pnl > 0.

### 3. Blocked wallets in probation list (overlap)
**Scenario**: swisstony has 53 settled, -$10.95 P/L. It's both blocked (negative daily P/L) AND technically qualifies for probation (total_pnl ≤ 0). The block check runs FIRST in `track_wallets()` — `wallet.lower() in blocked_wallets` → continue. Probation status is irrelevant for blocked wallets.

## Configurable env vars

| Var | Default | Purpose |
|-----|---------|---------|
| `PROBATION_MIN_SETTLED_TRADES` | 5 | Settled trades needed to graduate |
| `COPY_BLOCKED_WALLETS` | "" | Hard blacklist (comma-separated) |
| `COPY_MAX_DAILY_LOSS_PER_WALLET` | -5.00 | Daily loss threshold for auto-block |
| `COPY_HIGH_SCORE_MIN_PRICE` | 0.30 | Graduated 90+ tier min price |
| `COPY_HIGH_SCORE_MIN_SIZE` | 15 | Graduated 90+ tier min size |
| `COPY_MIN_PRICE` | 0.67 | Probation + graduated 70-89 tier min price |
| `COPY_MIN_SIZE` | 25 | Probation + graduated 70-89 tier min size |

## Live trading integration

When flipping to real-money trading:
1. Paper trading remains as the probation layer
2. Graduated wallets route through CLOB order execution instead of paper_trades inserts
3. Real-money P/L replaces paper P/L as the graduation/blocking signal
4. `get_probation_wallets()` and `get_blocked_wallets()` queries switch from `paper_trades` to the live trades table

No algorithm changes needed — just the table name.
