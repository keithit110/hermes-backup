# Momentum Follower — full live trading audit (2026-07-01)

## Summary

57 closed live trades, 0 open positions. **Technically profitable but effectively breakeven** — the same -100% loss structural problem as directional, compounded by 5-share sizing.

| Metric | Value |
|--------|-------|
| Total trades | 57 |
| Wins | 39 (68.4%) |
| Losses | 18 (31.6%) |
| Avg win | +45.4% (+$1.57 per trade) |
| Avg loss | -100% (-$3.45 per trade) |
| Avg entry | $0.689 |
| Total deployed | $194.27 |
| Total returned | $195.00 |
| **Net P/L** | **+$0.73 (0.4%)** |

## Risk/reward

```
Win:  $0.69 × 5 shares × 45.4% = +$1.57
Loss: $0.69 × 5 shares × 100%  = -$3.45
Risk/reward: 2.2 to 1 against
Required breakeven win rate: 68.8%
Actual win rate:             68.4%  ← dead center
```

## The $0.75 max entry cap saved the strategy

| Era | Trades | Wins | Losses | Avg Entry | Max Entry | Net P/L |
|-----|--------|------|--------|-----------|-----------|---------|
| Pre-cap (IDs ≤1400) | 5 | 1 | 4 | $0.732 | $0.94 | **-$11.32** (-69%) |
| Post-cap (IDs >1400) | 52 | 38 | 14 | $0.684 | $0.75 | **+$12.05** (+6.8%) |

Without the cap, trade #1399 at $0.94 lost $4.70 alone.

## Entry price distribution

| Band | Trades | Wins | Losses | Net P/L |
|------|--------|------|--------|---------|
| <$0.60 | 5 | 2 | 3 | -$3.45 |
| $0.60–0.64 | 8 | 7 | 1 | +$10.00 |
| $0.65–0.67 | 10 | 4 | 6 | -$11.17 |
| $0.68–0.70 | 10 | 9 | 1 | +$10.30 |
| >$0.70 | 24 | 17 | 7 | ~-$1.50 |

Entry price alone is not a clean predictor — $0.65-0.67 range performed worse than $0.68-0.70.

## Proposed fix: -30% mark-to-market stop-loss

Same pattern as directional. Capping losses at -30% changes the math:

| | Current (no SL) | With -30% SL |
|---|---|---|
| Avg win | +$1.57 | +$1.57 (unchanged) |
| Avg loss | -$3.45 | -$1.03 |
| Required win rate | 68.8% | 39.8% |
| Actual win rate | 68.4% | 68.4% |
| Projected net (57 trades) | +$0.73 | **+$42.69 (22.0%)** |

## Resolution bug (fixed 2026-07-01)

`resolve_pending_trades()` originally queried momentum trades with `status='open'` only. But `_close_old_window_positions()` marks ALL open trades `closed_resolved_win/loss` using Chainlink BTC data BEFORE the Gamma API correction runs. If Chainlink says UP but Polymarket settles DOWN, the correction never fires.

Example: Trade at 11:10-11:15 AM showed as a win (+$1.55) based on BTC data, but Polymarket resolved it as a loss (-$3.45). The correction query missed it because it was already `closed_resolved_win`.

Fixed: `status='open' or status like 'closed%'` — re-resolves ALL trades for the slug.
