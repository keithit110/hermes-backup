# Midpoint Scalp Risk/Reward Analysis (2026-06-29)

Conducted after 50 live trades on a $46.71 starting wallet (ended at ~$31).

## Raw numbers

| | UP | DOWN | Combined |
|---|---|---|---|
| Trades | 25 | 25 | 50 |
| TP wins | 16 | 15 | 31 (62%) |
| Resolution wins | 8 | 0 | 8 (16%) |
| Resolution losses | 0 | 10 | 10 (20%) |
| TP profit | +$6.71 | +$4.10 | +$10.81 |
| Resolution profit | +$18.56 | $0 | +$18.56 |
| Resolution loss | $0 | -$24.26 | -$24.26 |
| Net | **+$25.27** | **-$20.16** | **+$5.11** |

## Risk/Reward Math

- Average TP profit: $0.35 (5 shares × ~$0.50 entry × ~14% P/L)
- Average resolution loss: $2.43 (5 shares × $0.50 entry → $0)
- Ratio: 1:7 (risk $2.43 to make $0.35)
- Required break-even win rate: 87.5%
- Actual DOWN win rate: 60%
- Actual combined win rate: 78%

## DOWN is bleeding

DOWN alone net: +$4.10 (TP) − $24.26 (losses) = **−$20.16**. Every dollar UP makes, DOWN loses three. The 10 resolution losses on DOWN are the entire profit drain. At 5 shares × $0.50 entry = $2.50 per loss, one bad window wipes 7 TP wins.

## Impact of parameter changes

### Increase TP to $0.10
- Profit per win: $0.50 (doubled)
- Ratio: 1:4.9 (improved)
- Required win rate: 83%
- Risk: fewer trades reach $0.10 → more go to resolution (more wins BUT also more full losses)

### Reduce DOWN to 2 shares
- Loss per DOWN resolution: $1.00 (was $2.50)
- 10 losses → -$10.00 (was -$24.26)
- TP wins: $1.64 (was $4.10)
- DOWN net: -$8.36 (was -$20.16)
- Combined net: ~$16.91 (was $5.11)
- UP stays at 5 shares, unchanged

### Drop DOWN entirely
- Lose $4.10 in DOWN TP wins
- Save $24.26 in DOWN losses
- Combined net: ~$29.37
- Risk: when BTC trends DOWN, zero hedge — UP gets destroyed

## BTC trend bias

BTC was trending UP during the analysis window. UP has 16-0 win/loss record — but this is unsustainable. When BTC trends DOWN for a few windows, UP takes the same -100% hits. The 100% UP win rate is a product of market bias, not strategy edge.

## Recommendation priority

1. Reduce DOWN to 2 shares (highest impact, lowest risk — surgical sizing fix)
2. Increase TP to $0.10 if you want bigger wins (fewer trades but better math)
3. Drop DOWN entirely only if you're willing to ride pure directional exposure
