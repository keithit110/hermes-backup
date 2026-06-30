# Momentum Follower — Live Performance Analysis

Last updated: 2026-07-01

## Summary (57 closed live trades)

| Metric | Value |
|---|---|
| Total trades | 57 |
| Wins | 39 (68.4%) |
| Losses | 18 |
| Avg win | +45.4% (+$1.57 @ 5 shares) |
| Avg loss | -100% (-$3.45 @ 5 shares) |
| Avg entry | $0.689 |
| Total deployed | $194.27 |
| Total returned | $195.00 |
| Net P/L | +$0.73 (0.4%) |

## Risk/Reward

```
Win:  $0.69 × 5 shares × 45.4% = +$1.57
Loss: $0.69 × 5 shares × 100%  = -$3.45

Risk/reward: 2.2 to 1 against
Required breakeven win rate: 68.8%
Actual win rate:             68.4%  ← 0.4 points BELOW
```

The strategy is technically breakeven. The uncapped -100% losses cancel almost all the +45% wins.

## The $0.75 cap impact

| Era | Trades | Wins | Losses | Avg Entry | Max Entry | Net P/L |
|---|---|---|---|---|---|---|
| Pre-cap (IDs ≤1400) | 5 | 1 | 4 | $0.732 | $0.94 | -$11.32 (-69%) |
| Post-cap (IDs >1400) | 52 | 38 | 14 | $0.684 | $0.75 | +$12.05 (+6.8%) |

Without the max entry cap, the strategy bleeds money. Trade #1399 at $0.94 alone lost $4.70.

## Entry price distribution

| Band | Trades | Wins | Losses | Net |
|---|---|---|---|---|
| <$0.60 | 5 | 2 | 3 | -$3.45 |
| $0.60-0.64 | 8 | 7 | 1 | +$10.00 |
| $0.65-0.67 | 10 | 4 | 6 | -$11.17 |
| $0.68-0.70 | 10 | 9 | 1 | +$10.30 |
| >$0.70 | 26 | 19 | 7 | -$1.50 |

Entry price alone is not a clean predictor — $0.65-0.67 is the worst band (40% win rate) while $0.68-0.70 is the best (90% win rate). The $0.60-0.64 band is the second best (87.5% win rate).

## Proposed improvements

### Option A: -30% stop-loss (not yet implemented)

```
With -30% SL:  Win +$1.57 | Loss -$1.04 | Net +$42.69 (22%)
Required win rate: 40% ← 68.4% actual → 28pt cushion
```

Biggest impact. Same pattern as directional SL. Freeze window on SL to prevent re-entry.

### Option B: Entry band tightening (implemented 2026-07-01)

Added `MOMENTUM_MIN_ENTRY=0.58` — skip sub-$0.58 entries (5 trades, only 2 wins, net -$3.45). Combined with existing `MOMENTUM_MAX_ENTRY=0.75`.

## Current config (2026-07-01)

| Setting | Value |
|---|---|
| BTC threshold | >0.05% |
| Entry window | T+45s to T+85s |
| Entry band | $0.58 – $0.75 |
| Shares | 5 |
| Order type | FAK |
| Live | Yes (real USDC) |
| Stop-loss | None |
