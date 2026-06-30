# Midpoint Scalp Strategy

## Parameters (current — 2026-06-28)

| Parameter | Value | Source |
|-----------|-------|--------|
| SCALP_MIN_ENTRY | 0.47 | docker-compose.yml |
| SCALP_MAX_ENTRY | 0.53 | docker-compose.yml |
| SCALP_TAKE_PROFIT | 0.04 | docker-compose.yml (exit = entry + 0.04) |
| SCALP_SHARES | 1 | docker-compose.yml (changed from 2 for live trading) |
| SCALP_MAX_OPEN | 3 | docker-compose.yml |
| MAX_SPREAD | 0.03 | crypto engine env |
| Eval cycle | ~1 second | time.sleep(0.7) + compute |

## Decision flow (every ~1 second)

```
1. up_ask ≈ 0 or down_ask ≈ 0? → SKIP
2. spread > MAX_SPREAD? → SKIP
3. Count open scalp positions (all windows)

For UP:
  $0.47 ≤ up_ask ≤ $0.53 AND no OPEN UP in this window AND open_count < 3
  → BUY UP @ up_ask, 1 share
  → IMMEDIATELY PLACE SELL TP @ up_ask + 0.04

For DOWN: (same logic, independent)
  $0.47 ≤ down_ask ≤ $0.53 AND no OPEN DOWN in this window AND open_count < 3
  → BUY DOWN @ down_ask, 1 share
  → IMMEDIATELY PLACE SELL TP @ down_ask + 0.04
```

## Exit rules

- **TP Hit**: current_bid ≥ entry + 0.04 → CLOSE @ bid → `closed_take_profit`
- **No TP**: Hold to resolution. Winner gets $1.00, loser gets $0.00.
- **Re-entry**: After TP close, `has_scalp_position()` returns False → can re-enter same side if price still in range

## Known pitfall history

1. **LIKE pattern bug (fixed 2026-06-28)**: `%"side":"UP"%` should be `%"side": "UP"%` (space after colon). `json.dumps()` adds a space. Without it, guards never matched any row.

2. **Bailout removed (2026-06-28)**: 30s deadline bailout was closing positions at -53% avg loss. Now holds to resolution (matches nj23's approach).

3. **One-entry-per-side removed (2026-06-28)**: Over-correction that killed volume. Re-entry after TP is the intended behavior.

## nj23adsknml3 — the real strategy (verified via predicts.guru 2026-06-28)

Wallet: `0x674887d1ac838099a48b629dff53f25b7b87ee08`

| Metric | Value |
|--------|-------|
| Net PnL | +$33,898 |
| Volume | $12,212,308 |
| Fees | $260,708 (7.7× profit) |
| Trades | 2,500 |
| Avg bet | $5 |
| Markets | 1,574 |
| Open/Total positions | 1,571/1,578 |

He's a **market maker**, not a midpoint scalper. Trades at prices from 15¢ to 78¢ (not waiting for midpoint). Our midpoint scalp strategy is a different approach — focused, rules-based, achievable at small scale. We cannot replicate nj23's capital-intensive market-making.
