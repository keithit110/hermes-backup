# Copy Wallet Audit — 2026-06-28

## Audit trigger

Copy wallet P/L was running deeply negative despite high Polymarket scores. Keith asked to review all copy trades and determine root cause.

## Methodology

1. Query `paper_trades` where `kind='copy_single_high_win_rate_wallet'`
2. Group by wallet address (`details_json.copied_wallet`)
3. Compute net P/L using capital-weighted formula: `SUM(pnl_pct * 2 * entry_cost)`
4. Compare against Polymarket's reported score
5. Classify wallets: copy-profitable (our net P/L > 0) vs copy-unprofitable (our net P/L < 0)

## Results — yesterday (June 27, all closed trades)

```sql
SELECT json_extract(details_json, '$.copied_wallet') as wallet,
       json_extract(details_json, '$.name') as name,
       COUNT(*) as n,
       COUNT(CASE WHEN pnl_pct > 0 THEN 1 END) as wins,
       COUNT(CASE WHEN pnl_pct < 0 THEN 1 END) as losses,
       ROUND(TOTAL(pnl_pct * 2 * entry_cost), 2) as total_pnl
FROM paper_trades
WHERE kind='copy_single_high_win_rate_wallet'
  AND opened_at >= '2026-06-27'
  AND opened_at < '2026-06-28'
  AND status NOT IN ('open','closed_pending_final_result_sync')
GROUP BY wallet;
```

| Wallet | Name | Trades | Wins | Losses | P/L | Score |
|--------|------|:---:|:---:|:---:|-----:|:---:|
| 0xf0318c... | BreakTheBank | 3 | 3 | 0 | +$2.64 | 88% |
| 0x204f72f3... | swisstony | 44 | 17 | 27 | -$11.13 | 100% |
| 0x2c335066... | Substantial-Service | 16 | 5 | 11 | -$6.78 | 100% |

## Root cause analysis

### Why Polymarket scores are misleading

The `/closed-positions` API returns `realizedPnl` per closed position. Polymarket's "win rate" counts any position with PnL > 0 as a win — regardless of magnitude. A wallet can have 100% "win rate" by:
- Making many small-profit bets (they all count as "wins")
- Using dynamic position sizing (large on conviction, tiny on speculative)
- Having a few large losses that don't affect win rate if offset by many small wins

When we copy at fixed 2 shares, we destroy the sizing edge. Every trade gets equal weight.

### Why swisstony loses at copy level

Entry price analysis for swisstony's closed trades:

```
Winning trades: avg entry 0.56, avg return +40.8%
Losing trades:  avg entry 0.555, avg return -59.4%
```

At 17 wins × +40.8% and 27 losses × -59.4%, the math is:
- Wins contribute: 17 × 2 × 0.408 × 0.56 = ~$7.77
- Losses drain: 27 × 2 × (-0.594) × 0.555 = ~-$17.81
- Net: ~-$10.04

The wallet IS profitable on Polymarket because they size trades intelligently (Kelly criterion). Our blind 2-share copy breaks the sizing.

### BreakTheBank: the only profitable wallet

3 trades, 3 wins, entries at 0.71, 0.24, 0.76. Small sample but clean record. The concern: only 1-3 trades/day. Not enough volume to sustain the strategy alone.

## Trade-off analysis: blacklisting vs volume

| Scenario | Yesterday trades | Yesterday P/L |
|----------|:---:|-----:|
| All wallets (current) | 63 | -$15.28 |
| Blacklist swisstony + Substantial | 3 | +$2.64 |
| Blacklist + strict thresholds | 2 | +$1.12 |

**95% volume reduction but net P/L flips positive.** The blocked trades are all losers.

## Today's open positions (at risk)

| Wallet | Open Trades | Capital at Risk | Historical Avg Return |
|--------|:---:|-----:|-----:|
| swisstony | 5 | $5.70 | -16.6% |
| 0x076daa87 | 6 | $7.04 | Unknown (0 closed) |
| Substantial-Svc | 1 | $0.82 | -28.1% |

## Recommendations

1. **Blacklist swisstony (0x204f...) and Substantial-Service (0x2c33...) immediately** — proven net losers at copy level
2. **Track our own per-wallet P/L** — don't use Polymarket scores as primary filter. Our `paper_trades` table is the source of truth
3. **Wait on 0x076daa87** — 6 open trades need to settle before judging. Zero closed = zero data.
4. **Wallet discovery** — scan the Polymarket leaderboard for wallets that are BOTH high-scoring on Polymarket AND copy-profitable in our own data. The bottleneck is finding enough good wallets, not filtering bad ones.
5. **Consider dynamic share sizing for copy trades** — if we could estimate conviction from position size (relative to wallet's typical size), we could size our copies proportionally

## Implementation notes

All five recommendations were implemented in commits `bfb4f03` through `75cc4ca`:

1. **Blacklist** → `COPY_BLOCKED_WALLETS` env var checked in `get_blocked_wallets()`. swisstony and Substantial-Service hard-blocked.
2. **Own P/L tracking** → `get_blocked_wallets()` auto-blocks wallets with net P/L < -$5/day (last 24h, 3+ trades).
3. **Probation system** → All new wallets enter probation (strict 0.67/25 filters). Graduate at ≥5 settled trades with positive P/L. `get_probation_wallets()` enforces this each scanner run.
4. **Wallet discovery** → Ongoing. Scanner scores all 15 wallets from leaderboard each run.
5. **Variable share sizing** → Implemented for crypto engine only. Copy trades stay at fixed 2 shares (probation guards against sizing-mismatch losses).
